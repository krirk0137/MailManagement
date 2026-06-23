# MailSweep — Multi-Account Email Triage & Cleanup (Gmail + Outlook/Hotmail)

**เป้าหมาย:** เว็บส่วนตัวสำหรับ คัดแยก → จัดกลุ่ม → เสนอรายการ → ผู้ใช้ติ๊กเลือก → ลบเป็นชุด (ย้ายเข้า Trash) ครอบคลุมทั้ง Inbox และ Spam หลายบัญชี (Gmail + Hotmail/Outlook) ในหน้าเดียว รองรับ 5,000+ ฉบับ

**กฎเหล็ก:** ลบเฉพาะแบบกู้คืนได้ (Trash / Deleted Items) เท่านั้น — ไม่ทำ permanent delete

---

## 0. ทำไมเลือก "สร้างเว็บเอง" ไม่ใช่ Cowork/Code

| | Cowork / Claude Code | เว็บนี้ |
|---|---|---|
| ไล่อ่าน/คัดแยก 5000 ฉบับ | ช้ามาก + เปลือง token | เร็ว (metadata-only + rule engine) |
| ย้าย Trash / batch ลบ | native connector **ทำไม่ได้** | ได้ (batchModify / $batch) |
| หลายบัญชี Gmail+Hotmail พร้อมกัน | 1 บัญชี/connector | ได้ในหน้าเดียว |
| Flow "ติ๊กเลือกแล้วลบ" | ไม่ครบ | ตรงเป๊ะ |

สำหรับงาน mechanical ปริมาณมาก = งานของ "แอป" ไม่ใช่ "agent"

---

## 1. สถาปัตยกรรม

```
┌────────────────────┐     HTTPS      ┌──────────────────────────┐
│  GitHub Pages SPA  │ ◄────────────► │  Supabase Edge Functions │
│  (Bootstrap5 + JS) │                │  (Deno / TypeScript)     │
│  - login (Supabase │                │  - /oauth/start          │
│    Auth)           │                │  - /oauth/callback       │
│  - dashboard       │                │  - /sync                 │
│  - bucket lists +  │                │  - /trash                │
│    checkboxes      │                │  - /keepalive (cron)     │
└─────────┬──────────┘                └───────┬──────────┬───────┘
          │ read cache (RLS)                  │          │
          ▼                                   ▼          ▼
┌────────────────────┐              ┌──────────────┐ ┌──────────────┐
│ Supabase Postgres  │              │  Gmail API   │ │  MS Graph    │
│ - oauth_tokens     │              │ gmail.modify │ │ Mail.ReadWrite│
│ - accounts         │              └──────────────┘ └──────────────┘
│ - messages_cache   │
│ (RLS ทุกตาราง)      │
└────────────────────┘
                ▲
                │ daily cron (keep-alive + refresh token)
        ┌───────┴────────┐
        │ GitHub Actions │
        └────────────────┘
```

**หลักการแยกหน้าที่**
- อ่าน cache → frontend คุยกับ Supabase ตรงๆ ผ่าน RLS (ไม่ต้องผ่าน function)
- ทุก action ที่แตะ OAuth token หรือเรียก provider API → ผ่าน Edge Function เท่านั้น (token ไม่เคยหลุดถึง client)

---

## 2. Stack & Free-Tier (ฟรีล้วน)

| ส่วน | บริการ | ลิมิตฟรีที่เกี่ยวข้อง |
|---|---|---|
| Frontend host | GitHub Pages | ฟรี static + HTTPS (ใช้เป็น OAuth redirect ได้) |
| Backend | Supabase Edge Functions | 500k invocations/เดือน |
| DB | Supabase Postgres | 500MB (5000 metadata rows = จิ๋วมาก) |
| App login | Supabase Auth | ฟรี |
| Cron | GitHub Actions | public repo ไม่จำกัดนาที |
| Gmail API | Google | 1B quota units/วัน |
| Graph API | Microsoft | ฟรี (มี throttling) |
| AI จัดหมวด (optional) | Claude API | **มีค่าใช้จ่าย** — ทำเป็น opt-in เท่านั้น |

