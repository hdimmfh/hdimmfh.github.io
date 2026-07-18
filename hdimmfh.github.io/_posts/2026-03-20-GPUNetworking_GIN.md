---
title: GPU Networking — GIN
by: hdimmfh
date: 2026-03-21 16:00:00 +0900
categories: [GPU, GPU Network]
tags: [GPUDirect, RDMA, GIN, NCCL]
---

🔍 **Understanding GPU-Initiated Networking (GIN)**

> While GPUDirect RDMA optimized the `Data-Plane`, the `Control-Plane` remained tethered to the CPU. GIN overcomes this by shifting the initiation of network operations directly to the GPU.

---

## ① GPU Data Movement Architecture

Early GPU workloads relied on `CPU-centric` orchestration, where the CPU acted as the `Air Traffic Controller` for every data packet. Even with direct data paths, the control flow was heavily bottlenecked by CPU-GPU synchronization:

```
[GPU Kernel Execution & Remote Transfer Flow]

① Sender Node (Host CPU & GPU)
1-1. Wait for GPU: Poll CUDA Event (Check if Kernel is complete)
1-2. Work Submission: Create WQE (Work Queue Element) with Remote Address
1-3. Doorbell Ring: Notify NIC (Trigger RDMA Send/Write)

② Network Layer (NIC / RDMA)
2-1. Direct Access: NIC reads data directly from GPU Memory (GPUDirect RDMA)
2-2. Data Transport: Execute RDMA Operation (Zero-copy transfer)

③ Receiver Node (Host CPU & GPU)
3-1. Check Completion: Poll CQ (Completion Queue) for incoming data
3-2. Kernel Launch: Signal/Trigger GPU Kernel (Start processing received data)
```

While data paths have been optimized, the CPU-centric control plane remains a critical bottleneck. The constant need for host intervention to trigger network operations introduces significant scheduling bubbles and prevents the GPU from achieving its full communication potential.

<div style="background-color: white; padding: 2px; margin-top: 10px; border-radius: 8px; text-align: center; border: 1px solid #eee;">
  <img src="https://github.com/hdimmfh/blog-img-repo/blob/main/img/gpu/network/cpu-centeric-gpu-networking.png?raw=true" 
    alt="CPU-Centric GPU Networking">
  <p style="margin-top: 15px; margin-bottom: 0px; padding: 0px; font-style: italic; color: #666;">Figure 1. CPU-centric Networking.</p>
</div>
 
- Unblock GPU Processing(3): Once the data transfer is complete, the CPU verifies the arrival and signals the waiting GPU to begin processing. This step, highlighted in red, represents the CPU-centric Control Plane where host intervention is required to trigger the next task.

> 🔗 For a deeper dive into the history and types of GPU data transfers, refer to:
> - [NVIDIA Documentation: GPUDirect RDMA ](https://docs.nvidia.com/cuda/gpudirect-rdma/) 
> - [HDIMMFH Blog: GPU Data Movement History](https://hdimmfh.github.io/posts/gpu-data-movement-history/)

---

## ② GPU-Initiated Networking (GIN)

If GPUDirect RDMA built the "Highway" for data, `GIN` finally hands the "Steering Wheel" to the GPU.

<div style="background-color: white; padding: 2px; margin-top: 10px; border-radius: 8px; text-align: center; border: 1px solid #eee;">
  <img src="https://github.com/hdimmfh/blog-img-repo/blob/main/img/gpu/network/gpu-initiated-networking.png?raw=true" 
    alt="GPU-Initiated Networking">
  <p style="margin-top: 15px; margin-bottom: 0px; padding: 0px; font-style: italic; color: #666;">Figure 2. GPU-centric with the GPU controlling NIC.</p>
</div>

- Direct NIC Control(1), (4): Notice that the red arrows for "Receive Packets" and "Send Packets" now originate directly from the CUDA Processing block. The GPU issues the "Doorbell Ring" and manages Work Queues (WQE) without waiting for CPU instructions.

> 🔗 For a deeper dive into the history and types of GPU data transfers, refer to:
> - [Related Paper(arxiv): GPU-Initiated Networking for NCCL](https://arxiv.org/abs/2511.15076/)
> - [NVIDIA Official Blog: DOCA GPUNetIO Blog](https://developer.nvidia.com/ko-kr/blog/realizing-the-power-of-real-time-network-processing-with-nvidia-doca-gpunetio/) 
> - [NVIDIA  Documentation: DOCA-GPUNetIO for Optimized Inferencing](https://docs.nvidia.com/doca/sdk/doca-gpunetio/index.html)


### 2-1. Seizing the Control-Plane
In traditional GPUDirect RDMA, even though the data flows directly, `the trigger` (ringing the NIC's Doorbell register) is pulled by the CPU. The GPU must signal the CPU that a kernel is finished, and the CPU then issues the command to the NIC. This synchronization adds several microseconds of latency.

> 💡 GIN eliminates this bottleneck:
> * Direct Access: GPU threads directly write to the NIC’s `Doorbell register` over the PCIe bus.
> * Kernel Initiation: Communication is triggered from within the CUDA kernel, allowing the GPU to send data as soon as it is generated without waiting for CPU coordination.

### 2-2. Implementation via DOCA & NCCL
This is made possible through NVIDIA `DOCA GPUNetIO`, which provides a lightweight network stack callable from GPU threads. 
* The GDAKI Backend: Leverages direct GPU-to-NIC communication for ultra-low latency.
* Semaphore-based Sync: Instead of the CPU polling a Completion Queue (CQ), the receiving GPU kernel polls a local `Semaphore` updated directly by the NIC.

---

## ③ Hardware Context: PCIe Switch P2P

On platforms like the **HPE ProLiant DL380a Gen11**, the **PCIe Switch Board** is the physical enabler. It allows Peer-to-Peer (P2P) traffic to "fold" at the switch level, preventing packets from ever reaching the CPU's Root Complex. This hardware topology is critical for GIN to achieve its full potential in **Mixture-of-Experts (MoE)** workloads.

---

## ⑤ TL;DR

- GPUDirect RDMA: Data path is direct, but the Control-Plane is still CPU-dependent.
- GIN: Both Data and Control are managed by the GPU.
- Benefits: Zero CPU overhead, microsecond-level latency reduction.
- Requirements: NCCL 2.28+, DOCA GPUNetIO, and a P2P-capable PCIe topology.