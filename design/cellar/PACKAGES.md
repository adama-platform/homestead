# Cellar packages
A small library where nearly everything is a boundary, so it is kept small enough to read in full
------

> Status: a first inference, written to be argued with and tuned. Nothing here is ratified.

## The idea

A bug in cellar is total: it hands a secret to the wrong service, leaves one in memory, or lets the ciphertext lie about what it is. So cellar is mostly R0, and the design work is keeping each R0 package tiny. The cut:

- **The format** (`cellar_envelope`) knows bytes and nothing about keys.
- **Sealing** (`cellar_seal`) needs only a public key, so rancher can link it without linking anything that opens.
- **Opening** (`cellar_open`) and **the vault** (`cellar_vault`) are the only packages that touch a private key, and **locked memory** (`cellar_buffer`) is the only place plaintext lives.
- **Bookkeeping** (which sealed secrets a process holds, versions, `for` checks, audit counts) is R1 and R2, outside the boundary.

Cellar is a homestead library: it depends on `common_*` and links into the services that use it, as [common/PACKAGES.md](../common/PACKAGES.md#library-packages) allows for pure code. It has no threads, no sockets, and no files except the vault adapter's one read at start.

## The packages

| package | namespace | does | depends on | tier |
| --- | --- | --- | --- | --- |
| `cellar_buffer` | `homestead::cellar::buffer` | `SecureBuffer`: allocated from OpenSSL's secure heap, `mlock`ed, excluded from core dumps, cleansed on release, not copyable or printable | `common_crypto` | R0 |
| `cellar_envelope` | `homestead::cellar::envelope` | The sealed format: header, binding FlatBuffer, suites, the associated data, chunked bodies; encode, decode, and `inspect` | `common_core` | R0 |
| `cellar_seal` | `homestead::cellar::seal` | `Sealer`: HPKE seal to a public key, bound to a `Binding`; suite 2 streaming seal for backups | `cellar_envelope`, `common_crypto` | R0 |
| `cellar_vault` | `homestead::cellar::vault` | `Vault` and its adapters (`file`, `systemd`; later `tpm`, `kms`): load once, check mode and owner, reload on SIGHUP | `cellar_buffer`, `common_crypto` | R0 |
| `cellar_open` | `homestead::cellar::open` | HPKE open with the vault's private key into a `SecureBuffer`, checking the binding against the caller's service | `cellar_envelope`, `cellar_vault`, `cellar_buffer` | R0 |
| `cellar_secrets` | `homestead::cellar::secrets` | Holds sealed secrets by name and version as they arrive from the config log; picks the newest openable version; refuses names not `for` this service before calling open | `cellar_envelope`, `common_core` | R1 |
| `cellar_audit` | `homestead::cellar::audit` | Counts opens per name, purpose, and minute; flushes to an `AuditSink`; never sees a value | `common_core`, `common_log` | R2 |
| `cellar_api` | `homestead::cellar` | The facade: `Cellar`, `Opened`, `Name`, `Purpose`, `AuditSink`. The only header services include for opening. | `cellar_open`, `cellar_secrets`, `cellar_audit` | R1 (its header is R0) |
| `cellar_fake` | `homestead::cellar::fake` | A test vault with a fixed key and a recording `Cellar` | `cellar_api` | R2 |
| `cellar_fuzz` | | Fuzz targets: envelope decode, `inspect`, open of mutated envelopes | `cellar_open` | R2 |
| `cellar_main` | | The `cellar` command line: `keygen`, `seal`, `inspect`, `verify`, `reseal`, `open-backup` | `cellar_seal`, `cellar_open`, `common_config` | R2 |

Who links what:

| linker | links | why |
| --- | --- | --- |
| rancher | `cellar_seal` only | It seals and stores; it must never be able to open. A CI check fails rancher's build if `cellar_open` appears in its link closure. |
| barn | `cellar_api`, `cellar_seal` | Opens S3, hound, and reply-token secrets; seals per-user tokens into pond, and values entered at porch. |
| permit, barker | `cellar_api` | Open signing keys, DKIM keys, and provider credentials. |
| pond, ranger | `cellar_seal` | Seal backups and archives to the backup public key; restores happen offline. |
| farmhand | `cellar_main` (as a binary) | Key generation, rotation, and restores. |

## Dependencies

```
                         common_core   common_crypto (OpenSSL 3)   common_log
                              ^              ^       ^                 ^
                              |              |       |                 |
   cellar_envelope -----------+        cellar_buffer |                 |
        ^       ^                         ^   ^      |                 |
        |       |                         |   |      |                 |
   cellar_seal  |                cellar_vault |      |                 |
     (rancher,  |                      ^      |      |                 |
      pond,     +------- cellar_open --+------+      |                 |
      ranger)                ^                       |                 |
                             |     cellar_secrets    |     cellar_audit+
                             |          ^                       ^
                             +----- cellar_api -----------------+
                                       ^       ^
                         (barn, permit, barker)  cellar_fake
                                       
   cellar_main -> cellar_seal, cellar_open        cellar_fuzz -> cellar_open
```

## Quality seeds

### cellar_buffer (R0)

- **Promise.** Plaintext lives only in locked, non-dumpable memory and is cleansed before that memory is freed, on every path, including exceptions and moves.
- **Tests.** A moved-from buffer is empty. Destruction cleanses (checked with a debug allocator hook). The type can't be copied or streamed (a compile-fail test). Allocation failure is an error, never a fallback to the ordinary heap.
- **Won't.** Grow, concatenate, or convert to `std::string`.

### cellar_envelope (R0)

- **Promise.** Decoding either yields exactly what was encoded, or refuses by name. Every byte before the ciphertext is in the associated data. Unknown versions and suites are refused.
- **Tests.** Gold vectors per suite. Every single-byte mutation of a gold envelope is refused at decode or open. Length fields at their limits. Fuzzing.
- **Won't.** Know a key, or decide which service may open what.

### cellar_seal (R0)

- **Promise.** A sealed value opens only with the matching private key and the identical binding. A fresh encapsulation is used for every seal.
- **Tests.** RFC 9180 test vectors through OpenSSL. Round trips with `cellar_open`. Two seals of one value differ. Chunked bodies detect truncation, reordering, and a missing final chunk.
- **Won't.** Link or include anything that opens.

### cellar_vault (R0)

- **Promise.** The private key is read once at start (and on SIGHUP), only from a source whose permissions are right, and lives only in a `SecureBuffer`.
- **Tests.** Wrong mode, wrong owner, a path inside the data directory, a missing systemd credential: each refuses to start with a message naming the fix. Reload adds and removes keys, and a failed reload keeps the old ones.
- **Won't.** Write a key, print one, or call out on a request path.

### cellar_open (R0)

- **Promise.** A value opens only for a service named in its binding's `for`, under its own name, with a key the vault holds, and the plaintext goes straight into a `SecureBuffer`.
- **Tests.** Every mismatch (service, name, version, key id, suite) refuses with its own code. The plaintext never exists outside a `SecureBuffer` (checked by an allocator-tracing test). Fuzzing on mutated envelopes.
- **Won't.** Cache plaintext, or decide which version is newest.

### cellar_secrets (R1)

- **Promise.** The newest version of each secret this service was sent is the one opened, a dropped version is forgotten, and a name not `for` this service is refused before any cryptography runs.
- **Tests.** Versions arriving out of order; a drop during an open; a config log `reset` replaces everything.
- **Won't.** Open anything itself.

### cellar_api (R1, header R0)

- **Promise.** `open` is the only path to plaintext, and every call is counted, whether it succeeds or not.
- **Tests.** Contract tests against `cellar_fake`. Every failure maps to its problem code.
- **Won't.** Offer a way to read every secret, or a value as a string.

## How it's wired

Cellar has no system of its own. A service that opens secrets builds, in its `_system`:

1. The `Vault`, from the service's config, before anything else. A failure stops the start.
2. The `Cellar`, with the vault, the service's name, an `AuditSink` bound to its `common_log` audit stream, and its clock.
3. A small adapter in the service (not in cellar) that subscribes to rancher's config log for the `secret.*` kinds and calls `Cellar::put` and `drop`.

Then the service passes `Cellar&` to the packages that need a secret (in barn: `barn_silo`, the hound client, `barn_mail`), and nothing else. On SIGHUP, the system calls `Vault::reload()`.

## Review queue

| tier | packages |
| --- | --- |
| R0 | `cellar_open`, `cellar_seal`, `cellar_vault`, `cellar_envelope`, `cellar_buffer`, the header of `cellar_api` |
| R1 | `cellar_secrets`, `cellar_api` |
| R2 | `cellar_audit`, `cellar_main`, `cellar_fake`, `cellar_fuzz` |

Build order, so agents can work in parallel:

1. `cellar_buffer` and `cellar_envelope` (no cellar dependencies).
2. `cellar_seal` and `cellar_vault`, in parallel.
3. `cellar_open`, then `cellar_secrets` and `cellar_audit` in parallel.
4. `cellar_api`, `cellar_fake`, `cellar_main`, `cellar_fuzz`.

A person reads every line of the R0 packages. Together they should stay under about 1,500 lines, and growth past that means code is moving out.

## The invariants

- **Every R0 package is small enough to read in full,** and together they stay that way.
- **Rancher's link closure never contains `cellar_open`.** CI checks it.
- **Only `cellar_open` and `cellar_vault` touch a private key, and only `cellar_buffer` holds plaintext.**
- **Cellar has no threads, sockets, or files** beyond the vault's read at start.
- **Mainstream primitives only,** through `common_crypto`.
