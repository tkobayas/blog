---
layout: post
title: "and/or — and the AND that ate `&&`"
date: 2026-04-23
type: phase-update
entry_type: note
subtype: diary
projects: [drlx-parser]
tags: [drlx, antlr, drools, group-elements, and, or, logic-transformer, lexer]
---

#11 — `and(...)` and `or(...)`, plus letting NOT/EXISTS contain nested
CEs — landed this morning. 14 commits, 73 → 84 tests passing. The
feature itself was mostly as the spec promised: the same group-element
shape as NOT/EXISTS, minus the single-child wrap (since Drools'
`GroupElement.addChild` only enforces one-child on NOT and EXISTS;
AND/OR accept N natively). The DRLXXXX spec calls for an explicit
top-level `and(...)`, an `or(...)` that can nest, and full symmetry
between NOT/EXISTS and AND/OR — so children of any of the four should
be a unified `groupChild` that can be an oopath or another CE.

What stole an hour was a lexer collision I walked into with my eyes
closed.

## Scope, locked fast

Before any code I had three questions:

1. **How much of DRLXXXX §16 is in scope?** The spec there also has
   bound patterns (`and(var l : /locations, ...)`) and a passive
   `?and(...)` marker. I picked option B: accept nested CEs as
   children, skip bindings and passive for a follow-up. That's the
   interesting capability (`or(and(...), and(...))`) without
   inflating the grammar change.

2. **Do NOT/EXISTS also get nested CEs in this PR?** The DRLXXXX
   spec says "'not' and 'exists' follow the same rules as 'and'
   'or'." Claude confirmed Drools' `DRL10Parser.g4` has treated
   `not(lhsExpression)` and `exists(lhsExpression)` this way for
   years, and `GroupElement.pack()` has specific logic for
   `NOT(EXISTS(...))` → unwrap and `EXISTS(NOT(...))` → rewrite as
   NOT. So nested CEs inside NOT/EXISTS aren't speculative runtime
   territory; they're load-bearing in Drools. Option B2: symmetric.

3. **Grammar shape — where does the comma live?** Current
   `notElement` bakes the trailing `,` right into its rule. When
   NOT/EXISTS gets nested inside another CE, the inner form
   shouldn't carry that `,` — the outer group separates children
   with its own. Two choices: duplicate each CE rule into "top-level"
   and "nested" variants, or lift the `,` up to `ruleItem`. I picked
   γ (lift the comma). One source of truth, and the NOT/EXISTS rules
   cleanly widen to use a new `groupChild` non-terminal shared with
   AND/OR.

## The AND that ate `&&`

Task 1 was meant to be trivial: add `AND : 'and';` and `OR : 'or';`
to `DrlxLexer.g4`. Five lines in, I committed it, the build was
green, moved on to Task 2.

Task 2 (the grammar refactor) compiled clean on the first attempt.
But then this happened:

```
error(126): org/drools/drlx/parser/DrlxParser.g4:129:23:
  cannot create implicit token for string literal in
  non-combined grammar: '&&'
error(126): org/drools/drlx/parser/DrlxParser.g4:131:23:
  cannot create implicit token for string literal in
  non-combined grammar: '||'
error(126): org/drools/drlx/parser/DrlxParser.g4:723:57: '&&'
error(126): org/drools/drlx/parser/DrlxParser.g4:724:21: '&&'
```

Four errors, all about `&&` and `||` — operators I never touched. The
line numbers didn't even exist in my file (`DrlxParser.g4` has 134
lines; line 723 is in the imported `JavaParser.g4`). The errors
appeared *after* the clean compile. My first instinct was stale
`.tokens` cache — the gotcha from last session — but `mvn clean
install` didn't help.

Then Claude did the right grep:

```
JavaLexer.g4:166: AND      : '&&';
JavaLexer.g4:167: OR       : '||';
```

There it was. `JavaLexer`, imported transitively through `Mvel3Lexer`,
had already defined `AND` as the token for `&&` and `OR` as the token
for `||`. My new `DrlxLexer.g4` declarations —

```antlr
AND    : 'and';
OR     : 'or';
```

— didn't add new tokens. They *overrode* the existing ones. And since
DRLX's parser grammar is non-combined, implicit tokens for string
literals like `'&&'` have to resolve to an explicit lexer token —
which no longer existed. Everywhere Mvel3Parser or JavaParser
referenced `'&&'` or `'||'`, ANTLR refused to create a token for it.

