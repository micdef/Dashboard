# ADR-0006: Hexagonal cores where there is domain logic, no CQRS

| Field | Value |
|---|---|
| Status | Accepted |
| Date | 2026-11-03 (proposed), 2026-11-04 (accepted) |
| Decider | Michaël Defraene (product owner, architect) |
| Drafted by | Claude (main developer); the decisions had a second reading by the architecture review agent during scoping |

## Context

The product owner first considered clean architecture (ports and adapters) applied to every component. An architect consulted earlier criticised a CQRS proposal as over-engineered for this size. Microsoft documents Clean, Hexagonal (ports and adapters) and Onion architectures as names for the same family. Applying the pattern to every component would add layers to components that only translate data.

A supervision tool must also never present missing data as good news: a collection that failed must not look like "nothing to report".

## Decision

### Where the hexagon applies

- **No CQRS.**
- A real **hexagonal core** (ports and adapters) exists only where there is real domain logic:
  - **Fleet** (shared by the Collector and the API, ADR-0005): host state derived from heartbeats (time through `TimeProvider`), command lifecycle, deterministic rules, permissions per role.
  - **Agent**: XML log parsing engine (a pure function), local buffer with resume, command allowlist; Linux and Windows adapters around it.
- The data gateway and the web site have **no hexagon**: the data gateway is a pure adapter (ADR-0003) and the web site is a user interface over the API. Each is a single project.

### One project per layer in the hexagonal cores

| Layer | Content | May reference |
|---|---|---|
| `*.Domain` | entities, value objects, domain rules | nothing (base class library only) |
| `*.Application` | use cases and ports (interfaces the core needs) | `*.Domain` |
| `*.Infrastructure` | adapters implementing the ports (data gateway client, operating system access, clock, LLM client…) | `*.Application`, external packages |
| host | composition root and inbound adapters (gRPC services, worker loop) | all of the above |

- An **architecture test** fails if a `*.Domain` or `*.Application` project references gRPC, `Microsoft.Data.SqlClient` or ASP.NET Core, or if a layer references a layer it may not reference.

### Solution structure

| Path / project | Content |
|---|---|
| `src/contracts/protos` | `.proto` files (ADR-0004) |
| `Dashboard.Contracts.Edge`, `Dashboard.Contracts.Internal` | generated C# contracts |
| `Dashboard.Agent.Domain`, `.Application`, `.Infrastructure` | agent core; Linux and Windows adapters in `Infrastructure` |
| `Dashboard.Agent` | agent host |
| `Dashboard.Fleet.Domain`, `.Application`, `.Infrastructure` | fleet core shared by the Backend processes |
| `Dashboard.Collector`, `Dashboard.Api` | the two Backend hosts (ADR-0005) |
| `Dashboard.DataGateway` | data gateway |
| `Dashboard.Web` | Blazor web site |
| `Dashboard.Database` | SQL project (`.sqlproj`, DACPAC) |
| Python project | `.pyproj` (Visual Studio) next to `pyproject.toml` (packaging, dependencies, tools) |
| `test/` | one test project per `Domain`, `Application` and `Infrastructure` project and per host, plus the architecture test |

ADR-0007 later adds the host project `Dashboard.Enrolment` (Enrolment service), under the rules of this ADR.

The solution is a `.slnx`. The .NET CI builds through a solution filter (`.slnf`) that excludes the SQL and Python projects, which have their own CI jobs.

### Failure visibility

- Every derived indicator (host state, anomaly score, CVE exposure, report) carries a **freshness timestamp** and an explicit **"unknown" state**. Missing data is never shown as the absence of a problem.
- Host state is computed with the **Backend's receive time**, not with the agent's clock.
- The agent's **local buffer is bounded** (sizes set in configuration). When it is full, the **oldest low-severity entries are dropped first**, and on reconnection the agent sends a record "N entries dropped between T1 and T2", shown in the dashboard.
- A log line that the parser cannot understand is **kept raw and tagged**, never dropped silently.

## Alternatives considered

- **CQRS.** Rejected: separate read and write models are not justified by the load or the domain.
- **Clean architecture applied to every component.** Rejected: empty layers in pure adapters add code without protecting any logic.
- **One project per core, layers as folders.** Rejected by the product owner: separate projects make the layer boundaries visible in the solution, and the compiler enforces the direction of references.
- **Unbounded agent buffer.** Rejected: a long outage would fill the disk of the supervised machine.

## Consequences

### Positive

- The domain logic is testable without network, database or host.
- The layer boundaries are enforced by project references and by a test.
- Data loss on the agent side is bounded, chosen and visible.
- An operator can always tell "nothing found" from "could not check".

### Negative

- About twice as many projects as with layers in folders, each with its references to maintain.
- Two styles coexist (hexagonal cores, plain adapters); the reason must be documented for new contributors.
- Every indicator and every screen must handle the "unknown" state.
- Whether `dotnet build` refuses a `.pyproj` like it refuses a `.sqlproj` is **not verified**; to confirm at the first build.
- A `.slnf` filter over a `.slnx` solution is **not verified** as a documented combination; to confirm at the first build.

## References

- Microsoft Learn, *Common web application architectures*, section "Clean architecture" (consulted during scoping, 2026-11-03).
- ADR-0003, ADR-0004, ADR-0005, ADR-0011.
