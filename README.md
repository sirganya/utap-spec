# UTAP -- URI Token Audit Protocol

An open protocol specification for agent-to-agent payments with corporate audit trails.

## What is UTAP?

UTAP defines how autonomous software agents transfer electronic monetary tokens via URIs, validated by a trusted Central Financial Provider (CFP). It is designed for enterprise environments where AI agents transact on behalf of humans and organizations.

**Key properties:**

- **URI-native** -- tokens travel as query parameters in standard HTTPS URIs
- **Three-party validation** -- a trusted CFP prevents double-spending and enforces policy
- **Delegation chains** -- every agent action traces back to a human principal
- **Purpose binding** -- tokens declare machine-readable intent (what the money is for)
- **Budget enforcement** -- hierarchical spending limits at the protocol level
- **Hash-chained audit trail** -- tamper-evident, cryptographically verifiable records of every transaction

## Specification

The full specification is in [UTAP-v0.1.md](./UTAP-v0.1.md).

**Status:** Draft (v0.1.0)

## Intellectual Property

This specification describes methods covered by [US Patent 11,455,625 B2](https://patents.google.com/patent/US11455625B2). The patent holder has committed to RAND licensing terms for conforming implementations. See the IPR notice in the specification for details.

## License

Apache 2.0 with patent grant. See [LICENSE](./LICENSE).

## Contributing

This specification is in draft status. Feedback, questions, and contributions are welcome via GitHub issues and pull requests.
