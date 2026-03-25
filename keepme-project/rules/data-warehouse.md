# Data Warehouse Rules

## Glue is the source of truth
- Never ALTER a Redshift external table directly — update Glue, Redshift auto-syncs
- `MSCK REPAIR TABLE` is invalid in Redshift — use Glue `batch_create_partition` API instead

## Schema compatibility
- Columns with type conflicts across Schema Registry versions must be removed from Glue
- `array` types: define as `array<string>` in Glue — but NOT queryable via datashare (SUPER limitation)
- `timestamp-millis` ↔ `string` conflict → remove the column
- `int` ↔ `string` conflict → remove the column

## Redshift / Datashare
- External schema in Redshift: `cdc_raw` (maps to Glue DB `cdc_keepme_production`)
- User-facing database: `keep-me` (datashare) — query as `SELECT * FROM "keep-me"."cdc"."<table>"`
- **Datashare does NOT support SUPER type** — views with `array<string>` columns will fail via datashare
- For array columns in views: use `NULL::varchar AS <col>` instead of `json_serialize()` for datashare compatibility

## Views
- `WITH NO SCHEMA BINDING` goes at the **END** of the SQL, not after the view name:
  ```sql
  CREATE VIEW cdc.visitors AS SELECT ... FROM cdc_raw.visitors WHERE ... WITH NO SCHEMA BINDING;
  ```
- Always use `WITH NO SCHEMA BINDING` on views over external tables
- Quote reserved words: `"time"`, `"type"`, `"date"`
- `created_date` / `updated_date` cast with `::TIMESTAMP` (they are string in Parquet)
- `__ts_ms` is `BIGINT` → convert with `dateadd(millisecond, __ts_ms, '1970-01-01'::timestamp)`
- For columns that may not exist in all Parquet files yet (snapshot in progress): use `NULL::varchar AS <col>`
  - Spectrum with Parquet throws "column does not exist" if a column is in Glue schema but absent from ALL files in the partition

## Partitions
- Always include `year`, `month`, `day` filters in queries to avoid full S3 scans
- Partition format: `year=YYYY/month=MM/day=DD/hour=HH`
- Register via Glue `batch_create_partition` — update partition SD Location to match S3 path

## excludeFields pattern
- Format: `keep-me.<collection>.<field>` for top-level, `keep-me.<collection>.<nested>.<subfield>` for nested
- Only exclude fields NOT defined in the developer schema JSON (`/Kafka_schema/<collection>.json`)
- Fields in the schema JSON should flow through (even if array — handle via SMT or Glue type)

## SMT castDates
- Use `Cast$Value` to cast `timestamp-millis` → `string` and prevent type conflicts
- Also cast `person_id` and any field that changes between `int` and `string` across documents
- Arrays cannot be cast to string via `Cast$Value` SMT — use Glue `array<string>` type instead

## SQL file
- Do NOT update `keepme-infra/deployments/keepme-datawarehouse/redshift/dev_external_tables.sql` — this step is skipped

## Debezium offset rewrite — CRITICAL
When rewriting a connector offset key, always inspect the stored offset **value** first:
- If value contains `"initsync":true` → **STOP immediately**, flag to user — this is a snapshot-phase token
  - Proceeding will cause Debezium to trigger a full re-snapshot → duplicate data in S3/Parquet
  - Ask user: re-snapshot intentionally, or replace value with a valid streaming-phase resume_token?
- If value has `sec`/`ord`/`resume_token` with NO `initsync` → safe to rewrite key only, preserve value
- Always add `snapshot.mode: "never"` to connector config as a safeguard against accidental re-snapshots

## Duplicate data in Parquet (S3)
- Re-snapshots append new Parquet files alongside old ones — S3 sink never overwrites
- Fix options:
  1. Deduplicate in Redshift view: `ROW_NUMBER() OVER (PARTITION BY _id ORDER BY __ts_ms DESC NULLS LAST)`
  2. Clear S3 prefix + reset sink to re-process topic from offset 0 (clean but destructive)
