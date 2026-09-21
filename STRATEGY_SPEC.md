# Liquidity Backtest — Strategy Spec & Handoff

**Purpose:** turn the `Liquidity Radar` TradingView **indicator** into a TradingView
**strategy** (`strategy()`) so the liquidity approach can be backtested with real fills,
equity curve, and stats. This doc is the knowledge handoff so a fresh session can build
the backtester without needing the original chat history.

> Source indicator: `jordanmartinek/liquidityindicator` → `LiquidityRadar.pine` (Pine v6,
> "STREAMLINED CORE"). This spec mirrors its logic exactly so backtest signals == chart signals.

---

## 0. What the indicator already does (the map)

The indicator tracks **liquidity levels**, scores their **draw-on-liquidity (DOL) pull**, and
flags **sweep + reversal (SFP)** events. The strategy reuses all of this; it only adds
**entries, exits, and position management**.

Levels tracked (each removed once swept):
- **Previous period:** PDH/PDL, PWH/PWL, PMH/PML, daily open.
- **Sessions:** Asia / London / New York H & L (optionally reset daily; unswept survivors can be reclassified as swings).
- **Structure:** swing highs/lows (pivot length default 10) + optional HTF swings (default 4h, pivot 5).
- **Equal highs/lows (EQH/EQL):** two pivots within `eqTol` (default 0.35× ATR) — the strongest engineered-liquidity magnets.
- **Clustering:** levels within `clustDist` (default 0.5× ATR) merge into strength-rated **zones** with a 0–100 **confluence score**.

---

## 1. The entry trigger — SFP (sweep + reversal)

This is the **core validity trigger** and must be the strategy's entry condition. Ported
verbatim from the indicator (bar-close logic):

```
// candidate levels this bar: swing/EQ highs in lvHiNow, lows in lvLoNow
// SHORT: a wick pierces ABOVE a tracked high, but the candle CLOSES back below it
sweptHiLvl = highest tracked high with (high > lv and close < lv)
// LONG: a wick pierces BELOW a tracked low, but the candle CLOSES back above it
sweptLoLvl = lowest  tracked low  with (low  < lv and close > lv)

volOK   = (not sfpUseVol) or (rvol >= rvolMult)     // rvol over rvolLen bars, default mult 1.5, len 20
sfpShort = not na(sweptHiLvl) and volOK
sfpLong  = not na(sweptLoLvl) and volOK
```

**Entry conditions (recommend as the strategy default):**
- `strategy.entry` **long** when `sfpLong` AND DOL bias is up AND (optional) swept level's confluence ≥ threshold.
- `strategy.entry` **short** when `sfpShort` AND DOL bias is down AND (optional) confluence ≥ threshold.
- Enter on **bar close** of the SFP bar (do NOT use intrabar `calc_on_every_tick` for the signal — keep backtest == live).

The DOL-bias and confluence filters are **optional gates** to A/B test — start with just SFP+volume,
then layer the DOL/confluence gates and measure whether they improve expectancy.

---

## 2. The DOL gravity + confidence model (for bias filter & targets)

Per-level **pull** and distance-neutral **quality** (from `f_pull`, most recent version):

```
dist  = |levelPrice - close|
distW = (1 / (1 + sensK * (dist/ATR))) ^ (1 + dolNearBias)     // dolNearBias default 1.5 → exp 2.5
typeW = 1.0 (+ importance): EQ 2.4, PM 2.2, PW 1.9, PD 1.6, session 1.2  (scaled by sensK)
ageW  = 1 + min(1, log(1+ageBars)/6) * sensK                   // older resting liquidity pulls harder
freshW= 0.6 if dist < 0.15*ATR else 1.0                        // discount a level price is sitting on
confW = 1 + confluenceScore/100                                 // 1.0 .. 2.0

qual  = typeW * ageW * freshW * confW        // distance REMOVED  → "how strong a magnet for its distance"
pull  = distW * qual                          // proximity-weighted gravity
sensK = High 2.0 / Medium 1.0 / Low 0.5       (input dolSens)
```

- **DOL bias:** sum pull for levels above price (BSL) vs below (SSL). `liqBias = (bsl - ssl)/(bsl+ssl)`,
  then tilt by live momentum: `bias = (1 - dolMomWt/100)*liqBias + (dolMomWt/100)*mom` (dolMomWt default 30%).
  `bias >= 0` → draw UP, else DOWN. Use as the directional filter.
