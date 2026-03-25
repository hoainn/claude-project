---
description: Full reset of a Kafka Connect connector — wipes Kafka, Schema Registry, S3, restarts source
argument-hint: <connector-name>
---

## Connector reset: $ARGUMENTS

## Current pod status
!`kubectl get pods -n data-warehouse-prod | grep "$ARGUMENTS" 2>/dev/null`

## Existing Kafka topics
!`kubectl exec kafka-controller-0 -n data-warehouse-prod -c kafka -- env JMX_PORT="" KAFKA_JMX_OPTS="" /opt/bitnami/kafka/bin/kafka-topics.sh --bootstrap-server 10.200.170.138:9092 --list 2>/dev/null | grep -i "$ARGUMENTS"`

## Consumer groups
!`kubectl exec kafka-controller-0 -n data-warehouse-prod -c kafka -- env JMX_PORT="" KAFKA_JMX_OPTS="" /opt/bitnami/kafka/bin/kafka-consumer-groups.sh --bootstrap-server 10.200.170.138:9092 --list 2>/dev/null | grep -i "$ARGUMENTS"`

Full reset for `$ARGUMENTS`. Follow this order exactly:

1. Scale down **both** source and sink (`replicaCount: 0`), commit + push + ArgoCD sync, wait for pods to terminate
2. Delete ALL Kafka topics: CDC (`cdc.keep-me.$ARGUMENTS`), DLQ (`dlq-sink-$ARGUMENTS`), all internal `_connect-*` topics for source and sink
3. Delete consumer groups
4. Purge Schema Registry subject (soft delete then `?permanent=true`)
5. Clear S3: `aws s3 rm s3://antares-data-warehouse-prod/topics/cdc.keep-me.$ARGUMENTS/ --recursive`
6. Scale up **source only**, commit + push + ArgoCD sync
7. **Stop — ask user before scaling up sink**
