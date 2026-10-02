# PRD — Trading Strategy Management Platform

**Version:** 1.0  
**Project:** TradePilot  
**Development methodology:** Spec-Driven Development (GitHub Spec Kit)

## 1. Product Overview

TradePilot is a web-based trading strategy management platform that allows users to configure, simulate, monitor and execute rule-based trading strategies.

The initial implementation focuses on options straddle strategies.

## 2. Problem Statement

Traditional time-based trading algorithms can miss stop-loss and take-profit thresholds during rapid market movements.

The platform must support event-driven risk monitoring independently of strategy evaluation.

## 3. Functional Requirements

### FR-001 — Strategy Management

- Create trading strategies.
- Update strategy parameters.
- Enable and disable strategies.
- Maintain strategy configuration history.

### FR-002 — Straddle Strategy

- Configure CE and PE positions.
- Capture individual entry premiums.
- Calculate combined entry premium.
- Support configurable lot sizes.

### FR-003 — Risk Management

- Configure combined-premium stop-loss (100% default).
- Configure combined-premium take-profit (50% default).
- Monitor combined premium using live market events.
- Support individual-leg emergency stop-loss.
- Prevent duplicate exit execution.
- Account for slippage and partial fills.

### FR-004 — Market Data

- Integrate broker market-data WebSocket.
- Process CE and PE price updates.
- Detect stale prices and connection failures.
- Maintain timestamped market snapshots.

### FR-005 — Execution Engine

- Support paper trading initially.
- Support live trading in a later phase, subject to explicit activation and applicable broker requirements.
- Maintain order lifecycle states.
- Reconcile execution status with broker responses.
- Provide emergency exit functionality.

### FR-006 — Dashboard

- Display active strategies.
- Show CE and PE premiums.
- Show combined premium.
- Display unrealized and realized P&L.
- Display current risk thresholds.
- Display execution history.

## 4. Non-Functional Requirements

- Independent strategy and risk-monitoring processes.
- Event-driven risk evaluation.
- Idempotent order-execution requests.
- Structured application logs.
- Automated unit and integration tests.
- Secure broker credential storage.
- Graceful handling of WebSocket disconnections.

## 5. Initial Technology Proposal

Technology selection will be finalized during the Spec Kit planning phase.

- Backend: Python / FastAPI.
- Frontend: React / TypeScript.
- Database: PostgreSQL.
- Cache and messaging: Redis (if justified).
- Testing: Pytest and Vitest.
- Infrastructure: Docker.

## 6. MVP Acceptance Criteria

1. Users can configure a paper-trading straddle.
2. Combined premium updates on incoming market ticks.
3. The risk engine detects configured SL and TP breaches.
4. Only one exit workflow can be active per position group.
5. All simulated executions are recorded.
6. Disconnections and stale market data are visibly reported.
7. SL/TP triggers do not imply guaranteed execution prices.
