---
title: "MPS vs MIG vs MxGPU — Modern GPU Virtualization Explained"
by: hdimmfh
date: 2025-11-14 00:20:00 +0900
categories: [GPU, GPU Virtualization]
tags: [MPS, MIG, MxGPU, CUDA, ROCm]
---

🔍**Understanding Modern GPU Virtualization**

> “One GPU, many jobs — but not all sharing is the same.” \
> In this post, we’ll explore how `cVIDIA MPS`, `NVIDIA MIG`, and `AMD MxGPU` differ in design, purpose, and execution model.

---

## ① Overview

Modern GPUs are no longer single-user accelerators.  
They can now be **shared across multiple processes or even multiple virtual machines**, thanks to technologies like:

- MPS (Multi-Process Service) — software-level GPU sharing for CUDA processes  
- MIG (Multi-Instance GPU) — hardware-level partitioning on Ampere+ GPUs  
- MxGPU (Multiuser GPU) — AMD’s SR-IOV–based GPU virtualization for ROCm environments

---

## ② MPS — Multi-Process Service

`MPS` allows multiple CUDA processes (typically MPI ranks) to share a single GPU concurrently. It replaces CUDA’s per-process context scheduling with a single shared context, reducing overhead.

![NVIDIA MPS Architecture](https://github.com/hdimmfh/blog-img-repo/blob/main/img/gpu/architecture/nvidia-mps-architecture.png?raw=true)
*Figure 1. NVIDIA MPS Architecture.*

✔️ Key traits
- Type: Software-level, runtime multiplexing  
- Scope: Multiple processes of the *same job* (cooperative)  
- Architecture: Client–server (via `nvidia-cuda-mps-server`)  
- Benefit: Better utilization, lower context-switch overhead  
- Limitation: Single user per GPU (unless Volta+ `multiuser-server` mode)  

✔️ Memory behavior
- Pre-Volta: Shared GPU virtual address space → out-of-range writes could overwrite other clients  
- Volta+: Fully isolated address space per client  

✔️ Best for
- Multi-process CUDA or MPI workloads where each process underutilizes the GPU.

---

## ③ MIG — Multi-Instance GPU

`MIG` (introduced with NVIDIA Ampere) enables true hardware-level GPU partitioning. Each partition, or GPU instance, has dedicated compute cores, memory, and cache slices.

![NVIDIA MIG Architecture](https://github.com/hdimmfh/blog-img-repo/blob/main/img/gpu/architecture/nvidia-mig-architecture.jpg?raw=true)
*Figure 2. NVIDIA MIG Architecture.*

✔️ Key traits
- Type: Hardware-level virtualization  
- Scope: Independent GPU partitions per job or container  
- Isolation:** Strong — each MIG instance behaves like a separate GPU  
- Management: Controlled via `nvidia-smi mig` or Kubernetes device plugin  
- Compatibility: Requires A100 or newer GPUs  

✔️ Memory & performance
- No shared VRAM region; each instance has fixed VRAM capacity.  
- Fault isolation between MIG instances.  
- No context sharing or inter-instance communication.

✔️ Best for
- Multi-tenant environments (Kubernetes, Slurm) where **strict isolation** and **predictable QoS** are required.

---

## ④ MxGPU — AMD’s Multiuser GPU

`MxGPU (Multiuser GPU)` is AMD’s SR-IOV–based GPU virtualization solution.Unlike NVIDIA’s MIG or MPS, it’s designed from the start for virtual machines (VMs).

![AMD MxGPU Architecture](https://github.com/hdimmfh/blog-img-repo/blob/main/img/gpu/architecture/amd-sr-iov-mxgpu-architecture.jpg?raw=true)
*Figure 3. AMD SR-IOV MxGPU Architecture.*

✔️ Key traits
- Type: Hardware-assisted virtualization (SR-IOV)  
- Scope: Multiple VMs share one physical GPU via virtual functions (VFs)  
- Platform: ROCm + AMD Instinct MI-series or Radeon Pro GPUs  
- Isolation: Each VF has independent memory and scheduler  
- Driver: `amdgpu-pro` with SR-IOV support  

✔️ Integration
- Works seamlessly with **KVM**, **VMware**, and **Proxmox**.  
- Each VF appears as a discrete GPU device to the guest OS.  
- Supports ROCm/HIP workloads on supported hardware.

✔️ Best for
- Cloud and VDI environments where **GPU passthrough per VM** is needed.

💡 What is SR-IOV?
> SR-IOV (Single Root I/O Virtualization) is a PCIe standard that allows a single physical device (like a GPU or NIC) to expose multiple Virtual Functions (VFs) to the system.  
>
>Each VF acts as an independent device that can be directly assigned to a virtual machine or container — enabling hardware-level GPU sharing with minimal overhead.

---

## ⑤ Comparison Summary

| Feature | **MPS** | **MIG** | **MxGPU** |
|----------|----------|----------|------------|
| Level | Software | Hardware | Hardware (SR-IOV) |
| Vendor | NVIDIA | NVIDIA | AMD |
| Architecture | Client–Server | Partitioned SM & VRAM | Virtual Functions (VFs) |
| Isolation | Weak (Pre-Volta)</br> → Strong (Volta+) | Strong | Strong |
| Use Case | Multi-process job sharing | Multi-tenant GPU isolation | GPU sharing across VMs |
| CUDA/ROCm | CUDA only | CUDA only | ROCm only |
| Scheduling | Cooperative | Dedicated | Hardware-assisted |
| Typical Env | HPC clusters, MPI | Data centers, Kubernetes | Cloud, Virtualization hosts |

---

## ⑥ Practical Notes

- **MPS** ≈ “many MPI ranks, one GPU”  
- **MIG** ≈ “many GPUs inside one GPU”  
- **MxGPU** ≈ “one GPU shared across many VMs”

> ⚠️ These technologies are **mutually exclusive** on a single device.  
> You cannot enable MIG and MPS on the same GPU simultaneously.

---

## ⑦ TL;DR

| Goal | Recommended Tech |
|------|------------------|
| Maximize GPU utilization for MPI jobs | **MPS** |
| Strong isolation between tenants | **MIG** |
| Virtualization and VM-level GPU sharing | **MxGPU** |

---

## 📚 References

- [NVIDIA MPS Documentation](https://docs.nvidia.com/deploy/mps/index.html)  
- [NVIDIA MIG User Guide](https://docs.nvidia.com/datacenter/tesla/mig-user-guide/index.html)  
- [AMD MxGPU (SR-IOV) Overview](https://www.amd.com/en/technologies/sr-iov.html)  
- [ROCm Documentation](https://rocmdocs.amd.com/en/latest/)  
