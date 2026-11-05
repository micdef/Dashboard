# Second reading of the ADRs (2026-11-04)

After every ADR was decided, the team reviewed them again, as the project rules require. Each reviewer worked independently, read-only, without the main developer's conclusions. The main developer checked the key findings against the files and the sources before reporting them.

| Reviewer | Role | Scope |
|---|---|---|
| Jocasta Nu | keeper of the ADRs | format, index, consistency between ADRs, cross-references, accuracy |
| Ackbar | security review (OWASP Top 10 2021) | design of ADR-0013, ADR-0014, ADR-0017, with ADR-0003, 0004, 0005, 0012, 0016, 0018 |
| Cassian | CVE and dependency audit | every platform and library named in the ADRs |
| Boba Fett | silent failures | every accepted ADR |

## Outcome

No finding questions the overall architecture. Two blocking security findings and several important ones require new ADRs; the rest is editorial or is deferred to the time of coding.

### Blocking (new ADRs)

| # | Finding | Reviewer | Checked | Goes to |
|---|---|---|---|---|
| B1 | A compromised web site process can act as any user, Admin included: the API trusts the identity sent by the site and relays password hashes (and TOTP keys) to it. ADR-0014 states the opposite in its consequences, which is wrong; ADR-0004 states it correctly. | Ackbar | yes | new ADR superseding ADR-0014 |
| B2 | The audit trail can be emptied through the official purge path: retention lowered to its minimum, Backend account with write access to the audit archive, the retention change itself purged later. The hash chain has no secret and only protects what an anchor covers; a "create-only" share does not stop the owner of a file from changing its rights. The purge rules are enforced by the caller, not by the procedure. | Ackbar, Boba Fett | yes | new ADR superseding ADR-0013 |

### Important (new ADRs)

| # | Finding | Reviewer | Checked | Goes to |
|---|---|---|---|---|
| I1 | ADR-0017 has the enrolment token created in the web site, but ADR-0015 places enrolment in 0.1, before the web site's accounts and the audit trail (0.2). | Jocasta Nu | yes | new ADR superseding ADR-0017 |
| I2 | The component that serves the revocation lists is named in no topology; agents need a third endpoint. | Jocasta Nu | yes | same |
| I3 | A revoked Collector certificate stays trusted until the cached Server CA list expires (12 months), according to the .NET source (`OpenSslCrlCache.cs`, Linux; Windows not verified). | Ackbar | partly (reasoning on the cited source) | same |
| I4 | .NET floor 10.0.10 is outdated: the latest 10.0 release is 10.0.14 (2026-10-25). It fixes CVE-2026-111497 (Brotli), reported in the CISA KEV catalogue (not re-checked by the main developer). Brotli decompression is off by default and must stay off. Other floors: llama.cpp build after b11935 and never `--rpc`; `onnx` 1.22.0; Ollama 0.35.0 if used; SQL Server 2025 latest CU. | Cassian | .NET release and the 10.0.14 notes: yes | same, and the dependency floors |
| I5 | No alert delivery channel ("raises an alert" appears about 15 times), no monitoring of the monitor itself, no database backup policy. | Boba Fett | yes | new ADR on operations |
| I6 | Export completeness is not reconciled: an ingestion-time watermark can skip rows of transactions still open; a checksum proves integrity, not completeness; a partial failure in a batch can be acknowledged; late metrics can arrive after aggregation. | Boba Fett | reasoning (SQL Server behaviour not verified) | same |
| I7 | Web site hardening: Data Protection key ring, TOTP keys and recovery codes at rest, Blazor circuits keep the old role up to 30 minutes, step-up before sensitive actions, security settings changes audited. | Ackbar | partly (Learn pages cited) | new ADR superseding ADR-0014 |
| I8 | CIFS mounts are shared by every local user unless `uid`, `gid` and file modes are set; the enrolment token must never be passed on the command line. | Ackbar | reasoning (man page cited) | ADRs on cold storage and identity |

### Editorial (done in place on 2026-11-05)

- Cross-references to superseded ADR-0001 and ADR-0007 replaced by ADR-0016 and ADR-0017 in ADR-0002 to ADR-0006.
- ADR-0002: the Learn page has two rows on partitioned table parallelism that disagree (Windows "Scalability and performance": Standard yes; "Data warehouse": Enterprise only; Linux page: Enterprise only). The ADR cited one row word for word; both are now cited. The reviewer had reported the ADR as wrong; the main developer found it incomplete instead.
- ADR-0018: the QNAP row is marked "not verified".
- The readme no longer calls the audit trail "immutable" but "tamper-evident".
- The index defines VBE, Backend and cold storage, and its note on "raised in review" points is updated.
- The QNAP page points to the single list of shares in `cold-storage.md`.

ADR-0013, ADR-0014 and ADR-0017 were left unchanged: they are superseded by the new ADRs, which correct them.

### Deferred to the time of coding

- Canonical byte format of the hash chain (field order, encoding, `datetime2` precision, NULL marker).
- Agent checks: maximum command lifetime, target `HostId`, configuration and trust anchors read-only for the agent user.
- `PasswordSignInAsync(..., lockoutOnFailure: true)`; second-factor failures counted for lockout; HSTS.
- An audited break-glass host command when the only Admin is locked out.
- Partition number computed inside the purge procedure; least-privilege user for `TRUNCATE`.
- AI model monitoring: model age, share of unknown templates, score distribution; distinct LLM failure reasons; pseudonymisation scan before writing prepared data; HMAC key backup.
- Agent counters (dropped, set aside, per-source liveness) persisted and reported in each heartbeat.
- Windows agent rights (Event Log Readers, never `SeSecurityPrivilege`): in the 0.4 ADR.
