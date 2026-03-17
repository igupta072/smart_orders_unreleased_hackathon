# Automation Testing Report

**Date:** 2026-03-14
**Status:** ALL PASS — **201 tests** across 3 services

---

## Executive Summary

| Service | Framework | Tests | Duration | Pass Rate |
|---------|-----------|-------|----------|-----------|
| **okto-hl-service** (Go) | testify v1.11.1, go-sqlmock v1.5.2, miniredis v2.37.0 | 132 | ~5.4s | 100% |
| **coindcx_app-production** (Flutter) | flutter_test, mockito 5.4.4, mocktail ^1.0.4 | 69 | ~2s | 100% |
| **okto-bff** (Go) | N/A — thin proxy layer, no strategy-specific tests | — | — | — |
| **Total** | | **201** | **~7.4s** | **100%** |

---

## Test Pyramid

```
                    ┌───────────┐
                    │   E2E     │  3 tests (order signing full flow)
                    │  Signing  │
                  ┌─┴───────────┴─┐
                  │  Integration   │  39 tests (sqlmock + miniredis)
                  │  (DB + Redis)  │
                ┌─┴────────────────┴─┐
                │    Unit Tests       │  159 tests (handlers, validation,
                │  (Go + Flutter)     │   executor, models, controllers)
                └─────────────────────┘
```

---

## 1. okto-hl-service — Go Backend (132 Tests)

### Package Breakdown

| Package | Tests | Duration | Category |
|---------|-------|----------|----------|
| `api/apiorder/handler/vpc/v1/strategy` | 29 | ~2.1s | API Handler |
| `apicommon/types` | 26 | ~0.8s | Validation Layer |
| `internal/repository/automationrules` | 19 | ~0.6s | Repository (sqlmock) |
| `internal/repository/strategypendingorders` | 13 | ~0.9s | Repository (sqlmock) |
| `internal/service/strategy` | 45 | ~1.0s | Executor + Redis + Signing E2E |

### 1.1 Strategy API Handler Tests (29 tests)

**Files:** `strategy_templates_test.go`, `automation_rules_test.go`
**Technique:** `httptest.NewRecorder()` + `gin.CreateTestContext()` with mocked `StrategyServiceInterface`

| # | Test Name | Status | Category |
|---|-----------|--------|----------|
| 1 | `TestCreateStrategyTemplate_Success` | PASS | Template CRUD |
| 2 | `TestCreateStrategyTemplate_InvalidBody` | PASS | Error Handling |
| 3 | `TestCreateStrategyTemplate_ValidationError` | PASS | Validation |
| 4 | `TestCreateStrategyTemplate_ServiceError` | PASS | Error Handling |
| 5 | `TestCreateStrategyTemplate_MissingUserId` | PASS | Edge Case |
| 6 | `TestCreateStrategyTemplate_ResponseStructure` | PASS | Response Format |
| 7 | `TestListStrategyTemplates_Success` | PASS | Template CRUD |
| 8 | `TestListStrategyTemplates_WithActiveFilter` | PASS | Query Filters |
| 9 | `TestListStrategyTemplates_InactiveFilter` | PASS | Query Filters |
| 10 | `TestListStrategyTemplates_ServiceError` | PASS | Error Handling |
| 11 | `TestGetStrategyTemplate_Success` | PASS | Template CRUD |
| 12 | `TestGetStrategyTemplate_NotFound` | PASS | Error Handling |
| 13 | `TestUpdateStrategyTemplate_Success` | PASS | Template CRUD |
| 14 | `TestUpdateStrategyTemplate_InvalidDuration` | PASS | Validation |
| 15 | `TestDeleteStrategyTemplate_Success` | PASS | Template CRUD |
| 16 | `TestDeleteStrategyTemplate_ServiceError` | PASS | Error Handling |
| 17 | `TestCreateAutomationRule_Success` | PASS | Rule CRUD |
| 18 | `TestCreateAutomationRule_InvalidBody` | PASS | Error Handling |
| 19 | `TestCreateAutomationRule_ValidationError_MissingSide` | PASS | Validation |
| 20 | `TestCreateAutomationRule_ServiceError` | PASS | Error Handling |
| 21 | `TestListAutomationRules_Success` | PASS | Rule CRUD |
| 22 | `TestListAutomationRules_WithStatusFilter` | PASS | Query Filters |
| 23 | `TestListAutomationRules_ServiceError` | PASS | Error Handling |
| 24 | `TestGetAutomationRule_Success` | PASS | Rule CRUD |
| 25 | `TestGetAutomationRule_NotFound` | PASS | Error Handling |
| 26 | `TestDeleteAutomationRule_Success` | PASS | Rule CRUD |
| 27 | `TestDeleteAutomationRule_ServiceError` | PASS | Error Handling |
| 28 | `TestAttachOrderToRule_Success` | PASS | Pending Orders |
| 29 | `TestAttachOrderToRule_ValidationError_EmptyGrouping` | PASS | Validation |

