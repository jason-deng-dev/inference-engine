# 05 — The Final Form

> **Canonical master:** [`00-master.md`](00-master.md). Sizing, cut order, gates, and operating rules live there — if this file and the master disagree, the master wins.
> **Status: target document.** Nothing here is built. This is the finished project's claim set, enumerated once so the plan can be checked against its own goal. Companion docs: [`04-inference-engine-pivot.md`](04-inference-engine-pivot.md) (why the project exists) · [`20-ai-extension.md`](20-ai-extension.md) (steps A/G and the post-engine tiers) · [`90-resumes.md`](90-resumes.md) (the one-page projection of this file)

**What this file is for.** Every other doc says what to *build* and in what order. This one says what the project *claims* when it is finished, which artifact produces each claim, and what the claim costs to earn. It exists because the resume appendices are compressed on purpose — one page, with a cut order — and a compressed projection is a bad place to check whether the underlying plan is ambitious enough. Read this to see the ceiling; read [`90-resumes.md`](90-resumes.md) to see what a page can carry.

**This document adds no commitments of its own.** Every row below is derived from a phase doc, [`20-ai-extension.md`](20-ai-extension.md), or [`31-verification-and-observability.md`](31-verification-and-observability.md), and cites it. Two rows arrived with the v4.3 AI-infra completeness pass — prefix caching and the Triton comparison — and their commitments live in the docs that own them ([`16-phase-6-scheduler.md`](16-phase-6-scheduler.md) rung 2b, [`20-ai-extension.md`](20-ai-extension.md) A6). If a row here is not in the plan, the row is wrong and the plan wins.

---

## 1. The system, finished

One C++17 repository, one CMake build, one CI, one benchmark directory. A loopback request stream enters through a sub-microsecond admission path — token-bucket rate limits, per-tenant concurrency caps, a restart-safe circuit breaker, a lock-free accounting ledger — crosses an SPSC dispatch queue into an iteration-level continuous-batching scheduler that admits on a projected KV budget, and is served by a from-scratch transformer decoder whose attention runs over a KV cache behind a swappable allocator interface. The decoder's achieved memory bandwidth is quoted against a ceiling measured by a GEMM ladder living in the same repo, on the same machine, under the same harness. Every number in the README regenerates from `bench/`; every optimization claim has an ablation or a profiler counter under it; every correctness claim has a differential oracle or a replay hash behind it. The model is a deliberate placeholder — the work is the serving and performance layer, and the docs, the resume, and the interview script all say so in the same words.

---

## 2. Claim set

Tier column: **C** = core (phases 0–6, steps A and G — the spine) · **P** = post-engine (v4.2 stretch tiers) · **N** = never claimed, see §3.

### Kernel and GPU

| Claim | Producing artifact | Measurement | Tier | Source |
| --- | --- | --- | --- | --- |
| GEMM ladder: naive → coalesced → shared-memory tiled → register-blocked | `src/kernels/`, tables in `bench/` | Achieved FP32 FLOPs and achieved DRAM bandwidth against the machine's ceilings; occupancy sweep; the roofline figure; every rung's Nsight Compute output linked | C | [20-ai-extension.md](20-ai-extension.md) A3–A4 |
| Where the ladder stops scaling, and which ceiling stops it | Same | Named limiter per rung; a rung that buys nothing reported as a finding | C | [20-ai-extension.md](20-ai-extension.md) A4 |
| One quantized or fused kernel (INT8 dequant-GEMM **or** fused attention, not both) | A5 | Throughput gain and accuracy cost against the FP32 baseline; **percentage of cuBLAS** with the gap attributed to a named cause cuBLAS does and you do not | C | [20-ai-extension.md](20-ai-extension.md) A5 |
| The same GEMM in Triton, diffed against hand-written CUDA and cuBLAS | A6 (added v4.3) | Three-way table on one shape and one GPU: where the abstraction matches, where autotuning a search space beats a hand sweep, where explicit pipeline depth and shared-memory control win | C (cut early) | [20-ai-extension.md](20-ai-extension.md) A6 |
| GPU Monte Carlo with seeded replay bit-identical across the device boundary | Backtester | Same path hashes and same terminal equities on CPU and GPU | C | [20-ai-extension.md](20-ai-extension.md) A2 |

### Decoder

