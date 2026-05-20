# TCV2 AtSea — Why Feb 9–19 Has No Exclusion But Analyzed Weather Data Is Present

## IMO 9722182 (STI Gratitude) — February 2025

---

## The Question

For the period **Feb 9–19 2025**, the passage rows in `TcV2AtSeaPassageDetails` show non-null values for `EventAnalyzedWindForce`, `EventAnalyzedWave`, and `EffectiveCurrent` ("analyzed weather data is present"), yet no weather exclusion condition (`WI`, `C`, `WIC`, etc.) is set. Why?

---

## Root Cause: Two Different Data Sources

The "analyzed weather data" visible in the passage record and the data that drives the exclusion decision come from **completely separate sources**.

### Source 1 — Passage-level display fields (what you see)

The SP `dbo.Job_GetTCV2AtSeaData` pulls a passage-level summary from `AnalyzedWeather` where `Is24Hour = 1`. These values are loaded into `PassageData` and then written unconditionally to `PassageDetail`:

```csharp
// WeatherExclusion.cs:239-243
passageForm.EventAnalyzedWindForce = reportData.AnalysedWind;   // ← always set
passageForm.EventAnalyzedWave      = reportData.AnalysedWave;   // ← always set
passageForm.EffectiveCurrent       = reportData.EffectiveCurrent; // ← always set
```

These are the values you see in `TcV2AtSeaPassageDetails` columns `EventAnalyzedWindForce`, `EventAnalyzedWave`, `EffectiveCurrent`. They are populated **regardless of whether any exclusion logic ran**.

### Source 2 — Hourly slots used for exclusion (what drives the decision)

The exclusion engine uses `dataFromDB.Weathers` — the **hourly `Is24Hour = 0` rows** — filtered and matched per FormsId:

```csharp
// Passages.cs:274
weatherData = dataFromDB.Weathers?.Where(w => w.FormsId == passage.FormsId).ToList()
              ?? new List<WeatherData>();

// WeatherExclusion.cs:248
if (weatherData != null && weatherData.Count > 0
    && weatherData.Any(w => w.Conditions != ExclusionConditions.None))
{
    // bad-hour ratio check → sets isWindExcluded / isWaveExcluded / isCurrentExcluded
}
```

If `weatherData` is **empty** or **all slots have `Conditions = None`**, the outer `if` is never entered, all flags remain `false`, and no exclusion code is written.

---

## Data Flow Diagram

```
AnalyzedWeather table (DB)
│
├── Is24Hour = 1  (daily summary per FormId)
│     └──→ PassageData.AnalysedWind / AnalysedWave / EffectiveCurrent
│              └──→ passageForm.EventAnalyzedWindForce  ← ALWAYS stored in PassageDetails
│                   passageForm.EventAnalyzedWave        ← ALWAYS stored
│                   passageForm.EffectiveCurrent         ← ALWAYS stored
│
└── Is24Hour = 0  (hourly rows per FormId)
      └──→ dataFromDB.Weathers
               └──→ ProcessWeatherData() → computes Hours, tags Conditions per slot
               └──→ RemoveAll(Hours <= 0 OR Hours > 2)   ← filter step
               └──→ IdentifyAnalyzedWeather()
                        if empty  → Gate 11 → NO exclusion
                        if all None → Gate 12 → NO exclusion
                                         ↑
                              FOR FEB 9–19: one of these two
```

---

## Why It Happens for Feb 9–19 — Three Possible Scenarios

### Scenario A — Provider did not submit hourly rows (most likely)

The weather provider submitted a `Is24Hour = 1` daily summary for each day (which populates the display fields) but **did not deliver the `Is24Hour = 0` hourly breakdown** for those FormsIds. Result: `weatherData` is empty per FormsId → **Gate 11** fires.

This matches the existing documentation note:
> `08–19-Feb | Bad Current Hrs = 0 | NULL (no analyzed weather loaded for these forms)`

