# Fixes Applied — Live vs Backtest Win Rate Gap

## Problem

The bot was closing ~90% of live trades as losses despite the same strategy
backtesting cleanly. Loss sizes were uniformly clustered near the SL distance
(0.5%), and log analysis showed positive adverse slippage on nearly every
trade. Multiple correlated SHORT entries were also opening within the same
second on different alts and stopping out together.

## Root cause

`step3_order_manager.py` rebased the SL/TP off the **actual fill price** after
slippage, instead of the **signal price** the strategy intended:

```python
# OLD (buggy)
sl_pct   = signal.sl_price / signal.entry_price
sl_price = actual_entry * sl_pct
```

When an entry filled 0.10–0.15% past the signal price (normal for market
orders on confirmed crossover candles), the SL got pushed the same 0.10–0.15%
deeper. Combined with a flat 0.5% SL, this left the trade pre-stopped before
it had a chance to develop.

## Fixes

### Fix 1 — Slippage rejection (step3_order_manager.py)
After the entry fills, compare actual fill to signal price. If the fill is
worse by more than `MAX_ADVERSE_SLIPPAGE_PCT` (default 0.15%), close the
position immediately and skip the trade. Better to take a tiny round-trip fee
loss than a full SL hit on a pre-stopped trade.

### Fix 2 — Anchor SL/TP to signal price (step3_order_manager.py)
SL and TP are now placed at `signal.sl_price` and `signal.tp_price` directly,
not rebased off the slipped entry. This makes live execution match the
backtest assumption that SL/TP distances are measured from the signal candle's
close.

### Fix 3 — ATR-based stops (step2_signal_detector.py)
SL distance is now `max(0.5%, 1.5 × ATR%)` instead of a flat 0.5%. TP distance
is `3 × SL distance` (preserving the original 1:3 risk-reward). On volatile
candles where the average bar range exceeds 0.5%, this prevents normal noise
from chopping out the trade. Falls back to flat 0.5%/1.5% if ATR is
unavailable for any reason.

### Fix 4 — Per-bar correlated entry cap (step3_order_manager.py)
Limits new entries to `MAX_NEW_POSITIONS_PER_BAR` (default 3) within any single
15-minute candle. When multiple alts fire EMA crossovers simultaneously
(common during market-wide moves), this prevents a single reversal tick from
stopping out 4–5 positions at once.

## Tunable parameters

In `step3_order_manager.py`:
- `MAX_ADVERSE_SLIPPAGE_PCT = 0.15` — tighten to 0.10 for stricter execution,
  loosen to 0.20 if too many trades are getting rejected
- `MAX_NEW_POSITIONS_PER_BAR = 3` — raise if you want more aggressive
  participation, lower if you still see clustered losses

In `step2_signal_detector.py`:
- `S1_ATR_SL_MULT = 1.5` — raise to 2.0 for wider stops on volatile assets
- `S1_RR_RATIO = 3.0` — your original 1:3 ratio; lower it (e.g. 2.0) if you
  want higher win rate at the cost of smaller wins

## What was NOT changed

- Entry logic (the 6 filters in `_check_s1`)
- Strategy parameters (ADX threshold, EMA periods)
- WebSocket / candle engine
- Order sizing / leverage / margin caps
- Symbol list

## Files modified

- `step2_signal_detector.py` — ATR-based SL/TP, extra indicators in signal
- `step3_order_manager.py` — slippage rejection, anchored SL/TP, per-bar cap
