---
title: GPU Memory Data Paths — How Bytes Actually Move
by: hdimmfh
date: 2025-11-07 10:55:00 +0900
categories: [GPU, GPU Network]
tags: [GPU, PCIe, BAR, Switch, RDMA, GDS, Memory]
---

🔍**GPU Memory Data Paths — How Bytes Actually Move**

In the [previous post](https://hdimmfh.github.io/posts/2025-11-06-gpu-memory-architecture),  
we explored **the structure of the GPU’s memory system** — how SMs, caches, controllers, and HBM stacks connect.  

![PCIe Switchs Fabric](https://github.com/hdimmfh/blog-img-repo/blob/main/img/gpu/network/pcie_switch_fabric.webp?raw=true)
*Figure 1. PCIe Switchs Fabric.*


Now it’s time to follow the **actual movement of data** through that structure.  
This post focuses on **how bytes travel** across the PCIe fabric, BAR mappings, and DMA engines  
— from CPU or NIC all the way into GPU VRAM and HBM.

Understanding these paths completes the picture:  
if the last article explained *where data can live*,  
this one shows *how it actually gets there.*

---

## ① PCIe Hierarchy Overview

```
Host CPU
 └─ Root Complex (RC)
      └─ PCIe Switch
            ├─ GPU (Endpoint)
            └─ NIC (Endpoint)
```

- **Root Complex (RC)** connects CPU/system memory to the PCIe fabric.  
- **PCIe Switch** routes transactions between devices (GPUs, NICs, NVMe).  
- Each endpoint has its own **BARs (Base Address Registers)**, exposing parts of its memory to the PCIe bus.

---

## ② Base Address Register (BAR) — The Window to Device Memory

**BAR (Base Address Register)** defines a memory window that maps a device’s internal memory into the PCIe address space.

![PCI BAR memory addresses](https://github.com/hdimmfh/blog-img-repo/blob/main/img/gpu/network/pcie_bar.png?raw=true)
*Figure 2. Device memory regions (BARs) are mapped into the same address space as system memory.*

Example for a GPU:
```
BAR0: MMIO control registers
BAR1: VRAM window (device memory space)
BAR2+: extended or configuration ranges
```

> BAR = a PCIe-visible window into GPU memory, not the memory itself.

When the CPU or another device writes to a BAR address,  
it’s actually sending packets over PCIe that get translated into GPU memory accesses.

---

## ③ PCIe Switch Fabric — The Router of Data

The **PCIe Switch** routes all transactions based on their target address range.  
Each downstream port corresponds to a device’s BAR region.

```
PCIe Switch
 ├─ Upstream Port → Root Complex
 ├─ Downstream Port 1 → GPU
 ├─ Downstream Port 2 → NIC
 └─ Downstream Port 3 → NVMe
```

When the CPU accesses a GPU BAR address,  
the switch simply forwards that PCIe **Transaction Layer Packet (TLP)** to the correct device.

> The switch doesn’t know what the data means — it only routes packets by address.

---

## ④ Local Node Transfers — CPU ↔ GPU

**Intra-node** data movement uses BARs within the same PCIe fabric.

```
CPU → PCIe Root Complex → PCIe Switch → GPU (BAR1 → VRAM)
```

1. The CPU issues a DMA or MMIO write to a GPU BAR address.  
2. The PCIe controller wraps it into **TLPs** (Transaction Layer Packets).  
3. The switch forwards them to the GPU port.  
4. The GPU decodes the BAR address, resolves it to a VRAM offset,  
   and its **DMA engine** performs the actual write into VRAM.

> The CPU never moves the bytes itself — it just tells the DMA engine *where* to move them.

This mechanism is what enables **GPUDirect Storage (GDS)** —  
NVMe devices can DMA directly into GPU VRAM via BAR1, bypassing host memory entirely.

---

## ⑤ Peer-to-Peer Transfers — GPU ↔ GPU (Same Node)

Modern GPUs can also communicate directly over PCIe without CPU involvement.

```
GPU A (BAR) ↔ PCIe Switch ↔ GPU B (BAR)
```

- Each GPU exposes its memory window through BAR1.  
- The initiating GPU uses its **DMA engine** to write directly into the peer’s BAR address.  
- The switch routes the transaction between GPUs.  
- CUDA’s **peer-to-peer (P2P)** and **NCCL** backends rely on this mechanism.

> BAR-to-BAR transfers are the foundation of **GPUDirect P2P** communication.

---

## ⑥ RDMA Transfers — GPU ↔ GPU (Across Nodes)

For **inter-node** transfers, the NIC becomes part of the data path.  
The NIC’s **DMA engine** accesses GPU memory via BAR, while another NIC mirrors the process remotely.

```
GPU A (VRAM)
   ↑
PCIe ←→ NIC A ──── RDMA Fabric ──── NIC B ←→ PCIe
                                               ↓
                                         GPU B (VRAM)
```

Steps:
1. The CPU driver (e.g., `mlx5_core`) registers GPU memory and exposes it as a **Remote BAR (RKey)**.  
2. The NIC’s DMA engine performs PCIe reads/writes directly into that BAR region.  
3. The remote NIC does the same, completing **GPU↔GPU transfers** with no CPU copy.  

> RDMA turns the BAR window into a globally addressable GPU memory region — enabling *true zero-copy* between nodes.

---

## ⑦ The Full Data Path

```
Host Memory / Storage
   ↓
PCIe Root Complex
   ↓
PCIe Switch Fabric
   ↓
GPU BAR (PCIe-visible address)
   ↓
GPU DMA Engine → Memory Controller → HBM (Physical Data Store)
```

- **CPU or NIC** initiates a PCIe transaction to the GPU BAR address.  
- **PCIe Switch** routes the TLPs to the GPU.  
- **GPU’s DMA Engine** writes data into VRAM.  
- The **Memory Controller** inside the GPU moves it into **HBM** through internal DRAM channels.

Thus, bytes flow from **system or remote memory → PCIe fabric → GPU BAR → HBM stack**.

---

## ⑧ Why It Matters

In the [previous post](https://hdimmfh.github.io/posts/2025-11-04-inside-gpu-command-pipeline),  
we saw how *commands* travel to tell the GPU *what* to do.  
In this post, we examined how *data* actually travels to where it’s needed.  

Together with [GPU Memory Architecture](https://hdimmfh.github.io/posts/2025-11-06-gpu-memory-architecture),  
these layers complete the story of **how commands and data meet** inside the GPU.

> Command Path = PFIFO → PBDMA → SMs  
> Data Path = PCIe BAR → DMA → Memory Controller → HBM

---

## Summary

| Path Type | Initiator | Mechanism | Example |
|------------|------------|------------|----------|
| CPU ↔ GPU | CPU DMA | PCIe BAR | GPUDirect Storage |
| GPU ↔ GPU (local) | GPU DMA | BAR-to-BAR | NCCL / CUDA P2P |
| GPU ↔ GPU (remote) | NIC DMA | RDMA (Remote BAR) | GPUDirect RDMA |

