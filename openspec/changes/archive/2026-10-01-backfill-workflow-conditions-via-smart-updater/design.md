## Context

See [proposal.md](proposal.md) - Why. Relevant existing pieces this design builds on, all already present on `release`:

- `TaskHelpers::SmartUpdater::Base` (`lib/task_helpers/smart_updater/base.rb`): batches over an ID range via `fetch(lower, upper)` + `update_batch(records)`, queues one DelayedJob per batch on `vice.sneakers.smart_updater` via `create_batch_background_jobs` (dynamically re-buckets only matching rows, unlike `create_background_jobs`'s fixed-size sweep - better here since matches are sparse against the full table).
- `TaskHelpers::WorkflowCutover::BooleanEqualToInSmartUpdater` (`lib/task_helpers/workflow_cutover/boolean_equal_to_in_smart_updater.rb`): the mirror target - computes initial bounds from `scope.minimum/maximum(:id)`, guards the empty-scope case, exposes `preview`, and delegates each record in `update_batch` to a gem-side one-shot conversion method.
- `Workflows::Conditionable#sync_conditions_from_expression!` (workflow-engine gem, `app/models/concerns/workflows/conditionable.rb`): already the exact per-record conversion primitive needed. It reads `expression`/`entry_expression`, calls `Workflows::Conditions.from_expression`, and writes the result via `update_column(:conditions, ...)` - bypassing validations and PaperTrail (which hooks `save`, not `update_column`). On a `StandardError` from conversion, it writes `Workflows::Conditions::CONVERSION_FAILED_SENTINEL` (`{'==' => [1, 2]}`, a JSONLogic rule that always evaluates false - fail-closed) and reports via `Workflows.configuration.silent_error_handler` (wired to `RollbarHelper.error` in `config/initializers/workflows.rb`). Its doc comment already anticipates being called from "a future backfill task."
- `workflows_event_handlers` and `workflows_workflow_definition_versions` both carry `owner_id`, an expression FK (`expression_id` / `entry_expression_id`), and `conditions:text`. Indexes: `event_handlers` has `(owner_id, code, type)` and a standalone `expression_id` index; `workflow_definition_versions` has standalone `owner_id` and `entry_expression_id` indexes (no composite of the two). `conditions` itself is an unindexed `text` column.
- `rake workflows:boolean_equal_to_in` / `rake workflows:total_loss_okay_to_pay_boolean_to_enum` (`lib/tasks/workflows.rake`): the operational pattern to mirror - `OptionParser`-driven, `--preview` prints row/job counts without enqueueing, real run calls `create_batch_background_jobs`, runbook-style `desc` text.

## Goals / Non-Goals

**Goals:**
- Reuse `sync_conditions_from_expression!` as the sole conversion path, so behavior (including the fail-closed sentinel and error reporting) stays identical between the live `after_save` shadow write and this backfill.
- Scope every run to one owner, and make "is this owner done" an observable query rather than something only inferable from job-queue draining.
- Keep the change additive: no new queue, no new gem, no schema change, no modification to `Conditionable` itself.

**Non-Goals:**
- Building a new persistent conversion-audit table or report. The user confirmed reusing the existing `silent_error_handler` (Rollbar) + querying the sentinel value as the audit trail is sufficient; this design does not add audit infrastructure beyond that.
- Running across all owners in a single invocation. Per-owner scoping is the explicit shape (confirmed with the user); a full company-wide backfill is "run the rake task once per owner," not a feature of the updater itself.
- Changing `new_conditions_met?`'s fallback behavior for blank `conditions`, or removing the legacy expression-tree evaluator. That TODO is explicitly out of scope here.

## Decisions

**Scope predicate: `conditions IS NULL OR conditions = <sentinel>`, not just `IS NULL`.**
The UAC requires re-runs to retry previously-failed records, not just untouched ones. Because `sync_conditions_from_expression!` writes the sentinel (a non-null value) on failure, a naive `conditions IS NULL` scope would silently stop retrying failed rows after their first attempt. Alternative considered: track failures in a separate column/table - rejected as unnecessary complexity when the sentinel value already round-trips through the same column and the serialized column type already supports equality comparison via ActiveRecord's `serialize` coder.

**Per-record `update_batch`, not a new batch-conversion method.**
`update_batch(records)` will call `sync_conditions_from_expression!` once per record, exactly like `BooleanEqualToInSmartUpdater#update_batch` calls `migrate_expression!` per record. Alternative considered: write a new bulk/batch-oriented conversion method in the gem for efficiency - rejected because `sync_conditions_from_expression!` already handles the sentinel-on-failure and error-reporting logic per record, and duplicating that logic bulk-side would risk drift between the shadow-write path and the backfill path.

**Two separate SmartUpdater instances (one per model), not one polymorphic updater.**
`EventHandler` and `WorkflowDefinitionVersion` have different expression association names (`expression` vs `entry_expression`) and different owner-scoping joins (`EventHandler` has `owner_id` directly; `WorkflowDefinitionVersion` also has `owner_id` directly per the schema, so both scope directly - simpler than `WorkflowDefinitionVersion`'s `by_owner` scope, which joins through `workflow_definition` and is used elsewhere for a different purpose). One instance per model keeps `fetch` a plain `where` and avoids `STI`/polymorphic branching inside a shared class. The rake task runs both instances back-to-back for a given `--owner_id`.

**Rake task takes a required `--owner_id`, mirroring `--codes` on `boolean_equal_to_in`.**
Confirmed with the user: per-owner runs, not a global sweep. `--owner_id` is required (raise `ArgumentError` if absent), `--step`/`--wait_minutes` default the same as the mirror task, `--preview` prints counts without enqueueing.

**`--report` mode counts remaining-NULL and sentinel rows per owner, relying on existing indexes rather than adding one.**
The count query is always `owner_id = ? AND expression_id/entry_expression_id IS NOT NULL AND (conditions IS NULL OR conditions = <sentinel>)`. MySQL can use the existing `owner_id` (or composite `(owner_id, code, type)` on `event_handlers`) and expression-FK indexes to narrow to that owner's expression-bearing rows before touching the unindexed `conditions` column, so the `conditions` comparison only runs against an already-small row set - no new index needed. Confirmed with the user (see conversation); flagged as a task to verify with `EXPLAIN` against a realistically-sized owner before running in production, since `workflow_definition_versions` lacks a composite `(owner_id, entry_expression_id)` index and could still scan more rows than `event_handlers` for a very large owner.

## Risks / Trade-offs

- **[Risk] The sentinel-equality `where` clause depends on how `serialize :conditions, coder: JSON` type-casts a Ruby Hash for comparison in SQL.** If `ActiveRecord::Base.where(conditions: Workflows::Conditions::CONVERSION_FAILED_SENTINEL)` doesn't serialize the hash the same way it was written (key order, whitespace), the query could miss sentinel rows. → Mitigation: verify empirically in a spec that a record written with the sentinel via `sync_conditions_from_expression!` is found by the scope's `where` clause, before relying on it; task list calls this out explicitly.
- **[Risk] Running this against a very large owner without `--wait_minutes` could still generate a burst of `update_column` writes if `--step` is set too high.** → Mitigation: default `--step` to the same conservative value as `boolean_equal_to_in` (1000) and document the runbook pattern (preview first, small step, increase only after observing production load) in the rake task's `desc`.
- **[Trade-off] No new persistent audit table means failure visibility is only as good as Rollbar retention + the sentinel query.** Acceptable per user direction; if this proves insufficient in practice, a dedicated audit table is a small follow-up, not a rework of this design.
- **[Risk] Two independent SmartUpdater instances per owner (one per model) means two separate job sequences with no shared "owner is done" signal until both drain.** → Mitigation: the `--report` mode checks both tables and only reports "done" when both are clear.

## Migration Plan

- No schema changes; the `conditions` column and its migration already exist on `release` via the workflows engine's shadow-write change.
- Rollout is operational, not code-deploy-gated beyond merging this change: an operator runs `rake workflows:backfill_json_logic_conditions -- --owner_id=<id> --preview` first, then without `--preview`, then `--report` to confirm convergence, per owner, before that owner is cut over to JSONLogic evaluation (a separate, later step outside this change's scope).
- Rollback: the backfill only ever writes to `conditions` via `update_column`; if a bad run needs to be undone, `conditions` can be reset to `NULL` for the affected owner/table without touching `expression`/`entry_expression`, and the backfill can be re-run once whatever caused the bad conversion is fixed. No `expression`/`entry_expression` data is ever modified.

## Open Questions

None - the two questions that would have changed scope (audit mechanism, owner scoping) were resolved with the user before this design was written.
