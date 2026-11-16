# Permit API
What barn and rancher may ask permit, and the two libraries barn links instead of asking
------

> Status: a first inference, written to be argued with and tuned. Nothing here is ratified.

## The idea

Permit is off the request path, so its API is shaped to keep it there. Barn never asks permit "who is this?" per request: it holds permit's public keys and a feed of revocations and grant changes, verifies tokens itself with the `permit_token` library, and checks route requirements itself with the `permit_require` library. Permit is called when something about identity *changes*: a sign-in, a refresh, a consent, a grant, a revocation. And it is called by rancher, which is the only caller of its control API.

So permit has four surfaces:

1. **The feed** barn follows: keys, revocations, and grant changes.
2. **The pages** barn proxies: sign-in, consent, and OAuth, under `/_permit/`.
3. **The effects** permit drains from pond's outbox: grants, revocations, and tokens a route gave.
4. **The control API** rancher drives: realms, users, operators, audit, and key rotation.

Plus the **contracts** of the two libraries, which are as much permit's API as any method, since barn runs them on every request.

## The surface

| method | kind | callers | changes state? | answers |
| --- | --- | --- | --- | --- |
| `permit.keys.subscribe` | stream | barn | no | A snapshot of signing keys and live revocations at a seq, then key rotations, revocations, and grant changes |
| `permit.grants.get` | call | barn | no | A subject's live grants, with the feed seq they reflect |
| `permit.http.serve` | call | barn | yes (sign-in, consent, tokens) | An `HttpResponse` for a `/_permit/` request |
| `permit.control.realm.create`, `.suspend`, `.delete` | call | rancher | yes | The realm and its state |
| `permit.control.user.delete` | call | rancher | yes | What was removed, counted |
| `permit.control.operator.put`, `.drop` | call | rancher | yes | The operator account and its `host:` roles |
| `permit.control.grant`, `.revoke` | call | rancher | yes | The grant, and the feed seq it landed at |
| `permit.control.audit.read` | call | rancher | no (the read is itself audited) | Audit entries, newest first, paged |
| `permit.control.keys.rotate` | call | rancher | yes | The new key id and when the old one retires |
| `permit.control.status` | call | rancher | no | Per realm: users, sessions, grants, tokens, feed seq |

Permit also calls out:

| method | to | for |
| --- | --- | --- |
| `pond.ids.lease` | pond | Ranges of ids for users, sessions, tokens, and grants |
| `pond.outbox.lease`, `pond.outbox.ack` | pond | Grant, revoke, and token effects addressed `to: "permit"` (Q12.1) |
| `pond.deletions.subscribe` | pond | User and tenant deletions (Q12.2) |
| `barker.send` | barker | Sign-in codes and grant notices, category `auth` |
| `rancher.problems.report` | rancher | Problems |
| `ranger.ingest` | ranger | The `identity` and `audit` streams |

## Tokens: the `permit_token` contract

Barn links `permit_token` and verifies every token with it. The format is permit's to change, and only by adding a new version.

```
v1.<key id>.<base64url claims>.<base64url signature>
```

- **The signature** is Ed25519 over `v1.<key id>.<claims>`, with the realm's current signing key. One key set per deployment, not per realm: the realm is a claim, and barn checks it against the tenant it resolved from the domain.
- **Claims are JSON**, canonical (sorted keys, no whitespace), so a token is easy to read in a debugger and the bytes signed are unambiguous.

```json
{
	"v": 1,
	"id": "4294967801",
	"realm": "bobs_plumbing",
	"kind": "agent",
	"sub": "4294967100",
	"session": "4294967555",
	"grants": ["editor/project_42"],
	"iat": 1762870400,
	"exp": 1762874000
}
```

