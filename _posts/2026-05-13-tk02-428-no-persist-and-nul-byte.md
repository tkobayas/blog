---
layout: post
title: "mvel#428 — no-persist, and a NUL byte that ate a comment"
date: 2026-05-13
type: phase-update
entry_type: note
subtype: diary
projects: [mvel3, drlx-parser]
tags: [mvel3, drlx-parser, lambda-registry, refactor, java, jls, tdd]
---

# mvel#428 — no-persist, and a NUL byte that ate a comment

Yesterday I closed Plan 2 and Phase C, then a follow-up review left two correctness items on the table: a batch-compile path that leaked global state in "no-persist" mode, and `Path.of(classFile)` throwing unchecked through both typed metadata exceptions. Today was paying that bill. Two commits on the MVEL side, one on DRLX. Both landed, both green.

## Follow-up 1 — the no-persist branch that wasn't local enough

`MVELCompiler.compileEvaluator` already branches on `LambdaRuntime.isPersistenceEnabled()` and skips the catalog and persistence-manager touch when persistence is off. `MVELBatchCompiler` didn't. Every `add(...)` call still ran `registerAndRename(...)` and `persistenceManager().artifactExists(...)` regardless of whether the batch was constructed with a `persistenceDir`. So no-persist batch mode silently leaked into the global catalog and could even short-circuit on a previously persisted artifact.

The fix should have been: top-level branch on `persistenceDir == null`, mirror what the single-compiler path does, done.

### The probe-register test

Before touching the production code, we wrote the tests:

- **M12** — after a no-persist batch, probe-register a sentinel against `LambdaRuntime.getInstance().catalog()`. Its `logicalId` must be 0. If the batch leaked any registrations, the probe would have come back with a non-zero ID instead.
- **M13** — pre-seed a persisted artifact via the single-compiler path, then run a no-persist batch on the *same* lambda. It must compile fresh (compile-invocation count goes up by one), not load from disk.

```java
LambdaKey probe = LambdaUtils.createLambdaKeyFromMethodDeclarationString(
        "public boolean test(int x) { return x > 0; }");
RegistrationResult result =
        LambdaRuntime.getInstance().catalog().register(probe);
assertThat(result.logicalId()).isZero();
```

The probe pattern is satisfying because it tests a *negative* side effect without any mocking or test-only seams in production code. Both tests went RED on the unmodified compiler exactly as expected — M12 came back with logicalId 2 (the batch's two `add()` calls), M13 with a compile count of N+1 instead of N+2 (PRE\_PERSISTED short-circuit). I'd felt slightly cheated of a real failure when I wrote them; the diagnostic clarity of those numbers made up for it.

### The fix that broke `testBatchCompileMultipleExpressions`

First attempt: in the no-persist branch, dedup by FQN. The class name is md5-derived from the source expression, so equal expressions get the same FQN — natural in-batch deduplication, no catalog needed.

Full MVEL suite came back red on one test: `testBatchCompileMultipleExpressions` had two *different* expressions collapse to one source. Log said `Batch-compiling 1 lambda sources` when it should have said 2. Two different expressions, one FQN.

`CompilationUnitGenerator.createMapEvaluatorUnit` returns `input.getUnit()` directly — no `renameTemplateClass(...)` call. Map mode keeps the template class name (`GeneratorEvaluator__`) regardless of the expression. The reason the existing persistence path worked was `registerAndRename` appended `_<physicalId>` to make them unique. The moment I dropped that rename, the FQN became a collision key for anything Map-mode.

The cleaner fix was already there in shape: keep the rename, just don't register against the *global* catalog. Give the batch its own `LambdaCatalog` instance when `persistenceDir == null`:

```java
public MVELBatchCompiler(ClassManager classManager, Path persistenceDir) {
    this.classManager = classManager;
    this.persistenceDir = persistenceDir;
    this.localCatalog = persistenceDir == null ? new LambdaCatalog() : null;
}
```

Then pass that catalog into a new overload of `registerAndRename`:

```java
static LambdaRegistration registerAndRename(
        CompilationUnit unit, String currentFqn, LambdaCatalog catalog) {
    // ...
    int physicalId = catalog.register(lambdaKey).physicalId();
    // ...
}
```

Now the no-persist path goes through identical rename logic with a private catalog. M12 / M13 stayed green. The Map-mode test came back. Tests: 736 MVEL, 182 DRLX, no regressions.

## Follow-up 2 — `Path.of(classFile)` and JLS §3.3

The second fix is mechanical: both `LambdaRegistryStore.load(...)` and `DrlxLambdaMetadata.load(...)` call `Path.of(classFile)` directly while parsing persisted properties. If the persisted classFile contains a character the local filesystem rejects (NUL on Unix, certain reserved chars on Windows), `Path.of` throws unchecked `InvalidPathException`. That bypasses `InvalidLambdaRegistryException` on the MVEL side and `InvalidDrlxLambdaMetadataException` on the DRLX side, and on DRLX it also bypasses the `DrlxMetadataMismatchMode` routing consumers rely on for malformed metadata.

The fix is one try/catch on each side:

```java
Path classFilePath;
try {
    classFilePath = Path.of(classFile);
} catch (InvalidPathException e) {
    throw new InvalidLambdaRegistryException(
            "Invalid classFile path for artifact physicalId "
                    + physicalId + ": " + classFile, e);
}
```

The test, on both sides, writes a properties file containing a NUL byte in the `classFile` value — encoded in the file as the Unicode escape `\u0000`, which `Properties.load` decodes faithfully. Then it asserts the typed exception comes back. Two tests, both RED on current code (InvalidPathException leaking), both GREEN after the wrap.

### The test file ate the comment

While *writing* the MVEL test, I put a comment explaining what the `\u0000` in the property file did:

```java
// \u0000 in the Properties file decodes to a NUL byte
```

The Edit tool then refused to do its next search because the file didn't match what it had just stored. The bytes on disk had a literal NUL where my source had `\u0000`. The Java lexer had eaten my comment.

JLS §3.3 says Unicode escapes are processed by the lexer *before* comment recognition. A `\u0000` in a `//` comment becomes a NUL byte in the source the same way `\u0000` in a string literal becomes a NUL character in the string — except the comment isn't even meant to be parsed. The neutralising rule is also in §3.3: a `\u` preceded by an odd number of backslashes (e.g. `\\u0000`) is not an escape. So the comment ended up as:

```java
// The \\u0000 escape in the Properties file decodes to a NUL byte
```

This is one of those JLS facts you can write Java for ten years without ever bumping into. It also explains the classic puzzle:

```java
// \u000A System.out.println("This actually runs");
```

which compiles and prints, because the `\u000A` is a newline before the lexer notices the `//`.

Submitted that one to the cross-project knowledge garden — high confidence anyone documenting Unicode handling in Java tests will run into it eventually. The probe-register pattern went in too.

## What landed

| Repo | Branch | Commit | Tests |
|------|--------|--------|-------|
| MVEL | `lambda-registry-refactor-followup` → `main` via #430 | `70d9569` (merged) | 735 → 736 |
| MVEL | `lambda-registry-refactor-followup2` | `32bdb97` | 736 → 737 |
| DRLX | `main` | `46f7d2f` | 181 → 182 |

The "follow-up review" loop is now what I want it to be: PR review catches integration-edge correctness, a tight TDD pass closes each finding, both sides land in lockstep. Compared to yesterday's five-commit Plan 2 marathon, today felt like the system working as designed.
