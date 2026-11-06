# ADR-0027: Machine identity with Agent CA anchor sets and a complete re-key

| Field | Value |
|---|---|
| Status | Accepted |
| Date | 2026-11-06 |
| Decider | Michaël Defraene (product owner, architect) |
| Drafted by | Claude (main developer), after the review of ADR-0026 on 2026-11-06 |
| Classification | Security-sensitive [SEC] |
| Supersedes | ADR-0021 |

## Context

ADR-0021 states that "each channel trusts exactly one CA", and keeps from ADR-0017 the transport settings in which the Agent CA is "the only anchor" and "the only entry of `CustomTrustStore`". The same ADR-0021 plans a re-key of the Agent CA every 2 years with an overlap period, during which "an agent update first ships the new anchor next to the old one": during that period, the channels that trust the Agent CA must trust two Agent CAs at once. The two statements were never reconciled.

The review of the first draft of ADR-0026 (Jocasta Nu, Ackbar, Boba Fett, C-3PO, 2026-11-06) found that changing this rule is a change of decision, which needs a new ADR that supersedes ADR-0021, and that the re-key itself is incomplete:

- the old Agent CA must keep publishing its revocation list while it is trusted, or strict revocation cuts the agents off;
- with two anchors, issuance must still come from the active CA only;
- the coverage report does not count the hosts that do not report, and nothing stops the old CA from being retired while hosts still depend on it;
- after the retirement, the old key and its offline backup could still sign a false Enrolment service that the agents accept;
- no actor is named, and the Enrolment service is a separate process that the host channel of the API does not drive (ADR-0021, ADR-0023).

If the trust stores accept a single anchor in 0.1, the first re-key, about two years after installation, would need a redesign of the agent.

This ADR keeps every decision of ADR-0021 except where it says otherwise.

## Decision

### Kept from ADR-0021

The list below is a reminder of the main points, not a complete list: every decision of ADR-0021 that this ADR does not change stays, and ADR-0021 holds the details (renewal rules, CSR policy, rate limits, sockets, the details of `Dashboard.Crl`).

- The machine is identified by the SHA-256 hash of its authenticated client certificate, mapped to a `HostId` by the API; the mapping fails closed.
- **Two independent CAs**: the Server CA (self-signed, key offline with the product owner) signs only the Collector's server certificate; the Agent CA (self-signed, key online in the Enrolment service) signs agent certificates and the Enrolment service's server certificate; the channels use **distinct DNS names**.
- The Enrolment service is a signer only; the API checks and consumes the token, verifies the renewal proof and checks every issued certificate before registering it.
- **Strict revocation on every channel**, an unknown or offline status being a failure; the Agent CA list valid 7 days and regenerated daily; the Server CA list valid 12 months, re-signed at least every 6 months and alerted 60 days before its `nextUpdate`.
- Enrolment tokens created by a host command from 0.1 and in the web site from 0.2; the token read from a root-only file or standard input, never from the command line; the **issuance register**, the only record in 0.1.
- The **`Dashboard.Crl`** service, which serves the revocation lists as static files.
- The limit of the revocation of the Collector certificate (up to 12 months) and the written **compromise procedure** of the Collector key.
- "Down: refused" and "down: silent" on the host page; the days before the certificate expires in the heartbeat.
- The .NET runtime on the latest patch of the 10.0 channel, never below the floor stated in ADR-0021 (10.0.14); Brotli decompression never enabled; the other floors in [dependencies](../dependencies.md).

### Trust rule

- **Each channel trusts exactly one CA role**: the Server CA role on the channel from the agent to the Collector; the Agent CA role on the channel from the agent to the Enrolment service and on the client certificates checked by the Collector. This replaces "each channel trusts exactly one CA" of ADR-0021 and the single anchor of the transport settings kept from ADR-0017.
- A channel of the Agent CA role holds a **set of Agent CA anchors**, from **0.1**. Outside a re-key, the set holds one anchor; during the overlap period, two.
- A channel never trusts the Server CA and an Agent CA together. The anchor set is checked when the service starts, and the service refuses to start, with a clear message, unless:
  - the set holds one or two anchors;
  - each anchor is self-signed and marked as a CA;
  - each anchor is known as an Agent CA, active or in an overlap period: on the "app" VM, from the state of the re-key; on an agent, from the signed package that delivered it;
  - no anchor is a retired Agent CA, nor the Server CA.

  If the state of the re-key cannot be read, the service also refuses to start.
- Microsoft Learn documents `X509ChainPolicy.CustomTrustStore` as a collection of certificates, used with `X509ChainTrustMode.CustomRootTrust`. That several independent self-signed roots are each accepted, on Linux and on Windows, and how Kestrel behaves on its own in its client-certificate callback, are **not verified**: a test checks them when the 0.1 work starts, with two Agent CAs that must each succeed and a third CA that must fail. If the test fails, the 0.1 work stops and the product owner decides before any single-anchor fallback.

