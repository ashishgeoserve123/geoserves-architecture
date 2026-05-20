# Analyzed Current Exclusion Debug — IMO 9722182 (STI Gratitude) — February 2025

## Background

A product manager raised a question: on **03-Feb-2025**, the "Analyzed Current" displayed for the passage is **0.58 kts**, which sits within the allowed CP terms range of **0.0 – 5.0 kts**, yet the passage is flagged with condition **`C`** (current excluded). This document traces the full computation pipeline to explain why, and provides a per-hour breakdown for every passage in February 2025 where the condition was in question.

---

## CP Terms — Term ID 147 (STI Gratitude / Koch Shipping Pte Ltd)

| Parameter | Value |
|---|---|
| Vessel | STI Gratitude (IMO 9722182) |
| Term Period | 24-Dec-2021 → 21-Dec-2027 |
| Weather Source | **Analyzed Weather** (`WeatherSourceId = 2`) |
| Current Min Limit | **0.000 kts** |
| Current Max Limit | **5.000 kts** |
| Wind Threshold | **BF 4** |
| Wave Threshold | NULL (not configured) |
| Bad Day Hours | **8 hrs** |
| Bad Day Mode | **`AS_Ratio_Of_24_Hours`** (`BadDayHrsUnitId = 1`) |

### Bad Day Threshold Formula

```
badDayHoursEventRatio = badHours / passageHours
badDayHoursTermsRatio = 8 / 24 = 0.3333...

Exclusion triggered only if: badDayHoursEventRatio > badDayHoursTermsRatio  (strictly greater than)
```

This means for a 24-hour passage the threshold is `badHours > 8` (i.e., 9+ bad hours trigger exclusion; exactly 8 does not).

---

## Root Cause: Two Different Current Values, Two Different Purposes

The `AnalyzedWeather` table stores **two types of rows per form**, differentiated by the `Is24Hour` flag:

| `Is24Hour` | Count (per form) | Purpose |
|---|---|---|
| `0` | ~23 rows | **Hourly observations** — fed into the per-hour exclusion logic |
| `1` | 1 row | **24-hour summary** — stored as `EffectiveCurrent` on the passage; **this is what the UI displays** |

The stored procedure `dbo.Job_GetTCV2AtSeaData` joins `AnalyzedWeather` **without filtering on `Is24Hour`**, so all rows (hourly + summary) are loaded into `TcV2AnalyzedWeatherDetails`. The summary record at the passage end-timestamp becomes the 24th entry in the hourly list.

### How the 0.58 Is Generated

```
AnalyzedWeather table (Is24Hour = 1) for FormId 1700573:
  CalculatedTimeStamp : 2025-02-03 17:00:00
  AnalyzedCurrent     : 0.58   ← weather provider's 24-hr aggregate current
  AnalyzedWind        : 7.84 m/s → BF 3
  AnalyzedWave        : 0.91 m → DSS 3

↓ via SP passage-level query (AZ.AnalyzedCurrent AS AnalyzedCurrent WHERE Is24Hour=1)

TcV2AtSeaPassageDetails.EffectiveCurrent = 0.58   ← displayed in UI as "Analyzed Current"
```

The 0.58 is the **weather provider's own 24-hour summary value** for the passage. It is a separately computed aggregate — not an average of the hourly records — and it has no bearing on whether the passage gets excluded.

### How the Exclusion Decision Is Made

```
Per-hour loop over all 24 TcV2AnalyzedWeatherDetails records:
  if current < 0.0 OR current > 5.0  →  tag record as C

After loop:
  badHours  = count of records tagged C
  badRatio  = badHours / passageHours
  if badRatio > (8/24)  →  isBadDayViolated = true
  if isBadDayViolated AND any C-tagged record exists  →  isCurrentExcluded = true
  final passage condition = C (since isWindExcluded=false, isWaveExcluded=false)
```

---

## Why the Condition Is `C` and NOT `WI`

Wind **never reaches BF 4** at any hourly record across the entire Feb 3 passage. The raw wind from the weather provider ranges from 6.0–9.0 m/s, which converts to **BF 2–3** after unit conversion. The wind threshold of BF 4 is never breached, so `isWindExcluded = false` throughout. Only the current is violated → condition is **`C`**, not `WI` or `WIC`.

---

## Feb 03 — Complete Hourly Breakdown

**Passage FormsId:** 1700573 | **Report Date:** 2025-02-03 17:00 UTC | **Passage Hours:** 24

