# Third reading: ADR-0019 to ADR-0022 (2026-11-05)

ADR-0019 to ADR-0022 were written after the second reading, to correct its findings. The project rules require a review after the decision, so the team read them again before the pull request. Each reviewer worked independently, read-only, without the main developer's conclusions. The main developer checked the key findings against the files and the sources before reporting them.

| Reviewer | Role | Scope | Prefix |
|---|---|---|---|
| Ackbar | security review (OWASP Top 10 2021) | ADR-0019, ADR-0020, ADR-0021, with ADR-0022 and the backup page | T- |
| Jocasta Nu | keeper of the ADRs | ADR-0019 to ADR-0022, the index, the pages changed with them | J- |
| Cassian | CVE and dependency audit | the version floors and the platforms named in the ADRs | C- |

## Outcome

Two blocking findings: the corrections of the second reading closed only half of findings B1 and B2. They require two new ADRs that supersede ADR-0019 and ADR-0020. The important findings are handled in a third ADR that complements the others, in editorial fixes, or in the administration pages. The product owner decided the four design choices on 2026-11-05 (one writer of the audit chain, blocking on tail truncation, signed packages for agent updates, a "monitor" identity for the health checks).

### Blocking

| # | Finding | Checked | Goes to |
|---|---|---|---|
| T-B1 | The API rereads the role of the user that the web site names, but nothing binds a request to a sign-in verified by the API: a compromised web site can still act as any active account. ADR-0019 states the opposite in its consequences. Jocasta Nu found the same (J-I1). | yes (ADR-0019 lines 42 and 66, ADR-0004 line 60) | ADR-0023, superseding ADR-0019 |
| T-B2 | On Windows, the account that creates a file chooses its rights at creation, and a file moved within a volume keeps its rights: "create files only" with `OWNER RIGHTS` removed does not make the audit archive write-once, and "temporary name, then rename" makes it easier. Jocasta Nu found the contradiction with ADR-0022's removal of temporary files (J-I3). | yes (Microsoft Learn, *DACL for a New Object*) | ADR-0024, superseding ADR-0020 |

### Important

| # | Finding | Checked | Goes to |
|---|---|---|---|
| T-I1 | The "command on the API host" has no defined channel; on the web site's socket it would let a compromised site weaken the settings. | reasoning | ADR-0023 (host channel) |
| T-I2 | An HMAC chain does not detect the removal of its last rows; command requests were not exported at once. | reasoning | ADR-0024 |
| T-I3 | The HMAC key is shared by several processes and by the verifier, without rotation; Jocasta Nu adds that no writer and no "verified" recorder is named (J-I5). | yes (ADR-0020 lines 39 and 61) | ADR-0024 (one writer, receipts); asymmetric signature and rotation in the future improvements |
| T-I4 | Configuration files with secrets copied into the deployment repository. | yes | ADR-0025, backup page |
| T-I5 | The agent update channel needed by the compromise procedure is not defined (also J-I6). | yes (grep) | ADR-0025 (signed RPM and MSI) |
| T-I6 | A token for a `HostId` that already has an active certificate lets a rogue machine replace a production one. | reasoning | ADR-0025 |
| T-I7 | The step-up list misses several sensitive actions, and a proof can be replayed for 5 minutes; reuse of a TOTP code not addressed. | reasoning | ADR-0023 |
| T-I8 | Alert recipients editable from the web site; no alert on privilege changes. | reasoning | ADR-0025 |
| J-I2 | The CIFS half of second-reading finding I8 (`uid`, `gid`, modes) was not handled. | yes (grep) | cold storage page |
| J-I4 | ADR-0022 adds links that ADR-0004 does not list; the verifier cannot reach the API, which listens on a Unix socket only. | yes | ADR-0025 (links, health service), editorial fixes |
| J-I7 | The work of ADR-0022 has no increment; ADR-0015 still says "hash chain". | yes | ADR-0025 (work per increment), editorial fix of ADR-0015 |
| C-I1 | CVE-2026-111497 (Brotli) is in the CISA catalogue of known exploited vulnerabilities (due date 2026-11-10); the documents did not say so. | release metadata: yes; CISA catalogue: as reported | dependencies page, editorial fix of ADR-0021 |
| C-I2 | "SDK 10.0.14" is wrong: the latest SDK is 10.0.403. | yes (release metadata) | dependencies page |
| C-I3, C-I4 | Ollama and llama.cpp floors correct but worded too narrowly; CVE-2026-107183 hits `llama-server` itself; never `--reranking`. | as reported | dependencies page |

