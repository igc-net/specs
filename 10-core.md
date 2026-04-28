# igc-net — Core Identifiers and Cryptographic Primitives

**Status:** Normative  
**Depends on:** `00-overview.md`

---

## 1. Cryptographic primitives

All algorithm choices are single mandatory selections. No alternatives
are permitted. Interoperability requires identical algorithm use across all nodes.

### 1.1 Hash function

**BLAKE3** is the exclusive hash function for all content hashes and record IDs.

No other hash function (SHA-256, SHA-3, etc.) is permitted in any context where
this specification requires a hash. `(R-CORE-01)`

### 1.2 Signing algorithm

**Ed25519** is the exclusive signing algorithm. `(R-CORE-02)`

- Keypairs are generated from a 32-byte seed using standard Ed25519 key
  generation.
- Signing input is the canonical JSON bytes of the record's non-signature fields
  (see §5).
- All signatures are made directly from the role's identity key.
- There are no delegated signing keys in this specification.
- There are no rotating signing keys in this specification.
- A signature from any key other than the signer's current identity key MUST be
  rejected. `(R-CORE-03)`

### 1.3 Transport encryption

igc-net does NOT apply protocol-level authenticated encryption to
content. Confidentiality in flight is provided by the underlying iroh
peer-to-peer transport, which encrypts all traffic end-to-end between
authenticated node endpoints. Confidentiality at rest for non-public
content held at a private-access node is a compliance and legal obligation
on the node operator (see `60-keys-and-access.md §1`, §2.2, §8).

The requirement on implementations: the transport used by a compliant
node MUST provide end-to-end encryption and peer authentication between
`node_id` endpoints. `(R-CORE-04)` An implementation that uses a transport
without end-to-end encryption is NOT conformant.

No AEAD envelope is applied to flight artifacts, metadata advertisements, or
fetch-request bodies. Signatures over RFC 8785 canonical JSON provide
integrity and origin authentication where needed (see §5).

### 1.4 Hex encoding

All binary values (hashes, public keys) that appear as strings in records MUST
be encoded as **lowercase hexadecimal ASCII** with no `0x` prefix and no
separators. `(R-CORE-07)`

### 1.5 Timestamp format

Timestamps use the following RFC 3339 subset:

```
YYYY-MM-DDTHH:MM:SSZ
```

- No fractional seconds.
- Always UTC.
- Always `Z` suffix; no numeric offset.

The `created_at` field in all records uses this format. `created_at` is
**informational** and serves as an **expiry baseline** for challenge records
only. It is NOT a general record-ordering authority and MUST NOT be used as a
conflict-resolution or supersession mechanism.

---

## 2. Core object classes

Three object classes structure the protocol. They are used consistently across
all normative documents.

### 2.1 Identity anchor

`raw_igc_hash` is the universal flight identity anchor.

```
raw_igc_hash = BLAKE3(raw_igc_bytes)
```

- Input: the exact raw bytes of the IGC file as received, with no modification.
- Output: 32 bytes encoded as 64-character lowercase hex.
- `raw_igc_hash` is independent of upload time, uploader identity, portal
  identity, and any metadata.
- `raw_igc_hash` is immutable once computed. `(R-CORE-08)`
- All subsequent flight-scoped records (claims, artifacts, flight-scoped
  metadata advertisements, governance records) MUST reference `raw_igc_hash` as
  the flight identity. `(R-CORE-09)`

Two portals independently receiving the same IGC file bytes MUST compute the
same `raw_igc_hash`. `(R-CORE-10)`

### 2.2 Artifact identity

Artifact identity identifies a specific artifact derived from the flight.
It is distinct from the identity anchor and changes if the artifact
changes.

```
protected_hash = BLAKE3(sanitized_igc_bytes)
```

- `protected_hash`: identifies the sanitized public artifact for a
  `protected` flight. The sanitization algorithm that produces
  `sanitized_igc_bytes` is defined in `20-artifacts.md`. `(R-CORE-17)`
- `protected_hash` is 32 bytes encoded as 64-character lowercase hex.

For `public` and `private` modes, the artifact is the raw IGC bytes
themselves and its identity is `raw_igc_hash`. No separate ciphertext-artifact
identity is defined: no AEAD envelope is applied at the protocol layer.

### 2.3 Locator

A **ticket** is a locator. It encodes a serving node endpoint and a blob
identifier, indicating where an artifact may be fetched.

**Ticket possession is not authorization.** The serving node decides at delivery
time whether the requester may receive the artifact, based on governance state
and key-possession proof.

Multiple tickets for the same `raw_igc_hash` represent different serving peers
holding the same content. They are stored for redundancy, not treated as
distinct flights.

---

## 3. Pilot identity

```
pilot_id = "igcnet:id:" + hex(root_public_key_bytes)
```

- `root_public_key_bytes` is the 32-byte compressed Ed25519 public key of the
  pilot's root identity keypair.
- The hex encoding uses lowercase ASCII.
- `pilot_id` is stable for the lifetime of the root keypair.
- A different root keypair produces a different `pilot_id`.

