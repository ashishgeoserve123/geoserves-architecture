# BOG Worker Null Crashes and AT SEA Average CP Warranted Speed Fix - Issue Overview

## Part 1: The Problem

### What are the Affected Components?
This fix covers two separate but related components in the TCV2LNG pipeline:

1. **BOG Worker** (`src/BusinessLogic/Workers/TC2.0_LNG/BOG/Worker.cs`) — The BOG worker processes Boil-Off Gas performance by iterating over identified pump logs, computing single passage details, and assembling voyage-level performance summaries. It also handles "ongoing" passages (in-progress voyages not yet completed).

2. **AT SEA VoyageOrderPerformance** (`src/BusinessLogic/Workers/TC2.0_LNG/AtSea/VoyageOrderPerformance.cs`) — This class computes `AverageCPWarrantedSpeed` for each voyage order segment, which is a core input to warranted time and time loss/gain calculations.

### The Data
```
BOG Worker:
- IdentifiedPumpLogs: pump log boundaries used to delimit individual BOG passages
- ReportData: noon/departure/arrival reports for the vessel
- WeatherEventDetails: exclusion events applied per passage
- Terms: BOG contract terms and warranties

AT SEA VoyageOrderPerformance:
- VoyageOrderWithSegments: ordered segments belonging to each voyage order
- Segment.OrderStartDate / VoyageOrderEndTime: the contractual start and end of the voyage order
- ArrivalTimeSET: the actual last good-weather report date within the segment
- HasRevisionInVoyageOrder: whether the voyage order has a revision (amended scheduled arrival)
```

**Affected Vessels**: All LNG vessels processed via the TCV2LNG pipeline, particularly those with:
- Voyage orders where `GetPassageDetails()` returns null or an empty detail list
- Pump log data where no completed passages exist yet (ongoing voyage, `lastReport` is null)
- Voyage orders **without** a revision, where the old code incorrectly used `arrivalTimeSET` instead of `VoyageOrderEndTime` for warranted speed

---

### Bug 1 — Worker.cs: `CalculateSinglePassagePerformance` called on null/empty passage

**File**: `src/BusinessLogic/Workers/TC2.0_LNG/BOG/Worker.cs`, lines 88–91

`GetPassageDetails()` can return `null`, or return a `TcV2BOGSinglePassage` object with a null or empty `BOGSinglePassageDetails` list (when no passage reports fall within the pump log boundary). The code previously called `CalculateSinglePassagePerformance()` unconditionally on whatever `GetPassageDetails()` returned, causing a `NullReferenceException` downstream when the method tried to iterate over or access the empty detail collection.

**Original Problematic Code**:
```csharp
singlePassage = GetPassageDetails(identifiedPumpLog, data.ReportData, data.WeatherEventDetails, terms.ExclusionCriteria, vessel.TermsId, terms);
performance = CalculateSinglePassagePerformance(identifiedPumpLog, singlePassage, terms, vessel);
```

**Fixed Code**:
```csharp
singlePassage = GetPassageDetails(identifiedPumpLog, data.ReportData, data.WeatherEventDetails, terms.ExclusionCriteria, vessel.TermsId, terms);

if (singlePassage == null || singlePassage.BOGSinglePassageDetails == null || singlePassage.BOGSinglePassageDetails.Count == 0)
    continue;

performance = CalculateSinglePassagePerformance(identifiedPumpLog, singlePassage, terms, vessel);
```

---

### Bug 2 — Worker.cs: `NullReferenceException` when `lastReport` is null in ongoing passage check

**File**: `src/BusinessLogic/Workers/TC2.0_LNG/BOG/Worker.cs`, line 138

After iterating all identified pump logs, the worker builds an "ongoing" passage from the last pump log in the database if it hasn't already been accounted for. The check compared `lastReport.FormID` with `lastPumpLog.FormId` to decide whether the last pump log is already covered. However, when no completed passages were found (i.e., `passages` is empty), `lastReport` is `null` — making `lastReport.FormID` throw a `NullReferenceException`.