### Minor

| # | Finding | Goes to |
|---|---|---|
| T-M1 | Data Protection: where the certificate's private key lives; fixed application name. | ADR-0023 |
| T-M2 | Two revalidation settings, not one (cookie and circuit). | ADR-0023 |
| T-M3 | Rate limit in the web site only; break-glass needed before 0.2. | ADR-0023 |
| T-M4 | `Dashboard.Crl` hardening; replay of an old Agent CA list over HTTP. | ADR-0025 |
| T-M5 | The 0.1 issuance register enters the audit trail without proof. | ADR-0025 ("not proven" mark) |
| T-M6 | Nobody can delete expired audit files on a server without WORM (GDPR). | ADR-0024, cold storage page |
| T-M7 | The agent's XML parser reads files that local non-root processes can write. | deferred to coding (below) |
| J-M | Duplicate reference in ADR-0019; "never receives a recovery code" vs codes shown once; overridden Identity methods; no backup retention; undefined terms in ADR-0022; stale counts in ADR-0016; revocation in 0.1; RFC 5280 wording; ambiguous deletion sentence in ADR-0020; cold storage page description; OS of the SQL Server VM; ADR-0004 line 61 now false; complemented ADRs without back-links. | ADR-0023, ADR-0024, ADR-0025, and editorial fixes |
| C-M | CVE attribution of the .NET floor; Fedora source-built package (CVE-2026-114148); `onnx` 1.22.0; NVD coverage incomplete for ONNX Runtime. | dependencies page, editorial fix of ADR-0021 |

### Editorial (done in place on 2026-11-05)

- ADR-0019 and ADR-0020: status set to superseded; the duplicate reference in ADR-0019 removed.
- References to ADR-0019 and ADR-0020 in the accepted ADRs (0004, 0005, 0015, 0021, 0022) and in the pages replaced by ADR-0023 and ADR-0024.
- ADR-0004: editorial notes on its two Negative lines about the web site (now handled by ADR-0023), and a pointer to the links added by ADR-0022 and ADR-0025.
- ADR-0015: "hash chain" replaced by "keyed audit trail", pointer to ADR-0025.
- ADR-0016: service and endpoint counts completed with `Dashboard.Crl`.
- ADR-0021: RFC 5280 wording marked not verified; CVE attribution of the .NET floor corrected, with the CISA mark.
- ADR-0022: its context sentence now states the link it adds to ADR-0004.
- Back-links from ADR-0004, 0006, 0008, 0012, 0015, 0016, 0018, 0021 and 0022 to the ADRs that complement them.

### Deferred to the time of coding

- Agent XML parser: `DtdProcessing = Prohibit` and `XmlResolver = null` (T-M7).
- PBKDF2 iteration count of the password hasher, checked against current guidance.
- Exact list of the overridden `UserManager` methods, with tests (ADR-0023).
- Whether `--api-key` of `llama-server` protects `/completion`, and whether Ollama's OpenAI route checks a key: tested during the benchmark.
- Behaviour of Samba `vfs_worm` on deletion and rename, and of the QNAP WORM folder, tested at installation.

## Not verified

- Windows behaviour of the CRL cache; Samba and QNAP write-once behaviour; whether ASP.NET Core Identity refuses a reused TOTP code; whether SMB 2 lets a client set the security descriptor at creation (the main developer expects so, from the Windows behaviour, but did not check the protocol).
- CISA catalogue entry and EPSS of CVE-2026-111497, the llama.cpp and Ollama CVEs: as reported by Cassian, not re-checked by the main developer.
