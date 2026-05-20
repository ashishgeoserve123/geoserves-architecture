# TCV2 AtSea — From Computed Data to Final Report: Full Story

> This document explains the complete journey of how computed charter party performance data is stored,
> what every column actually means, how exclusion conditions are determined and what they tell you,
> and — critically — where the data can be misleading or misread if you look at a single column
> without understanding how it was produced.
>
> A specific focus is placed on: **why a report can show `WI` (high wind exclusion) while the
> analyzed wind value stored is BELOW the threshold**, and **whether the bad hours value is the
> correct thing to look at to judge an exclusion**.

---

## Table of Contents

1. [The Five Output Tables](#1-the-five-output-tables)
2. [How to Read: PassageDetails](#2-how-to-read-passagedetails)
3. [The Conditions Column — What Each Code Means and How It Was Set](#3-the-conditions-column--what-each-code-means-and-how-it-was-set)
4. [The Wind Problem: WI Shown But Analyzed Wind Is Below Threshold](#4-the-wind-problem-wi-shown-but-analyzed-wind-is-below-threshold)
5. [The Bad Hours Problem: What Is Stored vs. What Drives the Decision](#5-the-bad-hours-problem-what-is-stored-vs-what-drives-the-decision)
6. [How to Read: ConsumptionDetails](#6-how-to-read-consumptiondetails)
7. [How to Read: PassageSpeedPerformance](#7-how-to-read-passagespeedperformance)
8. [How to Read: PassageFuelPerformance](#8-how-to-read-passagefuelperformance)
9. [How to Read: AnalyzedWeatherDetails](#9-how-to-read-analyzedweatherdetails)
10. [How to Read: ExcludedReports](#10-how-to-read-excludedreports)
11. [Column-by-Column Due Diligence: What the Value Really Represents](#11-column-by-column-due-diligence-what-the-value-really-represents)
12. [Summary of Known Interpretation Traps](#12-summary-of-known-interpretation-traps)

---

## 1. The Five Output Tables

The charterparty jobs compute and write to five sets of tables. All of these are written by the stored procedure `dbo.Job_InsertTCV2AtSeaPassageData` plus a separate one for weather.

| Output Table | Rows Written | What It Represents |
|---|---|---|
| `TcV2AtSeaPassageDetails` | One per report per passage | Individual noon/arrival reports enriched with performance fields and exclusion flags |
| `TcV2AtSeaConsumptionDetails` | One per report × fuel group | Fuel burned in that reporting period split by fuel type and engine type |
| `TcV2AtSeaPassageSpeedPerformance` | One per distinct ordered speed per passage | Aggregated speed performance for the full passage: was the vessel fast enough? |
| `TcV2AtSeaPassageFuelPerformance` | One per speed × fuel group per passage | Aggregated fuel performance: did the vessel consume too much? |
| `TcV2AtSeaWeatherDetails` | One per hourly weather slot | Raw hourly analysed weather data with tagged exclusion conditions |
| `TcV2AtSeaExcludedReports` | One per manually excluded report per fuel group | Snapshot of excluded report details for reporting |

Before any of these are written, the job first calls `dbo.Job_DeleteTCV2AtSea` (hard delete) for the vessel + term ID combination. Every run is a **full recalculation and replacement** — there is no incremental update.

---

## 2. How to Read: PassageDetails

`TcV2AtSeaPassageDetails` is the **base layer** of the report. Every other table joins back to this one via `PassageID` and `FormsId`.

### What One Row Represents

One row = one voyage report within a passage. A passage is the set of reports from the moment a vessel departs one port to the moment it arrives at the next. The **departure report** starts the passage but has no performance data (there is no previous point to measure from). Every **noon report** and the **arrival report** carry the computed performance data.

### How a Passage Is Grouped

All reports in the same departure→arrival leg share the same `PassageID` (a GUID). This is the key to joining all related records. If you want to see all reports for one voyage leg, filter by `PassageID`.

### Reading the Report Date

`FormDateInUTC` is always UTC. The source system stores dates with a timezone offset, and the job converts them. If you are comparing this to local time in the voyage report, apply the vessel's timezone offset.

### Reading Hours

`Hours` tells you how many hours elapsed between this report and the previous report in the same passage. This is **after event deductions** — if the vessel had a manoeuvring event during that period, those hours have already been subtracted. So `Hours` in the table may be smaller than the raw gap between two report timestamps.

If you need the raw period length, compute it yourself from `FormDateInUTC` of consecutive rows. The table does not store the raw gap separately.

### Reading Distance

`Distance` is the observed nautical miles as logged by the ship for that period. Like `Hours`, this has already had event distances subtracted. A manoeuvring event, for example, would have its distance removed from this field. The raw reported distance from the source form is not stored.

### Reading Wind and Wave

There are **four weather-related columns** on this table and they are NOT all the same thing:

| Column | What It Is | Who Provided It |
|---|---|---|
| `ReportedWindForce` | The wind force the ship's officer recorded on the form | Ship crew |
| `ReportedWave` | The wave height the ship's officer recorded | Ship crew |
| `EventAnalyzedWindForce` | The wind force from the external weather analysis provider, at the report level | External weather provider |
| `EventAnalyzedWave` | The wave height from the external weather analysis provider, at the report level | External weather provider |

These two sources can — and frequently do — disagree. This is the root cause of the most common confusion in the report: **you can have a `WI` exclusion while `EventAnalyzedWindForce` is below the threshold.** This is explained in full in Section 4.

All four values are in the **charter party's agreed unit** (Beaufort for wind if terms say BF, Douglas Sea Scale for wave if terms say DS, etc.) — conversion has already been applied before the value is stored.

---

## 3. The Conditions Column — What Each Code Means and How It Was Set

The `Conditions` column is a comma-separated string. It can contain multiple codes, e.g. `"WI,SD"` or `"WW,E1"`.

**A report WITH a condition code is excluded from charter party performance calculations.**
**A report with NO condition code is counted as a good weather, valid performance day.**

Here is every code, what it means, and what inputs drove it:

---

### `WI` — Wind Exclusion

**Meaning:** The wind was bad enough during this reporting period to exclude it from performance calculations.

**How it was set (Reported Weather path):**
```
ReportedWindForce > windThreshold
AND badDayHoursRule was violated
```
The decision was made using the **ship's reported wind** (`ReportedWindForce`), not the analyzed wind.

**How it was set (Analysed Weather path):**
```
Hourly weather slots for this report period were analysed
→ badTime (hours of bad hourly slots) was compared against steamingHours
→ badDayHoursRule was violated
→ at least one bad hourly slot had a wind-related condition (WI, WIC, WWC, or WW)
```
The decision was made from the **hourly analysed weather data** (WeatherData table), not the single-value `EventAnalyzedWindForce`.

**What to look at in the report to verify it:** See Section 4.

---

### `WA` — Wave Exclusion

Same logic as `WI` but for wave height. In Reported Weather path: `ReportedWave > waveThreshold AND badDayViolated`. In Analysed path: bad hourly slots with wave conditions.

---

### `C` — Current Exclusion

**Meaning:** Adverse current was outside the allowed range defined in the charter terms.

**How it was set:**
```
EffectiveCurrent < MinimumAdvCurrentLimit
OR
EffectiveCurrent > MaximumAdvCurrentLimit
```
Note: Current exclusion does **not** go through the bad day hours test. If the current is outside range → the report is excluded regardless of how long it lasted.

**What to look at:** `EffectiveCurrent` column vs. the terms' current limits. The terms limits are NOT stored in the PassageDetails table — they come from `ExclusionCriteria` in the terms configuration.

---

### `WW` — Wind AND Wave Exclusion

Both wind and wave exceeded their thresholds in the same reporting period.

---

### `WIC` — Wind AND Current

Both wind and current were outside limits.

---

### `WAC` — Wave AND Current

Both wave and current were outside limits.

---

### `WWC` — Wind, Wave, AND Current

All three exceeded their respective limits.

---

### `SD` — Short Day

**Meaning:** The reporting period was too short to be statistically meaningful.

**How it was set:**
```
Hours (after event deductions) < MinimumHRsInADay (from terms)
AND Hours > 0
```
This is checked **after** event exclusions have reduced the hours. So if a noon report originally covers 24 hours but a 20-hour event is deducted, the remaining 4 hours triggers SD.

**Important:** The threshold `MinimumHRsInADay` is not stored in this table — it lives in the charter terms. You cannot verify the SD flag by looking at `Hours` alone unless you also know the terms minimum.

---

### `SP` — Short Passage

**Meaning:** The entire voyage from departure to arrival was too short to count.

**How it was set:**
```
(ArrivalDate - DepartureDate).TotalHours < MinimumPassageHours (from terms)
```
This is checked once per passage and applied to **every non-departure report** in that passage. So you may see SP on every noon report in a passage — this does not mean each individual period was short; it means the whole voyage was short.

**To verify:** Look at the gap between `DepartureDate` and `ArrivalDate` in the `PassageSpeedPerformance` table and compare against the terms.

---

### `EXC` — Manually Excluded

**Meaning:** Someone in the source voyage reporting system marked this report as excluded. The job honours that flag directly.

**How it was set:** `PassageData.IsExcluded = true` in the source report data.

**Note:** This exclusion is recorded but the report data is still stored in `PassageDetails`. The excluded report's fuel summary is written to `TcV2AtSeaExcludedReports`.

---

### `UW` — Unwarranted Speed

**Meaning:** The vessel was ordered to sail at a speed for which there is no warranty in the charter party terms. There is no benchmark to compare against, so the report cannot be evaluated.

**How it was set:**
```
No SeaPerformanceDetails row exists where:
    Speed == OrderedSpeed
    AND LoadCondition == passage.LoadCondition
```

**Note:** A report can have both weather conditions AND `UW` at the same time if the vessel happened to sail at an unwarranted speed during bad weather.

---

### `FCO` — Fuel Change Over

**Meaning:** The vessel was switching from one fuel type to another during this period. This affects consumption accuracy and is excluded.

**Note:** FCO data is fetched from the database but is not currently actively applied in the AtSea calculation path (the FCO condition code appears in the `BadWeatherConditions` list used to filter good weather reports, meaning FCO-tagged reports are excluded from performance summaries).

---

### Event Codes (e.g. `E1`, `DEVIATION`, `MANOEUVRE`)

**Meaning:** A specific operator-defined event occurred during this reporting period. The event type and code are configured in the source system.

**How it was set:** An `ExclusionEventDetail` record existed for this `FormsId`, and either:
- The event had a non-zero duration (hours were deducted), OR
- The event had a negative distance (distance was deducted), OR
- The event had associated fuel consumption that was deducted.

The code itself (e.g. "E1") comes from `ExclusionEventDetail.ConditionCode` — it is free text configured per charter party, not a fixed system code.

---

## 4. The Wind Problem: WI Shown But Analyzed Wind Is Below Threshold

This is the most common source of confusion in the report. Here is the complete explanation.

### The Scenario

A report shows `WI` in the `Conditions` column.  
You look at `EventAnalyzedWindForce` and it is **4 BF**.  
The charter terms wind threshold is **7 BF**.  
**4 < 7, so how can wind be excluded?**

### Why This Happens — Two Different Cases

---

#### Case 1: Weather Source = Reported Weather

When the charter terms say `WeatherSource = ReportedWeather`, the **exclusion decision is made using the ship crew's reported wind**, stored in `ReportedWindForce`.

```
Decision source:   ReportedWindForce   (ship crew)
Stored display:    EventAnalyzedWindForce   (external provider)
```

These are **completely independent values**. The ship may have reported 9 BF (above threshold, exclusion triggered). The external weather provider may have analysed the same time period and produced 4 BF (below threshold). Both values are stored but only one was used for the exclusion decision.

When you look at the report and see `EventAnalyzedWindForce = 4 BF` with condition `WI`, you are seeing the **external provider's assessment** — but the exclusion was **NOT based on that value**. It was based on `ReportedWindForce`, which you should check to verify the exclusion.

**The correct column to look at to verify a `WI` exclusion when WeatherSource = Reported:**
→ `ReportedWindForce` — compare this against the charter wind threshold.

**`EventAnalyzedWindForce` in this path is stored for informational purposes only.** It tells you what the external provider said, which can be used to see if the ship's crew over-reported weather conditions. But it does not determine and does not reflect the exclusion decision.

---

#### Case 2: Weather Source = Analysed Weather

When `WeatherSource = AnalysedWeather`, the exclusion decision uses the **hourly weather slots** from `TcV2AtSeaWeatherDetails`, NOT the single `EventAnalyzedWindForce` value.

Here is the sequence:
1. `TcV2AtSeaWeatherDetails` has many rows for this `FormsId`, each representing one hourly slot.
2. The job sums up hours from slots that were tagged as bad (had wind/wave/current over threshold).
3. If bad hours exceeded the bad day threshold → exclusion is triggered.
4. The `EventAnalyzedWindForce` stored on `PassageDetails` is the **report-level summary value** from the weather provider — one value for the whole reporting period. This is a different value from the hourly slot data.

After determining the exclusion from hourly data, the code tries to make `EventAnalyzedWindForce` consistent:

```
If wind IS excluded BUT EventAnalyzedWindForce <= windThreshold:
    → Force: EventAnalyzedWindForce = windThreshold + 1

If wind is NOT excluded BUT EventAnalyzedWindForce > windThreshold:
    → Force: EventAnalyzedWindForce = windThreshold
```

This adjustment ensures `EventAnalyzedWindForce` signals "above threshold" when exclusion happened. **However, this only applies to the Analysed Weather path.** There is no equivalent adjustment in the Reported Weather path.

**So in Analysed Weather path:** `EventAnalyzedWindForce` should always be consistent with the exclusion decision — if you see `WI`, `EventAnalyzedWindForce` should be above threshold (either naturally or because it was forced to `threshold + 1`).

**In Reported Weather path:** `EventAnalyzedWindForce` has no such guarantee. It can be below threshold with `WI` present, because the exclusion was based on `ReportedWindForce`.

---

### Summary Table: Which Column Drove the Exclusion

| WeatherSource | WI Decision Based On | Column That Shows The Decision Value | Column That May NOT Reflect Decision |
|---|---|---|---|
| `ReportedWeather` | `ReportedWindForce` | `ReportedWindForce` | `EventAnalyzedWindForce` (external, independent) |
| `AnalysedWeather` | Hourly slots in `WeatherDetails` | `EventAnalyzedWindForce` (adjusted to be consistent) | `ReportedWindForce` (ship crew, not used) |

---

### The Right Way to Verify a WI Exclusion in a Report

**Step 1:** Find the `WeatherSource` for this vessel's charter terms.

**Step 2 if ReportedWeather:**
- Look at `ReportedWindForce` on the `PassageDetail` row.
- If `ReportedWindForce > windThreshold` → that is why `WI` was set.
- `EventAnalyzedWindForce` tells you the weather provider's view but did NOT drive the exclusion.
- Also check the bad day hours condition (see Section 5).

**Step 3 if AnalysedWeather:**
- Look at `TcV2AtSeaWeatherDetails` rows where `FormsId` matches this report.
- Sum `Hours` of rows where `Conditions != empty` → this is the `badTime`.
- Check the bad day hours rule (see Section 5).
- `EventAnalyzedWindForce` on `PassageDetail` should be > threshold if `WI` is set (it was forced).

---

## 5. The Bad Hours Problem: What Is Stored vs. What Drives the Decision

The "bad day hours" rule exists to prevent a brief moment of bad weather from excluding an entire 24-hour reporting period. The charter party defines a minimum threshold of bad weather that must occur before the report is excluded.

### What Is Stored in the Tables

**The `PassageDetails` table does NOT store:**
- The total bad hours for the period
- The good hours for the period
- The bad hours threshold from the terms
- The bad hours percentage/ratio

**Only the result is stored:** the condition code (`WI`, `WA`, etc.) or no code at all.

**The `TcV2AtSeaWeatherDetails` table stores:**
- Individual hourly slots with their individual conditions
- `Hours` per slot

From these you CAN reconstruct the bad hours total. But this only works for the **Analysed Weather path**. For Reported Weather, there is no hourly breakdown — the bad day check uses the report's own `Hours` value.

---

### Two Types of Bad Hours Logic

The charter terms define `BadDayHoursType`. There are two variants:

---

#### Type 1: `Hours` (Fixed Threshold)

```
badTime >= termsBadDayHours   → bad day violated → exclusion applies
```

Example: `termsBadDayHours = 12`
- If the bad hours in the period are 12 or more → the report is excluded.
- For Reported Weather path: the entire reporting period is treated as either all-bad or all-good. `badTime = passageDetail.Hours`. There is no partial concept.
- For Analysed Weather path: `badTime` = sum of hours from bad hourly slots.

**Is this the right number to look at?**
For Reported Weather: `Hours` on `PassageDetail` is the period length. The question is whether that whole period's wind exceeded the threshold. If `ReportedWindForce > threshold` is true AND `Hours >= termsBadDayHours` → exclusion. You cannot see this intermediate test from the stored data alone.

For Analysed Weather: sum `Hours` from bad `WeatherDetails` rows for this `FormsId`. If that sum >= `termsBadDayHours` → that is why exclusion was triggered.

---

#### Type 2: `AS_Ratio_Of_24_Hours` (Proportional)

```
termsRatio = termsBadDayHours / 24
eventRatio = badTime / steamingHours
eventRatio > termsRatio   → bad day violated
```

Example: `termsBadDayHours = 12` with `BadDayHoursType = AS_Ratio_Of_24_Hours`
```
termsRatio = 12 / 24 = 0.5 (50%)
```
If more than 50% of the steaming period was bad → excluded.

This type is fairer for short periods. A 6-hour reporting period with 4 hours of bad weather (66%) would be excluded. A 24-hour period with 11 hours bad (45.8%) would NOT be excluded under a 50% rule.

**For Reported Weather path with this type:** `steamingHours = passageDetail.Hours` (the whole period), `badTime = passageDetail.Hours` (same value, since the whole period is treated as bad). So `eventRatio = 1.0 = 100%`. This means: if Reported Weather path uses `AS_Ratio_Of_24_Hours`, a report is excluded as long as the wind was above threshold for any part of the period — the ratio is always 100% because there is no sub-period data.

**This is a potential issue:** the `AS_Ratio_Of_24_Hours` type is only meaningful with hourly weather data (Analysed path). Applied to Reported Weather, it degenerates to "if wind was bad at all → exclude."

---

### What You Can and Cannot Determine From the Stored Data

| Question | Can You Answer From Stored Data? | What To Look At |
|---|---|---|
| Was WI triggered by reported or analysed wind? | No — need to know `WeatherSource` from terms | Charter terms `ExclusionCriteria.WeatherSource` |
| What wind value triggered WI? | Only if Reported Weather | `ReportedWindForce` (Reported path); hourly `WeatherDetails` (Analysed path) |
| How many bad hours caused the exclusion? | Only for Analysed Weather path | Sum `Hours` from `WeatherDetails` where `Conditions != empty` for this `FormsId` |
| What is the bad hours threshold? | No — not stored in output tables | Charter terms `ExclusionCriteria.BadDayHours` |
| Was the bad day ratio exceeded? | Not directly | Compute: `badHours / passageDetail.Hours > termsBadDayHours / 24` |
| Why is EventAnalyzedWindForce below threshold when WI is set? | Yes — if WeatherSource = Reported | Exclusion was based on `ReportedWindForce`, not `EventAnalyzedWindForce` |

---

## 6. How to Read: ConsumptionDetails

`TcV2AtSeaConsumptionDetails` has one row per **reporting period × fuel group**. It records how much fuel was consumed in that period.

### Key Points

**`MEConsumption`, `AUXConsumption`, `OtherConsumption`** — these are already **event-adjusted**. If an exclusion event (e.g. manoeuvring) occurred and it had associated fuel consumption, that fuel was **subtracted** before being stored here. The raw consumption from the source form is not stored.

**`FuelGroup`** — "FO", "GO", or "BF" (BioFuel). BioFuel records from the source are remapped: if the charter says bio-fuel counts as FO → it appears here under `FuelGroup = "FO"`. There is no indication in this table that some of the FO consumption originated from bio-fuel.

**`OrderedSpeed`** — copied from the report at the time it was processed. This is used as the join key to link consumption records to the correct speed warranty row in `PassageFuelPerformance`.

**Zero rows are not stored.** If ME, AUX, and Others consumption are all zero or null for a given fuel group on a given report, no row exists in this table. When joining to `PassageDetails`, a missing row means no fuel consumption was recorded for that fuel group in that period.

---

## 7. How to Read: PassageSpeedPerformance

`TcV2AtSeaPassageSpeedPerformance` has one row per **distinct ordered speed per passage**. This is the aggregated speed performance summary.

### Key Points

**`IsUnWarranted = true`** → all performance columns (`AllWeatherHours`, `GWeatherHours`, etc.) are zero. There is no benchmark to measure against. The row exists but carries no meaningful performance data.

**`IsShortPassage = true`** → same as above. No performance data calculated.

**`AllWeather*` vs. `GWeather*`** — "All Weather" includes every report at this speed (good and bad weather). "G Weather" (Good Weather) includes only reports with no exclusion condition codes. The charter party performance evaluation is primarily based on Good Weather figures.

**`AllWPerformedSpeed` and `GWPerformedSpeed`**:
```
PerformedSpeed = Distance / Hours
```
If the charter terms include current factor (`CurrentFactor = Include`), the average effective current is **subtracted** from the performed speed before storing. This means the stored speed represents the vessel's performance adjusted for current — the "through the water" effective performance rather than the raw "over the ground" speed.

**`IsInRange`** — whether the vessel performed within the charter's "about" band:
```
MinAllowedTime = GWeatherDistance / (OrderedSpeed + UpperSpeedLimit)
MaxAllowedTime = GWeatherDistance / (OrderedSpeed - LowerSpeedLimit)
IsInRange = GWeatherHours between MinAllowedTime and MaxAllowedTime
```
A vessel that arrived too quickly (`GWeatherHours < MinAllowedTime`) is IN range — it simply performed better than required. A vessel that was too slow (`GWeatherHours > MaxAllowedTime`) is OUT of range with positive `GWTimeGainLoss`.

**`TotalTimeLossGain`** — the final claim figure in hours:
- Without extrapolation: equals `GWTimeGainLoss` (only good weather performance counts).
- With extrapolation: `PerMileFactor × AllWeatherDistance` (the good weather rate is applied to the full voyage distance, including bad weather miles).
- Zero if `IsInRange = true`.

---

## 8. How to Read: PassageFuelPerformance

`TcV2AtSeaPassageFuelPerformance` has one row per **distinct ordered speed × fuel group per passage**.

### Key Points

**Warranted Consumption Calculation:**
```
WarrantedDaily = MEWarranty from charter terms (MT/day)
WarrantedForDistance = (GWDistance / OrderedSpeed) * (WarrantedDaily / 24)
```
The charter gives a daily consumption rate. To compare it to the actual voyage, the system converts it to "total consumption for the good weather distance at ordered speed." This is the fair comparison point — not a straight time comparison, because different passages cover different distances.

**Scrubber Allowance:** If the vessel has a scrubber fitted and the charter allows for it, `MEWarrantedConsumption` has already been adjusted upward by `ScrubberAllowance` for the FO fuel group. The raw warranty from the terms is NOT what is stored — the scrubber-adjusted value is.

**`GWMEConsumption`, `GWAUXConsumption`, `GWOthersConsumption`** — actual consumption during good weather periods only. These are already event-adjusted (events were deducted at the `ConsumptionDetails` level before being summed here).

**`IsInRage` (note: "Rage" is a typo in the code — means "Range")** — true only if ALL of ME, AUX, and Others are within their bounds simultaneously. One engine type out of range makes the whole row `IsInRage = false`.

**`FuelLossGain`** — the final claim figure in metric tonnes. Positive = vessel consumed more than warranted (owner's liability or charterer's claim depending on terms). Negative = vessel under-consumed (vessel owner gains).

**With extrapolation vs. without:**
- Without: `FuelLossGain = sum of good weather deviations per engine type`
- With: `FuelLossGain = (per-mile deviation rate from good weather) × (total voyage distance)`

---

## 9. How to Read: AnalyzedWeatherDetails

`TcV2AtSeaWeatherDetails` stores the hourly weather analysis. This is the most granular weather data available.

### Key Points

**One row ≠ one hour precisely.** The `Hours` field on each row is computed as:
```
Hours = (thisSlot.AnalyzedWeatherDate - previousSlot.AnalyzedWeatherDate).TotalHours
```
The first slot in any sequence has `Hours = 0`. Slots with `Hours <= 0` or `Hours > 2` were removed before saving.

**`FormsId`** links this weather slot to a specific voyage report. Multiple rows with the same `FormsId` cover the reporting period for that form. Summing their `Hours` gives the total covered time (approximately equal to the report's `Hours` field in `PassageDetails`).

**`Conditions`** on each row is the condition for that specific hour-long slot, independent of the per-report decision. A slot might be tagged `WI` but the report might not be excluded if not enough hours were bad overall.

**`WindSpeedInBF`, `WaveHeightInDSS`, `CurrentInKTS`** — these are in charter units (already converted from metres/knots). They are the external weather provider's values for that slot.

**How this table relates to the PassageDetail decision (Analysed Weather path):**
```
Bad slots:     WeatherDetails rows WHERE Conditions != empty AND FormsId = X
badTime:       SUM(bad slots.Hours)
steamingHours: PassageDetail.Hours WHERE FormsId = X

Decision: badTime/steamingHours > termsBadDayHours/24   (if ratio type)
       or badTime >= termsBadDayHours                   (if fixed hours type)
```

**This is the correct table to look at when verifying why a report was weather-excluded in the Analysed Weather path.** The `EventAnalyzedWindForce` on `PassageDetail` is a summary value — the actual decision came from this table.

---

## 10. How to Read: ExcludedReports

`TcV2AtSeaExcludedReports` is a snapshot of reports that were manually excluded (`IsExcluded = true` in the source system). It is separate from the main `PassageDetails` table.

| Column | What It Means |
|---|---|
| `IMONumber` | The vessel |
| `VoyageNumber` | The voyage the report belonged to |
| `DateTimeInUTC` | The report date/time |
| `Hours` | The period length (after event deductions) |
| `Distance` | The distance sailed (after event deductions) |
| `Remark` | Free-text remarks from the original report |
| `FormId` | The original form ID |
| `FuelGroup` | Which fuel group this consumption row covers |
| `Consumption` | Total fuel consumed in this period (ME + AUX + Others, summed) |

Note: One excluded report generates multiple rows if it has consumption in multiple fuel groups.

---

## 11. Column-by-Column Due Diligence: What the Value Really Represents

This section goes column by column across all tables and flags where the stored value may not mean what it looks like.

---

### PassageDetails

| Column | Apparent Meaning | What It Actually Is | Watch Out For |
|---|---|---|---|
| `Hours` | Period length | **Event-adjusted** hours. Already has exclusion events subtracted. | Cannot be used to reconstruct the raw period. Check consecutive `FormDateInUTC` for raw gap. |
| `Distance` | Distance sailed | **Event-adjusted** distance. Event distances subtracted. | Same as Hours — raw distance is not stored. |
| `OrderedSpeed` | Contracted speed | The `CPSpeed` field from the voyage form | If null in source → stored as 0. A row with `OrderedSpeed = 0` should be treated as having no ordered speed, not literally zero knots. |
| `ReportedWindForce` | Ship's reported wind | Already unit-converted to charter units (BF or kts). | Raw Beaufort or raw knots from the form is NOT what's stored. |
| `EventAnalyzedWindForce` | External analyzed wind | **Only meaningful for WI/WA exclusion verification in Analysed Weather path.** In Reported Weather path this is the provider's value regardless of what drove the exclusion. | **DO NOT use this column alone to verify WI exclusion.** Check `WeatherSource` from terms first. |
| `EventAnalyzedWave` | External analyzed wave | Same caveat as `EventAnalyzedWindForce`. | Same as above for WA. |
| `EffectiveCurrent` | Net current experienced | Used for current exclusion and optionally subtracted from performed speed. | If current factor is NOT included in terms, this field is stored but not used in speed calculation. |
| `Conditions` | Exclusion reasons | A comma-separated string. Empty = good weather valid record. | **Must be parsed as a string-contains check.** A record with `"WW"` does NOT also individually contain `"WI"` and `"WA"` — `WW` is its own combined code. |
| `ROB` | Fuel remaining on board | JSON string. Only covers active fuel types from charter terms. | Fuel types not in `Terms.ActiveFuelTypes` are silently omitted. If a fuel type shows no ROB, it may not be configured in terms, not necessarily zero on board. |
| `LoadCondition` | Laden or Ballast | Source value from voyage form | Null `LoadCondition` causes `UW` to NOT be checked (guard exists). If consistently null, no speed will ever be flagged UW — which may incorrectly omit the flag. |

---

### ConsumptionDetails

| Column | What It Really Is | Watch Out For |
|---|---|---|
| `MEConsumption` | Main engine fuel in this period | **Event-adjusted**: event fuel already subtracted. Value is the NET consumption after deductions. A low value may mean events happened, not that the engine ran clean. |
| `AUXConsumption` | Auxiliary engine fuel | Same event-adjustment applies. |
| `OtherConsumption` | Other consumption | Same. |
| `FuelGroup` | Which fuel type | BioFuel is remapped to FO or GO before storage. There is no way to tell from this table which records came from bio-fuel originally. |
| `OrderedSpeed` | Speed during this period | This is what links the consumption to a warranty row. If `OrderedSpeed = 0` or null, the consumption row exists but will not match any warranty and will be excluded from performance calculations. |

---

### PassageSpeedPerformance

| Column | What It Really Is | Watch Out For |
|---|---|---|
| `AllWPerformedSpeed` | Vessel speed over the full passage | Already current-adjusted if terms include current. Value may be lower than `AllWeatherDistance / AllWeatherHours` by the average current. |
| `GWPerformedSpeed` | Vessel speed in good weather | Same current adjustment. |
| `MinAllowedTime` | Fastest allowed time for the distance | Only meaningful when `IsUnWarranted = false` and `IsShortPassage = false`. Zero otherwise. |
| `MaxAllowedTime` | Slowest allowed time | Same caveat. |
| `IsInRange` | Was the vessel on time? | **False when passage is unwarranted or short** (all values are zero) — IsInRange = false does NOT mean a claim exists in these cases. Check `IsUnWarranted` and `IsShortPassage` first. |
| `TotalTimeLossGain` | Final claim figure in hours | Zero when `IsInRange = true` OR when unwarranted/short passage. A zero here does NOT distinguish between "performed fine", "unwarranted", and "short passage". |
| `PerMileFactor` | Time deviation per mile | Used for extrapolation. Only meaningful when `IsInRange = false` and `GWeatherDistance > 0`. |

---

### PassageFuelPerformance

| Column | What It Really Is | Watch Out For |
|---|---|---|
| `MEWarrantedConsumption` | Daily warranted ME consumption | **Scrubber-adjusted for FO group.** If vessel has a scrubber, this value is higher than the raw warranty in the charter terms for FO. |
| `MEMinWarrantedCons` | Lower bound for the passage | Computed as `(GWDistance / OrderedSpeed) × (MEWarrantedConsumption / 24) × minPercent`. **Based on GWDistance, not total voyage distance.** |
| `MEMaxWarrantedCons` | Upper bound for the passage | Same. If `GWDistance = 0` (all weather was bad), these will be zero and the consumption checks will be trivially out of range. |
| `GWMEConsumption` | Actual ME consumption in good weather | Event-adjusted. Sum of `MEConsumption` from `ConsumptionDetails` rows that correspond to good weather reports. |
| `IsInRage` | Was total fuel within bounds? | Note: all three engine types (ME, AUX, Others) must be in range for this to be true. One out-of-range engine makes the whole row `IsInRage = false`. |
| `FuelLossGain` | Final claim figure in MT | Zero when `IsInRage = true`. A zero does NOT mean the vessel performed perfectly — it means it was within the "about" tolerance band. |
| `GWMEFuelLossGain` | ME deviation from warranted | Positive = over-consumed. Negative = under-consumed. Zero if in range. |

---

### AnalyzedWeatherDetails

| Column | What It Really Is | Watch Out For |
|---|---|---|
| `Hours` | Duration of this slot | Computed from time gap to previous slot. First slot always has `Hours = 0`. |
| `WindSpeedInBF` | Analysed wind for this hour | In charter units (may be BF or kts depending on terms). NOT in raw knots. |
| `WaveHeightInDSS` | Analysed wave for this hour | In charter units (may be DSS, ft, or m). NOT in raw metres. |
| `Conditions` | Condition for this individual slot | A slot tagged `WI` does NOT mean the report was excluded — the bad day hours threshold may not have been met. You must aggregate across all slots for the same `FormsId`. |
| `FormDate` | The report this slot belongs to | The report date, not the slot's own observation time. Use `CalculatedDate` for the actual observation time. |

---

## 12. Summary of Known Interpretation Traps

These are the specific scenarios where a straightforward reading of a column gives the wrong conclusion:

---

### Trap 1: `WI` present but `EventAnalyzedWindForce` < threshold

**Why it happens:** `WeatherSource = ReportedWeather`. Exclusion was based on `ReportedWindForce`. `EventAnalyzedWindForce` is the external provider's value, stored independently.

**How to correctly verify:** Check `ReportedWindForce` against the charter threshold and verify bad day hours (Section 4).

---

### Trap 2: `EventAnalyzedWindForce` > threshold but NO `WI` condition

**Why it happens:** Could be Analysed Weather path where bad hours were not enough to violate the bad day rule. OR `WeatherSource = Reported` where `ReportedWindForce` was not above threshold even though the external provider reported high wind.

**How to correctly verify:** Check `WeatherSource`. If Analysed: sum bad hours from `WeatherDetails` and compare to terms threshold. If Reported: check `ReportedWindForce`.

---

### Trap 3: `Hours` looks short but no `SD` flag

**Why it happens:** The `MinimumHRsInADay` threshold may not be set in the charter terms (it is optional). If null, SD is never checked.

**How to correctly verify:** Confirm `MinimumHRsInADay` exists in the charter terms. Absence of SD with short hours = the charter has no short day rule, not a system error.

---

### Trap 4: `TotalTimeLossGain = 0` looks like perfect performance

**Why it happens:** Could also mean `IsUnWarranted = true` or `IsShortPassage = true`. In both those cases performance is not calculated and the field defaults to zero.

**How to correctly verify:** Always check `IsUnWarranted` and `IsShortPassage` before interpreting `TotalTimeLossGain`. Only trust the zero if both are false.

---

### Trap 5: `IsInRage = false` looks like a fuel claim

**Why it happens:** `IsInRage` is false whenever `IsUnWarranted` or `IsShortPassage` is true (because no bounds are computed). Also, `FuelLossGain = 0` even when `IsInRage = false` if the passage was unwarranted/short.

**How to correctly verify:** Check `IsUnWarranted` and `IsShortPassage` first. A fuel claim only exists when both are false AND `IsInRage` is false AND `FuelLossGain != 0`.

---

### Trap 6: A `WeatherDetails` slot has `WI` but the PassageDetail has no wind condition

**Why it happens:** Individual hourly slots are tagged independently. The report-level exclusion only fires if the cumulative bad hours from all slots (for that `FormsId`) exceeded the bad day hours threshold. A single bad hour in a 24-hour report may not be enough.

**How to correctly verify:** This is correct behaviour, not a bug. Sum all bad-conditioned slot hours for the `FormsId` and compare against the terms threshold.

---

### Trap 7: `MEWarrantedConsumption` is higher than the raw charter party terms value

**Why it happens:** Scrubber allowance has been added for FO fuel group. The value stored is `MEWarranty + ScrubberAllowance`.

**How to correctly verify:** If the vessel has a scrubber, subtract `ScrubberAllowance` from the stored value to get the base charter warranty. The scrubber allowance value is in the charter terms, not stored in this table.

---

### Trap 8: Bad hours percentage cannot be seen in the report

**Why it happens:** The bad hours intermediate calculation is not stored anywhere in the output tables. Only the final condition code result is stored.

**How to reconstruct it for Analysed Weather:**
```
badSlots = WeatherDetails WHERE FormsId = X AND Conditions != ''
badHours = SUM(badSlots.Hours)
steamingHours = PassageDetail.Hours WHERE FormsId = X

If BadDayHoursType = Hours:
    violated = badHours >= termsBadDayHours

If BadDayHoursType = AS_Ratio_Of_24_Hours:
    violated = (badHours / steamingHours) > (termsBadDayHours / 24)
```

For Reported Weather path, this reconstruction is not possible because there are no hourly slots — the whole period is treated as one block.

---

*Document generated from source code analysis of `gp-charterparty-jobs` TC2.0 AtSea worker.*
*Key source files: `WeatherExclusion.cs`, `Passages.cs`, `Speed.cs`, `Fuel.cs`, `EventExclusion.cs`, `PassageDetail.cs`, `PassageSpeedPerformance.cs`, `PassageFuelPerformance.cs`, `AnalyzedWeatherDetail.cs`, `Terms.cs`, `Enums.cs`, `Constants.cs`*
