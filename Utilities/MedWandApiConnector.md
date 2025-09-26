
---

# MedWand Exam Results Callback : Architecture, Code, Tests, and Debugging

## 1) What triggers a callback?

**MedWand only calls your Callback URL after an exam is completed** in the MedWand client (device/app). Creating, updating, or canceling an appointment does **not** trigger a results callback (those are synchronous API calls where you get `patientUrl/doctorUrl/guestUrl` in the response). The MedWand public docs list appointment endpoints (`/default/appointment/create|update|delete`) and describe the VirtualCare API used by enterprise systems to schedule and then receive exam data.

---

## 2) Callback configuration (MedWand console)

Configure these in MedWand:

* **Callback URL**: HTTPS endpoint we control (our `results-receiver` Edge Function).
  Example: `https://<PROJECT-REF>.functions.supabase.co/results-receiver`

* **Callback Type** *(choose one)*:

  * **Notify**: tiny JSON (e.g., `{ appointmentId | examId, endDate }`). We then **pull** the full results with `POST /default/v2/exam/results`. (Best for reliability with large payloads.)
  * **Full**: MedWand **pushes full JSON** exam to your URL.
  * **Html**: MedWand pushes a JSON wrapper that contains an `htmlDocument` (or raw HTML).

* **Callback Authentication Type** *(choose one)*:

  * **Bearer** → sends `Authorization: Bearer <token>`
  * **ApiToken** → sends `ApiToken: <token>`
  * **KeySecret** → sends `key: <key>`, `secret: <secret>`
  * **Custom** → your own header name & value
  * **None** → not recommended

> The MedWand API indicates appointment endpoints and header usage (`ApiToken`), and their integration model where your enterprise system stores results. We mirror those patterns for callbacks.

---

## 3) Supabase pieces we rely on

* **Database Webhook** on `public.appointments` → hits our `medwand-sync` function to call MedWand for **create/update/delete**; **doesn’t** involve results. Webhooks fire after commit and can include old/new rows.
* **Edge Functions** (`medwand-sync`, `results-receiver`) with **JWT verify disabled** and shared-secret header (for `medwand-sync`) or MedWand-sent auth header (for `results-receiver`).

---

## 4) Data model (minimal - will add more details)

```sql
-- appointments: created/updated/deleted by our system; synced upstream
-- (unchanged if you already applied earlier migrations)
create table if not exists public.appointments (
  id uuid primary key default gen_random_uuid(),
  appointment_id text unique, -- we set to id::text as stable key for upstream
  org_id uuid not null,
  patient_id uuid not null,
  clinician_id uuid not null,
  doctor_name text not null,
  patient_name text not null,
  exam_type text not null check (exam_type in ('Patient','OnSite','Remote')),
  scheduled_start_utc timestamptz not null,
  scheduled_duration_min int not null check (scheduled_duration_min > 0),
  status text not null check (status in ('requested','scheduled','rescheduled','canceled','completed')) default 'scheduled',
  reason text,
  metadata jsonb not null default '{}',
  patient_url text,
  doctor_url text,
  guest_url text,
  sync_state text not null default 'pending', -- pending|in_sync|error
  sync_error text,
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now()
);

-- results: parsed summary + pointer to raw payload we stream to Storage
create table if not exists public.appointment_results (
  id uuid primary key default gen_random_uuid(),
  appointment_id uuid not null references public.appointments(id) on delete cascade,
  org_id uuid not null,
  patient_id uuid not null,
  clinician_id uuid not null,
  summary jsonb not null default '{}',
  raw_blob_path text not null,
  created_at timestamptz not null default now()
);
```

---

## 5) Edge Function — `results-receiver` (handles Notify, Full, Html)

**Responsibilities**

1. **Authenticate** the callback header per your chosen Callback Auth Type.
2. Detect payload form:

   * **Notify**: has `appointmentId|examId`, lacks `readings/htmlDocument` → **pull** full JSON with `POST /default/v2/exam/results`.
   * **Full**: JSON includes exam fields (`readings`, `patientName`, etc.) → store as-is.
   * **Html**: JSON field `htmlDocument` or raw HTML payload → store HTML.
