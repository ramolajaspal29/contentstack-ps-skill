---
name: contentstack-bash
description: Manage Contentstack via bash on Linux/macOS using curl — create, update, delete, publish, and query content types, entries, assets, taxonomies, releases, workflows, webhooks, environments, and more using curl against both the Content Management API (CMA) and Content Delivery API (CDA). Supporting reference files (Postman collections) live in ~/.claude/skills/contentstack/.
---

# Contentstack Bash/curl Skill

## Overview

This skill enables full management of Contentstack stacks via **bash on Linux/macOS** using `curl`. It covers both the **Content Delivery API (CDA)** for reading published content and the **Content Management API (CMA)** for creating, updating, and publishing content.

**Requirements:** `curl`, `python3` (for JSON building), `jq` (optional, for pretty-printing responses — falls back to `python3 -m json.tool`).

---

## Configuration & Authentication

Source these variables in your shell session before using any function:

```bash
# === REQUIRED: Set these before using any function ===

# Stack API Key (from Contentstack > Stack Settings > API Credentials)
export CS_API_KEY="your_stack_api_key"

# Management Token (CMA write operations — from Stack Settings > Tokens > Management Tokens)
export CS_MANAGEMENT_TOKEN="your_management_token"

# Auth Token (user session token — from Login API or Contentstack dashboard)
export CS_AUTH_TOKEN="your_authtoken"

# Delivery Token (CDA read operations — from Stack Settings > Tokens > Delivery Tokens)
export CS_DELIVERY_TOKEN="your_delivery_token"

# Region base URLs — pick ONE matching your stack's region:
export CS_CMA_URL="https://api.contentstack.io"       # North America (default)
# export CS_CMA_URL="https://eu-api.contentstack.com"   # Europe
# export CS_CMA_URL="https://azure-na-api.contentstack.com"  # Azure NA

export CS_CDA_URL="https://cdn.contentstack.io"        # North America CDN
# export CS_CDA_URL="https://eu-cdn.contentstack.com"    # Europe CDN

# Default locale and environment
export CS_LOCALE="en-us"
export CS_ENV="production"
```

---

## Core Helper Functions

Paste these helper functions into your shell session once (or add to `~/.bashrc`). All resource-specific functions call these.

```bash
# Pretty-print JSON — uses jq if available, else python3
_cs_pretty() {
    if command -v jq &>/dev/null; then
        jq .
    else
        python3 -m json.tool
    fi
}

# Content Management API request
# Usage: cs_cma <METHOD> <PATH> [JSON_BODY] [QUERY_STRING] [use_auth]
cs_cma() {
    local method="${1:-GET}"
    local path="$2"
    local body="$3"
    local qs="$4"
    local use_auth="${5:-}"

    local url="${CS_CMA_URL}${path}"
    [[ -n "$qs" ]] && url="${url}?${qs}"

    local auth_header
    if [[ -n "$use_auth" ]]; then
        auth_header="authtoken: ${CS_AUTH_TOKEN}"
    else
        auth_header="authorization: ${CS_MANAGEMENT_TOKEN}"
    fi

    local curl_args=(-s -X "$method" "$url"
        -H "api_key: ${CS_API_KEY}"
        -H "${auth_header}"
        -H "Content-Type: application/json")

    if [[ -n "$body" ]]; then
        curl_args+=(-d "$body")
    fi

    curl "${curl_args[@]}"
}

# Content Delivery API request (read-only, published content)
# Usage: cs_cda <PATH> [QUERY_STRING]
cs_cda() {
    local path="$1"
    local qs="$2"

    local full_qs="environment=${CS_ENV}"
    [[ -n "$qs" ]] && full_qs="${full_qs}&${qs}"

    curl -s -X GET "${CS_CDA_URL}${path}?${full_qs}" \
        -H "api_key: ${CS_API_KEY}" \
        -H "access_token: ${CS_DELIVERY_TOKEN}"
}

# URL-encode a string
_cs_urlencode() {
    python3 -c "import urllib.parse, sys; print(urllib.parse.quote(sys.argv[1], safe=''))" "$1"
}
```

---

## STACKS

### Get stack details
```bash
cs_get_stack() {
    cs_cma GET /v3/stacks | _cs_pretty
}
```

### Update stack
```bash
cs_update_stack() {
    local name="$1"
    local description="${2:-}"
    local body
    body=$(python3 -c "
import json, sys
print(json.dumps({'stack': {'name': sys.argv[1], 'description': sys.argv[2]}}))
" "$name" "$description")
    cs_cma PUT /v3/stacks "$body" | _cs_pretty
}
```

### Get all stack users
```bash
cs_get_stack_users() {
    cs_cma GET /v3/stacks/users | _cs_pretty
}
```

### Share stack with a user
```bash
cs_share_stack() {
    local email="$1"
    local role_uid="$2"
    local body
    body=$(python3 -c "
import json, sys
email, role = sys.argv[1], sys.argv[2]
print(json.dumps({'emails': [email], 'roles': {email: [role]}}))
" "$email" "$role_uid")
    cs_cma POST /v3/stacks/share "$body" | _cs_pretty
}
```

### Unshare stack from a user
```bash
cs_unshare_stack() {
    local email="$1"
    local body
    body=$(python3 -c "import json, sys; print(json.dumps({'email': sys.argv[1]}))" "$email")
    cs_cma POST /v3/stacks/unshare "$body" | _cs_pretty
}
```

### Get stack settings
```bash
cs_get_stack_settings() {
    cs_cma GET /v3/stacks/settings | _cs_pretty
}
```

---

## BRANCHES

### List all branches
```bash
cs_get_branches() {
    cs_cma GET /v3/stacks/branches | _cs_pretty
}
```

### Get a single branch
```bash
cs_get_branch() {
    local branch_uid="$1"
    cs_cma GET "/v3/stacks/branches/${branch_uid}" | _cs_pretty
}
```

