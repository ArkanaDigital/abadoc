# Model Data dan Relasi

## Entity Relationship Diagram (Simplified)

```
┌─────────────────┐
│ loyalty.program │
│  (extended)     │
└────────┬────────┘
         │ 1:N
         │
    ┌────▼────────┐      ┌──────────────┐
    │ loyalty.card│      │ loyalty.rule │
    │  (extended) │      │  (extended)  │
    └────┬────────┘      └──────┬───────┘
         │                      │
         │ 1:N                  │ 1:N
         │                      │
    ┌────▼──────────────────────▼──────┐
    │    loyalty.point.move             │
    │    (custom model)                │
    └────┬──────────────────────────────┘
         │ 1:N
         │
    ┌────▼──────────────────┐
    │ loyalty.point.history │
    │    (custom model)     │
    └───────────────────────┘
         │
         │ N:1
         │
    ┌────▼────────┐
    │ res.partner │
    │  (extended) │
    └────┬────────┘
         │ 1:N
         │
    ┌────▼────────┐
    │ pos.order   │
    │  (extended) │
    └────┬────────┘
         │ 1:N
         │
    ┌────▼────────────┐
    │ pos.order.line  │
    │   (extended)    │
    └─────────────────┘
```

## Model Detail

### 1. `loyalty.program` (Extended)

**Base Model**: `loyalty.program` (Odoo core)

**Extended Fields**:
```python
membership_product_id = Many2one('product.product')
is_primary_loyalty = Boolean
birthday_treatment = Boolean
expired_interval = Integer
expired_interval_type = Selection
date_from = Datetime
date_to = Datetime
aba_next_order_coupon = Boolean
program_type = Selection (added 'referral')
max_referral_usage = Integer
has_generate_referral = Boolean
limit_selection = Selection
limit_per_customer = Boolean
max_usage_per_customer = Integer
loyalty_member_limit_ids = Many2many
```

**Key Relationships**:
- `membership_product_id` → `product.product` (membership level)
- `coupon_ids` → `loyalty.card` (1:N)
- `rule_ids` → `loyalty.rule` (1:N)
- `reward_ids` → `loyalty.reward` (1:N)

**Critical Constraints**:
- Unique constraint: `unique(program_type, membership_product_id, is_primary_loyalty)`
- Only one primary loyalty per membership product

### 2. `loyalty.card` (Extended)

**Base Model**: `loyalty.card` (Odoo core)

**Extended Fields**:
```python
partner_referral_code = Char
remaining_referral_usage = Integer
```

**Key Relationships**:
- `program_id` → `loyalty.program` (N:1)
- `partner_id` → `res.partner` (N:1)
- `point_move_ids` → `loyalty.point.move` (1:N via reverse)
- `point_history_ids` → `loyalty.point.history` (1:N via reverse)

**Critical Fields**:
- `points` - Current balance (computed from point moves)
- `code` - Unique card code
- `source_pos_order_id` - Order yang membuat card

### 3. `loyalty.point.move` (Custom Model)

**Purpose**: FEFO (First Expired First Out) point management

**Fields**:
```python
name = Char (required)
gain_date = Datetime (required)
expiration_date = Datetime (computed)
expired_interval = Integer
expired_interval_type = Selection
initial_point = Float (required)
remaining_point = Float (required)
loyalty_card_id = Many2one('loyalty.card', required)
partner_id = Many2one (related)
origin = Char (source document)
membership_product_id = Many2one('product.product', required)
state = Selection ['active', 'depleted', 'expired']
history_ids = One2many('loyalty.point.history')
```

**Key Relationships**:
- `loyalty_card_id` → `loyalty.card` (N:1)
- `membership_product_id` → `product.product` (N:1)
- `history_ids` → `loyalty.point.history` (1:N)

**Critical Logic**:
- FEFO consumption: Points consumed dari yang paling cepat expired
- Lifetime points: `expiration_date = False` diprioritaskan terakhir
- State management: active → depleted/expired

### 4. `loyalty.point.history` (Custom Model)

**Purpose**: Complete audit trail untuk semua transaksi poin

**Fields**:
```python
name = Char (required)
transaction_time = Datetime (required)
points_amount = Float (required)  # + untuk earn, - untuk redeem
points_balance = Float (required)  # Balance setelah transaksi
loyalty_card_id = Many2one('loyalty.card', required)
partner_id = Many2one (related, stored)
point_move_id = Many2one('loyalty.point.move')
origin = Char (indexed, source document)
membership_product_id = Many2one('product.product')
status_membership_product_id = Many2one('product.product')
```

**Key Relationships**:
- `loyalty_card_id` → `loyalty.card` (N:1)
- `partner_id` → `res.partner` (N:1)
- `point_move_id` → `loyalty.point.move` (N:1)

