## Why

The workflow engine now accepts and exposes JSONLogic `conditions` through its config API (workflow-engine PR 768, WRK-1802) and passes `conditions:` to `visible?`/`options`, alongside the legacy `entry_expression:` during a migration window. vice-backend must produce and consume JSONLogic conditions, or the config API/UI integration breaks for providers cut over to conditions. The cut-over is per provider, so the new behavior must sit behind the existing `workflows_json_logic_conditions` feature flag.

## What Changes

- `WorkflowsIntegration::OwnerService` gains `#default_entry_condition`, returning a JSONLogic condition (an `and` group holding an `intersects` on `claim.claim_type`) that the engine renders through its default-entry-condition endpoint. `#default_entry_expression` is kept while the flag is off. Both return `nil` when there is no default claim type.
- A new `Helpers::EntryConditionHelper` is a pure JSONLogic builder: `requires_claim_types` -> `intersects`, `excludes_claim_types` -> `!` wrapping `intersects` (the engine has no `not_intersects` operator). `Helpers::EntryExpressionHelper` is left untouched for the flag-off path and is deleted with the other legacy code in the follow-up.
- Visibility filtering: every attribute / action / event definition class and concern that declares or forwards `visible?` / `options` requires `conditions:` alongside `entry_expression:` (both required, so a missed call site fails loudly) and forwards both through `super` (56 files under `app/`).
- `ClaimTypeVisibilityConcern.visible?` is gated per provider by `workflows_json_logic_conditions`:
  - flag on: decides via the engine primitive `Workflows::Conditions.satisfiable_with?` on `conditions`;
  - flag off (or no `conditions`): existing `entry_expression` / `allows_expression?` behavior, unchanged.
- Specs updated and extended for flag-on and flag-off paths.
- Gem: bump `workflows` in `Gemfile`/`Gemfile.lock` and `Gemfile.next`/`Gemfile.next.lock` to the release containing PR 768.
- **BREAKING** (internal contract): requires the engine release with `conditions:` support; with the flag off nothing changes for providers.

Out of scope: removing `entry_expression` and the Expression structures (follow-up once all providers are on JSONLogic), the `workflows:backfill_json_logic_conditions` task, and the flag mechanism itself (already specced in `json-logic-conditions-flag`).

## Capabilities

### New Capabilities
- `workflows-integration/entry-conditions`: how vice-backend supplies default entry conditions and filters selectable attributes, actions, and events against a workflow's entry condition using JSONLogic, gated per provider by the JSON Logic conditions flag.

### Modified Capabilities
<!-- None: json-logic-conditions-flag and conditions-backfill are unchanged; this capability consumes the flag. -->

## Impact

- Code: `app/workflows/workflows_integration/owner_service.rb`, `app/workflows/helpers/entry_expression_helper.rb`, `app/workflows/concerns/claim_type_visibility_concern.rb` and the other visibility concerns, ~56 definition files under `app/workflows/{actions,events,attributes}`.
- Specs: `spec/workflows/**` (54 files) and shared `Workflows::OwnerService` examples.
- Dependencies: `workflows` gem (currently 7.13.2) must include PR 768; both Gemfile lock pairs stay consistent.
- Rollout: per provider via the `workflows_json_logic_conditions` Squire flag; rollback is flipping it off, no deploy.
