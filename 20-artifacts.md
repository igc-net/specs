# igc-net v0.2 — Artifacts and Publication Modes

**Status:** Normative  
**Depends on:** `10-core.md`

---

## 1. Publication modes

Every flight in igc-net has a `publication_mode`. The mode governs the
artifact access state for that flight: which byte artifacts exist, who may
fetch them, and whether a fetch requires authorization.

`publication_mode` is a property of artifact access state. It must not be
confused with `visibility`, which is a property of metadata record privacy
state (defined in `40-pilot-and-metadata.md`).

The three modes are:

| Mode | Artifact(s) held | Fetch authorization |
|------|------------------|---------------------|
| `public` | Raw IGC plaintext | None required |
| `protected` | Sanitized IGC plaintext (public) + raw IGC plaintext (companion) | None for sanitized; signed fetch request for raw companion |
| `private` | Raw IGC plaintext | Signed fetch request required |

All bytes are stored and served as plaintext. Confidentiality in flight is
provided by iroh's end-to-end transport encryption (see
`60-keys-and-access.md §9`). Confidentiality at rest on the serving node is
a compliance and legal obligation on the node operator. No protocol-level
content encryption (AEAD) is applied.

---

## 2. Public mode

In `public` mode, the raw IGC bytes are served openly. No authorization is
required to fetch the artifact.

```
raw_igc_hash = BLAKE3(raw_igc_bytes)
```

`raw_igc_hash` is the identity anchor. Any metadata policy allowed by the
pilot may be publicly associated with the flight.

No sanitization is applied to a public artifact. Any node MAY fetch a public
artifact without authorization. `(R-ART-12)`

Serving nodes MUST still apply the governance-state pre-check defined in
`30-transport.md §7.5` before transmitting.

---

## 3. Protected mode

Protected mode exposes public flight track data without exposing personal
metadata. Ownership on the public artifact is pseudonymous on the network.

Protected mode has two companion artifacts derived from the same raw IGC
bytes:

1. **Sanitized public artifact**: the sanitized `.igc` file, produced by the
   algorithm in §3.1. Served without authorization.
2. **Raw companion**: the original raw IGC bytes, unmodified. Served only to
   requesters who present a valid signed fetch request (see
   `60-keys-and-access.md §4`).

```
raw_igc_hash    = BLAKE3(raw_igc_bytes)
protected_hash  = BLAKE3(sanitized_igc_bytes)

raw_igc_hash --sanitized_as--> protected_hash
```

An accepted owner claim on `raw_igc_hash` covers the `protected_hash`
artifact linked by the `sanitized_as` relation. `(R-ART-16)`

Both artifacts are plaintext bytes at rest and on the wire (over iroh). The
raw companion is NOT encrypted at the protocol layer.

### 3.1 Normative sanitization algorithm

The sanitization algorithm produces a modified IGC file that removes
personally identifying fields. The algorithm MUST produce byte-for-byte
identical output for identical byte input across all compliant
implementations. `protected_hash` reproducibility depends on this guarantee.

**Rewrite table** — applied to IGC header lines only:

| IGC field prefix | Rewrite rule |
|-----------------|--------------|
| `HFPLT` | Replace the entire header line with exactly `HFPLT:REDACTED` |
| `HFCID` | Replace the entire header line with exactly `HFCID:REDACTED` |
| `HFGID` | Replace the entire header line with exactly `HFGID:REDACTED` |
| `HFRFW` | Replace the entire header line with exactly `HFRFW:REDACTED` |
| `HFFTYFRTYPE` | Replace the entire header line with exactly `HFFTYFRTYPE:REDACTED` |
| `HOPLT` | Replace the entire header line with exactly `HOPLT:REDACTED` |
| `HOCID` | Replace the entire header line with exactly `HOCID:REDACTED` |

**Rules for applying the rewrite table:**

- Each listed header line is **replaced**, not deleted. The output file has
  the same number of lines as the input file.
- The replacement byte content is exactly `<PREFIX>:REDACTED` followed by
  the same line ending that terminated the original line (LF or CRLF). Line
  endings are preserved; no normalisation is applied.
- All header lines with prefixes not in the rewrite table are preserved
  byte-for-byte, including their original line endings.
- All B records (tracklog) are preserved byte-for-byte.
- No other records (C, D, E, F, G, H, I, J, K, L) are modified.
- No whitespace normalisation is applied anywhere.
- The rewrite table is applied exhaustively once; no iterative passes.

An implementation MUST apply this algorithm exactly to produce a valid
`protected_hash` that other nodes can verify. `(R-ART-01)`

### 3.2 Protected artifact privacy rule

The sanitized public artifact MUST NOT contain any field that identifies
the pilot. `(R-ART-02)`

A portal MAY display pilot identity alongside a protected flight ONLY if:

- it is a private-access node (Category 2) for that pilot, holding a valid
  `private_access_keypair`, OR
- the pilot is the locally authenticated account user on a full-custody
  portal that authored the original upload.

The source of any displayed pilot identity MUST NOT be the sanitized
artifact itself. It MUST be an authorized separate record (a
`visibility: "private"` record accessed under the pilot's own
`private_access_keypair`, or a `visibility: "public"` record the pilot has
published).

---

## 4. Private mode

In `private` mode, the raw IGC bytes are served only to requesters who
present a valid signed fetch request. The bytes themselves are stored and
served as plaintext; confidentiality at rest on the serving node and
authorized requester is a compliance and legal obligation, not a
cryptographic protocol property.

Public existence records carry only the identity anchor:

```
raw_igc_hash = BLAKE3(raw_igc_bytes)
```

- `raw_igc_hash` is published: it enables deduplication, ownership claims,
  and policy-conflict detection.
