# Phase 2 — Reflection: where does parallelism stop helping?

Copy this file to `submission/phase2_reflection.md`, fill in every answer slot, and commit it:

```bash
cp phase-2-tiny-gpt/reflection_template.md submission/phase2_reflection.md
```

**How to fill it in.** Each answer sits between a pair of `<!--answer:...-->` markers. Replace
the `TODO` line with your answer — **leave the markers themselves untouched**, they are how the
grader finds your answers. Anything you write outside the markers is ignored, so add extra
prose, tables or your threading plot freely.

Numbers in section 1 must **match `submission/phase2_report.json`** — copy them across, do not
retype from memory; they are checked against it.

---

## 1. Your measurements

| | Value |
|---|---|
| Naive attention, ms (`attention.naive_ms`) | <!--answer:attn_naive_ms-->243.568<!--/answer--> |
| Vectorized attention, ms (`attention.vectorized_ms`) | <!--answer:attn_vectorized_ms-->0.993<!--/answer--> |
| CPU training, ms for 100 steps (`devices.cpu_ms`) | <!--answer:cpu_ms-->441795.8<!--/answer--> |
| GPU training, ms for 100 steps (`devices.gpu_ms`) | <!--answer:gpu_ms-->13536.6<!--/answer--> |
| Final training loss (`training.final_loss`) | <!--answer:final_loss-->1.459<!--/answer--> |

**The thread sweep.** Give your four timings from `threads` — 1, 2, 4 and 8 threads — and say
which count was fastest. (verbatim: the numbers, in order)

<!--answer:thread_sweep-->
1 thread: 2562.528 ms; 2 threads: 2160.708 ms; 4 threads: 2113.507 ms; 8 threads: 2205.57 ms. Fastest: 4 threads.
<!--/answer-->

---

## 2. Intra-node parallelism, three ways

**Why vectorizing wins.** Both attention implementations do the same arithmetic and produce the
same result (the unit test checks that). Explain where the naive version's time actually goes,
and what the vectorized one does differently — in terms of the machine, not the Python.
(~70 words)

<!--answer:vectorization_why-->
Both versions do the same arithmetic, but the naive one works element by element in Python loops, so most of its time goes to interpreter overhead and repeated memory accesses instead of useful math. The vectorized version hands whole arrays to NumPy's BLAS library, which uses SIMD instructions to process several numbers per instruction and makes good use of the cache. That is why it took 0.993 ms against 243.568 ms, about 245x faster.
<!--/answer-->

**Where the thread curve flattens, and why.** Using your sweep: more threads stopped helping —
or started hurting — at some point. Say where, and give the reason. Your Colab runtime's vCPU
count is part of the answer; so is what the threads have to coordinate on. (~80 words)

<!--answer:threading_plateau-->
The time drops from 2562.5 ms at 1 thread to 2160.7 ms at 2 threads, then flattens: 4 threads gives 2113.5 ms, essentially the same, and 8 threads gets worse at 2205.6 ms. The Colab runtime only has 2 vCPUs, so beyond 2 threads there are no extra cores to run on and the threads just compete for the same ones. The threads also have to synchronize and share the same memory and cache, and that coordination ends up costing more than the extra threads save.
<!--/answer-->

**What made the GPU faster.** The model, the data and the step count were identical; only the
device changed. Say what about a transformer training step suits a GPU, and name one workload
that would *not* see this speed-up. (~70 words)

<!--answer:gpu_why-->
A transformer training step is dominated by large dense matrix multiplications, which split into thousands of independent multiply-adds. A GPU has thousands of arithmetic units that run them at the same time, while the CPU has only a couple of cores. This is why 100 steps took 13536.6 ms on the T4 against 441795.8 ms on the CPU, about 32.6x faster. A workload that would not benefit is one with sequential steps or heavy branching, where each step depends on the previous one.
<!--/answer-->

---

## 3. The question

**Where does parallelism stop helping, and why?** Pull the three experiments together. Your
answer should distinguish the *kinds* of limit you met — work that cannot be divided, hardware
you do not have, coordination that costs more than it saves, data that has to move — and say
which one bound each experiment. One concrete number from section 1 per claim. (~150 words)

<!--answer:parallelism_limit-->
Parallelism stops helping when extra workers cannot be used productively, and each experiment hit a different limit. Vectorizing worked almost perfectly because the arithmetic divides cleanly: attention fell from 243.568 ms to 0.993 ms, so the only limit was how many numbers one SIMD instruction handles. The thread sweep hit missing hardware and coordination cost: with only 2 vCPUs, 2 threads cut a step from 2562.5 ms to 2160.7 ms, but 8 threads was slower at 2205.6 ms because the threads spent more time synchronizing than computing. On the GPU, hardware was plentiful (13536.6 ms against 441795.8 ms on CPU, 32.6x), so the remaining limit is work that cannot be divided: the 2000 training steps still run one after another (278.3 s), because each step needs the weights updated by the previous one.
<!--/answer-->

**What you would measure next.** One experiment this phase did not run that would sharpen the
answer, and what result would change your mind. (~50 words)

<!--answer:next_experiment-->
I would repeat the thread sweep on a machine with 8 or more vCPUs instead of 2. If the time kept falling up to 8 threads, the plateau at 2113.5 ms was caused by missing hardware. If it still flattened or got worse, the real limit would be coordination between threads, not the number of cores.
<!--/answer-->
