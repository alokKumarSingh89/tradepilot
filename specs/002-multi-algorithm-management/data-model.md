# Data Model: Multi-Algorithm Management and Execution Lifecycle

## Core Purpose

The data model separates configuration metadata from run execution state, so users can save and version algorithm definitions without mutating the behavior of active runs. The implementation keeps a durable record for restart recovery, lifecycle state transitions, risk evaluation, and run auditing.

## Entities

### 1. User

Represents the account owner whose algorithm configurations and run records are managed.

- id: UUID
- email or username: string
- created_at: timestamp
- status: active/inactive

Validation rules:
- A user record must exist before creating a configuration or run.
- User-scoped uniqueness is enforced for configuration names.

### 2. AlgorithmConfiguration

Represents the user-defined algorithm definition and its version history.

- id: UUID
- user_id: UUID
- name: string
- algorithm_type: enum (short_straddle, short_strangle, directional, positional, future extensible)
- config_payload: JSONB
- execution_mode: enum (paper, live)
- is_active: boolean
- created_at: timestamp
- updated_at: timestamp
- current_version_id: UUID

Validation rules:
- Name must be unique per user within the active configuration namespace.
- Required strategy fields must be present when the configuration is saved.
- Live execution mode is prohibited in this MVP; the system defaults to paper.

### 3. AlgorithmConfigurationVersion

Represents a saved revision of an algorithm definition.

- id: UUID
- configuration_id: UUID
- version_number: integer
- snapshot_payload: JSONB
- created_by: UUID or user reference
- created_at: timestamp
- parent_version_id: UUID, nullable
- notes: text, nullable

Validation rules:
- version_number increases monotonically for each configuration.
- Each version is immutable once created.
- A new version is created on save, but the active run remains coupled to its original runtime version.

### 4. AlgorithmRun

Represents a single algorithm execution instance created from a configuration version.

- id: UUID
- configuration_id: UUID
- configuration_version_id: UUID
- user_id: UUID
- run_name: string or generated token
- status: enum (created, ready, running, stopping, stopped, failed, recovered, completed)
- execution_mode: enum (paper, live) but live is not supported in MVP
- start_requested_at: timestamp, nullable
- started_at: timestamp, nullable
- stopped_at: timestamp, nullable
- last_heartbeat_at: timestamp, nullable
- last_market_data_at: timestamp, nullable
- active_position_count: integer
- claim_token: string, nullable
- stop_decision: enum (none, pending_user_confirmation, keep_position, close_position)
- recovery_state: enum (none, pending, resolved, discarded)
- recovery_decision: enum (none, recover, restart, discard)
- created_at: timestamp
- updated_at: timestamp

Validation rules:
- A run is unique to a single configuration version and can have its own status transitions.
- Duplicate start requests are rejected when the same logical configuration is already active for that user.
- A run may be flagged as recovered only after user confirmation or a safe reconciliation rule.
- If stale market data or a worker interruption is detected, the run enters a degraded or recovering state rather than silently continuing.

### 5. RunSnapshot

Captures a point-in-time view of a run for auditing and recovery.

- id: UUID
- run_id: UUID
- snapshot_type: enum (start, status_change, risk_threshold, manual, recovery)
- captured_at: timestamp
- state_payload: JSONB
- risk_state_payload: JSONB
- order_state_payload: JSONB, nullable
- position_state_payload: JSONB, nullable

Validation rules:
- Record is append-only.
- Snapshot generation is triggered per run lifecycle transitions and risk threshold events.
- Snapshot payload must be sufficient to reconstruct run state after restart.

### 6. Position

Represents the active or historical position state for a run.

- id: UUID
- run_id: UUID
- symbol: string
- side: enum (call, put, long, short)
- quantity: decimal
- average_price: decimal
- unrealized_pnl: decimal
- status: enum (open, partial, closed, rejected)
- updated_at: timestamp
- created_at: timestamp

Validation rules:
- Position state is isolated to the run; no cross-run aggregation.
- A stop or exit action may not silently change state without an explicit decision or workflow.

### 7. OrderRecord

Represents every order or simulated order intent emitted by or for a run.

- id: UUID
- run_id: UUID
- position_id: UUID, nullable
- order_type: enum (entry, exit, stop_loss, take_profit, manual)
- order_side: enum (buy, sell)
- status: enum (queued, accepted, partial_fill, filled, rejected, cancelled)
- quantity: decimal
- execution_mode: enum (paper, live)
- broker_reference: string, nullable
- idempotency_key: string
- parent_command_id: UUID, nullable
- created_at: timestamp
- updated_at: timestamp

Validation rules:
- All order events are immutable after execution and remain audit-visible.
- Duplicate command ids are rejected by idempotency keys when the same request is retried.
- Paper mode records order intents without live broker execution.

### 8. RiskState

Represents the latest risk evaluation for a run.

- id: UUID
- run_id: UUID
- combined_premium: decimal
- pnl: decimal
- stop_loss_threshold: decimal
- take_profit_threshold: decimal
- stale_market_data: boolean
- market_data_staleness_seconds: integer
- last_evaluated_at: timestamp
- last_price_received_at: timestamp, nullable
- status: enum (normal, warning, breached, recovering, degraded)

Validation rules:
- Risk state is updated only by risk evaluation logic for a single run.
- Stale market data is treated as a degraded or paused state, not as a silent proceed condition.
- We need a record of both the latest risk evaluation time and the last fresh price time so stale-data checks are traceable.

### 9. LifecycleEvent

Represents operational state changes or user actions for a run.

- id: UUID
- run_id: UUID
- event_type: enum (created, started, stopped, recovered, failed, snapshot_taken, position_changed, risk_breach, user_prompted)
- event_payload: JSONB
- created_at: timestamp

Validation rules:
- Must be appended in order and never overwritten.
- Used for auditability, UI timeline, and recovery reasoning.

## Relationships

- One user owns many algorithm configurations.
- One algorithm configuration has many versions.
- One configuration version may be used by many runs.
- One run owns many snapshots, positions, risk states, orders, and lifecycle events.
- A run belongs to one user and one configuration.
- The system preserves version history independent of run execution history.

## State transitions

The design should implement the following state model at minimum:

- created -> ready -> running -> stopped
- running -> failed -> recovered
- running -> completed
- any active state may produce snapshot and lifecycle_event records

The exact status enum should be finalized during task planning, but the data model must support user-visible lifecycle transitions, recovery prompts, and run-level status isolation.

## Design notes for resilience and concurrency

- The orchestration layer claims runs using a database-level lock or row-state compare-and-swap to prevent double start claims.
- Status changes and snapshot writes are transactional so the run cannot be partially mutated.
- Recovery is modeled as a distinct state rather than an implicit restart to avoid silent execution duplication.
- A run can be stopped only after finalizing any open-position decision in a user confirmation workflow.
- Run commands and order intents require idempotency keys so retry requests do not create duplicate execution or duplicate exit triggers.
- When stale data or worker interruption occurs, the run enters a degraded or recovering state and blocks silent continuation until user-visible reconciliation succeeds.
