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

| ผู้เรียก → ผู้ถูกเรียก                                   | หลักฐานที่ AI อ้าง                                                                                                 | ยืนยันแล้วหรือไม่ ✅ / ❌ / ❓ | สิ่งที่นิสิตพบ                                                                                                                                                                                                                                        |
| -------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------ | ------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Customer → `frontend` (HTTP)                             | `kubernetes-manifests/frontend.yaml` (Service `frontend-external`, `type: LoadBalancer`, port 80 named `http`)     | ✅                             | yaml มี Service `frontend-external` แบบ `LoadBalancer` port 80 จริง และ `src/frontend/main.go:171` เปิด HTTP server ด้วย `http.ListenAndServe`                                                                                                        |
| `checkoutservice` → `currencyservice` (gRPC)             | `kubernetes-manifests/checkoutservice.yaml` (`CURRENCY_SERVICE_ADDR=currencyservice:7000`)                         | ✅                             | yaml มี `CURRENCY_SERVICE_ADDR` ตรงกัน, `src/checkoutservice/main.go:114,121` อ่าน env และสร้าง connection, `:360` เรียก `Convert`                                                                                                                    |
| `checkoutservice` → `cartservice` (gRPC)                 | `kubernetes-manifests/checkoutservice.yaml` (`CART_SERVICE_ADDR=cartservice:7070`)                                 | ✅                             | yaml มี `CART_SERVICE_ADDR` ตรงกัน, `src/checkoutservice/main.go:325` เรียก `GetCart` และ `:333` เรียก `EmptyCart`                                                                                                                                    |
| `recommendationservice` → `productcatalogservice` (gRPC) | `kubernetes-manifests/recommendationservice.yaml` (`PRODUCT_CATALOG_SERVICE_ADDR=productcatalogservice:3550`)      | ✅                             | yaml มี env ตรงกัน, `src/recommendationservice/recommendation_server.py:131–136` อ่าน env และสร้าง `ProductCatalogServiceStub`, `:73` เรียก `ListProducts`                                                                                            |
| `cartservice` → `redis-cart` (Redis/TCP)                 | `kubernetes-manifests/cartservice.yaml` (`REDIS_ADDR=redis-cart:6379`; Service `redis-cart` port name `tcp-redis`) | ✅                             | `src/cartservice/src/Startup.cs:29–40` ใช้ Redis เฉพาะเมื่อมี `REDIS_ADDR` (ถ้าไม่มีจะลอง Spanner, AlloyDB แล้วจึงใช้ in-memory) ใน deployment นี้ yaml ตั้ง `REDIS_ADDR` ไว้ จึงใช้ Redis จริง                                                       |
| `frontend` → OpenTelemetry collector (gRPC)              | ไม่มี (AI ไม่ได้ระบุ dependency นี้)                                                                               | ❌ ขาดไป                       | `src/frontend/main.go:113` เปิด tracing เมื่อ `ENABLE_TRACING=1`, `:178–179` อ่าน `COLLECTOR_SERVICE_ADDR` แล้วต่อ gRPC ไปส่ง trace; manifest ไม่ได้ตั้งค่านี้ไว้ จึงเป็น dependency แบบ optional ที่มองไม่เห็นจาก yaml                               |
| `checkoutservice` → OpenTelemetry collector (gRPC)       | ไม่มี (AI ไม่ได้ระบุ dependency นี้)                                                                               | ❌ ขาดไป                       | `src/checkoutservice/main.go:90` เช็ค `ENABLE_TRACING`, `:164–165` อ่าน `COLLECTOR_SERVICE_ADDR` แล้วต่อ gRPC; manifest ไม่ได้ตั้งค่านี้ไว้เช่นกัน                                                                                                    |
| `frontend` → `shoppingassistantservice` (HTTP)           | `frontend.yaml` (`SHOPPING_ASSISTANT_SERVICE_ADDR`) ใส่ไว้ในหัวข้อ Gaps แต่ไม่อยู่ในตารางและแผนภาพ                 | ❓                             | yaml บรรทัด 83 ตั้ง address ไว้ และ `src/frontend/handlers.go:464–465` ส่ง HTTP POST ไปหา service นี้จริง (ไม่ใช่ gRPC) แต่ไม่มี manifest ไหน deploy มัน และ `ENABLE_ASSISTANT` ถูก comment ไว้ (`frontend.yaml:93`) ใน deployment นี้จึงยังไม่ได้ใช้ |

