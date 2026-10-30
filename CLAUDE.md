# CLAUDE.md — RCG Payroll (admin web app)

เจ้าของระบบ: Mon (HR/Owner ของ ReadyCheckGo) · รายละเอียดเต็มและงานค้างอยู่ใน `HANDOFF.md`

## กฎการทำงาน (ต้องทำตามทุกครั้ง)

1. **เสนอแผนและรอ Mon confirm ก่อนแก้โค้ดทุกครั้ง** ห้ามลงมือเองก่อน
2. ทำงานจากไฟล์ล่าสุดใน repo เท่านั้น ห้ามเดาจากความจำ (อ่านโค้ดจริงก่อนเสนอ)
3. admin web app เป็น**ไฟล์เดียว (`index.html`) โดยตั้งใจ** ห้ามแตกเป็นหลายไฟล์ถ้า Mon ไม่ได้สั่ง
4. บั๊กที่เกี่ยวกับข้อมูลจริง: วินิจฉัยด้วย console snippet แบบ**อ่านอย่างเดียว**ให้ Mon รันใน browser ให้เห็นหลักฐานก่อน แล้วค่อยแก้
5. ทดสอบ logic ด้วยตัวเลขจริงจาก screenshot ของ Mon ก่อนส่งงาน (ไม่มี test อัตโนมัติ ให้ใช้ script node แยกพร้อม mock data)
6. Mon ใช้ทั้งไทยและอังกฤษ ตอบไทยเมื่อเธอเขียนไทย
7. Claude ทำ deploy, ตั้ง env var บน Railway หรือดูแล Firebase billing ไม่ได้ ส่วนนี้ Mon ทำเอง
8. **สร้าง Pull Request เข้า `main` ให้ทุกครั้ง**หลัง push งานที่แก้โค้ด แล้วส่งลิงก์ให้ Mon กด Merge เอง (Claude ไม่ merge เอง) PR ที่ merge แล้วห้ามใช้ซ้ำ งานใหม่ให้เริ่ม branch จาก `main` ล่าสุดแล้วเปิด PR ใหม่ · คำอธิบาย PR เขียนเป็นภาษาไทยแบบเข้าใจง่าย บอกว่าเปลี่ยนอะไรและต้องทำอะไรหลัง merge

## โปรเจกต์ที่เกี่ยวข้อง

| Repo | คืออะไร | Deploy |
|---|---|---|
| `rcg-payroll` (repo นี้) | Admin web app ไฟล์เดียว `index.html` (~4,000 บรรทัด) โหลด SheetJS และ Chart.js จาก CDN | GitHub Pages: `monzai69.github.io/rcg-payroll` |
| `rcg-payroll-line` | Node.js server (LINE bot, LIFF API, PDF ด้วย puppeteer, cron) และ `index.html` ที่เป็น **LIFF Staff Portal** (คนละไฟล์กับ admin app) | Railway |
| slip bot tracker | โปรเจกต์ที่สาม ยังไม่เคยเห็นโค้ด | ถามและตรวจก่อนแตะ |

Database: Firebase Firestore (collection `config`)

## สถาปัตยกรรม

### ค้นหาส่วนต่าง ๆ ใน index.html
ส่วนต่าง ๆ คั่นด้วย banner `/* ══════════════════ NAME ══════════════════ */` เช่น `DASHBOARD`, `PAYROLL CALCULATION ENGINE`, `REPORTS PAGE`, `BRANCH SETTINGS`, `NAVIGATION` ให้ grep `══════` เพื่อดูแผนที่ไฟล์

### Global state และการเซฟ
- `S` staff, `P` payroll, `C` company config, `A` advances, `AP` anytime/doctor payments, `B` branches, `LVR` leaves
- แต่ละตัวเซฟผ่าน `svS/svP/svC/svA/svAP/svB/svLV` → `cloudSave` → Firestore `config/<name>` เป็น `{data: JSON string, updatedAt}`
- `svP` debounce 1.2 วินาที, Save All / Save & Copy ใช้ `flushP` เขียนทันที
- **Conflict guard** ของ payroll: ไม่เซฟถ้า `updatedAt` บน cloud ไม่ตรงกับ `window._payStamp` (มีแท็บหรืออุปกรณ์อื่นเซฟไปแล้ว)

### ข้อมูลเงินเดือน (`config/payroll`)
- JSON ก้อนเดียว key แบบ `YYYY_M` (เช่น `2026_10`) ข้างในเป็น entry รายพนักงาน
- input ของ entry: `rev`, `bonus`, `otHrs`, `leaveDays`, `leaveHrs`, `dOth`, `periodic{itemId: ค่า}`, `diligence{}`
- `snap`: snapshot ของ base/rates/contract/items ที่แช่ไว้ตอน Save All
- `net`, `payRows`: ผลคำนวณที่ cache ไว้ (`payRows` ใช้โดย slip PDF และหน้า LIFF)
- `P[key]._meta.unlocked`: ปลดล็อกงวด (งวดล็อกอัตโนมัติวันที่ 2 ของเดือนถัดไป) · `P[key]._paid`: ติ๊กจ่ายแล้วบน Dashboard

