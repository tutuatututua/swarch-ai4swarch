ตัวแทน: Codex; โมเดล: GPT-6

# ADR-NNNN: ใช้ gRPC แบบรอผลและรวมสัญญาบริการไว้ใน demo.proto

- **Status:** Reconstructed
- **Date:** 2026-11-01 (วันที่จัดทำย้อนหลัง; ไม่ทราบวันที่ตัดสินใจเดิม)
- **Deciders:** ไม่ปรากฏในไฟล์หลักฐานที่ได้รับ

## Context and problem

หมายเลข C1–C20 เชื่อมทุกข้อกล่าวอ้างกับประเภทและหลักฐานใน **Claim check** ท้ายเอกสาร

- [C1] **Evidenced:** `demo.proto` ใช้ `proto3`, package `hipstershop` และรวม 9 service ได้แก่ CartService, RecommendationService, ProductCatalogService, ShippingService, CurrencyService, PaymentService, EmailService, CheckoutService และ AdService; RPC ทุกตัวในไฟล์เป็น unary ไม่มี `stream` (หลักฐาน: `system/online-boutique/protos/demo.proto`, ประกาศ `service` และ `rpc`)
- [C2] **Evidenced:** หลายสัญญาใช้ชนิดข้อมูลร่วม เช่น `Money` ใน Product, ChargeRequest และ CurrencyConversionRequest และ `Address` ในคำขอจัดส่งกับ PlaceOrderRequest (หลักฐาน: `system/online-boutique/protos/demo.proto`, message ดังกล่าว)
- [C3] **Evidenced:** `checkoutservice` เปิด gRPC server และเตรียม connection ไปยัง product catalog, cart, currency, shipping, email และ payment (หลักฐาน: `system/online-boutique/src/checkoutservice/main.go`, `main`, `mustConnGRPC`, `pb.RegisterCheckoutServiceServer`)
- [C4] **Evidenced:** `PlaceOrder` รอเตรียมข้อมูลและราคา แล้วเรียกชำระเงิน จัดส่ง ล้างตะกร้า และส่งอีเมลก่อนคืนผล; helper เรียก RPC แล้วรับผลหรือ error โดยตรง (หลักฐาน: `system/online-boutique/src/checkoutservice/main.go`, `PlaceOrder` และ helper ที่เรียก)
- [C5] **Evidenced:** `frontend` รับ HTTP และเตรียม gRPC connection ไปยัง 7 บริการธุรกิจ ได้แก่ product catalog, currency, cart, recommendation, shipping, checkout และ ad (หลักฐาน: `system/online-boutique/src/frontend/main.go`, `main`, `mustConnGRPC`, `http.ListenAndServe`)
- [C6] **[Inferred]:** การใช้ `demo.proto` เป็นสัญญาร่วมสอดคล้องกับชื่อ client และ message ที่ checkout ใช้ แต่ขั้นตอนสร้างโค้ดและการใช้ไฟล์นี้เป็นต้นทางเดียวของทุกบริการไม่ปรากฏในหลักฐาน (ฐานอนุมาน: `system/online-boutique/protos/demo.proto`, `CheckoutService`, `PaymentService`; `system/online-boutique/src/checkoutservice/main.go`, import `genproto`, `chargeCard`)

ขอบเขต: คำกล่าวว่า “ทุกบริการสื่อสารแบบ synchronous ผ่าน gRPC เท่านั้น” ไม่ปรากฏหลักฐานยืนยันในไฟล์ที่ได้รับ (`Not evidenced in the provided files`) โดย `frontend/main.go` ไม่แสดงเนื้อหา handler ที่เรียกบริการธุรกิจ และไม่แสดง protocol ของ `shoppingAssistantSvcAddr` จึงไม่ขยาย C3–C5 ไปทั้งระบบ การประกาศ unary RPC เพียงอย่างเดียวไม่ยืนยันรูปแบบการรอของผู้เรียกทั้งหมด เหตุผลดั้งเดิม ผู้ตัดสินใจ และผล benchmark ก็ไม่ปรากฏในหลักฐาน

## Decision drivers

ลำดับต่อไปนี้เป็นการกู้คืนเหตุผล **[Inferred]** ไม่ใช่ลำดับที่ผู้พัฒนายืนยัน:

1. [C7] ความเข้าใจง่ายและ modifiability: ให้ขั้นตอน checkout และการจัดการผลตอบกลับอยู่ใน flow เดียว (ฐานอนุมาน: `system/online-boutique/src/checkoutservice/main.go`, `PlaceOrder`)
2. [C8] ความสอดคล้องของสัญญาและ maintainability: ลดการนิยามข้อมูลซ้ำด้วยชนิดข้อมูลร่วม (ฐานอนุมาน: `system/online-boutique/protos/demo.proto`, `Money`, `Address`, `OrderResult`)

