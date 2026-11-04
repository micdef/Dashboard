# Future improvements

Possible improvements identified during design and deliberately left out of V1. They are not commitments. When one is taken up, it gets its own ADR.

| Improvement | Why | Starting point |
|---|---|---|
| Port the database to PostgreSQL or Oracle | Run Dashboard on another database engine (V2 or later). | See below. |

## Port the database to PostgreSQL or Oracle

Recorded on 2026-11-04, during the decision on storage (ADR-0012).

- **What changes.** The stored procedures, views and table-valued parameters are T-SQL; a port rewrites them (PL/pgSQL for PostgreSQL, PL/SQL for Oracle). The contract seen by the VBE (procedure names, parameters, one JSON result per input row) stays the same.
- **Inventory history.** V1 uses SQL Server temporal tables. PostgreSQL 18 does not support system-versioned tables (SQL feature T180 is listed as unsupported in its documentation). A port rebuilds the history with an `AFTER UPDATE OR DELETE` trigger writing to a history table with the same `ValidFrom` and `ValidTo` columns, and the "state as of a date" procedures are rewritten on that table. Oracle offers its own mechanism (Flashback Data Archive); its fit is **not verified**.
- **What makes it possible.** `FOR SYSTEM_TIME` is used only inside procedures and views (ADR-0012), and the application only goes through the VBE (ADR-0003); no application code depends on the history mechanism.
- **Audit trail.** V1 uses a hash chain computed by the stored procedures, with anchors stored outside the database (ADR-0013). It relies on SHA-256 hashing and partitions only; a port rewrites the procedures and must keep the exact bytes hashed, so that the existing chain and archives stay verifiable.
- **Other points to check at that time.** Partitioning and partition truncation, the JSON functions used by the procedures, table-valued parameters (no direct equivalent in PostgreSQL), and the data access library of the VBE.
