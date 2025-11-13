---
title: Allocated Active Blocks per SM — The Hidden Driver of Occupancy
by: hdimmfh
date: 2025-11-13 15:20:00 +0900
categories: [GPU, Cuda]
tags: [GPU, Cuda, Occupancy, AABS]
---

🔍 **Understanding Allocated Active Blocks per SM (AABS)**

## ① What Is AABS?

**AABS (Allocated Active Blocks per SM)** refers to the number of **thread blocks** that are *simultaneously allocated and actively running* within a single Streaming Multiprocessor (SM).  
It represents how the GPU hardware distributes parallel work units (blocks) across SMs.

> **Formula:**  
> `AABS = min(BlockLimit, RegisterLimit, SharedMemoryLimit, WarpLimit)`

That means the number of active blocks per SM is determined by the *most restrictive resource* among the four limits.

---

## ② Resource Constraints

![The ncu Command Result](https://github.com/hdimmfh/blog-img-repo/blob/main/img/gpu/cuda/local_ncu_result.png?raw=true)
*Figure 1. The ncu Command Result.*


| Limiting Factor | Description | Example (A100) |
|------------------|-------------|----------------|
| **Block limit** | Architectural maximum number of blocks per SM | 32 blocks |
| **Register limit** | Limited register file per SM, affected by registers per thread | 65,536 registers total |
| **Shared memory limit** | SM shared memory divided among active blocks | 164 KB total |
| **Warp limit** | Depends on block size (warps per block × blocks per SM ≤ 64) | 64 warps |

> The smallest of these values defines how many blocks can coexist on an SM.

---

## ③ Relationship to Occupancy

**Occupancy** is the ratio of *active warps* to *maximum warps per SM*:  

> `Occupancy = Active Warps / Max Warps per SM`

Since each block contains several warps, the number of **active blocks** directly determines **active warps** — hence **occupancy**.

```
Registers / Shared Memory / Warp Limit / Block Limit
                ↓
     Allocated Active Blocks per SM
                ↓
         Active Warps per SM
                ↓
          Achieved Occupancy
```

> In short: *resource limits → blocks → warps → occupancy*.

---

## ④ Example Calculation

For NVIDIA A100:

- Total registers: 65,536 per SM  
- Shared memory: 164 KB per SM  
- Max warps per SM: 64  
- Block size: 256 threads (8 warps)  
- Each thread uses 48 registers  
- Each block uses 24 KB shared memory

| Factor | Calculation | Limit |
|---------|--------------|--------|
| Registers | 65,536 / (256×48) ≈ 5 blocks | 5 |
| Shared Memory | 164 KB / 24 KB = 6 blocks | 6 |
| Warp | 64 / 8 = 8 blocks | 8 |
| Block Limit | 32 blocks | 32 |
| **Result** | **min(5,6,8,32) = 5 blocks/SM** | ✅ |

Thus:  
`Active Warps = 5 × 8 = 40`,  
`Occupancy = 40 / 64 = 62.5%`

---

## ⑤ Why It Matters

Higher **AABS** values mean more concurrent blocks → more active warps → higher potential occupancy.  
However, just like occupancy, **more is not always better** — excessive register or memory pressure may cause performance drops.

> ⚠️ Optimizing AABS involves balancing block size, register usage, and shared memory to maximize useful concurrency.

---

## Summary

| Concept | Description |
|----------|--------------|
| **AABS** | Number of active blocks allocated per SM |
| **Determined by** | Minimum of {Block, Register, Shared Memory, Warp limits} |
| **Related to** | Occupancy (active warps / max warps) |
| **Goal** | Maximize concurrency without resource exhaustion |
| **Tool** | Nsight Compute → `sm__active_blocks_per_multiprocessor` |

---

## References

- NVIDIA CUDA Programming Guide, v12.4  
- Nsight Compute Metrics: `sm__active_blocks_per_multiprocessor`, `sm__warps_active.avg.pct_of_peak_sustained_active`
- NVIDIA A100 Whitepaper — Architecture and Resource Partitioning
