# Reset Connector

Full reset of a Kafka Connect connector pipeline: scale down, wipe all Kafka state, clear S3, scale up source only.

## Usage
`/project:reset-connector <connector-name>`

Example: `/project:reset-connector redis-jobs`

## Steps

1. Scale down **both** source and sink (`replicaCount: 0`), commit + push + ArgoCD sync, wait for pods to terminate
2. Delete ALL Kafka topics:
   - CDC topic: `cdc.keep-me.<collection>`
   - DLQ topic: `dlq-sink-<name>`
   - Internal topics: `_connect-configs/offsets/status-prod-dw-connector-source-<name>` and `_connect-configs/offsets/status-sink-<name>` and `_connect-configs/offsets/status-sink-sink-<name>`
3. Check and delete consumer groups (`connect-S3-Sink-<Name>`)
4. Purge Schema Registry subject (soft delete + `?permanent=true`)
5. Clear S3 via **background sub-agent** (`aws s3 rm ... --recursive`)
6. Scale up **source only** (`replicaCount: 1`), commit + push + ArgoCD sync
7. Check Schema Registry for newly registered schema
8. Update Glue table to match
9. **Ask user before scaling up sink**
10. After confirmation: scale up sink, register partitions, create/update view, verify with SELECT

See full runbook: Step 9 in the `debug-data-pipeline` skill.
