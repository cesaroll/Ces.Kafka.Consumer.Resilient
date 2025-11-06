# Avro Message Producer

Este proyecto produce **mensajes Avro reales** (binarios) con keys para testing.

## 🎯 Características

- ✅ Mensajes Avro binarios (no JSON)
- ✅ Schema automáticamente registrado en Schema Registry
- ✅ Keys para particionamiento correcto
- ✅ Serialización eficiente (~59 bytes vs ~120+ bytes de JSON)

## 🚀 Uso

### Opción 1: Con Makefile

```bash
# Producir 10 mensajes Avro
make produce-avro

# Producir cantidad específica
make produce-avro N=20
```

### Opción 2: Directamente

```bash
# Script bash
./produce-avro-messages.sh 10

# Comando C# directo
dotnet run --project Kafka.Consumer.Resilient.AvroProducer/ 10
```

### Opción 3: Con parámetros

```bash
dotnet run --project Kafka.Consumer.Resilient.AvroProducer/ \
  [count] \
  [bootstrap-servers] \
  [schema-registry-url] \
  [topic]

# Ejemplo
dotnet run --project Kafka.Consumer.Resilient.AvroProducer/ \
  5 \
  localhost:9092 \
  http://localhost:8081 \
  orders
```

## 🔍 Verificación

### Ver el schema registrado

```bash
curl http://localhost:8081/subjects/orders-value/versions/latest | python3 -m json.tool
```

### Ver mensajes con keys

```bash
kcat -b localhost:9092 -t orders -C -f 'Key: %k | Offset: %o | Size: %S bytes\n' -e -o beginning
```

### Consumir con Avro console consumer

```bash
docker exec schema-registry kafka-avro-console-consumer \
  --bootstrap-server kafka:29092 \
  --topic orders \
  --from-beginning \
  --property schema.registry.url=http://localhost:8081
```

### Comparar tamaños JSON vs Avro

```bash
# Producir JSON
./produce-test-messages.sh 1

# Producir Avro
./produce-avro-messages.sh 1

# Ver tamaños
kcat -b localhost:9092 -t orders -C -f 'Offset %o: %S bytes\n' -e
```

## 📝 Implementación

El productor implementa `ISpecificRecord` de Apache Avro:

```csharp
public class OrderMessage : ISpecificRecord
{
    public static Schema _SCHEMA = Schema.Parse(@"{...}");
    public Schema Schema => _SCHEMA;

    // Propiedades
    public string OrderId { get; set; }
    public string CustomerId { get; set; }
    public double Amount { get; set; }
    public string OrderDate { get; set; }
    public string Status { get; set; }

    // Métodos ISpecificRecord
    public object Get(int fieldPos) { ... }
    public void Put(int fieldPos, object fieldValue) { ... }
}
```

## ⚙️ Configuración

### appsettings.json del Consumer

```json
{
  "KafkaConsumer": {
    "SchemaRegistryUrl": "http://localhost:8081",
    ...
  }
}
```

## 🎓 Aprendizajes

### ¿Por qué ISpecificRecord?

El serializador de Confluent requiere:
- Tipos primitivos (int, string, double, etc.)
- Implementaciones de `ISpecificRecord`
- Subclases de `SpecificFixed`

### Schema Registry

- El schema se registra automáticamente en la primera producción
- Subject name: `{topic-name}-value` (ej: `orders-value`)
- Compatible con Schema Evolution de Avro

### Ventajas de Avro

1. **Compacto**: ~50% más pequeño que JSON
2. **Tipado**: Validación en tiempo de compilación y runtime
3. **Evolución**: Schema evolution con backward/forward compatibility
4. **Interoperabilidad**: Funciona entre diferentes lenguajes

## 🔧 Troubleshooting

### Error: "Value serialization error"

Asegúrate de que la clase implemente `ISpecificRecord` correctamente.

### Error: "Cannot connect to Schema Registry"

Verifica que Schema Registry esté corriendo:

```bash
docker ps | grep schema-registry
curl http://localhost:8081/subjects
```

### Messages no aparecen

Verifica el flush:

```csharp
producer.Flush(TimeSpan.FromSeconds(10));
```

## 📚 Referencias

- [Apache Avro](https://avro.apache.org/)
- [Confluent Schema Registry](https://docs.confluent.io/platform/current/schema-registry/)
- [Confluent Kafka .NET](https://docs.confluent.io/kafka-clients/dotnet/current/overview.html)

