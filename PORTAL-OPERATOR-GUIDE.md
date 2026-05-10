# `igc-net` Portal Operator Guide

This guide is for teams integrating `igc-net` into an existing portal. It is
informative. Read it together with `NODE-OPERATOR-GUIDE.md`.

## First Decision: Access Category

Before any UI or schema design, decide which node access category your
portal is designed to operate in **per pilot**:

| Category | What it means |
|---|---|
| Identity-linked (Category 1) | The pilot has logged in. You know their `pilot_id`, you discover and serve their `public` and `protected` (sanitized) artifacts, you do NOT hold their `private_access_keypair`. |
| Private-access (Category 2) | All of the above plus the pilot has granted you their `private_access_keypair`, so you can fetch private IGC and protected raw companions. |

A portal can be Category 1 for one pilot and Category 2 for another.
What matters is that your product language and your UX make this
distinction obvious. Many portals want to default to Category 1 and
offer Category 2 only when the pilot has a reason to grant it.

If you do not make this decision early, the portal will tend to ask
for too much.

## "Log In" vs "Log In AND Grant Private Access"

Treat these as **two separate user-facing actions**:

1. **Log in** — identity-linked. Your portal displays the pilot's
   public and protected (sanitized) flights. No key handover happens.
   This is what "Log in with igc-net" should mean by default.
2. **Log in AND grant private access** — private-access. In addition
   to (1), the pilot transfers their `private_access_keypair` via the
   portal-to-portal grant flow in `60-keys-and-access.md §5`. Your
   portal now sees everything the pilot has.

Never bundle these silently. A grant screen should be explicit,
reversible, and describe in plain language what the portal will be
able to read after the grant.

### Grant-flow UX guidelines

- Show the exact categories of content that will become readable
  (private flights and protected raw companions).
- Offer a "grant later" option alongside "grant now" on first login.
- Describe revocation as part of the grant flow, not as a hidden
  setting.
- Do not ask for the `private_access_keypair` when you only need
  Category 1 capabilities.

## Common-Case Product Flows

Most portals need to get five flows right:

1. Upload with a clear `public` / `protected` / `private` choice.
2. Explain what trust the pilot is granting (and clearly separate
   login from private-access grant).
3. Add a new portal or rotate `private_access_keypair` when the pilot
   needs it.
4. React safely to contested flights, deletion, mode changes, and
   `private-access-rotation-record`.
5. Display protected flights without assuming identity display is
   allowed.

Everything else is secondary.

## Storage Classes

Do not collapse everything into one generic flight object.

| Class | Typical contents |
|---|---|
| Public storage | Public raw artifacts, protected sanitized artifacts, public indexes |
| Restricted storage | Protected raw companion plaintext, private raw IGC plaintext |
| Governance state | Claims, approvals, challenges, resolutions, mode changes, deletion requests, `private-access-rotation-record`, `pilot-auth-did-record`, roster updates |

Restricted storage means stricter serving rules, cache invalidation
on rotation/revocation, and delete handling. It is plaintext at rest
— you are responsible for encrypting the disk, controlling access,
and auditing exposure. This is a legal and compliance obligation, not
a protocol-layer guarantee.

## Safety Rules

Your portal must do these correctly:

- Governance state before restricted serving.
- Immediate stop-serve on restrictive mode change.
- Immediate stop-serve on valid deletion request.
- Immediate swap of the authorized `private_access_public_key` on
  receipt of a new `private-access-rotation-record`; stop honoring
  fetch-request signatures under the superseded key at once.
- No identity display for protected flights unless the portal has a
  separate valid basis to read and show it (for example, a latest
  verified non-stale `PilotProfileCredential`).
- No scraping of personal-identity fields from IGC headers outside
  records the pilot has explicitly authorized (see
  `60-keys-and-access.md §2.1`).
- Metadata advertisements are public, portal-defined discovery records. Do not
  publish derived values that your portal policy treats as sensitive.

The failure mode is usually stale local state, not bad hashing.

## Integration Boundary

Use the reference implementation or a shared protocol layer for:

- hashing and canonicalization
- signing and verification
- announcement and fetch machinery
- governance-state interpretation (including rotation records)
- pilot authentication DID state interpretation
- fetch-request signing and validation against the current
  `private_access_public_key`

Keep portal-specific code focused on:

- UX (especially the login vs private-access-grant distinction)
- account model
- trust presentation
- retention policy
- local disclosure policy
- plaintext-at-rest protection

