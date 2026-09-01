# Branch Prediction

## CPU Pipelines

Modern CPUs execute instructions speculatively.

Branches require prediction:

```go
if condition {
    work()
}
```

## Predictable Branches

Patterns like:

```text
false false false false true true true
```

are easier to predict.

## Unpredictable Branches

Random patterns cause more misprediction.

This can stall the pipeline.

## Branchless Code

Branchless programming is not automatically faster.

Replacing a predictable branch with extra arithmetic may increase total work.

Use measurement:

- benchmark;
- profiling;
- hardware counters when necessary.
