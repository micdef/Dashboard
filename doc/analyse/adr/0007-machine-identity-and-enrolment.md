# ADR-0007: Machine identity by client certificate, agent enrolment

| Field | Value |
|---|---|
| Status | Accepted |
| Date | 2026-11-03 (proposed), 2026-11-04 (accepted) |
| Decider | Michaël Defraene (product owner, architect) |
| Drafted by | Claude (main developer); the decisions had a second reading by the architecture review agent during scoping |
| Classification | Security-sensitive [SEC] |

## Context

The dashboard must show which log belongs to which machine, and remote commands must reach the right machine. Any field inside a message (host name, OTLP resource attributes) can be forged by a compromised or misconfigured agent. Conversely, the agent must be sure it talks to the real Collector, since it obeys the commands it receives (ADR-0013).

Certificates must be issued, renewed and revoked for every machine. Doing this by hand does not scale and makes revocation lists easy to forget; a revocation list that expires blocks every agent when revocation checking is strict. .NET provides the building blocks natively: `CertificateRequest` creates self-signed or CA-signed certificates and loads PKCS#10 signing requests (`LoadSigningRequest`), and `CertificateRevocationListBuilder` builds and signs revocation lists, including the distribution point extension.

## Decision

### Machine identity

- **The origin machine is derived from the authenticated mTLS client certificate**: a **SHA-256 hash** of the certificate maps to a `HostId` (not the SHA-1 `Thumbprint`). A field of the message is **never** used to identify the machine; resource attributes sent by the agent are informative only.
- The mapping **fails closed**: a certificate that chains to the private CA but is not mapped, or is marked revoked, is refused and the refusal is logged with its hash.

### Certificate authority: offline root, two intermediates

| Certificate | Where its key lives | Signs |
|---|---|---|
| Root CA (self-signed) | **offline**, kept by the product owner, never on the "app" VM | the two intermediates and the root revocation list |
| Agent-issuing intermediate | the Enrolment service (below) | agent client certificates only (client authentication usage) |
| Server-issuing intermediate | **offline**, kept by the product owner | the server certificates of the Collector and of the Enrolment service |

- Agents validate the Collector **only against the server-issuing intermediate**. A compromised Enrolment service can therefore create false agents (detectable and revocable), but it cannot impersonate the Collector, and so cannot send commands to the machines.

### Enrolment service

- A dedicated daemon, **`Dashboard.Enrolment`**, runs on the "app" VM as its own systemd service under its own system user. It is not part of the Backend tier (ADR-0005) and has **no access to the data gateway**: it records and checks everything through dedicated services of the API, over a Unix domain socket reserved for it.
- **First enrolment (from V1)**:
  1. An administrator creates a **one-time enrolment token** in the web site. The token is bound to an expected machine and expires quickly.
  2. The agent generates its key pair locally and sends a **certificate signing request (CSR)** with the token. Its private key never leaves the machine.
  3. The Enrolment service checks the token, then issues the certificate. It **ignores the extensions requested in the CSR** and sets them itself (client authentication usage, validity, distribution point).
- **Renewal**: before expiry, the agent requests a new certificate, authenticated by its current valid certificate. The old and the new certificates overlap, and the mapping is updated by the API in the same operation.
- **Revocation**: an administrator revokes a machine in the web site. The mapping marks it revoked at once (immediate refusal by the Collector), and the Enrolment service adds the certificate to its revocation list.
- **Revocation lists**: the Enrolment service regenerates the list of the agent-issuing intermediate automatically, well before its expiry, and publishes the lists over HTTP for the Collector and the agents. The lists signed offline (root, server-issuing intermediate) are published by the same service.

### Transport settings

- The Collector's agent endpoint (ADR-0005) uses Kestrel with `ClientCertificateMode.RequireCertificate` and the `Microsoft.AspNetCore.Authentication.Certificate` handler, set to trust **only the private CA** (`ChainTrustValidationMode = X509ChainTrustMode.CustomRootTrust`, with the CA chain in `CustomTrustStore`; by default it would trust the system store). It checks the validity period, the client authentication usage and the revocation status.
- **Revocation checking is strict**: without a valid revocation list, the connection is refused.
- The agent validates the server certificate against the private CA only, never against the system store.
- gRPC hardening: a bounded `MaxReceiveMessageSize`; `EnableDetailedErrors` is never enabled in production.
- The Collector applies a **rate and volume limit per machine** (thresholds in configuration, set from measured volumes).
- The agent stores its private key with restricted access: readable only by the agent's user on Linux, non-exportable on Windows.

### Monitoring

- Certificate expiry dates (agents, servers, intermediates, root) and the freshness of every revocation list are tracked and **alerted in advance** (ADR-0006, "Failure visibility").

## Alternatives considered

- **Identity from a field of the message.** Rejected: forgeable.
- **API keys or shared secrets per agent.** Rejected: they travel with every request, are harder to rotate and do not bind the identity to the TLS session.
- **TLS terminated by the existing Nginx, certificate forwarded in a header.** Rejected: see ADR-0001.
- **A private CA operated by hand with a script.** Rejected: manual issuing and renewal, a script that handles the private key of every machine, and revocation lists easy to forget.
- **The CA inside the Collector.** Rejected: the CA key would live in the process most exposed to the network of the supervised machines.
- **A single CA key on the "app" VM.** Rejected: compromising it would allow impersonating the Collector and commanding every machine.
- **No revocation list in V1 (revocation by the mapping only).** Rejected by the product owner: standard X.509 revocation is required from V1; the mapping remains as an immediate second barrier.

## Consequences

### Positive

- A log or a heartbeat cannot be attributed to another machine without its private key.
- Agent private keys never leave their machines; enrolment, renewal and revocation are automated.
- Revocation is both immediate (mapping) and standard (revocation lists).
- The worst outcome of a compromised Enrolment service is bounded: false agents, not control of the machines.

### Negative

- One more service to build, deploy and secure, and it is security-sensitive code.
- Strict revocation means that an expired or unreachable revocation list blocks the affected connections; the monitoring must catch it first.
- The product owner must handle the offline keys: signing the intermediates, the server certificates and the offline revocation lists, on a schedule driven by the alerts.
- The handler's certificate-use check also accepts certificates **without** any extended key usage; this is harmless only because the Enrolment service always sets the client authentication usage. It must stay so.

## References

- Microsoft Learn, *Configure certificate authentication in ASP.NET Core*, sections "Review proxy and load balancer scenarios" and "Configure certificate validation" (consulted 2026-11-03).
- Microsoft Learn, API reference: `CertificateRequest`, `CertificateRequest.LoadSigningRequest`, `CertificateRevocationListBuilder` (.NET 10, consulted 2026-11-04).
- ADR-0001, ADR-0004, ADR-0005, ADR-0006, ADR-0013, ADR-0016.
