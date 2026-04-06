# TPO Indicator — Logic Document

Based on: https://www.tradingview.com/support/solutions/43000713306-time-price-opportunity-tpo-indicator/
Author: Custom build for tradingview-playbook
Target: Pine Script v5

---

## 1. Overview

The TPO (Time Price Opportunity) indicator, also known as Market Profile, divides a session into equal time blocks and tracks which price levels were visited during each block. The result is a horizontal histogram showing time spent at each price — exposing fair value (POC), accepted range (Value Area), and extremes.

---

## 2. Core Concepts

### 2.1 Session
A period of time over which one profile is drawn. Options:
- **Daily** (default) — one profile per trading day
- **Weekly** — one profile per week
- **Monthly** — one profile per month

### 2.2 Time Block
Each session is split into equal-duration blocks. Each block gets a letter:
- Blocks 1–26 → A–Z
- Blocks 27–52 → a–z
- Repeats if > 52 blocks in the session

Block size options: 5m, 10m, 15m, 30m, 1h, 2h, 4h

### 2.3 Price Row
The vertical resolution of the profile. Each row represents a fixed number of ticks.

**Auto row size** (based on last 300 bars):
```
range_ticks = (Highest High - Lowest Low) / syminfo.mintick
ticks_per_row = round(range_ticks / 80 / increment) × increment

Increment scale:
  price 1–100:       increment = 5
  price 100–1000:    increment = 50
  price 1000–10000:  increment = 500
  price 10000–100000 increment = 5000
```

**Manual row size**: user-defined ticks per row.

