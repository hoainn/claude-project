# Adding or Changing a Field

Use this note whenever a field is added, renamed, or removed from a MongoDB collection.
Every new field must travel the full pipeline: connector → Kafka → S3 → Glue → Redshift view → datashare.

---

## What NOT to Do

- **Do NOT exclude a field that is in the developer schema JSON.**
  Fields in `keepme-infra/Kafka_schema/{collection}.json` must flow through. Excluding them breaks downstream consumers.

- **Do NOT use `json_serialize()` in a Redshift view.**
  The `keep-me` datashare does not support SUPER type. Use `NULL::varchar` for array/nested columns.

- **Do NOT DROP VIEW when only adding a column.**
  `CREATE OR REPLACE VIEW` is safe. Only DROP when changing an existing column's output type.

- **Do NOT ignore existing Parquet files when renaming or retyping.**
  The sink never overwrites Parquet files. Re-snapshotting appends duplicates — a full reset is required.

- **Do NOT rewrite a connector offset if the stored value contains `"initsync": true`.**
  The connector is mid-snapshot. Rewriting triggers a full re-snapshot and duplicate S3 data.

- **Do NOT run Kafka CLI commands without disabling JMX.**
  Always prefix: `env JMX_PORT="" KAFKA_JMX_OPTS=""`

- **Do NOT update `dev_external_tables.sql`.**

---

## When is a Full Reset Required?

| Situation | Action |
|-----------|--------|
| Adding a brand-new field (never ingested) | No reset needed |
| Changing cast type on a field not yet in S3 | No reset needed |
| Removing a field from `excludeFields` | No reset needed |
| Field already ingested with the wrong Avro type | **Full reset** |
| Renaming a field that already has Parquet data in S3 | **Full reset** |
| Schema Registry subject has a type conflict | **Full reset** |

**Full reset order:** scale down source + sink → delete Kafka topics + DLQ + internal topics → purge Schema Registry subject → clear S3 prefix → scale up source → wait for confirmation before scaling up sink.

See [[../05 - Operations/Connector Reset Procedure]].

---

## Step 1 — Check the Developer Schema JSON

```
keepme-infra/Kafka_schema/<collection>.json
```

- Field **is listed** → must flow through. Do NOT add to `excludeFields`.
- Field **is not listed** → add to `excludeFields` in the source connector.

---

## Step 2 — Identify Field Type and Required SMT

| Field type | Why it needs special handling | What to do |
|------------|-------------------------------|------------|
| Plain string / number | No issue | Nothing extra — add to Glue + view |
| BSON Date / timestamp | Debezium emits epoch long; Avro type conflicts | `castDates.spec`: `field:string` |
| Nullable boolean | Type inference inconsistent when field is absent | `castDates.spec`: `field:boolean` |
| ObjectId or mixed int/string | Different documents emit different types | `castDates.spec`: `field:string` |
| Array `[...]` | Must stay `array<string>`; cannot be SUPER | Glue: `array<string>`; view: `NULL::varchar` |
| Nested object `{a:{b:v}}` | Parquet sink can't serialise struct | Enable `flatten.struct=true`; Glue column becomes `a_b` |
| Key with ` - ` (space-dash-space) | Avro sanitises to `___` | `renameFields` SMT: `key___sub` → `key_sub` |
| Field not in schema | Should never reach S3 | `excludeFields`: `keep-me.{col}.{field}` |

---

## Step 3 — Update the Source Connector

File: `keepme-argo/prod/data-warehouse/kafka-connect-connectors/source-<collection>/values.yaml`

```yaml
# castDates — add new field alongside existing ones
transforms.castDates.spec: "existing_field:string,new_field:string"

# excludeFields — top-level
excludeFields: "keep-me.<collection>.<field>"

# excludeFields — nested
excludeFields: "keep-me.<collection>.<parent>.<child>"
```

Then: commit → push → ArgoCD sync:
```bash
kubectl annotate application source-<collection> -n argocd \
  argocd.argoproj.io/refresh=normal --overwrite
```

---

## Step 4 — Update Glue (`glue.tf`)

File: `keepme-infra/infrastructure/eu-west-2/production/terraform/data-warehouse/glue.tf`

Add inside the collection's `locals.table_columns` block:

```hcl
{ name = "new_field",    type = "varchar(256)" }  # plain string
{ name = "new_date",     type = "varchar(64)"  }  # date cast to string
{ name = "new_flag",     type = "boolean"      }  # boolean
{ name = "new_arr",      type = "array<string>"}  # array
{ name = "parent_child", type = "varchar(256)" }  # flattened nested
```

Then:
```bash
terraform apply
```

After applying, register the current partition:
```bash
aws glue batch-create-partition --database-name cdc_keepme_production \
  --table-name <collection> --partition-input-list '[...]'
```

---

## Step 5 — Update the Redshift View

Run in the **`keep-me` database** (not `dev`):

```sql
CREATE OR REPLACE VIEW cdc.<collection> AS
SELECT
    -- existing columns ...,

    new_field,                                        -- plain string

    dateadd(ms, new_date::bigint, '1970-01-01')
        AS new_date,                                  -- epoch ms → timestamp

    new_flag::boolean AS new_flag,                    -- boolean cast

    NULL::varchar AS new_arr,                         -- array: datashare limit

    NULL::varchar AS parent_child,                    -- nested: datashare limit

    __op,
    __ts_ms
FROM cdc_raw.<collection>
WHERE __op != 'd'
WITH NO SCHEMA BINDING;
```

---

## Step 6 — Verify End-to-End

1. **Connector healthy** — check pod logs for `SchemaException` or `DataException` errors
2. **Parquet file written** — check S3 path for a new file after next flush (~60 s)
3. **Glue column present** — query `cdc_raw.<collection>` in Redshift Serverless
4. **View returns data** — `SELECT * FROM "keep-me"."cdc"."<collection>" LIMIT 5;`

---

## Quick Checklist

- [ ] Check `Kafka_schema/<collection>.json` — is the field listed?
- [ ] Decide field type (string / date / boolean / ObjectId / array / nested / exclude)
- [ ] Update `castDates.spec` or `excludeFields` in source connector `values.yaml`
- [ ] Commit, push, ArgoCD sync source connector
- [ ] Add column to `glue.tf` with correct type
- [ ] Run `terraform apply`
- [ ] Register partition via `batch_create_partition`
- [ ] Run `CREATE OR REPLACE VIEW` in `keep-me` DB
- [ ] Verify with `SELECT * FROM "keep-me"."cdc"."<collection>" LIMIT 5`

---

## Related Notes
- [[SMT Transforms]]
- [[Glue & Redshift]]
- [[Kafka Connectors]]
- [[../05 - Operations/Connector Reset Procedure]]
