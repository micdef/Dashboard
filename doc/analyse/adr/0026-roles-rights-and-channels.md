# ADR-0026: Roles, rights and channels

| Field | Value |
|---|---|
| Status | Accepted |
| Date | 2026-11-06 |
| Decider | Michaël Defraene (product owner, architect) |
| Drafted by | Claude (main developer); second reading by the architecture review agent before the decisions, then team review |
| Classification | Security-sensitive [SEC] |
| Complements | ADR-0022, ADR-0023, ADR-0024, ADR-0025 |

## Context

The use case diagrams of V1 ([use cases](../uml/v1/use-cases.md)) showed actions that the ADRs place under step-up or on the host channel without saying **who** may perform them:

- The rights of the roles are still in force, but only by inheritance. ADR-0023 keeps every decision of ADR-0019, which keeps every decision of ADR-0014. The right of the Reader ("sees everything, changes nothing") is written only in ADR-0014, which is superseded. No accepted ADR gathers the rights.
- ADR-0023 puts revoking a machine, deleting a host and silencing an alert under step-up, without naming the role. "Deleting a host" is defined nowhere.
- ADR-0025 protects a list of critical alerts from being silenced from the web site. ADR-0022 leaves the level of the other alerts to a catalogue written at coding time. Neither "silence" nor "acknowledge" is defined for an alert.
- ADR-0010 speaks of reports "requested", without saying by whom.
- ADR-0025 enrols the verifier with a "monitor" token, without saying who creates it.
- ADR-0019 lets each user set up their own second factor, but the step-up list of ADR-0023 does not cover changing it, nor regenerating the recovery codes, so a stolen session could take over the account.
- The list of keys to back up offline in ADR-0022 misses the Agent CA key, the package signing keys and the recorded head of the audit chain, and nothing detects a missed or outdated backup.

The anchor sets and the re-key of the Agent CA, first drafted here, change a rule of ADR-0021 and are decided in ADR-0027, which supersedes it.

This ADR adds rules to the ADRs it complements. Where it extends a rule, it says so.

## Decision

### Rights of the roles in the web site

The roles stay **Reader**, **Operator** and **Admin**, each one with every right of the previous one.

| Action in the web site | Reader | Operator | Admin | Step-up |
|---|---|---|---|---|
| View the fleet, machines, logs, metrics, anomalies, reports, rules, CVE, alerts, health and audit trail | yes | yes | yes | no |
| Set up, replace or remove one's own second factor; regenerate one's own recovery codes | yes | yes | yes | yes |
| Request an AI report | — | yes | yes | no |
| Acknowledge an alert | — | yes | yes | no |
| Silence a Warning alert | — | yes | yes | yes |
| Issue a catalogue command | — | yes | yes | yes, single use |
| Manage accounts and roles, reset another user's second factor, unlock an account | — | — | yes | yes, single use |
| Create an enrolment token for an agent | — | — | yes | yes |
| Revoke a machine | — | — | yes | yes, single use |
| Retire a revoked host | — | — | yes | yes, single use |
| Raise a security setting or the audit retention | — | — | yes | yes |

