# SalesHierarchy

ASP.NET Core MVC application for managing a sales organization: salespersons and their reporting hierarchy, POD stations, PODs, dealers, and dealer mappings. Business data is accessed through ADO.NET repositories (`Microsoft.Data.SqlClient`), while authentication uses ASP.NET Core Identity.

## Stack

- .NET 10 / ASP.NET Core MVC with Razor views
- SQL Server (`SalesProdAuthDb`)
- ADO.NET + Repository pattern for all business tables
- ASP.NET Core Identity for login, roles, password hashing, lockout, and password reset tokens
- Bootstrap 5.3 with a custom design system (`wwwroot/css/site.css`)
- xUnit tests in `SalesHierarchy.Tests`

## Structure

```
SalesHierarchy/
├── SalesHierarchy.sln              # solution (Web, Core, Infrastructure, Tests, SQL project)
├── src/
│   ├── SalesHierarchy.Web/          # controllers, Razor views, view models, Identity, DI
│   ├── SalesHierarchy.Core/         # models, DTOs, repository interfaces, validation rules
│   └── SalesHierarchy.Infrastructure/# SqlConnectionFactory + ADO.NET repositories
├── tests/
│   └── SalesHierarchy.Tests/        # xUnit unit + LocalDB integration tests
├── Data/Excel/                       # source Excel workbooks for initial master data import
│   ├── pods.xlsx
│   └── Sales Mapping With Dealer and POD.xlsx
└── database/
    ├── SalesHierarchy.Database/     # SQL Server Database Project (DACPAC)
    ├── Import-Dealer.sql            # Excel master data import: dealers
    ├── Import-POD.sql                # Excel master data import: PODs
    ├── Import-PODStation.sql        # Excel master data import: POD stations
    ├── Import-SalesMapping.sql      # Excel master data import: sales mappings
    ├── Upgrade-AuditLog.sql         # idempotent AuditLog deployment
    ├── Upgrade-SalesPersonPOD.sql   # idempotent POD ownership / station upgrade
    └── Upgrade-PODNameUniqueness.sql # idempotent per-station POD name uniqueness upgrade
```

> Build with `dotnet build SalesHierarchy.sln`. The SQL project requires SQL Server Data Tools (SSDT) in Visual Studio; plain `dotnet build` skips it so the .NET projects can build anywhere. Apply the table scripts with `database/Setup-LocalDb.ps1` or publish the DACPAC from Visual Studio.

## Business tables

| Table | Key | Notes |
|---|---|---|
| `PODStation` | `ID` | Optional `OSSPODStationId` (`int`), required `Name` (`nvarchar(150)`) |
| `SalesPerson` | `ID` | Required `Name`/`Designation`; `ManagerId` self-reference is the source of truth for hierarchy |
| `POD` | `ID` | Optional `OSSPODID` (`int`); `PODStationId` is nullable until a station is assigned; unique `Name` |
| `SalesPersonPOD` | `(SalesPersonId, PODId)` | Many-to-many POD ownership. Composite primary key, `NO ACTION` foreign keys, index on `PODId` |
| `Dealer` | `UniqueID` | Required `Name` and unique `Code` (`nvarchar(50)`); when no verified name exists, the code is used as the name |
| `salesdealermapping` | `UniqueID` | Links a salesperson to a dealer; duplicate pairs blocked by a unique constraint |
| `AuditLog` | `ID` | Append-only history of business/account changes (added by this application); optional `RelatedSalesPersonId` scopes an event to a team |

POD ownership is modelled **only** by `dbo.SalesPersonPOD`. The legacy `POD.salespersonId` column is left in place temporarily so no data is lost; the application no longer reads or writes it. `database/Upgrade-SalesPersonPOD.sql` copies any legacy values into the junction table before the new UI is used.

The scripts in `database/SalesHierarchy.Database/Tables` match the existing production schema exactly (column names, types, nullability, keys, unique constraints, and `NO ACTION` foreign keys).

No tables are recreated and no indexes/clustering are redesigned. Delete operations are blocked with a clear message while dependent records exist; no cascading deletes were added.

## Local development (localhost / LocalDB)

No connection string is committed. Supply it through user secrets (development) or environment variables (deployment). A setup script creates the database, business tables, and (optionally) demo data:

