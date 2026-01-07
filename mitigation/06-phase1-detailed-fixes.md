# Fase 1: Detail Perbaikan Odoo - Issue Analysis & Fixes

## Overview

Dokumen ini menganalisis secara detail semua issue yang ada di code saat ini, potensi issue kedepannya, dan rekomendasi perbaikan lengkap dengan code examples untuk Fase 1 migrasi.

## Metodologi Analisa

1. **Code Review**: Analisa langsung dari source code
2. **Pattern Analysis**: Identifikasi pattern masalah
3. **Root Cause Analysis**: Analisa penyebab masalah
4. **Impact Assessment**: Dampak dari setiap issue
5. **Fix Recommendations**: Rekomendasi perbaikan dengan code

---

## ISSUE 1: Point Balance Inconsistency (P0 - Critical)

### Analisa Issue Saat Ini

#### Evidence dari Code

**Location**: `asd_aba_loyalty/models/loyalty_point_move.py:557-575`

```python
def _update_card_and_move_history(self, loyalty_card, partner, change_level=False):
    point_moves, points = self._get_latest_balance_point(partner)
    # ...
    return loyalty_card.write({"points": points})
```

**Location**: `asd_aba_loyalty/models/pos_order.py:167-169` (COMMENTED OUT!)

```python
# loyalty_card.write({
#     'points': latest_point
# })  # COMMENTED OUT!
```

**Location**: `asd_aba_loyalty/models/loyalty_point_move.py:296-300` (COMMENTED OUT!)

```python
# if not diff_membership_product_id:
#     loyalty_card.write({'points': latest_point})
# else:
#     dest_loyalty_card = self.env['loyalty.card'].browse(dest_loyalty_card_id)
#     dest_loyalty_card.write({'points': latest_point})
```

#### Root Cause Analysis

1. **Inconsistent Update Points**:
   - Points diupdate di beberapa tempat berbeda
   - Beberapa update di-comment out
   - Tidak ada single source of truth

2. **Calculation Method Inconsistency**:
   - `_get_latest_balance_point()`: Calculate dari `point_moves.remaining_point`
   - `_record_point_history()`: Calculate dari `point_history.points_balance`
   - Bisa berbeda hasilnya

3. **Race Condition**:
   - Multiple concurrent updates bisa overwrite satu sama lain
   - Tidak ada lock mechanism

#### Impact

- **Data Corruption**: Card points tidak match dengan actual balance
- **Financial Loss**: Customer bisa redeem lebih dari available
- **Customer Complaints**: Balance tidak akurat
- **Operational Overhead**: Perlu manual adjustment

#### Current Workaround

Ada automatic adjustment system di `membership_log.py` yang menunjukkan masalah ini sering terjadi.

### Potensi Issue Kedepannya

1. **Scale Issues**: 
   - Semakin banyak concurrent orders, semakin tinggi kemungkinan inconsistency
   - Performance degradation saat calculate balance

2. **Data Migration Issues**:
   - Sulit untuk migrate data yang inconsistent
   - Perlu cleanup sebelum migration

3. **Reporting Issues**:
   - Reports tidak akurat karena data inconsistent
   - Analytics tidak reliable

### Rekomendasi Perbaikan Detail

#### Fix 1.1: Single Source of Truth untuk Point Balance

**Strategy**: Always calculate dari `point_history.points_balance`, never dari `point_moves`

**Implementation**:

```python
# asd_aba_loyalty/models/loyalty_card.py

class LoyaltyCard(models.Model):
    _inherit = 'loyalty.card'
    
    @api.depends('point_history_ids')
    def _compute_points_from_history(self):
        """Compute points from latest point history - SINGLE SOURCE OF TRUTH"""
        for card in self:
            latest_history = self.env['loyalty.point.history'].search([
                ('loyalty_card_id', '=', card.id)
            ], order='id desc', limit=1)
            
            if latest_history:
                card.points = latest_history.points_balance
            else:
                card.points = 0.0
    
    points = fields.Float(
        compute='_compute_points_from_history',
        store=True,
        readonly=True,
        help='Current point balance calculated from point history'
    )
    
    def write(self, vals):
        """Override write to prevent manual point updates"""
        if 'points' in vals:
            # Remove points from vals - always compute from history
            vals.pop('points')
            _logger.warning(
                f"Attempted to manually update points for card {self.id}. "
                f"Points are computed from history and cannot be manually updated."
            )
        return super().write(vals)
```

