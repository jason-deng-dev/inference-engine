# Resumes — house style, discipline, and the two variants

> **Canonical master:** [`00-master.md`](00-master.md). Sizing, cut order, and operating rules live there — if this doc and the master disagree, the master wins.
> **Rule 4 applies here:** the resume describes work that exists. Until every bracket is filled, Appendix A and B are *targets*, not documents.

## House style

Match the Orderbook/Backtester entries: verb-led, mechanism stated, measured number attached. One idea per bullet — the constraint that binds is total page lines, not bullets per project; correctness second (the differentiator — almost no candidate claims fuzzed invariants + deterministic replay); ablation numbers are probe-bait for "how did you measure that?" questions; "loopback" keeps the feed claim honest.

**The inference-engine bullets live in Appendix B below** — that is their single home, so the wording can't drift between a draft block and the shipped page. Appendix A re-spines the same facts for quant roles: Orderbook, Backtester, and the engine framed as low-latency systems evidence. The per-phase resume lines in the phase docs are the *raw material* those bullets are cut from.

**Space-constrained fallback:** cut whole lines, never merge two claims into one bullet. Stacking ideas into a line is what makes a project block unreadable, and it saves less space than dropping a weak line does.

**Bracket rule:** any bullet with an unfilled placeholder ships as the fallback — an `[X]` on a resume is worse than a shorter line.

**Keyword discipline — what NOT to add:** industry terms you haven't built (FIX/OUCH parsing, DPDK, RDMA, FPGA) are probe-bait, not signal — every one converts a defensible bullet into a question you can't survive. They live in the design doc's industry-context Q&A and in interview answers attached to your measurements, never on the page.

**Conditional tier (v4.1) — keywords that graduate when the code lands, and only on the project that built them:** *kernel bypass* and *ITCH parsing* are no longer permanent exclusions. `41-network-ingestion.md` builds both on the Orderbook, so:

- **ITCH:** the feed bullet may say "ITCH-style" today, with "I modelled rather than parsed ITCH, here's the tradeoff" as the answer. Once the real file runs the same harness, it upgrades to "parsed NASDAQ ITCH 5.0 (MoldUDP64 framing, sequence-gap handling) and replayed a real message mix" — on the Orderbook block, with the boundary that it is a sample file, not a production feed handler.
- **Kernel bypass:** stays off the page until a rung ships, then appears **only on the Orderbook block**, in the form the rung supports ("AF_XDP user-space data path, copy mode" is a different and equally honest claim from "zero-copy"). It never appears on the engine block — that repo's ingress is loopback by design ([`00-master.md`](00-master.md) non-goals).
- **Still excluded regardless of anything:** DPDK and RDMA, unless built. Explaining them well is a design-doc answer, not a page claim.

---

## Claim ↔ code audit (2026-09-29)

Run against the repos on this machine by grepping the code each claim names. **Shipped** = the code exists and the sentence is defensible as written. **Target** = does not ship until the work lands. **Unverifiable** = no code on this machine to check against.

### Backtester

| Claim | Status | Evidence |
| --- | --- | --- |
| Serial and parallel samplers agree bit-for-bit | **shipped** | `tests/montecarlo_test.cc:385`, `serialAndParallelAgree` |
| Index-partitioned sampling, per-run seeding | **shipped** | `src/montecarlo.cpp:148-155`, `:177` (`std::mt19937 gen(seed + i)`) |
| Trade-level Monte Carlo, Markov regime switching | **shipped** | `include/montecarlo.h:14-31` (`Regime`, `RegimeData`, transition matrix) |
| Sharpe, max drawdown | **shipped** | `src/analytics.cpp:48`, `:137` |
| CAGR | **shipped** | `src/analytics.cpp:335`, `include/analytics.h:134` |
| `lock-free SPSC queues` | **removed** | zero `spsc`/`lock-free`/`ring`/`queue` hits, and no `mutex`/`atomic` anywhere in `include/` or `src/`. Corrected to index partitioning |
| `null ensembles separating signal from luck` | **target** | zero `null`/`permutation` hits |
| `point-in-time datasets → PyTorch → ONNX → walk-forward` | **target** | zero `torch`/`onnx`/`point-in-time`/`walk-forward`/`leakage` hits |
| GPU Monte Carlo (CUDA, T4), CPU↔GPU bit-identical | **target — no fallback** | zero `cuda`/`hip`/`__global__`/`nvcc`/`rocm` hits across `include src tests CMakeLists.txt CMakePresets.json` |
| Arrow/Parquet point-in-time dataset layer | **target** | zero `arrow`/`parquet` hits (2026-10-10); step B carries the format decision, [`20-ai-extension.md`](20-ai-extension.md) |

