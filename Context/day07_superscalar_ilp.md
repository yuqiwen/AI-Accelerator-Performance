# Day 07 — Superscalar and Instruction-Level Parallelism

## Learning Goals

- Explain superscalar execution.
- Distinguish pipelining from multiple-issue execution.
- Define instruction-level parallelism (ILP).
- Understand how dependencies limit ILP.
- Explain why multiple accumulators and loop unrolling help.
- Understand why a wide CPU does not always achieve high IPC.

## 1. Pipeline vs. Superscalar

A pipeline overlaps stages of different instructions.

A scalar pipelined CPU may still issue only one instruction per cycle.

A superscalar CPU can issue multiple instructions in one cycle when:

```text
instructions are independent
operands are ready
execution units are available
```

Example:

```text
Cycle 1:
issue add
issue multiply
issue load
```

## 2. Processor Width

A processor may be described as:

```text
2-wide
4-wide
6-wide
```

A 4-wide CPU may decode or issue up to four instructions per cycle.

But:

```text
4-wide does not guarantee IPC = 4
```

Actual IPC depends on dependencies, branches, caches, execution ports, and front-end bandwidth.

## 3. Instruction-Level Parallelism

Independent instructions:

```asm
add r1, r2, r3
mul r4, r5, r6
sub r7, r8, r9
```

These may execute concurrently.

Dependent instructions:

```asm
add r1, r2, r3
mul r4, r1, r5
sub r6, r4, r7
```

This forms:

```text
add → mul → sub
```

The dependency chain limits ILP.

## 4. Execution Units

A CPU core may contain:

```text
integer ALUs
floating-point units
vector units
load units
store units
branch units
```

Independent instructions still compete when they need the same execution port.

## 5. Latency vs. Throughput

Suppose FP add has:

```text
latency = 4 cycles
throughput = 2 independent adds/cycle
```

One dependent add chain advances according to latency.

Many independent adds can approach execution throughput.

```text
dependency chain → latency-limited
independent stream → throughput-limited
```

## 6. Single Accumulator

```cpp
float sum = 0.0f;

for (int i = 0; i < n; ++i) {
    sum += a[i];
}
```

Every iteration depends on the previous value of `sum`.

```text
sum[i + 1] depends on sum[i]
```

This creates one long dependency chain.

## 7. Multiple Accumulators

```cpp
float s0 = 0.0f;
float s1 = 0.0f;
float s2 = 0.0f;
float s3 = 0.0f;

for (int i = 0; i < n; i += 4) {
    s0 += a[i];
    s1 += a[i + 1];
    s2 += a[i + 2];
    s3 += a[i + 3];
}

float sum = s0 + s1 + s2 + s3;
```

Now there are four independent chains.

The CPU can overlap them and hide add latency.

## 8. Loop Unrolling

Original:

```cpp
for (int i = 0; i < n; ++i) {
    y[i] = a[i] + b[i];
}
```

Unrolled:

```cpp
for (int i = 0; i < n; i += 4) {
    y[i]     = a[i]     + b[i];
    y[i + 1] = a[i + 1] + b[i + 1];
    y[i + 2] = a[i + 2] + b[i + 2];
    y[i + 3] = a[i + 3] + b[i + 3];
}
```

Benefits:

```text
less loop overhead
more visible ILP
better scheduling
possible vectorization
```

Costs:

```text
larger code
more register pressure
instruction-cache pressure
```

## 9. Front-End Limits

Before execution, instructions must be:

```text
fetched
decoded
renamed
dispatched
```

Possible front-end bottlenecks:

```text
instruction-cache misses
decode bandwidth
branch misprediction
micro-op cache misses
```

A wide back end is useless if the front end cannot feed it.

## 10. Back-End Limits

Possible back-end bottlenecks:

```text
execution-port contention
dependency chains
load/store queue pressure
cache misses
insufficient independent work
```

## 11. Branches and ILP

A branch misprediction discards speculative work and flushes the pipeline.

Wide CPUs can lose many in-flight instructions at once.

This is why branch prediction is crucial for superscalar CPUs.

## 12. Memory Dependencies

Independent loads:

```cpp
x = a[i];
y = b[j];
```

Dependent loads:

```cpp
p = node->next;
value = p->data;
```

The second address depends on the first load result.

This limits both ILP and memory-level parallelism.

## 13. ILP vs. SIMD

ILP:

```text
different independent instructions execute in parallel
```

SIMD:

```text
one vector instruction processes multiple data elements
```

Modern CPUs can use both simultaneously.

## 14. ILP vs. Thread-Level Parallelism

```text
ILP:
parallelism inside one thread

Thread-level parallelism:
multiple threads or cores
```

SMT can use another hardware thread when one thread lacks enough ILP.

## 15. Why IPC Is Below Width

A 4-wide CPU may still achieve IPC below 4 because of:

```text
dependencies
cache misses
branch mispredictions
port conflicts
front-end starvation
serialization
```

Width is a maximum capability, not a guarantee.

## 16. GPU Connection

GPUs also benefit from ILP:

```cpp
acc0 += a0 * b0;
acc1 += a1 * b1;
acc2 += a2 * b2;
acc3 += a3 * b3;
```

The accumulators are independent.

GPU latency hiding uses:

```text
ILP within a thread
TLP across warps
```

More ILP may reduce the number of warps needed, but it can increase register pressure.

## 17. Floating-Point Reordering

Multiple accumulators change addition order.

Because floating-point addition is not exactly associative:

```text
(a + b) + c may differ from a + (b + c)
```

The result may differ slightly.

This is a correctness-performance tradeoff.

## 18. Interview Takeaway

> Superscalar processors issue multiple independent instructions per cycle using multiple execution units. Actual IPC depends on ILP, operand readiness, branch prediction, front-end bandwidth, memory behavior, and execution-port availability. Long dependency chains limit ILP, so loop unrolling and multiple accumulators can improve performance by exposing independent operations.

## 19. Key Summary

```text
Pipelining:
overlap stages

Superscalar:
issue multiple instructions per cycle

ILP:
independent instructions inside one thread
```

```text
dependent chain → latency-limited
independent operations → throughput-limited
```

## Exercises

### Exercise 1

Which has more ILP?

```asm
add r1, r2, r3
mul r4, r1, r5
sub r6, r4, r7
```

or:

```asm
add r1, r2, r3
mul r4, r5, r6
sub r7, r8, r9
```

### Exercise 2

Does a 4-wide CPU guarantee IPC = 4?

### Exercise 3

Why do multiple accumulators improve reduction?

### Exercise 4

How can loop unrolling hurt performance?

## Answers

1. The second sequence has more ILP because the instructions are independent.
2. No. Dependencies, branches, cache misses, and execution-port conflicts can lower IPC.
3. They create independent dependency chains and hide instruction latency.
4. It can increase code size, register pressure, and instruction-cache pressure.
