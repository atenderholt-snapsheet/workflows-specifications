# Conditions Backfill Spec - Provider Code Enhancement

This change enhances the existing `workflows/conditions-backfill` specification in `openspec/specs/workflows/conditions-backfill/spec.md` with new requirements for provider-code-based parameter handling.

## Spec Reference

See `openspec/specs/workflows/conditions-backfill/spec.md` for the complete specification, including:

### Original Requirements
- Backfill scope limited to one owner and unconverted rows
- Batching and throttling support
- Idempotent and resumable behavior
- Audit history transparency
- Conversion failure handling and reporting
- Full owner backlog conversion

### New Requirements (Added by this change)
- Backfill accepts provider code instead of owner ID
- Single provider code resolution
- Multiple provider codes support
- "all" and "*" special values for all providers
- Invalid code error handling
- Provider code works with preview and report modes

## Relationship to Implementation

The task in `lib/tasks/workflows.rake` (specifically `backfill_json_logic_conditions`) is the primary implementation artifact. The SmartUpdater class `lib/task_helpers/workflow_cutover/json_logic_conditions_smart_updater.rb` remains unchanged and continues to accept `owner_id` internally; the parameter mapping happens at the rake task level.

## Additional Context

The `seed_fallback_events` task in the same `lib/tasks/workflows.rake` file already implements the provider-code pattern with "all"/"*" support. This change applies the same proven pattern to the backfill task for consistency.
