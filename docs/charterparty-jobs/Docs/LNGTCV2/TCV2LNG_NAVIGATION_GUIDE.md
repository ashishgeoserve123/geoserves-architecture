# TCV2LNG API Documentation - Complete Index & Navigation Guide

## Quick Start

If you're new to the TCV2LNG API, start here:

1. **First Read** (5 min): TCV2LNG_DOCUMENTATION_SUMMARY.md
2. **Deep Dive** (30 min): TCV2LNG_API_DOCUMENTATION.md - Sections: API Overview, System Architecture, The 5 Workers
3. **Implementation** (1 hour): TCV2LNG_COMPLETE_IMPLEMENTATION.cs - Review code structure and algorithms

---

## Document Locations & Contents

### 📖 Document 1: TCV2LNG_API_DOCUMENTATION.md (54 KB)

**Purpose**: Comprehensive technical documentation with algorithms, SQL queries, and marine engineering concepts

**Section 1: API Overview** (Line 1-50)
- Endpoint details (POST /TCV2LNG)
- Input/output specifications
- High-level flow diagram

**Section 2: System Architecture** (Line 51-150)
- Component hierarchy diagram
- Design patterns used (Service, Factory, Repository, Strategy)
- Component relationships

**Section 3: Core Components** (Line 151-400)
- TCV2LNGController (HTTP handler)
- TCV2LNGService (orchestrator)
- JobService (adapter)
- ObjectFactory (factory pattern)

**Section 4: Algorithms & Business Logic** (Line 401-1000)
- **BOG Worker**: Passage ID, weather exclusion, energy calc, constraint validation
- **AtSea Worker**: Speed/fuel performance calculation
- **InPort Worker**: Load/discharge rate analysis
- **Pumplog Worker**: Pump record validation
- **IGS Worker**: Inert gas system fuel consumption

**Section 5: SQL Queries & Database Operations** (Line 1001-1300)
- BOG Repository (4 SQL procedures detailed)
- Similar procedures for other workers (conceptual)
- Table-Valued Parameter (TVP) usage
- Database schema overview

**Section 6: Marine Engineering Concepts** (Line 1301-1400)
- BOG definition & thermodynamics
- Vessel loading conditions (Laden/Ballast)
- Weather exclusion criteria (Wind/Wave/Current)
- Performance metrics & warranty terms
- Speed, fuel, loading rates

**Section 7: Data Flow** (Line 1401-1450)
- Complete flow diagram from request to response
- Database interactions
- Worker coordination

**Section 8: Error Handling & Bug Fixes** (Line 1451-1553)
- BOG Arithmetic Overflow error explanation
- Root cause analysis
- Solution implementation
- Test results

---

### 💻 Document 2: TCV2LNG_COMPLETE_IMPLEMENTATION.cs (58 KB)

**Purpose**: Production-ready C# implementation of all TCV2LNG API components

**Section 1: Input/Output Models** (Line 1-70)
- `BaseInputParam` - Request parameters
- `BaseReturn` - Response object

**Section 2: Core Processing Classes** (Line 71-350)
- `TCV2LNGController` - HTTP POST handler with algorithm explanation
- `TCV2LNGService` - Orchestrator with sequential worker execution
- `JobService` - Worker adapter
- `ObjectFactory` - Factory pattern for worker instantiation

**Section 3: Worker Interface & Implementations** (Line 351-1200)
- `IWorker` - Interface contract
- **BOGWorker** (Line 370-750)
  - `RunWorkerAsync()` - Main execution flow
  - `RetrieveBOGWarrantyTerms()` - Fetch warranty terms
  - `RetrieveBOGDatabaseData()` - Fetch operational data
  - `IdentifyPumpLogPassages()` - Passage identification algorithm
  - `ExtractBOGPassageDetails()` - Extract passage information
  - `DetermineWeatherExclusionCode()` - Weather exclusion logic
  - `CalculateBOGPassagePerformance()` - Performance calculation
  - `CleanBOGDecimalConstraints()` - CRITICAL: Decimal constraint validation
  - `InsertBOGDataToDatabase()` - TVP insertion
  
- **AtSeaWorker** (Line 751-850)
  - `CalculateSpeedAndFuelPerformance()` - Speed/fuel algorithm
  
- **InPortWorker** (Line 851-950)
  - `CalculatePortPerformance()` - Port operation efficiency
  
- **PumplogWorker** (Line 951-1050)
  - `ValidatePumpRecord()` - Record validation logic
  