**Original Problematic Code**:
```csharp
var lastReport = passages.LastOrDefault()?.BOGSinglePassageDetails.OrderBy(s => s.DateTimeInUTC).LastOrDefault();
// ...
if (lastPumpLog != null && lastReport.FormID != lastPumpLog.FormId)
```

**Fixed Code**:
```csharp
if (lastPumpLog != null && (lastReport == null || lastReport.FormID != lastPumpLog.FormId))
```

The null-check on `lastReport` is added with the `||` short-circuit: if `lastReport` is null there are no completed passages, so the last pump log is always considered "not yet accounted for" and the ongoing passage is built.

---

### Bug 3 — Worker.cs: `CalculateOngoingSinglePassagePerformance` called before null-check

**File**: `src/BusinessLogic/Workers/TC2.0_LNG/BOG/Worker.cs`, lines 160–167

After building the ongoing passage details, the worker checked whether `ongoingPassage` had valid detail records before adding it to the results. However, `CalculateOngoingSinglePassagePerformance()` was called **before** this null-check. If `ongoingPassage` was null or had no details, calling the calculation function first could throw or produce a meaningless `ongoingPerformance` result that was never used.

**Original Problematic Code**:
```csharp
ongoingPassage = GetOngoingPassageDetails(ongoingLog, data.ReportData, data.WeatherEventDetails, terms.ExclusionCriteria, vessel.TermsId, terms);
ongoingPerformance = CalculateOngoingSinglePassagePerformance(ongoingLog, ongoingPassage, terms, vessel);

if (ongoingPassage != null && ongoingPassage.BOGSinglePassageDetails != null && ongoingPassage.BOGSinglePassageDetails.Count > 0)
{
    passages.Add(ongoingPassage);
    voyagePassagePerformces.Add(ongoingPerformance);
}
```

**Fixed Code**:
```csharp
ongoingPassage = GetOngoingPassageDetails(ongoingLog, data.ReportData, data.WeatherEventDetails, terms.ExclusionCriteria, vessel.TermsId, terms);

if (ongoingPassage != null && ongoingPassage.BOGSinglePassageDetails != null && ongoingPassage.BOGSinglePassageDetails.Count > 0)
{
    ongoingPerformance = CalculateOngoingSinglePassagePerformance(ongoingLog, ongoingPassage, terms, vessel);
    passages.Add(ongoingPassage);
    voyagePassagePerformces.Add(ongoingPerformance);
}
```

---

### Bug 4 — VoyageOrderPerformance.cs: Wrong time denominator for `AverageCPWarrantedSpeed` on non-revised voyage orders

**File**: `src/BusinessLogic/Workers/TC2.0_LNG/AtSea/VoyageOrderPerformance.cs`, lines 80–95

`AverageCPWarrantedSpeed` is calculated as the total segment distance divided by the time between the order start date and a reference end time. The reference end time depends on context:

- **Revised voyage orders** (`HasRevisionInVoyageOrder == true`): the warranted speed is contractually evaluated up to the actual arrival SET time (`arrivalTimeSET`), not the scheduled end.
- **Non-revised voyage orders**: the warranted speed should use the full contractual voyage order end time (`VoyageOrderEndTime`).

Previously, `totalHoursInBetweenStartTimeAndSET` (derived from `arrivalTimeSET`) was computed outside both branches and used as the denominator unconditionally, meaning **non-revised orders also used the actual SET time** instead of the contractual end time. This caused incorrect warranted speeds for non-revised orders when the actual SET differed from the scheduled voyage order end.

