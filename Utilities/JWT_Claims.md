# Supabase JWTs + Custom Claims : Concepts & Hands-on Lab (RBAC + ABAC)

> This doc explains what JWTs are, how Supabase issues and validates them, how to add **custom claims** safely, how token TTL/validity works, and a step-by-step **lab** to implement and test hybrid **RBAC + ABAC** with Postgres RLS. Copy-paste friendly; secure-by-default.

---

## 1) JWTs in 5 minutes

**JWT (JSON Web Token)** is a compact, URL-safe token with three parts:

- **Header**: `{ alg, typ }` — e.g., `alg: "RS256"`.
- **Payload (claims)**: e.g., `sub` (user id), `exp` (expiry), `iat` (issued at), plus **custom claims**.
- **Signature**: proves the token was issued by Supabase and not altered.

Supabase Auth issues JWTs as access tokens. Postgres decodes them and enforces **Row-Level Security (RLS)** with helpers like:

- `auth.uid()` → current user ID
- `auth.jwt()` → raw claims JSON

> **Rule**: don’t trust client-set fields. Use DB state + RLS for enforcement. Custom claims are for hints/UI.

---

## 2) Claims

- **Standard**: `sub`, `iss`, `aud`, `exp`, `iat`.
- **Supabase**:
  - `app_metadata` → server/admin-managed (safe).
  - `user_metadata` → user-editable (not safe for auth).
- **Custom**: fields you add (e.g., `tenant`, `feature_flags`) via Auth Hook or admin API.

---

## 3) Validity, TTL, rotation

- **Access token**: short-lived (~1h by default). Controlled by `exp`.
- **Refresh token**: long-lived; used to rotate access tokens. Revoked on sign-out.
- **Checks**:
  - Inspect `exp`/`iat` via `auth.jwt()`.
  - Update expiry in Auth settings and re-login to test.
  - Tokens remain valid until `exp` even if roles change.

---

## 4) RBAC + ABAC

- **RBAC** = who (roles/permissions).  
- **ABAC** = what (row attributes).  

Combine both:

- Tables: `roles`, `permissions`, `user_roles`, plus resource tables with attributes (`org_id`, `sensitivity`).
- Policies: enforce role permissions + attribute conditions.

---

## 5) Secure patterns

✅ Do:
- Use `auth.uid()` + RLS joins.
- Wrap logic in helpers (`has_perm`).
- Keep `app_metadata` minimal/admin-only.

❌ Don’t:
- Trust `user_metadata`.
- Skip RLS by overloading JWT claims.
- Use `using (true)` in policies.

---

## 6) Hands-on Lab

### 6.1. Prereqs
- Supabase project
- SQL Editor
- (Optional) Edge Functions for admin ops

### 6.2. Schema

```sql
create extension if not exists pgcrypto;

create table if not exists public.organizations (
  id uuid primary key default gen_random_uuid(),
  name text not null
);

create table if not exists public.users (
  id uuid primary key,
  primary_org_id uuid references public.organizations(id),
  created_at timestamptz default now()
);

create table if not exists public.roles (
  id serial primary key,
  name text unique not null
);

create table if not exists public.permissions (
  id serial primary key,
  name text unique not null
);

create table if not exists public.role_permissions (
  role_id int references public.roles(id) on delete cascade,
  permission_id int references public.permissions(id) on delete cascade,
  primary key (role_id, permission_id)
);

create table if not exists public.user_roles (
  user_id uuid references public.users(id) on delete cascade,
  org_id uuid references public.organizations(id) on delete cascade,
  role_id int references public.roles(id) on delete cascade,
  primary key (user_id, org_id, role_id)
);

create table if not exists public.patients (
  id uuid primary key default gen_random_uuid(),
  org_id uuid not null references public.organizations(id),
  assigned_clinician uuid references public.users(id),
  sensitivity text check (sensitivity in ('low','normal','high')) default 'normal',
  full_name text not null,
  created_at timestamptz default now()
);

### 6.3. RLS + helpers

```sql
alter table public.users enable row level security;
alter table public.user_roles enable row level security;
alter table public.patients enable row level security;

-- Permission helper
create or replace function public.has_perm(_perm text, _org uuid)
returns boolean language sql stable as $$
  select exists (
    select 1
    from public.user_roles ur
    join public.role_permissions rp on rp.role_id = ur.role_id
    join public.permissions p on p.id = rp.permission_id
    where ur.user_id = auth.uid()
      and ur.org_id = _org
      and p.name = _perm
  );
$$;

-- Patients policies (RBAC)
create policy if not exists patients_read_by_permission
on public.patients for select to authenticated
using (public.has_perm('patients.read', org_id));

create policy if not exists patients_write_by_permission
on public.patients for insert to authenticated
with check (public.has_perm('patients.write', org_id));

create policy if not exists patients_update_by_permission
on public.patients for update to authenticated
using (public.has_perm('patients.write', org_id))
with check (public.has_perm('patients.write', org_id));

-- ABAC additions
create policy if not exists patients_read_if_assigned
on public.patients for select to authenticated
using (assigned_clinician = auth.uid());

create policy if not exists patients_read_high_sensitivity
on public.patients for select to authenticated
using (
  sensitivity = 'high'
  and public.has_perm('patients.read.high', org_id)
);


### 6.4. Seed minimal data

```sql
-- Org
insert into public.organizations(name) values ('Acme Health') returning id;
-- Copy the returned org UUID -> :ORG

-- Create permission catalog
insert into public.permissions(name) values
  ('patients.read'), ('patients.write'), ('patients.read.high')
  on conflict do nothing;

-- Create a role and map permissions
insert into public.roles(name) values ('clinician') on conflict do nothing;
insert into public.role_permissions(role_id, permission_id)
select r.id, p.id from public.roles r, public.permissions p
where r.name='clinician' and p.name in ('patients.read')
on conflict do nothing;
```

