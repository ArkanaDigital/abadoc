# ETL Implementation Detail (Fase 2)

## Overview

Detail implementasi ETL paralel untuk sync data dari Odoo ke Rust database secara read-only.

## ETL Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    Odoo Database                         │
│  (PostgreSQL) - Source of Truth                          │
└────────────────────┬────────────────────────────────────┘
                     │
                     │ Read-Only Connection
                     │
        ┌────────────▼────────────┐
        │   ETL Service            │
        │   ┌──────────────────┐  │
        │   │ Extract          │  │
        │   │ - Query Odoo     │  │
        │   │ - Read data      │  │
        │   └────────┬─────────┘  │
        │            │             │
        │   ┌────────▼─────────┐  │
        │   │ Transform        │  │
        │   │ - Validate       │  │
        │   │ - Clean          │  │
        │   │ - Enrich         │  │
        │   └────────┬─────────┘  │
        │            │             │
        │   ┌────────▼─────────┐  │
        │   │ Load             │  │
        │   │ - Write Rust DB  │  │
        │   │ - Upsert logic   │  │
        │   └──────────────────┘  │
        └────────────┬────────────┘
                     │
        ┌────────────▼────────────┐
        │   Rust Engine Database   │
        │   (PostgreSQL)           │
        │   - Read-only replica    │
        └─────────────────────────┘
```

## Extract Layer Implementation

### Odoo Connection

```rust
use sqlx::postgres::PgPool;
use sqlx::Row;

pub struct OdooExtractor {
    pool: PgPool,
}

impl OdooExtractor {
    pub fn new(odoo_db_url: &str) -> Result<Self> {
        let pool = PgPool::connect(odoo_db_url).await?;
        Ok(Self { pool })
    }
    
    pub async fn extract_loyalty_cards(
        &self,
        last_sync: Option<DateTime<Utc>>,
    ) -> Result<Vec<OdooLoyaltyCard>> {
        let query = if let Some(sync_time) = last_sync {
            format!(
                r#"
                SELECT 
                    id, code, partner_id, program_id, points,
                    create_date, write_date
                FROM loyalty_card
                WHERE write_date > '{}'
                ORDER BY write_date ASC
                "#,
                sync_time.format("%Y-%m-%d %H:%M:%S")
            )
        } else {
            r#"
            SELECT 
                id, code, partner_id, program_id, points,
                create_date, write_date
            FROM loyalty_card
            ORDER BY write_date ASC
            "#
            .to_string()
        };
        
        let rows = sqlx::query(&query)
            .fetch_all(&self.pool)
            .await?;
        
        Ok(rows.into_iter().map(|row| {
            OdooLoyaltyCard {
                id: row.get("id"),
                code: row.get("code"),
                partner_id: row.get("partner_id"),
                program_id: row.get("program_id"),
                points: row.get("points"),
                create_date: row.get("create_date"),
                write_date: row.get("write_date"),
            }
        }).collect())
    }
    
    pub async fn extract_point_history(
        &self,
        last_sync: Option<DateTime<Utc>>,
    ) -> Result<Vec<OdooPointHistory>> {
        // Similar implementation
        let query = if let Some(sync_time) = last_sync {
            format!(
                r#"
                SELECT 
                    lph.id,
                    lph.loyalty_card_id,
                    lph.point_move_id,
                    lph.points_amount,
                    lph.points_balance,
                    lph.transaction_time,
                    lph.origin,
                    lph.membership_product_id,
                    lph.status_membership_product_id
                FROM loyalty_point_history lph
                WHERE lph.transaction_time > '{}'
                ORDER BY lph.transaction_time ASC
                "#,
                sync_time.format("%Y-%m-%d %H:%M:%S")
            )
        } else {
            r#"
            SELECT 
                lph.id,
                lph.loyalty_card_id,
                lph.point_move_id,
                lph.points_amount,
                lph.points_balance,
                lph.transaction_time,
                lph.origin,
                lph.membership_product_id,
                lph.status_membership_product_id
            FROM loyalty_point_history lph
            ORDER BY lph.transaction_time ASC
            "#
            .to_string()
        };
        
        // Execute and transform
        // ...
    }
    
