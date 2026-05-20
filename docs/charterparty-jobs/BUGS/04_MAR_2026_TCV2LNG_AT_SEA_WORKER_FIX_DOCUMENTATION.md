# TCV2LNG AT SEA Worker Failures - Issue Overview

## Part 1: The Problem

### What is the TCV2LNG AT SEA Worker?
The TCV2LNG AT SEA worker is responsible for computing sea performance metrics on LNG vessels operating under TC v2.0 contracts. It processes passage data retrieved from the database, identifies voyage orders, assembles voyage segments, computes speed and fuel performance, and inserts the results into SQL Server. The worker runs as part of a five-worker pipeline (BOG, AT SEA, In Port, Pumping, IGS) triggered via the `TCV2LNGController` POST endpoint.

### The Data
When processing AT SEA performance for a vessel, the system works with the following data:

```
- VoyageOrderDetails: voyage order records with form dates and voyage numbers
- Passages: individual passage records with position, report type, and timestamps
- PumpLogDetails: LNG quantity logs at voyage boundaries (for BOG calculation)
- Terms: contractual performance warranties and exclusion criteria
- ExclusionCriteria: weather/event-based exclusion rules
```

**Test Case**: IMO Number 9682576 with date range 2024-01-01 to 2026-03-04.

### The Error
```
System.InvalidOperationException: Sequence contains no matching element
```

This error occurred at:
- **File**: `src/BusinessLogic/Workers/TC2.0_LNG/AtSea/Worker.cs`, line 160
- **Propagation**: Thrown deep inside `GetBOGDetail()` in `Voyages.cs`, re-thrown from the inner vessel loop, caught and logged by the outer try/catch in `Worker.RunWorkerAsync()`
- **Surface Response**: `"Sea Performance failed to process."` in the API response message

### Root Cause Analysis

Three bugs were identified and fixed in this session. The third was the one causing the live `InvalidOperationException`.

---

**Bug 1 — Repository.cs: Wrong DataTable used as guard condition**

**File**: `src/Repositories/TC2.0_LNG/AtSea/Repository.cs`, line 146

The guard condition that controls whether `VoyageOrderDetails` are loaded was checking the row count of the wrong DataTable (`dtReportCons` instead of `dtVoyageOrder`). This caused `VoyageOrderDetails` to never be populated, so the entire AT SEA computation produced no results.

**Original Problematic Code**:
```csharp
// BROKEN — used dtReportCons instead of dtVoyageOrder
if (dtVoyageOrder != null && dtReportCons.Rows.Count > 0)
{
    dataFromDB.VoyageOrderDetails = dtVoyageOrder.DataTableToList<VoyageOrderDetail>();
}
```

**Fixed Code**:
```csharp
// FIXED — checks the correct DataTable
if (dtVoyageOrder != null && dtVoyageOrder.Rows.Count > 0)
{
    dataFromDB.VoyageOrderDetails = dtVoyageOrder.DataTableToList<VoyageOrderDetail>();
}
```

---

**Bug 2 — Passages.cs: Direct mutation of shared PassageData object in the "At SEA" branch**

**File**: `src/BusinessLogic/Workers/TC2.0_LNG/AtSea/Passages.cs`, lines 117–141

In the "At SEA" detection branch, the code was directly mutating the `PortName` of the last `PassageData` object from the `passageFromDB` list (a shared reference). This permanently corrupted the object in the list, causing incorrect passage detection in subsequent iterations.

**Original Problematic Code**:
```csharp
// BROKEN — mutated the live object inside passageFromDB list
arrivalReport = passageFromDB.Last();
arrivalReport.PortName = "At SEA";
```

**Fixed Code**:
```csharp
// FIXED — creates a defensive copy, does not mutate the shared list
PassageData lastPassage = passageFromDB.Last();
arrivalReport = new PassageData
{
    FormsId = lastPassage.FormsId,
    PortName = "At SEA",
    FormDate = lastPassage.FormDate,
    FormDateInUTC = lastPassage.FormDateInUTC,
    FormDateOffset = lastPassage.FormDateOffset,
    IsExcluded = lastPassage.IsExcluded,
    IsFormReportExcluded = lastPassage.IsExcluded
};
```

---

**Bug 3 — Voyages.cs: Unsafe `.Last()` and `.First()` LINQ fallbacks in `GetBOGDetail()`**

**File**: `src/BusinessLogic/Workers/TC2.0_LNG/AtSea/Voyages.cs`, lines 250–258

The `GetBOGDetail()` method selects pump log boundaries for a voyage using `.FirstOrDefault()` with a fallback. The fallback used `.Last(predicate)` and `.First(predicate)` — the non-`OrDefault` LINQ variants — which throw `InvalidOperationException: Sequence contains no matching element` when no element satisfies the predicate. This was the direct cause of the live crash.

