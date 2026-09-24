# ML Integration Plan — Liquidity Strategy

**Status:** plan / R&D. No ML code yet. This doc is the strategy for *how* machine learning
should (and should not) improve the `Liquidity Radar` indicator and the liquidity backtester.

> ⚠️ Read the constraints first. ML here **optimizes an existing edge — it does not create one.**
> If the base liquidity logic has no edge in the backtest, ML will only overfit and look good
> until it doesn't. Prove the base strategy first (see `STRATEGY_SPEC.md`), then apply ML.

---

## 0. The hard constraint (why this lives here, not in the Pine indicator)

TradingView **Pine Script cannot run machine learning** — no libraries, no model loading,
no training in the indicator. So ML enters only two honest ways:

1. **Train offline, hardcode the result.** Learn weights/thresholds in Python on historical
   data, then bake the *fixed learned numbers* back into the Pine indicator (`f_pull` /
   `f_confScore` weights, thresholds). The indicator stays deterministic and auditable; ML
   merely *tuned* it.
2. **ML runs outside the indicator** — in this backtester and/or the `liq-ai-bot` repo, where
   Python can run scikit-learn / lightgbm live.

Anything claiming "live ML inside the Pine indicator" is either a hand-tuned formula relabeled
"AI" or a misuse of `request.security`. We do not build that.

---

## 1. What the indicator already gives us (the feature set)

The indicator already computes a rich, per-level feature vector — this is exactly what a model
needs, and it's the reason ML is viable here at all. Per tracked level / cluster:

- **confluence score (0–100)** and its parts (stack, OB graded, FVG graded, OTE, prem/disc, round, VWAP, breaker)
- **DOL pull** and distance-neutral **magnet-quality**
- **distance to price** (× ATR), **level type** (EQ / PM / PW / PD / session / swing / POC / VA / opens / breaker / rEQ)
- **age** (bars since birth), **freshness**, **tested/virgin** state
- **path-clearance** (graded OB/FVG obstacles to the level), **resistance-adjusted** side pull
- **DOL bias & conviction**, **dominance cap** context
- **session**, **rvol**, **equilibrium / premium-discount** position in the dealing range
- **SFP trigger** context (sweep + reversal + volume)

The label we want to predict: **did price actually reach this level (before an opposing move of X·ATR / 1R)?**
and/or **did the SFP setup reach its target before its stop?**

---

## 2. Where ML genuinely helps (ranked by value × honesty)

### 2.1 Calibrate the confidence % into a real probability ★ highest value
Today the confidence % is a *relative share* ("this level's slice of the pull"). ML turns it into
an *absolute, calibrated probability*: "levels that read 70% actually get hit ~70% of the time."
- **Model:** logistic regression or isotonic/Platt calibration on the pull/quality score + top features.
- **Output:** a calibrated `P(reach)` per level. Ship as learned coefficients hardcoded into the
  indicator, or served live in the bot.
- **Why first:** it improves the number you look at most, and it *forces* the labeled dataset to exist.

### 2.2 Learn the confluence / pull weights (replace hand-picked constants)
Every weight in `f_pull` / `f_confScore` (EQ 2.4, OB +? graded, tested −25%, dolNearBias, …) was
chosen by hand. Fit a model to learn the weights that best predict "level gets reached," then
**hardcode the learned coefficients** back into the Pine functions.
- **Model:** logistic regression (interpretable coefficients map cleanly to weights) or
  gradient-boosted trees for feature interactions (then approximate back to weights).
- **Guardrail:** only adopt learned weights if they beat the current hand-tuned weights **out of sample**.

### 2.3 Setup-quality classifier for the SFP trigger
Given an SFP's features (rvol, DOL alignment, confluence, session, distance-to-target, path-clearance),
predict `P(target before stop)`. Trade/flag only setups above a learned threshold.
- **Home:** the bot (`liq-ai-bot`) as the ML filter its `DESIGN.md` already specs; can also drive an
  indicator "high-quality SFP" star.

### 2.4 Regime detection
Cluster market state (trending / ranging / high-vol) so the system sizes down or stands aside where
the liquidity edge historically decays.
- **Model:** k-means / GMM on volatility, trend, and range features; or an HMM for regime transitions.
- **Output:** a regime label → indicator tag / bot sizing.

