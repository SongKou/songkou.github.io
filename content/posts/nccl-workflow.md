+++
title = 'Who Chooses What? The NCCL Workflow from Parallelism to GPU Communication'
date = 2026-09-09T12:00:00+08:00
draft = false
categories = ['Network']
tags = ['NCCL', 'GPU', 'Distributed Training', 'Tensor Parallelism', 'Pipeline Parallelism', 'Data Parallelism', 'AllReduce', 'NVLink', 'AI Networking']
+++

> **Note:** this post corrects a wrong mental model I had after studying the two companion posts below. Every claim was cross-checked against NVIDIA's NCCL user guide, the NCCL tuning blog, and the Megatron Core documentation — all linked in the references.

After studying [TP, PP, and DP](/posts/tp-pp-dp-llm-parallelism/) and [NCCL and NVLink](/posts/nvlink-nccl-scaleup-scaleout/), I initially compressed the process into two steps:

1. Choose TP, PP, or DP based on workload size and concurrent users.
2. Detect the network topology and choose something like Reduce or AllReduce to synchronize GPUs.

That description mixes decisions made at different layers. **The application or framework decides how to partition the work and what communication result it needs. NCCL decides how to carry out the requested communication.**

NVIDIA describes NCCL as a library of topology-aware GPU communication primitives, rather than a complete parallel programming framework. It supports both collectives and point-to-point communication. [1]

## 1. Four decisions that should stay separate

| Decision | Examples | Owner |
|---|---|---|
| How is the workload distributed? | Tensor, pipeline, and data parallelism | User, framework, or planner |
| What communication result is required? | AllReduce, AllGather, ReduceScatter, Send/Recv | Application or framework |
| How is that operation implemented? | Ring, Tree, or hardware-specific algorithms | NCCL, subject to configuration and support |
| How is execution organized? | Protocol, chunk sizes, communication resources | NCCL, subject to configuration and support |

The hardware provides the available links and capabilities. NCCL uses topology information to organize communication over supported paths such as NVLink, PCIe, and network transports. [1][4]

### Parallelism is a framework decision

TP splits computation within layers. PP divides layers into stages. DP replicates a model—or a model-parallel group—to process different data or requests. These strategies can be combined. [2]

The decision depends on model memory, activations or KV cache, sequence length, batch size, GPU count, interconnect performance, and performance targets. Total training dataset size alone does not determine the partition. Concurrent users matter for serving capacity, but NCCL does not receive a user count and choose a deployment layout.

For example, a deployment might use **TP=4 × DP=2**: two replicas, each distributed over four GPUs. The framework defines those groups before asking the communication backend to move tensors between their members.

### A collective is a contract

Reduce and AllReduce are different operations:

- **Reduce:** combine the inputs and place the result on one designated rank.
- **AllReduce:** combine the inputs and place the result on every participating rank.
- **AllGather:** collect each rank's contribution so every rank receives all contributions, concatenated in rank order—no reduction is applied.
- **ReduceScatter:** combine the inputs, then give each rank a different portion of the reduced result.

NCCL cannot replace a requested AllReduce with Reduce simply because one is cheaper: the application would receive the wrong result. The implementation can change; the requested semantics must remain intact. [3]

## 2. The workflow

![NCCL workflow showing framework decisions, communicator setup, repeated communication execution, and shutdown](/posts/nccl-workflow/nccl-workflow.svg)

The diagram describes the conventional framework-to-NCCL host API path. It separates communicator setup from repeated execution; since NCCL 2.22, some connection setup can be deferred until an algorithm is first used. [9]

### Step 1 — Define parallelism and GPU groups

The framework assigns computation to GPUs and determines which ranks communicate together. A rank identifies a participant within a group. A GPU may participate in several groups—for example, one TP group and one DP group.

NCCL receives a rank-to-device mapping. It does not decide which layers or requests belong on each GPU. [2][5]

### Step 2 — Initialize communicators

A communicator represents a set of participating GPUs and the state needed for their communication.

In a typical multi-process initialization, the application distributes an NCCL unique ID through an external rendezvous mechanism (MPI, a TCP store, or any other CPU-side channel), then each participant initializes its communicator with its rank and group size. Frameworks normally hide these details behind their process-group APIs. [5]

### Step 3 — Discover topology and prepare communication structures

NCCL examines the topology and capabilities available to the group, including GPU connectivity and network paths. This information supports the communication graphs and transport choices used to execute operations. [1][4]

This is not an unrestricted discovery of every switch and route in the network. NCCL's view depends on the platform, exposed topology information, and network implementation. The physical fabric still determines what connectivity and bandwidth are available.

### Step 4 — Receive a communication request

During execution, the framework produces a tensor and requests an operation. Conceptually, an AllReduce request says:

```text
Combine these input buffers using SUM.
Use this datatype and element count.
Include the ranks in this communicator.
Write the result to these output buffers.
Enqueue the work on this CUDA stream.
```

The participating ranks must issue compatible operations with matching requirements. NCCL cannot repair an application in which one rank requests AllReduce while another expects a different collective. [3]

### Step 5 — Plan the requested operation

