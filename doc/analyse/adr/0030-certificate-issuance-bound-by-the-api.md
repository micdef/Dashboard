# ADR-0030: Certificate issuance bound by the API

| Field | Value |
|---|---|
| Status | Accepted |
| Date | 2026-11-06 |
| Decider | Michaël Defraene (product owner, architect) |
| Drafted by | Claude (main developer), after the team review of the sequence diagrams of 0.1; second reading by the architecture review agent before the decisions |
| Classification | Security-sensitive [SEC] |
| Complements | ADR-0027 |

## Context

The Enrolment service is a signer only: the API takes every identity decision and never accepts a statement of the Enrolment service as proof (ADR-0017, kept by ADR-0027). The first drafts of the sequence diagrams of 0.1 showed that the rules, as written, leave gaps:

- after signing, the API checks the issued certificate against a CSR that the Enrolment service itself supplies. At renewal, a compromised Enrolment service could forward the agent's genuine signed request, sign a certificate for its own key, and register it with its own CSR: within the 20 days of the renewal cycle, it would hold the identity of every machine. ADR-0017 states that a compromised Enrolment service cannot take over a machine;
- two renewals sent at the same time from the same certificate, by an agent and by the thief of its key, can both read "no successor yet": the one-successor rule, which detects a stolen key, would not fire. An honest agent that lost the answer to a successful renewal and tries again would be taken for a second renewal;
- "the host is marked suspect" has no defined effect;
- a certificate refused at registration stays validly signed by the Agent CA for 30 days;
- the API checks a list of properties of the issued certificate, not a complete profile;
- on the unauthenticated enrolment endpoint, the CSR is loaded, and its signature checked, before its size is bounded, and the key allowlist has no upper bound for RSA;
- every issuance is recorded (ADR-0017), but the Enrolment service's own server certificate is renewed by the service itself, and the socket from the Enrolment service to the API exposes "only the enrolment, renewal and revoked-serials services" (ADR-0017).

This ADR adds to ADR-0027 and tightens six rules that ADR-0027 keeps from ADR-0017 and ADR-0021; each one is named below. It weakens none.

## Decision

### Issuance ticket

- Before any signature, the API issues an **issuance ticket**: single use, with a short lifetime set in the configuration, stored through the data gateway, and bound to:
  - the `HostId` and the kind of identity (agent; monitor from 0.2);
  - the SHA-256 hash of the new public key (its `SubjectPublicKeyInfo`);
  - for a renewal, the hash of the predecessor certificate; for a first enrolment, the identifier of the consumed token (ADR-0031).
- The API reads the new public key **itself**:
  - at renewal, from the CSR inside the request that the agent signed with its current key (ADR-0017), once it has verified that signature;
  - at first enrolment, from the CSR passed with the token, which the API loads with the same CSR policy as the Enrolment service. What a compromised Enrolment service can still do at first enrolment is stated in ADR-0031.
- The Enrolment service registers the certificate with `register(ticket, certificate)`, **never with a CSR**. The API refuses the certificate unless its public key matches the hash bound in the ticket, together with the profile below.
- This tightens the check of the issued certificate in ADR-0017 ("public key equal to the CSR's"), kept by ADR-0027: the CSR that counts is the one the API read itself.

### Exact certificate profile

The API compares each issued agent certificate with one exact profile, not with a list of properties:

- the complete list of extensions, and nothing else;
- `CA=false`; key usage digital signature only;
- client authentication as the **only** extended usage;
- no DNS name in the subject alternative names;
- a signature with SHA-256 or stronger;
- issuer and authority key identifier equal to the **active** Agent CA (ADR-0027);
- the validity of ADR-0017 (30 days), within a tolerance set in the configuration.

This tightens the checks of ADR-0017 kept by ADR-0027.

### One successor, reserved atomically

- When the API accepts a renewal request, it reserves the successor of the predecessor certificate **in the same transaction** as the consumption of the nonce and the creation of the ticket, under a unique constraint on the predecessor. Two concurrent requests can therefore never both succeed.
- A request that comes again with the **same new public key** is a retry: the API answers with the same ticket, or with the certificate already issued, which it keeps for that purpose. The agent writes its new key to disk before it sends the request, so that it can retry.
- A request with a **different public key** from the same predecessor is a second renewal: it is refused and the host is marked suspect (ADR-0017).
- This tightens the one-successor rule of ADR-0017 kept by ADR-0027.

