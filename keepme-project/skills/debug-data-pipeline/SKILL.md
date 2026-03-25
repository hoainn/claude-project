---
name: debug-data-pipeline
description: Debug data pipeline and data warehouse issues on Redshift Spectrum + Glue + Kafka. Use when user says "debug data pipeline", "redshift query failing", "spectrum scan error", "glue schema mismatch", "kafka connector issue", "missing columns in redshift", "parquet schema incompatible", or "data warehouse debug".
license: MIT
metadata:
    skill-author: KeepMe
---

# Debug Data Pipeline & Redshift Data Warehouse

Systematically diagnose and fix issues in the KeepMe data pipeline:

```
MongoDB → Kafka Connect (Debezium) → Kafka → S3 (Parquet) → Glue Data Catalog → Redshift Spectrum
```

## Environment

| Component | Value |
|---|---|
| Redshift Serverless | workgroup `dev`, region `eu-west-2` |
| Prod DB | `keep-me` |
| Dev DB | `dev` |
| Glue DB (prod) | `cdc_keepme_production` |
| Glue DB (dev) | `cdc_keepme_development` |
| S3 (prod) | `s3://antares-data-warehouse-prod/topics/cdc.keep-me.<table>/` |
| S3 (dev) | `s3://antares-data-warehouse-dev/topics/cdc.keepme-development.<table>/` |
| Schema Registry | `http://10.200.95.52:8081` (K8s namespace: `data-warehouse-prod`) |
| SQL definition file | `keepme-infra/deployments/keepme-datawarehouse/redshift/dev_external_tables.sql` |
| Kafka bootstrap server | `10.200.170.138:9092` (ClusterIP of `kafka` svc in `data-warehouse-prod`) |

---

## Step 1 — Run the query and capture the error

```bash
STMT_ID=$(aws redshift-data execute-statement \
  --region eu-west-2 \
  --workgroup-name dev \
  --database keep-me \
  --sql '<query>' \
  --query 'Id' --output text)
sleep 10
aws redshift-data describe-statement --id "$STMT_ID" --region eu-west-2
```

**Common errors:**

| Error | Cause | Go to |
|---|---|---|
| `Spectrum Scan Error: incompatible Parquet schema` | Column type mismatch between Glue and Parquet | Step 3 |
| `column "X" does not exist` | View references removed/renamed column | Step 5 |
| `relation does not exist` | Wrong DB name or schema not created | Step 4 |
| `0 rows` with no error | Missing Glue partitions | Step 4 |
| `syntax error at or near "X"` | Reserved word not quoted (`"time"`, `"type"`) | Step 5 |

---

## Step 2 — Inspect Schema Registry to find type conflicts

```bash
# List all subjects
curl -s http://10.200.95.52:8081/subjects

# Latest schema for a table
curl -s http://10.200.95.52:8081/subjects/cdc.keep-me.<table>-value/versions/latest \
  | python3 -c "
import json, sys
data = json.load(sys.stdin)
schema = json.loads(data['schema'])
print(f'Version: {data[\"version\"]}')
for f in schema['fields']:
    t = f['type']
    if isinstance(t, list): t = [x for x in t if x != 'null']; t = t[0] if t else 'null'
    if isinstance(t, dict): t = t.get('logicalType') or t.get('type','?')
    print(f'  {f[\"name\"]}: {t}')
"
```

**Check for type changes across ALL versions** (finds incompatible columns):

```python
import urllib.request, json

BASE = "http://10.200.95.52:8081"
def get(p):
    with urllib.request.urlopen(BASE+p) as r: return json.loads(r.read())

def field_type(f):
    t = f['type']
    if isinstance(t, list): t = [x for x in t if x != 'null']; t = t[0] if t else 'null'
    if isinstance(t, dict): return t.get('logicalType') or t.get('type','?')
    return t

versions = get("/subjects/cdc.keep-me.<table>-value/versions")
field_history = {}
for v in versions:
    data = get(f"/subjects/cdc.keep-me.<table>-value/versions/{v}")
    schema = json.loads(data["schema"])
    for f in schema["fields"]:
        name = f["name"]; typ = field_type(f)
        if name not in field_history: field_history[name] = []
        last = field_history[name][-1][1] if field_history[name] else None
        if typ != last: field_history[name].append((v, typ))

print("Fields with type changes (potential incompatibilities):")
for field, history in sorted(field_history.items()):
    if len(history) > 1:
        print(f"  {field}: " + " -> ".join(f"v{v}:{t}" for v,t in history))
```

**Resolution rules:**

