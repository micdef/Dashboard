# NAS: QNAP TS-133

Deployment notes for the NAS of the home lab, which holds the cold storage: the log archive, the prepared training data and the exported models (ADR-0012, ADR-0018). The general requirements and the other platforms are in [cold-storage.md](cold-storage.md). The settings below come from QNAP's documentation; they are **not yet tested** on this NAS.

## Hardware

| Item | Value |
|---|---|
| Model | QNAP TS-133 (one drive bay) |
| Processor | 4-core ARM Cortex-A55, 1.8 GHz |
| Memory | 2 GB, not expandable |
| Encryption | hardware encryption engine; AES 256-bit volume encryption (FIPS 140-2 CAVP validated, according to the specification page) |

Source: QNAP, *TS-133 Specification*, https://www.qnap.com/en/product/ts-133/specs (consulted 2026-11-04).

## Encryption at rest

- The shared folders used by Dashboard are placed on an **encrypted volume** (AES 256-bit).
- The encryption key is stored outside the NAS. Whether the volume unlocks automatically at startup is a choice to make at deployment: automatic unlocking keeps the key on the NAS and protects only against the theft of the drive alone.

## SMB 3 with encryption

According to QNAP's FAQ *How do I enable and manage SMB encryption in QNAP NAS* (consulted 2026-11-04):

1. **Control Panel** > **Network & File Services** > **Win/Mac/NFS** > **Microsoft Networking** > **Advanced Options**.
2. Set **Highest SMB version** to **SMB 3.0** (or later), then **Apply**.
3. Enable encryption on the shared folders used by Dashboard. In older QTS versions this is the **SMB Encryption** option in the folder's **Edit Properties** (**Privilege Settings** > **Shared Folders**); the exact place in QTS 5 is **to check on the NAS**.

On the Linux hosts, the shares are mounted with `mount.cifs` and the `seal` option (encryption required), with `vers=3.0` or later.

QNAP notes that encryption costs processor time. The TS-133 has an ARM processor; QNAP's mention of AES-NI acceleration in the QTS 5.0.1 release notes concerns x86 processors, so its benefit here is **not verified**. The archive files are small (hourly, gzip-compressed), so the throughput needed is low.

## Accounts and shares

| Share | Backend account | AI node account |
|---|---|---|
| Raw log archive | read and write | no access |
| Prepared training data (pseudonymised) | read and write | read only |
| Models | read only | read and write |

## Checks after setup

- [ ] A share mounted with `seal` from a Linux host works.
- [ ] A mount without encryption is refused by the shares used by Dashboard (to check: whether QTS can require encryption rather than only allow it).
- [ ] The AI node account cannot list the raw archive share.
- [ ] The volume is encrypted, and the key is stored outside the NAS.
- [ ] Measured write throughput of an hourly export file.
