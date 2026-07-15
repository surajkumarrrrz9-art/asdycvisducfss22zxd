# MD-9 Durable Candle Cache — Project Context & Continuity

**Read this entire document before doing anything else.** This is the single continuity thread for a project being carried across many separate Claude sessions, because usage limits mean no one session sees it from start to finish. It is written chronologically. Everything before "Where things stand right now" is history — read it to understand *why* things are built the way they are, not just what to do next. Everything after that section is the actual instruction for whoever is reading this now. When you finish a piece of work, append a new dated section at the end and update "Where things stand right now" so the next session isn't left guessing.

This document is orientation, not a substitute for the actual files it references. All of the numbered plan/task/audit files named below should also be in this Drive folder — read the specific file relevant to your task in full; don't work from this summary alone.

---

## The app, and what was actually being asked for

The app is a full-stack, end-to-end Node.js / React / Vite financial platform — referred to as **Vazulium** in its README and package branding, and as **Vector** throughout its own architecture documentation, same monorepo. It's a pattern-analysis and market-replay platform: detecting chart patterns, analyzing seasonal behavior, replaying historical markets with paper trading, with live trading planned eventually. It's organized as a domain-driven monorepo with eight bounded domains (Platform, Connectivity, MarketData, Charting, Replay, Analysis, Trading, Portfolio), each with its own boundary documentation and architecture decision records.

The person driving this project came to Claude wanting to add persistent, IndexedDB-backed storage for OHLCV candle data across the app. Their reasoning, as they explained it from the very first message: they didn't want more than roughly 10,000 to 100,000 candles sitting in memory at once during analysis or replay sessions. Long-running Replay v2 sessions on a one-minute timeframe spanning years — their example was January 2020 through 2026 — were overloading and lagging the entire app, because candle data was being held in memory with no cap. They also wanted to stop re-fetching the same data from the database every time a user returned to a date window they'd already looked at, whether for a fresh analysis pass or simply reopening the app. They were explicitly thinking ahead to a future where multiple symbols and multiple chart panels would be open simultaneously, multiplying memory pressure further, and wanted the architecture built with that scale in mind from the start rather than retrofitted later.

Critically, they didn't just want a static cache — they wanted it to **stay current**: new data arriving should update the durable store, not leave it serving stale data forever. And they wanted real architectural flexibility: the ability to enable the durable cache for one data provider or even one specific symbol while bypassing it entirely for another — their example was IndexedDB on for XAUUSD but off for GBPUSD on the same provider — all driven by configuration rather than hardcoded logic, so that changing this in the future would never require refactoring a running production app. They raised the possibility that a new architectural domain might be needed for this, though left that decision to Claude's judgment, and asked for an overload/eviction strategy for the durable store itself, suggesting something like a least-recently-used approach. They explicitly gave Claude latitude to recommend new tooling, improve on their own framing, and make the call on open questions using its own judgment — a posture that continued through the entire project.

---

## Research first: understanding the real codebase before designing anything

Claude does not have direct access to this repository — the person works with Claude in one window and with Cursor (running Grok 4.5, with actual repo access) in another, copying results back and forth between them. Rather than let Claude design an architecture against assumptions, Claude's first move was to write a **research-only** prompt for the person to paste into their Cursor session. That prompt deliberately asked for facts, not design: three separate markdown files covering (1) the repo and product overview including any documented long-term roadmap or deferred-work notes, (2) the current state of OHLCV data flow end-to-end including how Replay v2 actually works today, and (3) existing patterns and constraints — state management conventions, config/feature-flag patterns, testing conventions, build tooling, and whether the Node.js backend needed any awareness of a client-side cache. Cursor was told explicitly not to propose any architecture and to cite a file path for every non-trivial claim, so that everything downstream would be grounded in the app's actual code rather than inference.

