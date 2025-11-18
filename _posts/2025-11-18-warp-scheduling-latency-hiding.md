---
title: "NVIDIA GPU Warp Latency Hiding — How SM Partitions Actually Execute"
by: hdimmfh
date: 2025-11-18 01:40:00 +0900
categories: [GPU, Cuda]
tags: [Warp Scheduler, Latency Hiding, SM Partition, CUDA, Nsight]
---

🔍 **Understanding Warp Scheduling and Latency Hiding**

> “Many warps, four partitions — and only one warp per partition can run each cycle.”  
> This post breaks down how SM partitions actually execute warps, how latency hiding really works, and why warp eligibility matters more than warp count.

---

## ① Overview

Modern NVIDIA GPUs rely on **massive concurrency** rather than making a single warp fast.  
On A100:

- 4 SM partitions
- 1 warp issued per partition per cycle
- Max 4 executing warps per SM

Active warps (16–64+) only increase the chance that at least one warp is *eligible* when others stall.

---

## ② SM Partition Architecture (A100)

A single SM is divided into 4 independent partitions, each containing:

- Warp Scheduler  
- Dispatch Unit  
- Register File slice  
- L0/L1 cache slice  
- Execution pipelines (FP32 / INT / LDST / Tensor)

Each partition selects its own warp every cycle:

```
Partition 0 → warp A
Partition 1 → warp B
Partition 2 → warp C
Partition 3 → warp D
```

<div style="text-align:center;">
  <img src="https://github.com/hdimmfh/blog-img-repo/blob/main/img/gpu/component/warp-scheduler.gif?raw=true" width="350">
  <br>
  <em>Figure 1. Warp-Scheduler: Selected Warp.</em>
</div>

---

## ③ Warp States — Active, Eligible, Stalled, Selected

- Active :
    - Active = Eligible + Stalled + Selected
    - Warp is resident on the SM.
- Eligible — warp is ready to issue
- Stalled — warp is waiting (memory, dependency, pipeline)
- Selected — warp chosen by the scheduler this cycle

> ⚠️ Note: \
> A Selected warp is also conceptually `a subset of eligible warps`, but separating it provides a clearer view of the full state distribution. A stalled warp does not leave the SM; it simply cannot issue.
---

## ④ Cache Line Hit vs Cache Line Divergence

1. Case1 : `Same` Cache Line Hit (Ideal)
    - 32 threads hit same line  
    - 1 transaction  
    - Zero contention

2. Case2 : `Different` Cache Line Hits  
Even when all accesses are hits:
- Multiple load requests  
- L1 port contention  
- Extra cycles  

This is effectively **micro‑serialization inside the cache**.

---

## ⑤ Shared Memory: Why Bank Distribution Matters

![Bank Conflict](https://github.com/hdimmfh/blog-img-repo/blob/main/img/gpu/architecture/nvidia-sm-cache-memory-conflict.png?raw=true)
*Figure 2. Bank Conflict in Shared Memory per SM.*

1. Shared Memory (SMEM) is:
    - on-chip  
    - multi-banked  
    - extremely low latency  

2. When 32 threads hit `32 different banks`:
    - all banks operate in **parallel**  
    - zero cycles wasted  

Even if other warps hit the same bank, SMEM is so fast that cross‑warp conflicts are rarely harmful.
> ✔️ Within a warp → distribute accesses `across banks` \
> ✔️ Across warps → `low risk`; SMEM is conflict‑resistant

---

## ⑥ How Latency Hiding Works

Latency hiding = swap out a `stalled warp` for a `ready(eligible) warp`.

When a warp stalls:
```
if (eligible warp exists):
    scheduler issues it → latency hidden
else:
    no eligible warp → wasted cycle
```

Requirement for full throughput on A100:  
> Eligible warps ≥ 4** (one per partition)

Warp refill is instant:
> Warp X finishes → another becomes eligible.

---

## ⑦ Timeline Example

1. Cycle 10  
    - Warp 3 selected  
    - Warp 7 **eligible**
    - Warp 12 stalled  
2. Cycle 11  
    - Warp 3 stalls  
    - Warp 7 **becomes selected**

> ⚠️ Note: \
> If no warp is eligible → partition idle → SM underutilized.

---

## ⑧ Summary Table

| Concept | Meaning |
|--------|---------|
| Active Warp | Lives on SM |
| Eligible Warp | Ready to issue |
| Stalled Warp | Waiting |
| Selected Warp | Issued this cycle |
| Partition Rule | 1 warp per cycle |
| A100 Limit | 4 executing warps |
| Latency Hiding | Needs eligible warps |

---

## ⑨ TL;DR

- A100 SM = 4 independent partitions  
- Each partition issues **one warp per cycle**  
- True execution parallelism = **4 warps per SM**  
- Occupancy increases the chance of eligible warps  
- Latency hiding = replacing stalled warps instantly  
- No eligible warp → cycle wasted

Maintaining enough **eligible warps** is the real key to GPU performance.
