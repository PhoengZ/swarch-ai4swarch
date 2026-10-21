# ADR-0002: การจัดการกระบวนการหลังการสั่งซื้อสินค้า (Post-Order Processing)

- **Agent:** Antigravity (Google DeepMind)
- **Model:** Gemini 3.8 Flash (High)
- **Status:** Proposed
- **Date:** 2026-10-21
- **Deciders:** ทีมสถาปัตยกรรม Online Boutique

## Context and problem

ระบบ Online Boutique มีความต้องการทางธุรกิจใหม่ (R-12) สำหรับกระบวนการหลังจากที่มีการสั่งซื้อสินค้าสำเร็จ (Post-order processing) ดังนี้:
1. ส่งอีเมลยืนยันคำสั่งซื้อผ่าน MailFast ซึ่งเป็นผู้ให้บริการภายนอกรายใหม่แทนที่ logic จำลองใน `emailservice`
2. มอบแต้มสะสม (Loyalty points) ผ่านบริการใหม่ `loyaltyservice`
3. บันทึกเหตุการณ์ analytics "order placed" ให้กับทีมการตลาด
4. รองรับการขยายตัวในอนาคตสำหรับงานหลังการสั่งซื้อเพิ่มเติมในปีหน้า (เช่น SMS, Fraud scoring, Recommendations update)

ปัญหาคือระบบจำเป็นต้องตัดสินใจเลือกรูปแบบสถาปัตยกรรมในการประสานงานระหว่าง `checkoutservice` กับบริการปลายทางเหล่านี้ เพื่อให้ระบบทำงานได้อย่างน่าเชื่อถือและไม่ส่งผลกระทบต่อขั้นตอนการชำระเงินหลัก

## Decision drivers

1. **Availability:** การล่มของบริการเสริมหลังการสั่งซื้อ (อีเมล, แต้ม, analytics) ต้องไม่ทำให้การสั่งซื้อสินค้าของผู้ใช้ล้มเหลว
2. **Modifiability & Extensibility:** ต้องรองรับการเพิ่มผู้รับเหตุการณ์ใหม่ๆ ในอนาคตได้ง่าย (เช่น SMS, การประเมินการทุจริต)
3. **Performance:** ขั้นตอนหลังการสั่งซื้อต้องไม่ทำให้เวลาตอบสนอง (latency) ของกระบวนการสั่งซื้อช้าลงเกินควร
4. **Reliability:** ข้อมูลการสั่งซื้อต้องไม่สูญหาย และสามารถส่งต่อไปยังระบบภายนอกได้อย่างถูกต้อง

## Considered options

1. **Synchronous REST/gRPC Calls:** `checkoutservice` ทำการเรียกไปยัง MailFast (หรือ `emailservice`), `loyaltyservice`, และระบบ Analytics แบบ synchronous ทีละบริการก่อนตอบกลับผู้ใช้
2. **Asynchronous Message Queue / Event Bus:** `checkoutservice` เผยแพร่เหตุการณ์ `OrderPlaced` ไปยัง Message Broker (เช่น RabbitMQ หรือ Kafka) แล้วให้แต่ละบริการแยกกันรับข้อมูลไปประมวลผลแบบ asynchronous
3. **Background Goroutine / In-Memory Worker:** `checkoutservice` ทำงานใน background thread/goroutine ภายในตัวบริการเองเพื่อเรียกบริการภายนอกแบบ asynchronous

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| 1. Synchronous Calls | เรียบง่าย ไม่ต้องมีโครงสร้างพื้นฐานใหม่, ควบคุมลำดับขั้นตอนได้ง่าย | ผูกมัดแน่น (Tight coupling), latency รวมเพิ่มขึ้นตามจำนวนบริการ, หากระบบใดล่มอาจทำให้ checkout ล้มเหลวหรือช้ามาก |
| 2. Asynchronous Message Queue | แยกส่วนการทำงานชัดเจน (Loose coupling), latency ต่ำมาก, รองรับการขยายบริการใหม่ในอนาคตได้ทันที, มี retry ในตัว | ต้องติดตั้งและดูแลรักษา Message Broker เพิ่มเติม, สถาปัตยกรรมมีความซับซ้อนขึ้น |
| 3. In-Memory Background Worker | ไม่ต้องมี Message Broker ภายนอก, ปรับปรุง latency ได้ดีกว่าแบบ synchronous | เสี่ยงต่อข้อมูลสูญหายหาก container restart/crash ระหว่างประมวลผล, จัดการ retry และ rate limit ข้าม replica ได้ยาก |

## Decision

เลือกใช้ **Option 2: Asynchronous Message Queue / Event Bus** โดยให้ `checkoutservice` เผยแพร่เหตุการณ์ `OrderPlaced` เมื่อบันทึกคำสั่งซื้อสำเร็จ จากนั้นให้ Consumer แต่ละระบบแยกกันอ่านและประมวลผลงานของตนเอง

## Consequences

- **Positive:** แยกความรับผิดชอบและขยายระบบได้ง่ายมากเมื่อมีบริการใหม่ในปีหน้า (Modifiability)
- **Positive:** ป้องกันไม่ให้ความล่าช้าหรือความล้มเหลวของ MailFast ส่งผลต่อการชำระเงิน (Availability & Performance)
- **Negative:** เพิ่มภาระในการติดตั้งและบริหารจัดการ Message Broker บนระบบ (Operability & Cost)
- **Risks and mitigations:** ข้อมูลในคิวอาจค้างสะสมหากบริการปลายทางล่ม แก้ไขด้วยการตั้งค่า Dead-letter queue และระบบ Alert ติดตามขนาดคิว

## Validation

ทดสอบสั่งซื้อสินค้าและตรวจสอบว่าได้รับอีเมลยืนยัน แต้มสะสม และบันทึก analytics ครบถ้วน โดยวัด latency ของ checkout ว่าไม่เพิ่มขึ้นอย่างมีนัยสำคัญ

## Open questions

1. ปริมาณคำสั่งซื้อสูงสุด (Peak traffic) และ Latency SLA ของระบบ checkout อยู่ที่เท่าใด?
2. ข้อจำกัดของ MailFast (เช่น Rate limit, Latency, Error rate) เป็นอย่างไร?
3. กฎเกณฑ์เรื่องความถูกต้องของแต้มสะสม (เช่น Exactly-once delivery หรือ SLA เวลาในการได้แต้ม) มีกำหนดไว้อย่างไร?
4. ทักษะของทีมพัฒนาและมาตรฐานการดูแลระบบ (Kubernetes/Cloud) เอื้อต่อการใช้ Message Broker เทคโนโลยีใดมากที่สุด?
