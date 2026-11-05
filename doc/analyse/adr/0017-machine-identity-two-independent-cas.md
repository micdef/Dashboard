# ADR-0017: Machine identity with two independent certificate authorities

| Field | Value |
|---|---|
| Status | Accepted |
| Date | 2026-11-04 |
| Decider | Michaël Defraene (product owner, architect) |
| Drafted by | Claude (main developer), after a review by the architecture, security and CVE review agents |
| Classification | Security-sensitive [SEC] |
| Supersedes | ADR-0007 |

## Context

ADR-0007 introduced the Enrolment service and a private CA made of an offline root and two intermediates. Its review found two blocking flaws:

1. The Enrolment service's own server certificate was signed by the same intermediate as the Collector's, with its key online. A compromised Enrolment service could therefore impersonate the Collector to the agents, and so send them commands.
2. On renewal, the API updated the certificate-to-host mapping on the word of the Enrolment service alone. A compromised Enrolment service could therefore take over any existing machine.

The review also noted that anchoring trust on a non-self-signed intermediate (`CustomRootTrust` with an intermediate only) is **not documented** by Microsoft, which describes the custom trust store as the chain "including the trusted root". This ADR replaces ADR-0007. The goals are unchanged: the machine identity comes from the certificate, the private key never leaves the machine, enrolment uses a one-time token and a CSR from V1, and revocation is strict from V1.

## Decision

### Machine identity

- The origin machine is derived from the authenticated mTLS client certificate: a **SHA-256 hash** of the certificate maps to a `HostId` (not the SHA-1 `Thumbprint`). The mapping also stores the issuer and the serial number, which the revocation list needs. A field of the message is never used to identify the machine.
- The mapping **fails closed**: a certificate that is not mapped, or is marked revoked, is refused and the refusal is logged with its hash.
- The mapping belongs to the API: only the API registers, renews or revokes an entry (see "Enrolment").

### Two independent certificate authorities

| CA | Key | Signs | Trusted by |
|---|---|---|---|
| **Server CA** (self-signed root) | **offline**, kept by the product owner | the Collector's server certificate only | the agents, for the Collector channel only |
| **Agent CA** (self-signed root) | online, in the Enrolment service | agent client certificates, and the Enrolment service's own server certificate | the Collector, for agent certificates; the agents, for the enrolment channel only |

- The two CAs share **no root**. Each trust anchor is a self-signed root, which is the documented use of `CustomRootTrust`.
- Each channel trusts exactly one CA: the agent validates the Collector **only** against the Server CA, and the Enrolment service **only** against the Agent CA. A certificate from the Agent CA can therefore never be accepted as the Collector, whatever happens to the Enrolment service.
- Defence in depth: the Collector and the Enrolment service have **distinct DNS names**, and the agent checks the expected name on each channel. The Collector requires the client-authentication usage explicitly (the handler alone would also accept a certificate without any usage).

### Enrolment

The Enrolment service (`Dashboard.Enrolment`, ADR-0016) is a **signer only**. The API takes every identity decision and verifies it itself; it never accepts a statement of the Enrolment service as proof.

- **Token**: created by an administrator in the web site; at least 128 bits from `RandomNumberGenerator`; only its SHA-256 hash is stored and it is compared in constant time; short lifetime (15 to 60 minutes); bound to an expected `HostId`. The token is delivered out of band (installation argument or a file readable only by root) and deleted by the agent after use; it never appears in logs.
- **First enrolment**:
  1. The agent generates its key pair locally and sends a CSR with the token.
  2. The **API checks and consumes the token** in one atomic operation (check, mark used, record the issuance), so it cannot be used twice.
  3. The Enrolment service signs the certificate with the subject set from the token's `HostId`, **never** from the CSR.
  4. Before registering it, the API checks the issued certificate: chain to the Agent CA, public key equal to the CSR's, expected `HostId`, client-authentication usage, expected validity.
