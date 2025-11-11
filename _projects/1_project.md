---
layout: page
title: HPC Project
published: true          # 초안 제외
show_on_home: true       # 홈 섹션 노출 허용 (버전에 따라 사용)
# 또는
visible: true            # 어떤 버전은 visible 사용
description: Optimizing GPT-2 on Multi-Node, Multi-GPU Environments
img: assets/img/hpc_gpt2_resized.png
importance: 1
category: work
related_publications: false
status: past
---

The goal of this project is to **optimize GPT-2 inference on a cluster consisting of four nodes, each equipped with two GPUs.** <br>
I utilized the MPI library to evenly distribute the given workload across the nodes and fully utilize the GPUs for optimal performance.
All computation kernels were implemented from scratch using CUDA and C.

The main workflow of the project is contained in the following four files: `main.cpp`, `tensor.cu`, `model.cu`, and `layer.cu`. <br>
In `main.cpp`, all the messy setup tasks, including initialization, MPI, and CUDA configurations, are handled. Then, the main generation function, generate_tokens(), is called. <br>
**generate_tokens()** iterates over the transformer blocks and invokes the corresponding groups of GPU kernels for inference.

<div class="row">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/transformer_loop.png" title="for-loop for transformer block execution" class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
    for-loop for transformer block execution
</div>

<!-- You can also put regular text between your rows of images, even citations {% cite einstein1950meaning %}. -->
The **transformer_block_batch_multi_gpu()** function consists of many subfunctions with finer-grained computations. <br>
Among them, mha_batch_multi_gpu() and ffn_batch_multi_gpu() incur most of the latency. <br>
These functions are also composed of more fine-grained operations. Ultimately, we can identify the main operation of the entire system — the GEMM (mha -> attn -> matmul_batch_cuda).

<div class="row">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/attn_func.png" title="The composition of the computation functions" class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
    The composition of the computation functions
</div>

The CUDA kernel for the GEMM operation is implemented as follows. <br>
First, I tile the matrix into blocks so that each thread block can execute a separate region of the GEMM. <br>
Then, I make each thread compute multiple elements within a block to reduce memory I/O. You can check the core part of this kernel design in the figure below. <br>
**Valuable Reference:** [https://siboehm.com/articles/22/CUDA-MMM](https://siboehm.com/articles/22/CUDA-MMM)

<div class="row">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/gemm_core.png" title=" Move blocks to shared mem, and a thread compute multiple elements for I/O efficiency" class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
    Move blocks to shared mem, and a thread compute multiple elements for I/O efficiency
</div>

Now let me introduce the most important optimization I made throughout this project — **KV caching**, which boosts the throughput to another level.
KV caching is a technique that saves the Key and Value states generated from the previous tokens and simply loads them during the current token generation.
Since this technique removes the need to recompute these heavy states, it significantly reduces the overall computation required during the decoding phase.

<div class="row">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/kv_cache.png" title="Concept of KV Caching" class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
    Concept of KV Caching
</div>

All the optimizations I applied to build the inference system for GPT-2 resulted in a throughput of approximately 65,000 tokens/sec.
The full codebase of this project is available on [GitHub](https://github.com/MaverickJune/HPC_GPT2_inference_optimization).

<div class="row">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/hpc_results.png" title="Final Throughput" class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
    Final Throughput
</div>



