# igc-net — Public Metadata Advertisements

**Status:** Normative  
**Depends on:** `10-core.md`, `20-artifacts.md`, `30-transport.md`

---

## 1. Scope

igc-net does not standardize native pilot metadata, native flight metadata,
IGC-derived provenance records, metadata merge rules, or analytics semantics in
the pre-v0.5 baseline.

The only metadata-plane object in this baseline is a lightweight public
**metadata advertisement**. It lets a portal announce that it has portal-defined
metadata or derived resources associated with one or more public igc-net
identifiers. The network provides discovery, attribution, and namespace
recognition only.

Metadata advertisements are not:

- ownership claims
- publication-mode records
- fetch authorization
- pilot profile authority
- standardized thermal, wind, scoring, replay, or analytics schemas
- evidence that a referenced resource is public-fetchable

`PilotProfileCredential` remains an application-layer credential presented by a
pilot wallet or portal-local account flow. It is not a native metadata
advertisement and is not fetched through the igc-net artifact fetch mechanism.
Pilot authentication DID binding is defined in `65-pilot-auth-did.md`.

---

## 2. Public-only rule

Metadata advertisements are always public. `(R-META-01)`

A portal MUST NOT put confidential, private, or access-controlled data directly
inside a metadata advertisement. `(R-META-02)`

Advertisements MAY refer to other resources. Those resources MAY be public,
protected, private, or portal-local, but access to a referenced resource is
governed by that resource's own access policy, not by the advertisement.
`(R-META-03)`

Public identifiers are not private in igc-net. In particular, `raw_igc_hash`,
`protected_hash`, portal namespaces, and public resource identifiers MAY appear
in public advertisements even when the referenced artifact or resource is
protected or private. `(R-META-04)`

Derived metadata can still reveal sensitive information. The publishing portal
is responsible for deciding whether a derived value is safe to advertise
publicly. igc-net does not guarantee that a metadata advertisement is
non-sensitive merely because the advertisement is syntactically valid.
`(R-META-05)`

---

## 3. Advertisement record

Metadata advertisements are native signed JSON records broadcast on the
data-plane announce topic. They use the same canonical JSON, `record_id`, and
signature rules as other native signed records.

### 3.1 Shape

```json
{
  "schema": "igc-net/metadata-advertisement",
  "schema_version": 1,
  "record_id": "<blake3-of-canonical-record-without-signature>",
  "portal_namespace": "org.example.portal",
  "advertisement_type": "org.example.portal.thermals",
  "raw_igc_hashes": ["<blake3-hex>", "..."],
  "resource_refs": [
    {
      "uri": "https://portal.example.org/flights/<id>/thermals.json",
      "access": "public|protected|private|portal-local",
      "media_type": "application/json"
    }
  ],
  "payload": {},
  "node_id": "<node-ed25519-public-key-hex>",
  "created_at": "YYYY-MM-DDTHH:MM:SSZ",
  "signature": "<ed25519-signature-hex>"
}
```

### 3.2 Required fields

- `schema` MUST be exactly `"igc-net/metadata-advertisement"`.
- `schema_version` MUST be `1` for this version.
- `record_id = BLAKE3(canonical_json(record_without_signature))`.
- `portal_namespace` identifies the portal-defined namespace. It MUST be a
  non-empty lowercase ASCII namespace string controlled by the publishing portal.
  Reverse-DNS naming is RECOMMENDED, for example `org.xcontest` or
  `net.cloudstreet`. `(R-META-06)`
- `advertisement_type` identifies the portal-defined advertisement kind. It
  MUST be scoped under `portal_namespace` or use an `x-` prefixed experimental
  namespace. `(R-META-07)`
- `raw_igc_hashes` MAY be empty. If present, every value MUST be a lowercase
  BLAKE3 hex hash.
- `resource_refs` MAY be empty. If present, each entry is a locator or pointer,
  not authorization.
