# Scribase plugin for coding agents

Scribase is a hosted backend (Postgres, auth, storage, realtime and edge functions) that a coding agent can create and operate. It works with supabase-js. This plugin connects Claude Code, Cursor and Codex to the hosted Scribase MCP server and adds skills that teach the agent how to use it safely.

## What is inside

- `.mcp.json`: one remote MCP server, `https://api.scribase.com/mcp` (Streamable HTTP). Nothing runs on your machine.
- `skills/scribase`: create a project, add tables with row-level security, sign-in, storage, realtime, edge functions, preview branches and generated types.
- `skills/scribase-safe-change`: ship a schema or policy change through a preview branch, a row-level-security test and an explicit confirm.
- `rules/scribase.mdc`: the same guidance as a Cursor project rule.
- `.claude-plugin/`, `.cursor-plugin/`, `.codex-plugin/`: the plugin manifests for each editor.

## Install in Claude Code

```bash
claude plugin marketplace add caelum0x/scribase-agents
claude plugin install scribase@scribase
```

To add only the MCP server:

```bash
claude mcp add --transport http scribase https://api.scribase.com/mcp
```

## Sign-in

The first time your editor connects, it opens the Scribase consent page in your browser. You approve there, and the agent gets its own scoped, expiring, revocable token. You never paste a key into a chat. Editors that need a fixed header are covered in the setup guide: https://docs.scribase.com/docs/agents/setup

## What the plugin sends, and where

The plugin has no hooks, scripts or telemetry. The skills are text instructions. The only network traffic is the MCP connection to `api.scribase.com`, carrying the requests your agent makes and the token you approved. Reads run immediately; every write returns the exact request and waits for your explicit confirm, and a schema change needs a passing row-level-security test.

## Links

- Agent setup: https://scribase.com/agents
- MCP server: https://scribase.com/mcp
- Docs: https://docs.scribase.com/docs/agents
- Pricing: https://scribase.com/pricing
- Support: support@scribase.com
