+++
date = '2026-11-15T15:35:25-07:00'
draft = false 
title = 'Some notes from Mojo GPU Puzzles'
summary = 'Working through Mojo GPU puzzles'
tags = ['mojo', 'gpu-kernels', 'gpu']
+++

# Introduction
Modular has a set of [GPU puzzles](https://puzzles.modular.com/introduction.html) for their [open-source systems language called Mojo](https://mojolang.org) that I have been working on to get more familiar with the language. I will write a longer post about my thoughts on Mojo later as I get more comfortable with it, but I thought I would write down my notes / worklog while working on these puzzles.

My solutions are [available here.](https://github.com/nikhilJain17/mojo-gpu-puzzles/tree/main)

### Problem 1: Map
- Use `unsafe_offset` for pointer arithmetic
- `TileTensor` makes index math (N-dim arrays, strides, padding, offsets) so simple
    - When writing `wgsl` kernels for llama.cpp and onnxruntime-web, this could be a big pain

### Problem 2: Zip
- There is an `unsafe_pointer` class for raw memory accesses, including GPU memory
- A `block` is another word for a `workgroup`

### Problem 3: Guards
- GPU threads are cheap to launch, and kernels work most efficiently with certain block sizes (e.g. 32, 64)
    - So launching extra / unused threads is fine for perf, but you still have to ensure correctness
- Gets more complex with 2D or 3D tensor layouts
- This is a fundamental pattern in GPU kernels

### Problem 4: 2D Map
- Again, 2D indexing is easier with `TileTensors`

### Problem 5: Broadcast
- broadcasting := reshaping lower-dim arrays to match higher-dim arrays for elementwise operations

### Problem 6: Blocks
- Blocks / workgroups are software abstractions
- Warps / subgroups (wgsl) are actual hardware units of threads that execute together
- The warp scheduler assigns warps to execute workgroups
- We can have multi-dimensional workgroups (3D in WGSL), and we use the same N-d indexing for workgroups as for tensors (nice!)

### Problem 7: 2D Blocks
- No notes

### Problem 8: Shared memory
- Workgroups have memory shared by all threads
- cooperative load := threads in a block cooperate to store values
- barrier := synchronization point for threads in a workgroup

### Problems 9, 10
- I had to skip these problems since they were only for NVIDIA GPUs

### Problem 11: Pooling
- Sliding window is a common pattern in GPU kernels, like applying a convolution filter
    - Leetcode came in handy after all!
- Don't forget to apply filters across cross-block values!

### Problem 12: Dot Product via Parallel Reduction
- Instead of reducing sequentially like in a single-threaded program, you can reduce in parallel!

### Problem 13: 1D Convolution
- For handling values that spill over blocks, just yank the extra values and store them in shared memory for your workgroup ahead of time

### Problem 14: Parallel Prefix Scan