# Microservices (keepme-services)

## Stack
- Fastify 5, TypeScript
- KafkaJS (Kafka client)
- Awilix (dependency injection)
- Knex.js (database/migrations)
- Vitest (testing)
- OpenTelemetry + Prometheus (observability)
- Pino (logging)

## Services

### km-webhook-service
- Receives inbound webhooks
- Produces events to Kafka topic: `webhook-{clientName}`
- REST API with Swagger docs

### km-webhook-consumer-service
- Consumes webhook events from Kafka
- Processes and forwards to external systems
- DLQ error handling

## Kafka Producer Pattern

```typescript
const kafkaMessage = {
  value: typeof message === 'string' ? message : JSON.stringify(message),
  ...(key && { key }),
};
await producer.send({ topic, messages: [kafkaMessage] });
```

- Uses `Partitioners.LegacyPartitioner`
- Supports SASL authentication (SASL_PLAINTEXT)
- Auto-creates topics if not exist (7-day retention, 1 partition, RF=3)

## Commands

```bash
npm run test     # Vitest
npm run build    # TypeScript compile
npm run start    # Run compiled service
```
