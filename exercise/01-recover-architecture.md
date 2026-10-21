**ภาษา:** ภาษาไทย | [English](01-recover-architecture.en.md)

# งานที่ 1: กู้คืนสถาปัตยกรรม (ประมาณ 15 นาที)

> **เชื่อมโยงกับการบรรยาย:** T7 การกู้คืนสถาปัตยกรรม, AICH2 (การตามรอย), AICH3 (บริบท) และข้อค้นพบจาก JSS ว่า 93% ของงานวิจัยไม่เคยตรวจสอบผลลัพธ์จาก AI
> **คำถาม:** AI กู้คืนสถาปัตยกรรมได้มากแค่ไหน และขึ้นอยู่กับหลักฐานที่ AI ได้อ่านมากเพียงใด

**ระหว่างทำงานนี้ ห้ามเปิด repository ต้นฉบับของ Online Boutique** เพราะ README ของต้นฉบับมีแผนภาพสถาปัตยกรรมที่นิสิตกำลังกู้คืนอยู่

ในแต่ละ run ให้เริ่ม **แชตหรือ session ใหม่** แล้วส่งข้อความด้านล่างให้ agent โดย agent จะทำตาม [`prompts/1-recover-architecture.md`](../prompts/1-recover-architecture.md) และเขียนผลลัพธ์ไว้ที่ `exercise/outputs/1<run>-architecture.md`

**Run A: เฉพาะ interface**
```
Follow prompts/1-recover-architecture.md
run: A
evidence: system/online-boutique/protos/demo.proto
```

**Run B: interface + ไฟล์ตั้งค่าการ deploy**
```
Follow prompts/1-recover-architecture.md
run: B
evidence: system/online-boutique/protos/demo.proto, system/online-boutique/kubernetes-manifests/
```

ระหว่างที่ agent ทำงาน ให้สังเกตว่า agent เปิดไฟล์ใดบ้าง หาก agent อ่านไฟล์นอกเหนือจากหลักฐานที่อนุญาต ให้บันทึกไว้ใน prompt log

## ผลลัพธ์

- Run A: [`outputs/1A-architecture.md`](outputs/1A-architecture.md)
- Run B: [`outputs/1B-architecture.md`](outputs/1B-architecture.md)

เปิดแต่ละไฟล์ด้วย **Markdown: Open Preview** (`Ctrl+Shift+V` / `⇧⌘V`) เพื่อดูแผนภาพ

## ตรวจสอบ Run B ด้วยตัวเอง (ส่วนที่สำคัญที่สุด)

ตรวจสอบ dependency ในตารางของ Run B **อย่างน้อยห้ารายการ** เทียบกับไฟล์จริง โดยเปิดไฟล์ที่ AI อ้างถึงและหาหลักฐาน นิสิตสามารถตรวจซอร์สโค้ดของ service ได้ด้วย เช่น `src/checkoutservice/main.go`

| ผู้เรียก → ผู้ถูกเรียก | หลักฐานที่ AI อ้าง | ยืนยันแล้วหรือไม่ ✅ / ❌ / ❓ | สิ่งที่นิสิตพบ |
|---|---|---|---|
| frontend → cartservice | kubernetes-manifests/frontend.yaml:73-74 | ✅ | ไปเช็คใน frontend.yaml มี env CART_SERVICE_ADDR ชี้ไปที่ cartservice:7070 ตรงกับพอร์ต gRPC ของ cartservice จริง |
| checkoutservice → paymentservice | kubernetes-manifests/checkoutservice.yaml:61-62 | ✅ | ใน checkoutservice.yaml กำหนด env PAYMENT_SERVICE_ADDR ตรงกับพอร์ต 50051 ของ paymentservice พอดี |
| checkoutservice → emailservice | kubernetes-manifests/checkoutservice.yaml:63-64 | ✅ | ใน checkoutservice.yaml มี env ชี้ไปที่ emailservice:5000 เช็คแล้วตรงกับพอร์ต service ใน emailservice.yaml |
| recommendationservice → productcatalogservice | kubernetes-manifests/recommendationservice.yaml:60-61 | ✅ | ใน recommendationservice.yaml ชี้ไปที่ productcatalogservice:3550 จริง เอาไว้ดึงรายการสินค้ามาคำนวณตัวแนะนำ |
| cartservice → redis-cart | kubernetes-manifests/cartservice.yaml:50-51 | ✅ | ใน cartservice.yaml มี env REDIS_ADDR ชี้ไปที่ redis-cart:6379 ซึ่งมีตัว deployment redis-cart อยู่ในไฟล์เดียวกันเลย |

- **สิ่งที่ผิดหรือขาดหายไป** ใน Run B (dependency ที่ไม่มีจริง, dependency ที่ขาดไป, container ที่น่าสงสัย):
ตอบ
  1. ใน frontend.yaml มี env เขียนว่า SHOPPING_ASSISTANT_SERVICE_ADDR แต่พอไปหาไฟล์ deploy จริงๆ กลับไม่มี แถมใน kustomization.yaml ก็คอมเมนต์ปิดไว้ สรุปคือตัวนี้ไม่ได้รันจริงแต่ AI ดึงมาใส่
  2. ดูไม่ออกว่าแต่ละ service เขียนด้วยภาษาอะไรกันแน่ (เช่น Go, Java, C#) เพราะในไฟล์ manifest มันบอกแค่ชื่อ docker image
  3. ไม่เห็นการต่อพวก third-party จริงๆ เช่นระบบจ่ายเงินหรือส่งเมลข้างนอก เพราะในโค้ดจำลองเป็น mock เอาไว้ข้างในหมด

## สะท้อนคิด (ข้อละ 2–3 ประโยค)

1. เปรียบเทียบ A กับ B: agent **ไม่สามารถ**รู้อะไรได้จากไฟล์ `.proto` เพียงไฟล์เดียว และเพราะเหตุใด
   ตอบ เพราะ ไฟล์ proto มันมีแค่สเปกของ API ฝั่ง server กับโครงสร้างข้อมูลแค่นั้น มันเลยไม่บอกว่าใครเป็นคนยิงเรียกใคร แล้วก็ไม่มีทางรู้เลยว่ามีตัวเว็บ Frontend หรือตัวเก็บข้อมูลอย่าง Redis อยู่ด้วย จนกว่าจะได้ไปดูพวกไฟล์ Kubernetes manifest

2. ข้อผิดพลาด**ข้อใดข้อหนึ่ง**ที่จะทำให้สมาชิกใหม่ของทีมเข้าใจผิดมากที่สุด หากเผยแพร่แผนภาพนี้โดยไม่ตรวจสอบ
   ตอบ ตัว shoppingassistantservice เพราะถ้าคนใหม่มาดูไดอะแกรมแล้วไม่เช็คโค้ดก่อน คงนึกว่าระบบมีบอทช่วยช้อปปิ้งทำงานอยู่จริงๆ แต่ความจริงโดนปิดไว้แล้วไม่ได้ deploy ด้วยซ้ำ
