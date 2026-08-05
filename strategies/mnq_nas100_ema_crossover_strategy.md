# MNQ / NAS100 EMA Crossover Trend Strategy (1H, Pine Script v6)

Companion doc for `mnq_nas100_ema_crossover_strategy.pine`.

## 1. Executive Summary

A 1-hour trend-following strategy for Micro Nasdaq futures (MNQ) and NAS100
CFDs. It trades a fast 10/20 EMA crossover for timing, filtered by a 50/200
EMA regime filter for direction, with position size and stops driven off a
fixed $50-per-trade risk budget and a 14-period ATR. It trades both
directions across all 24 hours of the session (no time-of-day filter by
default), which suits a swing/intraday hybrid that wants to catch Asia,
Europe, and New York moves without gaps in coverage.

## 2. Strategy Rules

**Long**
1. 10 EMA crosses above 20 EMA
2. The crossover bar closes (signal only fires on a confirmed bar — see §4)
3. 50 EMA > 200 EMA (bullish regime)

**Short**
1. 10 EMA crosses below 20 EMA
2. The crossover bar closes
3. 50 EMA < 200 EMA (bearish regime)

**Exit** — whichever comes first:
- Opposite 10/20 EMA crossover (unconditional close, ignores the 50/200 filter)
- ATR chandelier trailing stop
- Initial 2×ATR hard stop (active until the trail overtakes it)

One position at a time; a new signal is ignored while a trade is open.

## 3. Pine Script v6 Code

See `mnq_nas100_ema_crossover_strategy.pine` in this folder — paste directly
into TradingView's Pine Editor. Highlights:

- `//@version=6` strategy, `overlay=true`, `process_orders_on_close=true`
- All inputs grouped and adjustable (EMA lengths, risk %, ATR multiples,
  session window, backtest date range, long/short toggles)
- Contract sizing computed per trade from `riskUsd / (stopDistance × pointValue)`,
  floored to whole contracts and capped by `maxContracts`
- Built-in Strategy Report dashboard table (net profit, win rate, profit
  factor, expectancy, max DD, etc.) rendered on the chart

## 4. Risk Management Explanation

**Fixed $ risk, not fixed contracts.** Every trade risks the same `riskUsd`
(default $50), regardless of how wide the ATR stop is that bar. Contracts
are computed as:

```
stopDistance = ATR(14) × atrSlMult   // points
contracts    = floor(riskUsd / (stopDistance × pointValue))
```

`pointValue` is an input because it depends on the instrument: MNQ = $2/point,
NQ = $20/point, MES = $5/point, ES = $50/point. For NAS100 CFDs, set it to
whatever your broker quotes as $-per-point-per-lot. Get this wrong and every
other number in the backtest is wrong, so verify it against your broker's
contract spec before trusting results.

**Stop sequencing:**
1. On entry, the hard stop is set at `entry ∓ 2×ATR`.
2. From the bar *after* entry onward, the stop only ratchets in the
   trade's favor: `max(currentStop, close − trailMult×ATR)` for longs,
   the mirror for shorts. It never loosens.
3. Independently, an opposite 10/20 crossover closes the trade outright —
   this can fire before the trailing stop would, and is the strategy's
   primary "the setup is over" signal.

Because sizing is inversely proportional to stop distance, wide-ATR periods
(news, volatile opens) automatically trade smaller size for the same dollar
risk — this is what keeps risk actually fixed rather than nominal.

## 5. Optimization Guidelines

- **Don't over-fit the EMA lengths.** 10/20/50/200 are standard; if you tune
  them, optimize on a walk-forward basis (in-sample tune, out-of-sample
  test) rather than maximizing net profit on the full history.
- **Tune `atrTrailMult` before `atrSlMult`.** The initial stop mostly sets
  your worst-case loss per trade; the trail multiple is what determines how
  much of a trend you actually capture. Start at trail = initial (2.0/2.0)
  and only loosen the trail if you're getting stopped out inside normal
  chop.
- **Check trade count per regime**, not just aggregate stats — a 1H EMA
  cross strategy will look very different in a trending 2023-style tape vs.
  a choppy range-bound one. Segment the backtest by year.
- **Commission/slippage inputs matter at this size.** `$0.85` cash-per-contract
  commission and 1 tick slippage are defaults — replace with your actual
  broker's numbers, since a $50-risk strategy is sensitive to cost drag.
- **Session restriction is a lever, not a requirement.** If overnight/Asia
  session data proves noisy or low-quality for your feed, flip
  `restrictSession` on and test RTH-only (0930-1600 for NAS100, exchange
  session for MNQ) as a comparison, rather than assuming 24h is always best.

## 6. Backtesting Checklist

- [ ] Set `pointValue` correctly for the instrument you're testing (MNQ vs
      NQ vs NAS100 CFD) — this alone can 10x your position sizing
- [ ] Set commission/slippage to match your actual broker before trusting
      net profit numbers
- [ ] Confirm `Total Trades > 30` on the dashboard — anything less isn't a
      statistically meaningful sample
- [ ] Check `Max DD %` against what you could actually tolerate on a live
      or funded account, not just against the "< 20%" target in the table
- [ ] Run the backtest across at least 2-3 different market regimes (trend,
      chop, high-vol) using the date-range filter, not just the full
      default history
- [ ] Sanity-check a handful of individual trades on the chart (crossover
      bar, stop level, exit bar) to confirm the logic is doing what you
      think it's doing
- [ ] Re-run with `restrictSession` on vs. off to see how much of the edge
      (or noise) comes from the overnight/Asia session

## 7. Common Mistakes

- **Wrong point value.** Using NQ's $20/point on an MNQ backtest overstates
  P&L by 10x and understates true position size needed.
- **Ignoring commission on a small account.** At $50 risk/trade, a few
  dollars of round-trip commission is a meaningful fraction of expectancy —
  don't backtest commission-free and then trade live.
- **Reading the dashboard's "Achieved RR" as guaranteed forward RR.** It's
  a historical average across closed trades, not a promise per trade — the
  ATR trail means individual trade RR varies a lot.
- **Assuming 50/200 > filter alone equals "confirmed trend"** and disabling
  the crossunder exit — the strategy is designed to exit fast on the
  opposite cross specifically because the 50/200 filter is slow to flip.
- **Over-optimizing session/date filters** to the point of curve-fitting to
  a specific historical window — treat `restrictSession` and the backtest
  date range as diagnostic tools, not knobs to maximize backtest profit.

## 8. Future Enhancements

- **TradingView alerts → PickMyTrade → Tradovate**: wire up
  `alertcondition()`-driven webhook alerts (already defined for long/short
  entry and exit) through PickMyTrade to auto-execute on a Tradovate funded
  or live account. Test on a sim/eval account first.
- **Partial profit-taking**: split the exit into a first target at 1×ATR
  (reduce size) and let the remainder ride the trail.
- **Volatility regime filter**: skip entries when ATR is in the bottom
  decile of its own history (dead, low-follow-through markets).
- **Multi-timeframe confirmation**: require the 1H trend filter to agree
  with a 4H or daily EMA regime before taking the 1H crossover signal.
- **Correlated-instrument dashboard**: extend the Strategy Report table to
  show MNQ and NAS100 side-by-side equity curves if running both in
  parallel, to catch redundant/correlated risk.
