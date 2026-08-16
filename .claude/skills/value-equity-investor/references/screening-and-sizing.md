# Screening, judgment, and sizing

All numeric parameters below live in `trading/equity_config.json`. This document explains how to apply them — the config is the source of truth for the numbers themselves.

## 1. Run the quantitative screen

Use the saved Robinhood scanner at `entry.screening.scan_id` (`run_scan`). If it's ever missing or deleted, recreate it with `create_scan` using the filters in `entry.screening.quantitative_filters`, via `get_scanner_filter_specs` for exact filter_type names.

**Unit gotcha, confirmed by testing:** PERCENTAGE-unit filters (net profit margin, ROE, etc.) take fractional values, not whole percentage points. 8% is `"0.08"`, not `"8"` — passing whole numbers silently returns zero matches instead of an error. Dollar-unit and PLAIN-unit filters (market cap, P/E, price) take literal numbers as configured.

This produces a shortlist of statistically cheap, profitable, liquid, decent-cap companies — a starting pool, not a buy list.

## 2. Filter out names already held

Drop any symbol already in the account's equity positions (from step 3 of SKILL.md) — this skill diversifies, it doesn't average down or concentrate further into an existing pick without the user asking.

## 3. Apply qualitative judgment

For each remaining candidate under consideration (roughly the top 15-20 by however you're ranking the shortlist — e.g., lowest P/E, or a blend of P/E and ROE — is plenty to work from):

- **Check the trend, not just the snapshot.** `get_financials(symbols, period="quarterly", limit=6)` — revenue and net income over the last several quarters. A cheap multiple next to visibly declining revenue or margin is the classic value-trap pattern; skip it and say why. A cheap multiple next to flat-to-growing revenue and stable-or-improving margin is the target pattern.
- **Prefer the understandable over the exotic.** For an account this size, favor well-known, standard US common stock. Be cautious with foreign ADRs (VIE structures, delisting/regulatory risk), and with thin/illiquid/highly complex names — the scan will surface some of these and they're not automatically disqualified, but they need a clearer quality signal to earn a slot than a familiar blue-chip does.
- **Watch for sector pile-up.** Low-P/E screens structurally over-represent financials, insurance, and energy (their multiples run lower for structural reasons, not necessarily mispricing). Don't fill the whole target position count from one or two sectors — diversify across as many distinct businesses/sectors as the shortlist reasonably supports.
- **It's fine to buy fewer than `target_position_count` in one run.** There's no obligation to force a fill. If only 3 candidates clear both the quantitative and qualitative bar this run, buy 3 and log the rest as skipped with reasons. More can be added on a later run as the screen refreshes or the account gets more cash.

Log a `skip` event (see `log-schema.md`) for every candidate considered and rejected at this stage, with the specific reason (value-trap trend, sector concentration, foreign/complex structure, etc.) — this is what makes "broader judgment" auditable rather than a black box.

## 4. Size and place each buy

For each candidate that clears both screens, in priority order:

```
account_value = current total_value from get_portfolio
cash_available = current cash from get_portfolio
positions_held = distinct symbols from get_equity_positions
slots_remaining = max(1, entry.target_position_count[1] - positions_held_count)  # use the upper bound of the target range

room_by_position_cap = risk_limits.max_position_pct_of_account * account_value
room_by_cash_split = (cash_available - risk_limits.cash_reserve_usd) / slots_remaining

position_size_usd = min(room_by_position_cap, room_by_cash_split)
```

- If `position_size_usd < risk_limits.min_new_position_usd`, skip this candidate (log `skip`, reason `"sizing: room below min_new_position_usd"`) and move to the next-ranked candidate — don't force an undersized position.
- Otherwise: `review_equity_order` first (`type: "market"`, `dollar_amount: position_size_usd` rounded to cents, `side: "buy"`, `market_hours: "regular_hours"`). Treat its alerts as the automated go/no-go gate — there is no human present to confirm, consistent with the account's existing fully-autonomous authorization. Any blocking alert means skip this candidate and log why.
- If review is clean, call `place_equity_order` with the same parameters plus a fresh `ref_id` (UUID).
- Log an `entry` event with the order id, state, dollar amount, and the specific reasons this candidate passed both the quantitative and qualitative screens.
- Recompute `cash_available` and `slots_remaining` after each fill before sizing the next one this run.

## Daily new-trade cap

Count only buys placed this run against `risk_limits.max_new_trades_per_day`. This is a generous ceiling (20), not a target — most runs will place far fewer than that, since the strategy is capital- and opportunity-constrained, not cap-constrained. Stop buying once the cap is hit even if more qualifying candidates remain, and log the remainder as `skip`, reason `"daily trade cap reached"`.
