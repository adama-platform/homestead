# Permit packages
How permit is cut so its many boundaries each stay small enough to read
------

> Status: a first inference, written to be argued with and tuned. Nothing here is ratified.

## The idea

Permit is the service where the most code is a boundary. Every authentication method is a way in, every token is a key to the house, and every grant is a decision about who sees whose data. If permit were cut like a normal service, nearly all of it would be R0, and "a person reads every line" would mean "a person reads permit," which nobody does twice.

So permit is cut by one rule: **an R0 package holds the decision, and nothing else.** The decision to accept a passkey assertion is R0; the HTML that shows the button is not. The rule that an agent token holds a subset of its subject's grants is R0; the SQL that stores grants is not. The signature over a token is R0; the feed that carries public keys to barns is not. Each R0 package is a pure function, or close to one, over data the R1 and R2 packages fetch and store for it. That gives permit twelve R0 packages, each a few hundred lines, instead of one R0 service.

Two of those R0 packages are libraries barn links: `permit_token` and `permit_require`. They are the reason permit is off the request path, and they are the most read code in permit, because a bug in them is a bug in every request.

## The packages

| package | namespace | does | depends on | tier |
| --- | --- | --- | --- | --- |
| `permit_token` | `homestead::permit::token` | **Library.** Token format v1: claims, canonical encoding, sign, verify, refusals. Pure. | `common_core`, `common_crypto` | R0 |
| `permit_require` | `homestead::permit::require` | **Library.** Requirement parsing, substitution, `check`, `bound`. Pure. | `common_core`, `common_json` | R0 |
| `permit_keys` | `homestead::permit::keys` | Signing key lifecycle: generate, publish-before-sign, rotate, retire. Private keys held through cellar, in memory only to sign. | `permit_token`, `cellar` (library), `common_crypto`, `common_core` | R0 |
| `permit_grant` | `homestead::permit::grant` | The grant rules: role syntax, 250 cap, no `host:` or `_` from tenants, agent grants ⊆ subject's, revoke cascades. Pure decisions over a `GrantStore` interface. | `permit_require`, `common_core` | R0 |
| `permit_session` | `homestead::permit::session` | Sessions and refresh: issue, rotate refresh tokens, detect reuse, expire, revoke. | `permit_token`, `permit_keys`, `common_crypto` | R0 |
| `permit_passkey` | `homestead::permit::passkey` | WebAuthn registration and assertion: challenge, origin and RP id, signature, counter. | `common_crypto`, `common_json` | R0 |
| `permit_email` | `homestead::permit::email` | Email codes: mint, hash at rest, 10 minutes, 5 tries, rate per address hash. | `common_crypto`, `common_core` | R0 |
| `permit_oidc` | `homestead::permit::oidc` | External sign-in: state, nonce, ID token checks against the provider's keys. | `common_crypto`, `common_http`, `common_json` | R0 |
| `permit_agents` | `homestead::permit::agents` | Delegation decisions: may this parent delegate, is the scope a subset after the role tree, depth at most three, lifetime bounded, lineage built; and the subtree a revocation reaches. Pure, over a `SessionTree` interface. | `permit_grant`, `porch_roles`, `common_core` | R0 |
| `permit_oauth` | `homestead::permit::oauth` | OAuth 2.1 server: client registration, authorize with PKCE, code exchange, consent decision, token exchange for delegation, agent token issue. | `permit_grant`, `permit_agents`, `permit_session`, `common_crypto` | R0 |
| `permit_db` | `homestead::permit::db` | The SQLite schema and statements for realms, users, credentials, sessions, grants, tokens, codes, audit. Implements the store interfaces R0 packages declare. | `common_sqlite`, `common_core` | R1 |
| `permit_feed` | `homestead::permit::feed` | The keys/revocations/grant-changes log, its window, snapshot and follow. | `permit_db`, `common_wire` | R1 |
| `permit_audit` | `homestead::permit::audit` | Writes the audit table and the `identity` and `audit` ranger streams; hashes addresses. | `permit_db`, `common_log` | R1 |
| `permit_pages` | `homestead::permit::pages` | The `/_permit/` HTTP surface: routes requests to the R0 packages, renders sign-in and consent HTML, sets cookies, checks `Origin`. | every R0 auth package, `common_http`, `common_mustache` | R1 |
| `permit_effects` | `homestead::permit::effects` | Drains `pond.outbox.lease {to: permit}`, dedupes by key, applies through `permit_grant`, acks (Q12.1). Follows `pond.deletions.subscribe` (Q12.2). | `permit_grant`, `permit_db`, `pond_client` | R1 |
| `permit_control` | `homestead::permit::control` | The control API's logic: realms, users, operators, operator grants, rotation, dry runs and widenings. | `permit_grant`, `permit_keys`, `permit_db`, `permit_audit` | R1 |
| `permit_proto` | `homestead::permit::proto` | `permit.fbs` and generated code. | `common_wire` | R3 |
| `permit_server` | `homestead::permit::server` | Binds methods to `permit_feed`, `permit_db`, `permit_pages`, `permit_control`; checks callers. | `permit_proto`, core | R2 |
| `permit_client` | `homestead::permit::client` | The `Permit` interface barn uses (`keys`, `grants`, `http`) and its wire implementation; contract tests. | `permit_proto`, `common_wire` | R2 |
| `permit_fake` | `homestead::permit::fake` | In-memory permit: realms, subjects, grants, a feed, a test key that signs real tokens. | `permit_client`, `permit_token` | R2 |
| `permit_system` | `homestead::permit::system` | The composition root. Follows `rancher.config.subscribe` for the role tree. | everything above, `pond_client`, `barker_client`, `rancher_client`, `common_config` | R2 |
| `permit_main` | | The binary. | `permit_system` | R3 |

