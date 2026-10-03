# Research: Multi-Timeframe Indicator-Based Strategy Engine

## Decision 1: Extend the Feature 002 algorithm orchestration instead of introducing a competing execution system

**Decision**: The multi-timeframe engine will be implemented as a runtime extension of the existing algorithm worker and lifecycle model. It will reuse the Feature 002 run orchestration, configuration versioning, market-data distribution, idempotent execution processing, run snapshots, and event-driven risk workflow, while adding a strategy-specific runtime layer for timeframe evaluation, indicators, and trade-leg state.

**Rationale**: The Product Requirements and the Feature 002 architecture both require clear separation of responsibilities, independent state, and lifecycle safety. Maintaining a single orchestration system reduces cross-feature drift and minimizes duplicate governance, persistence, and recovery logic. Feature 003 does not own the parent `AlgorithmRun` lifecycle; it owns the derived strategy evaluation layer and must reconcile against the authoritative Feature 002 lifecycle, risk, and execution records.

**Alternatives considered**:

- Separate worker for every strategy family: rejected because it would duplicate run lifecycle, risk monitoring, and persistence logic and would compete with the approved Feature 002 architecture.
- Embedded multi-timeframe logic inside the existing straddle engine: rejected because it would couple unrelated strategy families and prevent a clean extension path for rule-based indicator strategies.

## Decision 2: Use immutable run snapshots as the runtime source of truth

**Decision**: Every run acquires a configuration snapshot at start and again whenever material strategy or risk parameters change. The runtime engine reads the snapshot, then persists decision, trade, and risk records against it to ensure reproducible execution and auditability.

**Rationale**: The spec requires immutable configuration snapshots and explicit recovery behavior. This reduces ambiguity during restart recovery and prevents silent mutation of an already-running strategy.

**Alternatives considered**:

- Live config fetch on every evaluation cycle: rejected because it would permit changes to affect execution mid-run and violate the requirement for immutable behavior.
- Snapshot only at start: rejected because the spec requires snapshots when material strategy and risk state changes occur.

## Decision 3: Standardize a timeframe registry with explicit roles and extensibility

**Decision**: Define a reusable timeframe registry with supported types: 15M, 1H, and 4H at MVP, plus an extensible contract for additional intervals. Every timeframe is bound to a role: primary trend, confirmation, or entry/exit.

**Rationale**: This supports strict role-based evaluation without hard-coding a single strategy shape. It also keeps new timeframes additive and avoids redesigning the engine for every interval.

**Alternatives considered**:

- Hard-coded assumptions for only 4H and 15M: rejected because it excludes optional confirmation and future extensibility.
- Dynamic free-form timeframe strings without validation: rejected because it creates drift, inconsistent timestamps, and schedule misalignment.

## Decision 4: Use a registry-based indicator system with deterministic validation

**Decision**: Indicators are registered under a common interface and support explicit validation rules, warm-up periods, and normalized output for consistent evaluation across timeframes. Initial supported indicators are MACD and Heikin Ashi, with the registry extensible to EMA/RSI/Supertrend later without redesigning the engine.

**Rationale**: The spec requires configurable indicators, parameter validation, and deterministic logic with explicit warm-up and missing-candle handling. A registry-based approach makes validation and reuse consistent across all strategy definitions.

**Alternatives considered**:

- Inline indicator implementations per timeframe: rejected because it duplicates logic and makes future indicators harder to add safely.
- Single global indicator definition without per-timeframe registration: rejected because it omits the requirement for configuration validation and separate timeframe behavior.

## Decision 5: Separate rule evaluation from position execution and risk execution

**Decision**: Three runtime layers will coexist: market-candle ingestion, indicator-and-rule evaluation, and runtime trade-leg management. Risk evaluation remains at the Feature 002 risk layer, while the strategy engine emits candidate signals and state transitions that are interpreted by the risk and execution workflow.

