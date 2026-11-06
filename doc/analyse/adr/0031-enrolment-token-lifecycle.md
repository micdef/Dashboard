# ADR-0031: Enrolment token lifecycle

| Field | Value |
|---|---|
| Status | Accepted |
| Date | 2026-11-06 |
| Decider | Michaël Defraene (product owner, architect) |
| Drafted by | Claude (main developer), after the team review of the sequence diagrams of 0.1; second reading by the architecture review agent before the decisions |
| Classification | Security-sensitive [SEC] |
| Complements | ADR-0025, ADR-0026, ADR-0027 |

## Context

An enrolment token is created by the host administrator (ADR-0021) or, from 0.2, by an Admin in the web site (ADR-0026); only its SHA-256 hash is stored, it is compared in constant time, its lifetime is 15 to 60 minutes, and it is bound to an expected `HostId` (ADR-0017). A token can be created only for a new `HostId` or for a host in the revoked state, never for a retired one (ADR-0025, ADR-0026). The audit trail records the identifier of a token, never its value nor its hash (ADR-0026).

The team review of the sequence diagrams of 0.1 found that these rules leave gaps:

- several valid tokens can exist for one `HostId`, and consuming a token does not check the host's state again: a second machine could enrol on top of an active certificate, which ADR-0025 and ADR-0026 forbid;
- a lookup by hash in the database is not a comparison in constant time, and no token identifier distinct from its value and its hash is defined;
- a `HostId` whose token was consumed but which never received a certificate is neither new nor revoked, so no new token can be created for it;
- the creation of a token and the refused uses of a token are recorded nowhere, while each issuance record names "the administrator who created the token" (ADR-0017);
- the agent reads the token from a file readable by root only, but runs as its own user; when the file is deleted is not settled, and a path opened by root can be a symbolic link;
- the Enrolment service terminates the TLS of the enrolment channel, so it sees each token in clear;
- the identity lifetime of 1 year requires a new token (ADR-0017), and a token for an existing host requires its revocation first (ADR-0026), which raises a critical alert (ADR-0025).

This ADR adds to the ADRs it complements, and tightens two rules: the storage of the token kept by ADR-0027 from ADR-0017, and the creation of a token for an existing host (ADR-0026). It weakens none; the two points where a reader could see a weakening are argued below.

## Decision

### Token format and storage

- A token is made of a **selector** and a **verifier**, both from `RandomNumberGenerator`, the verifier of at least 128 bits. The agent receives them as one value.
- The database stores the selector, the SHA-256 hash of the verifier, the `HostId`, the kind (agent; monitor from 0.2), the expiry and the state. The API finds the token by its selector, then compares the hash of the verifier with `CryptographicOperations.FixedTimeEquals`, which makes the comparison in constant time exact.
- The token **identifier** recorded in the issuance register and the audit trail (ADR-0026) is the row identifier, neither the selector nor any hash.
- This tightens the storage of ADR-0017 kept by ADR-0027. It does not weaken it: the selector is stored in clear, but it cannot enrol a machine without the verifier, of which only the hash is stored.

### One live token per `HostId`, checked when it is used

- Creating a token **invalidates every earlier unused token** of the same `HostId`, and the invalidation is recorded.
- Creating a token is one atomic operation of the data gateway: check the state of the `HostId`, invalidate the earlier tokens, insert the new one.
- Consuming a token is one atomic operation of the data gateway (ADR-0017), which also checks the state of the host **again**: it is refused unless the host is new, pending or revoked, not retired, and has no active mapping. It returns the `HostId` and the kind.
- A `HostId` is **pending** from the creation of its first token until its first certificate is mapped. A new token can be created for a pending host. This does not weaken ADR-0026: the risk that its rule prevents, a rogue machine taking the place of a host whose agent stopped, needs a host that held a certificate, which a pending host never did.

### Records

- From **0.1**, the issuance register records the creation of each token: its identifier, the `HostId`, the kind, the expiry and the host account that ran the command. Reading the host account from the peer credentials of the host channel's socket is **not verified** in .NET; if it is not available, the record says "member of the host channel's group". This lets each issuance record name who created its token (ADR-0017).
- Each **refused** use of a token is recorded with its reason (unknown, used, expired or invalidated, host state) and its source address. As for retired hosts (ADR-0026), the records are aggregated and rate-limited, because the enrolment endpoint is unauthenticated. A token that is **used** again, expired or invalidated, with a known selector, is a suspect event: shown on the development status page in 0.1, alerted from 0.2. An unknown selector is only counted.
- The source address comes from the Enrolment service, which can state a false one (ADR-0030).

