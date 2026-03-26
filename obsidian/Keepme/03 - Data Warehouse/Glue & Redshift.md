# Glue & Redshift

## Glue Data Catalog

- Prod DB: `cdc_keepme_production`
- Dev DB: `cdc_keepme_development`
- **No crawler** — schema changes are manual: update `glue.tf` → `terraform apply`

### Array columns in Glue
Define as `array<string>` — do NOT exclude array fields from connector.

### After Glue schema update
1. Run `terraform apply`
2. Register current partition via `batch_create_partition` API
3. New column appears in Redshift Spectrum on next query automatically

---

## Redshift External Tables (cdc_raw)

```sql
CREATE EXTERNAL TABLE cdc_raw.visitors (
  _id VARCHAR,
  organization_id VARCHAR,
  ...
  __op VARCHAR,
  __ts_ms BIGINT
)
PARTITIONED BY (year VARCHAR, month VARCHAR, day VARCHAR, hour VARCHAR)
STORED AS PARQUET
LOCATION 's3://antares-data-warehouse-prod/topics/cdc.keep-me.visitors/';
```

- Column names match Parquet/Avro output (Debezium naming)
- Types: VARCHAR, BIGINT, BOOLEAN — no SUPER
- Glue is source of truth — never ALTER external tables directly

---

## Redshift Views (cdc schema)

```sql
CREATE OR REPLACE VIEW cdc.visitors AS
SELECT
  _id AS visitor_id,
  organization_id AS client_id,
  dateadd(ms, created_date::bigint, '1970-01-01') AS created_date,
  ...
  NULL::varchar AS array_field,   -- array/nested: datashare limit
  __op,
  __ts_ms
FROM cdc_raw.visitors
WHERE __op != 'd'
WITH NO SCHEMA BINDING;
```

### Rules
- Use `CREATE OR REPLACE VIEW` — no DROP needed unless changing an existing column's **output type**
- Always end with `WITH NO SCHEMA BINDING`
- Filter soft-deletes: `WHERE __op != 'd'`
- Epoch ms → timestamp: `dateadd(ms, col::bigint, '1970-01-01')`
- Array/nested columns: `NULL::varchar AS <col>` (datashare does not support SUPER)
- For missing columns (snapshot in progress): `NULL::varchar AS <col>`
- Update views in `keep-me` database, NOT `dev`
- Verify: `SELECT * FROM "keep-me"."cdc"."<table>" LIMIT 5;`

---

## Datashare ("keep-me" DB)

- Read-only, user-facing
- Query: `SELECT * FROM "keep-me"."cdc"."<table>" LIMIT 5;`
- **Does NOT support SUPER type** — never use `json_serialize()` in views
- For array/nested fields: use `NULL::varchar AS <col>` in views

---

## Deduplication

Window functions (`ROW_NUMBER() OVER`) fail on external tables.

Use instead:
```sql
SELECT v.*
FROM cdc_raw.visitors v
INNER JOIN (
  SELECT _id, MAX(__ts_ms) AS max_ts
  FROM cdc_raw.visitors
  WHERE year='...' AND month='...' AND day='...'
  GROUP BY _id
) dedup ON v._id = dedup._id AND v.__ts_ms = dedup.max_ts
```

---

## Query Tips

- Always include partition filters: `WHERE year='YYYY' AND month='MM' AND day='DD'`
- Full S3 scans are very slow without partitions
- Do NOT auto-query Redshift after Glue changes — wait for user to ask
