# Rust Engine Documentation

## Overview

Dokumentasi untuk Rust Promotion & Membership Engine - solusi enterprise-grade untuk mengatasi semua masalah di sistem Python saat ini.

## Struktur Dokumentasi

### Core Documentation
- `01-overview.md` - Overview dan benefits
- `02-architecture.md` - Arsitektur detail
- `03-database-architecture.md` - **Arsitektur database (standalone vs shared)**
- `04-integration-flow.md` - **Flow integrasi dan sinkronisasi dengan Odoo**
- `05-reporting-architecture.md` - **Arsitektur reporting detail**
- `06-integration.md` - Detail integrasi dengan Odoo
- `07-queue-architecture.md` - **Queue architecture dengan PostgreSQL NOTIFY**

## Quick Start

1. Baca `01-overview.md` untuk memahami konsep
2. Pelajari `02-architecture.md` untuk arsitektur
3. **Baca `03-database-architecture.md` untuk database design**
4. **Baca `04-integration-flow.md` untuk integration flow**
5. **Baca `05-reporting-architecture.md` untuk reporting**
6. **Baca `07-queue-architecture.md` untuk queue system dengan pgnotify**
7. Rujuk `06-integration.md` untuk detail implementasi

## Key Questions Answered

### Database Architecture
- ✅ **Standalone database** dengan sync ke Odoo
- ✅ Hybrid architecture untuk performance dan consistency
- ✅ Real-time dan batch sync strategies

### Integration Flow
- ✅ API Gateway pattern
- ✅ Event-driven architecture
- ✅ Error handling dan retry mechanisms

### Reporting
- ✅ ClickHouse untuk analytics
- ✅ PostgreSQL untuk operational reports
- ✅ Real-time dan batch reporting
- ✅ Export functionality (CSV, Excel)

### Queue System
- ✅ PostgreSQL NOTIFY/LISTEN untuk queue
- ✅ No external dependencies
- ✅ ACID guarantees
- ✅ Reliable delivery

Lihat detail di masing-masing file dokumentasi.
