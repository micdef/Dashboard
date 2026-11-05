# Dependencies and version floors

Minimum versions of the platforms and components named in the ADRs. The rule is set in ADR-0021: follow the latest patch, and never go below the floor listed here. This page is checked again by the CVE audit before each release; "no known vulnerability" only means none known on the date of the check.

Last check: 2026-11-05 (CVE audit of the third reading); .NET release metadata re-checked by the main developer on 2026-11-05.

| Component | Floor | Why | Rules |
|---|---|---|---|
| .NET 10 runtime | **10.0.14** | latest 10.0 release (2026-10-25). It fixes CVE-2026-111497 (Brotli decoder), which is in the **CISA catalogue of known exploited vulnerabilities** (added 2026-10-27, due date 2026-11-10), and includes the fixes of earlier patches (CVE-2026-57108 and CVE-2026-50528 since 10.0.10) | latest 10.0 patch on the "app" VM and on every agent; Brotli decompression never enabled (off by default, according to the 10.0.14 release notes) |
| .NET 10 SDK | **10.0.403** | latest SDK of the 10.0 channel on 2026-11-05 | development machines and build agents |
| .NET from the Fedora repositories | the package must carry runtime 10.0.14 or later | the source-built Linux archives have their own CVE (CVE-2026-114148) | check with `dotnet --list-runtimes`; if the package lags, use the Microsoft install script (not verified) |
| Other users of the Brotli library | Brotli 1.2.1, 1.1.1 or 1.0.10 or later | same CVE-2026-111497, outside .NET (for instance an Nginx Brotli module or the Python `brotli` package) | as reported by the audit; which components embed Brotli is not verified |
| SQL Server 2025 | latest CU (CU10, 17.0.5035.5, on 2026-10-22), or the latest GDR of the chosen track | security fixes; no SQL Server 2025 CVE in the CISA catalogue on 2026-11-05 | check with `SELECT @@VERSION;` |
| llama.cpp `llama-server` | build **b11935 or later**, at the time of the benchmark | memory-safety CVEs in 2026: the RPC backend (CVE-2026-119052, fixed in b11935), and `llama-server` itself (CVE-2026-107183, unauthenticated use-after-free on `POST /completion`, fixed in b11393) | never `--rpc`, never `rpc-server`; never `--reranking` (CVE-2026-52132, no fixed build cited); `--sleep-idle-seconds` off until its fix is confirmed; trusted GGUF files only; `--api-key` and the firewall rule kept. Whether `--api-key` protects `/completion` is **not verified**: test it during the benchmark |
| Ollama (fallback only) | latest release, never below **0.35.0** | CVE-2026-103663 (path traversal on `/api/pull`, fixed in 0.35.0); other advisories without a fixed version (model management without authentication) | localhost or firewall only; `/api/pull` and `/api/create` never exposed; whether its OpenAI route checks an API key is **not verified** |
| `onnx` (Python, through skl2onnx) | **1.22.0** | 2026 advisories on 1.21.0 for loading and saving models | load only models produced by the pipeline |
| ONNX Runtime (NuGet and PyPI) | latest stable | no advisory in OSV on 2026-11-05 (NVD search incomplete) | models only from the pipeline's own share |
| `Microsoft.Data.SqlClient` | latest stable, at least 5.2.2 or 6.x | old advisories only | |
| `scikit-learn` | latest, at least 1.5.0 | CVE-2024-5206 | |
| `protobuf` (Python, if used) | 6.33.5 or 5.29.6 | CVE-2026-0994 | |
| Samba (if used as cold storage) | latest distribution package | 2026 CVEs in printing, password scripts and some VFS modules | minimal `smb.conf`: no printing, no `check password script`, no domain controller role |
| QNAP QTS | latest firmware | several 2025-2026 CVEs, one without authentication (CVE-2025-66277) | NAS never exposed to the internet; unused services disabled |

Sources: the CVE audits of the [second reading](reviews/2026-11-04-second-reading.md) and of the [third reading](reviews/2026-11-05-third-reading.md), which list the advisories and the links checked. Items the main developer did not re-check are as reported by those audits.

Once projects exist, the audit also runs `dotnet package list --vulnerable --include-transitive`.
