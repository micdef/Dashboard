# Review of 2026-11-06: roles, rights and Agent CA anchor sets

Readings of the decisions that came out of the use case overview of V1 ([use cases](../uml/v1/use-cases.md)), and of the first draft of ADR-0026. Every finding was checked by the main developer before it was acted on; the decisions were taken by the product owner.

## 1. Second reading before the decisions (Yoda, architecture)

The architecture review agent read the nine open points of the use case overview without seeing the main developer's recommendations. Main results:

- the rights of the roles are still in force by inheritance (ADR-0023 keeps ADR-0019, which keeps ADR-0014): the gap is in the documentation, not in the decisions;
- each user already sets up their own second factor (ADR-0019); the real gap is that changing it needs no step-up;
- the list of keys of ADR-0022 misses the Agent CA key, the package signing keys and the recorded head of the audit chain;
- the Agent CA re-key has two actors, and the trust stores must accept a set of anchors from 0.1;
- one new ADR is enough for the rights, the alerts, the monitor tokens, the second factor and the keys.

The product owner decided the nine points on 2026-11-06, mostly as recommended by both readings. The result was the first draft of ADR-0026.

## 2. Team review of the first draft of ADR-0026

| Agent | Blocking | Important | Minor |
|---|---|---|---|
| Jocasta Nu | 1 | 7 | 3 |
| Boba Fett | 2 | 6 | 6 |
| Ackbar | 0 | 6 | 6 |
| C-3PO | 0 | 6 | 6 |
| Cassian | 0 | 0 | 0 |

### Blocking findings

| Id | Finding | Outcome |
|---|---|---|
| J-B1 | Reading "each channel trusts exactly one CA" (ADR-0021) as "one CA role per channel" changes an accepted decision; a complement cannot do it. | Product owner: the anchor sets and the complete re-key move to **ADR-0027, which supersedes ADR-0021**. |
| F-B1 | "Retire a host without an active certificate" also accepts an expired certificate: a compromised host whose agent was stopped could leave the monitoring without revocation or alert (also Ackbar I-3). | Product owner: only a host **revoked explicitly** can be retired; a connection attempt from a retired host raises a Warning; no token for a retired host, which comes back under a new `HostId`. |
| F-B2 | The end of a silence is undefined: an open alert is never sent again (ADR-0022), and the resolution can be lost. | Product owner: silence per instance, Warning only, limits in the host's configuration file, "silenced until T by X", notified again at the end, resolution always notified. |

### Important findings, and their outcome

- **Second factor** (Ackbar I-1, I-2; Jocasta Nu): recovery codes, an Admin resetting their own factor, and a replaced factor bypassed the step-up or raised no alert. Product owner: step-up for regenerating recovery codes; a recovery code is not a step-up proof except to replace a lost factor (critical alert); no self-reset for an Admin; critical alert when a factor is added or replaced on an Operator or Admin account; forced set-up at the next sign-in under the policy "required".
- **Acknowledged critical alerts** (Boba Fett I4): could stay open for ever. Product owner: a reminder at an interval set on the host.
- **AI report failures** (Boba Fett I3): product owner: explicit refusal with a retry time, failed generations do not count, refusal during training, Warning on a failed or late daily report.
- **Key backups** (Boba Fett I5; Ackbar M-3): a missed or outdated backup was not detected, and the list was incomplete. Product owner: a **key backup register** in V1; the list completed (configuration files, ADR-0025 secrets, backup certificate password, internal CA of the MSI certificate).
- **Revocation with a 5-minute proof** (Ackbar M-6): a compromised web site could revoke the whole fleet. Product owner: single-use proof for revoking and retiring.
- **Re-key lifecycle** (Ackbar I-5; Boba Fett I1, I2): old CA list, issuance by the active CA, coverage of silent hosts, destruction of the old key. Recorded in ADR-0027.
- **Audit content** (Ackbar I-6): the audit trail records token and session identifiers, never their values. Added to ADR-0026.
- **Accuracy** (C-3PO I3 to I6; Jocasta Nu): the step-up list was misquoted ("silencing" was already in ADR-0023), a reference was broken, the legal basis document was cited as if it existed, the host channel was said to drive the Enrolment service. Fixed; the last point is an open point of ADR-0027.
- **Decisions recorded on a diagram page** (C-3PO I1; Jocasta Nu): the training frequency and the increments move to the pull request description.
- **Back-references** (Jocasta Nu): "ADR-0026 complements this ADR" added to ADR-0022 to ADR-0025, as an editorial correction.

### Disputed

- Boba Fett, second reading of the use cases: "the 60-day alert is not in ADR-0021". Disputed: ADR-0021 line 30 states it.

### Not verified

- That `X509ChainPolicy.CustomTrustStore` accepts several independent roots, on Linux and Windows, and Kestrel's behaviour on its own (Cassian, Ackbar): Microsoft Learn documents a collection, nothing more. ADR-0027 requires a test at the start of 0.1.
