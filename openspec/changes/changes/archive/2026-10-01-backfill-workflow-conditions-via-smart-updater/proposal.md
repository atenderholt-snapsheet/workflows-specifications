## Why

The `workflows` engine's shadow write only populates the JSONLogic `conditions` column on `EventHandler` and `WorkflowDefinitionVersion` rows that are created or updated after the shadow write shipped (`Workflows::Conditionable#sync_conditions_from_expression!`, wired as an `after_save` callback). The large backlog of pre-existing rows still has `conditions = NULL`, which blocks any owner (carrier) from being cut over to JSONLogic condition evaluation — `new_conditions_met?` falls back to the slower legacy expression-tree evaluator whenever `conditions` is blank. A single one-shot backfill across `workflows_event_handlers` and `workflows_workflow_definition_versions` is risky at production scale, so we need a throttled, resumable, per-owner backfill driven from vice-backend, mirroring the existing `TaskHelpers::WorkflowCutover::BooleanEqualToInSmartUpdater` pattern.

## What Changes

- Add `TaskHelpers::WorkflowCutover::JsonLogicConditionsSmartUpdater`, a `TaskHelpers::SmartUpdater::Base` subclass that batches over one owner's `EventHandler` rows (`expression_id IS NOT NULL`) and `WorkflowDefinitionVersion` rows (`entry_expression_id IS NOT NULL`) whose `conditions` is either `NULL` (never attempted) or equal to `Workflows::Conditions::CONVERSION_FAILED_SENTINEL` (previously failed) — excluding rows that already converted successfully.
- Each batch calls the existing `Workflows::Conditionable#sync_conditions_from_expression!` per record, reusing its `update_column` write (no validation callbacks, no PaperTrail version) and its existing fail-closed sentinel + `Workflows.configuration.silent_error_handler` (Rollbar) reporting on conversion failure — no new audit-log infrastructure.
- Add a `rake workflows:backfill_json_logic_conditions` task (mirroring `workflows:boolean_equal_to_in`) that takes a required `--owner_id`, plus `--step`, `--wait_minutes`, and `--preview`, and prints a post-preview / post-enqueue summary. A `--report` mode counts remaining-NULL and sentinel (failed) rows for the owner after a run has drained, so re-running is driven by an observable "done" signal.
- Runs are scoped to one owner (carrier) per invocation, matching how carriers are cut over to JSONLogic independently; a full backfill across all owners means running the rake task once per owner.

## Capabilities

### New Capabilities
- `workflows/conditions-backfill`: throttled, resumable, per-owner backfill of the JSONLogic `conditions` column on `EventHandler` and `WorkflowDefinitionVersion` records that still carry an unconverted or failed-conversion expression.

### Modified Capabilities
(none — no existing spec capabilities are defined in this repo's OpenSpec store yet, and this change does not alter the behavior of `Workflows::Conditionable#sync_conditions_from_expression!` itself, only invokes it from a new caller)

## Impact

- **New code**: `lib/task_helpers/workflow_cutover/json_logic_conditions_smart_updater.rb`, a new task in `lib/tasks/workflows.rake`, specs under `spec/lib/task_helpers/workflow_cutover/` and `spec/tasks/` (or wherever existing SmartUpdater/rake specs live).
- **Reused, unmodified**: `TaskHelpers::SmartUpdater::Base`, `Workflows::Conditionable#sync_conditions_from_expression!`, `Workflows::Conditions::CONVERSION_FAILED_SENTINEL`, `Workflows.configuration.silent_error_handler` (all in the `workflows` gem, already present in this repo's bundle).
- **Runtime**: jobs run on the existing `vice.sneakers.smart_updater` DelayedJob queue; no new queue, no new gem dependency, no schema change (the `conditions` column and its migration already exist on `release`).
- **Operational**: this is a manually-run rake task (runbook-driven), not a scheduled job — matches the existing `boolean_equal_to_in` / `total_loss_okay_to_pay_boolean_to_enum` operational pattern.
