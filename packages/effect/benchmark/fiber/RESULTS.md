# Fiber runtime optimization: results and experiment ledger

Baseline: upstream `757821fe99b7179f907d6d1a34a4e86de4173112`. Host: Intel Xeon Platinum 8272CL (8 vCPU, 2.6 GHz),
31 GiB, Linux 6.18. Toolchain from the repository flake: Node 26.7.0, Bun 1.3.13, Deno 2.9.4. Every definitive run held
`flock /tmp/effect-fiber-bench.lock`, which a second optimization effort on the same host also used. Raw reports, logs
and profiles live in the ignored `tmp/fiberperf/` directory.

## Retained changes

| Change                                                      | Main effect                                                                                                                                                                    |
| ----------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Combine interrupt causes without global hashing             | Interruption no longer hashes causes through the global `Hash`/`Equal` caches: interrupt, race and timeout 40–80% faster; about 125 B per interrupted fiber no longer retained |
| Literal strings for primitive property keys                 | `${EffectTypeId}/args`-style keys made V8 keyed stores megamorphic (17% of ticks in a flatMap loop)                                                                            |
| Dedicated sites for map/flatMap and generator continuations | Inline evaluation and `getCont`/`continueWith` fast paths for the dominant frame types                                                                                         |
| Per-kind constructors for frequent primitives               | Success, Failure, Sync, Suspend, WithFiber and WithFiberSucceed no longer share one megamorphic constructor                                                                    |
| Dispatcher lanes                                            | About 300–400 B less allocation per scheduler cycle; idle dispatchers retain no lanes                                                                                          |
| Async finalizer success path                                | Skips a mask push/pop that cancelled out                                                                                                                                       |
| Context shape, derived fiber caches, `merge` fast paths     | Spans, `Effect.fn` and scoped resources avoid cache rebuilds and full context copies                                                                                           |
| Fresh stack array on completion                             | Avoids V8's length setter; releases the grown backing store                                                                                                                    |
| Queue field primitives, one-slot MutableList buckets        | Queue offer/take without closures or intermediate Exits                                                                                                                        |

## Final comparison (Node, baseline vs final, 20 interleaved fresh-process pairs, polluted type feedback)

Each value is the paired change in mean time per iteration with its 95% bootstrap interval. The p99 column is the paired
change in per-process p99. All 26 workloads improve, with the whole interval below zero for both mean and p99.

| Workload                | Baseline | Final    | Mean change [95% CI]  | p99 change |
| ----------------------- | -------- | -------- | --------------------- | ---------- |
| succeed-flatmap-loop    | 20.86 ms | 10.66 ms | −48.6% [−49.1, −48.0] | −46.9%     |
| map-chain-deep          | 9.76 ms  | 4.47 ms  | −54.5% [−55.2, −53.7] | −51.4%     |
| left-assoc-flatmap-deep | 14.33 ms | 7.74 ms  | −46.0% [−47.0, −45.3] | −32.1%     |
| gen-loop                | 7.02 ms  | 5.60 ms  | −20.2% [−20.7, −20.0] | −17.0%     |
| error-unwind            | 7.77 ms  | 1.88 ms  | −75.9% [−75.9, −75.7] | −74.3%     |
| finalizers-success      | 2.68 ms  | 2.30 ms  | −14.0% [−14.7, −13.4] | −12.4%     |
| finalizers-failure      | 3.59 ms  | 3.26 ms  | −8.8% [−9.5, −7.6]    | −8.1%      |
| sync-runSync            | 4.03 ms  | 2.06 ms  | −49.0% [−49.5, −47.9] | −46.8%     |
| fork-join-sequential    | 16.85 ms | 14.79 ms | −12.6% [−13.3, −11.7] | −12.0%     |
| fork-fanout-join-all    | 14.61 ms | 12.77 ms | −12.2% [−13.3, −11.1] | −18.4%     |
| forEach-bounded         | 14.41 ms | 11.28 ms | −20.8% [−22.8, −19.0] | −18.8%     |
| forEach-unbounded       | 2.95 ms  | 2.35 ms  | −20.8% [−26.6, −19.5] | −46.9%     |
| short-lived-fibers      | 2.70 ms  | 1.75 ms  | −34.7% [−35.5, −31.9] | −31.9%     |
| callback-resume         | 5.11 ms  | 4.60 ms  | −9.8% [−10.7, −9.1]   | −9.9%      |
| promise-interop         | 5.41 ms  | 4.85 ms  | −10.5% [−11.6, −9.1]  | −10.2%     |
| yield-contention        | 27.01 ms | 21.25 ms | −21.5% [−22.5, −20.9] | −19.2%     |
| deferred-pingpong       | 9.20 ms  | 7.64 ms  | −16.5% [−17.4, −14.9] | −14.5%     |
| interrupt-suspended     | 16.81 ms | 3.36 ms  | −79.8% [−80.3, −79.6] | −95.2%     |
| race                    | 25.34 ms | 11.31 ms | −55.3% [−55.6, −55.0] | −80.8%     |
| timeout                 | 15.99 ms | 8.19 ms  | −48.8% [−49.3, −48.3] | −80.5%     |
| scope-finalizers        | 12.12 ms | 10.20 ms | −15.8% [−16.9, −14.8] | −16.5%     |
| context-locals          | 12.96 ms | 11.78 ms | −9.2% [−10.7, −7.0]   | −9.6%      |
| tracing-spans           | 16.23 ms | 12.92 ms | −20.2% [−21.3, −19.5] | −19.5%     |
| queue-bounded-pc        | 30.21 ms | 22.71 ms | −24.9% [−25.6, −23.9] | −21.5%     |
| queue-mpmc              | 11.70 ms | 7.74 ms  | −34.2% [−34.6, −32.3] | −29.9%     |
| mixed-service           | 13.05 ms | 10.43 ms | −20.1% [−20.6, −19.2] | −25.7%     |

