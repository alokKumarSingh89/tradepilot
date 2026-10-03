# Feature Specification: Multi-Timeframe Indicator-Based Strategy Engine

**Feature Branch**: `003-multi-timeframe-indicator-based-strategy-engine`

**Created**: 2026-10-03

**Status**: Draft

**Input**: User description: "Read PRD.md and the existing TradePilot Constitution. Create a NEW Feature 003: Multi-Timeframe Indicator-Based Strategy Engine. Extend TradePilot to support UI-configurable indicator-based trading algorithms. Requirements: 1. Users can configure two or more timeframes per algorithm. 2. Each timeframe has an explicit role: primary trend, optional confirmation, or entry/exit. 3. Support indicator configuration per timeframe, beginning with MACD and Heikin Ashi. 4. Indicator parameters must be configurable. 5. Allow conditions such as MACD crossover, MACD slope/bend, histogram changes, and HA candle colour changes. 6. Support a primary trade governed by higher-timeframe conditions. 7. Support independent lower-timeframe trades that follow the configured higher-timeframe directional bias. 8. Allow supporting trades to enter and exit multiple times while the higher-timeframe bias remains valid. 9. Define configurable behaviour when the primary trend reverses. 10. Maintain independent position state, risk rules, execution history, and P&L for main and supporting trades. 11. Support an optional shared risk budget across related trades. 12. Persist all strategy and indicator configuration in PostgreSQL. 13. Capture an immutable configuration snapshot for each run. 14. Evaluate indicators using timestamped, appropriately completed candles; define incomplete-candle behaviour explicitly. 15. Prevent duplicate signal processing and look-ahead bias. 16. Use paper trading by default. Preserve Features 001 and 002. Generate business requirements, user stories, acceptance criteria, and edge cases. Identify ambiguous trading rules for clarification. Do not generate application code or a technical plan."

## Architecture and ownership constraints

Feature 002 remains the sole authority for `AlgorithmRun` lifecycle transitions, duplicate-run prevention, restart reconciliation, and recovery authorization. Feature 003 does not introduce a competing lifecycle state machine or independently authorize pause, failure, restart, recovery, or activation of the parent `AlgorithmRun`.

Feature 003 owns the logical main/supporting trade relationships, strategy decisions, candle processing, indicator calculations, directional bias, signal generation, and strategy-specific runtime state. The shared execution and position records remain authoritative for actual orders, fills, quantities, and open exposure. Strategy state must be reconciled against authoritative execution records rather than assuming that an emitted exit signal means a position has closed.

During recovery or reconciliation, Feature 003 may reconstruct derived strategy state only as part of the Feature 002-coordinated recovery flow. No new strategy entries are permitted during recovery, existing exposure remains subject to the approved recovery and risk contract, and Feature 002 must complete reconciliation and explicitly authorize resumption before any strategy evaluation may continue. All restart/resume operations must respect duplicate-run prevention.

## Clarifications

### Session 2026-10-03

