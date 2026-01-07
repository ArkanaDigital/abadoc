# Kesalahan Perhitungan

## Overview

Dokumen ini menjelaskan detail semua kesalahan perhitungan yang teridentifikasi.

## Calculation Errors

### Issue 1: Point Calculation dengan Redeem Line

**Location**: `asd_aba_loyalty/models/pos_order.py::_recompute_membership_earn_point()`

**Problem**: 
Point tidak dihitung dengan benar jika ada redeem line.

**Evidence**:
```python
# FIXME: this method should be not used, but for quick fix
# the problem is that the point is not calculated correctly,
# if there is a redeem line even just some cases
```

**Impact**: Customer dapat points lebih sedikit

### Issue 2: Discount Accumulation Error

**Location**: `asd_pos_customize/models/pos_orders.py::calculate_discount_percentage()`

**Problem**: 
Discount dihitung dari price yang sudah didiscount.

**Impact**: Over-discount

### Issue 3: Balance Calculation Inconsistency

**Location**: Multiple locations

**Problem**: 
Balance calculated dari `point_moves` vs `point_history` bisa berbeda.

**Impact**: Wrong balance

Lihat dokumentasi berikutnya:
- `05-sync-issues.md` - Sync issues
- `../mitigation/06-phase1-detailed-fixes.md` - Detail fixes
