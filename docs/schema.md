# Schema Reference

All files in this repo follow strict schemas so strategies, indicators, and watchlists are composable across team members.

---

## Strategy (`strategies/*.json`)

A strategy defines bias rules and entry/exit conditions for one or more symbols.

```json
{
  "id": "smc-trend-continuation",
  "name": "SMC Trend Continuation",
  "author": "dan",
  "version": "1.0.0",
  "description": "H1 structure + M5 OB/FVG entries with TPO confirmation",
  "timeframes": ["H1", "M5"],
  "symbols": ["XAUUSD", "EURUSD", "GBPUSD"],
  "indicators_required": ["EMA50", "EMA200", "RSI", "ATR"],
  "bias_rules": [
    {
      "name": "Bullish structure",
      "condition": "EMA50 > EMA200 AND RSI > 50",
      "bias": "bullish"
    },
    {
      "name": "Bearish structure",
      "condition": "EMA50 < EMA200 AND RSI < 50",
      "bias": "bearish"
    }
  ],
  "entry_rules": [
    {
      "direction": "long",
      "condition": "bias == bullish AND price_at_OB AND RSI > 40",
      "notes": "Enter at OB on M5 pullback into H1 bullish structure"
    }
  ],
  "exit_rules": [
    {
      "type": "target",
      "condition": "price_at_next_FVG OR RR >= 3",
      "notes": "Take profit at next FVG or 3R"
    },
    {
      "type": "stop",
      "condition": "price_below_OB",
      "notes": "Stop below OB invalidation"
    }
  ],
  "notes": ""
}
```

---

## Indicator (`indicators/*.json`)

Describes a custom indicator — either a Pine Script or a built-in indicator with custom settings.

```json
{
  "id": "nwe-bands",
  "name": "Nadaraya-Watson Envelope",
  "author": "dan",
  "type": "pinescript",
  "version": "1.0.0",
  "description": "Kernel regression envelope for mean reversion zones",
  "source_file": "indicators/nwe-bands.pine",
  "inputs": {
    "bandwidth": 8,
    "multiplier": 3.0
  },
  "outputs": ["upper_band", "lower_band", "basis"],
  "used_in_strategies": ["mean-reversion"]
}
```

For built-in TradingView indicators:

```json
{
  "id": "rsi-14",
  "name": "Relative Strength Index",
  "author": "tradingview",
  "type": "builtin",
  "inputs": {
    "length": 14,
    "source": "close"
  },
  "outputs": ["rsi"]
}
```

---

## Watchlist (`watchlists/*.json`)

A named list of symbols with optional per-symbol metadata.

```json
{
  "id": "forex-majors",
  "name": "Forex Majors",
  "author": "dan",
  "symbols": [
    { "ticker": "OANDA:EURUSD", "label": "EURUSD" },
    { "ticker": "OANDA:GBPUSD", "label": "GBPUSD" },
    { "ticker": "OANDA:XAUUSD", "label": "XAUUSD" }
  ]
}
```

---

## Playbook (`playbooks/*.json`)

A playbook composes strategies + watchlists into a complete session plan. This is what the morning brief runs against.

```json
{
  "id": "london-session",
  "name": "London Session Playbook",
  "author": "dan",
  "session": "london",
  "watchlist": "forex-majors",
  "strategies": ["smc-trend-continuation", "mean-reversion"],
  "priority": "strategy order defines precedence — first match wins",
  "notes": ""
}
```

---

## rules.json (Active Config)

The root `rules.json` is the active config loaded by the morning brief. It points to the playbook and watchlist to use.

```json
{
  "active_playbook": "london-session",
  "default_watchlist": "forex-majors",
  "bias_timeframe": "H1",
  "entry_timeframe": "M5",
  "risk_per_trade_pct": 1.0
}
```
