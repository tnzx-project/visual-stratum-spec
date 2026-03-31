# Visual Stratum Protocol Specification

    draft-tnzx-visual-stratum-00

## Status of This Document

This document is a **draft specification (draft-00)** of the Visual Stratum protocol suite. It is a work in progress published to enable independent review, feedback, and interoperable implementation.

This specification has not been submitted to or reviewed by the IETF or any standards body. The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119 [RFC2119].

    Authors:       TNZX Project
    Contact:       tnzx@proton.me
    Date:          March 2026
    Version:       draft-00
    License:       LGPL-2.1-only (protocol suite)
    Repository:    https://github.com/tnzx-project/tnzx-protocol

## Abstract

Visual Stratum is a family of steganographic communication protocols that embed encrypted messages within standard cryptocurrency mining traffic. By exploiting the inherent randomness of proof-of-work share submissions, Visual Stratum encodes data in Stratum protocol fields that are entropy-equivalent to legitimate mining data. The protocol introduces Mining Gate, an access control mechanism that binds communication bandwidth to active proof-of-work, simultaneously providing spam prevention, economic sustainability, and censorship resistance.

This specification defines the wire format, encoding profiles, cryptographic design, frame fragmentation, Mining Gate state machine, ghost share detection, proxy architecture, and multi-channel transport of the Visual Stratum protocol across three generations (VS1, VS2, VS3). An implementer SHOULD be able to construct a conforming encoder, decoder, and proxy using only this document.

---

## Table of Contents

- 1\. Introduction
- 2\. Protocol Overview
- 3\. Encoding Profiles
- 4\. Frame Format
- 5\. Mining Gate
- 6\. Ghost Share Detection
- 7\. Proxy Architecture
- 8\. Cryptographic Design
- 9\. Chain Adaptation
- 10\. Multi-Channel Transport (VS3)
- 11\. Security Considerations
- 12\. Test Vectors
- 13\. References
- Appendix A -- Protocol Constants
- Appendix B -- Example Stratum JSON Exchanges
- Appendix C -- Changelog

---

## 1. Introduction

### 1.1 Purpose

This document specifies the Visual Stratum protocol suite: a set of steganographic communication protocols designed to create covert, encrypted communication channels within standard cryptocurrency mining traffic. The protocols operate within the Stratum mining protocol, requiring no additional network endpoints, no identifiable protocol signatures beyond those inherent to the Stratum connection, and no separate funding model.

### 1.2 Scope

This specification covers:

- Three protocol generations: VS1 (image steganography), VS2 (Stratum embedding + Mining Gate), and VS3 (multi-channel adaptive transport).
- Six encoding profiles for embedding data in Stratum share fields.
- The VS3 frame format for message fragmentation and reassembly.
- The Mining Gate proof-of-work access control mechanism.
- The E2E cryptographic design (key exchange, encryption, replay protection).
- Ghost share detection at the pool.
- Proxy architecture for transparent deployment.
- Multi-channel transport with adaptive mode selection and timing decorrelation.

This specification does NOT cover the Falo anonymous coordination protocol, which is specified in a separate document.

### 1.3 Conventions and Terminology

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119].

| Term | Definition |
|------|-----------|
| **Cover traffic** | Legitimate network traffic within which covert data is hidden |
| **Ghost share** | A Stratum share submitted below the pool's difficulty threshold, used as a pure data carrier; no valid proof-of-work is required |
| **Mining Gate** | The access control mechanism that binds VS channel access to active proof-of-work |
| **Sentinel** | A fixed or derived byte value used to identify ghost shares at the pool |
| **Share** | A Stratum `mining.submit` message containing a candidate block solution |
| **Stratum** | The dominant JSON-RPC mining pool communication protocol (v1) |
| **VS-aware pool** | A mining pool that implements VS ghost share detection and frame extraction |
| **VS-enhanced miner** | A mining client (e.g., tnzxminer) that implements VS encoding profiles |
| **ghostDiffMax** | Pool-side configuration parameter; shares with difficulty <= ghostDiffMax are treated as ghost shares |
| **CSPRNG** | Cryptographically Secure Pseudo-Random Number Generator |
| **AAD** | Additional Authenticated Data (used in AEAD ciphers) |
| **PFS** | Perfect Forward Secrecy |

All multi-byte integer values in the frame format are encoded **big-endian** (network byte order) unless explicitly stated otherwise.

Hexadecimal values are prefixed with `0x` for single values and presented as lowercase hex strings for sequences (e.g., `"aa48656c"`).

Byte offsets are 0-indexed.

---

## 2. Protocol Overview

### 2.1 Architecture

Visual Stratum operates as a layered protocol stack:

```
+-------------------------------------------------------------+
|                    APPLICATION LAYER                         |
|           Chat, Files, Key Exchange, Metadata                |
+-------------------------------------------------------------+
|                    ENCRYPTION LAYER                          |
|         AES-256-GCM + X25519 ECDH + HKDF-SHA256             |
+-------------------------------------------------------------+
|                    FRAMING LAYER                             |
|         VS3 Frame Format (8-byte header + payload)           |
+-------------------------------------------------------------+
|                    TRANSPORT LAYER                           |
|  +----------+----------+-----------+--------------------+    |
|  | L4: PNG  |L3: WS    |L2: HTTP/2 |L1: Stratum Stego   |    |
|  | 45KB/s   |50KB/s    |100KB/s    |5-256 B/share        |    |
|  | (down)   |(bidir)   |(bidir)    |(up)                  |    |
|  +----------+----------+-----------+--------------------+    |
+-------------------------------------------------------------+
|                    MINING GATE                               |
|         Proof-of-work verification (sliding window)          |
+-------------------------------------------------------------+
```

The stack is modular. Implementations MAY support any subset of transport layers. A minimal conforming implementation MUST support at least the L1 Stratum embedding channel and the framing layer.

### 2.2 Protocol Versions

| Version | Name | Innovation | Status |
|---------|------|------------|--------|
| VS1 | Image Steganography | LSB embedding in PNG chart images | Archived |
| VS2 | Stratum Embedding | Upload channel via share fields + Mining Gate | Reference implementation |
| VS3 | Multi-Channel Transport | 4-channel adaptive transport, ghost shares, timing decorrelation | L1 implemented; L2-L4 specified |

Each version is a superset of the previous. A VS3 implementation SHOULD support VS1 and VS2 encoding profiles for backward compatibility.

### 2.3 Deployment Models

#### 2.3.1 Native VS-Aware Pool

```
 VS-enhanced Miner  <--- Stratum :3333 --->  VS-Aware Pool
        |                                         |
        +--- (ghost shares with embedded data) ---+
        |                                         |
        +--- (pool extracts frames, routes)   ----+
```

The pool natively understands VS encoding, extracts frames from ghost shares, and routes messages. This is the simplest deployment model.

Requirements:
- Pool MUST implement ghost share detection (Section 6).
- Pool MUST configure `ghostDiffMax`.
- Miner MUST use a VS-enhanced client (e.g., tnzxminer).

#### 2.3.2 Proxy Deployment

```
 VS-enhanced Miner  <--- Stratum --->  VS Proxy  <--- Stratum --->  Standard Pool
        |                                  |
        +--- (ghost shares)            ----+
        |                                  |
        +--- (proxy extracts, forwards) ---+
```

A VS proxy sits between the miner and a standard (non-VS-aware) pool. The proxy:
- Forwards regular mining shares unmodified to the upstream pool.
- Intercepts ghost shares (below ghostDiffMax), extracts embedded frames, and does NOT forward them upstream.
- Injects VS response frames into downstream pool-to-miner traffic.

See Section 7 for the full proxy specification.

---

## 3. Encoding Profiles

Visual Stratum defines six encoding profiles that determine how payload bytes are embedded in Stratum share fields. All profiles feed into the same framing layer (Section 4); they differ only in the share-level transport.

### 3.1 V1 -- Nonce LSB Embedding (1 byte/share)

    Profile ID:     VERSION_V1 (0x01)
    Chain:          Any Stratum-compatible chain
    Miner:          VS-enhanced miner REQUIRED
    Bytes/share:    1
    Stealth:        Highest (entropy-equivalent, see Section 11.2)

#### 3.1.1 Encoding

The miner embeds one payload byte in the two least-significant nibbles of the nonce field before initiating the PoW search. The nonce is at least 4 bytes (32 bits) for Bitcoin-style Stratum.

```
 Nonce (N bytes, big-endian hex):
 +------+------+------+-----+--------+--------+
 | ...  | ...  | ...  | ... | N-2    | N-1    |
 +------+------+------+-----+--------+--------+
                              ^        ^
                              |        |
               high nibble = (byte >> 4) & 0x0F
                              low nibble = byte & 0x0F
```

The encoding algorithm for a single payload byte `b`:

```
nonce[N-2] = (nonce[N-2] & 0xF0) | ((b >> 4) & 0x0F)
nonce[N-1] = (nonce[N-1] & 0xF0) | (b & 0x0F)
```

The miner MUST set these nibbles BEFORE beginning PoW search. The upper 4 bits of `nonce[N-2]` and `nonce[N-1]`, plus all other nonce bytes, remain free for PoW search. This constrains 8 bits of a 32-bit (or larger) nonce, reducing the search space by a factor of 256.

#### 3.1.2 Extraction

The pool (or proxy) extracts the payload byte by reading the low nibbles:

```
b = ((nonce[N-2] & 0x0F) << 4) | (nonce[N-1] & 0x0F)
```

#### 3.1.3 PoW Validity

The miner MUST find a valid PoW solution with the constrained nonce. The share MUST meet the pool's standard difficulty target. Invalid shares are rejected by the pool regardless of any embedded data.

#### 3.1.4 Undetectability

The low nibbles of a legitimate mining nonce are uniformly distributed (see Section 11.2). Encrypted payload bytes (AES-256-GCM output) are also uniformly distributed. No statistical test can distinguish a V1-modified nonce from an unmodified one. This argument holds with full strength for the V1 profile.

### 3.2 V2 -- Nonce + Extranonce2 Embedding (3 bytes/share)

    Profile ID:     VERSION_V2 (0x02)
    Chain:          Bitcoin-style Stratum only (requires extranonce2)
    Miner:          VS-enhanced miner REQUIRED
    Bytes/share:    3
    Stealth:        High (nonce channel: full; extranonce2: weaker, see 11.2)

#### 3.2.1 Encoding

V2 embeds 3 payload bytes per share across two Stratum fields:

```
 Byte 0:  nonce LSB nibbles          (same as V1)
 Byte 1:  extranonce2[len-2]         (second-to-last byte, full replacement)
 Byte 2:  extranonce2[len-1]         (last byte, full replacement)
```

