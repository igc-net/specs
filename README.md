# `igc-net` Protocol Specification

Open protocol for publishing, discovering, and governing IGC flight logs over a
peer-to-peer network.

For normative protocol scope, terminology, invariants, document precedence, and
the reading order, start with [00-overview.md](./00-overview.md).

## Purpose and Motivation

`igc-net` defines a neutral, content-addressed exchange layer for IGC flight
artifacts, ownership governance, and controlled access. It is protocol-first
and implementation-language independent.

The protocol is designed so that:

- identical raw IGC bytes produce identical content identities across all
  publishers
- publication, serving, and indexing are not tied to a central registry or a
  single portal
- governance over ownership and access is explicit rather than implicit in one
  application's database state
- metadata, analytics, and operator tooling can evolve without redefining the
  base identity and transport model
- independent implementations in Rust, Python, Go, or other languages can
  interoperate from the specification alone

The specification formalises a two-plane architecture:

- a decentralised data plane for artifact announcement, discovery, and fetch
- a permissioned governance plane for ownership, policy, disputes, and sync

That separation is a protocol constraint, not a documentation convenience.

## Status Snapshot

- Latest SPECS tagged release: `v0.1.0`
- Latest REFERENCE IMPLEMENTATION tagged release: `v0.1.0`
- Current specification work: `v0.3` draft
- Public `v0.3` release/tag: not yet published

## How to Read This Repository

### Normative specification

Read these in dependency order:

1. [00-overview.md](./00-overview.md)
2. [10-core.md](./10-core.md)
3. [20-artifacts.md](./20-artifacts.md)
4. [30-transport.md](./30-transport.md)
5. [40-pilot-and-metadata.md](./40-pilot-and-metadata.md)
6. [50-governance.md](./50-governance.md)
7. [55-governance-sync.md](./55-governance-sync.md)
8. [60-keys-and-access.md](./60-keys-and-access.md)
9. [65-pilot-auth-did.md](./65-pilot-auth-did.md)
10. [70-durability.md](./70-durability.md)
11. [80-analytics.md](./80-analytics.md)
12. [90-conformance.md](./90-conformance.md)
13. [92-threat-model.md](./92-threat-model.md)

### Informative guides and contributor docs

- [getting-started.md](./getting-started.md) — implementer onboarding
- [PILOT-GUIDE.md](./PILOT-GUIDE.md) — pilot-facing operational guide
- [NODE-OPERATOR-GUIDE.md](./NODE-OPERATOR-GUIDE.md) — node operator guide
- [PORTAL-OPERATOR-GUIDE.md](./PORTAL-OPERATOR-GUIDE.md) — portal operator
  guide
- [CONTRIBUTING.md](./CONTRIBUTING.md) — change and review rules

If this README conflicts with a numbered normative document, the numbered
document wins.

## Interoperability Notes

The protocol's normative identifiers, hash rules, canonicalization rules, and
key-derivation inputs are defined in the numbered specification documents.

- For cryptographic primitives, identity anchors, and record IDs, read
  [10-core.md](./10-core.md).
- For interoperability-critical derivation strings and a minimal implementation
  path, read [getting-started.md](./getting-started.md).

## Repository Boundary

This repository defines protocol semantics and related implementation guidance.
It does not define language-specific APIs.

Implementation repositories such as `igc-net-rs` and `igc-net-py` are
informative with respect to the protocol. They may demonstrate feasible
behavior, but they do not override the specification.

## Planned Work

### v0.3

- normative transition from native `pilot-profile` to `PilotProfileCredential`
- `pilot_auth_did` as canonical wallet-held `did:key`
- optional `did:web` for portal-issued/public-facing issuer identity and
  non-authoritative public aliases
- threat model, abuse analysis, extension model, interop fixtures, interop test suite

### v0.4

- add sync-independent performance-oriented and bulk-operation primitives
- governance hardening, resolver/federation evolution, privacy leakage mitigation
- add sync-independent observability, health, and auditability primitives

### v0.5

- sync redesign and convergence semantics
- bulk/state-transfer and replication primitives

### v1.0

v1.0 is to be treated as production release of the standard.
All the lower version designations are to be considered experimental, and no backwards compatibility should be expected.

## Change Process

Protocol changes should be reviewed via issues and pull requests. See
[CONTRIBUTING.md](./CONTRIBUTING.md).
