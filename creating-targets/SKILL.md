---
name: creating-targets
description: Creates, authenticates, runs, and optionally deploys Hotglue Singer targets. Use when scaffolding a target, linking OAuth, reproducing a target job, or checking Singer input and target state output. Extend to ETL, Hotglue deploy, and remote jobs only when the user prompt asks for them.
---

# Creating Singer Targets

Default scope: scaffold → auth → implement → local verify. Hand off to sibling skills rather than duplicating them.

**Prompt-gated steps:** ETL, deploy to Hotglue, flow/tenant wiring, remote jobs, and the remote feedback loop (sections 5–9) run **only when the user prompt or instruction set explicitly asks for them**. Do not invent an end-to-end platform rollout when the request is local target work only.

Use the Hotglue Docs MCP for target and platform behavior.

## 1. Scaffold

```bash
cookiecutter https://github.com/hotgluexyz/hotglue-target-cookiecutter.git
cd target-<service>
uv venv --python 3.10 --seed .venv
uv pip install --python .venv/bin/python -e .
```

Read the generated `README.md` and `AGENTS.md`. If the user may already have a connector for this destination, confirm with them before scaffolding a new target.

Use a dedicated target venv. Match deployed revision and pins when reproducing a job.

## 2. Auth

- **OAuth:** use `authenticating-oauth` (magic link → open/print → verify linked config). That skill returns control after verification — continue this playbook; do not stop at auth.
- **API key / Bearer / Basic:** write config under `.secrets/config.json` (git-ignored). Never commit credentials.

## 3. Implement

Follow the cookiecutter `AGENTS.md` and HotglueSingerSDK conventions for sinks, auth, config schema, and target-state bookmarks. Keep the PR minimal; when reviewing connector PRs, follow plugin `hotglue-connector-coding-rules`.

## 4. Local verify

```bash
mkdir -p .secrets && cd .secrets
../.venv/bin/target-<service> < data.singer --config config.json > target-state.json
```

Validate: Singer input is valid JSONL with expected streams; `target-state.json` is JSON with per-record bookmarks (`success`, `failure`, `existing`, `updated`). Trace failures by bookmark error and external ID.

## 5. ETL (prompt-gated)

Skip unless the prompt asks for an ETL script, transform, or mapping between source and target.

If instructed: use `creating-hotglue-etl-scripts`. Require `externalId` on every ETL export before declaring a direction complete (`working-with-hotglue` v2 guardrails).

## 6. Deploy to Hotglue (prompt-gated)

Skip unless the prompt asks to deploy, register, fork, or update the target in a Hotglue environment.

If instructed, use `managing-available-connectors`:

1. `hotglue env download` → inspect existing available entities before adding or forking.
2. Set `write_connector.install_uri` to the target git URI (branch/tag/commit as needed).
3. Set `singer_sdk: true` under `write_connector` for Singer SDK targets.
4. Preserve auth UI fields and role flags; mark `isForked: true` only for real forks.
5. `hotglue env deploy` and confirm the change.

## 7. Wire flow and tenant (prompt-gated)

Skip unless the prompt asks to set up a flow, support the entity, link a tenant, discover, or set a field map in Hotglue.

If instructed:

1. Ensure the entity is supported on the v2 flow.
2. Link the tenant (OAuth via `authenticating-oauth`, or API-key link with secrets kept out of chat).
3. Run discover once for the entity being read; confirm the catalog before setting a field map — see `managing-available-connectors`.
4. For write jobs, set the field map on the linked **source** (the read side), not on the primary connector unless the connector itself is the read side.
5. Re-fetch the linked entity and verify the persisted field map and selected streams independently of the catalog.

## 8. Remote jobs and debug (prompt-gated)

Skip unless the prompt asks to run Hotglue jobs, verify end-to-end in an environment, or debug remote job logs.

If instructed:

- Create and poll jobs with `running-hotglue-remote-jobs`.
- Check incremental/snapshot traps with `understanding-hotglue-jobs` before treating empty syncs as bugs.
- For S3 job artifacts and logs, use plugin `hotglue-job-details` / `job-summary` / `job-logs`.
- Hand back `job_url` or `job_id`; project CLI/`--json` output to non-secret fields only.

## 9. Feedback loop (prompt-gated)

Skip unless the prompt includes remote verify/debug (section 8) or explicitly asks to iterate on job failures.

On remote failure when that work is in scope:

1. Inspect job status and logs.
2. Fix target and/or ETL.
3. Redeploy `install_uri` or ETL as needed.
4. Retrigger a previous job for ETL-only changes (preserve sync output). Run a new job when source data or config changed.
5. Repeat until local and remote results agree.
