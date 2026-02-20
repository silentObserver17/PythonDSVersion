# Private Self-Hosted Password Manager Plan (Low Cost + High Security)

## 1. Goals and Constraints

- **Primary goal:** secure password manager for you + family with private self-hosting.
- **Secondary goal:** keep deployment and maintenance cost as low as possible.
- **Security target:** near zero-knowledge architecture (server should not learn vault plaintext).
- **Scale target:** 2-10 users, light traffic, high reliability over high throughput.
- **Budget target:** keep total hosting close to **<$10/month** in MVP.
- **Policy decision:** account recovery is allowed, but must preserve zero-knowledge as much as possible.

---

## 2. Product Scope (MVP -> Mature)

## MVP Features (must have)

1. User accounts and login.
2. Vault items:
   - website/app login
   - secure notes
   - card/bank metadata (no raw card PAN if possible)
3. Item CRUD with folders/tags.
4. Client-side encryption before data upload.
5. Password generator (strong, policy based).
6. Family sharing (basic shared vault) with fallback to phase 2 only if complexity blocks MVP timeline.
7. TOTP-based 2FA for account login.
8. Encrypted export + import.
9. Secure device/session management.
10. Backup + restore workflow.
11. Account recovery flow (recovery key + secure reset workflow).

## v2 Features (nice to have)

1. Mobile clients (iOS/Android).
2. Expanded browser extension features and autofill hardening.
3. Passkeys (WebAuthn) for login unlock flow.
4. Breach password check integration (k-anonymity model).
5. Audit log with privacy-safe event metadata.

---

## 3. Recommended Architecture

## High-level

- **Client apps (Web first, then mobile):** perform key derivation + encryption locally.
- **API server (Go):** auth, storage orchestration, sharing logic, policy.
- **Database (PostgreSQL):** encrypted blobs + metadata.
- **Object storage (optional):** encrypted file attachments.
- **Reverse proxy (Caddy or Nginx):** TLS termination + routing.

## Zero-knowledge model

- Master password is entered on client.
- Client derives encryption key from master password.
- Server stores only encrypted vault payloads + salts/params + non-sensitive metadata.
- Server should not be able to decrypt user vault data at rest.

---

## 4. Language and Stack Choice

## Backend: **Go** (recommended)

Why Go fits your case:

1. Good memory/runtime profile for small VPS.
2. Simple static binary deployment.
3. Strong standard library + mature crypto libraries.
4. Good concurrency without heavy operational complexity.

Reality check:

- Go is generally lower memory than Node for API services, but exact usage depends on your framework, ORM, caches, and query design.
- For your scale, Go is an excellent cost-performance choice.

## Frontend

- **Web app:** TypeScript + React (or Svelte if you prefer lighter stack).
- Keep logic simple and place crypto operations in a dedicated client crypto module.

## Database

- PostgreSQL (best balance of reliability, cost, and tooling).

## Deployment

- Docker Compose on one small VPS first.
- Later optional split: API + DB separate hosts.

---

## 5. Detailed Crypto and Hashing Plan

> Important: this is engineering guidance, not legal/compliance advice.

## 5.1 Key derivation from master password (client-side)

- Use **Argon2id** for deriving the main vault key.
- Store per-user random salt and Argon2 parameters with account metadata.

Suggested starting parameters for modern consumer devices:

- memory: 64-128 MB
- iterations: 3
- parallelism: 1-4
- output key length: 32 bytes

Tuning rule:

- target ~250ms to ~1000ms unlock time on average family devices.
- prioritize higher memory over only increasing iterations.

Why Argon2id:

- better resistance to GPU/ASIC cracking than PBKDF2/bcrypt for password-derived keys.

## 5.2 Vault encryption (data at rest)

Use one of these AEAD ciphers:

1. **XChaCha20-Poly1305** (excellent nonce safety margin, modern choice).
2. **AES-256-GCM** (also strong if nonce handling is perfect).