Layout within Stratum share fields:

```
 Nonce (N bytes):
 +------+-----+--------+--------+
 | ...  | ... | N-2    | N-1    |
 +------+-----+--------+--------+
                 ^        ^
                 |        |
                 +-- payload byte 0 (nibble-split) --+

 Extranonce2 (M bytes, M >= 2):
 +------+-----+--------+--------+
 | ...  | ... | M-2    | M-1    |
 +------+-----+--------+--------+
                 ^        ^
                 |        |
                 byte 1   byte 2
```

The miner MUST preset the extranonce2 payload bytes BEFORE beginning PoW search. In Bitcoin-style Stratum, extranonce2 is part of the coinbase transaction and participates in the full PoW chain (coinbase -> merkle root -> block header -> hash). Modifying extranonce2 after finding a nonce would invalidate the share.

Standard Bitcoin miners iterate extranonce2 as a sequential counter. V2 encoding therefore requires a VS-enhanced miner that controls extranonce2 layout.

#### 3.2.2 Extraction

```
byte0 = ((nonce[N-2] & 0x0F) << 4) | (nonce[N-1] & 0x0F)
byte1 = extranonce2[M-2]
byte2 = extranonce2[M-1]
```

#### 3.2.3 Undetectability Caveat

The extranonce2 channel has a weaker undetectability profile than the nonce channel. Standard miners iterate extranonce2 sequentially; replacing the last 2 bytes with uniformly random encrypted payload creates a distributional shift detectable by a sufficiently motivated observer monitoring extranonce2 values across many shares. See Section 11.2 for the full analysis.

### 3.3 V3-Monero -- Ghost Share Embedding (5 bytes/share)

    Profile ID:     VERSION_V3 (0x03)
    Chain:          Monero (RandomX) with TNZX pool extensions
    Miner:          VS-enhanced miner (tnzxminer) REQUIRED
    Bytes/share:    5 (3 from nonce + 2 from ntime extension)
    Stealth:        Maximum for nonce[1..3]; sentinel byte detectable (see 6.1, 11.2)

#### 3.3.1 Protocol Context

Standard Monero Stratum `mining.submit` contains exactly: `nonce` (4 bytes), `result` (32 bytes), `job_id`, and `params.id`. The fields `ntime` and `extranonce2` do NOT exist in standard Monero Stratum.

In VS3-Monero, the `ntime` field is a TNZX extension field added by the VS-enhanced miner (tnzxminer) to the submit parameters. Standard XMRig does NOT send this field. Standard XMRig does NOT produce ghost shares. VS3-Monero communication REQUIRES both a VS-enhanced miner and a VS-aware pool with `ghostDiffMax` configured.

#### 3.3.2 Ghost Share Definition

A ghost share is a Stratum share submitted with difficulty at or below `ghostDiffMax` (a pool-side configuration parameter, see Section 6.2). Because the share does not need to meet a real PoW difficulty target, the miner may freely choose all nonce bytes. The `result` field of a ghost share contains a hash that does NOT meet the pool's standard difficulty, but the pool accepts the submission as a VS data carrier.

For ghost shares in the reference implementation:

- `nonce`: `'aa'` + hex(3 bytes) -- sentinel 0xAA followed by 3 payload bytes
- `ntime`: 2 bytes payload in low word
- `result`: `'0' x 64` (64 zero characters = 32 zero bytes)

Ghost share senders SHOULD set the `result` field to all zeros (`"0" x 64` hex). While not required for ghost share classification (which relies on sentinel detection and difficulty threshold), a zero result serves as an additional signal to VS-aware proxies, reducing false positive rates in mixed-traffic environments.

#### 3.3.3 Encoding

```
 Share submit fields (VS-enhanced miner, VS-aware pool):
 +-----------------------------+---------------------------+
 |       NONCE (4 bytes)       |  NTIME (4 bytes, ext)     |
 | [0xAA][ b0 ][ b1 ][ b2 ]   | [ hi ][ hi ][ b3 ][ b4 ] |
 | sentinel  payload[0..2]     | preserved   payload[3..4] |
 +-----------------------------+---------------------------+
```

The encoding algorithm:

```
nonce[0] = 0xAA                      // sentinel (ghost share marker)
nonce[1] = payload[0]                // payload byte 0
nonce[2] = payload[1]                // payload byte 1
nonce[3] = payload[2]                // payload byte 2
ntime[0] = current_ntime[0]          // real epoch high byte (PRESERVED)
ntime[1] = current_ntime[1]          // real epoch high byte (PRESERVED)
ntime[2] = payload[3]                // payload byte 3
ntime[3] = payload[4]                // payload byte 4
```

The ntime high 16 bits are taken from the pool's current job `ntime` value. This keeps the timestamp within acceptable drift (well inside the +/-7200s pool acceptance window for ntime drift tolerance on chains that validate it).

#### 3.3.4 Extraction

The pool extracts 5 payload bytes from a confirmed ghost share:

```
payload[0] = nonce[1]
payload[1] = nonce[2]
payload[2] = nonce[3]
payload[3] = ntime[2]
payload[4] = ntime[3]
```

`nonce[0]` (the 0xAA sentinel) is consumed by the pool's ghost share detector and is NOT part of the payload.

#### 3.3.5 Sentinel Limitation

The fixed sentinel `nonce[0] = 0xAA` creates a statistical distinguisher. After observing O(256) shares from a miner, a chi-square test on `nonce[0]` can detect VS3 ghost share usage with high confidence. This is a known limitation. See Section 6.3 for the HMAC-based mitigation and Section 11.2 for the full security analysis.

### 3.4 V3-Generic -- Ghost Share Embedding (7 bytes/share)

    Profile ID:     VERSION_V3 (0x03)
    Chain:          Bitcoin/Ethereum-style Stratum (extranonce2 present)
    Miner:          VS-enhanced miner REQUIRED
    Bytes/share:    7
    Stealth:        Maximum for nonce channel; weaker for extranonce2/ntime
    Status:         Specified, not yet implemented

#### 3.4.1 Encoding

V3-Generic distributes 7 payload bytes across three Stratum fields:

```
 Nonce (8 bytes):
 +---+---+---+---+---+---+--------+--------+
 | 0 | 1 | 2 | 3 | 4 | 5 |   6    |   7    |
 +---+---+---+---+---+---+--------+--------+
                            ^        ^
                      high nibble  low nibble
                      of byte 0    of byte 0

 Extranonce2 (M bytes, M >= 4):
 +---+-----+--------+--------+--------+--------+
 | 0 | ... | M-4    | M-3    | M-2    | M-1    |
 +---+-----+--------+--------+--------+--------+
              ^        ^        ^        ^
              byte 1   byte 2   byte 3   byte 4

 Ntime (4 bytes):
 +--------+--------+--------+--------+
 |   0    |   1    |   2    |   3    |
 +--------+--------+--------+--------+
                     ^        ^
                     byte 5   byte 6
```

The encoding algorithm:

```
// Byte 0: split across nonce last 2 bytes as nibbles
nonce[6] = (nonce[6] & 0xF0) | ((payload[0] >> 4) & 0x0F)
nonce[7] = (nonce[7] & 0xF0) | (payload[0] & 0x0F)

// Bytes 1-4: replace last 4 bytes of extranonce2
extranonce2[M-4] = payload[1]
extranonce2[M-3] = payload[2]
extranonce2[M-2] = payload[3]
extranonce2[M-1] = payload[4]

// Bytes 5-6: replace last 2 bytes of ntime
ntime[2] = payload[5]
ntime[3] = payload[6]
```

#### 3.4.2 Extraction

```
payload[0] = ((nonce[6] & 0x0F) << 4) | (nonce[7] & 0x0F)
payload[1] = extranonce2[M-4]
payload[2] = extranonce2[M-3]
payload[3] = extranonce2[M-2]
payload[4] = extranonce2[M-1]
payload[5] = ntime[2]
payload[6] = ntime[3]
```

#### 3.4.3 Detection Ambiguity with Bitcoin Miners

Standard Bitcoin miners always include `extranonce2` in `mining.submit`. The `extranonce2 present` detection branch (Section 6.1) only applies meaningfully when `difficulty <= ghostDiffMax`, which filters out regular mining shares. However, legitimate sub-difficulty shares during vardiff transitions from Bitcoin miners also contain extranonce2 and would be misidentified. Pools SHOULD apply additional heuristics or REQUIRE an out-of-band capability negotiation before treating sub-difficulty Bitcoin shares as VS3-Generic.

### 3.5 Higher-Bandwidth Profiles

These profiles trade stealth for throughput. They are OPTIONAL extensions.

#### 3.5.1 V3-BURST (VERSION 0x04)

    Bytes/share:    ~200
    Carrier:        extranonce2 full field (base64 encoded)
    Stealth:        High
    Chain:          Any with extranonce2
    Status:         Specified, not yet implemented

Uses the full extranonce2 field to carry base64-encoded payload data. The extranonce2 values will deviate strongly from normal sequential patterns.

#### 3.5.2 V3-GHOST (VERSION 0x05)

    Bytes/share:    ~200
    Carrier:        difficulty-1 cover shares
    Stealth:        Medium
    Chain:          Any
    Status:         Specified, not yet implemented

Uses difficulty-1 cover shares (trivially computed) to carry large payloads. The share rate spike from difficulty-1 submissions is detectable under traffic analysis.

#### 3.5.3 V3-TURBO (VERSION 0x06)

    Bytes/share:    ~256
    Carrier:        Worker password field
    Stealth:        Lower
    Chain:          Any
    Status:         Specified, not yet implemented

Uses the `pass` field in the Stratum `mining.authorize` message to carry payload. TURBO requires a reconnect per message, producing reconnect patterns detectable under DPI. Intended for burst transfers where reconnect frequency is already high.

### 3.6 Profile Selection

An implementation MUST negotiate the encoding profile before beginning data transfer. Profile selection is based on the Stratum variant and available fields:

```
Profile Selection Algorithm:

1. Determine Stratum variant (Monero vs Bitcoin-style)
2. Check pool capabilities (ghostDiffMax, extranonce2 size)
3. Select highest-bandwidth compatible profile:

   if chain == Monero AND pool.ghostDiffMax > 0:
       if miner.supports_ntime_ext:
           profile = VS3-Monero (5 B/share)
       else:
           profile = V1 (1 B/share)  // nonce nibbles only
   elif chain == Bitcoin-style:
       if pool.extranonce2_size >= 4 AND pool.ghostDiffMax > 0:
           profile = VS3-Generic (7 B/share)
       elif pool.extranonce2_size >= 2:
           profile = V2 (3 B/share)
       else:
           profile = V1 (1 B/share)
```

