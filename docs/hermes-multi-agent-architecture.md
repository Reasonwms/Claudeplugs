# Hermes Multi-Agent Architecture

A pipeline of narrow, single-purpose agents for developing and vetting
strategy changes on the Hermes/KubChuah bot (Hostinger VPS), before
anything reaches live execution via PickMyTrade/Tradovate.

## Why split it up

One agent trying to do research, strategy design, backtesting, risk
checking, and QA in a single pass tends to skip steps under pressure to
produce an answer. Splitting into a pipeline forces each concern to be
addressed explicitly and makes every stage's output inspectable before it
moves on — especially important for the risk and QA stages, where a
skipped check has real money consequences.

```
Research → Strategy → Backtest/Validation → Risk (KubChuah) → Reviewer/QA → your inbox
```

Each arrow is a handoff: the output of one stage is the complete input to
the next. No stage reaches ahead into execution.

## Roles

### 1. Research Agent
- **Input:** instrument (MNQ, etc.), current date/session.
- **Does:** pulls price action, volatility regime, correlated instruments
  (ES, VIX), upcoming high-impact economic events. This is the Firecrawl
  consumer — see `docs/firecrawl-trading-research-strategy.md` for the
  source list (economic calendars, CME specs).
- **Output:** a short market-conditions summary. No code, no trade ideas.
- **Does not:** touch Pine Script, propose parameters, or see account/risk
  state.

### 2. Strategy Agent
- **Input:** Research Agent's summary + the current live strategy's
  recent performance (win rate, drawdown, exit-type breakdown — the kind
  of data `tradingview-trade-log-analyzer` produces).
- **Does:** proposes Pine Script v6 logic changes or parameter tweaks.
  Explicitly the "creative" stage — ideas, not final production code.
- **Output:** a diff or new script + the reasoning for the change.
- **Does not:** run backtests, check prop-firm rules, or deploy anything.

### 3. Backtest/Validation Agent
- **Input:** Strategy Agent's proposed script.
- **Does:** runs it against historical data via TradingView Strategy
  Tester (or a local backtest harness — see `mnq-trading-bot` skill for
  contract specs and harness conventions). Compares against the current
  live version on win rate, max drawdown, and net PnL.
- **Output:** pass/fail verdict with numbers, not just "looks good."
  Rejects anything that underperforms the live baseline.
- **Does not:** decide if the numbers are compliant with prop-firm rules —
  that's a separate concern, deliberately deferred to the next stage.

### 4. Risk Agent (KubChuah)
- **Input:** a strategy that has already passed backtest validation.
- **Does:** checks it against the active prop-firm rule set — daily loss
  limit, consistency rule, drawdown type (trailing vs. static) — the same
  domain as the `funded-eval-risk-manager` skill. This is a hard gate, not
  a suggestion: a strategy that would blow a consistency rule doesn't
  proceed regardless of how good its backtest numbers are.
- **Output:** viable / not-viable, with the specific rule and margin if
  not-viable.

### 5. Reviewer/QA Agent
- **Input:** a strategy that is both backtest-valid and risk-viable.
- **Does:** final logic check — repainting, look-ahead bias, off-by-one
  session/timezone bugs, alert/webhook syntax that PickMyTrade actually
  expects. The things a careless script would still get wrong even with
  good backtest numbers.
- **Output:** approved-for-review, or flagged with the specific defect.
- **Does not:** auto-approve. Every strategy that reaches this stage still
  needs your manual sign-off — QA catches machine-checkable defects, not
  judgment calls.

### 6. Orchestrator
- A lightweight coordinator — doesn't need to be its own "agent" with
  judgment, just a script/workflow that calls the five stages in order,
  passes each stage's output to the next, stops the pipeline the moment
  any stage fails or flags, and delivers the final package (proposed
  script + backtest numbers + risk check + QA notes) to your inbox for
  approval. Nothing reaches Hermes/live execution without that manual
  approval step.

## Where this runs

- **Hostinger VPS (Hermes):** stays the execution layer only — it runs the
  approved strategy and talks to PickMyTrade/Tradovate. It should not run
  the research/strategy/backtest pipeline itself; keep the dev pipeline
  off the box that's holding live risk.
- **Pipeline execution:** run the five-stage sequence wherever you already
  do development work (this repo, locally, or an n8n workflow if you want
  it scheduled) and only ship the final approved `.pine` file to Hermes.
- **Health check:** `hermes-health-check` skill verifies Hermes'
  connections (Anthropic API, Telegram, Hostinger VPS, broker execution)
  independently of this pipeline — run it before trusting any live
  deployment, especially after credential rotation or a VPS redeploy.

## What this doc does NOT decide yet

- Whether each stage is a literal separate LLM call/agent, or a single
  session working through five explicit checklist steps. Either works;
  the checklist form is simpler to start with and easier to debug.
- Trigger cadence (on-demand vs. scheduled). Given the manual-approval
  gate at the end, on-demand (you kick off a pipeline run when you want a
  new strategy candidate) is the safer default until the pipeline has a
  track record.

## Next steps

- Decide checklist-vs-separate-agents for a first pass.
- Wire Backtest/Validation Agent to an actual data source (TradingView
  export via `tradingview-trade-log-analyzer`, or a local harness per
  `mnq-trading-bot`).
- Define the exact prop-firm rule set the Risk Agent checks against (which
  firm/account — rules differ enough that this needs to be explicit).