That research came back as three documents and revealed a great deal that shaped everything after. Most importantly: this work was **already anticipated and named** in the repo's own documentation, as a deferred item called **MD-9** ("IndexedDB offline cache") on the MarketData package's own roadmap, referenced across an accepted architecture decision record (ADR-0013), a domain-maturity tracking table, and a reserved-but-empty source path (`packages/market-data/src/offline/`). The repo's own ownership-review documentation had already concluded, before this project started, that any durable client cache belonged inside the existing MarketDataDomain — not a new domain — composed as a tier sitting between the existing in-memory cache and the HTTP layer, using the same canonical series identity (`{symbol, source, timeframe}`) that the rest of the system already used. The research also surfaced that today's candle storage is entirely memory-only: a `MemoryCandleCache` with whole-series LRU eviction capped at 48 series, an uncapped `ReplayDatasetStore` that was the most likely actual source of the memory-lag complaint, and a legacy `ReplayRangeCache` already slated for retirement as a separate, already-planned piece of work referred to as **MD-4E**. It also found that three different parts of the codebase keyed candle series in three subtly different string formats — a real inconsistency that later became a required prerequisite fix. Finally, it confirmed the app used React Context for state management with no Redux/Zustand/etc., and had **zero** existing usage of Web Workers or IndexedDB anywhere — meaning whatever got built here would be genuinely new infrastructure for this codebase, not an extension of an existing pattern.

---

## The pre-implementation architecture plan

With that grounding, Claude wrote the main architecture plan, `00-MD9-durable-candle-cache-pre-implementation-plan.md`. It confirmed the domain-placement question was already settled by the repo's own documentation (MarketDataDomain, not a new domain) and focused the real design work on the parts that weren't already decided: a layered cache with the existing in-memory cache as a fast first tier and a new durable tier beneath it; a chunked, time-bucketed storage model (rather than one giant record per series) using columnar typed-array encoding for performance at the volume involved — a six-year span of one-minute candles is on the order of two to two-and-a-half million rows for a single symbol; a freshness model distinguishing a candle that's still "forming" from one that's genuinely closed, so the most recent bar is always revalidated rather than trusted blindly from cache; a "tiering policy" interface designed specifically to satisfy the dual-architecture / no-hardcoding requirement, so a provider or symbol's storage behavior could be changed later without touching consuming code; and a chunk-level least-recently-used eviction strategy for the durable store's own footprint, directly answering the overload concern from the original ask.

Because several of these design points were genuinely open — which storage library to use, whether to build with a Web Worker from the start or add one later, where the tiering configuration should live, what the default on/off state should be, what eviction granularity to use, how large the in-memory working budget should be, and how this work should sequence against the separate MD-4E effort — Claude wrote a companion decisions document rather than silently picking answers and burying them in prose, so the person could see and confirm or override each fork explicitly.

## Finalizing the architecture

The person came back with final decisions on all seven open points, explicitly stating these should be treated as fixed defaults going forward unless production evidence justified revisiting them: **Dexie.js** as the IndexedDB wrapper; build with a **Web Worker from day one**, not deferred to a later phase as Claude had originally proposed; keep the tiering configuration in a **frontend `config.ts`** file rather than server-driven YAML; **enable the durable cache by default in production**, allowed to be off in dev/test; **chunk-level LRU** eviction; a **100,000-candle working-memory budget shared dynamically across whatever replay datasets are actively open** at once, rather than allocated per symbol; and **Replay's own integration waits for MD-4E** to complete first, specifically to avoid ever running the old legacy cache, the in-memory cache, and the new durable cache simultaneously.

Folding these in required genuine redesign in two places, not just filling in blanks: the concurrency model changed from "main thread first, worker added later" to a worker that owns the Dexie connection exclusively from the start, with Comlink used for typed RPC and a hard rule that the worker itself would never perform network I/O — gap-fetching still happens on the main thread through the app's existing, guardrail-protected HTTP path, and only the persistence step happens inside the worker. And the memory-budget model changed from a simple per-series cap into a new coordinating component — later named the `ReplayWorkingSetCoordinator`, owned by the Replay domain, not MarketData — that arbitrates the shared 100,000-candle budget across however many replay datasets are active, evicting from memory (never from the durable store) using the same chunk-level LRU discipline used everywhere else in the design. One note on continuity: the session that folded these decisions into the actual `00` plan document and its companion decisions log was a different Claude session than the one that had drafted the plan initially — the person's usage limit was hit partway through that step. The version of `00` that exists in the actual repository does correctly reflect all seven finalized decisions; every implementation round since has confirmed this.

