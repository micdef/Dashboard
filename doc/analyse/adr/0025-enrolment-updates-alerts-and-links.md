# ADR-0025: Enrolment safeguards, signed agent updates, protected alerts and the added links

| Field | Value |
|---|---|
| Status | Accepted |
| Date | 2026-11-05 |
| Decider | Michaël Defraene (product owner, architect) |
| Drafted by | Claude (main developer), after the third reading of 2026-11-05 |
| Classification | Security-sensitive [SEC] |
| Complements | ADR-0004, ADR-0015, ADR-0016, ADR-0021, ADR-0022 |

## Context

The third reading found gaps that do not change the decisions of the ADRs above, but leave them incomplete:

- an enrolment token can target a `HostId` that already has an active certificate, so a malicious Admin, or a web site that captured a step-up, could replace a production machine by a rogue one (T-I6); in 0.1 nobody can revoke a machine, because there is no web site yet (Jocasta Nu); the 0.1 issuance register enters the audit trail without proof (T-M5);
- the compromise procedure and the Agent CA re-key of ADR-0021 need an agent update, which no ADR defines; if it went through the Collector, whoever holds the Collector's key would also control the response (T-I5, Jocasta Nu);
- an Admin who can change the alert recipients from the web site can blind the monitoring before acting, and privilege changes raise no alert (T-I8);
- ADR-0022 adds network links that ADR-0004 does not list, and the verifier cannot reach the API, which listens only on a Unix socket (Jocasta Nu);
- secrets would be copied into the deployment repository with the configuration files (T-I4); backups have no retention; some terms of ADR-0022 are not defined; the new work of ADR-0021 to ADR-0024 has no increment (Jocasta Nu);
- `Dashboard.Crl` could be asked for more than its files, and an old Agent CA list can be replayed over HTTP (T-M4).

This ADR adds to the ADRs it complements. It adds network links to ADR-0004 and work to the increments of ADR-0015, without changing their order.

## Decision

### Enrolment safeguards

- A token can be created only for a **new** `HostId` or for one **without an active certificate**. Enrolling a machine again requires an explicit revocation first, which is audited and raises a **critical alert**.
- Each use of a token is shown on the dashboard with the source address of the request.
- In **0.1**, machines are revoked with a host command (ADR-0023, host channel).
- The entries of the 0.1 issuance register are written into the audit trail in 0.2 marked **"recorded before the audit trail, not proven"**; the audit views show this mark.

### Agent updates: signed, outside Dashboard

- The Linux agent is shipped from **0.1** as an **RPM package signed with the product owner's GPG key**, installed and updated from a dnf repository; the machines check the signature (`gpgcheck=1`).
- The Windows agent is shipped from **0.4** as an **MSI package signed** with a code signing certificate trusted by the machines (an internal one is enough).
- New trust anchors (compromise procedure, Agent CA re-key, ADR-0021) are delivered **only** in these packages. Agent updates **never** go through the Collector or through remote commands.

### Alerts that cannot be silenced from the web site

- The alert configuration (SMTP server, account, recipients, webhook) is a **file on the host**, never editable from the web site. The independent verification tool has its own configuration, on its own machine.
- A **critical alert** is raised for: a new Admin or Operator; a role raised; a second factor removed from an account; a change of retention or of a security setting; a machine revoked; a break-glass unlock (ADR-0023); an audit incident (ADR-0024).
- Silencing an alert from the web site needs step-up (ADR-0023) and is audited; critical alerts of this list cannot be silenced from the web site.

### Health of the Backend, reachable by the verifier

- The API and the VBE each record a **self-heartbeat** every minute (through the VBE). The Collector's own self-heartbeat is the time of its last successful write.
- The Collector serves the standard gRPC **health service** on its mTLS endpoint. Its answer includes the age of the three self-heartbeats.
- The verification tool is enrolled **like an agent, with a token of kind "monitor"**: its certificate is mapped to an identity that can call the health service only, nothing else.