> Supabase free จะ pause project หลังไม่มี activity 7 วัน → keep-alive cron แก้ได้ (นายทำ pattern นี้อยู่แล้ว)

---

## 3. กับดัก OAuth ที่ต้องจัดการ (verified)

### 3.1 Google refresh token หมดใน 7 วัน
- ถ้า consent screen = **Testing** (External) → refresh token ตาย 7 วัน
- **ทางแก้:** ตั้ง publishing status = **In Production** แบบ unverified ได้เลย เพราะเป็น personal-use app (<100 users) → กดผ่านหน้าเตือน "unverified app" ตอน login ครั้งแรก → token อยู่ยาว ไม่ต้องทำ CASA/security review
- ต้องขอ `access_type=offline` + `prompt=consent` ตอน auth ครั้งแรกเพื่อให้ได้ refresh token

### 3.2 สาเหตุอื่นที่ทำให้ Google token ตาย (ออกแบบให้รองรับ)
- ไม่ถูกใช้ 6 เดือน → keep-alive cron แตะ token ทุกวันแก้ได้
- เปลี่ยนรหัสผ่าน Google (เมื่อ token มี Gmail scope) → revoke (เลี่ยงไม่ได้ ต้องมีปุ่ม reconnect)
- เกิน 100 refresh token ต่อ client → ตัวเก่าสุดถูกลบเงียบ

### 3.3 Microsoft
- ไม่มีกับดัก 7 วัน; refresh token ส่วนตัว (MSA) อายุ ~90 วัน แต่ต่ออายุเองเมื่อถูกใช้ → keep-alive ครอบคลุม
- Basic Auth ตายไปแล้วตั้งแต่ ต.ค. 2022 → ต้องใช้ OAuth 2.0 authorization code + PKCE เท่านั้น

### 3.4 จัดการ token หมดอายุแบบ graceful
ทุก function ที่เรียก provider: เจอ `invalid_grant` / 401 → mark account = `reauth_required` → frontend ขึ้นปุ่ม "Reconnect" ไม่ crash

---

## 4. OAuth Scopes

**Gmail** (`https://www.googleapis.com/auth/gmail.modify`)
- ครอบคลุม: อ่าน, label, **ย้าย Trash** — ไม่รวม permanent delete (ตรงกับที่ต้องการ)

**Microsoft Graph** (delegated)
- `Mail.Read`, `Mail.ReadWrite`, `offline_access`, `openid`, `email`, `profile`
- Authority: `https://login.microsoftonline.com/common`
- Azure App audience: **Accounts in any organizational directory and personal Microsoft accounts** (รองรับ outlook.com/hotmail.com/live.com + M365)

---

## 5. Data Model (Postgres + RLS)

### 5.1 ตาราง token (เข้าถึงได้เฉพาะ service role)

```sql
-- token table: ไม่มี policy ให้ authenticated → client อ่านไม่ได้เลย
-- Edge Functions ใช้ service_role key (bypass RLS) เท่านั้น
create table oauth_tokens (
  id           uuid primary key default gen_random_uuid(),
  user_id      uuid not null references auth.users(id) on delete cascade,
  provider     text not null check (provider in ('gmail','outlook')),
  email        text not null,
  refresh_token text not null,             -- เข้ารหัสด้วย Supabase Vault (ดูข้อ 11)
  scopes       text,
  status       text default 'active',      -- active | reauth_required
  updated_at   timestamptz default now(),
  unique (user_id, provider, email)
);
alter table oauth_tokens enable row level security;
-- ไม่สร้าง policy ใดๆ → authenticated/anon เข้าไม่ได้
```

### 5.2 ตาราง account (metadata บัญชี — client อ่านได้)

