# igc-net v0.2 — Pilot Profile and Metadata

**Status:** Normative  
**Depends on:** `10-core.md`

---

## 1. Authority model

v0.2 defines three metadata record types with distinct authority levels:

```
pilot-profile   = authoritative identity-level pilot data
flight-metadata = authoritative accepted-owner-signed flight data
igc-metadata    = private source-observed provenance (never authoritative)
```

`igc-metadata` records what was observed in the uploaded IGC file. They are
never authoritative over `pilot-profile` or `flight-metadata`. The
distinction between IGC-derived provenance, pilot-authored authority, and
portal-local display state is fundamental to this model.

All three record types are stored and served as plaintext JSON. Records
whose `visibility` is `"private"` (or, for `igc-metadata`, whose type is
always private) are access-gated: serving nodes deliver them only to
requesters who present a valid fetch request signed by the pilot's
currently authorized `private_access_keypair`. Confidentiality at rest on
the serving node is a compliance and legal obligation; no protocol-level
content encryption (AEAD) is applied.

---

## 2. Pilot profile

`pilot-profile` is the authoritative identity-level profile for a pilot. It
is attached to `pilot_id`.

### 2.1 Shape

```json
{
  "schema": "igc-net/pilot-profile",
  "schema_version": 1,
  "record_id": "<blake3-of-canonical-record-without-signature>",
  "pilot_id": "igcnet:id:<root-public-key-hex>",
  "full_name": "Alice Example",
  "country": "NO",
  "visibility": "private",
  "created_at": "YYYY-MM-DDTHH:MM:SSZ",
  "supersedes": null,
  "signature": "<ed25519-signature-hex>"
}
```

### 2.2 Rules

- Signed by the pilot's root identity key (`pilot_id`).
- `visibility` controls who may read this record. Values: `"public"` or
  `"private"`. Default: `"private"`.
- `country` uses ISO 3166-1 alpha-2.
- At most one active `pilot-profile` record per `pilot_id` at any time. `(R-META-01)`
- A later pilot-signed record supersedes the earlier active record. The
  `supersedes` field MUST reference the `record_id` of the record being
  replaced. Set to `null` for the initial record.
- Records with `visibility: "private"` are served only to requesters who
  present a valid fetch request signed by the pilot's currently authorized
  `private_access_keypair` (see `60-keys-and-access.md §4`). Bytes are
  served as plaintext over iroh; nodes holding the record are bound by
  confidentiality obligations.
- Records with `visibility: "public"` are served in plaintext without
  authorization.
- `pilot-profile` is NOT a flight-scoped record. It is not created per
  flight. Flight-specific wing and equipment data belongs in
  `flight-metadata`.

---

## 3. Flight metadata

`flight-metadata` is accepted-owner-signed metadata about wings and similar
flight-level data. It uses two scopes: `"default"` and `"flight"`.

### 3.1 Default scope

The `default` scope applies to all flights for this pilot unless overridden
by a `flight`-scope record.

```json
{
  "schema": "igc-net/flight-metadata",
  "schema_version": 1,
  "record_id": "<blake3-of-canonical-record-without-signature>",
  "pilot_id": "igcnet:id:<root-public-key-hex>",
  "scope": "default",
  "raw_igc_hashes": [],
  "wing_manufacturer": "Ozone",
  "wing_model": "Alpina 4",
  "wing_class_en": "CCC",
  "visibility": "private",
  "created_at": "YYYY-MM-DDTHH:MM:SSZ",
  "supersedes": null,
  "signature": "<ed25519-signature-hex>"
}
```

`raw_igc_hashes` MUST be empty for `scope: "default"`.

### 3.2 Flight scope

The `flight` scope applies to a specific set of `raw_igc_hash` values,
representing one logical flight unit (e.g., multiple IGC files from the
same flight recorder session).

```json
{
  "schema": "igc-net/flight-metadata",
  "schema_version": 1,
  "record_id": "<blake3-of-canonical-record-without-signature>",
  "pilot_id": "igcnet:id:<root-public-key-hex>",
  "scope": "flight",
  "raw_igc_hashes": [
    "<blake3-hex-a>",
    "<blake3-hex-b>"
  ],
  "wing_manufacturer": "Advance",
  "wing_model": "Sigma DLS",
  "wing_class_en": "C",
  "visibility": "private",
  "created_at": "YYYY-MM-DDTHH:MM:SSZ",
  "supersedes": null,
  "signature": "<ed25519-signature-hex>"
}
```

`raw_igc_hashes` MUST be non-empty for `scope: "flight"`.

### 3.3 Rules for all flight-metadata records

- Signed by the pilot's root identity key (`pilot_id`). Governance records
  (mode records, deletion requests) require accepted-owner status. Metadata
  records (pilot-profile, flight-metadata) require only a valid pilot
  signature.
- A compliant node MUST reject a `flight-metadata` record where `pilot_id`
  does not match the accepted owner for all listed `raw_igc_hash` values.
- `visibility` default: `"private"`. Values: `"public"` or `"private"`.
- Every `raw_igc_hash` listed in `raw_igc_hashes` MUST already be owned by
  the same accepted owner.
- At most one active `scope: "default"` record per `pilot_id` at any time. `(R-META-02)`
- For any given `raw_igc_hash`, at most one active `scope: "flight"` record
  may apply at any time. `(R-META-03)`
- Supersession is whole-record: a new record replaces the entire previous
  active record. No field-level merge. `(R-META-09)`
- Records with `visibility: "private"` are served only to requesters who
  present a valid fetch request signed by the pilot's currently authorized
  `private_access_keypair`. Bytes are served as plaintext; nodes holding
  the record are bound by confidentiality obligations.

### 3.4 Atomic supersession for flight-scope records

