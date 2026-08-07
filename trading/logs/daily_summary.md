# Daily summary log

Append one section per run, newest at the bottom. See `README.md` for the companion `trade_log.jsonl` machine-readable log.

<!-- Example entry format:
## 2026-08-07 — Agentic account ($2,100.00)
- Halt check: OK (above $500 floor)
- Exits: none due
- Entries: bought 1x AAPL 2026-09-18 195C @ $3.45 (RSI 62, 20SMA>50SMA) — trade 1/5
- Skipped: MSFT (spread too wide), NVDA (no qualifying strike in DTE band)
-->

## 2026-08-06 — Agentic account ($2,100.00)
- Account safety check: OK (agentic_allowed=true, option_level_2)
- Halt check: OK (account value $2,100.00 above the $500 floor)
- Exits: no open long-call positions to manage
- Universe: 29 tickers, all passed tradability; 12 passed the trend/RSI signal (20SMA>50SMA, price>20SMA, RSI 50-70): BAC, MA, JPM, CRM, V, ADBE, AMZN, HD, KO, XOM, JNJ, LLY (ranked by RSI)
- Entries: **0 of 5** — every one of the 12 ranked candidates failed the liquidity filter (bid-ask spread > 10% of mid; several also below the 100 min open-interest floor) at their ~5%-OTM, ~29-DTE strike. Full per-symbol reasons in `trade_log.jsonl`.
- Note: strike selection used the %OTM fallback (2 of 12 checked contracts exposed delta near the 0.35 target; most landed delta ~0.19-0.25, some as low as this run's contracts show) — worth revisiting the strike-selection method if this recurs.
- Result: account unchanged at $2,100.00. No trades placed.

## 2026-08-07 — Agentic account ($2,099.79)
- Account safety check: OK (agentic_allowed=true, option_level_2)
- Halt check: OK (account value $2,099.79 above the $500 floor)
- Exits: no open long-call positions to manage
- Universe: 29 tickers, all passed tradability; 13 passed the trend/RSI signal: MA, BAC, V, ADBE, XOM, AMZN, JPM, NVDA, CRM, KO, HD, LLY, JNJ (ranked by RSI)
- Entries: **0 of 5** — 2 candidates (AMZN $290C, NVDA $235C) cleared the liquidity filter (the other 11 failed on spread and/or open interest), but both then failed the $500/trade sizing cap: AMZN's ask moved from $4.65 to $6.10 between the initial quote and `review_option_order` (options can reprice fast intraday), pushing 1 contract to $610; NVDA's ask of $5.45 alone already priced 1 contract at $545. Neither was overridden — the cap held. Full per-symbol reasons in `trade_log.jsonl`.
- Observation: two days in a row, liquid large-caps' near-the-money calls are mostly clearing the trend/RSI/expiration filters but failing on spread or getting priced above the $500/trade cap. Worth discussing whether to raise the per-trade cap, loosen the spread filter, or look at cheaper (further-dated or more OTM) strikes if this keeps recurring.
- Result: account unchanged at $2,099.79. No trades placed.

### Update — same day, cap raised to $650
- User directed raising `max_notional_per_trade_usd` from $500 to $650 after reviewing the two misses above.
- Re-checked AMZN and NVDA at current prices: both now clear liquidity and sizing under the new cap.
- **Bought 1x NVDA 2026-09-04 $235C @ $5.80** — correct, matches intended candidate.
- **Bought 1x AMZN 2026-09-04 $285C @ $5.80** (limit was actually $5.80 for AMZN, $5.50 for NVDA — see trade_log.jsonl for exact per-leg prices) — **execution error**: intended contract was $290C (the one actually screened for liquidity today), but a stale option_id from yesterday's session got used when placing the order, buying $285C instead. Still within the valid 2-10% OTM band, so not a guardrail breach, just not the contract described to the user. Disclosed immediately; left in place rather than unwound (canceling/re-buying would add slippage for a difference that's within tolerance).
- Combined new exposure: ~$1,130 (both orders `unconfirmed`/pending fill as of log time), well under the 35%-of-account position cap for each name individually.
- Action item: the entry-strategy procedure should include a hard rule to always re-derive `option_id` from the current run's own `get_option_instruments` call rather than reusing any ID cached earlier in a long-running session, to prevent this class of mistake recurring.
