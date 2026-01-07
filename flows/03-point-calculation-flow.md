# Flow Perhitungan Poin

## Overview

Flow perhitungan poin mencakup proses dari order creation hingga point earning dan redemption, termasuk FEFO (First Expired First Out) consumption logic.

## Flow Diagram

```
┌─────────────────┐
│ POS Order       │
│ Created         │
└────────┬────────┘
         │
    ┌────▼────────┐
    │ Calculate   │
    │ Earning     │
    │ Points      │
    └────┬────────┘
         │
    ┌────▼────────┐
    │ Calculate   │
    │ Redeem      │
    │ Points      │
    └────┬────────┘
         │
    ┌────▼────────┐
    │ FEFO        │
    │ Consumption │
    └────┬────────┘
         │
    ┌────▼────────┐
    │ Create      │
    │ Point Move  │
    └────┬────────┘
         │
    ┌────▼────────┐
    │ Create      │
    │ Point       │
    │ History     │
    └────┬────────┘
         │
    ┌────▼────────┐
    │ Update Card │
    │ Balance     │
    └─────────────┘
```

## Detailed Flow

### Phase 1: Order Processing

**Location**: `asd_pos_customize/models/pos_orders.py::create_from_ui()`

1. **Order Created**
   ```python
   order = pos.order.create(order_fields)
   ```

2. **Point Calculation Triggered**
   ```python
   if self._record_point_immediately():
       self._process_record_loyalty_point(...)
   else:
       # Process later in confirm_coupon_programs
   ```

### Phase 2: Earning Points Calculation

**Location**: `asd_pos_customize/models/pos_orders.py::_recompute_membership_earn_point()`

1. **Get Loyalty Program**
   ```python
   loyalty_program = loyalty.program.search([
       ('program_type', '=', 'loyalty'),
       ('membership_product_id', '=', membership_product_id),
       ('is_primary_loyalty', '=', True)
   ])
   ```

2. **Calculate Based on Rules**
   ```python
   for rule in program.rule_ids:
       if rule.reward_point_mode == 'order':
           points += rule.reward_point_amount
       elif rule.reward_point_mode == 'money':
           points += rule.reward_point_amount * amount_paid
       elif rule.reward_point_mode == 'unit':
           points += rule.reward_point_amount * qty
   ```

### Phase 3: Redeem Points Calculation

**Location**: `asd_pos_customize/models/pos_orders.py::_recompute_membership_redeem_point()`

1. **Find Redeem Lines**
   ```python
   redeem_lines = pos.order.line.search([
       ('order_id', '=', order.id),
       ('reward_id.program_id', '=', program.id),
       ('is_reward_line', '=', True),
       ('coupon_id', '=', card.id)
   ])
   ```

2. **Calculate Redeem Points**
   ```python
   redeem_points = sum(redeem_lines.mapped('points_cost'))
   ```

### Phase 4: FEFO Point Consumption

**Location**: `asd_aba_loyalty/models/loyalty_point_move.py::handle_deducted_points()`

1. **Find Point Moves (FEFO)**
   ```python
   # Priority: Expired first, then by expiration date, then lifetime
   point_move = search([
       ('state', '=', 'active'),
       ('loyalty_card_id', '=', card_id),
       ('membership_product_id', '=', membership_product_id)
   ], order='expiration_date ASC, gain_date ASC')
   ```

2. **Consume Points**
   ```python
   remaining = point_move.remaining_point - points_to_consume
   point_move.write({
       'remaining_point': max(0, remaining),
       'state': 'depleted' if remaining <= 0 else 'active'
   })
   ```

### Phase 5: Point History Creation

**Location**: `asd_aba_loyalty/models/loyalty_point_move.py`

1. **Create Point Move** (for earning)
   ```python
   point_move = loyalty.point.move.create({
       'initial_point': points_amount,
       'remaining_point': points_amount,
       'loyalty_card_id': card_id,
       'membership_product_id': membership_product_id,
       'origin': order.name
   })
   ```

2. **Create Point History**
   ```python
   latest_history = point_history.search([
       ('loyalty_card_id', '=', card_id)
   ], order='id desc', limit=1)
   
   new_balance = latest_history.points_balance + points_amount
   
   point_history.create({
       'points_amount': points_amount,
       'points_balance': new_balance,
       'loyalty_card_id': card_id,
       'point_move_id': point_move.id,
       'origin': order.name
   })
   ```

### Phase 6: Card Balance Update

**Location**: `asd_aba_loyalty/models/loyalty_point_move.py::_update_card_and_move_history()`

1. **Calculate Balance**
   ```python
   point_moves = search([
       ('state', '=', 'active'),
       ('partner_id', '=', partner.id)
   ])
   balance = sum(point_moves.mapped('remaining_point'))
   ```

2. **Update Card**
   ```python
   loyalty_card.write({'points': balance})
   ```

## Known Issues

### Issue 1: Balance Calculation Inconsistency

**Problem**: Balance calculated dari `point_moves` vs `point_history` bisa berbeda

**Impact**: Card points tidak akurat

### Issue 2: FEFO Race Condition

**Problem**: Concurrent orders bisa consume same point move

**Impact**: Negative balance atau double consumption

### Issue 3: Point Calculation dengan Redeem Line

**Problem**: Earning points tidak dihitung dengan benar jika ada redeem line

**Impact**: Customer dapat points lebih sedikit

## Recommended Improvements

1. **Single Source of Truth**: Always calculate dari `point_history.points_balance`
2. **Database Locking**: Use `SELECT FOR UPDATE` untuk FEFO
3. **Atomic Operations**: Use transactions untuk ensure consistency

Lihat dokumentasi berikutnya:
- `04-discount-application-flow.md` - Flow discount application
- `05-refund-flow.md` - Flow refund processing
- `../mitigation/06-phase1-detailed-fixes.md` - Detail fixes
