# Rust Engine - Arsitektur Detail

## Arsitektur Layered

```
┌─────────────────────────────────────────────────────────┐
│                    API Layer                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │ REST API     │  │ gRPC API     │  │ WebSocket    │  │
│  └──────────────┘  └──────────────┘  └──────────────┘  │
└─────────────────────────────────────────────────────────┘
                         │
┌─────────────────────────────────────────────────────────┐
│                  Service Layer                           │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │ Promotion    │  │ Membership   │  │ Point        │  │
│  │ Service      │  │ Service      │  │ Service      │  │
│  └──────────────┘  └──────────────┘  └──────────────┘  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │ Discount     │  │ Order        │  │ Event        │  │
│  │ Service      │  │ Service      │  │ Service      │  │
│  └──────────────┘  └──────────────┘  └──────────────┘  │
└─────────────────────────────────────────────────────────┘
                         │
┌─────────────────────────────────────────────────────────┐
│                  Domain Layer                            │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │ Promotion    │  │ Membership   │  │ Point        │  │
│  │ Domain       │  │ Domain       │  │ Domain       │  │
│  └──────────────┘  └──────────────┘  └──────────────┘  │
└─────────────────────────────────────────────────────────┘
                         │
┌─────────────────────────────────────────────────────────┐
│                  Infrastructure Layer                    │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │ Repository   │  │ Cache        │  │ Event Bus    │  │
│  │ (PostgreSQL) │  │ (Redis)      │  │ (Redis)      │  │
│  └──────────────┘  └──────────────┘  └──────────────┘  │
└─────────────────────────────────────────────────────────┘
```

## Domain Models

### Promotion Domain

```rust
pub struct PromotionProgram {
    pub id: ProgramId,
    pub name: String,
    pub program_type: ProgramType,
    pub date_from: Option<DateTime<Utc>>,
    pub date_to: Option<DateTime<Utc>>,
    pub rules: Vec<PromotionRule>,
    pub rewards: Vec<PromotionReward>,
    pub pos_config_ids: Vec<PosConfigId>,
    pub active: bool,
}

pub enum ProgramType {
    Promotion,
    BuyXGetY,
    Coupons,
    PromoCode,
    Referral,
    NextOrderCoupons,
}

pub struct PromotionRule {
    pub id: RuleId,
    pub minimum_amount: Decimal,
    pub minimum_qty: Decimal,
    pub product_ids: Vec<ProductId>,
    pub membership_status: Option<MembershipProductId>,
    pub reward_point_amount: Decimal,
    pub reward_point_mode: RewardPointMode,
}

pub enum RewardPointMode {
    Order,
    Money,
    Unit,
}

pub struct PromotionReward {
    pub id: RewardId,
    pub reward_type: RewardType,
    pub discount: Option<Discount>,
    pub product: Option<ProductReward>,
    pub required_points: Decimal,
}

pub enum RewardType {
    Discount,
    Product,
    SpecialPrice,
    SpecialSet,
    MultipleDiscount,
    Point,
}
```

### Membership Domain

```rust
pub struct Membership {
    pub partner_id: PartnerId,
    pub membership_product_id: MembershipProductId,
    pub membership_start: DateTime<Utc>,
    pub membership_stop: Option<DateTime<Utc>>,
    pub loyalty_card: LoyaltyCard,
}

pub struct LoyaltyCard {
    pub id: CardId,
    pub program_id: ProgramId,
    pub partner_id: PartnerId,
    pub code: String,
    pub points: Decimal, // Computed from point moves
}

pub struct MembershipLevelPath {
    pub id: LevelPathId,
    pub membership_product_id: MembershipProductId,
    pub threshold: Decimal,
    pub sequence: i32,
    pub is_upgradable: bool,
    pub is_downgradable: bool,
}
```

### Point Domain

```rust
pub struct PointMove {
    pub id: PointMoveId,
    pub loyalty_card_id: CardId,
    pub membership_product_id: MembershipProductId,
    pub gain_date: DateTime<Utc>,
    pub expiration_date: Option<DateTime<Utc>>,
    pub initial_point: Decimal,
    pub remaining_point: Decimal,
    pub state: PointMoveState,
    pub origin: String, // Source document
}

pub enum PointMoveState {
    Active,
    Depleted,
    Expired,
}

pub struct PointHistory {
    pub id: PointHistoryId,
    pub loyalty_card_id: CardId,
    pub point_move_id: Option<PointMoveId>,
    pub points_amount: Decimal, // + for earn, - for redeem
    pub points_balance: Decimal,
    pub transaction_time: DateTime<Utc>,
    pub origin: String,
    pub membership_product_id: MembershipProductId,
    pub status_membership_product_id: MembershipProductId,
}
```

