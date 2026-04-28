# igc-net Protobuf Contracts

This directory contains the normative gRPC/protobuf contract for igc-net.

During the v0.3 draft period, the contract evolves in place and breaking changes
are allowed. Rust, Python, and portal clients must regenerate from the current
files in this directory rather than maintaining implementation-local proto
forks.

The proto contract is part of the protocol specification. If generated clients
or implementation behavior diverge from this directory, this directory is
the canonical source of truth.

Current draft contract:

- [`igc_net_v0.proto`](./igc_net_v0.proto)

The first high-value slice covers igc-net status, publish, artifact fetch,
index query, event subscription, and portal-mediated provisioning/revocation of
igc-net-held `private_access_keypair` material.

Deployment note: igc-net is commonly run as an auxiliary sidecar-style service
next to a portal, with the official API and package naming as `igc-net` /
`igc_net`.
