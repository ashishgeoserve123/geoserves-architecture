# Request Pipeline Flow Diagram
## gp-reportengine-api · Full HTTP Request Lifecycle

---

## Report Request → PDF Delivery Pipeline

```mermaid
sequenceDiagram
    actor Client as Client (Web Portal / API Consumer)
    participant GW as API Gateway / Load Balancer
    participant RC as ReportController
    participant Auth as RBAC Middleware
    participant RS as ReportService
    participant DB as Report DB (SQL Server)
    participant RW as ReportWorker (Background)
    participant SC as SchedulerController (internal)
    participant OF as ObjectFactory
    participant IR as IReport impl (TCV2Report / TCV2LNGReport)
    participant IS as ISection impl (AtSea / BOG / InPort / etc.)
    participant PDB as perform-prod RDS
    participant QP as QuestPDF Renderer
    participant BLOB as Azure Blob Storage

    %% ─── Phase 1: Enqueue ───────────────────────────────────
    Note over Client, DB: Phase 1 — Enqueue Report Request

    Client->>GW: POST /Report/NewReportRequest/{reportType}<br/>Body: { vesselId, voyageId, cpPeriodStart, cpPeriodEnd, ... }
    GW->>RC: Forward request
    RC->>Auth: Validate JWT + RBAC module permission
    Auth-->>RC: 200 Authorized (or 403 Forbidden)
    RC->>RS: NewReport(reportType, inputParams)
    RS->>DB: INSERT ReportQueue<br/>{ ReportId, Status=Queued, ReportType, InputParamsJson }
    DB-->>RS: ReportId = guid
    RS-->>RC: ReportId
    RC-->>Client: 200 OK { reportId: "abc-123", status: "Queued" }

    %% ─── Phase 2: Section Analysis ──────────────────────────
    Note over RW, DB: Phase 2 — Background Worker Picks Up Request

    RW->>SC: GET /Scheduler/FetchOldestQueueReportReqId
    SC->>DB: SELECT TOP 1 WHERE Status=Queued ORDER BY CreatedAt
    DB-->>SC: ReportId = "abc-123"
    SC-->>RW: ReportId

    RW->>SC: GET /Scheduler/FetchSectionsForReport/{reportId}
    SC->>RS: FetchSectionsForReport(reportId)
    RS->>DB: SELECT InputParamsJson WHERE ReportId = reportId
    DB-->>RS: InputParamsJson
    RS->>OF: GetReportObject(reportType)
    OF-->>RS: IReport (TCV2Report or TCV2LNGReport)
    RS->>IR: PrepareInputParams(baseInputParams)
    IR-->>RS: Typed InputParams (TCV2InputParams / TCV2LNGInputParams)
    RS->>IR: AnalyzeAndFetchSections(sections[], inputParams)
    IR->>IR: NeedParticipation() check per section<br/>→ filters to active sections only
    IR-->>RS: List<ReportSectionType> [AtSea, InPort, Pumping, ...]
    RS->>DB: Serialize InputParams → JSON asset<br/>INSERT ReportSections per section type<br/>{ Status=Queued, SectionType, InputParamsAsset }
    RS->>DB: UPDATE ReportQueue SET Status=SectionsPrepared
    SC-->>RW: List of ReportSectionIds

    %% ─── Phase 3: Section Processing ────────────────────────
    Note over RW, PDB: Phase 3 — Process Each Section (Parallel.ForEach)

    loop For each ReportSectionId
        RW->>SC: POST /Scheduler/ProcessSection/{reportSectionId}
        SC->>RS: ProcessSection(reportSectionId)
        RS->>DB: SELECT SectionType, InputParamsAsset WHERE SectionId = id
        DB-->>RS: SectionType, InputParamsAsset JSON
        RS->>OF: GetSectionObject(sectionType)
        OF-->>RS: ISection (e.g. AtSeaSection, BOGPerformanceSection)
        RS->>IS: ConvertToSectionObject(inputParamsJson)
        IS-->>RS: Typed SectionInputParams
        RS->>IS: GenerateData(sectionInputParams)
        IS->>PDB: Query performance data<br/>(PassageDetails, BOGPerformance,<br/>ExcludedReports, WeatherDetails, ...)
        PDB-->>IS: ResultSet
        IS->>IS: Compute/aggregate section data
        IS-->>RS: BaseSectionData (strongly typed)
        RS->>IS: ConvertToJsonString(sectionData)
        IS-->>RS: SectionDataJson
        RS->>DB: UPDATE ReportSection SET Status=Processed, SectionDataAsset=json
        SC-->>RW: 200 OK
    end

    RW->>DB: CHECK all sections Status=Processed?
    DB-->>RW: Yes → all sections done

    %% ─── Phase 4: Document Generation ───────────────────────
    Note over RW, BLOB: Phase 4 — Render & Upload Document

    RW->>SC: GET /Scheduler/ReportDocGeneration/{reportId}
    SC->>RS: GenerateDocument(reportId)
    RS->>DB: SELECT all SectionDataAssets WHERE ReportId = reportId
    DB-->>RS: Dict<SectionType, SectionDataJson>
    RS->>OF: GetReportObject(reportType)
    OF-->>RS: IReport
    RS->>IR: ConvertJsonToBaseInputParams(inputParamsJson)
    IR-->>RS: InputParams
    RS->>IR: GenerateDocument(Dict<ReportSectionType, BaseSectionData>)
    IR->>QP: Compose PDF pages:<br/>FrontPage → TotalPerformance →<br/>Warranties → Summary →<br/>Section(s) → Methodology →<br/>ExcludedReports
    QP-->>IR: PDF byte stream
    IR->>BLOB: Upload PDF
    BLOB-->>IR: BlobDocumentId / URL
    IR-->>RS: DocumentId
    RS->>DB: UPDATE ReportQueue SET Status=Complete, DocumentId=id
    SC-->>RW: 200 OK { documentId }

    %% ─── Phase 5: Client Poll & Download ────────────────────
    Note over Client, BLOB: Phase 5 — Client Polls & Downloads

    Client->>RC: GET /Report/FetchStatusOfReport/{reportId}
    RC->>RS: GetReportStatus(reportId)
    RS->>DB: SELECT Status, DocumentId WHERE ReportId = id
    DB-->>RS: Status=Complete, DocumentId
    RS-->>RC: { status: "Complete", documentId }
    RC-->>Client: 200 OK

    Client->>RC: GET /Report/DownloadReport/{reportId}
    RC->>BLOB: Generate pre-signed download URL
    BLOB-->>RC: Signed URL (time-limited)
    RC-->>Client: 302 Redirect to signed URL / 200 with stream
```