```sql
create table accounts (
  id           uuid primary key default gen_random_uuid(),
  user_id      uuid not null references auth.users(id) on delete cascade,
  provider     text not null check (provider in ('gmail','outlook')),
  email        text not null,
  status       text default 'active',
  total_cached int default 0,
  last_sync_at timestamptz,
  connected_at timestamptz default now(),
  unique (user_id, provider, email)
);
alter table accounts enable row level security;
create policy "own accounts" on accounts
  for all using (user_id = auth.uid()) with check (user_id = auth.uid());
```

### 5.3 ตาราง cache metadata อีเมล

```sql
create table messages_cache (
  id              bigint generated always as identity primary key,
  account_id      uuid not null references accounts(id) on delete cascade,
  provider_msg_id text not null,
  thread_id       text,
  from_addr       text,
  from_name       text,
  subject         text,
  received_at     timestamptz,
  size_estimate   int,
  folder          text,                  -- inbox | spam | promotions | social | ...
  labels          text[],
  has_unsubscribe boolean default false, -- มี List-Unsubscribe header
  is_unread       boolean,
  is_starred      boolean default false,
  has_attachment  boolean default false,
  category        text,                  -- promo | social | newsletter | notif | personal | receipt | other
  suggested_action text,                 -- trash | keep | review
  decided_action  text,                  -- null | trash | keep (ผู้ใช้ตัดสิน)
  trashed_at      timestamptz,
  synced_at       timestamptz default now(),
  unique (account_id, provider_msg_id)
);
alter table messages_cache enable row level security;
create policy "own messages" on messages_cache
  for all using (
    account_id in (select id from accounts where user_id = auth.uid())
  ) with check (
    account_id in (select id from accounts where user_id = auth.uid())
  );
create index on messages_cache (account_id, suggested_action);
create index on messages_cache (account_id, category);
create index on messages_cache (account_id, received_at desc);
```

### 5.4 RPC สำหรับสรุป bucket (เรียกจาก frontend)

```sql
create or replace function bucket_summary(p_account uuid)
returns table(category text, suggested_action text, cnt bigint)
language sql security invoker stable as $$
  select category, suggested_action, count(*)
  from messages_cache
  where account_id = p_account and decided_action is null
  group by category, suggested_action
  order by cnt desc;
$$;
```

---

## 6. OAuth Setup (ทำครั้งเดียว)

### Google Cloud
1. สร้าง project → เปิด **Gmail API**
2. OAuth consent screen: User type **External**, เพิ่มตัวเองเป็น test user, ใส่ scope `gmail.modify`
3. **กด Publish app → In Production** (จุดนี้สำคัญ แก้ปัญหา 7 วัน) — ยอมรับสถานะ unverified
4. Credentials → OAuth Client ID (Web application) → Authorized redirect URI = `https://<project>.supabase.co/functions/v1/oauth-callback`
5. เก็บ Client ID/Secret ไว้ใน Supabase secrets

### Azure (Microsoft)
1. Entra ID → App registrations → New
2. Supported account types = **Accounts in any organizational directory and personal Microsoft accounts**
3. Redirect URI (Web) = `https://<project>.supabase.co/functions/v1/oauth-callback`
4. API permissions → Microsoft Graph (Delegated): `Mail.Read`, `Mail.ReadWrite`, `offline_access`
5. Certificates & secrets → สร้าง client secret → เก็บใน Supabase secrets

---

## 7. Edge Functions

### 7.1 `/oauth/start?provider=gmail|outlook`
- สร้าง PKCE `code_verifier`/`code_challenge` (เก็บ verifier ใน state ชั่วคราว)
- redirect ไป consent URL
  - Gmail: `...&access_type=offline&prompt=consent&scope=...gmail.modify`
  - Graph: `...&scope=Mail.ReadWrite offline_access ...`

### 7.2 `/oauth/callback`
- แลก `code` → tokens (ใช้ code_verifier)
- ดึง email เจ้าของ → upsert `accounts` + เก็บ `refresh_token` (เข้ารหัส) ใน `oauth_tokens`
- redirect กลับ SPA