- Raw IGC bytes are NOT served to any requester that does not present a
  valid fetch request signed by the pilot's currently authorized
  `private_access_keypair`. `(R-ART-14)`
- Private raw IGC bytes MUST NOT be embedded in announcements or gossiped
  as broadcast content (see `30-transport.md §2.1`).

The existence announcement for a private flight (carrying only
`raw_igc_hash`) belongs on the **data-plane announce topic**, not the
governance topic. See `30-transport.md`.

For private mode the raw IGC plaintext is itself the artifact, and its
identity is `raw_igc_hash`.

---

## 5. Sanitized artifact integrity binding

When `publication_mode` is `protected`, the pilot MUST sign both
`raw_igc_hash` and `protected_hash` together in the
`publication-mode-record`. This provides:

1. **Authorization binding**: the pilot explicitly authorized this specific
   sanitized artifact as the public representation of this flight.
2. **Tamper-evidence**: because `protected_hash = BLAKE3(sanitized_igc_bytes)`,
   any node can verify that the bytes it holds hash to `protected_hash`.
3. **Algorithm auditability**: any node that also holds the raw IGC bytes
   can independently re-run the normative sanitization algorithm and
   confirm that the result hashes to `protected_hash`.

`protected_hash` MUST be present in the `publication-mode-record` when
`publication_mode` is `"protected"`. `(R-ART-03)`

`protected_hash` MUST be absent from the `publication-mode-record` when
`publication_mode` is `"public"` or `"private"`. `(R-ART-04)`

---

## 6. Publication-mode-record

The `publication-mode-record` is the signed record that establishes and
updates the `publication_mode` for a flight.

### 6.1 Shape

```json
{
  "schema": "igc-net/publication-mode-record",
  "schema_version": 1,
  "record_id": "<blake3-of-canonical-record-without-signature>",
  "raw_igc_hash": "<blake3-hex>",
  "publication_mode": "<public|protected|private>",
  "protected_hash": "<blake3-hex>",    ← present ONLY when publication_mode is "protected"
  "supersedes": "<previous-record-id-or-null>",
  "pilot_id": "igcnet:id:<root-public-key-hex>",
  "signature": "<ed25519-signature-hex>",
  "created_at": "YYYY-MM-DDTHH:MM:SSZ"
}
```

`protected_hash` MUST be present when `publication_mode` is `"protected"`
and MUST be absent for `"public"` and `"private"` modes (see §5).

`supersedes` MUST reference the `record_id` of the previous
`publication-mode-record` for this `raw_igc_hash`, or `null` for the
initial record. Because `created_at` is not an ordering authority,
`supersedes` provides the explicit ordering chain for mode changes.

### 6.2 Signing rules

- The record MUST be signed by the pilot's root identity key (`pilot_id`).
- `record_id = BLAKE3(canonical_json(record_without_signature))`.
- `created_at` is informational. It MUST NOT be used as the mode-change
  ordering authority.

### 6.3 Initial mode authority

At upload time, the **claimant** (the uploader) asserts the initial
`publication_mode` by creating and signing this record. This record is
created as a companion to the ownership claim.

Privacy-restrictive modes (`protected`, `private`) asserted by the claimant
MUST be honored immediately by compliant nodes, before ownership is
resolved. This protects pilot privacy even in the window before a resolver
has processed the claim.

After ownership is resolved, only the **accepted owner** MAY sign
subsequent `publication-mode-record` entries that change the mode.

---

## 7. Mode transitions

### 7.1 Mode upgrade (more restrictive)

A mode upgrade makes a flight less accessible: `public` → `protected`,
`public` → `private`, or `protected` → `private`.

When a compliant serving node receives a valid mode-upgrade
`publication-mode-record` signed by the accepted owner:

- It MUST immediately stop serving the previously permitted artifact. `(R-ART-05)`
- It MUST NOT wait for a cache TTL to expire. `(R-ART-06)`
- It MUST NOT require a separate deletion request to enforce the upgrade. `(R-ART-07)`

### 7.2 Mode downgrade (less restrictive)

A mode downgrade makes a flight more accessible: `private` → `protected`,
`private` → `public`, or `protected` → `public`.

A downgrade MUST be signed by the accepted owner. No resolver involvement
is required for an owner-initiated downgrade. `(R-ART-15)`

Compliant nodes honor the downgrade going forward. Previously served
content that has propagated to other nodes is not recalled.

---

## 8. Policy conflict detection

Because `raw_igc_hash` is deterministic and publicly derivable from the raw
IGC bytes, any compliant node can detect privacy downgrade conflicts.

### 8.1 Conflict: privacy downgrade attempt

If a compliant node receives a new upload for a `raw_igc_hash` that already
has an existing `publication-mode-record` with a more restrictive mode:

- The node MUST detect the conflict. `(R-ART-08)`
- The node MUST NOT publish a `public` mode record for a hash that is
  already marked `protected` or `private`. `(R-ART-09)`
- The node MUST NOT silently accept the downgrade. `(R-ART-10)`
- The node does not need to know the accepted owner before detecting the
  conflict; the existing mode record is sufficient.

### 8.2 Retroactive mode change

When an accepted owner claim exists for a `raw_igc_hash` that was published
as `public` by a third party before the pilot joined the network, the
accepted owner MAY sign a `publication-mode-record` to change the mode
retroactively. Compliant nodes MUST honor this change going forward. `(R-ART-11)`

---

## 9. Artifact relations summary

| Relation | From | To | When present |
|----------|------|----|-------------|
| `sanitized_as` | `raw_igc_hash` | `protected_hash` | `protected` mode only |

The `sanitized_as` relation is a directed link from the identity anchor to
the sanitized artifact identity. It is published with the protected flight
record. No `sanitized_as` relation exists for `public` or `private` mode
artifacts.