- **Renewal**: the agent signs a request (new CSR, a nonce supplied by the API, its `HostId`) with its **current private key**. The API verifies this signature with the public key of the certificate currently mapped, then applies the same checks as above. Rules:
  - a **new key pair** is mandatory at each renewal;
  - **one successor** per certificate: a second renewal from the same certificate is refused, alerted, and the host is marked suspect;
  - the old certificate is revoked at the end of the overlap;
  - an **identity lifetime** of 1 year: after it, a new token is required;
  - the source address of each renewal is logged, and a change is alerted.
- **CSR policy**: `LoadSigningRequest` with the default options (which checks the CSR signature, the proof of possession, and does not load the requested extensions); `UnsafeLoadCertificateExtensions` and `SkipSignatureValidation` are never used. Key allowlist: ECDSA P-256 or P-384, or RSA of at least 3072 bits. The CSR size is capped. Serial numbers are random (at least 64 bits). The certificate sets `CA=false`, the digital signature usage, the client-authentication usage, the key identifiers and the revocation list distribution point. Signatures use SHA-256 or stronger.
- **Endpoint**: the unauthenticated enrolment endpoint listens only on the interface facing the supervised machines and is rate-limited per source address, with a lockout after repeated failures. Renewal requires the client certificate.
- **Enrolment service → API**: a Unix domain socket reserved to the Enrolment service's group, exposing only the enrolment, renewal and revoked-serials services. Every call is audited.

### Revocation: strict on every channel

| Revocation list | Signed by | Validity (`nextUpdate`) | Renewal |
|---|---|---|---|
| Agent CA list (agent certificates, Enrolment server certificate) | the Enrolment service | 7 days | regenerated daily |
| Server CA list (Collector server certificate) | offline, by the product owner | 12 months | re-signed at least every 6 months |

- **Strict everywhere**: the Collector checks the agents, and the agents check the Collector and the Enrolment service. Without a valid revocation list, the connection is refused. On the agent side, revocation checking is off by default in .NET (`CertificateRevocationCheckMode` defaults to `NoCheck`); it is enabled explicitly, and an unknown or offline revocation status is treated as a failure.
- Every chain has a self-signed root and one leaf, so the default `ExcludeRoot` flag checks exactly the leaf certificate.
- The download timeout (`UrlRetrievalTimeout`, unlimited by default) is set to a few seconds.
- The revocation lists are served as **static files** by a component independent of the Enrolment service, so they stay available when it is down. An outage of the Enrolment service of up to about 6 days has no effect.
- **Revocation of a machine**: an administrator revokes it in the web site; the API marks the mapping revoked; the API notifies the Collector, which **closes the open streams** of that machine at once (same path as commands, ADR-0005); the Enrolment service adds the serial to its next list.
- A **break-glass procedure** is written for an expired Server CA list (re-signing it offline).

### Transport settings

- Kestrel requires the client certificate (`ClientCertificateMode.RequireCertificate`) and builds the chain itself, with the Agent CA as the only anchor and revocation enabled, in its client-certificate validation callback. `AllowAnyClientCertificate` is forbidden outside tests.
- The `Microsoft.AspNetCore.Authentication.Certificate` handler uses `ChainTrustValidationMode = X509ChainTrustMode.CustomRootTrust` with the Agent CA in `CustomTrustStore` (by default it would trust the system store).
- The agent sets its chain policy explicitly: `TrustMode = CustomRootTrust`, the expected CA as the only entry of `CustomTrustStore`, revocation online.
- gRPC hardening: a bounded `MaxReceiveMessageSize`; `EnableDetailedErrors` is never enabled in production.
- The Collector applies a **rate and volume limit per machine**.
- The agent's private key is readable only by the agent's user on Linux, and non-exportable on Windows. The Agent CA key is readable only by the Enrolment service's user (mode 0400), and its systemd unit is hardened.
- Runtime: **.NET 10.0.10 or later** on the "app" VM and on every agent (CVE-2026-57108, X.509 parsing denial of service on Linux and macOS; CVE-2026-50528, security feature bypass; both fixed in 10.0.10). The system OpenSSL, on which .NET depends on Linux, is kept up to date.

### Validity periods

