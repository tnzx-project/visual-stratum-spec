# Visual Stratum Protocol Specification

Formal specification of the Visual Stratum protocol suite for steganographic
communication over cryptocurrency mining traffic.

## Status

| Document | Version | Status |
|----------|---------|--------|
| [Visual Stratum Protocol](spec/visual-stratum.md) | draft-00 | Work in progress |

## Scope

This specification defines the wire format, encoding profiles, frame
reassembly, pool detection logic, error handling, chain adaptation rules,
and proxy architecture for the Visual Stratum protocol.

The goal: an implementer should be able to build a compatible encoder,
decoder, and proxy from this document alone.

## Related Repositories

- [tnzx-protocol](https://github.com/tnzx-project/tnzx-protocol) — Reference implementation, test vectors, design papers
- [tnzx-pool-demo](https://github.com/tnzx-project/tnzx-pool-demo) — Proof-of-concept pool and VS3 proxy

## Conventions

This specification uses the keywords MUST, MUST NOT, REQUIRED, SHALL,
SHALL NOT, SHOULD, SHOULD NOT, RECOMMENDED, MAY, and OPTIONAL as
described in [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119).

## License

This specification is licensed under [CC-BY-SA-4.0](LICENSE).
The reference implementation is licensed under LGPL-2.1.
