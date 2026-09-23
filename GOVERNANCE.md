# Governance

## Draft status

The Agent Package Specification is currently a draft proposal. The repository
exists to collect implementation experience and community feedback before any
claim of adoption or standardization.

## Decision process

Changes are discussed in GitHub issues and proposed through pull requests.
Maintainers evaluate proposals using the following considerations:

- technical correctness;
- interoperability across agent hosts, registries, and tooling;
- compatibility with existing Skill and Plugin formats;
- security and supply-chain implications;
- implementation complexity; and
- evidence from working implementations.

Substantive decisions should be recorded in the issue or pull request that
motivated them. The specification document remains the source of truth for the
current proposal.

## Change categories

- **Editorial:** Clarifies wording without changing required behavior.
- **Compatible:** Adds optional behavior or guidance without invalidating
  conforming implementations.
- **Breaking:** Changes required behavior, wire representation, media types,
  or validation rules.

Breaking changes require explicit discussion of migration and versioning.

## Releases

Until a versioning policy is adopted, the `main` branch represents the latest
draft. A future tagged release does not imply recognition by a standards body
unless that status is stated explicitly.
