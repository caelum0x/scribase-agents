---
name: scribase
description: Create and operate a Scribase backend (Postgres, Auth, Storage, Realtime, Edge Functions; works with supabase-js) from a coding agent. Use when the user asks for a database, backend, auth/sign-in, file uploads, realtime, edge functions, migrations, RLS policies, generated types, or mentions Scribase, SCRIBASE_API_URL, scb_agt_, scb_pat_ or sbk_ tokens, scribase login, or an existing project to import (`scribase import supabase`). Covers signing in without secret keys (scribase login device flow, paste-safe scb_agt_ tokens), the one-line MCP install, creating a project and writing .env, tables with RLS, Apple/Google sign-in, storage, realtime broadcast, functions deploy, branch-per-task, and types.
---

# Scribase

Scribase is an application cloud that works with supabase-js. The **data plane**
(`/auth/v1`, `/rest/v1`, `/storage/v1`, `/realtime/v1`, `/functions/v1`) speaks
the same HTTP API, so apps use `@supabase/supabase-js` unchanged. The
**control plane** is the `/v1` management API (orgs, projects, environments,
branches, migrations, keys, backups), reached through the MCP server, the
`scribase` CLI, or the `scribase` SDK.

Everything here is doable without opening the console, except sign-up and the
one-time $1 card check, which a human does once.

## 0. Signing in (never ask the user for a key)

**Never ask the user for API keys, `service_role` / secret keys, database
passwords or personal access tokens.** Nobody needs to paste a secret into
this chat. Instead:

```bash
scribase login --json          # or: npx scribase login --json
```

It prints a line such as
`{"event":"device_authorization","verification_uri_complete":"https://console.scribase.com/device?user_code=SCRB-7F3K-Q2MX","user_code":"SCRB-7F3K-Q2MX",...}`.
Show the user the link and the code, ask them to approve it in the browser,
and wait for the final line (`{"ok":true,"status":"approved",...}`). The CLI
stores the token itself (OS keychain, else `~/.config/scribase/credentials`,
mode 0600) and never prints it. If your shell shows output only when a
command ends, run `scribase login --json --no-wait`, show the link, and run
`scribase login --json` again once the user has approved. `access_denied`
means the user said no: ask, do not retry in a loop.

Other paths:

- **No account, or a throwaway backend:** `npx scribase init --temp --yes
  --json` creates a 72-hour project with its own scoped token and a claim
  link for the user. `npx scribase init --yes --json` does the full setup
  and signs in through the same browser approval when needed.
- **Claude Code MCP:** `claude mcp add --transport http scribase
  https://api.scribase.com/mcp` (the browser consent page opens on first
  connect; no header, no key).
- **A pasted `scb_agt_` token is safe to use.** Agent tokens are designed to
  be shared with coding agents: scoped (usually one project), expiring (8
  hours by default), revocable and audited, and without the secret keys
  unless `keys:secret` was granted. Put it in `SCRIBASE_ACCESS_TOKEN` or the
  gitignored `.env`; never commit or echo it. The segment before the last
  `_` summarizes it: `1p-rw-sqlw-mig-fn-x2609260412` is one project,
  read/write, read-write SQL, migrations, functions, expiring 2026-09-26
  04:12 UTC.
- **Anything else secret that gets pasted** (`service_role`, `sb_secret_`,
  a database password or connection string, `scb_pat_`, `sbk_`): do not
  repeat it or use it. Tell the user to revoke or rotate it and run
  `scribase login` instead.

The API URL defaults to `https://api.scribase.com`; set `SCRIBASE_API_URL`
(or pass `--api-url`) to point at a different control plane.

Other agents: see `references/editors.md` (Cursor, Codex, Windsurf, local
stdio server). CLI install: `cargo install --locked --path crates/scribase-cli`
from the Scribase repository, then `scribase doctor` must pass.

If neither the MCP tools nor the CLI are available, stop and tell the user the
exact setup line above; do not guess endpoints, and do not ask for a key.

