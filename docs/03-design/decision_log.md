# Pace — Decision Log

> บันทึกการตัดสินใจด้าน business logic ล่าสุด เพื่อใช้เป็นแหล่งอ้างอิงก่อน implement
> Core rules เดิมยังอยู่ใน [`design_usecase.md`](design_usecase.md) เอกสารนี้บันทึกเฉพาะการตัดสินใจที่ปิดเพิ่มและคำเตือนที่ต้องไม่หลุดตอนเขียนโค้ด
>
> อัปเดตล่าสุด: 2026-09-13

## 1. Resilience ตอน Pace ล่ม

### Decision R1 — Fail-open

เมื่อ Pace ล่มหรือเชื่อมต่อไม่ได้:

- ธุรกรรมจ่ายเงิน/โอนออกจากบัญชีหลักยังทำต่อได้
- ห้าม Auto-Split, Backfill และการถอนจาก Safety Vault ระหว่างระบบล่ม
- รายการที่เกิดขึ้นช่วงล่มถือเป็น `outside_pace`
- เมื่อระบบกลับมา ให้ reconcile ยอดบัญชีหลักและ `reserved` กับข้อมูลธนาคาร
- ห้ามย้อนสร้าง Daily Target หรือ Backfill จากรายการช่วงล่ม

### Decision R2 — เงินกันบิลถูกใช้ระหว่างระบบล่ม

- อนุญาตให้ธุรกรรมใช้เงินที่ถูก `reserved` ได้ตาม Fail-open
- ถ้า `balance < reserved` ให้ทำเครื่องหมายบิลเป็น `underfunded`
- เงินเข้ารอบถัดไปต้องกันยอดที่ขาดก่อน Auto-Split
- หากมีหลายบิล ให้เรียงตาม Priority ที่ผู้ใช้ตั้งไว้ก่อน
- ถ้า Priority เท่ากัน ให้เรียงตามวันครบกำหนดใกล้ที่สุด
- หากบิลแรกมีเงินไม่พอ ให้ถามผู้ใช้ก่อนว่าจะกันบางส่วนหรือข้ามไปบิลถัดไป

## 2. เงินเข้าถูกตีกลับหลัง Auto-Split

### Decision R3 — Automatic reversal

เมื่อธนาคารตีกลับเงินเข้า หลังจาก Pace แบ่งเงินไปแล้ว:

- Reverse อัตโนมัติทันที
- ดึงเงินที่ยังเหลือจากทุกกระเป๋าที่เกี่ยวข้อง รวมถึง Safety Vault โดยไม่รอ cooldown
- ห้ามทำให้กระเป๋าใดติดลบ
- ถ้าเงินที่เหลือไม่พอ ให้บันทึกยอดขาดเป็น `reversal_shortfall`
- ยอดขาดไม่ถูกสร้างเป็นหนี้และไม่ถูกหักจากเงินเข้าครั้งถัดไปโดย Pace
- ส่งต่อเป็นกรณี dispute ของธนาคาร พร้อม ledger และ audit log

## 3. Priority และการเรียนรู้บิล

### Decision R4 — ผู้ใช้เป็นผู้กำหนด Priority

- ผู้ใช้กำหนดลำดับความสำคัญของบิลเอง
- ถ้าไม่ได้กำหนด ให้ใช้วันครบกำหนดใกล้ที่สุดเป็นค่าเริ่มต้น
- Priority ที่ผู้ใช้ตั้งมีผลกับรอบถัดไป
- เมื่อเงินไม่พอสำหรับบิล Priority สูงสุด ระบบไม่ตัดสินใจแทน แต่ถามผู้ใช้ว่าจะกันบางส่วนหรือข้ามบิล
- การเลือกแบบถามผู้ใช้มีผลเฉพาะรอบนั้น เว้นแต่ผู้ใช้บันทึกเป็นกฎถาวร

### Decision R5 — แยกบิลผู้รับเดียวกัน

