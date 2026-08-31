# 00. Document Conventions

## Purpose

这套文档是未来 Go Performance Engineering Skill 的完整知识源，因此采用统一结构和标签。

## Recommendation Labels

```text
Production Safe
Conditional
Library-Level
Runtime Internal
Historical
Experimental
```

## Evidence Labels

```text
L0 Source inspection
L1 Runtime/compiler diagnostics
L2 Reproducible microbenchmark
L3 Component/service benchmark
L4 Production/canary
L5 Hardware PMU evidence
```

## Technique Entry Template

未来新增技巧时尽量使用：

```markdown
## Technique Name

### Cost Model

### Trigger / Symptom

### Why It Works

### Example

### Risks / Counterexamples

### Verification

### Recommendation
```

## Anti-Pattern

不要新增纯粹：

```text
“某写法比另一写法快”
```

的结论。

必须回答：

1. 哪一种成本被减少？
2. workload 条件是什么？
3. 如何证明？
4. 失败/反例是什么？
5. 风险等级是什么？

## Version Rule

任何依赖：

- runtime internals；
- compiler heuristics；
- allocator thresholds；
- GC scanning implementation；
- instruction selection；

的结论都必须标为版本敏感。
