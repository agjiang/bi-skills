# Golden Dashboard Skill

Reverse-engineer a Tableau Cloud workbook and write a dbt `exposures:` YAML entry into the
appropriate mart folder. No warehouse queries — everything comes from the Tableau VizPortal API
and the downloaded `.twb` XML file.

## Trigger

Use when the user provides a Tableau workbook URL and asks to create a dbt exposure, document
a dashboard, or "stamp" a dashboard golden.

## Requirements

A browser backend must be available:
- **Playwright MCP** (preferred for Claude Code): `claude mcp add playwright -- npx @playwright/mcp@latest`
- **Claude in Chrome extension**: user must be signed into Tableau in Chrome

The Playwright browser has its own session. If it redirects to a Tableau login page, pause and
ask the user to sign in before continuing.

## Process

### Step 1 — Navigate and confirm login

Navigate to the workbook URL. Confirm the user is logged in (their avatar visible, not a login
page). Capture the workbook title.

### Step 2 — Get workbook metadata via VizPortal API

Call `POST /vizportal/api/web/v1/getWorkbooks` with the `X-XSRF-TOKEN` cookie header and filter
by workbook ID (from the URL). Extract:
- `luid`, `id`, `name`, `repositoryUrl`, `downloadUrl`
- `ownerId`, owner display name and email (from the `users` array)
- `defaultViewUrl`, `sheetCount`, `hasExtracts`
- Project name (from the `projects` array)

### Step 3 — Download and parse the `.twb`

Fetch `https://<SERVER>/t/<SITE>/workbooks/<RepositoryUrl>.twb` in-page using `browser_evaluate`
with the XSRF token. Start in the background (`window.__twbRaw`), poll for completion.

Parse the XML with `DOMParser`:
- `relation[type="text"]` → Custom SQL queries
- `relation[type="table"]` → plain table references (`[DB].[SCHEMA].[TABLE]` format)
- `connection:not([class="federated"])` → Snowflake connection details
- `column > calculation[formula]` → calculated fields

### Step 4 — Get viz structure (light skim for description)

Navigate to the default view and read aria-labels from the viz iframe:
- `[aria-label]` matching `/Data Visualization|chart of/i` → viz type and measure
- `h1,h2,h3` → sheet title and filter names

For multi-sheet workbooks, switch tabs using the full mouse event sequence
(`pointerdown`, `mousedown`, `pointerup`, `mouseup`, `click`) dispatched on the tab element.

### Step 5 — Map physical tables to dbt models

For each physical table found in the SQL or table relations, resolve the dbt reference in order
of preference:

1. **Mart model** (`ref('model_name')`) — preferred; use the future source of truth if one exists
   or is planned in `models/marts/`
2. **Staging model** (`ref('stg_...')`) — use only if no mart model exists
3. **Source** (`source('source_name', 'table_name')`) — last resort for tables with no dbt model

Avoid referencing `cp_bi_derived` as a source name — it is the old database. If a table has no
dbt equivalent (e.g. utility/ETL tables like `ETL_REPORT_LOG_ALL`), omit it from `depends_on`
and note it in the description.

Search the repo: `grep -r "<table_name>" models/ --include="*.sql" --include="*.yml" -l`

### Step 6 — Write the exposure YAML

**Output file:** `models/marts/<domain>/exposures_<domain>.yml`
- If the file already exists, append the new exposure entry.
- Multiple exposures for the same domain belong in the same file.

**Template:**
```yaml
version: 2

exposures:
  - name: <snake_case_workbook_title>
    label: <Workbook Title>
    type: dashboard
    maturity: high
    url: https://<SERVER>/#/site/<SITE>/workbooks/<id>/views
    description: >
      <Goal of the dashboard and who uses it. Core measure(s). Available filters.
      No Snowflake connection details, infrastructure, or technical SQL notes.>
    depends_on:
      - ref('<mart_model>')
      # add more ref() / source() entries as needed
    owner:
      name: <Display Name>
      email: <username>@playlist.com
```

## Conventions

- **Owner email:** always `<whoami>@playlist.com` — never use `@mindbodyonline.com` or whatever
  Tableau shows for the owner.
- **Description style:** focus on the business goal and key information — what the dashboard
  answers, who uses it, the core measure(s), and available filters. Do not mention Snowflake
  connection details, warehouse names, database/schema paths, or extract vs. live status.
- **`depends_on`:** prefer mart `ref()` over staging `ref()` over `source()`. When the team is
  migrating a table to a new dbt model, point to the future model, not the old source.
- **No data values:** never extract or document actual data from the viz — structure and logic only.
