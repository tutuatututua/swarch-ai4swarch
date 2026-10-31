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
| `checkoutservice` → `productcatalogservice` | `system/online-boutique/kubernetes-manifests/checkoutservice.yaml:57–58` — `PRODUCT_CATALOG_SERVICE_ADDR=productcatalogservice:3550`; `system/online-boutique/kubernetes-manifests/productcatalogservice.yaml:79–81` — พอร์ต gRPC 3550 | ✅ | `system/online-boutique/src/checkoutservice/main.go:341–344` สร้าง `ProductCatalogServiceClient` และเรียก `GetProduct` เพื่อดึงข้อมูลสินค้า |
| `checkoutservice` → `shippingservice` | `system/online-boutique/kubernetes-manifests/checkoutservice.yaml:59–60` — `SHIPPING_SERVICE_ADDR=shippingservice:50051`; `system/online-boutique/kubernetes-manifests/shippingservice.yaml:79–81` — พอร์ต gRPC 50051 | ✅ | `system/online-boutique/src/checkoutservice/main.go:314–317` เรียก `GetQuote` และบรรทัด 387–389 เรียก `ShipOrder` |
| `checkoutservice` → `paymentservice` | `system/online-boutique/kubernetes-manifests/checkoutservice.yaml:61–62` — `PAYMENT_SERVICE_ADDR=paymentservice:50051`; `system/online-boutique/kubernetes-manifests/paymentservice.yaml:79–81` — พอร์ต gRPC 50051 | ✅ | `system/online-boutique/src/checkoutservice/main.go:370–372` เรียก `Charge` โดยส่งจำนวนเงินและข้อมูลบัตร |
| `checkoutservice` → `emailservice` | `system/online-boutique/kubernetes-manifests/checkoutservice.yaml:63–64` — `EMAIL_SERVICE_ADDR=emailservice:5000`; `system/online-boutique/kubernetes-manifests/emailservice.yaml:81–83` — พอร์ต gRPC 5000 ส่งต่อไปพอร์ต 8080 | ✅ | `system/online-boutique/src/checkoutservice/main.go:380–382` เรียก `SendOrderConfirmation`; พอร์ต 5000 ที่ผู้เรียกใช้ตรงกับพอร์ตของ Kubernetes Service |
| `checkoutservice` → `currencyservice` | `system/online-boutique/kubernetes-manifests/checkoutservice.yaml:65–66` — `CURRENCY_SERVICE_ADDR=currencyservice:7000`; `system/online-boutique/kubernetes-manifests/currencyservice.yaml:80–82` — พอร์ต gRPC 7000 | ✅ | `system/online-boutique/src/checkoutservice/main.go:360–362` เรียก `Convert` โดยส่งจำนวนเงินต้นทางและสกุลเงินปลายทาง |
| `checkoutservice` → `cartservice` | `system/online-boutique/kubernetes-manifests/checkoutservice.yaml:67–68` — `CART_SERVICE_ADDR=cartservice:7070`; `system/online-boutique/kubernetes-manifests/cartservice.yaml:80–82` — พอร์ต gRPC 7070 | ✅ | `system/online-boutique/src/checkoutservice/main.go:325` เรียก `GetCart` และบรรทัด 333 เรียก `EmptyCart` |

`system/online-boutique/src/checkoutservice/main.go:111–123` อ่านค่า environment variable และส่งค่าปลายทางให้ `mustConnGRPC` สำหรับทั้งหก service ส่วน RPC ที่เรียกตรงกับสัญญาใน `system/online-boutique/protos/demo.proto` (`ProductCatalogService`, `ShippingService`, `PaymentService`, `EmailService`, `CurrencyService`, `CartService`)

