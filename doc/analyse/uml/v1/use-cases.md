# Use cases of V1

Overview of what each actor can do with Dashboard in V1, drawn from the accepted ADRs. Each use case carries the increment that delivers it (ADR-0015, ADR-0025). The detail of each use case (its steps, its errors) is given by the sequence diagrams of its increment.

The diagrams are written in [Mermaid](https://mermaid.js.org/), which GitHub and Obsidian render in Markdown files. They use the flowchart syntax: Mermaid's dedicated use case diagram exists only from Mermaid 12.0 and is in beta, and the versions embedded by GitHub and Obsidian are **not verified**. Notation used here:

| Shape | Meaning |
|---|---|
| rectangle with «actor» | actor (a person or an external system) |
| rounded box | use case |
| box around use cases | the system boundary |
| solid arrow | the actor takes part in the use case |
| dotted arrow «include» | the use case always includes the other one |
| dotted arrow «inherits» | the role has every right of the other role |

## Actors

| Actor | Kind | Who or what | Source |
|---|---|---|---|
| Reader | person, web site | sees everything, changes nothing | ADR-0014, ADR-0023 |
| Operator | person, web site | Reader, plus issuing the commands of the catalogue | ADR-0014, ADR-0023 |
| Admin | person, web site | Operator, plus accounts, roles and configuration | ADR-0014, ADR-0023 |
| Host administrator | person, on the API host | runs the host commands through the host channel; installs the agents | ADR-0023, ADR-0025 |
| Key holder | person, offline | holds the Server CA key and the package signing keys, backs up the keys outside the database | ADR-0021, ADR-0022, ADR-0025 |
| Agent | external system, on each supervised machine | enrols, reports, executes catalogue commands | ADR-0016, ADR-0021, ADR-0024 |
| Verifier | external system, on another machine | the independent verification tool, enrolled with a "monitor" identity | ADR-0022, ADR-0024, ADR-0025 |
| Scheduled jobs | time | systemd timers on Linux, SQL Server Agent for the backups | ADR-0010, ADR-0012, ADR-0022 |

In the home lab, the host administrator and the key holder are the same person, the product owner. They are two actors because they hold different rights: the host channel on the API host, and keys that never reach that host.

## Web site users

```mermaid
flowchart LR
    Reader["«actor»<br/>Reader"]
    Operator["«actor»<br/>Operator"]
    Admin["«actor»<br/>Admin"]

    Operator -. «inherits» .-> Reader
    Admin -. «inherits» .-> Operator

    subgraph Dashboard["Dashboard (web site)"]
        SignIn(["Sign in with password and second factor (0.2)"])
        OwnFactor(["Set up one's own second factor (0.2)"])
        Fleet(["View the fleet status (0.1, development page; 0.2)"])
        Machine(["View a machine: inventory, logs, metrics (0.2; Windows 0.4)"])
        AiReport(["View anomalies and AI reports (0.3)"])
        RulesCve(["View rule hits and CVE exposure (0.5)"])
        Alerts(["View alerts (0.2)"])
        Audit(["View the audit trail (0.2)"])
        Command(["Issue a catalogue command (0.6)"])
        Accounts(["Manage accounts and roles (0.2)"])
        Token(["Create an enrolment token (0.2)"])
        Revoke(["Revoke a machine or delete a host (0.2)"])
        Settings(["Raise a security setting or the audit retention (0.2)"])
        Silence(["Silence an alert that is not critical (0.2)"])
        StepUp(["Confirm with step-up (0.2)"])
    end

    Reader --> SignIn
    Reader --> OwnFactor
    Reader --> Fleet
    Reader --> Machine
    Reader --> AiReport
    Reader --> RulesCve
    Reader --> Alerts
    Reader --> Audit
    Operator --> Command
    Admin --> Accounts
    Admin --> Token
    Admin --> Revoke
    Admin --> Settings
    Admin --> Silence

    Command -. «include» .-> StepUp
    Accounts -. «include» .-> StepUp
    Token -. «include» .-> StepUp
    Revoke -. «include» .-> StepUp
    Settings -. «include» .-> StepUp
    Silence -. «include» .-> StepUp
```

- The status page of 0.1 is for development only and is never deployed elsewhere (ADR-0015).
- Critical alerts cannot be silenced from the web site (ADR-0025).
- Lowering a security setting is not possible from the web site: it goes through the host channel (ADR-0023).

## Host administrator and key holder

```mermaid
flowchart LR
    HostAdmin["«actor»<br/>Host administrator"]
    KeyHolder["«actor»<br/>Key holder"]

    subgraph HostChannel["Dashboard (host channel of the API)"]
        FirstAdmin(["Create the first Admin (0.2)"])
        HostToken(["Create an enrolment token (0.1)"])
        HostRevoke(["Revoke a machine (0.1)"])
        Lower(["Lower the two-factor policy or the audit retention (0.2)"])
        Incident(["Acknowledge an audit incident (0.2)"])
        BreakGlass(["Unlock the last Admin, break-glass (0.2)"])
    end

    subgraph Host["On the hosts, outside the Dashboard processes"]
        AlertConfig(["Edit the alert configuration file (0.2)"])
        Install(["Install or update the agent from its signed package (RPM 0.1, MSI 0.4)"])
    end

    subgraph Offline["Offline"]
        SignServer(["Sign the Collector certificate and the Server CA revocation list (0.1)"])
        SignPackages(["Sign the agent packages (0.1)"])
        BackupKeys(["Back up the keys outside the database (0.1)"])
        RunVerifier(["Run the verifier on demand (0.2)"])
    end

    HostAdmin --> FirstAdmin
    HostAdmin --> HostToken
    HostAdmin --> HostRevoke
    HostAdmin --> Lower
    HostAdmin --> Incident
    HostAdmin --> BreakGlass
    HostAdmin --> AlertConfig
    HostAdmin --> Install
    KeyHolder --> SignServer
    KeyHolder --> SignPackages
    KeyHolder --> BackupKeys
    KeyHolder --> RunVerifier
```

- Every host command is audited with the host account that ran it (ADR-0023); a machine revoked and a break-glass unlock raise a critical alert (ADR-0025).
- The "Run the verifier" use case sits in "Offline" for want of a better place: the verifier runs on another machine than the API host, not necessarily offline (ADR-0024).

## Systems and scheduled jobs

```mermaid
flowchart LR
    Agent["«actor»<br/>Agent"]
    Verifier["«actor»<br/>Verifier"]
    Jobs["«actor»<br/>Scheduled jobs"]

    subgraph Dashboard["Dashboard (Collector, Enrolment service, Dashboard.Crl, AI node)"]
        Enrol(["Enrol with a token and a CSR (0.1)"])
        Renew(["Renew its certificate (0.1)"])
        Crl(["Fetch the revocation lists (0.1)"])
        Heartbeat(["Send a heartbeat (0.1)"])
        Telemetry(["Send logs and metrics (Linux 0.2, Windows 0.4)"])
        Execute(["Execute a catalogue command (0.6)"])
        Health(["Check the health of the Backend (0.2)"])
        Export(["Export to the cold storage, then purge (0.2)"])
        Backup(["Back up the database and test the restore (0.2)"])
        Train(["Train the anomaly model (0.3)"])
        Report(["Generate an AI report (0.3)"])
    end

    subgraph Outside["Outside Dashboard"]
        VerifyChain(["Verify the audit chain and write the receipts (0.2)"])
        OwnAlerts(["Send its own critical alerts (0.2)"])
    end

    Agent --> Enrol
    Agent --> Renew
    Agent --> Crl
    Agent --> Heartbeat
    Agent --> Telemetry
    Agent --> Execute
    Verifier --> Health
    Verifier --> VerifyChain
    Verifier --> OwnAlerts
    Jobs --> Export
    Jobs --> Backup
    Jobs --> Train
    Jobs --> Report
```

- The agent always connects to the Backend; Dashboard never opens a connection to a machine (ADR-0004, ADR-0024).
- The verifier never connects to the database; it reads the exported files and calls the health service of the Collector only (ADR-0024, ADR-0025).
- Agent updates never go through the Collector nor through remote commands (ADR-0025).

## Not decided, or not stated by an ADR

- **What triggers an AI report**: a schedule, a user request, or both. ADR-0009 says the API calls the LLM, not when. Drawn here as a scheduled job, to be confirmed.
- **What triggers the training**: drawn as a scheduled job, because ADR-0010 describes a training task that stops and restarts the LLM service; to be confirmed.
- **The increment of the inventory**: the agent collects an inventory (ADR-0016), shown on the machine page of 0.2; no ADR gives the increment of its collection.
