# Sim Config — structure &amp; live-competition runbook

## Structure

Single source of truth: `configStore` in `index.html`. Persisted to
`localStorage['hbr-sim-config-v1']` (debounced 300ms). Shape:

```
SimConfig {
  schemaVersion: 1
  countries: { [code]: CountryConfig }    // 12 countries, editable
  modes:     { F, JV, WOS: EntryModeConfig }
  rules:     GameRulesConfig              // starting cash, total years,
                                          // category thresholds, low-income list,
                                          // political-risk list, home country,
                                          // active event preset
  algorithm: AlgorithmCoefficients        // ~40 coefficients, each a Coefficient
  events:    { [eventId]: EventConfig }   // editable year schedule + impacts
  news:      { 1..6: NewsConfig }         // headline, body, coach playbook
  seed:      { value, frozen }
}

Coefficient {
  value:  number      // default / pinned value
  min, max: number    // editor validation bounds
  pinned: boolean     // if true, `value` is used; if false, drawn from rng range
  rngMin, rngMax: number   // uniform draw range when unpinned
  desc:   string
}
```

## Algorithm — order of operations

For each active market `i` in year `y`:

```
meals_i = M_base
        × growth(gdpGrowth_i, popGrowth_i, y)          // compounds (1+g)^(y-1) each
        × culturalFit(i, mode)                         // distance category × mode_cultural_mitigation
        × rivalryShare(i) × (1 + adaptation_rivalry_defense × adaptation)
        × indulgence(i)                                // by category
        × priceResponse((priceOpt_i / price_i)^|price_elasticity|)   // capped [0.5, 1.5]
        × promoResponse(1 + promo_max_lift × promo/(promo+promo_half_sat))
        × (1 + adaptation_demand_coef × adaptation × distNumeric)
        × mode_foreignness[mode]
        × event_demandMult(i, mode, y)
        × saturation(outlets, population_i / population_per_outlet_saturation)

price_i = P_base × (GDP_i / GDP_USA_ref)^price_gdp_elasticity × priceMultiplier

VC_i   = VC_base
       × (1 + sourcingΔ)                              // sourcingΔ = tariff_i when global+tradeWar, else 0
       × (1 + input_inflation)^(y-1)
       × (1 − low_income_discount if i in low_income)
       × (1 + adaptation_cost_uplift × adaptation)
       × event_costMult(i, y)

FC_i   = FC_base_per_outlet × (GDP_i / GDP_USA_ref)^fc_gdp_elasticity
       × (1 − low_income_discount if applicable)
       × (1 + mode_fc_multiplier[mode])               // F = -1 (franchisee pays), JV = -0.5, WOS = 0

marketRevenue = outlets × meals × price
marketVC      = outlets × meals × VC
marketFC      = outlets × FC
marketPromo   = outlets × promo   (zero for F — franchisee self-funds)
operatingProfit = marketRevenue − marketVC − marketFC − marketPromo
```

Mode-specific player share:

```
F   : playerRevenue = initialFees × newOutlets + royalty_rate × marketRevenue (× opportunism)
      playerCosts   = HQ_franchise_support
JV  : playerRevenue = 0.5 × operatingProfit × opportunism
      playerCosts   = HQ_per_market_support
WOS : playerRevenue = operatingProfit
      playerCosts   = HQ_per_market_support
```

Year aggregate:

```
total  = Σ playerProfit_i
       − HQ_base_overhead
       − entryCostsThisYear
       − growthCostsThisYear
       + sourcing_royalty_rate × Σ_gross (when sourcingRoyaltyActive)
       − entry_penalty_rate × Σ_newRev (when newMarkets >= entry_penalty_threshold)
       + bonus_25M  (first time cum crosses $25M)
       + bonus_50M  (first time cum crosses $50M)
```

`sourcingRoyaltyActive` defaults to `true` in Y1–Y2 AND only when no
Trade War is active. Alternatively, an explicit `peak_globalization`
event can toggle it; Trade War always overrides.

## Live-competition runbook

**Before the session:**
1. Open Settings (gear icon). Export current config as baseline.
2. Switch the event preset to match the facilitator's announced schedule
   (Rules tab → Active event preset, or Import/Export → Load Spec/HBR Default).
3. Toggle Competition mode (Settings header). The red banner + autosave
   pulse should appear.
4. Set seed and freeze if you want deterministic runs.

**During the session:**
1. Observe one HBR output — e.g., "USA Franchise Y1, 25 outlets, standard
   decisions, revealed player profit $0.5M."
2. Press `Cmd/Ctrl+E` to Quick Edit → Algorithm tab.
3. Compare to the Calibration Check panel (Results screen). Add the
   observed row via "Import from HBR":
   `US,F,1,0,match,med,global,25,500000,25`
4. Failed row exposes a "Why?" link with candidate coefficients to tune.
5. Edit one coefficient; the card re-runs (auto-run is on by default).
   Scenarios turn ✓ progressively as you calibrate.
6. Press `Cmd/Ctrl+S` at any point to snapshot the current game state +
   config + recommendations for post-mortem.

**Calibration tuning order (spec §8):**
1. Revenue too high/low → `M_base`
2. Revenue scales wrongly with GDP → `price_gdp_elasticity`, `P_base`
3. Costs off → `VC_base`, `FC_base_per_outlet`, `low_income_discount`
4. Franchise-specific → `royalty_rate`, `franchise_initial_fee`, `HQ_franchise_support`
5. JV specific → share pct (in Modes tab), `mode_fc_multiplier.JV`
6. WOS specific → `mode_foreignness.WOS`, `mode_cultural_mitigation.WOS`
7. Year-on-year drift → `input_inflation`
8. Event-year specific → event modifier values in News &amp; Events tab

**Tip:** pin every coefficient you calibrate before trusting the recommender.
The recommender uses the resolved (draw OR pinned) algorithm for its Monte
Carlo rollouts — unpinned coefficients add variance that can swamp small
tuning signals.
