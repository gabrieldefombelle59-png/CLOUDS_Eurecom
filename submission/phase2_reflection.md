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
| Naive attention, ms (`attention.naive_ms`) | <!--answer:attn_naive_ms-->246.699<!--/answer--> |
| Vectorized attention, ms (`attention.vectorized_ms`) | <!--answer:attn_vectorized_ms-->0.992<!--/answer--> |
| CPU training, ms for 100 steps (`devices.cpu_ms`) | <!--answer:cpu_ms-->434812.6<!--/answer--> |
| GPU training, ms for 100 steps (`devices.gpu_ms`) | <!--answer:gpu_ms-->13251.2<!--/answer--> |
| Final training loss (`training.final_loss`) | <!--answer:final_loss-->1.521<!--/answer--> |

**The thread sweep.** Give your four timings from `threads` — 1, 2, 4 and 8 threads — and say
which count was fastest. (verbatim: the numbers, in order)

<!--answer:thread_sweep-->
1 thread: 2314.168
2 threads: 1988.514
4 threads: 2665.338
8 threads: 2291.955
The fastest count was 2 threads.
<!--/answer-->

---

## 2. Intra-node parallelism, three ways

**Why vectorizing wins.** Both attention implementations do the same arithmetic and produce the
same result (the unit test checks that). Explain where the naive version's time actually goes,
and what the vectorized one does differently — in terms of the machine, not the Python.
(~70 words)

<!--answer:vectorization_why-->
The time of the naive version is mostly spent on the Python interpreter overhead (dynamic type checking), and on fetching scattered variables from RAM. The vectorized version pushes the math down to a low level C library (BLAS) that makes use of the CPU's SIMD (Single Instruction, Multiple Data) registers. This enables the CPU hardware to perform the same arithmetic operation on multiple adjacent memory blocks in parallel in one clock tick.
<!--/answer-->

**Where the thread curve flattens, and why.** Using your sweep: more threads stopped helping —
or started hurting — at some point. Say where, and give the reason. Your Colab runtime's vCPU
count is part of the answer; so is what the threads have to coordinate on. (~80 words)

<!--answer:threading_plateau-->
The curve flattens and performance starts degrading right after 2 threads. The standard colab T4 runtime only has 2 physical vCPUs. While 2 threads map perfectly to this hardware, forcing 4 or 8 threads forces the OS scheduler to constantly pause, swap and resume threads (context switching). And on top of that you have multiple threads locking and coordinating access to shared memory arrays . There is a coordination overhead that far exceeds any theoretical parallel gains .
<!--/answer-->

**What made the GPU faster.** The model, the data and the step count were identical; only the
device changed. Say what about a transformer training step suits a GPU, and name one workload
that would *not* see this speed-up. (~70 words)

<!--answer:gpu_why-->
Training transformers is dominated by large dense matrix multiplications. A GPU consists of thousands of light-weight ALU cores designed to execute these independent, massively parallel arithmetic operations simultaneously. An example of a workload that would not see this speed-up is a heavily sequential algorithm with deep branching logic such as parsing complex JSON structures or recursive depth-first tree traversals where the next step is entirely dependent on the previous one.
<!--/answer-->

---

## 3. The question

**Where does parallelism stop helping, and why?** Pull the three experiments together. Your
answer should distinguish the *kinds* of limit you met — work that cannot be divided, hardware
you do not have, coordination that costs more than it saves, data that has to move — and say
which one bound each experiment. One concrete number from section 1 per claim. (~150 words)

<!--answer:parallelism_limit-->
Parallelism ceases to help when the overhead of coordination, the movement of data or hard limits of the hardware outstrip the gains from computation. In the vectorization experiment we skipped the interpreter and obtained a huge speedup (0.992 ms vs 246.699 ms). Here, the parallelism is ultimately constrained by the physical width of the CPU’s SIMD registers and the speed with which contiguous data can be moved from cache.

In the threading experiment we encountered a hard limit of hardware we do not have: performance deteriorated at 4 threads (2665.338 ms) because the machine only has 2 vCPUs. The time spent context-switching and coordinating locks on shared memory was more expensive than the parallelism gained .

Finally, despite the GPU being massively faster than the CPU (13251.2 ms vs 434812.6 ms) due to its thousands of parallel cores, it is still bottlenecked by data movement, specifically the latency of copying training batches from the CPU system RAM to the GPU VRAM over the PCIe bus (Amdahl’s Law).
<!--/answer-->

**What you would measure next.** One experiment this phase did not run that would sharpen the
answer, and what result would change your mind. (~50 words)

<!--answer:next_experiment-->
I would measure how incrementally scaling the batch size impacts GPU step time versus CPU step time. If scaling the batch size heavily increases GPU step time without improving overall throughput, it would prove we are bound by PCIe data transfer bottlenecks rather than compute limits.
<!--/answer-->
