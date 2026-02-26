# UTAP -- URI Token Audit Protocol

**Version:** 0.1.0
**Status:** Draft Specification
**Date:** 2026-02-26
**Author:** Gregory Peter Kavanagh
**Document ID:** UTAP-SPEC-0.1

---

## Intellectual Property Notice

This specification describes methods that may be covered by US Patent
11,455,625 B2 ("Method of electronic payment by means of a uniform resource
identifier (URI)"), granted September 27, 2022, to Gregory Peter Kavanagh.

The patent holder commits to Reasonable and Non-Discriminatory (RAND) licensing
terms for any party implementing this specification in conformance with its
normative requirements. Inquiries regarding licensing should be directed to the
author.

No other intellectual property claims are known to the author at the time of
publication.

---

## Table of Contents

1. [Abstract](#1-abstract)
2. [Terminology and Conformance Keywords](#2-terminology-and-conformance-keywords)
3. [Introduction and Motivation](#3-introduction-and-motivation)
4. [Architecture Overview](#4-architecture-overview)
5. [Identity and Delegation Model](#5-identity-and-delegation-model)
6. [Token Specification](#6-token-specification)
7. [URI Specification](#7-uri-specification)
8. [Protocol Flows](#8-protocol-flows)
9. [Audit Trail Specification](#9-audit-trail-specification)
10. [Budget and Policy Enforcement](#10-budget-and-policy-enforcement)
11. [Idempotency Mechanism](#11-idempotency-mechanism)
12. [Security Considerations](#12-security-considerations)
13. [Error Codes](#13-error-codes)
14. [Transport Binding](#14-transport-binding)
15. [Worked Examples](#15-worked-examples)
16. [References](#16-references)
- [Appendix A: Purpose Category Vocabulary](#appendix-a-purpose-category-vocabulary)
- [Appendix B: JSON Schema Definitions](#appendix-b-json-schema-definitions)
- [Appendix C: Reference Implementation Notes](#appendix-c-reference-implementation-notes)

---

## 1. Abstract

The URI Token Audit Protocol (UTAP) defines an open protocol for transferring
electronic monetary tokens between autonomous software agents by means of
Uniform Resource Identifiers (URIs). A trusted Central Financial Provider (CFP)
issues tokens, validates ownership, prevents double-spending, and maintains a
cryptographically verifiable audit trail.

UTAP is designed for enterprise environments where AI agents transact on behalf
of humans and organizations. The protocol provides delegation chains linking
every agent action to a human principal, purpose-bound tokens declaring
machine-readable intent, hierarchical budget enforcement, and a hash-chained
audit ledger suitable for corporate compliance and financial controls.

---

## 2. Terminology and Conformance Keywords

### 2.1 Conformance Keywords

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in BCP 14 [RFC 2119] [RFC 8174]
when, and only when, they appear in all capitals, as shown here.

### 2.2 Definitions

**Agent**
: An autonomous software entity that performs actions -- including financial
transactions -- on behalf of a Principal. An Agent may be an AI model, a
traditional API integration, a robotic process automation script, or any
software that can issue HTTP requests.

**Principal**
: A human, organization, or Agent that delegates authority to another Agent.
The root Principal in any delegation chain MUST be a human or a legal entity.

**Central Financial Provider (CFP)**
: The trusted third party that issues tokens, maintains accounts, validates
token ownership on request, executes transfers, prevents double-spending,
and maintains the authoritative audit ledger. The CFP is the single source
of truth for all token state.

**Token**
: An electronic monetary instrument, identified by a UUID, representing a
fixed monetary value. Tokens are embedded in URIs for transport between
Agents. Each token is single-use: it is burned (permanently invalidated)
upon successful transfer.

**Requesting Agent (RA)**
: The Agent that initiates a payment -- the buyer. The RA mints tokens from
its account on the CFP and presents them to the Providing Agent.

**Providing Agent (PA)**
: The Agent that receives a payment in exchange for a resource or service --
the seller. The PA validates tokens with the CFP before releasing resources.

**Delegation Chain**
: An ordered sequence of (delegator, delegate) pairs establishing the
authority under which an Agent acts. Each link is cryptographically signed
by the delegator. The chain traces from the acting Agent back to a root
Principal.

**Audit Record**
: A structured log entry capturing a single state transition of a token or
delegation. Each record is hash-linked to its predecessor, forming a
tamper-evident chain. The CFP signs each record.

**Purpose Binding**
: Machine-readable metadata attached to a token at minting time, declaring
the intended use of the funds. Purpose bindings are immutable after minting.

**Budget Scope**
: A hierarchical spending limit structure, expressed as a slash-delimited
path (e.g., `acme/engineering/ml-team`), constraining how much an Agent or
group of Agents may spend within a given period.

**Idempotency Key**
: A client-generated string submitted with a request to ensure that retried
operations produce the same result as the original, preventing accidental
double-spending due to network failures.

---

## 3. Introduction and Motivation

### 3.1 The Problem

Autonomous software agents are increasingly performing tasks that involve
financial transactions: purchasing API access, renting compute resources,
licensing datasets, paying for model inference, and procuring services from
other agents. These transactions are growing in volume and occurring without
direct human involvement at the point of sale.

No existing protocol addresses the intersection of three requirements:

1. **Agent-native transport.** Agents communicate via HTTP. A payment mechanism
   should operate within the same transport layer, not require integration
   with card networks, banking APIs, or blockchain nodes.

2. **Delegated authority with audit trails.** When an agent spends money, the
   organization that authorized the agent needs to know: who spent it, how
   much, on what, under whose authority, and whether the spending was within
   policy. This audit trail must be cryptographically verifiable, not merely
   a log file.

3. **Double-spend prevention without distributed consensus.** Corporate
   payments require deterministic finality. The sender, the receiver, and the
   auditor must all agree on what happened. Blockchain-style probabilistic
   finality and gas fees are unnecessary overhead for transactions between
   known, authenticated agents within or across organizations.

### 3.2 Why Existing Approaches Fall Short

**Credit card / banking APIs** (Stripe, Plaid, ACH): Designed for
human-to-merchant transactions. Require KYC per entity, assume human identity
at the point of sale, and provide no native concept of agent delegation or
machine-readable purpose binding. Settlement latency (hours to days) is
mismatched with agent-speed transactions (milliseconds to seconds).

**Cryptocurrency / blockchain**: Provides decentralized double-spend prevention
but introduces unnecessary complexity for corporate environments where a
trusted central authority already exists (the organization's own financial
controls). Gas fees, wallet management, and probabilistic finality add friction
without benefit when both parties already trust a common provider.

**OAuth / API keys**: Authorize access, not payment. An OAuth token says "this
agent may call this API" but says nothing about value transfer, spending limits,
or financial audit trails.

### 3.3 The Core Insight

US Patent 11,455,625 B2 establishes a method for electronic payment where
tokens are embedded directly in URIs, a trusted third party validates ownership
before resources are released, and tokens are transferred atomically between
accounts upon validation. This mechanism -- URI as payment instrument, server as
validator, token as single-use bearer credential -- maps naturally to
agent-to-agent communication.

UTAP adapts this patented method for the agent economy by adding:

- **Delegation chains** linking every transaction to a human-authorized chain
  of command.
- **Purpose binding** making every token self-documenting: not just "who paid
  whom" but "why."
- **Budget hierarchies** enforcing organizational spending policies at the
  protocol level.
- **A hash-chained audit ledger** providing tamper-evident, cryptographically
  verifiable records of every token lifecycle event.

### 3.4 Design Principles

1. **URI-native.** Tokens travel in URIs. No sideband channels, no separate
   payment sessions. The URI *is* the payment.

2. **Centrally validated.** The CFP is the single source of truth. This is not
   a limitation -- it is a feature. Corporates already have central financial
   controls. UTAP formalizes them into an auditable protocol.

3. **Burn after transfer.** Every token is single-use. There is no "balance" to
   protect, no wallet to secure, no change to make. The token exists, it
   transfers, it is destroyed. This eliminates entire classes of attacks.

4. **Audit by default.** Every state transition produces an audit record. Audit
   is not an afterthought or an optional module. It is structural.

5. **Delegation, not impersonation.** An agent never pretends to be a human. It
   acts *on behalf of* a human, and the delegation chain is always visible to
   the CFP and to auditors.

6. **Settlement-agnostic.** UTAP defines the token lifecycle and audit trail.
   Actual movement of money (ACH, wire, internal ledger, stablecoin) is
   handled by the settlement layer, which is pluggable and outside the scope
   of this specification.

---

## 4. Architecture Overview

### 4.1 The Three Roles

UTAP defines three roles. Every transaction involves all three.

```
 +---------------------+                    +---------------------+
 |  Requesting Agent   |   URI with token   |  Providing Agent    |
 |  (RA / Buyer)       | -----------------> |  (PA / Seller)      |
 |                     |                    |                     |
 |  - Mints tokens     |                    |  - Receives tokens  |
 |  - Embeds in URI    |                    |  - Validates w/ CFP |
 |  - Has delegation   |                    |  - Releases resource|
 |    chain + budget   |                    |  - Confirms burn    |
 +----------+----------+                    +----------+----------+
            |                                          |
            |  mint / query                            |  validate / transfer
            |                                          |
            v                                          v
 +----------------------------------------------------------+
 |              Central Financial Provider (CFP)             |
 |                                                          |
 |  - Issues tokens          - Validates ownership          |
 |  - Manages accounts       - Executes transfers           |
 |  - Enforces budgets       - Burns spent tokens           |
 |  - Maintains audit log    - Signs audit records          |
 |  - Verifies delegations   - Provides audit retrieval API |
 +----------------------------------------------------------+
```

**Requesting Agent (RA):** The agent that wants to pay. It authenticates with
the CFP, mints a token (debiting its budget), and embeds the token in a URI
sent to the Providing Agent.

**Providing Agent (PA):** The agent that provides a resource or service. Upon
receiving a URI containing a token, it asks the CFP to validate the token.
If valid, the PA requests a transfer, delivers the resource, and confirms
the burn.

**Central Financial Provider (CFP):** The trusted ledger. It issues tokens,
validates ownership, prevents double-spending, executes atomic transfers,
enforces budget policies, and maintains the hash-chained audit log. The CFP
is logically centralized. It MAY be deployed as a distributed system
internally, but it presents a single authoritative API to agents.

### 4.2 Trust Model

- The RA and PA do **not** need to trust each other.
- Both the RA and the PA MUST trust the CFP.
- The CFP is the sole arbiter of token validity and ownership.
- The PA MUST NOT release resources until the CFP confirms token validity.
- The CFP MUST NOT transfer a token without explicit instruction from the PA
  (the PA decides whether to accept the payment after validation).

### 4.3 Protocol Layers

UTAP is organized into four layers, each building on the one below:

```
 +--------------------------------------------------+
 |  Layer 4: Audit & Compliance                     |
 |  Hash-chained records, access-controlled retrieval|
 +--------------------------------------------------+
 |  Layer 3: Token Lifecycle                         |
 |  Mint, hold, transfer, burn, expire, revoke      |
 +--------------------------------------------------+
 |  Layer 2: Identity & Delegation                   |
 |  Agent registration, JWT auth, delegation chains  |
 +--------------------------------------------------+
 |  Layer 1: Transport                               |
 |  HTTPS, JSON, URI query parameters               |
 +--------------------------------------------------+
```

**Layer 1 -- Transport:** HTTPS with JSON request/response bodies. Tokens are
embedded in URI query parameters prefixed with `utap_`. See Section 14 for the
full transport binding.

**Layer 2 -- Identity & Delegation:** Agents register with the CFP, present
delegation chains signed by their principals, and authenticate via JWT bearer
tokens. See Section 5.

**Layer 3 -- Token Lifecycle:** Tokens are minted, optionally held (escrowed),
transferred, and burned. Each state transition is atomic and produces an audit
record. See Section 6.

**Layer 4 -- Audit & Compliance:** Every state transition is recorded as a
hash-linked, CFP-signed audit record. Tokens carry the hash of their latest
audit record for offline integrity verification. See Section 9.

### 4.4 Conceptual Mapping

For readers familiar with the reference implementation (Secure File Flow),
the following table maps concepts:

| Secure File Flow            | UTAP                           |
|-----------------------------|--------------------------------|
| Ticket (`ticket_id`)        | Token (`token_id`)             |
| `AVAILABLE`                 | `MINTED`                       |
| `RESERVED`                  | `HELD`                         |
| `CLAIMED` (burned)          | `TRANSFERRED` then `BURNED`    |
| `POST /api/mint`            | Token Issuance (Section 8.2)   |
| `POST /api/claim`           | Payment Presentation (8.4)     |
| `POST /api/mass/ack`        | Acknowledgment & Burn (8.6)    |
| JWT auth (`requireAuth`)    | Agent Authentication (5.1)     |
| `?ticket_id=<UUID>`         | `?utap_token=<UUID>`           |
| Durable Object              | CFP persistence layer          |
| WebSocket notifications     | CFP event notifications        |
| Worker (Hono)               | CFP API server                 |
| QR code / claim URL         | Payment URI                    |

---

## 5. Identity and Delegation Model

### 5.1 Agent Identity

Every agent participating in UTAP MUST have a unique **Agent Identifier
(AID)**, formatted as a URI:

```
utap:agent:<domain>:<local-id>
```

**Examples:**

```
utap:agent:acme.com:purchasing-bot-7
utap:agent:cloudco.com:billing-agent
utap:agent:acme.com:cfo-alice
```

The `<domain>` component identifies the organization that controls the agent.
The `<local-id>` component is a string unique within that domain. The CFP MUST
enforce uniqueness of AIDs across all registered agents.

### 5.2 Agent Authentication

Agents authenticate with the CFP using JWT bearer tokens, presented in the
`Authorization` header of every request:

```
Authorization: Bearer <JWT>
```

The JWT MUST contain the following claims:

| Claim               | Type     | Description                              |
|---------------------|----------|------------------------------------------|
| `sub`               | string   | The Agent's AID                          |
| `iss`               | string   | The CFP's domain                         |
| `iat`               | number   | Issued-at timestamp (Unix seconds)       |
| `exp`               | number   | Expiration timestamp (Unix seconds)      |
| `delegation_chain`  | string[] | Ordered list of AIDs from root to agent  |
| `scopes`            | string[] | Budget scope paths this agent may access |

**Example JWT payload:**

```json
{
  "sub": "utap:agent:acme.com:purchasing-bot-7",
  "iss": "cfp.example.com",
  "iat": 1740000000,
  "exp": 1740086400,
  "delegation_chain": [
    "utap:agent:acme.com:ceo-board",
    "utap:agent:acme.com:cfo-alice",
    "utap:agent:acme.com:purchasing-bot-7"
  ],
  "scopes": [
    "acme/engineering/ml-team"
  ]
}
```

The CFP MUST sign JWTs using RS256 or ES256. JWTs MUST have an expiration of
no more than 24 hours. Agents MUST re-authenticate before their token expires.

### 5.3 Delegation Chains

A delegation chain establishes the authority under which an Agent acts. Each
link is a signed **Delegation Token** -- a JWT issued by the delegator granting
specific capabilities to the delegate.

**Delegation Token format:**

```json
{
  "type": "utap-delegation",
  "version": "0.1",
  "delegator": "utap:agent:acme.com:cfo-alice",
  "delegate": "utap:agent:acme.com:purchasing-bot-7",
  "scopes": ["acme/engineering/ml-team"],
  "constraints": {
    "max_amount_per_tx": "10000.00",
    "max_amount_per_day": "50000.00",
    "allowed_purposes": ["compute", "data-license", "api-access"],
    "can_delegate": false,
    "expires_at": "2026-06-01T00:00:00Z"
  },
  "chain": [
    "utap:agent:acme.com:ceo-board",
    "utap:agent:acme.com:cfo-alice"
  ],
  "iat": 1740000000,
  "exp": 1748736000
}
```

**Rules:**

- The `delegator` MUST sign the delegation token with its own private key.
- The `chain` field MUST contain the full ancestry from the root Principal to
  (but not including) the delegate.
- The `delegate` inherits the *intersection* of the delegator's scopes and the
  scopes listed in the delegation token. A delegator MUST NOT grant scopes it
  does not itself possess.
- If `can_delegate` is `false`, the delegate MUST NOT create further delegation
  tokens.
- Delegation chains SHOULD NOT exceed 5 levels in depth.
- The CFP MUST validate the entire chain on agent registration and MUST reject
  chains with broken signatures, expired links, or scope escalation.

### 5.4 Agent Registration

Before transacting, an Agent MUST register with the CFP by presenting its
delegation chain.

**Request:**

```http
POST /cfp/v1/agents/register HTTP/1.1
Host: cfp.example.com
Content-Type: application/json

{
  "agent_id": "utap:agent:acme.com:purchasing-bot-7",
  "delegation_tokens": [
    "<signed-jwt: ceo-board delegates to cfo-alice>",
    "<signed-jwt: cfo-alice delegates to purchasing-bot-7>"
  ],
  "public_key": {
    "kty": "RSA",
    "n": "...",
    "e": "AQAB"
  }
}
```

**Response (success):**

```http
HTTP/1.1 201 Created
Content-Type: application/json

{
  "agent_id": "utap:agent:acme.com:purchasing-bot-7",
  "auth_token": "<JWT signed by CFP>",
  "token_expires_at": "2026-02-27T10:00:00Z",
  "delegation_chain": [
    "utap:agent:acme.com:ceo-board",
    "utap:agent:acme.com:cfo-alice",
    "utap:agent:acme.com:purchasing-bot-7"
  ],
  "effective_scopes": ["acme/engineering/ml-team"],
  "effective_constraints": {
    "max_amount_per_tx": "10000.00",
    "max_amount_per_day": "50000.00",
    "allowed_purposes": ["compute", "data-license", "api-access"]
  }
}
```

The CFP validates the delegation chain, computes the effective constraints
(intersection of all constraints in the chain), and issues a JWT. The Agent
uses this JWT for all subsequent API calls.

### 5.5 Delegation Revocation

A delegator MAY revoke a delegation at any time:

```http
POST /cfp/v1/delegations/revoke HTTP/1.1
Host: cfp.example.com
Authorization: Bearer <delegator-jwt>
Content-Type: application/json

{
  "delegate": "utap:agent:acme.com:purchasing-bot-7",
  "reason": "Employee departure"
}
```

Upon revocation:

- The CFP MUST immediately invalidate the delegate's JWT.
- The CFP MUST reject any pending operations by the delegate.
- All agents further down the chain (sub-delegates) are also revoked.
- A `DELEGATION_REVOKED` audit record is created.

---

## 6. Token Specification

### 6.1 Token Format

A token is a JSON object with the following fields:

```json
{
  "token_id": "550e8400-e29b-41d4-a716-446655440000",
  "version": "utap-0.1",
  "issuer": "cfp.example.com",
  "amount": {
    "value": "1500.00",
    "currency": "USD"
  },
  "owner": "utap:agent:acme.com:purchasing-bot-7",
  "status": "MINTED",
  "purpose": {
    "category": "compute",
    "description": "GPU rental for ML training job #4821",
    "reference": "PO-2026-0042"
  },
  "budget_scope": "acme/engineering/ml-team",
  "delegation_chain_hash": "sha256:a1b2c3d4e5f6...",
  "audit_chain_hash": "sha256:d4e5f6a7b8c9...",
  "idempotency_key": "purchasing-bot-7-1740000000-x7f3a",
  "created_at": "2026-02-26T10:00:00Z",
  "expires_at": "2026-02-26T11:00:00Z",
  "metadata": {}
}
```

**Field definitions:**

| Field                   | Type   | Req. | Description                                          |
|-------------------------|--------|------|------------------------------------------------------|
| `token_id`              | string | MUST | UUID v4 identifier, unique across all tokens         |
| `version`               | string | MUST | Protocol version (`utap-0.1`)                        |
| `issuer`                | string | MUST | Domain of the issuing CFP                            |
| `amount.value`          | string | MUST | Decimal string, no floating point (e.g., `"1500.00"`)|
| `amount.currency`       | string | MUST | ISO 4217 currency code (e.g., `"USD"`)               |
| `owner`                 | string | MUST | AID of the current token owner                       |
| `status`                | string | MUST | Current lifecycle state (see 6.2)                    |
| `purpose.category`      | string | MUST | Category from controlled vocabulary (Appendix A)     |
| `purpose.description`   | string | SHOULD| Human-readable description of intended use           |
| `purpose.reference`     | string | MAY  | External identifier (PO number, invoice, etc.)       |
| `budget_scope`          | string | MUST | Budget path this token is charged against            |
| `delegation_chain_hash` | string | MUST | SHA-256 hash of the minting agent's delegation chain |
| `audit_chain_hash`      | string | MUST | SHA-256 hash of the most recent audit record         |
| `idempotency_key`       | string | MUST | Client-generated key for replay protection           |
| `created_at`            | string | MUST | ISO 8601 creation timestamp                          |
| `expires_at`            | string | MUST | ISO 8601 expiration timestamp                        |
| `metadata`              | object | MAY  | Arbitrary key-value pairs for application use        |

### 6.2 Token Lifecycle States

A token progresses through the following states:

```
                  +--------+
                  | MINTED |
                  +---+----+
                      |
         +------------+------------+
         |            |            |
         v            v            v
      +------+   +---------+  +---------+
      | HELD |   | EXPIRED |  | REVOKED |
      +--+---+   +---------+  +---------+
         |
    +----+----+
    |         |
    v         v
+-------------+  +---------+
| TRANSFERRED |  | REVOKED |
+------+------+  +---------+
       |
       v
  +--------+
  | BURNED |
  +--------+
```

**MINTED:** The token has been created and assigned to the RA. The RA's budget
has been debited. The token is ready to be used in a payment URI.

**HELD:** The token has been presented to a PA and the PA has requested a hold
(escrow). The token cannot be used in another transaction while held. A hold
MUST expire after a configurable timeout (default: 5 minutes). If the hold
expires, the token returns to MINTED.

**TRANSFERRED:** Ownership has been atomically transferred from the RA to the
PA. The PA now owns the token. This state exists to allow the PA to confirm
delivery before the token is burned.

**BURNED:** The token has been permanently consumed. It cannot be reused,
transferred, or refunded through the same token. (Refunds require minting a
new token in the reverse direction.) This is a terminal state.

**EXPIRED:** The token reached its `expires_at` timestamp without being used.
The RA's budget is credited back. This is a terminal state.

**REVOKED:** The token was explicitly cancelled by the owner or by the CFP
(e.g., due to delegation revocation). The RA's budget is credited back. This
is a terminal state.

### 6.3 Token ID Format

Token IDs MUST be UUID v4 strings matching the following pattern:

```
/^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$/i
```

The CFP MUST generate token IDs using a cryptographically secure random
number generator.

### 6.4 Amount Representation

Amounts MUST be represented as decimal strings, not as floating-point numbers.
This prevents rounding errors inherent in IEEE 754 arithmetic.

**Valid:** `"1500.00"`, `"0.01"`, `"99999.99"`
**Invalid:** `1500.00` (number), `"1,500.00"` (comma), `"$1500"` (symbol)

The `currency` field MUST be a valid ISO 4217 alphabetic code. Implementations
MUST support at least `USD`, `EUR`, and `GBP`. The CFP MAY support additional
currencies.

### 6.5 Token Expiration

Every token MUST have an `expires_at` field set at minting time. The default
TTL SHOULD be 1 hour. The CFP MUST reject any operation on an expired token
with a `TOKEN_EXPIRED` error (HTTP 410).

The CFP SHOULD periodically evict expired tokens from active storage. Expired
tokens MUST remain in the audit log.

### 6.6 Purpose Binding

The `purpose` object is set at minting time and is **immutable** -- it MUST NOT
be modified after the token is created. This ensures that audit records
accurately reflect the original intent of the transaction.

The `purpose.category` field MUST contain a value from the controlled
vocabulary defined in Appendix A. The `purpose.description` field SHOULD
contain a human-readable explanation suitable for display in audit reports.

---

## 7. URI Specification

### 7.1 URI Format

UTAP tokens are transported as query parameters in HTTPS URIs. All
UTAP-specific parameters are prefixed with `utap_` to avoid collision with
other query parameters.

**Payment Request URI** (PA to RA -- "I need payment"):

```
https://{cfp-host}/pay?utap_version=0.1&utap_amount={amount}&utap_currency={currency}&utap_purpose={category}&utap_callback={callback_url}
```

**Payment URI** (RA to PA -- "Here is my payment"):

```
https://{cfp-host}/pay?utap_token={token_id}&utap_version=0.1
```

### 7.2 Query Parameters

| Parameter          | Req.     | Description                                      |
|--------------------|----------|--------------------------------------------------|
| `utap_token`       | MUST     | Token UUID, or the literal string `NEW` in a payment request |
| `utap_version`     | MUST     | Protocol version (e.g., `0.1`)                   |
| `utap_amount`      | MUST*    | Decimal string amount (*REQUIRED in requests)     |
| `utap_currency`    | MUST*    | ISO 4217 code (*REQUIRED in requests)             |
| `utap_purpose`     | SHOULD   | Purpose category from Appendix A vocabulary       |
| `utap_callback`    | MAY      | URL for async status notifications                |
| `utap_expires`     | MAY      | ISO 8601 expiration for the payment request       |
| `utap_idempotency` | MAY      | Idempotency key for the transaction               |
| `utap_ref`         | MAY      | External reference (PO number, invoice ID, etc.)  |

### 7.3 Payment Request Flow (URI Alteration)

The core mechanism of UTAP -- derived from the patent -- is **URI alteration**:
the PA constructs a URI requesting payment, and the RA alters it by replacing
the request parameters with a minted token.

**Step 1: PA constructs a payment request.**

The PA generates a URI indicating that payment is required:

```
https://cfp.example.com/pay?utap_token=NEW&utap_version=0.1&utap_amount=1500.00&utap_currency=USD&utap_purpose=compute&utap_callback=https://provider.example.com/webhook/utap
```

The `utap_token=NEW` sentinel indicates that no token has been provided yet.

**Step 2: RA parses the request and mints a token.**

The RA extracts the amount, currency, and purpose from the URI. It mints a
token via the CFP (see Section 8.2) and replaces the request parameters:

```
https://cfp.example.com/pay?utap_token=550e8400-e29b-41d4-a716-446655440000&utap_version=0.1
```

**Step 3: RA sends the completed URI to the PA.**

The PA receives the URI, extracts the `utap_token` value, and validates it
with the CFP (see Section 8.4).

### 7.4 URI Security

- UTAP URIs MUST be transmitted over HTTPS (TLS 1.2 or later).
- A `utap_token` value in a URI is a **bearer credential**: any party that
  possesses the URI can initiate a validation request. This is by design --
  it enables simple, stateless transport.
- Mitigations against interception: short TTL (default 1 hour), single-use
  (burned after transfer), HTTPS transport, and the requirement that only the
  token owner can authorize a transfer.
- URIs SHOULD NOT be logged in plaintext by intermediary systems. Agents
  SHOULD treat UTAP URIs with the same confidentiality as bearer tokens.

---

## 8. Protocol Flows

This section defines the message sequences for all UTAP operations. Each flow
includes an ASCII sequence diagram and normative HTTP request/response
examples.

### 8.1 Agent Registration and Delegation

See Section 5.4 for the full registration flow. Summary:

```
Principal                    Agent                       CFP
    |                          |                          |
    |-- Sign delegation JWT -->|                          |
    |                          |                          |
    |                          |-- POST /agents/register ->|
    |                          |   { agent_id,            |
    |                          |     delegation_tokens,   |
    |                          |     public_key }         |
    |                          |                          |
    |                          |<-- 201 { auth_token,  ---|
    |                          |     effective_scopes,    |
    |                          |     effective_constraints}|
    |                          |                          |
```

### 8.2 Token Issuance (Minting)

The RA mints a token by requesting one from the CFP. The CFP validates the
agent's identity, checks budget availability, and creates the token.

```
RA                                           CFP
 |                                            |
 |-- POST /cfp/v1/tokens                   -->|
 |   Authorization: Bearer <jwt>              |
 |   Idempotency-Key: <key>                   |
 |   { amount, currency, purpose,             |
 |     budget_scope, idempotency_key }        |
 |                                            |
 |   [CFP validates: identity, delegation,    |
 |    budget, purpose, idempotency]           |
 |                                            |
 |<-- 201 { token object, status=MINTED }  ---|
 |                                            |
```

**Request:**

```http
POST /cfp/v1/tokens HTTP/1.1
Host: cfp.example.com
Authorization: Bearer <agent-jwt>
Content-Type: application/json
Idempotency-Key: purchasing-bot-7-1740000000-x7f3a

{
  "amount": {
    "value": "1500.00",
    "currency": "USD"
  },
  "purpose": {
    "category": "compute",
    "description": "GPU rental for ML training job #4821",
    "reference": "PO-2026-0042"
  },
  "budget_scope": "acme/engineering/ml-team"
}
```

**Response (success):**

```http
HTTP/1.1 201 Created
Content-Type: application/json

{
  "token_id": "550e8400-e29b-41d4-a716-446655440000",
  "version": "utap-0.1",
  "issuer": "cfp.example.com",
  "amount": {
    "value": "1500.00",
    "currency": "USD"
  },
  "owner": "utap:agent:acme.com:purchasing-bot-7",
  "status": "MINTED",
  "purpose": {
    "category": "compute",
    "description": "GPU rental for ML training job #4821",
    "reference": "PO-2026-0042"
  },
  "budget_scope": "acme/engineering/ml-team",
  "delegation_chain_hash": "sha256:a1b2c3d4e5f6...",
  "audit_chain_hash": "sha256:f7e8d9c0b1a2...",
  "idempotency_key": "purchasing-bot-7-1740000000-x7f3a",
  "created_at": "2026-02-26T10:00:00Z",
  "expires_at": "2026-02-26T11:00:00Z",
  "metadata": {}
}
```

**Validation rules (CFP MUST enforce):**

1. The agent's JWT is valid and not expired.
2. The delegation chain grants access to the specified `budget_scope`.
3. The `purpose.category` is in the agent's `allowed_purposes`.
4. The amount does not exceed `max_amount_per_tx` in the delegation chain.
5. The cumulative spending does not exceed `max_amount_per_day` or per-month
   limits.
6. The budget scope has sufficient remaining allocation.
7. The idempotency key has not been used with different parameters.

### 8.3 Payment Request

The PA communicates a payment requirement to the RA. This step occurs
out-of-band relative to the CFP -- the PA sends a URI directly to the RA.

```
PA                                            RA
 |                                             |
 |-- Payment Request URI                    -->|
 |   utap_token=NEW&utap_amount=1500.00       |
 |   &utap_currency=USD&utap_purpose=compute  |
 |                                             |
 |   [RA parses URI, mints token via CFP]      |
 |                                             |
 |<-- Payment URI                           ---|
 |   utap_token=550e8400-...                   |
 |                                             |
```

The transport mechanism for this exchange is application-specific. The PA may
send the URI in an HTTP response body, an API call, a message, or any other
channel. UTAP does not prescribe the transport for this step -- only the URI
format.

### 8.4 Payment Presentation and Validation

Upon receiving a payment URI, the PA extracts the `utap_token` and validates
it with the CFP.

```
PA                                           CFP
 |                                            |
 |-- POST /cfp/v1/tokens/{id}/validate    -->|
 |   Authorization: Bearer <pa-jwt>           |
 |   { presenting_agent, expected_amount,     |
 |     expected_currency, expected_purpose }   |
 |                                            |
 |   [CFP checks: token exists, not expired,  |
 |    not already claimed, amount matches,     |
 |    purpose matches]                         |
 |                                            |
 |<-- 200 { valid: true, token details }   ---|
 |                                            |
```

**Request:**

```http
POST /cfp/v1/tokens/550e8400-e29b-41d4-a716-446655440000/validate HTTP/1.1
Host: cfp.example.com
Authorization: Bearer <pa-jwt>
Content-Type: application/json

{
  "presenting_agent": "utap:agent:cloudco.com:billing-agent",
  "expected_amount": {
    "value": "1500.00",
    "currency": "USD"
  },
  "expected_purpose": "compute"
}
```

**Response (valid):**

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "valid": true,
  "token_id": "550e8400-e29b-41d4-a716-446655440000",
  "amount": {
    "value": "1500.00",
    "currency": "USD"
  },
  "owner": "utap:agent:acme.com:purchasing-bot-7",
  "status": "MINTED",
  "purpose": {
    "category": "compute",
    "description": "GPU rental for ML training job #4821",
    "reference": "PO-2026-0042"
  },
  "audit_chain_hash": "sha256:f7e8d9c0b1a2...",
  "expires_at": "2026-02-26T11:00:00Z"
}
```

**Response (invalid -- already claimed):**

```http
HTTP/1.1 410 Gone
Content-Type: application/json

{
  "error": {
    "code": "TOKEN_ALREADY_CLAIMED",
    "message": "Token 550e8400-... has already been transferred",
    "token_id": "550e8400-e29b-41d4-a716-446655440000",
    "retry": false
  }
}
```

The PA MUST NOT release resources until validation succeeds.

### 8.5 Hold (Optional Escrow)

After validation, the PA MAY request a hold on the token before delivering the
resource. This prevents the RA from revoking or re-using the token during
resource delivery.

```http
POST /cfp/v1/tokens/550e8400-e29b-41d4-a716-446655440000/hold HTTP/1.1
Host: cfp.example.com
Authorization: Bearer <pa-jwt>
Content-Type: application/json

{
  "hold_duration_seconds": 300
}
```

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "token_id": "550e8400-e29b-41d4-a716-446655440000",
  "status": "HELD",
  "held_by": "utap:agent:cloudco.com:billing-agent",
  "hold_expires_at": "2026-02-26T10:10:00Z"
}
```

A hold MUST expire after the specified duration (maximum 5 minutes by default).
If the hold expires without a transfer, the token returns to `MINTED` status.

### 8.6 Transfer

The PA requests transfer of the token from the RA to itself.

```
PA                                           CFP
 |                                            |
 |-- POST /cfp/v1/tokens/{id}/transfer    -->|
 |   { to: <PA AID>,                         |
 |     idempotency_key }                      |
 |                                            |
 |   [CFP: atomic state change                |
 |    MINTED/HELD -> TRANSFERRED              |
 |    owner: RA -> PA                         |
 |    audit record created]                   |
 |                                            |
 |<-- 200 { token, status=TRANSFERRED }    ---|
 |                                            |
```

**Request:**

```http
POST /cfp/v1/tokens/550e8400-e29b-41d4-a716-446655440000/transfer HTTP/1.1
Host: cfp.example.com
Authorization: Bearer <pa-jwt>
Content-Type: application/json
Idempotency-Key: billing-agent-1740000300-t9k2

{
  "to": "utap:agent:cloudco.com:billing-agent"
}
```

**Response:**

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "token_id": "550e8400-e29b-41d4-a716-446655440000",
  "status": "TRANSFERRED",
  "previous_owner": "utap:agent:acme.com:purchasing-bot-7",
  "owner": "utap:agent:cloudco.com:billing-agent",
  "transferred_at": "2026-02-26T10:05:30Z",
  "audit_chain_hash": "sha256:b2c3d4e5f6a7..."
}
```

The transfer MUST be **atomic**: either it completes fully or it fails with no
side effects. The CFP MUST use serialized access (e.g., database-level locking
or serializable transactions) to prevent race conditions.

### 8.7 Acknowledgment and Burn

After the PA has delivered the resource, it burns the token to finalize the
transaction.

```
PA                                           CFP
 |                                            |
 |-- POST /cfp/v1/tokens/{id}/burn        -->|
 |   { confirmation: "service-delivered" }    |
 |                                            |
 |   [CFP: TRANSFERRED -> BURNED             |
 |    final audit record created              |
 |    settlement triggered]                   |
 |                                            |
 |<-- 200 { status=BURNED,                ---|
 |     final_audit_hash }                     |
 |                                            |
```

**Request:**

```http
POST /cfp/v1/tokens/550e8400-e29b-41d4-a716-446655440000/burn HTTP/1.1
Host: cfp.example.com
Authorization: Bearer <pa-jwt>
Content-Type: application/json

{
  "confirmation": "service-delivered",
  "delivery_reference": "gpu-session-8821"
}
```

**Response:**

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "token_id": "550e8400-e29b-41d4-a716-446655440000",
  "status": "BURNED",
  "burned_at": "2026-02-26T10:06:00Z",
  "final_audit_hash": "sha256:c3d4e5f6a7b8...",
  "settlement": {
    "status": "PENDING",
    "estimated_settlement": "2026-02-27T00:00:00Z"
  }
}
```

After burning, the token is permanently consumed. The `settlement` field
indicates the status of the actual fund movement, which is handled by the
settlement layer (outside the scope of this specification).

### 8.8 Settlement (Informative)

Settlement -- the actual movement of money between accounts -- is outside the
normative scope of UTAP. The protocol defines the trigger (a token reaching
`BURNED` status) and the audit record format, but the settlement mechanism is
pluggable.

Common settlement approaches:

- **Internal ledger:** For agents within the same organization, the CFP
  simply adjusts account balances. No external transfer needed.
- **Batch ACH/wire:** The CFP aggregates burned tokens and initiates periodic
  batch transfers between organizations.
- **Real-time payment rails:** Integration with FedNow, SEPA Instant, or
  similar systems for cross-org settlement.
- **Stablecoin:** Settlement via on-chain stablecoin transfer for
  organizations that prefer blockchain-based finality.

The CFP SHOULD expose a settlement status API:

```
GET /cfp/v1/settlements?token_id=550e8400-...
```

### 8.9 Batch Operations

For high-volume scenarios (e.g., an agent purchasing many API calls), tokens
MAY be minted in batch:

```http
POST /cfp/v1/tokens/batch HTTP/1.1
Host: cfp.example.com
Authorization: Bearer <agent-jwt>
Content-Type: application/json
Idempotency-Key: purchasing-bot-7-batch-001

{
  "count": 100,
  "amount": {
    "value": "1.00",
    "currency": "USD"
  },
  "purpose": {
    "category": "api-access",
    "description": "Inference API calls"
  },
  "budget_scope": "acme/engineering/ml-team"
}
```

**Response:**

```http
HTTP/1.1 201 Created
Content-Type: application/json

{
  "batch_id": "batch-a1b2c3d4-...",
  "count": 100,
  "token_ids": [
    "110e8400-e29b-41d4-a716-446655440001",
    "110e8400-e29b-41d4-a716-446655440002",
    "..."
  ],
  "total_amount": {
    "value": "100.00",
    "currency": "USD"
  },
  "status": "MINTED",
  "created_at": "2026-02-26T10:00:00Z",
  "expires_at": "2026-02-26T11:00:00Z"
}
```

The CFP MUST apply budget validation against the total batch amount. Each
token in the batch is independently tracked through the lifecycle.

### 8.10 Event Notifications

The CFP SHOULD support real-time event notifications via WebSocket or webhooks
to inform agents of token state changes.

**WebSocket connection:**

```
GET /cfp/v1/events?token=<jwt> HTTP/1.1
Host: cfp.example.com
Upgrade: websocket
Connection: Upgrade
```

**Event message format:**

```json
{
  "type": "token.transferred",
  "token_id": "550e8400-e29b-41d4-a716-446655440000",
  "timestamp": "2026-02-26T10:05:30Z",
  "data": {
    "previous_owner": "utap:agent:acme.com:purchasing-bot-7",
    "new_owner": "utap:agent:cloudco.com:billing-agent"
  }
}
```

Agents SHOULD use event notifications rather than polling for status changes.

---

## 9. Audit Trail Specification

### 9.1 Design Philosophy

Every UTAP token state transition produces an audit record. The audit trail is
not optional, not configurable, and not deferrable. It is a structural property
of the protocol.

Audit records are stored on the CFP and hash-chained for tamper evidence. The
token itself carries only the hash of its latest audit record (the
`audit_chain_hash` field), keeping tokens compact while enabling offline
integrity verification.

This is the "hybrid" approach: lightweight hash in the token, full records on
the CFP, with access-controlled retrieval.

### 9.2 Audit Record Format

```json
{
  "audit_id": "aud-a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "token_id": "550e8400-e29b-41d4-a716-446655440000",
  "event_type": "TOKEN_TRANSFERRED",
  "timestamp": "2026-02-26T10:05:30.123Z",
  "actor": "utap:agent:cloudco.com:billing-agent",
  "actor_delegation_chain": [
    "utap:agent:cloudco.com:ceo-jane",
    "utap:agent:cloudco.com:billing-agent"
  ],
  "counterparty": "utap:agent:acme.com:purchasing-bot-7",
  "amount": {
    "value": "1500.00",
    "currency": "USD"
  },
  "purpose": {
    "category": "compute",
    "description": "GPU rental for ML training job #4821"
  },
  "budget_scope": "acme/engineering/ml-team",
  "previous_hash": "sha256:f7e8d9c0b1a2...",
  "record_hash": "sha256:b2c3d4e5f6a7...",
  "cfp_signature": "RS256:..."
}
```

**Field definitions:**

| Field                    | Type   | Description                                     |
|--------------------------|--------|-------------------------------------------------|
| `audit_id`               | string | Unique identifier for this audit record         |
| `token_id`               | string | The token this record pertains to               |
| `event_type`             | string | The type of event (see 9.3)                     |
| `timestamp`              | string | ISO 8601 with millisecond precision             |
| `actor`                  | string | AID of the agent that triggered the event       |
| `actor_delegation_chain` | string[]| Full delegation chain of the actor              |
| `counterparty`           | string | AID of the other party (if applicable)          |
| `amount`                 | object | Token amount at time of event                   |
| `purpose`                | object | Token purpose at time of event                  |
| `budget_scope`           | string | Budget scope charged                            |
| `previous_hash`          | string | Hash of the preceding audit record for this token (null for first record) |
| `record_hash`            | string | SHA-256 of this record (see 9.4)                |
| `cfp_signature`          | string | CFP's RS256 signature over `record_hash`        |

### 9.3 Audit Event Types

| Event Type              | Trigger                                       |
|-------------------------|-----------------------------------------------|
| `TOKEN_MINTED`          | Token created via issuance endpoint            |
| `TOKEN_HELD`            | Token placed in escrow hold                    |
| `TOKEN_RELEASED`        | Token released from hold (returned to MINTED)  |
| `TOKEN_TRANSFERRED`     | Token ownership changed                        |
| `TOKEN_BURNED`          | Token permanently consumed                     |
| `TOKEN_EXPIRED`         | Token TTL exceeded                             |
| `TOKEN_REVOKED`         | Token cancelled by owner or CFP                |
| `VALIDATION_REQUESTED`  | PA requested token validation                  |
| `VALIDATION_FAILED`     | Token validation failed (bad amount, expired)  |
| `DELEGATION_CREATED`    | New delegation chain registered                |
| `DELEGATION_REVOKED`    | Delegation chain revoked                       |
| `BUDGET_DEBITED`        | Budget decreased (minting)                     |
| `BUDGET_CREDITED`       | Budget restored (expiry, revocation)           |

### 9.4 Hash Chain Construction

The hash chain provides tamper evidence. Each audit record is cryptographically
linked to its predecessor.

**Hashing algorithm:**

1. Construct the record object **without** the `record_hash` and
   `cfp_signature` fields.
2. Serialize to **canonical JSON**: keys sorted lexicographically, no
   whitespace, no trailing commas, UTF-8 encoding.
3. Compute: `record_hash = SHA-256(canonical_json_bytes)`
4. Encode as: `"sha256:" + hex(hash)`

**Chain linking:**

- The first audit record for a token has `previous_hash: null`.
- Each subsequent record sets `previous_hash` to the `record_hash` of the
  immediately preceding record for the same `token_id`.
- The token's `audit_chain_hash` field MUST always equal the `record_hash` of
  the most recent audit record.

**CFP signature:**

After computing `record_hash`, the CFP signs it:

```
cfp_signature = RS256(cfp_private_key, record_hash_bytes)
```

This allows any party with the CFP's public key to verify that the record was
produced by the CFP and has not been altered.

### 9.5 Audit Log Retrieval

The CFP MUST provide API endpoints for retrieving audit records:

**By token:**

```http
GET /cfp/v1/audit/tokens/550e8400-e29b-41d4-a716-446655440000 HTTP/1.1
Host: cfp.example.com
Authorization: Bearer <agent-jwt>
```

**Response:**

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "token_id": "550e8400-e29b-41d4-a716-446655440000",
  "records": [
    {
      "audit_id": "aud-001-...",
      "event_type": "TOKEN_MINTED",
      "timestamp": "2026-02-26T10:00:00.000Z",
      "actor": "utap:agent:acme.com:purchasing-bot-7",
      "previous_hash": null,
      "record_hash": "sha256:f7e8d9c0b1a2...",
      "cfp_signature": "RS256:..."
    },
    {
      "audit_id": "aud-002-...",
      "event_type": "VALIDATION_REQUESTED",
      "timestamp": "2026-02-26T10:05:00.000Z",
      "actor": "utap:agent:cloudco.com:billing-agent",
      "previous_hash": "sha256:f7e8d9c0b1a2...",
      "record_hash": "sha256:a2b3c4d5e6f7...",
      "cfp_signature": "RS256:..."
    },
    {
      "audit_id": "aud-003-...",
      "event_type": "TOKEN_TRANSFERRED",
      "timestamp": "2026-02-26T10:05:30.123Z",
      "actor": "utap:agent:cloudco.com:billing-agent",
      "counterparty": "utap:agent:acme.com:purchasing-bot-7",
      "previous_hash": "sha256:a2b3c4d5e6f7...",
      "record_hash": "sha256:b2c3d4e5f6a7...",
      "cfp_signature": "RS256:..."
    },
    {
      "audit_id": "aud-004-...",
      "event_type": "TOKEN_BURNED",
      "timestamp": "2026-02-26T10:06:00.000Z",
      "actor": "utap:agent:cloudco.com:billing-agent",
      "previous_hash": "sha256:b2c3d4e5f6a7...",
      "record_hash": "sha256:c3d4e5f6a7b8...",
      "cfp_signature": "RS256:..."
    }
  ],
  "chain_valid": true
}
```

**By agent (with pagination):**

```http
GET /cfp/v1/audit/agents/utap:agent:acme.com:purchasing-bot-7?from=2026-02-01T00:00:00Z&to=2026-02-28T23:59:59Z&cursor=abc123&limit=50 HTTP/1.1
Host: cfp.example.com
Authorization: Bearer <agent-jwt>
```

**By budget scope:**

```http
GET /cfp/v1/audit/budgets/acme/engineering/ml-team?event_type=BUDGET_DEBITED&from=2026-02-01T00:00:00Z HTTP/1.1
Host: cfp.example.com
Authorization: Bearer <principal-jwt>
```

### 9.6 Access Control

- An Agent MAY retrieve audit records for tokens it currently owns or has
  previously owned.
- A Principal MAY retrieve audit records for any Agent in its delegation
  subtree (i.e., agents it directly or transitively delegated to).
- The CFP MAY define a **compliance role** that can read all audit records
  for a given organization, for use by internal audit teams.
- Requests for audit records outside the caller's access scope MUST return
  HTTP 403.

### 9.7 Integrity Verification

Any party with access to the audit chain and the CFP's public key can verify
integrity offline:

**Pseudocode (Python):**

```python
import hashlib
import json

def canonical_json(record):
    """Serialize to canonical JSON: sorted keys, no whitespace, UTF-8."""
    stripped = {k: v for k, v in record.items()
                if k not in ("record_hash", "cfp_signature")}
    return json.dumps(stripped, sort_keys=True, separators=(",", ":")).encode("utf-8")

def verify_audit_chain(records, cfp_public_key):
    """Verify the integrity of an audit chain."""
    previous_hash = None

    for record in sorted(records, key=lambda r: r["timestamp"]):
        # 1. Verify hash computation
        expected_hash = "sha256:" + hashlib.sha256(
            canonical_json(record)
        ).hexdigest()
        assert record["record_hash"] == expected_hash, \
            f"Hash mismatch on {record['audit_id']}"

        # 2. Verify chain linkage
        assert record["previous_hash"] == previous_hash, \
            f"Chain break at {record['audit_id']}"

        # 3. Verify CFP signature
        assert verify_rs256(
            cfp_public_key,
            record["cfp_signature"],
            bytes.fromhex(record["record_hash"].removeprefix("sha256:"))
        ), f"Invalid signature on {record['audit_id']}"

        previous_hash = record["record_hash"]

    return previous_hash  # Should match token.audit_chain_hash
```

**Pseudocode (JavaScript):**

```javascript
async function verifyAuditChain(records, cfpPublicKey) {
  const sorted = [...records].sort((a, b) =>
    a.timestamp.localeCompare(b.timestamp)
  );

  let previousHash = null;

  for (const record of sorted) {
    // 1. Compute expected hash
    const stripped = Object.fromEntries(
      Object.entries(record)
        .filter(([k]) => k !== "record_hash" && k !== "cfp_signature")
        .sort(([a], [b]) => a.localeCompare(b))
    );
    const canonical = JSON.stringify(stripped);
    const hashBuffer = await crypto.subtle.digest(
      "SHA-256",
      new TextEncoder().encode(canonical)
    );
    const expectedHash = "sha256:" + Array.from(new Uint8Array(hashBuffer))
      .map(b => b.toString(16).padStart(2, "0")).join("");

    if (record.record_hash !== expectedHash) {
      throw new Error(`Hash mismatch on ${record.audit_id}`);
    }

    // 2. Verify chain linkage
    if (record.previous_hash !== previousHash) {
      throw new Error(`Chain break at ${record.audit_id}`);
    }

    // 3. Verify CFP signature (using Web Crypto API)
    const valid = await crypto.subtle.verify(
      "RSASSA-PKCS1-v1_5",
      cfpPublicKey,
      base64ToBuffer(record.cfp_signature.replace("RS256:", "")),
      hexToBuffer(record.record_hash.replace("sha256:", ""))
    );
    if (!valid) {
      throw new Error(`Invalid signature on ${record.audit_id}`);
    }

    previousHash = record.record_hash;
  }

  return previousHash; // Should match token.audit_chain_hash
}
```

---

## 10. Budget and Policy Enforcement

### 10.1 Budget Hierarchy

Budgets are organized in a slash-delimited hierarchy reflecting organizational
structure:

```
acme/                               $1,000,000/month (organization)
  engineering/                      $500,000/month   (department)
    ml-team/                        $100,000/month   (team)
      purchasing-bot-7              $10,000/day      (agent)
    platform-team/                  $200,000/month   (team)
  sales/                            $300,000/month   (department)
```

A child budget is always a subset of its parent. The CFP MUST enforce that the
sum of child allocations does not exceed the parent allocation.

### 10.2 Budget Object

```json
{
  "scope": "acme/engineering/ml-team",
  "limits": {
    "per_transaction": {
      "value": "2000.00",
      "currency": "USD"
    },
    "per_day": {
      "value": "10000.00",
      "currency": "USD"
    },
    "per_month": {
      "value": "100000.00",
      "currency": "USD"
    }
  },
  "spent": {
    "today": {
      "value": "3500.00",
      "currency": "USD"
    },
    "this_month": {
      "value": "42000.00",
      "currency": "USD"
    }
  },
  "allowed_purposes": ["compute", "data-license", "api-access"],
  "requires_approval_above": {
    "value": "5000.00",
    "currency": "USD"
  },
  "created_at": "2026-01-01T00:00:00Z",
  "updated_at": "2026-02-26T10:00:00Z"
}
```

### 10.3 Budget Enforcement Rules

The CFP MUST enforce budget constraints at **token minting time**, not at
transfer time. This ensures that an agent cannot mint tokens it cannot cover.

**Enforcement sequence:**

1. Agent requests token minting with `budget_scope` and `amount`.
2. CFP resolves the full budget hierarchy from the specified scope up to the
   organization root.
3. CFP checks **all** levels:
   - `per_transaction` limit at the agent's scope.
   - `per_day` limit at the agent's scope.
   - `per_month` limit at every level from agent to organization root.
4. CFP checks `allowed_purposes` at the agent's effective scope.
5. If `requires_approval_above` is exceeded, the CFP places the mint request
   in a `PENDING_APPROVAL` state and notifies the relevant Principal.
6. If all checks pass, the CFP atomically debits the budget and creates the
   token.

**Budget debit is atomic with token creation.** If minting fails for any
reason, the budget is not debited. If a token expires or is revoked, the
budget is credited back.

### 10.4 Budget Management API

**Query budget:**

```http
GET /cfp/v1/budgets/acme/engineering/ml-team HTTP/1.1
Host: cfp.example.com
Authorization: Bearer <agent-jwt>
```

**Create or update budget (principal only):**

```http
PUT /cfp/v1/budgets/acme/engineering/ml-team HTTP/1.1
Host: cfp.example.com
Authorization: Bearer <principal-jwt>
Content-Type: application/json

{
  "limits": {
    "per_transaction": { "value": "2000.00", "currency": "USD" },
    "per_day": { "value": "10000.00", "currency": "USD" },
    "per_month": { "value": "100000.00", "currency": "USD" }
  },
  "allowed_purposes": ["compute", "data-license", "api-access"],
  "requires_approval_above": { "value": "5000.00", "currency": "USD" }
}
```

**Rejection due to budget exhaustion:**

```http
HTTP/1.1 403 Forbidden
Content-Type: application/json

{
  "error": {
    "code": "BUDGET_EXCEEDED",
    "message": "Daily spending limit exceeded for scope acme/engineering/ml-team",
    "budget_scope": "acme/engineering/ml-team",
    "limit": { "value": "10000.00", "currency": "USD" },
    "spent": { "value": "9200.00", "currency": "USD" },
    "requested": { "value": "1500.00", "currency": "USD" },
    "retry": false
  }
}
```

### 10.5 Approval Workflow

When a minting request exceeds the `requires_approval_above` threshold:

1. The CFP creates the token in `PENDING_APPROVAL` status (a sub-state of
   MINTED that prevents the token from being used in a payment URI).
2. The CFP notifies the relevant Principal via the event notification system
   (Section 8.10).
3. The Principal reviews and approves or rejects via:

```http
POST /cfp/v1/tokens/550e8400-.../approve HTTP/1.1
Host: cfp.example.com
Authorization: Bearer <principal-jwt>
Content-Type: application/json

{
  "decision": "approved",
  "reason": "Within project budget, approved per policy AIP-2026-042"
}
```

4. Upon approval, the token moves to `MINTED` status and can be used normally.
5. Upon rejection, the token moves to `REVOKED` status and the budget is
   credited back.

---

## 11. Idempotency Mechanism

### 11.1 The Problem

Agents operate over networks that may drop connections, return timeouts, or
produce ambiguous errors. Without idempotency, an agent that retries a failed
mint request might create two tokens. An agent that retries a transfer might
attempt a double-spend.

### 11.2 Idempotency Key Format

The idempotency key is a client-generated string, maximum 128 characters,
submitted in the `Idempotency-Key` HTTP header and (for token operations)
also in the request body.

**Recommended format:**

```
{agent-local-id}-{unix-timestamp}-{random-nonce}
```

**Example:**

```
purchasing-bot-7-1740000000-x7f3a
```

The key MUST be unique per logical operation. The same key MUST NOT be reused
for different operations.

### 11.3 CFP Behavior

When the CFP receives a request with an `Idempotency-Key` header:

| Scenario | CFP Action |
|----------|------------|
| Key is new | Process normally. Store the result keyed by idempotency key with a 24-hour TTL. |
| Key exists, previous request succeeded | Return the cached result. Add `X-Idempotent-Replay: true` header. |
| Key exists, previous request still processing | Return `409 Conflict` with `Retry-After` header. |
| Key exists, previous request failed | Allow retry. Process as new request. |

### 11.4 Distinguishing Retries from Double-Spends

The interaction between idempotency keys and token IDs provides double-spend
detection:

| Same Idempotency Key | Same Token | Interpretation       | CFP Action         |
|-----------------------|------------|----------------------|--------------------|
| Yes                   | Yes        | Retry (safe)         | Return cached result |
| No                    | Yes        | Double-spend attempt | Reject with 409    |
| Yes                   | No         | Client error         | Reject with 400    |
| No                    | No         | Independent operations | Process both     |

### 11.5 Request Example

**First request:**

```http
POST /cfp/v1/tokens HTTP/1.1
Host: cfp.example.com
Authorization: Bearer <agent-jwt>
Content-Type: application/json
Idempotency-Key: purchasing-bot-7-1740000000-x7f3a

{
  "amount": { "value": "1500.00", "currency": "USD" },
  "purpose": { "category": "compute" },
  "budget_scope": "acme/engineering/ml-team"
}
```

```http
HTTP/1.1 201 Created
Content-Type: application/json

{
  "token_id": "550e8400-...",
  "status": "MINTED"
}
```

**Retry (same idempotency key):**

```http
POST /cfp/v1/tokens HTTP/1.1
Host: cfp.example.com
Authorization: Bearer <agent-jwt>
Content-Type: application/json
Idempotency-Key: purchasing-bot-7-1740000000-x7f3a

{
  "amount": { "value": "1500.00", "currency": "USD" },
  "purpose": { "category": "compute" },
  "budget_scope": "acme/engineering/ml-team"
}
```

```http
HTTP/1.1 201 Created
Content-Type: application/json
X-Idempotent-Replay: true

{
  "token_id": "550e8400-...",
  "status": "MINTED"
}
```

The response is identical. The token was only created once. The budget was only
debited once.

---

## 12. Security Considerations

### 12.1 Transport Security

All UTAP communications MUST use TLS 1.2 or later. The CFP MUST NOT accept
plaintext HTTP connections. Agents MUST validate TLS certificates and MUST NOT
disable certificate verification.

### 12.2 Token as Bearer Credential

A UTAP token ID, when embedded in a URI, functions as a bearer credential.
Any party that obtains the URI can attempt to validate the token with the CFP.
However, only the PA that has been validated can request a transfer. The
following mitigations reduce bearer credential risk:

- **Short TTL:** Tokens expire after 1 hour by default.
- **Single use:** Tokens are burned after transfer and cannot be reused.
- **HTTPS:** Transport encryption prevents interception.
- **Transfer authorization:** The CFP requires the PA to authenticate before
  executing a transfer. Possessing a token ID alone is not sufficient to
  steal funds.

### 12.3 Agent Authentication Security

- JWTs MUST be signed by the CFP using RS256 or ES256. Symmetric algorithms
  (HS256) MUST NOT be used in production.
- JWTs MUST expire within 24 hours.
- The CFP MUST validate JWTs on every request, including signature, expiry,
  and issuer claims.
- JWT secrets and signing keys MUST be stored securely (e.g., HSM, cloud
  secret manager) and MUST NOT be committed to source control.
- Agents SHOULD implement token refresh to avoid service interruption near
  expiry.

### 12.4 Delegation Chain Attacks

**Confused deputy:** An agent presents a valid delegation chain but uses its
authority for unauthorized purposes (e.g., purchasing personal compute under a
corporate budget). *Mitigation:* Purpose binding restricts token use to
declared categories. Budget scoping restricts which budgets an agent can
access. Audit trails enable post-hoc detection.

**Chain elongation:** An agent creates unauthorized sub-delegations.
*Mitigation:* The `can_delegate` flag explicitly controls sub-delegation. The
CFP validates the full chain on registration. Maximum chain depth of 5 limits
blast radius.

**Revocation delay:** A principal revokes a delegation, but the agent acts
before revocation propagates. *Mitigation:* Short-lived JWTs (max 24 hours)
limit the window. The CFP checks revocation status on every minting request.
Critical revocations SHOULD trigger immediate JWT invalidation.

**Key compromise:** An agent's private key is compromised, allowing an
attacker to impersonate the agent. *Mitigation:* Short-lived JWTs limit
exposure. Budget limits cap financial loss. Anomaly detection (unusual spending
patterns, unexpected purposes) SHOULD trigger alerts.

### 12.5 Double-Spend Prevention

UTAP prevents double-spending through three mechanisms:

1. **Single-use tokens:** Each token transitions through a linear lifecycle
   terminating in BURNED or EXPIRED. There is no mechanism to reuse a token.

2. **Atomic state transitions:** The CFP MUST use serialized access for token
   state changes. Two concurrent transfer requests for the same token MUST
   result in exactly one success and one `TOKEN_ALREADY_CLAIMED` error.

3. **Idempotency keys:** Retries are distinguished from duplicate requests,
   preventing accidental double-minting.

### 12.6 Audit Trail Integrity

The hash chain provides **tamper evidence**: if any audit record is modified
or deleted, the chain breaks and verification fails. The CFP signature on each
record provides **non-repudiation**: the CFP cannot deny having produced a
specific record.

These properties assume:

- The CFP's signing key is not compromised.
- The SHA-256 hash function remains collision-resistant.
- At least one party (agent, principal, or external auditor) retains a copy of
  the `audit_chain_hash` for comparison.

Implementers SHOULD consider periodic external archival of audit chain hashes
to a system independent of the CFP (e.g., a corporate compliance database)
for defense-in-depth.

### 12.7 Rate Limiting

The CFP SHOULD implement rate limiting per agent to prevent abuse:

- Minting: SHOULD limit to a configurable number of requests per minute.
- Validation: SHOULD limit per presenting agent per minute.
- Batch operations: SHOULD enforce maximum batch sizes.

Rate-limited requests MUST return HTTP 429 with a `Retry-After` header.

### 12.8 Privacy Considerations

Audit records contain information about agent identity, delegation chains,
transaction amounts, and purposes. This data is sensitive.

- The CFP MUST enforce access control on audit retrieval (Section 9.6).
- Cross-organization transactions reveal counterparty identity to both
  organizations' auditors. This is intentional for compliance but should be
  disclosed to participants.
- The CFP SHOULD support data retention policies aligned with regulatory
  requirements (e.g., 7-year retention for financial records).
- The CFP MUST NOT expose audit data to agents or principals outside the
  access control rules defined in Section 9.6.

---

## 13. Error Codes

### 13.1 Error Response Format

All error responses MUST use the following JSON structure:

```json
{
  "error": {
    "code": "UTAP_ERROR_CODE",
    "message": "Human-readable description",
    "retry": false,
    "details": {}
  }
}
```

The `retry` field indicates whether the client SHOULD retry the request. The
`details` field MAY contain additional context specific to the error code.

### 13.2 Error Code Table

| HTTP | UTAP Code               | Retry | Description                                      |
|------|-------------------------|-------|--------------------------------------------------|
| 400  | `INVALID_REQUEST`       | No    | Malformed request body or missing required fields|
| 400  | `INVALID_TOKEN_ID`      | No    | Token ID fails UUID validation                   |
| 400  | `INVALID_AMOUNT`        | No    | Amount is not a valid decimal string             |
| 400  | `INVALID_PURPOSE`       | No    | Purpose category not in controlled vocabulary    |
| 400  | `INVALID_IDEMPOTENCY`   | No    | Idempotency key reused with different parameters |
| 401  | `UNAUTHORIZED`          | No    | Missing, expired, or invalid JWT                 |
| 403  | `FORBIDDEN`             | No    | Valid JWT but insufficient permissions           |
| 403  | `BUDGET_EXCEEDED`       | No    | Transaction exceeds budget scope limits          |
| 403  | `PURPOSE_NOT_ALLOWED`   | No    | Purpose category not in agent's allowed list     |
| 403  | `DELEGATION_INVALID`    | No    | Delegation chain verification failed             |
| 404  | `TOKEN_NOT_FOUND`       | No    | Token ID does not exist                          |
| 404  | `AGENT_NOT_FOUND`       | No    | Agent ID not registered                          |
| 404  | `BUDGET_NOT_FOUND`      | No    | Budget scope does not exist                      |
| 408  | `HOLD_EXPIRED`          | Yes   | Token hold timed out; re-validate and retry      |
| 409  | `TOKEN_ALREADY_CLAIMED` | No    | Token has already been transferred               |
| 409  | `IDEMPOTENCY_CONFLICT`  | Yes   | Previous request with this key still processing  |
| 409  | `TOKEN_STATE_CONFLICT`  | No    | Token is not in the expected state for this operation |
| 410  | `TOKEN_BURNED`          | No    | Token has been permanently consumed              |
| 410  | `TOKEN_EXPIRED`         | No    | Token TTL exceeded                               |
| 410  | `TOKEN_REVOKED`         | No    | Token was revoked                                |
| 413  | `AMOUNT_TOO_LARGE`      | No    | Amount exceeds system or per-transaction limits  |
| 429  | `RATE_LIMITED`          | Yes   | Too many requests; see `Retry-After` header      |
| 500  | `INTERNAL_ERROR`        | Yes   | CFP internal error                               |
| 503  | `SERVICE_BUSY`          | Yes   | CFP at capacity; see `Retry-After` header        |

### 13.3 Retry Behavior

When `retry` is `true`, the client SHOULD retry with exponential backoff:

- Base delay: 1 second
- Multiplier: 2x per attempt
- Maximum attempts: 5
- Maximum delay: 32 seconds
- Jitter: SHOULD add random jitter of 0-1 seconds to prevent thundering herd

When a `Retry-After` header is present, the client MUST wait at least the
specified duration before retrying.

---

## 14. Transport Binding

### 14.1 HTTP/JSON Binding (Normative)

This specification defines a normative binding for HTTP/1.1 and HTTP/2 with
JSON request and response bodies.

**Base URL:** `https://{cfp-host}/cfp/v1/`

**Content type:** `application/json` for all request and response bodies.

**Authentication:** `Authorization: Bearer <JWT>` header on all authenticated
endpoints.

**API versioning:** URL path segment (`/v1/`). Future versions will use `/v2/`,
etc. The CFP SHOULD support at least one prior version during transitions.

### 14.2 Endpoint Summary

| Method | Path                              | Auth | Description                  |
|--------|-----------------------------------|------|------------------------------|
| POST   | `/cfp/v1/agents/register`         | No*  | Register agent               |
| POST   | `/cfp/v1/agents/refresh`          | Yes  | Refresh JWT                  |
| POST   | `/cfp/v1/delegations/revoke`      | Yes  | Revoke delegation            |
| POST   | `/cfp/v1/tokens`                  | Yes  | Mint token                   |
| POST   | `/cfp/v1/tokens/batch`            | Yes  | Mint batch of tokens         |
| GET    | `/cfp/v1/tokens/{id}`             | Yes  | Get token status             |
| POST   | `/cfp/v1/tokens/{id}/validate`    | Yes  | Validate token               |
| POST   | `/cfp/v1/tokens/{id}/hold`        | Yes  | Place hold on token          |
| POST   | `/cfp/v1/tokens/{id}/transfer`    | Yes  | Transfer token ownership     |
| POST   | `/cfp/v1/tokens/{id}/burn`        | Yes  | Burn token                   |
| POST   | `/cfp/v1/tokens/{id}/revoke`      | Yes  | Revoke token                 |
| POST   | `/cfp/v1/tokens/{id}/approve`     | Yes  | Approve pending token        |
| GET    | `/cfp/v1/audit/tokens/{id}`       | Yes  | Get audit trail for token    |
| GET    | `/cfp/v1/audit/agents/{aid}`      | Yes  | Get audit trail for agent    |
| GET    | `/cfp/v1/audit/budgets/{scope}`   | Yes  | Get audit trail for budget   |
| GET    | `/cfp/v1/budgets/{scope}`         | Yes  | Get budget status            |
| PUT    | `/cfp/v1/budgets/{scope}`         | Yes  | Create/update budget         |
| GET    | `/cfp/v1/events`                  | Yes  | WebSocket event stream       |

*Agent registration authenticates via delegation tokens in the request body,
not via JWT header.

### 14.3 CORS

CFP implementations that serve browser-based agents MUST support CORS with:

- `Access-Control-Allow-Methods: POST, GET, OPTIONS`
- `Access-Control-Allow-Headers: Content-Type, Authorization, Idempotency-Key`
- `Access-Control-Allow-Origin:` configured per deployment

### 14.4 Future Transport Bindings (Informative)

Future versions of this specification MAY define bindings for:

- **gRPC:** For high-performance agent-to-CFP communication with streaming
  support and strongly-typed message definitions.
- **WebSocket:** For bidirectional real-time communication, extending the
  event notification mechanism defined in Section 8.10.
- **MQTT:** For constrained IoT agents with limited bandwidth and compute.

These bindings are informative and are not part of the v0.1 specification.

---

## 15. Worked Examples

### 15.1 Simple Agent-to-Agent Payment

**Scenario:** Acme Corp's ML training agent (`purchasing-bot-7`) needs to
purchase 2 hours of GPU compute from CloudCo's billing agent
(`billing-agent`). The compute costs $1,500.

**Preconditions:**
- Both agents are registered with the same CFP (`cfp.example.com`).
- `purchasing-bot-7` has a valid JWT and budget allocation.
- `billing-agent` has a valid JWT.

---

**Step 1: PA sends payment request to RA.**

CloudCo's billing agent responds to an API call from Acme's agent with a
payment request URI:

```
https://cfp.example.com/pay?utap_token=NEW&utap_version=0.1&utap_amount=1500.00&utap_currency=USD&utap_purpose=compute&utap_ref=gpu-quote-8821
```

---

**Step 2: RA mints a token.**

```http
POST /cfp/v1/tokens HTTP/1.1
Host: cfp.example.com
Authorization: Bearer <purchasing-bot-7-jwt>
Content-Type: application/json
Idempotency-Key: pb7-1740000000-mint-8821

{
  "amount": { "value": "1500.00", "currency": "USD" },
  "purpose": {
    "category": "compute",
    "description": "2hr GPU rental, CloudCo quote #gpu-quote-8821",
    "reference": "PO-2026-0042"
  },
  "budget_scope": "acme/engineering/ml-team"
}
```

```http
HTTP/1.1 201 Created

{
  "token_id": "550e8400-e29b-41d4-a716-446655440000",
  "status": "MINTED",
  "amount": { "value": "1500.00", "currency": "USD" },
  "expires_at": "2026-02-26T11:00:00Z",
  "audit_chain_hash": "sha256:f7e8d9c0b1a2..."
}
```

---

**Step 3: RA sends payment URI to PA.**

```
https://cfp.example.com/pay?utap_token=550e8400-e29b-41d4-a716-446655440000&utap_version=0.1
```

---

**Step 4: PA validates the token.**

```http
POST /cfp/v1/tokens/550e8400-e29b-41d4-a716-446655440000/validate HTTP/1.1
Host: cfp.example.com
Authorization: Bearer <billing-agent-jwt>
Content-Type: application/json

{
  "presenting_agent": "utap:agent:cloudco.com:billing-agent",
  "expected_amount": { "value": "1500.00", "currency": "USD" },
  "expected_purpose": "compute"
}
```

```http
HTTP/1.1 200 OK

{ "valid": true, "token_id": "550e8400-...", "status": "MINTED" }
```

---

**Step 5: PA requests transfer.**

```http
POST /cfp/v1/tokens/550e8400-e29b-41d4-a716-446655440000/transfer HTTP/1.1
Host: cfp.example.com
Authorization: Bearer <billing-agent-jwt>
Content-Type: application/json
Idempotency-Key: ba-1740000300-transfer-8821

{
  "to": "utap:agent:cloudco.com:billing-agent"
}
```

```http
HTTP/1.1 200 OK

{
  "token_id": "550e8400-...",
  "status": "TRANSFERRED",
  "previous_owner": "utap:agent:acme.com:purchasing-bot-7",
  "owner": "utap:agent:cloudco.com:billing-agent"
}
```

*PA provisions the GPU session for Acme's agent.*

---

**Step 6: PA burns the token after service delivery.**

```http
POST /cfp/v1/tokens/550e8400-e29b-41d4-a716-446655440000/burn HTTP/1.1
Host: cfp.example.com
Authorization: Bearer <billing-agent-jwt>
Content-Type: application/json

{
  "confirmation": "service-delivered",
  "delivery_reference": "gpu-session-8821"
}
```

```http
HTTP/1.1 200 OK

{
  "token_id": "550e8400-...",
  "status": "BURNED",
  "final_audit_hash": "sha256:c3d4e5f6a7b8..."
}
```

**Total HTTP calls to CFP: 4** (mint, validate, transfer, burn).
**Total URI exchanges between agents: 2** (request, payment).

---

### 15.2 Delegated Payment with Budget Rejection

**Scenario:** Acme's CEO delegates to CFO Alice, who delegates to
`purchasing-bot-7`. The bot attempts to buy a $12,000 dataset (exceeding its
$10,000/day limit), gets rejected, then tries a $3,000 subset.

---

**Step 1: Bot attempts to mint $12,000 token.**

```http
POST /cfp/v1/tokens HTTP/1.1
Host: cfp.example.com
Authorization: Bearer <purchasing-bot-7-jwt>
Content-Type: application/json
Idempotency-Key: pb7-1740100000-dataset-full

{
  "amount": { "value": "12000.00", "currency": "USD" },
  "purpose": { "category": "data-license", "description": "Full enterprise dataset" },
  "budget_scope": "acme/engineering/ml-team"
}
```

**Rejected:**

```http
HTTP/1.1 403 Forbidden

{
  "error": {
    "code": "BUDGET_EXCEEDED",
    "message": "Daily spending limit exceeded for scope acme/engineering/ml-team",
    "budget_scope": "acme/engineering/ml-team",
    "limit": { "value": "10000.00", "currency": "USD" },
    "spent": { "value": "3500.00", "currency": "USD" },
    "requested": { "value": "12000.00", "currency": "USD" },
    "retry": false
  }
}
```

---

**Step 2: Bot mints $3,000 token for a subset instead.**

```http
POST /cfp/v1/tokens HTTP/1.1
Host: cfp.example.com
Authorization: Bearer <purchasing-bot-7-jwt>
Content-Type: application/json
Idempotency-Key: pb7-1740100001-dataset-subset

{
  "amount": { "value": "3000.00", "currency": "USD" },
  "purpose": { "category": "data-license", "description": "Subset: training data only" },
  "budget_scope": "acme/engineering/ml-team"
}
```

```http
HTTP/1.1 201 Created

{
  "token_id": "660f9500-...",
  "status": "MINTED",
  "amount": { "value": "3000.00", "currency": "USD" }
}
```

The budget rejection is recorded in the audit trail as a `VALIDATION_FAILED`
event, providing visibility to the CFO that the agent attempted and adapted.

---

### 15.3 Audit Trail Retrieval and Verification

**Scenario:** Acme's CFO Alice wants to verify the complete audit trail for
the GPU payment from Example 15.1.

---

**Step 1: Retrieve audit trail.**

```http
GET /cfp/v1/audit/tokens/550e8400-e29b-41d4-a716-446655440000 HTTP/1.1
Host: cfp.example.com
Authorization: Bearer <cfo-alice-jwt>
```

```http
HTTP/1.1 200 OK

{
  "token_id": "550e8400-e29b-41d4-a716-446655440000",
  "records": [
    {
      "audit_id": "aud-001",
      "event_type": "TOKEN_MINTED",
      "timestamp": "2026-02-26T10:00:00.000Z",
      "actor": "utap:agent:acme.com:purchasing-bot-7",
      "actor_delegation_chain": [
        "utap:agent:acme.com:ceo-board",
        "utap:agent:acme.com:cfo-alice",
        "utap:agent:acme.com:purchasing-bot-7"
      ],
      "amount": { "value": "1500.00", "currency": "USD" },
      "purpose": { "category": "compute", "description": "2hr GPU rental" },
      "budget_scope": "acme/engineering/ml-team",
      "previous_hash": null,
      "record_hash": "sha256:f7e8d9c0b1a2...",
      "cfp_signature": "RS256:..."
    },
    {
      "audit_id": "aud-002",
      "event_type": "VALIDATION_REQUESTED",
      "timestamp": "2026-02-26T10:05:00.000Z",
      "actor": "utap:agent:cloudco.com:billing-agent",
      "previous_hash": "sha256:f7e8d9c0b1a2...",
      "record_hash": "sha256:a2b3c4d5e6f7...",
      "cfp_signature": "RS256:..."
    },
    {
      "audit_id": "aud-003",
      "event_type": "TOKEN_TRANSFERRED",
      "timestamp": "2026-02-26T10:05:30.123Z",
      "actor": "utap:agent:cloudco.com:billing-agent",
      "counterparty": "utap:agent:acme.com:purchasing-bot-7",
      "amount": { "value": "1500.00", "currency": "USD" },
      "previous_hash": "sha256:a2b3c4d5e6f7...",
      "record_hash": "sha256:b2c3d4e5f6a7...",
      "cfp_signature": "RS256:..."
    },
    {
      "audit_id": "aud-004",
      "event_type": "TOKEN_BURNED",
      "timestamp": "2026-02-26T10:06:00.000Z",
      "actor": "utap:agent:cloudco.com:billing-agent",
      "previous_hash": "sha256:b2c3d4e5f6a7...",
      "record_hash": "sha256:c3d4e5f6a7b8...",
      "cfp_signature": "RS256:..."
    }
  ],
  "chain_valid": true
}
```

---

**Step 2: Verify locally.**

Alice's compliance tool runs the verification algorithm from Section 9.7. It
confirms:

1. Each `record_hash` matches the SHA-256 of the canonical JSON of its record.
2. Each `previous_hash` matches the `record_hash` of the preceding record.
3. Each `cfp_signature` is valid against the CFP's public key.
4. The final `record_hash` (`sha256:c3d4e5f6a7b8...`) matches the token's
   `final_audit_hash` returned when the token was burned.

**Result:** The audit chain is intact. Alice can see that `purchasing-bot-7`
(acting under her delegation) paid $1,500 to CloudCo's billing agent for GPU
compute, and the service was delivered and confirmed.

---

## 16. References

### Normative References

- **[RFC 2119]** Bradner, S., "Key words for use in RFCs to Indicate
  Requirement Levels", BCP 14, RFC 2119, March 1997.

- **[RFC 8174]** Leiba, B., "Ambiguity of Uppercase vs Lowercase in RFC 2119
  Key Words", BCP 14, RFC 8174, May 2017.

- **[RFC 7519]** Jones, M., Bradley, J., and N. Sakimura, "JSON Web Token
  (JWT)", RFC 7519, May 2015.

- **[RFC 7518]** Jones, M., "JSON Web Algorithms (JWA)", RFC 7518, May 2015.

- **[RFC 3986]** Berners-Lee, T., Fielding, R., and L. Masinter, "Uniform
  Resource Identifier (URI): Generic Syntax", RFC 3986, January 2005.

- **[ISO 4217]** International Organization for Standardization, "Currency
  codes", ISO 4217.

- **[ISO 8601]** International Organization for Standardization, "Date and
  time format", ISO 8601.

### Informative References

- **[US 11,455,625 B2]** Kavanagh, G.P., "Method of electronic payment by
  means of a uniform resource identifier (URI)", US Patent 11,455,625,
  September 27, 2022.

- **[RFC 7807]** Nottingham, M. and E. Wilde, "Problem Details for HTTP
  APIs", RFC 7807, March 2016.

- **[FIPS 180-4]** National Institute of Standards and Technology, "Secure
  Hash Standard (SHS)", FIPS PUB 180-4, August 2015. (SHA-256)

---

## Appendix A: Purpose Category Vocabulary

The following purpose categories are defined for UTAP v0.1. Implementations
MUST support all categories listed here. The CFP MAY define additional
categories as extensions, prefixed with `x-` (e.g., `x-custom-category`).

| Category            | Description                                          |
|---------------------|------------------------------------------------------|
| `compute`           | Rental or purchase of computational resources        |
| `model-inference`   | Payment for AI/ML model inference calls              |
| `data-license`      | Licensing of datasets or data access                 |
| `api-access`        | Payment for API call quotas or subscriptions         |
| `storage`           | Cloud storage allocation or usage                    |
| `bandwidth`         | Network bandwidth or data transfer                   |
| `human-labor`       | Payment for human services (review, annotation, etc.)|
| `subscription`      | Recurring subscription payments                      |
| `internal-transfer` | Transfer between accounts within the same org        |
| `refund`            | Refund of a previous transaction                     |
| `other`             | Catch-all for categories not listed above             |

When `refund` is used as the purpose category, the `purpose.reference` field
SHOULD contain the `token_id` of the original transaction being refunded.

---

## Appendix B: JSON Schema Definitions

### B.1 Token Object

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://utap.dev/schema/v0.1/token.json",
  "title": "UTAP Token",
  "type": "object",
  "required": [
    "token_id", "version", "issuer", "amount", "owner", "status",
    "purpose", "budget_scope", "delegation_chain_hash",
    "audit_chain_hash", "idempotency_key", "created_at", "expires_at"
  ],
  "properties": {
    "token_id": {
      "type": "string",
      "pattern": "^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$"
    },
    "version": {
      "type": "string",
      "const": "utap-0.1"
    },
    "issuer": {
      "type": "string",
      "format": "hostname"
    },
    "amount": {
      "$ref": "#/$defs/amount"
    },
    "owner": {
      "type": "string",
      "pattern": "^utap:agent:.+:.+$"
    },
    "status": {
      "type": "string",
      "enum": ["MINTED", "HELD", "TRANSFERRED", "BURNED", "EXPIRED", "REVOKED"]
    },
    "purpose": {
      "$ref": "#/$defs/purpose"
    },
    "budget_scope": {
      "type": "string",
      "pattern": "^[a-z0-9_-]+(/[a-z0-9_-]+)*$"
    },
    "delegation_chain_hash": {
      "type": "string",
      "pattern": "^sha256:[0-9a-f]{64}$"
    },
    "audit_chain_hash": {
      "type": "string",
      "pattern": "^sha256:[0-9a-f]{64}$"
    },
    "idempotency_key": {
      "type": "string",
      "maxLength": 128
    },
    "created_at": {
      "type": "string",
      "format": "date-time"
    },
    "expires_at": {
      "type": "string",
      "format": "date-time"
    },
    "metadata": {
      "type": "object",
      "additionalProperties": true
    }
  },
  "$defs": {
    "amount": {
      "type": "object",
      "required": ["value", "currency"],
      "properties": {
        "value": {
          "type": "string",
          "pattern": "^[0-9]+(\\.[0-9]{1,2})?$"
        },
        "currency": {
          "type": "string",
          "pattern": "^[A-Z]{3}$"
        }
      }
    },
    "purpose": {
      "type": "object",
      "required": ["category"],
      "properties": {
        "category": {
          "type": "string",
          "enum": [
            "compute", "model-inference", "data-license", "api-access",
            "storage", "bandwidth", "human-labor", "subscription",
            "internal-transfer", "refund", "other"
          ]
        },
        "description": {
          "type": "string",
          "maxLength": 500
        },
        "reference": {
          "type": "string",
          "maxLength": 256
        }
      }
    }
  }
}
```

### B.2 Audit Record

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://utap.dev/schema/v0.1/audit-record.json",
  "title": "UTAP Audit Record",
  "type": "object",
  "required": [
    "audit_id", "token_id", "event_type", "timestamp", "actor",
    "previous_hash", "record_hash", "cfp_signature"
  ],
  "properties": {
    "audit_id": {
      "type": "string"
    },
    "token_id": {
      "type": "string",
      "pattern": "^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$"
    },
    "event_type": {
      "type": "string",
      "enum": [
        "TOKEN_MINTED", "TOKEN_HELD", "TOKEN_RELEASED",
        "TOKEN_TRANSFERRED", "TOKEN_BURNED", "TOKEN_EXPIRED",
        "TOKEN_REVOKED", "VALIDATION_REQUESTED", "VALIDATION_FAILED",
        "DELEGATION_CREATED", "DELEGATION_REVOKED",
        "BUDGET_DEBITED", "BUDGET_CREDITED"
      ]
    },
    "timestamp": {
      "type": "string",
      "format": "date-time"
    },
    "actor": {
      "type": "string",
      "pattern": "^utap:agent:.+:.+$"
    },
    "actor_delegation_chain": {
      "type": "array",
      "items": { "type": "string" }
    },
    "counterparty": {
      "type": ["string", "null"]
    },
    "amount": {
      "$ref": "token.json#/$defs/amount"
    },
    "purpose": {
      "$ref": "token.json#/$defs/purpose"
    },
    "budget_scope": {
      "type": "string"
    },
    "previous_hash": {
      "type": ["string", "null"],
      "pattern": "^sha256:[0-9a-f]{64}$"
    },
    "record_hash": {
      "type": "string",
      "pattern": "^sha256:[0-9a-f]{64}$"
    },
    "cfp_signature": {
      "type": "string"
    }
  }
}
```

### B.3 Error Response

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://utap.dev/schema/v0.1/error.json",
  "title": "UTAP Error Response",
  "type": "object",
  "required": ["error"],
  "properties": {
    "error": {
      "type": "object",
      "required": ["code", "message", "retry"],
      "properties": {
        "code": {
          "type": "string"
        },
        "message": {
          "type": "string"
        },
        "retry": {
          "type": "boolean"
        },
        "details": {
          "type": "object",
          "additionalProperties": true
        }
      }
    }
  }
}
```

### B.4 Delegation Token

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://utap.dev/schema/v0.1/delegation.json",
  "title": "UTAP Delegation Token",
  "type": "object",
  "required": [
    "type", "version", "delegator", "delegate", "scopes",
    "constraints", "chain", "iat", "exp"
  ],
  "properties": {
    "type": {
      "type": "string",
      "const": "utap-delegation"
    },
    "version": {
      "type": "string",
      "const": "0.1"
    },
    "delegator": {
      "type": "string",
      "pattern": "^utap:agent:.+:.+$"
    },
    "delegate": {
      "type": "string",
      "pattern": "^utap:agent:.+:.+$"
    },
    "scopes": {
      "type": "array",
      "items": { "type": "string" }
    },
    "constraints": {
      "type": "object",
      "properties": {
        "max_amount_per_tx": { "type": "string" },
        "max_amount_per_day": { "type": "string" },
        "allowed_purposes": {
          "type": "array",
          "items": { "type": "string" }
        },
        "can_delegate": { "type": "boolean" },
        "expires_at": { "type": "string", "format": "date-time" }
      }
    },
    "chain": {
      "type": "array",
      "items": { "type": "string" }
    },
    "iat": { "type": "number" },
    "exp": { "type": "number" }
  }
}
```

---

## Appendix C: Reference Implementation Notes

The [Secure File Flow](https://github.com/gregkavanmern/secure-file-flow)
project implements the core architectural patterns that UTAP formalizes, applied
to encrypted file transfer rather than financial payments.

### Mapping to UTAP Concepts

| Secure File Flow Component           | UTAP Equivalent                       |
|--------------------------------------|---------------------------------------|
| `TicketDispenser` Durable Object     | CFP token management engine           |
| `AVAILABLE/RESERVED/CLAIMED` states  | `MINTED/HELD/TRANSFERRED+BURNED`      |
| `POST /api/mint` (with JWT auth)     | Token issuance endpoint               |
| `POST /api/claim` (ticket + pubkey)  | Payment presentation + validation     |
| `POST /api/mass/ack` (burn ticket)   | Token burn endpoint                   |
| `requireAuth` JWT middleware         | Agent authentication                  |
| `?ticket_id=<UUID>` in URL           | `?utap_token=<UUID>` in payment URI   |
| WebSocket seller notifications       | CFP event notifications               |
| `maxConcurrentDownloads` throttling  | Rate limiting                         |
| `isValidUUID` validation             | Token ID validation                   |
| `errorJson` structured errors        | UTAP error response format            |
| 1-hour ticket expiration             | Default 1-hour token TTL              |
| Two-step reserve/claim in mass flow  | HELD state + transfer                 |

### Architecture Validation

The reference implementation demonstrates that the three-party handshake
pattern (agent mints credential, shares via URI, counterparty validates with
server, server burns after delivery) is practical, performant, and
implementable on modern serverless infrastructure (Cloudflare Workers +
Durable Objects).

Implementers building UTAP-compliant CFPs are encouraged to study the reference
codebase for patterns around atomic state transitions, concurrent access
control, WebSocket event broadcasting, and ephemeral credential management.

---

*End of UTAP Specification v0.1.0*
