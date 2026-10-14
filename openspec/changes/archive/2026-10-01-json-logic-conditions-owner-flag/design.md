## Context

See [proposal.md](proposal.md) - Why. The `workflows` engine already calls `Workflows.json_logic_conditions_enabled_for_provider?(provider_code)`, which looks up `Workflows.configuration.owner_service_class` (registered in vice-backend as `WorkflowsIntegration::OwnerService`) and calls `<owner_service>.json_logic_conditions_enabled?` if the method exists, defaulting to `false` otherwise (`Workflows.provider_feature_enabled?`, private, in the gem). Every other per-provider engine flag in this codebase (`enhanced_object_tracking_enabled?`, `webhook_v2_payload_enabled?`, `datetime_anchors_enabled?`) follows the same shape: a public predicate method on `WorkflowsIntegration::OwnerService` that delegates to `provider.feature_enabled?('<squire_id>')`, backed by a boolean `configuration_definition` in `config/squire/workflows.yml`.

The engine-side JSONLogic gating is currently only available via a local `path:` gem override (`Gemfile.common:118`) pointing at the `workflows` repo's `jsonlogic_conditions` branch; the real version pin (`workflows`, `7.12.0`) is commented out directly above it. That branch is not yet merged/released.

## Goals / Non-Goals

**Goals:**
- Give the engine a real answer (not the "method missing → false" default) when it asks whether a provider has JSON Logic conditions enabled.
- Keep the flag's default `false` for every provider until explicitly turned on, matching how `enhanced_object_tracking` and `enable_datetime_anchors` were introduced.
- Sequence the Gemfile pin bump so this repo only depends on a released `workflows` version, not the local path override.

**Non-Goals:**
- Implementing or changing JSON Logic evaluation, `WorkflowHandler` dispatch, the `conditions` column, or `Workflows.trigger` — all engine-side and already built.
- Building any UI, admin tooling, or automation for turning the flag on/off per provider beyond what Squire already provides.
- Performing or scheduling the operator-parity verification or the per-provider `conditions` backfill that must happen before a given provider's flag is turned on — those are prerequisites for enabling the flag, tracked outside this change.
- Changing the config/API write path for conditions.

## Decisions

**Flag name: `workflows_json_logic_conditions`.** Matches the existing naming convention in `config/squire/workflows.yml` (`enhanced_object_tracking`, `enable_datetime_anchors`, `enable_date_and_datetime_conditions`) — short, `workflows`-domain-scoped, boolean-shaped id. Alternative considered: mirroring the engine's own method name literally (`json_logic_conditions_enabled`) — rejected because every existing Squire id in this file drops the `_enabled` suffix (the predicate method adds `_enabled?`, the Squire id doesn't).

**Squire category: `dev_only` at launch.** Every other engine-integration flag in `config/squire/workflows.yml` that gates internal evaluation behavior (as opposed to a customer-visible UI toggle) uses `dev_only`. This flag is flipped by the workflows/platform team per provider during rollout, not by carrier admins, so it should not be exposed in a customer-facing configuration UI. Default value `false`, consistent with a flag that must not silently change behavior for any provider until parity + backfill are confirmed.

**Default-false via absence, not an explicit fallback in the new method.** `provider.feature_enabled?('workflows_json_logic_conditions') || false` mirrors `enhanced_object_tracking_enabled?`/`datetime_anchors_enabled?` exactly (`feature_enabled?` can return `nil`; `|| false` normalizes it to a boolean). No new fallback logic is introduced beyond that existing pattern.

**Gem pin bump is sequenced, not bundled into the same commit as the OwnerService change.** The `WorkflowsIntegration::OwnerService#json_logic_conditions_enabled?` method itself has no compile-time dependency on the new gem version — it's just a predicate method the engine calls if present. Landing it against the current pinned `workflows` release (once one exists with the JSONLogic gating merged) avoids shipping a change that depends on an uncommitted local path override. Alternative considered: bump the Gemfile pin in the same PR — rejected because the release isn't cut yet as of this proposal, and coupling the two would block this small, low-risk change on the engine team's release timeline.

## Risks / Trade-offs

- **[Risk]** The `workflows` gem release containing the JSONLogic gating may not exist yet when this change is ready to implement, leaving the Gemfile pin bump task blocked. → **Mitigation**: land the `OwnerService`/Squire/spec changes independently of the pin bump; the predicate method is harmless (unused) against the currently-released gem version since no code path calls it until the engine's `json_logic_conditions_enabled_for_provider?` ships. Track the pin bump as a separate task that can complete once a release exists.
- **[Risk]** A provider's flag gets turned on before operator parity or backfill is actually complete, causing JSON Logic and legacy evaluation to diverge for that provider's live traffic. → **Mitigation**: this is an operational rollout risk, not something this change's code can enforce; call it out explicitly in the proposal's rollout note and leave gating the actual per-provider enablement to the runbook/process the workflows team owns, not to code in this change.
- **[Trade-off]** Using `dev_only` means enabling the flag for a provider requires whatever internal tooling toggles `dev_only` Squire flags today, rather than a carrier-facing UI. Acceptable because the intended operator is the platform/workflows team performing a supervised rollout, not carrier admins.

## Migration Plan

1. Add the `workflows_json_logic_conditions` Squire flag definition (default `false`) to `config/squire/workflows.yml`.
2. Implement `WorkflowsIntegration::OwnerService#json_logic_conditions_enabled?` and its spec.
3. Once the `workflows` gem ships a release including the JSONLogic gating, bump `Gemfile.common` from the local `path:` override to that version, regenerate `Gemfile.lock` and `Gemfile.next.lock`, and remove the commented-out old pin line.
4. No data migration or backfill is part of this change — per-provider `conditions` backfill is a separate, provider-specific operational step gated by the process described in the proposal's rollout note.

**Rollback:** the flag defaults to `false`, so no action is needed to roll back the flag mechanism itself. Rolling back a specific provider that was switched on is a Squire config change (flip the flag back to `false`) with no deploy required, per the spec's "no deploy" requirement.
