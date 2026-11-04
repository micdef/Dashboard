# ADR-0012: Storage and retention: hot, cold and inventory history

| Field | Value |
|---|---|
| Status | Accepted |
| Date | 2026-11-04 |
| Decider | Michaël Defraene (product owner, architect) |
| Drafted by | Claude (main developer); the decisions had a second reading by the architecture review agent during scoping |

## Context

Logs must be searchable quickly for recent events, kept longer for analysis and training, and removed when no longer needed. Logs can hold personal data under the General Data Protection Regulation (GDPR): IP addresses, but also user names, account identifiers, sometimes e-mail addresses or tokens in URLs. Metrics (ADR-0008) hold no personal data, but arrive at a fixed rate on every machine. The dashboard must show what each machine has (operating system, packages, pending updates, CVE, state) and which log belongs to which machine. The real log volume is unknown.

## Decision

### Retention

| Data | Where | Kept |
|---|---|---|
| Logs | SQL Server (hot) | 30 days (between 30 and about 37 days, with weekly partitions) |
| Logs | NAS archive | 12 months from the log's timestamp |
| Metrics, full resolution | SQL Server | 30 days (same weekly partitioning) |
| Metrics, hourly aggregates (minimum, average, maximum) | SQL Server | 12 months |
| Inventory | SQL Server | current state always; history with temporal tables |

### Inventory history

- The history of the inventory is kept with **SQL Server temporal tables** (system-versioned).
- `FOR SYSTEM_TIME` is used **only inside stored procedures and views** ("state as of a date"), never by the application (ADR-0003). The period columns are named `ValidFrom` and `ValidTo`, and each history table is named explicitly.
- This keeps a later port to PostgreSQL or Oracle possible without changing the contract: only the history mechanism would change. PostgreSQL 18 does not support system-versioned tables (feature T180 is listed as unsupported), so a port would rebuild the history with triggers and history tables. This port is not planned; it is recorded as a possible improvement in [future improvements](../future-improvements.md).

### Hot logs and metrics

- Logs and full-resolution metrics are stored in tables **partitioned by week**. Old weeks are purged with `TRUNCATE TABLE ... WITH (PARTITIONS (...))`, which requires the table and its indexes to be aligned on the same partition function.
- A columnstore index is used only if measurements justify it: Microsoft recommends at least about one million rows per partition, below which rows stay in delta rowgroups without columnstore compression.
- The volume is **measured from the first increments**; partition size and columnstore are revisited with these figures.
- The **hourly metric aggregates** are computed by the maintenance job. A week of full-resolution metrics is purged only after its hourly aggregates are confirmed written. The aggregates are partitioned by month and purged after 12 months the same way.
- Metrics are **not archived on the NAS**: they hold no personal data, and the training data receives its metric aggregates through the preparation (ADR-0009).

### Partition maintenance

- A maintenance job creates partitions ahead of time (sliding window) and keeps **at least four empty future weeks**.
- Its last successful run is shown as a **health indicator** on the dashboard. The 12-month purge of the NAS archive is a monitored job too.

### Log archive

- The **Collector** runs the export and requests the purge. It reads the rows through the VBE (data gateway, ADR-0003) and writes the files to the NAS (the "Backend → NAS" link of ADR-0004). The VBE never accesses the NAS.
- The export is **hourly and per host**, in **gzip-compressed JSON Lines**. Python converts these files to Parquet for its own needs.
- The export selects rows by **ingestion time**, using a persisted watermark, not by event time. Logs delivered late by an agent are therefore exported in the next file.
- Each file is written under a temporary name and renamed once its size and checksum have been computed and checked.
- A partition is purged **only after every export covering its rows is confirmed written and verified**. A failed export raises an alert and blocks the purge.
- The purge runs in a stored procedure with `EXECUTE AS` (ADR-0003).

### Pseudonymisation

