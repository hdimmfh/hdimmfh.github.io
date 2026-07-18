---
title: CUDA — Command Pipeline
by: hdimmfh
date: 2025-11-04 10:40:00 +0900
categories: [GPU, GPU Network]
tags: [CUDA, PFIFO, PBDMA, GPU, Kernel, CommandBuffer]
---

🔍**Understanding How Commands Travel to the GPU**

When we say *“PyTorch sends a kernel to the GPU”*, it’s easy to imagine that the CPU somehow pushes arithmetic instructions like `add` or `mul` directly into the GPU.  
But that’s not what really happens.  

![MMIO Between CPU(Cuda driver) and GPU BAR](https://github.com/hdimmfh/blog-img-repo/blob/main/img/gpu/network/gpu_command_flow.png?raw=true)
*Figure 1. MMIO Between CPU(Cuda driver) and GPU.*

In reality, what travels through PCIe is not a stream of math operations —  
it’s a **set of control packets** describing *where in GPU memory the kernel code resides*.

---

## ① CUDA Driver — The Origin

The **CUDA Driver** acts as a memory manager for the GPU.  
When a kernel is loaded, the driver allocates GPU memory (VRAM), uploads the kernel binary (`.cubin`),  
and already knows its **GPU Virtual Address (GPU VA)** on the device.

So when it builds a **Command Buffer**, the driver doesn’t include the kernel’s instructions —  
it writes the **GPU VA where those instructions live**.  
For example:

```
[SET_SHADER_ADDRESS 0x8000_2000]
[SET_GRID_DIM (128,1,1)]
[LAUNCH]
```

The Command Buffer itself lives in VRAM,  
but the addresses inside point to *other* VRAM regions — where the real kernel binary sits.  

That’s the elegant part:  
> the CPU never sees GPU physical memory directly,  
> but the CUDA Driver already knows all the GPU VAs, because it manages the device memory map.

---

## ② MMIO and BAR — How the CPU Talks to the GPU

To notify the GPU that new commands are ready,  
the driver performs an **MMIO write** to the GPU’s **BAR0 (Base Address Register)** over PCIe.  
This triggers the **doorbell register** in the GPU’s **PFIFO** block — the command dispatcher.

In other words, MMIO is simply the CPU sending a "new commands available" signal  
to a memory address that actually corresponds to GPU control logic.

---

## ③ PFIFO — The Command Dispatcher

The **PFIFO** hardware inside the GPU manages multiple **Channels** (command queues).  
Each Channel owns its own Command Buffer (base address, GET/PUT pointers).  
Channels correspond to CUDA Streams, processes, or contexts.

PFIFO detects that a Channel’s PUT pointer has advanced,  
and schedules that Channel to a hardware DMA engine called **PBDMA**.  

> Commands inside one Channel are executed *sequentially*,  
> but multiple Channels run *in parallel* — this is GPU’s Command-Level Parallelism.

---

## ④ PBDMA — Fetching Commands from VRAM

The **PBDMA (Push Buffer DMA)** engine performs the real work:  
it reads Command Buffers from VRAM via DMA and decodes their packets.  
Each packet tells it what to do next — usually, send the command to a specific **GPU Engine**.

- `LAUNCH` packets → Compute Engine  
- `MEMCOPY` packets → Copy Engine  
- `GRAPHICS` packets → GFX Engine  

Here’s the surprising part:  
> PBDMA doesn’t hold the kernel instructions themselves.  
> It passes the **GPU Virtual Address (VA)** — the pointer to the kernel binary in VRAM —  
> to the Compute Engine.

---

## ⑤ Compute Engine — Resolving the Real Kernel

The **Compute Engine** receives this VA,  
consults the **GMMU (GPU Memory Management Unit)** to translate it into a GPU physical address,  
and fetches the kernel binary (SASS code) from VRAM.

![GPU L1 L2 DRAM Architecture](https://github.com/hdimmfh/blog-img-repo/blob/main/img/gpu/architecture/cache_memory_architecture.png?raw=true)
*Figure 2. GPU L1, L2, DRAM Architecture.*

That code then flows down the cache hierarchy:

```
VRAM → L2 Cache → L1 Instruction Cache (per SM)
```

At this point, the kernel’s *actual instructions* —  
like `add.f32`, `mul.rn`, or `fma` —  
are inside each **Streaming Multiprocessor (SM)** and begin executing in parallel warps.

So the Command Buffer never contained the arithmetic itself —  
only the *address* where the arithmetic lives.

---

## ⑥ From Control to Execution

Let’s recap the full journey:

```
CPU (CUDA Driver)
 ├─ Builds Command Buffer in VRAM
 │   └─ Contains GPU VAs, not actual code
 ├─ MMIO Write → GPU BAR0 (doorbell)
 ▼
PFIFO (Command Dispatcher)
 ├─ Detects active Channels
 └─ Assigns them to PBDMA
 ▼
PBDMA (DMA Engine)
 ├─ Reads Command Buffer from VRAM
 └─ Sends VA to Compute/Copy Engine
 ▼
Compute Engine
 ├─ Resolves GPU VA via GMMU
 ├─ Fetches kernel binary (VRAM → L2 → L1)
 └─ Launches grid
 ▼
SM
 └─ Executes kernel instructions (add/mul/fma…)
```

---

## ⑦ Serial Inside, Parallel Across

- **Inside one Channel:** Commands are sequential (`GET→PUT` order).  
- **Across Channels:** Commands execute concurrently across multiple PBDMAs and Engines.  

This design allows the GPU to maintain strict ordering within one stream  
while achieving massive concurrency across processes and contexts.

---

## ⑧ What About the Data?

So far, we’ve seen how *commands* travel —  
control signals, kernel addresses, and synchronization packets.  

But the story isn’t over.  
> The next big question is:  
> **if commands flow this way, how does the *data* itself travel?**

That’s where memory transfers, DMA engines, and GPUDirect paths come into play —  
a topic for the next post. 🚀

