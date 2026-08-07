# Trading logs

Every run of the `daily-options-trader` skill must write here before finishing, whether or not it took any action.

- `trade_log.jsonl` — append-only, one JSON object per event (entry, exit, skip, halt). Machine-readable audit trail. Never edit past lines; only append.
- `daily_summary.md` — append-only, one short human-readable section per run (date, account value, what happened, why). Read this first to see what the skill has been doing.

## `trade_log.jsonl` event schema

One line per event, each a JSON object:

```json
{
  "timestamp": "2026-08-07T14:35:00Z",
  "run_date": "2026-08-07",
  "event": "entry | exit | skip | halt | error",
  "account_number": "596067660",
  "account_value_usd": 2100.00,
  "symbol": "AAPL",
  "option_id": "uuid-from-get_option_instruments",
  "contract": "AAPL 2026-09-18 195C",
  "side": "buy | sell",
  "quantity_contracts": 1,
  "limit_price": 3.45,
  "order_id": "uuid-from-place_option_order",
  "order_state": "queued | filled | rejected | ...",
  "reason": "free text: signal that triggered entry, exit rule that triggered close, or why a candidate was skipped",
  "pnl_pct": null
}
```

`skip` events (a candidate was considered but not traded) and `halt` events (daily halt condition hit) are logged the same way with the relevant fields filled and others `null` — these are as important as trades for auditing why the account did or didn't act.
