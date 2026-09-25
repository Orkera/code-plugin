---
name: deploy
description: Deploy the user's current Dockerized application with Orkera. Use when the user asks to deploy, publish, put online, or make their app live on Orkera.
---

# Deploy with Orkera

Use the bundled Orkera MCP server to deploy the user's application. The user and agent own the local code and Git operations; Orkera only builds and runs the pushed commit.

## Before deployment

- A root `Dockerfile` is required. Add or fix it when the user has asked to deploy.
- Ensure the application listens on `0.0.0.0` and identify its HTTP port.
- Commit the deployable code. Do not push unrelated local changes.
- Treat `ORKERA_*` names as reserved. `build(env_vars=...)` is only for additional application environment variables.
- Do not print, commit, or otherwise disclose Git credentials returned by Orkera.

## Deployment flow

1. Call `list_workspaces` when there is no known Orkera workspace for this app. Reuse a workspace only when the user has identified it or its purpose is unambiguous. If another workspace is already present and a new app is needed, explain the limit and ask the user whether to reuse or delete it.
2. Call `create_workspace` only when a new workspace is appropriate. If it reports a workspace limit, use the returned workspace details or `list_workspaces`; never delete an existing workspace automatically.
3. Call `get_git_credentials`, add the returned remote locally, and push the exact commit to it. Never expose the returned token.
4. Call `build` with the workspace ID and the full Git commit SHA. Use `get_build` to inspect progress and build logs until it reaches a terminal state.
5. If the application needs persistent SQLite storage, call `create_db`. The application receives its database connection through `ORKERA_DB_URL`; do not ask the user for it or set it manually.
6. Call `run` once after a successful build. Check `get_workspace` for its state rather than repeating `run` blindly.
7. Call `forward_port` only for the application HTTP port. Its returned URL is public; never expose a database port.

## Failures and lifecycle

- Report API errors exactly and act on their meaning. Do not retry a failed tool call repeatedly without a state change.
- If a build fails, use `get_build` and fix the Dockerfile or code before creating a new build.
- If starting fails, inspect `get_workspace` and its `last_error` before deciding what to change.
- `stop` stops compute but preserves the workspace, repository, release history, and any SQLite volume.
- `delete_workspace` is destructive: only use it when the user explicitly asks to delete the app/workspace.

When deployment succeeds, give the user the public URL and a compact summary of the workspace state.
