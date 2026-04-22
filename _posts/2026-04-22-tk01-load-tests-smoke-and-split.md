---
layout: post
title: "Load-tests module: a smoke test and a cost-profile split"
date: 2026-04-22
type: phase-update
entry_type: note
subtype: diary
projects: [drools-ansible-rulebook-integration-2.0.x]
tags: [load-testing, drools, ha, debugging, bash]
---

Two sessions ago I cut a `reorganize-load-test` branch and started moving load-test infrastructure out of `drools-ansible-rulebook-integration-main/` into a greenfield Maven module. The motivation was isolation — `main/Main.java` had organically grown into a CLI-plus-generator-plus-harness doing four jobs badly. This session finished the plan and then took one pivot I didn't see coming.

## Where I came in

HANDOFF.md said Tasks 1–7 of the plan were committed and Task 8 (`PayloadGenerator` + 11 generated JSON resources) was mid-flight. The generator had been run once; the JSONs were written but uncommitted. Last session's smoke test — the thing that would confirm the generator produced a RuleSet the engine actually accepts — was interrupted before I could check it.

Picking up there was meant to be a 30-second thing. Rebuild the fat-jar, run it against `24kb_1k_events.json`, read the stderr metric line. Then commit and move on.

That's not what happened.

## The discard trap

The smoke test failed loudly. Session stats said the engine matched every single event — `rulesTriggered=1000, eventsMatched=1000` — but `OutcomeCheck` threw:

```
Exception in thread "main" java.lang.RuntimeException: Expected at least one match but got 0 (events: 24kb_1k_events.json)
```

Claude caught the root cause in one pass: `Payload.java` had been copied byte-identically from `main/` per the plan's Task 3. The copy preserves the legacy `discard_matched_events` shortcut — when that flag is true, match results aren't accumulated in the returned list. The new `OutcomeCheck.verify(List<Map>, …)` signature was inspecting that list and seeing empty when it should have seen 1000 matches.

The generator had been asked to produce `discard_matched_events: true` for every match scenario. That choice is load-bearing: at 1m events, accumulating a million match records client-side OOMs a 512MB heap. Flipping the flag wasn't an option.

Claude surfaced four:

| | Approach | Trade-off |
|---|---|---|
| A | Thread an `int matchCount` counter through `PayloadRunner → Payload.Execution → TimedResult` and change `OutcomeCheck.verify(int, …)` | Cleanest; 6 files + one test |
| B | Remove the discard guard in `PayloadRunner` | OOMs at 1m events |
| C | Generator emits `discard_matched_events: false` | Same OOM |
| D | Parse `engine.sessionStats(id)` JSON for `eventsMatched` | Brittle stats-format coupling |

I picked A. The list was never the right authority for "did matches occur" — the `discard` flag is specifically designed to drop the list. A counter that increments regardless of the flag matches the actual intent.

The fix was five files in `loadtests/` (`Payload.java` gained an `Execution` wrapper, `Measurement.TimedResult` gained `matchCount`, `OutcomeCheck.verify` changed signature, runners passed the count, test updated). Commit `b6f1b338`, followed by Task 8 proper (`47fc7636`, the generator + 11 JSONs).

Tasks 9–12 then slotted in without drama: port `MemoryLeakAnalyzer` with four-group HA-aware analysis, write `lib/common.sh` with prefixed helpers (`require_*`, `pg_*`, `jvm_*`, `fmt_*`), the four shell scripts, and an E2E smoke covering match + unmatch + retention against noHA and Docker-Postgres HA-PG. All three smokes clean, no exceptions in any log.

## The cost-profile split

Then I ran `load_test_all.sh`. It drives 16 runs — 4 sizes × {match, unmatch} × {noHA, HA-PG}. That was the original design.

100k HA-PG was punishingly slow. I didn't wait for 1m HA-PG. This isn't a performance bug — HA-PG does real JSON persistence to Postgres per event, and at high event counts it's minutes of wall-clock per run. The "all-in-one" script was making the whole thing unusable for interactive iteration.

I asked Claude whether splitting by cost profile made sense. We settled on two scripts:

- `load_test_match_unmatch_noHA.sh` — 4 sizes × {match, unmatch} × noHA = 8 runs. No Docker dependency; noHA is fast at every size. Mirrors the legacy `main/load_test_all.sh` shape.
- `load_test_match_unmatch_HA.sh` — 3 sizes (`1k`, `5k`, `10k`) × {match, unmatch} × {noHA, HA-PG} = 12 runs. Needs Docker for Postgres. Cap at 10k because 100k HA-PG is the thing we were running from.

The 3 sizes in the HA script weren't 1k/10k/100k — I chose 1k/5k/10k deliberately. The `MemoryLeakAnalyzer`'s consecutive-acceleration check needs three data points per group to fire (two ratios). Two data points only gets you absolute-increase and total-increase thresholds. A 5k middle point keeps the analyzer's full signal intact without dragging runtime into the minutes.

Adding a 5k variant meant a new generator size and a new `24kb_5k_*.json` pair. The nice surprise: regenerating all 11 existing JSONs with the updated generator produced byte-identical output. Fixed random seed + `LinkedHashMap` key ordering + deterministic message-building algorithm. Before the regen I sha256'd the existing files; after, all 11 hashes matched. Git saw exactly two new files.

`MemoryLeakAnalyzer.extractEventCount` gained a `5k_` branch. `PayloadGenerator.sizeToRepeatCount` gained a `"5k" → 5_000` case. Two commits — `198c77db` for the 5k variant, `364cb289` for the script reorg. Git detected the second as a rename from `load_test_all.sh → load_test_match_unmatch_HA.sh` (74% similarity), which is accurate — the HA script is structurally the all.sh pattern minus two sizes.

## What I'd change

The `OutcomeCheck` bug was preventable if I'd thought harder about list semantics during the plan-writing phase. The design doc wrote `OutcomeCheck.verify(List<Map>, …)` assuming the list was authoritative. The `Payload` copy intentionally empties that list under memory pressure. Those two designs couldn't coexist and nobody noticed until Task 8 smoke.

The script split wasn't visible from the plan because the plan was written with a performance assumption that didn't survive first contact — HA-PG being ~10-100× slower than noHA at the same event count isn't the kind of thing you cost-out on paper. You run it, you feel it, you split. Writing a perf-budget line into the spec next time would have flagged "the 16-run script will be dominated by HA-PG at 100k+" before we shipped it.

Branch pushed. Scripts tiered by cost. The `result_*.txt` files and `out_*.log` for the three E2E runs are on disk locally; no PR opened yet.