## Considered options

รายการนี้เป็นทางเลือกที่เสนอเพื่อวิเคราะห์ย้อนหลัง ไม่ใช่หลักฐานว่าทีมเคยพิจารณาจริง:

1. ก. ใช้ synchronous gRPC และรวมสัญญาไว้ใน `demo.proto` ตามรูปแบบที่พบ
2. ข. ใช้ synchronous gRPC แต่แยกสัญญาตามบริการและกำหนดขอบเขตชนิดข้อมูลร่วม
3. ค. ใช้ gRPC สำหรับการอ่านที่ต้องได้คำตอบทันที และใช้ message broker สำหรับขั้นตอนคำสั่งซื้อแบบ asynchronous พร้อมสถานะให้ติดตาม

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| ก | [C7–C8, Inferred] อ่าน flow ได้ต่อเนื่องและตรวจสัญญาร่วมจากจุดเดียว | [C9, Inferred] ต้องรอหลาย RPC บนเส้นทางตอบกลับ; [C10, Inferred] การแก้ชนิดข้อมูลร่วมอาจกระทบหลายสัญญา |
| ข | [C11, Assumption] แบ่งความรับผิดชอบในการเปลี่ยนสัญญาของแต่ละบริการได้ชัดขึ้น | [C12, Assumption] ต้องบริหาร dependency/version ของชนิดข้อมูลร่วม หรือเพิ่มการแปลงข้อมูล; [C9, Inferred] การแยกไฟล์อย่างเดียวไม่ลดการรอ RPC ใน flow เดิม |
| ค | [C13, Assumption] คิวรองรับงานที่รอประมวลผลและแยกเวลารับงานจากเวลาทำงานได้ | [C14, Assumption] เพิ่มภาระดูแล broker และออกแบบสถานะ การส่งซ้ำ และ idempotency; [C15, Inferred] ถ้าย้ายการจัดส่งไปทำภายหลัง ต้องปรับวิธีให้ tracking ID ซึ่งปัจจุบันคืนผ่าน PlaceOrder |

## Decision

กู้คืนเป็นทางเลือก **ก** เฉพาะขอบเขตที่หลักฐานรองรับ: รวมสัญญาในไฟล์เดียวและเรียก gRPC แบบรอผลใน checkout (C1–C4) ส่วนการใช้ต้นทางเดียวของทุกบริการยังเป็นข้ออนุมาน C6

เหตุผลที่น่าจะเลือก **[Inferred]** คือให้ checkout อ่านและประสานงานได้ตรงไปตรงมา (C7) พร้อมใช้ชนิดข้อมูลสอดคล้องกัน (C8) โดยยอมรับต้นทุนการรอและผลกระทบจากการเปลี่ยนสัญญาร่วม (C9–C10) นี่เป็นคำอธิบายย้อนหลัง ไม่ใช่การยืนยันเจตนาของผู้พัฒนา ไม่มีหลักฐานว่าตัดสินใจเพราะ gRPC เร็วกว่า protocol อื่น

## Consequences

- **Positive:** [C7, Inferred] ติดตามขั้นตอน checkout ได้จาก flow เดียว (คุณลักษณะคุณภาพ: understandability, modifiability); [C8, Inferred] นิยามข้อมูลร่วมช่วยลดความซ้ำซ้อนของสัญญา (maintainability)
- **Negative:** [C9, Inferred] latency ของ RPC ที่ต้องรอมีส่วนเพิ่มเวลาตอบกลับ รวมถึงอีเมลที่เรียกก่อนคืนผล (performance); [C10, Inferred] การเปลี่ยน `Money` หรือ `Address` อาจต้องประเมินผู้ใช้หลายสัญญา (modifiability)
- **Negative:** [C16, Evidenced] error จากการเตรียมข้อมูล ชำระเงิน หรือจัดส่งทำให้ `PlaceOrder` คืน error; error ล้างตะกร้าถูกละทิ้ง และ error อีเมลถูกบันทึกแล้วคืนผลสำเร็จได้ (availability, reliability; `system/online-boutique/src/checkoutservice/main.go`, `PlaceOrder`)
- **Risks and mitigations:** [C17, Inferred] เสี่ยงชำระเงินสำเร็จแต่จัดส่งล้มเหลว เพราะเรียกชำระเงินก่อนจัดส่งและไม่แสดงการชดเชยในสาขา error นี้; เสนอให้กำหนดวิธีชดเชยและ idempotency แล้วทดสอบ ยังไม่ใช่กลไกที่ยืนยันว่ามีอยู่ (reliability; `system/online-boutique/src/checkoutservice/main.go`, `PlaceOrder`)
- **Risks and mitigations:** [C18, Evidenced] `convertCurrency` ใช้ `context.TODO()` แทน `ctx` ที่รับมา และ `mustConnGRPC` ในทั้งสองไฟล์ทิ้ง context ที่ได้จาก `WithTimeout`; จึงไม่ใช่หลักฐานของ deadline 3 วินาทีสำหรับ business RPC (performance, availability; `system/online-boutique/src/checkoutservice/main.go`, `convertCurrency`, `mustConnGRPC`; `system/online-boutique/src/frontend/main.go`, `mustConnGRPC`)
- **Risks and mitigations:** [C19, Inferred] เสนอให้ตรวจการส่งต่อ context และกำหนด deadline ตามงบ latency รวม (performance, availability; ฐานอนุมาน C9, C18) รวมทั้งตรวจ compatibility ก่อนเปลี่ยนชนิดข้อมูลร่วม (modifiability; ฐานอนุมาน C2, C10) ทั้งหมดเป็นข้อเสนอ ไม่ใช่สิ่งที่ยืนยันว่าทำแล้ว

