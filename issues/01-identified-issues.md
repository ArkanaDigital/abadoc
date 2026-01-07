# Daftar Masalah yang Teridentifikasi

## Kategori Masalah

Masalah dikelompokkan menjadi beberapa kategori berdasarkan jenis dan dampaknya:

1. **Data Integrity Issues** - Masalah integritas data
2. **Race Condition Issues** - Masalah concurrency
3. **Calculation Errors** - Kesalahan perhitungan
4. **Sync Issues** - Masalah sinkronisasi
5. **Business Logic Errors** - Kesalahan logika bisnis

## Priority Classification

- **P0 - Critical**: Data corruption, financial impact
- **P1 - High**: Functional issues, user impact
- **P2 - Medium**: Edge cases, minor inconsistencies
- **P3 - Low**: Cosmetic, optimization

## P0 - Critical Issues

### Issue #1: Point Balance Inconsistency

**Priority**: P0 - Critical

**Description**: 
Point balance di `loyalty.card.points` tidak konsisten dengan `loyalty.point.history.points_balance`.

**Location**: 
- `asd_aba_loyalty/models/loyalty_point_move.py`
- `asd_aba_loyalty/models/loyalty_card.py`

**Root Cause**:
1. Card points diupdate manual di beberapa tempat
2. History balance dihitung incremental dari previous history
3. Race condition saat concurrent point updates
4. Tidak ada validation untuk consistency

**Impact**:
- Customer melihat balance salah
- Point redemption bisa gagal atau double
- Financial loss
- Customer complaints

**Frequency**: High (terjadi sering di production)

**Evidence**:
```python
# Di loyalty_point_move.py line 167-169
# loyalty_card.write({'points': latest_point})  # COMMENTED OUT!
# Balance tidak diupdate dari point moves
```

### Issue #2: FEFO Point Consumption Race Condition

**Priority**: P0 - Critical

**Description**:
Concurrent orders bisa consume point move yang sama, menyebabkan negative balance atau double consumption.

**Location**: 
`asd_aba_loyalty/models/loyalty_point_move.py::handle_deducted_points()`

**Root Cause**:
1. Search point moves tanpa database lock
2. Multiple orders read same point move
3. Update bisa overwrite satu sama lain
4. No transaction isolation

**Impact**:
- Points bisa digunakan lebih dari available
- Negative balance bisa terjadi
- Financial loss
- Data corruption

**Frequency**: Medium (terjadi saat high concurrency)

**Evidence**:
```python
# Line 216-237: Search tanpa lock
point_move = self.search([
    ('state', '=', 'active'),
    ('loyalty_card_id', '=', loyalty_card_id),
    ('membership_product_id', '=', membership_product_id)
], order='expiration_date', limit=1)

# No lock, bisa diambil oleh concurrent request
```

### Issue #3: Membership Mismatch di Order Processing

**Priority**: P0 - Critical

**Description**:
Order dibuat dengan membership level A, tapi membership berubah ke B sebelum order confirmed. Points dihitung untuk level A (salah).

**Location**: 
- `asd_pos_customize/models/pos_orders.py::create_from_ui()`
- `asd_aba_loyalty/models/pos_order.py::_record_point_history()`

**Root Cause**:
1. Order `membership_product_id` di-set saat order created
2. Membership evaluation terjadi setelah order
3. Membership bisa berubah sebelum point calculation
4. Point calculation menggunakan order's membership, bukan current

**Impact**:
- Points dihitung untuk membership level yang salah
- Customer dapat points lebih sedikit atau lebih banyak
- Membership upgrade/downgrade tidak akurat

**Frequency**: High (terjadi saat membership change)

**Evidence**:
```python
# pos_orders.py line 781
membership_product_id = partner.membership_product_id.id
# Ini bisa berbeda dengan order.membership_product_id
```

## P1 - High Priority Issues

### Issue #4: Discount Recalculation Race Condition

**Priority**: P1 - High

**Description**:
`_recompute_discount()` dipanggil multiple times, bisa overwrite discount yang sudah dihitung.

**Location**: 
`asd_pos_customize/models/pos_orders.py::_recompute_discount()`

**Root Cause**:
1. Method dipanggil dari multiple places
2. Tidak ada lock atau flag untuk prevent re-entry
3. Discount fields di-reset setiap call
4. Tidak atomic

**Impact**:
- Discount hilang atau double
- Wrong order total
- Customer overcharge atau undercharge

**Frequency**: Medium

**Evidence**:
```python
# Line 407-414: Reset semua discount fields
for line in pos_order.lines:
    line.write({
        'discount_prog': 0,
        'discount_prog_all': 0,
        'disc_voucher': 0,
        'disc_promo_code': 0
    })
# Jika dipanggil concurrent, bisa reset discount yang sedang dihitung
```

### Issue #5: Discount Accumulation Error

**Priority**: P1 - High

**Description**:
Discount dihitung dari price yang sudah include discount sebelumnya, menyebabkan discount lebih besar dari seharusnya.

**Location**: 
`asd_pos_customize/models/pos_orders.py::calculate_discount_percentage()`

**Root Cause**:
1. Discount calculation menggunakan `price_subtotal_incl`
2. Price sudah include previous discounts
3. Tidak ada validation untuk max discount
4. Multiple discounts bisa overlap

**Impact**:
- Discount lebih besar dari seharusnya
- Negative order total bisa terjadi
- Financial loss

**Frequency**: Medium

**Evidence**:
```python
# Line 358-376
price = rec_line.price_subtotal_incl  # Already includes discounts!
disc_val = rec_line.discount_prog + (((price + existing_disc) / total_count) * disc)
# Discount dihitung dari price yang sudah didiscount
```