### Orderbook

| Claim | Status | Evidence |
| --- | --- | --- |
| p50 add 142ns, cancel 125ns, 3-level sweep 357ns | **stale — re-measure** | Those numbers exist only in `docs/prototype/benchmark/benchmark_results.md` (`BM_BuyEmptyBook` et al.) — the **pre-rewrite prototype**, measured through the old `Orderbook`/`Trader` API, which is different code from today's book. There is no `docs/benchmark/` at the repo root; the README's results table is empty pending the harness; and `benchmarks/main.cpp` is fully commented out, so `orderbook_bench` builds an empty binary. Do not ship this row. The replacement comes from the `Orderbook_Vector` vs `Orderbook_Pool` replay comparison (corrected 2026-10-10) |
| Pooled intrusive FIFO per level, O(1) best-quote, zero hot-path allocations | **target** | levels are a searchable vector in `orderbook_vector.h`; orders are `std::list<Order>` nodes; zero `pmr`/`intrusive`/pool hits. Ladder dropped 2026-10-10 (would be a fourth container rewrite — cap at [`40-deepening-queue.md`](40-deepening-queue.md)) |
| Generational slot map for the order-ID index | **target** | `idMap` is `std::unordered_map<OrderId, std::list<Order>::iterator>` — `orderbook_vector.h:39` |
| PMR-pooled intrusive FIFO per level | **target** | zero `pmr`/`intrusive`/pool hits |
| OFI / microprice / queue-position feature layer | **target** | zero `microprice`/`imbalance`/`queuePosition` hits |
| AF_XDP kernel-bypass ingestion | **target** | zero `xdp`/`af_xdp` hits; the spec and its rung ladder are [`41-network-ingestion.md`](41-network-ingestion.md) |
| Wire-to-wire, hardware-timestamped latency | **target** | no `SO_TIMESTAMPING`/PHC code; NIC capability unverified (`ethtool -T` is step 0) |
| Parsed NASDAQ ITCH 5.0 message mix | **target** | zero `itch`/`moldudp` hits; the feed is modelled, not parsed |
| Linear-vs-binary-vs-branchless search comparison | **implementation shipped, comparison still target** | `include/search_policies.h` ships all three; `orderbook_vector.h` is templated on the policy; `tests/search_policies_test.cc` passes (58 of the 123 in the suite). Only the head-to-head *comparison* is missing, and it is gated on the replay harness, not on the implementation (corrected 2026-10-10) |

### Inference engine (v4 pivot, adopted 2026-10-09)

| Claim | Status | Evidence |
| --- | --- | --- |
| All engine bullets — gateway, scheduler, decoder, kernels | **target** | the repo holds `TODO.md` and `docs/` only — there is no `src/`. The pre-trade risk-engine claims audited here previously are superseded wholesale; the pre-pivot spec is in git history before the v4 commit |

### GoodSoft work bullet 1

