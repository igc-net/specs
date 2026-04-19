# igc-net — Pilot Profile and Metadata

**Status:** Normative  
**Depends on:** `10-core.md`

---

## 1. Authority model

This specification defines two native metadata record types and one VC-based profile authority:

```text
PilotProfileCredential = authoritative identity-level pilot profile data
flight-metadata        = authoritative accepted-owner-signed flight data
igc-metadata           = private source-observed provenance (never authoritative)
```

`igc-metadata` records what was observed in the uploaded IGC file. It is never
authoritative over `PilotProfileCredential` or `flight-metadata`. The
distinction between source-observed provenance, pilot-authored authoritative
flight metadata, and VC-based profile presentation is fundamental to the
identity model.

`flight-metadata` and `igc-metadata` are native `igc-net` JSON records. They
are stored and served as plaintext JSON over the protocol.

`PilotProfileCredential` is different:

- it is a self-issued VC-JWT, not a native governance record
- it is presented at the application layer by the pilot's wallet or by a
  pilot-authorized sync service
- it is not broadcast on the governance topic or the announce topic
- it is not fetched via `fetch-request`

No protocol-level content encryption is applied to native metadata; see
`10-core.md §1.3`.

---

## 2. Pilot profile authority

### 2.1 Authoritative object

The authoritative identity-level pilot profile object is
`PilotProfileCredential`, a self-issued VC-JWT signed by the pilot's currently
active `pilot_auth_did`.

The native `pilot-profile` record from prior versions is removed as a live
authority. An implementation MUST NOT treat a native `pilot-profile` JSON
record as the authoritative source of profile fields. `(R-META-01)`

`PilotProfileCredential` is authoritative for pilot profile display fields only.
It is not authoritative for:

- ownership or accepted-owner status
- publication mode
- fetch authorization
- governance topic state
- flight-scoped metadata such as wing or equipment data

### 2.2 Required VC-JWT shape

The minimal `PilotProfileCredential` profile is:

- JOSE header:
  - `alg` MUST be `"EdDSA"`
  - `typ` MUST be `"vc+jwt"`
  - `kid` MUST identify the verification method of the issuer DID

- JWT claims:
  - `iss` MUST be the pilot's currently active `pilot_auth_did`
  - `sub` MUST be the pilot's `pilot_id`
  - `iat` MUST be present
  - `nbf` MUST be present
  - `jti` MUST be present
  - `vc` MUST be present

- `vc` object:
  - `@context` MUST include `"https://www.w3.org/2018/credentials/v1"`
  - `type` MUST include `"VerifiableCredential"`
  - `type` MUST include `"PilotProfileCredential"`
  - `credentialSubject.id` MUST equal `sub`
  - `credentialSubject.pilot_auth_did` MUST equal `iss`
  - `credentialSubject.country`, if present, MUST use ISO 3166-1 alpha-2

Profile fields such as display name and country live inside
`vc.credentialSubject`. Flight-level metadata does not belong in this VC.

### 2.3 Verification rules

A portal or relying party MUST perform all of the following before treating a
presented `PilotProfileCredential` as authoritative: `(R-META-02)`

1. Decode the compact JWT and reject malformed input.
2. Verify `alg = "EdDSA"` and `typ = "vc+jwt"`.
3. Resolve `iss` as a DID of an accepted pilot authentication method. The
   required portable pilot method is `did:key`; other pilot-held methods are
   not authoritative unless explicitly admitted by `65-pilot-auth-did.md`.
4. Verify the JOSE signature using the public key resolved from `iss`.
5. Read `sub` and confirm it is a syntactically valid `pilot_id`.
6. Fetch or consult the authoritative `pilot-auth-did-record` chain for `sub`
   and confirm the currently active `pilot_auth_did` equals `iss`.
7. Confirm `credentialSubject.id = sub`.
8. Confirm `credentialSubject.pilot_auth_did = iss`.

If Step 6 fails because no authoritative `pilot-auth-did-record` is available
for `sub`, the VC MUST NOT be treated as authoritative. `(R-META-03)`

