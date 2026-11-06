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

When a machine is revoked, the API notifies the Collector, which closes the open streams of that machine, and the Enrolment service adds the serial to its next revocation list (ADR-0017, kept by ADR-0027). In 0.1 there is no stream: commands arrive in 0.6. The Agent CA revocation list is valid 7 days and regenerated daily (ADR-0021); the Enrolment service is a client of the API, which exposes to it a "revoked serials" service (ADR-0017); there is no link from the API to the Enrolment service. The Collector reads the Agent CA revocation list from the local file written by the Enrolment service, never through HTTP (ADR-0025), but no ADR says how.

The identity resolution and the propagation of revocations are recorded together because the second relies on the first: the list may be late only because the mapping refuses at once.

## Decision

### Identity resolution

- The Collector resolves the hash of the client certificate to a `HostId` **on every call**, not once per connection, by a read-only procedure of the data gateway on its `ingest` socket (ADR-0004, ADR-0005). Only the API writes the mapping.
- The Collector keeps **no cache** of the mapping. The certificate validation cache of ASP.NET Core (`AddCertificateCache`) is **never enabled**: its expiry is sliding, extended each time the certificate is used (2 minutes by default), so a machine that calls more often would stay accepted after its revocation; and it keeps failed results for the configured entry lifetime too (Microsoft Learn, `CertificateValidationCacheOptions.CacheEntryExpiration` and *Configure certificate authentication in ASP.NET Core*, aspnetcore-10.0). Adding a cache later needs an ADR, because it would delay a revocation.
- The procedure that stores the heartbeat checks the mapping and the host state again, as every layer checks what it receives.
- A hash that is not mapped, a revoked mapping, or a host marked suspect (ADR-0030) is refused: the host is "down: refused", with the reason (ADR-0021, ADR-0028). The refusals are recorded with the hash of the certificate (ADR-0017), aggregated and rate-limited, so that a flood of calls cannot fill the records; the records dropped by the limit are counted.
- When the data gateway cannot be reached, the call fails with a retryable error, and the dashboard shows the fleet as **"unknown"** (ADR-0022), never as "down: refused".

### Open streams (from 0.6)

- The notification that closes the open streams of a revoked machine (ADR-0017) is built with the streams, in 0.6. This is not a weakening: no ADR defines a stream before 0.6, and the next call of a revoked machine is refused at once by the resolution above. If an earlier increment adds a stream, the notification and the check below come with it.
- From the first stream on, the Collector also checks again the mapping of every open stream, at an interval set in the configuration, so that a lost notification does not leave a revoked stream open. This **tightens** the rule of ADR-0017 kept by ADR-0027, which relies on the notification alone, as the commands already rely on a periodic check for lost notifications (ADR-0005).

### Agent CA revocation list

- The Enrolment service regenerates the list **daily** (ADR-0021). At each regeneration, it reads the revoked serials from the API, through the "revoked serials" service of its socket (ADR-0017). No new link is added.
- An error of the API is never read as an empty list. If the API cannot be reached, or answers an error, the Enrolment service keeps the previous list while it is valid, and the failure is a critical state (ADR-0032): logged, shown by the host status command in 0.1, alerted from 0.2 (ADR-0025). The 7-day validity gives about 6 days to repair it (ADR-0017).
- The Enrolment service never publishes a list with fewer serials than the previous one, except the serials of certificates that have expired; it refuses to publish such a list, logs it as a critical state and keeps the previous one. The CRL number of each new list is higher than the previous one.
- The list file is written atomically: a temporary file, then a rename.
- **Delay**: a revoked machine is refused at once by the mapping. A party that checks the list itself may accept the revoked certificate until it loads a newer list: up to about 7 days for one that keeps the previous list until its `nextUpdate`.
- **The Collector** loads the list from the local file (ADR-0025). It checks the signature against the Agent CA anchor set, the `nextUpdate`, and that the CRL number is not lower than the highest one it has loaded, which it keeps across restarts. It loads the file again when the file changes:
  - a new file that fails these checks is refused and logged; the Collector keeps the last valid list until its `nextUpdate`, and the refusal is a critical state at once (ADR-0032), with the time left on the list it keeps;
  - with no valid list at all (missing, expired or invalid), it refuses every agent certificate (strict revocation, ADR-0017). This is a fault of the Collector, not of the hosts: it is a critical state (ADR-0032), and the dashboard shows the fleet as "unknown", not as "down: refused";
  - if the highest CRL number it keeps cannot be read at start-up, it loads no list, which is the case above. The recovery is a written procedure in `doc/adm/`. There is no override for a list with a lower number in V1.
- The Collector records its critical states through its `ingest` socket, with a procedure of its own; the mapping stays read-only for it.

## Alternatives considered

- **Resolution once per connection.** Rejected: an HTTP/2 connection stays open, and how to close a Kestrel connection from outside a request is not documented.
- **A cache of the mapping with a lifetime set in the configuration.** Rejected: a revocation would wait for the cache to expire, which weakens the immediate refusal of ADR-0017; that would need an ADR that supersedes it.
- **The Enrolment service also reading the revoked serials at a short interval, and regenerating the list when they change.** Not adopted: on Linux, .NET keeps a revocation list until its `nextUpdate` and does not download it again before (read in the .NET source on its main branch; the 10.0 branch and Windows are **not verified**), so the agents would not see a newer list; the immediate refusal already comes from the mapping. The product owner first chose this option, and changed his decision after the second reading.
- **A notification link from the API to the Enrolment service.** Rejected: one more link, and it reverses the dependency of ADR-0016.
- **A shorter list validity** (for instance one day). Not adopted: the tolerance to an outage of the Enrolment service would drop from about 6 days to less than one, for a gain that the mapping already gives.

## Consequences

### Positive

- A revoked or suspect machine is refused at its next call, with no state kept in memory.
- An outage of the data gateway, or an unusable revocation list, never looks like a refused identity.
- The revocation list keeps the tolerance of ADR-0017 to an outage of the Enrolment service, with stated failure rules.

### Negative

- One read through the data gateway on every call of every agent.
- A party that checks only the revocation list sees a revocation up to about 7 days late; the code must never rely on the list alone where the mapping can be read.
- How the Collector, and the Enrolment service at renewal, check an agent certificate against the loaded list (their own check of the serial, or the .NET chain building from a local file) is **not verified**: a test decides it when the 0.1 work starts, with a revoked certificate that must fail and an expired list that must fail.

## References

- Microsoft Learn: `CertificateValidationCacheOptions.CacheEntryExpiration`, *Configure certificate authentication in ASP.NET Core*, "Cache certificate validation results" (aspnetcore-10.0), consulted on 2026-11-06.
- ADR-0004, ADR-0005, ADR-0016, ADR-0017 (superseded), ADR-0021 (superseded), ADR-0022, ADR-0025, ADR-0027, ADR-0028, ADR-0030, ADR-0032.
- [Review report of the 0.1 identity sequences](../reviews/2026-11-06-identity-sequences-of-0.1.md).
