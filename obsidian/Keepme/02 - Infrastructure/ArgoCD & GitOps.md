# ArgoCD & GitOps (keepme-argo)

## Repository Structure

```
keepme-argo/
├── dev/                   # Development deployments
├── staging/               # Staging deployments
├── prod/
│   ├── data-warehouse/    # Kafka, Glue, Redshift
│   └── [other services]/
├── infra/                 # Infrastructure apps
└── root.yaml              # Root ArgoCD Application
```

## App Naming Convention
- `prod-{service-name}`
- `dev-{service-name}`
- `prod-dw-connector-source-{collection}`
- `prod-dw-connector-sink-{collection}`

## After Every Git Push

```bash
kubectl annotate application <app-name> -n argocd \
  argocd.argoproj.io/refresh=normal --overwrite
```

Always trigger for both source and sink when changing connector config.

## Scaling Connectors via GitOps

```yaml
# values.yaml
replicaCount: 1   # scale up
replicaCount: 0   # scale down (keep enabled: true)

connector:
  enabled: true   # NEVER set false unless deleting deployment
```

## ArgoCD ignoreDifferences (Kafka)

Prevents rolling restarts from Kafka StatefulSet annotation drift:

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

## SSH Key for Git Push

```bash
GIT_SSH_COMMAND="ssh -i /Users/hoainn/Documents/Project/Keepme/keepme_minhlt" git push
```

## Related Notes
- [[../03 - Data Warehouse/Kafka Connectors]]
- [[../05 - Operations/Connector Reset Procedure]]
