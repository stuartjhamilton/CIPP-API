# CIPP-API Developer Guide for AI Assistants

## Project Overview

CIPP-API is the backend for the **CIPP (CyberDrain Improved Partner Portal)** project — a Microsoft 365 multi-tenant management platform for MSPs (Managed Service Providers). It is built as an **Azure Functions App using PowerShell 7.4**, leveraging Azure Durable Functions for background orchestration.

- **Current version**: 10.2.1
- **Runtime**: Azure Functions v4 + PowerShell 7.4
- **Language**: PowerShell (100%)
- **Infrastructure**: Azure Table Storage, Azure Queue Storage, Azure Key Vault, Azure Static Web Apps (frontend)

---

## Repository Structure

```
CIPP-API/
├── CIPPHttpTrigger/          # Azure Function binding for all HTTP requests (routes to CippEntrypoints)
├── CIPPQueueTrigger/         # Azure Function binding for queue-triggered work
├── CIPPOrchestrator/         # Durable Functions orchestrator binding
├── CIPPActivityFunction/     # Durable activity function binding
├── CIPPTimer/                # Timer trigger binding (runs scheduled tasks)
├── CIPPTimers.json           # Timer schedule definitions (cron expressions, commands)
├── Modules/
│   ├── CIPPCore/             # Main module: all business logic and entrypoints
│   │   ├── CIPPCore.psd1     # Module manifest (requires PowerShell 7.0+)
│   │   ├── CIPPCore.psm1     # Module loader (dot-sources Public/ and Private/)
│   │   ├── Public/           # ~1393 exported functions
│   │   │   ├── Entrypoints/  # ~583 functions (HTTP, Activity, Timer, Orchestrator)
│   │   │   ├── Standards/    # ~177 standards enforcement functions
│   │   │   ├── GraphHelper/  # Graph API, token management, logging, table access
│   │   │   ├── GraphRequests/# Generic Graph request helpers
│   │   │   ├── Authentication/# Access control, role checking, API auth
│   │   │   ├── CippQueue/    # Queue task management
│   │   │   ├── Alerts/       # Alert sending functions
│   │   │   ├── AuditLogs/    # Audit log processing
│   │   │   ├── DeltaQueries/ # Microsoft Graph delta query support
│   │   │   ├── Webhooks/     # Webhook subscription management
│   │   │   ├── TenantGroups/ # Tenant group management
│   │   │   ├── Tools/        # Utility functions
│   │   │   └── CustomData/   # Custom data attributes
│   │   ├── Private/          # Internal helpers (not exported)
│   │   ├── lib/              # .NET assemblies (NCrontab.Advanced.dll, AppInsights)
│   │   └── build.psd1        # ModuleBuilder config
│   ├── CippEntrypoints/      # Trigger receivers (HTTP, Queue, Orchestrator, Activity, Timer)
│   ├── AzBobbyTables/        # Azure Table Storage PowerShell module
│   ├── CippExtensions/       # Optional extension integrations (NinjaOne, Halo, etc.)
│   ├── DNSHealth/            # DNS health check module
│   ├── HuduAPI/              # Hudu documentation integration
│   └── MicrosoftTeams/       # Teams PowerShell module
├── Shared/
│   └── AppInsights/          # Application Insights .dll for telemetry
├── Config/                   # Static JSON config files
│   ├── cipp-roles.json       # Role definitions (readonly/editor/admin/superadmin)
│   ├── standards.json        # Standards definitions
│   └── *.json                # Templates (CA, Intune, BPA, Transport Rules)
├── Tests/                    # Pester-style test definitions
├── Tools/                    # Build and maintenance scripts
├── profile.ps1               # Azure Functions startup profile (module loading, telemetry init)
├── host.json                 # Azure Functions host configuration
├── requirements.psd1         # Intentionally empty (modules bundled in Modules/)
├── docker-compose.yml        # Local dev with Azurite + nginx load balancer
├── Dockerfile                # Based on mcr.microsoft.com/azure-functions/powershell:4-powershell7.4
└── version_latest.txt        # Current version string
```

---

## Architecture & Request Flow

### HTTP Request Lifecycle

