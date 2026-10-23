Agent: Claude Code — Model: Claude Opus 5.5 (`claude-opus-5-5`)

# ADR-0002: Publish an "order placed" event to a message broker for post-order actions

- **Status:** Proposed
- **Date:** 2026-10-23
- **Deciders:** ยังไม่ระบุ (ดู Open questions)

## Context and problem

หลังจากลูกค้าสั่งซื้อสำเร็จ Online Boutique ต้องทำงานต่อเนื่องเพิ่มเติม (post-order actions) สามอย่าง:

| # | งาน | ระบบที่เกี่ยวข้อง | หมายเหตุ |
|---|---|---|---|
| 1 | ส่งอีเมลยืนยันคำสั่งซื้อ | **MailFast** (external email provider รายใหม่) | แทนที่ logic จำลองใน `emailservice` |
| 2 | ให้ loyalty points | `loyaltyservice` (ต้องสร้างใหม่) | |
| 3 | บันทึก analytics event "order placed" | ระบบของทีม marketing | ปลายทาง/รูปแบบยังไม่ระบุ |

- ทีม marketing คาดว่าจะมี post-order actions **เพิ่มขึ้นในปีหน้า** (SMS, fraud scoring, การอัปเดต recommendations) ดังนั้นจำนวน consumer ของเหตุการณ์ "สั่งซื้อแล้ว" จะโตขึ้นเรื่อย ๆ
- MailFast เป็นระบบภายนอก จึงควบคุม latency และ availability ของมันไม่ได้
- ปัญหาที่ต้องตัดสินใจ: **บริการที่รับคำสั่งซื้อควรสั่งงาน post-order actions อย่างไร** — เรียกแต่ละ service ตรง ๆ แบบ synchronous หรือประกาศเหตุการณ์ให้ service อื่นมารับไปทำเอง
- ข้อจำกัดของ ADR นี้: เขียนจาก requirement อย่างเดียว ไม่ได้อ่านโค้ดของระบบ ชื่อบริการที่รับคำสั่งซื้อ (เช่น `checkoutservice`), protocol ที่ใช้อยู่ (เช่น gRPC) และ infrastructure ที่มีอยู่ **Not evidenced in the provided files** — การอ้างถึงสิ่งเหล่านี้ด้านล่างเป็น **[Inferred]**

## Decision drivers

1. **Modifiability / extensibility** — ต้องเพิ่ม post-order action ใหม่ (SMS, fraud scoring, recommendations) ได้โดยไม่ต้องแก้บริการที่รับคำสั่งซื้อทุกครั้ง
2. **Availability ของ checkout** — การสั่งซื้อต้องไม่ล้มเหลวเพราะ MailFast, `loyaltyservice` หรือระบบ analytics ล่ม
3. **Performance (checkout latency)** — ผู้ใช้ไม่ควรรอให้ post-order actions ทั้งหมดเสร็จก่อนเห็นหน้ายืนยัน
4. **Reliability ของแต่ละ action** — ห้ามทำ event หาย (เช่น ลูกค้าไม่ได้แต้มหรือไม่ได้อีเมล) และไม่ควรให้แต้มซ้ำ
5. **Cost และ operational complexity** — จำนวน component ใหม่ที่ทีมต้องดูแล

## Considered options

