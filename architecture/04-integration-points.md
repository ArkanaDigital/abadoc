# Titik Integrasi

## Overview

Dokumen ini menjelaskan semua titik integrasi antara modul promosi/membership dengan sistem lain, termasuk Odoo core, external systems, dan frontend.

## Integration Points

### 1. Odoo Core Integration

#### `pos_loyalty` Module

**Location**: `src/odoo/addons/pos_loyalty/`

**Integration Points**:
- `loyalty.program` model extension
- `loyalty.card` model extension
- `loyalty.rule` model extension
- `loyalty.reward` model extension
- POS frontend JavaScript extensions

**Custom Extensions**:
```python
# asd_aba_loyalty/models/loyalty_program.py
class LoyaltyProgram(models.Model):
    _inherit = 'loyalty.program'
    
    # Custom fields
    membership_product_id = fields.Many2one(...)
    is_primary_loyalty = fields.Boolean(...)
    # ...
```

**Frontend Integration**:
```javascript
// asd_aba_loyalty/static/src/js/Loyalty.js
// Extends Odoo's Loyalty.js
```

#### `point_of_sale` Module

**Location**: `src/odoo/addons/point_of_sale/`

**Integration Points**:
- `pos.order` model extension
- `pos.order.line` model extension
- `pos.session` model extension
- `pos.config` model extension
- POS frontend components

**Custom Extensions**:
```python
# asd_pos_customize/models/pos_orders.py
class PosOrder(models.Model):
    _inherit = 'pos.order'
    
    # Custom fields
    membership_product_id = fields.Many2one(...)
    referral_code_used = fields.Char(...)
    # ...
    
    # Custom methods
    def _recompute_discount(self, pos_order):
        # Custom discount calculation
        pass
```

#### `membership` Module

**Location**: `src/odoo/addons/membership/`

**Integration Points**:
- `membership.membership_line` model extension
- Membership evaluation logic

**Custom Extensions**:
```python
# asd_aba_loyalty/models/membership_line.py
class MembershipLine(models.Model):
    _inherit = 'membership.membership_line'
    
    # Custom fields
    member_transaction_amount_total = fields.Float(...)
    # ...
```

### 2. Frontend Integration

#### POS Frontend (JavaScript/OWL)

**Location**: `asd_aba_loyalty/static/src/js/`

**Components**:
- `Loyalty.js` - Main loyalty logic
- `RewardButton.js` - Reward button component
- `RedeemButton.js` - Redeem button component
- `PartnerLine.js` - Partner line component

**Integration Points**:
```javascript
// Extends Odoo's POS components
import PosComponent from 'point_of_sale.PosComponent';
import ProductScreen from 'point_of_sale.ProductScreen';

export class PromoCodeButton extends PosComponent {
    // Custom component
}
```

#### POS Customize Frontend

**Location**: `asd_pos_customize/static/src/js/`

**Components**:
- `ProductScreen.js` - Product screen customization
- `PaymentScreen.js` - Payment screen customization
- `OrderSummary.js` - Order summary customization

**Integration**:
- Replaces Odoo core components
- Extends Odoo's POS models

### 3. Database Integration

#### PostgreSQL

**Tables**:
- `loyalty_program` (extended)
- `loyalty_card` (extended)
- `loyalty_point_move` (custom)
- `loyalty_point_history` (custom)
- `pos_order` (extended)
- `pos_order_line` (extended)

**Views**:
- `promotion_result` - Promotion analytics
- `promotion_result_details` - Promotion details
- `pos_order_first` - First transaction tracking

**Stored Procedures**: None (using Odoo ORM)

### 4. External System Integration

#### Reporting System

**Location**: `asd_aba_reporting/`

**Integration**:
- Reads from `loyalty_program`, `loyalty_card`, `pos_order`
- Generates promotion reports
- Analytics and insights

**Data Flow**:
```
loyalty_program → promotion_result (view) → Report
loyalty_card → promotion_result_details (view) → Report
```

#### Celery (Async Tasks)

**Location**: `asd_aba_celery/`

**Integration**:
- Async point processing
- Background membership evaluation
- Scheduled tasks

**Tasks**:
- Point expiration check
- Membership evaluation
- Data sync

### 5. API Integration

#### XML-RPC / JSON-RPC

**Endpoints**:
- Standard Odoo RPC endpoints
- Custom endpoints untuk loyalty operations

