# Contributing to TradingView Playbook

## Workflow

1. **Branch** from `main` — name your branch `<type>/<your-name>/<description>`
   - `strategy/dan/smc-trend-continuation`
   - `indicator/alice/nwe-bands`
   - `playbook/bob/london-session`

2. **Add your file** in the correct folder following the schema in `docs/schema.md`

3. **Open a PR** — describe what the strategy/indicator does and what you tested it on

4. **Review** — at least one other team member reviews before merge

## Folder Guide

| Folder | What goes here |
|--------|---------------|
| `strategies/` | Strategy JSON files — bias rules, entry/exit conditions |
| `indicators/` | Indicator JSON + `.pine` source files |
| `watchlists/` | Symbol lists |
| `playbooks/` | Session playbooks composing strategies + watchlists |
| `docs/` | Schema reference, research notes |

## Naming Conventions

- File names: `kebab-case.json` / `kebab-case.pine`
- IDs: match the file name (without extension)
- Authors: use your first name or handle consistently

## Rules

- Never push directly to `main`
- Every strategy must list `indicators_required` so others know what needs to be on the chart
- Pine scripts go in `indicators/` with a matching `.json` descriptor
- If a strategy has been backtested, add a `backtest/` section to the strategy JSON with PF, win rate, sample size