## Validation

[C20, Inferred] แผนเสนอจากโครงสร้าง C2, C4, C6 และข้อกังวล C9, C16–C19; ยังไม่ได้รันทดสอบและไม่มีผลวัดในหลักฐาน:

- วัด latency p50/p95/p99 ของ `PlaceOrder` และแต่ละ RPC โดยเพิ่มจำนวนสินค้าและจำลองปลายทางช้า; กำหนดเกณฑ์ยอมรับร่วมกับผู้รับผิดชอบก่อนทดสอบ
- จำลอง payment สำเร็จแต่ shipping ล้มเหลว รวมถึง email ล้มเหลวและ currency ไม่ตอบกลับ เพื่อตรวจผลลัพธ์คำสั่งซื้อ การชดเชย และ deadline
- ตรวจ compatibility และขั้นตอนสร้างโค้ดเพื่อยืนยันต้นทางสัญญาร่วม โดยต้องได้รับขอบเขตหลักฐานเพิ่มเติมก่อนอ่านไฟล์นอกชุดนี้

## Claim check

ข้อกล่าวอ้างที่กล่าวซ้ำใช้แถวเดิม; ช่องหลักฐานของข้ออนุมานระบุเพียงฐานที่ใช้อนุมาน ไม่ใช่ข้อพิสูจน์เหตุผลดั้งเดิม

