---
title: Understanding the Stack: PyTorch → (CUDA / GDS) → GPU
author: hdimmfh
date: 2025-11-04 20:47:00 +0900
categories: [Study, GPU]
tags: [PyTorch, CUDA, GDS, GPU, DeepLearning]
---

🔍 **Understanding the Stack: PyTorch → (CUDA / GDS) → GPU**

When people say “PyTorch runs on GPU,” it’s only partly true.  
In reality, several layers cooperate to move data and execute computation efficiently.

---

## 🧩 1️⃣ PyTorch — The Orchestrator

PyTorch never touches GPU hardware directly.  
It defines what operations to run and where to run them, then calls CUDA APIs internally.

```python
x = x.to("cuda")  # calls cudaMemcpy()
y = torch.mm(x, x.T)  # launches cuBLAS kernel
```

PyTorch decides the logic; CUDA executes it.

---

## ⚙️ 2️⃣ CUDA — The Executor

CUDA (Compute Unified Device Architecture) is NVIDIA’s runtime that manages GPU memory and launches kernels.  
It handles:

- Memory allocation (`cudaMalloc`)
- Data transfer (`cudaMemcpy`)
- Kernel launch (`cudaLaunchKernel`)
- Stream synchronization

CUDA translates PyTorch’s high-level operations into GPU instructions.

---

## 💪 3️⃣ GPU — The Worker

The GPU executes kernels scheduled by CUDA.  
It reads data from its VRAM, performs matrix ops, and writes back results.  
It never acts on its own — every action begins with a CUDA command.

---

## 💾 4️⃣ GDS (GPUDirect Storage) — Direct Data Path

Traditionally, data flows like this:

```
NVMe SSD → CPU RAM → GPU VRAM
```

With **GPUDirect Storage**, it becomes:

```
NVMe SSD → GPU VRAM
```

No CPU memory copy — only a control signal from the CPU (`cuFileRead()`),  
then DMA engines move data directly between storage and GPU memory.  
The CPU initiates the command, but the transfer bypasses system memory entirely.

---

## 🧠 Summary Flow

**PyTorch → CUDA → GPU → GDS**

| Layer | Role |
| ------ | ------ |
| PyTorch | Defines operations, calls CUDA |
| CUDA | Manages memory, launches GPU kernels |
| GPU | Executes computations in parallel |
| GDS | Streams data from NVMe directly to GPU VRAM |

---

## 🧩 In One Line

① PyTorch tells CUDA what to do,  
② CUDA tells the GPU how to do it,  
③ the GPU performs the work,  
④ and GDS feeds data straight from storage to GPU memory.

---

*Originally posted on [Tistory](https://hdimmfh.tistory.com/62)* ✨
