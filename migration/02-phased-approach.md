# Pendekatan Migrasi Bertahap

## Overview

Pendekatan migrasi bertahap yang aman dengan 5 fase utama, memastikan zero downtime dan data consistency di setiap tahap.

## Fase Migrasi

```
Fase 1: Fix Odoo Issues (8 weeks)
    ↓
Fase 2: ETL Parallel (Read-Only) (6-10 weeks)
    ↓
Fase 3: Odoo → Rust Integration (4-6 weeks)
    ↓
Fase 4: Staging Testing (2-4 weeks)
    ↓
Fase 5: Production Rollout (2-4 weeks)
```

**Lihat Executive Summary**: [07-phase1-executive-summary.md](07-phase1-executive-summary.md)

---

## Fase 1: Perbaiki Semua Flow di Odoo (DETAIL)

**Detail Issue Analysis**: Lihat [../mitigation/06-phase1-detailed-fixes.md](../mitigation/06-phase1-detailed-fixes.md)
**Potensi Issues**: Lihat [../mitigation/07-potential-future-issues.md](../mitigation/07-potential-future-issues.md)
**Executive Summary**: Lihat [07-phase1-executive-summary.md](07-phase1-executive-summary.md)

## Fase 1: Perbaiki Semua Flow di Odoo

### Tujuan
Memperbaiki semua issue di sistem Odoo sampai tidak ada masalah lagi sebelum migrasi.

### Timeline
**8 minggu** (4 weeks critical fixes + 4 weeks high priority + optimization)

### Tasks

#### 1.1 Fix Critical Issues (P0)

**Detail Issue Analysis**: Lihat [../mitigation/06-phase1-detailed-fixes.md](../mitigation/06-phase1-detailed-fixes.md)

- [ ] **Fix 1.1**: Single source of truth untuk point balance
- [ ] **Fix 1.2**: Atomic balance update dengan SQL
- [ ] **Fix 1.3**: Database constraints untuk prevent negative balance
- [ ] **Fix 1.4**: Validation method untuk check consistency
- [ ] **Fix 2.1**: Database-level locking dengan SELECT FOR UPDATE
- [ ] **Fix 2.2**: Retry logic dengan exponential backoff
- [ ] **Fix 2.3**: Validation sebelum consumption
- [ ] **Fix 3.1**: Membership snapshot dengan timestamp
- [ ] **Fix 3.2**: Validation sebelum point calculation
- [ ] **Fix 3.3**: Re-evaluation untuk pending orders

**Deliverables**:
- Point balance selalu konsisten (validated)
- Tidak ada race condition (locked)
- Membership selalu correct (snapshotted)
- Data integrity guaranteed (constraints)

#### 1.2 Fix High Priority Issues (P1)

**Detail Issue Analysis**: Lihat [../mitigation/06-phase1-detailed-fixes.md](../mitigation/06-phase1-detailed-fixes.md)

- [ ] **Fix 4.1**: Lock mechanism dengan flag untuk discount recalculation
- [ ] **Fix 4.2**: Single calculation pass dengan priority
- [ ] **Fix 5.1**: Calculate discount dari base price (not discounted price)
- [ ] **Fix 5.2**: Discount validation untuk prevent over-discount
- [ ] **Fix 6.1**: Unified point processing untuk offline dan online
- [ ] **Fix 6.2**: Idempotency check untuk prevent double processing

**Deliverables**:
- Discount calculation reliable (validated, locked)
- Sync issues resolved (unified processing)
- Validation comprehensive (all operations)

#### 1.3 Code Quality Improvements

**Potensi Issues**: Lihat [../mitigation/07-potential-future-issues.md](../mitigation/07-potential-future-issues.md)

