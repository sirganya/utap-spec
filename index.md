---
layout: default
title: UTAP - URI Token Audit Protocol
---

# UTAP -- URI Token Audit Protocol

An open protocol specification for agent-to-agent payments with corporate audit trails.

**Version:** 0.1.0 | **Status:** Draft | **Author:** Gregory Peter Kavanagh

---

## Overview

UTAP defines how autonomous software agents transfer electronic monetary tokens via URIs, validated by a trusted Central Financial Provider (CFP). It is designed for enterprise environments where AI agents transact on behalf of humans and organizations.

- **URI-native** -- tokens travel as query parameters in standard HTTPS URIs
- **Three-party validation** -- a trusted CFP prevents double-spending and enforces policy
- **Delegation chains** -- every agent action traces back to a human principal
- **Purpose binding** -- tokens declare machine-readable intent
- **Budget enforcement** -- hierarchical spending limits at the protocol level
- **Hash-chained audit trail** -- tamper-evident, cryptographically verifiable transaction records

## Read the Specification

**[UTAP v0.1.0 -- Full Specification](./UTAP-v0.1)**

## Intellectual Property

This specification describes methods covered by [US Patent 11,455,625 B2](https://patents.google.com/patent/US11455625B2). The patent holder has committed to RAND licensing terms for conforming implementations.

## License

Apache 2.0 with patent grant. See [LICENSE](https://github.com/sirganya/utap-spec/blob/main/LICENSE).

## Contributing

This specification is in draft status. Feedback and contributions are welcome via [GitHub Issues](https://github.com/sirganya/utap-spec/issues).