- ใช้รูปแบบยอดเงินร่วมกับวันที่จ่ายเพื่อหา candidate bills
- เมื่อพบความเป็นไปได้ว่าผู้รับเดียวกันมีหลายบิล ให้ผู้ใช้ยืนยันก่อน
- ผู้ใช้แก้ชื่อ ยอด วันที่ หรือรวมกลุ่มกลับได้
- กลุ่มที่ยังไม่ยืนยันจะยังไม่ถูกนำไปกันเงิน
- ห้ามรวมยอดอัตโนมัติจน transaction เดิมหายหรือแยกความหมายไม่ได้

## 4. บิลรายเดือน/รายไตรมาส/รายปี

### Decision R6 — ผู้ใช้เลือกวิธีกันเงิน

- ผู้ใช้เลือกเป็นรายบิลว่าจะกันเต็มก้อนหรือทยอยกัน
- ต้องแสดงยอดที่จะกันต่อรอบและยอดสะสมก่อนยืนยัน
- ห้ามใช้ค่าเริ่มต้นอัตโนมัติสำหรับบิลรอบไม่รายเดือน
- ถ้ายังไม่เลือก ระบบยังไม่บันทึกบิลและยังไม่กันเงิน
- การเปลี่ยนวิธีมีผลกับรอบถัดไป ไม่ย้อนหลัง

## 5. Safety Vault และ Daily Target

### Decision R7 — ปิด Safety Vault กับปิดบัญชีธนาคารเป็นคนละกรณี

- ปิดเฉพาะ Safety Vault: เงินยังต้องรอ cooldown ตาม policy เดิม
- ปิดบัญชีธนาคารทั้งบัญชี: คืนเงินตามกระบวนการปิดบัญชีของธนาคารโดยไม่บังคับรอ cooldown ของ Pace
- ปิด Auto-Split หรือ Daily Refill: ปิดได้ทันทีและไม่กระทบเงินที่ล็อกอยู่

### Decision R8 — แก้ Daily Target กลางรอบ

- ผู้ใช้แก้เป้ารายวันได้
- ค่าใหม่เริ่มใช้ตั้งแต่วันถัดไป
- ไม่แก้ยอดย้อนหลังและไม่ดึงเงินสะสมคืน
- ถ้าเพิ่มเป้าสูงกว่าความสามารถของบัญชีรายเดือน ต้องยืนยันคำเตือนก่อน
- ระบบเติมได้เท่าที่มีและแจ้งยอดที่เติมจริง
- ห้ามดึงเงินจาก Safety Vault อัตโนมัติเพื่อให้ถึงเป้า

## 6. เงินเข้าหลายก้อน

### Decision R9 — แยก transaction และรวมรายการค้างใน UI

- เงินเข้าแต่ละก้อนแยกตามธนาคาร transaction ID
- ไม่รวมยอดและไม่ re-split เงินก้อนเดิม
- รายการที่ยังรอจัดการแสดงรวมในหน้าหรือ dialog เดียว
- ผู้ใช้จัดการทีละก้อนได้ โดยแต่ละก้อนยังมีสถานะและยอดของตัวเอง
- ไม่ส่ง reminder รายวันเป็นค่าเริ่มต้น

## 7. ขอบเขต PoC และ Data Model

### Decision R10 — Payment scope ของ PoC

- รองรับ mock โอนเงิน, สแกนจ่าย และจ่ายบิลที่ผ่าน flow ของ Pace
- ATM และบัตรเดบิตอยู่นอกขอบเขต PoC
- รายการ ATM/บัตรต้องถูกแสดงเป็น `outside_pace` ใน roadmap/architecture เท่านั้น
- ห้ามอ้างว่า PoC ควบคุมการจ่ายเงินได้ทุกช่องทาง

### Decision R11 — Data Model scope J2

PoC จะออกแบบข้อมูลครบตาม logic ที่ตัดสินแล้ว ได้แก่:

