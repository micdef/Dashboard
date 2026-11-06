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

## 3. Second reading of ADR-0026 and ADR-0027

All findings of the first review were fixed or decided. The second reading (Jocasta Nu, Ackbar, Boba Fett, C-3PO, Cassian) found no blocking point, and new important points:

| Finding | Outcome |
|---|---|
| ADR-0027 kept the decisions of ADR-0021 without listing them, so pages cited ADR-0027 for rules it did not contain (Jocasta Nu, C-3PO, Cassian) | a "Kept from ADR-0021" section, as ADR-0021 did for ADR-0017 |
| About twenty references to ADR-0021 in accepted ADRs, without "superseded" (Jocasta Nu) | product owner: one rule in the ADR index, "a reference to a superseded ADR is read as a reference to the ADR that supersedes it" |
| Only acknowledged critical alerts were notified again (Boba Fett) | product owner: every open critical alert is notified again; its age is shown |
| The start-up check of the anchor set refused only the Server CA (Ackbar) | product owner: one or two anchors, self-signed CAs, known Agent CAs only, no retired CA; a bounded overlap, alerted |
| Regenerating recovery codes raised no alert (Ackbar) | product owner: critical alerts for regeneration and sign-in by recovery code on Operator and Admin accounts; remaining codes invalidated after a lost-factor reset |
| A retired host that tries to connect raised only a Warning, which an Operator can silence (Ackbar) | product owner: a critical alert |
| ADR-0025 accepted a token for a host whose certificate had only expired (Ackbar, a defect older than this review) | product owner: fixed in ADR-0026, a token for an existing host needs the revoked state |
| "Nothing can still sign for the old CA" was false: agents keep the old anchor until the next package, and older VM images hold the key (Ackbar) | product owner: sentence corrected, the coverage report follows the removal of the old anchor, older images stay encrypted and are destroyed at the end of their retention |
| Configuration values without a stated behaviour when missing (Boba Fett) | no default: the service refuses to start (rule of CLAUDE.md, section 4) |
| Key register: hashes of low-entropy secrets, no "destroyed" state, self-declared (Ackbar, Boba Fett) | public key fingerprint or version identifier, never a hash; a "destroyed" state; the limit stated |
| Revoked hosts blocked the retirement of the old CA; coverage self-reported (Boba Fett) | revoked hosts are out of the report, the check uses fresh data, the limit stated |
| The backup guide did not list the new keys (Ackbar) | `doc/adm/backup-and-recovery.md` updated |
| Accuracy and language (C-3PO, Jocasta Nu) | fixed: the Operator row, the tightening of ADR-0023 stated, the stored secrets named, the register paragraph split |

## 4. Third reading

The third reading (Jocasta Nu, Ackbar, Boba Fett, C-3PO) found every earlier finding fixed, no blocking point, and:

| Finding | Outcome |
|---|---|
| ADR-0026, a complement, tightens rules of ADR-0022, ADR-0023 and ADR-0025 (Jocasta Nu, Boba Fett, C-3PO) | product owner: a rule in the ADR index, "a complement may tighten a rule, if it says so and names it; it never weakens one", and a "tightened by" reference in the tightened ADRs |
| Anyone holding the (public) certificate of a retired host could raise the critical alert at will (Ackbar) | product owner: one open alert per retired host with a counter, refused attempts aggregated and rate-limited, the limit of the alert stated |
| A symmetric key replaced without a new version would not be detected (Ackbar) | product owner: an HMAC with a dedicated register key, backed up offline |
| The reading rule for references to superseded ADRs broke references made on purpose (Jocasta Nu) | the rule names the last ADR of the chain and excludes references made on purpose |
| Minor points: the "Kept" list not exhaustive, the runtime floor, the anchor check on the agent, an unreadable re-key state, the alerts expected after a long outage, the empty lines in a diagram | fixed |

Deferred: splitting two long bullets of the use case page (C-3PO, minor), with the next change of that page.