### Create a branch
```bash
cs_new_branch() {
    local uid="$1"
    local source="${2:-main}"
    local body
    body=$(python3 -c "
import json, sys
print(json.dumps({'branch': {'uid': sys.argv[1], 'source': sys.argv[2]}}))
" "$uid" "$source")
    cs_cma POST /v3/stacks/branches "$body" | _cs_pretty
}
```

### Delete a branch
```bash
cs_delete_branch() {
    local branch_uid="$1"
    cs_cma DELETE "/v3/stacks/branches/${branch_uid}" "" "force=true" | _cs_pretty
}
```

### Compare two branches
```bash
cs_compare_branches() {
    local compare_branch="$1"
    local qs="compare_branch=$(_cs_urlencode "$compare_branch")"
    cs_cma GET /v3/stacks/branches_compare "" "$qs" | _cs_pretty
}
```

### Merge branches
```bash
cs_merge_branches() {
    local compare_branch="$1"
    local merge_strategy="${2:-merge_prefer_base}"
    local body='{"item_merge_strategies":[]}'
    local qs="compare_branch=$(_cs_urlencode "$compare_branch")&default_merge_strategy=$(_cs_urlencode "$merge_strategy")"
    cs_cma POST /v3/stacks/branches_merge "$body" "$qs" | _cs_pretty
}
```

---

## CONTENT TYPES

### List all content types
```bash
cs_get_content_types() {
    local skip="${1:-0}"
    local limit="${2:-100}"
    cs_cma GET /v3/content_types "" "skip=${skip}&limit=${limit}" | _cs_pretty
}
```

### Get a single content type
```bash
cs_get_content_type() {
    local ct_uid="$1"
    cs_cma GET "/v3/content_types/${ct_uid}" | _cs_pretty
}
```

### Create a content type
```bash
# cs_new_content_type <title> <uid> <schema_json> [options_json]
# schema_json: JSON array of field definitions
# Example: cs_new_content_type "Blog Post" "blog_post" '[{"display_name":"Title","uid":"title","data_type":"text","mandatory":true}]'
cs_new_content_type() {
    local title="$1"
    local uid="$2"
    local schema_json="$3"
    local options_json="${4:-{}}"
    local body
    body=$(python3 -c "
import json, sys
print(json.dumps({
    'content_type': {
        'title':   sys.argv[1],
        'uid':     sys.argv[2],
        'schema':  json.loads(sys.argv[3]),
        'options': json.loads(sys.argv[4])
    }
}))
" "$title" "$uid" "$schema_json" "$options_json")
    cs_cma POST /v3/content_types "$body" | _cs_pretty
}
```

**Example — create a blog post content type:**
```bash
schema='[
  {"display_name":"Title","uid":"title","data_type":"text","field_metadata":{"_default":true},"mandatory":true,"unique":false,"multiple":false},
  {"display_name":"URL","uid":"url","data_type":"text","field_metadata":{"_default":true},"unique":false,"multiple":false}
]'
options='{"title":"title","publishable":true,"is_page":true,"singleton":false,"sub_title":["url"]}'
cs_new_content_type "Blog Post" "blog_post" "$schema" "$options"
```

### Update a content type
```bash
cs_update_content_type() {
    local ct_uid="$1"
    local title="$2"
    local schema_json="$3"
    local options_json="${4:-{}}"
    local body
    body=$(python3 -c "
import json, sys
print(json.dumps({
    'content_type': {
        'title':   sys.argv[2],
        'uid':     sys.argv[1],
        'schema':  json.loads(sys.argv[3]),
        'options': json.loads(sys.argv[4])
    }
}))
" "$ct_uid" "$title" "$schema_json" "$options_json")
    cs_cma PUT "/v3/content_types/${ct_uid}" "$body" | _cs_pretty
}
```

### Delete a content type
```bash
cs_delete_content_type() {
    local ct_uid="$1"
    cs_cma DELETE "/v3/content_types/${ct_uid}" | _cs_pretty
}
```

### Get content type references
```bash
cs_get_content_type_refs() {
    local ct_uid="$1"
    cs_cma GET "/v3/content_types/${ct_uid}/references" | _cs_pretty
}
```

### Export a content type
```bash
cs_export_content_type() {
    local ct_uid="$1"
    cs_cma GET "/v3/content_types/${ct_uid}/export" | _cs_pretty
}
```

### Import a content type
```bash
cs_import_content_type() {
    local json_file="$1"
    curl -s -X POST "${CS_CMA_URL}/v3/content_types/import" \
        -H "api_key: ${CS_API_KEY}" \
        -H "authorization: ${CS_MANAGEMENT_TOKEN}" \
        -H "Content-Type: application/json" \
        -d "@${json_file}" | _cs_pretty
}
```

---

## GLOBAL FIELDS

### List all global fields
```bash
cs_get_global_fields() {
    cs_cma GET /v3/global_fields | _cs_pretty
}
```

### Get a single global field
```bash
cs_get_global_field() {
    local gf_uid="$1"
    cs_cma GET "/v3/global_fields/${gf_uid}" | _cs_pretty
}
```

### Create a global field
```bash
# cs_new_global_field <title> <uid> <schema_json>
cs_new_global_field() {
    local title="$1"
    local uid="$2"
    local schema_json="$3"
    local body
    body=$(python3 -c "
import json, sys
print(json.dumps({'global_field': {'title': sys.argv[1], 'uid': sys.argv[2], 'schema': json.loads(sys.argv[3])}}))
" "$title" "$uid" "$schema_json")
    cs_cma POST /v3/global_fields "$body" | _cs_pretty
}
```

### Update a global field
```bash
cs_update_global_field() {
    local gf_uid="$1"
    local title="$2"
    local schema_json="$3"
    local body
    body=$(python3 -c "
import json, sys
print(json.dumps({'global_field': {'title': sys.argv[2], 'uid': sys.argv[1], 'schema': json.loads(sys.argv[3])}}))
" "$gf_uid" "$title" "$schema_json")
    cs_cma PUT "/v3/global_fields/${gf_uid}" "$body" | _cs_pretty
}
```