| # | Claim | Type | Evidence |
|---|---|---|---|
| C1 | ไฟล์เดียวประกาศ 9 บริการใน proto3 และ RPC ทั้งหมดเป็น unary | Evidenced | `system/online-boutique/protos/demo.proto`: `syntax`, `package`, ทุก `service/rpc` |
| C2 | หลายสัญญาใช้ Money และ Address ร่วมกัน | Evidenced | `system/online-boutique/protos/demo.proto`: `Product`, `ChargeRequest`, `CurrencyConversionRequest`, `GetQuoteRequest`, `ShipOrderRequest`, `PlaceOrderRequest` |
| C3 | checkoutservice เปิด gRPC server และเตรียม connection ไปยัง 6 บริการ | Evidenced | `system/online-boutique/src/checkoutservice/main.go`: `main`, `mustConnGRPC`, `pb.RegisterCheckoutServiceServer` |
| C4 | checkout รอผลแต่ละขั้นตอนและ RPC ก่อนคืนคำตอบ | Evidenced | `system/online-boutique/src/checkoutservice/main.go`: `PlaceOrder`, `prepareOrderItemsAndShippingQuoteFromCart`, `getUserCart`, `prepOrderItems`, `quoteShipping`, `convertCurrency`, `chargeCard`, `shipOrder`, `emptyUserCart`, `sendOrderConfirmation` |
| C5 | frontend รับ HTTP และเตรียม gRPC connection ไปยัง 7 บริการธุรกิจ | Evidenced | `system/online-boutique/src/frontend/main.go`: `main`, `mustConnGRPC`, `http.ListenAndServe` |
| C6 | demo.proto น่าจะเป็นต้นทางสัญญาร่วม แต่ยังยืนยันทุกบริการไม่ได้ | Inferred | `system/online-boutique/protos/demo.proto`: `CheckoutService`, `PaymentService`; `system/online-boutique/src/checkoutservice/main.go`: import `genproto`, `chargeCard`; ไม่ปรากฏขั้นตอนสร้างโค้ด |
| C7 | ความง่ายในการอ่านและประสาน checkout น่าจะเป็นแรงจูงใจและข้อดี | Inferred | `system/online-boutique/src/checkoutservice/main.go`: `PlaceOrder`; ไม่ระบุเจตนาผู้พัฒนา |
| C8 | ลดนิยามข้อมูลซ้ำและรักษาความสอดคล้องน่าจะเป็นแรงจูงใจและข้อดี | Inferred | `system/online-boutique/protos/demo.proto`: `Money`, `Address`, `OrderResult`; ไม่ระบุเหตุผลดั้งเดิม |
| C9 | การรอ RPC ตามลำดับเพิ่มเวลาบนเส้นทางตอบกลับ; การแยกไฟล์สัญญาอย่างเดียวไม่ลดการรอนี้ | Inferred | `system/online-boutique/src/checkoutservice/main.go`: `PlaceOrder`, `prepOrderItems`, `sendOrderConfirmation`; ไม่มีผลวัด latency |
| C10 | การแก้ชนิดข้อมูลร่วมอาจกระทบหลายสัญญา | Inferred | `system/online-boutique/protos/demo.proto`: ผู้ใช้ `Money` และ `Address`; ไม่แสดงกระบวนการ release |
| C11 | แยกสัญญาตามบริการช่วยแบ่งความรับผิดชอบการเปลี่ยนแปลง | Assumption | หลักการออกแบบทั่วไป; ไม่ปรากฏหลักฐานการทดลองทางเลือกนี้ในไฟล์ที่ได้รับ |
| C12 | การแยกสัญญาต้องบริหารชนิดข้อมูลร่วม หรือการแปลงข้อมูล | Assumption | trade-off ทั่วไปของการแบ่งสัญญา; ไม่ใช่ข้อเท็จจริงของระบบที่ตรวจพบ |
| C13 | คิวสามารถพักงานและแยกเวลารับงานจากเวลาประมวลผล | Assumption | คุณสมบัติทั่วไปของทางเลือก asynchronous; ไม่ปรากฏหลักฐานการใช้ broker ในไฟล์ที่ได้รับ |
| C14 | ทางเลือก broker เพิ่มภาระดูแลและออกแบบสถานะ การส่งซ้ำ และ idempotency | Assumption | trade-off ทั่วไปของทางเลือกที่เสนอ; ไม่ใช่เหตุผลดั้งเดิมที่มีบันทึก |
| C15 | การจัดส่งภายหลังต้องปรับวิธีให้ tracking ID ในผล checkout | Inferred | `system/online-boutique/protos/demo.proto`: `PlaceOrderResponse`, `OrderResult.shipping_tracking_id`; `system/online-boutique/src/checkoutservice/main.go`: `PlaceOrder` |
| C16 | error ขั้นตอนหลักทำให้ checkout คืน error แต่ล้างตะกร้าและอีเมลมีการจัดการต่างกัน | Evidenced | `system/online-boutique/src/checkoutservice/main.go`: สาขา error ใน `PlaceOrder`, `_ = cs.emptyUserCart`, การบันทึกคำเตือนหลังส่งอีเมล |
| C17 | เสี่ยงจ่ายแล้วแต่จัดส่งล้มเหลว; ควรออกแบบการชดเชยและ idempotency | Inferred | `system/online-boutique/src/checkoutservice/main.go`: `PlaceOrder` เรียก `chargeCard` ก่อน `shipOrder` และคืน error โดยไม่แสดงการชดเชย; ไม่ยืนยันพฤติกรรมภายในบริการปลายทาง |
| C18 | convertCurrency ไม่ใช้ ctx ที่รับมา และ context timeout ใน mustConnGRPC ถูกทิ้ง | Evidenced | `system/online-boutique/src/checkoutservice/main.go`: `Convert(context.TODO(), ...)`, `mustConnGRPC`; `system/online-boutique/src/frontend/main.go`: `mustConnGRPC`, `_, cancel := context.WithTimeout(...)` |
| C19 | ควรตรวจ context/deadline และ compatibility ของชนิดข้อมูลร่วม | Inferred | ข้อเสนอจาก `system/online-boutique/src/checkoutservice/main.go`: `PlaceOrder`, `convertCurrency`; `system/online-boutique/protos/demo.proto`: `Money`, `Address`; ไม่ปรากฏหลักฐานว่ามีกระบวนการเหล่านี้แล้ว |
| C20 | ควรวัด latency จำลองความล้มเหลว และตรวจ compatibility/ต้นทางสร้างโค้ด | Inferred | แผนเสนอจาก `system/online-boutique/src/checkoutservice/main.go`: `PlaceOrder`, `prepOrderItems`, `convertCurrency`, import `genproto`; `system/online-boutique/protos/demo.proto`: `Money`, `Address`; ยังไม่มีผลทดสอบ |