3. **Stream-first**: always upload raw to a **private Storage bucket** (e.g., `appointment-results/…`) before parsing/DB writes (robust for large payloads?).
4. Upsert a **summary** row in `appointment_results`, linked to the matching `appointments` row by `appointment_id`.

> Supabase docs: Edge Functions config, Webhooks behavior.

** code (TypeScript/Deno)**

```ts
import { serve } from "https://deno.land/std@0.224.0/http/server.ts";

// --- env ---
const SUPABASE_URL = must("SUPABASE_URL", Deno.env.get("SUPABASE_URL"));
const SERVICE_KEY  = must("SUPABASE_SERVICE_ROLE_KEY", Deno.env.get("SUPABASE_SERVICE_ROLE_KEY"));
const MEDWAND_BASE = "https://api.medwand.cloud/default/v2";
const MEDWAND_API_TOKEN = must("MEDWAND_API_TOKEN", Deno.env.get("MEDWAND_API_TOKEN")); // base64("key:secret")
const BUCKET = Deno.env.get("RESULTS_BUCKET") ?? "appointment-results";

// Match your MedWand Callback Authentication Type
const CB_AUTH_TYPE = (Deno.env.get("MW_CB_AUTH_TYPE") || "none").toLowerCase();
const CB_BEARER    = Deno.env.get("MW_CB_BEARER");
const CB_API_TOKEN = Deno.env.get("MW_CB_API_TOKEN");
const CB_KEY       = Deno.env.get("MW_CB_KEY");
const CB_SECRET    = Deno.env.get("MW_CB_SECRET");
const CB_CUSTOM_H  = Deno.env.get("MW_CB_CUSTOM_HEADER");
const CB_CUSTOM_V  = Deno.env.get("MW_CB_CUSTOM_VALUE");

function must(name: string, v?: string | null) {
  if (!v) { console.error(`[results-receiver] missing env ${name}`); throw new Error(`missing env ${name}`); }
  return v;
}

function validateCallbackAuth(req: Request) {
  switch (CB_AUTH_TYPE) {
    case "none":     return true;
    case "bearer":   return req.headers.get("authorization") === `Bearer ${CB_BEARER}`;
    case "apitoken": return req.headers.get("ApiToken") === CB_API_TOKEN;
    case "keysecret":return req.headers.get("key") === CB_KEY && req.headers.get("secret") === CB_SECRET;
    case "custom":   return req.headers.get(CB_CUSTOM_H!) === CB_CUSTOM_V;
    default:         return false;
  }
}

async function pullFullResults(appointmentId: string, includeHtml = false) {
  const r = await fetch(`${MEDWAND_BASE}/exam/results`, {
    method: "POST",
    headers: { "content-type": "application/json", "ApiToken": MEDWAND_API_TOKEN },
    body: JSON.stringify({ appointmentId, includeHtml })
  });
  const text = await r.text();
  return { ok: r.ok, text };
}

async function storageUpload(path: string, body: string, contentType = "application/json") {
  const res = await fetch(`${SUPABASE_URL}/storage/v1/object/${BUCKET}/${path}`, {
    method: "POST",
    headers: { authorization: `Bearer ${SERVICE_KEY}`, "content-type": contentType },
    body
  });
  if (!res.ok) throw new Error(`storage upload failed: ${res.status}`);
}

async function upsertSummary(appointmentId: string, summary: Record<string, unknown>, blobPath: string) {
  // Find the appointments row by its upstream appointment_id
  const apptRes = await fetch(
    `${SUPABASE_URL}/rest/v1/appointments?appointment_id=eq.${encodeURIComponent(appointmentId)}&select=id,org_id,patient_id,clinician_id`,
    { headers: { apikey: SERVICE_KEY, authorization: `Bearer ${SERVICE_KEY}` } }
  );
  const [appt] = await apptRes.json();

  await fetch(`${SUPABASE_URL}/rest/v1/appointment_results`, {
    method: "POST",
    headers: { apikey: SERVICE_KEY, authorization: `Bearer ${SERVICE_KEY}`, "content-type": "application/json" },
    body: JSON.stringify({
      appointment_id: appt?.id,
      org_id: appt?.org_id,
      patient_id: appt?.patient_id,
      clinician_id: appt?.clinician_id,
      summary,
      raw_blob_path: blobPath
    })
  });
}

serve(async (req) => {
  try {
    if (!validateCallbackAuth(req)) return new Response("unauthorized", { status: 401 });

    const bodyText = await req.text();
    let json: any | null = null; try { json = JSON.parse(bodyText); } catch { /* maybe raw HTML */ }

    const notifiedId = json?.appointmentId ?? json?.examId;

    // A) Notify → pull full JSON
    if (json && notifiedId && !json.htmlDocument && !json.readings) {
      const pulled = await pullFullResults(notifiedId, false);
      if (!pulled.ok) return new Response("pull failed", { status: 502 });
      const path = `${notifiedId}/${Date.now()}.json`;
      await storageUpload(path, pulled.text, "application/json");
      let summary: Record<string, unknown> = {};
      try { summary = JSON.parse(pulled.text); } catch {}
      await upsertSummary(notifiedId, summary, path);
      return new Response("ok");
    }

    // B) Full JSON pushed
    if (json && (json.readings || json.exam || json.patientName || json.doctorName)) {
      const id = notifiedId ?? json?.appointmentId ?? "unknown";
      const path = `${id}/${Date.now()}.json`;
      await storageUpload(path, bodyText, "application/json");
      await upsertSummary(id, json, path);
      return new Response("ok");
    }

    // C) Html pushed (JSON wrapper)
    if (json?.htmlDocument) {
      const id = notifiedId ?? json?.appointmentId ?? "unknown";
      const path = `${id}/${Date.now()}.html.json`;
      await storageUpload(path, bodyText, "application/json");
      await upsertSummary(id, { htmlPushed: true, size: bodyText.length }, path);
      return new Response("ok");
    }

    // D) Raw HTML body
    if (!json) {
      const id = notifiedId ?? "unknown";
      const path = `${id}/${Date.now()}.html`;
      await storageUpload(path, bodyText, "text/html");
      await upsertSummary(id, { htmlPushed: true, size: bodyText.length }, path);
      return new Response("ok");
    }

    return new Response("unrecognized payload", { status: 400 });
  } catch (e) {
    console.error("[results-receiver] error", e);
    return new Response("error", { status: 500 });
  }
});
```