### Delete a global field
```bash
cs_delete_global_field() {
    local gf_uid="$1"
    cs_cma DELETE "/v3/global_fields/${gf_uid}" "" "force=true" | _cs_pretty
}
```

---

## ENTRIES

### List all entries in a content type
```bash
# cs_get_entries <content_type_uid> [locale] [skip] [limit] [query_json] [asc_field] [desc_field]
cs_get_entries() {
    local ct_uid="$1"
    local locale="${2:-$CS_LOCALE}"
    local skip="${3:-0}"
    local limit="${4:-100}"
    local query="${5:-}"
    local asc="${6:-}"
    local desc="${7:-}"

    local qs="locale=${locale}&skip=${skip}&limit=${limit}"
    [[ -n "$query" ]] && qs="${qs}&query=$(_cs_urlencode "$query")"
    [[ -n "$asc"   ]] && qs="${qs}&asc=${asc}"
    [[ -n "$desc"  ]] && qs="${qs}&desc=${desc}"

    cs_cma GET "/v3/content_types/${ct_uid}/entries" "" "$qs" | _cs_pretty
}
```

### Get a single entry
```bash
cs_get_entry() {
    local ct_uid="$1"
    local entry_uid="$2"
    local locale="${3:-$CS_LOCALE}"
    cs_cma GET "/v3/content_types/${ct_uid}/entries/${entry_uid}" "" "locale=${locale}" | _cs_pretty
}
```

### Create an entry
```bash
# cs_new_entry <content_type_uid> <fields_json> [locale]
# fields_json: JSON object e.g. '{"title":"Hello","url":"/hello"}'
cs_new_entry() {
    local ct_uid="$1"
    local fields_json="$2"
    local locale="${3:-$CS_LOCALE}"
    local body
    body=$(python3 -c "
import json, sys
print(json.dumps({'entry': json.loads(sys.argv[1])}))
" "$fields_json")
    cs_cma POST "/v3/content_types/${ct_uid}/entries" "$body" "locale=${locale}" | _cs_pretty
}
```

**Example:**
```bash
cs_new_entry "blog_post" '{"title":"My First Post","url":"/my-first-post","body":"Hello world!"}'
```

### Update an entry
```bash
# cs_update_entry <content_type_uid> <entry_uid> <fields_json> [locale]
cs_update_entry() {
    local ct_uid="$1"
    local entry_uid="$2"
    local fields_json="$3"
    local locale="${4:-$CS_LOCALE}"
    local body
    body=$(python3 -c "
import json, sys
print(json.dumps({'entry': json.loads(sys.argv[1])}))
" "$fields_json")
    cs_cma PUT "/v3/content_types/${ct_uid}/entries/${entry_uid}" "$body" "locale=${locale}" | _cs_pretty
}
```

### Delete an entry
```bash
cs_delete_entry() {
    local ct_uid="$1"
    local entry_uid="$2"
    local locale="${3:-$CS_LOCALE}"
    cs_cma DELETE "/v3/content_types/${ct_uid}/entries/${entry_uid}" "" "locale=${locale}" | _cs_pretty
}
```

### Publish an entry
```bash
# cs_publish_entry <content_type_uid> <entry_uid> [environments_csv] [locales_csv] [version]
# environments_csv: comma-separated e.g. "production,staging"
cs_publish_entry() {
    local ct_uid="$1"
    local entry_uid="$2"
    local envs_csv="${3:-$CS_ENV}"
    local locales_csv="${4:-$CS_LOCALE}"
    local version="${5:-1}"
    local body
    body=$(python3 -c "
import json, sys
envs    = sys.argv[3].split(',')
locales = sys.argv[4].split(',')
print(json.dumps({
    'entry':   {'environments': envs, 'locales': locales},
    'locale':  locales[0],
    'version': int(sys.argv[5])
}))
" "$ct_uid" "$entry_uid" "$envs_csv" "$locales_csv" "$version")
    cs_cma POST "/v3/content_types/${ct_uid}/entries/${entry_uid}/publish" "$body" | _cs_pretty
}
```

### Unpublish an entry
```bash
cs_unpublish_entry() {
    local ct_uid="$1"
    local entry_uid="$2"
    local envs_csv="${3:-$CS_ENV}"
    local locales_csv="${4:-$CS_LOCALE}"
    local body
    body=$(python3 -c "
import json, sys
envs    = sys.argv[3].split(',')
locales = sys.argv[4].split(',')
print(json.dumps({'entry': {'environments': envs, 'locales': locales}, 'locale': locales[0]}))
" "$ct_uid" "$entry_uid" "$envs_csv" "$locales_csv")
    cs_cma POST "/v3/content_types/${ct_uid}/entries/${entry_uid}/unpublish" "$body" | _cs_pretty
}
```

### Get entry versions
```bash
cs_get_entry_versions() {
    local ct_uid="$1"
    local entry_uid="$2"
    cs_cma GET "/v3/content_types/${ct_uid}/entries/${entry_uid}/versions" | _cs_pretty
}
```

### Get entry references
```bash
cs_get_entry_refs() {
    local ct_uid="$1"
    local entry_uid="$2"
    cs_cma GET "/v3/content_types/${ct_uid}/entries/${entry_uid}/references" | _cs_pretty
}
```

### Get entry locales
```bash
cs_get_entry_locales() {
    local ct_uid="$1"
    local entry_uid="$2"
    cs_cma GET "/v3/content_types/${ct_uid}/entries/${entry_uid}/locales" | _cs_pretty
}
```

### Set entry workflow stage
```bash
cs_set_entry_workflow() {
    local ct_uid="$1"
    local entry_uid="$2"
    local stage_uid="$3"
    local comment="${4:-}"
    local body
    body=$(python3 -c "
import json, sys
print(json.dumps({'workflow': {'workflow_stage': {'uid': sys.argv[3], 'comment': sys.argv[4]}}}))
" "$ct_uid" "$entry_uid" "$stage_uid" "$comment")
    cs_cma POST "/v3/content_types/${ct_uid}/entries/${entry_uid}/workflow" "$body" | _cs_pretty
}
```

---

