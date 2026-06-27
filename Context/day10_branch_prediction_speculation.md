# Day 10 — Branch Prediction and Speculation

## Learning Goals
- Explain why branches hurt deep, wide pipelines.
- Distinguish branch direction prediction from target prediction.
- Understand static and dynamic prediction.
- Explain speculative execution and misprediction recovery.
- Connect branch behavior to C++ and GPU control flow.

## 1. Why Branches Are Expensive

```cpp
if (x > 0) y = a;
else y = b;
```

Before the condition resolves, the CPU does not know which instruction address to fetch next. Waiting would create pipeline bubbles, so the CPU predicts and continues speculatively.

## 2. Two Predictions

```text
Direction prediction:
taken or not taken

Target prediction:
if taken, where does execution continue?
```

A predictor may know the branch is taken but still need the target address.

## 3. Static Prediction

Simple fixed rules include:

```text
always predict not taken
backward branches predict taken
forward branches predict not taken
```

Backward branches are often loops, so “taken” is usually a good guess until loop exit.

## 4. Dynamic Prediction

Dynamic predictors learn from runtime behavior.

A one-bit predictor remembers the last outcome. A two-bit saturating counter uses four states:

```text
Strongly Not Taken
Weakly Not Taken
Weakly Taken
Strongly Taken
```

One unusual outcome does not immediately reverse the prediction.

## 5. Branch History

Predictors may use:

```text
local history:
past outcomes of this branch

global history:
recent outcomes of multiple branches
```

History indexes prediction tables that estimate future outcomes.

## 6. Branch Target Buffer

A BTB maps:

```text
branch instruction address
→ predicted target address
```

It helps instruction fetch continue immediately after a predicted-taken branch.

## 7. Return Address Stack

Function returns are indirect branches. A return-address stack predicts them by pushing call return addresses and popping them on return.

## 8. Speculative Execution

After predicting a branch, the CPU fetches, renames, schedules, and may execute instructions from the predicted path.

```text
correct prediction:
speculative work becomes useful

wrong prediction:
younger wrong-path instructions are discarded
```

## 9. Recovery

On misprediction, the CPU must:

```text
flush younger instructions
restore rename state
reclaim wrong-path physical registers
redirect fetch
```

The ROB and rename checkpoints support recovery.

## 10. Why Deep and Wide CPUs Care More

A deeper pipeline has more stages between fetch and branch resolution.

A wider CPU may fetch and decode many instructions per cycle.

Therefore one misprediction may waste many cycles and many in-flight instructions.

## 11. Predictable vs. Unpredictable Branches

Predictable:

```cpp
for (int i = 0; i < n; ++i) { ... }
```

Hard to predict:

```cpp
if (random_value & 1) { ... }
```

Random-looking outcomes can cause frequent pipeline flushes.

## 12. Sorted vs. Random Data

```cpp
for (int i = 0; i < n; ++i) {
    if (a[i] > threshold) {
        sum += a[i];
    }
}
```

Sorted data may produce long runs of the same outcome, which are easy to predict.

Random data near the threshold may alternate unpredictably and run slower even with the same Big-O complexity.

## 13. Branchless Code

Possible branchless form:

```cpp
sum += (a[i] > threshold) ? a[i] : 0;
```

The compiler may use a conditional move or masked instruction.

Potential benefits:

```text
avoid misprediction
help vectorization
```

Potential costs:

```text
execute unnecessary work
more instructions
higher register pressure
```

Branchless code is not automatically faster.

## 14. Predication

Predication conditionally commits an instruction result instead of changing control flow.

CPUs may use conditional moves or vector masks.

GPUs often use predication for short divergent branches.

## 15. CPU Branching vs. GPU Divergence

```text
CPU:
predict one control-flow path and speculate

GPU:
different warp lanes may need different paths
```

A GPU may execute multiple paths under different active masks.

## 16. Security Note

Speculative instructions may change caches even when architectural results are rolled back.

This distinction underlies Spectre-class attacks:

```text
architectural state rolls back
microarchitectural side effects may remain
```

## 17. Interview Takeaway

> Branch prediction keeps a deep, wide CPU pipeline busy by predicting branch direction and target before the condition resolves. The CPU speculatively executes the predicted path and commits only after validation. On a misprediction, younger instructions are flushed and rename state is restored. Dynamic predictors use history, saturating counters, BTBs, and return-address stacks.

## 18. Key Summary

```text
Direction predictor:
taken or not taken

BTB:
predicted target address

Return Address Stack:
predict function returns

ROB/checkpoints:
recover after misprediction
```

## Exercises

1. Why does misprediction cost more on a deeper pipeline?
2. Why is a two-bit predictor better for loops than a one-bit predictor?
3. What is the difference between a BTB and a direction predictor?
4. When can branchless code be slower?

## Answers

1. More wrong-path instructions and stages must be discarded and refilled.
2. One loop-exit outcome does not immediately reverse the prediction.
3. Direction predicts taken/not taken; BTB predicts the taken target.
4. When the branch is predictable or branchless code performs expensive unnecessary work.
