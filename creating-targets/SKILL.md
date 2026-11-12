---
name: creating-targets
description: Creates, runs, validates, and debugs Hotglue Singer targets. Use when scaffolding a target, reproducing a target job, or checking Singer input and target state output.
---

# Running Singer Targets

## Scaffold

```bash
cookiecutter https://github.com/hotgluexyz/hotglue-target-cookiecutter.git
cd target-<service>
uv venv --python 3.10 --seed .venv
uv pip install --python .venv/bin/python -e .
```

Read the generated `README.md` and `AGENTS.md`. If the user may already have a connector for this destination, confirm with them before scaffolding a new target.

Use a dedicated target venv. Match deployed revision and pins when reproducing a job.

## Run locally

```bash
mkdir -p .secrets && cd .secrets
../.venv/bin/target-<service> < data.singer --config config.json > target-state.json
```

Validate: Singer input is valid JSONL with expected streams; `target-state.json` is JSON with per-record bookmarks (success, failure, existing, updated). Trace failures by bookmark error and external ID.

Use the Hotglue Docs MCP for target and platform behavior.