#### Fix 1.2: Atomic Balance Update

**Strategy**: Use database-level calculation untuk ensure consistency

**Implementation**:

```python
# asd_aba_loyalty/models/loyalty_point_history.py

class LoyaltyPointHistory(models.Model):
    _name = 'loyalty.point.history'
    _inherit = 'loyalty.point.history'
    
    @api.model
    def create(self, vals):
        """Override create to ensure balance consistency"""
        # Calculate balance from previous history
        if 'points_balance' not in vals or not vals.get('points_balance'):
            card_id = vals.get('loyalty_card_id')
            if card_id:
                latest = self.search([
                    ('loyalty_card_id', '=', card_id)
                ], order='id desc', limit=1)
                
                previous_balance = latest.points_balance if latest else 0.0
                points_amount = vals.get('points_amount', 0.0)
                vals['points_balance'] = previous_balance + points_amount
        
        # Validate balance non-negative
        if vals.get('points_balance', 0) < 0:
            raise ValidationError(
                f"Point balance cannot be negative. "
                f"Attempted balance: {vals['points_balance']}"
            )
        
        history = super().create(vals)
        
        # Update card points atomically
        self._update_card_points_atomic(history.loyalty_card_id)
        
        return history
    
    @api.model
    def _update_card_points_atomic(self, card_id):
        """Update card points atomically using SQL"""
        self.env.cr.execute("""
            UPDATE loyalty_card
            SET points = (
                SELECT points_balance
                FROM loyalty_point_history
                WHERE loyalty_card_id = %s
                ORDER BY id DESC
                LIMIT 1
            )
            WHERE id = %s
        """, (card_id, card_id))
        
        self.env.cr.commit()
```

#### Fix 1.3: Database Constraint

**Strategy**: Add database constraint untuk prevent negative balance

**Implementation**:

```sql
-- Migration script
ALTER TABLE loyalty_card 
ADD CONSTRAINT check_points_non_negative 
CHECK (points >= 0);

ALTER TABLE loyalty_point_history 
ADD CONSTRAINT check_points_balance_non_negative 
CHECK (points_balance >= 0);

ALTER TABLE loyalty_point_move 
ADD CONSTRAINT check_remaining_point_non_negative 
CHECK (remaining_point >= 0);
```

#### Fix 1.4: Validation Method

**Strategy**: Add validation method untuk check consistency

**Implementation**:

```python
# asd_aba_loyalty/models/loyalty_card.py

class LoyaltyCard(models.Model):
    _inherit = 'loyalty.card'
    
    def validate_point_consistency(self):
        """Validate that card points match history balance"""
        self.ensure_one()
        
        latest_history = self.env['loyalty.point.history'].search([
            ('loyalty_card_id', '=', self.id)
        ], order='id desc', limit=1)
        
        expected_balance = latest_history.points_balance if latest_history else 0.0
        
        if abs(self.points - expected_balance) > 0.01:  # Allow small floating point differences
            _logger.error(
                f"Point inconsistency detected for card {self.id}: "
                f"card.points={self.points}, history.balance={expected_balance}"
            )
            return False
        
        return True
    
    @api.model
    def cron_validate_all_points(self):
        """Cron job to validate all card points"""
        cards = self.search([])
        inconsistent = []
        
        for card in cards:
            if not card.validate_point_consistency():
                inconsistent.append(card.id)
                # Auto-fix
                card._compute_points_from_history()
        
        if inconsistent:
            _logger.warning(
                f"Found {len(inconsistent)} cards with inconsistent points: {inconsistent}"
            )
        
        return len(inconsistent)
```

### Testing

```python
def test_point_balance_consistency(self):
    """Test that card points always match history balance"""
    card = self.env['loyalty.card'].create({...})
    
    # Add points
    self.env['loyalty.point.move'].handle_added_points(
        points_amount=100,
        loyalty_card_id=card.id,
        ...
    )
    
    # Refresh card
    card.refresh()
    
    # Validate
    self.assertTrue(card.validate_point_consistency())
    self.assertEqual(card.points, 100.0)
```

---

## ISSUE 2: FEFO Point Consumption Race Condition (P0 - Critical)

