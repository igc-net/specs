# igc-net — Governance

**Status:** Normative  
**Depends on:** `10-core.md`, `20-artifacts.md`

---

## 1. Permissioned federation model

igc-net uses a human-curated **trusted resolver roster**. Only resolvers
listed on this roster may issue governance records that the network accepts:
approvals, challenges, resolutions, and roster updates.

All nodes may publish blobs and claims. Governance attestations from nodes not
on the trusted resolver roster are ignored.

This is a permissioned federation. The benefit is simplicity and predictable
behaviour. The cost is that governance is not permissionless.

---

## 2. Trusted resolver roster

### 2.1 Resolver profile shape

```json
{
  "schema": "igc-net/resolver-profile",
  "schema_version": 1,
  "record_id": "<blake3-of-canonical-record-without-signature>",
  "resolver_id": "<ed25519-public-key-hex>",
  "display_name": "Example Resolver",
  "service_url": "https://resolver.example.org",
  "privacy_policy_url": "https://resolver.example.org/privacy",
  "public_key_url": "https://resolver.example.org/.well-known/igc-net-resolver-key",
  "created_at": "YYYY-MM-DDTHH:MM:SSZ",
  "signature": "<ed25519-signature-hex>"
}
```

### 2.2 Verification rules

- The profile MUST be signed by the key identified in `resolver_id`.
- `public_key_url` MUST serve the same public key bytes as `resolver_id`.
- `service_url`, `privacy_policy_url`, and `public_key_url` MUST be public URLs.
- `created_at` is informational; it is NOT the authority for roster ordering.

### 2.3 Bootstrap — pinned project-root trust model

The igc-net project maintains a canonical resolver roster signed by a pinned
project root key. This key is published at a well-known project URL and is
pinned: it is not discoverable through the protocol itself.

**Bootstrap procedure:** `(R-GOV-10)`

1. Operator fetches the canonical roster out of band (website or project URL).
2. Operator verifies the roster signature against the pinned project root key.
3. Operator pins the project root key locally — it is the top-level trust anchor.
4. Node may then automatically consume subsequent signed governance records from
   the listed trusted resolvers.

Roster authenticity traces to the pinned project root key. The DNS, public key
infrastructure, or any gossip-discoverable key MUST NOT be used as a substitute
for the pinned project root key.

The long-term path is to migrate roster governance to an independent community
body. Operators should understand that the initial bootstrap authority is the
igc-net project.

### 2.4 Roster updates

A roster-update record MUST be signed by either: `(R-GOV-16)`

- the current project root key, OR
- a resolver already on the roster with roster-update authority.

```json
{
  "schema": "igc-net/roster-update",
  "schema_version": 1,
  "record_id": "<blake3-of-canonical-record-without-signature>",
  "action": "add|remove",
  "resolver_id": "<ed25519-public-key-hex>",
  "signer_id": "<ed25519-public-key-hex>",
  "resolver_profile": {
    "display_name": "Example Resolver",
    "service_url": "https://resolver.example.org",
    "privacy_policy_url": "https://resolver.example.org/privacy",
    "public_key_url": "https://resolver.example.org/.well-known/igc-net-resolver-key"
  },
  "created_at": "YYYY-MM-DDTHH:MM:SSZ",
  "signature": "<ed25519-signature-hex>"
}
```

`resolver_profile` MUST be present when `action` is `"add"`. It MUST be absent
when `action` is `"remove"`. A node receiving a roster-update with `action: "add"`
MUST use the embedded `resolver_profile` to materialize the resolver's profile —
no separate out-of-band lookup is required.

`resolver_id` identifies the resolver being added or removed, not the signer.
`signer_id` identifies the actual signer of this record. Verifiers MUST verify
the signature against `signer_id` and MUST confirm that `signer_id` is either:
- the pinned project root key, OR
- a resolver already on the trusted roster with roster-update authority.

A roster-update where `signer_id` is not an authorized key MUST be rejected even
if the signature over the record is cryptographically valid. `(R-GOV-21)`

---

## 3. Claims

A claim is a signed assertion by a pilot that they own a specific flight.

`owner` is the only governance claim type. `uploader` and `custodian`
are operational roles; they carry no governance weight and are not recorded as
claim types.

### 3.1 Claim shape

```json
{
  "schema": "igc-net/claim",
  "schema_version": 1,
  "record_id": "<blake3-of-canonical-claim-without-signature>",
  "raw_igc_hash": "<blake3-hex>",
  "claim_type": "owner",
  "pilot_id": "igcnet:id:<root-public-key-hex>",
  "signature": "<ed25519-signature-hex>",
  "created_at": "YYYY-MM-DDTHH:MM:SSZ",
  "evidence": []
}
```

