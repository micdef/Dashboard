# Dependencies and version floors

Minimum versions of the platforms and components named in the ADRs. The rule is set in ADR-0021: follow the latest patch, and never go below the floor listed here. This page is checked again by the CVE audit before each release; "no known vulnerability" only means none known on the date of the check.

Last check: 2026-11-04 (CVE audit of the second reading), .NET release re-checked on 2026-11-05.

| Component | Floor | Why | Rules |
|---|---|---|---|
| .NET 10 runtime and SDK | **10.0.14** | latest 10.0 release (2026-10-25); fixes CVE-2026-57108, CVE-2026-50528, CVE-2026-111497 (Brotli) and others | latest 10.0 patch on the "app" VM and on every agent; Brotli decompression never enabled |
| SQL Server 2025 | latest CU (CU10, 17.0.5035.5, on 2026-10-22), or the latest GDR of the chosen track | security fixes | check with `SELECT @@VERSION;` |
| llama.cpp `llama-server` | a build after **b11935** at the time of the benchmark | memory-safety CVEs in 2026, including the RPC backend | never `--rpc`, never `rpc-server`; `--sleep-idle-seconds` off until its fix is confirmed; trusted GGUF files only; `--api-key` and the firewall rule kept |
| Ollama (fallback only) | **0.35.0** | CVE-2026-103663 (path traversal on `/api/pull`) and others; model management without authentication | localhost or firewall only; `/api/pull` and `/api/create` never exposed |
| `onnx` (Python, through skl2onnx) | **1.22.0** | 2026 advisories on loading and saving models | load only models produced by the pipeline |
| ONNX Runtime (NuGet and PyPI) | latest stable | no advisory found on 2026-11-04 | models only from the pipeline's own share |
| `Microsoft.Data.SqlClient` | latest stable, at least 5.2.2 or 6.x | old advisories only | |
| `scikit-learn` | latest, at least 1.5.0 | CVE-2024-5206 | |
| `protobuf` (Python, if used) | 6.33.5 or 5.29.6 | CVE-2026-0994 | |
| Samba (if used as cold storage) | latest distribution package | 2026 CVEs in printing, password scripts and some VFS modules | minimal `smb.conf`: no printing, no `check password script`, no domain controller role |
| QNAP QTS | latest firmware | several 2025-2026 CVEs, one without authentication (CVE-2025-66277) | NAS never exposed to the internet; unused services disabled |

Sources: the CVE audit of the [second reading](reviews/2026-11-04-second-reading.md), which lists the advisories and links checked. Items the main developer did not re-check are as reported by that audit.