- **IGSWorker** (Line 1051-1150)
  - `CalculateIGSFuelConsumption()` - IGS fuel rate

**Section 4: Data Models** (Line 1201-1300)
- BOG Models (TcV2LNG BOG-specific classes)
- AtSea Models (Speed/fuel performance classes)
- InPort Models (Load/discharge operation classes)
- Pumplog Models (Pump operation classes)
- IGS Models (Inert gas system classes)

**Section 5: Enums** (Line 1301-1320)
- `WorkerType` enum with 5 worker types

**Section 6: Demo Program** (Line 1321-1390)
- Executable example showing complete API flow
- Creates request, executes controller, displays response

---

### 📋 Document 3: TCV2LNG_DOCUMENTATION_SUMMARY.md (15 KB)

**Purpose**: Quick reference guide for all documentation

**Sections**:
- Overview & file descriptions
- API overview & request/response examples
- System architecture summary
- The 5 workers explained (purpose, algorithms, marine context, SQL procedures)
- Key algorithms with pseudocode
- SQL databases & stored procedures
- Marine engineering concepts
- Critical bug fix details
- How to use the documentation files
- File structure in codebase
- Summary

---

## How to Find Specific Information

### Looking for...

**Algorithm Documentation** → TCV2LNG_API_DOCUMENTATION.md Section 4
- Passage identification: Lines 401-450
- Weather exclusion: Lines 451-520
- BOG energy calculation: Lines 521-570
- Decimal constraint validation: Lines 571-620

**SQL Queries** → TCV2LNG_API_DOCUMENTATION.md Section 5
- dbo.Job_GetTCV2LNGBOGTerms: Lines 1100-1150
- dbo.Job_GetTCV2LNGBOGFormData: Lines 1151-1220
- [dbo].[Job_InsertTCV2LNGBOGData]: Lines 1221-1300
- Similar procedures for other workers: Lines 1301-1350

**C# Code Implementation** → TCV2LNG_COMPLETE_IMPLEMENTATION.cs
- Controller handler: Lines 71-140
- Service orchestration: Lines 142-250
- BOG Worker complete: Lines 370-750
- Database operations: Lines 651-750
- Data models: Lines 1201-1300

**Marine Engineering Context** → Either document
- BOG explanation: TCV2LNG_API_DOCUMENTATION.md Section 6 or Summary.md "Marine Engineering Concepts"
- Vessel conditions: Search "Laden" or "Ballast"
- Weather criteria: Search "exclusion" or "WI, WA, WW"
- Performance metrics: Search "Index" or "Warranty"

**Bug Fix Details** → TCV2LNG_API_DOCUMENTATION.md Section 8 OR Summary.md "Critical Bug Fix"
- Problem: Lines 1451-1480 (Documentation) or Search "overflow"
- Root cause: Lines 1481-1510
- Solution: Lines 1511-1553

**Architecture Diagrams** → TCV2LNG_API_DOCUMENTATION.md Section 2 & 7
- Component hierarchy: Lines 100-130
- Complete data flow: Lines 1401-1450

---

## Algorithms Quick Reference

### 1. Passage Identification (BOGWorker.IdentifyPumpLogPassages)
**Location**: TCV2LNG_COMPLETE_IMPLEMENTATION.cs Line 510-560
**Documentation**: TCV2LNG_API_DOCUMENTATION.md Line 430-450
**Summary**: Sort pump logs, find Load events, match with Discharge events to identify passages

### 2. Weather-Based Exclusion (BOGWorker.DetermineWeatherExclusionCode)
**Location**: TCV2LNG_COMPLETE_IMPLEMENTATION.cs Line 600-650
**Documentation**: TCV2LNG_API_DOCUMENTATION.md Line 451-490
**Summary**: Check wind, wave, current thresholds and determine exclusion code (WI/WA/WW/C/etc)

### 3. BOG Performance Calculation (BOGWorker.CalculateBOGPassagePerformance)
**Location**: TCV2LNG_COMPLETE_IMPLEMENTATION.cs Line 670-720
**Documentation**: TCV2LNG_API_DOCUMENTATION.md Line 491-520
**Summary**: Compare actual BOG loss to warranty limits, calculate performance percentage

### 4. Speed & Fuel Performance (AtSeaWorker.CalculateSpeedAndFuelPerformance)
**Location**: TCV2LNG_COMPLETE_IMPLEMENTATION.cs Line 800-830
**Documentation**: TCV2LNG_API_DOCUMENTATION.md Section "AtSea Processing Algorithm"
**Summary**: Calculate actual speed and fuel consumption, compare to warranty