### Scenario B — Hourly rows were filtered out by the Hours validator

Hourly rows existed but consecutive timestamp gaps exceeded 2 hours (e.g. 3-hourly or 6-hourly provider delivery):

```csharp
// Passages.cs:65
dataFromDB.Weathers.RemoveAll(w => w.Hours <= 0 || w.Hours > 2);
```

After this filter, all hourly rows for those FormsIds were removed, leaving `weatherData` empty → **Gate 11** fires.

### Scenario C — FormsId mismatch

The hourly weather rows in `AnalyzedWeather` are linked to a different FormsId than the passage record uses. The `.Where(w => w.FormsId == passage.FormsId)` filter returns empty even though rows exist in the DB.

### Scenario D — All hourly slots have Conditions = None

Hourly rows exist and pass the Hours filter, but every slot's wind was ≤ BF 4, current was within 0–5 kts, and wave was not configured. All slots get `Conditions = None` → the `.Any(w => w.Conditions != None)` check in `IdentifyAnalyzedWeather` is false → **Gate 12** fires. This would mean the weather was genuinely fine during those hours.

---

## How to Confirm Which Scenario Applies

Run this against the source `AnalyzedWeather` table for the Feb 9–19 FormsIds:

```sql
SELECT
    FormsId,
    Is24Hour,
    COUNT(*)          AS RowCount,
    MIN(CalculatedTimeStamp) AS EarliestSlot,
    MAX(CalculatedTimeStamp) AS LatestSlot
FROM AnalyzedWeather
WHERE FormsId IN (/* Feb 9–19 FormsIds from TcV2AtSeaPassageDetails */)
GROUP BY FormsId, Is24Hour
ORDER BY FormsId, Is24Hour;
```

| Result | Diagnosis |
|---|---|
| Only `Is24Hour = 1` rows, no `Is24Hour = 0` | **Scenario A** — provider gap, no hourly data |
| `Is24Hour = 0` rows present with gaps > 2 hrs between consecutive timestamps | **Scenario B** — Hours filter removed them |
| `Is24Hour = 0` rows present but under a different FormsId | **Scenario C** — FormsId mismatch |
| `Is24Hour = 0` rows present, gaps ≤ 2 hrs, all currents 0–5kts, wind ≤ BF4 | **Scenario D** — genuine fine weather |

---

## Key Takeaways

1. **`EventAnalyzedWindForce`, `EventAnalyzedWave`, and `EffectiveCurrent` in `TcV2AtSeaPassageDetails` are NOT what drives exclusion.** They come from the 24-hour summary row (`Is24Hour = 1`) and are stored unconditionally. A passage can show non-null weather values and still have no exclusion condition.

2. **Exclusion requires hourly `Is24Hour = 0` rows** for the matching FormsId, with at least one slot tagged with a bad condition, AND the bad-hour ratio must exceed `8/24 = 0.333`.

3. **The absence of exclusion for Feb 9–19 is a silent outcome.** No warning or flag is written to indicate that the hourly data was missing or filtered. The passage appears the same as a day with genuinely fine weather.

---

## Code References

| What | File | Line |
|---|---|---|
| Unconditional assignment of display fields | `WeatherExclusion.cs` | 239–243 |
| `weatherData` filtered per FormsId | `Passages.cs` | 274 |
| Gate 11/12 outer `if` check | `WeatherExclusion.cs` | 248 |
| Hours filter removing slots | `Passages.cs` | 65 |
| `GetAnalyzedWeatherDetails` writing to output table | `WeatherExclusion.cs` | 157–180 |
| Worker calling `GetAnalyzedWeatherDetails` | `Worker.cs` | 104–113 |

---

*Analysis based on code inspection of `gp-charterparty-jobs` TC2.0 AtSea worker.*
*Vessel: STI Gratitude (IMO 9722182) | Term ID: 147 | Weather Source: AnalysedWeather (ID=2)*
