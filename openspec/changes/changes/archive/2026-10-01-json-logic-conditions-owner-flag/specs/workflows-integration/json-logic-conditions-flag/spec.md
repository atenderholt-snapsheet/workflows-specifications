## Purpose

Lets the workflows engine ask, per provider, whether that provider should evaluate JSON Logic conditions instead of the legacy expression tree, so the rollout can proceed one provider at a time and be reversed instantly by an operator.

## ADDED Requirements

### Requirement: Provider-scoped JSON Logic conditions flag
The owner service SHALL report, for a given provider, whether JSON Logic condition evaluation is enabled for that provider, based on a Squire feature flag scoped to that provider.

#### Scenario: Flag enabled for a provider
- **WHEN** the workflows engine asks whether JSON Logic conditions are enabled for a provider whose Squire feature flag for this capability is turned on
- **THEN** the owner service reports `true`

#### Scenario: Flag disabled for a provider
- **WHEN** the workflows engine asks whether JSON Logic conditions are enabled for a provider whose Squire feature flag for this capability is turned off
- **THEN** the owner service reports `false`

#### Scenario: Flag not configured for a provider
- **WHEN** the workflows engine asks whether JSON Logic conditions are enabled for a provider that has no explicit value set for this capability's Squire feature flag
- **THEN** the owner service reports `false`

### Requirement: Enabling or disabling requires no deploy
Changing whether JSON Logic conditions are enabled for a provider SHALL be an operator action against the provider's Squire feature flag configuration, taking effect for that provider's subsequent condition evaluations without requiring an application deploy or restart.

#### Scenario: Operator flips the flag for a provider
- **WHEN** an operator changes a provider's Squire feature flag value for this capability
- **THEN** the next time the workflows engine asks whether JSON Logic conditions are enabled for that provider, it reports the newly configured value

### Requirement: Flag scope is isolated per provider
The flag value for one provider SHALL NOT affect the flag value reported for any other provider.

#### Scenario: Two providers with different flag values
- **WHEN** provider A has JSON Logic conditions enabled and provider B does not
- **THEN** the owner service reports `true` for provider A and `false` for provider B
