# Rust Engine - Database Architecture

## Overview

Rust engine menggunakan **hybrid database architecture** dengan:
1. **Standalone database** untuk Rust engine operations (high-performance)
2. **Integration layer** untuk sync dengan Odoo database
3. **Reporting database** untuk analytics (optional, bisa shared atau separate)

## Database Architecture Options

### Option 1: Standalone Database (Recommended)

```
┌─────────────────────────────────────────────────────────┐
│                    Odoo Database                        │
│  (PostgreSQL)                                           │
│  - Core Odoo data                                       │
│  - Product, Partner, etc.                               │
└────────────────────┬────────────────────────────────────┘
                     │
                     │ Sync (Read/Write)
                     │
        ┌────────────▼────────────┐
        │   Rust Engine Database  │
        │   (PostgreSQL)          │
        │   - Loyalty data        │
        │   - Point moves         │
        │   - Point history       │
        │   - Promotion data      │
        └────────────┬────────────┘
                     │
        ┌────────────▼────────────┐
        │   Reporting Database    │
        │   (PostgreSQL/ClickHouse)│
        │   - Analytics data      │
        │   - Aggregated data     │
        │   - Historical data     │
        └─────────────────────────┘
```

**Benefits**:
- **Performance**: Optimized schema untuk Rust engine
- **Scalability**: Independent scaling
- **Isolation**: Tidak impact Odoo database
- **Flexibility**: Bisa optimize tanpa constraint Odoo

**Challenges**:
- Data synchronization complexity
- Consistency management
- Transaction coordination

### Option 2: Shared Database (Alternative)

```
┌─────────────────────────────────────────────────────────┐
│              Shared PostgreSQL Database                  │
│  ┌──────────────────┐  ┌──────────────────┐            │
│  │  Odoo Schema     │  │  Rust Schema     │            │
│  │  - Core tables   │  │  - Engine tables │            │
│  └──────────────────┘  └──────────────────┘            │
└─────────────────────────────────────────────────────────┘
```

**Benefits**:
- **Simplicity**: No sync needed
- **Consistency**: ACID transactions
- **Reporting**: Direct access to all data

**Challenges**:
- Performance impact on Odoo
- Schema conflicts
- Limited optimization

## Recommended: Standalone with Sync

### Architecture Detail

```
┌─────────────────────────────────────────────────────────┐
│                    Odoo Application                     │
│  - POS Frontend                                         │
│  - Backend API                                          │
└────────────────────┬────────────────────────────────────┘
                     │
                     │ HTTP/gRPC
                     │
        ┌────────────▼────────────┐
        │   Rust Engine API       │
        │   - Order Processing   │
        │   - Point Calculation  │
        └────────────┬────────────┘
                     │
        ┌────────────▼────────────┐
        │   Rust Engine Database   │
        │   (PostgreSQL)          │
        │                         │
        │   Tables:               │
        │   - loyalty_cards       │
        │   - point_moves         │
        │   - point_history       │
        │   - promotion_programs  │
        │   - membership_levels   │
        └────────────┬────────────┘
                     │
                     │ Sync Service
                     │
        ┌────────────▼────────────┐
        │   Odoo Database         │
        │   (PostgreSQL)          │
        │   - Reference data      │
        │   - Product, Partner    │
        └─────────────────────────┘
```

## Database Schema Design

### Rust Engine Schema

