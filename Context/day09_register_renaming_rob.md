# Day 09 — Register Renaming and Reorder Buffer

## Learning Goals
- Explain architectural vs. physical registers.
- Distinguish RAW, WAR, and WAW dependencies.
- Explain how register renaming removes false dependencies.
- Explain how the reorder buffer enables out-of-order execution with in-order retirement.
- Understand precise exceptions and branch recovery.

## 1. Architectural Registers Are Names

The ISA exposes a limited set of architectural registers such as:

```text
r0, r1, r2, ...
```

The same name can represent different logical values at different times.

```asm
1: add r1, r2, r3
2: mul r4, r1, r5
3: sub r1, r6, r7
4: add r8, r1, r9
```

Instruction 1 produces one version of `r1`; instruction 3 produces a newer version.

## 2. Dependency Types

### RAW — Read After Write

```asm
add r1, r2, r3
mul r4, r1, r5
```

The multiply truly needs the add result. This is a true dependency and cannot be renamed away.

### WAR — Write After Read

```asm
add r4, r1, r2
sub r1, r5, r6
```

The first instruction must read the old `r1` before the second overwrites the name. This is a false name dependency.

### WAW — Write After Write

```asm
add r1, r2, r3
sub r1, r4, r5
```

Both write the same architectural name. This is also a false name dependency.

## 3. Register Renaming

Modern CPUs have more physical registers than architectural registers.

```text
architectural: r0, r1, r2, ...
physical:      p0, p1, p2, ...
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

The false conflict between instructions 1 and 3 disappears.

## 4. Rename Table

The CPU tracks:

```text
architectural register → latest physical register
```

When a new instruction writes `r1`, a fresh physical register is allocated and the mapping is updated.

Older instructions still refer to the old physical register.

## 5. What Renaming Removes

```text
RAW → preserved
WAR → removed
WAW → removed
```

Renaming increases instruction-level parallelism by eliminating false name conflicts.

## 6. Reorder Buffer

The reorder buffer, or ROB, tracks in-flight instructions in program order.

A simplified ROB entry may contain:

```text
completion state
destination information
exception status
branch status
```

Instructions may execute and finish out of order, but retirement occurs from the ROB head in order.

## 7. Execute, Complete, Retire

```text
Execute:
sent to an execution unit

Complete:
result has been produced

Retire:
result becomes architecturally visible
```

A younger instruction may complete before an older one but still cannot retire first.

## 8. Precise Exceptions

```asm
1: load r1, [invalid_address]
2: mul r4, r5, r6
3: add r7, r8, r9
```

Instructions 2 and 3 may execute early.

If instruction 1 faults, the machine must appear as if:

```text
older instructions completed
instruction 1 did not complete
younger instructions did not complete
```

Because younger results have not retired, they can be discarded.

## 9. Branch Recovery

After a branch misprediction:

```text
wrong-path instructions are discarded
rename mappings are restored
wrong-path physical registers are reclaimed
```

The ROB and rename checkpoints help restore the correct state.

## 10. Finite Resources

The ROB and physical register file are finite.

If the ROB fills:

```text
no more instructions can enter the out-of-order window
rename/dispatch stalls
the front end eventually stalls
```

If free physical registers run out:

```text
renaming also stalls
```

## 11. Step-by-Step Example

```asm
1: add r1, r2, r3
2: mul r4, r1, r5
3: sub r1, r6, r7
4: add r8, r1, r9
```

Rename:

```text
1 writes p10 for r1
2 reads p10
3 writes p11 for new r1
4 reads p11
```

Execution may be out of order, but retirement remains:

```text
1 → 2 → 3 → 4
```

## 12. CPU vs. GPU Registers

CPU physical registers:

```text
support renaming, speculation, and out-of-order execution
```

GPU registers:

```text
store per-thread values for resident threads
```

Both are fast on-chip storage, but they serve different roles.

## 13. Interview Takeaway

> Register renaming maps architectural registers to a larger set of physical registers, removing false WAR and WAW dependencies while preserving true RAW dependencies. The reorder buffer allows instructions to execute and complete out of order but retire in order, preserving precise exceptions and recoverable architectural state.

## 14. Key Summary

```text
Rename table:
architectural name → physical register

ROB:
tracks instructions in program order

Retirement:
makes results architecturally visible
```

## Exercises

1. Which dependency is `add r1,...` followed by `mul ...,r1,...`?
2. Which dependency types can renaming remove?
3. Why can a completed younger instruction not retire before an older faulting instruction?
4. What happens if the ROB becomes full?

## Answers

1. RAW.
2. WAR and WAW.
3. It would violate precise exceptions and expose speculative state too early.
4. Rename/dispatch stalls, and eventually the front end stops accepting more work.