### Analisa Issue Saat Ini

#### Evidence dari Code

**Location**: `asd_aba_loyalty/models/loyalty_point_move.py:215-262`

```python
while points_to_consume > 0:
    lifetime_point_move = self.search([
        ('state', '=', 'active'),
        ('loyalty_card_id', '=', loyalty_card_id),
        ('membership_product_id', '=', membership_product_id),
        ('expiration_date', '=', False)
    ], order='gain_date', limit=1)
    
    # ... multiple search paths ...
    
    if point_move:
        previous_remaining_point = point_move.remaining_point
        remaining_points = point_move.remaining_point - points_to_consume
        point_move.write({'remaining_point': max(0, remaining_points),
                        'state': 'depleted' if remaining_points <= 0 else 'active'})
        points_to_consume = max(0, -remaining_points)
```

#### Root Cause Analysis

1. **No Database Lock**:
   - Search tanpa `FOR UPDATE`
   - Multiple concurrent requests bisa read same point_move
   - Write bisa overwrite satu sama lain

2. **Multiple Search Paths**:
   - Logic complex dengan multiple search paths
   - Debug variables menunjukkan uncertainty
   - Tidak deterministic

3. **No Transaction Isolation**:
   - Tidak ada proper transaction boundary
   - Race condition bisa terjadi

#### Impact

- **Negative Balance**: Points bisa digunakan lebih dari available
- **Double Consumption**: Same points bisa digunakan multiple times
- **Data Corruption**: Point moves state tidak konsisten
- **Financial Loss**: Customer bisa redeem lebih dari seharusnya

### Potensi Issue Kedepannya

1. **High Concurrency**:
   - Semakin banyak concurrent orders, semakin tinggi risk
   - Peak hours akan lebih problematic

2. **Scale Issues**:
   - Performance degradation dengan banyak concurrent requests
   - Database lock contention

3. **Data Integrity**:
   - Sulit untuk recover dari corrupted data
   - Perlu extensive cleanup

### Rekomendasi Perbaikan Detail

#### Fix 2.1: Database-Level Locking

**Strategy**: Use `SELECT FOR UPDATE` untuk lock point moves

**Implementation**:

```python
# asd_aba_loyalty/models/loyalty_point_move.py

@api.model
def handle_deducted_points(self, **kwargs):
    """Handle point deduction with proper locking"""
    self.ensure_one()
    
    loyalty_card_id = kwargs['loyalty_card_id']
    membership_product_id = kwargs['membership_product_id']
    points_amount = abs(kwargs['points_amount'])
    origin = kwargs.get('origin', '')
    
    # Use database transaction with lock
    with self.env.cr.savepoint():
        # Lock card first
        self.env.cr.execute("""
            SELECT id FROM loyalty_card 
            WHERE id = %s 
            FOR UPDATE
        """, (loyalty_card_id,))
        
        if not self.env.cr.fetchone():
            raise ValidationError(f"Loyalty card {loyalty_card_id} not found")
        
        points_consumed = []
        points_to_consume = points_amount
        
        while points_to_consume > 0:
            # Find and lock point move with FOR UPDATE
            self.env.cr.execute("""
                SELECT id, remaining_point, expiration_date, gain_date
                FROM loyalty_point_move
                WHERE state = 'active'
                AND loyalty_card_id = %s
                AND membership_product_id = %s
                ORDER BY 
                    CASE WHEN expiration_date IS NULL THEN 1 ELSE 0 END,
                    expiration_date ASC NULLS LAST,
                    gain_date ASC
                LIMIT 1
                FOR UPDATE SKIP LOCKED
            """, (loyalty_card_id, membership_product_id))
            
            row = self.env.cr.fetchone()
            
            if not row:
                # No more point moves available
                _logger.warning(
                    f"Insufficient points for card {loyalty_card_id}. "
                    f"Required: {points_to_consume}, Available: {sum(p[1] for p in points_consumed)}"
                )
                raise ValidationError(
                    f"Insufficient points. Required: {points_to_consume}, "
                    f"Available: {sum(p[1] for p in points_consumed)}"
                )
            
            point_move_id, remaining_point, expiration_date, gain_date = row
            
            # Calculate consumption
            consume_amount = min(points_to_consume, remaining_point)
            new_remaining = remaining_point - consume_amount
            new_state = 'depleted' if new_remaining <= 0 else 'active'
            
            # Update atomically
            self.env.cr.execute("""
                UPDATE loyalty_point_move
                SET remaining_point = %s,
                    state = %s
                WHERE id = %s
            """, (new_remaining, new_state, point_move_id))
            
            points_consumed.append((point_move_id, consume_amount))
            points_to_consume -= consume_amount
        
        # Create point history records
        self._create_point_history_for_consumption(
            loyalty_card_id,
            membership_product_id,
            points_consumed,
            origin
        )
        
        # Update card balance
        self._update_card_balance_atomic(loyalty_card_id)
        
        return points_consumed
```

