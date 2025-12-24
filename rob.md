# ROB in Detail

## What is a Reorder Buffer?

The **Reorder Buffer (ROB)** is a hardware structure that enables out-of-order execution while maintaining the illusion of in-order completion. It's the key mechanism for:

1. **Precise exceptions** - handling interrupts at exact program points
2. **Branch misprediction recovery** - rolling back speculative state
3. **Register renaming** - eliminating false dependencies (WAR, WAW)

---

## Intel Skylake Pipeline Overview

Based on Matt Godbolt's deep dive into Skylake microarchitecture:

```
┌─────────────────────────────────────────────────────────────────────┐
│                          FRONTEND                                    │
│  ┌──────────┐   ┌───────────┐   ┌──────────────┐   ┌────────────┐  │
│  │  Branch  │──→│  Fetch    │──→│  Pre-decode  │──→│  Decode    │  │
│  │Predictor │   │ (16B/cyc) │   │  (find inst  │   │ (1 complex │  │
│  │          │   │           │   │  boundaries) │   │  3 simple) │  │
│  └──────────┘   └───────────┘   └──────────────┘   └─────┬──────┘  │
│                                                          │         │
│                    ┌────────────────┐    ┌───────────────┘         │
│                    │   uop Cache    │←───┘                          │
│                    │ (decoded uops) │                               │
│                    └───────┬────────┘                               │
└────────────────────────────┼────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    RENAME / ALLOCATE                                 │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │  Register Alias Table (RAT)                                  │    │
│  │  - Maps 16 arch regs → hundreds of physical regs             │    │
│  │  - "Free" instructions: xor eax,eax → point to zero reg      │    │
│  │  - Move elimination: mov rax,rbx → point to same phys reg    │    │
│  └─────────────────────────────────────────────────────────────┘    │
└────────────────────────────┬────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────────┐
│                         BACKEND                                      │
│  ┌─────────────────┐      ┌─────────────────────────────────────┐   │
│  │  Reorder Buffer │      │  Scheduler (Reservation Stations)   │   │
│  │  (224 entries)  │      │  - Wait for operands ready          │   │
│  │  - Tracks order │      │  - Wait for port available          │   │
│  │  - Enables undo │      └──────────────┬──────────────────────┘   │
│  └─────────────────┘                     │                          │
│                                          ▼                          │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                    Execution Ports                            │   │
│  │  Port 0,1: Math (ALU, FP)    Port 2,3: Load    Port 4: Store │   │
│  └──────────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────────┐
│                        RETIREMENT                                    │
│  - Commit in original program order                                  │
│  - Write stores to L1 cache                                          │
│  - Free physical registers for reuse                                 │
└─────────────────────────────────────────────────────────────────────┘
```

---

## The Frontend: Fetch and Decode

The frontend's goal: **feed the backend with micro-operations (uops)**.

### Fetching
- Fetches code in **16-byte chunks**
- Guided by **branch predictor** - doesn't wait for execution
- Speculative: follows predicted path "blindly"

### Pre-decode
x86 instructions are **variable length** (1-15 bytes), so CPU must:
1. Identify instruction boundaries
2. Perform **Macro-fusion**: combine instructions like `cmp` + `jne` into single uop

### Decoding
x86 → micro-operations (uops):

| Decoder Type | Capability |
|--------------|------------|
| 1 Complex decoder | Up to 4 uops |
| 3 Simple decoders | 1 uop each |
| Microcode Sequencer (MSROM) | Complex ops like `idiv`, string ops |

### uop Cache
```
After decode, uops cached to avoid re-decode:
┌─────────────────────────────────────────┐
│            uop Cache                     │
│  - Stores decoded micro-operations       │
│  - If loop fits: power down fetch/decode │
│  - Stream directly from cache            │
└─────────────────────────────────────────┘
```

### Loop Stream Detector (LSD)
- Locks small loops directly into instruction queue
- Even more efficient than uop cache
- **Disabled on Skylake** via microcode patch due to hyperthreading bug (discovered by OCaml community)

---

## ROB Structure

