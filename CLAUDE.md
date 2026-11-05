# CLAUDE.md

Working rules for Claude Code on this repository. They complement the user-level instructions of the product owner; where both cover a point, this file is the project's own rule.

## Project

Dashboard supervises a small fleet of Linux and Windows servers: heartbeats, inventory, readable logs, CVE exposure, AI-based anomaly detection and reports, remote commands without SSH. It is the final project of the AI developer training at Technofutur TIC (V1 for the jury in March 2027), and a showcase project: every rule below applies at professional level, and none is relaxed to go faster.

- The architecture is recorded in the [ADRs](doc/analyse/adr/README.md). An accepted ADR is **never amended**: it is superseded by a new ADR. Only editorial fixes are made in place, and they are listed in the review report that asked for them.
- Version floors of the platforms and libraries: [doc/analyse/dependencies.md](doc/analyse/dependencies.md).
- Deferred ideas: [doc/analyse/future-improvements.md](doc/analyse/future-improvements.md).

## 1. Roles

- **Michaël Defraene** is the product owner and the architect. He checks everything and keeps the security-sensitive subjects; his own changes in that code are legitimate.
- **Claude** is the main developer, with the agent team (section 5). Claude leads the split of a version into tasks; architecture decisions stay with the product owner: Claude proposes, with a second reading by the architect agent, and he decides.
- **No code until he asks for it.** During the design phase we discuss architecture, analysis and scope, one phase at a time. Conclusions go into `doc/` only after his approval.
- **Human readability first.** No trick that trades clarity for a few nanoseconds. An optimisation is acceptable only if it is measured, needed and explained.

## 2. Language and documentation

