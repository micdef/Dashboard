# ADR-0015: Order of the increments up to V1

| Field | Value |
|---|---|
| Status | Proposed |
| Date | 2026-11-03 |
| Decider | Michaël Defraene (product owner, architect) |
| Drafted by | Claude (main developer); the decisions had a second reading by the architecture review agent during scoping |

## Context

V1 must be a complete, working system with integrated AI, presented to the training jury in March 2027. The work is delivered in small visible increments. If time runs short, something must be dropped, and the choice must be made in advance.

## Decision

| Increment | Content |
|---|---|
| 0.1 | Heartbeat end to end: database, data gateway, Backend, Linux agent, status page |
| 0.2 | Linux logs, Linux metrics (processor, memory, disks; ADR-0008) and the machine page |
| 0.3 | AI: .NET preparation, Python training, ONNX inference, LLM report |
| 0.4 | Windows agent (logs and metrics) and XML log parser |
| 0.5 | Rules and CVE |
| 0.6 | Remote commands |

- The AI comes in **0.3**, early, because it is the core of the training evaluation. In 0.3 the facts given to the LLM are counts and anomalies only; rule hits are added in 0.5.
- If time runs short, **remote commands are dropped, not the AI**.
- The detailed breakdown of each increment into features is done by the planning step, not in this ADR.

## Alternatives considered

- **AI last.** Rejected: the part evaluated by the jury would be the most exposed to lack of time.

## Consequences

### Positive

- Each increment produces something visible and testable.
- The AI is built on real Linux logs collected since 0.2.

### Negative

- The AI is trained on a few weeks of data from Linux machines only; Windows logs and metrics arrive in 0.4 and require retraining. Until then, Windows sources are flagged "not covered by the model" (ADR-0009).
- The V1 model uses metrics as input (ADR-0009), so the Linux metrics of 0.2 must be collected long enough before 0.3 to be usable for training.

## Open points

1. The order 0.1, 0.2, 0.4, 0.5, 0.6 around the AI (0.3 is already decided).
2. Metrics placed in 0.2 for Linux and in 0.4 for Windows (added on 2026-11-04 after ADR-0008), rather than in an increment of their own.

## References

- ADR-0009, ADR-0013.
