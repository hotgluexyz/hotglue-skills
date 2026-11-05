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
  - UI: `https://hotglue.xyz/env/{env_id}/content/flows/v2/{flow_id}/connectors`
  - `env_id`: `{env_id}`
  - `flow_id`: `{flow_id}`
  - `tenant`: `default`
- **API root**: the instance's client API root.
  - `https://api.hotglue.com`
- **Auth**: ask the user for the env **Secret API Key** before any API call
  (not the Public API Key, and not the JWT private key). They find it in the
  dashboard under env settings → **Environment API Keys** → **Secret API Key**.
  Send it as the `x-api-key` header on every call.
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
{"connector_id": "crm", "job_type": "read",
 "reset_source_state": true}
```

- `reset_source_state: true` forces a full sync (clears bookmarks). For full vs
  incremental behavior, follow **understanding-hotglue-jobs** (and its linked
  [sync jobs](https://docs.hotglue.com/key-concepts/jobs/sync-jobs) /
  [job settings](https://docs.hotglue.com/environment-settings/job-settings#basic-settings)
  docs).
- Useful variations:
  - `override_field_map` / `override_selected_tables` — one-off stream/field
    selection without touching the saved field map. For a minimal job, select
    one stream with a single field.
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
  **Terminal success is only `JOB_COMPLETED`.** Sync can fully succeed while
  the job still ends `ETL_FAILED` (e.g. a broken transformation script).
- Read results from `metrics`: `recordCount` per stream is what actually
  synced; other metric keys (e.g. `estimatedRecordCount`) appear only when the
  deployed tap emits them. `rolled_back: true` means the failure cleared the
  source state, so the next job is a full sync again.
- The dashboard link in `job_url` is a good thing to hand back to the user.
