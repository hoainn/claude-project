# Infrastructure Rules

## Kubernetes
- Cluster: AWS EKS (eu-west-2, us-east-1)
- Deployments managed by ArgoCD (auto-healing, 15 retries with exponential backoff)
- Namespaces of note: `data-warehouse-prod`, `keepme-database-dump`

## Data Pipeline (Kafka → S3 → Glue → Redshift)
- Schema Registry: `http://10.200.95.52:8081` (in-cluster, ns `data-warehouse-prod`)
- S3 prod bucket: `antares-data-warehouse-prod`
- S3 dev bucket: `antares-data-warehouse-dev`
- Glue prod DB: `cdc_keepme_production` → Redshift external schema: `cdc_raw`
- Glue dev DB: `cdc_keepme_development`
- Redshift Serverless: workgroup `dev`, region `eu-west-2`, namespace `antares-data-warehouse`
- IAM role for Redshift: `arn:aws:iam::760505282981:role/service-role/AmazonRedshift-CommandsAccessRole-20260321T132138`
- IAM role for Kafka sink (S3 write): `arn:aws:iam::760505282981:role/kafka-connector-sink`
- Developer schema definitions: `/Users/hoainn/Documents/Project/KeepMe/Kafka_schema/<collection>.json`

## Kafka Admin (in K8s)
- **Namespace:** `data-warehouse-prod`
- **Pod:** `kafka-controller-0`, container `kafka`
- **Scripts:** `/opt/bitnami/kafka/bin/`
- **Bootstrap server (confirmed):** `10.200.170.138:9092` (ClusterIP of `kafka` service)
- **CRITICAL:** Must disable JMX or commands fail — always pass `env JMX_PORT="" KAFKA_JMX_OPTS=""`

```bash
# Template for all Kafka admin commands
kubectl exec kafka-controller-0 -n data-warehouse-prod -c kafka -- \
  env JMX_PORT="" KAFKA_JMX_OPTS="" \
  /opt/bitnami/kafka/bin/kafka-topics.sh \
  --bootstrap-server 10.200.170.138:9092 \
  [--list | --delete --topic <topic> | ...]  2>/dev/null

# Consumer groups
kubectl exec kafka-controller-0 -n data-warehouse-prod -c kafka -- \
  env JMX_PORT="" KAFKA_JMX_OPTS="" \
  /opt/bitnami/kafka/bin/kafka-consumer-groups.sh \
  --bootstrap-server 10.200.170.138:9092 \
  [--list | --delete --group <group>]  2>/dev/null
```

## GitOps / ArgoCD
- Scale down a connector: set `replicaCount: 0` in `values.yaml` (keep `enabled: true` — `enabled: false` deletes the Deployment entirely via Helm `{{- if .Values.connector.enabled }}` gate)
- Scale up: set `replicaCount: 1`
- **After every `git push` to keepme-argo, always trigger ArgoCD sync immediately via kubectl:**

```bash
# Trigger sync for one app
kubectl annotate application <app-name> -n argocd argocd.argoproj.io/refresh=normal --overwrite

# Trigger sync for both source and sink of a connector
kubectl annotate application prod-dw-connector-source-<name> -n argocd argocd.argoproj.io/refresh=normal --overwrite
kubectl annotate application prod-dw-connector-sink-<name>   -n argocd argocd.argoproj.io/refresh=normal --overwrite

# Watch pod status after sync
kubectl get pods -n data-warehouse-prod | grep <name>
```

## ArgoCD ignoreDifferences — Kafka rolling restart fix
The `prod-data-warehouse-kafka` app with `selfHeal: true` triggers rolling restarts every sync because
Helm recomputes the `kafka-user-passwords` Secret checksum. Fix via `ignoreDifferences` in `app.yaml`:

```yaml
ignoreDifferences:
  - group: apps
    kind: StatefulSet
    name: kafka-controller
    jsonPointers:
      - /spec/template/metadata/annotations/checksum~1passwords-secret
  - group: ""
    kind: Secret
    name: kafka-user-passwords
    jsonPointers:
      - /data
```

Applied to: `prod/data-warehouse/kafka/app.yaml`

## Raw Kubernetes manifest apps (ArgoCD)
For CronJobs/raw manifests, use single-source app with `path:` and `kustomization.yaml`:
```yaml
spec:
  source:
    repoURL: https://github.com/keepme-ai/keepme-argo.git
    targetRevision: main
    path: prod/data-warehouse/<dir>
```

## Terraform
- IaC in `keepme-infra/infrastructure/` — modules: vpc, eks, iam, eks-addons
- Secrets managed with SOPS + AWS KMS — never commit plaintext secrets

## CI/CD
- GitHub Actions workflows: `build.yml` (prod), `build_dev.yml` (dev), `build_staging.yml` (staging)
- Three environments: dev → staging → prod