### 3.2 Signing rules

- Signed by `pilot_id` following `10-core.md §5`. `(R-GOV-17)`
- `record_id = BLAKE3(canonical_json(record_without_signature))`.
- `created_at` is informational; see `10-core.md §1.5`.

---

## 4. Evidence

A claim MAY include supporting evidence in the `evidence` array. Evidence
supports a claim but does not alone determine the outcome — a trusted resolver
evaluates it and issues an approval or rejection using their own judgement.
Private evidence MAY remain resolver-local; only a basis label need be
published if the underlying material is sensitive.

---

## 5. Governance state machine

### 5.1 States

| State | Meaning |
|-------|---------|
| `pending` | Claim submitted; no approval yet |
| `approved` | Claim approved by a trusted resolver; pilot is accepted owner |
| `contested` | Approval challenged or conflicting approvals detected |
| `rejected` | Claim rejected by resolution |
| `superseded` | Claim replaced by a resolution accepting a different claimant |
| `revoked` | Accepted owner revoked their own claim |

A resolution record whose `resolution` value is `"revoked"` (§8.2) produces
the `revoked` governance state. These are two levels of the same concept:
the resolution value names the outcome; the governance state reflects it.

### 5.2 Normal path

```
pending → (approval) → approved
```

### 5.3 Dispute path

```
approved → (challenge) → contested → (resolution) → approved | rejected | superseded
```

### 5.4 Self-revocation fast path

```
approved → (self-revocation + competing claim) → resolver fast-path resolution
```

If the accepted owner issues a signed self-revocation and at least one competing
claim exists, a trusted resolver MAY resolve directly without a challenge record
or the full contested-state path. `(R-GOV-20)`

### 5.5 Automatic contested state

If two trusted resolvers independently approve two different owner claims for
the same `raw_igc_hash`, compliant nodes MUST detect this and treat the second
approval as an automatic challenge. `(R-GOV-01)` The flight enters `contested` state without
requiring a separate challenge record.

---

## 6. Approval

One trusted resolver approval is sufficient to advance a claim to `approved`
state. `(R-GOV-18)`

### 6.1 Approval shape

```json
{
  "schema": "igc-net/claim-approval",
  "schema_version": 1,
  "record_id": "<blake3-of-canonical-record-without-signature>",
  "claim_record_id": "<blake3-hex>",
  "raw_igc_hash": "<blake3-hex>",
  "resolver_id": "<ed25519-public-key-hex>",
  "signature": "<ed25519-signature-hex>",
  "created_at": "YYYY-MM-DDTHH:MM:SSZ"
}
```

- Signed by `resolver_id` following `10-core.md §5`.
- `claim_record_id` is the `record_id` of the claim being approved.
- `created_at` is informational; see `10-core.md §1.5`.

---

## 7. Challenge

A trusted resolver may challenge an existing approval.

### 7.1 Challenge shape

```json
{
  "schema": "igc-net/claim-challenge",
  "schema_version": 1,
  "record_id": "<blake3-of-canonical-record-without-signature>",
  "claim_record_id": "<blake3-hex>",
  "raw_igc_hash": "<blake3-hex>",
  "challenger_resolver_id": "<ed25519-public-key-hex>",
  "signature": "<ed25519-signature-hex>",
  "reason": "ownership_dispute",
  "created_at": "YYYY-MM-DDTHH:MM:SSZ"
}
```

- Signed directly from the challenging resolver's identity key (`challenger_resolver_id`).
- `claim_record_id` is the `record_id` of the claim being challenged.

### 7.2 Challenge effects

- The flight transitions from `approved` to `contested`. `(R-GOV-02)`
- No further private content release MUST occur while contested. `(R-GOV-03)`
- Serving nodes MUST refuse fetch requests for `contested` flights. `(R-GOV-04)`

### 7.3 Challenge expiry

If no resolution is recorded within **15 days** of the challenge record's own
`created_at`, the challenge lapses and the previously approved claim reverts to
`approved` state. `(R-GOV-19)`

`created_at` in the challenge record is the expiry baseline for this purpose
only. It is NOT a general record-ordering authority.

Trusted resolver operators SHOULD review open challenges regularly and close
them within this window.

### 7.4 Challenger roster revocation