```
┌──────────────────────────────────────────────────────────────┐
│                     REORDER BUFFER                            │
├────┬────────┬─────────┬───────────┬───────┬─────────┬────────┤
│ ID │  PC    │ Dest Reg│   Value   │ Ready │ Except  │ Branch │
├────┼────────┼─────────┼───────────┼───────┼─────────┼────────┤
│  0 │ 0x1000 │   R3    │    42     │   ✓   │    -    │   -    │ ← Head (commit ptr)
│  1 │ 0x1004 │   R5    │   ---     │   ✗   │    -    │   -    │
│  2 │ 0x1008 │   R7    │   128     │   ✓   │    -    │   -    │
│  3 │ 0x100C │   -     │    -      │   ✓   │    -    │  Taken │
│  4 │ 0x1010 │   R2    │   ---     │   ✗   │    -    │   -    │ ← Tail (alloc ptr)
└────┴────────┴─────────┴───────────┴───────┴─────────┴────────┘
```

### ROB Entry Fields

| Field | Description |
|-------|-------------|
| **Instruction Type** | Branch, store, ALU operation |
| **Destination** | Register number (or memory address for stores) |
| **Value** | Result once computed |
| **Ready** | Whether execution has completed |
| **Exception** | Exception status (page fault, overflow, etc.) |

## ROB Lifecycle

### 1. Dispatch (Allocation)
```
When instruction enters pipeline:
1. Allocate ROB entry at tail
2. ROB index becomes physical register name
3. Update register alias table (RAT): arch_reg → ROB_index
4. Increment tail pointer
```

### 2. Issue (to Execution Units)
```
When operands ready:
1. Read operands from:
   - Physical register file (if committed)
   - ROB entry (if in-flight, via bypass)
2. Send to execution unit
```

### 3. Completion (Writeback)
```
When execution finishes:
1. Write result to ROB entry
2. Mark entry as Ready
3. Broadcast result for dependent instructions (CDB)
```

### 4. Commit (Retirement)
```
When head entry is Ready:
1. Write result to architectural register file
2. Free ROB entry
3. Increment head pointer
4. Handle any exceptions
```

## Register Renaming with ROB

### Problem: False Dependencies
```asm
R1 = R2 + R3    ; (1)
R4 = R1 × R5    ; (2) True dependency on R1
R1 = R6 + R7    ; (3) WAW hazard with (1), WAR with (2)
R8 = R1 + R9    ; (4) True dependency on (3)
```

### Solution: ROB as Rename Register

```
Instruction    Renamed To
-----------    ----------
R1 = R2 + R3   ROB0 = R2 + R3
R4 = R1 × R5   ROB1 = ROB0 × R5
R1 = R6 + R7   ROB2 = R6 + R7      ; Different physical reg!
R8 = R1 + R9   ROB3 = ROB2 + R9
```

All false dependencies eliminated!

---

## Skylake Register Renaming Details

The CPU has **hundreds of internal physical registers** - far more than the 16 architectural registers visible to programmers (RAX, RBX, RCX, etc.).

### Register Alias Table (RAT)
```
┌─────────────────────────────────────────────────────────┐
│               Register Alias Table                       │
├──────────────┬──────────────────────────────────────────┤
│ Arch Reg     │ Physical Reg                             │
├──────────────┼──────────────────────────────────────────┤
│ RAX          │ → P47                                    │
│ RBX          │ → P23                                    │
│ RCX          │ → P156                                   │
│ ...          │ ...                                      │
└──────────────┴──────────────────────────────────────────┘

Multiple in-flight instructions can use "RAX" simultaneously
because they're actually using different physical registers!
```

### "Free" Instructions (Zero Execution Cost)

Skylake recognizes certain patterns and handles them **without using execution units**:

#### 1. Zeroing Idioms
```asm
xor eax, eax      ; Common way to zero a register
```
- CPU detects this pattern at rename stage
- **No execution needed** - simply points RAX entry in RAT to a dedicated "zero" physical register
- Result: 0 cycles, 0 port usage

