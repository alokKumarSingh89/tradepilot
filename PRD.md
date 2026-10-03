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

## FR-007 — Multi-Algorithm Management

Users can create, configure, start, stop,
and monitor multiple algorithm instances
from the web interface.

Requirements:

- Select a supported algorithm type.
- Configure instrument, strategy parameters,
  SL, TP, quantity, and execution mode.
- Persist configurations in the database.
- Load configurations before execution.
- Run multiple instances concurrently.
- Maintain independent positions, P&L,
  risk state, and execution history.
- Start and stop individual instances.
- Do not silently modify running instances
  when saved configurations change.
- Prevent unintended duplicate runs.
- Recover and reconcile state after restarts.
- Prevent duplicate order submissions.
- Use paper trading by default.
- Share market-data updates where appropriate.
- Monitor risk independently of strategy frequency.
- Handle stale market data safely.

Initial strategy: Short Straddle.

Future strategies:

- Short Strangle
- Intraday Directional
- Positional

Acceptance criteria:

1. Create three configurations from the UI.
2. Configurations survive application restart.
3. Two configurations can run concurrently
   in paper-trading mode.
4. Stopping one does not stop another.
5. Each instance has independent risk state.
6. Recovery prevents duplicate execution.
7. UI displays individual lifecycle status.

## FR-008 — Multi-Timeframe Indicator-Based Strategy Engine

### Objective

TradePilot must allow users to create indicator-based trading algorithms through the UI, configure multiple timeframes, and execute main and supporting trades independently based on configured conditions.

### FR-008.1 — Multi-Timeframe Configuration

- Users can configure two or more timeframes per algorithm.
- Each timeframe must have a defined responsibility:
  - Primary Trend (Higher Timeframe).
  - Confirmation (Optional).
  - Entry/Exit (Lower Timeframe).
- Timeframes must be configurable from the UI.
- Initially support 5M, 15M, 30M, 1H, 4H and 1D.

### FR-008.2 — Indicator Configuration

Users must be able to select indicators independently for each timeframe.

Initial supported indicators:

- MACD (configurable Fast, Slow and Signal periods).
- Heikin Ashi (HA).

The system must support configurable conditions such as:

- MACD crossover.
- MACD histogram direction and colour changes.
- MACD slope/bend changes (using an explicitly defined calculation).
- Heikin Ashi bullish/bearish candle.
- Heikin Ashi candle colour change from the previous candle.

Additional indicators (EMA, RSI, Supertrend, etc.) must be extensible without redesigning the complete strategy engine.

### FR-008.3 — Primary Trade (Higher Timeframe)

- The primary trade follows the configured higher-timeframe trend.
- The algorithm determines bullish, bearish or neutral market conditions.
- Primary trade entry and exit conditions must be configurable.
- The primary trade maintains independent position and P&L information.

### FR-008.4 — Supporting Trade (Lower Timeframe)

- Supporting trades follow the configured higher-timeframe directional bias.
- Lower-timeframe indicators determine supporting trade entries and exits.
- Multiple supporting trades may occur during one higher-timeframe trend.
- Supporting trades must maintain independent entry, exit, quantity, P&L and risk information.
- The system must prevent duplicate entries from the same signal.
- The behaviour when the primary trend reverses must be configurable.

### FR-008.5 — Risk Management

- Support independent SL and TP for main and supporting trades.
- Support optional shared risk limits across related trades.
- Risk monitoring must operate independently of candle-based strategy evaluation.
- Market-data updates should trigger risk evaluation without waiting for the next strategy candle.
- Actual exit prices may differ from configured thresholds due to slippage and execution latency.

### FR-008.6 — Configuration Persistence

- All timeframe, indicator, entry, exit and risk configurations must be persisted in PostgreSQL.
- The algorithm worker retrieves configuration from the database before execution.
- Each execution must use an immutable configuration snapshot.
- Configuration changes must not silently modify running algorithms.

### FR-008.7 — Candle and Signal Processing

- Candle timestamps and timeframe boundaries must be consistent.
- The system must explicitly define whether conditions use completed or incomplete candles.
- Prevent look-ahead bias during strategy evaluation and backtesting.
- Handle missing candles, delayed ticks and WebSocket disconnections.
- Maintain an auditable signal and execution history.

### FR-008.8 — Example Trading Configuration

**Algorithm:** NIFTY Multi-Timeframe Trend Following

| Configuration          | Value                   |
| ---------------------- | ----------------------- |
| Primary Timeframe      | 4H                      |
| Confirmation Timeframe | 1H                      |
| Supporting Timeframe   | 15M                     |
| Indicators             | MACD + Heikin Ashi      |
| Main Trade             | Based on 4H trend       |
| Supporting Trade       | Based on 15M entry/exit |
| Direction Filter       | Higher-timeframe trend  |
| Execution Mode         | Paper Trading (MVP)     |

### Acceptance Criteria

1. A user can configure three timeframes through the UI.
2. Each timeframe supports independent indicator settings.
3. The algorithm configuration is saved in PostgreSQL.
4. The worker loads the saved configuration before starting.
5. Main and supporting trades maintain independent runtime states.
6. Supporting trades can enter and exit repeatedly while the primary trend remains valid.
7. Risk evaluation is independent of candle evaluation frequency.
8. Duplicate signals do not create duplicate orders.
9. All decisions and execution events are recorded.
10. Application restarts do not silently duplicate trading positions.
11. Paper trading is the default execution mode.

### Out of Scope for Initial Implementation

- AI-generated trading signals.
- Automatic strategy optimization.
- Live broker execution.
- Machine-learning price prediction.
- Multi-account portfolio management.