| claim | meaning |
| --- | --- |
| v | The claims version. Unknown versions are refused. |
| id | The token's id, from pond's counter. Revocation is by id. |
| realm | The tenant. A token is valid only on a domain whose tenant is this realm. Operators carry `_host`. |
| kind | `user`, `agent`, `capability`, `invite`, or `operator`. |
| sub | The subject. Absent for `capability` and `invite`. |
| session | The session it was refreshed from, for `user` and `agent`. Revoking a session revokes its tokens. |
| grants | For `agent`, `capability`, and `invite`: the grants it carries. For `user`, absent: grants are live. For `operator`, `host:` roles only. |
| iat, exp | Issued and expiry, in seconds. Access tokens live 15 minutes; capabilities and invites as long as they were minted for. |

```cpp
namespace homestead::permit::token {
enum class Kind { user, agent, capability, invite, operator_ };
struct Claims { Id id; std::string realm; Kind kind; std::optional<Id> sub, session;
                std::vector<std::string> grants; int64_t iat, exp; };
struct KeySet;   // public keys by id, with their retire times
enum class Refusal { malformed, unknown_version, unknown_key, bad_signature, expired, wrong_realm, revoked };
std::variant<Claims, Refusal> verify(std::string_view token, const KeySet&, const Revocations&,
                                     std::string_view realm, int64_t now_s);
std::string sign(const Claims&, const PrivateKey&);   // permit only; barn never holds a private key
}
```

`verify` is pure: barn passes the key set and revocations it holds from the feed, the realm it resolved, and the request's time.

## Requirements: the `permit_require` contract

A route's `permit` field is checked in barn by `permit_require`:

```json
"permit": { "any": ["owner", "admin/project_{project_id}", "viewer/project_{project_id}"] }
```

| rule | |
| --- | --- |
| Grammar | A clause is `role` or `role/kind_{param}` or `role/kind_literal`. Roles are `[a-z][a-z0-9_]*`; `_signed_in` and other `_` roles are system roles; `host:role` names a host role. |
| Substitution | `{param}` is replaced by the route's typed parameter. A value that isn't `[a-z0-9_]+` fails the clause, never the match. |
| Satisfied | When the token's grants (for `agent`, `capability`, `invite`) or the subject's live grants (for `user`) contain any clause exactly, after substitution. |
| Bound | A parameter is bound when every clause uses it, or no clause does and the requirement is realm-wide. Rancher refuses a route whose object uses an unbound parameter; `permit_require::bound()` is how it knows. |
| Host roles | Satisfy only clauses that name `host:` roles. A tenant clause is never satisfied by an operator token. |

```cpp
namespace homestead::permit::require {
struct Requirement;                                         // parsed once, at config load
std::variant<Requirement, ParseError> parse(const json::value&);
std::set<std::string> bound(const Requirement&);            // for rancher's route check
struct Outcome { bool ok; std::string satisfied_by; };      // logged in barn's routing record
Outcome check(const Requirement&, std::span<const std::string> grants, const Params&);
}
```

## The feed: `permit.keys.subscribe`