- Discussion with the product owner is in French. **Everything that goes into Git is in English**: code, comments, documentation, commit messages, pull request descriptions.
- One bilingual `README.md` (English section, then French section), never two files.
- Layout: `src/`, `test/`, `doc/dev` (developer documentation; generated output is not versioned), `doc/analyse` (decisions, roadmap, reviews), `doc/user`, `doc/adm` (administration and installation). Visual examples provided by the product owner are in `doc/assets/example/`. There is no `docs/` folder.
- **A wrong comment or a wrong document is a defect**, like a bug: every sentence must match the real code and the real behaviour. What was not verified is marked "not verified".
- Comments describe **purpose and behaviour**, not the code.
- Documentation extractable by the language tools: XML `///` compatible with Doxygen (C#), docstrings (Python).
- No secret, no internal address, no exploitable detail in the repository: it is public.

## 3. Git

### Branches

- `main` holds published versions. `dev` (from `main`) is the last safeguard. `release/X.Y.Z` (from `dev`) holds the work of one version.
- `feature/…`, `fix/…`, `docs/…`, `chore/…`, `test/…` start from `release/X.Y.Z`. `hotfix/…` starts from `main`, then is carried into `dev` and the current version.
- Changes go up feature → release → dev → main, **always through a pull request**.
- Annotated tag `vX.Y.Z` on `main`, and a **GitHub pre-release for every 0.x.0 version** ("vX.Y.Z - <title>"), announced when it is created.
- Branch names are lowercase English words with hyphens.

### Commits and pull requests

- **Conventional Commits**, in English: imperative, lowercase, no final period, under 72 characters. The body says **what and why**, not how.
- Commit messages are written to a file, then committed with `git commit -F` (the shell mangles some characters).
- Keep the attribution lines (`Co-Authored-By`, "Generated with Claude Code").
- **Merge commits only**: never squash, never rebase. **No history rewriting** on a shared branch. Merged branches are deleted.
- SemVer; before 1.0.0: `0.x.y`, with `-alpha.N` / `-beta.N`.
- GitHub rulesets (to be set up by the product owner): pull request required, required checks, merge commit only, no force push and no deletion on `main` and `dev`, no bypass.

### Who merges

- Claude may merge pull requests into `release/X.Y.Z` under three conditions:
  1. a green CI, without bypass;
  2. the review of **the whole team** (section 5), with every finding fixed or justified in the pull request;
  3. a merge commit.
- **The product owner merges**, even into `release/`: every **[SEC]** pull request, every pull request that touches a safeguard (`.github/`, rulesets, `.editorconfig`, analyser rules) and his own work. He also merges release → dev, dev → main and hotfixes, and creates the tag and the pre-release.
- Agents never merge and never push.

### No overlapping pull requests

- Claude is the only one who opens pull requests, and **two open pull requests never touch the same file**.
- Before each `gh pr create`: compare `git diff --name-only origin/release/X.Y.Z...HEAD` with the files of the open pull requests (`gh pr view <n> --json files`). If a file is shared, wait for the other one to be merged.
- Files at risk: `CHANGELOG.md`, test READMEs, solution files, the central package versions file, the roadmap, `CONTRIBUTING.md`.
- Chains of pull requests (each branch containing the previous one) are **not used** on this project until the product owner approves them.

### Changelog

- [Keep a Changelog](https://keepachangelog.com/). **One fragment per pull request** in `changelog/unreleased/<branch-with-hyphens>.md`; a pull request never touches `CHANGELOG.md`.
- The `docs/release-X.Y.Z` pull request gathers the fragments (with a script) and finalises the version. The CI refuses any other pull request that changes `CHANGELOG.md`.

### CI

- Build and tests on every pull request to `main`, `dev` and `release/**`, plus a safeguard that fails if a test project is not run.
- **One required job runs on every pull request, without path filters**, otherwise it blocks the merge.
- Claude never polls the CI: the app reports failures.
- No CI exists yet (no project to build); until it does, condition 1 of "Who merges" cannot be met.

## 4. Tests

- **Every new behaviour is proven by a test**, and tests must be able to fail: at least one **mutation** of the production code (break a check, see the test go red, restore the code), reported in the pull request.
- **Never a fixed delay** to wait: waits are driven by events. The only exception is justified by a comment in the test. Time goes through an **injectable clock** (`TimeProvider` / `FakeTimeProvider`), never real seconds.
- Fuzzing and property tests use **fixed seeds**, printed in every failure message so that the failure can be replayed.
- Concurrency tests are run several times. **A flaky test is a finding, not bad luck.**
- Network tests: **never port 0, never a port "freed" then reused**. A fixed block reserved for tests (17000-17999, below the dynamic range), a sub-block of 100 ports per test project, a base movable by an environment variable, and a clear message if the port is taken.
- One test starts each tier with its **shipped configuration**, to validate the real ports.
- SQL tests are skipped cleanly without a test database (prefix `TEST_` or `DEV_`), and a local gate is mandatory before each version merge.

## 5. Review before merge: the whole team

Required for **every** pull request that Claude merges, documentation only included.

| Agent | Code pull request | Documentation pull request |
|---|---|---|
| C-3PO (C#) / AP-5 (Python) | style and conventions, then memory and CPU leaks (`Dispose` / `using`, subscriptions, timers, unbounded collections, busy loops, sync-over-async, allocations on hot paths, cancellation token passed on) | language and style, accuracy |
| Ackbar + Cassian | data leaks and OWASP (secrets or personal data in logs and errors, internal details exposed, injections, denial of service), CVEs and dependencies. **Run without an explicit request in this case** | no secret, no internal address, no exploitable detail; versions quoted exactly |
| Boba Fett | swallowed errors, silent fallbacks, network | error behaviours described accurately |
| Kenobi | every behaviour proven by a test, flaky tests | every announced behaviour covered by a test |
| Yoda | only if the pull request carries an architecture decision (second reading **before** coding, without showing him the conclusion) | — |
| Jocasta Nu | the pull request contradicts no accepted ADR; an architecture decision without an ADR is reported | on `doc/analyse/adr/`: format, accuracy, consistent index and statuses |

- Claude **checks every finding** before acting on it, then sorts: blocking → fixed; important and cheap → fixed; the rest → deferred **with the reason**, listed in the pull request.
- Review fixes get a **second reading** by the agents that found them.
- Every pull request lists **who reviewed what**, with the findings fixed or justified. Review reports of the ADRs are archived in `doc/analyse/reviews/`.
- Agents read, test and cite their sources (Microsoft Learn, official documentation); what they could not check is marked "not verified".

## 6. Autonomous work

- Work from a clear brief, with as few round trips as possible.
- **Never decide alone what belongs to the product owner** (going back on a merged choice, an exception to a rule): keep what exists, bound the risk, and ask.
- The closing report is short: the order in which to read the pull requests, what is done, what is **proven**, what is **not verified**, what **waits for his decision**, the clean-up he must do himself.

## 7. C# code style

The reference is `doc/dev/coding-conventions.md` (to be written with the first code).

- **File header** in `/* */` with `@file`, `@author Defraene Michaël (AKA Sonic) (co-written by Claude)` (his name alone for files he writes alone) and `@brief`. **No versions block**: Git keeps the history.
- Fixed using blocks: `// Using`, `// Static using`, `// Named using`, `// Namespace` (file-scoped namespace), with `// #N/A` when a block is empty.
- **Fixed and complete regions**, in the same order, even when empty (`// #N/A`): Members (Constants, Variables, Delegates), Enumerations, Services, Accessors/Properties, Constructors, Destructors, Methods (Privates, Publics), Overrides (Methods, Operators).
- **XML documentation on everything, private members included**: `<summary>Method used to …</summary>`, `<remarks>` with **Purpose** and **Behavior**, `[Can be null]` / `[Cannot be null]` on parameters, `<returns>` on several lines (`True  ==> …` / `False ==> …`), one `<exception>` per thrown case.
- **Step comments**: `// Check(s)`, `// Setting …`, `// Transforming data`, and always `// Function's return` before a `return`.
- **Never `var`**; `new Type()` written in full; guard checks at the top of a method, without braces when they fit on one line.
- Naming: `_camelCase` (also for `private static readonly`), services `_svcXxx`, suffix `Async`.
- Logging through `[LoggerMessage]`, with a **reserved EventId range** per library class, recorded in the conventions.
- Analysers: a rule is never suppressed (it is a safeguard owned by the product owner). Respect the rule instead, for instance with the standard exception constructors rather than suppressing RCS1194.
- **SQL access: plain ADO.NET** (`Microsoft.Data.SqlClient`), **never Entity Framework**, Dapper or any other ORM. Everything goes through stored procedures, table-valued parameters and views, through the data gateway only (ADR-0003). The SQL project keeps its original format (`.sqlproj`, DACPAC), never the SDK format.

## 8. Python: model training

Same standard as C#: production code, not notebooks. Python only **trains** the models; data preparation is done by .NET and the models run in .NET through ONNX Runtime (ADR-0009). The tools below are **proposed defaults**, each to be approved by the product owner when the Python work starts, one decision at a time.

### Visual Studio solution

- Python code is a `.pyproj` project in the `.slnx`, next to the C# and SQL projects, in one solution folder per layer. The `.pyproj` coexists with `pyproject.toml` (package, dependencies, tool settings) and points to the project's venv.
- Neither the `.pyproj` nor the `.sqlproj` builds with `dotnet build`: the .NET CI uses a `.slnf` solution filter that excludes them (a safeguard checks that the `.slnf` follows the `.slnx`). The SQL project has its own job (Visual Studio MSBuild on `windows-latest`), Python its own (venv, linter, type checker, `pytest`). That `dotnet build` refuses the `.pyproj` is **not verified**.

### Environment and repository

- Always the project's **venv**, never the system Python. Dependencies declared in `pyproject.toml`, **pinned by a lock file**, audited by Cassian.
- Python version fixed (`requires-python`), the same locally and in CI.
- Layout `src/<package>/` + `tests/`, separate from `notebooks/` (exploration only, never imported).
- Raw data, models and outputs are **never versioned** (see `.gitignore`): their **description** is (source, version, hash).

### Code quality

- **Strict typing** everywhere (`mypy --strict` or equivalent), PEP 8, formatter and linter (`ruff`) in CI, failing the build like the C# analysers.
- **Docstrings on everything**, private functions included: purpose, parameters, return, raised exceptions. A module header states its purpose.
- No unreadable trick: no comprehension nested three levels deep, no metaclass magic without a measured reason.

### Reproducibility

- **Every seed fixed and logged** (Python, NumPy, framework), deterministic mode on when available; what cannot be fixed is documented.
- **The whole training configuration in a versioned file** (hyperparameters, paths, seeds), never hard-coded. A training run is replayed from commit + configuration + data version.
- Each run records its parameters, metrics, environment and artefacts in an experiment tracker (tool to choose with the product owner).

### Data

- **Data contract**: the expected schema (columns, types, ranges, missing values) is checked on input; out-of-contract data is refused with a clear message, never silently corrected.
- **No leak between training and evaluation**: one split, made once, before any learned transformation, then frozen; a time-based split for dated data.
- Personal data (IP addresses, user names) is **pseudonymised** before training (ADR-0012). No personal data ends up in a log, an exported model or a report.

### Evaluation and tests

- A simple **baseline** first; a complex model must prove it does better.
- Metrics chosen before training, from the real need, computed on the test set **once**, at the end, and shown with their uncertainty (several seeds or cross-validation).
- `pytest` unit tests of the transformations and metrics with small fixed datasets; a training **smoke test** that runs in seconds in CI and lowers the loss; a deliberate **overfitting test** on a tiny batch; a **non-regression test** of the exported model (same inputs, same outputs within a tolerance).

### Model delivery

- Exported to ONNX with its model card (data, metrics, known limits, date, commit).
- A model is an artefact like a binary: it goes through a pull request, the team review and a rollback plan (the previous version stays deployable).

## 9. Operations

- Target hosts run **Fedora or Windows** only: `dnf`, SELinux, firewalld. Scheduled tasks on Linux are **systemd timers**, never cron.
- **Everything is set in the configuration file**, nothing hard-coded in a script or a service (endpoints, paths, delays, retries). If a value changes, the configuration changes, never the code.
- Changes on the product owner's servers and firewall are applied by him: Claude prepares the files and the commands, he runs them and returns the output.

## 10. Verified tooling pitfalls (Windows)

- Claude's tools write **LF**; the working tree uses CRLF (`.gitattributes`), except files read on Linux (`*.sh`, `*.service`, `*.socket`). Once an `.editorconfig` exists, run the formatter (`dotnet format … --verify-no-changes`) before each pull request.
- **IDE1006** (naming) is not enforced by the build: only `dotnet format` detects it.
- xUnit v3 runs on **Microsoft.Testing.Platform**: `"test": {"runner": "Microsoft.Testing.Platform"}` in `global.json`, then `dotnet test --solution X` / `--project X`.
- The shell blocks `Remove-Item` and some deletions (`git worktree remove`, `git branch -D`): leave them to the product owner.
- Never change directory in a command (`cd`, `Set-Location`): use absolute paths or `-WorkingDirectory`.
- **Heredocs** containing quotes break the Bash tool: write edit scripts to a scratch file.
- **Agent worktrees start from `main`** (the default branch): the brief must impose the base `release/X.Y.Z`.
- An agent that builds in the main folder while Claude switches branches loses its ground: work in a **separate worktree** while agents review.
- An agent definition (`~/.claude/agents/*.md`) is loaded only when a session starts.
