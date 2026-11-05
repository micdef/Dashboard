# ADR-0013: Remote commands without SSH

| Field | Value |
|---|---|
| Status | Superseded by [ADR-0020](0020-remote-commands-and-keyed-audit-trail.md) |
| Date | 2026-11-04 |
| Decider | Michaël Defraene (product owner, architect) |
| Drafted by | Claude (main developer); the decisions had a second reading by the architecture review agent during scoping |
| Classification | Security-sensitive [SEC] |

## Context

The initial outline planned to send commands to the machines through SSH. Remote command execution is the most dangerous feature of the system: a compromised Backend, web site process or user account could take control of every machine.

## Decision

### Channel and catalogue

- **No SSH.** The agent opens a **bidirectional gRPC stream** to the Collector over mTLS (ADR-0004, ADR-0017); commands arrive on this stream. No inbound port is opened on the machines.
- Commands are **typed messages from a catalogue** (for example "restart service X"), never free text.
- Each machine has a **local allowlist** that only root (or an administrator on Windows) can modify. A command outside the allowlist is refused by the agent, whatever the Collector sends. Typed parameters (such as a service name) are **matched exactly** against the allowlist before use.
- The roles allowed to issue commands are decided in ADR-0014.
- **Signing of commands** (so that the agent can verify the issuer independently of the Backend) is planned for **V2**.

### Lifecycle: at most once

- Each command carries a **unique identifier** and an **expiry timestamp**.
- The expiry is checked twice: the Collector never delivers an expired command, and the agent refuses it too. A command not delivered in time (machine offline) is never executed late.
- The agent refuses an identifier it has already seen. The seen identifiers are **kept on disk until their expiry**, so that a restart of the agent does not reopen a replay.
- The agent compares the expiry with its own clock, which must be synchronised (NTP on Linux, W32Time on Windows). The heartbeat reports the **clock offset** of the machine, shown on the dashboard, and the agent refuses commands when its offset exceeds a configured tolerance.

### Execution: least privilege, never a shell

- The agent runs fixed commands directly, **never through a shell**. Each execution has a **maximum duration** and a **capped output size**.
- **Linux**: the agent calls `sudo` with an argument list, without a shell. The sudoers entries are **exact commands without wildcards**. No interactive command and no command that can open a pager or an editor goes through `sudo` (for example, `sudo systemctl status` opens a pager from which a root shell can be started); reading a state does not need `sudo`.
- **Windows**: the agent does **not run as LocalSystem**. It runs under a dedicated service account with rights only on the allowed services (through `ServiceController`). The exact rights it needs (reading the Security log, Windows Update Agent) are set with the Windows agent in increment 0.4, in a new ADR if they change this decision.

### Immutable audit trail

- Every step is written to the audit trail: request, delivery, and outcome (**executed, refused, timed out, expired, result lost**), with the requesting user and the role used.
- The audit trail is a regular table, **partitioned by month**, on which the application account can only insert, through a stored procedure (ADR-0003).
- **Hash chain.** Each row stores a SHA-256 hash computed over its own content and the hash of the previous row. The insert procedure serialises the inserts (one chain, no gap). Changing, removing or inserting a row breaks the chain from that point on.
- **External anchors.** On a schedule, the latest hash (with its row identifier and time) is written outside the database, to a share of the cold storage (ADR-0018) where the Backend account can create files but not modify or delete them.
- **Verification.** On a schedule, the chain is recomputed and compared with the anchors; the result of the last verification is shown as a health indicator.
- **Command output** is not stored in the audit trail: the audit row keeps its hash, and the output itself is stored in a regular table, purged like the logs (ADR-0012).

### Audit retention and purge

