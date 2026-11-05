# ADR-0024: Audit trail written by the data gateway and anchored by the independent verifier

| Field | Value |
|---|---|
| Status | Accepted |
| Date | 2026-11-05 |
| Decider | Michaël Defraene (product owner, architect) |
| Drafted by | Claude (main developer), after the third reading of 2026-11-05 |
| Classification | Security-sensitive [SEC] |
| Supersedes | ADR-0020 |

## Context

ADR-0020 kept the remote commands of ADR-0013 and protected the audit trail with an HMAC chain, exported every hour to a "write-once" archive that also served as the anchors. The third reading found that:

- on Windows, an account that can create files in a folder can give the new file its own access rights at creation (Microsoft Learn, *DACL for a New Object*: the DACL is the one the creator specifies, merged with inheritable entries), and a file moved within a volume keeps its rights. Removing `OWNER RIGHTS` therefore does not make the archive write-once, and the "temporary name, then rename" pattern of ADR-0012 makes it easier (T-B2);
- an HMAC chain does not detect that its last rows were deleted, and command requests were not in the immediate exports (T-I2);
- the HMAC key was shared by the API and the Collector, it was not said which process computes the chain, nor who records an export as "verified" before a purge (T-I3, Jocasta Nu);
- the rule of ADR-0022 that removes temporary files cannot apply to an archive where the account cannot delete (Jocasta Nu);
- on a server without a WORM folder, nobody can delete expired archive files (T-M6).

This ADR keeps every decision of ADR-0020 except where it says otherwise.

## Decision

### Kept from ADR-0020

- **Remote commands** unchanged (catalogue, local allowlist, at most once, least privilege, never a shell, signing in V2); roles and step-up in ADR-0023.
- **What is audited**, and **fail-closed**: an action takes place only if its audit row is written.
- A regular table **partitioned by month**, insert only through a stored procedure; command output outside the audit trail, with its hash in the row.
- An **HMAC-SHA256** per row over its canonical content and the previous HMAC; the key never in the database.
- **Retention** values, defaults (2 years in the database, 5 years in total) and minimums (1 year and 2 years); raising from the web site after step-up, **lowering only through the host channel** (ADR-0023); every change audited.
- The **purge procedure** reads the retention, refuses partitions that are too young, computes the partition number itself, and raises an alert on refusal.

### One writer: the data gateway

- The **data gateway (VBE)** is the only process that holds the HMAC key and computes the chain. The API and the Collector send the content of an audit event; they never see the key.
- In one transaction, under a lock, the VBE reads the head of the chain, computes the new row's HMAC, and inserts it. Each row carries a **sequence number** without gaps and a **key identifier**, so that a later key rotation (see [future improvements](../future-improvements.md)) does not change the row format.
- The key file is readable only by the VBE's system user.

### Tail truncation is detected and blocks

- After each commit, the VBE records the sequence number and HMAC of the head in a file **outside the database**, on the "app" VM.
- At startup and before each audit write, the VBE checks that the head in the database is the recorded head or a valid successor of it in the chain. If the database is **behind** (rows removed at the end), the VBE raises a **critical alert** and **refuses every new audit write**, so commands and account changes are blocked (fail-closed), until the product owner acknowledges the incident with a host command (ADR-0023). The acknowledgement is itself the first audit row written afterwards.
- Command requests are added to the events that trigger an **immediate export** (with account, role, security setting, enrolment and revocation events).

### Export files: written once, never renamed

- Every hour, and after each event listed above, the Collector exports the new audit rows (read through the VBE) as one file in the audit archive. The file is **created under its final name, in exclusive mode** (the creation fails if the name exists), and ends with a closing line giving its row count, its first and last sequence numbers and the HMAC of its last row. A file without its closing line is incomplete: it is alerted and the next export covers its rows again in a new file.
- Audit files are never renamed and never deleted by Dashboard. The "temporary name, then rename" pattern of ADR-0012 and the removal of temporary files of ADR-0022 **do not apply** to the audit archive.

### The anchor is the verifier's own copy