- Q: What should count as a valid 4H primary trend before the system treats the market as bullish, bearish, or neutral for the main trade? → A: The 4H trend rule is user-defined per strategy and is not hardcoded to a single default indicator mix.
- Q: Is the optional 1H confirmation timeframe required for every strategy, or may a valid strategy run with only a 4H primary trend and a 15M supporting entry/exit timeframe? → A: The confirmation timeframe is optional and user-selected per strategy; a strategy may run without 1H confirmation if the user chooses that configuration.
- Q: How should the 15M timeframe generate supporting entries and exits while the 4H primary bias stays valid? → A: The 15M timeframe acts as a repeatable entry/exit engine that can generate multiple supporting trades independently while the 4H bias remains valid, each with its own state and risk handling.
- Q: Can the main trade and supporting trades coexist at the same time, or must the strategy treat them as a single combined position? → A: The main trade and supporting trades can coexist independently, with each trade leg maintaining its own state, P&L, and risk while sharing the same overarching strategy context.
- Q: Can multiple supporting positions remain open at the same time under one valid higher-timeframe bias, or should the system limit the strategy to a single active supporting trade? → A: Multiple supporting positions may remain open simultaneously under a valid higher-timeframe bias, provided exposure and shared risk caps are respected.
- Q: How should the system treat existing open positions versus future entries when the higher-timeframe trend reverses? → A: The system separates those decisions into two independent concerns: existing position disposition uses `CLOSE` or `HOLD`, while new entry admission uses `BLOCK_NEW_ENTRIES` or `ALLOW_WHEN_NEW_BIAS_CONFIRMED`; the MVP keeps `auto_reverse_entry` disabled and does not auto-create opposite-direction entries.
- Q: What exact conditions should count as a MACD bend, a histogram colour change, and a Heikin Ashi reversal for the strategy engine? → A: The exact indicator logic is user-configurable per strategy; the default safe behavior is to require explicit threshold-based definitions rather than a single hardcoded interpretation.
- Q: Should the engine evaluate indicator conditions only on fully closed candles, or is it acceptable to use the current in-progress candle while it is still forming? → A: Indicator evaluation is user-configurable per strategy, with a default requirement to evaluate only completed candles; live-candle usage remains an explicitly enabled override rather than the default behavior.
- Q: How should independent risk and shared risk limits interact when a strategy has both a main trade and several supporting trades? → A: Each trade leg enforces its own risk limits, while the shared risk budget acts as an optional strategy-level cap on additional exposure when enabled.
- Q: If conflicting signals appear across timeframes, which timeframe wins and how should the system resolve the conflict? → A: The higher-timeframe bias governs directional decisions, and lower-timeframe signals must align with that bias; conflicting lower-timeframe signals are ignored unless the user explicitly configures a different precedence rule.
- Q: Should a lower-timeframe signal that conflicts with the higher-timeframe direction be ignored entirely, or should it be recorded as a suppressed signal for audit and review? → A: Conflicting lower-timeframe signals are ignored for execution but recorded as suppressed signals in the audit trail so the user can review why the signal was rejected.
- Q: How should repeated supporting entries be limited while the higher-timeframe bias remains valid? → A: A user-defined maximum number of supporting positions within the active bias window is allowed, with the limit configured per strategy; supporting entries stop once the cap is reached, even if additional signals remain valid.
- Q: How should the system treat repeated evaluations of the same confirmed reversal event? → A: Reversal processing must be idempotent; repeated evaluation of the same confirmed reversal must not produce duplicate exit intents, and actual closure must be validated against authoritative position and execution records rather than assuming the intent succeeded.
- Q: What should count as the “same candle” or “same decision window” for duplicate signal suppression? → A: Duplicate signal suppression is evaluated by the same timeframe bucket across the strategy, and the system considers repeated signals to be duplicates when they originate from the same trade leg, same direction, and same candle window; cross-leg or opposite-direction signals are treated as distinct decisions.
- Q: How should the system behave when data arrives late or out of order across different timeframes? → A: Late or out-of-order data is ignored when it does not materially change a previously finalized candle state; the system keeps the last valid candle state and does not reprocess prior finalized decisions unless the user explicitly enables a reprocessing mode.

## User Scenarios & Testing _(mandatory)_

### User Story 1 - Configure a multi-timeframe strategy (Priority: P1)

A user wants to build a strategy that blends multiple timeframes and indicator logic so a primary directional view can drive trade selection and execution.

**Why this priority**: Without a working multi-timeframe model, the feature does not provide the core trading value described in the PRD and Feature 002.

**Independent Test**: A user configures a strategy with at least two timeframes, assigns explicit roles to each, and confirms the configuration is saved and retrievable.

**Acceptance Scenarios**:

1. **Given** a user is creating a new indicator-based strategy, **When** they add a primary trend timeframe and at least one confirmation or entry/exit timeframe, **Then** the system accepts the configuration and stores each timeframe with its explicit role.
2. **Given** a user supplies missing or invalid indicator settings, **When** they save the strategy, **Then** the system rejects the submission and explains which settings need correction.

---

### User Story 2 - Define indicator conditions across timeframes (Priority: P1)