### Re-key of the Agent CA

- **Actors**:
  - the **key holder** signs the agent package that ships the new anchor next to the old one (ADR-0025);
  - the **host administrator** runs the steps on the "app" VM: generate the new Agent CA, switch the Enrolment service to it, read the coverage report, retire the old CA. These steps are host commands, audited once the audit trail exists (0.2).
- **Bounded overlap**: the maximum duration of the overlap is set in the host's configuration. Beyond it, a Warning is raised, then a critical alert after a second limit, also set there.
- **Issuance by the active CA only**: from the switch on, the Enrolment service issues and renews certificates with the new CA only, and the API checks that every issued certificate chains to the **active** Agent CA, not to any anchor of the set.
- **Revocation of both CAs**: until it is retired, the old CA keeps its key online in the Enrolment service and keeps publishing its revocation list, valid 7 days and regenerated daily, like the active one. A missing or expired list of either CA is a failure, as for any list (strict revocation).
- **Coverage report**: each agent reports in its heartbeat the fingerprints of the Agent CA anchors it holds. The report lists **every enrolled host that is neither revoked nor retired**, monitor identities included, as "covered", "not covered" or "unknown" (no anchor report since the switch). It carries its generation time.
- **Retirement of the old CA** is refused while a host is "not covered" or "unknown", checked on fresh data when the command runs, unless the host administrator passes an explicit override, which is audited and raises a critical alert.
- **After the retirement**:
  - the old anchor is removed from the Collector at once, and a certificate that chains to the retired CA is refused there;
  - the agents keep the old anchor until the next signed package removes it; the coverage report then follows its removal on each agent;
  - the old key and its offline backup are destroyed, and the destruction is recorded in the key backup register (ADR-0026);
  - virtual machine images of the "app" VM taken before the retirement still hold the old key: they stay encrypted (ADR-0025) and are destroyed at the end of their retention.

## Alternatives considered

- **Keep the rule of ADR-0021 and add a note.** Rejected: it would change a testable rule of an accepted ADR without superseding it, which the rule of the ADR index forbids.
- **One anchor per channel until the first re-key.** Rejected: the first re-key would need a redesign of the agent.
- **A new Agent CA signed by the old one (cross-signing) instead of an anchor set.** Not adopted, for the same reason as ADR-0017: an intermediate certificate used as a trust anchor is not documented by Microsoft.
- **Retiring the old CA on a fixed date, whatever the coverage.** Rejected: the hosts that missed the update would be cut off without warning.

## Consequences

### Positive

- The Agent CA can be re-keyed without changing the agent, and without cutting off a host that has not received the new anchor.
- The separation between the Server CA and the Agent CA stays enforced, and an unexpected anchor stops the service when it starts.
- The Collector refuses the old CA at once after the retirement, and the agents once the next signed package is installed.

### Negative

- The trust stores of 0.1 handle a set of anchors, which their tests must cover: one anchor, two anchors, the Server CA refused on an Agent CA channel, an unknown root refused at start-up, a chain from a retired CA refused, a missing list of either CA.
- The coverage is reported by the agents themselves: a compromised or faulty agent can claim to be covered.
- Until the next signed package is installed, and in the virtual machine images taken before the retirement, the old key can still sign for the agents that keep the old anchor.
- The heartbeat carries one more field (the anchor fingerprints).
- During the overlap, two revocation lists and two online keys of the Agent CA role must be kept.
- 0.1 depends on a behaviour that is not verified yet; if it fails, the 0.1 work stops for a decision.

## Open points

- How the host commands reach the Enrolment service (a host channel of the Enrolment service itself, or the API relaying them) is decided when the 0.1 work starts; ADR-0023 defines the host channel of the API only.

## References

- All references of ADR-0021.
- Microsoft Learn, `X509ChainPolicy.CustomTrustStore`, `X509ChainTrustMode` (net-10.0), `CertificateAuthenticationOptions.CustomTrustStore`, and *Configure certificate authentication in ASP.NET Core* (aspnetcore-10.0), consulted by the review on 2026-11-06.
- [Review report of 2026-11-06](../reviews/2026-11-06-roles-and-anchor-sets.md).
- ADR-0017 (superseded), ADR-0021 (superseded), ADR-0023, ADR-0025, ADR-0026; ADR-0029 complements this ADR, and tightens the closing of the open streams of a revoked machine; ADR-0030 complements this ADR, and tightens the check of the issued certificate, the one-successor rule, the effect of a suspect host and the CSR policy; ADR-0031 complements this ADR, and tightens the storage of the enrolment token.
