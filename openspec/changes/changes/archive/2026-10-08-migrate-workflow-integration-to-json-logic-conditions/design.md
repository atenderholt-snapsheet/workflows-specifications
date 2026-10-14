## Context

See proposal.md. `OwnerService#default_entry_expression` builds `Workflows::CompoundExpression`/`LogicalExpression`; `Helpers::EntryExpressionHelper` builds an Expression via `ExpressionBuilder`; `ClaimTypeVisibilityConcern.visible?` calls `allows_expression?`. 56 files under `app/workflows` declare `self.visible?(entry_expression:, locale:, provider_code:, **)`, most forwarding via `super(entry_expression: ..., ...)`; 54 spec files call it with `entry_expression:`. `AnchoredTaskEffectiveDateConcern` calls `visible?(entry_expression: nil, ...)`.

Engine facts (workflow-engine PR 768, WRK-1802):
- `visible?`/`options` receive `entry_expression:` AND `conditions:` side by side during a migration window.
- `Workflows::Conditions.satisfiable_with?(condition_a, condition_b, locale:, provider_code:)` compares two JSONLogic rules; it raises if one is an `and`/`or` group and the other a single condition.
- JSONLogic shapes the engine emits: IN -> `{"intersects" => [{"val" => "claim.claim_type"}, [..]]}`; NOT_IN -> `{"!" => [intersects]}` (no `not_intersects` operator); compounds -> `{"and"|"or" => [...]}`.
- The per-provider flag `workflows_json_logic_conditions` is already exposed via `OwnerService#json_logic_conditions_enabled?` (spec `json-logic-conditions-flag`).

## Goals / Non-Goals

**Goals:**
- Produce and consume JSONLogic `conditions` behind the existing per-provider flag.
- Keep the allow/deny decision in the engine, never reimplemented in vice-backend. The engine primitive is chosen by the provider's `workflows_json_logic_conditions` flag: `Workflows::Conditions.satisfiable_with?` on `conditions` when the flag is on, and the legacy `allows_expression?` on `entry_expression` when it is off.
- Flag off = zero behavior change; flag on = same visibility results.

**Non-Goals:**
- Removing `entry_expression`/Expression code (follow-up once all providers are cut over).
- Changing the backfill task or the flag mechanism.
- Changing which definitions are claim-type restricted.

## Decisions

1. **Additive keyword, not a rename.** Every `visible?` signature becomes `(entry_expression:, conditions:, locale:, provider_code:, **)` with both keywords required, and `super` calls forward both. Required (not defaulted) on purpose: a call site or `super` that forgets one raises `ArgumentError` instead of silently treating `conditions` as `nil` (which means "visible"). Callers that only had `entry_expression:` (e.g. `AnchoredTaskEffectiveDateConcern`, specs) pass `conditions: nil`; the follow-up cleanup removes `entry_expression:` everywhere. Required by the engine's side-by-side calls; alternative (rename to `conditions:` only) rejected because the engine still passes `entry_expression:` and flag-off providers need it. Mechanical scripted edit across the 56 files, reviewed by diff.
2. **Gate in the shared concern.** `ClaimTypeVisibilityConcern.visible?` checks `Provider.find_by!(code: provider_code).feature_enabled?('workflows_json_logic_conditions')` (same lookup the owner service uses): flag on and `conditions` present -> `Workflows::Conditions.satisfiable_with?(helper_condition, conditions, locale:, provider_code:)`; otherwise the existing `entry_expression` path. Nil short-circuit preserved. Other overrides stay flag-agnostic and just forward. Alternative (gate in each class) rejected: 56 copies.
3. **Separate JSONLogic builder class.** New `Helpers::EntryConditionHelper` (own state, no dependency on the Expression hash internals) returns `{"and" => [clause, ...]}` with `requires_claim_types` -> `intersects` on `{"val" => "claim.claim_type"}` and `excludes_claim_types` -> `{"!" => [intersects]}`. The `and` wrapper is required so both sides of `satisfiable_with?` are the same shape. `Helpers::EntryExpressionHelper` is left untouched for the flag-off path, so the legacy code can be deleted wholesale later. Alternative (add `#condition` to the existing helper) rejected: it would derive from the legacy `child_expressions` structure that goes away.
4. **Owner service.** Add `default_entry_condition` reusing the auto/property selection and the helper; keep `default_entry_expression`. The engine chooses which to render.
5. **Ordering.** Bump `workflows` in both Gemfile/lock pairs first.

## Risks / Trade-offs

- [`satisfiable_with?` raises on shape mismatch or unsupported operators] -> `conditions` from the UI is an `and` group of IN/NOT_IN leaves; cover compound/non-compound and unsupported shapes in specs, and decide whether to rescue-and-fall-back to the legacy path or surface the error.
- [Flag lookup per `visible?` call adds a Provider query] -> reuse the memoization/caching `Provider#feature_enabled?` already has; verify no N+1 across large definition lists.
- [Engine call sites that pass only `entry_expression:` (anchor_attribute, v2 action definition show, v2 relationship endpoints) would raise `ArgumentError` against required `conditions:`] -> prerequisite engine change to pass `conditions: nil` there, verified in tasks 5.1/5.4; this change cannot ship before that engine release.
- [Missing one of 56 files raises ArgumentError on `conditions:`] -> the signatures use `**`, but explicit forwarding must be checked; grep task plus existing per-definition specs.
- [Parity drift between paths] -> parity specs run the same cases flag-on and flag-off.
- [Dual-Gemfile drift] -> update both lock files, run both suites.
- [Two code paths linger] -> tracked as an explicit follow-up change to delete the legacy path.

## Migration Plan

1. Bump gem. 2. Deploy with flag off everywhere (no behavior change). 3. Enable `workflows_json_logic_conditions` per provider after backfill. 4. Rollback: flip the flag off, no deploy. 5. Follow-up change removes `entry_expression` and the Expression helper.

## Open Questions

None blocking. Resolved by task 1.1 (engine checked at local checkout, PR 768 merged to `release` as `4b8eaee0`, still under CHANGELOG "Unreleased", `VERSION` 7.13.2):

- `satisfiable_with?(condition_a, condition_b, locale:, provider_code:)` and the emitted shapes match "Context". `conditions:` reaches `visible?`/`options` as a parsed Hash with string keys (base64url JSON decoded by the engine), and fields are `{"val" => "claim.claim_type"}`. Base `visible?(**)`/`options(**)` accept `conditions:`.
- The engine currently exposes only `GET /workflows/v1/expressions/default_entry_expression`, which calls the owner service's `default_entry_expression`. No default-entry-condition endpoint exists in PR 768, and nothing in the engine calls `default_entry_condition`. Decision 4 therefore only provides the owner-service method; rendering it needs a separate engine change (flagged to the team, tracked outside this change). Until it ships, the flag-on default entry condition cannot be verified end to end (task 4.2).
