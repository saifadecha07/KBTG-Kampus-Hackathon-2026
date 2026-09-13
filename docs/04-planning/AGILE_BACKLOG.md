# Pace — Agile Backlog

> Backlog สำหรับนำงาน Pace เข้า GitHub Project/Issues โดยใช้สถานะ `Backlog` / `Ready` / `In Progress` / `Review` / `Done` / `Blocked`
> แหล่งอ้างอิง business rule คือ [`../03-design/design_usecase.md`](../03-design/design_usecase.md) และ [`../03-design/decision_log.md`](../03-design/decision_log.md)
>
> อัปเดตล่าสุด: 2026-09-13

## Working Agreement

### Definition of Ready

งานจะย้ายจาก `Backlog` เป็น `Ready` เมื่อมี:

- ขอบเขตชัดเจนและไม่ขัดกับ decision log
- Acceptance criteria ตรวจได้
- Dependency ถูกระบุแล้ว
- มีข้อมูลพอให้คนหนึ่งคนเริ่มทำได้โดยไม่ต้องเดา business rule

### Definition of Done

งานจะเป็น `Done` เมื่อ:

- โค้ด/เอกสารเสร็จตาม acceptance criteria
- มี test ที่เหมาะกับความเสี่ยง
- ไม่ทำลาย M1–M12, L10–L14 และ warnings ใน decision log
- ผ่าน review และไม่มี secret/ข้อมูลจริงใน repo
- อัปเดต traceability หรือ board ถ้ากระทบ workflow

### Priority

- `P0` ต้องทำก่อน demo หรือก่อนงานอื่นเริ่มได้
- `P1` จำเป็นต่อ flow หลักหรือหลักฐาน Track 1
- `P2` เพิ่มความสมบูรณ์ แต่ตัดได้ถ้าเวลาไม่พอ

## Epics and Stories

### EPIC-01 — Architecture Foundation

| ID | Priority | Status | Story | Acceptance criteria | Dependency |
|---|---|---|---|---|---|
| ARC-01 | P0 | Backlog | ในฐานะทีมพัฒนา เราต้องเลือก tech stack ที่ทีมถนัด เพื่อเริ่ม implement ได้ | ระบุ frontend, backend, database, test runner และเหตุผลสั้น ๆ | ไม่มี |
| ARC-02 | P0 | Backlog | ในฐานะ backend เราต้องมี schema ตาม scope J2 เพื่อเก็บสถานะเงินครบ | มี `Pocket`, `LedgerEntry`, `IncomeTransaction`, `WithdrawRequest`, `SplitRule`, `DailyTargetConfig`, `OverspendLog`, `Bill`, `SenderRule`, `AuditLog`, `IdempotencyKey`; เงินเป็น integer satang | ARC-01 |
| ARC-03 | P0 | Backlog | ในฐานะ frontend/backend เราต้องมี API contract กลาง | ครอบคลุม mock income, pockets, mock payment, withdraw/cancel, split undo, bills และ stats; ระบุ error response | ARC-01, ARC-02 |
| ARC-04 | P0 | Backlog | ในฐานะทีม demo เราต้องจำลองเวลาได้โดยไม่แก้เวลาระบบจริง | มี server-controlled demo clock/config สำหรับข้ามวันและเร่ง cooldown; production path ยังใช้ server time | ARC-01 |

### EPIC-02 — Money Ledger and Rules

| ID | Priority | Status | Story | Acceptance criteria | Dependency |
|---|---|---|---|---|---|
| CORE-01 | P0 | Backlog | ในฐานะระบบ เราต้อง split เงินโดยรักษา invariant | split รวมยอดตรง, เศษเข้าบัญชีรายเดือน, ไม่มีกระเป๋าติดลบ, ledger สองขาใน transaction เดียว | ARC-02 |
| CORE-02 | P0 | Backlog | ในฐานะระบบ เราต้องเติมกระเป๋าใช้จ่ายรายวันและ backfill ได้ถูกต้อง | เติมเฉพาะส่วนต่าง, carry-over ไม่หาย, backfill atomic, เงินรายเดือนหมดแล้วไม่ crash | CORE-01 |
| CORE-03 | P0 | Backlog | ในฐานะระบบ เราต้องรองรับ idempotency และ concurrency | retry ไม่สร้างรายการซ้ำ, cron ซ้ำไม่เติมซ้ำ, row lock ลำดับคงที่, multi-device ไม่สร้าง request ซ้ำ | CORE-01 |
| CORE-04 | P0 | Backlog | ในฐานะระบบ เราต้องรองรับ Fail-open ตอน Pace ล่ม | จ่าย/โอนจากบัญชีหลักต่อได้, หยุด split/backfill/vault withdrawal, บันทึก `outside_pace`, reconcile ภายหลัง | ARC-02 |
| CORE-05 | P1 | Backlog | ในฐานะระบบ เราต้องจัดการเงินตีกลับหลัง split | reverse เงินที่เหลือทุกกระเป๋าโดยไม่รอ cooldown, ไม่ทำให้ pocket ติดลบ, บันทึก `reversal_shortfall` เพื่อส่ง dispute ธนาคาร | CORE-01, ARC-02 |

