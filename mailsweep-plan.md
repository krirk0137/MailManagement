# MailSweep — Build Plan

> เว็บคัดแยก/ลบอีเมลหลายบัญชี (Gmail + Outlook/Hotmail) รองรับ 5,000+ ฉบับ
> Flow: คัดแยก → จัดกลุ่ม → ติ๊กเลือก → ลบเป็นชุด (ย้าย Trash เท่านั้น ไม่ลบถาวร)
> Stack: GitHub Pages (Bootstrap 5 + vanilla JS) + Supabase (Edge Functions + Postgres) + GitHub Actions
> **โปรเจกต์ greenfield — ยังไม่เริ่ม build**

---

## 0. สถานะปัจจุบัน (จุดที่ค้างไว้)

- [x] สเปกสถาปัตยกรรม + schema + 9-phase plan เขียนครบแล้ว (ดูไฟล์ `email-manager-spec.md` เดิม)
- [ ] ยังไม่เริ่มเขียนโค้ดเลย — เริ่มที่ Phase 1
- [ ] **ของใหม่:** มี local Qwen (qwen3.6-35B-A3B) ของเพื่อนแล้ว → เพิ่ม AI classification layer ได้ (เดิม spec ตั้งใจใช้ rule-based ล้วนเพื่อเลี่ยงค่า token — ตอนนี้ AI ฟรีแล้ว)

**เริ่มที่นี่เมื่อกลับบ้าน → ไปที่ "Phase 1" ด้านล่าง**

---

## 1. สถาปัตยกรรม (เดิม + ส่วนเพิ่ม)

```
┌────────────────────┐     HTTPS      ┌──────────────────────────┐
│  GitHub Pages SPA  │ ◄────────────► │  Supabase Edge Functions │
│  (Bootstrap5 + JS) │                │  (Deno / TypeScript)     │
│  - login           │                │  - /oauth/start          │
│  - dashboard       │                │  - /oauth/callback       │
│  - bucket lists +  │                │  - /sync                 │
│    checkboxes      │                │  - /trash                │
│                    │                │  - /classify  ◄── ใหม่   │
│                    │                │  - /keepalive (cron)     │
└─────────┬──────────┘                └──┬─────────┬─────────┬───┘
          │ read cache (RLS)             │         │         │
          ▼                              ▼         ▼         ▼
┌────────────────────┐          ┌──────────┐ ┌────────┐ ┌──────────────┐
│ Supabase Postgres  │          │ Gmail API│ │MS Graph│ │ Qwen (เพื่อน) │
│ - oauth_tokens     │          │ .modify  │ │.ReadWr │ │ ผ่าน tunnel   │
│ - accounts         │          └──────────┘ └────────┘ └──────────────┘
│ - messages_cache   │
│ (RLS ทุกตาราง)      │
└────────────────────┘
        ▲
        │ daily cron (keep-alive + refresh token)
   ┌────┴───────────┐
   │ GitHub Actions │
   └────────────────┘
```

**หลักการแยกหน้าที่ (ไม่เปลี่ยน):**
- อ่าน cache → frontend คุย Supabase ตรง ๆ ผ่าน RLS
- action ที่แตะ OAuth token / เรียก provider API / เรียก Qwen → ผ่าน Edge Function เท่านั้น (key/token ไม่หลุดถึง browser)

---

## 2. ส่วนเพิ่มหลัก: AI Classification แบบ Hybrid

แนวคิด: **rule-based ก่อน แล้วค่อยส่งเฉพาะตัวที่ rule ไม่มั่นใจไปให้ Qwen** — เร็ว ประหยัด และไม่พังถ้าเครื่องเพื่อนปิด

```
อีเมลเข้า cache
   │
   ▼
[Rule engine]  ← เร็ว/ฟรี/deterministic
   ├─ มั่นใจ (เช่น มี List-Unsubscribe header, โดเมน promo ที่รู้จัก) → ตี action เลย
   └─ ไม่มั่นใจ / กำกวม
          │
          ▼
   [/classify → Qwen]  ← batch 20–50 ฉบับ/ครั้ง
          └─ ได้ action + confidence → เก็บลง messages_cache
```

