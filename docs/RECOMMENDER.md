# Recommender — design doc

## Contract

```
recommend(screen, { mode: 'quick' | 'deep', force?: boolean }) → Array<{
  type: 'enter' | 'grow' | 'exit' | 'marketing' | 'idle' | 'hold_all'
  country?: string
  mode?: 'F' | 'JV' | 'WOS'          // for 'enter'
  decisions?: DecisionShape          // for 'marketing'
  label: string                      // human-readable
  expectedProfitY6: number           // mean cumulative profit at EoY6
  stdDev: number
  p10, p50, p90: number
  reasoning: string                  // why this candidate
  warnings: string[]                 // flags (e.g., "Political Crisis Y5")
}>
```

Results sorted by `expectedProfitY6` descending. Top row is the green-
highlighted recommendation in the UI.

## Candidate enumeration

Per `screen`:

- **`market-entry`** — for each not-yet-entered country × each mode
  {F, JV, WOS} = up to 33 candidates. Plus "grow {existing country}" for
  each active market. Plus "idle this year."
- **`exit`** — for each active market: "exit now." Plus "hold all active
  markets." (Multi-exit combinations not enumerated — bound search space.)
- **`marketing-overview`** — for each active WOS/JV market, grid-search
  sourcing {local, global} × priceChoice {match, premium} × promoChoice
  {low, med, high} = 12 combos per market. Adaptation held at current
  decision. Aggressive price omitted (narrow window rarely optimal).
  Franchise markets skipped (franchisee self-manages).

## Rollout policy

For each candidate:
1. Clone current `game` state.
2. Apply the candidate's action at `game.year` (entry / growth / exit /
   marketing override).
3. Run `computeYearProfit` for that year.
4. For each remaining year, run `_greedyFuture`: double the largest
   affordable existing market; update decisions with a simple distance-
   and-year heuristic; compute year.
5. Record the final cumulative profit.
6. Repeat `N_QUICK=30` (quick) or `N_DEEP=200` (deep) times, each with a
   fresh seed.

**Quick mode** — ~500ms budget, greedy future.
**Deep mode** — ~5s budget, still greedy (2-step lookahead deferred; noted
below as a limitation). Progress indicator shown while running.

## Caching

Key: `configHash | gameStateHash | screen | mode`.
- `configHash` serializes every `.value` in the algorithm + rules + events.
  Edit any coefficient → cache miss → recompute.
- `gameStateHash` serializes home, year, cash, positions, exited.
  Entering/growing/exiting → cache miss.
- Cache invalidation is wholesale on any config change via a store
  subscriber.

## Reasoning strings

Built from the country's Forio category labels (cultural distance, rivalry,
indulgence) plus mode-specific narrative:
- F: "Franchise: low capital, 25 outlets, fastest scale"
- JV: "JV: shared risk, 50% profit share"
- WOS: "WOS: full control + profit, liability of foreignness"

## Warnings

Generated from config, not from simulation — fast and deterministic:
- If entering WOS in a `highRiskPoliticalCountries` member and the Political
  Crisis event is enabled for Y5: "⚠ Political Crisis Y5 will force-exit
  this WOS."
- If entering a country with `tariff > 10` while a `tradeWar`-kind event is
  enabled: "⚠ High tariff country — Trade War will raise VC significantly
  if sourcing globally."

## Live recalc

The store subscriber clears `_recCache` on any change and re-renders the
Coach card if the corresponding decision screen is active. No debounce on
the recommender itself (fast enough with N_QUICK). Debounced at 300ms at
the `replayAllYears` layer above it.

## Tuning knobs

Documented constants near the recommender:
- `N_QUICK = 30` — rollouts per candidate in quick mode
- `N_DEEP  = 200` — rollouts per candidate in deep mode

Increase `N_QUICK` for smoother ranking at the cost of 400ms+ latency on
every decision screen entry. Decrease to 10 for instant response at the
cost of noise.

## Known limitations

- **Deep mode uses greedy future**, not the 2-step lookahead specified in
  the original prompt. 2-step lookahead multiplies the per-candidate cost
  by `#candidates^2`, bringing 200-rollout runtime into minutes rather
  than seconds. Revisit if needed. The stochastic outer loop (200
  rollouts) already does most of the variance work.
- **Greedy future** always prefers growing the largest existing market.
  It does not model later-year entries after the candidate turn.
- **Marketing candidate enumeration** omits the `adaptation` axis — it's
  held at the current decision value to keep search bounded. Tune
  adaptation manually on the Country Marketing screen or via Settings.
- The recommender assumes the candidate applies only in the current year.
  A multi-year strategy (e.g., "enter UK now AND WOS Japan next year")
  isn't enumerated.
