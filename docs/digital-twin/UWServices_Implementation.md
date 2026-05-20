# Under Water Services — Backend Implementation Guide

## Overview

This document is the single source of truth for the **Under Water Services** backend module in the `gp-digital-twin` repository. It explains what was built, where every file lives, how the pieces connect, and how to extend or maintain the feature.

---

## Repository and Location

**Repo:** `gp-digital-twin`
**Path:** `src/DTAPI/UwServices/`
**API base path:** `/api/digital-twin/UwServices/`

The entire module is self-contained inside the `UwServices/` folder — you can understand, test, or replace any part without touching the rest of the DTAPI codebase.

---

## File Map

```
src/DTAPI/UwServices/
│
├── Models/
│   ├── UwServiceDto.cs             Single-record DTO (list, add, edit)
│   ├── BulkUploadRow.cs            Raw parsed row (pre-validation)
│   ├── BulkUploadRowResult.cs      Validated row result + RowStatus enum
│   └── BulkUploadDtos.cs           BulkUploadValidationResponse, BulkConfirmRequest, BulkConfirmResponse
│
├── Validators/
│   ├── IRowValidator.cs            Strategy interface
│   ├── RowValidationContext.cs     Shared mutable context flowing through the chain
│   ├── ImoMissingValidator.cs      ERROR: IMO absent or not an integer
│   ├── ImoNotFoundValidator.cs     ERROR: IMO not in vessel master
│   ├── ServiceDateMissingValidator.cs  ERROR: date absent or unparseable
│   ├── ServiceMissingValidator.cs  ERROR: service field empty
│   ├── FutureDateValidator.cs      WARN: service date is in the future
│   ├── DuplicateRecordValidator.cs WARN: same IMO + date + service already in DB
│   └── RowValidationChain.cs       Assembles and runs the chain
│
├── Parsers/
│   ├── IFileParser.cs              Strategy interface
│   ├── ExcelFileParser.cs          Handles .xlsx / .xls via EPPlus
│   ├── CsvFileParser.cs            Handles .csv via CsvHelper
│   └── FileParserResolver.cs       Selects the right parser by file extension
│
├── Repository/
│   ├── IUwServicesRepository.cs    Data-access contract
│   └── UwServicesRepository.cs     EF Core implementation
│
├── Service/
│   ├── IUwServicesService.cs       Business logic contract
│   └── UwServicesService.cs        Orchestrates parsers + validators + repository
│
├── Controller/
│   └── UwServicesController.cs     9 REST endpoints
│
└── Extensions/
    └── UwServicesServiceExtensions.cs   builder.Services.AddUwServicesModule()
```

### Files modified in the wider project

| File | Change |
|------|--------|
| `src/DTAPI/Program.cs` | Added `using DTAPI.UwServices.Extensions;` and `builder.Services.AddUwServicesModule();` |
| `src/DTAPI/DTAPI.csproj` | Added `CsvHelper 33.0.1` NuGet package |
| `gp-rbac/.../ModuleMenuConstants.cs` | Added `SettingsUnderWaterServices = "Under Water Services"` |

---

## API Endpoints

All endpoints are under `/api/digital-twin/UwServices/`.
All require `Under Water Services` module permission (set in the RBAC database).

| Method | Endpoint | Permission | Description |
|--------|----------|-----------|-------------|
| GET | `GetAll` | Read | All records, RBAC-filtered by vessel access |
| GET | `GetById?id=` | Read | Single record |
| POST | `Create` | Write | Add a new record |
| PUT | `Update` | Write | Edit ServiceDate + Service (Vessel/IMO locked) |
| DELETE | `Delete?id=` | Full | Soft-delete (IsDeleted = 1) |
| GET | `GetVesselsDropdown` | Read | A-Z vessel list for the Add Record form |
| GET | `GetTemplate` | Read | Download `.xlsx` bulk-upload template |
| POST | `ValidateBulkUpload` | Write | Validate file — returns preview, NO commit |
| POST | `ConfirmBulkUpload` | Write | Replace all data with confirmed rows |

---

## Design Patterns

### Chain of Responsibility — Row Validators

```
ImoMissing → ImoNotFound → ServiceDateMissing → ServiceMissing → FutureDate → Duplicate
```

- **ERROR validators** call `return false` to short-circuit the chain. Once a row is an ERROR, no further checks run.
- **WARN validators** always `return true` — the row is still importable.
- All validators share a `RowValidationContext` object that carries inputs (row data, vessel lookup, existing record keys) and outputs (status, notes, resolved typed values).

**Adding a new rule:** create a class implementing `IRowValidator`, add it to the list in `RowValidationChain`. Nothing else changes.

