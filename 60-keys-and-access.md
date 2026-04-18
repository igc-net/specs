# igc-net v0.2 — Keys and Access

**Status:** Normative  
**Depends on:** `10-core.md`, `20-artifacts.md`, `40-pilot-and-metadata.md`,
`50-governance.md`, `55-governance-sync.md`

---

## 1. Access model overview

igc-net distinguishes two independent access concerns:

1. **Identity linkage** — a node knows which `pilot_id` a local user account
   corresponds to, and can therefore discover, serve, and group that pilot's
   `public` and `protected` artifacts under the right identity.
2. **Private access** — a node holds a key that lets it authorize fetches for
   that pilot's non-public content: private IGC files, the raw companion that
   backs a protected flight, and any metadata record whose `visibility` is
   `"private"` (including `igc-metadata`).

The protocol represents private access with a single **`private_access_keypair`**
per pilot. Possession of this keypair is the sole authorization credential for
all non-public content of that pilot. There are no separate per-content-type
keys.

igc-net does NOT perform protocol-level content encryption in v0.2. Private
content is served as plaintext bytes over iroh's end-to-end encrypted
peer-to-peer transport. A serving node that is asked for non-public content
MUST verify a signed fetch request (see §4) before returning the bytes.
Confidentiality at the serving node is a **compliance and legal obligation**
on the node, not a cryptographic protocol property. This is the same trust
model igc-net applies to archive nodes: the network trusts authorized
custodians to keep what they hold private, and uses signatures to authorize
who may fetch.

---

## 2. Node access categories

Every node that is logged in as a particular pilot falls into exactly one of
two categories for that pilot:

### 2.1 Identity-linked node (Category 1)

An identity-linked node has bound a local user account to a `pilot_id` through
a node-local authentication mechanism (password, OAuth, magic link, etc.; the
mechanism itself is outside the v0.2 protocol scope).

An identity-linked node:

- MAY discover, fetch, store, serve, index, and display `public` and
  `protected` artifacts that belong to that `pilot_id`.
- MUST NOT request or accept the pilot's `private_access_keypair`. `(R-ACCESS-01)`
- MUST NOT fetch private-mode raw IGC bytes for that pilot.
- MUST NOT fetch the raw companion bound to a `protected` flight of that
  pilot.
- MUST NOT fetch, decrypt, store, index, or display any metadata record with
  `visibility: "private"` belonging to that pilot (`pilot-profile`,
  `flight-metadata`, `igc-metadata`). `(R-ACCESS-02)`
- MUST NOT display personal-identity fields (name, surname, nationality,
  wing, competitions, etc.) for that pilot unless the pilot themselves has
  published a `visibility: "public"` record carrying those fields.
- SHOULD NOT extract, index, or otherwise harvest personal-identity fields
  from the headers of `public` IGC files it serves, outside records the
  pilot has explicitly authorized as public. `(R-ACCESS-03)` This obligation
  cannot be enforced cryptographically; operators accept it as part of
  running a compliant node.

### 2.2 Private-access node (Category 2)

A private-access node is an identity-linked node that the pilot has also
granted the `private_access_keypair` to (see §5).

A private-access node:

- Has every capability of an identity-linked node.
- MAY sign fetch requests for private content of that pilot using the
  `private_access_keypair` (private-mode raw IGC, protected raw companion,
  private-visibility `pilot-profile`, private-visibility `flight-metadata`,
  `igc-metadata`).
- MAY store plaintext private content it has fetched.
- MUST keep stored plaintext private content confidential as a compliance and
  legal obligation of its terms of service. The protocol does not enforce
  confidentiality at rest cryptographically. `(R-ACCESS-04)`

A node operator declares the category at which the node operates (per pilot)
through the node-local login / grant UX described in the operator guides.

---

## 3. The `private_access_keypair`

### 3.1 Nature of the keypair

- `private_access_keypair` is an ordinary Ed25519 keypair.
- It is generated once by the pilot's first full-custody portal (or by the
  pilot directly, using any compliant tool).
- The keypair is distinct from `pilot_id`, `node_id`, and `resolver_id`. `(R-ACCESS-05)`
- There is no deterministic derivation from `master_secret`, `pilot_pin`, or
  any other password-based input. The keypair is itself the credential the
  pilot backs up.
- The public half is published via a `private-access-rotation-record` (see
  §6) so that serving nodes know which public key to verify fetch-request
  signatures against.

### 3.2 One keypair, all private content

One `private_access_keypair` grants or revokes access to **all** of a pilot's
non-public content simultaneously. `(R-ACCESS-06)` There is no per-flight,
per-record-type, or per-visibility sub-scoping in v0.2.