### Suspect host

- A suspect host is refused **at renewal and at the Collector**, for every certificate mapped to it, until the host administrator revokes it explicitly and enrols it again (ADR-0026). The Collector sees the state at the next call, through the resolution of ADR-0029.
- Becoming suspect raises a critical alert from 0.2 (ADR-0017 already alerts it); in 0.1, it is shown on the development status page.
- Nobody can know which holder of the key is the honest one: refusing both is the only safe choice. The thief of a key can therefore cut the host off, which is visible, and less harmful than taking its place.
- This defines the effect of "the host is marked suspect" of ADR-0017, kept by ADR-0027.

### Refused certificates

- A certificate refused at registration is recorded in the issuance register with its serial, and its serial is **revoked**: it is on the next Agent CA list, which the Enrolment service reads from the API (ADR-0029). The serial leaves the list once the certificate has expired (ADR-0029).

### CSR intake

- On the enrolment endpoint, in this order: the rate limit per source address (ADR-0017), a bound on the size of the request (`MaxReceiveMessageSize`), a bound in bytes on the CSR, set in the configuration, and only then the loading of the CSR.
- The key allowlist becomes: ECDSA P-256 or P-384, or **RSA 3072 or 4096**. This tightens the CSR policy of ADR-0017 ("RSA of at least 3072 bits"), kept by ADR-0027.
- The Enrolment service applies this policy, and the API applies it again to every CSR it reads.

### The Enrolment service's server certificate

- The Enrolment service keeps renewing its own server certificate (ADR-0017). Nothing is added to its socket to the API.
- The API reads the **public certificate file** of the Enrolment service (read access for its group, never the key), when the file changes and at least once a day. It checks its profile (server authentication, the DNS name of the Enrolment service, issued by the active Agent CA), records its serial, hash and expiry in the issuance register, and tracks its expiry: alerted from 0.2, shown on the development status page in 0.1.
- When, within its 90 days, the service renews it is set in its configuration.
- **Limit**: this records what the file holds, not what the service serves. A compromised Enrolment service holds the Agent CA key and can serve a certificate it never wrote to the file.

### What a compromised Enrolment service can still do

With these rules, a compromised Enrolment service can still: refuse to sign, and so delay enrolments and renewals; sign certificates that are never registered, which the Collector refuses because the mapping fails closed (ADR-0027); serve an unregistered server certificate on the enrolment channel; state a false source address, since the address that the API records comes from it; leave serials out of the revocation list it writes, which the mapping makes harmless for agent certificates (ADR-0029); and misuse the tokens that pass through it (ADR-0031). It can never pass as the Collector (ADR-0027).

## Alternatives considered

- **Keep the CSR in the registration call.** Rejected: the check would compare the certificate with what the checked party supplies.
- **Mark a suspect host on the dashboard only, or refuse it at renewal only.** Rejected: if the thief renewed first, it would keep a valid and mapped certificate.
- **Let a refused certificate expire on its own.** Rejected: it would stay usable for 30 days on the enrolment channel.
- **A fourth service on the socket of the Enrolment service, to register its server certificate.** Rejected: it widens a surface that ADR-0017 limits, which would need an ADR that supersedes.
- **No record of the Enrolment service's server certificate.** Rejected: every issuance is recorded (ADR-0017).

## Consequences

### Positive

- A compromised Enrolment service can no longer take over an enrolled machine at renewal.
- The one-successor rule holds under concurrency, and an honest retry is never taken for an attack.
- Every certificate signed by the Agent CA is either registered or revoked.

### Negative

- The API keeps the tickets and the certificates it issued, and loads each CSR a second time.
- A thief of a key can cut its host off until it is enrolled again.
- The idempotent retry must be tested under concurrency, several times (CLAUDE.md, section 4): a badly bounded retry would become a second successor.

## References

- ADR-0017 (superseded; it holds the details of the rules tightened here), ADR-0021 (superseded), ADR-0026, ADR-0027, ADR-0029, ADR-0031.
- The sequence diagrams of the machine identity of 0.1 (`doc/analyse/uml/v0.1/identity.md`).