```powershell
# Local SQL Server instance (default)
powershell -ExecutionPolicy Bypass -File .\database\Setup-LocalDb.ps1 -SeedDemoData

# Or SQL Server LocalDB
powershell -ExecutionPolicy Bypass -File .\database\Setup-LocalDb.ps1 -Server "(localdb)\MSSQLLocalDB" -SeedDemoData
```

The setup script creates the business tables (including `dbo.SalesPersonPOD` and `dbo.AuditLog`) and runs the idempotent upgrade scripts. To upgrade an existing database manually, run in this order:

```powershell
sqlcmd -S <server> -d SalesProdAuthDb -U <user> -P <password> -C -b -i .\database\Upgrade-AuditLog.sql
sqlcmd -S <server> -d SalesProdAuthDb -U <user> -P <password> -C -b -i .\database\Upgrade-SalesPersonPOD.sql
sqlcmd -S <server> -d SalesProdAuthDb -U <user> -P <password> -C -b -i .\database\Upgrade-PODNameUniqueness.sql
```

Then configure the connection, apply the committed Identity migration, and set the first administrator:

```powershell
dotnet tool restore
dotnet user-secrets set "ConnectionStrings:DefaultConnection" "Server=(localdb)\MSSQLLocalDB;Database=SalesProdAuthDb;Trusted_Connection=True;TrustServerCertificate=True;MultipleActiveResultSets=True" --project src/SalesHierarchy.Web
dotnet ef database update --project src/SalesHierarchy.Web

dotnet user-secrets set "AdminBootstrap:Email" "admin@saleshierarchy.local" --project src/SalesHierarchy.Web
dotnet user-secrets set "AdminBootstrap:Password" "Admin#12345" --project src/SalesHierarchy.Web

dotnet run --project src/SalesHierarchy.Web
```

Open the URL shown in the console and sign in with the bootstrapped administrator. In Development the app also applies pending Identity migrations automatically at startup.

### Existing SQL Server database

For the shared `SalesProdAuthDb` instance, point `ConnectionStrings:DefaultConnection` at the server with SQL authentication and `TrustServerCertificate=True`:

```powershell
dotnet user-secrets set "ConnectionStrings:DefaultConnection" "Server=<server>;Database=SalesProdAuthDb;User Id=<user>;Password=<password>;TrustServerCertificate=True;MultipleActiveResultSets=True" --project src/SalesHierarchy.Web
```

Run `database/Upgrade-AuditLog.sql`, `database/Upgrade-SalesPersonPOD.sql`, and `dotnet ef database update` once against that database. The app never creates or drops the business tables, and does not touch the legacy `dbo.Users` table or stored procedures.

`Upgrade-SalesPersonPOD.sql` is additive: it creates `dbo.SalesPersonPOD` when missing, copies legacy `POD.salespersonId` ownership into it, and makes `POD.PODStationId` nullable. It never drops the legacy column or invents station mappings. Take a backup first and review the change on a copy of the database.

## Configuration

All secrets must come from user secrets (development) or environment variables (production). Nothing sensitive is committed.

```powershell
cd src/SalesHierarchy.Web
dotnet user-secrets init
dotnet user-secrets set "ConnectionStrings:DefaultConnection" "Server=localhost;Database=SalesProdAuthDb;Trusted_Connection=True;TrustServerCertificate=True;MultipleActiveResultSets=True"
dotnet user-secrets set "AdminBootstrap:Email" "admin@yourcompany.com"
dotnet user-secrets set "AdminBootstrap:Password" "Use-A-Strong-Password1"
```

### First System Admin

On startup the application:

1. Creates the `SystemAdmin` and `SalesPerson` roles if they are missing.
2. If no `SystemAdmin` exists **and** `AdminBootstrap:Email` + `AdminBootstrap:Password` are configured, creates that account.

There is no public self-registration and no hard-coded seed password. Remove the `AdminBootstrap` secrets after the first successful login. The bootstrap is skipped when an administrator already exists.

> If the database is not reachable at startup, the bootstrap logs a warning instead of crashing the application. Deploy the database first, then restart.

### E-mail delivery

Password setup and reset links are sent through SMTP:

