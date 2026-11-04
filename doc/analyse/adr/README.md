# Architecture Decision Records

This folder holds the Architecture Decision Records (ADR) of the Dashboard project. An ADR records one architectural decision: its context, the decision itself, the alternatives that were considered and the consequences.

## Rules

- One decision per file, named `NNNN-short-title.md`, numbered in creation order. Numbers are never reused.
- An ADR is written in English by the main developer, validated by the product owner (who decides), then reviewed for format and accuracy.
- An accepted ADR is never rewritten to change the decision. A new ADR supersedes it, and both files reference each other.
- Every statement must be exact. A claim that could not be checked is marked **not verified**.

## Statuses

| Status | Meaning |
|---|---|
| Proposed | Drafted, waiting for the product owner's decision. Open points are listed in the ADR. |
| Accepted | Decided. The code must follow it. |
| Rejected | Considered and not adopted; kept for the record. |
| Superseded by ADR-NNNN | Replaced by a later ADR (linked). |
| Deprecated | No longer applies, without a replacement. |

Open points marked "raised in review" were added after the team review of 2026-11-03; they are proposals until the product owner decides.

## Index

| ADR | Title | Status | Date |
|---|---|---|---|
| [0001](0001-components-and-deployment.md) | Components and deployment topology | Accepted | 2026-11-04 |
| [0002](0002-sql-server-developer-edition.md) | SQL Server 2025 Standard Developer edition | Accepted | 2026-11-04 |
| [0003](0003-data-access-through-a-single-gateway.md) | Data access through a single gateway, ADO.NET only | Accepted | 2026-11-04 |
| [0004](0004-grpc-everywhere-and-transports.md) | gRPC everywhere, mTLS for agents, Unix sockets inside the host | Accepted | 2026-11-04 |
| [0005](0005-single-backend-process.md) | One Backend process for the collector and the API | Proposed | 2026-11-03 |
| [0006](0006-hexagonal-cores-and-solution-structure.md) | Hexagonal cores where there is domain logic, no CQRS | Proposed | 2026-11-03 |
| [0007](0007-machine-identity-and-enrolment.md) | Machine identity by client certificate, agent enrolment | Proposed | 2026-11-03 |
| [0008](0008-log-format-otlp-data-model.md) | Log format: the OTLP logs data model | Proposed | 2026-11-03 |
| [0009](0009-ai-pipeline.md) | AI pipeline: rules, .NET preparation, Python training, ONNX, LLM | Proposed | 2026-11-03 |
| [0010](0010-llm-hosting.md) | Hosting the local LLM on the dedicated AI node | Proposed | 2026-11-03 |
| [0011](0011-cve-source.md) | Source of CVE exposure data | Accepted | 2026-11-04 |
| [0012](0012-storage-and-retention.md) | Storage and retention: hot, cold and inventory history | Proposed | 2026-11-03 |
| [0013](0013-remote-commands.md) | Remote commands without SSH | Proposed | 2026-11-03 |
| [0014](0014-authentication-and-roles.md) | Web authentication: local accounts and roles | Proposed | 2026-11-03 |
| [0015](0015-increment-order.md) | Order of the increments up to V1 | Proposed | 2026-11-03 |
