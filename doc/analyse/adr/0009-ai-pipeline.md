# ADR-0009: AI pipeline: rules, .NET preparation, Python training, ONNX, LLM

| Field | Value |
|---|---|
| Status | Accepted |
| Date | 2026-11-04 |
| Decider | Michaël Defraene (product owner, architect) |
| Drafted by | Claude (main developer); the decisions had a second reading by the architecture review agent during scoping |

## Context

Dashboard is the final project of an AI developer training, which requires Python (pandas and similar libraries) for analysis and training. The system needs three capabilities: detect known attacks, detect anomalies nobody wrote a rule for, and produce readable reports. A classic risk of machine learning in production is **training/serving skew**: the data is prepared one way for training and another way in production, and the model silently degrades.

## Decision

The pipeline has **deterministic rules, one anomaly model (ONNX) and one local LLM**.

### 1. Deterministic rules

Rules are evaluated in real time by the **Collector** (.NET) and detect known attacks.

### 2. Anomaly model

- The **.NET code collects and prepares** the data and writes the prepared data to the NAS. The data is pseudonymised before training (ADR-0012).
- The **input features of the V1 model**, per machine and per time window, are:
  - the counts of log templates;
  - aggregates of the metrics defined in ADR-0008 (processor, memory, disks).
- Log templates are mined with **Drain**, implemented in-house in C# in the Fleet core. It is tested against the outputs of the Python library Drain3 on a fixed corpus; Drain3 is only a test oracle, never part of the pipeline. An existing .NET port (DrainDotNet 1.0.1 on NuGet) was considered and not adopted (see Alternatives).
- **Python only trains** (pandas, scikit-learn): a naive baseline first, then IsolationForest. The model is exported to **ONNX (Open Neural Network Exchange) from V1**.
- There is **one preparation implementation** (the .NET one), used both for training data and in production, so there is no training/serving skew.
- The **Collector applies the model with ONNX Runtime**, next to the rules, on the incoming data.
- **Loading the model is checked.** When it loads a model, the Collector checks its hash, its input schema and its version. If loading fails:
  - the rules keep running;
  - an alert is raised;
  - anomaly scoring is shown as unavailable, never as "no anomaly".
- Sources that are absent from the training data are flagged **"not covered by the model"**.
- The Drain state (the template tree) is **versioned with the model**: a model is always used with the templates it was trained on.

### 3. Local LLM

- The LLM writes reports from **computed facts** (counts, retained anomalies, rule hits once rules exist, see ADR-0015), never from raw logs, and explains the retained anomalies.
- The **API** calls the LLM through the HTTP API of its server (Ollama or llama.cpp, ADR-0010) on the AI node. It reads the facts through the VBE (data gateway). The latency of the LLM therefore never affects ingestion.
- The call has an **explicit timeout**, sized on the latency measured during the benchmark of ADR-0010.
- Each report records:
  - the model;
  - the hardware used (GPU or CPU);
  - its generation status;
  - the facts it was generated from.
- A failed generation is shown as failed, never replaced by an older report.
- Reports are displayed as plain text (see Consequences).

### Python rules

The Python code follows the project's Python rules: reproducible runs (fixed and logged seeds, configuration in a file), a data contract checked on input, a baseline before any complex model, and a non-regression test of the exported model.

## Alternatives considered

- **Python prepares and trains.** Rejected: having two preparation implementations (Python for training, .NET in production) creates exactly the skew this decision avoids.
- **Python model served by a Python service.** Rejected: one more service on the data path; ONNX Runtime keeps inference inside the Collector.
- **LLM reading raw logs.** Rejected: unbounded input on small hardware, higher risk of invented content, and personal data sent to the model.
- **Scoring and LLM calls in the Collector.** Rejected: a slow or failing LLM would share resources with ingestion.
- **Scoring and LLM calls in the API.** Rejected: scoring would run after the fact, by reading the data back through the VBE, and anomalies would no longer be raised as the data arrives.
- **DrainDotNet (NuGet package).** Rejected: a very small port (about one GitHub star and a few hundred downloads on 2026-11-04), whose licence is reported inconsistently (MIT or Apache 2.0 depending on the source), and which adds a non-standard extension to Drain. Depending on it at the heart of the preparation was judged too fragile.
- **Log templates only as model input, metrics later.** Proposed by the main developer to keep the first model small; not adopted by the product owner, who wants the metrics in the V1 model.

## Consequences

### Positive

- The same code prepares training and production data.
- Inference has no Python dependency at run time.
- Reports are grounded in facts that can be checked.
- Rules and anomaly scoring run in the same process, on the same data, as it arrives.
- A broken model or a failing LLM is visible as such, and never hides behind a normal-looking screen.

### Negative

- Drain must be implemented and maintained in C#, with its own tests against Drain3.
- ONNX export must be validated for each model version (same inputs, same outputs within a tolerance). IsolationForest is listed among the scikit-learn models supported by `skl2onnx` (verified 2026-11-03).
- The data contract is larger: the features combine log templates and metrics, which must be aligned on the same time windows. A window with logs but without metrics (or the reverse) must be handled explicitly by the contract.
- Windows logs and metrics arrive in 0.4 (ADR-0015): the model must be retrained then, and until that point Windows sources are flagged "not covered by the model".
- Log templates are text an attacker can influence: facts given to the LLM may carry prompt-injection attempts, and reports must be displayed as plain text.
- Two languages to maintain, each with its own CI job.

## Open points

None. The choice of the LLM model is made in ADR-0010.

## References

- sklearn-onnx documentation, *Supported scikit-learn Models* (consulted 2026-11-03).
- NuGet, *DrainDotNet 1.0.1*, https://www.nuget.org/packages/DrainDotNet (consulted 2026-11-04).
- ADR-0005, ADR-0008, ADR-0010, ADR-0012, ADR-0015.
