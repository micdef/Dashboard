# ADR-0022: Operations: alerts, monitoring of the monitor, backups and reconciliation

| Field | Value |
|---|---|
| Status | Accepted |
| Date | 2026-11-05 |
| Decider | Michaël Defraene (product owner, architect) |
| Drafted by | Claude (main developer), after the second reading of 2026-11-04 |
| Complements | ADR-0004, ADR-0006, ADR-0008, ADR-0012, ADR-0018, ADR-0024 |

## Context

The ADRs say "raises an alert" about fifteen times, but none says where an alert goes: every failure indicator lives on the dashboard, which nobody may be watching, and which disappears with the Collector or the "app" VM. No ADR covers the backup of the database, the system of record. The second reading also found ways for data to be lost without anyone noticing: an export that skips rows of transactions still open, a checksum that proves a file is intact but not complete, a partial failure acknowledged as a success, metrics arriving after their hour was aggregated, a log source that stops while the heartbeat stays green, and a cold storage mount that disappears (findings I5 and I6).

This ADR adds to the ADRs it complements; it changes none of their decisions, but it adds one link to the list of ADR-0004 (SQL Server VM → cold storage). ADR-0025 lists every link added since ADR-0004.

## Decision

### Alert channel

- One alert channel, set in the configuration (nothing hard-coded): **e-mail through SMTP** (server, port, account, recipients), and optionally a **webhook**.
- Three levels:

  | Level | Delivery |
  |---|---|
  | Critical | sent at once |
  | Warning | grouped in a daily summary |
  | Information | dashboard only |

- An alert is not sent again while it is open; its resolution is sent too. Every alert of the ADRs is given a level in the catalogue of alerts, written at coding time.

### Monitoring of the monitor

- The **independent verification tool** of ADR-0024, which runs on another machine than the "app" VM under a systemd timer, also checks at each run the health checks of the Collector and the API, and the age of the Collector's last self-heartbeat.
- If they have not answered for a configured number of minutes, the tool sends a **critical alert itself**, through the same channel, without depending on the "app" VM.
- When the Collector or the VBE is unreachable, the dashboard shows the whole fleet as **"unknown"**, never as "down" nor as "up" (ADR-0006).

### Database backups and disaster recovery

- **Native SQL Server backups**, encrypted with a certificate (backup encryption is available in the Standard edition, Microsoft Learn), run by SQL Server Agent to a dedicated share of the cold storage:

  | Backup | Frequency |
  |---|---|
  | Full | weekly |
  | Differential | daily |
  | Transaction log | hourly (FULL recovery model) |

- A **restore test** runs automatically every month into a disposable database. The age of the last backup of each kind and the result of the last restore test are shown on the dashboard and alerted.
- The backups are written by the SQL Server service, not by the VBE: this is a documented exception to the link list of ADR-0004 (SQL Server VM → cold storage), with its own account on the share.
- **Keys and secrets outside the database** are backed up offline by the product owner: the HMAC key of the audit trail (ADR-0024), the pseudonymisation key (ADR-0012), the key that encrypts the authenticator keys (ADR-0023), the backup encryption certificate, the Server CA key (ADR-0021), and the configuration files.
- **Whole-system backups**: the installation guide states that the virtual machines themselves (the "app" VM, the SQL Server VM, the AI node) must also be backed up with the system administrator's own tool (snapshot or backup software). These are the administrator's responsibility; the guide lists what must be backed up, how each item is restored, and in which order, as a **disaster recovery plan**.

### Reconciliation of exports and purges

- **Safety margin**: the log export selects rows by ingestion time only up to a margin behind the current time, longer than the longest allowed write transaction, so that rows of transactions still open are not skipped.
- **Counts before purge**: each export records its row count, range and checksum in the database. The **purge procedure itself** compares the row count of the partition with the sum of the exported counts covering it, and refuses on any difference (as for the audit trail, ADR-0024). A refusal raises an alert.
- **Batches**: the batch identifier (ADR-0008) is written in the same transaction as the data it covers. Row-level failures returned by the procedures (one result per input row, ADR-0003) are mapped to `partial_success`; the Collector keeps the counters "received = stored + rejected" per machine, and a difference is shown and alerted. The retention of the batch identifiers is longer than the maximum age of an agent's buffer. A split batch gets deterministic identifiers.
- **Late metrics**: a week of full-resolution metrics is purged only after its hourly aggregates exist **and** the maximum age of an agent's buffer has passed. Each aggregate stores its number of samples, and an aggregate is recomputed when late samples arrive before the purge.

### Liveness of each source

- The heartbeat carries, for each log and metric source of the machine, the time of its last event and its counters (read, sent, dropped, set aside). The agent keeps these counters on disk, so a restart does not lose them.
- The Collector flags a source **"silent"** against what is expected of it (metrics arrive at a fixed interval). A silent source is shown even when the heartbeat is green.

### Cold storage writes

- The mount check of ADR-0018 (the path is a mount point and is writable) applies to **every write** to the cold storage, on the "app" VM and on the AI node: exports, audit files, prepared data, models, backups.
- Every input/output on the cold storage has a **timeout**, so that a hung mount fails instead of blocking a job.
- Temporary files left by an interrupted write are listed, alerted and removed.

## Alternatives considered

- **Dashboard only.** Rejected: nobody may be watching, and it disappears with the component that fails.
- **Webhook only.** Rejected: it needs a receiving service before anyone is warned.
- **A separate tool for the monitoring of the monitor.** Rejected: one more component; the independent verification tool already runs outside the "app" VM.
- **Virtual machine backups only.** Rejected as the only protection: restoring is coarse and nothing checks the consistency of the database; they are kept as a second layer, under the administrator's responsibility.
- **Export by watermark without margin, purge on checksum only.** Rejected: rows can be skipped and the files still look valid.

## Consequences

### Positive

- A failure is pushed to someone, including the failure of the system itself.
- The database can be restored, and the restore is tested every month.
- No purge can remove a row that was not exported, and no acknowledged batch can hide a lost row without a visible difference.
- A source that stops is seen even when its machine looks healthy.

### Negative

- SMTP credentials and a webhook address are secrets in the configuration.
- More jobs to watch: backups, restore test, reconciliation; each one has its own indicator.
- A new link (SQL Server VM → cold storage) and a dedicated share for the backups.
- The backup encryption certificate must be backed up offline; without it, the backups cannot be restored.
- The independent verification tool becomes part of the safety net: if it stops, the monitoring of the monitor stops too. Its own last run is shown on the dashboard, and it is the product owner's tool to watch.

## References

- Microsoft Learn, *Editions and supported features of SQL Server 2025*, section "Security" (encryption for backups in Standard) (consulted 2026-11-04).
- [Second reading report](../reviews/2026-11-04-second-reading.md), findings I5 and I6.
- ADR-0003, ADR-0004, ADR-0006, ADR-0008, ADR-0012, ADR-0018, ADR-0021, ADR-0023, ADR-0024; ADR-0025 complements this ADR.