- [ ] Remove quick fixes (FIXME comments)
- [ ] Remove debug code (debug variables)
- [ ] Uncomment validations (commented ValidationError)
- [ ] Refactor long methods (`_recompute_discount` 300+ lines)
- [ ] Add proper indexes untuk performance
- [ ] Implement caching untuk frequently accessed data
- [ ] Add comprehensive tests (unit + integration)
- [ ] Add monitoring dan alerting

**Deliverables**:
- Code quality improved (no quick fixes, no debug code)
- Technical debt reduced (refactored, documented)
- Test coverage > 80% (comprehensive tests)
- Performance optimized (indexes, caching)
- Monitoring in place (metrics, alerts)

#### 1.4 Monitoring & Validation
- [ ] Setup monitoring
- [ ] Track error rates
- [ ] Validate fixes
- [ ] Performance testing

**Deliverables**:
- Error rate < 0.1%
- Performance acceptable
- No critical issues

### Success Criteria
- ✅ Zero critical issues
- ✅ Error rate < 0.1%
- ✅ Data consistency 100%
- ✅ All tests passing
- ✅ Performance acceptable

### Rollback Plan
Jika ada issue baru muncul:
1. Revert fix
2. Investigate
3. Fix properly
4. Retry

## Fase 2: ETL Parallel (Read-Only)

### Tujuan
Membangun middleware Rust dengan ETL yang membaca dari Odoo dan menulis ke Rust database secara paralel, sampai data di Rust benar-benar stable dan konsisten.

### Timeline
**6-10 minggu**

### Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    Odoo Database                         │
│  (PostgreSQL)                                            │
│  - Source of Truth                                       │
│  - All operations tetap di Odoo                          │
└────────────────────┬────────────────────────────────────┘
                     │
                     │ ETL (Read-Only)
                     │
        ┌────────────▼────────────┐
        │   Rust ETL Service       │
        │   - Extract from Odoo    │
        │   - Transform            │
        │   - Load to Rust DB      │
        └────────────┬────────────┘
                     │
        ┌────────────▼────────────┐
        │   Rust Engine Database   │
        │   (PostgreSQL)           │
        │   - Read-only replica    │
        │   - Data validation      │
        │   - Consistency check   │
        └─────────────────────────┘
```

### Tasks

#### 2.1 Setup Rust ETL Infrastructure
- [ ] Setup Rust project structure
- [ ] Setup database connections (Odoo + Rust)
- [ ] Setup connection pooling
- [ ] Setup logging dan monitoring

**Code Structure**:
```rust
rust-etl/
├── src/
│   ├── extract/
│   │   ├── odoo_extractor.rs
│   │   └── data_reader.rs
│   ├── transform/
│   │   ├── data_transformer.rs
│   │   └── validator.rs
│   ├── load/
│   │   ├── rust_loader.rs
│   │   └── data_writer.rs
│   └── main.rs
└── Cargo.toml
```

#### 2.2 Implement Extract Layer
- [ ] Extract loyalty cards dari Odoo
- [ ] Extract point moves dari Odoo
- [ ] Extract point history dari Odoo
- [ ] Extract promotion programs dari Odoo
- [ ] Extract membership data dari Odoo

**Implementation**:
```rust
pub struct OdooExtractor {
    odoo_client: Arc<OdooClient>,
}

impl OdooExtractor {
    pub async fn extract_loyalty_cards(&self, last_sync: Option<DateTime<Utc>>) -> Result<Vec<LoyaltyCard>> {
        let domain = if let Some(sync_time) = last_sync {
            vec![("write_date", ">", sync_time)]
        } else {
            vec![]
        };
        
        let cards = self.odoo_client.search_read(
            "loyalty.card",
            domain,
            vec!["id", "code", "partner_id", "program_id", "points", "write_date"]
        ).await?;
        
        Ok(cards.into_iter().map(|c| self.transform_card(c)).collect())
    }
    
