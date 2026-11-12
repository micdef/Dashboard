# ADR-0031: Enrolment token lifecycle

| Field | Value |
|---|---|
| Status | Accepted |
| Date | 2026-11-06 |
| Decider | Michaël Defraene (product owner, architect) |
| Drafted by | Claude (main developer), after the team review of the sequence diagrams of 0.1; second reading by the architecture review agent before the decisions; the storage of the token amended after the team review of this ADR, by decision of the product owner |
| Classification | Security-sensitive [SEC] |
| Complements | ADR-0025, ADR-0026, ADR-0027 |

## Context

An enrolment token is created by the host administrator (ADR-0021) or, from 0.2, by an Admin in the web site (ADR-0026); only its SHA-256 hash is stored, it is compared in constant time, its lifetime is 15 to 60 minutes, and it is bound to an expected `HostId` (ADR-0017). A token can be created only for a new `HostId` or for a host in the revoked state, never for a retired one (ADR-0025, ADR-0026). The audit trail records the identifier of a token, never its value nor its hash (ADR-0026).

The team review of the sequence diagrams of 0.1 found that these rules leave gaps:

- several valid tokens can exist for one `HostId`, and consuming a token does not check the host's state again: a second machine could enrol on top of an active certificate, which ADR-0025 and ADR-0026 forbid;
- "new `HostId`" is not defined: a `HostId` whose token was consumed but which never received a certificate is neither clearly new nor revoked, so no new token could be created for it;
- the creation of a token and the refused uses of a token are recorded nowhere, while each issuance record names "the administrator who created the token" (ADR-0017);
- the agent reads the token from a file readable by root only, but runs as its own user; when the file is deleted is not settled, and a path opened by root can be a symbolic link;
- an honest retry after a lost answer would find its token already used;
- the Enrolment service terminates the TLS of the enrolment channel, so it sees each token in clear;
- the identity lifetime of 1 year requires a new token (ADR-0017), and a token for an existing host requires its revocation first (ADR-0026), which raises a critical alert (ADR-0025).

This ADR adds to the ADRs it complements. It **tightens** the creation and the consumption of a token (ADR-0026: one live token per host, the host state checked again when the token is used), and **defines** a term that ADR-0025 and ADR-0026 use without defining ("new `HostId`"). It weakens none.

## Decision

### Storage

- A token is a value of at least 128 bits from `RandomNumberGenerator` (ADR-0017). The database stores its SHA-256 hash, the `HostId`, the kind (agent; monitor from 0.2), the expiry and the state; never the value.
- The API looks the token up by its hash, then compares the stored hash with the computed one using `CryptographicOperations.FixedTimeEquals` (ADR-0017), as a defence in depth. The protection comes from the value itself: the index lookup can at most reveal the beginning of the hash of a random value of 128 bits, from which the value cannot be found (reasoning, not measured).
- The token **identifier** recorded in the issuance register and the audit trail (ADR-0026) is the row identifier, never the value nor the hash.

### One live token per `HostId`, checked when it is used

- A **new** `HostId` is one that never had a mapped certificate and is not retired. Such a host is shown as **pending** from the creation of its first token. A new token can therefore be created for a pending host, as for any new host (ADR-0025, ADR-0026). The risk that ADR-0026 prevents, a rogue machine taking the place of a host whose agent stopped, needs a host that held a certificate, which a new host never did.
- Creating a token is one atomic operation of the data gateway: check the state of the `HostId`, invalidate **every** earlier token of that `HostId`, used or not, with its right to a retry, and its open issuance tickets (ADR-0030), insert the new token, and record its creation.
- Consuming a token is one atomic operation of the data gateway (ADR-0017). It first checks whether the consumption is a retry (below); otherwise it checks the state of the host **again**: it is refused unless the host is new or revoked, not retired, has no active mapping and no open ticket. It returns the `HostId`, the kind and the issuance ticket (ADR-0030).
- A consumption that comes again with the **same token and the same public key** is a retry, accepted only while the token is unexpired and still the latest token of the host. The API answers with the certificate already issued (registered), if any, otherwise with a new ticket bound to the same key, the earlier ticket of this token being invalidated, and records no refusal. The same token with a different public key, or a retry of a token that is no longer the latest, is refused as "used".
- The host page shows how long a host has been pending; beyond a duration set in the configuration, a warning is logged. A pending host that is abandoned is revoked (it has no certificate to revoke, but the state change is recorded, and alerted from 0.2 as any revocation, ADR-0025), then retired (ADR-0026). Revoking a host also invalidates its unused tokens.

