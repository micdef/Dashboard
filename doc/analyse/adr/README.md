# Architecture Decision Records

This folder holds the Architecture Decision Records (ADR) of the Dashboard project. An ADR records one architectural decision: its context, the decision itself, the alternatives that were considered and the consequences.

## Rules

- One decision per file, named `NNNN-short-title.md`, numbered in creation order. Numbers are never reused.
- An ADR is written in English by the main developer, validated by the product owner (who decides), then reviewed for format and accuracy.
- An accepted ADR is never rewritten to change the decision. A new ADR supersedes it, and both files reference each other. (Single exception, before the repository was published: ADR-0001 and ADR-0004 were amended on the day of their acceptance to follow ADR-0005. The rule applies without exception from then on.) Fixing a cross-reference or a name in an accepted ADR, without changing its decision, is an editorial correction and is allowed. A reference to a superseded ADR, in another ADR, is read as a reference to the last ADR of its chain of replacement (for instance ADR-0017 → ADR-0021 → ADR-0027), unless the text names the superseded ADR on purpose (its history, a quotation, or "holds the details"); the references, including the `Complements` fields, are not rewritten.
- A complement may **tighten** a rule of an accepted ADR, if it says so and names the rule; the tightened ADR then gets a reference "tightened by ADR-NNNN", as an editorial correction. A complement never weakens a rule: weakening needs an ADR that supersedes.
- Every statement must be exact. A claim that could not be checked is marked **not verified**.

## Statuses

| Status | Meaning |
|---|---|
| Proposed | Drafted, waiting for the product owner's decision. Open points are listed in the ADR. |
| Accepted | Decided. The code must follow it. |
| Rejected | Considered and not adopted; kept for the record. |
| Superseded by ADR-NNNN | Replaced by a later ADR (linked). |
| Deprecated | No longer applies, without a replacement. |

Open points marked "raised in review" were added after the team review of 2026-11-03; every one of them was decided on 2026-11-04. A second team review took place on 2026-11-04, after the decisions ([report](../reviews/2026-11-04-second-reading.md)), and a third one on 2026-11-05 on the ADRs that came out of it ([report](../reviews/2026-11-05-third-reading.md)). ADR-0026 and ADR-0027 come from the review of 2026-11-06 ([report](../reviews/2026-11-06-roles-and-anchor-sets.md)). ADR-0028 to ADR-0032 come from the sequence diagrams of the machine identity of 0.1 and their team review ([report](../reviews/2026-11-06-identity-sequences-of-0.1.md)), which found points that no ADR settled.

## Vocabulary

| Term | Meaning |
|---|---|
| Backend | the Collector and the API together (ADR-0005) |
| VBE | the **data gateway** ("very-back-end" in the first outline): the only process with database access (ADR-0003). Both names are used in the ADRs. |
| Cold storage | the SMB 3 file share that holds the archive, the prepared training data and the models; "the NAS" in the home lab (ADR-0018) |

## Index

