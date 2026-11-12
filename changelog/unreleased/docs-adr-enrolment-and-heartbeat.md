### Added

- ADR-0028: the heartbeat is a unary call, separate from the command stream.
- ADR-0029: the Collector resolves the agent identity on every call, without a cache, and the Agent CA revocation list is regenerated daily and as soon as the revoked serials change, read at a short interval, with stated failure rules.
- ADR-0030: certificate issuance bound by the API, and its records (issuance ticket, exact certificate profile, atomic successor, suspect hosts refused, refused certificates revoked, bounded CSR intake, record of the Enrolment service's server certificate).
- ADR-0031: enrolment token lifecycle (one live token per host, new and pending hosts, records of creation and refusals, one-shot enrolment command, yearly re-enrolment).
- ADR-0032: critical states logged and listed by a host command from 0.1.
- Review report of the identity sequences of 0.1 and of ADR-0028 to ADR-0032.

### Changed

- ADR index, back-references in ADR-0004, ADR-0005, ADR-0022, ADR-0023, ADR-0025, ADR-0026 and ADR-0027, and the future improvements.
