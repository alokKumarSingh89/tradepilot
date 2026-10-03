# Feature Specification: Multi-Timeframe Indicator-Based Strategy Engine

**Feature Branch**: `003-multi-timeframe-indicator-based-strategy-engine`

**Created**: 2026-10-03

**Status**: Draft

**Input**: User description: "Read PRD.md and the existing TradePilot Constitution. Create a NEW Feature 003: Multi-Timeframe Indicator-Based Strategy Engine. Extend TradePilot to support UI-configurable indicator-based trading algorithms. Requirements: 1. Users can configure two or more timeframes per algorithm. 2. Each timeframe has an explicit role: primary trend, optional confirmation, or entry/exit. 3. Support indicator configuration per timeframe, beginning with MACD and Heikin Ashi. 4. Indicator parameters must be configurable. 5. Allow conditions such as MACD crossover, MACD slope/bend, histogram changes, and HA candle colour changes. 6. Support a primary trade governed by higher-timeframe conditions. 7. Support independent lower-timeframe trades that follow the configured higher-timeframe directional bias. 8. Allow supporting trades to enter and exit multiple times while the higher-timeframe bias remains valid. 9. Define configurable behaviour when the primary trend reverses. 10. Maintain independent position state, risk rules, execution history, and P&L for main and supporting trades. 11. Support an optional shared risk budget across related trades. 12. Persist all strategy and indicator configuration in PostgreSQL. 13. Capture an immutable configuration snapshot for each run. 14. Evaluate indicators using timestamped, appropriately completed candles; define incomplete-candle behaviour explicitly. 15. Prevent duplicate signal processing and look-ahead bias. 16. Use paper trading by default. Preserve Features 001 and 002. Generate business requirements, user stories, acceptance criteria, and edge cases. Identify ambiguous trading rules for clarification. Do not generate application code or a technical plan."

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
2. **Given** the primary higher-timeframe trend flips or reverses, **When** the reversal condition is met, **Then** the system follows the configured reversal behaviour for the primary trade and the supporting trades associated with that bias.
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
- **FR-006**: The system MUST support Heikin Ashi candle-colour conditions and related directional colour transitions.
- **FR-007**: The system MUST support a primary trade governed by the higher-timeframe directional bias of the strategy.
- **FR-008**: The system MUST support independent lower-timeframe trades that follow the configured higher-timeframe direction while the bias remains valid.
- **FR-009**: The system MUST allow supporting trades to enter and exit multiple times while the higher-timeframe directional bias remains valid, without being forced to collapse into a single trade state.
- **FR-010**: The system MUST support configurable behaviour when the primary trend reverses, including the rule for whether supporting trades close, pause, flatten, or remain under a user-defined override.
- **FR-011**: The system MUST maintain independent position state, risk rules, execution history, and P&L for the primary trade and each supporting trade.
- **FR-012**: The system MUST support an optional shared risk budget across related trades within the same strategy run.
- **FR-013**: The system MUST persist all strategy metadata, timeframe assignments, indicator definitions, parameter sets, and associated risk settings in PostgreSQL.
- **FR-014**: The system MUST capture an immutable configuration snapshot for each run at the time the run starts and whenever material strategy or risk state changes occur.
- **FR-015**: The system MUST evaluate indicators using timestamped, appropriately completed candles and MUST define explicit handling for incomplete candles, delayed data, and out-of-order updates.
- **FR-016**: The system MUST prevent duplicate signal processing, repeated execution of the same decision within the same candle, and look-ahead bias during strategy evaluation.
- **FR-017**: The system MUST default to paper trading for all multi-timeframe strategy runs and prevent live execution from being activated as part of this feature.
- **FR-018**: The system MUST treat primary and supporting trades as separate execution entities while still allowing them to share an overall strategy context and optional risk budget.
- **FR-019**: The system MUST allow users to review the strategy configuration, indicator settings, and run snapshot without altering the executing strategy state.
- **FR-020**: The system MUST preserve the existing behavior of Features 001 and 002 and treat this feature as an additive strategy-engine capability rather than a replacement or redesign of earlier features.

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
- **SC-006**: The system supports a configured reversal policy for the primary trend and keeps supporting-trade behaviour consistent with that policy.
- **SC-007**: 100% of active runs preserve independent position state, risk state, execution history, and P&L for all trade legs within the strategy.
- **SC-008**: The feature defaults to paper trading and does not create live trades as part of the MVP.

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

- The exact primary trend reversal policy should be defined, including whether a reversal closes all related supporting trades immediately, pauses the strategy, or requires a user-confirmed override.
- The precise rules for lower-timeframe trade entry and exit repetition should be clarified, including how many repeated entries are allowed within a valid higher-timeframe bias window and whether they are capped by total exposure.
- The exact definition of a “completed candle” should be confirmed for each timeframe, including how to handle delayed updates, partial bars, out-of-order data, and session boundary transitions.
- The shared risk budget override rules should be clarified, including whether the budget is allocated globally across all trade legs or by trade family within a strategy.
- The exact set of MACD and Heikin Ashi conditions should be finalized for MVP, including the degree of condition complexity allowed per timeframe.

## Notes on Scope

This feature extends TradePilot with indicator-based multi-timeframe strategy logic but does not define broker connectivity, live order execution, or external signal providers. It is intentionally targeted at strategy configuration, evaluation rules, risk state, and run history for paper-trading workflows only.
