# Coindcx Smart Orders
## Strategy Automation System — Full-Stack Technical Documentation

A production-grade, full-stack **Strategy Automation System** for US Perpetual Futures on Hyperliquid. Users define reusable trading strategies with configurable indicators, apply them to specific tokens, and have orders automatically executed when live market conditions are met.

---

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
  - [System Architecture](#system-architecture)
  - [Service Breakdown](#service-breakdown)
  - [Data Flow](#data-flow)
- [High-Level Design (HLD)](#high-level-design-hld)
  - [Core Concepts](#core-concepts)
  - [3-Step User Journey](#3-step-user-journey)
  - [System Interaction Diagram](#system-interaction-diagram)
- [Low-Level Design (LLD)](#low-level-design-lld)
  - [Database Schema](#database-schema)
  - [Strategy Executor](#strategy-executor)
  - [Indicator Evaluation](#indicator-evaluation)
  - [Order Signing Flow](#order-signing-flow)
  - [Flutter App Layer](#flutter-app-layer)
  - [BFF Proxy Layer](#bff-proxy-layer)
  - [API Contract](#api-contract)
- [Tech Stack](#tech-stack)
- [Integrations](#integrations)
- [Setup Guide](#setup-guide)
  - [Prerequisites](#prerequisites)
  - [Database Setup](#database-setup)
  - [Backend Services](#backend-services)
  - [Flutter App](#flutter-app)
  - [Environment Variables](#environment-variables)
- [Testing](#testing)
- [Performance](#performance)
- [Production Optimization](#production-optimization)
- [CI/CD Pipeline](#cicd-pipeline)
- [Security Model](#security-model)

---

## Overview

| Property | Detail |
|----------|--------|
| **Domain** | Crypto derivatives trading (US Perpetual Futures) |
| **Exchange** | Hyperliquid DEX |
| **Pattern** | Monorepo — 3 services (Flutter mobile, Go BFF, Go order/strategy service) |
| **Key Feature** | Automated order execution based on configurable indicator conditions |

### What It Does

1. **Strategy Templates** — Reusable trading blueprints (side, indicators, duration)
2. **Automation Rules** — Per-token applications of strategies with concrete indicator values
3. **Pending Orders** — Hyperliquid-format order payloads attached to rules
4. **Strategy Executor** — Background poller (5s interval) that evaluates conditions against live mark prices and auto-executes signed orders

---

## Architecture

### System Architecture

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│                              Flutter Mobile App                                  │
│                          (coindcx_app-production)                                │
│                                                                                  │
│   ┌──────────────────┐   ┌──────────────────────┐   ┌────────────────────────┐  │
│   │  Strategy List   │   │  Automation Rule      │   │  Indicator Picker      │  │
│   │  Screen          │   │  Form Controller      │   │  (Price Schedule /     │  │
│   │                  │   │                        │   │   Trailing Stop Loss)  │  │
│   └────────┬─────────┘   └──────────┬─────────────┘   └──────────┬────────────┘  │
│            └─────────────────────────┼────────────────────────────┘               │
│                                      │                                            │
│                          ┌───────────▼────────────┐                               │
│                          │  AutomationApiService   │                               │
│                          └───────────┬────────────┘                               │
│                                      │                                            │
│                          ┌───────────▼────────────┐                               │
│                          │  SharedPreferences      │  (offline-first local cache) │
│                          └────────────────────────┘                               │
└──────────────────────────────────┬───────────────────────────────────────────────┘
                                   │ HTTPS (Bearer JWT)
                                   │ /api/v1/cedefi/derivatives/HyperLiquid/*
                                   ▼
┌──────────────────────────────────────────────────────────────────────────────────┐
│                              okto-bff (:5001)                                    │
│                                                                                  │
│   LogMiddleware → AuthorizationMiddleware → DerivativePathExtractor → Handler    │
│                   (JWT or dev bypass)                                             │
│                                                                                  │
│   CedefiDerivative Service: raw JSON proxy with circuit breaker                  │
└──────────────────────────────────┬───────────────────────────────────────────────┘
                                   │ VPC HTTP (x-authorization-secret)
                                   │ /vpc/api/v1/HyperLiquid/*
                                   ▼
┌──────────────────────────────────────────────────────────────────────────────────┐
│                          okto-hl-service                                          │
│                                                                                  │
│  ┌──────────────────┐  ┌───────────────────┐  ┌──────────────────────────────┐  │
│  │ Order API (:2002)│  │ Read API (:2005)  │  │ Strategy Executor (:5003)    │  │
│  │  - Strategy CRUD │  │  - Markets        │  │  - 5s poll interval          │  │
│  │  - Rule CRUD     │  │  - User state     │  │  - Indicator evaluation      │  │
│  │  - Pending orders│  │  - Fees           │  │  - ECDSA order signing       │  │
│  └────────┬─────────┘  └───────────────────┘  │  - Hyperliquid execution     │  │
│           │                                    └──────────┬───────────────────┘  │
│           ▼                                               │                      │
│  ┌──────────────────┐  ┌──────────────────┐               │                      │
│  │ PostgreSQL       │  │ Redis            │◄──────────────┘                      │
│  │ (3 tables)       │  │ (locks + TSL     │  (peak tracking, dedup locks)        │
│  └──────────────────┘  │  peak tracking)  │                                      │
│                        └──────────────────┘                                      │
│                                                                                  │
│  External Calls:                                                                 │
│  ├── Custodial Service → resolve userId → traderAddress                          │
│  ├── Hyperliquid Info API → mark prices (allMids), asset metadata                │
│  └── Hyperliquid Exchange API → order submission                                 │
└──────────────────────────────────────────────────────────────────────────────────┘
```

### Service Breakdown

| Service | Language | Port(s) | Responsibility |
|---------|----------|---------|----------------|
| **coindcx_app-production** | Flutter/Dart | — | Mobile frontend: strategy creation UI, rule management, order forms, offline-first persistence |
| **okto-bff** | Go 1.25.7 | 5001 | API gateway: JWT auth, request routing, circuit-breaker proxy to HL service |
| **okto-hl-service** | Go 1.25.7 | 2002, 2005, 5003 | Core backend: strategy CRUD, order execution, background strategy executor, Hyperliquid integration |

### Data Flow

```
User creates strategy → Flutter saves locally + POST to BFF
                                                    │
BFF authenticates (JWT) → proxies to HL Service ────┘
                                                    │
HL Service validates → persists to PostgreSQL ──────┘
                                                    │
Strategy Executor (every 5s):                       │
  1. Sweep expired rules                            │
  2. Fetch active rules from DB ◄───────────────────┘
  3. Get mark price from Hyperliquid
  4. Evaluate indicators (AND logic)
  5. Acquire Redis lock (dedup)
  6. Build + ECDSA sign order payload
  7. Submit to Hyperliquid Exchange API
  8. Update rule/order status → "executed"
```

---

## High-Level Design (HLD)

### Core Concepts

| Concept | Description |
|---------|-------------|
| **Strategy Template** | Reusable blueprint: name, side (BUY/SELL), indicator types (priceSchedule, trailingStopLoss), duration. Not bound to any specific token. |
| **Automation Rule** | Per-token application of a template. Contains actual indicator values (target price, trail %), coin, expiration, and execution status. |
| **Pending Order** | Hyperliquid wire-format order payload attached to a rule. Executed automatically when conditions are met. |
| **Strategy Executor** | Background goroutine polling active rules every 5 seconds, evaluating conditions against live mark prices, and submitting signed orders. |

### 3-Step User Journey

```
Step 1: Create Strategy Template
  ┌─────────────────────────────────────────────────────┐
  │ User selects: side=BUY, indicators=[priceSchedule], │
  │ duration=7 days                                      │
  │ → POST /strategy-templates                           │
  │ → Reusable template stored in DB                     │
  └─────────────────────────────────────────────────────┘
                          │
                          ▼
Step 2: Apply Strategy to Token
  ┌─────────────────────────────────────────────────────┐
  │ User enters: coin=BTC, targetPrice=85000,            │
  │ condition=lte, expires=2026-12-31                     │
  │ → POST /automation-rules                             │
  │ → Active rule stored with trader address              │
  └─────────────────────────────────────────────────────┘
                          │
                          ▼
Step 3: Attach Pending Order
  ┌─────────────────────────────────────────────────────┐
  │ User configures: qty=0.001 BTC, limitPx=85000,      │
  │ orderType=GTC                                        │
  │ → POST /automation-rules/:id/orders                  │
  │ → Order payload stored; executor begins monitoring   │
  └─────────────────────────────────────────────────────┘
```

### System Interaction Diagram

```
┌───────────┐         ┌───────────┐         ┌───────────────┐         ┌───────────────┐
│  Flutter   │  HTTPS  │  okto-bff │  VPC    │ okto-hl-      │  REST   │  Hyperliquid  │
│  App       │────────►│  (:5001)  │────────►│ service       │────────►│  Exchange API │
│            │◄────────│           │◄────────│ (:2002/5003)  │◄────────│               │
└───────────┘         └───────────┘         └───────┬───────┘         └───────────────┘
                                                     │
                                              ┌──────┴──────┐
                                              │             │
                                         ┌────▼────┐  ┌────▼────┐
                                         │PostgreSQL│  │  Redis  │
                                         │(strategy │  │(locks,  │
                                         │ tables)  │  │ peaks)  │
                                         └─────────┘  └─────────┘
```

---

## Low-Level Design (LLD)

### Database Schema

#### `strategy_templates`

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | `uuid` | PK | Auto-generated UUID |
| `userId` | `uuid` | NOT NULL, indexed | Owner of the template |
| `name` | `text` | NOT NULL | Display name |
| `side` | `text` | NOT NULL | `BUY` or `SELL` |
| `indicatorTypes` | `jsonb` | NOT NULL | `["priceSchedule", "trailingStopLoss"]` |
| `duration` | `text` | NOT NULL | `sevenDays` / `twentyDays` / `oneMonth` |
| `isActive` | `boolean` | DEFAULT true, indexed | Soft-delete flag |
| `createdAt` | `timestamptz` | NOT NULL | |
| `updatedAt` | `timestamptz` | NOT NULL | |

#### `automation_rules`

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | `uuid` | PK | Auto-generated UUID |
| `userId` | `uuid` | NOT NULL, indexed | Owner |
| `traderAddress` | `text` | NOT NULL | Resolved via Custodial Service |
| `strategyTemplateId` | `uuid` | FK → `strategy_templates`, nullable | Linked template |
| `coinId` | `text` | NOT NULL | e.g. `HyperLiquid:BTC` |
| `dqlAssetId` | `uuid` | | Resolved from `coinId` |
| `coinName` | `text` | | e.g. `BTC` |
| `dexName` | `text` | | e.g. `HyperLiquid` |
| `side` | `text` | NOT NULL | `BUY` or `SELL` |
| `indicators` | `jsonb` | NOT NULL | `[{type, params}]` |
| `duration` | `text` | | |
| `expiresAt` | `timestamptz` | NOT NULL, indexed | Rule expiration |
| `isActive` | `boolean` | DEFAULT true | |
| `status` | `enum` | indexed | `pending` / `executed` / `expired` / `failed` |
| `executedAt` | `timestamptz` | nullable | Set on execution |
| `retryCount` | `int` | DEFAULT 0 | Max 3 before marking failed |
| `lastError` | `text` | nullable | Last execution error |
| `createdAt` | `timestamptz` | | |
| `updatedAt` | `timestamptz` | | |

#### `strategy_pending_orders`

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | `uuid` | PK | |
| `automationRuleId` | `uuid` | FK → `automation_rules`, CASCADE | Parent rule |
| `userId` | `uuid` | | |
| `traderAddress` | `text` | | |
| `grouping` | `text` | NOT NULL | `na` / `normalTpsl` / `positionTpsl` |
| `orderPayload` | `jsonb` | NOT NULL | Hyperliquid order wire format |
| `vaultAddress` | `text` | | |
| `status` | `enum` | | `pending` / `executed` / `cancelled` / `failed` |
| `triggerPrice` | `text` | | Filled on execution |
| `executedOrderId` | `text` | nullable | HL order ID after execution |
| `lastError` | `text` | nullable | |
| `createdAt` | `timestamptz` | | |
| `updatedAt` | `timestamptz` | | |

#### Entity Relationship

```
strategy_templates (1) ──────◄ (0..N) automation_rules (1) ──────◄ (0..N) strategy_pending_orders
       │                                    │                                      │
       │ userId (index)                     │ status (index)                       │ status
       │ isActive (index)                   │ expiresAt (index)                    │ automationRuleId (FK CASCADE)
       │                                    │ userId (index)                       │
```

### Strategy Executor

The executor is the core automation engine. It runs as a standalone binary (`hlqdstrategy`) with a background goroutine.

```
┌─────────────────────────────────────────────────┐
│             StrategyExecutor.Run()                │
│             ticker: 5s (configurable)            │
└────────────────────┬────────────────────────────┘
                     │
         ┌───────────▼────────────────┐
         │   isUSMarketHours()?        │  9:30–16:00 ET, weekdays
         │   (bypass in DEVELOPMENT    │  STRATEGY_BYPASS_MARKET_HOURS=true
         │    or via env flag)         │
         └───────────┬────────────────┘
                     │ Yes
         ┌───────────▼────────────────┐
         │   sweepExpired()            │  Bulk-mark past-due rules → "expired"
         └───────────┬────────────────┘
                     │
         ┌───────────▼────────────────┐
         │   GetActiveForExecution()   │  WHERE status=pending
         │                             │  AND isActive=true AND NOT expired
         └───────────┬────────────────┘
                     │
             ┌───────▼───────┐
             │  For each rule │  (concurrent, max 20 goroutines)
             └───────┬───────┘
                     │
         ┌───────────▼────────────────┐
         │  GetMarkPriceByCoinName()   │  Hyperliquid allMids endpoint
         └───────────┬────────────────┘
                     │
         ┌───────────▼────────────────┐
         │  evaluate(rule, markPrice)  │  ALL indicators must be satisfied
         │  ├── priceSchedule          │
         │  └── trailingStopLoss       │
         └───────────┬────────────────┘
                     │ All conditions met
         ┌───────────▼────────────────┐
         │  acquireLock(Redis SET NX)  │  60s TTL, prevents duplicate execution
         └───────────┬────────────────┘
                     │
         ┌───────────▼────────────────┐
         │  GetPendingByRuleId()       │  Fetch attached pending order
         └───────────┬────────────────┘
                     │
         ┌───────────▼────────────────┐
         │  HasOpenPosition()          │  Check user's HL clearinghouse state
         │  (skipped in DEVELOPMENT)   │
         └───────────┬────────────────┘
                     │
         ┌───────────▼────────────────┐
         │  buildAndSign()             │
         │  ├── BuildOrderWirePayload  │  Resolve coin → HL asset index
         │  ├── MsgPackHashing         │  Deterministic payload hash
         │  └── ECDSA Sign (P-256)     │  Agent private key
         └───────────┬────────────────┘
                     │
         ┌───────────▼────────────────┐
         │  ExecuteStrategyOrder()     │  POST to Hyperliquid Exchange API
         └───────────┬────────────────┘
                     │
         ┌───────────▼────────────────┐
         │  Update status:             │
         │  rule → "executed"          │
         │  pending_order → "executed" │
         │  set executedAt, trigger    │
         └─────────────────────────────┘
```

### Indicator Evaluation

#### Price Schedule

```
Params: { targetPrice: float64, condition: "lte" | "gte" }

condition == "lte" → triggered when markPrice <= targetPrice
condition == "gte" → triggered when markPrice >= targetPrice
```

#### Trailing Stop Loss (TSL)

```
Params: { trailPercent: float64, activationPrice?: float64 }

1. If activationPrice is set and not yet crossed → skip (not active)
2. Read peak/trough from Redis  (key: strategy:rule:{ruleId}:peak)
3. For BUY side (long position protection):
   - If markPx > peak → update peak in Redis (new high), not triggered
   - stopPrice = peak × (1 - trailPercent/100)
   - Triggered when markPx <= stopPrice
4. For SELL side (short position protection):
   - If markPx < trough → update trough in Redis (new low), not triggered
   - stopPrice = trough × (1 + trailPercent/100)
   - Triggered when markPx >= stopPrice

Redis keys have TTL matching rule expiration for automatic cleanup.
```

### Order Signing Flow

```
OrderPayloadItem (from DB, jsonb)
  │
  ▼ convertPayloadToOrderReq()
OrderReq[]
  │
  ▼ BuildOrderWirePayloadByCoinName()
  │  ├── resolveHLAssetIdByCoinName()  → numeric asset index from HL meta API
  │  └── orderRequestToOrderWire()     → OrderWire[]
  │
  ▼ OrderWirePayload { Type: "order", Orders, Grouping }
  │
  ▼ MsgPackHashing(payload, nonce, vaultAddress)
  │
  ▼ hashBytes (deterministic)
  │
  ▼ gethcrypto.Sign(hashBytes, agentPrivateKey)  → ECDSA P-256 signature
  │
  ▼ ExecuteStrategyOrder(payload, nonce, signatureHex, vaultAddress)
    → POST to Hyperliquid Exchange API
```

### Flutter App Layer

#### Directory Structure

```
lib/components/dcx_us_perps/
├── automation/
│   ├── models/
│   │   └── automation_rule.dart          # AutomationDuration, IndicatorType,
│   │                                     # IndicatorCondition, AutomationStrategy,
│   │                                     # AutomationRule
│   ├── controllers/
│   │   ├── automation_rule_form_controller.dart   # Form state, save strategy/rule
│   │   ├── strategies_controller.dart             # Strategy list, server sync
│   │   └── token_strategies_controller.dart       # Per-token rule list
│   ├── repositories/
│   │   └── automation_rule_repo.dart     # AutomationStrategyRepo, AutomationRuleRepo
│   │                                     # (SharedPreferences + API sync)
│   └── network/
│       ├── automation_api_service.dart    # HTTP client for BFF
│       └── automation_request_models.dart # DTOs
├── presentation/
│   ├── bottomsheets/
│   │   ├── us_perps_automation_bs.dart           # Entry point bottom sheet
│   │   ├── us_perps_create_strategy_bs.dart      # Create strategy form
│   │   ├── us_perps_indicator_picker_bs.dart     # Indicator value entry
│   │   └── us_perps_strategy_detail_bs.dart      # Strategy detail view
│   └── screens/strategies/
│       └── us_perps_strategies_screen.dart        # Main strategies list
└── bindings/
    └── us_perps_base_bindings.dart                # DI registration
```

#### Offline-First Architecture

```
                      ┌──────────────┐
  User Action ───────►│  Repository  │
                      │  .save()     │
                      └──────┬───────┘
                             │
                   ┌─────────┼──────────┐
                   ▼                    ▼
           ┌──────────────┐    ┌──────────────┐
           │ API Call      │    │ SharedPrefs  │
           │ (best-effort) │    │ (always)     │
           └──────┬───────┘    └──────────────┘
                  │
           ┌──────┴───────┐
           │ Success?     │
           │  Y: synced   │
           │  N: unsynced │
           └──────────────┘
```

- Write operations persist locally first (SharedPreferences)
- API calls are best-effort; failures set `isSynced = false`
- Background sync retries unsynced items on next load
- `isPublic = true` on the Dio client prevents 401 errors from triggering PIN lock

### BFF Proxy Layer

The BFF is a thin pass-through proxy with no business logic for strategy endpoints. It adds authentication, circuit-breaking, and request routing.

#### Middleware Chain

```
Request → LogMiddleware → AuthorizationMiddleware → AddIdentifier → DerivativePathExtractor → Handler
                               │
                               ├── Production: JWT validation via CEFI Auth Service
                               └── Development: bypass with ?user_id=<id> query param
```

#### Proxy Pattern

Each strategy endpoint follows the same pattern:

```go
func (svc *CedefiDerivative) CreateStrategyTemplate(ctx, userId, payload) {
    queryParams := map[string][]string{
        "user_id":          {userId},
        "account_owner_id": {accountOwnerId},
    }
    return apiCallWithCb(ctx, svc, "CREATE_STRATEGY_TEMPLATE", payload, nil, queryParams, nil, "POST")
}
```

`apiCallWithCb` wraps the HTTP call with a circuit breaker and adds the `x-authorization-secret` VPC header.

### API Contract

#### Strategy Endpoints (12 total)

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/strategy-templates` | Create strategy template |
| `GET` | `/strategy-templates` | List user's templates |
| `GET` | `/strategy-templates/:tmpl_id` | Get template by ID |
| `PATCH` | `/strategy-templates/:tmpl_id` | Partial update template |
| `DELETE` | `/strategy-templates/:tmpl_id` | Delete template |
| `POST` | `/automation-rules` | Create automation rule |
| `GET` | `/automation-rules` | List rules (filterable by status) |
| `GET` | `/automation-rules/:rule_id` | Get rule by ID |
| `DELETE` | `/automation-rules/:rule_id` | Delete/deactivate rule |
| `POST` | `/automation-rules/:rule_id/orders` | Attach pending order |
| `GET` | `/automation-rules/:rule_id/orders` | List pending orders |
| `DELETE` | `/automation-rules/:rule_id/orders/:order_id` | Cancel pending order |

**BFF base path:** `/api/v1/cedefi/derivatives/:market-name/`
**HL Service base path:** `/vpc/api/v1/HyperLiquid/`

Full API documentation with request/response schemas: [`okto-hl-service/docs/strategy-api.md`](okto-hl-service/docs/strategy-api.md)

---

## Tech Stack

| Layer | Technology | Version | Purpose |
|-------|-----------|---------|---------|
| **Mobile** | Flutter | 3.32.7+ | Cross-platform mobile app |
| **Mobile** | Dart | >=3.5.0 | Language |
| **Mobile** | GetX | — | State management, dependency injection |
| **Mobile** | Riverpod | — | Reactive state (other modules) |
| **Mobile** | SharedPreferences | — | Offline-first local persistence |
| **BFF** | Go | 1.25.7 | BFF service language |
| **BFF** | Gin | — | HTTP framework |
| **BFF** | Redis | 7+ | Caching |
| **BFF** | PostgreSQL | 14+ | Primary database (okto-swap) |
| **BFF** | MongoDB | — | Other modules |
| **HL Service** | Go | 1.25.7 | Core backend language |
| **HL Service** | Gin | — | HTTP framework |
| **HL Service** | PostgreSQL | 14+ | Primary database (okto-hl-service) |
| **HL Service** | Redis | 7+ | Distributed locks, TSL peak tracking |
| **HL Service** | Confluent Kafka | — | Order notifications, analytics |
| **HL Service** | AWS SQS | — | Order reconciliation queue |
| **Observability** | OpenTelemetry | — | Distributed tracing |
| **Observability** | Datadog | — | APM, metrics, logs |
| **Infrastructure** | Docker | — | Container builds |
| **Infrastructure** | GitHub Actions | — | CI/CD pipeline |
| **Crypto** | go-ethereum | — | ECDSA P-256 order signing |
| **Crypto** | MsgPack | — | Deterministic payload hashing |

---

## Integrations

| Service | Protocol | Purpose |
|---------|----------|---------|
| **Hyperliquid Info API** | REST | Mark prices (`allMids`), asset metadata, user state |
| **Hyperliquid Exchange API** | REST | Order submission, cancellation, leverage updates |
| **Custodial Service** | REST (VPC) | Resolve `userId` → Hyperliquid `traderAddress` |
| **CEFI Auth Service** | REST | JWT token validation (production) |
| **Token Pricing Service** | REST | Token prices, candles |
| **Portfolio Service** | REST | Positions, balances |
| **DQL (Data Query Layer)** | REST | Entity resolution, token metadata |
| **Experience Config** | REST | Fee configuration |
| **OMS** | REST | Order management |
| **Firebase** | SDK | Push notifications, remote config |
| **MoEngage** | SDK | Analytics and engagement tracking |
| **QuickNode** | RPC | Hyperliquid blockchain RPC |

---

## Setup Guide

### Prerequisites

| Tool | Version | Required For |
|------|---------|-------------|
| Flutter | 3.x+ | Mobile app |
| Go | 1.21+ | Backend services |
| PostgreSQL | 14+ | Database |
| Redis | 7+ | Caching, locks |
| Android Studio / Xcode | Latest | Mobile emulator/simulator |

### Database Setup

```bash
# Create databases
psql -U postgres -c "CREATE DATABASE \"okto-swap\";"
psql -U postgres -c "CREATE DATABASE \"okto-hl-service\";"

# Run HL Service migrations
cd okto-hl-service
go run scripts/database/main.go -direction up -database "postgres://postgres:postgres@localhost:5432/okto-hl-service?sslmode=disable"
```

### Backend Services

```bash
# Terminal 1: HL Service — Order API
cd okto-hl-service
ENV=DEVELOPMENT \
ORDER_PORT=2002 \
DATABASE_URL="postgres://postgres:postgres@localhost:5432/okto-hl-service?sslmode=disable" \
REDIS_URL="localhost:7001" \
HYPERLIQUID_API_SERVICE_BASE_URL="https://api.hyperliquid.xyz" \
go run cmd/hlqdorder/main.go

# Terminal 2: HL Service — Strategy Executor
cd okto-hl-service
ENV=DEVELOPMENT \
STRATEGY_PORT=5003 \
DATABASE_URL="postgres://postgres:postgres@localhost:5432/okto-hl-service?sslmode=disable" \
REDIS_URL="localhost:7001" \
HYPERLIQUID_API_SERVICE_BASE_URL="https://api.hyperliquid.xyz" \
HL_STRATEGY_AGENT_PRIVATE_KEY="your-ecdsa-private-key" \
STRATEGY_BYPASS_MARKET_HOURS=true \
go run cmd/hlqdstrategy/main.go

# Terminal 3: BFF
cd okto-bff
ENV=DEVELOPMENT \
PORT=5001 \
DATABASE_URL="postgres://postgres:postgres@localhost:5432/okto-swap?sslmode=disable" \
REDIS_URL="localhost:6379" \
HL_ORDER_SERVICE_BASE_URL="http://localhost:2002" \
HL_READ_SERVICE_BASE_URL="http://localhost:2005" \
go run cmd/main.go
```

### Flutter App

```bash
cd coindcx_app-production
flutter pub get
flutter run \
  --dart-define=LOCAL_BFF_URL=http://10.0.2.2:5001 \
  --flavor=gopro
```

> **Note:** Use `10.0.2.2` for Android emulator (maps to host `localhost`). For iOS simulator, use `localhost` directly.

### Environment Variables

#### okto-hl-service

| Variable | Required | Description |
|----------|----------|-------------|
| `ENV` | Yes | `DEVELOPMENT` / `STAGING` / `PRODUCTION` |
| `ORDER_PORT` | Yes | Order API port (default: 2002) |
| `READ_PORT` | Yes | Read API port (default: 2005) |
| `STRATEGY_PORT` | Yes | Strategy executor port (default: 5003) |
| `DATABASE_URL` | Yes | PostgreSQL connection string |
| `REDIS_URL` | Yes | Redis host:port |
| `HYPERLIQUID_API_SERVICE_BASE_URL` | Yes | `https://api.hyperliquid.xyz` |
| `CUSTODIAL_SERVICE_BASE_URL` | Yes | Custodial service URL |
| `HL_STRATEGY_AGENT_PRIVATE_KEY` | Yes | ECDSA P-256 private key for order signing |
| `STRATEGY_BYPASS_MARKET_HOURS` | No | `true` to skip US market hours check |
| `HL_STRATEGY_POLL_INTERVAL_SECS` | No | Executor poll interval (default: 5) |
| `OKTO_VPC_SECRET` | Yes | VPC authentication secret |

#### okto-bff

| Variable | Required | Description |
|----------|----------|-------------|
| `ENV` | Yes | `DEVELOPMENT` / `STAGING` / `PRODUCTION` |
| `PORT` | Yes | BFF port (default: 5001) |
| `DATABASE_URL` | Yes | PostgreSQL connection string (okto-swap) |
| `REDIS_URL` | Yes | Redis host:port |
| `HL_ORDER_SERVICE_BASE_URL` | Yes | HL order API URL |
| `HL_READ_SERVICE_BASE_URL` | Yes | HL read API URL |
| `CEFI_AUTH_BASE_URL` | Yes | Auth service URL |
| `OTEL_EXPORTER_OTLP_ENDPOINT` | No | OpenTelemetry collector |

### Port Reference

| Service | Port |
|---------|------|
| BFF | 5001 |
| HL Order API | 2002 |
| HL Read API | 2005 |
| HL Strategy Executor | 5003 |
| PostgreSQL | 5432 |
| Redis (BFF) | 6379 |
| Redis (HL Service) | 7001 |

---

## Testing

Comprehensive test suites across all three services. Full test reports are in the [`automation-testing/`](automation-testing/) folder.

### Summary

| Service | Tests | Duration | Status | Framework |
|---------|-------|----------|--------|-----------|
| **okto-hl-service** | 132 | ~5.4s | ALL PASS | testify, go-sqlmock, miniredis |
| **coindcx_app-production** | 69 | ~2s | ALL PASS | flutter_test, mockito |
| **okto-bff** | N/A (thin proxy) | — | — | — |

### Test Categories

| Category | Count | Description |
|----------|-------|-------------|
| API Handler Tests | 29 | HTTP status codes, response structure, validation errors |
| Validation Layer Tests | 26 | All `Validate()` methods, boundary conditions |
| Repository Integration Tests | 32 | SQL generation, JSON serialization (sqlmock) |
| TSL Redis Integration Tests | 7 | Peak tracking, TTL, lock contention (miniredis) |
| Order Signing E2E Tests | 3 | Real ECDSA key, full buildAndSign flow |
| Executor Unit Tests | 35 | Price evaluation, sweep, concurrency, failure handling |
| Flutter Model Tests | 26 | JSON round-trip, enums, copyWith |
| Flutter Repo Tests | 17 | SharedPreferences CRUD, corrupted data handling |
| Flutter Controller Tests | 26 | State management, validation, form logic |

### How to Run

```bash
# HL Service — All 132 strategy tests
cd okto-hl-service
go test -v -count=1 \
  ./internal/service/strategy/ \
  ./internal/repository/automationrules/ \
  ./internal/repository/strategypendingorders/ \
  ./apicommon/types/ \
  ./api/apiorder/handler/vpc/v1/strategy/

# HL Service — Benchmarks
go test -bench=. -benchmem -benchtime=3s -run='^$' ./internal/service/strategy/

# Flutter — Automation tests
cd coindcx_app-production
flutter test test/components/dcx_us_perps/automation/ --no-pub

# BFF — All tests
cd okto-bff
go test -v ./...
```

See [`automation-testing/README.md`](automation-testing/README.md) for the full test matrix and results.

---

## Performance

Performance benchmarks run on **darwin/arm64 (Apple Silicon)** using Go's built-in benchmarking framework with `miniredis` for Redis operations.

### Key Benchmarks

| Benchmark | ns/op | B/op | allocs/op | Verdict |
|-----------|-------|------|-----------|---------|
| `EvaluatePriceSchedule` (fast path) | **13.96** | 0 | 0 | Zero-alloc hot path |
| `EvaluateTrailingStopLoss` (not triggered) | **10,897** | 8,282 | 84 | Redis I/O dominates |
| `EvaluateTrailingStopLoss` (triggered) | **4,762** | 4,083 | 41 | Faster due to early return |
| `ProcessTick` (1 rule) | ~100K | — | — | Single rule baseline |
| `ProcessTick` (100 rules) | ~3M | — | — | Linear scaling confirmed |
| `SweepExpired` (0 rules) | **5,399** | 4,327 | 40 | Baseline overhead |
| `SweepExpired` (500 rules) | ~62K | — | — | ~124ns per rule |
| `AcquireLock` (Redis) | **14,683** | 11,908 | 119 | Redis SET NX overhead |

### Scaling Analysis

| Rules | Tick Latency | Per-Rule Cost | Status |
|-------|-------------|---------------|--------|
| 1 | ~100μs | 100μs | Well within 5s budget |
| 10 | ~300μs | ~30μs | Concurrent goroutines amortize |
| 50 | ~1.5ms | ~30μs | |
| 100 | ~3ms | ~30μs | 0.06% of 5s budget |
| 1,000 | ~30ms (est.) | ~30μs | Easily handles production scale |
| 10,000 | ~300ms (est.) | ~30μs | Still within budget |

See [`performance-testing/README.md`](performance-testing/README.md) for detailed benchmark results, profiling methodology, and optimization recommendations.

---

## Production Optimization

### Current Optimizations

| Area | Implementation | Impact |
|------|---------------|--------|
| **Zero-alloc price evaluation** | `evaluatePriceSchedule` performs pure float comparison | 13.96 ns/op, 0 allocs |
| **Concurrent rule processing** | Goroutine pool with `sync.WaitGroup` (max 20 concurrent) | Linear scaling to 1000+ rules |
| **Redis distributed locks** | `SET NX` with 60s TTL prevents duplicate execution | Exactly-once order submission |
| **Circuit breaker** | Configurable per-service circuit breakers (BFF → HL, HL → Hyperliquid) | Cascading failure prevention |
| **Interface segregation** | Narrow 4-method `exchangeService` interface for executor | Clean testability, minimal coupling |
| **Offline-first mobile** | SharedPreferences local persistence + best-effort API sync | Zero-latency UI, resilient to network issues |
| **Bulk operations** | `sweepExpired` uses batch UPDATE for multiple rule IDs | Single DB round-trip |
| **Dev auth bypass** | `ENV=DEVELOPMENT` + `user_id` query param skips JWT | Rapid local development |

### Recommended Production Enhancements

#### Infrastructure

| Enhancement | Description | Expected Impact |
|-------------|-------------|-----------------|
| **Connection pooling** | Configure `sql.DB` max open/idle connections with health checks | Reduce DB connection overhead by ~30% |
| **Redis Cluster** | Migrate from standalone Redis to cluster mode with read replicas | Horizontal scalability for TSL peak reads |
| **Rate limiting** | Add per-user rate limits on strategy creation and rule attachment | Prevent abuse, protect HL API quota |
| **Read replicas** | PostgreSQL read replicas for `GetActiveForExecution` queries | Reduce primary DB load |

#### Application

| Enhancement | Description | Expected Impact |
|-------------|-------------|-----------------|
| **Mark price caching** | Cache `allMids` response with 1s TTL (shared across rules for same tick) | Reduce Hyperliquid API calls by ~99% per tick |
| **Batch price fetching** | Single `allMids` call per tick instead of per-rule | O(1) API calls instead of O(N) |
| **Executor sharding** | Partition rules across executor instances by `userId` hash | Horizontal scaling |
| **Dead letter queue** | Route failed executions to SQS DLQ for manual review | Prevent silent order failures |
| **Metrics & alerting** | Prometheus metrics: tick duration, rules processed, execution success rate | Real-time operational visibility |
| **Graceful shutdown** | Drain in-flight executions on SIGTERM with configurable timeout | Zero-downtime deployments |

#### Database

| Enhancement | Description | Expected Impact |
|-------------|-------------|-----------------|
| **Partial indexes** | `CREATE INDEX ... WHERE status = 'pending' AND isActive = true` | Faster executor queries, smaller index |
| **Table partitioning** | Partition `automation_rules` by `status` (hot/cold data separation) | Query performance at scale |
| **JSONB GIN indexes** | Index `indicators` JSONB for query-time indicator filtering | Sub-ms indicator lookups |
| **Vacuum tuning** | Aggressive autovacuum for high-churn tables (`automation_rules`) | Prevent bloat, maintain query performance |

#### Security

| Enhancement | Description | Expected Impact |
|-------------|-------------|-----------------|
| **Key rotation** | HSM-backed agent private key with rotation schedule | Compliance, reduced blast radius |
| **Audit logging** | Log all strategy executions to immutable audit trail | Regulatory compliance |
| **Input sanitization** | Strict validation on indicator params (price bounds, trail % limits) | Prevent invalid orders |
| **VPC secret rotation** | Automated rotation via AWS Secrets Manager | Reduced credential exposure |

---

## CI/CD Pipeline

### okto-hl-service

| Trigger | Pipeline | Steps |
|---------|----------|-------|
| PR opened | `ci_pr.yml` | Snyk vulnerability scan, Sysdig image scan |
| Push to branch | `ci_push.yml` | Go test suite (132 strategy tests + existing) |
| Push to branch | `ci_quality.yml` | Full test suite with PostgreSQL + Redis services |
| Push to `release/*` | `cd.yml` | Build Docker image → Push to ECR → Deploy via golden-images |

### okto-bff

| Trigger | Pipeline | Steps |
|---------|----------|-------|
| PR opened | `ci_pr.yml` | Snyk + Sysdig scan |
| Push to branch | `ci_push.yml`, `ci_quality.yml` | Go tests with PostgreSQL, Redis, DynamoDB |
| Push to `release/*` | `cd.yml` | Build app + migration images → ECR → Deploy |

### coindcx_app-production

| Trigger | Pipeline | Steps |
|---------|----------|-------|
| PR opened | `pr-validation.yml` | PR title and commit message validation |
| PR to `develop` | `require-pr-milestone.yml` | Milestone enforcement |
| PR (options) | `options_test_automation.yml` | Coverage guardrail (95%/90%/80% tiers) |
| Push | `ci_quality.yml` | Flutter tests + Sonar analysis |

---

## Security Model

| Layer | Mechanism | Details |
|-------|-----------|---------|
| **Flutter → BFF** | JWT Bearer token | Validated by CEFI Auth Service |
| **BFF → HL Service** | VPC shared secret | `x-authorization-secret` header |
| **HL Service → Custodial** | VPC shared secret | Same mechanism |
| **Order Signing** | ECDSA P-256 | `HL_STRATEGY_AGENT_PRIVATE_KEY` signs all automated orders |
| **Deduplication** | Redis `SET NX` lock | 60s TTL per rule prevents duplicate execution |
| **Dev Mode** | ENV check | Auth bypass only when `ENV=DEVELOPMENT` AND `user_id` query param present |
| **Market Hours** | Time gate | Executor only runs during US market hours (9:30–16:00 ET, weekdays) unless bypassed |

### Error Handling & Resilience

| Scenario | Behavior |
|----------|----------|
| API call fails (Flutter) | Save locally with `isSynced = false`; retry on next sync |
| BFF unreachable | Flutter falls back to local SharedPreferences |
| HL Service Custodial lookup fails | Returns "user address not found" error |
| Executor order fails | `retryCount` incremented; marked `failed` after 3 retries |
| Rule expires | `sweepExpired()` bulk-marks as `expired` each tick |
| Duplicate execution | Redis `SET NX` lock prevents concurrent execution |
| Redis restart (TSL) | Peak/trough re-initializes from current mark price |
| Circuit breaker open | BFF returns fast-fail response, no downstream call |

---

## Project Structure

```
hackathon/
├── README.md                          ← You are here
├── automation-testing/                ← Test results and coverage
│   └── README.md
├── performance-testing/               ← Benchmark results and optimization
│   └── README.md
│
├── okto-bff/                          # Go BFF (Backend For Frontend)
│   ├── cmd/main.go                    # Entry point
│   ├── api/
│   │   ├── handler/                   # HTTP handlers
│   │   ├── middleware/                # Auth, logging, singleflight
│   │   └── route/                     # Route registration
│   ├── config/                        # Circuit breaker, cache configs
│   ├── internal/
│   │   ├── service/cedefiderivative/  # Strategy proxy, HL API calls
│   │   └── repository/               # Database access
│   ├── scripts/database/             # Migrations, Dockerfile
│   └── .github/workflows/            # CI/CD
│
├── okto-hl-service/                   # Go Hyperliquid Service
│   ├── cmd/
│   │   ├── hlqdorder/main.go         # Order API entry point
│   │   ├── hlqdread/main.go          # Read API entry point
│   │   └── hlqdstrategy/main.go      # Strategy executor entry point
│   ├── api/apiorder/
│   │   ├── handler/vpc/v1/strategy/  # Strategy API handlers
│   │   └── route/                    # Route registration
│   ├── apicommon/types/              # Request/response types, validation
│   ├── internal/
│   │   ├── service/strategy/         # Executor, evaluation logic
│   │   ├── service/hyperliquid/      # HL API integration
│   │   └── repository/              # DB access (automationrules, pendingorders)
│   ├── docs/strategy-api.md          # API documentation
│   ├── scripts/database/migration/   # 21 SQL migration files
│   └── .github/workflows/           # CI/CD
│
└── coindcx_app-production/            # Flutter Mobile App
    ├── lib/components/dcx_us_perps/
    │   ├── automation/               # Models, controllers, repos, network
    │   └── presentation/             # Screens, bottomsheets, UI
    ├── test/components/dcx_us_perps/
    │   └── automation/               # 69 tests (models, repo, controller)
    ├── LOCAL_SETUP.md                # Detailed local setup guide
    ├── ARCHITECTURE.md               # Technical architecture doc
    └── .github/workflows/            # CI/CD
```

---

## License

Internal — CoinDCX proprietary software.