```sql
-- Core Tables
CREATE TABLE loyalty_cards (
    id BIGSERIAL PRIMARY KEY,
    odoo_card_id INTEGER, -- Reference to Odoo
    program_id BIGINT NOT NULL,
    partner_id BIGINT NOT NULL, -- Reference to Odoo partner
    code VARCHAR(255) NOT NULL UNIQUE,
    points DECIMAL(18, 2) NOT NULL DEFAULT 0,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW(),
    synced_to_odoo BOOLEAN DEFAULT FALSE,
    sync_version INTEGER DEFAULT 0
);

CREATE TABLE point_moves (
    id BIGSERIAL PRIMARY KEY,
    loyalty_card_id BIGINT NOT NULL REFERENCES loyalty_cards(id),
    membership_product_id BIGINT NOT NULL,
    gain_date TIMESTAMP NOT NULL,
    expiration_date TIMESTAMP,
    initial_point DECIMAL(18, 2) NOT NULL,
    remaining_point DECIMAL(18, 2) NOT NULL,
    state VARCHAR(20) NOT NULL, -- active, depleted, expired
    origin VARCHAR(255), -- Source document
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    synced_to_odoo BOOLEAN DEFAULT FALSE
);

CREATE TABLE point_history (
    id BIGSERIAL PRIMARY KEY,
    loyalty_card_id BIGINT NOT NULL REFERENCES loyalty_cards(id),
    point_move_id BIGINT REFERENCES point_moves(id),
    points_amount DECIMAL(18, 2) NOT NULL,
    points_balance DECIMAL(18, 2) NOT NULL,
    transaction_time TIMESTAMP NOT NULL,
    origin VARCHAR(255),
    membership_product_id BIGINT NOT NULL,
    status_membership_product_id BIGINT NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    synced_to_odoo BOOLEAN DEFAULT FALSE
);

-- Indexes for Performance
CREATE INDEX idx_point_moves_card_state ON point_moves(loyalty_card_id, state, expiration_date);
CREATE INDEX idx_point_history_card ON point_history(loyalty_card_id, transaction_time DESC);
CREATE INDEX idx_loyalty_cards_partner ON loyalty_cards(partner_id);
CREATE INDEX idx_loyalty_cards_sync ON loyalty_cards(synced_to_odoo, sync_version);

-- Sync Tracking
CREATE TABLE sync_log (
    id BIGSERIAL PRIMARY KEY,
    entity_type VARCHAR(50) NOT NULL, -- 'loyalty_card', 'point_move', etc.
    entity_id BIGINT NOT NULL,
    action VARCHAR(20) NOT NULL, -- 'create', 'update', 'delete'
    status VARCHAR(20) NOT NULL, -- 'pending', 'synced', 'failed'
    error_message TEXT,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    synced_at TIMESTAMP
);
```

## Data Synchronization Strategy

### 1. Real-time Sync (Critical Data)

**Use Case**: Point transactions, order processing

**Flow**:
```
Rust Engine Operation
    ↓
Write to Rust DB
    ↓
Queue Sync Event
    ↓
Sync Service (Async)
    ↓
Write to Odoo DB
    ↓
Update Sync Status
```

**Implementation**:
```rust
pub struct SyncService {
    odoo_client: Arc<OdooClient>,
    event_bus: Arc<EventBus>,
}

impl SyncService {
    pub async fn sync_point_history(&self, history: &PointHistory) -> Result<()> {
        // 1. Write to Rust DB
        self.rust_repo.save_history(history).await?;
        
        // 2. Queue sync event
        self.event_bus.publish(SyncEvent::PointHistoryCreated {
            history_id: history.id,
        }).await?;
        
        // 3. Async sync to Odoo
        tokio::spawn(async move {
            self.sync_to_odoo(history).await
        });
        
        Ok(())
    }
    
    async fn sync_to_odoo(&self, history: &PointHistory) -> Result<()> {
        // Sync to Odoo via XML-RPC/JSON-RPC
        self.odoo_client.create_point_history(history).await?;
        
        // Update sync status
        self.rust_repo.mark_synced(history.id).await?;
        
        Ok(())
    }
}
```

### 2. Batch Sync (Non-Critical Data)

**Use Case**: Historical data, reporting data

**Flow**:
```
Rust Engine Operations
    ↓
Write to Rust DB
    ↓
Mark for Sync
    ↓
Batch Sync Job (Every 5 minutes)
    ↓
Sync to Odoo
```

**Implementation**:
```rust
pub struct BatchSyncService {
    odoo_client: Arc<OdooClient>,
    rust_repo: Arc<dyn PointRepository>,
}

impl BatchSyncService {
    pub async fn sync_pending(&self) -> Result<()> {
        // Get pending syncs
        let pending = self.rust_repo.get_pending_syncs(100).await?;
        
        for item in pending {
            match self.sync_item(&item).await {
                Ok(_) => {
                    self.rust_repo.mark_synced(item.id).await?;
                }
                Err(e) => {
                    self.rust_repo.mark_sync_failed(item.id, &e.to_string()).await?;
                }
            }
        }
        
        Ok(())
    }
}
```

### 3. Reference Data Sync (Odoo → Rust)

