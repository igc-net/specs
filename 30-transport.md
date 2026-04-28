# igc-net — Transport

**Status:** Normative  
**Depends on:** `10-core.md`, `20-artifacts.md`

---

## 1. Data-plane announce topic

The data-plane announce topic is the well-known publish/subscribe topic used
for public data-plane records: flight artifact announcements and metadata
advertisements. It is separate from the governance topic.

```
announce_topic_id = BLAKE3(UTF-8("igc-net/announce/v1"))[0..32]
```
`(R-TRANS-13)`

Derivation rules:

- Input: the exact UTF-8 bytes of the string `"igc-net/announce/v1"`, with no
  null terminator and no length prefix.
- Output: the first 32 bytes of the BLAKE3 hash, encoded as 64-character
  lowercase hex.

All data-plane artifact announcements and metadata advertisements are broadcast
on this topic. The
governance topic (defined in `55-governance-sync.md`) is separate;
governance records MUST NOT be broadcast on the announce topic `(R-TRANS-01)`,
and artifact announcements or metadata advertisements MUST NOT be broadcast on
the governance topic. `(R-TRANS-02)`

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

- Signed by the announcing node's `node_id` key following `10-core.md §5`.
- `record_id = BLAKE3(canonical_json(record_without_signature))`.
- `created_at` is informational; see `10-core.md §1.5`.

### 2.5 Announcement validation and idempotency

A node receiving an announcement MUST validate the record before indexing or
re-announcing it: `(R-TRANS-18)`

- `schema` is exactly `"igc-net/announcement"`.
- `schema_version` is supported by the implementation.
- `record_id` equals `BLAKE3(canonical_json(record_without_signature))`.
- `signature` verifies against the `node_id` public key.
- Hash fields and public keys are lowercase hex in the formats defined by
  `10-core.md`.
- Mode-specific fields obey §2.3.

Invalid announcements MUST be dropped. A node MUST NOT index, re-announce, or
serve because of an invalid announcement.

Announcement processing is idempotent. Re-receiving the same `record_id` MUST
NOT create a second index entry, duplicate tickets, or reset fresher local
state. `(R-TRANS-19)`

### 2.6 Announcement ordering and stale availability

The announce topic is an availability plane, not an authority plane. It does
not provide a global total order for flight state. `created_at` on an
announcement is informational and MUST NOT be used to override governance
records or to decide ownership, deletion, contest status, or the active
publication mode. `(R-TRANS-20)`

When multiple valid announcements for the same `raw_igc_hash` are received, a
node MUST merge locator information by artifact class and serving `node_id`.
The canonical flight identity remains `raw_igc_hash`; announcements do not
create competing flight identities. `(R-TRANS-21)`

If a received announcement conflicts with current governance state, governance
wins. In particular, a stale `public` or `protected` announcement MUST NOT
downgrade a current `private`, `deleted`, `contested`, or `rejected` state.
The node MAY retain the locator as stale availability metadata, but MUST NOT
serve or expose it as currently fetchable in contradiction of governance.
`(R-TRANS-22)`

Tickets are soft locators. A failed fetch, unreachable peer, or stale ticket
MAY cause that locator to be deprioritized or removed, but it MUST NOT delete
the canonical flight identity or override governance state. `(R-TRANS-23)`

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

Before re-announcing a ticket, a node MUST verify that the artifact bytes it
serves or references match the advertised hash for the artifact class. For
public raw IGC this means `BLAKE3(raw_igc_bytes) == raw_igc_hash`; for the
protected sanitized artifact this means `BLAKE3(sanitized_igc_bytes) ==
protected_hash`; for protected raw companion and private raw IGC this means
the raw bytes hash to `raw_igc_hash`. `(R-TRANS-24)`

A node MUST NOT re-announce a ticket for an artifact class that the current
governance state forbids it to serve. `(R-TRANS-25)` If governance state is
unknown or stale for the affected `raw_igc_hash`, the node MAY keep the
artifact locally but MUST NOT create new availability claims for it until the
relevant governance state is current.

---

## 7. Fetch rules by mode

The requester identifies the desired artifact class. The serving node MUST
evaluate the requested class against the current effective `publication_mode`
and governance state before transmitting bytes. The artifact-class serving
matrix is defined in `20-artifacts.md §1.1`.

### 7.1 Public artifact

- Any node MAY fetch without authorization.
- The serving node MUST check governance state before transmitting. `(R-TRANS-12)`
- If the flight is in `contested` or `rejected` governance state, the
  serving node MUST refuse the fetch regardless of the requester's
  identity.

### 7.2 Protected sanitized artifact

- Any node MAY fetch the sanitized artifact without authorization.
- Same governance-state pre-check as §7.1 applies.

### 7.3 Non-public artifacts (protected raw companion and private raw IGC)

Both the protected raw companion and private raw IGC require the same
authorization:

- The requester MUST prove possession of the pilot's
  `private_access_keypair` by signing the fetch request with the
  corresponding private key. `(R-TRANS-06)`
- The serving node MUST verify this signature against the pilot's current
  `private_access_public_key` (see `60-keys-and-access.md §4`, §6) before
  transmitting. `(R-TRANS-07)`
- The signed fetch request MUST bind the requested artifact class; see
  `60-keys-and-access.md §4.3`.
- Governance-state pre-check applies.
- Bytes are transmitted as plaintext over iroh; see `10-core.md §1.3`.

### 7.5 Governance state pre-check

Before transmitting any artifact, the serving node MUST check the current
governance state for the `raw_igc_hash`:

- `contested` governance state: MUST refuse all fetches. `(R-TRANS-08)`
- `rejected` governance state: MUST refuse all fetches. `(R-TRANS-09)`
- valid deletion request processed: MUST refuse all fetches. `(R-TRANS-26)`
- `approved` or no claim: proceed with mode-appropriate fetch rules.
- stale or incomplete governance state for the requested hash: MUST refuse
  fetches for that hash until catch-up succeeds.

Governance state takes precedence over key possession. A requester who
holds a valid `private_access_keypair` but requests a flight in
`contested` or `rejected` state MUST be refused.

---

## 8. Catch-up for public content discovery

A node that was offline and then rejoins the network MUST be able to
discover public flights it missed.

Implementations MUST support at least one of: `(R-TRANS-10)`

- A gossip history window of sufficient depth. The default public discovery
  catch-up window is 3 days. Implementations MAY expose a longer configurable
  window through node configuration. `(R-TRANS-16)`
- A pull-based sync endpoint exposed by serving nodes, allowing a rejoining
  node to fetch announcement records by time range.

This requirement applies to public content discovery on the data-plane
announce topic. Governance catch-up requirements are defined in
`55-governance-sync.md`.

The public discovery catch-up window is not a complete governance authority.
A node MUST NOT treat a 3-day announcement catch-up as proof that older
deletion requests, challenges, rejections, restrictive mode changes, or
private-access rotations do not exist. Serving decisions that depend on
governance state require the governance baseline and catch-up rules in
`55-governance-sync.md`. `(R-TRANS-17)`
