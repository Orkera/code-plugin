---
name: deploy
description: Deploy or update the user's Dockerized app with Orkera. Use when asked to deploy, publish, put online, or make an app live on Orkera, and when diagnosing an Orkera deployment.
---

# Deploy with Orkera

Use the bundled Orkera MCP tools to deploy the user's project. The coding agent owns the source code and all Git operations. Orkera provisions the workspace, builds the exact pushed commit, and runs it.

## Before changing anything

- Inspect the project and Git status first. A root-level `Dockerfile` is required; make or fix one only as part of the requested deployment.
- Identify the app's HTTP port and make sure it listens on `0.0.0.0` inside the container. Pass that exact listening port as `http_port` to `deploy`; `0.0.0.0` is only the bind address, not the public URL.
- Check that the Dockerfile starts the web server in the foreground so the container stays alive. Do not assume Orkera guesses the app's startup command.
- Preserve unrelated user changes. Commit only the deployable app changes; never push unrelated or uncommitted work.
- Never print, commit, place in a persistent Git remote URL, or otherwise disclose a Git credential returned by Orkera. Use a temporary `GIT_ASKPASS` helper for Git HTTPS operations and delete it immediately after the push. Keep the Git remote free of credentials.
- Use `deploy(env_vars={...})` only for additional, non-sensitive runtime configuration. The agent can see every value passed to this MCP tool, so do not request or transmit passwords, API keys, tokens, or other secrets through chat or `env_vars`.
- If deployment requires a secret that is not configured, tell the user to add it through the Orkera UI when secret management is available, then pause any step that depends on it. If that UI is not available yet, explain that secret injection is not supported through MCP and pause. Do not ask the user to paste the secret into chat. Names beginning with `ORKERA_` are reserved.

## Community listing and app descriptions

- A running app with a public URL appears automatically in Orkera's Community apps page. The gallery preview loads the app's real public URL in an iframe; it is not generated from a screenshot and needs no separate upload or MCP call.
- Before creating a workspace, inspect the project to understand what the app does and who it is for. Write one clear, factual sentence of at most 240 characters and pass it as `description` to `create_workspace`. The description is public, so do not include private details, secrets, or claims the app does not support. If the purpose is unclear, ask the user or leave it blank rather than inventing one.
- If the workspace already exists, call `update_workspace` with the same concise description after inspecting the project. Update it if the app's purpose changes materially.
- Make the app's root page useful and responsive at a narrow viewport, since that is what the gallery embeds. Avoid a blank root page or a login wall when the app is intended to be publicly previewable. Do not weaken CSP, frame, or authentication protections just to force an embed; if the app blocks framing, keep the direct app link working and tell the user that its preview may be limited.

## Workspace and Git

1. Orkera defaults to one workspace per account; selected accounts may have administrator-granted higher limits. Call list_workspaces first. If the user supplies an existing workspace ID, inspect it with `get_workspace` and reuse it only when it is clearly for this app. Otherwise call `create_workspace`; if it reports the workspace limit, use the returned workspace details to explain the available recovery choices. Never delete a workspace to get around the limit.
2. If a new workspace is appropriate, call `create_workspace`. Its response includes the Git URL and an initial short-lived credential under `git`; configure the temporary `GIT_ASKPASS` helper with that exact credential and use it for the push. Do not immediately call `get_git_credentials`.
3. `get_git_credentials` **rotates** the repository credential: it revokes the previously issued token and returns a replacement. Use it only for an existing workspace or after a real credential failure/expiry. When it is called, discard the old temporary helper, recreate it with the returned username and token, then retry the Git operation once. Never reuse the former helper or call this tool repeatedly.
4. A first HTTP `401` during a Git HTTPS exchange is normally Git's authentication challenge, not proof that Orkera rejected the credential. Treat authentication as failed only if Git fails after invoking the configured `GIT_ASKPASS` helper (for example, a final `Authentication failed` error).
5. Add the returned Git URL as a remote, commit the intended source changes, and push the exact commit to the repository's `main` branch. Do not assume that creating a workspace pushes or builds the user's local code.
6. Record the full commit SHA locally. Pass that SHA to `deploy`; do not build a branch name or a different commit.

## Deploy and follow the operation

1. If SQLite storage is needed, call create_db before deploy. The app reads ORKERA_DB_URL; never pass database credentials through env_vars.
2. Call deploy(workspace_id, commit_sha, http_port, env_vars={...}) with the exact pushed SHA and explicit container HTTP port. Use only non-sensitive configuration.
3. Follow get_operation with the returned operation ID, next_log_cursor and optional wait_seconds up to 30. Do not resubmit while the operation is active.
4. Only report success after state=succeeded and health_check=passed. Return public_url and commit_sha. On failure inspect logs; failed updates attempt runtime restoration but cannot reverse database changes.
5. A frontend and backend share one workspace when packaged into one root Dockerfile and public port. Orkera does not orchestrate Compose services.

## Recovery and lifecycle

- For a workspace-limit or other API error, use its returned details and `list_workspaces` to explain the available recovery choices. Ask before reusing an ambiguous workspace or deleting anything.
- `stop` stops compute but preserves the workspace, Git repository, release history, and SQLite volume. Only call it when the user asks to stop the app.
- `delete_workspace` is destructive. Call it only when the user explicitly asks to delete the workspace/app; explain the consequence if needed.
- delete_workspace returns an operation ID. Follow get_operation until deletion finishes before creating a replacement. Deletion is permanent; do not silently reuse an unrelated app.
- On success, report the public URL, workspace state, and deployed commit in a concise summary.
