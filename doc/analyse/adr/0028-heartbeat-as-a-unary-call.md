# ADR-0028: Heartbeat as a unary call, separate from the command stream

| Field | Value |
|---|---|
| Status | Accepted |
| Date | 2026-11-06 |
| Decider | Michaël Defraene (product owner, architect) |
| Drafted by | Claude (main developer), from the sequence diagrams of 0.1; second reading by the architecture review agent before the decision |
| Complements | ADR-0004 |

## Context

ADR-0004 states that the agent always opens the connection to the Collector, and that commands travel on a bidirectional stream opened by the agent. No ADR says how the **heartbeat** travels. Increment 0.1 delivers the heartbeat end to end, but commands arrive only in 0.6, and they are the first thing dropped if time runs short (ADR-0015).

The heartbeat must fit three rules that already exist: every gRPC call has a deadline (ADR-0004); the Collector acknowledges a batch only after the data gateway has stored it (ADR-0004); and the host page tells "down: refused" from "down: silent" (ADR-0021, kept by ADR-0027). The heartbeat also carries more fields with each increment: the days before the certificate expires (ADR-0021), the fingerprints of the Agent CA anchors (ADR-0027), the counters of each log and metric source (ADR-0022), and the clock offset used by the commands (ADR-0020, kept by ADR-0024).

## Decision

- The heartbeat is a **unary gRPC call** from the agent to the Collector, on the mTLS channel of ADR-0004. It is not sent on the command stream, in 0.1 or later.
- The agent sends it at an **interval** set in its configuration, with a random **jitter** bounded in the configuration, so that a fleet started at the same time does not call at the same moment.
- Each call has a **deadline** (ADR-0004). The answer is the acknowledgement: the Collector answers only after the data gateway has stored the heartbeat, and answers a retryable gRPC status otherwise.
- A heartbeat describes the **current state** of the machine; it is not a batch in the sense of ADR-0004, whose data the agent keeps until it is acknowledged. A heartbeat that failed is not replayed: the next one replaces it. The counters it carries are kept on disk by the agent (ADR-0022), so a failed call loses nothing.
- After a failed call, the agent retries at the next interval, with a back-off bounded by its configuration; it never retries without a bound (CLAUDE.md, section 4). It logs the number of consecutive failures, at a severity that rises with it.
- The size of the message is bounded on the Collector (`MaxReceiveMessageSize`, ADR-0017, kept by ADR-0027). The HTTP/2 limits of Kestrel that bound the cost of many short calls (`KestrelServerOptions.Limits.Http2`: concurrent streams per connection, header size, keep-alive pings) are set in the Collector's configuration (Microsoft Learn, *Security considerations for the ASP.NET Core Kestrel web server*, aspnetcore-10.0).
- **Host state**: the Fleet core (ADR-0006) flags a host:
  - "down: silent" when no heartbeat attempt reached the Collector for a number of intervals set in the configuration;
  - "down: refused" when its attempts were refused, with the reason (ADR-0021): certificate not mapped, revoked, or host suspect (ADR-0029, ADR-0030);
  - "unknown" for the whole fleet when the Collector cannot be reached (the API checks its health service, ADR-0004), when the data gateway cannot be reached, or when the Collector has no usable revocation list (ADR-0022, ADR-0029). A host is never shown "down: refused" because of a fault of Dashboard itself.

  Time goes through `TimeProvider` (ADR-0006).
- From 0.6, the command stream (ADR-0004) is a separate call on the same channel. Its state ("command channel connected") is shown as its own indicator, never as the liveness of the host.

## Alternatives considered

- **Heartbeats on the bidirectional stream from 0.1.** Rejected, for four reasons:
  - a long-lived stream cannot carry the per-call deadline of ADR-0004;
  - each heartbeat would need an acknowledgement message correlated with it, instead of the gRPC status;
  - a half-open connection looks alive until HTTP/2 keep-alive pings detect it, which adds settings and tests;
  - if 0.6 is dropped, the heartbeat would depend on a stream that must not ship half-finished (ADR-0013, kept by ADR-0020 and ADR-0024).
- **A client-streaming call.** Not adopted: it has the deadline problem of the stream, for no gain.
- **Hints in the answer** (for instance "renew now"). Not adopted for 0.1: no need for it yet.

## Consequences

### Positive

- The heartbeat is acknowledged, bounded and retried like any other call, with the gRPC status as its error.
- "Down: silent" is the absence of calls since a given time, which a test drives with a fake clock.
- Dropping the commands of 0.6 leaves the heartbeat untouched.

### Negative

- One more call per interval from each agent, and one more write per call through the data gateway.
- The heartbeat grows with each increment; its bounded size must follow the fields added.
- That the unary call and the stream of 0.6 share one HTTP/2 connection in grpc-dotnet is **not verified**: to be checked in 0.6.

## References

- Microsoft Learn, *Security considerations for the ASP.NET Core Kestrel web server*, "HTTP/2 security" (aspnetcore-10.0), consulted on 2026-11-06.
- ADR-0004, ADR-0006, ADR-0013 (superseded), ADR-0015, ADR-0017 (superseded), ADR-0020 (superseded), ADR-0021 (superseded), ADR-0022, ADR-0024, ADR-0027, ADR-0029, ADR-0030.
- [Review report of the 0.1 identity sequences](../reviews/2026-11-06-identity-sequences-of-0.1.md).