| ADR | Title | Status | Date |
|---|---|---|---|
| [0001](0001-components-and-deployment.md) | Components and deployment topology | Superseded by ADR-0016 | 2026-11-04 |
| [0002](0002-sql-server-developer-edition.md) | SQL Server 2025 Standard Developer edition | Accepted | 2026-11-04 |
| [0003](0003-data-access-through-a-single-gateway.md) | Data access through a single gateway, ADO.NET only | Accepted | 2026-11-04 |
| [0004](0004-grpc-everywhere-and-transports.md) | gRPC everywhere, mTLS for agents, Unix sockets inside the host | Accepted | 2026-11-04 |
| [0005](0005-backend-split-into-collector-and-api.md) | Backend split into a Collector and an API process | Accepted | 2026-11-04 |
| [0006](0006-hexagonal-cores-and-solution-structure.md) | Hexagonal cores where there is domain logic, no CQRS | Accepted | 2026-11-04 |
| [0007](0007-machine-identity-and-enrolment.md) | Machine identity by client certificate, agent enrolment | Superseded by ADR-0017 | 2026-11-04 |
| [0008](0008-telemetry-format-otlp-logs-and-metrics.md) | Telemetry format: the OTLP logs and metrics data models | Accepted | 2026-11-04 |
| [0009](0009-ai-pipeline.md) | AI pipeline: rules, .NET preparation, Python training, ONNX, LLM | Accepted | 2026-11-04 |
| [0010](0010-llm-hosting.md) | Hosting the local LLM on the dedicated AI node | Accepted | 2026-11-04 |
| [0011](0011-cve-source.md) | Source of CVE exposure data | Accepted | 2026-11-04 |
| [0012](0012-storage-and-retention.md) | Storage and retention: hot, cold and inventory history | Accepted | 2026-11-04 |
| [0013](0013-remote-commands.md) | Remote commands without SSH | Superseded by ADR-0020 | 2026-11-04 |
| [0014](0014-authentication-and-roles.md) | Web authentication: local accounts and roles | Superseded by ADR-0019 | 2026-11-04 |
| [0015](0015-increment-order.md) | Order of the increments up to V1 | Accepted | 2026-11-04 |
| [0016](0016-components-and-deployment-with-enrolment.md) | Components and deployment topology, with the Enrolment service | Accepted | 2026-11-04 |
| [0017](0017-machine-identity-two-independent-cas.md) | Machine identity with two independent certificate authorities | Superseded by ADR-0021 | 2026-11-04 |
| [0018](0018-cold-storage-any-smb3-file-server.md) | Cold storage on any SMB 3 file server, mounted by the operating system | Accepted | 2026-11-04 |
| [0019](0019-web-authentication-verified-by-the-api.md) | Web authentication verified by the API, with step-up for sensitive actions | Superseded by ADR-0023 | 2026-11-05 |
| [0020](0020-remote-commands-and-keyed-audit-trail.md) | Remote commands, and a keyed audit trail that cannot be purged early | Superseded by ADR-0024 | 2026-11-05 |
| [0021](0021-machine-identity-from-the-first-increment.md) | Machine identity from the first increment, with a dedicated revocation list service | Superseded by ADR-0027 | 2026-11-05 |
| [0022](0022-operations-alerts-backups-and-reconciliation.md) | Operations: alerts, monitoring of the monitor, backups and reconciliation | Accepted | 2026-11-05 |
| [0023](0023-web-sessions-issued-by-the-api.md) | Web sessions issued by the API, and a host channel for security settings | Accepted | 2026-11-05 |
| [0024](0024-audit-trail-anchored-by-the-verifier.md) | Audit trail written by the data gateway and anchored by the independent verifier | Accepted | 2026-11-05 |
| [0025](0025-enrolment-updates-alerts-and-links.md) | Enrolment safeguards, signed agent updates, protected alerts and the added links | Accepted | 2026-11-05 |
| [0026](0026-roles-rights-and-channels.md) | Roles, rights and channels | Accepted | 2026-11-06 |
| [0027](0027-machine-identity-with-agent-ca-anchor-sets.md) | Machine identity with Agent CA anchor sets and a complete re-key | Accepted | 2026-11-06 |
| [0028](0028-heartbeat-as-a-unary-call.md) | Heartbeat as a unary call, separate from the command stream | Accepted | 2026-11-06 |
| [0029](0029-identity-resolution-and-revocation-propagation.md) | Identity resolution on the agent channel, and how revocations propagate | Accepted | 2026-11-06 |
| [0030](0030-certificate-issuance-bound-by-the-api.md) | Certificate issuance bound by the API, and its records | Accepted | 2026-11-06 |
| [0031](0031-enrolment-token-lifecycle.md) | Enrolment token lifecycle | Accepted | 2026-11-06 |
| [0032](0032-critical-states-visible-on-the-host-in-0.1.md) | Critical states visible on the host from 0.1 | Accepted | 2026-11-06 |
