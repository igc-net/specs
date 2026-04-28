# igc-net — Pilot Authentication DID Records

**Status:** Normative

**Depends on:** `10-core.md`, `50-governance.md`,
`55-governance-sync.md`, `60-keys-and-access.md`

---

## 1. Purpose

A pilot's cross-site identity is anchored by `pilot_id`: a non-rotating
Ed25519 root public key that signs all pilot-authored governance records
(claims, publication-mode records, deletion requests,
`private-access-rotation-record`). For cross-portal authentication and for
issuing self-signed Verifiable Credentials such as pilot profile credentials,
the pilot
uses a second, wallet-held credential: `pilot_auth_did`.

Because `pilot_auth_did` lives in a pilot wallet and may be rotated
independently of `pilot_id`, serving nodes and relying parties require an
authoritative, pilot-signed binding from `pilot_id` to the currently
active `pilot_auth_did`. This document specifies that binding as the
`pilot-auth-did-record` governance record.

The role of `pilot_auth_did` is deliberately scoped: it is an
authentication credential and a VC-issuer credential. It does NOT sign
governance records and it does NOT authorize fetches for private content.
Governance authority remains with `pilot_id`; private-content authorization
remains with `private_access_keypair` (see `60-keys-and-access.md`).

---

## 2. Record shape

```json
{
  "schema": "igc-net/pilot-auth-did-record",
  "schema_version": 1,
  "record_id": "<blake3-of-canonical-record-without-signature>",
  "pilot_id": "igcnet:id:<root-public-key-hex>",
  "pilot_auth_did": "did:key:z6Mk...",
  "supersedes": "<previous-record-id-or-null>",
  "created_at": "YYYY-MM-DDTHH:MM:SSZ",
  "signature": "<ed25519-signature-hex>"
}
```

### 2.1 Field rules

- `schema` MUST be exactly the string `"igc-net/pilot-auth-did-record"`.
  `(R-AUTH-01)`
- `schema_version` is the integer `1` for this specification.
- `record_id = BLAKE3(canonical_json(record_without_signature))`, encoded
  as 64-character lowercase hex. `(R-AUTH-02)`
- `pilot_id` is the pilot's root identity, formatted exactly as defined
  in `10-core.md §3`.
- `pilot_auth_did` is the pilot's canonical wallet-held authentication DID.
  It MUST be a `did:key` string. Portal-issued or public-facing
  `did:web` identifiers do not replace this field and are not authoritative for
  pilot binding. Implementations MUST parse the DID string and reject records
  whose `pilot_auth_did` does not parse as a valid `did:key`. `(R-AUTH-03)`
- `supersedes` MUST be either:
  - `null` for the initial `pilot-auth-did-record` issued for this
    `pilot_id`, or
  - the 64-character lowercase hex `record_id` of the previous
    authoritative `pilot-auth-did-record` for this `pilot_id`.
- `created_at` uses the RFC 3339 subset from `10-core.md §1.5`; it is
  informational (see `10-core.md §1.5`).
- `signature` is a 64-byte Ed25519 signature, encoded as 128-character
  lowercase hex.

### 2.2 Example

```json
{
  "schema": "igc-net/pilot-auth-did-record",
  "schema_version": 1,
  "record_id": "8a37f5...e2",
  "pilot_id": "igcnet:id:1b2c3d4e5f...ab",
  "pilot_auth_did": "did:key:z6MkeTG3bFFSLYVU7VqhgZxqr6YzpaGrQtFMh1uvqGy1vDnP",
  "supersedes": null,
  "created_at": "2026-05-01T09:14:00Z",
  "signature": "a9f2c4...17"
}
```

---

## 3. Signing

The signing authority for `pilot-auth-did-record` is `pilot_id`. The
pilot's root Ed25519 private key signs the record. `pilot_auth_did` does
NOT sign this record. `(R-AUTH-04)`

Signing and verification follow `10-core.md §5` exactly, using the public key
derived from `pilot_id` as the verification key. A record whose signature
fails to verify MUST be rejected. `(R-AUTH-05)`

`10-core.md §5.3` lists the schemas signed by `pilot_id`;
`igc-net/pilot-auth-did-record` belongs to that row.

---

## 4. Authority and supersession

At any time, at most one `pilot-auth-did-record` is authoritative for a
given `pilot_id`. The authoritative record determines the current
`pilot_auth_did` for that pilot and is used by relying parties and
serving nodes for VC verification and cross-site authentication.

### 4.1 Supersession chain

A pilot MAY publish multiple `pilot-auth-did-record`s over time. These
form a chain linked by the `supersedes` field:

- The initial record has `supersedes = null`.
- Each subsequent record sets `supersedes` to the `record_id` of the
  record it replaces.

The authoritative record is the record at the tip of the chain: the
record no other record supersedes. `(R-AUTH-06)`

Authority does NOT depend on `created_at`. This matches the general
supersession convention in `50-governance.md` and invariant #9 in
`00-overview.md §10`: the supersession graph is the ordering authority,
not wall-clock timestamps.

### 4.2 Concurrent publication tiebreak

If two or more well-formed, signature-valid `pilot-auth-did-record`s for
the same `pilot_id` each claim to supersede the same parent (or both
have `supersedes = null`), the authoritative record is the one whose
`record_id` is lexicographically smallest when compared as
64-character lowercase hex strings. `(R-AUTH-07)`

This tiebreak rule is deterministic across implementations. Non-selected
concurrent records remain signature-valid and are retained for audit,
but are not used for current VC verification or authentication.

### 4.3 Incomplete chains