**ทำไม hybrid:** rule จัดการพวกชัด ๆ (newsletter, no-reply, โดเมนซ้ำ) ได้ฟรีและเร็ว เหลือพวกกำกวม (อีเมลคนจริงปนโปรโมชัน, ภาษาไทย/อังกฤษผสม) ที่ rule แพ้ ค่อยใช้ AI — และถ้าเครื่องเพื่อนปิด ก็ยัง fallback เป็น rule-only ได้ ไม่ตาย

**Edge Function `/classify` (sketch):**
```typescript
// supabase/functions/classify/index.ts  — NEW
const QWEN_BASE  = Deno.env.get("QWEN_BASE_URL")!;  // public URL จาก tunnel เพื่อน
const QWEN_KEY   = Deno.env.get("QWEN_API_KEY")!;
const QWEN_MODEL = Deno.env.get("QWEN_MODEL")!;     // เช่น "qwen3.6-35b-a3b"

// รับ array ของอีเมล (id, from, subject, snippet, hasUnsubLink)
// → ส่งเข้า Qwen แบบ batch, temperature 0, บังคับ JSON array กลับมา
const SYSTEM = `
You are an email triage classifier. Input: JSON array of emails.
Output ONLY a JSON array, no prose, no markdown fences. One object per email:
{ "id": "<echo>", "action": "keep|archive|unsubscribe|spam", "confidence": 0.0-1.0 }
Rules:
- Receipts/tickets/personal/bank -> keep
- Newsletters/promos with unsubscribe + low value -> unsubscribe
- Phishing/junk -> spam
- If unsure -> keep (low confidence). Never tell the user to delete on doubt.
`;
```

> หมายเหตุ: action AI = "เสนอ" เท่านั้น ผู้ใช้ต้องติ๊กยืนยันก่อนลบจริงเสมอ (กฎเหล็กเดิมยังอยู่: ลบ = ย้าย Trash, กู้คืนได้)

---

## 3. แผนงานแบ่งเฟส

> MVP = Phase 1–6 (เดินได้เองโดยไม่ต้องมี Qwen). Phase 7 ค่อยเสริม AI.

**Phase 1 — Setup**
- [ ] สร้าง repo + โครง GitHub Pages (Bootstrap 5 + vanilla JS, no build step)
- [ ] สร้าง Supabase project, ตั้ง Data API: Enable Data API **ON**, auto-expose new tables **OFF**, automatic RLS **ON**
- [ ] หน้า login (Supabase Auth)

**Phase 2 — Schema + RLS**
- [ ] ตาราง `oauth_tokens`, `accounts`, `messages_cache` + เปิด RLS ทุกตาราง + policy "เห็น/แก้เฉพาะของตัวเอง"
- [ ] `05_grants.sql` ให้สิทธิ์ตารางแบบ explicit (Supabase ใหม่ไม่ auto-grant แล้ว)
- [ ] เก็บ token แบบเข้ารหัส (Supabase Vault)

**Phase 3 — OAuth (ส่วนยากสุด)**
- [ ] Edge Function `/oauth/start` + `/oauth/callback` สำหรับ Google (scope `gmail.modify`)
- [ ] เพิ่ม Azure (scope `Mail.ReadWrite`) สำหรับ Outlook/Hotmail
- [ ] **ปม Google refresh token หมดอายุ 7 วันใน Testing mode** → publish เป็น Production (unverified) แก้ได้สำหรับ personal use <100 users

**Phase 4 — Sync + Cache**
- [ ] `/sync` ดึง metadata อีเมล (header/subject/snippet เท่านั้น ไม่ดึง body เต็ม) ลง `messages_cache`
- [ ] รองรับหลายบัญชีในหน้าเดียว

