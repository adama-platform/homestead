# Barker API
What barker answers on the wire, what it consumes, and the reply-token contract barn links
------

> Status: a first inference, written to be argued with and tuned. Nothing here is ratified.

## The idea

Barker's surface is small because its job is small: take a message, decide whether it may go, and get it delivered. Mail reaches it two ways. **Directly**, from the two services whose mail can't wait on a tenant's write: permit (sign-in codes, grant notices) and rancher (operator mail, test sends). **Through pond's outbox**, for everything a box asks for, so a box's mail exists if and only if its write landed. Everything else on this page is feedback flowing back (bounces, unsubscribes, threads), the state barn needs to refuse early (suppression hashes), and the operator controls rancher proxies.

Barker listens on 7405 (see [network](../common/NETWORK.md#ports)). It dials pond (ids, outbox, deletions), rancher (templates through the config log, problems), and ranger (logs). Nothing it serves is on the request path of a box.

## The surface

| method | kind | callers | changes state? | answers |
| --- | --- | --- | --- | --- |
| `barker.send` | call | permit, rancher | yes (idempotency key) | the message id and its state |
| `barker.suppression.subscribe` | stream | barn | no | a snapshot of suppression hashes per tenant, then changes |
| `barker.thread.note` | call | barn | yes | ok |
| `barker.bounce.ingest` | call | barn | yes | what barker concluded: bounce, complaint, or ignored |
| `barker.http.serve` | call | barn | sometimes (POST unsubscribe, provider notifications) | an `HttpResponse` |
| `barker.control.domain.put` | call | rancher | yes | the DNS records the tenant must publish |
| `barker.control.domain.check` | call | rancher | no | findings per record, with levels and fixes |
| `barker.control.domain.drop` | call | rancher | yes | ok |
| `barker.control.dkim.rotate` | call | rancher | yes | the new selector and its record |
| `barker.control.dead.list` | call | rancher | no | dead letters by tenant, template, reason |
| `barker.control.dead.redrive` | call | rancher | yes | how many were requeued |
| `barker.control.dead.discard` | call | rancher | yes | how many were discarded |
| `barker.control.suppression.lookup` | call | rancher | no | whether an address is suppressed, for which categories, and why |
| `barker.control.suppression.remove` | call | rancher | yes | ok, audited |
| `barker.control.limits.put` | call | rancher | yes | the tenant's limits in effect |
| `barker.control.status` | call | rancher | no | queue depth, oldest message, sends per provider, rates per tenant |

Barker consumes, as a dialer:

| method | on | for |
| --- | --- | --- |
| `pond.outbox.lease`, `pond.outbox.ack` | pond | Mail elements addressed `to: "barker"` (the lean of [Q12.1](../OPEN.md#q121-who-drains-the-outbox)) |
| `pond.deletions.subscribe` | pond | Forgetting a user's or tenant's queue, counts, and threads (the lean of [Q12.2](../OPEN.md#q122-how-do-deletions-reach-permit-sun-and-barker)) |
| `pond.ids.lease` | pond | Message ids |
| `rancher.config.subscribe` | rancher | Templates and layouts per tenant and environment, and mail services' fixed from addresses |
| `rancher.problems.report`, `ranger.ingest` | rancher, ranger | Problems and the `mail` log stream |

## Sending directly

```json
{
	"tenant": "bobs_plumbing",
	"category": "auth",
	"from-service": "sign-in",
	"to": "ann@example.org",
	"template": "sign-in-returning",
	"vars": { "code": "481 207" },
	"fallback": { "subject": "Your sign-in code", "text": "Your code for bobsplumbing.com is 481 207." },
	"key": "permit/signin/4294967555/1"
}
```

```json
{ "message": "4294969001", "state": "queued", "from": "no-reply@bobsplumbing.com" }
```

| field | meaning |
| --- | --- |
| tenant | Whose mail it is. Absent only for platform mail (an operator's sign-in on the platform domain). Trusted because the caller is permit or rancher. |
| category | `auth` from permit, `operator` from rancher. A direct call with `transactional` or `bulk` is refused: those only come through the outbox. |
| from-service | The mail service whose fixed from address applies. A direct call never names an address. |
| template, vars | A template by name. `auth` callers may name auth templates; rancher may name `plain` and the tenant's templates, and a test send prefixes `[Test]`. |
| fallback | Auth only. Plain text sent if the template fails to render, so a template can never stop a sign-in code. |
| key | The idempotency key. A repeat answers the original message. Also sent as `Open.idempotency`. |

| state | means |
| --- | --- |
| queued | Durable in barker's queue; barker owns retries from here. |
| refused | Never queued: a limit, suppression, or an unverified domain for a non-auth category. `Status.problem` says which. |

Errors: `invalid` (unknown template, vars missing a required name, a category this caller can't send), `refused` (`mail.limit_address`, `mail.suppressed`, `mail.domain_unverified`, `mail.auth_hammered`), `unavailable` while barker starts. An auth send is acknowledged once it is queued, never after delivery, so permit's sign-in page answers in milliseconds.

## The outbox

Following the lean of Q12.1, barker drains its own slice of each tenant's outbox. It holds one `pond.outbox.lease` stream per pond, with credit for how many elements it can accept at once:

```json
{ "to": "barker", "lease-ms": 60000, "from": { "bobs_plumbing": 88120 } }
```

```json
{
	"tenant": "bobs_plumbing", "seq": 88121, "element": 0, "key": "88121/0",
	"user": "4294967311",
	"effect": { "op": "send", "service": "receipts", "template": "order-ready", "template-version": "sha256:41c0...",
	            "to-user": "4294967311", "to": "ann@example.org", "vars": { "name": "Ann", "job_id": "4294967312" },
	            "reply-token": { "route": "mail:reply", "claims": { "job_id": "4294967312" } } }
}
```

1. **Accept.** Barker writes the element into its queue keyed by `key`. A second delivery of the same element finds the key and is the same message.
2. **Ack.** `pond.outbox.ack {tenant, keys}` once the queue write is durable. From then on, barker owns retries; pond's lease is released.
3. **Refuse into the dead letter box, not back to pond.** A message barker can't send (suppressed since barn checked, over a daily limit, a template version that no longer renders) is accepted, acked, and dead-lettered with its reason. Pond only redelivers what barker never acknowledged.

The element carries the template, its version, and vars, never rendered mail (the lean of Q10.6). Barker renders against that exact version.

## Suppression, for barn

```
barn -> barker   barker.suppression.subscribe { from: 3310 }
barker -> barn   { "seq": 3310, "snapshot": true, "tenants": { "bobs_plumbing": { "key-id": 3, "hashes": ["9c1f...", "..."] } } }
barker -> barn   { "seq": 3311, "tenant": "bobs_plumbing", "add": "a71e...", "categories": ["bulk", "transactional"] }
barker -> barn   { "seq": 3312, "tenant": "bobs_plumbing", "remove": "a71e..." }
```

A hash is `HMAC-SHA256(tenant suppression key, lowercased address)`, truncated to 16 bytes. The key is per tenant and shared with barn through this stream's snapshot (`key-id` names it, and rotating it sends a fresh snapshot), so barn can refuse a box's send to a suppressed address in the box's call, without barker. Auth categories are never in this list.

## Threads and bounces, from barn

`barker.thread.note` is how barn tells barker that a person wrote to the tenant, so replies to that thread get the reply allowance:

```json
{ "tenant": "bobs_plumbing", "sender": "5d02...", "message-id": "e7a1...", "at": 1762870000000 }
```

Both values are keyed hashes; barker keeps them 30 days. A note is idempotent by `(tenant, message-id)`.

`barker.bounce.ingest` carries mail barn received for the bounce address, under the lean of Q12.3:

```json
{ "domain": "bobsplumbing.com", "rcpt": "bounce+4294969001@bobsplumbing.com", "mime": "<raw bytes>" }
```

```json
{ "kind": "hard-bounce", "message": "4294969001", "tenant": "bobs_plumbing", "suppressed": true }
```

| kind | barker did |
| --- | --- |
| `hard-bounce` | Suppressed the address for the tenant's non-auth mail. |
| `soft-bounce` | Counted it; suppressed after 3 in 7 days. |
| `complaint` | Suppressed for every non-auth category. |
| `ignored` | Not a DSN or ARF report it could tie to a message it sent. Logged, not a problem. |

The bounce address names the message id, so barker never has to trust the report's own claims about who it was for. Mail is capped at 256 KiB here; barker keeps only the parsed result.

## Pages, through barn

`barker.http.serve` takes the [proxied control plane](../common/NETWORK.md#the-proxied-control-plane) envelope for two prefixes:

| path | method | does |
| --- | --- | --- |
| `/_unsubscribe/{token}` | GET | A page that asks "Unsubscribe from Bob's Plumbing's updates?" with one button. Changes nothing, since scanners open links. |
| `/_unsubscribe/{token}` | POST | RFC 8058 one-click (`List-Unsubscribe=One-Click`) or the button: suppresses for the token's category. Idempotent. |
| `/_barker/ses` | POST | Amazon SES notifications (via SNS): bounces, complaints, deliveries. The SNS signature is verified before anything else is read. |

An unsubscribe token is `{tenant, address hash, category, issued}`, signed with barker's key; a forged or expired token answers 404 and is counted.

## The control API

Every control call is rancher's, so operators and agents reach barker through rancher's operator API. Changing calls take an idempotency key, and the operator acting is named in the payload for the audit line.

```json
{ "tenant": "bobs_plumbing", "domain": "bobsplumbing.com" }
```

```json
{
	"domain": "bobsplumbing.com", "sending": "unverified",
	"findings": [
		{ "record": "dkim", "level": "error", "name": "hs1._domainkey.bobsplumbing.com",
		  "found": null, "want": "v=DKIM1; k=ed25519; p=...", "fix": "Add this TXT record." },
		{ "record": "spf", "level": "warn", "found": "v=spf1 include:_spf.google.com ~all",
		  "fix": "Add include:amazonses.com to the existing record; don't add a second SPF record." },
		{ "record": "dmarc", "level": "ok" },
		{ "record": "mx", "level": "ok" }
	]
}
```

- **`domain.put`** registers a sending domain for a tenant, generates its DKIM key (by reference to the key store), and answers the records to publish. **`domain.check`** reads DNS now and answers findings; barker also rechecks every verified domain daily, and a domain that stops verifying is a problem.
- **`dead.list`** pages dead letters by tenant, template, and reason, with every attempt; rendered bodies are gone after 7 days. **`dead.redrive`** and **`dead.discard`** take message ids or a filter, and a dry run.
- **`suppression.lookup`** takes an address, hashes it, and answers the categories and the reason (`hard-bounce` on a date). **`suppression.remove`** is an operator overriding a suppression, audited with who and why; a complaint can't be removed by a tenant's own operator.

## The reply-token contract

Barn verifies reply tokens that barker minted, without calling barker, by linking the `barker_reply` library. This is its whole contract:

```
reply+<token>@<domain>
token = base32(payload) "." base32(mac)         lowercase, no padding, at most 64 characters before the @
payload = FlatBuffer ReplyClaims
mac = HMAC-SHA256(reply key[key_id], domain || 0x00 || payload), truncated to 10 bytes
```

```
namespace homestead.barker;

table ReplyClaims {
  key_id: uint16;          // which reply key; barn holds the current and previous
  tenant: string;
  route: string;           // "mail:reply": the mail route this token may reach
  message: int64;          // the message the reply answers
  claims: [Claim];         // small, typed values the route's objects may bind
  expires: int64;          // ms; default 90 days after sending
}
table Claim { name: string; value: string; }
```

```cpp
namespace homestead::barker::reply {
std::string mint(const Key&, std::string_view domain, const Claims&);
Result<Claims> verify(const KeyRing&, std::string_view domain, std::string_view local_part, Clock::time_point now);
}
```

- **The domain is in the MAC**, so a token can't be moved to another tenant's domain.
- **A token reaches one route.** Barn refuses a token whose `route` isn't the mail route the address matched.
- **Keys come from rancher's config log** (by reference to the key store), current and previous, so rotation never breaks a reply in flight.
- **Verification is constant time**, and a failure says only "invalid," never which part.

## Central types

```
namespace homestead.barker;

enum Category : ubyte { auth, transactional, bulk, operator }
enum MessageState : ubyte { queued, sending, sent, refused, dead }

table SendRequest {
  tenant: string; category: Category; from_service: string;
  to: string; template: string; template_version: string;
  vars: [ubyte];            // JSON, opaque to the wire
  fallback: Fallback; reply_token: ReplyRequest; in_reply_to: string;
  key: string;
}
table SendAnswer { message: int64; state: MessageState; from: string; }

table SuppressionEvent {
  seq: uint64; snapshot: bool; tenant: string; key_id: uint16;
  add: [ubyte]; remove: [ubyte]; categories: [Category];
}
```

## Problem codes

| code | raised when |
| --- | --- |
| `mail.dead_letter` | A message reached the dead letter box; grouped by tenant, template, reason. |
| `mail.domain_unverified` | Transactional or bulk mail is waiting on a domain that doesn't verify. |
| `mail.bounce_rate`, `mail.complaint_rate` | A tenant passed a threshold; its bulk mail is paused. |
| `mail.render_drift` | A template rendered in barn but not in barker. |
| `mail.auth_hammered` | Auth mail to one address passed 10 an hour. |
| `mail.provider_failing` | A provider's temporary failures passed a rate for 10 minutes. |

## The invariants

- **Box mail only arrives through the outbox.** A direct `barker.send` is auth or operator mail, from permit or rancher.
- **An element is acked only once it is durable in barker's queue.** From then, barker owns it.
- **Every message has one key**, and the same key is the same message.
- **Callers never name a from address.** They name a mail service; the address is fixed.
- **Suppression travels as keyed hashes**, and never includes auth mail.
- **Reply tokens are bound to a domain and a route**, and barn verifies them with the same library barker mints them with.
- **Only rancher calls the control API**, and every override of a suppression is audited.
