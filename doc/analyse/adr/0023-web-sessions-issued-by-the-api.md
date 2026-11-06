# ADR-0023: Web sessions issued by the API, and a host channel for security settings

| Field | Value |
|---|---|
| Status | Accepted |
| Date | 2026-11-05 |
| Decider | Michaël Defraene (product owner, architect) |
| Drafted by | Claude (main developer), after the third reading of 2026-11-05 |
| Classification | Security-sensitive [SEC] |
| Supersedes | ADR-0019 |

## Context

ADR-0019 moved the verification of passwords and second-factor codes into the API, so that the web site no longer holds any stored secret. The third reading (finding T-B1) showed that the API still trusts the web site for **who** the user is: the API rereads the role and the account state of the user that the web site names, but nothing binds a request to a sign-in that the API itself verified. A compromised web site could therefore read everything any active account can see, Admin included, and make every change that needs no step-up. ADR-0019 stated the opposite in its consequences ("nor act as a user who is not signed in"), which was wrong, as ADR-0004 already said.

The same reading found that the "command on the API host" used to weaken security settings has no defined channel, so it could end up on the socket that the web site uses (T-I1); that the step-up list misses several sensitive actions and that a proof can be replayed for 5 minutes (T-I7); that the rate limit lives only in the web site (T-M3); and that two revalidation settings, not one, must be changed (T-M2).

This ADR keeps every decision of ADR-0019 except where it says otherwise.

## Decision

### Kept from ADR-0019

- Roles **Reader**, **Operator** and **Admin**; lockout and security stamp mandatory; the two-factor policy chosen at installation; the first Admin created by a one-time host command; the web site on the local network over HTTPS, a reverse proxy being the administrator's choice declared to the site; the cookie settings.
- **The API alone verifies secrets**: it hashes and compares passwords, verifies second-factor codes and recovery codes, counts failures and applies the lockout. The web site never receives a **stored** secret (password hash, authenticator key, recovery code hash). An authenticator key and new recovery codes are shown **once**, when they are generated, and never returned again.
- Authenticator keys are stored encrypted with a key held outside the database; recovery codes are stored hashed.
- Every change of a security setting is audited (ADR-0024).

### Sessions are issued and checked by the API

- When the API has verified the password, and the second factor if the account has one, it opens a **session**: a random 256-bit identifier (`RandomNumberGenerator`), of which the database stores only the SHA-256 hash, with the user, the creation time, an idle expiry, an absolute expiry and the security stamp of the account at that time.
- The identifier is returned **once** to the web site, which keeps it in the protected sign-in cookie, and sends it with **every** call to the API, reads included.
- On every call, the API checks that the session exists, has not expired and has not been revoked, that the account is enabled and not locked out, and that its security stamp is unchanged. For a request that changes something, it rereads all of this from the database (through the VBE); for a read, it may use a cached result for at most one minute.
- The API **revokes** the session at sign-out, and revokes every session of an account when the account is disabled or locked out, or when its security stamp changes (password, second factor, role).
- So the web site can act only for users whose session is open, never as any other user.

### Step-up for sensitive actions

- The user enters their password again, and their second-factor code if they have one, before: issuing a command; creating, changing, disabling or unlocking an account, or resetting its second factor; changing a role; changing a security setting upwards; creating an enrolment token; revoking a machine or deleting a host; silencing an alert.
- The proof is kept **by the API** and bound to the API session. For issuing a command and for account and role changes it is **single use**; for the other actions it is valid for **5 minutes**.
- The API refuses a second-factor code already accepted for the same account in the same time step. It records the last accepted time step itself; whether ASP.NET Core Identity already does so is **not verified**, and the API does not rely on it.

### Rate limiting in the API

- The API limits the attempts to verify a password or a second-factor code, **globally and per account**, in addition to the rate limit of the sign-in page. A compromised web site cannot bypass it to spray passwords below the lockout threshold.

### Host channel

- The commands run on the API host (create the first Admin, lower the two-factor policy, lower the audit retention, create an enrolment token, revoke a machine, acknowledge an audit incident, break-glass unlock of the last Admin) go through a **separate gRPC service of the API, on its own Unix socket**, owned by root, with a group reserved to the host administrator (mode `0660`). The web site user is not in that group, and this service is not exposed on the socket that the web site uses.
- These commands are audited like any other change, with the host account that ran them.
- The **break-glass** command unlocks one Admin account and resets its second factor when no Admin can sign in; it raises a critical alert. It is available from 0.2.