```powershell
dotnet user-secrets set "Email:Enabled" "true"
dotnet user-secrets set "Email:SmtpHost" "smtp.yourprovider.com"
dotnet user-secrets set "Email:SmtpPort" "587"
dotnet user-secrets set "Email:UseSsl" "true"
dotnet user-secrets set "Email:UserName" "smtp-user"
dotnet user-secrets set "Email:Password" "smtp-password"
dotnet user-secrets set "Email:FromAddress" "no-reply@yourcompany.com"
dotnet user-secrets set "Email:FromName" "SalesHierarchy"
```

If e-mail is not configured or delivery fails, the UI reports the invitation as **pending/failed** instead of pretending it was sent. Invitations can be re-sent by a System Administrator from **User Accounts**. Passwords and tokens are never logged.

#### Invitation flow

1. An administrator creates a salesperson account (individually or via bulk upload). No password is set by the administrator.
2. The application creates the Identity account without a password and generates a one-time password-reset token.
3. The salesperson receives an e-mail containing a **Set my password** link.
4. They choose their own password; only the hash is stored. The link expires according to Identity's token lifespan and becomes invalid after use.

#### Local e-mail without an SMTP server

`appsettings.Development.json` enables a local pickup directory:

```json
"Email": {
  "Enabled": true,
  "PickupDirectory": "App_Data/emails",
  "FromAddress": "no-reply@saleshierarchy.local"
}
```

Invitation messages are written as `.eml` files under `src/SalesHierarchy.Web/App_Data/emails` instead of being sent. Open the file and copy the **Set my password** URL into the browser. This folder is git-ignored and contains sensitive one-time links — delete it after testing. For real delivery, remove `Email:PickupDirectory` and configure the SMTP settings above (user secrets or environment variables).

### Identity schema

The application uses the existing database connection for both business data and Identity. The Identity tables (`AspNetUsers`, `AspNetRoles`, …) are created by the committed migration in `src/SalesHierarchy.Web/Data/Migrations`:

```powershell
dotnet tool restore
dotnet ef database update --project src/SalesHierarchy.Web
```

In Development the application also applies pending Identity migrations automatically at startup. In production, run `dotnet ef database update` (or generate an idempotent SQL script with `dotnet ef migrations script --idempotent`) during deployment.

Inspect the existing `SalesProdAuthDb` schema first and do not recreate tables that already exist. `ApplicationUser` adds only `SalesPersonId` (nullable) to link an account to a `SalesPerson.ID`.

## Running

```powershell
dotnet build SalesHierarchy.sln
dotnet run --project src/SalesHierarchy.Web
```

Then open the HTTPS URL shown in the console and sign in with the bootstrapped administrator.

## Features

### Public (anonymous)

- Home and public directory
- Approved sales hierarchy (names and designations only)
- Approved POD station and POD lists
- Search on each public list

Public pages use read-only DTOs (`PublicSalesPersonDto`, `PublicPODStationDto`, `PublicPODDto`). Emails, dealer mappings, account data, and all write operations are unavailable.

### System Admin

- Full CRUD for all business tables (list, details, create, edit, delete confirmation)
- Search, filters, and server-side pagination on list pages and database-populated dropdowns
- POD station assignment: apply one station to the selected PODs in a single transaction without touching other PODs
- POD ownership assignment: choose a salesperson and replace only that person's PODs in a single transaction
- Reporting-manager assignment with designation-aware rules (CXO → none, Head → CXO, Sales Manager/Supervisor → Head, Sales Officer → Sales Manager/Supervisor), self-reporting and circular-hierarchy rejection. This is the only place where a salesperson's manager can be created or changed.
- Delete guards: subordinates, assigned PODs, dealer mappings, and linked accounts block salesperson deletion; PODs block station deletion; mappings block dealer deletion
- Individual salesperson account creation (business record + Identity account + invitation in one flow, with compensating rollback)
- Excel bulk upload with validation, preview, confirmation, result summary, and downloadable results
- Account management: invitation status, resend invitation, disable/enable, delete account
- Complete reporting hierarchy view per salesperson
- Activity log: filterable, read-only history of business and account changes stored in `dbo.AuditLog` (before/after values for edits; passwords and tokens excluded)

### SalesPerson

