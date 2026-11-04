# NYC Needs Survey V2 — ระบบสำรวจความต้องการใช้งานระบบสารสนเทศ

โปรแกรมสำหรับวิทยาลัยการอาชีพนครยะลา: ฟอร์มสำหรับครู บุคลากร นักเรียน นักศึกษา, แดชบอร์ดผู้ดูแล, จัดอันดับระบบที่ต้องการ, เปิด–ปิดแบบสำรวจ และเครื่องมือจัดเตรียมไฟล์นำเข้าผู้ใช้ A–W แบบประมวลผลในเบราว์เซอร์

## เหตุผลที่แยกสองส่วน

- **แบบสำรวจ** ไม่ต้องใช้เลขบัตรประชาชน ชื่อ วันเกิด หรืออีเมล เพื่อวิเคราะห์ความต้องการระบบ
- **ไฟล์ข้อมูล A–W** เป็นข้อมูลทะเบียนผู้ใช้ที่มีข้อมูลส่วนบุคคล จึงแยกเป็นเครื่องมือจัดเตรียมข้อมูลไฟล์และตรวจ CSV เฉพาะเบราว์เซอร์ ไม่ส่งเข้า Google Sheets แบบสำรวจ
- โครงสร้าง A–W ยึดชื่อคอลัมน์ 23 ช่องจากไฟล์ importUserV3_NoOfRecord.xlsx และโน้ตที่แนบมา โดยเว้นช่องรหัสหน่วยงาน/หลักสูตรที่ยังไม่มีข้อมูลจริง ไม่เดาแทน

## สถาปัตยกรรม

- `site/` HTML/CSS/JavaScript (GitHub Pages)
- `worker/` Cloudflare Worker API proxy; ควบคุม CORS และรหัสผ่านผู้ดูแล
- `apps-script/` Apps Script Web App เชื่อมกับ Google Sheets ที่ clasp สร้างเป็น bound project
- `site/user_import_template.xlsx` แม่แบบ Excel A–W (หัวตารางเปล่า ไม่มีข้อมูลส่วนบุคคล)
- `install.sh` สร้าง Google Sheets, Apps Script, Worker, GitHub repo v2 และ GitHub Pages โดยคงระบบเก่าไว้

## ติดตั้งบน Windows ด้วย Git Bash

1. มี Node.js, Git, GitHub CLI, `clasp` และ `npx wrangler`
2. รัน `gh auth login`, `npx wrangler login`, `clasp login` และเปิด Google Apps Script API ใน https://script.google.com/home/usersettings
3. แตกไฟล์ ZIP แล้วเปิด Git Bash **ที่โฟลเดอร์ที่เห็นไฟล์ install.sh**
4. รัน `bash install.sh`  ใช้ค่า repo `nyc-needs-survey-v2` และ Worker `nyc-needs-survey-v2-api` เพื่อไม่ชนระบบเดิม
5. โปรแกรมจะสร้าง Apps Script/Google Sheet, deploy Worker และตั้ง Cloudflare Secrets อัตโนมัติ
6. **ข้อจำกัดสำคัญของ Google:** clasp อาจ deploy Web App โดยไม่เปิดให้คนทั่วไปใช้งาน ต้องเข้า Apps Script -> Deploy -> Manage deployments -> Web app -> Execute as: Me -> Who has access: Anyone แล้วอนุญาตสิทธิ์และใช้ URL `/exec` ใหม่ โปรแกรมตรวจการเข้าถึงก่อนเปิดเผยหน้าเว็บ และจะแจ้งให้ทำขั้นตอนนี้หากจำเป็น
7. เมื่อเว็บไซต์ GitHub Pages เปิดใช้งานแล้ว ทดสอบกรอก 1 รายการ และตรวจชีต `Responses` ก่อนส่งลิงก์จริง

## หน้าผู้ดูแล

