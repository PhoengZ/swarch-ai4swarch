**ภาษา:** ภาษาไทย | [English](00-prompt-log.en.md)

# บันทึกการใช้ prompt

**ชื่อ / รหัสนิสิต:** เผ่าลาภ กุลธีระ (Phaolap Kulteera) / 6630199021
**agent ที่ใช้:** Antigravity (Gemini Code Assist / CLI)
**โมเดล:** Gemini 3.8 Flash (High)
**แพ็กเกจที่ใช้:** Gemini รุ่นฟรี

บันทึกทุกคำขอที่ส่งให้ agent ช่อง "ไฟล์ที่ agent อ่าน" คือไฟล์ที่ agent รายงาน หรือที่นิสิตเห็นว่า agent เปิด

| # | งาน / run | agent + โมเดล | ข้อความที่ส่ง (ย่อ) | ไฟล์ที่ agent อ่าน | อยู่ในขอบเขตหลักฐานที่อนุญาตหรือไม่ | มีประโยชน์ (1–5) | หมายเหตุ |
|---|---|---|---|---|---|---|---|
| 1 | 1-A | Antigravity (Gemini 3.8 Flash) | `prompts/1-recover-architecture.md` run A | `system/online-boutique/protos/demo.proto` | ใช่ | 4 | กู้คืน interface ทั้ง 9 services ได้ครบ แต่ไม่มี caller dependencies |
| 2 | 1-B | Antigravity (Gemini 3.8 Flash) | `prompts/1-recover-architecture.md` run B | `demo.proto`, `kubernetes-manifests/*.yaml` (13 ไฟล์) | ใช่ | 5 | กู้คืน frontend, redis-cart และ caller dependencies ได้ครบ 17 เส้น |
| 3 | 2-a | Antigravity (Gemini 3.8 Flash) | `prompts/2-write-adr.md` run a (zero-shot) | `docs/adr/template.md` | ใช่ | 3 | แนะนำ Message Queue แต่ไม่มีตัวเลข SLA หรือข้อจำกัดของ MailFast |
| 4 | 2-b | Antigravity (Gemini 3.8 Flash) | `prompts/2-write-adr.md` run b (with context) | `docs/scenario.md`, `docs/adr/0001-...`, `checkoutservice/main.go` | ใช่ | 5 | นำตัวเลข p95 ≤ 500ms, โหลดพีก 200/s, MailFast 50/s rate limit มาออกแบบ RabbitMQ ได้แม่นยำ |
| 5 | 3 | Antigravity (Gemini 3.8 Flash) | `prompts/3-review-adr.md` | `docs/adr/0002-...`, `docs/scenario.md` | ใช่ | 5 | จับจุดเสี่ยงเรื่อง Message loss (Dual-write) และ Queue drain latency ได้ดี |
| 6 | Bonus | Antigravity (Gemini 3.8 Flash) | `prompts/bonus-recover-decision.md` | `demo.proto`, `checkoutservice/main.go`, `frontend/main.go` | ใช่ (อ่านเฉพาะ evidence ที่กำหนด) | 5 | กู้คืน ADR เรื่อง gRPC และ proto กลางได้ดี พร้อมตารางแยก evidenced/inferred ชัดเจน |
| 7 | | | | | | | |