## The working method that got established

From this point on, a consistent pattern took hold, and it is the single most important thing for any session picking this up to continue: **Claude writes a detailed, numbered implementation task-breakdown document** — each issue given concrete file-level acceptance criteria, explicit "this needs to be verified against the real code" callouts wherever Claude wasn't certain of a current fact rather than guessing, and explicit dependency ordering. **The person pastes a short instruction into Cursor** telling it to read the relevant prior documents, verify any flagged assumptions against the actual current tree *before* writing any code, implement the specified work, and produce a detailed, evidence-based audit of exactly what it did — including where it deviated from the spec and why. **The person brings that audit back to Claude**, and Claude reads it critically rather than accepting a passing summary at face value: checking claimed deviations against the original intent, and specifically hunting for the gap between "the isolated piece was tested in isolation" and "the full, composed, real system actually behaves correctly end to end." That last habit is what caught two separate real bugs in this project so far, described below — an audit claiming success has consistently been treated as a starting point for scrutiny, not a stopping point, and that discipline should not soften just because a summary looks clean.

## Phase 0 and Phase 1: building the foundation

A different Claude session (again, due to usage limits) picked up the finalized architecture and wrote the actual Phase 0–1 implementation task breakdown, `02-MD9-implementation-tasks-phase-0-1.md`, scoping thirteen concrete issues: exporting canonical series-key helpers and building a Replay-side adapter for them, to finally close the three-different-key-formats inconsistency the original research had found; documenting the key-format contract so it wouldn't drift again; adding Dexie, Comlink, and fake-indexeddb as dependencies; building the actual Worker-and-Comlink RPC boundary — the first Web Worker anywhere in this application; the Dexie schema itself, with a series-coverage metadata table and a chunked candle table; the chunk-span and columnar-encoding codec; the worker's durable read/write service; a decorator provider composing the new durable tier into the existing candle-repository stack; wiring all of it through the app's existing runtime bootstrap; the `config.ts` file and the `VITE_MARKET_DATA_DURABLE_CACHE` flag, defaulting on in production per the finalized decision; graceful degradation so that any Worker or IndexedDB failure falls back to ordinary HTTP fetching rather than breaking candle loading; and automated architecture guardrails plus a real browser Worker smoke test using Playwright, specifically to prove the Worker actually bundles and runs in a real browser and not just inside a simulated test environment.

Cursor implemented all thirteen issues and produced `MD9-production-architecture-audit.md`, showing 99 package-level tests and 14 frontend tests passing, with the implementation confirmed to match the approved architecture — including the guardrail-enforced rule that the worker never touches the network, and the memory cache correctly remaining on the main thread rather than being moved into the worker.

## Reading that audit critically, and planning hardening plus real Phase 2

When that audit came back to Claude, the underlying Phase 0–1 specification was read directly as well, not just the audit's own summary of it — and this turned up three things worth closing before building further, none of which the audit itself had flagged as blocking: a real correctness question about whether a partially failed write could cause the cache to silently believe a range was fully covered when it wasn't; the complete absence of any check on how the new Worker would behave under Vite's development hot-reload cycle; and an outstanding manual sanity check — actually watching IndexedDB populate inside a real browser's developer tools — that the audit had itself flagged as not yet done, since an automated coding session has no way to perform that check itself. Claude wrote `03-MD9-implementation-tasks-phase-2.md` covering those three hardening issues plus the real Phase 2 feature work: giving the durable cache genuine freshness semantics by distinguishing a still-forming candle from a closed one and always revalidating the forming one on read; wiring the app's existing live-tick feed so a newly closed candle gets appended into durable storage incrementally rather than waiting for the next full range request; a boot/reconnect check so a series that fell behind while the app was closed catches back up; a forward-compatible hook for eventual data-restatement handling, deliberately left unwired to anything for now; and documentation cleanup.

## Verification catches a design gap and a false premise, before any code was written

