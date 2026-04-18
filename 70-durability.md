# igc-net v0.2 — Durability

**Status:** Normative  
**Depends on:** `10-core.md`, `20-artifacts.md`, `50-governance.md`,
`60-keys-and-access.md`

---

## 1. Durability by publication mode

| Mode | Default replication | Credential required for non-public durability |
|------|--------------------|-----------------------------------------------|
| `public` | Free; any node may replicate without pilot approval | — |
| `protected` | Sanitized artifact: free. Raw companion: requires fetch authorization via the pilot's `private_access_keypair` | `private_access_keypair` |
| `private` | Not replicated by default; explicit pilot approval required | `private_access_keypair` |

### 1.1 Public and protected flights

Public and protected flights may be replicated by any node without pilot
approval. `(R-DUR-09)`

For protected flights, the replicated public artifact is the sanitized
`.igc` file. It excludes personal data. The raw companion is plaintext at
rest; a replicating node that wants to serve the raw companion MUST hold
the pilot's `private_access_keypair` and be prepared to verify incoming
fetch-request signatures before delivery.

Replication of public artifacts is desirable for durability and resilience.
Replication of protected raw companions is opt-in by the pilot through the
key grant; without the key, a node can neither fetch an authoritative copy
nor authorize subsequent deliveries.

### 1.2 Private flights

Private raw IGC bytes are NOT replicated by other nodes by default. `(R-DUR-10)`

If a pilot wants archival durability for private flights, the pilot MUST
explicitly approve a durability node by granting it the
`private_access_keypair` through the flow in `60-keys-and-access.md §5`.

A durability node that holds `private_access_keypair`:

- Stores plaintext private raw IGC bytes and plaintext private metadata
  records it has fetched. No protocol-level content encryption is applied;
  iroh's transport encryption protects bytes in flight only.
- Keeps that plaintext content confidential as a **compliance and legal
  obligation** on its terms of service. The protocol does not enforce
  confidentiality at rest cryptographically.
- MAY sign new fetch requests as long as it holds a valid, un-revoked
  keypair matching the pilot's current `private-access-rotation-record`.

### 1.3 Archive is an operational role

Archive is an operational role, not a protocol class or conformance
profile. There is no `archive` conformance tier. A node acting as an
archive is an ordinary private-access node (Category 2) that has been
granted the pilot's `private_access_keypair` and has agreed to operational
and legal obligations through its terms of service.

---

## 2. Compliant-node obligations after governance events

### 2.1 After receiving a valid deletion request

1. Stop serving all artifacts (public, protected, private) for the
   referenced `raw_igc_hash` immediately. `(R-DUR-01)`
2. Within 30 days: `(R-DUR-02)`
   - Remove `igc-metadata` records for this `raw_igc_hash`.
   - Remove `flight-metadata(scope="flight")` records that reference this
     hash.
   - Remove flight-scoped governance records and indexes for this hash.
   - Remove any derived analytics records generated from this hash.
3. End the display and association of this flight in the pilot's index.

Deletion does **NOT** delete the `pilot-profile` object. `pilot-profile`
is identity-level, not flight-scoped. Ending a flight's association with
the pilot does not erase the pilot's identity or affect their other
flights.

### 2.2 After receiving a mode upgrade record

When a `publication-mode-record` changes the mode to a more restrictive
value (e.g., `public` → `protected`, `protected` → `private`):

- The serving node MUST immediately stop serving the previously permitted
  artifact. `(R-DUR-03)`
- The serving node MUST NOT wait for a cache TTL to expire. `(R-DUR-04)`
- No separate deletion request is required to enforce the upgrade.

### 2.3 After `private_access_keypair` deletion

When a pilot instructs a node to delete the `private_access_keypair`:

1. The node MUST delete the keypair. `(R-DUR-05)`
2. The node MUST stop serving all non-public content of that pilot,
   because it can no longer sign fetch requests to refresh content nor
   verify its own authority to serve it. `(R-DUR-06)`
3. The node MAY retain plaintext bytes it already holds, but SHOULD delete
   them as part of the revocation. Retention without the keypair is a
   compliance matter, not a cryptographic enforcement.
4. If the pilot has also issued a deletion request for the relevant
   `raw_igc_hash`, the node MUST delete the raw IGC bytes entirely.

### 2.4 After processing a `private-access-rotation-record`

When a new `private-access-rotation-record` signed by the pilot arrives on
the governance topic, and the node's currently held
`private_access_keypair` (if any) no longer matches the new
`private_access_public_key`:

- The node MUST stop signing new fetch requests with the obsolete keypair.
- The node MUST stop accepting fetch-request signatures that use the
  obsolete `private_access_public_key` when serving content.
- If the node needs ongoing private access, the pilot MUST grant the new
  keypair through the flow in `60-keys-and-access.md §5`.

### 2.5 After receiving a challenge record

When a valid challenge is received for a `raw_igc_hash`:

- The node MUST freeze further non-public content release for that hash. `(R-DUR-07)`
- The flight enters `contested` state; all fetch requests for restricted
  content MUST be refused until the challenge is resolved. `(R-DUR-08)`

---

## 3. Erasure and GDPR compliance

GDPR Article 17 gives pilots the right to request erasure of their
personal data. Complete distributed deletion is not enforceable in a
decentralised network.

Compliant portals MUST inform pilots that: `(R-DUR-11)`

- Deletion requests are protocol obligations for **compliant nodes only**.
- Non-compliant nodes cannot be cryptographically forced to delete.
- Distributed deletion is best-effort.

A node that has received and processed a valid deletion request and has
complied with the obligations in §2.1 is in full protocol compliance,
regardless of non-compliant peers.

---

## 4. Durability obligations summary

| Event | Immediate obligation | 30-day obligation |
|-------|---------------------|------------------|
| Deletion request received | Stop serving all artifacts for hash | Remove flight-scoped records and indexes |
| Mode upgrade record received | Stop serving previously permitted artifact | — |
| `private_access_keypair` deleted at node | Stop serving non-public content for that pilot | Delete bytes if deletion request also issued |
| `private-access-rotation-record` processed with non-matching key | Stop signing or honoring signatures under the old key | — |
| Challenge record received | Freeze non-public release for hash | — |
