# Barker packages
How barker is cut, and which parts a person reads line by line
------

> Status: a first inference, written to be argued with and tuned. Nothing here is ratified.

## The idea

Barker is mostly a queue with opinions. The parts where a bug hurts someone are few and easy to name: **suppression** (mailing someone who asked us to stop is a legal and trust failure), **DKIM signing** (a key or canonicalization mistake makes every tenant's mail look forged), and **reply tokens** (a forged token would let a stranger wake a box with someone else's claims). Those three are R0, and each is cut down to the pure function at its heart: a hash and a lookup, a signature over bytes, a MAC over claims.

Everything around them (rendering, MIME, queueing, retries, limits, providers, DNS checks) is R1 or R2, because its failures are a late, a duplicate, or an ugly mail, which tests and the problem log catch. The rule from [common](../common/PACKAGES.md#review-tiers) applies: when an R0 package starts growing policy (which categories a suppression applies to, when to rotate a key), that policy moves to an R1 package that calls it.

## The packages

| package | namespace | does | depends on | tier |
| --- | --- | --- | --- | --- |
| `barker_proto` | `homestead::barker::proto` | `barker.fbs` and generated code | `common_wire` | R3 |
| `barker_db` | `homestead::barker::db` | Barker's SQLite schema and statements: queue, dead letters, suppression, counts, domains, threads | `common_sqlite`, `common_core` | R1 |
| `barker_suppress` | `homestead::barker::suppress` | Keyed hash of an address, the suppression set per tenant, `is_suppressed(tenant, hash, category)` | `common_crypto`, `barker_db` | R0 |
| `barker_dkim` | `homestead::barker::dkim` | DKIM signing of a finished MIME message (relaxed/relaxed, Ed25519 and RSA-SHA256) | `common_crypto` | R0 |
| `barker_reply` | `homestead::barker::reply` | **Library.** Mint and verify reply tokens; linked by barn | `common_crypto`, `common_core` | R0 |
| `barker_render` | `homestead::barker::render` | Template lookup (tenant, platform, built-in), layout, footer, `[Test]`, auth fallback text | `common_mustache`, `rancher_model` | R1 |
| `barker_mime` | `homestead::barker::mime` | Build MIME: headers, text and HTML parts, `List-Unsubscribe`, Message-ID, encoded words | `common_core` | R1 |
| `barker_limit` | `homestead::barker::limit` | Per-tenant, per-address, per-thread, and auth-per-hour counts; take and give back | `barker_db`, `common_core` | R1 |
| `barker_queue` | `homestead::barker::queue` | The queue: accept by key, order per address, attempts, backoff, give-up, dead letter | `barker_db`, `common_core` | R1 |
| `barker_provider` | `homestead::barker::provider` | `send-email@1`: the `Provider` interface and `dev`, `smtp`, `ses` implementations | `common_http`, `common_blob`, `common_core` | R1 |
| `barker_bounce` | `homestead::barker::bounce` | Parse DSNs (RFC 3464), ARF reports (RFC 5965), and SES notifications into bounce or complaint | `common_core`, `common_crypto` (SNS signatures) | R1 |
| `barker_intake` | `homestead::barker::intake` | Consume `pond.outbox.lease` for `to: "barker"`; accept into the queue, then ack | `pond_client`, `barker_queue` | R1 |
| `barker_pages` | `homestead::barker::pages` | `http.serve` for `/_unsubscribe/` and `/_barker/ses`; unsubscribe tokens | `common_http`, `barker_suppress`, `barker_bounce` | R1 |
| `barker_domain` | `homestead::barker::domain` | DNS checks (DKIM, SPF, DMARC, MX), findings and fixes, the daily recheck | `common_core` (libresolv) | R2 |
| `barker_feed` | `homestead::barker::feed` | The suppression stream barn follows: snapshot, then changes by seq | `barker_db`, `common_wire` | R2 |
| `barker_send` | `homestead::barker::send` | The sending loop: take due messages, check limits and suppression, render, build MIME, sign, hand to the provider, record | all of the above but intake, pages, feed | R1 |
| `barker_server` | `homestead::barker::server` | Binds `barker.*` methods to the core packages; checks caller services | `barker_proto`, core | R2 |
| `barker_client` | `homestead::barker::client` | The `Barker` interface (send, suppression feed, thread note, bounce ingest, http) and its wire implementation | `barker_proto`, `common_wire` | R2 |
| `barker_fake` | `homestead::barker::fake` | In-memory barker for permit's, rancher's, and barn's tests; records sends | `barker_client` | R2 |
| `barker_system` | `homestead::barker::system` | The composition root | everything | R2 |
| `barker_main` | | The binary | `barker_system` | R3 |

```
                          barker_main
                              |
                         barker_system
        +-----------+---------+----------+-------------+------------+
        |           |         |          |             |            |
  barker_server  barker_intake  barker_send  barker_pages  barker_feed  barker_domain
        |           |    \       /   |   \        |    \        |
  barker_proto  pond_client  barker_queue  |  barker_render  barker_bounce
        |                     |   barker_limit  |                 |
   common_wire                |       |     barker_mime      common_crypto
                              +-- barker_db --+    |
                                      |        barker_dkim (R0)
                       barker_suppress (R0)        |
                                      |       common_crypto
                                common_crypto

   barker_reply (R0, library) <-- barn_mail, and barker_send (to mint)
```

## Quality seeds

### barker_suppress (R0)
- **Promise.** A suppressed address never receives non-auth mail of a suppressed category from that tenant. Addresses are stored only as keyed hashes; the hash of an address is stable for a tenant until its key rotates, and a rotation rehashes nothing it can't (a rotation keeps the old key to answer lookups until every entry is rewritten from a send).
- **Tests.** Case and whitespace normalization table; every category against every reason; rotation with entries under both keys; a property test that `suppress` then `is_suppressed` holds for any address and category except `auth`.
- **Won't.** Decide what a bounce means, store an address, or know about templates.

### barker_dkim (R0)
- **Promise.** A signature verifies under the published record for every message barker builds, and a key never leaves the key store reference except into the signer.
- **Tests.** RFC 6376 and RFC 8463 test vectors; round trips through an independent verifier (OpenDKIM's test suite in CI); folded headers, 8-bit bodies, trailing whitespace.
- **Won't.** Build MIME, choose a key, or look up DNS.

### barker_reply (R0, library)
- **Promise.** `verify` accepts exactly what `mint` produced, for the same domain, before expiry, under a current or previous key; it answers in constant time and says nothing about why it failed.
- **Tests.** Gold tokens; every byte flipped; domain swaps; expired and not-yet-valid; old key after two rotations; fuzzed local parts. Linked by barn, so its QUALITY.md names `barn_mail` as a linker.
- **Won't.** Decide which route a claim may bind, or read the network.

### barker_db, barker_render, barker_mime, barker_limit, barker_queue, barker_provider, barker_bounce, barker_intake, barker_pages, barker_send (R1)
| package | promise | tests | won't |
| --- | --- | --- | --- |
| `barker_db` | Every write is one transaction; the schema migrates forward only | Schema snapshots; crash between statements | Hold policy |
| `barker_render` | One template version renders the same everywhere; every value escaped in HTML; auth always produces something to send | Gold renders per built-in template; missing vars; the auth fallback path | Pick a template a caller can't send |
| `barker_mime` | Output parses in three independent parsers; headers never carry a CR or LF from input | Header injection corpus; non-ASCII names and subjects | Sign or send |
| `barker_limit` | A count is taken before send and given back on failure; no limit is ever below zero or over by more than in-flight sends | Concurrent takes; day boundaries in the tenant's zone | Decide what happens when one is hit |
| `barker_queue` | A key is one message; messages to one address leave in queued order; every message ends sent or dead | Restart at every state; backoff table; per-address ordering under retries | Talk to a provider |
| `barker_provider` | A temporary failure is reported temporary and a permanent one permanent, per provider | Recorded SES and SMTP responses; the dev provider writes one file per message | Retry |
| `barker_bounce` | Only a report tied to a message barker sent becomes a bounce or complaint | Real DSNs and ARF reports from major providers; fuzzed; SNS signature vectors | Suppress anything itself |
| `barker_intake` | An element is acked only after its queue write commits; a redelivered element is the same message | Kill between accept and ack; duplicate leases | Render or send |
| `barker_pages` | GET never changes state; POST is idempotent; a forged token is a 404 | Scanner-style GETs; RFC 8058 POSTs; bad SNS signatures | Show an address |
| `barker_send` | A message passes limits, suppression, and a verified domain before it reaches a provider | Every refusal path; the auth lane never stopped by tenant limits or suppression | Accept messages |

## How it's wired

`barker_system` builds, in order: the config, `barker_db` on its file, `barker_suppress`, `barker_limit`, `barker_queue`, the provider named in the config, `barker_render` fed by a `rancher_client` config subscription, `barker_send`, `barker_intake` on a `pond_client`, `barker_feed`, `barker_pages`, `barker_domain`'s daily timer, and finally `barker_server` on the wire listener.

| thread | runs |
| --- | --- |
| network (Asio) | The listener, the clients to pond, rancher, and ranger |
| send workers (4 by default) | `barker_send`: render, sign, and hand to the provider. Provider calls are async on the network threads. |
| one DB thread | Every `barker_db` write, so the queue has one writer |

**Start:** open the database, then follow rancher's config log until templates are loaded, then start sending, then open the outbox lease, then listen. Barker answers `unavailable` until templates are loaded, because it can't render without them. **Shutdown:** send GOAWAY, stop leasing, let in-flight provider calls finish (up to 10 seconds; anything unfinished stays queued and is retried on start), flush the spool, close the database.

## Review queue

| tier | packages | read |
| --- | --- | --- |
| R0 | `barker_suppress`, `barker_dkim`, `barker_reply` | Every line, after an adversarial agent review |
| R1 | `barker_send`, `barker_queue`, `barker_intake`, `barker_bounce`, `barker_pages`, `barker_render`, `barker_limit`, `barker_mime`, `barker_provider`, `barker_db` | QUALITY.md, headers, and tests |
| R2 | `barker_server`, `barker_client`, `barker_fake`, `barker_feed`, `barker_domain`, `barker_system` | QUALITY.md, skimmed |
| R3 | `barker_proto`, `barker_main` | Agent review |

Build order, so agents can work in parallel:

1. `barker_proto`, `barker_db`, `barker_reply`, `barker_dkim`, `barker_mime`, `barker_bounce` (no barker dependencies on each other).
2. `barker_suppress`, `barker_limit`, `barker_queue`, `barker_render`, `barker_provider`, `barker_domain`, `barker_client` then `barker_fake`.
3. `barker_send`, `barker_intake`, `barker_pages`, `barker_feed`.
4. `barker_server`, `barker_system`, `barker_main`, then barker in `sim_world`.

## The invariants

- **Three R0 packages, each a pure function at heart.** Suppression, signing, and reply tokens.
- **Policy lives outside R0.** Which category a suppression covers is `barker_send`'s decision; the set is `barker_suppress`'s.
- **One writer to the queue.**
- **Barker doesn't send until it has templates**, and doesn't lease the outbox until it can send.
- **`barker_reply` is the only barker code barn links.**
