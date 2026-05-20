# TCV2 Job Architecture
## Charter Party Performance — Oil Tankers
### `gp-charterparty-jobs`

---

```mermaid
flowchart TD
    HF([Hangfire Scheduler\nRecurring Job Trigger]) --> TCV2C

    subgraph API["API Layer"]
        TCV2C["TCV2Controller\nPOST /api/TCV2/RunJob"]
    end

    TCV2C --> TCV2S

    subgraph SVC["Service Layer"]
        TCV2S["TCV2Service\nOrchestrates all workers\nCollects results & errors\nReturns job summary"]
        TCV2S --> JS["JobService\n(Adapter)\nResolves vessel list\nLoops per vessel"]
        JS --> OF["ObjectFactory\nResolves IWorker\nby WorkerType enum"]
    end

    OF --> W1
    OF --> W2
    OF --> W3
    OF --> W4
    OF --> W5
    OF --> W6
    OF --> W7

    subgraph WORKERS["Workers — run sequentially per vessel, failure-isolated"]

        subgraph W1["AtSea Worker"]
            direction TB
            AS1["sp_GetCPTerms_AtSea\n(Charter Party Terms)"] --> AS2
            AS2["sp_GetReportData_AtSea\n(Daily Noon Reports)"] --> AS3
            AS3["Build Passages\n(group noon reports\ninto sea passages)"] --> AS4
            AS4["Weather Exclusion Filter\n🔲 Black Box\n(WI/WA/WW/WIC/WAC/WWC\n/C/SD/SP/EXC/UW/FCO\nexclusion codes)"] --> AS5
            AS5["Performance Calculation\n🔲 Black Box\n(Speed & Fuel vs\nCP Warranty)"] --> AS6
            AS6["Insert Results\nvia Table-Valued\nParameter TVP"]
        end

        subgraph W2["InPort Worker"]
            direction TB
            IP1["sp_GetCPTerms_InPort"] --> IP2
            IP2["sp_GetReportData_InPort\n(Arrival/Departure reports)"] --> IP3
            IP3["Port Stay Calculation\n🔲 Black Box"] --> IP4
            IP4["Insert Results via TVP"]
        end

        subgraph W3["Pumping Worker"]
            direction TB
            PM1["sp_GetCPTerms_Pumping"] --> PM2
            PM2["sp_GetReportData_Pumping\n(Discharge pump logs)"] --> PM3
            PM3["Pump Performance Calc\n🔲 Black Box"] --> PM4
            PM4["Insert Results via TVP"]
        end

        subgraph W4["CargoHeating Worker"]
            direction TB
            CH1["sp_GetCPTerms_CargoHeating"] --> CH2
            CH2["sp_GetReportData_CargoHeating"] --> CH3
            CH3["Heating Performance Calc\n🔲 Black Box"] --> CH4
            CH4["Insert Results via TVP"]
        end

        subgraph W5["CargoCooling Worker"]
            direction TB
            CC1["sp_GetCPTerms_CargoCooling"] --> CC2
            CC2["sp_GetReportData_CargoCooling"] --> CC3
            CC3["Cooling Performance Calc\n🔲 Black Box"] --> CC4
            CC4["Insert Results via TVP"]
        end

        subgraph W6["TankCleaning Worker"]
            direction TB
            TC1["sp_GetCPTerms_TankCleaning"] --> TC2
            TC2["sp_GetReportData_TankCleaning"] --> TC3
            TC3["Cleaning Performance Calc\n🔲 Black Box"] --> TC4
            TC4["Insert Results via TVP"]
        end

        subgraph W7["IGS Worker\n(Inert Gas System)"]
            direction TB
            IG1["sp_GetCPTerms_IGS"] --> IG2
            IG2["sp_GetReportData_IGS"] --> IG3
            IG3["IGS Performance Calc\n🔲 Black Box"] --> IG4
            IG4["Insert Results via TVP"]
        end
    end

    subgraph DB["perform-prod · AWS RDS · SQL Server"]
        direction LR
        DB1[("PassageDetails\nConsumptionDetails\nPassageSpeedPerformance\nPassageFuelPerformance\nAnalyzedWeatherDetails\nExcludedReports")]
    end

    W1 --> DB1
    W2 --> DB1
    W3 --> DB1
    W4 --> DB1
    W5 --> DB1
    W6 --> DB1
    W7 --> DB1

    subgraph ATSEA_OUTPUTS["AtSea Output Tables (6 tables written by AtSea Worker)"]
        O1["PassageDetails\n(passage metadata, dates, ports)"]
        O2["ConsumptionDetails\n(fuel consumption per passage)"]
        O3["PassageSpeedPerformance\n(actual vs warranted speed)"]
        O4["PassageFuelPerformance\n(actual vs warranted fuel)"]
        O5["AnalyzedWeatherDetails\n(weather data used in analysis)"]
        O6["ExcludedReports\n(noon reports excluded + reason code)"]
    end

    style W1 fill:#dbeafe,stroke:#3b82f6
    style W2 fill:#dcfce7,stroke:#16a34a
    style W3 fill:#fef9c3,stroke:#ca8a04
    style W4 fill:#ffe4e6,stroke:#e11d48
    style W5 fill:#f3e8ff,stroke:#9333ea
    style W6 fill:#ffedd5,stroke:#ea580c
    style W7 fill:#e0f2fe,stroke:#0284c7
    style DB fill:#f8fafc,stroke:#64748b
    style WORKERS fill:#fafafa,stroke:#e2e8f0
```