### 5. Port Operation Performance (InPortWorker.CalculatePortPerformance)
**Location**: TCV2LNG_COMPLETE_IMPLEMENTATION.cs Line 900-930
**Documentation**: TCV2LNG_API_DOCUMENTATION.md Section "InPort Processing Algorithm"
**Summary**: Calculate loading/discharge rates in tons/hour, compare to warranty

### 6. Pump Record Validation (PumplogWorker.ValidatePumpRecord)
**Location**: TCV2LNG_COMPLETE_IMPLEMENTATION.cs Line 1000-1020
**Documentation**: TCV2LNG_API_DOCUMENTATION.md Section "Pumplog Processing Algorithm"
**Summary**: Validate timestamps, quantity, check for duplicates

### 7. IGS Fuel Consumption (IGSWorker.CalculateIGSFuelConsumption)
**Location**: TCV2LNG_COMPLETE_IMPLEMENTATION.cs Line 1100-1120
**Documentation**: TCV2LNG_API_DOCUMENTATION.md Section "IGS Processing Algorithm"
**Summary**: Calculate fuel rate per hour for IGS operations

### 8. Decimal Constraint Validation (BOGWorker.CleanBOGDecimalConstraints)
**Location**: TCV2LNG_COMPLETE_IMPLEMENTATION.cs Line 735-780
**Documentation**: TCV2LNG_API_DOCUMENTATION.md Line 571-620
**Summary**: CRITICAL FIX - Constrain decimals to SQL Server precision (decimal 5,2)

---

## SQL Stored Procedures Reference

### BOG Worker (4 procedures)
1. `dbo.Job_GetTCV2LNGBOGTerms` - Get warranty terms (3 output tables)
2. `dbo.Job_GetTCV2LNGBOGFormData` - Get operational data (4 output tables)
3. `[dbo].[Job_InsertTCV2LNGBOGData]` - Insert via TVP (3 input TVPs)
4. `[dbo].[Job_DeleteTCV2LNGBOG]` - Delete records

### AtSea Worker (4 procedures)
1. `dbo.Job_GetTCV2LNGAtSeaTerms` - Get speed/fuel warranty
2. `dbo.Job_GetTCV2LNGAtSeaData` - Get sea passage data
3. `[dbo].[Job_InsertTCV2LNGAtSeaPassageData]` - Insert performance
4. `[dbo].[Job_DeleteTCV2LNGAtSea]` - Delete records

### InPort Worker (4 procedures)
1. `dbo.Job_GetTCV2LNGInPortTerms` - Get loading rates
2. `dbo.Job_GetTCV2LNGInPortData` - Get port operations
3. `[dbo].[Job_InsertTCV2LNGInPortData]` - Insert results
4. `[dbo].[Job_DeleteTCV2LNGInPort]` - Delete records

### Pumplog Worker (3 procedures)
1. `dbo.Job_GetTCV2LNGPumpLogTerms` - Get pump specs
2. `dbo.Job_GetTCV2LNGPumpLogData` - Get pump records
3. `[dbo].[Job_InsertTCV2LNGPumpLogData]` - Insert validated records

### IGS Worker (3 procedures)
1. `dbo.Job_GetTCV2LNGIGSTerms` - Get IGS specs
2. `dbo.Job_GetTCV2LNGIGSData` - Get IGS operations
3. `[dbo].[Job_InsertTCV2LNGIGSData]` - Insert results

---

## Key Concepts & Definitions

| Concept | Definition | Location |
|---------|-----------|----------|
| **BOG** | Boil-Off Gas - evaporation of LNG cargo | Summary.md, API_Docs.md Section 6 |
| **Passage** | Single voyage segment (Load → Discharge) | Multiple sections |
| **Voyage** | Complete commercial journey (Port A → Port B) | Summary.md |
| **Laden** | Fully loaded with cargo | Summary.md "Vessel Loading Conditions" |
| **Ballast** | Empty, filled with seawater for stability | Summary.md "Vessel Loading Conditions" |
| **Weather Exclusion** | Bad weather periods exempt from warranty | Summary.md "Weather Exclusions" |
| **Performance Index** | (Actual - Warranty) / Warranty × 100% | Summary.md "Performance Metrics" |
| **TVP** | Table-Valued Parameter for bulk SQL inserts | API_Docs.md Section 5 |
| **Decimal(5,2)** | SQL Server precision = max 999.99 | API_Docs.md Section 8 |
| **IGS** | Inert Gas System - prevents explosions | Summary.md "IGS" |

