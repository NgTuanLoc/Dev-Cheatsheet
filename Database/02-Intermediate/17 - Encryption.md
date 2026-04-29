---
tags: [database, intermediate, security, postgresql]
aliases: [TLS, pgcrypto, Column Encryption, Data at Rest]
level: Intermediate
---

# Encryption

> **One-liner**: Encrypt in transit (TLS), at rest (disk/filesystem), and selectively at the column level for sensitive fields — and never confuse hashing with encryption.

---

## Quick Reference

| Layer | Mechanism | Postgres specifics |
|-------|-----------|---------------------|
| **In transit** | TLS | `sslmode=require\|verify-ca\|verify-full` in client; `ssl=on` server |
| **At rest (disk)** | LUKS, BitLocker, cloud-provider KMS | transparent to Postgres |
| **At rest (column)** | `pgcrypto` (`pgp_sym_encrypt`), app-side AES | manual; DB doesn't know plaintext |
| **At rest (TDE)** | not built-in to community Postgres; available in EDB / Cybertec / cloud variants | use disk encryption + column encryption |
| **Hashing (one-way)** | `pgcrypto` (`crypt`, `digest`); SCRAM for auth | for passwords, never for "encrypted" data |
| **Backups** | encrypt the dump file (gpg, age, S3 SSE) | see [[15 - Backup and Restore]] |
| **Replication** | TLS + `host` rules in `pg_hba.conf` | encrypts streaming WAL |

| Concept | Meaning |
|---------|---------|
| **Encryption** | reversible with the key |
| **Hashing** | one-way; same input → same output |
| **Salt** | random per-record; prevents rainbow tables |
| **KMS** | external key store (AWS KMS, Azure Key Vault, HashiCorp Vault) |

---

## Core Concept

Three layers, three threats:

- **In transit** — TLS protects against network eavesdroppers and MITM. Trivially enabled, no excuse not to.
- **At rest (disk-level)** — encrypts the storage volume. Protects against stolen disks; doesn't protect against an attacker who's logged into the DB.
- **At rest (column-level)** — encrypts specific values (PII, tokens, secrets) so even a SQL-injecting attacker who reads the column gets ciphertext. Tradeoffs: indexes don't work on ciphertext; you can't filter or sort on encrypted columns; key management is the hard part.

**Hashing ≠ encryption.** A password is **hashed** (one-way). A credit card number you need to charge later is **encrypted** (reversible). Storing passwords reversibly is a security incident waiting to happen.

For column-level encryption in Postgres: prefer **app-side** AES-GCM with keys from a KMS — it keeps plaintext out of the DB entirely. `pgcrypto` is fine but the key material has to live somewhere accessible to the DB session.

---

## Syntax & API

### TLS in transit (client)
```text
# Connection string
Host=db.example.com;Port=5432;Database=shop;Username=app;Password=...;
SSL Mode=VerifyFull;Root Certificate=/etc/ssl/certs/ca.pem;
```

