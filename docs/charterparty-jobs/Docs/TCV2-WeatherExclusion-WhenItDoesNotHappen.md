# TCV2 AtSea — When Weather Exclusion Does NOT Happen: Complete Gate Analysis

> This document maps every single point in the code where weather exclusion can be bypassed,
> silently skipped, or produce no result — covering every `if` guard, every null check, every
> data condition, and every configuration state that causes a report to pass through without
> receiving a weather condition code even when weather may genuinely have been bad.
>
> The user-reported case — **"no analysed weather data for a day so no exclusion happened"** —
> is Gate 11 in this document. All other gates are catalogued the same way.

---

## How to Read This Document

Each "Gate" is one specific point in the code where weather exclusion can be prevented.
For each gate you get:
- Where in the code it lives (file + line)
- What condition causes it to skip
- What the result is (no exclusion? partial? silent crash?)
- What you can check in the data to detect it happened
- Whether it is a bug, by-design, or a data problem

The gates are ordered from the outermost layer (Worker.cs) inward to the innermost comparison.

---

## Table of Contents

1. [The Complete Decision Tree (Text Diagram)](#1-the-complete-decision-tree-text-diagram)
2. [Gate 1 — Terms Has No SeaPerformanceDetails](#2-gate-1--terms-has-no-seaperformancedetails)
3. [Gate 2 — No Raw Passage Data Returned From DB](#3-gate-2--no-raw-passage-data-returned-from-db)
4. [Gate 3 — ExclusionCriteria Is Null](#4-gate-3--exclusioncriteria-is-null)
5. [Gate 4 — Operator Date Filter Removes All Reports](#5-gate-4--operator-date-filter-removes-all-reports)
6. [Gate 5 — TransformData Skipped: Units Never Converted](#6-gate-5--transformdata-skipped-units-never-converted)
7. [Gate 6 — No Weather Data In DB At All For This Vessel](#7-gate-6--no-weather-data-in-db-at-all-for-this-vessel)
8. [Gate 7 — Only One Weather Slot Exists (ProcessWeatherData Does Nothing)](#8-gate-7--only-one-weather-slot-exists-processweatherdata-does-nothing)
9. [Gate 8 — Weather Slots Filtered Out By Hours Validator](#9-gate-8--weather-slots-filtered-out-by-hours-validator)
10. [Gate 9 — Report Is The Departure Report](#10-gate-9--report-is-the-departure-report)
11. [Gate 10 — ExclusionCriteria Is Null At Per-Report Level](#11-gate-10--exclusioncriteria-is-null-at-per-report-level)
12. [Gate 11 — No Weather Slots For This Specific FormsId (The User's Case)](#13-gate-11--no-weather-slots-for-this-specific-formsid-the-users-case)
13. [Gate 12 — All Weather Slots For This Report Are "Fine" (Conditions = None)](#14-gate-12--all-weather-slots-for-this-report-are-fine-conditions--none)
14. [Gate 13 — Silent Exception Swallows The Entire Weather Calculation](#15-gate-13--silent-exception-swallows-the-entire-weather-calculation)
15. [Gate 14 — WeatherSource Is Neither ReportedWeather Nor AnalysedWeather](#16-gate-14--weathersource-is-neither-reportedweather-nor-analysedweather)
16. [Gate 15 — BadDayHoursType Is Unknown](#17-gate-15--baddayhourstype-is-unknown)
17. [Gate 16 — BadDayHours Is Null In Terms (Defaults to 100)](#18-gate-16--baddayhours-is-null-in-terms-defaults-to-100)
18. [Gate 17 — Bad Hours Do Not Exceed The Threshold](#19-gate-17--bad-hours-do-not-exceed-the-threshold)
19. [Gate 18 — SteamingHours Is Zero (Ratio Calculation Produces Zero)](#20-gate-18--steaminghours-is-zero-ratio-calculation-produces-zero)
20. [Gate 19 — WindThreshold Is Null In Terms (Defaults to 9999)](#21-gate-19--windthreshold-is-null-in-terms-defaults-to-9999)
21. [Gate 20 — WaveHeight Is Null In Terms (Defaults to 9999)](#22-gate-20--waveheight-is-null-in-terms-defaults-to-9999)
22. [Gate 21 — ReportedWindForce Is Null (Reported Weather Path)](#23-gate-21--reportedwindforce-is-null-reported-weather-path)
23. [Gate 22 — ReportedWave Is Null (Reported Weather Path)](#24-gate-22--reportedwave-is-null-reported-weather-path)
24. [Gate 23 — No Current Limits Defined In Terms](#25-gate-23--no-current-limits-defined-in-terms)
25. [Gate 24 — Wind Exactly At Threshold (Not Strictly Greater)](#26-gate-24--wind-exactly-at-threshold-not-strictly-greater)
26. [Gate 25 — AnalysedWind/Wave Is Null In Weather Slot](#27-gate-25--analysedwindwave-is-null-in-weather-slot)
27. [Summary Table: All Gates at a Glance](#28-summary-table-all-gates-at-a-glance)

---

## 1. The Complete Decision Tree (Text Diagram)

```
Worker.RunWorkerAsync()
│
├── [Gate 1]  terms == null OR terms.SeaPerformanceDetails empty?
│             → SKIP entire vessel, no data written
│
├── [Gate 2]  dataFromDB.Passages == null or empty?
│             → SKIP entire vessel
│
├── [Gate 3]  terms.ExclusionCriteria == null?
│             → TransformData skipped (units not converted)
│             → new Passages(terms) CRASHES → vessel silently skipped
│
├── [Gate 4]  Operator filter removes all reports?
│             → dataFromDB.Passages becomes empty → SKIP vessel
│
├── [Gate 5]  TransformData: ExclusionCriteria null?
│             → Wind/wave unit conversion skipped
│             → Comparisons use raw unconverted values
│
│   Passages.GetPassage()
│   │
│   ├── [Gate 6]  dataFromDB.Weathers null or empty?
│   │             → ProcessWeatherData NOT called
│   │             → All reports processed with empty weather list
│   │
│   ├── [Gate 7]  Weathers.Count <= 1?
│   │             → ProcessWeatherData loop never runs (starts at index 1)
│   │             → Single slot gets Hours=0 → filtered out in next step
│   │
│   ├── [Gate 8]  After ProcessWeatherData: RemoveAll(Hours<=0 OR Hours>2)
│   │             → Slots with invalid gaps removed
│   │             → If ALL slots removed → empty weather list for all reports
│   │
│   │   GetBasicPassageDetail() — per report loop
│   │   │
│   │   ├── [Gate 9]  Report is the Departure report?
│   │   │             → ENTIRE performance block skipped
│   │   │             → No weather exclusion ever for departure reports
│   │   │
│   │   ├── [Gate 10] exclusionCriteria == null at per-report level?
│   │   │             → Weather exclusion block NOT entered
│   │   │
│   │   ├── [Gate 11] No WeatherData rows for this FormsId?   ← USER'S CASE
│   │   │             → weatherData = empty list
│   │   │             → IdentifyAnalyzedWeather outer if = false
│   │   │             → No exclusion set
│   │   │
│   │   ├── [Gate 12] All weather slots for FormsId have Conditions=None?
│   │   │             → IdentifyAnalyzedWeather outer if = false
│   │   │             → No exclusion (weather was genuinely fine)
│   │   │
│   │   ├── [Gate 13] Exception thrown inside weather block?
│   │   │             → Caught silently (Debug.WriteLine only)
│   │   │             → Report continues with no weather conditions
│   │   │
│   │   │   CalculateWeatherDetails()
│   │   │   │
│   │   │   ├── [Gate 14] WeatherSource not ReportedWeather or AnalysedWeather?
│   │   │   │             → Neither branch entered → all flags false → no exclusion
│   │   │   │
│   │   │   │   IdentifyShipWeather() [Reported path]
│   │   │   │   │
│   │   │   │   ├── [Gate 15] BadDayHoursType == Unknown?
│   │   │   │   │             → IsBadHoursViolated always returns false
│   │   │   │   │
│   │   │   │   ├── [Gate 16] BadDayHours null in terms (becomes 100)?
│   │   │   │   │             → No report period can ever be >= 100 hours
│   │   │   │   │             → Bad day never violated → no exclusion
│   │   │   │   │
│   │   │   │   ├── [Gate 17] Bad hours don't exceed threshold?
│   │   │   │   │             → isBadDayViolated = false → no exclusion
│   │   │   │   │
│   │   │   │   ├── [Gate 18] steamingHours == 0 (Hours null or zero)?
│   │   │   │   │             → Ratio = 0 → bad day never violated
│   │   │   │   │
│   │   │   │   ├── [Gate 19] WindSpeed null in terms (becomes 9999)?
│   │   │   │   │             → ReportedWindForce can never exceed 9999
│   │   │   │   │             → No wind exclusion
│   │   │   │   │
│   │   │   │   ├── [Gate 20] WaveHeight null in terms (becomes 9999)?
│   │   │   │   │             → No wave exclusion
│   │   │   │   │
│   │   │   │   ├── [Gate 21] ReportedWindForce == null?
│   │   │   │   │             → isWindExcluded = false
│   │   │   │   │
│   │   │   │   ├── [Gate 22] ReportedWave == null?
│   │   │   │   │             → isWaveExcluded = false
│   │   │   │   │
│   │   │   │   └── [Gate 23] No current limits in terms?
│   │   │   │                 → _hasCurrentExclusionFromTerms = false
│   │   │   │                 → No current exclusion ever
│   │   │   │
│   │   │   │   IdentifyAnalyzedWeather() [Analysed path]
│   │   │   │   │
│   │   │   │   ├── [Gate 11] weatherData empty → outer if false → skip (same as above)
│   │   │   │   │
│   │   │   │   ├── [Gate 12] No bad slots → outer if false → skip
│   │   │   │   │
│   │   │   │   ├── [Gate 15] BadDayHoursType == Unknown → always false
│   │   │   │   │
│   │   │   │   ├── [Gate 16] BadDayHours null (100) → never reached
│   │   │   │   │
│   │   │   │   ├── [Gate 17] Bad hours < threshold → isBadDayViolated false
│   │   │   │   │
│   │   │   │   └── [Gate 18] steamingHours == 0 → ratio = 0 → false
│   │   │   │
│   │   │   └── [Gate 24] Wind value exactly == threshold (not >)?
│   │   │                 → isWindExcluded = false (strict greater than check)
│   │   │
│   │   └── [Gate 25] AnalyzedWind/Wave is null in slot?
│   │                 → null > threshold = false → slot gets Conditions=None
│   │                 → Slot contributes to goodTime not badTime
```

---

## 2. Gate 1 — Terms Has No SeaPerformanceDetails

**File:** `Worker.cs:57`

```csharp
if (terms != null && terms.SeaPerformanceDetails.CheckNotNullAndExists())
{
    // all processing here
}
```

**What causes it:** The stored procedure `Job_GetTCV2AtSeaTerms` returned no warranty rows (Table 6 was empty), OR `terms` itself is null.

**Result:** The entire vessel is silently skipped. No data is fetched, no passages processed, nothing written to the output tables. Weather exclusion never has a chance to run.

**How to detect it:** Check the output tables — if a vessel has no rows at all for a given run date, this is the most likely cause. Verify the terms setup: does `Job_GetTCV2AtSeaTerms` return warranty rows for this `TermsId`?

**Bug or by-design:** By design. A vessel with no warranted speeds has nothing to compare against.

---

## 3. Gate 2 — No Raw Passage Data Returned From DB

**File:** `Worker.cs:63`

```csharp
if (dataFromDB != null && dataFromDB.Passages != null && dataFromDB.Passages.Count > 0)
```

**What causes it:** The stored procedure `Job_GetTCV2AtSeaData` returned no voyage reports for this vessel in the requested date range.

**Result:** Entire vessel skipped. No weather processing.

**How to detect it:** No rows in output tables for this vessel. Check if voyage reports exist in the source system within the `fromDate`/`toDate` window for this IMO number.

**Bug or by-design:** By design. Nothing to process.

---

## 4. Gate 3 — ExclusionCriteria Is Null

**File:** `Worker.cs:66-70` + `Passages.cs:33`

```csharp
// Worker.cs
if(terms.ExclusionCriteria != null)
{
    _weather = new WeatherExclusion(terms.ExclusionCriteria);
    _weather.TransoformData(terms.ExclusionCriteria, ref dataFromDB);
}

// Passages.cs constructor — runs unconditionally after the above
_weatherExclusion = new WeatherExclusion(terms.ExclusionCriteria); // CRASHES if null
```

**What causes it:** The stored procedure `Job_GetTCV2AtSeaTerms` returned no exclusion criteria row (Table 3 was empty), so `terms.ExclusionCriteria` is null.

**Result:**
- `TransformData` is guarded and correctly skipped.
- But `new Passages(terms)` then crashes in its constructor because `new WeatherExclusion(null)` immediately tries to access `terms.WindSpeed` on a null reference.
- This exception is caught by the vessel-level `try/catch` in `Worker.cs:118`.
- **The vessel is silently skipped.** The error is logged via `_logger.Error(...)`. No data is written.

**How to detect it:** Look at the logs for `[ERR] Exception Logged At:` entries with `NullReferenceException` in the Passages or WeatherExclusion stack. Cross-check: does `Job_GetTCV2AtSeaTerms` return an exclusion row for this `TermsId`?

**Bug or by-design:** This is a **partial bug**. The `Worker.cs` code correctly guards `TransformData` but does NOT guard the `new Passages(terms)` call. If ExclusionCriteria is null, the intent is likely to process passages without weather exclusion — but instead the whole vessel crashes. A fix would be to either ensure ExclusionCriteria always has a default value, or guard `new Passages(terms)` similarly.

---

## 5. Gate 4 — Operator Date Filter Removes All Reports

**File:** `Worker.cs:72-76`

```csharp
List<OperatorDates> dates = _operator.GetValidDates(vessel.OperatorDates);
List<PassageData> passageData = _operator.GetFiltered(dataFromDB.Passages, dates);
dataFromDB.Passages = passageData;
if (dataFromDB.Passages != null && dataFromDB.Passages.Count > 0)
{
    // all processing
}
```

**What causes it:** All voyage reports fall outside the valid operator date windows. Can happen if:
- The vessel's operator dates are not configured.
- The operator dates are configured but none overlap with the date range of the fetched reports.
- `vessel.OperatorDates` is null or empty — in that case `GetValidDates` returns an empty list and `GetFiltered` returns an empty list.

**Result:** `dataFromDB.Passages` becomes empty. The inner `if` is false. Weather exclusion never runs. No data written.

**How to detect it:** Query the vessel's `OperatorDates` and check if they cover the date range being processed.

**Bug or by-design:** By design — only process data for the vessel's active operator period.

---

## 6. Gate 5 — TransformData Skipped: Units Never Converted

**File:** `Worker.cs:66-70`

```csharp
if(terms.ExclusionCriteria != null)
{
    _weather = new WeatherExclusion(terms.ExclusionCriteria);
    _weather.TransoformData(terms.ExclusionCriteria, ref dataFromDB);
}
```

**What causes it:** `terms.ExclusionCriteria` is null (same root cause as Gate 3).

**Result in isolation (if it didn't crash at Gate 3):** Wind and wave values in `PassageData` and `WeatherData` remain in their raw database units (knots for wind, metres for wave). When the charter terms threshold is expressed in Beaufort (BF) or Douglas Sea Scale (DSS), the comparison is between incompatible units:
- Example: Wind in DB = 40 knots. Terms threshold = 7 BF. `40 > 7` = true → INCORRECTLY excluded.
- Example: Wave in DB = 3.0 metres. Terms threshold = 4 DSS. `3.0 > 4` = false → INCORRECTLY NOT excluded (4 DSS ≈ 1.25–2.5m, so 3.0m should be excluded).

**Note:** This gate only matters theoretically if Gate 3 doesn't crash first. In practice, if ExclusionCriteria is null, the vessel crashes before reaching TransformData's effect on the downstream comparison.

**Bug or by-design:** By design the guard exists to prevent the crash — but as noted in Gate 3, the crash happens anyway downstream.

---

## 7. Gate 6 — No Weather Data In DB At All For This Vessel

**File:** `Passages.cs:59-66`

```csharp
if(dataFromDB.Weathers != null && dataFromDB.Weathers.Count > 0)
{
    dataFromDB.Weathers = dataFromDB.Weathers.OrderBy(w => w.AnalyzedWeatherDate).ToList();
    _weatherExclusion.ProcessWeatherData(terms.ExclusionCriteria, ref weathers);
    dataFromDB.Weathers.RemoveAll(w => w.Hours <= 0 || w.Hours > 2);
}
```

**What causes it:** The stored procedure `Job_GetTCV2AtSeaData` returned no rows in Table 6 (WeatherData). This means the external weather provider did not supply any analysed weather data for this vessel in this date range.

**Result for Reported Weather (`WeatherSource = ReportedWeather`):** No impact. Reported weather path does not use `dataFromDB.Weathers`. The ship's own reported wind and wave values are used. Weather exclusion works normally.

**Result for Analysed Weather (`WeatherSource = AnalysedWeather`):** `ProcessWeatherData` is never called. Later, when `CalculateWeatherDetails` is called per report, the `weatherData` list for every `FormsId` will be empty. This means `IdentifyAnalyzedWeather` enters but its outer `if` at line 248 is always false → **no weather exclusion for any report in the entire vessel run**.

**How to detect it:** Check `TcV2AtSeaWeatherDetails` table for this vessel. If it has no rows for the run period, no weather data was available. Check if `WeatherSource = AnalysedWeather` in the charter terms — if so, all reports are unexcluded due to missing data.

**Bug or by-design:** This is a **data gap issue**. The code handles the null gracefully (doesn't crash) but the consequence for the Analysed path is that bad weather days pass through without exclusion. There is no fallback to reported weather when analysed weather is missing.

---

## 8. Gate 7 — Only One Weather Slot Exists (ProcessWeatherData Does Nothing)

**File:** `Passages.cs:64` → `WeatherExclusion.cs:85`

```csharp
public void ProcessWeatherData(ExclusionCriteria terms, ref List<WeatherData> weatherData)
{
    if (weatherData != null && weatherData.Count > 1)  // ← GATE: count must be > 1
    {
        for (int i = 1; i < weatherData.Count; i++)
        {
            weatherData[i].Hours = ...
            ...
        }
    }
}
```

**What causes it:** Only one `WeatherData` row was returned from the DB for the entire vessel date range. The loop starts at `i = 1` so it never runs if `Count == 1`.

**Result:** The single slot is never processed. Its `Hours` field remains the DB default (0). Then `RemoveAll(w => w.Hours <= 0)` removes it. Every report in the vessel run ends up with an empty `weatherData` list → same consequence as Gate 6: no weather exclusion in Analysed path.

**How to detect it:** `TcV2AtSeaWeatherDetails` table for this vessel has zero or one row. The issue is that a single weather observation for an entire voyage is an unusually thin dataset.

**Bug or by-design:** By design — the `Hours` computation requires a previous slot to subtract from. With one slot there is nothing to subtract from. However, the consequence (no exclusion at all) could be considered a gap.

---

## 9. Gate 8 — Weather Slots Filtered Out By Hours Validator

**File:** `Passages.cs:65`

```csharp
dataFromDB.Weathers.RemoveAll(w => w.Hours <= 0 || w.Hours > 2);
```

**What causes it:** After `ProcessWeatherData` computes `Hours` for each slot, slots are removed if:
- `Hours <= 0` — this always removes the first slot in the sequence (which gets `Hours = 0` since there's no predecessor). It also removes any slot where the previous slot has the same timestamp.
- `Hours > 2` — removes slots where the gap to the previous observation is more than 2 hours. This happens when the weather provider has a gap in data (e.g. no data for 3 hours between two observations).

**Result:** After filtering, some or all slots may be removed. Specifically:
- If the weather provider provides data only every 3 hours (e.g. 6-hourly weather data), ALL slots would have `Hours > 2` after the first slot → ALL removed → empty list → no weather exclusion.
- If there is a gap in the data (e.g. 6 hours without weather data), the slot after the gap is removed. Any bad weather in that slot goes undetected.

**Example:** Weather data at 00:00, 01:00, 02:00, 06:00 UTC.
- Slot 00:00: Hours = 0 → **removed**
- Slot 01:00: Hours = 1 → kept
- Slot 02:00: Hours = 1 → kept
- Slot 06:00: Hours = 4 → **removed** (> 2)

If the bad weather was in that 02:00–06:00 gap, it is not represented in any remaining slot.

**How to detect it:** Query the raw `WeatherData` before filtering. Check timestamps — if the provider delivers data at intervals > 2 hours (e.g. every 3 or 6 hours), the filter removes most/all data.

**Bug or by-design:** The 2-hour filter is intentional to prevent large gaps from inflating bad hours (a 10-hour gap would count as 10 hours of bad weather if not filtered). But it also silently discards data from providers who deliver at lower frequency than hourly. There is no warning logged when slots are removed.

---

## 10. Gate 9 — Report Is The Departure Report

**File:** `Passages.cs:188`

```csharp
if (passages.Count > 0 && !passageDetail.Position.Trim().ToLower().Contains(AtSeaConstants.DepartureReportName))
{
    // All performance and exclusion logic here
}
else
{
    passages.Add(passageDetail);  // ← departure just gets added with no exclusion logic
}
```

**What causes it:** The current report is identified as a Departure report (its `Position` field contains "departure").

**Result:** The entire performance block is skipped. No hours, no distance, no fuel, no weather exclusion is evaluated. The departure report is always stored with `Conditions = null`.

**By design:** The departure is the reference starting point. There is no previous report to measure from, so no performance period exists.

---

## 11. Gate 10 — ExclusionCriteria Is Null At Per-Report Level

**File:** `Passages.cs:270`

```csharp
if (exclusionCriteria != null)
{
    try
    {
        // weather exclusion logic
    }
    catch (Exception ex) { ... }
}
```

**What causes it:** `terms.ExclusionCriteria` was null when assigned to the local variable at line 172. Since Gate 3 already explains this leads to a crash in `new Passages(terms)`, this guard would only be reached if somehow `ExclusionCriteria` is null but the constructor didn't crash (theoretically impossible with current code).

**Result if reached:** Weather exclusion block is skipped for every report. All reports get no weather condition codes.

**By design:** The guard exists as a defensive check.

---

## 12. Gate 11 — No Weather Slots For This Specific FormsId (The User's Case)

**File:** `Passages.cs:274` + `WeatherExclusion.cs:248`

```csharp
// Passages.cs — per report
weatherData = dataFromDB.Weathers?.Where(w => w.FormsId == passage.FormsId).ToList()
              ?? new List<WeatherData>();
```

```csharp
// WeatherExclusion.cs — IdentifyAnalyzedWeather
if (weatherData != null && weatherData.Count > 0
    && weatherData.Any(w => w.Conditions != ExclusionConditions.None))
{
    // bad hours calculation and exclusion logic
}
// If we never enter the above block, all flags remain false → no exclusion
```

**What causes it:** This is the exact case the user observed. Even when the overall `dataFromDB.Weathers` list is not empty, the filter `Where(w => w.FormsId == passage.FormsId)` may return an empty list for a specific report if:

1. **The weather provider had no data for that specific day.** The provider delivers hourly observations for most days but has a gap on some dates. The report for that day therefore has no matching `WeatherData` rows.

2. **All weather slots for this FormsId were removed by the Hours filter (Gate 8).** The slots existed but were filtered out because their intervals were > 2 hours or = 0 hours.

3. **The FormsId in WeatherData does not match the FormsId in PassageData.** This can happen if the weather data is linked to a different form ID in the source system (e.g. the weather data was loaded against a corrected/amended form ID while the original form ID is in the passage data).

4. **The report date falls outside the range queried for weather data.** The weather SP might have a different date range parameter than the passages SP.

**Result:** `weatherData` is an empty list for this report. Inside `IdentifyAnalyzedWeather`, the outer `if` condition `weatherData.Count > 0` is false. The code never enters the block. All three flags (`isWindExcluded`, `isWaveExcluded`, `isCurrentExcluded`) remain `false`. No weather condition code is added to `Conditions` for this report.

**The report is treated as a perfectly good weather day even if conditions were genuinely severe.**

**How to detect it:**
1. Find the report's `FormsId` in `PassageDetails`.
2. Query `TcV2AtSeaWeatherDetails` where `FormsId = <value>`. If zero rows → Gate 11 fired.
3. Check whether the raw weather source table (before the job ran) had data for that date.
4. Check whether `Hours > 2` filter removed the slots by looking at the raw observation timestamps.

**Bug or by-design:** This is a **data gap in the Analysed Weather path**. There is no fallback and no flag to indicate "weather data was missing for this report". The report silently passes as good weather. For the Reported Weather path this gate does not matter (reported weather doesn't use `WeatherData` records).

---

## 13. Gate 12 — All Weather Slots For This Report Are "Fine" (Conditions = None)

**File:** `WeatherExclusion.cs:248`

```csharp
if (weatherData != null && weatherData.Count > 0
    && weatherData.Any(w => w.Conditions != ExclusionConditions.None))
```

**What causes it:** Weather slots exist for this `FormsId` but every single slot was analysed as having `Conditions = None`. This means:
- Every hour: `AnalyzedWind <= windThreshold` AND `AnalyzedWave <= waveThreshold` AND current was within bounds.

**Result:** The outer `if` is false (`.Any()` returns false). No exclusion applied.

**This is correct behaviour** — weather was genuinely fine. This gate is not a bug.

**Important distinction from Gate 11:** Gate 11 = no data at all. Gate 12 = data exists but all slots are fine. Both produce the same result (no exclusion) but for different reasons. You can distinguish them by checking if `WeatherDetails` rows exist for this `FormsId`.

---

## 14. Gate 13 — Silent Exception Swallows The Entire Weather Calculation

**File:** `Passages.cs:272-293`

```csharp
if (exclusionCriteria != null)
{
    try
    {
        weatherData = dataFromDB.Weathers?.Where(...).ToList() ?? new List<WeatherData>();
        PassageData data = dataFromDB.Passages != null && dataFromDB.Passages.Any(w => w.FormsId == passage.FormsId)
            ? dataFromDB.Passages.First(w => w.FormsId == passage.FormsId)
            : passage;

        if (data != null)
        {
            List<ExclusionConditions> weatherConditions =
                _weatherExclusion.CalculateWeatherDetails(exclusionCriteria, weatherData, data, ref passageDetail);
            conditions.AddRange(weatherConditions);
        }
    }
    catch (Exception ex)
    {
        // Log exception but continue processing other exclusion conditions
        System.Diagnostics.Debug.WriteLine($"Weather exclusion error for FormsId {passage.FormsId}: {ex.Message}");
    }
}
```

**What causes it:** Any unhandled exception inside the try block. Examples:
- A null reference inside `CalculateWeatherDetails` (e.g. a null property on `reportData` that isn't guarded)
- An arithmetic exception (divide by zero in conversion functions)
- Any exception in the `ConvertKnotsToBeaufortNumber` or similar extension methods

**Result:** The exception is caught. It is written to `System.Diagnostics.Debug.WriteLine` — this only appears in a debugger session. **It is NOT logged to Serilog (the production logger).** The catch block has no `_logger.Error(...)` call. In production, this exception is completely invisible.

Processing continues. The report gets no weather condition codes. No indication in the output data that weather exclusion failed.

**How to detect it:** There is no way to detect this in production from the output data alone. The only way is to attach a debugger or add Serilog logging to the catch block.

**This is a bug.** The catch block should at minimum log to Serilog:
```csharp
catch (Exception ex)
{
    _logger.Warning("Weather exclusion error for FormsId {FormsId}: {Message}", passage.FormsId, ex.Message);
}
```

---

## 15. Gate 14 — WeatherSource Is Neither ReportedWeather Nor AnalysedWeather

**File:** `WeatherExclusion.cs:116-123`

```csharp
if (terms.WeatherSource == WeatherSourceForExclusion.ReportedWeather)
{
    IdentifyShipWeather(...);
}
else if (terms.WeatherSource == WeatherSourceForExclusion.AnalysedWeather)
{
    IdentifyAnalyzedWeather(...);
}
// No else block — if neither matches, nothing runs
```

**What causes it:** The `WeatherSource` enum has two defined values:
- `ReportedWeather = 1`
- `AnalysedWeather = 2`

If the database stores `0`, `null`, or any other unmapped integer for `WeatherSource`, the C# enum variable holds value `0` (the default). Neither `if` branch is entered.

This can happen if:
- The field was not configured in the charter terms setup.
- The DB value is null (maps to integer 0 in C# enum).
- A new source type was added to the DB but not yet to the enum.

**Result:** All three flags remain `false`. No weather exclusion for any report in the entire vessel run.

**How to detect it:** Check the `ExclusionCriteria` record for this `TermsId` in the terms configuration. Verify `WeatherSource` is set to a valid value (1 or 2).

**Bug or by-design:** A **configuration gap**. There is no validation or warning when `WeatherSource` is unrecognised.

---

## 16. Gate 15 — BadDayHoursType Is Unknown

**File:** `WeatherExclusion.cs:186`

```csharp
private bool IsBadHoursViolated(BadDayHoursType badDayHoursType, ...)
{
    switch (badDayHoursType)
    {
        case BadDayHoursType.Unknown: return false;  // ← GATE
        case BadDayHoursType.AS_Ratio_Of_24_Hours: ...
        case BadDayHoursType.Hours: ...
    }
    return false;
}
```

The `BadDayHoursType` enum:
- `Unknown = 0`
- `AS_Ratio_Of_24_Hours = 1`
- `Hours = 2`

**What causes it:** Same as Gate 14 — if the DB value is null or 0, the C# enum defaults to `Unknown = 0`.

**Result:** `IsBadHoursViolated` always returns `false`. Even if wind/wave clearly exceeded thresholds, the bad day check is never satisfied → `isWindExcluded = false`, `isWaveExcluded = false` → no weather exclusion.

This affects both Reported and Analysed weather paths. On the Reported path: `isBadDayViolated = false` so `isWindExcluded = false` regardless of `ReportedWindForce`. On the Analysed path: same, the `isBadDayViolated` inside `IdentifyAnalyzedWeather` returns false.

**How to detect it:** Check `ExclusionCriteria.BadDayHoursType` for this `TermsId`. If it's 0 or null → Gate 15 is permanently active for this vessel.

**Bug or by-design:** A **configuration gap**. No validation or warning is raised.

---

## 17. Gate 16 — BadDayHours Is Null In Terms (Defaults to 100)

**File:** `WeatherExclusion.cs:27`

```csharp
_badDayHours = terms.BadDayHours == null ? 100M : terms.BadDayHours.Value;
```

**What causes it:** `terms.BadDayHours` (i.e. `ExclusionCriteria.BadDayHours`) is null in the charter terms.

**Result:** `_badDayHours` is set to `100`. This means:
- For `BadDayHoursType.Hours`: `eventHours >= 100` → a single reporting period would need to be 100 hours long to violate the bad day threshold. Impossible. **No exclusion ever.**
- For `BadDayHoursType.AS_Ratio_Of_24_Hours`: `termsRatio = 100/24 = 4.17`. The event ratio (`badHours / steamingHours`) can never exceed 4.17 because bad hours can never be more than steaming hours (ratio max = 1.0). **No exclusion ever.**

**How to detect it:** Query `ExclusionCriteria.BadDayHours` for this `TermsId`. If null → this gate is active.

**Bug or by-design:** Ambiguous. If the charter party genuinely has no bad day hours rule, null is a valid state and the 9999/100 defaults effectively disable the feature. But there is no documentation or flag indicating "this was deliberately disabled vs. accidentally null." Consider treating null BadDayHours as "no bad day check required" (skip the test entirely and always exclude if wind/wave exceeds threshold) rather than using a large magic number.

---

## 18. Gate 17 — Bad Hours Do Not Exceed The Threshold

**File:** `WeatherExclusion.cs:191-197`

```csharp
case BadDayHoursType.AS_Ratio_Of_24_Hours:
    decimal badDayHoursEventRatio = steamingHours != 0
        ? (eventHours / steamingHours) : 0;
    if (badDayHoursEventRatio > badDayHoursTermsRatio) return true;
    return false;

case BadDayHoursType.Hours:
    return eventHours >= termsBadDayHours;
```

**What causes it:** The actual bad hours (or ratio) is below the threshold from the charter terms. This is normal operating behaviour — bad weather occurred but not enough to reach the exclusion threshold.

**Result:** `isBadDayViolated = false`. No weather exclusion for this report. The bad weather hours are not stored anywhere.

**This is correct behaviour by design.** The charter party requires a minimum bad weather duration before excluding a report. A brief squall that passes in 1 hour out of a 24-hour noon-to-noon period would not be excluded if the threshold is 12 hours.

**Verification question:** Is the threshold configured correctly? For some charters, any bad weather should trigger exclusion — in that case `BadDayHours` should be set to 0 (or a very small number) in the terms, not left at a high value.

---

## 19. Gate 18 — SteamingHours Is Zero (Ratio Calculation Produces Zero)

**File:** `WeatherExclusion.cs:189`

```csharp
decimal badDayHoursEventRatio = steamingHours != 0 ? (eventHours / steamingHours) : 0;
```

**What causes it (Reported Weather path — `IdentifyShipWeather`):**
```csharp
bool isBadDayViolated = IsBadHoursViolated(
    terms.BadDayHoursType,
    passageForm.Hours ?? 0,   // ← eventHours
    passageForm.Hours ?? 0,   // ← steamingHours (same value)
    _badDayHours);
```
If `passageForm.Hours` is null or 0, both `eventHours` and `steamingHours` are 0. The guard `steamingHours != 0` is false → ratio = 0 → not violated.

**What causes it (Analysed Weather path — `IdentifyAnalyzedWeather`):**
```csharp
bool isBadDayViolated = IsBadHoursViolated(
    terms.BadDayHoursType,
    badTime,                  // ← eventHours (bad slots sum)
    passageForm.Hours ?? 0,   // ← steamingHours
    _badDayHours);
```
If `passageForm.Hours` is null or 0, steamingHours = 0 → same result.

`passageForm.Hours` can be zero/null if:
- Event exclusions reduced hours to zero or below (and the code sets a negative hours value which is compared as `Hours > 0` later for SD, but the bad day check uses `??0` which would give 0 for null).
- The report was filed with identical timestamps as the previous one.

**Result:** `isBadDayViolated = false` → no weather exclusion.

**How to detect it:** Check `Hours` on the `PassageDetail` row. A zero or null `Hours` means this gate fired and weather exclusion was never evaluated regardless of conditions.

---

## 20. Gate 19 — WindThreshold Is Null In Terms (Defaults to 9999)

**File:** `WeatherExclusion.cs:26`

```csharp
_windThreshold = terms.WindSpeed == null ? 9999 : terms.WindSpeed.Value;
```

**What causes it:** `ExclusionCriteria.WindSpeed` is null in the charter terms.

**Result:** `_windThreshold = 9999`. No real-world wind speed can exceed 9999 BF or 9999 knots. **Wind exclusion is permanently disabled for this vessel.**

This also affects `ProcessWeatherData` — every hourly slot will have `isWindExcluded = false` regardless of actual wind. Slots will only get `WA` or `C` conditions at most, never `WI`, `WW`, `WIC`, or `WWC`.

**How to detect it:** No `WI`, `WW`, `WIC`, or `WWC` in any `Conditions` values for this vessel's records, even during obviously stormy periods. Check `ExclusionCriteria.WindSpeed` in the charter terms.

---

## 21. Gate 20 — WaveHeight Is Null In Terms (Defaults to 9999)

**File:** `WeatherExclusion.cs:28`

```csharp
_waveHeight = terms.WaveHeight == null ? 9999 : terms.WaveHeight.Value;
```

Same logic as Gate 19 but for wave height. `_waveHeight = 9999` → wave exclusion permanently disabled. No `WA`, `WW`, `WAC`, or `WWC` conditions ever.

---

## 22. Gate 21 — ReportedWindForce Is Null (Reported Weather Path)

**File:** `WeatherExclusion.cs:219`

```csharp
isWindExcluded = passageForm.ReportedWindForce != null   // ← GATE
                    && _windThreshold > 0
                    && passageForm.ReportedWindForce.Value > _windThreshold
                    && isBadDayViolated;
```

**What causes it:** The ship's officer did not record a wind force on this report. `ReportedWindSpeedFromDB` was null or empty in the source form → `PassageData.ReportedWind = null`.

**Result:** `isWindExcluded = false`. Even if the sea conditions were severe, if the officer left the wind field blank, no wind exclusion is applied under the Reported Weather path.

**How to detect it:** `ReportedWindForce` is null on the `PassageDetail` row. Check the source voyage form for this `FormsId`.

**Bug or by-design:** By design for the Reported path — if the officer didn't report it, the system can't evaluate it.

---

## 23. Gate 22 — ReportedWave Is Null (Reported Weather Path)

**File:** `WeatherExclusion.cs:223`

```csharp
isWaveExcluded = passageForm.ReportedWave != null   // ← GATE
                    && _waveHeight > 0
                    && passageForm.ReportedWave.Value > _waveHeight
                    && isBadDayViolated;
```

Same as Gate 21 but for wave. If `ReportedWave` is null → no wave exclusion.

---

## 24. Gate 23 — No Current Limits Defined In Terms

**File:** `WeatherExclusion.cs:29` + `WeatherExclusion.cs:229`

```csharp
// Constructor
_hasCurrentExclusionFromTerms = (terms.MinimumAdvCurrentLimit != null)
                               || (terms.MaximumAdvCurrentLimit != null);

// IdentifyShipWeather
isCurrentExcluded = _hasCurrentExclusionFromTerms && ...;
```

**What causes it:** Neither `MinimumAdvCurrentLimit` nor `MaximumAdvCurrentLimit` is set in the charter terms.

**Result:** `_hasCurrentExclusionFromTerms = false`. Current exclusion is permanently disabled regardless of actual current values. `isCurrentExcluded` is always false.

**By design:** Current exclusion is optional in charter parties. Many charters do not include a current clause.

---

## 25. Gate 24 — Wind Exactly At Threshold (Not Strictly Greater)

**File:** `WeatherExclusion.cs:90` (ProcessWeatherData) and `WeatherExclusion.cs:220` (IdentifyShipWeather)

```csharp
// ProcessWeatherData
bool isWindExcluded = weatherData[i].AnalyzedWind > _windThreshold;  // strict >

// IdentifyShipWeather
isWindExcluded = passageForm.ReportedWindForce.Value > _windThreshold;  // strict >
```

**What causes it:** Wind is exactly equal to the threshold. E.g. threshold = 7 BF, recorded wind = 7 BF.

**Result:** `7 > 7 = false` → not excluded. A wind reading of exactly the threshold value is treated as acceptable.

**By design:** "Exceeds the threshold" is typically how charter party clauses are worded. Equal to threshold is not a violation.

---

## 26. Gate 25 — AnalyzedWind/Wave Is Null In Weather Slot

**File:** `WeatherExclusion.cs:90-91`

```csharp
bool isWindExcluded = weatherData[i].AnalyzedWind > _windThreshold;
bool isWaveExcluded = weatherData[i].AnalyzedWave > _waveHeight;
```

**What causes it:** The weather provider delivered a slot record but left the wind or wave value null (or zero). `AnalyzedWind` is a nullable decimal. `null > threshold` evaluates to `false` in C#.

**Result:** The slot is treated as fine weather (`Conditions = None`). It contributes to `goodTime`, not `badTime`, when summing hours per report.

**How to detect it:** Check `WeatherDetails` rows with `Conditions = null/empty` but also `WindSpeedInBF = null`. These are slots where the data was missing, not where weather was genuinely fine.

**Bug or by-design:** This is a **data quality issue from the provider**. The code has no special handling for null weather values — they are treated as "fine weather." This could cause a report with genuinely missing data to appear as good weather rather than being flagged for investigation.

---

## 27. Summary Table: All Gates at a Glance

| Gate | Where | Cause | Reported Path Affected | Analysed Path Affected | Silent? | Type |
|---|---|---|---|---|---|---|
| 1 | Worker.cs:57 | No sea performance details in terms | Yes | Yes | No (vessel skipped) | By design |
| 2 | Worker.cs:63 | No passage data from DB | Yes | Yes | No (vessel skipped) | By design |
| 3 | Worker.cs + Passages ctor | ExclusionCriteria null | Yes | Yes | Partial (crash logged) | Bug |
| 4 | Worker.cs:76 | Operator dates filter empties passages | Yes | Yes | No (vessel skipped) | By design |
| 5 | Worker.cs:66 | TransformData not called (unit mismatch) | Yes | Yes | Yes — silent wrong comparisons | Bug |
| 6 | Passages.cs:59 | No WeatherData in DB for whole vessel | No | **Yes — all reports unexcluded** | Yes — no warning | Data gap |
| 7 | WeatherExclusion.cs:85 | Only 1 weather slot exists | No | Yes — slot removed, empty list | Yes | Data gap |
| 8 | Passages.cs:65 | Hours filter removes all slots | No | Yes — specific reports unexcluded | Yes — no logging | Data gap / config |
| 9 | Passages.cs:188 | Report is a Departure report | Yes | Yes | By design | By design |
| 10 | Passages.cs:270 | ExclusionCriteria null at report level | Yes | Yes | Yes | Bug (see Gate 3) |
| 11 | Passages.cs:274 + WE:248 | **No WeatherData for this FormsId** | No | **Yes — this report unexcluded** | **Yes — completely silent** | **Data gap** |
| 12 | WeatherExclusion.cs:248 | All slots fine (Conditions=None) | No | Yes (correct) | N/A | By design |
| 13 | Passages.cs:289 | Exception in weather block | Yes | Yes | **Yes — Debug.WriteLine only** | **Bug** |
| 14 | WeatherExclusion.cs:116 | WeatherSource = 0/unknown | Yes | Yes | Yes | Config gap |
| 15 | WeatherExclusion.cs:186 | BadDayHoursType = Unknown (0) | Yes | Yes | Yes | Config gap |
| 16 | WeatherExclusion.cs:27 | BadDayHours null → 100 hours | Yes | Yes | Yes | Config gap |
| 17 | WeatherExclusion.cs:191 | Bad hours below threshold | Yes | Yes | N/A (correct) | By design |
| 18 | WeatherExclusion.cs:189 | SteamingHours = 0 | Yes | Yes | Yes | Data issue |
| 19 | WeatherExclusion.cs:26 | WindSpeed null → 9999 | Yes | Yes | Yes | Config gap |
| 20 | WeatherExclusion.cs:28 | WaveHeight null → 9999 | Yes | Yes | Yes | Config gap |
| 21 | WeatherExclusion.cs:219 | ReportedWindForce null | Yes | No | Yes | Data issue |
| 22 | WeatherExclusion.cs:223 | ReportedWave null | Yes | No | Yes | Data issue |
| 23 | WeatherExclusion.cs:29 | No current limits in terms | Yes | Yes | N/A (correct) | By design |
| 24 | WeatherExclusion.cs:90 | Wind = threshold (not >) | Yes | Yes | N/A (correct) | By design |
| 25 | WeatherExclusion.cs:90 | AnalyzedWind/Wave null in slot | No | Yes | Yes | Data quality |

---

*Document generated from source code analysis of `gp-charterparty-jobs` TC2.0 AtSea worker.*
*Files: `Worker.cs`, `Passages.cs`, `WeatherExclusion.cs`, `Enums.cs`, `Terms.cs`*