```
HTTP Request
    → CIPPHttpTrigger/function.json  (route: {*CIPPEndpoint})
    → Receive-CippHttpTrigger        (CippEntrypoints.psm1)
        → handles $batch processing (up to 20 requests in parallel)
        → New-CippCoreRequest        (per-request routing)
            → Test-CIPPAccess        (auth + RBAC check)
            → Invoke-{CIPPEndpoint}  (the actual function)
            → HttpResponseContext    (returned to caller)
```

**Key point**: The URL route `{*CIPPEndpoint}` maps to the function name `Invoke-{CIPPEndpoint}`. For example, a request to `/api/ListUsers` calls `Invoke-ListUsers`.

### Batch API Support
Requests to `/api/$batch` are processed in parallel (up to 20 requests, throttle limit 10). Each batch item follows the same routing, modeled after the Microsoft Graph API `$batch` format.

### Queue Trigger Flow
```
Azure Queue (cippqueue)
    → CIPPQueueTrigger/function.json
    → Receive-CippQueueTrigger       (CippEntrypoints.psm1)
        → { Cmdlet: "FunctionName", Parameters: {...} }
        → & $Cmdlet @Parameters
```

### Durable Orchestration Flow
```
Timer / HTTP trigger starts orchestration
    → Receive-CippOrchestrationTrigger
        → CIPPActivityFunction (Receive-CippActivityTrigger)
            → Push-{FunctionName} -Item $Item
            → Updates CippQueueTasks table
```

---

## Function Naming Conventions

| Prefix | Usage | Example |
|--------|-------|---------|
| `Invoke-List*` | Read/query data (HTTP GET-style) | `Invoke-ListUsers` |
| `Invoke-Exec*` | Execute an action (HTTP POST-style) | `Invoke-ExecResetPassword` |
| `Invoke-Get*` | Retrieve specific data | `Invoke-GetVersion` |
| `Invoke-Add*` | Add/create resources | `Invoke-AddTestReport` |
| `Invoke-Remove*` / `Invoke-Delete*` | Remove resources | `Invoke-DeleteTestReport` |
| `Push-*` | Activity trigger entrypoints (called by orchestrator) | `Push-CIPPDBCacheData` |
| `Start-*` | Timer function entrypoints | `Start-DurableCleanup` |
| `Get-CIPP*` | Internal get helpers | `Get-CIPPTable` |
| `Set-CIPP*` | Internal set/write helpers | `Set-CIPPDBCacheUsers` |
| `New-CIPP*` | Internal create helpers | `New-GraphGetRequest` |
| `Invoke-CIPPStandard*` | Standards enforcement functions | `Invoke-CIPPStandardAuditLog` |

---

## Writing HTTP Entrypoint Functions

All HTTP endpoint functions follow this template:

```powershell
function Invoke-ExecMyFunction {
    <#
    .FUNCTIONALITY
        Entrypoint
    .ROLE
        CIPP.SomeCategory.ReadWrite
    #>
    [CmdletBinding()]
    param($Request, $TriggerMetadata)

    $APIName = $Request.Params.CIPPEndpoint
    $Headers = $Request.Headers

    # Access request data
    $TenantFilter = $Request.Query.TenantFilter ?? $Request.Body.TenantFilter
    $SomeParam    = $Request.Query.SomeParam ?? $Request.Body.SomeParam

    # Log access
    Write-LogMessage -headers $Headers -API $APIName -message 'Accessed this API' -Sev 'Debug'

    # Do work...
    $Results = Some-CIPPFunction -TenantFilter $TenantFilter

    # Return response
    return ([HttpResponseContext]@{
        StatusCode = [HttpStatusCode]::OK
        Body       = $Results
    })
}
```

### `.FUNCTIONALITY` Values
- `Entrypoint` — Standard authenticated endpoint
- `Entrypoint,AnyTenant` — Available to all tenants regardless of role filtering
- `Internal` — Not an HTTP endpoint, not routed

### `.ROLE` Format
Roles follow the pattern `Domain.Resource.Permission`:
- Examples: `CIPP.AppSettings.Read`, `CIPP.AppSettings.ReadWrite`, `CIPP.SuperAdmin.ReadWrite`
- Special role `Public` skips auth entirely
- Roles are validated against `Config/cipp-roles.json` and user's assigned role