### Delivery to the machine

- The machine is enrolled by a **one-shot enrolment command, run as root**. It reads the token from standard input first; a file is the fallback.
- A token file is opened without following symbolic links, and used only if it is a regular file owned by root with mode `0400` or `0600`, in a directory owned by root that others cannot write. It is read through the open descriptor. .NET reads the mode of an open file (`File.GetUnixFileMode(SafeFileHandle)`, Microsoft Learn, net-10.0); whether it offers an open that does not follow links, and the owner of an open file, is **not verified**. If it does not, the check and the opening are two steps, which leaves a race to be stated in the code.
- The command generates the key pair and writes the private key, readable by the agent's user only, before it sends the request. The token is kept in its memory only. The token file is **deleted once the token has been sent** to the Enrolment service: before that, a failure (TLS, revocation list) does not cost the token; after it, the token may have been consumed. Retries are bounded by the configuration.
- Once the certificate is received, the command hands the key and the certificate to the agent's user, and the agent runs under that user.

### What a compromised Enrolment service can do at first enrolment

The Enrolment service sees the token in clear. A compromised Enrolment service can consume a token that passes through it with its own key, and so enrol one machine per token in place of the real one. The theft is visible: the real machine is refused, its token is "used", which is a suspect event, and its host stays without a working agent. The issuance ticket of ADR-0030 prevents it from changing the key **after** the token is consumed.

### The identity lifetime of 1 year

- The identity lifetime of ADR-0017 (1 year) is kept. Its age is counted from the first enrolment of the chain of renewals.
- At the end of it, the host administrator revokes the host, then creates a new token (ADR-0026); the revocation raises its critical alert (ADR-0025). The host page shows the end of the identity in advance, and a yearly procedure is written in `doc/adm/` with the 0.1 work.
- A host whose certificate expired, for instance after more than 10 days offline, follows the same path: revocation first (ADR-0026).
- Removing the limit, or an exception for it, would weaken ADR-0026 or the rule kept by ADR-0027, and would need an ADR that supersedes; it is recorded in the [future improvements](../future-improvements.md) for a larger fleet.

## Alternatives considered

- **The agent sends a keyed hash (HMAC) of its CSR under the verifier, so that the Enrolment service never sees the verifier.** Rejected: to check it, the API would need the verifier or a key derived from it, and a read of the database would give a usable credential; that weakens "only its hash is stored".
- **A lookup by the hash of the whole token.** Rejected: it is not a comparison in constant time.
- **Several live tokens per `HostId`.** Rejected: a second enrolment could land on an active certificate.
- **No pending state.** Rejected: an enrolment that failed after the token was consumed would need a new `HostId`.
- **Deleting the token file as soon as it is read, or after the answer.** Rejected: the first wastes a token on a failure that happens before it is sent; the second turns an honest retry into a "used" suspect event.
- **Removing the 1-year limit.** Not adopted for V1 (see above).

## Consequences

### Positive

- A token can enrol one machine, once, and only on a host that may receive one at the moment it is used.
- Every token leaves a trace from its creation, and a stolen token that is used shows up.
- The token file cannot be swapped for a link, and is not wasted by an early failure.

### Negative

- One more state (pending) to test in the mapping and the token rules.
- Each host is revoked and enrolled again once a year, with a critical alert.
- A compromised Enrolment service can still steal the enrolment of the machines whose tokens pass through it.

## References

- ADR-0017 (superseded; it holds the details of the token rules), ADR-0021 (superseded), ADR-0022, ADR-0025, ADR-0026, ADR-0027, ADR-0030.
- Microsoft Learn (net-10.0), consulted on 2026-11-06: `CryptographicOperations.FixedTimeEquals` ("an amount of time that depends on the length of the sequences, but not their values"), `File.GetUnixFileMode(SafeFileHandle)`.
- The sequence diagrams of the machine identity of 0.1 (`doc/analyse/uml/v0.1/identity.md`).
