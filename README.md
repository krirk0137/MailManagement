# MailSweep

เว็บส่วนตัวสำหรับคัดแยกและล้างเมล **5,000+ ฉบับ** ข้ามหลายบัญชี (Gmail + Outlook/Hotmail) ในหน้าเดียว
คัดแยก → จัดกลุ่ม → เสนอรายการ → ติ๊กเลือก → ลบเป็นชุด (ย้ายเข้า Trash)

> **กฎเหล็ก:** ลบแบบกู้คืนได้เท่านั้น (Trash / Deleted Items) — ไม่มี permanent delete

## Stack (ฟรีล้วน)

- **Frontend:** GitHub Pages SPA (Bootstrap 5 + vanilla JS)
- **Backend:** Supabase Edge Functions (Deno / TypeScript)
- **DB / Auth:** Supabase Postgres + RLS, Supabase Auth, Vault (เก็บ token)
- **Providers:** Gmail API (`gmail.modify`) · Microsoft Graph (`Mail.ReadWrite`)
- **Cron:** GitHub Actions (keep-alive + refresh token)
- **AI (optional, opt-in):** Claude API — ใช้จัดหมวดเฉพาะ bucket `review`

## สถานะ

📐 **ช่วงออกแบบ** — สเปกเสร็จแล้ว ยังไม่เริ่มเขียนโค้ด

- ดีไซน์เต็ม: [`email-manager-spec.md`](email-manager-spec.md)
- บริบทสำหรับ Claude Code: [`CLAUDE.md`](CLAUDE.md)
- แผนเป็นเฟส: ดู section 13 ของสเปก (Phase 1 = Supabase schema + RLS)

## License

[MIT](LICENSE)