## ASSETS

### List all assets
```bash
cs_get_assets() {
    local skip="${1:-0}"
    local limit="${2:-100}"
    local folder_uid="${3:-}"
    local qs="skip=${skip}&limit=${limit}"
    [[ -n "$folder_uid" ]] && qs="${qs}&folder=${folder_uid}"
    cs_cma GET /v3/assets "" "$qs" | _cs_pretty
}
```

### Get a single asset
```bash
cs_get_asset() {
    local asset_uid="$1"
    cs_cma GET "/v3/assets/${asset_uid}" | _cs_pretty
}
```

### Upload a new asset
```bash
# cs_upload_asset <file_path> [description] [folder_uid]
cs_upload_asset() {
    local file_path="$1"
    local description="${2:-}"
    local folder_uid="${3:-}"

    local curl_args=(-s -X POST "${CS_CMA_URL}/v3/assets"
        -H "api_key: ${CS_API_KEY}"
        -H "authorization: ${CS_MANAGEMENT_TOKEN}"
        -F "asset[upload]=@${file_path}")

    [[ -n "$description" ]] && curl_args+=(-F "asset[description]=${description}")
    [[ -n "$folder_uid"  ]] && curl_args+=(-F "asset[parent_uid]=${folder_uid}")

    curl "${curl_args[@]}" | _cs_pretty
}
```

### Update asset metadata
```bash
cs_update_asset() {
    local asset_uid="$1"
    local title="${2:-}"
    local description="${3:-}"
    local body
    body=$(python3 -c "
import json, sys
asset = {}
if sys.argv[2]: asset['title']       = sys.argv[2]
if sys.argv[3]: asset['description'] = sys.argv[3]
print(json.dumps({'asset': asset}))
" "$asset_uid" "$title" "$description")
    cs_cma PUT "/v3/assets/${asset_uid}" "$body" | _cs_pretty
}
```

### Delete an asset
```bash
cs_delete_asset() {
    local asset_uid="$1"
    cs_cma DELETE "/v3/assets/${asset_uid}" | _cs_pretty
}
```

### Publish an asset
```bash
cs_publish_asset() {
    local asset_uid="$1"
    local envs_csv="${2:-$CS_ENV}"
    local locales_csv="${3:-$CS_LOCALE}"
    local body
    body=$(python3 -c "
import json, sys
print(json.dumps({'asset': {'environments': sys.argv[2].split(','), 'locales': sys.argv[3].split(',')}}))
" "$asset_uid" "$envs_csv" "$locales_csv")
    cs_cma POST "/v3/assets/${asset_uid}/publish" "$body" | _cs_pretty
}
```

### Unpublish an asset
```bash
cs_unpublish_asset() {
    local asset_uid="$1"
    local envs_csv="${2:-$CS_ENV}"
    local locales_csv="${3:-$CS_LOCALE}"
    local body
    body=$(python3 -c "
import json, sys
print(json.dumps({'asset': {'environments': sys.argv[2].split(','), 'locales': sys.argv[3].split(',')}}))
" "$asset_uid" "$envs_csv" "$locales_csv")
    cs_cma POST "/v3/assets/${asset_uid}/unpublish" "$body" | _cs_pretty
}
```

### Create an asset folder
```bash
cs_new_asset_folder() {
    local name="$1"
    local parent_uid="${2:-}"
    local body
    body=$(python3 -c "
import json, sys
folder = {'name': sys.argv[1]}
if sys.argv[2]: folder['parent_uid'] = sys.argv[2]
print(json.dumps({'asset': folder}))
" "$name" "$parent_uid")
    cs_cma POST /v3/assets/folders "$body" | _cs_pretty
}
```

### Delete an asset folder
```bash
cs_delete_asset_folder() {
    local folder_uid="$1"
    cs_cma DELETE "/v3/assets/folders/${folder_uid}" | _cs_pretty
}
```

### Get asset versions
```bash
cs_get_asset_versions() {
    local asset_uid="$1"
    cs_cma GET "/v3/assets/${asset_uid}/versions" | _cs_pretty
}
```

---

## BULK OPERATIONS

### Bulk publish entries and assets
```bash
# cs_bulk_publish <entries_json> <assets_json> [environments_csv] [locales_csv]
# entries_json: '[{"uid":"e1","content_type":"blog_post","version":1,"locale":"en-us"}]' or '[]'
# assets_json:  '[{"uid":"a1","version":1}]' or '[]'
cs_bulk_publish() {
    local entries_json="${1:-[]}"
    local assets_json="${2:-[]}"
    local envs_csv="${3:-$CS_ENV}"
    local locales_csv="${4:-$CS_LOCALE}"
    local body
    body=$(python3 -c "
import json, sys
payload = {
    'details': {'environments': sys.argv[3].split(','), 'locales': sys.argv[4].split(',')},
    'entries': json.loads(sys.argv[1]),
    'assets':  json.loads(sys.argv[2])
}
print(json.dumps(payload))
" "$entries_json" "$assets_json" "$envs_csv" "$locales_csv")
    cs_cma POST /v3/bulk/publish "$body" | _cs_pretty
}
```

### Bulk unpublish entries and assets
```bash
cs_bulk_unpublish() {
    local entries_json="${1:-[]}"
    local assets_json="${2:-[]}"
    local envs_csv="${3:-$CS_ENV}"
    local locales_csv="${4:-$CS_LOCALE}"
    local body
    body=$(python3 -c "
import json, sys
payload = {
    'details': {'environments': sys.argv[3].split(','), 'locales': sys.argv[4].split(',')},
    'entries': json.loads(sys.argv[1]),
    'assets':  json.loads(sys.argv[2])
}
print(json.dumps(payload))
" "$entries_json" "$assets_json" "$envs_csv" "$locales_csv")
    cs_cma POST /v3/bulk/unpublish "$body" | _cs_pretty
}
```

