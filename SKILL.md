# Contentstack PowerShell Skill

## Overview

This skill enables full management of Contentstack stacks via PowerShell on Windows using `Invoke-WebRequest`. It covers both the **Content Delivery API (CDA)** for reading published content and the **Content Management API (CMA)** for creating, updating, and publishing content.

Always use `-UseBasicParsing` with every `Invoke-WebRequest` call to avoid the CVE-2025-54100 security prompt.

---

## Configuration & Authentication

Before running any commands, set up your credentials as PowerShell variables:

```powershell
# === REQUIRED: Set these before using any function ===

# Stack API Key (from Contentstack > Stack Settings > API Credentials)
$CS_API_KEY = "your_stack_api_key"

# Management Token (CMA write operations — from Stack Settings > Tokens > Management Tokens)
$CS_MANAGEMENT_TOKEN = "your_management_token"

# Auth Token (user session token — from Login API or Contentstack dashboard)
$CS_AUTH_TOKEN = "your_authtoken"

# Delivery Token (CDA read operations — from Stack Settings > Tokens > Delivery Tokens)
$CS_DELIVERY_TOKEN = "your_delivery_token"

# Region base URLs — pick ONE matching your stack's region:
$CS_CMA_URL = "https://api.contentstack.io"      # North America (default)
# $CS_CMA_URL = "https://eu-api.contentstack.com"  # Europe
# $CS_CMA_URL = "https://azure-na-api.contentstack.com"  # Azure NA

$CS_CDA_URL = "https://cdn.contentstack.io"       # North America CDN
# $CS_CDA_URL = "https://eu-cdn.contentstack.com"   # Europe CDN

# Default locale and environment
$CS_LOCALE    = "en-us"
$CS_ENV       = "production"
```

---

## Core Helper Functions

Paste these helper functions into your PowerShell session once. All resource-specific functions call these.

```powershell
function Invoke-CMA {
    <#
    .SYNOPSIS
    Sends a Content Management API request.
    .PARAMETER Method   HTTP method: GET, POST, PUT, DELETE
    .PARAMETER Path     API path starting with /v3/...
    .PARAMETER Body     Hashtable payload (auto-converted to JSON)
    .PARAMETER Query    Hashtable of query string parameters
    .PARAMETER UseAuth  Use authtoken instead of management token
    #>
    param(
        [string]$Method = "GET",
        [string]$Path,
        [hashtable]$Body = $null,
        [hashtable]$Query = @{},
        [switch]$UseAuth
    )

    # Build query string
    $qs = ""
    if ($Query.Count -gt 0) {
        $parts = $Query.GetEnumerator() | ForEach-Object { "$($_.Key)=$([uri]::EscapeDataString($_.Value))" }
        $qs = "?" + ($parts -join "&")
    }

    $uri = "$CS_CMA_URL$Path$qs"

    $headers = @{
        "api_key"      = $CS_API_KEY
        "Content-Type" = "application/json"
    }
    if ($UseAuth) {
        $headers["authtoken"] = $CS_AUTH_TOKEN
    } else {
        $headers["authorization"] = $CS_MANAGEMENT_TOKEN
    }

    $params = @{
        UseBasicParsing = $true
        Uri             = $uri
        Method          = $Method
        Headers         = $headers
    }

    if ($Body) {
        $params["Body"] = ($Body | ConvertTo-Json -Depth 20 -Compress)
    }

    try {
        $response = Invoke-WebRequest @params
        return $response.Content | ConvertFrom-Json
    } catch {
        $err = $_.Exception.Response
        if ($err) {
            $reader = [System.IO.StreamReader]::new($err.GetResponseStream())
            $errBody = $reader.ReadToEnd() | ConvertFrom-Json
            Write-Error "CMA Error $($err.StatusCode): $($errBody | ConvertTo-Json -Depth 5)"
        } else {
            Write-Error "Request failed: $_"
        }
    }
}

function Invoke-CDA {
    <#
    .SYNOPSIS
    Sends a Content Delivery API request (read-only, published content).
    .PARAMETER Path     API path starting with /v3/...
    .PARAMETER Query    Hashtable of query string parameters
    #>
    param(
        [string]$Path,
        [hashtable]$Query = @{}
    )

    $Query["environment"] = $CS_ENV

    $qs = ""
    if ($Query.Count -gt 0) {
        $parts = $Query.GetEnumerator() | ForEach-Object { "$($_.Key)=$([uri]::EscapeDataString($_.Value))" }
        $qs = "?" + ($parts -join "&")
    }

    $uri = "$CS_CDA_URL$Path$qs"

    $headers = @{
        "api_key"       = $CS_API_KEY
        "access_token"  = $CS_DELIVERY_TOKEN
    }

    try {
        $response = Invoke-WebRequest -UseBasicParsing -Uri $uri -Method GET -Headers $headers
        return $response.Content | ConvertFrom-Json
    } catch {
        Write-Error "CDA Error: $_"
    }
}
```

---

## STACKS

### Get stack details
```powershell
function Get-CSStack {
    Invoke-CMA -Method GET -Path "/v3/stacks"
}
```

### Update stack
```powershell
function Update-CSStack {
    param(
        [string]$Name,
        [string]$Description = ""
    )
    $body = @{ stack = @{ name = $Name; description = $Description } }
    Invoke-CMA -Method PUT -Path "/v3/stacks" -Body $body
}
```

### Get all stack users
```powershell
function Get-CSStackUsers {
    Invoke-CMA -Method GET -Path "/v3/stacks/users"
}
```

### Share stack with a user
```powershell
function Share-CSStack {
    param(
        [string]$Email,
        [string]$RoleUID   # UID of the role to assign
    )
    $body = @{
        emails = @($Email)
        roles  = @{ $Email = @($RoleUID) }
    }
    Invoke-CMA -Method POST -Path "/v3/stacks/share" -Body $body
}
```

### Unshare stack from a user
```powershell
function Unshare-CSStack {
    param([string]$Email)
    $body = @{ email = $Email }
    Invoke-CMA -Method POST -Path "/v3/stacks/unshare" -Body $body
}
```

### Get stack settings
```powershell
function Get-CSStackSettings {
    Invoke-CMA -Method GET -Path "/v3/stacks/settings"
}
```

### Update stack settings
```powershell
function Update-CSStackSettings {
    param([hashtable]$Settings)
    $body = @{ stack_settings = $Settings }
    Invoke-CMA -Method POST -Path "/v3/stacks/settings" -Body $body
}
```