`pilot_id` MUST be derived exclusively from the root public key bytes. No other
input (content, timestamp, upload context) may contribute to `pilot_id`. `(R-CORE-15)`

---

## 4. Key-role separation

igc-net defines three identity roles, each requiring a distinct Ed25519 keypair:

| Role | Identifier | Keypair purpose |
|------|-----------|----------------|
| Pilot | `pilot_id` | Signs pilot-authored governance records |
| Serving node | `node_id` | Signs announcements and fetch tickets |
| Trusted resolver | `resolver_id` | Signs approvals, challenges, resolutions |

**MUST rules:**

- `pilot_id`, `node_id`, and `resolver_id` MUST be distinct Ed25519 keypairs. `(R-CORE-11)`
- A machine operating multiple roles simultaneously MUST generate and store
  separate keypairs for each role. `(R-CORE-12)`
- No two roles MAY share a keypair. `(R-CORE-13)`
- The pilot's `private_access_keypair` (defined in `60-keys-and-access.md`)
  is an authorization credential, not an identity. It MUST be a distinct
  Ed25519 keypair from `pilot_id`, `node_id`, and `resolver_id`.
- The pilot's `pilot_auth_did` (defined in `65-pilot-auth-did.md`) is a
  distinct authentication / VC-issuer credential. It MUST be a distinct
  Ed25519 keypair from `pilot_id`, `private_access_keypair`, `node_id`, and
  `resolver_id`.

---

## 5. Canonical form and record IDs

### 5.1 Canonical JSON

All records that are signed or hashed for a record ID MUST be serialised using
**RFC 8785 (JSON Canonicalization Scheme)**: `(R-CORE-14)`

- Keys are ordered lexicographically by Unicode code point.
- No unnecessary whitespace or newlines.
- Unicode characters are normalised per RFC 8785.
- The `signature` field is **excluded** from the canonical form before signing
  and before computing the record ID.

### 5.2 Record ID

```
record_id = BLAKE3(canonical_json(record_without_signature))
```
`(R-CORE-16)`

- All fields except `signature` are included.
- The `record_id` is stable: identical records (same fields, same values) always
  produce the same `record_id`.
- Any change to any non-signature field produces a different `record_id`.

### 5.3 Signing

Records carry a `signature` field but do NOT carry a separate `signing_key`
field. The verification key is always derivable from the record's schema-specific
identity field:

| Schema | Identity field | Signing key |
|--------|---------------|-------------|
| `igc-net/claim`, `igc-net/publication-mode-record`, `igc-net/deletion-request`, `igc-net/private-access-rotation-record`, `igc-net/pilot-auth-did-record` | `pilot_id` | Strip `igcnet:id:` prefix; decode hex → Ed25519 public key |
| `igc-net/claim-approval`, `igc-net/claim-challenge`, `igc-net/claim-resolution` | `resolver_id` | Decode hex → Ed25519 public key |
| `igc-net/roster-update` | `signer_id` | Decode hex → Ed25519 public key; verifier MUST confirm `signer_id` is the project root key or an authorized roster member (see `50-governance.md §2.4`) |
| `igc-net/resolver-profile` | `resolver_id` | Decode hex → Ed25519 public key |
| `igc-net/announcement` | `node_id` | Decode hex → Ed25519 public key |
| `igc-net/metadata-advertisement` | `node_id` | Decode hex → Ed25519 public key |
| `igc-net/fetch-request` | `requester_key` | Decode hex → Ed25519 public key (MUST match current `private_access_public_key` for the pilot) |

To sign a record:

1. Remove the `signature` field from the record object.
2. Serialise the remaining fields as RFC 8785 canonical JSON.
3. Sign the resulting bytes with the signer's Ed25519 identity key.
4. Encode the signature as 64-byte lowercase hex and set it in the `signature`
   field.

To verify a record:

1. Determine the expected signing key from the record's `schema` field using
   the table above.
2. Remove `signature` from the record object.
3. Serialise as RFC 8785 canonical JSON.
4. Verify the Ed25519 signature over those bytes using the determined public key.
5. Reject the record if signature verification fails or if the schema-identified
   signer is not authorised to produce this record type.

There are no delegated or rotating governance-signing keys. All native
record signatures MUST be made directly from the signer's current identity key.
`pilot_auth_did` is a separate authentication / VC-issuer credential, not a
native governance-record signing key.

---

## 6. Protocol extensions

This specification distinguishes between:

- **native protocol records** that participate in `igc-net` signing and
  `record_id` rules, and
- **application-layer credential objects** such as VC-JWT payloads that do not.

Extension naming rule:

- A native protocol extension that is not part of the numbered core document set
  MUST use an `x-` prefixed schema or field namespace to avoid collision with
  future standardized core names.

Extension signing rule:

- If an extension is defined as a native signed JSON record, it MUST follow the
  same RFC 8785 canonical JSON and BLAKE3 `record_id` rules as other native
  signed records.

Boundary rule:

- JOSE / VC-JWT objects are not native `igc-net` records and MUST NOT be forced
  into the native `record_id` or canonical-JSON signature model. They follow
  their own JOSE / VC verification rules.
