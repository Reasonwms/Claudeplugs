# Firecrawl Trading Research Strategy

How Firecrawl fits into the trading research and strategy-development
workflow, feeding current, sourced data into AI analysis instead of relying
on model memory.

## Why

Pine Script strategies, risk rules, and eval requirements in this repo are
only as good as the source data behind them. Broker specs, prop firm rules,
and platform docs change; an LLM's training data does not. Firecrawl closes
that gap: it fetches a page, converts it to clean Markdown, and that
Markdown becomes the grounding context for an AI research pass.

```
Firecrawl  →  Markdown  →  ChatGPT / Claude  →  Strategy notes  →  Pine Script v6  →  TradingView
```

## Priority source list

Ranked by how directly they affect live trading decisions:

1. **Prop firm rules** — daily loss limits, consistency rules, drawdown
   type (trailing vs. static), payout policy. Directly feeds
   `funded-eval-risk-manager`.
2. **CME contract specs** — tick size, point value, session times,
   rollover dates for MNQ and related contracts. Feeds
   `mnq-trading-bot` backtests and position sizing.
3. **Broker / execution API docs** — PickMyTrade and Tradovate API and
   webhook behavior. Feeds `hermes-health-check` and any execution-layer
   changes.
4. **Economic calendars** — high-impact release schedules, used to gate
   entries around news events.
5. **Pine Script v6 / TradingView docs** — language reference, strategy
   tester behavior, alert/webhook syntax. Feeds `pine-script-strategy`.
6. **Risk management and strategy research articles** — external
   perspective on position sizing, ICT/order-flow concepts, etc., used as
   input rather than gospel.

## Workflow

1. **Collect.** Run Firecrawl against a target URL (or a small batch of
   related URLs — a docs section, a firm's rules page). Firecrawl returns
   clean Markdown with nav/ads stripped.
2. **Store.** Save the Markdown under a per-topic folder so it accumulates
   into a durable reference set instead of being re-fetched from scratch
   each time:
   ```
   research/
     prop-firms/
     cme-specs/
     broker-apis/
     pine-script-docs/
     risk-management/
   ```
3. **Analyze.** Feed the saved Markdown to Claude/ChatGPT with a specific
   question (e.g. "What is this firm's trailing drawdown calculation and
   how does it interact with a 2-contract MNQ position?"). Keep the source
   file attached so answers are traceable back to the actual page, not
   paraphrased from memory.
4. **Apply.** Turn the analysis into a concrete artifact: a risk parameter
   in `funded-eval-risk-manager`, a Pine Script v6 change in
   `strategies/`, or an SOP note.
5. **Refresh.** Re-crawl a source when a rule change is suspected (firm
   policy update, CME spec change) rather than on a fixed schedule —
   trading rule pages don't change often enough to justify polling.

## What this repo does NOT need yet

- A vector database / RAG index. The source list above is small enough
  (dozens of pages, not thousands) that flat Markdown files organized by
  topic, searched with grep/Claude directly, are sufficient. Revisit if
  the research folder grows past a few hundred documents.
- Scheduled/automated crawling. Rule and spec pages change infrequently;
  an on-demand crawl before each eval attempt or strategy revision is
  enough and avoids stale-vs-fresh ambiguity.
- A general-purpose scraper for SEO, credit repair, or affiliate research.
  Those are separate use cases with their own source lists and should get
  their own strategy doc if pursued — this one stays scoped to trading.

## Next steps

- Pick the first 3-5 concrete URLs (specific prop firm rules pages, the
  CME MNQ spec page, PickMyTrade/Tradovate API docs) to crawl as a pilot.
- Decide where `research/` should live long-term — this repo, or a
  separate private notes location — before committing to the folder
  structure above.
