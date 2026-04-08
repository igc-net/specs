# Getting Started with `igc-net`

This document is for implementers of the protocol, not users of a specific
language binding.

## Read Order

1. `specs_igc.md`
2. `specs_meta.md`
3. `requirements.md`
4. `metadata-schema.md`

If those documents conflict, the normative specification documents
(`specs_igc.md`, `specs_meta.md`, `requirements.md`) take precedence.

## Minimal Compliant Node

A minimal `igc-net` implementation needs to do four things:

1. accept raw `.igc` bytes
2. compute BLAKE3 over the exact bytes to derive `igc_hash`
3. publish or validate a metadata blob using the canonical metadata schema
4. announce blob availability on the well-known topic using the canonical wire
   format

That is sufficient for a base-protocol implementation. Analytics are optional.

## Canonical v1 Identifiers

- metadata schema: `igc-net/metadata`
- announce topic derivation string: `igc-net/announce/v1`
- analytics topic derivation string: `igc-net/analytics/v1`
- analytics namespace derivation prefix: `igc-net/meta-doc/v1`

These are interoperability-critical. Do not substitute local names.

## Minimal Publisher Flow

1. Receive raw `.igc` bytes
2. Compute `igc_hash = BLAKE3(igc_bytes)`
3. Construct metadata conforming to `specs_meta.md` Part I
4. Serialize metadata as UTF-8 JSON
5. Compute `meta_hash = BLAKE3(metadata_bytes)`
6. Store both blobs locally
7. Produce fetch tickets for both blobs
8. Broadcast the announcement JSON on the announce topic

## Minimal Indexer Flow

1. Join the announce topic on startup
2. Parse incoming announcement JSON
3. Validate required fields and ticket consistency
4. Deduplicate using `(meta_hash, node_id)`
5. Fetch metadata via `meta_ticket`
6. Validate the metadata blob
7. Store metadata locally
8. Apply a local raw-blob fetch policy

## Metadata Expectations

The base metadata blob is intentionally small and conservative:

- it is derived only from the IGC file and local publish context
- optional fields are omitted, not set to `null`
- hashes and node IDs are lowercase hex
- timestamps are canonical UTC strings

Use `metadata-schema.md` for the compact field reference and
`specs_meta.md` for the normative rules.

## Analytics Extension

Part II in `specs_meta.md` specifies an optional analytics exchange model using
an `IGC_META_DOC` document per flight. Implementers of the base protocol do not
need Part II for compliance.

## Interoperability Checks

Before calling an implementation compliant, verify at minimum:

- topic IDs match the canonical derivation strings
- metadata blobs use `igc-net/metadata`
- hash verification rejects corrupted blobs
- duplicate announcements from the same `(meta_hash, node_id)` are ignored
- announcements from different `node_id` values for the same `meta_hash` are
  treated as distinct serving peers
