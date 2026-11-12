# ADR-0030: Certificate issuance bound by the API, and its records

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

- After signing, the API checks the issued certificate against a CSR that the Enrolment service itself supplies. At renewal, a compromised Enrolment service could forward the agent's genuine signed request, sign a certificate for its own key, and register it with its own CSR.
- Within the 20 days of the renewal cycle, it would then hold the identity of every machine, although ADR-0017 states that a compromised Enrolment service cannot take over a machine.
- Two renewals sent at the same time from the same certificate, by an agent and by the thief of its key, can both read "no successor yet": the one-successor rule, which detects a stolen key, would not fire. An honest agent that lost the answer to a successful renewal, and tries again, would be taken for a second renewal.
- "The host is marked suspect" has no defined effect.
- A certificate refused at registration stays validly signed by the Agent CA for 30 days.
- The API checks a list of properties of the issued certificate, not a complete profile.
- On the unauthenticated enrolment endpoint, the CSR is loaded, and its signature checked, before its size is bounded; the key allowlist has no upper bound for RSA.
- The Enrolment service renews its own server certificate itself, so that issuance is not recorded, although ADR-0017 requires every issuance to be recorded; and its socket to the API cannot carry a registration, since ADR-0017 limits it to "only the enrolment, renewal and revoked-serials services".