The `VERSION` byte in the frame header (Section 4.1) indicates the encoding profile used. The receiver MUST use the correct extraction method for the indicated version.

### 3.7 Profile Summary Table

| Profile | Version Byte | Chain | Fields Used | B/share | Stealth | Impl Status |
|---------|-------------|-------|-------------|---------|---------|-------------|
| V1 | `0x01` | Any | nonce LSB nibbles | 1 | Highest | Reference impl |
| V2 | `0x02` | Bitcoin-style | nonce LSB + extranonce2 (2B) | 3 | High | Reference impl |
| V3-Monero | `0x03` | Monero | nonce ghost (3B) + ntime ext (2B) | 5 | Maximum/sentinel | Reference impl |
| V3-Generic | `0x03` | Bitcoin-style | nonce (1B) + extranonce2 (4B) + ntime (2B) | 7 | Maximum | Specified |
| BURST | `0x04` | Any | extranonce2 full (base64) | ~200 | High | Specified |
| GHOST | `0x05` | Any | difficulty-1 cover shares | ~200 | Medium | Specified |
| TURBO | `0x06` | Any | worker password field | ~256 | Lower | Specified |

**VERSION byte semantics:** The VERSION byte in the frame header (Section 4.1) identifies the **protocol generation** (VS1=0x01, VS2=0x02, VS3=0x03), NOT the encoding profile. A VS3 frame header is identical regardless of whether the frame was transported via VS3-Monero or VS3-Generic ghost shares. The encoding profile is determined at the share-classification layer (Section 6.1), not at the frame layer. A receiver that processes pre-assembled frames (e.g., via WebSocket relay) does not need to know the original encoding profile.

---

## 4. Frame Format

The VS3 frame format is the canonical framing layer for all Visual Stratum versions. It is **transport-independent**: a frame assembled from VS3-Monero shares is byte-for-byte identical to one assembled from VS3-Generic shares carrying the same payload.

### 4.1 Frame Header

The frame consists of an 8-byte fixed header followed by a variable-length payload.

```
  0                   1                   2                   3
  0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 |     MAGIC     |    VERSION    |      TYPE     |   MESSAGE_ID  |
 |     (0xAA)    |    (0x03)     |   (MSG_TYPE)  |   (high byte) |
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 | MESSAGE_ID    |  FRAG_INDEX   |  FRAG_TOTAL   |  PAYLOAD_LEN  |
 | (low byte)    |   (0-based)   |   (1-50)      |   (0-128)     |
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 |                                                               |
 |                    PAYLOAD (0..128 bytes)                      |
 |                                                               |
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

Field descriptions:

| Offset | Size | Field | Description |
|--------|------|-------|-------------|
| 0 | 1 | MAGIC | Frame boundary marker. MUST be `0xAA`. |
| 1 | 1 | VERSION | Protocol version. `0x01` = VS1, `0x02` = VS2, `0x03` = VS3, `0x04` = BURST, `0x05` = GHOST, `0x06` = TURBO. |
| 2 | 1 | TYPE | Message type (see Section 4.2). |
| 3-4 | 2 | MESSAGE_ID | 16-bit message identifier, big-endian. Links all fragments of one logical message. |
| 5 | 1 | FRAG_INDEX | 0-based index of this fragment within the message. |
| 6 | 1 | FRAG_TOTAL | Total number of fragments for this message. MUST be in range [1, 50]. |
| 7 | 1 | PAYLOAD_LEN | Byte length of this fragment's payload. Encoders MUST generate values in [0, 128] (MAX_FRAGMENT_SIZE). Decoders SHOULD accept values up to 255 for forward compatibility (robustness principle). |
| 8..8+N | N | PAYLOAD | Fragment content. N = PAYLOAD_LEN. |

The total frame size is `8 + PAYLOAD_LEN` bytes, with a maximum of `8 + 128 = 136` bytes.

#### 4.1.1 MAGIC Byte Dual Use

The value `0xAA` serves two purposes:

1. **Frame boundary marker** at byte offset 0 of every VS3 frame.
2. **Ghost share sentinel** as `nonce[0]` in VS3-Monero encoding (Section 3.3).

This dual use is intentional: the first byte of the first frame fragment embedded in a ghost share's nonce naturally occupies `nonce[1..3]`, while `nonce[0]` carries the sentinel. The sentinel and the frame magic byte happen to share the same value `0xAA`, but they serve distinct roles.

#### 4.1.2 Message ID Generation

Implementations MUST generate message IDs that are unique within a reasonable window. The RECOMMENDED approach is:

```
msgId = (CSPRNG(2 bytes) XOR (timestamp_ms & 0xFFFF)) & 0xFFFF
```

Implementations MUST track used message IDs and avoid collisions. When the tracking set exceeds 10,000 entries, the oldest half SHOULD be evicted.

Implementations MUST NOT generate message ID `0xFFFF`, which is reserved for dummy traffic padding (Section 10.3.1). Similarly, for the 32-bit message ID in the multi-channel header, `0xFFFFFFFF` MUST NOT be generated.

### 4.2 Message Types

| Value | Name | Description |
|-------|------|-------------|
| `0x01` | TEXT | UTF-8 text message |
| `0x02` | ACK | Acknowledgment of received message |
| `0x03` | PING | Keepalive / presence check |
| `0x04` | KEY_EXCHANGE | X25519 public key for session establishment |
| `0x05` | ENCRYPTED | AES-256-GCM encrypted payload (opaque to framing layer) |
| `0x06` | HASHCASH | Mining Gate proof-of-work token |

Implementations MUST ignore frames with unknown message types and MUST NOT treat them as errors. This allows future extension.

### 4.3 Fragmentation

Messages larger than `MAX_FRAGMENT_SIZE` (128 bytes) MUST be fragmented before transmission. The framing layer is responsible for splitting and reassembling.

#### 4.3.1 Splitting Algorithm

Given a message payload of L bytes:

```
total_fragments = ceil(L / MAX_FRAGMENT_SIZE)

For i = 0 to total_fragments - 1:
    start = i * MAX_FRAGMENT_SIZE
    end   = min(start + MAX_FRAGMENT_SIZE, L)
    fragment_payload = message[start..end]

    Build frame header:
        MAGIC        = 0xAA
        VERSION      = <encoding profile version>
        TYPE         = <message type>
        MESSAGE_ID   = <generated 16-bit ID>
        FRAG_INDEX   = i
        FRAG_TOTAL   = total_fragments
        PAYLOAD_LEN  = end - start
```

Each frame (header + fragment payload) is then chunked into `BYTES_PER_SHARE`-byte slices and embedded one slice per Stratum share submission.

#### 4.3.2 Shares per Frame

The number of shares required to transmit one frame depends on the encoding profile:

```
shares_per_frame = ceil(frame_size / BYTES_PER_SHARE)
```

Where `frame_size = 8 + PAYLOAD_LEN` and `BYTES_PER_SHARE` depends on the profile (1, 3, 5, or 7).

Example for a maximum-size frame (136 bytes) across profiles:

| Profile | B/share | Shares per frame |
|---------|---------|-----------------|
| V1 | 1 | 136 |
| V2 | 3 | 46 |
| V3-Monero | 5 | 28 |
| V3-Generic | 7 | 20 |

### 4.4 Frame Reassembly

The receiver maintains a pending message table keyed by MESSAGE_ID.

#### 4.4.1 State per Pending Message

```
pending[msgId] = {
    msgId:          uint16,
    msgType:        uint8,
    totalFrags:     uint8,
    fragments:      Array[totalFrags] of (Buffer | null),
    receivedCount:  uint16,
    startTime:      timestamp
}
```

#### 4.4.2 Processing Algorithm

For each received frame:

1. **Validate header**: frame length >= 8, `frame[0] == 0xAA`, `FRAG_TOTAL` in [1, MAX_TOTAL_FRAGMENTS], `FRAG_INDEX < FRAG_TOTAL`.
2. **Check pending table**: if `msgId` not in pending table, create a new entry.
3. **Store fragment**: if `fragments[FRAG_INDEX]` is null, store the payload and increment `receivedCount`. If the slot is already filled, ignore (duplicate/replay).
4. **Check completion**: if `receivedCount == totalFrags`, concatenate fragments in index order, remove from pending table, add to completed messages sliding window.

Frames that fail validation in step 1 MUST be silently discarded. A frame with `frame[0] != 0xAA` indicates a normal (non-VS) share and MUST be flagged as `isNormalShare` for the caller.

#### 4.4.3 Out-of-Order Delivery

Fragments MAY arrive in any order. The reassembly buffer is indexed by FRAG_INDEX, enabling correct reconstruction regardless of arrival order.

### 4.5 Implementation Limits

Implementations MUST enforce the following limits to prevent denial-of-service:

| Parameter | Value | Rationale |
|-----------|-------|-----------|
| MAX_PENDING_MESSAGES | 1,000 | Caps in-flight reassembly state; prevents memory exhaustion from incomplete frame floods |
| MESSAGE_TIMEOUT_MS | 300,000 (5 min) | Incomplete messages older than 5 minutes MUST be discarded |
| MAX_COMPLETED_MESSAGES | 500 | Sliding window of decoded messages for deduplication and replay detection |
| MAX_TOTAL_FRAGMENTS | 50 | Maximum fragments per logical message. Caps message size at 50 * 128 = 6,400 bytes |
| MAX_FRAGMENT_SIZE | 128 | Maximum payload bytes per frame fragment |

Implementations MAY adjust these values based on deployment constraints but SHOULD NOT remove them entirely. Removing reassembly limits creates a denial-of-service vector where an attacker sends unbounded incomplete fragments to exhaust receiver memory.

### 4.6 Error Handling

#### 4.6.1 Eviction Strategies

Implementations MUST apply two independent eviction strategies:

**Strategy 1 -- TTL Eviction** (run at most once per 60 seconds):
Scan all pending messages. Remove entries whose `startTime` is older than `MESSAGE_TIMEOUT_MS`. This guards against abandoned partial messages from senders who disconnect mid-transmission.

**Strategy 2 -- Capacity Eviction** (run whenever pending table reaches MAX_PENDING_MESSAGES):
Sort pending entries by `startTime` ascending. Evict the oldest 20% of entries. This bounds worst-case memory usage regardless of sender behavior.

```
Memory bound = O(MAX_PENDING_MESSAGES * MAX_FRAGMENT_SIZE * MAX_TOTAL_FRAGMENTS)
             = O(1000 * 128 * 50) = O(6.4 MB)
