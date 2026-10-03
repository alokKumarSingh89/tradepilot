# Quickstart: Multi-Timeframe Strategy Engine Validation

## Purpose

This guide explains the validation steps needed to confirm the Feature 003 design works with the approved Feature 002 lifecycle, persistence, risk, and execution model. It is intentionally a validation guide, not an implementation guide.

## Prerequisites

- Docker Compose environment for PostgreSQL and platform services.
- Python 3.12+ and FastAPI backend environment.
- React + TypeScript frontend environment.
- Pytest and Vitest configured in the repository.
- Feature 002 orchestration and risk components available and running.

## Environment setup

1. Start the project services with Docker Compose.
2. Confirm PostgreSQL is reachable and the schema for configurations, runs, snapshots, and strategy-runtime tables is created via Alembic migrations.
3. Confirm the backend API is reachable and the frontend can load the strategy management screen.

## Validation scenario 1: create a valid multi-timeframe strategy

1. Open the strategy creation screen.
2. Create a strategy that includes a 4H primary trend timeframe and a 15M entry/exit timeframe.
3. Assign an optional 1H confirmation timeframe if the user selects it.
4. Configure MACD and Heikin Ashi for each active timeframe.
5. Save the configuration and confirm the record persists in PostgreSQL.
6. Expected result: the strategy is stored, the configured roles are visible, and the runtime accepts the configuration.

## Validation scenario 2: restart recovery with an active run

1. Start a paper-trading run from the strategy configuration.
2. Confirm a run snapshot exists before and after the run becomes active.
3. Restart the backend application or worker process.
4. Reconcile the run from stored state without creating duplicate execution.
5. Expected result: the run retains its state, snapshot, and trade-leg context without duplicate entries or silent reprocessing.

## Validation scenario 3: higher-timeframe bias plus supporting trade behavior

1. Configure an active 4H bullish bias with supporting 15M entries.
2. Ensure the strategy evaluates only completed candles by default.
3. Trigger a valid entry signal on the 15M timeframe while the 4H bias remains bullish.
4. Confirm the main trade and supporting trades coexist independently.
5. Expected result: supporting positions open under the valid bias, and each trade leg keeps its own lifecycle and P&L state.

## Validation scenario 4: reversal handling

1. Create a run with a configured primary trend reversal policy.
2. Trigger a primary trend reversal while supporting trades are still active.
3. Confirm the run executes the configured reversal behavior.
4. Expected result: supporting trades are closed or paused according to the selected strategy policy, and the state transition is recorded in the run history.

## Validation scenario 5: duplicate signal prevention

1. Feed the same timeframe signal repeatedly within the same candle bucket and trade-leg context.
2. Confirm the strategy engine suppresses duplicates.
3. Feed the same signal to a different trade leg or opposite direction.
4. Expected result: same-leg duplicate signals are ignored, while distinct trade-leg decisions continue to be evaluated.

## Validation scenario 6: late or out-of-order candle handling

1. Deliver a late candle update for a finalized timeframe bucket.
2. Confirm the final state is preserved and no previous signal is reprocessed by default.
3. Expected result: the system ignores the stale update or evaluates it only under an explicit reprocessing workflow.

## Validation scenario 7: shared risk budget enforcement

1. Enable a shared risk budget on a strategy with a main trade and multiple supporting trades.
2. Open several support legs until the shared risk budget nears its limit.
3. Attempt to open another support trade.
4. Expected result: the engine blocks additional trade-leg creation when the configured budget cap is reached.

## Proposed checks to run

- Backend tests for strategy runtime validation, risk budget handling, and snapshot recovery.
- Frontend tests for strategy config screens and run detail display.
- Integration tests for worker lifecycle and run state reconciliation.
- Migration validation to confirm tables for strategy-runtime state and trade-leg history are created correctly.

## Expected outcomes

- Configuration remains persistent and versioned.
- Multiple trade legs remain independent while operating under the same primary bias.
- No duplicate or look-ahead signals are executed.
- Restart recovery is explicit and safe.
- The design remains compatible with the Feature 002 orchestration and worker model.