Every SalesPerson has a personal, read-only workspace: dashboard, profile, reporting chain, assigned PODs, mapped dealers, and a view of their own subtree (direct and indirect reports). They can also see team accounts and team PODs, and can add or remove dealer mappings for themselves and their descendants. **They cannot** create or edit accounts, change reporting lines, reassign PODs, modify master data, or reach another team's records by changing an ID in a URL. All of those actions are System Admin–only and are enforced on the controller, not just hidden in the UI.

## Excel bulk upload

1. Download the template from **Admin → Bulk Upload** (`Name`, `Designation`, `Email`, `ManagerEmail`).
2. Fill in rows and upload the `.xlsx` file (maximum 1,000 rows / 5 MB).
3. Every row is validated: required fields, e-mail format, duplicate e-mails (in file and existing accounts), unknown managers, self-reporting, and circular relationships.
4. Review the preview — nothing is written yet.
5. Confirm the import. Salesperson records and Identity accounts are created without passwords, links are resolved in a second pass (so managers can appear later in the file), and invitations are sent.
6. Results show imported, failed, skipped, and invitation status. Download the results workbook for failed rows.

Re-uploading the same file is safe: existing e-mails are detected as duplicates and skipped. Failed rows are compensated (created business records and identity accounts are removed) and reported.

## Excel master data import (initial population)

The application does **not** include an Excel upload feature for master data. The source
workbooks are kept inside the project under `Data/Excel/`:

```
Data/Excel/
├── pods.xlsx                                    # worksheets: pod, station
└── Sales Mapping With Dealer and POD.xlsx      # worksheet: Sheet1
```

- `pods.xlsx` → `dbo.PODStation` (worksheet `station`: numeric id → `OSSPODStationId`, name → `Name`) and
  `dbo.POD` (worksheet `pod`: `PODId` → `OSSPODID`, `PODName` → `Name`).
- `Sales Mapping With Dealer and POD.xlsx` → `dbo.Dealer` (OSS dealer codes), `dbo.SalesPerson` hierarchy
  (CXO → Head → Sales Manager/Supervisor → Sales Officer) plus `dbo.SalesPersonPOD` ownership and
  `dbo.salesdealermapping` dealer links.

Initial population uses the prepared, idempotent SQL scripts in `database/`:

```powershell
powershell -ExecutionPolicy Bypass -File .\database\Setup-LocalDb.ps1 -ImportExcel
```

or directly with `sqlcmd` against the target database, for example:

```powershell
sqlcmd -S <server> -d SalesProdAuthDb -U <user> -P <password> -C -b -i .\database\Import-PODStation.sql
sqlcmd -S <server> -d SalesProdAuthDb -U <user> -P <password> -C -b -i .\database\Import-POD.sql
sqlcmd -S <server> -d SalesProdAuthDb -U <user> -P <password> -C -b -i .\database\Import-Dealer.sql
sqlcmd -S <server> -d SalesProdAuthDb -U <user> -P <password> -C -b -i .\database\Import-SalesMapping.sql
sqlcmd -S <server> -d SalesProdAuthDb -U <user> -P <password> -C -b -i .\database\Verify-ExcelImport.sql
```

The import scripts are idempotent (skip existing rows), run inside a transaction, and skip source rows
that cannot be resolved (for example POD-station references that do not match the station list).
The import summary and orphan/uniqueness checks live in `database/Verify-ExcelImport.sql`.

> `PODStation.xlsx` was planned as a separate workbook but was not found in the repository; the
> station sheet inside `pods.xlsx` is used instead.

## Testing

```powershell
dotnet test SalesHierarchy.sln
```

Automated tests cover:

- Hierarchy rules: self-reporting, direct/deep cycles, valid reassignment, descendants, manager chain, cycle tolerance, subtree scope
- Designation manager rules: allowed and forbidden manager combinations, unknown designations, self/cycle guards
- Bulk row rules: required fields, e-mail formats, length limits, self-reporting, in-file circular managers
- Bulk upload status logic
- Authorization: controller role requirements, POST/anti-forgery protection of admin and SalesPerson mutations, and the absence of removed SalesPerson write actions
- ADO.NET repositories against a throwaway SQL Server LocalDB database: CRUD, unique constraints, foreign-key guards, duplicate dealer mappings, nullable POD station, transactional POD ownership replacement, bulk station assignment, and audit filters by user/action/entity/date