**Unverifiable from this machine.** No C++ Monte Carlo source exists under `/home/jason/repos`, and `github.com` is blocked from this session, so the pybind11/FastAPI engine cannot be checked. Two things to reconcile before it ships: the linked `automation-ecosystem` README documents **five** Docker services while the bullet claims **six** (the sixth is presumably this engine — make the README show it, or drop the count), and no C++ service appears in that README at all.

### Consequences

The Orderbook block currently ships one defensible bullet of four, and the GPU bullet ships none. Appendix A is a TARGET document and rule 1 covers `[N]` placeholders — but it does not cover an *unbracketed* assertion. `Cacheline-aligned, NUMA-pinned hot path` has no bracket, no number, and no code, and it reads as shipped. Extend rule 1: **an unbracketed assertion needs code behind it, not just a bracketed number.**

Separately, the C++ version claim has **inverted** since this was written (corrected 2026-10-10): `Orderbook/CMakeLists.txt:4` now sets `CMAKE_CXX_STANDARD 23`, so the README's `C++23` is correct and the resume's `C++17` is the stale side — the opposite of the note above. Fix the resume. The lesson matches the one this audit exists for: both claims were checked against a snapshot and neither was re-checked. **An audit that is not re-run is an assertion.**

The pipeline lead-in under **Projects** is itself one of these unbracketed assertions: it names four steps and all four are targets. It ships when the first link ships — step H, the Orderbook feature layer — not before.

---

## Appendix A — Resume, Quant Variant (TARGET — brackets must be filled)

Jason Deng
778-847-3750 | jasondeng.dev@gmail.com | linkedin.com/in/jason-deng-dev | github.com/jason-deng-dev
Canadian citizen | Fluent in English and Mandarin

**Education**
B.Sc. (Honours), Economics, Minor in Computer Science & Mathematics — University of Toronto, June 2025

**Work History**
**GoodSoft Co., Ltd., Osaka, Japan** — Software Engineer (June 2025 – May 2026); Technical Consultant (June 2026 – Present, retained post-handoff)
(github.com/jason-deng-dev/automation-ecosystem)

- Built a multithreaded C++17 Monte Carlo engine (bootstrap resampling, EWMA decay, Sharpe-ratio optimization) deriving content-type allocation weights from 60k+ observed outcomes; shipped to production via pybind11/FastAPI.
- Sole engineer on 6 services in production (AWS Lightsail, Docker, PostgreSQL, per-service DB isolation, GitHub Actions CI/CD): owned schema, deploys, and incident response for a year; hardened the public APIs with Redis rate limiting and a proxy layer.
- Built semaphore-bounded concurrent ingestion of 2,000+ products and a weekly race calendar — DeepL-translated, zero manual steps.

**Projects**

*Three systems, one standard: a matching engine, a statistical simulation engine, and a low-latency serving engine — nanosecond-scale latency measured with controls, determinism verified bit-for-bit.*

**Order Book Matching Engine — C++23** (github.com/jason-deng-dev/Orderbook)

- **Low-Latency Core:** Price-time priority matching engine in C++23 with self-trade prevention and partial-fill semantics; reverse-ordered contiguous price levels, best price at the back so book updates never shift the container, with the search policy as a compile-time template parameter.
- **Replay & Validation:** Implemented ITCH 5.0 ingestion with sequence-gap detection and a Synced/Recovering resync state machine, and a post-match apply path (execute/cancel/replace against resting orders) decoupled from the order-entry matching path.
- **Data Structure Optimization:** Replaced the std::map prototype with a reverse-ordered level vector (**[X]×** throughput on the [N]M-message replay) and the order index with a generational slot map (cancel p50: **[A→B]**ns); fills bit-identical across rewrites. Pooled slab with intrusive FIFO links removes the per-order heap node.
- **Hot-Path Optimization:** Zero hot-path allocation via a pre-committed slab arena and a hand-rolled free-list, verified by interposing malloc/free and operator new/delete rather than by inspection; tail latency attributed with per-op LLC and dTLB miss counts. Benchmarked 3 search strategies on the same replay, located the depth **[D]** crossover where linear search wins on hardware prefetching, confirmed against `perf stat` branch-miss and IPC counters.
- **Correctness:** Replays one capture through every backend and asserts identical book-state hashes plus continuous invariants (uncrossed book, per-level volume conservation, order-index consistency), catching **[C]** divergences the unit suite missed.
- **ML Feature Integration:** Zero-alloc microstructure feature layer (order-flow imbalance, microprice, queue-position estimate) in the hot path at **[X]** msgs/sec, feeding the backtester's signal inputs. *— gated on step H; do not ship until it exists.*
- **Kernel-Bypass Ingestion:** Replaced the socket feed with an AF_XDP user-space data path at rung [R]: kernel-stack cycles per message [A]→[B] (perf-verified), syscalls per message [N]→0, wire-to-book p99.9 [C]ns against NIC hardware timestamps; fills bit-identical to the socket path on the same recorded stream. *— rungs 0–1 need no special hardware; rungs 2–3 pause first.*

