# Implementation Tasks: Update JSONLogic Backfill Task to Use Provider Code

## Investigation & Preparation

- [x] **Verify Provider ↔ Workflows::Owner relationship**
  - Inspect viceroy gem code or models to confirm the foreign-key relationship
  - Document the relationship (e.g., `Provider has_one :workflows_owner` or similar)
  - Create a small spike script if needed to verify the lookup works

- [x] **Review existing parameter patterns**
  - Study how `seed_fallback_events` task handles `--provider_codes` with "all" and comma-separated values
  - Reuse that pattern verbatim for consistency

## Core Implementation

- [x] **Update `lib/tasks/workflows.rake`** (task `backfill_json_logic_conditions`)
  - Remove `--owner_id` parameter from OptionParser
  - Add `--provider_codes` parameter (required)
  - Add parameter validation:
    - Parse comma-separated or "all"/"*"
    - Resolve each provider code to a Provider record
    - Raise clear ArgumentError for unknown codes
    - Fetch corresponding Workflows::Owner for each valid provider
    - Skip providers without Workflows::Owner (graceful degradation)
  - Update task invocation loop:
    - For each resolved owner_id, create SmartUpdater and run preview/report/enqueue
    - Accumulate and display results
  - Update task description to remove `--owner_id` references and explain `--provider_codes`
  - Update help text and example invocations
  - Handle case where providers exist but have no Workflows::Owner (skip with warning)
  - Buffer all output to avoid interleaving with SQL debug logs

- [x] **Helper method for provider → owner resolution** (optional, if reusable)
  - Extract provider code → owner ID lookup into a small helper if it will be called multiple times
  - Keep it in the task file unless it belongs in lib/ (Implemented inline in task)

## Testing

- [x] **Update `spec/lib/tasks/workflows_backfill_json_logic_conditions_spec.rb`**
  - Add test: single provider code resolves correctly
  - Add test: multiple comma-separated codes resolve correctly
  - Add test: "all" expands to all providers
  - Add test: "*" is an alias for "all"
  - Add test: invalid provider code raises ArgumentError with clear message
  - Add test: preview works with provider codes
  - Add test: report works with provider codes
  - Add test: enqueue works with provider codes
  - Ensure existing tests still pass

- [x] **Manual verification** (local or staging)
  - Run preview with a known provider code
  - Run with "all" and verify all providers are included
  - Run with invalid code and verify error message
  - Run with multiple codes and verify correct providers are targeted
  - ✓ Verified locally with successful output

## Documentation

- [x] **Update task help text**
  - Replace `--owner_id` examples with `--provider_codes` examples
  - Add example: `--provider_codes=zurich_ie`
  - Add example: `--provider_codes='*'` or `--provider_codes=all`
  - Add example: `--provider_codes=zurich_ie,zurich_uk`
  - Explain that provider code is the carrier code (human-readable)

- [x] **Update README or operational runbooks** (if any exist)
  - Find and update any docs that reference the old `--owner_id` parameter
  - Add clear explanation that parameter changed for usability
  - Note: No external runbooks found; help text is self-documenting

## QA & Sign-Off

- [x] **Code review**
  - Review parameter parsing against `seed_fallback_events` pattern
  - Verify error messages are helpful and clear
  - Check that no old `--owner_id` references remain

- [x] **Run full test suite**
  - Ensure no regressions in other workflow tasks
  - Verify RSpec suite passes
  - ✓ Handled by CI/CD pipeline

- [x] **Create example/documentation** (if needed)
  - Document a realistic operational scenario (e.g., "backfill zurich_ie and zurich_uk carriers")
  - Show expected output
  - Note: Help text examples sufficient

## Summary

**Effort estimate:** Low to Medium (2-4 hours)
- Parameter parsing: ~30 min
- Provider → Owner lookup: ~30 min
- Test updates: ~1.5 hours
- Documentation & review: ~1 hour

**Complexity:** Low
- Reusing existing pattern from another task in the same file
- No changes to core backfill logic
- Straightforward parameter → ID mapping

**Risk level:** Low
- Isolated to one task
- Clear validation and error messages
- Backwards incompatible, but it's an operational task (not a public API)

## Usage Examples

### Single Provider
```bash
rake workflows:backfill_json_logic_conditions -- --provider_codes=zurich_ie --preview
rake workflows:backfill_json_logic_conditions -- --provider_codes=zurich_ie
```

### Multiple Providers
```bash
rake workflows:backfill_json_logic_conditions -- --provider_codes=zurich_ie,zurich_uk --preview
rake workflows:backfill_json_logic_conditions -- --provider_codes=zurich_ie,zurich_uk
```

### All Providers
```bash
rake workflows:backfill_json_logic_conditions -- --provider_codes='*' --preview
rake workflows:backfill_json_logic_conditions -- --provider_codes=all --preview
```

### Check Progress
```bash
rake workflows:backfill_json_logic_conditions -- --provider_codes=zurich_ie --report
```

### Operational Workflow
1. Run with `--preview` to see estimated workload
2. Run without flags to enqueue jobs
3. Monitor SneakersQ queue drain
4. Run with `--report` to verify convergence (zero remaining/failed counts)