The person then had Cursor verify that Phase 2 specification against the real code before implementing anything — and this verification-only pass, with no code changes, found two real problems and correctly stopped rather than building on top of them. First, the live-tick candle-close detection that the freshness-append issue had assumed already existed in the app's chart context simply did not exist; the live feed only ever updated a raw price, nothing else. Second, and more significant, the verification surfaced that a **High-severity bug** already flagged in the very first Phase 0–1 audit — that candle coverage was being tracked as a single min/max range, which silently claims a span is fully covered even when two separate, individually successful fetches left a real gap in between — had never actually been assigned its own fix in any task breakdown. Claude reviewed this and concluded, plainly, that this was a genuine gap in **Claude's own original architecture plan** — the coverage data model specified in section 4.4 of the `00` document — not merely something an implementer had missed. Claude rewrote the task breakdown as `04-MD9-implementation-tasks-phase-2-revised.md`: the coverage-correctness fix was redesigned around verifying actual chunk presence in the existing chunk table rather than trusting a derived range summary at all, and the candle-close-append feature was redesigned around treating each incoming live tick purely as a trigger signal — watching for the moment a tick's timestamp crosses into a new timeframe bucket, and at that moment fetching just the one newly closed bar through the app's existing HTTP range API and persisting it through the already-built write path, avoiding both the fictional detection hook and any need for a backend change.

## Phase 1 hardening + Phase 2 (revised): implementation and audit

Cursor implemented file `04` and produced `MD9-phase-1H-phase-2-revised-implementation-audit.md`, showing seven of eight issues fully passing — 114 package-level tests plus 6 frontend tests, all green — with the disjoint-coverage bug confirmed closed by a regression test that specifically reproduces the two-separate-successful-fetches-with-a-gap-between-them scenario. The only partial item was the manual, human-only browser sanity check, which by its nature an automated session cannot perform.

## A second hypothesis, and a second confirmed bug

Reading that audit, Claude noticed a detail its own language hinted at without following through on: a note that the tick-triggered append's persistence "goes through the full repository stack (MCC may absorb)." Tracing the actual request layering — the existing in-memory cache sits in front of the new durable tier and only falls through to it on a cache miss — Claude raised the concern that for the exact scenario the freshness work exists to solve, an actively-watched chart where the in-memory cache is almost certainly already warm, the newly-closed-candle fetch might never actually reach the durable tier or a fresh network call at all, silently defeating the entire feature. This was written up as a verification-first issue, `05-MD9-mcc-freshness-shadow-check.md`, explicitly asking for it to be proven or disproven with a direct test rather than assumed true. Cursor's verification, `MD9-phase-2-freshness-verification.md`, confirmed it was real, with exact file-and-line code citations and four regression tests reproducing both the broken and the working paths: a warm in-memory cache genuinely does short-circuit the tick-triggered fetch before it ever reaches the durable tier, the network, or the durable write-through — which matters most precisely for the actively-viewed-chart case the feature was built for. The same investigation found the separate boot/reconnect frontier-sync mechanism is structurally correct once it's actually entered, but can also be skipped if the in-memory cache happens to satisfy the initial window request before the durable tier is ever reached — assessed as a lower-severity, delayed-rather-than-lost gap, not an immediate correctness bug.

---

## Where things stand right now

MD9-2.6 has now been implemented and independently audited. The minimal correction approved after `MD9-phase-2-freshness-verification.md` landed exactly as designed: `bypassCache` was added to `CandleRangeQuery`, honored by `CachingDataProvider.getRange`, and used only by the MD9-2.2 ChartContext tick-trigger path. No tier ownership, repository composition, or architectural boundaries changed.

The additional acceptance criteria introduced during planning were also completed. Rather than assuming the bypass fetch corrected the in-memory cache, the implementation verified `mergeCandles` directly. It already performs an overwrite/upsert on duplicate timestamps using incoming data, so no merge rewrite or cache invalidation mechanism was required. A new regression test demonstrated the complete user-visible behavior: after a bypass refresh, a subsequent ordinary `getRange` returns the corrected candle from MCC rather than the stale one. MD9-2.6 is therefore considered closed.

