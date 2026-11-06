### Added

- ADR-0026, roles, rights and channels: the rights of each role in one table, single-use step-up for revoking and retiring, no critical alert silenced from the web site, hosts retired instead of deleted, "monitor" tokens on the host channel only, step-up and alerts on one's own second factor, AI report requests and their failures, the completed list of keys to back up offline and a key backup register.
- ADR-0027, which supersedes ADR-0021: each channel trusts one CA role, with a set of Agent CA anchors from 0.1, and a complete re-key of the Agent CA.
- Review report of 2026-11-06 in `doc/analyse/reviews/`.

### Changed

- Use case diagrams of V1: the nine open points decided by the product owner on 2026-11-06 are drawn and recorded.
- ADR-0021 marked superseded by ADR-0027; ADR-0022 to ADR-0025 point to ADR-0026; references to ADR-0021 in the administration and dependency pages point to ADR-0027.
