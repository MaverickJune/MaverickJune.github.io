---
layout: page
title: SBVR Kernels
published: true          # 초안 제외
show_on_home: true       # 홈 섹션 노출 허용 (버전에 따라 사용)
# 또는
visible: true            # 어떤 버전은 visible 사용
description: Hardware-Friendly Kernel Design of SBVR
img: assets/img/sbvr_fig1_resized.png
importance: 1
category: work
related_publications: false
status: ongoing
---

First, I recommend reading the full paper on SBVR available at [arXiv](https://arxiv.org/abs/2509.18172).
Here, I will discuss how we designed a GPU-friendly kernel for SBVR-formatted weights, focusing on the code-level details.

We defined a struct template to store the bitvectors in the SBVR format. 
Each bitvector is divided into 32-bit integers, and one bitvector is stored in a struct instance named `bvrs`.

<div class="row justify-content-center">
  <div class="col-sm-auto mt-3 mt-md-0 text-center">
    {% include figure.liquid 
        loading="eager"
        path="assets/img/bvr_format.png"
        title="Struct template bvrs (truncated at target-bit 6)"
        class="img-fluid rounded z-depth-1"
        style="width: 50%; height: auto; display: inline-block;" %}
  </div>
</div>
<div class="caption">
    Struct template bvrs (truncated at target-bit 6)
</div>

The interface of the kernel that actually computes the GEMV for the **SBVR format** is as follows.  
It takes the following inputs:  

- `l/r_bvr`: the set of bitvectors representing the LLM weights,  
- `l/r_coeff_cache`: the corresponding coefficients for the bitvectors in the SBVR representation,  
- `l/r_coeff_idx`: the indices indicating the coefficient positions for each bitvector in `coeff_cache`.


<div class="row justify-content-center">
  <div class="col-sm-auto mt-3 mt-md-0 text-center">
    {% include figure.liquid 
        loading="eager"
        path="assets/img/sbvr_gemv_interface.png"
        title="SBVR GEMV interface"
        class="img-fluid rounded z-depth-1"
        style="width: 50%; height: auto; display: inline-block;" %}
  </div>
</div>
<div class="caption">
    SBVR GEMV interface
</div>

The actual computation starts from the inner dimension, K (1×K, K×N, GEMV). Here, we divide the K dimension according to the size of the bitvector used for quantization (typically 128).  
The inner product is computed between two `K_PER_BVR*32`-sized vectors and accumulated to produce one final element of the GEMV. 
This process is executed in parallel across multiple thread blocks, each handling different parts of the outer dimension, N.

<div class="row justify-content-center">
  <div class="col-sm-auto mt-3 mt-md-0 text-center">
    {% include figure.liquid 
        loading="eager"
        path="assets/img/kernel_for_loop.png"
        title="Computation order of the kernel"
        class="img-fluid rounded z-depth-1"
        style="width: 50%; height: auto; display: inline-block;" %}
  </div>
</div>
<div class="caption">
    Computation order of the kernel
</div>

Now, here comes the interesting part. 
As mentioned in the paper, we replace the fused multiply-add operation in GEMV with a series of logical bitwise operations (`&` and `popcount`), followed by a reduction along the K dimension.

<div class="row justify-content-center">
  <div class="col-sm-auto mt-3 mt-md-0 text-center">
    {% include figure.liquid 
        loading="eager"
        path="assets/img/sbvr_inner_product_schematic.png"
        title="SBVR Inner Product Computation"
        class="img-fluid rounded z-depth-1"
        style="width: 50%; height: auto; display: inline-block;" %}
  </div>
</div>
<div class="caption">
    SBVR Inner Product Computation
</div>

This part can be implemented as follows. First, fetch the target bitvectors from the parameters `l_bvr` and `r_bvr`. Then compute the **popcount** of the bitwise **AND**.
As noted above, each bitvector is stored in the `bvrs` struct. In the innermost loop, take one `uint32_t` segment from each bitvector and accumulate the partial result `__popc(l_i & r_i)` into the `uchar4` array `pop_cache`. After the outer loop sweeps the K dimension—covering all `K_PER_BVR` segments—the `pop_cache` contains all combinations of the logical-operation results for the participating bitvectors.

<div class="row justify-content-center">
  <div class="col-sm-auto mt-3 mt-md-0 text-center">
    {% include figure.liquid 
        loading="eager"
        path="assets/img/sbvr_inner_logic_bitwiseops.png"
        title="Handling Bitwise-Operations"
        class="img-fluid rounded z-depth-1"
        style="width: 50%; height: auto; display: inline-block;" %}
  </div>
</div>
<div class="caption">
    Handling Bitwise-Operations
</div>

Finally, we multiply the corresponding coefficients of the bitvectors with the logical-operation results stored in `pop_cache`.  
The final reduction result is accumulated in a variable named `sum`, and the threads within a warp—each computing different parts of the K dimension—are reduced using `__shfl_down_sync()`.  
The outcome of these computations is then written to the output vector `out`. <br>  
This concludes the computation of the SBVR GEMV kernel.

<div class="row justify-content-center">
  <div class="col-sm-auto mt-3 mt-md-0 text-center">
    {% include figure.liquid 
        loading="eager"
        path="assets/img/sbvr_final_reduction.png"
        title="Final Reduction"
        class="img-fluid rounded z-depth-1"
        style="width: 50%; height: auto; display: inline-block;" %}
  </div>
</div>
<div class="caption">
    Final Reduction
</div>

You can find more details about the SBVR implementation—such as the parameters used to launch the kernel (grid and block sizes) and how this kernel is integrated into the SBVR forward pass—on [GitHub](https://github.com/MaverickJune/sbvr).