The spec had warned me about this, in prose:

> `and` / `or` lexer tokens — verify these aren't already reserved
> or in use for another purpose.

I'd read that line and thought: "Sure, `and` isn't a Java keyword,
and `not`/`exists` worked, so `and`/`or` should be fine." I didn't
look at *what names* were already in use. `AND` was in use. Not for
`and` the word — for `&&` the operator.

## Fix: align with the DRL convention

The rename is localised. I asked which suffix to use — `AND_KW`?
`KW_AND`? — and you pointed at Drools' `DRL10Lexer.g4`, which uses
`DRL_AND`, `DRL_OR`, `DRL_NOT`, etc. Consistent with that convention
for DRLX: `DRLX_AND` and `DRLX_OR`. Token names only differ; the
string literals `'and'` and `'or'` still match the same lexemes. One
fix commit between Task 1 and Task 2 — no amend, per the repo's
"always a new commit" rule:

```
fix(lexer): rename AND/OR to DRLX_AND/DRLX_OR
```

After that, clean build, all 73 existing tests still green, and I
carried on.

## The compile-order tax I didn't budget

The plan's Task 6 Step 8 says "Run existing tests — 73 pass." In
reality that step couldn't run, because the *LSP* visitor
(`TolerantDrlxToJavaParserVisitor`) also called `ctx.oopathExpression(0)`
on the NOT/EXISTS paren form, and the grammar refactor had moved
paren-form children under a new `groupChild` non-terminal. The
tolerant visitor wouldn't compile until Task 9 rewrote it.

So Tasks 2 through 8 all committed with the build in a state that
compiled the *Rule-AST* visitor fine but had two persistent errors
in the *LSP* visitor. Each task ended with a check of the form "same
2 LSP errors, no new ones" — the useful green bar, given the ordering.
Tests finally ran at Task 9, green at 73. Not a bug in the plan per
se — more a missed dependency. I noted it in the checkpoint summary
rather than revising the plan mid-execution.

## OR actually splits rules

`OrTest.orBothBranchesFire` inserts a senior and a junior, with a
top-level `or(/seniors[age>60], /juniors[age<18])`:

```
assertThat(kieSession.fireAllRules()).isEqualTo(2);
```

It fires twice. I'd written that assertion half-expecting to rewrite
it once I saw the actual number — but `LogicTransformer` (Drools'
compile-time rewrite step) really does expand a top-level OR into
two internal rule instances, and each fires once for its satisfying
branch. I didn't have to do anything for it; the parser emits one
`GroupElement(OR)` at the root, Drools splits it downstream. The
test is more a contract check on Drools than on our parser — but
it's the kind of behaviour a future refactor might quietly break,
so it earns its place.

## Nested NOT/EXISTS actually works too

`NestedGroupTest.notContainingOr` was the one I was least sure
about: `not(or(/persons[name=="Alice"], /persons[name=="Bob"]))` —
DeMorgan in the LHS. Empty session: rule fires once (NOT vacuously
true). Insert Charlie (name doesn't match either literal): still
vacuously true, but the match has already fired, so no new activation.
Insert Alice: OR becomes satisfied, NOT becomes false. No fire. That
sequence passed first try. `GroupElement.pack()` is doing quiet work
for us; we don't have to understand the details beyond "emit the
literal tree."

## What landed

- 14 commits on `main` (range `bb6f3c4..184daaa`).
- 4 `AndTest` + 4 `OrTest` + 3 `NestedGroupTest` = 11 new tests.
  73 → 84 passing. Zero regressions on existing NOT/EXISTS coverage.
- `GroupElementIR.Kind` now `{NOT, EXISTS, AND, OR}`.
  `GROUP_ELEMENT_KIND_{AND,OR} = 3,4` in the proto (previously
  reserved by comment — the plugin-managed regen from last session
  made this a pure `.proto` edit).
- Grammar now has a single `groupChild` non-terminal used by all
  four CE paren forms, with `,` owned by `ruleItem`. Adding the
  next CE — accumulate, collect, whatever comes — is one production
  and one arm in each visitor.

What's deliberately still out: bound patterns as group children
(`and(var l : /locations, ...)`), passive `?and`, and any LogicTransformer
behaviour assertions. The bindings piece is the biggest one — it
meaningfully widens `groupChild` to accept `rulePattern` shape, and
it's the gate to DRLXXXX §16 feeling "done." That's the next ticket.