- The audit trail is kept **2 years in the database**, then exported to the cold storage, where the archive is kept until **5 years in total**, then deleted. Both durations are **set by the administrator at installation** in the configuration; these are the default values.
- Each value has a minimum (90 days in the database; the archive retention cannot be shorter than the database retention), so that the audit trail cannot be switched off through configuration. The values in force, and every change, are recorded in the audit trail itself.
- **Purge**, by the maintenance job, one month at a time:
  1. the month is exported to the cold storage, with its hashes, in gzip-compressed JSON Lines;
  2. the export is verified (size, checksum, and the chain recomputed from the file);
  3. a "purge" row is inserted in the chain, recording the purged range and the hash of its last row;
  4. the partition is truncated (`EXECUTE AS`, ADR-0003).
  A failed export or verification raises an alert and blocks the purge.
- The exported files remain verifiable: their chain links to the "purge" row that follows them and to the anchors.

## Alternatives considered

- **SSH from the Backend.** Rejected: it requires an inbound port and credentials on every machine, and a shell on the remote side.
- **Free-text commands.** Rejected: the allowlist could not be checked reliably.
- **Seen identifiers kept in memory only.** Rejected: a restart of the agent would allow a replay.
- **Expiry checked by the Collector only.** Rejected: a command held back (by a delay in the network or a compromised Collector) could still be executed late.
- **Windows agent as LocalSystem, restricted later.** Rejected: a temporary full-privilege account tends to stay.
- **A regular audit table with `INSERT` permission only, without a hash chain.** Rejected: a database administrator could change rows without leaving a trace.
- **An append-only ledger table of SQL Server.** Considered first: the engine refuses `UPDATE` and `DELETE`, even for administrators, and ledger digests detect tampering. Rejected because it can never be purged: Microsoft Learn lists deleting older data, `TRUNCATE TABLE` and partition `SWITCH` as unsupported, and a dropped ledger table stays physically in the database. The audit trail would grow without limit, against the storage limitation principle of the GDPR (article 5(1)(e)), since it holds user names.
- **One ledger database per year, dropped after export.** Rejected: several databases and SQL projects, and several connections in the VBE, for a very small volume.
- **Purge triggered by a volume threshold, or manual only.** Rejected: the retention would be unpredictable, and the GDPR requires a documented duration.

## Consequences

### Positive

- In V1, the commands a compromised Backend can run are limited to those both in the catalogue and in the local allowlist. (A compromised Backend can still read the collected logs and falsify what the dashboard shows.)
- A command runs at most once, never late, even across agent restarts.
- A clock drift is visible on the dashboard instead of silently breaking expiry.
- Every action is traceable, and tampering with the audit trail is **detected** by the chain verification.
- The audit trail has a documented, configurable retention, and its purge never loses the proof: the exported months stay verifiable.
- The mechanism is plain SQL (hashing and partitions) and can be ported to another engine.

### Negative

- Until command signing (V2), the agent trusts any allowed command coming from an authenticated Collector; the allowlist and the agent's validation of the Collector's certificate (ADR-0017) are the main safeguards.
- Each new command type needs a catalogue entry, an agent implementation, an allowlist entry and, on Linux, an exact sudoers entry.
- If the remote commands increment is dropped (ADR-0015), the stream, the allowlist and the command lifecycle must not ship half-finished.
- The hash chain **detects** tampering but does not prevent it: a database administrator can still change rows; the change is caught at the next verification. Detection depends on the anchors being out of reach of that person.
- The chain is security-sensitive code written in-house: its serialisation (the exact bytes hashed), the serialised inserts and the verification need their own tests, including tampering tests (changed, removed and inserted rows).
- Serialising the inserts makes the audit trail a single point of contention; with the expected volume (a few commands a day), this is acceptable.
- Whether the cold storage can offer a write-once share for the anchors (WORM) is **not verified** for the QNAP NAS; otherwise, the share rights described above apply.

## Open points

None.

## References

- Microsoft Learn, *Append-only ledger tables* and *Ledger considerations and limitations* (consulted 2026-11-04, for the rejected alternative).
- GDPR, article 5(1)(e) (storage limitation).
- ADR-0003, ADR-0004, ADR-0012, ADR-0014, ADR-0015, ADR-0017, ADR-0018.