---

## 6) End-to-end timing: when does what happen?

1. **Our app schedules** an appointment → we insert a row → Supabase **webhook** posts to `medwand-sync` → we call MedWand **create/update/delete** (`ApiToken` header). Appointment APIs live under `/default/appointment/...` (v2 path is `/default/v2/...` for results).
2. **Exam is performed** on MedWand client and **completed** → **now** MedWand posts to our **Callback URL** (type Notify/Full/Html, with chosen auth). That’s the only time the results callback occurs. 
3. `results-receiver` authenticates the callback, **streams** payload to Storage, **upserts** summary row.

---

## 7) How to test

### A) Create appointment (triggers medwand-sync → MedWand create)

```sql
insert into public.appointments (
  org_id, patient_id, clinician_id,
  doctor_name, patient_name, exam_type,
  scheduled_start_utc, scheduled_duration_min,
  status, reason, metadata
) values (
  '11111111-1111-1111-1111-111111111111',
  'aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa',
  'bbbbbbbb-bbbb-bbbb-bbbb-bbbbbbbbbbbb',
  'Dr. Smith','Jane Doe','Patient',
  now() + interval '60 minutes', 30,
  'scheduled','smoke','{}'::jsonb
)
returning id;
```

Check:

```sql
select id, appointment_id, sync_state, patient_url, doctor_url, guest_url, sync_error
from public.appointments
order by updated_at desc limit 1;
```