If the VC is cryptographically valid but `iss` no longer matches the currently
active `pilot_auth_did` for `sub`, the VC is stale and MUST NOT be treated as
authoritative. `(R-META-04)`

If a portal displays fields from a VC that failed any of the checks above for
debugging or audit purposes, it MUST label those fields as non-authoritative.

### 2.4 Distribution and caching

`PilotProfileCredential` is distributed outside the native transport plane:

- the pilot's wallet MAY present it directly to a portal during authentication
  or profile-sharing flows
- a pilot-authorized sync endpoint MAY serve it to portals outside the native
  `igc-net` fetch-request mechanism

`PilotProfileCredential` MUST NOT be embedded in:

- governance records
- announce-topic records
- `pilot-auth-did-record`
- `fetch-request`

Portals MAY cache the latest verified `PilotProfileCredential` they have
received for a given `pilot_id`.

A cached VC becomes stale if either of the following occurs:

- the portal receives a newer verified profile VC for the same `pilot_id`
- the portal processes a `pilot-auth-did-record` that changes the active
  `pilot_auth_did` for that `pilot_id`

A portal that detects either condition MUST stop treating the older cached VC as
authoritative until a fresh VC is verified. `(R-META-05)`

There is no `supersedes` chain for profile VCs. Operationally, the latest
verified non-stale VC replaces the older one.

---

## 3. Flight metadata

`flight-metadata` is accepted-owner-signed metadata about wings and similar
flight-level data. It uses two scopes: `"default"` and `"flight"`.

### 3.1 Default scope

The `default` scope applies to all flights for this pilot unless overridden by a
`flight`-scope record.

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
representing one logical flight unit (for example, multiple IGC files from the
same recorder session).

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

### 3.3 Rules for all `flight-metadata` records

- Signed by the pilot's root identity key (`pilot_id`).
- A compliant node MUST reject a `flight-metadata` record where `pilot_id` does
  not match the accepted owner for all listed `raw_igc_hash` values.
- `visibility` default: `"private"`. Values: `"public"` or `"private"`.
- Every `raw_igc_hash` listed in `raw_igc_hashes` MUST already be owned by the
  same accepted owner.
- At most one active `scope: "default"` record per `pilot_id` at any time.
  `(R-META-06)`
- For any given `raw_igc_hash`, at most one active `scope: "flight"` record may
  apply at any time. `(R-META-07)`
- Supersession is whole-record: a new record replaces the entire previous active
  record. No field-level merge. `(R-META-08)`
- Records with `visibility: "private"` are served only to requesters who
  present a valid fetch request signed by the pilot's currently authorized
  `private_access_keypair`. Bytes are served as plaintext; nodes holding the
  record are bound by confidentiality obligations.

### 3.4 Atomic supersession for flight-scope records

When a `flight`-scope record is superseded: `(R-META-09)`

- The old record's coverage ends atomically for all `raw_igc_hash` values it
  listed.
- `raw_igc_hash` values not covered by the new record fall back to the
  `default`-scope record.
- There is no partial or per-hash supersession. Either the entire old record is
  superseded or none of it is.

---

## 4. IGC metadata

`igc-metadata` records source-observed provenance extracted from the raw uploaded
IGC file. It is always private.

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
`signature` is the Ed25519 signature over the RFC 8785 canonical JSON of all
fields except `signature`, made with the uploading portal's `node_id` key. The
signature binds the record's integrity and origin; a recipient MUST verify it
before trusting the record's contents.

### 4.2 Rules

- Always private. There is no `visibility` field; `igc-metadata` MUST NOT be
  served to any requester that does not present a valid fetch request signed by
  the pilot's currently authorized `private_access_keypair`. `(R-META-10)`
- Bytes are served as plaintext over iroh's encrypted transport; no
  protocol-level content encryption is applied. Nodes holding `igc-metadata` are
  bound by confidentiality obligations.
- Provenance only. `igc-metadata` is NOT authoritative over
  `PilotProfileCredential` or `flight-metadata` for any field.
