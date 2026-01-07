# Arsitektur Sistem - Overview

## Pendahuluan

Sistem promosi dan membership ABA dibangun di atas Odoo 16 dengan beberapa modul custom yang saling terintegrasi. Sistem ini menangani kompleksitas tinggi dalam hal:

- Program loyalty dan membership
- Perhitungan poin yang kompleks
- Aplikasi diskon multi-layer
- Sinkronisasi offline/online
- Manajemen membership level

## Komponen Utama

### 1. Modul Core

#### `asd_aba_loyalty`
Modul utama untuk sistem loyalty dan membership:
- **Lokasi**: `addons-custom/aba/asd_aba_loyalty/`
- **Fungsi**: 
  - Manajemen program loyalty
  - Perhitungan dan tracking poin
  - Manajemen membership level
  - Referral program
- **Dependencies**: 
  - `pos_loyalty` (Odoo core)
  - `base_automation`
  - `membership_variable_period`

#### `asd_pos_customize`
Customisasi untuk Point of Sale:
- **Lokasi**: `addons-custom/aba/asd_pos_customize/`
- **Fungsi**:
  - Custom discount calculation
  - Order processing
  - Membership integration di POS
  - Visitor tracking
- **Dependencies**:
  - `point_of_sale`
  - `pos_loyalty`
  - `asd_aba_loyalty`

#### `asd_aba_reporting`
Reporting untuk promosi:
- **Lokasi**: `addons-custom/aba/asd_aba_reporting/`
- **Fungsi**:
  - Promotion result tracking
  - Analytics dan reporting
- **Dependencies**:
  - `asd_aba_loyalty`

### 2. Model Data Kunci

#### Loyalty Program (`loyalty.program`)
Extends Odoo core dengan:
- Membership product linkage
- Primary loyalty flag
- Birthday treatment
- Referral program support
- Program expiration
- Usage limits

#### Loyalty Card (`loyalty.card`)
Extends Odoo core dengan:
- Point tracking
- Membership association
- Referral code support

#### Loyalty Point Move (`loyalty.point.move`)
Custom model untuk:
- FEFO (First Expired First Out) point consumption
- Point expiration tracking
- Point transfer saat membership change

#### Loyalty Point History (`loyalty.point.history`)
Custom model untuk:
- Audit trail semua transaksi poin
- Balance tracking
- Membership status tracking

#### POS Order (`pos.order`)
Extends dengan:
- Membership product tracking
- Referral code usage
- Discount recomputation
- Point processing

## Arsitektur Integrasi

```
┌─────────────────────────────────────────────────────────┐
│                    Odoo Core Modules                     │
│  (pos_loyalty, point_of_sale, membership, etc.)        │
└────────────────────┬────────────────────────────────────┘
                     │
        ┌────────────┴────────────┐
        │                         │
┌───────▼────────┐      ┌─────────▼──────────┐
│ asd_aba_loyalty│      │ asd_pos_customize │
│                │      │                    │
│ - Programs     │◄─────┤ - Order Processing│
│ - Points       │      │ - Discount Calc    │
│ - Membership   │      │ - POS Integration   │
└───────┬────────┘      └─────────┬──────────┘
        │                         │
        └────────────┬────────────┘
                     │
        ┌────────────▼────────────┐
        │  asd_aba_reporting      │
        │  - Promotion Analytics  │
        └─────────────────────────┘
```

## Data Flow

### Order Processing Flow
1. POS Order dibuat di frontend
2. `asd_pos_customize` memproses order
3. Discount dihitung via `_recompute_discount`
4. Points dihitung dan dikirim ke `asd_aba_loyalty`
5. `loyalty.point.move` dan `loyalty.point.history` diupdate
6. Membership status dievaluasi
7. Order dikonfirmasi

### Point Calculation Flow
1. Order amount dihitung
2. Rule matching untuk program loyalty
3. Point earning dihitung berdasarkan rule
4. Point redemption dihitung jika ada
5. Point move dibuat dengan FEFO logic
6. Point history direkam
7. Loyalty card balance diupdate

## Teknologi Stack

- **Backend**: Python 3.x, Odoo 16 Framework
- **Frontend**: JavaScript (OWL Framework)
- **Database**: PostgreSQL
- **ORM**: Odoo ORM
- **API**: XML-RPC, JSON-RPC

## Pola Desain yang Digunakan

1. **Inheritance Pattern**: Extends Odoo core models
2. **Service Layer**: Business logic di model methods
3. **Event-driven**: Menggunakan Odoo signals dan automation
4. **State Machine**: Point move states (active, depleted, expired)

## Keterbatasan Arsitektur Saat Ini

1. **Monolithic**: Semua logic di satu codebase Python
2. **Tight Coupling**: Modul saling bergantung erat
3. **No Separation of Concerns**: Business logic tercampur dengan data access
4. **Limited Scalability**: Sulit untuk scale horizontal
5. **Error Handling**: Tidak konsisten di seluruh modul
6. **Transaction Safety**: Race conditions pada concurrent orders

## Next Steps

Lihat dokumentasi berikutnya:
- `02-module-structure.md` - Detail struktur modul
- `03-data-model.md` - Detail model data
- `04-integration-points.md` - Titik integrasi
