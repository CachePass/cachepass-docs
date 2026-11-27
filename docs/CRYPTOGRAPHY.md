# CachePass Cryptography Specification — v1

Status: **implemented, not independently audited.** Do not rely on CachePass for
high-value secrets until an external cryptographic review has been completed
(see [ROADMAP.md](ROADMAP.md)).

This document is normative. The implementations are:

| Component | Location | Role |
|---|---|---|
| Crypto engine (Rust → WASM) | `cachepass-webapp/crypto/` | Key derivation, key wrapping, item encryption/decryption |
| TypeScript bindings | `cachepass-webapp/src/lib/crypto/` | Encoding, NFC normalisation, handle lifetime |
| Envelope validation | `cachepass-backend/crates/cachepass-envelope/` | Structural validation only — the server cannot decrypt |

Changing anything in this document requires bumping the relevant version byte
and keeping decryption support for every older version that may exist in
stored data.

## 1. Goals and non-goals

Goals:

1. The server stores only ciphertext for vault contents, folder names and the
   vault key. It never receives the master password, the derived keys or the
   vault key in plaintext.
2. A database dump alone does not reveal vault contents. Offline guessing of the
   master password is slowed by a memory-hard KDF.
3. Ciphertexts are bound to their location: the server cannot swap an item's
   ciphertext into another item, another vault or another object type without
   decryption failing.
4. Every format is versioned so algorithms and parameters can be upgraded.

Non-goals (see [THREAT_MODEL.md](THREAT_MODEL.md) for the full residual risks):

- Protection against a malicious or compromised server **delivering malicious
  JavaScript/WASM**. Browser-based cryptography cannot defend against the party
  that serves the code.
- Protection against a compromised browser, extension or operating system.
- Rollback protection: the server can return an older, authentic ciphertext.
- Hiding metadata: item count, ciphertext sizes (rounded, see §6), timestamps,
  revision numbers and soft-delete state are visible to the server.

## 2. Primitives

All primitives come from audited, widely-used RustCrypto crates. CachePass does
not implement any primitive itself.

| Purpose | Algorithm | Crate |
|---|---|---|
| Password-based KDF | Argon2id v1.3 (RFC 9106) | `argon2` |
| Key derivation / domain separation | HKDF-SHA-256 (RFC 5869) | `hkdf`, `sha2` |
| Authenticated encryption | XChaCha20-Poly1305 (draft-irtf-cfrg-xchacha) | `chacha20poly1305` |
| Randomness | OS CSPRNG via `getrandom` → `crypto.getRandomValues()` in browsers | `getrandom` |
| Memory hygiene | `zeroize` on all key material held in Rust | `zeroize` |

XChaCha20-Poly1305 was chosen over AES-GCM because its 192-bit nonce makes
random nonces safe for an effectively unlimited number of messages per key
(collision probability ≈ 2⁻⁹⁶ after 2⁴⁸ messages), and because its software
implementation is constant-time without hardware AES support.

## 3. Key hierarchy

```
master password (user memory only)
   │  NFC-normalise, UTF-8 encode
   ▼
Argon2id(password, salt, m, t, p) ──► IKM (32 bytes)          [never leaves WASM memory]
   │
   ▼
HKDF-SHA-256(IKM, salt = "", info = "cachepass/v1/kek") ──► KEK (32 bytes)   [never leaves WASM memory]
   │
   │ XChaCha20-Poly1305 wrap (AAD binds vault id + KDF parameters)
   ▼
wrapped vault key  ◄── uploaded to server
   ▲
   │
Vault key / DEK (32 random bytes) ── encrypts ──► items, folder names
```

- The **vault key (DEK)** is generated once per vault from the CSPRNG. It is
  independent of the master password, so a master password change only
  re-wraps the DEK (no item re-encryption).
- The **KEK** exists only for the duration of a wrap/unwrap operation and is
  zeroized immediately afterwards.
- The **account password** (used to log in) is a different secret. It is
  hashed server-side with Argon2id and has no role in vault encryption.
  Resetting it by email can never decrypt a vault.

### 3.1 Master password input