    pub async fn extract_point_moves(
        &self,
        last_sync: Option<DateTime<Utc>>,
    ) -> Result<Vec<OdooPointMove>> {
        // Similar implementation
    }
}
```

### Batch Processing

```rust
impl OdooExtractor {
    pub async fn extract_loyalty_cards_batch(
        &self,
        last_sync: Option<DateTime<Utc>>,
        batch_size: usize,
    ) -> Result<impl Stream<Item = Result<Vec<OdooLoyaltyCard>>>> {
        let mut offset = 0;
        
        Ok(stream::unfold((), move |_| {
            let pool = self.pool.clone();
            let sync_time = last_sync;
            
            async move {
                let query = format!(
                    r#"
                    SELECT 
                        id, code, partner_id, program_id, points,
                        create_date, write_date
                    FROM loyalty_card
                    WHERE {}
                    ORDER BY write_date ASC
                    LIMIT {} OFFSET {}
                    "#,
                    if let Some(st) = sync_time {
                        format!("write_date > '{}'", st.format("%Y-%m-%d %H:%M:%S"))
                    } else {
                        "1=1".to_string()
                    },
                    batch_size,
                    offset
                );
                
                let rows = sqlx::query(&query)
                    .fetch_all(&pool)
                    .await?;
                
                if rows.is_empty() {
                    return None;
                }
                
                let cards: Vec<OdooLoyaltyCard> = rows.into_iter()
                    .map(|row| /* transform */)
                    .collect();
                
                offset += batch_size;
                
                Some((Ok(cards), ()))
            }
        }))
    }
}
```

## Transform Layer Implementation

### Data Transformation

```rust
pub struct DataTransformer {
    validator: Arc<DataValidator>,
}

impl DataTransformer {
    pub fn transform_loyalty_card(
        &self,
        odoo_card: OdooLoyaltyCard,
    ) -> Result<RustLoyaltyCard> {
        // Validate
        self.validator.validate_card(&odoo_card)?;
        
        // Transform
        Ok(RustLoyaltyCard {
            odoo_card_id: odoo_card.id,
            code: self.clean_code(&odoo_card.code)?,
            partner_id: odoo_card.partner_id,
            program_id: odoo_card.program_id,
            points: self.calculate_accurate_points(&odoo_card)?,
            created_at: odoo_card.create_date,
            updated_at: odoo_card.write_date,
            synced_to_odoo: false, // Not synced, this is from Odoo
            sync_version: 0,
        })
    }
    
    fn clean_code(&self, code: &str) -> Result<String> {
        let cleaned = code.trim().to_uppercase();
        if cleaned.is_empty() {
            return Err(Error::Validation("Card code cannot be empty"));
        }
        Ok(cleaned)
    }
    
    fn calculate_accurate_points(
        &self,
        card: &OdooLoyaltyCard,
    ) -> Result<Decimal> {
        // Calculate from point history instead of using card.points
        // This ensures accuracy
        // Implementation depends on having access to point history
        Ok(card.points) // For now, use card points
    }
}
```

### Data Validation

```rust
pub struct DataValidator;

impl DataValidator {
    pub fn validate_card(&self, card: &OdooLoyaltyCard) -> Result<()> {
        if card.code.is_empty() {
            return Err(Error::Validation("Card code cannot be empty"));
        }
        
        if card.partner_id == 0 {
            return Err(Error::Validation("Partner ID cannot be zero"));
        }
        
        if card.program_id == 0 {
            return Err(Error::Validation("Program ID cannot be zero"));
        }
        
        if card.points < 0.into() {
            return Err(Error::Validation("Points cannot be negative"));
        }
        
        Ok(())
    }
    
    pub fn validate_point_history(&self, history: &OdooPointHistory) -> Result<()> {
        if history.loyalty_card_id == 0 {
            return Err(Error::Validation("Loyalty card ID cannot be zero"));
        }
        
        if history.points_balance < 0.into() {
            return Err(Error::Validation("Points balance cannot be negative"));
        }
        
        Ok(())
    }
}
```

## Load Layer Implementation

### Upsert Logic

```rust
pub struct RustLoader {
    pool: PgPool,
}

impl RustLoader {
    pub async fn load_loyalty_cards(
        &self,
        cards: Vec<RustLoyaltyCard>,
    ) -> Result<LoadResult> {
        let mut tx = self.pool.begin().await?;
        let mut inserted = 0;
        let mut updated = 0;
        let mut errors = 0;
        
        for card in cards {
            match self.upsert_card(&mut tx, &card).await {
                Ok(UpsertResult::Inserted) => inserted += 1,
                Ok(UpsertResult::Updated) => updated += 1,
                Err(e) => {
                    errors += 1;
                    _logger.error!("Failed to load card {}: {}", card.odoo_card_id, e);
                }
            }
        }
        
        tx.commit().await?;
        
        Ok(LoadResult {
            total: cards.len(),
            inserted,
            updated,
            errors,
        })
    }
    