**Rationale**: The spec requires event-driven risk monitoring, independent positions for main and supporting trades, and a clean separation between decision generation and execution management. This matches the constitution’s emphasis on separation of concerns.

**Alternatives considered**:

- Embedding risk checks directly in the indicator engine: rejected because it creates hidden coupling and makes restart recovery more difficult.
- Treating all trade legs as a single position: rejected because it violates the core product requirement of independent state.

## Decision 6: Use one shared execution envelope with independent trade legs

**Decision**: Each strategy run owns a single strategy context, but each trade leg (main and supporting) is an independent runtime object with its own positions, P&L, state transitions, and risk checks. A user-defined shared risk budget is applied only as optional cross-leg enforcement.

**Rationale**: This preserves the mandatory Feature 002 architecture while satisfying the requirement that main and supporting trades can coexist independently within a single algorithm run.

**Alternatives considered**:

- Single monolithic position record: rejected because it collapses trade-leg independence and prevents separate execution tracking.
- Separate algorithm runs for every supporting trade: rejected because it fractures one strategy into multiple independent runs and breaks the requirement for a shared strategy context.

## Decision 7: Default to completed-candle evaluation with explicit overrides

**Decision**: The default safe behavior is to evaluate indicators only on finalized candles. Live or partial-candle evaluation remains an explicitly enabled override for specialized strategies, but it is not the default for the MVP.

**Rationale**: The PRD and constitution require safety and deterministic behavior. Completed-candle evaluation avoids look-ahead and partial-data hazards while making late-data handling explicit and auditable.

**Alternatives considered**:

- Evaluate every incoming tick as if it were a candle: rejected because it introduces look-ahead and incomplete-bar ambiguity.
- Use both live and final candles without configuration: rejected because it creates inconsistent strategy behavior and weak auditability.

## Decision 8: Duplicate suppression is defined by trade leg, direction, and candle window

**Decision**: Duplicate signals are suppressed when they share the same strategy run, trade leg, signal direction, and candle bucket. A signal from a different trade leg or opposite direction remains valid and separate.

**Rationale**: The clarified requirement states that duplicate suppression should operate at the trade-leg and decision-window level rather than based on broad cross-strategy deduplication. This preserves valid repeated entries across independent supporting legs while preventing repeat submission of the same decision.

**Alternatives considered**:

- Suppress all signals within a shared candle across the entire strategy: rejected because it blocks distinct trade legs and opposite-direction decisions from being evaluated independently.
- Suppress only exact duplicate payloads across all fields: rejected because it is too strict and prevents legitimate strategy variation within the same candle window.

## Decision 9: Late or out-of-order data is ignored unless it changes finalized state

**Decision**: A late or out-of-order market update that does not materially change a previously finalized candle is ignored. The strategy engine preserves the existing state and does not reprocess finalized decisions unless the system is in an explicit reprocessing mode.

**Rationale**: This meets the clarified requirement for safe default behavior while maintaining the integrity of finalized candle-based evaluation and avoiding silent replays of historical decisions.

**Alternatives considered**:

- Recompute history on every late update: rejected because it creates re-entrancy and audit complexity.
- Pause all execution when late data arrives: rejected because it is too conservative for an operational readiness product.

## Decision 10: Feature 002 compatibility changes are additive and explicit

**Decision**: The build will not rewrite Feature 002. Instead, it adds a strategy engine mode to the existing algorithm configuration and worker contracts. Proposed compatibility changes are documented as extension points rather than replacements. Feature 003 may reconstruct derived strategy state during recovery, but it may not independently authorize new entries, restart, recovery, or activation; those actions remain controlled by Feature 002 after duplicate-run prevention and reconciliation pass.

**Rationale**: The user requirement explicitly says not to silently rewrite Feature 002 and to preserve the existing feature directories and filenames. This approach maintains backward compatibility while making the new strategy runtime available in a controlled way and keeps lifecycle authority with the run-management layer.