- Every file that the training code can read is **pseudonymised**. The training code never reads the raw archive.
- The method is a keyed hash, **HMAC-SHA256**, applied to IP addresses, e-mail addresses and user names, by the Collector during preparation (ADR-0009).
- The key stays on the Backend host and **never reaches the AI node**. The same value always gives the same pseudonym, so correlations remain usable for training, but a pseudonym cannot be traced back to its value without the key.

### NAS

- Protocol: **SMB 3 with encryption** (the `seal` mount option of `mount.cifs`).
- Separate accounts:
  - the Backend account writes the raw archive and the prepared data;
  - the AI node account only reads the prepared data and only writes the models;
  - the raw archive is reachable by the Backend account only.
- Encryption at rest is configured on the NAS itself, at deployment.

### Legal basis

The purpose of the processing (security monitoring of the home lab) and its legal basis (legitimate interest, GDPR article 6(1)(f)) are documented in the project documentation before V1.

## Alternatives considered

- **Daily partitions.** Rejected as a default: with a small fleet, a day is unlikely to reach the volume a columnstore needs.
- **Deleting old rows with `DELETE`.** Rejected: slow on large tables and fragments columnstore indexes; truncating a partition is the documented approach.
- **Keeping everything in SQL Server.** Rejected: the database would grow without limit; the NAS is cheaper for cold data.
- **History written by the stored procedures, or by triggers, from V1** (portable to PostgreSQL). Rejected for V1: more code, and a forgotten insert or a disabled trigger leaves a hole in the history without any error. Temporal tables are guaranteed by the engine; portability is kept by the contract instead.
- **The VBE runs the export itself.** Rejected: it would need a VBE → NAS link that ADR-0004 does not have.
- **Pseudonymising IP addresses only.** Rejected: user names and e-mail addresses would remain readable in the training data.
- **NFS v4.** Rejected: it is only encrypted with Kerberos (`sec=krb5p`), heavy to set up in a home lab.
- **Full-resolution metrics for 12 months.** Rejected: about 52 million rows a year for 10 machines with 10 series at one point a minute, against about 876,000 hourly rows.
- **Archiving metrics on the NAS.** Rejected: more code for data that holds no personal data and is rarely used.

## Consequences

### Positive

- Purging a week is a metadata operation, not a row-by-row delete.
- The inventory history answers "what did this machine have on that date" without extra code.
- No data is purged before it is safely archived, and retention cannot stop silently: the maintenance job is visible.
- Logs delivered late are not lost by the export.
- The AI node never sees personal data in clear, nor the key that would reveal it.

### Negative

- `TRUNCATE TABLE` needs `ALTER` on the table, and cannot be used on a table referenced by a foreign key: the purge runs in a stored procedure with `EXECUTE AS`, and the hot log and metric tables are not referenced by foreign keys.
- If the maintenance job stops for more than four weeks, rows pile up in the last partition; the health indicator must make this visible long before.
- The raw archive keeps personal data for 12 months; its access is restricted to the Backend account.
- Pseudonymisation detects user names and e-mail addresses by their position in known formats and by patterns; a value in an unexpected place can be missed. The detection rules are tested on real samples.
- Rotating the HMAC key changes every pseudonym; a rotation therefore goes with a full retraining of the model.
- Honouring the right to erasure on archive files means rewriting compressed files on the NAS; the procedure is written in the administration documentation before V1.
- Support of the `seal` option and SMB 3 encryption by the NAS used is **not verified**, nor is its encryption at rest.
- A port to PostgreSQL or Oracle would require rebuilding the inventory history (see Inventory history).

## Open points

None.

## References

- Microsoft Learn, *Columnstore indexes - design guidance*, section "Use table partitions for data management and query performance" (consulted 2026-11-03).
- Microsoft Learn, *TRUNCATE TABLE (Transact-SQL)* and *Columnstore indexes - query performance* (consulted by the review, 2026-11-03).
- PostgreSQL 18 documentation, *Unsupported features* (T180 System-versioned tables), https://www.postgresql.org/docs/current/unsupported-features-sql-standard.html (consulted 2026-11-04).
- ADR-0003, ADR-0004, ADR-0008, ADR-0009; ADR-0018 complements this ADR on the cold storage.
