# Scale Connector

Scale a Kafka Connect connector up or down via ArgoCD GitOps.

## Usage
`/project:scale-connector <connector-name> <up|down>`

Example: `/project:scale-connector redis-jobs down`

## Steps

1. Locate the connector values file in `keepme-argo/prod/data-warehouse/<connector-name>/`
2. Set `replicaCount: 0` (down) or `replicaCount: 1` (up) — never set `enabled: false`
3. Commit and push to keepme-argo
4. Trigger ArgoCD sync:
   ```bash
   kubectl annotate application prod-dw-connector-source-<name> -n argocd argocd.argoproj.io/refresh=normal --overwrite
   kubectl annotate application prod-dw-connector-sink-<name> -n argocd argocd.argoproj.io/refresh=normal --overwrite
   ```
5. Watch pod status: `kubectl get pods -n data-warehouse-prod | grep <name>`

## Notes
- Scale down **both** source and sink before any connector reset
- Scale up **source only** after reset — ask user before scaling sink
- `enabled: false` deletes the Deployment entirely — always use `replicaCount: 0` instead