### 2.4 TPO Block
When a time block visits a price row (i.e. the bar's high–low range touches that row), a TPO block is recorded. One block per letter per row — a letter cannot appear twice in the same row within the same session.

### 2.5 Point of Control (POC)
The price row with the **highest TPO block count** in the session. Represents the fair value price — where the market spent the most time.

### 2.6 Value Area (VA)
The price range containing a configurable percentage (default 70%) of all TPO blocks in the session.

**Calculation (8-step algorithm):**
1. Count total TPO blocks in the session
2. Calculate target: `va_target = floor(total_blocks × va_pct / 100)`
3. Start at POC row, accumulated = POC row count
4. Look at the 2 rows above current VA top and 2 rows below current VA bottom
5. Sum blocks in the 2-row group above vs 2-row group below
6. Add the group with the higher sum; if tied, add the group closer to POC
7. Repeat steps 4–6 until accumulated >= va_target
8. VAH = highest row included in VA, VAL = lowest row included

### 2.7 TPO Midpoint
```
tpo_mid = (session_high + session_low) / 2
```
The median between the profile's price extremes.

### 2.8 Initial Balance Range (IBR)
The high–low range visited during the first N time blocks (default: first 2 blocks = A+B). Shown as a vertical line at the session start. Helps identify the initial auction range.

---

## 3. Data Structures

```
// Per-session state
session_start_time   : int       // timestamp of session open
session_open         : float     // first bar open of session
rows[]               : array     // price row base levels, sorted ascending
tpo_counts[]         : array     // count of TPO blocks per row (parallel to rows[])
tpo_letters[][]      : matrix    // which letter appears at each row (for display)
current_block_idx    : int       // current time block index (0 = A, 1 = B, ...)
block_visited[]      : bool[]    // which rows this block has already visited (dedup)
session_high         : float     // running high of session
session_low          : float     // running low of session
poc_row_idx          : int       // index of POC row
vah_row_idx          : int       // index of VAH row
val_row_idx          : int       // index of VAL row
ib_high              : float     // initial balance high
ib_low               : float     // initial balance low
```

---

## 4. Algorithm — Per Bar

### Step 1: Detect session boundary
```
is_new_session = (period == "D") ? dayofweek changes or date changes
               : (period == "W") ? weekofyear changes
               : (period == "M") ? month changes
```
On new session: reset all per-session state, rebuild rows[] from price range.

### Step 2: Determine current time block index
```
block_duration_mins = user setting (30 default)
mins_since_session_open = (time - session_start_time) / 60000
block_idx = floor(mins_since_session_open / block_duration_mins)
```
If block_idx changed from previous bar: reset block_visited[] for new block.

### Step 3: Map current bar's price range to rows
```
for row_idx from 0 to rows.size()-1:
    row_low  = rows[row_idx]
    row_high = row_low + ticks_per_row * syminfo.mintick
    
    if bar.high >= row_low AND bar.low <= row_high:
        if NOT block_visited[row_idx]:
            tpo_counts[row_idx] += 1
            tpo_letters[row_idx].push(letter(block_idx))
            block_visited[row_idx] = true
```

### Step 4: Update session high/low
```
session_high = math.max(session_high, high)
session_low  = math.min(session_low, low)
```

### Step 5: Update Initial Balance
```
if block_idx < ib_blocks:
    ib_high = math.max(ib_high, high)
    ib_low  = math.min(ib_low, low)
```

### Step 6: Recalculate POC
```
poc_row_idx = array.indexof(tpo_counts, array.max(tpo_counts))
poc_price   = rows[poc_row_idx]
```

### Step 7: Recalculate Value Area
```
total_blocks  = array.sum(tpo_counts)
va_target     = math.floor(total_blocks * va_pct / 100)
accumulated   = tpo_counts[poc_row_idx]
va_top_idx    = poc_row_idx
va_bot_idx    = poc_row_idx

while accumulated < va_target:
    above_sum = sum of tpo_counts[va_top_idx+1] + tpo_counts[va_top_idx+2]
    below_sum = sum of tpo_counts[va_bot_idx-1] + tpo_counts[va_bot_idx-2]
    
    if above_sum >= below_sum:
        va_top_idx += 2 (or 1 if at edge)
        accumulated += above_sum
    else:
        va_bot_idx -= 2 (or 1 if at edge)
        accumulated += below_sum

vah_price = rows[va_top_idx] + ticks_per_row * syminfo.mintick
val_price = rows[va_bot_idx]
```

### Step 8: Draw profile (on session close or real-time)
```
// Draw TPO blocks as boxes
for each row in rows[]:
    if tpo_counts[row] > 0:
        for each letter in tpo_letters[row]:
            draw box at (session_x_offset + letter_index, row_price)
            color: VA = value_area_color, outside VA = outside_color, POC = poc_color

// Draw POC line
draw horizontal line at poc_price from session_start to session_end

// Draw VAH/VAL lines
draw horizontal line at vah_price
draw horizontal line at val_price

// Draw IBR line
draw vertical line at session_start spanning ib_low to ib_high

// Draw TPO Midpoint
draw horizontal dashed line at (session_high + session_low) / 2
```

---

## 5. Visual Design

| Element | Color | Style |
|---------|-------|-------|
| TPO blocks inside VA | Semi-transparent blue | Filled box |
| TPO blocks outside VA | Semi-transparent grey | Filled box |
| POC row | Yellow / Gold | Solid line, thicker |
| VAH line | Green | Dashed |
| VAL line | Red | Dashed |
| TPO Midpoint | White | Dotted |
| IBR line | Orange | Solid vertical |
| Extend POC right | Gold | Dotted, extends to current bar |
| Extend VAH/VAL right | Green/Red | Dotted, extends to current bar |

---

## 6. User Inputs

| Input | Default | Options |
|-------|---------|---------|
| Period | Day | Day / Week / Month |
| Block size | 30m | 5m / 10m / 15m / 30m / 1h / 2h / 4h |
| Row size | Auto | Auto / Manual (ticks) |
| Value Area % | 70 | 1–100 |
| IBR blocks | 2 | 1–10 |
| Sessions to show | 3 | 1–10 |
| Extend POC right | true | bool |
| Extend VAH/VAL right | true | bool |
| Show IBR | true | bool |
| Show Midpoint | true | bool |
| Show letters | true | bool |

---

## 7. Key Constraints (Pine Script v5)

- Max drawing objects: ~500 per script → limit sessions shown
- Use `box.new()` for TPO blocks
- Use `line.new()` for POC / VAH / VAL / IBR
- Use `label.new()` for text labels if needed
- Delete old drawings on session reset to avoid hitting limits
- Arrays limited to 100,000 elements — rows per session well within limit (~200 max rows)
- All calculations must be on `barstate.islast` or `barstate.isconfirmed` for efficiency

---

## 8. Output Values (for MCP readability)

To make POC, VAH, VAL readable by the TradingView MCP `data_get_study_values` tool, expose them as `plot()` values:

```pinescript
plot(poc_price, title="POC", display=display.data_window)
plot(vah_price, title="VAH", display=display.data_window)
plot(val_price, title="VAL", display=display.data_window)
plot(tpo_mid,   title="TPO Midpoint", display=display.data_window)
plot(ib_high,   title="IB High", display=display.data_window)
plot(ib_low,    title="IB Low",  display=display.data_window)
```

This ensures Claude can read POC/VAH/VAL programmatically via `data_get_study_values` without needing screenshots.

---

## 9. Limitations & Simplifications

- Pine Script cannot look back at arbitrary historical bars during a session efficiently — we accumulate state bar-by-bar
- Letter display inside boxes is not possible with `box.new()` text — labels will be used sparingly
- Very short timeframes (1m/3m) with large sessions may create too many bars per block — minimum chart TF should be ≤ block_size
- The indicator works best when chart timeframe ≤ block size (e.g., block size 30m → use on 15m or 30m chart)
