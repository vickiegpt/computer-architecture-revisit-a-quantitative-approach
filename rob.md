# ROB in Detail

## What is a Reorder Buffer?

The **Reorder Buffer (ROB)** is a hardware structure that enables out-of-order execution while maintaining the illusion of in-order completion. It's the key mechanism for:

1. **Precise exceptions** - handling interrupts at exact program points
2. **Branch misprediction recovery** - rolling back speculative state
3. **Register renaming** - eliminating false dependencies (WAR, WAW)

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

## Summary

The ROB is the cornerstone of modern out-of-order execution:

1. **Maintains program order** for precise state
2. **Enables speculation** with safe recovery
3. **Supports renaming** to eliminate false deps
4. **Manages commit** for architectural state updates

Without ROB, we cannot have both high ILP and precise exceptions.

## References

1. Hennessy & Patterson, "Computer Architecture: A Quantitative Approach"
2. BOOM Documentation - https://github.com/riscv-boom/riscv-boom
3. Intel Optimization Manual - ROB and Backend Architecture
