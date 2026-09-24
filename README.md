# liquiditybacktest

Building a **TradingView backtesting strategy** from the `Liquidity Radar` indicator's
liquidity approach (sweep + reversal entries, DOL bias, confluence, liquidity-pool targets).

> This repo starts as a **knowledge handoff**. Read **[`STRATEGY_SPEC.md`](./STRATEGY_SPEC.md)**
> first — it captures the exact strategy rules, the DOL/confidence model, the Pine `strategy()`
> conversion notes, the validation plan, and the open questions to confirm before building.

## Status
- **Seed:** `STRATEGY_SPEC.md` handoff doc. No strategy code yet.
- **Next:** build `LiquidityStrategy.pine` (`strategy()`) per the spec, then backtest.

## Machine learning
See **[`ML_INTEGRATION_PLAN.md`](./ML_INTEGRATION_PLAN.md)** for how (and how not) to apply ML:
the honest constraints (Pine can't run ML — train offline, hardcode results, or run in the bot),
the per-level feature set the indicator already produces, ranked high-value uses (calibrate the
confidence %, learn the confluence/pull weights, SFP setup classifier, regime detection), and the
data → train → validate → ship pipeline. ML optimizes an existing edge; it does not create one.

## Related
- Source indicator: https://github.com/jordanmartinek/liquidityindicator
- Bot / ML-filter design: `liq-ai-bot/DESIGN.md`