Reading the completed implementation audit prompted one further verification before declaring Phase 2 finished. The same cache-shadowing pattern fixed in MD9-2.6 may exist in MD9-2.1's tip-revalidation path, since that logic also executes only after control reaches the durable tier. Unlike MD9-2.2, however, there is no single known call site where a targeted bypass can simply be applied. This has therefore been recorded as a verification task (`07-MD9-tip-revalidate-mcc-shadow-check.md`) rather than another implementation issue. The objective is to determine, with runtime evidence and regression tests, whether ordinary reads of a warm forming bucket bypass the intended revalidation logic, or whether the current implementation is already correct. Because the application's primary live chart updates from the separate `livePrice` stream rather than repeated repository reads, the expected exposure is narrower than MD9-2.2, but it should be verified rather than assumed before Phase 2 is considered complete.

## What the next session needs to do

1. Perform the verification described in `07-MD9-tip-revalidate-mcc-shadow-check.md`. This is a verification-only task. Trace the runtime, add the required regression test, and determine whether MCC shadows MD9-2.1's tip-revalidation path on ordinary reads.

2. If the hypothesis is disproven, document why with code evidence and close the issue. If confirmed, determine the smallest correction appropriate for MD9-2.1 rather than automatically reusing the MD9-2.6 `bypassCache` approach.

3. Complete MD9-1H.3, the manual browser DevTools IndexedDB sanity check. This still requires a human operator.

4. Once MD9-2.7 has been verified (whether it closes as PASS or results in another implementation task) and MD9-1H.3 has been completed, Phase 2 can be considered fully complete.

5. Continue with the deferred roadmap:
   - **Phase 3** — per-provider / per-symbol tiering policy.
   - **Phase 4** — durable cache eviction and quota management.
   - **Phase 5** — Replay integration via `ReplayWorkingSetCoordinator` (blocked on MD-4E).
   - **Phase 6** — telemetry, repository maturity updates, and final architecture documentation.

## Reference file index (chronological)

- `00-MD9-durable-candle-cache-pre-implementation-plan.md` — the core architecture plan, since amended in place for the MD9-1H.2 coverage-model correction
- `01-confirmed-decisions.md` (originally `01-open-decisions.md`, renamed during Phase 2 housekeeping) — the seven finalized architecture decisions and their reasoning
- `02-MD9-implementation-tasks-phase-0-1.md` — the thirteen-issue Phase 0–1 specification
- `MD9-production-architecture-audit.md` — Phase 0–1 as-built audit (99+14 tests)
- `03-MD9-implementation-tasks-phase-2.md` — **superseded, do not implement** — first hardening + Phase 2 draft
- `MD9-phase-1H-phase-2-architecture-audit.md` — verification-only pass that found the false premise and the unowned disjoint-envelope bug, before file 03 was ever implemented
- `04-MD9-implementation-tasks-phase-2-revised.md` — the corrected, actually-implemented hardening + Phase 2 specification
- `MD9-phase-1H-phase-2-revised-implementation-audit.md` — as-built audit for file 04 (7 PASS / 1 PARTIAL / 0 FAIL)
- `05-MD9-mcc-freshness-shadow-check.md` — the verification-first issue raising the in-memory-cache-shadowing hypothesis
- `MD9-phase-2-freshness-verification.md` — confirms the hypothesis with code evidence and regression tests
- `06-MD9-2.6-fix-implementation.md` — implementation specification for the MD9-2.6 freshness bypass correction
- `MD9-2.6-bypass-cache-implementation-audit.md` — implementation audit confirming the minimal fix, merge upsert semantics, and user-visible regression behavior (120 tests)
- `07-MD9-tip-revalidate-mcc-shadow-check.md` — verification-only investigation into whether MCC shadows MD9-2.1's tip-revalidation path on ordinary reads

---

## How to keep updating this document

When you complete work, don't just close it out in isolation — come back to this file and append a new dated section below this line, in the same plain, narrative style as the rest of this document, describing what was done, what was found, and what changed. Then rewrite "Where things stand right now" and "What the next session needs to do" so they describe the actual current state rather than the state as of this writing. Keep the reference file index current as new numbered files are added. The goal is that any session, at any point, can read this one document top to bottom and know exactly what this project is, why every major decision was made the way it was, and precisely what to do next — without needing to reconstruct any of it from scratch.

