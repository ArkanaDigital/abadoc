# Flow Aplikasi Diskon

## Overview

Flow aplikasi diskon mencakup proses dari reward selection hingga discount calculation dan application ke order lines.

## Flow Diagram

```
┌─────────────────┐
│ User Selects    │
│ Reward          │
└────────┬────────┘
         │
    ┌────▼────────┐
    │ Validate    │
    │ Reward      │
    └────┬────────┘
         │
    ┌────▼────────┐
    │ Create      │
    │ Reward Line │
    └────┬────────┘
         │
    ┌────▼────────┐
    │ Recompute   │
    │ Discount    │
    └────┬────────┘
         │
    ┌────▼────────┐
    │ Calculate   │
    │ Discount    │
    │ Amount      │
    └────┬────────┘
         │
    ┌────▼────────┐
    │ Apply to    │
    │ Order Lines │
    └────┬────────┘
         │
    ┌────▼────────┐
    │ Validate    │
    │ Total       │
    │ Discount    │
    └─────────────┘
```

## Detailed Flow

### Phase 1: Reward Selection

**Location**: Frontend (JavaScript)

1. **User Selects Reward**
   ```javascript
   // User clicks reward button
   this.applyReward(reward);
   ```

2. **Validate Eligibility**
   ```javascript
   // Check if customer eligible
   if (!this.isEligible(reward)) {
       this.showError("Not eligible");
       return;
   }
   ```

### Phase 2: Reward Line Creation

**Location**: `asd_pos_customize/models/pos_orders.py`

1. **Create Reward Line**
   ```python
   reward_line = pos.order.line.create({
       'order_id': order.id,
       'product_id': reward.reward_product_id.id,
       'reward_id': reward.id,
       'is_reward_line': True,
       'price_unit': reward.discount or reward.reward_product_id.list_price
   })
   ```

### Phase 3: Discount Recalculation

**Location**: `asd_pos_customize/models/pos_orders.py::_recompute_discount()`

1. **Reset All Discounts**
   ```python
   for line in order.lines:
       line.write({
           'discount_prog': 0,
           'discount_prog_all': 0,
           'disc_voucher': 0,
           'disc_promo_code': 0
       })
   ```

2. **Process Each Reward**
   ```python
   for reward_line in order.lines.filtered(lambda x: x.reward_id):
       reward = reward_line.reward_id
       program = reward.program_id
       
       if program.program_type == 'promotion':
           self._apply_promotion_discount(order, reward)
       elif program.program_type == 'coupons':
           self._apply_coupon_discount(order, reward)
   ```

### Phase 4: Discount Calculation

**Location**: `asd_pos_customize/models/pos_orders.py::calculate_discount_percentage()`

1. **Calculate Base Amount**
   ```python
   base_price = line.price_unit * line.qty
   total_count = sum(all_lines.mapped('price_subtotal_incl'))
   ```

2. **Calculate Discount**
   ```python
   discount_amount = (base_price / total_count) * discount_percentage
   ```

3. **Apply Discount**
   ```python
   line.write({'discount_prog': discount_amount})
   ```

### Phase 5: Validation

**Location**: `asd_pos_customize/models/pos_orders.py::_validate_total_discount()`

1. **Check Total Discount**
   ```python
   for line in order.lines:
       total_discount = (
           line.discount_prog +
           line.discount_prog_all +
           line.disc_voucher +
           line.disc_promo_code
       )
       
       if total_discount > line.price_subtotal_incl:
           raise ValidationError("Discount exceeds price")
   ```

## Known Issues

### Issue 1: Discount Accumulation Error

**Problem**: Discount dihitung dari price yang sudah didiscount

**Impact**: Over-discount

### Issue 2: Race Condition

**Problem**: Multiple calls bisa overwrite discount

**Impact**: Discount loss atau double

## Recommended Improvements

1. **Calculate from Base Price**: Always use base price, not discounted
2. **Lock Mechanism**: Prevent concurrent recalculation
3. **Validation**: Validate total discount before apply

Lihat dokumentasi berikutnya:
- [05-refund-flow.md](05-refund-flow.md) - Flow refund processing
- [../mitigation/06-phase1-detailed-fixes.md](../mitigation/06-phase1-detailed-fixes.md) - Detail fixes
