## Purpose

Provides a throttled, resumable, per-owner backfill that populates the JSONLogic `conditions` column on historical `EventHandler` and `WorkflowDefinitionVersion` records so an owner (carrier) can be cut over to JSONLogic condition evaluation once its backlog is converted.

## ADDED Requirements

### Requirement: Backfill scope is limited to one owner and unconverted rows
The backfill SHALL operate on a single owner (carrier) per invocation, and SHALL only select `EventHandler` rows with a non-null expression association and `WorkflowDefinitionVersion` rows with a non-null entry-expression association whose `conditions` is not already a successfully-converted value.

#### Scenario: Only rows with an expression are selected
- **WHEN** the backfill runs for an owner
- **THEN** it selects only `EventHandler` rows for that owner with a non-null expression association, and only `WorkflowDefinitionVersion` rows for that owner with a non-null entry-expression association

#### Scenario: Successfully converted rows are excluded
- **WHEN** a record's `conditions` already holds a successfully-converted JSONLogic value (neither null nor the conversion-failed sentinel)
- **THEN** the backfill does not select that record and does not attempt to re-convert it

### Requirement: Backfill runs in bounded, throttleable batches
The backfill SHALL process matching records in batches bounded by a configurable batch size, and SHALL support spacing successive batches apart in time via a configurable delay, so a run does not apply unbounded write load to the database in a single burst.

#### Scenario: Batch size is configurable
- **WHEN** an operator runs the backfill with a batch size option
- **THEN** each unit of work touches at most that many records

#### Scenario: Batches can be spaced out over time
- **WHEN** an operator runs the backfill with a wait-between-batches option
- **THEN** successive batches are scheduled that many minutes apart rather than all at once

### Requirement: Backfill is idempotent and resumable
Re-running the backfill for the same owner SHALL only act on records that are still unconverted (null `conditions`) or that previously failed conversion (sentinel `conditions`); it SHALL NOT re-process records that already converted successfully, and running it multiple times SHALL NOT change the outcome for a record beyond its first successful conversion.

#### Scenario: Re-run skips already-converted records
- **WHEN** the backfill is run a second time for an owner after a first run already converted some records successfully
- **THEN** the second run does not re-select or rewrite those already-converted records

#### Scenario: Re-run retries previously-failed records
- **WHEN** a record failed conversion on a prior run (its `conditions` holds the conversion-failed sentinel)
- **THEN** a subsequent run for that owner selects and retries that record

### Requirement: Backfill writes produce no audit-history noise
Writes made by the backfill to the `conditions` column SHALL NOT create PaperTrail versions or other audit-history records.

#### Scenario: No PaperTrail version is created
- **WHEN** the backfill converts a record's expression into `conditions`
- **THEN** no new PaperTrail version is recorded for that write

### Requirement: Conversion failures are surfaced, not silently dropped
When a record's expression cannot be converted to JSONLogic, the backfill SHALL mark that record so it is distinguishable from both an unconverted record and a successfully-converted record, and SHALL report the failure through the system's existing error-reporting channel rather than raising out of the batch.

#### Scenario: A failing record does not abort its batch
- **WHEN** one record's expression fails conversion during a batch
- **THEN** that failure is reported through the existing error-reporting channel and the remaining records in the batch are still processed

#### Scenario: Failed conversions are queryable after a run
- **WHEN** a backfill run for an owner has finished
- **THEN** an operator can determine, from the state of the `conditions` column, how many of that owner's in-scope records still need conversion versus how many previously failed

### Requirement: A full run for an owner leaves every expression-bearing record converted
After repeated backfill runs for an owner converge (no records remain in the unconverted-or-failed scope, or all remaining failures are reported and accepted), every `EventHandler` and `WorkflowDefinitionVersion` record for that owner with a non-null expression association SHALL have a non-null `conditions` value.

#### Scenario: Owner backlog is fully converted
- **WHEN** an operator has run the backfill for an owner until no records remain in the unconverted-or-failed scope
- **THEN** every `EventHandler` and `WorkflowDefinitionVersion` record for that owner with a non-null expression association has a non-null `conditions` value
