# Day 02 — Memory Hierarchy and Data Movement

## Learning Goals

By the end of this lesson, you should be able to:

- Explain why modern processors use a memory hierarchy.
- Distinguish registers, caches, shared memory, DRAM, and HBM.
- Explain why data movement often costs more than arithmetic.
- Relate memory hierarchy to CPU, GPU, and TPU optimization.
- Identify common ways to increase locality and data reuse.

---

## 1. Why a Memory Hierarchy Exists

Processors can perform arithmetic much faster than large memories can deliver data. A single multiply or add may be cheap, while fetching its operands from distant memory can be much more expensive.

```text
small and fast
    ↓
large and slow
```

General tradeoff:

```text
closer to compute → lower latency, lower capacity, higher cost per byte
farther from compute → higher latency, higher capacity, lower cost per byte
```

---

## 2. CPU Memory Hierarchy

```text
Registers
L1 Cache
L2 Cache
L3 Cache
Main Memory (DRAM)
Storage
```

- **Registers:** closest to execution units; hold operands and intermediate results.
- **L1 cache:** very small and very fast; usually private to a CPU core.
- **L2 cache:** larger and slower than L1.
- **L3 cache:** larger again and often shared among cores.
- **DRAM:** much larger, but substantially higher latency.

---

## 3. GPU Memory Hierarchy

```text
Per-thread registers
Per-block shared memory
Per-SM L1 cache
GPU-wide L2 cache
Global memory / HBM / GDDR
Host memory
```

### Registers

Registers are logically private to each thread.

```cpp
float acc = 0.0f;
```

They are ideal for temporary values, loop accumulators, and thread-local output tiles. However, excessive register usage can reduce occupancy.

### Shared Memory

Shared memory is on-chip, programmer-managed storage shared by the threads of one block.

```cpp
__shared__ float tile[32][32];
```

Typical uses:

```text
tiling
block-level communication
data reuse
reductions
```

### L1 Cache

L1 is generally associated with an SM and managed by hardware.

```text
shared memory = software-managed storage
L1 cache      = hardware-managed cache
```

### L2 Cache

L2 is shared across the GPU's SMs and helps reduce traffic to HBM or GDDR.

### Global Memory

Global memory is large off-chip memory. It provides high capacity and bandwidth, but also high latency compared with on-chip storage.

---

## 4. Why Data Movement Matters

Consider:

```text
acc += a * b
```

The arithmetic is cheap. But the processor may first need to load `a`, load `b`, obtain `acc`, perform the operation, and eventually store the result.

If the values are already in registers, the operation is efficient. If they repeatedly come from global memory, data movement can dominate execution time.

> The fastest data is data that does not need to move.

---

## 5. Data Reuse

Data reuse means loading a value once and using it multiple times.

### Poor reuse

```cpp
for (...) {
    float value = global_memory[index];
    use(value);
}
```

### Better reuse

```cpp
float value = global_memory[index];

for (...) {
    use(value);
}
```

The value may stay in a register.

### Block-level reuse

A block can cooperatively load a tile into shared memory, then allow many threads to reuse it.

```text
many global loads
```

becomes:

```text
one cooperative global load
many shared-memory accesses
```

---

## 6. Locality

### Temporal Locality

A recently used value is likely to be used again soon.

Example: reusing the same weight tile for several output calculations.

### Spatial Locality

Nearby addresses are likely to be accessed together.

```text
thread 0 reads x[0]
thread 1 reads x[1]
thread 2 reads x[2]
...
```

On GPUs, this supports coalesced global-memory accesses.

---

## 7. Matrix Multiplication Example

For:

```text
C = A × B
```

A naive implementation may repeatedly fetch elements of `A` and `B` from global memory. A tiled implementation uses:

```text
global memory
    ↓
shared-memory tile
    ↓
register operands and accumulators
    ↓
multiply-accumulate
```

Example:

```text
A tile: 128 × 32
B tile: 32 × 128
C tile: 128 × 128
```

- The `A` tile is reused across many output columns.
- The `B` tile is reused across many output rows.
- Partial sums for `C` stay in registers.

This is the foundation of high-performance GEMM.

---

## 8. CPU vs. GPU vs. TPU

### CPU

```text
cache locality
prefetching
vectorization
out-of-order execution
```

### GPU

```text
coalesced global-memory access
shared-memory tiling
register reuse
latency hiding
high memory-level parallelism
```

### TPU / Systolic Array

```text
on-chip SRAM
regular dataflow
weight or activation reuse
keeping the PE array busy
```

Common principle:

```text
move data less
reuse data more
```

---

## 9. A Simple Bandwidth Estimate

For an FP32 kernel:

```cpp
y[i] = x[i] * 2.0f;
```

Per element:

```text
read x[i]  = 4 bytes
write y[i] = 4 bytes
total      = 8 bytes
```

For 100 million elements:

```text
total traffic ≈ 800 MB
```

At an effective memory bandwidth of 800 GB/s, the ideal lower bound is approximately:

```text
800 MB / 800 GB/s = 1 ms
```

This ignores overhead and assumes perfect bandwidth utilization.

For a bandwidth-bound kernel:

```text
runtime lower bound ≈ bytes moved / effective bandwidth
```

---

## 10. Register Pressure and Occupancy

Registers are fast but finite. If each thread uses many registers, fewer warps may fit on an SM.

```text
more register reuse is useful
too much register usage can reduce occupancy
```

Optimization therefore balances:

```text
register reuse
occupancy
instruction-level parallelism
```

---

## 11. Shared Memory Is Not Automatically Better

Shared memory introduces:

```text
explicit loads and stores
synchronization
address calculation
possible bank conflicts
```

If a value is used only once, copying it into shared memory may add unnecessary work.

A useful question is:

> Will this value be reused enough to justify placing it in shared memory?

---

## 12. Interview Takeaway

> Modern processors use a memory hierarchy because fast storage is expensive and limited. High-performance kernels keep frequently reused data close to the compute units, usually in registers or on-chip SRAM, while minimizing traffic to DRAM or HBM. On GPUs, this is commonly achieved through coalesced loads, shared-memory tiling, and register blocking.

---

## 13. Key Summary

```text
Registers      → fastest, smallest, thread-local
Shared memory  → on-chip, block-shared, software-managed
L1 cache       → on-chip, per-SM, hardware-managed
L2 cache       → shared across SMs
HBM / GDDR     → large, high bandwidth, high latency
Host memory    → farther from GPU
```

Core optimization principle:

```text
maximize locality
maximize reuse
minimize off-chip data movement
```

---

## Exercises

### Exercise 1

Why is loading the same value from HBM ten times worse than loading it once and reusing it from a register?

### Exercise 2

For the following FP16 kernel, calculate the logical memory traffic per element:

```cpp
z[i] = x[i] + y[i];
```

### Exercise 3

Why is shared memory useful for tiled matrix multiplication?

### Exercise 4

True or false: using more registers always improves GPU performance. Explain.

---

## Exercise Answers

### Exercise 1

HBM access has much higher latency and energy cost than register access. Register reuse reduces memory traffic and waiting.

### Exercise 2

```text
read x[i]  = 2 bytes
read y[i]  = 2 bytes
write z[i] = 2 bytes
total      = 6 bytes
```

### Exercise 3

A tile loaded once from global memory can be reused by many threads, reducing repeated global-memory accesses and increasing arithmetic intensity.

### Exercise 4

False. More registers can improve reuse, but excessive register usage can reduce the number of resident warps and lower occupancy.