Recommendation for your project: **XChaCha20-Poly1305** for simpler nonce safety in distributed/client-heavy contexts.

Encryption model:

- Generate random Data Encryption Key (DEK) per vault item or per vault section.
- Encrypt item plaintext with DEK.
- Wrap DEK with a Key Encryption Key (KEK) derived from master password/root key.
- Store only encrypted DEK + ciphertext + nonce + versioned algorithm metadata.

## 5.3 Password hashing for server authentication

If you keep separate server-side auth secret (recommended), hash using:

- **Argon2id** (preferred)

Fallback only if needed for compatibility:

- bcrypt (acceptable, but less memory-hard)
- PBKDF2 (legacy compatibility only)

Store:

- algorithm id
- salt
- cost parameters
- hash

Never store plaintext passwords, never reversible encryption for login secrets.

## 5.4 Sharing between family members

For secure item sharing:

- Each user has asymmetric key pair:
  - **X25519** for key agreement/envelope encryption
  - **Ed25519** for signatures (optional but useful for integrity/authorship)
- Shared item DEK is encrypted for each recipient using recipient public key.
- Re-sharing does not require re-encrypting whole vault ciphertext.

## 5.5 Integrity and authenticity

- Rely on AEAD authentication tags for encrypted blobs.
- Use digital signatures for critical operations (shared vault mutation, invitation acceptance) if you want stronger tamper attribution.

## 5.6 Transport security

- Enforce HTTPS only (TLS 1.2+, prefer TLS 1.3).
- HSTS enabled.
- Secure cookies (`HttpOnly`, `Secure`, `SameSite=Strict/Lax` as appropriate).

## 5.7 Token/session design

- Short-lived access tokens (5-15 min).
- Rotating refresh tokens.
- Device-bound session records with revocation endpoint.
- Hash refresh tokens at rest with SHA-256 + server secret pepper (or use random opaque IDs stored hashed).

## 5.8 Sensitive metadata minimization

Even in zero-knowledge systems, metadata leaks are common.

Minimize stored plaintext metadata:

- avoid plaintext URL path/query where possible
- avoid plaintext usernames if possible
- optionally encrypt item titles; keep searchable blind index if required

## 5.9 Search without decrypting server-side

If you need server-side search on encrypted fields:

- use blind indexes: `HMAC-SHA-256(normalized_field, index_key)`
- this leaks equality patterns; document this tradeoff.

## 5.10 Backup encryption

- Database backups must be encrypted independently (age/GPG or storage-level encryption).
- Keep 3-2-1 style backup policy if feasible.
- Test restore quarterly.

## 5.11 Account recovery model (enabled)

Recommended secure recovery design:

- Generate a high-entropy **Recovery Key** at account creation (client-side).
- Show it once to user and require offline storage (print/save in secure place).
- Use recovery key + second factor (email OTP or TOTP) to authorize reset flow.
- On successful recovery, rotate all key-wrapping material and invalidate existing sessions/devices.
- Log all recovery events and send security notification to account owner.

Important tradeoff:

- Recovery introduces additional attack surface. Keep the flow heavily rate-limited and auditable.

---

## 6. Cost-Optimized Deployment Plan

## Stage 1 (cheapest practical)

- One VPS (2 vCPU, 2 GB RAM) running:
  - reverse proxy
  - Go API
  - PostgreSQL
  - backup agent
- Typical budget host: low monthly tier targeting **$6-$10/month**.
- Avoid paid managed DB/services in MVP to stay under budget.
- Pros: cheapest and simplest.
- Cons: single-point-of-failure.

## Stage 2 (still affordable, safer)

- Separate managed Postgres or second small DB node.
- Nightly encrypted off-site backups.
- Better uptime and recovery posture.

## Stage 3 (optional later)

- Add object storage, monitoring stack, multi-region backups.

---

## 7. Security Controls Checklist