NCCL considers the operation, its message size, communicator dimensions, topology, and additional execution details. Its cost model selects an algorithm and protocol; its scheduler determines execution details such as chunking and GPU resource allocation. Tuner plugins and configuration can influence these choices. [4]

**Message size means the tensor payload for this communication operation—not the size of the training corpus.**

Ring and Tree are examples of algorithms. Simple, LL, and LL128 are protocol choices with different bandwidth and latency tradeoffs. The supported combinations depend on the operation, hardware, and NCCL version. Automatic selection estimates a good choice; it does not guarantee the globally fastest result on every system. (The `NCCL_ALGO` / `NCCL_PROTO` environment variables can force a choice, but NVIDIA recommends them for benchmarking rather than production; tuner plugins are the supported mechanism.) [4]

### Step 6 — Execute and satisfy dependencies

NCCL enqueues communication on a CUDA stream. A successful return from an ordinary call means the work has been enqueued, not that the result is already ready for the CPU or another stream. CUDA stream ordering, events, or synchronization establish when dependent computation can consume it. [6]

Independent computation can overlap communication. The framework is responsible for scheduling that work and expressing dependencies correctly.

### Step 7 — Reuse, then release

The application reuses communicators across many operations. It does not rebuild the whole topology for every gradient bucket or generated token. At shutdown, it completes outstanding work and releases communicator resources. [5]

## 3. What TP, PP, and DP ask NCCL to do

These are common patterns, rather than a mandatory one-to-one mapping:

| Parallelism | Typical communication |
|---|---|
| Replicated DP training | AllReduce gradient buckets across replicas |
| Sharded DP / distributed optimizer | ReduceScatter gradients and AllGather parameters |
| TP | AllReduce partial results; AllGather and ReduceScatter for some tensor layouts |
| PP | Send/Recv activations forward and activation gradients backward between stages |
| Independent dense-model inference replicas | No gradient synchronization between replicas |

NVIDIA's Megatron documentation describes the training patterns, including TP collectives and PP point-to-point communication. [2][7][8]

The distinction between training DP and inference replicas matters. Independent serving replicas do not run a backward pass and exchange gradients. Each replica can still use TP or PP internally, and systems that combine data and expert parallelism can introduce communication between otherwise separate request-processing ranks. [7]

## 4. A concrete AllReduce example

Suppose four training replicas each produce one scalar gradient:

```text
Rank 0: 1
Rank 1: 2
Rank 2: 3
Rank 3: 4
```

The framework requests **AllReduce with SUM**. After completion, every rank holds **10**. If the training implementation averages across the four replicas, the gradient used for the update is **2.5**.

The framework chose the mathematical operation. NCCL chose how to exchange and reduce the values. Ring, Tree, or another supported implementation must preserve the same AllReduce semantics, allowing for ordinary floating-point reduction-order differences. [3]

NCCL's documentation notes that executing ReduceScatter followed by AllGather is equivalent to AllReduce, and ring implementations exploit exactly this structure: a ReduceScatter phase that circulates and combines partial sums, then an AllGather phase that distributes the completed results. That explains the movement of data, but does not mean every NCCL AllReduce is executed as two separate public API calls. [3]

## 5. A more accurate two-step description

The original summary becomes:

1. **The user or framework selects and combines parallelism strategies**, assigns work to GPUs, and determines the communication operations required by the computation.
2. **NCCL initializes communication groups and uses topology and operation characteristics to execute those requests**, selecting supported algorithms, protocols, and execution settings.

When reasoning about performance, first ask which layer owns the decision. Changing the parallelism layout, changing the requested communication pattern, and tuning NCCL are three different ways to change the system.

## References

1. [NVIDIA — Overview of NCCL](https://docs.nvidia.com/deeplearning/nccl/user-guide/docs/overview.html)
2. [NVIDIA Megatron Core — Parallelism Strategies Guide](https://docs.nvidia.com/megatron-core/developer-guide/latest/user-guide/parallelism-guide.html)
3. [NVIDIA — Collective Operations](https://docs.nvidia.com/deeplearning/nccl/user-guide/docs/usage/collectives.html)
4. [NVIDIA — Understanding NCCL Tuning to Accelerate GPU-to-GPU Communication](https://developer.nvidia.com/blog/understanding-nccl-tuning-to-accelerate-gpu-to-gpu-communication/)
5. [NVIDIA — Creating a Communicator](https://docs.nvidia.com/deeplearning/nccl/user-guide/docs/usage/communicators.html)
6. [NVIDIA — CUDA Stream Semantics](https://docs.nvidia.com/deeplearning/nccl/user-guide/docs/usage/streams.html)
7. [NVIDIA Megatron Core — Communication and Parallelism in MoE](https://docs.nvidia.com/megatron-core/developer-guide/latest/user-guide/features/moe.html)
8. [NVIDIA Megatron Core — Tensor Parallel Layers](https://docs.nvidia.com/megatron-core/developer-guide/latest/apidocs/core/core.tensor_parallel.layers.html)
9. [NVIDIA — Memory Efficiency, Faster Initialization, and Cost Estimation with NCCL 2.22](https://developer.nvidia.com/blog/memory-efficiency-faster-initialization-and-cost-estimation-with-nvidia-collective-communications-library-2-22/)
