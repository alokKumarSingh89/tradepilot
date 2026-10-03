# Data Model: Multi-Timeframe Indicator-Based Strategy Engine

## Core Purpose

The model extends the approved Feature 002 algorithm and run structure rather than replacing it. The new data model adds strategy-specific runtime state for multi-timeframe rule evaluation, indicator validation, trade-leg independence, and signal auditability while staying compatible with the existing configuration, run, risk, lifecycle, and order tables.

## Ownership and lifecycle precedence

Feature 002 remains the sole authority for `AlgorithmRun` lifecycle transitions, duplicate-run prevention, run reconciliation, and recovery authorization. Feature 003 may persist derived strategy evaluation state, but that state is subordinate to the authoritative Feature 002 lifecycle and must never independently transition the parent run into an active state.

The shared risk engine remains authoritative for risk enforcement. Feature 003 may define risk policies, thresholds, and request context, but it must not create a competing risk execution mechanism. The shared execution engine remains authoritative for orders, fills, idempotency, and reconciliation. Strategic trade-leg state must reconcile against authoritative execution records rather than assuming that a signal automatically implies an order or a position closure.

## Shared Entities Extended from Feature 002

### 1. AlgorithmConfiguration

Represents the persisted user strategy configuration and remains the parent object for all run instances.

Existing fields retained:

- id
- user_id
- name
- algorithm_type
- execution_mode
- config_payload
- version metadata
- current_version_id

Proposed extension fields:

- strategy_family: enum (multi_timeframe_indicator, legacy_algorithm, future_extensions)
- runtime_engine: enum (feature_002_core, multi_timeframe_indicator_engine)
- config_schema_version: integer
- strategy_runtime_payload: JSONB

Validation rules:

- strategy_family must be set when the configuration uses the multi-timeframe runtime.
- runtime_engine must match the strategy_family and cannot be changed silently on a live run.
- config_payload and strategy_runtime_payload must both be present when the configuration is saved.

### 2. AlgorithmRun

Represents a single execution instance created from a configuration version.

Existing fields retained:

- id
- configuration_id
- configuration_version_id
- user_id
- status
- execution_mode
- start/stop timestamps
- recovery state
- claim_token

Proposed extension fields:

- strategy_runtime_state: enum (idle, evaluating, awaiting_confirmation, paused, degraded, failed)
- primary_bias: enum (bullish, bearish, neutral)
- active_trade_leg_count: integer
- last_signal_timestamp: timestamp, nullable
- last_snapshot_version: integer

Validation rules:

- strategy_runtime_state is a derived strategy-evaluation state only and cannot silently overwrite the authoritative core lifecycle state controlled by Feature 002.
- primary_bias remains a derived value from the latest valid higher-timeframe condition.
- The run must retain the original configuration snapshot used to start the strategy engine.
- Feature 003 strategy state must not independently authorize pause, failure, restart, recovery, or activation of the parent `AlgorithmRun`.

### 3. RunSnapshot

Represents the immutable record of the strategy and runtime state for an active run.

Existing fields retained:

- id
- run_id
- snapshot_type
- captured_at
- state_payload
- risk_state_payload
- order_state_payload

Proposed extension fields:

- strategy_runtime_payload: JSONB
- trade_leg_state_payload: JSONB
- suppressed_signal_payload: JSONB

Validation rules:

- snapshot_type must include strategy evaluation and trade-leg changes, not only risk events.
- snapshot payloads must be append-only and immutable after creation.

### 4. RiskState

Retains the existing event-driven risk model from Feature 002.

Proposed extension fields:

- shared_risk_budget: decimal, nullable
- used_shared_risk: decimal
- trade_leg_risk_summary: JSONB

Validation rules:

- used_shared_risk must never exceed the configured shared_risk_budget when the shared budget is enabled.
- Each trade leg keeps its own independent risk state and contributes to the summary only through the strategy runtime context.
- The shared risk engine remains the authoritative enforcement layer; Feature 003 defines policy context and requested actions but must not create a competing risk execution mechanism.

## New Feature 003 Entities

### 5. StrategyRuntimeConfiguration

Represents the multi-timeframe runtime definition associated with a configuration version.

Fields:

- id: UUID
- configuration_version_id: UUID
- strategy_name: string
- primary_timeframe_id: UUID
- confirmation_timeframe_id: UUID, nullable
- entry_exit_timeframe_ids: UUID[] or mapping
- default_direction_bias: enum (bullish, bearish, neutral)
- main_trade_reversal_action: enum (CLOSE, HOLD)
- supporting_trade_reversal_action: enum (CLOSE, HOLD)
- reversal_entry_policy: enum (BLOCK_NEW_ENTRIES, ALLOW_WHEN_NEW_BIAS_CONFIRMED)
- auto_reverse_entry: boolean = false
- shared_risk_budget_enabled: boolean
- shared_risk_budget_value: decimal, nullable
- paper_trading_only: boolean
- created_at: timestamp
- updated_at: timestamp

Validation rules:

- At least two timeframes must be configured.
- Each timeframe must have an assigned role and be unique within the runtime definition.
- If a confirmation timeframe is configured, it must be optional and not mandatory for valid strategy operation.
- `main_trade_reversal_action` and `supporting_trade_reversal_action` are mutually independent and may differ per strategy configuration.
- `reversal_entry_policy` is evaluated separately from `reversal_action` and does not imply a position exit.
- `auto_reverse_entry` must remain `false` for MVP and cannot be overridden by the strategy engine at runtime.

### 6. TradeLegDefinition

Represents a configured trade leg within the strategy runtime and holds the immutable reversal policy for that leg.

Fields:

- id: UUID
- strategy_runtime_config_id: UUID
- leg_name: string
- leg_type: enum (main, supporting)
- reversal_action: enum (CLOSE, HOLD)
- allowed_open_positions: integer, nullable
- created_at: timestamp
- updated_at: timestamp

Validation rules:

- `reversal_action` is immutable for the run once the strategy has started.
- Multiple open positions under the same trade-leg definition must share the same run-level reversal configuration.
- Supporting trade-leg policies may be configured independently per trade-leg definition and are not treated as a global default across all supporting legs.

### 7. TimeframeDefinition

Represents a valid timeframe role within the strategy runtime.

Fields:

- id: UUID
- strategy_runtime_config_id: UUID
- timeframe: enum (15m, 1h, 4h, 1d, custom)
- role: enum (primary_trend, confirmation, entry_exit)
- priority: integer
- enabled: boolean
- created_at: timestamp

Validation rules:

- role must be declared explicitly.
- timeframe values must be normalized to the project calendar and session boundaries.
- a custom timeframe requires a valid registry definition and session-alignment metadata.

### 8. IndicatorDefinition

Represents the indicator configuration for a specific timeframe and strategy runtime.

Fields:

- id: UUID
- timeframe_definition_id: UUID
- indicator_type: enum (macd, heikin_ashi)
- parameters: JSONB
- evaluation_mode: enum (completed_candle_only, live_candle_override)
- warmup_requirement: integer
- enabled: boolean
- created_at: timestamp

Validation rules:

- indicator_type must be one of the registered supported indicator types.
- parameter set must validate against the indicator registry schema.
- warmup requirements must be at least zero and must be satisfied before a signal can be produced.

### 9. IndicatorRule

Represents the rule definition that maps indicator output to a directional or threshold-based decision.

Fields:

- id: UUID
- indicator_definition_id: UUID
- rule_type: enum (crossover, slope_bend, histogram_change, candle_colour_change, reversal_detected, threshold_crossing)
- direction: enum (bullish, bearish, neutral, either)
- threshold_value: decimal, nullable
- comparison_operator: enum (gt, lt, gte, lte, eq, neq), nullable
- previous_candle_required: boolean
- enabled: boolean
- created_at: timestamp

Validation rules:

- A rule must correspond to a valid indicator_type.
- The threshold and comparison operator must be coherent when used together.
- Reversal detection must be explicitly configured and cannot assume a single hard-coded interpretation.

### 10. TradeLegRuntimeState

Represents the state of either the main trade or a supporting trade in a strategy run.

Fields:

- id: UUID
- run_id: UUID
- leg_type: enum (main, supporting)
- leg_index: integer
- direction_bias: enum (long, short, neutral)
- status: enum (idle, ready, open, partial_exit, paused, closed, rejected)
- entry_price: decimal, nullable
- exit_price: decimal, nullable
- quantity: decimal
- realized_pnl: decimal
- unrealized_pnl: decimal
- stop_loss: decimal, nullable
- take_profit: decimal, nullable
- risk_budget_reserved: decimal
- created_at: timestamp
- updated_at: timestamp