**Original Problematic Code**:
```csharp
// BROKEN — .Last() and .First() throw if no element matches the predicate
var pumpStartLog = startLogQuery.OrderByDescending(p => p.ReportDateTime).FirstOrDefault()
    ?? pumpLogDetails.Last(p => p.ReportDateTime <= voyageOrder.FormDate);

var pumpEndLog = pumpLogDetails
    .Where(p => p.ReportDateTime >= voyageOrder.FormDate && p.ReportDateTime <= nextVoyageOrder.FormDate)
    .OrderByDescending(p => p.ReportDateTime)
    .FirstOrDefault()
    ?? pumpLogDetails.First(p => p.ReportDateTime >= voyageOrder.FormDate);
```

Additionally, no null guard existed after these lookups, meaning subsequent code would NullReferenceException if both queries returned nothing.

### Data Flow Chain
```
GetBOGDetail()
    ↓
startLogQuery.FirstOrDefault()  →  null (no pump log in previous→current range)
    ↓
Fallback: pumpLogDetails.Last(p => p.ReportDateTime <= voyageOrder.FormDate)
    ↓
❌ InvalidOperationException: Sequence contains no matching element
    (no pump log found before voyageOrder.FormDate)
    ↓
Worker inner catch: throw ex  (re-throws, loses original stack trace)
    ↓
Worker outer catch: _logger.Error(...)  +  throw
    ↓
TCV2LNGService catch: swallows exception, appends "Sea Performance failed to process."
```

---

## Part 2: The Solution

### Solution Overview
Fix all three bugs independently:
1. **Bug 1**: Correct the guard condition DataTable reference in `Repository.cs`
2. **Bug 2**: Replace the direct object mutation with a defensive copy in `Passages.cs`
3. **Bug 3**: Replace unsafe `.Last(predicate)` / `.First(predicate)` fallbacks with safe `.LastOrDefault()` / `.FirstOrDefault()` equivalents in `Voyages.cs`, and add a null guard to gracefully return an empty `VoyageOrderBOGDetail` when no pump log boundary is found

### The Fix Implementation

**Bug 3 — Fixed Code** (`Voyages.cs`, `GetBOGDetail()`):
```csharp
// FIXED — safe fallbacks that return null instead of throwing
var pumpStartLog = startLogQuery.OrderByDescending(p => p.ReportDateTime).FirstOrDefault()
    ?? pumpLogDetails.OrderByDescending(p => p.ReportDateTime).FirstOrDefault(p => p.ReportDateTime <= voyageOrder.FormDate);

var pumpEndLog = pumpLogDetails
    .Where(p => p.ReportDateTime >= voyageOrder.FormDate && p.ReportDateTime <= nextVoyageOrder.FormDate)
    .OrderByDescending(p => p.ReportDateTime)
    .FirstOrDefault()
    ?? pumpLogDetails.OrderBy(p => p.ReportDateTime).FirstOrDefault(p => p.ReportDateTime >= voyageOrder.FormDate);

// ADDED — null guard: if either boundary log is missing, return empty detail
if (pumpStartLog == null || pumpEndLog == null)
    return detail;
```

### Key Constraint Logic

**Why `.Last(predicate)` and `.First(predicate)` are dangerous**:
- These are LINQ methods that throw `InvalidOperationException` when the sequence has no matching element
- Their `OrDefault` counterparts (`.LastOrDefault()`, `.FirstOrDefault()`) return `null` instead of throwing
- The fallback expressions were intended as a "last resort" safety net, but were themselves unsafe
- The fix converts them to `OrDefault` variants and handles the null case explicitly

**Why a null guard was also needed**:
- Even with safe fallbacks, it is possible that neither the primary query nor the fallback finds a pump log (e.g., no pump log data exists at all for the voyage boundary dates)
- Without the null guard, the code proceeds to dereference `pumpStartLog.DateTimeBeforeLoading` etc., which would cause a `NullReferenceException`
- The null guard returns an empty `VoyageOrderBOGDetail` — the same result as if no BOG data was computable — allowing the rest of the AT SEA workflow to continue

### Code Changes Made

**1. Repository.cs — Wrong DataTable guard** (`src/Repositories/TC2.0_LNG/AtSea/Repository.cs`, line 146):
```csharp
// BEFORE
if (dtVoyageOrder != null && dtReportCons.Rows.Count > 0)

// AFTER
if (dtVoyageOrder != null && dtVoyageOrder.Rows.Count > 0)
```

**2. Passages.cs — Defensive copy for "At SEA" arrival report** (`src/BusinessLogic/Workers/TC2.0_LNG/AtSea/Passages.cs`, lines 117–141):
```csharp
// BEFORE
arrivalReport = passageFromDB.Last();
arrivalReport.PortName = "At SEA";

// AFTER
PassageData lastPassage = passageFromDB.Last();
arrivalReport = new PassageData
{
    FormsId = lastPassage.FormsId,
    PortName = "At SEA",
    FormDate = lastPassage.FormDate,
    FormDateInUTC = lastPassage.FormDateInUTC,
    FormDateOffset = lastPassage.FormDateOffset,
    IsExcluded = lastPassage.IsExcluded,
    IsFormReportExcluded = lastPassage.IsExcluded
};
```

