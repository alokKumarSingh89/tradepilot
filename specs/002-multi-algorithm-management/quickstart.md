# Quickstart: Multi-Algorithm Management Validation

## Purpose

This guide provides a lightweight validation path for the Multi-Algorithm Management feature. It is intentionally design-focused and does not include implementation code.

## Prerequisites

- Docker Compose available locally
- Python 3.13+
- Node.js and package manager for frontend tooling
- PostgreSQL service available via local Compose stack
- Access to the TradePilot repo and the Feature 002 specification

## Local environment setup

1. Start the database and supporting services with Docker Compose.
2. Apply database migrations using Alembic for the backend schema.
3. Start the FastAPI backend and the algorithm worker in separate processes or containers.
4. Start the React frontend for the management UI.

## Validation scenarios

### 1. Create and persist multiple algorithm configurations

- Open the algorithm management screen.
- Create three distinct algorithm configurations with unique names and different parameter sets.
- Save them and confirm each is persisted in PostgreSQL.
- Refresh the UI or restart the backend and verify the saved records remain retrievable.

Expected outcome: each configuration is unique per user, saved as a versioned record, and available for later execution.

### 2. Start and monitor independent runs

- Start two different algorithm configurations in paper mode.
- Confirm both runs are visible in the monitoring dashboard.
- Verify each run has independent status, risk state, position list, and history.

Expected outcome: stopping one run does not stop the other, and both continue to maintain isolated execution state.

### 3. Validate stop behavior with open positions

- Start a run, create an open position state, and request stop.
- Confirm the UI prompts the user to keep or close the position before finalizing the stop action.
- Compare the status and risk state before and after confirmation.

Expected outcome: no silent forced exit is performed, and the correct user decision is logged.

### 4. Validate duplicate-run prevention

- Trigger a second start action for an already active configuration.
- Confirm the API rejects the request and displays a clear duplicate-run message.

Expected outcome: the system blocks the duplicate start request and preserves the original active run.

### 5. Validate recovery on restart

- Start a run and simulate an application restart.
- Reopen the dashboard and confirm the run is visible in a recoverable or reconciled state.
- Choose recover, restart, or discard and verify the resulting state matches the selected option.

Expected outcome: the system does not silently create a second active execution and keeps the run state transparent to the user.

### 6. Validate versioning and snapshots

- Save a configuration change without affecting an active run.
- Verify the new version is stored and the old version remains accessible.
- Confirm the run still uses its original version and that snapshots reflect the run’s historical state.

Expected outcome: configuration version history remains separate from active run execution state.

### 7. Validate stale-market-data and restart failure handling

- Simulate a stale or delayed market-data event while a run is active.
- Confirm the risk state marks the run as degraded or recovering and does not silently continue.
- Restart the application and confirm the run enters a recovery prompt rather than auto-resuming or creating a duplicate run.

Expected outcome: stale-market-data conditions remain visible, auditable, and safe.

### 8. Validate duplicate-request resilience and partial-fill handling

- Trigger duplicate start, stop, and exit requests for the same run using repeated calls.
- Confirm that idempotency keys and database constraints prevent duplicate execution.
- Simulate a partial fill or partial exit state and verify it is logged without being treated as guaranteed completion.

Expected outcome: duplicate requests are harmless, and partial execution states remain explicitly visible and auditable.

## Exit criteria

The feature is considered ready for implementation tasks when all validation scenarios above can be executed end-to-end in the local environment and the key business rules remain consistent with the requirements in the feature specification.