#### Fix 2.2: Retry Logic dengan Exponential Backoff

**Strategy**: Retry jika lock failed

**Implementation**:

```python
from tenacity import retry, stop_after_attempt, wait_exponential

@retry(
    stop=stop_after_attempt(3),
    wait=wait_exponential(multiplier=1, min=0.1, max=2)
)
def handle_deducted_points_with_retry(self, **kwargs):
    """Handle point deduction with retry on lock failure"""
    try:
        return self.handle_deducted_points(**kwargs)
    except psycopg2.errors.LockNotAvailable:
        # Another transaction has the lock, retry
        raise
    except Exception as e:
        _logger.error(f"Point deduction failed: {e}")
        raise
```

#### Fix 2.3: Validation Before Consumption

**Strategy**: Validate available points sebelum consume

**Implementation**:

```python
@api.model
def validate_sufficient_points(self, card_id, points_required):
    """Validate that card has sufficient points"""
    available_points = self.env.cr.execute("""
        SELECT COALESCE(SUM(remaining_point), 0)
        FROM loyalty_point_move
        WHERE loyalty_card_id = %s
        AND state = 'active'
    """, (card_id,))
    
    available = self.env.cr.fetchone()[0]
    
    if available < points_required:
        raise ValidationError(
            f"Insufficient points. Required: {points_required}, "
            f"Available: {available}"
        )
    
    return True
```

### Testing

```python
def test_concurrent_point_consumption(self):
    """Test concurrent point consumption doesn't cause negative balance"""
    card = self.env['loyalty.card'].create({...})
    
    # Add 100 points
    self.env['loyalty.point.move'].handle_added_points(
        points_amount=100,
        loyalty_card_id=card.id,
        ...
    )
    
    # Simulate 10 concurrent orders trying to redeem 20 points each
    import threading
    
    def redeem_points():
        try:
            self.env['loyalty.point.move'].handle_deducted_points(
                points_amount=-20,
                loyalty_card_id=card.id,
                ...
            )
        except ValidationError:
            pass  # Expected if insufficient
    
    threads = [threading.Thread(target=redeem_points) for _ in range(10)]
    for t in threads:
        t.start()
    for t in threads:
        t.join()
    
    # Validate final balance
    card.refresh()
    self.assertGreaterEqual(card.points, 0)
    # Should be 0 (all consumed) or 20 (if one failed)
```

---

## ISSUE 3: Membership Mismatch di Order Processing (P0 - Critical)

### Analisa Issue Saat Ini

#### Evidence dari Code

**Location**: `asd_pos_customize/models/pos_orders.py:117-156`

```python
@api.model
def _order_fields(self, ui_order):
    # ...
    order_fields['membership_product_id'] = partner.membership_product_id.id
    # ...
    return order_fields
```

**Location**: `asd_aba_loyalty/models/pos_order.py:182`

```python
move_data = {
    'membership_product_id': self.membership_product_id.id,  # From order!
    # ...
}
```

#### Root Cause Analysis

1. **Snapshot Missing**:
   - Order `membership_product_id` di-set saat order created
   - Membership bisa berubah sebelum order confirmed
   - Point calculation menggunakan order's membership (bisa salah)

2. **No Validation**:
   - Tidak ada check apakah membership masih valid
   - Tidak ada check consistency dengan partner

3. **Timing Issue**:
   - Order created → Membership changed → Order confirmed
   - Points calculated dengan membership yang salah

#### Impact

- **Wrong Point Calculation**: Points dihitung untuk membership level yang salah
- **Customer Loss**: Customer dapat points lebih sedikit
- **Data Inconsistency**: Order membership != Partner membership

