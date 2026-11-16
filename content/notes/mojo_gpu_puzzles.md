+++
date = '2026-11-15T00:00:00-08:00'
draft = false 
title = 'Some notes from Mojo GPU Puzzles'
summary = 'Working through Mojo GPU puzzles'
tags = ['mojo', 'gpu-kernels', 'gpu']
math = true
+++

# Introduction
Modular has a set of [GPU puzzles](https://puzzles.modular.com/introduction.html) for their [open-source systems language called Mojo](https://mojolang.org) that I have been working on to get more familiar with the language. I will write a longer post about my thoughts on Mojo later as I get more comfortable with it, but I thought I would write down my notes / worklog while working on these puzzles.

My solutions are [available here.](https://github.com/nikhilJain17/mojo-gpu-puzzles/tree/main)

**Jump to:** [15: Axis Sum](#problem-15-axis-sum) ·
[14: Parallel Prefix Scan](#problem-14-parallel-prefix-scan) ·
[13: 1D Convolution](#problem-13-1d-convolution) ·
[12: Dot Product via Parallel Reduction](#problem-12-dot-product-via-parallel-reduction) ·
[11: Pooling](#problem-11-pooling) ·
[10, 9](#problems-10-9) ·
[8: Shared memory](#problem-8-shared-memory) ·
[7: 2D Blocks](#problem-7-2d-blocks) ·
[6: Blocks](#problem-6-blocks) ·
[5: Broadcast](#problem-5-broadcast) ·
[4: 2D Map](#problem-4-2d-map) ·
[3: Guards](#problem-3-guards) ·
[2: Zip](#problem-2-zip) ·
[1: Map](#problem-1-map)

## Problem 15: Axis Sum

Basically multiple parallel reductions.

{{< figure src="/images/mojo_gpu_puzzles/axis_sum.png" class="media-figure" alt="Multiple parallel reductions, zooming in on one in particular." >}}

## Problem 14: Parallel Prefix Scan

{{< figure src="/images/mojo_gpu_puzzles/parallel_prefix_sum.png" class="media-figure" alt="Double offset at each iteration." >}}

This is the first algorithm where you really have to change your way of thinking from sequential to parallel. 
For an array of size $n$, we iterate $\lceil \log_2 n \rceil$ times to calculate $arr[i] = arr[i] + arr[i-offset]$ for $i \ge offset$, using values from the previous iteration and doubling $offset$ at each iteration.
The intuition is that at each step, the value at index $k$ knows about twice as many values as it did last iteration, looking backwards from index $k$.


Here is a brief sketch of a proof:

**Claim:** After iteration $k$, each thread $i$ holds the sum of up to $2^k$ elements in the range $[\max(0, i-2^k+1), i]$.

**Base case:** At iteration $0$, each thread $i$ only knows about its own value at index $i$ (i.e., the array is unmodified).

**Inductive step:** Assume the claim is true for iteration $k$. Then thread $i$ holds the sum of elements in the range

$$
[\max(0, i-2^k+1), i]
$$

and thread $i-2^k$ holds the sum of elements in the range

$$
[\max(0, (i-2^k)-2^k+1), i-2^k]
= [\max(0, i-2^{k+1}+1), i-2^k].
$$

Notice that these ranges are adjacent, nonoverlapping, and both contain $2^k$ elements (unless truncated due to indexing). So at iteration $k+1$, thread $i$ adds thread $i-2^k$'s sum to its current sum, obtaining the sum of elements in the range

$$
[\max(0, i-2^{k+1}+1), i].
$$

Thus, the claim holds for iteration $k+1$.

If $i < 2^k$, there is no thread $i-2^k$. However, thread $i$ already holds the sum of all elements from index $0$ to $i$, so the claim still holds.

**Conclusion:** After $\lceil \log_2 n \rceil$ iterations, $2^k \ge n$, so every thread $i$ holds the sum of elements from index $0$ to $i$. Therefore, the algorithm computes the correct prefix sums.

## Problem 13: 1D Convolution

- For handling values that spill over blocks, just yank the extra values and store them in shared memory for your workgroup ahead of time
{{< figure src="/images/mojo_gpu_puzzles/pooling_across_workgroups.png" class="media-figure" alt="Pooling." >}}

## Problem 12: Dot Product via Parallel Reduction

- Instead of reducing sequentially like in a single-threaded program, you can reduce in parallel!

{{< figure src="/images/mojo_gpu_puzzles/parallel_reduction.png" class="media-figure" alt="Halve the input array at each iteration." >}}

## Problem 11: Pooling

- Sliding window is a common pattern in GPU kernels, like applying a convolution filter
    - Leetcode came in handy after all!
- Don't forget to apply filters across cross-block values!

## Problems 10, 9

- I had to skip these problems since they were only for NVIDIA GPUs

## Problem 8: Shared memory

- Workgroups have memory shared by all threads
- cooperative load := threads in a block cooperate to store values
- barrier := synchronization point for threads in a workgroup

## Problem 7: 2D Blocks

- No notes

## Problem 6: Blocks

- Blocks / workgroups are software abstractions
- Warps / subgroups (wgsl) are actual hardware units of threads that execute together
- The warp scheduler assigns warps to execute workgroups
- We can have multi-dimensional workgroups (3D in WGSL), and we use the same N-d indexing for workgroups as for tensors (nice!)

## Problem 5: Broadcast

- broadcasting := reshaping lower-dim arrays to match higher-dim arrays for elementwise operations

## Problem 4: 2D Map

- Again, 2D indexing is easier with `TileTensors`

## Problem 3: Guards

- GPU threads are cheap to launch, and kernels work most efficiently with certain block sizes (e.g. 32, 64)
    - So launching extra / unused threads is fine for perf, but you still have to ensure correctness
- Gets more complex with 2D or 3D tensor layouts
- This is a fundamental pattern in GPU kernels

## Problem 2: Zip

- There is an `unsafe_pointer` class for raw memory accesses, including GPU memory
- A `block` is another word for a `workgroup`

## Problem 1: Map

- Use `unsafe_offset` for pointer arithmetic
- `TileTensor` makes index math (N-dim arrays, strides, padding, offsets) so simple
    - When writing `wgsl` kernels for llama.cpp and onnxruntime-web, this could be a big pain
