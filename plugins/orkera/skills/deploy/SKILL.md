---
name: deploy
description: Deploy or update the user's Dockerized app with Orkera. Use when asked to deploy, publish, put online, or make an app live on Orkera, and when diagnosing an Orkera deployment.
---

# Deploy with Orkera

Use the bundled Orkera MCP tools to deploy the user's project. The coding agent owns the source code and all Git operations. Orkera provisions the workspace, builds the exact pushed commit, and runs it.

## Before changing anything

- Inspect the project and Git status first. A root-level `Dockerfile` is required; make or fix one only as part of the requested deployment.
- Identify the app's HTTP port and make sure it listens on `0.0.0.0` inside the container. Pass that exact listening port to `forward_port`; `0.0.0.0` is only the bind address, not the public URL.
- Check that the Dockerfile starts the web server in the foreground so the container stays alive. Do not assume Orkera guesses the app's startup command.
- Preserve unrelated user changes. Commit only the deployable app changes; never push unrelated or uncommitted work.
- Never print, commit, place in a persistent Git remote URL, or otherwise disclose a Git credential returned by Orkera. Use a temporary credential mechanism for Git HTTPS operations and remove it afterward.
- Use `build(env_vars={...})` only for additional, non-sensitive runtime configuration. The agent can see every value passed to this MCP tool, so do not request or transmit passwords, API keys, tokens, or other secrets through chat or `env_vars`.
- If deployment requires a secret that is not configured, tell the user to add it through the Orkera UI when secret management is available, then pause any step that depends on it. If that UI is not available yet, explain that secret injection is not supported through MCP and pause. Do not ask the user to paste the secret into chat. Names beginning with `ORKERA_` are reserved.

## Workspace and Git

1. Call `list_workspaces` before creating a workspace. Orkera V0 allows one workspace per account. Reuse an existing workspace only when it is clearly the workspace for this app; if ownership or intended app is ambiguous, ask before changing it. Never delete a workspace to get around the limit.
2. If a new workspace is appropriate, call `create_workspace`. Its response includes the Git URL and the initial short-lived Git credentials under `git`; use those credentials as-is. Do not immediately call `get_git_credentials`, because that rotates the credential. For an existing workspace, call `get_workspace` and then `get_git_credentials` when fresh Git credentials are needed.
3. Add the returned Git URL as a remote, commit the intended source changes, and push the exact commit to the repository's `main` branch. Do not assume that creating a workspace pushes or builds the user's local code.
4. Record the full commit SHA locally. Pass that SHA to `build`; do not build a branch name or a different commit.

## Build, database, and run

1. Call `build(workspace_id, commit_sha, env_vars={...})`, supplying only additional non-sensitive runtime variables the app needs. The agent sees these values, even though Orkera encrypts them for runtime injection. Never pass secrets this way.
2. Use `get_build` with the returned build ID to inspect status and logs. Continue from its log cursor when fetching more output. Wait for a successful terminal state before running; on failure, diagnose the logs and fix the cause rather than submitting repeated builds unchanged.
3. If the app needs persistent SQLite storage, call `create_db` before `run`. Orkera injects `ORKERA_DB_URL` and `ORKERA_DB_PASSWORD` automatically; the app should read those variables instead of receiving them through `env_vars`. Store SQLite data at the database path supplied by `ORKERA_DB_URL` on Orkera's mounted data volume, not in the container filesystem, which is replaceable between releases. For any other secret the app requires, pause and ask the user to configure it in the Orkera UI when that feature is available; until then, explain the limitation. Never ask them to reveal the secret to the agent.
4. Call `run` once after a successful build. If it fails or returns an unclear result, inspect `get_workspace` and its `last_error` before taking another action. Do not blindly repeat `run`.
5. When the user wants the app publicly reachable, call `forward_port` for its HTTP port and return the resulting public URL. Never forward database ports.

## Recovery and lifecycle

- For a workspace-limit or other API error, use its returned details and `list_workspaces` to explain the available recovery choices. Ask before reusing an ambiguous workspace or deleting anything.
- `stop` stops compute but preserves the workspace, Git repository, release history, and SQLite volume. Only call it when the user asks to stop the app.
- `delete_workspace` is destructive. Call it only when the user explicitly asks to delete the workspace/app; explain the consequence if needed.
- On success, report the public URL, workspace state, and deployed commit in a concise summary.