- `Pocket`
- `LedgerEntry`
- `IncomeTransaction` (pending state, nullable `due_date`, no `cycle_id`)
- `WithdrawRequest`
- `SplitRule`
- `DailyTargetConfig`
- `OverspendLog`
- `Bill`
- `SenderRule`
- `AuditLog`
- `IdempotencyKey`

Ledger เป็นแหล่งความจริงหลัก ส่วน `balance` เป็น cache ที่ต้อง reconcile ย้อนกลับได้ตาม M1–M12

## Warnings

### W1 — `reserved` อาจต่ำกว่ายอดที่ต้องจ่าย

Fail-open ช่วยไม่ให้ Pace ขวางการจ่ายเงินจริง แต่ทำให้ `balance < reserved` เกิดได้ ต้องมีสถานะ `underfunded` และห้ามแสดง available เป็นยอดใช้ได้ปกติโดยไม่แจ้งผู้ใช้

### W2 — `reversal_shortfall` อยู่นอกขอบเขต Pace

ห้ามนำยอดขาดจากเงินตีกลับไปทำเป็นยอดติดลบในกระเป๋าหรือหนี้อัตโนมัติ เพราะข้อสรุปคือให้ธนาคารจัดการ dispute ต้องมี audit trail แยกชัดเจน

### W3 — กลุ่มบิลที่ยังไม่ยืนยันห้ามถูกกันเงิน

การใช้ heuristic ผิดอาจกันเงินผิดบิลได้ จึงต้องแยกสถานะ candidate/unconfirmed ออกจาก active bill และห้ามให้ bill learner ข้ามขั้นยืนยัน

### W4 — การถามผู้ใช้เมื่อเงินไม่พอเป็นจุดที่ flow ค้างได้

ต้องมีสถานะรอการตัดสินใจและป้องกันการสร้างการกันเงินซ้ำ หากผู้ใช้ปิดหน้าจอหรือมีเงินก้อนใหม่เข้าระหว่างรอ

### W5 — ATM/บัตรไม่ใช่ Daily Pocket ใน PoC

ห้ามใช้ข้อความว่า Daily Pocket ผูกกับทุกช่องทางการจ่าย เพราะรายการ ATM/บัตรอาจหักจากบัญชีหลักโดยตรงและต้อง reconcile ภายหลัง

### W6 — J2 เพิ่ม blast radius ของ schema

การออกแบบครบทุก entity ไม่ได้แปลว่าต้องทำทุก feature ใน demo พร้อมกัน ต้องใช้ migration แบบเล็ก, transaction เดียวสำหรับการเคลื่อนเงิน และทดสอบ ledger/invariant ก่อน UI polish

### W7 — ห้ามแก้ cooldown ตามหน้าตั้งค่า

Cooldown ตั้งครั้งเดียวตอนเปิด Safety Vault และ immutable หลังจากนั้น หน้าตั้งค่าต้องแสดงเป็น read-only ไม่ใช่ช่องแก้ไข

### W8 — H2.3 ต้องตรวจให้ตรงกับ G12

**Resolved:** H2.3 ใช้กับรายการเงินเข้าที่อยู่ใน pending queue แต่ไม่มี deadline ผูกอยู่ จึงไม่มี reminder อัตโนมัติและรอให้ผู้ใช้เปิดดูเอง ส่วน G12 ใช้เป็น escalation เฉพาะรายการที่ผูกกับบิลซึ่งมี `due_date` จริง

กฎ implement:

- Pending item ทุกก้อนแยกตาม transaction ID และมี `due_date` ที่เป็น nullable
- ถ้า `due_date == null` ให้อยู่ใน passive queue ตาม H2.3 โดยไม่มี timer หรือ reminder อัตโนมัติ
- ถ้า `due_date != null` ให้ reminder engine ตรวจ deadline และความเงียบของผู้ใช้ก่อนเข้า escalation
- เริ่ม escalation เมื่อ `today >= due_date - ESCALATION_LEAD_DAYS`
- เมื่อเข้า escalation ให้เตือนวันละครั้งสูงสุด 3 ครั้งตาม G12
- หลัง escalation ครบ ให้ auto-reserve เงินบิล แต่ยังไม่ split เข้า Pocket จนกว่าผู้ใช้จะยืนยัน
- `due_date` ของ pending item ที่ไม่มีบิลห้ามถูกสร้างจากค่าเดา เช่น วันเงินเดือนถัดไป

