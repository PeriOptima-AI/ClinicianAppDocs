# Supabase JWT **Custom Claims** & **Auth Hooks** — (w/ FlutterFlow)

This doc shows **what you can add to a Supabase JWT**, how to build and register a **Custom Access Token Hook**, how to **read claims in FlutterFlow**, and how to keep security **in Postgres RLS**.

---

## 0) JWTs 

**JWT (JSON Web Token)** is a compact, URL-safe token with three parts:

- **Header**: `{ alg, typ }` — e.g., `alg: "RS256"`.
- **Payload (claims)**: e.g., `sub` (user id), `exp` (expiry), `iat` (issued at), plus **custom claims**.
- **Signature**: proves the token was issued by Supabase and not altered.

Supabase Auth issues JWTs as access tokens. Postgres decodes them and enforces **Row-Level Security (RLS)** with helpers like:

- `auth.uid()` → current user ID
- `auth.jwt()` → raw claims JSON

> **Rule**: don’t trust client-set fields. Use DB state + RLS for enforcement. Custom claims are for hints/UI.

---

* Add **small, server-derived** facts to `claims.app_metadata` (e.g., `org_id`, `roles`, `features`) for **UI hints**.
* **Never** authorize with JWT contents. Keep **RBAC/ABAC** in **RLS** tables/policies.
* Hook lives in **`public`** schema, returns **`{"claims": {...}}`**, and is called by Supabase Auth at token issue time.
* After changing hook/context, **refresh the session** to mint a new token.

---

## 1) JWT Claims: what’s there by default

**Required claims** (examples): `iss`, `aud`, `exp`, `iat`, `sub`, `role`, `aal`, `session_id`, `email`, `phone`, `is_anonymous`
**Optional claims**: `jti`, `nbf`, `amr`, `app_metadata`, `user_metadata`

Your hook **must** return the `claims` object (you typically merge into `claims.app_metadata`). `user_metadata` is user-editable → do **not** use it for auth.

---

## 2) What to put in **app_metadata** (safe & useful)

These are common, **UI-only** fields for multi-tenant/RBAC apps:

### Tenant & context

* `org_id`, `org_slug`, `org_name` (short label)
* `department_id`, `department_slug`
* `tenant` (display label)
* `env`: `"prod" | "staging"`


### Capability hints (UI toggles)

* `roles`: `["nurse","doctor"]` (names only)
* `perms`: `["patients.read","patients.write"]` (keep short)
* `features`: `["clinician_ui","beta_notes"]`
* `mfa_enforced`: `true/false`
* `locale`: `"en-US"`, `theme`: `"light" | "dark"`

### Session/ops signals

* `auth_method`: e.g. `"password"`, `"sso/saml"`, `"magiclink"`
* `session_tags`: like `["mobile","kiosk"]`
* `limits`: e.g. `{ "max_upload_mb": 25 }`

> **Avoid** PHI/PII, secrets, or big blobs. Aim to keep the whole JWT **< ~2 KB**.

---

## 3) Build the Custom Access Token Hook

Create it in **`public`**, **SECURITY DEFINER**, and **return `{"claims": ...}`**.

```sql
-- 3.1 Hook: add compact, UI-only facts to claims.app_metadata
create or replace function public.custom_access_token_hook(event jsonb)
returns jsonb
language plpgsql
security definer
set search_path = public
as $$
declare
  v_user   uuid := (event->>'user_id')::uuid;
  v_org    uuid; 
  v_dept   uuid; 
  v_org_slug text; 
  v_tenant text;
  v_roles  jsonb := '[]'::jsonb;
  v_perms  jsonb := '[]'::jsonb;
  v_features jsonb := '[]'::jsonb;
  claims   jsonb := coalesce(event->'claims','{}'::jsonb);
  auth_method text := replace(coalesce(event->>'authentication_method',''), '"', '');
begin
  -- Active context from your mirror row (public.users)
  select u.org_id, u.department_id
    into v_org, v_dept
  from public.users u
  where u.id = v_user;

  -- Optional labels
  if v_org is not null then
    select o.slug, o.name into v_org_slug, v_tenant
    from public.organizations o
    where o.id = v_org;
  end if;

  -- Role names for the active org
  select coalesce(jsonb_agg(r.name order by r.name),'[]'::jsonb)
    into v_roles
  from public.user_roles ur
  join public.roles r on r.id = ur.role_id
  where ur.user_id = v_user
    and (v_org is null or ur.org_id = v_org);

  -- Distinct permissions for the active org
  select coalesce(jsonb_agg(distinct p.name order by p.name),'[]'::jsonb)
    into v_perms
  from public.user_roles ur
  join public.role_permissions rp on rp.role_id = ur.role_id
  join public.permissions p on p.id = rp.permission_id
  where ur.user_id = v_user
    and (v_org is null or ur.org_id = v_org);

  -- Example feature flag
  if v_roles ? 'doctor' or v_roles ? 'clinician' then
    v_features := v_features || to_jsonb('clinician_ui');
  end if;

  -- Merge into claims.app_metadata (keep it small)
  claims := jsonb_set(
    claims, '{app_metadata}',
    coalesce(claims->'app_metadata','{}'::jsonb) ||
    jsonb_strip_nulls(jsonb_build_object(
      'tenant',        v_tenant,
      'org_id',        v_org,
      'org_slug',      v_org_slug,
      'department_id', v_dept,
      'roles',         v_roles,
      'perms',         v_perms,
      'features',      v_features,
      'auth_method',   nullif(auth_method,''),
      'env',           'prod',
      'locale',        'en-US',
      'theme',         'light',
      'mfa_enforced',  true
    )),
    true
  );

  -- Ensure a jti exists (optional)
  if not (claims ? 'jti') then
    claims := jsonb_set(claims, '{jti}', to_jsonb(gen_random_uuid()::text), true);
  end if;

  return jsonb_build_object('claims', claims);

exception when others then
  -- Failsafe
  return jsonb_build_object('claims', coalesce(event->'claims','{}'::jsonb));
end
$$;

-- 3.2 Grants: only the Auth service should call the hook
grant usage on schema public to supabase_auth_admin;
grant execute on function public.custom_access_token_hook(jsonb) to supabase_auth_admin;
revoke execute on function public.custom_access_token_hook(jsonb) from authenticated, anon, public;
```