| Claim | Producing artifact | Measurement | Tier | Source |
| --- | --- | --- | --- | --- |
| From-scratch GPT-2-class transformer in C++17: multi-head attention, KV cache behind an allocator interface, greedy decode | `src/decoder/` | Correctness gate first: output token-for-token identical to a reference running **the same weights and the same tokenizer** on a fixed prompt | C | [20-ai-extension.md](20-ai-extension.md) step G |
| Single-stream performance | Same | tok/s; TTFT at 512 / 1024 / 2048 context; achieved DRAM bandwidth during decode as a percentage of the ladder's measured ceiling; KV footprint against context length | C | [20-ai-extension.md](20-ai-extension.md) step G |
| Architecture breadth: a Llama-shaped config — RoPE, RMSNorm, SwiGLU, grouped-query attention — behind the *same* token-for-token gate | Decoder, extended | GQA's measured KV reduction by the query-to-KV-head ratio; correctness against a reference on a tiny checkpoint | P | [20-ai-extension.md](20-ai-extension.md) Tier A item 2 |

### Scheduler

| Claim | Producing artifact | Measurement | Tier | Source |
| --- | --- | --- | --- | --- |
| Iteration-level continuous batching with KV-budget admission (worst-case projection, reject-at-capacity, no preemption at rung 1) | `src/scheduler/` | Throughput vs. per-token latency by batch size, presented **with** achieved decode bandwidth against the roofline — the curve flattens where bandwidth saturates | C | [16-phase-6-scheduler.md](16-phase-6-scheduler.md) rung 1 |
| Rung ladder: naive → continuous batching → chunked prefill → prefix caching | Same, plus the rung-0 reference scheduler | Rung deltas on one workload; a rung that buys nothing stated as such | C | [16-phase-6-scheduler.md](16-phase-6-scheduler.md) |
| Prefix caching: KV reuse across requests sharing a prefix | Scheduler rung 2b, over the decoder's KV allocator interface | Hit rate and TTFT delta against a shared-prefix workload with caching on and off; memory cost of retained blocks and the eviction policy that bounds it; token-identical output with caching on and off | C (cut early) | [16-phase-6-scheduler.md](16-phase-6-scheduler.md) rung 2b |
| Batching is a scheduling change, not a numerical one | Output-equality gate | Token-for-token identical per request across batch compositions **and** thread counts; fuzzed streams differential-tested against rung 0 with zero divergences | C | [16-phase-6-scheduler.md](16-phase-6-scheduler.md) |
| Scheduler overhead per admission decision | Same | Nanoseconds, quoted against a per-token cost measured in milliseconds | C | [16-phase-6-scheduler.md](16-phase-6-scheduler.md) |
| Two SLOs, two budgets | Phase-3 harness | TTFT and TPOT under concurrent load at stated offered rates, coordinated-omission-correct; an error budget per SLO and what burns it | C | [15-phase-5-docs.md](15-phase-5-docs.md) |
| Fairness policy with a stated starvation boundary | Scheduler | A named policy (WFQ / deficit round robin family), not "the caps handle it" | C | [16-phase-6-scheduler.md](16-phase-6-scheduler.md) rung 1 |
| Decode graph capture per batch-size bucket (stretch) | Scheduler rung 3 | Launch cost before/after from Nsight Systems, **with the bucketing padding cost as its own row**; unbuilt if the trace shows launch overhead is noise | C (cut early) | [16-phase-6-scheduler.md](16-phase-6-scheduler.md) rung 3 |

### Gateway, correctness, and measurement

