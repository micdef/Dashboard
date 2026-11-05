# ADR-0020: Remote commands, and a keyed audit trail that cannot be purged early

| Field | Value |
|---|---|
| Status | Accepted |
| Date | 2026-11-05 |
| Decider | Michaël Defraene (product owner, architect) |
| Drafted by | Claude (main developer), after the second reading of 2026-11-04 |
| Classification | Security-sensitive [SEC] |
| Supersedes | ADR-0013 |

## Context

ADR-0013 defined remote commands and an audit trail protected by a SHA-256 hash chain, purgeable after a verified export. The second reading (finding B2) showed that the audit trail could be emptied through the official purge path: an Admin or a compromised web site could lower the retention to its minimum, the Backend account could rewrite the exported files, and the retention change itself would be purged later. It also showed that a plain hash chain has no secret (a database administrator can change a row and recompute every hash after it), that a "create-only" share does not stop the owner of a file from changing its rights, that the purge rules were enforced by the caller rather than by the procedure, and that a failed audit write was not said to block the action.

This ADR keeps the remote commands of ADR-0013 unchanged and replaces its audit trail.

## Decision

### Kept from ADR-0013: remote commands

- **No SSH.** The agent opens a bidirectional gRPC stream to the Collector over mTLS (ADR-0004, ADR-0021); commands arrive on this stream. No inbound port is opened on the machines.
- Commands are **typed messages from a catalogue**, never free text. Each machine has a **local allowlist** that only root (or an administrator on Windows) can modify; typed parameters are matched exactly against it.
- The roles allowed to issue commands, and the step-up required before issuing one, are decided in ADR-0019.
- **At most once**: each command carries a unique identifier and an expiry timestamp; the expiry is checked by the Collector and by the agent; the agent keeps the seen identifiers on disk until their expiry; the heartbeat reports the clock offset, and the agent refuses commands when its offset exceeds a configured tolerance.
- **Least privilege, never a shell**: maximum duration and capped output; on Linux, `sudo` with an argument list and exact sudoers entries, never an interactive command or one that can open a pager; on Windows, a dedicated service account, never LocalSystem (exact rights in 0.4).
- Signing of commands is planned for V2.

### What is audited, and fail-closed

- Every step of a command is audited: request, delivery, and outcome (executed, refused, timed out, expired, result lost), with the requesting user and the role used. So are account changes, role changes, changes of security settings (ADR-0019), machine enrolments and revocations (ADR-0021), and the creation of the first Admin.
- **Fail-closed**: an action takes place only if its audit row is written. The audit row and the action are written in the same transaction, or the audit row is written first; if it cannot be written, the action is refused. A command is never delivered without its audit row.
- Command output is not stored in the audit trail: the audit row keeps its hash, and the output is stored in a regular table, purged like the logs (ADR-0012).

### Keyed chain

- The audit trail is a regular table, **partitioned by month**, on which the application account can only insert, through a stored procedure (ADR-0003).
- Each row stores an **HMAC-SHA256** computed over its own content (in a canonical byte format, fixed at coding time) and the HMAC of the previous row.
- The **HMAC key is never in the database**. It is a file on the "app" VM, readable only by the Backend processes that write audit rows; it is backed up offline by the product owner, like the Server CA key (ADR-0021). A database administrator alone cannot change, remove or insert a row without breaking the chain.
- The insert procedure accepts a row only if the previous HMAC it carries is the current head of the chain, under a lock: one chain, no gap.

### Hourly export to a write-once archive

- **Every hour**, and right after each security event (account, role, security setting, enrolment, revocation), the Collector exports the audit rows written since the previous export to the cold storage (ADR-0018), as one file that ends with the HMAC of its last row. These files are both the **archive** and the **anchors** of the chain.
- The audit archive is a **write-once** location: a WORM folder when the file server offers one; otherwise an account that can create files but not modify or delete them, **with the owner's implicit rights removed** (`OWNER RIGHTS` on Windows; the equivalent on Samba and QNAP is **not verified**). The Backend account cannot rewrite or delete an exported file.
- Archive files are deleted only when their total retention has passed, by the retention of the write-once folder where available.

