---
name: understanding-hotglue-jobs
description: Explains Hotglue taps, targets, incremental jobs, settings, and snapshots. Use before reviewing sync requirements, missing records, stream dependencies, or repeat-sync identity.
---

# Understanding Hotglue Jobs

Definitions live in the docs linked below. This skill is the judgment layer for
sync complaints: do not treat incremental, zero records, or missing lookups as
bugs until these traps are checked.

## Roles

- Identify job direction (read vs write) and which component owns each stage
  before diagnosing missing data.
- Docs: [connector roles](https://docs.hotglue.com/key-concepts/connectors/connectors#connectors-vs-sources-vs-targets).

## Incremental syncs

- Docs: [sync jobs](https://docs.hotglue.com/key-concepts/jobs/sync-jobs) (first
  job = historical from start date; later jobs = new/updated).
- A stream returning zero records can be correct when nothing changed.
- Some streams are not incremental, regardless of the configured `start_date`.

## Job settings

- Docs: [job settings](https://docs.hotglue.com/environment-settings/job-settings#basic-settings)
  (Force full sync, Never sync historical data, Force discover, Save snapshots
  even when jobs fail).
- Inspect tenant settings and job overrides before inferring what a particular
  job fetched.
- Source-state resets and full extraction do not clear ETL lookup or target-ID
  snapshots.

## ETL lookup snapshots

- Docs: [snapshotting](https://docs.hotglue.com/transformation/snapshots#what-is-snapshotting).
- Current sync output may contain only changed rows; lookups may exist only in
  the snapshot cache.
- Selecting a dependency does not guarantee rows every run. A dependency
  changing does not automatically re-export its consumers — inspect the ETL and
  actual snapshot contents.

## Target-ID snapshots

- Docs: [saving record IDs](https://docs.hotglue.com/key-concepts/jobs/write-jobs#saving-record-ids-in-snapshots)
  (`InputId` / `RemoteId`).
- Distinct from source bookmarks and ETL lookup snapshots.
- An ID lookup alone does not guarantee correct upserts or deduplication —
  inspect target behavior.
