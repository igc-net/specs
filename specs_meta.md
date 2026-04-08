# `igc-net`: Metadata Schema and Analytics Extension

**Status:** Working draft  
**Version:** 0.1  
**Date:** 2026-04-03

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be
interpreted as described in [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119).

This document specifies:

- **Part I** — the base metadata schema (`schema_version = 1`) for flights
  published via `igc-net`
- **Part II** — the optional `IGC_META_DOC` analytics extension

The base IGC blob publishing and announcement protocol is specified in
`specs_igc.md`.

Part I is implemented in the current Rust reference implementation. Part II is
specified here but not yet implemented in that reference implementation.

## Part I — Base Metadata Schema

### 1. Encoding

- the metadata blob MUST be a UTF-8 JSON object
- no BOM is permitted
- field order is not significant
- a trailing newline is OPTIONAL

### 2. Field Reference

| Field | JSON type | Required | Constraints | Source |
|---|---|---|---|---|
| `schema` | string | yes | MUST equal `igc-net/metadata` | literal |
| `schema_version` | integer | yes | MUST equal `1` | literal |
| `igc_hash` | string | yes | 64 lowercase hex chars | computed |
| `original_filename` | string | no | upload-time filename | publisher |
| `flight_date` | string | no | `YYYY-MM-DD` | `HFDTE` |
| `started_at` | string | no | canonical UTC timestamp | first valid B record |
| `ended_at` | string | no | canonical UTC timestamp | last valid B record |
| `duration_s` | integer | no | non-negative | derived |
| `pilot_name` | string | no | as present in the IGC file | `HFPLT` |
| `glider_type` | string | no | as present in the IGC file | `HFGTY` |
| `glider_id` | string | no | as present in the IGC file | `HFGID` |
| `device_id` | string | no | logger or device identifier if present | IGC headers |
| `fix_count` | integer | no | all B records | derived |
| `valid_fix_count` | integer | no | B records with validity flag `A` | derived |
| `bbox` | object | no | see §3 | derived |
| `launch_lat` | number | no | WGS84 decimal degrees | first parsed B record |
| `launch_lon` | number | no | WGS84 decimal degrees | first parsed B record |
| `landing_lat` | number | no | WGS84 decimal degrees | last parsed B record |
| `landing_lon` | number | no | WGS84 decimal degrees | last parsed B record |
| `max_alt_m` | integer | no | metres | derived |
| `min_alt_m` | integer | no | metres | derived |
| `publisher_node_id` | string | no | 64 lowercase Ed25519 public key hex | publisher |
| `published_at` | string | no | canonical UTC timestamp | publish time |

### 3. `bbox`

When present, `bbox` MUST contain exactly:

| Field | Type | Constraints |
|---|---|---|
| `min_lat` | number | `-90 <= value <= 90` |
| `max_lat` | number | `-90 <= value <= 90` and `max_lat >= min_lat` |
| `min_lon` | number | `-180 <= value <= 180` |
| `max_lon` | number | `-180 <= value <= 180` and `max_lon >= min_lon` |

### 4. Derivation Rules

| Field | Derivation |
|---|---|
| `duration_s` | integer seconds from `started_at` to `ended_at`; omit if either timestamp is absent |
| `fix_count` | count of all B records |
| `valid_fix_count` | count of B records with validity flag `A` |
| `bbox` | min/max latitude and longitude over parsed B records |
| `launch_lat`, `launch_lon` | coordinates of the first parsed B record |
| `landing_lat`, `landing_lon` | coordinates of the last parsed B record |
| `max_alt_m`, `min_alt_m` | pressure altitude if all pressure altitude values are non-zero; otherwise GPS altitude |

### 5. Serialization Rules

- optional fields that are unavailable MUST be omitted
- optional fields MUST NOT be serialized as `null`
- publishers MUST NOT fabricate flight-derived values not derivable from the
  IGC source file
- publish-context fields such as `original_filename`, `publisher_node_id`, and
  `published_at` MAY be added by the publisher at publish time
- string values SHOULD be trimmed before inclusion
- timestamps MUST use `YYYY-MM-DDTHH:MM:SSZ`
- hashes and node IDs MUST use lowercase hexadecimal

### 6. Immutability

A metadata blob MUST NOT be modified after publication. Changing the bytes
changes the blob hash and therefore creates a different metadata blob.

### 7. Multiple Metadata Blobs for the Same `igc_hash`

Multiple metadata blobs for the same `igc_hash` are valid. Different publishers
may populate different fields or publish at different times.

Current recommended selection rule:

1. prefer the blob with more populated fields
2. if tied, prefer the blob with the later `published_at`

### 8. Schema Versioning

- `schema_version` is a positive integer
- unsupported versions MUST be skipped silently
- additive optional fields MAY be introduced without incrementing
  `schema_version` if they do not break existing parsers