### 7.3 `/sync?account_id=...` (ดูข้อ 8)
### 7.4 `/trash` (ดูข้อ 10)
### 7.5 `/keepalive` (ดูข้อ 12)

> Helper ที่ทุก function ใช้ร่วม: `getAccessToken(account_id)` — อ่าน refresh token (service role) → ขอ access token ใหม่ → ถ้า `invalid_grant` ตั้ง `status='reauth_required'`

---

## 8. Sync Strategy สำหรับ 5,000+ ฉบับ (metadata-only)

**Gmail** — แบ่ง bucket ด้วย query แล้วดึงเฉพาะ header
```
messages.list  q="in:inbox category:promotions"   → promo
messages.list  q="in:inbox category:social"        → social
messages.list  q="in:inbox older_than:1y"          → old
messages.list  q="in:spam"                          → spam
messages.list  q="is:starred OR is:important"       → keep signals
```
- ต่อ id ดึง `messages.get?format=metadata&metadataHeaders=From&metadataHeaders=Subject&metadataHeaders=Date&metadataHeaders=List-Unsubscribe`
- ยิงแบบ concurrency จำกัด (เช่น 10–15 พร้อมกัน) เคารพ rate limit (250 quota units/user/วินาที)
- upsert ลง `messages_cache` — ออกแบบให้ **resumable** (sync รอบถัดไปข้ามตัวที่ cache แล้วด้วย `synced_at`)

**Graph** — list คืน metadata ครบในตัว (เร็วกว่า Gmail)
```
GET /me/messages?$select=id,from,subject,receivedDateTime,isRead,
    hasAttachments,bodyPreview&$top=100      → วนด้วย @odata.nextLink
GET /me/mailFolders/junkemail/messages?...   → spam
```

**ปริมาณ:** 5000 ฉบับ ≈ Gmail get 5000 ครั้ง (~25k quota units) — ห่างจากเพดาน 1B/วันมาก ไม่มีปัญหา

---

## 9. Rule Engine — คัดว่า "ลบได้/เก็บ" (ไม่ใช้ AI = ฟรี)

จัด `category` + `suggested_action` ตอน sync:

**suggested_action = `trash` (ติ๊กไว้ล่วงหน้า)**
- `category:promotions` อายุ > 90 วัน
- `category:social` (แจ้งเตือนโซเชียล)
- มี `List-Unsubscribe` + ยังไม่อ่าน + เก่ากว่า 30 วัน (newsletter ที่ไม่เคยเปิด)
- ผู้ส่ง `no-reply@/noreply@` อายุ > 90 วัน
- ทุกฉบับใน Spam

**suggested_action = `keep` (ไม่ติ๊กให้)**
- starred / flagged / Gmail `IMPORTANT`
- มี attachment
- อยู่ใน thread ที่เคยตอบ / ผู้ส่งเป็น contact
- ใบเสร็จ/การเงิน (โดเมนธนาคาร/ payment ที่ whitelist)

**suggested_action = `review` (แสดง ไม่ติ๊ก)**
- ที่เหลือทั้งหมด

> เก็บกฎเป็น config object ปรับได้ ไม่ฝัง hardcode
> **(Optional, มีค่าใช้จ่าย)** เฉพาะ bucket `review` ค่อยส่งเข้า Claude API (`claude-sonnet-4-6`) เป็น batch ให้ช่วยจัด — เปิด/ปิดได้ ค่า default = ปิด

---

## 10. Flow รีวิว → ลบ (ตามที่ออกแบบ)

1. Dashboard แสดงจำนวนต่อ bucket ต่อบัญชี (จาก `bucket_summary`)
2. กดเข้า bucket → ตารางรายการ + checkbox (trash-candidate ติ๊กมาให้, keep ไม่ติ๊ก)
3. ผู้ใช้ปรับ checkbox ตามใจ → กด **"ย้าย N ฉบับเข้า Trash"**
4. Modal ยืนยัน → ยิง `/trash`
5. อัปเดต `decided_action='trash'`, `trashed_at=now()` ในแถวที่ลบ → หายจากรายการ