**Critical Logic**:
- `points_balance` selalu dihitung dari history sebelumnya
- `origin` digunakan untuk tracking order
- `status_membership_product_id` track membership saat transaksi

### 5. `pos.order` (Extended)

**Extended Fields**:
```python
membership_product_id = Many2one('product.product')
referral_code_used = Char
pos_state = Selection ['online', 'offline']
is_first_transaction = Boolean
birthday_transaction = Boolean
CouponPoint = Text (JSON)
recompute_disc = Boolean
```

**Key Relationships**:
- `partner_id` → `res.partner` (N:1)
- `lines` → `pos.order.line` (1:N)
- `session_id` → `pos.session` (N:1)

**Critical Methods**:
- `confirm_coupon_programs()` - Process coupons dan points
- `_record_point_history()` - Record point transactions
- `_recompute_discount()` - Recalculate discounts
- `_recompute_membership_earn_point()` - Recalculate earning

### 6. `pos.order.line` (Extended)

**Extended Fields**:
```python
promo_code = Char
disc_promo_code = Monetary
disc_voucher_code = Char
disc_voucher = Monetary
discount_prog = Monetary  # Program specific discount
discount_prog_all = Monetary  # Program all discount
giftcard_code = Char
amount_redeem = Monetary
price_subtotal_redeem = Monetary
price_unit_redeem = Monetary
```

**Key Relationships**:
- `order_id` → `pos.order` (N:1)
- `product_id` → `product.product` (N:1)
- `reward_id` → `loyalty.reward` (N:1)
- `coupon_id` → `loyalty.card` (N:1)

**Critical Logic**:
- Multiple discount fields untuk tracking sumber diskon
- `is_reward_line` flag untuk membedakan reward dari regular line

## Data Integrity Issues

### 1. Point Balance Consistency

**Problem**: `loyalty.card.points` bisa tidak sync dengan `loyalty.point.history.points_balance`

**Root Cause**:
- Card points diupdate manual di beberapa tempat
- History balance dihitung dari previous history
- Race condition saat concurrent updates

**Impact**: 
- Customer melihat balance salah
- Point redemption bisa gagal atau double

### 2. Membership Product Mismatch

**Problem**: `membership_product_id` di order berbeda dengan partner

**Root Cause**:
- Membership berubah saat order processing
- Order dibuat dengan membership lama
- Point move menggunakan membership yang salah

**Impact**:
- Point dihitung untuk membership level yang salah
- Upgrade/downgrade tidak akurat

### 3. Discount Field Conflicts

**Problem**: Multiple discount fields bisa overlap atau conflict

**Fields**:
- `discount` (Odoo core)
- `discount_prog` (program specific)
- `discount_prog_all` (program all)
- `disc_voucher` (voucher)
- `disc_promo_code` (promo code)

**Root Cause**:
- Discount calculation di `_recompute_discount()` complex
- Tidak ada validation untuk total discount
- Bisa terjadi double discount

### 4. FEFO Point Consumption Race Condition

**Problem**: Concurrent orders bisa consume point move yang sama

**Root Cause**:
- `handle_deducted_points()` search point moves tanpa lock
- Multiple orders bisa read same point move
- Update bisa overwrite satu sama lain

**Impact**:
- Point bisa digunakan lebih dari available
- Negative balance bisa terjadi

## Data Validation Gaps

### Missing Constraints

1. **Point Balance Validation**:
   - Tidak ada constraint untuk ensure `points_balance >= 0`
   - Tidak ada validation untuk `points_balance` consistency

2. **Discount Validation**:
   - Tidak ada max discount limit
   - Tidak ada validation untuk total discount <= price

3. **Membership Consistency**:
   - Tidak ada constraint untuk ensure order membership = partner membership
   - Tidak ada validation untuk membership change timing

4. **Point Move State**:
   - Tidak ada constraint untuk ensure `remaining_point >= 0`
   - Tidak ada validation untuk state transitions

## Recommended Data Model Improvements

1. **Add Database Constraints**:
   ```sql
   ALTER TABLE loyalty_point_history 
   ADD CONSTRAINT check_balance_non_negative 
   CHECK (points_balance >= 0);
   ```

2. **Add Computed Fields with Store**:
   - Store `loyalty.card.points` dari sum of active point moves
   - Add validation on write

3. **Add Audit Fields**:
   - `created_at`, `updated_at` untuk semua models
   - `created_by`, `updated_by` untuk tracking

4. **Normalize Discount Fields**:
   - Single discount calculation table
   - Link to order line dengan type

Lihat dokumentasi berikutnya:
- [04-integration-points.md](04-integration-points.md) - Titik integrasi
- `../flows/` - Flow proses
