# ADR-0010: Hosting the local LLM on the dedicated AI node

| Field | Value |
|---|---|
| Status | Proposed |
| Date | 2026-11-03 |
| Decider | Michaël Defraene (product owner, architect) |
| Drafted by | Claude (main developer); the decisions had a second reading by the architecture review agent during scoping |

## Context

Reports are written by a **local** LLM, so that logs and facts about the infrastructure never leave the home lab. Two machines are available:

- a PC with an NVIDIA GTX 1050 with **2 GB of VRAM** (Windows);
- a GMKtec NucBox G10: AMD Ryzen 5 3500U, integrated Radeon Vega 8 GPU, **16 GB of RAM shared** with the GPU (Linux).

Reports are generated in batch; latency of minutes is acceptable, but the model must fit in memory.

## Decision

- The LLM runs on the **G10**, which is the AI node (ADR-0001). It has no database access.
- Target model size: **3 to 4 billion parameters, quantised to 4 bits**. Rough estimate: 4 billion parameters at about half a byte each is about 2 GB of weights, plus context cache and runtime overhead, which fits in 16 GB of shared memory. By the same estimate, the 2 GB of VRAM of the GTX 1050 limit a model on that GPU to roughly 1 to 1.5 billion parameters.
- GPU acceleration through **Vulkan**, with llama.cpp or Ollama. In Ollama, Vulkan is on by default once its Vulkan backend is installed (`OLLAMA_VULKAN=0` disables it); the use of integrated GPUs is governed by `OLLAMA_IGPU_ENABLE`, which appears in the Ollama source but not in its user documentation, and must be tested on the Vega 8.
- A **benchmark is run at the start** of the AI increment; if the Vega 8 is not usable or too slow, the model runs on the **CPU**.

## Alternatives considered

- **GTX 1050.** Rejected: 2 GB of VRAM only fit models too small for useful reports.
- **Hosted LLM API.** Rejected: data about the infrastructure would leave the home lab.

## Consequences

### Positive

- Logs and facts stay inside the home lab.
- The AI node can be rebuilt without touching the data path.

### Negative

- Support of the Vega 8 by the Vulkan drivers and the actual throughput are **not verified**; the benchmark decides.
- Ollama can fall back to the CPU on its own if the GPU is no longer detected (after a driver or kernel update, for example), which slows reports down without an error; each report records the hardware used (ADR-0009).
- The shared memory is also used by the training jobs: training and report generation must not run at the same time (a lock, not a convention).
- The model choice (family, exact size) is still open and will be made after the benchmark.

## Open points

1. Choice of the model, after the benchmark.

## References

- Ollama documentation, *GPU* page, and source file `envconfig/config.go` (consulted by the review, 2026-11-03).
- GMKtec NucBox G10 product page; TechRadar review; Phoronix articles on Vulkan inference (consulted during scoping).
- ADR-0001, ADR-0009.
