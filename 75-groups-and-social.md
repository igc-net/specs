# igc-net §75 — Groups and Social

**Status:** Normative  
**Depends on:** `60-keys-and-access.md`, `70-durability.md`

## 1. Overview

This specification introduces two social features layered on top of the existing
permission system:

- **Groups** — pilots share private flight access with others without exposing
  content to the global public.
- **Follow** — pilots subscribe to upload notifications from other pilots, with
  access level elevated if they share a group.

Groups operate on the **data plane**: group records are signed by pilots and
stored locally by serving nodes; no trusted resolver approval is required.
Group access is a data-plane authorization layer and is independent of the
governance plane's publication-mode gate.

The existing `PublicationMode` (public / protected / private) is unchanged and
continues to express global visibility.  A valid `GroupFetchProof` **bypasses
the publication-mode gate**: a serving node that confirms current group
membership MUST serve the raw IGC bytes regardless of the flight's global
publication mode, subject to the governance state not being `contested` or
`rejected` and to the node holding the raw IGC bytes locally.

## 2. Group types

### 2.1 Private groups

A private group is created by one pilot (the *owner*).  The owner adds other
pilots by their `PilotId`.  Membership confers access to **all** of the owner's
non-public flights (past and future), bypassing the global publication-mode gate
at the serving node. `(R-GROUP-01)`  Added members can see that they are in the
group via `ListMyGroups` but cannot opt out. `(R-GROUP-02)`  The owner may
remove members at any time.

### 2.2 Public groups

A public group is created by any pilot or portal node.  Pilots are invited and
may opt in or out.  A member MUST share **all** of their flights (past and
future) with all other group members, regardless of global publication mode.
`(R-GROUP-03)`  The obligation applies retroactively at join time and lapses
immediately upon leaving. `(R-GROUP-04)`

All members can see the full member list.

## 3. Identifiers

### 3.1 GroupId

    igcnet:group:<32 lowercase hex chars>

A `GroupId` is content-addressed: it is derived from the `GroupCreationRecord`
fields rather than being random.  The derivation payload is the RFC 8785
canonical JSON of the following fields:

| Field | Value |
|---|---|
| `schema` | `"igc-net/group-creation"` |
| `schema_version` | `1` |
| `group_type` | `"private"` or `"public"` |
| `creator_pilot_id` | the creator's `igcnet:id:<hex>` |
| `name` | display-name string, or `null` if absent |
| `created_at` | RFC3339 timestamp |

```
group_id = "igcnet:group:" + hex(BLAKE3(canonical_json(derivation_payload))[0..16])
```

The `created_at` field provides temporal uniqueness: two groups with the same
creator, type, and name created in different seconds yield different GroupIds.

A `GroupId` is immutable once the corresponding `GroupCreationRecord` is
published; it MUST NOT be reused or reassigned. `(R-GROUP-05)`

A receiving node MUST recompute the `GroupId` from the named fields and MUST
reject the record if the recomputed value does not match the `group_id` field
in the record. `(R-GROUP-15)`

## 4. Record schemas

All records are signed canonical JSON (RFC 8785).  The `record_id` field is
`BLAKE3(canonical_json(payload_without_record_id_and_signature))`.  The
`signature` covers the canonical JSON of all fields except `signature` itself
(i.e. including `record_id`).  Signatures MUST use the signer's Ed25519 root
pilot identity key (the key embedded in `igcnet:id:<hex>`). `(R-GROUP-06)`

### 4.1 GroupCreationRecord

Schema: `igc-net/group-creation` v1

| Field | Type | Notes |
|---|---|---|
| `schema` | string | `"igc-net/group-creation"` |
| `schema_version` | u8 | `1` |
| `record_id` | Blake3Hex | derived |
| `group_id` | GroupId | derived from creation payload (§3.1); receivers MUST verify |
| `group_type` | string | `"private"` or `"public"` |
| `creator_pilot_id` | PilotId | signer |
| `name` | string? | optional display name |
| `created_at` | string | canonical UTC RFC3339 seconds |
| `signature` | string | 128 lowercase hex Ed25519 |

### 4.2 PrivateGroupMemberAddRecord

Schema: `igc-net/private-group-member-add` v1

Signer: group owner; `added_by_pilot_id` MUST equal the `creator_pilot_id` of the group's `GroupCreationRecord`. `(R-GROUP-07)`

| Field | Type |
|---|---|
| `schema` | string |
| `schema_version` | u8 |
| `record_id` | Blake3Hex |
| `group_id` | GroupId |
| `member_pilot_id` | PilotId |
| `added_by_pilot_id` | PilotId |
| `created_at` | string |
| `signature` | string |

### 4.3 PrivateGroupMemberRemoveRecord

Schema: `igc-net/private-group-member-remove` v1

Signer: group owner.  Same structure as §4.2.

### 4.4 PublicGroupInviteRecord

Schema: `igc-net/public-group-invite` v1

Signer: any existing member of the public group. `(R-GROUP-08)`

| Field | Type |
|---|---|
| `schema` | string |
| `schema_version` | u8 |
| `record_id` | Blake3Hex |
| `group_id` | GroupId |
| `invited_pilot_id` | PilotId |
| `invited_by_pilot_id` | PilotId |
| `created_at` | string |
| `signature` | string |