| # | Timestamp (UTC) | Is24Hour | Raw Wind (m/s) | Stored BF | Raw Wave (m) | Stored DSS | Current (kts) | Wind Violation? | Current Violation? | Tag |
|---|---|---|---|---|---|---|---|---|---|---|
| 1  | 02-Feb 18:00 | 0 | 9.00 | BF 3 | 0.40 | DSS 2 | +1.68 | No (3 ≤ 4) | No (≥ 0.0) | — |
| 2  | 02-Feb 19:00 | 0 | 9.00 | BF 3 | 0.40 | DSS 2 | +1.88 | No | No | — |
| 3  | 02-Feb 20:00 | 0 | 9.00 | BF 3 | 0.40 | DSS 2 | +1.94 | No | No | — |
| 4  | 02-Feb 21:00 | 0 | 9.00 | BF 3 | 0.40 | DSS 2 | +1.84 | No | No | — |
| 5  | 02-Feb 22:00 | 0 | 9.00 | BF 3 | 0.40 | DSS 2 | +1.85 | No | No | — |
| 6  | 02-Feb 23:00 | 0 | 9.00 | BF 3 | 0.40 | DSS 2 | +1.86 | No | No | — |
| 7  | 03-Feb 00:00 | 0 | 9.00 | BF 3 | 0.40 | DSS 2 | +1.88 | No | No | — |
| 8  | 03-Feb 01:00 | 0 | 8.26 | BF 3 | 0.49 | DSS 2 | +1.79 | No | No | — |
| 9  | 03-Feb 02:00 | 0 | 8.00 | BF 3 | 0.61 | DSS 3 | +1.79 | No | No | — |
| 10 | 03-Feb 03:00 | 0 | 8.00 | BF 3 | 0.70 | DSS 3 | +0.25 | No | No | — |
| **11** | **03-Feb 04:00** | 0 | 8.00 | BF 3 | 0.77 | DSS 3 | **-0.54** | No | **YES (< 0.0)** | **C** |
| **12** | **03-Feb 05:00** | 0 | 8.00 | BF 3 | 0.80 | DSS 3 | **-0.25** | No | **YES (< 0.0)** | **C** |
| 13 | 03-Feb 06:00 | 0 | 9.00 | BF 3 | 0.90 | DSS 3 | +0.02 | No | No | — |
| **14** | **03-Feb 07:00** | 0 | 8.42 | BF 3 | 0.96 | DSS 3 | **-0.23** | No | **YES (< 0.0)** | **C** |
| **15** | **03-Feb 08:00** | 0 | 8.00 | BF 3 | 1.07 | DSS 3 | **-0.47** | No | **YES (< 0.0)** | **C** |
| **16** | **03-Feb 09:00** | 0 | 7.63 | BF 3 | 1.18 | DSS 3 | **-0.60** | No | **YES (< 0.0)** | **C** |
| **17** | **03-Feb 10:00** | 0 | 7.00 | BF 3 | 1.30 | DSS 4 | **-0.60** | No | **YES (< 0.0)** | **C** |
| **18** | **03-Feb 11:00** | 0 | 7.00 | BF 3 | 1.40 | DSS 4 | **-0.50** | No | **YES (< 0.0)** | **C** |
| **19** | **03-Feb 12:00** | 0 | 6.00 | BF 2 | 1.50 | DSS 4 | **-0.28** | No | **YES (< 0.0)** | **C** |
| 20 | 03-Feb 13:00 | 0 | 6.00 | BF 2 | 1.58 | DSS 4 | +0.03 | No | No | — |
| 21 | 03-Feb 14:00 | 0 | 6.00 | BF 2 | 1.60 | DSS 4 | +0.05 | No | No | — |
| 22 | 03-Feb 15:00 | 0 | 6.00 | BF 2 | 1.60 | DSS 4 | +0.06 | No | No | — |
| **23** | **03-Feb 16:00** | 0 | 6.00 | BF 2 | 1.60 | DSS 4 | **-0.20** | No | **YES (< 0.0)** | **C** |
| 24 | **03-Feb 17:00** | **1** | 7.84 | BF 3 | 0.91 | DSS 3 | **+0.58** | No | No | — ← **displayed value** |

### Aggregation

