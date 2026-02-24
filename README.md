# Fraud Detection

Real-time fraud detection pipeline that scores card transactions using Kafka Streams, behavioral embeddings, and rule-based guardrails — without training a custom model.

## Tech Stack

- Java 21
- Spring Boot 4
- Kafka Streams
- MongoDB *(coming soon)*
- Voyage AI *(embedding generation)*

## Kafka Topics

| Topic | Description |
|-------|-------------|
| `transactions` | Receives all incoming transactions |
 | `transactions-suspicious` | Blocked transactions (fraud detected by rules) |
| `transactions-to-score` | Approved transactions (ready for embedding scoring) |
| `transactions-dlq` | Dead Letter Queue for malformed messages (e.g., invalid JSON) |

## How It Works

```
                                       ┌─────────────────────────┐
                                       │ transactions-suspicious │
                                       │       (FraudAlert)      │
                                       └────────────▲────────────┘
                                                    │ YES
                                                    │
┌─────────────┐    ┌──────────────┐    ┌───────────┴───────────┐
│ Transaction │───▶│ transactions │───▶│     Kafka Streams     │
└─────────────┘    └──────────────┘    │     (Fraud Rules)     │
                                       │                       │
                                       │   hasFraudAlert()?    │
                                       └───────────┬───────────┘
                                                   │ NO
                                       ┌───────────▼───────────┐
                                       │ transactions-to-score  │
                                       │     (Transaction)      │
                                       └───────────────────────┘
```

1. A transaction is sent via REST API to `transactions` topic
2. Kafka Streams processes and groups by card number
3. Rules evaluate: Impossible Travel, Velocity Check
4. **Blocked** → `transactions-suspicious` (fraud detected)
5. **Approved** → `transactions-to-score` (ready for scoring)
6. Malformed messages → `transactions-dlq`

## Fraud Rules

| Rule | Description | Example |
|------|-------------|---------|
| `IMPOSSIBLE_TRAVEL` | Same card used in distant locations in short time | Sao Paulo → New York in 5 minutes |
| `VELOCITY_CHECK` | Too many transactions in a time window | 4+ transactions in 1 minute |

## Running

### 1. Start Kafka

```bash
docker run -d --name kafka -p 9092:9092 apache/kafka:latest
```

### 2. Create Topics

```bash
bin/kafka-topics.sh --bootstrap-server localhost:9092 --create --topic transactions --partitions 1

bin/kafka-topics.sh --bootstrap-server localhost:9092 --create --topic transactions-suspicious --partitions 1

bin/kafka-topics.sh --bootstrap-server localhost:9092 --create --topic transactions-to-score --partitions 1
```

### 3. Run the Application

```bash
./mvnw spring-boot:run
```

### 4. Send a Transaction

```bash
curl -X POST http://localhost:8081/api/transactions \
  -H "Content-Type: application/json" \
  -d '{
    "transactionId": "TXN-001",
    "userId": "USR-123",
    "merchant": "Starbucks",
    "city": "Sao Paulo",
    "transactionAmount": 25.50,
    "transactionTime": "2025-01-15T10:30:00Z",
    "cardNumber": "4444-5555-6666-7777",
    "latitude": -23.5489,
    "longitude": -46.6388
  }'
```

### 5. Watch Suspicious Transactions

```bash
bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic transactions-suspicious --from-beginning
```

### Auto-generate Transactions

Enable the generator to publish one transaction per second:

```bash
mvn spring-boot:run -Dspring-boot.run.arguments="--app.generator.enabled=true"
```

You can override the interval with `--app.generator.interval-ms=1000`.

## Testing Fraud Detection

**Impossible Travel:** Send two transactions with the same card from distant locations in quick succession (e.g., Sao Paulo → New York).

**Velocity Check:** Send 4+ transactions with the same card within 1 minute.

See `src/main/resources/http/fraud-detection.http` for example requests.
