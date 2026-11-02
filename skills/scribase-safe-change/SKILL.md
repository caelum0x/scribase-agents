---
name: scribase-safe-change
description: Ship a schema or row-level-security change to a Scribase project safely. Creates a preview branch, applies the migration there, proves RLS isolation with policy.test (the Aegis mutation harness), shows the diff, waits for explicit human confirmation, then merges and cleans up. Use whenever you add or change tables, columns, indexes, functions, or policies on Scribase.
---

# Scribase safe change: preview, test, confirm, merge

Never change production schema directly. Every change goes through this flow,
in order. If any step fails, stop and report; do not skip ahead.

Inputs you need: `ORG`, `PROJECT`, the project `REGION`, a short change
name `CHANGE` (for example `add-invoices`), and the migration files under
`migrations/`.
Credentials come from the environment only (`SCRIBASE_API_URL`,
`SCRIBASE_ACCESS_TOKEN`, or the token `scribase login` stored). Never print
them, and never ask the user to paste a key: run `scribase login --json` if
you are not signed in.

## 1. Create a preview branch

A preview is a copy of production with sanitized data and a TTL, so a forgotten
preview deletes itself.

```bash
PREVIEW="preview-$CHANGE"
scribase env create "$ORG" "$PROJECT" "$PREVIEW" preview sanitized "$REGION" "$PREVIEW" 24
scribase operation get "$ORG" "$OPERATION_ID"      # repeat until state is succeeded
scribase branch register "$ORG" "$PROJECT" "$PREVIEW" --base production
```

With the MCP server: `environment.create` (kind `preview`, data mode
`sanitized`, TTL 24), then `operation.get` until it succeeds, then
`branch.register`. Both mutations need `confirm: true`; creating a
preview is safe to confirm yourself because it cannot touch production.

## 2. Apply the migration to the preview only

```bash
scribase migrate "$ORG" "$PROJECT" "$PREVIEW" --dir migrations           # plan + lint
scribase migrate "$ORG" "$PROJECT" "$PREVIEW" --dir migrations --apply   # apply to the preview
```

If the linter blocks a change (a lock-heavy ALTER, a dropped column still in
use), fix the migration. Do not force it.

## 3. Prove row level security

Every table that holds user data must have RLS enabled and policies that keep
user A out of user B's rows.

- MCP: call `policy.test` with the full schema source. It applies the
  schema to a scratch branch, runs the cross-user isolation matrix, then
  breaks each guarantee on purpose (FORCE removed, USING (true), a dropped
  WITH CHECK) on throwaway branches and requires every break to be caught.
  A pass returns an apply token; a failure lists every leak or missed mutant.
- CLI: run Aegis against the preview. `--mutate` breaks each policy on
  purpose and checks the tests notice, which proves the tests can fail.

```bash
aegis compile aegis.yaml
aegis test
aegis test --mutate
scribase insights "$ORG" "$PROJECT" "$PREVIEW"   # advisors: missing RLS, unindexed FKs
```

A single leak or a surviving mutation is a failure. Fix the policies and
return to step 2.

## 4. Show the change and wait for a human

```bash
scribase branch diff "$ORG" "$PROJECT" production "$PREVIEW"
scribase branch merge "$ORG" "$PROJECT" production "$PREVIEW" --dry-run
```

Present, in one message: the schema diff, the merge dry run, the RLS results
(simulate verdict, Aegis pass and mutation score), and any insights findings.
Then stop and ask for explicit approval to merge into production. Do not
treat silence, an earlier approval, or approval of a different change as
consent. With MCP, `branch.merge` without `confirm` returns the exact
request; show it and wait.

## 5. Merge after approval

```bash
scribase backup create "$ORG" "$PROJECT" production "pre-$CHANGE"   # restore point
scribase branch merge "$ORG" "$PROJECT" production "$PREVIEW"
```

With MCP: `backup.create` then `branch.merge` with `confirm: true`, only
after the human approved in step 4. Poll any returned operation with
`operation.get` until it reaches a terminal state. A 202 is not success.

## 6. Clean up and report

```bash
scribase env delete "$ORG" "$PROJECT" "$PREVIEW" "$PREVIEW-delete"
```

Report: the migration files applied, the RLS evidence, the backup id taken
before the merge, and the operation ids. If anything failed after the merge,
say so first and offer `scribase restore start` from the pre-merge backup.