| Type conflict | Fix |
|---|---|
| `timestamp-millis` ↔ `string` | Remove column from Glue |
| `int` ↔ `string` | Remove column from Glue |
| `array` ↔ `string` | Use `SUPER` type in Redshift |
| Field not in latest schema | Safe to remove from Glue |

---

## Step 3 — Fix Glue table schema

> Glue is the single source of truth. Redshift external schema auto-syncs — **never ALTER the Redshift external table directly**.

```python
import boto3

glue = boto3.client('glue', region_name='eu-west-2')
GLUE_DB = 'cdc_keepme_production'
TABLE = '<table>'

table = glue.get_table(DatabaseName=GLUE_DB, Name=TABLE)['Table']
sd = table['StorageDescriptor']

# --- Add missing columns ---
existing = {c['Name'] for c in sd['Columns']}
to_add = [
    {'Name': 'new_col', 'Type': 'varchar(128)'},
]
for col in to_add:
    if col['Name'] not in existing:
        sd['Columns'].append(col)

# --- Fix a column type ---
for col in sd['Columns']:
    if col['Name'] == 'bad_col':
        col['Type'] = 'timestamp'  # fix type

# --- Remove conflicting columns ---
remove = {'conflicting_col1', 'conflicting_col2'}
sd['Columns'] = [c for c in sd['Columns'] if c['Name'] not in remove]

# --- Apply update ---
allowed = {'Name','Description','Owner','LastAccessTime','LastAnalyzedTime','Retention',
           'StorageDescriptor','PartitionKeys','ViewOriginalText','ViewExpandedText',
           'TableType','Parameters','TargetTable','ViewDefinition'}
glue.update_table(
    DatabaseName=GLUE_DB,
    TableInput={k: v for k, v in table.items() if k in allowed} | {'StorageDescriptor': sd}
)
print("Glue updated")
```

**Avro → Glue → Redshift type mapping:**

| Avro type | Glue type | Redshift type |
|---|---|---|
| `string` | `varchar(N)` | `VARCHAR(N)` |
| `boolean` | `boolean` | `BOOLEAN` |
| `long` | `bigint` | `BIGINT` |
| `int` | `int` | `INT` |
| `timestamp-millis` | `timestamp` | `TIMESTAMP` |
| `array<string>` | `string` (Glue) | `SUPER` (Redshift) |

---

## Step 4 — Check and register Glue partitions

After dropping/recreating a Glue table, partitions must be re-registered from S3:

```python
import boto3

s3 = boto3.client('s3', region_name='eu-west-2')
glue = boto3.client('glue', region_name='eu-west-2')
BUCKET = 'antares-data-warehouse-prod'
PREFIX = 'topics/cdc.keep-me.<table>/'
GLUE_DB = 'cdc_keepme_production'
TABLE = '<table>'

sd_template = glue.get_table(DatabaseName=GLUE_DB, Name=TABLE)['Table']['StorageDescriptor']
paginator = s3.get_paginator('list_objects_v2')

partitions = []
for yp in paginator.paginate(Bucket=BUCKET, Prefix=PREFIX, Delimiter='/').search('CommonPrefixes'):
    if not yp: continue
    year = yp['Prefix'].split('year=')[1].rstrip('/')
    for mp in paginator.paginate(Bucket=BUCKET, Prefix=yp['Prefix'], Delimiter='/').search('CommonPrefixes'):
        if not mp: continue
        month = mp['Prefix'].split('month=')[1].rstrip('/')
        for dp in paginator.paginate(Bucket=BUCKET, Prefix=mp['Prefix'], Delimiter='/').search('CommonPrefixes'):
            if not dp: continue
            day = dp['Prefix'].split('day=')[1].rstrip('/')
            for hp in paginator.paginate(Bucket=BUCKET, Prefix=dp['Prefix'], Delimiter='/').search('CommonPrefixes'):
                if not hp: continue
                hour = hp['Prefix'].split('hour=')[1].rstrip('/')
                partitions.append((year, month, day, hour, f"s3://{BUCKET}/{hp['Prefix'].rstrip('/')}"))

existing = set()
for page in glue.get_paginator('get_partitions').paginate(DatabaseName=GLUE_DB, TableName=TABLE):
    for p in page['Partitions']: existing.add(tuple(p['Values']))

to_add = [{'Values': [y,m,d,h], 'StorageDescriptor': {**sd_template, 'Location': loc}, 'Parameters': {}}
          for y,m,d,h,loc in partitions if (y,m,d,h) not in existing]

for i in range(0, len(to_add), 100):
    glue.batch_create_partition(DatabaseName=GLUE_DB, TableName=TABLE, PartitionInputList=to_add[i:i+100])
print(f"Added {len(to_add)} partitions ({len(existing)} already existed)")
```

---