A user can configure the indicators and conditions that govern a trade, including MACD and Heikin Ashi behaviour, without needing a technical implementation.

**Why this priority**: The feature is fundamentally defined by indicator-based conditions and parameterization, so this is the primary functional driver.

**Independent Test**: A user configures MACD and Heikin Ashi parameters and verifies that the system stores the conditions for each timeframe.

**Acceptance Scenarios**:

1. **Given** a strategy uses MACD on a timeframe, **When** the user configures crossover, slope or bend conditions, histogram changes, or signal thresholds, **Then** the configuration is stored for later evaluation.
2. **Given** a strategy includes Heikin Ashi on a timeframe, **When** the user configures candle-colour rules, **Then** the system records those conditions and uses them in the configured timeframe role.

---

### User Story 3 - Run a primary trade and supporting lower-timeframe trades (Priority: P1)

A user expects the platform to support a primary trade governed by the higher-timeframe trend while allowing lower-timeframe trades to follow that bias and open or close multiple times while the bias remains valid.

**Why this priority**: This is the core product value of the feature: a directional primary trade backed by independent lower-timeframe execution within the same bias.

**Independent Test**: A user creates a strategy with a higher-timeframe primary trend and lower-timeframe entries and confirms that supporting trades are evaluated under the same directional bias and remain independent from the primary trade.

**Acceptance Scenarios**:

1. **Given** a multi-timeframe strategy has a valid higher-timeframe bias, **When** lower-timeframe conditions are satisfied, **Then** the system may open or close supporting trades consistent with that bias.
2. **Given** the primary higher-timeframe trend flips or reverses, **When** the reversal condition is met, **Then** the system resolves existing position disposition according to the configured `reversal_action` and separately enforces the configured `reversal_entry_policy`; it does not create an opposite-direction position or duplicate exit intent.
3. **Given** the higher-timeframe trend remains valid, **When** lower-timeframe conditions repeatedly trigger, **Then** the system allows supporting trades to enter and exit multiple times without reusing the same signals or state incorrectly.

---

### User Story 4 - Monitor independent trade state and risk (Priority: P1)

A user can review the execution history, positions, and risk status for the main trade and supporting trades independently.

**Why this priority**: Independent risk and execution state are required to prevent cross-trade contamination and protect capital.

**Independent Test**: A user runs a strategy with both main and supporting trades and confirms that each trade has its own state, risk rules, history, and P&L record.

**Acceptance Scenarios**:

1. **Given** a strategy includes both a primary trade and one or more supporting trades, **When** the run is active, **Then** each trade has separate position state, execution history, and P&L.
2. **Given** the system is configured with a shared risk budget, **When** risk limits are reached, **Then** the run enforces the configured shared budget without silently exceeding it.

---

### User Story 5 - Recover strategy configuration and run state safely (Priority: P2)

A user expects the strategy configuration, run snapshot, and execution history to persist across restarts without creating duplicate execution or hidden look-ahead behaviour.

**Why this priority**: Recovery and consistency are essential for operational trust, but they follow the core multi-timeframe configuration and execution value.

**Independent Test**: A user restarts the application after a configured run is saved and confirms that the previous configuration snapshot and run state are restored without duplicate signals or duplicate execution.

**Acceptance Scenarios**:

1. **Given** a multi-timeframe strategy has been saved and run before a restart, **When** the application restarts, **Then** the persisted configuration and immutable snapshot remain available without creating duplicate active decisions.
2. **Given** indicator data is incomplete for the current candle, **When** the system evaluates conditions, **Then** it uses only appropriately completed candles and does not use partial or look-ahead values.

---

### Edge Cases

- What happens when a user configures a timeframe without assigning a valid role?
- What happens when the primary trend reverses while supporting trades are still active?
- What happens when MACD or Heikin Ashi parameters are invalid, missing, or inconsistent across timeframes?
- What happens when a lower-timeframe signal repeats the same trade decision multiple times within the same candle or tick window?
- What happens when one candle is incomplete or an update arrives out of order?
- What happens when the higher-timeframe bias changes while supporting trades are already open?
- What happens when a strategy uses a shared risk budget that is exceeded by one supporting trade but not another?
- What happens when the user saves a new configuration version while a run is active?

