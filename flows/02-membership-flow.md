# Flow Proses Membership

## Overview

Flow membership mencakup proses dari registrasi member, tracking transaksi, evaluasi level membership, hingga upgrade/downgrade otomatis berdasarkan spending.

## Flow Diagram

```
┌─────────────────┐
│ Customer        │
│ Registration    │
└────────┬────────┘
         │
    ┌────▼────────┐
    │ Assign      │
    │ Membership  │
    │ Level       │
    └────┬────────┘
         │
    ┌────▼────────┐
    │ Create      │
    │ Loyalty     │
    │ Card        │
    └────┬────────┘
         │
    ┌────▼────────┐
    │ POS Order   │
    │ Transaction │
    └────┬────────┘
         │
    ┌────▼────────┐
    │ Calculate   │
    │ Points      │
    └────┬────────┘
         │
    ┌────▼────────┐
    │ Evaluate    │
    │ Membership  │
    │ Status      │
    └────┬────────┘
         │
    ┌────▼────────┐
    │ Upgrade/    │
    │ Downgrade?  │
    └────┬────────┘
         │
    ┌────▼────────┐
    │ Transfer    │
    │ Points      │
    └─────────────┘
```

## Detailed Flow

### Phase 1: Member Registration

**Location**: `asd_aba_loyalty/models/partner.py`

1. **Create Partner**
   ```python
   partner = res.partner.create({
       'name': 'Customer Name',
       'is_company': False,
       # ... other fields
   })
   ```

2. **Assign Initial Membership**
   ```python
   # Get lowest membership level
   member_level = membership.level.path.get_lowest_level(pos_config_id)
   partner.membership_product_id = member_level.membership_product_id
   ```

3. **Create Loyalty Card**
   ```python
   # Auto-created via automation or manually
   loyalty_program = loyalty.program.search([
       ('program_type', '=', 'loyalty'),
       ('membership_product_id', '=', partner.membership_product_id.id),
       ('is_primary_loyalty', '=', True)
   ])
   
   loyalty_card = loyalty.card.create({
       'program_id': loyalty_program.id,
       'partner_id': partner.id,
       'code': loyalty.card._generate_code(),
       'points': 0
   })
   ```

### Phase 2: Order Processing dengan Membership

**Location**: `asd_pos_customize/models/pos_orders.py::create_from_ui()`

1. **Order Creation dengan Partner**
   ```python
   order = pos.order.create({
       'partner_id': partner.id,
       'membership_product_id': partner.membership_product_id.id,
       # ... other fields
   })
   ```

2. **Capture Membership di Order**
   - `membership_product_id` di order = partner's membership saat order dibuat
   - Ini penting untuk tracking membership level saat transaksi

### Phase 3: Point Calculation

**Location**: `asd_aba_loyalty/models/pos_order.py::_record_point_history()`

1. **Calculate Earning Points**
   ```python
   # Get loyalty program
   loyalty_program = loyalty.program.search([
       ('program_type', '=', 'loyalty'),
       ('membership_product_id', '=', order.membership_product_id.id),
       ('is_primary_loyalty', '=', True)
   ])
   
   # Calculate based on rules
   earning_points = order._recompute_membership_earn_point(loyalty_program)
   ```

2. **Calculate Redeem Points**
   ```python
   # Check if there are redeem lines
   if order._has_redeem_line(loyalty_program, loyalty_card):
       redeem_points = order._recompute_membership_redeem_point(
           loyalty_program, loyalty_card
       )
   ```

3. **Record Points**
   ```python
   # Create point move
   loyalty.point.move.handle_added_points(
       points_amount=earning_points,
       loyalty_card_id=loyalty_card.id,
       membership_product_id=order.membership_product_id.id,
       transaction_model='pos.order',
       transaction_id=order.id
   )
   
   if redeem_points:
       loyalty.point.move.handle_deducted_points(
           points_amount=-redeem_points,
           # ... same params
       )
   ```

### Phase 4: Membership Evaluation

**Location**: `asd_aba_loyalty/models/partner.py::action_member_evaluate()`

1. **Calculate Total Spending**
   ```python
   # Get all orders untuk partner
   orders = partner.pos_order_ids.filtered(
       lambda o: o.state in ['paid', 'done']
   )
   total_spending = sum(orders.mapped('amount_total'))
   ```

2. **Check Membership Level Path**
   ```python
   # Get current membership level
   current_level = membership.level.path.get_membership_level(
       partner.membership_product_id
   )
   
   # Check if upgradeable
   if current_level.is_upgradable():
       # Check spending threshold
       next_level = current_level.get_next_level()
       if total_spending >= next_level.threshold:
           # Upgrade!
   ```