**Backtesting + Monte Carlo Engine — C++17** (github.com/jason-deng-dev/Backtester)

- **Orthogonal Architecture:** Strategy logic factored into Signal, Sizing, and Risk axes composed at runtime; zero-allocation CSV ingest via `std::from_chars`; FIFO lot accounting with per-position analytics (Sharpe, max drawdown, CAGR).
- **Execution Realism:** Depth-aware fills that consume visible book depth with slippage as a first-class output, plus TWAP child-order slicing — the simulator prices its own execution instead of assuming a fill.
- **Streaming Ingest:** SPSC ring as bar producer with the strategy loop as consumer, replacing the blocking full-file load; the Monte Carlo sampler stays index-partitioned parallel-for rather than queued, because a queue buys nothing where every path costs the same and writes one slot.
- **Statistical Rigor & Regime Awareness:** Trade-level Monte Carlo with percentile distributions over EV, max drawdown, and terminal equity; two-state regime classification with a fitted transition matrix for non-stationary dynamics; null-ensemble baselines separating signal from luck.
- **Parallel & GPU Execution:** Index-partitioned writes with per-run seeding across **[C]** cores — serial and parallel samplers agree bit-for-bit under test, **[S]×** speedup; path generation ported to CUDA (cloud T4) at **[N]×** the 16-thread CPU path with seeded replay bit-identical across CPU and GPU.
- **ML Data Pipeline:** Leakage-free point-in-time datasets stored as memory-mapped Parquet, handed to PyTorch zero-copy through the Arrow C Data Interface → ONNX export → walk-forward evaluation with null-ensemble validation.

**Low-Latency Inference Engine — C++17** (repo) — *framed for this page as systems evidence; the serving-layer bullets live in Appendix B*

- Sub-microsecond O(1) admission control on the request path (token-bucket rate limits, per-tenant concurrency caps, circuit breaker, restart-safe kill switch): [A]× under a mutex + unordered_map baseline measured on the same harness.
- Cacheline-aligned, NUMA-pinned hot path: zero hot-path allocations under malloc interposition, cross-node atomics [D]× a local atomic.
- Ablation on that layout: false-sharing elimination cut p99 [B]×, branchless checks cut branch misses [C]%.
- Verification: [10M] fuzzed request sequences differential-tested against a reference implementation with zero divergences; TSan-clean lock-free SPSC ledger with decisions bit-identical across runs and thread counts.
- Driven end-to-end from an open-loop load generator over a loopback TCP request stream; per-stage latency measured coordinated-omission-correct.

**Editing rules:** cut adjectives, tool-drops, redundant tails — never numbers, never "bit-identical", never method names. Cut order under pressure: engine bullet 5 (load generator) → engine bullet 3 (ablation) → backtester parallel bullet. The Orderbook ingestion bullet ([41](41-network-ingestion.md)) is the one line that is *first* to cut on page pressure and *last* to cut on content value: it is the strongest single line on this page if a rung ships, and it is nothing if none does. One page, always.

