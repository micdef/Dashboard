# ADR-0015: Order of the increments up to V1

| Field | Value |
|---|---|
| Status | Accepted |
| Date | 2026-11-04 |
| Decider | Michaël Defraene (product owner, architect) |
| Drafted by | Claude (main developer); the decisions had a second reading by the architecture review agent during scoping |

## Context

V1 must be a complete, working system with integrated AI, presented to the training jury in March 2027. The work is delivered in small visible increments. If time runs short, something must be dropped, and the choice must be made in advance.

The AI model is trained on data collected by the system itself (ADR-0009), so collection must start early. Machine identity (ADR-0017) is needed by the first heartbeat, and accounts and the audit trail (ADR-0013, ADR-0014) are needed as soon as the web site shows sensitive data.

## Decision

| Increment | Content |
|---|---|
| 0.1 | Heartbeat end to end: database, VBE, Backend, Linux agent, **Enrolment service** (two certificate authorities, token and CSR, renewal, revocation lists; ADR-0017), status page |
| 0.2 | Linux logs, Linux metrics (processor, memory, disks; ADR-0008), hot storage with its maintenance job and the archive export to the cold storage (ADR-0012, ADR-0018), the machine page, and **accounts, roles and the audit trail** (ADR-0013, ADR-0014: Identity stores, first Admin command, two-factor policy, hash chain) |
| 0.3 | AI: LLM benchmark (ADR-0010), .NET preparation with pseudonymisation, Python training, ONNX inference, LLM report |
| 0.4 | Windows agent (logs and metrics) and XML log parser |
| 0.5 | Rules and CVE |
| 0.6 | Remote commands |

- **Machine identity is built properly from 0.1**: there are never test certificates to replace later. It is security-sensitive code, written by the main developer and reviewed by the security agents.
- **Authentication arrives in 0.2**, when the web site shows sensitive data (logs) for the first time. The status page of 0.1 is for development only and is never deployed elsewhere.
- **The audit trail ships in 0.2**, because the creation of the first Admin and configuration changes are audited (ADR-0014). It does not depend on remote commands.
- The AI comes in **0.3**, early, because it is the core of the training evaluation. In 0.3 the facts given to the LLM are counts and anomalies only; rule hits are added in 0.5.
- **Metrics** arrive with the logs of each platform: Linux in 0.2 (the V1 model uses them, ADR-0009), Windows in 0.4.
- If time runs short, **remote commands are dropped, not the AI**. The audit trail stays, since it is delivered in 0.2.
- The detailed breakdown of each increment into features is done by the planning step, not in this ADR.

## Alternatives considered

- **AI last.** Rejected: the part evaluated by the jury would be the most exposed to lack of time.
- **Hand-made certificates in 0.1, enrolment in 0.2.** Rejected: a temporary mechanism to replace later.
- **Accounts and audit in an increment of their own before the logs.** Rejected: the AI would move to 0.4 and the collection of training data would start later.
- **Accounts and audit in 0.1.** Rejected: a very heavy first increment before anything visible works.
- **Rules and CVE before the Windows agent.** Rejected: rule hits would reach the LLM earlier, but Windows would join the model later.
- **Metrics in an increment of their own.** Rejected: the V1 model needs the Linux metrics from the start of the collection.

## Consequences

### Positive

- Each increment produces something visible and testable.
- The AI is built on real Linux logs and metrics collected since 0.2.
- Machine identity and the audit trail are never temporary.

### Negative

- 0.1 and 0.2 are the largest increments; the planning step must split them into small visible steps. 0.1 contains security-sensitive code (the certificate authorities).
- The AI is trained on a few weeks of data from Linux machines only; Windows logs and metrics arrive in 0.4 and require retraining. Until then, Windows sources are flagged "not covered by the model" (ADR-0009).
- The Linux logs and metrics of 0.2 must be collected long enough before 0.3 to be usable for training: 0.2 must be deployed on the home lab as soon as it works, not at the end of its polishing.

## Open points

None.

## References

- ADR-0008, ADR-0009, ADR-0010, ADR-0012, ADR-0013, ADR-0014, ADR-0017, ADR-0018.
