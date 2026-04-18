# Getting Started with igc-net

This is the shortest useful entry point for implementers. It is informative.
For protocol rules, use the numbered specification documents.

## Read Order

Read in this order:

1. `00-overview.md` — system model, terminology, document map
2. `10-core.md` — hashes, signatures, canonical JSON, identity anchors
3. `20-artifacts.md` — publication modes and sanitization
4. `30-transport.md` — announcements, tickets, fetch rules
5. `40-pilot-and-metadata.md` — profile, flight metadata, `igc-metadata`
6. `50-governance.md` — claims, approvals, challenges, resolutions
7. `55-governance-sync.md` — governance propagation and catch-up
8. `60-keys-and-access.md` — node categories, `private_access_keypair`,
   fetch authorization, rotation
9. `70-durability.md` — deletion, archive, mode-upgrade obligations
10. `80-analytics.md` — optional analytics extension
11. `90-conformance.md` — requirement labels and implementation profiles

If an informative guide conflicts with a numbered document, the numbered
document wins.

## Interoperability-Critical Constants

Use these strings exactly:

| Purpose | UTF-8 input string |
|---------|-------------------|
| Governance topic ID | `igc-net/governance/v1` |
| Announce topic ID | `igc-net/announce/v1` |
| Analytics topic ID (optional) | `igc-net/analytics/v1` |

Topic IDs are derived as `BLAKE3(UTF-8(input_string))[0..32]`, encoded as
64-character lowercase hex.

The pilot's `private_access_keypair` is an ordinary Ed25519 keypair held
and backed up by the pilot. No key-derivation info-strings are defined.

## Minimal Useful Implementation

A practical minimal node does the following:

1. Accept raw `.igc` bytes.
2. Compute `raw_igc_hash = BLAKE3(raw_igc_bytes)`.
3. Choose `publication_mode`: `public`, `protected`, or `private`.
4. For `protected`, run the normative sanitization algorithm and compute
   `protected_hash = BLAKE3(sanitized_igc_bytes)`.
5. Construct and sign the `publication-mode-record`.
6. Announce availability on the data-plane announce topic. Tickets for
   `protected` raw companions and `private` raw IGC are locators only;
   the serving node gates delivery on a fetch request signed by the
   pilot's currently authorized `private_access_keypair`.
7. Subscribe to the governance topic and process records — including
   `private-access-rotation-record` — before serving any restricted
   content.

No protocol-layer AEAD is applied to content. Confidentiality in flight
is provided by iroh's end-to-end encrypted transport; confidentiality at
rest for non-public content is a compliance and legal obligation on the
holder.

## Key Model Summary

Each pilot has two Ed25519 keypair classes:

| Key | Purpose |
|-----|---------|
| `pilot_id` root key | Signs pilot-authored records (claims, publication-mode records, deletion requests, pilot-profile, flight-metadata, `private-access-rotation-record`). Does not rotate. |
| `private_access_keypair` | Authorization credential for all non-public content belonging to this pilot: signs fetch requests for private raw IGC, protected raw companions, private `pilot-profile`, private `flight-metadata`, and `igc-metadata`. MAY be rotated via `private-access-rotation-record` signed by `pilot_id`. |

Serving nodes also have their own `node_id` Ed25519 keypairs (used to
sign announcements and tickets); trusted resolvers have their own
`resolver_id` keypairs. All three identity-role keypairs are distinct.

Read `60-keys-and-access.md` before implementing any private-content flow.
