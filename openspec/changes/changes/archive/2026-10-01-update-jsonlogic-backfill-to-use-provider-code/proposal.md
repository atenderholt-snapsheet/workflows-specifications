# Update JSONLogic Backfill Conditions Script to Use Provider Code

## What

Update the `rake workflows:backfill_json_logic_conditions` task to:
1. Accept `--provider_codes` (carrier/provider code, e.g. "zurich_ie") instead of `--owner_id` (internal database ID)
2. Support "all" or "*" as special values to run backfill for all providers/owners
3. Support comma-separated provider codes for running against multiple specific providers

## Why

- **Usability**: Provider codes are human-readable identifiers that operators know and use. Internal owner IDs require database lookups.
- **Safety**: Accepting "all" reduces the need for operators to discover and list all provider IDs manually, reducing the risk of accidental omissions.
- **Consistency**: Other workflow backfill tasks in the same file (`seed_fallback_events`) already use this pattern with `--provider_codes`.
- **Maintainability**: This change aligns the task interface with the established convention in the codebase.

## Acceptance Criteria

- The task accepts `--provider_codes` parameter (required)
- `--provider_codes=all` and `--provider_codes='*'` trigger backfill for all owners/providers
- Comma-separated provider codes (e.g., `--provider_codes=zurich_ie,zurich_uk`) work correctly
- Invalid provider codes raise clear error messages
- Preview, report, and run modes all work with the new parameter
- Existing usage and documentation are updated
- Tests cover the new parameter handling

## Out of Scope

- This does not change the underlying backfill logic or the SmartUpdater behavior
- This does not modify the `EventHandler` or `WorkflowDefinitionVersion` models
- This does not affect other workflow tasks beyond `backfill_json_logic_conditions`
