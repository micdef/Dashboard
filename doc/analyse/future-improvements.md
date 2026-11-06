# Future improvements

Possible improvements identified during design and deliberately left out of V1. They are not commitments. When one is taken up, it gets its own ADR.

| Improvement | Why | Starting point |
|---|---|---|
| Port the database to PostgreSQL or Oracle | Run Dashboard on another database engine (V2 or later). | See below. |
| Asymmetric signature of the audit exports, and key rotation | The verifier holds a copy of the HMAC key, so a compromised verifier machine could forge rows it would itself accept (ADR-0024). | See below. |
| Yearly re-enrolment without a revocation | The identity lifetime of 1 year (ADR-0017, kept by ADR-0027) forces a revocation and a new token for every host once a year, with a critical alert (ADR-0031). Acceptable for a small fleet, not for a large one. | Remove the limit, or add an exception for identity expiry: both need an ADR that supersedes ADR-0026 or ADR-0027. |
| Single-use step-up proof for every sensitive action | V1 makes the proof single use for commands and account changes only; the other actions keep a 5-minute proof (ADR-0023). | Extend the single-use rule when the list of actions is final. |

## Port the database to PostgreSQL or Oracle

Recorded on 2026-11-04, during the decision on storage (ADR-0012).

- **What changes.** The stored procedures, views and table-valued parameters are T-SQL; a port rewrites them (PL/pgSQL for PostgreSQL, PL/SQL for Oracle). The contract seen by the VBE (procedure names, parameters, one JSON result per input row) stays the same.
- **Inventory history.** V1 uses SQL Server temporal tables. PostgreSQL 18 does not support system-versioned tables (SQL feature T180 is listed as unsupported in its documentation). A port rebuilds the history with an `AFTER UPDATE OR DELETE` trigger writing to a history table with the same `ValidFrom` and `ValidTo` columns, and the "state as of a date" procedures are rewritten on that table. Oracle offers its own mechanism (Flashback Data Archive); its fit is **not verified**.
- **What makes it possible.** `FOR SYSTEM_TIME` is used only inside procedures and views (ADR-0012), and the application only goes through the VBE (ADR-0003); no application code depends on the history mechanism.
- **Audit trail.** V1 uses an HMAC-SHA256 chain whose key is outside the database, computed by the VBE and exported every hour, the anchor being the independent verifier's copy (ADR-0024). It relies on partitions and on the canonical byte format of the rows; a port rewrites the procedures and must keep that byte format, so that the existing chain and archives stay verifiable.
- **Other points to check at that time.** Partitioning and partition truncation, the JSON functions used by the procedures, table-valued parameters (no direct equivalent in PostgreSQL), and the data access library of the VBE.

## Asymmetric signature of the audit exports

Recorded on 2026-11-05, during the third reading (finding T-I3).

- **Idea.** The VBE (or a signer on the "app" VM) signs each exported audit file with a private key (Ed25519 or ECDSA P-256); the verifier holds only the public key and can no longer forge anything.
- **Rotation.** Each audit row already carries a key identifier (ADR-0024). A rotation writes a switch row chained under the old key and the new one, and the verifier accepts each key only for its range of sequence numbers.
- **Cost.** One more key pair to keep offline, a rotation procedure to write and test, and the verifier's code to change.