**Phase 5 — Rule Engine + UI**
- [ ] rule-based categorize (List-Unsubscribe header, โดเมน, keyword) → ตี bucket
- [ ] Dashboard: bucket lists + checkboxes (ติ๊กเลือกเป็นชุด)
- [ ] กรอง Inbox/Spam, ค้นหา

**Phase 6 — Batch Trash + Keep-alive**
- [ ] `/trash`: Gmail `batchModify` (≤1000 id/ครั้ง), Graph `$batch` (≤20 sub-req) → ย้าย Trash/Deleted Items เท่านั้น
- [ ] GitHub Actions daily: keep-alive (กัน Supabase pause 7 วัน) + refresh OAuth token

**Phase 7 — AI Layer (ต้องมี Qwen เพื่อน)** ← เสริม
- [ ] tunnel เครื่องเพื่อน (Cloudflare Tunnel ฟรี) → ได้ public URL
- [ ] Edge Function `/classify` + ตั้ง secret (QWEN_BASE_URL / QWEN_API_KEY / QWEN_MODEL)
- [ ] ต่อ hybrid flow: rule → ส่งตัวกำกวมเข้า `/classify` → เก็บ action + confidence
- [ ] UI แสดง confidence, ตั้ง threshold ก่อนติ๊กลบเป็นชุด

---

## 4. Qwen integration — รายละเอียดที่ต้องรู้

- โมเดล: **qwen3.6-35B-A3B** (MoE 35B/active 3B, OpenAI-compatible endpoint)
- ต้องขอเพื่อน: base URL + port, ชื่อ model, context window, ยืนยัน public URL (tunnel)
- เรียกผ่าน Edge Function เท่านั้น (อย่าใส่ key ใน frontend — static site = key หลุด)
- **Privacy:** อีเมลวิ่งผ่านเครื่องเพื่อน → โอเคสำหรับเมลส่วนตัว/junk แต่ **ห้ามใช้กับ work email**
- **Availability:** เครื่องเพื่อนปิด = ข้าม AI, fallback rule-only (ออกแบบมาให้ไม่พัง)

---

## 5. Build ที่บ้านด้วย Claude Code + Qwen (ออปชัน)

อยากให้ Qwen ของเพื่อนเป็นตัว "เขียนโค้ด" โปรเจกต์นี้ก็ได้ (ฟรี, qwen3.6 เป็น coding-agent model):
- ถ้าเพื่อนรันผ่าน **LM Studio 0.4.1+** → มี endpoint `/v1/messages` แบบ Anthropic native ชี้ Claude Code มาตรงได้เลย
- ถ้ารัน **vLLM/Ollama เปล่า** → ตั้ง **LiteLLM** เป็น gateway แปลง OpenAI → Anthropic format
- env: `ANTHROPIC_BASE_URL`, `ANTHROPIC_AUTH_TOKEN`, `ANTHROPIC_MODEL=qwen3.6-35b-a3b`, `ANTHROPIC_SMALL_FAST_MODEL` (ปล่อย `ANTHROPIC_API_KEY` ว่าง)
- วิธีสั่ง: วางไฟล์นี้ใน repo แล้วบอก *"อ่าน mailsweep-plan.md แล้วเริ่ม Phase 1"*
- ⚠️ ใช้กับ personal project นี้ได้ แต่อย่าเอา Claude Code+Qwen ไปรันโค้ดงานบริษัท

---

## 6. ต้องเช็ค/ตัดสินใจ

- [ ] เพื่อนรัน Qwen ด้วยอะไร (Ollama / vLLM / LM Studio) → กำหนดว่าต้อง LiteLLM ไหม
- [ ] ยืนยัน Supabase free-tier limits ปัจจุบันที่ supabase.com/pricing (pause 7 วัน, ไม่มี auto backup)
- [ ] ตัดสินใจ: ทำ Outlook/Hotmail ตั้งแต่แรก หรือ Gmail-only ก่อนแล้วค่อยเพิ่ม