The master password is normalised to Unicode **NFC** (as in the RFC 8265
OpaqueString profile) and UTF-8 encoded before being passed to Argon2id, so the
same password typed on different platforms derives the same key. Normalisation
happens in TypeScript (`String.prototype.normalize("NFC")`); the Rust engine
accepts raw bytes. Length: 12–1024 characters (enforced client-side).

At registration the client refuses a master password equal to the account
password.

### 3.2 KDF parameters

| Parameter | Default | Minimum accepted | Maximum accepted |
|---|---|---|---|
| Algorithm | Argon2id (id `0x01`), version `0x13` | — | — |
| Memory (`m`, KiB) | 65 536 (64 MiB) | 19 456 (19 MiB) | 1 048 576 (1 GiB) |
| Iterations (`t`) | 3 | 2 | 32 |
| Parallelism (`p`) | 1 | 1 | 16 |
| Salt | 16 random bytes, per vault | exactly 16 | exactly 16 |
| Output | 32 bytes | — | — |

Minimums follow the OWASP Password Storage Cheat Sheet (m = 19 MiB, t = 2,
p = 1). The default exceeds them; 64 MiB / t = 3 completes in roughly 0.5–2 s in
current desktop browsers. The client **rejects** parameters outside these
bounds before deriving anything, so a malicious server cannot make the client
burn unbounded memory or silently weaken new envelopes. The parameters are also
bound into the wrap AAD (§5), so tampering with them makes unwrapping fail.

The salt is generated by the client and is not secret. It is only returned to
an authenticated session.

## 4. Envelope format (v1)

Every ciphertext is a byte string, transported as **base64url without
padding** (RFC 4648 §5):

```
offset  length  field
0       1       version      = 0x01
1       1       algorithm    = 0x01  (XChaCha20-Poly1305)
2       24      nonce        random, from the CSPRNG
26      n + 16  ciphertext ‖ Poly1305 tag (16 bytes)
```

Minimum length is 42 bytes (empty plaintext). Parsers must reject: an unknown
version, an unknown algorithm, input shorter than 42 bytes, and anything for
which AEAD verification fails. The two header bytes are authenticated (§5).

### 4.1 Nonces

Each encryption draws a fresh 24-byte nonce from the CSPRNG. Nonces are never
derived from counters, timestamps or identifiers, and no fixed nonce exists in
production code. Fixed nonces appear only in the test vectors, through a
test-only constructor that is not exported to WASM.

## 5. Associated data (AAD)

The AAD binds a ciphertext to its purpose and location. It is built from
length-prefixed fields so that no two distinct inputs can produce the same
byte string:

```
lp(x) = u32_be(len(x)) ‖ x
AAD   = version ‖ algorithm ‖ lp(context) ‖ lp(field₁) ‖ lp(field₂) ‖ …
```

UUIDs are encoded as their 16 raw bytes (RFC 9562 byte order).

| Object | `context` (ASCII) | Fields |
|---|---|---|
| Wrapped vault key | `cachepass/v1/vault-key` | vault id (16), KDF descriptor (29) |
| Vault item | `cachepass/v1/item` | vault id (16), item id (16) |
| Folder | `cachepass/v1/folder` | vault id (16), folder id (16) |

KDF descriptor (29 bytes):
`algorithm_id (u8 = 0x01) ‖ u32_be(m) ‖ u32_be(t) ‖ u32_be(p) ‖ salt (16)`.

Item and folder ids are random UUIDv4 values generated **by the client**
before encryption, so the AAD can be computed before the object exists on the
server. The server enforces uniqueness and ownership.

Consequences:

- Moving a ciphertext between items, between vaults, or from a folder to an
  item makes decryption fail.
- The revision number is deliberately **not** bound. Soft-delete and restore
  change server-side state without re-encryption, and binding the revision
  would not give rollback protection anyway (the server can replay an old
  ciphertext together with its old revision).

## 6. Plaintext encoding and padding

Items and folders are encrypted as a **single JSON document** (all fields,
including the item type, folder assignment and favourite flag, are inside the
ciphertext). Before encryption the JSON bytes are padded with ISO/IEC 7816-4
padding: one `0x80` byte, then `0x00` bytes up to the next multiple of
**64 bytes**. After decryption the padding is checked strictly and removed;
malformed padding is treated as a decryption failure.

