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
Each bitvector is divided into 32-bit integers, and one bitvector is stored in a struct instance named `bvr`.

<div class="row justify-content-center">
  <div class="col-sm-auto mt-3 mt-md-0 text-center">
    {% include figure.liquid 
        loading="eager"
        path="assets/img/bvr_format.png"
        title="Struct template bvr (truncated at target-bit 6)"
        class="img-fluid rounded z-depth-1"
        style="width: 50%; height: auto; display: inline-block;" %}
  </div>
</div>
<div class="caption">
    Struct template bvr (truncated at target-bit 6)
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