### 4.5 PublicGroupAcceptRecord

Schema: `igc-net/public-group-accept` v1

Signer: accepting pilot (`member_pilot_id`).

| Field | Type |
|---|---|
| `schema` | string |
| `schema_version` | u8 |
| `record_id` | Blake3Hex |
| `group_id` | GroupId |
| `member_pilot_id` | PilotId |
| `created_at` | string |
| `signature` | string |

### 4.6 PublicGroupLeaveRecord

Schema: `igc-net/public-group-leave` v1

Signer: leaving pilot (`member_pilot_id`).  Same structure as §4.5.

## 5. Group-based artifact fetch

### 5.1 GroupFetchProof wire format

Schema: `igc-net/group-fetch-request` v1

The proof travels inside `FetchArtifactRequest.group_fetch_proof`.  The
`raw_igc_hash` and `artifact_class` from the outer request MUST be included in
the signing payload to bind the proof to the specific artifact. `(R-GROUP-09)`

Signing payload (canonical JSON of):

```json
{
  "schema": "igc-net/group-fetch-request",
  "schema_version": 1,
  "raw_igc_hash": "<64-hex>",
  "artifact_class": "<snake_case>",
  "requester_pilot_id": "igcnet:id:<hex>",
  "group_id": "igcnet:group:<32-hex>",
  "seq_num": 1
}
```

The `requester_pilot_id` encodes the Ed25519 public key directly, so no
external key-registry lookup is needed during verification.

### 5.2 Sequence numbers

Group-fetch `seq_num` values are keyed by `requester_pilot_id` and MUST be
stored separately from private-access `seq_num` values.  Group-fetch `seq_num`
values MUST be strictly monotonically increasing per `requester_pilot_id`.
`(R-GROUP-10)` (Extends R-ACCESS-11 to group fetches.)

### 5.3 Server-side access decision

```
FetchArtifactRequest arrives
│
├─ artifact class is PUBLIC → serve (existing path)
├─ group_fetch_proof present?
│   ├─ verify signature against pubkey in requester_pilot_id
│   ├─ verify seq_num monotonicity (group seq_num store)
│   ├─ check governance: contested/rejected blocks group access
│   ├─ verify node holds raw IGC bytes for this raw_igc_hash
│   ├─ requester in owner's PRIVATE GROUP? → serve raw_igc_hash blob
│   └─ requester and owner share a PUBLIC GROUP? → serve raw_igc_hash blob
│   └─ neither → DENY
└─ existing private-access key path (unchanged)
```

Normative requirements for each step:

- A serving node MUST verify the `GroupFetchProof` signature against the
  Ed25519 public key encoded in `requester_pilot_id`. `(R-GROUP-11)`
- A serving node MUST refuse group-based fetch requests for any `raw_igc_hash`
  in `contested` or `rejected` governance state. `(R-GROUP-12)`
- A serving node MUST serve the raw IGC blob if the requester is a confirmed
  current member of the artifact owner's private group, or if the requester
  and the artifact owner share at least one public group. `(R-GROUP-13)`
- A serving node MUST deny group-based fetch requests where group membership
  cannot be confirmed from locally stored group records. `(R-GROUP-14)`

Group members always receive the raw IGC bytes (`private_raw_igc` semantics
regardless of global publication mode). `(R-GROUP-01)`  If the raw IGC bytes
have been locally purged (e.g., a protected-mode flight after sanitisation, or
following a deletion request), the node returns `not_found`.

## 6. Follow records

### 6.1 FollowRecord

Schema: `igc-net/follow` v1

Signer: follower pilot (`follower_pilot_id`).

| Field | Type |
|---|---|
| `schema` | string |
| `schema_version` | u8 |
| `record_id` | Blake3Hex |
| `follower_pilot_id` | PilotId |
| `followee_pilot_id` | PilotId |
| `created_at` | string |
| `signature` | string |

### 6.2 UnfollowRecord

Schema: `igc-net/unfollow` v1.  Same structure as §6.1.

### 6.3 Follow visibility semantics

- Notification events for new uploads from followed pilots are surfaced via the
  existing `SubscribeEvents` stream (client-side filtering by pilot_id).
- What the follower can see follows the followee's global `PublicationMode` by
  default.
- If the follower and followee share at least one group, the group access rules
  (§5.3) apply when the follower fetches.

## 7. Membership visibility

| Group type | Who sees membership |
|---|---|
| Private | Owner sees all members; member sees that they are in the group via `ListMyGroups` |
| Public | All members see the full member list via `ListGroupMembers` |

## 8. Group storage layout

```
{data-root}/groups/
  creations/              ← one JSON file per GroupCreationRecord (named by record_id)
  private-member-adds/    ← PrivateGroupMemberAddRecord files
  private-member-removes/ ← PrivateGroupMemberRemoveRecord files
  public-invites/         ← PublicGroupInviteRecord files
  public-accepts/         ← PublicGroupAcceptRecord files
  public-leaves/          ← PublicGroupLeaveRecord files

{data-root}/follows/
  follow-records/         ← FollowRecord files (named by record_id)
  unfollow-records/       ← UnfollowRecord files

{data-root}/seq-nums-group/
  <pilot_id_hex>.json     ← last-seen group-fetch seq_num per requester pilot_id
```
