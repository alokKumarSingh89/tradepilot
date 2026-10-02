# Feature Specification: Straddle Strategy Configuration

**Feature Branch**: `001-straddle-configuration`

**Created**: 2026-10-02

**Status**: Draft

**Input**: User description: "Create the first feature specification for a short straddle strategy configuration. Business requirements include creating a strategy with unique name, underlying instrument, expiry, strike price, CE/PE entry premiums, quantity, default and configurable stop-loss/take-profit thresholds, calculated combined premium and exit thresholds, review before saving, validation messages, listing saved configurations, and no actual order placement during save. Scope excludes broker integration, live market data, order execution, automatic entries/exits, authentication, and dashboard-only UI beyond configuration review."

## Clarifications

### Session 2026-10-02

- Q: How is combined entry premium calculated? A: CE + PE, assuming equal positive quantities.
- Q: Can users edit configurations? A: Yes, saved configurations can be edited, but no live positions are affected.
- Q: How should the combined-premium stop-loss and take-profit thresholds be calculated from the entry premium when percentages are entered? → A: Use the combined entry premium as the baseline; thresholds are fixed premium values relative to entry, where stop-loss equals combined entry premium × (1 + stop-loss percentage) and take-profit equals combined entry premium × (1 - take-profit percentage).
- Q: How should the system treat equal leg quantities for a short straddle configuration? → A: Require positive whole-number quantities for each leg so the strategy defines a clear and non-fractional position size.

## User Scenarios & Testing _(mandatory)_

### User Story 1 - Create and validate a short straddle strategy (Priority: P1)

A user who wants to configure a short straddle can enter the strategy details, review the calculated risk thresholds, and save the configuration only after the system validates the inputs.

**Why this priority**: This is the core product capability. If the user cannot create and validate a valid configuration, the feature does not provide business value.

**Independent Test**: A user can enter a valid name, instrument, expiry, strike, premium values, and quantities, review the calculated thresholds, and successfully save the strategy without placing any order.

**Acceptance Scenarios**:

1. **Given** a user is creating a new strategy, **When** they provide a unique strategy name and valid premium, quantity, and expiry values, **Then** the system accepts the configuration and shows the calculated combined premium and exit thresholds.
2. **Given** a user enters invalid data such as missing required fields, a duplicate strategy name, or non-positive premium or quantity values, **When** they try to save, **Then** the system rejects the entry and shows a clear validation message without saving the record.

---

### User Story 2 - Review risk thresholds before saving (Priority: P1)

A user can see the combined entry premium and the stop-loss and take-profit thresholds derived from their configured values before confirming the strategy.

**Why this priority**: This ensures the user understands the risk profile before the configuration is saved and supports transparent decision making in a trading workflow.

**Independent Test**: A user can configure a valid straddle with CE and PE premiums, review the calculated threshold values, and decide whether to save or correct the inputs.

**Acceptance Scenarios**:

1. **Given** a valid short straddle configuration, **When** the user reviews the strategy before save, **Then** the system shows the combined entry premium and the stop-loss and take-profit exit thresholds based on the configured percentages.
2. **Given** the user changes the stop-loss or take-profit percentages, **When** they review the strategy again, **Then** the system recalculates the thresholds immediately and displays the updated values.

---

### User Story 3 - View and edit saved strategy configurations (Priority: P2)

A user can see previously saved strategy configurations so they can review, compare, and manage recurrent strategies, and can update a saved configuration without affecting any live positions.

**Why this priority**: Saved configurations are useful for reuse and review, but they are secondary to valid strategy creation and risk checks. Editing support matters because users may refine a saved setup before reuse without creating trading activity.

**Independent Test**: A user can save multiple valid configurations, review them, edit one of them, and confirm that the change updates the saved configuration while leaving live positions unaffected.

**Acceptance Scenarios**:

1. **Given** the user has previously saved one or more valid strategies, **When** they view their strategy list, **Then** they see the saved configurations and their key attributes.
2. **Given** a user is reviewing a saved strategy, **When** they edit the configuration values and save the changes, **Then** the saved record updates and no live position or market order is affected.
3. **Given** a user is reviewing saved strategies, **When** they inspect a configuration, **Then** they can confirm the original name, premium values, strike, expiry, and configured risk thresholds without any order execution being triggered.

---

### Edge Cases

- What happens when a user enters a strategy name that already exists for the same user?
- What happens when a required field is left blank or a value is zero, negative, or malformed?
- What happens when stop-loss or take-profit percentages are edited to values outside a reasonable range?
- What happens when the user tries to save a valid configuration but the system has not yet calculated the thresholds?
- What happens when a user attempts to save more than one configuration with the same name but different expiry or strike details?
- What happens when the strategy is saved with valid data but the user expects an order to be placed?