- `payload` MAY be any JSON object. Its structure is portal-defined.
- `node_id` identifies the publishing portal/node.
- `signature` MUST verify against `node_id`. `(R-META-08)`

### 3.3 Validation and handling

A node receiving a metadata advertisement MUST validate `schema`,
`schema_version`, `record_id`, `node_id`, `signature`, namespace fields, and hash
formats before indexing it. Invalid advertisements MUST be dropped.
`(R-META-09)`

A node that does not recognize `portal_namespace` or `advertisement_type` MUST
ignore the advertisement payload and MAY retain only minimal index information
needed to show that the portal advertised metadata for the referenced hashes.
Unknown namespaces MUST NOT cause fetch, governance, or publication-mode
behavior to change. `(R-META-10)`

Recognizing a namespace means only that the receiver knows how to interpret the
portal-defined payload. It does not make the payload authoritative over igc-net
identity, artifact hashes, publication modes, governance state, or access
control. `(R-META-11)`

Metadata advertisements MUST NOT be broadcast on the governance topic.
`(R-META-12)`

---

## 4. Resource references

`resource_refs` are pointers. They do not grant access and they do not imply
that igc-net can fetch the referenced resource. `(R-META-13)`

The `access` field is advisory and describes the publishing portal's intended
access policy for the referenced resource:

| Value | Meaning |
|-------|---------|
| `public` | The portal expects the resource to be publicly reachable. |
| `protected` | The portal expects some non-igc-net or future igc-net access rule. |
| `private` | The portal expects explicit pilot/portal authorization. |
| `portal-local` | The resource is meaningful only within the publishing portal. |

For pre-v0.5, igc-net defines no standard fetch path for advertisement
resources. A portal that follows a `resource_ref` uses portal-specific policy
and transport outside the core artifact fetch rules. `(R-META-14)`

---

## 5. Examples

The following are examples only. They are not standardized payload schemas and
they are not conformance requirements.

### 5.1 Thermal annotations

```json
{
  "schema": "igc-net/metadata-advertisement",
  "schema_version": 1,
  "record_id": "<blake3-of-canonical-record-without-signature>",
  "portal_namespace": "org.example.portal",
  "advertisement_type": "org.example.portal.thermals",
  "raw_igc_hashes": ["<blake3-hex>"],
  "resource_refs": [
    {
      "uri": "https://portal.example.org/derived/<hash>/thermals.json",
      "access": "public",
      "media_type": "application/json"
    }
  ],
  "payload": {
    "summary": "thermal annotations available"
  },
  "node_id": "<node-ed25519-public-key-hex>",
  "created_at": "2026-05-01T12:00:00Z",
  "signature": "<ed25519-signature-hex>"
}
```

### 5.2 Wind estimates

```json
{
  "schema": "igc-net/metadata-advertisement",
  "schema_version": 1,
  "record_id": "<blake3-of-canonical-record-without-signature>",
  "portal_namespace": "org.example.portal",
  "advertisement_type": "org.example.portal.wind",
  "raw_igc_hashes": ["<blake3-hex>"],
  "resource_refs": [],
  "payload": {
    "summary": "wind estimate available"
  },
  "node_id": "<node-ed25519-public-key-hex>",
  "created_at": "2026-05-01T12:00:00Z",
  "signature": "<ed25519-signature-hex>"
}
```

Future versions may standardize thermal, wind, scoring, replay, or other
derived-data schemas if operational practice converges. Until then, portals use
their own namespaces and receivers treat payloads as portal-defined data.

---

## 6. Signature-verification attestations

### 6.1 Purpose

A signature-verification attestation is a native signed JSON record by which a
node publicly vouches that it ran a vendor-specific validator against the raw
IGC bytes for a given `raw_igc_hash` and observed a particular result.

igc-net does NOT define cryptographic verification of recorder-device
signatures. Recorder-device signature schemes are vendor-specific and are
verified by external vendor-supplied tools. This record carries only the
result of running such a tool, signed by the attestor.

