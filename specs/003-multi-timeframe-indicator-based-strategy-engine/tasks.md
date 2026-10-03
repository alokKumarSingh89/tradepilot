# Tasks: Multi-Timeframe Indicator-Based Strategy Engine

**Input**: Design documents from `/specs/003-multi-timeframe-indicator-based-strategy-engine/`

**Status**: Provisional pending consistency analysis.

**Prerequisites**: plan.md (required), spec.md (required for user stories), research.md, data-model.md, contracts/

**Architecture constraint**: Reuse the Feature 002 modular monolith, algorithm worker, lifecycle model, risk workflow, and persistence patterns. Do not create a second algorithm orchestrator or a competing order execution engine.

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Confirm the shared runtime, database, and project structure required to support Feature 003 without duplicating Feature 002 ownership.

- [ ] T001 Create Feature 003 task scaffolding and document the provisional dependency boundaries in `specs/003-multi-timeframe-indicator-based-strategy-engine/tasks.md`
- [ ] T002 [P] Review and confirm the Feature 002 run lifecycle, snapshot, and risk contracts for compatibility with Feature 003 in `specs/002-multi-algorithm-management/plan.md`, `research.md`, `data-model.md`, and `contracts/`
- [ ] T003 [P] Define the Feature 003 folder structure under `backend/app/domain/strategies/`, `backend/app/services/`, `backend/tests/strategies/`, and `frontend/src/features/strategy-runtime/` per the plan
- [ ] T004 [P] Confirm PostgreSQL migration layout and Alembic change set strategy for Feature 003-specific tables while preserving Feature 002 tables and naming conventions
- [ ] T005 [P] Set up backend and frontend test harnesses for strategy runtime validation in `backend/tests/strategies/` and `frontend/src/features/strategy-runtime/`

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Establish the shared contracts and model boundaries that Feature 003 depends on before any story-level implementation begins.

**Critical rule**: No user-story implementation may begin until this phase is complete.

- [ ] T006 Document the Feature 002 dependency map and explicit interface ownership for configuration lifecycle, run lifecycle, duplicate-run prevention, recovery authorization, risk state, order execution, and run recovery; Feature 003 remains subordinate to Feature 002 lifecycle authority
- [ ] T007 [P] Define the Feature 003 runtime adapter contract that maps strategy signals and derived trade-leg state to the Feature 002 risk and execution workflow in `backend/app/services/strategy_runtime_manager/` without creating a competing risk or order mechanism
- [ ] T008 [P] Define the strategy runtime persistence contract for `StrategyRuntimeConfiguration`, `TimeframeDefinition`, `IndicatorDefinition`, `IndicatorRule`, `CandleBucket`, `SignalEvent`, and `TradeLegRuntimeState` in the database model layer
- [ ] T009 [P] Define the canonical validation rules for the feature in the service layer, including: "At least two timeframes must be configured", "Each timeframe must have an explicit role", "Indicator parameter sets must validate against the registry schema", and "completed-candle evaluation is the default safe behavior"
- [ ] T010 [P] Define the immutable snapshot and run-recovery extension points for Feature 003 in the shared model and repository contracts
- [ ] T011 [P] Define the worker-side integration contract so the Feature 002 algorithm worker can dispatch strategy messages to Feature 003 without creating a competing orchestrator
- [ ] T012 Establish the repository and migration boundaries for the Feature 003 runtime tables and the Feature 002-compatible extension points in `backend/app/db/models/` and `backend/app/db/repositories/`
- [ ] T013 Define the logging and audit model for suppressed signals, reversal events, late-data handling, and run history in the existing lifecycle event structure
- [ ] T014 Create a shared checklist for unit, integration, and restart-recovery validation before user-story implementation begins

**Checkpoint**: Foundation ready - user story implementation can now begin in parallel.

---

## Phase 3: User Story 1 - Configure multi-timeframe strategies and indicators (Priority: P1) 🎯 MVP

**Goal**: Allow users to create a multi-timeframe strategy configuration with explicit timeframe roles, indicator definitions, and validation checks while reusing the Feature 002 configuration and run lifecycle.

**Independent Test**: A user can save a valid 4H + 15M strategy configuration with indicator parameters and retrieve it without creating duplicate run state.

### Tests for User Story 1

