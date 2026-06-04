# Gascity Upstream Contribution Analysis

Working triage for contributions from the `jsgerman-oss` fork → `gastownhall/gascity`, maintained on
branch `contribution-analysis`. Investigations fan out across **Claude + Codex** (model-advisor picks
tiers); fixes land on per-bug branches. Verified against `origin/main` (this branch's base).

## In-flight (ours)
| Branch | What | Bead |
|---|---|---|
| `codex-reasoning-effort` | forward Codex reasoning effort (`gc.reasoning` → `-c model_reasoning_effort`) | — |
| `fix-hook-projection-dedup` | dedup hook entries on overlay merge (matcherless `{hooks:[…]}` shape) | `bh-hm6` |

## Incident-correlated bugs (first-hand data from incident `bh-wisp-l4base`)
We just resolved a HIGH town outage — supervisor FD exhaustion (245k FDs) → EMFILE → wedged supervisor
→ dispatcher stall + Dolt hot-loop on suspended rigs. Several open upstream bugs are in the same
failure families:

### #2954 (P2) + #2903 (P1 tracker) — supervisor FD leaks  ·  **FIX CANDIDATE #1 (best first PR)**
**#2954 and our `bh-zz2` are DISTINCT leaks, same EMFILE symptom.**
- **#2954 = kqueue WATCH-FD leak (macOS).** Recursive config watcher: `cmd/gc/controller.go:617`
  `WalkDir` → `:651` `watcher.Add` per subdir → `:802` re-enqueue on Create/Rename. It watches
  `node_modules` because the ignore lists — `shouldIgnoreConfigWatchEvent` (`controller.go:834`) and
  `isIgnoredPackRuntimePath` (`internal/config/pack.go:2838`) — exclude only `.gc`/`.beads`, NOT
  `node_modules`. fsnotify kqueue = **1 FD per watched path** → ~51k FDs. `WatchTargets`
  (`internal/config/revision.go:272`) also ignores `rig.Suspended`. (Reporter's "revision hasher"
  hypothesis is wrong: `os.ReadDir`/`ReadFile` close immediately; the FD holder is the kqueue watcher.)
- **`bh-zz2` (ours) = Dolt CONNECTION leak — NOT in current `main`.** The supervisor's recurring Dolt
  loop (`internal/supervisor/maintenance.go:610` `runDoltGC`) is correct (open-once-per-cycle +
  `defer Close`). Our 245k-FD leak came from an older path (running `gc` is brew **1.2.1**, behind
  `main`). Distinct from #2954; #2903 tracks neither.
- **Fix A (PRIMARY):** add `node_modules`/`.git`/`.cache`/`dist`/`vendor`/`.next`/… to BOTH ignore
  lists (`controller.go:834` → `filepath.SkipDir`; `pack.go:2843`). **S / Low.** Closes #2954; ~51k
  FDs gone. PR must correct the mechanism attribution. → **dispatch to Codex.**
- **Fix B (2nd PR):** skip suspended rigs in `WatchTargets` (`revision.go:272`).
- **Fix C (`bh-zz2`):** file a new P2 ref #2903 + a regression test (no live repro on `main`).

### #2893 (P1) — order_dispatch BFS blocks the dispatch loop  ·  **FIX CANDIDATE #2**
**Root cause (verified):** `storeHasOpenDescendants` (`cmd/gc/order_dispatch.go:1438`) BFS-walks the
wisp subtree calling `store.List(ParentID,IncludeClosed)` per node → one `bd list` subprocess per node
(`internal/beads/bdstore.go:1661`). Only the wisp ROOT carries the `order-run:<scoped>` label
(`cmd_order.go:700`, `order_dispatch.go:1262`); `molecule.Options` has no label propagation, so
descendants are unlabeled → the single labeled query misses open steps → falls into the per-node BFS.
Dozens of sequential `bd` spawns/tick under Dolt write contention → dispatch goroutine blocks for
minutes → feeders never tick. (PR #2878's 8s gate timeout is merged — a mitigation, not the fix.)
**Fix:** Option B reshaped — replace the per-node `bd` BFS with an in-memory subtree walk over one
cache-served `CachedList` snapshot (CachingStore currently *misses* on `ParentID`/`IncludeClosed` —
`internal/beads/caching_store_reads.go:18,66,107` — so the query must be reshaped to hit cache),
keeping the `bd`-walk as cache-miss fallback. Must preserve closed-intermediate/open-grandchild +
single-flight re-dispatch guards. **M / Medium.** First PR *with strong tests* (0 `bd list` calls when
warm; open-grandchild-under-closed-child → true; orphan-all-closed → false; cache-miss fallback).
→ **dispatch to Claude** (correctness-sensitive).

### #2984 (P2) — `gc supervisor status` reports not-running while healthy  ·  **TO INVESTIGATE (Codex)**
Our incident showed the exact split: `status` read the pidfile (said "running") while `reload`/`stop`
hit the dead socket (said "not running") under FD exhaustion. Likely a pidfile-vs-socket health-check
inconsistency.

### #2958 (P1, accepted) — `gc sling` success on suspended rig; work stalls  ·  **TO INVESTIGATE**
Our incident hot-looped on suspended rigs.

## Test coverage gaps
134 packages, 25 without tests — most are test-helpers (`*test`, `testutil`, `fakecmd`). Real
candidates: `internal/clock`, `internal/beads/closeorder`, `internal/bootstrap/packs/core`, the
`benchmarks/coordstore/adapters/*`. (Detailed coverage pass: pending — a fanned-out agent.)

## Other notable open bugs (backlog)
- **#3048 P0** — 0-diff/0-commit branch recorded close-as-merged → silent work loss. [high value]
- #3008 P1 — pack-relative `gc.check_path` fails with control-bead `work_dir`.
- #3001 P2 — `gc init` doesn't detect codex/claude CLI installed via pnpm.
- #2987 P2 — `bd list`/`ready` serves stale cached state under write load.
- #2940 P2 — workflow snapshot GET scans all stores (2–4s) on SQL fast-path fallback.
- … full list: `gh issue list -R gastownhall/gascity --state open`

## PR plan (priority)
1. **#2954 Fix A** — `node_modules` ignore-list (S/Low) — **first PR**, **Codex**.
2. **#2893** — cache-served descendant walk (M/Med) — **second PR**, **Claude**.
3. **#2984** — supervisor status pidfile/socket — investigate (Codex) → fix.
4. `bh-zz2` — file new issue + regression test.
5. Coverage: `internal/clock` + `internal/beads/closeorder` tests.
