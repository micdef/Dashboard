# ADR-0008: Log format: the OTLP logs data model

| Field | Value |
|---|---|
| Status | Proposed |
| Date | 2026-11-03 |
| Decider | Michaël Defraene (product owner, architect) |
| Drafted by | Claude (main developer); the decisions had a second reading by the architecture review agent during scoping |

## Context

Agents read logs from very different sources: journald, Nginx log files, the Windows Event Log, and custom files described by an XML parser definition. The Backend needs one representation for storage, rules and AI preparation. The agents are in-house .NET Worker Services, not the OpenTelemetry Collector.

## Decision

- Logs travel from the agent to the Backend in the **logs data model of the OpenTelemetry Protocol (OTLP)**, through the standard OTLP logs export gRPC service, generated from a **pinned version** of the `opentelemetry-proto` definitions.
- The Backend **implements the generated service itself**. We do not know of a ready-made OTLP server for .NET (**not verified**).
- The machine identity comes from the client certificate (ADR-0007), not from the OTLP resource attributes.
- Heartbeats, inventory and commands use project-specific contracts (ADR-0004). Metrics and traces are not covered by this decision.

## Alternatives considered

- **A project-specific log message.** Rejected: the OTLP model already covers timestamps, severity, body and attributes, and keeps the door open to standard OpenTelemetry senders later.
- **The OpenTelemetry Collector as agent.** Rejected during scoping: the agents also carry inventory, heartbeats and commands, and building them is part of the project.

## Consequences

### Positive

- A documented, widely used data model instead of an in-house one.
- A standard OpenTelemetry exporter could later send logs to the Backend, subject to the same certificate authentication.

### Negative

- The pinned proto version must be upgraded deliberately; the OTLP model is richer than what V1 uses.
- The Backend must map the OTLP model to the storage model (ADR-0012), with tests.
- A retried batch can be stored twice and skew the counts used by the anomaly model.

## Open points

1. Confirm the OTLP logs data model and the in-house service implementation.
2. Raised in review, proposed: ingestion is idempotent per batch (batch identifier); non-retryable errors (size, validation) are logged and the batch is set aside instead of retried forever; partial rejections are counted and shown; a mismatch between the resource attributes and the certificate-derived host is logged, and the certificate always wins.

## References

- OpenTelemetry, `opentelemetry-proto` repository, logs collector service definition (exact service and file names to be confirmed when the version is pinned).
- ADR-0004, ADR-0007, ADR-0012.
