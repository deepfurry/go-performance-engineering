# Go Performance Engineering Skill

This folder is the first Skill extraction of the detailed `go-performance-engineering` knowledge base.

## Layout

```text
go-performance-engineering-skill/
├── SKILL.md
├── README.md
└── references/
    ├── methodology.md
    ├── optimization-review.md
    ├── cpu-memory.md
    ├── concurrency.md
    ├── compiler.md
    ├── memory-gc.md
    ├── unsafe-runtime.md
    ├── benchmarking-profiling.md
    ├── technique-catalog.md
    ├── tool-reference.md
    ├── version-notes.md
    └── official-sources.md
```

## Design

`SKILL.md` is intentionally much smaller than the source documentation. It contains:

- the mandatory workflow;
- diagnostic routing;
- evidence levels;
- risk/escalation rules;
- maintainability requirements;
- stop conditions.

Detailed technical explanations live under `references/` and should be loaded only when relevant.

## Maintainability policy

The Skill treats maintainability as part of performance correctness.

Non-obvious performance changes must preserve the rationale in adjacent comments, and unsafe/runtime-boundary changes must document their correctness and lifetime contract.

Ordinary idiomatic optimizations should not be buried under redundant comments.

## Source knowledge base

The references are extracted from the detailed Go Performance Engineering documentation created before this Skill. They remain intentionally more detailed than `SKILL.md` so the Skill can progressively load context rather than carrying all performance knowledge in every invocation.
