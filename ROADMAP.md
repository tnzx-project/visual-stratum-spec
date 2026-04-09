# Visual Stratum Spec — Roadmap

## Phase A — Critical fixes (spec completeness)

### A1. Share-to-Frame Serialization Layer (Section 4.3 new) **DONE (draft-01)**
The spec defines byte extraction from shares (Section 3) and frame parsing
(Section 4), but the bridge between them is unspecified. Two independent
implementers would not interoperate.

Must define:
- Byte stream model: receiver accumulates extracted bytes into a running buffer
- Frame boundary detection: scan for MAGIC (0xAA) + validate header triple
- Last-chunk padding: sender zero-pads the final share's unused slots;
  receiver uses PAYLOAD_LEN from the header to trim
- Per-connection demultiplexing: bytes are keyed by Stratum TCP session
- Resync after lost/corrupt share: discard current partial frame after
  MESSAGE_TIMEOUT_MS, advance one byte past failed MAGIC and retry
- Interleaved frames from same sender: NOT allowed on L1 (one frame at a
  time per connection); allowed on L2/L3 via multi-channel MESSAGE_ID

### A2. VERSION byte contradiction (Section 3.7 vs 4.1) **DONE (draft-01)**
Section 4.1 lists VERSION values 0x01-0x06 (matching encoding profiles).
Section 3.7 says VERSION identifies protocol generation (0x01-0x03 only).
BURST/GHOST/TURBO profiles claim VERSION 0x04/0x05/0x06 in Section 3.5.

Resolution: VERSION in the frame header = protocol generation (0x01-0x03).
BURST/GHOST/TURBO are transport-level profiles, NOT frame-level versions.
Update Section 4.1 table to remove 0x04-0x06. Add note that encoding profile
is determined at share classification, not frame parsing.

### A3. Define message type payloads (Section 4.2) **DONE (draft-01)**
Currently undefined:
- ACK (0x02): payload = 2-byte MESSAGE_ID being acknowledged
- PING (0x03): payload = empty (0 bytes); receiver responds with ACK
- KEY_EXCHANGE (0x04): payload = 32-byte X25519 public key
- HASHCASH (0x06): defer to future version, mark as RESERVED

### A4. Related Work section (Section 1.4 new) **DONE (draft-01)**
Position VS relative to:
- obfs4/meek (Tor pluggable transports): protocol mimicry vs stego-in-real-traffic
- StegoTorus/DeltaShaper: HTTP/video stego vs mining stego
- Partala2018/Frkat2020/Cao2023: on-chain vs mining-layer stego
- Signal sealed sender: analogous pool trust model
State the novel contribution explicitly: (1) mining traffic as cover,
(2) Mining Gate as PoW-based access control, (3) no separate network endpoint.

### A5. Ethical / dual-use section (Section 11.4 new) **DONE (draft-01)**
Required for responsible disclosure. Cover:
- Dual-use acknowledgment
- Mining Gate as natural rate limiter (impractical for high-volume abuse)
- E2E encryption prevents content moderation (feature and risk)
- Designed for 1-to-1/small-group, not mass broadcast

### A6. Expand threat model (Section 11.1) **DONE (draft-01)**
Add subsections:
- Traffic analysis: correlated timing, share rate anomalies
- Communication graph: pool learns who-talks-to-whom (same as Signal server)
- Intersection attacks: small anonymity set if few miners use vs-miner
- Sequential correlation: frame structure across shares is a second-order
  distinguisher (mitigated by encryption, but header bytes if unencrypted)
- User-agent fingerprinting: recommend mimicking standard miner UA
- Zero-result distinguisher: recommend random invalid result instead of 0x00*64

### A7. Key distribution (Section 8.6 new or 1.2 scope note) **DONE (draft-01)**
At minimum, state: "Key distribution (how Alice discovers Bob's public key)
is outside the scope of this specification. The Falo protocol (separate
document) addresses group key management. For 1-to-1 communication, users
exchange public keys out-of-band (QR code, secure messenger, in-person)."

### A8. Minor fixes **DONE (draft-01)**
- Add Hopper et al. 2002 "Provably Secure Steganography" to references
- Mention Stratum v2 (Sv2) forward compatibility limitation
- Nonce hex string byte order: explicitly state big-endian (first 2 hex chars = byte[0])
- ghostDiffMax communication: state as out-of-band configuration (no in-band mechanism yet)
- Max encrypted plaintext calculation: 6400 - 120 = 6280 bytes (state explicitly)

---

## Phase B — Performance optimizations (protocol evolution)

### B1. Compact encryption envelope
Replace 120-byte one-shot envelope (replayId 16 + ephPub 32 + salt 32 + nonce 24 + tag 16)
with a compact session envelope (counter 4 + nonce 24 + tag 16 = 44 bytes) after session
establishment. Requires counter synchronization protocol.

Impact: encrypted "Hello" drops from ~26 to ~11 shares (-58%).
Breaking: yes, requires capability negotiation.

### B2. Negotiable L1 fragment size
Allow L1 Stratum channel to use MAX_FRAGMENT_SIZE = 64 (vs default 128).
Halves per-fragment latency at 5.2% more header overhead.
Non-breaking if negotiated via session capability exchange.

### B3. Raise compression threshold
From 50 to 128 bytes. Eliminates counterproductive compression on
short messages where LZ4 header + padding exceeds savings.
Non-breaking (receivers detect compression via magic bytes).

### B4. Implicit ACK (deferred)
Piggyback acknowledgment in outgoing message ID. Saves 2 shares per ACK.
Breaking: requires new field or message format. Consider for v2.

---

## Review findings source
- External implementer review (2026-03-31): interoperability risks, state gaps
- Privacy expert review (2026-03-31): threat model, ethics, related work
- Protocol efficiency analysis (2026-03-31): overhead calculations, optimizations
