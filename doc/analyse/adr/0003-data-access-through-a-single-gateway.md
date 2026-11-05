# ADR-0003: Data access through a single gateway, ADO.NET only

| Field | Value |
|---|---|
| Status | Accepted |
| Date | 2026-11-03 (proposed), 2026-11-04 (accepted) |
| Decider | Michaël Defraene (product owner, architect) |
| Drafted by | Claude (main developer); the decisions had a second reading by the architecture review agent during scoping |

## Context

The product owner's SQL conventions follow a "contract as a service" approach: the application never touches tables directly; it calls stored procedures, passes bulk data as table-valued parameters (TVP) and reads views. Each layer validates what it receives. The database must be reachable by a single SQL login, and the SQL code must stay portable to PostgreSQL.

## Decision

- The **data gateway** is the **only process** that connects to SQL Server, under **one SQL login**.
- The data gateway uses **ADO.NET only** (`Microsoft.Data.SqlClient`). **No ORM**: neither Entity Framework nor Dapper nor any other mapper.
- Every access goes through **stored procedures, TVPs and views**. Procedures return **one JSON result for each input row**.
- The business rules of the data (integrity, validation, audit) live in the stored procedures. The data gateway is a pure adapter: it translates gRPC calls (ADR-0004) into procedure calls and has no domain logic of its own.
- Operations that need more than `EXECUTE` (for example truncating a partition, ADR-0012) run inside stored procedures with `EXECUTE AS`, never through extra rights granted to the login.
- The SQL project keeps its original format (`.sqlproj`, DACPAC). It is never converted to the SDK format (`Microsoft.Build.Sql`).

## Alternatives considered

- **Entity Framework Core.** Rejected: it encourages direct table access and generated SQL, which contradicts the procedure-only contract, and it adds a dependency without benefit when every call is a procedure.
- **Dapper.** Rejected for a similar reason, to a lesser degree: the mapping it saves is small compared with the explicit, reviewed ADO.NET code.
- **Each service connecting to the database.** Rejected: one login and one process keep the attack surface and the permissions minimal.

## Consequences

### Positive

- The SQL login can be restricted to `EXECUTE` on procedures and `SELECT` on views (plus what TVPs need on their table types, to be checked when the first one is written).
- The database contract is explicit and testable on its own.
- Swapping SQL Server for PostgreSQL later only affects the SQL project and the data gateway.

### Negative

- More hand-written code (parameter mapping, JSON parsing) than with an ORM; it must be covered by tests.
- The data gateway is a single point of failure for all data access. How callers behave when it is unreachable is defined in ADR-0004.

## References

- ADR-0004, ADR-0012, ADR-0016.
