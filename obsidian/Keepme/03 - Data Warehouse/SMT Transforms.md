# SMT (Single Message Transforms)

All source connectors use the `unwrap` transform (`ExtractNewDocumentState`) as a base. Additional SMTs are applied per collection.

## 1. `unwrap` — ExtractNewDocumentState (all connectors)

Strips Debezium envelope, promotes actual document fields, adds `op` and `ts_ms`.

```yaml
transforms.unwrap.type: "io.debezium.connector.mongodb.transforms.ExtractNewDocumentState"
transforms.unwrap.delete.handling.mode: "rewrite"
transforms.unwrap.drop.tombstones: "false"
transforms.unwrap.array.encoding: "array"
transforms.unwrap.add.fields: "op,ts_ms"
```

---

## 2. `flatten.struct` — on `unwrap`

**Used by**: `source-leadmappings`, `source-visitors`

Needed when MongoDB documents contain **nested sub-documents (objects)**. Promotes `{ parent: { child: "..." } }` → `parent_child`.

```yaml
transforms.unwrap.flatten.struct: "true"
transforms.unwrap.flatten.struct.delimiter: "_"
```

### When to use
Add `flatten.struct` whenever a collection has nested object fields that must become flat columns in Redshift.

**leadmappings** — `leadMappings` is a nested object:
- `leadMappings.organic`
- `leadMappings.paid - google`
- `leadMappings.paid - meta`
- `leadMappings.paid - other`

**visitors** — `details` is a nested object:
- `details.first_name`, `details.last_name`, `details.name`
- `details.email_address`, `details.phone_number`
- `details.source_group`, `details.source_name`
- `details.type`, `details.language_code`

---

## 3. `renameFields` — ReplaceField$Value

**Used by**: `source-leadmappings` only

Fixes field names mangled by Avro sanitization. The ` - ` in `paid - google` becomes `___` after sanitization.

```yaml
transforms.renameFields.type: "org.apache.kafka.connect.transforms.ReplaceField$Value"
transforms.renameFields.renames: "leadMappings_paid___google:leadMappings_paid_google,leadMappings_paid___meta:leadMappings_paid_meta,leadMappings_paid___other:leadMappings_paid_other"
```

### When to use
Any field key containing special characters (spaces, dashes, dots) that survive Avro sanitization as `___`.

---

## 4. `castDates` — Cast$Value

**Used by**: `source-visitors`, `source-visitor-leads`, `source-redis-jobs`

Debezium infers BSON dates as epoch longs and ObjectIds as wrong types. Cast to string/boolean for correct Parquet output.

```yaml
transforms.castDates.type: "org.apache.kafka.connect.transforms.Cast$Value"
```

| Connector | Fields |
|-----------|--------|
| `source-visitors` | `created_date`, `updated_date`, `lead_status_updated_date`, `person_id` → `string` |
| `source-visitor-leads` | `start_date`, `end_date`, `confirmation_sent_date` → `string`; `confirm_book`, `cancel_book`, `use_automation`, `email_opted_out`, `sms_opted_out` → `boolean` |
| `source-redis-jobs` | `scheduledDate`, `createdAt`, `updatedAt` → `string` |

### When to use

| Field type | Problem | Fix |
|------------|---------|-----|
| Date/timestamp | Debezium emits epoch long | Cast → `string` |
| Boolean (nullable) | Type inference inconsistency | Explicit Cast → `boolean` |
| ObjectId that can also be int | Schema conflicts | Cast → `string` |
| Nested object | S3 sink can't serialize struct | `flatten.struct` |
| Key with ` - ` or spaces | Avro sanitizes to `___` | `renameFields` |

---

## Developer Checklist

When adding a new field to a CDC collection, flag it if it is:
- [ ] A **nested object** → needs `flatten.struct`
- [ ] A key with **special characters** (dash, space) → needs `renameFields`
- [ ] A **date/timestamp** field → needs `castDates` → `string`
- [ ] A **nullable boolean** → needs `castDates` → `boolean`

Do not add fields without flagging — wrong types silently break the sink or write bad data.
