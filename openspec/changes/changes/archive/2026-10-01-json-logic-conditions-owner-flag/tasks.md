## 1. Squire flag definition

- [x] 1.1 Add a `workflows_json_logic_conditions` boolean `configuration_definition` (default `false`, `dev_only` category) to `config/squire/workflows.yml`, under the same `workflows_settings` group as `enhanced_object_tracking` and `enable_datetime_anchors`; verify by loading the Squire config (e.g. `rake squire:validate` or the app boot) without errors and confirming the new id appears in `configuration_definitions`.

## 2. Owner service method

- [x] 2.1 Implement `WorkflowsIntegration::OwnerService#json_logic_conditions_enabled?` returning `provider.feature_enabled?('workflows_json_logic_conditions') || false`, placed alongside `enhanced_object_tracking_enabled?`/`datetime_anchors_enabled?` in `app/workflows/workflows_integration/owner_service.rb`.
- [x] 2.2 Add a spec to `spec/workflows/workflows_integration/owner_service_spec.rb` asserting `#json_logic_conditions_enabled?` delegates to `provider.feature_enabled?('workflows_json_logic_conditions')`, mirroring the existing `#enhanced_object_tracking_enabled?` test; verify with `bundle exec rspec spec/workflows/workflows_integration/owner_service_spec.rb`.
- [x] 2.3 Run the full `owner_service_spec.rb` file and confirm the shared example `it_behaves_like 'Workflows::OwnerService'` still passes with the new method present.

## 3. Gem pin (blocked on upstream release)

- [x] 3.1 Once a released `workflows` gem version containing the JSONLogic gating (`Workflows::WorkflowHandler#json_logic_conditions_enabled?`, `Workflows.json_logic_conditions_enabled_for_provider?`, the `conditions` column, `Conditionable#new_conditions_met?`) is available, replace the local `path:` override in `Gemfile.common` with that version pin and remove the now-stale commented-out `gem 'workflows', '7.12.0'` line; verify with `bundle install` succeeding and `bundle exec rspec spec/lib/workflows` (or the equivalent gem-integration specs) passing against the pinned version.
- [x] 3.2 Regenerate `Gemfile.lock` and, per `BUNDLE_GEMFILE=Gemfile.next`, `Gemfile.next.lock`; verify both lockfiles reference the same released `workflows` version and `bundle exec rspec` runs clean under both Gemfiles in CI.

## 4. Verification

- [x] 4.1 Manually confirm (e.g. in a Rails console against a seeded provider) that toggling the provider's `workflows_json_logic_conditions` Squire flag changes the return value of `WorkflowsIntegration::OwnerService.new(provider.code).json_logic_conditions_enabled?` immediately, with no restart — verify by calling the method before and after toggling the flag in the same process.
- [x] 4.2 Confirm a provider with no explicit value set for the flag returns `false` from `#json_logic_conditions_enabled?`, matching the spec's "not configured" scenario.
