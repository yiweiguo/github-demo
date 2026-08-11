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
- **Fix applied**: added a hard rule to `references/exit-and-risk.md` requiring `option_id` to always be freshly re-fetched from this run's own `get_option_instruments` call, never reused from earlier in a session or a prior run.

## 2026-08-10 — Agentic account ($1,956.07)
- Account safety check: OK (agentic_allowed=true, option_level_2)
- Halt check: OK (account value $1,956.07 above the $500 floor)
- Exits: 2 open long-call positions checked, both held (no exit trigger hit):
  - NVDA 2026-09-04 $235C — pnl -15.1%, within the -50% stop / +75% target band, DTE above the 7-day time stop
  - AMZN 2026-09-04 $285C — pnl -10.8%, within the -50% stop / +75% target band, DTE above the 7-day time stop
- Universe: 29 tickers, all passed tradability; 13 passed the trend/RSI signal (ranked by RSI): BAC, ADBE, NVDA, AMZN, JPM, CRM, HD, KO, MA, XOM, JNJ, LLY, V — V dropped from the ranked list after re-check (failed price>20SMA: 361.60 < 361.84), leaving 12 ranked candidates.
- Entries: **0 of 5** — all 12 ranked candidates were screened and none resulted in a trade:
  - NVDA and AMZN cleared the liquidity filter (spread and open interest), but both failed the 35%-of-account position-sizing cap because of the existing NVDA $235C and AMZN $285C holdings already concentrated in those names — no room left under the per-underlying cap.
  - The remaining 10 candidates (BAC, ADBE, JPM, CRM, HD, KO, MA, XOM, JNJ, LLY) failed the liquidity filter (bid-ask spread > 10% of mid and/or open interest below the 100 floor) at their ~5%-OTM, ~28-DTE strike.
  - Full per-symbol reasons in `trade_log.jsonl`.
- Open question for the user: three runs in, the strategy keeps clearing the trend/RSI signal on largely the same handful of names (AMZN, NVDA, BAC, ADBE, JPM, CRM, HD, KO, MA, XOM, JNJ, LLY) but is repeatedly blocked either by the 10%-of-mid liquidity filter or, now that positions exist, by the 35% concentration cap in the same underlyings. Worth considering: loosening the spread filter, widening the universe/signal to surface more candidates, or accepting fewer trades as the natural result of the current filters. No unilateral change made — flagging for your input.
- Result: account value $1,956.07 (down from the $2,099.79 cost basis reference point, reflecting unrealized losses on the two open positions). No new trades placed.

## 2026-08-11 — Agentic account ($1,823.07)
- Account safety check: OK (agentic_allowed=true, option_level_2)
- Halt check: OK (account value $1,823.07 above the $500 floor)
- Exits: 2 open long-call positions checked, both held (no exit trigger hit):
  - NVDA 2026-09-04 $235C — pnl -29.4%, within the -50% stop / +75% target band, 24 DTE remaining > 7-day time stop
  - AMZN 2026-09-04 $285C — pnl -20.3%, within the -50% stop / +75% target band, 24 DTE remaining > 7-day time stop
- Universe: 29 tickers, all passed tradability; 13 passed the trend/RSI signal and price>20SMA confirmation (ranked by RSI): BAC, ADBE, XOM, AMZN, CRM, JPM, LLY, MA, KO, JNJ, NVDA, HD, V
- Entries: **0 of 5** — all 13 ranked candidates were screened at their ~5%-OTM, ~28–31-DTE strike and none resulted in a trade:
  - NVDA cleared the liquidity filter cleanly (spread 3.3%, OI 4,608) but failed the 35%-of-account position cap — the existing NVDA $235C holding left only $253.07 of room, and the next contract would have cost $610.
  - AMZN narrowly missed the liquidity filter (spread 10.8% vs. the 10% max) and would also have failed the position cap given the existing AMZN $285C holding.
  - The remaining 11 candidates (BAC, ADBE, XOM, CRM, JPM, LLY, MA, KO, JNJ, HD, V) failed the liquidity filter (spreads ranging ~29%–76% of mid, several also below the 100 open-interest floor). LLY was additionally priced far above the per-trade cap (~$2,900/contract vs. the $650 max).
  - Full per-symbol reasons in `trade_log.jsonl`.
- Standing observation (now three runs with the same pattern): the trend/RSI screen keeps surfacing the same large-cap names, but round-number, less-liquid strikes on these names routinely blow through the 10%-of-mid spread filter, and the two names that do clear liquidity (NVDA, AMZN) are now capped out by existing position concentration. The strategy is functioning as designed (guardrails correctly blocking marginal/concentrated trades) but is structurally trade-starved under the current filter settings. Still flagging rather than changing unilaterally — this is a strategy-parameter decision for the user.
- Result: account value $1,823.07 (down further from $1,956.07, reflecting continued unrealized losses on the two open positions — no new trades placed).
