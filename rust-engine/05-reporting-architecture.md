# Rust Engine - Reporting Architecture

## Overview

Arsitektur reporting untuk Rust engine dirancang untuk memberikan insights detail tentang promosi, membership, dan point transactions dengan performa tinggi dan real-time capabilities.

## Reporting Architecture

```
┌─────────────────────────────────────────────────────────┐
│              Data Sources                                │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐ │
│  │ Rust Engine  │  │ Odoo Database │  │ External     │ │
│  │ Database     │  │               │  │ Systems      │ │
│  └──────────────┘  └──────────────┘  └──────────────┘ │
└────────────────────┬────────────────────────────────────┘
                     │
        ┌────────────▼────────────┐
        │   ETL Pipeline           │
        │   - Extract              │
        │   - Transform            │
        │   - Load                 │
        └────────────┬────────────┘
                     │
        ┌────────────▼────────────┐
        │   Reporting Database     │
        │   (ClickHouse/PostgreSQL)│
        │   - Aggregated data      │
        │   - Historical data      │
        │   - Materialized views   │
        └────────────┬────────────┘
                     │
        ┌────────────▼────────────┐
        │   Reporting API          │
        │   - REST API             │
        │   - GraphQL API          │
        │   - gRPC API             │
        └────────────┬────────────┘
                     │
        ┌────────────▼────────────┐
        │   Reporting Frontend     │
        │   - Dashboard            │
        │   - Analytics            │
        │   - Reports              │
        └─────────────────────────┘
```

## Reporting Database Options

### Option 1: ClickHouse (Recommended untuk Analytics)

**Benefits**:
- **High Performance**: Columnar storage, optimized for analytics
- **Scalability**: Horizontal scaling
- **Real-time**: Fast aggregations
- **Compression**: Efficient storage

**Use Case**: Large-scale analytics, time-series data

### Option 2: PostgreSQL (Recommended untuk Operational Reports)

**Benefits**:
- **ACID**: Transactional consistency
- **Familiar**: Same as main database
- **Flexibility**: Rich SQL features
- **Integration**: Easy integration with Odoo

**Use Case**: Operational reports, detailed reports

### Option 3: Hybrid (Recommended)

**Architecture**:
```
┌─────────────────┐
│ ClickHouse       │
│ - Analytics      │
│ - Aggregations   │
│ - Time-series    │
└────────┬─────────┘
         │
┌────────▼─────────┐
│ PostgreSQL        │
│ - Detailed data   │
│ - Transactions    │
│ - Reference data  │
└──────────────────┘
```

## Data Pipeline Architecture

### ETL Pipeline

```
┌─────────────────────────────────────────────────────────┐
│                    Extract Layer                         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐ │
│  │ Rust DB      │  │ Odoo DB      │  │ Event Stream │ │
│  │ Connector    │  │ Connector    │  │ Connector    │ │
│  └──────────────┘  └──────────────┘  └──────────────┘ │
└────────────────────┬────────────────────────────────────┘
                     │
        ┌────────────▼────────────┐
        │   Transform Layer       │
        │   - Data cleaning      │
        │   - Aggregation         │
        │   - Enrichment          │
        └────────────┬────────────┘
                     │
        ┌────────────▼────────────┐
        │   Load Layer             │
        │   - Batch load           │
        │   - Stream load           │
        │   - Incremental load     │
        └─────────────────────────┘
```

### Real-time Streaming

```
┌─────────────────┐
│ Rust Engine      │
│ Events           │
└────────┬────────┘
         │
┌────────▼─────────┐
│ Kafka/Redis      │
│ Stream           │
└────────┬─────────┘
         │
┌────────▼─────────┐
│ Stream Processor │
│ - Aggregate      │
│ - Transform      │
└────────┬─────────┘
         │
┌────────▼─────────┐
│ Reporting DB     │
│ - Real-time      │
│ - Materialized   │
└──────────────────┘
```

## Reporting Schema Design

### ClickHouse Schema (Analytics)