```

#### 4.6.2 Error Conditions

| Condition | Action |
|-----------|--------|
| Frame shorter than 8 bytes | Discard, return error "Frame too short" |
| `frame[0] != 0xAA` | Flag as `isNormalShare`, return to caller |
| `FRAG_TOTAL == 0` or `FRAG_TOTAL > MAX_TOTAL_FRAGMENTS` | Discard, return error "Invalid fragment count" |
| `FRAG_INDEX >= FRAG_TOTAL` | Discard, return error "Invalid fragment index" |
| Duplicate fragment (slot already filled) | Ignore silently (idempotent) |
| Pending table full | Apply capacity eviction, then process |
| Message timeout exceeded | Evict stale entry on next cleanup pass |

Implementations SHOULD log discarded messages for operational monitoring.

---

## 5. Mining Gate

Mining Gate is the access control mechanism that binds VS communication bandwidth to active proof-of-work. The VS covert channel functions ONLY while the user is actively submitting valid mining shares via the Stratum protocol.

### 5.1 State Machine

```
                                [first share]
  INACTIVE -----------------------------------------> GRACE
                                                        |
                                          [>= MIN_ACTIVATION shares
                                           within GRACE_PERIOD]
                                                        |
                                                        v
                                                      ACTIVE
                                                     /      \
                        [hashrate < MIN_HASHRATE]   /        \ [hashrate >= MIN_HASHRATE]
                                                   v          \
                                              SUSPENDED        |
                                                   |           |
                         [COOLDOWN elapsed          |          |
                          AND hashrate              |          |
                          >= MIN_HASHRATE]          |          |
                                                   \          |
                                                    +--------+
```

| State | VS Channel | Transition |
|-------|-----------|------------|
| INACTIVE | Closed | -> GRACE on first valid share |
| GRACE | Closed | -> ACTIVE after MIN_ACTIVATION shares within GRACE_PERIOD |
| ACTIVE | **Open** | -> SUSPENDED if hashrate < MIN_HASHRATE during periodic check |
| SUSPENDED | Closed | -> ACTIVE after COOLDOWN elapsed AND hashrate >= MIN_HASHRATE |

### 5.2 Share Validation

Every share recorded by Mining Gate MUST be a valid proof-of-work submission verified against the current block template. The pool MUST compute the hash (e.g., RandomX for Monero, SHA-256d for Bitcoin) and confirm it meets the pool's difficulty target.

Mining Gate is algorithm-agnostic: it operates on Stratum-reported difficulty, not raw hash values. Any chain whose Stratum implementation reports valid difficulty is compatible.

Ghost shares (difficulty <= ghostDiffMax) are NOT counted by Mining Gate for access control purposes. Only standard-difficulty shares contribute to the share rate calculation.

### 5.3 Hashrate Calculation and Suspension Condition

Mining Gate calculates each miner's hashrate from a sliding window of recent shares:

```
hashrate = total_difficulty_in_window / elapsed_seconds

Where:
  total_difficulty_in_window = SUM(share.difficulty)
                               for all shares with timestamp > (now - WINDOW_SIZE)
  elapsed_seconds = MAX(last_share_time - first_share_time, 1.0)
```

The elapsed time is measured between the first and last share in the window (not the full window size). This avoids underestimating hashrate for miners who recently connected.

The Mining Gate suspension condition is an absolute hashrate threshold:

```
ACTIVE -> SUSPENDED   when hashrate < MIN_HASHRATE during periodic check
SUSPENDED -> ACTIVE   when COOLDOWN elapsed AND hashrate >= MIN_HASHRATE
```

Where `MIN_HASHRATE` defaults to 10 H/s (see Section 5.4). This absolute threshold replaces a relative share-ratio model, simplifying implementation while maintaining adequate spam prevention.

### 5.4 Configuration Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `WINDOW_SIZE` | 600,000 ms (10 min) | Rolling observation window for hashrate calculation |
| `MIN_HASHRATE` | 10 H/s | Minimum hashrate to maintain ACTIVE state |
| `GRACE_PERIOD` | 120,000 ms (2 min) | Initial connection allowance |
| `MIN_ACTIVATION` | 3 shares | Minimum shares to exit GRACE and enter ACTIVE |
| `COOLDOWN` | 300,000 ms (5 min) | Penalty period after suspension |
| `CHECK_INTERVAL` | Random [60,000 - 180,000] ms | Randomized verification interval (CSPRNG) |

The CHECK_INTERVAL MUST be randomized per check using a CSPRNG. A deterministic or predictable check schedule enables burst-and-stop gaming strategies.

### 5.5 Anti-Gaming Properties

| Attack | Defense |
|--------|---------|
| Fake shares | PoW cryptographically verified against block template |
| Old shares | Timestamp MUST fall within observation window |
| Burst-and-stop | Checks are continuous and randomly timed (60-180s) |
| Wallet spoofing | Session cryptographically bound to Stratum connection |
| Predicting checks | Check interval is CSPRNG random, unpredictable |
| Hashrate spoofing | Absolute MIN_HASHRATE threshold; miner must sustain real PoW |

### 5.6 Periodic Verification

The pool MUST run periodic verification of all ACTIVE miners:

```
for each miner with state == ACTIVE:
    prune shares older than WINDOW_SIZE
    hashrate = miner.getHashrate(WINDOW_SIZE)
    if hashrate < MIN_HASHRATE:
        miner.state = SUSPENDED
        miner.suspendedAt = now
```

Stale miners (no share activity for 2 * WINDOW_SIZE) SHOULD be removed from the miner state table entirely.

---

## 6. Ghost Share Detection

Ghost shares are the primary data carrier for VS3 encoding profiles. The pool MUST implement detection logic to distinguish ghost shares from regular mining shares.

### 6.1 Pool-Side Detection Logic

The pool MUST apply the following detection logic to every incoming `mining.submit`:

```
function classifyShare(share, ghostDiffMax):
    if share.difficulty > ghostDiffMax:
        return REGULAR_SHARE              // standard mining, pass through

    if share.nonce[0] == 0xAA:
        return VS3_MONERO_GHOST           // 5 bytes payload

    if share.has_extranonce2:
        return VS3_GENERIC_GHOST          // 7 bytes payload

    return UNKNOWN_LOW_DIFF               // reject or log
```

All code paths for ghost shares MUST return `{"status": "OK"}` in the Stratum response, making the pool's behavior indistinguishable from a standard share acceptance to a passive observer.

### 6.2 ghostDiffMax Configuration

The `ghostDiffMax` parameter determines the difficulty threshold below which shares are treated as ghost shares (data carriers) rather than mining contributions.

| Setting | Value | Behavior |
|---------|-------|----------|
| RECOMMENDED | 1 | Minimum difficulty. Ghost shares at difficulty 1 require negligible computation. |
| Conservative | 500 | Allows some computational cover. Increases false positive risk from legitimate low-diff shares. |
| Disabled | 0 | No ghost share support. VS3 encoding profiles are unavailable. |

Pools MUST document their `ghostDiffMax` setting. Miners MUST be informed of the pool's ghost share capability before attempting VS3 encoding.

### 6.3 HMAC Rotating Sentinel

The fixed `0xAA` sentinel is a known statistical distinguisher (Section 3.3.5), as described in Appendix D of the Visual Stratum design paper [VS-PAPER]. The following HMAC-based scheme replaces the fixed sentinel with a per-session, per-share derived tag.

    Status: Specified, not yet implemented

#### 6.3.1 Key Derivation

At login, both miner and pool derive a session key:

```
session_key = HKDF-SHA256(
    IKM  = miner_password,
    salt = pool_salt,
    info = "tnzx-ghost-v1",
    len  = 32
)
```

The `miner_password` is the `pass` field from `mining.authorize`. The `pool_salt` is a per-pool constant provided during connection setup. The `pass` field survives proxy chains (unlike `job_id` and `params.id` which may be rewritten).

#### 6.3.2 Per-Share Tag Computation

**Miner side (encoding):**

```
nonce[1..3] = payload[0..2]                              // 3 payload bytes
nonce[0]    = HMAC-SHA256(session_key, nonce[1..3])[0]   // 1-byte tag
```

**Pool side (detection):**

```
expected = HMAC-SHA256(session_key, nonce[1..3])[0]
if nonce[0] == expected AND share.difficulty <= ghostDiffMax:
    candidate ghost share -> attempt frame decode
else:
    normal share
```

#### 6.3.3 Properties

| Property | Value |
|----------|-------|
| Payload capacity | 3 bytes/share (unchanged from fixed sentinel) |
| DPI detectability | None -- `nonce[0]` is uniformly distributed over [0x00, 0xFF] |
| Counter sync required | No |
| Proxy-safe | Yes -- key derived from `pass` field |
| False positive rate | 1/256 per share (~0.39%); resolved by frame header validation in the candidate |
| Compute cost (pool) | ~400 ns/share with SHA-NI (~0.4% of 1 core at 10K shares/sec) |
| Memory cost (pool) | ~160 bytes/miner (session_key + lookup) |

#### 6.3.4 False Positive Handling

When a normal share matches the HMAC tag by coincidence (1/256 probability), the pool attempts frame header validation on the payload bytes. A legitimate normal share will not contain a valid VS3 frame header (MAGIC byte at the correct position, valid VERSION, valid FRAG_TOTAL, etc.). The second-stage validation reduces the effective false positive rate to negligible levels.

### 6.4 False Positive Analysis

#### 6.4.1 Fixed Sentinel (0xAA)

With the fixed sentinel, false positives occur when a legitimate mining share happens to have `nonce[0] == 0xAA`:

```
P(false positive per share) = 1/256 = 0.0039
```

At 100 shares/minute, approximately 0.39 shares/minute will falsely match. The pool MUST verify that these candidate ghost shares contain valid VS3 frame headers before processing them as data carriers. The combined false positive rate (sentinel match AND valid frame header) is negligible.

#### 6.4.2 HMAC Sentinel

With the HMAC sentinel, the false positive rate per share is identical (1/256), but the tag is uniformly distributed rather than fixed, eliminating the statistical distinguisher visible to external observers.

---

## 7. Proxy Architecture

The VS proxy enables Visual Stratum communication with standard (non-VS-aware) upstream pools.

### 7.1 Connection Model

```
 VS Miner <--- Stratum TCP ---> VS Proxy <--- Stratum TCP ---> Upstream Pool
                                    |
                              Frame extraction
                              Frame injection
                              Share filtering
