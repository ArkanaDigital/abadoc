# Strategi Migrasi

## Overview

Migrasi dari sistem Python ke Rust engine harus dilakukan secara bertahap dan hati-hati untuk meminimalkan risiko dan downtime. Dokumen ini menjelaskan strategi migrasi yang komprehensif.

## Prinsip Migrasi

1. **Zero Downtime**: Sistem harus tetap berjalan selama migrasi
2. **Gradual Rollout**: Migrasi bertahap, bukan big bang
3. **Rollback Capability**: Bisa rollback kapan saja
4. **Data Consistency**: Data harus konsisten di kedua sistem
5. **Testing**: Extensive testing di setiap fase

## Fase Migrasi (Updated Approach)

**Pendekatan baru yang lebih aman dan bertahap:**

1. **Fase 1**: Perbaiki semua flow di Odoo sampai issue hilang
2. **Fase 2**: Buat middleware Rust dengan ETL paralel (read-only dari Odoo)
3. **Fase 3**: Development di Odoo untuk connect ke Rust
4. **Fase 4**: Test di staging
5. **Fase 5**: Rollout production dengan data yang sudah ter-update di Rust

Lihat detail di: `02-phased-approach.md` dan `06-etl-implementation.md`

---

## Fase Migrasi (Original Approach - Reference)

### Phase 1: Preparation (2-4 weeks)

**Tujuan**: Persiapan infrastruktur dan development

**Tasks**:
1. Setup Rust development environment
2. Setup CI/CD pipeline
3. Setup staging environment
4. Database schema migration
5. API contract definition
6. Integration testing framework

**Deliverables**:
- Rust engine deployed di staging
- API documentation
- Integration test suite
- Migration scripts

### Phase 2: Parallel Run (4-8 weeks)

**Tujuan**: Run kedua sistem secara parallel

**Tasks**:
1. Deploy Rust engine di production (read-only)
2. Route read operations ke Rust engine
3. Keep write operations di Python
4. Compare results dari kedua sistem
5. Fix discrepancies

**Deliverables**:
- Rust engine running di production
- Comparison reports
- Bug fixes

### Phase 3: Gradual Cutover (4-6 weeks)

**Tujuan**: Gradually migrate write operations

**Tasks**:
1. Migrate new orders ke Rust engine
2. Keep old orders di Python
3. Monitor dan fix issues
4. Increase percentage gradually (10% → 50% → 100%)

**Deliverables**:
- All new orders processed by Rust
- Monitoring dashboards
- Issue tracking

### Phase 4: Full Migration (2-4 weeks)

**Tujuan**: Migrate semua operations ke Rust

**Tasks**:
1. Migrate remaining operations
2. Migrate historical data processing
3. Deprecate Python code
4. Remove Python dependencies

**Deliverables**:
- All operations di Rust
- Python code deprecated
- Documentation updated

### Phase 5: Optimization (Ongoing)

**Tujuan**: Optimize dan improve

**Tasks**:
1. Performance tuning
2. Cost optimization
3. Feature enhancements
4. Monitoring improvements

## Migration Approach

### Approach 1: Strangler Fig Pattern (Recommended)

```
┌─────────────────────────────────────────┐
│         Odoo POS Frontend                │
└──────────────┬──────────────────────────┘
               │
        ┌──────┴──────┐
        │             │
   ┌────▼────┐   ┌───▼────┐
   │ Python  │   │ Rust   │
   │ (Old)   │   │ (New)  │
   └────┬────┘   └───┬────┘
        │            │
        └──────┬─────┘
               │
        ┌──────▼──────┐
        │ PostgreSQL  │
        └─────────────┘
```

**Benefits**:
- Gradual migration
- Easy rollback
- Low risk

### Approach 2: Feature Flag

```rust
pub struct OrderProcessor {
    use_rust_engine: bool,
}

impl OrderProcessor {
    pub async fn process_order(&self, order: Order) -> Result<ProcessedOrder> {
        if self.use_rust_engine {
            self.rust_engine.process(order).await
        } else {
            self.python_engine.process(order).await
        }
    }
}
```

**Benefits**:
- Easy A/B testing
- Instant rollback
- Gradual rollout

## Data Migration

### Schema Migration

```sql
-- Create new tables for Rust engine
CREATE TABLE rust_loyalty_card (
    id BIGSERIAL PRIMARY KEY,
    odoo_card_id INTEGER REFERENCES loyalty_card(id),
    code VARCHAR NOT NULL,
    points DECIMAL NOT NULL,
    created_at TIMESTAMP NOT NULL,
    updated_at TIMESTAMP NOT NULL
);

-- Migrate existing data
INSERT INTO rust_loyalty_card (odoo_card_id, code, points, created_at, updated_at)
SELECT id, code, points, create_date, write_date
FROM loyalty_card;
```

