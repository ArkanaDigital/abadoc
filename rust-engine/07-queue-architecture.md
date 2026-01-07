# Rust Engine - Queue Architecture dengan PostgreSQL NOTIFY

## Overview

Rust engine menggunakan PostgreSQL NOTIFY/LISTEN untuk implementasi queue system yang terintegrasi dengan database. Ini memberikan reliability, consistency, dan tidak memerlukan external queue system.

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│              Rust Engine Components                       │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │ Order        │  │ Point        │  │ Sync        │  │
│  │ Service      │  │ Service      │  │ Service     │  │
│  └──────┬───────┘  └──────┬───────┘  └──────┬──────┘  │
│         │                 │                 │          │
│         └─────────┬───────┴─────────────────┘          │
│                   │                                     │
│         ┌─────────▼─────────┐                          │
│         │   Queue Publisher │                          │
│         │   (NOTIFY)        │                          │
│         └─────────┬─────────┘                          │
└───────────────────┼───────────────────────────────────┘
                    │
                    │ PostgreSQL NOTIFY
                    │
        ┌───────────▼───────────┐
        │   PostgreSQL Database  │
        │   - Queue Channels      │
        │   - Event Storage       │
        └───────────┬───────────┘
                    │
                    │ PostgreSQL LISTEN
                    │
        ┌───────────▼───────────┐
        │   Queue Workers        │
        │   - Order Worker       │
        │   - Point Worker       │
        │   - Sync Worker        │
        └───────────────────────┘
```

## PostgreSQL NOTIFY/LISTEN

### Concept

PostgreSQL NOTIFY/LISTEN adalah built-in pub/sub mechanism:
- **NOTIFY**: Publish event ke channel
- **LISTEN**: Subscribe ke channel
- **Atomic**: Terintegrasi dengan transactions
- **Reliable**: Guaranteed delivery dalam transaction

### Advantages

1. **No External Dependency**: Tidak perlu Redis/RabbitMQ
2. **ACID Guarantees**: Events hanya delivered jika transaction committed
3. **Simple**: Built-in PostgreSQL feature
4. **Efficient**: Low overhead
5. **Reliable**: Transaction-based delivery

## Queue Channels Design

### Channel Structure

```sql
-- Channel naming convention: queue.{entity_type}.{action}
-- Examples:
-- queue.order.process
-- queue.point.earn
-- queue.point.redeem
-- queue.sync.odoo
-- queue.report.generate
```

### Channel Categories

1. **Order Processing**: `queue.order.*`
2. **Point Operations**: `queue.point.*`
3. **Sync Operations**: `queue.sync.*`
4. **Reporting**: `queue.report.*`
5. **Membership**: `queue.membership.*`

## Implementation

### Queue Publisher

```rust
use sqlx::PgPool;
use serde::Serialize;

pub struct QueuePublisher {
    pool: Arc<PgPool>,
}

impl QueuePublisher {
    pub async fn publish<T: Serialize>(
        &self,
        channel: &str,
        payload: &T,
    ) -> Result<()> {
        let payload_json = serde_json::to_string(payload)?;
        
        // Publish via NOTIFY
        sqlx::query(
            "SELECT pg_notify($1, $2)"
        )
        .bind(channel)
        .bind(&payload_json)
        .execute(&*self.pool)
        .await?;
        
        Ok(())
    }
    
    pub async fn publish_order_processed(&self, order_id: OrderId) -> Result<()> {
        let payload = OrderProcessedEvent {
            order_id,
            timestamp: Utc::now(),
        };
        
        self.publish("queue.order.processed", &payload).await
    }
    
    pub async fn publish_point_earned(
        &self,
        card_id: CardId,
        points: Decimal,
    ) -> Result<()> {
        let payload = PointEarnedEvent {
            card_id,
            points,
            timestamp: Utc::now(),
        };
        
        self.publish("queue.point.earn", &payload).await
    }
    
    pub async fn publish_sync_required(
        &self,
        entity_type: &str,
        entity_id: i64,
    ) -> Result<()> {
        let payload = SyncRequiredEvent {
            entity_type: entity_type.to_string(),
            entity_id,
            timestamp: Utc::now(),
        };
        
        self.publish("queue.sync.required", &payload).await
    }
}
```

### Queue Subscriber (Worker)

```rust
use tokio_postgres::{Client, Notification};
use futures::StreamExt;

pub struct QueueSubscriber {
    client: Client,
}

impl QueueSubscriber {
    pub async fn subscribe(&self, channel: &str) -> Result<impl Stream<Item = Notification>> {
        // Connect to database
        let (client, connection) = tokio_postgres::connect(
            &self.connection_string,
            NoTls,
        ).await?;
        
        // Spawn connection handler
        tokio::spawn(async move {
            if let Err(e) = connection.await {
                eprintln!("Connection error: {}", e);
            }
        });
        
        // Listen to channel
        client.execute(
            &format!("LISTEN {}", channel),
            &[],
        ).await?;
        
        // Get notifications stream
        let notifications = client.notifications();
        
        Ok(notifications)
    }
    