A node receiving a record whose `supersedes` references an unknown
`record_id` MUST NOT treat the received record as authoritative until
the referenced ancestor has been fetched and verified. The node
continues to use the previously known authoritative record, if any, in
the interim. Catch-up MUST resolve the chain before the newer record
replaces the current authority. `(R-AUTH-08)`

A node that has never observed any `pilot-auth-did-record` for a given
`pilot_id` has no current `pilot_auth_did` and MUST reject VCs that
claim to be signed by a `pilot_auth_did` for that pilot.
`(R-AUTH-09)`

### 4.4 Signature failure

A record whose signature does not verify under `pilot_id` is not part of
the chain. Implementations MUST NOT use it to determine authority, MUST
NOT propagate it as authoritative, and SHOULD drop it.

---

## 5. Rotation

To rotate `pilot_auth_did`, a pilot:

1. Generates a new `pilot_auth_did` keypair in their wallet as a new `did:key`.
2. Constructs a `pilot-auth-did-record` in which:
   - `pilot_auth_did` is the new DID,
   - `supersedes` is the `record_id` of the currently authoritative
     record.
3. Signs with `pilot_id` (§3).
4. Publishes on the governance topic (§7).

After the new record is accepted as authoritative on a serving or relying
node:

- New `PilotProfileCredential` JWTs SHOULD be re-issued under the new
  `pilot_auth_did`. Relying parties MUST reject VCs signed by the
  superseded `pilot_auth_did` once the rotation record is authoritative
  on that node. `(R-AUTH-10)`
- Cross-site authentication sessions that were established using the
  superseded `pilot_auth_did` SHOULD be revalidated or invalidated by
  the portal that holds them.

Rotation does NOT affect `pilot_id`, the `private_access_keypair`, the
private-access rotation chain, or any previously-signed governance
record.

---

## 6. Validity

A `pilot-auth-did-record` is valid if and only if all of the following
hold:

- All fields listed in §2.1 are present and well-formed.
- The record's canonical JSON parses under RFC 8785.
- `signature` verifies against the Ed25519 public key derived from
  `pilot_id` (§3).
- `pilot_auth_did` parses as a DID of an accepted method (§2.1).
- If `supersedes` is non-null, the referenced record is known to the
  verifying node and is itself valid.
- `created_at` parses under the RFC 3339 subset in `10-core.md §1.5`.

An invalid record is not authoritative and MUST NOT be used for
authentication or VC verification. `(R-AUTH-11)`

---

## 7. Governance-plane behaviour

### 7.1 Topic

`pilot-auth-did-record` travels on the governance topic
`igc-net/governance/v1` alongside claims, approvals, publication-mode
records, and `private-access-rotation-record`. See `55-governance-sync.md §1`
and `30-transport.md §1` for topic identifiers.

### 7.2 Propagation

Compliant nodes subscribed to the governance topic MUST accept and
propagate well-formed, signature-valid `pilot-auth-did-record`s under
the rules in `55-governance-sync.md`. Propagation is governed by the
same anti-flooding and rate-limiting obligations that apply to other
per-pilot governance records.

### 7.3 Catch-up

A newly-joining node MUST fetch the full supersession chain for a
given `pilot_id` before treating any `PilotProfileCredential` or
cross-site authentication token signed by that pilot's
`pilot_auth_did` as valid. `(R-AUTH-12)` Partial-chain state produces
only a tentative authority that SHOULD NOT be used for high-trust
actions (e.g., granting portal access) until the chain resolves.

### 7.4 Independence from `private-access-rotation-record`

`pilot-auth-did-record` and `private-access-rotation-record` are two
independent per-pilot governance record types, both signed by
`pilot_id`. They carry disjoint payloads:

- `pilot-auth-did-record` binds `pilot_auth_did` (authentication /
  VC-issuer credential).
- `private-access-rotation-record` binds `private_access_public_key`
  (private-content fetch authorization).

They have independent supersession chains. A rotation of one does NOT
rotate the other. `(R-AUTH-13)`

---

## 8. Initial publication

A pilot establishes their first `pilot_auth_did` by publishing a
`pilot-auth-did-record` with `supersedes = null`. Before this record is
accepted by a relying party:

- No `PilotProfileCredential` for that pilot is verifiable by the
  relying party.
- No cross-site authentication token signed by any wallet-held key is
  authoritative for that pilot.

Compliant relying parties MUST NOT treat wallet-held credentials as
authoritative in the absence of a `pilot-auth-did-record` for the
claimed `pilot_id`. `(R-AUTH-14)`

A pilot MAY operate without a `pilot_auth_did`. In that case the pilot
publishes native records (signed by `pilot_id`) and authorises private
access by `private_access_keypair` as usual, but does not present
VC-based profile data or cross-site authentication tokens.

---

## 9. Compromise and recovery

If a pilot's `pilot_auth_did` private key is compromised:

- The pilot MUST rotate `pilot_auth_did` (§5) using `pilot_id` to sign
  the new `pilot-auth-did-record`.
- `PilotProfileCredential` JWTs signed by the compromised
  `pilot_auth_did` SHOULD be re-issued under the new credential.
- Cross-site authentication sessions authenticated by the compromised
  `pilot_auth_did` SHOULD be invalidated by the portals that hold them.

Because `pilot_auth_did` does not sign governance records and does not
authorise fetches for private content, a compromise of
`pilot_auth_did` does NOT:

- Compromise `pilot_id` or any governance authority.
- Compromise `private_access_keypair` or authorization for private
  content.
- Enable ownership transfer, publication-mode change, deletion, or any
  record-plane operation.

The blast radius is bounded to authentication and VC issuance. Recovery
is a single governance publication (the rotation record) without any
dependency on the resolver network.
