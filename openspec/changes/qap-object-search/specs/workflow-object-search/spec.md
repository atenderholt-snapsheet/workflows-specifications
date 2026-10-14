## Purpose

Lets the workflows quick access panel page, filter, search and sort a claim's workflow-capable objects (claim, exposures, litigations) so the object picker scales to ~1000 entries.

## ADDED Requirements

### Requirement: Object search endpoint
The system SHALL expose a JSON:API-style `GET` endpoint, following the existing V2 conventions, that returns the objects of one claim for the requesting provider. Results SHALL be backed by ElasticSearch.

#### Scenario: Scoped to claim and provider
- **WHEN** a client requests objects for a claim
- **THEN** only objects of that claim and the requester's provider are returned

### Requirement: Termed query
The endpoint SHALL accept a termed query that defaults to `*` (match all).

#### Scenario: No term
- **WHEN** no query is supplied
- **THEN** all objects of the claim that pass the filters are returned

#### Scenario: Term supplied
- **WHEN** a query term is supplied
- **THEN** only objects whose label, description or workflow label match the term are returned

#### Scenario: Workflow label term
- **WHEN** the term matches the label of one of an object's workflows but not the object's own label
- **THEN** the object is returned

### Requirement: Filters
The endpoint SHALL support `filter[...]` on object type, workflow status and workflow stage. An object SHALL match a workflow filter if any of its workflows matches.

#### Scenario: Any workflow matches
- **WHEN** an object has one `started` and one `cancelled` workflow and the status filter is `cancelled`
- **THEN** the object is returned

#### Scenario: Stage filter by name
- **WHEN** the stage filter is a stage name used by workflows of two different definitions
- **THEN** objects with a workflow in a stage of that name are returned, regardless of definition

#### Scenario: Object type filter
- **WHEN** the object type filter is `exposure`
- **THEN** no claim or litigation objects are returned

### Requirement: Default state filter
When no workflow status filter is supplied, the endpoint SHALL behave as if the status filter were `started`. Other statuses (`created`, `failed_to_start`, `cancelled`) SHALL be returned only when explicitly filtered.

#### Scenario: Default hides non-started
- **WHEN** no status filter is supplied and an object has only a `cancelled` workflow
- **THEN** the object is not returned

#### Scenario: Object with no workflows
- **WHEN** no status filter is supplied and an object has no workflows
- **THEN** the object is not returned

#### Scenario: Explicit expansion
- **WHEN** the status filter explicitly includes `cancelled`
- **THEN** objects with a `cancelled` workflow are returned

### Requirement: Sorting
Results SHALL be grouped by object type and sorted by object label ascending, case-insensitively, within each group. Object types SHALL NOT be interleaved. The default order is claim, exposure, litigation. The caller MAY select a different order by passing a named sort keyword; callers SHALL NOT need to know how the keyword is implemented.

#### Scenario: Default order
- **WHEN** no sort keyword is supplied
- **THEN** the claim is first, then all exposures by label, then all litigations by label

#### Scenario: Caller-selected order
- **WHEN** the caller passes the keyword for claim, litigation, exposure order
- **THEN** the claim is first, then all litigations by label, then all exposures by label

#### Scenario: Not interleaved
- **WHEN** any supported sort keyword is used
- **THEN** no litigation appears between two exposures, and no exposure between two litigations

#### Scenario: Unknown keyword
- **WHEN** the caller passes an unrecognized sort keyword
- **THEN** the request is rejected with a client error

### Requirement: Available filters
The response SHALL include facets for object type, workflow status and workflow stage, each value with a count of matching objects (not workflows). Each facet SHALL be computed from the termed query, the claim/provider scope and every active filter except its own filter. Facets SHALL cover all matches, not only the current page.

#### Scenario: Facets returned
- **WHEN** a client searches a claim's objects
- **THEN** the response includes object type, workflow status and workflow stage facets with object counts alongside the documents

#### Scenario: Facet ignores its own filter
- **WHEN** the object type filter is `exposure`
- **THEN** the object type facet still reports counts for claim and litigation

#### Scenario: Default status hides, facet reveals
- **WHEN** no status filter is supplied and an object has only a `cancelled` workflow
- **THEN** that object is not in the documents, but the status facet still reports it under `cancelled`

#### Scenario: Same-named stages merge
- **WHEN** workflows from two different definitions each have a stage named `Review`
- **THEN** the stage facet has one `Review` bucket counting each matching object once

#### Scenario: Object counted once
- **WHEN** an object has two `started` workflows
- **THEN** the `started` count includes the object once

#### Scenario: Litigation flag off
- **WHEN** the provider has `enable_litigations` disabled
- **THEN** litigation does not appear in the object type facet

#### Scenario: Not paginated
- **WHEN** matches exceed the page size
- **THEN** facet counts cover all matches, not only the returned page

### Requirement: Pagination
The endpoint SHALL paginate results and report total count.

#### Scenario: Page of results
- **WHEN** a claim has more objects than the page size
- **THEN** only one page is returned with pagination metadata

### Requirement: Response contents
Each returned object SHALL include its id, type, label, description and its nested workflows. Each workflow SHALL include id, label, status, stage and whether it matched the supplied filters.

#### Scenario: Matched flag
- **WHEN** an object is returned because one of two workflows matched the filter
- **THEN** both workflows are returned and only the matching one is flagged as matched

### Requirement: Litigation flag
When `enable_litigations` is not enabled for the provider, litigation objects SHALL NOT be returned.

#### Scenario: Flag off
- **WHEN** the provider has `enable_litigations` disabled
- **THEN** no litigation objects are returned, with or without an object type filter

### Requirement: Authorization
The endpoint SHALL use the same authorization as the current workflows quick access panel endpoints.

#### Scenario: Unauthorized
- **WHEN** a user lacking the current workflows access permission calls the endpoint
- **THEN** the request is rejected as it would be for the existing endpoints

### Requirement: Label locale
Labels and descriptions SHALL be returned as indexed, in the provider's default locale, without request-time translation.

#### Scenario: Request locale differs
- **WHEN** the request locale differs from the provider default locale
- **THEN** the label is still in the provider default locale