---

## BRANCHES

### List all branches
```powershell
function Get-CSBranches {
    Invoke-CMA -Method GET -Path "/v3/stacks/branches"
}
```

### Get a single branch
```powershell
function Get-CSBranch {
    param([string]$BranchUID)
    Invoke-CMA -Method GET -Path "/v3/stacks/branches/$BranchUID"
}
```

### Create a branch
```powershell
function New-CSBranch {
    param(
        [string]$UID,           # new branch UID e.g. "release"
        [string]$Source = "main"
    )
    $body = @{ branch = @{ uid = $UID; source = $Source } }
    Invoke-CMA -Method POST -Path "/v3/stacks/branches" -Body $body
}
```

### Delete a branch
```powershell
function Remove-CSBranch {
    param([string]$BranchUID)
    Invoke-CMA -Method DELETE -Path "/v3/stacks/branches/$BranchUID" -Query @{ force = "true" }
}
```

### Compare two branches
```powershell
function Compare-CSBranches {
    param([string]$CompareBranch)
    Invoke-CMA -Method GET -Path "/v3/stacks/branches_compare" -Query @{ compare_branch = $CompareBranch }
}
```

### Merge branches
```powershell
function Merge-CSBranches {
    param(
        [string]$CompareBranch,
        [string]$MergeStrategy = "merge_prefer_base",  # or merge_prefer_compare, merge_new
        [array]$ItemStrategies = @()   # optional per-item overrides
    )
    $body = @{ item_merge_strategies = $ItemStrategies }
    $query = @{
        compare_branch         = $CompareBranch
        default_merge_strategy = $MergeStrategy
    }
    Invoke-CMA -Method POST -Path "/v3/stacks/branches_merge" -Body $body -Query $query
}
```

---

## CONTENT TYPES

### List all content types
```powershell
function Get-CSContentTypes {
    param([switch]$IncludeCount, [int]$Skip = 0, [int]$Limit = 100)
    $q = @{ skip = "$Skip"; limit = "$Limit" }
    if ($IncludeCount) { $q["include_count"] = "true" }
    Invoke-CMA -Method GET -Path "/v3/content_types" -Query $q
}
```

### Get a single content type
```powershell
function Get-CSContentType {
    param([string]$ContentTypeUID)
    Invoke-CMA -Method GET -Path "/v3/content_types/$ContentTypeUID"
}
```

### Create a content type
```powershell
function New-CSContentType {
    param(
        [string]$Title,
        [string]$UID,
        [array]$Schema,            # Array of field definition hashtables
        [hashtable]$Options = @{}
    )
    $body = @{
        content_type = @{
            title   = $Title
            uid     = $UID
            schema  = $Schema
            options = $Options
        }
    }
    Invoke-CMA -Method POST -Path "/v3/content_types" -Body $body
}
```

**Example — create a simple content type with title and URL fields:**
```powershell
$schema = @(
    @{ display_name="Title"; uid="title"; data_type="text";
       field_metadata=@{_default=$true}; mandatory=$true; unique=$false; multiple=$false },
    @{ display_name="URL"; uid="url"; data_type="text";
       field_metadata=@{_default=$true}; unique=$false; multiple=$false }
)
New-CSContentType -Title "Blog Post" -UID "blog_post" -Schema $schema `
    -Options @{ title="title"; publishable=$true; is_page=$true; singleton=$false; sub_title=@("url") }
```

### Update a content type
```powershell
function Update-CSContentType {
    param(
        [string]$ContentTypeUID,
        [string]$Title,
        [array]$Schema,
        [hashtable]$Options = @{}
    )
    $body = @{
        content_type = @{
            title   = $Title
            uid     = $ContentTypeUID
            schema  = $Schema
            options = $Options
        }
    }
    Invoke-CMA -Method PUT -Path "/v3/content_types/$ContentTypeUID" -Body $body
}
```

### Delete a content type
```powershell
function Remove-CSContentType {
    param([string]$ContentTypeUID)
    Invoke-CMA -Method DELETE -Path "/v3/content_types/$ContentTypeUID"
}
```

### Get all references of a content type
```powershell
function Get-CSContentTypeReferences {
    param([string]$ContentTypeUID)
    Invoke-CMA -Method GET -Path "/v3/content_types/$ContentTypeUID/references"
}
```

### Export / Import content types
```powershell
function Export-CSContentType {
    param([string]$ContentTypeUID)
    Invoke-CMA -Method GET -Path "/v3/content_types/$ContentTypeUID/export"
}