**Bracket fallbacks (current, per rule 1 — an unfilled `[X]` ships as the shorter line):** engine bullet 1 drops the `[A]× … same harness` parenthetical, keeping "sub-microsecond O(1) admission control". Engine bullet 2 drops the cross-node clause if `[D]` is unfilled, keeping the layout and interposition clauses. Engine bullet 3 drops the two ablation factors and the line goes with them. Engine bullet 4 drops the fuzz/differential clause if `[10M]` is unfilled, keeping the TSan/bit-identical clauses. Engine bullet 5 drops entirely until the load generator exists. Orderbook bullet 3 drops the `[125 → N]ns` parenthetical. Orderbook bullet 4 drops entirely if `[X]` is unfilled. Backtester bullet 3 (GPU) does not ship at all until step A exists in the repo: as of 2026-09-29 `grep -rniE "cuda|hip/|__global__|nvcc|rocm"` over `include src tests CMakeLists.txt` returns zero hits, so there is no fallback for that bullet, only deletion. Backtester bullet 4 drops entirely if `[S]×` and `[C]` are unfilled.

---

## Appendix B — Resume, AI-Infra Variant (same facts, re-spined)

Jason Deng
778-847-3750 | jasondeng.dev@gmail.com | linkedin.com/in/jason-deng-dev | github.com/jason-deng-dev
Canadian citizen

C++ systems engineer — low-latency inference serving, GPU computing, and performance engineering.
Skills: C++17/20 · CUDA · Python/PyTorch · ONNX Runtime · Linux (perf, NUMA, GDB, TSan) · TCP/UDP · CMake

**Education** — identical to Appendix A.

**Work History**

**GoodSoft Co., Ltd., Osaka, Japan** — Software Engineer (June 2025 – May 2026); Technical Consultant (June 2026 – Present, retained post-handoff)
(github.com/jason-deng-dev/automation-ecosystem)

- Built a multithreaded C++ Monte Carlo engine (bootstrap resampling, EWMA, Sharpe optimization) deriving allocation weights from 60k+ observations; shipped to production via pybind11/FastAPI.
- Sole engineer on a 6-service production platform (AWS, Docker, PostgreSQL, CI/CD): automated content and catalog operations end-to-end — eliminated routine manual ops and gave non-technical operators self-serve control without SSH; hardened public APIs with Redis rate limiting and a proxy layer.
- Built and operated a Claude-API content pipeline in production — prompt strategy and content weights tuned from historical performance data across 4 content types.

**Projects** (reordered: the engine leads — it is the flagship)

**Low-Latency Inference Engine — C++17/CUDA** (github.com/jason-deng-dev/inference-engine)

- End-to-end low-latency serving stack built as one system: from-scratch transformer decoder, continuous-batching scheduler, admission-control gateway, and a measured CUDA kernel study — single CMake build, every claim regenerable from `bench/`.
- Decoder: from-scratch GPT-2-class transformer (multi-head attention, KV cache behind a swappable allocator, greedy decode); output verified token-for-token against llama.cpp on fixed prompts; [X] tok/s single-stream, [N] ms TTFT at 2k context; decode at [B]% of peak DRAM bandwidth, roofline-analyzed against the kernel ladder's measured ceiling.
- Scheduler: iteration-level continuous batching with KV-budget admission (reject-at-capacity); outputs token-identical to the single-stream reference across batch compositions and thread counts; [N]× throughput at [B]× TPOT, explained by achieved decode bandwidth; TTFT/TPOT under concurrent load, coordinated-omission-correct.
- Gateway: sub-microsecond O(1) admission control (token-bucket rate limits, per-tenant concurrency caps, circuit breaker, restart-safe kill switch) over SPSC dispatch; cacheline-aligned, NUMA-pinned, zero hot-path allocations (LD_PRELOAD-verified); [A]× over a mutex baseline; perf c2c-verified false-sharing elimination cut p99 [B]×.
- Kernels: GEMM ladder naive→coalesced→shared-memory-tiled→register-blocked at [A]% of peak FP32 FLOPs and [B]% of peak DRAM bandwidth; occupancy sweep; one INT8 kernel with measured throughput gain and accuracy cost.
- Correctness: [10M]+ fuzzed request sequences differential-tested against a naive reference scheduler with zero divergences; deterministic replay bit-identical across thread counts (FP non-associativity analyzed, not assumed); TSan-clean.

