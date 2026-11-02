# Storage: buckets, uploads, signed URLs

Storage works with supabase-js (`/storage/v1`). Access is decided by RLS on
`storage.objects`, so write the bucket and its policies as a migration.

```sql
-- 003_avatars.sql
insert into storage.buckets (id, name, public) values ('avatars', 'avatars', false)
  on conflict (id) do nothing;

create policy "avatars: owner reads" on storage.objects for select to authenticated
  using (bucket_id = 'avatars' and (storage.foldername(name))[1] = (select auth.uid())::text);
create policy "avatars: owner writes" on storage.objects for insert to authenticated
  with check (bucket_id = 'avatars' and (storage.foldername(name))[1] = (select auth.uid())::text);
```

Upload under the user's own folder:

```ts
const { data: { user } } = await supabase.auth.getUser()
const path = `${user!.id}/avatar.png`
await supabase.storage.from('avatars').upload(path, file, { upsert: true, contentType: 'image/png' })
const { data } = await supabase.storage.from('avatars').createSignedUrl(path, 60 * 60)
```

List buckets while debugging with the MCP `storage.buckets` tool.
