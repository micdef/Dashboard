# ADR-0021: Machine identity from the first increment, with a dedicated revocation list service

| Field | Value |
|---|---|
| Status | Superseded by [ADR-0027](0027-machine-identity-with-agent-ca-anchor-sets.md) |
| Date | 2026-11-05 |
| Decider | Michaël Defraene (product owner, architect) |
| Drafted by | Claude (main developer), after the second reading of 2026-11-04 |
| Classification | Security-sensitive [SEC] |
| Supersedes | ADR-0017 |

## Context

ADR-0017 defined machine identity with two independent certificate authorities (CA), enrolment by one-time token and CSR, and strict revocation. The second reading found that:

- the enrolment token was created in the web site, while ADR-0015 places enrolment in increment 0.1, before the web site's accounts and the audit trail (0.2) (finding I1);
- the component that serves the revocation lists was named in no topology (finding I2);
- a revoked Collector certificate stays trusted until the cached Server CA list expires, up to 12 months, according to the .NET source on Linux, so revocation on that channel was not "immediate" as ADR-0017 claimed (finding I3);
- the .NET floor 10.0.10 is outdated (finding I4), and the token could be passed on a command line (finding I8).

This ADR keeps every decision of ADR-0017 except where it says otherwise. It also completes the topology of ADR-0016 with one component.

## Decision

### Kept from ADR-0017

- The machine is identified by the SHA-256 hash of its authenticated client certificate, mapped to a `HostId` by the API; the mapping fails closed.
- **Two independent CAs**: the **Server CA** (self-signed, key offline with the product owner) signs only the Collector's server certificate; the **Agent CA** (self-signed, key online in the Enrolment service) signs agent certificates and the Enrolment service's server certificate. Each channel trusts exactly one CA, with distinct DNS names.
- The Enrolment service is a **signer only**: the API checks and consumes the token, verifies the renewal proof (signed with the current private key), and checks every issued certificate before registering it. Renewal rules, CSR policy, endpoint rate limiting, and the Unix socket reserved to the Enrolment service are unchanged.
- **Strict revocation on every channel**, with an unknown or offline status treated as a failure; the Agent CA list valid 7 days and regenerated daily; the Server CA list valid 12 months and re-signed at least every 6 months, alerted 60 days before its `nextUpdate`.
- Transport settings, validity periods, offline key custody, reconciliation, and the tests on Linux of undocumented behaviours.

### Enrolment token

- From **0.1**, a token is created with a **command on the API host**, like the first Admin (ADR-0023). The command stays available afterwards.
- From **0.2**, an Admin can also create a token in the web site, after step-up (ADR-0023).
- The agent accepts the token **only from a file readable by root only, or from standard input**, never as a command-line argument (which would be visible in the process list and in the shell history). It deletes the file after use.
- Every issuance and revocation is recorded in the **issuance register**, which also serves the reconciliation. In 0.1, before the audit trail exists, the register is the only record. When 0.2 is installed, the existing register entries are written as the first rows of the audit trail (ADR-0024), marked "recorded before the audit trail".

### Revocation list service

- A dedicated service, **`Dashboard.Crl`**, serves the revocation lists as static files over plain HTTP. Revocation lists are signed, so their integrity does not depend on the transport; plain HTTP is a scheme that RFC 5280 provides for distribution points (section 4.2.1.13; the wording of the RFC is **not verified**).
- It is a small Kestrel host: `GET` only, a read-only directory, its own user and a hardened systemd unit. It listens **only on the interface facing the supervised machines**, never on the internet-facing reverse proxy.
- The Enrolment service writes the Agent CA list into that directory; the product owner places the Server CA list there after signing it offline. `Dashboard.Crl` only reads.
- It does not depend on the Enrolment service, so the lists stay available when that service is down. It is a third endpoint for the agents, to be allowed in the firewall.

### Revocation of the Collector certificate: limit and procedure

