# ADR-0010: Hosting the local LLM on the dedicated AI node

| Field | Value |
|---|---|
| Status | Accepted |
| Date | 2026-11-04 |
| Decider | Michaël Defraene (product owner, architect) |
| Drafted by | Claude (main developer); the decisions had a second reading by the architecture review agent during scoping |

## Context

Reports are written by a **local** LLM, so that logs and facts about the infrastructure never leave the home lab. Two machines are available:

- a PC with an NVIDIA GTX 1050 with **2 GB of VRAM** (Windows);
- a GMKtec NucBox G10: AMD Ryzen 5 3500U, integrated Radeon Vega 8 GPU, **16 GB of RAM shared** with the GPU (Linux).

Reports are generated in batch; latency of minutes is acceptable, but the model must fit in memory. The same node also runs the Python training jobs (ADR-0009), which use the same shared memory.

## Decision

### Node and model size

- The LLM runs on the **G10**, which is the AI node (ADR-0016). It has no database access.
- Target model size: **3 to 4 billion parameters, quantised to 4 bits**. Rough estimate: 4 billion parameters at about half a byte each is about 2 GB of weights, plus context cache and runtime overhead, which fits in 16 GB of shared memory. By the same estimate, the 2 GB of VRAM of the GTX 1050 limit a model on that GPU to roughly 1 to 1.5 billion parameters.

### Server and client

- The API (ADR-0009) calls the LLM through the **OpenAI-compatible chat completions route** (`/v1/chat/completions`). Both candidate servers offer it, so changing the server is a configuration change, not a code change.
- The preferred server is **llama.cpp's `llama-server`**, because it supports an API key (`--api-key` or `--api-key-file`). The key is a second layer behind the firewall rule of ADR-0004. Ollama documents that it ignores the API key on its OpenAI-compatible route, so with Ollama the firewall would be the only protection.
- Ollama is used instead only if the benchmark shows a real gap in its favour (throughput, memory or stability on the Vega 8).

### Acceleration and benchmark

- GPU acceleration through **Vulkan**. In `llama-server`, the Vulkan backend depends on how the binary is built, and the devices are listed with `--list-devices` and chosen with `--device`. In Ollama, Vulkan is on by default once its Vulkan backend is installed (`OLLAMA_VULKAN=0` disables it); the use of integrated GPUs is governed by `OLLAMA_IGPU_ENABLE`, which appears in the Ollama source but not in its user documentation, and must be tested on the Vega 8.
- A **benchmark is run at the start of increment 0.3** (ADR-0015). It measures, for each candidate model and server: throughput (tokens per second), peak memory, and the quality of a reference report generated from a fixed set of facts. If the Vega 8 is not usable or too slow, the model runs on the **CPU**.
- The **choice of the model** (family and exact size) is recorded in a **new ADR** written after the benchmark, with its results. This ADR does not decide it.

### Training and reports never run at the same time

- The training job **stops the LLM service** (systemd) before it starts, and **restarts it when it ends**, including when it fails (`ExecStopPost=` in the unit, or an equivalent trap in the script).
- While the service is stopped, the API records the reports it could not generate as failed with the reason "LLM unavailable" (ADR-0009); they are never replaced by older reports.
- Training is scheduled at night.

## Alternatives considered

- **GTX 1050.** Rejected: 2 GB of VRAM only fit models too small for useful reports.
- **Hosted LLM API.** Rejected: data about the infrastructure would leave the home lab.
- **Ollama as the default server.** Rejected as the default: simpler to install and to manage models, but no authentication on its OpenAI-compatible route. Kept as a fallback.
- **The server's native API (Ollama API) in the client.** Rejected: it would tie the code to one server.
- **Time slots (training at night, reports during the day) without a lock.** Rejected: a convention, not a lock; a late training job would compete with report generation for the shared memory.
- **Choosing the model now.** Rejected: without a benchmark on the Vega 8, the choice would not rest on measurements.

## Consequences

### Positive

- Logs and facts stay inside the home lab.
- The AI node can be rebuilt without touching the data path.
- The server can be swapped without changing the code.
- Training and report generation cannot compete for memory: while the service is stopped, nothing can load the model.

### Negative

- Support of the Vega 8 by the Vulkan drivers and the actual throughput are **not verified**; the benchmark decides.
- A server can fall back to the CPU on its own if the GPU is no longer detected (documented for Ollama; **not verified** for `llama-server`), which slows reports down without an error. Each report records the hardware used (ADR-0009).
- The OpenAI-compatible routes are only partly compatible (Ollama documents its limits); the client must only use the fields both servers support.
- The API key travels in the HTTP request. Unless the link is encrypted, it can be read on the network. Whether `llama-server` can serve TLS itself is **not verified**; the protection of this link is settled at deployment, within the exception of ADR-0004.
- The API key is a secret to store and rotate on both the AI node and the API.
- Reports requested during training fail and must be requested again.

## Open points

None. The model choice will be a new ADR after the benchmark.

## References

- llama.cpp, `tools/server/README.md` (`--api-key`, `--api-key-file`, `--device`, `--list-devices`; OpenAI-compatible routes), https://github.com/ggml-org/llama.cpp/blob/master/tools/server/README.md (consulted 2026-11-04).
- Ollama documentation, *OpenAI compatibility*, https://docs.ollama.com/api/openai-compatibility (consulted 2026-11-04).
- Ollama documentation, *GPU* page, and source file `envconfig/config.go` (consulted by the review, 2026-11-03).
- GMKtec NucBox G10 product page; TechRadar review; Phoronix articles on Vulkan inference (consulted during scoping).
- ADR-0004, ADR-0009, ADR-0015, ADR-0016.
