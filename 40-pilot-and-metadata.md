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
