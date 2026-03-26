# Kafka Connectors

## Source Connectors (MongoDB → Kafka)

Location: `keepme-argo/prod/data-warehouse/kafka-connect-connectors/source-*/values.yaml`

- Type: Debezium MongoDB Source Connector
- One connector per collection
- `snapshot.mode: "never"` (prevent accidental re-snapshots)
- `capture.mode: "change_streams_update_full"`

## Sink Connectors (Kafka → S3)

Location: `keepme-argo/prod/data-warehouse/kafka-connect-sink-connectors/sink-*/values.yaml`

- Type: Confluent S3 Sink Connector
- Converts Avro → Parquet
- Partitions by `year/month/day/hour`
- DLQ: `dlq-sink-{collection}`

## Scaling via GitOps

```yaml
replicaCount: 1   # up
replicaCount: 0   # down
connector:
  enabled: true   # always keep true
```

Always commit + push + trigger ArgoCD sync after changes.

## excludeFields Format

```
keep-me.<collection>.<field>               # top-level
keep-me.<collection>.<nested>.<subfield>   # nested
```

Only exclude fields NOT in developer schema JSON. Fields in schema should flow through.

## Debezium Offset Rewrite — CRITICAL

Before rewriting an offset key:
1. Inspect the stored offset **value**
2. If it contains `"initsync": true` → **STOP** — connector is in snapshot phase
3. Proceeding causes full re-snapshot → duplicate S3 data

## Connector ArgoCD App Names

| Collection | Source App | Sink App |
|------------|-----------|---------|
| visitors | `prod-dw-connector-source-visitors` | `prod-dw-connector-sink-visitors` |
| visitor_leads | `prod-dw-connector-source-visitor-leads` | `prod-dw-connector-sink-visitor-leads` |
| branches | `prod-dw-connector-source-branches` | `prod-dw-connector-sink-branches` |
| branchmappings | `prod-dw-connector-source-branchmappings` | `prod-dw-connector-sink-branchmappings` |
| clients | `prod-dw-connector-source-clients` | `prod-dw-connector-sink-clients` |
| regions | `prod-dw-connector-source-regions` | `prod-dw-connector-sink-regions` |
| leadmappings | `prod-dw-connector-source-leadmappings` | `prod-dw-connector-sink-leadmappings` |
| history | `prod-dw-connector-source-history` | `prod-dw-connector-sink-history` |
| redis_jobs | `prod-dw-connector-source-redis-jobs` | `prod-dw-connector-sink-redis-jobs` |
| re_engaged_leads | `prod-dw-connector-source-re-engaged-leads` | `prod-dw-connector-sink-re-engaged-leads` |

## Schema Registry Rules

- Do NOT auto-check Schema Registry after scaling — wait for user to ask
- Do NOT auto-reset Kafka/S3 after scale down unless explicitly asked

## Related Notes
- [[SMT Transforms]]
- [[../05 - Operations/Connector Reset Procedure]]
- [[../02 - Infrastructure/ArgoCD & GitOps]]