### Potensi Issue Kedepannya

1. **Membership Upgrade/Downgrade**:
   - Saat membership berubah, semua pending orders akan salah
   - Perlu re-calculation untuk semua pending orders

2. **Reporting Issues**:
   - Reports tidak akurat karena membership mismatch
   - Analytics tidak reliable

### Rekomendasi Perbaikan Detail

#### Fix 3.1: Membership Snapshot dengan Timestamp

**Strategy**: Snapshot membership saat order created dengan timestamp

**Implementation**:

```python
# asd_pos_customize/models/pos_orders.py

class PosOrder(models.Model):
    _inherit = 'pos.order'
    
    membership_product_id = fields.Many2one(
        'product.product',
        string='Membership Product',
        readonly=True,
        help='Membership level at order creation time (snapshot)'
    )
    membership_snapshot_date = fields.Datetime(
        string='Membership Snapshot Date',
        readonly=True,
        help='Timestamp when membership was snapshotted'
    )
    membership_snapshot_valid = fields.Boolean(
        string='Membership Snapshot Valid',
        compute='_compute_membership_snapshot_valid',
        help='Whether membership snapshot is still valid'
    )
    
    @api.depends('membership_product_id', 'partner_id.membership_product_id', 'membership_snapshot_date')
    def _compute_membership_snapshot_valid(self):
        """Check if membership snapshot is still valid"""
        for order in self:
            if not order.membership_product_id or not order.partner_id:
                order.membership_snapshot_valid = False
                continue
            
            # Snapshot is valid if partner membership hasn't changed
            # OR if order was created within last 5 minutes (processing window)
            time_diff = fields.Datetime.now() - (order.membership_snapshot_date or order.create_date)
            is_recent = time_diff.total_seconds() < 300  # 5 minutes
            
            order.membership_snapshot_valid = (
                order.membership_product_id == order.partner_id.membership_product_id
            ) or is_recent
    
    @api.model
    def _order_fields(self, ui_order):
        order_fields = super()._order_fields(ui_order)
        
        # Snapshot membership saat order created
        if ui_order.get('partner_id'):
            partner = self.env['res.partner'].browse(ui_order['partner_id'])
            order_fields['membership_product_id'] = partner.membership_product_id.id
            order_fields['membership_snapshot_date'] = fields.Datetime.now()
        
        return order_fields
```

#### Fix 3.2: Validation Before Point Calculation

**Strategy**: Validate membership snapshot sebelum calculate points

**Implementation**:

```python
# asd_aba_loyalty/models/pos_order.py

def _record_point_history(self, coupon_data, confirmed_coupon_data):
    """Record point history with membership validation"""
    self.ensure_one()
    
    # Validate membership snapshot
    if not self.membership_snapshot_valid:
        _logger.warning(
            f"Order {self.name}: Membership snapshot invalid. "
            f"Order membership: {self.membership_product_id.id}, "
            f"Partner membership: {self.partner_id.membership_product_id.id}"
        )
        
        # Use partner's current membership instead
        membership_product_id = self.partner_id.membership_product_id.id
    else:
        # Use order's snapshot membership
        membership_product_id = self.membership_product_id.id
    
    # Continue with point calculation using validated membership
    # ...
```

#### Fix 3.3: Re-evaluation untuk Pending Orders

**Strategy**: Re-evaluate membership untuk pending orders jika membership berubah

**Implementation**:

```python
# asd_aba_loyalty/models/partner.py

class ResPartner(models.Model):
    _inherit = 'res.partner'
    
    def action_member_evaluate(self):
        """Override to re-evaluate pending orders"""
        result = super().action_member_evaluate()
        
        # Re-evaluate pending orders jika membership berubah
        if self._membership_changed():
            self._re_evaluate_pending_orders()
        
        return result
    
    def _membership_changed(self):
        """Check if membership has changed"""
        # Compare with previous value
        # Implementation depends on tracking previous value
        return True
    
    def _re_evaluate_pending_orders(self):
        """Re-evaluate pending orders with new membership"""
        pending_orders = self.env['pos.order'].search([
            ('partner_id', '=', self.id),
            ('state', 'in', ['draft', 'paid']),
            ('membership_product_id', '!=', self.membership_product_id.id)
        ])
        
        for order in pending_orders:
            # Update membership snapshot
            order.write({
                'membership_product_id': self.membership_product_id.id,
                'membership_snapshot_date': fields.Datetime.now(),
            })
            
            # Recalculate points if needed
            if order.state == 'paid':
                order._recalculate_points_with_new_membership()
```

