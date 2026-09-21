# liquiditybacktest

A **TradingView backtesting strategy** built from the `Liquidity Radar` indicator's
liquidity approach: sweep + reversal (SFP) entries, Draw-on-Liquidity (DOL) bias,
confluence scoring, and liquidity-pool targets.

- **[`LiquidityStrategy.pine`](./LiquidityStrategy.pine)** — the Pine v6 `strategy()`. Paste
  it into the TradingView Pine Editor and add it to a chart to backtest.
- **[`STRATEGY_SPEC.md`](./STRATEGY_SPEC.md)** — the knowledge handoff: exact strategy rules,
  the DOL/confidence model, the Pine conversion notes, and the validation plan.

## What it does

The signal core (level tracking, SFP detection, DOL pull/quality model, confluence score,
order blocks) is ported **verbatim** from the source indicator
([`jordanmartinek/liquidityindicator` → `LiquidityRadar.pine`](https://github.com/jordanmartinek/liquidityindicator)),
so backtest signals match the chart signals. Only the cosmetic drawing (banners, ghost lines,
FVG boxes, tug-of-war meters) is dropped — none of it affected signals.

On top of that signal core, the strategy adds real execution:

- **Entry** on the **close of the SFP bar** (a wick pierces a tracked level, then the candle
  closes back on the origin side — a stop-run), filling on the next bar. `calc_on_every_tick`
  is left **off** so the backtest matches bar-close live behavior.
- **Stop** just beyond the sweep wick, padded by `padATR × ATR`. This defines **1R**.
- **Take-profit** at the **opposing liquidity pool** (the DOL magnet / nearest un-swept pool on
  the other side), or a **fixed R multiple**. Falls back to a fixed R if no opposing pool exists.
- **Optional scale-out:** close part of the position at 1R (`TP1`), let the runner ride to the pool.
- **Risk-based sizing:** `qty = (equity × riskPerTrade%) / |entry − stop|`.
- **Optional session (killzone) filter**, one position at a time (`pyramiding = 0`).
- **Costs modeled** in the header: `0.02%` commission + `2` ticks slippage (adjust for your market).

## Key inputs (grouped in the settings panel)

| Group | What to tune |
|---|---|
| ① Previous-Period Levels | PDH/PDL, PWH/PWL, PMH/PML, daily open |
| ② Sessions | Asia / London / NY windows, timezone, reset/reclassify behavior |
| ③ Structure | swing pivot length, HTF swings, EQH/EQL, SFP volume filter, order blocks |
| ④ Clustering & Confluence | cluster distance, round-number confluence step |
| ⑤ Draw-on-Liquidity Model | gravity sensitivity, type/age/freshness/confluence weights, momentum tilt |
| ⑥ **Strategy** | **DOL-bias gate, confluence gate, stop pad, target mode, scale-out, risk %, session filter** |
| ⑦ Style | signal markers, active stop/target lines, colors |

## Validation plan (from the spec — prove or kill the edge)

1. Backtest on your target instrument + timeframe (e.g. 5m/15m/1h). Watch win rate, avg R,
   expectancy, max drawdown, profit factor.
2. **A/B the gates incrementally:** SFP + volume only → add DOL bias → add confluence gate →
   pool target vs fixed R. Keep only what earns its place.
   - Start point: `Gate entries by DOL bias` ON, `confluence gate` OFF, `Opposing pool` target.
3. Walk-forward / out-of-sample to guard against curve-fitting.
4. Keep commission + slippage on — don't trust a pre-cost edge.
5. Only then consider forward/paper testing.

## Related
- Source indicator: https://github.com/jordanmartinek/liquidityindicator
