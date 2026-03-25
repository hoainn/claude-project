# Debug Data Pipeline

Debug issues in the KeepMe data pipeline for a specific table.

## Usage
`/project:debug-pipeline <table>`

## Steps

1. Run failing query on Redshift and capture error
2. Check Schema Registry for type conflicts across all versions
3. Compare Schema Registry vs Glue table columns
4. Fix Glue table (add/remove/retype columns) — Redshift syncs automatically
5. Re-register S3 partitions if table was recreated
6. Rebuild the `cdc.<table>` view

See full runbook: invoke `/debug-data-pipeline` skill.
