# igc-net — Threat Model

**Status:** Normative (adversary assumptions) / Informative (analysis)

**Depends on:** `00-overview.md`, `10-core.md`, `20-artifacts.md`,
`30-transport.md`, `40-pilot-and-metadata.md`, `50-governance.md`,
`55-governance-sync.md`, `60-keys-and-access.md`,
`65-pilot-auth-did.md`, `70-durability.md`

---

## 1. Scope and assumptions

This document covers protocol-level threats: attacks against the
cryptographic primitives, the record and transport planes, the
credential model, and the governance workflow. Deployment-layer risks
(node operator OpSec, regulator takedown, domain hijack, insider abuse
at a portal) are out of scope except where the protocol provides an
explicit lever that binds operator behaviour.

Abuse patterns that misuse the protocol within its rules (spam, soft coercion,
sybil accounts, legal-takedown churn, portal impersonation pressure) are
analysed in this document's abuse section (§10).

Trust boundaries:

- **Pilot ↔ node.** The pilot authenticates to a node through a
  node-local mechanism (outside protocol scope); the protocol
  contributes key-possession proofs and signed governance records.
- **Node ↔ node.** Peers authenticate each other through iroh
  `node_id` keys on the transport layer; protocol-plane signatures
  authenticate records end-to-end independent of the transporting
  peers.
- **Node ↔ resolver.** Nodes accept resolver-signed governance
  records only from resolvers whose `resolver_id` they have locally
  trusted.
- **Operator ↔ regulator.** At-rest confidentiality of plaintext
  private content is a compliance and legal obligation on the node
  operator (see `60-keys-and-access.md §1`). The protocol does not
  enforce it cryptographically.

### 1.1 Credentials at a glance

Three Ed25519 credential classes are held by a pilot:

- `pilot_id` — root identity key. Signs all pilot-authored governance
  records. Does not rotate.
- `pilot_auth_did` — wallet-held authentication / VC-issuer credential.
  Rotatable via `pilot-auth-did-record` (see `65-pilot-auth-did.md`).
- `private_access_keypair` — authorization credential for private
  content. Rotatable via `private-access-rotation-record` (see
  `60-keys-and-access.md §6`).

### 1.2 DID boundary

This threat model assumes the identity boundary defined throughout this
specification:

- `pilot_auth_did` remains a wallet-held `did:key`
- optional `did:web` is permitted only for portal-issued/public-facing issuer
  identity and for non-authoritative public pilot aliases or mirrors
- `did:web` does not replace `pilot_id`
- `did:web` does not override the authoritative
  `pilot_id` -> `pilot-auth-did-record` binding

---

## 2. Adversary model

### 2.1 Adversary classes

- **Passive network observer.** Sees iroh-encrypted traffic between
  peers. Cannot break iroh's transport encryption. Can observe topic
  subscriptions at the iroh layer and infer flight-existence signals
  from governance-topic gossip.
- **Malicious serving node.** A node that misattributes records,
  lies about publication state, serves corrupted artifacts, ignores
  deletion requests, or selectively withholds records from peers.
- **Malicious resolver.** A resolver that issues bogus approvals,
  challenges, resolutions, or identity-recovery records.
- **Credential-compromise adversary.** An attacker who has obtained
  the private half of one of the pilot's three credentials
  (independently per credential).
- **Scrape-only adversary.** A passive harvester of personal-identity
  fields from public IGC headers and public records.
- **Content-tampering adversary.** An attacker who modifies raw IGC
  bytes, sanitized artifacts, or governance records in transit or at
  rest.

### 2.2 Capability bounds under the protocol invariants

- Ed25519 is treated as unforgeable under EUF-CMA (RFC 8032).
- BLAKE3 is treated as collision-resistant and pre-image resistant
  (per current cryptographic consensus).
- RFC 8785 canonical JSON is assumed to produce a unique byte
  serialisation for any given JSON object.
- Iroh's transport is assumed to provide end-to-end authenticated
  encryption with forward secrecy between `node_id` endpoints
  (required by `10-core.md §1.3` / `R-CORE-04`).

An attacker that breaks any of these assumptions is outside the threat
model addressed by this specification.

---

## 3. Credential compromise

Each of the three pilot credentials has a distinct blast radius and
recovery path. The bounds below depend critically on the identity
split: `pilot_auth_did` is additive and does NOT sign governance records.

