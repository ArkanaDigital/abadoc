# Flow Proses Refund

## Overview

Flow refund mencakup proses dari refund order creation hingga point reversal dan membership re-evaluation.

## Flow Diagram

```
┌─────────────────┐
│ Create Refund   │
│ Order           │
└────────┬────────┘
         │
    ┌────▼────────┐
    │ Find        │
    │ Original    │
    │ Order       │
    └────┬────────┘
         │
    ┌────▼────────┐
    │ Reverse     │
    │ Points      │
    └────┬────────┘
         │
    ┌────▼────────┐
    │ Re-evaluate │
    │ Membership  │
    └────┬────────┘
         │
    ┌────▼────────┐
    │ Update      │
    │ Point       │
    │ History     │
    └─────────────┘
```

## Detailed Flow

### Phase 1: Refund Order Creation

**Location**: `asd_pos_customize/models/pos_orders.py::_compute_order_name()`

1. **Create Refund Order**
   ```python
   refund_order = pos.order.create({
       'refunded_order_ids': [(4, original_order.id)],
       'partner_id': original_order.partner_id.id,
       ...
   })
   ```

### Phase 2: Point Reversal

**Location**: `asd_pos_customize/models/pos_orders.py::_compute_order_name()`

1. **Find Point History**
   ```python
   point_history = loyalty.point.history.search([
       ('origin', '=', original_order.name)
   ])
   ```

2. **Reverse Points**
   ```python
   for point in point_history:
       if point.points_amount > 0:
           # Reverse earning
           points_to_consume = -point.points_amount
           loyalty.point.move.handle_deducted_points(
               points_amount=points_to_consume,
               ...
           )
       else:
           # Reverse redemption
           points_to_add = abs(point.points_amount)
           loyalty.point.move.handle_added_points(
               points_amount=points_to_add,
               ...
           )
   ```

### Phase 3: Membership Re-evaluation

**Location**: `asd_pos_customize/models/pos_orders.py::_execute_refund_condition()`

1. **Calculate New Spending**
   ```python
   total_spending = sum(
       partner.pos_order_ids.filtered(
           lambda o: o.state in ['paid', 'done']
       ).mapped('amount_total')
   ) - refund_order.amount_total
   ```

2. **Check Membership Level**
   ```python
   if total_spending < membership_threshold:
       # Downgrade membership
       partner.action_member_evaluate()
   ```

## Known Issues

### Issue 1: Point Reversal Race Condition

**Problem**: Concurrent refunds bisa cause issues

**Impact**: Points tidak ter-reverse dengan benar

### Issue 2: Membership Downgrade Timing

**Problem**: Membership downgrade bisa terjadi terlalu cepat

**Impact**: Customer kehilangan benefits

## Recommended Improvements

1. **Atomic Refund**: Use transaction untuk ensure consistency
2. **Validation**: Validate refund eligibility
3. **Grace Period**: Add grace period untuk membership downgrade

Lihat dokumentasi berikutnya:
- [../mitigation/06-phase1-detailed-fixes.md](../mitigation/06-phase1-detailed-fixes.md) - Detail fixes
- [../issues/02-data-integrity-issues.md](../issues/02-data-integrity-issues.md) - Data integrity issues
