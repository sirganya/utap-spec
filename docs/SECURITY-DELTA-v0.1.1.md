# UTAP v0.1.1 Security Hardening Delta (Normative Patch)

**Status:** Proposed  
**Applies to:** `UTAP-v0.1.md` (v0.1.0 draft)  
**Goal:** Close high-impact security gaps while preserving UTAP architecture and flows.  
**Companion rationale (non-normative):** `docs/SECURITY-RATIONALE-v0.1.1.md`

---

## 0. Executive Summary

This delta introduces:

1. **Recipient binding** (`intended_payee`) to prevent cross-PA token theft.
2. **Atomic claim endpoint** to remove validate→hold race windows.
3. **Stronger transport confidentiality requirements** for bearer credentials.
4. **Clock/skew policy** for deterministic expiry/auth behavior.
5. **Validation error-surface hardening** and stronger anti-enumeration throttling.
6. **Idempotency scope clarification** to prevent cross-tenant collisions.
7. **Credential leakage controls** for URI query deployments.
8. **Lifecycle/schema consistency fixes** (`PENDING_APPROVAL` in schema).
9. Optional **PoP profile** for stronger credential possession assurance.
10. **Security profile matrix** for deployment clarity.

---

## 1) Recipient Binding (Critical)

### 1.1 New Token Field

Add a token field:

- `intended_payee` (string, AID or AID wildcard pattern)

If present, it binds a token to one PA identity (or constrained PA set).

### 1.2 Normative Requirements

- The CFP **MUST** enforce `intended_payee` at claim/hold/transfer time.
- If the authenticated PA does not match, CFP **MUST** reject with:
  - HTTP `403`
  - `PAYEE_MISMATCH`
- The PA identity used for matching **MUST** come from CFP-authenticated JWT context, not request body.

### 1.3 Credential Construction Update (Section 6.4)

Update canonical input:

```text
credential_input =
  token_id || owner_aid || amount || currency || timestamp || nonce || intended_payee
```

Canonicalization rule:
- If `intended_payee` absent: use empty string (`""`) in that position.

### 1.4 Schema / Examples

- Add `intended_payee` to token schema and examples.
- Add pattern consistent with AID format used in Section 5.

---

## 2) Atomic Claim Endpoint (Critical)

### 2.1 New Endpoint

Add endpoint to Section 14.3:

- `POST /cfp/v1/tokens/claim`

### 2.2 Request

```json
{
  "credential": "<base64url-hmac>",
  "expected_amount": { "value": "1500.00", "currency": "USD" },
  "expected_purpose": "compute",
  "claim_ttl_seconds": 300
}
```

### 2.3 Behavior

CFP **MUST** perform, atomically, in one serializable transaction:

1. Credential verification
2. Token state checks
3. Policy checks (including `intended_payee`)
4. Transition to `HELD` bound to presenting PA
5. Hold expiry assignment
6. Audit record write

Concurrent claims for same token **MUST** result in exactly one success.

### 2.4 Error Behavior

- Losers in race: `TOKEN_ALREADY_CLAIMED` or `TOKEN_STATE_CONFLICT`
- Hold timeout: `HOLD_EXPIRED`

---

## 3) Transport Confidentiality Tightening (Critical)

### 3.1 Agent-to-Agent Credential Transport

Update Section 14.1/14.5 language:

- Production credential exchange **MUST** use encrypted transport.
- Non-confidential transport allowed only in explicitly configured `HIGH_RISK_PROFILE` mode.

### 3.2 HIGH_RISK_PROFILE Requirements

If using QR/SMS/plain channels, deployment **MUST** enforce all:

- Token TTL ≤ 120 seconds
- Immediate `POST /tokens/claim` on receipt
- Replay anomaly detection and alerting enabled
- Explicit operator acknowledgement of elevated risk

### 3.3 Logging and Telemetry

- Systems **MUST** redact `utap_credential` from logs, traces, metrics labels, crash reports, and analytics sinks.

---

## 4) Clock Drift / Time Authority Policy (High)

### 4.1 CFP Time Requirements

- CFP nodes **MUST** use authenticated NTP (or equivalent secure sync).
- Inter-node skew **MUST NOT** exceed 2 seconds.
- Health checks **MUST** alarm and optionally fail closed if skew exceeds policy.

### 4.2 JWT and Expiry Leeway

- JWT `iat/nbf/exp` validation **MUST** apply bounded leeway (RECOMMENDED ±30s).
- Token expiry (`expires_at`) decisions **MUST** use CFP authoritative server time.
- Client clocks **MUST NOT** be trusted for security decisions.

### 4.3 Timestamp Precision Clarification

- Business timestamps MAY remain second-level ISO8601.
- Security-critical ordering/checks **MUST** use server-side monotonic/authoritative time.

---

## 5) Validation Oracle and Enumeration Hardening (Medium-High)

### 5.1 Error Surface

