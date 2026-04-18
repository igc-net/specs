# igc-net v0.2 — Transport

**Status:** Normative  
**Depends on:** `10-core.md`, `20-artifacts.md`

---

## 1. Data-plane announce topic

The data-plane announce topic is the well-known publish/subscribe topic used
for flight artifact announcements. It is separate from the governance topic.

```
announce_topic_id = BLAKE3(UTF-8("igc-net/announce/v1"))[0..32]
```
`(R-TRANS-13)`

Derivation rules:

- Input: the exact UTF-8 bytes of the string `"igc-net/announce/v1"`, with no
  null terminator and no length prefix.
- Output: the first 32 bytes of the BLAKE3 hash, encoded as 64-character
  lowercase hex.

All data-plane artifact announcements are broadcast on this topic. The
governance topic (defined in `55-governance-sync.md`) is separate;
governance records MUST NOT be broadcast on the announce topic `(R-TRANS-01)`,
and artifact announcements MUST NOT be broadcast on the governance topic. `(R-TRANS-02)`

---

## 2. Announcement record

### 2.1 Purpose

An announcement record advertises that an artifact identified by
`raw_igc_hash` is available at the listed ticket(s). The `node_id`
identifies the announcing node. The ticket(s) identify where the artifact
may be fetched — which may be a different serving node.

An announcement MUST NOT embed raw IGC bytes. `(R-TRANS-11)`

### 2.2 Shape

```json
{
  "schema": "igc-net/announcement",
  "schema_version": 1,
  "record_id": "<blake3-of-canonical-record-without-signature>",
  "raw_igc_hash": "<blake3-hex>",
  "publication_mode": "<public|protected|private>",
  "tickets": ["<ticket>", "..."],
  "node_id": "<node-ed25519-public-key-hex>",
  "signature": "<ed25519-signature-hex>",
  "created_at": "YYYY-MM-DDTHH:MM:SSZ"
}
```

For `protected` mode, the announcement MUST additionally carry
`protected_hash`. The announcement SHOULD carry `companion_tickets` when the
announcing node holds and serves the raw companion; it MAY be omitted when
the node holds only the sanitized artifact.

```json
{
  "protected_hash": "<blake3-hex>",
  "companion_tickets": ["<ticket>", "..."]
}
```

For `private` mode, no additional artifact-identity fields are carried —
`raw_igc_hash` alone identifies the private flight and its raw plaintext
artifact.

### 2.3 Required fields

- `raw_igc_hash`: the identity anchor for the flight.
- `publication_mode`: the current mode at the time of announcement.
- `tickets`: one or more locators for the announced artifact. For `public`
  and `protected` (sanitized artifact), these point to the plaintext public
  artifact. For `private`, these point to the plaintext raw IGC served
  under authorization.
- `node_id`: the announcing node's identity (Ed25519 public key hex).
- `protected_hash`: MUST be present when `publication_mode` is
  `"protected"`. `(R-TRANS-03)` MUST be absent for `"public"` and
  `"private"` modes. `(R-TRANS-04)`
- `companion_tickets`: SHOULD be present when the announcing node holds and
  serves the raw companion for a protected flight. MAY be omitted when the
  node holds only the sanitized artifact. MUST be absent for `"public"` and
  `"private"` modes. `(R-TRANS-15)`

### 2.4 Signing rules

- The announcement MUST be signed by the announcing node's `node_id` key.
- `record_id = BLAKE3(canonical_json(record_without_signature))`.
- `created_at` is informational.

---

## 3. Private-flight existence announcement

For `private` flights, the data plane owns existence discovery.

The announcement for a private flight carries `raw_igc_hash` and a ticket
list. This allows private-access nodes (Category 2) that hold the pilot's
`private_access_keypair` to:

1. Discover that this private flight exists.
2. Obtain a ticket to fetch the raw IGC plaintext.
3. Authenticate with a fetch request signed by `private_access_keypair` to
   receive the bytes.

This announcement belongs on the **data-plane announce topic**.
Private-flight existence records MUST NOT be carried on the governance
topic. The governance topic carries only policy state (claims, approvals,
challenges, resolutions, mode records, deletion requests, private-access
rotation records).