function Import-CSContentType {
    param([string]$JsonFilePath)
    $content = Get-Content $JsonFilePath -Raw
    $uri = "$CS_CMA_URL/v3/content_types/import"
    $headers = @{ "api_key"="$CS_API_KEY"; "authorization"="$CS_MANAGEMENT_TOKEN" }
    $response = Invoke-WebRequest -UseBasicParsing -Uri $uri -Method POST -Headers $headers `
        -ContentType "application/json" -Body $content
    $response.Content | ConvertFrom-Json
}
```

---

## GLOBAL FIELDS

### List all global fields
```powershell
function Get-CSGlobalFields {
    Invoke-CMA -Method GET -Path "/v3/global_fields"
}
```

### Get a single global field
```powershell
function Get-CSGlobalField {
    param([string]$GlobalFieldUID)
    Invoke-CMA -Method GET -Path "/v3/global_fields/$GlobalFieldUID"
}
```

### Create a global field
```powershell
function New-CSGlobalField {
    param(
        [string]$Title,
        [string]$UID,
        [array]$Schema
    )
    $body = @{ global_field = @{ title=$Title; uid=$UID; schema=$Schema } }
    Invoke-CMA -Method POST -Path "/v3/global_fields" -Body $body
}
```

### Update a global field
```powershell
function Update-CSGlobalField {
    param(
        [string]$GlobalFieldUID,
        [string]$Title,
        [array]$Schema
    )
    $body = @{ global_field = @{ title=$Title; uid=$GlobalFieldUID; schema=$Schema } }
    Invoke-CMA -Method PUT -Path "/v3/global_fields/$GlobalFieldUID" -Body $body
}
```

### Delete a global field
```powershell
function Remove-CSGlobalField {
    param([string]$GlobalFieldUID)
    Invoke-CMA -Method DELETE -Path "/v3/global_fields/$GlobalFieldUID" -Query @{ force="true" }
}
```

---

## ENTRIES

### List all entries in a content type
```powershell
function Get-CSEntries {
    param(
        [string]$ContentTypeUID,
        [string]$Locale      = $CS_LOCALE,
        [int]$Skip           = 0,
        [int]$Limit          = 100,
        [switch]$IncludeCount,
        [string]$Query       = "",   # JSON query string e.g. '{"title":"Hello"}'
        [string]$Asc         = "",   # field to sort ascending
        [string]$Desc        = "",   # field to sort descending
        [switch]$IncludeAll  # include all references
    )
    $q = @{ locale="$Locale"; skip="$Skip"; limit="$Limit" }
    if ($IncludeCount) { $q["include_count"] = "true" }
    if ($Query)        { $q["query"] = $Query }
    if ($Asc)          { $q["asc"] = $Asc }
    if ($Desc)         { $q["desc"] = $Desc }
    if ($IncludeAll)   { $q["include_all"] = "true" }
    Invoke-CMA -Method GET -Path "/v3/content_types/$ContentTypeUID/entries" -Query $q
}
```

### Get a single entry
```powershell
function Get-CSEntry {
    param(
        [string]$ContentTypeUID,
        [string]$EntryUID,
        [string]$Locale = $CS_LOCALE
    )
    Invoke-CMA -Method GET -Path "/v3/content_types/$ContentTypeUID/entries/$EntryUID" `
        -Query @{ locale=$Locale }
}
```

### Create an entry
```powershell
function New-CSEntry {
    param(
        [string]$ContentTypeUID,
        [hashtable]$Fields,          # e.g. @{ title="Hello"; url="/hello" }
        [string]$Locale = $CS_LOCALE
    )
    $body = @{ entry = $Fields }
    Invoke-CMA -Method POST -Path "/v3/content_types/$ContentTypeUID/entries" `
        -Body $body -Query @{ locale=$Locale }
}
```

**Example:**
```powershell
New-CSEntry -ContentTypeUID "blog_post" -Fields @{
    title = "My First Post"
    url   = "/my-first-post"
    body  = "Hello world!"
}
```

### Update an entry
```powershell
function Update-CSEntry {
    param(
        [string]$ContentTypeUID,
        [string]$EntryUID,
        [hashtable]$Fields,
        [string]$Locale = $CS_LOCALE
    )
    $body = @{ entry = $Fields }
    Invoke-CMA -Method PUT -Path "/v3/content_types/$ContentTypeUID/entries/$EntryUID" `
        -Body $body -Query @{ locale=$Locale }
}
```

### Delete an entry
```powershell
function Remove-CSEntry {
    param(
        [string]$ContentTypeUID,
        [string]$EntryUID,
        [string]$Locale = $CS_LOCALE
    )
    Invoke-CMA -Method DELETE -Path "/v3/content_types/$ContentTypeUID/entries/$EntryUID" `
        -Query @{ locale=$Locale }
}
```

### Publish an entry
```powershell
function Publish-CSEntry {
    param(
        [string]$ContentTypeUID,
        [string]$EntryUID,
        [string[]]$Environments = @($CS_ENV),
        [string[]]$Locales      = @($CS_LOCALE),
        [string]$Locale         = $CS_LOCALE,
        [int]$Version           = 1,
        [string]$ScheduledAt    = ""   # ISO 8601 e.g. "2025-12-01T10:00:00.000Z"
    )
    $entry = @{ environments=$Environments; locales=$Locales }
    $body  = @{ entry=$entry; locale=$Locale; version=$Version }
    if ($ScheduledAt) { $body["scheduled_at"] = $ScheduledAt }
    Invoke-CMA -Method POST `
        -Path "/v3/content_types/$ContentTypeUID/entries/$EntryUID/publish" -Body $body
}
```

### Unpublish an entry
```powershell
function Unpublish-CSEntry {
    param(
        [string]$ContentTypeUID,
        [string]$EntryUID,
        [string[]]$Environments = @($CS_ENV),
        [string[]]$Locales      = @($CS_LOCALE),
        [string]$Locale         = $CS_LOCALE
    )
    $body = @{
        entry  = @{ environments=$Environments; locales=$Locales }
        locale = $Locale
    }
    Invoke-CMA -Method POST `
        -Path "/v3/content_types/$ContentTypeUID/entries/$EntryUID/unpublish" -Body $body
}
```

### Localize an entry
```powershell
function Set-CSEntryLocale {
    param(
        [string]$ContentTypeUID,
        [string]$EntryUID,
        [string]$Locale,
        [hashtable]$Fields
    )
    $body = @{ entry = $Fields }
    Invoke-CMA -Method PUT -Path "/v3/content_types/$ContentTypeUID/entries/$EntryUID" `
        -Body $body -Query @{ locale=$Locale }
}
```

### Unlocalize an entry
```powershell
function Remove-CSEntryLocale {
    param(
        [string]$ContentTypeUID,
        [string]$EntryUID,
        [string]$Locale
    )
    Invoke-CMA -Method POST `
        -Path "/v3/content_types/$ContentTypeUID/entries/$EntryUID/unlocalize" `
        -Query @{ locale=$Locale }
}
```

### Get entry versions
```powershell
function Get-CSEntryVersions {
    param([string]$ContentTypeUID, [string]$EntryUID)
    Invoke-CMA -Method GET -Path "/v3/content_types/$ContentTypeUID/entries/$EntryUID/versions"
}
```

### Get entry references
```powershell
function Get-CSEntryReferences {
    param([string]$ContentTypeUID, [string]$EntryUID)
    Invoke-CMA -Method GET -Path "/v3/content_types/$ContentTypeUID/entries/$EntryUID/references"
}
```

### Get entry locales
```powershell
function Get-CSEntryLocales {
    param([string]$ContentTypeUID, [string]$EntryUID)
    Invoke-CMA -Method GET -Path "/v3/content_types/$ContentTypeUID/entries/$EntryUID/locales"
}
```

### Set entry workflow stage
```powershell
function Set-CSEntryWorkflowStage {
    param(
        [string]$ContentTypeUID,
        [string]$EntryUID,
        [string]$WorkflowStageUID,
        [string]$Comment = "",
        [string]$DueDate = ""
    )
    $wf = @{ workflow_stage = @{ uid=$WorkflowStageUID; comment=$Comment } }
    if ($DueDate) { $wf.workflow_stage["due_date"] = $DueDate }
    $body = @{ workflow = $wf }
    Invoke-CMA -Method POST `
        -Path "/v3/content_types/$ContentTypeUID/entries/$EntryUID/workflow" -Body $body
}
```

---

## ASSETS

### List all assets
```powershell
function Get-CSAssets {
    param(
        [int]$Skip   = 0,
        [int]$Limit  = 100,
        [string]$FolderUID = "",
        [switch]$IncludeFolders
    )
    $q = @{ skip="$Skip"; limit="$Limit" }
    if ($FolderUID)      { $q["folder"] = $FolderUID }
    if ($IncludeFolders) { $q["include_folders"] = "true" }
    Invoke-CMA -Method GET -Path "/v3/assets" -Query $q
}
```

### Get a single asset
```powershell
function Get-CSAsset {
    param([string]$AssetUID)
    Invoke-CMA -Method GET -Path "/v3/assets/$AssetUID"
}
```

### Upload a new asset
```powershell
function Upload-CSAsset {
    param(
        [string]$FilePath,
        [string]$Description = "",
        [string]$FolderUID   = ""
    )
    $uri = "$CS_CMA_URL/v3/assets"
    $headers = @{
        "api_key"       = $CS_API_KEY
        "authorization" = $CS_MANAGEMENT_TOKEN
    }

    # Build multipart body manually using .NET
    $boundary = [System.Guid]::NewGuid().ToString()
    $fileName  = [System.IO.Path]::GetFileName($FilePath)
    $fileBytes = [System.IO.File]::ReadAllBytes($FilePath)
    $enc       = [System.Text.Encoding]::UTF8

    $bodyParts = [System.Collections.Generic.List[byte]]::new()

    # File part
    $filePart = $enc.GetBytes("--$boundary`r`nContent-Disposition: form-data; name=`"asset[upload]`"; filename=`"$fileName`"`r`nContent-Type: application/octet-stream`r`n`r`n")
    $bodyParts.AddRange($filePart)
    $bodyParts.AddRange($fileBytes)
    $bodyParts.AddRange($enc.GetBytes("`r`n"))

    if ($Description) {
        $descPart = $enc.GetBytes("--$boundary`r`nContent-Disposition: form-data; name=`"asset[description]`"`r`n`r`n$Description`r`n")
        $bodyParts.AddRange($descPart)
    }
    if ($FolderUID) {
        $folderPart = $enc.GetBytes("--$boundary`r`nContent-Disposition: form-data; name=`"asset[parent_uid]`"`r`n`r`n$FolderUID`r`n")
        $bodyParts.AddRange($folderPart)
    }

    $bodyParts.AddRange($enc.GetBytes("--$boundary--`r`n"))

    $response = Invoke-WebRequest -UseBasicParsing -Uri $uri -Method POST -Headers $headers `
        -ContentType "multipart/form-data; boundary=$boundary" -Body $bodyParts.ToArray()
    $response.Content | ConvertFrom-Json
}
```

### Update asset metadata
```powershell
function Update-CSAsset {
    param(
        [string]$AssetUID,
        [string]$Title       = "",
        [string]$Description = "",
        [hashtable]$Tags     = @{}
    )
    $asset = @{}
    if ($Title)       { $asset["title"]       = $Title }
    if ($Description) { $asset["description"] = $Description }
    $body = @{ asset = $asset }
    Invoke-CMA -Method PUT -Path "/v3/assets/$AssetUID" -Body $body
}
```

### Delete an asset
```powershell
function Remove-CSAsset {
    param([string]$AssetUID)
    Invoke-CMA -Method DELETE -Path "/v3/assets/$AssetUID"
}
```

### Publish an asset
```powershell
function Publish-CSAsset {
    param(
        [string]$AssetUID,
        [string[]]$Environments = @($CS_ENV),
        [string[]]$Locales      = @($CS_LOCALE)
    )
    $body = @{ asset = @{ environments=$Environments; locales=$Locales } }
    Invoke-CMA -Method POST -Path "/v3/assets/$AssetUID/publish" -Body $body
}
```

### Unpublish an asset
```powershell
function Unpublish-CSAsset {
    param(
        [string]$AssetUID,
        [string[]]$Environments = @($CS_ENV),
        [string[]]$Locales      = @($CS_LOCALE)
    )
    $body = @{ asset = @{ environments=$Environments; locales=$Locales } }
    Invoke-CMA -Method POST -Path "/v3/assets/$AssetUID/unpublish" -Body $body
}
```

### Create an asset folder
```powershell
function New-CSAssetFolder {
    param(
        [string]$Name,
        [string]$ParentUID = ""
    )
    $folder = @{ name=$Name }
    if ($ParentUID) { $folder["parent_uid"] = $ParentUID }
    $body = @{ asset = $folder }
    Invoke-CMA -Method POST -Path "/v3/assets/folders" -Body $body
}
```

### Delete an asset folder
```powershell
function Remove-CSAssetFolder {
    param([string]$FolderUID)
    Invoke-CMA -Method DELETE -Path "/v3/assets/folders/$FolderUID"
}
```

### Get asset versions
```powershell
function Get-CSAssetVersions {
    param([string]$AssetUID)
    Invoke-CMA -Method GET -Path "/v3/assets/$AssetUID/versions"
}
```

---

## BULK OPERATIONS

### Bulk publish entries and assets
```powershell
function Publish-CSBulk {
    param(
        [array]$Entries,       # @(@{ uid="uid1"; content_type="ct1"; version=1; locale="en-us" })
        [array]$Assets,        # @(@{ uid="asset_uid1"; version=1 })
        [string[]]$Environments = @($CS_ENV),
        [string[]]$Locales      = @($CS_LOCALE)
    )
    $details = @{ environments=$Environments; locales=$Locales }
    $body = @{ details=$details }
    if ($Entries) { $body["entries"] = $Entries }
    if ($Assets)  { $body["assets"]  = $Assets }
    Invoke-CMA -Method POST -Path "/v3/bulk/publish" -Body $body
}
```

### Bulk unpublish entries and assets
```powershell
function Unpublish-CSBulk {
    param(
        [array]$Entries,
        [array]$Assets,
        [string[]]$Environments = @($CS_ENV),
        [string[]]$Locales      = @($CS_LOCALE)
    )
    $details = @{ environments=$Environments; locales=$Locales }
    $body = @{ details=$details }
    if ($Entries) { $body["entries"] = $Entries }
    if ($Assets)  { $body["assets"]  = $Assets }
    Invoke-CMA -Method POST -Path "/v3/bulk/unpublish" -Body $body
}
```

### Bulk delete entries and assets
```powershell
function Remove-CSBulk {
    param(
        [array]$Entries,
        [array]$Assets
    )
    $body = @{}
    if ($Entries) { $body["entries"] = $Entries }
    if ($Assets)  { $body["assets"]  = $Assets }
    Invoke-CMA -Method POST -Path "/v3/bulk/delete" -Body $body
}
```

### Get bulk job status
```powershell
function Get-CSBulkJobStatus {
    param([string]$JobID)
    Invoke-CMA -Method GET -Path "/v3/bulk/jobs/$JobID"
}
```

---

## TAXONOMY

### List all taxonomies
```powershell
function Get-CSTaxonomies {
    Invoke-CMA -Method GET -Path "/v3/taxonomies"
}
```

### Get a single taxonomy
```powershell
function Get-CSTaxonomy {
    param([string]$TaxonomyUID)
    Invoke-CMA -Method GET -Path "/v3/taxonomies/$TaxonomyUID"
}
```

### Create a taxonomy
```powershell
function New-CSTaxonomy {
    param(
        [string]$UID,
        [string]$Name,
        [string]$Description = ""
    )
    $body = @{ taxonomy = @{ uid=$UID; name=$Name; description=$Description } }
    Invoke-CMA -Method POST -Path "/v3/taxonomies/" -Body $body
}
```

### Update a taxonomy
```powershell
function Update-CSTaxonomy {
    param(
        [string]$TaxonomyUID,
        [string]$Name,
        [string]$Description = ""
    )
    $body = @{ taxonomy = @{ name=$Name; description=$Description } }
    Invoke-CMA -Method PUT -Path "/v3/taxonomies/$TaxonomyUID" -Body $body
}
```

### Delete a taxonomy
```powershell
function Remove-CSTaxonomy {
    param([string]$TaxonomyUID, [switch]$Force)
    $q = @{ force = if ($Force) { "true" } else { "false" } }
    Invoke-CMA -Method DELETE -Path "/v3/taxonomies/$TaxonomyUID" -Query $q
}
```

### List all terms in a taxonomy
```powershell
function Get-CSTerms {
    param([string]$TaxonomyUID, [int]$Depth = 0)
    $q = @{}
    if ($Depth -gt 0) { $q["depth"] = "$Depth" }
    Invoke-CMA -Method GET -Path "/v3/taxonomies/$TaxonomyUID/terms" -Query $q
}
```

### Get a single term
```powershell
function Get-CSTerm {
    param([string]$TaxonomyUID, [string]$TermUID)
    Invoke-CMA -Method GET -Path "/v3/taxonomies/$TaxonomyUID/terms/$TermUID"
}
```

### Create a term
```powershell
function New-CSTerm {
    param(
        [string]$TaxonomyUID,
        [string]$UID,
        [string]$Name,
        [string]$ParentUID = ""
    )
    $term = @{ uid=$UID; name=$Name }
    if ($ParentUID) { $term["parent_uid"] = $ParentUID }
    $body = @{ term = $term }
    Invoke-CMA -Method POST -Path "/v3/taxonomies/$TaxonomyUID/terms" -Body $body
}
```

### Update a term
```powershell
function Update-CSTerm {
    param(
        [string]$TaxonomyUID,
        [string]$TermUID,
        [string]$Name
    )
    $body = @{ term = @{ name=$Name } }
    Invoke-CMA -Method PUT -Path "/v3/taxonomies/$TaxonomyUID/terms/$TermUID" -Body $body
}
```

### Delete a term
```powershell
function Remove-CSTerm {
    param([string]$TaxonomyUID, [string]$TermUID, [switch]$Force)
    $q = @{ force = if ($Force) { "true" } else { "false" } }
    Invoke-CMA -Method DELETE -Path "/v3/taxonomies/$TaxonomyUID/terms/$TermUID" -Query $q
}
```

### Get term descendants
```powershell
function Get-CSTermDescendants {
    param([string]$TaxonomyUID, [string]$TermUID)
    Invoke-CMA -Method GET -Path "/v3/taxonomies/$TaxonomyUID/terms/$TermUID/descendants"
}
```

### Get term ancestors
```powershell
function Get-CSTermAncestors {
    param([string]$TaxonomyUID, [string]$TermUID)
    Invoke-CMA -Method GET -Path "/v3/taxonomies/$TaxonomyUID/terms/$TermUID/ancestors"
}
```

---

## ENVIRONMENTS

### List all environments
```powershell
function Get-CSEnvironments {
    Invoke-CMA -Method GET -Path "/v3/environments"
}
```

### Get a single environment
```powershell
function Get-CSEnvironment {
    param([string]$EnvironmentName)
    Invoke-CMA -Method GET -Path "/v3/environments/$EnvironmentName"
}
```

### Create an environment
```powershell
function New-CSEnvironment {
    param(
        [string]$Name,
        [array]$Urls = @()   # @(@{ locale="en-us"; url="https://mysite.com" })
    )
    $body = @{ environment = @{ name=$Name; urls=$Urls } }
    Invoke-CMA -Method POST -Path "/v3/environments" -Body $body
}
```

### Delete an environment
```powershell
function Remove-CSEnvironment {
    param([string]$EnvironmentName)
    Invoke-CMA -Method DELETE -Path "/v3/environments/$EnvironmentName"
}
```

---

## LANGUAGES (LOCALES)

### List all languages
```powershell
function Get-CSLanguages {
    Invoke-CMA -Method GET -Path "/v3/locales"
}
```

### Add a language
```powershell
function Add-CSLanguage {
    param(
        [string]$Code,          # e.g. "fr-fr"
        [string]$Name,          # e.g. "French - France"
        [string]$FallbackLocale = "en-us"
    )
    $body = @{ locale = @{ code=$Code; name=$Name; fallback_locale=$FallbackLocale } }
    Invoke-CMA -Method POST -Path "/v3/locales" -Body $body
}
```

### Delete a language
```powershell
function Remove-CSLanguage {
    param([string]$LocaleCode)
    Invoke-CMA -Method DELETE -Path "/v3/locales/$LocaleCode"
}
```

---

## RELEASES

### List all releases
```powershell
function Get-CSReleases {
    param([int]$Skip=0, [int]$Limit=100, [switch]$IncludeCount)
    $q = @{ skip="$Skip"; limit="$Limit" }
    if ($IncludeCount) { $q["include_count"] = "true" }
    Invoke-CMA -Method GET -Path "/v3/releases" -Query $q
}
```

### Get a single release
```powershell
function Get-CSRelease {
    param([string]$ReleaseUID)
    Invoke-CMA -Method GET -Path "/v3/releases/$ReleaseUID"
}
```

### Create a release
```powershell
function New-CSRelease {
    param(
        [string]$Name,
        [string]$Description = "",
        [boolean]$Locked     = $false
    )
    $body = @{ release = @{ name=$Name; description=$Description; locked=$Locked } }
    Invoke-CMA -Method POST -Path "/v3/releases" -Body $body
}
```

### Add item to a release
```powershell
function Add-CSReleaseItem {
    param(
        [string]$ReleaseUID,
        [string]$UID,
        [string]$ContentTypeUID = "",   # required for entries
        [string]$Type,                  # "entry" or "asset"
        [string]$Action = "publish",    # "publish" or "unpublish"
        [string]$Locale = $CS_LOCALE,
        [int]$Version   = 1
    )
    $item = @{ uid=$UID; type=$Type; action=$Action; locale=$Locale; version=$Version }
    if ($ContentTypeUID) { $item["content_type_uid"] = $ContentTypeUID }
    $body = @{ item = $item }
    Invoke-CMA -Method POST -Path "/v3/releases/$ReleaseUID/item" -Body $body
}
```

### Add multiple items to a release
```powershell
function Add-CSReleaseItems {
    param(
        [string]$ReleaseUID,
        [array]$Items   # @(@{uid="e1"; content_type_uid="blog"; type="entry"; action="publish"; locale="en-us"; version=1})
    )
    $body = @{ items = $Items }
    Invoke-CMA -Method POST -Path "/v3/releases/$ReleaseUID/items" -Body $body
}
```

### Deploy a release
```powershell
function Deploy-CSRelease {
    param(
        [string]$ReleaseUID,
        [string[]]$Environments = @($CS_ENV),
        [string[]]$Locales      = @($CS_LOCALE),
        [string]$Action         = "publish",     # "publish" or "unpublish"
        [string]$ScheduledAt    = ""
    )
    $body = @{ release = @{ environments=$Environments; locales=$Locales; action=$Action } }
    if ($ScheduledAt) { $body.release["scheduled_at"] = $ScheduledAt }
    Invoke-CMA -Method POST -Path "/v3/releases/$ReleaseUID/deploy" -Body $body
}
```

### Delete a release
```powershell
function Remove-CSRelease {
    param([string]$ReleaseUID)
    Invoke-CMA -Method DELETE -Path "/v3/releases/$ReleaseUID"
}
```

---

## WORKFLOWS

### List all workflows
```powershell
function Get-CSWorkflows {
    Invoke-CMA -Method GET -Path "/v3/workflows"
}
```

### Get a single workflow
```powershell
function Get-CSWorkflow {
    param([string]$WorkflowUID)
    Invoke-CMA -Method GET -Path "/v3/workflows/$WorkflowUID"
}
```

### Create a workflow
```powershell
function New-CSWorkflow {
    param(
        [string]$Name,
        [string]$Description     = "",
        [array]$ContentTypes     = @(),   # @("blog_post", "page")
        [array]$WorkflowStages   = @()    # array of stage hashtables
    )
    $body = @{
        workflow = @{
            name            = $Name
            description     = $Description
            content_types   = $ContentTypes
            workflow_stages = $WorkflowStages
        }
    }
    Invoke-CMA -Method POST -Path "/v3/workflows" -Body $body
}
```

### Enable/Disable workflow
```powershell
function Enable-CSWorkflow {
    param([string]$WorkflowUID)
    Invoke-CMA -Method GET -Path "/v3/workflows/$WorkflowUID/enable"
}

function Disable-CSWorkflow {
    param([string]$WorkflowUID)
    Invoke-CMA -Method GET -Path "/v3/workflows/$WorkflowUID/disable"
}
```

### Delete a workflow
```powershell
function Remove-CSWorkflow {
    param([string]$WorkflowUID)
    Invoke-CMA -Method DELETE -Path "/v3/workflows/$WorkflowUID"
}
```

---

## WEBHOOKS

### List all webhooks
```powershell
function Get-CSWebhooks {
    Invoke-CMA -Method GET -Path "/v3/webhooks"
}
```

### Get a single webhook
```powershell
function Get-CSWebhook {
    param([string]$WebhookUID)
    Invoke-CMA -Method GET -Path "/v3/webhooks/$WebhookUID"
}
```

### Create a webhook
```powershell
function New-CSWebhook {
    param(
        [string]$Name,
        [string]$TargetUrl,
        [array]$Channels    = @("publish"),   # events: publish, unpublish, create, update, delete
        [hashtable]$AuthHeader = @{},          # e.g. @{ header_name="Authorization"; value="Bearer token" }
        [boolean]$Enabled   = $true,
        [boolean]$ConcisePayload = $true
    )
    $wh = @{
        name             = $Name
        destinations     = @(@{ target_url=$TargetUrl; http_basic_auth=""; http_basic_password="" })
        channels         = $Channels
        enabled          = $Enabled
        concise_payload  = $ConcisePayload
    }
    if ($AuthHeader.Count -gt 0) { $wh["custom_headers"] = @($AuthHeader) }
    $body = @{ webhook = $wh }
    Invoke-CMA -Method POST -Path "/v3/webhooks" -Body $body
}
```

### Delete a webhook
```powershell
function Remove-CSWebhook {
    param([string]$WebhookUID)
    Invoke-CMA -Method DELETE -Path "/v3/webhooks/$WebhookUID"
}
```

### Get webhook executions
```powershell
function Get-CSWebhookExecutions {
    param([string]$WebhookUID)
    Invoke-CMA -Method GET -Path "/v3/webhooks/$WebhookUID/executions"
}
```

### Retry a webhook execution
```powershell
function Invoke-CSWebhookRetry {
    param([string]$ExecutionUID)
    Invoke-CMA -Method POST -Path "/v3/webhooks/$ExecutionUID/retry"
}
```

---

## TOKENS

### List all management tokens
```powershell
function Get-CSManagementTokens {
    Invoke-CMA -Method GET -Path "/v3/stacks/management_tokens"
}
```

### Create a management token
```powershell
function New-CSManagementToken {
    param(
        [string]$Name,
        [string]$Description = "",
        [string[]]$Scope     = @("read", "write"),
        [string]$ExpiresAt   = ""   # ISO date, leave blank for no expiry
    )
    $token = @{ name=$Name; description=$Description; scope=@(@{ module="environment"; acl=@{ read=$true; write=$true } }) }
    if ($ExpiresAt) { $token["expires_at"] = $ExpiresAt }
    $body = @{ token = $token }
    Invoke-CMA -Method POST -Path "/v3/stacks/management_tokens" -Body $body
}
```

### List all delivery tokens
```powershell
function Get-CSDeliveryTokens {
    Invoke-CMA -Method GET -Path "/v3/stacks/delivery_tokens"
}
```

### Create a delivery token
```powershell
function New-CSDeliveryToken {
    param(
        [string]$Name,
        [string]$Description = "",
        [string]$Scope       = $CS_ENV   # environment name
    )
    $body = @{ token = @{ name=$Name; description=$Description; scope=@(@{ environments=@($Scope); module="environment" }) } }
    Invoke-CMA -Method POST -Path "/v3/stacks/delivery_tokens" -Body $body
}
```

---

## ROLES

### List all roles
```powershell
function Get-CSRoles {
    Invoke-CMA -Method GET -Path "/v3/roles"
}
```

### Create a role
```powershell
function New-CSRole {
    param(
        [string]$Name,
        [string]$Description = "",
        [array]$Rules        = @()
    )
    $body = @{ role = @{ name=$Name; description=$Description; rules=$Rules } }
    Invoke-CMA -Method POST -Path "/v3/roles" -Body $body
}
```

### Delete a role
```powershell
function Remove-CSRole {
    param([string]$RoleUID)
    Invoke-CMA -Method DELETE -Path "/v3/roles/$RoleUID"
}
```

---

## LABELS

### List all labels
```powershell
function Get-CSLabels {
    Invoke-CMA -Method GET -Path "/v3/labels"
}
```

### Create a label
```powershell
function New-CSLabel {
    param(
        [string]$Name,
        [array]$Parent = @()   # UIDs of parent labels
    )
    $body = @{ label = @{ name=$Name; parent=$Parent } }
    Invoke-CMA -Method POST -Path "/v3/labels" -Body $body
}
```

### Delete a label
```powershell
function Remove-CSLabel {
    param([string]$LabelUID)
    Invoke-CMA -Method DELETE -Path "/v3/labels/$LabelUID"
}
```

---

## AUDIT LOG

### Get audit log
```powershell
function Get-CSAuditLog {
    param(
        [int]$Skip  = 0,
        [int]$Limit = 100
    )
    Invoke-CMA -Method GET -Path "/v3/audit-logs" -Query @{ skip="$Skip"; limit="$Limit" }
}
```

### Get a single audit log item
```powershell
function Get-CSAuditLogItem {
    param([string]$LogItemUID)
    Invoke-CMA -Method GET -Path "/v3/audit-logs/$LogItemUID"
}
```

---

## PUBLISH QUEUE

### Get publish queue
```powershell
function Get-CSPublishQueue {
    param([int]$Skip=0, [int]$Limit=100)
    Invoke-CMA -Method GET -Path "/v3/publish-queue" -Query @{ skip="$Skip"; limit="$Limit" }
}
```

### Cancel a scheduled publish action
```powershell
function Stop-CSScheduledPublish {
    param([string]$PublishQueueUID)
    Invoke-CMA -Method GET -Path "/v3/publish-queue/$PublishQueueUID/unschedule"
}
```

---

## CONTENT DELIVERY API (CDA) — Read Published Content

The CDA functions use the CDN endpoint and delivery token. They only return **published** content.

### Get all published content types (CDA)
```powershell
function Get-CSPublishedContentTypes {
    Invoke-CDA -Path "/v3/content_types"
}
```

### Get all published entries (CDA)
```powershell
function Get-CSPublishedEntries {
    param(
        [string]$ContentTypeUID,
        [string]$Locale = $CS_LOCALE,
        [int]$Skip      = 0,
        [int]$Limit     = 100,
        [string]$Query  = "",    # JSON query string
        [string]$Asc    = "",
        [string]$Desc   = ""
    )
    $q = @{ locale=$Locale; skip="$Skip"; limit="$Limit" }
    if ($Query) { $q["query"] = $Query }
    if ($Asc)   { $q["asc"]   = $Asc }
    if ($Desc)  { $q["desc"]  = $Desc }
    Invoke-CDA -Path "/v3/content_types/$ContentTypeUID/entries" -Query $q
}
```

### Get a single published entry (CDA)
```powershell
function Get-CSPublishedEntry {
    param(
        [string]$ContentTypeUID,
        [string]$EntryUID,
        [string]$Locale = $CS_LOCALE
    )
    Invoke-CDA -Path "/v3/content_types/$ContentTypeUID/entries/$EntryUID" `
        -Query @{ locale=$Locale }
}
```

### Get all published assets (CDA)
```powershell
function Get-CSPublishedAssets {
    param([int]$Skip=0, [int]$Limit=100)
    Invoke-CDA -Path "/v3/assets" -Query @{ skip="$Skip"; limit="$Limit" }
}
```

### Get a single published asset (CDA)
```powershell
function Get-CSPublishedAsset {
    param([string]$AssetUID)
    Invoke-CDA -Path "/v3/assets/$AssetUID"
}
```

### Query entries with operators (CDA)
```powershell
# Equals
Get-CSPublishedEntries -ContentTypeUID "blog_post" -Query '{"title":"My Post"}'