## Step 5 — Rebuild the Redshift view

After any Glue schema change, recreate the view via Redshift Data API:

```bash
STMT_ID=$(aws redshift-data execute-statement \
  --region eu-west-2 \
  --workgroup-name dev \
  --database keep-me \
  --sql "
CREATE OR REPLACE VIEW cdc.<table> AS
SELECT
  _id         AS id,
  -- ... columns ...
  __op        AS cdc_op,
  dateadd(millisecond, __ts_ms, '1970-01-01'::timestamp) AS cdc_ts,
  year, month, day, hour
FROM cdc_raw.<table>
WHERE __op != 'd' WITH NO SCHEMA BINDING;
" \
  --query 'Id' --output text)
sleep 8
aws redshift-data describe-statement --id "$STMT_ID" --region eu-west-2 --query '[Status,Error]' --output text
```

**View rules:**
- Always use `WITH NO SCHEMA BINDING`
- Quote reserved words: `"time"`, `"type"`, `"date"`
- `created_date` / `updated_date` are already `TIMESTAMP` — do NOT wrap with `dateadd()`
- `__ts_ms` is `BIGINT` → use `dateadd(millisecond, __ts_ms, '1970-01-01'::timestamp)`
- `SUPER` columns (e.g. `interests`) pass through as-is

---

## Step 6 — Compare dev-provided schema JSON vs Glue

When a developer provides a schema JSON file:

```python
import boto3, json

with open('schema.json') as f:
    dev_schema = json.load(f)

def avro_type(field):
    t = field['type']
    if isinstance(t, list): t = [x for x in t if x != 'null']; t = t[0] if t else 'null'
    if isinstance(t, dict): return t.get('logicalType') or t.get('type','?')
    return t

dev_fields = {f['name']: avro_type(f) for f in dev_schema['fields']}

glue = boto3.client('glue', region_name='eu-west-2')
table = glue.get_table(DatabaseName='cdc_keepme_production', Name='<table>')['Table']
glue_cols = {c['Name']: c['Type'] for c in table['StorageDescriptor']['Columns']}

print("=== MISSING in Glue ===")
for name, typ in dev_fields.items():
    if name not in glue_cols:
        print(f"  + {name}: {typ}")

print("\n=== TYPE MISMATCHES ===")
for name, typ in dev_fields.items():
    if name in glue_cols:
        g = glue_cols[name]
        if 'timestamp' in str(typ) and 'timestamp' not in g:
            print(f"  ! {name}: dev={typ} glue={g}")
        if 'array' in str(typ) and 'super' not in g.lower():
            print(f"  ! {name}: array needs SUPER, glue={g}")

print("\n=== EXTRA in Glue (historical) ===")
for name in glue_cols:
    if name not in dev_fields and name not in ('__op','__ts_ms'):
        print(f"  - {name}: {glue_cols[name]}")
```

---

## Step 7 — Check Kafka Connect connector status

```bash
# List connector pods
kubectl get pods -n data-warehouse-prod | grep -E "source|sink"

# Check connector logs
kubectl logs -n data-warehouse-prod \
  deployment/prod-dw-connector-source-<table>-kafka-connect-source --tail=100

# Check connector config (what fields are captured)
kubectl get deployment prod-dw-connector-source-<table>-kafka-connect-source \
  -n data-warehouse-prod -o yaml | grep -A100 "connector.class"
```

---

---

## Step 9 — Reset a connector (full pipeline reset)

Use when a connector needs to be re-synced from scratch (schema changed, topic corrupted, S3 needs to be cleared).

### 9.1 — Scale down via GitOps

Edit `keepme-argo/prod/data-warehouse/kafka-connect-connectors/source-<name>/values.yaml`
and `keepme-argo/prod/data-warehouse/kafka-connect-sink-connectors/sink-<name>/values.yaml`:

```yaml
replicaCount: 0   # scale to zero

connector:
  enabled: true   # KEEP true — false deletes the Deployment via Helm gate
  ...
```

> The Helm chart wraps the entire Deployment in `{{- if .Values.connector.enabled }}`.
> Setting `enabled: false` deletes the Deployment. Use `replicaCount: 0` to just stop pods.

Commit, push, then **immediately trigger ArgoCD sync via kubectl** (do not wait for auto-sync):

```bash
cd keepme-argo
git add prod/data-warehouse/kafka-connect-connectors/source-<name>/values.yaml \
        prod/data-warehouse/kafka-connect-sink-connectors/sink-<name>/values.yaml
git commit -m "scale down <name> source/sink connectors"
git push

# Always trigger ArgoCD sync right after push
kubectl annotate application prod-dw-connector-source-<name> -n argocd argocd.argoproj.io/refresh=normal --overwrite
kubectl annotate application prod-dw-connector-sink-<name>   -n argocd argocd.argoproj.io/refresh=normal --overwrite

# Verify pods terminated / started
kubectl get pods -n data-warehouse-prod | grep <name>
```

