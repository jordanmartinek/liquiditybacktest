# liquiditybacktest

Building a **TradingView backtesting strategy** from the `Liquidity Radar` indicator's
liquidity approach (sweep + reversal entries, DOL bias, confluence, liquidity-pool targets).

> This repo starts as a **knowledge handoff**. Read **[`STRATEGY_SPEC.md`](./STRATEGY_SPEC.md)**
> first — it captures the exact strategy rules, the DOL/confidence model, the Pine `strategy()`
> conversion notes, the validation plan, and the open questions to confirm before building.

## Status
- **Seed:** `STRATEGY_SPEC.md` handoff doc. No strategy code yet.
- **Next:** build `LiquidityStrategy.pine` (`strategy()`) per the spec, then backtest.

## Related
- Source indicator: https://github.com/jordanmartinek/liquidityindicator
