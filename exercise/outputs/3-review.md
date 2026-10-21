# Architecture Review: ADR-0002 Post-Order Processing (ATAM Style)

- **Agent:** Antigravity (Google DeepMind)
- **Model:** Gemini 3.8 Flash (High)

## 1. Quality-attribute scenarios

1. **Latency during Peak Sale:**
   - **Source:** ลูกค้า 200 คน/วินาที
   - **Stimulus:** ทำการสั่งซื้อสินค้าพร้อมกัน (PlaceOrder)
   - **Environment:** ช่วงเทศกาลลดราคา 11.11 ปริมาณโหลดพีก 200 orders/s
   - **Response:** ระบบตัดเงิน, จัดส่งสินค้า, และ publish เหตุการณ์ `OrderPlaced` เข้า RabbitMQ แบบ asynchronous
   - **Response Measure:** Latency p95 ของ `PlaceOrder` ต้องไม่เกิน 500 ms (ปัจจุบัน ~300 ms)

2. **Throughput & External Service Rate Limiting:**
   - **Source:** คำสั่งซื้อที่เกิดขึ้น 200 orders/s
   - **Stimulus:** ข้อความยืนยันคำสั่งซื้อถูกส่งไปยัง MailFast
   - **Environment:** MailFast มีอัตราจำกัด (Rate limit) สูงสุด 50 emails/s และมีความหน่วง p95 อยู่ที่ 1.5 s
   - **Response:** RabbitMQ ทำหน้าที่เป็น Queue Buffer และ Consumer ปรับความเร็วการยิง API ไม่ให้เกินพิกัด
   - **Response Measure:** อัตราการยิงออกไปยัง MailFast ต้องไม่เกิน 50 req/s ตลอดเวลา และไม่มีคำขอใดถูก reject ด้วย HTTP 429

3. **Data Integrity for Loyalty Points:**
   - **Source:** RabbitMQ Consumer
   - **Stimulus:** ข้อความ `OrderPlaced` ถูกส่งซ้ำเนื่องจากการ retry เครือข่าย (Network blip)
   - **Environment:** ภาวะโหลดสูงและการเชื่อมต่อเครือข่ายขัดข้องชั่วคราว
   - **Response:** `loyaltyservice` ตรวจสอบ `order_id` ก่อนทำการบันทึกแต้ม
   - **Response Measure:** แต้มสะสมถูกมอบให้ลูกค้าถูกต้องภายใน 1 นาที และมอบเพียงครั้งเดียวเท่านั้น (0% duplicate awards)

## 2. Findings table

| Type | Finding | Why it matters |
|---|---|---|
| **Non-risk** | การเลือกใช้ RabbitMQ เพื่อแยกการทำงานของ MailFast ออกจาก synchronous checkout path | ช่วยปลดภาระ latency ของ MailFast (p95 1.5s, p99 8s) ออกจาก checkout ทำให้รักษา SLA p95 ≤ 500 ms ได้สำเร็จอย่างแน่นอน |
| **Non-risk** | การใช้ RabbitMQ เป็น buffer สำหรับจำกัด rate limit 50 emails/s | ช่วยซับโหลดพีก 200 orders/s แล้วค่อยๆ ระบายออก 50/s ป้องกัน MailFast ล่มหรือถูกแบน |
| **Trade-off point** | ประสิทธิภาพ (Latency) แลกกับความซับซ้อนในการดูแลระบบ (Operability) | ได้ความเร็วและ decouple บริการ แต่เพิ่มภาระในการดูแล StatefulSet ของ RabbitMQ บน Kubernetes สำหรับทีมขนาดเล็ก 4 คน |
| **Sensitivity point** | ความจุของคิว RabbitMQ และอัตรา Consumer Rate Limit | ในช่วงพีก 200 orders/s คิวอีเมลจะสะสมอัตราสุทธิ 150 ข้อความ/วินาที หากช่วงพีกยาวนาน คิวจะโตขึ้นเรื่อยๆ และส่งผลต่อหน่วยความจำของ RabbitMQ |
| **Risk** | ข้อมูลแต้มสะสมอาจล่าช้าเกิน 1 นาที หากผูกคิวแบบรวมหรือเกิดปัญหา Bottleneck | หากคิวของ `loyalty-service` ติดขัดหรือประมวลผลช้า อาจทำให้ผู้ใช้ไม่ได้แต้มภายใน SLA 1 นาทีตามที่ธุรกิจกำหนด |
| **Risk** | ปัญหา Dual-write / Message loss หาก checkoutservice crash หลังตัดเงินแต่ก่อน publish เข้า RabbitMQ | เงินลูกค้าถูกตัดแล้วแต่ไม่มี event ส่งออก ทำให้ลูกค้าไม่ได้รับทั้งอีเมลยืนยันและแต้มสะสม |

## 3. Missing or weak

- **ขาดการกล่าวถึง Outbox Pattern:** การตัดเงินและ publish ข้อความสู่ RabbitMQ แยกอิสระจากกัน หากเครือข่ายหลุดหรือ pod ดับระหว่างนั้น จะเกิดคำสั่งซื้อที่ไม่มีการส่งอีเมล/แต้ม
- **การระบายคิวอีเมล (Queue Drain Time):** อัตรา 200/s เข้าคิว แต่ออกได้แค่ 50/s หมายความว่าอีเมลของลูกค้าคนท้ายๆ ในช่วงพีก 10 นาที อาจต้องรอในคิวนานกว่า 30 นาทีจึงจะถูกส่ง
- **ตัวเลือก Redis Streams:** ADR ไม่ได้เปรียบเทียบกับ Redis Streams ทั้งที่ระบบมี `redis-cart` อยู่แล้ว ซึ่งอาจช่วยลดภาระไม่ต้องรัน RabbitMQ เพิ่ม

## 4. Verdict

**Accept with changes (อนุมัติโดยมีเงื่อนไขให้ปรับปรุง)**

ข้อปรับปรุงสำคัญ 3 ประการ:
1. **เพิ่ม Transactional Outbox Pattern หรือ Fallback Persistence:** ป้องกันข้อความสูญหายหาก publish เข้า RabbitMQ ล้มเหลวหลังจากตัดเงินสำเร็จ
2. **แยก Queue และ Worker Pool ของ Loyalty Points ออกจาก Email:** เพื่อรับประกันว่าแต้มสะสมจะถึงมือลูกค้าภายใน 1 นาทีอย่างเคร่งครัดโดยไม่โดนคิวอีเมลที่หน่วงเวลามาแย่งคิว
3. **กำหนดนโยบาย Dead-Letter Queue (DLQ) และ Alerting:** สำรองข้อความที่ MailFast ปฏิเสธ (0.5% failure) เพื่อให้ทีมเข้าตรวจสอบและ retry ได้
