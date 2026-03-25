# Working Preferences

## Responses
- Concise responses, no trailing summaries
- Skip explaining obvious steps

## ArgoCD / GitOps
- After every `git push` to keepme-argo, always trigger ArgoCD sync immediately:
  ```bash
  kubectl annotate application <app-name> -n argocd argocd.argoproj.io/refresh=normal --overwrite
  ```

## Data Pipeline
- Do NOT update `keepme-infra/deployments/keepme-datawarehouse/redshift/dev_external_tables.sql`
- Do NOT auto-check Schema Registry after scaling up a connector — wait for user to ask
- Do NOT auto-query Redshift after Glue changes — wait for user to ask
- Do NOT auto-reset Kafka/S3 after scale down unless user explicitly asks — scale down alone is sometimes enough

## Planning
- Default to executing directly — do not plan unless the task is large, multi-system, and has irreversible steps
- Plan when: full pipeline reset, schema migration across multiple systems, approach is unclear

## Connector Reset Order
1. Scale down source + sink (commit + push + ArgoCD sync both)
2. Delete ALL Kafka topics: CDC (`cdc.keep-me.<collection>`), DLQ (`dlq-sink-<name>`), all internal `_connect-configs/offsets/status-*` topics for both source and sink
3. Delete consumer groups if any
4. Purge Schema Registry subject (soft delete + permanent delete)
5. Clear S3: `aws s3 rm s3://antares-data-warehouse-prod/topics/cdc.keep-me.<collection>/ --recursive`
6. Scale up **source only**
7. Wait for user to confirm before scaling up sink

## Connector Config — excludeFields
- Only exclude fields NOT in the developer schema JSON (`/Kafka_schema/<collection>.json`)
- Fields in schema JSON should flow through — handle array types via Glue `array<string>`, not exclusion
- Format: `keep-me.<collection>.<field>` (top-level), `keep-me.<collection>.<nested>.<subfield>` (nested)

## Glue / Redshift View Updates
- After Glue update, register partition via `batch_create_partition`
- View syntax: `CREATE VIEW cdc.<name> AS SELECT ... FROM cdc_raw.<name> WHERE year=... WITH NO SCHEMA BINDING;`
- For columns missing from Parquet files (snapshot in progress): use `NULL::varchar AS <col>` in view
- Datashare (`keep-me` DB) does NOT support SUPER type — never use `json_serialize()` in views
- Verify with: `SELECT * FROM "keep-me"."cdc"."<table>" LIMIT 5;`
