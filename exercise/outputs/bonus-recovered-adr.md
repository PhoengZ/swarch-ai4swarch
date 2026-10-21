# ADR-0003: ใช้การสื่อสารแบบ Synchronous ผ่าน gRPC และใช้ demo.proto ร่วมกันไฟล์เดียว

- **Agent:** Antigravity (Gemini 3.8 Flash)
- **Status:** Reconstructed
- **Date:** 2026-10-21
- **Deciders:** ทีมพัฒนา Online Boutique ดั้งเดิม (กู้คืนจากซอร์สโค้ด)

## Context and problem

ระบบ Online Boutique เป็นแอปพลิเคชันต้นแบบ e-commerce แบบ microservices ที่เขียนด้วยภาษาที่หลากหลาย (Polyglot) ได้แก่ Go, Java, Python, C#, และ Node.js ในขั้นตอนการออกแบบการสื่อสารระหว่างบริการภายใน (Inter-service communication) ทีมพัฒนาต้องตัดสินใจเลือกโปรโตคอลการสื่อสารและวิธีการจัดการ API Contract:

1. บริการส่วนหน้า (`frontend`) ต้องเรียกใช้บริการย่อยถึง 7 บริการ (เช่น `productcatalogservice`, `cartservice`, `currency-service`) ในระหว่างการประมวลผลคำขอ HTTP ของผู้ใช้
2. บริการสั่งซื้อ (`checkoutservice`) ต้องเรียกใช้บริการชำระเงิน ตะกร้าสินค้า แปลงสกุลเงิน และส่งอีเมล แบบขั้นตอนต่อเนื่องเพื่อประมวลผลการสั่งซื้อ
3. สเปกของข้อความและ RPC endpoints ทั้งหมดของระบบถูกรวมศูนย์ไว้ในไฟล์เดียวคือ `protos/demo.proto`

ปัญหาที่ต้องตัดสินใจคือ: ควรเลือกโปรโตคอลการสื่อสารใดระหว่างบริการภายใน และควรจัดระเบียบ Interface Contract อย่างไรจึงจะรองรับระบบหลายภาษาได้อย่างมีประสิทธิภาพและดูแลรักษาง่ายที่สุดในฐานะระบบตัวอย่าง

## Decision drivers

1. **Polyglot Interoperability:** ต้องรองรับการทำงานร่วมกันของ microservices ที่เขียนด้วยภาษาโปรแกรมต่างกันได้อย่างราบรื่น
2. **Performance & Latency:** ต้องลด overhead ในการแปลงข้อมูล (serialization) และการเชื่อมต่อเครือข่ายระหว่าง microservices ที่เรียกกันเป็นลูกโซ่
3. **Simplicity & Maintainability for Demo:** โครงสร้างของสัญญาระบบ (API contract) ต้องเรียบง่าย ติดตามได้ง่าย และสะดวกต่อการ deploy ในฐานะ Cloud Native Demo
4. **Strong Typing:** การเรียกใช้งานข้ามบริการต้องมีการตรวจสอบความถูกต้องของชนิดข้อมูลและโครงสร้างข้อความตั้งแต่ขั้นตอน compile-time

## Considered options

1. **Synchronous gRPC with a Single Shared Protobuf (`demo.proto`):** ใช้ gRPC เป็นโปรโตคอลหลักทุกจุด และรวม RPC Definition ของทั้ง 9 บริการไว้ในไฟล์ `demo.proto` ไฟล์เดียว
2. **REST over HTTP/1.1 with JSON and OpenAPI/Swagger:** แต่ละบริการเปิด HTTP REST endpoints และส่งข้อมูลด้วย JSON โดยแยกไฟล์เอกสาร OpenAPI ประจำแต่ละบริการ
3. **Asynchronous Message-Driven Architecture (Event Bus):** สื่อสารผ่าน Message Broker (เช่น Kafka หรือ RabbitMQ) โดยกระจาย event เมื่อมีการเปลี่ยนแปลงสถานะ
4. **Multi-file Protobuf (Per-service `.proto` files):** ใช้ gRPC แต่แยกไฟล์ `.proto` ประจำแต่ละบริการ เช่น `cart.proto`, `checkout.proto` แล้ว import หากัน

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| 1. Synchronous gRPC + Shared `demo.proto` | Serialized แบบ Binary รวดเร็ว ประหยัดแบนด์วิธ; สร้างโค้ด client/server ได้หลายภาษาอัตโนมัติ; ไฟล์สัญญาอยู่ที่เดียว ง่ายต่อการ clone และ build ทั้งโปรเจกต์ | เกิด Tight coupling ระดับ schema ทุกบริการต้องอิงไฟล์เดียวกัน; การเรียก synchronous ยาวๆ เสี่ยงต่อ Cascading failure |
| 2. REST over HTTP/1.1 (JSON) | เรียบง่าย Debug ง่าย ดูผ่าน cURL/เบราว์เซอร์ได้ทันที; decoupling ระดับเอกสารแยกกัน | Latency สูงกว่าจาก text-based JSON และ HTTP/1.1 connection overhead; ไม่มี strong typing ข้ามภาษาแบบ native |
| 3. Asynchronous Event Bus | ความพร้อมใช้งานสูง (High Availability) และ Loose coupling; ไม่เกิด cascading latency | ซับซ้อนเกินไปสำหรับแอปพลิเคชันสาธิต; ต้องติดตั้งและดูแล Message Broker เพิ่ม; ไม่เหมาะกับ request ที่ต้องการ response ทันที เช่น หน้าเว็บดึงรายการสินค้า |
| 4. Multi-file Protobuf | แยกความเป็นเจ้าของของแต่ละ service ชัดเจน (High modularity) | จัดการ path การ import ข้ามโฟลเดอร์ซับซ้อนขึ้นในกระบวนการ build ของ Docker แต่ละภาษา |