- The **independent verification tool** runs on another machine than the "app" VM, under a **systemd timer, every hour**. At each run it copies the new audit files into its **own store**, where it only adds, and checks the whole chain against its copy.
- **Its copy is the official anchor.** A file of the archive that differs from the verifier's copy, a gap in the sequence numbers, or a newest file older than one hour plus a configured margin, raises a **critical alert** sent by the tool itself (ADR-0022).
- For each file it has copied and verified, the tool writes a **receipt** in a receipts folder where only its account can write: the file name, its last sequence number and its last HMAC, authenticated with a key derived from the HMAC key under a distinct label.
- The tool holds a copy of the HMAC key to verify the chain; it never connects to the database (ADR-0003).

### Purge needs the verifier's receipts

- The Collector passes the receipts covering a partition to the purge request. The VBE checks each receipt with the derived key before it calls the purge procedure, and the procedure checks that the receipts cover **every sequence number** of the partition.
- A compromised Collector therefore cannot declare an export verified: it has no key to make a receipt.

### Write-once on the file server: a second layer, stated honestly

- **A plain share with "create files only" rights does not make the archive write-once**, on Windows at least, because the creator chooses the rights of the new file. The guarantee rests on the verifier's copy.
- Where the file server offers it, server-side write-once is added as a second layer: a WORM folder (QNAP), the Samba `vfs_worm` module (it refuses writes after a grace period; its behaviour on deletion and rename is **not verified**), or on Windows a server-side task under another account that moves each closed file into a folder where the Backend account can only read, and resets its rights by inheritance.
- The installation guide states which layer is in place.

### Deleting expired archive files

- Archive files whose total retention has passed are deleted by the WORM retention where the server has one; otherwise by a documented **administration procedure**, run by an administrator account of the file server, never by a Dashboard account. The verifier's copy follows the same retention.
- The total retention of 5 years must be justified in the legal basis document of the project (GDPR, article 5(1)(e)); the audit holds user names.

## Alternatives considered

- **Keep ADR-0020: anchors on the archive, protected by share rights.** Rejected: the share rights do not hold against the account that writes the files.
- **The API as the only writer.** Not adopted: the Collector would need a new link to the API to audit command delivery, while the VBE already writes every audit row and faces no network.
- **A critical alert only on tail truncation, without blocking.** Rejected: the system would keep acting on an audit trail known to be damaged.
- **Asymmetric signature of the exports, the verifier holding only the public key.** Deferred to [future improvements](../future-improvements.md): it closes the case of a compromised verifier forging rows, at the cost of a second key pair and a rotation procedure. The key identifier in each row prepares it.
- **Receipts recorded by the Collector in the database.** Rejected: the Collector would declare its own exports verified.

## Consequences

### Positive

- Forging rows needs the database and the VBE's key; already copied hours cannot be changed without the verifier noticing.
- Removing the last rows of the chain is detected and stops the system until someone looks at it.
- A purge cannot remove rows that the verifier has not copied and verified.
- The guarantee does not depend on what the file server can enforce.

### Negative

- The verifier becomes essential: if it stops, no receipt is written and no audit partition can be purged; its last run is shown on the dashboard and its silence is alerted (ADR-0022).
- The verifier holds a copy of the HMAC key: a compromise of the verifier's machine allows forging rows that the verifier itself would accept (deferred fix above).
- A compromise of the "app" VM as root gives the key: new rows can be forged from then on; hours already copied by the verifier stay proven.
- A tail truncation blocks commands and account changes until the product owner acknowledges it on the host. A database restored from a backup is also behind the recorded head: the block is then expected, and the restore procedure includes its acknowledgement ([backup and recovery](../../adm/backup-and-recovery.md)).
- One more file on the "app" VM to protect (the recorded head), and a receipts folder on the cold storage.

## References

- Microsoft Learn, *DACL for a New Object* (consulted 2026-11-05); *File Security and Access Rights* and *Permissions when copying and moving files*, cited by the third reading.
- Samba, *vfs_worm(8)*, cited by the third reading.
- GDPR, article 5(1)(e) (storage limitation).
- [Third reading report](../reviews/2026-11-05-third-reading.md), findings T-B2, T-I2, T-I3, T-M6 and J-I3, J-I5.
- ADR-0003, ADR-0012, ADR-0018, ADR-0020 (superseded), ADR-0021, ADR-0022, ADR-0023, ADR-0025.
