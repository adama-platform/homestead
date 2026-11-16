# Cellar
Keys and secrets, sealed where they are set and opened only where they are used
------

> Status: a first inference, written to be argued with and tuned. Nothing here is ratified.

## The idea

Goober's keys design held up. A vault held one root key, touched only when the host started and when keys rotated. Every other secret was encrypted under envelope encryption in a database no box could reach. A box named a key and never saw its value, every use was audited by name, and values were write-only: nobody could read a key back (goober human/064, human/325). What changes in homestead is that there is no single host. Barn needs S3 credentials, hound's credentials, and the reply-token key. Permit needs signing keys. Barker needs DKIM keys and provider credentials. Pond and ranger need backup and archive keys. Each of them already says "by reference to the key store," and none of them should run a key store of its own.

Cellar is a **library**, not a service. It makes the vault design work across many processes with one asymmetric twist: **anything can seal a secret, and only a host holding the vault's private key can open one.**

- **Rancher seals and stores.** The operator tools `secret_put`, `secret_list`, `secret_drop`, and `secret_rotate` live in rancher. Rancher seals each value to the deployment's public wrap key and keeps only ciphertext. Rancher links the sealing half of cellar and never the opening half, so it can't read back what it holds.
- **The config log carries sealed secrets by name**, like `system/ses` or `tenant/bobs_plumbing/hevy_api_key`. It delivers each one only to the services named in that secret's `for` list.
- **Each host's vault holds the private unwrap key.** The cellar library loads it once at start, opens a secret at the moment of use, holds the plaintext only briefly in locked, zeroed memory, and counts every open into ranger's audit stream by name, never by value.

That keeps goober's posture: the expensive, careful thing (the vault) is touched at start and at rotation, and everything else is ordinary encryption done well.

## What cellar keeps

| secret | name | for | used by |
| --- | --- | --- | --- |
| S3 credentials | `system/s3` (or per tenant: `tenant/{tenant}/s3`) | barn | Silo reads and writes, scoped to the tenant's prefixes |
| Third-party credentials | `tenant/{tenant}/hevy_api_key`, `system/geocoder` | barn | Hound attaches them to outbound calls; a box never sees them |
| Reply-token key | `system/reply_token` | barn, barker | Barker mints `reply+<token>@` addresses; barn verifies them |
| Token signing keys | `system/permit_signing` | permit | Signing access, capability, and invite tokens |
| DKIM keys | `tenant/{tenant}/dkim/{selector}` | barker | Signing outbound mail |
| Mail provider credentials | `system/ses`, `tenant/{tenant}/ses` | barker | Sending through SES or SMTP |
| OIDC client secrets | `tenant/{tenant}/oidc/google` | permit | "Sign in with Google" per realm |
| Per-user OAuth tokens | `user/{user}/strava` | barn | Hound acting for one person; stored in pond, not in rancher (see below) |