```sql
-- Promotion Performance
CREATE TABLE promotion_performance (
    program_id UInt64,
    program_name String,
    program_type String,
    date Date,
    hour DateTime,
    total_transactions UInt32,
    total_qty UInt32,
    total_amount Decimal(18, 2),
    total_discount Decimal(18, 2),
    avg_transaction_value Decimal(18, 2),
    unique_customers UInt32,
    conversion_rate Float32
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(date)
ORDER BY (program_id, date, hour);

-- Membership Analytics
CREATE TABLE membership_analytics (
    membership_product_id UInt64,
    membership_level String,
    date Date,
    new_members UInt32,
    active_members UInt32,
    upgraded_members UInt32,
    downgraded_members UInt32,
    total_spending Decimal(18, 2),
    avg_spending Decimal(18, 2),
    total_points_earned Decimal(18, 2),
    total_points_redeemed Decimal(18, 2)
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(date)
ORDER BY (membership_product_id, date);

-- Point Transaction Analytics
CREATE TABLE point_analytics (
    card_id UInt64,
    partner_id UInt64,
    date Date,
    hour DateTime,
    points_earned Decimal(18, 2),
    points_redeemed Decimal(18, 2),
    points_expired Decimal(18, 2),
    balance Decimal(18, 2),
    transaction_type String
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(date)
ORDER BY (partner_id, date, hour);
```

### PostgreSQL Schema (Operational)

```sql
-- Detailed Promotion Report
CREATE TABLE promotion_report_detail (
    id BIGSERIAL PRIMARY KEY,
    program_id BIGINT NOT NULL,
    order_id VARCHAR(255) NOT NULL,
    partner_id BIGINT NOT NULL,
    transaction_date TIMESTAMP NOT NULL,
    product_id BIGINT,
    qty DECIMAL(18, 2),
    amount DECIMAL(18, 2),
    discount DECIMAL(18, 2),
    points_earned DECIMAL(18, 2),
    points_redeemed DECIMAL(18, 2),
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_promotion_report_program_date 
ON promotion_report_detail(program_id, transaction_date);

CREATE INDEX idx_promotion_report_partner 
ON promotion_report_detail(partner_id, transaction_date);

-- Materialized Views for Performance
CREATE MATERIALIZED VIEW mv_promotion_daily_summary AS
SELECT
    program_id,
    DATE(transaction_date) as report_date,
    COUNT(DISTINCT order_id) as total_orders,
    COUNT(DISTINCT partner_id) as unique_customers,
    SUM(qty) as total_qty,
    SUM(amount) as total_amount,
    SUM(discount) as total_discount,
    SUM(points_earned) as total_points_earned,
    SUM(points_redeemed) as total_points_redeemed
FROM promotion_report_detail
GROUP BY program_id, DATE(transaction_date);

CREATE UNIQUE INDEX ON mv_promotion_daily_summary(program_id, report_date);
```

## Reporting API Design

### REST API

```rust
// Promotion Performance
GET /api/v1/reports/promotions/performance
Query Params:
  - program_id: optional
  - date_from: required
  - date_to: required
  - group_by: hour|day|week|month

Response:
{
    "data": [
        {
            "program_id": 123,
            "program_name": "Summer Sale",
            "date": "2024-01-15",
            "total_transactions": 150,
            "total_amount": 150000.00,
            "total_discount": 15000.00,
            "avg_transaction_value": 1000.00,
            "conversion_rate": 0.15
        }
    ],
    "pagination": {
        "page": 1,
        "per_page": 100,
        "total": 150
    }
}

// Membership Analytics
GET /api/v1/reports/membership/analytics
Query Params:
  - membership_product_id: optional
  - date_from: required
  - date_to: required

Response:
{
    "data": [
        {
            "membership_level": "Gold",
            "date": "2024-01-15",
            "new_members": 50,
            "active_members": 500,
            "total_spending": 500000.00,
            "avg_spending": 1000.00,
            "total_points_earned": 50000.00
        }
    ]
}

// Point Transaction Report
GET /api/v1/reports/points/transactions
Query Params:
  - partner_id: optional
  - card_id: optional
  - date_from: required
  - date_to: required
  - transaction_type: earn|redeem|expire

Response:
{
    "data": [
        {
            "card_id": 123,
            "partner_id": 456,
            "transaction_date": "2024-01-15T10:30:00Z",
            "transaction_type": "earn",
            "points_amount": 100.00,
            "balance_after": 1100.00,
            "origin": "POS-ORDER-12345"
        }
    ]
}
```

