# Rust Engine - Overview

## Pendahuluan

Rust Promotion & Membership Engine adalah solusi enterprise-grade yang dirancang untuk mengatasi semua masalah yang ada di sistem Python saat ini. Engine ini dibangun dengan Rust untuk performa tinggi, memory safety, dan concurrency yang aman.

## Mengapa Rust?

### 1. Performance
- **Zero-cost abstractions**: Secepat C/C++ tanpa overhead
- **No garbage collector**: Predictable performance
- **Optimized compilation**: LLVM backend optimization

### 2. Memory Safety
- **Compile-time guarantees**: Tidak ada null pointer, buffer overflow, atau data races
- **Ownership system**: Mencegah memory leaks dan use-after-free
- **Type system**: Strong typing mencegah banyak bugs

### 3. Concurrency Safety
- **Fearless concurrency**: Thread-safe by default
- **No data races**: Compiler guarantees
- **Async/await**: High-performance async I/O

### 4. Enterprise Readiness
- **Stability**: Backward compatibility guarantees
- **Ecosystem**: Rich crate ecosystem
- **Tooling**: Excellent tooling (cargo, rustfmt, clippy)

## Arsitektur High-Level

```
┌─────────────────────────────────────────────────────────┐
│                    Odoo POS Frontend                      │
│                    (JavaScript/OWL)                       │
└────────────────────┬────────────────────────────────────┘
                     │
                     │ HTTP/gRPC
                     │
        ┌────────────▼────────────┐
        │   Rust Engine API       │
        │   (REST/gRPC Gateway)   │
        └────────────┬────────────┘
                     │
        ┌────────────▼────────────┐
        │   Core Engine           │
        │   - Promotion Engine    │
        │   - Membership Engine   │
        │   - Point Engine        │
        │   - Discount Engine     │
        └────────────┬────────────┘
                     │
        ┌────────────▼────────────┐
        │   Data Layer            │
        │   - PostgreSQL Client   │
        │   - Redis Cache         │
        │   - Event Bus           │
        └────────────┬────────────┘
                     │
        ┌────────────▼────────────┐
        │   Odoo Integration      │
        │   - XML-RPC/JSON-RPC    │
        │   - Webhook Callbacks   │
        └─────────────────────────┘
```

## Komponen Utama

### 1. Promotion Engine
- Rule matching
- Reward calculation
- Discount application
- Program management

### 2. Membership Engine
- Level evaluation
- Upgrade/downgrade logic
- Status tracking
- Path management

### 3. Point Engine
- FEFO point consumption
- Balance calculation
- History tracking
- Expiration management

### 4. Discount Engine
- Multi-layer discount calculation
- Validation
- Priority handling
- Audit trail

## Keuntungan vs Sistem Saat Ini

| Aspek | Python (Current) | Rust Engine |
|-------|-----------------|-------------|
| **Performance** | ~100ms per order | ~10ms per order |
| **Concurrency** | GIL limitations | True parallelism |
| **Memory Safety** | Runtime errors | Compile-time guarantees |
| **Data Races** | Possible | Impossible |
| **Error Handling** | Try/except | Result types |
| **Type Safety** | Dynamic typing | Static typing |
| **Scalability** | Limited | High |

## Teknologi Stack

### Core
- **Rust**: 1.70+ (latest stable)
- **Tokio**: Async runtime
- **SQLx**: Type-safe SQL
- **Serde**: Serialization

### API
- **Axum**: Web framework
- **Tonic**: gRPC framework
- **Tower**: Middleware

### Data
- **PostgreSQL**: Primary database
- **Redis**: Caching & pub/sub
- **Postgres**: Connection pooling

### Testing
- **Criterion**: Benchmarking
- **Proptest**: Property testing
- **Mockall**: Mocking

## Deployment Architecture

```
┌─────────────────┐
│  Load Balancer  │
└────────┬────────┘
         │
    ┌────┴────┐
    │         │
┌───▼───┐ ┌──▼────┐
│Engine │ │Engine │
│Node 1 │ │Node 2 │
└───┬───┘ └──┬────┘
    │        │
    └───┬────┘
        │
┌───────▼────────┐
│  PostgreSQL    │
│  (Primary)     │
└───────┬────────┘
        │
┌───────▼────────┐
│  PostgreSQL    │
│  (Replica)     │
└───────────────┘
```

## Integration dengan Odoo

### Option 1: HTTP API (Recommended)
- REST API untuk order processing
- Webhook untuk events
- Simple integration

### Option 2: gRPC
- High-performance RPC
- Type-safe contracts
- Streaming support

### Option 3: Message Queue
- Async processing
- Better scalability
- Event-driven

## Migration Path

1. **Phase 1**: Deploy Rust engine alongside Python
2. **Phase 2**: Route new orders to Rust engine
3. **Phase 3**: Migrate existing functionality
4. **Phase 4**: Deprecate Python code

## Performance Targets

- **Order Processing**: < 10ms (p99)
- **Point Calculation**: < 5ms (p99)
- **Discount Calculation**: < 3ms (p99)
- **Concurrent Orders**: 10,000+ per second
- **Memory Usage**: < 100MB per instance

## Security Considerations

1. **Input Validation**: All inputs validated
2. **SQL Injection**: Prevented by SQLx
3. **Race Conditions**: Impossible in Rust
4. **Memory Safety**: Guaranteed by compiler
5. **Authentication**: JWT/OAuth2
6. **Authorization**: Role-based access control

## Monitoring & Observability

- **Metrics**: Prometheus
- **Logging**: Structured logging (tracing)
- **Tracing**: OpenTelemetry
- **Health Checks**: Built-in endpoints

## Next Steps

Lihat dokumentasi berikutnya:
- `02-architecture.md` - Detail arsitektur
- `03-api-design.md` - Desain API
- `04-data-structures.md` - Struktur data
- `05-performance.md` - Optimasi performa
- `06-integration.md` - Integrasi dengan Odoo
