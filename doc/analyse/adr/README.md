# Architecture Decision Records

This folder holds the Architecture Decision Records (ADR) of the Dashboard project. An ADR records one architectural decision: its context, the decision itself, the alternatives that were considered and the consequences.

## Rules

- One decision per file, named `NNNN-short-title.md`, numbered in creation order. Numbers are never reused.
- An ADR is written in English by the main developer, validated by the product owner (who decides), then reviewed for format and accuracy.
- An accepted ADR is never rewritten to change the decision. A new ADR supersedes it, and both files reference each other. (Single exception, before the repository was published: ADR-0001 and ADR-0004 were amended on the day of their acceptance to follow ADR-0005. The rule applies without exception from then on.) Fixing a cross-reference or a name in an accepted ADR, without changing its decision, is an editorial correction and is allowed.
- Every statement must be exact. A claim that could not be checked is marked **not verified**.

## Statuses

| Status | Meaning |
|---|---|
| Proposed | Drafted, waiting for the product owner's decision. Open points are listed in the ADR. |
| Accepted | Decided. The code must follow it. |
| Rejected | Considered and not adopted; kept for the record. |
| Superseded by ADR-NNNN | Replaced by a later ADR (linked). |
| Deprecated | No longer applies, without a replacement. |

Open points marked "raised in review" were added after the team review of 2026-11-03; every one of them was decided on 2026-11-04. A second team review took place on 2026-11-04, after the decisions ([report](../reviews/2026-11-04-second-reading.md)).

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
| [0019](0019-web-authentication-verified-by-the-api.md) | Web authentication verified by the API, with step-up for sensitive actions | Accepted | 2026-11-05 |
| [0020](0020-remote-commands-and-keyed-audit-trail.md) | Remote commands, and a keyed audit trail that cannot be purged early | Accepted | 2026-11-05 |
| [0021](0021-machine-identity-from-the-first-increment.md) | Machine identity from the first increment, with a dedicated revocation list service | Accepted | 2026-11-05 |
