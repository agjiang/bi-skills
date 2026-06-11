---
name: medallion-architecture
description: Stamp a dbt marts model as gold tier — adds owner metadata and strict grain tests. Use when the user says things like "apply medallion architecture to <model>", "mark <model> gold", "stamp <model> gold", "add me as owner and gold medallion to <model>", or "promote <model> to gold". Only trigger when the user explicitly names a specific table.
license: MIT
---

# Medallion Architecture — gold-stamp a dbt marts model

Promote a dbt marts model to "gold" status: stamp ownership metadata on the SQL model and
enforce its grain with strict tests in the schema YAML. The grain tests use `severity: error`
deliberately — this overrides the repo's default "always warn" testing convention, because a
gold model's uniqueness contract breaking should fail the run, not warn quietly.

## When to use

Only apply this when the user explicitly names specific tables — e.g. "apply medallion
architecture to <model>", "stamp <model> gold", or "add me as owner and gold medallion to
<model>". Do NOT trigger proactively or suggest applying it to other tables. Whether a table
deserves gold status is the user's judgment call.

## Step 1 — Resolve the owner email

Run `whoami` and append `@playlist.com` (e.g. `agnes.jiang` → `agnes.jiang@playlist.com`).
Always use `@playlist.com`, never `@classpass.com`.

## Step 2 — Stamp the SQL config block

Add or update `owner` and `medallion_layer` in the model's config `meta`:

```sql
{{
    config(
        materialized = 'table',
        meta = {'owner': '<user>@playlist.com',
                'medallion_layer': 'gold'}
    )
}}
```

Preserve the file's existing config syntax style (quote style, existing keys like `tags`).

## Step 3 — Update the schema YAML description

Convert the model description to a `>` block-scalar prefixed with `Gold mart:`:

```yaml
  - name: <model_name>
    description: >
        Gold mart: <what this model represents>.
```

## Step 4 — Determine the model's grain

- **Single-column primary key**: usually `<object>_id`.
- **Composite grain**: infer from join/group-by structure of the SQL, or look for an existing
  `dbt_utils.unique_combination_of_columns` test.
- **Unclear**: ASK the user before writing any test. Do not guess — a wrong grain assertion at
  severity error will fail production runs.

## Step 5 — Enforce the grain at severity error

**Single-column PK** — column-level tests; the PK's description becomes `Primary key.`:

```yaml
      - name: <pk_column>
        description: Primary key.
        tests:
          - unique:
              config:
                severity: error
          - not_null:
              config:
                severity: error
```

**Composite grain** — model-level test plus `not_null` (error) on each grain component:

```yaml
    tests:
      - dbt_utils.unique_combination_of_columns:
          arguments:
            combination_of_columns:
              - col_a
              - col_b
          config:
            severity: error
    columns:

      - name: col_a
        description: ...
        tests:
          - not_null:
              config:
                severity: error
```

Important notes:
- The `arguments:` wrapper is required (`dbt_project.yml` sets `require_generic_test_arguments_property: true`).
- Do NOT create a `pk_hash` / surrogate-key column. Use the model-level combination test instead.
- If tests already exist at `severity: warn`, escalate them to `error` in place.
- Leave all other columns' tests at their existing severity.

## Step 6 — Verify

```bash
dbt parse          # catches YAML/test syntax errors immediately
dbt build -s <model_name>   # builds model and runs tests
```
