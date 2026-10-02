# 📧 BetterGrowth Mail System

A lightweight, zero-cost web app for sending transactional emails from a custom domain — built with **Cloudflare Workers**, **Supabase**, and **Resend**.

![Status](https://img.shields.io/badge/status-production-green)
![Cost](https://img.shields.io/badge/cost-%E2%82%B90-brightgreen)
![License](https://img.shields.io/badge/license-internal-blue)

---

## ✨ Features

- 🔐 **Secure login** — Supabase Auth (hashed passwords)
- ✉️ **Send emails** — from `hr@bettergrowthsolutions.com`
- 📎 **Attachments** — images auto-compressed (~85% smaller)
- 👥 **CC / BCC** — comma-separated multiple recipients
- 📬 **Sent history** — every email logged
- 🔍 **Detail view** — full preview with image/PDF viewer
- ♻️ **Resend** — re-send any email in one click
- 🗑️ **Delete** — removes record + stored file
- 💰 **Zero cost** — all free tiers

---

## 🏗️ Architecture

```
Browser (Cloudflare Workers)
   │
   │  HTTPS + JWT
   ▼
Supabase Edge Function (Deno)
   ├── Auth check
   ├── Send via Resend API
   ├── Upload attachments to Storage
   └── Log to Postgres
        │
        ▼
   Supabase (Auth + DB + Storage)
```

---

## 🧰 Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend | HTML, CSS, JavaScript |
| Hosting | Cloudflare Workers |
| Backend | Supabase Edge Functions (Deno) |
| Database | Supabase Postgres |
| Storage | Supabase Storage |
| Auth | Supabase Auth |
| Email | Resend API |

---

## 📁 Project Structure

```
/
├── index.html      →  /              (login + summary)
├── send.html       →  /send          (compose form)
├── sent.html       →  /sent          (history table)
├── view.html       →  /view?id=<id>  (full details)
├── _worker.js      →  clean-URL router
└── README.md       →  this file
```

---

## 🚀 Quick Start

### Prerequisites

- GitHub account
- Cloudflare account
- Supabase account
- Resend account
- DNS access to your domain

### Setup Time

~15 minutes

---

## 📦 Installation

### 1️⃣ Supabase Setup

#### 1.1 Create Project
1. Go to [supabase.com](https://supabase.com) → **New Project**
2. Name: `bettergrowth-mail`
3. Save the DB password
4. Wait ~2 minutes

Copy from **Project Settings → API**:
- Project URL
- anon (public) key
- service_role key

#### 1.2 Create Auth User
1. **Authentication → Users → Add User**
2. Email: `hr@bettergrowthsolutions.com`
3. Strong password
4. ✅ **Auto Confirm User**
5. Click **Create User**

#### 1.3 Create Storage Bucket
1. **Storage → New bucket**
2. Name: `attachments`
3. ✅ Public bucket
4. Size limit: `10 MB`

#### 1.4 Create Tables

Run in **SQL Editor**:

```sql
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

#### 1.5 Storage Policies

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

### 2️⃣ Resend Setup

#### 2.1 Create Account
1. Sign up at [resend.com](https://resend.com)
2. Verify email

#### 2.2 Verify Domain
1. **Domains → Add Domain**
2. Enter: `bettergrowthsolutions.com`
3. Add 3 DNS records to Cloudflare (**DNS only**, not proxied):
   - MX
   - SPF (TXT)
   - DKIM (TXT)
4. Click **Verify**

#### 2.3 Create API Key
1. **API Keys → Create API Key**
2. Name: `BetterGrowth Mail`
3. Permission: **Full access**
4. Copy key: `re_xxxxxxxxxxxx`

> ⚠️ Store safely — shown only once.

---

### 3️⃣ Edge Function

#### 3.1 Set Secrets
Supabase → **Edge Functions → Manage Secrets**:

| Name | Value |
|------|-------|
| `RESEND_API_KEY` | `re_xxxxxxxx` |
| `FROM_EMAIL` | `HR <hr@bettergrowthsolutions.com>` |

> `SUPABASE_URL`, `SUPABASE_SERVICE_ROLE_KEY`, `SUPABASE_ANON_KEY` are auto-injected.

#### 3.2 Deploy Function
1. **Edge Functions → Create function**
2. Name: `send-email`
3. Verify JWT: **OFF**
4. Paste code (see below)
5. Click **Deploy**

<details>
<summary><b>📄 Click to view Edge Function code</b></summary>

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

    /* LOGIN */
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

    /* AUTH */
    const jwt = (req.headers.get("Authorization") ?? "").replace("Bearer ", "").trim();
    if (!jwt) return json({ error: "Unauthorized" }, 401);
    const { data: userData, error: userErr } = await admin.auth.getUser(jwt);
    if (userErr || !userData?.user) return json({ error: "Unauthorized" }, 401);
    const userId = userData.user.id;

    /* SEND */
    if (action === "send") {
      const body = await req.json();
      const { to, cc = "", bcc = "", subject, html, attachment = null } = body;
      if (!to || !subject || !html) return json({ error: "Missing to / subject / html" }, 400);

      let attachmentUrl: string | null = null;
      let attachmentPath: string | null = null;

      if (attachment?.filename && attachment?.content) {
        try {
          const binary = atob(attachment.content);
          const bytes = new Uint8Array(binary.length);
          for (let i = 0; i < binary.length; i++) bytes[i] = binary.charCodeAt(i);
          const safeName = attachment.filename.replace(/[^a-zA-Z0-9._-]/g, "_");
          attachmentPath = `${userId}/${Date.now()}-${safeName}`;

          const { error: upErr } = await admin.storage.from(BUCKET).upload(attachmentPath, bytes, {
            contentType: attachment.contentType || "application/octet-stream",
            upsert: false,
          });
          if (!upErr) {
            const { data: pub } = admin.storage.from(BUCKET).getPublicUrl(attachmentPath);
            attachmentUrl = pub.publicUrl;
          }
        } catch (e) { console.error("Upload failed:", e); }
      }

      const payload: Record<string, unknown> = {
        from: FROM_EMAIL,
        to: to.split(",").map((s: string) => s.trim()).filter(Boolean),
        subject, html,
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
        to_email: to, cc_email: cc, bcc_email: bcc, subject, body: html,
        attachment_name: attachment?.filename ?? null,
        attachment_url: attachmentUrl, attachment_path: attachmentPath,
        resend_id: resendData?.id ?? null,
        status: resendRes.ok ? "sent" : "failed",
        sent_by: userId,
      });

      return json({ ok: resendRes.ok, resend: resendData, attachment_url: attachmentUrl },
                  resendRes.ok ? 200 : 400);
    }

    /* LIST */
    if (action === "sent") {
      const { data } = await admin.from("sent_emails").select("*")
        .eq("sent_by", userId).order("created_at", { ascending: false }).limit(200);
      return json(data ?? []);
    }

    /* GET ONE */
    if (action === "get") {
      const id = url.searchParams.get("id");
      if (!id) return json({ error: "id required" }, 400);
      const { data, error } = await admin.from("sent_emails").select("*")
        .eq("id", id).eq("sent_by", userId).maybeSingle();
      if (error) return json({ error: error.message }, 500);
      if (!data) return json({ error: "Not found" }, 404);
      return json(data);
    }

    /* LAST */
    if (action === "last") {
      const { data } = await admin.from("sent_emails").select("*")
        .eq("sent_by", userId).order("created_at", { ascending: false }).limit(1).maybeSingle();
      return json(data ?? null);
    }

    /* DELETE */
    if (action === "delete") {
      const { id } = await req.json();
      const { data: row } = await admin.from("sent_emails")
        .select("attachment_path, sent_by").eq("id", id).maybeSingle();
      if (!row || row.sent_by !== userId) return json({ error: "Not found" }, 404);
      if (row.attachment_path) await admin.storage.from(BUCKET).remove([row.attachment_path]);
      await admin.from("sent_emails").delete().eq("id", id);
      return json({ ok: true });
    }

    return json({ error: "Unknown action" }, 404);
  } catch (err) {
    return json({ error: String(err) }, 500);
  }
});
```

</details>

#### 3.3 Note Your Function URL

```
https://<PROJECT_REF>.supabase.co/functions/v1/send-email
```

---

### 4️⃣ Frontend Deployment

#### 4.1 Update Function URL
In all HTML files, replace:

```js
const FN = "https://YOUR_PROJECT.supabase.co/functions/v1/send-email";
```

#### 4.2 Push to GitHub

```bash
git init
git add .
git commit -m "Initial mail system"
git remote add origin <your-repo-url>
git push -u origin main
```

#### 4.3 Deploy on Cloudflare

1. Cloudflare → **Workers & Pages → Create → Pages**
2. **Connect to Git** → select repo
3. Build settings:
   - Framework: **None**
   - Build command: *(empty)*
   - Output: `/`
4. **Save and Deploy**

#### 4.4 Optional — `_worker.js`

If `/view` returns 404, add this file:

```js
export default {
  async fetch(request, env) {
    const url = new URL(request.url);
    let path = url.pathname;
    if (path === "/" || path === "") path = "/index.html";
    if (!path.includes(".")) {
      const res = await env.ASSETS.fetch(
        new Request(new URL(path + ".html", url), request)
      );
      if (res.status !== 404) return res;
    }
    return env.ASSETS.fetch(request);
  },
};
```

---

## 🎯 Usage

### Login
Open the site → enter credentials → click **Login**

### Send Email
1. Click **✉️ Send Email**
2. Fill To / CC / BCC / Subject / Body
3. Attach file (optional — images compressed automatically)
4. Click **Send**

### View History
- Click **📬 Sent** → click any row
- Or click **View full details →** on home summary

### Detail Actions
- **Rendered** — see body as recipient does
- **HTML source** — raw markup
- **Attachment** — image/PDF preview + download
- **↻ Resend** — send again
- **🗑 Delete** — remove record + file

---

## 🔌 API Reference

**Base URL**
```
https://<PROJECT_REF>.supabase.co/functions/v1/send-email
```

**Auth header for all except login**
```
Authorization: Bearer <access_token>
```

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `?action=login` | Authenticate |
| POST | `?action=send` | Send email |
| GET | `?action=sent` | List emails |
| GET | `?action=get&id=<uuid>` | Fetch one |
| GET | `?action=last` | Latest email |
| POST | `?action=delete` | Delete email |

---

## 🗄️ Database Schema

### Table: `sent_emails`

| Column | Type | Description |
|--------|------|-------------|
| id | uuid | Primary key |
| to_email | text | Recipients |
| cc_email | text | CC list |
| bcc_email | text | BCC list |
| subject | text | Subject |
| body | text | HTML body |
| attachment_name | text | Filename |
| attachment_url | text | Public URL |
| attachment_path | text | Storage path |
| resend_id | text | Provider ID |
| status | text | `sent` / `failed` |
| sent_by | uuid | User FK |
| created_at | timestamptz | Timestamp |

---

## 🔐 Security

| ✅ Safe to expose | ❌ Keep secret |
|---|---|
| Supabase anon key | Resend API key |
| Project URL | Supabase service_role |
| Function URL | User passwords |

**Rules:**
1. Never commit secrets to GitHub
2. Never hardcode keys in HTML
3. Rotate keys immediately if leaked
4. Keep DNS records in **DNS-only** mode
5. RLS enabled on all tables

---

## 🐛 Troubleshooting

| Error | Cause | Fix |
|-------|-------|-----|
| `Invalid login credentials` | Wrong password | Reset in Supabase Auth |
| `Unauthorized` | Expired token | Re-login |
| `Missing email ID` | URL missing `?id=` | Use `view?id=<uuid>` |
| `Not found` | Ownership mismatch | Check `sent_by` |
| `Unknown action` | Function not deployed | Redeploy |
| Image not previewing | `attachment_url` null | Check Storage logs |

**Check logs at:**
- Supabase → Edge Functions → Logs
- Cloudflare → Workers → Logs
- Resend → Emails

---

## 💰 Cost Breakdown

| Service | Free Tier |
|---------|-----------|
| Cloudflare Workers | 100,000 req/day |
| Supabase Auth | Unlimited users |
| Supabase DB | 500 MB |
| Supabase Storage | 1 GB |
| Edge Functions | 500k invocations/mo |
| Resend | 3,000 emails/mo |

**Total: ₹0/month**

---

## 📊 Limits

| Resource | Limit |
|----------|-------|
| Email size | 40 MB |
| Attachment size | 10 MB |
| Emails per month | 3,000 |
| Emails per day | 100 |
| Storage | 1 GB |
| Emails listed | 200 |

---

## 🧪 Testing

### Test Login
```js
fetch("https://<REF>.supabase.co/functions/v1/send-email?action=login", {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({ email: "hr@...", password: "..." })
}).then(r => r.json()).then(console.log);
```

### Test Send
```js
fetch("https://<REF>.supabase.co/functions/v1/send-email?action=send", {
  method: "POST",
  headers: {
    "Content-Type": "application/json",
    "Authorization": "Bearer " + localStorage.getItem("token")
  },
  body: JSON.stringify({
    to: "you@example.com",
    subject: "Test",
    html: "<p>Hello!</p>"
  })
}).then(r => r.json()).then(console.log);
```

---

## 📝 License

Internal use only — BetterGrowth Solutions.

---

## 📞 Support

| Issue | Contact |
|-------|---------|
| Code bugs | Engineering Team |
| Email delivery | Resend Support |
| Database | Supabase Support |
| Hosting | Cloudflare Community |

---

## 🗺️ Roadmap

- [ ] Multi-user support
- [ ] Email templates
- [ ] Scheduled sends
- [ ] Contact book
- [ ] Analytics dashboard
- [ ] Bulk import

---

## 📚 Resources

- [Supabase Docs](https://supabase.com/docs)
- [Resend Docs](https://resend.com/docs)
- [Cloudflare Workers Docs](https://developers.cloudflare.com/workers/)
- [MDN Web Docs](https://developer.mozilla.org/)

---

**Built with ❤️ by BetterGrowth Solutions**