- **สิ่งที่ผิดหรือขาดหายไป** ใน Run B (dependency ที่ไม่มีจริง, dependency ที่ขาดไป, container ที่น่าสงสัย): ยังไม่พบ dependency ผิดใน 6 รายการที่ตรวจ ผลนี้ยังไม่ครอบคลุม dependency ทั้งหมดในแผนภาพ
- ข้อมูลที่เติมได้คือผลกระทบเมื่อ dependency ล้มเหลวแตกต่างกัน: การชำระเงินผิดพลาดทำให้ `PlaceOrder` คืน error (`system/online-boutique/src/checkoutservice/main.go:252–255`) แต่การส่งอีเมลผิดพลาดจะบันทึกคำเตือนและยังคืนผลสำเร็จได้ (`system/online-boutique/src/checkoutservice/main.go:273–279`) รายละเอียดนี้ช่วยเติมข้อมูลด้าน availability ซึ่ง Run B ยังไม่ได้แสดง
- `loadgenerator` มี Deployment ใน `system/online-boutique/kubernetes-manifests/loadgenerator.yaml:14–19` แต่ถูก comment ออกจาก base Kustomization ตาม `system/online-boutique/kubernetes-manifests/kustomization.yaml:24` โดย Run B ระบุเงื่อนไขนี้ไว้ถูกต้องแล้ว

## สะท้อนคิด (ข้อละ 2–3 ประโยค)

1. เปรียบเทียบ A กับ B: agent **ไม่สามารถ**รู้อะไรได้จากไฟล์ `.proto` เพียงไฟล์เดียว และเพราะเหตุใด

   `.proto` ระบุ interface, RPC และโครงสร้างข้อความ เช่น `CheckoutService.PlaceOrder` แต่ไม่ได้ระบุว่า service ใดเป็นผู้เรียก รวมถึงที่อยู่ปลายทางและขอบเขต deployment (`system/online-boutique/protos/demo.proto:224–239`) Run B จึงระบุ dependency ที่ตั้งค่าไว้ได้จาก `*_SERVICE_ADDR` (`system/online-boutique/kubernetes-manifests/checkoutservice.yaml:57–68`) และพบ `redis-cart` จาก `REDIS_ADDR` กับ Deployment (`system/online-boutique/kubernetes-manifests/cartservice.yaml:50–51,89–120`) ซึ่งอนุมานจากรูปแบบข้อมูลใน `.proto` อย่างเดียวไม่ได้

2. ข้อผิดพลาด**ข้อใดข้อหนึ่ง**ที่จะทำให้สมาชิกใหม่ของทีมเข้าใจผิดมากที่สุด หากเผยแพร่แผนภาพนี้โดยไม่ตรวจสอบ

   จากรายการที่ตรวจยังไม่พบเส้นเชื่อมผิด แต่ **[Inferred]** การตีความเส้น `checkoutservice` → `emailservice` ว่าหากส่งอีเมลไม่สำเร็จแล้วการสั่งซื้อต้องล้มเหลว อาจทำให้ประเมิน availability ผิด โค้ดบันทึกคำเตือนเมื่อส่งอีเมลผิดพลาดแล้วคืน `PlaceOrderResponse` สำเร็จต่อได้ จึงควรเพิ่มคำอธิบายผลกระทบของ dependency นี้ประกอบแผนภาพ (`system/online-boutique/src/checkoutservice/main.go:273–279`)

## ไฟล์ที่อ่านในการจัดทำคำตอบ

- `exercise/01-recover-architecture.md`
- `prompts/1-recover-architecture.md`
- `exercise/outputs/1A-architecture.md`
- `exercise/outputs/1B-architecture.md`
- `system/online-boutique/protos/demo.proto`
- `system/online-boutique/kubernetes-manifests/checkoutservice.yaml`
- `system/online-boutique/kubernetes-manifests/productcatalogservice.yaml`
- `system/online-boutique/kubernetes-manifests/shippingservice.yaml`
- `system/online-boutique/kubernetes-manifests/paymentservice.yaml`
- `system/online-boutique/kubernetes-manifests/emailservice.yaml`
- `system/online-boutique/kubernetes-manifests/currencyservice.yaml`
- `system/online-boutique/kubernetes-manifests/cartservice.yaml`
- `system/online-boutique/kubernetes-manifests/kustomization.yaml`
- `system/online-boutique/kubernetes-manifests/loadgenerator.yaml`
- `system/online-boutique/src/checkoutservice/main.go`
