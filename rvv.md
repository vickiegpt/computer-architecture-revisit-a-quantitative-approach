# RISC-V Vector Extension (RVV) and VLIW Perspectives

## Is VLIW View of Committing to GPU Good or Bad?

The VLIW (Very Long Instruction Word) approach to GPU/accelerator commit has both advantages and disadvantages:

### VLIW Characteristics
- **Static scheduling**: Compiler determines parallelism at compile time
- **Wide instruction word**: Multiple operations packed into one fetch
- **Predictable execution**: No dynamic scheduling hardware needed
- **Deterministic latency**: Easier to reason about timing

### Why RISC-V Vector Extension Has VLIW-like Properties

RVV requires setting configuration registers (CSRs) before launching vector operations:

```asm
# Set vector configuration
vsetvli t0, a0, e32, m4    # Set: SEW=32bit, LMUL=4
                            # Returns actual VL in t0

# Now launch vector operations
vle32.v v0, (a1)           # Vector load
vadd.vv v4, v0, v8         # Vector add
vse32.v v4, (a2)           # Vector store
```

This is analogous to **launching CUDA kernels** with predefined GPU context:
- Block dimensions
- Shared memory size
- SM parameters

### The Control Flow Model

```
┌─────────────────────────────────────────┐
│           Scalar Core (Control)          │
│  ┌─────────────────────────────────┐    │
│  │  1. Set vtype, vl (MSR regs)    │    │
│  │  2. Dispatch vector ops         │    │
│  │  3. Wait for completion         │    │
│  └─────────────────────────────────┘    │
└────────────────┬────────────────────────┘
                 │ RoCC / Vector Interface
                 ▼
┌─────────────────────────────────────────┐
│           Vector Unit (Execution)        │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐    │
│  │ Lane 0  │ │ Lane 1  │ │ Lane N  │    │
│  │ SIMD    │ │ SIMD    │ │ SIMD    │    │
│  └─────────┘ └─────────┘ └─────────┘    │
└─────────────────────────────────────────┘
```

## RoCC: RISC-V Coprocessor Interface

The Vector Unit in BOOMv3 uses the **RoCC (Rocket Custom Coprocessor)** protocol - a VLIW-like interface for accelerators.

### RoCC Interface Signals

```scala
class RoCCCommand extends Bundle {
  val inst = new RoCCInstruction  // Custom instruction
  val rs1 = UInt(xLen.W)          // Source register 1
  val rs2 = UInt(xLen.W)          // Source register 2
}

class RoCCResponse extends Bundle {
  val rd = UInt(xLen.W)           // Destination value
}
```

### Why RoCC is VLIW-like

1. **Decoupled execution**: Vector unit runs independently
2. **Bulk dispatch**: Single instruction triggers many operations
3. **Static resource allocation**: VL/VTYPE set before execution
4. **Coarse-grain synchronization**: fence.v for completion

### RoCC vs Traditional OoO Integration

| Aspect | RoCC (VLIW-like) | Full OoO Integration |
|--------|------------------|---------------------|
| **Scheduling** | Static (compiler) | Dynamic (hardware) |
| **ROB tracking** | Token-based | Per-element |
| **Recovery** | Coarse (re-execute) | Fine-grain |
| **Complexity** | Lower | Higher |
| **Flexibility** | Less | More |

## Tenstorrent's Approach to RISC-V Vector Extension

Tenstorrent takes a unique approach combining RISC-V scalar cores with their custom vector/tensor engines.

### Architecture Overview

```
┌──────────────────────────────────────────────────────┐
│                  Tenstorrent Chip                     │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐      │
│  │   RISC-V   │  │   RISC-V   │  │   RISC-V   │      │
│  │   Core     │  │   Core     │  │   Core     │      │
│  └─────┬──────┘  └─────┬──────┘  └─────┬──────┘      │
│        │               │               │              │
│        ▼               ▼               ▼              │
│  ┌─────────────────────────────────────────────┐     │
│  │         Tensix Core / Vector Engine          │     │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐   │     │
│  │  │  FPU     │  │  Matrix  │  │  Vector  │   │     │
│  │  │  Unit    │  │  Engine  │  │  Unit    │   │     │
│  │  └──────────┘  └──────────┘  └──────────┘   │     │
│  └─────────────────────────────────────────────┘     │
└──────────────────────────────────────────────────────┘
```

### Key Design Decisions

#### 1. Decoupled Vector Engine
- RISC-V cores handle control flow
- Vector/tensor operations offloaded to Tensix cores
- Similar to RoCC but more sophisticated

#### 2. Data-Centric Execution
```
Traditional: Bring data to compute
Tenstorrent: Bring compute to data (dataflow)
```

