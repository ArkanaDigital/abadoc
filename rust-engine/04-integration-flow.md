# Rust Engine - Integration Flow

## Overview

Dokumen ini menjelaskan detail flow integrasi antara Rust engine dengan Odoo, termasuk sinkronisasi data, event handling, dan error recovery.

## Integration Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    Odoo POS Frontend                     │
│                    (JavaScript/OWL)                      │
└────────────────────┬────────────────────────────────────┘
                     │
                     │ HTTP/gRPC
                     │
        ┌────────────▼────────────┐
        │   Rust Engine API        │
        │   - REST Endpoints      │
        │   - gRPC Services       │
        └────────────┬────────────┘
                     │
        ┌────────────▼────────────┐
        │   Rust Engine Core       │
        │   - Business Logic      │
        │   - Data Processing      │
        └────────────┬────────────┘
                     │
        ┌────────────▼────────────┐
        │   Rust Engine Database   │
        │   (PostgreSQL)           │
        └────────────┬────────────┘
                     │
        ┌────────────▼────────────┐
        │   Sync Service           │
        │   - Event Bus            │
        │   - Sync Queue           │
        └────────────┬────────────┘
                     │
        ┌────────────▼────────────┐
        │   Odoo Integration       │
        │   - XML-RPC/JSON-RPC    │
        │   - Webhooks            │
        └────────────┬────────────┘
                     │
        ┌────────────▼────────────┐
        │   Odoo Database          │
        │   (PostgreSQL)           │
        └─────────────────────────┘
```

## Integration Patterns

### Pattern 1: API Gateway (Recommended)

**Architecture**:
```
POS Frontend → Rust Engine API → Rust Engine → Rust DB
                                      ↓
                                 Sync Service → Odoo
```

**Benefits**:
- Clean separation
- Independent scaling
- Easy to test

### Pattern 2: Event-Driven

**Architecture**:
```
POS Frontend → Rust Engine → Rust DB
                    ↓
              Event Bus
                    ↓
         ┌──────────┴──────────┐
         │                     │
    Sync Service          Reporting
         │                     │
       Odoo              Analytics
```

**Benefits**:
- Decoupled
- Scalable
- Real-time

## Detailed Integration Flows

### Flow 1: Order Processing

```
┌──────────────┐
│ POS Frontend │
└──────┬───────┘
       │
       │ POST /api/v1/orders
       │
┌──────▼──────────────┐
│ Rust Engine API     │
│ - Validate          │
│ - Authenticate      │
└──────┬──────────────┘
       │
┌──────▼──────────────┐
│ Order Service       │
│ - Process order     │
│ - Calculate points  │
│ - Apply discounts   │
└──────┬──────────────┘
       │
┌──────▼──────────────┐
│ Point Service       │
│ - Earn points       │
│ - Redeem points     │
│ - Update balance    │
└──────┬──────────────┘
       │
┌──────▼──────────────┐
│ Rust DB             │
│ - Write transaction │
│ - Update balances   │
└──────┬──────────────┘
       │
┌──────▼──────────────┐
│ Sync Service        │
│ - Queue sync event  │
└──────┬──────────────┘
       │
┌──────▼──────────────┐
│ Async Sync          │
│ - Sync to Odoo      │
│ - Update status     │
└─────────────────────┘
```

**Code Implementation**:

```rust
pub struct OrderService {
    point_service: Arc<PointService>,
    discount_service: Arc<DiscountService>,
    sync_service: Arc<SyncService>,
    repo: Arc<dyn OrderRepository>,
}

impl OrderService {
    pub async fn process_order(&self, order: Order) -> Result<ProcessedOrder> {
        // 1. Validate order
        self.validate_order(&order).await?;
        
        // 2. Calculate discounts
        let discounts = self.discount_service.calculate(&order).await?;
        
        // 3. Process points
        let point_result = self.point_service.process_order_points(&order).await?;
        
        // 4. Save to Rust DB
        let processed_order = self.repo.save_order(&order, &discounts, &point_result).await?;
        
        // 5. Queue sync
        self.sync_service.queue_sync(SyncEvent::OrderProcessed {
            order_id: processed_order.id,
        }).await?;
        
        Ok(processed_order)
    }
}
```

### Flow 2: Point Synchronization

```
┌──────────────┐
│ Rust Engine  │
│ Point Update │
└──────┬───────┘
       │