1. **Synchronous fan-out** — บริการที่รับคำสั่งซื้อเรียก MailFast, `loyaltyservice` และ analytics ตรง ๆ ทีละตัว (เช่น ผ่าน gRPC/HTTPS) ก่อนตอบกลับผู้ใช้
2. **Event-driven publish/subscribe ผ่าน message broker** — บริการที่รับคำสั่งซื้อ publish event `OrderPlaced` หนึ่งครั้งไปยัง message broker (เช่น RabbitMQ, Kafka หรือ Google Cloud Pub/Sub — ยังไม่เลือก) แต่ละ consumer (`emailservice` → MailFast, `loyaltyservice`, analytics) subscribe เอง
3. **Orchestrator แบบ asynchronous (post-order workflow service)** — บริการที่รับคำสั่งซื้อส่งคำสั่งไปยัง service ใหม่หนึ่งตัว (`postorderservice`) ซึ่งเป็นผู้เรียกแต่ละ action ตามลำดับ พร้อม retry

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| 1. Synchronous fan-out | ง่ายที่สุด ไม่มี infrastructure ใหม่; รู้ผลทันทีว่าแต่ละ action สำเร็จหรือไม่; debug ง่าย (trace เดียว) | checkout latency = ผลรวมของทุก call รวม MailFast (ระบบภายนอก); ถ้า action ใดล่ม checkout ล่มหรือต้องเขียน error handling เอง; บริการรับคำสั่งซื้อ coupled กับทุก consumer — เพิ่ม SMS/fraud/recommendations ต้องแก้และ deploy บริการนี้ทุกครั้ง |
| 2. Pub/sub ผ่าน message broker | เพิ่ม consumer ใหม่ได้โดยไม่แตะบริการรับคำสั่งซื้อ (modifiability); checkout ไม่ขึ้นกับ availability ของ MailFast/loyalty/analytics; latency ของ checkout เพิ่มแค่การ publish หนึ่งครั้ง; broker buffer และ retry ได้เมื่อ consumer ล่ม; แต่ละ consumer scale แยกกัน | ต้องดูแล message broker ใหม่ (cost, ops); eventual consistency — อีเมล/แต้มมาถึงช้ากว่า; ต้องจัดการ duplicate delivery (idempotency) และ dead-letter queue; ถ้า publish หลังบันทึกคำสั่งซื้อแล้วล้มเหลว event อาจหาย (dual-write problem); trace ข้าม service ยากขึ้น; schema ของ event กลายเป็น contract ที่ต้อง version |
| 3. Async orchestrator | checkout ไม่ต้องรอ post-order actions; มีจุดเดียวที่เห็นสถานะของทุก action และคุม retry/ลำดับได้; ไม่จำเป็นต้องมี broker เต็มรูปแบบ | orchestrator ยังต้องรู้จักทุก action — เพิ่ม action ใหม่ต้องแก้ orchestrator (ย้าย coupling มาไว้ที่เดียว ไม่ได้ลบออก); orchestrator เป็น single point of failure ใหม่; ต้องมีที่เก็บสถานะ workflow เพื่อให้ retry ได้หลัง restart |

## Decision

เลือก **Option 2: Event-driven publish/subscribe ผ่าน message broker**

- บริการที่รับคำสั่งซื้อ publish event `OrderPlaced` (order ID, ข้อมูลลูกค้า, รายการสินค้า, ยอดรวม, timestamp) **หนึ่งครั้ง** หลังคำสั่งซื้อสำเร็จ แล้วตอบผู้ใช้ทันที
- Consumers ที่ subscribe:
  - `emailservice` — เปลี่ยนจาก logic จำลองเป็นเรียก MailFast API
  - `loyaltyservice` (ใหม่) — คำนวณและบันทึกแต้ม; ใช้ order ID เป็น idempotency key
  - analytics consumer — ส่ง event "order placed" ให้ระบบของ marketing
- เหตุผล: driver อันดับ 1 (modifiability) สำคัญที่สุดเพราะรู้แน่ว่าจะมี action เพิ่มอีกอย่างน้อยสามอย่างในปีหน้า; Option 2 เป็นตัวเลือกเดียวที่เพิ่ม action ได้โดยไม่แก้ทั้งบริการรับคำสั่งซื้อและ orchestrator และยังตอบ driver 2–3 ด้วยการตัด MailFast ออกจาก critical path ของ checkout
- เลือก broker ตัวไหน (RabbitMQ / Kafka / managed Pub/Sub) ยังไม่ตัดสินใน ADR นี้ — ขึ้นกับ Open questions เรื่อง volume, ทักษะทีม และ hosting

## Consequences

