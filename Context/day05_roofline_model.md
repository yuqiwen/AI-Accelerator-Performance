# Day 05 — Roofline Model

## Learning Goals

- Explain the Roofline Model.
- Interpret arithmetic intensity and achieved performance.
- Calculate the ridge point.
- Distinguish memory-bound and compute-bound regions.
- Understand whether an optimization moves a kernel right or up.

## 1. Core Equation

```text
Attainable Performance
=
min(
    Peak Compute,
    Arithmetic Intensity × Memory Bandwidth
)
```

Units:

```text
FLOPs/byte × bytes/second = FLOPs/second
```

## 2. Axes

```text
x-axis: Arithmetic Intensity, FLOPs/byte
y-axis: Performance, FLOPs/second
```

Low arithmetic intensity appears on the left. High arithmetic intensity appears on the right.

## 3. Memory Roof

At low arithmetic intensity:

```text
Performance = AI × Memory Bandwidth
```

This is the sloped part of the Roofline.

Example:

```text
Bandwidth = 1 TB/s
AI = 2 FLOPs/byte
Performance limit = 2 TFLOP/s
```

## 4. Compute Roof

The processor has a maximum arithmetic throughput:

```text
Peak Compute = 100 TFLOP/s
```

No kernel can exceed this compute ceiling. This is the flat part of the Roofline.

## 5. Ridge Point

The ridge point is:

```text
Peak Compute / Memory Bandwidth
```

Example:

```text
100 TFLOP/s / 2 TB/s = 50 FLOPs/byte
```

Interpretation:

```text
AI < 50 → memory-bound region
AI > 50 → compute-bound region
```

## 6. Example: Vector Addition

For FP32:

```cpp
z[i] = x[i] + y[i];
```

```text
AI = 1/12 ≈ 0.083 FLOPs/byte
Bandwidth = 2 TB/s
Memory roof = 0.083 × 2 ≈ 0.166 TFLOP/s
```

Even if peak compute is 100 TFLOP/s, the kernel is limited to about 0.166 TFLOP/s by memory traffic.

## 7. Example: AI = 20

```text
AI = 20 FLOPs/byte
Bandwidth = 2 TB/s
Peak Compute = 100 TFLOP/s
```

```text
Memory roof = 20 × 2 = 40 TFLOP/s
Attainable = min(100, 40) = 40 TFLOP/s
```

This kernel is memory-bound.

## 8. Example: AI = 80

```text
AI = 80 FLOPs/byte
Bandwidth = 2 TB/s
Peak Compute = 100 TFLOP/s
```

```text
Memory roof = 80 × 2 = 160 TFLOP/s
Attainable = min(100, 160) = 100 TFLOP/s
```

This kernel is compute-bound.

## 9. Moving Right vs. Moving Up

### Move Right

Increase arithmetic intensity:

```text
tiling
register reuse
shared-memory reuse
kernel fusion
batching
lower precision
```

### Move Up

Improve achieved efficiency at roughly the same intensity:

```text
better coalescing
fewer bank conflicts
better instruction scheduling
higher Tensor Core utilization
less synchronization
better latency hiding
```

## 10. Important Distinction

Coalescing may not change logical FLOPs/byte, but it reduces wasted memory transactions. It mainly moves performance upward.

Kernel fusion can remove HBM reads and writes, increasing FLOPs per byte. It mainly moves the kernel right.

## 11. Why Real Kernels Sit Below the Roof

Possible causes:

```text
poor coalescing
cache misses
bank conflicts
branch divergence
synchronization
dependency chains
small problem size
kernel-launch overhead
low utilization
```

Roofline gives a high-level bound. Profiling explains the remaining gap.

## 12. Multiple Roofs

Advanced Roofline models may include:

```text
HBM bandwidth roof
L2 bandwidth roof
shared-memory bandwidth roof
CUDA Core roof
Tensor Core roof
```

A kernel can stop being HBM-bound and become limited by an on-chip bandwidth roof.

## 13. Precision Effects

Lower precision can:

```text
reduce bytes moved
increase arithmetic intensity
raise Tensor Core peak throughput
```

So FP16 or FP8 can both move a kernel right and raise the compute roof.

## 14. LLM Connection

### Prefill

```text
large GEMM
high token parallelism
high arithmetic intensity
often near compute roof
```

### Decode

```text
small batch
skinny GEMM or GEMV
low weight reuse
low arithmetic intensity
often on memory roof
```

Implications:

```text
prefill → improve Tensor Core and GEMM efficiency
decode → quantize, batch, reduce weight/KV traffic
```

## 15. Practical Workflow

1. Estimate FLOPs.
2. Estimate HBM bytes.
3. Compute arithmetic intensity.
4. Compute the hardware ridge point.
5. Determine the likely region.
6. Measure achieved performance.
7. Use profiler metrics to explain the gap.

## 16. Interview Takeaway

> The Roofline Model bounds attainable performance using arithmetic intensity, memory bandwidth, and peak compute throughput. Low-intensity kernels follow the sloped memory roof, while high-intensity kernels are capped by the flat compute roof. The ridge point is peak compute divided by memory bandwidth. Tiling and fusion move a kernel right, while better utilization and fewer stalls move it upward.

## 17. Key Summary

```text
Performance = min(Peak Compute, AI × Memory Bandwidth)
```

```text
left of ridge  → memory-bound
right of ridge → compute-bound
```

```text
move right → improve reuse or reduce bytes
move up    → improve implementation efficiency
```

## Exercises

### Exercise 1

```text
Peak Compute = 60 TFLOP/s
Memory Bandwidth = 1.5 TB/s
```

Calculate the ridge point.

### Exercise 2

For `AI = 10 FLOPs/byte`, calculate the memory-roof performance and classify the kernel.

### Exercise 3

For `AI = 100 FLOPs/byte`, calculate the Roofline upper bound.

### Exercise 4

Classify these as mainly moving right or up:

```text
kernel fusion
better coalescing
shared-memory tiling
removing bank conflicts
```

## Answers

### Exercise 1

```text
60 / 1.5 = 40 FLOPs/byte
```

### Exercise 2

```text
10 × 1.5 = 15 TFLOP/s
```

The kernel is memory-bound.

### Exercise 3

```text
memory roof = 100 × 1.5 = 150 TFLOP/s
compute roof = 60 TFLOP/s
upper bound = 60 TFLOP/s
```

The kernel is compute-bound.

### Exercise 4

```text
kernel fusion           → mainly right
better coalescing       → mainly up
shared-memory tiling    → mainly right
removing bank conflicts → mainly up
```
