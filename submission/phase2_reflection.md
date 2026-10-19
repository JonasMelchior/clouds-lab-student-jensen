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
| Naive attention, ms (`attention.naive_ms`) | <!--answer:attn_naive_ms-->24.008<!--/answer--> |
| Vectorized attention, ms (`attention.vectorized_ms`) | <!--answer:attn_vectorized_ms-->1.3<!--/answer--> |
| CPU training, ms for 100 steps (`devices.cpu_ms`) | <!--answer:cpu_ms-->427923.6<!--/answer--> |
| GPU training, ms for 100 steps (`devices.gpu_ms`) | <!--answer:gpu_ms-->12909.2<!--/answer--> |
| Final training loss (`training.final_loss`) | <!--answer:final_loss-->1.5069<!--/answer--> |

**The thread sweep.** Give your four timings from `threads` — 1, 2, 4 and 8 threads — and say
which count was fastest. (verbatim: the numbers, in order)

<!--answer:thread_sweep-->
threads -> ms: {'1': 2424.759, '2': 1981.093, '4': 2111.278, '8': 2346.417}
The count with 2 threads was the fastest
<!--/answer-->

---

## 2. Intra-node parallelism, three ways

**Why vectorizing wins.** Both attention implementations do the same arithmetic and produce the
same result (the unit test checks that). Explain where the naive version's time actually goes,
and what the vectorized one does differently — in terms of the machine, not the Python.
(~70 words)

<!--answer:vectorization_why-->
For 'attention_naive', the scaled dot product of Q and K is computed in a nested loop of T,T iterations, and the product of weight and K is computed with same complexcity. For the vectorized attention utilizes the machine utilizes SIMD functionality suppored by the CPU such that we can operate on matrixes and not only scalars.  
<!--/answer-->

**Where the thread curve flattens, and why.** Using your sweep: more threads stopped helping —
or started hurting — at some point. Say where, and give the reason. Your Colab runtime's vCPU
count is part of the answer; so is what the threads have to coordinate on. (~80 words)

<!--answer:threading_plateau-->
PyTorch splits each operation into chunks and runs them on separate threads, each doing the same work on different data (SPMD on MIMD hardware). The sweet spot is 2 threads, which matches the 2 vCPUs. Above 2, the threads have to share the two CPUs, so they compete for compute and the OS has to context switch between them. The threads also synchronize at the end of each operation, so if one is waiting for CPU time, the others wait too. That’s why 4 and 8 threads are slower. Even at 2 threads the speedup is only about 1.2×, because of synchronization overhead and shared memory bandwidth.
<!--/answer-->

**What made the GPU faster.** The model, the data and the step count were identical; only the
device changed. Say what about a transformer training step suits a GPU, and name one workload
that would *not* see this speed-up. (~70 words)

<!--answer:gpu_why-->
A transformer training step is mostly big matrix multiplications and elementwise ops, so the same operation is done on lots of independent data. A GPU runs this as kernels with thousands of threads, grouped into warps of 32 that execute the same instruction in lockstep. One that would not speed up: a small task where each step depends on the one before, like a loop with lots of if-statements.
<!--/answer-->

---

## 3. The question

**Where does parallelism stop helping, and why?** Pull the three experiments together. Your
answer should distinguish the *kinds* of limit you met — work that cannot be divided, hardware
you do not have, coordination that costs more than it saves, data that has to move — and say
which one bound each experiment. One concrete number from section 1 per claim. (~150 words)

<!--answer:parallelism_limit-->
For the fist naive/vectorization experiment, we removed naive overhad from 248 to 1.3 ms, what's left is moving Q,K,V through memory. The GPU was appr. 33x faster than the cpu (12909 ms vs 427923 ms) for a full training set, but is still limited by the fact that we need to do a kernel launch for each batch. For the intra-op experiment, the primary limitation was nr. of vCPUs - extra threads just shared the same vCPUs. Coordination also costs more here, since synchronization adds overhead and doesen't give a true 2x speedup, for e.g. 2 threads. The fact that training our model relies on sequential training steps, in which the weights are iterativally updated for each step, meaning the GPU also had to run all steps one after another. 
<!--/answer-->

**What you would measure next.** One experiment this phase did not run that would sharpen the
answer, and what result would change your mind. (~50 words)

<!--answer:next_experiment-->
I would train on the GPU with bigger batch sizes (16, 64, 256) and look at the time per step. If the time stays about the same, the GPU was mostly waiting for kernel launches and had spare capacity. If the time goes up as much as the batch does, the GPU was already working at full speed.
<!--/answer-->
