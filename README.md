# Baton — MCP setup

Let your AI assistant work in a [Baton](https://baton.dudko.dev) workspace.
Baton is a git-native API client (a Postman alternative): requests, collections
and environments are plain YAML files in a repository you own. The MCP server
runs **locally** with one configuration and two modes, picked at start:

- **File mode** — the Baton app is not open on the workspace: an edit is a
  write to the YAML file, the same bytes the app would write, and goes through
  git review like any other change.
- **App mode** — the desktop app is open on the workspace: an edit lands as a
  **draft** on the request's tab for you to accept or discard, a send runs
  through the app with the app's secrets, a send into a production environment
  or a delete waits for your confirmation, and the assistant sees which request
  and environment you have open.

- **Server:** `baton`
- **Transport:** stdio (local process)
- **Package:** [`@dudko.dev/baton-mcp`](https://www.npmjs.com/package/@dudko.dev/baton-mcp) — Node.js 24+
- **Auth:** none — it runs locally with your user's permissions; secrets never pass through it.

## What you can do

In file mode the assistant can read and write the workspace but **not send
requests** unless you add `--allow-run`. In app mode every send goes through
the app, on your screen.

| Tool | What it does |
|---|---|
| `workspace_info` | Collections, environments, schedules, load issues, and whether the app is connected. |
| `list_requests` | Requests with method, URL and tags — optionally under one collection. |
| `get_request` | A parsed request, its raw YAML, and where its auth comes from — secrets appear as references only. |
| `upsert_request` | Write a request as deterministic YAML (file mode) or propose it as a draft in the app. |
| `move_request` | Move or rename a request, keeping its id and its scripts. |
| `delete_request` | Delete a request; in app mode, after you confirm. |
| `upsert_collection` | Create or replace a collection or folder: variables, auth, order. |
| `list_environments` | Variables and secret **names**, with each secret's source. |
| `upsert_environment` | Write an environment (secret names only; a secret may map to Vault). |
| `get_script / upsert_script` | Read and write pre-request, test and shared library scripts. |
| `run_request / run_collection` | Execute a request or every request under a collection — file mode needs `--allow-run`; app mode sends through the app. |
| `list_examples / save_example` | A request's saved responses, and save the last response as one. |
| `list_mocks / start_mock / stop_mock` | Stand a mock up from `mocks/<name>.yaml` on `127.0.0.1`, answering from the examples. |
| `app_state / select_request` | App mode only: what is open in the app, and open a request there. |

In app mode the resources `baton://app/state` and `baton://app/selection` are
subscribable, so the assistant learns of a click, an accepted draft or a
finished send as it happens.

---

## Install in Claude Code (plugin)

```
/plugin marketplace add dudko-dev/baton-mcp-setup
/plugin install baton@baton
/reload-plugins
```

The plugin starts the server in the current project directory, so open Claude
Code in your Baton workspace (the folder with `baton.yaml`). Open the Baton app
on the same workspace to get app mode. To point the server elsewhere, or to let
the assistant execute requests in file mode, add it by hand instead (below)
with `--workspace <dir>` and/or `--allow-run`. In the app, Settings → Agents
writes a `.mcp.json` into the workspace for you.

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

- **In file mode, running requests is off by default.** Add `--allow-run` to the
  args only when you want the assistant to fire requests — editing files should
  never silently send traffic.
- **In app mode, you stay in the loop.** Edits arrive as drafts you accept or
  discard; a send into a production environment and every delete wait for your
  confirmation in the app.
- **Secrets stay out of the workspace.** The server reads and writes secret
  *names* and `{{secret:name}}` references only; the values live in the OS
  secure store or in Vault, and a plaintext value where a reference belongs is
  refused. Responses are masked by value, bodies truncated, and
  `Authorization`, `Cookie`, `Set-Cookie` and `X-API-Key` replaced by name.
- **The server never touches git.** Commits stay yours.

| Flag | Meaning |
|---|---|
| `--workspace <dir>` | The workspace root — the folder holding `baton.yaml` (default: current directory). |
| `--allow-run` | Let the agent execute requests in file mode. Off by default; app-mode sends go through the app. |
| `--agent <name>` | What the app calls this agent on its banners and confirmations. |

---

## Troubleshooting

- **"not a Baton workspace" / the client says "Connection closed"** — the server
  exits at start when the folder has no `baton.yaml`. Open Claude Code in your
  workspace, or add the server by hand with `--workspace /path/to/workspace`.
- **`run_request` refuses** — the server is in file mode: add `--allow-run`, or
  open the Baton app on the workspace and restart the server for app mode.
- **Edits are written straight to files although the app is open** — the mode
  is decided when the server starts. Restart the MCP server in your client
  after opening the app on that workspace.
- **`npm warn EBADENGINE … required: { node: '>=24' }`, then nothing** — the
  server needs Node.js 24+; `node --version` in the shell your client starts.

## Support

- Website: https://baton.dudko.dev
- Contact: siarhei@dudko.dev