### 3.3 Key backup

The pilot MUST back up the private half of `private_access_keypair` to a
location independent of any portal (offline backup, password manager,
hardware key, etc.). Loss of the keypair makes all private content
permanently unreachable through the protocol.

Because no deterministic re-derivation is possible, a lost keypair cannot be
regenerated. The pilot MAY generate a replacement and publish a rotation
record (§6); this gives future authorization to the new key but does not
restore reachability of content held only at nodes that have already deleted
the old key.

---

## 4. Fetch authorization

### 4.1 Authorization model

Authorization for non-public content is key-possession-based. No
gossip-propagated grant record is used. The requester signs a fetch request
(§4.3) with the `private_access_keypair` private half. The serving node
verifies the signature using the public half published in the pilot's
current `private-access-rotation-record` (§6) on the governance topic.

Governance state takes precedence. For every fetch, including fetches for
which no authorization is required (e.g., `public` mode), the serving node
MUST check the current governance state for the `raw_igc_hash` first:

- `contested` or `rejected`: MUST refuse regardless of key possession. `(R-ACCESS-07)`
- Otherwise: proceed to §4.2.

### 4.2 Authorization sequence for non-public content

1. **Governance check first** (above).
2. **Signature verification**: the serving node verifies the requester's
   Ed25519 signature over the fetch request using the pilot's current
   authorized `private_access_public_key` from the most recent
   `private-access-rotation-record`. `(R-ACCESS-08)`
   - Valid signature: transmit the requested plaintext bytes.
   - Invalid, expired, replayed, or signed by a non-current key: reject the
     request. `(R-ACCESS-09)`

Key-possession proof does not override governance state.

### 4.3 Fetch request shape

```json
{
  "schema": "igc-net/fetch-request",
  "schema_version": 1,
  "raw_igc_hash": "<blake3-hex>",
  "requester_key": "<ed25519-public-key-hex>",
  "nonce": "<32-byte-random-hex>",
  "expires_at": "YYYY-MM-DDTHH:MM:SSZ",
  "signature": "<ed25519-signature-hex>"
}
```

- `requester_key` is the public half of the `private_access_keypair`. It
  MUST match the currently authorized `private_access_public_key` for the
  pilot who owns `raw_igc_hash`. `(R-ACCESS-10)`
- `nonce` is 32 bytes of random data encoded as 64-character lowercase hex.
  The serving node MUST reject duplicate
  `(raw_igc_hash, requester_key, nonce)` triples seen within the last
  5 minutes. `(R-ACCESS-11)`
- `expires_at` MUST be at most 5 minutes after the request is created. The
  serving node MUST reject expired requests. `(R-ACCESS-12)`
- `signature` is the Ed25519 signature over the RFC 8785 canonical JSON of
  all fields except `signature`, signed with the private half of
  `private_access_keypair`.

### 4.4 Transmission

The serving node transmits the requested bytes as plaintext over the iroh
peer-to-peer connection to the requester. Iroh provides end-to-end encryption
on the transport layer; no additional protocol-layer encryption is applied to
the content.

The serving node MUST NOT transmit any non-public byte to a requester that
has not produced a valid, non-expired, non-replayed fetch request signed by
the pilot's currently authorized `private_access_public_key`. `(R-ACCESS-13)`

No grant record is presented by the requester. No gossip record is consulted
for access authorization other than the current
`private-access-rotation-record` and the current governance state.

---

## 5. Granting private access to a node

This is the flow a pilot follows to upgrade an identity-linked node
(Category 1) to a private-access node (Category 2).

### 5.1 Flow (RECOMMENDED)

1. **Initiate**: Pilot triggers the grant at the requesting node. The node
   generates an ephemeral Ed25519 exchange keypair and constructs a compact
   exchange token containing its `node_id`, the ephemeral exchange public
   key, a nonce, and an expiry timestamp.
2. **Pilot redirected**: Pilot scans a QR code or follows a URL to a portal
   where they hold the `private_access_keypair` (typically the node they
   originally generated it on, or a manual key management tool).
3. **Approval**: The granting portal authenticates the pilot, displays the
   requesting node's identity and a human-readable description, and the
   pilot confirms.
4. **Key transfer**: The granting portal transmits the
   `private_access_keypair` private half directly to the requesting node's
   `node_id` endpoint over iroh. Iroh's transport encryption protects the
   keypair in flight. No additional sealing envelope is applied at the
   protocol layer; the transport cipher is the single confidentiality layer.
5. **Receipt**: The requesting node stores the received keypair, discards
   the ephemeral exchange keypair, and advances to Category 2 for that
   pilot.