- breaking semantic changes MUST increment `schema_version`

### 9. Examples

Minimal valid metadata blob:

```json
{
  "schema": "igc-net/metadata",
  "schema_version": 1,
  "igc_hash": "3a7f2c8e1b4d5f6a9b0c1d2e3f4a5b6c7d8e9f0a1b2c3d4e5f6a7b8c9d0e1f2a"
}
```

Expanded metadata blob:

```json
{
  "schema": "igc-net/metadata",
  "schema_version": 1,
  "igc_hash": "3a7f2c8e1b4d5f6a9b0c1d2e3f4a5b6c7d8e9f0a1b2c3d4e5f6a7b8c9d0e1f2a",
  "original_filename": "2024-08-10_kluszkowce.igc",
  "flight_date": "2024-08-10",
  "started_at": "2024-08-10T10:15:32Z",
  "ended_at": "2024-08-10T13:47:11Z",
  "duration_s": 12699,
  "pilot_name": "Jan Kowalski",
  "glider_type": "Ozone Zeno 2",
  "glider_id": "D-3412",
  "fix_count": 13020,
  "valid_fix_count": 12980,
  "bbox": {
    "min_lat": 49.38,
    "max_lat": 49.51,
    "min_lon": 20.11,
    "max_lon": 20.31
  },
  "launch_lat": 49.4812,
  "launch_lon": 20.1923,
  "landing_lat": 49.3941,
  "landing_lon": 20.2847,
  "max_alt_m": 2134,
  "min_alt_m": 612,
  "publisher_node_id": "a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1b2",
  "published_at": "2024-08-10T14:02:00Z"
}
```

## Part II — `IGC_META_DOC` Analytics Extension

### 10. Purpose

`IGC_META_DOC` defines how derived analytics for a published flight can be
published, discovered, and exchanged.

The extension sits above the base protocol:

```text
analytics blobs + IGC_META_DOC
        above
base igc-net protocol
```

Part II is OPTIONAL. Base-protocol-only nodes remain fully compliant.

### 11. Deterministic Identifiers

All analytics traffic uses deterministic derivations from public identifiers.

#### 11.1 Analytics Gossip Topic

```text
analytics_topic_id = BLAKE3("igc-net/analytics/v1")[0..32]
```

Canonical hex value:

```text
4f29345b6986886351dbff97b4231b7d9a601a5098f18480c2cb4d28fbe30950
```

#### 11.2 Per-flight Namespace Secret

```text
namespace_secret = hash("igc-net/meta-doc/v1" + igc_hash_bytes)
```

Because this value is derived from a public `igc_hash`, writes are
protocol-open. Attribution and trust are handled at the application layer.

### 12. Analytics Announcements

#### 12.1 Topic

All analytics announcement and re-announcement request traffic uses the single
well-known analytics topic.

#### 12.2 Announcement Message