### Bulk delete entries and assets
```bash
cs_bulk_delete() {
    local entries_json="${1:-[]}"
    local assets_json="${2:-[]}"
    local body
    body=$(python3 -c "
import json, sys
print(json.dumps({'entries': json.loads(sys.argv[1]), 'assets': json.loads(sys.argv[2])}))
" "$entries_json" "$assets_json")
    cs_cma POST /v3/bulk/delete "$body" | _cs_pretty
}
```

### Get bulk job status
```bash
cs_get_bulk_job() {
    local job_id="$1"
    cs_cma GET "/v3/bulk/jobs/${job_id}" | _cs_pretty
}
```

---

## TAXONOMY

### List all taxonomies
```bash
cs_get_taxonomies() {
    cs_cma GET /v3/taxonomies | _cs_pretty
}
```

### Get a single taxonomy
```bash
cs_get_taxonomy() {
    local tax_uid="$1"
    cs_cma GET "/v3/taxonomies/${tax_uid}" | _cs_pretty
}
```

### Create a taxonomy
```bash
cs_new_taxonomy() {
    local uid="$1"
    local name="$2"
    local description="${3:-}"
    local body
    body=$(python3 -c "
import json, sys
print(json.dumps({'taxonomy': {'uid': sys.argv[1], 'name': sys.argv[2], 'description': sys.argv[3]}}))
" "$uid" "$name" "$description")
    cs_cma POST /v3/taxonomies/ "$body" | _cs_pretty
}
```

### Update a taxonomy
```bash
cs_update_taxonomy() {
    local tax_uid="$1"
    local name="$2"
    local description="${3:-}"
    local body
    body=$(python3 -c "
import json, sys
print(json.dumps({'taxonomy': {'name': sys.argv[2], 'description': sys.argv[3]}}))
" "$tax_uid" "$name" "$description")
    cs_cma PUT "/v3/taxonomies/${tax_uid}" "$body" | _cs_pretty
}
```

### Delete a taxonomy
```bash
cs_delete_taxonomy() {
    local tax_uid="$1"
    local force="${2:-false}"
    cs_cma DELETE "/v3/taxonomies/${tax_uid}" "" "force=${force}" | _cs_pretty
}
```

### List all terms in a taxonomy
```bash
cs_get_terms() {
    local tax_uid="$1"
    local depth="${2:-0}"
    local qs=""
    [[ "$depth" -gt 0 ]] && qs="depth=${depth}"
    cs_cma GET "/v3/taxonomies/${tax_uid}/terms" "" "$qs" | _cs_pretty
}
```

### Get a single term
```bash
cs_get_term() {
    local tax_uid="$1"
    local term_uid="$2"
    cs_cma GET "/v3/taxonomies/${tax_uid}/terms/${term_uid}" | _cs_pretty
}
```

### Create a term
```bash
cs_new_term() {
    local tax_uid="$1"
    local uid="$2"
    local name="$3"
    local parent_uid="${4:-}"
    local body
    body=$(python3 -c "
import json, sys
term = {'uid': sys.argv[2], 'name': sys.argv[3]}
if sys.argv[4]: term['parent_uid'] = sys.argv[4]
print(json.dumps({'term': term}))
" "$tax_uid" "$uid" "$name" "$parent_uid")
    cs_cma POST "/v3/taxonomies/${tax_uid}/terms" "$body" | _cs_pretty
}
```

### Update a term
```bash
cs_update_term() {
    local tax_uid="$1"
    local term_uid="$2"
    local name="$3"
    local body
    body=$(python3 -c "import json, sys; print(json.dumps({'term': {'name': sys.argv[1]}}))" "$name")
    cs_cma PUT "/v3/taxonomies/${tax_uid}/terms/${term_uid}" "$body" | _cs_pretty
}
```

### Delete a term
```bash
cs_delete_term() {
    local tax_uid="$1"
    local term_uid="$2"
    local force="${3:-false}"
    cs_cma DELETE "/v3/taxonomies/${tax_uid}/terms/${term_uid}" "" "force=${force}" | _cs_pretty
}
```

### Get term descendants / ancestors
```bash
cs_get_term_descendants() {
    local tax_uid="$1"; local term_uid="$2"
    cs_cma GET "/v3/taxonomies/${tax_uid}/terms/${term_uid}/descendants" | _cs_pretty
}

cs_get_term_ancestors() {
    local tax_uid="$1"; local term_uid="$2"
    cs_cma GET "/v3/taxonomies/${tax_uid}/terms/${term_uid}/ancestors" | _cs_pretty
}
```

---

## ENVIRONMENTS

### List all environments
```bash
cs_get_environments() {
    cs_cma GET /v3/environments | _cs_pretty
}
```

### Get a single environment
```bash
cs_get_environment() {
    local env_name="$1"
    cs_cma GET "/v3/environments/${env_name}" | _cs_pretty
}
```

### Create an environment
```bash
# cs_new_environment <name> [urls_json]
# urls_json: '[{"locale":"en-us","url":"https://mysite.com"}]'
cs_new_environment() {
    local name="$1"
    local urls_json="${2:-[]}"
    local body
    body=$(python3 -c "
import json, sys
print(json.dumps({'environment': {'name': sys.argv[1], 'urls': json.loads(sys.argv[2])}}))
" "$name" "$urls_json")
    cs_cma POST /v3/environments "$body" | _cs_pretty
}
```

### Delete an environment
```bash
cs_delete_environment() {
    local env_name="$1"
    cs_cma DELETE "/v3/environments/${env_name}" | _cs_pretty
}
```

---

## LANGUAGES (LOCALES)

### List all languages
```bash
cs_get_languages() {
    cs_cma GET /v3/locales | _cs_pretty
}
```

### Add a language
```bash
cs_add_language() {
    local code="$1"           # e.g. "fr-fr"
    local name="$2"           # e.g. "French - France"
    local fallback="${3:-en-us}"
    local body
    body=$(python3 -c "
import json, sys
print(json.dumps({'locale': {'code': sys.argv[1], 'name': sys.argv[2], 'fallback_locale': sys.argv[3]}}))
" "$code" "$name" "$fallback")
    cs_cma POST /v3/locales "$body" | _cs_pretty
}
```

