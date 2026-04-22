# Changelog

## Phase 1 — Data layer extraction
`configStore` (pub/sub + localStorage persist at `hbr-sim-config-v1`). Every
hardcoded country stat, entry mode, rule, coefficient, event and news string
moved behind the store. Live bindings keep existing render code working
unchanged.

## Phase 1.5 — Canonical algorithm rewrite
`countryEconomics` / `variableCostPerMeal` deleted. `computeMarketPnL` +
`computeYearProfit` implement the spec's decomposable model (meals × price
− meals × VC − FC − promo, with mode-specific player share). All ~40
coefficients in `config.algorithm` each `{value, min, max, pinned, rngMin,
rngMax, desc}`. Structured per-mode / per-category multipliers nested.
`buildEventsView` replaces the flat `impacts{}` dict. Event preset system
(`spec` / `hbr`); Spec active on first load. Country `population` added.
Radio-pill decision UI replaces sliders. Calibration Check panel on Results
with 9 default scenarios, auto-run on config change, CSV import, "Why?"
hints, localStorage persistence at `hbr-sim-calibration-v1`.

## Phase 2 — Settings modal
Fixed-position gear button (top-right). Full-screen modal with 6 tabs:
- **Countries** — inline editable table with "Set home" per row
- **Entry Modes** — per-mode fields + linked algorithm coefficients
- **Game Rules** — starting cash, total years, category thresholds, seed +
  freeze toggle, bonus tiers, entry-penalty threshold, active event preset
- **News &amp; Events** — per-year headline / body / coachPlaybook (one-tip-
  per-line textarea) with "Duplicate from Y(n-1)"; per-event enabled +
  year + countries chips + impact JSON editor
- **Algorithm** — every coefficient with pin toggle, value, rngMin, rngMax,
  plus a live formula-preview card showing the meals/price/VC/FC formulas
  with current values substituted
- **Import / Export** — download JSON, paste + validate + apply, reset,
  preset loaders
Live edits debounce-save (300ms) to localStorage; "Saved ✓" indicator;
dirty-vs-default warning banner with one-click Export or Reset.
Escape closes; Cmd/Ctrl+E opens Quick Edit (Algorithm tab).

## Phase 3 — Formula trace + auto-recalc
Expandable "Formula trace" section on Results below the Income Statement.
Dropdowns to pick country + year; shows every factor (growth, culturalFit,
rivalryShare, indulgence, priceResponse, promoResponse, adaptationLift,
foreignness, eventDemandMult, saturation) plus derived meals, price, VC,
FC, operating profit, player share. Links to Settings → Algorithm for
direct coefficient editing.

When any config value changes, `replayAllYears()` re-derives the income
statement and per-country results from stored position snapshots against
the new algorithm (debounced 300ms); Results screen re-renders when
visible. Coefficient changes therefore re-grade completed years without
losing the player's decision history.

## Phase 4 — Competition Mode
Toggle in the Settings modal header. When on:
- Red banner across the top of every screen: `COMPETITION MODE · Y{n} · Cash $X.XM`
- Pulse-dot autosave indicator; `saveConfig()` fires every 10s
- Floating "⚡ Quick Edit" button (bottom-right) opens Settings → Algorithm
- `Cmd/Ctrl+E` → Quick Edit
- `Cmd/Ctrl+S` → `snapshotGame()` — downloads
  `hbr-sim-snapshot-YYYY-MM-DD...json` with full config + decisions + results
  + recommendations for post-competition review
- `Cmd/Ctrl+R` → Deep recommend on the currently-visible decision screen

## Phase 5 — Recommender engine
`recommend(screen, {mode:'quick'|'deep'})` — candidate enumeration + seeded
Monte Carlo rollout + greedy future play.
- Quick mode: 30 rollouts/candidate, target <500ms
- Deep mode: 200 rollouts/candidate with progress indicator
- Cache keyed by `configHash | gameStateHash | screen | mode`; invalidates on
  any config change
- Returns ranked actions with `expectedProfitY6`, `stdDev`, `p10/p50/p90`,
  one-line reasoning, and warnings (e.g., "Political Crisis Y5 will force-
  exit this WOS")

"Coach recommends" card at the top of every decision screen (Market Entry,
Exit, Marketing Overview) shows the top 3 with EoY6 + P10/P90 range, a
"Deep recommend" button, and a "Show all candidates" expandable list.
Auto-refreshes on config change.

## Deliverables
- `index.html` — single file, all phases applied
- `market-entry-v2.html` — snapshot copy of Phase 1.5 for comparison
- `CHANGELOG.md` — this file
