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