Padding hides the exact length of short values (for example the length of a
password); sizes are still visible at 64-byte granularity.

The wrapped vault key is not padded: its plaintext is always exactly 32 bytes,
so a valid wrapped key is exactly 74 bytes.

Size limits (enforced by both client and server):

| Object | Max decoded envelope size |
|---|---|
| Item | 262 144 bytes (256 KiB) |
| Folder | 4 096 bytes |
| Wrapped vault key | exactly 74 bytes |

## 7. Key envelope document

Stored by the server and returned from `GET /api/v1/vault/key-envelope`:

```json
{
  "version": 1,
  "kdf": {
    "algorithm": "argon2id",
    "memory_kib": 65536,
    "iterations": 3,
    "parallelism": 1,
    "salt": "<base64url, 16 bytes>"
  },
  "wrapped_key": "<base64url, 74 bytes>",
  "revision": 1
}
```

The server validates the version, the KDF bounds, the salt length and the
wrapped key structure (§4). It cannot check whether the wrapped key is
correct.

## 8. Key lifecycle

| Event | Behaviour |
|---|---|
| **Registration** | Client generates vault id, salt and DEK; derives the KEK; wraps the DEK; sends `{email, account password, vault id, key envelope}`. The KEK is zeroized. The server creates the user, the vault and the envelope in one transaction. |
| **Unlock** | After login the client fetches the envelope, checks KDF bounds, derives the KEK, unwraps the DEK into a WASM-held `VaultKey` handle and zeroizes the KEK. A tag failure means "wrong master password"; no other information is revealed. |
| **In use** | The DEK stays inside the WASM linear memory of a dedicated Web Worker, behind an opaque handle, and is never exported to JavaScript. Decrypted items exist as JS objects in an in-memory store only. Nothing decrypted is written to `localStorage`, `sessionStorage`, IndexedDB, cookies or URLs. |
| **Lock** | Manual lock (⇧⌘L), idle timeout (default 15 min; options 1–240 min), optional lock when the tab is hidden, logout and session expiry: the `VaultKey` handle is freed (zeroizing the DEK), the crypto Web Worker is terminated (discarding its whole WASM memory), the decrypted store is cleared and a pending clipboard secret is cleared. A page reload also starts locked. |
| **Master password change** | Requires an unlocked vault and re-entry of the current master password. A fresh salt is generated, the current default KDF parameters are applied, the same DEK is re-wrapped and uploaded with the expected envelope revision (optimistic concurrency). |
| **Account password reset** | Changes only the login credential. The vault stays encrypted under the master password. |
| **Lost master password** | The vault **cannot be recovered**. No escrow, recovery key or server-side backdoor exists. The user can delete the account and start over. |
| **Account deletion** | All rows (vault, items, folders, envelope, sessions, tokens) are deleted in one transaction. Database backups taken earlier still contain the ciphertext until they rotate out. |

### 8.1 Known lifecycle limitations

- **No DEK rotation yet.** A master password change does not rotate the DEK.
  Anyone holding an *old* wrapped key (for example from an old database
  backup) together with the *old* master password can still decrypt current
  items. Full re-keying is on the roadmap.
- **JavaScript strings cannot be zeroized.** The master password (briefly) and
  decrypted item fields live as JS strings until garbage-collected. CachePass
  minimises their lifetime and drops references on lock, but cannot guarantee
  erasure from JS heap memory.

## 9. Test vectors

Fixed keys, salts and nonces appear **only** in tests. The vectors below are
checked by:

- the Rust unit tests in `cachepass-webapp/crypto/src/vectors.rs`;
- the WASM integration tests in `cachepass-webapp/src/lib/crypto/*.spec.ts`,
  which recompute them with the independent `@noble/hashes` and
  `@noble/ciphers` JavaScript implementations;
- the envelope validation tests in `cachepass-backend/crates/cachepass-envelope`.

### 9.1 Vector values

Inputs common to all vectors:

