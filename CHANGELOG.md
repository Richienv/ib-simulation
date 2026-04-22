# Changelog

## Phase 1 (commit 0949a90 → latest)

**Data layer extraction.** Introduced `configStore` (pub/sub + debounced localStorage
persistence under `hbr-sim-config-v1`). Every formerly-hardcoded country stat,
entry-mode parameter, game rule, algorithm coefficient, event, and news narrative
is now a single edit surface. Phase 1 details live in commit `0949a90`.

## Phase 1.5 — Canonical Algorithm rewrite (this commit)

**Scope:** Replace the entire profit model per the Canonical Algorithm Spec.
Zero feature-flag preservation of the old model (Q1 decision). Zero behavior
loss on the existing UI flow.

- **`config.algorithm`** — replaced the 16-coefficient legacy table with the full
  Canonical Spec §6 table: `M_base`, `P_base`, `VC_base`, `FC_base_per_outlet`,
  `GDP_USA_ref`, `price_elasticity`, `price_gdp_elasticity`, `promo_max_lift`,
  `promo_half_sat`, `promo_{low,med,high}`, `cultural_fit_{medium,high}`,
  `mode_cultural_mitigation.{F,JV,WOS}`, `rivalry_share_{medium,high}`,
  `adaptation_rivalry_defense`, `adaptation_demand_coef`, `adaptation_cost_uplift`,
  `indulgence.{Low,Medium,High}`, `mode_foreignness.{F,JV,WOS}`,
  `saturation_exponent`, `population_per_outlet_saturation`, `input_inflation`,
  `low_income_discount`, `fc_gdp_elasticity`, `mode_fc_multiplier.{F,JV,WOS}`,
  `HQ_base_overhead`, `HQ_per_market_support`, `HQ_franchise_support`,
  `royalty_rate`, `franchise_initial_fee`, `sourcing_royalty_rate`,
  `entry_fee_per_country`, `opening_fee_{F,JV,WOS}`, `exit_payout`,
  `bonus_{25M,50M}`, `entry_penalty_{threshold,rate}`.
- **Coefficient shape** — every coefficient is now `{value, min, max, pinned,
  rngMin, rngMax, desc}`. `min/max` = editor validation bounds; `rngMin/rngMax`
  = uniform draw range when unpinned. Structured coefficients (per-mode,
  per-category) are nested objects of leaves.
- **`resolveAlgorithm(algo, rng)`** — resolves the nested Coefficient table to a
  flat object of numbers for the hot path. Pinned → `.value`; unpinned →
  `rng.uniform(rngMin, rngMax)`.
- **`computeMarketPnL(ctx)`** — pure function implementing the full Canonical
  Spec §2–§3 model per market per year. Returns profit plus a full trace object
  for the upcoming "Why this?" panel.
- **`computeYearProfit(state, algo, year, eventConfig, meta)`** — aggregates
  `computeMarketPnL` across markets, applies HQ overhead, sourcing royalty,
  over-entry penalty, and handles forced exits.
- **`buildEventsView(cfg, year, positions)`** — new event system returning
  `{demandMultiplierFor, costMultiplierFor, tariffAmpFor, opportunismPenalty,
  forcedExits, sourcingRoyaltyActive, tagsFor, allTags}`. Replaces the old
  `impacts{}` dictionary. Supports event `impact.kind` of `revMult`, `tradeWar`,
  `fixedMult`, `opportunism`, `political`, `sourcingRoyalty`.
- **Deleted:** `countryEconomics()`, `variableCostPerMeal()`, the old 16-coef
  `randomParams()`, `config.rules.variableCost`, `config.rules.sourcingRoyalty`,
  `config.rules.franchiseeBehavior`, `config.rules.firstYearRevenueFactor`,
  `config.rules.wosLiabilityCoef`, `config.rules.brandAwarenessThreshold`,
  `config.rules.profitBonusTiers` (replaced by algorithm `bonus_25M/50M`),
  `config.rules.exitPayout` (replaced by algorithm `exit_payout`, pinned).

**Event-preset system (Q2).**
- Two presets shipped: `spec` (Peak Globalization Y1 · Trade War Y2 · Financial
  Crisis Y3 · Weather Y4 · Opportunism Y4 · Political Y5) and `hbr` (Teaching
  Note schedule: Trade War Y4 · Min Wage Y5 · Political Y5). `spec` is active
  on first load per user decision.
- `applyEventPreset(name)` hot-swaps the events block with whole-object replace.
- `listEventPresets()` returns `['spec','hbr']`.

**Profit bonuses (Q3).** `$5M` at $25M cumulative, `$10M` at $50M cumulative —
per spec §6.

**Decision shape (Q4).** Country Marketing screen now uses segmented radio
pills: Adaptation (Standardized / Partial / Full Local → 0 / 0.5 / 1.0), Price
stance (Aggressive / Match / Premium → 0.80 / 1.00 / 1.20), Promotion per
outlet (Low $100K / Medium $200K / High $400K), Sourcing (Local / Global).
`normalizeDecision()` also accepts the legacy shape (menuLoc / price / promo
/ mix) for back-compat with any lingering reads.

**Sourcing (Q5).** Two-way (Local / Global). Legacy `mix`/`home` collapse to
`global` at decision normalization.

**Population (Q6).** `country.population` added with 2024 World Bank values
(US 333M, UK 68M, DE 84M, FR 68M, JP 125M, CN 1.412B, IN 1.428B, BR 216M,
AU 26M, CA 40M, EG 112M, ZA 62M). Feeds saturation curve.

**Calibration Check panel.** New card on Results screen.
- 9 default scenarios derived from Canonical Spec §9
- Each row: name, inputs, metric, target, tolerance ±%, actual, status (✓/⚠/✗)
- "Add scenario", "Import from HBR" (CSV paste), "Run all", "Auto-run on
  coefficient change" toggle (debounced 400ms on configStore changes)
- "Why?" hints per failing row, drawn from spec §8 calibration playbook
- Aggregate pass-count summary; "Calibrated ✓" when all pass
- Scenarios persist in `localStorage['hbr-sim-calibration-v1']`

**Starting-state calibration.** On fresh load with Spec Default preset:
- 4/9 event-mechanic scenarios PASS (political crisis, sourcing royalty
  active under Peak Globalization, sourcing royalty gone under Trade War,
  over-entry penalty firing, bonus_25M is $5M)
- 5/9 profit-magnitude scenarios FAIL by 4–7× (USA/UK Franchise Y1 overshoot,
  India WOS Y1 undershoot)
- Expected starting state per the approach-(iii) calibration workflow.
  Tune during live competition via the Settings → Algorithm tab.

**Verified headlessly:**
- Y1 2-franchise (UK + US): $7.1M annual, $1.29M sourcing royalty active,
  no entry penalty
- Y1 3-franchise (UK + US + AU): $-17.5M annual, $27.1M entry penalty fires
- Y5 political crisis: CN (WOS) force-exited, UK (F) survives
- Preset swap `spec → hbr` correctly replaces event key set
- localStorage persists and hydrates; invalid/partial configs fall back to
  defaults