The full run measured `911b5fac3`. The shipped stack-release commit differs from it: it replaces a guarded
`length = 0` with a fresh array. Measured against that run, it costs fork-join-sequential +1.2% [+0.5, +2.6] and
mixed-service +1.7% [+0.4, +2.4], and fixes a +128 B per completed fiber retention regression.

Additional checks:

- Fixed-count tail check with 1000 iterations per process on both sides (16 pairs): all eight tail-sensitive workloads
  improve their p99, including fork-fanout-join-all −13.9% [−19.5, −9.0].
- Five workloads added after a methodology review (20 pairs): root-run-promise −19.3%, sleep-resume −19.7%,
  timeout-fires −54.4%, race-both-work −43.0%, interrupt-busy-finalizers −56.6%.
- Without type-feedback pollution (12 pairs): all 11 sampled workloads improve. gen-loop (−5.8%) and callback-resume
  (−3.4%) gain less than with pollution.
- Full-suite A/A run (baseline against itself, 10 pairs): no mean flags, one time-window p99 flag.
- Fairness (yield-contention): completion spread 0.043 → 0.044, at most one step behind on both sides.

## Memory (forced GC, paired fresh processes, n = 50 000 unless noted)

| Measurement                                                     | Baseline                 | Final     |
| --------------------------------------------------------------- | ------------------------ | --------- |
| Retained per fiber after interrupt and release                  | 124.7 B (5.94 MiB total) | about 0   |
| Fiber suspended on `Effect.never` (per heap space, n = 100 000) | 386.4 B                  | 386.4 B   |
| Fiber suspended on a Deferred after one yield                   | 1036 B                   | 1012 B    |
| Completed handle                                                | 216.8 B                  | 216.7 B   |
| Completed handle after one yield                                | 432.8 B                  | 409.3 B   |
| Peak heap per item, 100 000-item fan-out                        | 1.4 KiB                  | 1.1 KiB   |
| Peak RSS growth, 100 000-item fan-out                           | 146.9 MiB                | 125.1 MiB |

Allocation per iteration falls on yield-contention (−38%, 6 fewer scavenges), queue-mpmc (−32%), race (−23%),
mixed-service (−19%), callback-resume (−8%) and fork-fanout-join-all (−7%). succeed-flatmap-loop is flat and gen-loop is
+2.4% [+0.2, +4.4].

## Other engines

- **Deno 2.9.4 (V8), 8 pairs:** 27 of 30 workloads improve and 3 are inconclusive. In a fixed-count tail check,
  race p99 is +0.8% [−13.5, +4.1] and forEach-bounded p99 +8.9% [−8.0, +25.0] (inconclusive).