### Testing

```python
def test_membership_snapshot(self):
    """Test membership snapshot works correctly"""
    partner = self.env['res.partner'].create({...})
    partner.membership_product_id = membership_a
    
    # Create order
    order = self.env['pos.order'].create({
        'partner_id': partner.id,
        ...
    })
    
    # Verify snapshot
    self.assertEqual(order.membership_product_id, membership_a)
    self.assertTrue(order.membership_snapshot_valid)
    
    # Change membership
    partner.membership_product_id = membership_b
    
    # Verify snapshot still valid (recent)
    order.refresh()
    self.assertTrue(order.membership_snapshot_valid)
    
    # After 10 minutes, should be invalid
    order.membership_snapshot_date = fields.Datetime.now() - timedelta(minutes=10)
    order._compute_membership_snapshot_valid()
    self.assertFalse(order.membership_snapshot_valid)
```

---

## ISSUE 4: Discount Recalculation Race Condition (P1 - High)

### Analisa Issue Saat Ini

#### Evidence dari Code

**Location**: `asd_pos_customize/models/pos_orders.py:404-414`

```python
@api.model
def _recompute_discount(self, pos_order):
    pos_order.write({'recompute_disc': True})
    
    # PERBAIKAN 1: Reset semua discount fields di awal
    for line in pos_order.lines:
        line.write({
            'discount_prog': 0,
            'discount_prog_all': 0,
            'disc_voucher': 0,
            'disc_promo_code': 0
        })
```

#### Root Cause Analysis

1. **No Lock Mechanism**:
   - Method bisa dipanggil multiple times concurrently
   - Reset discount fields bisa terjadi saat sedang dihitung
   - Tidak ada flag untuk prevent re-entry

2. **Multiple Entry Points**:
   - Dipanggil dari berbagai tempat
   - Tidak ada coordination
   - Bisa overlap

3. **Non-Atomic Operation**:
   - Reset → Calculate → Write
   - Bisa interrupted di tengah
   - Partial updates

#### Impact

- **Discount Loss**: Discount hilang saat concurrent calls
- **Double Discount**: Discount bisa double jika timing tepat
- **Wrong Total**: Order total tidak akurat

### Potensi Issue Kedepannya

1. **High Traffic**:
   - Semakin banyak concurrent orders, semakin tinggi risk
   - Peak hours problematic

2. **Complex Discounts**:
   - Multiple discount types akan lebih complex
   - Higher chance of errors

### Rekomendasi Perbaikan Detail

#### Fix 4.1: Lock Mechanism dengan Flag

**Strategy**: Use database lock dan flag untuk prevent re-entry

**Implementation**:

```python
# asd_pos_customize/models/pos_orders.py

@api.model
def _recompute_discount(self, pos_order):
    """Recompute discount with proper locking"""
    pos_order.ensure_one()
    
    # Check if already computing
    if pos_order.recompute_disc:
        _logger.warning(f"Discount recalculation already in progress for order {pos_order.name}")
        return  # Already computing, skip
    
    # Lock order
    with self.env.cr.savepoint():
        # Set flag atomically
        self.env.cr.execute("""
            UPDATE pos_order
            SET recompute_disc = TRUE
            WHERE id = %s AND recompute_disc = FALSE
            RETURNING id
        """, (pos_order.id,))
        
        if not self.env.cr.fetchone():
            # Another process is computing, skip
            return
        
        try:
            # Reset all discount fields
            self.env.cr.execute("""
                UPDATE pos_order_line
                SET discount_prog = 0,
                    discount_prog_all = 0,
                    disc_voucher = 0,
                    disc_promo_code = 0
                WHERE order_id = %s
            """, (pos_order.id,))
            
            # Calculate discounts
            self._calculate_discounts_internal(pos_order)
            
        finally:
            # Reset flag
            self.env.cr.execute("""
                UPDATE pos_order
                SET recompute_disc = FALSE
                WHERE id = %s
            """, (pos_order.id,))
            
            self.env.cr.commit()
```