A signature-verification attestation is a standardized public record. It is
distinct from a portal-defined metadata advertisement (§3): its schema, fields,
and semantics are normative and do not vary by portal. `(R-SIG-09)`

A receiver decides for itself whether to trust any given attestor. igc-net
defines no global verification authority and no protocol-level conflict
resolution between attestations from different attestors.

The parser-side attribute `g_record_present` (defined in `20-artifacts.md §10`)
is the authoritative answer for files that carry no G-record. An attestation
MUST NOT be issued for an IGC where `g_record_present == false`. `(R-SIG-10)`

A signature-verification attestation MUST NOT be used to gate artifact
serving, deduplication, ownership resolution, publication-mode evaluation, or
any governance decision. `(R-SIG-11)`

### 6.2 Shape

```json
{
  "schema": "igc-net/signature-attestation",
  "schema_version": 1,
  "record_id": "<blake3-of-canonical-record-without-signature>",
  "raw_igc_hash": "<blake3-hex>",
  "result": "pass|fail|unsupported",
  "validator_vendor": "<lowercase-ascii-vendor-id>",
  "validator_version": "<freeform-string>",
  "verified_at": "YYYY-MM-DDTHH:MM:SSZ",
  "node_id": "<node-ed25519-public-key-hex>",
  "created_at": "YYYY-MM-DDTHH:MM:SSZ",
  "signature": "<ed25519-signature-hex>"
}
```

### 6.3 Required fields

- `schema` MUST be exactly `"igc-net/signature-attestation"`. `(R-SIG-12)`
- `schema_version` MUST be `1` for this version.
- `record_id = BLAKE3(canonical_json(record_without_signature))`.
- `raw_igc_hash` MUST be a lowercase BLAKE3 hex hash. The attestation anchors
  on the raw IGC identity. An attestation that anchors on any other
  identifier (including `protected_hash`) MUST be rejected. `(R-SIG-13)`
- `result` MUST be exactly one of `"pass"`, `"fail"`, or `"unsupported"`.
  `(R-SIG-14)`
- `validator_vendor` MUST be a non-empty lowercase ASCII string identifying
  the vendor whose validator produced the result. `(R-SIG-15)`
- `validator_version` is a freeform string identifying the validator build,
  release, or invocation parameters. It MAY be empty.
- `verified_at` is the timestamp at which the attestor ran the validator. It
  uses the timestamp format defined in `10-core.md §1.5`.
- `node_id` identifies the attesting node.
- `created_at` is the timestamp at which the record was authored. It uses the
  timestamp format defined in `10-core.md §1.5` and is informational
  (see `10-core.md §1.5`).
- `signature` MUST verify against `node_id`. `(R-SIG-16)`

### 6.4 Result vocabulary

| Value | Meaning |
|-------|---------|
| `pass` | The attestor ran the named validator against the raw IGC bytes for `raw_igc_hash` and the validator returned a positive result. |
| `fail` | The attestor ran the named validator against the raw IGC bytes for `raw_igc_hash` and the validator returned a negative result. |
| `unsupported` | The file carries a G-record but the attestor does not have a validator available for the file's vendor signature scheme. |

`unsupported` is informational only. A receiver MUST NOT treat `unsupported`
as evidence of either signature validity or invalidity. `(R-SIG-17)`

### 6.5 Signing rules

- Signed by the attesting node's `node_id` key following `10-core.md §5`.
- `record_id = BLAKE3(canonical_json(record_without_signature))`.
- `verified_at` is part of the signed payload.

A signature-verification attestation MUST be signed only by `node_id`. It
MUST NOT be signed by `pilot_id`, `resolver_id`, or any other key. Pilot
self-attestation is not meaningful in this protocol because verification
requires a vendor validator that the pilot does not control. `(R-SIG-18)`

### 6.6 Validation and handling

A node receiving a signature-verification attestation MUST validate `schema`,
`schema_version`, `record_id`, `node_id`, `signature`, hash format, result
vocabulary, vendor identifier format, and timestamp formats before indexing
it. Invalid attestations MUST be dropped. `(R-SIG-19)`

