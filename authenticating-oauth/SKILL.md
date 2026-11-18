---
name: authenticating-oauth
description: Authenticate OAuth taps, targets, and connectors via Hotglue magic links. Use when linking a tenant for a source or destination, completing OAuth to obtain credentials, or reading credentials after authorization.
---

# Authenticating OAuth taps, targets, and connectors

For AI agents authenticating Singer **taps**, **targets**, or bidirectional **connectors** through Hotglue. Prefer magic links. Check for an existing linked entity before starting OAuth. OAuth is a blocking sub-step — verify, then resume the parent task (`creating-taps`, `creating-targets`, jobs, etc.).

Look up magic links, tenants, linked sources/connectors/targets, and connector roles in Hotglue docs (Docs MCP / search).

## Flow

1. **Prerequisites** — Hotglue API credentials available to the agent (env, PAT, or harness-managed). Entity supported on the flow with OAuth configured. Know the entity role (source / connector / target) and id.
2. **Tenant** — Use a dedicated non-production test tenant unless explicitly prompted otherwise. Do not reuse shared or production tenants.
3. **Check existing link** — Call the linked-entities API (`linkedSources`, `linkedConnectors`, or `linkedTargets`). Do not print raw responses and omit sensitive fields (`config`, tokens, secrets, auth URLs). If the entity is already linked, skip OAuth and resume.
4. **Magic link** — Create a magic link for the tenant, flow, and entity via the Hotglue API. Treat the URL as sensitive.
5. **Open** — Prefer automatic browser opening, otherwise give the URL to the user. Wait for the user to complete connection.
6. **Verify** — Re-check the linked entity (omit sensitive fields). Confirm linked and not disconnected.
7. **Credentials** — When local testing needs config, fetch linked config into a git-ignored file (e.g. `.secrets/`). Never log tokens or full linked JSON.
8. **Resume** — Continue the original task immediately after verification.

## Security

- Never expose access tokens, refresh tokens, client secrets, or authorization codes in logs or chat.
- Never bypass MFA, CAPTCHA, or provider security controls.
- Do not commit OAuth URLs or credentials to Git.
