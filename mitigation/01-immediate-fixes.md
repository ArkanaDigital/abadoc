# Perbaikan Jangka Pendek

## Overview

Dokumen ini menjelaskan perbaikan yang bisa dilakukan segera untuk mengurangi risiko dan masalah yang ada saat ini, sebelum migrasi ke Rust engine.

## Priority Fixes

### Fix #1: Point Balance Consistency

**Priority**: P0 - Critical

**Problem**: Card points tidak sync dengan point history

**Solution**: Always calculate balance from history

**Implementation**:

```python
# asd_aba_loyalty/models/loyalty_card.py

@api.model
def _compute_points_from_history(self):
    """Compute points from latest history"""
    for card in self:
        latest_history = self.env['loyalty.point.history'].search([
            ('loyalty_card_id', '=', card.id)
        ], order='id desc', limit=1)
        
        if latest_history:
            card.points = latest_history.points_balance
        else:
            card.points = 0

# Override write to always sync
def write(self, vals):
    result = super().write(vals)
    if 'points' in vals:
        # Recalculate from history
        self._compute_points_from_history()
    return result
```

**Testing**:
```python
# Test script
def test_point_consistency():
    card = env['loyalty.card'].browse(card_id)
    latest_history = env['loyalty.point.history'].search([
        ('loyalty_card_id', '=', card.id)
    ], order='id desc', limit=1)
    
    assert card.points == latest_history.points_balance
```

### Fix #2: FEFO Point Consumption Lock

**Priority**: P0 - Critical

**Problem**: Race condition saat consume points

**Solution**: Use database lock

**Implementation**:

```python
# asd_aba_loyalty/models/loyalty_point_move.py

def handle_deducted_points(self, **kwargs):
    # Use SELECT FOR UPDATE
    self.env.cr.execute("""
        SELECT id, remaining_point
        FROM loyalty_point_move
        WHERE state = 'active'
        AND loyalty_card_id = %s
        AND membership_product_id = %s
        ORDER BY 
            CASE WHEN expiration_date IS NULL THEN 1 ELSE 0 END,
            expiration_date ASC
        LIMIT 1
        FOR UPDATE
    """, (loyalty_card_id, membership_product_id))
    
    row = self.env.cr.fetchone()
    if row:
        point_move_id, remaining = row
        # Update with lock
        # ...
```

**Testing**:
```python
# Concurrent test
def test_concurrent_point_consumption():
    # Create 10 concurrent orders
    # All try to redeem points
    # Verify no negative balance
```

### Fix #3: Membership Snapshot di Order

**Priority**: P0 - Critical

**Problem**: Membership berubah saat order processing

**Solution**: Snapshot membership saat order created

**Implementation**:

```python
# asd_pos_customize/models/pos_orders.py

@api.model
def _order_fields(self, ui_order):
    order_fields = super()._order_fields(ui_order)
    
    # Snapshot membership
    if ui_order.get('partner_id'):
        partner = self.env['res.partner'].browse(ui_order['partner_id'])
        order_fields['membership_product_id'] = partner.membership_product_id.id
        order_fields['membership_snapshot_date'] = fields.Datetime.now()
    
    return order_fields

# Use snapshot for point calculation
def _record_point_history(self, coupon_data, confirmed_coupon_data):
    # Use order's membership_product_id, not partner's current
    membership_product_id = self.membership_product_id.id
    # Not self.partner_id.membership_product_id.id
```

### Fix #4: Discount Recalculation Lock

**Priority**: P1 - High

**Problem**: Multiple calls bisa overwrite discount

**Solution**: Use flag untuk prevent re-entry

**Implementation**:

