---
title: GPU Memory Architecture
by: hdimmfh
date: 2025-11-07 20:04:00 +0900
categories: [GPU, Network]
tags: [GPU, HBM, CUDA, Architecture, Memory]
---

🔍**GPU Memory Architecture (Top-Down)**

This post explains the GPU memory system **top-down**, from compute units (SMs) to physical HBM stacks.  
Each part is described directly: what it is and what it does.

---

## ① GPU Overview

![NVIDIA A100 GPU Architecture Diagram](https://developer-blogs.nvidia.com/wp-content/uploads/2020/05/CSP-multi-user-with-MIG-1.png)
*Figure 1. NVIDIA A100 GPU Architecture (MIG example).*
![NVIDIA Memory hierarchy](https://developer-blogs.nvidia.com/wp-content/uploads/2020/06/memory-hierarchy-in-gpus-1.png)
*Figure 2. Memory hierarchy in GPUs.*

```
SMs → L2 Cache → Memory Controllers → HBM Stacks
```

- A GPU has many **Streaming Multiprocessors (SMs)**.  
- SMs run thousands of threads in parallel.  
- All SMs share a global **L2 cache**.  
- The L2 cache connects to multiple **memory controllers**.  
- Each memory controller connects to one **HBM stack**.

---

## ② Streaming Multiprocessor (SM)

```
SM = { CUDA cores + Warp Scheduler + L1 Cache + Registers }
```

- The SM executes instructions.  
- Each SM has CUDA cores, Tensor cores, and registers.  
- Each warp (32 threads) runs the same instruction at once.  
- When one warp waits for memory, another runs.  
- This keeps all compute units busy.

---

## ③ L2 Cache and GMMU

```
SM → GMMU → L2 Cache → Memory Controller
```

- **L2 Cache** stores data shared between SMs.  
- **GMMU (GPU Memory Management Unit)** converts virtual addresses to physical addresses.  
- The GMMU and L2 connect to each memory controller.  
- They route and schedule memory requests from all SMs.

---

## ④ Memory Controller

```
Controller count = Channel count
```

- Each **Memory Controller** manages one **HBM channel**.  
- It sends read/write commands to DRAM.  
- It controls timing for bank groups and banks.  
- All controllers work in parallel for maximum bandwidth.

---

## ⑤ High-Bandwidth Memory(HBM) Stack

![HBM Internal Structure](https://figures.semanticscholar.org/8a6b0e52c39cdd3394dde7a1a4587d5a3b311171/2-Figure1-1.png)
*Figure 3. HBM Stacked DRAM Architecture.*

```
HBM Stack
 ├─ Channel 0–7
 │    ├─ BankGroup 0–7
 │    │    └─ Banks (data arrays)
 └─ TSV links (vertical connections)
```

- Each HBM stack has **8 channels**.  
- Each channel has **8 bank groups**, each with multiple banks.  
- Each channel is 128-bit wide.  
- 8 channels × 128 bits = 1024-bit total per stack.  
- Each channel has its own memory controller.

---

## ⑥ Data Path Summary

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

- The data path is hierarchical.  
- SM runs compute.  
- GMMU translates address.  
- L2 buffers data.  
- Controller sends commands.  
- HBM stores data.

---

## 📘 Summary

```
SM → GMMU → L2 Cache → Controller → HBM
```

| Layer | Role | Parallel Unit |
|--------|------|---------------|
| SM | Compute | Warp |
| GMMU | Address mapping | Page |
| L2 Cache | Data buffer | Slice |
| Memory Controller | Command scheduling | Channel |
| HBM | Physical memory | BankGroup / Bank |

---

*Written by [hdimmfh](https://github.com/hdimmfh)*  
*Nov 2025 — Understanding the GPU Stack Series, Part III*
