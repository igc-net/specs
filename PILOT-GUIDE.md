# `igc-net` Pilot Guide

This guide explains `igc-net` from the pilot's point of view. It is
informative. For protocol rules, use the numbered specification documents.

## What `igc-net` Changes

`igc-net` gives the same flight one shared identity across participating
portals. Ownership, publication mode, and private access are not locked inside
one portal's local database.

This does not remove trust. It makes trust more explicit.

## Pick a Home First

Most pilots should start with one trusted home:

- a main portal, or
- a self-hosted node

Use that home for identity, profile VC issuance/presentation, and recovery. Add other
portals only when they need a specific role.

## Two Node Categories

Every node you log in to is in exactly one of two categories for you:

| Category | What it means | Typical use |
|---|---|---|
| Identity-linked node | The node knows you are logged in. It can discover and show your `public` flights and the sanitized side of your `protected` flights. It does NOT hold the key to read your private content. | Sign-in on a scoring site or a secondary portal that only needs to show your public activity. |
| Private-access node | All of the above, plus you have given it your `private_access_keypair` so it can fetch your `private` flights and the unsanitized raw companion of your `protected` flights. It still does not become the authority for your identity; that comes from your wallet-held `did:key` plus governance state. | Your main home portal, your self-hosted node, an archive node you trust, or any portal you want to see "everything". |

## Your identity credentials

You have three distinct credentials:

- `pilot_id` — your stable root protocol identity
- `pilot_auth_did` — your wallet-held authentication / profile credential DID
- `private_access_keypair` — your fetch authorization credential for non-public content

Your canonical portable authentication identity is a `did:key`.
Some portals may also expose a public-facing `did:web` alias or issuer identity,
but that does not replace your authoritative pilot binding inside `igc-net`.

The two actions are clearly separate in your head:

- **Log me in** → identity-linked node. The portal knows you're Alice;
  it can show your public and sanitized-protected flights.
- **Log me in AND grant private access** → private-access node. The
  portal also holds your `private_access_keypair` and can see
  everything that is yours.

You can revoke private access at any time without signing out (see below).

## Groups and Follow

Groups let you share private flight access with specific pilots without making your flights public.

| Type | What it means |
|---|---|
| Private group | You are the owner. You add members individually. Members can fetch all of your non-public flights — past and future — via a `GroupFetchProof`, without needing your `private_access_keypair`. |
| Public group | Opt-in. Any member's flights are accessible to all other current members, regardless of publication mode. Membership is by invitation and explicit acceptance. |

**Follow** — subscribe to another pilot's upload notifications. Your access to their flights still follows their publication mode, elevated if you share a group with them.

Group records are signed by you and stored on the data plane; no trusted resolver approval is required.

## Your `private_access_keypair`

This is a single Ed25519 keypair that controls access to all of your
non-public IGC content: private IGC files and protected raw
companions. It is one of two paths that authorize non-public fetch
requests — the other is a `GroupFetchProof` from a group you own or
belong to.

Your home portal (or self-hosted node) generates this keypair and
helps you back it up. When you want to grant a new private-access
node, you hand over this keypair through the portal-to-portal grant
flow — typically a one-time approval screen with a QR code or short
URL.

**Back it up.** If you lose your home portal AND lose your backup of
this key, you have to publish a new one (see the next section). You
will not lose your flights, because ownership is anchored on your
`pilot_id`, not on this key.

## Revoking One Node's Private Access

You can take private access away from a single node (for example, a
portal you no longer trust) without affecting the others.

Ask that node to delete its copy of your `private_access_keypair`.
Compliant nodes honor this immediately. After deletion the node
cannot sign new fetch requests for your private content.

If you cannot trust the node to cooperate (for example, a compromised
portal), asking it to delete is not sufficient — you must also publish
a `private-access-rotation-record` signed by your `pilot_id`. Compliant
nodes stop honoring fetch-request signatures under the superseded key as
soon as they process the rotation record.