| Metric | Value |
|---|---|
| Total records | 24 |
| Bad hours (tagged C) | **9** (hrs 11–12, 14–19, 23) |
| Good hours | 15 |
| Passage hours | 24 |
| Bad ratio | 9 / 24 = **0.375** |
| Threshold ratio | 8 / 24 = **0.333** |
| 0.375 > 0.333? | **YES** → bad day violated |
| isWindExcluded | false (BF 3 never > BF 4) |
| isWaveExcluded | false (not configured) |
| isCurrentExcluded | **true** (C-tagged records exist) |
| **Final condition** | **`C`** |

---

## Feb 05 — Complete Hourly Breakdown (Boundary Case: Exactly at Threshold)

**Passage FormsId:** 1701207 | **Report Date:** 2025-02-05 16:00 UTC | **Passage Hours:** 24

| # | Timestamp (UTC) | BF | Current (kts) | Current Violation? | Tag |
|---|---|---|---|---|---|
| 1  | 04-Feb 17:00 | BF 4 | +0.21 | No | — |
| 2  | 04-Feb 18:00 | BF 4 | +0.43 | No | — |
| 3  | 04-Feb 19:00 | BF 4 | +0.15 | No | — |
| 4  | 04-Feb 20:00 | BF 4 | +0.02 | No | — |
| 5  | 04-Feb 21:00 | BF 4 | +0.25 | No | — |
| 6  | 04-Feb 22:00 | BF 4 | +0.29 | No | — |
| **7**  | **04-Feb 23:00** | BF 4 | **-0.14** | **YES (< 0.0)** | **C** |
| 8  | 05-Feb 00:00 | BF 4 | +0.39 | No | — |
| 9  | 05-Feb 01:00 | BF 4 | +0.27 | No | — |
| 10 | 05-Feb 02:00 | BF 4 | +0.31 | No | — |
| 11 | 05-Feb 03:00 | BF 4 | +0.33 | No | — |
| 12 | 05-Feb 04:00 | BF 4 | +0.33 | No | — |
| 13 | 05-Feb 05:00 | BF 4 | +0.19 | No | — |
| 14 | 05-Feb 06:00 | BF 4 | +0.15 | No | — |
| 15 | 05-Feb 07:00 | BF 4 | +0.02 | No | — |
| **16** | **05-Feb 08:00** | BF 4 | **-0.23** | **YES (< 0.0)** | **C** |
| 17 | 05-Feb 09:00 | BF 4 | +0.10 | No | — |
| **18** | **05-Feb 10:00** | BF 4 | **-0.13** | **YES (< 0.0)** | **C** |
| **19** | **05-Feb 11:00** | BF 4 | **-0.25** | **YES (< 0.0)** | **C** |
| **20** | **05-Feb 12:00** | BF 4 | **-0.29** | **YES (< 0.0)** | **C** |
| **21** | **05-Feb 13:00** | BF 4 | **-0.31** | **YES (< 0.0)** | **C** |
| **22** | **05-Feb 14:00** | BF 4 | **-0.38** | **YES (< 0.0)** | **C** |
| **23** | **05-Feb 15:00** | BF 4 | **-0.35** | **YES (< 0.0)** | **C** |
| 24 | 05-Feb 16:00 | BF 4 | +0.06 | No | — |

### Aggregation

| Metric | Value |
|---|---|
| Bad hours (tagged C) | **8** |
| Passage hours | 24 |
| Bad ratio | 8 / 24 = **0.3333...** |
| Threshold ratio | 8 / 24 = **0.3333...** |
| 0.3333 > 0.3333? | **NO** (equal is not strictly greater) |
| **Final condition** | **`NULL`** (no exclusion) |

This is the edge case that reveals the threshold is a **strict `>`**, not `>=`. Feb 5 sits exactly on the boundary and escapes exclusion.

---

## Full February 2025 Per-Day Review