---

## Report Status State Machine

```mermaid
stateDiagram-v2
    [*] --> Queued : POST NewReportRequest

    Queued --> SectionsPrepared : FetchSectionsForReport\n(IReport.AnalyzeAndFetchSections)

    SectionsPrepared --> Processing : ReportWorker picks up\n(Parallel.ForEach begins)

    Processing --> SectionProcessed : Each ISection.GenerateData()\ncompletes + JSON asset saved

    SectionProcessed --> Processing : More sections remaining

    SectionProcessed --> Rendering : All sections complete\n→ GenerateDocument triggered

    Rendering --> Complete : QuestPDF renders PDF\n→ uploaded to Blob

    Complete --> [*] : Client downloads report

    Queued --> Failed : Unhandled exception\nin any phase
    Processing --> Failed : Section GenerateData()\nthrows exception
    Rendering --> Failed : PDF render fails\nor Blob upload fails
    Failed --> Queued : Manual retry\n(re-queue request)
```

---

## ObjectFactory — Report Type Routing

```mermaid
flowchart TD
    OF["ObjectFactory\n(static factory)"]

    OF -->|"ReportType.TCV2"| R1["TCV2Report\n(IReport)\n→ 7 section types:\nAtSea, InPort, Pumping\nCargoHeating, CargoCooling\nTankCleaning, IGS"]

    OF -->|"ReportType.TCV2LNG"| R2["TCV2LNGReport\n(IReport)\n→ Sections: BOG + AtSea\n+ InPort + Pumplog + IGS"]

    OF -->|"ReportType.PoolRecap\n(FeatureFlag=V1)"| R3["PoolRecapReport\n(IReport)"]

    OF -->|"ReportType.PoolRecap\n(FeatureFlag=V2)"| R4["PoolRecapReportV2\n(IReport)\nFeatureFlagService toggle"]

    OF -->|"ReportType.Scorpio"| R5["ScorpioReport\n(IReport)"]

    subgraph SECTIONS["Section Resolver — GetSectionObject(ReportSectionType)"]
        S1["TCV2AtSeaSection → ISection"]
        S2["TCV2InPortSection → ISection"]
        S3["TCV2PumpingSection → ISection"]
        S4["TCV2CargoHeatingSection → ISection"]
        S5["TCV2CargoCoolingSection → ISection"]
        S6["TCV2TankCleaningSection → ISection"]
        S7["TCV2IGSSection → ISection"]
        S8["TCV2LNGBOGSection → ILNGPDF"]
        S9["TCV2LNGAtSeaSection → IPDF"]
        S10["TCV2LNGInPortSection → IPDF"]
        S11["TCV2LNGPumplogSection → IPDF"]
        S12["TCV2LNGIGSSection → IPDF"]
    end

    OF --> SECTIONS

    style OF fill:#fef9c3,stroke:#ca8a04
    style R1 fill:#dbeafe,stroke:#3b82f6
    style R2 fill:#e0f2fe,stroke:#0284c7
    style R3 fill:#dcfce7,stroke:#16a34a
    style R4 fill:#f3e8ff,stroke:#9333ea
    style R5 fill:#ffe4e6,stroke:#e11d48
    style SECTIONS fill:#f8fafc,stroke:#94a3b8
```

---

## Middleware & Cross-Cutting Concerns

```mermaid
flowchart LR
    REQ["Incoming HTTP Request"] --> MW1

    subgraph PIPELINE["ASP.NET Core Middleware Pipeline"]
        MW1["Exception Handler\n(global error formatting)"]
        MW1 --> MW2["HTTPS Redirection"]
        MW2 --> MW3["CORS Policy\n(configured origins)"]
        MW3 --> MW4["Authentication\nJWT Bearer validation"]
        MW4 --> MW5["Authorization\nRBAC module-level\npermission check"]
        MW5 --> MW6["Route Matching\n+ Controller Dispatch"]
    end

    MW6 --> CTRL["Controller Action\n(ReportController\nSchedulerController\nScorpioReportController\nHangfireJobController)"]
    CTRL --> RESP["HTTP Response"]

    style PIPELINE fill:#fafafa,stroke:#e2e8f0
    style MW4 fill:#fef9c3,stroke:#ca8a04
    style MW5 fill:#ffe4e6,stroke:#e11d48
```