```python
# asd_pos_customize/models/pos_orders.py

def _recompute_discount(self, pos_order):
    # Check if already computing
    if pos_order.recompute_disc:
        return  # Already computing, skip
    
    pos_order.write({'recompute_disc': True})
    
    try:
        # Reset discounts
        for line in pos_order.lines:
            line.write({
                'discount_prog': 0,
                'discount_prog_all': 0,
                'disc_voucher': 0,
                'disc_promo_code': 0
            })
        
        # Calculate discounts
        # ...
        
    finally:
        pos_order.write({'recompute_disc': False})
```

### Fix #5: Discount Validation

**Priority**: P1 - High

**Problem**: Total discount bisa > price

**Solution**: Add validation

**Implementation**:

```python
# asd_pos_customize/models/pos_orders.py

def _validate_discounts(self, pos_order):
    """Validate that total discount doesn't exceed price"""
    for line in pos_order.lines:
        if line.is_reward_line:
            continue
            
        total_discount = (
            abs(line.discount_prog) +
            abs(line.discount_prog_all) +
            abs(line.disc_voucher) +
            abs(line.disc_promo_code)
        )
        
        base_price = line.price_unit * line.qty
        
        if total_discount > base_price:
            raise ValidationError(
                f"Total discount {total_discount} exceeds price {base_price} "
                f"for line {line.id}"
            )

# Call after _recompute_discount
def _recompute_discount(self, pos_order):
    # ... calculate discounts ...
    self._validate_discounts(pos_order)
```

## Quick Wins

### Quick Win #1: Add Database Constraints

```sql
-- Prevent negative points
ALTER TABLE loyalty_card 
ADD CONSTRAINT check_points_non_negative 
CHECK (points >= 0);

-- Prevent negative remaining points
ALTER TABLE loyalty_point_move 
ADD CONSTRAINT check_remaining_non_negative 
CHECK (remaining_point >= 0);

-- Prevent excessive discount
-- (Add check in application layer, not DB)
```

### Quick Win #2: Add Logging

```python
import logging
_logger = logging.getLogger(__name__)

def handle_added_points(self, **kwargs):
    _logger.info(
        f"Adding points: card={kwargs['loyalty_card_id']}, "
        f"points={kwargs['points_amount']}, origin={kwargs.get('origin')}"
    )
    # ... rest of code ...
```

### Quick Win #3: Add Monitoring

```python
# Add metrics
from odoo.addons.base.models.ir_mail_server import MailDeliveryException

def handle_deducted_points(self, **kwargs):
    start_time = time.time()
    try:
        # ... process ...
        duration = time.time() - start_time
        _logger.info(f"Point deduction took {duration:.2f}s")
    except Exception as e:
        _logger.error(f"Point deduction failed: {e}")
        raise
```

## Implementation Plan

### Week 1: Critical Fixes
- [ ] Fix #1: Point balance consistency
- [ ] Fix #2: FEFO lock
- [ ] Fix #3: Membership snapshot

### Week 2: High Priority Fixes
- [ ] Fix #4: Discount recalculation lock
- [ ] Fix #5: Discount validation

### Week 3: Quick Wins
- [ ] Database constraints
- [ ] Enhanced logging
- [ ] Monitoring

### Week 4: Testing & Validation
- [ ] Unit tests
- [ ] Integration tests
- [ ] Production validation

## Testing Checklist

- [ ] Point balance consistency test
- [ ] Concurrent point consumption test
- [ ] Membership snapshot test
- [ ] Discount validation test
- [ ] Performance test
- [ ] Load test

## Rollback Plan

Jika fix menyebabkan masalah:
1. Revert code changes
2. Restore database backup (if needed)
3. Investigate issue
4. Fix and retry

## Dokumentasi Terkait

- [06-phase1-detailed-fixes.md](06-phase1-detailed-fixes.md) - **Detail lengkap issue analysis dan fixes untuk Fase 1**
- [07-potential-future-issues.md](07-potential-future-issues.md) - **Potensi issue kedepannya dan prevention**
- [02-data-validation.md](02-data-validation.md) - Validasi data
- [03-transaction-safety.md](03-transaction-safety.md) - Keamanan transaksi