**Register the hook:** Studio → **Auth → Hooks → Access token** → `public.custom_access_token_hook`.

---

## 4) Optional: Change “active context” at runtime

When a user switches org/department, update your mirror row and **refresh**:

```sql
create or replace function public.set_active_context(p_org uuid, p_department uuid default null)
returns void
language plpgsql
security definer
set search_path = public
as $$
begin
  -- Guard: caller must be a member of this org[/dept]
  if not exists (
    select 1 from public.user_roles ur
    where ur.user_id = auth.uid()
      and ur.org_id  = p_org
      and (p_department is null
        or ur.department_id = p_department
        or ur.department_id is null)
  ) then
    raise exception 'Not a member of this org/department.';
  end if;

  update public.users
  set org_id = p_org, department_id = p_department
  where id = auth.uid();
end;
$$;

grant execute on function public.set_active_context(uuid, uuid) to authenticated;
```

**Client flow:** call `rpc('set_active_context', …)` → `await auth.refreshSession()` → read `user.appMetadata` again.

---

## 5) Read claims in Flutter/FlutterFlow

```dart
import 'package:supabase_flutter/supabase_flutter.dart';

final user = Supabase.instance.client.auth.currentUser;
final meta = user?.appMetadata;

final orgId  = meta?['org_id'] as String?;
final deptId = meta?['department_id'] as String?;
final roles  = (meta?['roles'] as List?)?.cast<String>() ?? const [];
final perms  = (meta?['perms'] as List?)?.cast<String>() ?? const [];
final tenant = meta?['tenant'] as String?;
final features = (meta?['features'] as List?)?.cast<String>() ?? const [];
final authMethod = meta?['auth_method'] as String?;
```

> After enabling/editing the hook, **sign out/in** or `await Supabase.instance.client.auth.refreshSession()` to mint a new token.

---

## 6) Server-side verification helper (what RLS “sees”)

```sql
create or replace function public.debug_my_claims()
returns table(uid uuid, app_metadata jsonb, user_metadata jsonb, exp timestamptz)
language sql stable as $$
  select auth.uid(),
         auth.jwt()->'app_metadata',
         auth.jwt()->'user_metadata',
         to_timestamp((auth.jwt()->>'exp')::bigint)
$$;

grant execute on function public.debug_my_claims() to authenticated;
```

From the app:

```dart
final out = await Supabase.instance.client.rpc('debug_my_claims');
```

**In Studio** (to simulate a user):

```sql
select set_config(
  'request.jwt.claims',
  '{"role":"authenticated","sub":"<USER_UUID>"}',
  true
);
select * from public.debug_my_claims();
```

---

## 7) Extra patterns (optional)

### (a) Slim down the JWT (size budget)

Keep only essential claims (respect required ones):

```sql
create or replace function public.custom_access_token_hook(event jsonb)
returns jsonb language plpgsql as $$
declare original jsonb := event->'claims'; new jsonb := '{}'::jsonb;
        k text;
begin
  foreach k in array array['iss','aud','exp','iat','sub','role','aal','session_id','email','phone','is_anonymous'] loop
    if original ? k then new := jsonb_set(new, array[k], original->k); end if;
  end loop;
  return jsonb_build_object('claims', new);
end $$;
```

### (b) Add an `admin` flag from a table

```sql
create table if not exists public.profiles (
  user_id uuid primary key references auth.users(id),
  is_admin boolean not null default false
);

create or replace function public.custom_access_token_hook(event jsonb)
returns jsonb language plpgsql as $$
declare claims jsonb := coalesce(event->'claims','{}'::jsonb);
        is_admin boolean := false;
begin
  select p.is_admin into is_admin from public.profiles p
  where p.user_id = (event->>'user_id')::uuid;

  if is_admin then
    claims := jsonb_set(
      claims, '{app_metadata,admin}', 'true'::jsonb, true
    );
  end if;

  return jsonb_build_object('claims', claims);
end $$;

grant select on public.profiles to supabase_auth_admin;
grant execute on function public.custom_access_token_hook(jsonb) to supabase_auth_admin;
revoke execute on function public.custom_access_token_hook(jsonb) from authenticated, anon, public;
```



---

**Simulate the hook directly (great for debugging):**

```sql
select public.custom_access_token_hook(
  jsonb_build_object(
    'user_id', '<USER_UUID>',
    'claims', jsonb_build_object('role','authenticated'),
    'authentication_method', 'password'
  )
);
```

---

## 9) Security model (why RLS)

* JWT custom claims are **convenience** for the **UI**.
* Real security comes from **RLS policies** that check **live DB state** (e.g., `user_roles`, `assignments`, row attributes).
* Changes to roles/assignments take effect **immediately** (next query), regardless of what an old token says.

---
