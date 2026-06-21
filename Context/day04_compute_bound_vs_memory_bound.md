# Day 04 — Compute-Bound vs. Memory-Bound

## Learning Goals

- Distinguish compute-bound, bandwidth-bound, and latency-bound kernels.
- Use arithmetic intensity and the hardware balance point.
- Match optimizations to the actual bottleneck.
- Connect the ideas to reduction, GEMM, and LLM inference.

## 1. Two Main Limits

A GPU kernel mainly consumes:

```text
compute resources
memory-system resources
```

Compute resources include CUDA cores, Tensor Cores, integer units, and instruction issue capacity.

Memory resources include HBM bandwidth, cache bandwidth, memory latency, and load/store pipelines.

The resource that reaches its limit first determines the bottleneck.

## 2. Compute-Bound

A kernel is compute-bound when arithmetic throughput is the main limitation.

Typical signs:

```text
high arithmetic intensity
high compute-unit utilization
performance scales with compute throughput
```

Examples:

```text
large well-tiled GEMM
large convolution
Tensor Core-heavy kernels
```

Useful optimizations:

```text
Tensor Core usage
better tiling
register blocking
instruction scheduling
reducing unnecessary FLOPs
```

## 3. Memory-Bound

A kernel is memory-bound when data movement is the main limitation.

Typical signs:

```text
low arithmetic intensity
high memory-bandwidth utilization
low compute utilization
```

Examples:

```text
vector addition
copy kernels
simple elementwise operations
embedding lookup
some reductions
small-batch LLM decode
```

Useful optimizations:

```text
reduce bytes moved
coalesce accesses
fuse kernels
use lower precision
increase data reuse
avoid redundant loads and stores
```

## 4. Hardware Balance Point

Suppose:

```text
peak compute = 80 TFLOP/s
memory bandwidth = 2 TB/s
```

Then:

```text
balance point = 80 / 2 = 40 FLOPs/byte
```

A first-pass rule:

```text
AI below 40 FLOPs/byte → likely memory-bound
AI above 40 FLOPs/byte → potentially compute-bound
```

This is approximate because real kernels rarely reach both theoretical peaks.

## 5. Vector Addition

```cpp
z[i] = x[i] + y[i];
```

For FP32:

```text
bytes moved = 12
FLOPs = 1
AI = 1/12 ≈ 0.083 FLOPs/byte
```

If the hardware balance point is 40 FLOPs/byte:

```text
0.083 << 40
```

The kernel is strongly memory-bound.

Optimizing the addition instruction is unlikely to help much.

## 6. Large GEMM

For:

```text
C = A × B
```

a tiled implementation reuses `A` and `B` values many times.

This can create high arithmetic intensity and make the kernel compute-bound.

Then useful optimizations include:

```text
Tensor Core utilization
better block/warp tile shapes
register reuse
pipeline overlap
instruction efficiency
```

## 7. Memory Latency vs. Memory Bandwidth

Memory-bound is not one single condition.

### Latency-Bound

The kernel waits on individual accesses that cannot be hidden.

Examples:

```text
pointer chasing
irregular graph traversal
dependent loads
small workloads
low useful concurrency
```

Possible fixes:

```text
increase independent work
prefetch
improve locality
remove dependency chains
increase useful occupancy
```

### Bandwidth-Bound

The kernel has enough parallel requests to saturate the memory channel.

Examples:

```text
large streaming reads/writes
vector operations
memory copy
simple elementwise kernels
```

Possible fixes:

```text
move fewer bytes
fuse kernels
use lower precision
reuse data
avoid redundant traffic
```

Important:

```text
latency-bound != bandwidth-bound
```

## 8. Reduction

For summing FP32 values:

```text
4 bytes read per value
about 1 addition per value
AI ≈ 0.25 FLOPs/byte
```

The first stage of a large reduction is often bandwidth-bound.

However, reduction also introduces:

```text
synchronization
cross-warp communication
dependency chains
final aggregation overhead
```

