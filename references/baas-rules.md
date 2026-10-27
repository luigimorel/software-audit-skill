# Supabase, Firebase, and other backend-as-a-service rules

Read this whenever the client talks directly to a database or storage service. In that setup, the rules are the only security boundary.

## Contents

- Backend-as-a-service rules (Supabase, Firebase, etc.)
- Supabase in depth
- Firebase in depth
- Convex
- Sources

---

## Backend-as-a-service rules (Supabase, Firebase, etc.)

If the app talks to the database directly from the client, **the rules are the only security.**

**Supabase:**
- Tables in exposed schemas with **RLS disabled** (check `ALTER TABLE ... ENABLE ROW LEVEL SECURITY` in migrations or the dashboard)
- Policies like `USING (true)` or `auth.role() = 'authenticated'` with no ownership condition
- `service_role` key in client code or used in client-callable edge functions
- Views and `SECURITY DEFINER` functions bypassing RLS
- Storage buckets set to public that hold private files; missing storage policies
- Missing `WITH CHECK` on insert/update policies (users can write rows as someone else)
- Policies like `USING (auth.uid() IS NOT NULL)`: same problem as `USING (true)` for any logged-in user

**Firebase:**
- Firestore/Realtime DB/Storage rules with `allow read, write: if true` or `if request.auth != null` and nothing else
- Test-mode rules left on after launch (expiry dates in rules)
- Cloud Functions without auth checks; App Check not enabled

**Fix pattern:** default-deny, then add the narrowest policy per table/collection, scoped to `auth.uid()` or `request.auth.uid`.

```sql
alter table public.notes enable row level security;

create policy "owner can read" on public.notes
  for select using (auth.uid() = user_id);

create policy "owner can write" on public.notes
  for all using (auth.uid() = user_id) with check (auth.uid() = user_id);
```

## Supabase in depth

Tables created through the SQL editor or migrations have RLS **off** by default, and the anon key is public, so a table without RLS is readable and writable by anyone. Check each of these:

- **Catch-all RLS enablement.** A migration that loops over every table in `public` and enables RLS stops forgotten tables from slipping through:

  ```sql
  DO $$ DECLARE r RECORD;
  BEGIN
    FOR r IN SELECT tablename FROM pg_tables WHERE schemaname = 'public'
    LOOP
      EXECUTE format('ALTER TABLE public.%I ENABLE ROW LEVEL SECURITY;', r.tablename);
    END LOOP;
  END $$;
  ```

- **Policy shape.** Scope to the owner and wrap the helper as `(select auth.uid())` so Postgres evaluates it once per query rather than per row.

  ```sql
  -- Bad: any logged-in user reads every row
  create policy "read docs" on public.documents for select to authenticated using (true);

  -- Good
  create policy "read own docs" on public.documents
    for select to authenticated using ((select auth.uid()) = user_id);
  ```

- **`WITH CHECK` on updates.** Without it a user can change `user_id` to someone else's and hand over (or steal) the row.

  ```sql
  create policy "update own tasks" on public.tasks
    for update to authenticated
    using ((select auth.uid()) = user_id)
    with check ((select auth.uid()) = user_id);
  ```

- **Privileged columns on self-editable tables.** If users can update their own `profiles` row, they can set `is_admin`, `credits`, or `subscription_tier`. Either move those columns to a table in a non-exposed `private` schema, or restrict column privileges:

  ```sql
  revoke update on profiles from authenticated;
  grant update (display_name, avatar_url) on profiles to authenticated;
  ```

- **Forgotten tables.** Junction, audit-log, and metadata tables often lack policies even when the main table has them. Every table exposed over the REST API needs its own. (RLS on with zero policies denies everything: safe, but can cause confusing bugs.)
- **`SECURITY DEFINER` functions** bypass RLS. In the `public` schema they are callable over the REST API by anyone. Keep them in a `private` schema, set `search_path = ''`, and validate inputs inside.
- **Storage buckets** need their own policies, otherwise any authenticated user can upload, read, or delete any file. Scope paths to the user's id:

  ```sql
  create policy "upload to own folder" on storage.objects
    for insert to authenticated
    with check (
      bucket_id = 'avatars'
      and (storage.foldername(name))[1] = (select auth.uid())::text
    );
  ```

## Firebase in depth

- **Never ship** `allow read, write: if true;` or `if request.auth != null;` alone. Ownership check example: `allow read, write: if request.auth.uid == userId;`
- **Field-level protection on updates.** Without it users can set `isAdmin: true` or `credits: 99999`:

  ```
  allow update: if request.auth.uid == userId
    && request.resource.data.diff(resource.data).affectedKeys().hasOnly(['displayName', 'avatarUrl']);
  ```

- **Subcollection trap.** Subcollections are **not** covered by the parent document's rules. Each needs explicit rules; AI assistants often miss this.
- **Validate on write.** Check types and sizes (`request.resource.data.displayName is string && ...size() <= 50`) and force server time (`request.resource.data.createdAt == request.time`).
- **Roles via custom claims** (`request.auth.token.role`), not by reading a user document the user may be able to edit. Claims also save a read.
- **Cloud Storage rules** must check `contentType`, `size`, and path ownership. Otherwise users can upload executables or write into other users' paths.

## Convex

- Every public `query` and `mutation` calls `ctx.auth.getUserIdentity()` and handles the unauthenticated case.
- Authentication is not authorization: mutations must also verify the caller owns the specific document.
- Functions only meant for internal use must be `internalQuery` / `internalMutation` / `internalAction`. Anything declared as `query` or `mutation` is callable by anyone.

## Sources

- [Supabase: Row Level Security](https://supabase.com/docs/guides/database/postgres/row-level-security)
- [Firebase: Security Rules](https://firebase.google.com/docs/rules)
- [Convex: Authentication](https://docs.convex.dev/auth)