### Revalidation in the web site

- Both intervals are set to **one minute**: `SecurityStampValidatorOptions.ValidationInterval` (the sign-in cookie) and the `RevalidationInterval` of the revalidating authentication state provider (Blazor circuits). Microsoft Learn gives 30 minutes as the default for both, and notes that the revalidation of a circuit does not change the cookie.
- This is a second line: the real check is the API session, on every call.

### Overridden Identity methods

- Every method of the web site's `UserManager` that compares a secret or changes the failure counter calls the API instead. The review names `CheckPasswordAsync`, `AccessFailedAsync`, `VerifyTwoFactorTokenAsync` and `RedeemTwoFactorRecoveryCodeAsync` (the list called by `SignInManager` in .NET 10 is **not verified**); the exact list is fixed and tested at coding time.

### Web site keys

- The Data Protection key ring is persisted in a directory owned by the web site user (mode `0700`), protected with a certificate (`PersistKeysToFileSystem`, `ProtectKeysWithCertificate`), with a fixed application name (`SetApplicationName`). Microsoft Learn states that with an explicit location the automatic encryption at rest is disabled, hence the certificate.
- The private key of that certificate is given to the service as a **systemd credential** owned by root. This protects copies of the key ring directory (in a backup, for instance), not a compromise of the running web site.
- The key ring is not backed up: losing it only signs every user out.

## Alternatives considered

- **Keep ADR-0019: the API rereads the role of the user named by the web site.** Rejected: the web site can name anyone.
- **Move the whole sign-in to the API, the web site only forwarding the cookie.** Not adopted, as in ADR-0019: Identity keeps the pages and the cookie in the web site; only the session identifier needed to be the API's.
- **A token signed by the API (JWT) instead of a stored session.** Not adopted: it cannot be revoked before its expiry without a store anyway, and the store is one table.
- **A step-up proof valid 5 minutes for every action.** Rejected for commands and account changes: a compromised web site could replay it.
- **Host commands on the API socket used by the web site, checked by caller.** Rejected: on a single SQL login and one socket, the API could not tell the host administrator from the web site.

## Consequences

### Positive

- A compromised web site can no longer read stored secrets, nor act as a user whose session is not open.
- A disabled, locked-out or demoted user loses every right at once, reads included within one minute.
- The protections cannot be weakened without access to the API host, through a channel the web site cannot reach.
- Password spraying is limited by the API, whatever the web site does.

### Negative

- The damage of a compromised web site is reduced, not removed: it can capture the passwords and codes that users type, use the sessions that are open, and read the enrolment tokens it displays (ADR-0025).
- One table of sessions, one check per call, and one database read per changing request.
- `UserManager` customisation, the session store and the host channel are security-sensitive code, with their own tests (lockout, replay of codes and proofs, revocation, timing).
- The key that encrypts the authenticator keys must be kept and backed up outside the database (ADR-0022).
- Lowering a security setting, and unlocking the last Admin, need access to the API host.

## References

- Microsoft Learn, *ASP.NET Core Blazor authentication and authorization*, section on server-side authentication (revalidation interval, circuit and cookie), consulted by the third reading on 2026-11-05.
- Microsoft Learn, `SecurityStampValidatorOptions.ValidationInterval` (default 30 minutes), consulted by the third reading on 2026-11-05.
- Microsoft Learn, *Key storage providers in ASP.NET Core* and *Data Protection default settings*, consulted by the third reading on 2026-11-05.
- Microsoft Learn, `UserManager<TUser>.CheckPasswordAsync` (virtual), consulted by the third reading on 2026-11-05.
- [Third reading report](../reviews/2026-11-05-third-reading.md), findings T-B1, T-I1, T-I7, T-M1 to T-M3.
- ADR-0003, ADR-0004, ADR-0005, ADR-0019 (superseded), ADR-0022, ADR-0024, ADR-0025; ADR-0026 complements this ADR, and tightens the step-up proof for revoking a machine and retiring a host, and the second-factor warning of ADR-0019.