## Recovering If You Lose Everything

If you lose your `pilot_id` itself (for example, your main portal is
gone AND you never backed anything up), you go through the resolver-
assisted identity recovery flow in `50-governance.md §10`. The
resolver issues an `identity-recovery` record that binds your old
`pilot_id` to a new one, and all your flights move to the new
identity on compliant nodes.

After identity recovery you also publish a fresh
`private-access-rotation-record` signed by your new `pilot_id`,
re-establish your `pilot_auth_did`, and re-grant private access to
whichever nodes you want.

## Publication Modes

| Mode | What the network can see | Good default use |
|---|---|---|
| `public` | Raw IGC is served as plaintext to anyone over iroh. | Flights you want fully open. |
| `protected` | A sanitized public artifact is served to anyone; the unsanitized raw IGC is served only to nodes you have granted private access. | Flights you want shared without exposing identity in the public artifact. |
| `private` | Only nodes you have granted private access can fetch the raw IGC. | Flights you do not want publicly visible. |

Important limits:

- `protected` is not the same as `private`.
- `protected` hides identity in the public artifact, not necessarily from a
  portal that already has private access to your profile.
- `private` hides content, not the existence of the hash — the
  `raw_igc_hash` is broadcast on the data-plane announce topic and may
  be discovered by any participating node. Linking the hash to your
  pilot identity requires querying governance state separately.
- No flight content is encrypted at the protocol layer. Transport
  confidentiality is provided by iroh's end-to-end encryption between
  nodes; at-rest confidentiality is a legal and compliance obligation
  on each node that holds your plaintext.

## Adding Another Portal

Before you grant anything, a new portal can usually see:

- `public` flights you own
- the sanitized public side of your `protected` flights

If you only log in ("identity-linked"), that is where things stop.

If you log in AND grant private access ("private-access"), it can
also:

- fetch the raw IGC bytes of your `private` flights
- fetch the unsanitized raw companion of your `protected` flights

Login and private-access are separate decisions. A portal can need
one without needing the other.

## Public Metadata Advertisements

Portals may publish public metadata advertisements for derived resources such as
thermal annotations, wind estimates, scoring output, or replay data.

These advertisements are public discovery records. They may refer to public,
protected, private, or portal-local resources, but the advertisement itself does
not grant access to those resources.

A portal is responsible for deciding whether derived values are safe to
advertise publicly. igc-net does not guarantee that derived metadata is
non-sensitive.

## Revocation, Deletion, and Disputes

These are different actions:

- **Revocation**: tell a portal to delete its copy of your
  `private_access_keypair`; optionally publish a rotation record so
  serving nodes stop honoring the old key.
- **Deletion**: ask compliant nodes to stop serving the flight and
  remove required flight-scoped records.
- **Dispute**: if ownership becomes contested, compliant nodes stop
  further private release until the dispute is resolved.

Do not treat these as the same mechanism.

## Practical Default

For most pilots, the simplest workable model is:

1. Keep one trusted home for identity, `pilot_auth_did`, and recovery.
2. Treat new portals as identity-linked by default (log in only).
3. Grant private access only when a portal clearly needs it.
4. Use `protected` for flights you want shared but not identity-bearing in the
   public artifact.
5. Use `private` when you do not want the flight track publicly visible.
6. Keep a secure backup of your `private_access_keypair`. Rotate it
   if you believe a node has been compromised.
7. Use private groups to share non-public flights with specific pilots
   without changing your publication mode.

## Read Next

- `NODE-OPERATOR-GUIDE.md` if you run your own node
- `PORTAL-OPERATOR-GUIDE.md` if you are evaluating a portal integration
- `20-artifacts.md`, `40-pilot-and-metadata.md`, `50-governance.md`,
  `60-keys-and-access.md`, and `75-groups-and-social.md` for the underlying rules
