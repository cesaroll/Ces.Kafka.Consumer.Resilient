# kcat Quick Reference Guide

`kcat` (formerly `kafkacat`) is a powerful command-line utility for working with Kafka. It's simpler and more reliable than Kafka's built-in console tools.

## Installation

```bash
# macOS
brew install kcat

# Linux
apt-get install kafkacat  # or kcat

# Or use Docker
docker run -it --network=host edenhill/kcat:1.7.1
```

## Basic Usage

### Producing Messages

#### Simple Message (No Key)
```bash
echo "Hello Kafka" | kcat -b localhost:9092 -t orders -P
```

#### Message with Key
```bash
echo "my-key:my-value" | kcat -b localhost:9092 -t orders -P -K:
```

#### JSON Message with Key
```bash
echo 'ORD-001:{"OrderId":"ORD-001","CustomerId":"CUST-100","Amount":99.99,"OrderDate":"2025-11-06T23:00:00Z","Status":"Pending"}' | \
  kcat -b localhost:9092 -t orders -P -K:
```

#### Multiple Messages
```bash
cat <<EOF | kcat -b localhost:9092 -t orders -P -K:
ORD-001:{"OrderId":"ORD-001","CustomerId":"CUST-100","Amount":99.99,"OrderDate":"2025-11-06T23:00:00Z","Status":"Pending"}
ORD-002:{"OrderId":"ORD-002","CustomerId":"CUST-101","Amount":149.99,"OrderDate":"2025-11-06T23:01:00Z","Status":"Pending"}
ORD-003:{"OrderId":"ORD-003","CustomerId":"CUST-102","Amount":79.99,"OrderDate":"2025-11-06T23:02:00Z","Status":"Pending"}
EOF
```

### Consuming Messages

#### Basic Consumption
```bash
# Consume from beginning
kcat -b localhost:9092 -t orders -C -o beginning

# Consume from end (latest)
kcat -b localhost:9092 -t orders -C -o end

# Consume specific number of messages
kcat -b localhost:9092 -t orders -C -c 10 -o beginning
```

#### Show Keys and Values
```bash
kcat -b localhost:9092 -t orders -C -f 'Key: %k | Value: %s\n' -o beginning
```

#### Show Detailed Metadata
```bash
kcat -b localhost:9092 -t orders -C \
  -f 'Topic: %t | Partition: %p | Offset: %o | Key: %k | Value: %s\n' \
  -o beginning
```

#### Consume Last Message
```bash
kcat -b localhost:9092 -t orders -C -c 1 -o end -o-1
```

#### Consume and Exit (No Waiting)
```bash
kcat -b localhost:9092 -t orders -C -e -o beginning
```

## Format Specifiers

Use with `-f` flag to customize output:

| Specifier | Description |
|-----------|-------------|
| `%s` | Message payload (value) |
| `%k` | Message key |
| `%t` | Topic name |
| `%p` | Partition number |
| `%o` | Offset |
| `%T` | Message timestamp |
| `%h` | Message headers |
| `%K` | Message key length |
| `%S` | Message payload length |

## Useful Examples

### List All Topics
```bash
kcat -b localhost:9092 -L
```

### Get Topic Metadata
```bash
kcat -b localhost:9092 -L -t orders
```

### Count Messages in Topic
```bash
kcat -b localhost:9092 -t orders -C -e -o beginning | wc -l
```

### Check Topic Offsets
```bash
kcat -b localhost:9092 -Q -t orders
```

### Consume from Specific Partition
```bash
kcat -b localhost:9092 -t orders -C -p 0 -o beginning
```

### Consume with Timeout
```bash
timeout 5 kcat -b localhost:9092 -t orders -C -o beginning
```

### Pretty Print JSON Messages
```bash
kcat -b localhost:9092 -t orders -C -e -o beginning | jq .
```

### Consume and Filter with jq
```bash
kcat -b localhost:9092 -t orders -C -f '%s\n' -o beginning | \
  jq 'select(.Amount > 100)'
```

### Produce from File
```bash
cat messages.txt | kcat -b localhost:9092 -t orders -P -K:
```

### Copy Messages Between Topics
```bash
kcat -b localhost:9092 -t orders -C -e -o beginning -f '%k:%s\n' | \
  kcat -b localhost:9092 -t orders-backup -P -K:
```

## Testing with Our Example

