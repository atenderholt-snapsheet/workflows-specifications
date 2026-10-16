## 1. Engine hook (separate workflow-engine ticket; must release first)

- [x] 1.1 In workflow-engine, add a host-registered hook (same `register_*` pattern as `register_owner_service_class`) fired after commit on `Workflow` create, status change and current-stage change; payload is the owning object identity only; rescue and log errors. Verify with an engine spec showing the hook fires, and that a raising listener does not fail the change.
- [ ] 1.2 (DEFERRED: local path gem in `Gemfile.common` for now; do not commit it) Release the engine, then bump `workflows` in `Gemfile.common`, `Gemfile.lock` and `Gemfile.next.lock`. Verify `bundle install` resolves for both `BUNDLE_GEMFILE` values.

## 2. Index and document

- [x] 2.1 Define the index mapping: `provider_code`, `claim_id`, `object_type`, `label` (text + lowercase keyword subfield), `description`, nested `workflows` (`id`, `label`, `status`, `stage_id`, `stage_name`). Verify with a spec that creates the index and indexes a sample document.
- [x] 2.2 Build the document from the DB for claim, exposure and litigation: label and claimant description via the existing `ObjectService#name` / claimant label logic in the provider default locale, all four workflow statuses nested, litigations skipped when `enable_litigations` is off. Verify with specs per object type, a provider with the flag off, and a non-default-locale provider.
- [x] 2.3 Add the identity-only index job: load the object (and its workflows) from the DB, then write the document; delete the document when the object no longer exists. Verify with specs for update, create and delete, and that stale state in the message cannot reach ES.

## 3. Signalling

- [x] 3.1 Add a single `request reindex for object` seam: post-commit, rescued and logged via `RollbarHelper.error`, debounced per object id, durable across restarts (plain enqueue, no new table; Decision 6 option C). Verify with specs: a raising enqueue leaves the main change committed, and a burst of calls for one id yields one reindex.
- [x] 3.2 Hook vice-backend sources into the seam: claim, exposure and litigation create/destroy, and changes to label and description inputs (claim number, exposure name, vehicle, claimant). Trace callers first (fat models: `claim.rb`, `exposure.rb`). Verify with specs that each source requests a reindex of the right object.
- [x] 3.3 Register the engine hook (from 1.1) to call the seam. Verify with a spec that a stage advance requests a reindex of the owning object.

## 4. Reindex tasks

- [x] 4.1 Add a `reindex all` rake task and a `reindex batch` rake task (selection by object ids or id/time window) that enqueue the same identity-only job. Verify with task specs and a dry run on seeded data (`zurich_ie`).

## 5. Search endpoint

- [x] 5.1 Align parameter naming and routing with existing V2 controllers (`filter[...]`, `page[number]`/`page[size]`, `sort`, termed query defaulting to `*`); add the route and controller using the same authorization as the current QAP endpoints. Verify with request specs for allowed and rejected users.
- [x] 5.2 Query builder: scope to claim and provider; termed query over object `label`, `description` and nested workflow `label`. Verify specs for no term, a matching term and a workflow-label term.
- [x] 5.3 Filters: object type, workflow status and workflow stage name, with any-nested-workflow-matches semantics; default status `started` when none is given; litigations removed when `enable_litigations` is off. Verify specs for each scenario in the search spec, including no-workflow and cancelled-only objects.
- [x] 5.4 Response contents: id, type, label, description and nested workflows with status, stage and a `matched` flag from `inner_hits`. Verify a spec where one of two workflows matches.
- [x] 5.5 Sorting: named sort keywords mapped server-side to a type-order script sort, then `label` keyword; default claim, exposure, litigation; second keyword claim, litigation, exposure; unknown keyword returns a client error. Verify specs for default order, alternate order, no interleaving and the error.
- [x] 5.6 Facets: object type, workflow status and workflow stage name, counting objects (nested + `reverse_nested`), each facet applying every filter except its own; same-named stages from different workflows merge into one bucket; facets cover all matches, not the page. Verify specs for each facet scenario in the search spec.
- [x] 5.7 Pagination with total count. Verify a spec with more objects than the page size.
- [x] 5.8 Document the endpoint under `docs/api/oas3/v2/` and note the index and rake tasks in the relevant README. Verify the docs render and match the specs.
- [x] 5.9 Rename the workflow filters and their facets: `filter[status]` to `filter[workflow_status]`, `filter[stage_name]` to `filter[workflow_stage_name]`, and the `meta.facets` keys `status` and `stage_name` to `workflow_status` and `workflow_stage_name` (`object_type` is unchanged; the `status` and `stage` inside each workflow stay). Update `ObjectSearchParams`, `ObjectSearch`, the controller, their specs, the OpenAPI doc under `docs/api/oas3/v2/`, the Bruno requests and any README wording. Verify with the existing specs for each filter and facet scenario under the new names, and that the old names are no longer accepted.

## 6. Verification

- [ ] 6.1 Against a seeded claim with about 1000 objects, verify paging, filtering, sorting and facets work and response time is acceptable.
- [ ] 6.2 Verify provider isolation: a provider with `enable_litigations` off never sees litigations in documents or facets, while another provider does.
- [ ] 6.3 Verify that signal failure does not affect the main work (stop ES or the queue, advance a stage, confirm the stage change commits) and that pending signals survive a restart.
- [ ] 6.4 Hand the final contract (params including `filter[workflow_status]` and `filter[workflow_stage_name]`, sort keywords, facets keyed `object_type`, `workflow_status` and `workflow_stage_name`, absent-default-object case) to the vice-frontend ticket.
- [ ] 6.5 Follow-up, not part of this change: decide the exposure object label and create a new change for it (`openspec new change`). Today the indexed label is `ObjectService#name` only (e.g. `Subaru Legacy  `, with trailing spaces when year or registration is empty), while the QAP shows the name plus a frontend-added coverage suffix (e.g. `Subaru Legacy - Unknown Coverage`). Decide: (a) index the full display label including coverage, (b) index coverage as a separate field and let the frontend compose, or (c) leave the label as is. Also decide whitespace normalization, the backend rule for the coverage fallback and locale, `coverage_assignment.premium_class` as a reindex trigger, and whether the suffix applies when `enable_exposure_name_update` is on. Confirm the frontend's actual rule first.
