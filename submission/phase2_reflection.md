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
| Naive attention, ms (`attention.naive_ms`) | <!--answer:attn_naive_ms-->248.933<!--/answer--> |
| Vectorized attention, ms (`attention.vectorized_ms`) | <!--answer:attn_vectorized_ms-->1.054<!--/answer--> |
| CPU training, ms for 100 steps (`devices.cpu_ms`) | <!--answer:cpu_ms-->423622.7<!--/answer--> |
| GPU training, ms for 100 steps (`devices.gpu_ms`) | <!--answer:gpu_ms-->14054.8<!--/answer--> |
| Final training loss (`training.final_loss`) | <!--answer:final_loss-->1.4991<!--/answer--> |

**The thread sweep.** Give your four timings from `threads` — 1, 2, 4 and 8 threads — and say
which count was fastest. (verbatim: the numbers, in order)

<!--answer:thread_sweep-->
1 thread: 2293.979 ms, 2 threads: 2055.42 ms, 4 threads: 2445.775 ms, 8 threads: 2333.623 ms. Fastest: 2 threads.
<!--/answer-->

---

## 2. Intra-node parallelism, three ways

**Why vectorizing wins.** Both attention implementations do the same arithmetic and produce the
same result (the unit test checks that). Explain where the naive version's time actually goes,
and what the vectorized one does differently — in terms of the machine, not the Python.
(~70 words)

<!--answer:vectorization_why-->
Basic Python loops for attention run 65,536 slow operations one by one due to interpreter overhead. Vectorized NumPy handles Q @ K^T via compiled OpenBLAS with AVX2 and FMA instructions for parallel SIMD processing. Same math, but time drops from 248.933 ms to 1.054 ms for a 236x speedup.
<!--/answer-->

**Where the thread curve flattens, and why.** Using your sweep: more threads stopped helping —
or started hurting — at some point. Say where, and give the reason. Your Colab runtime's vCPU
count is part of the answer; so is what the threads have to coordinate on. (~80 words)

<!--answer:threading_plateau-->
In my sweep, 2 threads was the fastest (2055.42 ms vs 2293.979 ms with 1 thread), but only by about 12%. With 4 threads (2445.775 ms) and 8 threads (2333.623 ms) it was actually slower than with 1. When I checked with lscpu, Colab gives 2 vCPUs but they are just two hyperthreads of one physical core, so they share the same execution units and the second thread doesn't add much. With 4 or 8 threads there are more threads than CPUs, so they have to take turns and wait for each other after every matrix operation, and that overhead costs more than it saves.
<!--/answer-->

**What made the GPU faster.** The model, the data and the step count were identical; only the
device changed. Say what about a transformer training step suits a GPU, and name one workload
that would *not* see this speed-up. (~70 words)

<!--answer:gpu_why-->
Training a transformer is mostly big matrix multiplications, and each value in the output can be computed independently, so the work splits easily across the thousands of small cores on the T4. That's why 100 steps took 14054.8 ms on the GPU against 423622.7 ms on the CPU, about 30x faster. Something that would not get this speed-up is small sequential work, like generating text one token at a time with batch size 1, because there isn't enough parallel work and the time goes into launching kernels and copying data to the GPU.
<!--/answer-->

---

## 3. The question

**Where does parallelism stop helping, and why?** Pull the three experiments together. Your
answer should distinguish the *kinds* of limit you met — work that cannot be divided, hardware
you do not have, coordination that costs more than it saves, data that has to move — and say
which one bound each experiment. One concrete number from section 1 per claim. (~150 words)

<!--answer:parallelism_limit-->
Each experiment ran into a different kind of limit. For attention, the problem was doing the work one piece at a time in Python, and once it was vectorized it went from 248.933 ms to 1.054 ms, so the hardware was never the limit there. For threads, the limit was hardware I didn't have: Colab only has one real core with two hyperthreads, so 2 threads only went from 2293.979 ms to 2055.42 ms, and with 4 threads (2445.775 ms) coordination between threads made it slower. The GPU gave the biggest gain, from 423622.7 ms to 14054.8 ms, but it didn't reach the 50x the notebook mentions. I think that's because data still has to be copied to the GPU every step, the model is small so many kernels are short, and the training loop itself runs in Python on the CPU. That part can't be parallelised, so as Amdahl's law says, it caps the total speed-up.
<!--/answer-->

**What you would measure next.** One experiment this phase did not run that would sharpen the
answer, and what result would change your mind. (~50 words)

<!--answer:next_experiment-->
I would run the CPU vs GPU comparison again with bigger batch sizes, like 16, 64 and 256. If the GPU speed-up grows a lot with bigger batches, it means the GPU was underused with my settings. If it stays around 30x, then the GPU was already being used well and the limit is somewhere else.
<!--/answer-->