# Greater than
Get-CSPublishedEntries -ContentTypeUID "products" -Query '{"price":{"$gt":100}}'

# Array IN
Get-CSPublishedEntries -ContentTypeUID "products" -Query '{"price":{"$in":[99,149,199]}}'

# AND
Get-CSPublishedEntries -ContentTypeUID "blog_post" -Query '{"$and":[{"title":"Hello"},{"locale":"en-us"}]}'

# OR
Get-CSPublishedEntries -ContentTypeUID "blog_post" -Query '{"$or":[{"color":"Gold"},{"color":"Black"}]}'

# Regex
Get-CSPublishedEntries -ContentTypeUID "blog_post" -Query '{"title":{"$regex":"^Hello"}}'

# Reference search
Get-CSPublishedEntries -ContentTypeUID "products" -Query '{"brand":{"$in_query":{"title":"Apple"}}}'
```

### Synchronization (CDA)
```powershell
# Initial sync — fetches all published content
function Start-CSSync {
    Invoke-CDA -Path "/v3/stacks/sync" -Query @{ init="true"; type="entry_published" }
}

# Continue sync with pagination token
function Get-CSSyncPage {
    param([string]$PaginationToken)
    Invoke-CDA -Path "/v3/stacks/sync" -Query @{ pagination_token=$PaginationToken }
}