| Claim | Producing artifact | Measurement | Tier | Source |
| --- | --- | --- | --- | --- |
| Sub-microsecond O(1) admission control over cache-dense per-tenant state | `src/gateway/` | p99 decision latency, and the multiple over the deliberately naive reference — mutex, `std::unordered_map`, heap allocation per decision | C | [11-phase-1-core.md](11-phase-1-core.md), [14-phase-4-zero-alloc.md](14-phase-4-zero-alloc.md) |
| Layout and micro-arch ablation | Phases 2 and 4 | Cross-node atomics vs. local (2-socket bare metal); false-sharing elimination's p99 contribution; branch-miss reduction with the documented case where branchless *loses*; each factor's p99.9 contribution on the integrated pipeline | C | [12-phase-2-microarch.md](12-phase-2-microarch.md), [14-phase-4-zero-alloc.md](14-phase-4-zero-alloc.md) |
| Zero hot-path allocations, verified not asserted | Phase 4 | Interposition of `malloc`/`free`/`realloc` **and** `operator new`/`delete` in one `LD_PRELOAD` library, cross-checked once with a heap profiler | C | [14-phase-4-zero-alloc.md](14-phase-4-zero-alloc.md) |
| Correctness: fuzz, differential, replay | Phase 1 (+ 3) harness | Structured sequences via a custom mutator (not raw bytes) run through both the reference and the real path; decisions must match exactly; replay bit-identical across runs and thread counts | C | [11-phase-1-core.md](11-phase-1-core.md) |
| Open-loop, coordinated-omission-correct latency | `bench/` load generator | Per-stage TTFT/TPOT under concurrent load; overload behaviour at a stated multiple of capacity, with the shedding policy and the breaker threshold named | C | [13-phase-3-loadgen.md](13-phase-3-loadgen.md) |
| A system whose hot path cannot be scraped, still observable | Phase 1 seqlock + snapshot publisher | Measured uprobe perturbation on the path; counters published from a seqlock-protected snapshot; what was kept in eBPF because it cannot perturb | P | [31-verification-and-observability.md](31-verification-and-observability.md) |
| Ring all-reduce in C++ with an alpha–beta cost model and a measured crossover | Standalone, plus the decoder | Latency and bandwidth terms fitted against message size; the crossover per interconnect (1GbE, 2.5GbE, loopback, PCIe, NVLink); **NCCL run as the reference at the same sizes and ranks** and diffed against the hand-rolled fit | P | [20-ai-extension.md](20-ai-extension.md) Tier A item 1 |
| Tensor parallelism on the decoder across a rented node | Decoder, sharded | Scaling efficiency at 1, 2, and 4 ranks against the linear ideal per batch size, shortfall attributed to a named term (all-reduce latency vs. per-layer compute, KV-shard imbalance, absent communication/compute overlap); token-identical across rank counts. **At GPT-2 124M the honest result is that TP loses** — the artifact is the measured crossover | P | [20-ai-extension.md](20-ai-extension.md) Tier B item 6 |
| Quantization end to end | Decoder, INT8 path | Throughput gain against accuracy cost: perplexity delta plus token-agreement rate versus the FP32 gate | P | [20-ai-extension.md](20-ai-extension.md) Tier B item 4 |
| Speculative decoding | Decoder, draft + target | Speedup at batch 1 decaying toward 1× as batch grows — the technique and continuous batching pulling one knob in opposite directions, measured on one system | P | [20-ai-extension.md](20-ai-extension.md) Tier B item 5 |

---

## 3. What this project never claims

Each is a boundary with a prepared answer, not a gap to hide. [15-phase-5-docs.md](15-phase-5-docs.md) holds the scripts.

| Not claimed | The answer |
| --- | --- |
| Distributed training; NCCL as a *build* | Out of scope by design. NCCL appears only as the reference implementation the hand-rolled ring is validated against ([20-ai-extension.md](20-ai-extension.md)). |
| Multi-GPU across the two home machines | Impossible, not deferred: ROCm and CUDA cannot join one collective, and consumer Ethernet costs roughly 4–6 ms of pure latency per token across 64 collectives ([04-inference-engine-pivot.md](04-inference-engine-pivot.md) §13). |
| Kubernetes, cluster operations, Grafana/OpenTelemetry/operators | Tooling, not findings. Two hours of manifests earn no claim ([20-ai-extension.md](20-ai-extension.md)). |
| Serving a real 7B model | Deliberate. Architecture breadth (RoPE/GQA) closes the vocabulary gap with engineering instead of a download ([20-ai-extension.md](20-ai-extension.md)). |
| Kernel bypass **in this repo** | Structural: the ingress is a load generator, so bypassing the kernel removes cost from the wrong side of the measurement. The build lives in the Orderbook ([41-network-ingestion.md](41-network-ingestion.md)). |
| A second backend (ONNX Runtime, TensorRT) | First item in the cut order ([00-master.md](00-master.md)). |
| Production deployment, on-call, SLO ownership at scale | A different claim, made on the GoodSoft work-history block, not on a portfolio project ([90-resumes.md](90-resumes.md)). |
| Frontier-scale models, MLOps at scale, RAG | Vocabulary only. |