## 1. Ground rules (binding)

- **Secrets never appear in output, logs, commits or chat.** Write them to
  `.env` (and make sure `.env` is in `.gitignore`); refer to them by name.
- **Only the anon / publishable key goes into client code.** The
  `service_role` / `sb_secret_` key is for servers and functions only.
- **Every mutating MCP tool refuses without `confirm: true`** and returns the
  exact request instead. Show that request to the user for anything touching
  `production`; you may confirm on your own previews.
- **`database.query` is read-only.** Schema changes go through migrations
  (`migrate.run` / `scribase migrate`), never ad-hoc DDL against production.
- **Every table in an exposed schema gets RLS in the same migration** that
  creates it. No table ships with RLS off.
- **Destructive operations** (`environment.delete`, `scribase project delete`,
  `scribase keys rotate`, `branch.merge` into production, restores) need the
  user's explicit go-ahead for that specific operation, in this session.
- Lifecycle calls return an **operation**. Poll `operation.get` /
  `scribase operation get` until `succeeded`; a 202 is not success.
- Use stable idempotency keys derived from the task (`create-pr-42`), so a
  retry never creates a second resource.
- Run `scribase help` (or list MCP tools) for the exact surface. Do not invent
  commands, flags or tools.

## 2. First backend in one pass

Identifiers: `ORG` (from `org.list` / `scribase org list`), a lowercase
`PROJECT` such as `my-app`, `REGION` (hosted Scribase runs in
`eu-frankfurt`).

MCP: `org.list` → `project.put {organization_id, project_id, region}` (confirm)
→ `environment.create {kind: "production", data_mode: "schema_only",
environment_id: "production", idempotency_key: "<project>-production"}`
(confirm) → `operation.get` until `succeeded`.

CLI:

```bash
scribase org list
scribase project put "$ORG" my-app eu-frankfurt
scribase env create "$ORG" my-app production production schema_only eu-frankfurt my-app-production
scribase operation get "$ORG" <operation_id>          # until succeeded
scribase apps switch "$ORG" my-app --format dotenv >> .env   # URL + anon key only
```

`apps switch --format expo` prints `EXPO_PUBLIC_SUPABASE_URL` /
`EXPO_PUBLIC_SUPABASE_ANON_KEY`; `--format dotenv` prints
`SUPABASE_URL` / `SUPABASE_ANON_KEY`. For Next.js rename them to
`NEXT_PUBLIC_SUPABASE_URL` / `NEXT_PUBLIC_SUPABASE_ANON_KEY`. Server-side keys:
`scribase keys show "$ORG" my-app production --service-role --json`, piped
straight into the server's secret store, never printed.

Then the client:

```ts
import { createClient } from '@supabase/supabase-js'
export const supabase = createClient(process.env.SUPABASE_URL!, process.env.SUPABASE_ANON_KEY!)
```

## 3. Common tasks

| Task | Read |
|---|---|
| Tables with RLS, migrations, generated types | `references/database.md` |
| Sign in with Apple / Google (`signInWithIdToken`), email | `references/auth.md` |
| File upload, buckets, signed URLs | `references/storage.md` |
| Realtime broadcast, presence, row changes | `references/realtime.md` |
| Edge functions: write, deploy, secrets, logs | `references/functions.md` |
| Branch per task, preview, merge | `references/branching.md` (and the `scribase-safe-change` skill for production schema changes) |
| Import an existing project | `scribase import supabase --project-ref <ref> --org ... --project ... --env production --dry-run` first, then without `--dry-run`; MCP `import.supabase` + `import.status` |
| Debug | `logs.tail`, `insights.get`, `database.tables`, `auth.users`, `functions logs` |

## 4. Report back

After provisioning or changing anything, tell the user: what was created
(org / project / environment ids), which files were written (for example
`.env`, `supabase/migrations/...`), the operation ids, and anything that still
needs a human (enabling a paid plan, an Apple Services ID, approving a
production merge). Never include key values in the report.