| Date | Passage Hrs | Bad Current Hrs | Ratio | > 0.333? | Min Current | Max Current | Max Wind (BF) | Displayed Current | Final Condition |
|---|---|---|---|---|---|---|---|---|---|
| 01-Feb | 24.0 | 14 | 0.583 | YES | -0.94 | +1.01 | BF 4 | -0.03 | **C** |
| 02-Feb | 23.0 | 0  | 0.000 | NO  | +0.69 | +1.40 | BF 4 | +1.13 | NULL |
| 03-Feb | 24.0 | 9  | 0.375 | YES | -0.60 | +1.94 | BF 3 | **+0.58** | **C** ← subject of PM query |
| 04-Feb | 23.0 | 11 | 0.478 | YES | -0.65 | +0.37 | BF 3 | -0.06 | **C** |
| 05-Feb | 24.0 | 8  | 0.333 | NO (equal) | -0.38 | +0.43 | BF 4 | +0.06 | NULL ← boundary case |
| 06-Feb | 23.0 | 6  | 0.261 | NO  | -0.27 | +0.36 | BF 4 | +0.11 | NULL |
| 07-Feb | 24.0 | 10 | 0.417 | YES | -0.11 | +0.31 | BF 4 | +0.05 | **C** |
| 08–19-Feb (noon) | various | 0 | — | NO | — | — | varies | varies | NULL (no analyzed weather loaded for these forms) |
| 19-Feb 17:48 | 6.8 | 121* | high | YES | -2.53 | +2.41 | BF 7 | +0.03 | **WIC** (wind + current) |
| 20-Feb 21:30 | — | — | — | — | — | — | — | — | NULL |
| 21-Feb 11:00 | 13.5 | 16 | high | YES | -0.35 | +0.07 | BF 5 | -0.10 | **WIC,SP** |
| 21-Feb 20:00 | 2.5  | 7  | high | YES | -0.14 | +0.04 | BF 4 | -0.04 | **C,SP,X4** |
| 23-Feb 13:00 | — | — | — | — | — | — | — | — | NULL |
| 24-Feb 11:00 | 22.0 | 20 | 0.952 | YES | -0.14 | +0.05 | BF 4 | -0.05 | **C,UW,SP** |
| 24-Feb 22:59 | — | — | — | — | — | — | — | — | NULL |
| 25-Feb 11:00 | 12.0 | 9  | 0.750 | YES | -0.47 | +0.29 | BF 4 | -0.16 | **C,UW** |
| 26-Feb 11:00 | 24.0 | 18 | 0.750 | YES | -0.90 | +0.09 | BF 4 | -0.29 | **C,UW** |
| 27-Feb 08:00 | 21.0 | 9  | 0.429 | YES | -0.84 | +0.58 | BF 5 | +0.03 | **WIC,UW** (wind exceeded too) |
| 28-Feb 15:30 | — | — | — | — | — | — | — | — | NULL |

*The 19-Feb 17:48 passage has 291 weather records for a 6.8-hour passage — this covers a multi-form voyage segment.

---

## Code References

| What | File | Line(s) |
|---|---|---|
| Per-hour current exclusion check | `src/BusinessLogic/Workers/TC2.0/AtSea/WeatherExclusion.cs` | 92 |
| `GetConditions()` — assigns `C` / `WIC` / etc. per record | `src/BusinessLogic/Workers/TC2.0/AtSea/WeatherExclusion.cs` | 281–316 |
| `IdentifyAnalyzedWeather()` — bad-hour aggregation | `src/BusinessLogic/Workers/TC2.0/AtSea/WeatherExclusion.cs` | 234–279 |
| `IsBadHoursViolated()` — ratio check (strictly >) | `src/BusinessLogic/Workers/TC2.0/AtSea/WeatherExclusion.cs` | 182–196 |
| `BadDayHoursType` enum (`AS_Ratio_Of_24_Hours = 1`) | `src/DataModels/TC2.0/Enums.cs` | — |
| `PassageData.AnalyzedCurrent` → `EffectiveCurrent` (display) | `src/DataModels/TC2.0/AtSea/DataFromDB.cs` | 82–83 |
| `WeatherData.AnalyzedCurrentDB` → `AnalyzedCurrent` (hourly) | `src/DataModels/TC2.0/AtSea/DataFromDB.cs` | 221–222 |
| SP passage-level query — `Is24Hour=1` join | `dbo.Job_GetTCV2AtSeaData` | position ~7810 |
| SP hourly weather query — no `Is24Hour` filter | `dbo.Job_GetTCV2AtSeaData` | position ~11490 |

---

## Key Takeaways

1. **The displayed "Analyzed Current" (0.58) and the exclusion decision are derived from different data.** The displayed value is the weather provider's 24-hour summary (`Is24Hour=1`). The exclusion uses all 24 hourly rows including that summary as the final entry.

2. **A passage can show an in-range displayed current and still be excluded** if enough of the individual hourly observations were out of range during the period.

3. **The bad-day check is a strict `>` comparison**, not `>=`. A passage with exactly 8 bad hours in 24 (ratio = 0.3333) is NOT excluded (see Feb 05).

4. **`C` not `WI`**: Wind exclusion fires independently of current exclusion. On Feb 03, wind was BF 2–3 at every hour — below the BF 4 threshold — so only the current exclusion triggered.
