# AI-Accelerator-Performance

This repo is my study notebook for AI hardware and accelerator performance.

The goal is to build a practical review path for understanding how GPUs, TPUs,
and other AI accelerators execute deep learning workloads. The notes focus on
performance concepts that show up often in interviews, system design
discussions, and real AI infrastructure work.

## What This Covers

- Core performance metrics: latency, throughput, bandwidth, utilization, and bottlenecks.
- GPU execution basics: warps, occupancy, memory hierarchy, and latency hiding.
- AI workload patterns: GEMM, attention, inference serving, batching, and token generation.
- Hardware tradeoffs: compute-bound vs. memory-bound workloads, interconnects, HBM, and scaling.
- Interview-style takeaways and exercises for quick review.

## How to Use This Repo

Read the daily notes in order if you are building the foundation from scratch.
For quick review, jump directly to the topic you want from the table below.
Each note is designed to be short enough for daily study but concrete enough to
connect hardware concepts back to AI systems.

## Daily Notes

| Day | Topic |
| --- | --- |
| [Day 01](Context/day01_latency_throughput_bandwidth.md) | Latency, throughput, and bandwidth |
| [Day 02](Context/day02_memory_hierarchy_data_movement.md) | Memory hierarchy and data movement |
| [Day 03](Context/day03_arithmetic_intensity.md) | Arithmetic intensity |
| [Day 04](Context/day04_compute_bound_vs_memory_bound.md) | Compute-bound vs. memory-bound workloads |
| [Day 05](Context/day05_roofline_model.md) | Roofline model |
| [Day 06](Context/day06_cpu_pipeline.md) | CPU pipeline |
| [Day 07](Context/day07_superscalar_ilp.md) | Superscalar execution and instruction-level parallelism |
| [Day 08](Context/day08_out_of_order_execution.md) | Out-of-order execution |
| [Day 09](Context/day09_register_renaming_rob.md) | Register renaming and reorder buffer |