### 5.2 Conformance note

Implementations that do not support this flow remain conformant. Manual
out-of-band key provisioning by the pilot (e.g., entering the key bytes at
the new node) is valid.

### 5.3 No gossip trace

No `access-grant` gossip record is emitted at any step of §5.1. The key
transfer is direct, node-to-node, and leaves no protocol-plane trace.

---

## 6. Authorization key publication and rotation

Serving nodes must know which public key to verify fetch signatures against.
The pilot publishes this authorization by emitting a
`private-access-rotation-record` on the governance topic.

### 6.1 Record shape

```json
{
  "schema": "igc-net/private-access-rotation-record",
  "schema_version": 1,
  "record_id": "<blake3-of-canonical-record-without-signature>",
  "pilot_id": "igcnet:id:<root-public-key-hex>",
  "private_access_public_key": "<ed25519-public-key-hex>",
  "supersedes": "<previous-record-id-or-null>",
  "created_at": "YYYY-MM-DDTHH:MM:SSZ",
  "signature": "<ed25519-signature-hex>"
}
```

- Signed directly by the pilot's root identity key (`pilot_id`).
- `record_id = BLAKE3(canonical_json(record_without_signature))`.
- `supersedes` MUST reference the `record_id` of the previous active
  `private-access-rotation-record` for this `pilot_id`, or `null` for the
  initial record.
- `created_at` is informational; `supersedes` is the ordering authority.

### 6.2 Initial record and rotation semantics

- A pilot MUST publish an initial `private-access-rotation-record` before
  signing any fetch request. `(R-ACCESS-14)`
- To rotate the keypair, the pilot generates a new `private_access_keypair`
  and publishes a new record referencing the previous record's `record_id`
  in `supersedes`.
- Serving nodes MUST accept a fetch request only if `requester_key` matches
  the `private_access_public_key` of the **most recent active** record in
  the rotation chain. `(R-ACCESS-15)`
- A rotation invalidates future authorization by the previous key. It does
  not itself remove plaintext content already held at private-access nodes;
  see §7.

### 6.3 Propagation

`private-access-rotation-record` is a governance-plane record. It travels on
the governance topic alongside claims, approvals, and mode records. See
`55-governance-sync.md §2`.

---

## 7. Revocation

Access revocation is a **local action** at the node holding the
`private_access_keypair`. `(R-ACCESS-16)`

The pilot instructs the node to delete the keypair. The node has a trust and
legal obligation to comply.

Effects:

- After key deletion, the node can no longer produce valid fetch-request
  signatures for any non-public content of that pilot.
- Other serving nodes stop receiving valid fetch requests from the revoked
  node and therefore stop delivering new content to it.
- Any locally held plaintext private content SHOULD also be deleted; this is
  a node compliance obligation, not a cryptographic enforcement. `(R-ACCESS-17)`

If the pilot wants to deny authority to a compromised or untrusted node
whose cooperation cannot be assumed, the pilot MUST rotate the keypair
(§6.2). Rotation prevents the old key from producing valid fetch requests
at any compliant serving node, regardless of whether the node voluntarily
deletes it.

No revocation record is emitted. There is no revocation gossip record type;
rotation plus local deletion cover the required cases.

---

## 8. Durability custody and private access

A node that stores a pilot's private content for durability MUST hold the
`private_access_keypair`. That key is the only credential that grants fetch
authorization for private content, and a durability node typically needs to
re-verify or refresh content over time.

A durability node holding the keypair stores plaintext private content. The
durability node's obligations:

- Keep stored plaintext confidential as a compliance and legal obligation on
  its terms of service. `(R-ACCESS-18)`
- Stop serving the pilot's private content after receiving a key deletion
  instruction from the pilot.
- Stop serving the pilot's private content after processing any rotation
  record the pilot publishes that it cannot obtain the new keypair for.

Durability is an operational role, not a separate conformance class. Any
Category 2 node that has agreed to a durability arrangement is a durability
node.

---

## 9. Interaction with iroh transport

igc-net v0.2 assumes iroh (or an equivalent transport) that:

- Authenticates peers by their `node_id` public key.
- Encrypts all peer-to-peer traffic end-to-end.
- Provides forward secrecy on the transport layer.

The protocol's no-protocol-level-content-encryption position (§1, §4.4) is
conditional on this property. An implementation that uses a transport
without end-to-end encryption is NOT conformant with v0.2. `(R-ACCESS-19)`

The fetch-request signature check (§4) is a content-authorization check,
not a transport-authentication check. Both are required: the transport
authenticates the peer endpoint; the fetch-request signature authenticates
the pilot-level authority to receive the bytes.