## Requirements _(mandatory)_

### Functional Requirements

- **FR-001**: The system MUST allow a user to create a strategy that combines two or more timeframes in a single algorithm configuration.
- **FR-002**: The system MUST allow each timeframe in a strategy to be assigned an explicit role: primary trend, confirmation, or entry/exit.
- **FR-003**: The system MUST allow a user to configure the indicators used on each timeframe, starting with MACD and Heikin Ashi.
- **FR-004**: The system MUST allow users to configure indicator parameters for each timeframe and each indicator.
- **FR-005**: The system MUST support MACD-based conditions including crossover, slope or bend detection, histogram changes, and signal threshold comparisons.
- **FR-005A**: The system MUST allow MACD bend, histogram direction, and histogram colour-change conditions to be configured per strategy using explicit threshold-based rules rather than a single fixed interpretation.
- **FR-006**: The system MUST support Heikin Ashi candle-colour conditions and related directional colour transitions.
- **FR-006A**: The system MUST allow Heikin Ashi reversal and candle-colour conditions to be configured per strategy using explicit user-defined logic, including threshold and previous-candle comparison rules when applicable.
- **FR-006B**: The system MUST allow the confirmation timeframe to be optional and user-selected per strategy, so a valid configuration may omit 1H confirmation and proceed with a primary trend timeframe and lower-timeframe supporting entries only.
- **FR-006C**: The system MUST allow users to configure whether indicator evaluation uses only completed candles or explicitly enables live in-progress candle evaluation for a given strategy, with completed-candle evaluation as the default safe behavior.
- **FR-007**: The system MUST support a primary trade governed by the higher-timeframe directional bias of the strategy, with the exact 4H trend rule defined per strategy by the user rather than by a single fixed default rule.
- **FR-008**: The system MUST support independent lower-timeframe trades that follow the configured higher-timeframe direction while the bias remains valid.
- **FR-008A**: When timeframes disagree, the system MUST prioritize the configured higher-timeframe directional bias for strategy direction, and conflicting lower-timeframe signals must not create opposite-direction execution unless a user-defined override is explicitly enabled.
- **FR-008B**: The system MUST record suppressed lower-timeframe signals as audit events when they conflict with the governing higher-timeframe bias, while not executing those signals.
- **FR-009**: The system MUST allow supporting trades to enter and exit multiple times while the higher-timeframe directional bias remains valid, without being forced to collapse into a single trade state.
- **FR-009A**: The system MUST allow the 15M entry/exit timeframe to generate repeated supporting positions independently, with each supporting position tracking its own entry, exit, quantity, P&L, and risk state while the 4H bias remains valid.
- **FR-009B**: The system MUST allow the primary trade and supporting trades to coexist simultaneously as separate execution entities, each retaining independent position state, P&L, and risk controls while sharing a common strategy context.
- **FR-009C**: The system MUST allow multiple supporting positions to remain open simultaneously under a valid higher-timeframe bias, subject to strategy-defined exposure caps and shared risk settings.
- **FR-009D**: The system MUST enforce per-strategy exposure caps and optional shared risk budget limits before allowing additional supporting positions to open under the same directional bias.
- **FR-009E**: The system MUST support a user-defined maximum number of supporting positions within the active higher-timeframe bias window, and once that cap is reached, additional valid lower-timeframe signals must not open more supporting positions until the cap is reset by the strategy rules or position closure.
- **FR-010**: The system MUST support separate reversal handling for existing position disposition and new entry admission when the primary trend reverses.
- **FR-010A**: The system MUST support `main_trade.reversal_action` and `supporting_trade.reversal_action` with values `CLOSE` or `HOLD`. `CLOSE` generates an exit intent for the affected open positions; `HOLD` retains the positions under their configured exit and risk rules.
- **FR-010B**: The system MUST support `reversal_entry_policy` with values `BLOCK_NEW_ENTRIES` and `ALLOW_WHEN_NEW_BIAS_CONFIRMED`. `BLOCK_NEW_ENTRIES` prevents new entries after a confirmed reversal, while `ALLOW_WHEN_NEW_BIAS_CONFIRMED` permits future entries only after the configured directional and entry conditions are satisfied and Feature 002 lifecycle authorization and shared risk controls are still valid.
- **FR-010C**: The system MUST allow supporting-trade reversal policies to be configured per trade-leg definition, and multiple open positions belonging to the same leg MUST follow that leg's immutable run configuration without creating independent contradictory exit intents.
- **FR-010D**: The system MUST evaluate a confirmed reversal idempotently. Repeated evaluation of the same reversal event MUST NOT create duplicate exit intents, and `CLOSE` MUST NOT be treated as successful execution unless the authoritative execution and position records confirm the closure.
- **FR-010E**: The system MUST keep `auto_reverse_entry` disabled for the MVP (`false`), MUST NOT create an opposite-direction position automatically, and MUST never disable mandatory shared risk controls when a trade leg is held under `HOLD`.
- **FR-011**: The system MUST maintain independent position state, risk rules, execution history, and P&L for the primary trade and each supporting trade.
- **FR-011A**: The system MUST enforce independent risk limits for each trade leg while also enforcing an optional strategy-level shared risk budget that limits additional exposure across related positions when enabled.
- **FR-012**: The system MUST support an optional shared risk budget across related trades within the same strategy run.
- **FR-013**: The system MUST persist all strategy metadata, timeframe assignments, indicator definitions, parameter sets, and associated risk settings in PostgreSQL.
- **FR-014**: The system MUST capture an immutable configuration snapshot for each run at the time the run starts and whenever material strategy or risk state changes occur.
- **FR-015**: The system MUST evaluate indicators using timestamped, appropriately completed candles and MUST define explicit handling for incomplete candles, delayed data, and out-of-order updates.
- **FR-015A**: The system MUST ignore late or out-of-order data that does not materially alter a previously finalized candle state, while preserving the last valid completed-candle state and not reprocessing earlier finalized decisions unless an explicit reprocessing workflow is enabled.
- **FR-016**: The system MUST prevent duplicate signal processing, repeated execution of the same decision within the same candle, and look-ahead bias during strategy evaluation.
- **FR-016A**: Duplicate signal suppression MUST be based on the relevant timeframe bucket and trade-leg context, treating repeated signals from the same trade leg, same direction, and same candle window as duplicates while allowing distinct trade legs or opposite-direction decisions to be evaluated independently.
- **FR-017**: The system MUST default to paper trading for all multi-timeframe strategy runs and prevent live execution from being activated as part of this feature.
- **FR-018**: The system MUST treat primary and supporting trades as separate execution entities while still allowing them to share an overall strategy context and optional risk budget.
- **FR-019**: The system MUST allow users to review the strategy configuration, indicator settings, and run snapshot without altering the executing strategy state.
- **FR-020**: The system MUST preserve the existing behavior of Features 001 and 002 and treat this feature as an additive strategy-engine capability rather than a replacement or redesign of earlier features.
- **FR-020A**: The system MUST NOT introduce a competing `AlgorithmRun` lifecycle state machine in Feature 003, and Feature 003 MUST NOT independently authorize pause, failure, restart, recovery, or activation of the parent `AlgorithmRun`.
- **FR-020B**: Feature 003 MUST reconcile main/supporting trade strategy state against the authoritative Feature 002 run, risk, and execution records rather than assuming that an emitted signal closes or reopens a position.
- **FR-020C**: During recovery or reconciliation, Feature 003 MUST NOT create new strategy entries until Feature 002 explicitly authorizes resumption, and all restart/resume operations MUST respect Feature 002 duplicate-run prevention.

