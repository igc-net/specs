# igc-net — Keys and Access

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
   that pilot's non-public IGC content: private raw IGC files and the raw
   companion that backs a protected flight.

The protocol represents private access with a single **`private_access_keypair`**
per pilot. Possession of this keypair is the sole authorization credential for
all pre-v0.5 non-public IGC content of that pilot. There are no separate per-content-type
keys.

igc-net does not perform protocol-level content encryption; see
`10-core.md §1.3`. A serving node that is asked for non-public content MUST
verify a signed fetch request (see §4) before returning the bytes. The network
trusts authorized custodians to keep what they hold private, and uses
signatures to authorize who may fetch.

---

## 2. Node access categories

Every node that is logged in as a particular pilot falls into exactly one of
two categories for that pilot:

### 2.1 Identity-linked node (Category 1)

An identity-linked node has bound a local user account to a `pilot_id` through
a node-local authentication mechanism (password, OAuth, magic link, etc.; the
mechanism itself is outside the protocol scope).

An identity-linked node:

- MAY discover, fetch, store, serve, index, and display `public` and
  `protected` artifacts that belong to that `pilot_id`.
- MUST NOT request or accept the pilot's `private_access_keypair`. `(R-ACCESS-01)`
- MUST NOT fetch private-mode raw IGC bytes for that pilot.
- MUST NOT fetch the raw companion bound to a `protected` flight of that
  pilot.
- MUST NOT treat public metadata advertisements as authorization to fetch
  non-public IGC bytes or portal-hosted protected/private resources.
  `(R-ACCESS-02)`
- MUST NOT display personal-identity fields (name, surname, nationality,
  wing, competitions, etc.) for that pilot unless the portal has a separately
  verified, non-stale `PilotProfileCredential` or another explicit
  pilot-authorized basis outside raw IGC headers.
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
  `private_access_keypair` (private-mode raw IGC and protected raw companion).
- MAY store plaintext private content it has fetched.
- MUST keep stored plaintext private content confidential as a compliance and
  legal obligation of its terms of service. The protocol does not enforce
  confidentiality at rest cryptographically. `(R-ACCESS-04)`

`PilotProfileCredential` is not fetched via `private_access_keypair`. It is an
application-layer credential presented by the pilot or by a pilot-authorized
sync channel.

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
pre-v0.5 non-public IGC content simultaneously. `(R-ACCESS-06)` There is no
per-flight or per-artifact-class sub-scoping.

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
   - Invalid, replayed (`seq_num` ≤ last seen for this `requester_key`), or
     signed by a non-current key: reject the request. `(R-ACCESS-09)`

Key-possession proof does not override governance state.

### 4.3 Fetch request shape

```json
{
  "schema": "igc-net/fetch-request",
  "schema_version": 1,
  "raw_igc_hash": "<blake3-hex>",
  "artifact_class": "protected_raw_companion|private_raw_igc",
  "requester_key": "<ed25519-public-key-hex>",
  "seq_num": 1,
  "signature": "<ed25519-signature-hex>"
}
```

- `requester_key` is the public half of the `private_access_keypair`. It
  MUST match the currently authorized `private_access_public_key` for the
  pilot who owns `raw_igc_hash`. `(R-ACCESS-10)`
- `artifact_class` MUST identify the restricted artifact being requested.
  It MUST be either `protected_raw_companion` or `private_raw_igc`, and it
  MUST match the artifact class the serving endpoint is being asked to
  transmit. A serving node MUST reject a restricted fetch request whose signed
  `artifact_class` does not match the requested artifact class or whose class
  is not allowed by the current `publication_mode`. `(R-ACCESS-28)`
- `seq_num` MUST be a positive integer (≥ 1) strictly greater than the
  serving node's last-accepted `seq_num` for this `requester_key`. The
  serving node MUST persist the last-accepted `seq_num` per `requester_key`
  and MUST reject any request where `seq_num` ≤ `last_seen`. `(R-ACCESS-11)`
  The serving node SHOULD durably store (fsync) the updated `seq_num` before
  transmitting bytes; a node that loses its counter state on restart loses
  replay protection until `seq_num` advances past the lost value.

  **POC limitation:** `seq_num` registers are per-serving-node. A captured
  request with `seq_num = N` can be replayed to a different serving node
  whose last-seen is below `N`. Cross-node `seq_num` state synchronization
  is deferred as post-POC hardening.
- `signature` is the Ed25519 signature over the RFC 8785 canonical JSON of
  all fields except `signature`, signed with the private half of
  `private_access_keypair`.

### 4.4 Transmission

The serving node transmits the requested bytes as plaintext over the iroh
peer-to-peer connection to the requester; see `10-core.md §1.3`.

The serving node MUST NOT transmit any non-public byte to a requester that
has not produced a valid fetch proof with a strictly increasing `seq_num`,
signed by the pilot's currently authorized `private_access_public_key`.
`(R-ACCESS-13)`

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

### 5.4 Portal-mediated igc-net handover

