# Intent schema v2 — machine-readable extension

v1 records (free text + conditionals in prose) stay valid for the archive.
From 2026-11-16 (Mon) onward, every **trade** intent also carries a
`machine` block so the analyst-side shadow book can execute it
mechanically. `no_trade` records are never booked; their conditionals are
price-scored in the daily review only.

## machine block

```json
"machine": {
  "compiled_from": "<record_id>",
  "trigger": {
    "type": "zone_stall | level_break_retest | level_touch",
    "zone": [525.0, 527.5],
    "confirm": "5m_close_below_zone_low",
    "break_level": 520.0,
    "retest_level": 519.0
  },
  "instrument": {
    "underlying": "AMD",
    "type": "put_spread | call_spread | put | call",
    "expiry_rule": "nearest",
    "strike_window_pct": 5,
    "delta_window": [0.42, 0.52],
    "delta_target": 0.47,
    "spread_width": 5,
    "qty": 1
  },
  "risk": {
    "stop_underlying": 530.0,
    "targets_underlying": [515.0, 510.0],
    "exit_target_index": 0
  }
}
```

## Trigger semantics (the only compilable forms)

- `zone_stall`: once any 5m bar trades at or above `zone[0]` (zone entry),
  the first later 5m bar that **closes below `zone[0]`** fires the trigger
  (short form; mirror for longs: closes above `zone[1]`).
- `level_break_retest`: a 5m **close below `break_level`** arms the
  trigger; it fires at the first check where price has traded back up to
  `retest_level` since the break and the latest 5m close is below
  `retest_level`. No retest before `valid_until` = expired, untriggered.
- `level_touch`: first trade at or through the level fires immediately.

Fuzzy words ("stalls", "weak", "holds") are not allowed in trade intents
unless compiled into one of the forms above.

## Shadow accounting (mirrors the v3 shadow conventions)

- 1 unit per intent (`qty` contracts / spreads).
- Contract selection at trigger time: nearest expiry, strikes within
  ±`strike_window_pct`% of spot, long-leg |delta| inside `delta_window`,
  closest to `delta_target`. No candidate in window = trigger recorded,
  no fill (same gate as v3's NO_CONTRACT_IN_DELTA_WINDOW).
- Entry at ask (long leg) / bid (short leg); exit at bid / ask.
- Fees $0.65 per contract per side per leg.
- Exits: underlying reaches `stop_underlying` or
  `targets_underlying[exit_target_index]` (checked on latest quotes /
  completed 5m bars at each check), or forced flat at the 15:45 ET check
  for same-day expiries.
- Checks run every ~15 minutes during US hours; **fills are priced at
  check-time quotes**, not at the exact trigger tick. This is a
  scoreboard, not an execution simulator.
- This book is the analyst's own shadow ledger. It is separate from the
  server engine's ledgers and never places real or paper orders.