#### 3. VLIW-style Dispatch
- Pack multiple operations per cycle
- Compiler schedules resource usage
- Reduces control overhead

### Tenstorrent Vector vs Standard RVV

| Feature | Standard RVV | Tenstorrent |
|---------|--------------|-------------|
| **Vector Length** | Variable (vl) | Fixed tiles |
| **Data Types** | General purpose | AI-optimized (BF16, INT8) |
| **Memory Model** | Load/store | Explicit DMA |
| **Scheduling** | Mixed | VLIW/dataflow |
| **Target** | General compute | ML workloads |

### Why This Matters for ML

```
Matrix Multiply: C[M,N] = A[M,K] × B[K,N]

Standard RVV approach:
  for i in range(M):
    for j in range(N):
      vsetvli...
      vle... vfmacc... vse...

Tenstorrent approach:
  - Tile A, B into blocks
  - Stream tiles through matrix engine
  - Dataflow between operations
```

## Vector-ROB Integration Challenges

### The Commit Problem

Vector instructions produce many results but appear as single ROB entry:

```
vadd.vv v0, v4, v8    # With VL=256, this is 256 operations!
                      # But only ONE ROB entry

# What if element 150 faults?
```

### Solutions

#### 1. Imprecise Exceptions (Simpler)
- Re-execute entire vector op on fault
- Lose progress but simpler hardware

#### 2. First-Fault Only (RVV approach)
```asm
vle32ff.v v0, (a1)    # First-fault load
csrr t0, vl           # Check how many succeeded
# Handle partial completion
```

#### 3. Chunked Commit
- Break vector op into chunks
- Each chunk gets ROB tracking
- More precise but complex

### BOOMv3 Vector Implementation

```
BOOM ROB Entry:
┌─────────────────────────────────────────┐
│ PC | Type=VECTOR | VL | Completed_mask  │
└─────────────────────────────────────────┘
       │
       │ On completion:
       ▼
┌─────────────────────────────────────────┐
│ Check: all elements done?               │
│ Check: any exceptions?                  │
│ → Commit or handle fault                │
└─────────────────────────────────────────┘
```

## Performance Implications

### VLIW Advantages for Vectors
1. **Lower power**: No dynamic scheduling
2. **Predictable latency**: Compiler knows timing
3. **Simpler hardware**: Fewer OoO structures
4. **Better for throughput**: Batch processing

### VLIW Disadvantages
1. **Compiler complexity**: Static scheduling hard
2. **Code bloat**: NOPs for unused slots
3. **Inflexibility**: Can't adapt to runtime conditions
4. **Memory latency**: Can't hide variable delays

### When VLIW Works Well
- Regular, predictable workloads (ML, DSP)
- High data parallelism
- Known memory access patterns
- Batch processing

### When VLIW Struggles
- Irregular control flow
- Pointer-chasing workloads
- Variable-latency operations
- Dynamic workloads

## The Hybrid Future

Modern designs combine approaches:

```
┌─────────────────────────────────────────────────┐
│                 Hybrid Architecture              │
│                                                  │
│  ┌──────────────┐    ┌──────────────────────┐   │
│  │ OoO Scalar   │ ←→ │ VLIW-style Vector    │   │
│  │ Core (ROB)   │    │ Engine (Dataflow)    │   │
│  └──────────────┘    └──────────────────────┘   │
│         │                      │                 │
│         │    Coherent Memory   │                 │
│         └──────────┬───────────┘                 │
│                    ▼                             │
│  ┌─────────────────────────────────────────┐    │
│  │            Shared L2/L3 Cache            │    │
│  └─────────────────────────────────────────┘    │
└─────────────────────────────────────────────────┘
```

This allows:
- Fine-grain control on scalar core
- Bulk throughput on vector engine
- Efficient synchronization between domains

## Summary

| Aspect | Traditional OoO | VLIW Vector | Hybrid (RVV/Tenstorrent) |
|--------|-----------------|-------------|--------------------------|
| Control | Dynamic | Static | Mixed |
| Scheduling | Hardware | Compiler | Both |
| Exceptions | Precise | Imprecise | First-fault |
| Use Case | General | Regular | ML/HPC |

The VLIW view of vector commit is neither purely good nor bad - it's a **tradeoff** between hardware complexity and software flexibility, optimized for different workload characteristics.

## Reference

1. [BOOMv3 RoCC](https://github.com/riscv-boom/riscv-boom)
2. [Tenstorrent Vector Engine](https://www.youtube.com/watch?v=H253qdZjbhQ)
3. [RISC-V Vector Extension Spec](https://github.com/riscv/riscv-v-spec)
4. [RoCC Interface Documentation](https://chipyard.readthedocs.io/en/latest/Customization/RoCC-Accelerators.html)
