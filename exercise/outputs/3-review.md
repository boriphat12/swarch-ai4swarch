> Reviewer: Claude Code (agent) · model Claude Opus 5.5 (`claude-opus-5-5`)

# ATAM review: ADR-0002 Publish `OrderPlaced` to RabbitMQ

ไฟล์ที่ใช้: `docs/adr/0002-publish-order-placed-to-rabbitmq.md`, `docs/scenario.md` เท่านั้น (ไม่ได้เปิด `main.go` จึงตรวจเลขบรรทัดที่ ADR อ้างไม่ได้)

## 1. Quality-attribute scenarios

| # | Source | Stimulus | Environment | Response | Response measure |
|---|---|---|---|---|---|
| QAS-1 Performance | ลูกค้าช่วง 11.11 | 200 `PlaceOrder`/s | peak; MailFast p95 1.5 s, p99 8 s | order ยืนยันและ event ถูกบันทึกแบบ durable ก่อนตอบลูกค้า | p95 `PlaceOrder` ≤ 500 ms (ปัจจุบันประมาณ 300 ms) |
| QAS-2 Availability | infrastructure | `loyaltyservice`, email consumer หรือ RabbitMQ ล่ม 10 นาที | peak | order ยังสำเร็จ event รอใน outbox/queue แล้วถูกส่งต่อเมื่อกลับมา | order ล้ม 0 รายการ; event หาย 0 รายการ |
| QAS-3 Correctness | RabbitMQ | redelivery หลัง pod ของ `loyaltyservice` ถูก kill ระหว่างประมวลผล | peak | award points หนึ่งครั้งต่อ `order_id` | 100 % ของ order ได้ points ภายใน 60 s; ซ้ำ 0 รายการ |

## 2. Findings