## Service Layer

### Promotion Service

```rust
pub struct PromotionService {
    repo: Arc<dyn PromotionRepository>,
    cache: Arc<dyn Cache>,
}

impl PromotionService {
    pub async fn find_eligible_programs(
        &self,
        order: &Order,
        pos_config_id: PosConfigId,
    ) -> Result<Vec<PromotionProgram>> {
        // 1. Load active programs for POS config
        // 2. Match rules against order
        // 3. Return eligible programs
    }
    
    pub async fn apply_reward(
        &self,
        order: &mut Order,
        reward: &PromotionReward,
    ) -> Result<AppliedReward> {
        // 1. Validate reward eligibility
        // 2. Apply reward to order
        // 3. Calculate discount
        // 4. Return applied reward
    }
}
```

### Membership Service

```rust
pub struct MembershipService {
    repo: Arc<dyn MembershipRepository>,
    point_service: Arc<PointService>,
}

impl MembershipService {
    pub async fn evaluate_membership(
        &self,
        partner_id: PartnerId,
    ) -> Result<MembershipEvaluation> {
        // 1. Calculate total spending
        // 2. Check membership level path
        // 3. Determine if upgrade/downgrade needed
        // 4. Return evaluation result
    }
    
    pub async fn change_membership(
        &self,
        partner_id: PartnerId,
        new_membership_id: MembershipProductId,
    ) -> Result<()> {
        // 1. Lock partner
        // 2. Transfer points
        // 3. Update membership
        // 4. Unlock partner
    }
}
```

### Point Service

```rust
pub struct PointService {
    repo: Arc<dyn PointRepository>,
    cache: Arc<dyn Cache>,
}

impl PointService {
    pub async fn add_points(
        &self,
        card_id: CardId,
        points: Decimal,
        membership_product_id: MembershipProductId,
        origin: String,
    ) -> Result<PointMove> {
        // 1. Lock card
        // 2. Create point move
        // 3. Create point history
        // 4. Update card balance
        // 5. Unlock card
    }
    
    pub async fn deduct_points(
        &self,
        card_id: CardId,
        points: Decimal,
        membership_product_id: MembershipProductId,
        origin: String,
    ) -> Result<Vec<PointMove>> {
        // 1. Lock card
        // 2. Find point moves (FEFO)
        // 3. Consume points
        // 4. Create point history
        // 5. Update card balance
        // 6. Unlock card
    }
    
    pub async fn get_balance(
        &self,
        card_id: CardId,
    ) -> Result<Decimal> {
        // 1. Try cache first
        // 2. If not in cache, calculate from history
        // 3. Cache result
        // 4. Return balance
    }
}
```

## Repository Pattern

### Promotion Repository

```rust
#[async_trait]
pub trait PromotionRepository: Send + Sync {
    async fn find_by_pos_config(
        &self,
        pos_config_id: PosConfigId,
        active: bool,
    ) -> Result<Vec<PromotionProgram>>;
    
    async fn find_by_id(
        &self,
        id: ProgramId,
    ) -> Result<Option<PromotionProgram>>;
    
    async fn save(
        &self,
        program: &PromotionProgram,
    ) -> Result<()>;
}
```

### Point Repository

```rust
#[async_trait]
pub trait PointRepository: Send + Sync {
    async fn find_active_moves(
        &self,
        card_id: CardId,
        membership_product_id: MembershipProductId,
    ) -> Result<Vec<PointMove>>;
    
    async fn save_move(
        &self,
        move: &PointMove,
    ) -> Result<PointMoveId>;
    
    async fn save_history(
        &self,
        history: &PointHistory,
    ) -> Result<PointHistoryId>;
    
    async fn get_latest_balance(
        &self,
        card_id: CardId,
    ) -> Result<Option<Decimal>>;
}
```

## Concurrency & Locking

### Database-Level Locking

```rust
impl PointService {
    pub async fn deduct_points(&self, ...) -> Result<Vec<PointMove>> {
        // Use SELECT FOR UPDATE
        let mut tx = self.repo.begin_transaction().await?;
        
        // Lock card
        let card = tx
            .lock_card_for_update(card_id)
            .await?;
        
        // Find and lock point moves
        let moves = tx
            .find_active_moves_for_update(card_id, membership_product_id)
            .await?;
        
        // Consume points
        for move in moves {
            // Update move
            tx.update_move(move).await?;
        }
        
        // Commit transaction
        tx.commit().await?;
        
        Ok(moves)
    }
}
```

