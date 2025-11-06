# Changelog

All notable changes to the Ces.Kafka.Consumer.Resilient project will be documented in this file.

## [Unreleased]

### Added
- Initial release of Ces.Kafka.Consumer.Resilient library
- Resilient Kafka consumer with automatic retry logic
- Support for JSON and Avro message formats
- Configurable retry policies with custom delays and attempts
- Multiple concurrent consumers support
- Dead letter queue (error topic) for failed messages
- Comprehensive documentation (README, ARCHITECTURE, GETTING_STARTED, KAFKA_SETUP)
- Makefile for easy development operations
- Docker Compose setup with Kafka KRaft mode
- Example console application with complete implementation
- Helper script for producing test messages
- Three result types: SuccessResult, RetryableResult, ErrorResult
- Automatic topic creation via init-kafka service
- Kafka UI integration for monitoring

### Changed
- Using Kafka in **KRaft mode** (no Zookeeper dependency)
- Modern Docker Compose format (removed version field)
- Result types renamed from *Return to *Result (SuccessResult, RetryableResult, ErrorResult)
- Updated to Confluent Platform 7.8.0
- Topics created with 1 partition by default (configurable)

### Technical Details
- .NET 10.0 target framework
- Confluent.Kafka 2.3.0
- Confluent.SchemaRegistry 2.3.0
- Confluent.SchemaRegistry.Serdes.Avro 2.3.0
- Microsoft.Extensions.* integration for DI, Logging, Configuration

### Infrastructure
- Kafka broker on port 9092
- Kafka controller (KRaft) on port 9093
- Schema Registry on port 8081
- Kafka UI on port 8080
- Automatic topic creation:
  - `{topic}` (main topic)
  - `{topic}.retry.1` through `{topic}.retry.N`
  - `{topic}.error` (dead letter queue)

## Architecture Highlights

### Message Flow
```
Main Topic → Handler
  ├─► SuccessResult → ✓ Commit
  ├─► RetryableResult → Retry Topic N
  └─► ErrorResult → Error Topic (DLQ)
```

### Retry Strategy
- Configurable retry attempts (default: 3)
- Configurable delay between retries (default: 1000ms)
- Messages flow through retry topics: topic.retry.1 → topic.retry.2 → topic.retry.3
- After max retries, messages go to error topic

### KRaft Mode Benefits
- ✅ Simpler architecture (no Zookeeper)
- ✅ Faster startup and operations
- ✅ Better reliability with built-in consensus
- ✅ Production-ready (Kafka 3.3+)
- ✅ Future-proof (Zookeeper support deprecated in Kafka 4.0)

## Getting Started

```bash
# Start infrastructure
make up

# Run example consumer
make run

# Produce test messages
make produce

# Monitor in Kafka UI
open http://localhost:8080

# Stop everything
make down
```

## Documentation

- [README.md](README.md) - Main library documentation
- [GETTING_STARTED.md](GETTING_STARTED.md) - Step-by-step setup guide
- [ARCHITECTURE.md](ARCHITECTURE.md) - Technical architecture details
- [KAFKA_SETUP.md](KAFKA_SETUP.md) - Kafka operations and commands
- [Example README](Kafka.Consumer.Resilient.Example/README.md) - Example application guide

## Future Considerations

Potential enhancements:
- Circuit breaker pattern
- Exponential backoff strategy
- Metrics/telemetry (Prometheus, OpenTelemetry)
- Custom serializers/deserializers
- Admin API for runtime configuration
- Message filtering capabilities
- Transaction support
- Unit and integration tests

---

**Status**: Production-ready
**License**: MIT
**Author**: Cesar L

