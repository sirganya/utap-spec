# UTAP v0.1.1 Security Rationale (Non-Normative)

**Status:** Informational / non-normative companion to `docs/SECURITY-DELTA-v0.1.1.md`  
**Audience:** Editors, reviewers, implementers, operators  
**Purpose:** Capture motivation, threat-model context, and trade-offs behind v0.1.1 hardening changes.

---

## 0) Why this companion exists

Normative specs should state required behavior precisely and concisely. The surrounding reasoning (threat framing, trade-offs, alternatives considered, migration concerns) is useful during design/review but can make core spec text harder to consume.

This document keeps that “squishier” context in one place so:

- the normative patch remains crisp,
- reviewer intent is preserved,
- and future editors can understand *why* the requirements exist.

---

## 1) Risk analysis update

Compared with v0.1.0, two classes of risk dominated:

1. **Credential theft/front-running by authenticated-but-unintended participants**
   - The token credential behaves like a bearer artifact in many flows.
   - Any channel/log exposure can turn into unauthorized claim attempts.

2. **Race/timing ambiguity under real load**
   - Validate→hold separation creates windows where multiple contenders can pass checks.
   - Clock differences across services can destabilize expiry and JWT boundary checks.

The v0.1.1 changes prioritize these risks first, with minimal architecture disruption.

---

## 2) Section-by-section motivation

### 2.1 Recipient binding (`intended_payee`)

**Problem addressed:** cross-PA misuse of leaked/forwarded credentials.  
**Why now:** strongest risk reduction per unit of protocol change.

Binding ties credential use to an authenticated identity, so credential possession alone is no longer sufficient (in STRICT profile).

### 2.2 Atomic claim endpoint

**Problem addressed:** TOCTOU race between validation and hold.  
**Why now:** closes concurrency loopholes without inventing new lifecycle states.

Atomic transaction semantics ensure only one claimant wins, with deterministic and auditable outcomes.

### 2.3 Transport confidentiality tightening

**Problem addressed:** interception/leakage in transit.  
**Why now:** bearer-style artifacts demand stronger default transport posture.

Previous flexibility was implementation-friendly but security-fragile. v0.1.1 moves production defaults to encrypted transport, while preserving explicit high-risk opt-in for constrained channels.

### 2.4 Clock drift and time authority policy

**Problem addressed:** false accepts/rejects at expiry and JWT boundaries; skew abuse.  
**Why now:** distributed systems frequently fail at boundary-time consistency.

Authoritative server time + bounded leeway provides deterministic behavior and clearer incident diagnosis.

### 2.5 Validation oracle and enumeration hardening

**Problem addressed:** attacker learning through error shape/timing differences.  
**Why now:** low-cost hardening that reduces reconnaissance value.

Uniform-ish failure behavior and rate limits decrease credential probing efficiency.

### 2.6 Idempotency scope clarification

**Problem addressed:** accidental replay collisions and cross-tenant dedup confusion.  
**Why now:** ambiguity at API edges often becomes security-impacting in multitenant systems.

Including actor + endpoint + payload hash in the key model constrains unsafe reuse.

### 2.7 Query-string exposure controls

**Problem addressed:** credential leakage via logs, referrers, intermediaries, analytics.  
**Why now:** query propagation is common and often invisible to application teams.

This is defensive hygiene to prevent passive leakage paths.

### 2.8 Lifecycle/schema consistency fixes

**Problem addressed:** spec/schema drift causing divergent behavior.  
**Why now:** correctness defects can become security defects when state machines diverge.

Explicit alignment avoids interoperability surprises and mis-validated states.

### 2.9 Optional proof-of-possession profile

**Problem addressed:** residual bearer risk after transport controls.  
**Why optional:** introduces key-management and operational complexity.

PoP is positioned as an incremental hardening profile, with potential promotion in v0.2 after implementation feedback.

### 2.10 Security profile matrix

**Problem addressed:** hidden assumptions and uneven deployment posture.  
**Why now:** operators need explicit, auditable posture choices.

Profiles make trade-offs transparent (STRICT vs COMPAT vs HIGH_RISK_PROFILE) and support phased migration.

### 2.11 Error code additions

**Problem addressed:** ambiguous client handling and weak incident triage.  
**Why now:** clearer cross-implementation behavior with limited additional surface.

Codes remain specific enough for interoperability while allowing response-shaping controls for anti-enumeration.

### 2.12 Minimal patch checklist

**Problem addressed:** implementation omissions in multi-section updates.  
**Why now:** this delta touches endpoints, schemas, lifecycle, and error tables.

Checklist format supports reliable editor/reviewer sign-off.

### 2.13 Backward compatibility and versioning

**Problem addressed:** mixed-verifier ambiguity after canonicalization changes.  
**Why now:** protocol upgrades fail if migration edges are underspecified.

Version bump + verifier branching avoids false verification outcomes and allows staged rollout.

---

## 3) Trade-offs and constraints

- **Security gain vs migration cost:** recipient binding and claim atomicity provide substantial gain with moderate implementation effort.
- **Strict defaults vs ecosystem compatibility:** COMPAT remains available temporarily to avoid breaking existing integrations abruptly.
- **Stronger possession assurance vs operational burden:** PoP is deferred to profile mode pending field experience.

---

## 4) Suggested folding into main spec (editor guidance)

When integrating v0.1.1 into the base UTAP spec:

1. Keep normative requirements in the relevant core sections.
2. Retain concise motivation where it helps interpretation:
   - short context in **Introduction**,
   - threat-focused text in **Security Considerations**,
   - deployment caveats in **Operational Considerations**.
3. Keep extensive deliberative history out of the final normative body.

---

## 5) Suggested publication workflow (RFC-style)

During draft iteration:
- Maintain detailed “changes since” notes (in drafts or repo release notes).

Before final publication:
- Keep the normative text self-contained.
- Keep this rationale as an informational companion (or distill into a short non-normative appendix).
- Remove transient draft-only change logs from the normative artifact if desired.

---

## 6) One-line summary

v0.1.1 prioritizes practical, high-impact defenses against token front-running and timing/race ambiguity, while preserving UTAP’s core architecture and enabling phased adoption.
