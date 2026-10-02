<!-- Sync Impact Report
- Version change: 0.1.0 -> 1.0.0
- Modified principles: N/A -> I. Architecture & Separation of Concerns, II. Code Quality & Maintainability, III. Test-First and Regression Discipline, IV. Trading Safety & Risk Controls, V. Reliability, Resilience, and Operational Transparency
- Added sections: Technology & Delivery Constraints, Specification-Driven Development
- Removed sections: none
- Deferred items: none
-->

# TradePilot Constitution

## Core Principles

### I. Architecture & Separation of Concerns

TradePilot MUST separate strategy evaluation, market data processing, risk monitoring, and order execution into distinct responsibilities with explicit interfaces. Components MUST depend on abstractions rather than concrete broker or data-provider implementations, and the system MUST favor clear contracts over hidden coupling. Architecture changes MUST justify any move toward distributed deployment, and microservices MUST NOT be introduced without a documented need for independent scaling, failure isolation, or team ownership.

This rule exists to preserve testability, reduce operational risk, and keep strategy logic independent from infrastructure failures. When boundaries are unclear, the default is to keep the system modular and cohesive rather than splitting it prematurely.

### II. Code Quality & Maintainability

All implementation work MUST favor simple, readable, and maintainable code. Teams MUST use strong typing, explicit error handling, and small testable components. Complex logic MUST be decomposed into narrow functions and well-defined domains, with clear naming and minimal hidden state. Code that is difficult to reason about or validate is treated as a defect, not a technical preference.

This principle reduces operational mistakes in trading workflows and makes it easier to review, extend, and debug the platform without introducing hidden coupling or brittle behavior.

### III. Test-First and Regression Discipline

Automated tests are non-negotiable for all business and risk calculations, broker adapter behavior, and critical bug fixes. Unit tests MUST cover strategy and risk logic, integration tests MUST validate broker adapters and external contracts, and regression tests MUST be added whenever a real production defect is corrected. New behavior MUST be proven by tests before implementation is considered complete.

This principle ensures that financial logic remains trustworthy, that external integrations remain compatible, and that previously fixed defects do not reappear under future changes.

### IV. Trading Safety & Risk Controls

Paper trading is the default operating mode. Live execution requires explicit activation and a clear operational decision to proceed. The platform MUST prevent duplicate exit requests, handle partial fills safely, and treat stop-loss or take-profit orders as advisory signals rather than guaranteed execution prices. Risk limits, order validation, and exit handling MUST be enforced as first-class logic, not optional checks.

This principle protects capital and prevents unsafe automation from misinterpreting incomplete execution outcomes as successful risk mitigation.

### V. Reliability, Resilience, and Operational Transparency

The system MUST handle stale market data, WebSocket disconnections, broker failures, and application restarts without silently violating trading rules. Important trading and risk decisions MUST be logged with sufficient context to reconstruct operational state and explain outcomes after the fact. When components fail or recover, the system MUST degrade predictably and preserve safe defaults.

This rule exists because trading systems are highly sensitive to latency, partial failures, and missing context; resilience must be designed into the platform before it is needed under live conditions.

## Technology & Delivery Constraints

The platform MUST remain compatible with Python and FastAPI for server-side execution, React and TypeScript for user interfaces, and PostgreSQL for durable data storage. Technical choices are intentionally left open during planning so the team can select the most suitable implementation details for the product, but they MUST remain consistent with these core platform boundaries and with the architecture principles above.

The project MUST avoid introducing technology that undermines maintainability, testability, or operational safety. Any dependency or framework decision MUST be justified by the requirements of the feature and the constraints of the platform.

## Specification-Driven Development

Every feature MUST have documented requirements and acceptance criteria before implementation begins. Material ambiguities MUST be resolved before planning or design work proceeds. The project MUST treat requirements as a governing artifact that bounds scope, informs prioritization, and provides a basis for testing.

The team MUST reject implementation work that begins without a clear description of user value, expected behavior, and measurable outcomes. If a requirement is not sufficiently specific, it MUST be clarified before code is written.

## Governance

This Constitution governs project decisions and supersedes informal preferences or ad hoc implementation shortcuts. Amendments require clear rationale, versioning, and explicit review against these principles before they are adopted. Any change that alters architecture, safety, testing, or governance expectations MUST be documented and reviewed before release.

The project MUST review compliance with this Constitution during planning, implementation review, and release readiness checks. If a proposal conflicts with this document, the conflict MUST be resolved by either narrowing the proposal or amending the Constitution through the formal change process.

**Version**: 1.0.0 | **Ratified**: 2026-10-01 | **Last Amended**: 2026-10-01