### Delete a language
```bash
cs_delete_language() {
    local locale_code="$1"
    cs_cma DELETE "/v3/locales/${locale_code}" | _cs_pretty
}
```

---

## RELEASES

### List all releases
```bash
cs_get_releases() {
    local skip="${1:-0}"
    local limit="${2:-100}"
    cs_cma GET /v3/releases "" "skip=${skip}&limit=${limit}" | _cs_pretty
}
```

### Get a single release
```bash
cs_get_release() {
    local release_uid="$1"
    cs_cma GET "/v3/releases/${release_uid}" | _cs_pretty
}
```

### Create a release
```bash
cs_new_release() {
    local name="$1"
    local description="${2:-}"
    local locked="${3:-false}"
    local body
    body=$(python3 -c "
import json, sys
print(json.dumps({'release': {'name': sys.argv[1], 'description': sys.argv[2], 'locked': sys.argv[3]=='true'}}))
" "$name" "$description" "$locked")
    cs_cma POST /v3/releases "$body" | _cs_pretty
}
```

### Add items to a release
```bash
# cs_add_release_items <release_uid> <items_json>
# items_json: '[{"uid":"e1","content_type_uid":"blog_post","type":"entry","action":"publish","locale":"en-us","version":1}]'
cs_add_release_items() {
    local release_uid="$1"
    local items_json="$2"
    local body
    body=$(python3 -c "import json, sys; print(json.dumps({'items': json.loads(sys.argv[1])}))" "$items_json")
    cs_cma POST "/v3/releases/${release_uid}/items" "$body" | _cs_pretty
}
```

### Deploy a release
```bash
cs_deploy_release() {
    local release_uid="$1"
    local envs_csv="${2:-$CS_ENV}"
    local locales_csv="${3:-$CS_LOCALE}"
    local action="${4:-publish}"
    local body
    body=$(python3 -c "
import json, sys
print(json.dumps({'release': {
    'environments': sys.argv[2].split(','),
    'locales':      sys.argv[3].split(','),
    'action':       sys.argv[4]
}}))
" "$release_uid" "$envs_csv" "$locales_csv" "$action")
    cs_cma POST "/v3/releases/${release_uid}/deploy" "$body" | _cs_pretty
}
```

### Delete a release
```bash
cs_delete_release() {
    local release_uid="$1"
    cs_cma DELETE "/v3/releases/${release_uid}" | _cs_pretty
}
```

---

## WORKFLOWS

### List all workflows
```bash
cs_get_workflows() {
    cs_cma GET /v3/workflows | _cs_pretty
}
```

### Get a single workflow
```bash
cs_get_workflow() {
    local wf_uid="$1"
    cs_cma GET "/v3/workflows/${wf_uid}" | _cs_pretty
}
```

### Create a workflow
```bash
# cs_new_workflow <name> [description] [content_types_csv] [stages_json]
cs_new_workflow() {
    local name="$1"
    local description="${2:-}"
    local ct_csv="${3:-}"
    local stages_json="${4:-[]}"
    local body
    body=$(python3 -c "
import json, sys
cts = [c.strip() for c in sys.argv[3].split(',')] if sys.argv[3] else []
print(json.dumps({'workflow': {
    'name':            sys.argv[1],
    'description':     sys.argv[2],
    'content_types':   cts,
    'workflow_stages': json.loads(sys.argv[4])
}}))
" "$name" "$description" "$ct_csv" "$stages_json")
    cs_cma POST /v3/workflows "$body" | _cs_pretty
}
```

### Enable / Disable workflow
```bash
cs_enable_workflow()  { cs_cma GET "/v3/workflows/${1}/enable"  | _cs_pretty; }
cs_disable_workflow() { cs_cma GET "/v3/workflows/${1}/disable" | _cs_pretty; }
```

### Delete a workflow
```bash
cs_delete_workflow() {
    cs_cma DELETE "/v3/workflows/${1}" | _cs_pretty
}
```

---

## WEBHOOKS

### List all webhooks
```bash
cs_get_webhooks() {
    cs_cma GET /v3/webhooks | _cs_pretty
}
```

### Get a single webhook
```bash
cs_get_webhook() {
    local wh_uid="$1"
    cs_cma GET "/v3/webhooks/${wh_uid}" | _cs_pretty
}
```

### Create a webhook
```bash
# cs_new_webhook <name> <target_url> [channels_csv] [enabled]
# channels_csv: "publish,unpublish,create,update,delete"
cs_new_webhook() {
    local name="$1"
    local target_url="$2"
    local channels_csv="${3:-publish}"
    local enabled="${4:-true}"
    local body
    body=$(python3 -c "
import json, sys
print(json.dumps({'webhook': {
    'name':            sys.argv[1],
    'destinations':    [{'target_url': sys.argv[2], 'http_basic_auth': '', 'http_basic_password': ''}],
    'channels':        sys.argv[3].split(','),
    'enabled':         sys.argv[4] == 'true',
    'concise_payload': True
}}))
" "$name" "$target_url" "$channels_csv" "$enabled")
    cs_cma POST /v3/webhooks "$body" | _cs_pretty
}
```

### Delete a webhook
```bash
cs_delete_webhook() {
    cs_cma DELETE "/v3/webhooks/${1}" | _cs_pretty
}
```

### Get webhook executions
```bash
cs_get_webhook_executions() {
    local wh_uid="$1"
    cs_cma GET "/v3/webhooks/${wh_uid}/executions" | _cs_pretty
}
```

### Retry a webhook execution
```bash
cs_retry_webhook() {
    local exec_uid="$1"
    cs_cma POST "/v3/webhooks/${exec_uid}/retry" | _cs_pretty
}
```

---

## TOKENS

### List management tokens
```bash
cs_get_management_tokens() {
    cs_cma GET /v3/stacks/management_tokens | _cs_pretty
}
```