`sslmode` ladder:
- `disable` — no TLS (don't)
- `prefer` — TLS if available, else cleartext (default-ish; risky)
- `require` — TLS required, but no cert validation
- `verify-ca` — TLS + cert chains to known CA
- `verify-full` — TLS + cert + hostname matches → use this in production

### TLS server config (postgresql.conf + certs)
```ini
ssl                          = on
ssl_cert_file                = '/etc/postgresql/server.crt'
ssl_key_file                 = '/etc/postgresql/server.key'
ssl_ca_file                  = '/etc/postgresql/ca.pem'
ssl_min_protocol_version     = 'TLSv1.2'
```

```text
# pg_hba.conf — require TLS
hostssl  shop  app  0.0.0.0/0  scram-sha-256
hostnossl shop app  0.0.0.0/0  reject
```

### Hash a password (pgcrypto)
```sql
CREATE EXTENSION IF NOT EXISTS pgcrypto;

-- Generate a salted bcrypt hash
INSERT INTO users (email, password_hash)
VALUES ('alice@x.com', crypt('s3cret!', gen_salt('bf', 12)));

-- Verify
SELECT id FROM users
WHERE email = 'alice@x.com'
  AND password_hash = crypt('s3cret!', password_hash);
```

### Symmetric encryption with pgcrypto (column-level)
```sql
-- pgp_sym_encrypt with a passphrase (don't hardcode in production!)
INSERT INTO contacts (id, ssn_enc)
VALUES (1, pgp_sym_encrypt('123-45-6789', current_setting('app.crypto_key')));

-- Read back
SELECT id,
       pgp_sym_decrypt(ssn_enc, current_setting('app.crypto_key')) AS ssn
FROM contacts;
```

The session's `app.crypto_key` GUC must be set securely (e.g., from secrets manager via the app layer):
```sql
SET LOCAL app.crypto_key = '<key-from-vault>';
```

### App-side AES-GCM (preferred for column-level)
```csharp
// Pseudocode — real impl uses System.Security.Cryptography.AesGcm
public byte[] Encrypt(byte[] plaintext, byte[] key)
{
    var nonce = RandomNumberGenerator.GetBytes(12);
    var cipher = new byte[plaintext.Length];
    var tag = new byte[16];
    using var aes = new AesGcm(key);
    aes.Encrypt(nonce, plaintext, cipher, tag);
    return [..nonce, ..tag, ..cipher];          // store concatenated
}
```

```sql
-- Schema for app-side encrypted column
CREATE TABLE contacts (
    id      INT PRIMARY KEY,
    ssn_enc BYTEA      -- nonce || tag || ciphertext
);
```

### Backup encryption (off-host)
```bash
pg_dump -d shop -F c | \
    gpg --encrypt --recipient backups@example.com | \
    aws s3 cp - s3://backups/shop/$(date +%F).dump.gpg
```

### Hash for fingerprinting (digest)
```sql
SELECT encode(digest('hello', 'sha256'), 'hex');
-- One-way; useful for content-addressed storage / dedup, not for passwords
```

### Confirm TLS in use
```sql
SELECT pid, ssl, version, cipher
FROM pg_stat_ssl
JOIN pg_stat_activity USING (pid)
WHERE state IS NOT NULL;
```

---

## Common Patterns

```sql
-- Pattern: deterministic encryption when you must search
-- (Use a derivation: HMAC-SHA256 of value with a stable key)
ALTER TABLE users ADD COLUMN email_hash BYTEA;
UPDATE users SET email_hash = digest(email || :pepper, 'sha256');
CREATE INDEX idx_users_email_hash ON users (email_hash);

SELECT * FROM users WHERE email_hash = digest('a@b.com' || :pepper, 'sha256');
-- Tradeoff: deterministic = same plaintext → same ciphertext (vulnerable to frequency analysis)
```

```sql
-- Pattern: separate DB columns for searchable and encrypted fields
-- e.g., last 4 of SSN as plaintext for display; full SSN encrypted
ssn_last4   TEXT,
ssn_full_enc BYTEA
```

```csharp
// Pattern: KMS-backed envelope encryption
// 1. App asks KMS for a Data Encryption Key (DEK), wrapped with a Master Key
// 2. App encrypts column with DEK (AES-GCM)
// 3. App stores wrapped DEK + ciphertext in DB
// On read: ask KMS to unwrap DEK, decrypt with it
// Key rotation: rotate Master Key in KMS; ciphertext columns don't change
```

---

## Gotchas & Tips

- **`sslmode=require` is *not* enough** — it doesn't validate the cert. Use `verify-full` in production.
- **Don't roll your own crypto** — `AES-GCM` from the standard library, never `AES-CBC` without auth, never custom protocols.
- **Hashing passwords with `digest()` directly is wrong** — use `crypt(p, gen_salt('bf'))` or app-side `BCrypt`/`Argon2`. SHA-256 alone is bypassable with GPUs.
- **Encrypted columns can't be indexed for search** — except via deterministic encryption (which leaks frequency) or HMAC fingerprints.
- **Key management is the hard part** — store keys in a KMS, not in the DB itself. If keys live next to ciphertext, you have obfuscation, not encryption.
- **TDE is not in community Postgres** — disk encryption + column encryption is the open-source equivalent. Cloud variants (RDS, Azure Database, Aurora) do TDE for you.
- **Backups inherit the security model** — an unencrypted dump in S3 leaks everything. Encrypt at the source.
- **Rotate keys on cadence and on suspicion** — envelope encryption (DEK + KEK) makes this much cheaper.
- **`pg_stat_ssl` reveals which backends are TLS-encrypted** — cross-check with `pg_stat_activity`.
- **Connection-level encryption only protects the wire** — once the row is in memory or in the WAL, it's plaintext unless the column itself is encrypted.

---

## See Also

- [[16 - Database Security]]
- [[15 - Backup and Restore]]
- [[02 - Replication]]
- [[17 - Cloud Databases]]