### Issue #6: Offline/Online Sync Issues

**Priority**: P1 - High

**Description**:
Orders dibuat offline tidak sync dengan benar, menyebabkan point calculation duplicate atau missing.

**Location**: 
`asd_pos_customize/models/pos_orders.py::create_from_ui()`

**Root Cause**:
1. Offline orders processed differently
2. Point calculation untuk offline orders di `create_from_ui()`
3. Point calculation untuk online orders di `_process_record_loyalty_point()`
4. Bisa terjadi double calculation atau missing

**Impact**:
- Points tidak terhitung atau double
- Data inconsistency
- Customer complaints

**Frequency**: Medium (terjadi saat offline mode)

**Evidence**:
```python
# Line 855-890: Offline order processing
if(order['data'].get('pos_state')=="offline"):
    # Process points here
    loyalty_point_move_obj.handle_added_points(**move_data)
# But online orders processed in _process_record_loyalty_point()
# Could cause double or missing
```

## P2 - Medium Priority Issues

### Issue #7: First Transaction Flag Inconsistency

**Priority**: P2 - Medium

**Description**:
`is_first_transaction` flag tidak konsisten, multiple orders bisa marked as first.

**Location**: 
`asd_pos_customize/models/pos_orders.py::_compute_is_first_transaction()`

**Root Cause**:
1. Check menggunakan view `pos_order_first`
2. View tidak update real-time
3. Race condition saat concurrent first orders
4. No atomic flag update

**Impact**:
- Multiple first transaction flags
- Wrong first transaction bonus
- Data inconsistency

**Frequency**: Low

### Issue #8: Birthday Transaction Validation

**Priority**: P2 - Medium

**Description**:
Birthday transaction validation bisa bypass jika ada refund.

**Location**: 
`asd_pos_customize/models/pos_orders.py::create_from_ui()`

**Root Cause**:
1. Check existing birthday order
2. Skip jika order is_refunded
3. Tapi tidak check jika refund order itself
4. Bisa claim multiple times

**Impact**:
- Multiple birthday bonuses
- Financial loss

**Frequency**: Low

**Evidence**:
```python
# Line 829-839
existing_birthday_order = self.env['pos.order'].search([
    ('partner_id', '=', partner.id),
    ('birthday_transaction', '=', True),
    # ... date range
], limit=1, order='date_order desc')

if existing_birthday_order:
    if not existing_birthday_order.is_refunded:
        raise ValidationError(...)
# Tidak check jika current order adalah refund dari birthday order
```

### Issue #9: Referral Code Usage Tracking

**Priority**: P2 - Medium

**Description**:
Referral code usage tidak properly tracked, bisa digunakan lebih dari limit.

**Location**: 
`asd_aba_loyalty/models/loyalty_card.py`

**Root Cause**:
1. `remaining_referral_usage` decremented tanpa lock
2. Race condition saat concurrent usage
3. Tidak ada validation untuk limit

**Impact**:
- Referral code bisa digunakan lebih dari limit
- Financial loss

**Frequency**: Low

## P3 - Low Priority Issues

### Issue #10: Discount Field Naming Confusion

**Priority**: P3 - Low

**Description**:
Multiple discount fields dengan naming yang confusing.

**Fields**:
- `discount` (Odoo core)
- `discount_prog` (program specific)
- `discount_prog_all` (program all)
- `disc_voucher` (voucher)
- `disc_promo_code` (promo code)

**Impact**:
- Developer confusion
- Maintenance difficulty
- Potential bugs

### Issue #11: Missing Error Handling

**Priority**: P3 - Low

**Description**:
Banyak methods tidak memiliki proper error handling.

**Impact**:
- Errors tidak ter-handle dengan baik
- Difficult debugging
- Poor user experience

## Summary Statistics

- **Total Issues**: 11
- **P0 - Critical**: 3
- **P1 - High**: 3
- **P2 - Medium**: 3
- **P3 - Low**: 2

## Real-World Evidence

Berdasarkan analisa code, ditemukan evidence kuat untuk masalah-masalah ini:

### Evidence dari Code Analysis

1. **Point Balance Inconsistency**:
   - Ada automatic adjustment system (`membership_log.py`)
   - Quick fix untuk point calculation (`_recompute_membership_earn_point`)
   - FIXME comment: "this method should be not used, but for quick fix"

2. **FEFO Race Condition**:
   - Debug variables di production code (`debug=0, debug=1, debug=2, debug=3`)
   - Commented ValidationError dengan debug info
   - Multiple search paths tanpa lock

3. **Membership Mismatch**:
   - Extensive error handling untuk membership issues
   - Adjustment system untuk fix membership-related errors
   - Multiple validation errors terkait membership

4. **Discount Issues**:
   - Long method `_recompute_discount()` (300+ lines)
   - Multiple discount fields
   - Complex calculation logic

5. **Offline/Online Sync**:
   - Different code paths untuk offline vs online
   - Extensive logging untuk track sync issues
   - Error handling untuk sync failures

Lihat dokumentasi berikutnya:
- [06-real-world-cases.md](06-real-world-cases.md) - **Kasus real-world dari analisa code**
- [07-code-analysis-summary.md](07-code-analysis-summary.md) - **Ringkasan analisa code**
- [02-data-integrity-issues.md](02-data-integrity-issues.md) - Detail data integrity issues
- [03-race-condition-issues.md](03-race-condition-issues.md) - Detail race condition issues
- [04-calculation-errors.md](04-calculation-errors.md) - Detail calculation errors
- `../mitigation/` - Solusi untuk setiap masalah
