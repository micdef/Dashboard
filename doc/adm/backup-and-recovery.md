# Backup and disaster recovery

What must be backed up for Dashboard, and in which order it is restored (ADR-0022, ADR-0025, ADR-0026). Dashboard's own jobs cover the database; the virtual machines themselves are backed up by the system administrator with their own tool (snapshot or backup software). Nothing on this page is tested yet.

## What to back up

| Item | Where it lives | Backed up by | How |
|---|---|---|---|
| Database (system of record) | SQL Server VM | Dashboard (SQL Server Agent) | encrypted native backups to the cold storage: full weekly, differential daily, log hourly; kept 5 weeks by default; restore test monthly |
| Virtual machines ("app" VM, SQL Server VM, AI node, verifier machine) | Hyper-V host | the system administrator | snapshot or backup software of their choice; images **encrypted** and stored outside the shares that Dashboard accounts can reach: the "app" VM image holds every key, the SQL Server VM image holds the backup certificate; images taken before the retirement of an Agent CA still hold its old key and are destroyed at the end of their retention (ADR-0027) |
| Cold storage (archive, audit files and receipts, prepared data, models, database backups) | file server or NAS | the system administrator | replication or backup of the file server; the audit archive stays write-once where the server enforces it |
| Verifier store, its own copy of the audit files (ADR-0024) | verifier machine | the system administrator | backup of that machine; this copy is the anchor of the audit trail |
| HMAC key of the audit trail (ADR-0024) | "app" VM (data gateway user only) and verifier machine | the product owner | offline copy, kept as long as the audit archive |
| Pseudonymisation key (ADR-0012) | "app" VM | the product owner | offline copy |
| Key encrypting the authenticator keys (ADR-0023) | "app" VM | the product owner | offline copy |
| Backup encryption certificate, its private key and the password of that key | SQL Server VM | the product owner | offline copy; without it, no backup can be restored |
| Server CA key (ADR-0027) | offline medium | the product owner | second offline medium, stored separately |
| Agent CA key (ADR-0027); during a re-key, both Agent CA keys | "app" VM | the product owner | offline copy; the old key and its copy are destroyed after its retirement (ADR-0027) |
| Recorded head of the audit chain (ADR-0024) | "app" VM | the product owner | offline copy, at the interval set in the configuration (ADR-0026) |
| GPG key of the agent packages, code signing certificate and, if it is internal, the key of the CA that issues it (ADR-0025, ADR-0026) | product owner | the product owner | offline copy |
| Configuration files of every component | each host | the system administrator | with the virtual machines; the deployment repository holds the configuration **without secrets** |
| Secrets (SMTP and mount credentials, webhook address) | each host, in files readable by their service user only, or systemd credentials | the product owner | offline copy, with the keys |

Each offline backup, and each destruction of a key, is recorded by the key holder in the key backup register with a host command; the dashboard shows the age of each backup and, from 0.2, raises a Warning when a key has no backup or was replaced (ADR-0026). The register proves that a backup was recorded, not that it can be restored: the checks below remain needed.

The Data Protection key ring of the web site is not backed up: losing it only signs every user out (ADR-0023).

## Restore order

1. The SQL Server VM, or a new SQL Server; restore the backup certificate, then the last full, differential and log backups.
2. The "app" VM, or a new one; restore the keys, the secrets and the configuration files, then start the VBE, the API, the Collector, the Enrolment service and `Dashboard.Crl`.
3. **Expected audit block**: a restored database is behind the head that the VBE recorded outside the database, so the VBE raises a critical alert and refuses audit writes (ADR-0024). Check with the independent verification tool that the missing rows are in its copy of the audit files, then acknowledge the incident with the host command.
4. The AI node; restore its configuration; the models are on the cold storage.
5. Agents reconnect on their own; a host whose certificate expired during the outage must be revoked, then gets a new enrolment token (ADR-0026).

## Checks

- [ ] A monthly restore test has succeeded (shown on the dashboard).
- [ ] The age of the last full, differential and log backup is within its threshold.
- [ ] The offline copies of the keys, certificates and secrets are present and readable.
- [ ] A full restore of the "app" VM has been rehearsed at least once before V1, including the acknowledgement of the audit block.