| Certificate | Validity |
|---|---|
| Agent certificate | 30 days, renewed at two thirds of its life |
| Enrolment service server certificate | 90 days, renewed by the service itself |
| Collector server certificate | 1 year |
| Agent CA | 2 years, with a written re-key procedure |
| Server CA | 5 years |

### Offline key custody

- The Server CA key is kept on an **encrypted offline medium**, with a **backup** on a second medium stored separately. Losing it means rebuilding the Server CA and redeploying its anchor on every agent.
- The offline operations (signing the Collector certificate, re-signing the Server CA list) follow a **written procedure** and are triggered by the alerts. The manual part is limited to a few signatures a year.

### Audit and monitoring

- Every issuance is recorded in the immutable audit trail: serial, SHA-256 hash, `HostId`, token identifier, the administrator who created the token, source address, timestamp. Every revocation: administrator, reason, timestamp.
- **Reconciliation**: a certificate seen by the Collector whose serial is not in the issuance record raises an alert.
- Expiry dates (agents, servers, both CAs) and the freshness of every revocation list are tracked and alerted in advance (ADR-0006, "Failure visibility"); the Server CA list is alerted 60 days before its `nextUpdate`.

### Installation package and tests

- The agent installation package, built by the administrator, carries the two trust anchors (Server CA, Agent CA), the two DNS names and the enrolment address. No trust material is fetched at enrolment time.
- Behaviours not documented by Microsoft are **tested on Linux** before the code is accepted: strict revocation with an expired or unreachable list, the revocation list cache of a system user without a home directory, and the refusal of a certificate from the other CA on each channel.

## Alternatives considered

- **ADR-0007's offline root with two intermediates.** Rejected: the Enrolment service's server certificate shared the Collector's trust path, it relies on an undocumented intermediate anchor, and it requires two offline keys and a root revocation list.
- **A single CA.** Rejected: compromising its key would allow impersonating the Collector.
- **Revocation checked only on agent certificates.** Rejected by the product owner: revocation is strict on every channel.
- **The Enrolment service registering the mapping.** Rejected: the API must verify every identity decision itself.

## Consequences

### Positive

- A compromised Enrolment service can neither impersonate the Collector (separate CA, separate name) nor take over a machine (the API verifies the token and the renewal proof itself). The worst it can do is refuse to sign, which delays enrolments and renewals.
- One offline key instead of two, and each trust anchor uses the documented mechanism.
- Revocation is both immediate (mapping, closed streams) and standard (revocation lists).

### Negative

- Strict revocation on the Collector channel depends on a list signed offline: if it expires, no agent can connect. The 12-month validity, the 60-day alert and the break-glass procedure are the safeguards.
- Many security-sensitive rules to implement and review, each needing a test.
- Compromise of the whole "app" VM as root (including the Collector's key) is outside these guarantees.
- The Enrolment service depends on the API: no enrolment or renewal while the API is down.

## References

- Microsoft Learn, *Configure certificate authentication in ASP.NET Core* (consulted 2026-11-03 and 2026-11-04).
- Microsoft Learn, *Security considerations for Kestrel*; `X509ChainPolicy.UrlRetrievalTimeout`; `SslClientAuthenticationOptions.CertificateRevocationCheckMode` (consulted by the reviews, 2026-11-04).
- Microsoft Learn, API reference: `CertificateRequest`, `CertificateRequest.LoadSigningRequest`, `CertificateRequestLoadOptions`, `CertificateRevocationListBuilder` (.NET 10, consulted 2026-11-04).
- Microsoft Learn, *Cross-platform cryptography in .NET* (OpenSSL dependency on Linux).
- GitHub security advisories GHSA-rp2p-6cmp-jxj9 (CVE-2026-57108) and GHSA-qvw7-jm5c-6hqw (CVE-2026-50528), dotnet/runtime (consulted 2026-11-04).
- RFC 5280, sections 4.2.1.13, 5.1.2.5 and 6.3.3.
- ADR-0004, ADR-0005, ADR-0006, ADR-0007 (superseded), ADR-0020, ADR-0016.
