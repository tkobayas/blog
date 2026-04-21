---
layout: post
title: "Rule annotations — two, not twelve"
date: 2026-04-20
type: phase-update
entry_type: note
subtype: diary
projects: [drlx-parser]
tags: [drlx, annotations, language-design]
---

DRLX's rule-attribute support was on hold. The spec — `docs/DRLXXXX.md` — had
one worked example, `@Salience(10)`, and a single line deferring the rest:
"Supported attributes are to be determined." Implementation was trivial; scope
was not.

The classic DRL7 list is a dozen-plus attributes — `@NoLoop`, `@AgendaGroup`,
`@RuleflowGroup`, timers, and more — all legitimately useful, none named in
DRLXXXX.md. Porting them forward by default is exactly the kind of decision
that accumulates cruft in a nominally-greenfield language.

I picked the minimum. `@Salience(int)` for firing priority, `@Description(String)`
for metadata. Everything else throws `"unsupported DRLX rule annotation"` at
parse time. The whitelist is easy to extend deliberately once the spec designer
weighs in; hard to contract once rule authors depend on something.

## Real classes, strict imports

DRLXXXX.md says rule attributes "Need to be imported." That means real Java types
— a new `org.drools.drlx.annotations` package with two `@interface` declarations,
`@Target(TYPE)` (a DRLX `rule` is class-like) and `@Retention(RUNTIME)` for
future tooling.

Resolution is strict. `@Salience(10)` without a matching import throws
`"unresolved annotation '@Salience' at line:col — missing import?"`. Fully-qualified
`@org.drools.drlx.annotations.Salience(10)` works without an import. Exactly
how `javac` resolves annotation types — the DRLX author's mental model borrows
directly from Java.

## `Integer.parseInt` as a literal validator

First-pass scope rejects expression-based salience — only a plain int literal
is allowed. The rejection logic is one line:

```java
String text = annCtx.elementValue().getText();
try {
    return String.valueOf(Integer.parseInt(text));
} catch (NumberFormatException e) {
    throw new RuntimeException(
        "@Salience expects int literal, got '" + text + "' at " + line + ":" + col);
}
```

`Integer.parseInt` rejects `"ten"` (quotes in the text), `1 + 2` (not numeric),
`10L` (trailing L), `0xA` (hex), `p.age` (identifier) — all as
`NumberFormatException`. Signed forms `-5` and `+7` are accepted because
`Integer.parseInt` already handles them. Five rejection paths, one validator,
no AST walking.

## Claude's commit-message autopilot

The spec commit went in with `Refs #2` in the footer — except issue #2 was a
closed issue about test reshuffling, not this work. Claude had guessed the next
number without checking. I caught it in the commit log.

One-line amend, not a real problem. But a useful reminder: Claude will fabricate
a plausible-looking identifier if nothing forces verification. Footer refs to
issues, PRs, line numbers — all easy to get wrong on autopilot. The guard is
one `gh issue list` before every commit. The project's workflow now creates
the issue *before* the commit, not after.

---

Nine tests landed. Test count 40 → 49. Out of scope until DRLXXXX.md gets more
specific: classic DRL7 attributes, expression-based salience, unit- and
query-level annotations. Next candidate is bigger — `GroupElement` infrastructure
for `not`/`exists`/passive patterns. Refactor first, feature second.
