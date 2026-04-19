# igc-net — Durability

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
serving all non-public content for that pilot `(R-DUR-06)`.

### 2.4 After processing a `private-access-rotation-record`

See `60-keys-and-access.md §6.2`. Stop signing or accepting fetch-request
signatures under the obsolete keypair immediately.

### 2.5 After receiving a challenge record

See `50-governance.md §7.2`. Freeze non-public content release for the hash
`(R-DUR-07)`; refuse all restricted fetch requests until resolved
`(R-DUR-08)`.

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
