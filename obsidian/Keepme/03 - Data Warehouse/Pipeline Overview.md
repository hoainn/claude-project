# Data Warehouse Pipeline Overview

## Architecture

```
MongoDB Atlas
    ↓  Debezium MongoDB Source Connector
Kafka Topics  (cdc.keep-me.{collection})
    ↓  S3 Sink Connector (Avro → Parquet)
S3  (year/month/day/hour partitioned)
    ↓  Partition registrar (hourly CronJob)
AWS Glue Data Catalog  (cdc_keepme_production)
    ↓  Redshift Spectrum
External Tables  (cdc_raw.{table})
    ↓  Mapped Views
Redshift Views  (cdc.{table})
    ↓  Datashare
"keep-me" DB  (user-facing, read-only)
```

## CDC Collections (10 total)

| Collection | Topic |
|------------|-------|
| `visitors` | `cdc.keep-me.visitors` |
| `visitor_leads` | `cdc.keep-me.visitor_leads` |
| `branches` | `cdc.keep-me.branches` |
| `branchmappings` | `cdc.keep-me.branchmappings` |
| `clients` | `cdc.keep-me.clients` |
| `regions` | `cdc.keep-me.regions` |
| `leadmappings` | `cdc.keep-me.leadmappings` |
| `history` | `cdc.keep-me.history` |
| `redis_jobs` | `cdc.keep-me.redis_jobs` |
| `re_engaged_leads` | `cdc.keep-me.re_engaged_leads` |

## S3 Buckets

| Env | Bucket |
|-----|--------|
| Production | `antares-data-warehouse-prod` |
| Development | `antares-data-warehouse-dev` |

Path format: `topics/cdc.keep-me.{collection}/year=YYYY/month=MM/day=DD/hour=HH/*.parquet`

## Glue

| Setting | Value |
|---------|-------|
| Prod DB | `cdc_keepme_production` |
| Dev DB | `cdc_keepme_development` |
| Tables | 10 (one per collection), defined in `glue.tf` |
| Partition format | `year=YYYY/month=MM/day=DD/hour=HH` |
| Partition registrar | CronJob `glue-partition-registrar`, hourly, `batch_create_partition` |

> **No Glue Crawler.** Schema changes are applied manually — update `glue.tf` and run `terraform apply`. Redshift Spectrum picks up new columns automatically on the next query.

## Redshift Serverless

| Setting | Value |
|---------|-------|
| Workgroup | `dev` |
| Region | `eu-west-2` |
| Namespace | `antares-data-warehouse` |
| IAM Role | `AmazonRedshift-CommandsAccessRole-20260321T132138` |
| Base capacity | 128 RPUs |
| External schema | `cdc_raw` (→ Glue `cdc_keepme_production`) |
| Views | `cdc.{table}` |
| Datashare | `"keep-me"."cdc"."<table>"` (read-only) |

## Critical Rules

- **Always query with partition filters**: `WHERE year='...' AND month='...' AND day='...'`
- **Views**: use `CREATE OR REPLACE VIEW ... WITH NO SCHEMA BINDING` — no DROP unless changing an existing column's output type
- **Datashare limitation**: does NOT support SUPER type — use `NULL::varchar AS <col>` for arrays/nested
- **Deduplication**: use `INNER JOIN MAX(__ts_ms)` — window functions fail on external tables
- **Glue is source of truth**: never ALTER Redshift external tables directly
- **Never update**: `dev_external_tables.sql`
- **Always update views in `keep-me` DB** — not `dev` (dev points to a non-existent Glue DB)

## Related Notes
- [[Kafka Connectors]]
- [[SMT Transforms]]
- [[Glue & Redshift]]
- [[Adding a Field]]
- [[../05 - Operations/Connector Reset Procedure]]