### Batch trash (recoverable เท่านั้น)
**Gmail** — `users.messages.batchModify` (สูงสุด 1000 id/ครั้ง)
```json
POST /gmail/v1/users/me/messages/batchModify
{ "ids": ["..."], "addLabelIds": ["TRASH"], "removeLabelIds": ["INBOX"] }
```
> เพิ่ม label `TRASH` = ย้ายเข้าถังขยะ (กู้คืนได้ 30 วัน) — ไม่ใช่ลบถาวร

**Graph** — `$batch` (สูงสุด 20 sub-request/ครั้ง) แต่ละตัว move ไป `deleteditems`
```json
POST /v1.0/$batch
{ "requests": [
  { "id":"1","method":"POST","url":"/me/messages/{id}/move",
    "body":{"destinationId":"deleteditems"},
    "headers":{"Content-Type":"application/json"} }
]}
```
> chunk เป็นชุดละ 20, เคารพ 429 + `Retry-After`

---

## 11. ความปลอดภัย Token

- เก็บ refresh token ใน `oauth_tokens` เข้ารหัสด้วย **Supabase Vault** (หรือ pgsodium) — ไม่เก็บ plain text
- ตาราง token **ไม่มี RLS policy** สำหรับ client → อ่านได้เฉพาะ Edge Function (service role)
- Client secret ของ Google/Microsoft + encryption key → เก็บใน **Supabase Edge Function secrets** เท่านั้น ไม่ฝังใน frontend/repo
- ไม่ใส่ข้อมูลส่วนตัว/token ลงใน URL query string

---

## 12. Keep-Alive (GitHub Actions)

```yaml
# .github/workflows/keepalive.yml
name: keepalive
on:
  schedule: [{ cron: "0 3 * * *" }]   # ทุกวัน 03:00 UTC
  workflow_dispatch:
jobs:
  ping:
    runs-on: ubuntu-latest
    steps:
      - name: ping supabase keepalive function
        run: curl -fsS "$SUPABASE_KEEPALIVE_URL" -H "Authorization: Bearer $KEY"
        env:
          SUPABASE_KEEPALIVE_URL: ${{ secrets.SUPABASE_KEEPALIVE_URL }}
          KEY: ${{ secrets.SUPABASE_ANON_KEY }}
```
`/keepalive` ทำ 2 อย่าง: (1) query เบาๆ กัน Supabase pause (2) วน refresh token ทุกบัญชี กัน idle 6 เดือน + ตรวจสถานะ

---

## 13. แผนสร้างเป็นเฟส

1. **Foundation** — Supabase project, schema + RLS, Supabase Auth (login ตัวเอง)
2. **OAuth setup** — Google Cloud (Production-unverified) + Azure app registration
3. **OAuth functions** — `/oauth/start` + `/oauth/callback` + เก็บ token + Vault
4. **Sync** — Gmail + Graph metadata pull → cache (resumable)
5. **Rule engine** — จัด category/suggested_action
6. **Frontend** — dashboard + bucket list + checkbox UI
7. **Trash** — `/trash` batch (Gmail batchModify / Graph $batch)
8. **Keep-alive** — GitHub Action + `/keepalive`
9. **(Optional)** — ปุ่มเปิด AI classify สำหรับ bucket `review`

---

## 14. นอก Scope (ตั้งใจไม่ทำ)
- Permanent delete (ต้อง verify app + scope เต็ม + กู้คืนไม่ได้) — ใช้ Trash พอ
- หลาย Gmail พร้อมกันเกิน 100 refresh token/client (เคสไม่เกิดในการใช้ส่วนตัว)
- ส่ง/ตอบอีเมล (คนละ scope คนละงาน — เพิ่มทีหลังได้ถ้าต้องการ)