### EPIC-03 — Bill Learner and Reservation

| ID | Priority | Status | Story | Acceptance criteria | Dependency |
|---|---|---|---|---|---|
| BILL-01 | P0 | Backlog | ในฐานะระบบ เราต้องเรียนรู้บิลจากประวัติ 6 เดือนตาม B1–B5 | แยก fixed/variable, วันประจำ, cross-month และสถานะ `active`/`late`/`stopped` ได้ตาม traceability | ARC-02 |
| BILL-02 | P0 | Backlog | ในฐานะผู้ใช้ เราต้องยืนยัน candidate bill ก่อนระบบกันเงิน | candidate ที่ยังไม่ยืนยันไม่ถูก reserve; ผู้รับเดียวกันหลายบิลแยกด้วยยอด+วันที่และให้แก้/รวมได้ | BILL-01 |
| BILL-03 | P0 | Backlog | ในฐานะผู้ใช้ เราต้องกำหนด Priority ของบิลเอง | priority ใช้ก่อน due date; ถ้าไม่ตั้งใช้ due date ใกล้สุด; เงินไม่พอสำหรับบิลแรกให้ถามผู้ใช้ก่อน | BILL-02 |
| BILL-04 | P1 | Backlog | ในฐานะผู้ใช้ เราต้องเพิ่มบิลรายไตรมาส/รายปีเอง | ค่าเริ่มต้นคือทยอยกันเฉลี่ย, แสดงยอดต่อรอบ+ยอดสะสมก่อนยืนยัน, สลับเป็นกันเต็มก้อนได้ก่อนยืนยันหรือแก้ทีหลัง; รายปีตรวจเจอเองไม่ได้แต่เพิ่มเองได้ | BILL-02 |
| BILL-05 | P0 | Backlog | ในฐานะระบบ เราต้อง escalate เงินเข้าที่ค้างเฉพาะเมื่อมี deadline จริง | pending item มี nullable `due_date`, ไม่มี `cycle_id`; `null` อยู่ passive queue; `ESCALATION_LEAD_DAYS=5`; เตือนวันละครั้ง 3 ครั้งแล้ว auto-reserve แบบ recompute/idempotent แต่ไม่ split | BILL-02, CORE-01 |
| BILL-06 | P0 | Backlog | ในฐานะระบบ เราต้องรักษา `balance`, `reserved`, `pending_unsplit`, `available` ให้สื่อความจริง | `available = balance - reserved - pending_unsplit`; underfunded ถูกแสดงชัด; classification ทุกครั้งมี ledger/audit; เงินเข้ารอบถัดไปจัดการตาม priority | BILL-03, CORE-01 |

### EPIC-04 — Safety Vault and Income Queue

| ID | Priority | Status | Story | Acceptance criteria | Dependency |
|---|---|---|---|---|---|
| VAULT-01 | P0 | Backlog | ในฐานะผู้ใช้ เราต้องขอถอนและยกเลิกได้ตาม cooldown | server เป็นผู้ตัดสินเวลา, merge เพิ่มยอดแล้ว reset timer, ลดจำนวนไม่ reset, notification fail ไม่ขวาง release | ARC-02, CORE-03 |
| VAULT-02 | P0 | Backlog | ในฐานะระบบ เราต้องแยกการปิด Vault กับปิดธนาคารทั้งบัญชี | ปิด Vault ยังรอ cooldown; ปิดบัญชีธนาคารคืนเงินตาม bank process; cooldown read-only หลังเปิด | VAULT-01 |
| INCOME-01 | P0 | Backlog | ในฐานะผู้ใช้ เราต้องจัดการเงินเข้าหลายก้อนได้โดยไม่เกิด split ซ้ำ | แยกตาม transaction ID, queue รวมใน UI เดียว, แต่ละก้อนมีสถานะ/ยอดของตัวเอง, ไม่มี daily reminder ถ้าไม่มี deadline | CORE-01 |

### EPIC-05 — Prototype and Demo

