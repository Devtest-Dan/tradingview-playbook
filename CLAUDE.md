# TradingView Playbook — Claude Instructions

This is a collaborative trading system built on top of TradingView's MCP integration. Three team members contribute strategies, indicators, and playbooks. Claude helps each member build, validate, and run their work against the shared schema.

---

## Project Structure

```
tradingview-playbook/
├── strategies/       ← Strategy JSON files (bias rules, entry/exit conditions)
├── indicators/       ← Pine Script files + JSON descriptor per indicator
├── watchlists/       ← Symbol lists (JSON)
├── playbooks/        ← Session playbooks (compose strategies + watchlists)
├── docs/schema.md    ← Authoritative JSON schemas — always follow these
├── CONTRIBUTING.md   ← Branching and PR workflow
└── rules.json        ← Active config read by the morning brief
```

**Always read `docs/schema.md` before creating any strategy, indicator, watchlist, or playbook file.** Never invent fields or deviate from the schema without updating it first and noting the change in your PR.

---

## Git Workflow

- **Never push directly to `main`**
- Branch naming: `<type>/<your-name>/<description>`
  - e.g. `strategy/alice/mean-reversion`, `indicator/bob/vwap-anchored`
- Open a PR and get at least one review before merging
- Commit messages should describe what the strategy does, not just "add file"

---

## Adding a Strategy

1. Read `docs/schema.md` → Strategy section
2. Create `strategies/<id>.json` following the schema exactly
3. Fill in all required fields: `id`, `name`, `author`, `timeframes`, `symbols`, `indicators_required`, `bias_rules`, `entry_rules`, `exit_rules`
4. If backtested, add a `backtest` object:
   ```json
   "backtest": {
     "period": "2024-01-01 to 2025-12-31",
     "symbols_tested": ["XAUUSD", "EURUSD"],
     "profit_factor": 2.08,
     "win_rate_pct": 61,
     "total_trades": 444,
     "notes": ""
   }
   ```
5. Commit on your branch and open a PR

---

## Adding an Indicator

1. Read `docs/schema.md` → Indicator section
2. If it's a Pine Script: save the `.pine` file in `indicators/` and create a matching `.json` descriptor
3. If it's a built-in TradingView indicator with custom settings: JSON descriptor only
4. List which strategies use this indicator in `used_in_strategies`

---

## Adding a Watchlist

1. Read `docs/schema.md` → Watchlist section
2. Use full TradingView tickers including exchange prefix (e.g. `OANDA:EURUSD`, `BINANCE:BTCUSDT`)
3. Save as `watchlists/<id>.json`

---

## Adding a Playbook

1. Read `docs/schema.md` → Playbook section
2. A playbook references existing strategy IDs and a watchlist ID — make sure they exist first
3. Save as `playbooks/<id>.json`
4. When a playbook is ready to go live, update `rules.json` → `active_playbook` to point to it

---

## Running the Morning Brief

The morning brief uses the TradingView MCP tool. It reads `rules.json` to know which playbook and watchlist to scan, then Claude applies the strategy bias rules to the indicator data.

To run it, make sure:
- `rules.json` has `active_playbook` and `default_watchlist` set
- TradingView Desktop is open and the MCP is connected (run `tv_health_check` first)
- The indicators listed in `indicators_required` for each active strategy are visible on the chart

Then invoke: `morning_brief` (no arguments — it reads `rules.json` automatically).

---

## Reviewing a PR

When reviewing a teammate's strategy or indicator PR:
- Check that the schema is followed exactly (`docs/schema.md`)
- Verify `indicators_required` lists everything the bias/entry/exit rules reference
- If backtested, sanity-check the numbers (PF > 1.5, sample size > 30 trades minimum)
- Leave comments on the PR, not in the file itself

---

## Key Principles

- **Schema first** — the JSON schemas are the contract between team members. If your idea doesn't fit the schema, update the schema in a separate PR first.
- **No indicator assumptions** — if a rule references an indicator, it must be in `indicators_required`. Claude will flag missing ones.
- **Backtest before merge to main** — strategies without backtest data go into a `draft/` subfolder until tested.
- **rules.json is the live config** — only update `active_playbook` when the playbook is fully reviewed and ready.
