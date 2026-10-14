## Why

The `workflows` engine gem already gates all JSONLogic-vs-legacy-expression condition evaluation in `Workflows::WorkflowHandler` behind `Workflows.json_logic_conditions_enabled_for_provider?(provider_code)`, which delegates to the host app's registered owner-service class. In vice-backend, `WorkflowsIntegration::OwnerService` — the class registered via `config.register_owner_service_class` — has no `json_logic_conditions_enabled?` method, so the engine's `provider_feature_enabled?` check finds no matching method and always evaluates to `false`. Every provider is silently stuck on the legacy expression path with no way to opt in, even after the engine-side JSONLogic work ships. We need the vice-backend-side flag wired up per-provider so a provider can be switched to JSONLogic evaluation, and rolled back, without a deploy.

## What Changes

- Add `WorkflowsIntegration::OwnerService#json_logic_conditions_enabled?`, following the existing `enhanced_object_tracking_enabled?` / `datetime_anchors_enabled?` pattern: delegates to `provider.feature_enabled?('workflows_json_logic_conditions')`, defaulting to `false` when the Squire flag is absent or the provider lookup fails to return a truthy value.
- Add a new Squire configuration definition `workflows_json_logic_conditions` (boolean, default `false`, `dev_only` category to start) to `config/squire/workflows.yml`, under the same `Workflows Settings` group as the other engine-integration flags.
- Extend `spec/workflows/workflows_integration/owner_service_spec.rb` with a test proving `#json_logic_conditions_enabled?` delegates to `provider.feature_enabled?('workflows_json_logic_conditions')`, matching the existing `#enhanced_object_tracking_enabled?` / `#datetime_anchors_enabled?` test style.
- Bump the `workflows` gem dependency in `Gemfile.common` from the local `path:` override (currently pointing at a `jsonlogic_conditions` branch checkout) to the released version once the engine team ships it, and regenerate `Gemfile.lock` / `Gemfile.next.lock`.

This change does **not** touch `WorkflowHandler`, JSONLogic evaluation, the `conditions` column, `Workflows.trigger`, or any condition-evaluation dispatch logic — all of that already exists in the `workflows` gem (see its own archived OpenSpec change `2026-09-23-jsonlogic-conditions`) and is out of scope here. It also does not change how conditions are written (the config/API write path stays engine-owned until the API ticket lands), and does not itself enable the flag for any provider — enabling requires operator parity and backfill to be complete first, which is an operational/rollout step tracked separately, not part of this change's tasks.

## Capabilities

### New Capabilities
- `workflows-integration/json-logic-conditions-flag`: per-provider Squire feature flag, surfaced through `WorkflowsIntegration::OwnerService#json_logic_conditions_enabled?`, that controls whether the `workflows` engine evaluates JSONLogic conditions or the legacy expression tree for a given provider.

### Modified Capabilities
(none — no existing capability specs in this repo yet)

## Impact

- **Code:** `app/workflows/workflows_integration/owner_service.rb` (new method), `config/squire/workflows.yml` (new flag definition), `spec/workflows/workflows_integration/owner_service_spec.rb` (new test).
- **Dependencies:** `Gemfile.common`, `Gemfile.lock`, `Gemfile.next.lock` — pin bump from local `path:` override to a released `workflows` gem version that includes the JSONLogic gating (`Workflows::WorkflowHandler#json_logic_conditions_enabled?`, `Workflows.json_logic_conditions_enabled_for_provider?`, the `conditions` column, and `Conditionable#new_conditions_met?`).
- **Systems:** Squire (feature flag storage/UI), the `workflows` engine's condition-evaluation path (consumer of this flag, not modified here). No API, controller, or serializer changes.
- **Rollout:** enabling the flag for a given provider is an operational step gated on operator parity being complete and that provider's `conditions` data being backfilled — this change only makes the flag available to set, it does not perform or schedule any provider's backfill or enablement.