```
                        permit_main
                            |
                       permit_system ---------------------------+
                      /      |      \                            |
            permit_server  permit_effects  permit_control   pond_client, barker_client,
              |    |           |      \       |       \       rancher_client, ranger (common_log)
     permit_proto  permit_pages  permit_feed  permit_audit
                   /  |   |   \        \        /
     permit_oauth  passkey email oidc   permit_db (implements the R0 store interfaces)
        |    \
 permit_session permit_agents ---> porch_roles
        |            |
        |       permit_grant
        |            |
   permit_keys   permit_require  <---- barn links
        |
   permit_token  <---------------------- barn links
        |
   common_crypto, common_core
```

R0 packages point down at other R0 packages and `common_*` only. They declare the interfaces they need storage through (`GrantStore`, `CredentialStore`, `SessionStore`), and `permit_db` implements them. So no R0 package includes SQL, HTML, or the wire.

## Quality seeds

### R0

**permit_token**
- *Promise:* `verify` accepts exactly the tokens `sign` produced with a live, published key, for the realm asked, before expiry, not revoked. Everything else is a named refusal.
- *Tests:* gold vectors (claims, key, token text) checked into the package; a revoked ancestor in `lineage`; tokens without `lineage` still verifying; every refusal; one-byte flips anywhere in a token; non-canonical claims refused; fuzzing `verify` on arbitrary bytes; a token for realm A on realm B.
- *Won't:* fetch keys, read the clock, hold a private key in barn's build (signing is behind a separate header barn doesn't include).

**permit_require**
- *Promise:* a requirement is satisfied only by an exact grant after substitution; host roles satisfy only host clauses; `bound` names exactly the parameters every clause uses.
- *Tests:* tables of requirement × grants × params → outcome; substitution of hostile values (`../`, `*`, `/`, empty, unicode); host and system roles; property test that adding a grant never turns a satisfied check into a refusal.
- *Won't:* wildcards, role hierarchies, or any grant that implies another. The `ops/` role tree is `porch_roles`'s, applied by porch and `permit_agents`, never here.

**permit_keys**
- *Promise:* a key is published at least one feed round before it signs; a retired key verifies until the longest token it signed has expired, then never.
- *Tests:* rotation under a manual clock; a barn that missed a round; rotation during a restart.
- *Won't:* store private key bytes in permit's database; they stay in cellar.

**permit_grant**
- *Promise:* tenants hold only tenant roles; at most 250 per subject; an agent token never holds a grant its subject doesn't; revoking from a subject revokes it from every agent token derived from it.
- *Tests:* the cap at 249/250/251; `host:` and `_` from every entry point; agent subset after subject revocation; property test over random grant/revoke sequences.
- *Won't:* decide what a grant lets anyone do.

**permit_agents**
- *Promise:* a child token holds only grants its parent holds (after the role tree's containment, and only within `ops/`), lives no longer than its parent's session and 24 hours, is at most three deep, and carries the full lineage; only a parent holding a delegating role may mint; revoking a session reaches exactly its subtree.
- *Tests:* tables of (tree, parent grants, requested scope) → outcome; depth 3 and 4; lifetime clamping under a manual clock; property tests: a child's grants are always a subset of every ancestor's; revoking any ancestor makes `permit_token::verify` refuse every descendant; product grants and `host:` roles never pass through delegation.
- *Won't:* ask for consent, store sessions, or sign (it decides; `permit_oauth` and `permit_session` act).

**permit_session**
- *Promise:* refresh tokens rotate on every use, and reuse of an old one revokes the whole session.
- *Tests:* reuse detection; concurrent refresh; expiry; sign-out revokes derived access tokens.
- *Won't:* set cookies (that's `permit_pages`).

**permit_passkey**
- *Promise:* an assertion is accepted only for the exact RP id and origin of the realm's domain, with a fresh challenge and a valid signature.
- *Tests:* WebAuthn conformance vectors; wrong origin, wrong RP id, replayed challenge, counter regression; fuzzing the CBOR and client data parsers.
- *Won't:* attestation policy beyond "none" until a product asks.

**permit_email**
- *Promise:* a code is single-use, expires in 10 minutes, allows 5 tries, and is stored only as a hash.
- *Tests:* each limit at its edge under a manual clock; timing-safe comparison.
- *Won't:* send mail (it returns what to send; `permit_pages` asks barker).

**permit_oidc**
- *Promise:* an external identity is accepted only with matching state and nonce, a valid signature from the provider's current keys, the right audience, and an unexpired token.
- *Tests:* recorded provider responses; each check failing alone.
- *Won't:* trust email claims from a provider not marked as verifying them.

**permit_oauth**
- *Promise:* a code is issued only after consent, redeemable once, only with the PKCE verifier and the registered redirect URI, and yields an agent token whose grants are those consented to and a subset of the subject's.
- *Tests:* the OAuth 2.1 security checklist as tests; code replay; redirect URI mismatch; consent for more than the subject holds.
- *Won't:* implicit flow, password grant, or tokens without PKCE.

### R1

- **permit_db.** *Promise:* every row is keyed by realm, every user row by user id, so a realm or user deletes completely. *Tests:* deleting a user leaves no row with their id outside the audit. *Won't:* make decisions.
- **permit_feed.** *Promise:* every revocation and grant change is in the feed, in seq order, until its token would have expired. *Tests:* snapshot then follow; `reset` past the window. *Won't:* carry grant lists or secrets.
- **permit_audit.** *Promise:* every grant, revoke, sign-in, failed sign-in, operator entry, and audit read is recorded once. *Tests:* each action writes one entry; addresses are hashed. *Won't:* hold an address or a token's text.
- **permit_pages.** *Promise:* every POST checks `Origin`, every cookie is `__Host-`, and every decision is delegated to an R0 package. *Tests:* HTTP-level tests per path against cellar's fake; CSRF attempts. *Won't:* compare a credential itself.
- **permit_effects.** *Promise:* each outbox element is applied once and acknowledged after it is durable. *Tests:* redelivery, crash between apply and ack, ordering per subject. *Won't:* decide whether a grant is allowed (that's `permit_grant`).
- **permit_control.** *Promise:* every changing call has a dry run and states its widenings. *Tests:* dry run changes nothing; idempotency. *Won't:* bypass `permit_grant`.