    pub async fn start_worker(&self, channel: &str, handler: impl Fn(Notification)) {
        let mut stream = self.subscribe(channel).await?;
        
        while let Some(notification) = stream.next().await {
            handler(notification);
        }
    }
}
```

### Queue Worker Implementation

```rust
pub struct OrderWorker {
    subscriber: Arc<QueueSubscriber>,
    order_service: Arc<OrderService>,
}

impl OrderWorker {
    pub async fn start(&self) -> Result<()> {
        let mut stream = self.subscriber.subscribe("queue.order.processed").await?;
        
        while let Some(notification) = stream.next().await {
            let payload: OrderProcessedEvent = serde_json::from_str(
                &notification.payload()
            )?;
            
            // Process order
            self.order_service.process_order_async(payload.order_id).await?;
        }
        
        Ok(())
    }
}

pub struct PointWorker {
    subscriber: Arc<QueueSubscriber>,
    point_service: Arc<PointService>,
}

impl PointWorker {
    pub async fn start(&self) -> Result<()> {
        // Subscribe to point channels
        let mut earn_stream = self.subscriber.subscribe("queue.point.earn").await?;
        let mut redeem_stream = self.subscriber.subscribe("queue.point.redeem").await?;
        
        tokio::select! {
            Some(notification) = earn_stream.next() => {
                let payload: PointEarnedEvent = serde_json::from_str(
                    &notification.payload()
                )?;
                self.point_service.earn_points_async(
                    payload.card_id,
                    payload.points,
                ).await?;
            }
            Some(notification) = redeem_stream.next() => {
                let payload: PointRedeemedEvent = serde_json::from_str(
                    &notification.payload()
                )?;
                self.point_service.redeem_points_async(
                    payload.card_id,
                    payload.points,
                ).await?;
            }
        }
        
        Ok(())
    }
}

pub struct SyncWorker {
    subscriber: Arc<QueueSubscriber>,
    sync_service: Arc<SyncService>,
}

impl SyncWorker {
    pub async fn start(&self) -> Result<()> {
        let mut stream = self.subscriber.subscribe("queue.sync.required").await?;
        
        while let Some(notification) = stream.next().await {
            let payload: SyncRequiredEvent = serde_json::from_str(
                &notification.payload()
            )?;
            
            // Sync to Odoo
            self.sync_service.sync_entity(
                &payload.entity_type,
                payload.entity_id,
            ).await?;
        }
        
        Ok(())
    }
}
```

## Event Payload Design

### Event Types

```rust
#[derive(Serialize, Deserialize)]
pub struct OrderProcessedEvent {
    pub order_id: OrderId,
    pub timestamp: DateTime<Utc>,
}

#[derive(Serialize, Deserialize)]
pub struct PointEarnedEvent {
    pub card_id: CardId,
    pub points: Decimal,
    pub origin: String,
    pub timestamp: DateTime<Utc>,
}

#[derive(Serialize, Deserialize)]
pub struct PointRedeemedEvent {
    pub card_id: CardId,
    pub points: Decimal,
    pub origin: String,
    pub timestamp: DateTime<Utc>,
}

#[derive(Serialize, Deserialize)]
pub struct SyncRequiredEvent {
    pub entity_type: String,
    pub entity_id: i64,
    pub priority: SyncPriority,
    pub timestamp: DateTime<Utc>,
}

#[derive(Serialize, Deserialize)]
pub enum SyncPriority {
    High,    // Immediate sync
    Medium,  // Batch sync
    Low,     // Scheduled sync
}
```

## Queue Table untuk Persistence

### Queue Table Schema

```sql
-- Queue table untuk persistent storage
CREATE TABLE queue_jobs (
    id BIGSERIAL PRIMARY KEY,
    channel VARCHAR(255) NOT NULL,
    payload JSONB NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'pending', -- pending, processing, completed, failed
    priority INTEGER DEFAULT 0,
    retry_count INTEGER DEFAULT 0,
    max_retries INTEGER DEFAULT 3,
    error_message TEXT,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    processed_at TIMESTAMP,
    completed_at TIMESTAMP
);

CREATE INDEX idx_queue_jobs_status ON queue_jobs(status, priority DESC, created_at);
CREATE INDEX idx_queue_jobs_channel ON queue_jobs(channel, status);
```

### Hybrid Approach: NOTIFY + Table

```rust
pub struct HybridQueue {
    publisher: Arc<QueuePublisher>,
    pool: Arc<PgPool>,
}