**Backtesting + Monte Carlo Engine — C++17** (github.com/jason-deng-dev/Backtester)

- **Orthogonal Architecture:** Signal / Sizing / Risk axes composed at runtime; zero-alloc CSV ingest via `std::from_chars`; FIFO lot accounting with per-position statistics (Sharpe, max drawdown, CAGR, annualized volatility).
- **Execution Realism:** Depth-aware fills consuming visible book depth, slippage as a first-class output, TWAP child-order slicing — execution is priced by the simulator rather than assumed.
- **Streaming Ingest:** SPSC ring as bar producer, strategy loop as consumer, replacing the blocking full-file load; the Monte Carlo sampler is index-partitioned parallel-for instead, since a queue has no job where per-path cost is uniform and each path writes exactly one slot.
- **Statistical Rigor & Regime Awareness:** Trade-level Monte Carlo — percentile distributions over EV, max drawdown and terminal equity; two-state regime classification with a fitted transition matrix (`CalmToCalm`/`CalmToVol`/`VolToCalm`/`VolToVol`); null-ensemble baselines separating signal from luck.
- **Parallel & GPU Execution:** Index-partitioned writes with per-run seeding across **[C]** cores — serial and parallel samplers agree bit-for-bit under test, **[S]×** speedup; CUDA path generation (cloud T4) at **[N]×** CPU, Nsight-profiled, seeded replay bit-identical across CPU/GPU.
- **ML Data Pipeline:** Leakage-free point-in-time datasets as memory-mapped Parquet → PyTorch zero-copy via the Arrow C Data Interface → ONNX export → walk-forward evaluation with null-ensemble validation.

**Order Book Matching Engine — C++23** (github.com/jason-deng-dev/Orderbook)

- **Low-Latency Core:** Price-time priority matching engine in C++23; reverse-ordered contiguous price levels, best price at the back, search policy as a compile-time template parameter; self-trade prevention and partial-fill semantics.
- **Replay & Validation:** ITCH 5.0 ingestion with sequence-gap detection and a Synced/Recovering resync state machine; post-match apply path (execute/cancel/replace against resting orders) decoupled from the order-entry matching path.
- **Data Structure Optimization:** Replaced std::map with a reverse-ordered level vector (**[X]×** throughput) and the order index with a generational slot map (cancel p50: **[A→B]**ns); fills bit-identical across rewrites; pooled slab with intrusive FIFO links removes the per-order heap node.
- **Hot-Path Optimization:** Zero hot-path allocation via a pre-committed slab arena and hand-rolled free-list, verified by interposing malloc/free and operator new/delete; tail attributed with per-op LLC and dTLB miss counts. Benchmarked linear vs binary vs branchless search, located the depth **[D]** crossover where linear wins on hardware prefetching, confirmed via `perf stat` branch-miss and IPC counters.
- **Correctness:** Replays one capture through every backend, asserting identical book-state hashes plus continuous invariants (uncrossed book, per-level volume conservation, order-index consistency) — caught **[C]** divergences the unit suite missed.
- **ML Feature Integration:** Zero-alloc microstructure feature layer (order-flow imbalance, microprice, queue position) at **[X]** msgs/sec, feeding the backtester's signal inputs. *— gated on step H.*
- **Kernel-Bypass Ingestion:** AF_XDP user-space ingestion at rung [R]: kernel-stack cycles per message [A]→[B], syscalls per message [N]→0, wire-to-book p99.9 [C]ns against NIC hardware timestamps; fills bit-identical to the socket path on the same recorded stream; real ITCH 5.0 message mix replayed through the same harness. *— rungs 0–1 need no special hardware.*

