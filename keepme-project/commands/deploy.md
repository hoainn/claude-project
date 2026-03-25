# Deploy to Environment

Deploy a service to the specified environment (dev/staging/prod) via ArgoCD GitOps.

## Usage
`/project:deploy <service> <environment>`

## Steps

1. Check current branch and ensure changes are committed
2. Identify the Helm chart in `charts/<service>/` or `keepme-infra/charts/`
3. Update the image tag or values in the appropriate ArgoCD app under `keepme-argo/<environment>/`
4. Commit and push — ArgoCD auto-syncs (15 retry attempts with exponential backoff)
5. Monitor rollout: `kubectl rollout status deployment/<service> -n <namespace>`

## Environments
- `dev` — `keepme-argo/dev/`
- `staging` — `keepme-argo/staging/`
- `prod` — `keepme-argo/prod/`
