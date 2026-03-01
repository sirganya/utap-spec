# UTAP v0.1.1 Security Delta (Applied Change Map)

This document captures the applied v0.1.1 security changes to `UTAP-v0.1.md`, with minimal explanatory rewrites and a single centralized profile exception description.

---

## Global metadata

- `Version` → `0.1.1`
- `Document ID` → `UTAP-SPEC-0.1.1`
- End marker updated to v0.1.1

---

## Section 2 (Terminology)

- **Token Credential** definition updated to include optional `intended_payee` in credential inputs.

---

## Section 5 (Identity / JWT)

- Added `nbf` claim to JWT claim table and example payload.

---

## Section 6 (Token Specification)

### 6.1 Token format

- Added `intended_payee` to token example and field definitions.

### 6.4 Credential construction

- HMAC input updated:

```text
credential_input = token_id || owner_aid || amount || currency || timestamp || nonce || intended_payee
```

- Canonicalization rule added: if `intended_payee` is absent, use `""`.
- Mint/verify pseudocode updated for `intended_payee`.
- Versioned verifier behavior clarified:
  - `utap-0.1` → legacy input (no `intended_payee`)
  - `utap-0.1.1` → input includes `intended_payee`

### 6.8 Recipient Binding (`intended_payee`) (new)

- CFP MUST enforce recipient binding on claim/hold/transfer.
- Matching identity MUST come from authenticated JWT context.
- Mismatch MUST return:
  - HTTP `403`
  - `PAYEE_MISMATCH`

---

## Section 7 (URI)

- URI examples updated to `utap_version=0.1.1`.
- Query exposure controls added:
  - Prefer header/body placement when URI alteration is not required.
  - If query is used, MUST redact `utap_credential` from logs/telemetry sinks.
  - MUST prevent outbound leakage via `Referer`.

---

## Section 8 (Protocol Flows)

### 8.2 Minting

- Request/response examples include `intended_payee`.
- Validation rules include `intended_payee` validity check.

### 8.4 Validation

- Clarified presenting PA identity must come from JWT context (not request body).
- Added `PAYEE_MISMATCH` behavior.
- Added validation error-surface hardening note.

### 8.4.1 Atomic Claim (new)

- Added endpoint:
  - `POST /cfp/v1/tokens/claim`
- Claim request fields:
  - `credential`
  - `expected_amount`
  - `expected_purpose`
  - `claim_ttl_seconds`
- Must execute atomically in one serializable transaction:
  1. Credential verification
  2. Token state checks
  3. Policy checks (including `intended_payee`)
  4. Transition to `HELD` bound to authenticated PA
  5. Hold expiry assignment
  6. Audit write
- Concurrent claims must yield exactly one success.
- Losing races must return `CLAIM_CONFLICT` or `TOKEN_STATE_CONFLICT`.
- Expired hold behavior returns `HOLD_EXPIRED`.

### 8.5 Hold / 8.6 Transfer

- Added recipient binding enforcement on hold and transfer (`PAYEE_MISMATCH`).

---

## Section 11 (Idempotency)

### 11.3 Scope clarification

- Idempotency uniqueness now MUST be keyed by:

```text
(agent_id, endpoint, idempotency_key, canonical_request_hash)
```

- Same key+agent+endpoint with different payload hash → `INVALID_IDEMPOTENCY` (400).
- Same key string from different agents MUST NOT collide.

### 11.4 Retry vs conflict behavior

- Clarified retry handling using tuple semantics + token state checks.

---

## Section 12 (Security)

### 12.2 Token credential security

- Core attribute list updated to include `intended_payee`.

### 12.2.1 Compromised PA

- Attack/mitigation text aligned with claim endpoint and recipient binding.

### 12.3 Agent auth security

- JWT validation requirement now explicitly includes `iat`/`nbf`/`exp` bounded leeway.

### 12.7 Rate limiting

- CFP MUST enforce limits on:
  - `/cfp/v1/tokens/validate`
  - `/cfp/v1/tokens/claim`
- Required dimensions:
  - per-agent
  - per-source-IP/subnet
  - burst + sustained windows
- Repeated failures SHOULD trigger progressive backoff + security alerts.

### 12.9 Time authority and clock skew (new)

- CFP nodes MUST use authenticated NTP (or equivalent secure sync).
- Inter-node skew MUST NOT exceed 2 seconds.
- JWT `iat/nbf/exp` checks MUST use bounded leeway (recommended ±30s).
- Token expiry MUST use CFP authoritative server time.
- Client clocks MUST NOT be trusted for security decisions.
- Security-critical ordering MUST use authoritative/monotonic server time.

### 12.10 Validation error surface (new)

- CFP SHOULD minimize distinguishability of malformed/unknown/non-transferable validation failures.

### 12.11 Security profiles (new)

- Profiles defined: `STRICT`, `COMPAT`, `HIGH_RISK_PROFILE`.
- `STRICT` (recommended): recipient binding + atomic claim + encrypted exchange + redaction.
- `COMPAT`: migration mode (including tokens without `intended_payee`) with published migration deadline.
- `HIGH_RISK_PROFILE`: non-confidential channels permitted only with:
  - TTL ≤ 120 seconds
  - immediate claim
  - replay anomaly alerting
  - explicit operator risk acknowledgement

> Production vs non-production/channel exception behavior is centralized here.

### 12.12 Optional PoP profile (new)

- Optional `POP_CLAIM` profile: PA signs nonce in claim; CFP verifies signature and freshness.

---

## Section 13 (Error Codes)

Added:

- `INVALID_CLAIM_REQUEST` (400)
- `PAYEE_MISMATCH` (403)
- `CLAIM_CONFLICT` (409)

---

## Section 14 (Transport Binding)

### 14.1 Transport policy

- Preserved original replay-focused explanatory structure.
- Tightened requirement: agent-to-agent credential exchange MUST be encrypted.
- Added credential redaction requirement.

### 14.3 Endpoint summary

- Added `POST /cfp/v1/tokens/claim`.

### 14.5 transport subsections

- Flow references updated to include claim.
- Queue/gRPC/MQTT transport requirements tightened for encrypted transport.

---

## Section 15 (Worked Examples)

- Version literals updated to `0.1.1`.
- Main flow updated to use atomic claim (`mint → claim → transfer → burn`).
- Added `intended_payee` in mint-related examples.

---

## Appendix B (Schemas)

- Schema IDs updated to `v0.1.1`.
- Token schema:
  - `version` const → `utap-0.1.1`
  - add `intended_payee` pattern/description
  - include `PENDING_APPROVAL` in status enum
- Delegation schema version const → `0.1.1`

---

## Compatibility and migration note

- Existing tokens without `intended_payee` remain valid under `COMPAT` during migration.
- Verifiers branch HMAC recomputation by token `version`.
