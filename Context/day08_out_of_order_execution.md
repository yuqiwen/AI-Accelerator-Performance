# Day 08 — Out-of-Order Execution

## Learning Goals

- Explain why in-order execution wastes CPU resources.
- Distinguish program order, execution order, and retirement order.
- Understand instruction readiness, reservation stations, the reorder buffer, and register renaming.
- Distinguish true dependencies from false dependencies.
- Connect out-of-order execution to CPU latency hiding.

## 1. Core Problem

```asm
load r1, [slow_address]
add  r2, r1, r3
mul  r4, r5, r6
```

`add` depends on `load`, but `mul` is independent.

An in-order CPU may wait:

```text
load waits
add waits
mul waits behind add
```

An out-of-order CPU can do:

```text
load starts
add waits
mul executes early
```

## 2. Three Orders

```text
Program order:
the order written by the program

Execution order:
the dynamic order chosen by the CPU

Retirement order:
the original program order
```

Modern CPUs commonly:

```text
execute out of order
retire in order
```

## 3. Why Retire In Order

Suppose:

```asm
1: load r1, [bad_address]
2: mul  r4, r5, r6
```

The multiply may execute first internally.

But if the older load faults, the multiply must not become architecturally visible.

In-order retirement preserves:

```text
precise exceptions
correct register state
program semantics
```

## 4. Instruction Readiness

An instruction is ready when:

```text
all operands are ready
the required execution unit is available
```

Example:

```asm
add r1, r2, r3
mul r4, r5, r6
sub r7, r1, r8
```

Initially:

```text
add ready
mul ready
sub waits for r1
```

## 5. Reservation Stations / Issue Queue

Decoded instructions wait in structures that track:

```text
operation type
operand readiness
source values or producer tags
destination
```

The scheduler selects ready instructions and sends them to execution units.

## 6. Reorder Buffer

The reorder buffer, or ROB, tracks in-flight instructions in program order.

It stores information such as:

```text
completion state
destination
exception state
branch state
```

Instructions may finish out of order, but only completed safe instructions at the ROB head retire.

## 7. Register Renaming

The ISA exposes architectural registers:

```text
r1, r2, r3
```

Internally the CPU has more physical registers:

```text
p0, p1, p2, ...
```

Example:

```asm
add r1, r2, r3
mul r1, r4, r5
```

The two writes to architectural `r1` can map to different physical registers:

```text
first r1 → p10
second r1 → p18
```

## 8. Dependency Types

### RAW: Read After Write

```asm
add r1, r2, r3
mul r4, r1, r5
```

This is a true dependency. Renaming cannot remove it.

### WAR: Write After Read

A later instruction writes a register before an earlier instruction has read the old value.

This is a false name dependency and can be removed by renaming.

### WAW: Write After Write

Two instructions write the same architectural register.

This is also a false name dependency and can be removed by renaming.

## 9. Example Mapping

```asm
1: add r1, r2, r3
2: mul r4, r1, r5
3: sub r1, r6, r7
4: add r8, r1, r9
```

Possible mapping:

```text
instruction 1 writes p10 for r1
instruction 2 reads p10
instruction 3 writes p11 for new r1
instruction 4 reads p11
```

True dependencies remain:

```text
1 → 2
3 → 4
```

The false conflict between instructions 1 and 3 is removed.

## 10. Load Miss Example

```asm
load r1, [A]
add  r2, r1, r3
mul  r4, r5, r6
add  r7, r8, r9
```

If the load misses:

```text
load waits
dependent add waits
mul executes
independent add executes
```

This is CPU latency hiding using ILP from the same thread.

## 11. Limits

Out-of-order execution cannot solve:

```text
long true dependency chains
too-small instruction windows
all later instructions depending on one cache miss
execution-port saturation
branch mispredictions
```

## 12. Instruction Window

The number of in-flight instructions is limited by:

```text
ROB size
issue queue size
physical registers
load/store queue size
```

A larger window can find more distant independent work, but costs area, power, and complexity.

## 13. Memory Ordering

For:

```asm
store [p], x
load  y, [q]
```

the CPU must determine whether `p` and `q` alias.

It uses:

```text
load queue
store queue
memory disambiguation
store-to-load forwarding
```

to execute memory operations aggressively while preserving correctness.

## 14. Speculation

Out-of-order execution works with speculation.

The CPU may predict:

```text
a branch direction
that a load does not conflict with an older store
```

If wrong, speculative work is discarded and state is recovered using structures such as the ROB.

## 15. CPU vs. GPU

CPU:

```text
look ahead in one thread
find independent instructions
execute them early
```

GPU:

```text
when one warp stalls
schedule another ready warp
```

```text
CPU → instruction-level latency hiding
GPU → warp/thread-level latency hiding
```

## 16. Precise Exceptions

The machine must appear as if:

```text
all older instructions completed
the faulting instruction did not complete
no younger instruction completed
```

Younger instructions may have executed internally, but their results have not retired.

## 17. Interview Takeaway

> Out-of-order execution allows a CPU to execute ready instructions before older stalled instructions while respecting dependencies. Instructions execute out of order but retire in order through the reorder buffer. Reservation stations track operand readiness, and register renaming removes false WAR and WAW dependencies. This lets CPUs hide latency using instruction-level parallelism within one thread.

## 18. Key Summary

```text
reservation station / issue queue:
find ready instructions

register renaming:
remove false name dependencies

reorder buffer:
retire in order and recover safely
```

## Exercises

1. In `load → dependent add → independent mul`, which instruction can run while the load waits?
2. Why can a younger multiply not retire before an older faulting load?
3. Which dependencies can renaming remove: RAW, WAR, WAW?
4. Why is linked-list traversal still slow on an out-of-order CPU?

## Answers

1. The independent multiply.
2. It would break precise architectural state if the older load faults.
3. WAR and WAW, not RAW.
4. Each next address depends on the previous load result, leaving little independent work.
