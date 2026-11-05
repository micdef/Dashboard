# Backup and disaster recovery

What must be backed up for Dashboard, and in which order it is restored (ADR-0022). Dashboard's own jobs cover the database; the virtual machines themselves are backed up by the system administrator with their own tool (snapshot or backup software). Nothing on this page is tested yet.

## What to back up

| Item | Where it lives | Backed up by | How |
|---|---|---|---|
| Database (system of record) | SQL Server VM | Dashboard (SQL Server Agent) | encrypted native backups to the cold storage: full weekly, differential daily, log hourly; restore test monthly |
| Virtual machines ("app" VM, SQL Server VM, AI node) | Hyper-V host | the system administrator | snapshot or backup software of their choice |
| Cold storage (archive, audit files, prepared data, models, database backups) | file server or NAS | the system administrator | replication or backup of the file server; the audit archive stays write-once |
| HMAC key of the audit trail (ADR-0020) | "app" VM | the product owner | offline copy, kept as long as the audit archive |
| Pseudonymisation key (ADR-0012) | "app" VM | the product owner | offline copy |
| Key encrypting the authenticator keys (ADR-0019) | "app" VM | the product owner | offline copy |
| Backup encryption certificate and its private key | SQL Server VM | the product owner | offline copy; without it, no backup can be restored |
| Server CA key (ADR-0021) | offline medium | the product owner | second offline medium, stored separately |
| Agent CA key (ADR-0021) | "app" VM | the product owner | offline copy |
| Configuration files of every component | each host | the system administrator | with the virtual machines, and a copy in the deployment repository |

## Restore order

1. The SQL Server VM, or a new SQL Server; restore the backup certificate, then the last full, differential and log backups.
2. The "app" VM, or a new one; restore the keys and the configuration files, then start the VBE, the API, the Collector, the Enrolment service and `Dashboard.Crl`.
3. Check the audit chain with the independent verification tool, against the audit archive.
4. The AI node; restore its configuration; the models are on the cold storage.
5. Agents reconnect on their own; agents whose certificate expired during the outage need a new enrolment token.

## Checks

- [ ] A monthly restore test has succeeded (shown on the dashboard).
- [ ] The age of the last full, differential and log backup is within its threshold.
- [ ] The offline copies of the keys and certificates are present and readable.
- [ ] A full restore of the "app" VM has been rehearsed at least once before V1.
