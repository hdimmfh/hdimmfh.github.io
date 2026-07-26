---
title: Inside PyTorch(2) — How PyTorch Compiles for GPUs
by: hdimmfh
date: 2026-07-26 13:00:00 +0900
categories: [GPU, PyTorch]
tags: [PyTorch, TorchInductor, Triton, CUDA Graphs, torch.compile]
---

🔍 **Does TorchInductor Build CUDA Graphs from Triton Kernels?**

> Broadly, yes. TorchInductor optimizes the PyTorch computation graph and commonly generates GPU kernels through Triton. CUDA Graphs can then capture those kernels together with other CUDA operations and replay the prepared workload with lower CPU launch overhead.

---

## ① GPU Kernels and CPU Launches

A GPU kernel is a function executed in parallel by many GPU threads. For example, the following PyTorch code contains three tensor operations.

```python
x = a + b
y = torch.relu(x)
z = y * c
```

Without compiler fusion, these operations may result in multiple GPU kernel launches.

```text
add kernel
    ↓
relu kernel
    ↓
multiply kernel
```

The CPU does not perform these calculations itself. Instead, host code uses the CUDA Runtime and Driver to submit work to a CUDA stream.

```text
CPU
    ↓
CUDA Runtime / Driver
    ↓
CUDA Stream
    ↓
GPU Kernel
```

A kernel launch is normally asynchronous. The CPU submits the work, and the GPU executes it when its dependencies are satisfied. This means GPU execution has two different costs.

* **Kernel execution time:** the time spent performing the calculation on the GPU.
* **Kernel launch overhead:** the CPU-side cost of preparing and submitting the kernel.

When many short-lived kernels are launched, CPU launch overhead can become a bottleneck even if each kernel performs only a small amount of work.

---

## ② Why Did Triton and TorchInductor Appear?

Writing efficient GPU kernels directly in CUDA C++ requires detailed control over parallelism, memory access, synchronization, and hardware-specific behavior.

Triton provides a different programming model. It is a Python-based language and compiler for writing parallel GPU programs. Developers describe work at a higher level than CUDA threads while still controlling important details such as memory loads, reductions, tiling, and program instances. A simplified Triton kernel looks like this.

```python
@triton.jit
def add_kernel(
    x_ptr,
    y_ptr,
    output_ptr,
    n_elements: tl.constexpr,
):
    offsets = tl.program_id(0) * BLOCK_SIZE + tl.arange(0, BLOCK_SIZE)
    mask = offsets < n_elements

    x = tl.load(x_ptr + offsets, mask=mask)
    y = tl.load(y_ptr + offsets, mask=mask)

    tl.store(output_ptr + offsets, x + y, mask=mask)
```

Triton makes individual GPU kernels easier to express, but it does not decide how an entire PyTorch model should be optimized.

That is the role of TorchInductor. PyTorch Eager Mode dispatches each tensor operation for execution as soon as it is encountered, without first optimizing the surrounding computation graph.

```text
Python Program
    ↓
PyTorch Operation
    ↓
Execute Immediately
    ↓
ATen / CUDA Kernel
```

Because each operation is handled separately, Eager Mode has limited visibility into the surrounding computation. `torch.compile()` changes this execution path.

```text
PyTorch Program
    ↓
TorchDynamo
    ↓
AOTAutograd
    ↓
TorchInductor
    ↓
Optimized CPU or GPU Code (Triton Kernel or ATen / cuBLAS / cuDNN)
```

`TorchDynamo` captures compilable regions of the Python program. `AOTAutograd` prepares forward and backward graphs. `TorchInductor` then analyzes those graphs and **generates optimized code** for the target device.

---

## ③ TorchInductor vs Triton

TorchInductor and Triton are both compilers, but they operate at different levels.

| TorchInductor                                      | Triton                                                 |
| -------------------------------------------------- | ------------------------------------------------------ |
| Receives PyTorch computation graphs                | Receives Triton kernel programs                        |
| Examines multiple tensor operations                | Implements an individual GPU kernel                    |
| Decides fusion and scheduling                      | Compiles the kernel for GPU execution                  |
| Plans intermediate buffers and memory reuse        | Expresses loads, stores, reductions, and parallel work |
| May generate Triton code or use external libraries | Does not optimize the entire PyTorch model             |

Consider the following expression.

```python
output = torch.relu((a + b) * c)
```

TorchInductor may determine that the addition, multiplication, and activation can be fused.

```text
FX Graph
    │
    ├── add
    ├── multiply
    └── relu
```

Instead of launching three independent kernels, TorchInductor can generate one fused Triton kernel.

```text
triton_fused_add_mul_relu
```

The resulting execution changes from this:

```text
launch add
launch multiply
launch relu
```

to this:

```text
launch fused_add_mul_relu
```

Fusion can reduce:

1. the number of kernel launches,
2. intermediate tensor allocations,
3. repeated global-memory reads and writes.