    async fn upsert_card(
        &self,
        tx: &mut Transaction<'_, Postgres>,
        card: &RustLoyaltyCard,
    ) -> Result<UpsertResult> {
        let result = sqlx::query!(
            r#"
            INSERT INTO loyalty_cards (
                odoo_card_id, code, partner_id, program_id,
                points, created_at, updated_at, synced_to_odoo, sync_version
            ) VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9)
            ON CONFLICT (odoo_card_id) 
            DO UPDATE SET
                code = EXCLUDED.code,
                points = EXCLUDED.points,
                updated_at = EXCLUDED.updated_at,
                sync_version = loyalty_cards.sync_version + 1
            RETURNING (xmax = 0) AS inserted
            "#,
            card.odoo_card_id,
            card.code,
            card.partner_id,
            card.program_id,
            card.points,
            card.created_at,
            card.updated_at,
            card.synced_to_odoo,
            card.sync_version
        )
        .fetch_one(&mut **tx)
        .await?;
        
        if result.inserted {
            Ok(UpsertResult::Inserted)
        } else {
            Ok(UpsertResult::Updated)
        }
    }
}
```

### Batch Loading

```rust
impl RustLoader {
    pub async fn load_batch(
        &self,
        items: Vec<LoadItem>,
        batch_size: usize,
    ) -> Result<LoadResult> {
        let mut total_result = LoadResult::default();
        
        for chunk in items.chunks(batch_size) {
            let result = match chunk[0].entity_type {
                EntityType::LoyaltyCard => {
                    let cards: Vec<_> = chunk.iter()
                        .map(|i| i.as_card().unwrap())
                        .collect();
                    self.load_loyalty_cards(cards).await?
                }
                EntityType::PointHistory => {
                    let histories: Vec<_> = chunk.iter()
                        .map(|i| i.as_history().unwrap())
                        .collect();
                    self.load_point_history(histories).await?
                }
                // ... other types
            };
            
            total_result.merge(result);
        }
        
        Ok(total_result)
    }
}
```

## ETL Orchestration

### Main ETL Service

```rust
pub struct ETLService {
    extractor: Arc<OdooExtractor>,
    transformer: Arc<DataTransformer>,
    loader: Arc<RustLoader>,
    sync_state: Arc<SyncState>,
    validator: Arc<DataValidator>,
}

impl ETLService {
    pub async fn run_full_sync(&self) -> Result<SyncResult> {
        _logger.info("Starting full sync");
        
        // Extract all data
        let odoo_cards = self.extractor.extract_loyalty_cards(None).await?;
        let odoo_history = self.extractor.extract_point_history(None).await?;
        let odoo_moves = self.extractor.extract_point_moves(None).await?;
        
        _logger.info!(
            "Extracted: {} cards, {} history, {} moves",
            odoo_cards.len(),
            odoo_history.len(),
            odoo_moves.len()
        );
        
        // Transform
        let rust_cards: Vec<_> = odoo_cards.into_iter()
            .filter_map(|c| self.transformer.transform_loyalty_card(c).ok())
            .collect();
        
        let rust_history: Vec<_> = odoo_history.into_iter()
            .filter_map(|h| self.transformer.transform_point_history(h).ok())
            .collect();
        
        // Load
        let card_result = self.loader.load_loyalty_cards(rust_cards).await?;
        let history_result = self.loader.load_point_history(rust_history).await?;
        
        // Update sync state
        self.sync_state.update_last_sync(Utc::now()).await?;
        
        _logger.info!("Full sync completed: {:?}", card_result);
        
        Ok(SyncResult {
            cards: card_result,
            history: history_result,
            // ...
        })
    }
    
    pub async fn run_incremental_sync(&self) -> Result<SyncResult> {
        let last_sync = self.sync_state.get_last_sync().await?;
        
        _logger.info!("Starting incremental sync from {:?}", last_sync);
        
        // Extract changed data
        let odoo_cards = self.extractor.extract_loyalty_cards(last_sync).await?;
        let odoo_history = self.extractor.extract_point_history(last_sync).await?;
        
        // Transform and load
        // Similar to full sync but only changed data
        
        // Update sync state
        self.sync_state.update_last_sync(Utc::now()).await?;
        
        Ok(SyncResult { /* ... */ })
    }
}
```

### Scheduled Sync

```rust
pub struct ScheduledETL {
    etl_service: Arc<ETLService>,
}