**Alternatives considered**:

- Replace the worker with a new algorithm family runtime: rejected because it would conflict with the existing architecture and create upgrade risk.
- Add a parallel orchestration system: rejected because it would duplicate lifecycle and risk responsibilities.

## Decision 11: Separate reversal disposition from entry-admission policy

**Decision**: The approved Feature 003 reversal model is split into two independent concerns: existing position disposition and new entry admission. Existing positions are governed by `main_trade.reversal_action` and `supporting_trade.reversal_action`, which support only `CLOSE` or `HOLD`. New entries are governed by `reversal_entry_policy`, which supports only `BLOCK_NEW_ENTRIES` or `ALLOW_WHEN_NEW_BIAS_CONFIRMED`. The MVP keeps `auto_reverse_entry` fixed at `false`.

**Rationale**: The clarified business requirement explicitly separates the treatment of open positions from future entries and forbids opposite-direction automatic entries. This makes the strategy engine auditable, avoids contradictory defaults, and preserves the authoritative Feature 002 lifecycle and risk enforcement model.

**Alternatives considered**:

- Single reversal action covering both exit and entry decisions: rejected because it collapses two independent concerns and creates contradictory defaults such as pause/close semantics.
- `PAUSE_NEW_ENTRIES` as a position-exit action: rejected because the approved rule separates position disposition from entry admission and requires explicit new-entry gating rather than modeling a pause as a closure event.
- Auto-open reverse entries after reversal: rejected because the requirement explicitly forbids an opposite-direction entry from being created automatically.

## Open technical questions retained for implementation design

- Whether a strategy’s higher-timeframe rule should be expressed as a user-defined rule object or a reusable trend classifier registry.
- Whether the 15M entry/exit engine should be represented as a single supporting-trade family or a standalone runtime component that can be mapped to multiple trade legs.
- Whether market-data session boundaries should be normalized per exchange or by a project-defined session policy.
- The precise storage format for strategy configuration history and signal audit events when the same rule is re-used across multiple runs.

## Proposed Feature 002 compatibility changes

The following changes are proposed to extend the approved Feature 002 model without altering its core architecture:

1. Extend AlgorithmConfiguration with a runtime strategy family and engine payload for multi-timeframe settings.
2. Extend AlgorithmRun with a strategy_runtime_state field that is distinct from generic lifecycle state.
3. Add a strategy runtime adapter to the worker contract so the worker can dispatch market data to the appropriate engine implementations.
4. Extend the run snapshot payload to include multi-timeframe runtime context, trade-leg state, and suppressed-signal history.
5. Add a runtime event type for signal evaluation, suppression, and supporting-trade state changes, alongside existing lifecycle and risk events.
6. Add a UI strategy details panel that reuses the existing run lifecycle model while exposing multi-timeframe decision state and audit history.

These are documented as compatibility changes, not as replacements for the approved lifecycle architecture.

## Cross-plan review: Feature 002 vs Feature 003

### 1) Architecture

- Classification: Compatible
- Finding: Both plans adopt the same modular monolith architecture with a dedicated worker runtime and a shared PostgreSQL-backed state model. Feature 003 explicitly extends Feature 002 rather than creating a separate orchestration layer.
- Why this is compatible: The approved Feature 002 design already separates HTTP/API concerns from execution runtime concerns, and Feature 003 fits inside that architecture by adding a strategy-specific runtime layer.

- Classification: Requires clarification
- Finding: Feature 002 names the worker as the owner of algorithm orchestration, while Feature 003 introduces strategy runtime modules under the backend domain. The boundary between "generic worker orchestration" and "strategy-engine implementation" should be stated explicitly so responsibilities do not blur.
- Clarification needed: Is the strategy engine a subcomponent of the worker runtime, or a separate domain service invoked by the worker with a clear adapter contract?

