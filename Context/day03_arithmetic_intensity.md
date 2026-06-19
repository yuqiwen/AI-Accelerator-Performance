# Day 03 — Arithmetic Intensity

## Learning Goals

- Define arithmetic intensity.
- Calculate FLOPs per byte for simple kernels.
- Explain why low arithmetic intensity often means memory-bound behavior.
- Explain how tiling, fusion, batching, and lower precision improve performance.
- Connect arithmetic intensity to GEMM and LLM prefill/decode.

## 1. Definition

Arithmetic intensity measures computation per byte moved:

```text
Arithmetic Intensity = FLOPs / Bytes Moved
```

Unit:

```text
FLOPs per byte
```

Low arithmetic intensity means lots of data movement for little computation. High arithmetic intensity means each loaded byte supports more arithmetic.

## 2. Vector Addition

```cpp
z[i] = x[i] + y[i];
```

For FP32:

```text
read x = 4 bytes
read y = 4 bytes
write z = 4 bytes
total = 12 bytes
```

Work:

```text
1 addition = 1 FLOP
```

Therefore:

```text
AI = 1 / 12 ≈ 0.083 FLOPs/byte
```

This is very low, so the kernel is likely memory-bandwidth-bound.

## 3. Fused Multiply-Add

```cpp
y[i] = a * x[i] + y[i];
```

For FP32:

```text
read x = 4 bytes
read y = 4 bytes
write y = 4 bytes
total = 12 bytes
```

Work:

```text
1 multiply + 1 add = 2 FLOPs
```

Therefore:

```text
AI = 2 / 12 ≈ 0.167 FLOPs/byte
```

A hardware FMA may execute this as one instruction, but performance models usually count it as 2 FLOPs.

## 4. Why Reuse Matters

If a value is loaded once and reused many times from a register or shared memory, the byte traffic grows slowly while the FLOP count grows more quickly.

```cpp
float x = input[i];

for (int k = 0; k < 10; ++k) {
    acc += x * weights[k];
}
```

The value `x` can remain in a register instead of being reloaded for every iteration.

## 5. Matrix Multiplication

For square GEMM:

```text
C = A × B
```

with matrices of size `N × N`, the operation count is approximately:

```text
2N³ FLOPs
```

The minimum logical FP32 data size is on the order of:

```text
A + B + C ≈ 12N² bytes
```

So the idealized arithmetic intensity is approximately:

```text
2N³ / 12N² = N/6 FLOPs/byte
```

As `N` grows, arithmetic intensity grows. This is one reason large GEMM can become compute-bound.

## 6. Naive GEMM vs. Tiled GEMM

A naive implementation may repeatedly load the same `A` and `B` values from global memory.

A tiled implementation does:

```text
global memory
    ↓
shared-memory tiles
    ↓
register operands and accumulators
    ↓
many multiply-accumulate operations
```

Tiling does not reduce the mathematical FLOP count. It reduces expensive global-memory traffic, increasing FLOPs per byte.

## 7. Arithmetic Intensity Depends on Memory Level

Arithmetic intensity can be measured relative to:

```text
HBM traffic
L2 traffic
shared-memory traffic
register traffic
```

For first-pass GPU analysis, it usually means FLOPs per byte transferred to or from HBM/global memory.

## 8. Precision Changes the Byte Count

For:

```cpp
z[i] = x[i] + y[i];
```

### FP32

```text
traffic = 12 bytes
AI = 1/12 ≈ 0.083 FLOPs/byte
```

### FP16 or BF16

```text
traffic = 6 bytes
AI = 1/6 ≈ 0.167 FLOPs/byte
```

### FP8

```text
traffic = 3 bytes
AI = 1/3 ≈ 0.333 FLOPs/byte
```

Lower precision increases arithmetic intensity relative to memory traffic, although the kernel may still remain bandwidth-bound.

## 9. LLM Prefill vs. Decode

### Prefill

Many tokens are processed together, creating large matrix-matrix multiplications. Weights and activations are reused across many operations.

```text
higher arithmetic intensity
more likely compute-bound
```

### Decode

Only one or a few new tokens are processed at a time. Operations become skinny GEMM or GEMV-like, and large weight matrices may be read for relatively little computation.

```text
lower arithmetic intensity
often memory-bandwidth-bound
```

## 10. Hardware Balance Point

Suppose a GPU has:

```text
peak compute = 100 TFLOP/s
HBM bandwidth = 2 TB/s
```

The balance point is:

```text
100 / 2 = 50 FLOPs/byte
```

Interpretation:

```text
AI below 50 FLOPs/byte → bandwidth may dominate
AI above 50 FLOPs/byte → compute may dominate
```

This is the foundation of the Roofline Model.

## 11. Ways to Increase Arithmetic Intensity

### Tiling

Load a tile once and reuse it many times.

### Register Blocking

Keep operands and partial sums in registers.

### Kernel Fusion

Avoid writing intermediate values to HBM and reading them back.

Unfused:

```text
tmp = x + bias
y = relu(tmp)
```

Fused:

```text
y = relu(x + bias)
```

### Batching

Process more inputs together so weights are reused.

### Lower Precision

Reduce the number of bytes per element.

## 12. Interview Takeaway

> Arithmetic intensity is the ratio of arithmetic operations to bytes transferred, usually measured in FLOPs per byte. Low-intensity kernels such as elementwise operations are often memory-bandwidth-bound, while high-intensity kernels such as well-tiled GEMM can become compute-bound. Tiling, fusion, batching, register reuse, and lower precision improve arithmetic intensity by doing more work per byte moved.

## 13. Key Summary

```text
Arithmetic Intensity = FLOPs / Bytes Moved
```

```text
low AI  → lots of movement, little compute
high AI → more compute per byte
```

The core idea:

```text
reuse loaded data before fetching new data
```

## Exercises

### Exercise 1

For FP32:

```cpp
y[i] = x[i] * x[i];
```

Assume one load and one store. Calculate bytes moved, FLOPs, and arithmetic intensity.

### Exercise 2

For FP16:

```cpp
z[i] = x[i] + y[i];
```

Calculate arithmetic intensity.

### Exercise 3

A GPU has:

```text
peak compute = 60 TFLOP/s
memory bandwidth = 1.5 TB/s
```

Calculate its balance point.

### Exercise 4

Why does tiling improve arithmetic intensity even though it does not reduce the mathematical FLOP count?

## Exercise Answers

### Exercise 1

```text
read x = 4 bytes
write y = 4 bytes
total = 8 bytes
FLOPs = 1
AI = 1/8 = 0.125 FLOPs/byte
```

### Exercise 2

```text
traffic = 6 bytes
FLOPs = 1
AI = 1/6 ≈ 0.167 FLOPs/byte
```

### Exercise 3

```text
60 / 1.5 = 40 FLOPs/byte
```

### Exercise 4

Tiling lets data loaded from global memory be reused many times from on-chip storage. The FLOP count stays the same, but HBM byte traffic decreases.
