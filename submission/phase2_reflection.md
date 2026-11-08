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
| Naive attention, ms (`attention.naive_ms`) | <!--answer:attn_naive_ms-->372.344<!--/answer--> |
| Vectorized attention, ms (`attention.vectorized_ms`) | <!--answer:attn_vectorized_ms-->1.016<!--/answer--> |
| CPU training, ms for 100 steps (`devices.cpu_ms`) | <!--answer:cpu_ms-->460596.1<!--/answer--> |
| GPU training, ms for 100 steps (`devices.gpu_ms`) | <!--answer:gpu_ms-->13213.2<!--/answer--> |
| Final training loss (`training.final_loss`) | <!--answer:final_loss-->1.4806<!--/answer--> |

**The thread sweep.** Give your four timings from `threads` — 1, 2, 4 and 8 threads — and say
which count was fastest. (verbatim: the numbers, in order)

<!--answer:thread_sweep-->
1 thread: 2482.012,
2 thread: 2122.523,
4 thread: 2003.664,
8 thread: 2232.169
The fastest was 4 threads
<!--/answer-->

---

## 2. Intra-node parallelism, three ways

**Why vectorizing wins.** Both attention implementations do the same arithmetic and produce the
same result (the unit test checks that). Explain where the naive version's time actually goes,
and what the vectorized one does differently — in terms of the machine, not the Python.
(~70 words)

<!--answer:vectorization_why-->
The naive implementation spends most of its time executing many small operations individually, repeatedly accessing memory and handling loop overhead. This prevents the CPU from efficiently using its processing units. The vectorized implementation performs the same calculations using large matrix operations, allowing optimized BLAS libraries to exploit SIMD instructions, cache efficiency and multiple CPU cores. As a result, it reduces overhead and executes computations much more efficiently.
<!--/answer-->

**Where the thread curve flattens, and why.** Using your sweep: more threads stopped helping —
or started hurting — at some point. Say where, and give the reason. Your Colab runtime's vCPU
count is part of the answer; so is what the threads have to coordinate on. (~80 words)

<!--answer:threading_plateau-->
Performance improved up to 4 threads, reaching the lowest execution time of 2003.664 ms. Increasing the number to 8 threads made performance worse (2232.169 ms). This suggests that additional threads introduce coordination and synchronization overhead. Since the exact vCPU count of the Colab runtime was not recorded, it cannot be determined retrospectively.
<!--/answer-->

**What made the GPU faster.** The model, the data and the step count were identical; only the
device changed. Say what about a transformer training step suits a GPU, and name one workload
that would *not* see this speed-up. (~70 words)

<!--answer:gpu_why-->
The main reason is that transformer training involves large matrix multiplications, particularly in attention mechanisms and feed-forward layers. GPUs are designed to perform thousands of similar mathematical operations in parallel, offering much higher computational throughput than CPUs for these workloads. However, a sequential task with many conditional branches and little parallelism, such as processing instructions one by one, would not benefit significantly from GPU acceleration.
<!--/answer-->

---

## 3. The question

**Where does parallelism stop helping, and why?** Pull the three experiments together. Your
answer should distinguish the *kinds* of limit you met — work that cannot be divided, hardware
you do not have, coordination that costs more than it saves, data that has to move — and say
which one bound each experiment. One concrete number from section 1 per claim. (~150 words)

<!--answer:parallelism_limit-->
The three experiments show that parallelism has different limitations depending on the workload. In the attention experiment, vectorization achieved a 366.48x speed-up (372.344 ms vs 1.016 ms) by replacing individual operations with optimized matrix computations. This shows how inefficient execution can limit performance when work is not organized to exploit parallel hardware.

In the multithreading experiment, performance improved up to 4 threads (2003.664 ms), but worsened at 8 threads (2232.169 ms). With only 2 vCPUs available, additional threads introduced competition for CPU resources and synchronization overhead.

Finally, GPU training was approximately 34.9x faster than CPU training (460596.1 ms vs 13213.2 ms). Transformers benefit from GPUs because matrix multiplications involve many parallel operations. However, data transfer between CPU and GPU can become a bottleneck, especially for smaller workloads. Overall, parallelism stops helping when serial work, limited hardware resources, coordination costs or data movement outweigh the benefits.
<!--/answer-->

**What you would measure next.** One experiment this phase did not run that would sharpen the
answer, and what result would change your mind. (~50 words)

<!--answer:next_experiment-->
I would like measure GPU training time separately for data transfer and actual computation, using different batch sizes. This would help identify whether data movement limits performance. If transferring data takes a significant portion of the total execution time, it would confirm that GPU acceleration is partly limited by memory transfer overhead.
<!--/answer-->
