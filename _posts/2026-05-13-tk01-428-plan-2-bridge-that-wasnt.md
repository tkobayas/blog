---
layout: post
title: "mvel#428 — Plan 2, and the bridge that wasn't"
date: 2026-05-13
type: phase-update
entry_type: note
subtype: diary
projects: [mvel3, drlx-parser]
tags: [mvel3, drlx-parser, lambda-registry, refactor, two-repo, codex]
---

# mvel#428 — Plan 2, and the bridge that wasn't

Yesterday I closed Plan 1: MVEL3's `LambdaRegistry` split into four components, eight commits, 733 tests. The DRLX side was a corpse — `mvn compile` failed on three files referencing the deleted class. Today was Plan 2: rewrite the metadata schema, migrate the consumers, write the Phase 0 tests, push both repos in lockstep.

The most interesting moment turned out to be a bridge that wasn't needed.

## Task 6 caught a plan-vs-implementation drift

Plan 2 Task 6 was meant to add `DrlxRuleBuilder.getBatchCompilerForTests()` — a test-only accessor exposing `MVELBatchCompiler.compileInvocationCount()` so the D1 test could assert "the runtime build didn't recompile anything". The spec named both the accessor and the counter as part of the locked DRLX-facing seam.

Claude paused at Task 6 and flagged it. Plan 1 had actually implemented the counter as a **static** method on `MVELCompiler`, not as an instance method on `MVELBatchCompiler` like the spec said. A static counter is JVM-global, so D1 can just call `MVELCompiler.compileInvocationCount()` directly. No accessor needed.

```java
int before = MVELCompiler.compileInvocationCount();
KieBase kieBase = builder.build(SIMPLE_RULE, metadata);
int after  = MVELCompiler.compileInvocationCount();

assertThat(after)
        .as("Runtime build with pre-built metadata must not recompile lambdas")
        .isEqualTo(before);
```

Two options: add a delegating method on MVELBatchCompiler so the spec's wording holds, or drop the accessor and update the spec. I picked drop. Adding a delegating method just to honour spec text I'd written a day earlier was the wrong reason to touch MVEL.

**Validate spec promises against the actual Plan 1 code before consuming them in Plan 2.** It's a single grep. Cheap insurance.

## The benchmark module wasn't in the inventory

Task 1 Step 1.5 inventoried files that would need migration:

```
grep -rln "LambdaRegistry\|getPhysicalPath\|PERSISTENCE_ENABLED\|DEFAULT_PERSISTENCE_PATH" \
    drlx-parser-core/src
```

Five hits. That matched the plan's "files modified" table — looked complete. We worked through Tasks 2 through 5, then `mvn compile` failed on four files in `drlx-parser-benchmark/`. The grep was scoped to `drlx-parser-core`. The benchmark module also referenced `LambdaRegistry`.

No rework — Task 5 was already the "constant migration" task, so the benchmark files folded into the same commit. But the inventory should have been repo-wide. **Inventory greps don't get to assume module scope.**

## Phase C in lockstep

Tasks 1–8 done locally on DRLX `main` — the previous session agreed to develop directly on main rather than carry a branch, on the basis that the no-net-cost in a private fork made it cheap. Five DRLX commits accumulated, none pushed. Final count: 176 main + 5 no-persist = 181 tests. Baseline was 175, so net +6 — the D-tests (D1–D4, D6, D7). D5 stayed covered by existing `DrlxMetadataMismatchMode` tests.

Then Phase C: `mvn install` MVEL one last time, push the branch to my fork, open the upstream PR, merge once green. Then push DRLX `main`. Both issues closed. The lockstep matters because between the MVEL merge and the DRLX push, anyone pulling DRLX `main` against MVEL `3.0.0-SNAPSHOT` would have hit exactly the compile failure I started today with — but without the local SNAPSHOT to fall back on.

## The post-merge bill

Right after the cutover, a follow-up review flagged two correctness issues that survived the refactor:

1. **`MVELBatchCompiler` no-persist mode still touches global runtime state.** The single-compiler path branches on `persistenceDir == null` before calling into the catalog and persistence manager. The batch path doesn't. So "no-persist batch mode" is not actually pure in-memory — it can still accumulate global dedup state and reuse persisted artifacts. Fix: mirror the single-compiler structure inside `MVELBatchCompiler.add(...)` and `compile(...)`.

2. **`Path.of(classFile)` bypasses typed metadata exceptions.** Both `LambdaRegistryStore` (MVEL) and `DrlxLambdaMetadata` (DRLX) call `Path.of(...)` directly while parsing persisted metadata. `Path.of` throws unchecked `InvalidPathException`, which escapes `InvalidLambdaRegistryException` and `InvalidDrlxLambdaMetadataException`, and on the DRLX side bypasses `DrlxMetadataMismatchMode` routing entirely.

Neither is a bug the spec would have caught — both are integration points the refactor preserved without re-examining.

Two notes for next time. Don't trust spec promises from yesterday — re-grep the code today before consuming them. And inventory greps must be repo-wide; module-scoped ones look complete and aren't. Tomorrow: the two findings, both lockstep across MVEL and DRLX. The refactor that was supposed to be done isn't, quite.
