# Masalah Integritas Data

## Overview

Masalah integritas data adalah masalah kritis yang dapat menyebabkan data corruption, financial loss, dan customer complaints. Dokumen ini mendokumentasikan semua masalah integritas data yang teridentifikasi.

## Issue Categories

1. **Point Balance Inconsistency**
2. **Membership Product Mismatch**
3. **Discount Field Conflicts**
4. **Order State Inconsistency**
5. **Referral Usage Tracking**

## 1. Point Balance Inconsistency

### Problem Description

Point balance di `loyalty.card.points` tidak selalu konsisten dengan `loyalty.point.history.points_balance`. 

**Expected Behavior**:
```python
card.points == sum(history.filtered(lambda h: h.loyalty_card_id == card).mapped('points_balance'))[-1]
```

**Actual Behavior**:
- Card points bisa berbeda dengan latest history balance
- Card points bisa tidak ter-update setelah point move
- History balance bisa tidak match dengan card points

### Root Causes

#### Cause 1: Manual Point Updates

**Location**: Multiple locations

**Code Evidence**:
```python
# loyalty_point_move.py line 167-169
# loyalty_card.write({'points': latest_point})  # COMMENTED OUT!

# Card points tidak diupdate dari point moves
# Balance hanya diupdate via _update_card_and_move_history()
```

**Impact**: Card points bisa stale

#### Cause 2: Race Condition

**Location**: `loyalty_point_move.py::handle_added_points()` dan `handle_deducted_points()`

**Scenario**:
1. Order A: Earn 100 points
2. Order B: Redeem 50 points (concurrent)
3. Both read card.points = 0
4. Order A: Write card.points = 100
5. Order B: Write card.points = -50 (overwrite!)

**Impact**: Negative balance atau incorrect balance

#### Cause 3: History Balance Calculation

**Location**: `loyalty_point_history.py`

**Code**:
```python
# loyalty_point_move.py line 161
latest_point = latest_point_history.points_balance + point_move.initial_point
```

**Problem**: 
- Jika `latest_point_history` tidak ada, `points_balance` = 0
- Jika history tidak sequential, balance bisa salah
- Jika ada missing history, balance tidak akurat

### Detection Query

```sql
-- Find cards with inconsistent balance
SELECT 
    lc.id as card_id,
    lc.points as card_points,
    (
        SELECT points_balance 
        FROM loyalty_point_history 
        WHERE loyalty_card_id = lc.id 
        ORDER BY id DESC 
        LIMIT 1
    ) as latest_history_balance,
    (
        SELECT SUM(remaining_point)
        FROM loyalty_point_move
        WHERE loyalty_card_id = lc.id 
        AND state = 'active'
    ) as active_move_balance
FROM loyalty_card lc
WHERE lc.points != (
    SELECT points_balance 
    FROM loyalty_point_history 
    WHERE loyalty_card_id = lc.id 
    ORDER BY id DESC 
    LIMIT 1
);
```

### Mitigation

1. **Always Update Card from History**:
   ```python
   def _update_card_balance(self, loyalty_card):
       latest_history = self.env['loyalty.point.history'].search([
           ('loyalty_card_id', '=', loyalty_card.id)
       ], order='id desc', limit=1)
       
       if latest_history:
           loyalty_card.points = latest_history.points_balance
       else:
           loyalty_card.points = 0
   ```

2. **Use Database Constraints**:
   ```sql
   ALTER TABLE loyalty_card 
   ADD CONSTRAINT check_points_non_negative 
   CHECK (points >= 0);
   ```

3. **Atomic Updates**:
   ```python
   # Use SELECT FOR UPDATE
   with self.env.cr.savepoint():
       card = self.env['loyalty.card'].browse(card_id).with_lock()
       # Update points
   ```

## 2. Membership Product Mismatch

### Problem Description

`membership_product_id` di order berbeda dengan `partner.membership_product_id` saat order confirmed.

**Expected Behavior**:
```python
order.membership_product_id == order.partner_id.membership_product_id
```

**Actual Behavior**:
- Order dibuat dengan membership A
- Membership berubah ke B sebelum order confirmed
- Points dihitung untuk membership A (salah!)

### Root Causes

#### Cause 1: Membership Change During Order Processing

**Location**: `pos_orders.py::create_from_ui()`

**Scenario**:
1. Order created dengan `membership_product_id = A`
2. Order processing takes time
3. Another order triggers membership evaluation
4. Membership changed to B
5. Current order confirmed dengan membership A
6. Points calculated untuk membership A (wrong!)

#### Cause 2: Point Calculation Uses Order Membership

**Location**: `pos_order.py::_record_point_history()`

**Code**:
```python
# Line 182
move_data = {
    'membership_product_id': self.membership_product_id.id,  # From order!
    # ...
}
```

**Problem**: Uses order's membership, not partner's current membership

### Detection Query

```sql
-- Find orders with membership mismatch
SELECT 
    po.id as order_id,
    po.name as order_name,
    po.membership_product_id as order_membership,
    rp.membership_product_id as partner_membership,
    po.date_order
FROM pos_order po
JOIN res_partner rp ON po.partner_id = rp.id
WHERE po.membership_product_id != rp.membership_product_id
AND po.state IN ('paid', 'done');
```

### Mitigation

1. **Use Partner Membership for Point Calculation**:
   ```python
   def _record_point_history(self, coupon_data, confirmed_coupon_data):
       # Use partner's current membership, not order's
       membership_product_id = self.partner_id.membership_product_id.id
       # Not self.membership_product_id
   ```