### 1.2 Validation Layer Tests (26 tests)

**File:** `strategy_test.go`
**Technique:** Direct `Validate()` method calls on request DTOs with boundary conditions

| # | Test Name | Status | Category |
|---|-----------|--------|----------|
| 1 | `TestCreateStrategyTemplateReq_Valid` | PASS | Happy Path |
| 2 | `TestCreateStrategyTemplateReq_ValidWithTSL` | PASS | TSL Indicator |
| 3 | `TestCreateStrategyTemplateReq_ValidMultipleIndicators` | PASS | Multi-indicator |
| 4 | `TestCreateStrategyTemplateReq_InvalidSide` | PASS | Validation Error |
| 5 | `TestCreateStrategyTemplateReq_EmptyIndicatorTypes` | PASS | Validation Error |
| 6 | `TestCreateStrategyTemplateReq_TooManyIndicatorTypes` | PASS | Max Limit (3) |
| 7 | `TestCreateStrategyTemplateReq_UnsupportedIndicatorType` | PASS | Unknown Type |
| 8 | `TestCreateStrategyTemplateReq_InvalidDuration` | PASS | Validation Error |
| 9 | `TestUpdateStrategyTemplateReq_Valid_AllNil` | PASS | Partial Update |
| 10 | `TestUpdateStrategyTemplateReq_Valid_WithDuration` | PASS | Partial Update |
| 11 | `TestUpdateStrategyTemplateReq_InvalidDuration` | PASS | Validation Error |
| 12 | `TestCreateAutomationRuleReq_Valid` | PASS | Happy Path |
| 13 | `TestCreateAutomationRuleReq_ValidWithTSL` | PASS | TSL Indicator |
| 14 | `TestCreateAutomationRuleReq_MixedIndicators` | PASS | Multi-indicator |
| 15 | `TestCreateAutomationRuleReq_EmptyCoinId` | PASS | Required Field |
| 16 | `TestCreateAutomationRuleReq_InvalidSide` | PASS | Validation Error |
| 17 | `TestCreateAutomationRuleReq_EmptyIndicators` | PASS | Required Field |
| 18 | `TestCreateAutomationRuleReq_TooManyIndicators` | PASS | Max Limit |
| 19 | `TestCreateAutomationRuleReq_UnsupportedIndicator` | PASS | Unknown Type |
| 20 | `TestCreateAutomationRuleReq_InvalidDuration` | PASS | Validation Error |
| 21 | `TestCreateAutomationRuleReq_EmptyDuration_Allowed` | PASS | Optional Field |
| 22 | `TestCreateAutomationRuleReq_MissingExpiresAt` | PASS | Required Field |
| 23 | `TestCreateAutomationRuleReq_InvalidExpiresAtFormat` | PASS | RFC3339 Format |
| 24 | `TestAttachOrderReq_Valid` | PASS | Happy Path |
| 25 | `TestAttachOrderReq_EmptyGrouping` | PASS | Required Field |
| 26 | `TestAttachOrderReq_EmptyOrders` | PASS | Required Field |

