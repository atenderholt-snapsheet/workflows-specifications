# Design: Update JSONLogic Backfill Task to Accept Provider Code

## Overview

This change updates the `rake workflows:backfill_json_logic_conditions` task to accept human-readable provider codes (carriers, e.g., "zurich_ie") instead of internal owner IDs, and supports running across multiple providers or all providers in a single invocation.

## Architecture & Implementation Strategy

### 1. Parameter Handling in the Rake Task

**File:** `lib/tasks/workflows.rake` (task `backfill_json_logic_conditions`)

**Changes:**
- Replace `--owner_id` parameter with `--provider_codes`
- Add validation to resolve provider codes to owner IDs before invoking the SmartUpdater
- Support comma-separated provider codes
- Support special values "all" and "*"

**Implementation approach:**
```
Parse --provider_codes parameter
├─ If "all" or "*" → fetch all Provider codes
├─ Otherwise → split by comma
└─ Validate each code exists in the Provider table
   └─ Raise ArgumentError with list of invalid codes if any

For each valid provider code:
├─ Look up the associated Workflows::Owner
│  (relationship: Provider → Workflows::Owner)
├─ Create SmartUpdater instances for that owner_id
├─ Run preview/report/or enqueue as requested
└─ Continue to next provider
```

### 2. Provider → Workflows::Owner Resolution

**Challenge:** The task needs to map from `Provider.code` to `Workflows::Owner.id`.

**Solution:** The Workflows engine (viceroy gem) likely establishes a relationship between Provider and Workflows::Owner. We'll use that relationship to look up the owner:

```ruby
provider = Provider.find_by!(code: provider_code)
owner = Workflows::Owner.find_by!(provider_id: provider.id)
# OR, depending on the actual relationship:
owner = provider.workflows_owner
```

**Note:** If the relationship is unclear from inspection, it will need to be determined during implementation.

### 3. Backwards Compatibility

**Decision:** Do NOT support the old `--owner_id` parameter. This is a breaking change, but:
- The task is an operational/one-off task, not a public API
- Using provider codes is strictly better UX
- The previous invocation approach is already documented and known to ops teams

**Error message:** If someone tries `--owner_id`, clearly explain that the parameter has changed to `--provider_codes`.

### 4. Input Validation & Error Messages

**Valid inputs:**
- Single code: `--provider_codes=zurich_ie`
- Multiple codes: `--provider_codes=zurich_ie,zurich_uk`
- All providers: `--provider_codes=all` or `--provider_codes='*'`

**Error handling:**
- Invalid/unknown code → `ArgumentError: Provider code(s) <codes> not found.`
- Empty provider list after filtering → clear error

### 5. Sequential Processing for Multiple Providers

When multiple providers are specified:
1. Run preview/report for each provider sequentially (no parallelism)
2. If enqueueing, queue all SmartUpdater jobs across all providers
3. Display cumulative results at the end

**Example output:**
```
Preview for zurich_ie: 500 rows across 1 batch
Preview for zurich_uk: 300 rows across 1 batch
Total: 800 rows across 2 batches
```

### 6. Documentation & Help Text

Update the task description and help text to:
- Remove references to `--owner_id`
- Explain `--provider_codes` and its special values
- Update example invocations
- Keep all other documentation (batching, idempotency, failure handling) unchanged

## Testing Strategy

**Unit tests:** Update/expand `spec/lib/tasks/workflows_backfill_json_logic_conditions_spec.rb`:
- Test provider code → owner ID resolution
- Test "all" expands to all providers
- Test "*" works as alias for "all"
- Test comma-separated provider codes
- Test invalid provider code raises clear error
- Test preview/report work with provider codes
- Test multiple providers are processed in order

**Integration test:** Manual verification with local data or staging

## Files Modified

1. **`lib/tasks/workflows.rake`** — update task parameter parsing and provider resolution logic
2. **`lib/task_helpers/workflow_cutover/json_logic_conditions_smart_updater.rb`** — no changes to the updater itself (still accepts owner_id)
3. **`spec/lib/tasks/workflows_backfill_json_logic_conditions_spec.rb`** — add tests for new parameter handling
4. **`openspec/specs/workflows/conditions-backfill/spec.md`** — add requirements for provider code (done in proposal phase)

## Risk & Mitigations

| Risk | Mitigation |
|------|-----------|
| Provider → Workflows::Owner relationship unclear | Check viceroy gem source; adjust if needed during implementation |
| Wrong provider mapped to owner | Extensive testing with known providers; manual verification |
| Multiple providers cause unexpected interactions | Queue all jobs with same throttling; no inter-provider dependencies |
| Operators confused by parameter change | Clear deprecation in error if they use `--owner_id`; update runbooks |

## Out of Scope

- Changes to SmartUpdater::Base or the core backfill logic
- Changes to error reporting or conversion failure handling
- Performance optimizations
- New backfill modes or capabilities beyond what the original task supports