### Records

- From **0.1**, the issuance register records the creation of each token: its identifier, the `HostId`, the kind, the expiry and the host account that ran the command. Reading the host account from the peer credentials of the host channel's socket is **not verified** in .NET; if it cannot be read, the token is still created, and the record is marked "host account not read" in a field of its own. This lets each issuance record name who created its token (ADR-0017).
- Each **refused** use of a token is recorded with its reason (unknown, used, expired or invalidated, host state) and its source address. As for retired hosts (ADR-0026), these records are aggregated and rate-limited, because the enrolment endpoint is unauthenticated; the records dropped by the limit are counted.
- A token that is presented again once used, expired or invalidated is a **suspect event**: a critical state (ADR-0032), alerted from 0.2. One exception: a token presented again with the **same public key** as at its consumption, once it has expired or is no longer the latest, is an honest late retry; it is refused and recorded, but it is not a suspect event. The hash of the public key bound at the consumption is kept with the token's record, as long as the issuance register keeps it, so that this case can be recognised. An unknown token is only counted.
- The machine always receives the same generic refusal, so that the reasons cannot be used to probe the state of a token.
- The source address comes from the Enrolment service, which can state a false one (ADR-0030).

### Delivery to the machine

- The machine is enrolled by a **one-shot enrolment command, run as root**, shipped in the agent package. It reads the token from standard input first, and a file is the fallback; the token is never passed as a command-line argument nor as an environment variable. The procedure in `doc/adm/` has the host administrator type it at a prompt without echo, not through a shell pipe that would leave it in the history.
- A token file is opened without following symbolic links, and used only if it is a regular file owned by root with mode `0400` or `0600`, in a directory owned by root that others cannot write. It is read through the open descriptor. .NET reads the mode of an open file (`File.GetUnixFileMode(SafeFileHandle)`, Microsoft Learn, net-10.0). Whether it offers an open that does not follow links, and reads the owner of an open file, is **not verified**; if it does not, the check and the opening are two steps, which leaves a race to be stated in the code.
- The command refuses to run when the agent already holds a **valid** certificate: enrolling a working machine again goes through its revocation first (ADR-0026). A certificate is valid when it is unexpired, chains to the Agent CA anchor set, and its serial is not on the Agent CA revocation list that the command fetches and checks before the token is sent. The command therefore makes this check once it holds that list; if it cannot fetch a valid list, or cannot read the agent's certificate file, it stops. Once a host is revoked, its serial appears on the list within the short interval of ADR-0029, and the command can then run.
- The command generates the key pair and writes the private key durably to a **pending** path, readable by root only, together with the SHA-256 hash of the token it serves, before it sends the request. The token itself is kept in its memory only; its hash on disk, readable by root only, gives nothing usable, since the token is a random value of at least 128 bits. A run with the same token, after a stop, finds that hash and reuses the pending key, so that it is a retry; a run with a new token generates a new pending key. The key and the certificate of the agent are replaced only once the new certificate is received, so that a refused run never destroys a working key.
- The token file is **deleted once the token has been sent** to the Enrolment service: before that, a failure (TLS, revocation list) does not cost the token; after it, the token may have been consumed. This is how the command applies "it deletes the file after use" (ADR-0021). If the file cannot be deleted, the command stops with an error and a clear message.
- The command retries with the same token and the same key, within bounds set in the configuration, after a transport failure only (the Enrolment service cannot be reached, a deadline); a refusal stops it with the generic message. A retry after a lost answer gets the same answer (see above), while the token is unexpired. If the command stops before it has a certificate, the host is still pending, and the host administrator creates a new token for it; the command says so.
- Once the certificate is received, the command moves the pending key and the certificate to the agent's user, each by a rename, the key first; a key found without its certificate counts as no valid certificate. The agent runs under that user. If it cannot store them, it fails visibly, and a rerun with the same token gets the same certificate while the token is unexpired.
- This delivery is the one of the Linux agent. The Windows agent, in 0.4 (ADR-0015), has its delivery decided with it.

