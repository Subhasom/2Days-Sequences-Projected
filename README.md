# 2 Days Seq Projection

**2-Days Sequences (Projected) with Best Match: an intraday analog-forecasting overlay for TradingView (Pine Script v6)**

| | |
|---|---|
| **Author** | Subhasom Mandal ([github.com/Subhasom](https://github.com/Subhasom)) |
| **Current release** | 12.12092026.0014 |
| **Platform** | TradingView, Pine Script v6, overlay indicator |
| **Market** | NSE (National Stock Exchange of India), intraday charts |
| **File** | `2dsp.<release>.pine` |

> ⚠️ **Disclaimer:** 2 Days Seq Projection is an analysis tool, not financial advice. A past day looking like today does not mean the next day will repeat. Always test on your own instruments and timeframes, and use proper risk management.

---

## Table of contents

1. [What 2 Days Seq Projection does](#1-what-2-days-seq-projection-does)
2. [Installation](#2-installation)
3. [Reading the chart](#3-reading-the-chart)
4. [How it works: full logic](#4-how-it-works-full-logic)
5. [Parameter reference](#5-parameter-reference)
6. [How to use it: practical guide](#6-how-to-use-it-practical-guide)
7. [Limitations and honest notes](#7-limitations-and-honest-notes)
8. [Release versioning](#8-release-versioning)
9. [Credits](#9-credits)

---

## 1. What 2 Days Seq Projection does

2 Days Seq Projection overlays recent price history onto today's session so you can see how the market *might* move today and into tomorrow. It combines three ideas:

- **Rolling 2-day sequences.** At the first bar of every NSE session, the script takes the last **8 completed sessions** and stitches them into **7 overlapping 2-day paths** (Days 2 & 1 ago, Days 3 & 2 ago … Days 8 & 7 ago).
- **Projection from today's open.** Each 2-day path is drawn as a **blue line** starting at today's first bar and running through today and on into the next session. By default each line is shifted to start at today's open, so you compare shapes and move sizes, not old price levels. Newer sequences are more solid; older ones fade.
- **Best-match forecast.** On every bar, today's price path so far is compared with the **first day** of each sequence using a **volume-weighted** or **plain mean-squared error**. The closest sequence is redrawn as a **white line**, showing both "the recent day that looked most like today" and "what happened the day after it."

---

## 2. Installation

1. Open [TradingView](https://www.tradingview.com) and load an NSE symbol (e.g. `NSE:NIFTY`, `NSE:RELIANCE`, `NSE:BANKNIFTY1!`).
2. Switch to an intraday timeframe (3m–15m recommended, see [§6](#6-how-to-use-it-practical-guide)).
3. Open the **Pine Editor** tab at the bottom of the screen.
4. Delete the template code, then paste the full contents of `2dsp.<release>.pine`.
5. Click **Save**, then **Add to chart**. The indicator appears on the chart as **2 Days Seq Projection**.

> 💡 **When updating to a new release**, remove the old indicator from the chart and add the new one. TradingView remembers old settings, so new defaults only apply to a freshly added indicator.

The chart needs at least **8 completed sessions** of history to draw all seven sequences. TradingView normally loads far more, so this only matters on newly listed symbols.

---

## 3. Reading the chart

| Element | Look | Meaning |
|---|---|---|
| **Seq 1 line** | Solid blue line | Days 2 & 1 ago, projected from today's open. The most recent sequence. |
| **Seq 2 – Seq 7 lines** | Progressively fainter blue lines | Older 2-day sequences. Seq 7 (Days 8 & 7 ago) is the faintest. |
| **Best-match line** | White line on top of one blue line | The sequence whose first day most resembles today so far. Can switch to another sequence during the day. |
| **First half of every line** | Over today's candles | The historical "day 1", laid over today for direct comparison with live price. |
| **Second half of every line** | Beyond today's last bar | The historical "day 2", projected into the next session. |
| **Step inside a line** | Jump near the session boundary | The historical overnight gap between the two days in that sequence, carried into the projection. |
| **Fan of blue lines** | Spread of the 7 lines | An informal picture of recent intraday range. Tight fan = recent days behaved alike; wide fan = they diverged. |

> **Important:** the white line is **not a prediction**. It is a replay of a recent day that happened to start like today. It is recalculated on every bar and can change, so what it showed at the close is not what it showed at 10:00.

All lines are deleted and rebuilt on the first bar of each new session, so only the current day's fan is ever on the chart.

---

## 4. How it works: full logic

The pipeline runs on every bar:

```
Price & volume
   │
   ├─► 1. Session detection (first bar of the NSE session?)
   │
   ├─► 2. Rolling storage (today + last 8 sessions, price and volume)
   │
   ├─► 3. Stitch 7 two-day sequences
   │
   ├─► 4. Project onto today (align to today's open) ──► 7 blue lines  (first bar only)
   │
   ├─► 5. Similarity scoring vs today's path so far ──► error per sequence
   │
   └─► 6. Lowest error wins ──► white best-match line  (redrawn every bar)
```

### 4.1 Session detection

`session.isfirstbar` is true on the first bar of each session as defined by the symbol (NSE regular session 09:15–15:30 IST). When it fires, the script stores that bar's `bar_index` as `startBarIdx`. Every line for the day starts at this x-position.

### 4.2 Rolling storage

The script keeps nine price arrays and nine volume arrays, one value per bar:

| Array | Contents |
|---|---|
| `day0` / `vol0` | Today's bars so far (still growing) |
| `day1` / `vol1` | 1 session ago |
| `day2` / `vol2` … `day7` / `vol7` | 2 … 7 sessions ago |
| `day8` / `vol8` | 8 sessions ago |

Every bar, the **Source** value is pushed into `day0` and the bar's `volume` into `vol0`. On the first bar of a new session the arrays rotate: `day8 ← day7`, `day7 ← day6` … `day1 ← day0`, then `day0` is cleared. The session that was in `day8` is dropped.

### 4.3 Stitching the 2-day sequences

Each sequence is the older day followed directly by the newer day (`f_concat`):

| Sequence | Built from | First day (used for matching) | Second day (projection) |
|---|---|---|---|
| Seq 1 | `day2` + `day1` | 2 sessions ago | Yesterday |
| Seq 2 | `day3` + `day2` | 3 sessions ago | 2 sessions ago |
| Seq 3 | `day4` + `day3` | 4 sessions ago | 3 sessions ago |
| Seq 4 | `day5` + `day4` | 5 sessions ago | 4 sessions ago |
| Seq 5 | `day6` + `day5` | 6 sessions ago | 5 sessions ago |
| Seq 6 | `day7` + `day6` | 7 sessions ago | 6 sessions ago |
| Seq 7 | `day8` + `day7` | 8 sessions ago | 7 sessions ago |

Because the days are joined bar-for-bar, the historical overnight gap between them stays inside the line.

### 4.4 Projection onto today

For a sequence with `n` values, `f_buildPoly` creates `n` points:

```
x = startBarIdx + i                     (i = 0 … n−1)
y = sequence[i] + offset

offset = alignOpen ? (todayOpen − sequence[0]) : 0
```

`todayOpen` is the first value in `day0` (with Source = close, that is the first bar's close). With alignment on, the historical path keeps its shape but is slid vertically to start at today's open. With alignment off, it is drawn at the actual historical prices. The points are joined into a straight-segment `polyline` using `xloc.bar_index`, which lets the line extend past the current bar into the future.

The seven blue lines are built **once per session**, on the first bar, each with a fixed transparency step:

| Seq 1 | Seq 2 | Seq 3 | Seq 4 | Seq 5 | Seq 6 | Seq 7 |
|---|---|---|---|---|---|---|
| 15 | 26 | 37 | 48 | 59 | 70 | 81 |

### 4.5 Similarity scoring (best match)

On every bar, with `N` bars of today recorded, `f_calcError` compares today's first `N` values with each sequence's first `N` values (on a normal day these all fall inside the sequence's first day). Both paths are converted to moves from their own start, so price level and gaps do not matter:

```
Δtoday[i] = today[i]    − today[0]
Δseq[i]   = sequence[i] − sequence[0]

error = Σ w[i] · (Δtoday[i] − Δseq[i])²  ÷  max( Σ w[i], 1 )        i = 0 … N−1
```

| Matching Model | Weight `w[i]` | Effect |
|---|---|---|
| **Volume-Weighted Price Match** (default) | `max(today's volume at bar i, 1)` | Mismatches during heavy-volume bars (open, breakouts, close) count more than quiet ones. |
| **Pure Price Similarity (MSE)** | `1` | Every bar counts equally; a plain mean squared error of the shapes. |

A sequence with no data, or with fewer values than today's bar count, gets an error of `1e12` and effectively cannot win. The error is in squared price points, so it is only meaningful for ranking sequences against each other on the same bar.

### 4.6 Choosing and drawing the white line

The lowest of the seven errors wins. Ties go to the **more recent** sequence (checked Seq 1 → Seq 7). If every sequence is disqualified, Seq 1 is used by default, and nothing is drawn if it is empty. The previous white line is deleted and the winner is redrawn with `f_buildPoly`, using the same start bar and alignment rule, in white. Because it is drawn after the blue lines, it sits on top of the blue line it duplicates.

---

## 5. Parameter reference

### General

| Parameter | Default | Description |
|---|---|---|
| Source (price used for the lines) | close | Price stored for every bar and used for both drawing and matching. `hl2` / `hlc3` / `ohlc4` give smoother shapes on noisy timeframes; `open` makes lines start at the true first-bar open. |
| Align each past day to today's open | ✅ On | Shift every historical line (including the white line) so it starts at today's open. Off = draw at the original historical prices. Matching is unaffected either way. |
| Line color | Blue | Base colour for the 7 sequence lines. The script applies its own transparency ladder on top, so pick a fully opaque colour. |
| Line width | 1 | Thickness of the blue lines (1–4). Keep at 1 with all seven shown; raise it if you show only one or two. |

### Show / Hide by Sequence

| Parameter | Default | Description |
|---|---|---|
| Seq 1 (Days 2 & 1 ago) | ✅ On | 2 sessions ago → yesterday. |
| Seq 2 (Days 3 & 2 ago) | ✅ On | 3 sessions ago → 2 sessions ago. |
| Seq 3 (Days 4 & 3 ago) | ✅ On | 4 sessions ago → 3 sessions ago. |
| Seq 4 (Days 5 & 4 ago) | ✅ On | 5 sessions ago → 4 sessions ago. |
| Seq 5 (Days 6 & 5 ago) | ✅ On | 6 sessions ago → 5 sessions ago. |
| Seq 6 (Days 7 & 6 ago) | ✅ On | 7 sessions ago → 6 sessions ago. |
| Seq 7 (Days 8 & 7 ago) | ✅ On | 8 sessions ago → 7 sessions ago. |

> These toggles are **display only**. The best-match engine always evaluates all seven sequences, so the white line can land on a sequence whose blue line is hidden. Hiding all seven and keeping only the white line is a valid, clean setup.

### Best Match Forecast

| Parameter | Default | Description |
|---|---|---|
| Show Best-Match Forecast (White Line) | ✅ On | Show or hide the white line. Matching still runs when hidden. |
| Matching Model | Volume-Weighted Price Match | `Volume-Weighted Price Match` or `Pure Price Similarity (MSE)` (see §4.5). |
| White Line Width | 1 | Thickness of the white line (1–5). 2–3 makes it stand out from the blue fan. |

---

## 6. How to use it: practical guide

**Getting started**

1. Add 2 Days Seq Projection to a 5-minute NSE chart with default settings.
2. Let the first 30–60 minutes of the session trade. Early on the white line is based on very few bars and jumps around.
3. Watch whether today's candles hug the white line. A white line that stays on one sequence for most of the morning indicates a stronger resemblance than one that keeps switching.
4. Use the white line's second half as one scenario for the next session, alongside the spread of the blue lines.

**Recommended timeframes**

A full NSE session is 375 minutes, so each 2-day line has about twice the bars-per-session count:

| Timeframe | Bars per session | Bars per 2-day line | Suitability |
|---|---|---|---|
| 1m | 375 | ~750 | Not recommended (future-drawing limit, see §7) |
| 2m | ~188 | ~375 | OK |
| 3m | 125 | 250 | Good |
| 5m | 75 | 150 | **Recommended** |
| 10m | ~38 | ~75 | Good |
| 15m | 25 | 50 | Good, coarser matching |
| 30m / 1h | 13 / 7 | 26 / 14 | Works, but very few points to match |
| Daily and above | 1 | 2 | Not meaningful |

**Suggested starting points**

| Situation | Try |
|---|---|
| Chart looks cluttered | Turn off Seq 4–7 (or all seven) and set **White Line Width** to 2. |
| Index symbol with no volume (e.g. `NSE:NIFTY`) | Use **Pure Price Similarity (MSE)**. Without volume, both models give the same result anyway. |
| Liquid stock or futures | Keep **Volume-Weighted Price Match**; compare with MSE and keep whichever tracks better on your symbol. |
| Noisy, spiky bars | Set **Source** to `hl2` or `hlc3`. |
| Want to see where recent sessions actually traded | Turn **Align each past day to today's open** off; clusters of lines can act as reference levels. |
| Error about drawing into the future | Move from 1m to 3m or higher. |
| White line hard to see | Use a dark chart theme or raise **White Line Width**. |

**Good practice**

- Use the lines as **context**, not as entries. Combine them with support/resistance, volume and your own risk rules.
- Read the **whole fan**, not just the white line. Where the seven lines end at today's close gives an informal best-case / worst-case range from the last week and a half.
- Remember the memory is only 8 sessions. After an event day (results, budget, policy announcements), recent sequences may not represent normal behaviour.
- For **options**, remember time decay: a projected move that takes a full day may not pay off even if direction is right.

---

## 7. Limitations and honest notes

- **Future-drawing limit on low timeframes.** TradingView limits how far into the future `bar_index`-based drawings can go (around 500 bars). On a 1-minute chart a 2-day line reaches about 750 bars ahead of the session's first bar, which can cause a runtime error or missing lines. Use 2m or higher.
- **Historical volume is not used.** `f_calcError` receives each sequence's volume (`seqVol`) but ignores it. The volume-weighted model decides *which of today's bars matter more*; it does not check whether volume profiles resemble each other.
- **Error is in absolute points.** Matching uses raw price points, not percentages. After a large change in price level within the 8-day window, older days' move sizes are compared directly with today's.
- **Unequal session lengths.** Half-days, Muhurat trading, special sessions or missing bars change a day's bar count. Because lines are laid out bar-by-bar, the session boundary inside a line will not line up with today's close on those days.
- **Short memory.** Only the last 8 sessions are considered. This reflects recent behaviour, not long-term statistics.
- **Early-session instability.** With few bars, the best match rests on little evidence and switches often.
- **The white line repaints by design.** It is recalculated every bar. On historical sessions you only see its final state for the day's last bar, so do not judge or backtest it from how it looks afterwards.
- **White colour is fixed.** There is no colour input for the best-match line; it can be hard to see on a light theme.
- **Performance.** Seven 2-day sequences are copied and scored on every bar. On long histories at low timeframes this may be slow; use a higher timeframe if TradingView reports a calculation timeout.

---

## 8. Release versioning

Releases follow the format:

```
<incremental number>.<DDMMYYYY>.<HHMM>
```

- The number increases by 1 with every code change.
- Date and time are in IST (UTC+5:30).
- The script file is named `2dsp.<release>.pine`.

| Release | Date | Summary |
|---|---|---|
| 12.12092026.0014 | 12 Sep 2026 | Header description, GitHub link. Renamed to "2-Days Sequences (Projected) with Best Match", short name "2 Days Seq Projection". No logic changes. |

---

## 9. Credits

Concept, 2-day sequence projection, best-match scoring and Pine Script v6 implementation: **Subhasom Mandal** ([github.com/Subhasom](https://github.com/Subhasom)).
