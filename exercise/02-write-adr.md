**ภาษา:** ภาษาไทย | [English](02-write-adr.en.md)

# งานที่ 2: เขียน ADR สำหรับความต้องการใหม่ (ประมาณ 20 นาที)

> **เชื่อมโยงกับการบรรยาย:** T3 (GPT-4 แบบ zero-shot ให้ผลที่สอดคล้องกัน แต่ _ครบถ้วนน้อยกว่ามนุษย์_ ส่วน few-shot ช่วยลดช่องว่างได้มาก), AICH3 (บริบท), สัปดาห์ที่ 10 (message broker)
> **คำถาม:** ADR ที่ AI เขียนดีขึ้นแค่ไหน เมื่อให้ตัวอย่างและบริบทจริงแก่ AI

อ่าน [สถานการณ์ (ภาษาไทย)](../docs/scenario.th.md) ก่อน: ความต้องการ **R-12** ว่าด้วยสิ่งที่เกิดขึ้นหลังจากลูกค้าสั่งซื้อสินค้า

ในแต่ละ run ให้เริ่ม **แชตหรือ session ใหม่** แล้วส่งข้อความด้านล่าง โดยแทนที่ `<paste R-12 here>` ด้วยข้อความ R-12 ภาษาอังกฤษที่อยู่ในกรอบอ้างอิงของ scenario

**Run a: ไม่มีตัวอย่าง (zero-shot)** → [`outputs/2a-adr.md`](outputs/2a-adr.md)

```
Follow prompts/2-write-adr.md
run: a
context: none
requirement:
After an order is placed, Online Boutique must:
1. send the order confirmation email through **MailFast**, a new external email provider, which replaces the mock `emailservice` logic;
2. award **loyalty points** through a new `loyaltyservice` (to be built);
3. record an **"order placed" analytics event** for the marketing team.
Marketing expects **more post-order actions next year** (SMS, fraud scoring, a recommendations update).
language: Thai
```

**Run b: มีตัวอย่าง + บริบท** → [`outputs/2b-adr.md`](outputs/2b-adr.md)

```
Follow prompts/2-write-adr.md
run: b
context: docs/scenario.md, docs/adr/0001-use-redis-for-cart-storage.md, system/online-boutique/src/checkoutservice/main.go
requirement:
After an order is placed, Online Boutique must:
1. send the order confirmation email through **MailFast**, a new external email provider, which replaces the mock `emailservice` logic;
2. award **loyalty points** through a new `loyaltyservice` (to be built);
3. record an **"order placed" analytics event** for the marketing team.

Marketing expects **more post-order actions next year** (SMS, fraud scoring, a recommendations update).
language: Thai
```

> บรรทัด `language: Thai` ทำให้ agent เขียน ADR เป็นภาษาไทย (หัวข้อของ template และชื่อทางเทคนิคยังคงเป็นภาษาอังกฤษ) หากต้องการภาษาอังกฤษให้ลบบรรทัดนี้ **ใช้ภาษาเดียวกันทั้ง Run a และ Run b** เพื่อให้เปรียบเทียบกันได้

> ใน Run a ให้ตรวจว่า agent ไม่ได้อ่านไฟล์อื่นนอกจาก template จริง ๆ หาก agent เปิด `docs/scenario.md` ให้บันทึกไว้ใน prompt log เพราะจะทำให้การเปรียบเทียบเปลี่ยนไป

## เปรียบเทียบ a กับ b

