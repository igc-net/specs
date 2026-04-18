# igc-net v0.2 — Overview

**Status:** Normative  
**Depends on:** nothing — this is the entry point

---

## 1. Scope

igc-net v0.2 is a federated protocol for flight identity, artifact publication,
ownership governance, and private access control for IGC flight recorder files.

v0.2 defines:

- flight file identity
- publication modes and artifact access state
- pilot identity and key model
- ownership claims and dispute resolution
- permissioned federation governance
- private access proof and key exchange
- protected and private metadata handling
- transport-layer confidentiality dependency on iroh
- publication mode changes and access grant revocation
- erasure and deletion semantics
- governance topic and sync semantics

v0.2 uses a **decentralised data plane** for artifact publication and transport,
and a **permissioned governance plane** for ownership, access policy, and dispute
resolution. These two planes are defined separately and must not be collapsed.

---

## 2. Out of scope

The following are explicitly out of scope for v0.2:

- tandem flights (single-pilot ownership only)
- club or shared ownership models
- instructor/student ownership distinctions
- multi-owner policy
- permissionless resolver governance

---

## 3. Two-plane model

igc-net separates two distinct protocol planes.

### Data plane

Responsible for:

- artifact publication and announcement
- flight existence records and locator (ticket) distribution
- artifact fetch and serving
- deduplication across portals
- private-flight existence and locator discovery

The data plane is open: any node may receive and re-announce artifact
availability. Stale serving-peer data is tolerated.

### Governance plane

Responsible for:

- ownership claims, approvals, challenges, and resolutions
- publication mode policy and mode-change records
- deletion requests
- governance record propagation and sync

The governance plane is permissioned: stale governance state is not tolerated
because it can cause wrongful private content release or stale ownership decisions.

These planes use separate transport topics. A record on the data-plane announce
topic does not carry governance meaning, and a record on the governance topic does
not carry artifact-locator data.

---

## 4. Actor roles

### Identity roles

These are cryptographic identities defined in the protocol. Each is a distinct
Ed25519 keypair.

| Role | Identifier | Purpose |
|------|-----------|---------|
| Pilot | `pilot_id` | Authors flights, signs claims and metadata |
| Serving node | `node_id` | Signs announcements and fetch tickets |
| Trusted resolver | `resolver_id` | Signs approvals, challenges, and resolutions |

A single machine may operate multiple roles simultaneously. Each role MUST use
a distinct keypair. No two roles may share a keypair.

### Operational roles

The following are deployment and operational roles. They are not governance
claim types and do not have dedicated protocol record kinds.

| Operational role | Description |
|-----------------|-------------|
| Uploader | Portal or agent that submits a flight to the network at upload time |
| Custodian | Portal that stores and serves a flight on behalf of a pilot |
| Archive node | Node that retains private content for durability purposes; always a private-access node |

### Node access categories

Per pilot, every logged-in node operates in exactly one of two access
categories. These are properties of how the pilot has configured the node,
not governance claim types.

| Category | Description |
|----------|-------------|
| Identity-linked node (Category 1) | Knows the pilot's `pilot_id`. May discover, fetch, serve, and index the pilot's `public` and `protected` artifacts. Does NOT hold the pilot's `private_access_keypair`. MUST NOT access private content or personal-identity fields. |
| Private-access node (Category 2) | All capabilities of Category 1 plus holds the pilot's `private_access_keypair`. May sign fetch requests for the pilot's private content and private metadata records. |

Full definitions are in `60-keys-and-access.md §2`.

`owner` is the only governance claim type in v0.2.

---

## 5. Core object classes

Three object classes structure the protocol. These classes are used consistently
across all normative documents.

### Identity anchor

`raw_igc_hash` — computed as `BLAKE3(raw_igc_bytes)`.

The identity anchor uniquely identifies a flight across the entire network. It is
immutable once computed. All subsequent records (claims, artifacts, metadata,
governance records) reference this anchor. Two portals independently receiving
the same IGC file bytes will compute the same `raw_igc_hash`.

### Artifact identity

`protected_hash` identifies the sanitized public artifact derived from a
`protected` flight.

- `protected_hash = BLAKE3(sanitized_igc_bytes)`

Artifact identity changes if the artifact changes. It is distinct from the
identity anchor. `public` and `private` flights have no separate artifact
identity; the raw IGC bytes are themselves the artifact and their identity
is `raw_igc_hash`.

### Locator

A ticket is a locator: it encodes a serving node endpoint and a blob identifier.
A ticket tells a client *where* to fetch an artifact.

**Ticket possession is not authorization.** Whether a requester may receive the
artifact is decided by the serving node at delivery time, based on governance
state and key-possession proof.

---

## 6. Terminology

| Term | Definition |
|------|-----------|
| `raw_igc_hash` | `BLAKE3(raw_igc_bytes)` — universal flight identity anchor |
| `protected_hash` | `BLAKE3(sanitized_igc_bytes)` — sanitized artifact identity |
| `pilot_id` | `"igcnet:id:" + hex(root_public_key_bytes)` — pilot identity string |
| `node_id` | Serving node Ed25519 keypair identity |
| `resolver_id` | Trusted resolver Ed25519 keypair identity |
| `private_access_keypair` | Pilot's Ed25519 keypair used to authorize fetch requests for non-public content (private raw IGC, protected raw companion, private metadata records, igc-metadata). Single credential covering all non-public content. |
| `private_access_public_key` | Public half of `private_access_keypair`, published via `private-access-rotation-record` on the governance topic |
| identity-linked node (Category 1) | Node that knows a pilot's `pilot_id` but does NOT hold the pilot's `private_access_keypair`; may serve `public` and `protected` artifacts only |
| private-access node (Category 2) | Node that holds a pilot's `private_access_keypair`; may sign fetch requests for the pilot's non-public content |
| `publication_mode` | Artifact access state: `public`, `protected`, or `private` |
| `visibility` | Metadata record privacy state: controls who may read a metadata record |
| ticket | Locator encoding a serving node endpoint and blob identifier |
| claim | Governance record asserting pilot ownership of a `raw_igc_hash` |
| approval | Resolver record accepting a claim |
| challenge | Resolver record contesting an accepted claim |
| resolution | Record ending a challenge (accept or reject) |
| accepted owner | A pilot whose claim has been approved by a trusted resolver |
| governance topic | The well-known publish/subscribe topic for governance records |
| announce topic | The well-known publish/subscribe topic for data-plane announcements |
| canonical JSON | RFC 8785 serialisation used for signing and record-ID computation |
| `record_id` | `BLAKE3(canonical_json(record_without_signature))` |

