# Barker
Every mail homestead sends, signed, limited, and delivered at least once
------

> Status: a first inference, written to be argued with and tuned. Nothing here is ratified.

## The idea

Goober's mailer lived in the host. One mailer sent everything (sign-in codes, grant notices, and boxes' mail), and around it grew SES signing, templates with overrides, an unsubscribe list, daily limits per tenant and per address, domain checks, and an outbox element with the verb `MAIL`. Most of that held up. What didn't was the place: mail shared a process with routing, identity, and storage, its ledgers sat beside the host's other files, and bounces and complaints were left to SES because nothing in the host had room to read them.

Barker is outbound mail as its own service. **Something asks barker to send a message; barker renders it, checks it against the tenant's limits and suppression list, signs it for the sender's domain, hands it to a provider, and retries until it is delivered or dead.** It reads what comes back (bounces and complaints) and stops mailing addresses that don't want mail. That is all it does.

Barker never receives mail. Inbound mail stays a barn route, handed to barn by a mail receiver (Stalwart), exactly as the barn design describes. The two meet in only three places, covered in [Inbound mail](#inbound-mail): reply addresses, threads, and bounces.

## The shape

```
   permit ---- sign-in codes, grant notices (direct) ----+
                                                         |
   rancher --- operator mail, test sends (direct) -------+
                                                         v
   barn -> pond (outbox in the request's write) ---> +----------------------+
                     drained by barker, leased       | barker (one machine) |
                                                     |  render, limit,      |
   receiver (Stalwart) -- bounces, complaints ------>|  suppress, sign,     |
                                                     |  queue, retry        |
                                                     +----------------------+
                                                         |
                                       provider: SES, SMTP, or dev (files)
```

## A message

```json
{
	"message": "4294969001",
	"tenant": "bobs_plumbing",
	"category": "transactional",
	"source": { "kind": "outbox", "request": "9f3c...", "element": 0 },
	"from": "receipts@bobsplumbing.com",
	"from-name": "Bob's Plumbing",
	"to": "ann@example.org",
	"template": "order-ready",
	"vars": { "name": "Ann", "job_id": "4294967312" },
	"reply-token": { "route": "mail:reply", "claims": { "job_id": "4294967312" } },
	"in-reply-to": "<CAF...@mail.example.org>",
	"key": "9f3c.../0"
}
```

| field | meaning |
| --- | --- |
| message | An id from pond's counter. |
| tenant | Whose mail it is, set by barn, permit, or rancher; never by a box. System mail (an operator's sign-in) has no tenant. |
| category | `auth`, `transactional`, `bulk`, or `operator`. The category picks the limits, the footer, and whether suppression applies. |
| source | Where it came from: an outbox element, permit, or rancher. |
| from | One address, fixed per mail service on the route, never chosen per call. Must be on a domain the tenant owns and barker has verified. |
| to | One address. Barker never sends to a list in one message. |
| template, vars | A template by name and its values. Text-only mail is the template `plain` with `subject` and `text`. |
| reply-token | Optional. Barker mints a signed `reply+<token>@` address for a barn mail route, so a reply can only reach what this mail was about. |
| in-reply-to | Optional, so an answer lands in the person's thread. |
| key | The idempotency key: the outbox element's key, or the caller's. A second message with the same key is the same message. |

## Where mail comes from

| source | how it reaches barker | category | notes |
| --- | --- | --- | --- |
| permit | Directly, over the internal protocol | auth | Sign-in codes and grant notices. Its own lane: never held by a tenant's limits, never stopped by suppression or unsubscribe, sent from the tenant's domain when verified and the platform's domain otherwise, so signing in never depends on a business's DNS. |
| A box | Through the outbox, in pond | transactional, bulk | The box calls the route's mail service. Barn checks the call (addresses, template exists, vars fit, from is the service's), then the message becomes an outbox element in the request's pond write. It exists if and only if the write landed. |
| rancher | Directly | operator | Mail an operator writes in the panel, replies as `admin@`, and template test sends (subject prefixed `[Test]`). |

### The outbox

The author's answer to Q7.1 makes pond the transaction coordinator for side effects. A request's write to pond carries its data and its bundle of side effects together, and pond makes the bundle durable the same way it makes the data durable (with a tiny chain replica, so a lost disk doesn't lose a committed bundle). Every service that performs a side effect drains its own slice of that outbox; barker drains mail.

1. **Lease.** Barker asks pond for mail elements past its cursor, with a lease. Pond hands each element to one barker at a time.
2. **Accept.** Barker writes the element into its own queue, keyed by the element's key, so a second delivery of the same element is the same message.
3. **Ack.** Barker acknowledges the element to pond once it is in barker's queue. From then on, barker owns retries.

The outbox design itself (the replica, the cursor, the leases) belongs to pond. Barker is a consumer of it, like sun and permit will be.

## Templates

A template is three Mustache parts, as in goober:

| part | rendered as | notes |
| --- | --- | --- |
| subject | plain text | One line, at most 200 characters after rendering. |
| text | plain text | Required. Every mail has a text part. |
| html | HTML, every value escaped | Empty means "make HTML from the text." |

A `layout` template wraps every mail and places `{{{content}}}`, the one unescaped value. Every template also gets `site` (the domain the mail is about), `email` (the recipient, whatever vars say), `preheader`, and, for transactional and bulk mail, the unsubscribe link and postal address.

The template in effect for a tenant is the first of: the tenant's copy, the platform's copy, the built-in default. Templates are configuration, held by rancher and followed by barker and barn through rancher's config log, so they move from beta to production with everything else.

| template | sent by | rule |
| --- | --- | --- |
| `sign-in-new`, `sign-in-returning` | permit | Names the site, warns never to share the code, carries no link. If it fails to render, permit's fallback text goes instead: a template can never stop a sign-in code. |
| `new-grant`, `agent-token` | permit | Notices that someone can now act for a person. |
| `plain`, and any tenant template | boxes | A box may send its tenant's templates by name, never the auth templates or the layout, so no box can send a mail that looks like a sign-in code. |

Barn checks that a template renders with the given vars when the box calls, so a bad template fails the request rather than the delivery. Barker renders again at send, against the same template version.

## Sender domains

A tenant sends from its own domain. Barker holds what that takes:

| record | barker does |
| --- | --- |
| DKIM | Holds a key per sending domain (or uses the provider's, like SES Easy DKIM) and signs every message. Publishes the record the tenant must add, and checks it is there. |
| SPF | Checks the domain's `v=spf1` record includes the provider, and warns on none or two. |
| DMARC | Checks `_dmarc` exists; suggests `p=none` with a report address to start. |
| MX | Checks replies can come back, when the tenant has mail routes. |

`domain_check {domain}` answers each finding with a level and the plain fix. A domain that isn't verified can't send transactional or bulk mail: those messages wait, and the domain is a problem. Auth mail falls back to the platform's domain.

## Providers

Sending is the `send-email@1` capability, and a provider fulfills it:

| provider | sends through | notes |
| --- | --- | --- |
| dev | Nothing: each message is a file | The default, and what a beta or sandbox uses. |
| smtp | An SMTP server, TLS required | |
| ses | Amazon SES v2, raw MIME | Bounces and complaints arrive through a configuration set. |

A tenant may bring its own provider credentials (its own SES account and reputation), named by key, never seen by a box. Switching providers changes barker's configuration, never a box.

## Delivery

| setting | default |
| --- | --- |
| attempts | 6 |
| backoff | 1 minute, doubling, at most 4 hours apart |
| give up after | 24 hours |

A temporary failure (a 4xx, a timeout, a provider's throttle) is retried. A permanent failure (a 5xx, a refused address) is a dead letter at once. Messages to one address go out in the order they were queued.

Delivery is at least once. Barker sends, then records the send; a crash between the two sends again. SES takes no idempotency key, so a person can, rarely, get one mail twice. That is the honest price, and it is named here rather than discovered.

The dead letter box keeps each message's attempts and reasons, but drops its rendered body after 7 days. Operators and agents can read, redrive, and discard dead letters through rancher's API, which proxies barker's.

## Bounces, complaints, and suppression

| signal | from | barker does |
| --- | --- | --- |
| Hard bounce | The provider, or a DSN to the bounce address | Suppress the address for the tenant. |
| Soft bounce | Same | Count it; suppress after 3 in 7 days. |
| Complaint | The provider's feedback loop | Suppress the address for every non-auth category of that tenant. |
| Unsubscribe | The one-click link (RFC 8058) in every transactional and bulk mail | Suppress for bulk, or for the category the link names. |

The suppression list is per tenant, keyed by a keyed hash of the address, so it holds no addresses. A send to a suppressed address fails in barn's call when barn can see it (barker publishes the list's hashes to barn, like permit's revocation feed), and is a dead letter if it was already queued. Auth mail ignores suppression, since a person asking for a sign-in code is asking for mail.

The unsubscribe link lives on the tenant's domain under a reserved prefix that barn proxies to barker, as it proxies rancher's and permit's. A GET asks, since mail scanners open links; a POST unsubscribes.

A tenant whose bounce or complaint rate passes a threshold has its bulk mail paused, and that is a problem.

## Limits

| limit | default | when hit |
| --- | --- | --- |
| Mail per tenant per day | 200 | Transactional and bulk wait until the next day. |
| Mail to one address per day | 3 | Further mail to that address is refused in the call. |
| Replies in a thread per address per day | 20 | Applies when `in-reply-to` is a Message-ID that address sent the tenant in the last 30 days. |
| Auth mail to one address per hour | 10 | Refused, and counted as a problem: someone is hammering sign-in. |

Counts are keyed by address hashes. A send is counted before it goes and given back if it fails.

## Inbound mail

Barker sends; barn receives, through its mail routes and the receiver. They touch in three places:

- **Reply addresses.** Barker mints `reply+<token>@domain`, signed with a key it shares with barn. Barn's mail route verifies the token and turns its claims into the request's context, so a forged token never wakes a box.
- **Threads.** When barn accepts a message, it tells barker the hashes of the sender and its Message-ID, so a reply to that thread gets the reply allowance instead of the outreach limit.
- **Bounces.** DSNs and feedback-loop reports arriving at the receiver for the bounce address go to barker, not to a box.

## Storage

Barker keeps its own SQLite database: the queue (by next attempt), the dead letter box, suppression, counts, domains and their DKIM keys (keys by reference to the key store), and recent Message-IDs as hashes. Rendered bodies stay only until delivered or 7 days dead. Barker's clock is injected, so tests run retries by hand.

## Problems and logs

Problems (see [problems](../common/PROBLEMS.md)): a dead letter, grouped by tenant, template, and reason; a domain not verified; a bounce or complaint rate past its threshold; a template that rendered in barn but not in barker; auth mail being hammered.

Every send, attempt, bounce, and complaint is a line to ranger, with the recipient as a keyed hash and its domain, the tenant, the template, the provider's message id, and the outcome. Never the body.

## What barker is not

- **Not a mail server.** It never accepts SMTP from the world. The receiver and barn do that.
- **Not a newsletter tool.** Lists, double opt-in, and campaigns are a box's job. Barker sends one message to one address and enforces the rules on each.
- **Not in the request path.** A box's mail leaves after its write lands, through the outbox.
- **Not the owner of the outbox.** Pond is. Barker drains its slice.

## Open questions

1. **Who renders?** Barn rendering at the call catches errors early; barker rendering at send keeps barn smaller. Rendering in both, against one template version, is the start.
2. **Platform fallback for auth mail.** Sending a business's sign-in codes from the platform's domain is reliable but looks like someone else's mail. Should a verified tenant ever be forced to its own domain?
3. **Bulk mail at all?** Goober grew categories and lists for outreach. Barker could refuse bulk entirely until a real product needs it.
4. **DKIM keys: barker's or the provider's?** Holding them makes switching providers seamless; SES Easy DKIM is less to run.
5. **Languages.** Templates per language, chosen by the person's language, aren't designed.
6. **Bodies in the outbox.** Should the outbox element carry rendered mail, or only template, vars, and the template version? The second is smaller and keeps personal text out of pond's history for longer than a send needs.

## The invariants

- **Mail from a box leaves only if its write landed.** It rides pond's outbox.
- **Delivery is at least once, keyed by the element.** A retry never invents a new message.
- **Auth mail can't be stopped by a tenant.** Not by limits, suppression, templates, or an unverified domain.
- **No box can send auth mail.** The auth templates and the layout are permit's and barker's alone.
- **The from address is fixed per mail service.** A box never chooses it per call.
- **Suppressed means suppressed.** Hard bounces, complaints, and unsubscribes stop non-auth mail to that address.
- **Barker keeps no addresses where a hash will do.** Suppression, counts, and logs are keyed by hashes.
- **Barker never receives mail.** Inbound is barn's.