impl ScheduledETL {
    pub async fn run(&self) -> Result<()> {
        // Initial full sync
        self.etl_service.run_full_sync().await?;
        
        // Then incremental sync every minute
        let mut interval = tokio::time::interval(Duration::from_secs(60));
        
        loop {
            interval.tick().await;
            
            match self.etl_service.run_incremental_sync().await {
                Ok(result) => {
                    _logger.info!("Incremental sync successful: {:?}", result);
                }
                Err(e) => {
                    _logger.error!("Incremental sync failed: {}", e);
                    // Retry logic
                }
            }
        }
    }
}
```

## Data Consistency Validation

### Consistency Checker

```rust
pub struct ConsistencyChecker {
    odoo_extractor: Arc<OdooExtractor>,
    rust_loader: Arc<RustLoader>,
}

impl ConsistencyChecker {
    pub async fn check_consistency(&self) -> Result<ConsistencyReport> {
        // Get sample from both
        let odoo_cards = self.odoo_extractor.extract_loyalty_cards(None).await?;
        let rust_cards = self.rust_loader.get_all_cards().await?;
        
        let mut discrepancies = Vec::new();
        
        // Compare
        for odoo_card in &odoo_cards {
            if let Some(rust_card) = rust_cards.iter()
                .find(|c| c.odoo_card_id == odoo_card.id) {
                
                // Compare fields
                if odoo_card.points != rust_card.points {
                    discrepancies.push(Discrepancy {
                        entity_type: "loyalty_card",
                        entity_id: odoo_card.id,
                        field: "points",
                        odoo_value: odoo_card.points.to_string(),
                        rust_value: rust_card.points.to_string(),
                    });
                }
                
                if odoo_card.code != rust_card.code {
                    discrepancies.push(Discrepancy {
                        entity_type: "loyalty_card",
                        entity_id: odoo_card.id,
                        field: "code",
                        odoo_value: odoo_card.code.clone(),
                        rust_value: rust_card.code.clone(),
                    });
                }
            } else {
                discrepancies.push(Discrepancy {
                    entity_type: "loyalty_card",
                    entity_id: odoo_card.id,
                    field: "existence",
                    odoo_value: "exists".to_string(),
                    rust_value: "missing".to_string(),
                });
            }
        }
        
        // Check for extra in Rust
        for rust_card in &rust_cards {
            if !odoo_cards.iter().any(|c| c.id == rust_card.odoo_card_id) {
                discrepancies.push(Discrepancy {
                    entity_type: "loyalty_card",
                    entity_id: rust_card.odoo_card_id,
                    field: "existence",
                    odoo_value: "missing".to_string(),
                    rust_value: "exists".to_string(),
                });
            }
        }
        
        let consistency_rate = 1.0 - (discrepancies.len() as f64 / odoo_cards.len() as f64);
        
        Ok(ConsistencyReport {
            total_entities: odoo_cards.len(),
            discrepancies,
            consistency_rate,
            checked_at: Utc::now(),
        })
    }
}
```

## Monitoring & Alerting

### ETL Monitor

```rust
pub struct ETLMonitor {
    etl_service: Arc<ETLService>,
    consistency_checker: Arc<ConsistencyChecker>,
    alert_service: Arc<AlertService>,
}

impl ETLMonitor {
    pub async fn monitor_loop(&self) -> Result<()> {
        let mut interval = tokio::time::interval(Duration::from_secs(300)); // 5 minutes
        
        loop {
            interval.tick().await;
            
            // Check consistency
            let report = self.consistency_checker.check_consistency().await?;
            
            // Alert if consistency < 99%
            if report.consistency_rate < 0.99 {
                self.alert_service.send_alert(
                    Alert::DataInconsistency {
                        consistency_rate: report.consistency_rate,
                        discrepancies: report.discrepancies.len(),
                        report: report.clone(),
                    }
                ).await?;
            }
            
            // Log metrics
            _logger.info!(
                "Consistency check: {:.2}% ({}/{} discrepancies)",
                report.consistency_rate * 100.0,
                report.discrepancies.len(),
                report.total_entities
            );
        }
    }
}
```

## Success Criteria untuk Fase 2

- ✅ ETL running stable
- ✅ Data consistency > 99.9%
- ✅ Sync latency < 1 minute
- ✅ Zero data loss
- ✅ Monitoring working
- ✅ Ready untuk Fase 3

Lihat dokumentasi berikutnya:
- [02-phased-approach.md](02-phased-approach.md) - Overview semua fase
- [05-testing-plan.md](05-testing-plan.md) - Testing untuk ETL