┌──────▼──────────────┐
│ Write to Rust DB    │
│ - Point move        │
│ - Point history     │
│ - Card balance      │
└──────┬──────────────┘
       │
┌──────▼──────────────┐
│ Mark for Sync       │
│ - Set sync flag     │
│ - Create sync log   │
└──────┬──────────────┘
       │
┌──────▼──────────────┐
│ Sync Worker         │
│ (Every 1 second)    │
└──────┬──────────────┘
       │
┌──────▼──────────────┐
│ Fetch Pending       │
│ - Get unsynced      │
│ - Batch process     │
└──────┬──────────────┘
       │
┌──────▼──────────────┐
│ Sync to Odoo        │
│ - XML-RPC call      │
│ - Create/Update     │
└──────┬──────────────┘
       │
┌──────▼──────────────┐
│ Update Status       │
│ - Mark synced       │
│ - Log result        │
└─────────────────────┘
```

**Code Implementation**:

```rust
pub struct SyncWorker {
    rust_repo: Arc<dyn PointRepository>,
    odoo_client: Arc<OdooClient>,
    batch_size: usize,
}

impl SyncWorker {
    pub async fn run(&self) -> Result<()> {
        loop {
            // Get pending syncs
            let pending = self.rust_repo.get_pending_syncs(self.batch_size).await?;
            
            if pending.is_empty() {
                tokio::time::sleep(Duration::from_secs(1)).await;
                continue;
            }
            
            // Process batch
            for item in pending {
                match self.sync_item(&item).await {
                    Ok(_) => {
                        self.rust_repo.mark_synced(item.id).await?;
                    }
                    Err(e) => {
                        self.rust_repo.mark_sync_failed(item.id, &e.to_string()).await?;
                        // Retry logic
                        self.schedule_retry(item.id).await?;
                    }
                }
            }
        }
    }
    
    async fn sync_item(&self, item: &SyncItem) -> Result<()> {
        match item.entity_type {
            EntityType::PointHistory => {
                let history = self.rust_repo.get_point_history(item.entity_id).await?;
                self.odoo_client.create_point_history(&history).await?;
            }
            EntityType::PointMove => {
                let move = self.rust_repo.get_point_move(item.entity_id).await?;
                self.odoo_client.create_point_move(&move).await?;
            }
            // ... other types
        }
        Ok(())
    }
}
```

### Flow 3: Reference Data Sync (Odoo → Rust)

```
┌──────────────┐
│ Odoo Change  │
│ - Partner    │
│ - Product    │
│ - Membership │
└──────┬───────┘
       │
┌──────▼──────────────┐
│ Webhook/Event       │
│ - POST /webhook     │
└──────┬──────────────┘
       │
┌──────▼──────────────┐
│ Rust Webhook Handler│
│ - Validate          │
│ - Process           │
└──────┬──────────────┘
       │
┌──────▼──────────────┐
│ Update Cache        │
│ - Redis cache       │
│ - In-memory cache   │
└─────────────────────┘
```

**Code Implementation**:

```rust
pub struct WebhookHandler {
    cache: Arc<RedisCache>,
    odoo_client: Arc<OdooClient>,
}

impl WebhookHandler {
    pub async fn handle_partner_update(&self, partner_id: PartnerId) -> Result<()> {
        // Fetch latest from Odoo
        let partner = self.odoo_client.get_partner(partner_id).await?;
        
        // Update cache
        self.cache.set_partner(&partner).await?;
        
        // Invalidate related caches
        self.cache.invalidate_loyalty_card(partner_id).await?;
        
        Ok(())
    }
}
```

## Synchronization Strategies

### Strategy 1: Real-time Sync (Critical)

**Use Case**: Point transactions, order confirmations

**Implementation**:
```rust
// Immediate sync via event
event_bus.publish(SyncEvent::PointEarned { card_id, points }).await?;

