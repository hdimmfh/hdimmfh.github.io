---
title: GPU Memory(1) — Architecture
by: hdimmfh
date: 2025-11-06 10:00:00 +0900
categories: [GPU, GPU Network]
tags: [GPU, HBM, CUDA, Memory, Pipeline]
---

🔍**Understanding the GPU Memory Network**

In the [previous post](https://hdimmfh.github.io/posts/inside-gpu-command-pipeline),  
we traced **how commands** travel — from the CUDA driver to SMs.  
Now, to understand **how operands (data)** move,  
we must first understand the **GPU memory network** that physically delivers that data.

Because no instruction runs without its operands,  
understanding the *data path* is just as essential as the *command path.*

---

## ① Big Picture — The Memory Highway

![NVIDIA A100 GPU Architecture Diagram](https://github.com/hdimmfh/blog-img-repo/blob/main/img/gpu/architecture/CSP-multi-user-with-MIG.png?raw=true)
*Figure 1. NVIDIA A100 GPU Architecture (MIG Example).*

![Memory hierarchy](https://github.com/hdimmfh/blog-img-repo/blob/main/img/gpu/architecture/memory-hierarchy-in-gpus.png?raw=true)
*Figure 2. Memory hierarchy in GPUs.*

From compute cores down to physical DRAM:

```
SMs → L2 Cache → Memory Controllers → HBM Stacks
```

Every byte fetched by a CUDA kernel flows through this route.  
Each layer adds coordination, bandwidth, and parallelism.

---

## ② Streaming Multiprocessor (SM) — Where Computation Starts

```
SM = { CUDA Cores + Tensor Cores + Warp Scheduler + L1 Cache + Registers }
```

- The **SM (Streaming Multiprocessor)** executes kernel instructions.
- Each SM runs thousands of threads in groups of 32 (warps).
- When a warp stalls on memory, another warp is scheduled — maximizing utilization.
- The **L1 cache and shared memory** sit closest to execution units.
- Operands first come from registers, then from L1, then from global memory (via L2).

---

## ③ GMMU & L2 Cache — Translating and Buffering

```
SM → GMMU → L2 Cache → Memory Controller
```

- The **GMMU (GPU Memory Management Unit)** converts **virtual addresses (GPU VA)** into **physical addresses (PA)**.  
- After translation, the data request hits **L2 Cache**, which is **shared by all SMs**.  
- The L2 acts as a crossroad — merging and reordering requests before they go to memory controllers.  
- On a GPU like NVIDIA A100, the L2 is divided into slices connected to each controller via a **high-speed on-die fabric.**

---

## ④ Memory Controller — The Traffic Dispatcher

```
Controller count = Channel count
```

- Each **Memory Controller** corresponds to **one HBM channel**.  
- It translates memory requests into DRAM-level commands (`ACT`, `READ`, `WRITE`, `PRE`).  
- It handles timing for **bank groups** and **banks**,  
  ensuring proper precharge and activation cycles.  
- Multiple controllers operate in parallel — achieving **aggregate terabyte-level bandwidth.**

---

## ⑤ High-Bandwidth Memory (HBM) — The Physical Store

![HBM Internal Structure](https://github.com/hdimmfh/blog-img-repo/blob/main/img/gpu/architecture/HBM-architecture.png?raw=true)
*Figure 3. HBM Stacked DRAM Architecture.*

```
HBM Stack
 ├─ Channel 0–7
 │    ├─ BankGroup 0–7
 │    │    └─ Banks (DRAM cells)
 └─ TSV links (vertical connections)
```

- Each **HBM stack** has **8 independent channels**, each 128-bit wide.  
- 8 × 128 bits = **1024-bit aggregate bus** per stack.  
- Each channel has its own command/address bus and I/O interface.  
- Inside each channel are **bank groups**, each containing multiple **banks** (storage arrays).  
- Data moves vertically between DRAM layers through **TSVs (Through-Silicon Vias)**.

---

## ⑥ Data Path — From Compute to Silicon

```
SMs (Compute)
   ↓
GMMU (Address Translation)
   ↓
L2 Cache (Shared Buffer)
   ↓
Memory Controller (Command Control)
   ↓
HBM Channel → BankGroup → Bank (Physical Storage)
```

- **SMs** generate load/store requests.  
- **GMMU** maps virtual to physical memory.  
- **L2** caches and merges requests from all SMs.  
- **Memory Controllers** issue low-level DRAM commands.  
- **HBM** stores the actual operands that kernels operate on.

Each layer contributes to latency hiding and bandwidth scaling —  
a vertical network that bridges software addresses and physical electrons.

---

## ⑦ Layer Summary

| Layer | Role | Parallelism Unit |
|--------|------|------------------|
| SM | Compute instructions | Warp |
| GMMU | Address translation | Page |
| L2 Cache | Shared buffer | Slice |
| Memory Controller | Command scheduling | Channel |
| HBM | Physical storage | BankGroup / Bank |

---

## ⑧ Why This Matters

In the previous article, commands told the GPU **what to do**.  
In this one, we explored **where the data lives** and **how it moves** when those commands execute.

> Every `ld.global` or `st.global` instruction inside a kernel ultimately traverses  
> the GMMU → L2 → Controller → HBM hierarchy you just saw.

Understanding this path lays the groundwork for the **next post**,  
where we’ll connect both sides — command flow and data flow —  
to see how **computation and memory** converge on the GPU. ⚡️
