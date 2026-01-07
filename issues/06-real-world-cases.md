# Kasus Real-World dari Analisa Code

## Overview

Dokumen ini menganalisis kasus-kasus aktual yang terjadi berdasarkan:
1. Code comments (FIXME, TODO)
2. Error handling patterns
3. Adjustment/fixing mechanisms
4. Logging statements
5. Validation errors

## Kasus 1: Point Calculation Error dengan Redeem Line

### Evidence dari Code

**Location**: `asd_aba_loyalty/models/pos_order.py:637-648`

```python
def _recompute_membership_earn_point(self, program):
    """
    Recompute membership earning point for quick fix point calculation
    Currently this method only for program type "loyalty"
    @param program: loyalty.program
    @returns: int
    """
    # FIXME: this method should be not used, but for quick fix point calculation
    # the problem is that the point is not calculated correctly,
    # if there is a redeem line even just some cases
    # this method implement in asd_pos_customize
    return 0
```

### Analisa Masalah

**Problem**: 
Point tidak dihitung dengan benar jika ada redeem line di order. Ini adalah **quick fix** yang seharusnya tidak digunakan, tapi masih digunakan karena masalah belum teratasi.

**Root Cause**:
1. Point calculation logic tidak handle redeem line dengan benar
2. Redeem line mempengaruhi earning point calculation
3. Logic di `asd_pos_customize` berbeda dengan `asd_aba_loyalty`

**Impact**:
- Customer dapat points lebih sedikit dari seharusnya
- Atau points lebih banyak (double earning)
- Data inconsistency

**Frequency**: Sering terjadi (ada quick fix berarti sering terjadi)

### Solusi Saat Ini

Quick fix dengan method `_recompute_membership_earn_point()` di `asd_pos_customize` yang override method di `asd_aba_loyalty`.

**Masalah dengan Solusi Saat Ini**:
- Quick fix, bukan permanent solution
- Logic terpisah di 2 modul berbeda
- Tidak konsisten

## Kasus 2: Automatic Point Adjustment System

### Evidence dari Code

**Location**: `asd_pos_customize/models/membership_log.py:293-407`

Sistem adjustment otomatis untuk memperbaiki point calculation yang salah:

```python
def _process_log_adjustment_fixing(self, pos_order, result):
    """
    Automatic adjustment untuk memperbaiki point calculation yang salah
    """
    # Calculate current points
    current_loyalty_point = sum(
        self.env['loyalty.point.history'].search([
            ('partner_id', '=', partner.id)
        ]).mapped('points_amount'))
    
    # Calculate existing earn/redeem untuk order ini
    existing_earn = sum(
        self.env['loyalty.point.history'].search([
            ('partner_id', '=', partner.id), 
            ('name', 'ilike', pos_order.name), 
            ('name', 'ilike', 'Added')
        ]).mapped('points_amount'))
    
    existing_redeem = abs(sum(
        self.env['loyalty.point.history'].search([
            ('partner_id', '=', partner.id), 
            ('name', 'ilike', pos_order.name), 
            ('name', 'ilike', 'Deducted')
        ]).mapped('points_amount')))
    
    # Calculate needed adjustment
    needed_earn = self.earn_point - existing_earn
    needed_redeem = self.redeem_point - existing_redeem
```

### Analisa Masalah

**Problem**: 
Ada sistem **automatic adjustment** yang berarti point calculation sering salah dan perlu diperbaiki manual/otomatis.

**Root Cause**:
1. Point calculation tidak akurat dari awal
2. Ada discrepancy antara expected dan actual points
3. Perlu adjustment untuk fix

**Impact**:
- Operational overhead (perlu adjustment)
- Customer complaints
- Data integrity issues
- Manual intervention required

**Frequency**: Sering (ada sistem khusus untuk handle ini)

### Pattern yang Terlihat

1. **Order dibuat** → Point dihitung
2. **Point salah** → Terdeteksi via membership_log
3. **Adjustment dibuat** → Auto-fix point
4. **Point diperbaiki** → Balance correct

Ini menunjukkan bahwa **point calculation dari awal tidak reliable**.

## Kasus 3: Membership Log Error Handling

### Evidence dari Code

**Location**: `asd_pos_customize/models/membership_log.py`

Banyak error handling untuk berbagai skenario:

```python
# Line 32, 60, 158, 161, 194, 215, 218, 251, 285, 396, 412, 436, 453
result['error'] = True/False
```

### Analisa Masalah

**Problem**: 
Banyak error cases yang perlu di-handle, menunjukkan sistem tidak robust.

**Error Scenarios**:
1. Order tidak ditemukan
2. Point history tidak ditemukan
3. Adjustment sudah ada
4. Partner tidak ditemukan
5. Calculation error

**Impact**:
- System tidak reliable
- Banyak edge cases
- Error handling complexity tinggi

## Kasus 4: Birthday Transaction Validation

### Evidence dari Code

**Location**: `asd_pos_customize/models/pos_orders.py:824-843`

