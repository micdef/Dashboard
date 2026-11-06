### Added

- ADR-0028: the heartbeat is a unary call, separate from the command stream.
- ADR-0029: the Collector resolves the agent identity on every call, without a cache, and the Agent CA revocation list keeps its daily regeneration, with stated failure rules.
- ADR-0030: certificate issuance bound by the API (issuance ticket, exact certificate profile, atomic successor, suspect hosts refused, refused certificates revoked, CSR intake bounded).
- ADR-0031: enrolment token lifecycle (selector and verifier, one live token per host, pending state, records of creation and refusals, one-shot enrolment command, yearly re-enrolment).

### Changed

- ADR index, back-references in ADR-0004, ADR-0005, ADR-0025, ADR-0026 and ADR-0027, and the future improvements.