### Independent verification

- An **independent verification tool** checks the chain across the archive files and against its **own copy** of them. It runs on another machine than the "app" VM, under an account that can only read the audit archive, and holds its own copy of the HMAC key.
- It never connects to the database (ADR-0003): it works only on the exported files.
- It is run by the product owner, on demand or on a schedule (systemd timer); its result and the age of its last run are reported on the dashboard. The Backend's own verification is kept, as an early warning, but it is not the proof.

### Retention and purge

- **Retention values** (database and total) are stored in the database. Defaults: **2 years** in the database, **5 years** in total. Minimums: **1 year** in the database, **2 years** in total, and the total never shorter than the database retention. They are checked by the procedure that changes them.
- **Raising** a retention is possible from the web site (Admin, after step-up, ADR-0019). **Lowering** it is possible **only with the installation command on the API host**. Every change is audited.
- The **purge procedure itself**:
  - reads the retention values;
  - refuses any partition younger than the database retention;
  - refuses any partition whose rows are not all covered by exported archive files recorded as written and verified;
  - computes the partition number itself from a date, never receiving it from the caller.
  A refusal raises an alert. The partition is then truncated (`EXECUTE AS`, ADR-0003).

## Alternatives considered

- **Plain SHA-256 chain with anchors (ADR-0013).** Rejected: a database administrator can rewrite the tail since the last anchor, and the anchors could be rewritten by the Backend account.
- **Append-only ledger table.** Rejected in ADR-0013: it can never be purged.
- **Retention changes from the web site in both directions.** Rejected: a stolen Admin session would be enough to empty the audit trail.
- **Minimums of 90 days in the database.** Rejected: too short to prove who did what.
- **Verification only on the Backend.** Rejected: the verification would run on the machine it is meant to watch.
- **A read-only SQL login for the verification tool.** Rejected: it would break the single login of ADR-0003; the tool works on the exported files instead.
- **Best-effort audit writes, retried later.** Rejected: a gap in the audit trail at the worst moment.

## Consequences

### Positive

- Forging the audit trail without detection needs both the database and the HMAC key on the "app" VM, and the exported hours cannot be rewritten afterwards.
- No action happens without its audit row.
- The audit trail cannot be shortened from the web site, nor purged before its retention or before it is archived, whatever the caller asks.
- The proof does not depend on the Backend: the product owner verifies it with his own tool and his own copy.
- The exported files are at most one hour behind, so a loss of the database loses at most one hour of audit.

### Negative

- The HMAC key is a secret on the "app" VM, to be protected, backed up offline and kept for as long as the archive; losing it makes the existing chain unverifiable.
- A compromise of the "app" VM gives the key: from then on, new rows can be forged, but already exported hours cannot be changed.
- The write-once location depends on the file server; on QNAP and Samba it is **not verified**, and the installation must check it.
- One more tool to write, test and run (the independent verifier), with its own copy of the key.
- A slow or unavailable database blocks commands and account changes (fail-closed).
- The canonical byte format of the rows is a contract: a later port to another engine must keep it, so that the existing chain stays verifiable.

## References

- Microsoft Learn, *Append-only ledger tables* and *Ledger considerations and limitations* (consulted 2026-11-04, for the rejected alternative).
- Microsoft Learn, *Owner of a New Object* (owner's implicit rights), cited by the second reading; not re-read by the main developer.
- GDPR, article 5(1)(e) (storage limitation).
- [Second reading report](../reviews/2026-11-04-second-reading.md), finding B2.
- ADR-0003, ADR-0004, ADR-0012, ADR-0013 (superseded), ADR-0015, ADR-0021, ADR-0018, ADR-0019.