- **🎯 primary magnet** = level with the greatest `pull` overall (nearest strong pool). Natural **take-profit target**.
- **Per-level confidence %** = a level's share of total **quality** (Magnet-quality mode, default) or total **pull** (Raw-pull mode),
  computed over only the **N nearest levels each side** (`lvlConfWin`, default 5). This is the "which target is most likely" signal.

---

## 3. Exits (what to build & A/B test)

- **Stop loss:** beyond the sweep wick that triggered the SFP, padded by `padATR × ATR` (e.g. 0.25× ATR).
  - Long stop = `low[sweptBar] - pad`; short stop = `high[sweptBar] + pad`. This defines **1R**.
- **Take profit (primary):** the **opposing liquidity pool** — the DOL magnet / highest-confidence level on the other side.
  - Long TP = nearest un-swept BSL above; short TP = nearest un-swept SSL below.
  - **Fallback** if no opposing pool: fixed R multiple (default 2R).
- **Optional scale-out:** TP1 at 1R (partial), runner to the pool.
- **Invalidations:** stop hit, or structure shift against the trade.

Make stop pad, target mode (pool vs fixed-R), and scale-out **inputs** so they can be optimized.

---

## 4. Turning the indicator into a `strategy()` — Pine specifics

- Change the declaration: `indicator(...)` → **`strategy("Liquidity Backtest", overlay=true, ...)`**.
  Keep `max_bars_back=1000`, `max_lines/labels/boxes_count` as needed.
- Use `strategy.entry`, `strategy.exit` (with `stop=` and `limit=`), `strategy.close`.
- **`calc_on_every_tick` is strategy-only** (it is INVALID on `indicator()` — error CE10120). For a faithful
  bar-close backtest, leave it **off** (default false) so signals fire on closed bars.
- Set realistic backtest costs in the `strategy()` header or properties: `commission_type`,
  `commission_value`, `slippage`, and a sane `default_qty_type`/`default_qty_value` (or use `%_of_equity`).
- **Position sizing:** derive quantity from risk — `qty = (equity * riskPerTrade) / (|entry - stop|)`.
  Expose `riskPerTrade` (e.g. 0.5–1%) as an input.
- **One position at a time** to start (`pyramiding=0`), matching the indicator's one-setup-at-a-time nature.

### Known Pine v6 gotchas (learned building the indicator — avoid re-hitting these)
- `max_bars_back=1000` prevents "historical offset beyond buffer" errors.
- Bound array-index loops by `array.size(...)`, not a captured length.
- Functions can't reference vars declared later or inside a `barstate.islast` scope (CE10272).
- **Tuple return in a ternary is invalid** — assign tuples on their own line.
- No nested (indented) `f_x() =>` function definitions.
- Keep U+FE0F (emoji variation selectors) OUT of `input.string` option lists.
- Static sanity before compiling: exactly one `strategy(`/`indicator(` line, balanced parens & brackets,
  no stray backslashes, no literal `\n`.

---

## 5. Validation plan (prove or kill the edge)

1. Backtest on the target instrument/timeframe. Metrics: win rate, avg R, expectancy, max DD, profit factor.
2. Test filters incrementally: SFP+vol only → +DOL bias → +confluence gate → +pool-target. Keep only what earns its place.
3. Walk-forward / out-of-sample to guard against curve-fitting.
4. Model costs (commission + slippage) — don't trust a pre-cost edge.
5. Only then consider forward/paper testing.

---

## 6. Open questions for the user (confirm before/while building)

1. **Instrument + execution timeframe?** (indicator is TF-agnostic; the backtest needs a concrete TF, e.g. 5m/15m/1h.)
2. **Target logic:** ride to the opposing liquidity pool, or fixed R multiple? Scale out at 1R?
3. **Which gates on by default:** SFP+volume only, or also DOL bias and confluence threshold?
4. **Risk per trade** and starting equity for the backtest?
5. **Pyramiding / multiple positions**, or strictly one at a time?
6. **Session filter?** (e.g. only take setups during London/NY killzones.)

---

## 7. Reference

- Indicator repo: https://github.com/jordanmartinek/liquidityindicator (`LiquidityRadar.pine`)
- Recent indicator changes relevant here: Magnet-quality confidence mode (distance-neutralized) and the
  N-nearest-each-side confidence window (`lvlConfWin`, default 5) — both affect which level is the
  "most likely target," useful when choosing take-profit pools.
