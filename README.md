# 2-Days Sequences (Projected) with Best Match

**File:** `2dsp.12.12092026.0014.pine`
**Indicator name:** 2-Days Sequences (Projected) with Best Match
**Short name (chart label):** 2 Days Seq Projection
**Release:** `12.12092026.0014`
**Author:** Subhasom Mandal — [github.com/Subhasom](https://github.com/Subhasom)
**Date:** Saturday 12th Sep 2026
**Platform:** TradingView, Pine Script v6
**Market:** NSE (National Stock Exchange of India), intraday charts

---

## Table of Contents

1. [What the Indicator Does](#1-what-the-indicator-does)
2. [Installation](#2-installation)
3. [Quick Start](#3-quick-start)
4. [How It Works — Detailed Logic](#4-how-it-works--detailed-logic)
5. [User Manual — Every Input Explained](#5-user-manual--every-input-explained)
6. [Reading the Chart](#6-reading-the-chart)
7. [Recommended Timeframes](#7-recommended-timeframes)
8. [Practical Workflows](#8-practical-workflows)
9. [Limitations and Known Behaviours](#9-limitations-and-known-behaviours)
10. [Troubleshooting / FAQ](#10-troubleshooting--faq)
11. [Changelog](#11-changelog)
12. [Disclaimer](#12-disclaimer)

---

## 1. What the Indicator Does

This is an **analog (pattern-memory) forecasting overlay**. The idea behind it is simple: the way price moved on recent days is a reasonable reference for how it *might* move today and tomorrow, and if today is starting out like one particular recent day, the day that followed it is a useful "what could happen next" template.

The indicator does two things:

**Blue lines — 7 historical 2-day sequences.** At the first bar of every NSE session, it takes the last eight completed sessions and builds seven overlapping 2-day paths: Days 2 & 1 ago, Days 3 & 2 ago, and so on up to Days 8 & 7 ago. Each path is drawn starting at today's first bar, so the first half of each line overlays today's session and the second half projects forward into the next session. By default every line is shifted so it starts exactly at today's opening price. Newer sequences are drawn more opaque; older ones fade.

**White line — best-match forecast.** On every bar, the indicator compares how price has moved *so far today* against how price moved during the first day of each of the seven sequences. The sequence whose first day most closely resembles today is redrawn as a white line. Because that sequence is two days long, the white line shows both "the day that looked like today" and "what happened the day after it."

---

## 2. Installation

1. Open TradingView and load an NSE symbol (e.g. `NSE:NIFTY`, `NSE:RELIANCE`, `NSE:BANKNIFTY1!`).
2. Switch to an intraday timeframe (see [Section 7](#7-recommended-timeframes)).
3. Open the **Pine Editor** at the bottom of the screen.
4. Delete the template code, paste the full contents of `2dsp.12.12092026.0014.pine`, and click **Save**.
5. Click **Add to chart**. The indicator appears on the chart as **2 Days Seq Projection**.
6. Open the indicator's **Settings** (gear icon on the indicator label) to adjust inputs.

The script needs at least **8 completed sessions** of history on the chart to draw all seven sequences. TradingView normally loads far more than that, so this is only a concern on very new symbols or heavily limited history.

---

## 3. Quick Start

With the default settings, after adding the script you will see:

- Up to 7 blue lines fanning out from today's open, extending past the end of today's session into the next one.
- One white line lying on top of one of the blue lines — the current best match.
- As the day progresses the white line may jump to a different sequence whenever another historical day starts fitting today better.

Watch where today's candles sit relative to the fan, and treat the white line's second half as a scenario for the next session, not a prediction.

---

## 4. How It Works — Detailed Logic

### 4.1 Session detection

```pine
newDay = session.isfirstbar
```

`session.isfirstbar` is `true` on the first bar of each trading session as defined by the symbol's session (NSE regular session 09:15–15:30 IST). Everything in the script is keyed off this flag. When it fires, the script records `startBarIdx := bar_index`, which becomes the x-coordinate where all lines for today begin.

### 4.2 Rolling storage of sessions

The script keeps nine price arrays and nine volume arrays:

| Array | Contents |
|---|---|
| `day0` / `vol0` | Today's bars so far (still growing) |
| `day1` / `vol1` | 1 session ago |
| `day2` / `vol2` | 2 sessions ago |
| … | … |
| `day8` / `vol8` | 8 sessions ago |

On every bar, the current `Source` value is pushed into `day0` and the bar's `volume` into `vol0`.

On the first bar of a new session, the arrays are **rotated**: `day8` receives a copy of `day7`, `day7` of `day6`, and so on down to `day1` receiving yesterday's `day0`. Then `day0` is cleared and starts filling with the new session. The same happens for the volume arrays. The oldest session (the previous `day8`) is discarded.

Each array holds one value per bar, so its length equals the number of bars in that session (e.g. 75 bars for a full session on a 5-minute chart).

### 4.3 Building the 2-day sequences

A 2-day sequence is the older day's array followed by the newer day's array, joined by `f_concat`:

| Sequence | Built from | First day (used for matching) | Second day (the "next day" projection) |
|---|---|---|---|
| Seq 1 | `day2` + `day1` | 2 sessions ago | Yesterday |
| Seq 2 | `day3` + `day2` | 3 sessions ago | 2 sessions ago |
| Seq 3 | `day4` + `day3` | 4 sessions ago | 3 sessions ago |
| Seq 4 | `day5` + `day4` | 5 sessions ago | 4 sessions ago |
| Seq 5 | `day6` + `day5` | 6 sessions ago | 5 sessions ago |
| Seq 6 | `day7` + `day6` | 7 sessions ago | 6 sessions ago |
| Seq 7 | `day8` + `day7` | 8 sessions ago | 7 sessions ago |

Because the two days are joined bar-for-bar, the **overnight gap** between them (the older day's close to the newer day's open) is preserved inside the line. When the line is projected, that historical gap appears where today's session ends and the next one would begin.

### 4.4 Projecting a sequence onto today (`f_buildPoly`)

For a sequence with `n` values, the helper creates `n` chart points:

```
x = startBarIdx + i                  (i = 0 … n-1)
y = sequence[i] + offset
```

where

```
offset = alignOpen ? (todayOpen − sequence[0]) : 0
```

- `todayOpen` is the first value in `day0`, i.e. today's first-bar `Source` value (the first bar's close if `Source = close`).
- `sequence[0]` is the first value of the older day in the pair.

With alignment on, the whole historical path is slid vertically so that it starts at today's opening value; only its **shape** (point moves relative to its own start) is kept. With alignment off, the line is drawn at the actual historical price levels.

The points are joined into a single straight-segment `polyline` using `xloc.bar_index`, so the line advances one step per bar and extends into the future beyond the current bar.

### 4.5 When the blue lines are drawn

The seven blue polylines are built **once per session**, on the first bar. Before building, the previous session's seven lines are deleted, so only today's fan is ever on the chart. Each sequence gets a progressively higher transparency:

| Sequence | Transparency | Visual |
|---|---|---|
| Seq 1 | 15 | Most solid |
| Seq 2 | 26 | |
| Seq 3 | 37 | |
| Seq 4 | 48 | |
| Seq 5 | 59 | |
| Seq 6 | 70 | |
| Seq 7 | 81 | Faintest |

A sequence is skipped (no line) if its toggle is off or if it has no data yet (e.g. in the first 8 sessions of chart history).

### 4.6 The best-match algorithm (`f_calcError`)

This runs on **every bar** of the session.

Let today have `N` bars so far. For each sequence the script compares today's first `N` values with the sequence's first `N` values (which, on a normal day, all fall inside the sequence's *first* day).

Both paths are converted to **moves relative to their own start**, so absolute price level and gaps do not matter:

```
Δtoday[i] = today[i]    − today[0]
Δseq[i]   = sequence[i] − sequence[0]
```

The error score is a weighted mean squared difference:

```
            Σ  w[i] · (Δtoday[i] − Δseq[i])²
Error  =   ──────────────────────────────────        i = 0 … N-1
            max( Σ w[i] , 1 )
```

The weight depends on the **Matching Model** input:

| Matching Model | Weight `w[i]` | Effect |
|---|---|---|
| Volume-Weighted Price Match (default) | `max(today's volume at bar i, 1)` | Bars where today traded heavily count more. A mismatch during a high-volume opening drive or breakout matters more than a mismatch during a quiet lunch period. |
| Pure Price Similarity (MSE) | `1` | Every bar counts equally — a plain mean squared error of the shapes. |

**Disqualification.** If a sequence has fewer total values than today's bar count (`nSeq < N`), or has no data, its error is set to `1e12` so it effectively cannot win.

**Picking the winner.** The script takes the minimum of the seven errors. If several sequences tie, the check order Seq 1 → Seq 7 means the **most recent** tied sequence wins. If every sequence is disqualified (all errors `1e12`), Seq 1 is selected by default; if it is empty, nothing is drawn.

**Units.** The error is measured in squared price points of the chosen `Source`, so its magnitude is only meaningful for comparing sequences against each other on the same bar, not across symbols.

### 4.7 Drawing the white line

After choosing the winner, the previous white polyline is deleted and a new one is built with `f_buildPoly` — same start bar, same alignment rule, colour white, width from **White Line Width**. Because this happens every bar, the white line can switch between sequences during the day as the evidence changes. Early in the session (few bars) the match is based on very little data and tends to jump around; later it usually stabilises.

The white line is drawn after the blue lines, so it sits on top of the blue line it duplicates.

### 4.8 Full per-bar execution order

1. Detect whether this is the first bar of the session.
2. If yes: record `startBarIdx`, delete old blue lines, rotate the 9 price + 9 volume arrays.
3. Push the current `Source` and `volume` into today's arrays.
4. If first bar: build the 7 blue projections.
5. Build the 7 stitched price/volume sequences.
6. Compute 7 error scores against today's path so far.
7. Pick the lowest error; delete and redraw the white line.

---

## 5. User Manual — Every Input Explained

### Group: General

#### Source (price used for the lines)
- **Type:** Source | **Default:** `close`
- **What it does:** The price series stored for every bar, used both for drawing all lines and for the similarity match.
- **Options & when to use them:**
  - `close` — standard choice; lines represent bar closes.
  - `hl2` / `hlc3` / `ohlc4` — smoother, less sensitive to where exactly each bar closed; good on noisy lower timeframes.
  - `open` — makes the very first point of each line the true session open price.
  - `high` / `low` — rarely useful here; they bias the shape toward one side of each bar.
  - You can also select another indicator's output as the source if it is on the chart.
- **Note:** "Today's open" for alignment is the first stored value of today, so with `close` it is the first bar's close, not the exchange opening print.

#### Align each past day to today's open
- **Type:** Checkbox | **Default:** On
- **On:** Every historical line is shifted vertically so it starts at today's opening value. You compare **shapes and move sizes** (e.g. "that day rallied 120 points by 11:00"). This is the recommended mode.
- **Off:** Lines are drawn at their original historical price levels. Useful for seeing where past sessions actually traded relative to today (acts a bit like a support/resistance map), but the fan can be far from current price after trending days.
- **Also affects:** the white best-match line. The match calculation itself is unaffected, because it always uses relative moves.

#### Line color
- **Type:** Colour | **Default:** Blue
- **What it does:** Base colour for the 7 historical sequence lines. The script applies its own transparency ladder (15 → 81) on top of whatever colour you choose, so choose a fully opaque colour here for best results.
- **Tip:** Pick a colour that contrasts with your candles and with white.

#### Line width
- **Type:** Integer 1–4 | **Default:** 1
- **What it does:** Thickness of the 7 blue lines. Keep at 1 when all seven are shown so the chart stays readable; increase if you only display one or two sequences.

### Group: Show / Hide by Sequence

| Input | Default | Controls |
|---|---|---|
| Seq 1 (Days 2 & 1 ago) | On | Line built from 2 sessions ago → yesterday |
| Seq 2 (Days 3 & 2 ago) | On | 3 sessions ago → 2 sessions ago |
| Seq 3 (Days 4 & 3 ago) | On | 4 sessions ago → 3 sessions ago |
| Seq 4 (Days 5 & 4 ago) | On | 5 sessions ago → 4 sessions ago |
| Seq 5 (Days 6 & 5 ago) | On | 6 sessions ago → 5 sessions ago |
| Seq 6 (Days 7 & 6 ago) | On | 7 sessions ago → 6 sessions ago |
| Seq 7 (Days 8 & 7 ago) | On | 8 sessions ago → 7 sessions ago |

- **What they do:** Show or hide the individual blue lines.
- **Important:** These toggles affect **display only**. The best-match engine always evaluates all seven sequences, so the white line can land on a sequence whose blue line is hidden. That is by design — you can hide all seven blue lines and keep only the white one for a clean chart.
- **Typical setups:**
  - *Clean chart:* all seven off, white line on.
  - *Recent context only:* Seq 1–3 on, Seq 4–7 off.
  - *Full range envelope:* all on — the spread of the fan gives a rough idea of how far price has typically travelled in recent sessions.

### Group: Best Match Forecast

#### Show Best-Match Forecast (White Line)
- **Type:** Checkbox | **Default:** On
- **What it does:** Shows or hides the white best-match line. When off, the matching still runs internally but nothing is drawn.

#### Matching Model
- **Type:** Dropdown | **Default:** Volume-Weighted Price Match
- **Volume-Weighted Price Match:** Each bar's squared difference is weighted by **today's** volume on that bar. Prioritises matching the parts of today's session where real participation happened (usually the open, breakouts, and the close). Recommended for liquid stocks and futures.
- **Pure Price Similarity (MSE):** All bars count equally. Use it on symbols where volume is missing or unreliable — for example **index symbols such as `NSE:NIFTY` or `NSE:BANKNIFTY` typically carry no volume**, in which case every weight falls back to 1 and both models behave the same anyway. Also use it if you want quiet periods to matter as much as active ones.

#### White Line Width
- **Type:** Integer 1–5 | **Default:** 1
- **What it does:** Thickness of the best-match line. Setting it to 2 or 3 makes the forecast stand out clearly from the blue fan.

---

## 6. Reading the Chart

```
 today's open ──►  ════ today (first half of every line) ════ │ ═══ next session (second half) ═══►
                                                              │
                   candles print over this part               │ this part is pure projection
                                                          session end
```

- **First half of the lines:** overlays today. Compare live candles against the fan. If price is tracking one line closely, the white line will usually lock onto it.
- **Second half of the lines:** extends past today's close into the next session. For the white line, this is "what happened on the day after the historical day that most resembled today." It includes that historical overnight gap.
- **Fan width:** how spread out the seven lines are gives a rough, informal sense of recent intraday range. A tight fan means recent days behaved similarly; a wide fan means they diverged.
- **White line switching:** frequent switching means no historical day fits today well; a white line that stays on one sequence for most of the session indicates a stronger resemblance.
- **All lines reset** at the next session's first bar: the fan is rebuilt from the new set of eight prior days, and the previous day's projection is removed.

---

## 7. Recommended Timeframes

A full NSE regular session (09:15–15:30) is 375 minutes. Each 2-day line therefore has roughly twice the bars-per-session count:

| Timeframe | Bars per session | Bars per 2-day line | Suitability |
|---|---|---|---|
| 1 min | 375 | ~750 | Not recommended (see Section 9 — future-drawing limit) |
| 2 min | ~188 | ~375 | OK |
| 3 min | 125 | 250 | Good |
| 5 min | 75 | 150 | **Recommended** |
| 10 min | ~38 | ~75 | Good |
| 15 min | 25 | 50 | Good, coarser matching |
| 30 min / 1 h | 13 / 7 | 26 / 14 | Works, but matching has very few points |
| Daily and above | 1 | 2 | Not meaningful — every bar is a "first bar" |

The 3-minute to 15-minute range gives the best balance between matching detail and a readable chart.

---

## 8. Practical Workflows

**Morning bias check.** After the first 30–60 minutes, look at which sequence the white line has settled on. If it is stable and today's candles are hugging it, the second half of that line is a plausible template for the rest of the session and the next open.

**Scenario planning.** Rather than trusting one line, look at where the seven blue lines end at today's session close. The highest and lowest end points give an informal best-case/worst-case range based on the last week and a half of behaviour.

**Clean forecast view.** Turn off all seven blue lines, set **White Line Width** to 2, and use the chart purely with the best-match line.

**Level study.** Turn **Align each past day to today's open** off to see where recent sessions actually traded. Areas where several historical lines cluster can act as reference levels.

**Index vs. stock.** On indices without volume use **Pure Price Similarity (MSE)**. On stocks and futures try both models and see which one's white line tracks better on your symbol.

---

## 9. Limitations and Known Behaviours

- **Future-drawing limit on low timeframes.** TradingView restricts how far into the future drawings positioned by `bar_index` may be placed (around 500 bars). On a 1-minute chart a 2-day line reaches about 750 bars ahead of the session's first bar, which can cause a runtime error or missing lines. Use 2-minute or higher.
- **Sequence volume is not used.** `f_calcError` receives the historical sequence's volume (`seqVol`) but does not use it; weighting is based only on **today's** volume. The volume-weighted model therefore changes *which bars matter*, not whether volume profiles resemble each other.
- **Absolute-point error.** Matching is in raw price points, not percentages. This is fine within one symbol, but after a large price-level change inside the 8-day window, older days' move sizes are compared directly with today's.
- **Unequal session lengths.** Half-days, Muhurat trading sessions, special sessions, or missing bars make sessions different lengths. Because lines are laid out bar-by-bar, the session boundary inside a 2-day line will not line up with today's actual close on those days.
- **Short history on first days.** The first 8 sessions on the chart cannot produce all seven sequences; lines appear as history accumulates.
- **Only the last 8 sessions are considered.** The "memory" is deliberately short. It reflects recent behaviour, not long-term statistics.
- **Early-session instability.** With only a few bars, the best match is based on little evidence and can switch frequently.
- **White colour is fixed.** The best-match line is always white; there is no colour input. On a light chart theme it may be hard to see — use a dark theme or raise its width.
- **Repainting by design.** The white line is recalculated every bar and its chosen sequence can change. On historical bars you only see the final state for each session's last bar. Do not backtest from what the line looked like after the fact.
- **Processing load.** Seven 2-day arrays are copied and compared on every bar. On long histories at low timeframes this may be slow; reduce loaded history or use a higher timeframe if TradingView reports a calculation timeout.

---

## 10. Troubleshooting / FAQ

**No lines appear.**
Make sure you are on an intraday timeframe and that at least 2–3 sessions of history are loaded. On a daily chart the script does not produce meaningful output.

**Only some blue lines appear.**
Check the Show / Hide toggles. If all are on, the chart may not yet have 8 completed sessions of history.

**I get an error about drawing too far into the future.**
You are likely on a 1-minute chart. Switch to 2 minutes or higher.

**The white line covers one of the blue lines exactly.**
Expected — it is a copy of the winning sequence drawn on top.

**The white line points to a sequence I have hidden.**
Expected — matching always considers all seven sequences regardless of display toggles.

**Both matching models give identical results.**
The symbol probably has no volume data (common on indices), so every weight becomes 1.

**The lines start slightly away from the actual 09:15 open.**
With `Source = close`, "today's open" is the close of the first bar. Set **Source** to `open` if you want lines to start at the first bar's opening price.

---

## 11. Changelog

| Release | Date | Notes |
|---|---|---|
| 12.12092026.0014 | Sat 12 Sep 2026 | Added standard author/release header with updated description. Renamed indicator to "2-Days Sequences (Projected) with Best Match" (short name "2 Days Seq Projection"); file renamed to `2dsp.12.12092026.0014.pine`. No logic changes. |

---

## 12. Disclaimer

This indicator is a visual research tool that replays recent price behaviour. It does not predict the future, and the similarity of today to a past day does not imply the next day will repeat. Nothing produced by this script is financial advice. Always use proper risk management and your own judgement before trading.
