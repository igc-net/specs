# `igc-net`: IGC Publishing and Announcement Protocol

**Status:** Working draft  
**Version:** 0.1  
**Date:** 2026-04-03

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be
interpreted as described in [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119).

This document specifies the wire format, gossip protocol, blob model, node
identity, fetch flow, and hash verification rules for `igc-net`. It is
language-agnostic and normative.

- Metadata schema and analytics extension: `specs_meta.md`
- Functional and non-functional requirements: `requirements.md`

## 1. Cryptographic Primitives

### 1.1 Hash Function

All content addressing in `igc-net` uses **BLAKE3**.

- digests are 32 bytes / 256 bits
- wire-format digests MUST be encoded as 64-character lowercase hexadecimal
  strings
- implementations MUST use a production BLAKE3 library

### 1.2 Node Identity

Each `igc-net` node has an Ed25519 key pair. The 32-byte public key is the
node's stable network identity.

- public keys in wire formats MUST be 64-character lowercase hex strings
- the secret key MUST persist across restarts
- implementations MUST NOT generate a new identity on every startup

## 2. Blob Model

Every published flight consists of exactly two content-addressed blobs:

| Blob | Content | Hash field |
|---|---|---|
| IGC blob | Raw `.igc` file bytes, unmodified | `igc_hash` |
| Metadata blob | UTF-8 JSON conforming to `specs_meta.md` Part I | `meta_hash` |

Both blobs are addressed by BLAKE3. The pair `(igc_hash, meta_hash)` identifies
the published flight version on the network.

## 3. Gossip Protocol

### 3.1 Topic ID

All compliant nodes MUST join a single well-known gossip topic derived from:

```text
topic_id = BLAKE3("igc-net/announce/v1")[0..32]
```

The canonical hex value is:

```text
2f06567e5d7148b56349a753f8b407fbc35b2f0d90ff366ff2673d060245dda9
```

Implementations should embed this value as a compile-time constant.

### 3.2 Announcement Message

An announcement asserts that the announcing node currently holds both blobs and
can serve them. It is a UTF-8 JSON object.

| Field | Type | Description |
|---|---|---|
| `igc_hash` | string | BLAKE3 hash of the IGC blob |
| `meta_hash` | string | BLAKE3 hash of the metadata blob |
| `node_id` | string | serving node's Ed25519 public key |
| `igc_ticket` | string | Iroh `BlobTicket` for the IGC blob |
| `meta_ticket` | string | Iroh `BlobTicket` for the metadata blob |

Consistency constraints:

- the hash embedded inside `igc_ticket` MUST equal `igc_hash`
- the hash embedded inside `meta_ticket` MUST equal `meta_hash`
- the node identity embedded in both tickets MUST equal `node_id`

Receivers that detect any mismatch MUST discard the announcement without
protocol response.

Encoding rules:

- JSON MUST be valid UTF-8
- the message MUST NOT exceed 1024 bytes
- unknown fields MUST be ignored
- announcements missing any required field MUST be discarded without protocol
  response

Example:

```json
{
  "igc_hash": "3a7f2c8e1b4d5f6a9b0c1d2e3f4a5b6c7d8e9f0a1b2c3d4e5f6a7b8c9d0e1f2a",
  "meta_hash": "1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1b2c",
  "node_id": "a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1b2",
  "igc_ticket": "blobafyreiczsq3p4mbgrs7m2phruxqxgzebdmtqtbmm7q4wlfnqbjnfpziq",
  "meta_ticket": "blobafyreick7w5alf3nsp2n5ggfq3n7jkl2phruxqxgzebdmtqtbmm7q4wl"
}
```

### 3.3 Topic Versioning

- `igc-net/announce/v1` encodes the protocol major version
- breaking wire-format changes MUST use a new derivation string
- nodes MAY bridge multiple versions during migrations

### 3.4 Re-announcement

Any node that has fetched and hash-verified both blobs MAY re-announce the
flight using its own tickets and `node_id`.

Rules:

- the re-announcing node MUST already hold and verify both blobs
- `igc_hash` and `meta_hash` remain unchanged
- `node_id`, `igc_ticket`, and `meta_ticket` MUST belong to the re-announcing
  node

Receiver behavior:

- deduplication key is `(meta_hash, node_id)`
- the same `meta_hash` from a different `node_id` is a distinct serving peer
- if `meta_hash` is already known, the receiver SHOULD retain the additional
  fetch source without re-fetching metadata

## 4. Node Identity and Key Management

### 4.1 Key Pair

- node identity is an Ed25519 key pair
- the secret key MUST persist across restarts
- the public key is the stable network identity

### 4.2 Storage

Implementations MAY store the secret key in:

- a file on disk
- an OS keychain or secure enclave
- an HSM or equivalent protected store

The storage mechanism is implementation-defined.

### 4.3 `publisher_node_id`

When present in the metadata blob, `publisher_node_id` records the original
publishing node's Ed25519 public key as an attribution field. It is not a fetch
address and MUST NOT be treated as one.

## 5. Discovery and Fetch Flow

On receiving an announcement, a node MUST process it in this order:

```text
1. Parse JSON
2. Discard malformed payloads or payloads missing required fields without
   protocol response
3. Validate ticket/hash/node consistency
4. Deduplicate using (meta_hash, node_id)
5. If meta_hash known but node_id new, record new fetch source and stop
6. Fetch metadata via meta_ticket
7. Verify BLAKE3(metadata_bytes) == meta_hash
8. Validate metadata blob:
   - valid UTF-8 JSON
   - schema == "igc-net/metadata"
   - supported schema_version
   - metadata igc_hash == announcement igc_hash
9. Store metadata locally
10. Apply local raw-blob fetch policy
11. If fetching raw IGC:
    - fetch via igc_ticket
    - verify BLAKE3(igc_bytes) == igc_hash
    - store only if verified
```

Metadata and raw-blob fetches MUST NOT block the gossip receive loop.

## 6. Content Addressing Rules

- `igc_hash` MUST equal `BLAKE3(raw_igc_bytes)`
- `meta_hash` MUST be checked against `BLAKE3(metadata_blob_bytes)` after
  fetch
- any blob that fails hash verification MUST be discarded

## 7. References

- `specs_meta.md`
- `requirements.md`
- [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119)
- [BLAKE3 specification](https://github.com/BLAKE3-team/BLAKE3-specs)
- [Iroh documentation](https://docs.iroh.computer)
