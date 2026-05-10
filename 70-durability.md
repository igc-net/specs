# igc-net — Durability

**Status:** Normative  
**Depends on:** `10-core.md`, `20-artifacts.md`, `50-governance.md`,
`60-keys-and-access.md`

See also: `75-groups-and-social.md §2.6` for group membership record obligations.

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

- Stores plaintext private raw IGC bytes it has fetched. No protocol-level
  content encryption is applied; iroh's transport encryption protects bytes in
  flight only.
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

The normative obligations are defined in the originating documents. This
section provides cross-references for each trigger event.

### 2.1 After receiving a valid deletion request

See `50-governance.md §13.2`. Stop serving immediately `(R-DUR-01)`;
remove flight-scoped records and indexes within 30 days `(R-DUR-02)`.
Deletion does not affect the pilot's identity-level profile authority.

### 2.2 After receiving a mode upgrade record

See `20-artifacts.md §7.1`. Stop serving the previously permitted artifact
immediately `(R-DUR-03)`; MUST NOT wait for a cache TTL `(R-DUR-04)`.

### 2.3 After `private_access_keypair` deletion

See `60-keys-and-access.md §7`. Delete the keypair `(R-DUR-05)` and stop
serving all non-public content for that pilot `(R-DUR-06)`. Also delete
locally held protected raw companion and private raw IGC plaintext unless a
separate active durability custody grant explicitly authorizes retention.
`(R-DUR-12)`

The node MAY retain only non-secret tombstone state needed to avoid accidental
access resurrection. That tombstone MUST NOT contain restricted plaintext,
private key material, or a locator that the node continues to expose as
fetchable. `(R-DUR-13)`

Physical erasure across every local storage backend is a best-effort
obligation, not a provable protocol guarantee. An implementation MUST NOT claim
cryptographically or mechanically provable erasure of all restricted plaintext
copies. It MUST stop serving the restricted artifact, remove any plaintext copy
from storage layers it explicitly manages where deletion is supported, and
ensure retained tombstones do not expose the content as fetchable.
`(R-DUR-14)`

### 2.4 After processing a `private-access-rotation-record`

See `60-keys-and-access.md §6.2`. Stop signing or accepting fetch-request
signatures under the obsolete keypair immediately.

### 2.5 After receiving a challenge record

See `50-governance.md §7.2`. Freeze non-public content release for the hash
`(R-DUR-07)`; refuse all restricted fetch requests until resolved
`(R-DUR-08)`.

### 2.6 Group membership records and flight deletion

Group membership records (`75-groups-and-social.md §4`) and follow records are
not flight-scoped. A valid deletion request for a `raw_igc_hash` does not
require those records to be removed. `(R-DUR-15)`

After a deletion request for a `raw_igc_hash` is processed, a compliant node
MUST refuse any subsequent group-based fetch request for that hash. The raw IGC
bytes are no longer held; the governance-state check in
`75-groups-and-social.md §5.3` enforces this by returning `not_found` once the
bytes are purged. `(R-DUR-16)`

Group membership records and follow records constitute personal data. A node
SHOULD remove a pilot's own group membership records and follow records within
30 days of a pilot-initiated personal-data erasure request. The igc-net
governance-topic deletion mechanism covers flight-scoped records only; personal
social records require out-of-band erasure coordination between the pilot and
the portal. `(R-DUR-17)`

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
| `private_access_keypair` deleted at node | Stop serving non-public content for that pilot; delete key material | Delete restricted plaintext unless a separate active durability custody grant authorizes retention |
| `private-access-rotation-record` processed with non-matching key | Stop signing or honoring signatures under the old key | — |
| Challenge record received | Freeze non-public release for hash | — |
| Deletion request received for `raw_igc_hash` | Refuse subsequent group-based fetch requests for that hash | — |
| Pilot-initiated personal-data erasure request | (no protocol mechanism) | Remove pilot's group membership and follow records |
