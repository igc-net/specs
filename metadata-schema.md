# `igc-net` Metadata Schema Reference

This is the compact reference for the base metadata schema published as
`schema = "igc-net/metadata"` and `schema_version = 1`.

The normative definition remains `specs_meta.md` Part I.

## Required Fields

| Field | Type | Constraints |
|---|---|---|
| `schema` | string | MUST equal `igc-net/metadata` |
| `schema_version` | integer | MUST equal `1` |
| `igc_hash` | string | 64-character lowercase BLAKE3 hex |

## Optional Fields

| Field | Type | Notes |
|---|---|---|
| `original_filename` | string | upload-time filename |
| `flight_date` | string | `YYYY-MM-DD`, typically from `HFDTE` |
| `started_at` | string | canonical UTC timestamp from first valid B record |
| `ended_at` | string | canonical UTC timestamp from last valid B record |
| `duration_s` | integer | derived from `started_at` and `ended_at` |
| `pilot_name` | string | from `HFPLT` |
| `glider_type` | string | from `HFGTY` |
| `glider_id` | string | from `HFGID` |
| `device_id` | string | logger serial / device identifier if present |
| `fix_count` | integer | count of all B records |
| `valid_fix_count` | integer | count of B records with validity flag `A` |
| `bbox` | object | `min_lat`, `max_lat`, `min_lon`, `max_lon` |
| `launch_lat` | number | first parsed B-record latitude |
| `launch_lon` | number | first parsed B-record longitude |
| `landing_lat` | number | last parsed B-record latitude |
| `landing_lon` | number | last parsed B-record longitude |
| `max_alt_m` | integer | pressure altitude if complete/non-zero, else GPS altitude |
| `min_alt_m` | integer | pressure altitude if complete/non-zero, else GPS altitude |
| `publisher_node_id` | string | 64-character lowercase Ed25519 public key hex |
| `published_at` | string | canonical UTC publish timestamp |

## Serialization Rules

- Optional fields MUST be omitted if unavailable
- Optional fields MUST NOT be serialized as `null`
- JSON MUST be UTF-8
- Canonical timestamps use `YYYY-MM-DDTHH:MM:SSZ`
- Coordinate values are WGS84 decimal degrees

## Minimal Valid Example

```json
{
  "schema": "igc-net/metadata",
  "schema_version": 1,
  "igc_hash": "3a7f2c8e1b4d5f6a9b0c1d2e3f4a5b6c7d8e9f0a1b2c3d4e5f6a7b8c9d0e1f2a"
}
```

## Full Example

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