### 1.3 Repository Integration Tests — Automation Rules (19 tests)

**File:** `automation_rules_test.go`
**Technique:** `go-sqlmock` for SQL verification, JSON indicator serialization round-trips

| # | Test Name | Status | Category |
|---|-----------|--------|----------|
| 1 | `TestNewAutomationRulesRepository` | PASS | Constructor |
| 2 | `TestCreate_GeneratesUUID` | PASS | Create |
| 3 | `TestCreate_WithExistingId` | PASS | Create |
| 4 | `TestCreate_WithTemplateId` | PASS | Create (FK) |
| 5 | `TestGetById_Found` | PASS | Read |
| 6 | `TestGetById_NotFound` | PASS | Read (404) |
| 7 | `TestGetAllByUserId_NoFilters` | PASS | List |
| 8 | `TestGetAllByUserId_WithStatusFilter` | PASS | List (filter) |
| 9 | `TestUpdateStatus_WithExecutedAt` | PASS | Update |
| 10 | `TestUpdateStatus_WithoutExecutedAt` | PASS | Update |
| 11 | `TestIncrementRetry` | PASS | Retry Logic |
| 12 | `TestDelete` | PASS | Delete |
| 13 | `TestBulkExpire_MultipleIds` | PASS | Bulk Update |
| 14 | `TestBulkExpire_EmptyIds_NoOp` | PASS | Edge Case |
| 15 | `TestGetActiveForExecution_ReturnsRules` | PASS | Executor Query |
| 16 | `TestGetAllExpiredPending_Empty` | PASS | Sweep Query |
| 17 | `TestDbModelToRule_IndicatorDeserialization` | PASS | JSON Roundtrip |
| 18 | `TestDbModelToRule_NullableFields` | PASS | Nullable Handling |
| 19 | `TestDbModelToRule_InvalidJSON` | PASS | Error Handling |

### 1.4 Repository Integration Tests — Pending Orders (13 tests)

**File:** `strategy_pending_orders_test.go`
**Technique:** `go-sqlmock` for SQL verification

| # | Test Name | Status | Category |
|---|-----------|--------|----------|
| 1 | `TestNewStrategyPendingOrdersRepository` | PASS | Constructor |
| 2 | `TestCreate_GeneratesUUID` | PASS | Create |
| 3 | `TestCreate_WithExistingId` | PASS | Create |
| 4 | `TestGetByRuleId_Found` | PASS | Read |
| 5 | `TestGetByRuleId_NotFound` | PASS | Read (404) |
| 6 | `TestGetPendingByRuleId_Found` | PASS | Read (filtered) |
| 7 | `TestUpdateStatus` | PASS | Update |
| 8 | `TestCancel` | PASS | Cancel |
| 9 | `TestGetAllByRuleId_Multiple` | PASS | List |
| 10 | `TestGetAllByRuleId_Empty` | PASS | List (empty) |
| 11 | `TestGetAllByUserId` | PASS | List |
| 12 | `TestScanRow_NullableFields` | PASS | Nullable Handling |
| 13 | `TestCreate_OrderPayload_RoundTrip` | PASS | JSON Roundtrip |

### 1.5 TSL Redis Integration Tests (7 tests)

**File:** `executor_redis_test.go`
**Technique:** `miniredis` — pure-Go in-process Redis server (no external Redis required)

| # | Test Name | Status | Category |
|---|-----------|--------|----------|
| 1 | `TestTSL_RedisKeyFormat` | PASS | Key Format Validation |
| 2 | `TestTSL_Redis_SetAndGetPeak` | PASS | Value Persistence |
| 3 | `TestTSL_Redis_LockTTL` | PASS | Lock TTL Expiry |
| 4 | `TestTSL_Redis_MultiTickLifecycle_Buy` | PASS | BUY Lifecycle Simulation |
| 5 | `TestTSL_Redis_MultiTickLifecycle_Sell` | PASS | SELL Lifecycle Simulation |
| 6 | `TestTSL_Redis_KeyIsolation` | PASS | Multi-rule Key Isolation |
| 7 | `TestTSL_Redis_LockContention` | PASS | Concurrent Lock Race |

