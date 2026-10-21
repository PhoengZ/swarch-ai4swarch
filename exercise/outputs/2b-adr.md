# ADR-0002: ใช้ RabbitMQ สำหรับการประมวลผลกระบวนการหลังการสั่งซื้อสินค้า (R-12)

- **Agent:** Antigravity (Google DeepMind)
- **Model:** Gemini 3.8 Flash (High)
- **Status:** Proposed
- **Date:** 2026-10-21
- **Deciders:** ทีมสถาปัตยกรรม Online Boutique

## Context and problem

ระบบ Online Boutique มีความต้องการทางธุรกิจใหม่ (R-12) สำหรับกระบวนการหลังจากที่มีการสั่งซื้อสินค้าสำเร็จ ได้แก่:
1. ส่งอีเมลยืนยันคำสั่งซื้อผ่านผู้ให้บริการภายนอกรายใหม่ **MailFast** แทนที่ mock logic ใน `emailservice`
2. มอบแต้มสะสม (**Loyalty points**) ผ่านบริการใหม่ `loyaltyservice`
3. บันทึกเหตุการณ์ **Analytics "order placed"** สำหรับทีมการตลาด
4. เตรียมรองรับงานเพิ่มเติมในปีหน้าตามแผนของทีมการตลาด (SMS, Fraud scoring, Recommendations update)

ปัจจุบันใน `checkoutservice.PlaceOrder` (Go) การเรียกบริการหลังการชำระเงินถูกทำแบบ synchronous ผ่าน gRPC ต่อท้ายการตัดเงินและการส่งสินค้า ซึ่ง latency ปัจจุบันอยู่ที่ประมาณ 300 ms อย่างไรก็ตาม ข้อเท็จจริงและข้อจำกัดทางเทคนิคมีดังนี้:
- **Checkout latency SLA:** p95 ของ `PlaceOrder` ต้องไม่เกิน **500 ms**
- **ปริมาณงานช่วงพีก:** **200 orders/s** ในช่วงเทศกาลลดราคา 11.11
- **ข้อจำกัดของ MailFast:** latency สูงมาก (p95 **1.5 s**, p99 **8 s**), มีโอกาสล้มเหลว **0.5%**, และถูกจำกัดอัตราส่ง (Rate limit) ที่ **50 emails/s** (ซึ่งต่ำกว่าพีก 200 orders/s ถึง 4 เท่า)
- **เงื่อนไขแต้มสะสม:** ต้องได้รับแต้ม**ภายใน 1 นาที**หลังการสั่งซื้อ และต้องได้**ครั้งเดียวเท่านั้น (Exactly-once)** ห้ามได้แต้มซ้ำ
- **ความพร้อมใช้งาน (Availability):** การสั่งซื้อต้อง**ไม่ล้มเหลว**หากระบบอีเมล แต้มสะสม หรือ analytics มีปัญหา
- **ทักษะทีมและโครงสร้างพื้นฐาน:** ทีมพัฒนา 4 คนเชี่ยวชาญ Go และ gRPC โดยมี 2 คนเคยใช้งาน **RabbitMQ** มาก่อน; ระบบทำงานบน Kubernetes และมีนโยบายเลือกใช้คอมโพเนนต์แบบ **Open-source ติดตั้งเอง** มากกว่า Cloud-managed service

หากใช้การเรียกแบบ synchronous แบบเดิม Latency p95 ของ checkout จะพุ่งสูงเกิน 1.8 วินาที (หลุด SLA 500 ms) และจะล่มทันทีในช่วงพีกเนื่องจากข้อจำกัด 50 emails/s ของ MailFast

## Decision drivers

1. **Performance (Latency):** รักษา p95 ของ `PlaceOrder` ให้ไม่เกิน 500 ms ภายใต้โหลดพีก 200 orders/s
2. **Availability & Fault Tolerance:** การสั่งซื้อสำเร็จต้องไม่ขึ้นอยู่กับความพร้อมของระบบภายนอกหรือบริการเสริม (Failure isolation)
3. **Throughput & Rate Limiting:** รองรับพีก 200 orders/s และจำกัดอัตราการยิงออกไปยัง MailFast ไม่ให้เกิน 50 req/s (Buffer & Throttling)
4. **Data Integrity (Exactly-once processing):** ให้แต้มสะสมครั้งเดียวเท่านั้นและไม่เกิน 1 นาที
5. **Modifiability:** รองรับการเพิ่ม subscriber รายใหม่ในปีหน้า (SMS, Fraud check) ได้อย่างอิสระ
6. **Operability & Team Familiarity:** ต้องเป็น Open-source รันบน Kubernetes และตรงกับทักษะของทีม 4 คน

## Considered options

