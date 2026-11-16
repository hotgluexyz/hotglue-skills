---
name: authenticating-oauth
description: Authenticate OAuth taps, targets, and connectors for harnessed AI agents (Claude Code, Codex, Cursor, and similar) via Hotglue magic links and browser automation. Use when linking a tenant for a source or destination, completing OAuth from an independent agent environment, or reading credentials after authorization.
---

# Authenticating OAuth taps, targets, and connectors

For harnessed AI agents in an independent environment (local shell, container, remote sandbox). Applies whenever a Singer **tap**, **target**, or bidirectional **connector** uses OAuth — the Hotglue linking path is the same; only the linked-entity API and what you resume afterward differ.

Prefer Hotglue magic links. Check for existing linked tokens before starting OAuth. OAuth is a blocking sub-step — verify credentials, then resume the original task (`creating-taps`, `creating-targets`, jobs, etc.).

Docs: [Magic Links](https://docs.hotglue.com/connection-methods/magic-links), [Tenants](https://docs.hotglue.com/key-concepts/tenants), [connector roles](https://docs.hotglue.com/key-concepts/connectors/connectors#connectors-vs-sources-vs-targets).

## Prerequisites

- Secret API key in `$HOTGLUE_SECRET_API_KEY` (never paste into chat or logs).
- Flow has a supported OAuth entity (source, connector, or target) with Client ID, Client Secret, and redirect URI configured.
- Know the entity role and id (`ENTITY_ID`): source (tap / read side), connector (bidirectional), or target (write side). See `managing-available-connectors`.
- API host matches the UI host (default `https://api.hotglue.com`; see `running-hotglue-remote-jobs`).
- Note the pre-auth goal and next steps before starting OAuth so you can resume after verification.
- Check for existing tokens first (section 2). Only initiate OAuth when the tenant is not already linked with usable credentials.

## 1. Tenant naming

Prefer a stable test tenant such as `agent-<entity>-<yyyyMMdd>`, or reuse an existing tenant. Tenants are created on first successful link.

## 2. Check existing tokens first

Before generating a magic link or opening a browser, inspect the tenant's linked entity for this role:

| Role | GET linked entities |
|---|---|
| Connector | `/v2/${ENV_ID}/${FLOW_ID}/${TENANT}/linkedConnectors` |
| Source (tap) | `/v2/${ENV_ID}/${FLOW_ID}/${TENANT}/linkedSources` |
| Target | `/v2/${ENV_ID}/${FLOW_ID}/${TENANT}/linkedTargets` |

```bash
curl --silent --request GET \
  --url "https://api.hotglue.com/v2/${ENV_ID}/${FLOW_ID}/${TENANT}/${LINKED_PATH}" \
  --header "x-api-key: ${HOTGLUE_SECRET_API_KEY}"
```

Use `LINKED_PATH` = `linkedConnectors`, `linkedSources`, or `linkedTargets` per the table.

- If the entity is already linked with tokens/config present, skip OAuth. Read credentials into `.secrets/` if needed (section 6) and resume the original task.
- Project output to non-secret fields only — never echo full linked configuration.
- The Connect UI may still show **Connect** after tokens exist; trust the API/config, not the button label.
- Re-run OAuth only when the user requests a re-link, tokens are missing/expired/invalid, or a minimal authenticated check fails (e.g. tap discover/sync or target write).

## 3. Generate a magic link

Entity-scoped when possible (`entityId` is the available/supported entity id — same for taps and targets):

```bash
curl --request POST \
  --url "https://api.hotglue.com/${ENV_ID}/magicLink" \
  --header "Content-Type: application/json" \
  --header "x-api-key: ${HOTGLUE_SECRET_API_KEY}" \
  --data "{\"tenant\":\"${TENANT}\",\"flow\":\"${FLOW_ID}\",\"entityId\":\"${ENTITY_ID}\",\"type\":\"connector\"}"
```

Response includes a `url` (e.g. `https://connect.hotglue.com/?token=...`; QA may use `qa.connect.hotglue.xyz`). Treat the URL as sensitive.

## 4. Open the link

1. Inspect available tools (browser MCP/extension, open commands). Verify they are invokable — do not assume an installed extension works.
2. **Preferred:** open the magic link with native browser automation when available.
3. **Fallback:** print the URL clearly and/or write it to a workspace-relative file (e.g. `oauth-url.txt` at the workspace root). Tell the user to open it and complete Connect plus provider consent.
4. MFA, CAPTCHA, and consent stay manual — wait for the user.
5. If a tool shells out via `$BROWSER` and would fail without a GUI, optionally point `BROWSER` at a small script that writes `"$1"` to `oauth-url.txt` instead of opening a browser. Only when that mechanism exists; do not require a specific container setup.
6. Optional host setup: if the harness prompts to install a browser extension for automation, that is host-side — not an environment prerequisite.

Never extract browser cookies, passwords, or unrelated session tokens.

## 5. Verify link success

After the user confirms auth (or after a short poll window), re-check the same linked-entity GET as section 2 for this role.

Confirm the entity appears with tokens/config present. Project output to non-secret fields (`id`, connection status) — never echo full linked configuration.

The Connect UI may still show **Connect** after tokens are written. Trust the API/config, not the button label.

## 6. Read credentials safely

When local tap or target testing needs config, fetch linked config into `.secrets/` and keep it git-ignored. Never log access tokens, refresh tokens, client secrets, authorization codes, or full linked JSON.

Delete temporary URL files (e.g. `oauth-url.txt`) after authentication succeeds.

## 7. Resume the original task

OAuth is not the end of the task unless the user only asked for authentication.

- After verification (or after skipping OAuth because tokens already exist), immediately continue — e.g. tap discover/sync (`creating-taps`), target sinks/local run (`creating-targets`), deploy, or remote jobs.
- Do not end the turn with only “OAuth complete” when parent work remains.
- Do not abandon the parent task because a GUI browser is unavailable — use the best supported open/print path, verify, then resume.
- On failure, troubleshoot and retry; ask the user only for interactive challenges or missing env/config.

## Provider-direct OAuth (rare with magic links)

When authenticating a tap or target against the provider without a Hotglue magic link:

1. Check for existing usable credentials first (linked config or local `.secrets/`); skip OAuth if they already work.
2. Generate the provider authorization URL only when needed.
3. Open or print it using the same capability rules as above.
4. Ensure the registered redirect URI matches exactly and the callback is reachable from the agent environment (publish or forward the callback port when the listener runs inside an isolated environment). Preserve OAuth state and PKCE where applicable.
5. Complete the code exchange, verify with a minimal authenticated request (tap discover or a trivial API call), then resume.

## Errors

| Problem | Action |
|---|---|
| Tokens/config already linked | Skip OAuth; read into `.secrets/` if needed and resume |
| Wrong linked-entity path | Use `linkedSources` / `linkedConnectors` / `linkedTargets` for the entity role |
| Browser tool unavailable | Print URL and/or write workspace `oauth-url.txt` |
| Browser navigation fails | Same fallback; ask user to open the URL |
| User interaction required | Wait for Connect / MFA / CAPTCHA / consent |
| `$BROWSER` required by a tool | Optional URL-capture helper; else print the URL |
| OAuth URL file missing | Regenerate the magic link or auth URL |
| Localhost callback unreachable | Fix reachability and redirect URI match |
| Invalid redirect URI | Check supported-entity OAuth app config |
| Token exchange / link fails | Inspect OAuth or API errors without exposing secrets |
| UI still shows Connect | Re-check the role’s linked-entity API for tokens/config |
| Credentials disappear | Re-fetch linked config; verify secure persistence |

## Security

- Never expose access tokens, refresh tokens, client secrets, or authorization codes in logs or chat.
- Never bypass MFA, CAPTCHA, OAuth state checks, or provider security controls.
- Use sandbox credentials and least-privilege scopes when possible.
- Do not commit OAuth URLs or credentials to Git.
- Never claim browser automation or auth succeeded without verifying linked config or a token exchange.
