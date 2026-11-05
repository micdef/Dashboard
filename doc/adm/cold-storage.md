# Cold storage

The cold storage holds the log archive, the prepared training data and the exported models (ADR-0012). Any SMB 3 file server can hold it, provided it meets the requirements of ADR-0018. Nothing on this page is tested yet.

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
| Audit archive, hourly files that are also the anchors (ADR-0020) | **create files only**: no modification or deletion, owner's implicit rights removed; WORM folder if the server offers one | no access | read only |

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
- The mount point path is given to the Collector and to the AI node in their configuration.
- The Collector checks that the path is a mount point and is writable, at startup and before each export.

## Checks after setup

- [ ] A share mounted with `seal` works from each host.
- [ ] A mount without encryption is refused.
- [ ] The AI node account cannot list the raw archive share.
- [ ] With the share unmounted, the Collector raises an alert and the export fails.
