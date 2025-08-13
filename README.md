# X_RecurringPayment

# ภาพรวม ขอบเขตโดเมน (Bounded Contexts)

* **Core Domain**
1. Recurring Orchestrator – หัวใจของระบบ recurring: สร้าง/จัดคิวรอบตัดบัตร, สร้างรายการตัดเงิน, ติดตามสถานะ, วางแผน retry
     
* **Supporting Domains**
2. **Payer & Mandate (Consent/Payment Method)** – เก็บผู้ชำระ/วิธีชำระ/การยินยอม (mandate)
3. **Billing / Invoicing** – ออกใบแจ้งหนี้/อ้างอิงยอดที่จะเรียกเก็บ
4. **Policy** (จากธุรกิจประกัน) – แหล่งความจริงของข้อมูลกรมธรรม์/เบี้ย/งวดชำระ
5. **Channel & Product Catalog** – ตั้งค่าช่องทางและสินค้าที่เปิดให้ recurring ได้/รูปแบบที่รองรับ
6. **Payment Gateway Connector** – ครอบ (Adapter/ACL) สำหรับเรียกเก็บจริงกับ PG และเก็บ transaction log
7. **Notification** – ส่งข้อความตาม template (SMS/LINE/Email) ตามเหตุการณ์
  
* **Generic Subdomains**
8. **IAM/Admin** – สิทธิ์เมนู บทบาท พนักงาน
9. **Reference/Utility** – RunningNumbers, TitleNames (อ้างอิงทั่วไป)

> หมายเหตุ “Mandate” = การให้สิทธิ์ตัดบัญชี/บัตรแบบต่อเนื่อง (เช่น Direct Debit/บัตรเครดิต) ซึ่งในระบบนี้ผูกกับ **Consent / PaymentMethodConsent**





# Domain Overview
<img width="1917" height="760" alt="image" src="https://github.com/user-attachments/assets/c55f5b70-e82f-4ee9-8505-f86e69a880a4" />



# Mapping ตาราง → Bounded Context

## 1) Recurring Orchestrator (Core)

**หน้าที่**: ลงทะเบียนแผน recurring, create/queue payment attempt, เปลี่ยนสถานะ, เก็บประวัติ, วางแผน retry
**ตาราง**

* `RecurringRequests` (แม่แบบ/แผน recurring ต่อกรมธรรม์/ผู้ชำระ)
* `RecurringPayments` (instance ของการตัดเงินแต่ละครั้ง)
* `RecurringStatusHistories` (ไทม์ไลน์สถานะ)
* `RecurringTypies` *(ควรสะกด RecurringTypes)*
* `RecurringTypeByChannels` (ข้อจำกัดกติกา per channel)
* `RetryPaymentByChannels` (กติกา retry ต่อ channel)
* `RunningNumbers` *(ถ้าใช้ร่วม—แนะนำค่อยๆ แยกเป็นบริการเลขรันภายหลัง)*

**เหตุการณ์โดเมน (ตัวอย่าง)**

* `RecurringRequestCreated`
* `InvoicePrepared` / `ChargePlanned`
* `PaymentAttempted`, `PaymentSucceeded`, `PaymentFailed`
* `RetryScheduled`, `StatusChanged`

## 2) Payer & Mandate (Consent / Payment Method)

**หน้าที่**: จัดเก็บผู้ชำระ วิธีชำระ token/alias และการยินยอม (mandate) อย่างปลอดภัย
**ตาราง**

* `Payers`
* `PaymentMethods`
* `Consents`
* `PaymentMethodConsents`
* `TitleNames` *(อ้างอิงชื่อคำนำหน้า)*

**หมายเหตุ**: ต้องหลีกเลี่ยงการเก็บข้อมูลบัตรดิบ ใช้ token/alias จาก PG/Token Vault แทน

## 3) Billing / Invoicing

**หน้าที่**: สร้างใบแจ้งหนี้/ยอดที่จะเรียกเก็บ, ออกเลขอ้างอิง, เชื่อมโยง policy
**ตาราง**

* `Invoices`
* *(อาจใช้ `RunningNumbers` ชั่วคราวสำหรับเลขที่เอกสาร)*

## 4) Policy (Insurance)

**หน้าที่**: แหล่งข้อมูลกรมธรรม์/เบี้ย/งวด—ระบบ recurring เป็น downstream ใช้เพื่อคำนวณยอดหรือตรวจสิทธิ
**ตาราง**

* `Policies`
* `PolicyHistories`

## 5) Channel & Product Catalog

**หน้าที่**: นิยามช่องทาง (Web, App, Counter, BPSS ฯลฯ) และสินค้าที่เปิด recurring + รูปแบบที่รองรับ
**ตาราง**

* `Channels`
* `ProductSetupByChannels`
* (เชื่อมกับ `RecurringTypeByChannels`, `RecurringTypes` ใน Orchestrator)

## 6) Payment Gateway Connector

**หน้าที่**: ผนวกรูปแบบและภาษาของ PG ภายนอกผ่าน Anti-Corruption Layer, บันทึก log ทุกครั้ง
**ตาราง**

* `PaymentGatewayTransactionLogs`

## 7) Notification

**หน้าที่**: รับ event แล้วส่งข้อความตาม template
**ตาราง**

* `MessageTemplates`

## 8) IAM / Admin (Generic)

**หน้าที่**: จัดการพนักงาน/บทบาท/เมนู (ไม่ใช่โดเมนธุรกิจหลัก)
**ตาราง**

* `Employees`, `Roles`, `EmployeeRoles`, `Menus`, `RoleMenus`

---

# ความสัมพันธ์ระหว่าง Contexts (Up/Downstream)

* **Policy → Recurring Orchestrator**: *Upstream Supplier*. Recurring อ่านข้อมูลกรมธรรม์/งวด/ยอด (แนะนำใช้ **Published Language** + อาจมี **ACL** แปลงคำ)
* **Payer & Mandate → Recurring Orchestrator**: *Upstream Supplier*. ให้สิทธิ์ตัดและข้อมูลวิธีชำระ (token/consent)
* **Channel & Product → Recurring Orchestrator**: กำหนดว่า policy/product นี้ตัดผ่านช่องทาง/ประเภทใดได้
* **Recurring Orchestrator → Billing**: สั่งสร้างใบแจ้งหนี้ตามรอบ
* **Recurring Orchestrator → Payment Gateway Connector**: ขอตัดยอดจริง (Connector ทำงานกับ PG ภายนอกและส่งผลกลับ)
* **Recurring Orchestrator → Notification**: โยนอีเวนต์ให้ส่งแจ้งเตือน
* **IAM** ให้สิทธิ์การใช้งานกับแอปของทุก context (generic)

---

# ขนาดบริการที่แนะนำ (MVP → Scale)

เริ่มต้นให้ “แยกตามขอบเขตข้อมูลและการเปลี่ยนแปลง” ก่อน แล้วค่อยแยกฐานข้อมูลจริงเมื่อเสถียร

**MVP 4–5 services**

1. `recurring-orchestrator-service` (Core)
2. `payer-mandate-service` (Supporting)
3. `billing-invoice-service` (Supporting)
4. `pg-connector-service` (Supporting, ACL)
5. `notification-service` (Generic; จะรวมกับระบบแจ้งเตือนเดิมก็ได้)

**ภายหลังค่อยแยก/เชื่อม**

* `policy-service` (ถ้ามีอยู่แล้ว ใช้เป็น Upstream)
* `catalog-channel-service`
* `iam-service`

 
