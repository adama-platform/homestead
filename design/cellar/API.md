# Cellar API
The C++ library, the sealed envelope, the vault adapters, and the command line
------

> Status: a first inference, written to be argued with and tuned. Nothing here is ratified.

## The idea

Cellar has no wire API. It is a library with two halves: **seal**, which anyone may link, and **open**, which only the services that use secrets link. Rancher links seal alone, so a bug or a compromise in rancher can't read what it stores. Everything another service sees is in `<homestead/cellar/api/cellar.h>`. The rest of this doc is that header's contract, the bytes a sealed secret is made of, the vault adapters, and the `cellar` command line that farmhand drives.

| surface | linked by | does |
| --- | --- | --- |
| `cellar::Sealer` | rancher, barn (per-user tokens, porch's edge sealing), pond and ranger (backups), farmhand | Seals a value to a public key, bound to its name |
| `cellar::Cellar` | barn, permit, barker | Holds sealed secrets and the unwrap key; opens a secret at the moment of use |
| `cellar::Vault` | the same | Loads the private key once, through an adapter |
| `cellar` (CLI) | farmhand, operators | `keygen`, `seal`, `verify`, `reseal`, `fingerprint-pub` |

## Names

```cpp
namespace homestead::cellar {

enum class Scope : uint8_t { system = 1, tenant = 2, user = 3 };

struct Name {
  Scope scope;
  std::string owner;   // "" for system, the tenant name, or the user id as a string
  std::string name;    // [a-z][a-z0-9_]* segments joined by '/', like "dkim/s1"
  static Result<Name> parse(std::string_view);   // "tenant/bobs_plumbing/hevy_api_key"
  std::string str() const;
};

}
```

Parsing refuses rather than repairs: an empty owner for a tenant or user scope, a `..`, an upper-case letter, or a segment over 63 characters is an error that names the rule.

## Sealing

```cpp
namespace homestead::cellar {

struct Binding {
  Name name;
  uint32_t version;                    // the secret's version, from rancher
  std::vector<std::string> for_;       // services that may open it: {"barn"}, sorted
};

struct PublicKey {                     // an X25519 public key and its id
  std::array<uint8_t, 32> bytes;
  std::array<uint8_t, 8> id;           // the first 8 bytes of SHA-256(bytes)
};

class Sealer {
 public:
  explicit Sealer(PublicKey wrap);
  // Seals plaintext bound to binding. The plaintext is read once and never copied.
  Result<std::vector<uint8_t>> seal(const Binding&, std::span<const uint8_t> plaintext) const;
  // Reads the envelope's header without opening it: key id, binding, sizes.
  static Result<Header> inspect(std::span<const uint8_t> sealed);
};

}
```

`Sealer` is thread-safe and holds nothing secret. `inspect` lets rancher check that a sealed value it was handed (Q1 in [the idea](IDEA.md#open-questions)) names what the caller says it does, without opening it.

## Opening

```cpp
namespace homestead::cellar {

class Opened {                         // a plaintext in locked, zeroed memory
 public:
  Opened(Opened&&) noexcept;
  Opened(const Opened&) = delete;
  ~Opened();                           // OPENSSL_cleanse, then the secure heap frees it
  std::span<const uint8_t> bytes() const;
  std::string_view text() const;       // for keys that are text; no operator<<, no fmt
};

struct Purpose { std::string_view what; };   // "hound:hevy", "barker:dkim", "permit:sign"

class Cellar {
 public:
  Cellar(Vault&, std::string service, AuditSink&, Clock&);

  // Secrets arrive from rancher's config log; cellar keeps the sealed bytes by name and version.
  void put(Name, uint32_t version, std::vector<uint8_t> sealed);
  void drop(Name, uint32_t version);

  // Opens the newest version of a secret for one use. Errors: not_found (missing),
  // refused (this service isn't in `for`), invalid (envelope or binding mismatch).
  Result<Opened> open(const Name&, Purpose);

  // Opens a sealed value held elsewhere (pond's _credentials), checking it against binding.
  Result<Opened> open_sealed(std::span<const uint8_t> sealed, const Binding&, Purpose);

  // Metadata only: names, versions, key ids. Never values.
  std::vector<Header> list() const;
};

}
```

- **`open` is the only way to plaintext**, and it always counts. The count goes to `AuditSink`, which the service's system wires to its `common_log` audit stream; cellar aggregates per name, purpose, and minute.
- **Errors carry problem codes**: `cellar.missing`, `cellar.not_for_service`, `cellar.binding_mismatch`, `cellar.unknown_key`, `cellar.open_failed`. The caller reports them, with the route or job that needed the secret.
- **A rejected secret** (a third party answered 401 or 403 to a call made with it) is reported by the caller as `cellar.rejected`, with the same name, so the urgent request in rancher is one record however it failed.
- **Thread-safety.** `Cellar` is safe to call from any thread. `Opened` belongs to one thread.

## The sealed envelope

Every sealed value is the same bytes, whether it travels in rancher's config log, sits in pond's `_credentials`, or wraps a backup:

```
offset  size  field
0       4     magic "HSEC"
4       1     format version (1)
5       1     suite (1 = HPKE base, DHKEM(X25519, HKDF-SHA256), HKDF-SHA256, AES-256-GCM)
6       8     wrap key id (first 8 bytes of SHA-256 of the recipient public key)
14      2     binding length B (u16, little-endian)
16      B     binding: FlatBuffer { scope, owner, name, version, for: [string] }
16+B    32    HPKE encapsulated key (enc)
48+B    4     ciphertext length C (u32, little-endian)
52+B    C     AEAD ciphertext and 16-byte tag
```

- **HPKE `info`** is `"homestead-cellar/1"`. **AEAD associated data** is every byte before the encapsulated key: magic, version, suite, key id, and binding. Changing any of them makes the open fail.
- **Large values** (backups) use HPKE to seal a fresh 32-byte content key, then stream the payload in 1 MiB AES-256-GCM chunks with the chunk index and a final flag in each chunk's associated data, so truncation and reordering are detected. This is suite 2, the same header, a different body.
- **New suites only add.** A reader refuses an unknown suite or version by name; nothing is ever silently downgraded.

## The vault

```cpp
namespace homestead::cellar {

struct VaultConfig {
  std::string adapter = "file";        // "file", "systemd"; later "tpm", "kms"
  std::string path;                    // file: the key file; systemd: the credential name
  std::vector<std::string> previous;   // during rotation: the old keys, opened too
};

class Vault {
 public:
  static Result<std::unique_ptr<Vault>> load(const VaultConfig&);  // once, at start
  std::span<const PublicKey> keys() const;                         // current first
  // Internal to cellar_open: performs the X25519 step without exposing the private key.
};

}
```

| adapter | load does | refuses to start when |
| --- | --- | --- |
| `file` | Reads a 32-byte X25519 private key into the secure heap, then forgets the path | The file isn't 0400, isn't owned by the process's user, or is inside the service's data directory |
| `systemd` | Reads `$CREDENTIALS_DIRECTORY/$name`, decrypted by systemd | The variable or the credential is missing |
| `tpm`, `kms` | Later. Unwrap once at start. | The device or the KMS call fails |

A SIGHUP reloads the vault: new keys are added, and keys no longer configured are cleansed and dropped. A reload that fails keeps the old keys and raises a problem.

## The command line

`cellar` is the operator's and farmhand's tool. It never prints a private key or a plaintext to a terminal.

| command | does |
| --- | --- |
| `cellar keygen --out DIR` | Makes a wrap key pair: `wrap.key` (0400) and `wrap.pub`, and prints the public key's id. |
| `cellar keygen --backup --out DIR` | The same for the offline backup key pair. |
| `cellar seal --pub wrap.pub --name NAME --version N --for barn,barker < value` | Seals stdin and writes the envelope to stdout. For bootstrapping before rancher runs. |
| `cellar inspect < sealed` | Prints the header: key id, binding, sizes. Never opens. |
| `cellar verify --key wrap.key < sealed` | Opens in memory and answers ok or why not, without printing the value. |
| `cellar reseal --old old.key --new new.pub` | Reads envelopes as JSON lines on stdin, writes re-sealed ones on stdout. What farmhand runs during rotation. |
| `cellar open-backup --key backup.key --in FILE --out FILE` | Decrypts a sealed backup to a file, for restores. |

Exit codes: 0 ok, 1 refused (an envelope, binding, or key mismatch), 2 bad arguments, 3 an I/O failure.

## Fakes

`cellar_fake` gives tests a `Vault` built from a fixed test key, and a `Cellar` that records every open, so a test can assert that hound opened `tenant/t1/hevy_api_key` once for `hound:hevy` and that no other secret was touched.

## The invariants

- **Seal and open are separate targets.** A package that seals can't open, and rancher only seals.
- **`open` is the only path to plaintext, and every open is counted.**
- **Plaintext can't be copied, printed, or logged** through any type cellar returns.
- **Every byte before the ciphertext is authenticated.** The binding and the key id can't be changed.
- **Formats only add.** Unknown versions and suites are refused by name.
- **The CLI never prints a private key or a value.**
