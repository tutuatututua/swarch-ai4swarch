**ภาษา:** ภาษาไทย | [English](bonus-recover-decision.en.md)

# โบนัส (ไม่บังคับ ทำที่บ้าน): กู้คืนการตัดสินใจที่ไม่ได้บันทึกไว้ (ประมาณ 15 นาที)

> **เชื่อมโยงกับการบรรยาย:** T5 (Dhar et al. สร้างการตัดสินใจจากโค้ดและแผนภาพเพื่อลด *การสูญหายของความรู้* หรือ knowledge vaporization), AICH4
> **คำถาม:** เมื่อ AI อธิบายว่า*ทำไม*ระบบจึงถูกสร้างแบบนี้ คำอธิบายส่วนใดมาจากหลักฐาน และส่วนใดเป็นการเดาอย่างมั่นใจ

## ลงมือทำ

เริ่มแชตหรือ session ใหม่ แล้วส่งข้อความนี้ให้ agent

```
Follow prompts/bonus-recover-decision.md
decision: Why do all services communicate synchronously over gRPC, using one shared demo.proto?
evidence: system/online-boutique/protos/demo.proto, system/online-boutique/src/checkoutservice/main.go, system/online-boutique/src/frontend/main.go
language: Thai
```

บรรทัด `language: Thai` ทำให้ ADR ที่กู้คืนเป็นภาษาไทย หากต้องการภาษาอังกฤษให้ลบบรรทัดนี้

agent จะเขียน ADR ที่กู้คืนมา พร้อมตาราง **Claim check** ไว้ที่ [`outputs/bonus-recovered-adr.md`](outputs/bonus-recovered-adr.md)

## ตรวจสอบตาราง claim check

เลือกข้อกล่าวอ้าง**ห้าข้อ**จากตารางของ agent โดยเป็นข้อที่ agent ระบุว่า *Evidenced* (มีหลักฐาน) อย่างน้อยสองข้อ และ *Inferred* (อนุมานเอา) อย่างน้อยสองข้อ แล้วตรวจสอบแต่ละข้อด้วยตัวเอง

| # | ข้อกล่าวอ้าง (ย่อ) | AI ระบุว่า | นิสิตระบุว่า | เหตุผล (path ของไฟล์หรือการให้เหตุผล) |
|---|---|---|---|---|
| 1 | C1: `demo.proto` รวม 9 บริการ ใช้ proto3 และ RPC ทุกตัวเป็น unary | Evidenced | Evidenced | ตรวจ `system/online-boutique/protos/demo.proto` แล้วพบ `syntax = "proto3"` และประกาศ `service` 9 รายการจริง ส่วน RPC ไม่มีตัวไหนระบุ `stream` แต่ข้อมูลนี้ยังบอกไม่ได้ว่าผู้เรียกทุกบริการต้องรอผลเสมอ |
| 2 | C16: checkout จัดการ error แต่ละขั้นตอนไม่เหมือนกัน | Evidenced | Evidenced | ใน `system/online-boutique/src/checkoutservice/main.go`, ฟังก์ชัน `PlaceOrder` ถ้าเตรียมข้อมูล ชำระเงิน หรือจัดส่งไม่สำเร็จ จะคืน error แต่ตอนล้างตะกร้าใช้ `_ = cs.emptyUserCart(...)` จึงไม่สนใจ error ส่วนส่งอีเมลไม่สำเร็จจะบันทึกคำเตือนแล้วคืนผลสำเร็จได้ |
| 3 | C18: `convertCurrency` ไม่ใช้ `ctx` ที่รับมา และ `mustConnGRPC` ไม่ได้นำ context ที่มี timeout ไปใช้ | Evidenced | Evidenced | ใน `system/online-boutique/src/checkoutservice/main.go`, `convertCurrency` ใช้ `context.TODO()` จริง และ `mustConnGRPC` ทั้งในไฟล์นี้กับ `system/online-boutique/src/frontend/main.go` ใช้ `_, cancel := context.WithTimeout(...)` จึงยังสรุปไม่ได้ว่า RPC ถูกจำกัดเวลาไว้ 3 วินาที |
| 4 | C7: ทีมเลือกวิธีนี้เพราะอยากให้ checkout อ่านและจัดการได้ง่าย | Inferred | Inferred | `system/online-boutique/src/checkoutservice/main.go`, `PlaceOrder` แสดงขั้นตอนต่อกันในฟังก์ชันเดียวจริง แต่ไม่ได้บอกว่าความอ่านง่ายเป็นเหตุผลที่ทีมเลือกใช้วิธีนี้ จึงยังเป็นการคาดเดาเหตุผลจากโค้ด |
| 5 | C8: ทีมรวมสัญญาไว้ด้วยกันเพื่อลดข้อมูลซ้ำและให้บริการใช้ข้อมูลตรงกัน | Inferred | Inferred | `system/online-boutique/protos/demo.proto` ใช้ `Money` และ `Address` ร่วมกันในหลาย message จริง แต่การบอกว่าทีมทำแบบนี้ “เพื่อลดความซ้ำซ้อน” ยังไม่มีคำอธิบายจากผู้พัฒนารองรับ |

