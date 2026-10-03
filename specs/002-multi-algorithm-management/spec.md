# Feature Specification: Multi-Algorithm Management and Execution Lifecycle

**Feature Branch**: `002-multi-algorithm-management`

**Created**: 2026-10-02

**Status**: Draft

**Input**: User description: "Create a new feature specification for FR-007: Multi-Algorithm Management and Execution Lifecycle. Users must be able to create algorithm configurations from the UI, persist them in PostgreSQL, and start multiple independent algorithm instances. The feature must cover algorithm creation and configuration, database persistence and retrieval, start, stop, and lifecycle status, concurrent algorithm execution, independent risk state and history, configuration versioning and run snapshots, restart recovery and duplicate-run prevention, paper-trading-first execution, and UI monitoring of individual runs. The scope excludes broker integration, live-market order placement, and implementation detail decisions not required to define business behavior."

## User Scenarios & Testing _(mandatory)_

### User Story 1 - Create and configure multiple algorithm entries (Priority: P1)

A user who wants to manage several automated strategies can create separate algorithm configurations from the UI and save them for later use.

**Why this priority**: Without a saveable, reviewable algorithm definition, the user cannot run or monitor multiple strategies independently.

**Independent Test**: A user creates two or more valid algorithm configurations with different settings and confirms that each is saved as a distinct record ready to run.

**Acceptance Scenarios**:

1. **Given** a user is creating a new algorithm, **When** they enter a unique name, supported algorithm type, and required configuration values, **Then** the system accepts the configuration and stores it for later use.
2. **Given** a user attempts to create an algorithm with missing required details or invalid values, **When** they submit the form, **Then** the system rejects the submission and explains which fields need correction.

---

### User Story 2 - Start independent runs and monitor them concurrently (Priority: P1)

A user can start more than one algorithm instance at the same time and monitor each one independently without affecting the other runs.

**Why this priority**: Concurrent execution and independent monitoring are the core business value of the feature and the primary risk area for operational correctness.

**Independent Test**: A user starts two valid algorithm instances with different configurations and confirms that each run has its own status, active state, and lifecycle history.

**Acceptance Scenarios**:

1. **Given** two valid algorithm configurations exist, **When** the user starts both in paper-trading mode, **Then** both algorithms enter an active running state and remain independently tracked.
2. **Given** one running algorithm is stopped, **When** the user checks the system, **Then** only the selected run changes state and the other run continues without interruption.
3. **Given** a running algorithm has an open position, **When** the user requests a stop, **Then** the system asks whether to keep or close the position before finalizing the stop action and updates the run status only after the user’s decision.

---

### User Story 3 - Recover and review run state after restart (Priority: P1)

A user expects the system to retain saved configuration and execution state so the environment can recover safely after a restart without creating duplicate runs.

**Why this priority**: Recovery and duplicate prevention are essential for operational reliability and reduce the risk of repeated or unintended execution.

**Independent Test**: A user restarts the application after a configured run is saved, then confirms the last known state is restored without creating a duplicate execution.

**Acceptance Scenarios**:

1. **Given** an algorithm has been saved and previously started, **When** the application restarts, **Then** the system restores the persisted configuration and last known execution state without creating a duplicate active run and presents the run as recoverable for user confirmation.
2. **Given** a user attempts to start an algorithm run that is already active, **When** they trigger the action again, **Then** the system prevents the duplicate run and surfaces an understandable status message.
3. **Given** the application restarts after an interrupted run, **When** the user reviews the recovered state, **Then** the system offers the choice to recover, restart, or discard the interrupted run rather than auto-resuming it silently.

---

### User Story 4 - Review histories and versioned configurations (Priority: P2)

A user can review a configuration’s history, compare versions, and inspect each run snapshot or lifecycle history without affecting live execution.

**Why this priority**: Versioning and history are important for governance and operational understanding, but they are secondary to successful execution and recovery.

**Independent Test**: A user edits a safe configuration, saves a new version, and confirms the system keeps the version history and run snapshot while the active run remains isolated.

**Acceptance Scenarios**:

1. **Given** a saved algorithm configuration is updated, **When** the user saves the new version, **Then** the system retains the prior version and records the change as part of the configuration history.
2. **Given** a run is active, **When** the user views history or snapshots, **Then** the system shows the run-specific state without modifying the running algorithm or its risk state.
3. **Given** a saved algorithm configuration is active in a running instance, **When** the user edits and saves the configuration, **Then** the system creates a new version for future use while leaving the current active run unchanged and unaffected.

---

### Edge Cases

- What happens when a user creates multiple algorithms with the same name but different configurations?
- What happens when two users create algorithms with the same name in the same system?
- What happens when a user starts the same algorithm configuration twice in rapid succession?
- What happens when a running algorithm is edited while active?
- What happens when the application restarts while an algorithm is running and a duplicate run is possible?
- What happens when market or risk data is stale during a run?
- What happens when a user stops one algorithm instance and the others continue running?

## Requirements _(mandatory)_

### Functional Requirements

