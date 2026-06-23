# CLAUDE.md — MailSweep

Context for Claude Code working in this repo. (Loaded automatically each session.)

## โปรเจกต์นี้คืออะไร

**MailSweep** — เว็บส่วนตัวสำหรับคัดแยกและล้างเมล 5,000+ ฉบับ ข้ามหลายบัญชี (Gmail + Outlook/Hotmail)
Flow: sort → group → suggest → ผู้ใช้ติ๊กเลือก → batch ย้ายเข้า Trash

**กฎเหล็ก:** ลบแบบ **กู้คืนได้เท่านั้น** (Trash / Deleted Items) — ห้าม permanent delete เด็ดขาด

## สถานะปัจจุบัน (อัปเดต 2026-06-23)

- ✅ เขียนสเปกเสร็จ → [`email-manager-spec.md`](email-manager-spec.md) คือ source of truth ของดีไซน์
- ✅ ลง skill `karpathy-guidelines` ที่ `.claude/skills/` (guidelines ลดความพลาดของ LLM ตอนเขียนโค้ด)
- ⬜ **ยังไม่เริ่มเขียนโค้ด** — อยู่ที่ Phase 1 (Supabase schema) ในแผน section 13 ของสเปก

## ไฟล์สำคัญ

| ไฟล์ | คืออะไร |
|---|---|
| `email-manager-spec.md` | สเปกเต็ม — สถาปัตยกรรม, schema+RLS, OAuth, sync, rule engine, แผนเป็นเฟส |
| `.claude/skills/karpathy-guidelines/` | coding guidelines skill (Think→Simple→Surgical→Goal-driven) |
| `CLAUDE.md` | ไฟล์นี้ — สรุป context ให้ตามไปทำที่เครื่องอื่นได้ |

## การตัดสินใจหลัก (อย่าทวนใหม่)

- **ทำเว็บเอง ไม่ใช้ Cowork/Code** — native connector ทำ batch-trash ไม่ได้ และคัด 5000 ฉบับคืองานของ "แอป" ไม่ใช่ "agent" (เปลือง token + ช้า)
- **Rule engine = ฟรี เป็นค่า default** — AI classify (Claude API, `claude-sonnet-4-6`) เป็น opt-in เฉพาะ bucket `review` เท่านั้น
- **Stack ฟรีล้วน:** GitHub Pages SPA (Bootstrap5+JS) + Supabase (Edge Functions/Deno, Postgres+RLS, Auth, Vault) + Gmail API `gmail.modify` + MS Graph `Mail.ReadWrite` + GitHub Actions keep-alive cron
- **Token security:** ตาราง `oauth_tokens` ไม่มี RLS policy (เข้าได้แค่ service_role), refresh token เข้ารหัสด้วย Vault, secret อยู่ใน Edge Function secrets เท่านั้น

## ❓ คำถามเชิงกลยุทธ์ที่ยังค้าง (ตอบก่อนเริ่มโค้ด)

**นี่คือเครื่องมือ "ล้างครั้งเดียวจบ" หรือ "ใช้ประจำ"?**
- ใช้ประจำ/หลายเครื่อง → เดินตามสเปกเต็มได้เลย (multi-account + keep-alive + login คุ้ม)
- ล้างทีเดียวจบ → สถาปัตยกรรมเต็มอาจ overkill; ทางเบากว่าคือสคริปต์ Deno/Node รันบนเครื่อง (OAuth→list→rule→batchTrash) ไม่ต้อง host/DB

> สเปกปัจจุบันโน้มไปทาง "ใช้ประจำ" — ถ้ายืนยันแบบนั้นก็ลุยตามแผนได้

## ⚠️ ความเสี่ยงที่ต้องจัดการก่อน/ระหว่างเขียนโค้ด (จากรีวิว)

1. **Edge Function timeout ตอน sync** — Gmail ต้องยิง `messages.get` ทีละฉบับ ~5000 ครั้ง เกินลิมิตเวลาต่อ invocation แน่ → ออกแบบ sync เป็น chunk เล็ก (200–500/ครั้ง) คืน cursor แล้ว frontend วนเรียกซ้ำจนจบ (resumable จริงๆ ไม่ใช่รวดเดียว)
2. **PKCE state store** — `/oauth/start` กับ `/oauth/callback` คนละ invocation ต้องมีตารางชั่วคราว map `state → code_verifier` (อย่าฝากไว้ใน state param ตรงๆ)
3. **Frontend ห้าม render พันแถวรวด** — ใช้ virtual scroll / pagination ในแต่ละ bucket
4. **Batch trash สำเร็จบางส่วน** — set `decided_action='trash'` เฉพาะ id ที่ provider ตอบ OK ไม่ใช่ทั้งชุด
5. **Gmail rate limit** — ใส่ exponential backoff รับ `429/403 rateLimitExceeded` (ฝั่ง Graph สเปกใส่แล้ว)

## Workflow / Git conventions

- **`.claude/` ถูก commit เข้า repo โดยตั้งใจ** เพื่อให้ skill/settings ตามไปทุกเครื่อง — `.gitignore` กันแค่ machine-local (`.claude/settings.local.json`)
- Push: `git push origin main` (Git Credential Manager จะเด้ง browser login GitHub ครั้งแรก)
- Remote: `github.com/krirk0137/MailManagement`
- secret/token จริง **ห้าม** commit — อยู่ใน Supabase Edge Function secrets เท่านั้น
