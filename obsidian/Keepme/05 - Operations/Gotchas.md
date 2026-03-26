# Gotchas & Non-Obvious Behaviours

## Kafka

- **JMX port**: All Kafka CLI commands must include `env JMX_PORT="" KAFKA_JMX_OPTS=""` or they fail
- **Connector reset**: Never reset Kafka/S3 automatically after scale down — wait for explicit instruction
- **Schema Registry**: Do not auto-check after scaling up — wait for user

## Debezium / Kafka Connect

- **Offset rewrite**: Always check stored offset **value** first. If it contains `"initsync": true` → STOP. Proceeding causes full re-snapshot and duplicate S3 data.
- **snapshot.mode**: Keep as `"never"` on all connectors after initial setup
- **excludeFields**: Only exclude fields NOT in schema JSON. Fields in schema must flow through (even arrays).
- **Array fields**: Handle via Glue `array<string>`, not by excluding from connector

## SMT

- **flatten.struct**: Only needed for nested objects (sub-documents), not arrays
- **Avro sanitization**: ` - ` in field keys becomes `___` → need `renameFields` SMT
- **castDates**: BSON dates come through as epoch long — always cast date fields to string

## Redshift

- **WITH NO SCHEMA BINDING**: Required on all views over external tables
- **Partition filters**: Always include `year`, `month`, `day` in WHERE clause — full scans are very slow
- **Datashare (keep-me DB)**: Does NOT support SUPER type — use `NULL::varchar AS <col>` for arrays
- **Window functions**: Fail on external tables — use `INNER JOIN MAX(__ts_ms)` for deduplication
- **dev_external_tables.sql**: Never update this file
- **Always update views in `keep-me` database**, not `dev` (dev points to non-existent Glue DB)

## Backend

- **Tests**: Use SQLite `:memory:` — never run against real DB
- **Multi-tenancy**: Company code in JWT → automatic DB connection routing, no manual handling needed
- **API**: Two APIs — Node.js for auth only, V4 PHP for everything else

## ArgoCD

- **After every git push to keepme-argo**: Always trigger ArgoCD refresh immediately
- **connector.enabled**: Never set to `false` unless deleting the Deployment entirely
- **Kafka StatefulSet**: ignoreDifferences configured to prevent rolling restarts from annotation drift

## Git

- **SSH key**: `GIT_SSH_COMMAND="ssh -i /Users/hoainn/Documents/Project/Keepme/keepme_minhlt"`
- **Never commit**: `secret/`, `keepme_minhlt`, `keepme_server.pem`, `.env` files