---

## AtSea Worker — Detailed Data Flow

```mermaid
flowchart TD
    START([Worker.Execute called\nper vessel per voyage]) --> TERMS

    subgraph FETCH["1 — Fetch Phase"]
        TERMS["Fetch CP Terms\nsp_GetCPTerms_AtSea\n→ Warranty speed/fuel,\nloaded/ballast, weather limits,\nallowances, vessel class"]
        TERMS --> REPORTS["Fetch Daily Reports\nsp_GetReportData_AtSea\n→ Noon reports:\nLat/Lon, Wind, Swell, Current,\nSpeed, FOC, Distance, Hours"]
    end

    REPORTS --> VALIDATE

    subgraph VALIDATE["2 — Validation Phase"]
        VALIDATE{"Reports found\n& CP Terms valid?"}
        VALIDATE -- No --> SKIP([Skip vessel — log reason])
        VALIDATE -- Yes --> PASSAGE
    end

    subgraph BUILD["3 — Passage Builder"]
        PASSAGE["Build Passages\n(group noon reports\nbetween departure & arrival\ninto sea passages)"]
        PASSAGE --> VOY["Aggregate into Voyages\n(Laden vs Ballast legs)"]
    end

    VOY --> WX

    subgraph WEATHER["4 — Weather Exclusion  🔲 Black Box"]
        WX["Evaluate each noon report\nagainst weather limits from CP Terms"]
        WX --> WX1{"Wind/Swell/Current\nexceeds threshold?"}
        WX1 -- Yes --> EXCL["Tag report with\nexclusion code:\nWI / WA / WW / C\n+ combined WIC/WAC/WWC"]
        WX1 -- No --> KEEP["Keep report\nin analysis"]
        EXCL --> EXCL2["Write to ExcludedReports table\nwith reason code + raw values"]
    end

    KEEP --> PERF
    EXCL --> PERF

    subgraph PERF["5 — Performance Calculation  🔲 Black Box"]
        PERF["Compute Speed Performance\n(actual vs CP warranted speed\nadjusted for current + allowances)"]
        PERF --> FUEL["Compute Fuel Performance\n(actual vs CP warranted FOC\nadjusted for sea conditions)"]
        FUEL --> WEATHER2["Compute Analyzed Weather\n(avg wind/swell used in calculation)"]
    end

    WEATHER2 --> INSERT

    subgraph INSERT["6 — Insert Phase"]
        INSERT["Serialize results to\nDataTable / TVP"]
        INSERT --> SP1["sp_InsertPassageDetails"]
        INSERT --> SP2["sp_InsertConsumptionDetails"]
        INSERT --> SP3["sp_InsertPassageSpeedPerformance"]
        INSERT --> SP4["sp_InsertPassageFuelPerformance"]
        INSERT --> SP5["sp_InsertAnalyzedWeatherDetails"]
        INSERT --> SP6["sp_InsertExcludedReports"]
    end

    subgraph EXCLUSION_CODES["Exclusion Code Reference"]
        EC["WI = Wind\nWA = Wave/Swell\nWW = Weather Waiting\nC = Current\nWIC = Wind+Current\nWAC = Wave+Current\nWWC = WW+Current\nSD = Speed Deviation\nSP = Stoppage\nEXC = Exceptional\nUW = Under Warranty\nFCO = FCO Clause"]
    end

    style FETCH fill:#dbeafe,stroke:#3b82f6
    style VALIDATE fill:#f0fdf4,stroke:#16a34a
    style BUILD fill:#fef9c3,stroke:#ca8a04
    style WEATHER fill:#ffe4e6,stroke:#e11d48
    style PERF fill:#f3e8ff,stroke:#9333ea
    style INSERT fill:#dcfce7,stroke:#16a34a
    style EXCLUSION_CODES fill:#f8fafc,stroke:#94a3b8
```