### 3.1 `pilot_id` compromise

- **Blast radius.** Total control over the pilot's record plane: the
  attacker can forge claims, publication-mode records, deletion
  requests, `private-access-rotation-record`, and
  `pilot-auth-did-record`. The attacker can rotate the pilot out of
  their own `pilot_auth_did` and `private_access_keypair` by signing
  new rotation records.
- **Recovery path.** Resolver-assisted identity recovery (see
  `50-governance.md §10`). A trusted resolver issues an
  `identity-recovery` record that binds the compromised `pilot_id` to
  a new `pilot_id`. Compliant nodes transfer ownership to the new
  identity.
- **Mitigations.** Pilots MUST back up `pilot_id` to an independent
  location (offline store, hardware key). `(R-THREAT-01)` Pilots
  SHOULD prefer hardware key storage where available. Resolvers MUST
  authenticate identity-recovery requests through out-of-band channels
  before issuing the record; compliant nodes SHOULD only honour
  identity-recovery records from resolvers in their trusted set.
  `(R-THREAT-02)`
- **Aftermath.** Historical records signed by the compromised
  `pilot_id` retain their integrity (the signatures are still valid)
  but authority for future actions transfers to the new `pilot_id`
  once the recovery record propagates.

### 3.2 `pilot_auth_did` compromise

- **Blast radius.** The attacker can issue forged
  `PilotProfileCredential` JWTs and forged cross-site authentication
  tokens. The attacker CANNOT sign governance records, CANNOT
  authorize fetches for private content, and CANNOT rotate
  `pilot_auth_did` (rotation requires `pilot_id`). The bound is a
  direct consequence of that credential separation.
- **Recovery path.** The pilot publishes a `pilot-auth-did-record`
  superseding the compromised DID, signed by `pilot_id`. See
  `65-pilot-auth-did.md §5`.
- **Mitigations.** Wallet security is the pilot's responsibility.
  Relying parties MUST consult the `pilot-auth-did-record` chain
  before accepting a VC (see §8 and `R-AUTH-12`). Short VC expiry
  times reduce replay windows. `(R-THREAT-03)`
- **Aftermath.** Portals SHOULD invalidate sessions authenticated by
  the compromised `pilot_auth_did` as soon as the rotation record is
  authoritative on that portal.

### 3.3 `private_access_keypair` compromise

- **Blast radius.** The attacker can sign fetch requests for the
  pilot's private raw IGC, the raw companion of protected flights,
  and private native metadata (`flight-metadata`, `igc-metadata`). The
  attacker CANNOT sign governance records,
  CANNOT issue VCs, and CANNOT cause publication-mode or deletion
  changes.
- **Recovery path.** The pilot publishes a
  `private-access-rotation-record` superseding the compromised key,
  signed by `pilot_id`. Serving nodes thereafter reject fetch requests
  signed by the old key (`R-ACCESS-15`).
- **Mitigations.** Node-category model limits key spread: Category 1
  nodes MUST NOT hold `private_access_keypair` (`R-ACCESS-01`).
  `(R-THREAT-04)` The pilot SHOULD limit the number of Category 2
  nodes.
- **Aftermath.** Plaintext private content that a compromised node
  already fetched remains outside protocol control; its
  confidentiality is an operator and legal obligation.

### 3.4 Multi-credential compromise

If `pilot_id` is compromised in combination with any other credential,
the `pilot_id` compromise subsumes the others: identity recovery (§3.1)
re-issues all three credentials under the new identity. Combined
compromises of `pilot_auth_did` + `private_access_keypair` without
`pilot_id` compromise leave governance authority intact; the pilot
rotates both credentials using `pilot_id` as signer.

---

## 4. Record-layer threats

- **Signature forgery.** Infeasible under the Ed25519 assumption.
- **Replay of governance records.** Mitigated by (a) supersession
  chains (authoritative record is the chain tip), (b) `record_id`
  uniqueness (re-transmitting a known record is a no-op), (c)
  governance-topic subscription semantics (late-joining nodes
  catch up through ordered fetch, not through re-accepting stale
  records as "new").
- **Record withholding.** A malicious serving node may refuse to
  propagate specific records. Mitigated by gossip redundancy and by
  catch-up fetches against alternate peers. A single node cannot
  suppress a record that reached the governance topic's fan-out.
