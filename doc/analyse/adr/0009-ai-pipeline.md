# ADR-0009: AI pipeline: rules, .NET preparation, Python training, ONNX, LLM

| Field | Value |
|---|---|
| Status | Proposed |
| Date | 2026-11-03 |
| Decider | Michaël Defraene (product owner, architect) |
| Drafted by | Claude (main developer); the decisions had a second reading by the architecture review agent during scoping |

## Context

Dashboard is the final project of an AI developer training, which requires Python (pandas and similar libraries) for analysis and training. The system needs three capabilities: detect known attacks, detect anomalies nobody wrote a rule for, and produce readable reports. A classic risk of machine learning in production is **training/serving skew**: the data is prepared one way for training and another way in production, and the model silently degrades.

## Decision

The pipeline has **deterministic rules, one anomaly model (ONNX) and one local LLM**:

1. **Deterministic rules**, evaluated in real time by the Backend (.NET), detect known attacks.
2. **Anomaly model**:
   - The **.NET code collects and prepares** the data (log templates, counts per time window) and writes the prepared data to the NAS. The data is pseudonymised before training (ADR-0012).
   - **Python only trains** (pandas, scikit-learn): a naive baseline first, then IsolationForest. The model is exported to **ONNX (Open Neural Network Exchange) from V1**.
   - The Backend **applies the model with ONNX Runtime**.
   - There is **one preparation implementation** (the .NET one), used both for training data and in production, so there is no training/serving skew.
3. **Local LLM**: writes reports from **computed facts** (counts, retained anomalies, rule hits once rules exist, see ADR-0015), never from raw logs, and explains the retained anomalies.

The Python code follows the project's Python rules: reproducible runs (fixed and logged seeds, configuration in a file), a data contract checked on input, a baseline before any complex model, and a non-regression test of the exported model.

## Alternatives considered

- **Python prepares and trains.** Rejected: having two preparation implementations (Python for training, .NET in production) creates exactly the skew this decision avoids.
- **Python model served by a Python service.** Rejected: one more service on the data path; ONNX Runtime keeps inference inside the Backend.
- **LLM reading raw logs.** Rejected: unbounded input on small hardware, higher risk of invented content, and personal data sent to the model.

## Consequences

### Positive

- The same code prepares training and production data.
- Inference has no Python dependency at run time.
- Reports are grounded in facts that can be checked.

### Negative

- The log template algorithm must be implemented in .NET (open point 2).
- ONNX export must be validated for each model version (same inputs, same outputs within a tolerance). IsolationForest is listed among the scikit-learn models supported by `skl2onnx` (verified 2026-11-03).
- Log templates are text an attacker can influence: facts given to the LLM may carry prompt-injection attempts, and reports must be displayed as plain text.
- Two languages to maintain, each with its own CI job.

## Open points

1. **Who calls the LLM.** Since Python only trains, the proposal is that the **Backend (.NET)** calls the LLM through the HTTP API of its server (Ollama or llama.cpp) on the AI node.
2. **.NET implementation of log templates.** Drain (as in the Python library Drain3) was the planned algorithm. Proposal: an in-house C# implementation of Drain, tested against Drain3 outputs on a fixed corpus (Drain3 is then only a test oracle, never part of the pipeline). Whether a maintained .NET implementation already exists is **not verified**.
3. Raised in review, proposed: the Backend checks the model's hash, input schema and version when loading it; if loading fails, rules keep running, an alert is raised and anomaly scoring is shown as unavailable; sources absent from the training data are flagged "not covered by the model"; the template state is versioned with the model.
4. Raised in review, proposed: the LLM call has an explicit timeout sized on the measured latency; each report records the model, the hardware used (GPU or CPU), its generation status and the facts it was generated from; a failed generation is shown as failed, never replaced by an older report.

## References

- sklearn-onnx documentation, *Supported scikit-learn Models* (consulted 2026-11-03).
- ADR-0010, ADR-0012, ADR-0015.
