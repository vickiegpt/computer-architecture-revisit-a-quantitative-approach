# TPU Architecture Deep Dive

## Overview

Google's Tensor Processing Unit (TPU) is a domain-specific accelerator designed for neural network workloads, particularly matrix multiplication operations that dominate deep learning inference and training.

## Single Chip Architecture

A TPU chip contains **two TensorCores** sharing:
- **CMEM (Common Memory)**: 128 MiB
- **HBM (High Bandwidth Memory)**: 32 GiB

### TensorCore Components

Each TensorCore comprises:

| Component | Description | Size |
|-----------|-------------|------|
| **MXU (Matrix Multiply Unit)** | 128×128 systolic array - primary compute engine | - |
| **VPU (Vector Processing Unit)** | Handles elementwise operations (ReLU, reductions) | - |
| **VMEM (Vector Memory)** | Buffer for data from HBM before computation | 32 MiB |
| **Scalar Unit + SMEM** | Control flow and address generation | 10 MiB |

## Systolic Arrays

A systolic array is a hardware architecture consisting of a grid of interconnected processing elements (PEs). Each PE performs multiply-accumulate (MAC) operations and passes results to neighbors.

### How Systolic Arrays Work

```
        Weight Input (top)
            ↓   ↓   ↓   ↓
          ┌───┬───┬───┬───┐
Input  →  │PE │PE │PE │PE │  → Output
(left)    ├───┼───┼───┼───┤
       →  │PE │PE │PE │PE │  →
          ├───┼───┼───┼───┤
       →  │PE │PE │PE │PE │  →
          ├───┼───┼───┼───┤
       →  │PE │PE │PE │PE │  →
          └───┴───┴───┴───┘
```

Each PE performs: `acc += input × weight`

### Key Advantages

1. **Minimal external control logic** - once data enters, computation flows automatically
2. **No intermediate memory reads/writes** - only input and output access memory
3. **Perfect for fixed dataflow patterns** - matrix multiplication is highly regular
4. **Data reuse** - each value is used multiple times as it flows through the array

### Critical Limitation

**Systolic arrays show no performance gain for sparse matrices** - they perform the same number of cycles regardless of zero-valued elements. The regular dataflow pattern cannot skip zeros.

## Dataflow Strategies

### Weight Stationary

- Weights are preloaded and stay fixed in PEs
- Input activations flow through the array
- Good for batch processing with same weights

### Output Stationary

- Partial sums accumulate in place
- Inputs and weights flow through
- Reduces write bandwidth to output

### Input Stationary

- Input activations stay fixed
- Weights flow through
- Less common in practice

## Design Philosophy

### 1. Systolic Arrays + Pipelining

The architecture overlaps computation with data movement through pipeline stages:

```
Time →
Stage 1: [Load W1] [Load W2] [Load W3] ...
Stage 2:          [Comp W1] [Comp W2] [Comp W3] ...
Stage 3:                    [Store R1] [Store R2] ...
```

This enables continuous processing without stalls.

### 2. Ahead-of-Time (AoT) Compilation

The XLA compiler generates **fully static binary** programs by analyzing computation graphs beforehand:

- Memory access patterns are highly predictable
- No need for traditional caches
- **Dramatically reduced energy consumption**

Memory operations consume **orders of magnitude more energy** than arithmetic. TPUs minimize memory accesses through compiler-driven data placement.

## Multi-Chip Hierarchy

### Tray Level (4 chips)
- Connected via PCIe to CPU host
- Chip-to-chip uses **Inter-Core Interconnect (ICI)**

### Rack Level (64 chips)
- Forms a **4×4×4 3D torus topology**
- ICI for nearest-neighbor connections
- **Optical Circuit Switching (OCS)** for wraparound links

OCS reduces worst-case hops from `N-1` to `(N-1)/2` per axis.

### Pod/Superpod (4,096 chips for TPUv4)
- Maximum ICI-connected configuration
- Flexible slice configurations:
  - Cube topology
  - Cigar topology
  - Rectangle topology

### Multi-Pod
- Uses slower Data-Center Network (DCN) between pods

## Optical Circuit Switching (OCS) Benefits

1. **Wraparound**: Reduces communication latency through ring topology
2. **Noncontiguous Slices**: Allows treating entire pod as "bag of nodes" - enables multiple jobs and flexible scheduling
3. **Twisted Topologies**: Accelerates all-to-all communications by rewiring fixed dimensions

## TPU vs GPU Comparison

| Aspect | TPU | GPU |
|--------|-----|-----|
| **Compute Pattern** | Systolic array (regular) | SIMT (flexible) |
| **Memory Hierarchy** | Compiler-managed, no cache | Large caches, dynamic |
| **Sparsity Handling** | No benefit | Can skip (with structured sparsity) |
| **Control Flow** | Static, determined at compile | Dynamic branching |
| **Energy Efficiency** | Higher (fewer memory accesses) | Lower (more cache traffic) |
| **Programming Model** | XLA/JAX (graph-based) | CUDA (explicit) |

## Software Stack

### XLA Compiler
- Inserts **hierarchical collectives** for TPU topology automatically
- Uses **GSPMD (General and Scalable Parallelization)**
- Abstracts hardware complexity from researchers

### Programming Interfaces
- JAX (primary)
- TensorFlow
- PyTorch/XLA

## Matrix Multiplication on Systolic Array

For C = A × B where A is M×K and B is K×N:

```
Cycle 1:  a[0,0] enters, w[0,0] in PE(0,0)
Cycle 2:  a[0,1] enters, a[0,0] moves right; partial sum flows down
...
Cycle K:  First output element c[0,0] ready
Cycle K+M+N-2: All outputs complete
```

The systolic array achieves **O(K) cycles** for M×K × K×N multiplication, with **O(M×N×K) total operations** distributed across **M×N PEs**.

## Reference

1. [TPU Architecture Blog](https://henryhmko.github.io/posts/tpu/tpu.html)
2. [Google TPU Research Papers](https://cloud.google.com/tpu/docs/system-architecture-tpu-vm)
3. [Systolic Arrays for Matrix Operations](https://en.wikipedia.org/wiki/Systolic_array)
