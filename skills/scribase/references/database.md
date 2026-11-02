# Database: tables, RLS, migrations, types

## Migrations directory

Keep schema in numbered SQL files. `scribase migrate --dir` reads
`<version>_<name>.sql` (or `.up.sql` + optional `.down.sql`), ordered by the
numeric prefix. Never edit a migration that has been applied; add a new one.

```
supabase/migrations/
  001_init.sql
  002_todos.sql
```

## A table with RLS (the only acceptable shape)

```sql
-- 002_todos.sql
create table public.todos (
  id          bigint generated always as identity primary key,
  user_id     uuid not null default auth.uid() references auth.users (id) on delete cascade,
  title       text not null check (length(title) between 1 and 500),
  done        boolean not null default false,
  created_at  timestamptz not null default now()
);

create index todos_user_id_idx on public.todos (user_id);

alter table public.todos enable row level security;

create policy "todos: owner reads"   on public.todos for select to authenticated using (user_id = (select auth.uid()));
create policy "todos: owner inserts" on public.todos for insert to authenticated with check (user_id = (select auth.uid()));
create policy "todos: owner updates" on public.todos for update to authenticated
  using (user_id = (select auth.uid())) with check (user_id = (select auth.uid()));
create policy "todos: owner deletes" on public.todos for delete to authenticated using (user_id = (select auth.uid()));

grant select, insert, update, delete on public.todos to authenticated;
```

Rules: RLS enabled in the same file; one policy per command; `with check` on
insert/update; index every column a policy filters on; wrap `auth.uid()` in
`(select ...)` so Postgres evaluates it once per statement; never grant to
`anon` unless the data is meant to be public.

## Plan, then apply

```bash
scribase migrate "$ORG" "$PROJECT" "$ENV" --dir supabase/migrations          # lint + plan, changes nothing
scribase migrate "$ORG" "$PROJECT" "$ENV" --dir supabase/migrations --apply  # apply (lint must pass)
```

MCP: `migrate.run {organization_id, project_id, environment_id, migrations:
[{version, name, up, down?}]}` plans and lints; add `apply: true` and
`confirm: true` to apply. Apply to a preview branch first when production
already has data (see `branching.md`).

For RLS proof before production, the MCP server's `policy.test` runs the
cross-user isolation matrix and then deliberately breaks each policy to check
the tests notice; `schema.apply` only accepts a schema that passed it.

## Pulling an existing schema

```bash
scribase db pull "$ORG" "$PROJECT" "$ENV" --dir supabase/migrations --dry-run
scribase db pull "$ORG" "$PROJECT" "$ENV" --dir supabase/migrations
```

## Types

```bash
scribase gen types "$ORG" "$PROJECT" "$ENV" --out src/database.types.ts
```

```ts
import { createClient } from '@supabase/supabase-js'
import type { Database } from './database.types'
export const supabase = createClient<Database>(url, anonKey)
```

Regenerate after every applied migration.

## Reading data while debugging

`database.tables` (tables, row estimates, RLS state) and `database.query`
(one read-only `SELECT`/`WITH`/`EXPLAIN`, bounded transaction). Writes and DDL
are rejected by design. `insights.get` / `scribase insights` flags tables
without RLS and unindexed foreign keys.