#### Fix 4.2: Single Calculation Pass

**Strategy**: Calculate semua discounts dalam satu pass dengan priority

**Implementation**:

```python
def _calculate_discounts_internal(self, pos_order):
    """Internal method to calculate all discounts in one pass"""
    # Collect all rewards
    rewards = []
    for line in pos_order.lines:
        if line.reward_id and line.is_reward_line:
            rewards.append({
                'line': line,
                'reward': line.reward_id,
                'program': line.reward_id.program_id,
                'priority': self._get_discount_priority(line.reward_id.program_id.program_type),
            })
    
    # Sort by priority
    rewards.sort(key=lambda x: x['priority'])
    
    # Apply discounts in order
    for reward_info in rewards:
        self._apply_single_discount(pos_order, reward_info)
    
    # Validate total discount
    self._validate_total_discount(pos_order)
```

### Testing

```python
def test_concurrent_discount_recalculation(self):
    """Test concurrent discount recalculation doesn't cause issues"""
    order = self.env['pos.order'].create({...})
    
    # Simulate concurrent calls
    import threading
    
    def recalculate():
        order._recompute_discount(order)
    
    threads = [threading.Thread(target=recalculate) for _ in range(5)]
    for t in threads:
        t.start()
    for t in threads:
        t.join()
    
    # Validate discounts
    for line in order.lines:
        total_discount = (
            line.discount_prog + 
            line.discount_prog_all + 
            line.disc_voucher + 
            line.disc_promo_code
        )
        self.assertLessEqual(total_discount, line.price_subtotal_incl)
```

---

## ISSUE 5: Discount Accumulation Error (P1 - High)

### Analisa Issue Saat Ini

#### Evidence dari Code

**Location**: `asd_pos_customize/models/pos_orders.py:358-376`

```python
def calculate_discount_percentage(self, rec_line, qty_count, disc, total_count, type, reward, disc_special=0):
    disc_val = 0
    price = rec_line.price_subtotal_incl  # Already includes discounts!
    
    # ...
    
    disc_val = rec_line.discount_prog + (((price + existing_disc) / total_count) * disc)
```

#### Root Cause Analysis

1. **Discount on Discount**:
   - Calculate dari `price_subtotal_incl` yang sudah include discounts
   - Discount dihitung dari price yang sudah didiscount
   - Accumulation error

2. **No Base Price**:
   - Tidak ada base price calculation
   - Semua calculation dari discounted price

3. **Multiple Discount Types**:
   - Tidak ada clear order
   - Bisa overlap

#### Impact

- **Over Discount**: Discount lebih besar dari seharusnya
- **Negative Total**: Order total bisa negative
- **Financial Loss**: Revenue loss

### Rekomendasi Perbaikan Detail

#### Fix 5.1: Calculate dari Base Price

**Strategy**: Always calculate dari base price, apply discounts sequentially

**Implementation**:

```python
def calculate_discount_percentage(self, rec_line, qty_count, disc, total_count, type, reward, disc_special=0):
    """Calculate discount from base price"""
    # Get base price (before any discounts)
    base_price = rec_line.price_unit * rec_line.qty
    
    # Calculate discount amount
    if total_count == 0:
        total_count = 1
    
    discount_amount = (base_price / total_count) * disc
    
    # Get existing discounts (for tracking only)
    existing_discounts = (
        rec_line.discount_prog +
        rec_line.discount_prog_all +
        rec_line.disc_voucher +
        rec_line.disc_promo_code
    )
    
    # New discount value
    if reward in ["cuopon", "promo_code"]:
        if type == 2:
            disc_val = rec_line.disc_promo_code + discount_amount
        else:
            disc_val = rec_line.disc_voucher + discount_amount
    else:
        if type == 1:
            disc_val = rec_line.discount_prog_all + discount_amount
        else:
            disc_val = rec_line.discount_prog + discount_amount
    
    # Validate total discount doesn't exceed base price
    total_discount = existing_discounts + discount_amount
    if total_discount > base_price:
        _logger.warning(
            f"Total discount {total_discount} exceeds base price {base_price} "
            f"for line {rec_line.id}. Capping to base price."
        )
        disc_val = base_price - existing_discounts
    
    return disc_val
```

#### Fix 5.2: Discount Validation

**Strategy**: Validate total discount sebelum apply