For portal deployments, the portal is the user-facing handover boundary.
The local igc-net service holds the `private_access_keypair` after the grant,
but the initial transfer is carried by the portal that authenticated the pilot
and obtained the pilot's consent.

The portal-mediated handover sequence is:

1. The portal authenticates the pilot and displays the exact access being
   granted: protected raw companion and private raw IGC access for all of
   that pilot's non-public artifacts.
2. The portal obtains the pilot's `private_access_keypair` from the pilot,
   from an existing custody portal, or from a local key-management tool.
3. The portal calls the local igc-net key-provisioning RPC defined in
   `proto/` and transfers the keypair to igc-net over the local RPC channel.
4. igc-net validates that the private key matches the current
   `private_access_public_key` for the pilot, stores the keypair in its
   restricted state store, and becomes Category 2 for that pilot.
5. The portal discards any transient copy of the private key after a
   successful handover. A portal that is not itself an explicit custody
   portal MUST NOT persist the private key. `(R-ACCESS-20)`

After successful handover, restricted fetch signing is igc-net's
responsibility. The portal MUST NOT need to construct private fetch
challenges or sign restricted fetch requests on behalf of igc-net.
`(R-ACCESS-21)`

The key-provisioning RPC is an idempotent local custody operation:

- The request MUST identify the pilot, carry the private half of
  `private_access_keypair`, and carry the portal's expected current
  `private_access_public_key`.
- The private key bytes in the RPC MUST be the 32-byte Ed25519 seed used to
  generate the keypair, as defined in `10-core.md §1.2`. The expected public
  key MUST be the corresponding 32-byte compressed Ed25519 public key encoded
  as lowercase hex.
- igc-net MUST derive the public key from the supplied private key and reject
  the request without mutating stored key state if the derived key does not
  match the request's expected public key. `(R-ACCESS-23)`
- igc-net MUST compare the expected public key with the latest active
  `private-access-rotation-record` known for the pilot. If no active record is
  known, or if governance state is too stale to decide, igc-net MUST reject the
  request without mutating stored key state. `(R-ACCESS-24)`
- If the expected public key is not the current active key in governance state,
  igc-net MUST reject the request as rotated or stale and MUST NOT retain the
  supplied private key. `(R-ACCESS-25)`
- Re-provisioning the same key for the same pilot MUST be treated as success.
  Provisioning a different key for the same pilot is allowed only when that key
  matches the current active rotation record; replacement MUST be atomic, and
  the old private key MUST be deleted before the operation is reported as
  successful. `(R-ACCESS-26)`

Successful provisioning means igc-net has accepted custody of the key. It does
not by itself mean restricted serving is ready. Before signing restricted fetch
requests for that pilot, igc-net MUST satisfy the governance catch-up and
baseline requirements in `55-governance-sync.md §5`. `(R-ACCESS-27)`

The normative gRPC surface for this handover is `ProvisionPrivateAccessKey` in
`proto/igc_net_v0.proto`. Failure modes are:

| Condition | Required failure | Error reason |
|-----------|------------------|--------------|
| Malformed `pilot_id`, malformed key bytes, or key/public-key mismatch | reject as invalid argument | `ERROR_REASON_INVALID_ARGUMENT` |
| Local caller is not authorized to provision keys into this igc-net instance | reject as unauthorized | `ERROR_REASON_UNAUTHORIZED` |
| No active private-access rotation record is known for the pilot | reject as missing active authorization key | `ERROR_REASON_MISSING_ACTIVE_PRIVATE_ACCESS_RECORD` |
| Governance state is stale | reject as governance stale | `ERROR_REASON_GOVERNANCE_STALE` |
| No durable governance baseline exists | reject as governance stale | `ERROR_REASON_MISSING_GOVERNANCE_BASELINE` |
| Expected public key is older than the active rotation record | reject as rotated key | `ERROR_REASON_ROTATED_KEY` |
| State-store write, fsync, or atomic replacement fails | reject as internal failure and do not report success | `ERROR_REASON_INTERNAL` |

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

- Signed by `pilot_id` following `10-core.md §5`.
- `record_id = BLAKE3(canonical_json(record_without_signature))`.
- `supersedes` MUST reference the `record_id` of the previous active
  `private-access-rotation-record` for this `pilot_id`, or `null` for the
  initial record.
- `created_at` is informational; `supersedes` is the ordering authority
  (see `10-core.md §1.5`).

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
- The node MUST delete locally held protected raw companion and private raw
  IGC plaintext for that pilot unless a separate active durability custody
  grant explicitly authorizes retention. `(R-ACCESS-17)`
- The node SHOULD retain nothing after revocation. If retaining nothing would
  permit accidental access resurrection, the node MAY keep only the minimum
  non-secret tombstone needed to remember that access was revoked.
  `(R-ACCESS-22)`

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

The transport requirements are defined in `10-core.md §1.3`. `(R-ACCESS-19)`

The fetch-request signature check (§4) is a content-authorization check,
not a transport-authentication check. Both are required: the transport
authenticates the peer endpoint; the fetch-request signature authenticates
the pilot-level authority to receive the bytes.
