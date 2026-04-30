---
layout: post
title: "benchmark unit import — the class was there, the import wasn't"
date: 2026-04-30
type: phase-update
entry_type: note
subtype: diary
projects: [drlx-parser]
tags: [benchmarks, jmh, ruleunit]
---

# benchmark unit import — the class was there, the import wasn't

The benchmarks worked when they were first written. Then they stopped. The error was specific: `"Unit class 'MyUnit' not found — add import <fqn>.MyUnit; to the DRLX source."` The generated DRLX had `unit MyUnit;` but no import. Claude and I fixed it this afternoon.

## The generation methods were right there in the benchmark class

`KieBaseBuildNoPersistenceBenchmark` had eight static methods at the bottom — `generateDrlAlpha`, `generateDrlxAlpha`, `generateDrlJoin`, `generateDrlxJoin`, and four more covering multiJoin and multiAlpha patterns. Each DRLX variant built up a string with package declaration, domain import, and `unit MyUnit;` followed by rule blocks.

Missing: `import org.drools.drlx.ruleunit.MyUnit;`

The test module had `MyUnit` at `org.drools.drlx.ruleunit.MyUnit` — a simple class with four `DataStore<Person>` fields. The benchmark module had no unit class. The generated source declared the unit but never imported it. Runtime blew up when the compiler tried to resolve it.

## Extract, create, fix

We created two files:

**`DrlxSourceGenerator`** — all eight generation methods moved here. Made the class final, private constructor, public static entry points for `generateDrl` and `generateDrlx` that dispatch to the pattern-specific methods. Five benchmark classes updated to call `DrlxSourceGenerator.generateDrlx(count, ruleType)` instead of inline methods.

**`MyUnit`** — copied the test version to the benchmark module's `org.drools.drlx.ruleunit` package. Just the four `DataStore<Person>` fields matching the entry points used in generated rules: `persons`, `persons1`, `persons2`, `persons3`.

Then the fix: every `generateDrlx*` method got one line added after the domain import:

```java
sb.append("import org.drools.drlx.ruleunit.MyUnit;\n\n");
```

## 146 tests, PreBuildRunner clean

Compiled the benchmark module. Ran core tests: 146/146 green (141 regular, 5 no-persist). Ran `PreBuildRunner` separately to verify the pre-build phase works end-to-end — it generates DRLX source, compiles it through `DrlxCompiler.preBuild`, writes metadata and .class files to a temp directory. Finished clean. The kjar output step completed too.

The benchmarks are runnable again. The missing import is in. The generation logic is centralized instead of duplicated across five classes.
