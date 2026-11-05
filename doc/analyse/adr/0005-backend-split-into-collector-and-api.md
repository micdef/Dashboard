# ADR-0005: Backend split into a Collector and an API process

| Field | Value |
|---|---|
| Status | Accepted |
| Date | 2026-11-03 (proposed), 2026-11-04 (accepted) |
| Decider | Michaël Defraene (product owner, architect) |
| Drafted by | Claude (main developer); the decisions had a second reading by the architecture review agent during scoping |

## Context

The initial outline split the Backend into a collector (receives data from agents) and an API (serves the web site). Both use the same domain core (ADR-0006): host state, rules, command lifecycle. Commands created from the web site must reach the agent streams, which are held by the process that agents connect to.

The first draft of this ADR proposed one process with two endpoints. The product owner chose two processes instead, so that a failure of one side does not take the other down and the data gateway sockets `ingest` and `manage` (ADR-0004) separate something real.

## Decision

- The **Backend** is a tier made of **two processes**, each a separate systemd service under its own system user:

| Process | Serves | Listens on | Uses the data gateway socket |
|---|---|---|---|
| **Collector** | agents | mTLS over TCP, on the interface facing the supervised machines | `ingest` |
| **API** | the web site | a Unix domain socket | `manage` |

- Both processes use the same domain core, the Fleet core (ADR-0006).
- Responsibilities:
  - Collector: agent authentication (ADR-0017), ingestion of logs, heartbeats and inventory, host state, real-time rules, the agent command streams and the delivery of commands.
  - API: every service the web site uses, including the relay of account operations (ADR-0014), user authorisation per role, and the creation of commands.
  - Where ONNX scoring and LLM calls run is decided with the AI pipeline (ADR-0009).
- **Commands**: the database (through the data gateway) stays the **source of truth**.
  1. The API records the command through the data gateway.
  2. The API notifies the Collector, over gRPC on a Unix domain socket, with the **command identifier only**.
  3. The Collector reads the command through the data gateway and delivers it on the agent's stream.
  4. A **periodic check** by the Collector catches any notification that was lost, so no command waits silently.
- The hierarchy of ADR-0004 still holds: the web site talks only to the API, and only the Backend processes talk to the data gateway.

## Alternatives considered

- **One process with two endpoints.** Rejected by the product owner: a crash or an overload of one side would take down the other, and the `ingest` and `manage` sockets would protect nothing.
- **The Collector polls the database for pending commands, without notification.** Rejected: every command waits up to the polling interval, and the database is queried permanently.

## Consequences

### Positive

- A failure of the Collector does not take down the web site, and the reverse (commands then wait until the Collector is back, or expire, ADR-0013).
- The `ingest` and `manage` sockets now separate the Collector's rights from the API's rights.
- Each process only exposes its own services: agents cannot reach the web site's services, and the web site cannot reach the agent services.

### Negative

- One more service to deploy, monitor and secure, and one more internal link (API → Collector).
- The notification socket of the Collector must accept only the API's group.
- Users are authenticated in the API and agents in the Collector: two places, for two different kinds of callers.

## References

- ADR-0004, ADR-0006, ADR-0009, ADR-0013, ADR-0014, ADR-0017.
