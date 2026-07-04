# Computed Insights Reference

This document explains the algorithms behind every number
`get_product_details` returns in its `demand`, `seasonality`, and `insights`
blocks: what each metric measures, how it gets computed, what gates it has
to pass, and how to read it as a reseller. It is a companion to `TOOLS.md`
(the practitioner reference for *what* each tool does); this doc covers
*how* the deep-analysis tool turns raw Keepa time-series into the structured
fields you see in the output.

Read this when:

- A confidence label or output value surprises you and you want to know why.
- You want to understand the difference between "we couldn't compute this"
  and "the signal is genuinely weak."
- You're designing prompts or downstream tooling that consumes specific
  fields.

The algorithms live in six families, mapping to specific blocks of the
`get_product_details` output:

| Section | Output block | When it runs |
|---------|--------------|--------------|
| [1. Calibrated demand](#1-calibrated-demand) | `demand` | Every product with sufficient category data |
| [2. Seasonality](#2-seasonality) | `seasonality` | Every product with ≥ 26 weeks of BSR history |
| [3. Pattern detectors](#3-pattern-detectors) | `insights.{sawtooth, raceToBottom, sellerCliff}` | sellerCliff: always; sawtooth + raceToBottom: when offers data is present |
| [4. Stability & trend](#4-stability--trend) | `insights.{demandStability, amazonOOS, priceTrend, priceVelocity, pricePosition, priceCompression, reviewPurge}` | Most always-on; `priceCompression` needs offers |
| [5. Competition signals](#5-competition-signals) | `insights.{buyBoxVolatility, effectiveCompetition, fbaFbmConcentration, sellerConcentration, stockDepth}` | All require offers data |
| [6. Composite risk signals](#6-composite-risk-signals) | `insights.{ipRisk, raceToBottomWarning}` | `ipRisk`: always; `raceToBottomWarning`: with offers data |

**Source of truth.** Numeric constants in this doc are sampled from
`src/core/demand/`, `src/core/product/seasonality/`, and
`src/core/product/insights/` in the Agellic source at the time of
writing (the same engine powers both the full Agellic server and Agellic
Lite). The code remains authoritative, if a calibration changes, this
doc gets refreshed on the next release pass.

**Tone note.** Algorithms are described in plain English, not pseudocode.
If you need exact arithmetic, the source files referenced above are the
right place to look. Output shapes match the TypeScript types verbatim.

---

## 1. Calibrated demand

Every `get_product_details` result carries a `demand` block: a **velocity-implied** monthly-unit-sales estimate, *not* a verified sales figure. The estimator is **Demand Read v3**, a two-regime engine that turns a product's sales rank, rank-drops, and review velocity into a monthly-sales *range*. A range (rather than a point estimate) is honest about spread: the per-product signals Keepa exposes fan out across an order of magnitude even inside a tight category slot, so a single number would imply precision the data doesn't support.

The block is a discriminated union on `mode` with three members:

- **`read`**: the model produced an estimate (a range plus a plan-on number).
- **`badge`**: Amazon itself reports an "X+ bought past month" figure; that *is* the demand, so the model is suppressed and the badge value is echoed.
- **`no-read`**: nothing usable; a `reason` says why.

`mode` is the discriminant: it gates which other fields are present. Read it first.

### Output shape

**`read`**: `{ mode: 'read', source, confidence, centerLikely, rangeLow, rangeHigh, loSnap, hiSnap, hero, agree, clamped, revRollupScreened, cellKey, resolvedCategoryName, resolvedCategoryDepth, categoryVia, caveats }`

- **`source`**: `in-domain` (US/CA, backed by calibrated cell + multiplier tables) or `cross-marketplace`. In practice a `read` is always `in-domain`; unsupported domains demote to `no-read` (see Cross-domain fallback).
- **`centerLikely`**: the conservative **plan-on** monthly number. Regime A: the cell median (`soldP50`, a true centre). Regime B: the *floor* of the bracket (not a centre), so a slow mover plans on its honest low end.
- **`rangeLow` / `rangeHigh`**: numeric monthly-unit bounds (round-to-nearest). **No 50-unit floor**: a Regime B read can be single-digit (e.g. `~3`).
- **`loSnap` / `hiSnap`**: the bounds snapped *down* to a display ladder (`50, 100, 150, 200, 300, 500, 700, 1K, 1.5K, 2K`, with a `2K+` cap and bare integers below 50). At a grid edge the snapped label can sit one bucket below the rounded number, intentional (snapping always rounds conservative).
- **`hero`**: ready-to-print string built from the snaps: `~lo - hi`, collapsing to `~lo` when both snaps match, or `2K+` at the cap.
- **`confidence`**: `high` / `medium` / `low` (see Confidence assignment).
- **`agree`**: the two independent velocity reads (rank-drops and reviews) corroborated each other.
- **`clamped`**: the band's rank ceiling capped a bound (raw velocity implied more sales than the product's rank supports).
- **`revRollupScreened`**: review velocity looked like a variation-family rollup (inflated by sibling ASINs) and was excluded from the read.
- **`cellKey`**: the resolved cell slot `"<bsrBand>|<dropsBand>"`, or null when no cell anchored the read. This encodes the **rank-scale (cell) anchor**.
- **`resolvedCategoryName` / `resolvedCategoryDepth` / `categoryVia`**: the **velocity anchor** category and how it was chosen (see Category resolution). Depth is `0` (root) / `1` / `2`.
- **`caveats`**: free-text qualifiers to surface verbatim (always the velocity-implied disclaimer; plus rank-ceiling cap, rollup exclusion, unknown-categories, etc.).

**`badge`**: `{ mode: 'badge', badgeMonthlySold, caveats }`. `badgeMonthlySold` is Amazon's reported "X+ bought past month" value (also at `sales.monthly.value`).

**`no-read`**: `{ mode: 'no-read', reason, caveats }`. `reason` is one of seven values (see Mode: no-read); `caveats[0]` is the human-readable explanation.

### Mode selection

The resolver runs a fixed order; the first applicable branch wins.

1. **Badge short-circuit (all domains, no I/O).** If Amazon reports any "X+ bought" badge (`sales.monthly.value !== null`), return `badge` immediately: a reported number is ground truth at any magnitude, and the model is suppressed so no competing number sits beside it.
2. **Cross-marketplace fallback.** If the product's Keepa domain isn't US or CA (the only domains with first-class cell data), return `no-read` carrying at most a rough velocity hint in `caveats`, never a structured number (see Cross-domain fallback).
3. **Resolve candidate categories** from the product's `salesRanks` + Keepa breadcrumb. None resolvable → `no-read` (`unknown-categories` if every category was unknown to the demand tree, else `rank-missing`).
4. **Pick the anchors** (cell vs velocity, see Category resolution) and assemble the per-category constants by walking each category chain `subcat → parent → root`.
5. **Run the engine** on `(constants, bsr, drops30, reviewsAdded30)`.
6. **Map the engine result to a mode**: a signal (High/Medium/Low) → `read`; `stalled` / no-signal / out-of-range / no-rank → `no-read` with the matching reason.

### Category resolution (which category anchors the read)

Demand Read uses **two** category anchors, which usually coincide but can diverge:

- The **cell / rank-ceiling anchor** is the product's `salesRanks` resolution. The cell estimate and the per-band rank ceiling are absolute-rank-scale numbers, so they *must* stay on the category that produced them. This anchor is reflected in `cellKey`.
- The **velocity anchor** drives the scale-free multipliers (sales-per-drop, sales-per-review) and the review-rollup median. Because those are ratios, not rank-scale numbers, the velocity anchor can sit on a *deeper* category than the cell anchor. It's reported in `resolvedCategoryName` / `resolvedCategoryDepth`, with `categoryVia` recording how it was chosen.

**`categoryVia`:**

- **`sales-rank`** (the default): the velocity anchor is the same `salesRanks`-derived category as the cell anchor.
- **`breadcrumb`**: the `salesRanks` anchor landed on a **root** department (depth 0), so the velocity anchor was re-anchored onto the deepest **same-root** category carried by the product's Keepa breadcrumb (`categoryTree`). This targets pruned-leaf items (e.g. collectibles) whose `salesRanks` only resolve to a top-level department: anchoring velocity on the whole department understates the specific subcategory. The re-anchor fires **only** from root and **only** adopts a strictly-deeper, same-root category, so every already-deep or multi-root product is unaffected (stays `sales-rank`). The cell / ceiling anchor is never moved, only the velocity multipliers follow the breadcrumb.

When the two anchors diverge, `cellKey` (rank-scale) and `resolvedCategory*` (velocity) surface both, so you can see exactly what produced the number.

### Mode: read, the two-regime engine

The engine routes every product through exactly one of two regimes, decided by whether the matched cell is a trustworthy anchor.

A cell is a **usable anchor** when it passes the cell gate (`floorShare < 0.5` and `nUnique ≥ 5`) *and* its median sits above Keepa's badge floor (`soldP50 > 50`). That floor of 50 is Keepa's lowest monthly-sold tier, at or below it the cell is *censored* ("≤50, can't tell"), so it can't anchor a number.

**Regime A: uncensored cell (the cell IS the read).**

- `centerLikely = soldP50` (capped at the band rank ceiling); the range is the cell's empirical IQR `[soldP25, soldP75]`.
- The two velocity reads only set *confidence*, they never move the number. The range stays the IQR even when the velocities disagree (disagreement caps the *label*, not the bracket).
- Confidence is **High** when both velocity reads are live, agree with each other *and* corroborate the cell median (each within a 2× factor), and the category's multiplier spread isn't `wide`; otherwise **Medium**. Regime A never returns Low.

**Regime B: censored / blind cell (velocity-only, no floor).**

The censored "50" is discarded and the read becomes a real bracket built from the product's own velocity × the category's measured multipliers. There is **no** 50-unit floor, a one-a-month mover reads `~1`.

- **Drops + reviews both live:** the bracket floor is the larger of `drops30` and the smaller of the two **P25** velocity estimates; the ceiling is the larger of the two **P50** estimates (per-drop = `drops30 ×` the category sales-per-drop rate; per-review = `reviewsAdded30 ×` the sales-per-review rate). If the two P50 estimates diverge past **5×**, that's a conflict (one signal is lagging or noise) and the read collapses to the drops-only bracket `[drops30, drops30 × sales-per-drop P50]`, a divergent review can't pull the ceiling below the honest median the drops alone imply.
- **Drops only:** `[drops30, drops30 × sales-per-drop P50]`.
- **Reviews only (not rollup-screened):** `stalled`, reviews lag sales, so recent reviews with no rank movement most likely reflect *past* demand surfacing late, not current sales. Reported as `no-read` / `stalled`.
- **Neither, or reviews-only but rollup-screened:** `no-read` / `no-clear-read`.
- `centerLikely` is the bracket **floor** (the conservative plan-on number). `drops30` is a hard mechanical floor, each rank-drop implies ≥ 1 sale, so the lower bound never sits below it, and a category with no measured sales-per-drop rate values each drop at exactly 1 (understates, never overstates).
- Confidence maxes at **Medium** (when the two reads agree within 2× *or* drops are plentiful, ≥ 5); otherwise **Low**. Regime B never returns High.

**How to read it.**

- `centerLikely` is what to plan on: the typical volume in Regime A, a deliberately conservative low end in Regime B.
- `rangeLow` / `rangeHigh` bracket the realistic spread; real products sell at both ends.
- `agree === true` with `confidence: high` is the strongest corroboration, treat the number accordingly.
- `clamped === true` means the rank ceiling pulled a bound down: the raw velocity implied more sales than the product's rank supports.

### Mode: badge

When Amazon shows an "X+ bought past month" badge, that figure *is* the demand and the model steps aside: `badgeMonthlySold` echoes the value (the badge itself is at `sales.monthly.value`) and no model range is computed. This holds at any magnitude and stays correct if Amazon's display floor ever drops below 50. Treat the badge as ground truth.

### Mode: no-read

`no-read` means the lookup ran but produced no usable range. `reason` is one of:

- **`no-clear-read`**: the engine ran but no signal was live (no drops, no usable reviews).
- **`stalled`**: recent reviews but no rank movement; likely stale past sales, not current demand (reviews lag sales). A distinct, informative state, not the same as "no signal."
- **`rank-out-of-range`**: BSR beyond 300,000; too thin to read.
- **`rank-missing`**: no resolvable BSR / candidate category.
- **`unknown-categories`**: every candidate category was unknown to the demand tree.
- **`no-activity`**: a cross-marketplace product with no drops and no reviews.
- **`cross-marketplace`**: an unsupported domain; only a rough extrapolated hint in `caveats`, never a number.

(Distinct from `logical.demand === null`, which means the lookup threw or wasn't run at all.)

### Confidence assignment

Confidence is regime-bounded, so the label itself tells you which kind of estimate you're reading:

- **Regime A** (cell-anchored): **High** or **Medium**, never Low. High requires both velocity reads live, agreeing with each other *and* corroborating the cell median (all within a 2× factor), with a non-`wide` category. Anything less is Medium.
- **Regime B** (velocity-only): **Medium** or **Low**, never High. Medium when the drop and review reads agree within 2× *or* drops are plentiful (≥ 5); otherwise Low.

So `high` ⇒ a corroborated cell anchor, `low` ⇒ a thin velocity-only read, and `medium` sits between (a solid cell without full corroboration, or an agreeing / plentiful velocity read).

### Key constants

Sampled from `src/core/demand/reference-model.ts`; the code remains authoritative.

- `BSR_MAX = 300,000`: above this BSR, `rank-out-of-range`.
- `DEMAND_SALES_FLOOR = 50`: Keepa's lowest monthly-sold tier and the Regime A/B selector (the cell median must exceed it to anchor). Since the sub-50 fix it is **no longer a floor** on the output, nothing is clamped up to it.
- `AGREE_F = 2.0`: the "agree / corroborate" factor (a ratio ≤ 2× counts as agreement).
- `SPREAD_COLLAPSE = 5.0`: the Regime B conflict threshold (collapse to the drops-only bracket).
- `DROPS_PLENTIFUL = 5`: drops at/above which a one-sided or collapsed Regime B read earns Medium.
- Display ladder: `50 · 100 · 150 · 200 · 300 · 500 · 700 · 1K · 1.5K · 2K`, capped at `2K+`, bare rounded integers below 50.

### Cross-domain fallback

Calibrated cell + multiplier tables exist only for **US (domain 1)** and **CA (domain 6)**. Every other Keepa domain (UK, DE, FR, JP, IT, ES, IN, MX, …) lacks first-class data, so the resolver short-circuits before any in-domain work and **never emits a structured number** for those domains: a confident-looking figure off coarse constants is exactly the over-buy failure mode the engine is built to avoid.

Instead it returns `no-read`:

- with `reason: 'cross-marketplace'` plus a rough hint caveat (a `rough cross-marketplace velocity hint` of `~N/mo`, flagged as an extrapolated baseline rather than a sales figure) when there is drops or review activity, sized off CA bottom-quartile multipliers (one step under CA's median, itself ~50% of US, asymmetric conservatism: lean low);
- with `reason: 'no-activity'` when there are no recent drops or reviews.

Every cross-domain output also carries `estimate from cross-marketplace baseline; we lack data for this domain`. When a third domain (likely UK) gets calibrated cell data, these constants get re-derived.

---

## 2. Seasonality

Replaces an older autocorrelation-based detector that almost never fired on real Keepa series (autocorrelation needed long, dense, gap-free histories and produced no actionable peak window when it did trigger). The replacement is a two-layer calendar-Z + template approach: Layer A bins weekly BSR into 52 calendar buckets and finds the run of weeks that systematically beats a robust off-peak baseline; Layer B labels that run against 14 known calendar patterns (Q4, Halloween, BackToSchool, etc.). A third step does phase math: given the detected peak window and today's date, when is the next peak, when should the reseller be buying, and when should they be exiting.

The output tells a reseller four things: whether the product is seasonal at all, what shape the season takes, where in the cycle we are right now, and concrete buy/exit dates anchored to the next peak.

### Output shape

`seasonality: { detected, method, pattern, peak: {...}, yearsObserved, weeksAnalyzed, confidence, confirmationLevel, currentPhase, daysUntilNextPeakStart, sourcingWindow, leadOutByDay, summary }`

- **`detected`**: `true` only when a peak window passes the baseline + ratio + cross-year-confirmation gates.
- **`method`**: `'calendar-z'` when detected, `null` otherwise. (The `'autocorrelation'` value is retired and never emitted.)
- **`pattern`**: `'spike'` (≤ 4-week peak), `'shoulder'` (5+ weeks), `'two-tail'` (peak wraps Dec→Jan), or `'evergreen'` (no peak after a full year of data).
- **`peak.label`**: one of the 14 template names (see below) or `'Other'`. `null` when not detected.
- **`peak.startWeekOfYear` / `endWeekOfYear`**: 1–52, inclusive. If `end < start`, the peak wraps the year boundary.
- **`peak.durationWeeks`**: counts wraparound correctly.
- **`peak.peakToOffPeakRatio`**: peak-bucket median BSR ÷ off-peak baseline. < 1 means peak window has lower BSR (better demand); the gate rejects ≥ 0.85.
- **`yearsObserved`**: distinct calendar years with ≥ 26 weeks of data.
- **`weeksAnalyzed`**: non-null weekly observations.
- **`confidence`**: `'low'` / `'moderate'` / `'high'`. See decision rule under Layer A.
- **`confirmationLevel`**: `'none'` / `'candidate'` / `'confirmed'` based on YoY peak alignment.
- **`currentPhase`**: `'pre-peak'` / `'peak'` / `'post-peak'` / `'off-season'`, or `null` if not detected.
- **`daysUntilNextPeakStart`**: integer days from today to the next peak's start. `null` if not detected.
- **`sourcingWindow`**: `{ earliestByDay, latestByDay }` as ISO dates, or `null`.
- **`leadOutByDay`**: ISO date to exit by, or `null`.
- **`summary`**: one-line human-readable digest of the above.

### Layer A: calendar-Z peak detection

**What it does.** Identifies the contiguous run of weeks-of-year where weekly BSR systematically beats a robust off-peak baseline, then confirms the run against per-year repeat.

**Method.** Weekly BSR observations are bucketed into 52 calendar slots by week-of-year: for each (year, week) the per-year median is taken first (resists within-year outliers), then the cross-year median across years gives one number per week-of-year. An off-peak baseline is built by taking the global median of the 52 buckets, then keeping only buckets within ±0.5 robust-MAD of that median; the baseline `B` is the median of that filtered set and σ is `1.4826 × MAD` of the same set, floored at 1% of `B` and at 1 to keep Z-scores numerically meaningful. Per-week Z-scores are `(bucket − B) / σ`. A peak run is any contiguous span of weeks with `Z ≤ -Z_THRESHOLD`, with single one-week gaps bridged so noisy holes don't fragment a real peak. Runs are allowed to wrap from week 52 → week 1. The primary peak is the run with the lowest (most negative) mean Z. The same Z procedure is rerun per calendar year that has enough data; if at least 2 distinct years' per-year peaks align with the cross-year primary within ±2 weeks on both ends, the result is `confirmed`. A `peakToOffPeakRatio` gate then rejects "peaks" that are only 15% better than baseline as noise.

**Constants.**
- **`Z_THRESHOLD = 1.0`**: a week qualifies as "peak" when its Z-score is ≤ -1.0 (lower BSR = better demand).
- **`HIGH_CONFIDENCE_Z = -1.5`**: primary peak's mean Z must be below this for `high` confidence.
- **`MIN_DETECTION_RATIO = 0.85`**: peak-median ÷ baseline must be ≤ 0.85 (peak ≥ 15% better than baseline) or the candidate is rejected as noise.
- **`MAX_PEAK_DURATION = 26`**: runs longer than half a year are rejected (likely a generally well-ranked product, not a season). Sized to still accommodate ColdFlu (wk 40 → wk 12, 25 weeks).
- **`MIN_PEAK_DURATION = 2`**: single-week spikes are rejected (PrimeDay templates handle the legitimate single-week case separately).
- **`MAX_GAP_WEEKS = 1`**: a lone false sandwiched between two trues is bridged so a real peak isn't split by one noisy week.
- **`MIN_BUCKETS_FOR_BASELINE = 26`**: fewer than 26 of 52 calendar weeks populated → no baseline computed, no detection.
- **`MIN_WEEKS_PER_YEAR_FOR_CONFIRMATION = 26`**: a calendar year contributes to YoY confirmation only with ≥ 26 weeks of data.
- **`ALIGNMENT_TOLERANCE_WEEKS = 2`**: a per-year peak counts as aligned with the primary when both start and end are within ±2 weeks (circular distance).

**Confirmation levels.** `none`: no calendar year produced its own peak. `candidate`: at least one year produced a peak, but fewer than 2 of them aligned with the cross-year primary. `confirmed`: 2+ distinct years' peaks aligned within ±2 weeks. `detected` flips to `true` only when `confirmationLevel` is `candidate` or `confirmed`. Confidence then resolves to `high` when `yearsObserved ≥ 3` AND primary mean Z is below -1.5 AND confirmed; `moderate` for confirmed with weaker numbers; `low` for `candidate` or `none`.

**Caveats.** Requires `seriesStartMs` ≥ 2000-01-01: bad/missing series origin returns the not-detected fallback rather than scoring against year 1970. Needs ≥ 26 populated calendar buckets to compute a baseline at all. The baseline routine has an inversion guard: if the global median is sitting inside the peak cluster (peak-saturated products), it refuses to compute. Pattern classification needs ≥ 52 weeks of data; with sparse data a real peak run still bails to not-detected rather than emit a labeled peak with `pattern: null`.

### Layer B: 14 pattern templates

**What it does.** Labels the detected peak window against known calendar windows so the reseller sees "Q4" or "Halloween" instead of "weeks 44–52."

**Templates.** All 14:
- **Q4** (wk 44–52): holiday season.
- **Halloween** (wk 38–43): costumes, decor.
- **BackToSchool** (wk 28–34): supplies, dorm goods.
- **PrimeDay** (wk 28 OR wk 41, spike-only): modeled as two single-week templates; July Prime Day + October Big Deal Days.
- **Summer** (wk 20–35): broad summer goods.
- **FathersDay** (wk 24 only).
- **MothersDay** (wk 18–19).
- **Easter** (wk 12–17).
- **Spring** (wk 10–22): broad spring goods.
- **TaxSeason** (wk 6–16): tax-prep adjacent.
- **Valentines** (wk 5–7).
- **NewYearFitness** (wk 1–6).
- **ColdFlu** (wk 40 → wk 12, two-tail-only): wraparound winter respiratory.
- **Other**: fallback when nothing meets the Jaccard threshold.

**Method.** Pattern shape is decided first: `evergreen` (≥ 52 weeks of data, no peak), `two-tail` (peak wraps year boundary, any duration), `spike` (≤ 4-week peak), or `shoulder` (5+ week non-wrapping peak). Label assignment then compares the detected peak window against each template via Jaccard overlap (intersection ÷ union of week-of-year sets, wraparound-aware). Templates with a `patternRequirement` gate (PrimeDay → `spike`, ColdFlu → `two-tail`) are skipped when the detected pattern doesn't match. The template with the highest Jaccard score ≥ 0.5 wins; ties go to the narrower template.

**Caveats.** Below 0.5 Jaccard, the label falls to `'Other'`, a real peak still detected, just not matching the calendar library. Templates are US-marketplace-shaped (BackToSchool, TaxSeason, PrimeDay dates assume the US calendar). Single events Amazon does not pre-announce a calendar week for (e.g. spontaneous Lightning Deals) cannot match. PrimeDay's two single-week templates pick whichever year-portion the detected peak overlaps better; the loser is not separately reported.

### Phase math (currentPhase / sourcingWindow / leadOutByDay)

**What it does.** Given the detected peak window (week-of-year start/end) and today's date, classify where we are in the cycle and emit ISO dates for when to buy and when to exit.

**Method.** Today's week-of-year is computed from UTC day-of-year. If today is inside the peak window we're in `peak`; otherwise circular distance from today to peak start (and from peak end to today) decides `pre-peak`, `post-peak`, or `off-season` against an 8-week window. The next peak's start and end dates are anchored to the appropriate calendar year: for wraparound peaks, weeks in the head belong to next year's start while weeks in the tail belong to the previous year's start that already ran. From the next-start date, sourcing dates are shifted back 8 weeks (earliest) and 6 weeks (latest); the lead-out date is shifted back 4 weeks from the next-end date.

**Outputs.**
- **`currentPhase: 'pre-peak' | 'peak' | 'post-peak' | 'off-season'`**, `peak` when today falls inside `[startWeekOfYear, endWeekOfYear]`. `pre-peak` when peak starts within the next 1–8 weeks. `post-peak` when peak ended within the last 1–8 weeks. `off-season` otherwise.
- **`daysUntilNextPeakStart: number | null`**, integer days from today to the next peak start, rounded.
- **`sourcingWindow: { earliestByDay: ISO, latestByDay: ISO } | null`**, buy-by-here-or-you-miss-it. Earliest = next-peak start − 8 weeks; latest = next-peak start − 6 weeks.
- **`leadOutByDay: ISO | null`**, stop-replenishing date. Next-peak end − 4 weeks.

**How to read it.**
- `currentPhase: 'pre-peak'` + today between `earliestByDay` and `latestByDay` → buy now.
- `currentPhase: 'peak'` + today past `leadOutByDay` → stop replenishing; sell through.
- `currentPhase: 'off-season'` + `daysUntilNextPeakStart` > 60 → no urgency; revisit when pre-peak window opens.

**Caveats.** Wraparound peaks (e.g. ColdFlu, wk 40 → wk 12) need special-case year arithmetic: the tail of a wrap-peak belongs to the previous year's start. Lead times are hardcoded (8/6/4 weeks); they aren't tuned per pattern. Air vs. ocean freight isn't a factor: the reseller is expected to back the dates out further if their lead time is longer than 8 weeks.

---

## 3. Pattern detectors

Pattern detectors are shape-based, not statistic-based. They walk the raw event-transition series, identify discrete features (peaks, valleys, sustained slopes, single-step drops), and gate on those features rather than on a summary statistic like mean or standard deviation. A flat 90-day average can hide three undercut cycles or a 60% seller exit; these three detectors surface what the averages flatten. All three return a `detected` boolean plus a structured payload: read the boolean first, then the supporting fields.

### Sawtooth pattern

**What it measures.** Cyclical undercutting in the lowest 3P-New price: repeated peak-to-valley drops where price falls, recovers, falls again on a roughly stable cadence.

**Why it matters.** A regular sawtooth means competitors are re-undercutting on a predictable rhythm; you can either avoid the listing, time entries to the peak, or hold offers above the valley floor instead of chasing it down each cycle.

**Method.** Reads the **lowest 3P-New** price series rather than Buy Box because Buy Box rotates between sellers and obscures the actual price floor: sawtooth is fundamentally about *where the cheapest 3P offer sits*, not who's winning the box. Detects plateau-aware local peaks and valleys, then pairs each valley with the most recent peak that follows the prior valley (alternation enforced: prevents a monotonic staircase from registering as multi-tooth). Each peak-valley pair becomes a "tooth" if it clears both an amplitude floor and a cents floor. Cycle period is computed from valley-to-valley spacing on qualifying teeth only; regularity is the coefficient of variation of those cycle lengths. Optionally correlates each valley with a contemporaneous seller-count increase.

**Constants.**
- `SAWTOOTH_MIN_TEETH = 3`: need three qualifying peak/valley pairs.
- `SAWTOOTH_MIN_DROP_PERCENT = 10`: each tooth's peak-to-valley drop must be ≥ 10%.
- `SAWTOOTH_MIN_DROP_CENTS = 20`, and > 20 cents in absolute terms (filters penny noise on cheap items).
- `MIN_OBS_SAWTOOTH = 5`: minimum non-null price observations to attempt detection.
- Regularity gate: cycle-length CoV < 0.3 → `regular`, otherwise `irregular`.
- `SAWTOOTH_MAX_WINDOW_DAYS = 60`: adaptive seller-correlation lookback, sized as `min(60, max(15, daysAnalyzed/6))`.

**Output shape.**
```
{ detected, teethCount, avgCycleLengthDays, avgDropPercent,
  avgResetPercent, regularity: 'regular' | 'irregular' | 'none',
  correlatedWithSellerEntry, confidence: 'high' | 'moderate' | 'low',
  daysAnalyzed, summary }
```
`confidence` is `high` only when both `correlatedWithSellerEntry` AND `regularity === 'regular'`; `moderate` with correlation alone; `low` otherwise.

**How to read it.**
- `detected && regularity === 'regular' && correlatedWithSellerEntry`: a recurring undercut cycle driven by seller entry. Time bids to the peak; do not chase the valley.
- `detected && regularity === 'irregular'`: undercutting exists but is not on a clock; expect drops without being able to predict them.
- `avgResetPercent` materially less than `avgDropPercent`: price floor is drifting down between cycles. Cross-check with raceToBottom.

**Caveats.** Needs ≥ 5 non-null observations and ≥ 3 qualifying teeth to fire; thin-history ASINs return `detected: false` with an "insufficient data" summary. Sawtooth and raceToBottom can co-fire: oscillation overlaid on a downward trend. When both fire, the trend dominates the prognosis: the sawtooth tells you the *mechanism* (recurring entry + undercut), raceToBottom tells you the *direction*.

### Race to bottom

**What it measures.** A sustained downward price war on the lowest 3P-New series over a multi-month window, qualified by whether seller count is holding or growing while price drops.

**Why it matters.** Falling price with stable-or-growing sellers means the floor is moving against you, not stabilizing: margin assumptions made on today's snapshot will be wrong in 30 days.

**Method.** Reads the **lowest 3P-New** price (same rationale as sawtooth: Buy Box rotation masks the floor) paired with the seller-count series, daily-binned and aligned so dense seller-count clusters don't bias the regression. Computes the raw observed decline from first to last paired observation (no annualization: annualizing short windows produced false "severe" ratings on seasonal products). Seller-count trend is from a linear regression: > +15% change → `increasing`, < -15% → `decreasing`, else `stable`. Detection requires a minimum 84-day span, a meaningful decline, and sellers not exiting (a `decreasing` seller trend disqualifies: that's not a race-to-bottom, that's market abandonment).

**Constants.**
- `RACE_TO_BOTTOM_MIN_DECLINE_PERCENT = 5`: main detection threshold.
- `RACE_TO_BOTTOM_WARNING_PERCENT = 3`: soft warning band (3–5%).
- Minimum span: 84 days (12 weeks) of paired observations.
- Minimum paired observations: 3.
- Seller-trend bands: ±15% change over span.
- Severity bands (raw observed decline): ≥ 15% `severe`, ≥ 10% `moderate`, ≥ 5% `mild`, ≥ 3% `warning`.

**Output shape.**
```
{ detected, severity: 'none' | 'warning' | 'mild' | 'moderate' | 'severe',
  priceDeclinePercent, sellerCountTrend: 'increasing' | 'stable' | 'decreasing',
  supplementaryMinima, daysOfDecline, summary }
```
`supplementaryMinima` is a count of successively-lower local price minima, a secondary confirmation signal independent of the regression.

**How to read it.**
- `severity: 'severe' | 'moderate'` with `sellerCountTrend: 'increasing'`, classic race-to-bottom; new entrants pricing in below the floor. Avoid or model with a steeply-declining price assumption.
- `severity: 'warning'`, 3–5% decline only; treat as watchlist, not avoid.
- `sellerCountTrend: 'decreasing'` forces `detected: false` regardless of price drop: the price is falling because sellers exited, which is a different pattern (often a clearance or distress signal) and should be checked against sellerCliff.

**Caveats.** Anything under 84 days of paired data returns `detected: false` with `severity: 'none'`: short-history ASINs can't be assessed. Interplay with sawtooth: a sawtooth with a drifting floor will show as sawtooth `detected: true` AND raceToBottom `detected: true`; that combination is worse than either alone because the cyclical undercutting is *also* ratcheting the floor down each cycle.

### Seller cliff

**What it measures.** Discrete, large drops in seller count: single events where the listing's seller count fell ≥ 40% in one observation step (≤ 45 days apart) or ≥ 50% across two consecutive steps (≤ 60 days apart).

**Why it matters.** A major seller exiting can be an opportunity (less competition, room to win the Buy Box) or a warning (they know something you don't: IP claim, supplier issue, demand collapse). The recovery flag tells you which.

**Method.** Walks the `sellerCountEvents` raw transition series and at each step compares current vs. previous seller count. Flags a single-step cliff if the drop exceeds the single-step threshold within the adjacency gap; if not, checks the two-step cumulative drop within the wider gap. Requires the pre-cliff count to be ≥ 5 sellers (so a 4 → 2 listing doesn't register). For each cliff event, looks ahead up to 56 days to see whether seller count recovered to ≥ 80% of the pre-cliff level. Skips two indices after a detected cliff to avoid double-counting the same drop.

**Constants.**
- `CLIFF_SINGLE_WEEK_THRESHOLD = 0.4`: 40% drop in a single observation step.
- `CLIFF_TWO_WEEK_THRESHOLD = 0.5`: 50% cumulative drop across two steps.
- `SINGLE_CLIFF_MAX_GAP_DAYS = 45`: single-step adjacency cap (Keepa seller-count transitions are sparse).
- `TWO_OBS_CLIFF_MAX_GAP_DAYS = 60`: two-step cumulative cap.
- `CLIFF_RECOVERY_WINDOW = 8` weeks (`RECOVERY_WINDOW_DAYS = 56`): recovery lookahead.
- `CLIFF_RECOVERY_THRESHOLD = 0.8`: recovery requires reaching 80% of pre-cliff count.
- `MIN_SELLER_COUNT_FOR_CLIFF = 5`: pre-cliff seller floor.
- `MIN_OBSERVATIONS_SELLER_CLIFF = 3`: need before/after context to detect at all.

**Output shape.**
```
{ detected, events: Array<{ timestampMs, dropPercent, fromCount, toCount, recovered }>,
  mostRecentDaysAgo, riskLevel: 'none' | 'low' | 'medium' | 'high', summary }
```
Risk is recovery-first: if the most recent event `recovered`, risk is `low` regardless of recency; otherwise < 28 days ago → `high`, < 84 days ago → `medium`, else `low`.

**How to read it.**
- `riskLevel: 'high'` with `events[-1].recovered === false` and `mostRecentDaysAgo < 28`: a recent exit that hasn't been backfilled. Investigate before sourcing; check IP risk, supplier availability.
- `events[-1].recovered === true`: sellers came back. The cliff was a transient event (often a stockout cascade), not a structural exit. Safer to act on.
- Multiple events across the window: chronic instability in seller participation; the listing has structural issues attracting and keeping sellers.

**Caveats.** Keepa seller-count transitions are sparse (typical gaps of 7–90 days), which is why the adjacency caps are day-based and generous rather than weekly. Listings with < 3 non-null seller-count observations return `detected: false` with an "insufficient data" summary, common for short-history or low-traffic ASINs. The `MIN_SELLER_COUNT_FOR_CLIFF = 5` floor means low-competition niches (3–4 sellers steady) won't fire this detector even on a real exit; cross-reference with the `competition` block's `avgSellerCount` to know whether silence here is absence-of-signal or absence-of-data.

---

## 4. Stability & trend

This section groups the seven algorithms that score the "shape" of price, rank, and supply over time: how stable demand has been, what direction price is moving, where today's Buy Box sits inside the historical band, whether competing offers are converging, and how often Amazon itself goes dark on a listing. Each algorithm runs on the per-ASIN Keepa time series after promo spikes are excluded, and most operate on a 90- or 180-day window so recent behavior dominates the label. Together they form the trend half of `insights`, the volatility/direction layer that complements the point-in-time pricing and competition fields.

### Demand stability

**What it measures.** Detrended coefficient of variation of sales rank over the window: how much weekly BSR jitters around its own trend line, not its absolute level.

**Why it matters.** A stable rank means predictable sell-through; an erratic rank means the listing is fragile to competitor activity, OOS, or seasonality and your inventory math is less reliable.

**Method.** Filters the weekly rank series to positive values, log-transforms so the metric is scale-invariant across BSR bands, then detrends in log space so a steady upward or downward drift doesn't count as instability. Uses median absolute deviation (MAD) scaled by 1.4826, the normal-distribution consistency constant, instead of standard deviation, giving a 50% breakdown point so one bad week can't blow up the score. Divides scaled MAD by the absolute log mean to produce a unitless stability index, then maps to a five-tier label.

**Constants.**
- `MIN_WEEKS_DEMAND_STABILITY = 12`: minimum positive weekly observations or returns `insufficient_data`.
- MAD scaling factor `1.4826`: Fisher consistency constant for normal distribution.
- Label thresholds on log-space MAD CoV: `< 0.01 = very_stable`, `< 0.03 = stable`, `< 0.10 = moderate`, `< 0.18 = volatile`, `≥ 0.18 = erratic`.

**Output shape.**
```ts
{
  stabilityIndex: number;  // rounded to 3 decimals
  label: 'very_stable' | 'stable' | 'moderate' | 'volatile' | 'erratic' | 'insufficient_data';
  summary: string;
}
```

**How to read it.**
- `very_stable` / `stable` paired with a healthy `demand.centerLikely` is the textbook "boring winner": buy with confidence on the demand side.
- `volatile` / `erratic` means the BSR is swinging through orders of magnitude; treat the calibrated monthly-sales range as wide and weight `rangeLow` heavily when sizing a buy.
- `insufficient_data` simply means fewer than 12 positive weekly observations are available, usually a new listing or a recently-relaunched ASIN.

**Caveats.**
- Log-transform requires positive BSR; zero or negative weeks are dropped before the count check.
- Detrending absorbs linear drift but not seasonal cycles: strongly seasonal ASINs can register as `volatile` even when the seller knows the pattern.
- Operates on weekly medians, so intra-week spikes are already smoothed out before this metric sees them.

### Amazon OOS

**What it measures.** How often Amazon-as-a-seller is out of stock on this listing: both the time-weighted percentage and the rhythm of restock events. The metric has two independent paths: a derived calculation from raw price transitions on the Amazon-only price stream, and Keepa's own native OOS percentages.

**Why it matters.** Amazon's presence is the single biggest pricing force on most listings: when Amazon goes OOS, third-party sellers can sustain higher prices; when Amazon restocks, margins compress. A `seasonal` or `chronic` classification is a structural pricing opportunity; `none` means Amazon is the permanent ceiling.

**Method.** Two-path computation that the function combines at the end.
- *Derived path:* Takes the raw Amazon price transitions, converts them to in-state intervals via `transitionsToIntervals` (last interval extends to `windowEndMs`), then runs `timeWeightedPercent` over intervals where value is `null` (the OOS sentinel). Walks the interval list to build a list of contiguous OOS events with start, end, and duration. Computes inter-event gap CoV to decide whether restocks are regular (seasonal) or irregular (intermittent).
- *Native path:* Keepa publishes `oos30`, `oos90`, `oos180` percentages per ASIN. When the request is windowed (180-day), the algorithm prefers `oos180`; unwindowed, it prefers `oos90`. The pick is rounded and clamped to `[0,100]`.
- *Combine rule:* If both exist and `|derived - native| ≥ 40` (or one is 0 and the other ≥ 50), native overrides: classification is re-derived from the native percentage, event metrics are zeroed (they contradict the overridden percentage), and the summary discloses the override. Smaller mismatches (≥ 15) keep the derived numbers but mention the native value. The `seasonal` classification is preserved across native override when native still confirms OOS exists, because cadence information is genuinely meaningful even when the percentage is wrong.
- *Classification gates:* `totalOOSDays == 0` → `none`. Less than 4 weeks of total span → `none` (not enough history). More than 50% OOS over 12+ weeks of span → `chronic`. Three or more events with gap CoV below 0.3 → `seasonal`. Otherwise → `intermittent`.

**Constants.**
- `MIN_OOS_EVENTS_CADENCE = 3`: minimum OOS events to compute the cadence sub-object.
- `CADENCE_REGULARITY_COV_THRESHOLD = 0.3`: gap CoV below this is `regular` / `seasonal`, above is `irregular` / `intermittent`.
- `MIN_OOS_WEEKS_FOR_CLASSIFICATION = 4`: minimum total span (weeks) before any non-`none` label is allowed.
- `MIN_OOS_WEEKS_FOR_CHRONIC = 12`: additional span requirement for the `chronic` label.
- Native override severe-mismatch threshold: `≥ 40` percentage points (or 0-vs-≥ 50 polarity flip).
- Native override soft-mismatch threshold: `≥ 15` percentage points (mention only, no override).
- Chronic OOS percentage gate: `> 50%` over full history.

**Output shape.**
```ts
{
  classification: 'none' | 'intermittent' | 'seasonal' | 'chronic';
  eventsInPeriod: number;
  totalOOSDays: number;
  avgDurationDays: number;
  maxDurationDays: number;
  lastOOSDaysAgo: number | null;
  oosPercentage: number;
  nativeOOS30: number | null;
  nativeOOS90: number | null;
  nativeOOS180: number | null;
  seriesOOSPercentage: number | null;  // pre-override derived value
  summary: string;
  cadence?: {
    avgGapBetweenEventsDays: number;
    regularity: 'regular' | 'irregular' | 'insufficient_data';
    lastRestockDaysAgo: number | null;
    summary: string;
  };
}
```

**How to read it.**
- `seasonal` with a populated `cadence.avgGapBetweenEventsDays` is the highest-value pattern: you can time inventory arrivals to Amazon's known OOS windows.
- `chronic` (> 50% OOS over 12+ weeks) means Amazon is functionally absent; price the listing against third-party competition, not against Amazon's last-known price.
- Watch for `seriesOOSPercentage` and `oosPercentage` disagreeing in the summary string: that's the native-override path firing and means the event-level fields (`avgDurationDays`, `eventsInPeriod`) are zeroed by design.

**Caveats.**
- The derived path requires raw Amazon price transitions; if Keepa returns only flattened weekly medians, only the native path contributes.
- Native percentages are Keepa's own metric and may include in-stock-but-buy-box-suppressed states that the transition-based derivation doesn't.
- When `nativeOverrideFired`, the `cadence` sub-object is suppressed entirely: series-derived restock timing is unreliable when the percentage was wrong.
- A new listing under 4 weeks old returns `classification: 'none'` even if Amazon has been OOS the whole time.

### Price trend

**What it measures.** Direction and magnitude of price drift over the 90-day window: is the Buy Box trending up, down, or flat, and by how many percent.

**Why it matters.** Tells you whether you're catching a listing on the way up (margin expansion) or down (race-to-bottom risk) before sourcing. Combined with `priceVelocity`, distinguishes "drifting" from "actively moving."

**Method.** Detects promotional spikes via `detectPromoTransitions` and removes them, then runs a linear regression of price (cents) against real time (days from earliest in-window observation) over the non-null, non-promo points. Computes `changePercent` as the regression's predicted last value vs predicted first value, rounded to 0.1%. Labels direction by changePercent only: `> 5%` rising, `< -5%` falling, else flat.

**Constants.**
- Direction thresholds: `changePercent > 5%` = rising, `< -5%` = falling, else flat.
- Window: 90 days when `windowStartMs` is passed (the standard call).
- Requires `≥ 2` non-null observations to attempt regression.

**Output shape.**
```ts
{
  direction: 'rising' | 'falling' | 'flat';
  changePercent: number;       // rounded to 0.1%
  daysAnalyzed: number;
  observationCount: number;    // non-null events in window
  slopeCentsPerDay: number;    // regression slope, rounded to 0.01
  promoSpikesDetected: number;
  summary: string;
}
```

**How to read it.**
- `rising` + a healthy `priceVelocity.regime` of `accelerating_rise` is a clean uptrend: sourcing while it's still building gives you headroom.
- `falling` with a non-trivial `promoSpikesDetected` count means the trend is real even after lightning deals were excluded: be cautious about committing inventory.
- `flat` with `daysAnalyzed ≥ 80` is the strongest stability signal: combine with `pricePosition.position == 'normal'` for the boring-winner pattern.

**Caveats.**
- Regression uses real timestamps (days), not array indices, so irregular Keepa sampling doesn't distort the slope.
- When every observation is flagged as a promo spike, returns flat with `summary: 'All data points were promotional spikes.'`, rare but possible on heavily promoted listings.
- The ±5% direction band is generous on purpose so micro-fluctuations don't get a direction label.

### Price velocity

**What it measures.** Rate of price change in cents/day plus its acceleration, not just whether price is moving but whether the movement is speeding up, slowing, or reversing.

**Why it matters.** A rising trend that is *decelerating* is about to plateau; a falling trend that is *reversing* is the bounce you want to source into. Velocity is the leading edge of `priceTrend`.

**Method.** Dual linear regression on the promo-cleaned 90-day series.
- *Overall regression* over all clean points → `avgDailyChangeCents` (slope in cents/day).
- *Recent regression* over the last `max(3, ceil(n * 0.3))` points → `recentDailyChangeCents`.
- `accelerationValue = recentSlope - overallSlope` (cents/day shift).
- Direction comes from the overall regression's predicted-first vs predicted-last change percent (flat if `|changePct| < 2%`).
- Acceleration label: if overall and recent slopes have opposite signs and both exceed the steady threshold → `reversing`. If overall slope is near zero (below noise floor), uses absolute difference. Otherwise uses `|recentSlope/overallSlope|`: `> 1.3` accelerating, `< 0.7` decelerating, else steady.
- Regime collapses direction + acceleration into a single label (`accelerating_rise`, `decelerating_fall`, `reversing_from_rise`, etc.).

**Constants.**
- `MIN_WEEKS_PRICE_VELOCITY = 4`: minimum clean non-promo observations.
- `ACCELERATION_STEADY_THRESHOLD_CENTS = 5` per week → daily threshold of `5/7` cents/day for "above noise floor".
- Direction-flat band: `|totalChange / firstPredicted| < 2%`.
- Acceleration ratio bands: `> 1.3` accelerating, `< 0.7` decelerating, else steady.
- Recent window: `max(3, ceil(cleanCount * 0.3))` of the trailing observations.
- Promo-spike rejection: if `promoIndices.size ≥ nonNull.length / 3`, returns flat with "velocity unreliable" summary.

**Output shape.**
```ts
{
  avgDailyChangeCents: number;       // signed, rounded to 0.01
  recentDailyChangeCents: number;    // signed, rounded to 0.01
  acceleration: 'accelerating' | 'decelerating' | 'reversing' | 'steady';
  accelerationValue: number;         // recent - overall, cents/day
  direction: 'rising' | 'falling' | 'flat';
  regime: 'accelerating_rise' | 'decelerating_rise' | 'reversing_from_rise' |
          'stable' | 'decelerating_fall' | 'accelerating_fall' | 'reversing_from_fall';
  daysAnalyzed: number;
  summary: string;
}
```

**How to read it.**
- `regime: 'reversing_from_fall'` is the classic dead-cat-bounce-vs-real-recovery question: confirm with `pricePosition.position == 'well_below'` for a high-conviction bottom-feeding setup.
- `accelerating_rise` means the listing is in active price expansion: good for resellers already holding inventory, dangerous for new buys without margin headroom.
- `decelerating_rise` is the late-trend warning: momentum is fading, expect plateau or reversal soon.

**Caveats.**
- Both slopes derive from regression so direction, average rate, and recent rate always agree on sign: earlier versions mixed regression and arithmetic per-pair deltas and could disagree.
- At the minimum boundary (4 clean points), recent regression and overall regression cover nearly the same data, so the label defaults to `steady` by design, conservative on thin data.
- Acceleration is a shift in slope, not a second derivative: don't treat `accelerationValue` as a physics-style acceleration scalar.

### Price position

**What it measures.** Where today's Buy Box price sits inside the historical price distribution: percentile, z-score, and a label spanning `well_below` through `well_above`. Also counts how often the listing has touched within 5% of its all-time low.

**Why it matters.** Tells you whether you're buying at a discount, at typical, or near the ceiling. A `well_below` reading with high reversion likelihood is the entry signal; `well_above` warns of compression risk.

**Method.** Removes promo spikes from the full history, then converts the clean events to in-state intervals so each price contributes proportional to how long it actually held. `durationWeightedDistribution` returns parallel value/weight arrays. Percentile is the duration-weighted fraction of mass strictly below the current price plus half the mass equal to it. Computes a duration-weighted mean and variance, takes std dev, and floors std dev at `1% of mean` so a stable listing with near-zero variance doesn't produce extreme z-scores. Uses the canonical `currentPriceCents` (Buy Box item price, item-only, excluding shipping) rather than the latest series value when available. Floor is `historicalMinCents` if provided, else the min of the distribution values. Counts support touches as any clean event within 5% of the floor.

**Constants.**
- `PRICE_FLOOR_PROXIMITY_PERCENT = 5`: "within 5% of floor" = a support touch.
- Std dev floor: `max(stdDev, weightedMean * 0.01)` to prevent extreme z-scores on stable listings.
- Position label z-score bands: `< -2.0` well_below, `< -1.0` below, `> 2.0` well_above, `> 1.0` above, else normal.
- Reversion likelihood: `|z| > 1.5` high, `|z| > 0.8` moderate, else low.
- Single-observation fallback: returns 50th percentile, z=0, "normal" with computed distance from floor.
- Sentinel: `currentPrice ≤ 0` is treated as missing (free product or data error).

**Output shape.**
```ts
{
  currentPercentile: number;            // 0-100, rounded to 0.1
  zScore: number;                       // rounded to 0.01
  position: 'well_below' | 'below' | 'normal' | 'above' | 'well_above';
  reversionLikelihood: 'high' | 'moderate' | 'low';
  distanceFromFloorPercent: number;     // negative means below the floor (new low)
  supportTouches: number;
  summary: string;
}
```

**How to read it.**
- `well_below` + `reversionLikelihood: 'high'` + `supportTouches ≥ 3` is the textbook mean-reversion setup: price has bounced off this floor multiple times before.
- `distanceFromFloorPercent` going negative is a fresh all-time low: the summary explicitly says "below historical floor (new low)" rather than confusingly reporting a negative "above floor" value.
- `well_above` with `reversionLikelihood: 'high'` means today's Buy Box is anomalously high: don't size the buy from this price; expect compression.

**Caveats.**
- Distribution is duration-weighted, so a 30-day price plateau outweighs a 1-day spike: this is correct for "where prices actually live" but means brief deals don't drag the distribution down.
- Promo transitions are removed before percentile calculation, so a discount that wasn't classified as a promo (slow grind-down) will land in the distribution and lower the percentile.
- With one clean observation, returns a benign "normal, 50th percentile, z=0" so the field never blanks, but the reversion signal is effectively meaningless until more data accrues.

### Price compression

**What it measures.** Whether the spread between competing seller prices is narrowing (compressing) or widening (expanding) over the extended observation window. Operates on per-seller offer snapshot history rather than the single Buy Box series, and reports both a current-vs-typical snapshot and a recent-trend direction.

**Why it matters.** Compressing spreads mean sellers are converging on a single price, typically Amazon's or the lowest-FBA price, which signals incoming race-to-bottom dynamics. Expanding spreads mean dispersion (some sellers parking high, some chasing the floor) and often more pricing room.

**Method.** Takes per-seller New-condition `priceHistory` from `OfferStockSnapshot` (offers only), filters each seller to entries within the window with a single carried-forward boundary entry so infrequent repricers aren't dropped. Bins prices into weekly buckets with last-observation-wins dedup per seller per week so fast repricers don't over-weight the bin. Applies PC-3 forward-fill so a seller's last known price carries through subsequent weekly bins until they reprice: Keepa's state-change encoding means a price persists until updated, and without forward-fill static sellers vanish from the spread series. For each weekly bin with ≥ 2 prices, filters to "competitive" offers within 2× of the weekly floor (excludes parked/scalper listings), and takes the range (max - min) of the competitive set as that week's spread. Computes `currentSpreadCents` (last week), `avgSpreadCents` (mean of all weekly spreads), `vsTypicalPercent` (current vs avg, snapshot), and `recentTrendPercent` (avg of last 4 weeks vs earlier weeks). Status comes from `recentTrendPercent` when available: `|rt| < 15%` stable, `< 0` compressing, `> 0` expanding. Falls back to a regression-slope check if there's not enough data for the recent-trend split.

**Constants.**
- `PRICE_COMPRESSION_SLOPE_THRESHOLD = 0.001`: fraction of avg spread; regression slopes under this counted as stable (fallback path only).
- `MIN_WEEKS_COMPRESSION = 8`: minimum weeks with computable spread before returning a non-insufficient status.
- `RECENT_WEEKS = 4`: trailing-window size for `recentTrendPercent`.
- `COMPETITIVE_PRICE_MULTIPLIER = 2`: drop offers more than 2× the weekly floor as parked/scalper.
- Status thresholds on `recentTrendPercent`: `< -15%` compressing, `> 15%` expanding, else stable.
- Spread change percent capped at `[-500, 500]` to keep `spreadChangePercent` (deprecated field) sane.
- Minimum sellers with history: 2, otherwise returns `insufficient_data`.

**Output shape.**
```ts
{
  status: 'compressing' | 'expanding' | 'stable' | 'insufficient_data';
  currentSpreadCents: number | null;
  avgSpreadCents: number | null;
  spreadChangePercent: number | null;    // @deprecated, use the two below
  vsTypicalPercent: number | null;       // snapshot: current vs avg, signed %
  recentTrendPercent: number | null;     // direction: last 4 weeks vs earlier, signed %
  sellerCount: number;
  weeksWithData: number;
  summary: string;
}
```

**How to read it.**
- `compressing` + low `vsTypicalPercent` (well below typical) + high competition count = active race-to-bottom; cross-reference with `insights.raceToBottom`.
- `expanding` + high `vsTypicalPercent` means sellers are dispersing upward: usually a sign that Amazon or a low-priced FBA seller went OOS and the remaining offers are stretching.
- `stable` with `vsTypicalPercent` near zero is the boring-winner pattern on the competition side, predictable margin math.

**Caveats.**
- Requires offer-level price history, which only exists when `offerStockSnapshots` was populated upstream: older or thin listings return `insufficient_data` even when the Buy Box series looks rich.
- Forward-fill assumes a static seller is still listing at their last price; sellers who silently delist won't be detected by this metric until Keepa next observes the offer page.
- The 2× competitive multiplier excludes scalper listings but also excludes legitimate premium positioning: for very low-priced items the floor × 2 cutoff can be aggressive.
- The window is the "extended" horizon (180 days when windowed) rather than the 90-day price-trend window, because spread dynamics need more history to stabilize.

### Review purge

**What it measures.** Detects week-over-week decreases in cumulative review count: review counts are normally monotonically increasing, so any drop signals an Amazon enforcement event (fake-review purge, policy action, variation split-off).

**Why it matters.** A large purge can vaporize the social proof that a listing's conversion rate depends on; chronic small purges may indicate ongoing manipulation Amazon is actively policing. Either pattern is a risk signal for sourcing.

**Method.** Filters review-count events to non-null observations, then walks the series pairwise. For each prev → curr pair, if `curr < prev` it computes `reviewsLost = prev - curr` and `dropPercent = (prev - curr) / prev * 100`. A pair only counts as a purge event if it crosses BOTH thresholds: at least 10 reviews absolute AND at least 1% relative, so a 5-review jiggle on a 100-review listing is ignored, and a 50-review wobble on a 50,000-review listing is also ignored. Aggregates events into `totalReviewsLost`, finds the most recent in days-ago, and annotates the summary when all events are small-magnitude (potential variation merges rather than enforcement).

**Constants.**
- `MIN_REVIEW_PURGE_ABSOLUTE = 10`: minimum review drop count.
- `MIN_REVIEW_PURGE_PERCENT = 1`: minimum percent drop (both thresholds must be met).
- `MIN_OBSERVATIONS_REVIEW_PURGE = 3`: minimum non-null observations to attempt analysis.
- Small-event ceiling for the noise caveat: `dropPercent < 1` flagged as possible variation-merge noise.

**Output shape.**
```ts
{
  detected: boolean;
  events: Array<{
    timestampMs: number;
    reviewsLost: number;
    dropPercent: number;
  }>;
  totalReviewsLost: number;
  mostRecentDaysAgo: number | null;
  summary: string;
}
```

**How to read it.**
- A single large event (`dropPercent` in double digits) with a recent `mostRecentDaysAgo` is a fresh enforcement hit: expect downstream conversion impact for weeks.
- Multiple small events across the window with the "variation merges" caveat in the summary usually means Amazon merged or split variations rather than purging, lower-severity signal.
- `detected: true` with `mostRecentDaysAgo > 180` means the purge is old enough that the listing has likely re-stabilized, note for context but rarely a sourcing veto.

**Caveats.**
- Operates on cumulative review count, which Keepa samples at discrete intervals: purges between samples are detected, but exact timing is at sample granularity (days).
- Variation merges and splits can move review counts up or down without an actual purge; the dual absolute+percent threshold filters most noise but not all.
- Returns `insufficient_data` for listings with fewer than 3 non-null observations, typical for very new ASINs.

---

## 5. Competition signals

This section covers algorithms that score the competitive landscape: who is in the market, how the Buy Box wins are distributed, how concentrated the offer stack is, and how much inventory backs each offer. The five algos read seller counts, BB winner transitions, per-offer fulfillment flags, and per-offer stock snapshots to produce a coherent picture of competitive pressure: Buy Box volatility (winner churn), effective competition (sellers actually pricing near the BB), FBA/FBM concentration (fulfillment dominance), seller concentration (HHI on shares), and stock depth (inventory and runway).

### Buy Box volatility

**What it measures.** How often the Buy Box winner changes, and whether one seller dominates the wins. Outputs a changes-per-week rate plus a dominant seller share by both event count and time-on-BB.

**Why it matters.** A high-churn BB means you can win share by repricing, but margins erode fast. A stable BB with a dominant seller means the incumbent has a hold: entering as a third-party reseller is risky.

**Method.** Reads either a `(sellerId | null)[]` weekly array or a `{sellerId, timeMs}[]` transition stream of BB winners. The timestamped path buckets events into 7-day weeks, counts seller-id changes within and across weeks, and divides by elapsed weeks (window first event → window end, including trailing quiet weeks). The dominant seller is scored two ways: event-share (max appearances / total entries) and time-share (interval duration weighted, sentinel sellers `-1` and `-2` excluded). When a `windowStartMs` is passed, the analyzer carries the last pre-window seller forward as a synthetic event at `windowStartMs` so a stable-hold tail counts toward coverage. Trend compares the first vs second half of the weekly change array; second half > 1.5× first → `increasing`, first > 1.5× second → `decreasing`, else `stable`.

**Constants.**
- `MIN_WEEKS_BB_VOLATILITY = 4`: coverage minimum or returns `insufficient_history`.
- `BB_HIGH_VOLATILITY_CHANGES_PER_WEEK = 5`: above this is `high`.
- `> 1` change/week → `moderate`; otherwise `stable`.
- Sentinel seller IDs `-1` (no qualified seller) and `-2` (unknown) are dropped from time-share calculations.

**Output shape (`BuyBoxVolatilityResult`).**
- `changesPerWeek: number`, winner transitions per week, 2 decimals.
- `totalChanges: number`, raw transition count over the window.
- `volatility: 'stable' | 'moderate' | 'high' | 'insufficient_history'`
- `dominantSellerEventSharePercent: number`, top seller's share of all BB events, 1 decimal.
- `dominantSellerId: string | null`
- `dominantSellerTimeSharePercent: number`, top seller's share of total time-on-BB, 1 decimal.
- `changeTrend: 'increasing' | 'stable' | 'decreasing'`
- `summary: string`

**How to read it.**
- `volatility = high` + low `dominantSellerTimeSharePercent` → repricing war; expect thin margins.
- `volatility = stable` + `dominantSellerTimeSharePercent > 80` → one seller owns the box; entry is hard.
- `changeTrend = increasing` is an early signal of new entrants or a reprice-bot war starting up.

**Caveats.** Needs ≥ 4 weeks of coverage; sparser feeds return `insufficient_history`. The weekly-string input path lacks duration data, so it forces `dominantSellerTimeSharePercent = dominantSellerEventSharePercent` and `changeTrend = 'stable'`. Sentinel sellers in the rotation are skipped for time-share but still counted in change detection.

### Effective competition

**What it measures.** How many sellers are actually competing on price: specifically, how many new-condition offers sit within 5% of the reference price. Cuts through long-tail offers priced 30%+ above the BB that pose no real competition.

**Why it matters.** The raw seller count is misleading. Three sellers within 5% of the BB is far more competitive than ten sellers spread across a wide price band. This number tells you how many you actually have to beat.

**Method.** Filters offer snapshots to New condition (`condition === 1`), collects non-null `currentPrice` values, and picks a reference price: the live Buy Box price when available, otherwise the median of the new-condition prices. Then counts how many of those prices fall within ±5% of the reference. The `competitiveRatio` is effective ÷ total new sellers, rounded to 2 decimals.

**Constants.**
- `EFFECTIVE_COMPETITION_PRICE_THRESHOLD = 0.05`: the ±5% band around the reference price.
- Classification: `effectiveSellers ≥ 6` → `high`; `≥ 3` → `moderate`; otherwise `low`.
- Condition filter: New only (`condition === 1`); used/refurb offers ignored.

**Output shape (`EffectiveCompetitionResult`).**
- `effectiveSellers: number`, count within 5% of reference price.
- `totalNewSellers: number`, new-condition offers with a price.
- `competitiveRatio: number`, effective / total, 2 decimals.
- `referencePriceCents: number`, the anchor used for the 5% band.
- `referencePriceBasis: 'buy_box' | 'median_new_offer'`
- `classification: 'low' | 'moderate' | 'high'`
- `summary: string`

**How to read it.**
- `classification = low` + many `totalNewSellers` → most listed sellers are pricing themselves out; the real fight is among 1–2.
- `referencePriceBasis = 'median_new_offer'` means there was no live BB price: treat the band as approximate.
- `competitiveRatio` close to 1.0 → almost every new offer is in the fight; this is a packed market.

**Caveats.** Snapshot-only: measures the offer stack at one moment in time, not historical churn. No usable BB price and no new offers returns the `noResult` zero shape. Offers without `currentPrice` are dropped silently.

### FBA/FBM concentration

**What it measures.** Which fulfillment method dominates the offer stack and the Buy Box wins (FBA, FBM, or genuinely mixed) plus how the price gap between the two fulfillment types is moving.

**Why it matters.** An FBA-dominant market is hard to crack as an FBM seller: Prime eligibility wins the box. An FBM-dominant market with a widening price gap suggests FBA arbitrage room. Mixed markets give you the most fulfillment flexibility.

**Method.** Takes raw FBA-price and FBM-price transition streams plus the BB rotation. Converts each transition stream into intervals via `transitionsToIntervals`, then computes time-weighted availability: the % of the union span where each stream had a non-null price. Aligns both streams to compute per-aligned-point price differentials ((FBM − FBA) / FBA × 100), then time-weights the average. Differential trend needs ≥ 4 aligned observations, fits a linear regression on normalized day offsets, and classifies by total trend change. Dominant fulfillment prefers BB-winner data; falls back to availability ratios. A contradiction check (BB says one thing, availability says the other) forces `mixed`.

**Constants.**
- Dominance via BB win %: `fbaBuyBoxWinPercent ≥ 70` → `fba_dominant`; `≤ 30` → `fbm_dominant`; else `mixed`.
- Fallback dominance via availability: one side > 2× the other → that side dominant; both > 0 → `mixed`.
- Differential trend bands: total change > 2 pp → `widening`; < -2 pp → `narrowing`; else `stable`. Minimum 4 aligned observations.
- Calendar-week counting for `weeksWithBothSeries` uses `Math.floor(timeMs / (7 days))`.

**Output shape (`FbaFbmConcentrationResult`).**
- `fbaAvailabilityPercent: number | null`, % of window with an FBA price.
- `fbmAvailabilityPercent: number | null`, same for FBM.
- `avgPriceDifferentialPercent: number | null`, time-weighted (FBM − FBA) / FBA × 100, 1 decimal.
- `differentialTrend: 'widening' | 'narrowing' | 'stable' | 'insufficient_data'`
- `fbaBuyBoxRotationSharePercent: number | null`, % of BB-winning sellers that are FBA.
- `fbaBuyBoxWinPercent: number | null`, sum of FBA sellers' BB win %.
- `dominantFulfillment: 'fba_dominant' | 'fbm_dominant' | 'mixed' | 'unknown'`
- `fulfillmentBasis: 'buy_box_winners' | 'time_weighted_availability'`
- `weeksWithBothSeries: number`, distinct calendar weeks with a paired FBA + FBM observation.
- `summary: string`

**How to read it.**
- `dominantFulfillment = fba_dominant` + `differentialTrend = widening` → FBM offers pricing further above FBA over time; possible margin shelter for FBA entry.
- `fulfillmentBasis = time_weighted_availability` means no BB rotation data was available: dominance is inferred from price-coverage, not actual wins.
- A `summary` flagging "(Buy Box and availability metrics contradict.)" means BB winners and price-coverage disagree: treat dominance as `mixed` regardless of which signal is stronger.

**Caveats.** Single-observation series no longer inflate availability (v2 fix); sparse data just produces lower availability %. The differential regression collapses to `insufficient_data` below 4 aligned points. The contradiction override silently downgrades to `mixed`: check the summary string for the explanation.

### Seller concentration

**What it measures.** Concentration of seller market share via the Herfindahl-Hirschman Index (HHI). Tells you whether one or two sellers own the demand, or whether it's spread across a long tail.

**Why it matters.** High HHI markets (one dominant seller) usually mean entry requires a price war or feature differentiation. Low HHI (fragmented) markets reward operational efficiency: you can take share without disrupting an incumbent.

**Method.** Two computation paths. If BB rotation data is available with per-seller `winPercent`, HHI = sum of `winPercent²` over all sellers, actual market shares. Otherwise estimates HHI from weekly seller counts assuming equal shares: HHI per week = `10000 / n` (where n = seller count), averaged across weeks. `avgSellerCount` is always the mean of the weekly series rounded to 1 decimal. `sellerTrend` splits the series in half (middle element dropped for odd lengths), takes the median of each half, and classifies a > 10% change as growing/declining.

**Constants (HHI classification).**
- `hhi ≥ 10000` → `monopoly`
- `hhi ≥ 4000` → `duopoly`
- `hhi ≥ 2500` → `oligopoly`
- `hhi ≥ 1500` → `competitive`
- otherwise → `fragmented`
- `sellerTrend` band: median change > 10% → `growing`/`declining`; else `stable`.

**Output shape (`SellerConcentrationResult`).**
- `hhi: number`, Herfindahl index on a 0–10000 scale.
- `concentration: 'monopoly' | 'duopoly' | 'oligopoly' | 'competitive' | 'fragmented' | 'insufficient_data'`
- `avgSellerCount: number`, mean weekly seller count, 1 decimal.
- `sellerTrend: 'growing' | 'stable' | 'declining'`
- `isEstimated: boolean`, `true` when HHI was derived from the equal-share fallback rather than actual BB win shares.
- `summary: string`

**How to read it.**
- `concentration = monopoly | duopoly` + `sellerTrend = stable` → entrenched incumbents, expect retaliation on price.
- `isEstimated = true` is a precision flag: the equal-share assumption understates concentration when one seller actually dominates.
- `concentration = competitive` + `sellerTrend = growing` → fragmenting market; easier to enter, but margins likely thinning.

**Caveats.** The equal-share HHI is a coarse estimate: Keepa's seller count tells you `n`, not the share distribution. The BB-rotation path is more accurate; `isEstimated = false` flags it. Keepa's `percentageWon` values may not sum to exactly 100 due to rounding; HHI on sum of squares is valid regardless. Empty input returns `insufficient_data`.

### Stock depth

**What it measures.** Total inventory backing the live offer stack: how many units across all new-condition sellers, split FBA vs FBM, how concentrated the stock is (HHI on shares), and, when sales velocity is available, how long the stock lasts (weeks of supply, turnover, replenishment ratio).

**Why it matters.** A market with 8 sellers but 4 of them down to < 5 units is one stockout away from a price spike. Deep stock = competitors can defend a price for weeks. Shallow stock + high velocity = entry window opening soon.

**Method.** Filters to New condition offers, resolves per-offer current stock (Keepa sentinel < 0 → null), and sums to `totalCurrentStock` split by `isFBA`. HHI on stock shares gives `stockConcentrationHHI` (0–1 scale). When `weeklyVelocity` is provided, `velocityWeeksOfSupply = totalCurrentStock / weeklyVelocity` (clamped at 104 weeks) drives the stock-level classification; otherwise absolute unit thresholds apply. Trend uses `bucketStockByWeek` (last-observation-wins per seller per week, summed across sellers) and regresses weekly totals: needs ≥ 3 weekly buckets. Replenishment ratio = `1 − (depletionRate / weeklyVelocity)`; a growing-or-stable trend with velocity present forces ratio = 1.0.

**Constants.**
- Velocity-adjusted stock levels (weeks-of-supply): `< 1` → `critical`; `< 2` → `low`; `< 6` → `adequate`; `≥ 6` → `deep`.
- Absolute fallback (units): `< 5` → `critical`; `< 20` → `low`; `< 50` → `adequate`; `≥ 50` → `deep`.
- Stock trend bands: change pct `< -0.1` → `declining`; `> 0.1` → `growing`; else `stable`.
- Replenishment classification: `> 1.05` → `accumulating`; `≥ 0.85` → `stable`; `≥ 0.15` → `depleting`; `< 0.15` → `liquidating`.
- Annual turnover benchmarks: `< 2` → `sluggish`; `< 6` → `moderate`; `< 12` → `healthy`; `≥ 12` → `fast`.
- Turnover only computed when `totalCurrentStock ≥ weeklyVelocity` (SD-7: prevents 2-units-vs-100/wk producing 2600×/yr).
- Weeks-of-supply clamped at 104 (SD-6).

**Output shape (`StockDepthResult`).**
- `totalCurrentStock: number | null`, sum of resolved per-offer stock.
- `liveOfferCount: number`, new-condition offers that reported stock.
- `fbaStockUnits: number | null` / `fbmStockUnits: number | null`
- `stockConcentrationHHI: number | null`, HHI on stock shares, 0–1 scale, 2 decimals.
- `topSellerStockPercent: number | null`, % held by largest holder.
- `stockLevel: 'deep' | 'adequate' | 'low' | 'critical' | 'unknown'`
- `stockTrend: 'growing' | 'stable' | 'declining' | 'insufficient_history'`
- `depletionRateUnitsPerWeek: number | null`, absolute weekly drawdown (declining trend only).
- `weeklyVelocity: number | null`, passed-in sales velocity, echoed back.
- `replenishmentRatio: number | null`, % of velocity being restocked, 2 decimals (1.0 = full replacement).
- `inventoryClassification: 'accumulating' | 'stable' | 'depleting' | 'liquidating' | 'unknown'`
- `turnoverPerMonth: number | null` / `turnoverPerYear: number | null`
- `turnoverBenchmark: 'sluggish' | 'moderate' | 'healthy' | 'fast' | null`
- `velocityBasis?: 'observed-floor' | 'observed-history' | 'estimated' | null`
- `velocityDerivedFields?: string[]`, names of fields computed from velocity (so you know which are softer when basis is estimated).
- `summary: string`

**How to read it.**
- `stockLevel = critical | low` + `inventoryClassification = depleting` → near-term stockout likely; price spike window opening.
- `topSellerStockPercent > 60` → one holder is the supply; their pricing decisions move the BB.
- `turnoverBenchmark = fast` + `stockLevel = adequate` → healthy movement; competitors aren't sitting on dead stock.

**Caveats.** Requires offers + per-offer stock data: not derivable from price history alone. Keepa stock sentinel (-1) is dropped to null; offers with no stock data don't inflate `liveOfferCount`: only offers that reported stock are counted. Trend regression needs ≥ 3 weekly buckets; otherwise `insufficient_history`. Velocity-derived fields (`stockLevel`, `turnoverPerMonth`, `turnoverPerYear`, `turnoverBenchmark`, `replenishmentRatio`, `inventoryClassification`) inherit the confidence of `velocityBasis`: `estimated` means treat the numbers as directional only.

---

## 6. Composite risk signals

Two higher-order signals aggregate the per-family detectors above into a single actionable read. They introduce no new measurements: they combine outputs from sections 3–5 so you get one verdict instead of several raw flags.

### IP risk (`insights.ipRisk`)

**What it measures.** The likelihood that an intellectual-property complaint or platform enforcement action hit this listing, inferred from two corroborating footprints: a **seller cliff** (a sudden drop in seller count, §3) and a **review purge** (a sudden loss of reviews, §4). Runs for every product: both inputs are core signals, so offers data is not required.

**Output shape.** `{ riskLevel, signals, confidence, recoverySummary, timelineDays, summary }`

- **`riskLevel`**: `none` / `low` / `moderate` / `high` / `insufficient_data`.
- **`signals`**: which footprints fired: `seller_cliff`, `review_purge`, and `concurrent_cliff_purge` when both landed close together.
- **`confidence`**: `low` / `moderate` / `high`.
- **`timelineDays`**: `{ cliffDaysAgo, purgeDaysAgo }` (either may be null), or null when neither fired.
- **`recoverySummary` / `summary`**: plain-language status; surface verbatim.

**Method.** An 8-case decision tree over the two signals:

- A cliff and a purge are **concurrent** when any cliff event sits within **28 days** of any purge event (checked across all event pairs by timestamp).
- A purge is **small** when fewer than **50** reviews were lost *and* the largest single drop is under **5%**; otherwise large.
- A cliff is **recovered** only when *every* detected cliff event recovered (one lingering unrecovered cliff keeps the status unrecovered).

The cases:

1. Concurrent, with an unrecovered cliff → **high**.
2. Concurrent, recovered, small purge → **moderate** (concurrent + recovered + *large* purge → **high**).
3. Both present but not concurrent → **moderate**.
4. Cliff only, medium/high severity → **moderate** (a single signal is capped at moderate).
5. Purge only, small → **low**.
6. Purge only, large → **moderate**.
7. Cliff only, low severity → **low**.
8. Neither → **none**.

**How to read it.** `high` (a concurrent, unrecovered cliff + purge) is the strongest enforcement footprint: treat the ASIN as risky to source until you understand why sellers and reviews left together. A single signal never exceeds `moderate` on its own, because one footprint can have benign explanations (a seller exiting, a review-spam sweep). `timelineDays` tells you how fresh the events are.

**Caveats.** This is a *correlation* read, not proof of an IP action: the summary deliberately hedges ("possible", "suggesting"). `insufficient_data` means neither underlying detector had enough history to run.

### Race-to-bottom early warning (`insights.raceToBottomWarning`)

**What it measures.** Whether a margin-eroding price war is *emerging*, aggregated from up to six competitive signals grouped into **four families** so correlated signals can't double-count. Runs when offers data is present (it draws on the offer-derived detectors).

**Output shape.** `{ warningLevel, activeSignals, activeFamilies, signalDetails, priceDeclinePercent, seasonalAdjusted, summary }`

- **`warningLevel`**: `none` / `watch` / `warning` / `strong_pattern` / `insufficient_data`.
- **`activeSignals`**: raw count of individual signals that fired (reported for context).
- **`activeFamilies`**: which of the four families are active; this is what sets the level.
- **`signalDetails`**: the individual signal names that fired.
- **`priceDeclinePercent`**: measured price decline (from `raceToBottom`).
- **`seasonalAdjusted`**: true when a confirmed seasonal descending phase suppressed the price-direction family.

**Method.** Each individual signal maps to a family:

- **price_direction**: `race_to_bottom`, `price_falling` (price velocity falling).
- **repricing_behavior**: `sawtooth_pattern`, `buy_box_volatility` (high/moderate).
- **demand_supply**: `seller_growth` (seller count growing).
- **margin_pressure**: `price_compression` (spread compressing).

The level comes from the **active-family count**, *not* the raw signal count, so two correlated signals in one family can't inflate it: 0 → `none`, 1 → `watch`, 2–3 → `warning`, 4 → `strong_pattern`. It needs at least two of the five core inputs present, or it returns `insufficient_data`.

**Seasonal suppression.** When the product has a *confirmed* seasonality in a descending phase, the price-direction family is dropped (a seasonal post-peak markdown isn't a competitive price war), `seasonalAdjusted` is set, and the summary says so.

**How to read it.** `strong_pattern` (all four families) means broad, corroborated downward pressure: expect margin erosion. `watch` / `warning` are earlier signals; pair them with `priceDeclinePercent` and your own margin math. If `seasonalAdjusted` is true, discount the price-direction read: the decline is calendar-driven, not competition.

**Caveats.** Aggregation only: the underlying detectors carry their own caveats (§3–§5). `activeSignals` can exceed the family count (correlated signals), so always read the *level* off families, not the raw signal tally.