```
vault_id   = 0f1e2d3c-4b5a-6978-8796-a5b4c3d2e1f0
item_id    = 11111111-2222-4333-8444-555555555555
folder_id  = 66666666-7777-4888-9999-aaaaaaaaaaaa
password   = "correct horse battery staple" (UTF-8, already NFC)
salt       = 000102030405060708090a0b0c0d0e0f
kdf        = argon2id v0x13, m = 19456 KiB, t = 2, p = 1   (test-sized minimum)
dek        = 202122232425262728292a2b2c2d2e2f303132333435363738393a3b3c3d3e3f
nonce_wrap = 404142434445464748494a4b4c4d4e4f5051525354555657
nonce_item = 606162636465666768696a6b6c6d6e6f7071727374757677
item_json  = {"schema":1,"type":"note","name":"Test"}
```

Outputs (hex):

<!-- VECTORS:BEGIN -->
```
ikm      = 818259b6310026a8e0dbac5d2e6927abcfdb07b32258fac4f61b18b80f929085
kek      = a552e869c0e7842897ba0b8c0dbb118ef465a2ca81475c8c3cda02bcad121902
wrapped  = 0101404142434445464748494a4b4c4d4e4f5051525354555657
           766b9b6de612c45b9b386b77f75a2c279cde4e71d3de75dc8539aa7d05b7ed72
           a49df5ff889f4db69f8362e9338e4ac2                                  (74 bytes)
item_env = 0101606162636465666768696a6b6c6d6e6f7071727374757677
           93053345ed14f6a0883ba42330d556e912ce9637078aad03420e1b945bf8e7a5
           19b3297452c4065fc6b8276ec7b40652d6ee092f5dd26df31e40e8a56bde8090
           19e7bbe60e1d1cce6cb0439c8dbe60cf                                  (106 bytes: 26 + 64 padded + 16)
```
`ikm` is the raw Argon2id output, `kek` the HKDF output. `wrapped` encrypts `dek`
under `kek` with `nonce_wrap` and the vault-key AAD; `item_env` encrypts the
padded `item_json` under `dek` with `nonce_item` and the item AAD.
<!-- VECTORS:END -->

### 9.2 Regenerating

Run `cargo test -- --ignored print_vectors --nocapture` in
`cachepass-webapp/crypto/`. The values are pinned as constants in
`crypto/src/vectors.rs`, and `vectors_are_stable` fails if they change. Any change
is a breaking format change.

## 10. Browser compatibility

| Requirement | Needed for | Support |
|---|---|---|
| WebAssembly (MVP) | Crypto engine | All evergreen browsers since 2017 |
| `crypto.getRandomValues` | CSPRNG | Universal |
| ≥ 64 MiB WASM linear memory growth | Argon2id at default parameters | Desktop and current mobile browsers; very old low-memory devices may fail, in which case unlock reports an error instead of falling back to weaker parameters |
| CSP `'wasm-unsafe-eval'` | Instantiating WASM under a strict CSP without `'unsafe-eval'` | Chrome/Edge 97+, Firefox 102+, Safari 16+ |
| `String.prototype.normalize` | NFC normalisation | Universal |
| Secure context (HTTPS or `localhost`) | `crypto` APIs, `Secure` cookies | Required; CachePass refuses to run on plain HTTP origins other than localhost |

No WebCrypto subtle APIs are used for vault encryption (XChaCha20-Poly1305 and
Argon2id are not available in WebCrypto). The TOTP code generator uses WebCrypto
HMAC-SHA-1, which is universally available in secure contexts.

## 11. Server-side account cryptography (not part of the vault)

| Use | Mechanism |
|---|---|
| Account password storage | Argon2id PHC string, m = 19 456 KiB, t = 2, p = 1 (configurable upwards) |
| Session / verification / reset tokens | 32 random bytes; only SHA-256 hashes are stored |
| Account MFA (TOTP) secret at rest | XChaCha20-Poly1305 with a server key from `CACHEPASS_SECRET_KEY`, AAD = user id |
| MFA recovery codes | 10 random codes, stored as SHA-256 hashes, single use |
| CSRF token | 32 random bytes, double-submit (HttpOnly cookie + header), constant-time compare |

These protect the account, not the vault. A full server compromise exposes
account-level data, but vault plaintext stays protected by the master password
(subject to the malicious-code caveat in §1).
