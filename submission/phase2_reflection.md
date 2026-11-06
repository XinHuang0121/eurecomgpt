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
| Naive attention, ms (`attention.naive_ms`) | <!--answer:attn_naive_ms-->256.811<!--/answer--> |
| Vectorized attention, ms (`attention.vectorized_ms`) | <!--answer:attn_vectorized_ms-->1.056<!--/answer--> |
| CPU training, ms for 100 steps (`devices.cpu_ms`) | <!--answer:cpu_ms-->426650.4<!--/answer--> |
| GPU training, ms for 100 steps (`devices.gpu_ms`) | <!--answer:gpu_ms-->13038.6<!--/answer--> |
| Final training loss (`training.final_loss`) | <!--answer:final_loss-->1.4578<!--/answer--> |

**The thread sweep.** Give your four timings from `threads` — 1, 2, 4 and 8 threads — and say
which count was fastest. (verbatim: the numbers, in order)

<!--answer:thread_sweep-->
2686.569, 2141.014, 2237.639, 2736.65 ms/step for 1, 2, 4 and 8 threads, respectively. Two threads were fastest.
<!--/answer-->

---

## 2. Intra-node parallelism, three ways

**Why vectorizing wins.** Both attention implementations do the same arithmetic and produce the
same result (the unit test checks that). Explain where the naive version's time actually goes,
and what the vectorized one does differently — in terms of the machine, not the Python.
(~70 words)

<!--answer:vectorization_why-->
The naive version repeatedly enters small NumPy operations through nested Python loops, incurring interpreter dispatch, indexing and temporary-array overhead. Vectorization groups the arithmetic into matrix operations executed by optimized native kernels. NumPy reported OpenBLAS and supported SIMD extensions, consistent with efficient vector instructions and cache reuse. Time fell from 256.811 to 1.056 ms (243.19x). This measures the combined benefit of removing loop overhead and using optimized kernels, not SIMD alone.
<!--/answer-->

**Where the thread curve flattens, and why.** Using your sweep: more threads stopped helping —
or started hurting — at some point. Say where, and give the reason. Your Colab runtime's vCPU
count is part of the answer; so is what the threads have to coordinate on. (~80 words)

<!--answer:threading_plateau-->
The runtime reported 2 logical CPUs, with both available to the process. Two threads were fastest at 2141.014 ms/step, compared with 2686.569 for one. Four and eight threads took 2237.639 and 2736.65 ms/step. Beyond two threads, additional workers share the same available CPU capacity. Scheduling, synchronization and competition for caches and memory can outweigh additional parallel work. This pattern is consistent with oversubscription; the timings alone do not isolate memory bandwidth from coordination overhead.
<!--/answer-->

**What made the GPU faster.** The model, the data and the step count were identical; only the
device changed. Say what about a transformer training step suits a GPU, and name one workload
that would *not* see this speed-up. (~70 words)

<!--answer:gpu_why-->
Transformer training contains large matrix multiplications and many independent arithmetic operations across tokens, batches and features. These suit the T4 GPU, which can execute many operations concurrently. The 100-step runs took 426650.4 ms on CPU and 13038.6 ms on GPU, a 32.72x speedup. The CPU used the restored one-thread setting, not the fastest thread-sweep setting. Tiny batches or sequential, branch-heavy tasks may gain little because launch, transfer and coordination overhead can dominate.
<!--/answer-->

---

## 3. The question

**Where does parallelism stop helping, and why?** Pull the three experiments together. Your
answer should distinguish the *kinds* of limit you met — work that cannot be divided, hardware
you do not have, coordination that costs more than it saves, data that has to move — and say
which one bound each experiment. One concrete number from section 1 per claim. (~150 words)

<!--answer:parallelism_limit-->
Parallelism stops helping when remaining work or overhead becomes the bottleneck. Vectorizing attention reduced 256.811 ms to 1.056 ms, showing that repeated Python-level operations were expensive. This experiment does not establish what limits further vectorized speedups; serial setup and reductions still constrain scaling through Amdahl's law. In the thread sweep, 2141.014 ms with two threads beat 2736.65 ms with eight. Only two logical CPUs were available, so extra software threads added no CPU capacity; scheduling and shared-resource contention are plausible costs. GPU training reduced 426650.4 ms to 13038.6 ms for 100 steps, demonstrating useful parallel work at this batch size. CPU batch preparation, host-to-device copies, kernel launches and per-step loss reads still add overhead. Their individual contributions were not measured, so I cannot claim which dominates. The 32.72x comparison also uses the restored one-thread CPU baseline. These results show workload-dependent gains, not unlimited scaling or a universally optimal device.
<!--/answer-->

**What you would measure next.** One experiment this phase did not run that would sharpen the
answer, and what result would change your mind. (~50 words)

<!--answer:next_experiment-->
I would repeat the CPU/GPU comparison across batch sizes, using warm-up runs, repeated measurements and GPU synchronization. I would report time per token and compare against a two-thread CPU baseline. I expect GPU gains to shrink for tiny batches. Stable or increasing gains there would challenge my overhead-based explanation.
<!--/answer-->
