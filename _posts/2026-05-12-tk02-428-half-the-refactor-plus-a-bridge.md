---
layout: post
title: "mvel#428 — half the refactor, plus a bridge commit"
date: 2026-05-12
type: phase-update
entry_type: note
subtype: diary
projects: [mvel3, drlx-parser]
tags: [mvel3, lambda-registry, refactor, codex, two-repo, enum-init]
---

# mvel#428 — half the refactor, plus a bridge commit

Two sessions in one day. Morning closed #45 (DataStore.update). Afternoon I switched repos and started mvel/mvel#428 — the lambda-persistence refactor Codex had already drafted in `LambdaRegistry_Refactor.md` last week. The doc was thorough, so I asked Claude to brainstorm it with me step by step rather than just hand off. Twelve rounds of back-and-forth surfaced three real ambiguities Codex hadn't pinned down, and the plan we wrote caught two more.

## The three Codex didn't pin down

Codex's plan listed four new components but kept some boundaries fuzzy. Walking through it as a Q&A:

1. **Who holds `physicalId → ArtifactRef`?** Codex's literal API put it on `LambdaCatalog`. But Catalog "must not touch the filesystem" — and a path-bearing record is arguably an FS concern. I picked option (c): `LambdaPersistenceManager`. Codex confirmed and rewrote the plan accordingly.
2. **How heavy is the persistence manager?** Light — query/store only. Compilers keep orchestration. Codex agreed.
3. **What does the persisted format actually look like?** Codex hadn't picked one; said "replace it." I chose `java.util.Properties` with `format.version=2`. Boring, JDK-built-in, diffable.

Once those were nailed, the rest of the design fell out: narrow DRLX-facing surface (just `ArtifactRef`, `LambdaArtifactLoader`, two static accessors, and `MVELBatchCompiler.getArtifactRef(handle)`). DRLX never sees `LambdaCatalog` or `LambdaPersistenceManager`.

## What I caught in the plan

Claude wrote the implementation plan; I reviewed before execution. Two real blockers:

- **Phase 1 deleted `entriesByPhysicalId` while keeping `registerPhysicalPath`** — which uses that map. The facade would have thrown on every call. Fix: keep the map alive through Phase 4, transfer to PM only after PM exists.
- **Phase 4 reconstructed FQN from the persisted classfile path** — `DEFAULT_PERSISTENCE_PATH.relativize(...)`. Existing tests persist to `@TempDir`, so the reconstruction would have produced junk strings. Fix: thread FQN end-to-end. `RegistryEntry` got a new `fqn` field in Phase 1; `registerPhysicalPath` got a third argument; both MVEL callers (single + batch) updated.

These weren't subtle. They came out of a careful reading because I knew the plan was about to drive eight commits and I'd be debugging anything that slipped.

## The bridge commit

The execution surfaced one technique worth keeping. Between "introduce `LambdaRuntime`" (Phase 6a) and "delete `LambdaRegistry`" (Phase 6b), I made `LambdaRegistry` a pure pass-through to `LambdaRuntime.getInstance()` for one commit. Every facade method became a thin delegate; no own state.

```java
public Path getPhysicalPath(int physicalId) {
    return LambdaRuntime.getInstance().persistenceManager()
        .artifactFor(physicalId).map(ArtifactRef::classFile).orElse(null);
}
```

Tests stayed green. Then the next commit migrated all callers to call `LambdaRuntime` directly, deleted the facade, and the compiler told me every call site I missed. Two clean commits instead of one big-bang or many small ones. Captured to the garden as a technique entry — generally useful any time a central class needs renaming with many callers.

## What the day cost

Two real Java gotchas:

- **Enum constants are constructed before static fields are initialized.** An instance-field initializer in `LambdaRegistry` that read `DEFAULT_PERSISTENCE_PATH` failed to compile. Moved the wiring into the static block — works. Garden entry.
- **`mvel3.compiler.lambda.resetOnTestStartup=true` (set globally in `pom.xml`) wipes the persistence root including the `@TempDir` parent.** Three of my new tests crashed in `Files.writeString` because the lazy init had just `Files.walk`-deleted the temp directory. Fix: override the property to `false` in each affected test setup. Also captured.

## Where I am

`lambda-registry-refactor` branch on mvel: eight commits, 733 tests green (was 722), `LambdaRegistry` gone. SNAPSHOT installed locally. Tried `mvn compile` on `drlx-parser/main` to confirm the cross-repo break — it fails on `DrlxLambdaCompiler`, `DrlxRuleBuilder`, and `DrlxCompiler` exactly as Plan 2 will fix tomorrow.

The setup was bigger than I'd estimated, but the work itself was mechanical once the spec held. Tomorrow I do Plan 2: rewrite `DrlxLambdaMetadata` (drop `physicalId`, add `classFile`), update the three DRLX files that imported `LambdaRegistry`, write D1–D7, push both repos in lockstep. Nothing in Plan 2 looked surprising during the spec review — but I said that about Plan 1's Phase 4 too.