    pub async fn extract_point_history(&self, last_sync: Option<DateTime<Utc>>) -> Result<Vec<PointHistory>> {
        // Similar implementation
    }
}
```

#### 2.3 Implement Transform Layer
- [ ] Transform Odoo data ke Rust schema
- [ ] Data validation
- [ ] Data cleaning
- [ ] Data enrichment

**Implementation**:
```rust
pub struct DataTransformer;

impl DataTransformer {
    pub fn transform_loyalty_card(&self, odoo_card: OdooCard) -> Result<RustCard> {
        // Validate
        self.validate_card(&odoo_card)?;
        
        // Transform
        Ok(RustCard {
            odoo_card_id: odoo_card.id,
            code: odoo_card.code,
            partner_id: odoo_card.partner_id,
            program_id: odoo_card.program_id,
            points: self.calculate_points_from_history(&odoo_card)?,
            created_at: odoo_card.create_date,
            updated_at: odoo_card.write_date,
        })
    }
    
    fn validate_card(&self, card: &OdooCard) -> Result<()> {
        if card.code.is_empty() {
            return Err(Error::Validation("Card code cannot be empty"));
        }
        // More validations
        Ok(())
    }
}
```

#### 2.4 Implement Load Layer
- [ ] Load data ke Rust database
- [ ] Upsert logic (insert or update)
- [ ] Conflict resolution
- [ ] Batch processing

**Implementation**:
```rust
pub struct RustLoader {
    rust_db: Arc<PostgresPool>,
}

impl RustLoader {
    pub async fn load_loyalty_cards(&self, cards: Vec<RustCard>) -> Result<()> {
        let mut tx = self.rust_db.begin().await?;
        
        for card in cards {
            // Upsert logic
            sqlx::query!(
                r#"
                INSERT INTO loyalty_cards (
                    odoo_card_id, code, partner_id, program_id, 
                    points, created_at, updated_at
                ) VALUES ($1, $2, $3, $4, $5, $6, $7)
                ON CONFLICT (odoo_card_id) 
                DO UPDATE SET
                    points = EXCLUDED.points,
                    updated_at = EXCLUDED.updated_at
                "#,
                card.odoo_card_id,
                card.code,
                card.partner_id,
                card.program_id,
                card.points,
                card.created_at,
                card.updated_at
            )
            .execute(&mut *tx)
            .await?;
        }
        
        tx.commit().await?;
        Ok(())
    }
}
```

#### 2.5 Implement Incremental Sync
- [ ] Track last sync timestamp
- [ ] Incremental extraction
- [ ] Change detection
- [ ] Sync status tracking

**Implementation**:
```rust
pub struct IncrementalSync {
    extractor: Arc<OdooExtractor>,
    loader: Arc<RustLoader>,
    sync_state: Arc<SyncState>,
}

impl IncrementalSync {
    pub async fn sync_all(&self) -> Result<SyncResult> {
        let last_sync = self.sync_state.get_last_sync().await?;
        
        // Extract changed data
        let cards = self.extractor.extract_loyalty_cards(last_sync).await?;
        let point_history = self.extractor.extract_point_history(last_sync).await?;
        let point_moves = self.extractor.extract_point_moves(last_sync).await?;
        
        // Transform
        let transformed_cards = cards.into_iter()
            .map(|c| self.transformer.transform_loyalty_card(c))
            .collect::<Result<Vec<_>>>()?;
        
        // Load
        self.loader.load_loyalty_cards(transformed_cards).await?;
        
        // Update sync state
        self.sync_state.update_last_sync(Utc::now()).await?;
        
        Ok(SyncResult {
            cards_synced: transformed_cards.len(),
            history_synced: point_history.len(),
            // ...
        })
    }
}
```

#### 2.6 Data Validation & Consistency Check
- [ ] Compare data antara Odoo dan Rust
- [ ] Validate point balances
- [ ] Check data integrity
- [ ] Report discrepancies

**Implementation**:
```rust
pub struct DataValidator {
    odoo_client: Arc<OdooClient>,
    rust_db: Arc<PostgresPool>,
}