- **FR-001**: The system MUST allow a user to create a new algorithm configuration from the user interface.
- **FR-002**: The system MUST support multiple algorithm configurations, each with a unique name within the owning user context.
- **FR-003**: The system MUST allow a user to configure the algorithm type and the required strategy parameters for that type.
- **FR-004**: The system MUST persist algorithm configurations in PostgreSQL and allow them to be retrieved later for review or execution.
- **FR-005**: The system MUST allow the user to start, stop, and monitor individual algorithm instances independently.
- **FR-006**: The system MUST support multiple independent algorithm instances running concurrently without shared execution state.
- **FR-007**: The system MUST keep each algorithm run’s risk state, P&L, and lifecycle history independent from all other runs.
- **FR-008**: The system MUST maintain configuration version history so users can identify the current version and previous saved versions of an algorithm.
- **FR-009**: The system MUST record run snapshots that preserve the state of an algorithm instance at the time it starts or changes status.
- **FR-010**: The system MUST restore persisted configuration and run data after a restart without creating unintended duplicate active runs.
- **FR-011**: The system MUST prevent duplicate run requests for the same logical algorithm instance when an active run already exists.
- **FR-012**: The system MUST default algorithm execution to paper trading and require explicit activation before live execution is allowed.
- **FR-013**: The system MUST provide a user-visible status for each algorithm instance, including its lifecycle stage and whether it is active, stopped, or recovering.
- **FR-014**: The system MUST allow users to review individual run activity and risk history without changing the behavior of other algorithm instances.
- **FR-015**: The system MUST prevent live positions or market orders from being affected by saving or editing a configuration that is not actively trading.
- **FR-016**: The system MUST prompt the user to choose whether to keep or close an open position before finalizing a stop request for an algorithm run that still has active positions.
- **FR-017**: The system MUST block a new run start when the same algorithm configuration is already active for that user and present the duplicate run as a rejected action with an explanatory status message.
- **FR-018**: The system MUST allow a user to save a new version of an active algorithm configuration without altering the behavior of the currently running instance; the active run remains bound to its original version until a separate restart or explicit reactivation decision.
- **FR-019**: The system MUST detect interrupted or incomplete run state after restart and present the user with an explicit recovery choice to recover, restart, or discard the run without creating an unintended duplicate active execution.

### Key Entities _(include if feature involves data)_

- **Algorithm Configuration**: Represents a user-defined algorithm setup, including its name, type, parameter values, execution mode, and configuration history.
- **Algorithm Run**: Represents a single execution instance created from a configuration, with its own lifecycle state, timestamps, risk state, and run snapshots.
- **Risk State**: Represents the current risk metrics, P&L, position status, and state transitions associated with a single run.
- **Run Snapshot**: Represents a persisted point-in-time record of a run’s key state for recovery, auditing, and monitoring.
- **Configuration Version**: Represents a saved revision of an algorithm configuration that may be compared with earlier or later versions.
- **User**: Represents the account owner whose algorithms and runs are stored and managed separately from other users.

## Success Criteria _(mandatory)_

### Measurable Outcomes

- **SC-001**: Users can create and save at least three distinct algorithm configurations from the user interface in under 5 minutes.
- **SC-002**: At least 95% of valid algorithm configurations persist successfully and are retrievable after a restart without data loss.
- **SC-003**: Two independent algorithm instances can run concurrently in paper-trading mode without shared state corruption or lifecycle interference.
- **SC-004**: When a user stops one algorithm instance, the other running instances continue without interruption, and their statuses remain distinct.
- **SC-005**: 100% of duplicate start requests for the same active algorithm instance are blocked and identified as duplicates to the user.
- **SC-006**: Recovery after an application restart preserves current configuration and run state without creating an unintended second active run.
- **SC-007**: Users can review algorithm lifecycle status and run history for each instance in the UI without modifying the behavior of other runs.
- **SC-008**: The feature supports a paper-trading-first model with no live execution triggered by save, start, stop, or review actions.

## Assumptions

- Each user owns a separate set of algorithm configurations and runs, and uniqueness is enforced within the owning user context.
- Algorithm start requests are treated as run lifecycle actions, not direct order submissions, and paper trading is the default unless explicit live activation is later approved.
- Each algorithm configuration may produce multiple run instances over time, but each logical run is tracked separately from the configuration definition.
- The system records enough state to recover a run safely after restart without assuming that the risk engine can be resumed without validation.
- The feature is limited to management and monitoring behavior; it does not include live broker execution or direct market-order placement.
- Configuration edits may be saved as new versions, but active runs should not silently change state as a consequence of a saved configuration change unless a separate business decision is later approved.
- When a stop request is issued for a run with open positions, the system asks the user whether to keep or close the position before finalizing the stop action; it does not silently force a position decision.
- Duplicate run prevention is based on the active state of the same algorithm configuration for the same user; a second start action for that active configuration is rejected as a duplicate.
- Saving a new version of an active configuration does not alter the current running instance; the current run remains bound to its original version until a new run is started from the updated configuration.
- After restart, the system restores the last known run information and asks the user to choose whether to recover, restart, or discard the interrupted run rather than silently resuming or duplicating execution.
- The system distinguishes between configuration-level metadata and run-level execution state, with each one treated as a separate concern for recovery and monitoring.

## Notes on Unresolved Business Decisions

The following business decisions remain open and should be confirmed before implementation planning:

- The exact lifecycle state model for algorithm runs should be confirmed, including whether “created,” “ready,” “running,” “stopped,” “failed,” and “recovered” are required states or whether the system uses a simpler set.
- The UI requirements for monitoring individual runs should be defined more precisely, including whether users need a list view, a detail view, or both.
- The acceptable limits for concurrent algorithms should be confirmed, especially whether the platform supports a user-defined cap or a system-wide limit.
- The exact requirements for run snapshots should be clarified, including whether they are generated only on start or also on lifecycle transitions and risk threshold events.

## Notes on Scope

This feature is intentionally limited to algorithm management and lifecycle behavior. It does not define broker connectivity, order execution logic, or automated market entry and exit behavior beyond the paper-trading-first requirement and the prevention of duplicate or unintended execution.