#### 2. Move Elimination
```asm
mov rax, rbx      ; Copy RBX to RAX
```
- CPU handles by **pointing RAX to same physical register as RBX** in RAT
- No data movement occurs
- Result: 0 cycles, 0 port usage

```
Before mov rax, rbx:           After (with move elimination):
┌────────┬────────┐            ┌────────┬────────┐
│  RAX   │  P47   │            │  RAX   │  P23   │ ←─┐
│  RBX   │  P23   │            │  RBX   │  P23   │ ──┘ Same!
└────────┴────────┘            └────────┴────────┘
```

### Implications for Optimization
```asm
; This is essentially FREE:
xor eax, eax              ; Zero (free)
mov rcx, rax              ; Move eliminated (free)

; But this costs cycles:
mov eax, 0                ; Requires execution (uses immediate)
sub eax, eax              ; May not be recognized as zeroing idiom
```

## ROB vs Physical Register File

### ROB-based Renaming (Classic Tomasulo + ROB)

```
┌─────────────┐    ┌─────────────┐
│    RAT      │    │     ROB     │
│ R1 → ROB2   │───→│  Value: 42  │
│ R2 → ROB5   │    │  Ready: ✓   │
└─────────────┘    └─────────────┘
```

**Pros:**
- Simple allocation (just increment tail)
- Natural ordering for commit

**Cons:**
- Values duplicated (ROB and arch file)
- Limited by ROB size

### Merged Register File (Modern CPUs)

```
┌─────────────┐    ┌─────────────────────┐
│    RAT      │    │ Physical Reg File   │
│ R1 → P42    │───→│  P42: value         │
│ R2 → P17    │    │  (includes ROB-like │
└─────────────┘    │   metadata)         │
                   └─────────────────────┘
```

**Used in:** Most modern x86 processors (Zen, Golden Cove)

## Commit Strategies

### Single Commit
- One instruction per cycle
- Simple but limits throughput

### Multi-Commit (Superscalar)
```
Modern CPUs: 4-8 instructions/cycle commit
- Check N head entries simultaneously
- All must be Ready and no exceptions
- Enables high IPC
```

### Clustered Commit
- Separate commit logic per cluster
- Merges at final stage

## Exception Handling

### Precise Exceptions Requirement
When exception at instruction N:
1. Instructions 0..N-1: **committed** (effects visible)
2. Instruction N: **faulting** (reported to OS)
3. Instructions N+1..: **squashed** (effects discarded)

### ROB Enables This
```
Exception at ROB entry 5:
1. Commit entries 0-4 normally
2. At entry 5: stop, signal exception
3. Flush entries 6+ from ROB
4. Flush pipeline
5. Redirect to exception handler
```

## Branch Misprediction Recovery

### Checkpoint-based Recovery
```
At each branch:
1. Save RAT state (checkpoint)
2. Continue speculative execution

On misprediction:
1. Restore RAT from checkpoint
2. Flush ROB entries after branch
3. Redirect fetch to correct path
```

### Walk-based Recovery
```
On misprediction:
1. Walk backwards through ROB
2. Undo each rename mapping
3. Slower but less hardware
```

## ROB Sizing Considerations

### Too Small
- Limits instruction window
- Reduces ILP extraction
- Frequent stalls on structural hazard

### Too Large
- Increased access latency
- Higher power consumption
- Diminishing returns on ILP

### Modern Sizes
| Processor | ROB Size |
|-----------|----------|
| Intel Skylake | 224 entries |
| Intel Golden Cove | 512 entries |
| AMD Zen 4 | 320 entries |
| Apple M1 (Firestorm) | ~630 entries |
| RISC-V BOOM | 64-128 entries |

## Scheduler (Reservation Stations)

The scheduler is where uops wait until ready for execution.

### Scheduling Criteria
A uop can execute when:
1. **All input operands are ready** (produced by earlier instructions)
2. **Required execution port is available**

### Skylake Execution Ports

