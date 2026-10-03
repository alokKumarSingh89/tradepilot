# Implementation Plan: Multi-Timeframe Indicator-Based Strategy Engine

**Branch**: `003-multi-timeframe-indicator-based-strategy-engine` | **Date**: 2026-10-03 | **Spec**: [spec.md](spec.md)

**Input**: Feature specification from `/specs/003-multi-timeframe-indicator-based-strategy-engine/spec.md`

## Summary

This feature adds an indicator-based multi-timeframe strategy runtime to the approved Feature 002 orchestration and execution model. The design keeps the Feature 002 lifecycle, run-state model, market-data distribution, event-driven risk monitoring, persistence model, and restart recovery as the base architecture, then adds a reusable strategy runtime that manages timeframe roles, indicator evaluation, higher-timeframe directional bias, independent main/supporting trade legs, and audit-friendly signal suppression.

The design preserves the existing modular monolith and dedicated worker model, adds a strategy-engine runtime layer that reads immutable configuration snapshots, and keeps all feature-specific state explicit and auditable. It is additive by design and does not replace the earlier algorithm-management architecture.

## Technical Context

**Language/Version**: Python 3.12+, FastAPI, TypeScript 5.x, React 18/19-compatible frontend, SQLAlchemy 2, Alembic

**Primary Dependencies**: FastAPI, PostgreSQL, SQLAlchemy 2, Alembic, Pydantic v2, React, TypeScript, Pytest, Vitest, Docker Compose

**Storage**: PostgreSQL with SQLAlchemy ORM and Alembic migrations; strategy and run state persisted alongside the Feature 002 configuration, run, snapshot, order, and risk records

**Testing**: Pytest for backend logic and API integration, Vitest for UI validation, integration tests for orchestration and recovery behavior

**Target Platform**: Local Docker Compose services with backend, PostgreSQL, worker runtime, and web UI; browser-based strategy configuration and monitoring console

**Project Type**: Web application with a modular monolith backend and a dedicated worker runtime extension to Feature 002

**Performance Goals**: Support multiple concurrent runs per user with independent trade leg state; event-driven risk and signal evaluation on normalized candle updates; low-latency UI refresh after strategy state or risk transitions

**Constraints**: Paper trading only for MVP; no live broker execution; all strategy evaluation decisions must be deterministic and completed-candle-safe by default; duplicate signals and late/out-of-order data must be suppressed or ignored by policy; run recovery must be explicit rather than silent

**Scale/Scope**: Multi-algorithm portfolio workflows with concurrent paper-trading runs, each supporting one higher-timeframe bias and multiple lower-timeframe supporting legs; not a distributed microservice architecture

## Constitution Check

_GATE: Must pass before Phase 0 research. Re-check after Phase 1 design._

Passes the TradePilot Constitution because the design:

- preserves separation of concerns between strategy evaluation, market-data ingestion, risk monitoring, and execution orchestration;
- keeps long-running strategy evaluation inside the Worker runtime and outside HTTP request handlers;
- keeps paper trading as the default and requires explicit activation before any live execution path is considered;
- treats restart recovery, duplicate prevention, and stale-data handling as first-class logic rather than silent automation;
- stays within the approved modular monolith architecture and avoids premature service decomposition;
- makes the strategy engine auditable and testable via deterministic indicator validation and run snapshots.

No constitution waivers are required for Feature 003. The design remains additive to Feature 002 and does not alter the governing safety or architecture principles.

## Review Against Feature 002 Architecture

This plan was reviewed against the approved Feature 002 architecture and compatibility requirements:

- Core lifecycle state management remains owned by Feature 002: configuration versions, run status, snapshots, risk state, order events, and execution history.
- The new engine does not create a competing orchestration system. It plugs into the same algorithm worker contract and market-data distribution model.
- Strategy-specific runtime state is modeled as additive data, not a separate run execution model.
- Existing run recovery, idempotent command handling, and user-visible lifecycle statuses remain authoritative and are extended with signal and trade-leg state.

## Planned Compatibility Changes to Feature 002

The following changes are proposed as additive compatibility changes and are not a rewrite of the approved Feature 002 model:

1. Add a strategy_runtime_payload to AlgorithmConfiguration and AlgorithmRun so multi-timeframe algorithm definitions can be persisted without changing the command model.
2. Extend RunSnapshot with strategy runtime and trade-leg state so restart recovery includes the indicator engine context and supporting-leg state.
3. Add a strategy engine adapter to the worker runtime contract for runtime-specific signal evaluation and trade-leg state transitions.
4. Extend LifecycleEvent and RiskState with signal, reversal, and suppression events so strategy decisions remain auditable and visible in the same operational history.
5. Add a strategy runtime detail view to the frontend while preserving feature 002 run cards, lifecycle status panels, and risk monitoring surfaces.

These are additive extension points, not a replacement of the existing Feature 002 architecture.

## Project Structure

### Documentation (this feature)

```text
specs/003-multi-timeframe-indicator-based-strategy-engine/
├── spec.md              # business requirements and accepted clarifications
├── plan.md              # this document
├── research.md          # research and design decisions
├── data-model.md        # runtime and persistence model for Feature 003
├── quickstart.md        # validation checklist and run guide
├── contracts/           # runtime and interface contract extensions
│   └── multi-timeframe-strategy-runtime.md
├── checklists/
│   └── requirements.md
└── tasks.md             # not generated by /speckit-plan; reserved for /speckit-tasks
```

### Source Code (repository root)

```text
backend/
├── app/
│   ├── api/
│   │   ├── routes/
│   │   └── schemas/
│   ├── core/
│   │   ├── config/
│   │   └── logging/
│   ├── db/
│   │   ├── models/
│   │   ├── repositories/
│   │   └── migrations/
│   ├── domain/
│   │   ├── algorithms/
│   │   ├── market_data/
│   │   ├── risk/
│   │   ├── strategies/
│   │   │   ├── runtime/
│   │   │   ├── indicators/
│   │   │   ├── timeframes/
│   │   │   ├── rules/
│   │   │   └── trade_legs/
│   │   └── execution/
│   ├── services/
│   │   ├── algorithm_orchestrator/
│   │   ├── market_data_adapter/
│   │   ├── strategy_runtime_manager/
│   │   └── recovery/
│   └── workers/
│       └── algorithm_worker/
├── tests/
│   ├── api/
│   ├── integration/
│   ├── unit/
│   └── strategies/
├── alembic.ini
└── pyproject.toml

frontend/
├── src/
│   ├── features/
│   │   ├── algorithms/
│   │   ├── runs/
│   │   └── strategy-runtime/
│   ├── services/
│   ├── components/
│   └── routes/
├── tests/
│   └── unit/
└── package.json

worker/
├── app/
│   ├── orchestrator/
│   ├── strategy_runtime/
│   ├── risk_runtime/
│   └── queue/
└── tests/

docker-compose.yml
```

**Structure Decision**: The design remains a modular monolith with a dedicated worker runtime extension. The strategy runtime is implemented as a domain layer under backend/app/domain/strategies with explicit indicator, timeframe, trade-leg, and rule modules. This matches the approved Feature 002 architecture and keeps the new functionality additive without introducing its own orchestration system.

## Complexity Tracking

No constitution violations or exceptional complexity exceptions are expected for this feature. The architecture remains modular and cohesive, and the design intentionally avoids unnecessary service boundaries or parallel systems.