| Type | Finding | Why it matters |
|---|---|---|
| Risk | Decision บอกว่า "ถ้าเขียน outbox ไม่สำเร็จ ให้ log แต่ไม่ทำให้ order ล้ม" | เป็นเส้นทางที่ event หายแบบเงียบๆ ลูกค้าจ่ายเงินแล้วแต่ไม่ได้ points จึงละเมิด QAS-3 ("exactly once" แปลว่าห้ามขาดด้วย ไม่ใช่แค่ห้ามซ้ำ) และไม่มี reconcile job มาเก็บตก |
| Risk | ยังไม่ได้เลือก outbox store และ `checkoutservice` ไม่มี DB ของ order ที่จะเขียนใน transaction เดียวกัน | ประโยชน์หลักของ transactional outbox (atomic กับ state ของ order) จึงไม่มีจริง เหลือแค่ local buffer ที่เพิ่ม stateful component ตัวที่สอง (นอกจาก RabbitMQ) ให้ทีม 4 คนดูแล |
| Risk | email consumer limit "≤ 50 emails/s" แต่ไม่ได้บอกว่าเป็น limit **รวมทุก replica** | ถ้า scale เป็น 2 replicas แต่ละตัวยิง 50/s จะชน rate limit ของ MailFast และ retry ที่เกิดขึ้นก็กิน quota เดียวกัน |
| Sensitivity point | Concurrency/prefetch ของ email consumer | ถ้าจะได้ 50/s ขณะ latency p95 1.5 s ต้องมี in-flight ประมาณ 75 call (Little's law) และมากกว่านั้นในช่วง tail p99 8 s ถ้าตั้ง prefetch ต่ำ throughput จริงจะต่ำกว่า 50/s มาก ทำให้ backlog โตกว่าที่ ADR คำนวณไว้ |
| Sensitivity point | Polling interval ของ outbox relay + backoff ของ consumer + `queue age > 30 s` alert | ทั้งสามค่าใช้ budget 60 s ของ loyalty ร่วมกัน relay ที่ poll ทุก 5 s บวก backoff 3 รอบก็กิน budget ไปเกินครึ่งแล้ว |
| Trade-off point | เขียน outbox แบบ synchronous ใน `PlaceOrder` | durability (+) กับ latency (−): ADR ยอมรับเองว่าเพิ่ม latency แต่ไม่มีตัวเลขในขณะที่ headroom เหลือแค่ประมาณ 200 ms |
| Trade-off point | ตอบลูกค้าก่อนทำ post-order action | availability/latency (+) กับ consistency (−): อีเมลมาช้า และ fraud scoring ทำได้หลังยืนยัน order เท่านั้น |
| Non-risk | RabbitMQ แทน Kafka | 200 msg/s ต่ำมาก มี 2 devs เคยใช้ และ per-queue DLQ/retry ตรงกับความต้องการของ email ส่วน replay ถูกบันทึกเป็น open question แล้ว |
| Non-risk | dedup ด้วย unique constraint บน `order_id` + ack หลัง commit | ทำให้ได้ effectively-once ตาม QAS-3 โดยมีเงื่อนไขว่าต้อง insert `order_id` และเพิ่ม points ใน transaction เดียวกัน (ADR ควรเขียนให้ชัด) |

## 3. Missing or weak

- **Option ที่ยังไม่ได้พิจารณา:** (a) publish ตรงไป RabbitMQ ด้วย publisher confirms โดยไม่มี outbox และกำหนด loss budget ให้ชัด ซึ่ง Risks ก็พูดถึงทั้งสองแบบปนกัน ควรแยกเป็น option ให้ชัด (b) NATS JetStream เป็นทางเลือก open-source ที่เบากว่าและยังทำ replay ได้
- **ตัวเลขที่ลดคุณภาพ:** "ระบายภายใน 3T วินาที" สมมติว่าหลัง peak ไม่มี order เข้ามาใหม่เลย ถ้ายังมี order เข้ามา r/s เวลาระบายจะเป็น 150T/(50−r) และถ้า r ≥ 50 จะระบายไม่หมด
- **ผลกระทบที่ไม่ได้ระบุ:** analytics เป็น at-least-once จึงนับ order ซ้ำได้ แต่ไม่มีการกำหนดว่าจะ dedup ที่ไหน; ยังไม่มี contract/schema ของ `OrderPlaced` (เช่น protobuf + versioning) ทั้งที่ driver 4 (modifiability) ขึ้นอยู่กับสิ่งนี้
- **ความไม่สอดคล้อง:** diagram ไม่มี relay container (`Rel(outbox, mq)` แปลว่า DB เป็นตัว publish เอง); ใน Consequences ฝั่ง Positive อ้างว่า p95 < 500 ms แต่ฝั่ง Negative บอกว่า outbox เพิ่ม latency
- **ข้อกล่าวอ้างที่ไม่มีหลักฐานรองรับ:** "≥ 2 replicas" ไม่ได้แก้ปัญหากรณีที่ DB ของ `loyaltyservice` หรือ RabbitMQ ล่ม; ยังไม่ระบุว่า email consumer ใช้ภาษาอะไร (ADR ยอมรับเองว่ายังไม่รู้)

## 4. Verdict: **Accept with changes**

1. **ปิดเส้นทางที่ทำให้ event หาย:** เลือก outbox store ให้ชัด (หรือตัด outbox ออกแล้วใช้ publisher confirms) และเปลี่ยน "log แล้วข้าม" เป็น fallback ที่ durable + reconcile job ที่เทียบ order กับ points
2. **กำหนด rate limit ของ MailFast แบบ global** (token bucket ที่ใช้ร่วมกันหรือ consumer ตัวเดียว) พร้อม prefetch ประมาณ 75–100 และคำนวณเวลาระบาย backlog ใหม่ให้รวม order ที่ยังเข้ามาหลัง peak
3. **วัดก่อน commit:** ตั้ง latency budget ของการเขียน outbox/publish ใน `PlaceOrder` (เช่น ≤ 20 ms p95) และนิยาม schema ของ `OrderPlaced` ให้มี version; ยืนยันกับ marketing ว่า fraud scoring เป็นงานหลังยืนยัน order จริง