### 1.6 Order Signing E2E Tests (3 tests)

**File:** `executor_test.go`
**Technique:** Real `ecdsa.PrivateKey` (P-256), full `buildAndSign` → `MsgPackHashing` → `gethcrypto.Sign` path

| # | Test Name | Status | Category |
|---|-----------|--------|----------|
| 1 | `TestBuildAndSign_FullE2E` | PASS | Signing + Determinism |
| 2 | `TestBuildAndSign_BuildPayloadError` | PASS | Error Path |
| 3 | `TestBuildAndSign_ThenExecuteOrder_FullFlow` | PASS | Full evaluate → sign → execute |

### 1.7 Executor Unit Tests (35 tests)

**File:** `executor_test.go`
**Technique:** Mocked `exchangeService` (4-method interface), mocked repositories

| Component | Tests | Description |
|-----------|-------|-------------|
| `evaluate()` — Price Schedule | 5 | LTE/GTE met/not-met, empty/unknown indicators |
| `evaluateTrailingStopLoss()` — BUY | 5 | Peak tracking, trigger/no-trigger, boundary, peak update |
| `evaluateTrailingStopLoss()` — SELL | 4 | Trough tracking, trigger/no-trigger, trough update |
| `evaluateTrailingStopLoss()` — Activation | 3 | Activation price gate for both sides |
| `evaluateTrailingStopLoss()` — Validation | 3 | Zero %, >100%, missing trailPercent |
| `sweepExpired()` | 3 | No expired, has expired, DB error |
| `processTick()` | 3 | No rules, price error, 5 concurrent rules |
| `handleFailure()` | 2 | Under limit, exceeds limit → mark failed |
| `convertPayloadToOrderReq()` | 2 | Round-trip, empty payload |
| `acquireLock()` | 2 | Available, already held |
| `isUSMarketHours()` | 1 | Bypass env flag |
| `buildAndSign()` (E2E) | 3 | Full signing, error path, complete flow |

---

## 2. coindcx_app-production — Flutter (69 Tests)

### Package Breakdown

| Test File | Tests | Duration | Status |
|-----------|-------|----------|--------|
| `automation_models_test.dart` | 26 | <1s | ALL PASS |
| `automation_repo_test.dart` | 17 | <1s | ALL PASS |
| `automation_controller_test.dart` | 26 | <1s | ALL PASS |

### 2.1 Model Tests (26 tests)

**File:** `automation_models_test.dart`
**Scope:** Pure data layer — no I/O, no mocks

#### AutomationDuration (5 tests)

| # | Test | Status |
|---|------|--------|
| 1 | `defaultDays returns correct values` (7, 20, 30) | PASS |
| 2 | `apiValue returns wire names` (sevenDays, twentyDays, oneMonth) | PASS |
| 3 | `fromString handles legacy "custom" value` → maps to oneMonth | PASS |
| 4 | `fromString falls back to sevenDays for unknown values` | PASS |
| 5 | `label returns human-readable strings` | PASS |

#### IndicatorType (3 tests)

| # | Test | Status |
|---|------|--------|
| 6 | `displayName returns human-readable names` (all 4 types) | PASS |
| 7 | `fromString parses known values` | PASS |
| 8 | `fromString falls back to priceSchedule for unknown values` | PASS |

#### IndicatorCondition (7 tests)