impl DataValidator {
    pub async fn validate_consistency(&self) -> Result<ValidationReport> {
        // Get sample data from both
        let odoo_cards = self.odoo_client.get_all_cards().await?;
        let rust_cards = self.rust_db.get_all_cards().await?;
        
        let mut discrepancies = Vec::new();
        
        for odoo_card in odoo_cards {
            if let Some(rust_card) = rust_cards.iter().find(|c| c.odoo_card_id == odoo_card.id) {
                // Compare
                if odoo_card.points != rust_card.points {
                    discrepancies.push(Discrepancy {
                        card_id: odoo_card.id,
                        field: "points",
                        odoo_value: odoo_card.points,
                        rust_value: rust_card.points,
                    });
                }
            } else {
                discrepancies.push(Discrepancy {
                    card_id: odoo_card.id,
                    field: "existence",
                    odoo_value: "exists",
                    rust_value: "missing",
                });
            }
        }
        
        Ok(ValidationReport {
            total_cards: odoo_cards.len(),
            discrepancies,
            consistency_rate: 1.0 - (discrepancies.len() as f64 / odoo_cards.len() as f64),
        })
    }
}
```

#### 2.7 Monitoring & Alerting
- [ ] Sync status monitoring
- [ ] Data consistency monitoring
- [ ] Performance monitoring
- [ ] Alert on discrepancies

**Implementation**:
```rust
pub struct SyncMonitor {
    validator: Arc<DataValidator>,
    alert_service: Arc<AlertService>,
}

impl SyncMonitor {
    pub async fn monitor_loop(&self) -> Result<()> {
        let mut interval = tokio::time::interval(Duration::from_secs(300)); // 5 minutes
        
        loop {
            interval.tick().await;
            
            // Validate consistency
            let report = self.validator.validate_consistency().await?;
            
            // Alert if consistency < 99%
            if report.consistency_rate < 0.99 {
                self.alert_service.send_alert(
                    Alert::DataInconsistency {
                        consistency_rate: report.consistency_rate,
                        discrepancies: report.discrepancies.len(),
                    }
                ).await?;
            }
        }
    }
}
```

### Sync Schedule

**Initial Load**:
- Full sync semua historical data
- Run sekali di awal
- Estimated: 2-4 hours untuk 1M records

**Incremental Sync**:
- Every 1 minute: Critical data (point history, point moves)
- Every 5 minutes: Non-critical data (programs, membership)
- Every 1 hour: Reference data (products, partners)

### Success Criteria
- ✅ Data di Rust DB 100% konsisten dengan Odoo
- ✅ Sync latency < 1 minute untuk critical data
- ✅ Zero data loss
- ✅ Consistency rate > 99.9%
- ✅ Monitoring dan alerting working

### Validation Process

**Daily Validation**:
1. Run consistency check
2. Compare random samples
3. Validate point balances
4. Check data completeness

**Weekly Validation**:
1. Full data comparison
2. Historical data validation
3. Performance analysis
4. Report generation

## Fase 3: Odoo → Rust Integration Development

### Tujuan
Mengembangkan integrasi di Odoo untuk connect ke Rust engine, dengan fallback ke Odoo jika Rust tidak available.

### Timeline
**4-6 minggu**

### Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    Odoo POS Frontend                     │
└────────────────────┬────────────────────────────────────┘
                     │
        ┌────────────▼────────────┐
        │   Odoo Backend           │
        │   - Feature Flag         │
        │   - Rust Client          │
        │   - Fallback Logic       │
        └────────────┬────────────┘
                     │
        ┌────────────┴────────────┐
        │                         │
┌───────▼────────┐      ┌────────▼──────────┐
│ Rust Engine    │      │ Odoo (Fallback)  │
│ (Primary)      │      │ (If Rust down)    │
└────────────────┘      └─────────────────┘
```

