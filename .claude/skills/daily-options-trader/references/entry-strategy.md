# Entry strategy detail

All numeric parameters referenced below live in `trading/config.json` — read it first, do not hardcode values from this doc if the config has since been edited.

## 1. Universe

Start from `universe.tickers` in the config. Do not trade any symbol outside this list without the user explicitly adding it to the config first — the skill does not expand its own universe.

Re-validate the universe every run (never assume yesterday's tradability still holds):

1. `get_equity_tradability(account_number, symbols=<batch of up to 10>)` across the full list (3 calls for 29 tickers). Drop any symbol that isn't tradable on this account/session.
2. `get_equity_quotes(symbols=<survivors>)` for current price and a quick sanity check the instrument is actively trading (`has_traded`, recent `last_trade_price`).

## 2. Signal (which survivors are candidates today)

For each surviving symbol, pull daily-bar indicators:

- `get_equity_technical_indicators(symbol, type="sma", period=20, interval="day", start_time=<~120 days ago>, output="latest")`
- `get_equity_technical_indicators(symbol, type="sma", period=50, interval="day", start_time=<~120 days ago>, output="latest")`
- `get_equity_technical_indicators(symbol, type="rsi", period=14, interval="day", start_time=<~60 days ago>, output="latest")`

`start_time` needs enough warm-up bars for the period — 120 calendar days comfortably covers a 50-bar SMA on daily data; trim if the tool rejects the range.

A symbol qualifies as a candidate only if **all** of `entry.signal.rules` in the config hold (as of writing: 20-SMA > 50-SMA, last price > 20-SMA, RSI(14) in [50, 70]). Discard the rest — log them as `skip` with `reason: "signal not met"` only if you already spent a tool call on them; don't bother logging symbols that failed the tradability check.

Rank qualifying candidates by `entry.signal.rank_by` (RSI descending). This ranking determines the order you attempt to build a tradeable contract for each — stop once you've filled the day's trade budget (see risk doc), not once you've exhausted the list.

## 3. Expiration selection

For each candidate, in ranked order:

1. `get_option_chains(underlying_symbol=<symbol>)` → get the chain id and its list of expiration dates.
2. Filter expirations to `[today + expiration_dte_min, today + expiration_dte_max]` days out.
3. Pick the expiration closest to `expiration_dte_target` days out. If none fall in the band, skip this candidate (log `skip`, `reason: "no expiration in DTE band"`) and move to the next.

## 4. Strike selection

1. `get_option_instruments(chain_symbol=<symbol>, expiration_dates=<chosen date>, type="call", state="active", tradability="tradable")`.
2. If the returned contracts expose delta/greeks, pick the strike whose delta is closest to 0.35.
3. Otherwise (fallback), compute `%OTM = (strike - last_price) / last_price * 100` for each strike and pick the one closest to `target_pct_otm`, constrained to `acceptable_pct_otm_range`. If nothing falls in range, skip this candidate.

## 5. Liquidity check

`get_option_quotes(instrument_ids=[<chosen option_id>])`:

- Require open interest ≥ `liquidity_filters.min_option_open_interest` if the field is present in the response.
- Compute spread% = `(ask - bid) / mid * 100`. If it exceeds `max_bid_ask_spread_pct_of_mid`, skip this candidate (log `skip`, `reason: "spread too wide"`) — do not chase illiquid contracts even if the signal is strong.

A candidate that survives all five steps is ready for sizing and execution — see `exit-and-risk.md` for position sizing, the daily trade cap, and how to place the order.
