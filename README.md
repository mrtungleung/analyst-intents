# analyst-intents

Append-only records of the analyst layer's trade intents for the
yehangshe GEX / dealer-gamma trading system, kept so every call can be
scored mechanically in backtests: did the trigger fire, did stop or
target hit first, what were MFE / MAE.

## Division of labor (fixed)

- This repo is the **analyst layer only**: pre-market plans, GEX/Flow
  analysis, trade intents, status checks, post-market reviews.
- Execution (orders, cancels, stops, position management, broker
  reconciliation) lives in the server program on the Tencent Tokyo
  host (IBKR Paper + Futu Paper). Nothing in this repo places orders.

## Layout

- `intents/YYYY-MM-DD.jsonl` — one file per ET session date, one JSON
  object per line, **append-only** (never edit or delete a record;
  corrections are new records that reference the original `record_id`).
- `compiled/YYYY-MM-DD.jsonl` — machine-readable compilations of trade
  intents (schema in `SCHEMA.md`); the analyst shadow book runs these.
- `shadow/YYYY-MM-DD.json` — daily shadow-book results (fills and P&L),
  written by the post-close review.

## Shadow book (design B, since 2026-11-13)

Trade intents are shadow-executed on the analyst side, separate from
the server engine: K-lines are pulled every minute (exits judged on
1m bars), the GEX heatmap snapshot every 5 minutes (entries judged on
completed 5m bars), fills are booked at live bid/ask with the v3
accounting conventions, and daily P&L lands in `shadow/`.

## Record schema

| field | meaning |
|---|---|
| `record_id` | `YYYY-MM-DD-NNN`, unique per session |
| `issued_at_utc` | when the intent was issued |
| `session_date_et` | ET trading date |
| `analyst` | who issued it (`muse`) |
| `ticker` | underlying symbol |
| `stance` | `no_trade` / `conditional` / `active` |
| `spot_at_issue`, `prev_close`, `day_change_pct` | price context at issue time |
| `data_snapshot` | the raw inputs behind the call (GEX nodes, flip, walls, flow summary, source + snapshot time). Backtests die when only conclusions are kept — inputs are mandatory. |
| `intent` / `conditionals` | direction, trigger, stop, targets, instrument hint, size hint |
| `catalyst` | known news/regime context, if any |
| `rationale` | one-paragraph reasoning |
| `valid_until_utc` | intent expiry; stale intents are void |

## Scoring (consumer side)

The server pulls this repo on a schedule and joins records with its
own market data: a conditional intent is *triggered* when its trigger
condition is met before `valid_until_utc`; from the trigger bar,
record which of stop / target printed first and the MFE / MAE in
between. `no_trade` records score the road not taken (would the
conditionals have lost money?).
