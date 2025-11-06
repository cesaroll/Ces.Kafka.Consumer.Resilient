# Architecture Overview

## Project Structure

```
Ces.Kafka.Consumer.Resilient/
├── Kafka.Consumer.Resilient/              # Main Library Project
│   ├── Configuration/                      # Configuration models
│   │   ├── KafkaConsumerConfiguration.cs   # Main configuration
│   │   ├── ErrorConfiguration.cs           # Error topic config
│   │   ├── RetryPolicyConfiguration.cs     # Retry policy config
│   │   └── ConfigurationLoader.cs          # JSON config loader
│   ├── Models/                             # Data models
│   │   ├── ConsumerResult.cs               # Base result class
│   │   ├── SuccessResult.cs                # Success result
│   │   ├── RetryableResult.cs              # Retryable error result
│   │   ├── ErrorResult.cs                  # Fatal error result
│   │   └── MessageMetadata.cs              # Message metadata
│   ├── Interfaces/                         # Interfaces
│   │   ├── IResilientKafkaConsumer.cs      # Consumer interface
│   │   └── IMessageHandler.cs              # Message handler interface
│   ├── Services/                           # Service implementations
│   │   ├── ResilientKafkaConsumer.cs       # Main consumer implementation
│   │   └── RetryTopicNamingStrategy.cs     # Retry topic naming
│   ├── Extensions/                         # Extension methods
│   │   └── ServiceCollectionExtensions.cs  # DI registration
│   └── appsettings.example.json            # Example configuration
├── Kafka.Consumer.Resilient.Example/       # Example Application
│   ├── OrderMessage.cs                     # Sample message model
│   ├── OrderMessageHandler.cs              # Sample handler
│   ├── Program.cs                          # Application entry point
│   ├── appsettings.json                    # Configuration
│   └── README.md                           # Example documentation
├── Kafka.Consumer.Resilient.sln            # Solution file
├── docker-compose.yml                      # Kafka infrastructure
├── README.md                               # Main documentation
└── .gitignore                              # Git ignore rules
```

## Core Components

### 1. Configuration System

The library uses a hierarchical configuration model:

```
KafkaConsumerConfiguration
├── TopicName: string
├── ConsumerNumber: int
├── GroupId: string
├── SchemaRegistryUrl: string
├── BootstrapServers: string
├── Error
│   └── TopicName: string
└── RetryPolicy
    ├── Delay: int
    └── RetryAttempts: int
```

### 2. Consumer Results

Three result types guide message flow:

- **SuccessResult**: Message processed successfully, commit offset
- **RetryableResult**: Temporary failure, send to next retry topic
- **ErrorResult**: Permanent failure, send to error topic

### 3. Message Flow

```
┌─────────────────┐
│   Main Topic    │
│   (orders)      │
└────────┬────────┘
         │
         ▼
    ┌────────┐
    │ Handle │────► SuccessResult ───► ✓ Done
    └───┬────┘
        │
        ├─► RetryableResult ──┐
        │                     │
        └─► ErrorResult ──────┼───► Error Topic
                              │     (orders.error)
                              ▼
                    ┌──────────────────┐
                    │  Retry Topic 1   │
                    │ (orders.retry.1) │
                    └────────┬─────────┘
                             │
                             ▼
                        ┌────────┐
                        │ Handle │──► Success ──► ✓
                        └───┬────┘
                            │
                            ├─► RetryableResult ──┐
                            │                     │
                            └─► ErrorResult ──────┼──► Error Topic
                                                  │
                                                  ▼
                                        ┌──────────────────┐
                                        │  Retry Topic 2   │
                                        │ (orders.retry.2) │
                                        └────────┬─────────┘
                                                 │
                                                (continues...)
```

### 4. Retry Topic Naming Strategy

The library uses a predictable naming convention:
- Main topic: `{topicName}`
- Retry topics: `{topicName}.retry.{attemptNumber}`
- Error topic: `{topicName}.error`

Example:
- Main: `orders`
- Retry 1: `orders.retry.1`
- Retry 2: `orders.retry.2`
- Retry 3: `orders.retry.3`
- Error: `orders.error`

### 5. Consumer Architecture