- `admin.html` ใช้รหัสผ่าน 12+ ตัวที่กำหนดตอนติดตั้ง โดย Worker ตรวจ hash และส่งข้อมูลไป Apps Script ผ่าน secret เท่านั้น
- ดูจำนวนคำตอบ แยกตามบทบาท จัดอันดับระบบ ดาวน์โหลด CSV และตั้งค่าเปิด–ปิดแบบสำรวจ
- ใน Google Sheet มีชีต `Responses` และ `Settings`; ไม่มีการนำข้อมูลจากไฟล์ตัวอย่างขึ้น Google Sheets โดยอัตโนมัติ

## ข้อควรระวังข้อมูลส่วนบุคคล

- ห้ามบันทึก secret หรือรหัสผ่านใน GitHub repository สาธารณะ
- รหัสผ่านผู้ดูแลและ Shared Secret ถูกใส่เป็น Cloudflare secrets; Apps Script Web App ต้องลงสิทธิ์ Execute as Me เนื่องจาก backend ใช้ SpreadsheetApp
- เครื่องมือ A–W ประมวลผล CSV เฉพาะเครื่องผู้ใช้ ไม่ส่งไฟล์ไป Worker, Apps Script หรือ Google Sheets
- สำหรับการใช้งานจริง ควรเปิด Cloudflare Rate Limiting / Turnstile กับ endpoint submit และ admin รวมถึงมีนโยบายการเก็บและลบข้อมูล
- CSV อาจทำให้ Excel ตัด 0 นำหน้า จึงแนะนำ `user_import_template.xlsx` ที่กำหนดรูปแบบข้อความสำหรับรหัสและเลขบัตรประชาชน

## API

| Method | Route | Description |
| --- | --- | --- |
| GET | `/health` | ตรวจ Worker/Secret config |
| GET | `/api/config` | อ่านชื่อแบบสำรวจ ระบบ และสถานะเปิดรับ |
| POST | `/api/submit` | บันทึกคำตอบ (ไม่ส่ง PII) |
| GET | `/api/admin` | ข้อมูลรวม/รายการ (ต้องมี X-Admin-Password) |
| POST | `/api/admin/settings` | เปิด/ปิดและแก้ข้อความ (ต้องมี X-Admin-Password) |

## ตรวจสอบหลังติดตั้ง

- เปิด URL ฟอร์ม GitHub Pages และหน้า admin.html
- เปิด `/health` Worker ต้องได้ `ok: true` และ `configured: true`
- เปิด `/api/config` Worker ต้องได้ JSON `ok: true` (ไม่ควรเป็นหน้า Login ของ Google)
- ส่งแบบสำรวจ 1 ครั้งและเห็นแถวใน Google Sheets `Responses`
- ลองรหัสผ่านผิดในหน้า Admin แล้วต้องถูกปฏิเสธ
- ลองปิดแบบสำรวจใน Admin แล้วฟอร์มต้องแสดงปิดรับคำตอบ

## ความเข้ากันได้กับระบบเดิม

เวอร์ชันใหม่ใช้ GitHub repo, Cloudflare Worker และ Apps Script/Sheets คนละชุดโดยค่าเริ่มต้น ไม่เปลี่ยนหรือทำลายข้อมูลในระบบเดิม `nyc-needs-survey`.

## การทดสอบโค้ดก่อนติดตั้ง

รัน `node tests/smoke-worker.mjs` และ `node tests/smoke-apps-script.cjs` ได้โดยไม่เรียกบริการจริง เพื่อทดสอบรูปแบบ API, CORS, การตรวจสิทธิ์, การบันทึกจำลอง, สถิติ และการเปิด/ปิดแบบสำรวจ

## การแก้ปัญหา CORS จากเวอร์ชันเดิม

Cloudflare ต้องตั้ง `ALLOWED_ORIGIN` เป็น **origin** เช่น `https://eirchard.github.io` **ไม่ใช่** URL GitHub Pages ที่มี `/nyc-needs-survey` ต่อท้าย เนื่องจาก HTTP Origin header จะมีเพียง scheme + hostname + port เท่านั้น เวอร์ชันใหม่นี้แก้แล้วใน `install.sh` และมี smoke test ที่ส่ง Origin จำลองจริง