### Produce Test Orders
```bash
# Single order
echo 'ORD-001:{"OrderId":"ORD-001","CustomerId":"CUST-100","Amount":99.99,"OrderDate":"2025-11-06T23:00:00Z","Status":"Pending"}' | \
  kcat -b localhost:9092 -t orders -P -K:

# Multiple orders
for i in {1..10}; do
  ORDER_ID=$(printf "ORD-%03d" $i)
  CUSTOMER_ID=$(printf "CUST-%03d" $((RANDOM % 100 + 1)))
  AMOUNT=$(awk -v min=10 -v max=500 'BEGIN{srand(); print min+rand()*(max-min)}')
  DATE=$(date -u +"%Y-%m-%dT%H:%M:%SZ")

  echo "${ORDER_ID}:{\"OrderId\":\"${ORDER_ID}\",\"CustomerId\":\"${CUSTOMER_ID}\",\"Amount\":${AMOUNT},\"OrderDate\":\"${DATE}\",\"Status\":\"Pending\"}" | \
    kcat -b localhost:9092 -t orders -P -K:
done
```

### Verify Messages
```bash
# List all messages with keys
kcat -b localhost:9092 -t orders -C -e -o beginning \
  -f 'Key: %k | Order: %s\n'

# Count messages
kcat -b localhost:9092 -t orders -C -e -o beginning | wc -l

# Check error topic
kcat -b localhost:9092 -t orders.error -C -e -o beginning \
  -f 'Key: %k | Error: %s\n'

# Check retry topics
kcat -b localhost:9092 -t orders.retry.1 -C -e -o beginning \
  -f 'Key: %k | Retry 1: %s\n'
```

### Monitor in Real-Time
```bash
# Watch messages as they arrive
kcat -b localhost:9092 -t orders -C -f 'Key: %k | Value: %s\n'

# Monitor multiple topics
kcat -b localhost:9092 -t 'orders|orders.error|orders.retry.*' -C \
  -f 'Topic: %t | Key: %k | Value: %s\n'
```

## Tips & Best Practices

1. **Always specify keys for production messages** - Use `-K:` when producing
2. **Use `-e` for scripts** - Exit after consuming all messages
3. **Format output appropriately** - Use `-f` for structured logging
4. **Test with small batches first** - Use `-c N` to limit message count
5. **Pretty print JSON** - Pipe to `jq` for better readability
6. **Check offsets before consuming** - Use `-Q` to see current offsets
7. **Use consumer groups for load testing** - Add `-G group-name` flag

## Troubleshooting

### Can't Connect to Kafka
```bash
# Check if Kafka is accessible
kcat -b localhost:9092 -L

# Try with specific protocol
kcat -b localhost:9092 -L -X security.protocol=PLAINTEXT
```

### Messages Not Appearing
```bash
# Check topic exists
kcat -b localhost:9092 -L | grep orders

# Check current offsets
kcat -b localhost:9092 -Q -t orders

# Try consuming from beginning explicitly
kcat -b localhost:9092 -t orders -C -o beginning -e
```

### Key Not Being Set
```bash
# Verify key separator
echo "key:value" | kcat -b localhost:9092 -t orders -P -K: -v

# Check if key is present
kcat -b localhost:9092 -t orders -C -f 'Key: %k\n' -c 1 -o end -o-1
```

## Comparison with Kafka Console Tools

| Feature | kcat | kafka-console-consumer/producer |
|---------|------|--------------------------------|
| **Installation** | Simple | Requires full Kafka install |
| **Speed** | Fast | Slower |
| **Flexibility** | Highly flexible | Limited |
| **Format Options** | Extensive `-f` support | Basic |
| **Key Handling** | Simple `-K:` flag | Complex properties |
| **Metadata** | Easy access to all metadata | Limited |
| **JSON Support** | Works with jq | Requires additional parsing |

## Resources

- [kcat GitHub](https://github.com/edenhill/kcat)
- [kcat Documentation](https://docs.confluent.io/platform/current/app-development/kafkacat-usage.html)
- [Format Specifiers Reference](https://github.com/edenhill/kcat#message-properties)

## Integration with This Project

### Quick Workflow
```bash
# 1. Start infrastructure
make up

# 2. Produce test messages with kcat
echo 'ORD-001:{"OrderId":"ORD-001","CustomerId":"CUST-100","Amount":99.99,"OrderDate":"2025-11-06T23:00:00Z","Status":"Pending"}' | \
  kcat -b localhost:9092 -t orders -P -K:

# 3. Verify
kcat -b localhost:9092 -t orders -C -f 'Key: %k | Value: %s\n' -c 1

# 4. Run consumer
make run

# 5. Monitor all topics
kcat -b localhost:9092 -t 'orders.*' -C -f 'Topic: %t | Key: %k | Value: %s\n'
```

This provides a reliable, flexible way to test the Kafka consumer without relying on the potentially problematic `kafka-avro-console-producer`.

