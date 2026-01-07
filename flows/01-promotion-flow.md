# Flow Proses Promosi

## Overview

Flow promosi mencakup proses dari konfigurasi program promosi hingga aplikasi diskon di POS order. Sistem mendukung berbagai jenis promosi: discount, buy-x-get-y, special price, dan lainnya.

## Flow Diagram

```
┌─────────────────┐
│ Create Program  │
│  (Backend)      │
└────────┬────────┘
         │
    ┌────▼────────┐
    │ Configure   │
    │ Rules &     │
    │ Rewards     │
    └────┬────────┘
         │
    ┌────▼────────┐
    │ Assign to   │
    │ POS Config  │
    └────┬────────┘
         │
    ┌────▼────────┐
    │ POS Loads   │
    │ Programs    │
    └────┬────────┘
         │
    ┌────▼────────┐
    │ Customer    │
    │ Selects     │
    │ Products    │
    └────┬────────┘
         │
    ┌────▼────────┐
    │ System      │
    │ Matches     │
    │ Rules       │
    └────┬────────┘
         │
    ┌────▼────────┐
    │ Apply       │
    │ Rewards     │
    └────┬────────┘
         │
    ┌────▼────────┐
    │ Calculate   │
    │ Discounts   │
    └────┬────────┘
         │
    ┌────▼────────┐
    │ Finalize    │
    │ Order       │
    └─────────────┘
```

## Detailed Flow

### Phase 1: Program Configuration

**Location**: `asd_aba_loyalty/models/loyalty_program.py`

1. **Create Program**
   - User creates `loyalty.program` record
   - Set `program_type` (promotion, buy_x_get_y, etc.)
   - Configure `date_from` dan `date_to`
   - Set `pos_config_ids` untuk assign ke POS

2. **Configure Rules**
   - Create `loyalty.rule` records
   - Set conditions:
     - `minimum_amount` - Minimum order amount
     - `minimum_qty` - Minimum quantity
     - `product_ids` - Specific products
     - `membership_status` - Membership level required
   - Set earning:
     - `reward_point_amount` - Points to earn
     - `reward_point_mode` - order/money/unit

3. **Configure Rewards**
   - Create `loyalty.reward` records
   - Set reward type:
     - `discount` - Percentage or fixed discount
     - `product` - Free product
     - `specialprice` - Special price
     - `specialset` - Special set price
   - Set applicability:
     - `discount_applicability` - order/specific
     - `discount_product_ids` - Products untuk discount

### Phase 2: POS Order Processing

**Location**: `asd_pos_customize/models/pos_orders.py`

1. **Order Creation**
   ```python
   # Entry point: create_from_ui()
   order = pos.order.create(order_fields)
   ```

2. **Program Matching**
   - System loads active programs untuk POS config
   - Frontend (JS) matches rules terhadap order lines
   - Returns eligible rewards

3. **Reward Application**
   - User selects reward di POS
   - System creates reward line (`is_reward_line = True`)
   - Reward line linked ke `loyalty.reward`

### Phase 3: Discount Calculation

**Location**: `asd_pos_customize/models/pos_orders.py::_recompute_discount()`

**Critical Method**: This is where most issues occur!

```python
def _recompute_discount(self, pos_order):
    # 1. Reset all discount fields
    for line in pos_order.lines:
        line.write({
            'discount_prog': 0,
            'discount_prog_all': 0,
            'disc_voucher': 0,
            'disc_promo_code': 0
        })
    
    # 2. Process each reward line
    reward_lines = pos_order.lines.filtered(lambda x: x.reward_id)
    
    for rec_line in reward_lines:
        reward = rec_line.reward_id
        program = reward.program_id
        
        # 3. Handle different program types
        if program.program_type == 'promotion':
            # Calculate promotion discount
        elif program.program_type == 'coupons':
            # Calculate coupon discount
        elif program.program_type == 'promo_code':
            # Calculate promo code discount
```

**Discount Calculation Logic**:

1. **Order-level Discount**:
   ```python
   if reward.discount_applicability == 'order':
       total_count = calculate_total_count(all_lines)
       for line in all_lines:
           disc = calculate_discount_percentage(line, total_count, discount_amount)
           line.discount_prog = disc
   ```

2. **Product-specific Discount**:
   ```python
   if reward.discount_applicability == 'specific':
       discount_product_ids = reward.discount_product_ids.ids
       total_count = calculate_total_count(lines, discount_product_ids)
       for line in lines.filtered(lambda l: l.product_id.id in discount_product_ids):
           disc = calculate_discount_percentage(line, total_count, discount_amount)
           line.discount_prog = disc
   ```

3. **Special Price/Set**:
   ```python
   if reward.reward_type in ['specialprice', 'specialset']:
       # Complex logic untuk special pricing
       # Match products dengan rules
       # Apply special price
   ```

### Phase 4: Discount Distribution

**Method**: `calculate_discount_percentage()`

```python
def calculate_discount_percentage(self, rec_line, qty_count, disc, total_count, type, reward, disc_special=0):
    disc_val = 0
    price = rec_line.price_subtotal_incl
    
    # Accumulate existing discounts
    existing_disc = (rec_line.discount_prog_all + 
                     rec_line.discount_prog + 
                     rec_line.disc_voucher + 
                     rec_line.disc_promo_code)
    
    # Calculate proportional discount
    disc_val = existing_disc + (((price + existing_disc) / total_count) * disc)
    
    return disc_val
```

**Issues dengan Logic Ini**:
1. Discount dihitung berdasarkan `price_subtotal_incl` yang sudah include discount sebelumnya
2. Tidak ada validation untuk max discount
3. Multiple discounts bisa overlap
4. Rounding errors bisa terjadi

### Phase 5: Order Finalization

1. **Total Calculation**:
   ```python
   # Odoo core calculates:
   line.price_subtotal = (line.price_unit * line.qty) * (1 - line.discount/100)
   order.amount_total = sum(line.price_subtotal_incl for line in order.lines)
   ```

2. **Discount Fields Summary**:
   - `discount_prog` - Program specific discount
   - `discount_prog_all` - Program all discount
   - `disc_voucher` - Voucher/coupon discount
   - `disc_promo_code` - Promo code discount

3. **Validation** (Missing!):
   - Tidak ada check untuk total discount <= price
   - Tidak ada check untuk negative totals

## Known Issues dalam Flow

### Issue 1: Discount Recalculation Race Condition

**Problem**: `_recompute_discount()` dipanggil multiple times, bisa overwrite discount

**Location**: `pos_orders.py::_recompute_discount()`

**Impact**: Discount hilang atau double

### Issue 2: Discount Accumulation Error

**Problem**: Discount dihitung dari price yang sudah include discount sebelumnya

**Location**: `calculate_discount_percentage()`

**Impact**: Discount lebih besar dari seharusnya

### Issue 3: Multiple Discount Types Conflict

**Problem**: Tidak ada prioritas atau ordering untuk discount types

**Location**: `_recompute_discount()` loop

**Impact**: Discount tidak konsisten

### Issue 4: Special Price Calculation Complexity

**Problem**: Logic untuk special price sangat kompleks dan error-prone

**Location**: `_recompute_discount()` special price section

**Impact**: Wrong pricing untuk special products

## Recommended Flow Improvements

1. **Single Discount Calculation Pass**:
   - Calculate semua discounts dalam satu pass
   - Apply dengan priority order
   - Validate total discount

2. **Discount Validation**:
   - Max discount limit
   - Total discount <= price validation
   - Negative value prevention

3. **Atomic Discount Updates**:
   - Use database transaction
   - Lock order selama discount calculation
   - Rollback on error

4. **Discount Audit Trail**:
   - Log semua discount calculations
   - Track discount source
   - Enable discount history

Lihat dokumentasi berikutnya:
- [02-membership-flow.md](02-membership-flow.md) - Flow membership
- [03-point-calculation-flow.md](03-point-calculation-flow.md) - Flow perhitungan poin
- [04-discount-application-flow.md](04-discount-application-flow.md) - Detail discount application
