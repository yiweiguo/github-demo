---
name: value-equity-investor
description: Runs one day's cycle of the autonomous long-term value-equity strategy on the user's Robinhood Agentic account — screens the market for statistically undervalued, quality companies, applies qualitative judgment to avoid value traps, and buys diversified fractional-share positions with real money and no human confirmation step per trade. Never sells. Invoke once per trading day, normally via a scheduled Routine. Do not invoke for a one-off manual trade request — use the Robinhood MCP tools directly for those.
---

# Value equity investor

**This skill places real equity buy orders with real money on a live Robinhood account, with no human confirmation step per trade. It never sells.** It was built 2026-08-16 when the user switched the Agentic account's primary strategy away from short-dated options after a rough first week (-27%), to a long-term, diversified, value-oriented equity approach. Every numeric limit below is read from `trading/equity_config.json`, not hardcoded here — that file is the actual control surface.

Read `trading/equity_config.json` in full before doing anything else in a run. If any instruction here conflicts with the current config, the config wins.

## Non-negotiable order of operations

1. **Load config.** Read `trading/equity_config.json`. If `enabled: false`, log a `skip` event to `trading/logs/equity_trade_log.jsonl` (`reason: "trading disabled in config"`), append a one-line note to `trading/logs/equity_daily_summary.md`, and stop.
2. **Account safety check.** `get_accounts()`. Locate `account_number` from the config. Abort the run (log an `error` event, do not proceed) unless the account exists and `agentic_allowed == true`. Never substitute a different account. (No option-level requirement — this skill trades equities only.)
3. **Portfolio snapshot.** `get_portfolio(account_number)` for account value and cash, `get_equity_positions(account_number)` for current holdings and how many distinct names are already held.
4. **Halt check.** If `total_value <= risk_limits.halt_new_trades_below_account_value_usd`, log a `halt` event and stop — no buys this run.
5. **Screen and buy**, up to `risk_limits.max_new_trades_per_day` and until `entry.target_position_count` is reached (or the day's cash/opportunity runs out). Full procedure: `references/screening-and-sizing.md`.
6. **Log and summarize.** Every buy, skip, halt, or error this run must have a line in `trading/logs/equity_trade_log.jsonl` (schema in `references/log-schema.md`) and the run must get one short section appended to `trading/logs/equity_daily_summary.md` — account value, positions held, what was bought this run and why, what was screened out and why. Do this even on a halt or no-op run.

## Hard guardrails (apply on every single order, no exceptions)

- Only the account in `trading/equity_config.json`.
- **Buy only. This skill never places a sell order on an equity position, under any condition.** No stop-loss, no profit-target, no rebalancing trim, no thesis-break exit — none of that is automated here. If the user wants a position sold, that's a separate, explicit, one-off instruction from them, not something this skill decides.
- Only long equity, only `instrument_scope.order_type` (market, dollar-denominated, regular_hours) — no margin, no shorting, no options.
- Max `risk_limits.max_position_pct_of_account` of current account value in any one position.
- Max `risk_limits.max_new_trades_per_day` new buys per run.
- No new buys below the halt floor or while `enabled: false`.
- `review_equity_order`'s alerts are a hard go/no-go gate, evaluated programmatically since no human is present during a scheduled run — never place through a blocking alert.

## What this skill will never do

- Sell an equity position for any reason. Ever. Not as a stop loss, not to rebalance, not because a better opportunity appeared.
- Trade options, crypto, or any instrument other than long equity.
- Trade any account other than `trading/equity_config.json`'s `account_number`.
- Exceed the per-position or daily-trade caps in `risk_limits`.
- Buy through a blocking `review_equity_order` alert.

## How to pause or stop

Set `"enabled": false` in `trading/equity_config.json` and commit it. To stop the schedule entirely, disable or delete the Routine that invokes this skill (ask the user first).

## How to change the strategy

Edit `trading/equity_config.json` for parameter tweaks (risk limits, screening thresholds, target position count). Only edit this skill's markdown if the *procedure itself* needs to change — e.g., if the user ever asks for an exit policy, that's a deliberate, explicit change to the "What this skill will never do" section above, not something to add quietly.
