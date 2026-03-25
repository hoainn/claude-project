---
description: Scale a Kafka Connect connector up or down via GitOps
argument-hint: <connector-name> <up|down>
---

## Connector: $ARGUMENTS

## Current pod status
!`kubectl get pods -n data-warehouse-prod | grep "$(echo "$ARGUMENTS" | awk '{print $1}')" 2>/dev/null`

## Current source values.yaml
!`cat keepme-argo/prod/data-warehouse/kafka-connect-connectors/source-$(echo "$ARGUMENTS" | awk '{print $1}')/values.yaml 2>/dev/null`

Scale the connector. Steps:
1. Set `replicaCount: 0` (down) or `replicaCount: 1` (up) in values.yaml — never set `enabled: false`
2. Commit and push to keepme-argo
3. Trigger ArgoCD sync for both source and sink apps
4. Watch pod status until change takes effect

Notes:
- Scale down **both** source and sink before any connector reset
- Scale up **source only** after reset — always ask user before scaling sink