- [ ] T015 [P] [US1] Contract test for strategy runtime configuration API in `backend/tests/api/test_strategy_runtime_api.py`
- [ ] T016 [P] [US1] Integration test for strategy save and retrieval flow in `backend/tests/integration/test_strategy_runtime_config.py`
- [ ] T017 [P] [US1] Validation test for invalid indicator parameters, missing timeframe roles, and unsupported indicator types in `backend/tests/unit/test_strategy_validation.py`

### Implementation for User Story 1

- [ ] T018 [P] [US1] Add `StrategyRuntimeConfiguration` model in `backend/app/db/models/strategy_runtime.py`
- [ ] T019 [P] [US1] Add `TimeframeDefinition` and `IndicatorDefinition` models in `backend/app/db/models/strategy_runtime.py`
- [ ] T020 [US1] Implement the strategy runtime configuration repository and version access pattern in `backend/app/db/repositories/strategy_runtime_repository.py`
- [ ] T021 [US1] Implement the Feature 003 configuration validation service for timeframe-role assignment, indicator registry compatibility, warm-up requirements, and completed-candle defaults in `backend/app/domain/strategies/runtime/configuration_service.py`
- [ ] T022 [US1] Implement the indicator registry contract and parameter validation for MACD and Heikin Ashi in `backend/app/domain/strategies/indicators/`
- [ ] T023 [US1] Add the strategy runtime API routes and schemas for creation, retrieval, and versioned save behavior in `backend/app/api/routes/strategy_runtime.py` and `backend/app/api/schemas/strategy_runtime.py`
- [ ] T024 [US1] Ensure the runtime configuration is tied to the Feature 002 configuration version and run lifecycle without mutating active runs
- [ ] T025 [US1] Add audit logging for invalid configuration attempts, suppressed unsupported configurations, and successful save events tied to the existing lifecycle event model

**Checkpoint**: At this point, User Story 1 should be fully functional and independently testable.

---

## Phase 4: User Story 2 - Evaluate multi-timeframe signals and manage trade-leg state (Priority: P1)

**Goal**: Build the signal evaluation pipeline that calculates timeframe-aligned candles, evaluates MACD and HA rules, applies higher-timeframe bias, and maintains independent main and supporting trade-leg states under the Feature 002 run model.

**Independent Test**: A strategy run with a valid 4H bias and 15M supporting conditions can create, suppress, and maintain trade-leg state without duplicate execution or cross-leg contamination.

### Tests for User Story 2

- [ ] T026 [P] [US2] Unit test for candle normalization and completed-candle evaluation in `backend/tests/unit/test_candle_pipeline.py`
- [ ] T027 [P] [US2] Unit test for MACD and Heikin Ashi rule evaluation in `backend/tests/unit/test_indicator_rules.py`
- [ ] T028 [P] [US2] Integration test for higher-timeframe directional bias and lower-timeframe supporting entries in `backend/tests/integration/test_multi_timeframe_signal_evaluation.py`
- [ ] T029 [P] [US2] Integration test for duplicate-signal suppression, suppressed audit logging, and opposite-direction handling in `backend/tests/integration/test_signal_suppression.py`

### Implementation for User Story 2

- [ ] T030 [P] [US2] Add `CandleBucket` and `SignalEvent` models in `backend/app/db/models/strategy_runtime.py`
- [ ] T031 [P] [US2] Add `TradeLegRuntimeState` model and the run-level state relationship to `backend/app/db/models/strategy_runtime.py`
- [ ] T032 [US2] Implement the timeframe registry and session-boundary normalization in `backend/app/domain/strategies/timeframes/`
- [ ] T033 [US2] Implement the indicator registry for MACD and Heikin Ashi and enforce deterministic warm-up behavior in `backend/app/domain/strategies/indicators/`
- [ ] T034 [US2] Implement the completed-candle pipeline and late/out-of-order data policy in `backend/app/domain/strategies/runtime/candle_pipeline.py`
- [ ] T035 [US2] Implement the rule-evaluation engine that supports primary trend, confirmation, and entry/exit conditions in `backend/app/domain/strategies/runtime/rule_evaluator.py`
- [ ] T036 [US2] Implement signal generation and duplicate suppression keyed by run, trade leg, direction, and candle window in `backend/app/domain/strategies/runtime/signal_manager.py`
- [ ] T037 [US2] Implement the main/supporting trade state manager that maintains independent P&L, state, and lifecycle transitions in `backend/app/domain/strategies/trade_legs/`
- [ ] T038 [US2] Integrate the signal output with the Feature 002 risk and execution pipeline without creating a competing execution engine