---

## Database Configuration

| Parameter | Value |
|-----------|-------|
| **Server** | mssql-server.chqqwq0m85w2.ap-south-1.rds.amazonaws.com,1433 |
| **Database** | perform-prod |
| **User** | gp-app-prod |
| **Timeout** | 120 seconds |
| **MARS** | Enabled |
| **Tables** | ~40+ for vessel data, operations, metrics |

---

## Implementation Checklist

- [ ] Read TCV2LNG_DOCUMENTATION_SUMMARY.md (overview)
- [ ] Review System Architecture in API_DOCUMENTATION.md
- [ ] Understand the 5 workers (purpose, algorithms)
- [ ] Study SQL stored procedures (20+ procedures)
- [ ] Review C# implementation in COMPLETE_IMPLEMENTATION.cs
- [ ] Understand BOG decimal constraint fix (critical)
- [ ] Learn marine engineering concepts
- [ ] Review data models and relationships
- [ ] Test individual algorithms with sample data
- [ ] Integrate with existing codebase
- [ ] Deploy to production

---

## For Different Audiences

### For Developers
1. Read: COMPLETE_IMPLEMENTATION.cs - Understand code structure
2. Review: API_DOCUMENTATION.md Section 4 - Learn algorithms
3. Reference: SQL procedures documentation

### For QA/Testers
1. Read: DOCUMENTATION_SUMMARY.md - Understand purpose
2. Review: API_DOCUMENTATION.md Sections 1-2 - Understand flow
3. Test: Each of 5 workers with sample data

### For Business Analysts
1. Read: DOCUMENTATION_SUMMARY.md - High-level overview
2. Focus: Marine Engineering Concepts section
3. Understand: 5 workers' business purposes

### For DevOps/Database Admins
1. Read: API_DOCUMENTATION.md Section 5 - SQL procedures
2. Focus: Database configuration and TVP structure
3. Reference: Stored procedure details

### For Architects
1. Read: DOCUMENTATION_SUMMARY.md
2. Review: API_DOCUMENTATION.md Section 2 - Architecture
3. Study: Design patterns and component relationships

---

## File Structure in Codebase

```
Project Root
├── TCV2LNG_API_DOCUMENTATION.md          ← Main documentation
├── TCV2LNG_COMPLETE_IMPLEMENTATION.cs    ← Production code
├── TCV2LNG_DOCUMENTATION_SUMMARY.md      ← Quick reference
├── TCV2LNG_NAVIGATION_GUIDE.md           ← This file
│
├── src/CPJobAPI/
│   ├── Controllers/TCV2LNGController.cs
│   ├── Service/TCV2LNGService.cs
│   └── CronJobs/TCV2LNGJob.cs
│
├── src/BusinessLogic/
│   ├── JobService.cs
│   ├── ObjectFactory.cs
│   └── Workers/TC2.0_LNG/
│       ├── BOG/Worker.cs
│       ├── AtSea/Worker.cs
│       ├── Inport/Worker.cs
│       ├── Pumplog/Worker.cs
│       └── IGS/Worker.cs
│
├── src/Repositories/TC2.0_LNG/
│   ├── BOG/Repository.cs
│   ├── AtSea/Repository.cs
│   ├── InPort/Repository.cs
│   ├── Pumplog/Repository.cs
│   └── IGS/Repository.cs
│
└── src/DataModels/TC2.0_LNG/
    ├── BOG/*.cs
    ├── AtSea/*.cs
    ├── InPort/*.cs
    ├── Pumplog/*.cs
    └── IGS/*.cs
```

---

## Support & References

**Questions about**:
- **Algorithms**: See API_DOCUMENTATION.md Section 4
- **SQL**: See API_DOCUMENTATION.md Section 5
- **Marine concepts**: See Summary.md or API_DOCUMENTATION.md Section 6
- **Code**: See COMPLETE_IMPLEMENTATION.cs with inline comments
- **Architecture**: See API_DOCUMENTATION.md Sections 2-3
- **Bug fixes**: See API_DOCUMENTATION.md Section 8 or Summary.md

---

**Created**: 2026-02-20
**Documentation Version**: 1.0
**API Status**: Fully Documented & Implemented

Last Updated: 2026-02-20 11:59 UTC
