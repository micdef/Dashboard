# ADR-0004: gRPC everywhere, mTLS for agents, Unix sockets inside the host

| Field | Value |
|---|---|
| Status | Accepted |
| Date | 2026-11-03 (proposed), 2026-11-04 (accepted), 2026-11-04 (amended after ADR-0005) |
| Decider | Michaël Defraene (product owner, architect) |
| Drafted by | Claude (main developer); the decisions had a second reading by the architecture review agent during scoping |

## Context

The system has the following links:

| Link | Network | Purpose |
|---|---|---|
| agent → Collector | across the network | logs, heartbeats, inventory; commands in both directions (ADR-0020) |
| web site → API | inside the "app" VM | everything the user interface shows or requests, including the account operations of ASP.NET Core Identity (ADR-0019) |
| API → Collector | inside the "app" VM | notification of a new command, identifier only (ADR-0005) |
| Collector → data gateway (`ingest`), API → data gateway (`manage`) | inside the "app" VM | all data access (ADR-0003) |
| Backend → AI node | across the network | LLM report generation (ADR-0009) |
| Backend and AI node → NAS | across the network | archive, prepared training data, exported models (ADR-0012) |

Supervised machines must not expose any inbound port for the system.

ADR-0021 later adds two links that follow the rules of this ADR: agent → Enrolment service (enrolment and renewal, opened by the agent) and Enrolment service → API (a Unix domain socket reserved for it). See also ADR-0016.

## Decision

- **gRPC on every link between Dashboard components**, including between the web site and the Backend. Two links use third-party protocols and are exceptions: the LLM server's own HTTP API (Backend → AI node) and the NAS file protocol (decided in ADR-0012).
- **Agent to Collector**: gRPC over HTTP/2 with mutual TLS (mTLS, ADR-0021). The connection is **always opened by the agent**; no inbound port is opened on the supervised machines. Commands travel on a bidirectional stream opened by the agent.
- **Links inside the "app" VM**: gRPC over **Unix domain sockets**, protected by file permissions. Kestrel does not set these permissions: they come from the operating system (systemd `RuntimeDirectory=` with a restrictive mode, `UMask=`, group ownership). A stale socket file from a previous run is removed at start.
- **Strict hierarchy: the web site talks only to the Backend (its API process), and only the Backend talks to the data gateway.** Account operations (ASP.NET Core Identity stores) go through an API service, which relays them to the data gateway. The Backend is the single tier where callers are authenticated and authorised before any data access.
- The data gateway exposes **two sockets**, owned by different groups: `ingest` (used by the Collector) and `manage` (used by the API). Since the Backend runs as two processes under different users (ADR-0005), each process can only open its own socket.
- **Contracts**: one folder of `.proto` files, compiled to C# with `Grpc.Tools`. The contracts used by agents (`Contracts.Edge`) are separated from the internal ones (`Contracts.Internal`), so an agent never ships internal messages.
- **Robustness of every link**:
  - every gRPC call has a **deadline**;
  - each service exposes a **health check**;
  - the Collector **acknowledges an agent batch only after the data gateway has stored it**; otherwise it returns a retryable error, and the agent keeps the data.

## Alternatives considered

- **REST/JSON between the web site and the Backend.** Rejected: one protocol and one contract format everywhere is simpler to review; the web site needs no public API.
- **TCP on localhost for internal links.** Rejected: it uses ports, is reachable by any local process, and cannot rely on file permissions. Microsoft documents Unix domain sockets as more efficient than TCP on the same machine.
- **A dedicated data gateway socket for the web site's accounts.** Rejected: it would break the hierarchy (web site → Backend → data gateway) and spread the handling of connections and authorisations over two places; with a single path, every request of the web site goes through the same checks in the Backend.
- **One data gateway socket for the whole Backend.** Rejected: with two Backend processes (ADR-0005), separate sockets give each process only the rights it needs.
- **SSH or an inbound agent port for commands.** Rejected: see ADR-0020.

## Consequences

### Positive

- Strongly typed contracts shared by all .NET components.
- Internal links are invisible from the network and restricted by Unix permissions.
- Supervised machines expose no listening port for Dashboard.
- The web site has no route to the database other than through the Backend.
- A failure of the data gateway does not lose agent data silently: the agent keeps it until the Backend can store it.

### Negative

- The API trusts the web site for the identity and the role of the user who makes a request (see ADR-0019): a compromised web site process could act as any user.
- The API relays account records, password hashes included, between the web site and the data gateway; they must never be logged.
- A gRPC channel configured for Unix domain sockets (through `SocketsHttpHandler.ConnectCallback`) supports neither client-side load balancing nor connectivity state tracking. Neither is needed on a single VM.
- Browsers cannot call gRPC services directly (without gRPC-Web); this is not a problem with Blazor Interactive Server, whose calls to the Backend run on the server.
- The LLM HTTP API has no built-in authentication (**not verified** for the chosen server); its port must be reachable only from the Backend, by firewall.
- An agent that keeps data while the Collector refuses it needs a bounded local buffer (ADR-0006).

## References

- Microsoft Learn, *Inter-process communication with gRPC* and *Inter-process communication with gRPC and Unix domain sockets* (consulted 2026-11-03).
- Microsoft Learn, *Security considerations for Kestrel* (Unix socket permissions, consulted by the security review, 2026-11-03).
- ADR-0003, ADR-0005, ADR-0006, ADR-0009, ADR-0012, ADR-0020, ADR-0019, ADR-0016, ADR-0021.
