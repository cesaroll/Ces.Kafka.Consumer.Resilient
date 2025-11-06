# Kafka Setup Guide

## 🚀 Quick Start

The docker-compose.yml is configured to automatically create all required topics when you start the services.

### Start Everything

```bash
docker-compose up -d
```

This will:
1. ✅ Start Kafka broker (KRaft mode - no Zookeeper!)
2. ✅ Start Schema Registry
3. ✅ **Automatically create all topics** (via init-kafka service)
4. ✅ Start Kafka UI

### Topics Created

The following topics are automatically created:

| Topic | Purpose | Partitions |
|-------|---------|------------|
| `orders` | Main topic for incoming orders | 3 |
| `orders.retry.1` | First retry attempt | 3 |
| `orders.retry.2` | Second retry attempt | 3 |
| `orders.retry.3` | Third retry attempt | 3 |
| `orders.error` | Dead letter queue for failed messages | 3 |

## 🔍 Verify Setup

### Check if topics were created

```bash
# View init-kafka logs
docker logs init-kafka

# List topics directly
docker exec kafka kafka-topics --bootstrap-server localhost:9092 --list

# Describe a specific topic
docker exec kafka kafka-topics --bootstrap-server localhost:9092 --describe --topic orders
```

### Access Kafka UI

Open your browser: **http://localhost:8080**

You'll see:
- All topics
- Messages in each topic
- Consumer groups
- Schema Registry (for Avro)

## 📝 Produce Test Messages

### Using Console Producer

```bash
# Start producer
docker exec -it kafka kafka-console-producer \
  --bootstrap-server localhost:9092 \
  --topic orders

# Then type messages (one per line):
{"OrderId":"001","CustomerId":"C123","Amount":99.99,"OrderDate":"2024-01-15T10:30:00Z","Status":"Pending"}
{"OrderId":"002","CustomerId":"C456","Amount":149.50,"OrderDate":"2024-01-15T11:00:00Z","Status":"Pending"}
{"OrderId":"003","CustomerId":"C789","Amount":249.99,"OrderDate":"2024-01-15T11:30:00Z","Status":"Pending"}
```

Press `Ctrl+C` to exit the producer.

### Using Echo

```bash
echo '{"OrderId":"001","CustomerId":"C123","Amount":99.99,"OrderDate":"2024-01-15T10:30:00Z","Status":"Pending"}' | \
  docker exec -i kafka kafka-console-producer \
  --bootstrap-server localhost:9092 \
  --topic orders
```

### Using a Script

```bash
#!/bin/bash
# produce-messages.sh

for i in {1..10}; do
  ORDER_ID=$(printf "ORD-%03d" $i)
  CUSTOMER_ID=$(printf "CUST-%03d" $((RANDOM % 100 + 1)))
  AMOUNT=$(awk -v min=10 -v max=500 'BEGIN{srand(); print min+rand()*(max-min)}')

  MESSAGE="{\"OrderId\":\"$ORDER_ID\",\"CustomerId\":\"$CUSTOMER_ID\",\"Amount\":$AMOUNT,\"OrderDate\":\"2024-01-15T10:30:00Z\",\"Status\":\"Pending\"}"

  echo "$MESSAGE" | docker exec -i kafka kafka-console-producer \
    --bootstrap-server localhost:9092 \
    --topic orders

  echo "✓ Produced: $ORDER_ID"
  sleep 0.5
done

echo "✓ All messages produced!"
```

## 👀 Monitor Topics

### Watch Main Topic

```bash
docker exec kafka kafka-console-consumer \
  --bootstrap-server localhost:9092 \
  --topic orders \
  --from-beginning
```

### Watch Retry Topics

```bash
# Retry 1
docker exec kafka kafka-console-consumer \
  --bootstrap-server localhost:9092 \
  --topic orders.retry.1 \
  --from-beginning

# Retry 2
docker exec kafka kafka-console-consumer \
  --bootstrap-server localhost:9092 \
  --topic orders.retry.2 \
  --from-beginning

# Retry 3
docker exec kafka kafka-console-consumer \
  --bootstrap-server localhost:9092 \
  --topic orders.retry.3 \
  --from-beginning
```

### Watch Error Topic

```bash
docker exec kafka kafka-console-consumer \
  --bootstrap-server localhost:9092 \
  --topic orders.error \
  --from-beginning \
  --property print.headers=true
```

## 🔧 Advanced Operations

### Reset Consumer Group (start from beginning)

```bash
docker exec kafka kafka-consumer-groups \
  --bootstrap-server localhost:9092 \
  --group order-processor-group \
  --reset-offsets \
  --to-earliest \
  --topic orders \
  --execute
```

### View Consumer Group Status

```bash
docker exec kafka kafka-consumer-groups \
  --bootstrap-server localhost:9092 \
  --group order-processor-group \
  --describe
```

### Delete Topics (for clean restart)

```bash
docker exec kafka kafka-topics --bootstrap-server localhost:9092 --delete --topic orders
docker exec kafka kafka-topics --bootstrap-server localhost:9092 --delete --topic orders.retry.1
docker exec kafka kafka-topics --bootstrap-server localhost:9092 --delete --topic orders.retry.2
docker exec kafka kafka-topics --bootstrap-server localhost:9092 --delete --topic orders.retry.3
docker exec kafka kafka-topics --bootstrap-server localhost:9092 --delete --topic orders.error
```

Then restart docker-compose to recreate them.

## 🧪 Complete Testing Flow

### 1. Start Infrastructure

```bash
docker-compose up -d
```

### 2. Verify Topics

```bash
docker logs init-kafka
```

You should see output like:
```
Waiting for Kafka to be ready...
Creating topics...
Created topic orders.
Created topic orders.retry.1.
Created topic orders.retry.2.
Created topic orders.retry.3.
Created topic orders.error.

Available topics:
orders
orders.error
orders.retry.1
orders.retry.2
orders.retry.3

✓ Topics created successfully!
```

### 3. Run Your Consumer

```bash
dotnet run --project Kafka.Consumer.Resilient.Example
```

### 4. Produce Test Messages

In another terminal:

```bash
echo '{"OrderId":"001","CustomerId":"C123","Amount":99.99,"OrderDate":"2024-01-15T10:30:00Z","Status":"Pending"}' | \
  docker exec -i kafka kafka-console-producer --bootstrap-server localhost:9092 --topic orders
```

### 5. Watch the Magic! ✨

- Check your consumer logs
- Open Kafka UI (http://localhost:8080)
- See messages flow through retry topics
- See failed messages land in error topic

## 🛑 Stop Everything

```bash
# Stop services
docker-compose down

# Stop and remove volumes (clean slate)
docker-compose down -v
```

## 🐛 Troubleshooting

### Topics Not Created?

Check init-kafka logs:
```bash
docker logs init-kafka
```

### Kafka Not Ready?

Wait a few seconds and check:
```bash
docker exec kafka kafka-broker-api-versions --bootstrap-server localhost:9092
```

### Can't Connect?

Ensure ports are not in use:
```bash
lsof -i :9092  # Kafka
lsof -i :9093  # Kafka Controller (KRaft)
lsof -i :8080  # Kafka UI
lsof -i :8081  # Schema Registry
```

### Reset Everything

```bash
docker-compose down -v
docker-compose up -d
```

## 📚 Resources

- **Kafka UI**: http://localhost:8080
- **Schema Registry**: http://localhost:8081
- **Kafka Broker**: localhost:9092
- **Kafka Controller (KRaft)**: localhost:9093