```
┌───────────────────────────────────────────────┐
│     ResilientKafkaConsumer<TMessage>          │
├───────────────────────────────────────────────┤
│                                               │
│  ┌─────────────┐  ┌─────────────┐           │
│  │ Consumer 1  │  │ Consumer 2  │  ... N    │
│  └──────┬──────┘  └──────┬──────┘           │
│         │                │                   │
│         └────────┬───────┘                   │
│                  │                           │
│         ┌────────▼────────┐                  │
│         │ Message Handler │                  │
│         │  HandleAsync()  │                  │
│         └────────┬────────┘                  │
│                  │                           │
│         ┌────────▼────────┐                  │
│         │  Result Router  │                  │
│         └────┬───┬────┬───┘                  │
│              │   │    │                      │
│         ┌────▼   │    ▼────┐                 │
│         │Commit  │  Produce│                 │
│         │Offset  │  Retry/ │                 │
│         │        │  Error  │                 │
│         └────────┘  └──────┘                 │
└───────────────────────────────────────────────┘
```

### 6. Message Processing Pipeline

```
1. Consume Message
   ├─► Deserialize (JSON or Avro)
   ├─► Create Metadata
   ├─► Invoke Handler
   ├─► Route Result
   │   ├─► Success: Commit Offset
   │   ├─► Retryable: Produce to Retry Topic + Delay
   │   └─► Error: Produce to Error Topic
   └─► Commit Offset
```

## Design Patterns

### 1. Strategy Pattern
- **RetryTopicNamingStrategy**: Encapsulates topic naming logic

### 2. Template Method Pattern
- **ResilientKafkaConsumer**: Defines processing skeleton
- **IMessageHandler**: Custom processing logic

### 3. Dependency Injection
- All components registered via `AddResilientKafkaConsumer<TMessage, THandler>()`
- Supports both IConfiguration and Action<T> configuration

### 4. Factory Pattern
- Consumer instances created based on `ConsumerNumber` configuration

## Message Formats

### JSON
Automatic deserialization using System.Text.Json when `SchemaRegistryUrl` is not configured.

### Avro
Automatic deserialization using Confluent Schema Registry when `SchemaRegistryUrl` is configured.

## Error Handling

### Transient Errors (Retryable)
- Network timeouts
- Database connection failures
- Rate limiting
- Temporary service unavailability

These return `RetryableResult` and flow through retry topics with configurable delays.

### Permanent Errors (Non-Retryable)
- Invalid message format
- Business validation failures
- Authentication errors
- Missing required data

These return `ErrorResult` and go directly to the error topic.

## Scalability

### Horizontal Scaling
- Configure `ConsumerNumber` to run multiple consumers in parallel
- Each consumer processes messages independently
- Kafka automatically balances partitions across consumers

### Vertical Scaling
- Async/await throughout for efficient resource usage
- Minimal memory footprint per consumer
- Efficient serialization/deserialization

## Observability

### Logging
- Structured logging via Microsoft.Extensions.Logging
- Log levels: Debug, Information, Warning, Error
- Context-rich log messages with order IDs, retry attempts, etc.

### Monitoring Points
- Message consumption rate
- Success/retry/error rates
- Processing latency
- Retry attempt distribution

## Configuration Examples

### Minimal Configuration (JSON only)
```json
{
  "KafkaConsumer": {
    "TopicName": "my-topic",
    "GroupId": "my-group",
    "BootstrapServers": "localhost:9092",
    "Error": { "TopicName": "my-topic.error" }
  }
}
```

### Full Configuration (Avro with retries)
```json
{
  "KafkaConsumer": {
    "TopicName": "orders",
    "ConsumerNumber": 5,
    "GroupId": "order-processor",
    "SchemaRegistryUrl": "http://localhost:8081",
    "BootstrapServers": "kafka-1:9092,kafka-2:9092,kafka-3:9092",
    "Error": { "TopicName": "orders.dlq" },
    "RetryPolicy": {
      "Delay": 5000,
      "RetryAttempts": 5
    }
  }
}
```

## Testing Strategy

### Unit Tests (Recommended)
- Mock `IMessageHandler<T>` to test different result types
- Test `RetryTopicNamingStrategy` logic
- Test configuration validation

### Integration Tests (Recommended)
- Use Testcontainers to spin up Kafka
- Test end-to-end message flow
- Verify retry logic
- Validate error topic handling

### Load Tests (Optional)
- High-throughput scenarios
- Consumer scaling verification
- Retry backpressure handling

## Future Enhancements

Potential areas for expansion:
- Circuit breaker pattern integration
- Dead letter queue rotation
- Exponential backoff strategy
- Metrics/telemetry support (Prometheus, OpenTelemetry)
- Admin API for runtime configuration
- Message filtering capabilities
- Custom serializers/deserializers
- Transaction support