If the challenging resolver's roster membership is revoked during an open
dispute, the challenge is treated as withdrawn and the flight reverts to
`approved`. `(R-GOV-14)`

---

## 8. Resolution

A resolution records the final outcome for a contested or disputed claim.

### 8.1 Resolution shape

```json
{
  "schema": "igc-net/claim-resolution",
  "schema_version": 1,
  "record_id": "<blake3-of-canonical-record-without-signature>",
  "raw_igc_hash": "<blake3-hex>",
  "claim_record_id": "<blake3-hex>",
  "resolver_id": "<ed25519-public-key-hex>",
  "signature": "<ed25519-signature-hex>",
  "resolution": "approved|rejected|superseded|revoked",
  "basis": ["manual_review"],
  "created_at": "YYYY-MM-DDTHH:MM:SSZ",
  "supersedes": []
}

```

`claim_record_id` is the `record_id` of the claim being resolved.

`supersedes` is an **array** of `record_id` values of prior resolutions for
this same claim whose outcome this resolution overrides. Set to `[]` if this is
the first resolution for this claim. A counter-resolution (`§8.3`) MUST list
the `record_id` of each resolution it overrides.

Note: `supersedes` on resolution records is an array because a
counter-resolution may need to override multiple prior conflicting resolutions
simultaneously. Other record types (`publication-mode-record`,
`private-access-rotation-record`, `pilot-auth-did-record`) use a scalar
`supersedes` because they form a single-predecessor chain.

### 8.2 Resolution values

| Value | Meaning |
|-------|---------|
| `approved` | Claim is confirmed; pilot is accepted owner |
| `rejected` | Claim is denied |
| `superseded` | A competing claim is approved; this one is replaced |
| `revoked` | Accepted owner or their key is revoked |

### 8.3 Counter-resolution

A second trusted resolver may issue a counter-resolution overriding a stale or
unclosed challenge from a different resolver. `(R-GOV-15)` The counter-resolution
MUST list the `record_id` of the resolution it overrides in its `supersedes` array.

**Tiebreak rule:** When two resolutions for the same claim are both in scope and
neither's `supersedes` list references the other, the resolution whose
`record_id` is lexicographically smallest as a 64-character lowercase hex string
takes precedence. This is deterministic across all compliant nodes. `(R-GOV-22)`

A conflicting party MAY always initiate a new challenge to re-enter `contested`
state regardless of which resolution is currently active.

---

## 9. Double claims

If the same `raw_igc_hash` has multiple active owner claims and at least one
trusted challenge or conflicting governance state:

- The flight is marked `contested`.
- All claims and governance records are preserved.
- No further private content release is permitted until resolved.

---

## 10. Key loss and recovery

Key loss recovery is handled at the identity layer through resolver-assisted
re-claim. The outcome is recorded as an `identity-recovery` record, which is
fleet-wide: it transfers ownership of all flights linked to the old identity
in a single record, rather than requiring a per-flight `claim-resolution`.

### 10.1 Recovery flow

1. Pilot creates a new identity keypair.
2. Pilot proves continuity through a trusted portal or resolver using available
   account and provenance evidence.
3. Trusted resolver reviews the recovery request.
4. Resolver records an `identity-recovery` record (§10.2) linking `old_pilot_id`
   to `new_pilot_id`.

One `identity-recovery` record covers all flights linked to `old_pilot_id`. `(R-GOV-11)`
Compliant nodes MUST transfer accepted-owner status for all such flights to
`new_pilot_id` upon receiving a valid `identity-recovery` record.

Recovery transfers accepted-owner status to `new_pilot_id` but does not itself
re-establish private access. Because `private_access_keypair` authorization is
bound to `pilot_id` through a signed `private-access-rotation-record` (see
`60-keys-and-access.md §6`), the pilot MUST publish a fresh rotation record
signed by `new_pilot_id` after recovery. If the pilot still holds the previous
`private_access_keypair`, they MAY re-publish its public half under the new
identity; if the keypair itself is also lost, the pilot generates a new one.
Private-access nodes whose rotation record was bound to the old identity MUST
stop honoring fetch-request signatures until the new rotation record arrives on
the governance topic.

Recovery also leaves the `pilot_auth_did` chain unset under `new_pilot_id`.
No `PilotProfileCredential` for the pilot is verifiable by relying parties
until the pilot publishes a new `pilot-auth-did-record` signed by `new_pilot_id`
(see `65-pilot-auth-did.md §8`). The pilot SHOULD publish this record promptly
after recovery.

### 10.2 Identity-recovery record shape