1. Threat model document (attacker: DB leak, VPS compromise, phishing, stolen laptop).
2. Mandatory 2FA for all family admin accounts.
3. Rate limiting + account lockout/backoff for login attempts.
4. CSRF protections on cookie-auth flows.
5. Strict input validation and output encoding.
6. Dependency and container image scanning.
7. Secret management: env vars + restricted file perms; no secrets in git.
8. Audit logging for auth, key rotation, sharing, export, restore.
9. Regular key rotation strategy for server-side keys.
10. Incident recovery runbook.
11. Recovery abuse protections (cooldown window, alerts, step-up verification).

---

## 8. Proposed Tech Stack (Concrete)

- **Backend:** Go 1.22+
- **HTTP framework:** chi or echo (keep minimal)
- **DB access:** sqlc or pgx (avoid heavy ORM initially)
- **Crypto libs:**
  - `golang.org/x/crypto/argon2`
  - `golang.org/x/crypto/chacha20poly1305`
  - `crypto/aes` + `cipher` (if using AES-GCM)
  - `crypto/ed25519`
- **Database:** PostgreSQL 16+
- **Cache/queue:** none initially (add Redis only if needed)
- **Frontend:** React + TypeScript
- **Deploy:** Docker Compose + Caddy + automatic TLS

---

## 9. Data Model (MVP sketch)

Core tables/collections:

1. users
2. auth_credentials (argon2 hash metadata)
3. user_keys (public keys, encrypted key material)
4. vault_items (ciphertext blob, nonce, algorithm version, metadata)
5. vault_shares (recipient mapping + wrapped DEKs)
6. sessions (hashed refresh token, device info, expiry)
7. audit_events (privacy-safe logs)
8. backups registry

---

## 10. Implementation Roadmap

## Phase 1: Foundations

1. Threat model + crypto spec doc.
2. Auth flow with Argon2id password hashing + TOTP.
3. Basic vault CRUD with encrypted blobs.
4. Local deployment with Docker Compose.
5. Web app only (no mobile) for MVP.

## Phase 2: Family sharing

1. User public/private key system.
2. Share/unshare item workflows.
3. Key rotation and revocation flows.

Note: family sharing is intended for MVP, but this phase remains the fallback if implementation complexity is too high.

## Phase 3: Hardening

1. Backups + restore tests.
2. Monitoring + alerting.
3. Pen test checklist + dependency audits.

---

## 11. Practical Decisions for Your Specific Question

- Yes, **Go** is a strong choice for low memory and low-cost hosting in this project.
- Best overall key derivation/hash choice today: **Argon2id**.
- Best practical vault encryption choice: **XChaCha20-Poly1305** (or AES-256-GCM if your team has stronger AES expertise).
- Keep encryption client-side to maximize privacy even if server is compromised.
- Start with **web app first**, then browser extension, then mobile in v2.
- Account recovery is enabled via recovery key + step-up verification.
- Deployment target is **<$10/month**, so single VPS architecture is mandatory for MVP.

---

## 12. Locked Decisions (Confirmed)

1. Platform strategy: web-first for MVP; extension support added after core web flows are stable; mobile in v2.
2. Sharing strategy: family sharing should be included in MVP, with phase-2 fallback if complexity is too high.
3. Recovery policy: account recovery is supported with strict security controls.
4. Cost limit: infrastructure should stay under **$10/month**.

---

## 13. Immediate Build Plan (Next 3-4 Weeks)

1. Week 1: scaffold Go API + PostgreSQL schema + Argon2id auth + TOTP.
2. Week 1: implement client-side crypto module (Argon2id KDF + XChaCha20-Poly1305 item encryption).
3. Week 2: build web vault CRUD and session/device management.
4. Week 2: implement account recovery flow (recovery key generation, verification, secure key/session rotation).
5. Week 2-3: implement family sharing MVP (X25519 wrapped DEKs) or defer to phase 2 if unstable.
6. Week 3: Docker Compose deployment, HTTPS, encrypted backup, restore dry run.
7. Week 4: add initial browser extension flows (read vault, manual fill) after web stability.