impl HybridQueue {
    pub async fn enqueue<T: Serialize>(
        &self,
        channel: &str,
        payload: &T,
        priority: i32,
    ) -> Result<JobId> {
        // 1. Insert to queue table
        let job_id = sqlx::query!(
            r#"
            INSERT INTO queue_jobs (channel, payload, priority, status)
            VALUES ($1, $2, $3, 'pending')
            RETURNING id
            "#,
            channel,
            serde_json::to_value(payload)?,
            priority
        )
        .fetch_one(&*self.pool)
        .await?
        .id;
        
        // 2. NOTIFY untuk immediate processing
        self.publisher.publish(channel, payload).await?;
        
        Ok(job_id)
    }
    
    pub async fn process_pending_jobs(&self) -> Result<()> {
        // Process jobs from table (for retry, persistence)
        let jobs = sqlx::query!(
            r#"
            SELECT id, channel, payload, retry_count, max_retries
            FROM queue_jobs
            WHERE status = 'pending'
            ORDER BY priority DESC, created_at ASC
            LIMIT 100
            FOR UPDATE SKIP LOCKED
            "#
        )
        .fetch_all(&*self.pool)
        .await?;
        
        for job in jobs {
            match self.process_job(&job).await {
                Ok(_) => {
                    sqlx::query!(
                        "UPDATE queue_jobs SET status = 'completed', completed_at = NOW() WHERE id = $1",
                        job.id
                    )
                    .execute(&*self.pool)
                    .await?;
                }
                Err(e) => {
                    let retry_count = job.retry_count + 1;
                    if retry_count >= job.max_retries {
                        sqlx::query!(
                            "UPDATE queue_jobs SET status = 'failed', error_message = $1 WHERE id = $2",
                            e.to_string(),
                            job.id
                        )
                        .execute(&*self.pool)
                        .await?;
                    } else {
                        sqlx::query!(
                            "UPDATE queue_jobs SET status = 'pending', retry_count = $1 WHERE id = $2",
                            retry_count,
                            job.id
                        )
                        .execute(&*self.pool)
                        .await?;
                    }
                }
            }
        }
        
        Ok(())
    }
}
```

## Worker Pool Architecture

```
┌─────────────────────────────────────────────────────────┐
│              Worker Pool Manager                         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐ │
│  │ Order Worker │  │ Point Worker │  │ Sync Worker  │ │
│  │ (5 instances)│  │ (3 instances)│  │ (2 instances)│ │
│  └──────────────┘  └──────────────┘  └──────────────┘ │
│  ┌──────────────┐  ┌──────────────┐                   │
│  │ Report Worker│  │ Membership   │                   │
│  │ (2 instances)│  │ Worker       │                   │
│  │              │  │ (2 instances)│                   │
│  └──────────────┘  └──────────────┘                   │
└─────────────────────────────────────────────────────────┘
```

### Worker Manager

```rust
pub struct WorkerManager {
    workers: Vec<Arc<dyn Worker>>,
}

impl WorkerManager {
    pub async fn start_all(&self) -> Result<()> {
        let mut handles = Vec::new();
        
        for worker in &self.workers {
            let worker = worker.clone();
            let handle = tokio::spawn(async move {
                worker.start().await
            });
            handles.push(handle);
        }
        
        // Wait for all workers
        futures::future::join_all(handles).await;
        
        Ok(())
    }
    
    pub fn add_worker(&mut self, worker: Arc<dyn Worker>) {
        self.workers.push(worker);
    }
}
```

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
    pub async fn execute_with_retry<F, T>(
        &self,
        mut f: F,
    ) -> Result<T>
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
    pool: Arc<PgPool>,
}

impl DeadLetterQueue {
    pub async fn move_to_dlq(&self, job_id: JobId, error: &str) -> Result<()> {
        sqlx::query!(
            r#"
            INSERT INTO queue_dlq (job_id, channel, payload, error_message, created_at)
            SELECT id, channel, payload, $1, NOW()
            FROM queue_jobs
            WHERE id = $2
            "#,
            error,
            job_id
        )
        .execute(&*self.pool)
        .await?;
        
        Ok(())
    }
}
```

## Monitoring & Metrics

### Queue Metrics

```rust
pub struct QueueMetrics {
    pool: Arc<PgPool>,
}

impl QueueMetrics {
    pub async fn get_queue_stats(&self) -> Result<QueueStats> {
        let stats = sqlx::query!(
            r#"
            SELECT 
                status,
                COUNT(*) as count,
                AVG(EXTRACT(EPOCH FROM (processed_at - created_at))) as avg_processing_time
            FROM queue_jobs
            WHERE created_at > NOW() - INTERVAL '1 hour'
            GROUP BY status
            "#
        )
        .fetch_all(&*self.pool)
        .await?;
        
        Ok(QueueStats {
            pending: stats.iter().find(|s| s.status == "pending").map(|s| s.count as u64).unwrap_or(0),
            processing: stats.iter().find(|s| s.status == "processing").map(|s| s.count as u64).unwrap_or(0),
            completed: stats.iter().find(|s| s.status == "completed").map(|s| s.count as u64).unwrap_or(0),
            failed: stats.iter().find(|s| s.status == "failed").map(|s| s.count as u64).unwrap_or(0),
        })
    }
}
```

