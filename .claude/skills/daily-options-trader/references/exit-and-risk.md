# Risk guardrails, position sizing, and exits

All numeric parameters below live in `trading/config.json`. This document explains how to apply them — the config is the source of truth for the numbers themselves.

## Non-negotiable pre-flight checks (every run, before anything else)

1. Read `trading/config.json`. If `enabled` is `false`, stop immediately — log a `skip` event with `reason: "trading disabled in config"` and end the run. Do not evaluate the market, do not touch positions.
2. `get_accounts()`. Find the account matching `account_number`. If it is missing, `agentic_allowed != true`, or `option_level` is not in `required_option_level`, **abort the entire run** — this is a hard safety check, not a soft warning. Log an `error` event and stop. Never fall back to a different account.
3. `get_portfolio(account_number)`. This is your account value for every calculation below.

## Daily halt

If `total_value <= risk_limits.halt_new_trades_below_account_value_usd`:

- Do **not** open any new positions this run (skip section 4/5 of the entry strategy entirely).
- Still run the exit-management step below — de-risking existing positions is not blocked by the halt. Leaving live risk unmanaged because the account is small is worse than closing it down.
- Log a `halt` event with the current account value and continue to the exit step.

This is an absolute floor, not a trailing-drawdown stop — it fires whenever `total_value` is at or below the threshold, regardless of how the account got there.

## Managing existing positions (runs every day, halted or not)

1. `get_option_positions(account_number, nonzero=true, type="long", option_type="call")`.
2. For each open long call:
   - `get_option_quotes(instrument_ids=[<the position's option_id>])` for current bid/mark.
   - Compute `pnl_pct = (current_mark - avg_cost) / avg_cost * 100` using whatever cost-basis field the position object provides.
   - Compute remaining DTE from the contract's expiration date.
   - Close (sell to close) if **any** of:
     - `pnl_pct >= exit.profit_target_pct`
     - `pnl_pct <= exit.stop_loss_pct`
     - `remaining_dte <= exit.time_stop_dte`
   - To close: `review_option_order` then `place_option_order` with a single leg `{option_id, side: "sell", position_effect: "close"}`, `quantity` = full position size, `type: "limit"`, `price` = current bid (or midpoint if the spread is wide — never below bid), `time_in_force: "gfd"`.
   - Log an `exit` event either way (closed, or still held with current pnl_pct and DTE noted in `reason`).

Exits are **never** subject to the daily new-trade cap — that cap only limits new entries (see below).

## Sizing and executing a new entry

Once a candidate has passed every filter in `entry-strategy.md`, size it before placing anything:

```
contract_cost = ask_price * 100
max_by_trade_cap = floor(risk_limits.max_notional_per_trade_usd / contract_cost)

existing_exposure_usd = sum over this underlying's open long-call positions of
                         (quantity * 100 * current_mark_price)
position_room_usd = (risk_limits.max_position_pct_of_account * account_total_value) - existing_exposure_usd
max_by_position_cap = floor(position_room_usd / contract_cost)

contracts_to_buy = min(max_by_trade_cap, max_by_position_cap)
```

- If `contracts_to_buy < 1`, skip this candidate (log `skip`, `reason: "sizing cap too tight"`) and move to the next-ranked candidate.
- Otherwise: `review_option_order` first (single leg `{option_id, side: "buy", position_effect: "open"}`, `quantity: contracts_to_buy`, `type: "limit"`, `price` = current ask or midpoint per the liquidity rule, `time_in_force: "gfd"`, plus `chain_symbol` + `underlying_type: "equity"` so fees/collateral come back). Treat `review_option_order`'s `order_checks` alerts as the automated go/no-go gate for this run (there is no human present to confirm — the user has explicitly authorized fully autonomous execution under these exact limits). Any blocking alert (insufficient buying power, instrument halted, etc.) means skip this candidate and log why; do not override an alert.
- If review is clean, call `place_option_order` with the same parameters plus a fresh `ref_id` (UUID).
- Log an `entry` event with the order id and state.

## Daily new-trade cap

Count only **entries** (new long-call opens) placed this run. Stop opening new positions once you've placed `risk_limits.max_new_trades_per_day` of them, even if more qualifying candidates remain — log the remainder as `skip`, `reason: "daily trade cap reached"`.

## What this skill will never do

- Trade any account other than `trading/config.json`'s `account_number`.
- Place anything other than single-leg long calls (no puts, no spreads, no covered calls, no cash-secured puts, no equities, no crypto) unless the config's `instrument_scope` is explicitly changed.
- Exceed `max_notional_per_trade_usd` per trade or `max_position_pct_of_account` per underlying.
- Open new positions while halted or while `enabled: false`.
- Override or silently reinterpret a blocking `order_checks` alert.
