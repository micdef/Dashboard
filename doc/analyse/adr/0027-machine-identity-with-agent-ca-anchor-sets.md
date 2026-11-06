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

### Trust rule

- **Each channel trusts exactly one CA role**: the Server CA role on the channel from the agent to the Collector; the Agent CA role on the channel from the agent to the Enrolment service and on the client certificates checked by the Collector. This replaces "each channel trusts exactly one CA" of ADR-0021 and the single anchor of the transport settings kept from ADR-0017.
- A channel of the Agent CA role holds a **set of Agent CA anchors**, from **0.1**. Outside a re-key, the set holds one anchor; during the overlap period, two.
- A channel never trusts the Server CA and an Agent CA together. The anchor set is checked when the service starts: if it contains the Server CA, the service refuses to start, with a clear message.
- Microsoft Learn documents `X509ChainPolicy.CustomTrustStore` as a collection of certificates, used with `X509ChainTrustMode.CustomRootTrust`. That several independent self-signed roots are each accepted, on Linux and on Windows, and how Kestrel behaves on its own in its client-certificate callback, are **not verified**: a test checks them when the 0.1 work starts, with two Agent CAs that must each succeed and a third CA that must fail. If the test fails, the 0.1 work stops and the product owner decides before any single-anchor fallback.

### Re-key of the Agent CA

- **Actors**:
  - the **key holder** signs the agent package that ships the new anchor next to the old one (ADR-0025);
  - the **host administrator** runs the steps on the "app" VM: generate the new Agent CA, switch the Enrolment service to it, read the coverage report, retire the old CA. These steps are audited host commands.
- **Issuance by the active CA only**: from the switch on, the Enrolment service issues and renews certificates with the new CA only, and the API checks that every issued certificate chains to the **active** Agent CA, not to any anchor of the set.
- **Revocation of both CAs**: until it is retired, the old CA keeps its key online in the Enrolment service and keeps publishing its revocation list, valid 7 days and regenerated daily, like the active one. A missing or expired list of either CA is a failure, as for any list (strict revocation).
- **Coverage report**: each agent reports in its heartbeat the fingerprints of the Agent CA anchors it holds. The report lists **every enrolled host that is not retired**, monitor identities included, as "covered", "not covered" or "unknown" (no anchor report since the switch). It carries its generation time.
- **Retirement of the old CA** is refused while a host is "not covered" or "unknown", unless the host administrator passes an explicit override, which is audited and raises a critical alert.
- **After the retirement**: the old anchor is removed from the Collector at once and from the agents by the next signed package; a certificate that chains to the retired CA is refused; the old key and its offline backup are destroyed, and the destruction is recorded in the key backup register (ADR-0026).

## Alternatives considered

- **Keep the rule of ADR-0021 and add a note.** Rejected: it would change a testable rule of an accepted ADR without superseding it, which the rule of the ADR index forbids.
- **One anchor per channel until the first re-key.** Rejected: the first re-key would need a redesign of the agent.
- **A new Agent CA signed by the old one (cross-signing) instead of an anchor set.** Not adopted: it relies on intermediate anchors, which ADR-0017 rejected because Microsoft does not document them as trust anchors.
- **Retiring the old CA on a fixed date, whatever the coverage.** Rejected: the hosts that missed the update would be cut off without warning.

## Consequences

### Positive

- The Agent CA can be re-keyed without changing the agent, and without cutting off a host that has not received the new anchor.
- The separation between the Server CA and the Agent CA stays enforced, and is checked when each service starts.
- After a re-key, nothing can still sign for the old CA.

### Negative

- The trust stores of 0.1 handle a set of anchors, which their tests must cover: one anchor, two anchors, the Server CA refused on an Agent CA channel, a chain from a retired CA refused, a missing list of either CA.
- The heartbeat carries one more field (the anchor fingerprints).
- During the overlap, two revocation lists and two online keys of the Agent CA role must be kept.
- 0.1 depends on a behaviour that is not verified yet; if it fails, the 0.1 work stops for a decision.

## Open points

- How the host commands reach the Enrolment service (a host channel of the Enrolment service itself, or the API relaying them) is decided when the 0.1 work starts; ADR-0023 defines the host channel of the API only.

## References

- All references of ADR-0021.
- Microsoft Learn, `X509ChainPolicy.CustomTrustStore` and `X509ChainTrustMode` (net-10.0), and *Configure certificate authentication in ASP.NET Core* (aspnetcore-10.0), consulted by the review on 2026-11-06.
- ADR-0017 (superseded), ADR-0021 (superseded), ADR-0023, ADR-0025, ADR-0026.
