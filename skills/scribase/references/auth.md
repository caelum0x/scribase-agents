# Auth: Apple, Google, email

The auth service is GoTrue-compatible: use `supabase.auth.*` from
`@supabase/supabase-js`.

## Native Apple / Google sign-in (`signInWithIdToken`)

Mobile apps get an ID token from the platform SDK and exchange it; no browser
redirect, no client secret on the device.

1. Save the provider's client id on the environment and enable it (needs the
   `admin` role or an `admin`-scoped key). For native Apple/Google sign-in only
   the client id is required (the iOS bundle id for Apple; the web/iOS client
   id(s) for Google):

   ```bash
   curl -X PATCH "$SCRIBASE_API_URL/v1/organizations/$ORG/projects/$PROJECT/environments/$ENV/auth/config" \
     -H "Authorization: Bearer $SCRIBASE_ACCESS_TOKEN" \
     -H 'Content-Type: application/json' \
     -H "Idempotency-Key: auth-apple-$PROJECT" \
     -d '{"external":{"apple":{"client_id":"com.acme.app","enabled":true}}}'
   ```

   `PUT .../auth/providers/{provider}` with `{"enabled": true|false}` toggles a
   provider whose credentials are already saved. The response shows whether
   the new configuration is live.

2. In the app:

   ```ts
   // Apple (expo-apple-authentication)
   const credential = await AppleAuthentication.signInAsync({
     requestedScopes: [AppleAuthentication.AppleAuthenticationScope.EMAIL],
     nonce: hashedNonce,             // sha256 of rawNonce
   })
   await supabase.auth.signInWithIdToken({ provider: 'apple', token: credential.identityToken!, nonce: rawNonce })

   // Google (@react-native-google-signin/google-signin)
   const { data } = await GoogleSignin.signIn()
   await supabase.auth.signInWithIdToken({ provider: 'google', token: data!.idToken! })
   ```

Web OAuth redirects (`signInWithOAuth`) additionally need the provider secret
in the same `external.<provider>` object (`"secret": "..."`, stored sealed and
never returned) and your site URL in the redirect allow list.

## Email and password

```ts
await supabase.auth.signUp({ email, password })
await supabase.auth.signInWithPassword({ email, password })
```

## Using the session in RLS

Policies read the caller with `auth.uid()` and `auth.jwt()`; see
`database.md`. Inspect users with the MCP `auth.users` tool (no secrets are
returned).
