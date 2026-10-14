## Purpose

Defines what the QAP object index contains and how it is kept in sync with workflow-engine and vice-backend changes without risking the main work.

## ADDED Requirements

### Requirement: Index document
The index SHALL hold one document per workflow object containing only the fields needed by the quick access panel for filtering, sorting and rendering, with the object's workflows nested.

#### Scenario: Object with workflows
- **WHEN** an object with two workflows is indexed
- **THEN** one document exists for the object with two nested workflows including all four possible statuses where present

### Requirement: Identity-only signals
Reindex signals SHALL carry only the object identity. The indexer MUST load the current record from the database before writing to ElasticSearch.

#### Scenario: State changes after signal
- **WHEN** an object changes again after its signal was queued but before it is processed
- **THEN** the indexed document reflects the latest database state

### Requirement: Signal sources
Both workflow-engine and vice-backend SHALL emit signals; only vice-backend SHALL write to the index. Signals SHALL fire on workflow create, status change and stage change, on changes to anything feeding an object's label or description, and on object creation and removal.

#### Scenario: Stage advance
- **WHEN** a workflow advances to a new stage
- **THEN** the owning object is reindexed

#### Scenario: Object renamed
- **WHEN** an exposure's name or label inputs change
- **THEN** the exposure is reindexed

#### Scenario: Object removed
- **WHEN** an object is removed
- **THEN** its document is removed from the index

### Requirement: Main work isolation
Signalling SHALL NOT cause the originating operation to fail or roll back.

#### Scenario: Signalling fails
- **WHEN** emitting or enqueueing a signal raises an error
- **THEN** the originating change remains committed and the error is logged

### Requirement: Restart durability
Queued signals MUST survive a service restart.

#### Scenario: Restart with pending signals
- **WHEN** the service restarts with signals pending
- **THEN** those signals are still processed afterward

### Requirement: Debounce
Multiple signals for the same object within the debounce window SHALL result in a single reindex.

#### Scenario: Burst of events
- **WHEN** several signals arrive for one object within the window
- **THEN** the object is indexed once

### Requirement: Reindex tasks
The system SHALL provide a rake task to reindex all objects and a rake task to reindex a batch of objects.

#### Scenario: Reindex all
- **WHEN** the reindex-all task runs
- **THEN** every workflow object has a current document

#### Scenario: Reindex batch
- **WHEN** the reindex-batch task runs with a selection of objects
- **THEN** only those objects are reindexed

### Requirement: Litigation gating
Litigation objects SHALL be indexed only while `enable_litigations` is enabled for the provider.

#### Scenario: Flag disabled
- **WHEN** a provider has `enable_litigations` disabled
- **THEN** its litigations are not present in the index
