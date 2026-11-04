# ADR-0014: Web authentication: local accounts and roles

| Field | Value |
|---|---|
| Status | Accepted |
| Date | 2026-11-04 |
| Decider | Michaël Defraene (product owner, architect) |
| Drafted by | Claude (main developer); the decisions had a second reading by the architecture review agent during scoping |
| Classification | Security-sensitive [SEC] |

## Context

The web site shows sensitive data about the infrastructure and can issue remote commands (ADR-0013). V1 needs several accounts with different rights. Data access goes only through the VBE (data gateway), without an ORM (ADR-0003), so the default Entity Framework stores of ASP.NET Core Identity cannot be used. The project is meant to be installed by other people, in environments with or without a VPN, a reverse proxy or a second factor available.

## Decision

### Identity with custom stores

- V1 uses **local accounts**, several of them, with **roles**.
- Authentication relies on **ASP.NET Core Identity** with **custom stores**. The stores in the web site call a gRPC service of the API process (ADR-0005), which relays them to the account procedures through the VBE (ADR-0004). The web site never reaches the VBE itself.
- Store interfaces in V1: `IUserStore`, `IUserPasswordStore`, `IUserRoleStore`, `IRoleStore` (needed by `RoleManager`), `IUserLockoutStore`, `IUserSecurityStampStore`, and the interfaces needed for two-factor authentication with an authenticator application (`IUserTwoFactorStore` and the authenticator key and recovery code stores).

### Roles

| Role | Rights |
|---|---|
| Reader | sees everything, changes nothing |
| Operator | Reader, plus issuing the commands of the catalogue (ADR-0013) |
| Admin | Operator, plus accounts, roles and configuration (including the audit retention of ADR-0013) |

- For **every request that changes something**, the API reads the user's current role from the database (through the VBE). It never relies on a role sent by the web site.

### Account protection

- **Lockout** after repeated failed sign-ins, and **security stamp** (existing sign-ins are invalidated after a change of password, role or account state): mandatory in V1, whether or not remote commands ship.
- **Two-factor authentication (TOTP**, with an authenticator application) is available. Its use is a policy **chosen by the administrator at installation**, and changeable later by an Admin:
  - **disabled**: password only, for every account;
  - **per account**: each account may enable it;
  - **required for Operator and Admin**.
- The installation command asks for the policy explicitly; there is no silent default. When an Operator or Admin account has no second factor, the dashboard shows it as a warning.
- The sign-in endpoint is **rate limited** (ASP.NET Core rate limiting middleware), in addition to the lockout.

### First Admin

- The first Admin account is created by a **one-time installation command** on the API host, usable only while no Admin exists. The password is entered at that moment. The command refuses to run once an Admin exists, and the creation is written to the audit trail (ADR-0013).
- There is **no default account and no default password**.

### Exposure

- The web site listens on the **local network** only and always serves **HTTPS** itself.
- Publishing it through a **reverse proxy** is the choice of the system or network administrator. The reverse proxy must serve HTTPS.
- Forwarded headers (`X-Forwarded-For`, `X-Forwarded-Proto`) are accepted only from the proxy addresses set in the configuration (`KnownProxies` or `KnownIPNetworks`), with `ForwardLimit` left at 1. Without a configured proxy, they are ignored.
- The sign-in cookie is `Secure`, `HttpOnly` and `SameSite=Strict`.

## Alternatives considered

- **Identity with its Entity Framework stores.** Rejected: contradicts ADR-0003.
- **A hand-written authentication system.** Rejected: password hashing, lockout and cookie handling are provided and reviewed in Identity; rewriting them adds risk.
- **External identity provider (LDAP/Active Directory, OpenID Connect).** Not in V1; may be considered later.
- **Commands reserved to the Admin role.** Rejected: only two useful roles would remain in V1.
- **Lockout and security stamp only if remote commands ship.** Rejected: they cost little, and without them a disabled or demoted user keeps a valid sign-in and passwords can be guessed without limit.
- **Two-factor authentication always required for Operator and Admin.** Proposed by the main developer; not adopted by the product owner, so that an installation can run with passwords only. It remains one of the policies.
- **Site reachable only from the LAN and the VPN.** Not adopted as a rule: the administrator may publish it through a reverse proxy (planned for the training demonstration).
- **First account created through the web site becomes Admin.** Rejected: whoever reaches the site first becomes Admin.

## Consequences

### Positive

- Password hashing, lockout, security stamp, two-factor authentication and cookie authentication come from the framework.
- The account tables follow the same procedure-only contract as the rest of the database.
- A compromised web site process cannot grant itself the Admin role: the API decides from the database.
- The same build works on a LAN, behind a VPN or behind a reverse proxy; only the configuration changes.

### Negative

- Each store interface is code to write and test.
- Reading the role for each changing request costs one call through the VBE; the volume of such requests is small.
- With the "disabled" policy and a site published through a reverse proxy, a stolen password is enough to issue commands on the whole fleet. The installation documentation states this, and the dashboard shows the warning.
- A wrong proxy configuration either loses the client address (lockout and audit then see the proxy's address) or, if the trusted list is too wide, allows the client address to be spoofed. The administration guide documents the settings.
- The installation command must be protected like any administration tool: it runs on the API host, with the rights of an administrator of that host.

## Open points

None.

## References

- Microsoft Learn, *Custom storage providers for ASP.NET Core Identity* (consulted 2026-11-04).
- Microsoft Learn, *Configure ASP.NET Core to work with proxy servers and load balancers* and *Security considerations for the ASP.NET Core Kestrel web server* (consulted 2026-11-04).
- ADR-0003, ADR-0004, ADR-0005, ADR-0013.