```
┌─────────────────────────────────────────────────────────────────┐
│                     Execution Ports                              │
├─────────┬───────────────────────────────────────────────────────┤
│ Port 0  │ ALU, FMA, FP Mul, Branch                              │
│ Port 1  │ ALU, FMA, FP Add, Int Mul                             │
│ Port 2  │ Load Address Generation                                │
│ Port 3  │ Load Address Generation                                │
│ Port 4  │ Store Data                                             │
│ Port 5  │ ALU, Shuffle, Vector                                   │
│ Port 6  │ ALU, Branch                                            │
│ Port 7  │ Store Address Generation                               │
└─────────┴───────────────────────────────────────────────────────┘
```

### Micro-fusion Split

Instructions fused earlier (load + operation) get **split here**:

```asm
add rax, [rbx]    ; Fused in frontend as single uop

; Split in backend:
; → Load unit (Port 2/3): load from [rbx]
; → ALU (Port 0/1/5/6): add to rax
```

This allows load and compute to go to appropriate ports.

---

## ROB Interaction with Memory

### Store Buffer Integration
```
Stores in ROB:
1. Address computed → store buffer
2. Data available → store buffer
3. Commit → write to cache

Loads can bypass from store buffer!
```

### Memory Disambiguation
```
Load at ROB entry 10:
1. Check all stores at entries 0-9
2. If address matches: forward data
3. If unknown address: wait or speculate
```

---

## Skylake Memory Subsystem

### Store Buffer

Stores **cannot write to real memory** until the instruction is **retired** (confirmed valid, not speculative):

```
┌─────────────────────────────────────────────────────────────────┐
│                      Store Buffer                                │
│  ┌─────────┬─────────────┬────────────┬─────────────┐           │
│  │  Entry  │   Address   │    Data    │   Status    │           │
│  ├─────────┼─────────────┼────────────┼─────────────┤           │
│  │    0    │  0x7fff100  │  0xDEAD    │  Pending    │           │
│  │    1    │  0x7fff108  │  0xBEEF    │  Pending    │           │
│  │    2    │  0x7fff110  │  0xCAFE    │  Committed  │ → L1 Cache│
│  └─────────┴─────────────┴────────────┴─────────────┘           │
└─────────────────────────────────────────────────────────────────┘
```

### Store-to-Load Forwarding

If a load tries to read from an address **pending in the store buffer**, the CPU "short circuits":

```
mov [rax], rbx       ; Store to address in rax (goes to store buffer)
mov rcx, [rax]       ; Load from same address

; CPU detects address match in store buffer
; Forwards data directly WITHOUT going to L1 cache
; Saves ~4 cycles of L1 latency
```

```
┌────────────────────────────────────────────────────────────┐
│                Store-to-Load Forwarding                     │
│                                                             │
│   Store Buffer          Load Unit                           │
│   ┌──────────┐         ┌──────────┐                        │
│   │ addr: X  │────────→│ want: X  │  Match! Forward data   │
│   │ data: 42 │─────┐   └──────────┘                        │
│   └──────────┘     │                                       │
│                    │   ┌──────────┐                        │
│                    └──→│ data: 42 │  Bypass L1 cache       │
│                        └──────────┘                        │
└────────────────────────────────────────────────────────────┘
```

### When Forwarding Fails

Store-to-load forwarding can fail if:
- Partial overlap (store 8 bytes, load 4 from middle)
- Misaligned access
- Store address not yet computed

Failed forwarding = **pipeline stall** waiting for store to commit

## ROB in BOOM (RISC-V)

BOOM (Berkeley Out-of-Order Machine) implements ROB with:

```
- Unified physical register file
- Separate free list management
- Branch mask for speculation tracking
- Integration with RoCC accelerator interface
```

### BOOM ROB Configuration
```scala
// From BOOM configuration
val numRobEntries = 64  // or 128 for larger configs
val numPhysRegisters = numRobEntries + 32
```

## ROB and Accelerators

When integrating accelerators (like vector units):

### Challenges
1. Long-latency operations block commit
2. Resource management across domains
3. Exception handling across boundaries

### Solutions
1. **Decoupled commit** - accelerator has own ordering
2. **Token-based tracking** - ROB tracks completion tokens
3. **Coarse-grain checkpoints** - less precise but simpler

