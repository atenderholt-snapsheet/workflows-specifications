## Context

See proposal.md. Today the QAP object list is built client-side; the workflow list per object comes from the engine's unpaginated `GET /workflows/v1/workflows?object_id=`.

- Object label and claimant description are computed (`WorkflowsIntegration::ObjectService#name`, i18n + provider config), so they cannot be sorted or searched in SQL. This is why ES.
- Workflow status/stage/name live in engine tables; engine has no emit seam today (only `register_*` host hooks).
- Existing ES jobs (`ElasticsearchIndexJob`) are identity-in, load-from-DB, overwrite: last-write-wins, no ordering.
- Every query is scoped to one claim and one provider (<= ~1000 objects).

## Goals / Non-Goals

**Goals:**
- Page, filter, sort QAP objects at ~1000 per claim.
- Index never goes stale from stale message payloads.
- Signalling can never fail or roll back the main work.

**Non-Goals:**
- Replacing the per-object workflow list/detail/advance/cancel endpoints.
- Cross-claim search.
- Indexing full records; only QAP fields.

## Decisions

**1. One document per object, workflows nested.** Doc id = object global id. Fields: `provider_id`, `claim_id`, `object_type`, `label` (+ lowercase keyword subfield), `description`, `workflows[]` {`id`, `label`, `status`, `stage_id`, `stage_name`}. Alternative rejected: one doc per claim with nested objects; ES cannot paginate nested objects.

**2. Identity-only signals.** Messages carry object id only. The job loads the record from the DB, then writes to ES. Deltas/state in messages caused stale docs on the previous team.

**3. Sort** is object type position, then label ascending (case-insensitive). Types are never interleaved. Sorting is selected by a named sort keyword that the server maps to ES sort clauses (a `_script` sort over `object_type` with an ordered type list, then `label`). Callers and UI pass only the keyword, never ES details, so adding or changing a sort needs no reindex. Default keyword orders claim, exposure, litigation. Initial alternative: claim, litigation, exposure. Alternative rejected: a stored rank field per ordering; each new ordering would need a mapping change and reindex.

**4. Filter semantics.** Object matches if ANY nested workflow matches. `inner_hits` flags which workflows matched. State defaults to `started` when no state filter is given, so objects without started workflows are hidden by default (frontend must handle a preselected object that is absent).

**5. Label locale** is the provider default locale, stored at index time; not altered on response.

**6. Signal durability (undecided; options below).** Constraint: signalling must never roll back or fail the main transaction. Signals fire after commit and are rescued/logged. Debounce per object id in all options.

| Option | Pros | Cons |
|---|---|---|
| A. Transactional outbox (row written in the main transaction) | No lost signals; durable debounce via unique key | Outbox failure rolls back critical work (rejected by constraint unless written post-commit); new table + sweeper |
| B. Post-commit best-effort outbox/enqueue + reconciler | Main work isolated; lost signals self-heal; reuses "reindex batch" | Extra scheduled job; reconciler blind spots (label changes via associated records, deletions) need extra queries; cadence/staleness bound to define |
| C. Post-commit best-effort outbox/enqueue, no reconciler | Simplest; fewest moving parts; fine if reliability is high | A signal lost between commit and enqueue stays stale until next change or manual reindex |

Leaning: C, with the "reindex batch" task as the manual fix. If B is chosen, document reconciler cadence and staleness bound here.

Sequencing: this only needs deciding before the signalling task; the document, query, sort, index job, rake tasks and engine hook do not depend on it. Start with C (plain post-commit enqueue, no new table) behind a single "request reindex for object" seam used by all signal sources and the rake tasks, and revisit if drift is observed. Moving to B later only adds a scheduled job on the same reindex-batch path; moving to an outbox table only changes code behind the seam (plus a migration on both schemas).

**7. Rake tasks:** `reindex all` and `reindex batch` (by id/time window); both enqueue the same identity-only job.

**8. Facets.** The response carries aggregations for object type, workflow status and workflow stage. Counts are objects, not workflows (nested aggregation + `reverse_nested`). Each facet applies the termed query, scoping and every filter except its own, so selecting one value still shows the alternatives and the status facet reveals non-`started` counts despite the default filter. Facets cover the whole matching set, not the current page. Alternative rejected: separate facet endpoint; extra round trip and counts can disagree with the page.

**9. Stage facet and filter key on stage name.** Stage names can repeat across workflow definitions; same-named stages merge into one bucket, and an object counts once if any of its workflows is in a stage of that name. The stage filter matches by name. Workflow label is not a separate facet.

**10. Termed query fields.** The term matches object `label`, `description` and nested workflow `label`.

**11. Paging** is `from/size` via JSON:API `page[number]` / `page[size]`. At ~1000 objects per claim this stays well under the ES 10k window, so `search_after` is not needed.

## Risks / Trade-offs

- Aggregations add per-query cost → acceptable at ~1000 docs scoped to one claim.
- Stale docs from lost signals → manual `reindex batch`; revisit B if drift is observed.
- Stage advance burst fires many events → debounce per object id.
- Engine hook must ship first → separate engine ticket; vice-backend gated on release.
- Label sources outside the object (vehicle, claimant) may not touch the object's `updated_at` → explicit listeners on those models.
- Non-ASCII labels sort by code point in a lowercase keyword → accepted unless a collation analyzer is needed per provider locale.
- Script sort adds per-query cost → acceptable at ~1000 docs scoped to one claim.
- Default `started` filter hides objects with none → frontend fallback required.

## Open Questions

- Exact JSON:API param names (`filter[...]`, `page[...]`, `sort`, termed query default `*`): align with existing V2 controllers.
- Reconciler cadence and staleness bound (only if option B).
- Sort keyword names and where the keyword-to-sort mapping lives.
