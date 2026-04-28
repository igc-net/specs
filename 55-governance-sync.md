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

## 4. Gossip processing semantics

### 4.1 Validation

A node receiving a governance record MUST validate it before applying it to
governance state or relaying it as accepted state. `(R-GSYNC-12)`

At minimum, validation includes:

- supported `schema` and `schema_version`
- well-formed fields for that schema
- `record_id = BLAKE3(canonical_json(record_without_signature))`
- signature verification against the schema's signing authority
- signer authorization, including trusted-resolver roster authority for
  resolver-signed records
- schema-specific dependency checks defined in the owning document

Invalid governance records MUST NOT update governance state and MUST NOT be
used to authorize serving, key provisioning, authentication, deletion, or mode
changes. `(R-GSYNC-13)` Implementations MAY retain invalid records in local
diagnostics, but invalid records are outside the authoritative governance
state.

### 4.2 Idempotency and duplicate handling

Governance record processing is idempotent. Re-receiving a valid record whose
`record_id` is already applied MUST NOT change the resulting governance state,
emit a second state transition, or reset freshness markers. `(R-GSYNC-14)`

Nodes MAY gossip, replay, or return duplicate records during catch-up. Receivers
MUST deduplicate by `record_id`.

### 4.3 Dependency ordering

The governance topic does not provide a reliable global total order. Correctness
comes from explicit record references and deterministic conflict rules, not from
arrival order or `created_at`. `(R-GSYNC-15)`

A node that receives a validly signed record referencing an unknown prerequisite
record MUST keep it pending and MUST NOT treat it as authoritative until the
referenced record is known and valid. `(R-GSYNC-16)` This applies to:

- approvals, challenges, and resolutions that reference `claim_record_id`
- resolution records that reference prior resolutions in `supersedes`
- publication-mode records that supersede earlier mode records
- private-access rotation records that supersede earlier rotation records
- pilot-auth-DID records that supersede earlier authentication records
- roster updates whose signer authority depends on current roster state

If two valid records remain concurrent after dependency resolution, the
schema-specific deterministic tiebreak rule applies. If a schema has no
tiebreak rule for that concurrency class, the affected governance state is
stale and MUST fail closed for governance-dependent operations until a valid
record resolves the conflict. `(R-GSYNC-17)`

### 4.4 Stale-state failure modes

Governance state is stale for an artifact, pilot, resolver roster, or key chain
when the node knows that required dependencies or catch-up ranges are missing.
While stale, the node MUST fail closed for operations whose correctness depends
on that state. `(R-GSYNC-18)`

Fail-closed behavior includes:

- no restricted artifact serving for affected `raw_igc_hash` or pilot
- no private-access key provisioning for affected pilot
- no acceptance of superseding key or authentication records as authoritative
  until their chains are complete
- no honoring resolver-signed records whose signer roster authority is unknown
- no re-announcement of artifact classes whose serving permission depends on
  the stale state

The node MAY continue data-plane discovery and MAY serve artifacts whose
governance state is current and unaffected by the stale dependency.

### 4.5 Catch-up closure

Catch-up MUST recover enough governance dependency closure to make the target
state authoritative. `(R-GSYNC-19)` A catch-up response or replay that returns a
record without its required ancestors is incomplete for that target until those
ancestors are fetched and validated.

Time-range catch-up is sufficient only when it bridges from a durable baseline
that already contains the required ancestors. If the baseline is missing an
ancestor required by a fetched record, the implementation MUST fetch by
`record_id`, by `raw_igc_hash`, by pilot chain, or by another mechanism that
recovers the dependency closure before declaring the target state current.

---

## 5. Catch-up and delivery requirements

### 5.1 Catch-up obligation

Nodes MUST be able to catch up on governance records missed while offline. `(R-GSYNC-02)`

An implementation MUST support at minimum **one** of: `(R-GSYNC-03)`

1. A gossip history window of at least 30 days depth, allowing a rejoining
   node to receive records it missed by replaying the topic history.
2. A pull-based sync endpoint exposed by serving nodes, allowing a rejoining
   node to fetch governance records by `raw_igc_hash` or by time range.

### 5.2 Durable governance baseline

A node that serves artifacts whose availability depends on governance state
MUST maintain a durable governance baseline. `(R-GSYNC-08)`

A durable governance baseline is the node's persisted, crash-recoverable view
of applied governance records and conflict-resolution state. It MAY be stored
as the applied record log, as a snapshot plus replay cursor, or as another
implementation-specific checkpoint, provided it is sufficient to reconstruct
serving decisions for:

- claims, approvals, challenges, and resolutions
- deletion requests
- publication-mode records
- private-access rotation records
- pilot-auth-DID records
- resolver roster updates

The baseline MUST be updated atomically with application of governance records.
After a crash or restart, a node MUST either recover the last durable baseline
and catch up from it, or mark governance state unavailable and fail closed for
governance-dependent serving. `(R-GSYNC-09)`

The 3-day public discovery catch-up window in `30-transport.md §8` is only an
announcement freshness window. It is not a substitute for a durable governance
baseline. A node without a valid governance baseline MAY index freshly
discovered public/protected-sanitized announcements, but MUST NOT claim complete
serving authority for governance-dependent artifacts until governance catch-up
from a durable baseline succeeds. `(R-GSYNC-10)`

### 5.3 Delivery priority

Revocations and mode-change records MUST be processed before any new
private-access decision for the same `raw_igc_hash`.

A node MUST NOT serve restricted content (private raw IGC or protected raw
companion) for a `raw_igc_hash` until it has processed all pending governance
records for that hash. `(R-GSYNC-04)`

### 5.4 Startup catch-up for igc-net-held private access

An igc-net service that holds one or more pilots' `private_access_keypair` values MUST
perform full governance catch-up for those pilots on startup before serving
protected raw companions or private raw IGC for those pilots. `(R-GSYNC-06)`

Full catch-up means igc-net has reconstructed current governance state for
the pilot's known and discoverable artifacts, including claims, approvals,
challenges, resolutions, deletion requests, publication-mode records,
private-access rotation records, and pilot-auth-DID records relevant to
serving decisions.

For pilots whose private-access keys are held locally, startup catch-up MUST
start from a recovered durable governance baseline. If no baseline is available,
igc-net MUST build one through an implementation-supported full governance sync
before serving restricted artifacts for those pilots. `(R-GSYNC-11)`

If full catch-up cannot be completed, or if the recovered baseline is missing,
corrupt, or too old for the implementation's available catch-up mechanism,
igc-net MUST fail closed for restricted artifacts for the affected pilots.
`(R-GSYNC-07)` It MAY still perform public discovery and
public/protected-sanitized indexing under the data-plane freshness rules in
`30-transport.md §8`, provided it does not serve restricted bytes.

### 5.5 Best-available-state rule `(R-GSYNC-05)`

A node that served content before receiving a challenge or revocation is NOT in
protocol violation, provided:

- It acted on the best governance state available at the time of serving.
- It processes the challenge or revocation promptly upon receipt.
- It immediately stops serving restricted content for the affected
  `raw_igc_hash` after processing the update.

Past serving decisions made in good faith under the then-current governance
state are not retroactively penalised.