**Usage**:
```python
# External system calls
import xmlrpc.client

models = xmlrpc.client.ServerProxy(
    f'{url}/xmlrpc/2/object'
)

# Get loyalty card
card = models.execute_kw(
    db, uid, password,
    'loyalty.card', 'search_read',
    [[('partner_id', '=', partner_id)]]
)
```

#### REST API (Future - Rust Engine)

**Endpoints** (Planned):
- `POST /api/v1/orders` - Create order
- `POST /api/v1/points/earn` - Earn points
- `POST /api/v1/points/redeem` - Redeem points
- `GET /api/v1/cards/{id}/balance` - Get balance

### 6. Event Integration

#### Odoo Automation Rules

**Location**: `asd_aba_loyalty/data/base_automation.xml`

**Rules**:
- Auto-create loyalty card saat partner created
- Auto-evaluate membership setelah order
- Auto-process points

**Example**:
```xml
<record id="auto_create_loyalty_card" model="base.automation">
    <field name="name">Auto Create Loyalty Card</field>
    <field name="model_id" ref="model_res_partner"/>
    <field name="trigger">on_create</field>
    <!-- ... -->
</record>
```

#### Cron Jobs

**Location**: `asd_aba_loyalty/data/cron.xml`

**Jobs**:
- Point expiration check (daily)
- Membership evaluation (hourly)
- Data cleanup (weekly)

### 7. File System Integration

#### Report Generation

**Location**: `asd_aba_loyalty/report/`

**Reports**:
- Referral code report
- Loyalty report
- Promotion result report

**Integration**:
- Uses Odoo's QWeb reporting engine
- Generates PDF/Excel files

### 8. Message Queue (Future)

#### Redis Pub/Sub

**Planned for Rust Engine**:
- Order events
- Point events
- Membership events

**Usage**:
```rust
// Publish event
event_bus.publish(EngineEvent::OrderCreated { order_id }).await?;

// Subscribe to events
let mut stream = event_bus.subscribe().await?;
while let Some(event) = stream.next().await {
    handle_event(event).await?;
}
```

## Integration Issues

### Issue 1: Tight Coupling

**Problem**: Modul terlalu tightly coupled dengan Odoo core

**Impact**: Sulit untuk test, maintain, atau replace

**Solution**: 
- Abstract interfaces
- Dependency injection
- Service layer

### Issue 2: Frontend-Backend Coupling

**Problem**: Business logic di frontend JavaScript

**Impact**: 
- Logic duplication
- Inconsistency
- Hard to test

**Solution**:
- Move logic ke backend
- API-driven frontend
- Shared validation

### Issue 3: Database Dependency

**Problem**: Direct database access tanpa abstraction

**Impact**:
- Hard to test
- Hard to migrate
- SQL injection risk

**Solution**:
- Repository pattern
- Query builder
- Parameterized queries

## Integration Best Practices

### 1. Use Interfaces

```python
# Define interface
class ILoyaltyService:
    def calculate_points(self, order):
        raise NotImplementedError

# Implement
class LoyaltyService(ILoyaltyService):
    def calculate_points(self, order):
        # Implementation
        pass
```

### 2. Dependency Injection

```python
class OrderProcessor:
    def __init__(self, loyalty_service: ILoyaltyService):
        self.loyalty_service = loyalty_service
    
    def process_order(self, order):
        points = self.loyalty_service.calculate_points(order)
        # ...
```

### 3. Event-Driven

```python
# Publish event
self.env['base.automation']._trigger('order.created', order)

# Subscribe
@api.model
def on_order_created(self, order):
    # Handle event
    pass
```

## Migration Considerations

### For Rust Engine Integration

1. **API Gateway**: Centralized API untuk semua integrations
2. **Message Queue**: Async communication
3. **Event Bus**: Decoupled event handling
4. **Service Mesh**: Microservices communication

### Integration Points for Rust

1. **REST API**: Primary integration point
2. **gRPC**: High-performance RPC
3. **WebSocket**: Real-time updates
4. **Message Queue**: Async processing

## Monitoring Integration

### Key Metrics

1. **API Response Time**: < 10ms
2. **Error Rate**: < 0.1%
3. **Throughput**: > 1000 req/s
4. **Availability**: > 99.9%

### Monitoring Tools

- **Prometheus**: Metrics collection
- **Grafana**: Visualization
- **ELK Stack**: Logging
- **Jaeger**: Distributed tracing

Lihat dokumentasi berikutnya:
- [../rust-engine/06-integration.md](../rust-engine/06-integration.md) - Detail integrasi Rust engine
- `../migration/` - Rencana migrasi