- Classification: Blocking conflict
- Finding: A design that creates a second executor/worker for multi-timeframe strategies would conflict with the Feature 002 design and duplicate the run-lifecycle management model.
- Why this is blocking: Feature 002 treats run ownership, lifecycle state, and recovery as the single source of truth. A second execution system would split ownership and create race conditions across restart and recovery.

### 2) Data models

- Classification: Compatible
- Finding: Feature 002's core entities already cover run lifecycle, configuration versioning, risk state, snapshots, and order records. Feature 003 adds strategy-only extensions without replacing those entities.
- Why this is compatible: The data model extension is additive and keeps the canonical lifecycle entities in Feature 002 as the parent model.

- Classification: Requires clarification
- Finding: Feature 003 adds several new tables for timeframe definitions, indicator definitions, trade-leg runtime state, signal history, and candle buckets, but it does not yet define whether they are always persisted for every run or only when the strategy family is multi-timeframe.
- Clarification needed: Should these tables be optional and nullable for non-multi-timeframe algorithms, or should the schema include them for all configurations with a null/default runtime payload?

- Classification: Requires a proposed Feature 002 change
- Finding: Feature 003 proposes adding strategy-runtime payload fields to AlgorithmConfiguration and AlgorithmRun, plus additional snapshot payloads and runtime-specific event types.
- Proposed change: Extend the Feature 002 tables with optional strategy runtime fields and dedicated event payloads rather than creating a parallel schema outside the approved model.

### 3) Shared interfaces

- Classification: Compatible
- Finding: The Feature 003 proposal reuses the same lifecycle and execution boundaries defined by Feature 002: configuration lifecycle, run lifecycle, snapshot persistence, execution events, and risk state.
- Why this is compatible: Shared interfaces already exist for start/stop/recover, order execution, and risk evaluation; Feature 003 extends them with strategy-specific contracts rather than inventing a new public interface set.

- Classification: Requires clarification
- Finding: Feature 003 defines a strategy-runtime contract, but the exact boundary between the generic worker contract and the strategy runtime adapter is still not fully specified.
- Clarification needed: Which side owns validation of data freshness, signal suppression, and risk-policy interpretation? The answer should assign ownership to one layer to avoid double-processing.

- Classification: Requires a proposed Feature 002 change
- Finding: Feature 003 adds a new API extension surface for strategy runtime details and trade-leg state. These are not separate features; they are additive extensions to the existing run and configuration contract.
- Proposed change: Extend the existing API and worker contracts with optional runtime detail endpoints and event types instead of creating a completely separate strategy API surface.

### 4) Runtime ownership

- Classification: Compatible
- Finding: Feature 002 assigns the worker responsibility for claiming runs, managing lifecycle state, and processing market-data events. Feature 003 assigns the runtime strategy engine responsibility for timeframe evaluation, indicator rule evaluation, and trade-leg state management, which is a natural split.
- Why this is compatible: One component owns the run lifecycle, the other owns strategy-specific decision generation; both operate under the same orchestration model.

- Classification: Requires clarification
- Finding: Feature 003 states that risk evaluation remains in Feature 002 while signals are generated in Feature 003, but it does not yet state exactly when a strategy signal transitions from "candidate signal" to "execution intent".
- Clarification needed: Which component is responsible for deciding whether a valid signal becomes an order request versus remaining a monitored strategy event?

- Classification: Requires a proposed Feature 002 change
- Finding: The runtime event model needs to expose strategy-engine signals and suppression events as part of the same lifecycle history used by Feature 002.
- Proposed change: Extend LifecycleEvent and RiskState to support signal, reversal, and suppression events while preserving the generic run lifecycle semantics.

### 5) Failure handling

- Classification: Compatible
- Finding: Both plans explicitly require stale market data handling, restart recovery, duplicate prevention, and safe user-confirmed stop decisions.
- Why this is compatible: The underlying failure model is already aligned in Feature 002, and Feature 003 only adds strategy-specific failure cases such as delayed candles, suppressed signals, and invalid indicator windows.