**Checkpoint**: The signal pipeline and trade-leg runtime should be independently testable at this point.

---

## Phase 5: User Story 3 - Integrate risk and execution semantics for main and supporting trades (Priority: P1)

**Goal**: Ensure the Feature 003 runtime integrates with the Feature 002 risk engine and order execution semantics for independent trade legs, optional shared budget, and reversal handling.

**Independent Test**: A single strategy run with a main trade and supporting trades can manage independent risk without silent cross-leg contamination or duplicate exits.

### Tests for User Story 3

- [ ] T039 [P] [US3] Unit test for shared risk budget enforcement and independent leg budgets in `backend/tests/unit/test_trade_leg_risk.py`
- [ ] T040 [P] [US3] Contract test for reversal and trade-leg state transitions in `backend/tests/integration/test_trade_leg_reversal.py`
- [ ] T041 [P] [US3] Integration test for supporting-trade entry/exit cap enforcement and immediate close policy in `backend/tests/integration/test_supporting_trade_behaviour.py`
- [ ] T042 [P] [US3] Backend API test for strategy runtime detail and signal history retrieval in `backend/tests/api/test_strategy_runtime_detail_api.py`

### Implementation for User Story 3

- [ ] T043 [P] [US3] Add `SharedRiskBudget` and its run-scoped aggregation model in `backend/app/db/models/strategy_runtime.py` as policy context for the authoritative shared risk engine
- [ ] T044 [US3] Implement risk budget evaluation logic that respects per-leg risk and optional shared budget aggregation in `backend/app/domain/strategies/runtime/risk_budget_manager.py`, while leaving enforcement to the shared risk engine
- [ ] T045 [US3] Implement reversal policy orchestration that closes or pauses supporting trades when the higher-timeframe bias reverses, honoring explicit user override policy in `backend/app/domain/strategies/runtime/reversal_manager.py` and reconciling against authoritative execution records
- [ ] T046 [US3] Integrate strategy signal output with the Feature 002 risk event and order-intent contracts without creating a new order engine or bypassing the shared execution path
- [ ] T047 [US3] Add strategy runtime status and event exposure to the existing run history and monitoring workflows in `backend/app/services/run_lifecycle/` and `frontend/src/features/runs/` without inventing a competing lifecycle model
- [ ] T048 [US3] Add paper-trading gating for strategy runtime so live execution remains disabled unless explicitly enabled outside this feature

**Checkpoint**: Main and supporting trade behavior should now be independently validated against risk and execution integration.

---

## Phase 6: User Story 4 - Restart recovery, concurrency safety, and auditability (Priority: P2)

**Goal**: Ensure the strategy runtime recovers safely after restarts, prevents duplicate starts and duplicate execution, and maintains a recoverable history for signal evaluation and trade-leg changes.

**Independent Test**: A run can recover from state after a restart without duplicating signals, order intents, or trade-leg state.

### Tests for User Story 4

- [ ] T049 [P] [US4] Integration test for restart recovery of a strategy runtime and snapshot replay in `backend/tests/integration/test_strategy_runtime_recovery.py`
- [ ] T050 [P] [US4] Concurrency test for duplicate start and duplicate signal processing prevention in `backend/tests/integration/test_strategy_runtime_concurrency.py`
- [ ] T051 [P] [US4] Persistence test for immutable snapshot integrity and suppression audit logs in `backend/tests/unit/test_strategy_snapshot_audit.py`

### Implementation for User Story 4

- [ ] T052 [P] [US4] Extend the Feature 002 run snapshot contract with strategy runtime payload and suppressed-signal history in `backend/app/db/models/` and `backend/app/services/recovery/`, while keeping Feature 002 lifecycle authority intact
- [ ] T053 [US4] Implement safe restart reconciliation that restores the latest valid runtime state without reprocessing finalized candle decisions by default and without authorizing new strategy entries before Feature 002 restart/recovery approval
- [ ] T054 [US4] Implement duplicate execution guardrails for run-level start/stop and signal-level de-duping keyed by run, trade leg, direction, and candle window while preserving Feature 002 duplicate-run prevention
- [ ] T055 [US4] Add recovery and degraded-state handling for stale or delayed market data in the strategy runtime manager and worker integration without bypassing the approved recovery contract
- [ ] T056 [US4] Validate that the Feature 003 runtime remains compatible with the Feature 002 user-confirmed stop and recover workflow without bypassing recovery, duplicate-run, or risk safety gates

