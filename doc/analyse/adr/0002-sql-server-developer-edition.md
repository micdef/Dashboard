# ADR-0002: SQL Server 2025 Standard Developer edition

| Field | Value |
|---|---|
| Status | Accepted |
| Date | 2026-11-03 (proposed), 2026-11-04 (accepted) |
| Decider | Michaël Defraene (product owner, architect) |
| Drafted by | Claude (main developer); the decisions had a second reading by the architecture review agent during scoping |

## Context

The database is an SSDT project (`.sqlproj`, DACPAC format) targeting SQL Server 2025. It must stay portable to PostgreSQL later. The storage design (ADR-0012) relies on table partitioning, temporal tables and possibly columnstore indexes.

The project is a training project and a portfolio piece. It runs in the product owner's home lab, which is a learning and continuous development environment, and it is shown in demonstrations. The repository is public: other people may clone it and run it on their own SQL Server.

SQL Server 2025 introduced **two** free developer editions:

- **Enterprise Developer**: all Enterprise features; functionally equivalent to the Developer edition of earlier versions.
- **Standard Developer**: all Standard features.

Microsoft describes both as licensed for use as a development and test system, not as a production server.

## Decision

- The project runs on **SQL Server 2025 Standard Developer edition**, in its own Hyper-V VM (ADR-0001).
- The product owner considers the home lab a learning and continuous development environment; the Developer licence is kept for it.
- The **Standard edition** (and therefore Standard Developer, which has the same features) is the **reference edition of the repository**: the database code uses only features available in Standard, so anyone who clones the project can run it on Standard or Standard Developer.
- Enterprise-only features are not used. Express is not a supported target.

## Alternatives considered

- **Enterprise Developer.** Rejected: nothing would prevent the use of an Enterprise-only feature (for example star join optimisations, or parallel query processing on partitioned tables), and the code would no longer run on Standard.
- **Express edition.** Rejected: limited in SQL Server 2025 to 50 GB per database, a buffer pool of 1,410 MB, a columnstore segment cache of 352 MB, the lesser of 1 socket or 4 cores, and no SQL Server Agent. The log volume is unknown (ADR-0012), and scheduled jobs would have to run elsewhere.
- **PostgreSQL from the start.** Rejected for V1: the SQL conventions and tooling of the product owner (SSDT, DACPAC) are SQL Server based. Portability is kept as a constraint instead.

## Consequences

### Positive

- All the features needed by the storage design are available: partitioning, data compression, columnstore and temporal tables are listed for every edition of SQL Server 2025.
- SQL Server Agent is available for scheduled maintenance jobs.
- Measurements made in the home lab (ADR-0012) reflect what a Standard deployment would get.
- A person who clones the repository knows which edition the code is written for.

### Negative

- Standard limits that bear on ADR-0012: the degree of parallelism of batch-mode operations is limited to 2, parallel query processing on partitioned tables and indexes is Enterprise only, and online rebuild of nonclustered columnstore indexes is Enterprise only.
- Licensing remains the responsibility of whoever deploys the project: the Developer editions are licensed for development and test only, and a production deployment needs a paid Standard licence. This is stated in the installation documentation. This ADR is not legal advice; the Microsoft licensing terms prevail.

## References

- Microsoft Learn, *What's new in SQL Server 2025*, section "New developer editions" (consulted 2026-11-03).
- Microsoft Learn, *Editions and supported features of SQL Server 2025*, sections "SQL Server editions", "Scale limits", "Scalability and performance", "Data warehouse" and "Management tools" (consulted 2026-11-03).
- ADR-0001, ADR-0012.