- **สิ่งที่ผิดหรือขาดหายไป** ใน Run B (dependency ที่ไม่มีจริง, dependency ที่ขาดไป, container ที่น่าสงสัย):
  - **dependency ที่ไม่มีจริง:** ไม่พบ ทั้ง 5 แถวที่ตรวจมีจุดเรียกจริงในซอร์สโค้ด เพราะ AI ดึง dependency มาจาก env var `*_SERVICE_ADDR` ใน manifest ตรง ๆ ซึ่งตรงกับโค้ดที่อ่าน env เหล่านั้น (`mustMapEnv` ใน `src/frontend/main.go`, `src/checkoutservice/main.go`)
  - **dependency ที่ขาดไป:**
    - `frontend` และ `checkoutservice` → OpenTelemetry collector (gRPC) ทำงานเมื่อ `ENABLE_TRACING=1` (`src/frontend/main.go:113,178`, `src/checkoutservice/main.go:90,164`) AI ไม่ได้พูดถึงเลย เพราะ manifest ไม่ได้ตั้ง `COLLECTOR_SERVICE_ADDR` ไว้
    - `frontend` → `shoppingassistantservice` ผ่าน **HTTP** ไม่ใช่ gRPC (`src/frontend/handlers.go:464–465`) AI ใส่ไว้แค่ในหัวข้อ Gaps ไม่ได้ลงตารางหรือแผนภาพ และไม่ได้ระบุ protocol ทั้งที่ `src/shoppingassistantservice/` มีอยู่จริง
  - **container ที่น่าสงสัย:**
    - `loadgenerator` อยู่ในแผนภาพ แต่ใน deployment ปกติไม่ได้ถูก deploy เพราะถูก comment ไว้ใน `kustomization.yaml:24` จึงไม่ควรแสดงเป็นส่วนหนึ่งของระบบที่ใช้งานจริง (AI เขียนหมายเหตุไว้ แต่ในแผนภาพยังวาดอยู่)
    - `redis-cart` ใช้ `emptyDir` (`cartservice.yaml:141`) ข้อมูลตะกร้าจะหายเมื่อ pod ถูกย้าย แผนภาพที่วาดเป็น `ContainerDb` อาจทำให้คนเข้าใจผิดว่าเป็นที่เก็บข้อมูลถาวร
  - **ข้อมูลที่ AI บอกว่าไม่มีหลักฐาน แต่จริง ๆ หาได้:** ภาษาของแต่ละ service AI เขียนว่า "not evidenced" ทั้งหมด แต่ซอร์สโค้ดบอกชัด เช่น `checkoutservice` เป็น Go (`main.go`), `recommendationservice` เป็น Python (`recommendation_server.py`), `cartservice` เป็น C# (`Startup.cs`) ข้อนี้ AI ไม่ได้ผิด เพราะถูกจำกัดให้อ่านแค่ proto และ manifest แต่แผนภาพจะขาดข้อมูลส่วนนี้ไป

## สะท้อนคิด (ข้อละ 2–3 ประโยค)

1. เปรียบเทียบ A กับ B: agent **ไม่สามารถ**รู้อะไรได้จากไฟล์ `.proto` เพียงไฟล์เดียว และเพราะเหตุใด

   Run A รู้แค่ว่ามี service อะไรบ้าง 9 ตัวและแต่ละตัวมี RPC อะไร แต่ไม่รู้เลยว่าใครเรียกใคร (ตาราง dependency ว่าง แผนภาพไม่มี `Rel` สักเส้น) และไม่รู้ว่ามี `frontend`, `redis-cart`, port หรือทางเข้าของลูกค้า เพราะ `.proto` เป็นแค่สัญญาฝั่ง server ว่า service **ให้บริการ**อะไร ไม่ได้บอกว่าฝั่ง client คือใคร Run B ได้ dependency เพิ่มมา 17 เส้นจาก env var `*_SERVICE_ADDR` ใน manifest แต่ก็ยังมองไม่เห็น dependency ที่ตั้งค่าในโค้ดแบบมีเงื่อนไข เช่น OpenTelemetry collector

2. ข้อผิดพลาด**ข้อใดข้อหนึ่ง**ที่จะทำให้สมาชิกใหม่ของทีมเข้าใจผิดมากที่สุด หากเผยแพร่แผนภาพนี้โดยไม่ตรวจสอบ

   ข้อที่อันตรายที่สุดคือแผนภาพแสดง `redis-cart` เป็น `ContainerDb` เหมือนฐานข้อมูลถาวร ทั้งที่จริงใช้ `emptyDir` (`cartservice.yaml:141`) ข้อมูลตะกร้าจะหายทุกครั้งที่ pod ถูกย้ายหรือ restart สมาชิกใหม่อาจเข้าใจว่าตะกร้าของลูกค้าปลอดภัย แล้วออกแบบฟีเจอร์ที่พึ่งข้อมูลนี้ (เช่น ตะกร้าที่ค้างไว้ข้ามวัน) โดยไม่รู้ว่ามันหายได้ตลอด
