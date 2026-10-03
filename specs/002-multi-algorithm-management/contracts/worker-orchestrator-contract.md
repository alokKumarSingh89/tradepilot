# Algorithm Worker Orchestrator Contract

## Overview

The worker process is responsible for safely claiming eligible runs, subscribing to market-data events, evaluating risk, and maintaining per-run execution state without blocking management API calls.

## Responsibilities

- claim eligible runs from the database
- maintain run-specific runtime contexts
- monitor lifecycle and market-data events
- persist snapshots and risk events
- handle idempotent stop and exit semantics
- reconcile interrupted or recoverable states after restart

## Core contracts

### 1. Run claim contract

When a run is eligible to start:
- the worker must acquire a database-level lock or atomic row transition
- the run transitions from ready to running only once
- a duplicate claim is rejected with an explicit status message

### 2. Market-data subscription contract

The worker subscribes to market-data events for the run’s configured instruments and symbols.

Rules:
- subscription is per run and instrument
- stale data is flagged and handled as degraded risk state
- shared market-data adapters remain centralized, but per-run routing is isolated

### 3. Risk evaluation contract

Risk evaluation is event-driven and not tied to strategy loop cadence.

Rules:
- risk events may be triggered by order updates, price changes, or lifecycle changes
- a risk threshold breach results in a persisted event and a safe run state transition
- a stop request with open positions requires user confirmation before finalization

### 4. Stop and exit contract

The worker handles stop commands idempotently.

Rules:
- a stop request must not create a second exit subject to a duplicate command id
- if positions remain open, the worker pauses the run and waits for the decision workflow
- a force-close action is not allowed in this MVP without a user-authorized action

### 5. Recovery contract

After application restart:
- runs are reconciled from persisted state
- interrupted or partial states become recoverable rather than silently resumed
- all recovery decisions are logged and visible to the user

## Persistence guarantees

- snapshot writes are durable
- lifecycle events are append-only
- a status change is considered complete only after transaction commit
- duplicate start and stop requests are rejected through database-level uniqueness or concurrency checks

## Failure handling

- stale market data triggers a degraded state alert
- broker or adapter failures are recorded and visible without forcing execution
- shutdown is graceful: the worker marks the run as stopped or recovering, flushes last state, and exits without silent data loss
