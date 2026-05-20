# TCV2 AtSea — Passage Details: Column-by-Column Calculation Guide

> This document explains exactly how every column in every output table is calculated.
> It traces each value from its raw source table in the database all the way through every transformation, conversion, and algorithm until it lands in the final output.

---

## Table of Contents

1. [Raw Input Tables (from DB)](#1-raw-input-tables-from-db)
2. [Table: TcV2AtSeaPassageDetails — Every Column Explained](#2-table-tcv2atseapassagedetails--every-column-explained)
3. [Table: TcV2AtSeaConsumptionDetails — Every Column Explained](#3-table-tcv2atseaconsumptiondetails--every-column-explained)
4. [Table: TcV2AtSeaPassageSpeedPerformance — Every Column Explained](#4-table-tcv2atseapassagespeedperformance--every-column-explained)
5. [Table: TcV2AtSeaPassageFuelPerformance — Every Column Explained](#5-table-tcv2atseapassagefuelperformance--every-column-explained)
6. [Table: TcV2AtSeaWeatherDetails — Every Column Explained](#6-table-tcv2atseaweatherdetails--every-column-explained)
7. [Analyzed Wind: Full Journey from Raw Data to Final Value](#7-analyzed-wind-full-journey-from-raw-data-to-final-value)
8. [The Conditions Column: How Every Code Is Set](#8-the-conditions-column-how-every-code-is-set)
9. [The ROB Column: How It Is Built](#9-the-rob-column-how-it-is-built)
10. [Fuel Consumption Calculation Deep Dive](#10-fuel-consumption-calculation-deep-dive)

---

## 1. Raw Input Tables (from DB)

The stored procedure `dbo.Job_GetTCV2AtSeaData` returns 7 result sets. These are the **raw source tables** for everything.

### Result Set 0 — PassageData (voyage reports)

Each row = one report filed by the ship's officers.

| Raw Column | Maps To | Notes |
|---|---|---|
| `FormsId` | `PassageData.FormsId` | Unique ID for the report form |
| `EventId` | `PassageData.EventId` | Null for regular reports; filled for event reports |
| `VoyageNumber` | `PassageData.VoyageNumber` | Voyage number as filed |
| `PortName` | `PassageData.PortName` | Port name (only meaningful on arrival/departure) |
| `LoadCondition` | `PassageData.LoadCondition` | "Laden" or "Ballast" |
| `Position` | `PassageData.Position` | "Departure", "Noon", "Arrival" etc. |
| `FormDateWithOffset` + `FormDateOffset` | `PassageData.FormDate` | UTC date computed by stripping the offset: `FormDateWithOffset.GetUTCDateTime(FormDateOffset)` |
| `Distance` (string) | `PassageData.ObservedDistance` | Parsed to decimal, rounded to 4 dp. This is nautical miles since last report |
| `ObservedSpeed` (string) | `PassageData.SOG` | Speed Over Ground, parsed to decimal, rounded to 4 dp |
| `CPSpeed` (string) | `PassageData.OrderedSpeed` | The contracted/ordered speed. If null → returns 0 |
| `EngineRPM` (string) | `PassageData.RPM` | Rounds to 4 dp |
| `EngineSlip` (string) | `PassageData.Slip` | Rounds to 4 dp |
| `EngineShaftPower` (string) | `PassageData.ShaftPower` | Rounds to 4 dp |
| `EventType` | `PassageData.EventType` | Used to detect "Shifting from Last Berth to Sea" |
| `EventVoyageNumber` | `PassageData.EventVoyageNumber` | Used to override VoyageNumber on departure |
| `ReportedWindSpeedFromDB` (string) | `PassageData.ReportedWind` | Reported by ship officers. Converted to charter units (BF or kts) before use |
| `ReportedWaveFromDB` (string) | `PassageData.ReportedWave` | Reported by ship officers. Converted to charter units (DS, ft, m) before use |
| `AnalysedWindFromDB` (string) | `PassageData.AnalysedWind` | From external weather provider. Converted to charter units before use |
| `AnalysedWaveFromDB` (string) | `PassageData.AnalysedWave` | From external weather provider. Converted to charter units before use |
| `AnalyzedCurrent` (string) | `PassageData.EffectiveCurrent` | Current in knots from external provider |
| `IsFormReportExcluded` (bool?) | `PassageData.IsExcluded` | If true → this report was manually flagged for exclusion |
| `FormRemarks` | `PassageData.FormRemarks` | Free-text remarks from the officer |

### Result Set 1 — ReportFuelConsumption

Each row = fuel consumed in one category for one report.

| Raw Column | Notes |
|---|---|
| `FormsId` | Links back to the report |
| `Category` | Engine/system category name (e.g. "Main Engine", "Aux Engine") |
| `FuelGroup` | "FO", "GO", or "BioFuel" |
| `Consumption` (string) | Parsed to decimal, rounded to 4 dp = actual tonnes consumed |
| `FormDateWithOffset` + `FormDateOffset` | UTC date of the report |

### Result Set 2 — ROBData

Each row = remaining on board quantity for one fuel type on one report.

| Raw Column | Notes |
|---|---|
| `FormsId` | Links to the report |
| `FuelType` | e.g. "FO", "GO", "VLSFO" |
| `Remaining` (string) | Parsed to decimal, rounded to 4 dp = tonnes remaining |

### Result Set 3 — ExclusionEventDetail

Each row = a special event within a report that needs to be excluded from performance.

| Raw Column | Notes |
|---|---|
| `FormsId` | Report this event belongs to |
| `EventId` | Unique event identifier |
| `EventDistance` (string) | Distance during the event. Stored as negative: `parsed * -1` |
| `EventStartTimeWithOffset` | UTC start of event |
| `EventEndTimeWithOffset` | UTC end of event |
| `ConditionCode` | The condition tag for this event (e.g. "E1", "MANOEUVRE") |

### Result Set 4 — ExclusionEventsConsumptionDetail

Each row = fuel consumed during one exclusion event.

| Raw Column | Notes |
|---|---|
| `EventId` | Links to the event |
| `FormsId` | Links to the report |
| `Category` | Engine category |
| `FuelGroup` | FO/GO/BioFuel |
| `Consumption` (string) | Stored as negative: `parsed * -1` (will be deducted from report total) |

### Result Set 5 — FCOData

Fuel Change Over records. Currently fetched but not actively used in the AtSea calculation path.

### Result Set 6 — WeatherData

Each row = one hourly/sub-hourly analysed weather observation.

| Raw Column | Notes |
|---|---|
| `FormsId` | Links to the report this weather slot falls under |
| `AnalyzedWeatherDate` | Exact timestamp of this observation |
| `AnalyzedWindDB` (string) | Wind speed in knots from weather provider |
| `AnalyzedWaveDB` (string) | Wave height in metres from weather provider |
| `AnalyzedCurrentDB` (string) | Current speed in knots |
| `FormDateWithOffset` + `FormDateOffset` | The report date (used for grouping) |

---

## 2. Table: TcV2AtSeaPassageDetails — Every Column Explained

This is the main passage detail table. One row per voyage report within a processed passage.

---

### `TermId`
**Source:** `Terms.TermId` (from charter party configuration)
**How set:** Copied directly from the loaded `Terms` object for this vessel.

---

### `PassageID`
**Source:** Generated in code
**How set:** A `Guid.NewGuid()` is created **once per passage** (one departure→arrival group) and assigned to every report within that passage. This is how all reports belonging to the same voyage leg are linked together.

---

### `IMONumber`
**Source:** `OilTanker.IMONumber` from the vessel list
**How set:** Passed in directly. Identifies which vessel this record belongs to.

---

### `FormsId`
**Source:** `PassageData.FormsId` (Result Set 0)
**How set:** Copied directly. This is the unique ID of the original report form in the source system.

---

### `EventId`
**Source:** `PassageData.EventId` (Result Set 0)
**How set:** Copied directly. Null for regular noon/departure/arrival reports. Populated when the report row came from an event record.

---

### `VoyageNumber`
**Source:** `PassageData.VoyageNumber` with possible override
**How set:**
1. Start with the voyage number from the raw report.
2. **Special case for the departure report:** Look for any event row in the dataset where:
   - `FormDate` is before or equal to the departure report's date
   - `EventType` equals "shifting from last berth to sea" (case-insensitive)
   - `EventVoyageNumber` is not null
3. If such an event exists, **override** the departure's `VoyageNumber` with `EventVoyageNumber` from that event.
4. This ensures the correct voyage number is used when a vessel's outbound voyage is defined by the shifting event rather than the departure report itself.

---

### `PortName`
**Source:** `PassageData.PortName`
**How set:**
- Copied from the raw passage data.
- For the **arrival report**: if the arrival report's port name is empty, the system looks ahead in the full dataset for the next report with a non-empty port name and uses that. This handles cases where the vessel continues reporting after arrival before formally naming the port.
- If no arrival report was found (vessel still at sea), port name is set to `"At SEA"`.

---

### `LoadCondition`
**Source:** `PassageData.LoadCondition`
**How set:** Copied directly. Typically "Laden" or "Ballast". This is critical — it determines which warranty row to compare against in the charter party terms.

---

### `Position`
**Source:** `PassageData.Position` with normalization
**How set:** The raw position string is normalised:
- If it contains "arrival" → stored as `"Arrival"`
- If it contains "departure" → stored as `"Departure"`
- If it contains "noon" → stored as `"Noon"`
- Otherwise → stored as-is

---

### `FormDateInUTC`
**Source:** `PassageData.FormDateWithOffset` + `PassageData.FormDateOffset`
**How set:** The date-time offset stored in the DB (which is in the local time zone of the vessel's position) is converted to UTC using the offset string. The formula is: `FormDateWithOffset.GetUTCDateTime(FormDateOffset)`.

---

### `Hours`
**Source:** Calculated from consecutive report dates
**How set:**
- **Departure report:** `Hours` is null (not set — it's the starting point, there is no previous report).
- **All other reports (Noon, Arrival):**
  ```
  Hours = (thisReport.FormDateInUTC - previousReport.FormDateInUTC).TotalHours
  ```
  Rounded to 4 decimal places. "Previous report" means the immediately preceding report in the sorted passage list.
- **After event exclusion:** Hours is further adjusted by subtracting the duration of any excluded events:
  ```
  Hours = Hours - (event.EventEndTime - event.EventStartTime).TotalHours
  ```
  This means hours can be smaller than the raw gap between reports.

---

### `Distance`
**Source:** `PassageData.ObservedDistance` (from raw column `Distance`)
**How set:**
- **Departure report:** not set (null).
- **All other reports:** `passage.ObservedDistance` — the nautical miles observed since the last report as logged by the ship.
- **After event exclusion:** if the event has a distance (which is stored negative in the DB), it is added to `passageDetail.Distance`. Since it's negative, this effectively subtracts the event's distance.
  ```
  Distance = Distance + event.Distance   (event.Distance is already negative)
  ```

---

### `SOG`
**Source:** `PassageData.SOG` (from raw column `ObservedSpeed`)
**How set:** Copied directly (rounded to 4 dp). Speed over ground as observed by the ship. Not used in performance calculations — stored for reference.

---

### `OrderedSpeed`
**Source:** `PassageData.OrderedSpeed` (from raw column `CPSpeed`)
**How set:** Copied directly (rounded to 4 dp). This is the speed the vessel was contractually ordered to maintain under the charter party. Used to match against warranty rows.

---

### `RPM`
**Source:** `PassageData.RPM` (from raw column `EngineRPM`)
**How set:** Copied directly, rounded to 4 dp. Engine revolutions per minute at time of report.

---

### `Slip`
**Source:** `PassageData.Slip` (from raw column `EngineSlip`)
**How set:** Copied directly, rounded to 4 dp. Propeller slip percentage.

---

### `ShaftPower`
**Source:** `PassageData.ShaftPower` (from raw column `EngineShaftPower`)
**How set:** Copied directly, rounded to 4 dp. Shaft power in kW or equivalent.

---

### `ROB`
**Source:** `ROBData` (Result Set 2) + `Terms.ActiveFuelTypes`
**How set:** See [Section 9 — The ROB Column](#9-the-rob-column-how-it-is-built).
Stored as a JSON string. Example: `[{"FuelType":"FO","ROB":245.3},{"FuelType":"GO","ROB":12.1}]`

---

### `ReportedWindForce`
**Source:** `PassageData.ReportedWind` (from raw column `ReportedWindSpeedFromDB`)
**How set:**
1. Raw value is parsed from string to decimal.
2. **Unit conversion** (done in `TransformData` before processing begins):
   - If charter terms say wind unit is `BF` (Beaufort): the **analysed** wind is converted knots → Beaufort. The reported wind field here stays as filed by the ship.
   - If charter terms say wind unit is `kts` (knots): the **reported** wind is converted from Beaufort → knots using Beaufort scale conversion.
3. The converted value is stored on `PassageDetail.ReportedWindForce`.
4. Only used in performance calculations when `WeatherSource = ReportedWeather`.

---

### `ReportedWave`
**Source:** `PassageData.ReportedWave` (from raw column `ReportedWaveFromDB`)
**How set:**
1. Parsed from string to decimal.
2. **Unit conversion** (done in `TransformData`):
   - If terms unit is `DS` (Douglas Sea Scale): raw metres value → converted to DSS number.
   - If terms unit is `ft` (feet): raw metres value × 3.28084.
   - If terms unit is `m` (metres): no conversion.
3. Stored on `PassageDetail.ReportedWave`.
4. Only used when `WeatherSource = ReportedWeather`.

---

### `EventAnalyzedWindForce`
**Source:** `PassageData.AnalysedWind` (from raw column `AnalysedWindFromDB`), then possibly adjusted
**How set:** See [Section 7 — Analyzed Wind Full Journey](#7-analyzed-wind-full-journey-from-raw-data-to-final-value) for the complete path.

In short:
1. Raw analysed wind arrives in knots.
2. Unit converted to charter units (if terms say BF, convert knots → Beaufort).
3. Copied to `PassageDetail.EventAnalyzedWindForce` at the start of weather evaluation.
4. **May be adjusted** by the analysed weather logic to ensure it correctly reflects whether wind exclusion applies — e.g. if the hourly weather data says wind exclusion IS triggered but the event-level wind value is at or below threshold, the system forces `EventAnalyzedWindForce = windThreshold + 1` to make the stored value consistent with the exclusion decision.

---

### `EventAnalyzedWave`
**Source:** `PassageData.AnalysedWave` (from raw column `AnalysedWaveFromDB`), then possibly adjusted
**How set:** Same logic as `EventAnalyzedWindForce` but for wave height. Unit converted per terms (DSS/ft/m), then adjusted if needed to reflect the exclusion decision.

---

### `EffectiveCurrent`
**Source:** `PassageData.EffectiveCurrent` (from raw column `AnalyzedCurrent`)
**How set:** Parsed from string to decimal. This is the net current (knots) experienced by the vessel. Used in speed performance calculations if `CurrentFactor = Include` in the charter terms.

---

### `Conditions`
**Source:** Computed from multiple checks
**How set:** See [Section 8 — The Conditions Column](#8-the-conditions-column-how-every-code-is-set).

This is a comma-separated string of exclusion codes, e.g. `"WW,SD"` or `"WI,E1"`. The departure report never gets conditions — only noon and arrival reports do.

---

## 3. Table: TcV2AtSeaConsumptionDetails — Every Column Explained

One row per report per fuel group (e.g. a report using FO and GO generates two rows).

---

### `TermId`, `IMONumber`, `FormsId`, `FormDateInUTC`
Copied directly from the `PassageDetail` being processed.

---

### `OrderedSpeed`
Copied from `PassageDetail.OrderedSpeed` at the time the consumption detail is built. Same value that was pulled from `CPSpeed` in the raw report.

---

### `FuelGroup`
**Source:** `Terms.FuelGroups` list
One consumption row is created per fuel group defined in the charter terms (e.g. "FO", "GO"). The row's `FuelGroup` tells you which fuel this row covers.

---

### `MEConsumption`
**Source:** `ReportFuelConsumption` (Result Set 1)
**How calculated:**
1. Get all `ReportFuelConsumption` rows for this `FormsId`.
2. Filter to rows where:
   - `FuelGroup` matches this fuel group (e.g. "FO")
   - `Category` matches any category listed under engine type `"me"` in `Terms.ConsumptionCategories`
3. Sum all matching `FuelConsumption` values.
4. If no matches → `MEConsumption` = null.
5. **After event exclusion:** if an `ExclusionEventsConsumptionDetail` for this event and fuel group has a non-zero ME deduction (stored negative), it is added (i.e. subtracted) from `MEConsumption`.

---

### `AUXConsumption`
Same logic as `MEConsumption` but category must match engine type `"aux"` in `ConsumptionCategories`.

---

### `OtherConsumption`
Same logic but for engine type `"others"`.

---

### Bio-fuel normalization (affects all three consumption fields)

Before splitting by engine type, the system checks if any consumption record has `FuelGroup = "BioFuel"`. If yes:
- If `Terms.BioFuelFlags.HasBioFuelFO` is true → remap that record's FuelGroup to `"FO"`.
- Else if `HasBioFuelGO` → remap to `"GO"`.
- Else if `HasBioFuelBF` → keep as `"BF"`.

This ensures bio-fuel consumption is counted under the correct fuel group rather than being orphaned.

---

## 4. Table: TcV2AtSeaPassageSpeedPerformance — Every Column Explained

One row per distinct ordered speed per passage.

---

### `TermId`, `PassageID`, `IMONumber`, `VoyageNumber`
Copied from `passages[0]` (the first report in the passage).

---

### `PassageFrom`, `DepartureDate`
`PassageFrom` = `passages[0].PortName` (the departure port name)
`DepartureDate` = `passages[0].FormDateInUTC` (UTC time of the departure report)

---

### `PassageTo`, `ArrivalDate`
`PassageTo` = `passages[last].PortName` (the arrival port)
`ArrivalDate` = `passages[last].FormDateInUTC` (UTC time of the arrival report)

---

### `OrderedSpeed`
The distinct speed being evaluated. For each unique value of `OrderedSpeed` found across all reports in this passage (excluding nulls and zeros), one row is created.

---

### `IsUnWarranted`
**True** if there is no `SeaPerformanceDetails` entry in the charter terms matching:
- `Speed == orderedSpeed` AND
- `LoadCondition == passages[0].LoadCondition` (case-insensitive, null-safe)

If no warranty exists for this speed + load combination → the row is marked unwarranted and no performance numbers are calculated.

---

### `IsShortPassage`
**True** if any report in the passage has `"SP"` in its `Conditions` string.
SP is set when `(arrival.FormDate - departure.FormDate).TotalHours < Terms.ExclusionCriteria.MinimumPassageHours`.

---

### `AllWeatherHours`
**Condition:** Only calculated if `IsUnWarranted = false` AND `IsShortPassage = false`.

Sum of `Hours` from all reports at this ordered speed where `Hours > 0`.
```
AllWeatherHours = SUM(report.Hours) for all speedPassage reports where Hours > 0
```

---

### `AllWeatherDistance`
Sum of `Distance` from all reports at this ordered speed where `Distance > 0`.
```
AllWeatherDistance = SUM(report.Distance) for all speedPassage reports where Distance > 0
```

---

### `AllWAvgEffectiveCurrent`
Average of `EffectiveCurrent` across all reports at this speed that have a non-null current value.
```
AllWAvgEffectiveCurrent = AVERAGE(report.EffectiveCurrent) for reports where EffectiveCurrent != null
```
Rounded to 2 decimal places.

---

### `AllWPerformedSpeed`
```
AllWPerformedSpeed = AllWeatherDistance / AllWeatherHours   (rounded to 4 dp)
```
If `AllWeatherHours = 0` → result is 0.

**If current factor is included** (`Terms.ExclusionCriteria.CurrentFactor = Include`):
```
AllWPerformedSpeed = AllWPerformedSpeed - AllWAvgEffectiveCurrent
```
Adverse current (positive value) reduces the effective performed speed; favourable current (negative) increases it.

---

### `GWeatherHours`
Same as `AllWeatherHours` but only from **good weather reports** — reports that have **none** of the bad weather condition codes in their `Conditions` string.

Bad weather conditions = `WI, C, WIC, FCO, WWC, WAC, WA, WW, EXC, SD, SP`

```
gwPassages = speedPassage WHERE Conditions does NOT contain any bad weather code
GWeatherHours = SUM(gwPassage.Hours) for gwPassages where Hours > 0
```

---

### `GWeatherDistance`
Same filter as above but summing distance.
```
GWeatherDistance = SUM(gwPassage.Distance) for gwPassages where Distance > 0
```

---

### `GWAvgEffectiveCurrent`
Average current from good weather reports only.

---

### `GWPerformedSpeed`
```
GWPerformedSpeed = GWeatherDistance / GWeatherHours   (rounded to 4 dp)
```
Adjusted for current if current factor is included:
```
GWPerformedSpeed = GWPerformedSpeed - GWAvgEffectiveCurrent
```

---

### `MinAllowedTime`
The **minimum** hours the vessel should have taken to cover `GWeatherDistance`, based on the upper speed bound.

```
upperSpeed = hasAboutClause ? (orderedSpeed + seaPerformance.UpperSpeedLimit) : orderedSpeed
MinAllowedTime = GWeatherDistance / upperSpeed   (rounded to 4 dp)
```
If the charter has an "about" clause (e.g. "about 13 knots ± 0.5"), the vessel is allowed to go faster by `UpperSpeedLimit` without penalty. Going faster means it could legitimately take less time.

---

### `MaxAllowedTime`
The **maximum** hours the vessel should have taken, based on the lower speed bound.

```
lowerSpeed = hasAboutClause ? (orderedSpeed - seaPerformance.LowerSpeedLimit) : orderedSpeed
MaxAllowedTime = GWeatherDistance / lowerSpeed   (rounded to 4 dp)
```

---

### `IsInRange`
```
IsInRange = (GWeatherHours >= MinAllowedTime) AND (GWeatherHours <= MaxAllowedTime)
```
If `true` → the vessel performed within the allowed speed band. No time loss/gain.
If `false` → the vessel was either too slow (time loss) or too fast (time gain).

---

### `GWTimeGainLoss`
Only calculated if `IsInRange = false`.

```
If GWeatherHours > MaxAllowedTime:
    GWTimeGainLoss = GWeatherHours - MaxAllowedTime    (vessel was too slow → positive = time lost)
Else:
    GWTimeGainLoss = GWeatherHours - MinAllowedTime    (vessel was too fast → negative = time gained)
```

---

### `PerMileFactor`
The time loss/gain per nautical mile during good weather:
```
PerMileFactor = GWTimeGainLoss / GWeatherDistance   (rounded to 4 dp)
```
Zero if `GWeatherDistance = 0`.

---

### `TotalTimeLossGain`
The final charter party time deviation figure.

**Without extrapolation** (`Terms.HasExtrapolation = false`):
```
TotalTimeLossGain = GWTimeGainLoss
```
Only good weather performance counts.

**With extrapolation** (`Terms.HasExtrapolation = true`):
```
TotalTimeLossGain = PerMileFactor * AllWeatherDistance
```
The per-mile rate from good weather is extrapolated across the entire voyage distance (including bad weather miles), giving a total voyage performance figure.

Only set if `IsInRange = false`. Zero otherwise.

---

## 5. Table: TcV2AtSeaPassageFuelPerformance — Every Column Explained

One row per distinct ordered speed per fuel group per passage.

---

### `TermId`, `PassageID`, `IMONumber`, `VoyageNumber`, `PassageFrom`, `PassageTo`, `DepartureDate`, `ArrivalDate`
Same logic as speed performance — copied from first/last passage reports.

---

### `OrderedSpeed`, `FuelGroup`
The speed and fuel group being evaluated for this row.

---

### `IsUnWarranted`, `IsShortPassage`
Same logic as speed performance.

---

### `AllWeatherDistance`
```
AllWeatherDistance = SUM(passage.Distance)
  for passages at this speed
  that have at least one ConsumptionDetails row matching this fuelGroup
  and Distance > 0
```
Note: only passages that actually have consumption data for this fuel group are included, unlike the speed calc which uses all passages at that speed.

---

### `GWDistance`
Same as above but restricted to good weather passages only (no bad weather condition codes).

---

### `MEWarrantedConsumption`
Taken from `SeaPerformanceDetails.MEWarranty` for the matching speed + load condition row.

**Scrubber adjustment:** If `Terms.IsScrubberEnabled = true` AND `Terms.ScrubberAllowance` is set AND the current fuel group is `"FO"`:
```
MEWarrantedConsumption = MEWarranty + ScrubberAllowance
```
A scrubber increases fuel consumption slightly, so the warranty is adjusted upward for FO.

---

### `AUXWarrantedConsumption`
Taken directly from `SeaPerformanceDetails.AUXWarranty`. No scrubber adjustment.

---

### `OthersWarrantedConsumption`
Taken directly from `SeaPerformanceDetails.OthersWarranty`.

---

### `GWMEConsumption`
Sum of `MEConsumption` from all good weather consumption detail rows at this speed and fuel group:
```
GWMEConsumption = SUM(gwSpeedConsumptions.MEConsumption) where MEConsumption > 0
```
"Good weather consumption rows" = `ConsumptionDetails` whose `FormsId` matches a good weather `PassageDetail`.

---

### `GWAUXConsumption`, `GWOthersConsumption`
Same logic for AUX and Others respectively.

---

### Warranted Consumption Bounds (Min and Max)

The charter party gives a warranted daily consumption rate (e.g. 30 MT/day of FO at 13 knots). To convert this to a total for the actual distance sailed:

```
WarrantedTotalForDistance = (GWDistance / OrderedSpeed) * (WarrantedDaily / 24)
```
Logic: `GWDistance / OrderedSpeed` gives the ideal hours to cover that distance. `WarrantedDaily / 24` gives the hourly rate. Multiplied together → total expected consumption for that distance.

**MEMinWarrantedCons:**
```
minPercent = hasAboutClause ? (1 - ConsumptionLowerLimit / 100) : 1
MEMinWarrantedCons = (GWDistance / OrderedSpeed) * (MEWarrantedConsumption / 24) * minPercent
```

**MEMaxWarrantedCons:**
```
maxPercent = hasAboutClause ? (1 + ConsumptionUpperLimit / 100) : 1
MEMaxWarrantedCons = (GWDistance / OrderedSpeed) * (MEWarrantedConsumption / 24) * maxPercent
```

Same pattern applies for AUX and Others.

---

### `IsMEInRage`, `IsAUXInRage`, `IsOthersInRage`
```
IsMEInRage = GWMEConsumption >= MEMinWarrantedCons AND GWMEConsumption <= MEMaxWarrantedCons
```
True means the vessel consumed within the allowed band. False means over or under consumption.

---

### `IsInRage`
```
IsInRage = IsMEInRage AND IsAUXInRage AND IsOthersInRage
```
Only true if ALL three engine types are within range.

---

### `GWMEFuelLossGain`
Only set if `IsMEInRage = false`:
```
If GWMEConsumption > MEMaxWarrantedCons:
    GWMEFuelLossGain = GWMEConsumption - MEMaxWarrantedCons   (positive = excess = bunker loss for owner)
Else:
    GWMEFuelLossGain = GWMEConsumption - MEMinWarrantedCons   (negative = under-consumption = gain)
```
Same pattern for `GWAUXFuelLossGain` and `GWOthersFuelLossGain`.

---

### `MEPerMileFactor`, `AUXPerMileFactor`, `OthersPerMileFactor`
```
MEPerMileFactor = GWMEFuelLossGain / GWDistance   (0 if GWDistance = 0)
```
This is the fuel deviation per nautical mile during good weather. Used for extrapolation.

---

### `FuelLossGain`
The final charter party fuel deviation figure. Only set if `IsInRage = false`.

**Without extrapolation:**
```
FuelLossGain = GWMEFuelLossGain + GWAUXFuelLossGain + GWOthersFuelLossGain
```

**With extrapolation:**
```
FuelLossGain = (MEPerMileFactor * AllWeatherDistance)
             + (AUXPerMileFactor * AllWeatherDistance)
             + (OthersPerMileFactor * AllWeatherDistance)
```
The per-mile deviation rates from good weather are applied to the total voyage distance.

---

## 6. Table: TcV2AtSeaWeatherDetails — Every Column Explained

One row per hourly weather slot. Written separately from the passage data via `dbo.Job_InsertTCV2AtSeaWeatherData`.

---

### `TermId`, `IMONumber`
Copied from the vessel being processed.

---

### `FormsId`
The report form this weather slot belongs to (from `WeatherData.FormsId`).

---

### `FormDate`
The actual report date (`WeatherData.ReportDateTime`), computed from `FormDateWithOffset.GetUTCDateTime(FormDateOffset)`.

---

### `CalculatedDate`
`WeatherData.AnalyzedWeatherDate` — the exact timestamp of this specific weather observation (different from the report date; each report may have many weather slots within its period).

---

### `Hours`
How long this weather slot lasted:
```
Hours = (thisSlot.AnalyzedWeatherDate - previousSlot.AnalyzedWeatherDate).TotalHours
```
Computed during `ProcessWeatherData`. For the first slot in the sequence, `Hours` is 0.
Slots with `Hours <= 0` or `Hours > 2` are removed before saving (they represent invalid gaps in the weather data).

---

### `WindSpeedInBF`
`WeatherData.AnalyzedWind` after unit conversion. See Section 7 for full conversion logic.

---

### `WaveHeightInDSS`
`WeatherData.AnalyzedWave` after unit conversion.

---

### `CurrentInKTS`
`WeatherData.AnalyzedCurrent` — stored directly as-is in knots.

---

### `Conditions`
The condition code for this slot (e.g. `"WW"`, `"WI"`, `"C"`, or empty string for fine weather).
Set during `ProcessWeatherData`. See Section 8 for the logic.

---

## 7. Analyzed Wind: Full Journey from Raw Data to Final Value

This section traces the wind value from its source in the database all the way to what is stored in `PassageDetail.EventAnalyzedWindForce` and `AnalyzedWeatherDetail.WindSpeedInBF`.

---

### Step 1 — Raw DB Value

The weather provider delivers wind speed in **knots**. In `WeatherData`:
```
AnalyzedWindDB (string) → parsed to decimal → stored as AnalyzedWind (knots, rounded to 4 dp)
```

Separately, the passage report has ship-reported wind:
```
ReportedWindSpeedFromDB (string) → parsed to decimal → PassageData.ReportedWind
```
And ship-filed analysed wind (the weather routing company's summary at report level):
```
AnalysedWindFromDB (string) → parsed to decimal → PassageData.AnalysedWind
```

---

### Step 2 — Unit Conversion (`TransformData`)

This runs once on the entire dataset before any exclusion logic. The charter terms define what unit wind should be in (`WindSpeedUnit`: either `BF` = Beaufort or `kts` = knots).

**On `PassageData` records (report-level wind):**
```
If WindSpeedUnit == BF:
    passage.AnalysedWind = passage.AnalysedWind.ConvertKnotsToBeaufortNumber()
    (reported wind stays in Beaufort as filed by the ship)

If WindSpeedUnit == kts:
    passage.ReportedWind = passage.ReportedWind.ConvertBeaufortNumberToKnots()
    (analysed wind stays in knots as provided)
```

**On `WeatherData` records (hourly weather):**
```
If WindSpeedUnit == BF:
    weather.AnalyzedWind = weather.AnalyzedWind.ConvertKnotsToBeaufortNumber()
    (each hourly wind value converted from knots → Beaufort)
```

After this step, all wind values across both report-level and hourly data are in the charter's preferred unit.

---

### Step 3 — Hourly Weather Pre-processing (`ProcessWeatherData`)

The full list of hourly `WeatherData` for the vessel (all reports) is sorted by `AnalyzedWeatherDate`.

For each slot (index 1 onwards):
```
hours = (slot[i].AnalyzedWeatherDate - slot[i-1].AnalyzedWeatherDate).TotalHours
isWindExcluded = slot[i].AnalyzedWind > windThreshold
isWaveExcluded = slot[i].AnalyzedWave > waveThreshold
isCurrentExcluded = slot[i].AnalyzedCurrent < minCurrentLimit OR > maxCurrentLimit
slot[i].Conditions = combination code (WI / WA / C / WW / WIC / WAC / WWC / None)
```

Slots with `hours <= 0` or `hours > 2` are then removed.

---

### Step 4 — Per-Report Evaluation (`CalculateWeatherDetails`)

For each non-departure `PassageDetail`, the system evaluates whether the wind was bad enough to exclude it.

#### Path A — Reported Weather Source

```
passageDetail.EventAnalyzedWindForce = passageData.AnalysedWind   (unit already converted)
passageDetail.ReportedWindForce = passageData.ReportedWind         (unit already converted)

badDayViolated = IsBadHoursViolated(terms.BadDayHoursType, passageDetail.Hours, passageDetail.Hours, _badDayHours)

isWindExcluded = (ReportedWindForce != null)
              AND (windThreshold > 0)
              AND (ReportedWindForce.Value > windThreshold)
              AND badDayViolated
```

Note: in this path, `EventAnalyzedWindForce` is stored for reference only. The exclusion decision uses `ReportedWindForce`.

#### Path B — Analysed Weather Source

```
passageDetail.EventAnalyzedWindForce = passageData.AnalysedWind   (report-level summary)

weatherSlots = hourly WeatherData where FormsId == this report's FormsId

badTime = SUM(slot.Hours) where slot.Conditions != None
goodTime = SUM(slot.Hours) where slot.Conditions == None

badDayViolated = IsBadHoursViolated(terms.BadDayHoursType, badTime, passageDetail.Hours, _badDayHours)

If badDayViolated:
    exclusionCodes = all distinct Conditions from bad slots
    isWindExcluded = any code in exclusionCodes is a BadWind code (WI, WIC, WWC, WW)
```

---

### Step 5 — Adjusting `EventAnalyzedWindForce` to be consistent

After determining `isWindExcluded` via hourly data (Path B only), the system checks whether the stored `EventAnalyzedWindForce` value is consistent with the decision:

```
If isWindExcluded == true AND EventAnalyzedWindForce <= windThreshold:
    EventAnalyzedWindForce = windThreshold + 1
    (Force the stored value above threshold so any downstream report shows wind WAS bad)

If isWindExcluded == false AND EventAnalyzedWindForce > windThreshold:
    EventAnalyzedWindForce = windThreshold
    (Force the stored value to threshold so downstream report shows wind was NOT bad)
```

This alignment step ensures the stored `EventAnalyzedWindForce` always agrees with the actual exclusion decision even when the hourly analysis and the report-level summary disagree.

---

### Step 6 — What Gets Stored

| Column | Table | Value |
|---|---|---|
| `ReportedWindForce` | PassageDetail | Ship's reported wind, in charter units |
| `EventAnalyzedWindForce` | PassageDetail | Analysed wind summary (report level), in charter units, possibly adjusted in step 5 |
| `WindSpeedInBF` | AnalyzedWeatherDetail | Hourly analysed wind, in charter units (typically BF) |

---

### Bad Day Hours Logic (`IsBadHoursViolated`)

This function determines whether enough bad weather occurred to justify exclusion.

**Type: `Hours` (fixed hours threshold)**
```
return eventHours >= termsBadDayHours
```
If the bad hours in the period are >= the threshold (e.g. >= 12 hours) → bad day.

**Type: `AS_Ratio_Of_24_Hours` (proportional threshold)**
```
termsRatio = termsBadDayHours / 24          (e.g. 12/24 = 0.5 = 50% of a day)
eventRatio = badHours / steamingHours       (fraction of the steaming period that was bad)
return eventRatio > termsRatio
```
If more than 50% of the period was bad → bad day. This is proportional, so it works fairly for short periods (e.g. a 6-hour report needs 3+ bad hours to be excluded, not 12).

---

## 8. The Conditions Column: How Every Code Is Set

The `Conditions` column on `PassageDetail` is a comma-separated string built up in this exact order:

**1. Weather conditions** (from `CalculateWeatherDetails`)
Based on wind/wave/current evaluation. One of: `WI`, `WA`, `C`, `WW`, `WIC`, `WAC`, `WWC`.

**2. Unwarranted speed** (`UW`)
Added if there is no matching `SeaPerformanceDetails` row for this `OrderedSpeed` + `LoadCondition`.
Only checked if `LoadCondition` is not null/empty.

**3. Manual exclusion** (`EXC`)
Added if `PassageData.IsExcluded = true` (the report was manually flagged in the source system).

**4. Event exclusion codes** (e.g. `E1`, `MANOEUVRE`, `DEVIATION`)
Added after `ComputeEventExclusion` runs. These are free-text codes from `ExclusionEventDetail.ConditionCode`. Only added if the event actually changed hours, distance, or fuel.

**5. Short Day** (`SD`)
Added after event exclusion has adjusted `Hours`:
```
If Hours != null AND Hours > 0 AND Hours < MinimumHRsInADay → add SD
```

**6. Short Passage** (`SP`)
```
If (arrival.FormDate - departure.FormDate).TotalHours < MinimumPassageHours → add SP
```
Added to every non-departure report in a short passage.

**Final format example:**
```
"WW,SD"
"WI,E1"
"EXC,DEVIATION"
"WW,WI,SP"
```

---

## 9. The ROB Column: How It Is Built

**Source:** `ROBData` (Result Set 2), `Terms.ActiveFuelTypes`

For each non-departure `PassageDetail`:
1. Get all `ROBData` rows where `FormsId` matches this report.
2. For each fuel type in `Terms.ActiveFuelTypes` (e.g. ["FO", "GO", "VLSFO"]):
   - Find the matching `ROBData` row (case-insensitive fuel type match).
   - If found, take its `FuelROB` value (decimal, rounded to 4 dp).
   - If not found, `ROB` is null for that fuel type.
3. Build a list of `FuelROB` objects: `{ FuelType: "FO", ROB: 245.3 }`.
4. Serialize to JSON and store as a string.

If no ROB data exists for this `FormsId` at all, the `ROB` column is left null.

---

## 10. Fuel Consumption Calculation Deep Dive

This explains exactly how the `MEConsumption`, `AUXConsumption`, `OtherConsumption` values in `ConsumptionDetails` are computed from raw data.

### Step 1 — Fetch all consumption records for the report

```
consumptions = ReportFuelConsumptions WHERE FormsId == thisReport.FormsId
```

### Step 2 — Normalize bio-fuel

```
For each record where FuelGroup == "BioFuel":
    If HasBioFuelFO == true  → change FuelGroup to "FO"
    If HasBioFuelGO == true  → change FuelGroup to "GO"
    If HasBioFuelBF == true  → keep as "BF"
```

### Step 3 — Build one ConsumptionDetails per fuel group

Loop over each `fuelGroup` in `Terms.FuelGroups`:

**For ME consumption:**
```
engineConsumptions = consumptions
  WHERE Category (case-insensitive) matches any ConsumptionCategory with EngineType == "me"
  AND FuelGroup == fuelGroup

MEConsumption = SUM(engineConsumptions.FuelConsumption)
```

**For AUX consumption:**
```
engineConsumptions = consumptions
  WHERE Category matches any ConsumptionCategory with EngineType == "aux"
  AND FuelGroup == fuelGroup

AUXConsumption = SUM(engineConsumptions.FuelConsumption)
```

**For Others consumption:**
Same but EngineType == "others".

### Step 4 — Apply event exclusions

```
For each ExclusionEventDetail matching this FormsId:
    meDeduction = SUM(ExclusionEventsConsumptionDetail.FuelConsumption)
                  WHERE EventId matches AND FuelGroup matches AND EngineType is "me"
                  (values are already stored negative)

    MEConsumption = MEConsumption + meDeduction   (adding a negative = subtracting)
```

Same for AUX and Others.

### Step 5 — Only keep rows with at least one non-zero value

```
If MEConsumption > 0 OR AUXConsumption > 0 OR OtherConsumption > 0:
    Add to consumptionDetails list
```

Rows with all-zero or all-null consumption are discarded.

---

*Document generated from source code of `gp-charterparty-jobs` — TC2.0 AtSea worker.*
*Files referenced: `Passages.cs`, `Speed.cs`, `Fuel.cs`, `WeatherExclusion.cs`, `EventExclusion.cs`, `Repository.cs`, `PassageDetail.cs`, `PassageSpeedPerformance.cs`, `PassageFuelPerformance.cs`, `AnalyzedWeatherDetail.cs`, `DataFromDB.cs`, `Terms.cs`, `Enums.cs`, `Constants.cs`*