---

## Standards Functions

Standards are remediation/audit functions called by the tenant standards engine. They live in `Modules/CIPPCore/Public/Standards/Invoke-CIPPStandard*.ps1`.

```powershell
function Invoke-CIPPStandardMyStandard {
    <#
    .FUNCTIONALITY
        Internal
    .COMPONENT
        (APIName) MyStandard
    .SYNOPSIS
        (Label) Human-readable label
    .DESCRIPTION
        (Helptext) Short description shown in UI
        (DocsDescription) Longer description for docs
    .NOTES
        CAT
            Global Standards
        TAG
            "optional-tag"
        IMPACT
            Low Impact    # or Medium Impact / High Impact
        ADDEDDATE
            2024-01-01
        RECOMMENDEDBY
            "CIS"
    #>
    param($Tenant, $Settings)

    if ($Settings.remediate -eq $true) {
        # Apply the standard
        try {
            # ... remediation logic
            Write-LogMessage -API 'Standards' -tenant $Tenant -message 'Standard applied.' -sev Info
        } catch {
            $ErrorMessage = Get-NormalizedError -Message $_.Exception.Message
            Write-LogMessage -API 'Standards' -tenant $Tenant -message "Failed: $ErrorMessage" -sev Error
        }
    }

    if ($Settings.alert -eq $true) {
        # Check and alert on non-compliance
    }

    if ($Settings.report -eq $true) {
        # Report compliance state
        Add-CIPPBPAField -FieldName 'MyStandard' -FieldValue $compliant -StoreAs bool -Tenant $Tenant
    }
}
```

---

## Activity Trigger Functions (Push-*)

Activity functions are invoked by the durable orchestrator. They receive an `$Item` hashtable:

```powershell
function Push-MyActivityFunction {
    <#
    .FUNCTIONALITY
        Entrypoint
    #>
    param($Item)

    $TenantFilter = $Item.TenantFilter
    $QueueId      = $Item.QueueId

    # Do work across all tenants or specific tenant
    $Tenants = Get-Tenants
    foreach ($Tenant in $Tenants) {
        # ... work per tenant
    }
}
```

---

## Timer Functions (Start-*)

Timer functions are defined in `CIPPTimers.json` with cron schedules. They receive a `$Timer` object:

```powershell
function Start-MyTimer {
    <#
    .FUNCTIONALITY
        Entrypoint
    #>
    param($Timer)
    # Execute scheduled work
    # Can return an orchestrator ID (GUID) or nothing
}
```

Timer schedule entry in `CIPPTimers.json`:
```json
{
  "Id": "unique-guid",
  "Command": "Start-MyTimer",
  "Description": "Human readable description",
  "Cron": "0 */15 * * * *",
  "Priority": 1,
  "RunOnProcessor": true
}
```

---

## Key Helper Functions

### Table Storage (Azure Table Storage via AzBobbyTables)

```powershell
# Get a table context
$Table = Get-CIPPTable -tablename 'TableName'

# Read entities
$Entities = Get-CIPPAzDataTableEntity @Table -Filter "PartitionKey eq 'key'"

# Write/upsert entities
Add-CIPPAzDataTableEntity @Table -Entity @{
    PartitionKey = 'partition'
    RowKey       = 'row'
    SomeField    = 'value'
} -Force

# Update
Update-CIPPDbItem -TableName 'TableName' -Item $item
```

**Common table names**: `Config`, `Tenants`, `CippLogs`, `CippQueueTasks`, `CippReportingDB`, `CIPPTimers`, `Version`

### Graph API Requests

```powershell
# GET request (auto-paginated)
$Result = New-GraphGetRequest -uri 'https://graph.microsoft.com/v1.0/users' -tenantid $TenantFilter

# POST request
$Result = New-GraphPostRequest -uri 'https://graph.microsoft.com/v1.0/...' -tenantid $TenantFilter -body $Body

# Exchange Online request
$Result = New-ExoRequest -tenantid $TenantFilter -cmdlet 'Get-Mailbox' -cmdParams @{ResultSize = 'Unlimited'}

# Bulk Graph request (parallel)
$Result = New-GraphBulkRequest -Requests $RequestsArray -tenantid $TenantFilter
```

