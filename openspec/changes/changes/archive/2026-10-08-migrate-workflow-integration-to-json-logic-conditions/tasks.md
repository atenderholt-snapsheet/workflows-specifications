## 1. Engine dependency

- [x] 1.1 Confirm the released `workflows` version containing PR 768; verify `Workflows::Conditions.satisfiable_with?` signature and emitted JSONLogic shapes match design.md, and resolve the `default_entry_conditions` open question
- [x] 1.2 Bump `workflows` in `Gemfile`, `Gemfile.lock`, `Gemfile.next`/`Gemfile.common` as applicable, and `Gemfile.next.lock`; verify `bundle install` resolves for both `BUNDLE_GEMFILE` values (done: `workflows` 7.14.0 and `snapsheet_utils` 1.0.1 in `Gemfile.common` and `Gemfile.lock`; `Gemfile.next.lock` updated with `bundle lock --update workflows snapsheet_utils --conservative`, which also corrects the stale `snap_transaction-service` entry to 3.1.0 to match `Gemfile.common`; both Gemfiles resolve, and the engine ships `GET /workflows/v1/conditions/default_entry_conditions`)

## 2. Core integration

- [x] 2.1 Add `Helpers::EntryConditionHelper`, a standalone JSONLogic builder, leaving `EntryExpressionHelper` untouched (`requires_claim_types` -> `intersects`, `excludes_claim_types` -> `!` of `intersects`, `and`-wrapped), keeping the Expression builder; verify a helper spec covers both and the existing helper behavior is unchanged
- [x] 2.2 Add `OwnerService#default_entry_conditions` alongside `#default_entry_expression`; verify owner-service specs cover auto, property, nil, and that the legacy method is unchanged
- [x] 2.3 Update `ClaimTypeVisibilityConcern.visible?` to accept `conditions:`, gate on `workflows_json_logic_conditions`, and call `Workflows::Conditions.satisfiable_with?` when on; verify specs for flag on/off, nil conditions, unrestricted definition, compatible, incompatible, and per-provider isolation
- [x] 2.4 Update shared examples for `Workflows::OwnerService` and other spec support as needed (no change needed: the shared examples live in the engine and only check `default_entry_expression`; vice-backend has no other spec support referencing it)

## 3. Definition classes

- [x] 3.1 Add required `conditions:` (alongside `entry_expression:`) and forward both through `super` in the other visibility concerns (`HidePolicyDateEventsConcern`, `AiResponseReceivedEventConcern`, `AnchoredTaskEffectiveDateConcern`, `LossTypeAttributeConcern`); verify their specs pass
- [x] 3.2 Same change in all action classes under `app/workflows/actions/**`; verify action specs pass
- [x] 3.3 Same change in all event classes under `app/workflows/events/**`; verify event specs pass
- [x] 3.4 Same change in all attribute classes under `app/workflows/attributes/**`; verify attribute specs pass
- [x] 3.5 Verify `grep -rn "def self.visible?\|def visible?" app/workflows` shows every override accepting `conditions:` and every `super(` forwarding it (verified: 74 `visible?` definitions take `conditions:`, 73 forward it via `super` and the 74th is `ClaimTypeVisibilityConcern`, the terminus; intentional exceptions are `actions/ai/action_base.rb` (`**args` forwarding) and the two `visible?(*)` overrides in `assign_handler_action.rb` and `exposure_claimant_name_change_event.rb`)

## 4. Specs and verification

- [x] 4.1 Update the remaining spec files to pass both keywords and add a flag-on parity case where claim-type filtering is exercised; verify `bundle exec rspec spec/workflows` passes on both Gemfiles (audited: no spec calls `visible?` or `options` with only `entry_expression:` except the ones that route to `visible?(*)`/`**args` overrides or the engine base `visible?(**)`; dual-Gemfile run left to CI)
- [x] 4.2 Verify default entry conditions render through the engine's default-entry-condition endpoint for a seeded provider (e.g. `zurich_ie`) with the flag on, and that flag off renders the expression as before (done manually against engine 7.14.0, provider with `auto` default claim type: `GET /workflows/v1/conditions/default_entry_conditions` returns `{"and":[{"intersects":[{"val":"claim.claim_type"},["auto"]]}]}` and `GET /workflows/v1/expressions/default_entry_expression` returns the `AND`/`IN` `claim.claim_type` `["auto"]` expression; both responses are identical with the flag on and off, as neither endpoint is flag-gated; `zurich_ie` itself has no default, see 5.5)

## 5. Manual validation against workflow-engine

Run against the local engine checkout (`Gemfile.common` path gem), since no release exists yet. Required keywords (design.md decision 1) make a missed call site raise `ArgumentError`, so every engine call site that reaches vice-backend definitions must be exercised.