- **Positive:** เพิ่ม SMS, fraud scoring, recommendations ได้ด้วยการเพิ่ม subscriber ใหม่ ไม่ต้องแก้หรือ redeploy บริการรับคำสั่งซื้อ (quality attribute: modifiability)
- **Positive:** การสั่งซื้อยังสำเร็จได้แม้ MailFast, `loyaltyservice` หรือ analytics ล่ม; event รออยู่ใน broker จนกว่า consumer จะกลับมา (quality attribute: availability)
- **Positive:** checkout latency ไม่รวมเวลาเรียก MailFast และ action อื่น (quality attribute: performance)
- **Positive:** แต่ละ consumer scale และ deploy แยกกันได้ (quality attribute: scalability, deployability)
- **Negative:** มี message broker ที่ต้อง deploy, monitor และจ่ายค่าใช้จ่าย (quality attribute: cost, operability)
- **Negative:** อีเมลยืนยันและแต้มมาถึงแบบ eventual ไม่ใช่ทันที; หน้า confirmation ไม่สามารถยืนยันได้ว่าส่งอีเมลแล้ว (quality attribute: consistency / usability)
- **Negative:** debug และ trace ข้าม service ยากขึ้น ต้องส่ง correlation ID (order ID) ไปกับ event (quality attribute: observability)
- **Risks and mitigations:**
  - *Event หายระหว่างบันทึกคำสั่งซื้อกับ publish (dual write)* → ใช้ transactional outbox หรืออย่างน้อย retry การ publish พร้อม alert; ต้องดูก่อนว่าบริการรับคำสั่งซื้อมี datastore หรือไม่ (Open question)
  - *Duplicate delivery ทำให้ให้แต้มซ้ำหรือส่งอีเมลซ้ำ* → consumer ทุกตัวต้อง idempotent โดยใช้ order ID
  - *MailFast ล่มหรือ rate-limit* → retry แบบ exponential backoff + dead-letter queue + alert
  - *Event schema เปลี่ยนแล้ว consumer พัง* → กำหนด schema ที่มี version (เช่น protobuf/JSON schema) และเปลี่ยนแบบ backward-compatible เท่านั้น
  - *Broker กลายเป็น single point of failure ของ post-order actions* → ใช้ broker แบบ replicated หรือ managed service; ถ้า publish ไม่ได้ checkout ยังสำเร็จแต่ต้องเก็บ event ไว้ส่งทีหลัง (outbox)

## Validation

- **Load/latency test:** วัด checkout latency ก่อนและหลังเปลี่ยน — publish event ต้องไม่ทำให้ latency เพิ่มเกินเป้า (ตัวเลขเป้า: ดู Open questions)
- **Fault injection:** ปิด MailFast (หรือ mock ให้ timeout), `loyaltyservice` และ analytics consumer ทีละตัว แล้วยืนยันว่า checkout ยังสำเร็จ และ event ถูกประมวลผลครบเมื่อ consumer กลับมา
- **Idempotency test:** ส่ง event `OrderPlaced` ซ้ำ แล้วยืนยันว่าได้แต้มครั้งเดียวและอีเมลฉบับเดียว
- **Extensibility check:** เพิ่ม consumer ทดลอง (เช่น SMS stub) โดยไม่แก้โค้ดบริการรับคำสั่งซื้อ
- **Metrics ใน production:** consumer lag, จำนวนข้อความใน dead-letter queue, อัตราสำเร็จของการส่งผ่าน MailFast, เวลาจาก order ถึงอีเมล
- **Review date:** ทบทวน ADR นี้ก่อนเริ่มทำ post-order action ตัวแรกของปีหน้า (วันที่แน่นอน: ดู Open questions)

## Open questions

- ใครคือ Deciders ของ ADR นี้?
- บริการใดเป็นผู้รับคำสั่งซื้อ (เช่น `checkoutservice`) และสื่อสารด้วย protocol อะไร? — Not evidenced in the provided files
- ปริมาณคำสั่งซื้อ (orders/second, peak) เท่าไร? ใช้เลือก broker และขนาด
- เป้า latency ของ checkout และเป้า availability (SLO) เป็นเท่าไร?
- อีเมลยืนยันต้องถึงลูกค้าภายในกี่วินาที/นาที? แต้มต้องเห็นทันทีหลังสั่งซื้อหรือไม่?
- MailFast มี SLA, rate limit, API (REST/SMTP), การรองรับ idempotency key และค่าใช้จ่ายต่อข้อความอย่างไร?
- ระบบ analytics ของ marketing คืออะไร รับ event ในรูปแบบใด และยอมรับการสูญหายของ event ได้หรือไม่?
- กฎการคำนวณ loyalty points และ `loyaltyservice` ต้องมี datastore แบบไหน?
- บริการรับคำสั่งซื้อมี datastore ที่รองรับ transactional outbox หรือไม่?
- ระบบ deploy อยู่ที่ไหน (Kubernetes / cloud ใด) มี message broker หรือ managed Pub/Sub ให้ใช้อยู่แล้วหรือไม่?
- ทีมมีประสบการณ์กับ RabbitMQ, Kafka หรือ Pub/Sub แค่ไหน และมีงบสำหรับ infrastructure ใหม่เท่าไร?
- มีข้อกำหนดด้าน privacy (PDPA/GDPR) ในการส่งข้อมูลลูกค้าไปยัง MailFast และระบบ analytics หรือไม่?