// Async worker picks up immediately
```

**Latency**: < 1 second

### Strategy 2: Batch Sync (Non-Critical)

**Use Case**: Historical data, reporting data

**Implementation**:
```rust
// Batch every 5 minutes
tokio::spawn(async move {
    let mut interval = tokio::time::interval(Duration::from_secs(300));
    loop {
        interval.tick().await;
        sync_service.sync_pending().await?;
    }
});
```

**Latency**: < 5 minutes

### Strategy 3: On-Demand Sync

**Use Case**: User requests, reporting queries

**Implementation**:
```rust
pub async fn sync_on_demand(&self, entity_id: EntityId) -> Result<()> {
    // Force sync now
    self.sync_item(entity_id).await
}
```

**Latency**: Immediate

## Error Handling & Retry

### Retry Strategy

```rust
pub struct RetryPolicy {
    max_retries: u32,
    initial_delay: Duration,
    max_delay: Duration,
    backoff_multiplier: f64,
}

impl RetryPolicy {
    pub async fn execute<F, T>(&self, mut f: F) -> Result<T>
    where
        F: FnMut() -> Pin<Box<dyn Future<Output = Result<T>> + Send>>,
    {
        let mut delay = self.initial_delay;
        
        for attempt in 0..=self.max_retries {
            match f().await {
                Ok(result) => return Ok(result),
                Err(e) => {
                    if attempt == self.max_retries {
                        return Err(e);
                    }
                    
                    tokio::time::sleep(delay).await;
                    delay = min(
                        delay.mul_f64(self.backoff_multiplier),
                        self.max_delay
                    );
                }
            }
        }
        
        unreachable!()
    }
}
```

### Dead Letter Queue

```rust
pub struct DeadLetterQueue {
    repo: Arc<dyn SyncRepository>,
}

impl DeadLetterQueue {
    pub async fn handle_failed_sync(&self, item: SyncItem, error: &str) -> Result<()> {
        // After max retries, move to DLQ
        if item.retry_count >= MAX_RETRIES {
            self.repo.move_to_dlq(item.id, error).await?;
            
            // Alert
            self.alert_service.send_alert(
                Alert::SyncFailed {
                    entity_type: item.entity_type,
                    entity_id: item.entity_id,
                    error: error.to_string(),
                }
            ).await?;
        }
        
        Ok(())
    }
}
```

## Monitoring Integration

### Sync Metrics

```rust
pub struct SyncMetrics {
    pub pending_count: u64,
    pub synced_count: u64,
    pub failed_count: u64,
    pub sync_latency: Duration,
    pub sync_rate: f64, // synced per second
}
```

### Health Checks

```rust
pub async fn check_integration_health(&self) -> HealthStatus {
    // Check Rust DB
    let rust_db_ok = self.rust_db.ping().await.is_ok();
    
    // Check Odoo connection
    let odoo_ok = self.odoo_client.ping().await.is_ok();
    
    // Check sync queue
    let sync_queue_ok = self.sync_queue.is_healthy().await;
    
    match (rust_db_ok, odoo_ok, sync_queue_ok) {
        (true, true, true) => HealthStatus::Healthy,
        _ => HealthStatus::Degraded,
    }
}
```

## API Contracts

### REST API

```rust
// Order processing
POST /api/v1/orders
{
    "order_id": "123",
    "partner_id": 456,
    "lines": [...],
    "total": 1000.00
}

Response:
{
    "order_id": "123",
    "points_earned": 100,
    "points_redeemed": 50,
    "final_balance": 1050,
    "discounts": [...]
}
```

### gRPC Service

```protobuf
service LoyaltyService {
    rpc ProcessOrder(OrderRequest) returns (OrderResponse);
    rpc GetBalance(BalanceRequest) returns (BalanceResponse);
    rpc SyncStatus(SyncStatusRequest) returns (SyncStatusResponse);
}
```

Lihat dokumentasi berikutnya:
- `05-reporting-architecture.md` - Arsitektur reporting detail
