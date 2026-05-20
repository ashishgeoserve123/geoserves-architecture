# LNG TCV2.0 Documentation Repository

This directory contains comprehensive documentation for the LNG TC2.0 vessel performance analysis system.

## Directory Structure

```
Docs/
└── LNGTCV2/
    ├── TCV2LNG_API_DOCUMENTATION.md          (54 KB - Main reference)
    ├── TCV2LNG_COMPLETE_IMPLEMENTATION.cs    (58 KB - Production code)
    ├── TCV2LNG_DOCUMENTATION_SUMMARY.md      (15 KB - Quick reference)
    ├── TCV2LNG_NAVIGATION_GUIDE.md           (14 KB - How to navigate)
    └── README.md                              (This file)
```

## Quick Navigation

### 📚 For Complete Understanding
Start with **TCV2LNG_DOCUMENTATION_SUMMARY.md** for a high-level overview, then dive into **TCV2LNG_API_DOCUMENTATION.md** for detailed technical information.

### 💻 For Implementation
Review **TCV2LNG_COMPLETE_IMPLEMENTATION.cs** - production-ready C# code with all 5 workers fully implemented.

### 🧭 For Finding Specific Information
Use **TCV2LNG_NAVIGATION_GUIDE.md** to locate specific algorithms, SQL procedures, or marine engineering concepts.

## File Descriptions

### TCV2LNG_API_DOCUMENTATION.md
**Size**: 54 KB | **Lines**: 1,553

Comprehensive technical documentation covering:
- API overview and endpoint specifications
- System architecture with component hierarchy
- All core components explained (Controller, Service, Factory)
- 8 detailed algorithms with pseudocode
- 20+ SQL stored procedures documented
- Marine engineering concepts explained
- Data flow diagrams
- Error handling and critical bug fixes

**Use this for**: Understanding every detail of how the system works

### TCV2LNG_COMPLETE_IMPLEMENTATION.cs
**Size**: 58 KB | **Lines**: 1,390

Production-ready C# implementation including:
- TCV2LNGController - HTTP entry point
- TCV2LNGService - Orchestration logic
- JobService - Worker adapter pattern
- ObjectFactory - Factory pattern implementation
- 5 complete worker implementations:
  - BOGWorker (Boil-Off Gas analysis)
  - AtSeaWorker (Speed & Fuel performance)
  - InPortWorker (Load/Discharge operations)
  - PumplogWorker (Pump validation)
  - IGSWorker (Inert Gas System)
- 24 data models with annotations
- Executable demo program

**Use this for**: Understanding the code structure, serving as template for similar APIs

### TCV2LNG_DOCUMENTATION_SUMMARY.md
**Size**: 15 KB | **Lines**: 461

Quick reference guide including:
- High-level overview
- System architecture summary
- The 5 workers explained
- Key algorithms at a glance
- SQL databases and procedures
- Marine engineering concepts
- Critical bug fix details
- How to use the documentation
- File structure in codebase

**Use this for**: Quick overview without diving into details

### TCV2LNG_NAVIGATION_GUIDE.md
**Size**: 14 KB | **Lines**: 400+

Navigation and reference guide including:
- Quick start guide by role
- Document contents and navigation
- How to find specific information
- Algorithms quick reference (8 algorithms mapped)
- SQL procedures reference (20+ procedures)
- Key concepts and definitions table
- Database configuration
- Implementation checklist
- Role-specific reading guides

**Use this for**: Finding specific information quickly

## What is TCV2LNG?

**TCV2LNG** = **TC2.0 LNG** performance analysis system

A comprehensive API that processes Liquefied Natural Gas (LNG) vessel performance data through 5 sequential workers:

1. **BOG Worker** - Boil-Off Gas (cargo evaporation) analysis
2. **AtSea Worker** - Speed and fuel consumption performance
3. **InPort Worker** - Loading and discharge operation efficiency
4. **Pumplog Worker** - Pump operation validation
5. **IGS Worker** - Inert Gas System fuel consumption tracking

Each worker:
- Retrieves specific operational data from SQL Server
- Applies domain-specific calculations
- Validates results against warranty terms
- Inserts processed metrics to database

## Key Features Documented

✅ **8 Core Algorithms** - With pseudocode and C# implementation
✅ **20+ SQL Procedures** - Fully documented with parameters and logic
✅ **5 Workers** - Each with complete business logic and error handling
✅ **Marine Engineering** - BOG thermodynamics, weather exclusions, performance metrics
✅ **Critical Bug Fix** - Decimal constraint validation (SQL Server overflow prevention)
✅ **Production Code** - Ready to compile and run
✅ **Design Patterns** - Factory, Service, Repository, Strategy patterns
✅ **Data Models** - 24 models fully annotated

## Quick Start by Role

### 👨‍💻 Developers
1. Read: TCV2LNG_COMPLETE_IMPLEMENTATION.cs
2. Review: API_DOCUMENTATION.md Section 4 (algorithms)
3. Reference: API_DOCUMENTATION.md Section 5 (SQL)