### 9.2 — Delete ALL Kafka topics (CDC + DLQ + all internal connector topics)

```bash
# List all topics related to connector
kubectl exec kafka-controller-0 -n data-warehouse-prod -c kafka -- \
  env JMX_PORT="" KAFKA_JMX_OPTS="" \
  /opt/bitnami/kafka/bin/kafka-topics.sh \
  --bootstrap-server 10.200.170.138:9092 --list 2>/dev/null | grep <name>

# Delete CDC + DLQ topics
for topic in "cdc.keep-me.<collection>" "dlq-sink-<name>"; do
  kubectl exec kafka-controller-0 -n data-warehouse-prod -c kafka -- \
    env JMX_PORT="" KAFKA_JMX_OPTS="" \
    /opt/bitnami/kafka/bin/kafka-topics.sh \
    --bootstrap-server 10.200.170.138:9092 \
    --delete --topic "$topic" 2>/dev/null && echo "deleted $topic"
done

# Delete ALL internal connector topics (source + sink, including duplicate _sink-sink- pattern)
for topic in \
  "_connect-configs-prod-dw-connector-source-<name>" \
  "_connect-offsets-prod-dw-connector-source-<name>" \
  "_connect-status-prod-dw-connector-source-<name>" \
  "_connect-configs-sink-<name>" \
  "_connect-offsets-sink-<name>" \
  "_connect-status-sink-<name>" \
  "_connect-configs-sink-sink-<name>" \
  "_connect-offsets-sink-sink-<name>" \
  "_connect-status-sink-sink-<name>"; do
  kubectl exec kafka-controller-0 -n data-warehouse-prod -c kafka -- \
    env JMX_PORT="" KAFKA_JMX_OPTS="" \
    /opt/bitnami/kafka/bin/kafka-topics.sh \
    --bootstrap-server 10.200.170.138:9092 \
    --delete --topic "$topic" 2>/dev/null && echo "deleted $topic"
done
```

### 9.3 — Delete consumer groups (if any)

```bash
# List groups matching connector name
kubectl exec kafka-controller-0 -n data-warehouse-prod -c kafka -- \
  env JMX_PORT="" KAFKA_JMX_OPTS="" \
  /opt/bitnami/kafka/bin/kafka-consumer-groups.sh \
  --bootstrap-server 10.200.170.138:9092 --list 2>/dev/null | grep -i <name>

# Delete group (connector name format: connect-S3-Sink-<Name>)
kubectl exec kafka-controller-0 -n data-warehouse-prod -c kafka -- \
  env JMX_PORT="" KAFKA_JMX_OPTS="" \
  /opt/bitnami/kafka/bin/kafka-consumer-groups.sh \
  --bootstrap-server 10.200.170.138:9092 \
  --delete --group "connect-<ConnectorName>" 2>/dev/null && echo "deleted group"
```

### 9.4 — Purge Schema Registry subject

```bash
# Soft delete (marks versions as deleted)
curl -s -X DELETE "http://10.200.95.52:8081/subjects/cdc.keep-me.<collection>-value" > /dev/null
# Hard delete (permanent — version counter does not reset, that is normal)
curl -s -X DELETE "http://10.200.95.52:8081/subjects/cdc.keep-me.<collection>-value?permanent=true" > /dev/null
echo "schema registry purged"
```

### 9.5 — Clear S3 (use sub-agent to run in background)

Launch a background agent so S3 delete runs in parallel with other work:

```
Agent prompt: "Run: aws s3 rm s3://antares-data-warehouse-prod/topics/cdc.keep-me.<collection>/ --recursive
Report how many files were deleted."
```

Or run directly:
```bash
aws s3 rm s3://antares-data-warehouse-prod/topics/cdc.keep-me.<collection>/ --recursive
```

### 9.6 — Scale up source only, then ask before sink

- Set `replicaCount: 1` in **source** `values.yaml` only
- Commit, push, trigger ArgoCD sync
- Wait for source pod to be Running
- Check Schema Registry for the new schema version
- Update Glue table to match registered schema
- **Ask user before scaling up sink**

### 9.7 — Scale up sink (after user confirms)

- Set `replicaCount: 1` in **sink** `values.yaml`
- Commit, push, trigger ArgoCD sync
- Register new S3 partitions in Glue
- Create/update `cdc.<table>` view
- Run `SELECT * FROM "keep-me"."cdc"."<table>" LIMIT 5` to verify
