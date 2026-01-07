# Masalah Sinkronisasi Offline/Online

## Overview

Dokumen ini menjelaskan masalah sinkronisasi antara offline dan online orders.

## Sync Issues

### Issue 1: Different Code Paths

**Location**: 
- Offline: `asd_pos_customize/models/pos_orders.py::create_from_ui()` (line 855-890)
- Online: `asd_aba_loyalty/models/pos_order.py::_process_record_loyalty_point()`

**Problem**: 
Offline dan online orders processed dengan different logic.

**Impact**: 
- Points bisa double atau missing
- Data inconsistency

### Issue 2: No Idempotency Check

**Problem**: 
Tidak ada check apakah points sudah processed.

**Impact**: 
Double processing

### Issue 3: Sync Timing

**Problem**: 
Offline orders sync later, bisa cause timing issues.

**Impact**: 
Data inconsistency

## Recommended Fixes

1. **Unified Processing**: Single code path untuk offline dan online
2. **Idempotency**: Check jika sudah processed
3. **Coordination**: Proper state management

Lihat dokumentasi berikutnya:
- [06-real-world-cases.md](06-real-world-cases.md) - Real-world cases
- [../mitigation/06-phase1-detailed-fixes.md](../mitigation/06-phase1-detailed-fixes.md) - Detail fixes
