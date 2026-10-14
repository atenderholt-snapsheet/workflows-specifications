## Why

The Workflows quick access panel (QAP) "View" dropdown lists every claim, exposure and litigation, built client-side from the claim in memory. It breaks past ~20 entries and cannot paginate, filter or sort. Entries will grow to ~1000 per claim.

## What Changes

- New JSON:API-style `GET` endpoint in vice-backend that searches QAP objects (claim, exposure, litigation) via ElasticSearch: paging, `filter[...]`, termed query (default `*`), multi-field sort.
- The endpoint also returns filter facets (object type, workflow status, workflow stage) with object counts alongside the documents.
- New ES index: one document per object, QAP fields only, workflows nested.
- Event-driven reindex: workflow-engine and vice-backend emit identity-only signals; only vice-backend indexes, loading the record from the DB before writing to ES.
- "Reindex all" and "reindex batch" rake tasks.
- workflow-engine: new host-registered hook fired on workflow create / status / stage change (separate engine ticket, must release first).
- vice-frontend: dropdown becomes a searchable, paged picker (tracked separately).

## Capabilities

### New Capabilities
- `workflow-object-search`: the search endpoint contract (params, filters, sort, defaults, authorization, response shape).
- `workflow-object-index`: index document shape, signalling, debounce, durability, and reindex tasks.

### Modified Capabilities
<!-- none: no existing specs -->

## Impact

- vice-backend: new controller/route, ES index + query builder, indexing job/worker, rake tasks, listeners on Claim/Exposure/Litigation and related label sources.
- workflow-engine gem: new `register_*` hook (version bump, release before ship).
- vice-frontend: `WorkflowsListView` / `WorkflowsContainer`, including the case where `defaultExposureId` is absent from the filtered results.
- Ops: new ES index to create and backfill.