### 🧪 QA/Testers
1. Read: DOCUMENTATION_SUMMARY.md
2. Review: API_DOCUMENTATION.md Sections 1-2
3. Test: Each of 5 workers with sample data

### 👔 Business Analysts
1. Read: DOCUMENTATION_SUMMARY.md (overview)
2. Focus: Marine Engineering Concepts section
3. Understand: 5 workers' business purposes

### 🔧 DevOps/Database Admins
1. Read: API_DOCUMENTATION.md Section 5 (SQL procedures)
2. Focus: Database configuration and TVP structure
3. Reference: Stored procedure details

### 🏗️ Architects
1. Read: DOCUMENTATION_SUMMARY.md
2. Review: API_DOCUMENTATION.md Section 2 (architecture)
3. Study: Design patterns and component relationships

## Key Algorithms

1. **Passage Identification** - Matches Load-Discharge pump log pairs
2. **Weather-Based Exclusion** - Determines warranty exemptions based on weather
3. **BOG Performance Calculation** - Thermodynamic formula for evaporation
4. **Speed & Fuel Performance** - Warranty comparison metrics
5. **Port Operation Performance** - Loading/discharge rate analysis
6. **Pump Record Validation** - Data quality checks
7. **IGS Fuel Consumption** - System fuel rate tracking
8. **Decimal Constraint Validation** - Critical SQL Server precision fix

## Marine Engineering Concepts

### BOG (Boil-Off Gas)
Natural evaporation of LNG cargo at -162°C, affecting vessel economics. Rates vary by:
- Vessel insulation effectiveness
- Ambient temperature
- Loading condition (Laden vs Ballast)
- Weather conditions

### Vessel Conditions
- **Laden**: Fully loaded with LNG cargo (higher BOG rate)
- **Ballast**: Empty cargo tanks (lower BOG rate)

### Weather Exclusions
Bad weather periods excluded from warranty analysis:
- WI: Wind > 10 knots
- WA: Wave height > 4 meters
- C: Ocean current > 1.5 knots
- Combined conditions (WW, WIC, WAC, WWC)

### Performance Metrics
Formula: (Actual - Warranty) / Warranty × 100%
- Positive = Over-specification (poor performance)
- Negative = Under-specification (better than warranty)

## Database

**Server**: AWS RDS SQL Server
- **Host**: mssql-server.chqqwq0m85w2.ap-south-1.rds.amazonaws.com:1433
- **Database**: perform-prod
- **Tables**: ~40+ for vessel data, operations, metrics
- **Procedures**: 20+ stored procedures (4-5 per worker)

## System Architecture

```
POST /TCV2LNG
    ↓
TCV2LNGController.Post()
    ↓
TCV2LNGService.RunJob()
    ↓
[5 Sequential Workers]
├── BOGWorker
├── AtSeaWorker
├── InPortWorker
├── PumplogWorker
└── IGSWorker
    ↓
[5 SQL Stored Procedures via TVP]
    ↓
Aggregated Response
```

## Critical Bug Fix: BOG Arithmetic Overflow

**Issue**: SQL Server arithmetic overflow when inserting decimal values
**Root Cause**: Decimal precision mismatch (decimal 7,2 vs decimal 5,2)
**Solution**: Constraint validation before insertion

See **TCV2LNG_API_DOCUMENTATION.md Section 8** for complete details.

## Statistics

- **Total Documentation**: 3,404 lines
- **Total Size**: 141 KB
- **8 Detailed Algorithms**
- **20+ SQL Procedures**
- **5 Complete Workers**
- **24 Data Models**
- **4 Design Patterns**
- **Production-Ready Code**

## Getting Help

**To find information about**:
- **Algorithms**: See API_DOCUMENTATION.md Section 4
- **SQL**: See API_DOCUMENTATION.md Section 5
- **Marine concepts**: See DOCUMENTATION_SUMMARY.md or API_DOCUMENTATION.md Section 6
- **Code**: See COMPLETE_IMPLEMENTATION.cs with inline comments
- **Navigation**: See NAVIGATION_GUIDE.md

## Related Files in Codebase

```
src/
├── CPJobAPI/
│   ├── Controllers/TCV2LNGController.cs
│   ├── Service/TCV2LNGService.cs
│   └── CronJobs/TCV2LNGJob.cs
├── BusinessLogic/
│   ├── JobService.cs
│   ├── ObjectFactory.cs
│   └── Workers/TC2.0_LNG/
│       ├── BOG/Worker.cs
│       ├── AtSea/Worker.cs
│       ├── Inport/Worker.cs
│       ├── Pumplog/Worker.cs
│       └── IGS/Worker.cs
├── Repositories/TC2.0_LNG/
│   ├── BOG/Repository.cs
│   ├── AtSea/Repository.cs
│   ├── InPort/Repository.cs
│   ├── Pumplog/Repository.cs
│   └── IGS/Repository.cs
└── DataModels/TC2.0_LNG/
    ├── BOG/*.cs
    ├── AtSea/*.cs
    ├── InPort/*.cs
    ├── Pumplog/*.cs
    └── IGS/*.cs
```

---

**Documentation Created**: 2026-02-20
**Status**: Complete & Ready for Use
**Version**: 1.0