**Original Problematic Code**:
```csharp
decimal totalHoursInBetweenStartTimeAndSET = (decimal)((DateTime)arrivalTimeSET - performance.OrderStartDate).TotalHours;
decimal finalCalculatedSegment = segmentDistance;

if (segment.VoyageOrder.HasRevisionInVoyageOrder)
{
    finalCalculatedSegment = finalCalculatedSegment * (totalHoursInBetweenStartTimeAndSET / 24);
}
// Applied unconditionally to both revised and non-revised orders
performance.AverageCPWarrantedSpeed = totalHoursInBetweenStartTimeAndSET == 0 ? 0 : finalCalculatedSegment / totalHoursInBetweenStartTimeAndSET;
```

**Fixed Code**:
```csharp
decimal finalCalculatedSegment = segmentDistance;
decimal totalHoursInBetweenStartAndArrival = (decimal)(performance.VoyageOrderEndTime - performance.OrderStartDate).TotalHours;

if (segment.VoyageOrder.HasRevisionInVoyageOrder)
{
    decimal totalHoursInBetweenStartTimeAndSET = (decimal)((DateTime)arrivalTimeSET - performance.OrderStartDate).TotalHours;
    // Revised: use actual SET arrival time as denominator
    performance.AverageCPWarrantedSpeed = totalHoursInBetweenStartTimeAndSET == 0 ? 0 : finalCalculatedSegment / totalHoursInBetweenStartTimeAndSET;
}
else
{
    // Non-revised: use the full contractual voyage order end time as denominator
    performance.AverageCPWarrantedSpeed = totalHoursInBetweenStartAndArrival == 0 ? 0 : segmentDistance / totalHoursInBetweenStartAndArrival;
}
```

### Data Flow Chain — BOG Worker Crashes
```
RunWorkerAsync()
    ↓
foreach identifiedPumpLog
    ↓
GetPassageDetails()  →  returns null or empty BOGSinglePassageDetails
    ↓
[BUG 1] CalculateSinglePassagePerformance(null/empty singlePassage)
    ↓
❌ NullReferenceException inside performance calculation
```

```
passages = []  (all pump logs returned null/empty passage details)
    ↓
lastReport = passages.LastOrDefault()?.BOGSinglePassageDetails...  →  null
    ↓
[BUG 2] if (lastPumpLog != null && lastReport.FormID != ...)
    ↓
❌ NullReferenceException: Object reference not set to an instance of an object
```

### Data Flow Chain — AT SEA Incorrect Warranted Speed
```
ComputePerformance()
    ↓
HasRevisionInVoyageOrder == false
    ↓
[BUG 4] arrivalTimeSET used as denominator instead of VoyageOrderEndTime
    ↓
❌ AverageCPWarrantedSpeed computed against wrong time window
    ↓
❌ WarrantedTime, TimeLossGain, AllWeatherOverUnder all derived incorrectly
```

---

## Part 2: The Solution

### Solution Overview
Fix all four bugs independently:
1. **Bug 1**: Add a null/empty guard after `GetPassageDetails()` — skip the iteration with `continue` if no usable passage was produced.
2. **Bug 2**: Guard the `lastReport.FormID` access with a null check using `||` short-circuit logic.
3. **Bug 3**: Move `CalculateOngoingSinglePassagePerformance()` inside the null-check block so it is only invoked when `ongoingPassage` is confirmed to have valid details.
4. **Bug 4**: Split the `AverageCPWarrantedSpeed` formula into two distinct branches — revised and non-revised — using the correct time denominator for each case.

### Code Changes Made

**Worker.cs — Bug 1: Null guard after GetPassageDetails** (`src/BusinessLogic/Workers/TC2.0_LNG/BOG/Worker.cs`, lines 88–91):
```csharp
// BEFORE
singlePassage = GetPassageDetails(...);
performance = CalculateSinglePassagePerformance(identifiedPumpLog, singlePassage, terms, vessel);

// AFTER
singlePassage = GetPassageDetails(...);

if (singlePassage == null || singlePassage.BOGSinglePassageDetails == null || singlePassage.BOGSinglePassageDetails.Count == 0)
    continue;

performance = CalculateSinglePassagePerformance(identifiedPumpLog, singlePassage, terms, vessel);
```

