# ADR-0013: Remote commands without SSH

| Field | Value |
|---|---|
| Status | Proposed |
| Date | 2026-11-03 |
| Decider | Michaël Defraene (product owner, architect) |
| Drafted by | Claude (main developer); the decisions had a second reading by the architecture review agent during scoping |
| Classification | Security-sensitive [SEC] |

## Context

The initial outline planned to send commands to the machines through SSH. Remote command execution is the most dangerous feature of the system: a compromised Backend, web site process or user account could take control of every machine.

## Decision

- **No SSH.** The agent opens a **bidirectional gRPC stream** to the Backend over mTLS (ADR-0004, ADR-0017); commands arrive on this stream. No inbound port is opened on the machines.
- Commands are **typed messages from a catalogue** (for example "restart service X"), never free text.
- Each machine has a **local allowlist** that only root (or an administrator on Windows) can modify. A command outside the allowlist is refused by the agent, whatever the Backend sends.
- The agent runs fixed commands directly, **never through a shell**: on Linux with limited sudoers rules, on Windows with `ServiceController`. Each execution has a **maximum duration** and a **capped output size**.
- Every command (request, delivery, result) is written to an **immutable audit trail**.
- A command not delivered in time (machine offline) **expires**; it is never executed late.
- The roles allowed to issue commands are decided in ADR-0014.
- **Signing of commands** (so that the agent can verify the issuer independently of the Backend) is planned for **V2**.

## Alternatives considered

- **SSH from the Backend.** Rejected: it requires an inbound port and credentials on every machine, and a shell on the remote side.
- **Free-text commands.** Rejected: the allowlist could not be checked reliably.

## Consequences

### Positive

- In V1, the commands a compromised Backend can run are limited to those both in the catalogue and in the local allowlist. (A compromised Backend can still read the collected logs and falsify what the dashboard shows.)
- Every action is traceable.

### Negative

- Until command signing (V2), the agent trusts any allowed command coming from an authenticated Backend; the allowlist and the agent's validation of the Collector's certificate (ADR-0017) are the main safeguards.
- Each new command type needs a catalogue entry, an agent implementation and an allowlist entry.
- If the remote commands increment is dropped (ADR-0015), the stream, the allowlist and the command lifecycle must not ship half-finished.

## Open points

1. Raised in review, proposed: a command carries a unique identifier and an expiry timestamp, and the agent refuses expired or already-seen identifiers (at most once); all outcomes are audited (executed, refused, timed out, expired, result lost), with the requesting user and the role used.
2. Raised in review, proposed: sudoers entries are exact commands without wildcards, and typed parameters (such as a service name) are matched exactly against the allowlist before use.

## References

- ADR-0004, ADR-0014, ADR-0015, ADR-0017.