2. **Snapshot Membership at Order Creation**:
   ```python
   # Store membership snapshot
   order.membership_product_id = partner.membership_product_id.id
   order.membership_snapshot_date = fields.Datetime.now()
   ```

3. **Validate Before Confirmation**:
   ```python
   def action_pos_order_paid(self):
       if self.membership_product_id != self.partner_id.membership_product_id:
           # Re-evaluate or warn
           pass
   ```

## 3. Discount Field Conflicts

### Problem Description

Multiple discount fields bisa overlap atau conflict, menyebabkan total discount lebih besar dari price.

**Fields**:
- `discount` (Odoo core, percentage)
- `discount_prog` (program specific, monetary)
- `discount_prog_all` (program all, monetary)
- `disc_voucher` (voucher, monetary)
- `disc_promo_code` (promo code, monetary)

### Root Causes

#### Cause 1: No Total Discount Validation

**Location**: `pos_orders.py::_recompute_discount()`

**Problem**: Tidak ada check untuk total discount <= price

#### Cause 2: Discount Accumulation Error

**Location**: `pos_orders.py::calculate_discount_percentage()`

**Code**:
```python
# Line 360
price = rec_line.price_subtotal_incl  # Already includes discounts!
disc_val = rec_line.discount_prog + (((price + existing_disc) / total_count) * disc)
```

**Problem**: Discount dihitung dari price yang sudah didiscount

### Detection Query

```sql
-- Find lines with excessive discount
SELECT 
    pol.id,
    pol.order_id,
    pol.product_id,
    pol.price_unit,
    pol.qty,
    pol.price_subtotal,
    pol.discount_prog,
    pol.discount_prog_all,
    pol.disc_voucher,
    pol.disc_promo_code,
    (pol.discount_prog + pol.discount_prog_all + pol.disc_voucher + pol.disc_promo_code) as total_discount,
    pol.price_subtotal_incl
FROM pos_order_line pol
WHERE (pol.discount_prog + pol.discount_prog_all + pol.disc_voucher + pol.disc_promo_code) > pol.price_subtotal_incl;
```

### Mitigation

1. **Validate Total Discount**:
   ```python
   def _validate_discount(self, line):
       total_discount = (line.discount_prog + 
                        line.discount_prog_all + 
                        line.disc_voucher + 
                        line.disc_promo_code)
       
       if total_discount > line.price_subtotal_incl:
           raise ValidationError("Total discount exceeds price")
   ```

2. **Calculate Discount from Base Price**:
   ```python
   # Use base price, not price_subtotal_incl
   base_price = line.price_unit * line.qty
   # Calculate discount from base_price
   ```

3. **Single Discount Calculation**:
   ```python
   # Calculate all discounts in one pass
   # Apply with priority
   # Validate total
   ```

## 4. Order State Inconsistency

### Problem Description

Order state tidak konsisten dengan payment state atau point processing state.

**Expected States**:
- `draft` → `paid` → `done`
- Points processed only when `paid` or `done`

**Actual Behavior**:
- Points bisa processed untuk `draft` orders
- Orders bisa `paid` tanpa points processed
- Refund orders bisa tidak properly handled

### Detection Query

```sql
-- Find orders with state inconsistency
SELECT 
    po.id,
    po.name,
    po.state,
    COUNT(lph.id) as point_history_count,
    COUNT(lpm.id) as point_move_count
FROM pos_order po
LEFT JOIN loyalty_point_history lph ON lph.origin = po.name
LEFT JOIN loyalty_point_move lpm ON lpm.origin = po.name
WHERE po.state = 'paid'
AND (lph.id IS NULL OR lpm.id IS NULL);
```

### Mitigation

1. **State Machine Validation**:
   ```python
   def write(self, vals):
       if 'state' in vals:
           self._validate_state_transition(vals['state'])
       return super().write(vals)
   ```

2. **Ensure Points Processed**:
   ```python
   def action_pos_order_paid(self):
       # Process points before marking as paid
       self._process_loyalty_points()
       return super().action_pos_order_paid()
   ```

## 5. Referral Usage Tracking

### Problem Description

Referral code usage tidak properly tracked, bisa digunakan lebih dari `max_referral_usage`.

### Root Causes

#### Cause 1: No Atomic Decrement

**Location**: `loyalty_card.py`

**Code**:
```python
# No lock on remaining_referral_usage
card.remaining_referral_usage -= 1
```

**Problem**: Race condition saat concurrent usage

### Mitigation

1. **Atomic Decrement**:
   ```python
   # Use database-level decrement
   self.env.cr.execute(
       "UPDATE loyalty_card SET remaining_referral_usage = remaining_referral_usage - 1 WHERE id = %s AND remaining_referral_usage > 0",
       (card_id,)
   )
   ```

2. **Validation Before Use**:
   ```python
   def use_referral_code(self, code):
       card = self._get_card_by_code(code)
       if card.remaining_referral_usage <= 0:
           raise ValidationError("Referral code usage limit reached")
       # Use code
   ```

## Data Integrity Checklist

- [ ] Point balance consistency validation
- [ ] Membership product validation
- [ ] Discount total validation
- [ ] Order state validation
- [ ] Referral usage validation
- [ ] Database constraints
- [ ] Atomic operations
- [ ] Transaction safety

Lihat dokumentasi berikutnya:
- `../mitigation/` - Solusi untuk masalah integritas data
