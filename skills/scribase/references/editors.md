# Connecting an agent

All options use the same two values: the API URL
(`https://api.scribase.com`, or your self-hosted control API) and a token.
Never ask the user to paste a secret key to get one: clients with MCP OAuth
(Claude Code, Cursor, VS Code) open a browser consent page on first connect,
and `scribase login --json` gets a scoped agent token (`scb_agt_...`) after
the user approves a code in the browser. A pasted `scb_agt_` token is fine to
use. Keep the token in an environment variable, never in a committed file.

## Remote MCP (recommended; nothing to install)

Endpoint: `https://api.scribase.com/mcp` (streamable HTTP, bearer auth).

Claude Code:

```bash
claude mcp add --transport http scribase https://api.scribase.com/mcp
# or, with a token already in the environment:
claude mcp add --transport http scribase https://api.scribase.com/mcp \
  --header "Authorization: Bearer $SCRIBASE_ACCESS_TOKEN"
```

Or install the plugin (skills + MCP together) from a checkout of the
Scribase repository:

```bash
claude plugin marketplace add ./agents/plugin
claude plugin install scribase@scribase
```

Cursor (`.cursor/mcp.json`):

```json
{
  "mcpServers": {
    "scribase": {
      "url": "https://api.scribase.com/mcp",
      "headers": { "Authorization": "Bearer ${env:SCRIBASE_ACCESS_TOKEN}" }
    }
  }
}
```

Windsurf (`~/.codeium/windsurf/mcp_config.json`) uses `serverUrl` instead of
`url`; the rest is the same.

Codex (`~/.codex/config.toml`):

```toml
[mcp_servers.scribase]
url = "https://api.scribase.com/mcp"
bearer_token_env_var = "SCRIBASE_ACCESS_TOKEN"
```

## Local stdio server

Build once from the repository: `cargo install --locked --path crates/mcp`.
The schema tools (`schema.propose`, `policy.test`, `schema.apply`,
`sql.explain`) run on a scratch Postgres the server starts itself (needs
`initdb`, `pg_ctl`, `psql` on `PATH`) or on `PGHOST`; the control tools use
`SCRIBASE_API_URL` and `SCRIBASE_ACCESS_TOKEN`.

```json
{
  "mcpServers": {
    "scribase": {
      "command": "scribase-mcp",
      "env": { "SCRIBASE_API_URL": "https://api.scribase.com", "SCRIBASE_ACCESS_TOKEN": "${env:SCRIBASE_ACCESS_TOKEN}" }
    }
  }
}
```

## Project rules

Drop the matching file from `agents/templates/` into the project root:
`AGENTS.md` (Codex, and most agents), `CLAUDE.md`, `.cursor/rules/scribase.mdc`,
`.windsurf/rules/scribase.md`.
