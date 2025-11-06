# Getting Started Guide

## 🚀 Complete Setup in 5 Minutes

This guide will get you up and running with the Ces.Kafka.Consumer.Resilient library.

## Step 1: Start Kafka Infrastructure

```bash
# Clone/navigate to the repository
cd Ces.Kafka.Consumer.Resilient

# Start all services (Kafka in KRaft mode, Schema Registry, Kafka UI)
make up
```

This will:
- ✅ Start Kafka on port 9092 (KRaft mode - no Zookeeper!)
- ✅ Start Schema Registry on port 8081
- ✅ Start Kafka UI on port 8080
- ✅ Automatically create all required topics

**Verify it's running:**
```bash
make ps          # Check containers
make topics      # List topics
```

You should see:
- `orders`
- `orders.retry.1`
- `orders.retry.2`
- `orders.retry.3`
- `orders.error`

## Step 2: Build the Solution

```bash
make build
```

## Step 3: Run the Example Consumer

In one terminal:
```bash
make run
```

You should see output like:
```
=== Ces.Kafka.Consumer.Resilient Example ===

Starting Kafka consumer...
Configuration:
  - Topic: orders
  - Consumer Count: 2
  - Group ID: order-processor-group
  ...
Press Ctrl+C to stop...
```

## Step 4: Produce Test Messages

In another terminal:
```bash
# Produce 10 test messages
make produce

# Or produce more
make produce N=50
```

## Step 5: Watch the Magic! ✨

### In Your Console
You'll see messages being processed:
```
info: Kafka.Consumer.Resilient.Example.OrderMessageHandler[0]
      Processing order ORD-001 for customer CUST-123, amount: $99.99, retry attempt: 0
```

### In Kafka UI (http://localhost:8080)
1. Click on "Topics"
2. Watch messages flow through:
   - `orders` → main processing
   - `orders.retry.1` → first retry (if RetryableResult)
   - `orders.retry.2` → second retry
   - `orders.retry.3` → third retry
   - `orders.error` → final error destination

### Monitor with CLI
```bash
# Watch error topic
docker exec kafka kafka-console-consumer \
  --bootstrap-server localhost:9092 \
  --topic orders.error \
  --from-beginning \
  --property print.headers=true
```

## Understanding the Flow

The example handler simulates three scenarios:

1. **80% Success** - Messages process successfully
2. **10% Retryable** - Messages retry through retry topics
3. **10% Error** - Messages go directly to error topic

### Message Journey

```
orders (main topic)
  ↓
  ├─► SuccessResult → ✓ Done
  ├─► RetryableResult → orders.retry.1
  │                      ↓
  │                      ├─► Success → ✓ Done
  │                      └─► Retryable → orders.retry.2
  │                                      ↓
  │                                      └─► ... → orders.retry.3
  └─► ErrorResult → orders.error (DLQ)
```

## Common Operations

### View Logs
```bash
make logs          # All services
make logs-kafka    # Kafka only
make logs-init     # Topic creation logs
```

### Stop Services
```bash
make down          # Stop
make clean         # Stop and remove all data
```

### Restart
```bash
make restart
```

## Next Steps

### 1. Create Your Own Consumer

```bash
# Create new project
dotnet new console -n MyKafkaConsumer
cd MyKafkaConsumer

# Add reference
dotnet add reference ../Kafka.Consumer.Resilient/Kafka.Consumer.Resilient.csproj

# Or install from NuGet (once published)
dotnet add package Ces.Kafka.Consumer.Resilient
```

### 2. Define Your Message

```csharp
public class MyMessage
{
    public string Id { get; set; }
    public string Data { get; set; }
}
```

### 3. Implement Your Handler

```csharp
using Ces.Kafka.Consumer.Resilient.Interfaces;
using Ces.Kafka.Consumer.Resilient.Models;

public class MyMessageHandler : IMessageHandler<MyMessage>
{
    public async Task<ConsumerResult> HandleAsync(
        MyMessage message,
        MessageMetadata metadata,
        CancellationToken cancellationToken = default)
    {
        try
        {
            // Your business logic here
            await ProcessMessage(message);
            return new SuccessResult();
        }
        catch (TemporaryException ex)
        {
            // Will retry
            return new RetryableResult("Temporary failure", ex);
        }
        catch (Exception ex)
        {
            // Goes to error topic
            return new ErrorResult("Fatal error", ex);
        }
    }
}
```

### 4. Configure appsettings.json

```json
{
  "KafkaConsumer": {
    "TopicName": "my-topic",
    "ConsumerNumber": 3,
    "GroupId": "my-consumer-group",
    "BootstrapServers": "localhost:9092",
    "SchemaRegistryUrl": "",
    "Error": {
      "TopicName": "my-topic.error"
    },
    "RetryPolicy": {
      "Delay": 2000,
      "RetryAttempts": 3
    }
  }
}
```

### 5. Update docker-compose.yml Topics

Add your topics to `docker-compose.yml` in the `init-kafka` service:

```yaml
kafka-topics --bootstrap-server kafka:29092 --create --if-not-exists --partitions 1 --replication-factor 1 --topic my-topic
kafka-topics --bootstrap-server kafka:29092 --create --if-not-exists --partitions 1 --replication-factor 1 --topic my-topic.retry.1
kafka-topics --bootstrap-server kafka:29092 --create --if-not-exists --partitions 1 --replication-factor 1 --topic my-topic.retry.2
kafka-topics --bootstrap-server kafka:29092 --create --if-not-exists --partitions 1 --replication-factor 1 --topic my-topic.retry.3
kafka-topics --bootstrap-server kafka:29092 --create --if-not-exists --partitions 1 --replication-factor 1 --topic my-topic.error
```

### 6. Run Your Consumer

```bash
make up    # Restart to create new topics
make run
```

## Troubleshooting

### Kafka Not Starting?
```bash
make logs-kafka
```

Look for errors. Common issues:
- Port 9092 already in use
- Insufficient memory/resources

### Topics Not Created?
```bash
make logs-init
```

Should show topic creation output.

### Consumer Not Processing?
1. Check consumer is running: `make ps`
2. Verify topics exist: `make topics`
3. Check consumer logs
4. View in Kafka UI: http://localhost:8080

### Reset Everything
```bash
make clean    # Remove all data
make up       # Fresh start
```

## Production Considerations

When moving to production:

1. **Use multiple partitions** for parallel processing
   ```yaml
   --partitions 10  # Instead of 1
   ```

2. **Use proper replication**
   ```yaml
   --replication-factor 3  # Instead of 1
   ```

3. **Scale consumers**
   ```json
   "ConsumerNumber": 10  // Match partition count
   ```

4. **Configure proper retry delays**
   ```json
   "RetryPolicy": {
     "Delay": 5000,      // 5 seconds
     "RetryAttempts": 5
   }
   ```

5. **Use Avro for schema evolution**
   ```json
   "SchemaRegistryUrl": "http://schema-registry:8081"
   ```

6. **Monitor with Kafka UI or other tools**
   - Kafka UI: http://localhost:8080
   - Prometheus + Grafana
   - Confluent Control Center

## Additional Resources

- [README.md](README.md) - Library documentation
- [ARCHITECTURE.md](ARCHITECTURE.md) - Technical details
- [KAFKA_SETUP.md](KAFKA_SETUP.md) - Kafka operations guide
- [Example README](Kafka.Consumer.Resilient.Example/README.md) - Example documentation

## Need Help?

Check out the example application in `Kafka.Consumer.Resilient.Example/` for a complete working implementation.

Happy consuming! 🎉