### Logging

```powershell
# Standard log (writes to CippLogs table)
Write-LogMessage -API 'FunctionName' -tenant $TenantFilter -message 'What happened' -sev Info
# Severity values: Debug, Info, Warn, Error, Critical

# Alert log
Write-AlertMessage -tenant $TenantFilter -message 'Alert text'
```

### Authentication & Token

```powershell
# Get a Graph token for a tenant
$Token = Get-GraphToken -tenantid $TenantFilter

# Authenticate the function app
Get-CIPPAuthentication
```

### Tenant Management

```powershell
# Get all active tenants
$Tenants = Get-Tenants

# Get a specific tenant
$Tenant = Get-Tenants -TenantFilter 'contoso.onmicrosoft.com'

# Get all tenants including errors
$AllTenants = Get-Tenants -IncludeAll
```

### Queue Management

```powershell
# Add to CIPP queue
Add-CippQueueMessage -Message @{ Cmdlet = 'FunctionName'; Parameters = @{} }

# Track queue task status
$Task = Set-CippQueueTask -QueueId $QueueId -Name 'Task Name' -Status 'Running'
```

---

## Authentication & Authorization

### Auth Flow
1. Azure Static Web Apps (SWA) handles identity via AAD (`x-ms-client-principal-*` headers)
2. `Test-CIPPAccess` checks the function's `.ROLE` annotation against the user's assigned CIPP role
3. Direct API access (non-SWA) supported via API clients registered in `ApiClients` table
4. IP allowlisting supported per role configuration

### Role Hierarchy (from `Config/cipp-roles.json`)
| Role | Permissions |
|------|-------------|
| `readonly` | `*.Read` only, no SuperAdmin |
| `editor` | `*.Read` + `*.ReadWrite`, no Admin/SuperAdmin/AppSettings/Standards.ReadWrite |
| `admin` | Everything except `CIPP.SuperAdmin.*` |
| `superadmin` | Everything |

### Environment Variables for Auth
- `ApplicationID` — Azure AD App Registration client ID
- `ApplicationSecret` — App secret
- `RefreshToken` — OAuth refresh token for delegated auth
- `TenantID` — Home tenant ID (partner tenant)

---

## Environment Variables

| Variable | Purpose |
|----------|---------|
| `AzureWebJobsStorage` | Azure Storage connection string (use `UseDevelopmentStorage=true` for local dev) |
| `ApplicationID` | SAM App Registration client ID |
| `ApplicationSecret` | SAM App client secret |
| `RefreshToken` | OAuth refresh token for delegated permissions |
| `TenantID` | Partner/home tenant ID |
| `WEBSITE_HOSTNAME` | Function app hostname |
| `WEBSITE_SITE_NAME` | Function app name |
| `WEBSITE_SKU` | App service plan SKU (Premium enables FanOut durable mode) |
| `APPLICATIONINSIGHTS_CONNECTION_STRING` | App Insights connection string |
| `APPINSIGHTS_INSTRUMENTATIONKEY` | App Insights key (fallback) |
| `DebugMode` | Set to `true` to enable Debug log messages |
| `CIPP_PROCESSOR` | Set to `true` on processor-only instances |
| `NonLocalHostAzurite` | Set to `true` when using non-local Azurite in Docker |

---

## Local Development

### Prerequisites
- Docker + Docker Compose
- Azure Functions Core Tools v4
- PowerShell 7.4+

### Running Locally with Docker

```bash
# Copy and configure environment
cp .env.example .env
# Edit .env with your values

# Start all services (Azurite + 3 function app replicas + nginx load balancer)
docker-compose up

# API available at http://localhost:7071/api/{endpoint}
```

The Docker setup:
- **Azurite**: Local Azure Storage emulator (ports 10000-10002)
- **cippapi**: 3 replicas of the function app
- **nginx**: Load balancer on port 7071

### Running Without Docker (Azure Functions Core Tools)

```bash
# Create local.settings.json with required env vars
func start
```

---

## Adding a New HTTP Endpoint

