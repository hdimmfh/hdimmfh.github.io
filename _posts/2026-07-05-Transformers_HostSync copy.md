---
title: Contributing to Transformers — Eliminating Hidden Host Synchronization
by: hdimmfh
date: 2026-07-04 16:00:00 +0900
categories: [GPU, CUDA]
tags: [CUDA, DeepSpeed, Transformers, Nsight, PyTorch, Open Source]
---

🔍 **Finding a Hidden GPU Synchronization Bottleneck**

> While profiling DeepSpeed Sequence Parallel training with Nsight Systems, I found an unexpected performance gap inside the `compute_loss` region. The GPU appeared idle, yet no CUDA kernel explained the delay. The root cause eventually led to a Python conditional evaluating CUDA tensors.

---

## ① The Symptom

During distributed training on a **2-node NVIDIA B300 cluster (16 GPUs)**, I profiled the forward pass using `Nsight Systems`. Surprisingly, the `compute_loss` NVTX region contained a long idle section where almost every GPU metric dropped.

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

Since `good_tokens_per_rank` contains CUDA tensors returned by `all_gather`, the Python conditional may require evaluating a GPU value on the host. That evaluation can introduce an implicit synchronization, which appears in Nsight Systems as repeated small Device-to-Host memcpy operations together with host-side waiting. Although each synchronization transfers only **8 bytes**, repeating this operation many times creates noticeable bubbles in the execution timeline.

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

This keeps the filtering on the GPU and avoids evaluating CUDA tensors inside Python. Whether this implementation is appropriate depends on the intended semantics (for example, handling zero-token ranks and NaN propagation), but it illustrates how the host synchronization can potentially be removed.

---

## ⑤ Upstream Contribution

After confirming the root cause, I opened an issue in the Hugging Face Transformers repository to discuss the unexpected host synchronization observed during Sequence Parallel loss computation.

The discussion concluded that the Python conditional evaluating CUDA tensors could indeed introduce unnecessary host synchronization. A follow-up pull request replaced the Python-side branching with a GPU-side tensor implementation while preserving the original semantics.

The contribution was reviewed by the Transformers maintainers and ultimately merged into the upstream project.

**Related links**

- Issue: https://github.com/huggingface/transformers/issues/47068
- Pull Request: https://github.com/huggingface/transformers/pull/47073

## ⑥ Lessons Learned

This investigation reinforced an important lesson about GPU performance analysis.

Performance bottlenecks are not always caused by CUDA kernels. In this case, a single Python conditional evaluating CUDA tensors introduced repeated host synchronization that became visible only in the system timeline. The investigation ultimately led to an upstream improvement in Hugging Face Transformers, demonstrating how system-level profiling can uncover optimization opportunities beyond CUDA kernel implementations.

When profiling distributed training,

- CUDA kernels alone do not tell the full story.
- CPU-side synchronization deserves equal attention.
- Nsight Systems often reveals bottlenecks that kernel-level profilers cannot.
- Small synchronizations repeated at scale can become measurable training overhead.

---

## TL;DR

- Nsight Systems revealed repeated 8-byte Device-to-Host memcpy operations during Sequence Parallel loss computation.
- The synchronization originated from a Python conditional evaluating CUDA tensors.
- Replacing Python branching with GPU tensor operations removed the unnecessary host synchronization.
- The optimization was contributed upstream and merged into Hugging Face Transformers.