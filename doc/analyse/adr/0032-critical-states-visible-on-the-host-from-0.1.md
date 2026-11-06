# ADR-0032: Critical states visible on the host from 0.1

| Field | Value |
|---|---|
| Status | Accepted |
| Date | 2026-11-06 |
| Decider | Michaël Defraene (product owner, architect) |
| Drafted by | Claude (main developer), after the team review of the sequence diagrams of 0.1; second reading by the architecture review agent before the decisions on how the states are reported, read and ended |
| Classification | Security-sensitive [SEC] |
| Complements | ADR-0022, ADR-0023, ADR-0025, ADR-0030 |

## Context

The alert channel arrives in 0.2 (ADR-0025). Increment 0.1 already holds states that need a human: a host marked suspect (ADR-0030), a token presented again once used (ADR-0031), an Agent CA revocation list that cannot be regenerated, or that the Collector cannot use (ADR-0029), a certificate signed but not registered, an Enrolment service server certificate that expires or cannot be read (ADR-0030). In 0.1, the only place that shows them is the development status page (ADR-0015), which nobody may be watching.

The team review of the first draft of this ADR found three gaps:

- the Enrolment service detects some of these states, but has no access to the data gateway, and its socket to the API is limited to three services (ADR-0017, kept by ADR-0030);
- the host channel's group holds every host right (ADR-0023): a monitoring job that runs the status command under another account would need that group;
- some states are events, with no condition that clears them, so the command would fail for ever.

## Decision

### Critical states and their record

- Each state of the list above is a **critical state**, and so is every critical state that a later ADR adds. The service that detects it logs it at critical level, with a fixed `EventId` from the range of its class (CLAUDE.md, section 7).
- The state is recorded through the data gateway, with the time it began and, once it is over, the time it ended:
  - the Collector records its own states through its `ingest` socket (ADR-0029);
  - the API records its own states, and those of the Enrolment service;
  - repeated occurrences of the same state, for the same host or the same token, increment the counter of **one** open state, as for retired hosts (ADR-0026).

### States of the Enrolment service

- The Enrolment service writes a **state file**: atomically (a temporary file, then a rename), readable by the API's group, carrying the time it was written, refreshed at an interval set in its configuration, as it writes its public certificate file (ADR-0030). Nothing is added to its socket to the API.
- The API reads it when it changes and at that interval, after bounding its size, and records the states it holds. A state file that is missing, older than its interval allows, too large or invalid is "unknown", which is itself a critical state.
- The API also works out by itself every state it can see without the Enrolment service: the age, the `nextUpdate` and the CRL number of the Agent CA revocation list, read from the list file; and the issuance tickets that expired without a registration (ADR-0030). The state file adds only what the API cannot see: a list refused because it would hold fewer serials (ADR-0029), and the serial of a certificate signed but not registered.
- Once the API knows the serial of a certificate signed but not registered, it records it in the issuance register as "signed, not registered" and revokes it, as a refused certificate (ADR-0030).
- **Limit**: a compromised Enrolment service can hide the states that only it knows. The states that the API works out itself cannot be hidden by it.

### The host command "status"

- A host command **"status"**, on the host channel of the API (ADR-0023), lists the active critical states and **exits with an error** when at least one is active. It also fails when a state cannot be read (the data gateway cannot be reached, the state is "unknown"), and when the API itself cannot be reached.
- It is read-only. Its output is limited to the state, the time it began, its counter and the `HostId` concerned: no hash, no serial of an active certificate, no token identifier, no source address. Its output may end in the system journal, which accounts other than root can read.
- The host administrator runs it by hand, or from a systemd timer that runs it **as root**; scheduling it is the administrator's choice, not Dashboard's (CLAUDE.md, section 9). The group of the host channel is **never** given to a monitoring account, because it holds every host right (ADR-0023). A separate read-only channel for a monitor that is not root is recorded in the [future improvements](../future-improvements.md).

### How a critical state ends

- A **condition** state (a revocation list that cannot be regenerated or used, a state file or a certificate file that cannot be read, a certificate that expires) ends when the next evaluation finds the condition cleared.
- A **suspect host** ends with its explicit revocation (ADR-0030).
- A **certificate signed but not registered** ends once its serial is revoked and on a published list.
- An **event** state (a token presented again once used, expired or invalidated, ADR-0031; a refused certificate that reuses an active serial, ADR-0030) ends only through a host command **"resolve"**, with a reason, recorded with the host account (audited from 0.2). It is available on the host channel only, never in the web site. A new occurrence after the resolution opens a new state.
- "Resolve" is not "acknowledge": acknowledging an alert changes its display only, and the alert stays open (ADR-0026). From 0.2, resolving a critical state resolves its alert, which is notified (ADR-0026).

### From 0.2, and tests

- From 0.2, each critical state also raises its alert through the alert channel (ADR-0025). The commands stay.
- A test proves, for each critical state, that it makes "status" fail, and that "status" succeeds once the state is over.

## Alternatives considered

- **The development status page only, until 0.2.** Rejected: a suspect host or a stopped revocation list could stay unseen for days.
- **The alert channel in 0.1.** Not adopted: it belongs to 0.2 (ADR-0025), and would make 0.1 larger.
- **The states of the Enrolment service only logged in 0.1.** Rejected: they would be absent from "status", which ADR-0029 requires them in.
- **A fourth service on the socket of the Enrolment service, to report its states.** Rejected: it widens a surface that ADR-0017 limits.
- **A separate read-only channel for "status" in 0.1.** Not adopted for 0.1: one more surface, group and set of tests; the timer runs as root instead.
- **Event states that end after a delay set in the configuration.** Rejected: a stolen token could disappear from view without anyone seeing it.
- **Event states ended by "acknowledge".** Rejected: it would close a critical alert by acknowledging it, which ADR-0026 does not allow.

## Consequences

### Positive

- From 0.1, one command answers "does anything need me?", and a root timer can watch it.
- The same states feed the alerts of 0.2, with no second list.
- A stopped or silent Enrolment service shows as "unknown", never as "all clear".

### Negative

- Two more host commands, one more file written by the Enrolment service, and one more record of the states, to build in 0.1.
- In 0.1, a critical state is seen only if someone runs the command or watches its timer.
- A compromised Enrolment service can hide the states that only it knows.

## References

- ADR-0004, ADR-0015, ADR-0017 (superseded), ADR-0022, ADR-0023, ADR-0025, ADR-0026, ADR-0029, ADR-0030, ADR-0031.
- [Review report of the 0.1 identity sequences](../reviews/2026-11-06-identity-sequences-of-0.1.md).
