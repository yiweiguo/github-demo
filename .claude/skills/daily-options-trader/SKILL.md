---
name: daily-options-trader
description: Runs one day's cycle of the autonomous long-call options strategy on the user's Robinhood Agentic account — checks safety gates, manages/exits existing positions against profit/stop/time rules, screens a large-cap universe for bullish momentum, sizes and enters new long-call positions within strict risk limits, and logs everything. Invoke once per trading day, normally via a scheduled Routine. Do not invoke for a one-off manual trade request — use the Robinhood MCP tools directly for those.
---

# Daily options trader

> **2026-08-16: options is no longer the account's primary strategy.** The user switched to long-term value-equity investing — see `trading/equity_config.json` and the `value-equity-investor` skill. This skill stays alive only to run off the last open options position (NVDA) to its existing exit rules; `trading/config.json` has `max_new_trades_per_day: 0` so it will never open another options position. Don't raise that back up without the user explicitly asking to resume options trading.

**This skill places real single-leg long-call options orders with real money on a live Robinhood account, with no human confirmation step per trade.** It was built to the user's explicit specification: fully autonomous, high risk accepted, long calls only, on the Agentic account. Every numeric limit below is read from `trading/config.json`, not hardcoded here — that file is the actual control surface.

Read `trading/config.json` in full before doing anything else in a run. If any instruction here conflicts with the current config, the config wins.

## Non-negotiable order of operations

Follow this order every run. Steps 1–2 can abort the run outright; step 3 (exit management) always runs once you pass them; steps 4–5 (new entries) are skippable independently of step 3.

1. **Load config.** Read `trading/config.json`. If `enabled: false`, log a `skip` event to `trading/logs/trade_log.jsonl` (`reason: "trading disabled in config"`), append a one-line note to `trading/logs/daily_summary.md`, and stop. Do nothing else.
2. **Account safety check.** `get_accounts()`. Locate `account_number` from the config in the result. Abort the entire run (log an `error` event, do not proceed) unless **all** hold:
   - the account exists
   - `agentic_allowed == true`
   - `option_level` is `option_level_2` or `option_level_3`
   Never substitute a different account, even if this one is missing or ineligible.
3. **Manage existing positions.** Always runs, halted or not. Full procedure: `references/exit-and-risk.md` → "Managing existing positions". Close anything that hits the profit target, stop loss, or time stop.
4. **Halt check.** `get_portfolio(account_number)`. If `total_value <= risk_limits.halt_new_trades_below_account_value_usd`, log a `halt` event and skip straight to step 6 — no new entries this run. Otherwise continue.
5. **Screen and enter new positions**, up to `risk_limits.max_new_trades_per_day`. Full procedure: `references/entry-strategy.md` (universe → signal → expiration → strike → liquidity) for candidate selection, then `references/exit-and-risk.md` → "Sizing and executing a new entry" for how much to buy and how to place it.
6. **Log and summarize.** Every entry, exit, skip, and halt this run must have a line in `trading/logs/trade_log.jsonl` (schema in `trading/logs/README.md`) and the run must get one short section appended to `trading/logs/daily_summary.md` — account value, what was closed, what was opened, what was skipped and why. Do this even on a halt or a no-op run.

## Hard guardrails (apply on every single order, no exceptions)

- Only the account in `trading/config.json`.
- Only single-leg long calls (`instrument_scope.option_strategies_allowed`) — no puts, spreads, covered calls, cash-secured puts, equities, or crypto.
- Max `risk_limits.max_notional_per_trade_usd` premium per trade.
- Max `risk_limits.max_new_trades_per_day` new entries per run (exits are uncapped).
- Max `risk_limits.max_position_pct_of_account` of current account value in any one underlying.
- No new entries below the halt floor or while `enabled: false`.
- `review_option_order`'s `order_checks` alerts are a hard go/no-go gate, evaluated programmatically since no human is present during a scheduled run — never place through a blocking alert. This programmatic review is standing in for the human confirmation step specifically because the user explicitly asked for fully autonomous execution under these exact numeric limits; it does not license skipping the limits themselves.

## Defaults this skill applies that the user left to your judgment

The user specified: account, full autonomy, long calls only, $500/trade, 5 trades/day, 35% max position, halt at $500 account value, universe = "any liquid large-cap", expiration 14–45 DTE. They explicitly said "not sure" on strike selection and exit rules, and didn't specify an entry trigger. These are documented in `trading/config.json` and explained in the reference docs — review them before turning this on, and edit the config if any default doesn't match intent:

- **Universe**: a static curated list of ~29 large-cap, historically liquid, optionable US equities (`trading/config.json` → `universe.tickers`), re-validated for tradability every run rather than trusted blindly.
- **Entry signal**: 20-day SMA above 50-day SMA, price above the 20-day SMA, RSI(14) between 50–70 — a plain trend/momentum filter, nothing exotic. Ranked by RSI descending.
- **Strike selection**: target delta ~0.35 if available, else ~5% out-of-the-money (2–10% band).
- **Expiration**: middle of the user's 14–45 DTE band, targeting ~28 DTE.
- **Exit rules**: +75% profit target, -50% stop loss, or close by 7 DTE regardless of P&L — whichever hits first.
- **Halt semantics**: an absolute floor ($500), not a trailing drawdown; new entries stop, existing positions still get exited on schedule.
- **Daily cap semantics**: the 5-trade cap counts new entries only; managing (closing) existing positions is never capped.

## How to pause or stop

Set `"enabled": false` in `trading/config.json` and commit it — the very first thing every run does is check this flag. To stop the schedule entirely, disable or delete the Routine that invokes this skill (ask the user before doing either).

## How to change the strategy

Edit `trading/config.json`. Nothing in this skill or its references should need to change for a parameter tweak (risk limits, universe, DTE band, exit thresholds) — only edit the markdown here if the *procedure* itself needs to change.