### Create a management token
```bash
cs_new_management_token() {
    local name="$1"
    local description="${2:-}"
    local body
    body=$(python3 -c "
import json, sys
print(json.dumps({'token': {
    'name':        sys.argv[1],
    'description': sys.argv[2],
    'scope':       [{'module': 'environment', 'acl': {'read': True, 'write': True}}]
}}))
" "$name" "$description")
    cs_cma POST /v3/stacks/management_tokens "$body" | _cs_pretty
}
```

### List delivery tokens
```bash
cs_get_delivery_tokens() {
    cs_cma GET /v3/stacks/delivery_tokens | _cs_pretty
}
```

### Create a delivery token
```bash
cs_new_delivery_token() {
    local name="$1"
    local description="${2:-}"
    local scope_env="${3:-$CS_ENV}"
    local body
    body=$(python3 -c "
import json, sys
print(json.dumps({'token': {
    'name':        sys.argv[1],
    'description': sys.argv[2],
    'scope':       [{'environments': [sys.argv[3]], 'module': 'environment'}]
}}))
" "$name" "$description" "$scope_env")
    cs_cma POST /v3/stacks/delivery_tokens "$body" | _cs_pretty
}
```

---

## ROLES

### List all roles
```bash
cs_get_roles() {
    cs_cma GET /v3/roles | _cs_pretty
}
```

### Create a role
```bash
cs_new_role() {
    local name="$1"
    local description="${2:-}"
    local rules_json="${3:-[]}"
    local body
    body=$(python3 -c "
import json, sys
print(json.dumps({'role': {'name': sys.argv[1], 'description': sys.argv[2], 'rules': json.loads(sys.argv[3])}}))
" "$name" "$description" "$rules_json")
    cs_cma POST /v3/roles "$body" | _cs_pretty
}
```

### Delete a role
```bash
cs_delete_role() {
    cs_cma DELETE "/v3/roles/${1}" | _cs_pretty
}
```

---

## LABELS

### List all labels
```bash
cs_get_labels() {
    cs_cma GET /v3/labels | _cs_pretty
}
```

### Create a label
```bash
cs_new_label() {
    local name="$1"
    local parent_uids_json="${2:-[]}"
    local body
    body=$(python3 -c "
import json, sys
print(json.dumps({'label': {'name': sys.argv[1], 'parent': json.loads(sys.argv[2])}}))
" "$name" "$parent_uids_json")
    cs_cma POST /v3/labels "$body" | _cs_pretty
}
```

### Delete a label
```bash
cs_delete_label() {
    cs_cma DELETE "/v3/labels/${1}" | _cs_pretty
}
```

---

## AUDIT LOG

### Get audit log
```bash
cs_get_audit_log() {
    local skip="${1:-0}"
    local limit="${2:-100}"
    cs_cma GET /v3/audit-logs "" "skip=${skip}&limit=${limit}" | _cs_pretty
}
```

### Get a single audit log item
```bash
cs_get_audit_log_item() {
    local log_uid="$1"
    cs_cma GET "/v3/audit-logs/${log_uid}" | _cs_pretty
}
```

---

## PUBLISH QUEUE

### Get publish queue
```bash
cs_get_publish_queue() {
    local skip="${1:-0}"
    local limit="${2:-100}"
    cs_cma GET /v3/publish-queue "" "skip=${skip}&limit=${limit}" | _cs_pretty
}
```

### Cancel a scheduled publish action
```bash
cs_cancel_scheduled_publish() {
    local pq_uid="$1"
    cs_cma GET "/v3/publish-queue/${pq_uid}/unschedule" | _cs_pretty
}
```

---

## CONTENT DELIVERY API (CDA) — Read Published Content

The CDA functions use the CDN endpoint and delivery token. They only return **published** content.

### Get all published content types (CDA)
```bash
cs_published_content_types() {
    cs_cda /v3/content_types | _cs_pretty
}
```

### Get all published entries (CDA)
```bash
# cs_published_entries <content_type_uid> [locale] [skip] [limit] [query_json]
cs_published_entries() {
    local ct_uid="$1"
    local locale="${2:-$CS_LOCALE}"
    local skip="${3:-0}"
    local limit="${4:-100}"
    local query="${5:-}"
    local qs="locale=${locale}&skip=${skip}&limit=${limit}"
    [[ -n "$query" ]] && qs="${qs}&query=$(_cs_urlencode "$query")"
    cs_cda "/v3/content_types/${ct_uid}/entries" "$qs" | _cs_pretty
}
```

### Get a single published entry (CDA)
```bash
cs_published_entry() {
    local ct_uid="$1"
    local entry_uid="$2"
    local locale="${3:-$CS_LOCALE}"
    cs_cda "/v3/content_types/${ct_uid}/entries/${entry_uid}" "locale=${locale}" | _cs_pretty
}
```

### Get all published assets (CDA)
```bash
cs_published_assets() {
    local skip="${1:-0}"
    local limit="${2:-100}"
    cs_cda /v3/assets "skip=${skip}&limit=${limit}" | _cs_pretty
}
```

### Get a single published asset (CDA)
```bash
cs_published_asset() {
    cs_cda "/v3/assets/${1}" | _cs_pretty
}
```

### Query entries with operators (CDA)
```bash
# Equals
cs_published_entries "blog_post" "$CS_LOCALE" 0 100 '{"title":"My Post"}'

# Greater than
cs_published_entries "products" "$CS_LOCALE" 0 100 '{"price":{"$gt":100}}'

# Array IN
cs_published_entries "products" "$CS_LOCALE" 0 100 '{"price":{"$in":[99,149,199]}}'

# AND
cs_published_entries "blog_post" "$CS_LOCALE" 0 100 '{"$and":[{"title":"Hello"},{"locale":"en-us"}]}'

# OR
cs_published_entries "blog_post" "$CS_LOCALE" 0 100 '{"$or":[{"color":"Gold"},{"color":"Black"}]}'

# Regex
cs_published_entries "blog_post" "$CS_LOCALE" 0 100 '{"title":{"$regex":"^Hello"}}'
```