- **Bun 1.3.13 (JavaScriptCore), 8 pairs:** most workloads improve by 10–85%, including mixed-service −25%, but
  some regress:

  | Workload           | Change             |
  | ------------------ | ------------------ |
  | finalizers-success | +26%               |
  | finalizers-failure | +13%               |
  | short-lived-fibers | +4% (inconclusive) |
  | callback-resume    | +3% (p99 +5%)      |

  A bisect attributes this to the frame-type checks in the continuation fast paths. JavaScriptCore pays for
  `instanceof` misses on other frame types. All four alternatives tried recover Bun but cost V8 much more:

  | Variant                                    | V8 cost (succeed-flatmap-loop) |
  | ------------------------------------------ | ------------------------------ |
  | Remove the inline evaluation               | +20%                           |
  | Remove the `getCont` fast path             | +26%                           |
  | Prototype tag instead of `instanceof`      | +22%                           |
  | Prototype identity instead of `instanceof` | +38%                           |

  Node is the primary target, so the V8-optimal form is kept. Selecting the frame check per engine at load time is
  left as a follow-up.

## Rejected experiments

- **Explicit `AsyncResource` trigger id** (−7% callback-resume on Node): throws on Cloudflare Workers with
  `nodejs_compat`, and changes the trigger id seen by async_hooks inside Node's default-trigger scopes.
- **Reusing the `yield*` iterator as its own result object:** violates a tested iterator contract.
- **Guarded `length = 0` on completion:** a popped-empty stack keeps its backing store (+128 B per completed fiber).
  It was replaced by the fresh-array version.
- **Queue taker release task built once:** an extra own key changes structural `Equal`/`Hash` of queues.
- **Queue effects keeping the queue in `[args]` without overrides:** printing a take or offer walked the whole queue,
  and equality and hashing changed. This was fixed with identity `Equal`/`Hash` and a constant `toJSON`.
- **First dispatcher lane design:** retained up to two 64-slot lanes per idle dispatcher (+160–190 B per fiber). It was
  replaced by lanes released when idle.
- **Sharing one dispatcher across fibers** (large async gains in analysis): changes interleaving with microtasks and
  priority ordering. Not attempted.
- **`AsyncLocalStorage.snapshot()` instead of `AsyncResource`:** about 1.5 µs per call. Skipping the capture when the
  context seems unchanged would lose `enterWith` updates.

## Limitations

- Measurements run the TypeScript sources under Node's type stripping, not the built `dist`. A minifier could merge the
  dedicated call sites.
- Time-window p99 is a weak statistic for workloads with fewer than about 200 iterations per process, where it is close
  to the slowest iteration. The fixed-count tail check above is the tail evidence.
- Per-fiber `heapUsed` has a granularity of about 31 B at n = 50 000, so differences of that size are noise.
- Context objects now always carry `_fiberCache`, `_fiberCacheParent` and `_fiberCacheKey` own keys. Queue effects
  print as `{ _id: "Effect", op }` and use different op names in `Tracer.context` hooks.

## Reproduce

```sh
git worktree add --detach /tmp/base 757821fe99b7179f907d6d1a34a4e86de4173112
ln -s "$PWD/node_modules" /tmp/base/node_modules
ln -s "$PWD/packages/effect/node_modules" /tmp/base/packages/effect/node_modules
B=packages/effect/benchmark/fiber
flock /tmp/effect-fiber-bench.lock nix develop --command node $B/compare.mts --base /tmp/base --head . --rounds 20
flock /tmp/effect-fiber-bench.lock nix develop --command node $B/compare.mts --base /tmp/base --head . \
  --workloads short-lived-fibers,fork-fanout-join-all --rounds 16 --time 1 --min-iterations 1000
flock /tmp/effect-fiber-bench.lock nix develop --command node $B/memory.mts --base /tmp/base --head . \
  --scenario released --n 50000 --repeats 7
flock /tmp/effect-fiber-bench.lock nix develop --command node $B/profile.mts --root /tmp/base \
  --workload mixed-service --pollute --label base-mixed
nix develop --command node $B/compare.mts --engine bun --base /tmp/base --head . --rounds 8
```

Run Deno comparisons with the in-repository harness, because the repository's `deno.json` must be in scope.
