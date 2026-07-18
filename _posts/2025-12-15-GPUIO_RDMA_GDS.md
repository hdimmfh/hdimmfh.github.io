---
title: GPU I/O(1) — RDMA/GDS
by: hdimmfh
date: 2025-12-15 02:10:00 +0900
categories: [GPU, GPU Network]
tags: [GPUDirect, RDMA, GDS, NVMe]
---

🔍 **Understanding GPU-Centric Data Movement**

> “The fastest GPU is useless if data cannot reach it efficiently.”  
> This post traces the evolution of GPU data movement layers and explains how modern architectures minimize CPU involvement using RDMA, GPUDirect, and GDS.

---

## ① GPU Data Movement Layer History

Early GPU workloads were dominated by **CPU-centric data paths**:

```
Storage → CPU Memory → GPU Memory
```

This model introduced:
- Redundant memory copies
- CPU interrupts
- Cache pollution
- Latency amplification

To remove these bottlenecks, data movement evolved across three major axes:

![GPU Data Movement History](https://github.com/hdimmfh/hdimmfh.github.io/tree/main/_data/blog-img-repo/img/gpu/architecture/history_gpu_data_movement_layer.png?raw=true)
*Figure 1. GPU Data Movement History.*

1. **RDMA (1999+)**
    - NIC ↔ Host Memory
    - Zero-copy across hosts
2. **GPUDirect RDMA (2012+)**
    - NIC ↔ GPU Memory
    - GPU-visible memory mapping
3. **GDS (2020+)**
    - NVMe ↔ GPU Memory
    - Storage bypasses CPU entirely

Each layer progressively removes the CPU from the data path.

---

## ② GPU Data Movement Architecture

Modern GPU data movement is built on **DMA-capable endpoints**:

- GPU
- NIC
- NVMe SSD

All of them can act as **bus masters** on PCIe.

### Core Principle

> If two devices can perform DMA and share addressability, the CPU does not need to touch the data.

### Unified View (Local + Remote)

```
GPU Memory ↔ NIC ↔ Network ↔ NIC ↔ GPU Memory
GPU Memory ↔ NVMe (PCIe DMA)
```

The CPU’s role is reduced to:
- Queue setup
- Descriptor submission
- Control-plane orchestration

No payload data passes through CPU caches.

---

## ③ GDS Data Movement (Local)

**GPUDirect Storage (GDS)** enables direct data transfer:

![GPU Local Data Movement](https://github.com/hdimmfh/hdimmfh.github.io/tree/main/_data/blog-img-repo/img/gpu/architecture/history_gpu_data_movement_layer_gds.png?raw=true)
*Figure 2. GPU Local Data Movement.*

```
Local NVMe SSD → GPU Memory
```

### Without GDS

```
NVMe → System Memory → GPU Memory
```

- 2 DMA hops
- CPU page cache involvement
- Higher latency

### With GDS

```
NVMe ──DMA──▶ GPU Memory
```

Characteristics:
- Single DMA operation
- No CPU copy
- No cache pollution
- Deterministic latency

> GDS treats GPU memory as a first-class I/O target.

---

## ④ Remote GDS Data Movement

Remote GDS extends the same principle **across hosts**.

![GPU Remote Data Movement](https://github.com/hdimmfh/hdimmfh.github.io/tree/main/_data/blog-img-repo/img/gpu/architecture/history_gpu_data_movement_layer_remote_gds.png?raw=true)
*Figure 3. GPU Remote Data Movement.*

### Data Path

```
Remote NVMe
 → Remote NIC
 → RDMA Fabric
 → Local NIC
 → GPU Memory
```

Key technologies involved:
- NVMe-oF (RDMA)
- GPUDirect RDMA
- GDS

---

## ⑤ TL;DR

- RDMA removed CPU from **network data paths**
- GPUDirect RDMA extended RDMA to **GPU memory**
- GDS removed CPU from **storage I/O**
- Remote GDS combines both:
    - NVMe-oF + RDMA + GDS
    - GPU ↔ Storage, end-to-end, zero-copy