```

The proxy MUST:
- Accept Stratum connections from VS-enhanced miners on a configured port.
- Maintain a Stratum connection to the upstream pool.
- Forward standard mining traffic (authorize, subscribe, regular shares) transparently.
- Intercept ghost shares (difficulty <= ghostDiffMax) and extract embedded frames.
- NOT forward ghost shares to the upstream pool (they would be rejected as invalid PoW).
- Inject VS response frames into the downstream pool-to-miner notification stream.

### 7.2 Upstream Sanitization

The proxy MUST sanitize all traffic forwarded to the upstream pool to prevent VS protocol leakage:

1. **Ghost share filtering**: Shares with difficulty <= ghostDiffMax MUST NOT be forwarded.
2. **ntime extension stripping**: If the miner includes a `ntime` TNZX extension field in Monero Stratum submits, the proxy MUST strip it before forwarding.
3. **Nonce normalization**: The proxy SHOULD NOT normalize `nonce[0]` in forwarded real shares. Real mining shares have uniformly distributed nonce values by construction; overwriting any nonce byte would invalidate the share's proof-of-work. Normalization is neither necessary nor safe.

### 7.3 Download Path (Pool -> Miner)

The proxy injects VS response data into the miner's Stratum stream. The injection method depends on the Stratum variant:

For Monero-style Stratum, the proxy MAY include a `vs3` extension field in `job` notifications:

```json
{
    "jsonrpc": "2.0",
    "method": "job",
    "params": {
        "blob": "...",
        "job_id": "...",
        "target": "...",
        "height": 123456,
        "seed_hash": "...",
        "vs3": "<hex-encoded VS3 frame>"
    }
}
```

For Bitcoin-style Stratum, the proxy MAY inject VS3 frames using a `mining.vs3` extension method (a JSON-RPC notification with a single hex-encoded parameter). Pools that do not recognize this method will ignore it per standard JSON-RPC error handling. The proxy SHOULD prefer Monero-style field injection (embedding in job notification) when available, and fall back to the extension method for Bitcoin pools.

> **Note:** This mechanism is specified but not yet validated against
> production Bitcoin pools. Implementations SHOULD verify compatibility
> before deployment.

### 7.4 WebSocket Relay Bootstrap

    Status: Specified, not yet implemented

For bidirectional real-time communication, the proxy MAY offer a WebSocket relay endpoint:

```
 VS Miner <--- WebSocket :8443 ---> VS Proxy (relay)
```

The WebSocket channel carries VS3 frames directly, without Stratum embedding overhead. It provides higher bandwidth and lower latency than the Stratum channel but requires an additional network connection (reducing stealth).

The WebSocket channel is a transport option for VS3 BALANCED and SPEED adaptive modes (Section 10.2).

---

## 8. Cryptographic Design

### 8.1 Key Exchange (X25519)

Visual Stratum uses X25519 Elliptic Curve Diffie-Hellman (ECDH) [RFC 7748] for key exchange.

#### 8.1.1 Key Hierarchy

```
Wallet Seed (256-bit entropy)
    |
    +-- Ed25519 Keypair (signing, identity)
    |     +-- Public Key (32 bytes) -- user identity
    |     +-- Private Key (64 bytes) -- never leaves device
    |
    +-- X25519 Keypair (key exchange)
          +-- Derived via birational map: u = (1 + y) / (1 - y) mod p
```

The Ed25519 to X25519 conversion uses the standard birational map between twisted Edwards and Montgomery curves [Bernstein 2006]:

```
u = (1 + y) / (1 - y) mod p

Where:
  y = Ed25519 public key y-coordinate (sign bit cleared)
  p = 2^255 - 19 (Curve25519 field prime)
```

Implementations MUST clear the sign bit (`key[31] &= 0x7F`) before extracting the y-coordinate.

#### 8.1.2 Shared Secret Computation

Given Alice's private key `a_priv` and Bob's public key `b_pub`:

```
shared_secret = X25519(a_priv, b_pub)   // 32 bytes
```

The shared secret MUST NOT be used directly as an encryption key. It MUST be processed through HKDF (Section 8.3).

### 8.2 Authenticated Encryption (AES-256-GCM)

All message encryption uses AES-256-GCM [NIST SP 800-38D].

| Parameter | Value |
|-----------|-------|
| Algorithm | AES-256-GCM |
| Key size | 256 bits (32 bytes) |
| IV size | 96 bits (12 bytes), random per message |
| Authentication tag | 128 bits (16 bytes) |
| AAD | Protocol context string + message nonce |

#### 8.2.1 Session Encryption

For established sessions (both parties have completed key exchange):

**Encrypt:**
```
nonce      = CSPRNG(16 bytes)           // replay protection nonce
salt       = CSPRNG(32 bytes)           // HKDF salt
iv         = CSPRNG(12 bytes)           // GCM IV
key        = HKDF-SHA256(shared_secret, salt, "tnzx-stego-e2e-v2", 32)
aad        = "tnzx-e2e-v2" || nonce
ciphertext = AES-256-GCM.Encrypt(key, iv, plaintext, aad)
tag        = AES-256-GCM.Tag()

output = nonce(16) || salt(32) || iv(12) || ciphertext || tag(16)
```

**Decrypt:**
```
Parse: nonce(16) || salt(32) || iv(12) || ciphertext || tag(16)
Check: nonce not in seen_nonces (replay protection)
key   = HKDF-SHA256(shared_secret, salt, "tnzx-stego-e2e-v2", 32)
aad   = "tnzx-e2e-v2" || nonce
plaintext = AES-256-GCM.Decrypt(key, iv, ciphertext, tag, aad)
```

Minimum encrypted message size: `16 + 32 + 12 + 1 + 16 = 77 bytes` (for 1 byte of plaintext).

### 8.3 Key Derivation (HKDF-SHA256)

All encryption keys are derived via HKDF-SHA256 [RFC 5869]:

```
key = HKDF(
    hash    = SHA-256,
    IKM     = shared_secret,           // 32 bytes from X25519
    salt    = random(32 bytes),        // fresh per message
    info    = "tnzx-stego-e2e-v2",     // context string
    length  = 32                       // 256-bit output key
)
```

A fresh random salt per message ensures unique derived keys even when the same shared secret is reused across messages. This provides forward secrecy at the message level within an established session.

### 8.4 Perfect Forward Secrecy

For one-shot messages (no pre-established session), the sender generates an ephemeral X25519 keypair per message:

```
1. Generate ephemeral keypair: (e_priv, e_pub)
2. Compute shared secret: s = X25519(e_priv, recipient_pub)
3. Derive key: key = HKDF-SHA256(s, random_salt, "tnzx-stego-e2e-v2", 32)
4. Generate: nonce = CSPRNG(16), iv = CSPRNG(12)
5. Compute AAD: aad = "tnzx-oneshot-v2" || nonce || e_pub
6. Encrypt: ciphertext = AES-256-GCM.Encrypt(key, iv, plaintext, aad)
7. Output: nonce(16) || e_pub(32) || salt(32) || iv(12) || ciphertext || tag(16)
8. DISCARD e_priv immediately
```

> **Implementation note:** The current reference implementation uses the same HKDF
> info string (`"tnzx-stego-e2e-v2"`) for both session and one-shot encryption.
> Domain separation is achieved through the use of ephemeral keypairs in one-shot
> mode, which produce unique shared secrets per message. Future revisions of this
> specification SHOULD introduce a distinct info string (e.g., `"tnzx-oneshot-v2"`)
> for explicit domain separation.

Note that the AAD prefix for one-shot encryption (`"tnzx-oneshot-v2"`) is distinct from the session AAD prefix (`"tnzx-e2e-v2"`). The HKDF info string and the AAD prefix serve different roles: the info string parameterizes key derivation, while the AAD prefix binds the ciphertext to its encryption context. The AAD distinction is preserved even though the HKDF info string is currently shared.

Minimum one-shot message size: `16 + 32 + 32 + 12 + 1 + 16 = 109 bytes` (for 1 byte of plaintext).

Compromise of the recipient's long-term key cannot decrypt past one-shot messages because the ephemeral private key no longer exists.

### 8.5 Replay Protection

Each message includes a 128-bit (16-byte) cryptographic nonce. The receiver maintains a nonce cache:

| Parameter | Value |
|-----------|-------|
| Nonce size | 128 bits (16 bytes) |
| Cache TTL | 300,000 ms (5 minutes) |
| Eviction | Automatic expiry; scan on each decrypt |

**Algorithm:**
```
function checkNonce(nonce):
    hex = nonce.toHex()
    now = currentTime()

    // Evict expired entries
    for each (key, timestamp) in seenNonces:
        if now - timestamp > MAX_NONCE_AGE_MS:
            remove(key)

    if hex in seenNonces:
        return REJECT  // replay detected

    seenNonces[hex] = now
    return ACCEPT
