---
layout: post
title: "PCIe BAR and Switch Fabric"
date: 2025-11-07 20:47:00 +0900
categories: [GPU, Network]
tags: [GPU, PCIe, BAR, Switch, RDMA, GDS]
---

🔍**PCIe BAR and Switch Fabric Explained (Component View)**

Modern GPUs and NICs communicate through **PCI Express (PCIe)**.  
Every device on PCIe exposes part of its memory or control registers using **Base Address Registers (BARs)**.  
This post explains how BARs are accessed through a **PCIe switch fabric**, both for **local (same node)** and **remote (RDMA)** communication.

---

## ① PCIe Hierarchy Overview

```
Host CPU
 └─ Root Complex (RC)
      └─ PCIe Switch
            ├─ GPU (Endpoint)
            └─ NIC (Endpoint)
```

- **Root Complex (RC)** is part of the CPU/chipset.  
  It connects system memory and CPUs to the PCIe fabric.  
- **PCIe Switch** connects multiple endpoints such as GPUs and NICs.  
  It acts like a router for PCIe transactions.  
- Each endpoint (GPU, NIC, NVMe) is a **PCIe device** with its own BAR mappings.

---

## ② Base Address Register (BAR)

**BAR (Base Address Register)** defines a *window* of device memory visible to the PCIe bus.

![PCI BAR memory addresses](https://www.researchgate.net/profile/Jonas-Markussen-2/publication/353106078/figure/fig1/AS:1043973338578944@1625914048636/Device-memory-regions-BARs-are-mapped-into-the-same-address-space-as-system-memory.png)
*Figure 1. Device memory regions (BARs) are mapped into the same address space as system memory.*

- BAR0 → control registers (MMIO)
- BAR1 → device memory window (VRAM or GPU address space)
- BAR2+ → additional memory or configuration spaces

Example for a GPU:
```
BAR0: MMIO registers (control)
BAR1: Device memory window (VRAM)
BAR2/3: Extended 64-bit window
```

When the CPU or another PCIe device wants to access GPU memory,  
it writes to an **address within the GPU’s BAR window** — not the GPU’s internal physical address.

> BAR = translation window between PCIe bus addresses and GPU physical memory.

---

## ③ PCIe Switch as a Fabric

![PCIe Switch Architecture](https://fuse.wikichip.org/wp-content/uploads/2018/05/nvidia-dgx-1-v100-nvlink-gpu-xeon-config.png)
*Figure 2. Nvidia’s NVLink interconnection and the NVSwitch.*

The PCIe switch connects multiple devices using **internal routing tables**.  
It decides which downstream port (GPU, NIC, NVMe) should receive a given PCIe transaction.

```
PCIe Switch
 ├─ Upstream Port (to Root Complex)
 ├─ Downstream Port 1 → GPU
 ├─ Downstream Port 2 → NIC
 └─ Downstream Port 3 → NVMe
```

- Each port has a **Bus Address Range** for its BARs.  
- When the CPU accesses a GPU BAR address,  
  the PCIe switch routes that request to the GPU’s downstream port.

> The switch does not understand "memory" — it only routes packets by address range.

---

## ④ Local Node Access (CPU ↔ GPU BAR)

**Local (intra-node) access** uses the same PCIe fabric and BARs.

```
CPU → PCIe Switch → GPU (BAR1)
```

- The CPU issues a DMA or MMIO write to the GPU’s BAR address.  
- The PCIe switch routes the TLP (Transaction Layer Packet) to the GPU port.  
- The GPU receives it and maps it into its device memory.  
- CUDA runtime or GDS (GPUDirect Storage) uses this path for zero-copy I/O.

> This is how GPUDirect Storage bypasses CPU DRAM and transfers data directly into GPU VRAM.

---

## ⑤ Remote Node Access (GPU ↔ GPU via NIC)

**Remote (inter-node)** access happens through **RDMA** — using the NIC’s DMA engine.

```
Host CPU
 └─ NIC Driver (mlx5_core)
       │
       ▼
PCIe Bus
       │
       ▼
NIC Hardware (HCA)
 ├─ DMA Engine(s)
 ├─ Packet Processor
 ├─ Completion Queue (CQ)
 └─ Doorbell / Queue Pair (QP) Controller
```

Steps:
1. CPU driver (mlx5_core) registers GPU memory and exposes its **Remote BAR (RKey)**.  
2. NIC’s **DMA Engine** reads/writes directly to the GPU BAR address over PCIe.  
3. Remote NIC on another node performs the same operation.  
4. Data moves directly GPU↔GPU without CPU copy.

> Remote BAR = exported GPU BAR region used for RDMA.

---

## ⑥ Comparison Summary

| Type | Path | Initiator | Address Type |
|------|------|------------|---------------|
| **CPU–GPU (Local)** | PCIe Switch | CPU | GPU BAR (Local) |
| **GPU–GPU (Local)** | PCIe Switch | GPU DMA | GPU BAR (Peer) |
| **GPU–GPU (Remote)** | PCIe + RDMA | NIC DMA | GPU Remote BAR (RKey) |

---

## ⑦ BAR + PCIe Fabric Summary

```
[CPU / NIC Driver]
       ↓
PCIe Root Complex
       ↓
PCIe Switch
 ├─ GPU (BAR)
 ├─ NIC (DMA Engine)
 └─ NVMe (Controller)
```

- **BAR:** Window that maps GPU or device memory to the PCIe address space.  
- **PCIe Switch:** Routes packets based on BAR address range.  
- **NIC DMA Engine:** Accesses local or remote BARs directly over PCIe or RDMA.  

> The BAR is not memory itself — it’s a *bridge address space*  
> between PCIe transactions and real device memory.

---

### Key Takeaways

```
- BAR = PCIe-visible window to device memory
- PCIe Switch = address-based router for TLPs
- Local BAR = used by CPU / peer GPUs
- Remote BAR = used by NIC DMA via RDMA
```

---

*Written by [hdimmfh](https://github.com/hdimmfh)*  
*Nov 2025 — Understanding the GPU Stack Series, Part IV*