The integration tests create and drop their own database (`SalesHierarchyTests_<guid>`) using the scripts in `database/SalesHierarchy.Database/Tables`. They require SQL Server LocalDB (`(localdb)\MSSQLLocalDB`); set `DOTNET_SKIP_FIRST_TIME_EXPERIENCE` if the instance has never been started.

Manual checks worth repeating after deployment: anonymous public pages, login/lockout/forgot/reset/access-denied, admin CRUD and delete guards, bulk upload preview/confirm/error report, invitations and resends, and SalesPerson cross-team URL tampering.

## Database deployment

The SQL Server Database Project lives in `database/SalesHierarchy.Database`. See its [README](database/SalesHierarchy.Database/README.md) for DACPAC build/publish steps. Always take a backup before publishing and never drop production data.

For an existing database, deployment is additive only:

1. Run `database/Upgrade-AuditLog.sql` to create `dbo.AuditLog` / add `RelatedSalesPersonId` if missing (idempotent).
2. Run `database/Upgrade-SalesPersonPOD.sql` to create `dbo.SalesPersonPOD`, backfill legacy `POD.salespersonId` values, and make `POD.PODStationId` nullable (idempotent).
3. Run `database/Upgrade-PODNameUniqueness.sql` to replace the global `UQ_POD_Name` unique constraint with the filtered unique index `UX_POD_Name_PerStation` on `(PODStationId, Name) WHERE PODStationId IS NOT NULL` (idempotent) — enforcing per-station POD name uniqueness while allowing unassigned PODs to share names.
4. Run `dotnet ef database update --project src/SalesHierarchy.Web` (or an idempotent script from `dotnet ef migrations script --idempotent`) to create the ASP.NET Identity tables.
4. Set `AdminBootstrap:Email`/`Password` once, start the app to create the first SystemAdmin, then remove those secrets.
5. Configure SMTP through `Email:*` environment variables or user secrets.

The business tables, the legacy `dbo.Users` table, and existing stored procedures are never dropped or recreated.

### Pre-deployment checks for the shared database

The current shared `SalesProdAuthDb` predates this schema, so verify these points before pointing the new UI at it:

- **Take a verified backup first.** No recent full backup was found on the shared SQL Server during development. Do not run the upgrade scripts on the live database until a restorable backup exists.
- **Upgrade before starting the app.** The live database has `POD.PODStationId INT NOT NULL` and no `dbo.SalesPersonPOD`. Run `Upgrade-SalesPersonPOD.sql` (and `Upgrade-AuditLog.sql`) first; the new create/edit POD screens cannot save a POD without a station until the column is nullable.
- **Legacy ownership is copied, not guessed.** `Upgrade-SalesPersonPOD.sql` copies every non-null `POD.salespersonId` into `dbo.SalesPersonPOD`. Review the resulting rows for PODs that legitimately had no owner.
- **Do not guess POD → station mappings.** The upgrade leaves `POD.PODStationId` values untouched. Use **Admin → POD station assignments** to apply stations explicitly.
- **Resolve conflicting reporting lines manually.** Data inspection found records that violate the designation matrix and were intentionally left unchanged:
  - Bipindra Neupane — more than one CXO reference.
  - Ujjwol Neupane — more than one CXO reference.
  - Hritik Mishra — more than one manager reference.

  The application validates new or edited manager assignments, but it does not silently rewrite existing ones. Correct these records through the salesperson edit screen (or a reviewed SQL change) before relying on team-scoped views.
- **Dealer names were not invented.** Where no verified dealer name was available, `Dealer.Name` equals `Dealer.Code`.

## Security notes

- Parameterized SQL only (`Microsoft.Data.SqlClient` with `SqlParameter`).
- Anti-forgery tokens on all state-changing forms.
- Password rules: minimum 8 characters with upper/lower/digit; lockout after 5 failed attempts for 5 minutes.
- Authentication cookie: HTTP-only, sliding expiration, 2-hour lifetime.
- Generic forgot-password response so account existence is not revealed.
- No shared passwords; invitations use Identity reset tokens.
- Activity history is append-only in `dbo.AuditLog`; secrets, password hashes, and invitation tokens are never written there.
- Administrative authorization is enforced on controllers, not only by hiding navigation.
"# SalesHierarchyT" 
