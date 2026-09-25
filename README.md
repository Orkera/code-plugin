# Orkera for Claude Code

Deploy a Dockerized app with Orkera directly from Claude Code.

## Install

In Claude Code, add the Orkera marketplace and install the plugin:

```bash
claude plugin marketplace add Orkera/code-plugin
claude plugin install orkera@orkera
```

In Codex:

```bash
codex plugin marketplace add Orkera/code-plugin
codex plugin add orkera@orkera
```

The plugin adds the hosted Orkera MCP server. On first use, Claude Code opens the Orkera OAuth sign-in flow; no local token or adapter is required.

## Use

Ask Claude to deploy your project, for example:

```text
Deploy this app on Orkera.
```

Or invoke the deployment skill explicitly:

```text
/orkera:deploy
```

Every project needs a root `Dockerfile`. The agent commits and pushes the application code to the private Git repository Orkera creates, builds the exact commit, starts it, and returns the public URL.

## What the plugin does

- Adds `https://mcp.orkera.dev/mcp` as a remote HTTP MCP server.
- Uses OAuth for authentication.
- Teaches Claude the safe deployment lifecycle: workspace, Git push, build, run, and public port forwarding.

For more information, visit [orkera.dev](https://orkera.dev).
