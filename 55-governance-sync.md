# igc-net — Governance Sync

**Status:** Normative  
**Depends on:** `50-governance.md`, `30-transport.md`

---

## Why this document exists separately

Governance sync is not the same as data-plane sync.

The data plane tolerates stale availability state: a serving peer going offline
or returning a stale ticket is recoverable. Governance state cannot be stale:
stale governance state causes wrongful private content release or wrongful
ownership decisions. That asymmetry is why governance sync has its own document
and its own topic.

---

## 1. Governance topic

```
governance_topic_id = BLAKE3(UTF-8("igc-net/governance/v1"))[0..32]
```

Derivation rules:

- Input: the exact UTF-8 bytes of the string `"igc-net/governance/v1"`, with no
  null terminator and no length prefix.
- Output: the first 32 bytes of the BLAKE3 hash, encoded as 64-character
  lowercase hex.

All governance records are broadcast on this topic. The data-plane announce
topic (defined in `30-transport.md`) is separate; governance records MUST NOT
be broadcast on the announce topic. `(R-GSYNC-01)`

---

## 2. Record types that propagate on the governance topic

The following record types are broadcast on the governance topic:

- `claim` — pilot ownership claim
- `claim-approval` — trusted resolver approval
- `claim-challenge` — trusted resolver challenge
- `claim-resolution` — trusted resolver resolution
- `identity-recovery` — resolver-issued fleet-wide identity transfer record
- `publication-mode-record` — owner-signed mode change
- `deletion-request` — owner-signed erasure request
- `roster-update` — roster add or remove record
- `private-access-rotation-record` — pilot-signed authorization-key rotation
- `pilot-auth-did-record` — pilot-signed authentication-DID binding and rotation

Private-flight existence records (`raw_igc_hash` and serving-node tickets)
are **NOT** propagated on the governance topic. They are data-plane
announcements that belong on the announce topic defined in
`30-transport.md`.

The governance topic carries policy state only. It does not carry artifact
locators or content.

---

## 3. Common signing envelope

Every governance record MUST include these fields:

| Field | Content |
|-------|---------|
| `signature` | Ed25519 signature over RFC 8785 canonical JSON of all non-signature fields |
| `record_id` | `BLAKE3(canonical_json(record_without_signature))` |
| `created_at` | RFC 3339 timestamp in `YYYY-MM-DDTHH:MM:SSZ` format |

Records do NOT carry a separate `signing_key` field. The verification key is
derived from the record's schema-specific identity field (defined in
`10-core.md §5.3`):

- Claims, mode records, deletion requests, private-access rotation records, and
  `pilot-auth-did-record`s: signed by the `pilot_id` root key.
- Approvals, challenges, resolutions, roster updates: signed by the
  `resolver_id` key or the project root key as appropriate.
- For signing and key derivation rules see `10-core.md §5`.

`created_at` in governance records is informational; see `10-core.md §1.5`.

Nodes apply governance records in the order they are locally received. Local
processing order is not a network-wide state-selection authority: when two
valid resolutions conflict, the deterministic tiebreak defined in
`50-governance.md §8.3` applies, not the local arrival order.

---

## 4. Catch-up and delivery requirements

### 4.1 Catch-up obligation

Nodes MUST be able to catch up on governance records missed while offline. `(R-GSYNC-02)`

An implementation MUST support at minimum **one** of: `(R-GSYNC-03)`

1. A gossip history window of at least 30 days depth, allowing a rejoining
   node to receive records it missed by replaying the topic history.
2. A pull-based sync endpoint exposed by serving nodes, allowing a rejoining
   node to fetch governance records by `raw_igc_hash` or by time range.

### 4.2 Delivery priority

Revocations and mode-change records MUST be processed before any new
private-access decision for the same `raw_igc_hash`.

A node MUST NOT serve restricted content (private raw IGC or protected raw
companion) for a `raw_igc_hash` until it has processed all pending governance
records for that hash. `(R-GSYNC-04)`

### 4.3 Best-available-state rule `(R-GSYNC-05)`

A node that served content before receiving a challenge or revocation is NOT in
protocol violation, provided:

- It acted on the best governance state available at the time of serving.
- It processes the challenge or revocation promptly upon receipt.
- It immediately stops serving restricted content for the affected
  `raw_igc_hash` after processing the update.

Past serving decisions made in good faith under the then-current governance
state are not retroactively penalised.