`publication_mode` and `visibility` must not be used interchangeably.
`publication_mode` is a property of an artifact's access state.
`visibility` is a property of a metadata record's privacy state.

---

## 7. Document set

The v0.2 normative specification consists of eleven documents.

| # | File | Purpose | Depends on |
|---|------|---------|-----------|
| 00 | `00-overview.md` | Scope, terminology, two-plane model, document map | — |
| 10 | `10-core.md` | Cryptographic primitives, identity anchors, canonical form | 00 |
| 20 | `20-artifacts.md` | Publication modes, sanitization algorithm, artifact relations | 10 |
| 30 | `30-transport.md` | Announce topics, wire format, deduplication, fetch rules | 10, 20 |
| 40 | `40-pilot-and-metadata.md` | Pilot profile, flight metadata, igc metadata, record shapes | 10 |
| 50 | `50-governance.md` | Claim, approval, challenge, resolution, mode-change, deletion | 20, 40 |
| 55 | `55-governance-sync.md` | Governance topic, propagation, ordering, catch-up | 50 |
| 60 | `60-keys-and-access.md` | Node access categories, `private_access_keypair`, fetch authorization, rotation | 50 |
| 70 | `70-durability.md` | Durability obligations, archive custody, deletion enforcement | 60 |
| 80 | `80-analytics.md` | Optional analytics exchange model | 10, 20, 30 |
| 90 | `90-conformance.md` | Requirement IDs, conformance profiles, cross-document invariants | all |

---

## 8. Reading order and dependency graph

```
00-overview
     |
 10-core
     |
 20-artifacts
    /        \
30-transport  40-pilot-and-metadata
                        |
                  50-governance
                        |
               55-governance-sync
                        |
               60-keys-and-access
                        |
                  70-durability

80-analytics  ← depends on 10-core, 20-artifacts, 30-transport
90-conformance ← depends on all normative documents
```

Reading order for implementers: 00 → 10 → 20 → 30 → 40 → 50 → 55 → 60 → 70.
Read 80 only if implementing the analytics extension.
Read 90 for conformance mapping.

---

## 9. Document precedence

The eleven normative documents listed in §7 are authoritative.

`README.md` is informative. It exists as the repository landing page, index,
status page, and migration note for the specification work. It does not add
requirements.

`PILOT-GUIDE.md`, `NODE-OPERATOR-GUIDE.md`, and
`PORTAL-OPERATOR-GUIDE.md` are informative living documents. They provide
workflow-oriented and operational explanations and do not add requirements.
When `README.md` or an informative guide conflicts with a normative document,
the normative document takes precedence.

Implementation guides are informative. They describe how to build compliant
implementations and do not add requirements.

`90-conformance.md` summarises requirements from the normative documents using
`R-*` identifiers. These identifiers are reference labels only. The normative
text in the numbered documents remains the source of truth.

---

## 10. Cross-document invariants

The following invariants constrain all normative documents. No document in the
v0.2 set may contradict them.

1. `raw_igc_hash` is immutable once computed from a given set of IGC file
   bytes.
2. BLAKE3 is the exclusive hash function for all content hashes and record
   IDs.
3. Ed25519 is the exclusive signing algorithm. All identity signatures are
   made directly from the role's identity key. There are no delegated or
   rotating identity signing keys in v0.2. The pilot's
   `private_access_keypair` is an authorization credential (not an identity
   signing key) and MAY be rotated by the pilot through the
   `private-access-rotation-record` defined in `60-keys-and-access.md §6`.
4. No protocol-level authenticated encryption is applied to content. Content
   confidentiality in flight is provided by the underlying iroh peer-to-peer
   transport; confidentiality at rest for non-public content held at a
   private-access node is a compliance and legal obligation on the node
   operator.
5. RFC 8785 canonical JSON is used for all signed records and all record-ID
   computations.
6. Governance state takes precedence over key possession for access
   decisions. A serving node MUST refuse to deliver content for a flight in
   `contested` or `rejected` governance state regardless of whether the
   requester can sign a valid fetch request.
7. Non-public content is served as plaintext bytes over iroh's end-to-end
   encrypted transport, gated by a fetch request signed by the pilot's
   currently authorized `private_access_keypair`. No AEAD envelope is
   applied at the protocol layer; confidentiality at rest on serving and
   durability nodes is a compliance and legal obligation.
8. `publication_mode` governs artifact access state. `visibility` governs
   metadata record privacy state. These terms must not be interchanged.
9. `created_at` is informational and serves as an expiry baseline for
   challenge records only. It is not a general record-ordering authority
   and must not be used for conflict resolution or supersession decisions.
10. Identity signing keys (`pilot_id`, `node_id`, `resolver_id`) are not
    delegated and do not rotate in v0.2. The pilot's
    `private_access_keypair` is separate from `pilot_id` and MAY be rotated
    through the rotation record mechanism.