### Distributed Locking (Redis)

```rust
use redis::Commands;

impl PointService {
    pub async fn deduct_points_with_lock(&self, ...) -> Result<Vec<PointMove>> {
        let lock_key = format!("lock:card:{}", card_id);
        let lock = self.redis.lock(&lock_key, 30).await?; // 30 second timeout
        
        // Critical section
        let result = self.deduct_points_internal(...).await;
        
        lock.release().await?;
        result
    }
}
```

## Error Handling

### Result Types

```rust
pub type Result<T> = std::result::Result<T, EngineError>;

#[derive(Debug, thiserror::Error)]
pub enum EngineError {
    #[error("Database error: {0}")]
    Database(#[from] sqlx::Error),
    
    #[error("Insufficient points: required {required}, available {available}")]
    InsufficientPoints {
        required: Decimal,
        available: Decimal,
    },
    
    #[error("Invalid membership: {0}")]
    InvalidMembership(String),
    
    #[error("Concurrent modification detected")]
    ConcurrentModification,
    
    #[error("Validation error: {0}")]
    Validation(String),
}
```

## Event-Driven Architecture

### Event Bus

```rust
pub enum EngineEvent {
    OrderCreated { order_id: OrderId },
    PointsEarned { card_id: CardId, points: Decimal },
    PointsRedeemed { card_id: CardId, points: Decimal },
    MembershipChanged { partner_id: PartnerId, new_level: MembershipProductId },
    PromotionApplied { order_id: OrderId, program_id: ProgramId },
}

pub struct EventBus {
    redis: Arc<Redis>,
}

impl EventBus {
    pub async fn publish(&self, event: EngineEvent) -> Result<()> {
        // Publish to Redis pub/sub
        self.redis.publish("engine:events", event).await
    }
    
    pub async fn subscribe(&self) -> Result<EventStream> {
        // Subscribe to Redis pub/sub
        self.redis.subscribe("engine:events").await
    }
}
```

## Caching Strategy

### Cache Layer

```rust
pub struct Cache {
    redis: Arc<Redis>,
}

impl Cache {
    pub async fn get_card_balance(
        &self,
        card_id: CardId,
    ) -> Result<Option<Decimal>> {
        let key = format!("balance:card:{}", card_id);
        self.redis.get(&key).await
    }
    
    pub async fn set_card_balance(
        &self,
        card_id: CardId,
        balance: Decimal,
        ttl: Duration,
    ) -> Result<()> {
        let key = format!("balance:card:{}", card_id);
        self.redis.setex(&key, balance, ttl.as_secs()).await
    }
    
    pub async fn invalidate_card_balance(
        &self,
        card_id: CardId,
    ) -> Result<()> {
        let key = format!("balance:card:{}", card_id);
        self.redis.del(&key).await
    }
}
```

## Testing Strategy

### Unit Tests

```rust
#[cfg(test)]
mod tests {
    use super::*;
    
    #[tokio::test]
    async fn test_add_points() {
        // Test point addition
    }
    
    #[tokio::test]
    async fn test_deduct_points_fefo() {
        // Test FEFO consumption
    }
}
```

### Integration Tests

```rust
#[tokio::test]
async fn test_order_processing() {
    // Test full order processing flow
}
```

### Property Tests

```rust
use proptest::prelude::*;

proptest! {
    #[test]
    fn test_point_balance_consistency(
        points in 0.0..10000.0,
        earns in 0.0..1000.0,
        redeems in 0.0..1000.0,
    ) {
        // Test that balance is always consistent
    }
}
```

## Performance Optimizations

1. **Connection Pooling**: Reuse database connections
2. **Query Optimization**: Use indexes, prepared statements
3. **Caching**: Cache frequently accessed data
4. **Batch Processing**: Batch database operations
5. **Async I/O**: Non-blocking I/O operations

## Queue System

Queue system menggunakan PostgreSQL NOTIFY/LISTEN untuk async processing:
- Event-driven architecture
- Worker pool untuk parallel processing
- Retry mechanisms
- Dead letter queue

Lihat detail di: `07-queue-architecture.md`

Lihat dokumentasi berikutnya:
- `03-database-architecture.md` - Database architecture
- `04-integration-flow.md` - Integration flow
- `05-reporting-architecture.md` - Reporting architecture
- `07-queue-architecture.md` - Queue architecture