## How it's wired

`permit_system` builds, in order: the config; `permit_db` (opens the database, migrates its own schema); cellar and `permit_keys`; the R0 packages with `permit_db`'s store implementations; `permit_audit` with the ranger spool; `permit_feed`; `permit_pages` and `permit_control`; the clients it dials (pond, barker, rancher); `permit_effects`; and last `permit_server`, which starts listening.

| thread pool | runs |
| --- | --- |
| network (Asio) | the wire, encode and decode |
| work (small, fixed) | SQLite writes through one writer connection; signing; password-free crypto |

**Start.** Permit refuses OPENs with `unavailable` until it has an id range from pond and its current key is published. **Shutdown.** GOAWAY, finish open calls, stop the effects stream (unacked elements return to pond), flush the audit spool, close the database.

## Review queue

Permit has eleven R0 packages, so its review budget is planned rather than hoped for:

1. **`permit_token` and `permit_require` first and always.** Barn links them; a change here is read line by line by a person and reviewed adversarially by an agent before that.
2. **`permit_grant`, `permit_agents`, `permit_session`, `permit_keys`.** The rules every token depends on. `permit_agents` is new and small, and is where an agent hierarchy could leak, so it is read with `porch_roles` beside it.
3. **The four ways in**: `permit_passkey`, `permit_email`, `permit_oidc`, `permit_oauth`. Each is read against its standard's checklist, which lives in its QUALITY.md, so a reviewer checks the diff against a list, not against memory.
4. **R1**: QUALITY.md, headers, and tests. `permit_pages` is where most day-to-day changes land (wording, layout); it is R1 exactly so those changes don't queue behind R0 work.
5. **R2 and R3** by agent review.

**Build order for parallel agents.** `permit_token`, `permit_require`, `permit_proto`, and `permit_db` have no permit dependencies and start at once. Then `permit_keys`, `permit_grant`, `permit_passkey`, `permit_email`, `permit_oidc` in parallel. Then `permit_agents` (needs `porch_roles`), `permit_session`, `permit_oauth`, `permit_feed`, `permit_audit`, `permit_client`, `permit_fake`. Then `permit_pages`, `permit_effects`, `permit_control`, `permit_server`. Last `permit_system` and `permit_main`. Barn can start against `permit_fake` as soon as `permit_client` and `permit_token` exist.

## The invariants

- **R0 packages hold decisions, not plumbing.** No SQL, HTML, or wire code in an R0 package.
- **Barn links only `permit_token` and `permit_require`.** Both pure, both R0, both with gold vectors.
- **Signing is behind its own header.** Barn's build never includes code that can sign.
- **Storage is an interface the R0 packages declare.** `permit_db` implements it; fakes implement it in tests.
- **Every standard has its checklist in QUALITY.md.** WebAuthn, OIDC, and OAuth 2.1 changes are reviewed against it.