```json
{
  "igc_hash": "<blake3-hex>",
  "analytics_type": "<type-identifier>",
  "author_id": "<ed25519-public-key-hex>",
  "doc_ticket": "<iroh-docs-share-ticket>",
  "data_hash": "<blake3-hex>"
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `igc_hash` | string | yes | flight hash |
| `analytics_type` | string | yes | versioned type identifier |
| `author_id` | string | yes | analytics author key |
| `doc_ticket` | string | yes | document share ticket |
| `data_hash` | string | yes | BLAKE3 hash of the analytics blob |

Rules:

- announcements MUST be valid UTF-8 JSON
- announcements MUST NOT exceed 1024 bytes
- unknown fields MUST be ignored

#### 12.3 Re-announcement Request

```json
{
  "request": "announce",
  "igc_hash": "<blake3-hex>"
}
```

The request MUST NOT exceed 256 bytes.

#### 12.4 Handling Rules

On receiving an analytics announcement, an endpoint:

1. MUST verify required fields are present
2. MAY filter by `igc_hash` and `analytics_type`
3. MUST use `doc_ticket` to sync if it decides to consume the announcement

On receiving a re-announcement request:

1. a node holding analytics for that `igc_hash` SHOULD re-announce
2. nodes SHOULD rate-limit responses
3. nodes with no matching analytics MUST ignore the request

### 13. `IGC_META_DOC` Structure

#### 13.1 Document Identity

Each flight has one deterministic `IGC_META_DOC` namespace derived from
`igc_hash`.

#### 13.2 Document Keys

Entries use:

```text
analytics/{analytics_type}
```

Examples:

```text
analytics/igc-net/thermals/v1
analytics/igc-net/wind/v1
analytics/com.example/scoring/v1
```

#### 13.3 Entry Value

```json
{
  "data_hash": "<blake3-hex>",
  "data_ticket": "<iroh-blobs-ticket>",
  "computed_at": "<iso8601-utc>"
}
```

The document points to content-addressed analytics blobs rather than embedding
large analytics payloads inline.

### 14. Attribution and Trust

- every document write is attributable to the writer's author key
- the protocol defines no trust registry
- applications MUST NOT assume that the presence of an entry implies endorsement
  by the original publisher

### 15. Standard Analytics Types

All standard analytics types use the `igc-net/` prefix.

General header fields for analytics blobs:

| Field | Type | Required | Description |
|---|---|---|---|
| `schema` | string | yes | unversioned type identifier, e.g. `igc-net/thermals` |
| `schema_version` | integer | yes | currently `1` |
| `igc_hash` | string | yes | referenced flight hash |
| `computed_at` | string | yes | canonical UTC computation timestamp |

Unsupported analytics schema versions MUST be skipped silently.

#### 15.1 Classification — `igc-net/classification/v1`

```json
{
  "schema": "igc-net/classification",
  "schema_version": 1,
  "igc_hash": "<blake3-hex>",
  "computed_at": "2026-04-03T10:00:00Z",
  "discipline": "PG",
  "source": "igc_header",
  "confidence": 90
}
```

#### 15.2 Site Match — `igc-net/sites/v1`

```json
{
  "schema": "igc-net/sites",
  "schema_version": 1,
  "igc_hash": "<blake3-hex>",
  "computed_at": "2026-04-03T10:00:00Z",
  "launch": {
    "site_id": "dhv:1234",
    "site_name": "Wallberg",
    "site_registry": "dhv",
    "lat": 47.7015,
    "lon": 11.7283,
    "distance_m": 48,
    "confidence": 97
  }
}
```

`launch` and `landing` are each OPTIONAL.

#### 15.3 Thermals — `igc-net/thermals/v1`

```json
{
  "schema": "igc-net/thermals",
  "schema_version": 1,
  "igc_hash": "<blake3-hex>",
  "computed_at": "2026-04-03T10:00:00Z",
  "thermals": [
    {
      "start_time": "2026-04-03T10:15:00Z",
      "end_time": "2026-04-03T10:18:30Z",
      "duration_s": 210,
      "centre_lat": 46.1234,
      "centre_lon": 7.5678,
      "radius_m": 45,
      "altitude_entry_m": 1200,
      "altitude_exit_m": 1850,
      "altitude_gain_m": 650,
      "climb_rate_ms": 3.1,
      "climb_rate_source": "vario"
    }
  ]
}
```

#### 15.4 Wind — `igc-net/wind/v1`

```json
{
  "schema": "igc-net/wind",
  "schema_version": 1,
  "igc_hash": "<blake3-hex>",
  "computed_at": "2026-04-03T10:00:00Z",
  "summary": {
    "speed_ms": 4.5,
    "direction_deg": 270,
    "source": "derived",
    "confidence": 60
  },
  "profile": [
    {
      "altitude_band_low_m": 1000,
      "altitude_band_high_m": 1100,
      "speed_ms": 3.2,
      "direction_deg": 265,
      "source": "k_record",
      "confidence": 85,
      "sample_count": 12
    }
  ]
}
```

#### 15.5 Segments — `igc-net/segments/v1`

```json
{
  "schema": "igc-net/segments",
  "schema_version": 1,
  "igc_hash": "<blake3-hex>",
  "computed_at": "2026-04-03T10:00:00Z",
  "segments": [
    {
      "phase": "straight",
      "start_time": "2026-04-03T10:10:00Z",
      "end_time": "2026-04-03T10:15:00Z",
      "duration_s": 300,
      "start_lat": 46.1234,
      "start_lon": 7.5678,
      "end_lat": 46.1456,
      "end_lon": 7.5890,
      "wind_relation": "upwind"
    }
  ]
}
```

### 16. Provider Extensions

Custom analytics types MUST use reverse-DNS namespacing:

```text
{reverse-dns-domain}/{type-name}/v{n}
```

Examples:

```text
com.cloud-street/scoring/v1
org.xcontest/fai-triangle/v1
net.dhv-xc/thermal-tier/v1
```

Custom blobs MUST still include the standard analytics header fields.

### 17. Discovery and Sync Flow

Consumer flow:

```text
join analytics topic
  -> receive analytics announcement
  -> filter by igc_hash and analytics_type
  -> sync IGC_META_DOC via doc_ticket
  -> inspect author attribution
  -> fetch analytics blob via data_ticket
  -> validate schema, schema_version, and igc_hash
```

Publisher flow:

```text
compute analytics blob
  -> store analytics blob in iroh-blobs
  -> derive namespace from igc_hash
  -> write analytics entry into IGC_META_DOC
  -> publish analytics announcement
```

### 18. References

- `specs_igc.md`
- `requirements.md`
- [BLAKE3 specification](https://github.com/BLAKE3-team/BLAKE3-specs)
- [iroh-docs crate](https://docs.rs/iroh-docs)