- **What no role sees**: the value of an enrolment token after it was shown once at its creation, the sessions of other users, and the stored secrets of ADR-0023. The audit trail records the identifier of a token or a session, never its value nor its hash.
- **The audit trail is visible to every role** (the product owner's choice). It holds user names and source addresses: the legal basis document of the project, to be written before V1 (ADR-0012), must cover this visibility.
- **Step-up list**: compared with ADR-0023, this table adds changing one's own second factor and regenerating one's own recovery codes; "deleting a host" becomes "retiring a host" (a host is never deleted); revoking a machine and retiring a host need a **single-use** proof, like commands, so that a compromised web site cannot revoke the whole fleet with one proof.
- **Monitor identities** are never revoked from the web site: see "Host channel".

### Second factor

- A recovery code is not accepted as a step-up proof, except to set up a new second factor after losing the old one; that use raises a critical alert.
- An Admin cannot reset their own second factor.
- Adding or replacing a second factor on an Operator or Admin account raises a critical alert, like removing one (ADR-0025).
- Under the policy "required for Operator and Admin" (ADR-0019, kept by ADR-0023), an Operator or Admin account without a second factor, after a change of policy or a reset, must set one up at its next sign-in, and removing it is refused. This replaces, for these accounts, the warning of ADR-0019.

### AI reports

- A report is generated every day by a scheduled job, after the training window. An Operator or an Admin may request one.
- Requests are rate-limited by the configuration. A refused request returns an explicit error ("limit reached, retry at T") and is audited. A failed generation does not count against the limit.
- A request made during the training window is refused with a visible reason; it is not queued.
- A daily report that fails, or that the training delays beyond its window, raises a Warning alert.

### Alerts: acknowledge and silence

- **Acknowledging** an alert records that someone is handling it. It changes nothing but the display: the alert stays open, its notifications are not stopped, and the acknowledgement is audited. Acknowledging the "audit incident" alert in the web site does not lift the block of ADR-0024: only the host command does.
- An acknowledged critical alert that is still open is **notified again** at an interval set in the host's configuration file, which the web site cannot change.
- **Silencing** stops the notifications of one alert instance, of level Warning, for a duration. The maximum duration and the maximum number of renewals are read from the host's configuration file, not from a setting of the web site. The dashboard shows "silenced until T by X". When the silence ends and the alert is still open, it is notified again; its resolution is always notified.
- **No critical alert can be silenced from the web site**, whether it is in the list of ADR-0025 or not: this extends the rule of ADR-0025. Information alerts are shown on the dashboard only (ADR-0022), so there is nothing to silence.
- Creating or revoking a monitor identity raises a critical alert (added to the list of ADR-0025).

### Retiring a host

- A host is never deleted. **Retiring** a host is allowed only when it is in the **revoked** state, recorded by an explicit revocation (ADR-0025). A host whose certificate has only expired cannot be retired before it is revoked.
- A retired host is no longer expected: its silence raises no alert. A connection attempt from a retired host is shown and raises a Warning alert.
- No enrolment token can be created for a retired host. A machine that comes back is enrolled under a new `HostId`.
- Its data stays and follows the retention rules of ADR-0012 and ADR-0024.

### Host channel

The host channel of ADR-0023 also carries the creation and revocation of **"monitor" tokens and identities**, for the verifier. They are never available in the web site, because the verifier is part of the monitoring of the monitor (ADR-0022) and must not be replaceable from the most exposed process.

### Keys and secrets backed up offline

- The list of ADR-0022 is completed:

  | Key or secret | Backed up from |
  |---|---|
  | Server CA key (already in ADR-0022) | 0.1 |
  | Agent CA key, held online in the Enrolment service; during a re-key, both Agent CA keys (ADR-0027) | 0.1 |
  | GPG key that signs the RPM packages | 0.1 |
  | Configuration files, and the secrets of ADR-0025 (SMTP and mount credentials, webhook address) | 0.1, as each one is created |
  | HMAC key of the audit trail, key that encrypts the authenticator keys, backup encryption certificate and the password of its private key (already in ADR-0022, except the password) | 0.2 |
  | Recorded head of the audit chain (ADR-0024) | 0.2 |
  | Pseudonymisation key (already in ADR-0022) | 0.3 |
  | Code signing certificate of the MSI packages, and the key of the internal CA that issues it if it is internal | 0.4 |

- The Data Protection key ring of the web site is not backed up (ADR-0023). The verifier's copy of the HMAC key is a copy, not a separate key.
- **Key backup register**: for each key or secret of the list, the register records its fingerprint, the date it was created and the date of its last offline backup. The key holder records each backup with a host command, which is audited. The dashboard shows the age of each backup. A Warning alert is raised when a key has no backup, when its fingerprint differs from the backed-up one (a key was replaced), or when the backup of the recorded head of the audit chain is older than the interval set in the configuration. The register exists from 0.1; its alerts start with the alert channel, in 0.2.
- Each key is backed up in the increment that creates it, and again whenever it is replaced.

## Alternatives considered

- **An editorial note in ADR-0023 restating the rights.** Rejected: an editorial correction is limited to cross-references and names; new rules need a new ADR.
- **An ADR that supersedes ADR-0023.** Rejected: too heavy for a table; ADR-0023 stays valid.
- **The Operator may revoke a machine.** Rejected: revocation is the counterpart of the enrolment token, which only an Admin creates, and it cuts the machine off at once.
- **Physical deletion of a host.** Rejected by the product owner: a host is retired, and its data follows the retention rules.
- **Retiring any host without an active certificate.** Rejected: a compromised host whose agent was stopped would leave the monitoring without revocation and without alert.
- **Critical alerts outside the ADR-0025 list silenced by an Admin after step-up.** Rejected: they are the alerts a malicious Admin would want to silence before acting (ADR-0025), and one rule is simpler to test than a list.
- **"Monitor" tokens also created in the web site.** Rejected: a compromised web site could replace the verifier's identity.
- **AI reports on schedule only.** Rejected: the people who handle incidents would have no report on demand.
- **AI reports requested by every role.** Rejected: a request changes something, which the Reader does not do.
- **Reports requested during training queued.** Not adopted by the product owner: the request is refused visibly instead.
- **The key backups as a written procedure only, the register later.** Not adopted by the product owner: the register is part of V1.

## Consequences

### Positive

- Every action of the web site and the host channel has a role and a channel, in one table, which the authorisation tests can follow line by line.
- No critical alert can be cut from the web site, an acknowledged one keeps coming back until it is resolved, and the verifier's identity cannot be replaced from the web site.
- A stolen session cannot take over its user's second factor or recovery codes without the password and the current code, and a takeover attempt on an Operator or Admin raises a critical alert.
- A host can leave the fleet without breaking the audit trail or the history, and without hiding a compromise.
- A missed key backup is visible, before the key is needed.

### Negative

- More step-up prompts for users, one more state ("retired") in the host model, and more critical alerts (second factor changes, monitor identities).
- An Operator cannot silence a noisy critical alert: the cause must be fixed or the threshold changed in the configuration.
- The key backup register is one more host command and one more table to build in 0.1.

## References

- ADR-0010, ADR-0012, ADR-0014 (superseded), ADR-0019 (superseded), ADR-0022, ADR-0023, ADR-0024, ADR-0025, ADR-0027.
- [Use cases of V1](../uml/v1/use-cases.md), section "Decided on 2026-11-06".
- [Review report of 2026-11-06](../reviews/2026-11-06-roles-and-anchor-sets.md).
