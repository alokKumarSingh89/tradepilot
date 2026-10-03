# Research: Multi-Algorithm Management and Execution Lifecycle

## Decision 1: Use a modular monolith with a dedicated algorithm worker process

**Decision**: Keep backend logic in a modular monolith, but separate the long-running execution loop into a worker process that is independently scheduled and coordinated via the database.

**Rationale**: The feature requires run isolation, concurrency, recovery, and lifecycle events while the product statement explicitly says not to introduce unnecessary microservices. A worker process gives operational isolation for algorithm execution without creating a distributed system or duplicating persistence concerns across services.

**Alternatives considered**:

- Full microservice per algorithm: rejected because it adds deployment complexity, cross-service consistency issues, and unnecessary operational burden for an MVP.
- Inline execution inside request handlers: rejected because it violates the requirement that the management API must not execute long-running trading loops inside HTTP handlers.

## Decision 2: Persist run state and version history in PostgreSQL

**Decision**: Create durable tables for algorithm configurations, version history, run records, state snapshots, orders, positions, and lifecycle events. All run coordination and recovery decisions will be resolved from the database state rather than ephemeral memory.

**Rationale**: Restart recovery, duplicate prevention, and idempotent command handling require a stable source of truth. PostgreSQL is already required by the project, and SQLAlchemy 2 with Alembic fits the platform constraints while preserving a clean persistence model.

**Alternatives considered**:

- In-memory run registry: rejected because it cannot survive restart and cannot provide reliable duplicate prevention.
- Event-only persistence: rejected because it lacks the auditability and recovery semantics required for trade-state and configuration history.

## Decision 3: Treat runs as isolated state machines

**Decision**: Each run gets a unique identity, isolated strategy state, risk state, P&L ledger, and lifecycle record. Shared state exists only at the market-data adapter and orchestration level, not in the algorithm runtime itself.

**Rationale**: The Feature 002 requirements explicitly require independent state and no shared execution state across runs. This is also essential for user monitoring and safe restart/recovery logic.

**Alternatives considered**:

- Shared mutable runtime object per configuration: rejected because it introduces cross-run contamination and race conditions.
- Single global risk state: rejected because it would collapse multiple independent algorithm instances into one operational state.

## Decision 4: Use event-driven risk evaluation independent from strategy cadence

**Decision**: Strategy evaluation and risk evaluation are separate loops. Strategy logic reacts to market data or decision events, while risk evaluation is triggered by price, position, and lifecycle updates and all risk thresholds remain run-scoped.

**Rationale**: The constitution and PRD both require event-driven risk evaluation and independent monitoring. This also reduces the risk of stale data or timing issues causing inconsistent risk checks.

**Alternatives considered**:

- Candle-driven risk checks only: rejected because it fails the requirement for event-driven risk monitoring.
- Risk checks embedded in the strategy loop: rejected because it couples risk behavior to strategy timing and harms isolation.

## Decision 5: Use paper-trading default and explicit recovery prompts on restart

**Decision**: All new runs default to paper trading. When the application restarts, the system restores the last known run state and requires an explicit recovery choice to recover, restart, or discard the run rather than silently resuming or recreating execution.

**Rationale**: This matches the product requirement and the trading safety principle: the platform must avoid unintended activation and must preserve operational transparency under recovery scenarios.

**Alternatives considered**:

- Silent auto-resume: rejected because it creates duplicate or hidden execution.
- Automatic stop on restart: rejected because it discards state without confirmation and risks user surprise.

## Decision 6: Market-data adapter is shared, but subscriptions are per-run

**Decision**: A shared market-data adapter provides normalized data and subscription routing, while each run subscribes only to the instruments and events relevant to that run. Run-specific state is kept within the algorithm runtime, while data distribution remains centralized.

**Rationale**: This reduces duplication of market adapters and avoids operational drift, while still enabling per-run resilience and safe filtering of stale or partial data.

**Alternatives considered**:

- One adapter per run: rejected because it duplicates connection and data-handling behavior and can increase failure surface area.
- Shared mutable market snapshot with no filtering: rejected because it would allow risk and execution events to leak across runs.

## Decision 7: Use idempotent commands with database concurrency protections

**Decision**: Start, stop, and exit requests are idempotent at the command layer, and concurrency is protected with database constraints or compare-and-swap patterns around active-run state transitions.

**Rationale**: Preventing duplicate execution requests and race conditions is central to the feature. The system must guard against rapid repeated clicks and process-level duplication after app restarts.

**Alternatives considered**:

- Client-side prevention only: rejected because it does not protect the backend against duplicate requests or workers reclaiming the same run.
- No command-level idempotency: rejected because it creates double execution risk and makes recovery unpredictable.

## Decision 8: Define explicit lifecycle and recovery states

**Decision**: The lifecycle state machine must include at least `created`, `ready`, `running`, `stopping`, `stopped`, `failed`, and `recovered` with an explicit recovery decision path for interrupted runs.

**Rationale**: The requirement set and approved clarifications specifically call for recovery choices after restart and state transitions that preserve user visibility. A minimal set is sufficient for MVP, but the system must not collapse recovery, stop, and failed states into a single ambiguous status.

**Alternatives considered**:

- Single status field with a generic `active` or `inactive` value: rejected because it hides recovery and stop semantics and violates observability requirements.
- Implicit auto-resume after restart: rejected because it creates duplicate execution risk and silent state changes.

## Decision 9: Add idempotent command records and DB concurrency guardrails

**Decision**: The system will persist a command or intent record for each start, stop, and exit request using an idempotency key and a database uniqueness check on the active run state transition.

**Rationale**: The feature explicitly requires duplicate request prevention and database-level concurrency protection where appropriate, and the safety principle requires that repeated requests do not create a second execution.

**Alternatives considered**:

- Client-side deduplication only: rejected because it does not protect the backend or multiple workers.
- No command ledger: rejected because the system cannot safely reconcile retries or user double-clicks.

## Decision 10: Handle stale data and recovery failure as first-class run states

**Decision**: A run that loses fresh market data or experiences a worker interruption is marked as `degraded`, `recovering`, or `failed` instead of being silently treated as healthy. A restart must surface the latest state and prompt the user for recovery choices.

**Rationale**: The PRD and Constitution explicitly require stale data and restart handling to be visible and safe. This is a direct operational risk control for trading workflows.

**Alternatives considered**:

- Continuing on stale data without a warning: rejected because it creates hidden risk and violates transparency requirements.
- Automatically discarding the run state on restart: rejected because it erases history and makes recovery impossible for users.

## Open decisions retained for implementation review

- The exact lifecycle state enum should be finalized in backlog planning, but the MVP must at minimum support `running`, `stopping`, `recovering`, and `failed` semantics.
- The UI may need a list view and detail panel, but the exact monitoring layout can be finalized during implementation design.
- The maximum number of concurrent algorithms per user or system should be validated against operational assumptions and load tests before full release.
- Snapshot creation frequency should be formalized as a set of policy triggers: start, lifecycle transitions, risk threshold events, and explicit user confirmation actions.