For credential validation/claim failures, CFP **SHOULD** minimize distinguishability between:

- malformed credential
- unknown credential
- credential for non-transferable token

Where feasible, return uniform error shape and similar latency class.

### 5.2 Rate Limits (Normative)

CFP **MUST** enforce limits on `/tokens/validate` and `/tokens/claim`:

- per-agent
- per-source-IP / subnet
- burst + sustained windows

On repeated failures, CFP **SHOULD** apply progressive backoff and emit security alerts.

---

## 6) Idempotency Key Scope Clarification (Medium)

### 6.1 Canonical Scope

Idempotency uniqueness **MUST** be keyed by:

```text
(agent_id, endpoint, idempotency_key, canonical_request_hash)
```

### 6.2 Conflict Rules

- Same tuple replay: return cached response.
- Same `(agent_id, endpoint, idempotency_key)` but different payload hash:
  - `INVALID_IDEMPOTENCY` (400)
- Same key string from different agents **MUST NOT** collide.

---

## 7) Query-String Exposure Controls (Medium)

### 7.1 Preferred Placement

For HTTP integrations, credential transport in header/body **SHOULD** be preferred over query string where possible.

### 7.2 If Query Is Used

Deployments **MUST** enforce:

- Access log redaction of `utap_credential`
- Reverse proxy/WAF redaction
- Referrer policy preventing outbound credential leak
- No credential ingestion into analytics tooling

---

## 8) Lifecycle / Schema Consistency Fixes (Correctness)

### 8.1 Token Status Enum Alignment

Appendix B token `status` enum **MUST** include `PENDING_APPROVAL` to match Section 6.2.

### 8.2 Cross-Section Consistency

Any new endpoint/state/error introduced by this delta **MUST** be reflected in:

- Section 13 error table
- Section 14 endpoint summary
- Appendix B JSON schemas
- Worked examples

---

## 9) Optional Proof-of-Possession Profile (Optional Hardening)

Introduce optional `POP_CLAIM` profile:

- PA includes signed nonce in `POST /tokens/claim`.
- CFP verifies signature against PA registered public key and nonce freshness.

Purpose: reduce usefulness of stolen bearer credential in transit/log leaks.

**Note:** Optional in v0.1.1; candidate default in v0.2.

---

## 10) Security Profile Matrix (New Section)

Define explicit profiles:

### STRICT (Recommended default)
- Recipient binding enabled
- Atomic claim required
- Encrypted agent-to-agent transport only
- Credential redaction controls enabled
- Short TTL (≤ 10 min; 1–5 min preferred for high-value)

### COMPAT
- Legacy behavior for migration
- No recipient binding (temporary)
- Must publish migration deadline

### HIGH_RISK_PROFILE
- Allows non-confidential transport (QR/SMS/etc.)
- Requires strict TTL, immediate claim, and elevated monitoring

---

## 11) Error Code Additions

Add to Section 13.2:

| HTTP | UTAP Code              | Retry | Description |
|------|------------------------|-------|-------------|
| 403  | `PAYEE_MISMATCH`       | No    | Authenticated PA does not match `intended_payee` |
| 409  | `CLAIM_CONFLICT`       | Yes   | Concurrent claim in progress/just completed |
| 400  | `INVALID_CLAIM_REQUEST`| No    | Claim request malformed or violates constraints |

(Implementations may map `CLAIM_CONFLICT` to existing `TOKEN_STATE_CONFLICT` if desired.)

---

## 12) Minimal Patch Checklist

- [ ] Add `intended_payee` field definitions and schema.
- [ ] Update HMAC input canonicalization text and pseudocode.
- [ ] Add `/cfp/v1/tokens/claim` flow + endpoint summary.
- [ ] Add `PAYEE_MISMATCH` and claim-related errors.
- [ ] Tighten transport text from MAY/SHOULD to production MUSTs.
- [ ] Add NTP/skew and JWT/expiry leeway requirements.
- [ ] Add validation/claim rate-limit requirements.
- [ ] Clarify idempotency scope tuple.
- [ ] Enforce query redaction controls.
- [ ] Fix schema enum to include `PENDING_APPROVAL`.

---

## 13) Rationale

These changes preserve UTAP’s core design (CFP authority, single-use tokens, auditable lifecycle) while closing the most consequential practical risk: **theft/front-running of valid bearer credentials by unintended but authenticated participants**.

---

## 14) Backward Compatibility Notes

- Existing tokens without `intended_payee` remain valid under COMPAT profile.
- New security-sensitive deployments should require STRICT profile.
- Endpoint addition (`/tokens/claim`) is additive and non-breaking.
- HMAC canonicalization change requires versioning (`utap-0.1.1`) to avoid mixed-verifier ambiguity.

---

## 15) Versioning Recommendation

Given credential input changes, bump protocol version identifier from `utap-0.1` to `utap-0.1.1` and require verifiers to branch by token `version` when recomputing HMAC.
