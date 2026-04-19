# igc-net — Conformance

**Status:** Normative  
**Depends on:** all normative documents (00–92)

---

## 1. Purpose of this document

This document provides:

- requirement label namespaces (`R-*`) for tracing test cases to normative
  text
- conformance profiles for common implementation roles
- a cross-document invariants section collecting protocol-wide constraints
- a summary of operator-facing obligations

**Critical rule**: `R-*` labels are traceability aids only. They do NOT
replace the normative text in the main specification documents. The
numbered normative documents remain the authoritative source of normative
requirements.

---

## 2. Requirement namespaces

| Prefix | Covers | Normative source |
|--------|--------|-----------------|
| `R-CORE-xx` | Core identifiers and cryptographic primitives | `10-core.md` |
| `R-ART-xx` | Artifacts and publication modes | `20-artifacts.md` |
| `R-TRANS-xx` | Data-plane transport | `30-transport.md` |
| `R-META-xx` | Pilot and metadata records | `40-pilot-and-metadata.md` |
| `R-GOV-xx` | Governance semantics | `50-governance.md` |
| `R-GSYNC-xx` | Governance sync and propagation | `55-governance-sync.md` |
| `R-ACCESS-xx` | Keys, node categories, and access | `60-keys-and-access.md` |
| `R-AUTH-xx` | Pilot authentication DID binding and rotation | `65-pilot-auth-did.md` |
| `R-DUR-xx` | Durability and erasure | `70-durability.md` |
| `R-ANA-xx` | Analytics (optional) | `80-analytics.md` |
| `R-THREAT-xx` | Threat-driven security obligations | `92-threat-model.md` |

Every `R-*` label MUST trace to at least one normative MUST or MUST NOT
statement in the referenced source document. Labels are embedded inline in
the normative text at the specific statement they identify (see each spec
file).

---

## 3. Conformance profiles

These profiles define what a compliant implementation of each role MUST and
SHOULD implement.

Archive is an operational role, not a conformance profile. An archive node
is a Private-Access Node that has been granted the pilot's
`private_access_keypair`. It is covered by the Private-Access Node profile
plus the compliance obligations in `70-durability.md`.

### 3.1 Minimal publisher

A minimal publisher accepts raw IGC files, computes identity, and
publishes artifacts.

**MUST implement:**

- `R-CORE-*` — all cryptographic primitives, hash computation, identity
- `R-ART-*` — publication mode selection, sanitization algorithm, mode
  record
- `R-TRANS-*` — announce topic broadcast

**MAY omit:** governance, metadata, durability, analytics.

### 3.2 Serving node

A serving node stores artifacts and responds to fetch requests.

**MUST implement:**

- `R-CORE-*`
- `R-ART-*`
- `R-TRANS-*` — announce and deduplication
- `R-ACCESS-*` — fetch-request verification, governance state check
  before serving
- `R-GSYNC-*` — governance topic subscription; governance state check;
  processing of `private-access-rotation-record` and
  `pilot-auth-did-record`

**SHOULD implement:**

- `R-DUR-*` — mode upgrade and deletion obligations

### 3.3 Identity-linked node (Category 1)

An identity-linked node binds a local user account to a `pilot_id` and
serves that pilot's `public` and `protected` artifacts. It does NOT hold
the pilot's `private_access_keypair`.

**MUST implement:**

- Minimal publisher profile.
- Serving node profile for `public` artifacts and for the sanitized
  component of `protected` artifacts.
- The Category 1 obligations in `60-keys-and-access.md §2.1`, including
  refusing to accept the pilot's `private_access_keypair` during login
  and refraining from scraping personal-identity fields from public IGC
  headers.

**MUST NOT:**

- Fetch `private` raw IGC bytes, protected raw companions, private
  metadata records, or `igc-metadata` belonging to the logged-in pilot.

**MAY omit:** analytics.

### 3.4 Private-access node (Category 2)

A private-access node holds the pilot's `private_access_keypair` and can
sign fetch requests for the pilot's non-public content.

**MUST implement:**

- Identity-linked node profile.
- `R-ACCESS-*` — including holding, using, and rotating
  `private_access_keypair`; processing `private-access-rotation-record`;
  confidentiality-at-rest compliance obligations.
- `R-META-*` — native metadata records (`flight-metadata`,
  `igc-metadata`) and protected-flight identity display rules.
- `R-GOV-*` — claim submission, publication mode records, deletion
  requests.
- `R-DUR-*`.

**SHOULD implement:**

- Portal-to-portal key-exchange grant flow (`60-keys-and-access.md §5`).

### 3.5 Relying-party portal / authentication endpoint

A relying-party portal or authentication endpoint verifies
`PilotProfileCredential`, processes `pilot-auth-did-record` state for pilot
authentication, and may cache latest verified profile VCs.

**MUST implement:**

- `R-AUTH-*`
- `R-META-*` rules related to `PilotProfileCredential` verification,
  staleness, and identity display
- `R-THREAT-*` rules related to VC validation and authority boundaries

**MUST NOT:**

- Treat a pilot-facing `did:web` alias or mirror as overriding the
  authoritative `pilot-auth-did-record` chain.

### 3.6 Trusted resolver

A trusted resolver processes governance records and issues approvals,
challenges, and resolutions.

**MUST implement:**

- `R-CORE-*`
- `R-GOV-*` — approval, challenge, resolution, roster update shapes
- `R-GSYNC-*` — governance topic propagation, including
  `private-access-rotation-record` relay

---

## 4. Cross-document invariants

The following invariants constrain all normative documents. No document may
contradict them. No conformant implementation may violate them.
`00-overview.md §10` summarises these invariants for orientation; this section
is the canonical authority.