**Implementation**:

```python
def _validate_total_discount(self, pos_order):
    """Validate that total discount doesn't exceed price"""
    for line in pos_order.lines:
        if line.is_reward_line:
            continue
        
        base_price = line.price_unit * line.qty
        total_discount = (
            abs(line.discount_prog) +
            abs(line.discount_prog_all) +
            abs(line.disc_voucher) +
            abs(line.disc_promo_code)
        )
        
        if total_discount > base_price:
            raise ValidationError(
                f"Total discount {total_discount} exceeds base price {base_price} "
                f"for line {line.id} in order {pos_order.name}"
            )
```

---

## ISSUE 6: Offline/Online Sync Issues (P1 - High)

### Analisa Issue Saat Ini

#### Evidence dari Code

**Location**: `asd_pos_customize/models/pos_orders.py:855-890`

```python
if(order['data'].get('pos_state')=="offline"):
    for coupon_id, coupon_change in order['data'].get('couponPointChanges').items():
        # Process points untuk offline orders
        if(coupon_change['program_id'] == loyalty_program.id):
            loyalty_point_move_obj.handle_added_points(**move_data)
```

**Location**: `asd_aba_loyalty/models/pos_order.py:551-635`

```python
@api.model
def _process_record_loyalty_point(self, order_created_ids, orders_ui):
    """Process points untuk online orders"""
    # Different code path
```

#### Root Cause Analysis

1. **Different Code Paths**:
   - Offline: Processed di `create_from_ui()`
   - Online: Processed di `_process_record_loyalty_point()`
   - Different logic = different behavior

2. **No Coordination**:
   - Tidak ada check apakah sudah processed
   - Bisa double process
   - Bisa miss process

3. **Sync Issues**:
   - Offline orders sync later
   - Points bisa duplicate atau missing

#### Impact

- **Double Points**: Points bisa terhitung 2x
- **Missing Points**: Points tidak terhitung
- **Data Inconsistency**: Inconsistent antara offline dan online

### Rekomendasi Perbaikan Detail

#### Fix 6.1: Unified Point Processing

**Strategy**: Single code path untuk offline dan online

**Implementation**:

```python
# asd_aba_loyalty/models/pos_order.py

@api.model
def _process_loyalty_points_unified(self, order, coupon_data):
    """Unified point processing for both offline and online orders"""
    # Check if already processed
    if order.points_processed:
        _logger.warning(f"Order {order.name} points already processed")
        return
    
    # Mark as processing
    order.write({'points_processing': True})
    
    try:
        # Process points (same logic for both)
        self._process_points_internal(order, coupon_data)
        
        # Mark as processed
        order.write({
            'points_processed': True,
            'points_processing': False,
        })
    except Exception as e:
        order.write({'points_processing': False})
        raise
```

#### Fix 6.2: Idempotency Check

**Strategy**: Check jika points sudah processed

**Implementation**:

```python
def _check_points_already_processed(self, order):
    """Check if points already processed for this order"""
    # Check point history
    existing_history = self.env['loyalty.point.history'].search([
        ('origin', '=', order.name)
    ], limit=1)
    
    return existing_history.exists()
```

---

## Summary: Priority Fixes untuk Fase 1

### Critical (P0) - Must Fix

1. ✅ **Point Balance Consistency** - Fix 1.1, 1.2, 1.3, 1.4
2. ✅ **FEFO Race Condition** - Fix 2.1, 2.2, 2.3
3. ✅ **Membership Mismatch** - Fix 3.1, 3.2, 3.3

### High Priority (P1) - Should Fix

4. ✅ **Discount Race Condition** - Fix 4.1, 4.2
5. ✅ **Discount Accumulation** - Fix 5.1, 5.2
6. ✅ **Offline/Online Sync** - Fix 6.1, 6.2

### Implementation Order

1. Week 1-2: Fix P0 issues
2. Week 3-4: Fix P1 issues
3. Week 5-6: Testing & validation
4. Week 7-8: Monitoring & fine-tuning

Lihat dokumentasi berikutnya:
- [01-immediate-fixes.md](01-immediate-fixes.md) - Quick fixes
- [02-data-validation.md](02-data-validation.md) - Data validation
- [03-transaction-safety.md](03-transaction-safety.md) - Transaction safety
