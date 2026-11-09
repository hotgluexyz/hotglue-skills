---
name: running-hotglue-remote-jobs
description: >-
  Run and poll sync jobs on a hotglue environment over its client API: select
  streams/fields for a job, and create/poll remote jobs. Use when asked to run
  discovery or sync remotely, or change what a job syncs for one run. For
  available-connector install_uri / branch overrides, use managing-available-connectors.
---

# Running hotglue remote jobs

Operate jobs on a hotglue environment: choose streams, create a job, and poll
until it finishes. 

## Setup and auth

- **Defaults** (ask the user when unknown):
  - UI: `https://hotglue.com/env/{env_id}/content/flows/v2/{flow_id}/connectors`
  - `env_id`: `{env_id}`
  - `flow_id`: `{flow_id}`
  - `tenant`: `default`
- **API root**: the instance's client API root.
  - `https://api.hotglue.com`
- **Auth**: read the env **Secret API Key** from `$HOTGLUE_SECRET_API_KEY`
  (not the Public API Key or JWT private key). If unset, ask the user to export
  it — never ask them to paste the key into prompts or chat. Send it as the
  `x-api-key` header on every call; do not display or log the key.
- **Also need**: `connector_id`.
- List tenants with `GET /tenants/{env_id}` when the tenant is unknown or not
  `default`. Nonexistent envs return `Environment does not exist!`; missing
  routes return `Missing Authentication Token` — treat those as a wrong path,
  not auth failure.

## 1. Select streams/fields (field map)

Field-map meaning, which linked entity to patch, and verification after
discovery are in **managing-available-connectors**. API for a saved map:

```
PATCH /v2/{env_id}/{flow_id}/{tenant}/linkedConnectors
{"connector_id": "crm",
 "field_map": {
   "accounts": {"name": "accounts", "selected": ["*"]},
   "bills":    {"name": "bills",    "selected": ["*"]},
   "invoices": {"name": "invoices", "selected": ["*"]}
 }}
```

- One entry per stream; `"selected": ["*"]` = all fields.
- For a one-off selection change, prefer `override_field_map` /
  `override_selected_tables` on the job (section 2) over mutating the saved
  field map.

## 2. Execute a remote job

Create a read job. Discovery runs as part of the lifecycle
(`DISCOVER_STARTED`), then sync. Control which streams are selected with the
field map (section 1) or job overrides below.

```
POST /v2/{env_id}/{flow_id}/{tenant}/jobs
{"connector_id": "crm", "job_type": "read"}
```

- Omit `reset_source_state` (or set `false`) for a normal incremental run.
  `reset_source_state: true` forces a full sync (clears bookmarks) — use only
  when intentional. For full vs incremental behavior, follow
  **understanding-hotglue-jobs** (and its linked
  [sync jobs](https://docs.hotglue.com/key-concepts/jobs/sync-jobs) /
  [job settings](https://docs.hotglue.com/environment-settings/job-settings#basic-settings)
  docs).
- Useful variations:
  - `override_field_map` — one-off raw stream/field selection (same shape as a
    saved field map) without touching the saved map.
  - `override_selected_tables` — one-off unified-schema object selection
    (e.g. `{"Vendors": true}`), not a raw stream field map.
  - `override_start_date` / `override_end_date` — narrow the sync window.
  - `override_connector_config` / `override_source_config` — one-off config
    values.
  - `queue_if_active_job: true` — queue instead of erroring when a job is
    already running.
- **Poll** (the create response returns `s3_root`):

```
GET /{env_id}/{flow_id}/{tenant}/jobs/status?job_root=<url-encoded s3_root>
```

- Lifecycle: `JOB_CREATED` → `DISCOVER_STARTED`/`SYNC_STARTED` →
  `ETL_STARTED` → `ETL_SUCCESS`/`ETL_FAILED` → ... → `JOB_COMPLETED`.
  **Terminal success is only `JOB_COMPLETED`.** Stop polling on `JOB_COMPLETED`
  or any `*_FAILED` status and report the result — do not keep waiting for
  success after a failure. Sync can fully succeed while the job still ends
  `ETL_FAILED` (e.g. a broken transformation script).
- Read results from `metrics`: `recordCount` per stream is what actually
  synced; other metric keys (e.g. `estimatedRecordCount`) appear only when the
  deployed tap emits them. If `rolled_back` is present and true, the job
  restored pre-job bookmarks/snapshots — the next sync type depends on that
  restored state, not on the flag alone.
- Hand back `job_url` when the response includes it; otherwise share
  `job_id` so the user can open the job in the dashboard.
