# igc-net — Derived Metadata and Analytics

**Status:** Normative boundary for pre-v0.5  
**Depends on:** `40-pilot-and-metadata.md`

---

## 1. Status

Standard analytics records are deferred until after v0.5.

Pre-v0.5 igc-net does not define a native `igc-net/analytics` record, analytics
topic, merge policy, scoring schema, thermal schema, wind schema, or
`IGC_META_DOC`.

Portals that want to advertise derived data use the public
`igc-net/metadata-advertisement` record defined in
`40-pilot-and-metadata.md`.

---

## 2. Boundary

Derived metadata and analytics advertisements are portal-defined. They do not
alter:

- `raw_igc_hash`
- `protected_hash`
- ownership claims
- governance state
- publication mode
- fetch authorization
- protected/private artifact access policy

An advertisement can point at public, protected, private, or portal-local
resources. The advertisement itself is public. Access to the referenced resource
is the publishing portal's policy.

---

## 3. Non-normative examples

Thermal annotations, wind estimates, scoring outputs, task analysis, replay
geometry, and climb summaries are all valid examples of portal-defined metadata
advertisements.

These examples are intentionally not standardized in pre-v0.5. If ecosystem
practice converges around common thermal, wind, or scoring formats, a later
version may standardize those formats as explicit extension schemas.
