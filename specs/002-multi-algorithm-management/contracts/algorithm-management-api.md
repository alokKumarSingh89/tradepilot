# Algorithm Management API Contract

## Overview

This contract defines the management API for creating, versioning, starting, stopping, and reviewing algorithm configurations and runs. It covers the application-level contract for the frontend and does not define broker-facing or live trading behavior.

## Base path

- /api/v1

## Core entities

- AlgorithmConfiguration
- AlgorithmConfigurationVersion
- AlgorithmRun
- Position
- OrderRecord
- RiskState
- LifecycleEvent

## Endpoints

### GET /algorithms

Returns all algorithm configurations visible to the current user.

Response shape:
- id
- name
- algorithm_type
- execution_mode
- current_version_id
- created_at
- updated_at
- active_run_count

### POST /algorithms

Creates a new algorithm configuration.

Request body:
- name
- algorithm_type
- config_payload
- execution_mode

Validation:
- name must be unique within a user scope
- required config fields must be present
- live execution is rejected in MVP unless explicitly approved later

Response:
- created configuration record
- initial version record

### GET /algorithms/{id}

Returns full configuration details and version history.

Response includes:
- configuration details
- current version summary
- previous versions
- active and historical run references

### PATCH /algorithms/{id}

Saves a new version for an algorithm configuration without mutating the behavior of running instances.

Request body:
- config_payload
- notes, optional

Behavior:
- a new version record is created
- active run remains bound to the previous runtime version
- the config is updated as the latest version for future runs

### POST /algorithms/{id}/runs

Starts a new run from a saved configuration version.

Request body:
- configuration_version_id
- execution_mode
- run_name, optional

Rules:
- duplicate start is rejected when the same configuration is already active for the user
- a run record is created in a pending or ready state
- the worker claims the run asynchronously

Response:
- run id
- status
- recovery_state

### GET /runs/{runId}

Returns the current run state, latest risk state, position list, and lifecycle summary.

### POST /runs/{runId}/start

Triggers a start action for a run that exists but is not yet active. Must be idempotent.

### POST /runs/{runId}/stop

Initiates a stop sequence. If open positions remain, the API returns a prompt or confirmation state rather than silently closing the position.

### POST /runs/{runId}/stop/decision

Confirms the user decision for a stop that still has open positions.

Request body:
- decision: keep_position | close_position
- idempotency_key

Behavior:
- if `keep_position`, the run enters a stopped or pending state without forcibly closing the position
- if `close_position`, the run coordinates a controlled exit workflow and records the user decision in lifecycle history

### POST /runs/{runId}/recover

Recovers a run after restart or interrupted execution. This action requires explicit user confirmation or a safe state reconciliation process.

### POST /runs/{runId}/recover/decision

Confirms the user choice after a restart interruption.

Request body:
- decision: recover | restart | discard
- idempotency_key

Behavior:
- no silent auto-resume is allowed
- the chosen decision becomes the authoritative recovery action and is logged as a lifecycle event

### GET /runs/{runId}/history

Returns a time-ordered list of lifecycle events, risk transitions, and user actions.

### GET /runs/{runId}/snapshots

Returns saved run snapshots for review and state restoration.

## Error handling

- 409 Conflict: duplicate active run or duplicate command attempt
- 400 Bad Request: invalid config or missing required fields
- 404 Not Found: configuration or run does not exist
- 422 Unprocessable Entity: state transition is not allowed during current lifecycle

## Idempotency

The API must accept repeated requests by using request-level operation ids or a database uniqueness constraint. Repeated start or stop commands for the same run result in either a no-op or a clear state response, but never a second execution.

## Security and access control

- User access is scoped to the owning user account.
- Cross-user algorithm or run access is forbidden.
- Sensitive operational logs remain internal and auditable.