| ID | Priority | Status | Story | Acceptance criteria | Dependency |
|---|---|---|---|---|---|
| UI-01 | P0 | Backlog | ในฐานะผู้ใช้ เราต้องเห็นยอดเงินที่สำคัญในหน้าหลัก | แสดงบัญชีหลัก balance/reserved/available, Monthly, Daily, Vault, target และ overspend | ARC-03, CORE-02 |
| UI-02 | P0 | Backlog | ในฐานะผู้ใช้ เราต้องยืนยันเงินเข้าและเห็นบิลก่อน split | แสดง bill reserve, split preview, sender rule, target preview และ pending queue | ARC-03, BILL-03 |
| UI-03 | P1 | Backlog | ในฐานะผู้ใช้ เราต้องจัดการบิลเอง | เพิ่ม/แก้/ลบบิล, priority, source, status, full/gradual funding choice | BILL-04 |
| UI-04 | P0 | Backlog | ในฐานะผู้ใช้ เราต้องขอถอนและเห็นสถานะ cooldown | countdown จาก server, cancel, pending amount แยกจาก total, release state | ARC-03, VAULT-01 |
| DEMO-01 | P0 | Backlog | ในฐานะทีม pitch เราต้องสาธิต flow หลักได้ในไม่กี่นาที | ปุ่มจำลอง income/payment, ข้ามวัน, เร่ง cooldown, แสดง ledger/audit outcome | ARC-04, UI-01, UI-04 |

### EPIC-06 — Quality Evidence

| ID | Priority | Status | Story | Acceptance criteria | Dependency |
|---|---|---|---|---|---|
| TEST-01 | P0 | Backlog | ในฐานะทีม Track 1 เราต้องพิสูจน์ money invariant | property test สุ่มอย่างน้อย 10,000 operations; total balance ตรงและไม่มี pocket ติดลบ | CORE-01, CORE-02 |
| TEST-02 | P0 | Backlog | ในฐานะทีม เราต้องพิสูจน์ flow สำคัญแบบ integration | split, top-up, backfill, cooldown, cancel, release, underfunded, reversal shortfall | CORE-02, CORE-05, VAULT-01, BILL-06 |
| TEST-03 | P0 | Backlog | ในฐานะทีม เราต้องพิสูจน์ no-bypass/idempotency/concurrency | retry, duplicate event, cron repeat, multi-device, client clock tampering | CORE-03, VAULT-01 |
| TEST-04 | P1 | Backlog | ในฐานะทีม เราต้องทำ usability pilot ก่อนการทดสอบเต็ม | pilot 1 คน, แก้ critical issue, แล้วจึงรัน 5–8 คนตาม protocol | UI-01, UI-02, UI-04 |

## Suggested Sprint Flow

ไม่กำหนดวันตายตัวจนกว่าจะรู้ deadline จริง ให้ใช้เป้าหมายแทน:

| Sprint | Goal | Stories ที่ควรดึงเข้า sprint |
|---|---|---|
| Sprint 0 — Architecture | ปิดสิ่งที่ทำให้เริ่มโค้ดได้ | ARC-01 ถึง ARC-04 |
| Sprint 1 — Money Core | ทำ ledger/rule engine ที่ไม่ทำยอดเพี้ยน | CORE-01 ถึง CORE-05 |
| Sprint 2 — Bills and Vault | ทำ bill reservation, pending escalation และ cooldown | BILL-01 ถึง BILL-06, VAULT-01 ถึง VAULT-02, INCOME-01 |
| Sprint 3 — Demo Surface | ทำหน้าจอและ demo panel ครบ flow | UI-01 ถึง UI-04, DEMO-01 |
| Sprint 4 — Quality Gate | เก็บหลักฐาน Track 1 และแก้ critical issues | TEST-01 ถึง TEST-04 |

## GitHub Issue Mapping

ใช้ชื่อ issue ตามรูปแบบนี้เพื่อค้นง่าย:

```text
[ARC-01] Decide tech stack
[CORE-01] Implement money ledger invariant
[BILL-05] Add deadline-scoped pending escalation
[UI-02] Build income split confirmation
[TEST-01] Add property test for money invariant
```

ใช้ labels:

- `epic:architecture`, `epic:core`, `epic:bill`, `epic:vault`, `epic:ui`, `epic:quality`
- `priority:p0`, `priority:p1`, `priority:p2`
- `type:feature`, `type:bug`, `type:test`, `type:docs`
- `blocked`, `needs-decision`, `ready`, `review`

## Current Blockers

- `ARC-01`: ยังไม่เลือก tech stack
- `ARC-03`: รอ schema และ tech stack
- `ARC-04`: ต้องออกแบบ demo time model
- `BILL-05`: business rule ปิดแล้ว แต่ต้องทำ reminder engine ตาม `ESCALATION_LEAD_DAYS`
- **Deadline จริง: 1-Page Pitch ต้องส่งภายใน 21 ก.ย. 2026 (ปิดรับ Applications) — เหลือไม่กี่วัน** ดูตาราง `BOARD.md` § กำหนดการ; Final Slide Submission 30 ต.ค., Pitching Day 7 พ.ย. 2026
- ทีม 3 คน: Pornchanok Hongthong (Business Analyst — ฝั่ง business ของทั้งงาน) · Saifa Decha และ Wutthisak Boonkan (Software Engineer — sprint นี้ทำ logic/algorithm design) — owner ของแต่ละ story ในตารางด้านบนยังไม่ระบุ