---

## Log: 2026-07-14 — supplying the missing Cursor prompt for 06

The prior version of this document described `06-MD9-2.6-fix-implementation.md` as written but not yet pasted into Cursor, correctly — but it stopped short of actually supplying the prompt to paste, breaking the pattern every earlier round had followed of pairing a task file with its exact wrapper prompt. The person flagged this directly. Below is the exact prompt to use — paste it into Cursor as-is, don't rewrite it:

```
Read in order:
1. docs/architecture/audits/MD9-phase-2-freshness-verification.md (the verification that found this bug, with code evidence and the 4 existing regression tests)
2. docs/architecture/plan/.../06-MD9-2.6-fix-implementation.md (the fix — implement this)

Implement the fix exactly as specified in file 2: add optional bypassCache to
CandleRangeQuery, honor it in CachingDataProvider.getRange (skip tryGetRange
when true, fall through to the existing miss path), and pass
bypassCache: true from ChartContext's tick-trigger call only.

Before considering this done, complete the "Added AC" section in file 2 —
this is not optional cleanup, it is the actual point of this task. The four
existing regression tests only prove HTTP/durable get reached; none of them
check whether MCC's own held value gets corrected afterward. Specifically:

- Read mergeRangeResult's real upsert semantics first: does it overwrite an
  existing timestamp's candle with a newer value, or only add missing ones?
- Write the new regression test: warm MCC holds a stale (forming) OHLC for
  bucket X -> tick-trigger fires with bypassCache:true -> fresh candle
  fetched and durable-written -> a subsequent NORMAL (non-bypassed) getRange
  for bucket X returns the corrected OHLC, not the stale one.
- If merge does not overwrite: fix mergeRangeResult to upsert on conflict,
  or call the existing MemoryCandleCache.invalidateSeries for that series
  after the bypass fetch. Don't ship a state where HTTP was called
  correctly but MCC still hands back stale data on the next read.

When done, produce an audit in the same format as your prior ones — status,
files touched, AC pass/fail, test evidence — and be explicit about which of
the two merge-semantics outcomes you found, since that's the one thing no
prior audit in this project has actually checked.
```

Nothing else changed in this pass — no new files, no revised decisions. This is purely closing a process gap so the next session (or the person themselves) isn't left reconstructing a prompt that already existed in chat but hadn't been committed to this document.


## Log: 2026-07-15 — MD9-2.6 implemented and closed; new verification raised for MD9-2.1

The MD9-2.6 implementation has now been completed exactly as specified in `06-MD9-2.6-fix-implementation.md` and independently audited. The minimal fix landed without changing the established architecture: `bypassCache` was added to `CandleRangeQuery`, honored by `CachingDataProvider.getRange`, and used only by the MD9-2.2 ChartContext tick-trigger path. No repository composition, cache ownership, or durable-tier responsibilities changed.

The additional acceptance criteria introduced during planning were also completed. Rather than assuming the bypass fetch corrected the in-memory cache, the implementation verified `mergeCandles` directly. It already performs an overwrite/upsert on duplicate timestamps using incoming data, so no merge rewrite or cache invalidation mechanism was required. A new regression test demonstrated the complete user-visible behavior: after a bypass refresh, a subsequent ordinary `getRange` returns the corrected candle from MCC rather than the stale value. With this evidence, MD9-2.6 is considered closed.

Reviewing the completed implementation prompted one further verification before declaring Phase 2 finished. The same cache-shadowing pattern fixed in MD9-2.6 may exist in MD9-2.1's tip-revalidation path, since that logic also executes only after control reaches the durable tier. Unlike MD9-2.2, there is no single known call site where a targeted bypass can simply be applied. This has therefore been recorded as a verification task (`07-MD9-tip-revalidate-mcc-shadow-check.md`) rather than another implementation issue. The objective is to determine, with runtime evidence and regression tests, whether ordinary reads of a warm forming bucket bypass the intended revalidation logic, or whether the current implementation is already correct. No architectural changes are proposed unless that verification proves a real issue exists.
