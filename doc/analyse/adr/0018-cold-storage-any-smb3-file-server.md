# ADR-0018: Cold storage on any SMB 3 file server, mounted by the operating system

| Field | Value |
|---|---|
| Status | Accepted |
| Date | 2026-11-04 |
| Decider | Michaël Defraene (product owner, architect) |
| Drafted by | Claude (main developer) |

## Context

ADR-0012 places the log archive, the prepared training data and the exported models on "the NAS", reached through SMB 3 with encryption and separate accounts. The home lab uses a QNAP NAS, but someone who clones the repository may have no NAS and use a share on a Windows or Linux server instead. The project must not depend on one storage product, and its code must not change when the storage changes.

This ADR complements ADR-0012; it does not replace it. Its decisions on protocol, accounts and retention remain valid.

## Decision

- **"The NAS" is a role, not a product**: the cold storage. Any file server that meets the requirements below can hold it.
- **Requirements of the cold storage:**
  - SMB 3 with encryption **required** (clients that do not encrypt are refused), and SMB 1 disabled;
  - the separate accounts and share rights of ADR-0012;
  - encryption at rest of the volume that holds the shares.
- **Platforms expected to meet them**, according to their documentation (documented in the administration guide; none is tested yet):

  | Platform | Encryption required | Encryption at rest |
  |---|---|---|
  | QNAP (QTS) | SMB 3 and encryption on the shared folders; whether QTS can **require** encryption, and not only allow it, is **not verified** | AES 256-bit volume encryption |
  | Windows Server | `Set-SmbShare -EncryptData $true`, with `RejectUnencryptedAccess` left at `$true` | BitLocker |
  | Linux with Samba | `server smb encrypt = required`, globally or per share | LUKS |

- **The code never speaks SMB.** The Collector and the AI node read and write a **directory mounted by the operating system**, whose path comes from the configuration. The mount (`mount.cifs` with `seal` and `vers=3.0` or later) handles the protocol, the encryption and the credentials. In the hexagonal cores (ADR-0006), the archive is a port, and its only V1 adapter is a file system adapter.
- **The cold storage is on another machine than the database.** Otherwise a single disk failure would lose both hot and cold data, and the archive would no longer be a copy.

## Alternatives considered

- **An SMB client library inside the code.** Rejected: the code would carry the protocol, the encryption settings and the credentials, and would have to change for another storage.
- **Requiring a NAS.** Rejected: it would exclude people who have a file server but no NAS, for no technical reason.
- **Object storage (S3-compatible, such as MinIO).** Not adopted for V1: another protocol and another adapter, with no need yet. The archive port leaves the door open.

## Consequences

### Positive

- Changing the storage is a deployment change, never a code change.
- In development, the archive can be a local directory, with no file server at all.
- The requirements can be checked the same way on every platform.

### Negative

- The mount is a dependency of the Collector and of the AI node: if it is missing, writes would go to the empty local mount point. The Collector checks at startup and before each export that the path is really a mount point (and writable), and raises an alert otherwise; the export then fails and blocks the purge (ADR-0012).
- The mount credentials are stored on each host (a credentials file readable by root only) and must be rotated there.
- Each platform needs its own setup instructions in the administration guide.

## References

- Microsoft Learn, *SMB security enhancements*, section "Enable SMB Encryption", https://learn.microsoft.com/windows-server/storage/file-server/smb-security (consulted 2026-11-04).
- Samba, *smb.conf(5)*, parameter `server smb encrypt`, https://www.samba.org/samba/docs/current/man-html/smb.conf.5.html (consulted 2026-11-04).
- QNAP, *How do I enable and manage SMB encryption in QNAP NAS* (consulted 2026-11-04).
- ADR-0004, ADR-0006, ADR-0012.