**Worker.cs — Bug 2: Null-safe lastReport check** (`src/BusinessLogic/Workers/TC2.0_LNG/BOG/Worker.cs`, line 138):
```csharp
// BEFORE
if (lastPumpLog != null && lastReport.FormID != lastPumpLog.FormId)

// AFTER
if (lastPumpLog != null && (lastReport == null || lastReport.FormID != lastPumpLog.FormId))
```

**Worker.cs — Bug 3: Move CalculateOngoingSinglePassagePerformance inside null guard** (`src/BusinessLogic/Workers/TC2.0_LNG/BOG/Worker.cs`, lines 160–167):
```csharp
// BEFORE
ongoingPerformance = CalculateOngoingSinglePassagePerformance(ongoingLog, ongoingPassage, terms, vessel);

if (ongoingPassage != null && ongoingPassage.BOGSinglePassageDetails != null && ongoingPassage.BOGSinglePassageDetails.Count > 0)
{
    passages.Add(ongoingPassage);
    voyagePassagePerformces.Add(ongoingPerformance);
}

// AFTER
if (ongoingPassage != null && ongoingPassage.BOGSinglePassageDetails != null && ongoingPassage.BOGSinglePassageDetails.Count > 0)
{
    ongoingPerformance = CalculateOngoingSinglePassagePerformance(ongoingLog, ongoingPassage, terms, vessel);
    passages.Add(ongoingPassage);
    voyagePassagePerformces.Add(ongoingPerformance);
}
```

**VoyageOrderPerformance.cs — Bug 4: Correct AverageCPWarrantedSpeed denominator per revision status** (`src/BusinessLogic/Workers/TC2.0_LNG/AtSea/VoyageOrderPerformance.cs`, lines 79–95):
```csharp
// BEFORE
decimal totalHoursInBetweenStartTimeAndSET = (decimal)((DateTime)arrivalTimeSET - performance.OrderStartDate).TotalHours;
decimal finalCalculatedSegment = segmentDistance;
if (segment.VoyageOrder.HasRevisionInVoyageOrder)
{
    finalCalculatedSegment = finalCalculatedSegment * (totalHoursInBetweenStartTimeAndSET / 24);
}
performance.AverageCPWarrantedSpeed = totalHoursInBetweenStartTimeAndSET == 0 ? 0 : finalCalculatedSegment / totalHoursInBetweenStartTimeAndSET;

// AFTER
decimal finalCalculatedSegment = segmentDistance;
decimal totalHoursInBetweenStartAndArrival = (decimal)(performance.VoyageOrderEndTime - performance.OrderStartDate).TotalHours;
if (segment.VoyageOrder.HasRevisionInVoyageOrder)
{
    decimal totalHoursInBetweenStartTimeAndSET = (decimal)((DateTime)arrivalTimeSET - performance.OrderStartDate).TotalHours;
    performance.AverageCPWarrantedSpeed = totalHoursInBetweenStartTimeAndSET == 0 ? 0 : finalCalculatedSegment / totalHoursInBetweenStartTimeAndSET;
}
else
{
    performance.AverageCPWarrantedSpeed = totalHoursInBetweenStartAndArrival == 0 ? 0 : segmentDistance / totalHoursInBetweenStartAndArrival;
}
```

### Result After Fix

**BOG Worker Data Flow (Fixed)**:
```
foreach identifiedPumpLog
    ↓
GetPassageDetails()  →  null or empty
    ↓
Null guard: continue  (skip to next pump log, no crash)
    ↓
✅ Worker continues safely for all remaining pump logs
```

```
passages = []
    ↓
lastReport = null
    ↓
(lastReport == null || ...)  →  true  (short-circuit, no NullReferenceException)
    ↓
✅ Ongoing passage detection runs correctly
```