So real performance may have multiple bottlenecks.

## 9. Occupancy Is Not the Goal

Higher occupancy can help hide latency.

But if HBM bandwidth is already saturated:

```text
more warps cannot create more bandwidth
```

If Tensor Cores are already saturated:

```text
more warps may not increase compute throughput
```

Therefore:

```text
occupancy is a tool, not the final objective
```

## 10. Optimization by Bottleneck

### Memory-Bandwidth-Bound

```text
reduce bytes moved
coalesce memory access
use lower precision
fuse kernels
increase reuse
```

### Memory-Latency-Bound

```text
increase independent memory operations
improve locality
prefetch
remove dependent loads
increase useful concurrency
```

### Compute-Bound

```text
use Tensor Cores
improve tiling
reduce unnecessary FLOPs
improve instruction scheduling
increase register reuse
```

### Synchronization-Bound

```text
reduce barriers
use warp-level primitives
reduce cross-block communication
improve work partitioning
```

## 11. Kernel Fusion

Unfused:

```cpp
tmp[i] = x[i] + bias[i];
y[i] = relu(tmp[i]);
```

This may write `tmp` to HBM and read it back.

Fused:

```cpp
y[i] = relu(x[i] + bias[i]);
```

The intermediate can remain in a register.

Benefits:

```text
fewer bytes moved
higher arithmetic intensity
less kernel-launch overhead
```

## 12. LLM Prefill vs. Decode

### Prefill

```text
many tokens
large GEMM
high weight reuse
higher arithmetic intensity
often more compute-bound
```

### Decode

```text
one or a few new tokens
skinny GEMM or GEMV
low weight reuse
large weight traffic
often memory-bandwidth-bound
```

Practical consequences:

```text
batching improves decode throughput
quantization reduces weight bandwidth
continuous batching improves utilization
```

## 13. Diagnostic Process

1. Estimate total FLOPs.
2. Estimate total bytes moved.
3. Compute arithmetic intensity.
4. Compare it with the hardware balance point.
5. Confirm the hypothesis using a profiler.

Useful profiler metrics:

```text
achieved memory bandwidth
compute utilization
Tensor Core utilization
cache hit rate
stall reasons
occupancy
```

## 14. Interview Takeaway

> A compute-bound kernel is limited by arithmetic throughput, while a memory-bound kernel is limited by data movement. I first estimate arithmetic intensity and compare it with the hardware compute-to-bandwidth ratio. For memory-bound kernels, I reduce bytes moved through fusion, reuse, coalescing, or lower precision. For compute-bound kernels, I focus on Tensor Core utilization, tiling, instruction efficiency, and unnecessary arithmetic. I then verify the hypothesis with profiling.

## 15. Key Summary

```text
compute-bound:
compute units reach their limit first

memory-bandwidth-bound:
the memory channel reaches its limit first

memory-latency-bound:
individual accesses cannot be hidden effectively
```

The optimization must match the bottleneck.

## Exercises

### Exercise 1

A kernel has:

```text
AI = 0.5 FLOPs/byte
GPU balance point = 30 FLOPs/byte
```

What is the likely bottleneck?

### Exercise 2

A kernel reaches 95% of peak HBM bandwidth and 20% of peak compute throughput.

What should you optimize?

### Exercise 3

A pointer-chasing kernel reaches low memory bandwidth and low compute utilization.

Can it still be memory-limited?

### Exercise 4

Why might increasing occupancy fail to accelerate a vector-add kernel that already saturates memory bandwidth?

## Answers

### Exercise 1

It is likely memory-bound because:

```text
0.5 << 30
```

### Exercise 2

It is likely bandwidth-bound. Reduce bytes moved through fusion, lower precision, reuse, or elimination of redundant traffic.

### Exercise 3

Yes. It may be latency-bound because dependent accesses prevent enough concurrent requests from being issued.

### Exercise 4

The physical memory channel is already saturated. Additional warps cannot increase its capacity.