| # | Test | Status |
|---|------|--------|
| 9 | `chipLabel for priceSchedule with API condition` → "Price <= 150.0" | PASS |
| 10 | `chipLabel for priceSchedule with symbol condition` → "Price >= 200.0" | PASS |
| 11 | `chipLabel for superTrend` → "ST(10,3.0) bullish" | PASS |
| 12 | `chipLabel for movingAverage` → "EMA(20) above" | PASS |
| 13 | `chipLabel for trailingStopLoss without activation` → "TSL 5%" | PASS |
| 14 | `chipLabel for trailingStopLoss with activation price` → "TSL 3% @ 60000" | PASS |
| 15 | `JSON round-trip` preserves type + params | PASS |

#### AutomationStrategy (4 tests)

| # | Test | Status |
|---|------|--------|
| 16 | `JSON round-trip preserves all fields` | PASS |
| 17 | `JSON round-trip handles null serverStrategyId` | PASS |
| 18 | `copyWith updates specified fields only` | PASS |
| 19 | `copyWith sets isSynced independently` | PASS |

#### AutomationRule (7 tests)

| # | Test | Status |
|---|------|--------|
| 20 | `JSON round-trip preserves all fields` (including TSL) | PASS |
| 21 | `JSON handles optional fields gracefully` | PASS |
| 22 | `isExpired returns true for past expiresAt` | PASS |
| 23 | `isExpired returns true when status is "expired"` | PASS |
| 24 | `isExpired returns false for future expiresAt with pending status` | PASS |
| 25 | `status helpers` (isPending, isExecuted, isFailed) | PASS |
| 26 | `copyWith updates specified fields only` | PASS |

### 2.2 Repository Persistence Tests (17 tests)

**File:** `automation_repo_test.dart`
**Scope:** SharedPreferences CRUD — uses `SharedPreferences.setMockInitialValues({})`

#### Strategy Persistence (7 tests)

| # | Test | Status |
|---|------|--------|
| 27 | `persist and load single strategy` | PASS |
| 28 | `persist and load multiple strategies` | PASS |
| 29 | `strategy with trailingStopLoss indicator type round-trips` | PASS |
| 30 | `empty storage returns empty list` | PASS |
| 31 | `corrupted JSON returns empty list safely` | PASS |
| 32 | `update existing strategy preserves order` | PASS |
| 33 | `delete by removing from list` | PASS |

#### Rule Persistence (10 tests)

| # | Test | Status |
|---|------|--------|
| 34 | `persist and load single rule` | PASS |
| 35 | `rule with TSL indicator round-trips` (trailPercent + activationPrice) | PASS |
| 36 | `filter active rules for a specific coin` | PASS |
| 37 | `filter rules by strategy ID` | PASS |
| 38 | `rule status helpers work after persistence` | PASS |
| 39 | `rule copyWith persists updated fields` | PASS |
| 40 | `bulk delete rules for a strategy` | PASS |
| 41 | `mixed indicator types persist correctly` (priceSchedule + TSL) | PASS |
| 42 | `empty storage returns empty list` | PASS |
| 43 | `large number of rules persist and load correctly` (100 rules) | PASS |

### 2.3 Controller Tests (26 tests)

**File:** `automation_controller_test.dart`
**Scope:** `AutomationRuleFormController` state management — GetX test mode + mock `IApiService`

#### startCreatingNew (5 tests)

| # | Test | Status |
|---|------|--------|
| 44 | `sets isCreatingNew to true` | PASS |
| 45 | `clears selectedStrategy` | PASS |
| 46 | `resets pendingIndicatorTypes` | PASS |
| 47 | `resets strategyMode to 0` | PASS |
| 48 | `resets indicators and duration` | PASS |

#### toggleIndicatorType (4 tests)

| # | Test | Status |
|---|------|--------|
| 49 | `adds type when not selected` | PASS |
| 50 | `removes type when already selected` | PASS |
| 51 | `respects maxIndicators limit (3)` | PASS |
| 52 | `isIndicatorTypeSelected reflects state` | PASS |

#### setStrategyMode (2 tests)

| # | Test | Status |
|---|------|--------|
| 53 | `sets mode value` | PASS |
| 54 | `clears pendingIndicatorTypes on mode change` | PASS |