### Strategy — File Parsers

`IFileParser` is the strategy interface. `FileParserResolver` picks the right implementation at runtime by file extension. Both `ExcelFileParser` (EPPlus) and `CsvFileParser` (CsvHelper) are registered in DI as `IEnumerable<IFileParser>`.

**Adding a new format:** implement `IFileParser`, add one `AddTransient<IFileParser, YourParser>()` line in `UwServicesServiceExtensions`. Nothing else changes.

### Repository Pattern

`IUwServicesRepository` decouples the service layer from EF Core. The service depends only on the interface — easy to mock in tests and swap implementations.

### Vessel Access Control

Mirrors exactly the pattern used by `VesselRepository.GetAllVessels()`:
- `administrator` role → all non-deleted vessels
- All other roles → vessels reachable via `VesselGroupVesselMapping` ∩ `UserVesselGroupMappings`

---

## Bulk Upload Flow

```
User uploads file
       │
       ▼
POST /ValidateBulkUpload
  1. File size check    (≤ 5 MB)
  2. File type check    (FileParserResolver picks parser by extension)
  3. Parse raw rows     (ExcelFileParser or CsvFileParser)
  4. Row count check    (≤ 500 rows)
  5. Load vessel lookup + existing keys from DB  (one query each)
  6. Run RowValidationChain for every row
  7. Return BulkUploadValidationResponse
       │
       ▼ (user reviews preview table in UI, clicks "Confirm Import")
       │
       ▼
POST /ConfirmBulkUpload  { Rows: [...] }
  1. Filter ERROR rows out
  2. Transaction:
     a. ExecuteUpdate → soft-delete ALL existing records (IsDeleted = 1)
     b. AddRange     → insert OK + WARN rows as new records
  3. Return BulkConfirmResponse { ImportedCount, SkippedCount, Message }
```

**Important:** Every upload is a **full replace**. All existing `UwServices` rows are soft-deleted before the new rows are inserted. This is the stated requirement.

---

## Validation Rules

| Status | Condition | Row fate |
|--------|-----------|----------|
| ✅ OK | All checks pass, IMO found, no issues | Imported |
| ⚠ WARN | Service Date is in the future | Imported with flag |
| ⚠ WARN | Same IMO + date + service already in DB | Imported with flag |
| ❌ ERROR | IMO Number missing | Skipped |
| ❌ ERROR | IMO Number is not a valid integer | Skipped |
| ❌ ERROR | IMO Number not found in vessel master | Skipped |
| ❌ ERROR | Service Date missing | Skipped |
| ❌ ERROR | Service Date format unrecognisable | Skipped |
| ❌ ERROR | Service field empty | Skipped |

---

## Database Table

```sql
UwServices (
  UwServiceId       INT          IDENTITY PK
  VesselId          BIGINT       FK → Vessels.SFPM_VesselId
  ImoNumber         INT
  VesselName        VARCHAR(50)
  ServiceDate       DATETIME
  Service           VARCHAR(200)
  CreatedBy         VARCHAR(100)
  CreatedDateTime   DATETIME
  ModifiedBy        VARCHAR(100)
  ModifiedDateTime  DATETIME
  IsDeleted         BIT
)
```

Soft-delete: `IsDeleted = 1` hides the row from all queries. Physical rows are never removed.

---

## NuGet Package Added

| Package | Version | Purpose |
|---------|---------|---------|
| `CsvHelper` | 33.0.1 | Parsing `.csv` upload files |

`EPPlus 8.0.2` was already present in DTAPI and is reused for Excel parsing and template generation.

---

## RBAC Setup (database, not code)

The constant `ModuleMenuConstants.SettingsUnderWaterServices = "Under Water Services"` has been added to the `gp-rbac` library. A DB administrator must insert the corresponding rows into the RBAC tables (Modules, MenuItems, RoleModuleMenuMapping) before the endpoints will be accessible in a protected environment.

---

## How to Extend

### Add a new field to a record
1. Add the column to `UwService` entity in `gp-data-model`
2. Add the property to `UwServiceDto`
3. Update `UwServicesRepository` (Create, Update, Select projections)
4. Add a column to the template generator in `UwServicesService.GetUploadTemplate()`
5. Add a validator if the field needs validation

### Add a new validation rule
1. Create `MyValidator.cs` in `Validators/` implementing `IRowValidator`
2. Add it to the list in `RowValidationChain._validators`

### Support a new file format (e.g. ODS)
1. Create `OdsFileParser.cs` implementing `IFileParser`
2. Add `services.AddTransient<IFileParser, OdsFileParser>()` in `UwServicesServiceExtensions`