### Data Sync Strategy

1. **Initial Load**: Bulk import existing data
2. **Incremental Sync**: Sync new data in real-time
3. **Validation**: Compare data between systems
4. **Reconciliation**: Fix discrepancies

### Sync Script

```rust
pub struct DataSync {
    odoo_client: OdooClient,
    rust_repo: Arc<dyn PointRepository>,
}

impl DataSync {
    pub async fn sync_loyalty_cards(&self) -> Result<()> {
        // 1. Fetch from Odoo
        let cards = self.odoo_client.get_loyalty_cards().await?;
        
        // 2. Transform to Rust models
        let rust_cards = cards.into_iter()
            .map(|c| self.transform_card(c))
            .collect::<Vec<_>>();
        
        // 3. Save to Rust database
        for card in rust_cards {
            self.rust_repo.save_card(&card).await?;
        }
        
        Ok(())
    }
}
```

## Rollback Plan

### Rollback Triggers

1. **Error Rate**: > 1% error rate
2. **Performance**: Response time > 100ms (p99)
3. **Data Inconsistency**: > 0.1% inconsistency
4. **Critical Bug**: Security or data corruption

### Rollback Procedure

1. **Immediate**: Switch feature flag off
2. **Route Traffic**: Route semua traffic ke Python
3. **Investigate**: Investigate issue
4. **Fix**: Fix issue di Rust engine
5. **Retry**: Retry migration setelah fix

### Rollback Script

```bash
#!/bin/bash
# rollback.sh

# 1. Update feature flag
curl -X POST http://api/feature-flags/rust-engine -d '{"enabled": false}'

# 2. Verify traffic routing
curl http://api/health

# 3. Monitor error rate
# (monitoring dashboard)
```

## Testing Strategy

### Unit Tests

```rust
#[cfg(test)]
mod tests {
    #[tokio::test]
    async fn test_point_calculation() {
        // Test point calculation logic
    }
}
```

### Integration Tests

```rust
#[tokio::test]
async fn test_order_processing() {
    // Test full order processing
}
```

### Comparison Tests

```rust
#[tokio::test]
async fn test_python_vs_rust() {
    let python_result = python_engine.process_order(&order).await?;
    let rust_result = rust_engine.process_order(&order).await?;
    
    assert_eq!(python_result.total, rust_result.total);
    assert_eq!(python_result.points, rust_result.points);
}
```

### Load Tests

```rust
#[tokio::test]
async fn test_concurrent_orders() {
    // Test 1000 concurrent orders
    let orders = (0..1000).map(|i| create_test_order(i));
    
    let results = futures::future::join_all(
        orders.map(|o| rust_engine.process_order(&o))
    ).await;
    
    // Verify all succeeded
    assert!(results.iter().all(|r| r.is_ok()));
}
```

## Monitoring & Alerting

### Key Metrics

1. **Error Rate**: < 0.1%
2. **Response Time**: < 10ms (p99)
3. **Throughput**: > 1000 orders/second
4. **Data Consistency**: 100%

### Alerts

1. **High Error Rate**: > 1%
2. **Slow Response**: > 50ms (p99)
3. **Data Mismatch**: > 0.1%
4. **System Down**: Any downtime

### Monitoring Dashboard

- Real-time metrics
- Error tracking
- Performance graphs
- Comparison charts (Python vs Rust)

## Risk Mitigation

### Risk 1: Data Loss

**Mitigation**:
- Backup sebelum migrasi
- Transaction logging
- Data validation
- Rollback capability

### Risk 2: Performance Degradation

**Mitigation**:
- Load testing
- Performance monitoring
- Gradual rollout
- Auto-scaling

### Risk 3: Data Inconsistency

**Mitigation**:
- Data validation
- Comparison tests
- Reconciliation scripts
- Monitoring

### Risk 4: Integration Issues

**Mitigation**:
- API contract testing
- Integration tests
- Staging environment
- Gradual rollout

## Timeline

```
Week 1-2:   Preparation
Week 3-6:   Parallel Run
Week 7-10:  Gradual Cutover
Week 11-14: Full Migration
Week 15+:   Optimization
```

## Success Criteria

1. **Zero Data Loss**: 100% data integrity
2. **Performance**: < 10ms response time
3. **Error Rate**: < 0.1%
4. **Uptime**: > 99.9%
5. **Cost**: < current system cost

Lihat dokumentasi berikutnya:
- `02-phased-approach.md` - Detail pendekatan bertahap
- `03-data-migration.md` - Detail migrasi data
- `04-rollback-plan.md` - Detail rencana rollback