## Decision

เลือก **Option 1: Synchronous gRPC with a Single Shared Protobuf (`demo.proto`)** สำหรับการสื่อสารระหว่างบริการทั้งหมดในระบบ Online Boutique:

1. ใช้ **gRPC (HTTP/2 + Protocol Buffers)** เป็นโปรโตคอลมาตรฐานในการเรียกกันระหว่างบริการภายในทั้งหมด เพื่อให้ได้ประสิทธิภาพสูง (Multiplexing, Binary payload) และสามารถ generate client stub ในภาษา Go, C#, Java, Python ได้จาก source เดียวกัน
2. รวมคำนิยาม RPC และ Message ทั้งหมด 9 บริการไว้ใน **`system/online-boutique/protos/demo.proto`** ภายใต้แพ็กเกจ `hipstershop` เพื่อความสะดวกสูงสุดในการพัฒนาและทดสอบระบบตัวอย่าง

## Consequences

- **Positive:** ประสิทธิภาพความเร็วในการสื่อสารสูง มี Latency ต่ำ และใช้แบนด์วิธเครือข่ายน้อย (Performance)
- **Positive:** ป้องกันข้อผิดพลาดในการส่งข้อมูลผิดประเภทด้วย Type Safety ตั้งแต่ขั้นตอน Build (Reliability)
- **Positive:** โครงสร้างโปรเจกต์เรียบง่าย นักพัฒนาสามารถดู API contract ทั้งหมดของระบบได้จากไฟล์เดียว (Modifiability for Demo purpose)
- **Negative:** เกิด Coupling ด้านการ deploy หากมีการแก้ `demo.proto` อาจส่งผลกระทบต่อบริการอื่นและต้อง re-generate โค้ดใหม่ทุกบริการ (Maintainability / Independence)
- **Negative:** การสื่อสารแบบ Synchronous chain (เช่น `frontend` → `checkoutservice` → `paymentservice`) ทำให้ความพร้อมใช้งานของระบบโดยรวมขึ้นอยู่กับความพร้อมของทุกบริการที่ต่อกัน (Availability / Fault tolerance)
- **Risks and mitigations:**
  - *ความเสี่ยง:* การแก้ proto แล้วกระทบข้ามทีม → *วิธีแก้ไข:* สำหรับโปรเจกต์เดโมจะใช้การล็อกเวอร์ชัน schema ไว้ที่ไฟล์เดียว แต่หากเป็น Production จริงควรแยกไฟล์ตาม domain

## Validation

1. สังเกตการเชื่อมต่อ gRPC ในโค้ดจริง: ใน `checkoutservice/main.go` มีการสร้าง gRPC client เชื่อมต่อไปยัง 4 บริการ และใน `frontend/main.go` มีการเชื่อมต่อ gRPC ไปยัง 7 บริการ
2. ตรวจสอบการ build: โค้ดของทุก service สามารถ generate stub จาก `demo.proto` ได้อย่างสมบูรณ์โดยไม่มี circular dependency

---

## Claim check

| # | Claim | Type | Evidence |
|---|---|---|---|
| 1 | ระบบ Online Boutique ประกอบด้วยบริการย่อยที่ใช้การสื่อสารด้วย gRPC ผ่าน TCP | **Evidenced** | `checkoutservice/main.go:120-123` (`mustConnGRPC`), `frontend/main.go:141-147` |
| 2 | บริการทั้งหมด 9 บริการนิยามอยู่ในไฟล์ `demo.proto` ไฟล์เดียว | **Evidenced** | `protos/demo.proto:23, 52, 60, 89, 107, 137, 150, 163, 196` |
| 3 | การเรียกใช้ระหว่างบริการเป็นแบบ Synchronous Request-Response | **Evidenced** | `checkoutservice/main.go:197-250` (`PlaceOrder` เรียก RPC ย่อยต่อกันตามลำดับ) |
| 4 | เหตุผลที่เลือก gRPC เพราะต้องการลด Latency และประหยัดแบนด์วิธกว่า REST/JSON | **Assumption** | เป็นคุณสมบัติทั่วไปตามมาตรฐานอุตสาหกรรมของ gRPC/Protobuf เมื่อเทียบกับ HTTP/1.1 JSON (ไม่มีคอมเมนต์ในโค้ด) |
| 5 | การรวมทุกบริการไว้ใน `demo.proto` ไฟล์เดียวทำเพื่อความสะดวกในการสาธิตและ build โปรเจกต์ตัวอย่าง | **Inferred** | อนุมานจากชื่อไฟล์ `demo.proto`, ข้อความลิขสิทธิ์ของ Google Cloud Platform และบริบทของ repository ต้นทาง |
| 6 | ทีมพัฒนาพิจารณาเลือก Event-driven แล้วแต่ปฏิเสธเพราะซับซ้อนเกินไป | **Inferred** | อนุมานตามหลักวิศวกรรมซอฟต์แวร์ เนื่องจากไม่มีร่องรอยการเชื่อมต่อ Message Broker ในโค้ดทั้งสองบริการ |
| 7 | การเรียก Synchronous chain ทำให้เสี่ยงต่อ Cascading failure | **Assumption** | เป็นทฤษฎีและข้อเท็จจริงพื้นฐานด้าน Reliability ของ Microservices เมื่อไม่มี Circuit Breaker |
