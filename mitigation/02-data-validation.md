# Validasi Data

## Overview

Dokumen ini menjelaskan strategi validasi data untuk memastikan data integrity.

## Validation Strategies

### 1. Input Validation

- Validate semua input sebelum process
- Type checking
- Range validation
- Business rule validation

### 2. Business Rule Validation

- Point balance >= 0
- Discount <= price
- Membership consistency
- Order state validation

### 3. Data Consistency Validation

- Card points vs history balance
- Point moves vs history
- Membership consistency

Lihat dokumentasi berikutnya:
- `03-transaction-safety.md` - Transaction safety
- `06-phase1-detailed-fixes.md` - Detail fixes dengan validation
