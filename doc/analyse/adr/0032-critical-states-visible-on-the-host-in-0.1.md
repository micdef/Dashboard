# ADR-0032: Critical states visible on the host from 0.1

| Field | Value |
|---|---|
| Status | Accepted |
| Date | 2026-11-06 |
| Decider | Michaël Defraene (product owner, architect) |
| Drafted by | Claude (main developer), after the team review of the sequence diagrams of 0.1; second reading by the architecture review agent before the decisions |
| Complements | ADR-0022, ADR-0023, ADR-0025 |

## Context

The alert channel arrives in 0.2 (ADR-0025). Increment 0.1 already holds states that need a human: a host marked suspect (ADR-0030), a token presented again once used (ADR-0031), an Agent CA revocation list that cannot be regenerated, or that the Collector cannot use (ADR-0029), a certificate signed but not registered, an Enrolment service server certificate that expires or cannot be read (ADR-0030). In 0.1, the only place that shows them is the development status page (ADR-0015), which nobody may be watching.

## Decision

- Each of these states is a **critical state**. The service that detects it:
  - logs it at critical level, with a fixed `EventId` from the range of its class (CLAUDE.md, section 7);
  - records it through the data gateway, with the time it began and, once it is over, the time it ended.
- A host command **"status"**, on the host channel of the API (ADR-0023), lists the active critical states and **exits with an error** when at least one is active. A state that cannot be read, because the data gateway cannot be reached, is reported as "unknown" and also makes the command fail.
- The host administrator runs it by hand, or from a systemd timer, which shows its failures (CLAUDE.md, section 9); scheduling it is the administrator's choice, not Dashboard's.
- From 0.2, each critical state also raises its alert through the alert channel (ADR-0025). The command stays.
- A test proves, for each critical state, that it makes the command fail, and that the command succeeds once the state is over.
- Every critical state added later by an ADR joins this list.

## Alternatives considered

- **The development status page only, until 0.2.** Rejected: a suspect host or a stopped revocation list could stay unseen for days.
- **The alert channel in 0.1.** Not adopted: it belongs to 0.2 (ADR-0025), and would make 0.1 larger.

## Consequences

### Positive

- From 0.1, one command answers "does anything need me?", and a timer can watch it.
- The same states feed the alerts of 0.2, with no second list.

### Negative

- One more host command, and one more record of the states, to build in 0.1.
- In 0.1, a critical state is seen only if someone runs the command or watches its timer.

## References

- ADR-0015, ADR-0022, ADR-0023, ADR-0025, ADR-0029, ADR-0030, ADR-0031.
- [Review report of the 0.1 identity sequences](../reviews/2026-11-06-identity-sequences-of-0.1.md).
