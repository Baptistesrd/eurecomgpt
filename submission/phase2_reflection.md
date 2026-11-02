# Phase 2 — Reflection: where does parallelism stop helping?

---

## 1. Your measurements

| | Value |
|---|---|
| Naive attention, ms (`attention.naive_ms`) | <!--answer:attn_naive_ms-->259.705<!--/answer--> |
| Vectorized attention, ms (`attention.vectorized_ms`) | <!--answer:attn_vectorized_ms-->1.231<!--/answer--> |
| CPU training, ms for 100 steps (`devices.cpu_ms`) | <!--answer:cpu_ms-->467029.7<!--/answer--> |
| GPU training, ms for 100 steps (`devices.gpu_ms`) | <!--answer:gpu_ms-->13238.6<!--/answer--> |
| Final training loss (`training.final_loss`) | <!--answer:final_loss-->1.5207<!--/answer--> |

<!--answer:thread_sweep-->
1 thread: 2575.643 ms, 2 threads: 2021.481 ms, 4 threads: 2186.202 ms, 8 threads: 3435.136 ms. The fastest was 2 threads.
<!--/answer-->

---

## 2. Intra-node parallelism, three ways

<!--answer:vectorization_why-->
The naive attention is basically a Python loop that computes the dot products one by one.
Each iteration goes through the Python interpreter, which is very slow and does not use the
SIMD instructions of the CPU at all. The vectorized version calls numpy, which under the
hood uses BLAS (OpenBLAS in my case, I checked with np.show_config). BLAS runs tight C
code with AVX2/AVX512 instructions, so one CPU instruction processes several numbers at
the same time. That is why I get 259 ms vs 1.2 ms, a 211x speedup.
<!--/answer-->

<!--answer:threading_plateau-->
On my sweep the best time was with 2 threads (2021 ms). From 2 to 4 the time already went
up a bit (2186 ms) and at 8 threads it was way worse (3435 ms). The reason is that Colab
free gives only 2 vCPUs. So asking for more than 2 threads does not add real cores, it just
creates fake parallelism and the threads start fighting for the same CPU. There is also
synchronization overhead between threads (locks, memory bandwidth), so going from 2 to 8
makes things slower, not faster.
<!--/answer-->

<!--answer:gpu_why-->
A transformer training step is almost only matmul (Q.K^T, softmax, V, projections, MLPs).
Matmul is exactly what a GPU is built for: thousands of CUDA cores that do the same
operation in parallel on different pieces of the matrix (SIMT model). On CPU 100 steps took
467 sec, on GPU only 13 sec, so 35x faster. A workload that would not benefit much is
something sequential or branch-heavy, like a Python parser or a recursive algorithm with
small inputs: the GPU would spend more time launching kernels and moving data than
computing.
<!--/answer-->

---

## 3. The question

<!--answer:parallelism_limit-->
Parallelism stops helping for different reasons in the three experiments. For the attention,
the limit was the CPU itself: the naive version did not use SIMD, so even on the same
hardware I was 211x slower (259 ms vs 1.2 ms). The problem was not the number of cores but
the kind of instructions used. For the thread sweep, the limit was the hardware I actually
have. Colab gives 2 vCPUs, so above 2 threads I only get contention: the time went from
2021 ms (2 threads) to 3435 ms (8 threads), it got worse because threads have to coordinate
and share the same cores. For the GPU, the speedup was big (35x, 467 sec to 13 sec) because
the work is massively parallel matmul, but even here it would stop helping if the batch was
too small, because the kernel launch and the host to device transfer cost would dominate.
<!--/answer-->

<!--answer:next_experiment-->
I would redo the GPU vs CPU comparison with different batch sizes (very small, medium,
very large). If the GPU speedup drops a lot at small batch, it proves that the GPU is
starved and the overhead of launching kernels dominates. That would show the GPU is not
always the right answer.
<!--/answer-->