This ADR adds to ADR-0027. It **tightens** four rules that ADR-0027 keeps from ADR-0017 (the check of the issued certificate, its profile, the one-successor rule and the CSR policy), **defines** one (the effect of a suspect host), and adds new rules (the revocation of refused certificates, the order of the CSR intake, the record of the Enrolment service's server certificate). It weakens none. These decisions share one subject, what the API binds and records around each signature, and are kept in one file for that reason.

## Decision

### Issuance ticket

- Before any signature, the API issues an **issuance ticket**: single use, with a lifetime set in the configuration, stored through the data gateway, and bound to:
  - the `HostId` and the kind of identity (agent; monitor from 0.2);
  - the SHA-256 hash of the new public key (its `SubjectPublicKeyInfo`);
  - for a renewal, the hash of the predecessor certificate; for a first enrolment, the identifier of the consumed token (ADR-0031).
- At most **one open ticket** exists per `HostId`. Creating a token for the host, revoking it, or marking it suspect invalidates its open tickets.
- The API reads the new public key **itself**:
  - at renewal, from the CSR inside the request that the agent signed with its current key (ADR-0017), once it has verified that signature;
  - at first enrolment, from the CSR passed with the token, which the API loads with the same CSR policy as the Enrolment service. What a compromised Enrolment service can still do at first enrolment is stated in ADR-0031.
- The Enrolment service registers the certificate with `register(ticket, certificate)`, **never with a CSR**. Registration is one atomic operation of the data gateway, which checks **at that moment**:
  - the ticket is open and unexpired, and the public key matches the hash bound in the ticket;
  - for a renewal: the host is neither revoked, suspect nor retired, and the predecessor is still mapped and active;
  - for a first enrolment: the host is not retired, has no active mapping, and the token of the ticket is still the latest token of the host. A revoked host is accepted, since revocation then a new token is the way back of ADR-0026; once the certificate is registered, the host is active again and its suspect mark, if any, is lifted;
  - then the exact profile below.
- This tightens the check of the issued certificate in ADR-0017 ("public key equal to the CSR's"), kept by ADR-0027: the CSR that counts is the one the API read itself.
- If the registration does not succeed (the API cannot be reached, a deadline, a crash), the Enrolment service retries within bounds set in its configuration, and never after the expiry of the ticket, which the API returns with it. A ticket that expires with a certificate signed but not registered is a critical state (ADR-0032), logged with the serial. That certificate is never mapped, so the Collector refuses it (ADR-0027); the Enrolment service still submits it once the API answers again, so that it is refused and revoked (see "Refused certificates").

### Exact certificate profile

The API compares each issued agent certificate with one exact profile, not with a list of properties:

- exactly these extensions: basic constraints (`CA=false`), key usage (digital signature only), extended key usage (client authentication only), subject key identifier, authority key identifier, and the revocation list distribution point (ADR-0017); no subject alternative name;
- a serial that is unique in the issuance register;
- a signature with SHA-256 or stronger;
- issuer and authority key identifier equal to the **active** Agent CA (ADR-0027);
- the validity of ADR-0017 (30 days), within a tolerance set in the configuration.

This tightens the checks of ADR-0017 kept by ADR-0027.

### One successor, reserved atomically

- When the API accepts a renewal request, it reserves the successor of the predecessor certificate **in the same transaction** as the consumption of the nonce and the creation of the ticket, under a unique constraint on the predecessor. Two concurrent requests can therefore never both succeed. The reservation stays bound to the new public key.
- A request that comes again with the **same new public key** is a retry. It is a new signed request with a fresh nonce. The API answers with the certificate already issued, if any, which it keeps until that certificate expires; otherwise with a **new** ticket bound to the same key, the earlier one being invalidated. "Issued" means registered: a certificate signed but not registered is never handed back. The same holds for a retry of a first enrolment (ADR-0031).
- The agent writes its new key durably (written, flushed, then renamed) before it sends the request, so that it can retry. If it cannot write it, it does not send the request.
- A request with a **different public key** from the same predecessor is a second renewal: it is refused and the host is marked suspect (ADR-0017). An agent that lost its new key, for instance after a virtual machine snapshot or backup was restored between two renewals, is therefore marked suspect too: nothing tells it apart from a thief.
- This tightens the one-successor rule of ADR-0017 kept by ADR-0027.

### Suspect host

- A suspect host is refused **at renewal and at the Collector**, for every certificate mapped to it, until the host administrator revokes it explicitly and enrols it again (ADR-0026). The Collector sees the state at the next call, through the resolution of ADR-0029.
- Becoming suspect is a critical state (ADR-0032), alerted from 0.2 (ADR-0017 already alerts it).
- Nobody can know which holder of the key is the honest one: refusing both is the only safe choice. The thief of a key can therefore cut the host off, which is visible, and less harmful than taking its place.
- This defines the effect of "the host is marked suspect" of ADR-0017, kept by ADR-0027.

### Refused certificates

- A certificate refused at registration is recorded in the issuance register with its serial, and its serial is **revoked**, in the same atomic operation: it is on the next Agent CA revocation list, which the Enrolment service reads from the API (ADR-0029). The serial leaves the list once the certificate has expired (ADR-0029).
- A serial that belongs to an active registered certificate is never revoked this way: a refused certificate that reuses it is recorded as a suspect event instead.

### CSR intake

- On the enrolment endpoint (the listener of the enrolment channel that accepts a first enrolment without a client certificate), in this order: the rate limit per source address (ADR-0017), a bound on the size of the request (`MaxReceiveMessageSize`), a bound in bytes on the CSR, set in the configuration, and only then the loading of the CSR.
- The key allowlist becomes: ECDSA P-256 or P-384, or **RSA 3072 or 4096**. This tightens the CSR policy of ADR-0017 ("RSA of at least 3072 bits"), kept by ADR-0027.
- The Enrolment service applies this policy, and the API applies it again to every CSR it reads.

### The Enrolment service's server certificate

- The Enrolment service keeps renewing its own server certificate (ADR-0017), at a moment within its 90 days set in its configuration. Nothing is added to its socket to the API.
- The service writes the public certificate atomically (a temporary file, then a rename) to a file that the API's group can read; its key is in a directory that this group cannot read. Both services run on the "app" VM (ADR-0016).
- The API reads that file when it changes and at least once a day, after bounding its size. It checks its profile (server authentication, the DNS name of the Enrolment service, issued by the active Agent CA), records its serial, hash and expiry in the issuance register, and tracks its expiry.
- A file that is missing, unreadable, too large, invalid or with a wrong profile is logged as an error each time and leaves the state "unknown"; its expiry, or failed reads beyond a number set in the configuration, is a critical state (ADR-0032).
- **Limit**: this records what the file holds, not what the service serves. A compromised Enrolment service holds the Agent CA key and can serve a certificate it never wrote to the file.

### What a compromised Enrolment service can still do

With these rules, a compromised Enrolment service can still:

- refuse to sign, and so delay enrolments and renewals;
- sign certificates that it never submits to the API, which the Collector refuses because the mapping fails closed (ADR-0027);
- serve an unregistered server certificate on the enrolment channel;
- state a false source address, since the address that the API records comes from it;
- leave serials out of the revocation list it writes, which the mapping makes harmless for agent certificates (ADR-0029);
- misuse the tokens that pass through it (ADR-0031).

It can never pass as the Collector (ADR-0027).

## Alternatives considered

- **Keep the CSR in the registration call.** Rejected: the check would compare the certificate with what the checked party supplies.
- **Mark a suspect host on the dashboard only, or refuse it at renewal only.** Rejected: if the thief renewed first, it would keep a valid and mapped certificate.
- **Treat a renewal with a new key as a retry while no certificate is registered.** Rejected: it would reopen the race between the agent and the thief of its key.
- **Let a refused certificate expire on its own.** Rejected: it would stay usable for 30 days on the enrolment channel.
- **A fourth service on the socket of the Enrolment service, to register its server certificate.** Rejected: it widens a surface that ADR-0017 limits, which would need an ADR that supersedes.
- **No record of the Enrolment service's server certificate.** Rejected: every issuance is recorded (ADR-0017).

## Consequences

### Positive

- A compromised Enrolment service can no longer take over an enrolled machine at renewal.
- The one-successor rule holds under concurrency, and an honest retry with the same key is never taken for an attack.
- Every certificate submitted to the API is either registered or revoked.

### Negative

- The API keeps the tickets and the certificates it issued, and loads each CSR a second time.
- A thief of a key, or the restore of an older snapshot of a host, cuts the host off until it is enrolled again.
- The idempotent retry and the expiry of tickets must be tested under concurrency, several times (CLAUDE.md, section 4): a badly bounded retry would become a second successor.

## References

- ADR-0016, ADR-0017 (superseded; it holds the details of the rules tightened here), ADR-0021 (superseded), ADR-0026, ADR-0027, ADR-0029, ADR-0031, ADR-0032.
- [Review report of the 0.1 identity sequences](../reviews/2026-11-06-identity-sequences-of-0.1.md).