## Performance Optimization

### Connection Pooling

```rust
use sqlx::postgres::PgPoolOptions;

let pool = PgPoolOptions::new()
    .max_connections(50)
    .acquire_timeout(Duration::from_secs(5))
    .connect(&database_url)
    .await?;
```

### Batch Processing

```rust
impl QueueWorker {
    pub async fn process_batch(&self, batch_size: usize) -> Result<()> {
        let jobs = self.get_pending_jobs(batch_size).await?;
        
        // Process in parallel
        let results = futures::future::join_all(
            jobs.iter().map(|job| self.process_job(job))
        ).await;
        
        // Update status
        self.update_job_statuses(results).await?;
        
        Ok(())
    }
}
```

## Integration dengan Existing Services

### Order Service Integration

```rust
impl OrderService {
    pub async fn process_order(&self, order: Order) -> Result<ProcessedOrder> {
        // Process order
        let processed = self.process_order_internal(&order).await?;
        
        // Publish event
        self.queue_publisher.publish_order_processed(processed.id).await?;
        
        Ok(processed)
    }
}
```

### Point Service Integration

```rust
impl PointService {
    pub async fn add_points(&self, card_id: CardId, points: Decimal) -> Result<()> {
        // Add points
        self.add_points_internal(card_id, points).await?;
        
        // Publish event
        self.queue_publisher.publish_point_earned(card_id, points).await?;
        
        // Trigger sync
        self.queue_publisher.publish_sync_required("point_history", history_id).await?;
        
        Ok(())
    }
}
```

## Benefits

1. **Reliability**: Transaction-based delivery
2. **Consistency**: ACID guarantees
3. **Simplicity**: No external dependencies
4. **Performance**: Low overhead
5. **Scalability**: Can scale workers independently

## Limitations & Considerations

1. **Message Size**: NOTIFY payload limited to 8000 bytes
2. **No Persistence**: NOTIFY only (need table for persistence)
3. **No Guaranteed Order**: Messages bisa arrive out of order
4. **Connection Required**: Need persistent connection untuk LISTEN

## Solutions

1. **Hybrid Approach**: Use table untuk persistence, NOTIFY untuk immediate processing
2. **Payload Compression**: Compress large payloads
3. **Sequence Numbers**: Add sequence untuk ensure order
4. **Connection Pool**: Maintain connection pool untuk workers

## Integration dengan Services

### Order Processing dengan Queue

```rust
impl OrderService {
    pub async fn process_order(&self, order: Order) -> Result<ProcessedOrder> {
        // 1. Process order synchronously (critical path)
        let processed = self.process_order_internal(&order).await?;
        
        // 2. Publish async events via queue
        self.queue_publisher.publish_order_processed(processed.id).await?;
        self.queue_publisher.publish_sync_required("order", processed.id).await?;
        
        Ok(processed)
    }
}
```

### Point Processing dengan Queue

```rust
impl PointService {
    pub async fn add_points(&self, card_id: CardId, points: Decimal) -> Result<()> {
        // 1. Add points synchronously
        let history = self.add_points_internal(card_id, points).await?;
        
        // 2. Publish events
        self.queue_publisher.publish_point_earned(card_id, points).await?;
        self.queue_publisher.publish_sync_required("point_history", history.id).await?;
        
        Ok(())
    }
}
```

## Worker Startup

```rust
#[tokio::main]
async fn main() -> Result<()> {
    let pool = create_db_pool().await?;
    let publisher = Arc::new(QueuePublisher::new(pool.clone()));
    let subscriber = Arc::new(QueueSubscriber::new(pool.clone()));
    
    // Create workers
    let order_worker = Arc::new(OrderWorker::new(subscriber.clone(), order_service));
    let point_worker = Arc::new(PointWorker::new(subscriber.clone(), point_service));
    let sync_worker = Arc::new(SyncWorker::new(subscriber.clone(), sync_service));
    
    // Start workers
    tokio::join!(
        order_worker.start(),
        point_worker.start(),
        sync_worker.start(),
    );
    
    Ok(())
}
```

Lihat dokumentasi berikutnya:
- [04-integration-flow.md](04-integration-flow.md) - Integration flow dengan queue
- [03-database-architecture.md](03-database-architecture.md) - Database architecture
- [06-integration.md](06-integration.md) - Detail integration
