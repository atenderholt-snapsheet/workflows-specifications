## Purpose

Defines how vice-backend supplies default entry conditions to the workflow engine and filters selectable workflow attributes, actions, and events using JSONLogic conditions, gated per provider by the JSON Logic conditions flag.

## Requirements

### Requirement: Default entry condition is a JSONLogic condition
The owner service SHALL provide a default entry condition, as a JSONLogic condition, for a provider whose first allowed claim type is `auto` or `property`: an `and` group containing an `intersects` on `claim.claim_type` with that claim type. It SHALL provide no default entry condition otherwise. The default entry expression SHALL remain available unchanged for providers with the flag off.

#### Scenario: Auto default claim type
- **WHEN** the provider's first allowed claim type is `auto`
- **THEN** the default entry condition is an `and` group with an `intersects` on `claim.claim_type` with the value `["auto"]`

#### Scenario: Property default claim type
- **WHEN** the provider's first allowed claim type is `property`
- **THEN** the default entry condition is an `and` group with an `intersects` on `claim.claim_type` with the value `["property"]`

#### Scenario: No default claim type
- **WHEN** the provider has no allowed claim types, or the first is neither `auto` nor `property`
- **THEN** the default entry condition is `nil`

#### Scenario: Rendered by the engine
- **WHEN** the engine's default-entry-condition endpoint is requested for an owner with a default entry condition
- **THEN** the endpoint renders the condition returned by the owner service

#### Scenario: Legacy default entry expression unchanged
- **WHEN** the default entry expression is requested
- **THEN** it returns the same Expression structure as before this change

### Requirement: Entry-condition builder produces JSONLogic only
The entry-condition helper SHALL build JSONLogic conditions as an `and` group: requiring claim types SHALL produce an `intersects` on `claim.claim_type`, and excluding claim types SHALL produce a negation (`!`) of that `intersects`. It SHALL NOT construct Expression structures and SHALL NOT itself decide whether a candidate satisfies a condition.

#### Scenario: Requires claim types
- **WHEN** a condition is built requiring claim types `auto` and `property`
- **THEN** the result is an `and` group with an `intersects` on `claim.claim_type` with those values

#### Scenario: Excludes claim types
- **WHEN** a condition is built excluding claim type `liability`
- **THEN** the result is an `and` group with a `!` of an `intersects` on `claim.claim_type` with that value

### Requirement: Definitions accept both entry expression and conditions
Attribute, action, and event definitions SHALL accept `conditions:` (JSONLogic) in `visible?` and `options` alongside the legacy `entry_expression:`, each optional, and SHALL pass both through when delegating to their superclass or concerns.

#### Scenario: Caller passes both
- **WHEN** the engine calls `visible?` with both `entry_expression:` and `conditions:`
- **THEN** the call succeeds and both values reach the claim-type visibility check

#### Scenario: Caller passes neither
- **WHEN** `visible?` is called with neither `entry_expression:` nor `conditions:`
- **THEN** the call succeeds and the definition is visible subject to its provider feature-flag gates

### Requirement: Visibility filtering follows the provider's JSON Logic conditions flag
For a provider with the `workflows_json_logic_conditions` flag enabled, claim-type-restricted definitions SHALL be visible only when the engine's `Workflows::Conditions.satisfiable_with?` reports that the given `conditions` can be satisfied by the definition's claim types. For a provider with the flag disabled or unset, or when no `conditions` is given, filtering SHALL use `entry_expression` exactly as before this change. Results with the flag on SHALL equal results with the flag off for the same logical entry criteria.

#### Scenario: Flag on, compatible conditions
- **WHEN** the flag is on, a definition is restricted to `auto`, and `conditions` requires `auto`
- **THEN** the definition is visible

#### Scenario: Flag on, incompatible conditions
- **WHEN** the flag is on, a definition is restricted to `auto`, and `conditions` requires `property`
- **THEN** the definition is not visible

#### Scenario: Flag on, no conditions
- **WHEN** the flag is on and `conditions` is `nil`
- **THEN** a claim-type-restricted definition is visible

#### Scenario: Flag off uses entry expression
- **WHEN** the flag is off and an `entry_expression` is given
- **THEN** visibility is decided from the `entry_expression`, ignoring any `conditions`

#### Scenario: Definition has no claim-type restriction
- **WHEN** a definition declares no visible claim types
- **THEN** it is visible regardless of flag and conditions

#### Scenario: Provider feature-flag gating is preserved
- **WHEN** a definition's own provider feature flag is off
- **THEN** it is not visible regardless of the conditions flag or conditions

#### Scenario: Flag scoped per provider
- **WHEN** provider A has the flag on and provider B has it off
- **THEN** filtering for A uses `conditions` and filtering for B uses `entry_expression`
