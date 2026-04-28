# Contributing to `igc-net`

## Scope

This repository contains protocol-level documentation only:

- wire-format and content-addressing rules
- metadata advertisement and deferred analytics boundary definitions
- interoperability requirements
- governance notes and open protocol questions

Language-specific APIs, crate ergonomics, CLI behavior, and application
integration details belong in implementation repositories such as `igc-net-rs`.

## How to Contribute

- **Clarification or bug report**: open a GitHub issue describing the ambiguity,
  inconsistency, or missing constraint
- **Protocol change proposal**: open an issue before sending a pull request if
  the change is non-trivial
- **Implementor feedback**: if a real implementation exposes a mismatch between
  the spec and feasible interoperable behavior, document the divergence
  explicitly and propose a spec correction

## Specification Rules

- Normative language uses RFC 2119 keywords
- Breaking changes to wire formats, schema identifiers, or topic derivation
  strings require a version change
- Examples must be kept consistent with field tables and normative rules
- Spec text must distinguish clearly between:
  - normative behavior
  - implementation guidance
  - future work

## Review Standard

A good spec change:

- reduces ambiguity rather than moving it elsewhere
- states concrete interoperability consequences
- avoids application-specific assumptions
- updates every affected document in the same change

## Governance

The protocol is initially maintained by NTNU, the public university and DSE Lab.
The intention is to share governance within the soaring community and across
participating platforms.
