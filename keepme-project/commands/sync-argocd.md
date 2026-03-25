# Sync ArgoCD

Trigger an ArgoCD refresh for one or more applications after a git push.

## Usage
`/project:sync-argocd <app-name>`

Example: `/project:sync-argocd prod-dw-connector-source-redis-jobs`

## Steps

1. Trigger refresh:
   ```bash
   kubectl annotate application <app-name> -n argocd argocd.argoproj.io/refresh=normal --overwrite
   ```
2. Watch pod status:
   ```bash
   kubectl get pods -n data-warehouse-prod | grep <name>
   ```

## Common app names
- `prod-dw-connector-source-<name>`
- `prod-dw-connector-sink-<name>`
- `prod-data-warehouse-kafka`

## Notes
- Always run this immediately after every `git push` to keepme-argo
- Do not rely on ArgoCD auto-sync alone — trigger manually to avoid delays