### Tasks

#### 3.1 Setup Feature Flag System
- [ ] Implement feature flag di Odoo
- [ ] Configurable per POS config
- [ ] Gradual rollout capability

**Implementation**:
```python
# odoo/addons/asd_aba_loyalty/models/res_config.py

class ResConfig(models.Model):
    _inherit = 'res.config.settings'
    
    use_rust_engine = fields.Boolean(
        string='Use Rust Engine',
        config_parameter='asd_aba_loyalty.use_rust_engine',
        default=False,
    )
    
    rust_engine_url = fields.Char(
        string='Rust Engine URL',
        config_parameter='asd_aba_loyalty.rust_engine_url',
    )
```

#### 3.2 Implement Rust Client di Odoo
- [ ] Rust client library untuk Python
- [ ] Connection pooling
- [ ] Error handling
- [ ] Retry logic

**Implementation**:
```python
# odoo/addons/asd_aba_loyalty/services/rust_client.py

import requests
from odoo import api, models
from odoo.exceptions import UserError

class RustClient:
    def __init__(self, base_url, timeout=10):
        self.base_url = base_url
        self.timeout = timeout
        self.session = requests.Session()
    
    def process_order(self, order_data):
        """Process order via Rust engine"""
        try:
            response = self.session.post(
                f"{self.base_url}/api/v1/orders",
                json=order_data,
                timeout=self.timeout
            )
            response.raise_for_status()
            return response.json()
        except requests.exceptions.RequestException as e:
            raise UserError(f"Rust engine error: {e}")
    
    def get_balance(self, card_id):
        """Get card balance from Rust engine"""
        try:
            response = self.session.get(
                f"{self.base_url}/api/v1/cards/{card_id}/balance",
                timeout=self.timeout
            )
            response.raise_for_status()
            return response.json()
        except requests.exceptions.RequestException as e:
            raise UserError(f"Rust engine error: {e}")
```

#### 3.3 Implement Fallback Logic
- [ ] Check Rust availability
- [ ] Fallback ke Odoo jika Rust down
- [ ] Log fallback events

**Implementation**:
```python
# odoo/addons/asd_aba_loyalty/models/pos_order.py

class PosOrder(models.Model):
    _inherit = 'pos.order'
    
    def _process_loyalty_points(self, order_data):
        """Process loyalty points with Rust fallback"""
        config = self.env['ir.config_parameter'].sudo()
        use_rust = config.get_param('asd_aba_loyalty.use_rust_engine', 'False') == 'True'
        
        if use_rust:
            try:
                rust_client = self._get_rust_client()
                result = rust_client.process_order(order_data)
                
                # Log success
                _logger.info(f"Order {self.name} processed via Rust engine")
                return result
            except Exception as e:
                # Fallback to Odoo
                _logger.warning(f"Rust engine failed, falling back to Odoo: {e}")
                return self._process_loyalty_points_odoo(order_data)
        else:
            # Use Odoo
            return self._process_loyalty_points_odoo(order_data)
    
    def _process_loyalty_points_odoo(self, order_data):
        """Original Odoo processing"""
        # Existing Odoo logic
        pass
```

#### 3.4 Gradual Rollout
- [ ] Start dengan 0% traffic
- [ ] Increase gradually (10% → 50% → 100%)
- [ ] Monitor error rates
- [ ] Rollback jika needed

**Implementation**:
```python
# odoo/addons/asd_aba_loyalty/models/pos_config.py

class PosConfig(models.Model):
    _inherit = 'pos.config'
    
    rust_engine_rollout_percentage = fields.Integer(
        string='Rust Engine Rollout %',
        default=0,
        help='Percentage of orders to process via Rust engine'
    )
    
    def should_use_rust_engine(self):
        """Determine if should use Rust engine based on rollout percentage"""
        import random
        
        if not self.env['ir.config_parameter'].sudo().get_param(
            'asd_aba_loyalty.use_rust_engine', 'False'
        ) == 'True':
            return False
        
        # Random based on percentage
        return random.randint(1, 100) <= self.rust_engine_rollout_percentage
```