Validation rules:

- each leg must remain independent from the main trade while sharing the runtime context.
- supporting legs may have multiple open entries under the configured cap, but the cap cannot be exceeded.
- `reversal_action` and `reversal_entry_policy` are derived from the configured trade-leg definition; repeated reversal evaluation must be idempotent and must not create duplicate exit intents.
- `HOLD` retains the current position under the configured risk and exit rules and never disables required shared risk controls.

### 11. SignalEvent

Represents a candidate signal or a suppressed signal generated by the strategy engine.

Fields:

- id: UUID
- run_id: UUID
- trade_leg_id: UUID, nullable
- timeframe_definition_id: UUID
- signal_type: enum (entry, exit, reversal, suppression, warmup_wait, invalid_data)
- direction: enum (bullish, bearish, neutral)
- strength: decimal, nullable
- source_candle_open_time: timestamp
- source_candle_close_time: timestamp
- raw_payload: JSONB
- suppressed: boolean
- reason: text, nullable
- created_at: timestamp

Validation rules:

- suppressed events are still persisted and marked as audit records.
- signals must be tied to an actual timeframe and candle bucket.
- duplicate suppression operates only within the same trade leg, signal direction, and candle window.

### 12. SharedRiskBudget

Represents the optional budget available across multiple trade legs in a single run.

Fields:

- id: UUID
- run_id: UUID
- enabled: boolean
- total_budget: decimal
- used_budget: decimal
- allocation_policy: enum (global, trade_family, weighted)
- created_at: timestamp
- updated_at: timestamp

Validation rules:

- enabled must be false when the strategy does not allow shared risk.
- used_budget cannot exceed total_budget.
- each trade leg contributes to used_budget only when the strategy defines a shared budget.

### 13. CandleBucket

Represents the normalized time bucket used for strategy evaluation.

Fields:

- id: UUID
- run_id: UUID
- timeframe_definition_id: UUID
- candle_start: timestamp
- candle_end: timestamp
- is_finalized: boolean
- payload: JSONB
- created_at: timestamp

Validation rules:

- each candle bucket must include the relevant timeframe and timeframe alignment metadata.
- the engine must not evaluate incomplete candles by default.
- delayed or out-of-order updates must not silently mutate a finalized bucket unless the reprocessing mode is explicitly enabled.

## Relationships

- One AlgorithmConfiguration has many configuration versions.
- One configuration version is associated with one StrategyRuntimeConfiguration.
- One strategy runtime configuration has many TimeframeDefinition rows.
- One timeframe definition has many IndicatorDefinition rows and many SignalEvent rows.
- One run has many TradeLegRuntimeState rows.
- One run has many SignalEvent rows and many CandleBucket rows.
- One run has zero or one SharedRiskBudget and many TradeLegRuntimeState contributions.
- One run has many RunSnapshot rows that capture both core lifecycle state and strategy runtime state.

## State transitions

The model must support the following runtime transitions:

- configuration created -> saved -> runtime enabled
- run created -> ready -> running -> paused -> recovered -> stopped
- strategy engine evaluating -> signal generated -> trade leg updated -> risk event emitted
- reversal detected -> position disposition and entry admission resolved separately -> duplicate exit intents suppressed -> run snapshot persisted
- late data arrives -> ignored or reprocessed only under explicit override

## Reversal policy schema

The strategy runtime must persist the following reversal configuration in the immutable run snapshot:

- `main_trade.reversal_action`: `CLOSE` | `HOLD`
- `supporting_trade.reversal_action`: `CLOSE` | `HOLD`
- `reversal_entry_policy`: `BLOCK_NEW_ENTRIES` | `ALLOW_WHEN_NEW_BIAS_CONFIRMED`
- `auto_reverse_entry`: `false`

These fields are treated as strategy configuration and are immutable once the run is active. A confirmed reversal must be processed idempotently, and all derived execution decisions remain subject to the authoritative Feature 002 run, risk, and execution record.

## Compatibility with Feature 002

The following Feature 002 tables remain authoritative for lifecycle, recovery, persistence, and execution workflow:

- AlgorithmConfiguration
- AlgorithmRun
- RunSnapshot
- RiskState
- OrderRecord
- LifecycleEvent

The Feature 003 tables extend these with strategy-engine-specific state, not a separate execution model. This keeps the architecture additive and preserves the approved orchestration model.