However, TorchInductor does not generate every GPU operation as a Triton kernel.

Operations such as matrix multiplication or convolution may use optimized implementations from ATen, cuBLAS, cuDNN, or compiler-generated templates. A compiled region can therefore contain a mixture of Triton kernels and external library calls.

The relationship is better represented as follows.

```text
PyTorch FX Graph
        ↓
TorchInductor
        │
        ├── Fusion
        ├── Scheduling
        ├── Memory Planning
        └── Code Generation
                │
        ┌───────┴────────┐
        ↓                ↓
Triton Kernels     External Libraries
                   ATen / cuBLAS / cuDNN
```

`TorchInductor` determines what code should be generated or called.`Triton` helps implement and compile GPU kernels selected by that process.

---

## ④ Why Are CUDA Graphs Still Needed?

TorchInductor and Triton reduce the amount of GPU work by applying optimizations such as kernel fusion. They do not necessarily eliminate CPU launch overhead. A compiled model may still execute several GPU operations.

```text
Triton fused kernel
        ↓
cuBLAS matrix multiplication
        ↓
Triton reduction kernel
        ↓
ATen CUDA kernel
```

Without CUDA Graphs, the CPU submits these operations separately during every iteration.

```text
Iteration 1:
launch A
launch B
launch C

Iteration 2:
launch A
launch B
launch C

Iteration 3:
launch A
launch B
launch C
```

Even when the same sequence repeats, the CPU and CUDA Driver must perform launch-related work again. `CUDA Graphs` introduce a **different work-submission model**. A graph describes CUDA operations and their dependencies separately from execution. After the graph has been prepared, it can be launched repeatedly. The lifecycle can be simplified into three steps.

### Definition or Capture

CUDA records the operations and their dependencies.

```text
kernel A
    ↓
kernel B
    ├──────────┐
    ↓          ↓
kernel C     memcpy
```

### Instantiation

The graph definition is converted into an executable graph.

```text
CUDA Graph
    ↓
Executable CUDA Graph
```

### Replay

The executable graph is launched repeatedly.

```text
Iteration 1: graph launch
Iteration 2: graph launch
Iteration 3: graph launch
```

The main improvement is on the CPU side.

**Without CUDA Graphs**

```text
Every iteration:

CPU:
- prepare & launch kernel A
- prepare & launch kernel B
- prepare & launch kernel C
```

**With CUDA Graphs**

```text
First execution:
- capture CUDA workload
- instantiate executable graph

Every iteration:
- launch executable graph
```

`CUDA Graphs` do not necessarily make the code inside an individual `Triton` kernel faster. They reduce the cost of repeatedly submitting the overall GPU workload.

---

## ⑤ What Can a CUDA Graph Contain?

A CUDA Graph is not specific to Triton. 
It can represent multiple types of CUDA operations, including:

* Triton-generated kernels,
* ATen CUDA kernels,
* cuBLAS or cuDNN operations,
* memory copies,
* memory-set operations,
* CUDA events,
* child graphs.

Therefore, the following description is incomplete.

> Triton kernels are converted into a CUDA Graph.

A more accurate description is:

> CUDA Graphs capture a GPU workload that may contain Triton kernels together with other CUDA operations.

The graph represents both the operations and their dependencies. It is not merely a text list of kernel names.

---

## ⑥ How TorchInductor, Triton, and CUDA Graphs Work Together

The complete execution path can be summarized as follows.

```text
PyTorch Program
        │
        ▼
TorchDynamo
Captures a computation graph
        │
        ▼
AOTAutograd
Prepares forward and backward graphs
        │
        ▼
TorchInductor
Fuses and schedules operations
        │
        ├──────────────────┐
        ▼                  ▼
Triton Kernels       Library Operations
                     ATen / cuBLAS / cuDNN
        │                  │
        └────────┬─────────┘
                 ▼
          CUDA Operations
                 │
                 ▼
       CUDA Graph Capture
                 │
                 ▼
      Executable CUDA Graph
                 │
                 ▼
           Graph Replay
```

Each component optimizes a different layer.

| Component     | Optimization Layer                      |
| ------------- | --------------------------------------- |
| TorchInductor | Computation graph and generated program |
| Triton        | Individual GPU kernel implementation    |
| CUDA Graph    | CPU-to-GPU work submission              |

`TorchInductor` decides how the computation should be organized. `Triton` generates GPU kernels for compatible regions. `CUDA Graphs` reduce the overhead of repeatedly launching the resulting GPU workload.

---

## ⑦ What Changed After CUDA Graphs?

Suppose one model iteration contains four short GPU operations.

**Conventional execution**

```text
CPU → launch kernel A
CPU → launch kernel B
CPU → launch kernel C
CPU → launch kernel D
```

The launch path is repeated for every iteration.

**CUDA Graph execution**

```text
CPU → launch executable graph
```

The GPU still performs kernels A, B, C, and D.
The difference is how the work reaches the GPU.