```

Implementations MUST reject messages with duplicate nonces within the TTL window. One-shot decryption MUST maintain its own separate nonce tracker (module-level singleton).

---

## 9. Chain Adaptation

### 9.1 Monero Stratum

Monero Stratum uses a simplified JSON-RPC protocol with the following relevant messages:

#### 9.1.1 Login / Subscribe

```json
{
    "id": 1,
    "method": "login",
    "params": {
        "login": "<wallet_address>",
        "pass": "<password>",
        "agent": "tnzxminer/1.0"
    }
}
```

Response includes `job` parameters with `blob`, `job_id`, `target`, `height`, `seed_hash`.

#### 9.1.2 Share Submit (Standard Monero)

```json
{
    "id": 2,
    "method": "submit",
    "params": {
        "id": "<worker_id>",
        "job_id": "<job_id>",
        "nonce": "<8 hex chars = 4 bytes>",
        "result": "<64 hex chars = 32 bytes>"
    }
}
```

Note: Standard Monero `mining.submit` contains ONLY `nonce`, `result`, `job_id`, and `id`. There is NO `ntime` field and NO `extranonce2` field.

#### 9.1.3 Share Submit (VS3-Monero Extension)

A VS-enhanced miner adds the `ntime` TNZX extension field:

```json
{
    "id": 2,
    "method": "submit",
    "params": {
        "id": "<worker_id>",
        "job_id": "<job_id>",
        "nonce": "aa48656c",
        "result": "0000000000000000000000000000000000000000000000000000000000000000",
        "ntime": "65b26c6f"
    }
}
```

The `ntime` field is a TNZX extension; standard pools MUST ignore unknown fields per JSON-RPC convention. The `result` field contains all zeros for ghost shares (no valid PoW).

### 9.2 Bitcoin Stratum

Bitcoin Stratum (v1) uses the following share submit format:

```json
{
    "id": 4,
    "method": "mining.submit",
    "params": [
        "<worker_name>",
        "<job_id>",
        "<extranonce2>",
        "<ntime>",
        "<nonce>"
    ]
}
```

Both `extranonce2` and `ntime` are standard fields in Bitcoin Stratum. VS2 and VS3-Generic profiles embed data in these fields without protocol extensions.

### 9.3 Auto-Detection

A VS-aware pool or proxy can detect the Stratum variant from the login/subscribe exchange:

| Indicator | Stratum Variant |
|-----------|----------------|
| `method: "login"` with `params.login` | Monero Stratum |
| `method: "mining.subscribe"` | Bitcoin-style Stratum |
| `extranonce2_size` in subscribe response | Bitcoin-style (size determines V2/V3-Generic capacity) |

---

## 10. Multi-Channel Transport (VS3)

    Status: Specified, not yet implemented (except L1 Stratum channel)

VS3 extends the transport layer to support four parallel communication channels.

### 10.1 Channel Types

| Channel | Protocol | Bandwidth | Direction | Cover Story | Stealth |
|---------|----------|-----------|-----------|-------------|---------|
| L1 | Stratum share embedding | 5-256 B/share | Upload (miner -> pool) | Normal mining | 5/5 |
| L2 | HTTP/2 streams | ~100 KB/s | Bidirectional | Pool API calls | 4/5 |
| L3 | WebSocket | ~50 KB/s | Bidirectional | Real-time stats | 4/5 |
| L4 | PNG LSB steganography | ~45 KB/s | Download (pool -> miner) | Dashboard charts | 5/5 |

Combined design target bandwidth: ~195 KB/s (all channels active).

### 10.1.1 L4: PNG LSB Channel

PNG chart images (400x300 pixels, 24-bit RGB) serve as the download channel:

```
Capacity = width * height * 3 bits = 400 * 300 * 3 = 360,000 bits = 45,000 bytes
```

The embedding order MUST be pseudo-randomly permuted using a Fisher-Yates shuffle seeded by the session key. Without the key, payload bits cannot be located.

Both encoding and decoding MUST process the full image capacity (constant-time), padding unused space with encrypted random data. This prevents timing side-channel leakage of actual payload size.

### 10.2 Adaptive Modes (ANON, BALANCED, SPEED)

| Mode | Channels | Use Case | Stealth |
|------|----------|----------|---------|
| **ANON** | L1 only (Stratum) | Private chat, escrow, sensitive data | Maximum |
| **BALANCED** | L1 + bonus channels (delayed) | DNS queries, marketplace | High |
| **SPEED** | All channels parallel | File transfer, web hosting | Medium |

#### 10.2.1 Automatic Mode Selection

| Message Type | Mode |
|-------------|------|
| Chat, presence, escrow operations | ANON |
| DNS queries, marketplace listings | BALANCED |
| File transfers, web hosting | SPEED |

Implementations MAY allow manual mode override.

#### 10.2.2 Multi-Channel Transport Header

When fragments are routed across multiple channels (BALANCED or SPEED modes), each fragment is wrapped in a 13-byte transport header that extends the base 8-byte frame header with channel routing metadata:

```
  0                   1                   2                   3
  0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 |     MAGIC     |    VERSION    |    CHANNEL    |   MESSAGE_ID  |
 |     (0xAA)    |    (0x03)     |   (L1-L4)     |   (byte 3)    |
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 |            MESSAGE_ID         |         FRAG_INDEX            |
 |   (bytes 4-5)  (byte 6)      |   (bytes 7-8)                 |
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 |        TOTAL_FRAGS            |          DATA_LEN             |
 |   (bytes 9-10)                |   (bytes 11-12)               |
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

| Offset | Size | Field | Notes |
|--------|------|-------|-------|
| 0 | 1 | MAGIC | `0xAA` |
| 1 | 1 | VERSION | `0x03` |
| 2 | 1 | CHANNEL | Channel ID: 1=L1, 2=L2, 3=L3, 4=L4 |
| 3-6 | 4 | MESSAGE_ID | 32-bit message ID, big-endian (wider than base 8-byte header) |
| 7-8 | 2 | FRAG_INDEX | 16-bit fragment index, big-endian |
| 9-10 | 2 | TOTAL_FRAGS | 16-bit total fragments, big-endian |
| 11-12 | 2 | DATA_LEN | 16-bit payload length, big-endian |

On the Stratum-only (ANON) path, the base 8-byte header from Section 4.1 is used directly. The 13-byte transport header applies ONLY to multi-channel routing.

> **Compatibility note:** The 13-byte multi-channel transport header is a
> DISTINCT framing layer used ONLY on L2 (HTTP/2) and L3 (WebSocket) channels.
> It is NOT used on the L1 (Stratum) channel, which uses the 8-byte frame
> header defined in Section 4.1. Receivers distinguish the two headers by
> transport context: frames arriving via Stratum share extraction always use
> the 8-byte header; frames arriving via WebSocket or HTTP/2 always use the
> 13-byte header. The two header formats are never mixed on the same channel.
> 
> **Status:** The multi-channel transport header is specified for future
> implementation. Current implementations use only the 8-byte frame header.

The receiver MUST reassemble fragments from any channel, in any order.

### 10.3 Timing Decorrelation

In BALANCED mode, fragments sent on non-primary (bonus) channels MUST be delayed by a random interval:

```
delay = CSPRNG_uniform(delta_min, delta_max)

Where:
  delta_min = 500 ms
  delta_max = 3000 ms
```

The delay MUST be generated using a CSPRNG (e.g., `crypto.randomInt()`), NOT `Math.random()` or other non-cryptographic PRNGs.

This prevents an observer who monitors both Stratum and WebSocket traffic from correlating fragments of the same message across channels.

#### 10.3.1 Dummy Traffic

Chaff packets MUST be injected at configurable intervals on random channels:

| Parameter | Default | Range |
|-----------|---------|-------|
| Interval min | 10 s | >= 5 s (hard minimum) |
| Interval max | 30 s | configurable |
| Source | CSPRNG | REQUIRED |

Dummy packets:
- Use a reserved message ID: `0xFFFF` (8-byte header) or `0xFFFFFFFF` (13-byte header).
- Contain CSPRNG random data as payload.
- Are indistinguishable from real fragments to an observer without the decryption key.

Implementations MUST NOT set the interval minimum below 5 seconds to prevent denial-of-service via configuration injection.

### 10.4 Bandwidth Summary

| Application | Requirement | VS3 ANON | VS3 BALANCED | VS3 SPEED |
|-------------|-------------|----------|-------------|-----------|
| Text chat | < 1 KB/s | Yes | Yes | Yes |
| File transfer | 10-50 KB/s | No | Partial | Yes |
| Voice (Opus) | 6-12 KB/s | No | Partial | Theoretical* |
| Audio streaming | 16-32 KB/s | No | Partial | Yes |
| Web hosting | 45 KB/s | No | No | Yes |
| Low-res video | 50-100 KB/s | No | No | Yes (limited) |

\* Voice support in SPEED mode is theoretically sufficient but not yet implemented.

### 10.5 Compression

LZ4 compression [Collet 2011] is applied before encryption for payloads exceeding a threshold:

```
Processing pipeline:
  Plaintext -> LZ4 Compress (if size > threshold) -> Pad to 32-byte boundary
            -> AES-256-GCM Encrypt -> Fragment -> Send
```

| Parameter | Value | Rationale |
|-----------|-------|-----------|
| Compression threshold | 50 bytes | Below 50 bytes, LZ4 overhead (~14 bytes) exceeds compression gain |
| Padding granularity | 32 bytes | CRIME-style compression ratio attack mitigation |
| LZ4 header magic | `0x4C 0x5A 0x34 0x01` | Used by receiver to detect compressed payloads |

The receiver detects compression by checking the first 4 bytes of the decrypted payload for the LZ4 header magic. If present, decompress before processing.

---

## 11. Security Considerations

### 11.1 Threat Model

Visual Stratum considers four adversary classes:

| Adversary | Capabilities |
|-----------|-------------|
| **Passive network observer** | Monitors all network traffic between miner and pool. Can perform statistical analysis on Stratum field distributions. |
| **Active network attacker** | Can modify, delay, drop, or replay packets. Can perform man-in-the-middle attacks on unencrypted Stratum connections. |
| **Malicious pool operator** | Has access to all share submissions and server-side state. Can selectively drop shares. Cannot decrypt E2E-encrypted payloads. |
| **Compromised endpoint** | Has access to one party's device (keys, plaintext). Bypasses all network-level protections. |

### 11.2 Undetectability Analysis

The undetectability of the Stratum embedding is analyzed per field:

#### 11.2.1 Nonce Channel (V1, V2, V3) -- Strong

Mining nonces are uniformly random 32-bit values. The LSB of a random value is uniformly distributed. VS replaces LSBs with AES-256-GCM encrypted payload bytes, which are also uniformly distributed.

For a random nonce n and encrypted payload byte b:

```
H(n mod 2^8) = H(b) = 8 bits (maximum entropy)
```

No statistical test can distinguish the modified field from an unmodified one, because both contain maximum-entropy data. This argument holds for any cipher satisfying IND-CPA (indistinguishability under chosen-plaintext attack), which AES-256-GCM satisfies.

**Conclusion:** The nonce embedding channel is statistically undetectable under the entropy-equivalence argument.

#### 11.2.2 Extranonce2 Channel (V2, V3-Generic) -- Weaker

Standard Bitcoin miners iterate extranonce2 as a sequential counter (0x00000000, 0x00000001, ...). Its distribution is monotonically increasing, NOT uniform. Replacing the last bytes with uniformly random encrypted payload creates a distributional shift detectable by a sufficiently motivated observer monitoring extranonce2 values across many shares.

**Conclusion:** The extranonce2 channel is detectable by a sufficiently motivated observer with access to the full share stream. The entropy-equivalence argument does NOT apply with full strength.

#### 11.2.3 Ntime Channel (V2, V3) -- Weaker

The `ntime` field contains a Unix timestamp. The high 16 bits are preserved; the low 16 bits are replaced with payload. Legitimate low bits exhibit temporal structure (incrementing with real clock time), while encrypted payload bytes are uniformly distributed.

**Conclusion:** The ntime embedding creates a potential statistical distinguisher. The undetectability of the ntime channel is a design target, not a formal guarantee.

#### 11.2.4 Ntime in Monero -- Distinguishable

The `ntime` field does NOT exist in standard Monero Stratum. Its presence in a Monero `mining.submit` is itself a distinguishing signal to a sufficiently detailed Stratum protocol analyzer.

**Conclusion:** The TNZX ntime extension field is detectable by protocol fingerprinting. It is NOT undetectable.

#### 11.2.5 Sentinel Byte (V3-Monero Ghost Shares) -- Distinguishable

Ghost shares use `nonce[0] = 0xAA` with probability 1, while legitimate shares have `nonce[0]` uniformly distributed. An adversary monitoring `nonce[0]` across O(256) shares can detect ghost share usage with high confidence.