#### addIndicator / removeIndicator (5 tests)

| # | Test | Status |
|---|------|--------|
| 55 | `addIndicator adds a condition` | PASS |
| 56 | `addIndicator replaces existing condition of same type` | PASS |
| 57 | `addIndicator respects maxIndicators limit` | PASS |
| 58 | `removeIndicator removes at index` | PASS |
| 59 | `removeIndicator ignores invalid index (-1, 10)` | PASS |

#### Validation (5 tests)

| # | Test | Status |
|---|------|--------|
| 60 | `canSaveStrategy returns false when no types selected` | PASS |
| 61 | `canSaveStrategy returns true when >= 1 type selected` | PASS |
| 62 | `canSaveRule returns false when no strategy selected` | PASS |
| 63 | `canSaveRule returns false when required indicators incomplete` | PASS |
| 64 | `canSaveRule returns true when all required indicators provided` | PASS |

#### Duration & Misc (5 tests)

| # | Test | Status |
|---|------|--------|
| 65 | `effectiveDays returns preset days (7, 20)` | PASS |
| 66 | `effectiveDays returns customDays for oneMonth` | PASS |
| 67 | `selectStrategy sets strategy and clears creation mode` | PASS |
| 68 | `reset clears all state` | PASS |
| 69 | `resetForTokenChange preserves strategy list` | PASS |

---

## 3. okto-bff — Architecture Justification

The BFF's strategy module is a **thin passthrough proxy** — each method extracts auth context, assembles query/path parameters, and delegates to `apiCallWithCb()`. There is no business logic, validation, or data transformation, making it a low-value target for unit tests.

### Proxied Endpoints (12)

| BFF Function | HTTP Method | HL Service Path |
|-------------|-------------|-----------------|
| `CreateStrategyTemplate` | POST | `/strategy-templates` |
| `ListStrategyTemplates` | GET | `/strategy-templates` |
| `GetStrategyTemplate` | GET | `/strategy-templates/:tmpl_id` |
| `UpdateStrategyTemplate` | PATCH | `/strategy-templates/:tmpl_id` |
| `DeleteStrategyTemplate` | DELETE | `/strategy-templates/:tmpl_id` |
| `CreateAutomationRule` | POST | `/automation-rules` |
| `ListAutomationRules` | GET | `/automation-rules` |
| `GetAutomationRule` | GET | `/automation-rules/:rule_id` |
| `DeleteAutomationRule` | DELETE | `/automation-rules/:rule_id` |
| `AttachPendingOrder` | POST | `/automation-rules/:rule_id/pending-orders` |
| `ListPendingOrders` | GET | `/automation-rules/:rule_id/pending-orders` |
| `CancelPendingOrder` | DELETE | `/automation-rules/:rule_id/pending-orders/:order_id` |

### Risk Assessment

| Risk | Impact | Mitigation |
|------|--------|------------|
| Auth bypass in production | Critical | `ENV` check gates it; only active when `ENV=DEVELOPMENT` |
| HL service down | High | Circuit breaker via `apiCallWithCb` |
| Query param injection | Low | Dev bypass only reads `user_id`/`account_owner_id` |

---

## 4. Coverage by Feature

| Feature | Backend (Go) | Frontend (Flutter) | Total |
|---------|-------------|-------------------|-------|
| Strategy Templates (CRUD) | 16 handler + 8 validation | 4 model + 7 repo + 3 controller | **38** |
| Automation Rules (CRUD) | 13 handler + 12 validation + 19 repo | 7 model + 10 repo + 5 controller | **66** |
| Pending Orders | 2 handler + 3 validation + 13 repo | — | **18** |
| TSL Indicator | 12 executor + 7 Redis | 2 model + 2 repo | **23** |
| Price Schedule Indicator | 5 executor | 2 model | **7** |
| Order Signing | 3 E2E + 2 payload conversion | — | **5** |
| Executor Engine | 9 (sweep, tick, failure, lock, market hours) | — | **9** |
| Form State / UI Logic | — | 26 controller | **26** |
| Data Persistence | — | 17 repo | **17** |

