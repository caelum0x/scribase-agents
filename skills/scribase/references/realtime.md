# Realtime: broadcast, presence, row changes

Realtime works with supabase-js (`/realtime/v1`).

## Broadcast (no database involved)

```ts
const room = supabase.channel('room:42')
room.on('broadcast', { event: 'typing' }, ({ payload }) => console.log(payload.userId, 'is typing'))
await room.subscribe()
await room.send({ type: 'broadcast', event: 'typing', payload: { userId: 'alice' } })
```

## Presence

```ts
room.on('presence', { event: 'sync' }, () => console.log(room.presenceState()))
await room.track({ userId: 'alice', online_at: new Date().toISOString() })
```

## Row changes

```ts
supabase
  .channel('todos')
  .on('postgres_changes', { event: '*', schema: 'public', table: 'todos' }, (change) => console.log(change))
  .subscribe()
```

Change events respect RLS: a subscriber only receives rows its `select`
policy allows. Remember to `removeChannel` on unmount.