- MUST be signed by the uploading portal's `node_id` key.
- Held by the uploading node (portal or pilot's self-hosted node).
- The pilot operating a private-access node is always an authorized requester;
  the holding node MUST serve `igc-metadata` to the pilot on demand when the
  request is signed by the pilot's currently authorized `private_access_keypair`.
- If the pilot operates a self-hosted node at upload time, the uploading portal
  MUST replicate `igc-metadata` to that node at upload time. `(R-META-11)`

---

## 5. Precedence rules

When displaying metadata, portals MUST apply these precedence rules.

**For wing and glider fields:**

1. Latest active `flight-metadata(scope="flight")` covering the `raw_igc_hash`.
2. Otherwise: latest active `flight-metadata(scope="default")` for the pilot.
3. Otherwise: `igc-metadata` as provenance/fallback display source, clearly
   labelled as observed provenance, not authoritative data.

**For pilot identity fields:**

1. Latest verified non-stale `PilotProfileCredential` for that `pilot_id`.
2. Otherwise: `igc-metadata` as provenance/fallback display source, clearly
   labelled as observed provenance, not authoritative data.

`igc-metadata` MUST NOT silently replace authoritative pilot metadata.
`(R-META-12)`

If a portal displays values from `igc-metadata`, it MUST indicate to the user
that the source is observed provenance, not the pilot's authoritative record.
`(R-META-13)`

---

## 6. Visibility defaults for native metadata

`visibility` applies only to native metadata records.

| Object | Default visibility | Visibility field? |
|--------|--------------------|------------------|
| `PilotProfileCredential` | Not a native visibility-controlled record | No |
| `flight-metadata` | `"private"` | Yes |
| `igc-metadata` | Always private | No |

Semantics of `visibility: "private"`: the record is served only to requesters
who present a valid fetch request signed by the pilot's currently authorized
`private_access_keypair`. The record itself is stored and transmitted as
plaintext. `(R-META-14)`

`PilotProfileCredential` privacy is determined by pilot presentation and
portal-side policy outside the native fetch-request mechanism. It is not a
governance-controlled protocol object.

---

## 7. Portal-local display rules

### 7.1 Identity display rule `(R-META-15)`

The sanitized public artifact for a `protected` flight MUST NOT carry pilot
identity.

A portal MAY display pilot identity alongside a protected flight only if:

- the pilot is the locally authenticated account user on that portal, or
- the portal has a latest verified non-stale `PilotProfileCredential` for that
  pilot

The source of any displayed pilot identity MUST NOT be the sanitized artifact
itself. It MUST be a separate authoritative or clearly-labelled provenance
source.

### 7.2 Permission-loss obligation

If a portal loses authority to read private native metadata records, it MUST
immediately stop using cached copies of those private native records for display
or further processing. `(R-META-16)`

This applies to:

- private `flight-metadata` records
- `igc-metadata`

`PilotProfileCredential` does not follow `private_access_keypair` authorization.
Its validity instead follows the verification and staleness rules in §2.

---

## 8. Storage model for protected and private uploads

For `protected` uploads, a private-access portal holds:

1. The sanitized public artifact (plaintext `.igc`).
2. The raw IGC companion (plaintext `.igc`), access-gated by fetch-request
   signature.
3. The private `igc-metadata` record (plaintext JSON, access-gated).
4. The pilot's `flight-metadata` records as separate signed native JSON records
   (access-gated when `visibility: "private"`).
5. Optionally, the latest verified `PilotProfileCredential` cache for that pilot
   if it has been separately presented to the portal by the wallet or by a
   pilot-authorized sync channel.

For `private` uploads, a private-access portal holds the raw IGC bytes
(plaintext) plus the same metadata objects as above. All native non-public
content is access-gated by fetch-request signature.

The sanitized public artifact contains public-safe flight data only. The raw
companion and the private-mode raw IGC preserve exact provenance and future
reparsing capability. Confidentiality of stored plaintext native content is a
compliance and legal obligation on the portal operator.