# Subsequent sync with sync token (only changes since last sync)
function Get-CSSyncDelta {
    param([string]$SyncToken)
    Invoke-CDA -Path "/v3/stacks/sync" -Query @{ sync_token=$SyncToken }
}
```

---

## COMMON WORKFLOWS

### Create, populate, and publish an entry end-to-end
```powershell
# 1. Create entry
$newEntry = New-CSEntry -ContentTypeUID "blog_post" -Fields @{
    title = "Getting Started with Contentstack"
    url   = "/getting-started"
    body  = "Welcome to Contentstack!"
}
$entryUID = $newEntry.entry.uid
Write-Host "Created entry: $entryUID"

# 2. Publish entry
Publish-CSEntry -ContentTypeUID "blog_post" -EntryUID $entryUID `
    -Environments @("development", "production") -Locales @("en-us") -Version 1
Write-Host "Published entry: $entryUID"
```

### Bulk-publish all entries of a content type
```powershell
$result  = Get-CSEntries -ContentTypeUID "blog_post" -Limit 100
$entries = $result.entries | ForEach-Object {
    @{ uid=$_.uid; content_type="blog_post"; version=1; locale=$CS_LOCALE }
}
Publish-CSBulk -Entries $entries -Environments @($CS_ENV) -Locales @($CS_LOCALE)
```

