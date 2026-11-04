# ADR-0016: Components and deployment topology, with the Enrolment service

| Field | Value |
|---|---|
| Status | Accepted |
| Date | 2026-11-04 |
| Decider | Michaël Defraene (product owner, architect) |
| Drafted by | Claude (main developer) |
| Supersedes | ADR-0001 |

## Context

ADR-0001 listed the processes of the system and their deployment. ADR-0007 then added a dedicated Enrolment service, which issues, renews and revokes the agent certificates. Under the rule of the ADR index, an accepted ADR is not rewritten: this ADR restates the complete topology and supersedes ADR-0001. The context of ADR-0001 (fleet, existing infrastructure, single database login) is unchanged.

## Decision

The system is made of the following processes:

| Process | Host | Role |
|---|---|---|
| agent (.NET Worker Service, one binary) | every supervised machine | heartbeat, inventory, logs (journald, Nginx files, Windows Event Log), XML log parser module enabled by configuration, catalogue commands |
| Backend: Collector | Linux "app" VM | serves the agents: ingestion, host state, real-time rules, command delivery (ADR-0005) |
| Backend: API | Linux "app" VM | serves the web site and the Enrolment service: user-facing services, authorisation, command creation (ADR-0005) |
| Enrolment service | Linux "app" VM | agent certificate authority: enrolment by token and CSR, renewal, revocation lists (ADR-0007) |
| data gateway (called the "very-back-end" in the first outline) | Linux "app" VM | the only process that connects to SQL Server |
| web site (Blazor Web App, Interactive Server render mode) | Linux "app" VM, published through the existing Nginx | user interface, accounts and roles |
| AI node | GMKtec NucBox G10, Linux | Python training, local LLM; no database access |
| SQL Server | dedicated Hyper-V VM | system of record |
| NAS | existing NAS | cold archive and prepared training data |
| offline root CA | kept offline by the product owner | root and server-issuing keys of the private CA (ADR-0007) |

Deployment rules:

- The two Backend processes, the Enrolment service, the data gateway and the web site run on the same Linux VM as **separate systemd services**, each under its **own dedicated system user**.
- The data gateway reaches the SQL Server VM through a **dedicated network interface**.
- The existing Nginx stays separate and publishes **only the web site**.
- Agents connect **directly** to the Collector and to the Enrolment service. Certificate authentication only works behind a proxy if the proxy itself authenticates the client certificate and forwards the result (ADR-0007).
- The Enrolment service has no access to the data gateway; it goes through dedicated API services (ADR-0007).
- Firewall rules are designed and applied by the product owner.

## Alternatives considered

- **Everything on one host, including SQL Server.** Rejected: the database keeps its own VM, so its resources, backups and network exposure are managed separately.
- **Agent traffic through the existing Nginx.** Rejected: Nginx would have to authenticate the client certificates and forward them in a header, which moves the trust boundary to the proxy and couples the agent channel to the web front end.
- **The AI node with database access.** Rejected: the AI node only receives prepared files (ADR-0009) and computed facts; it never sees the database.
- **Certificate authority keys on the "app" VM only.** Rejected: see ADR-0007; the root and server-issuing keys stay offline.

## Consequences

### Positive

- Each process has the smallest set of rights it needs (separate users, one SQL login, one process with database access, a certificate authority without database access).
- The AI node can be stopped or rebuilt without touching the data path.
- The web site, the agent channel and the enrolment channel can fail or be secured independently.

### Negative

- Five services on one VM share its resources; a runaway process can starve the others. Resource limits per systemd unit should be considered at deployment.
- More units to deploy and monitor than a single process.
- The agents reach two endpoints of the "app" VM (Collector, Enrolment service), which the firewall rules must allow.

## References

- ADR-0001 (superseded), ADR-0003, ADR-0004, ADR-0005, ADR-0007, ADR-0009.
- Microsoft Learn, *Configure certificate authentication in ASP.NET Core*, section "Review proxy and load balancer scenarios" (consulted 2026-11-03).
