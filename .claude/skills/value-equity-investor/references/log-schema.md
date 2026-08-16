# Log schema

`trading/logs/equity_trade_log.jsonl` — append-only, one JSON object per line, newest at the bottom. Never rewrite or delete prior lines.

| field | type | notes |
|---|---|---|
| `timestamp` | string | RFC3339 UTC, when this event was logged |
| `run_date` | string | `YYYY-MM-DD`, the calendar date of this run |
| `event` | string | one of `entry`, `skip`, `halt`, `error` |
| `account_number` | string | always the config's account_number |
| `account_value_usd` | number | `total_value` from `get_portfolio` at time of event |
| `symbol` | string \| null | ticker, or null for a run-level summary line (e.g. daily cap reached) |
| `side` | string \| null | `"buy"` for entries, else null. This skill never logs a `"sell"`. |
| `dollar_amount` | number \| null | requested dollar_amount for a buy, else null |
| `quantity_shares` | number \| null | filled quantity if known, else null (fractional orders often fill async) |
| `order_id` | string \| null | Robinhood order id for entries, else null |
| `order_state` | string \| null | order state at log time (e.g. `queued`, `confirmed`, `filled`) |
| `reason` | string | for `entry`: which quantitative + qualitative checks it passed. For `skip`: the specific reason (value-trap trend, sector concentration, sizing too small, already held, foreign/complex structure, daily cap reached, etc.). For `halt`/`error`: what triggered it. |

`trading/logs/equity_daily_summary.md` — human-readable, one `## YYYY-MM-DD` section per run appended at the bottom, mirroring the format already established in `trading/logs/daily_summary.md` for the options skill: account value, what was bought and why, what was screened out and why, and one line of running commentary on how the diversification target is tracking.
