# Baton — MCP setup

Public repo. Nothing here runs on a server; it is what the registry, the plugin
marketplace and people read:

| File | Read by | A change reaches them when |
|---|---|---|
| `server.json` | the MCP Registry, as `dev.dudko/baton` | CI publishes it on merge to `main` — and only under a **new** `version` |
| `.claude-plugin/plugin.json` | Claude Code users of the `baton` plugin | its `version` changes: an installed plugin is pinned to it |
| `.claude-plugin/marketplace.json`, `README.md` | the marketplace listing, people | right away |

## Bump versions in the same change — always

Before finishing any change in this repo, check both:

1. **Touched `server.json`?** Bump its `"version"`. Registry versions are
   immutable: new content under an already-published version fails CI
   (`cannot publish duplicate version`), and nothing reaches the registry until
   the version moves.
   Here `version` and `packages[0].version` are both the npm release of
   `@dudko.dev/baton-mcp` and stay equal. That release must already be on npm with
   `"mcpName": "dev.dudko/baton"` in its package.json — release on npm first, then bump
   here. **Every new `@dudko.dev/baton-mcp` release needs a bump here too**, or the
   registry keeps listing the old one: CI warns on every PR while npm is ahead,
   and the weekly run fails.
2. **Touched the plugin?** That is anything in `.claude-plugin/plugin.json`
   besides `version`, or `.mcp.json`, `skills/`, `commands/`, `agents/`,
   `hooks/`. Bump `"version"` in `plugin.json`, or users who already installed
   the plugin keep the old one.

The two versions are independent — bump each one whose file changed. Don't add a
`version` to the `marketplace.json` entry: `plugin.json` overrides it anyway.
Semver: patch for wording and metadata, minor for a new capability or field,
major for anything breaking (URL, transport, auth). README-only and CI-only
changes need no bump.

## How publishing works

- `.github/workflows/publish-mcp.yml` runs on a push to `main` that changes
  `server.json`, on a `v<version>` tag (must equal `server.json`'s version), and
  by hand (Actions → Publish to MCP Registry → Run workflow). It publishes only a
  version the registry doesn't have; on an already-published, identical version
  it logs in and stops — so a manual run is also the health check for the key.
- `.github/workflows/validate-server-json.yml` runs on every PR: schema, the
  comparison with what the registry serves, the npm release and its `mcpName`, and the plugin version.
  A weekly run repeats it and fails while npm has a newer `@dudko.dev/baton-mcp` than `server.json` lists.
- The logic is in `.github/scripts/`. The same workflow and script files are in
  every MCP setup repo (miniaccountant, dudko-dev, ireceipt-pro, nacre-work) —
  change them everywhere together.
- Don't publish from a laptop with `mcp-publisher publish`; CI is the one path.

## Registry auth

The namespace `dev.dudko` is proven by DNS for `dudko.dev` (GitHub OIDC only grants
`io.github.*`):

- TXT record on the **apex** `dudko.dev` — not `_mcp-auth.dudko.dev`, the
  registry doesn't look there: `v=MCPv1; k=ed25519; p=<base64 public key>`
- the private half: organization secret `MCP_PRIVATE_KEY` of `dudko-dev` (PEM or
  hex seed), with this repository granted access to it. Every repo under the
  `dev.dudko` namespace uses the same key.
- the publish job prints the public key it derived from the secret and checks it
  against the TXT record, so a mismatch is reported as such.
- never commit `key.pem` or registry tokens; `.gitignore` covers them.

Rotating the key:

```sh
openssl genpkey -algorithm Ed25519 -out key.pem
echo "v=MCPv1; k=ed25519; p=$(openssl pkey -in key.pem -pubout -outform DER | tail -c 32 | base64)"
# 1. add that as a second TXT record on dudko.dev
# 2. put the contents of key.pem into the org secret MCP_PRIVATE_KEY, rm key.pem
# 3. Actions → Publish to MCP Registry → Run workflow; green = the new key works
# 4. delete the old TXT record
```
