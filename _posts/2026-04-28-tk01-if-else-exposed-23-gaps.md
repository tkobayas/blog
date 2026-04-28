---
layout: post
title: "if/else exposed #23's gaps"
date: 2026-04-28
type: phase-update
entry_type: note
subtype: diary
projects: [drlx-parser]
tags: [if-else, eval, mvel3, drools, runtime]
---

# if/else exposed #23's gaps

Started #12 (if/else, Form A) on the assumption #23's bridging was solid. Yesterday's
entry closed with "no risky bridging code remaining." The visitor work was easy.
The runtime wasn't.

## The plan was stale before I read it

Before touching code, Claude reviewed the plan against the current codebase and flagged
five separate references that wouldn't compile verbatim: IR types are nested in
`DrlxRuleAstModel` (not standalone), the identifier helper is `extractIdentifiers` not
`referencedBindings`, the parser entry must be `drlxCompilationUnit()` not `compilationUnit()`
(this gotcha was already in HANDOFF.md from #23 but didn't propagate to the plan),
the test pattern uses `new DrlxToRuleAstVisitor(tokens).visitDrlxCompilationUnit(ctx)`
with no `parseSingleRule` shortcut, and the IR-shape assertions needed `contains(...)`
not `isEqualTo(...)` because expression text preserves whitespace through the token
stream.

A real architectural decision also surfaced. The plan had `branchBody : '{' ruleItem* '}'`,
but `rulePattern : boundOopath ','` requires a trailing `,` at the rule level. A
single-pattern branch like `if (...) { var s : /seniors[...] }` wouldn't parse — the
pattern needs a comma it doesn't have. I chose to introduce a separate `branchItem`
production with comma-as-separator (no trailing) so single-item branches feel natural
inside braces. That side-stepped the issue cleanly.

## Five gaps in cascade

Tasks 1–3 (grammar + visitor + empty-branch test) went green in three commits. Task 4
(binary if/else end-to-end) hit a wall, then five walls in sequence as each fix
exposed the next layer.

**Gap 1 — eval expressions had no imports.** First failure: `Unsolved symbol: LOW`.
The `EvalIR` for `c.creditRating == Rating.LOW` got compiled with `MVEL.map(decls)`,
no `.imports(...)` call at all. #23 hadn't exposed this because every `test` example
used primitive comparisons (`p.age > 30`) — no enum constants, no external types.
Pattern compilation went through `MVEL.pojo(patternType, ...)` which gave MVEL access
to the pattern type's classloader, so primitives worked there too.

**Gap 2 — patterns had the same gap.** Fixed gap 1 by plumbing
`addImports(parseResult.imports())` into `DrlxLambdaCompiler` and threading
`.imports(new HashSet<>(imports))` through the eval and consequence batch builders.
Re-ran. New failure: `Unsolved symbol: HIGH`. Same problem in the pattern path —
`MVEL.pojo(patternType, ...)` doesn't auto-import same-package types either. Added
`.imports(...)` to both pattern paths. Three call sites total. The consequence path
was already passing `.imports(new HashSet<>())` — empty set. So consequences had the
same latent gap.

**Gap 3 — `getTypeMap` walked only direct children.** Imports fixed, next failure:
`Unsolved symbol in p: Solving p`. The consequence `do { System.out.println(c + " " + p); }`
couldn't find `p` because `p` lived inside `OR(AND(EvalIR, Pattern), AND(EvalIR, Pattern))`,
and `getTypeMap` only walked the root AND's direct children. Same-name bindings in two
OR branches need to unify into one entry; nested patterns need to be visible. Made
`collectPatternTypes` recursive.

**Gap 4 — MVEL3 won't bean-rewrite inside `!(...)`.** Compiles started succeeding for
the positive guard, but the else-branch failed with `creditRating has private access in
org.drools.drlx.domain.Customer`. `c.creditRating` was being compiled as direct field
access instead of `c.getCreditRating()`. Isolation test confirmed it: a plain
`test !(c.creditRating == Rating.LOW)` fails the same way. A plain
`test c.creditRating == Rating.LOW` works fine. Wrapping property access in `!(...)`
breaks the bean-getter rewrite somewhere in MVEL3's transpiler.

The cleanest workaround: change the visitor's negation from `"!(" + cond + ")"` to
`"(" + cond + ") == false"`. Same semantics, different parser path that keeps property
access at the top level. Tested in isolation (`test (c.creditRating == Rating.LOW) == false`)
— passes. Updated the visitor and the IR-shape assertions in one go.

```java
// Use `(cond) == false` rather than `!(cond)` to side-step an MVEL3
// limitation: property access inside `!(...)` doesn't get rewritten to
// a getter call, which fails for private fields. The `== false` form
// keeps property access at the top level and compiles correctly.
String negated = "(" + prior + ") == false";
andChildren.add(new EvalIR(negated, extractIdentifiers(negated)));
```

**Gap 5 — clone() NPE during OR expansion.** Compiles green, runtime fails:
`Cannot load from object array because "this.declarations" is null` from
`LogicTransformer$AndOrTransformation.transform`. Drools clones constraints when it
expands `OR(AND(...), AND(...))` into sibling sub-rules. `DrlxLambdaConstraint.clone()`
called the eager-init constructor with `this.declarations`, but on the deferred-compile
path declarations is null until `bindEvaluator` fires. Same in `DrlxLambdaBetaConstraint`.
The fix is to use the pre-compiled-evaluator constructor instead — the evaluator is
already bound, MVEL3 evaluators are stateless, sharing the reference is safe.

## Scope expansion was the right call

After gap 1, I asked the user to choose: (a) plumb imports through properly as part of
#12, or (b) work around in test rules with primitive comparisons. The plan didn't
anticipate this work. (a) widens scope; (b) ships #12 narrower but defers the gap.
The user picked (a). Without that decision, gaps 2 and the consequence-path latent gap
would have stayed buried.

The same call applies to gap 5. The clone bug only manifests when `OR(AND(...))`
appears in the LHS — which Form A is the first feature to produce. Without #12, no
test would have hit it. The fix doesn't change behaviour for any existing test; it
just stops a NPE that was waiting for the right shape.

These are #23-era oversights, but they were invisible until a feature with a richer
surface came along to exercise them. Calling them "#23 bugs" feels like blame
displacement — they were correct for what #23's tests covered. They weren't correct
for everything `EvalIR` would eventually be asked to do.

## End state

Tasks 5–9 went green without further changes — else-if chains, no-final-else,
same-name binding flow, multi-element branches, nested CEs, nested if/else,
property-reactivity. Sixteen new tests, full project suite at 141/0 (was 125).

Pushed nine project-repo commits to `origin/main` (`511696c..8f37d23`) plus one
workspace-spec commit. #12 closed on GitHub with a summary of the gap fixes. Form B
(per-branch consequences) still tracked in #22.