---

## 4. Ticket and locator semantics

A ticket is a locator. It encodes:

- the serving node endpoint (where to send the fetch request), and
- the blob identifier (which artifact to request from that node).

**Ticket possession is not authorization.** Whether a requester may receive
the artifact is decided by the serving node at delivery time, based on:

- the current governance state for `raw_igc_hash`, and
- key-possession proof from the requester (for `protected` raw companion
  and `private` content).

A node that cannot produce a valid fetch-request signature, or whose
request is for a flight in `contested` or `rejected` governance state,
receives nothing.

---

## 5. Deduplication

The same `raw_igc_hash` MUST deduplicate to one canonical identity record
in the index, regardless of how many portals announce it. `(R-TRANS-05)`

Multiple tickets for the same `raw_igc_hash` represent different serving
peers. They are stored for redundancy and do not create multiple flight
entries.

A flight appears once in the index regardless of how many portals hold and
announce it.

---

## 6. Re-announcement rules by mode

Re-announcement rights depend on `publication_mode`: `(R-TRANS-14)`

| Mode | What may be re-announced |
|------|--------------------------|
| `public` | Any node MAY re-announce the artifact ticket freely after verifying the bytes. Desirable for durability. |
| `protected` | The sanitized artifact ticket MAY be re-announced freely. The raw companion ticket MAY be re-announced by any node that serves the companion; the serving node gates delivery by fetch-request signature. |
| `private` | The raw-plaintext ticket MAY be re-announced by any private-access node that serves the content; the serving node gates delivery by fetch-request signature. The existence record (`raw_igc_hash`) propagates for cross-portal discovery. |

In all cases, the ticket is a locator only. Authorization is enforced at
delivery time by the serving node, not by ticket availability.

---

## 7. Fetch rules by mode

### 7.1 Public artifact

- Any node MAY fetch without authorization.
- The serving node MUST check governance state before transmitting. `(R-TRANS-12)`
- If the flight is in `contested` or `rejected` governance state, the
  serving node MUST refuse the fetch regardless of the requester's
  identity.

### 7.2 Protected sanitized artifact

- Any node MAY fetch the sanitized artifact without authorization.
- Same governance-state pre-check as §7.1 applies.

### 7.3 Protected raw companion

- The requester MUST prove possession of the pilot's
  `private_access_keypair` by signing the fetch request with the
  corresponding private key. `(R-TRANS-06)`
- The serving node MUST verify this signature against the pilot's current
  `private_access_public_key` (see `60-keys-and-access.md §4`, §6) before
  transmitting the raw companion plaintext. `(R-TRANS-07)`
- Governance-state pre-check applies.
- The raw companion is transmitted as plaintext over iroh; no
  protocol-level content encryption is applied.

### 7.4 Private raw IGC

- Same as §7.3: `private_access_keypair` possession proof required.
- The raw IGC is transmitted as plaintext over iroh; no protocol-level
  content encryption is applied.

### 7.5 Governance state pre-check

Before transmitting any artifact, the serving node MUST check the current
governance state for the `raw_igc_hash`:

- `contested` governance state: MUST refuse all fetches. `(R-TRANS-08)`
- `rejected` governance state: MUST refuse all fetches. `(R-TRANS-09)`
- `approved` or no claim: proceed with mode-appropriate fetch rules.

Governance state takes precedence over key possession. A requester who
holds a valid `private_access_keypair` but requests a flight in
`contested` or `rejected` state MUST be refused.

---

## 8. Catch-up for public content discovery

A node that was offline and then rejoins the network MUST be able to
discover public flights it missed.

Implementations MUST support at least one of: `(R-TRANS-10)`

- A gossip history window of sufficient depth (minimum recommended:
  30 days).
- A pull-based sync endpoint exposed by serving nodes, allowing a rejoining
  node to fetch announcement records by time range.

This requirement applies to public content discovery on the data-plane
announce topic. Governance catch-up requirements are defined in
`55-governance-sync.md`.
