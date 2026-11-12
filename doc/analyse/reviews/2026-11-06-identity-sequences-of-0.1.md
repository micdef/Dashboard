# Review of 2026-11-06: identity sequences of 0.1, and ADR-0028 to ADR-0032

Readings of the first sequence diagrams of increment 0.1, on the machine identity (`doc/analyse/uml/v0.1/identity.md`, added by the next pull request of the same chain), and of the ADRs that came out of them. Every finding was checked by the main developer before it was acted on; the decisions were taken by the product owner.

## 1. Second reading of three flow choices (Yoda, architecture)

Drawing the sequences found three points that no ADR settled: how the heartbeat travels, when the Collector resolves the identity of an agent, and how the Agent CA revocation list learns of a revocation. The product owner first chose, then the architecture review agent read the three points without seeing that choice.

- **Heartbeat**: a unary call, separate from the command stream. Agreed with the first choice.
- **Identity resolution**: on every call, not once per connection, with the ASP.NET Core certificate cache forbidden; an unreachable data gateway shown as "unknown". This refined the first choice ("at each new connection"), and the product owner adopted it.
- **Revocation list**: the agent recommended a daily regeneration only, with failure rules, against the first choice (daily plus a short polling interval): on Linux, .NET keeps a revocation list until its `nextUpdate` (read in the .NET source on its main branch, **not verified** on 10.0), so a list regenerated sooner would not reach the agents, and the mapping already refuses at once. The product owner first followed it, then, on 2026-11-12, kept the short polling interval after all: the Collector, which reloads the local file, sees a revocation through the list within minutes. The limit for the agents is written in ADR-0029.

Result: ADR-0028 (heartbeat) and ADR-0029 (identity resolution and revocation propagation).

## 2. Team review of the first draft of the identity sequences

| Agent | Blocking | Important | Minor |
|---|---|---|---|
| Ackbar | 3 | 8 | 7 |
| Boba Fett | 2 | 5 | 5 |
| Jocasta Nu | 0 | 5 | 5 |
| Cassian | 0 | 2 | 3 |
| C-3PO | 0 | 2 | 6 |

### Blocking findings

| Id | Finding | Outcome |
|---|---|---|
| A-B1 | At renewal, the API checked the issued certificate against a CSR supplied by the Enrolment service itself: a compromised Enrolment service could register its own key for every machine. | Issuance ticket bound to the key that the API reads itself (ADR-0030). |
| A-B2 | Two concurrent renewals from one stolen key could both succeed. | Atomic reservation of the successor under a unique constraint (ADR-0030). |
| A-B3 | Several live tokens per host, and a consumption that did not check the host again. | One live token per host, atomic creation and consumption that check the host state again (ADR-0031). |
| B-B1 | An honest retry after a lost renewal answer was taken for a second renewal. | Idempotent retry with the same key (ADR-0030). |
| B-B2 | A host whose token was consumed without a certificate could get no new token. | "New `HostId`" defined as never mapped, shown as pending (ADR-0031). |

### Important findings, in short

- **Token file** (Ackbar, Jocasta Nu, C-3PO, Boba Fett): deleted before the checks, readable by root while the agent runs as its own user, open to a symbolic link. One-shot enrolment command run as root, standard input first, file opened safely and deleted once the token is sent (ADR-0031).
- **Records** (Jocasta Nu, Ackbar): token creation and refused tokens recorded nowhere, which ADR-0017 needs to name who created a token. Recorded from 0.1 (ADR-0031).
- **Yearly identity** (Jocasta Nu): the 1-year limit forces a revocation and a critical alert per host per year. Kept, with the date shown in advance and a written procedure; a way out recorded in the future improvements (ADR-0031).
- **Enrolment service server certificate** (Jocasta Nu, Ackbar): its registration through the API would add a service to a socket that ADR-0017 limits, and the reason given was wrong. The API reads its public file instead (ADR-0030).
- **Hardening** (Ackbar, Cassian): refused certificates revoked, exact certificate profile, CSR bounded before it is loaded, RSA 3072 or 4096 only (ADR-0030).
- **Suspect host** (Ackbar): no defined effect. Refused at renewal and at the Collector until revoked and enrolled again (ADR-0030).
- **Failure paths** (Boba Fett): no branch for an unreachable callee, a deadline, a crash between two steps, a renewal that keeps failing. A rule for all diagrams, and the missing branches (identity sequences).

