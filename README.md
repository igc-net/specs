# `igc-net` Protocol Specification

Open protocol for publishing, discovering, and exchanging IGC flight logs over
a peer-to-peer network.

## Purpose

`igc-net` defines a neutral, content-addressed exchange layer for raw IGC
flight logs and associated metadata. It is protocol-first and
implementation-language independent.

The protocol is designed so that:

- identical IGC bytes produce identical BLAKE3 hashes across all publishers
- any node can publish or index flights without central registration
- metadata and derived analytics can evolve independently of any single
  application
- independent implementations in Rust, Python, Go, or other languages can
  interoperate from the specification alone

## Document Set

Read in this order:

1. `specs_igc.md` — raw IGC blob model, announcement wire format, topic IDs,
   node identity, fetch flow, and verification rules
2. `specs_meta.md` — base metadata schema and the optional analytics extension
3. `requirements.md` — functional and non-functional requirements and
   protocol-level constraints
4. `metadata-schema.md` — compact field-by-field reference for the base
   metadata schema
5. `getting-started.md` — implementation-oriented onboarding for new protocol
   implementers

## Status

- Base protocol and base metadata schema: specified and implemented in the Rust
  reference implementation
- Analytics extension (`IGC_META_DOC` / Part II): specified, but not yet
  implemented in the Rust reference implementation
- Public protocol release: not yet tagged

## Repository Boundary

This repository contains protocol and schema documentation only. It does not
define language-specific APIs.

The Rust reference implementation lives in the separate `igc-net-rs`
repository. That implementation is informative for interoperability and current
state, but this specification repository is the canonical source of truth for
protocol semantics.

## Core Identifiers

The current canonical v1 identifiers are:

- metadata schema: `igc-net/metadata`
- announce topic derivation string: `igc-net/announce/v1`
- analytics topic derivation string: `igc-net/analytics/v1`
- analytics namespace derivation prefix: `igc-net/meta-doc/v1`
- standard analytics type namespace: `igc-net/*`

These identifiers are normative. Implementations that use different strings do
not interoperate.

## Governance

Initial maintenance is by NTNU DSE Lab. Protocol changes should be reviewed via
GitHub issues and pull requests. See `CONTRIBUTING.md`.
