# ADR-0007: Machine identity by client certificate, agent enrolment

| Field | Value |
|---|---|
| Status | Proposed |
| Date | 2026-11-03 |
| Decider | Michaël Defraene (product owner, architect) |
| Drafted by | Claude (main developer); the decisions had a second reading by the architecture review agent during scoping |
| Classification | Security-sensitive [SEC] |

## Context

The dashboard must show which log belongs to which machine, and remote commands must reach the right machine. Any field inside a message (host name, OTLP resource attributes) can be forged by a compromised or misconfigured agent. Conversely, the agent must be sure it talks to the real Backend, since it obeys the commands the Backend sends (ADR-0013).

## Decision

- **The origin machine is derived from the authenticated mTLS client certificate**: the certificate hash maps to a `HostId`. A field of the message is **never** used to identify the machine; resource attributes sent by the agent are informative only.
- The Collector's agent endpoint (ADR-0005) uses Kestrel with `ClientCertificateMode.RequireCertificate` and the `Microsoft.AspNetCore.Authentication.Certificate` handler. The handler trusts **only the private certificate authority** (`ChainTrustValidationMode = X509ChainTrustMode.CustomRootTrust`, with the CA chain in `CustomTrustStore`; by default it would trust the system store). It checks the validity period, the revocation status and the certificate use.
- The **agent validates the Backend's server certificate** against the private CA only, never against the system store alone.
- gRPC hardening: a bounded `MaxReceiveMessageSize`; `EnableDetailedErrors` is never enabled in production.
- **Enrolment, V1**: a small **private certificate authority** and an enrolment script run by the product owner issue one certificate per machine. The CA private key **does not live on the "app" VM**.
- **Enrolment, V1.x**: one-time enrolment token plus a certificate signing request (CSR) generated on the machine, so the private key never leaves the machine.

## Alternatives considered

- **Identity from a field of the message.** Rejected: forgeable.
- **API keys or shared secrets per agent.** Rejected: they travel with every request, are harder to rotate and do not bind the identity to the TLS session.
- **TLS terminated by the existing Nginx, certificate forwarded in a header.** Rejected: see ADR-0001.

## Consequences

### Positive

- A log or a heartbeat cannot be attributed to another machine without its private key.
- The same identity secures logs and commands.
- Removing a certificate from the hash-to-`HostId` mapping revokes the machine immediately, without waiting for a revocation list.

### Negative

- The handler's certificate-use check also accepts a certificate **without** any extended key usage; the CA must always issue the client-authentication usage, or the check must be made explicit.
- The hash changes when a certificate is renewed: the mapping must be updated as part of the renewal procedure.
- Revocation checking requires the private CA to publish a revocation list reachable by the Backend.
- An agent rejected at the TLS layer (expired, revoked or unknown certificate) is invisible to the Backend and looks like an offline machine.
- In V1, the enrolment script handles a private key for each machine; the script and its output must be protected.

## Open points

1. Private CA plus product-owner-run script in V1, token plus CSR in V1.x.
2. Kestrel and authentication handler settings listed above.
3. Raised in review, proposed: the hash-to-`HostId` lookup fails closed (a valid certificate that is not mapped is refused and logged); the mapping uses a SHA-256 hash rather than the SHA-1 `Thumbprint`; revocation checking fails closed; certificate expiry dates are tracked per host and alerted in advance; old and new certificates overlap during renewal.
4. Raised in review, proposed: the agent stores its private key with restricted access (file readable only by the agent's user on Linux, non-exportable key on Windows); the Backend applies a per-host rate or volume limit.

## References

- Microsoft Learn, *Configure certificate authentication in ASP.NET Core*, sections "Review proxy and load balancer scenarios" and "Configure certificate validation" (consulted 2026-11-03).
- ADR-0001, ADR-0004, ADR-0013.