### 6.5. Create a user & mirror row in `public.users`

1) In **Auth → Users**, add a new user (email+password or magic link). After confirming, copy its **UUID**.
2) Insert the mirror row and assign role membership:

```sql
-- Mirror row
insert into public.users(id, primary_org_id) values ('<USER_UUID>', '<ORG_UUID>')
  on conflict (id) do nothing;

-- Role membership for org
insert into public.user_roles(user_id, org_id, role_id)
select '<USER_UUID>', '<ORG_UUID>', r.id from public.roles r where r.name='clinician'
  on conflict do nothing;
```

### 6.6. Insert some data rows

```sql
insert into public.patients(org_id, assigned_clinician, sensitivity, full_name)
values
  ('<ORG_UUID>', '<USER_UUID>', 'normal', 'Jane Patient'),
  ('<ORG_UUID>', null, 'high', 'VIP Patient');
```

### 6.7. Add **custom claims** safely

**Option A — Auth Hook (Custom Access Token Hook)**

```sql
create or replace function auth.custom_access_token_hook(event jsonb)
returns jsonb language plpgsql as $$
declare
  v_user_id uuid := (event->>'user_id')::uuid;
  v_tenant text;
  v_features jsonb := '[]'::jsonb;
begin
  select o.name into v_tenant
  from public.users u
  join public.organizations o on o.id = u.primary_org_id
  where u.id = v_user_id;

  if exists (
    select 1 from public.user_roles ur
    join public.roles r on r.id = ur.role_id and r.name = 'clinician'
    where ur.user_id = v_user_id
  ) then
    v_features := jsonb_insert(v_features, '{0}', '"clinician_ui"');
  end if;

  return jsonb_build_object(
    'app_metadata', jsonb_build_object(
      'tenant', coalesce(v_tenant, 'unknown'),
      'features', v_features
    )
  );
end; $$;
```

Then, in **Auth → Hooks**, register `auth.custom_access_token_hook` for **access token** issuance.

**Option B — Admin update (service role)**

```ts
import { createClient } from "https://esm.sh/@supabase/supabase-js@2";

export async function setAppMetadata(userId: string, metadata: Record<string, any>) {
  const supabase = createClient(DENO_ENV_SUPABASE_URL, DENO_ENV_SERVICE_ROLE_KEY);
  const { data, error } = await supabase.auth.admin.updateUserById(userId, {
    app_metadata: metadata,
  });
  if (error) throw error;
  return data;
}
```

### 6.8. Verify claims & TTL from SQL

```sql
select auth.uid() as user_id,
       auth.jwt() -> 'app_metadata' ->> 'tenant' as tenant_claim,
       auth.jwt() -> 'app_metadata' -> 'features'  as features,
       to_timestamp( (auth.jwt() ->> 'iat')::bigint ) as issued_at,
       to_timestamp( (auth.jwt() ->> 'exp')::bigint ) as expires_at;
```

### 6.9. End‑to‑end tests

- **Test 1 — RBAC read allowed**: clinician reads assigned patients.
- **Test 2 — ABAC override**: even without `patients.read`, clinician reads their assigned row.
- **Test 3 — High sensitivity gate**: only visible with `patients.read.high` permission.
- **Test 4 — Custom claim presence**: verify `tenant` and `features` appear in JWT.
- **Test 5 — TTL**: adjust Auth expiry, sign out/in, verify `exp`.

---

## 7) FAQ

- **Can I rely on custom claims alone for authorization?** No.
- **Where store org/tenant membership?** In normalized tables (`user_roles`).
- **Impersonate user for RLS testing?** Use API with that user’s session.
- **What expires and when?** Access tokens per `exp`; refresh tokens rotated/revoked.

---

## 8) Snippets: client‑side tests

```ts
import { createClient } from '@supabase/supabase-js'
const supabase = createClient(import.meta.env.VITE_SUPABASE_URL, import.meta.env.VITE_SUPABASE_ANON_KEY)

const { data: { user } } = await supabase.auth.getUser()
const { data: session } = await supabase.auth.getSession()
console.log('uid', user?.id)
console.log('jwt exp', session?.session?.expires_at)
console.log('claims', session?.session?.user?.app_metadata)

const { data: patients } = await supabase.from('patients').select('*')
```

---

## 9) Security checklist

- [ ] Default‑deny RLS on all user tables.
- [ ] Use helpers (`has_perm`) to avoid duplicated logic.
- [ ] Keep sensitive decisions in DB state; custom claims only mirror.
- [ ] Keep access tokens short‑lived.
- [ ] Audit admin operations that change roles/claims.

---

### Appendix: Quick teardown

```sql
drop policy if exists patients_read_high_sensitivity on public.patients;
drop policy if exists patients_read_if_assigned on public.patients;
drop policy if exists patients_update_by_permission on public.patients;
drop policy if exists patients_write_by_permission on public.patients;
drop policy if exists patients_read_by_permission on public.patients;

alter table public.patients disable row level security;
alter table public.user_roles disable row level security;
alter table public.users disable row level security;

drop function if exists public.has_perm(text, uuid);

drop table if exists public.patients;
drop table if exists public.user_roles;
drop table if exists public.role_permissions;
drop table if exists public.permissions;
drop table if exists public.roles;
drop table if exists public.users;
drop table if exists public.organizations;
```
```