**3. Voyages.cs — Safe LINQ fallbacks + null guard** (`src/BusinessLogic/Workers/TC2.0_LNG/AtSea/Voyages.cs`, lines 250–263):
```csharp
// BEFORE
?? pumpLogDetails.Last(p => p.ReportDateTime <= voyageOrder.FormDate);
...
?? pumpLogDetails.First(p => p.ReportDateTime >= voyageOrder.FormDate);
// (no null guard)

// AFTER
?? pumpLogDetails.OrderByDescending(p => p.ReportDateTime).FirstOrDefault(p => p.ReportDateTime <= voyageOrder.FormDate);
...
?? pumpLogDetails.OrderBy(p => p.ReportDateTime).FirstOrDefault(p => p.ReportDateTime >= voyageOrder.FormDate);

if (pumpStartLog == null || pumpEndLog == null)
    return detail;
```

### Result After Fix

**Data Flow Chain (Fixed)**:
```
GetBOGDetail()
    ↓
startLogQuery.FirstOrDefault()  →  null
    ↓
Fallback: pumpLogDetails.OrderByDescending(...).FirstOrDefault(predicate)
    →  returns null safely (no exception)
    ↓
Null guard: if (pumpStartLog == null || pumpEndLog == null) return detail
    ↓
✅ Empty VoyageOrderBOGDetail returned — AT SEA worker continues normally
    ↓
✅ Sea Performance processed successfully
```

### Test Results

**Before Fix**:
```json
{
  "success": true,
  "message": "BOG Performance processed sucessfuly.\nSea Performance failed to process.\nPort Performance processed sucessfuly.\nPumping Performance processed sucessfuly.\nIGS Performance processed sucessfuly.\n"
}
```

**After Fix**:
```json
{
  "success": true,
  "message": "BOG Performance processed sucessfuly.\nSea Performance processed sucessfuly.\nPort Performance processed sucessfuly.\nPumping Performance processed sucessfuly.\nIGS Performance processed sucessfuly.\n"
}
```

**Verified via**: Live API call to `http://localhost:5232/api/lng-job/TCV2LNG` with IMO `9682576`, date range `2024-01-01` to `2026-03-04`.

### Summary of Changes

| Aspect | Before | After |
|--------|--------|-------|
| VoyageOrderDetails guard | Checked `dtReportCons.Rows.Count` | Checks `dtVoyageOrder.Rows.Count` |
| "At SEA" arrival report | Mutated shared `PassageData` object | Creates a defensive copy |
| Pump log start fallback | `.Last(predicate)` — throws on no match | `.FirstOrDefault(predicate)` — returns null |
| Pump log end fallback | `.First(predicate)` — throws on no match | `.FirstOrDefault(predicate)` — returns null |
| Null boundary handling | None — NullReferenceException risk | Null guard returns empty `VoyageOrderBOGDetail` |
| AT SEA Result | ❌ Sea Performance failed to process | ✅ Sea Performance processed successfully |

---

## Files Modified

1. **src/Repositories/TC2.0_LNG/AtSea/Repository.cs**
   - Corrected the guard condition from `dtReportCons.Rows.Count > 0` to `dtVoyageOrder.Rows.Count > 0`
   - Ensures `VoyageOrderDetails` are correctly populated from the database result

2. **src/BusinessLogic/Workers/TC2.0_LNG/AtSea/Passages.cs**
   - Replaced direct mutation of the last `PassageData` object with a defensive copy
   - Prevents corruption of the shared `passageFromDB` list during "At SEA" passage detection

3. **src/BusinessLogic/Workers/TC2.0_LNG/AtSea/Voyages.cs**
   - Replaced unsafe `.Last(predicate)` and `.First(predicate)` LINQ fallbacks with safe `.FirstOrDefault(predicate)` equivalents in `GetBOGDetail()`
   - Added null guard to return an empty `VoyageOrderBOGDetail` when pump log boundaries cannot be found

---

## Commit Information

**Commit Hash**: N/A  
**Date**: 2026-03-04  
**Message**: `Fix TCV2LNG AT SEA worker failures: wrong DataTable guard, PassageData mutation, and unsafe LINQ fallbacks`

---

## Conclusion

The TCV2LNG AT SEA worker was failing due to three bugs in the data processing pipeline:

1. **Wrong DataTable Guard (Repository.cs)**: `VoyageOrderDetails` were never loaded because the guard condition checked the row count of `dtReportCons` instead of `dtVoyageOrder`. Fixed by correcting the DataTable reference.

2. **PassageData Mutation (Passages.cs)**: The "At SEA" branch was directly mutating a shared `PassageData` object's `PortName`, corrupting the source list and causing incorrect passage detection. Fixed by creating a new `PassageData` object with all required fields copied over.

3. **Unsafe LINQ Fallbacks (Voyages.cs)**: The `GetBOGDetail()` method used `.Last(predicate)` and `.First(predicate)` as fallback pump log lookups, which throw `InvalidOperationException` when no matching element exists. Fixed by switching to the safe `.FirstOrDefault(predicate)` variants and adding a null guard that gracefully returns an empty result rather than crashing the entire worker.
