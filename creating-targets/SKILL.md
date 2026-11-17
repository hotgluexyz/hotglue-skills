---
name: creating-targets
description: Creates, authenticates, implements, and locally verifies Hotglue Singer targets. Use when scaffolding a target, linking OAuth, or checking Singer input and target state output.
---

# Creating Singer Targets

Default scope: scaffold → auth → implement → local verify. Hand off further work to other skills if prompt asks for it.

Use the Hotglue Docs for target and platform behavior.

## Scaffold

```bash
cookiecutter https://github.com/hotgluexyz/hotglue-target-cookiecutter.git
cd target-<service>
uv venv --python 3.10 --seed .venv
uv pip install --python .venv/bin/python -e .
```

Read the generated `README.md` and `AGENTS.md`. If the user already has a connector for this destination, confirm with them before scaffolding a new target.

Use a dedicated target venv. Match deployed revision and pins when reproducing a job.

## Auth

- **OAuth:** use `authenticating-oauth` for OAuth connectors.
- **API key / Bearer / Basic:** write config under `.secrets/config.json` (git-ignored). Never commit credentials.

## Implement

Follow the cookiecutter `AGENTS.md` and HotglueSingerSDK conventions for building sinks, auth, config schema, and target-state bookmarks. Keep the PR minimal. When reviewing connector PRs, follow plugin `hotglue-connector-coding-rules`.

## Local verify

```bash
mkdir -p .secrets && cd .secrets
../.venv/bin/target-<service> < data.singer --config config.json > target-state.json
```

Validate: Singer input is valid JSONL with expected streams. `target-state.json` is JSON with per-record bookmarks (`success`, `failure`, `existing`, `updated`). Trace failures by bookmark error and external ID.

## Related skills

Only when the user asks for that work:

- ETL / mapping for connections from source -> destination: `creating-hotglue-etl-scripts`
- Deploy or register the target in Hotglue: `managing-available-connectors`
- Remote jobs: `running-hotglue-remote-jobs`
- Incremental/snapshot behavior: `understanding-hotglue-jobs`
