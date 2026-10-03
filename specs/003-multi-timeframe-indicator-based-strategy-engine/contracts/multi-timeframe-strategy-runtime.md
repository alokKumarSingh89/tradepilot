# Multi-Timeframe Strategy Runtime Contract

## Overview

This contract extends the Feature 002 orchestration and lifecycle model with strategy-specific evaluation for multi-timeframe indicator-based algorithms. It defines the integration points between the management API, algorithm worker, market-data pipeline, and the strategy engine.

## Shared interfaces

The following Feature 002 interfaces remain authoritative and must be extended rather than replaced:

- AlgorithmConfiguration
- AlgorithmRun
- RunSnapshot
- RiskState
- OrderRecord
- LifecycleEvent

Feature 003 adds the following runtime interfaces:

- StrategyRuntimeConfiguration
- TimeframeDefinition
- IndicatorDefinition
- IndicatorRule
- TradeLegRuntimeState
- SignalEvent
- SharedRiskBudget
- CandleBucket

## Authority and boundary contract

The Feature 002 lifecycle contract remains authoritative for `AlgorithmRun` transitions, duplicate-run prevention, restart reconciliation, and recovery authorization. Feature 003 may expose derived evaluation statuses for strategy analysis, but those statuses do not independently authorize pause, failure, restart, recovery, or activation of the parent run.

Feature 003 owns signal generation, indicator evaluation, higher-timeframe directional bias, and logical main/supporting trade strategy state. The shared risk engine remains authoritative for risk enforcement; the shared execution engine remains authoritative for order submission, idempotency, and order reconciliation. Strategy state must be reconciled against the authoritative execution and risk records rather than assuming that an emitted exit signal means a position has closed.

During recovery, Feature 003 may reconstruct derived strategy state only after Feature 002 has completed reconciliation and explicitly authorized resumption. No new strategy entries are permitted during recovery, and all restart/resume operations must respect duplicate-run prevention.

## Runtime responsibilities

The multi-timeframe engine is responsible for:

- loading the strategy runtime configuration from the configuration snapshot;
- validating indicator definitions and timeframe role assignments;
- normalizing market data into candle buckets per timeframe;
- evaluating the higher-timeframe bias, confirmation rules, and lower-timeframe entry/exit conditions;
- generating valid signals and suppressed-signal audit entries;
- managing independent main and supporting trade-leg state;
- surfacing decisions back to the Feature 002 risk engine and execution adapter.

## Contract boundaries

### 1. Strategy engine input contract

The worker passes the following to the strategy engine:

- run_id
- configuration_snapshot
- strategy_runtime_config
- latest finalized candle buckets
- latest market-data event state
- prior trade-leg and signal state

The strategy engine must not mutate the source snapshot; it returns derived decisions only.

### 2. Signal output contract

The strategy engine emits a structured result for each evaluation cycle:

- run_id
- trade_leg_id
- timeframe_definition_id
- signal_type
- direction
- created_at
- confidence or strength, when applicable
- suppressed flag and reason for suppressions
- source candle timestamps

### 3. Trade-leg state update contract

When a valid signal is accepted by the runtime, the strategy engine updates trade-leg state and emits the resulting signal or trade request. The execution layer then validates the state against Feature 002 risk and order policies before any order instruction is issued.

### 4. Reversal and risk interaction contract

When a higher-timeframe bias reverses, the strategy engine emits a reversal event that the worker forwards to the risk engine. The risk engine decides whether to close, pause, or maintain the current state based on the configured reversal policy and user-confirmed override.

## Persistence contract

The strategy engine must write:

- signal events
- candle bucket state
- suppressed-signal entries
- trade-leg runtime state updates
- snapshot records when the runtime state materially changes

These writes must be durable and aligned with the Feature 002 lifecycle transaction model.

## Failure and recovery contract

When late data, stale market data, or an application restart occurs:

- the strategy engine uses the last finalized candle state by default;
- it does not silently reprocess finalized decisions;
- it emits a degraded or recovering runtime status when state cannot be reconstructed safely;
- it records recovery actions in the existing lifecycle and snapshot history.

## API extension points

The following endpoints are proposed as additive API extensions to the approved Feature 002 management API:

- GET /api/v1/runs/{runId}/strategy-runtime
- GET /api/v1/runs/{runId}/signals
- GET /api/v1/runs/{runId}/trade-legs
- POST /api/v1/runs/{runId}/strategy-runtime/snapshot
- POST /api/v1/runs/{runId}/strategy-runtime/reconcile

These endpoints do not replace the Feature 002 lifecycle endpoints. They extend the same run model with strategy-specific detail and auditability.

## Compatibility guarantees

- Core lifecycle, recovery, and worker contracts remain unchanged for Feature 002.
- Feature 003 adds domain-specific state, not a separate execution orchestration model.
- The multi-timeframe strategy engine is expected to remain paper-trading only in the MVP.
