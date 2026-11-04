# ADR-0012: Storage and retention: hot, cold and inventory history

| Field | Value |
|---|---|
| Status | Proposed |
| Date | 2026-11-03 |
| Decider | Michaël Defraene (product owner, architect) |
| Drafted by | Claude (main developer); the decisions had a second reading by the architecture review agent during scoping |

## Context

Logs must be searchable quickly for recent events, kept longer for analysis and training, and removed when no longer needed. Logs can hold personal data under the General Data Protection Regulation (GDPR): IP addresses, but also user names, account identifiers, sometimes e-mail addresses or tokens in URLs. The dashboard must show what each machine has (operating system, packages, pending updates, CVE, state) and which log belongs to which machine. The real log volume is unknown.

## Decision

- **Retention**: **30 days hot** in SQL Server. The NAS archive keeps each log for **12 months from its timestamp**.
- **Pseudonymisation**: IP addresses are pseudonymised **before** the data is used for training.
- **Inventory**: the current state of each machine is always kept; its history is kept with **temporal tables**.
- **Hot logs**: a table **partitioned by week**; old weeks are purged with `TRUNCATE TABLE ... WITH (PARTITIONS (...))`, which requires the table and its indexes to be aligned on the same partition function. With weekly granularity, a log stays hot between 30 and about 37 days. A columnstore index is used only if measurements justify it: Microsoft recommends at least about one million rows per partition, below which rows stay in delta rowgroups without columnstore compression.
- **Archive**: an hourly export per host, in **gzip-compressed JSON Lines**, to the NAS. Python converts these files to Parquet for its own needs.
- The volume is **measured from the first increments**; partition size and columnstore are revisited with these figures.

## Alternatives considered

- **Daily partitions.** Rejected as a default: with a small fleet, a day is unlikely to reach the volume a columnstore needs.
- **Deleting old rows with `DELETE`.** Rejected: slow on large tables and fragments columnstore indexes; truncating a partition is the documented approach.
- **Keeping everything in SQL Server.** Rejected: the database would grow without limit; the NAS is cheaper for cold data.

## Consequences

### Positive

- Purging a week is a metadata operation, not a row-by-row delete.
- The inventory history answers "what did this machine have on that date" without extra code.

### Negative

- `TRUNCATE TABLE` needs `ALTER` on the table, and cannot be used on a table referenced by a foreign key: the purge runs in a stored procedure with `EXECUTE AS` (ADR-0003), and the hot log table is not referenced by foreign keys.
- Partitions must be created ahead of time by a maintenance job (sliding window); if the job does not run, rows pile up in the last partition and retention silently stops working.
- The raw archive keeps personal data for 12 months; access to it must be restricted.
- Honouring the right to erasure on archive files means rewriting compressed files on the NAS; the procedure must be designed.

## Open points

1. Temporal tables for the inventory history.
2. Weekly partitions and the conditional columnstore.
3. Hourly JSON Lines gzip archive per host, Parquet conversion done by Python.
4. Not yet decided: which component runs the export and the purge (it must go through the data gateway, ADR-0003), and the pseudonymisation method (for example a keyed hash; its key must never reach the AI node).
5. Raised in review, proposed: a partition is purged only after every export covering its rows is confirmed written and verified (size and checksum); a failed export raises an alert and blocks the purge.
6. Raised in review, proposed: the export selects rows by **ingestion time** (a persisted watermark), not by event time, so logs delivered late by an agent are exported in the next file; files are written under a temporary name and renamed after their checksum is computed.
7. Raised in review, proposed: any file readable by the training code is pseudonymised, and the training code never reads the raw archive; access control and encryption at rest of the NAS archive are defined; the purpose and legal basis of the processing (security monitoring) are documented.
8. NAS file protocol and access rights (moved from ADR-0004 on 2026-11-04).
9. Metrics storage and retention (added on 2026-11-04 after ADR-0008): table design, retention period, and whether older data points are aggregated (for example per hour) instead of being kept at full resolution.
10. Raised in review, proposed: the partition job keeps at least a few empty future partitions, and its last successful run is shown as a health indicator; the 12-month NAS purge is a monitored job too.

## References

- Microsoft Learn, *Columnstore indexes - design guidance*, section "Use table partitions for data management and query performance" (consulted 2026-11-03).
- Microsoft Learn, *TRUNCATE TABLE (Transact-SQL)* and *Columnstore indexes - query performance* (consulted by the review, 2026-11-03).
- ADR-0003, ADR-0009.