```python
if program.birthday_treatment:
    _logger.info('MEMBERSHIP LOG CHECK CATCH PROGRAM BIRTHDAY NO TRANSAKSI - %s - DATA : %s' % (pos_order.name, program.name))
    partner = pos_order.partner_id
    current_year = fields.Date.today().year

    existing_birthday_order = self.env['pos.order'].search([
        ('partner_id', '=', partner.id),
        ('birthday_transaction', '=', True),
        ('date_order', '>=', f'{current_year}-01-01'),
        ('date_order', '<=', f'{current_year}-12-31'),
        ('id', '!=', pos_order.id)
    ], limit=1, order='date_order desc')

    if existing_birthday_order:
        if not existing_birthday_order.is_refunded:
            raise ValidationError(f"Transaksi birthday untuk {partner.name} sudah pernah dilakukan tahun ini pada {existing_birthday_order.name}. Tidak valid silahkan transaksi ulang.")
```

### Analisa Masalah

**Problem**: 
Birthday transaction bisa di-claim multiple times jika ada refund.

**Root Cause**:
1. Check existing birthday order
2. Skip jika order is_refunded
3. Tapi tidak check jika current order adalah refund dari birthday order
4. Bisa claim multiple times via refund

**Impact**:
- Multiple birthday bonuses
- Financial loss
- Abuse potential

## Kasus 5: Point Processing Error Handling

### Evidence dari Code

**Location**: `asd_aba_loyalty/models/pos_order.py:628-634`

```python
except Exception as err:
    _logger.error(
        f"MEMBERSHIP PROCESS POINT ERROR: "
        f"Member: {partner.name}, "
        f"Order: {receipt_number}, "
        f"err: {err}"
    )
```

### Analisa Masalah

**Problem**: 
Point processing sering error dan perlu extensive error handling.

**Error Cases**:
1. Order tidak ditemukan
2. Partner tidak match
3. Loyalty card tidak ditemukan
4. Point calculation error
5. Database error

**Impact**:
- Points tidak ter-process
- Data inconsistency
- Manual intervention needed

## Kasus 6: Offline Order Point Processing

### Evidence dari Code

**Location**: `asd_pos_customize/models/pos_orders.py:855-890`

```python
if(order['data'].get('pos_state')=="offline"):
    for coupon_id, coupon_change in order['data'].get('couponPointChanges').items():
        # Process points untuk offline orders
        if(coupon_change['program_id'] == loyalty_program.id):
            loyalty_point_move_obj = self.env['loyalty.point.move'].sudo()
            # ... process points ...
```

### Analisa Masalah

**Problem**: 
Offline orders processed differently dari online orders, bisa menyebabkan:
1. Double point calculation
2. Missing point calculation
3. Inconsistency

**Root Cause**:
- Online orders: Processed via `_process_record_loyalty_point()`
- Offline orders: Processed di `create_from_ui()`
- Different code paths = different behavior

**Impact**:
- Data inconsistency
- Points missing atau double
- Hard to debug

## Kasus 7: Debug Variables di Production Code

### Evidence dari Code

**Location**: `asd_aba_loyalty/models/loyalty_point_move.py:213-238`

```python
debug=0
iteration_counter = 0
while points_to_consume > 0:
    lifetime_point_move = self.search([...])
    if lifetime_point_move:
        point_move = lifetime_point_move
        debug=1
    elif source_document:
        point_move = self.search([...])
        debug=2
    else:
        point_move = self.search([...])
        debug=3
```

### Analisa Masalah

**Problem**: 
Ada debug variables di production code yang menunjukkan:
1. Logic complex dan perlu debugging
2. Multiple code paths untuk same operation
3. Tidak yakin logic mana yang benar

**Impact**:
- Code quality rendah
- Hard to maintain
- Potential bugs

## Kasus 8: Commented Out Validation Errors

### Evidence dari Code

Banyak commented out ValidationError yang menunjukkan:
1. Masalah pernah terjadi
2. Validation di-disable untuk quick fix
3. Masalah belum teratasi

**Locations**:
- `loyalty_point_move.py:111, 112, 147, 150, 160, 254, 279`
- `membership_log.py:353-376, 60, 194, 353`

### Analisa Masalah

**Problem**: 
Validation di-comment out berarti:
1. Validation pernah block operations
2. Di-disable untuk allow operations
3. Masalah underlying belum fixed

**Impact**:
- Data integrity compromised
- Potential data corruption
- Hard to trace issues

## Summary Kasus Real-World

### Frequency Analysis

| Kasus | Frequency | Severity | Status |
|-------|-----------|----------|--------|
| Point calculation error | Very High | High | Quick fix applied |
| Automatic adjustment needed | High | High | System exists |
| Error handling complexity | High | Medium | Extensive handling |
| Birthday transaction abuse | Medium | Medium | Partial fix |
| Point processing errors | High | High | Error handling |
| Offline/online sync | High | High | Different paths |
| Debug code in production | Medium | Low | Code quality |
| Commented validations | Medium | High | Data integrity risk |

### Root Causes

1. **Point Calculation Logic**: Tidak robust, banyak edge cases
2. **Concurrency Issues**: Race conditions tidak di-handle
3. **Data Consistency**: Tidak ada proper validation
4. **Code Quality**: Quick fixes, debug code, commented validations
5. **Architecture**: Logic terpisah di multiple modules

### Recommendations

1. **Immediate**: Fix point calculation logic
2. **Short-term**: Remove quick fixes, implement proper solution
3. **Medium-term**: Refactor code, remove debug code
4. **Long-term**: Migrate ke Rust engine dengan proper architecture

Lihat dokumentasi berikutnya:
- `../mitigation/01-immediate-fixes.md` - Solusi untuk kasus-kasus ini
- `../rust-engine/` - Solusi jangka panjang
