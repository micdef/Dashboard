# ADR-0026: Roles, rights and channels

| Field | Value |
|---|---|
| Status | Accepted |
| Date | 2026-11-06 |
| Decider | Michaël Defraene (product owner, architect) |
| Drafted by | Claude (main developer); second reading by the architecture review agent before the decisions |
| Classification | Security-sensitive [SEC] |
| Complements | ADR-0021, ADR-0022, ADR-0023, ADR-0025 |

## Context

The use case diagrams of V1 ([use cases](../uml/v1/use-cases.md)) showed actions that the ADRs place under step-up or on the host channel without saying **who** may perform them:

- the rights of the roles are still in force, but only by inheritance: ADR-0023 keeps every decision of ADR-0019, which keeps every decision of ADR-0014, and the right of the Reader ("sees everything, changes nothing") is written only in ADR-0014, which is superseded. No accepted ADR gathers them;
- ADR-0023 puts revoking a machine, deleting a host and silencing an alert under step-up without naming the role; "deleting a host" is defined nowhere, and a physical deletion would conflict with the audit trail (ADR-0024) and the inventory history (ADR-0012);
- ADR-0025 protects a list of critical alerts from being silenced from the web site, but ADR-0022 defines other critical alerts (backups, restore test, verifier silence, refused purge, count difference), and "silence" is not defined;
- ADR-0010 speaks of reports "requested" without saying by whom;
- ADR-0025 enrols the verifier with a "monitor" token without saying who creates it;
- ADR-0019 lets each user set up their own second factor, but the step-up list of ADR-0023 does not cover changing one's own factor, so a stolen session could replace it;
- the list of keys to back up offline in ADR-0022 misses the Agent CA key, the package signing keys and the recorded head of the audit chain;
- the Agent CA re-key of ADR-0021 has no named actor, and its overlap period needs, from the first increment, a channel that trusts two Agent CAs at once, while ADR-0017 (kept by ADR-0021) says that each channel trusts exactly one CA.

This ADR adds to the ADRs it complements, without changing their decisions.

## Decision

### Rights of the roles in the web site

The roles stay **Reader**, **Operator** and **Admin**, each one with every right of the previous one.

| Action in the web site | Reader | Operator | Admin | Step-up |
|---|---|---|---|---|
| View the fleet, machines, logs, metrics, anomalies, reports, rules, CVE, alerts, health and audit trail | yes | yes | yes | no |
| Set up, replace or remove one's own second factor | yes | yes | yes | yes |
| Request an AI report | — | yes | yes | no |
| Acknowledge an alert, of any level | — | yes | yes | no |
| Silence a Warning or Information alert for a set duration | — | yes | yes | yes |
| Issue a catalogue command | — | yes | yes | yes, single use |
| Manage accounts and roles, reset a second factor, unlock an account | — | — | yes | yes, single use |
| Create an enrolment token for an agent | — | — | yes | yes |
| Revoke a machine | — | — | yes | yes |
| Retire a host | — | — | yes | yes |
| Raise a security setting or the audit retention | — | — | yes | yes |

- **Viewing has two exceptions**: no role sees the value of an enrolment token after it was shown once at its creation, nor the sessions of other users. The audit trail, which holds user names and source addresses, is visible to every role; the legal basis document of the project justifies it (ADR-0012, GDPR).
- **AI reports**: a report is generated every day by a scheduled job, after the training window, and an Operator or an Admin may request one. Requests are rate-limited by the configuration. A request writes a report and loads the AI node, so it is a change, not a read.
- **Own second factor**: adding, replacing or removing one's own factor needs step-up. Under the policy "required for Operator and Admin" (ADR-0014), removing it is refused for these roles, and the first sign-in of such an account forces its set-up. Removing a second factor raises the critical alert of ADR-0025.
- The step-up list of ADR-0023 is extended with: changing one's own second factor; retiring a host; silencing an alert.

### Alerts: acknowledge and silence

- **Acknowledging** an alert records that someone is handling it: the alert stays open and visible, its notifications are not stopped, and the acknowledgement is audited.
- **Silencing** an alert stops its notifications for a duration set in the configuration, then they resume. It is audited.
- **No critical alert can be silenced from the web site**, whether it is in the list of ADR-0025 or not. Critical alerts can only be acknowledged there. Warning and Information alerts can be silenced by an Operator or an Admin.

### Retiring a host