| เกณฑ์                                                                              | Run a                                                                                                         | Run b                                                                                                                                                                  |
| ---------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| ใช้ตัวเลขจาก scenario หรือไม่ (latency, ช่วงพีก, ข้อจำกัดของ MailFast)             | ไม่ใช้เลย ย้ายไปถามใน Open questions ทั้งหมด (ไม่แต่งตัวเลขเอง)                                               | ใช้ครบ (500 ms, 200/s, 1.5 s/8 s, 0.5%, 50/s) และคำนวณ backlog 150 ฉบับ/วินาทีช่วงพีก                                                                                  |
| ทางเลือกที่พิจารณา (ระบุ)                                                          | 3 ทาง: sync fan-out / pub-sub ผ่าน broker (เลือก แต่ไม่ระบุ broker) / async orchestrator                      | 4 ทาง: sync gRPC / async ใน process / RabbitMQ (เลือก) / Kafka                                                                                                         |
| จัดการแต้มสะสมแบบ "ครั้งเดียวเท่านั้น" และ rate limit ของ MailFast หรือไม่ อย่างไร | แต้ม: idempotent ด้วย order ID แต่ไม่พูดถึง "ภายใน 1 นาที"; rate limit: แค่ retry + backoff + DLQ ไม่มีตัวเลข | แต้ม: at-least-once + dedup ด้วย unique `order_id` + ack หลังบันทึก, replica >= 2 และ alert เพื่อให้ทัน 1 นาที rate limit: consumer จำกัด ≤ 50/s และ queue เป็น buffer |
| มีสิ่งที่แต่งขึ้นเองหรือไม่ (ตัวเลข เครื่องมือ หรือข้อเท็จจริงที่ไม่มีใน scenario) | แทบไม่มี ติด [Inferred] ไว้ชัด เสนอ Google Cloud Pub/Sub ซึ่งขัดนโยบาย self-hosted (แต่ไม่ได้เห็น scenario)   | บอกว่า "ไม่มีใครในทีมเคยใช้ Kafka" ซึ่ง scenario ไม่ได้บอก ตัวเลข replica >= 2, alert 30 s และ analytics เขียนด้วย Go ไม่มีที่มา                                       |

## รับผิดชอบ ADR ฉบับสุดท้ายของตัวเอง

นิสิตคือสถาปนิก ให้คัดลอก `outputs/2b-adr.md` ไปที่ `docs/adr/0002-<short-title>.md` แล้วแก้ไข**ด้วยตัวเอง**จนพร้อมจะลงชื่อรับรอง ห้ามให้ agent ทำส่วนนี้

- แก้ไขสิ่งที่ผิดหรือแต่งขึ้นเอง และเพิ่มสิ่งที่ขาดหายไป
- ตัดสินใจขั้นสุดท้ายด้วยตัวเอง นิสิตสามารถไม่เห็นด้วยกับ AI ได้

นิสิตจะต้องปกป้อง ADR นี้ต่อหน้าคณะกรรมการรีวิวในส่วน B จึงต้องอธิบายทุกการตัดสินใจใน ADR ได้

| #   | สิ่งที่เปลี่ยนจากร่างของ AI                                                                                                   | เหตุผล                                                                                                                                                                                       |
| --- | ----------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | แก้ข้อความใน Pros/cons ของ Kafka จาก "ไม่มีใครในทีมเคยใช้" เป็น "มี 2 คนเคยใช้ RabbitMQ; ประสบการณ์ Kafka ไม่ระบุใน scenario" | ข้อความเดิมไม่เป็นข้อเท็จจริง scenario บอกเพียงว่ามี 2 คนเคยใช้ RabbitMQ ไม่ได้บอกเรื่อง Kafka จึงเขียนเท่าที่มีหลักฐาน                                                                      |
| 2   | คง `loyaltyservice` ไว้ที่อย่างน้อย 2 replicas และเขียนเหตุผลกำกับ จากเดิมที่ AI ใส่ตัวเลขไว้โดยไม่มีที่มา                    | scenario กำหนดให้ได้แต้มภายใน 1 นาที ถ้ามี replica เดียวแล้วล่มนานเกิน 1 นาที แต้มจะเข้าไม่ทัน การมีอย่างน้อย 2 replicas ทำให้อีกตัวรับงานต่อได้ทันที                                        |
| 3   | เพิ่ม transactional outbox ใน `checkoutservice` แทนที่จะปล่อยเป็น Open question พร้อมเพิ่ม data store สำหรับตาราง outbox      | ป้องกัน event หายเมื่อ publish ไป RabbitMQ ไม่สำเร็จ (order สำเร็จแต่ไม่มีอีเมลหรือแต้ม) แลกกับการมี data store ใหม่ที่ทีมต้องดูแล และต้องตรวจว่า latency ของ `PlaceOrder` ยังไม่เกิน 500 ms |
