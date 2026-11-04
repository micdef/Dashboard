# ADR-0006: Hexagonal cores where there is domain logic, no CQRS

| Field | Value |
|---|---|
| Status | Proposed |
| Date | 2026-11-03 |
| Decider | Michaël Defraene (product owner, architect) |
| Drafted by | Claude (main developer); the decisions had a second reading by the architecture review agent during scoping |

## Context

The product owner first considered clean architecture (ports and adapters) applied to every component. An architect consulted earlier criticised a CQRS proposal as over-engineered for this size. Microsoft documents Clean, Hexagonal (ports and adapters) and Onion architectures as names for the same family. Applying the pattern to every component would add layers to components that only translate data.

## Decision

- **No CQRS.**
- A real **hexagonal core** (ports and adapters) exists only where there is real domain logic:
  - `Fleet.Core` (Backend): host state derived from heartbeats (time through `TimeProvider`), command lifecycle, deterministic rules, permissions per role.
  - `Agent.Core` (agent): XML log parsing engine (a pure function), local buffer with resume, command allowlist; Linux and Windows adapters around it.
- The data gateway and the web site have **no hexagon**: the data gateway is a pure adapter (ADR-0003) and the web site is a user interface over the Backend.
- A .NET project exists only if it has **different dependencies or a different deployment**; layers inside a project are folders.
- An **architecture test** fails if a `*.Core` project references gRPC, `Microsoft.Data.SqlClient` or ASP.NET Core.

Proposed solution structure:

| Path / project | Content |
|---|---|
| `src/contracts/protos` | `.proto` files (ADR-0004) |
| `Dashboard.Contracts.Edge`, `Dashboard.Contracts.Internal` | generated C# contracts |
| `Dashboard.Agent.Core`, `Dashboard.Agent` | agent core and host |
| `Dashboard.Fleet.Core`, `Dashboard.Backend` | domain core and Backend host |
| `Dashboard.DataGateway` | data gateway |
| `Dashboard.Web` | Blazor web site |
| `Dashboard.Database` | SQL project (`.sqlproj`, DACPAC) |
| Python project | `.pyproj` (Visual Studio) next to `pyproject.toml` (packaging, dependencies, tools) |
| `test/` | one test project per core and per host |

The solution is a `.slnx`. The .NET CI builds through a solution filter (`.slnf`) that excludes the SQL and Python projects, which have their own CI jobs.

## Alternatives considered

- **CQRS.** Rejected: separate read and write models are not justified by the load or the domain.
- **Clean architecture applied to every component.** Rejected: empty layers in pure adapters add code without protecting any logic.
- **One project per layer.** Rejected: projects without their own dependencies or deployment only add references to maintain; the architecture test protects the boundaries instead.

## Consequences

### Positive

- The domain logic is testable without network, database or host.
- The boundaries are enforced by a test, not only by convention.

### Negative

- Two styles coexist (hexagonal cores, plain adapters); the reason must be documented for new contributors.
- Whether `dotnet build` refuses a `.pyproj` like it refuses a `.sqlproj` is **not verified**; to confirm at the first build.
- A `.slnf` filter over a `.slnx` solution is **not verified** as a documented combination; to confirm at the first build.

## Open points

1. Content of `Fleet.Core` and `Agent.Core`.
2. The project rule (a project only for different dependencies or deployment) and the architecture test.
3. The solution structure above.
4. Raised in review, proposed: every derived indicator (host state, anomaly score, CVE exposure, report) carries a freshness timestamp and an explicit "unknown" state, so that missing data is never shown as the absence of a problem. Host state uses the Backend's receive time, not the agent's clock.
5. Raised in review, proposed: the agent's local buffer is bounded; when it is full, the oldest low-severity entries are dropped first, and a "dropped N entries between T1 and T2" record is sent on reconnection. A line that fails to parse is kept raw and tagged, never dropped silently.

## References

- Microsoft Learn, *Common web application architectures*, section "Clean architecture" (consulted during scoping, 2026-11-03).
- ADR-0003, ADR-0004.