- **Record backdating.** `created_at` is informational; authority is
  the supersession chain, not timestamps (`R-CORE` convention and
  invariant #9 in `00-overview.md §10`). Implementations MUST NOT use
  `created_at` as a supersession tiebreak. `(R-THREAT-05)`
- **Supersession forking.** Concurrent well-formed records that each
  claim to supersede the same parent are resolved by lexicographic
  `record_id` tiebreak (`R-AUTH-07` for `pilot-auth-did-record`;
  analogous rules in `50-governance.md` and `60-keys-and-access.md`).
  The tiebreak is deterministic across implementations.
- **Canonical JSON ambiguity.** Mitigated by strict RFC 8785
  enforcement. Implementations MUST reject records whose canonical
  form does not reproduce the expected `record_id` or signature.
  `(R-THREAT-06)`

---

## 5. Artifact-layer threats

- **Raw IGC tampering.** Detected by `raw_igc_hash`: any
  modification of the raw bytes produces a different hash and
  therefore refers to a different flight.
- **Sanitized artifact tampering.** Detected by the sanitized-artifact
  integrity binding in `20-artifacts.md §5`: the pilot signs over
  (`raw_igc_hash`, `protected_hash`), so a substituted sanitized
  artifact is rejected.
- **Identity leakage through public IGC headers.** The sanitization
  algorithm redacts identity fields in the public artifact served for
  `protected` flights. For `public` flights, headers remain intact by
  design. This gap is addressed by the scrape-avoidance obligation
  (`R-ACCESS-03`) and the protected-flight identity-display rule
  (`R-META-15`) rather than a cryptographic control.
- **Ticket spoofing / fetch-auth bypass.** Mitigated by serving-node
  verification of fetch-request signatures against the current
  `private_access_public_key` (`R-ACCESS-08`–`R-ACCESS-13`). A ticket
  alone is not authorization.

---

## 6. Transport-layer threats

- **Peer confidentiality.** Provided by iroh's end-to-end encrypted
  transport (`R-CORE-04`). Plaintext content is never transmitted over
  an unencrypted link in a conformant deployment.
- **Peer impersonation.** Mitigated by iroh's `node_id`-based peer
  authentication at the transport layer and by protocol-plane
  `node_id` signatures on announcements and tickets (`10-core.md §4`,
  `30-transport.md`).
- **Traffic analysis.** Partially mitigated by iroh's forward-secret
  transport. The *existence* of protocol-plane records (governance
  activity, announcement presence) is intentionally discoverable; the
  protocol does not attempt to hide the fact that a pilot has
  published.

---

## 7. Pilot-personal-data threats

`public` IGC files typically carry pilot identifiers in their headers
(pilot name, glider class, competition ID). This data is visible to
every node that fetches the public artifact.

Protocol-level controls:

- Compliant nodes SHOULD NOT scrape or index these fields outside
  records the pilot has explicitly authorized (`R-ACCESS-03`).
  `(R-THREAT-07)`
- Portals MUST NOT use the sanitized protected-flight artifact itself
  as the source of pilot identity display (`R-META-15`). `(R-THREAT-08)`

These controls are legal and compliance obligations on operators,
not cryptographic guarantees. The protocol cannot prevent a
non-compliant node from harvesting headers; it can only declare the
obligation and let ecosystems react to non-compliance.

---

## 8. Authentication and VC threats

- **VC-JWT forgery.** Infeasible under the EdDSA assumption.
- **VC replay across relying parties.** Mitigated by audience
  binding (`aud` claim) and by short expiry (`exp`) in
  `PilotProfileCredential` and in third-party VCs. Relying parties
  MUST verify `aud` matches their identifier and MUST verify `exp`
  if present. `(R-THREAT-09)`
- **Stale VC after rotation.** Relying parties MUST consult the
  `pilot-auth-did-record` chain before accepting a VC. A VC signed by
  a `pilot_auth_did` that has been superseded on the relying party's
  current governance state MUST be rejected (`R-AUTH-10`).
- **Cross-site session hijack.** Out of scope at the protocol layer;
  portals implement session security (cookies, CSRF, token storage).
  The protocol's contribution is a short-lived signed assertion from
  `pilot_auth_did`, not a session mechanism.
- **VC issuer impersonation (third-party VCs).** For pilot-held
  credentials, this is bounded by `did:key` self-description plus the
  `pilot-auth-did-record` chain. For portal-issued third-party VCs,
  issuer identity depends on the portal's `did:web` control and is
  covered in §11.

---

## 9. Governance-plane threats

- **Malicious challenges / dispute flooding.** A resolver can issue
  challenges; a resolver issuing frivolous challenges against many
  pilots is a governance-plane abuse. Mitigated by resolver-trust
  scoping (nodes honour only their trusted resolvers) and by
  portal-layer rate limits (out of protocol scope).
- **Resolver compromise.** A compromised resolver can issue bogus
  approvals, challenges, resolutions, and identity-recovery records,
  bounded to nodes that trust that resolver. Compliant nodes SHOULD
  trust only a minimal resolver set and SHOULD monitor resolver
  behaviour. `(R-THREAT-10)`
- **Identity-recovery abuse.** A compromised resolver can hijack
  pilots by issuing bogus `identity-recovery` records. Mitigations
  include out-of-band verification by the resolver before issuance,
  pilot notification flows at the portal layer, and multi-resolver
  attestation patterns for high-value identities (SHOULD-level
  guidance, deployment-specific).

---

## 10. Abuse analysis

This section covers protocol-adjacent abuse that stays within the
cryptographic rules but still harms users or operators.

- **Spam / governance noise.** Attackers may flood portals or nodes
  with malformed, duplicate, or low-value records. The protocol cannot
  prevent publication attempts; mitigation is duplicate suppression,
  rate-limiting, and resolver-trust scoping.
- **Identity squatting pressure.** A portal may pressure pilots to use
  a portal-hosted public alias or issuer flow that is simpler for the
  portal but weakens portability. This is mitigated by keeping
  `pilot_auth_did` wallet-held and authoritative while treating
  `did:web` pilot aliases as optional mirrors only.
- **Harassment / coercive re-identification.** Even when a flight is
  `protected`, operators may try to correlate identity from cached
  headers, off-protocol data, or prior sessions. The protocol can only
  impose scrape-avoidance and display-source obligations; it cannot
  cryptographically stop a malicious operator.
- **Legal-takedown churn.** An operator may repeatedly remove and
  re-add public-facing alias material or public VC endpoints to create
  instability without altering governance truth. This is primarily an
  operator-trust problem; governance state remains the protocol truth.
- **Resolver abuse.** A resolver or resolver set may selectively
  challenge, stall, or burden a pilot. The mitigation remains local
  resolver-trust minimization and auditability of resolver behavior.

The abuse model does not add a second authority path. It exists to explain where
the protocol can declare operator obligations but cannot enforce them
cryptographically.

---

## 11. `did:web` attack surface

Per this specification, `did:web` is not permitted as the authoritative pilot-held
`pilot_auth_did`. Its threat surface is therefore limited to:

- portal-issued third-party VC issuer identity
- optional public-facing pilot aliases or mirrors operated by a portal or public
  node

Threats:

- **DNS / CA / HTTPS trust-chain compromise.** An attacker who can hijack the
  portal's domain, TLS termination, or DID document hosting can substitute keys
  or issuer metadata for that `did:web`.
- **Domain seizure or hosting loss.** A legitimate issuer may lose control of
  its domain, making the `did:web` unavailable even though previously issued
  credentials still exist.
- **Stale or ambiguous historic verification.** Old credentials issued under an
  earlier portal key may become hard to validate if the `did:web` document no
  longer exposes that historic key material.
- **Alias / governance mismatch.** A portal-hosted pilot-facing `did:web` alias
  may drift from the pilot's actual current `pilot_auth_did` in governance state.
  The risk is user confusion, not protocol takeover, if verifiers obey the
  authority rule.

Mitigations:

- A verifier MUST NOT treat a pilot-facing `did:web` alias or mirror as an
  override of the authoritative `pilot-auth-did-record` chain. `(R-THREAT-11)`
- For pilot identity binding, the governance rule wins on conflict:
  `pilot_id` -> `pilot-auth-did-record` -> current `did:key`.
- Portal-issued third-party VCs using `did:web` remain issuer-controlled
  credentials. They do not alter pilot binding, ownership, or fetch rights.
- Portals exposing a pilot-facing `did:web` alias SHOULD describe it as a public
  alias or mirror, not as the authoritative root identity.

