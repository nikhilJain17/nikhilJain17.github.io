+++
date = '2026-10-20T15:35:25-07:00'
draft = false
title = 'Reducing Memory Usage for Llama.cpp in the Browser'
summary = 'Streaming weights to reduce memory usage'
tags = ['webgpu', 'llama.cpp', 'wllama', 'perf']
+++


### Motivation

Local inference is an exciting area of engineering and research, and the browser is a relatively new environment for running LLMs with the advent of WebGPU, a GPU programming framework deployed through the browser.

One challenge of running inference in the browser is the strict memory limitations imposed by browser vendors. The clip below shows a Safari tab running [Transformers.js](https://huggingface.co/docs/transformers.js/index), a popular WebGPU inference engine from HuggingFace. When using Transformers.js to run Llama-3.2-1b-fp16, we saw the Safari tab's memory usage spike to 20 GB before Safari killed the page.

{{< video src="/clips/wllama_streaming/transformers_oom_fast.mp4" >}}

Below is a memory usage figure from the paper showing the OOM from running a different model.

{{< figure src="/images/wllama_streaming/memory_comparison.svg" class="media-figure" alt="Memory usage over time for wllama, WebLLM, Transformers.js, and upstream wllama across four browser and hardware configurations. On Safari with an Apple M4 Pro, Transformers.js reaches an out-of-memory failure." caption="Our figure from the paper shows Transformers.js being killed by Safari due to OOM." >}}


So we must be careful about how much memory our browser-based inference engines use. If we can save memory usage, we can pass those savings on to load larger models, which are generally more capable than smaller models. So we should try to save memory wherever we can!

In this blog, we detail two memory-saving optimizations from our *Llamas on the Web* paper.

### Optimization #1: Streaming Model Weights from Hugging Face to Disk

Originally, wllama fetched model weights from Hugging Face and stored them in its custom file system called **HeapFS**. HeapFS is an in-memory file system that lives in the WebAssembly (WASM) heap.

Models were fetched and temporarily stored in the WASM heap, then loaded into GPU buffers via llama.cpp's weight loader.

But we can do better! Why not stream the weights from Hugging Face directly to disk, skipping the WASM heap?

{{< figure src="/images/wllama_streaming/wllama_streaming_sys_diag.png" class="media-figure" alt="Before: model weights flow from Hugging Face through wllama's WebAssembly heap into llama.cpp GPU buffers. After: weights flow from Hugging Face to a disk-backed file system, then into llama.cpp GPU buffers." >}}

#### Using OPFS

The [Origin Private File System (OPFS)](https://developer.mozilla.org/en-US/docs/Web/API/File_System_API/Origin_private_file_system) is a set of APIs available on mainstream browsers that provides tabs with private access to the machine's filesystem. We can use this to store models directly on disk for our wllama tab process.


#### Results

When we switched from storing models in the WASM-heap via HeapFS and stored them on disk via OPFS, we saw memory usage reductions across the board!

{{< figure src="/images/wllama_streaming/heapfs_opfs_memory_savings.png" class="media-figure memory-savings" alt="Steady-state memory usage with HeapFS and OPFS: total memory reductions of 13.8% for LFM2, 2.9% for Gemma 3, 15.7% for Phi 4, and 13.3% for Qwen 3.5." >}}



### Optimization #2: Streaming Model Weights from Disk to GPU Buffers

Now that we've optimized loading weights from Hugging Face to disk, can we optimize loading weights from disk to GPU buffers?

We use llama.cpp's weight loader to load tensors into GPU buffers, and it offers three loading modes:

1. Memory–mapped: `mmap` the tensors into main memory, and then create buffers from the memory region directly
2. Synchronous: load each tensor individually via a resizable scratch buffer 
3. Asynchronous: use a pre-allocated pool of four 1MB buffers concurrently

Reference: [llama.cpp:src/llama-model-loader.cpp](https://github.com/ggml-org/llama.cpp/blob/72d693e4fbf0a188172951fe667d5e2f354ee438/src/llama-model-loader.cpp)


#### Asynchronous Loading

The asynchronous loading mode uses a buffer pool of four 1 MB buffers, which bounds the intermediate staging memory to approximately 4 MB. The logic is in the llama.cpp [weight loader code here](https://github.com/ggml-org/llama.cpp/blob/72d693e4fbf0a188172951fe667d5e2f354ee438/src/llama-model-loader.cpp#L1569-L1631).


{{< figure src="/images/wllama_streaming/llama_streaming.png" class="media-figure" alt="Asynchronous loading streams model weights from wllama's OPFS file system through a pool of four llama.cpp scratch buffers into GPU buffers." >}}

#### Results

We saw even more memory savings by using the streaming weight loader!

{{< figure src="/images/wllama_streaming/opfs_llama_stream_memory_savings.png" class="media-figure memory-savings" alt="Warm-tab steady-state memory for HeapFS, OPFS, and OPFS plus streaming, respectively: LFM2 545, 470, 650 MB; Gemma 3 1580, 1528, 1257 MB; Phi 4 3815, 3208, 2748 MB; Qwen 3.5 2495, 2161, 1825 MB. Highlighted percentages show changes relative to HeapFS." >}}


Of course, we haven't eliminated all unnecessary memory usage, and mysteriously, memory usage for LFM2 went up. In particular, we still observed a mysterious cold-start memory spike in Safari, which turned out to be related to WebKit's WebAssembly JIT compiler. More on that investigation in a later blog post!

### Lessons Learned

#### 1. Peak Memory Matters More Than Steady-State Memory

Browsers impose memory limits on tabs, so even temporary allocations can cause a tab to be killed.

This is especially problematic with WebAssembly, whose linear memory can grow but generally cannot shrink. Temporary allocations can still therefore eat at the maximum memory budget.

#### 2. Streaming Turns Model-Sized Allocations into Bounded Allocations

Instead of staging an entire model in memory, streaming lets us process it in small chunks. This changes the scratch memory usage from being proportional to the model size, to being bounded by a fixed-size buffer pool.

In our case, asynchronous loading uses just four 1 MB buffers.

#### 3. Avoid Copies in Memory-Constrained Environments

If you are in a memory-constrained environment, be careful not to make unnecessary copies of data! That's an easy way to burn your memory budget without noticing. We noticed other WebGPU-based inference engines made that mistake, unnecessarily wasting memory budget.

<!-- TODO: Discuss tradeoffs between memory usage and loading latency. -->

<!-- TODO: Optional discussion of mmap on native systems and how browser storage differs. -->

### AI Disclosure
Code and blog post written by me, both lightly edited with Codex for correctness and feedback.
