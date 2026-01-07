# Integrasi dengan Odoo

## Overview

Dokumen ini menjelaskan detail integrasi Rust engine dengan Odoo, termasuk API design, authentication, dan error handling.

## Integration Architecture

```
┌─────────────────┐
│ Odoo POS         │
│ Frontend         │
└────────┬─────────┘
         │ HTTP/gRPC
┌────────▼─────────┐
│ Rust Engine API  │
│ Gateway          │
└────────┬─────────┘
         │
┌────────▼─────────┐
│ Rust Engine      │
│ Core             │
└──────────────────┘
```

## API Design

### REST API Endpoints

- `POST /api/v1/orders` - Process order
- `GET /api/v1/cards/{id}/balance` - Get balance
- `POST /api/v1/points/earn` - Earn points
- `POST /api/v1/points/redeem` - Redeem points

### Authentication

- JWT tokens
- API keys
- OAuth2

## Queue Integration

Queue system terintegrasi untuk:
- Async order processing
- Point operations
- Sync operations
- Reporting generation

Lihat detail di: `07-queue-architecture.md`

Lihat dokumentasi berikutnya:
- `07-queue-architecture.md` - Queue system dengan pgnotify
- `04-integration-flow.md` - Detail integration flow