Barn opens one stream per permit connection. It starts with a snapshot and then follows the log, as every config-like subscription does (see [network](../common/NETWORK.md#subscriptions)).

```json
{ "from": 0 }
```

```json
{ "seq": 20410, "snapshot": {
	"keys": [{ "kid": "k7", "public": "MCowBQYDK2VwAyEA...", "retires": null },
	         { "kid": "k6", "public": "MCowBQYDK2VwAyEA...", "retires": 1762960000 }],
	"revoked": [{ "token": "4294967801", "until": 1762874000 }, { "session": "4294967555", "until": 1765000000 }] } }
{ "seq": 20411, "revoke": { "session": "4294967560", "until": 1765100000 } }
{ "seq": 20412, "grants": { "realm": "bobs_plumbing", "sub": "4294967100" } }
{ "seq": 20413, "key": { "kid": "k8", "public": "MCowBQYDK2VwAyEA...", "retires": null } }
```

| event | barn does |
| --- | --- |
| `snapshot` | Replaces its key set and revocations. |
| `revoke` | Adds a token, session, or subject id to its revocations until the time it would have expired anyway, after which the entry is dropped. |
| `grants` | Drops its cached live grants for that subject. The event names the subject, not the grants, so the feed never carries more than it must. |
| `key` | Adds or retires a public key. A new key is published at least one feed round before it signs anything. |
| `reset` | Resubscribe from now; drop every cached grant. |

Revocations are kept in the feed only until their tokens would expire, so the snapshot stays small (access tokens live 15 minutes).

## Grants: `permit.grants.get`

```json
{ "realm": "bobs_plumbing", "sub": "4294967100" }
```

```json
{ "seq": 20412, "grants": ["_signed_in", "owner", "editor/project_42"] }
```

Barn calls it on a cache miss for a `user` token, and caches the answer until a `grants` event for that subject arrives with a higher seq. Never more than 250 grants come back (see [permit](IDEA.md#tokens-and-grants)).

| code | when |
| --- | --- |
| `not_found` | No such subject in the realm (deleted, or never existed). Barn treats the caller as signed out. |
| `refused` | The realm is suspended or deleted. |

## The pages: `permit.http.serve`

Barn proxies every request under `/_permit/` on a tenant's domain as an `HttpRequest` (see [network](../common/NETWORK.md#the-proxied-control-plane)), with the tenant already resolved. Permit answers HTML for people and JSON for OAuth clients.

| path | does |
| --- | --- |
| `GET /_permit/sign-in` | The sign-in page: passkey first, then email code, then the realm's OIDC providers. |
| `POST /_permit/passkey/options`, `/passkey/verify` | WebAuthn registration and assertion, bound to the exact domain. |
| `POST /_permit/email/start`, `/email/verify` | Sends a code through barker (`auth`), then checks it. Codes are 6 digits, 10 minutes, 5 tries. |
| `GET /_permit/oidc/{provider}`, `/oidc/{provider}/callback` | External sign-in. |
| `POST /_permit/refresh` | A fresh access token from a session cookie or refresh token. |
| `POST /_permit/sign-out` | Ends the session; revokes its tokens. |
| `GET /_permit/oauth/authorize` | OAuth 2.1 authorization code with PKCE; shows the consent screen naming the domain and the grants asked for. |
| `POST /_permit/oauth/token` | Code and refresh grants. Answers an `agent` token. |
| `POST /_permit/oauth/register` | Dynamic client registration for MCP clients. |
| `POST /_permit/oauth/revoke` | Token revocation. |
| `GET /.well-known/oauth-authorization-server` | Metadata, proxied from the domain's root by barn. |
| `GET /_permit/invite/{token}` | Redeem an invite after sign-in. |

- **Cookies** are set for the exact domain, `Secure`, `HttpOnly`, `SameSite=Lax`, named `__Host-session`.
- **Every state-changing POST** checks `Origin` against the request's domain.
- **Rate limits** on sign-in are permit's own: per address hash and per peer, and a hammered address is a problem.
- **Operators** sign in on any domain (usually `ops.`); the answer is an `operator` token, and the audit records the realm entered.

## Effects: grants from routes

Under the lean of [Q12.1](../OPEN.md#q121-who-drains-the-outbox), permit drains its own slice of pond's outbox: it opens `pond.outbox.lease { to: "permit" }`, applies each element, and acknowledges it. If the answer goes the other way, the same elements arrive from barn as `permit.effects.apply` calls with the same payloads and keys; only the dialer changes.

```json
{ "key": "bobs_plumbing/4472/0", "tenant": "bobs_plumbing", "seq": 4472,
  "effect": { "op": "grant", "subject": "4294967100", "grant": "editor/project_42",
              "by": "4294967311", "until": null } }
```

| op | does |
| --- | --- |
| `grant` | Adds the grant to the subject, unless it already holds it. |
| `revoke` | Removes it, and for `agent` tokens derived from the subject, revokes any that carried it. |
| `mint` | Mints a `capability` or `invite` token for the given grants and expiry, stores it by id, and records it. The token's text is never in pond: the box gets the token id, and the holder fetches the text once from `/_permit/` (Q-permit.1). |

- **Deduplicated by key.** The element's key (tenant, seq, position) is remembered for 30 days; a redelivery answers the first outcome.
- **In order per subject**, because pond leases a subject's elements in commit order.
- **Refusals are permanent and visible.** A grant that would pass 250 for a subject, or names a `host:` or `_` role, is acknowledged as refused, recorded in the audit, and reported as `permit.grant_refused`, since barn already checked `gives` and this means the check and the effect disagree.

## Deletions

Under the lean of [Q12.2](../OPEN.md#q122-how-do-deletions-reach-permit-sun-and-barker), permit follows `pond.deletions.subscribe`. A user deletion removes that user's credentials, sessions, grants, and tokens and publishes a `revoke { subject }` on the feed; a tenant deletion removes the realm. The audit keeps that the deletion happened and by whom, and nothing else about the user. `permit.control.user.delete` does the same for rancher, so the two paths converge; either may arrive first, and the second is a no-op.

## The control API

Only rancher calls `permit.control.*`. Every changing call takes an idempotency key, has `dry-run`, and answers what it widens, following rancher's operator conventions.

```json
{ "method": "permit.control.grant", "realm": "bobs_plumbing", "subject": "4294967100",
  "grant": "admin", "by": "operator:4294960001", "reason": "owner handover", "dry-run": false }
```

```json
{ "seq": 20415, "widens": ["4294967100 gains admin in bobs_plumbing"] }
```

| method | notes |
| --- | --- |
| `realm.create` | With the tenant's name, id, and primary domain. Idempotent by tenant id. |
| `realm.suspend` | Sign-in refused; existing sessions keep reading, as the tenant does. |
| `realm.delete` | Removes every row of the realm; the audit keeps the act. |
| `user.delete` | As in [Deletions](#deletions). |
| `operator.put`, `operator.drop` | Operator accounts and their `host:` roles. Dropping one revokes every token it holds. |
| `grant`, `revoke` | An operator acting in a tenant. Recorded as the operator, never as a tenant member. |
| `audit.read` | By realm, subject, kind, and time, paged by cursor. |
| `keys.rotate` | Publishes a new key, signs with it after one feed round, and retires the old one after the longest access token lifetime. |
| `status` | Counts only. |

## Problems

| code | when |
| --- | --- |
| `permit.grant_refused` | An effect named a grant permit won't hold. |
| `permit.grants_over_cap` | A subject would pass 250 grants. |
| `permit.sign_in_hammered` | Too many sign-in attempts for one address hash or peer. |
| `permit.requirement_unsatisfiable` | Reported by rancher's route check through `permit_require`: no grant anyone can hold satisfies it. |
| `permit.effect_dead` | An outbox element permit could not apply before its deadline. |

## The schema, in part

```
namespace homestead.permit;

table KeysSubscribe { from: uint64; }
table PublicKey { kid: string; public: [ubyte]; retires: int64; }
table Revoke { token: uint64; session: uint64; subject: uint64; until: int64; }
table GrantsChanged { realm: string; sub: uint64; }
union KeyEvent { Snapshot, Revoke, GrantsChanged, PublicKey, Reset }
table KeysEvent { seq: uint64; event: KeyEvent; }

table GrantsGet { realm: string; sub: uint64; }
table GrantsAnswer { seq: uint64; grants: [string]; }
```

## The invariants

- **Barn verifies without calling.** Tokens and requirements are checked by libraries barn links; permit is called only when identity changes.
- **Only permit holds a private key.** The feed carries public keys and ids, never secrets or grant lists.
- **A key is published before it signs.** A token is never signed by a key a barn hasn't been offered.
- **A grant from a route arrives through pond's outbox,** deduplicated by its key, in commit order per subject.
- **Only rancher calls the control API.** Operators act as operators, never as tenant members.
- **Every change to identity is in the feed and the audit.**
- **Token formats change only by a new version.** `v1` tokens verify for as long as any can be live.