* Expect `sync_state='in_sync'` and URLs present.

> Webhooks fire **after** row changes, and you can include row data in the payload—which our function expects.

### B) Simulate results callback (Notify → we pull full results)

```bash
curl -X POST 'https://<PROJECT-REF>.functions.supabase.co/results-receiver' \
  -H 'authorization: Bearer <MW_CB_BEARER>' \
  -H 'content-type: application/json' \
  -d '{ "appointmentId": "<appointment_id or row id string>", "endDate":"2025-10-01T11:00:00Z" }'
```

Then:

```sql
select appointment_id, raw_blob_path, left(cast(summary as text), 200) as preview
from public.appointment_results
order by created_at desc limit 1;
```

### C) Simulate Full JSON pushed

```bash
curl -X POST 'https://<PROJECT-REF>.functions.supabase.co/results-receiver' \
  -H 'authorization: Bearer <MW_CB_BEARER>' \
  -H 'content-type: application/json' \
  -d '{ "appointmentId":"<id>", "patientName":"Jane Doe", "doctorName":"Dr. Smith", "readings":[{"type":"SpO2","value":98}] }'
```

### D) Simulate Html pushed

```bash
curl -X POST 'https://<PROJECT-REF>.functions.supabase.co/results-receiver' \
  -H 'authorization: Bearer <MW_CB_BEARER>' \
  -H 'content-type: application/json' \
  -d '{ "appointmentId":"<id>", "htmlDocument":"<html>...</html>" }'
```

---

## 8) Debugging checklist

**If appointment row stays `pending`**

* Webhook didn’t reach `medwand-sync` or function exited early.
  *Check:* Supabase **Database → Webhooks → Deliveries** (2xx?), payload has **OLD_AND_NEW_ROWS**? Headers include your secret? Webhooks always run post-commit.

**If `medwand-sync` shows 401**

* Function is verifying JWT (disable) **or** `x-webhook-secret` mismatch.
  *Doc:* “Skipping authorization checks” for Functions. 

**If `medwand-sync` shows 500**

* Missing envs (e.g., `MEDWAND_API_TOKEN`, `SUPABASE_URL`) or payload missing `record/old_record`. (Enable **OLD_AND_NEW_ROWS**; set envs.)

**If `sync_state='error'`**

* MedWand returned non-2xx; we log body in `sync_error`.
  *Check fields:* `doctorName/patientName`, `scheduledStartDateUtc` format, `scheduledDuration`, and `ApiToken` validity. Appointment create/update/delete endpoints live under `/default/appointment/...`.

**If results aren’t saved**

* `results-receiver` returned 401 (auth header mismatch), 502 (pull failed), or 500 (Storage).
  *Fix:* Ensure callback auth envs match MedWand console settings; Storage bucket exists and is **private**; function has `SERVICE_ROLE_KEY`.

**Logs you should watch**

* Supabase **Functions → medwand-sync / results-receiver → Logs**
* Supabase **Database → Webhooks → Deliveries** (status & payload)

---

## 9) Operational & security notes

* **Private Storage** only; generate signed URLs for viewing.
* **Secret rotation**: rotate `MEDWAND_API_TOKEN` and callback tokens periodically.
* **Idempotency**: use stable `appointmentId` (we set to our `id::text` upstream) so retries don’t create dupes.
* **Rate limiting**: --.

---

## 10) Handy queries & scripts

**Recent sync failures**

```sql
select date_trunc('hour', created_at) as hour, count(*) as failures
from public.appointment_sync_events
where success = false
group by 1 order by 1 desc limit 24;
```

**Re-queue a failed appointment sync**

```sql
update public.appointments
set updated_at = now()
where sync_state = 'error';
```

**Last 10 results blobs**

```sql
select appointment_id, raw_blob_path, created_at
from public.appointment_results
order by created_at desc
limit 10;
```



