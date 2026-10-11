# TODO

Checklist mirror of the plan. Canonical spec: [docs/00-master.md](docs/00-master.md) — item numbers below are its §4 (Phase 0); later phases cite their §. When this file and the plan disagree, the plan wins.

Done-state carried over from the previous list (2026-10-11).

---

## Unblocked now — the dead-zone fix

Three builds do **not** wait for CiA ch. 5 (plan §4, "dead-zone fix"). Your completed items already cover their prerequisites.

- [ ] **Tools block** (§4 item 30) — [perf playbook](docs/00-master.md) §14 Sections 1–3, then the learning layers; `man perf-stat`; Google Benchmark README (~2 h). Gates the harness.
- [ ] **Measurement harness** (§4 build 1) — `perf stat` + Google Benchmark wrapper, machine-readable output, seeded workloads, environment metadata logged. Prereq: OSTEP 13–23 ✅ + tools block.
- [ ] **False-sharing toy** (§4 build 2) — two threads on one cache line, before/after separation. First interview prop.
- [ ] **SPSC ring v1** (§4 item 18, first half) — all-`seq_cst`, TSan-clean, written *by faith* in seq_cst on purpose. Prereq: OSTEP 28 ✅ + CiA 1–3 ✅. **Unblocked today.**
- [ ] Benchmark hygiene applied to the harness (observable side effects / `steady_clock`, warmup separated / first-touch pre-touch / `sched_setaffinity` pinning)

## Phase 0 reading — in plan order

| # | Item | State |
| --- | --- | --- |
| 1–7 | OSTEP 1–2, 3 (optional), 4–6 (**ch. 6 anchor**), 7 summary, 8 partial, 9–10 summary, 11 skip | ✅ (as "OSTEP Virtualization") |
| 8 | CiA ch. 1–3 (∥ with OSTEP 13–23) | ✅ (as "CiA c1-4") |
| 9–13 | OSTEP 13–15 fully, 16 narrative, 17–21 fully (**ch. 21 = zero-alloc story**), 22–23 skim, CiA ch. 4 (**§4.3 full attention**) | ✅ |
| 14 | **OSTEP 28 — NON-NEGOTIABLE** | ✅ |
| 15 | Preshing memory-ordering posts (plan names 2; you read 4 — superset, done) | ✅ |
| 16 | McKenney, *Memory Barriers* §1–4 | ✅ |
| 17 | **CiA ch. 5 FULL incl. 5.3** — the one real wall; budget 2–3× normal time; re-read at Phase 1 | ⬜ **next** |
| 18 | SPSC ring **v2** — relax to acquire/release one change at a time, TSan + tests after each, written justification per relaxation in the commit message | ⬜ after 17 |
| 19 | CiA ch. 6 — §6.1 guidelines; implementations = reference, don't type | ⬜ ("C6→11b") |
| 20 | CiA ch. 7 — 7.1 + 7.3 properly; 7.2 literacy (hazard pointers, ref counting, ABA, busy-wait) | ⬜ |
| 21 | CiA ch. 8 — **8.2 read twice** (2nd pass when Phase 2 starts); 8.3–8.4 normal; 8.5 comprehension | ⬜ |
| 22 | CiA ch. 9 — thread pools; concepts over implementations | ⬜ |
| 23 | CiA ch. 10 — `std::execution` policies; know what you're refusing | ⬜ |
| 24 | CiA ch. 11 — designing for testability; read right before the Phase 1 gate | ⬜ |
| 25 | OSTEP 32 & 33 — read fully | ⬜ |
| 26 | OSTEP 31 — skim (semaphores: producer/consumer, readers/writers) | ⬜ |
| 27 | Stokes, *Inside the Machine* — optional but recommended, before Drepper (~3–4 evenings) | ⬜ |
| 28 | **Drepper §1–3** — read fully (~60 pp). §4–6 → Phase 2; §7 → superseded by the perf playbook; §8 → skipped | ⬜ |
| 29 | **COD part 1** — alongside Drepper §1–3, each COD section *after* the Drepper section it backs: 1.6/1.7/1.8 **first** (framing), then §2 ↔ 5.2 (skim), §3 ↔ 5.1 + 5.3 + 5.4 + **5.10 Cache Coherence**. Skip list in plan §4 item 29 | ⬜ |
| 30 | Tools block (see "Unblocked now") | ⬜ |

**Off-plan item on your old list:** *OSTEP 29.1* — the plan records OSTEP 26–30 as covered by CiA in C++ form and does not schedule it. Read it if a specific gap shows up in a cold-recall session; don't schedule it.

**Recorded — do not schedule:** OSTEP 24–25, 26–30, 34 · persistence 36–51 (optional 2 h skim of 36/39/42) · security 52–57.

**Anki discipline:** card what you'll use within weeks, not everything.

## Phase 0 exit

- [ ] **✅ Verify** — explain from *your own ring's* disassembly why x86 makes acquire/release nearly free but seq_cst needs a full fence; explain the toy's cache-miss spike mechanism in two sentences; ring v2 TSan-clean with a written justification for every memory order
- [ ] **Transfer checkpoint** out loud first, then written (§2 five prompts)
- [ ] First **cold-recall session** (§13) — 20 min, fresh chat, this file + plan attached

---

## Later phases — gates only, detail in the plan

- [ ] **A — CUDA + kernels bench** (§11 A1–A6). Gate: Phase 0. Kernels bench lives in this repo; GPU Monte Carlo in the Backtester
- [ ] **G — decoder** (§11 step G). Gate: A. Token-for-token gate against a reference running the same weights + tokenizer
- [ ] **Phase 1 — admission core + correctness** (§5). Unlocks Stage-1 applications ([90-career.md](90-career.md) §7)
- [ ] **Phase 2 — micro-architecture** (§6). Includes the 2-socket cloud day
- [ ] **Phase 3 — load generator + integration** (§7)
- [ ] **Phase 4 — zero-alloc + ablation** (§8). Unlocks the Orderbook slot map and PMR list
- [ ] **Phase 5 — docs + proof pack** (§9)
- [ ] **Phase 6 — scheduler** (§10). Gate: Phase 1 green + A shipped + G's gate passed
- [ ] **Sanitizer CI matrix** (§15.1) — in a gap, no hardware; before the Phase 1 fuzz gate is claimed
- [ ] **Anchor: AF_XDP ladder** (§16.2) — rung 0/1 + wire timestamps first
- [ ] **Anchor: H → B → C → D** (§11) — pause whenever the engine needs the hours

## Review rules that bind this file

- **Weekly (10 min):** planned vs. actual Toggl hours per category; a stalled category is next week's first priority
- **Cold recall (20 min/week):** §13. Amber/Red → Anki card; Green → `drills/answers/`
- **Review trigger:** if Phase 0–1 reading is not complete ~6 months from start, switch to applying with the current portfolio and continue the spine part-time