Attestation processing is idempotent. Re-receiving the same `record_id` MUST
NOT create a second index entry. `(R-SIG-20)`

Multiple attestations for the same `raw_igc_hash` from different `node_id`s
are allowed and MUST be retained independently. `(R-SIG-21)`

Multiple attestations for the same `(raw_igc_hash, node_id)` pair are allowed.
A receiver MAY treat the attestation with the latest `verified_at` from that
attestor as the attestor's current vouch and MAY retain or discard older
attestations from the same attestor at its discretion. `(R-SIG-22)`

### 6.7 Topic placement

Signature-verification attestations are public records. They are broadcast on
the **data-plane announce topic** alongside artifact announcements and
metadata advertisements. They MUST NOT be broadcast on the governance topic.
`(R-SIG-23)`

### 6.8 Trust model

igc-net does not define which attestors a receiver should trust. Each receiver
selects its own trusted set (typically competition portals, federations, or
the receiver's own node).

A competition portal that requires verified signatures for scoring SHOULD
issue its own attestations for IGCs it has verified, rather than relying on
attestations from other portals. A portal MAY accept attestations from other
attestors as informational signals.

When two attestors disagree on the same `raw_igc_hash`, igc-net does not
resolve the conflict. Both attestations are retained and surfaced; the
receiver applies its own trust policy. `(R-SIG-24)`

### 6.9 Validator vendor identifiers

`validator_vendor` is a freeform lowercase ASCII string in this version. No
registry is defined. Implementations SHOULD use a stable manufacturer
identifier for interoperability, for example:

| Recorder family | Recommended `validator_vendor` |
|-----------------|--------------------------------|
| LXNAV / LX Navigation | `lxnav` |
| Naviter | `naviter` |
| Flytec / Bräuniger | `flytec` |
| FlyMaster | `flymaster` |
| XCTracer | `xctracer` |
| Garrecht / Air Avionics | `garrecht` |
| Volkslogger | `volkslogger` |
| FAI generic | `fai` |

This table is informative. A future revision may publish a normative
identifier registry.

### 6.10 Examples

The following examples are non-normative.

#### 6.10.1 Pass

```json
{
  "schema": "igc-net/signature-attestation",
  "schema_version": 1,
  "record_id": "<blake3-of-canonical-record-without-signature>",
  "raw_igc_hash": "<blake3-hex>",
  "result": "pass",
  "validator_vendor": "lxnav",
  "validator_version": "vali-lxn-2.13",
  "verified_at": "2026-05-07T09:30:00Z",
  "node_id": "<node-ed25519-public-key-hex>",
  "created_at": "2026-05-07T09:30:01Z",
  "signature": "<ed25519-signature-hex>"
}
```

#### 6.10.2 Fail

```json
{
  "schema": "igc-net/signature-attestation",
  "schema_version": 1,
  "record_id": "<blake3-of-canonical-record-without-signature>",
  "raw_igc_hash": "<blake3-hex>",
  "result": "fail",
  "validator_vendor": "naviter",
  "validator_version": "vali-naviter-1.4",
  "verified_at": "2026-05-07T10:15:00Z",
  "node_id": "<node-ed25519-public-key-hex>",
  "created_at": "2026-05-07T10:15:00Z",
  "signature": "<ed25519-signature-hex>"
}
```

#### 6.10.3 Unsupported

```json
{
  "schema": "igc-net/signature-attestation",
  "schema_version": 1,
  "record_id": "<blake3-of-canonical-record-without-signature>",
  "raw_igc_hash": "<blake3-hex>",
  "result": "unsupported",
  "validator_vendor": "xctracer",
  "validator_version": "",
  "verified_at": "2026-05-07T10:20:00Z",
  "node_id": "<node-ed25519-public-key-hex>",
  "created_at": "2026-05-07T10:20:00Z",
  "signature": "<ed25519-signature-hex>"
}
```