3. **Upgrade/Downgrade Decision**
   ```python
   # Check all levels
   for level in membership.level.path.search([]):
       if total_spending >= level.threshold:
           target_level = level
           break
   
   if target_level != partner.membership_product_id:
       # Change membership
       partner.action_assign_membership()
   ```

### Phase 5: Membership Change Processing

**Location**: `asd_aba_loyalty/models/loyalty_point_move.py::handle_transferred_points()`

1. **Detect Membership Change**
   ```python
   # Called from partner.action_assign_membership()
   if partner.membership_product_id != old_membership_product_id:
       loyalty.point.move.handle_transferred_points(
           partner, 
           prev_member_product=old_membership_product_id
       )
   ```

2. **Transfer Points**
   ```python
   # Get new loyalty program
   new_program = loyalty.program.search([
       ('program_type', '=', 'loyalty'),
       ('membership_product_id', '=', partner.membership_product_id.id),
       ('is_primary_loyalty', '=', True)
   ])
   
   # Get or create new loyalty card
   new_card = loyalty.card.search([
       ('program_id', '=', new_program.id),
       ('partner_id', '=', partner.id)
   ])
   
   if not new_card:
       new_card = loyalty.card.create({...})
   
   # Transfer point moves
   old_card.program_id = new_program.id
   # Update point moves membership_product_id
   ```

3. **Update Point Moves**
   ```python
   # Update all active point moves
   point_moves = loyalty.point.move.search([
       ('loyalty_card_id', '=', old_card.id),
       ('state', '=', 'active')
   ])
   
   point_moves.write({
       'membership_product_id': partner.membership_product_id.id
   })
   ```

## Critical Issues dalam Flow

### Issue 1: Membership Mismatch di Order

**Problem**: Order dibuat dengan membership lama, tapi membership berubah sebelum order confirmed

**Location**: `pos_orders.py::create_from_ui()`

**Scenario**:
1. Order dibuat dengan membership A
2. Customer spending mencapai threshold untuk membership B
3. Membership berubah ke B
4. Order confirmed dengan membership A
5. Points dihitung untuk membership A (salah!)

**Impact**: Points dihitung untuk membership level yang salah

### Issue 2: Point Transfer Race Condition

**Problem**: Multiple orders bisa trigger membership change simultaneously

**Location**: `loyalty_point_move.py::handle_transferred_points()`

**Scenario**:
1. Order 1 dan Order 2 processed concurrently
2. Both detect membership change
3. Both try to transfer points
4. Points bisa duplicate atau hilang

**Impact**: Point balance inconsistency

### Issue 3: Downgrade Point Loss

**Problem**: Saat downgrade, points tidak ditransfer dengan benar

**Location**: `loyalty_point_move.py::handle_transferred_points()`

**Scenario**:
1. Customer dengan membership A (high level)
2. Spending drop, downgrade ke membership B (low level)
3. Points di membership A tidak ditransfer
4. Customer kehilangan points

**Impact**: Customer kehilangan points yang sudah earned

### Issue 4: First Transaction Flag

**Problem**: `is_first_transaction` flag tidak konsisten

**Location**: `pos_orders.py::_compute_is_first_transaction()`

**Scenario**:
1. First transaction dibuat offline
2. Sync ke server
3. Another transaction dibuat sebelum sync
4. Both marked as first transaction

**Impact**: Multiple first transaction flags

## Membership Level Path Logic

**Location**: `asd_aba_loyalty/models/membership_level_path.py`

```python
class MembershipLevelPath(models.Model):
    _name = 'membership.level.path'
    
    membership_product_id = Many2one('product.product')
    threshold = Float  # Spending threshold untuk level ini
    sequence = Integer  # Order of levels
    
    def is_upgradable(self):
        # Check if there's a next level
        next_level = self.get_next_level()
        return next_level.exists()
    
    def is_downgradable(self):
        # Check if there's a previous level
        prev_level = self.get_previous_level()
        return prev_level.exists()
```

## Recommended Flow Improvements

1. **Atomic Membership Change**:
   - Lock partner selama membership evaluation
   - Use database transaction
   - Rollback on error

2. **Membership Snapshot di Order**:
   - Store membership snapshot saat order created
   - Don't change membership during order processing
   - Evaluate membership after order confirmed

3. **Point Transfer Validation**:
   - Validate point transfer completeness
   - Check point balance consistency
   - Log all transfers

4. **First Transaction Lock**:
   - Use database lock untuk first transaction check
   - Atomic flag update
   - Prevent race conditions

Lihat dokumentasi berikutnya:
- `03-point-calculation-flow.md` - Detail point calculation
- `05-refund-flow.md` - Refund processing
