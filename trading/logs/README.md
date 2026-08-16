# Trading logs

This directory holds logs for two independent, separately-scheduled strategies on the same Agentic account. Both write here before finishing a run, whether or not they took any action.

- **Options** (`daily-options-trader` skill — winding down since 2026-08-16, see below): `trade_log.jsonl` + `daily_summary.md`.
- **Equity** (`value-equity-investor` skill — the account's primary strategy since 2026-08-16): `equity_trade_log.jsonl` + `equity_daily_summary.md`, schema documented in `.claude/skills/value-equity-investor/references/log-schema.md`.

- `trade_log.jsonl` — append-only, one JSON object per event (entry, exit, skip, halt). Machine-readable audit trail. Never edit past lines; only append.
- `daily_summary.md` — append-only, one short human-readable section per run (date, account value, what happened, why). Read this first to see what the skill has been doing.

On 2026-08-16 the user switched the account's primary strategy from short-dated options to long-term value-equity investing after a rough first week. The options skill stays alive only to manage the last open options position (NVDA) to its existing exit rules — `trade_log.jsonl` / `daily_summary.md` will go quiet once that closes. All new activity going forward is in the equity log files.

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
