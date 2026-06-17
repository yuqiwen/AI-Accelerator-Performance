# Day 01 — Latency, Throughput, and Bandwidth

## Learning Goals

By the end of this lesson, you should be able to:

- Distinguish latency, throughput, and bandwidth.
- Explain why GPUs can tolerate high operation latency.
- Recognize when a simple kernel is likely memory-bandwidth-bound.
- Connect these concepts to GPU kernels and AI inference systems.

---

## 1. Latency

**Latency** measures how long one operation or task takes from start to completion.

Examples:

- A memory load takes a certain number of nanoseconds.
- A CUDA kernel launch takes a certain number of microseconds.
- One LLM decode step takes a certain number of milliseconds.

A useful mental model:

> Latency asks: “How long does one task wait before it finishes?”

### Example

If one inference request finishes in 20 ms:

```text
latency = 20 ms/request
```

---

## 2. Throughput

**Throughput** measures how much work a system completes per unit time.

Examples:

- Requests per second
- Tokens per second
- Images per second
- Floating-point operations per second

A useful mental model:

> Throughput asks: “How many tasks can the system finish per second?”

### Important distinction

Latency and throughput are not generally exact reciprocals.

A system may have relatively high per-request latency while still achieving high throughput by processing many requests concurrently.

For example:

```text
single-request latency = 10 ms
concurrent requests     = 100
overall throughput      = high
```

This is common on GPUs.

---

## 3. Bandwidth

**Bandwidth** measures how much data can be transferred per unit time.

Typical units:

```text
GB/s
TB/s
```

A useful mental model:

> Bandwidth asks: “How many bytes can move through the system per second?”

Examples include:

- CPU memory bandwidth
- GPU HBM bandwidth
- PCIe bandwidth
- NVLink bandwidth
- Network bandwidth

Bandwidth is about data movement, while throughput is about completed work.

---

## 4. Latency vs. Throughput on CPUs and GPUs

CPUs generally emphasize:

```text
low latency
strong single-thread performance
branch prediction
out-of-order execution
large caches
```

GPUs generally emphasize:

```text
high throughput
massive parallelism
high memory bandwidth
latency hiding
```

A GPU does not necessarily make one memory access very fast. Instead, it keeps many warps available.

When one warp stalls on memory:

```text
warp 0: waiting for memory
warp 1: ready, execute
warp 2: ready, execute
```

The original memory latency still exists, but useful work from other warps overlaps with it.

This is called:

```text
latency hiding
```

---

## 5. Memory-Bandwidth-Bound Example

Consider this elementwise kernel:

```cpp
__global__ void scale(const float* x, float* y, int n) {
    int i = blockIdx.x * blockDim.x + threadIdx.x;

    if (i < n) {
        y[i] = x[i] * 2.0f;
    }
}
```

For each element:

```text
read x[i]   = 4 bytes
write y[i]  = 4 bytes
computation = 1 multiplication
```

Therefore:

```text
memory traffic = 8 bytes
compute work   = 1 floating-point operation
```

This kernel performs very little computation relative to the amount of data moved.

It is therefore likely:

```text
memory-bandwidth-bound
```

Increasing raw arithmetic throughput may not help much because the compute units are waiting for data.

---

## 6. Another Example

Suppose a kernel computes:

```cpp
z[i] = x[i] + y[i];
```

For each element:

```text
read x[i]   = 4 bytes
read y[i]   = 4 bytes
write z[i]  = 4 bytes
add         = 1 operation
```

Total memory traffic:

```text
12 bytes per output element
```

Total compute:

```text
1 floating-point addition
```

This is also likely memory-bandwidth-bound.

---

## 7. Why Matrix Multiplication Is Different

For matrix multiplication:

```text
C = A × B
```

each element of `A` and `B` can be reused many times.

For example:

```text
A[i][k] contributes to many C[i][j]
B[k][j] contributes to many C[i][j]
```

If the implementation uses tiling correctly, one load from memory can support many multiply-accumulate operations.

This increases the amount of computation performed per byte transferred.

Large matrix multiplication is therefore much more likely to become:

```text
compute-bound
```

This is one reason GPUs and TPUs are especially effective for deep-learning workloads dominated by GEMM.

---

## 8. AI Infra Examples

### LLM Latency

```text
time from request submission to generated response
```

Common latency-related metrics include:

```text
time to first token
inter-token latency
end-to-end request latency
```

### LLM Throughput

```text
tokens generated per second
requests completed per second
```

### Accelerator Bandwidth

```text
HBM bytes transferred per second
PCIe bytes transferred per second
NVLink bytes transferred per second
```

An LLM server can have high total token throughput while an individual request still has noticeable latency.

---

## 9. Interview Takeaway

A strong interview answer:

> Latency measures how long one operation takes, throughput measures how many operations finish per unit time, and bandwidth measures how much data can be transferred per unit time. GPUs often tolerate high individual operation latency by scheduling many warps concurrently, which improves overall throughput through latency hiding.

---

## 10. Key Summary

```text
Latency    = time required for one task
Throughput = tasks completed per unit time
Bandwidth  = bytes transferred per unit time
```

The key GPU insight:

```text
GPUs usually optimize for throughput rather than minimum single-thread latency.
```

They achieve this through:

```text
many resident warps
fast scheduling
high memory bandwidth
latency hiding
```

---

## Exercises

### Exercise 1

A kernel performs the following for every element:

```text
read two float values
add them
write one float value
```

Answer:

1. How many bytes are transferred per element?
2. How many floating-point operations are performed?
3. Is the kernel more likely compute-bound or memory-bandwidth-bound?

### Exercise 2

System A:

```text
request latency = 1 ms
concurrency      = 1
```

System B:

```text
request latency = 10 ms
concurrency      = 100
```

Which system has lower latency?

Which system may achieve higher throughput?

### Exercise 3

Explain in your own words why a GPU can have high memory-access latency while still delivering high overall throughput.

---

## Exercise Answers

### Exercise 1

```text
two float reads = 8 bytes
one float write = 4 bytes
total           = 12 bytes
compute         = 1 addition
```

The kernel is likely memory-bandwidth-bound.

### Exercise 2

System A has lower per-request latency.

System B may achieve higher total throughput because it processes many requests concurrently.

### Exercise 3

When one warp waits for memory, the GPU scheduler can execute another ready warp. The memory latency remains, but useful work from other warps overlaps with the wait.