### การคำนวณ
- **`calcM(s, e, key)` เป็นที่เดียวที่คำนวณเงินเดือน** (ภาษี ภงด.1, SSO, WHT, commission, หักลา) ห้ามเขียนสูตรซ้ำที่อื่น ฝั่ง server ใช้ `payRows` ที่ cache ไว้
- ถ้ามี `e.snap` calcM ใช้ค่าจาก snapshot ไม่ใช้โปรไฟล์ปัจจุบัน Save All และ Save & Copy สร้าง snapshot ใหม่ทุกครั้ง

### Publish / confirmation (`config/publish_meta`)
- **server เขียนฝั่งเดียว web app อ่านอย่างเดียว** (โหลดเข้า `window.PUB`)
- รูปแบบ: `{ "2026_10": { published, publishedAt, unpublishedAt?, reminderAt, reminderSent, autoConfirmAt, autoConfirmDone, confirmations: { staffId: { confirmed, confirmedAt, autoConfirmed? } } } }`
- ข้อมูลเก่าที่ `payroll[key]._meta` ยังมี fallback อ่านได้
- Server endpoints: `GET /api/salary/:staffId/:year/:month`, `POST /api/publish-salary`, `POST /api/unpublish-salary`, `POST /api/confirm-salary`, cron ทุก 10 นาที (Asia/Bangkok) เตือนที่ 2 ชม. และ auto-confirm ที่ 5 ชม.

### แบ่งต้นทุนสำนักงานใหญ่ (Dashboard)
- สาขาที่ติ๊ก `backoffice` ใน `config/branches` (พร้อม `allocMode: 'equal'|'custom'`, `allocPct{brId: %}`) จะถูกแบ่งยอด net ให้สาขาปฏิบัติการ (สาขาที่มีพนักงานในงวดนั้น) ด้วย `calcBranchAlloc()` ใต้ banner DASHBOARD
- `config/alloc_hist` (**web app เขียนฝั่งเดียว** ผ่าน `freezeAllocDue()` เท่านั้น): `{ "2026_10": { frozenAt, staffBranch{sid: brId}, alloc: { frozenAt, settings[], opIds[] } | null } }` แช่ไว้เมื่องวดเลยวันที่ 2 ของเดือนถัดไป และ**ไม่เขียนทับของเดิม** งวดที่ยังเปิดใช้การตั้งค่าปัจจุบัน
- Payroll Summary บน Dashboard: งวดที่ล็อกนับทุกคนที่มี entry (ลาออกแล้วก็นับ) ใช้สาขาจาก `staffBranch` · งวดที่เปิดนับ Active + คนที่มี entry · ไม่รวม part-time/หมอ

## บทเรียน (กันบั๊กซ้ำ)

1. **ห้ามให้สองฝั่งเขียนเอกสารเดียวกัน** (เหตุการณ์ 29 ต.ค. 2026 สองแท็บเขียนทับกัน) เอกสารใหม่ต้องกำหนดชัดว่าใครเป็นคนเขียน
2. **Item ID ต้องคงที่** ค่า periodic/comm_flat_custom ผูกกับ `item.id` (`data-item-id`) ถ้า ID เปลี่ยน ข้อมูลจะหาไม่เจอ
3. **Snapshot บังโปรไฟล์**: แก้ Staff หลัง Save All แล้ว การ์ดจะยังไม่เปลี่ยนจนกว่าจะกด Save All หรือ Save & Copy ใหม่
4. `gatherE()` สร้าง entry ใหม่จาก input เท่านั้น จึงทิ้ง `payRows` เก่าไป ต้องกด Save & Copy หรือ Save All ให้ LIFF และ PDF ได้รายการเต็ม
5. การ์ดแสดงแถวหักตาม `s.ct` ของโปรไฟล์ แต่ calcM ใช้ `snap.ct` ถ้าสองค่าต่างกันเลขจะไม่ตรงกับแถวที่แสดง
6. Labor cost % ใน Reports เป็นค่าประมาณ (gross ÷ revenue ที่กรอกรายคน)
7. Dashboard ใช้ **net pay**, หน้า Reports ใช้ **gross**
8. ตอนแก้ฟอร์มที่สร้าง object ใหม่ทั้งก้อน (เช่น Save Branch สร้าง `nb` ใหม่) ระวังฟิลด์อื่นหาย ให้ merge กับ object เดิม
