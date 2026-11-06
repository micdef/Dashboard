# ADR-0029: Identity resolution on the agent channel, and how revocations propagate

| Field | Value |
|---|---|
| Status | Accepted |
| Date | 2026-11-06 |
| Decider | Michaël Defraene (product owner, architect) |
| Drafted by | Claude (main developer), from the sequence diagrams of 0.1; second reading by the architecture review agent before the decision |
| Classification | Security-sensitive [SEC] |
| Complements | ADR-0005, ADR-0025, ADR-0027 |

## Context

The machine is identified by the SHA-256 hash of its client certificate, mapped to a `HostId` by the API, and the mapping fails closed (ADR-0027). The ADRs do not say **when** the Collector reads that mapping, nor whether it may keep it in memory. With a unary heartbeat (ADR-0028), the agent reuses an HTTP/2 connection that stays open, so a check done only when the connection opens would miss a revocation until the connection closes.

When a machine is revoked, the API notifies the Collector, which closes the open streams of that machine, and the Enrolment service adds the serial to its next revocation list (ADR-0017, kept by ADR-0027). In 0.1 there is no stream: commands arrive in 0.6. The Agent CA list is valid 7 days and regenerated daily (ADR-0021); the Enrolment service is a client of the API, which exposes it a "revoked serials" service (ADR-0017); there is no link from the API to the Enrolment service. The Collector reads the Agent CA list from the local file written by the Enrolment service, never through HTTP (ADR-0025), but no ADR says how.

## Decision

### Identity resolution

- The Collector resolves the hash of the client certificate to a `HostId` **on every call**, not once per connection, by a read-only procedure of the data gateway on its `ingest` socket (ADR-0004, ADR-0005). Only the API writes the mapping.
- The Collector keeps **no cache** of the mapping. The certificate validation cache of ASP.NET Core (`AddCertificateCache`) is **never enabled**: its expiry is sliding, extended each time the certificate is used (2 minutes by default), so a machine that calls more often would stay accepted after its revocation, and it also keeps failed results for its lifetime (Microsoft Learn, `CertificateValidationCacheOptions.CacheEntryExpiration` and *Configure certificate authentication in ASP.NET Core*, aspnetcore-10.0). Adding a cache later needs an ADR, because it would delay a revocation.
- The procedure that stores the heartbeat checks the mapping again, as every layer checks what it receives.
- A hash that is not mapped, or a revoked mapping, is refused and recorded with its hash (ADR-0017): the host is "down: refused" (ADR-0021).
- When the data gateway cannot be reached, the call fails with a retryable error, and the dashboard shows the fleet as **"unknown"** (ADR-0022), never as "down: refused".

### Open streams (from 0.6)

- The notification that closes the open streams of a revoked machine (ADR-0017) is built with the streams, in 0.6; in 0.1 there is no stream to close, and the next call of a revoked machine is refused by the resolution above.
- From 0.6, the Collector also checks again the mapping of every open stream, at an interval set in the configuration, so that a lost notification does not leave a revoked stream open. This tightens the rule of ADR-0017 kept by ADR-0027, which relies on the notification alone, as the commands already rely on a periodic check for lost notifications (ADR-0005).

### Agent CA revocation list

- The Enrolment service regenerates the list **daily** (ADR-0021). At each regeneration, it reads the revoked serials from the API, through the "revoked serials" service of its socket (ADR-0017). No new link is added.
- If the API cannot be reached, the Enrolment service keeps the previous list while it is valid, and the failure is visible: logged, and alerted from 0.2, when the alert channel exists (ADR-0025). The 7-day validity gives about 6 days to repair it (ADR-0017).
- A new list never holds fewer serials than the previous one, except the serials of certificates that have expired. Its CRL number always increases.
- **Delay**: a revoked machine is refused at once by the mapping. A party that checks the list itself may accept the revoked certificate until it loads a newer list: up to about 7 days for one that keeps the previous list until its `nextUpdate`.
- **The Collector** loads the list from the local file (ADR-0025): it checks the signature against the Agent CA anchor set, the `nextUpdate`, and that the CRL number is not lower than the one already loaded; it loads the file again when it changes. A missing, expired or invalid list makes it refuse every agent certificate (strict revocation, ADR-0017).

## Alternatives considered

- **Resolution once per connection.** Rejected: an HTTP/2 connection stays open, and how to close a Kestrel connection from outside a request is not documented.
- **A cache of the mapping with a lifetime set in the configuration.** Rejected: a revocation would wait for the cache to expire, which weakens the immediate refusal of ADR-0017; that would need an ADR that supersedes it.
- **The Enrolment service also reading the revoked serials at a short interval, and regenerating the list when they change.** Not adopted: on Linux, .NET keeps a revocation list until its `nextUpdate` and does not download it again before (read in the .NET source on its main branch; the 10.0 branch and Windows are **not verified**), so the agents would not see a newer list; the immediate refusal already comes from the mapping. The product owner first chose this option, and changed his decision after the second reading.
- **A notification link from the API to the Enrolment service.** Rejected: one more link, and it reverses the dependency of ADR-0016.
- **A shorter list validity** (for instance one day). Not adopted: the tolerance to an outage of the Enrolment service would drop from about 6 days to less than one, for a gain that the mapping already gives.

## Consequences

### Positive

- A revoked machine is refused at its next call, with no state kept in memory.
- An outage of the data gateway never looks like a refused identity.
- The revocation list keeps the tolerance of ADR-0017 to an outage of the Enrolment service, with stated failure rules.

### Negative

- One read through the data gateway on every call of every agent.
- A party that checks only the revocation list sees a revocation up to about 7 days late; the code must never rely on the list alone where the mapping can be read.
- How the Collector, and the Enrolment service at renewal, check an agent certificate against the loaded list (their own check of the serial, or the .NET chain building from a local file) is **not verified**: a test decides it when the 0.1 work starts, with a revoked certificate that must fail and an expired list that must fail.

## References

- Microsoft Learn: `CertificateValidationCacheOptions.CacheEntryExpiration`, *Configure certificate authentication in ASP.NET Core*, "Cache certificate validation results" (aspnetcore-10.0), consulted on 2026-11-06.
- ADR-0004, ADR-0005, ADR-0016, ADR-0017, ADR-0021, ADR-0022, ADR-0025, ADR-0027, ADR-0028.
- The sequence diagrams of the machine identity of 0.1 (`doc/analyse/uml/v0.1/identity.md`).