**Checkpoint**: Restart recovery and concurrency safety should now be independently testable for Feature 003.

---

## Phase 7: Polish & Cross-Cutting Concerns

**Purpose**: Finalize the runtime consistency, documentation, and validation coverage across the feature.

- [ ] T057 [P] Review and tighten the Feature 003 plan, research, data model, and contracts for consistency with the approved Feature 002 architecture
- [ ] T058 [P] Add documentation or UI copy describing the strategy runtime, trade-leg state, suppression audit, and run-recovery semantics in `frontend/src/features/strategy-runtime/` and `backend/app/api/schemas/`
- [ ] T059 [P] Run the strategy runtime validation checklist from `quickstart.md` and confirm the main/supporting-trade scenarios, recovery cases, and concurrency cases are covered
- [ ] T060 [P] Confirm that all tasks follow the checklist format with IDs, file paths, and story labels required by the Spec Kit task template
- [ ] T061 [P] Confirm the feature remains additive to Feature 002 and that no competing orchestrator or execution engine has been introduced

---

## Dependencies & Execution Order

### Phase Dependencies

- **Phase 1 (Setup)**: no dependencies; can start immediately
- **Phase 2 (Foundational)**: depends on Phase 1; blocks all story implementation
- **Phase 3 (US1)**: depends on Phase 2
- **Phase 4 (US2)**: depends on Phase 2; may run in parallel with US1 if the team is staffed for it
- **Phase 5 (US3)**: depends on Phase 2; may run in parallel with US1/US2 when integration boundaries are clear
- **Phase 6 (US4)**: depends on Phase 2 and the runtime state model created in US1/US2
- **Phase 7 (Polish)**: depends on all desired stories being complete

### User Story Dependencies

- **User Story 1 (P1)**: can start after Phase 2; no dependency on other stories
- **User Story 2 (P1)**: depends on the runtime contract and database foundation from Phase 2; also depends on US1 configuration persistence
- **User Story 3 (P1)**: depends on US1 configuration and US2 signal/trade-leg state; should be integrated with the Feature 002 risk and execution contracts
- **User Story 4 (P2)**: depends on US1/US2/US3 lifecycle model and recovery surfaces; should run after runtime logic is stable

### Parallel Opportunities

- Setup tasks T002-T005 can run in parallel.
- Foundational tasks T007-T013 can run in parallel once the contract boundaries are stable.
- Test tasks within each user story can run in parallel.
- Data models and service definitions within a user story can run in parallel when they touch different modules.
- Story work for US1, US2, and US3 can proceed in parallel once the foundational contracts are complete, provided the runtime adapter and risk integration interfaces remain unchanged.

---

## Implementation Strategy

### MVP First (User Story 1 + core of US2)

1. Complete Phase 1 Setup.
2. Complete Phase 2 Foundation.
3. Complete User Story 1 configuration and validation.
4. Complete the core of User Story 2 for signal generation and trade-leg state.
5. Stop and validate the core runtime before additional risk and recovery features.

### Incremental Delivery

1. Add configuration and indicator parsing.
2. Add candle pipeline and rule evaluation.
3. Add trade-leg management and shared risk integration.
4. Add restart recovery and concurrency safeguards.
5. Validate the complete feature against the Feature 002 lifecycle and risk architecture.

### Parallel Team Strategy

With multiple developers:

1. Team completes Setup + Foundational tasks together.
2. Once the foundational contracts are complete:
   - Developer A: User Story 1 configuration + validation
   - Developer B: User Story 2 signal pipeline + trade-leg state
   - Developer C: User Story 3 risk + execution integration
   - Developer D: User Story 4 recovery + concurrency tests
3. Final polish runs after all user stories are complete.

---

## Notes

- All tasks intentionally remain planning-only and do not implement application code.
- No second algorithm orchestrator or duplicate order execution engine is allowed.
- Feature 002 ownership remains authoritative for lifecycle, persistence, and recovery behavior.
- Feature 003 extends that model with runtime-specific strategy evaluation, signal handling, trade-leg state, and auditability.
- Automated tests are explicitly included for validation, including strategy configuration, signal suppression, main/supporting trade scenarios, recovery, and concurrency.