### Key Entities _(include if feature involves data)_

- **Strategy Configuration**: Represents the complete multi-timeframe algorithm definition, including timeframes, roles, criteria, and risk settings.
- **Timeframe Rule**: Represents a timeframe assigned to a specific role, such as primary trend, confirmation, or entry/exit.
- **Indicator Definition**: Represents a configured indicator, such as MACD or Heikin Ashi, including its parameters and logic for that timeframe.
- **Trade Leg**: Represents either the primary trade or a supporting trade within the same strategy run, with independent state and risk.
- **Run Snapshot**: Represents an immutable point-in-time record of the strategy configuration and operational state used for recovery and audit purposes.
- **Risk Budget**: Represents the optional shared limit that spans multiple related trades within a strategy run.
- **Execution Event**: Represents a timestamped signal, entry, exit, risk evaluation, or reversal action for a strategy or trade leg.
- **User**: Represents the account owner whose strategy configurations and associated trades are managed separately from other users.

## Success Criteria _(mandatory)_

### Measurable Outcomes

- **SC-001**: Users can create and save a valid multi-timeframe strategy with at least two timeframes and explicit roles in under 5 minutes.
- **SC-002**: At least 95% of valid multi-timeframe configurations persist successfully in PostgreSQL and remain retrievable after a restart.
- **SC-003**: A primary higher-timeframe bias and at least one supporting lower-timeframe trade can coexist in the same strategy without cross-trade state corruption.
- **SC-004**: Users can review a strategy’s indicator configuration and run snapshot without changing the currently executing strategy state.
- **SC-005**: Duplicate signal processing is prevented for the same candle or decision window, and no look-ahead bias is applied during evaluation.
- **SC-006**: The system supports independent current-position disposition and future-entry admission policies for the primary trend and supporting trades, with idempotent reversal handling and without bypassing Feature 002 lifecycle, risk, or execution authority.
- **SC-007**: 100% of active runs preserve independent position state, risk state, execution history, and P&L for all trade legs within the strategy.
- **SC-008**: The feature defaults to paper trading and does not create live trades as part of the MVP.
- **SC-009**: Each strategy run allows the user to define the higher-timeframe primary-trend rule from the configured timeframe rules and indicator set without forcing a single built-in trend definition across all strategies.