When a `flight`-scope record is superseded: `(R-META-12)`

- The old record's coverage ends atomically for ALL `raw_igc_hash` values
  it listed.
- `raw_igc_hash` values not covered by the new record fall back to the
  `default`-scope record.
- There is no partial or per-hash supersession. Either the entire old
  record is superseded or none of it is.

---

## 4. IGC metadata

`igc-metadata` records source-observed provenance extracted from the raw
uploaded IGC file. It is always private.

### 4.1 Shape

```json
{
  "schema": "igc-net/igc-metadata",
  "schema_version": 1,
  "record_id": "<blake3-of-canonical-record-without-signature>",
  "raw_igc_hash": "<blake3-hex>",
  "source_service": "xcontest",
  "observed_full_name": "ALICE EXAMPLE",
  "observed_glider_type": "Ozone Alpina 4",
  "observed_glider_id": "D-1234",
  "observed_comment": "",
  "node_id": "<node-ed25519-public-key-hex>",
  "created_at": "YYYY-MM-DDTHH:MM:SSZ",
  "signature": "<ed25519-signature-hex>"
}
```

`node_id` identifies the uploading portal that created this record.
`signature` is the Ed25519 signature over the RFC 8785 canonical JSON of
all fields except `signature`, made with the uploading portal's `node_id`
key. The signature binds the record's integrity and origin; a recipient
MUST verify it before trusting the record's contents.

### 4.2 Rules

- Always private. There is no `visibility` field; `igc-metadata` MUST NOT
  be served to any requester that does not present a valid fetch request
  signed by the pilot's currently authorized `private_access_keypair`. `(R-META-04)`
- Bytes are served as plaintext over iroh's encrypted transport; no
  protocol-level content encryption is applied. Nodes holding
  `igc-metadata` are bound by confidentiality obligations.
- Provenance only. `igc-metadata` is NOT authoritative over `pilot-profile`
  or `flight-metadata` for any field.
- MUST be signed by the uploading portal's `node_id` key.
- Held by the uploading node (portal or pilot's self-hosted node).
- The pilot operating a private-access node is always an authorized
  requester; the holding node MUST serve `igc-metadata` to the pilot on
  demand when the request is signed by the pilot's currently authorized
  `private_access_keypair`.
- If the pilot operates a self-hosted node at upload time, the uploading
  portal MUST replicate `igc-metadata` to that node at upload time. `(R-META-10)`

---

## 5. Precedence rules

When displaying metadata, portals MUST apply these precedence rules:

**For wing and glider fields:**

1. Latest active `flight-metadata(scope="flight")` covering the
   `raw_igc_hash`.
2. Otherwise: latest active `flight-metadata(scope="default")` for the
   pilot.
3. Otherwise: `igc-metadata` as provenance/fallback display source (MUST be
   clearly labelled as observed provenance, not authoritative data).

**For pilot identity fields:**

1. Latest active `pilot-profile`.
2. Otherwise: `igc-metadata` as provenance/fallback display source (MUST be
   clearly labelled as observed provenance, not authoritative data).

`igc-metadata` MUST NOT silently replace authoritative pilot metadata. `(R-META-06)`
If a portal displays values from `igc-metadata`, it MUST indicate to the
user that the source is observed provenance, not the pilot's authoritative
record. `(R-META-07)`

---

## 6. Visibility defaults `(R-META-05)`

| Record type | Default visibility | Visibility field? |
|-------------|-------------------|------------------|
| `pilot-profile` | `"private"` | Yes |
| `flight-metadata` | `"private"` | Yes |
| `igc-metadata` | Always private | No |

Semantics of `visibility: "private"`: the record is served only to
requesters who present a valid fetch request signed by the pilot's
currently authorized `private_access_keypair`. The record itself is stored
and transmitted as plaintext.

---

## 7. Portal-local display rules

### 7.1 Identity display rule `(R-META-11)`

The sanitized public artifact for a `protected` flight MUST NOT carry
pilot identity.

A portal MAY display pilot identity alongside a protected flight ONLY if:

- it operates as a private-access node (Category 2) for that pilot and has
  obtained a private `pilot-profile` record under the pilot's
  `private_access_keypair`, OR
- it is displaying a `pilot-profile` record the pilot has published with
  `visibility: "public"`, OR
- the pilot is the locally authenticated account user on a full-custody
  portal.

The source of any displayed pilot identity MUST NOT be the sanitized
artifact itself. It MUST be an authorized separate record.

### 7.2 Permission-loss obligation

If a portal loses authority to read private authoritative records — for
example, the pilot has deleted the `private_access_keypair` from this
portal, the pilot has rotated the keypair and not re-granted this portal,
or the pilot has demoted this portal to an identity-linked node — the
portal MUST immediately stop using cached copies of those private records
for display or further processing. `(R-META-08)`

This applies to private `pilot-profile` records, private `flight-metadata`
records, and `igc-metadata` records.

---

## 8. Storage model for protected and private uploads

For `protected` uploads, a private-access portal holds:

1. The sanitized public artifact (plaintext `.igc`).
2. The raw IGC companion (plaintext `.igc`), access-gated by fetch-request
   signature.
3. The private `igc-metadata` record (plaintext JSON, access-gated).
4. The pilot's `pilot-profile` and `flight-metadata` as separate signed
   records (plaintext JSON; access-gated when `visibility: "private"`).

For `private` uploads, a private-access portal holds the raw IGC bytes
(plaintext) plus the same metadata records as above. All content is
access-gated by fetch-request signature.

The sanitized public artifact contains public-safe flight data only. The
raw companion and the private-mode raw IGC preserve exact provenance and
future reparsing capability. Confidentiality of stored plaintext is a
compliance and legal obligation on the portal operator.
