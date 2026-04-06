# TradingView Playbook

A rules-based morning brief and bias engine for TradingView — scans your watchlist, reads indicator values, and applies configurable strategy rules to generate daily session bias and trade setups.

## Structure

```
tradingview-playbook/
├── rules.json        # Watchlist, bias criteria, entry/exit conditions
├── strategies/       # Per-strategy rule sets
└── README.md
```

## How It Works

1. `rules.json` defines your watchlist and bias rules
2. The TradingView MCP morning brief scans all symbols and reads indicator values
3. Claude applies the rules and returns session bias + setups

## Contributing

Open a PR with your strategy or rule set in `strategies/`.
