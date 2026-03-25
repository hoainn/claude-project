# Infrastructure

## GitOps (ArgoCD)

- All deployments via ArgoCD — never apply manifests directly to clusters
- Application definitions in `redactable-argo-apps/` — each has `app.yaml` + `values.yaml`
- To deploy: edit `values.yaml`, commit, push — ArgoCD syncs automatically
- Clusters: `hub-cluster` (ArgoCD + monitoring), `uat-cluster`, `prod-cluster`

## Terraform (redactable-enterprise-infra)

- Separate Terraform state per environment: `dev/terraform-infra`, `prod/terraform-infra`
- S3 backend for state — always `make init` before plan/apply
- Commands:
  ```bash
  ENV=prod make init    # Initialize S3 backend
  ENV=prod make plan    # Preview changes
  ENV=prod make apply   # Apply changes
  ```
- Never apply prod without reviewing plan output first

## Helm Charts (redactable-charts)

- Wazuh security monitoring charts
- Charts follow standard Helm structure — `Chart.yaml`, `values.yaml`, `templates/`

## AWS Services

- S3 for document storage
- AWS Comprehend for entity detection
- ClamAV for virus scanning
- RabbitMQ for queue management
- Redis for caching and SignalR backplane
- MSSQL (AWS RDS) as primary database

## Kubernetes

- EKS clusters managed via Terraform
- Wazuh security monitoring deployed via Helm
- Environments: dev, uat, prod (separate clusters)

## Local Development Dependencies

```bash
# Start MSSQL + Redis
docker-compose -f docker-compose.services.yml up
```