### 2.5 Feature importance → prune the model
Train a model, read feature importances, and **remove factors that don't earn their place**. This
could reveal that (e.g.) age barely predicts while path-clearance × EQ is huge — letting us simplify
and sharpen the indicator honestly rather than accreting factors.

---

## 3. What ML does NOT help with (do not build)

- **Raw price / next-candle prediction** — unreliable, unauditable, overfits noise.
- **Deep learning on OHLC** — severe overfitting risk on limited, non-stationary financial data.
  Simple, explainable models (logistic regression, gradient boosting) beat neural nets here and
  stay interpretable.
- **Live ML inside the Pine indicator** — technically impossible (see §0).
- **Any black-box that replaces the deterministic rules** — rules decide *whether*; ML only decides
  *whether this instance is worth it* and *how much*. Hard risk limits are never ML.

---

## 4. The pipeline (order matters)

```
 Indicator features ──► BACKTESTER logs {features + outcome label} per level/setup
        │                                   │
        │                                   ▼
        │                          TRAINING DATASET (CSV/parquet)
        │                                   │
        │                                   ▼
        │                     OFFLINE TRAINING (Python: sklearn / lightgbm)
        │                       • calibrate confidence  • learn weights
        │                       • setup classifier      • feature importance
        │                                   │
        ├───────── hardcode learned weights back into Pine (f_pull / f_confScore)
        └───────── or serve model live in liq-ai-bot (ML filter / sizing)
                                            │
                                            ▼
                              OUT-OF-SAMPLE VALIDATION (walk-forward)
                              ship only if it beats hand-tuned weights on unseen data
```

### Milestones
- **M1 — Data.** Add a feature-logging / export mode: the backtester (or an indicator data-window
  export) writes, per level & per SFP setup, the full feature vector + the realized outcome label.
  *Nothing downstream is possible without this.*
- **M2 — Baseline.** Fit a simple calibrated classifier (§2.1) on the log. Report AUC, calibration
  curve, and whether calibrated `P(reach)` beats the current relative confidence % as a ranker.
- **M3 — Weights.** Fit interpretable weights (§2.2); compare learned vs hand-tuned out of sample.
- **M4 — Setup filter + regime** (§2.3, §2.4) in the bot.
- **M5 — Ship.** Hardcode winning weights into the indicator (a versioned PR) and/or deploy the
  model in the bot. Re-validate on fresh data before trusting any of it.

---

## 5. Data & method notes (avoid the classic mistakes)

- **Label leakage:** compute features using ONLY information available at the level's decision bar;
  the outcome is measured strictly forward. No lookahead in `request.security` (indicator already
  uses `lookahead_on` for periodic levels — the *dataset* must snapshot features at decision time).
- **Non-stationarity:** markets drift. Prefer **walk-forward** validation (train past → test future),
  never a random shuffle split. Re-train periodically.
- **Class imbalance:** "reached" vs "not reached" may be skewed; use proper metrics (AUC, calibration,
  precision/recall) — not raw accuracy.
- **Costs:** any setup-classifier edge must survive fees + slippage (see `STRATEGY_SPEC.md §4`).
- **Keep it explainable:** start with logistic regression / gradient boosting. Only escalate if a
  simple model is clearly insufficient AND validation supports it.
- **Small data:** financial samples are limited and noisy — favor strong regularization, few features,
  and skepticism over model complexity.

---

## 6. Open questions for the user
1. **Deployment target:** hardcode learned weights back into the Pine indicator, run the model live
   in the bot, or both?
2. **Prediction target:** per-level "reached before X·ATR opposing move", or per-SFP "target before stop"? (Both are useful; which first?)
3. **Instrument(s) + timeframe** for the training data (must match how you'll use it).
4. **Are you comfortable** with the honest outcome that ML may show the base edge is thin — in which
   case we fix/retire the strategy rather than dress it up?

---

## 7. Reference
- Strategy rules & backtest plan: `STRATEGY_SPEC.md` (this repo)
- Bot / ML-filter design: `liq-ai-bot/DESIGN.md`
- Indicator (feature source): https://github.com/jordanmartinek/liquidityindicator