1. Create a new file in the appropriate subdirectory under `Modules/CIPPCore/Public/Entrypoints/HTTP Functions/`:
   - `CIPP/Core/` — Core CIPP management
   - `CIPP/Settings/` — CIPP settings
   - `Identity/Administration/` — User/group management
   - `Identity/Reports/` — Identity reports
   - `Tenant/Administration/` — Tenant management
   - `Tenant/Standards/` — Standards management
   - `Email-Exchange/` — Exchange/email functions
   - `Endpoint/` — Intune/endpoint management
   - `Security/` — Security functions
   - `Teams-Sharepoint/` — Teams and SharePoint
   - `Tools/` — Utility tools

2. Name the file `Invoke-{EndpointName}.ps1`

3. Follow the function template (see "Writing HTTP Entrypoint Functions" above)

4. The endpoint is automatically available at `/api/{EndpointName}` — no registration needed

5. The function permissions cache at `lib/data/function-permissions.json` may need to be regenerated for access control. Run the Tools script to rebuild it if needed.

---

## Module Loading

`profile.ps1` runs on every cold start and:
1. Loads Application Insights SDK if configured
2. Imports `CIPPCore`, `CippExtensions`, `AzBobbyTables` modules
3. Initializes global `TelemetryClient` for App Insights
4. Sets environment variables from Azure Function App settings

Modules are bundled locally in `Modules/` — **managed dependency is disabled** in `host.json`.

---

## Telemetry & Performance

- App Insights integration via `Measure-CippTask` wrapper in `New-CippCoreRequest`
- HTTP request timing tracked in `$HttpTimings` dictionary per request
- `Write-CippFunctionStats` records function execution statistics
- `Write-Debug` outputs timing summaries: `#### HTTP Request Timings ####`

---

## Key Patterns & Conventions

### PowerShell Style
- Use approved PowerShell verbs (`Get-`, `Set-`, `New-`, `Remove-`, `Add-`, `Test-`, `Invoke-`)
- Use `[CmdletBinding()]` on all functions
- Use `Write-Information` for standard logging within functions, `Write-LogMessage` for persistent logs
- Use `Write-Warning` for non-fatal issues that should be visible
- Prefer `ConvertTo-Json -Depth 20 -Compress` for serialization

### Error Handling
```powershell
try {
    # ... operation
} catch {
    $ErrorMessage = Get-NormalizedError -Message $_.Exception.Message
    Write-LogMessage -API $APIName -tenant $TenantFilter -message "Failed: $ErrorMessage" -sev Error
}
```

### Tenant Filtering
- Always accept `$TenantFilter` as either `$Request.Query.TenantFilter` or `$Request.Body.TenantFilter`
- Pass `TenantFilter` to all Graph/Exchange API calls
- Use `AllTenants` as the filter value to iterate all managed tenants

### Response Format
```powershell
# Success
return ([HttpResponseContext]@{
    StatusCode = [HttpStatusCode]::OK
    Body       = $Results
})

# Error
return ([HttpResponseContext]@{
    StatusCode = [HttpStatusCode]::InternalServerError
    Body       = "Error message"
})
```

### Feature Flags
```powershell
# Check if a feature is enabled
$Flag = Get-CIPPFeatureFlag -FlagName 'MyFeature'

# Set a feature flag
Set-CIPPFeatureFlag -FlagName 'MyFeature' -Enabled $true
```

---

## Writing Alert Functions

Alert functions live in `Modules/CIPPCore/Public/Alerts/` and are named `Get-CIPPAlert<Something>.ps1`. See `.github/agents/CIPP-Alert-Agent.md` for the full authoring guide.

Key rules:
- Use `Write-AlertTrace` to emit alert results (not `Write-Output` or `return`)
- Use `New-GraphGetRequest` / `New-ExoRequest` for API calls — never inline raw HTTP
- Use `Test-CIPPStandardLicense` for any license/SKU capability checks
- Do **not** modify the module manifest — functions are auto-loaded from `Public/`
- No CodeQL; use `PSScriptAnalyzer` for linting