**Use Case**: Product, Partner, Membership levels

**Flow**:
```
Odoo Data Change
    ↓
Webhook/Event
    ↓
Rust Engine
    ↓
Update Reference Cache
```

**Implementation**:
```rust
pub struct ReferenceDataSync {
    odoo_client: Arc<OdooClient>,
    cache: Arc<RedisCache>,
}

impl ReferenceDataSync {
    pub async fn sync_partner(&self, partner_id: PartnerId) -> Result<()> {
        // Fetch from Odoo
        let partner = self.odoo_client.get_partner(partner_id).await?;
        
        // Update cache
        self.cache.set_partner(&partner).await?;
        
        Ok(())
    }
}
```

## Consistency Guarantees

### 1. Eventual Consistency

**Strategy**: Rust engine sebagai source of truth untuk loyalty data

**Guarantees**:
- Rust DB always consistent
- Odoo DB eventually consistent (within seconds)
- Conflict resolution: Rust wins

### 2. Transaction Coordination

**Two-Phase Commit** (for critical operations):

```rust
pub async fn process_order_with_sync(&self, order: Order) -> Result<()> {
    // Phase 1: Prepare
    let rust_tx = self.rust_db.begin().await?;
    let odoo_tx = self.odoo_client.begin_transaction().await?;
    
    // Phase 2: Commit
    match self.process_order_internal(order, &rust_tx).await {
        Ok(result) => {
            rust_tx.commit().await?;
            odoo_tx.commit().await?;
            Ok(result)
        }
        Err(e) => {
            rust_tx.rollback().await?;
            odoo_tx.rollback().await?;
            Err(e)
        }
    }
}
```

### 3. Conflict Resolution

**Strategy**: Last-write-wins dengan versioning

```rust
pub struct SyncConflictResolver;

impl SyncConflictResolver {
    pub async fn resolve(&self, rust_data: &Data, odoo_data: &Data) -> Data {
        // Compare versions
        if rust_data.sync_version > odoo_data.sync_version {
            // Rust is newer, use Rust
            rust_data.clone()
        } else {
            // Odoo is newer, sync from Odoo
            odoo_data.clone()
        }
    }
}
```

## Performance Optimization

### 1. Connection Pooling

```rust
use sqlx::postgres::PgPoolOptions;

let pool = PgPoolOptions::new()
    .max_connections(100)
    .acquire_timeout(Duration::from_secs(5))
    .connect(&database_url)
    .await?;
```

### 2. Read Replicas

```
┌─────────────────┐
│  Primary DB     │
│  (Writes)       │
└────────┬────────┘
         │
    ┌────┴────┐
    │         │
┌───▼───┐ ┌──▼────┐
│Replica│ │Replica│
│(Reads)│ │(Reads)│
└───────┘ └───────┘
```

### 3. Caching Strategy

```rust
pub struct CacheLayer {
    redis: Arc<Redis>,
}

impl CacheLayer {
    pub async fn get_card_balance(&self, card_id: CardId) -> Result<Option<Decimal>> {
        // Try cache first
        if let Some(balance) = self.redis.get(&format!("balance:{}", card_id)).await? {
            return Ok(Some(balance));
        }
        
        // Calculate from DB
        let balance = self.calculate_balance(card_id).await?;
        
        // Cache for 5 minutes
        self.redis.setex(
            &format!("balance:{}", card_id),
            balance,
            300
        ).await?;
        
        Ok(Some(balance))
    }
}
```

## Monitoring & Health Checks

### Database Health

```rust
pub async fn check_database_health(&self) -> HealthStatus {
    // Check connection
    match self.pool.acquire().await {
        Ok(_) => HealthStatus::Healthy,
        Err(_) => HealthStatus::Unhealthy,
    }
}
```

### Sync Status Monitoring

```rust
pub async fn get_sync_stats(&self) -> SyncStats {
    let pending = self.count_pending_syncs().await?;
    let failed = self.count_failed_syncs().await?;
    let synced = self.count_synced_today().await?;
    
    SyncStats {
        pending,
        failed,
        synced,
        sync_rate: synced as f64 / (pending + synced) as f64,
    }
}
```

Lihat dokumentasi berikutnya:
- `04-integration-flow.md` - Detail flow integrasi
- `05-reporting-architecture.md` - Arsitektur reporting