### Synchronization (CDA)
```bash
# Initial sync — fetches all published content
cs_sync_init() {
    cs_cda /v3/stacks/sync "init=true&type=entry_published" | _cs_pretty
}

# Continue sync with pagination token
cs_sync_page() {
    local pagination_token="$1"
    cs_cda /v3/stacks/sync "pagination_token=$(_cs_urlencode "$pagination_token")" | _cs_pretty
}

# Delta sync — only changes since last sync
cs_sync_delta() {
    local sync_token="$1"
    cs_cda /v3/stacks/sync "sync_token=$(_cs_urlencode "$sync_token")" | _cs_pretty
}
```

---

## COMMON WORKFLOWS

### Create, populate, and publish an entry end-to-end
```bash
# 1. Create entry
response=$(cs_new_entry "blog_post" '{"title":"Getting Started with Contentstack","url":"/getting-started","body":"Welcome!"}')
entry_uid=$(echo "$response" | python3 -c "import json,sys; print(json.load(sys.stdin)['entry']['uid'])")
echo "Created entry: $entry_uid"

# 2. Publish entry
cs_publish_entry "blog_post" "$entry_uid" "development,production" "en-us" 1
echo "Published entry: $entry_uid"
```

### Bulk-publish all entries of a content type
```bash
# Fetch all entries and build the bulk-publish payload
entries_json=$(cs_get_entries "blog_post" | python3 -c "
import json, sys
data = json.load(sys.stdin)
items = [{'uid': e['uid'], 'content_type': 'blog_post', 'version': 1, 'locale': 'en-us'} for e in data['entries']]
print(json.dumps(items))
")
cs_bulk_publish "$entries_json" "[]" "$CS_ENV" "$CS_LOCALE"
```

### Deploy a set of content changes via a Release
```bash
# 1. Create a release
release_uid=$(cs_new_release "Sprint-42 Release" "Blog updates" | \
    python3 -c "import json,sys; print(json.load(sys.stdin)['release']['uid'])")

# 2. Add entries to the release
items='[
  {"uid":"entry_uid_1","content_type_uid":"blog_post","type":"entry","action":"publish","locale":"en-us","version":2},
  {"uid":"entry_uid_2","content_type_uid":"blog_post","type":"entry","action":"publish","locale":"en-us","version":1}
]'
cs_add_release_items "$release_uid" "$items"

# 3. Deploy the release
cs_deploy_release "$release_uid" "production" "en-us"
```

### Upload and publish an asset
```bash
asset_uid=$(cs_upload_asset "/home/user/images/banner.jpg" "Home page banner" | \
    python3 -c "import json,sys; print(json.load(sys.stdin)['asset']['uid'])")
cs_publish_asset "$asset_uid" "production" "en-us"
```

### Sync published content and save to file
```bash
sync=$(cs_sync_init)
echo "$sync" > /tmp/cs-sync-all.json

# Follow pagination tokens until exhausted
while true; do
    token=$(echo "$sync" | python3 -c "import json,sys; d=json.load(sys.stdin); print(d.get('pagination_token',''))" 2>/dev/null)
    [[ -z "$token" ]] && break
    sync=$(cs_sync_page "$token")
    echo "$sync" >> /tmp/cs-sync-all.json
done

final_token=$(echo "$sync" | python3 -c "import json,sys; print(json.load(sys.stdin).get('sync_token',''))")
echo "Sync complete. Token: $final_token"
```

### Find and update entries matching a query
```bash
result=$(cs_get_entries "blog_post" "$CS_LOCALE" 0 100 '{"published_at":{"$exists":false}}')
echo "$result" | python3 -c "
import json, sys
for e in json.load(sys.stdin)['entries']:
    print(e['uid'])
" | while read -r uid; do
    cs_update_entry "blog_post" "$uid" '{"status":"draft"}'
done
```

---

## FURTHER REFERENCE — POSTMAN COLLECTIONS

Refer to the Postman collection JSON files in `~/.claude/skills/contentstack/` for details beyond this skill:

| File | Covers |
|---|---|
| `Content Management API - Contentstack.postman_collection.json` | All write operations — exact URLs, HTTP methods, headers, and sample bodies |
| `Content Delivery API - Contentstack.postman_collection.json` | All read operations against published content |

**Quick search in bash:**
```bash
skill_dir="$HOME/.claude/skills/contentstack"
python3 - "$skill_dir/Content Management API - Contentstack.postman_collection.json" <<'EOF'
import json, sys

def find_endpoints(items, search):
    for item in items:
        if 'item' in item:
            find_endpoints(item['item'], search)
        elif search.lower() in item.get('name','').lower():
            req = item['request']
            url = req['url'] if isinstance(req['url'], str) else req['url'].get('raw','')
            print(f"Name:   {item['name']}")
            print(f"Method: {req['method']}")
            print(f"URL:    {url}")
            body = req.get('body',{}).get('raw','')
            if body: print(f"Body:   {body[:200]}")
            print()

with open(sys.argv[1]) as f:
    cma = json.load(f)

find_endpoints(cma['item'], 'atomic')   # change search term as needed
EOF
```

---

## TIPS & TROUBLESHOOTING

| Issue | Fix |
|---|---|
| `curl: (6) Could not resolve host` | Check `$CS_CMA_URL` / `$CS_CDA_URL` are set and match your region |
| `{"error_message":"Unauthorized","error_code":105}` | Verify `$CS_API_KEY` and `$CS_MANAGEMENT_TOKEN` are exported correctly |
| `{"error_message":"You are not allowed..."}` | Management token lacks write scope; regenerate with read+write |
| `{"error_code":422}` | Schema validation failure — inspect the full response body |
| Rate limit (HTTP 429) | Add `sleep 0.5` between bulk loop iterations |
| Large result sets | Use `skip`/`limit` pagination; max `limit` is 100 per request |
| JSON parse error in python3 | Wrap argument in single quotes or use a heredoc to avoid shell interpolation |
| Multipart upload fails | Ensure the file path is absolute and the file exists: `[[ -f "$file_path" ]]` |
| `jq` not installed | All functions fall back to `python3 -m json.tool` automatically via `_cs_pretty` |
