# 📧 BetterGrowth Mail System — Complete Setup Guide

A simple email-sending web app with:
- **Login** (Supabase Auth)
- **Compose & send** (Resend API) with CC/BCC, attachments
- **Auto image compression** (browser-side, ~85% smaller)
- **Sent history** (stored in Supabase)
- **Full detail view** (with image/PDF preview + download)
- **Resend + Delete** actions
- **Cloudflare Workers** for hosting

---

## 🧱 Architecture

```
Browser (Cloudflare Worker)
   │
   │  fetch(functions/v1/send-email?action=...)
   ▼
Supabase Edge Function (Deno)
   ├── Auth check (JWT)
   ├── Sends email via Resend API
   ├── Uploads attachment to Supabase Storage
   └── Logs to sent_emails table
        │
        ▼
   Supabase DB + Storage + Auth
```

---

## 📁 Files in repo

```
/
├── index.html     → /          (login + summary)
├── send.html      → /send      (compose email)
├── sent.html      → /sent      (sent history)
├── view.html      → /view?id=  (full details)
├── _worker.js     → routes .html files to clean URLs
└── README.md      → this file
```

---

# 🧱 PART 1 — Supabase Setup

## Step 1.1 — Create Supabase project

1. Go to [supabase.com](https://supabase.com) → **New project**
2. Pick a name, region, and password
3. Wait for provisioning (~1 min)

Copy these from **Project Settings → API**:
- **Project URL**: `https://xxxx.supabase.co`
- **anon key** (public — safe to expose)
- **service_role key** (secret — server-side only)

---

## Step 1.2 — Create login user

1. Supabase → **Authentication → Users**
2. Click **Add User → Create New User**
3. Fill in:
   - Email: `hr@bettergrowthsolutions.com`
   - Password: *(strong one — remember it)*
   - ✅ **Auto Confirm User**
4. Click **Create User**

---

## Step 1.3 — Create storage bucket

1. Supabase → **Storage → New bucket**
2. Name: `attachments`
3. ✅ **Public bucket**
4. File size limit: `10 MB`
5. Click **Create**

---

## Step 1.4 — SQL: Create `sent_emails` table

Go to **SQL Editor → New query** → paste and run:

```sql
-- Drop if rebuilding from scratch
drop table if exists sent_emails cascade;

create table sent_emails (
  id               uuid primary key default gen_random_uuid(),
  to_email         text,
  cc_email         text,
  bcc_email        text,
  subject          text,
  body             text,
  attachment_name  text,
  attachment_url   text,
  attachment_path  text,
  resend_id        text,
  status           text,
  sent_by          uuid references auth.users(id),
  created_at       timestamptz default now()
);

-- Row Level Security
alter table sent_emails enable row level security;

create policy "users read own sends"
  on sent_emails for select
  to authenticated
  using (auth.uid() = sent_by);

create policy "users insert own sends"
  on sent_emails for insert
  to authenticated
  with check (auth.uid() = sent_by);

create policy "users delete own sends"
  on sent_emails for delete
  to authenticated
  using (auth.uid() = sent_by);
```

---

## Step 1.5 — SQL: Storage policies

**SQL Editor → New query** → paste and run:

```sql
drop policy if exists "auth upload attachments" on storage.objects;
drop policy if exists "auth read attachments"   on storage.objects;
drop policy if exists "auth delete attachments" on storage.objects;

create policy "auth upload attachments"
  on storage.objects for insert
  to authenticated
  with check (bucket_id = 'attachments');

create policy "auth read attachments"
  on storage.objects for select
  to public
  using (bucket_id = 'attachments');

create policy "auth delete attachments"
  on storage.objects for delete
  to authenticated
  using (bucket_id = 'attachments');
```

---

## Step 1.6 — Verify

Run this in SQL Editor:

```sql
select id, email from auth.users;
```

You should see:
```
id                                    | email
82431992-...-...                      | hr@bettergrowthsolutions.com
```

Save this **user ID** — you may need it for debugging.

---

# 🧱 PART 2 — Resend Setup

## Step 2.1 — Create account

1. Go to [resend.com](https://resend.com) → sign up
2. Verify your email

## Step 2.2 — Verify your domain

1. Resend → **Domains → Add Domain**
2. Enter: `bettergrowthsolutions.com`
3. Resend gives you 3 DNS records (SPF, DKIM, MX)
4. Add them to **Cloudflare DNS**:
   - All records must be **DNS only** (grey cloud, not orange)
5. Click **Verify** in Resend (takes 1–5 min)

## Step 2.3 — Create API key

1. Resend → **API Keys → Create API Key**
2. Name: `BetterGrowth Mail`
3. Permission: **Full access**
4. Copy the key (`re_...`) — you'll only see it once

> ⚠️ Never commit this key to GitHub. Store it in Supabase Edge Function secrets only.

---

# 🧱 PART 3 — Supabase Edge Function

## Step 3.1 — Set secrets

1. Supabase → **Edge Functions → Manage Secrets**
2. Add:
   - `RESEND_API_KEY` = `re_xxxxxxxxxxxx`
   - `FROM_EMAIL` = `HR <hr@bettergrowthsolutions.com>`

> `SUPABASE_URL`, `SUPABASE_SERVICE_ROLE_KEY`, `SUPABASE_ANON_KEY` are auto-injected — do not add.

---

## Step 3.2 — Create the Edge Function

1. Supabase → **Edge Functions → Create a new function**
2. Name: `send-email`
3. **Verify JWT**: ❌ OFF *(we verify manually inside code)*
4. Paste the code below → **Deploy**

```ts
import { createClient } from "https://esm.sh/@supabase/supabase-js@2";

const RESEND_API_KEY = Deno.env.get("RESEND_API_KEY")!;
const SUPABASE_URL   = Deno.env.get("SUPABASE_URL")!;
const SERVICE_KEY    = Deno.env.get("SUPABASE_SERVICE_ROLE_KEY")!;
const ANON_KEY       = Deno.env.get("SUPABASE_ANON_KEY")!;
const FROM_EMAIL     = Deno.env.get("FROM_EMAIL") ?? "HR <hr@bettergrowthsolutions.com>";
const BUCKET         = "attachments";

const cors = {
  "Access-Control-Allow-Origin": "*",
  "Access-Control-Allow-Headers": "authorization, content-type",
  "Access-Control-Allow-Methods": "POST, OPTIONS",
};

const json = (data: unknown, status = 200) =>
  new Response(JSON.stringify(data), {
    status,
    headers: { "Content-Type": "application/json", ...cors },
  });

Deno.serve(async (req) => {
  if (req.method === "OPTIONS") return new Response("ok", { headers: cors });

  try {
    const url = new URL(req.url);
    const action = url.searchParams.get("action");

    const admin = createClient(SUPABASE_URL, SERVICE_KEY);

    /* ---------- LOGIN ---------- */
    if (action === "login") {
      const { email, password } = await req.json();
      if (!email || !password) return json({ error: "Email and password required" }, 400);

      const anon = createClient(SUPABASE_URL, ANON_KEY);
      const { data, error } = await anon.auth.signInWithPassword({
        email: String(email).trim().toLowerCase(),
        password: String(password),
      });

      if (error) return json({ error: error.message }, 401);

      return json({
        ok: true,
        access_token: data.session.access_token,
        user: { id: data.user.id, email: data.user.email },
      });
    }

    /* ---------- AUTH ---------- */
    const jwt = (req.headers.get("Authorization") ?? "").replace("Bearer ", "").trim();
    if (!jwt) return json({ error: "Unauthorized" }, 401);

    const { data: userData, error: userErr } = await admin.auth.getUser(jwt);
    if (userErr || !userData?.user) return json({ error: "Unauthorized" }, 401);
    const userId = userData.user.id;

    /* ---------- SEND ---------- */
    if (action === "send") {
      const body = await req.json();
      const { to, cc = "", bcc = "", subject, html, attachment = null } = body;

      if (!to || !subject || !html) {
        return json({ error: "Missing to / subject / html" }, 400);
      }

      let attachmentUrl: string | null = null;
      let attachmentPath: string | null = null;

      if (attachment?.filename && attachment?.content) {
        try {
          const binary = atob(attachment.content);
          const bytes = new Uint8Array(binary.length);
          for (let i = 0; i < binary.length; i++) bytes[i] = binary.charCodeAt(i);

          const safeName = attachment.filename.replace(/[^a-zA-Z0-9._-]/g, "_");
          attachmentPath = `${userId}/${Date.now()}-${safeName}`;

          const { error: upErr } = await admin.storage
            .from(BUCKET)
            .upload(attachmentPath, bytes, {
              contentType: attachment.contentType || "application/octet-stream",
              upsert: false,
            });

          if (upErr) {
            console.error("Storage upload failed:", upErr);
          } else {
            const { data: pub } = admin.storage.from(BUCKET).getPublicUrl(attachmentPath);
            attachmentUrl = pub.publicUrl;
          }
        } catch (e) {
          console.error("Attachment processing failed:", e);
        }
      }

      const payload: Record<string, unknown> = {
        from: FROM_EMAIL,
        to: to.split(",").map((s: string) => s.trim()).filter(Boolean),
        subject,
        html,
      };
      if (cc)  payload.cc  = cc.split(",").map((s: string) => s.trim()).filter(Boolean);
      if (bcc) payload.bcc = bcc.split(",").map((s: string) => s.trim()).filter(Boolean);
      if (attachment?.filename && attachment?.content) {
        payload.attachments = [{ filename: attachment.filename, content: attachment.content }];
      }

      const resendRes = await fetch("https://api.resend.com/emails", {
        method: "POST",
        headers: {
          Authorization: `Bearer ${RESEND_API_KEY}`,
          "Content-Type": "application/json",
        },
        body: JSON.stringify(payload),
      });
      const resendData = await resendRes.json();

      await admin.from("sent_emails").insert({
        to_email: to,
        cc_email: cc,
        bcc_email: bcc,
        subject,
        body: html,
        attachment_name: attachment?.filename ?? null,
        attachment_url: attachmentUrl,
        attachment_path: attachmentPath,
        resend_id: resendData?.id ?? null,
        status: resendRes.ok ? "sent" : "failed",
        sent_by: userId,
      });

      return json(
        { ok: resendRes.ok, resend: resendData, attachment_url: attachmentUrl },
        resendRes.ok ? 200 : 400,
      );
    }

    /* ---------- LIST ---------- */
    if (action === "sent") {
      const { data, error } = await admin
        .from("sent_emails")
        .select("*")
        .eq("sent_by", userId)
        .order("created_at", { ascending: false })
        .limit(200);
      if (error) return json({ error: error.message }, 500);
      return json(data ?? []);
    }

    /* ---------- GET ONE ---------- */
    if (action === "get") {
      const id = url.searchParams.get("id");
      if (!id) return json({ error: "id required" }, 400);

      const { data, error } = await admin
        .from("sent_emails")
        .select("*")
        .eq("id", id)
        .eq("sent_by", userId)
        .maybeSingle();

      if (error) return json({ error: error.message }, 500);
      if (!data) return json({ error: "Not found" }, 404);
      return json(data);
    }

    /* ---------- LAST ---------- */
    if (action === "last") {
      const { data } = await admin
        .from("sent_emails")
        .select("*")
        .eq("sent_by", userId)
        .order("created_at", { ascending: false })
        .limit(1)
        .maybeSingle();
      return json(data ?? null);
    }

    /* ---------- DELETE ---------- */
    if (action === "delete") {
      const { id } = await req.json();
      if (!id) return json({ error: "id required" }, 400);

      const { data: row } = await admin
        .from("sent_emails")
        .select("attachment_path, sent_by")
        .eq("id", id)
        .maybeSingle();

      if (!row || row.sent_by !== userId) return json({ error: "Not found" }, 404);

      if (row.attachment_path) {
        await admin.storage.from(BUCKET).remove([row.attachment_path]);
      }

      await admin.from("sent_emails").delete().eq("id", id);
      return json({ ok: true });
    }

    return json({ error: "Unknown action" }, 404);
  } catch (err) {
    console.error("Function error:", err);
    return json({ error: String(err) }, 500);
  }
});
```

---

## Step 3.3 — Get your function URL

After deploy, the URL is:

```
https://YOUR-PROJECT-REF.supabase.co/functions/v1/send-email
```

Example: `https://cztelemfiqovsoizrohr.supabase.co/functions/v1/send-email`

**Copy this** — it goes into all 4 HTML files.

---

## Step 3.4 — Test the API

Open any browser → DevTools → **Console** → paste (replace URL + password):

```js
fetch("https://YOUR-PROJECT.supabase.co/functions/v1/send-email?action=login", {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({
    email: "hr@bettergrowthsolutions.com",
    password: "YOUR_PASSWORD"
  })
}).then(r => r.json()).then(console.log);
```

**Expected:** `{ ok: true, access_token: "eyJ...", user: {...} }`

---

# 🧱 PART 4 — Cloudflare Pages/Workers Setup

## Step 4.1 — Push files to GitHub

Create a new repo. Add these files:

```
index.html
send.html
sent.html
view.html
_worker.js
README.md
```

**Replace** the placeholder URL in all HTML files:

Find:
```js
const FN = "https://cztelemfiqovsoizrohr.supabase.co/functions/v1/send-email";
```

Replace `cztelemfiqovsoizrohr` with your actual project ref.

---

## Step 4.2 — Connect to Cloudflare

1. Cloudflare Dashboard → **Workers & Pages → Create → Pages → Connect to Git**
2. Select your repo
3. Build settings:
   - Framework preset: **None**
   - Build command: *(leave empty)*
   - Output directory: `/`
4. Click **Save and Deploy**

---

## Step 4.3 — Worker routing (if needed)

If Cloudflare doesn't auto-serve `/view` → `view.html`, add `_worker.js` (see below).

Otherwise skip and rely on Pages default routing.

### `_worker.js`

```js
export default {
  async fetch(request, env, ctx) {
    const url = new URL(request.url);
    let path = url.pathname;

    if (path === "/" || path === "") path = "/index.html";

    if (!path.includes(".")) {
      const asHtml = path + ".html";
      const res = await env.ASSETS.fetch(new Request(new URL(asHtml, url), request));
      if (res.status !== 404) return res;
    }

    return env.ASSETS.fetch(request);
  },
};
```

---

## Step 4.4 — Environment variables

You **don't need any** in Cloudflare. All secrets live in Supabase Edge Function secrets.

✅ Zero secrets in the frontend
✅ Zero secrets in GitHub
✅ Zero secrets in Cloudflare

---

# 🧱 PART 5 — Using the App

## Flow

1. **Login** at `/` with `hr@bettergrowthsolutions.com` + password
2. **Home page** shows the last email sent + **View full details →** link
3. **Send Email** (`/send`):
   - To, CC, BCC (comma-separated for multiple)
   - Subject
   - Body (HTML allowed)
   - Attachment (images auto-compressed to WebP)
   - Click **Send**
4. **Sent** (`/sent`) — table of all emails, click any row
5. **Details** (`/view?id=...`):
   - Full metadata
   - Body rendered + HTML source
   - Image/PDF preview + download
   - **↻ Resend** · **🗑 Delete**

---

## Feature summary

| Feature | Where |
|---|---|
| Login | Supabase Auth (hashed passwords) |
| Send | Resend API via Edge Function |
| CC / BCC | Supported, comma-separated |
| Attachment | Stored in Supabase Storage |
| Image compression | Browser-side (WebP @ 80%, max 1600px) |
| Sent history | `sent_emails` table |
| Full detail view | `/view?id=...` |
| Resend | Re-fetches attachment, re-sends |
| Delete | Removes row + storage file |
| Auth guard | Every page checks token, redirects on 401 |

---

# 🧱 PART 6 — Testing & Debugging

## Test checklist

| # | Test | Expected |
|---|---|---|
| 1 | Login with correct creds | Redirects to summary |
| 2 | Summary shows last email | ✅ or "No emails yet" |
| 3 | Send email without attachment | "✅ Sent!" |
| 4 | Send with image | Preview shows compression saved % |
| 5 | Sent page loads rows | ✅ |
| 6 | Click row | `view?id=...` opens |
| 7 | View page shows image | Image displayed |
| 8 | Resend from view | New row appears |
| 9 | Delete from view | Row + file removed |

---

## Common errors

| Error | Cause | Fix |
|---|---|---|
| `Invalid login credentials` | Wrong password / user not confirmed | Reset in Supabase Auth UI |
| `Unauthorized` | Token expired | Log out, log in again |
| `Missing email ID` | URL lacks `?id=...` | Ensure links use `view?id=` |
| `Not found` | Row's `sent_by` ≠ your user | Check `sent_by` matches auth user |
| `Unknown action` | Function missing block | Redeploy Edge Function |
| `Invalid input syntax for type uuid` | Placeholder wasn't replaced | Use real UUID |
| Image not previewing | `attachment_url` is null | Storage upload failed — check logs |
| 404 on `/view` | Worker not routing | Add `_worker.js` |

---

## Debug queries (SQL Editor)

**List recent emails:**
```sql
select id, to_email, subject, status, attachment_url, sent_by, created_at
from sent_emails
order by created_at desc
limit 20;
```

**Check ownership:**
```sql
select u.email, count(s.id) as emails_sent
from auth.users u
left join sent_emails s on s.sent_by = u.id
group by u.email;
```

**Fix orphaned rows:**
```sql
update sent_emails
set sent_by = (select id from auth.users where email = 'hr@bettergrowthsolutions.com')
where sent_by is null;
```

---

# 🔐 Security Notes

## What's safe to expose
- Supabase **anon / publishable** key — designed for browsers
- Supabase **project URL** — public
- Function URL — public

## What must stay secret
- `RESEND_API_KEY` — Supabase Edge Function secret only
- Supabase **service_role** key — auto-injected into Edge Function, never exposed
- User passwords — never stored anywhere except Supabase Auth (hashed)

## Rules
1. Never commit secrets to GitHub
2. Never hardcode keys in HTML
3. Rotate keys if they leak (Resend → API Keys → Delete → Create new)
4. Keep the bucket public only if URLs are non-guessable (they are: `userId/timestamp-name`)

---

# 📦 Summary

| Layer | Tech | Cost |
|---|---|---|
| Frontend | Cloudflare Workers (static) | Free |
| Backend | Supabase Edge Functions | Free tier |
| DB | Supabase Postgres | Free tier |
| Storage | Supabase Storage | Free 1 GB |
| Auth | Supabase Auth | Free tier |
| Email | Resend | Free 3,000/mo, 100/day |

**Total: ₹0** for typical HR use.

---

# 🚀 Quick reference

## URLs
```
/                    → Login + summary
/send                → Compose
/sent                → History
/view?id=<uuid>      → Details
```

## API
```
POST /functions/v1/send-email?action=login    { email, password }
POST /functions/v1/send-email?action=send     { to, cc, bcc, subject, html, attachment }
GET  /functions/v1/send-email?action=sent
GET  /functions/v1/send-email?action=get&id=<uuid>
GET  /functions/v1/send-email?action=last
POST /functions/v1/send-email?action=delete   { id }
```

All except `login` require: `Authorization: Bearer <access_token>`

---

# ✅ Done

You now have a working mail system with:
- ✅ Custom-domain email sending
- ✅ Attachments + auto compression
- ✅ Full sent history
- ✅ Detailed preview page
- ✅ Resend + Delete
- ✅ Zero cost, zero exposed secrets

Push to GitHub → Cloudflare auto-deploys → login → send. 🎉