**นับ:** จากข้อกล่าวอ้างทั้งหมด 20 ข้อ agent ระบุว่า Evidenced **7 ข้อ** และเมื่อตรวจเทียบกับโค้ดแล้ว ยังจัดเป็น Evidenced **7 ข้อ** ได้แก่ C1–C5, C16 และ C18 ส่วนในตารางที่เลือกมาตรวจ 5 ข้อข้างบน มี Evidenced **3 ข้อ**

## สะท้อนคิด

1. agent เคยระบุการเดาว่าเป็น *Evidenced* หรือไม่ ยกตัวอย่าง หรืออธิบายว่าตรวจสอบอย่างไรว่าไม่มี

   ใน ADR นี้ยังไม่พบว่า agent เอาการเดามาระบุเป็น Evidenced เพราะทั้ง 7 ข้อที่ใช้ป้ายนี้ตรวจย้อนกลับไปหาโค้ดได้ เช่น C1 นับบริการได้จาก `system/online-boutique/protos/demo.proto` และ C16 ดูวิธีจัดการ error ได้จาก `PlaceOrder` ใน `system/online-boutique/src/checkoutservice/main.go`

   วิธีตรวจคือแยกให้ออกว่าข้อความนั้นบอกว่า “โค้ดทำอะไร” หรือ “คนเขียนเลือกทำแบบนี้เพราะอะไร” เช่น C8 มีหลักฐานว่าใช้ `Money` และ `Address` ร่วมกันจริง แต่ยังไม่มีหลักฐานว่าทีมเลือกทำเพื่อให้ดูแลง่ายขึ้น การจัดเหตุผลส่วนนี้เป็น Inferred จึงเหมาะสมแล้ว

2. หากนักพัฒนาใหม่อ่านเฉพาะ ADR ที่กู้คืนมานี้ เขาอาจเข้าใจผิดในเรื่องใด

   อาจเข้าใจว่าทุกบริการในระบบใช้ gRPC แบบรอผลทั้งหมด ทั้งที่หลักฐานที่อ่านยืนยันได้แค่บางส่วน เช่น `PlaceOrder` ใน `system/online-boutique/src/checkoutservice/main.go` เรียกแต่ละขั้นตอนแล้วรอผล ส่วน `system/online-boutique/src/frontend/main.go` แสดงการเตรียม gRPC connection แต่ไม่ได้แสดงโค้ดข้างใน handler ที่เรียกบริการ จึงยังเหมารวมทั้งระบบไม่ได้

   อีกเรื่องคืออาจอ่านเหตุผลใน C7–C8 แล้วคิดว่าเป็นคำอธิบายจากทีมเดิม ทั้งที่เป็นเหตุผลที่ AI เสนอจากรูปแบบโค้ด เช่นเดียวกับทางเลือกใน Considered options ที่ยังไม่มีหลักฐานว่าทีมเคยนำมาเปรียบเทียบกันจริง ประวัติการตัดสินใจเหล่านี้ไม่ปรากฏในไฟล์หลักฐานที่ได้รับ (Not evidenced in the provided files)

3. AI กู้คืนอะไรจากโค้ดได้บ้าง และอะไรที่มีเพียงสถาปนิกผู้ออกแบบเดิมเท่านั้นที่รู้ ยกตัวอย่างหนึ่งตัวอย่างจาก ADR ของนิสิต

   AI ช่วยบอกได้ว่าระบบแบ่งเป็นบริการอะไร ใช้ข้อมูลแบบไหนร่วมกัน และทำงานตามลำดับอย่างไร ตัวอย่างจาก `system/online-boutique/src/checkoutservice/main.go`, `PlaceOrder` เห็นชัดว่าเรียก `chargeCard` ก่อน `shipOrder` จึงอธิบายลำดับการทำงานส่วนนี้ได้จากโค้ดโดยตรง

   แต่คำถามว่าทำไมทีมถึงเลือก synchronous gRPC หรือทำไมถึงรวมสัญญาไว้ในไฟล์เดียว ยังตอบให้แน่ชัดไม่ได้ เช่น C7 ที่บอกว่าเลือกเพื่อให้อ่าน flow ง่าย เป็นเพียงเหตุผลที่เป็นไปได้ หากอยากรู้เหตุผลจริง ต้องถามคนที่ร่วมตัดสินใจหรือหาบันทึกจากช่วงที่ออกแบบระบบ เพราะโค้ดชุดนี้ไม่ได้บอกว่าทีมให้ความสำคัญกับอะไรและเคยตัดทางเลือกไหนทิ้งไป
