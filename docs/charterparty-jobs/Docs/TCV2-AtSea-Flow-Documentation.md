# TCV2 API — Complete Flow Documentation

> This document covers the full end-to-end flow of the TCV2 At Sea job: from the HTTP request, through every layer of business logic, to what gets written to the database. Every intermediate entity, every algorithm, and every exclusion condition is explained in plain English.

---

## Table of Contents

1. [What Is This Job?](#1-what-is-this-job)
2. [Entry Points — How The Job Starts](#2-entry-points--how-the-job-starts)
3. [Step 1 — Fetch Vessels](#3-step-1--fetch-vessels)
4. [Step 2 — Fetch Terms (Charter Party Rules)](#4-step-2--fetch-terms-charter-party-rules)
5. [Step 3 — Fetch Raw Data From DB](#5-step-3--fetch-raw-data-from-db)
6. [Step 4 — Weather Unit Transformation](#6-step-4--weather-unit-transformation)
7. [Step 5 — Operator Date Filtering](#7-step-5--operator-date-filtering)
8. [Step 6 — Building Passages](#8-step-6--building-passages)
9. [Step 7 — Building PassageDetail (Per Report)](#9-step-7--building-passagedetail-per-report)
10. [Step 8 — Fuel Consumption Per Report](#10-step-8--fuel-consumption-per-report)
11. [Step 9 — ROB (Remaining On Board)](#11-step-9--rob-remaining-on-board)
12. [Step 10 — Weather Exclusion (Full Deep Dive)](#12-step-10--weather-exclusion-full-deep-dive)
13. [Step 11 — Event Exclusion](#13-step-11--event-exclusion)
14. [Step 12 — Short Day (SD) and Short Passage (SP) Checks](#14-step-12--short-day-sd-and-short-passage-sp-checks)
15. [Step 13 — Speed Performance Calculation](#15-step-13--speed-performance-calculation)
16. [Step 14 — Fuel Performance Calculation](#16-step-14--fuel-performance-calculation)
17. [Step 15 — Saving to the Database](#17-step-15--saving-to-the-database)
18. [All Exclusion Condition Codes Explained](#18-all-exclusion-condition-codes-explained)
19. [All Intermediate Entities Explained](#19-all-intermediate-entities-explained)
20. [Complete Data Flow Diagram (Text)](#20-complete-data-flow-diagram-text)

---

## 1. What Is This Job?

The TCV2 (Time Charter Version 2) job processes **sea performance data for oil tanker vessels** operating under a Time Charter Party agreement. It evaluates whether a vessel is performing as warranted by its charter terms — specifically looking at speed and fuel consumption.

The job runs in two modes:
- **On-demand**: Someone calls `POST /api/lng-job/TCV2` with a date range and optional vessel IMO number.
- **Scheduled**: Hangfire (background job scheduler) calls it automatically on a cron schedule defined in config.

The TCV2 job actually runs **7 sub-jobs** for the same vessel set:
1. At Sea (sea passage performance) ← this document focuses here
2. In Port
3. Pumping
4. Cargo Heating
5. Cargo Cooling
6. Tank Cleaning
7. IGS (Inert Gas System)

---

## 2. Entry Points — How The Job Starts

### Option A: HTTP API Call

```
POST /api/lng-job/TCV2
Content-Type: application/json

{
  "fromDate": "2024-01-01T00:00:00",
  "toDate": null,
  "imoNumber": null
}
```

- `fromDate` — mandatory. The start of the date window to process.
- `toDate` — optional. If null, defaults to current UTC time.
- `imoNumber` — optional. If provided, only that vessel is processed. If null, ALL vessels are processed.

The controller (`TCV2Controller`) calls `TCV2Service.RunJob()`.

### Option B: Scheduled (Hangfire Cron Job)

`TCV2Job.Execute()` is triggered by Hangfire on a schedule. It creates a `BaseInputParam` using dates from the config and calls the same `TCV2Service.RunJob()`.

---

### What TCV2Service Does

`TCV2Service.RunJob()` loops through 7 `WorkerType` values and for each one:
1. Creates a `JobService` for that worker type.
2. Calls `JobService.Start(input)`.
3. If one worker fails, it logs it, continues to the next — so one failure doesn't block the others.
4. Returns a summary string like "Sea Performance processed successfully. In Port Performance failed to process."

`JobService` uses `ObjectFactory.GetWorker(workerType)` to instantiate the correct worker class. For At Sea, this returns `Workers.TC2._0.AtSea.Worker`.

---

## 3. Step 1 — Fetch Vessels

**Code:** `Worker.RunWorkerAsync()` → `_commonRepository.GetVesselsTCV2(imoNumber)`

**Stored Procedure:** `dbo.Job_GetVesselsForTCV2Reports`

The system first asks: *which vessels should be processed?*

Each vessel record (`OilTanker`) contains:
- `IMONumber` — unique vessel identifier
- `TermsId` — links the vessel to its charter party terms
- `FromDate` / `ToDate` — the vessel's valid charter period
- `OperatorDates` — list of date ranges during which a specific operator was responsible

**Logic:**
- If the vessel has no `FromDate` set, use the `fromDate` from the API input.
- The date window used for data fetching = `max(inputParam.FromDate, vessel.FromDate)` to `min(inputParam.ToDate, vessel.ToDate or UTC now)`.

---

## 4. Step 2 — Fetch Terms (Charter Party Rules)

**Code:** `_repository.GetTerms(vessel.TermsId)`

**Stored Procedure:** `dbo.Job_GetTCV2AtSeaTerms`

Returns 8 result sets. These become the `Terms` object:

| Result Set | What It Contains | Stored As |
|---|---|---|
| Table 0 | Basic term info (TermId, scrubber, extrapolation flags) | `Terms` base fields |
| Table 1 | Active fuel types (e.g. FO, GO, BioFuel) | `Terms.ActiveFuelTypes` — `List<string>` |
| Table 2 | Fuel groups used in this charter (e.g. FO, GO) | `Terms.FuelGroups` — `List<string>` |
| Table 3 | Exclusion criteria (weather thresholds, passage hours) | `Terms.ExclusionCriteria` |
| Table 4 | Consumption categories (ME, AUX, Others) and their mapping | `Terms.ConsumptionCategories` |
| Table 5 | Clause customization (how to calculate speed/fuel deviation) | `Terms.Clause` |
| Table 6 | Warranted sea performance (speed + load + fuel warranty) | `Terms.SeaPerformanceDetails` |
| Table 7 | Bio-fuel flags (which fuel group bio-fuel maps to) | `Terms.BioFuelFlags` |

### Key Sub-Objects Explained

**ExclusionCriteria** — rules that define when a report should be excluded from performance calculations:
- `WindSpeed` — the threshold wind force above which weather is considered "bad"
- `WindSpeedUnit` — whether wind is measured in Beaufort (BF) or knots (kts)
- `WaveHeight` — wave height threshold
- `HeightUnit` — whether wave is in Douglas Sea Scale (DS), feet (ft), or metres (m)
- `BadDayHours` — how many hours of bad weather makes the whole day/report excluded
- `BadDayHoursType` — either a fixed number of hours, or a ratio out of 24
- `WeatherSource` — use reported weather (from ship) or analysed weather (from external provider)
- `MinimumAdvCurrentLimit` / `MaximumAdvCurrentLimit` — current speed bounds for current exclusion
- `MinimumPassageHours` — if the whole passage is shorter than this, it gets excluded (SP)
- `MinimumHRsInADay` — if a single report is shorter than this, it gets excluded (SD)

**SeaPerformanceDetails** — what the vessel is warranted to do at each speed and load condition:
- `Speed` — ordered speed (e.g. 13.0 knots)
- `LoadCondition` — "Laden" or "Ballast"
- `MEWarranty` — warranted main engine fuel consumption at this speed
- `AUXWarranty` — warranted auxiliary engine fuel consumption
- `OthersWarranty` — warranted other consumption
- `LowerSpeedLimit` / `UpperSpeedLimit` — the "about" band around the warranted speed
- `ConsumptionLowerLimit` / `ConsumptionUpperLimit` — the allowable fuel consumption band
- `HasAboutClause` — whether an "about" tolerance applies
- `FuelGroup` — which fuel type this warranty applies to

**If terms have no SeaPerformanceDetails, the vessel is skipped entirely.**

---

## 5. Step 3 — Fetch Raw Data From DB

**Code:** `_repository.GetDataFromDB(termsId, imoNumber, fromDate, toDate)`

**Stored Procedure:** `dbo.Job_GetTCV2AtSeaData`

Returns 7 result sets which populate the `DataFromDB` object:

| Result Set | Entity | What It Is |
|---|---|---|
| Table 0 | `List<PassageData>` | All sea reports (Noon, Departure, Arrival) for the vessel in the date range |
| Table 1 | `List<ReportFuelConsumption>` | Fuel consumed per category per report |
| Table 2 | `List<ROBData>` | Remaining On Board fuel per fuel type per report |
| Table 3 | `List<ExclusionEventDetail>` | Special events within a report that need to be excluded (e.g. slow steaming order, deviation) |
| Table 4 | `List<ExclusionEventsConsumptionDetail>` | The fuel consumed during those specific events |
| Table 5 | `List<FCOData>` | Fuel Change Over data (when vessel switches from one fuel to another) |
| Table 6 | `List<WeatherData>` | Hourly/sub-hourly analysed weather observations linked to each report |

### PassageData (raw report from DB)
Each row represents one voyage report filed by the ship. Key fields:
- `FormsId` — unique ID of the report form
- `EventId` — if this is an event report, this links to the event; otherwise null
- `VoyageNumber`
- `PortName` — port name (populated on arrival/departure reports)
- `LoadCondition` — "Laden" or "Ballast"
- `Position` — "Departure", "Noon", "Arrival"
- `FormDate` — report date/time in UTC (computed from offset-aware timestamp)
- `ObservedDistance` — nautical miles since last report
- `OrderedSpeed` (mapped from `CPSpeed`) — the speed the vessel was ordered to maintain
- `SOG` (mapped from `ObservedSpeed`) — speed over ground
- `RPM`, `Slip`, `ShaftPower` — engine performance metrics
- `ReportedWind`, `ReportedWave` — weather as reported by ship
- `AnalysedWind`, `AnalysedWave` — weather from external analysis provider
- `EffectiveCurrent` (mapped from `AnalyzedCurrent`)
- `IsExcluded` — whether this report was manually flagged as excluded in the source system
- `FormRemarks` — free text remarks

---

## 6. Step 4 — Weather Unit Transformation

**Code:** `_weather.TransoformData(terms.ExclusionCriteria, ref dataFromDB)`

Before any processing, all weather values are converted to match the units specified in the charter terms. This runs on **all raw PassageData and WeatherData records**.

**Wind conversion:**
- If terms say wind is in Beaufort (BF): the analysed wind (which comes in knots) is converted knots → Beaufort number.
- If terms say wind is in knots (kts): the reported wind (which comes in Beaufort) is converted Beaufort → knots.

**Wave conversion:**
- If terms say height is in Douglas Sea Scale (DS): wave height (in metres) is converted → DSS number.
- If terms say height is in feet (ft): wave height (in metres) is multiplied by 3.28084.
- If metres (m): no conversion needed.

This ensures that when we later compare `reportedWave > waveThreshold`, both values are in the same unit.

---

## 7. Step 5 — Operator Date Filtering

**Code:** `_operator.GetValidDates(vessel.OperatorDates)` → `_operator.GetFiltered(passages, dates)`

A vessel may have had different operators at different times during the charter. We only want to process reports that fall within valid operator periods.

**GetValidDates** takes the raw operator date ranges and deduplicates overlapping ones — keeping only non-overlapping ranges ordered by start date. This prevents double-counting reports that might fall in two overlapping operator windows.

**GetFiltered** then takes the full list of raw `PassageData` and returns only those whose `FormDate` falls within at least one valid operator date window.

---

## 8. Step 6 — Building Passages

**Code:** `Passages.GetPassage()`

This is the core algorithm that converts a flat list of raw reports into structured **voyage passages** (one departure → all noon reports → one arrival).

### How a Passage Is Identified

Reports are sorted by `FormDate` ascending.

The algorithm loops and for each iteration:

1. **Find the next Departure report** — the first report whose `Position` contains "departure".
2. **Voyage Number override** — if there is a "Shifting from Last Berth to Sea" event between the previous arrival and this departure, the departure's VoyageNumber is overridden with the event's VoyageNumber.
3. **Find the matching Arrival report** — the first report after the departure whose `Position` contains "arrival".
   - If an arrival is found but the port name is empty, look ahead for the next report with a port name and use that as the arrival port.
   - If no arrival is found (vessel is still at sea), use the last available report and mark `PortName = "At SEA"`.
4. **Collect all reports in between** — departure + all noon/intermediate reports + arrival, sorted by time.
5. **Process that passage** via `GetBasicPassageDetail()` to produce:
   - `List<PassageDetail>` — one entry per report in the passage
   - `List<ConsumptionDetails>` — fuel breakdown per report
6. **Calculate speed performance** for that passage.
7. **Calculate fuel performance** for that passage.
8. Append everything to the running output lists.
9. Loop until we have processed as many passages as there are arrival reports (or run out of departures).

### What Triggers the Loop to End
`arrivalCount` = total number of arrival reports in the data. The loop runs at most that many times. If it can't find a departure/arrival pair, it breaks early.

---

## 9. Step 7 — Building PassageDetail (Per Report)

**Code:** `Passages.GetBasicPassageDetail()`

For each raw `PassageData` report within the passage, a `PassageDetail` is built:

**Fields copied directly:**
- `TermId`, `IMONumber`, `FormsId`, `EventId`, `VoyageNumber`
- `PortName`, `LoadCondition`, `Position`
- `FormDateInUTC` — the UTC date-time of the report

**A unique `PassageID` (GUID)** is generated once per passage and shared across all reports in that passage. This groups them together.

**Fields only computed for non-departure reports** (noon/arrival reports):
- `Hours` — hours elapsed since the previous report = `(thisReportTime - previousReportTime).TotalHours`, rounded to 4 decimal places.
- `Distance` — observed nautical miles since last report.
- `SOG` — speed over ground.
- `OrderedSpeed` — the contracted speed for this leg.
- `EffectiveCurrent`, `RPM`, `Slip`, `ShaftPower` — engine metrics.
- `ROB` — serialized JSON of fuel remaining on board (see Step 9).
- Fuel consumption details (see Step 8).
- Exclusion conditions (see Steps 10–12).

The departure report itself is added with just the basic fields and no computed performance fields, because it is the starting point — there is no "previous report" to measure from.

---

## 10. Step 8 — Fuel Consumption Per Report

For each non-departure report, the system builds `ConsumptionDetails` — one entry per **fuel group** (e.g. FO, GO).

**Source:** `ReportFuelConsumption` records matching the report's `FormsId`.

**Process:**

1. Get all fuel consumption records for this `FormsId`.
2. If any record has fuel group "BioFuel" (BF), normalize it — remap it to FO, GO, or BF based on the vessel's `BioFuelFlags` from the charter terms.
3. Loop over each `FuelGroup` defined in the terms (e.g. ["FO", "GO"]).
4. For each fuel group, compute consumption for each engine type:
   - **ME (Main Engine):** Look at consumption records whose category matches a "ME" category from `ConsumptionCategories` and whose fuel group matches.
   - **AUX (Auxiliary Engines):** Same logic for "AUX" categories.
   - **Others:** Same for "Others" categories.
5. Only add a `ConsumptionDetails` entry if at least one of ME, AUX, or Others is > 0.

**ConsumptionDetails entity:**
- `TermId`, `IMONumber`, `FormsId`, `FormDateInUTC`
- `OrderedSpeed` — the speed at time of this report
- `FuelGroup` — which fuel (FO/GO/BF)
- `MEConsumption` — total ME fuel consumed in this period
- `AUXConsumption` — total AUX fuel consumed
- `OtherConsumption` — total other consumption

---

## 11. Step 9 — ROB (Remaining On Board)

For each report, the system looks up ROB data by `FormsId`. For every active fuel type in the terms, it finds the matching ROB record and stores the remaining quantity.

The ROB is stored as a JSON string on the `PassageDetail`. Example:
```json
[{"FuelType":"FO","ROB":245.3},{"FuelType":"GO","ROB":12.1}]
```

This is purely informational — ROB is not used in performance calculations but is stored for reference.

---

## 12. Step 10 — Weather Exclusion (Full Deep Dive)

Weather exclusion determines whether a report's sea conditions were bad enough to be excluded from charter party performance calculations. A vessel should not be held to its warranted speed/consumption if it was sailing through a storm.

### Two Sources of Weather

The charter terms specify which weather source to use:

**1. Reported Weather (`WeatherSource = ReportedWeather`)**
The ship's own officers recorded the wind and wave conditions. The system uses `reportedWind` and `reportedWave` from the `PassageData`.

**2. Analysed Weather (`WeatherSource = AnalysedWeather`)**
An external weather analysis provider (e.g. a weather routing company) provides hourly granular weather data independently of the ship's reports. This is stored in `WeatherData` records (Table 6 from the DB fetch), with multiple entries per report period.

---

### Pre-processing Analysed Weather (`ProcessWeatherData`)

Before any per-report evaluation happens, the system processes the full list of `WeatherData` records for the vessel:

1. Sort all weather records by `AnalyzedWeatherDate`.
2. For records index 1 onwards: compute `Hours` = time since the previous weather record.
3. For each weather record, evaluate independently whether **that hour-block** was bad:
   - `isWindExcluded = analyzedWind > windThreshold`
   - `isWaveExcluded = analyzedWave > waveThreshold`
   - `isCurrentExcluded = analyzedCurrent < minimumLimit OR analyzedCurrent > maximumLimit`
4. Assign a condition code to that weather record (e.g. `WI`, `WA`, `C`, `WW`, `WWC`, etc.).
5. Filter out any weather record with `Hours <= 0 or Hours > 2` — this removes invalid or too-long time gaps that would skew totals.

This gives us a clean list of hourly weather "slots", each tagged with whether that slot was bad and why.

---

### Per-Report Weather Evaluation (`CalculateWeatherDetails`)

For each non-departure `PassageDetail`, the system determines whether this report should be weather-excluded:

#### Path A — Reported Weather

1. Copy the report's own `reportedWind` and `reportedWave` onto the `PassageDetail`.
2. Check the **bad day hours** rule:
   - `BadDayHoursType.Hours`: if the report's total hours ≥ `BadDayHours` threshold → bad day violated.
   - `BadDayHoursType.AS_Ratio_Of_24_Hours`: compute ratio = `reportHours / steamingHours`. If that ratio > `(BadDayHours / 24)` → bad day violated.
   - **Important:** A report is only excluded if bad day is violated AND the wind/wave exceeded threshold. Both conditions must be true together.
3. `isWindExcluded = reportedWind > windThreshold AND badDayViolated`
4. `isWaveExcluded = reportedWave > waveThreshold AND badDayViolated`
5. `isCurrentExcluded = effectiveCurrent < min OR effectiveCurrent > max` (no bad day check for current)

#### Path B — Analysed Weather

1. Get all `WeatherData` records that belong to this report's `FormsId`.
2. Sum up the `Hours` of all bad-condition weather slots → `badTime`.
3. Sum up the `Hours` of all good-condition weather slots → `goodTime`.
4. Apply the **bad day hours** check against `badTime` vs total steaming hours.
5. If bad day is violated:
   - Collect all distinct condition codes from the bad weather slots.
   - `isWindExcluded` = any of those conditions is a "bad wind" condition (WI, WIC, WWC, WW).
   - `isWaveExcluded` = any is a "bad wave" condition (WWC, WAC, WA, WW).
   - `isCurrentExcluded` = any is a "bad current" condition (C, WIC, WWC, WAC).
6. Additionally, the system adjusts the `EventAnalyzedWindForce` / `EventAnalyzedWave` stored on the `PassageDetail` to reflect the exclusion determination — e.g. if wind IS excluded but the stored value is below threshold, force it above threshold so downstream reporting shows why it was excluded.

---

### Combining Into a Condition Code

After the above, the combination of wind/wave/current flags produces one code:

| Wind Excluded | Wave Excluded | Current Excluded | Code |
|:---:|:---:|:---:|---|
| ✓ | ✓ | ✓ | `WWC` — Wind + Wave + Current |
| ✓ | ✓ | — | `WW` — Wind + Wave |
| ✓ | — | ✓ | `WIC` — Wind + Current |
| — | ✓ | ✓ | `WAC` — Wave + Current |
| ✓ | — | — | `WI` — Wind only |
| — | ✓ | — | `WA` — Wave only |
| — | — | ✓ | `C` — Current only |
| — | — | — | No weather exclusion |

This code is added to the report's `Conditions` list.

---

### Unwarranted Speed Check

After weather, the system also checks: *is the ordered speed even in the charter party warranty table?*

If there is **no SeaPerformanceDetail entry matching this speed + load condition**, the report is tagged with `UW` (Unwarranted) — it was sailing at a speed that was never agreed upon in the charter terms.

---

### Analysed Weather Details (Separate Output)

Independently of per-report exclusion, the system also saves a detailed weather analysis table. After all passages are processed, `GetAnalyzedWeatherDetails()` converts the processed `WeatherData` list into `AnalyzedWeatherDetail` records — one per hourly weather slot — and saves them to the database. This is used for reporting/visualisation of weather patterns.

**AnalyzedWeatherDetail entity:**
- `TermId`, `IMONumber`, `FormsId`
- `FormDate` — the report date this weather slot belongs to
- `CalculatedDate` — the actual timestamp of this weather observation
- `Hours` — duration of this slot
- `WindSpeedInBF` — analysed wind in Beaufort
- `WaveHeightInDSS` — analysed wave in Douglas Sea Scale
- `CurrentInKTS` — analysed current in knots
- `Conditions` — the condition code for this slot (e.g. "WW", "WI", or empty if fine)

---

## 13. Step 11 — Event Exclusion

**Code:** `_eventExclusion.ComputeEventExclusion()`

Some reports contain **special events** — periods within the reporting window where the vessel was doing something that should be excluded from the performance calculation. Examples: manoeuvring in port approaches, anchor drifting, emergency stops.

These events come from `ExclusionEventDetails` (Table 3 from DB fetch).

### How Event Exclusion Works

For each report (`FormsId`), if there are any `ExclusionEventDetail` records matching that form:

1. **Adjust Hours:** `passageDetail.Hours -= (eventEndTime - eventStartTime).TotalHours` — subtract the event duration from the report's steaming hours.
2. **Adjust Distance:** `passageDetail.Distance += event.Distance` — the event distance is negative in the DB (stored as `-1 * actual distance`), so adding it effectively subtracts it from the passage distance.
3. **Adjust Fuel:** For each `ExclusionEventsConsumptionDetail` matching the event:
   - Match the event consumption to the right engine type (ME/AUX/Others) using `ConsumptionCategories`.
   - The event fuel consumption is already stored as negative in the DB.
   - Subtract the event fuel from the report's `ConsumptionDetails` (ME, AUX, Others each adjusted separately per fuel group).
4. **Tag the report:** If any hours were excluded, or distance was negative, or fuel was adjusted → add the event's `ConditionCode` (a free-text code like "E1", "E2") to the report's conditions list.

The SD (Short Day) check happens **after** event exclusion because the hours might have been reduced enough to make the remaining period shorter than the minimum.

---

## 14. Step 12 — Short Day (SD) and Short Passage (SP) Checks

After event exclusion has adjusted the hours:

**SD — Short Day**
- If the exclusion criteria defines a `MinimumHRsInADay` and the report's remaining `Hours` (after event deduction) is less than that minimum → tag with `SD`.
- This handles cases where a noon-to-noon period is very short (e.g. the vessel arrived at port partway through the day).

**SP — Short Passage**
- Evaluated once per passage (not per report).
- If `(arrival.FormDate - departure.FormDate).TotalHours < MinimumPassageHours` → tag with `SP`.
- A passage that is too short to be statistically valid is excluded entirely.

---

## 15. Step 13 — Speed Performance Calculation

**Code:** `Speed.GetPassageSpeedPerformances(passageFormDetails, terms)`

After all `PassageDetail` records for a passage are built with their conditions, speed performance is calculated.

### Algorithm

1. From the passage's reports, find all **distinct ordered speeds** (e.g. [12.0, 13.0]).
2. For each distinct speed:
   - Gather all reports in this passage with that ordered speed (`speedPassage`).
   - Create one `PassageSpeedPerformance` record.
   - Set passage metadata: TermId, PassageID, IMONumber, VoyageNumber, from/to ports, departure/arrival dates, ordered speed.
   - **IsUnWarranted** = is there NO entry in `SeaPerformanceDetails` matching this speed AND load condition? If no warranty exists → true.
   - **IsShortPassage** = does any report in this passage have the `SP` condition?
3. If neither IsUnWarranted nor IsShortPassage:
   - Find the matching `SeaPerformanceDetails` entry.
   - **Good Weather passages** = all reports that have no bad weather conditions (none of the BadWeatherConditions codes).
   - Calculate:
     - `AllWeatherHours` — total hours of all speed-matched reports
     - `AllWeatherDistance` — total distance
     - `GoodWeatherHours` — hours from good-weather reports only
     - `GoodWeatherDistance` — distance from good-weather reports only
     - `AllWeatherSpeed` = `AllWeatherDistance / AllWeatherHours`
     - `GoodWeatherSpeed` = `GoodWeatherDistance / GoodWeatherHours`
     - Warranted speed values from the terms (lower, warranted, upper)

**PassageSpeedPerformance entity:**
- `TermId`, `PassageID`, `IMONumber`, `VoyageNumber`
- `PassageFrom`, `PassageTo` — port names
- `DepartureDate`, `ArrivalDate`
- `OrderedSpeed`
- `IsUnWarranted`, `IsShortPassage`
- `AllWeatherHours`, `AllWeatherDistance`, `AllWeatherSpeed`
- `GoodWeatherHours`, `GoodWeatherDistance`, `GoodWeatherSpeed`
- Warranted speed bounds from terms

---

## 16. Step 14 — Fuel Performance Calculation

**Code:** `Fuel.GetFuelPerformances(passageFormDetails, consumptionDetails, terms)`

Similar structure to speed performance but broken down by **speed × fuel group**.

### Algorithm

1. Find all distinct ordered speeds from the passage.
2. For each speed:
   - For each fuel group in the terms (e.g. FO, GO):
     - Create one `PassageFuelPerformance` record.
     - Set passage metadata.
     - **IsUnWarranted** — same check as speed.
     - **IsShortPassage** — same check.
     - Get the relevant `ConsumptionDetails` records: match by speed, fuel group, and having at least one non-zero consumption value.
     - **Good weather consumptions** = those whose `FormsId` is in the good weather passages list.
     - Calculate:
       - `AllWeatherHours`, `AllWeatherDistance`
       - `GoodWeatherHours`, `GoodWeatherDistance`
       - ME, AUX, Others consumption totals for both all-weather and good-weather subsets
       - Warranted consumptions from the matching `SeaPerformanceDetails` (ME, AUX, Others warrantied values)

**PassageFuelPerformance entity:**
- `TermId`, `PassageID`, `IMONumber`, `VoyageNumber`
- `PassageFrom`, `PassageTo`
- `DepartureDate`, `ArrivalDate`
- `OrderedSpeed`, `FuelGroup`
- `IsUnWarranted`, `IsShortPassage`
- Per-weather-category: `AllWeatherMEConsumption`, `GoodWeatherMEConsumption`, etc.
- Warranted values: `WarrantedMEConsumption`, `WarrantedAUXConsumption`, etc.

---

## 17. Step 15 — Saving to the Database

**Code:** `Worker.RunWorkerAsync()` — after all passages are processed

Only if **all four output lists are non-empty** (passages, consumptions, speed performances, fuel performances) does the system proceed to write:

1. **Delete existing data** — `dbo.Job_DeleteTCV2AtSea` with `IsHardDelete = true` for this IMO + TermId. This cleans out any previously computed data before reinserting fresh results.

2. **Convert to DataTables** — all four lists + excluded reports are converted to `DataTable` objects (using `ListToDataTable()` extension).

3. **Insert main performance data** — `dbo.Job_InsertTCV2AtSeaPassageData` receives 5 table-valued parameters:
   - `@PassageData` — the `PassageDetail` list
   - `@ConsumptionData` — the `ConsumptionDetails` list
   - `@SpeedData` — the `PassageSpeedPerformance` list
   - `@FuelData` — the `PassageFuelPerformance` list
   - `@TcV2ExcludedReports` — any manually excluded reports

4. **Insert weather analysis** — if there is any analysed weather data, `dbo.Job_InsertTCV2AtSeaWeatherData` receives the `AnalyzedWeatherDetail` list.

---

## 18. All Exclusion Condition Codes Explained

These codes are stored as a comma-separated string in `PassageDetail.Conditions`.

| Code | Name | What It Means |
|---|---|---|
| `WI` | Wind | Wind speed exceeded the charter threshold |
| `WA` | Wave | Wave height exceeded the charter threshold |
| `C` | Current | Adverse current was outside the allowed range |
| `WW` | Wind + Wave | Both wind and wave exceeded thresholds |
| `WIC` | Wind + Current | Both wind and current exceeded thresholds |
| `WAC` | Wave + Current | Both wave and current exceeded thresholds |
| `WWC` | Wind + Wave + Current | All three exceeded thresholds simultaneously |
| `SD` | Short Day | The report's steaming hours (after event deduction) were less than the minimum required |
| `SP` | Short Passage | The total passage duration was less than the minimum hours in the terms |
| `EXC` | Excluded | The report was manually flagged as excluded in the source system |
| `UW` | Unwarranted | The ordered speed has no matching warranty in the charter party terms |
| `FCO` | Fuel Change Over | The vessel was in the process of changing fuel type |
| `FBOG` | Force BOG | (LNG specific) Forced boil-off gas condition |
| `X` | General Exclusion | Generic exclusion (used in some contexts) |
| Event codes | e.g. E1, E2 | Operator-defined event codes (manoeuvring, deviation, etc.) — free text from `ExclusionEventDetail.ConditionCode` |

**BadWeatherConditions** (used to identify "good weather" reports for performance calc):
`WI, C, WIC, FCO, WWC, WAC, WA, WW, EXC, SD, SP`

Any report with one of these conditions is excluded from the "good weather" performance calculation. Reports with event codes only (E1, E2, etc.) may still contribute to good-weather periods depending on whether hours/distance/fuel adjustments removed the issue.

---

## 19. All Intermediate Entities Explained

### Input / Configuration Entities

| Entity | Where Used | Purpose |
|---|---|---|
| `BaseInputParam` | API input | fromDate, toDate, optional IMONumber |
| `OilTanker` | Vessel record | IMO, TermsId, operator dates |
| `OperatorDates` | Vessel sub-record | Start/end dates for a specific charterer/operator |
| `Terms` | Charter party config | Top-level container for all charter terms |
| `ExclusionCriteria` | Inside Terms | Weather thresholds and hour minimums |
| `SeaPerformanceDetails` | Inside Terms | Warranted speed/consumption per speed/load |
| `SeaConsumptionCategories` | Inside Terms | Mapping of fuel categories to engine types |
| `ClauseCustomization` | Inside Terms | How to calculate deviations (which speed to use, etc.) |
| `BioFuelFlags` | Inside Terms | Which fuel group bio-fuel maps to |

### Raw Data Entities (from DB)

| Entity | Source Table | Purpose |
|---|---|---|
| `PassageData` | SP result set 0 | One row per voyage report |
| `ReportFuelConsumption` | SP result set 1 | Fuel consumed per category per report |
| `ROBData` | SP result set 2 | Fuel remaining on board per report |
| `ExclusionEventDetail` | SP result set 3 | Events within reports to be excluded |
| `ExclusionEventsConsumptionDetail` | SP result set 4 | Fuel consumed during those events |
| `FCOData` | SP result set 5 | Fuel change-over event data |
| `WeatherData` | SP result set 6 | Hourly analysed weather per report period |

### Computed / Output Entities

| Entity | Produced By | Purpose |
|---|---|---|
| `PassageDetail` | `GetBasicPassageDetail()` | One row per report, enriched with hours, distance, conditions, ROB |
| `ConsumptionDetails` | `GetBasicPassageDetail()` | Fuel breakdown per report per fuel group |
| `PassageSpeedPerformance` | `Speed.GetPassageSpeedPerformances()` | Speed performance summary per passage per speed |
| `PassageFuelPerformance` | `Fuel.GetFuelPerformances()` | Fuel performance summary per passage per speed per fuel group |
| `ExcludedReport` | `GetBasicPassageDetail()` | Summary of manually excluded reports with fuel totals |
| `AnalyzedWeatherDetail` | `WeatherExclusion.GetAnalyzedWeatherDetails()` | Hourly weather analysis with condition tags |

---

## 20. Complete Data Flow Diagram (Text)

```
HTTP POST /api/lng-job/TCV2
        │
        ▼
TCV2Controller.Post()
        │
        ▼
TCV2Service.RunJob()
        │
        ├── foreach WorkerType [AtSea, InPort, Pumping, Heating, Cooling, Cleaning, IGS]
        │           │
        │           ▼
        │   JobService.Start(inputParam)
        │           │
        │           ▼
        │   ObjectFactory.GetWorker(WorkerType.TCV2AtSea)
        │           │
        │           ▼
        │   TC2.0.AtSea.Worker.RunWorkerAsync()
        │           │
        │           ├── [DB] GetVesselsTCV2() → List<OilTanker>
        │           │
        │           └── foreach vessel
        │                   │
        │                   ├── [DB] GetTerms(termsId) → Terms
        │                   │       (8 result sets: basic, fuels, groups,
        │                   │        exclusion, categories, clause,
        │                   │        warranties, bio flags)
        │                   │
        │                   ├── [DB] GetDataFromDB(termsId, imoNumber,
        │                   │       fromDate, toDate) → DataFromDB
        │                   │       (7 result sets: passages, fuel cons,
        │                   │        ROB, events, event cons, FCO, weather)
        │                   │
        │                   ├── WeatherExclusion.TransformData()
        │                   │       Convert wind/wave to charter units
        │                   │
        │                   ├── Operator.GetValidDates()
        │                   │   Operator.GetFiltered()
        │                   │       Keep only reports in operator windows
        │                   │
        │                   ├── Passages.GetPassage()
        │                   │       │
        │                   │       ├── Sort passages by date
        │                   │       ├── WeatherExclusion.ProcessWeatherData()
        │                   │       │       Compute hourly weather slots
        │                   │       │       Tag each slot with condition code
        │                   │       │
        │                   │       └── Loop: find Departure → Arrival pairs
        │                   │               │
        │                   │               ├── GetBasicPassageDetail()
        │                   │               │       foreach report:
        │                   │               │       ├── Build PassageDetail
        │                   │               │       ├── Compute hours/distance
        │                   │               │       ├── Build ConsumptionDetails
        │                   │               │       │   (ME/AUX/Others per fuel group)
        │                   │               │       ├── Set ROB (JSON)
        │                   │               │       ├── WeatherExclusion.CalculateWeatherDetails()
        │                   │               │       │   → WI / WA / C / WW / WIC / WAC / WWC
        │                   │               │       ├── Check UW (unwarranted speed)
        │                   │               │       ├── Check EXC (manual exclusion)
        │                   │               │       ├── EventExclusion.ComputeEventExclusion()
        │                   │               │       │   → deduct event hours/distance/fuel
        │                   │               │       │   → add event condition codes
        │                   │               │       ├── Check SD (short day)
        │                   │               │       └── Check SP (short passage)
        │                   │               │
        │                   │               ├── Speed.GetPassageSpeedPerformances()
        │                   │               │       → PassageSpeedPerformance per speed
        │                   │               │
        │                   │               └── Fuel.GetFuelPerformances()
        │                   │                       → PassageFuelPerformance per speed × fuel group
        │                   │
        │                   ├── [DB] DeleteDetail(imoNumber, termId)
        │                   │       Hard delete existing data
        │                   │
        │                   ├── [DB] InsertData(passages, consumptions,
        │                   │       speedPerf, fuelPerf, excludedReports)
        │                   │
        │                   └── [DB] InsertData(analyzedWeatherDetails)
        │
        ▼
Return summary string to caller
```

---

*Document generated from source code analysis of `gp-charterparty-jobs` — TC2.0 AtSea worker.*
