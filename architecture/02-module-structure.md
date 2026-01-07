# Struktur Modul dan Dependensi

## Struktur Modul `asd_aba_loyalty`

```
asd_aba_loyalty/
├── __init__.py
├── __manifest__.py
├── models/
│   ├── __init__.py
│   ├── loyalty_program.py          # Program loyalty dengan membership
│   ├── loyalty_rule.py              # Rule untuk earning points
│   ├── loyalty_reward.py             # Reward configuration
│   ├── loyalty_card.py               # Loyalty card management
│   ├── loyalty_point_move.py         # FEFO point consumption
│   ├── loyalty_point_history.py     # Point transaction history
│   ├── membership_line.py            # Membership line extension
│   ├── membership_category.py        # Membership category
│   ├── membership_level_path.py      # Membership upgrade/downgrade path
│   ├── pos_order.py                  # POS order integration
│   ├── pos_session.py                # POS session extension
│   ├── pos_config.py                 # POS config extension
│   ├── partner.py                     # Partner/member extension
│   ├── referral_usage_history.py     # Referral tracking
│   └── loyalty_member_limit.py       # Usage limit per member
├── controllers/
│   └── controllers.py                # API endpoints
├── wizards/
│   ├── loyalty_rule_multiplier.py    # Rule multiplier wizard
│   ├── loyalty_point_adjust.py       # Manual point adjustment
│   ├── loyalty_generate_wizard.py    # Generate loyalty cards
│   ├── referral_generate.py          # Generate referral codes
│   └── create_coupon_wizard.py       # Create coupon wizard
├── static/src/js/
│   ├── models.js                     # JS model extensions
│   ├── Loyalty.js                    # Loyalty logic frontend
│   ├── RewardButton.js               # Reward button component
│   ├── RedeemButton.js                # Redeem button component
│   └── PartnerLine.js                # Partner line component
└── views/
    └── [XML view files]
```

## Struktur Modul `asd_pos_customize`

```
asd_pos_customize/
├── __init__.py
├── __manifest__.py
├── models/
│   ├── __init__.py
│   ├── pos_orders.py                 # POS order extension (MAIN LOGIC)
│   ├── pos_order_line.py             # Order line extension
│   ├── pos_session.py                # Session extension
│   ├── pos_config.py                 # Config extension
│   ├── pos_payment.py                # Payment extension
│   ├── pos_visitor.py                # Visitor tracking
│   ├── membership_log.py             # Membership transaction log
│   ├── partner.py                    # Partner extension
│   └── [other models]
├── static/src/js/
│   ├── models.js                     # JS model extensions
│   ├── ProductScreen.js              # Product screen customization
│   ├── PaymentScreen.js              # Payment screen customization
│   └── [other JS files]
└── views/
    └── [XML view files]
```

## Dependensi Antar Modul

### Dependency Graph

```
┌─────────────────┐
│  Odoo Core      │
│  - pos_loyalty  │
│  - point_of_sale│
│  - membership   │
└────────┬────────┘
         │
    ┌────┴────┐
    │         │
┌───▼───┐ ┌──▼──────────┐
│ asd_  │ │ asd_pos_    │
│ aba_  │ │ customize   │
│loyalty│ │             │
└───┬───┘ └──┬──────────┘
    │        │
    └───┬────┘
        │
   ┌────▼──────────┐
   │ asd_aba_      │
   │ reporting     │
   └───────────────┘
```

### Detail Dependensi

#### `asd_aba_loyalty` Dependencies
```python
"depends": [
    'base',
    'stock',
    'pos_loyalty',              # Odoo core loyalty
    'base_automation',          # Automation rules
    'partner_contact_gender',    # Partner gender
    'partner_contact_birthdate', # Partner birthdate
    'membership_variable_period',# Variable membership period
    'asd_aba_operating_unit',   # Operating unit support
]
```

#### `asd_pos_customize` Dependencies
```python
"depends": [
    'base',
    'pos_hr',                   # POS HR integration
    'point_of_sale',           # Core POS
    'pos_loyalty',             # POS loyalty
    'hr',                      # HR module
    'product_expiry',          # Product expiration
    'asd_aba_loyalty',         # ABA loyalty module
]
```

## Key Methods dan Tanggung Jawab

### `asd_aba_loyalty/models/loyalty_program.py`
- `_is_loyalty_program_membership()` - Check jika program terkait membership
- `generate_program_referral_code()` - Generate referral codes
- `_program_type_default_values()` - Default values untuk program types

### `asd_aba_loyalty/models/loyalty_point_move.py`
- `handle_added_points()` - **KRITIS**: Handle point earning
- `handle_deducted_points()` - **KRITIS**: Handle point redemption (FEFO)
- `handle_transferred_points()` - Handle point transfer saat membership change
- `handle_reset_points()` - Reset points saat downgrade
- `_update_card_and_move_history()` - Update card balance

### `asd_pos_customize/models/pos_orders.py`
- `_recompute_discount()` - **KRITIS**: Recalculate semua discount
- `_recompute_membership_earn_point()` - Recalculate earning points
- `_recompute_membership_redeem_point()` - Recalculate redeem points
- `create_from_ui()` - **KRITIS**: Create order dari POS UI
- `_process_record_loyalty_point()` - Process point recording
- `_record_point_history()` - Record point history
- `_record_referral_history()` - Record referral usage

## Entry Points

### POS Frontend → Backend
1. **Order Creation**: `pos.order.create_from_ui()`
2. **Point Processing**: `pos.order._process_record_loyalty_point()`
3. **Discount Calculation**: `pos.order._recompute_discount()`

### Backend → Database
1. **Point Move**: `loyalty.point.move.create()`
2. **Point History**: `loyalty.point.history.create()`
3. **Card Update**: `loyalty.card.write()`

## Extension Points

### Odoo Core Extensions
- `loyalty.program` → Extended dengan membership fields
- `loyalty.card` → Extended dengan referral support
- `pos.order` → Extended dengan discount calculation
- `pos.order.line` → Extended dengan discount fields

### Custom Models
- `loyalty.point.move` - FEFO point management
- `loyalty.point.history` - Audit trail
- `membership.log` - Transaction logging
- `referral.usage.history` - Referral tracking

## Data Flow Antara Modul

### Order Processing
```
POS Frontend (JS)
    ↓
asd_pos_customize/models/pos_orders.py::create_from_ui()
    ↓
asd_pos_customize/models/pos_orders.py::_recompute_discount()
    ↓
asd_aba_loyalty/models/pos_order.py::confirm_coupon_programs()
    ↓
asd_aba_loyalty/models/loyalty_point_move.py::handle_added_points()
    ↓
asd_aba_loyalty/models/loyalty_point_move.py::handle_deducted_points()
    ↓
Database (PostgreSQL)
```

## Masalah Struktur Saat Ini

1. **Circular Dependencies**: Potensi circular dependency antara modul
2. **Fat Models**: Logic terlalu banyak di model methods
3. **No Service Layer**: Business logic langsung di model
4. **Tight Coupling**: Modul terlalu bergantung satu sama lain
5. **Mixed Concerns**: Data access, business logic, dan presentation tercampur

## Rekomendasi Perbaikan Struktur

1. **Service Layer**: Pisahkan business logic ke service layer
2. **Repository Pattern**: Abstraksi data access
3. **Event Bus**: Decouple modul dengan event-driven architecture
4. **API Gateway**: Centralized API untuk inter-module communication
5. **Dependency Injection**: Reduce tight coupling

Lihat dokumentasi berikutnya:
- `03-data-model.md` - Detail model data dan relasi