**Conclusion:** The fixed sentinel is trivially detectable. The HMAC mitigation (Section 6.3) eliminates this specific distinguisher but does NOT address the inherent PoW-invalidity of ghost shares.

#### 11.2.6 Ghost Share PoW Invalidity -- Inherent

Ghost shares have invalid PoW (their `result` does not meet the pool's difficulty target). An adversary who parses Stratum traffic and verifies PoW can identify ghost shares regardless of sentinel design. This is inherent to the ghost share concept and cannot be mitigated at the protocol level.

**Mitigation:** At the minimum difficulty (ghostDiffMax = 1), the frequency of ghost shares relative to legitimate shares can be controlled by the miner's submission rate. The overall traffic pattern should be indistinguishable from a miner experiencing occasional sub-difficulty submissions (which occur naturally during vardiff transitions).

#### 11.2.7 PNG Channel (VS1/L4) -- Design Target

PNG pixel values have structure (unlike Stratum nonces). The undetectability of LSB embedding in images is an empirical claim requiring steganalysis validation.

Design targets:
- Chi-square test: target chi-squared < 3.84 (p > 0.05 at 1 d.f.)
- RS analysis: target difference < 0.1
- Entropy: 8.0 bits per byte in LSB plane

**These are design targets, NOT validated results.** Formal steganalysis evaluation has not yet been performed. The Stratum channel SHOULD be preferred for high-risk use cases.

### 11.3 Limitations and Non-Guarantees

1. **Bandwidth depends on hashrate.** Low-hashrate miners have slow upload and high latency in ANON mode.
2. **Latency depends on share rate.** If shares are infrequent, message latency is proportionally high.
3. **Pool trust.** A malicious pool can selectively drop ghost shares. It CANNOT read them (E2E encryption). Dropping valid shares reduces the pool's own mining revenue.
4. **Long-term traffic analysis.** An adversary observing traffic patterns over weeks may detect correlations between message activity and share submission patterns. Dummy traffic (Section 10.3.1) mitigates but does not eliminate this risk.
5. **Endpoint compromise.** If a device is compromised, all cryptographic protections are bypassed. This limitation is shared by all communication systems.
6. **No quantum resistance.** X25519 and AES-256 are vulnerable to quantum computing. Post-quantum key exchange (e.g., CRYSTALS-Kyber) is planned for a future version.
7. **No independent audit.** This protocol has not undergone independent third-party security audit. Multiple rounds of internal review have been conducted.
8. **Ghost shares are inherently distinguishable** from legitimate PoW shares by an adversary capable of verifying share validity. The protocol relies on the assumption that most adversaries (ISPs, firewalls) parse Stratum traffic at the JSON level but do NOT verify PoW.

---

## 12. Test Vectors

All test vectors are published in machine-readable JSON format in the `test-vectors/` directory of the tnzx-protocol repository.

### 12.1 VS1 Vectors

File: `test-vectors/vs1-vectors.json`

| Vector | Description |
|--------|-------------|
| `basic_embed_extract` | Embed/extract "Hello World!" via LSB in PNG |
| `ecdh_key_exchange` | RFC 7748 X25519 reference vector |
| `pixel_permutation` | Deterministic Fisher-Yates shuffle order for LSB insertion |
| `capacity_calculation` | 400x300 RGB PNG yields 44,984 bytes usable capacity |
| `header_structure` | VS1 16-byte header: magic(4) + version(1) + type(1) + length(4) + checksum(4) + reserved(2) |

### 12.2 VS2 Vectors

File: `test-vectors/vs2-vectors.json`

#### 12.2.1 Stratum Embedding

```
Test: v2_standard_3bytes
  Input:
    nonce_hex:       "a1b2c3d4e5f60000"
    extranonce2_hex: "00000000"
    data_bytes:      [0xAA, 0xBB, 0xCC]

  Expected output:
    nonce:       "a1b2c3d4e5f60a0a"
    extranonce2: "0000bbcc"

  Algorithm:
    byte 0 (0xAA) -> nonce LSB nibbles:
      nonce[6] = (0x00 & 0xF0) | ((0xAA >> 4) & 0x0F) = 0x0A
      nonce[7] = (0x00 & 0xF0) | (0xAA & 0x0F)        = 0x0A
    byte 1 (0xBB) -> extranonce2[2] = 0xBB
    byte 2 (0xCC) -> extranonce2[3] = 0xCC
```

#### 12.2.2 Mining Gate State Machine

```
Test: State transitions
  T+0s:    0 shares, state = INACTIVE
  T+5s:    1 share,  state = GRACE
  T+30s:   3 shares, state = ACTIVE (channel opens)
  T+900s:  hashrate = 5 H/s, state = SUSPENDED (hashrate < MIN_HASHRATE 10 H/s)
```

### 12.3 VS3 Vectors

File: `test-vectors/vs3-vectors.json`

#### 12.3.1 VS3-Monero Encoding

```
Test: monero_hello
  Input:
    ntime_hex:  "65b2a100"
    payload:    [0x48, 0x65, 0x6C, 0x6C, 0x6F]   ("Hello")

  Expected:
    nonce: "aa48656c"
    ntime: "65b26c6f"

  Extraction:
    payload[0] = nonce[1] = 0x48 ('H')
    payload[1] = nonce[2] = 0x65 ('e')
    payload[2] = nonce[3] = 0x6C ('l')
    payload[3] = ntime[2] = 0x6C ('l')
    payload[4] = ntime[3] = 0x6F ('o')
```

```
Test: monero_zero
  Input:  ntime_hex: "65b2a100", payload: [0,0,0,0,0]
  Expected: nonce: "aa000000", ntime: "65b20000"

Test: monero_ff
  Input:  ntime_hex: "65b2a100", payload: [0xFF,0xFF,0xFF,0xFF,0xFF]
  Expected: nonce: "aaffffff", ntime: "65b2ffff"

Test: monero_roundtrip
  Input:  ntime_hex: "67890abc", payload: [0x12,0x34,0x56,0x78,0x9A]
  Expected: nonce: "aa123456", ntime: "6789789a"

Test: monero_ntime_hi_preserved
  Input:  ntime_hex: "deadbeef", payload: [0xAA,0xBB,0xCC,0xDD,0xEE]
  Expected: nonce: "aaaabbcc", ntime: "deadddee"
  Note: ntime high word "dead" is preserved regardless of payload.
```

#### 12.3.2 VS3-Generic Encoding

```
Test: generic_reference
  Input:
    nonce_hex:       "a1b2c3d4e5f67800"
    extranonce2_hex: "0000000000000000"
    ntime_hex:       "65b2a100"
    payload:         [0xAA, 0xBB, 0xCC, 0xDD, 0xEE, 0xFF, 0x11]

  Expected:
    nonce:       "a1b2c3d4e5f67a0a"
    extranonce2: "00000000bbccddee"
    ntime:       "65b2ff11"

  Algorithm:
    payload[0] = 0xAA: high nibble = 0x0A -> nonce[6] low = 0xA
                        low nibble  = 0x0A -> nonce[7] low = 0xA
    payload[1..4] = [0xBB, 0xCC, 0xDD, 0xEE] -> extranonce2 last 4 bytes
    payload[5..6] = [0xFF, 0x11] -> ntime last 2 bytes
```

#### 12.3.3 Frame Format

```
Test: single_fragment_text
  Input: "Hi" (2 bytes UTF-8), msgId = 0x1234, type = TEXT (0x01)

  Frame (hex): AA 03 01 12 34 00 01 02 48 69
                |  |  |  |     |  |  |  |  |
                |  |  |  |     |  |  |  +--+-- payload: "Hi"
                |  |  |  |     |  |  +-------- payload_len: 2
                |  |  |  |     |  +----------- frag_total: 1
                |  |  |  |     +-------------- frag_index: 0
                |  |  |  +-------------------- msg_id: 0x1234 (BE)
                |  |  +----------------------- type: TEXT
                |  +-------------------------- version: VS3
                +----------------------------- magic: 0xAA

  Total frame size: 10 bytes
  Shares required (V3-Monero, 5B/share): ceil(10/5) = 2
```

### 12.4 Cryptographic Vectors

#### 12.4.1 X25519 Key Exchange (RFC 7748)

```
Alice Private: 77076d0a7318a57d3c16c17251b26645df4c2f87ebc0992ab177fba51db92c2a
Alice Public:  8520f0098930a754748b7ddcb43ef75a0dbf3a0d26381af4eba4a98eaa9b4e6a
Bob Private:   5dab087e624a8a4b79e17f8b83800ee66f3bb1292618b6fd1c2f8b27ff88e0eb
Bob Public:    de9edb7d7b7dc1b4d35b61c2ece435373f8343c85b78674dadfc7e146f882b4f
Shared Secret: 4a5d9d5ba4ce2de1728e3bf480350f25e07e21c947d19e3376f09b3c1e161742
```

#### 12.4.2 AES-256-GCM

The reference implementation uses AES-256-GCM (NIST SP 800-38D). For standard interoperability test vectors, see NIST SP 800-38D Appendix B, Test Case 4 (256-bit key).

#### 12.4.3 HKDF-SHA256

The reference implementation uses HKDF-SHA256 (RFC 5869) with the info string `"tnzx-stego-e2e-v2"` for both session and one-shot encryption (see Section 8.4 for rationale). The AAD prefix differs between modes: `"tnzx-e2e-v2"` for session, `"tnzx-oneshot-v2"` for one-shot. For standard HKDF test vectors, see RFC 5869 Appendix A.

---

## 13. References

### Normative References

- [RFC2119] Bradner, S., "Key words for use in RFCs to Indicate Requirement Levels", BCP 14, RFC 2119, March 1997.
- [RFC5869] Krawczyk, H. and Eronen, P., "HMAC-based Extract-and-Expand Key Derivation Function (HKDF)", RFC 5869, May 2010.
- [RFC7748] Langley, A., Hamburg, M., and Turner, S., "Elliptic Curves for Security", RFC 7748, January 2016.
- [NIST-GCM] Dworkin, M., "Recommendation for Block Cipher Modes of Operation: Galois/Counter Mode (GCM) and GMAC", NIST SP 800-38D, November 2007.

### Informative References

- [Bernstein2006] Bernstein, D.J., "Curve25519: New Diffie-Hellman Speed Records", PKC 2006.
- [Fridrich2009] Fridrich, J., "Steganography in Digital Media: Principles, Algorithms, and Applications", Cambridge University Press, 2009.
- [Back2002] Back, A., "Hashcash -- A Denial of Service Counter-Measure", 2002.
- [Partala2018] Partala, J., "Provably secure covert communication on blockchain", Cryptography, 2(3):18, 2018.
- [Frkat2020] Frkat, D., Annessi, R., and Zseby, T., "Chainchannels: Private botnet communication over public blockchains", IEEE ICBC, 2020.
- [Cao2023] Cao, Y., et al., "A survey of blockchain-based information hiding", Journal of Information Security and Applications, 71:103385, 2023.
- [StegoTorus] Weinberg, Z., et al., "StegoTorus: A Camouflage Proxy for the Tor Anonymity System", ACM CCS, 2012.
- [FreeWave] Houmansadr, A., Brubaker, C., and Shmatikov, V., "The Parrot is Dead: Observing Unobservable Network Communications", IEEE S&P, 2013.
- [DeltaShaper] Barradas, D., Santos, N., and Rodrigues, L., "DeltaShaper: Enabling Unobservable Censorship-Resistant TCP Tunneling over Videoconferencing Streams", PoPETs, 2017.
- [Collet2011] Collet, Y., "LZ4 -- Extremely Fast Compression", 2011.
- [CRIME] Rizzo, J. and Duong, T., "The CRIME Attack", Ekoparty, 2012.
- [RFC6455] Fette, I. and Melnikov, A., "The WebSocket Protocol", RFC 6455, December 2011.
- [RFC7540] Belshe, M., Peon, R., and Thomson, M., "Hypertext Transfer Protocol Version 2 (HTTP/2)", RFC 7540, May 2015.
- [RFC8439] Nir, Y. and Langley, A., "ChaCha20 and Poly1305 for IETF Protocols", RFC 8439, June 2018.

---

## Appendix A -- Protocol Constants

```
// ===== Magic Bytes =====
MAGIC_BYTE              = 0xAA

// ===== Protocol Versions =====
VERSION_V1              = 0x01    // Nonce LSB only (1 B/share)
VERSION_V2              = 0x02    // + extranonce2 (3 B/share)
VERSION_V3              = 0x03    // Ghost shares + ntime ext (5 B/share Monero, 7 B/share generic)
VERSION_V3_BURST        = 0x04    // Extended extranonce2 (~200 B/share)
VERSION_V3_GHOST        = 0x05    // Difficulty-1 cover shares (~200 B/share)
VERSION_V3_TURBO        = 0x06    // Worker password field (~256 B/share)

// ===== Message Types =====
MSG_TEXT                = 0x01
MSG_ACK                 = 0x02
MSG_PING                = 0x03
MSG_KEY_EXCHANGE        = 0x04
MSG_ENCRYPTED           = 0x05
MSG_HASHCASH            = 0x06

// ===== Frame Parameters =====
HEADER_SIZE             = 8       // bytes
MAX_FRAGMENT_SIZE       = 128     // bytes per fragment payload
MAX_TOTAL_FRAGMENTS     = 50      // fragments per logical message
MAX_MESSAGE_SIZE        = 6400    // MAX_FRAGMENT_SIZE * MAX_TOTAL_FRAGMENTS

// ===== Encoding Parameters =====
BYTES_PER_SHARE_V1      = 1
BYTES_PER_SHARE_V2      = 3
BYTES_PER_SHARE_V3_MONO = 5       // VS3-Monero (3B nonce + 2B ntime)
BYTES_PER_SHARE_V3_GEN  = 7       // VS3-Generic (1B nonce + 4B extranonce2 + 2B ntime)

// ===== Cryptographic Constants =====
KEY_LENGTH              = 32      // 256 bits (AES key, HKDF output, X25519 keys)
IV_LENGTH               = 12      // 96 bits (GCM standard)
AUTH_TAG_LENGTH         = 16      // 128 bits (GCM tag)
SALT_LENGTH             = 32      // HKDF salt
NONCE_LENGTH            = 16      // 128 bits (replay protection nonce)
FIELD_PRIME             = 2^255 - 19   // Curve25519 field prime

// ===== Security Limits =====
MAX_PENDING_MESSAGES    = 1000
MAX_COMPLETED_MESSAGES  = 500
MESSAGE_TIMEOUT_MS      = 300000  // 5 minutes
MAX_NONCE_AGE_MS        = 300000  // 5 minutes (replay window)

// ===== Compression =====
COMPRESS_THRESHOLD      = 50      // min bytes for LZ4 compression
PAD_GRANULARITY         = 32      // CRIME mitigation padding boundary
LZ4_MAGIC               = 0x4C5A3401  // "LZ4\x01"

// ===== Mining Gate Defaults =====
WINDOW_SIZE_MS          = 600000  // 10 minutes
MIN_HASHRATE            = 10      // H/s (absolute suspension threshold)
GRACE_PERIOD_MS         = 120000  // 2 minutes
MIN_ACTIVATION_SHARES   = 3
COOLDOWN_MS             = 300000  // 5 minutes
CHECK_INTERVAL_MIN_MS   = 60000   // 1 minute
CHECK_INTERVAL_MAX_MS   = 180000  // 3 minutes

// ===== Dummy Traffic =====
DUMMY_MSG_ID_8B         = 0xFFFF      // 8-byte header reserved ID
DUMMY_MSG_ID_13B        = 0xFFFFFFFF  // 13-byte header reserved ID
DUMMY_INTERVAL_MIN_S    = 10          // seconds (hard min: 5s)
DUMMY_INTERVAL_MAX_S    = 30          // seconds

// ===== HMAC Ghost Detection =====
HMAC_INFO_STRING        = "tnzx-ghost-v1"
HMAC_ALGORITHM          = "SHA-256"

// ===== Encryption Context Strings =====
SESSION_INFO            = "tnzx-stego-e2e-v2"
ONESHOT_INFO            = "tnzx-stego-e2e-v2"  // NOTE: same as SESSION_INFO in current impl; see Section 8.4
SESSION_AAD_PREFIX      = "tnzx-e2e-v2"
ONESHOT_AAD_PREFIX      = "tnzx-oneshot-v2"    // AAD prefix IS distinct from session

// ===== Timing =====
BALANCED_DELAY_MIN_MS   = 500
BALANCED_DELAY_MAX_MS   = 3000
```

---

## Appendix B -- Example Stratum JSON Exchanges

### B.1 Monero -- VS3 Ghost Share Session

```
// Step 1: Miner connects and logs in
--> {"id":1,"method":"login","params":{"login":"4...wallet...","pass":"x","agent":"tnzxminer/1.0"}}
<-- {"id":1,"result":{"id":"abc123","job":{"blob":"...","job_id":"j1","target":"...","height":3000000,"seed_hash":"..."},"status":"OK"}}

// Step 2: Miner submits a regular mining share (valid PoW)
--> {"id":2,"method":"submit","params":{"id":"abc123","job_id":"j1","nonce":"1a2b3c4d","result":"0000000f..."}}
<-- {"id":2,"result":{"status":"OK"}}

// Step 3: Miner submits a VS3 ghost share (no valid PoW)
//   nonce[0] = 0xAA (sentinel), nonce[1..3] = payload bytes
//   ntime = TNZX extension field (not standard Monero)
//   result = all zeros (ghost share)
--> {"id":3,"method":"submit","params":{"id":"abc123","job_id":"j1","nonce":"aa48656c","result":"0000000000000000000000000000000000000000000000000000000000000000","ntime":"65b26c6f"}}
<-- {"id":3,"result":{"status":"OK"}}

// Pool internally: detects nonce[0]==0xAA, difficulty <= ghostDiffMax
// Extracts payload: [0x48, 0x65, 0x6C, 0x6C, 0x6F] = "Hello"
// Does NOT credit mining reward for this share.

// Step 4: Pool sends response data in next job notification (vs3 extension)
<-- {"jsonrpc":"2.0","method":"job","params":{"blob":"...","job_id":"j2","target":"...","height":3000001,"seed_hash":"...","vs3":"aa030512340001054869"}}

// Miner extracts "vs3" field, decodes VS3 frame, reassembles message.
```

### B.2 Bitcoin-style -- V2 Embedding

```
// Step 1: Mining subscribe
--> {"id":1,"method":"mining.subscribe","params":["tnzxminer/1.0"]}
<-- {"id":1,"result":[[["mining.notify","subscription_id"]],"extranonce1_hex",4]}
//  extranonce2_size = 4 bytes

// Step 2: Mining authorize
--> {"id":2,"method":"mining.authorize","params":["worker.1","password"]}
<-- {"id":2,"result":true}

// Step 3: Pool sends work
<-- {"id":null,"method":"mining.notify","params":["job_id","prevhash","coinbase1","coinbase2",["merkle1","merkle2"],"version","nbits","ntime",true]}

// Step 4: Miner submits V2-encoded share
//   nonce LSB nibbles carry byte 0 (0xAA)
//   extranonce2 last 2 bytes carry bytes 1-2 (0xBB, 0xCC)
--> {"id":3,"method":"mining.submit","params":["worker.1","job_id","0000bbcc","ntime_hex","...nonce_with_aa_nibbles..."]}
<-- {"id":3,"result":true}
```

### B.3 Mining Gate Lifecycle

```
Timeline of Mining Gate state transitions:

T+0s     Miner connects.
         State: INACTIVE, Channel: CLOSED

T+5s     Miner submits share #1 (valid PoW, diff=10000).
         recordShare() -> State: GRACE, Channel: CLOSED

T+15s    Miner submits share #2 (valid PoW, diff=10000).
         State: GRACE (2 < MIN_ACTIVATION=3)

T+25s    Miner submits share #3 (valid PoW, diff=10000).
         State: ACTIVE, Channel: OPEN
         VS communication now possible.

T+30s    Miner sends ghost share with VS3 payload.
         Ghost share NOT counted for Mining Gate.
         Data extracted and processed.

T+700s   Periodic check fires (random interval).
         hashrate = total_difficulty / elapsed = 5 H/s
         5 H/s < MIN_HASHRATE (10 H/s)
         State: SUSPENDED, Channel: CLOSED
         suspendedAt = T+700s

T+1000s  Cooldown elapsed (300s since suspension).
         Miner resumes mining, submits shares.
         hashrate = 15 H/s >= MIN_HASHRATE (10 H/s)
         State: ACTIVE, Channel: OPEN
```

---

## Appendix C -- Changelog

| Version | Date | Changes |
|---------|------|---------|
| draft-00a | 2026-03-31 | Errata: fixed V2 test vector nonce, aligned oneshot HKDF info to implementation, simplified Mining Gate threshold to absolute hashrate, resolved TBD items, clarified VERSION byte semantics, added robustness principle for PAYLOAD_LEN. |
| draft-00 | 2026-03-31 | Initial draft. Consolidation of VS1/VS2/VS3 specifications, paper, and reference implementation into a single RFC-style document. |

---

*End of specification.*
