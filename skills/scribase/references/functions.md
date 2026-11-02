# Edge functions

Deno functions in the `supabase/functions/<name>/index.ts` layout. Every deploy is a new immutable version and
only becomes active if it boots, so a broken deploy never replaces a working
one.

```ts
// supabase/functions/hello/index.ts
Deno.serve(async (req) => {
  const { name } = await req.json()
  return new Response(JSON.stringify({ message: `Hello ${name}` }), {
    headers: { 'Content-Type': 'application/json' },
  })
})
```

```bash
scribase functions deploy "$ORG" "$PROJECT" "$ENV" hello supabase/functions/hello
# or every function in a Supabase functions directory (_shared, deno.json, config.toml)
scribase functions deploy "$ORG" "$PROJECT" "$ENV" --functions-dir supabase/functions

scribase functions invoke "$ORG" "$PROJECT" "$ENV" hello --data '{"name":"Ada"}'
scribase functions logs   "$ORG" "$PROJECT" "$ENV" hello --limit 50
scribase functions rollback "$ORG" "$PROJECT" "$ENV" hello <version>
```

Secrets (bound into the function at deploy time, kept current afterwards):

```bash
scribase secrets set "$ORG" "$PROJECT" "$ENV" STRIPE_KEY --from-env STRIPE_KEY   # value never on the command line
scribase secrets list "$ORG" "$PROJECT" "$ENV"                                   # names only
```

From the app: `await supabase.functions.invoke('hello', { body: { name: 'Ada' } })`.
MCP: `functions.list` and `logs.tail` for inspection.
