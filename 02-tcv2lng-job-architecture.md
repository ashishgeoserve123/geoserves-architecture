# TCV2LNG Job Architecture
## Charter Party Performance — LNG Carriers
### `gp-charterparty-jobs`

---

```mermaid
flowchart TD
    HF([Hangfire Scheduler\nRecurring Job Trigger]) --> LNGC

    subgraph API["API Layer"]
        LNGC["TCV2LNGController\nPOST /api/TCV2LNG/RunJob"]
    end

    LNGC --> LNGS

    subgraph SVC["Service Layer"]
        LNGS["TCV2LNGService\nOrchestrates 5 workers\nSequential execution order\nCollects errors per vessel"]
        LNGS --> JS["JobService\n(Adapter)\nResolves LNG vessel list\nLoops per vessel"]
        JS --> OF["ObjectFactory\nResolves IWorker\nby WorkerType enum\n(LNG variant)"]
    end

    OF -->|"1st"| W_BOG
    OF -->|"2nd"| W_SEA
    OF -->|"3rd"| W_PORT
    OF -->|"4th"| W_PUMP
    OF -->|"5th"| W_IGS

    subgraph WORKERS["5 LNG Workers — strict sequential order, failure-isolated per worker"]

        subgraph W_BOG["① BOG Worker\n(Boil-Off Gas)"]
            direction TB
            B1["sp_GetCPTerms_BOG\n(BOG warranty rate,\nallowances, laden/ballast)"] --> B2
            B2["sp_GetReportData_BOG\n(Noon reports with\nLNG tank temps & pressures)"] --> B3
            B3["Passage Assembly\n(Laden passages only\nvs Ballast passages only)"] --> B4
            B4["BOG Thermodynamics\n🔲 Black Box\n(actual BOG rate vs\nCP warranted BOG rate\nadjusted for loaded qty\n& sea temperature)"] --> B5
            B5["CleanPumpPassageDetails()\nClamp overflow values:\nHours/BOGConsumed ≤ 999.99\nEffectiveCurrent ≤ ±99.99"] --> B6
            B6["Insert BOGPassageDetails\nInsert BOGConsumptionDetails\nInsert BOGPerformance\nvia TVP"]
        end

        subgraph W_SEA["② AtSea Worker\n(LNG Variant)"]
            direction TB
            AS1["sp_GetCPTerms_AtSea_LNG\n(Speed/Fuel warranty\nfor laden & ballast)"] --> AS2
            AS2["sp_GetReportData_AtSea_LNG\n(Noon reports + reliq data)"] --> AS3
            AS3["Passage Builder\n(Laden legs & Ballast legs\nwith re-liquefaction hours)"] --> AS4
            AS4["Weather Exclusion Filter\n🔲 Black Box\n(same codes as TCV2\nWI/WA/WW/C + combos)"] --> AS5
            AS5["LNG Speed & Fuel Perf\n🔲 Black Box\n(Voyage Order speed\nvs CP warranty)"] --> AS6
            AS6["Insert AtSea results\nvia TVP"]
        end

        subgraph W_PORT["③ InPort Worker\n(LNG Variant)"]
            direction TB
            IP1["sp_GetCPTerms_InPort_LNG\n(Port consumption warranty\nfor load/discharge port)"] --> IP2
            IP2["sp_GetReportData_InPort_LNG\n(Arrival/Departure/Port reports)"] --> IP3
            IP3["Port Stay Duration Calc\n(Load port vs Discharge port\nseparated)"] --> IP4
            IP4["Port Fuel Performance\n🔲 Black Box\n(actual vs warranted\nport consumption)"] --> IP5
            IP5["Insert Port results\nvia TVP"]
        end

        subgraph W_PUMP["④ Pumplog Worker\n(LNG Discharge Pumps)"]
            direction TB
            PM1["sp_GetCPTerms_Pumplog\n(Max pump time warranty\nLoad / Discharge)"] --> PM2
            PM2["sp_GetReportData_Pumplog\n(Pump start/stop events,\nmanifold pressure, flow rate)"] --> PM3
            PM3["Pump Time Calculation\n🔲 Black Box\n(actual pump hours vs\nwarranted pump hours\nper port call)"] --> PM4
            PM4["Insert Pumplog results\nvia TVP"]
        end

        subgraph W_IGS["⑤ IGS Worker\n(Inert Gas System — LNG)"]
            direction TB
            IG1["sp_GetCPTerms_IGS_LNG"] --> IG2
            IG2["sp_GetReportData_IGS_LNG"] --> IG3
            IG3["IGS Performance Calc\n🔲 Black Box"] --> IG4
            IG4["Insert IGS results\nvia TVP"]
        end

    end

    subgraph DB["perform-prod · AWS RDS · SQL Server"]
        LNG_T[("BOGPassageDetails\nBOGConsumptionDetails\nBOGPerformance\nLNGPassageDetails\nLNGConsumptionDetails\nLNGSpeedPerformance\nLNGFuelPerformance\nLNGWeatherDetails\nLNGExcludedReports\nLNGPortPerformance\nLNGPumplogPerformance\nLNGIGSPerformance")]
    end

    W_BOG --> LNG_T
    W_SEA --> LNG_T
    W_PORT --> LNG_T
    W_PUMP --> LNG_T
    W_IGS --> LNG_T

    style W_BOG fill:#e0f2fe,stroke:#0284c7
    style W_SEA fill:#dbeafe,stroke:#3b82f6
    style W_PORT fill:#dcfce7,stroke:#16a34a
    style W_PUMP fill:#fef9c3,stroke:#ca8a04
    style W_IGS fill:#ffe4e6,stroke:#e11d48
    style DB fill:#f8fafc,stroke:#64748b
    style WORKERS fill:#fafafa,stroke:#e2e8f0
```

