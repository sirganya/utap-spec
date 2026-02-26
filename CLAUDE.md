# UTAP Project Context

## Core Assets

- **Granted patent:** US 11,455,625 B2 ("Method of electronic payment by means of a URI"), granted Sept 27, 2022, expires June 4, 2041. Originally individual, assigned to Quickpoint Teoranta March 2025.
- **Reference implementation:** secure-file-flow -- implements the same three-party trustless handshake pattern for encrypted file transfer (ticket minting, URI sharing, server validation, single-use burn).
- **Protocol spec:** UTAP v0.1 (this repo) -- open standard adapting the patent for agent-to-agent payments with audit trails.

## Architecture Decisions

### Audit Trail Design (Option C -- Hybrid)

Three options were evaluated for where audit data lives:

- **Option A (embedded in token):** Each transfer re-encrypts token with appended audit metadata. Problem: only the CFP can decrypt the full chain (it does the re-encryption), so you've built a less efficient version of Option B. If you try nested encryption, nobody can read the full chain. If you use plaintext audit + encrypted value, anyone intercepting the token sees full chain of custody.
- **Option B (pointer model):** Token stays compact, CFP maintains separate append-only audit ledger keyed by token ID. Agents get back only what they're authorized to see. Practical but no offline verification.
- **Option C (hybrid -- chosen):** Token carries a SHA-256 hash of its audit chain (proving integrity). Full audit records live on the CFP with access-controlled retrieval. Compact tokens, verifiable history, access-controlled audit.

Option C was chosen because it avoids token bloat, enables offline integrity checks via the hash, and keeps access control on the CFP where it belongs.

## Go-to-Market Strategy

### What to monetize

The protocol should be open. Payment systems win by adoption. The patent protects regardless -- open-sourcing an implementation doesn't surrender patent rights.

**The CFP is the business.** Every transaction validates through it. Revenue comes from:
- Per-transaction fees or basis-point cut
- Audit/compliance dashboard sold to corporate finance teams
- Policy enforcement, rate limits, budget controls
- Audit ledger storage and retrieval

**Model: open-source the agent SDKs and protocol spec, run the CFP as a hosted service.**

### Target Buyers

Not developers -- they don't control budgets. The buyers are:
- **Corporate finance/procurement** -- need to know what their agents are spending and why
- **Compliance teams** -- need audit trails for SOX, internal controls, etc.
- **Platform teams** -- need to give agents spending capabilities without giving them a corporate credit card

**The pitch is not "cool token system." The pitch is: "Your agents are about to start buying things from other agents. Who's watching the money?"**

### Positioning

UTAP sits between traditional payment rails (slow, expensive, not agent-native) and crypto (fast, but no corporate compliance story). A system that's agent-native, URI-based, auditable, and runs on a central corporate ledger fills a real gap.

Position UTAP as complementary to existing payment rails -- it can settle via ACH, wire, stablecoin, or internal ledger. UTAP is the protocol layer; settlement is pluggable.

### Risks

- **Timing risk:** The agent economy is early. Launch too soon and you're selling a solution before the problem is felt. Wait too long and Stripe or a cloud provider builds their own.
- **Adoption chicken-and-egg:** Agents need counterparties who accept the tokens. Start within a single enterprise (agents paying internal agents) before going cross-org.
- **Patent scope:** The patent covers URI-based token transfer with third-party validation. A competitor could build agent payments using a different transport mechanism (gRPC, message queues) and arguably not infringe. Worth understanding the claims boundary with a patent attorney.

### Concrete Path

1. **Publish the protocol spec as an open standard** -- gets visibility, establishes originator status. (Done -- this repo.)
2. **Build agent SDKs** (Python, TypeScript) that are trivially easy to integrate -- open source these.
3. **Run a hosted CFP** with a free tier for single-org/dev use, paid tier for production audit and compliance features.
4. **Target one vertical first** -- e.g., AI dev shops whose agents call paid APIs, or procurement teams piloting agent-based purchasing.
5. **The existing secure-file-flow codebase** serves as proof-of-architecture, not the product itself.

### The Moat

The moat is not the code. It's the patent plus being the ledger that corporates trust.

## Technical Differentiators from Patent

The patent (URI-based token payment with three-party validation) provides the foundation. UTAP extends it with:
- **Delegation chains** -- linking every transaction to a human-authorized chain of command
- **Purpose binding** -- machine-readable intent on every token
- **Budget hierarchies** -- organizational spending policies enforced at protocol level
- **Hash-chained audit ledger** -- tamper-evident, cryptographically verifiable records
- **Idempotency** -- distinguishing agent retries from double-spends
- **Escrow/hold states** -- extending the token lifecycle for pre-authorization scenarios

## Links

- **GitHub repo:** https://github.com/sirganya/utap-spec
- **GitHub Pages:** https://sirganya.github.io/utap-spec/
- **Patent:** https://patents.google.com/patent/US11455625B2
- **Reference implementation:** secure-file-flow (same local machine)