## Assumptions

- The strategy engine is additive to the existing features and does not remove or replace the earlier multi-algorithm management features.
- Each user owns a separate set of strategy configurations and runs, and uniqueness is enforced within the owning user context.
- A multi-timeframe strategy may include one primary directional view and one or more supporting trade legs, but the primary trade remains governed by the higher-timeframe bias.
- Indicator conditions are configured per timeframe and must be evaluated using the most recent completed candle data, not partial or future candle data.
- The MVP uses paper trading only and does not include live broker integration or live order placement.
- Shared risk budget behaviour is optional and is enforced only when enabled by the user for the strategy.
- Configuration snapshots are immutable and intended for audit and recovery, not for modifying a live run without an explicit reactivation or restart flow.
- Supporting trades may open and close repeatedly while the higher-timeframe directional bias remains valid, subject to configured risk and entry/exit rules.

## Notes on Unresolved Business Decisions

The following trading behaviours require stakeholder confirmation before implementation planning:

- The exact primary trend reversal policy should be defined at the trade-leg level, including whether existing open positions are closed or held and whether future entries are blocked or gated until a new bias is confirmed; however, the actual run-level recovery and activation decision remains the responsibility of Feature 002.
- The precise rules for lower-timeframe trade entry and exit repetition should be clarified, including how many repeated entries are allowed within a valid higher-timeframe bias window and whether they are capped by total exposure.
- The exact definition of a “completed candle” should be confirmed for each timeframe, including how to handle delayed updates, partial bars, out-of-order data, and session boundary transitions.
- The shared risk budget override rules should be clarified, including whether the budget is allocated globally across all trade legs or by trade family within a strategy. Risk enforcement remains in the shared risk engine, while Feature 003 defines strategy policy context and requests actions.
- The exact set of MACD and Heikin Ashi conditions should be finalized for MVP, including the degree of condition complexity allowed per timeframe.

## Notes on Scope

This feature extends TradePilot with indicator-based multi-timeframe strategy logic but does not define broker connectivity, live order execution, or external signal providers. It is intentionally targeted at strategy configuration, evaluation rules, risk state, and run history for paper-trading workflows only.