---

## 5. Mock Infrastructure

### Go Mocks

| Mock | Package | Implements |
|------|---------|------------|
| `mockStrategySvc` | `api/.../strategy` (test file) | `StrategyServiceInterface` (14 methods) |
| `AutomationRulesRepo` | `internal/repository/automationrules/mocks` | `AutomationRulesRepositoryInterface` |
| `StrategyPendingOrdersRepo` | `internal/repository/strategypendingorders/mocks` | `StrategyPendingOrdersRepositoryInterface` |
| `hlMock` (test-local) | `internal/service/strategy` (test file) | `exchangeService` (4 methods) |
| `Cache[string, string]` | `pkg/cache/mocks` | `cache.Cache[string, string]` |

### Flutter Mocks

| Mock | Technique | Purpose |
|------|-----------|---------|
| `SharedPreferences` | `setMockInitialValues({})` | Repository test isolation |
| `IApiService` | Mock (mocktail) | Controller test — API call simulation |

---

## 6. How to Run

### Go Backend Tests

```bash
cd okto-hl-service

# All 132 strategy tests
go test -v -count=1 \
  ./internal/service/strategy/ \
  ./internal/repository/automationrules/ \
  ./internal/repository/strategypendingorders/ \
  ./apicommon/types/ \
  ./api/apiorder/handler/vpc/v1/strategy/

# Just handler tests
go test -v -count=1 ./api/apiorder/handler/vpc/v1/strategy/

# Just validation tests
go test -v -count=1 ./apicommon/types/

# Just repository tests
go test -v -count=1 ./internal/repository/automationrules/ ./internal/repository/strategypendingorders/

# Just Redis integration tests
go test -v -count=1 -run "TestTSL_Redis" ./internal/service/strategy/

# Just signing E2E tests
go test -v -count=1 -run "TestBuildAndSign" ./internal/service/strategy/
```

### Flutter Tests

```bash
cd coindcx_app-production

# All automation tests
flutter test test/components/dcx_us_perps/automation/ --no-pub

# Individual files
flutter test test/components/dcx_us_perps/automation/automation_models_test.dart --no-pub
flutter test test/components/dcx_us_perps/automation/automation_repo_test.dart --no-pub
flutter test test/components/dcx_us_perps/automation/automation_controller_test.dart --no-pub

# Verbose output
flutter test test/components/dcx_us_perps/automation/ --no-pub --reporter expanded
```

### BFF Tests

```bash
cd okto-bff

# All BFF tests (non-strategy)
go test -v ./...
```

---

## 7. Gaps & Recommended Next Steps

| Priority | Area | Type | Description |
|----------|------|------|-------------|
| **P0** | AutomationApiService (Flutter) | Unit | Mock `IApiService.execute()` for HTTP layer testing |
| **P0** | Auth middleware dev bypass (BFF) | Unit | Verify `user_id` injection when `ENV=DEVELOPMENT` |
| **P1** | Strategy Service layer (Go) | Unit | Business logic between handler and repository |
| **P1** | saveStrategy / saveRule (Flutter) | Integration | Mocked API sync + local persistence |
| **P1** | Route registration (BFF) | Integration | Verify all 12 routes map to correct handlers |
| **P2** | Widget tests (Flutter) | Widget | Bottom sheet and picker UI tests |
| **P2** | Negative SQL tests (Go) | Integration | DB constraint violations (duplicate IDs, FK) |
| **P2** | Circuit breaker (BFF) | Unit | Strategy calls respect circuit breaker state |
| **P3** | Load/stress testing | Performance | `go test -bench` with higher `-benchtime`, memory profiling |
| **P3** | SharedPreferences scaling (Flutter) | Performance | Profile encode/decode with 500+ rules |
