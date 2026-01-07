# Masalah Race Condition

## Overview

Dokumen ini menjelaskan detail semua masalah race condition yang teridentifikasi dalam sistem promosi dan membership.

## Race Condition Issues

### Issue 1: FEFO Point Consumption

**Location**: `asd_aba_loyalty/models/loyalty_point_move.py::handle_deducted_points()`

**Problem**: 
Concurrent orders bisa consume point move yang sama karena tidak ada database lock.

**Evidence**:
```python
# No FOR UPDATE lock
point_move = self.search([
    ('state', '=', 'active'),
    ('loyalty_card_id', '=', loyalty_card_id),
], order='expiration_date', limit=1)
```

**Impact**: Negative balance, double consumption

**Fix**: Use `SELECT FOR UPDATE SKIP LOCKED`

### Issue 2: Point Balance Update

**Location**: Multiple locations

**Problem**: 
Multiple concurrent updates bisa overwrite card points.

**Impact**: Balance inconsistency

**Fix**: Atomic update dengan SQL

### Issue 3: Discount Recalculation

**Location**: `asd_pos_customize/models/pos_orders.py::_recompute_discount()`

**Problem**: 
Method bisa dipanggil multiple times concurrently.

**Impact**: Discount loss atau double

**Fix**: Lock mechanism dengan flag

Lihat dokumentasi berikutnya:
- [04-calculation-errors.md](04-calculation-errors.md) - Calculation errors
- [../mitigation/06-phase1-detailed-fixes.md](../mitigation/06-phase1-detailed-fixes.md) - Detail fixes