```json
{
  "schema": "igc-net/identity-recovery",
  "schema_version": 1,
  "record_id": "<blake3-of-canonical-record-without-signature>",
  "old_pilot_id": "igcnet:id:<old-root-public-key-hex>",
  "new_pilot_id": "igcnet:id:<new-root-public-key-hex>",
  "resolver_id": "<ed25519-public-key-hex>",
  "basis": "key_loss_recovery|key_compromise_recovery",
  "created_at": "YYYY-MM-DDTHH:MM:SSZ",
  "signature": "<ed25519-signature-hex>"
}
```

- Signed by `resolver_id` following `10-core.md §5`.
- `record_id = BLAKE3(canonical_json(record_without_signature))`.
- Propagated on the governance topic.
- `old_pilot_id` and `new_pilot_id` MUST be distinct.
- A single record covers all flights; no `raw_igc_hash` or `claim_record_id`
  field is present.

### 10.3 Recovery basis values

| Value | Meaning |
|-------|---------|
| `key_loss_recovery` | Key was lost; continuity proven by other means |
| `key_compromise_recovery` | Key may be compromised; new key issued |

### 10.4 Key compromise

If the old key may be compromised:

- Set `basis: "key_compromise_recovery"` in the `identity-recovery` record.
- Compliant nodes MUST reject further claims or governance records signed with
  `old_pilot_id` after receiving the recovery record. `(R-GOV-05)`
- Past disclosures made before the compromise was known cannot be undone.

---

## 11. Publication-mode record

The `publication-mode-record` establishes and updates the `publication_mode`
for a flight. Its full shape and mode-transition rules are defined in
`20-artifacts.md`. The governance-plane obligations are:

- At upload time, the **claimant** asserts the initial `publication_mode`.
  Privacy-restrictive modes are honored immediately. `(R-GOV-13)`
- After ownership is resolved, only the **accepted owner** may change the mode
  by signing a new `publication-mode-record`.
- `publication_mode` changes are broadcast on the governance topic. `(R-GOV-12)`
- Mode upgrade (more restrictive): serving nodes MUST stop serving previously
  permitted artifacts immediately. `(R-GOV-06)`

---

## 12. Private-access rotation record

The `private-access-rotation-record` establishes and updates the public key
that serving nodes use to verify fetch-request signatures for a pilot's
non-public content. Its full shape and semantics are defined in
`60-keys-and-access.md §6`. The governance-plane obligations are:

- Signed directly by the pilot's root identity key (`pilot_id`).
- Propagated on the governance topic alongside other policy records. `(R-GOV-23)`
- A new record supersedes the previous active record via its `supersedes`
  field. Serving nodes MUST honor the most recent active record when
  verifying fetch-request signatures.
- A rotation invalidates future authorization by the previous
  `private_access_keypair` but does not itself remove plaintext content
  already held at private-access nodes; local deletion at those nodes is
  covered by `60-keys-and-access.md §7`.

---

## 13. Deletion request

A deletion request is a signed record from the accepted owner requesting
compliant nodes to remove flight data.

### 13.1 Deletion request shape

```json
{
  "schema": "igc-net/deletion-request",
  "schema_version": 1,
  "record_id": "<blake3-of-canonical-request-without-signature>",
  "raw_igc_hash": "<blake3-hex>",
  "pilot_id": "igcnet:id:<root-public-key-hex>",
  "signature": "<ed25519-signature-hex>",
  "created_at": "YYYY-MM-DDTHH:MM:SSZ"
}
```

- `record_id = BLAKE3(canonical_json(record_without_signature))`.
- Signed directly from the accepted owner's identity key (`pilot_id`).

### 13.2 Compliant-node obligations

Upon receiving a valid deletion request for a `raw_igc_hash`:

1. Stop serving all artifacts for this `raw_igc_hash` immediately. `(R-GOV-07)`
2. Remove all flight-scoped governance records and indexes for this hash within
   30 days. `(R-GOV-08)`
3. Remove local metadata-advertisement indexes for this `raw_igc_hash` where
   the local node is the publisher. Remote public advertisements may persist
   until their publishers remove or supersede them.

Deletion does **NOT** delete the pilot's identity-level profile authority.
Deletion ends the flight's association with the pilot for display and custody
purposes; it does not erase the pilot's identity.

Complete distributed deletion is best-effort in a decentralised network.
Non-compliant nodes cannot be cryptographically forced to delete. Compliant
nodes MUST inform pilots that distributed deletion is a protocol obligation for
compliant peers only.