### GraphQL API

```graphql
type Query {
    promotionPerformance(
        programId: ID
        dateFrom: DateTime!
        dateTo: DateTime!
        groupBy: GroupBy!
    ): [PromotionPerformance!]!
    
    membershipAnalytics(
        membershipProductId: ID
        dateFrom: DateTime!
        dateTo: DateTime!
    ): [MembershipAnalytics!]!
    
    pointTransactions(
        partnerId: ID
        cardId: ID
        dateFrom: DateTime!
        dateTo: DateTime!
        transactionType: TransactionType
    ): [PointTransaction!]!
}

type PromotionPerformance {
    programId: ID!
    programName: String!
    date: Date!
    totalTransactions: Int!
    totalAmount: Decimal!
    totalDiscount: Decimal!
    avgTransactionValue: Decimal!
    conversionRate: Float!
}

type MembershipAnalytics {
    membershipLevel: String!
    date: Date!
    newMembers: Int!
    activeMembers: Int!
    totalSpending: Decimal!
    avgSpending: Decimal!
    totalPointsEarned: Decimal!
}
```

## Real-time Reporting

### Event-Driven Updates

```rust
pub struct ReportingService {
    event_bus: Arc<EventBus>,
    clickhouse: Arc<ClickHouseClient>,
    postgres: Arc<PostgresClient>,
}

impl ReportingService {
    pub async fn handle_order_event(&self, event: OrderProcessedEvent) -> Result<()> {
        // Update real-time aggregations
        self.update_promotion_performance(&event).await?;
        self.update_membership_analytics(&event).await?;
        self.update_point_analytics(&event).await?;
        
        Ok(())
    }
    
    async fn update_promotion_performance(&self, event: &OrderProcessedEvent) -> Result<()> {
        // Incremental update to ClickHouse
        self.clickhouse.execute(
            "INSERT INTO promotion_performance VALUES",
            &event.to_promotion_performance_row()
        ).await?;
        
        Ok(())
    }
}
```

### Materialized Views Refresh

```rust
pub struct MaterializedViewRefresher {
    postgres: Arc<PostgresClient>,
}

impl MaterializedViewRefresher {
    pub async fn refresh_daily_summary(&self) -> Result<()> {
        // Refresh materialized view
        self.postgres.execute(
            "REFRESH MATERIALIZED VIEW CONCURRENTLY mv_promotion_daily_summary"
        ).await?;
        
        Ok(())
    }
    
    pub async fn schedule_refresh(&self) -> Result<()> {
        // Refresh every hour
        let mut interval = tokio::time::interval(Duration::from_secs(3600));
        
        loop {
            interval.tick().await;
            self.refresh_daily_summary().await?;
        }
    }
}
```

## Reporting Queries Examples

### Promotion Performance Query

```sql
-- Daily promotion performance
SELECT
    program_id,
    program_name,
    DATE(transaction_date) as report_date,
    COUNT(DISTINCT order_id) as total_orders,
    COUNT(DISTINCT partner_id) as unique_customers,
    SUM(amount) as total_amount,
    SUM(discount) as total_discount,
    AVG(amount) as avg_transaction_value,
    SUM(points_earned) as total_points_earned
FROM promotion_report_detail
WHERE transaction_date >= :date_from
  AND transaction_date <= :date_to
  AND (:program_id IS NULL OR program_id = :program_id)
GROUP BY program_id, program_name, DATE(transaction_date)
ORDER BY report_date DESC, total_amount DESC;
```

### Membership Analytics Query

