# Avro Message Guide

This guide explains how to work with Avro messages in the Resilient Kafka Consumer library.

## 📋 Table of Contents

- [What is Avro?](#what-is-avro)
- [Setup](#setup)
- [Avro Schema](#avro-schema)
- [Producing Avro Messages](#producing-avro-messages)
- [Consuming Avro Messages](#consuming-avro-messages)
- [Schema Registry](#schema-registry)

## What is Avro?

Apache Avro is a data serialization system that provides:
- **Compact binary format**: Smaller message size compared to JSON
- **Schema evolution**: Safe schema changes over time
- **Strong typing**: Schema validation at runtime
- **Language agnostic**: Works across different programming languages

## Setup

### Prerequisites

1. **Schema Registry running** (already configured in `docker-compose.yml`):
   ```bash
   make up
   ```

2. **Update appsettings.json** with Schema Registry URL:
   ```json
   {
     "KafkaConsumer": {
       "SchemaRegistryUrl": "http://localhost:8081",
       ...
     }
   }
   ```

## Avro Schema

The example includes an Avro schema for `OrderMessage`:

**Location**: `Kafka.Consumer.Resilient.Example/Schemas/OrderMessage.avsc`

```json
{
  "type": "record",
  "name": "OrderMessage",
  "namespace": "Kafka.Consumer.Resilient.Example.Avro",
  "fields": [
    {
      "name": "OrderId",
      "type": "string"
    },
    {
      "name": "CustomerId",
      "type": "string"
    },
    {
      "name": "Amount",
      "type": "double"
    },
    {
      "name": "OrderDate",
      "type": "string"
    },
    {
      "name": "Status",
      "type": "string"
    }
  ]
}
```

### Schema Compatibility Notes

When defining C# classes for Avro:
- Use `double` instead of `decimal` (Avro doesn't have native decimal type)
- Use `string` for dates in ISO 8601 format (or `long` for Unix timestamps)
- Property names must match schema field names exactly

## Producing Avro Messages

### Option 1: Using kcat (Recommended for Testing)

```bash
# Produce JSON messages with keys using kcat
echo 'ORD-001:{"OrderId":"ORD-001","CustomerId":"CUST-100","Amount":99.99,"OrderDate":"2025-11-06T23:00:00Z","Status":"Pending"}' | \
  kcat -b localhost:9092 -t orders -P -K:

# Verify keys are present
kcat -b localhost:9092 -t orders -C -f 'Key: %k | Value: %s\n' -c 1 -o end -o-1
```

**Advantages of kcat:**
- Simple and reliable
- Works with both JSON and Avro
- Easy to verify keys
- No timing issues

### Option 2: Using the Shell Script

```bash
# Produce JSON messages with keys
make produce

# Or directly
./produce-test-messages.sh 10
```

**Note:** The Avro producer script (`produce-avro-messages.sh`) uses `kafka-avro-console-producer` which may have timing issues. For testing, we recommend using the C# producer (Option 3) or kcat (Option 1) with JSON messages.

### Option 3: Using C# Producer (Best for Production)

```csharp
using Confluent.Kafka;
using Confluent.SchemaRegistry;
using Confluent.SchemaRegistry.Serdes;

var schemaRegistryConfig = new SchemaRegistryConfig
{
    Url = "http://localhost:8081"
};

var producerConfig = new ProducerConfig
{
    BootstrapServers = "localhost:9092"
};

var schemaRegistry = new CachedSchemaRegistryClient(schemaRegistryConfig);

var producer = new ProducerBuilder<string, OrderMessage>(producerConfig)
    .SetKeySerializer(new AvroSerializer<string>(schemaRegistry))
    .SetValueSerializer(new AvroSerializer<OrderMessage>(schemaRegistry))
    .Build();

var message = new OrderMessage
{
    OrderId = "ORD-001",
    CustomerId = "CUST-100",
    Amount = 99.99,
    OrderDate = DateTime.UtcNow.ToString("O"),
    Status = "Pending"
};

await producer.ProduceAsync("orders",
    new Message<string, OrderMessage>
    {
        Key = message.OrderId,
        Value = message
    });
```

## Consuming Avro Messages

The library **automatically handles both JSON and Avro** messages! No code changes needed.

### How It Works

1. **Schema Registry Configuration**: Set `SchemaRegistryUrl` in `appsettings.json`
2. **Automatic Deserialization**: The library detects the message format and deserializes accordingly
3. **Your Handler**: Works with strongly-typed C# objects

```csharp
public class OrderMessageHandler : IMessageHandler<OrderMessage>
{
    public async Task<ConsumerResult> HandleAsync(
        OrderMessage message,
        MessageMetadata metadata,
        CancellationToken cancellationToken)
    {
        // message is already deserialized from Avro
        Console.WriteLine($"Order: {message.OrderId}, Amount: {message.Amount}");

        return new SuccessResult();
    }
}
```

### Testing Avro Consumption

```bash
# Terminal 1: Start consumer
make run

# Terminal 2: Produce Avro messages
make produce-avro

# Watch logs
make logs
```

## Schema Registry

### View Registered Schemas

```bash
# List all subjects
curl http://localhost:8081/subjects

# Get latest version of orders-value schema
curl http://localhost:8081/subjects/orders-value/versions/latest

# View in browser
open http://localhost:8081/subjects
```

### Schema Naming Convention

- **Value Schema**: `{topic-name}-value` (e.g., `orders-value`)
- **Key Schema**: `{topic-name}-key` (e.g., `orders-key`)

### Schema Evolution

Avro supports safe schema changes:

✅ **Safe Changes (Backward Compatible)**:
- Adding optional fields (with defaults)
- Removing fields (if they have defaults)

❌ **Breaking Changes**:
- Removing required fields
- Changing field types
- Renaming fields without aliases

Example - Adding optional field:

```json
{
  "type": "record",
  "name": "OrderMessage",
  "fields": [
    // ... existing fields ...
    {
      "name": "Priority",
      "type": ["null", "string"],
      "default": null
    }
  ]
}
```

## Monitoring

### Kafka UI
- **URL**: http://localhost:8080
- View messages in Avro or JSON format
- Inspect schema registry

### Schema Registry UI
- **URL**: http://localhost:8081
- Browse registered schemas
- View schema versions
- Check compatibility settings

### Console Consumer

```bash
# Consume Avro messages (shows deserialized JSON)
docker exec schema-registry kafka-avro-console-consumer \
  --bootstrap-server kafka:29092 \
  --topic orders \
  --from-beginning \
  --property schema.registry.url=http://localhost:8081
```

## Comparison: JSON vs Avro

| Feature | JSON | Avro |
|---------|------|------|
| **Size** | Larger (text) | Smaller (binary) |
| **Speed** | Slower | Faster |
| **Schema** | None | Required |
| **Validation** | Runtime | Compile + Runtime |
| **Evolution** | Manual | Built-in |
| **Human Readable** | Yes | No (binary) |

## Best Practices

1. **Version Your Schemas**: Use semantic versioning in documentation
2. **Default Values**: Provide defaults for optional fields
3. **Schema Evolution**: Plan for backward compatibility
4. **Test Both Formats**: Test with both JSON and Avro in development
5. **Monitor Schema Registry**: Keep an eye on registered schemas
6. **Documentation**: Document your schema changes

## Troubleshooting

### Schema Registry Connection Issues

```bash
# Check if Schema Registry is running
docker ps | grep schema-registry

# Check Schema Registry logs
docker logs schema-registry

# Test connection
curl http://localhost:8081/subjects
```

### Serialization Errors

- **Ensure C# class matches schema exactly**
- **Check data types** (double vs decimal, string vs DateTime)
- **Verify Schema Registry URL** in configuration

### Schema Not Found

```bash
# Register schema manually
curl -X POST -H "Content-Type: application/vnd.schemaregistry.v1+json" \
  --data @Kafka.Consumer.Resilient.Example/Schemas/OrderMessage.avsc \
  http://localhost:8081/subjects/orders-value/versions
```

## Additional Resources

- [Apache Avro Documentation](https://avro.apache.org/docs/)
- [Confluent Schema Registry](https://docs.confluent.io/platform/current/schema-registry/index.html)
- [Avro Best Practices](https://docs.confluent.io/platform/current/schema-registry/avro.html)