---

## BOG Worker — Passage & Voyage Lifecycle

```mermaid
flowchart LR
    subgraph VOYAGE["Single LNG Voyage Structure"]
        direction TB
        DEP["Departure Port\n(Load Port)"]
        DEP -->|Laden Sea Passage| ARR["Arrival Port\n(Discharge Port)"]
        ARR -->|Discharge Operations| PUMP_OP["Pump Log\n(Unloading)"]
        PUMP_OP -->|Ballast Sea Passage| DEP2["Return to\nLoad Port"]
    end

    subgraph BOG_CALC["BOG Calculation Segments"]
        direction TB
        LADEN["Laden BOG Rate\n(tanks full → heat ingress\nhigher evaporation)"]
        BALLAST["Ballast BOG Rate\n(tanks empty → lower rate\nor re-liquefied)"]
        LADEN --> CP_COMP["Compare vs\nCP Warranted BOG Rate\n(separate for laden/ballast)"]
        BALLAST --> CP_COMP
        CP_COMP --> RESULT["BOG Performance Score\n(under / over warranty)"]
    end

    subgraph EXCLUSION["Voyage Order Types"]
        VO1["Standard Voyage Order\n(fixed CP speed)"]
        VO2["Revised Voyage Order\n(agreed amended terms)"]
        VO3["Non-Revised VO\n(AverageCPWarrantedSpeed\ncalc uses VoyageOrderEndTime\nnot arrivalTimeSET)"]
    end

    style VOYAGE fill:#e0f2fe,stroke:#0284c7
    style BOG_CALC fill:#fef9c3,stroke:#ca8a04
    style EXCLUSION fill:#f3e8ff,stroke:#9333ea
```

---

## Known Bug Fixes Applied (all 3 fix batches)

```mermaid
timeline
    title TCV2LNG Bug Fix History

    20 Feb 2026 : BOG Arithmetic Overflow
                : decimal 7,2 → decimal 5,2 column mismatch
                : Fix CleanPumpPassageDetails clamping
                : DateTimeOffset → DateTime in ExtendedFunctions

    04 Mar 2026 : TCV2LNG AtSea Worker — 3 Bugs
                : Wrong DataTable guard in Repository
                : Direct mutation of shared PassageData object
                : Unsafe .Last()/.First() LINQ → FirstOrDefault + null guard

    17 Mar 2026 : BOG Null Crashes — 3 null guards in Worker.cs
                : AverageCPWarrantedSpeed wrong denominator
                : arrivalTimeSET → VoyageOrderEndTime for non-revised VOs
```
