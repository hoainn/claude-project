# Kubernetes

## Cluster
- Provider: AWS EKS
- Cluster name: `keepme-eu-west-2`
- Primary region: `eu-west-2` (London)
- Secondary: `us-east-1`

## Namespaces

| Namespace | Purpose |
|-----------|---------|
| `data-warehouse-prod` | Kafka, Kafka Connect, Schema Registry, Glue registrar |
| `keepme-database-dump` | DB utilities |
| `default` | Core services (API, frontend, microservices) |
| `argocd` | ArgoCD control plane |
| `monitoring` | Prometheus, Grafana |

## Network Addons
- AWS VPC CNI
- CoreDNS
- kube-proxy
- AWS Load Balancer Controller
- Cloudflare Tunnel

## Key Addresses (data-warehouse-prod)
- Kafka bootstrap: `10.200.170.138:9092`
- Schema Registry: `http://10.200.95.52:8081`

## Kafka Admin Commands

> Always disable JMX or commands will fail

```bash
kubectl exec kafka-controller-0 -n data-warehouse-prod -c kafka -- \
  env JMX_PORT="" KAFKA_JMX_OPTS="" \
  /opt/bitnami/kafka/bin/kafka-topics.sh \
  --bootstrap-server 10.200.170.138:9092 --list
```

## Helm Charts (keepme-infra/charts)

| Chart | Service |
|-------|---------|
| `keepme-react-v4` | Frontend |
| `keepme-v4` | Backend API |
| `keepme-legacy` | Legacy backend |
| `keepme-engage` | Marketing engagement |
| `keepme-billing` | Billing service |
| `keepme-api-legacy` | Legacy API |

Each chart includes: Deployment, Service, ConfigMap, Secret, Probes, Resource limits, HPA.

## Secret Management
- Tool: SOPS (Secrets OPerationS)
- KMS key: `arn:aws:kms:eu-west-2:760505282981:key/...`
- Encrypted files: `encrypted-secrets/` directory
- Decrypt: `sops --decrypt --kms $KEYS file.enc.yaml > file.yaml`