```powershell
function Get-CIPPAlertMyCondition {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory = $true)]
        $TenantFilter
    )

    try {
        # Check license if required
        $HasLicense = Test-CIPPStandardLicense -StandardName 'MyCheck' -TenantFilter $TenantFilter `
            -RequiredCapabilities @('AAD_PREMIUM')
        if (-not $HasLicense) { return }

        # Check condition via Graph
        $Data = New-GraphGetRequest -uri 'https://graph.microsoft.com/v1.0/...' -tenantid $TenantFilter

        if ($Data.SomeProperty -ne $ExpectedValue) {
            Write-AlertTrace -cmdletName $MyInvocation.MyCommand -tenantFilter $TenantFilter -data $Data
        }
    } catch {
        Write-LogMessage -API 'Alerts' -tenant $TenantFilter -message "Alert check failed: $($_.Exception.Message)" -sev Error
    }
}
```

---

## Adding a New Standard — Frontend Integration

When creating a new standard, you must also add an entry to `Config/standards.json`. The JSON format maps to the standard function via the naming convention `standards.<APIName>` (from `.COMPONENT (APIName)`).

Example entry for `Invoke-CIPPStandardMyStandard`:
```json
{
  "name": "standards.MyStandard",
  "cat": "Global Standards",
  "tag": [],
  "helpText": "Short description shown in the UI.",
  "docsDescription": "Longer description for documentation.",
  "executiveText": "Executive-friendly summary of what this standard does.",
  "addedComponent": [],
  "label": "My Standard Display Name",
  "impact": "Low Impact",
  "impactColour": "info",
  "addedDate": "2024-01-01",
  "powershellEquivalent": "Set-SomeCmdlet",
  "recommendedBy": []
}
```

If your standard has configurable inputs, add them in `addedComponent`. Each component maps to `$Settings.<PropertyName>` in your function:
```json
"addedComponent": [
  {
    "type": "textField",
    "name": "standards.MyStandard.SomeOption",
    "label": "Some Option Label",
    "required": false
  }
]
```

When submitting a PR for a new standard, include the `standards.json` payload in the PR description so the frontend team can update the CIPP frontend repository.

---

## Testing

Tests live in `Tests/` organized by category (Alerts, Standards, Endpoint). Test files follow the pattern `*.Tests.ps1`. The testing infrastructure uses `Invoke-ExecTestRun`, `Invoke-ListTests`, and `Add-CippTestResult` functions.

---

## Specialized Agent Files

The repository ships GitHub Copilot custom agents in `.github/agents/` for common engineering tasks. Reference these before writing alerts or standards:

| File | Purpose |
|------|---------|
| `.github/agents/CIPP-Alert-Agent.md` | Guide for creating `Get-CIPPAlert*` functions |
| `.github/agents/CIPP-Standards-Agent.md` | Guide for creating `Invoke-CIPPStandard*` functions + `standards.json` entries |

---

## Extension Modules

`CippExtensions` provides integrations with third-party platforms:
- **NinjaOne** — RMM platform integration
- **HaloPSA** — PSA platform integration
- **Hudu** — Documentation platform integration

Extensions are loaded optionally (`-ErrorAction SilentlyContinue`) in the batch processing parallel blocks.

---

## Important Files Quick Reference

| File | Purpose |
|------|---------|
| `profile.ps1` | App startup: module loading, telemetry init |
| `host.json` | Azure Functions runtime config (timeout: 10min, durable settings) |
| `CIPPTimers.json` | Timer schedule definitions |
| `Config/cipp-roles.json` | RBAC role permission matrices |
| `Config/standards.json` | Standards definitions for UI |
| `Modules/CippEntrypoints/CippEntrypoints.psm1` | All 5 trigger entry functions |
| `Modules/CIPPCore/Public/Entrypoints/HTTP Functions/New-CippCoreRequest.ps1` | HTTP routing + auth enforcement |
| `Modules/CIPPCore/Public/Authentication/Test-CIPPAccess.ps1` | RBAC access control logic |
| `Modules/CIPPCore/Public/GraphHelper/Get-GraphToken.ps1` | OAuth token acquisition |
| `Modules/CIPPCore/Public/GraphHelper/Get-CIPPTable.ps1` | Azure Table Storage context factory |
| `Modules/CIPPCore/Public/GraphHelper/Write-LogMessage.ps1` | Structured logging to table storage |
| `Modules/CIPPCore/Public/GraphHelper/New-GraphGetRequest.ps1` | Paginated Graph API GET |