### Success Criteria
- ✅ Feature flag working
- ✅ Rust client integrated
- ✅ Fallback logic working
- ✅ Gradual rollout capability
- ✅ Monitoring in place

## Fase 4: Staging Testing

### Tujuan
Testing comprehensive di staging environment sebelum production rollout.

### Timeline
**2-4 minggu**

### Test Scenarios

#### 4.1 Functional Testing
- [ ] Order processing
- [ ] Point calculation
- [ ] Discount application
- [ ] Membership evaluation
- [ ] Refund processing

#### 4.2 Integration Testing
- [ ] Odoo → Rust integration
- [ ] Rust → Odoo sync
- [ ] Fallback mechanism
- [ ] Error handling

#### 4.3 Performance Testing
- [ ] Load testing (1000+ orders/second)
- [ ] Latency testing (< 10ms p99)
- [ ] Concurrent testing
- [ ] Stress testing

#### 4.4 Data Consistency Testing
- [ ] Compare Odoo vs Rust data
- [ ] Validate point balances
- [ ] Check transaction integrity
- [ ] Verify sync accuracy

#### 4.5 Failure Testing
- [ ] Rust engine down scenario
- [ ] Network failure
- [ ] Database failure
- [ ] Sync failure

### Success Criteria
- ✅ All tests passing
- ✅ Performance acceptable
- ✅ Data consistency 100%
- ✅ Error rate < 0.1%
- ✅ No critical bugs

## Fase 5: Production Rollout

### Tujuan
Rollout ke production dengan data yang sudah ter-update di Rust karena ETL paralel.

### Timeline
**2-4 minggu**

### Rollout Strategy

#### Week 1: 10% Traffic
- Enable untuk 10% orders
- Monitor closely
- Fix any issues

#### Week 2: 50% Traffic
- Increase ke 50%
- Continue monitoring
- Validate data consistency

#### Week 3: 100% Traffic
- Full rollout
- Monitor for 1 week
- Validate everything

#### Week 4: Deprecate Odoo Logic
- Remove Odoo fallback
- Clean up old code
- Documentation update

### Monitoring

**Key Metrics**:
- Error rate < 0.1%
- Latency < 10ms (p99)
- Data consistency 100%
- Sync success rate > 99.9%

**Alerts**:
- Error rate spike
- Latency increase
- Data inconsistency
- Sync failure

### Rollback Plan

Jika ada critical issue:
1. Set feature flag to 0%
2. All traffic back to Odoo
3. Investigate issue
4. Fix and retry

## Timeline Summary

| Fase | Duration | Total |
|------|----------|-------|
| Fase 1: Fix Odoo | 4-8 weeks | 4-8 weeks |
| Fase 2: ETL Parallel | 6-10 weeks | 10-18 weeks |
| Fase 3: Integration Dev | 4-6 weeks | 14-24 weeks |
| Fase 4: Staging Test | 2-4 weeks | 16-28 weeks |
| Fase 5: Production Rollout | 2-4 weeks | 18-32 weeks |

**Total**: 4.5 - 8 months

## Success Criteria Overall

- ✅ Zero critical issues
- ✅ Data consistency 100%
- ✅ Performance improved
- ✅ Error rate < 0.1%
- ✅ Zero downtime
- ✅ Smooth migration

## Dokumentasi Terkait

- [06-etl-implementation.md](06-etl-implementation.md) - **Detail implementasi ETL untuk Fase 2**
- [03-data-migration.md](03-data-migration.md) - Detail data migration
- [04-rollback-plan.md](04-rollback-plan.md) - Detail rollback plan
- [05-testing-plan.md](05-testing-plan.md) - Detail testing plan