**AT SEA Warranted Speed (Fixed)**:
```
HasRevisionInVoyageOrder == false
    ↓
totalHoursInBetweenStartAndArrival = VoyageOrderEndTime - OrderStartDate
    ↓
AverageCPWarrantedSpeed = segmentDistance / totalHoursInBetweenStartAndArrival
    ↓
✅ Warranted time and loss/gain computed against correct contractual window
```

### Summary of Changes

| Aspect | Before | After |
|--------|--------|-------|
| Null passage from GetPassageDetails | Passed to CalculateSinglePassagePerformance — crash | Guarded with `continue` — skipped safely |
| lastReport null when no passages exist | `lastReport.FormID` — NullReferenceException | `(lastReport == null \|\| ...)` — null-safe |
| CalculateOngoingSinglePassagePerformance call order | Called before null-check on ongoingPassage | Called inside null-check, only when valid data exists |
| AverageCPWarrantedSpeed for non-revised orders | Used `arrivalTimeSET` denominator (wrong) | Uses `VoyageOrderEndTime` denominator (correct) |
| AverageCPWarrantedSpeed for revised orders | Used `arrivalTimeSET` (correct but shared with non-revised) | Uses `arrivalTimeSET` (correct, now isolated to revised branch) |

---

## Files Modified

1. **src/BusinessLogic/Workers/TC2.0_LNG/BOG/Worker.cs**
   - Added null/empty guard after `GetPassageDetails()` call with `continue` to skip invalid passages
   - Fixed `lastReport` null-dereference in the ongoing pump log check with `(lastReport == null || ...)` guard
   - Moved `CalculateOngoingSinglePassagePerformance()` inside the `ongoingPassage` null-check block

2. **src/BusinessLogic/Workers/TC2.0_LNG/AtSea/VoyageOrderPerformance.cs**
   - Split `AverageCPWarrantedSpeed` calculation into two branches based on `HasRevisionInVoyageOrder`
   - Revised orders: continue using `arrivalTimeSET` as the time denominator
   - Non-revised orders: use `VoyageOrderEndTime` as the time denominator (corrected)

---

## Commit Information

**Commit Hash**: N/A
**Date**: 2026-03-17
**Message**: `Fix BOG worker null crashes and correct AverageCPWarrantedSpeed denominator for non-revised voyage orders`

---

## Conclusion

Four related bugs were found and fixed across two files in the TCV2LNG pipeline:

1. **BOG Worker — Null passage crash (Worker.cs)**: `GetPassageDetails()` can return null or an empty passage, but the code immediately passed the result to `CalculateSinglePassagePerformance()` without checking. Fixed by adding a guard with `continue` to skip the iteration when no passage data is available.

2. **BOG Worker — NullReferenceException on lastReport (Worker.cs)**: The ongoing passage detection compared `lastReport.FormID` to the last pump log's form ID without checking if `lastReport` itself was null. This occurred when no completed passages existed. Fixed by making the condition null-safe: `(lastReport == null || lastReport.FormID != lastPumpLog.FormId)`.

3. **BOG Worker — Premature performance calculation for ongoing passage (Worker.cs)**: `CalculateOngoingSinglePassagePerformance()` was called before the null-check on `ongoingPassage`, risking a crash if no ongoing passage was found. Fixed by moving the call inside the null-check block so it is only invoked when the passage has valid data.

4. **AT SEA Warranted Speed — Wrong denominator for non-revised orders (VoyageOrderPerformance.cs)**: The `AverageCPWarrantedSpeed` calculation used the actual SET arrival time as its time denominator for all voyage orders, even non-revised ones where the warranted speed should be based on the full contractual voyage order window (`VoyageOrderEndTime`). Fixed by splitting the formula into two branches: revised orders use `arrivalTimeSET`, non-revised orders use `VoyageOrderEndTime`.
