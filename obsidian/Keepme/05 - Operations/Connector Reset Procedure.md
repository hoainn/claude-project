# Connector Reset Procedure

Use this when a connector needs a full reset (schema conflict, re-snapshot, data corruption).

## Order of Operations

### Step 1 — Scale down both connectors
```yaml
# source values.yaml
replicaCount: 0

# sink values.yaml
replicaCount: 0
```
Commit + push + ArgoCD sync **both**.

### Step 2 — Delete ALL Kafka topics
```bash
kubectl exec kafka-controller-0 -n data-warehouse-prod -c kafka -- \
  env JMX_PORT="" KAFKA_JMX_OPTS="" \
  /opt/bitnami/kafka/bin/kafka-topics.sh \
  --bootstrap-server 10.200.170.138:9092 \
  --delete --topic cdc.keep-me.<collection>

# Also delete:
# dlq-sink-<name>
# _connect-configs-*, _connect-offsets-*, _connect-status-* (source AND sink)
```

### Step 3 — Delete consumer groups (if any)
```bash
kubectl exec kafka-controller-0 -n data-warehouse-prod -c kafka -- \
  env JMX_PORT="" KAFKA_JMX_OPTS="" \
  /opt/bitnami/kafka/bin/kafka-consumer-groups.sh \
  --bootstrap-server 10.200.170.138:9092 \
  --delete --group <group-name>
```

### Step 4 — Purge Schema Registry subject
```bash
# Soft delete
curl -X DELETE http://10.200.95.52:8081/subjects/cdc.keep-me.<collection>-value

# Permanent delete
curl -X DELETE "http://10.200.95.52:8081/subjects/cdc.keep-me.<collection>-value?permanent=true"
```

### Step 5 — Clear S3
```bash
aws s3 rm s3://antares-data-warehouse-prod/topics/cdc.keep-me.<collection>/ --recursive
```

### Step 6 — Scale up source only
```yaml
# source values.yaml
replicaCount: 1
```
Commit + push + ArgoCD sync.

### Step 7 — Wait
Do NOT scale up sink until user confirms source is healthy and producing data correctly.

---

## Pre-Reset Checklist

- [ ] Check offset value for `"initsync": true` before any offset rewrite
- [ ] Confirm both connectors are at `replicaCount: 0` and synced
- [ ] Verify all internal topics deleted (configs, offsets, status)
- [ ] Schema Registry purged (soft + permanent)
- [ ] S3 cleared

## Rules

- Do NOT auto-reset after scale down alone — scale down may be enough
- Always wait for user confirmation before scaling up sink
- Scale down alone is sometimes sufficient — only do full reset if explicitly requested