| Before CUDA Graphs                               | With CUDA Graphs                                        |
| ------------------------------------------------ | ------------------------------------------------------- |
| Operations are submitted individually            | A prepared workload is submitted as a graph             |
| CPU launch work repeats for every operation      | Much of the preparation is reused                       |
| Flexible for changing execution paths            | Best suited to stable, repeated workloads               |
| Launch overhead grows with many short operations | CPU overhead can be reduced significantly               |
| Ordinary CUDA memory allocation behavior         | Graph execution may require persistent workspace memory |

CUDA Graphs are most effective for workloads that repeatedly execute the same sequence of GPU operations with stable tensor shapes and execution paths, especially when many short-lived kernels make CPU launch overhead a bottleneck. They are less suitable for workloads with frequently changing tensor shapes or control flow, unsupported operations, or memory layouts that prevent stable address reuse.

PyTorch exposes this optimization through `torch.compile()` modes and options.

```python
compiled_model = torch.compile(
    model,
    mode="reduce-overhead",
)
```

The `reduce-overhead` mode uses CUDA Graphs when the workload is compatible. CUDA Graphs can also be requested through a TorchInductor option.

```python
compiled_model = torch.compile(
    model,
    options={"triton.cudagraphs": True},
)
```

Enabling the option does not guarantee that every compiled region will be captured. PyTorch may skip CUDA Graphs when the workload violates capture requirements.

---

## Key Takeaway

TorchInductor, Triton, and CUDA Graphs do not perform the same optimization.

| TorchInductor                           | Triton                                       | CUDA Graph                                 |
| --------------------------------------- | -------------------------------------------- | ------------------------------------------ |
| Optimizes the PyTorch computation graph | Generates and compiles GPU kernels           | Captures and replays CUDA workloads        |
| Decides fusion and scheduling           | Implements kernel-level parallel computation | Reduces repeated CPU launch overhead       |
| Operates above individual kernels       | Operates at the kernel level                 | Operates at the execution-submission level |

`CUDA Graphs` do not replace Triton. `Triton` does not replace TorchInductor. They optimize different layers of the same execution path.

---

## TL;DR

Q. What does TorchInductor use to generate GPU kernels?

> TorchInductor commonly uses Triton as a key GPU code-generation component. It can also call ATen kernels, compiler templates, and optimized libraries such as cuBLAS or cuDNN.

Q. Are TorchInductor and Triton the same compiler?

> No. TorchInductor optimizes a PyTorch computation graph, while Triton implements and compiles individual GPU kernels.

Q. What does Triton optimize?

> Triton expresses kernel-level parallel computation, including memory access, tiling, reductions, and GPU program instances.

Q. Does generating Triton kernels remove CPU overhead?

> Not completely. Kernel fusion can reduce the number of launches, but the CPU must still submit the remaining GPU operations.

Q. What problem do CUDA Graphs solve?

> CUDA Graphs reduce CPU launch overhead by defining or capturing a repeated CUDA workload once and launching the prepared executable graph multiple times.

Q. Do CUDA Graphs make individual Triton kernels faster?

> Not necessarily. The kernel normally performs the same GPU calculation. CUDA Graphs optimize how the overall workload is submitted.

Q. Do CUDA Graphs contain only Triton kernels?

> No. A graph may contain Triton kernels, ATen kernels, library operations, memory copies, events, and other CUDA operations.

Q. Does `inductor_cudagraphs` mean that CUDA Graphs contain kernels generated by Triton?

> Broadly, yes. A more complete answer is:
> TorchInductor generates or selects GPU operations, including Triton kernels, and PyTorch can capture those operations into a CUDA Graph for repeated execution. Triton does not create the CUDA Graph itself, and the graph is not limited to Triton kernels.

---

## References

* [PyTorch Compiler](https://docs.pytorch.org/docs/stable/user_guide/torch_compiler/torch.compiler.html)
* [`torch.compile` API](https://docs.pytorch.org/docs/stable/generated/torch.compile.html)
* [PyTorch CUDAGraph API](https://docs.pytorch.org/docs/stable/generated/torch.cuda.CUDAGraph.html)
* [PyTorch CUDAGraph Trees](https://docs.pytorch.org/docs/2.9/torch.compiler_cudagraph_trees.html)
* [Triton Documentation](https://triton-lang.org/)
* [Triton Fused Softmax Tutorial](https://triton-lang.org/main/getting-started/tutorials/02-fused-softmax.html)
* [CUDA Programming Model](https://docs.nvidia.com/cuda/cuda-programming-guide/01-introduction/programming-model.html)
* [CUDA Graphs](https://docs.nvidia.com/cuda/cuda-programming-guide/04-special-topics/cuda-graphs.html)
* [Original PyTorch Developer Discussion](https://dev-discuss.pytorch.org/t/torchinductor-a-pytorch-native-compiler-with-define-by-run-ir-and-symbolic-shapes/747/3)