Backup and archive encryption for pond and ranger works differently: those services only ever seal, to a separate backup key pair (see [Backups seal to a key nobody online holds](#backups-seal-to-a-key-nobody-online-holds)).

## Names and scopes

A secret's name is its scope, then its owner, then its name, as in goober:

| scope | name | set by |
| --- | --- | --- |
| system | `system/$name` | A host operator, in a person's session |
| tenant | `tenant/$tenant/$name` | The tenant's owner, or a host operator acting for them (audited) |
| user | `user/$user/$name` | The person, through a connect flow; never through rancher |

`/` is reserved in names, as it is in grant roles. A route that names a credential may substitute only from the secure context (`tenant/{!tenant}/hevy_api_key`, `user/{!user_id}/strava`), so a box can only use the key of the tenant or person it is acting for.

## Sealing

```
 person (porch)                rancher                     config log            barn, permit, barker ...
   value ----secret_put-----> seal(value, wrap_pub,       sealed bytes  ------>   holds sealed bytes
                              aad = name, scope, for)     by name, only to          |
                              store ciphertext,           services in `for`         | at the moment of use
                              fingerprint, metadata                                 v
                              (plaintext dropped)                               open(sealed, wrap_priv)
                                                                                 -> locked buffer, used,
                                                                                    zeroed; open counted
```

- **HPKE, from a mainstream library.** A sealed secret is RFC 9180 HPKE in base mode, with DHKEM(X25519, HKDF-SHA256), HKDF-SHA256, and AES-256-GCM, through OpenSSL 3 via `common_crypto`. Cellar invents no primitives. It defines only the envelope around them, a versioned byte format (see [the API](API.md#the-sealed-envelope)).
- **Bound to its name.** The secret's name, scope, owner, version, and `for` list are the associated data. A sealed value copied under another name, or delivered to a service not in `for`, fails to open. Moving ciphertext around can't move a secret.
- **One wrap key pair per deployment.** Each secret gets its own ephemeral key from HPKE, so there are no data keys to manage. The wrap key plays the role goober's root key played.
- **Values are write-only.** Nobody reads a secret back, whether through the panel, an agent, rancher, or a log. Replacing a value is the only way to change it. Rancher shows metadata: name, scope, owner, fingerprint, created, last rotated, last used, uses this week, the routes that reference it, and recent failures.
- **Values come from people.** As in goober, an agent can list secrets, see their metadata, and read the urgent requests for missing ones. It can't put, rotate, or drop one. `secret_put` requires a person's session.

## Opening

The cellar library in each process holds the sealed secrets its service was sent and the unwrap key its vault gave it. Opening is a function call:

- **At the moment of use, not before.** Hound opens the Hevy key when it attaches it to a request and drops it when the request is sent. Permit opens its signing key into a long-lived signer, which is the one place a key stays open, because signing is the key's whole job.
- **Locked memory.** Plaintext lives in buffers allocated from OpenSSL's secure heap (`mlock`ed, excluded from core dumps) and zeroed on release. The handle can't be copied, logged, or formatted.
- **Every open is counted.** Opens are aggregated per name, purpose, and minute into ranger's `audit` stream, so the audit says who used what and how often without a line per request. A secret that fails to open is a problem, never a silent fallback.
- **A missing secret is the platform's failure.** As in goober, a request that needs a secret that doesn't exist, or one the third party rejects (a 401 or 403 to a call made with it), stops before anything else happens. It answers 500, discards the request's write, and becomes an urgent request in rancher's problem log, sent to whoever can fix it: host operators, the tenant's owner, or the person.

## The vault

The vault is wherever a host keeps the private unwrap key. Cellar reads it once, at start, through an adapter:

| adapter | where the key is | notes |
| --- | --- | --- |
| `file` | A file, mode 0400, owned by the service's user, outside its data directory | The default. Cellar refuses to start if the mode or owner is wrong. |
| `systemd` | A systemd credential (`LoadCredentialEncrypted=`), read from `$CREDENTIALS_DIRECTORY` | Encrypted at rest by systemd, with the host's TPM when there is one. |
| `tpm` | Sealed to the machine's TPM | Later. |
| `kms` | Wrapped by a cloud KMS key, unwrapped once at start | Later. One KMS call per start, never per use. |

**No vault, no start.** A service that needs secrets and can't load its key refuses to start, and farmhand's readiness check says why. A running service never touches the vault again until a SIGHUP after rotation.

## Per-user tokens live in pond

A person's Strava or calendar tokens change at runtime: OAuth refresh tokens rotate on every refresh. They can't be configuration in rancher's log. So they are **sealed documents in pond's reserved `_credentials` bucket**, keyed by the user's id, one field per provider. No route can name a bucket that starts with `_`, so no box can read them.

- **Barn seals them.** Barn holds only the public wrap key for sealing, so the connect flow (hound's OAuth client) seals the token it receives and writes it through pond with the user's id as the write's user, so deleting the user deletes their tokens.
- **Barn opens them** at the moment hound attaches one, and re-seals the refreshed token in the same request's pond write.
- **The associated data binds tenant, user, and provider**, so a sealed token copied to another user's document won't open.

## Backups seal to a key nobody online holds

Pond's snapshots and shipped WAL, ranger's sealed archives, and farmhand's backups of every other database are sealed with the same envelope, but to a second key pair, the **backup key**, whose private half lives in no running service. It is kept offline by the operator, and farmhand uses it for restores and restore drills. So a compromised pond machine can write backups and never read them, which is goober's "cloud credentials are as narrow as the job" applied to the keys themselves.

## Rotation

Rotating the wrap key is farmhand's job (see [farmhand](../farmhand/IDEA.md#rotating-the-cellar-wrap-key)), as a practiced routine rather than an emergency invention:

1. **Generate** a new wrap key pair on the operator's machine (`cellar keygen`).
2. **Distribute** the private half to every host's vault. Each service reloads on SIGHUP and holds both keys during the overlap.
3. **Publish** the new public key to rancher, which seals every new secret to it from then on.
4. **Re-seal** on one host holding both keys: open each sealed secret with the old key and seal it to the new one. Rancher stores the re-sealed ciphertext and publishes it through the config log. A sweep re-seals pond's `_credentials` documents, and any document it misses is re-sealed the next time it is written.
5. **Retire** the old private key from every vault once nothing sealed to it is left, which rancher can count.

Rotating one secret's value is `secret_rotate`: a person enters the new value, rancher seals it as a new version, and the old version stays openable for a stated overlap so in-flight work finishes.

## What cellar is not

- **Not a service.** It has no port, no database, and no peers. Secrets travel in rancher's config log and pond's `_credentials` bucket, and cellar is the code that seals and opens them.
- **Not a way to read a secret back.** No API returns a value to a caller who asks for it.
- **Not protection from the running host.** As goober said plainly, a compromised running service holds its unwrap key and can open what it was sent. Cellar limits what each service is sent (`for`), and keeps boxes from ever seeing a value.
- **Not end-to-end for people.** An operator holding a vault key can open per-user tokens. Keys only the person holds are a different design.

## Open questions

1. **Seal at the edge?** Porch runs in barn, and barn holds the public wrap key, so a value could be sealed in barn and reach rancher already sealed, meaning rancher never sees plaintext even for a moment. The lean is yes, with rancher still checking the envelope's name and `for` before storing it.
2. **Fingerprints.** Showing a fingerprint needs a keyed hash of the value. The lean is a fingerprint key held in rancher's own vault, computed at seal time, so a fingerprint can't be brute-forced from a short value.
3. **Per-tenant wrap keys.** One wrap key per deployment is simple. A wrap key per tenant would make deleting a tenant shred its secrets cryptographically, at the cost of one more key per tenant in every vault. The lean is per deployment until a tenant needs to be exportable with its keys.
4. **The re-seal sweep over `_credentials`.** Which process runs it (a barn system job or a rancher-driven one), and how its progress is shown.

## The invariants

- **Anything can seal; only a vault can open.** Rancher seals and stores, and never links the code that opens.
- **No secret ever enters a box.** Boxes name secrets on routes, and barn and hound use them.
- **A secret reaches only the services in its `for` list.** The associated data enforces it, and the config log does too.
- **Every sealed secret is bound to its name, scope, owner, and version.** Ciphertext moved elsewhere doesn't open.
- **Values are write-only, and come from people.** Agents see metadata, never values, and change none.
- **The vault is touched at start and at rotation, never per request.**
- **Plaintext is short-lived, locked, and zeroed.** Every open is counted by name into the audit, and no value is ever logged.
- **A missing or rejected secret is the platform's failure.** It is a 500, an urgent request, and a problem.
- **Backups seal to a key no running service holds.**