### Second reading before the decisions (Yoda)

The architecture review agent read the eight resulting points without the main developer's recommendations. It rejected the keyed hash of the CSR (the API would need the secret part of the token), recommended a ticket at renewal, the whole token package, keeping the 1-year limit, a one-shot root command, recording token creation, the hardening, refusing a suspect host everywhere, and reading the Enrolment certificate file; and two complementary ADRs. The product owner decided the eight points as recommended. Result: ADR-0030 and ADR-0031.

## 3. Team review of ADR-0028 to ADR-0031

| Agent | Blocking | Important | Minor |
|---|---|---|---|
| C-3PO | 1 | 5 | 7 |
| Boba Fett | 2 | 9 | 6 |
| Ackbar | 0 | 3 | 7 |
| Jocasta Nu | 0 | 5 | 8 |
| Cassian | 0 | 0 | 5 |

Ackbar confirmed that his three blocking findings of section 2 are closed, and stated the residual risk of the first enrolment as honestly written.

### Blocking findings

| Id | Finding | Outcome |
|---|---|---|
| C-B1 | The ADRs cited the sequence diagrams, which are not in their pull request. | The ADRs cite this report; the diagrams follow in the next pull request of the chain. |
| B-B1 | An expired ticket blocked a renewal, and a lost key turned an honest host suspect without saying so. | A retry with the same key gets a new ticket and a fresh nonce; a lost key stays a second renewal, and the restore of a snapshot is stated as a cause (ADR-0030). |
| B-B2 | "Every certificate signed is registered or revoked" was false. | Bounded retries of the registration, a critical state for a ticket that expires with a signed certificate, refusal and revocation in one operation; the claim restricted to the certificates submitted to the API (ADR-0030). |

### Decisions of the product owner

- **Token storage** (Jocasta Nu): the selector stored in clear departed from "only its SHA-256 hash is stored", which a complement cannot do. Back to the hash of the whole token, looked up by index and compared with `FixedTimeEquals` (ADR-0031).
- **Pending state** (Jocasta Nu): presented as the definition of "new `HostId`" (never mapped), which ADR-0025 and ADR-0026 do not define, not as an extension (ADR-0031). An abandoned pending host is revoked, then retired, because ADR-0026 allows retiring only from the revoked state.
- **Tickets** (Ackbar, Jocasta Nu, Boba Fett): the registration checks the host state again, at most one open ticket per host, open tickets invalidated by a new token, a revocation or a suspect state (ADR-0030, ADR-0031).
- **Critical states in 0.1** (Boba Fett): logged at critical level and listed by a host command "status" that fails when one is active (ADR-0032).

### Other findings, fixed

- The number of rules tightened, defined and added stated exactly (ADR-0030); the suspect state added to the identity resolution and the host states (ADR-0028, ADR-0029); the deferral of the stream notification argued as not a weakening (ADR-0029).
- The Collector keeps the last valid revocation list when a new file is invalid, keeps the highest CRL number across restarts, and shows an unusable list as a fault of the Collector; the list file and the certificate file are written atomically; an API error is never an empty list (ADR-0029, ADR-0030).
- Refusal records aggregated and rate-limited, with the dropped ones counted; the creation of a token recorded inside its atomic operation; a host account that cannot be read marked in the record; one generic refusal to the machine; a token file that cannot be deleted stops the command; the token never in an argument or an environment variable (ADR-0029, ADR-0031).
- A unique serial in the certificate profile, and the list of its extensions; a bound on the size of the certificate file; the HTTP/2 limits of Kestrel set in the configuration (ADR-0028, ADR-0030).
- Wording, terms and references (ADR-0013 named as superseded; the "unknown" state; the heartbeat is not a batch).

### Deferred

- A row for the gRPC packages in `doc/analyse/dependencies.md` (Cassian): no package is chosen yet; added with the first code.
- The lifetime of the stored issued certificates is bounded by their own expiry (ADR-0030); no other retention rule.

## 4. Second reading of the fixes

The fixes of section 3 are read again by the agents that found them, before the pull request is handed to the product owner. The results are in the pull request.