**The model is a placeholder.** Every document says it in the same words: *my work is the serving and performance layer; the model is a deliberate boundary, not a gap.*

---

## 4. Why the resume projects thin, and why that is not fixable by editing prose

Three rules compress this file into one page, and each removes a different kind of strength:

1. **The bracket rule** — an unfilled `[N]` ships as the shorter line, or the clause is deleted ([00-master.md](00-master.md) rule 1). Every measurement in §2 is therefore a deletion until it exists.
2. **Resume is output, not input** — the appendices describe work that exists; they are targets until the brackets are filled ([00-master.md](00-master.md) rule 4). An unbracketed assertion needs code behind it, not just a bracketed number ([90-resumes.md](90-resumes.md)).
3. **The cut order** — under page pressure the ablation clause goes, then the kernels line, then the correctness line ([90-resumes.md](90-resumes.md)). The page keeps the decoder and the scheduler last, because they are the block's reason to exist.

So the page's strength is bounded by **what has shipped**, and the plan's own sequencing decides what that is first: the kernels bench and the decoder are finished artifacts before the gateway needs them ([04-inference-engine-pivot.md](04-inference-engine-pivot.md) §5), and the Stage-1 application gate is Phase 1 ([92-algorithms-and-career.md](92-algorithms-and-career.md)). Two consequences worth stating plainly:

- **The Stage-1 page cannot contain the scheduler, the kernels ladder, or the distributed work** unless the sequence changes. It carries the admission core, the correctness machinery, and whatever else has landed.
- **Editing the block cannot substitute for shipping.** A stronger-sounding bullet with no measurement behind it is the exact failure the claim-versus-code audit exists to catch, and it fails on the first probing question — which is a worse outcome than a thin page.

**The honest ceiling.** Three things in §2 are genuinely rare and worth protecting through every cut: the correctness machinery (a differential oracle plus output-equality under batching; most portfolios have no correctness claim at all), the kernel ladder tied to the decoder's bandwidth number on one machine under one harness, and the scheduler's token-identical invariant. Model scale, distributed systems, and production operations are *not* closed by this project, and no amount of planning closes them.

---

## 5. Definition of done

The project is finished when every item below is true, each verified by running something rather than by inspection.

**Gates (never cut):**

- The decoder's output matches the reference token-for-token on a fixed prompt, with the reference named and the weights and tokenizer shared.
- The scheduler's outputs are token-identical across batch compositions and thread counts.
- Fuzzed request streams produce zero divergences against the naive reference implementations — the phase-1 admission oracle and the phase-6 rung-0 scheduler.
- Replay is bit-identical across runs and thread counts, with FP non-associativity analyzed rather than assumed, and the boundary of that guarantee written down.
- The zero-allocation claim survives `LD_PRELOAD` interposition of both the C and the C++ allocation paths.
- TSan is clean under concurrent arrivals and completions.

**Artifacts that must exist:**

- `bench/` regenerates every number in the README; no number is transcribed by hand.
- A roofline figure with the decoder's achieved bandwidth plotted against the measured ceiling.
- A batching table with rung deltas, and an ablation table with each factor's p99.9 contribution.
- A design doc carrying the two latency budgets, the overload policy, the admission policy, and the final-versus-reference delta.
- An overload run at a stated multiple of capacity, with the shedding policy and breaker threshold named.

**Explain-loop gates (shipping requirements, not suggestions):**

- Orca's iteration-level scheduling and vLLM's paging, cold and spoken, reconciled against your own scheduler's choices.
- Why decode is bandwidth-bound and prefill is compute-bound, answered from your own achieved-bandwidth figure.
- The KV-budget arithmetic on a whiteboard, on your own machine, at your own context length.
- The distributed boundary script — the alpha–beta terms, the two all-reduces per layer, the per-interconnect crossover ([15-phase-5-docs.md](15-phase-5-docs.md)).

**Cut, not failed.** Items the master's cut order removes — chunked prefill, graph capture, preemption rungs, the second backend, load-generator realism — are named as designed-but-unbuilt. Never claimed. A rung that bought nothing is a finding worth keeping and stating.
