# SentinelAuth

**A hardened authentication & abuse-prevention stack — paired with the adversarial
test harness that proves every control holds.**

Most auth code is a happy-path login form. SentinelAuth is the other 90%: the
cryptography, key management, token lifecycle, device binding, and rate limiting
that stand between an attacker and an account — and each control is paired with
an attack that *tries to break it and fails in CI*.

Built in **Python (FastAPI)** with a wire-compatible **Java** client SDK.

> **About this repository.** This is a public write-up of an independently
> authored personal project. The design draws on general, well-reviewed security
> patterns; it shares no code with any employer or third-party system and
> contains no secrets. Source is kept in a private repository — happy to walk
> through it on request.

---

## Why it exists

Authentication is where application security, cryptography, and abuse prevention
meet — which is exactly the surface a payments or platform-security team cares
about most. SentinelAuth is a compact, auditable reference for doing it well, and
a demonstration that each defensive claim is *tested*, not asserted.

## What's inside

| Capability | What it demonstrates |
|------------|----------------------|
| **ECDH session + AES-256-GCM** | Ephemeral P-256 key exchange (forward secrecy), HKDF derivation, AEAD — no plaintext credentials even inside TLS |
| **Envelope encryption + KMS** | DEK/KEK split, cheap key rotation, a provider seam for AWS/GCP KMS — the key-management core |
| **Request signing + replay guard** | HMAC-SHA256 *and* ECDSA (PKI) signing, nonce/timestamp replay defense, constant-time comparison |
| **Token lifecycle** | Short-lived JWTs (algorithm pinned), **rotating** refresh tokens with **reuse detection**, single-use operation tokens, sign-out-everywhere |
| **Device binding + step-up** | Fingerprint-based trusted-device store, challenge on new devices |
| **Rate limiting + lockout** | Token-bucket limiter + fixed-window account lockout |
| **Fail-closed plumbing** | Crash-safe atomic state, corruption quarantine, secrets required in production |
| **Java client SDK** | Request signer + ECDH session, cross-language parity-tested against the Python server |
| **Cloud posture** | Terraform for a least-privilege, auto-rotating KMS key; multi-version GitHub Actions CI |

## The adversarial harness

Six attack playbooks run against a live instance and assert each defense holds.
The run exits non-zero if any control regresses, so it is a **CI gate**:

```
SentinelAuth adversarial harness
==================================
  [HELD ] token_forgery                forged-secret=401, alg-none=401 (want 401/401)
  [HELD ] token_replay                 first=True, replay_blocked=True, tamper_rejected=True
  [HELD ] refresh_reuse                rotated=True, replay_blocked=True, chain_revoked=True
  [HELD ] device_stepup_bypass         login_gave_no_tokens=True, guessed_code_blocked=True
  [HELD ] brute_force_lockout          throttled_after=6 attempts (codes seen: [401, 429])
  [HELD ] cors_credentialed_wildcard   ACAO='none', ACAC='true'
==================================
6/6 controls held.
```

Each playbook is a real attack — JWT forgery (`alg:none` and wrong-secret),
request replay, stolen-refresh reuse, step-up bypass, online brute force, and
permissive-CORS credential theft.

## Cryptographic design (highlights)

- **Forward secrecy by default.** Ephemeral-ephemeral ECDH; a static server key
  was deliberately rejected because it collapses forward secrecy.
- **AEAD only.** AES-256-GCM with a random 96-bit nonce per message; no
  deterministic IVs.
- **Right hash for the job.** scrypt (memory-hard, salted) for password storage;
  HMAC-SHA256 for message integrity; constant-time comparisons throughout.
- **Keys never leave the KMS.** Envelope encryption means the master key wraps
  short-lived data keys; rotation re-wraps keys instead of re-encrypting data.
- **Algorithm pinning.** JWTs are verified with the algorithm fixed server-side,
  closing `alg:none` and RS→HS confusion attacks.

## Tech stack

`Python` · `FastAPI` · `cryptography` · `PyJWT` · `Java (JCE)` ·
`GitHub Actions` · `Terraform` · `AWS/GCP KMS (provider seam)`

## Engineering practices

- **Prove it, don't assert it** — every control has an adversarial test that fails
  if the control regresses.
- **Fail closed** — ambiguity, corruption, and missing secrets deny rather than
  allow.
- **One safe path** — the insecure option (unauthenticated encryption, unpinned
  JWT alg, non-constant-time compare) isn't reachable from the public API.
- **CI across Python 3.10–3.12**, lint, unit tests, the adversarial harness, and
  the Java client build all gate every change.

---

*Built by Sandeep Namini. Source available on request.*