### Decision R12 — Escalation lead time

กำหนด config เป็น `ESCALATION_LEAD_DAYS` โดยมีค่าเริ่มต้น **5 วัน**:

- วันเริ่ม escalation = `due_date - 5 วัน`
- มี 3 วันสำหรับ reminder cycle ที่วัน `-5`, `-4`, `-3`
- เหลือ buffer 2 วัน ที่ `-2`, `-1` ให้ auto-reserve ทำงานและให้ผู้ใช้ยืนยันก่อนถึงกำหนด
- ห้ามตั้งค่าเริ่มต้นต่ำกว่า 5 วัน เพราะ threshold 3 วันจะทำให้ reminder ครั้งสุดท้ายชน due date และไม่มี buffer
- เก็บเป็น parameter เพื่อรองรับ lead time ต่างกันในอนาคตตามประเภทบิล โดยไม่ refactor escalation engine
- ค่า 7 วันเป็นทางเลือก conservative สำหรับประเภทบิลที่ auto-pay ตัดเงินเร็ว แต่ยังไม่เปิดใช้เป็นค่าเริ่มต้น

### Decision R13 — Pending income lifecycle

- Pending income ไม่ผูกกับ `cycle_id` และไม่มีวันหมดอายุ
- รายการยังอยู่ใน pending queue ข้ามรอบได้จนกว่าผู้ใช้จะจัดการเองหรือเกิด reversal
- บัญชีหลักต้องมี `pending_unsplit` แยกจาก `balance` และ `reserved`
- สูตรคือ `available = balance - reserved - pending_unsplit`
- Pending income ที่ยังไม่จัดการห้ามถูกนำไปรวมในการคำนวณรอบใหม่หรือ Daily Target

### Decision R14 — Idempotent auto-reserve and audit trail

- การ auto-reserve จาก G12 ใช้การ recompute ตาม B3 ไม่ใช่ `reserved += amount`
- idempotency key คือ `bill_id + due_date` เพื่อให้ escalation/cron รันซ้ำแล้วได้ผลเดิม
- การเปลี่ยน `reserved` หรือ `pending_unsplit` ต้องมี classification ledger entry แบบสองขาและ audit log ใน transaction เดียว แม้ `balance` จริงไม่เปลี่ยน
- ห้ามแก้ค่า counter ตรง ๆ โดยไม่มี ledger/audit record

### W9 — Pending income ต้องไม่ถูกนับซ้ำในรอบใหม่

ห้ามใช้ pending income เป็นฐานของ L1 หรือ Daily Target รอบใหม่ เพราะยอดนี้ถูกหักออกจาก `available` และยังไม่ได้รับการยืนยันให้ split หากรวมเข้าฐานอีกครั้งจะทำให้ผู้ใช้เห็นเงินใช้ได้เกินจริงหรือเกิดการจัดสรรซ้ำ

## สิ่งที่ยังไม่ปิด — ต้องถามก่อนออกแบบต่อ

- Tech stack จริงของทีม
- API Contract และชื่อ endpoint
- Schema field-level, enum และ database migration
- วิธีจำลองเวลาใน Demo panel
- รายละเอียด dispute workflow ของธนาคารหลัง `reversal_shortfall`
- รายละเอียด UX ของหน้าถาม Priority เมื่อเงินไม่พอ

รายการเหล่านี้ยังไม่ถูกตัดสินใน decision log นี้ และไม่ควรเดาเพิ่มโดยไม่ถามผู้ใช้