- A host is never deleted. **Retiring** a host is allowed only when it has no active certificate (it was revoked first, ADR-0025). The host is then marked retired: it is no longer expected, so its silence raises no alert.
- Its data stays and follows the retention rules of ADR-0012 and ADR-0024; the audit trail is unchanged.

### Host channel

The host channel of ADR-0023 also carries:

- the creation and revocation of **"monitor" tokens and identities**, for the verifier. They are never available in the web site, because the verifier is part of the monitoring of the monitor (ADR-0022) and must not be replaceable from the most exposed process;
- the steps of the **Agent CA re-key** that run on the "app" VM (below).

### Agent CA re-key

- Two actors carry out the re-key of ADR-0021:
  - the **key holder** signs the agent package that ships the new Agent CA anchor next to the old one (ADR-0025);
  - the **host administrator** generates the new Agent CA in the Enrolment service, switches the Enrolment service to it, reads the coverage report, then retires the old Agent CA, through host commands.
- From **0.1**, the channels that trust the Agent CA (the agent towards the Enrolment service, the Collector towards the agent certificates) hold a **set of Agent CA anchors**, not a single one, so that the overlap period needs no change of the agent. Outside a re-key, the set holds one anchor. A channel never trusts the Server CA and an Agent CA together: the rule of ADR-0017, "each channel trusts exactly one CA", is read as one **CA role** per channel.
- That the .NET chain validation and Kestrel accept several anchors in `CustomTrustStore` is **not verified**; it is checked when the 0.1 work starts.

### Keys and secrets backed up offline

The list of ADR-0022 is completed with:

| Key or secret | Backed up from |
|---|---|
| Server CA key (already in ADR-0022) | 0.1 |
| Agent CA key, held online in the Enrolment service | 0.1 |
| GPG key that signs the RPM packages | 0.1 |
| HMAC key of the audit trail, key that encrypts the authenticator keys, backup encryption certificate (already in ADR-0022) | 0.2 |
| Recorded head of the audit chain (ADR-0024) | 0.2 |
| Pseudonymisation key (already in ADR-0022) | 0.3 |
| Code signing certificate of the MSI packages | 0.4 |

Each key is backed up in the increment that creates it.

## Alternatives considered

- **An editorial note in ADR-0023 restating the rights.** Rejected: adding a table of rights changes what the ADR decides, which the rule of the ADR index forbids.
- **An ADR that supersedes ADR-0023.** Rejected: far too heavy to add a table; ADR-0023 stays valid.
- **The Operator may revoke a machine.** Rejected: revocation is the counterpart of the enrolment token, which only an Admin creates, and it cuts the machine off at once.
- **Physical deletion of a host.** Rejected: it conflicts with the audit trail and the inventory history.
- **Critical alerts outside the ADR-0025 list silenced by an Admin after step-up.** Rejected: they are the alerts a malicious Admin would want to silence before acting (ADR-0025), and one rule is simpler to test than a list.
- **"Monitor" tokens also created in the web site.** Rejected: a compromised web site could replace the verifier's identity.
- **AI reports on schedule only, or requested by every role.** Rejected: no on-demand report for the people who handle incidents, or a Reader who changes something.
- **One anchor per channel until the first re-key.** Rejected: the first re-key, about two years after installation, would need a redesign of the agent.

## Consequences

### Positive

- Every action of the web site and the host channel has a role and a channel, in one table, which the tests of the authorisation can follow line by line.
- No critical alert can be cut from the web site, and the verifier's identity cannot be replaced from it.
- A stolen session cannot replace its user's second factor without the password and the current code.
- A host can leave the fleet without breaking the audit trail or the history.
- The Agent CA can be re-keyed without changing the agent's design, and the keys whose loss would force a re-enrolment of the fleet are backed up.

### Negative

- More step-up prompts for users, and one more state ("retired") in the host model.
- An Operator cannot silence a noisy critical alert: they can only acknowledge it, and the cause must be fixed or the threshold changed in the configuration.
- The trust stores of 0.1 handle a set of anchors, which their tests must cover (one anchor, two anchors, the Server CA refused on an Agent CA channel).
- More keys to back up offline from 0.1.

## Open points

- Whether a retired host can be enrolled again under the same `HostId`, or must get a new one, is decided when the host model is designed (0.2).

## References

- ADR-0009, ADR-0010, ADR-0012, ADR-0014 (superseded), ADR-0017 (superseded), ADR-0019 (superseded), ADR-0021, ADR-0022, ADR-0023, ADR-0024, ADR-0025.
- [Use cases of V1](../uml/v1/use-cases.md), section "Not decided, or not stated by an ADR", where these points were raised.
