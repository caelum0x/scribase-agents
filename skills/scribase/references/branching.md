# Branch per task

Give every task (feature branch, pull request, experiment) its own preview
environment: a copy of production with sanitized data and a TTL, so a
forgotten preview deletes itself. The agent can confirm mutations on its own
preview; production merges need the human.

```bash
TASK=add-invoices
scribase env create "$ORG" "$PROJECT" "pr-$TASK" preview sanitized "$REGION" "create-pr-$TASK" 24
scribase operation get "$ORG" <operation_id>                    # until succeeded
scribase branch register "$ORG" "$PROJECT" "pr-$TASK" --base production
scribase apps switch "$ORG" "$PROJECT" --env "pr-$TASK" --format dotenv > .env.preview

scribase migrate "$ORG" "$PROJECT" "pr-$TASK" --dir supabase/migrations --apply
scribase branch diff  "$ORG" "$PROJECT" production "pr-$TASK"
scribase branch merge "$ORG" "$PROJECT" production "pr-$TASK" --dry-run
```

Data modes: `sanitized` (copy, then anonymized; the default for previews),
`snapshot` (exact copy; avoid unless the user asks), `schema_only`.

Link the pull request so the console and the PR show each other:
`scribase branch pr "$ORG" "$PROJECT" "pr-$TASK" github acme/app 42`.

Extend a preview: `scribase env renew "$ORG" "$PROJECT" "pr-$TASK" 24 "renew-pr-$TASK"`.

Merge into production only after the user approved the diff and the dry run,
and after a backup:

```bash
scribase backup create "$ORG" "$PROJECT" production "pre-$TASK"
scribase branch merge  "$ORG" "$PROJECT" production "pr-$TASK"
scribase env delete    "$ORG" "$PROJECT" "pr-$TASK" "delete-pr-$TASK"
```

MCP equivalents: `environment.create`, `operation.get`, `branch.register`,
`migrate.run`, `branch.diff`, `branch.merge`, `backup.create`,
`environment.delete`. The `scribase-safe-change` skill walks the full
production flow including the RLS proof.