- The agents cache the Server CA list until its `nextUpdate`. Revoking the Collector certificate therefore takes effect only when the cached list expires: **up to 12 months**. This is measured from the .NET source on Linux; the Windows behaviour is **not verified**.
- Revocation is not the response to a compromise of the Collector's key. The response is a written **compromise procedure**: create a new Server CA, issue a new Collector certificate, and deploy the new trust anchor to every agent with an agent update; agents that have not received it are reported on the dashboard.
- A theft of the Collector's key almost always means a compromise of the "app" VM, which is outside the guarantees of this ADR (see Consequences).

### Agent CA re-key

- The re-key of the Agent CA (every 2 years) includes an **overlap period**: an agent update first ships the new anchor next to the old one, then the Enrolment service switches to the new CA. A coverage report lists the agents that do not trust the new CA yet, before the old one is retired.

### Visibility of identity failures

- The host page distinguishes **"down: refused"** (the Collector refused its certificate, with the reason) from **"down: silent"** (no connection attempt).
- Each agent reports in its heartbeat the number of days before its certificate expires.

### Runtime and dependencies

- The .NET runtime follows the **latest patch of the 10.0 channel**, and is never below **10.0.14** (the latest release on 2026-10-25, verified on the .NET release metadata). It includes the fixes of 10.0.10 (CVE-2026-57108 and CVE-2026-50528, cited by ADR-0017) and fixes CVE-2026-111497 (Brotli), which is in the CISA catalogue of known exploited vulnerabilities.
- **Brotli decompression is never enabled**: neither ASP.NET Core request decompression nor `HttpClient` automatic response decompression. The 10.0.14 release notes state that both are off by default.
- The version floors of the other components are kept in [dependencies](../dependencies.md), not in an ADR, because they change every month. They are checked again by the CVE audit before each release.

## Alternatives considered

- **Enrolment tokens only from the web site (ADR-0017).** Rejected: no web site accounts exist in 0.1.
- **Moving the audit trail into 0.1.** Rejected: an even larger first increment; the gap is limited to 0.1 in the home lab and is recorded.
- **Serving the revocation lists from the existing Nginx.** Rejected: in the home lab it is the internet-facing reverse proxy; mixing the agents' endpoint into its configuration would put two trust zones in one file.
- **A second, dedicated Nginx instance.** Not adopted: it works, but it is configuration to maintain outside the repository; the dedicated service ships with Dashboard.
- **A Server CA list valid 30 or 90 days.** Rejected: 12 or 4 offline signatures a year, and with strict revocation a missed signature cuts every agent off; the compromise procedure is the real response.
- **Version floors in the ADR.** Rejected: a new ADR would be needed at every patch release.

## Consequences

### Positive

- Machine identity is complete from 0.1, without a temporary mechanism.
- The revocation lists do not depend on the Enrolment service, nor on a component exposed to the internet.
- The limit of revocation on the Collector channel is stated, and the real response is written down.
- An agent refused for its certificate is not confused with a machine that is off.

### Negative

- In 0.1, enrolments are recorded only in the issuance register, not in the audit trail.
- One more small service to deploy and monitor (`Dashboard.Crl`), and one more endpoint in the firewall.
- After a revocation of the Collector certificate, agents may trust it for up to 12 months unless the compromise procedure is applied.
- The compromise procedure and the Agent CA re-key both need an agent update deployed to every machine.
- Compromise of the whole "app" VM as root (including the Collector's key) is outside these guarantees, as in ADR-0017.

## References

- All references of ADR-0017.
- .NET release metadata, `releases-index.json`, channel 10.0 (consulted 2026-11-05); .NET 10.0.14 release notes (consulted 2026-11-05).
- dotnet/runtime source, `OpenSslCrlCache.cs`, cited by the second reading; not re-read by the main developer.
- RFC 5280, section 4.2.1.13 (distribution points).
- [Second reading report](../reviews/2026-11-04-second-reading.md), findings I1 to I4 and I8.
- ADR-0004, ADR-0005, ADR-0015, ADR-0016, ADR-0017 (superseded), ADR-0023, ADR-0024; ADR-0025 complements this ADR.