Do not spread protocol logic through application code. Do not
re-implement primitives; igc-net is BLAKE3 + Ed25519 + RFC 8785 canonical
JSON, and the transport is iroh.

## Portal Authentication Trust Model

igc-net authenticates pilots solely via `(pilot_hash, access_pin) → PilotProfileCredentialJWT`.
The network holds no email addresses, session state, or recovery data. Portals that want
to spare their users from typing a raw hash and PIN on every visit act as **credential brokers**:
they authenticate the user through their own mechanism and call igc-net on the user's behalf.

### The broker model

1. **PIN generation** — at self-registration the portal generates a random, high-entropy
   `access_pin` (≥ 24 base62 characters) and passes it to the sidecar `RegisterPilot` RPC.
   The pilot is never expected to remember or type this PIN. It is opaque infrastructure.

2. **Client-side PIN encryption** — the portal uses the WebAuthn PRF extension
   (`credentials.prf`) to derive a symmetric key on the user's authenticator device. The
   browser encrypts the plaintext PIN with this key before the server sees it. The server
   stores only the opaque ciphertext (`encrypted_pin`) alongside the PRF salt. A pilot with
   N enrolled passkey devices has N independent encrypted copies of their PIN — one per
   device.

3. **Server-side storage** — the server stores `(credential_id, public_key, encrypted_pin,
   pin_salt)` per passkey row. It never persists the plaintext PIN.

4. **Login** — the WebAuthn assertion triggers the same PRF evaluation on the user's
   device. The browser decrypts the ciphertext client-side and sends the plaintext PIN to
   the server over TLS. The server calls `IssuePortalAuthToken(pilot_id, pin)`, creates a
   session, and discards the PIN from memory. The server is in possession of the PIN for
   the duration of one gRPC call only.

### Consequences

- **The portal operator cannot impersonate a pilot without user interaction.** The
  ciphertext is only decryptable on an enrolled authenticator device.
- **If the server database is stolen,** `encrypted_pin` values require breaking AES-256-GCM
  with a key derived from hardware authenticator PRF output — not feasible. The PIN is
  random and high-entropy, so offline dictionary attacks do not apply.
- **If a pilot loses all their passkey devices,** the portal cannot recover their PIN.
  Recovery requires the pilot to supply their pilot hash and the recovery PIN shown once at
  registration, then re-enroll a new passkey. The pilot hash is the cross-portal identity
  token — pilots should be encouraged to save it (e.g. screenshot the QR code).

### Informed consent at registration

Registration UX MUST:

- Describe that the portal holds encrypted igc-net credentials on the pilot's behalf.
- Show the plaintext PIN exactly once as a **recovery code** for offline storage. The user
  must acknowledge this before proceeding.
- Show the pilot hash as a QR code for cross-portal portability.
- State in plain language that losing both the recovery code and all passkey devices means
  permanent loss of access to this igc-net identity.

### QR / hash-only pilots

Pilots who already have a pilot hash (registered on another portal or directly) can import
it via QR code scan or paste. The portal verifies the `(hash, pin)` pair against the sidecar
using `IssuePortalAuthToken`, then proceeds to passkey enrollment. Email is optional in this
path. No igc-net changes are needed for cross-portal import.

### What igc-net does NOT need to provide

Under this model, igc-net does not need to store or verify email addresses, issue recovery
tokens, or maintain session state. All of that is a portal concern. The hash is the only
identity token igc-net manages; email is a portal-level convenience that binds to the hash
within that portal's local database.

---

## Release Checklist

Before shipping a portal integration, verify:

1. Access category per pilot is explicit in product language (no
   ambiguous "log in" that silently takes the key).
2. Upload mode selection is clear.
3. Protected-flight identity display is correct (never sourced from
   the sanitized artifact).
4. Contested, delete, mode-upgrade, and rotation-record handling are
   tested.
5. Offline catch-up does not leave restricted content served from
   stale state.
6. Restricted plaintext storage is at-rest encrypted, access-
   controlled, and audited.
7. Scrape-avoidance behavior is enforced on all IGC files the portal
   handles.

## Read Next

- `NODE-OPERATOR-GUIDE.md`
- `20-artifacts.md`
- `40-pilot-and-metadata.md`
- `50-governance.md`
- `55-governance-sync.md`
- `60-keys-and-access.md`
- `70-durability.md`