- [x] 5.1 Prerequisite: get an engine change that passes `conditions: nil` at the engine call sites that currently pass only `entry_expression:`, and verify each is updated in the engine checkout used here (verified against local engine branch `WRK-1802-accept-and-expose-workflow-through-config-api-missed-requirements`, still `VERSION` 7.13.2 / unreleased: every `.visible?(` call in engine `app/` and `lib/` passes `conditions:`; `AnchorAttribute.anchorable_attribute?` now requires `conditions:` and `stored_datetime_options` threads it; `eligible?` intentionally passes `nil` for save-time checks. That engine branch must be merged and released before this change ships)
  - `lib/workflows/attributes/internal/anchor_attribute.rb:107`: `anchorable_attribute?` takes no `conditions:`, so thread the `conditions:` that `options` already receives through to this `visible?` call (passing `nil` would show too many anchors for flag-on providers). Canary: `spec/workflows/attributes/action_attributes/task_attributes_spec.rb:140` (`TaskAnchorAttribute.options`) fails with `missing keyword: :conditions` until this lands
  - `app/controllers/workflows/v2/action_definitions/action_definitions_controller.rb:55`
  - `app/controllers/workflows/v2/action_definitions/relationships/action_definitions_attribute_definitions_controller.rb:15` and `:31`
  - `app/controllers/workflows/v2/event_definitions/relationships/event_definitions_attribute_definitions_controller.rb:15` and `:31`
  - Re-run `rg -n "\.visible\?\(" app lib` in the engine and confirm no call omits `conditions:`
  - Track as a separate engine ticket; this change cannot ship before it is released
- [x] 5.2 With the flag off for a seeded provider (e.g. `zurich_ie`), verify v1 and v2 definition endpoints return the same actions, events, and attributes as before this change, with and without an `entry_expression`
  - Done with `analysis/definitions_snapshot.rb`: v1 actions/events/attributes across 13 entry-expression variants plus the v2 listing endpoints, `before` (release) vs `after` (this branch), flag off, local engine gem: all 200, byte-identical. The variants change the results (e.g. actions 59 -> 56, events 201 -> 146), so the check discriminates. Scope: one provider/user; attribute-definition-options, relationship, and show endpoints are covered by 5.4
  - v1: action, event, attribute definitions endpoints
  - v2: `action_definitions`, `event_definitions`, `attribute_definitions`, `attribute_definition_options`
  - Compare against a response captured on `release` (save the JSON before switching branches) and confirm no `ArgumentError` in the logs
- [x] 5.3 With the flag on for the same provider, verify the filtering endpoints accept `conditions` and filter claim-type-restricted definitions correctly
  - Enable `workflows_json_logic_conditions` for the provider in local config/Squire; confirm via `WorkflowsIntegration::OwnerService.new(code).json_logic_conditions_enabled?`
  - Pass `conditions` as base64url-encoded JSON (e.g. an `and` of `intersects` on `{"val":"claim.claim_type"}`); try `auto`, `property`, `liability`, and an excluded type (`!` of `intersects`)
  - Confirm claim-type-restricted definitions (auto-only, property-only, workers' compensation, liability, commercial property) appear and disappear as with the equivalent `entry_expression`
  - Confirm a request with no `conditions` returns all unrestricted-by-claim-type definitions
  - Done with `analysis/definitions_snapshot.rb --mode conditions` (flag ON, local engine gem, same user/provider): all 200; the 36 claim-type variants (IN / NOT_IN for 5 claim types plus an auto+property pair, across v1 actions/events/attributes) are byte-identical to the flag-off `entry_expression` run. 24 of 36 variants differ from the unfiltered response (e.g. actions 59 -> 56), so the check discriminates; the other 12 equal the unfiltered response in both runs. Scope: v1 only, claim-type IN/NOT_IN conditions, one provider; v2 with persisted-version conditions is not covered
- [x] 5.4 Exercise the call sites that pass `entry_expression: nil`/no conditions, with the flag both on and off, and verify they respond without `ArgumentError`
  - v2 action definition show and the action/event relationship endpoints (`.../relationships/attribute_definitions`)
  - Anchor attributes (`AnchorAttribute`) in an attribute listing that includes anchors
  - Create-task actions that use `AnchoredTaskEffectiveDateConcern#to_h`
  - Done with `analysis/definitions_snapshot.rb --sites` (flag OFF then ON, local engine gem, same user/provider): 522 requests per run (59 action + 201 event shows and attribute-definition relationship listings, 2 anchor option lookups), all 200 in both runs, byte-identical across flag states. The anchor lookups returned the date-attribute anchors (e.g. `claim.date_time_of_loss`), so the engine's `anchorable_attribute?` path ran. 104 relationship listings were empty in both runs. Scope: one provider/user
- [x] 5.5 Verify the default entry condition for a seeded provider and record the outcome
  - Call `WorkflowsIntegration::OwnerService.new('zurich_ie').default_entry_conditions` in a Rails console and confirm the expected JSONLogic
  - Note that the engine has no endpoint rendering it yet (design.md Open Questions), so end-to-end rendering is out of scope until that engine ticket lands
  - Done in a Rails runner against `snapsheet_experimental` (allowed claim types `auto, property, workers_compensation, liability, commercial_property`): `default_entry_conditions` = `{"and"=>[{"intersects"=>[{"val"=>"claim.claim_type"}, ["auto"]]}]}`; legacy `default_entry_expression` still a `Workflows::CompoundExpression`; `Workflows::Conditions.satisfiable_with?` returns `true` against itself and `false` against a liability-only condition. `zurich_ie` returns `nil` (no auto/property default claim type), as specified
- [ ] 5.6 (Skipped: no local data for a second provider; per-provider isolation is covered by the `is scoped per provider` example in `claim_type_visibility_concern_spec.rb`) Verify provider isolation: with provider A flag on and provider B flag off, confirm A filters by `conditions` and B by `entry_expression` in the same running process