## Requirements _(mandatory)_

### Functional Requirements

- **FR-001**: The system MUST allow a user to create a short straddle strategy configuration.
- **FR-002**: The system MUST require each saved strategy to have a unique name within the user's strategy list.
- **FR-003**: The system MUST allow the user to configure the underlying instrument, expiry, strike price, CE entry premium, PE entry premium, and equal positive whole-number leg quantities.
- **FR-004**: The system MUST calculate the combined entry premium as the sum of the CE entry premium and the PE entry premium, assuming equal positive quantities for both legs.
- **FR-005**: The system MUST allow the user to configure stop-loss and take-profit percentages for the combined premium.
- **FR-006**: The system MUST default the combined stop-loss threshold to 100% of the combined entry premium, calculated as combined entry premium × (1 + 100%).
- **FR-007**: The system MUST default the combined take-profit threshold to 50% of the combined entry premium, calculated as combined entry premium × (1 - 50%).
- **FR-008**: The system MUST calculate the combined-premium exit thresholds from the configured percentages and the combined entry premium using the same formula: threshold = combined entry premium × (1 ± configured percentage).
- **FR-009**: The system MUST display the calculated thresholds before the user saves the strategy configuration.
- **FR-010**: The system MUST reject invalid configuration values with clear validation messages that explain what is wrong and how to correct it.
- **FR-011**: The system MUST allow a user to view previously saved strategy configurations.
- **FR-012**: The system MUST allow a user to edit a saved strategy configuration without affecting any live positions or placing market orders.
- **FR-013**: Saving a valid strategy configuration MUST NOT place any actual market orders.
- **FR-014**: The system MUST preserve strategy information in a form suitable for later review and reuse without executing trades.

### Key Entities _(include if feature involves data)_

- **Strategy Configuration**: Represents a saved short straddle setup for a user, including the strategy name, instrument, expiry, strike price, CE premium, PE premium, quantity, and configured risk thresholds.
- **Instrument**: Represents the underlying security or contract on which the straddle is based, including its identity and expiry context.
- **Risk Thresholds**: Represents the calculated combined-premium stop-loss and take-profit values derived from the configured percentages and entry premium.
- **User**: Represents the account owner who creates and manages a set of strategy configurations.

## Success Criteria _(mandatory)_

### Measurable Outcomes

- **SC-001**: Users can create a valid short straddle configuration in under 2 minutes from a blank starting state.
- **SC-002**: At least 95% of valid configurations are accepted on the first save attempt with no correction needed.
- **SC-003**: 100% of invalid configurations are blocked from saving and produce a clear validation message explaining the issue.
- **SC-004**: The system shows the combined entry premium and the stop-loss and take-profit thresholds before the user finalizes the save action.
- **SC-005**: Users can access saved configurations and verify the values they previously entered without triggering any order activity.
- **SC-006**: Users can edit a saved configuration and confirm the updated record is retained without affecting any live positions or placing an order.
- **SC-007**: The feature delivers a clear and reviewable configuration workflow with no live order placement during save, review, or editing.

## Assumptions

- Strategy names are unique within a single user account and are treated as case-insensitive identifiers for the purpose of duplicate detection.
- Required values such as instrument, expiry, strike price, CE premium, PE premium, and quantity are mandatory for a valid strategy configuration.
- Premium values are expected to be positive numeric values; zero and negative values are invalid unless explicit business approval is later provided. Quantity values are expected to be positive whole numbers; zero, negative values, and fractional lots are invalid unless explicit business approval is later provided.
- Stop-loss and take-profit percentages are entered as percentage values, not decimal fractions, and are validated as positive values within a reasonable operational range. A 100% stop-loss means the threshold is two times the combined entry premium, and a 50% take-profit means the threshold is half the combined entry premium.
- The default thresholds are derived from the combined premium only, and the product assumes these default values remain the same unless changed by the user.
- This feature does not include broker connectivity, live pricing, or execution events, so all review and save behavior is limited to validation, calculation, and persistence.
- The system will not execute or simulate market orders during configuration save; it only stores the strategy definition.

## Notes on Missing Business Decisions

The following areas are not fully defined by the supplied requirements and are therefore treated as explicit product assumptions rather than hidden implementation choices:

- The exact allowable range for stop-loss and take-profit percentages should be confirmed with product stakeholders if values above 100% or other edge cases become operationally important.
- The product requires a user-level uniqueness rule for strategy names, but the exact behavior for duplicate names across different users is already defined as per-user uniqueness.
- The product decision is to require positive whole-number leg quantities for a short straddle, which keeps the strategy definition consistent and prevents fractional-lot ambiguity during configuration review.