### Links added to ADR-0004

| Link | Network | Purpose | Decided in |
|---|---|---|---|
| verifier → Collector | across the network, mTLS | health service only | this ADR |
| verifier → cold storage | across the network, SMB 3 | read the audit archive, write the receipts | ADR-0024 |
| verifier → mail server, webhook | across the network | its own alerts | ADR-0022 |
| Backend → mail server, webhook | across the network | alerts | ADR-0022 |
| SQL Server VM → cold storage | across the network, SMB 3 | database backups, written by the SQL Server service | ADR-0022 |
| agent → `Dashboard.Crl` | across the network, HTTP | revocation lists | ADR-0021 |
| machines → dnf repository (and MSI share) | across the network | agent packages | this ADR |

### `Dashboard.Crl`

- It serves only files ending in `.crl`, without directory listing, and refuses any request with a body.
- The Collector reads the Agent CA list from the local file written by the Enrolment service, never through HTTP.

### Secrets and backups

- The deployment repository holds the configuration **without secrets**. Secrets (SMTP and mount credentials, webhook address, keys) are in files readable by their service user only, or in systemd credentials, and are backed up offline with the keys (ADR-0022).
- Database backups are kept for **5 weeks** by default (configurable): personal data deleted from the database stays in the backups for that time at most, which the legal basis document must state.
- Virtual machine images are encrypted and stored outside the shares that Dashboard accounts can reach: the image of the "app" VM holds every key, and the image of the SQL Server VM holds the backup certificate.
- The SQL Server service writes its backups to a UNC path on Windows or to a CIFS mount on Linux; the operating system of the SQL Server VM is the administrator's choice. This is SQL Server's own feature, not Dashboard code (ADR-0018).

### Terms of ADR-0022, defined

- The **longest allowed write transaction** and the **maximum age of an agent's buffer** are configuration keys, with defaults set at coding time; the export margin and the purge delays are derived from them.

### Work added to the increments of ADR-0015

| Increment | Added |
|---|---|
| 0.1 | `Dashboard.Crl`; the host channel with the commands for tokens and revocation (ADR-0023); the signed RPM package and its dnf repository |
| 0.2 | API sessions and step-up (ADR-0023); audit written by the VBE, the independent verifier with receipts and health checks (ADR-0024); the alert channel and its critical alerts; database backups and the monthly restore test (ADR-0022) |
| 0.4 | the signed MSI package |

## Alternatives considered

- **Agent updates through the Collector.** Rejected: the key that a compromise procedure must replace would control the delivery of its own replacement.
- **Manual agent deployment until V1.** Not adopted: the compromise procedure would stay manual on every machine until then.
- **Alert configuration in the web site, after step-up.** Rejected: a malicious Admin could blind the monitoring before acting.
- **The verifier reading the health from the export files.** Not adopted: detection after more than one hour, and it would change the decision of ADR-0022.
- **Re-enrolment by token over an active certificate.** Rejected: it allows replacing a machine silently.

## Consequences

### Positive

- A production machine cannot be replaced without an explicit, alerted revocation.
- The response to a key compromise does not depend on the component that may be compromised.
- Privilege changes are pushed to someone, and the alert channel cannot be cut from the web site.
- Every link of the system is listed, and the verifier reaches the health of the Backend without touching the database.

### Negative

- Increment 0.1 grows: host channel, `Dashboard.Crl`, packaging and repository.
- A GPG key and a code signing certificate to keep and back up offline.
- One more identity kind ("monitor") to test in the mapping.
- More critical alerts during installation (first accounts, first enrolments).

## References

- [Third reading report](../reviews/2026-11-05-third-reading.md), findings T-I4 to T-I6, T-I8, T-M4, T-M5, and J-I2, J-I4, J-I6, J-I7.
- ADR-0004, ADR-0015, ADR-0016, ADR-0018, ADR-0021, ADR-0022, ADR-0023, ADR-0024.