**Editing rules:** the skills strip is the FIRST cut under space pressure (bullets carry the keywords). Never submit as "ML engineer" — boundary discipline applies to applications too. Engine-block cut order: correctness line → kernels line → gateway ablation clause. The decoder and scheduler bullets are the block's reason to exist; they go last.

**Engine bracket fallbacks:** every engine bullet is a target until the code exists. An engine bullet with an unfilled bracket ships as its shorter form or not at all — and the block's lead-in line ships only when the first two subsystems (kernels and decoder) are measurable.

**Orderbook ingestion bullet:** ships at the rung that was actually measured, and **names the rung** — "AF_XDP" with no mode is the claim an interviewer dismantles in one question ("copy or zero-copy?"). If no rung ships, the existing loopback wording stays and the bullet does not exist. Same for the ITCH clause: it appears only once a real file has run the same harness.

**Post-engine tier (v4.2) — the claims this block gains if the stretch tier ships, in the order they become claimable.** Each is a *target* until its measurement exists, and each is capped at one line: **ring all-reduce + the TP crossover** ("implemented ring all-reduce in C++ and measured the interconnect crossover where tensor parallelism stops paying — [X] µs latency term, [Y] GB/s bandwidth term, 1GbE → NVLink"); **architecture breadth** ("GPT-2-class and Llama-class decoders — RoPE, RMSNorm, SwiGLU, grouped-query attention — both behind a token-for-token gate"); **quantization** ("INT8 decoder path at [X]% throughput for [Y] perplexity delta, token-agreement [Z]% against the FP32 gate"); **speculative decoding** ("[X]× at batch 1, decaying to [~1]× at batch [N] — the technique and continuous batching pull the same knob in opposite directions"); **TP on the decoder** ("[N]-way tensor parallelism on a rented node, token-identical across rank counts"). Nothing here displaces the engine block; on page pressure these are the first lines cut, since the engine, the decoder, and the scheduler are the reasons the page exists.

**Underused asset (added 2026-09-28): the GoodSoft bullets are framed as automation work, and for AI-infra and platform roles they should be framed as distributed-systems operations work** — six services, the interfaces and failure modes between them, rate limiting and a proxy layer as defence in depth, and incident ownership as the retained consultant. It is the only entry on the page that proves you have operated something in production and been accountable for it when it broke, which is the thing most junior candidates cannot claim. It currently reads as a side note. One rewriting pass at application time: same facts, distributed-systems vocabulary, no invented specifics.

### Addendum — artifact placement (resolved by v4, 2026-10-09)

The v3 question — where the kernels bench and the decoder live once they exist — is answered by the pivot: they are subsystems of the **Inference Engine** project block, not separate blocks and not fold-into-Backtester clauses. The Backtester keeps its own block (GPU Monte Carlo + ML pipeline); the kernels ladder's numbers, the decoder's tok/s and TTFT, and the scheduler's batching tables are all engine bullets.

- **Skills strip:** "Nsight" and a kernel-optimization phrase may be added once A ships, and "KV cache / attention" once G ships. Not before. The strip stays the first cut under pressure.
- **Bracket discipline** applies to every engine bullet: the ladder's bandwidth and FLOPs percentages, the quantized kernel's accuracy delta, the decoder's tok/s and TTFT, the scheduler's throughput multiple, and every gateway ablation factor all ship as fallbacks until measured. A blocked module with no measured number is worse than omitting the bullet, because "roofline-analyzed" or "continuous batching" with nothing behind it is the easiest claim in this project to cross-examine.