### What a compromised Enrolment service can do at first enrolment

The Enrolment service sees the token in clear. A compromised Enrolment service can consume a token that passes through it with its own key, and so enrol one machine per token in place of the real one. The theft is visible: the real machine's token is "used", which is a suspect event, and the host has no working agent. The issuance ticket of ADR-0030 prevents it from changing the key **after** the token is consumed.

### The identity lifetime of 1 year

- The identity lifetime of ADR-0017 (1 year) is kept. Its age is counted from the first enrolment of the chain of renewals.
- At the end of it, the host administrator revokes the host, then creates a new token (ADR-0026); the revocation raises its critical alert (ADR-0025). The host page shows the end of the identity in advance, and a yearly procedure is written in `doc/adm/` with the 0.1 work.
- A host whose certificate expired, for instance after more than 10 days offline, follows the same path: revocation first (ADR-0026).
- Removing the limit, or an exception for it, would weaken ADR-0026 or the rule kept by ADR-0027, and would need an ADR that supersedes; it is recorded in the [future improvements](../future-improvements.md) for a larger fleet.

## Alternatives considered

- **A token made of a selector, stored in clear, and a verifier, whose hash is stored.** Not adopted: it departs from "only its SHA-256 hash is stored" (ADR-0017), which a complement cannot do; the lookup by hash gives the same protection.
- **The agent sends a keyed hash (HMAC) of its CSR under a secret part of the token, so that the Enrolment service never sees it.** Rejected: to check it, the API would need that secret or a key derived from it, and a read of the database would give a usable credential.
- **Several live tokens per `HostId`.** Rejected: a second enrolment could land on an active certificate.
- **No pending state, a new `HostId` after each failed enrolment.** Rejected: it multiplies hosts that never worked.
- **Retiring an abandoned pending host directly.** Not adopted: ADR-0026 allows retiring only from the revoked state.
- **Deleting the token file as soon as it is read, or after the answer.** Rejected: the first wastes a token on a failure that happens before it is sent; the second keeps a consumed token on disk for no use.
- **Removing the 1-year limit.** Not adopted for V1 (see above).

## Consequences

### Positive

- A token can enrol one machine, once, and only on a host that may receive one at the moment it is used.
- Every token leaves a trace from its creation, and a stolen token that is used shows up.
- The token file cannot be swapped for a link, and is not wasted by an early failure.

### Negative

- One more displayed state (pending) to test in the mapping and the token rules.
- Each host is revoked and enrolled again once a year, with a critical alert.
- A compromised Enrolment service can still steal the enrolment of the machines whose tokens pass through it.

## References

- Microsoft Learn (net-10.0), consulted on 2026-11-06: `CryptographicOperations.FixedTimeEquals` ("an amount of time that depends on the length of the sequences, but not their values"), `File.GetUnixFileMode(SafeFileHandle)`.
- ADR-0015, ADR-0017 (superseded; it holds the details of the token rules), ADR-0021 (superseded), ADR-0025, ADR-0026, ADR-0027, ADR-0030, ADR-0032.
- [Review report of the 0.1 identity sequences](../reviews/2026-11-06-identity-sequences-of-0.1.md).
