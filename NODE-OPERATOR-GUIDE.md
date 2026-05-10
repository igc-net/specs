# `igc-net` Node Operator Guide

This guide is for self-hosted and technical node operators. It is informative.
For protocol rules, use the numbered specification documents.

## Start With the Node Role

Pick one role before you configure anything:

| Role | What it should do |
|---|---|
| Public mirror | Store and serve public content only |
| Trusted personal node | Hold your own (or your pilots') private data and grant private access |
| Durability node | Keep approved backup copies of private content, with an explicit pilot grant of `private_access_keypair` |

Most mistakes come from accidental over-custody: holding the
`private_access_keypair` or plaintext private content that the node
does not actually need.

## The Two-Category Model

Each time a pilot logs in to your node, your node operates in
exactly one of two categories for that pilot:

| Category | Description |
|---|---|
| Identity-linked (Category 1) | You know the pilot's `pilot_id`. You may discover, serve, and index the pilot's `public` and `protected` (sanitized) artifacts. You do NOT hold the pilot's `private_access_keypair`. |
| Private-access (Category 2) | All of the above, plus the pilot has granted you the `private_access_keypair`. You MAY sign fetch requests for their non-public content and you hold plaintext private content under confidentiality obligations. |

**Do not conflate "log the pilot in" with "hold the pilot's
`private_access_keypair`".** These are separate operator decisions and
separate user-facing actions. A Category 1 node MUST NOT accept a
transfer of the `private_access_keypair` during login. Ask for the key
only when the pilot is consciously making a private-access grant.

Declare each node's intended category in your operational
documentation so users know what they are agreeing to.

## What a Node May Store

Think in classes:

| Class | Examples |
|---|---|
| Public artifacts | Public raw IGC, protected sanitized artifacts, public metadata advertisements |
| Protected raw companion | The unsanitized raw IGC of a `protected` flight — plaintext, gated on signed fetch |
| Private raw IGC | The raw IGC of a `private` flight — plaintext, gated on signed fetch |
| Governance records | Claims, approvals, challenges, resolutions, mode changes, deletion requests, `private-access-rotation-record`, `pilot-auth-did-record`, roster updates |

Governance records are not just informational. They change what the node is
allowed to serve.

## Plaintext at Rest and Legal Obligation

igc-net deliberately has no protocol-layer content encryption. iroh
provides end-to-end encryption on the wire between authenticated node
endpoints, but anything a node receives for a pilot's private content
is **plaintext at rest** on that node.

If you are a Category 2 node, this means:

- You must protect plaintext restricted IGC bytes on disk at rest (disk
  encryption, access control, audit, backup hygiene — all standard operator
  hygiene).
- You must honor the pilot's revocation and deletion requests as
  compliance obligations. The protocol cannot force a non-compliant
  node to forget; only law and contract can.
- You must not re-share plaintext with third parties except as the
  pilot's own authorizations permit.

This is the same trust model igc-net uses for public archive nodes that
hold public content — just with confidentiality obligations added.

## Scrape-Avoidance Obligation

Even for `public` IGC files you may legitimately fetch and serve,
compliant nodes **SHOULD NOT** extract or index personal-identity
fields from IGC headers (e.g., `HFPLT`, `HFCID`, `HFGID`, `HFRFW`,
`HFFTYFRTYPE`, `HOPLT`, `HOCID`) outside records the pilot has
explicitly authorized. Display authoritative pilot identity only from a
verified, non-stale `PilotProfileCredential` or other explicitly
authorized source — not from raw IGC headers of files that happen to pass
through your node.

This applies to identity-linked and private-access nodes alike.

## What to Announce

Default rule:

- Re-announce public artifacts you actually hold and are willing to serve.
- Re-announce protected sanitized artifacts as public artifacts.
- You may also re-announce the private and protected-raw-companion
  tickets for discovery; serving those bytes still requires a signed
  fetch request.

Keep "we have it" separate from "anyone may fetch it".

## What to Serve

| Mode | Serve |
|---|---|
| `public` | Raw artifact to anyone |
| `protected` sanitized | Openly to anyone |
| `protected` raw companion | Only to requesters with a fetch request signed by the pilot's currently authorized `private_access_keypair` |
| `private` raw IGC | Only to requesters with a fetch request signed by the pilot's currently authorized `private_access_keypair` |

The "currently authorized" public key for each pilot is whichever one
is bound by the most recent valid `private-access-rotation-record` on
the governance topic.

## Governance Comes First

Before serving restricted content, apply governance state:

- `contested` means stop further restricted serving.
- `rejected` means refuse serving.
- mode upgrades take effect immediately.
- deletion requests stop serving immediately and trigger cleanup obligations.
- a new `private-access-rotation-record` takes effect immediately;
  stop honoring signatures under the superseded key.

If governance state is stale, serving decisions are unsafe.

## Durability Rule

For `public` content, replication is straightforward.

For `private` durability, the critical fact is:

- If your node holds a pilot's `private_access_keypair`, it holds
  plaintext private content and is a Category 2 node. There is no
  separate fetch-only or archive-only credential; the `private_access_keypair`
  both authorizes fetches and reads plaintext.
- Durability is a compliance role layered on top of Category 2:
  the durability node commits to retaining the plaintext and to
  honoring deletion and rotation events.

If you run a durability node, make the custody implication explicit
in your terms of service and your operational runbook.

## Minimal Operating Checklist

1. Decide whether the node is public-only, identity-linked-only, or private-access.
2. Never accept a pilot's `private_access_keypair` during login — ask only when the pilot is explicitly granting private access.
3. Never scrape or display personal-identity fields from IGC headers outside pilot-authorized records.
4. Subscribe to the governance topic and process records before any restricted serving.
5. Stop serving immediately on delete, challenge, restrictive mode change, or rotation record.
6. Protect plaintext private content at rest (disk encryption, access control, audit).
7. Keep a clear local policy for what the node stores, announces, and deletes.

## Read Next

- `20-artifacts.md`
- `30-transport.md`
- `50-governance.md`
- `55-governance-sync.md`
- `60-keys-and-access.md`
- `70-durability.md`
