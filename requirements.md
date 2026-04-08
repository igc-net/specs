# `igc-net` Protocol Requirements

**Status:** Working draft  
**Version:** 0.1  
**Date:** 2026-04-03

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be
interpreted as described in [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119).

## 1. Purpose and Scope

`igc-net` is a protocol for publishing, discovering, and exchanging IGC flight
log files over a peer-to-peer network built on the Iroh stack. It does not
replace soaring platforms or define application-specific business logic. It
defines interoperable content identifiers, announcement semantics, metadata
rules, and fetch behavior.

Publishing to `igc-net` is a public act. Any node that joins the network can
observe announcements and attempt to fetch announced content.

Normative details are split across:

- `specs_igc.md` — blob model, announcement wire format, topic IDs, fetch flow
- `specs_meta.md` — base metadata schema and analytics extension

## 2. Actor Roles

| Actor | Description |
|---|---|
| Publisher node | Accepts raw `.igc` bytes, stores them locally, constructs metadata, and announces availability |
| Indexer node | Subscribes to announcements, validates metadata, and maintains a local index |
| Archival node | An indexer that also fetches and stores raw IGC blobs |
| Application node | An embedded implementation that may publish, index, or both |
| Analytics endpoint | A node that implements Part II of `specs_meta.md` |

A single implementation MAY combine roles.

## 3. Publisher Requirements

- R-PUB-01: A publisher MUST accept raw `.igc` bytes as input without
  transforming them before hashing.
- R-PUB-02: A publisher MUST compute `igc_hash` as BLAKE3 over the exact raw
  IGC bytes.
- R-PUB-03: A publisher MUST construct a metadata blob conforming to
  `specs_meta.md` Part I and include the matching `igc_hash`.
- R-PUB-04: A publisher MUST store both the IGC blob and metadata blob locally
  before announcing.
- R-PUB-05: A publisher MUST NOT announce a flight before both blobs are
  locally retrievable.
- R-PUB-06: A publisher SHOULD populate all optional metadata fields derivable
  from the IGC source file.
- R-PUB-07: A publisher MUST NOT fabricate or infer metadata values not
  derivable from the IGC source.
- R-PUB-08: A publisher MUST omit unavailable optional fields rather than
  serializing them as `null`.
- R-PUB-09: A publisher MUST broadcast an announcement to the well-known topic
  defined in `specs_igc.md`.
- R-PUB-10: The announcement MUST contain `igc_hash`, `meta_hash`, `node_id`,
  `igc_ticket`, and `meta_ticket`.
- R-PUB-11: The embedded hashes and node identity in the tickets MUST match the
  top-level fields in the announcement.
- R-PUB-12: A publisher MAY republish the same IGC bytes with a new metadata
  blob if corrected metadata is needed.

## 4. Indexer Requirements

- R-INDEX-01: An indexer MUST subscribe to the canonical announce topic on
  startup.
- R-INDEX-02: It MUST reconnect automatically after transient network failure.
- R-INDEX-03: Deduplication MUST be keyed by `(meta_hash, node_id)`.
- R-INDEX-04: If `meta_hash` is known but `node_id` is new, the indexer MUST
  record the new serving peer and MUST NOT re-fetch the metadata blob.
- R-INDEX-05: Metadata fetch and raw blob fetch MUST NOT block the gossip event
  loop.
- R-INDEX-06: An indexer MUST validate fetched metadata as UTF-8 JSON.
- R-INDEX-07: The metadata `schema` field MUST equal `igc-net/metadata`.
- R-INDEX-08: Unsupported `schema_version` values MUST be skipped silently.
- R-INDEX-09: The metadata `igc_hash` MUST match the announcement `igc_hash`.
- R-INDEX-10: An indexer MUST store validated metadata in a local index.
- R-INDEX-11: An indexer MUST define a raw-blob fetch policy. Recommended
  policies are `metadata-only`, `eager`, and `geo-filtered`.
- R-INDEX-12: Failed fetches or malformed announcements MUST NOT crash the
  node.
- R-INDEX-13: Unknown additional announcement fields MUST be ignored.
- R-INDEX-14: A node that has fetched and verified both blobs MAY re-announce
  them using its own tickets and node identity.

## 5. Raw Blob Fetching Requirements

- R-FETCH-01: Any node MAY fetch a raw IGC blob via an `igc_ticket`.
- R-FETCH-02: A node that announces a blob MUST be able to serve it while the
  ticket remains valid.
- R-FETCH-03: A node with multiple serving peers for the same `igc_hash` MAY
  retry alternative peers on failure.
- R-FETCH-04: Blobs that fail BLAKE3 verification MUST be discarded.

## 6. Analytics Extension Requirements

These apply only to implementations of Part II in `specs_meta.md`.

- R-ANA-01: Part II is OPTIONAL. A node implementing only the base protocol is
  still compliant.
- R-ANA-02: An analytics endpoint that implements Part II MUST join the
  analytics topic defined in `specs_meta.md`.
- R-ANA-03: Analytics writes MUST be attributable to the writing endpoint's
  author key.
- R-ANA-04: Endpoints MUST silently skip unknown analytics types or unsupported
  analytics schema versions.
- R-ANA-05: Custom analytics types MUST use reverse-DNS namespacing.
- R-ANA-06: Trust decisions over analytics providers are application-layer
  policy, not protocol-layer policy.

## 7. Non-Functional Requirements

- R-NFR-01: Node state required for protocol continuity MUST survive restarts.
- R-NFR-02: Node identity MUST persist across restarts; implementations MUST
  NOT generate a new Ed25519 identity on every startup.
- R-NFR-03: All BLAKE3 hashes in wire formats MUST be encoded as 64-character
  lowercase hex strings.
- R-NFR-04: All canonical timestamps in metadata MUST use
  `YYYY-MM-DDTHH:MM:SSZ`.
- R-NFR-05: Implementations MUST use the exact canonical topic derivation
  strings and schema identifiers published by the specification.
- R-NFR-06: Nodes SHOULD continue operating without all bootstrap peers being
  reachable at all times.

## 8. Versioning Requirements

- R-VER-01: Breaking changes to the base wire protocol MUST produce a new
  announce topic derivation string.
- R-VER-02: Breaking changes to the analytics protocol MUST produce a new
  analytics topic derivation string.
- R-VER-03: Breaking changes to base metadata semantics MUST increment
  `schema_version`.
- R-VER-04: Additive optional metadata fields MAY be introduced without
  incrementing `schema_version` if they do not break existing parsers.
- R-VER-05: Unsupported schema versions MUST be skipped, not treated as
  transport or peer failures.
