# Common Workflows

## Deploy a connector config change

1. Edit `values.yaml` in `keepme-argo/prod/data-warehouse/kafka-connect-connectors/source-<name>/`
2. Commit and push
3. Trigger ArgoCD sync:
```bash
kubectl annotate application prod-dw-connector-source-<name> -n argocd \
  argocd.argoproj.io/refresh=normal --overwrite
```

## Scale connector up/down

```yaml
# values.yaml
replicaCount: 1   # up
replicaCount: 0   # down
```
Commit + push + ArgoCD sync.

## Add a new field to Glue table

1. Update the Glue table schema (add column)
2. Register new partition via `batch_create_partition`
3. Wait for user before querying Redshift

## Update a Redshift view

```sql
DROP VIEW cdc.<table>;
CREATE VIEW cdc.<table> AS
SELECT ...
FROM cdc_raw.<table>
WHERE __op != 'd'
WITH NO SCHEMA BINDING;
```

Always update in `keep-me` database (datashare), not `dev`.

## Check Kafka topics

```bash
kubectl exec kafka-controller-0 -n data-warehouse-prod -c kafka -- \
  env JMX_PORT="" KAFKA_JMX_OPTS="" \
  /opt/bitnami/kafka/bin/kafka-topics.sh \
  --bootstrap-server 10.200.170.138:9092 --list
```

## Check Schema Registry subjects

```bash
curl http://10.200.95.52:8081/subjects
curl http://10.200.95.52:8081/subjects/cdc.keep-me.<collection>-value/versions/latest
```

## Check DLQ errors

Inspect dead-letter queue topic: `dlq-sink-<collection>`

## Check connector status

```bash
curl http://<kafka-connect-service>/connectors/<connector-name>/status
```

## Run backend tests

```bash
docker-compose exec api php artisan config:clear
docker-compose exec api php artisan test
```

## Restart queue workers

```bash
docker-compose exec api php artisan queue:restart
```
