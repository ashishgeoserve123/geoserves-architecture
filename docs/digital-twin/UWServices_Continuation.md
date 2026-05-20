# Under Water Services — Continuation Notes

**Feature ticket:** GPR-1652  
**Branch (digital-twin):** `feature/gpr-1652/ashishgeoserve123`  
**Branch (database-project):** `feature/gpr-1652/ashishgeoserve123`  
**Last session:** backend module complete, EF Core optimisations applied, redundant SPs removed.

---

## What Is Done

### gp-digital-twin
The entire backend module lives at `src/DTAPI/UwServices/` and is production-ready.

| Layer | File | Status |
|---|---|---|
| Entity | `gp-data-model/Models/UwService.cs` | Done — mapped, FK to `Vessels` |
| DbSet | `gp-data-model/Models/DatabaseContext.cs` | Done — `DbSet<UwService> UwServices` |
| DTO | `UwServices/Models/UwServiceDto.cs` | Done |
| Bulk models | `UwServices/Models/BulkUploadDtos.cs`, `BulkUploadRow.cs`, `BulkUploadRowResult.cs` | Done |
| Validators | `UwServices/Validators/` (6 validators + chain) | Done |
| Parsers | `UwServices/Parsers/` (Excel via EPPlus, CSV via CsvHelper, resolver) | Done |
| Repository | `UwServices/Repository/UwServicesRepository.cs` | Done — pure EF Core, no SPs |
| Service | `UwServices/Service/UwServicesService.cs` | Done |
| Controller | `UwServices/Controller/UwServicesController.cs` | Done — 9 endpoints |
| DI wiring | `UwServices/Extensions/UwServicesServiceExtensions.cs` | Done — called from `Program.cs` |

**EF Core optimisations applied in last session:**
- `GetAllAsync` / `GetByIdAsync` — `.AsNoTracking()` added (read-only projections)
- `UpdateAsync` — replaced SELECT + SaveChanges with a single `ExecuteUpdateAsync`
- `DeleteAsync` — replaced SELECT + SaveChanges with a single `ExecuteUpdateAsync`
- `ReplaceAllAsync` (bulk confirm) — already used `ExecuteUpdateAsync` + `AddRangeAsync` in a transaction

### gp-database-project
- `gp-database/dbo/Tables/UwServices.sql` — table definition committed, deployed to UAT/prod.
- 6 redundant UwServices SP files (`UwServices_GetAll`, `GetById`, `Create`, `Update`, `Delete`, `SoftDeleteAll`) were **deleted** — they were never committed, the EF Core repository covers all of that logic.

---

## API Endpoints (all under `/api/digital-twin/UwServices/`)

| Method | Endpoint | Permission | Description |
|---|---|---|---|
| GET | `GetAll` | Read | All records, RBAC-filtered by vessel access |
| GET | `GetById?id=` | Read | Single record by PK |
| POST | `Create` | Write | Add a new record |
| PUT | `Update` | Write | Edit ServiceDate + Service (VesselId / IMO locked) |
| DELETE | `Delete?id=` | Full | Soft-delete (IsDeleted = 1) |
| GET | `GetVesselsDropdown` | Read | Vessel A–Z list for the Add Record form |
| GET | `GetTemplate` | Read | Download `.xlsx` bulk-upload template |
| POST | `ValidateBulkUpload` | Write | Parse + validate file, return preview — no commit |
| POST | `ConfirmBulkUpload` | Write | Replace all data with confirmed rows |

---

## What Still Needs to Be Done

### 1. RBAC database rows (manual DB task)
The RBAC constant is already in code (`ModuleMenuConstants.SettingsUnderWaterServices = "Under Water Services"`), but the matching rows must be inserted into the database by hand before the endpoints will be accessible in any protected environment.

Tables to update:
- `Modules` — insert the `Under Water Services` module row
- `MenuItems` — insert the menu item row linked to the module
- `RoleModuleMenuMapping` — map the relevant roles (at minimum admin + relevant user roles) to the module with Read / Write / Full permissions

### 2. Frontend (gp-website)
The backend is complete but the UI has not been started. A rough task breakdown:

- **List page** — table showing all UwService records for accessible vessels (calls `GetAll`), with Add / Edit / Delete actions and a Bulk Upload button
- **Add / Edit modal or form** — vessel picker dropdown (`GetVesselsDropdown`), service date picker, service text field; on save calls `Create` or `Update`
- **Bulk upload flow**
  1. Upload button → `ValidateBulkUpload` → show preview table with status colour-coding (green / amber / red per row) and summary bar (X valid · Y warnings · Z errors)
  2. "Confirm Import" button (shown only when `CanConfirm = true`) → `ConfirmBulkUpload` → show result toast
  3. Template download link → `GetTemplate`
- **Module guard** — the page should be gated behind the `Under Water Services` module permission, same as other Settings pages

### 3. Testing
- No unit tests have been written yet for the validators, service, or repository
- The Chain of Responsibility validators are pure functions and are easy to unit-test in isolation
- Suggested test project: add a `UwServices.Tests` project under `src/` mirroring the existing test project structure

---

## Key Design Decisions to Keep in Mind

- **Bulk upload = full replace.** Every `ConfirmBulkUpload` call soft-deletes ALL existing rows then inserts the new ones. This is intentional (stated requirement). Do not change this to a merge/upsert without discussion.
- **Soft-delete only.** `IsDeleted = 1` hides rows everywhere. Physical deletes never happen.
- **Editable fields are locked post-create.** Only `ServiceDate` and `Service` can be updated. `VesselId`, `ImoNumber`, and `VesselName` are set once on create and never changed — this is enforced in `UpdateAsync` via the `ExecuteUpdateAsync` property list.
- **RBAC vessel filter** uses the same pattern as `VesselRepository.GetAllVessels()` — administrator role sees all non-deleted vessels, everyone else sees only vessels reachable through their vessel group mappings.
- **No stored procedures.** All data access is EF Core. Do not introduce SPs for this module.

---

## File Locations Quick Reference

```
gp-digital-twin/
└── src/DTAPI/UwServices/
    ├── Controller/UwServicesController.cs
    ├── Extensions/UwServicesServiceExtensions.cs
    ├── Models/
    │   ├── UwServiceDto.cs
    │   ├── BulkUploadRow.cs
    │   ├── BulkUploadRowResult.cs
    │   └── BulkUploadDtos.cs
    ├── Parsers/
    │   ├── IFileParser.cs
    │   ├── ExcelFileParser.cs
    │   ├── CsvFileParser.cs
    │   └── FileParserResolver.cs
    ├── Repository/
    │   ├── IUwServicesRepository.cs
    │   └── UwServicesRepository.cs
    ├── Service/
    │   ├── IUwServicesService.cs
    │   └── UwServicesService.cs
    └── Validators/
        ├── IRowValidator.cs
        ├── RowValidationContext.cs
        ├── RowValidationChain.cs
        ├── ImoMissingValidator.cs
        ├── ImoNotFoundValidator.cs
        ├── ServiceDateMissingValidator.cs
        ├── ServiceMissingValidator.cs
        ├── FutureDateValidator.cs
        └── DuplicateRecordValidator.cs

gp-data-model/
├── Models/UwService.cs               ← EF Core entity
└── Models/DatabaseContext.cs         ← DbSet<UwService> UwServices

gp-database-project/
└── gp-database/dbo/Tables/UwServices.sql   ← table DDL (committed)
```