1. **`raw_igc_hash` is immutable.** Once computed from a given IGC file's
   raw bytes, `raw_igc_hash` MUST NOT change. All records that reference
   this hash continue to reference the same value permanently.

2. **BLAKE3 is the exclusive native-record hash function.** No other
   hash function is permitted for any native content hash or native
   `record_id`.

3. **Ed25519 is the exclusive native-record signing algorithm.** No
   other signing algorithm is permitted for native `igc-net` signed
   records. `pilot_id`, `node_id`, `resolver_id`,
   `private_access_keypair`, and `pilot_auth_did` are all Ed25519
   keypairs.

4. **No protocol-level content encryption.** igc-net does not apply any AEAD
   envelope to flight artifacts, metadata records, or fetch-request
   bodies. Content confidentiality in flight is provided by the
   underlying iroh peer-to-peer transport; confidentiality at rest for
   non-public content is a compliance and legal obligation on the
   private-access node that holds it.

5. **RFC 8785 canonical JSON for all signed records.** All records that
   are signed or hashed for a record ID MUST be serialised using
   RFC 8785.

6. **Governance state takes precedence over key possession.** A serving
   node MUST refuse to deliver any content for a `raw_igc_hash` in
   `contested` or `rejected` governance state, regardless of whether the
   requester can sign a valid fetch request.

7. **Non-public native content requires a signed fetch request.** Private
   raw IGC bytes, protected raw companions, and native
   `visibility: "private"` metadata records (including all
   `igc-metadata`) MUST be served only to requesters who present a
   fetch request signed by the pilot's currently authorized
   `private_access_keypair`. `PilotProfileCredential` is not a
   fetch-request-governed native object. The bytes transmitted are
   plaintext; the iroh transport provides in-flight confidentiality.

8. **`publication_mode` and `visibility` are distinct.**
   `publication_mode` governs artifact access state (`public`,
   `protected`, `private`). `visibility` governs metadata record privacy
   state. These MUST NOT be interchanged.

9. **`created_at` is informational and an expiry baseline only.** The
   `created_at` field in governance records is informational. It is used
   only as an expiry baseline for challenge records. It is NOT the
   authority for record ordering, conflict resolution, or supersession
   decisions.

10. **Governance identity signing is directly from the role's identity
    key.** All `pilot_id`, `node_id`, and `resolver_id` native-record
    signatures are made directly from the signer's current identity
    key. Governance identity signing keys do not delegate and do not
    rotate. The pilot's `private_access_keypair` is a separate
    authorization credential and MAY be rotated through the
    `private-access-rotation-record` mechanism. The pilot's
    `pilot_auth_did` is a separate authentication / VC-issuer
    credential and MUST NOT replace `pilot_id` as the authoritative
    protocol root identity.

---

## 5. Operator-facing obligations summary

This table summarises key obligations for portal operators. Normative
source references are given for each obligation.

| Trigger | Obligation | Timeframe | Source |
|---------|-----------|-----------|--------|
| Valid deletion request received | Stop serving all artifacts for `raw_igc_hash` | Immediate | `50-governance.md §13`, `70-durability.md §2.1` |
| Valid deletion request received | Remove flight-scoped records and indexes | Within 30 days | `70-durability.md §2.1` |
| Mode upgrade record received | Stop serving previously permitted artifact | Immediate | `20-artifacts.md §7.1`, `70-durability.md §2.2` |
| Pilot instructs `private_access_keypair` deletion | Delete key; stop serving non-public content | Immediate | `60-keys-and-access.md §7`, `70-durability.md §2.3` |
| `private-access-rotation-record` processed with non-matching key | Stop signing / honoring signatures under the old key | Immediate | `60-keys-and-access.md §6.2`, `70-durability.md §2.4` |
| Challenge record received | Freeze non-public content release for hash | Immediate | `50-governance.md §7.2`, `70-durability.md §2.5` |
| Resolution: `rejected` | Refuse all fetches for hash | Immediate | `30-transport.md §7.5` |
| Identity-linked node logged in | Refuse to accept `private_access_keypair`; refrain from scraping personal-identity fields | Continuous | `60-keys-and-access.md §2.1` |
| `pilot-auth-did-record` rotation processed | Stop treating VCs under the superseded `pilot_auth_did` as authoritative | Immediate | `65-pilot-auth-did.md §5`, `40-pilot-and-metadata.md §2.4` |

Operators MUST inform pilots that distributed deletion is best-effort and
that compliant obligations apply only to compliant nodes.

---

## 6. Test case to requirement mapping

The following test-file convention maps each requirement namespace to a
corresponding test document. These files are forward-looking; they are
created as implementation progresses.

| Test file | Requirement prefix |
|-----------|------------------|
| `TEST.CASES.CORE.md` | `R-CORE-*` |
| `TEST.CASES.ARTIFACTS.md` | `R-ART-*` |
| `TEST.CASES.TRANSPORT.md` | `R-TRANS-*` |
| `TEST.CASES.METADATA.md` | `R-META-*` |
| `TEST.CASES.GOVERNANCE.md` | `R-GOV-*`, `R-GSYNC-*` |
| `TEST.CASES.KEYS.md` | `R-ACCESS-*` |
| `TEST.CASES.AUTH.md` | `R-AUTH-*` |
| `TEST.CASES.DURABILITY.md` | `R-DUR-*` |
| `TEST.CASES.ANALYTICS.md` | `R-ANA-*` |
| `TEST.CASES.THREAT.md` | `R-THREAT-*` |

`R-*` labels are embedded inline at the relevant MUST/MUST NOT statements
in each normative document. Coverage is verifiable by grepping for a label
in both the spec file and its corresponding test file.
