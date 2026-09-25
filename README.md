# Baton — MCP setup

Let your AI assistant work in a [Baton](https://baton.dudko.dev) workspace.
Baton is a git-native API client (a Postman alternative): requests, collections
and environments are plain YAML files in a repository you own. The MCP server
runs **locally**, edits those files the same way the app does, and every change
shows up in the app instantly and goes through git review like any other change.

- **Server:** `baton`
- **Transport:** stdio (local process)
- **Package:** [`@dudko.dev/baton-mcp`](https://www.npmjs.com/package/@dudko.dev/baton-mcp) — Node.js 24+
- **Auth:** none — it runs locally with your user's permissions; secrets never pass through it.

## What you can do

By default the assistant can read and write the workspace but **not send requests**:

| Tool | What it does |
|---|---|
| `workspace_info` | Orientation: name, settings, environments, collection tree. |
| `list_requests` | Requests with method, URL and tags. |
| `get_request` | A parsed request plus its raw YAML — secrets appear as references only. |
| `upsert_request` | Write a request as deterministic YAML. |
| `move_request` | Move or rename a request, keeping its id. |
| `delete_request` | Delete a request. |
| `upsert_collection` | Create or update a collection or folder. |
| `list_environments` | Environments, with secret **names** only. |
| `upsert_environment` | Write an environment (secret names only). |
| `get_script / upsert_script` | Read and write pre-request and test scripts. |
| `run_request / run_collection` | Execute a request or a collection — **only with `--allow-run`**. |

---

## Install in Claude Code (plugin)

```
/plugin marketplace add dudko-dev/baton-mcp-setup
/plugin install baton@baton
/reload-plugins
```

The plugin starts the server in the current project directory, so open Claude
Code in (or under) your Baton workspace. To point it elsewhere, or to let the
assistant execute requests, add the server by hand instead (below) with
`--workspace <dir>` and/or `--allow-run`.

> Prefer not to use the marketplace? Add the server directly:
> ```
> claude mcp add baton -- npx -y @dudko.dev/baton-mcp --workspace /path/to/workspace
> ```

---

## Connect from other clients

### Claude Desktop

`claude_desktop_config.json` (Settings → Developer → Edit Config):

```json
{
  "mcpServers": {
    "baton": {
      "command": "npx",
      "args": [
        "-y",
        "@dudko.dev/baton-mcp"
      ]
    }
  }
}
```

### Cursor

`~/.cursor/mcp.json` or `.cursor/mcp.json`:

```json
{
  "mcpServers": {
    "baton": {
      "command": "npx",
      "args": [
        "-y",
        "@dudko.dev/baton-mcp"
      ]
    }
  }
}
```

### VS Code (GitHub Copilot / MCP) and other clients

```json
{
  "servers": {
    "baton": {
      "type": "stdio",
      "command": "npx",
      "args": [
        "-y",
        "@dudko.dev/baton-mcp"
      ]
    }
  }
}
```

---

## Safety

- **Running requests is off by default.** Add `--allow-run` to the args only when
  you want the assistant to fire requests — editing files should never silently
  send traffic.
- **Secrets stay out of the workspace.** The server reads and writes secret
  *names* and `{{secret:name}}` references only; a plaintext value where a
  reference belongs is refused.
- **The server never touches git.** Commits stay yours.

| Flag | Meaning |
|---|---|
| `--workspace <dir>` | Workspace folder (default: current directory). |
| `--allow-run` | Let the agent execute requests. Off by default. |

---

## Troubleshooting

- **"not a Baton workspace" / the client says "Connection closed"** — the server
  exits at start when the folder has no `baton.yaml`. Open Claude Code in your
  workspace, or add the server by hand with `--workspace /path/to/workspace`.
- **`run_request` refuses** — add `--allow-run`.
- **`npm warn EBADENGINE … required: { node: '>=24' }`, then nothing** — the
  server needs Node.js 24+; `node --version` in the shell your client starts.

## Support

- Website: https://baton.dudko.dev
- Contact: siarhei@dudko.dev