```sql
-- Membership level performance
SELECT
    ml.membership_level,
    DATE(po.transaction_date) as report_date,
    COUNT(DISTINCT CASE WHEN po.is_first_order THEN po.partner_id END) as new_members,
    COUNT(DISTINCT po.partner_id) as active_members,
    SUM(po.amount) as total_spending,
    AVG(po.amount) as avg_spending,
    SUM(po.points_earned) as total_points_earned,
    SUM(po.points_redeemed) as total_points_redeemed
FROM point_analytics po
JOIN membership_levels ml ON po.membership_product_id = ml.membership_product_id
WHERE po.transaction_date >= :date_from
  AND po.transaction_date <= :date_to
GROUP BY ml.membership_level, DATE(po.transaction_date)
ORDER BY report_date DESC, total_spending DESC;
```

### Point Transaction Analysis

```sql
-- Point transaction trends
SELECT
    DATE(transaction_date) as report_date,
    transaction_type,
    SUM(points_amount) as total_points,
    COUNT(*) as transaction_count,
    AVG(points_amount) as avg_points
FROM point_analytics
WHERE transaction_date >= :date_from
  AND transaction_date <= :date_to
  AND (:partner_id IS NULL OR partner_id = :partner_id)
GROUP BY DATE(transaction_date), transaction_type
ORDER BY report_date DESC, transaction_type;
```

## Caching Strategy untuk Reporting

### Cache Layer

```rust
pub struct ReportingCache {
    redis: Arc<Redis>,
    ttl: Duration,
}

impl ReportingCache {
    pub async fn get_promotion_performance(
        &self,
        params: &PromotionPerformanceParams,
    ) -> Result<Option<PromotionPerformance>> {
        let key = format!("report:promotion:{}", params.cache_key());
        
        // Try cache
        if let Some(cached) = self.redis.get(&key).await? {
            return Ok(Some(cached));
        }
        
        // Query database
        let result = self.query_promotion_performance(params).await?;
        
        // Cache result
        self.redis.setex(&key, &result, self.ttl.as_secs()).await?;
        
        Ok(Some(result))
    }
}
```

## Export Functionality

### CSV Export

```rust
pub async fn export_promotion_report(
    &self,
    params: &PromotionReportParams,
) -> Result<Vec<u8>> {
    let data = self.get_promotion_data(params).await?;
    
    let mut wtr = csv::Writer::from_writer(vec![]);
    
    for row in data {
        wtr.serialize(row)?;
    }
    
    Ok(wtr.into_inner()?)
}
```

### Excel Export

```rust
pub async fn export_excel_report(
    &self,
    params: &ReportParams,
) -> Result<Vec<u8>> {
    use calamine::{Workbook, Writer};
    
    let data = self.get_report_data(params).await?;
    
    let mut workbook = Workbook::new();
    let mut sheet = workbook.add_sheet("Report");
    
    // Write headers
    // Write data
    
    let mut buf = Vec::new();
    workbook.write(&mut buf)?;
    
    Ok(buf)
}
```

## Dashboard Integration

### Real-time Dashboard

```rust
// WebSocket for real-time updates
pub struct DashboardService {
    ws_clients: Arc<DashMap<String, WsClient>>,
    event_bus: Arc<EventBus>,
}

impl DashboardService {
    pub async fn broadcast_update(&self, update: DashboardUpdate) -> Result<()> {
        let message = serde_json::to_string(&update)?;
        
        for client in self.ws_clients.iter() {
            client.send(message.clone()).await?;
        }
        
        Ok(())
    }
}
```

## Performance Optimization

### Query Optimization

1. **Indexes**: Strategic indexes untuk common queries
2. **Partitioning**: Partition by date untuk time-series data
3. **Materialized Views**: Pre-aggregated data
4. **Query Caching**: Cache frequent queries

### Data Retention

```rust
pub struct DataRetentionPolicy {
    detailed_data_days: u32,      // 90 days
    aggregated_data_days: u32,    // 2 years
    archive_data_days: u32,       // 5 years
}

impl DataRetentionPolicy {
    pub async fn cleanup_old_data(&self) -> Result<()> {
        // Archive old detailed data
        // Keep aggregated data
        // Compress archived data
    }
}
```

Lihat dokumentasi berikutnya:
- [06-integration.md](06-integration.md) - Detail integrasi dengan Odoo
- `../migration/` - Rencana migrasi reporting
