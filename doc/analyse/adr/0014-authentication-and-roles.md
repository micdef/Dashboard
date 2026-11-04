# ADR-0014: Web authentication: local accounts and roles

| Field | Value |
|---|---|
| Status | Proposed |
| Date | 2026-11-03 |
| Decider | Michaël Defraene (product owner, architect) |
| Drafted by | Claude (main developer); the decisions had a second reading by the architecture review agent during scoping |
| Classification | Security-sensitive [SEC] |

## Context

The web site shows sensitive data about the infrastructure and can issue remote commands (ADR-0013). V1 needs several accounts with different rights. Data access goes only through the data gateway, without an ORM (ADR-0003), so the default Entity Framework stores of ASP.NET Core Identity cannot be used.

## Decision

- V1 uses **local accounts**, several of them, with **roles**.
- Authentication relies on **ASP.NET Core Identity** with **custom stores**: at least `IUserStore`, `IUserRoleStore` and `IUserPasswordStore`. The stores in the web site call a Backend gRPC service, which relays them to the account procedures through the data gateway (ADR-0004). The web site never reaches the data gateway itself.
- Proposed roles: **Reader**, **Operator**, **Admin**.

## Alternatives considered

- **Identity with its Entity Framework stores.** Rejected: contradicts ADR-0003.
- **A hand-written authentication system.** Rejected: password hashing, lockout and cookie handling are provided and reviewed in Identity; rewriting them adds risk.
- **External identity provider (LDAP/Active Directory, OpenID Connect).** Not in V1; may be considered later.

## Consequences

### Positive

- Password hashing and cookie authentication come from the framework.
- The account tables follow the same procedure-only contract as the rest of the database.

### Negative

- Each store interface is code to write and test. Roles may also need `IRoleStore` (**not verified**).
- The Backend receives the user and role from the web site: it trusts the web site process for identity (ADR-0004). It must re-check the role for every request that changes something.
- Without lockout and security stamp, a disabled or demoted user keeps a valid sign-in, and passwords can be guessed without limit.

## Open points

1. Identity with custom stores relayed through the Backend (the path was decided in ADR-0004 on 2026-11-04).
2. The three roles, and which roles may issue remote commands (proposal: Operator and Admin).
3. Additional store interfaces in V1. Proposal (raised in review): lockout (`IUserLockoutStore`) and security stamp (`IUserSecurityStampStore`, used to invalidate existing sign-ins after an account change) are mandatory if remote commands ship; two-factor authentication for Operator and Admin is considered.
4. Whether the web site is reachable only from the LAN and the VPN (raised in review).

## References

- Microsoft Learn, *Custom storage providers for ASP.NET Core Identity*.
- ADR-0003, ADR-0004, ADR-0013.
