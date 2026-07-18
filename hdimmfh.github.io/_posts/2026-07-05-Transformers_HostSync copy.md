---
title: Transformers — Detecting Host Sync
by: hdimmfh
date: 2026-07-04 16:00:00 +0900
categories: [GPU, CUDA]
tags: [CUDA, DeepSpeed, Transformers, Nsight, PyTorch]
---

🔍 **Finding a Hidden GPU Synchronization Bottleneck**

> While profiling DeepSpeed Sequence Parallel training with Nsight Systems, I found an unexpected performance gap inside the `compute_loss` region. The GPU appeared idle, yet no CUDA kernel explained the delay. The root cause eventually led to a Python conditional evaluating CUDA tensors.

---

## ① The Symptom

During distributed training on a **2-node NVIDIA B300 cluster (16 GPUs)**, I profiled the forward pass using **Nsight Systems**.

Surprisingly, the `compute_loss` NVTX region contained a long idle section where almost every GPU metric dropped.

Observed timeline:

```
NCCL Broadcast
    ↓
Tensor Transform
    ↓
SM80 GEMM
    ↓
Repeated 8-byte DtoH memcpy
Repeated pthread_cond_wait
    ↓
Gather Kernel
    ↓
Elementwise Kernels
```

During this period,

- SM utilization remained extremely low
- Memory bandwidth was almost idle
- NVLink traffic was minimal
- CPU repeatedly entered `pthread_cond_wait`

The most suspicious event was the repeated **8-byte Device-to-Host memcpy**.

---

## ② Chasing the Synchronization

The first assumption was that the synchronization originated from NCCL or DeepSpeed communication.

However,

- NCCL kernels had already completed.
- No large GPU memory transfers were occurring.
- No CUDA kernel occupied the GPU.

This suggested that the GPU was waiting on the host rather than executing device work.

Tracing the execution eventually narrowed the problem down to the loss computation path.

```
Forward
    ↓
compute_loss()
    ↓
deepspeed_sp_compute_loss()
```

---

## ③ The Hidden Synchronization

Inside `transformers/integrations/deepspeed.py`, the following code performs Sequence Parallel loss aggregation.

```python
good_tokens_per_rank = torch.distributed.nn.functional.all_gather(
    good_tokens,
    group=sp_group,
)

total_loss = sum(
    losses_per_rank[rank] * good_tokens_per_rank[rank]
    for rank in range(sp_world_size)
    if good_tokens_per_rank[rank] > 0
)
```

The critical line is

```python
if good_tokens_per_rank[rank] > 0
```

Since `good_tokens_per_rank` contains CUDA tensors returned by `all_gather`, the Python conditional may require evaluating a GPU value on the host.

That evaluation can introduce an implicit synchronization, which appears in Nsight Systems as repeated small Device-to-Host memcpy operations together with host-side waiting.

Although each synchronization transfers only **8 bytes**, repeating this operation many times creates noticeable bubbles in the execution timeline.

---

## ④ GPU-side Aggregation

Instead of branching in Python, the same computation can be expressed entirely using tensor operations.

```python
good_tokens = torch.stack(good_tokens_per_rank)
losses = torch.stack(losses_per_rank)

mask = good_tokens > 0

safe_losses = torch.where(
    mask,
    losses,
    torch.zeros_like(losses),
)

total_loss = (safe_losses * good_tokens).sum()
```

This keeps the filtering on the GPU and avoids evaluating CUDA tensors inside Python.

Whether this implementation is appropriate depends on the intended semantics (for example, handling zero-token ranks and NaN propagation), but it illustrates how the host synchronization can potentially be removed.

---

## ⑤ Upstream Discussion

After identifying the issue, I opened an upstream discussion in the **[Hugging Face Transformers](https://github.com/huggingface/transformers/issues/47068)** repository.

The discussion focused on whether the loss aggregation could remain entirely on the GPU while preserving the original behavior.

Following the discussion, a pull request implementing the GPU-side aggregation approach was opened. At the time of writing, the PR is still under review.

---

## ⑥ Lessons Learned

This investigation reinforced an important lesson about GPU performance analysis. Not every GPU stall originates from CUDA kernels. Sometimes the largest performance bubble is caused by a seemingly harmless Python statement.

When profiling distributed training,

- CUDA kernels alone do not tell the full story.
- Host synchronization must also be examined.
- Nsight Systems timelines are often more informative than kernel-level metrics alone.

Following the synchronization path—from CUDA kernels to host events and finally back to Python source code—was ultimately what revealed the bottleneck.

---

## TL;DR

- Nsight Systems revealed repeated 8-byte Device-to-Host memcpy operations inside `compute_loss`.
- GPU utilization remained unexpectedly low despite no visible CUDA kernel bottleneck.
- Root-cause analysis traced the synchronization to a Python conditional evaluating CUDA tensors.
- Replacing Python branching with GPU tensor operations can potentially eliminate unnecessary host synchronization while preserving the same computation.