- Classification: Requires clarification
- Finding: Feature 003 proposes ignoring stale or out-of-order data by default, but Feature 002 does not yet specify whether a stale multi-timeframe signal should degrade the entire run or only the affected strategy runtime state.
- Clarification needed: Is a degraded strategy runtime considered a full run degradation, or should the run remain healthy while only one strategy context is paused?

- Classification: Requires a proposed Feature 002 change
- Finding: Feature 003 needs the generic run state model to include strategy-runtime degraded states and per-leg pause/closure outcomes, not just run-level status values.
- Proposed change: Add a strategy_runtime_state field and optional trade-leg state summary to RunSnapshot and AlgorithmRun so the run can be degraded at the proper granularity without losing the lifecycle state model.

### 6) Concurrent execution

- Classification: Compatible
- Finding: Feature 002 supports concurrent algorithm runs per user, while Feature 003 supports multiple trade legs inside the same run. These are not conflicting concurrency models; they operate at different scope levels.
- Why this is compatible: One run can host multiple independent supporting legs, while multiple runs may still exist concurrently for other algorithms or strategies.

- Classification: Requires clarification
- Finding: Feature 003 allows multiple supporting positions to remain open under a shared bias, but Feature 002's run-level concurrency model assumes independent risk state per run. This is compatible only if the shared risk budget is explicitly implemented as a run-level aggregate, not as a cross-run sharing mechanism.
- Clarification needed: Is the shared risk budget scoped only within a single run, or is there any intended cross-run sharing in the future?

- Classification: Blocking conflict
- Finding: If Feature 003 were to treat supporting positions as separate runs rather than trade legs within a single run, it would conflict with the approved concurrency model and invalidate the core Feature 002 assumption of isolated run state.
- Why this is blocking: It would split the same strategy into independent run contexts and break lifecycle recovery, snapshot consistency, shared risk semantics, and execution auditability.

### 7) Restart recovery

- Classification: Compatible
- Finding: Both plans require persistent snapshots, explicit recovery choices, and no silent auto-resume on restart.
- Why this is compatible: This is already a core governance requirement in Feature 002 and is directly reinforced by the Feature 003 snapshot and trade-leg state expectations.

- Classification: Requires clarification
- Finding: Feature 003 uses immutable snapshots for strategy configuration and state, but it does not yet define whether the recovery workflow should recover the full strategy runtime, the last valid trade-leg state, or both.
- Clarification needed: Should restart reconciliation restore the full runtime context plus the last valid trade-leg positions, or should the strategy engine rebuild from the saved candle state and rehydrate trade legs from available records?

- Classification: Requires a proposed Feature 002 change
- Finding: Recovery needs to restore not only the run-level state but also the strategy runtime context and suppressed-signal history.
- Proposed change: Extend the Feature 002 run snapshot payload and recovery workflow to accommodate strategy runtime state, trade-leg state, and signal history for multi-timeframe runs without changing the recovery contract semantics.

### Overall assessment

- Most of the overlap between Feature 002 and Feature 003 is compatible and additive.
- The main risks are not architectural disagreement but ownership ambiguity and schema extension details.
- The required compatibility change is narrow and well-bounded: extend the existing Feature 002 run, snapshot, and event model to include strategy runtime state and strategy-specific events, while keeping the run lifecycle, orchestration, and recovery model as the single source of truth.
- No blocking conflict exists as long as Feature 003 remains a strategy-specific runtime extension of the approved Feature 002 architecture and does not create independent execution ownership or a second run lifecycle model.

### Final classification summary

- Compatible: 7 findings
- Requires clarification: 8 findings
- Requires a proposed Feature 002 change: 6 findings
- Blocking conflict: 1 finding

These findings should be treated as review notes for the implementation phase and should be resolved before any task decomposition or implementation work begins.
