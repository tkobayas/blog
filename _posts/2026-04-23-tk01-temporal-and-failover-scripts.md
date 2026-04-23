---
layout: post
title: "Temporal and failover load tests (and fixing once_within)"
date: 2026-04-23
type: phase-update
entry_type: note
subtype: diary
projects: [drools-ansible-rulebook-integration-2.0.x]
tags: [load-testing, drools, ha, temporal, failover]
---

Picking up from the 2026-04-22 handover, four scripts were still parked in `main/` that hadn't been ported to the new `-load-tests` module: `load_test_90kb.sh`, `load_test_temporal.sh`, `load_test_ha_compare.sh`, `load_test_failover.sh`. The question wasn't "how to port all four" but "which of them are worth porting at all".

## What to drop, what to port

`load_test_90kb.sh` runs the same match/unmatch shape as the 24KB tests but with a 90KB payload to exaggerate HA's per-event JSON-copy overhead. Useful signal, but not different enough from the existing 24KB runs to justify the extra payload variants. Cut.

`load_test_ha_compare.sh` compares the same workload across noHA / HA-H2 / HA-PG. The H2 path would need a whole H2 backend plus file-cleanup in the Java runners — significant surface for a mode we don't actually deploy. The noHA vs HA-PG half is already covered by `load_test_match_unmatch_noHA_HA-PG.sh`. Cut.

That left `load_test_temporal.sh` (HA persistence cost under temporal-operator rules) and `load_test_failover.sh` (timing `enableLeader()` recovery in a cold-start JVM against prior-session PG state). Both HA-PG only — noHA is structurally meaningless for either, there's no DB to persist to.

## Meta.uuid doesn't group anything

The legacy temporal test uses a rule that groups by `event.meta.uuid` with a 60-second `once_within` window. The old docstring claimed "every event lands in its own pending window". That claim was correct — and it was exactly why the test was broken.

Before I'd worked that out, Claude had gone to grep the upstream Drools source tree for a `GenericEventSource` class that might explain automatic per-event uuid injection. That class doesn't exist in Drools — events come from the Python client, and per-event uuids are assigned higher up the stack. Which is the whole answer: if every event has a unique `meta.uuid`, you can't group by it and get more than one event per group.

`once_within` fires on the first event per group and suppresses subsequent events in the same group for N seconds. A group of one has nothing to suppress. The rule fires on every event, the window does nothing.

What the legacy test was observing — `MATCHING_EVENT` rows accumulating under load — wasn't temporal-suppression state. It was an artefact of broken grouping creating a new group per event.

## Sizing the new test

The fix is a common `group_id` field with a small number of groups. I picked 10. Per-size, each group holds `size / 10` events; at 1000 events, 100 per group. MATCHING_EVENT row count stays flat at 10 regardless of size — what scales is per-event HA-write overhead, which is the cost signal we actually want.

The payload uses Drools-Ansible's 10-template trick:

```java
throttle.put("group_by_attributes", List.of("event.group_id"));
throttle.put("once_within", "60 seconds");
// payload array has 10 entries, one per group_id; repeat_count = N/10
```

`Payload.parsePayload` expands this into `[T0,T1,...,T9, T0,T1,...,T9, ...]` totalling N events. The round-robin interleaving is free — the existing code does it.

Smoke confirmed the fixed-count design: MATCHING = 10 at every size (100, 500, 1k).

## Failover as two JVMs

The failover script runs two JVMs sharing one Postgres. Phase 1 loads events under a per-size HA UUID (using the existing retention payloads — 2-condition join rule, partial matches retained in the DB). Phase 2 cold-starts a new JVM, `initializeHA` with the same UUID, `createRuleset` (which rebuilds the KieSession from persisted state), and times `engine.enableLeader()` — that call is the recovery measurement.

Phase 2 doesn't execute a payload and skips `OutcomeCheck` entirely. The new runner class was small:

```java
Measurement.TimedResult t = Measurement.timeWork(() -> {
    engine.enableLeader();
    return Payload.Execution.empty();
});
```

`Payload.Execution.empty()` is a one-line static factory I added so the recovery path could reuse `Measurement.timeWork(Supplier<Payload.Execution>)` without a second timing primitive. A little awkward — returning a value you don't care about just to satisfy the Supplier's type — but cleaner than duplicating the GC-dance logic in a `timeVoidWork(Runnable)` overload.

`MetricReporter` gained a `failoverRecovery` boolean that appends ` (failover-recovery)` to the metric line:

```
retention_100_events.json (HA-PG) (failover-recovery), 3000000, 50
```

The bash/Java contract — three CSV fields split on comma — stays intact. `fmt_parse_metrics`'s `grep "^${file}"` matches both load and recovery lines without modification.

## Recovery is cheaper than you'd think

Smoke output from the three sizes:

```
Size     Load(ms)   Recovery(ms)    Ratio
------------------------------------------
100          2649            196     7.4%
500         39937            464     1.2%
1k         148718            797     0.5%
```

Load scales linearly with events — roughly 150ms per event at 1k. The 2-condition join means each insert references a growing partial-match set. Recovery doesn't follow: 196ms → 464ms → 797ms for the same 10× growth in event count that took load from 2.6s to 149s.

The ratio drops because recovery is bounded by "read the accumulated state once" while load pays the per-event write cost. A bigger state does add to recovery time, but not linearly with event count.

## Two smaller things

While adding the new once_within files, I switched `PayloadGenerator` to emit pretty-printed JSON globally. The existing 13 files got reformatted in the same pass. To prove the reformat was semantically zero-op I ran each file through `json.dumps(..., sort_keys=True)` before and after, diffed the canonical forms, got empty output. 16 files, byte-different on disk, semantically identical.

After smoke passed I noticed `load_test_match_unmatch_noHA-PGHA.sh` read badly. `noHA-PGHA` is one token to a scanner — the separator fuses into the mode names. `noHA_HA-PG` reads as `{noHA, HA-PG}`: underscore as the set boundary, hyphen only inside `HA-PG`. Renamed both combined scripts and their output files. Git tracked them as 87% / 96% similarity renames.

## What I'd change

Catching the `meta.uuid` bug needed the brainstorming step to pause on "what does this rule actually test?" — if I hadn't paused there, the port would have produced another broken test with a more polished wrapper around it. The legacy test's documented behaviour ("every event lands in its own pending window") was factually right but semantically degenerate, and the wording didn't flag that. A spec-writing habit to add: for every ported test, write one sentence on what the test is *supposed* to measure before writing the port.

The noHA-PGHA name lasted one day in production because nobody read it out loud. I caught it the second time I looked at the filename after smoke. Should have caught it on the rename commit the day before.

Branch is 18 commits ahead of `2.0.x`, pushed to both remotes. No PR yet.