## Performance Metrics

### Key Indicators
- **ROB Full Stalls**: Pipeline blocked waiting for commit
- **Average ROB Occupancy**: How much ILP being exploited
- **Commit Rate**: IPC at retirement stage

### Optimization Targets
```
IPC = min(fetch_rate, issue_rate, commit_rate)

Often commit-bound when:
- Long-latency ops at head (cache miss, div)
- Frequent mispredictions
- Exception-heavy code
```

---

## Retirement (Commit Stage)

The final stage where CPU **commits work to architectural state**.

### Retirement Rules

1. Instructions retire **in exact program order**
2. Stores are written to L1 cache (from store buffer)
3. Physical registers no longer needed are **freed for reuse**
4. Exceptions are raised precisely at the faulting instruction

### Why In-Order Retirement Matters

```
; Instructions execute out-of-order:
;   Instruction 3 finishes first
;   Instruction 1 finishes second
;   Instruction 2 still executing...

; But retire in order:
;   Wait for instruction 1 to complete → retire
;   Wait for instruction 2 to complete → retire
;   Instruction 3 already done → retire immediately
```

This ensures that if instruction 2 causes an exception:
- Instruction 1's effects are visible
- Instructions 3+ are discarded (even though 3 "finished")

---

## Why Is This Undocumented?

From Matt Godbolt's talk - Intel doesn't fully document internal microarchitecture because:

### 1. IP Protection
- Competitive advantage in implementation details
- Trade secrets in specific optimizations

### 2. Hyrum's Law
> "With a sufficient number of users, every observable behavior becomes a dependency"

If Intel documents that `xor eax, eax` takes 0 cycles:
- Software will depend on this
- Future chips must maintain it forever
- **Limits future innovation**

### 3. Flexibility for Future Designs
- Undocumented behaviors can change between chip generations
- Allows microarchitecture improvements without breaking compatibility

### How This Gets Reverse Engineered

| Method | Description |
|--------|-------------|
| **Performance Counters** | Leak info about stalls, port usage, cache behavior |
| **Micro-benchmarks** | Careful tests to probe specific behaviors |
| **Timing Attacks** | Measure execution time variations |
| **Researchers** | Agner Fog, Travis Downs, etc. publish findings |

```c
// Example: Detecting move elimination
for (int i = 0; i < 1000000; i++) {
    asm volatile(
        "mov %%rax, %%rbx\n"
        "mov %%rbx, %%rcx\n"
        "mov %%rcx, %%rdx\n"
        : : : "rax", "rbx", "rcx", "rdx"
    );
}
// If this runs at 0 cycles/iteration → move elimination confirmed
```

---

## Summary

The ROB is the cornerstone of modern out-of-order execution:

1. **Maintains program order** for precise state
2. **Enables speculation** with safe recovery
3. **Supports renaming** to eliminate false deps
4. **Manages commit** for architectural state updates

Without ROB, we cannot have both high ILP and precise exceptions.

### Key Skylake Insights (Godbolt)

| Component | Role |
|-----------|------|
| **Frontend** | Fetch → Decode → uop Cache |
| **RAT** | Maps 16 arch regs to hundreds of physical regs |
| **Free Ops** | `xor eax,eax`, `mov` handled without execution |
| **ROB** | Tracks 224 in-flight uops in program order |
| **Scheduler** | Dispatches to 8 execution ports |
| **Store Buffer** | Holds stores until retirement |
| **Retirement** | In-order commit, frees resources |

## References

1. Hennessy & Patterson, "Computer Architecture: A Quantitative Approach"
2. BOOM Documentation - https://github.com/riscv-boom/riscv-boom
3. Intel Optimization Manual - ROB and Backend Architecture
4. [Matt Godbolt - "What Has My Compiler Done for Me Lately?"](https://www.youtube.com/watch?v=BVVNtG5dgks)
5. [Agner Fog's Microarchitecture Guide](https://www.agner.org/optimize/microarchitecture.pdf)
