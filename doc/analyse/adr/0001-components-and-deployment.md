# ADR-0001: Components and deployment topology

| Field | Value |
|---|---|
| Status | Accepted |
| Date | 2026-11-03 (proposed), 2026-11-04 (accepted) |
| Decider | Michaël Defraene (product owner, architect) |
| Drafted by | Claude (main developer); the decisions had a second reading by the architecture review agent during scoping |

## Context

Dashboard centralises the day-to-day supervision of a small fleet: five servers (Linux and Windows) and one workstation (Windows 11 with a WSL Fedora distribution). It collects heartbeats, inventory and logs, shows which machine has what, detects attacks and anomalies, writes reports with a local LLM and can run a fixed catalogue of commands on the machines.

The target infrastructure already exists: a Hyper-V host, an Nginx reverse proxy, a NAS, a firewall managed by the product owner, and a small dedicated machine for AI workloads (GMKtec NucBox G10). The database must be reachable by one process only, under one SQL login.

## Decision

The system is made of the following processes:

| Process | Host | Role |
|---|---|---|
| agent (.NET Worker Service, one binary) | every supervised machine | heartbeat, inventory, logs (journald, Nginx files, Windows Event Log), XML log parser module enabled by configuration, catalogue commands |
| Backend (collector and API) | Linux "app" VM | domain core: fleet, host state, rules, command lifecycle, ONNX model inference |
| data gateway (called the "very-back-end" in the first outline) | Linux "app" VM | the only process that connects to SQL Server |
| web site (Blazor Web App, Interactive Server render mode) | Linux "app" VM, published through the existing Nginx | user interface, accounts and roles |
| AI node | GMKtec NucBox G10, Linux | Python training, local LLM; no database access |
| SQL Server | dedicated Hyper-V VM | system of record |
| NAS | existing NAS | cold archive and prepared training data |

Deployment rules:

- The Backend, the data gateway and the web site run on the same Linux VM as **separate systemd services**, each under its **own dedicated system user**.
- The data gateway reaches the SQL Server VM through a **dedicated network interface**.
- The existing Nginx stays separate and publishes **only the web site**.
- Agents connect **directly** to the Backend. Certificate authentication only works behind a proxy if the proxy itself authenticates the client certificate and forwards the result (see ADR-0007).
- Firewall rules are designed and applied by the product owner.

## Alternatives considered

- **Everything on one host, including SQL Server.** Rejected: the database keeps its own VM, so its resources, backups and network exposure are managed separately.
- **Agent traffic through the existing Nginx.** Rejected: Nginx would have to authenticate the client certificates and forward them in a header, which moves the trust boundary to the proxy and couples the agent channel to the web front end.
- **The AI node with database access.** Rejected: the AI node only receives prepared files (ADR-0009) and computed facts; it never sees the database.

## Consequences

### Positive

- Each process has the smallest set of rights it needs (separate users, one SQL login, one process with database access).
- The AI node can be stopped or rebuilt without touching the data path.
- The web site and the agent channel can fail or be secured independently.

### Negative

- Three services on one VM share its resources; a runaway process can starve the others. Resource limits per systemd unit should be considered at deployment.
- More units to deploy and monitor than a single process.

## References

- ADR-0003, ADR-0004, ADR-0005, ADR-0007, ADR-0009.
- Microsoft Learn, *Configure certificate authentication in ASP.NET Core*, section "Review proxy and load balancer scenarios" (consulted 2026-11-03).
