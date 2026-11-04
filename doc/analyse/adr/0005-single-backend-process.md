# ADR-0005: One Backend process for the collector and the API

| Field | Value |
|---|---|
| Status | Proposed |
| Date | 2026-11-03 |
| Decider | Michaël Defraene (product owner, architect) |
| Drafted by | Claude (main developer); the decisions had a second reading by the architecture review agent during scoping |

## Context

The initial outline split the Backend into a collector (receives data from agents) and an API (serves the web site). Both use the same domain core (ADR-0006): host state, rules, command lifecycle. Commands issued from the web site must reach the agent streams held by the collector.

## Decision

- The collector and the API run in **one Backend process**, with **two listening endpoints**:
  - mTLS over TCP, on the interface facing the supervised machines, for agents;
  - a Unix domain socket for the web site.
- The **database stays the source of truth for commands** (state, audit). In-memory state is only a cache of open agent streams. This keeps a later split into two processes possible without changing the domain.

## Alternatives considered

- **Two processes (collector, API).** Rejected for V1: it requires an extra channel to hand commands from the API to the collector, doubles deployment work, and brings no measurable benefit for a fleet of six machines.

## Consequences

### Positive

- One deployment unit, one place where the domain core runs.
- A command created through the API reaches the agent stream without an extra hop.

### Negative

- A crash or an overload of the collector also takes down the API, and the reverse.
- Each endpoint must only expose its own services: an agent must not reach the web site's services, and the web site must not reach the agent services. A test must check this.
- The two data gateway sockets cannot separate the collector from the API (ADR-0004).

## Open points

1. Confirm one Backend process with two endpoints.

## References

- ADR-0004, ADR-0006, ADR-0013.