1. **Synchronous gRPC Calls with Goroutines:** เรียก `emailservice` และ `loyaltyservice` ภายใน `checkoutservice` โดยแยกเป็น goroutine ทำงานเบื้องหลังในหน่วยความจำ
2. **RabbitMQ (AMQP Message Broker แบบ Self-hosted):** ติดตั้ง RabbitMQ บน Kubernetes ใช้ Topic/Fanout Exchange เผยแพร่เหตุการณ์ `order.placed` ไปยังคิวต่างๆ
3. **Apache Kafka (Event Streaming Platform แบบ Self-hosted):** ติดตั้ง Kafka Cluster บน Kubernetes เพื่อจัดเก็บ event stream ของคำสั่งซื้อ

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| 1. Goroutine ในหน่วยความจำ | เร็ว, ไม่ต้องติดตั้งระบบใหม่, ทีมถนัด Go อยู่แล้ว | เสี่ยงข้อมูลสูญหายสูงมากหาก Pod ถูก restart หรือ crash; ไม่สามารถ buffer ข้อความ 200/s เพื่อจำกัดอัตรา 50/s ของ MailFast ข้าม pod ได้อย่างปลอดภัย; จัดการ retry ข้ามเครื่องไม่ได้ |
| 2. RabbitMQ | รองรับ Publish/Subscribe (Fanout/Topic Exchange) ได้ตรงจุด; มีระบบ Queuing buffer ชะลอโหลด 200/s ให้ส่งออก 50/s สู่ MailFast ได้ง่าย; ติดตั้งและดูแลง่ายบน Kubernetes แบบ Open-source; สมาชิกทีม 2 คนมีประสบการณ์ตรง | ต้องดูแล StatefulSet บน Kubernetes เพิ่ม 1 คลัสเตอร์; ต้องออกแบบ Idempotency key ป้องกันข้อความซ้ำ |
| 3. Apache Kafka | มี Event log ถาวร (Event replay), throughput มหาศาล | ซับซ้อนและใช้ทรัพยากรสูงเกินความจำเป็นสำหรับทีมขนาด 4 คน; ทีมไม่มีประสบการณ์; การจัดการ delayed queue หรือ message-level rate limit ทำได้ยากกว่า RabbitMQ |

## Decision

เลือกใช้ **Option 2: RabbitMQ (Self-hosted บน Kubernetes)** เพื่อสร้างสถาปัตยกรรม Event-Driven แบบ Asynchronous สำหรับกระบวนการหลังการสั่งซื้อ:
1. เมื่อ `checkoutservice.PlaceOrder` ตัดเงินและบันทึกคำสั่งซื้อสำเร็จ จะทำการ publish ข้อความ `OrderPlacedEvent` (มี `order_id`, `user_id`, ข้อมูลอีเมล, รายการสินค้า) ไปยัง RabbitMQ Fanout Exchange ก่อนส่ง response สำเร็จกลับไปยังผู้ใช้ทันที
2. แยกคิวอิสระสำหรับผู้รับแต่ละราย:
   - **Queue `email-service`:** มี consumer ของ `emailservice` ดึงข้อความไปส่งหา MailFast โดยตั้งค่า consumer prefetch และ concurrency limiter ไม่ให้เกิน 50 requests/s
   - **Queue `loyalty-service`:** ส่งต่อให้ `loyaltyservice` ประมวลผลคำนวณแต้ม โดยใช้ `order_id` เป็น Idempotency Key ในฐานข้อมูลเพื่อรับประกันว่ามอบแต้มครั้งเดียวเท่านั้น (Exactly-once) และประมวลผลทันทีเพื่อให้เสร็จในเวลาไม่เกิน 1 นาที
   - **Queue `analytics-service`:** ส่งต่อให้ทีมการตลาด
3. สำหรับงานใหม่ในปีหน้า (SMS, Fraud check) สามารถผูกคิวใหม่เข้ากับ Exchange เดิมได้ทันทีโดยไม่ต้องแก้ไขโค้ดของ `checkoutservice`

## Consequences

- **Positive:** Checkout latency p95 คงเดิมที่ ~300 ms (บวกเวลา publish RabbitMQ ไม่เกิน 5-10 ms) ผ่าน SLA ≤ 500 ms อย่างง่ายดาย (Performance)
- **Positive:** ป้องกันระบบล่มจากการล้มเหลวหรือความล่าช้าของ MailFast (Availability)
- **Positive:** อัตราการส่งหา MailFast ถูกควบคุมให้อยู่ในขอบเขต 50/s ผ่าน Message Queue Buffer (Throughput / Rate limiting)
- **Positive:** สามารถเพิ่ม Subscriber ใหม่ได้โดยไม่กระทบ `checkoutservice` (Modifiability)
- **Negative:** เพิ่มภาระในการบริหารจัดการ RabbitMQ cluster บน Kubernetes (Operability & Cost)
- **Risks and mitigations:**
  - *ความเสี่ยง:* RabbitMQ ล่มทำให้ publish ไม่ผ่าน → *วิธีแก้ไข:* ทำ RabbitMQ Quorum Queues / Clustering บน Kubernetes และในโค้ด `checkoutservice` หาก publish ล้มเหลวให้ log error แต่ไม่ rollback เงินที่ตัดไปแล้ว พร้อมมี fallback write ลง local buffer
  - *ความเสี่ยง:* ข้อความถูก consume ซ้ำ (At-least-once delivery) → *วิธีแก้ไข:* ออกแบบให้ Consumer ทั้งหมดเป็น **Idempotent** โดยใช้ `order_id` ตรวจสอบสถานะก่อนให้แต้มสะสม

## Validation

1. **Load Test:** ใช้ `loadgenerator` จำลองโหลดพีก 200 orders/s เป็นเวลา 15 นาที ตรวจสอบว่า p95 latency ของ `PlaceOrder` ต่ำกว่า 500 ms
2. **Rate Limit Test:** ตรวจสอบว่าปริมาณการเรียก API ไปยัง MailFast ไม่เกิน 50 req/s ตลอดช่วงโหลดพีก
3. **Idempotency & SLA Test:** ทดสอบยิงข้อความ `OrderPlaced` ซ้ำ ตรวจสอบว่าแต้มสะสมถูกบันทึกเพียงครั้งเดียวและเกิดขึ้นภายใน 60 วินาที

## Open questions

*(ไม่มีข้อสงสัยค้างคา เนื่องจากตัวเลข SLA, ข้อจำกัดของ MailFast, และเงื่อนไขแต้มสะสมมีระบุครบถ้วนใน scenario แล้ว)*
