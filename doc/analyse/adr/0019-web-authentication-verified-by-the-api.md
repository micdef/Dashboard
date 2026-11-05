# ADR-0019: Web authentication verified by the API, with step-up for sensitive actions

| Field | Value |
|---|---|
| Status | Accepted |
| Date | 2026-11-05 |
| Decider | Michaël Defraene (product owner, architect) |
| Drafted by | Claude (main developer), after the second reading of 2026-11-04 |
| Classification | Security-sensitive [SEC] |
| Supersedes | ADR-0014 |

## Context

ADR-0014 relayed the ASP.NET Core Identity stores from the web site to the API, so the web site received account records, password hashes included, and later the authenticator keys. ADR-0004 already notes that the API trusts the web site for the identity of the user. The second reading (finding B1) showed the consequence: the web site is the most exposed process (it may be published through a reverse proxy), and compromising it would give the powers of every user, Admin included, and every password hash for offline cracking. ADR-0014 stated the opposite in its consequences, which was wrong.

This ADR keeps every decision of ADR-0014 except where it says otherwise, and moves the verification of secrets into the API.

## Decision

### Kept from ADR-0014

- Local accounts with the roles **Reader**, **Operator** (Reader, plus issuing commands of the catalogue, ADR-0020) and **Admin** (Operator, plus accounts, roles and configuration).
- Lockout and security stamp mandatory; two-factor authentication (TOTP) under a policy chosen by the administrator at installation (disabled, per account, required for Operator and Admin), with a warning on the dashboard when an Operator or Admin account has no second factor.
- Rate limiting of the sign-in endpoint.
- The first Admin is created by a one-time installation command on the API host; no default account or password.
- The web site listens on the local network over HTTPS; a reverse proxy, in HTTPS, is the administrator's choice, declared to the site in its configuration (`KnownProxies` or `KnownIPNetworks`, `ForwardLimit` 1). The sign-in cookie is `Secure`, `HttpOnly` and `SameSite=Strict`.

### Secrets are verified by the API only

- The web site keeps ASP.NET Core Identity for its pages and the sign-in cookie, but its `UserManager` does not compare secrets itself: checking a password (`CheckPasswordAsync`, which is virtual) and checking a second-factor code are calls to the API, which alone hashes, compares, counts failures and applies the lockout.
- The web site never receives a password hash, an authenticator key or a recovery code. The API's account services return no secret.
- When a user sets up an authenticator application, the API generates the key and returns it **once**, for display; it is never returned again.
- The API stores the authenticator keys **encrypted**, with a key held outside the database, and the recovery codes **hashed**.

### Step-up for sensitive actions

- Before issuing a command, changing an account or a role, or changing a security setting, the user enters their password again, and their second-factor code if they have one. The API verifies them.
- The proof is kept **by the API**, bound to the user and to the session, and is valid for **5 minutes**. The web site cannot create it.

### Immediate revocation

- For every request that changes something, the API reads the user's current role **and** account state (enabled, not locked out, security stamp unchanged) from the database, through the VBE. It never relies on what the web site sends.
- The web site revalidates the security stamp **every minute** (instead of the default 30 minutes), including in Blazor circuits.

### Security settings can only be weakened on the host

- Lowering the two-factor policy is done **only with the installation command on the API host**, never from the web site. Raising it is possible from the web site, after step-up.
- Every change of a security setting is written to the audit trail. The audit retention follows the same rule (ADR-0020).

### Web site keys

- The Data Protection key ring of the web site, which protects the sign-in cookie, is persisted in a directory owned by the web site user (mode `0700`) and protected with a certificate (`PersistKeysToFileSystem`, `ProtectKeysWithCertificate`). The second reading notes, citing Microsoft Learn, that without a profile the keys stay in memory and are lost at restart, and that with an explicit location their automatic encryption at rest is disabled.

## Alternatives considered

- **Keep relaying the stores, hashes included (ADR-0014).** Rejected: the most exposed process would hold what is needed to crack every password offline and to act as anyone.
- **Move the whole sign-in to the API, the web site only forwarding the cookie.** Not adopted: it would rewrite what Identity already provides in the web site (pages, cookie, two-factor flow) for little gain over verifying the secrets in the API.
- **No step-up.** Rejected: a stolen session or a compromised web site could issue commands with any open session.
- **Revalidation every 5 or 30 minutes.** Rejected: a disabled account would keep its session too long; the cost of one light read per minute and per session is small.
- **Weakening security settings from the web site after step-up.** Rejected: a stolen Admin session with its password would be enough to switch the protections off.

## Consequences

### Positive

- A compromised web site can no longer read the password hashes or the authenticator keys, nor act as a user who is not signed in.
- Sensitive actions need a fresh proof of the user's secrets, checked by the API.
- A disabled or demoted user loses every right to change something at once, and their session within a minute.
- The protections cannot be switched off without access to the API host.

### Negative

- The damage of a compromised web site is reduced, not removed: it can still capture the passwords that users type and use the sessions that are open, and replay a step-up proof within its 5 minutes for the same session.
- The authenticator key passes through the web site once, at setup, to be displayed.
- `UserManager` customisation and the API's verification services are security-sensitive code, with their own tests (lockout, timing, revalidation).
- The key that encrypts the authenticator keys must be kept and backed up outside the database.
- One read per minute and per session for the revalidation, and one read per changing request.
- Changing the two-factor policy downwards needs access to the API host.

## References

- Microsoft Learn, *Custom storage providers for ASP.NET Core Identity* (consulted 2026-11-04).
- Microsoft Learn, *Configure ASP.NET Core to work with proxy servers and load balancers* (consulted 2026-11-04).
- Microsoft Learn, *ASP.NET Core Blazor authentication and authorization* (revalidation interval) and *Key management in ASP.NET Core* (key ring storage), cited by the second reading; not re-read by the main developer.
- [Second reading report](../reviews/2026-11-04-second-reading.md), finding B1 and I7.
- ADR-0003, ADR-0004, ADR-0005, ADR-0020, ADR-0014 (superseded), ADR-0020.
