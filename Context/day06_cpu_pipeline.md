# Day 06 — CPU Pipeline

## Learning Goals

- Explain why CPUs use pipelines.
- Describe common pipeline stages.
- Distinguish instruction latency from instruction throughput.
- Understand structural, data, and control hazards.
- Explain forwarding, stalls, bubbles, and pipeline flushes.
- Connect pipelining to superscalar and out-of-order execution.

## 1. Why Pipelines Exist

A CPU instruction usually passes through several steps:

```text
fetch
decode
read operands
execute
access memory
write back
```

Without pipelining, the CPU would wait for one instruction to finish before starting the next.

With pipelining, different instructions occupy different stages at the same time:

```text
instruction 1 executes
instruction 2 decodes
instruction 3 fetches
```

The main benefit is higher throughput.

## 2. Classic Five-Stage Pipeline

```text
IF  → Instruction Fetch
ID  → Decode / Register Read
EX  → Execute / Address Calculation
MEM → Data Memory Access
WB  → Write Back
```

Real CPUs use deeper and more complex pipelines, but this model is useful.

## 3. Overlapping Instructions

```text
Cycle 1: I1 IF
Cycle 2: I1 ID | I2 IF
Cycle 3: I1 EX | I2 ID | I3 IF
Cycle 4: I1 MEM| I2 EX | I3 ID | I4 IF
Cycle 5: I1 WB | I2 MEM| I3 EX | I4 ID | I5 IF
```

After the pipeline fills, an ideal scalar pipeline can complete about one instruction per cycle.

## 4. Latency vs. Throughput

A five-stage pipeline may still give one instruction a latency of roughly five cycles.

After filling, however:

```text
throughput ≈ 1 instruction/cycle
```

Pipelining mainly improves throughput, not necessarily single-instruction latency.

## 5. Fill and Drain

For `S` stages and `N` independent instructions:

```text
ideal pipelined cycles = S + N - 1
```

Without overlap:

```text
non-pipelined cycles = S × N
```

For large `N`, the average cost approaches one cycle per instruction.

## 6. Pipeline Hazards

Three common hazards:

```text
structural hazards
data hazards
control hazards
```

## 7. Structural Hazards

A structural hazard occurs when two instructions need the same hardware resource simultaneously.

Example:

```text
instruction fetch needs memory
load/store also needs memory
```

Possible solutions:

```text
duplicate resources
add more ports
stall one instruction
```

## 8. Data Hazards

Example:

```asm
add r1, r2, r3
sub r4, r1, r5
```

The second instruction needs `r1` before the first has finished producing it.

This is:

```text
Read After Write (RAW)
```

RAW is a true dependency.

## 9. Forwarding

The CPU may forward an execution result directly to a dependent instruction:

```text
EX result → next EX input
```

instead of waiting for:

```text
EX → WB → register file → next EX
```

This is called forwarding or bypassing.

## 10. Load-Use Hazard

```asm
load r1, [address]
add  r2, r1, r3
```

The loaded value may not be ready until after the memory stage.

The following `add` may need to stall even with forwarding.

This creates a bubble.

## 11. Stall and Bubble

A stall prevents part of the pipeline from advancing.

A bubble is an empty slot moving through the pipeline:

```text
useful instruction
bubble
useful instruction
```

Bubbles reduce throughput.

## 12. Control Hazards

For a branch:

```cpp
if (x > 0) {
    y = a;
} else {
    y = b;
}
```

the CPU may not immediately know the next instruction address.

Modern CPUs predict the branch and continue speculatively.

## 13. Branch Misprediction

If the prediction is wrong:

```text
wrong-path instructions are discarded
the pipeline is flushed
fetch restarts from the correct path
```

Deeper pipelines usually have larger branch-misprediction penalties because more in-flight work must be discarded.

## 14. Pipeline Depth Tradeoff

Deeper pipelines may allow higher clock frequency because each stage does less work.

Costs include:

```text
larger branch penalties
more pipeline registers
more forwarding complexity
more scheduling complexity
```

Deeper is not automatically better.

## 15. CPI and IPC

```text
CPI = Cycles Per Instruction
IPC = Instructions Per Cycle
```

An ideal scalar pipeline may approach:

```text
CPI ≈ 1
IPC ≈ 1
```

Stalls increase CPI.

A superscalar processor can achieve:

```text
IPC > 1
```

by issuing multiple instructions per cycle.

## 16. Pipeline vs. Multicore

```text
pipelining:
overlap stages of different instructions inside a core

multicore:
different cores execute different instruction streams
```

These are separate forms of parallelism.

## 17. Connection to Out-of-Order Execution

```asm
load r1, [slow_address]
add  r2, r1, r3
mul  r4, r5, r6
```

The `mul` is independent.

An in-order processor may be blocked behind the dependent `add`.

An out-of-order processor may execute `mul` while waiting for the load.

```text
pipeline → overlaps instruction stages
out-of-order → finds independent work to keep stages busy
```

## 18. CPU vs. GPU Latency Hiding

CPUs invest heavily in:

```text
branch prediction
out-of-order execution
large caches
speculation
```

GPUs rely heavily on:

```text
many resident warps
warp scheduling
```

Both aim to keep execution units busy.

## 19. C++ Reduction Example

```cpp
for (int i = 0; i < n; ++i) {
    sum += a[i];
}
```

This creates one dependency chain:

```text
sum_next depends on sum_previous
```

Multiple accumulators create independent chains:

```cpp
float s0 = 0, s1 = 0, s2 = 0, s3 = 0;

for (int i = 0; i < n; i += 4) {
    s0 += a[i];
    s1 += a[i + 1];
    s2 += a[i + 2];
    s3 += a[i + 3];
}

float sum = s0 + s1 + s2 + s3;
```

This exposes more instruction-level parallelism and improves pipeline utilization.

## 20. Interview Takeaway

> A CPU pipeline divides instruction execution into stages such as fetch, decode, execute, memory access, and write-back. Multiple instructions overlap across these stages, improving throughput even though one instruction may still take several cycles. Structural conflicts, data dependencies, and branches introduce stalls or flushes. Forwarding, branch prediction, and out-of-order execution help keep the pipeline busy.

## 21. Key Summary

```text
Pipelining:
overlap stages of multiple instructions

Main benefit:
higher instruction throughput

Main hazards:
structural
data
control
```

```text
forwarding reduces dependency stalls
branch prediction reduces control stalls
out-of-order execution finds independent work
```

## Exercises

### Exercise 1

A five-stage ideal pipeline executes 100 independent instructions. Estimate total cycles.

### Exercise 2

Classify the hazard:

```asm
add r1, r2, r3
sub r4, r1, r5
```

### Exercise 3

Why does a branch misprediction hurt more in a deeper pipeline?

### Exercise 4

Why can multiple accumulators accelerate a CPU reduction loop?

## Answers

### Exercise 1

```text
5 + 100 - 1 = 104 cycles
```

### Exercise 2

RAW data hazard.

### Exercise 3

More wrong-path instructions are in flight and must be flushed.

### Exercise 4

They create independent dependency chains, exposing more instruction-level parallelism.