### Find and update entries matching a query
```powershell
$result = Get-CSEntries -ContentTypeUID "blog_post" `
    -Query '{"published_at":{"$exists":false}}' -Limit 100
foreach ($entry in $result.entries) {
    Update-CSEntry -ContentTypeUID "blog_post" -EntryUID $entry.uid `
        -Fields @{ status = "draft" }
}
```

### Deploy a set of content changes via a Release
```powershell
# 1. Create a release
$release = New-CSRelease -Name "Sprint-42 Release" -Description "Blog updates"
$releaseUID = $release.release.uid

# 2. Add entries to the release
$items = @(
    @{ uid="entry_uid_1"; content_type_uid="blog_post"; type="entry"; action="publish"; locale="en-us"; version=2 },
    @{ uid="entry_uid_2"; content_type_uid="blog_post"; type="entry"; action="publish"; locale="en-us"; version=1 }
)
Add-CSReleaseItems -ReleaseUID $releaseUID -Items $items

# 3. Deploy the release
Deploy-CSRelease -ReleaseUID $releaseUID -Environments @("production") -Locales @("en-us")
```

### Upload and publish an asset
```powershell
$asset    = Upload-CSAsset -FilePath "C:\Images\banner.jpg" -Description "Home page banner"
$assetUID = $asset.asset.uid
Publish-CSAsset -AssetUID $assetUID -Environments @("production") -Locales @("en-us")
```

### Sync published content and cache locally
```powershell
$sync = Start-CSSync
$allItems = [System.Collections.Generic.List[object]]::new()
$allItems.AddRange($sync.items)

while ($sync.pagination_token) {
    $sync = Get-CSSyncPage -PaginationToken $sync.pagination_token
    $allItems.AddRange($sync.items)
}

# Save to disk
$allItems | ConvertTo-Json -Depth 20 | Out-File ".\contentstack-cache.json" -Encoding UTF8
Write-Host "Synced $($allItems.Count) items. Sync token: $($sync.sync_token)"
```

---

## FURTHER REFERENCE — POSTMAN COLLECTIONS

If you need details beyond what this skill covers — exact request body shapes, additional query parameters, edge-case endpoints, or example payloads — refer directly to the Postman collection JSON files included alongside this skill:

| File | Covers |
|---|---|
| `Content Management API - Contentstack.postman_collection.json` | All write operations: create, update, delete, publish, workflows, releases, webhooks, tokens, roles, etc. |
| `Content Delivery API - Contentstack.postman_collection.json` | All read operations against published content: entries, assets, taxonomies, sync, query operators |

**How to use them:**

1. Open the JSON file in any text editor and search for the endpoint name (e.g. `"Atomic updates"`, `"Set Field Visibility Rule"`) to find the exact URL, HTTP method, headers, and sample request body.
2. Import into [Postman](https://www.postman.com) to run requests interactively — set the collection variables (`base_url`, `api_key`, `authtoken`, `management_token`) in the collection's Variables tab.
3. Use the sample bodies in the collection as the `$Body` hashtable when calling `Invoke-CMA` or `Invoke-CDA` directly for any endpoint not wrapped by a named function in this skill.

**Quick lookup example — finding an endpoint in the JSON:**
```powershell
# Search the CMA collection for any endpoint containing "atomic"
$cma = Get-Content "Content Management API - Contentstack.postman_collection.json" | ConvertFrom-Json

function Find-CSEndpoint {
    param([object[]]$Items, [string]$Search)
    foreach ($item in $Items) {
        if ($item.item) { Find-CSEndpoint -Items $item.item -Search $Search }
        elseif ($item.name -match $Search) {
            $req = $item.request
            $url = if ($req.url -is [string]) { $req.url } else { $req.url.raw }
            [PSCustomObject]@{
                Name   = $item.name
                Method = $req.method
                URL    = $url
                Body   = $req.body.raw
            }
        }
    }
}

Find-CSEndpoint -Items $cma.item -Search "atomic" | Format-List
```

---

## TIPS & TROUBLESHOOTING

| Issue | Fix |
|---|---|
| `Invoke-WebRequest` prompts for security confirmation | Always include `-UseBasicParsing` |
| 401 Unauthorized | Check `$CS_API_KEY`, `$CS_MANAGEMENT_TOKEN`, and `$CS_AUTH_TOKEN` are set correctly |
| 403 Forbidden | Management token may lack write scope; generate one with read+write scope |
| 422 Unprocessable Entity | Inspect the error body — usually a schema validation failure |
| Rate limit (429) | Add `Start-Sleep -Milliseconds 500` between bulk calls |
| Large list results | Use `skip`/`limit` pagination; max `limit` is 100 per request |
| Response is empty | Call `$response.Content` before `ConvertFrom-Json` if using raw `Invoke-WebRequest` |
| JSON serialization of `$true`/`$false` | PowerShell booleans serialize correctly via `ConvertTo-Json`; avoid string `"true"` |
| Multipart upload fails | Ensure the file path uses backslashes and the file exists: `Test-Path $FilePath` |
