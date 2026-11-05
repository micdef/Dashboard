# Cold storage

The cold storage holds the log archive, the prepared training data and the exported models (ADR-0012), the audit archive and its receipts (ADR-0024), and the database backups (ADR-0022). Any SMB 3 file server can hold it, provided it meets the requirements of ADR-0018. Nothing on this page is tested yet.

## Requirements

- [ ] The file server is **another machine than the database host**.
- [ ] SMB 3 encryption is **required** on the shares: clients that do not encrypt are refused.
- [ ] SMB 1 is disabled.
- [ ] The volume that holds the shares is encrypted at rest.
- [ ] Shares and accounts follow the table below.

| Share | Backend account | AI node account | Audit verifier account |
|---|---|---|---|
| Raw log archive | read and write | no access | no access |
| Prepared training data (pseudonymised) | read and write | read only | no access |
| Models | read only | read and write | no access |
| Database backups (ADR-0022): written by the SQL Server service account only | no access | no access | no access |
| Audit archive, hourly files (ADR-0024) | **create files only**, no modification or deletion; WORM folder or server-side task if the server offers one (see below) | no access | read only |
| Audit receipts (ADR-0024) | read only | no access | **create files only** |

The anchor of the audit trail is the verifier's own copy, on its machine (ADR-0024). **"Create files only" rights do not make a share write-once**: on Windows, the account that creates a file chooses its rights at creation. Server-side write-once is a second layer, where the server offers one:

- QNAP: a WORM folder for the audit archive;
- Samba: the `vfs_worm` module (its behaviour on deletion and rename is not verified: test it before relying on it);
- Windows Server: a scheduled task under another account that moves each closed audit file into a folder where the Backend account can only read, and resets its rights by inheritance.

Write down which layer is in place in the installation record.

Expired audit files (total retention passed, ADR-0024) are deleted by the WORM retention where there is one; otherwise by an administrator account of the file server, never by a Dashboard account. Delete the verifier's copy of the same files at the same time.

## Server side

### QNAP

See [nas-qnap-ts-133.md](nas-qnap-ts-133.md).

### Windows Server

From an elevated PowerShell session (Microsoft Learn, *SMB security enhancements*):

```powershell
New-SmbShare -Name <sharename> -Path <pathname> -EncryptData $true
```

- `RejectUnencryptedAccess` must stay at `$true` (its default), so that clients that do not encrypt are refused: `Get-SmbServerConfiguration | Select-Object Encrypt*, RejectUnencryptedAccess`.
- SMB 1 must be disabled: Microsoft Learn notes that otherwise a connection can be downgraded to SMB 1, which has no encryption.
- Encryption at rest: BitLocker on the volume.

### Linux with Samba

In `smb.conf`, globally or in each share (Samba, *smb.conf(5)*):

```ini
server smb encrypt = required
```

- With `required`, clients that do not support encryption are denied access.
- Encryption at rest: LUKS on the volume.

## Client side (Collector and AI node hosts)

The shares are mounted by the operating system; the code only sees a directory (ADR-0018).

- Mount with `mount.cifs`, options `seal` and `vers=3.0` (or later), and a credentials file readable by root only.
- Set the owner and the modes of the mount, so that only the service that uses it can read it: `uid=<service user>`, `gid=<service group>`, `file_mode=0640`, `dir_mode=0750` (see *mount.cifs(8)*). Without them, other local users of the host may read the share.
- The mount point path is given to the Collector and to the AI node in their configuration.
- The Collector checks that the path is a mount point and is writable, at startup and before each export.

## Checks after setup

- [ ] A share mounted with `seal` works from each host.
- [ ] A mount without encryption is refused.
- [ ] The AI node account cannot list the raw archive share.
- [ ] With the share unmounted, the Collector raises an alert and the export fails.
