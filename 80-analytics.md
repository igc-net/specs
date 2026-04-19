# igc-net — Analytics (Optional Extension)

**Status:** Normative (optional)  
**Depends on:** `10-core.md`, `20-artifacts.md`, `30-transport.md`

---

## 1. Optional extension

Analytics are an optional extension. A node that does not implement this
document remains fully conformant with the rest of the core specification.

Analytics are explicitly excluded from the core specification. They exist on a
different timescale and with different trust semantics from artifact identity,
ownership, and permissions. This document provides the minimum framing needed
to prevent analytics from contaminating core metadata semantics.

---

## 2. Scope

Analytics in igc-net are derived data computed from flight artifacts:

- Derived from `public` and `protected` (sanitized) flights without any
  additional permission beyond what the flight's `publication_mode` already
  permits.
- Analytics for `private` flights require **explicit pilot consent**; the
  mechanism for tracking that consent is outside the core specification.

Most analytics providers have no legitimate need for pilot personal data
and MAY operate as **identity-linked nodes** (Category 1 — see
`60-keys-and-access.md §2.1`) for the pilots whose flights they process.
In that mode they consume only public and sanitized-protected artifacts
and they do not hold the pilot's `private_access_keypair`.

Providers that require authoritative pilot identity for their function
(e.g., competition scoring bound to a sanctioning body) operate as
**private-access nodes** (Category 2 — see `60-keys-and-access.md §2.2`)
under an explicit pilot grant. The same analytics record shapes apply; the
difference is only in what underlying records the provider is authorized
to read.

In either mode, analytics providers that store plaintext IGC bytes remain
bound by the scrape-avoidance obligation in `60-keys-and-access.md §2.1`:
they SHOULD NOT extract personal-identity fields from public IGC headers
outside records the pilot has explicitly authorized as public.

---

## 3. Analytics objects

An analytics object is a structured record containing derived data associated
with a `raw_igc_hash`. Minimal shape:

```json
{
  "schema": "igc-net/analytics",
  "schema_version": 1,
  "record_id": "<blake3-of-canonical-record-without-signature>",
  "raw_igc_hash": "<blake3-hex>",
  "provider_id": "<ed25519-public-key-hex>",
  "analytics_type": "<string>",
  "payload": {},
  "created_at": "YYYY-MM-DDTHH:MM:SSZ",
  "signature": "<ed25519-signature-hex>"
}
```

- `raw_igc_hash` links the analytics record to the flight identity anchor.
- `provider_id` identifies the producing service (its Ed25519 public key).
  The record MUST be signed by the key identified in `provider_id`.
- `analytics_type` is an opaque string identifying the kind of analytics.
- `payload` is analytics-type-specific content.

---

## 4. Trust boundary

Analytics objects are explicitly NOT authoritative over:

- `raw_igc_hash` identity or any derived hash.
- `PilotProfileCredential` or `flight-metadata` fields.
- Ownership or governance state.
- `publication_mode` or `visibility`.

Consumers of analytics data MUST treat them as **derived, optional, and
potentially stale**. `(R-ANA-01)` An analytics object does not alter, supersede, or
affect the canonical identity or authoritative metadata records. `(R-ANA-02)`

Derived track formats (e.g., pre-computed geometry for UI rendering) are
analytics objects. They are NOT canonical protected artifacts and do not
affect `protected_hash`.

---

## 5. Provider attribution

Analytics objects are attributed to the producing portal or service via
`provider_id` and `signature`. Attribution is not equivalent to flight
ownership and carries no governance weight.

The producing service's identity is verifiable: the record MUST be signed
by the key identified in `provider_id`.

---

## 6. Analytics transport

Analytics records MUST be broadcast on the data-plane announce topic (defined
in `30-transport.md`) using the distinct `schema` value `"igc-net/analytics"`. `(R-ANA-03)`
Receivers distinguish analytics records from artifact announcements by inspecting
the `schema` field.

For high-volume analytics deployments, a separate analytics topic MAY be used
in addition to the announce topic:

```
analytics_topic_id = BLAKE3(UTF-8("igc-net/analytics/v1"))[0..32]
```

The derivation follows the same pattern as the governance and announce topics:
exact UTF-8 input bytes, first 32 bytes of BLAKE3 output, 64-character
lowercase hex encoding.

Implementations that use a separate analytics topic MUST still broadcast on the
announce topic as the default. A node that subscribes only to the announce topic
MUST receive analytics from all conformant implementations.
