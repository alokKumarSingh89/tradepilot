# Implementation Plan: Multi-Algorithm Management and Execution Lifecycle

**Branch**: `002-multi-algorithm-management` | **Date**: 2026-10-03 | **Spec**: [spec.md](spec.md)

**Input**: Feature specification from `/specs/002-multi-algorithm-management/spec.md`

## Summary

This feature delivers a multi-algorithm management layer for TradePilot where users can create and version algorithm configurations, launch and monitor independent paper-trading runs, and recover safely after restarts. The plan uses a modular monolith with a dedicated algorithm worker process to keep lifecycle orchestration, risk evaluation, and HTTP request handling separate, while still avoiding unnecessary distributed infrastructure. The system persists configuration history, run snapshots, positions, orders, and execution events in PostgreSQL, and exposes a React/TypeScript management interface for create, edit, start, stop, and monitor workflows.

## Technical Context

**Language/Version**: Python 3.13+, TypeScript 5.x, React 18/19-compatible frontend, SQLAlchemy 2, Alembic

**Primary Dependencies**: FastAPI, SQLAlchemy 2, Alembic, PostgreSQL, Pydantic v2, React, TypeScript, Vitest, Pytest, Docker Compose

**Storage**: PostgreSQL with SQLAlchemy ORM and Alembic migrations; persisted algorithm configs, versions, runs, snapshots, positions, orders, and execution history

**Testing**: Pytest for backend services and business logic; Vitest for React UI behavior; integration tests for API and database workflows

**Target Platform**: Local Docker Compose environments for backend, database, worker, and UI; browser-based management console connected to the platform API

**Project Type**: Web application with a modular monolith backend and a separately deployable algorithm worker process

**Performance Goals**: Support multiple concurrent runs per user with independent risk state; handle event-driven updates with low-latency risk evaluation; UI list/detail refresh within a few seconds after state change

**Constraints**: Paper trading only for MVP; no live broker or live market order placement; all long-running execution loops must remain outside HTTP handlers; duplicate start, stop, and exit commands must be idempotent and concurrency-safe

**Scale/Scope**: MVP supports multiple algorithm configurations and concurrent paper-trading runs per user, with isolated state and restart recovery; not a multi-tenant large-scale trading engine.

## Constitution Check

_GATE: Must pass before Phase 0 research. Re-check after Phase 1 design._

Passes the TradePilot Constitution because the design:

- preserves separation of concerns between market data, risk logic, and execution orchestration;
- keeps long-running execution off HTTP request handlers;
- uses paper-trading defaults and explicit recovery decisions rather than silent automation;
- keeps reliability and restart safety as first-class behavior;
- favors a modular monolith over premature microservices, matching the product’s need for operational clarity and controlled incremental growth;
- includes explicit testing expectations for backend and frontend behavior.

No constitution waivers are required for this feature.

## Review Against Approved Specification

The plan was reviewed against the approved Feature 002 specification and the Constitution. The following gaps were corrected to keep design fidelity high:

- The plan explicitly preserves per-run isolation and versioned configuration history, which is required by FR-006 through FR-019.
- The design now treats stop actions and recovery as explicit user-confirmed workflows instead of silent state changes, matching the approved clarifications.
- The plan states that duplicate starts, duplicate stop attempts, and restart recovery must be idempotent and database-protected, aligning with the safety and reliability principles.
- Monitoring requirements are made explicit at the UI and API level, including list/detail status views and lifecycle history, matching the user-facing monitoring requirement.
- Failure-mode handling was expanded to include stale market data, worker crashes, partial fills, and reconciliation after restarts.

## Failure Scenarios the Design Must Cover

The feature design must address the following scenarios before implementation begins:

- stale or delayed market data while a run is active and risk thresholds remain pending;
- worker process crash or restart while a run is in `running` state and before a final snapshot is persisted;
- rapid duplicate start or stop requests for the same run, configuration, or logical position group;
- partial fill or partial stop signals from simulated execution without a silent success assumption;
- user edits that create a new configuration version while a run remains active and bound to its original version;
- recovery prompts after restart that require explicit `recover`, `restart`, or `discard` confirmation rather than silent auto-resume;
- graceful shutdown while other runs continue unaffected.

## Project Structure

### Documentation (this feature)

```text
specs/002-multi-algorithm-management/
├── spec.md
├── plan.md
├── research.md
├── data-model.md
├── quickstart.md
├── contracts/
│   ├── algorithm-management-api.md
│   └── worker-orchestrator-contract.md
└── tasks.md
```

### Source Code (repository root)

```text
backend/
├── app/
│   ├── api/
│   │   ├── deps/
│   │   ├── routes/
│   │   └── schemas/
│   ├── core/
│   │   ├── config/
│   │   ├── logging/
│   │   └── security/
│   ├── db/
│   │   ├── models/
│   │   ├── repositories/
│   │   └── migrations/
│   ├── domain/
│   │   ├── algorithms/
│   │   ├── risk/
│   │   ├── market_data/
│   │   └── execution/
│   ├── services/
│   │   ├── algorithm_orchestrator/
│   │   ├── market_data_adapter/
│   │   ├── run_lifecycle/
│   │   └── recovery/
│   └── workers/
│       └── algorithm_worker/
├── tests/
│   ├── api/
│   ├── integration/
│   └── unit/
├── alembic.ini
└── pyproject.toml

frontend/
├── src/
│   ├── features/
│   │   ├── algorithms/
│   │   ├── runs/
│   │   └── monitoring/
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

**Structure Decision**: A modular monolith backend with dedicated worker process and React frontend is the correct structure because the feature needs strong data isolation and lifecycle orchestration without premature decomposition. The worker remains separate from the HTTP layer so it can safely claim runs, poll or subscribe to market data, and manage long-lived execution loops without blocking API requests.

## Complexity Tracking

No constitution violations or exceptional complexity exceptions are expected for this feature. The design intentionally keeps the architecture cohesive and avoids unnecessary microservice boundaries until the need for isolated scaling or failure domains is demonstrated.
