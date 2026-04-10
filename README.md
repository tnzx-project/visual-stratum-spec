# Visual Stratum Protocol Specification

Formal specification of the **Visual Stratum** protocol suite — steganographic communication over cryptocurrency mining traffic.

> Hide encrypted messages inside real mining shares. The shares pass proof-of-work validation. The miner gets paid. An observer sees legitimate mining. Nothing more.

## What this specification defines

| Section | Content |
|---------|---------|
| Encoding Profiles | 6 profiles for embedding data in Stratum share fields (VS1/VS2/VS3) |
| Frame Format | Message fragmentation, reassembly, and type envelope |
| Mining Gate | Proof-of-work access control state machine (INACTIVE → GRACE → ACTIVE) |
| Ghost Share Detection | Pool-side identification of data-carrying shares |
| Cryptographic Design | X25519 ECDH, XChaCha20-Poly1305, HKDF-SHA256, replay protection |
| Proxy Architecture | Transparent VS3 middleware for unmodified Stratum pools |
| Multi-Channel Transport | Adaptive routing across Stratum, PNG, WebSocket, and HTTP/2 channels |
| Chain Adaptation | Monero (RandomX) and Bitcoin-style pool support |
| Security Considerations | Threat model, traffic analysis, detection vectors |

## Status

| Document | Version | Status |
|----------|---------|--------|
| [Visual Stratum Protocol](spec/visual-stratum.md) | draft-01 | Work in progress |

The specification follows RFC 2119 conventions. The goal: an implementer should be able to build a compatible encoder, decoder, and proxy from this document alone.

## Quick Overview

```
Miner A                    VS3 Proxy / Pool                    Miner B
   |                            |                                |
   |-- mining.submit (ghost) -->|  extract payload               |
   |   [sentinel + encrypted    |  reassemble frames             |
   |    payload in nonce/ntime] |  route to recipient             |
   |                            |-- job notification (injected) ->|
   |                            |   [VS3 frame in job fields]     |
   |                            |                                |
   Pool sees: valid Stratum     |   Observer sees: mining traffic |
   No content, no metadata      |   No distinguishing features    |
```

## Encoding at a Glance

| Profile | Bytes/share | Method | Stealth | Chain |
|---------|-------------|--------|---------|-------|
| VS1 | 1 | Nonce LSB (real share, valid PoW) | Maximum | Any |
| VS2 | 3 | Nonce + extranonce2 (real share, valid PoW) | Maximum | Bitcoin-style |
| VS3 | 5 | Ghost share (no PoW required) | High | Monero |

VS1 and VS2 are truly steganographic — shares pass full proof-of-work validation. VS3 trades some stealth for bandwidth using ghost shares with a sentinel marker.

## Related Repositories

| Repository | Description |
|------------|-------------|
| [tnzx-protocol](https://github.com/tnzx-project/tnzx-protocol) | Reference implementation, SDK, design paper, test vectors |
| [tnzx-pool-demo](https://github.com/tnzx-project/tnzx-pool-demo) | VS3 proxy and pool POC, tested on production pools |
| [@tnzx/sdk](https://www.npmjs.com/package/@tnzx/sdk) | Developer SDK — `npm install @tnzx/sdk` |

## Interactive Demos

- **[Protocol Explorer](https://tnzx-project.github.io/tnzx-protocol/learn/)** — step through protocol flows visually
- **[Messaging Flow Demo](https://tnzx-project.github.io/tnzx-pool-demo/demo.html)** — animated walkthrough of the full messaging flow

## Conventions

This specification uses the keywords MUST, MUST NOT, REQUIRED, SHALL,
SHALL NOT, SHOULD, SHOULD NOT, RECOMMENDED, MAY, and OPTIONAL as
described in [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119).

## License

This specification is licensed under [CC-BY-SA-4.0](LICENSE).
The reference implementation is licensed under LGPL-2.1.
