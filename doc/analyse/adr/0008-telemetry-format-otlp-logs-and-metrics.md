# ADR-0008: Telemetry format: the OTLP logs and metrics data models

| Field | Value |
|---|---|
| Status | Accepted |
| Date | 2026-11-03 (proposed), 2026-11-04 (accepted) |
| Decider | Michaël Defraene (product owner, architect) |
| Drafted by | Claude (main developer); the decisions had a second reading by the architecture review agent during scoping |

## Context

Agents read logs from very different sources: journald, Nginx log files, the Windows Event Log, and custom files described by an XML parser definition. They also measure the load of each machine (processor, memory, disks). The Collector needs one representation for storage, rules and AI preparation. The agents are in-house .NET Worker Services, not the OpenTelemetry Collector.

In the OpenTelemetry Protocol (OTLP), the export requests carry only the telemetry (`resource_logs`, `resource_metrics`): they have **no batch or request identifier**. The response only carries a `partial_success` field, with the number of rejected items and an optional message (verified in `opentelemetry-proto`).

## Decision

### Format and services

- **Logs** travel from the agent to the Collector in the **OTLP logs data model**, through `opentelemetry.proto.collector.logs.v1.LogsService/Export`.
- **Metrics** travel in the **OTLP metrics data model**, through `opentelemetry.proto.collector.metrics.v1.MetricsService/Export`.
- Both services are generated from a **pinned version** of the `opentelemetry-proto` definitions and **implemented by the Collector itself**. We do not know of a ready-made OTLP server for .NET (**not verified**).
- **Metrics in V1**: processor usage, memory usage and disk usage per file system, for every supervised machine. Their names follow the OpenTelemetry semantic conventions for system metrics (exact names to be confirmed against the pinned version). The collection interval is set in configuration.
- Heartbeats, inventory and commands use project-specific contracts (ADR-0004). Traces are not covered by this decision.

### Identity

- The machine identity comes from the client certificate (ADR-0021), not from the OTLP resource attributes. If the attributes contradict the certificate-derived machine, the mismatch is logged and shown, and **the certificate always wins**.

### Duplicates

- The agent sends a **batch identifier in gRPC metadata** (a request header), without changing the OTLP message. The Collector records the pair (`HostId`, batch identifier) through the data gateway and ignores a batch it has already stored. Combined with the acknowledgement after storage (ADR-0004), a batch is stored once even when the agent retries.

### Errors

- A batch above the maximum message size is **split by the agent**.
- Invalid records are rejected through `partial_success` (`rejected_log_records`, `rejected_data_points`); the agent counts them and the dashboard shows them. They are never resent forever.
- An error that retrying cannot fix (validation, size after splitting) **sets the batch aside** on the agent and logs it, so the buffer behind it keeps flowing.

## Alternatives considered

- **A project-specific telemetry message.** Rejected: the OTLP models already cover timestamps, severity, body, attributes and metric types, and keep the door open to standard OpenTelemetry senders.
- **The OpenTelemetry Collector as agent.** Rejected during scoping: the agents also carry inventory, heartbeats and commands, and building them is part of the project.
- **Duplicate detection by a hash of each record.** Rejected: an index over millions of hashes, and two identical legitimate records (two failed logins in the same second) would be merged by mistake.
- **Logs only in V1.** Rejected by the product owner: the machine page must also show the load of each machine.

## Consequences

### Positive

- Documented, widely used data models instead of in-house ones.
- A standard OpenTelemetry exporter could later send logs or metrics to the Collector, given a certificate from the Agent CA (ADR-0021).
- Retries never create duplicates, and a broken batch never blocks the others.

### Negative

- The pinned proto version must be upgraded deliberately; the OTLP models are richer than what V1 uses.
- The Collector must map both models to the storage model (ADR-0012), with tests.
- A standard OpenTelemetry exporter would not send our batch identifier; duplicate detection then does not apply to it.
- Metrics add a data volume and a storage design of their own (ADR-0012) and work to the increments (ADR-0015).

## References

- OpenTelemetry, `opentelemetry-proto`: `opentelemetry/proto/collector/logs/v1/logs_service.proto` and `opentelemetry/proto/collector/metrics/v1/metrics_service.proto` (consulted 2026-11-04).
- ADR-0004, ADR-0012, ADR-0015, ADR-0021.
