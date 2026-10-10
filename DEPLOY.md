# Lab 19 — โค้ดและวิธี Deploy บน Vercel

ผู้จัดทำ: เศรษฐวุฒิ สมาเกตุ — 680610725

โค้ดต้นฉบับที่แก้แล้วอยู่ใน `/Users/nostalgiax/Documents/Basic Com/lab19-680610725` และมีสำเนาใน ZIP ที่ส่งให้ งานนี้ยังไม่ได้ deploy ขึ้นบัญชี Vercel จริง

## ส่วนที่ทำเสร็จ

- Backend: PUT/DELETE students, PUT/DELETE enrollments พร้อม Zod, ตรวจสิทธิ์และ status code ตามโจทย์
- ลบนักศึกษาด้วย `$transaction` โดยลบ Enrollment และ File records ที่อ้างอิงก่อน
- Frontend: add/update/removeStudent แปลง emails เป็น string[] ก่อนส่ง API
- Frontend: updateEnrollment/dropEnrollment อัปเดต state หลัง API สำเร็จ
- หน้า STUDENT มี Dialog เปลี่ยนวิชา เลือกเฉพาะวิชาที่ยังไม่ได้ลง และ ConfirmDeleteButton ยืนยันก่อนยกเลิก
- ใส่ชื่อและรหัสผู้จัดทำใน footer ทั้ง ADMIN/STUDENT และหน้าแรก
- เพิ่ม frontend/vercel.json สำหรับ React Router; backend ใช้ Express app export โดยไม่เปิด app.listen บน Vercel
- กำหนด pnpm@12.4.1 เพื่อให้ install/build ใช้เวอร์ชันตรงกับที่ตรวจสอบ

## 1. เตรียม MongoDB Atlas และทดสอบในเครื่อง

1. สร้าง cluster และ Database User สำหรับฐานข้อมูล lab19 โดยให้สิทธิ์ read/write ฐานข้อมูลนี้
2. คัดลอก connection string จาก Connect → Drivers ใส่ชื่อฐานข้อมูล `lab19` หลัง hostname และใส่ username/password จริง หากรหัสผ่านมีอักขระพิเศษ ให้ URL-encode อักขระนั้น
3. Network Access ต้องอนุญาต IP ของเครื่องที่รันคำสั่ง และเครือข่ายของ backend บน Vercel หากใช้ Atlas สำหรับแลปและเลือก `0.0.0.0/0` หมายถึงเปิดให้เชื่อมต่อจากทุก IP จึงควรใช้ฐานข้อมูลทดสอบและบัญชีที่จำกัดสิทธิ์ เมื่อใช้ระบบจริงให้จำกัดเครือข่ายตามการตั้งค่าของผู้ให้บริการ
4. ใช้ Node.js 24.x และติดตั้ง pnpm เวอร์ชันที่ระบุ:

```bash
npm install -g pnpm@12.4.1
cd "/Users/nostalgiax/Documents/Basic Com/lab19-680610725/backend"
cp .env.example .env
```

แก้ `backend/.env`:

```dotenv
DATABASE_URL=mongodb+srv://USERNAME:PASSWORD@CLUSTER.mongodb.net/lab19?retryWrites=true&w=majority
JWT_SECRET=REPLACE_WITH_A_LONG_RANDOM_SECRET
CORS_ORIGIN=http://localhost:5173
PORT=3000
```

จากโฟลเดอร์ backend รัน:

```bash
pnpm install --frozen-lockfile
pnpm db:generate
pnpm db:push
pnpm db:seed
pnpm dev
```

`db:push` สร้าง indexes ตาม Prisma schema; MongoDB ไม่ใช้ `prisma migrate deploy` ส่วน `db:seed` เติม Students, Courses, Users จาก JSON ของโปรเจกต์และข้ามรายการที่มีอยู่แล้ว จึงควรใช้ฐานข้อมูลใหม่สำหรับแลป หากมี username เดิมแต่รหัสผ่านต่างกัน seed จะไม่เขียนทับ

เปิดอีก Terminal:

```bash
cd "/Users/nostalgiax/Documents/Basic Com/lab19-680610725/frontend"
pnpm install --frozen-lockfile
pnpm dev
```

เปิด URL ที่ Vite แสดง (ปกติ http://localhost:5173) หากไม่กำหนด `VITE_API_URL` โค้ดจะใช้ `http://localhost:3000/api/v3` อยู่แล้ว

บัญชีทดสอบหลัง seed:

| Role | Username | Password |
| --- | --- | --- |
| ADMIN | user4@abc.com | 5678 |
| STUDENT | user1@abc.com | 1234 |
| STUDENT | user2@abc.com / user3@abc.com | 1234 |

## 2. อัปโหลดโค้ดไป GitHub

ใช้ repository ของตัวเองที่มี `backend/` และ `frontend/` อยู่ระดับบนสุด ตรวจ `git diff` แล้ว commit/push ไฟล์ที่แก้ รวม `package.json`, `pnpm-lock.yaml`, `frontend/vercel.json` และคู่มือนี้ด้วย ห้ามอัปโหลด `backend/.env` ที่มีรหัสผ่านจริง, node_modules หรือ generated Prisma client

## 3. Deploy Backend ก่อน

ใน Vercel เลือก Add New → Project → Import repository ของงาน แล้วตั้งค่า:

| รายการ | ค่า |
| --- | --- |
| Project Name | lab19-680610725-backend (หรือชื่อที่ว่าง) |
| Root Directory | backend |
| Framework Preset | Other |
| Node.js Version | 24.x |
| Install Command | pnpm install --frozen-lockfile |
| Build Command | pnpm build (backend/vercel.json มีให้แล้ว) |
| Output Directory | ใช้ค่า default สำหรับ Other ไม่ต้องใส่ dist |

เพิ่ม Environment Variables ให้ Production และ Preview หากจะทดสอบ Preview:

| Key | ค่า |
| --- | --- |
| DATABASE_URL | MongoDB Atlas connection string เดียวกับฐานข้อมูลที่ seed ไว้ |
| JWT_SECRET | random secret ยาวที่กำหนดเอง |
| CORS_ORIGIN | http://localhost:5173 ชั่วคราว |
| ENABLE_EXPERIMENTAL_COREPACK | 1 |

ไม่ต้องกำหนด PORT บน Vercel กด Deploy แล้วคัดลอก Production URL จริง เช่น `https://lab19-680610725-backend.vercel.app`

เปิด URL `/` ของ backend ควรเห็น `Lecture10 API services` แต่การเปิดหน้านี้ได้ยังไม่ยืนยันว่าฐานข้อมูลเชื่อมต่อสำเร็จ ต้องทดสอบ login และอ่าน/แก้ข้อมูลในขั้นตอนสุดท้ายด้วย

## 4. Deploy Frontend

Add New → Project → Import repository เดิมอีกครั้ง:

| รายการ | ค่า |
| --- | --- |
| Project Name | lab19-680610725-frontend (หรือชื่อที่ว่าง) |
| Root Directory | frontend |
| Framework Preset | Vite |
| Node.js Version | 24.x |
| Install Command | pnpm install --frozen-lockfile |
| Build Command | pnpm build |
| Output Directory | dist |

เพิ่ม Environment Variables:

```dotenv
VITE_API_URL=https://YOUR-BACKEND.vercel.app/api/v3
ENABLE_EXPERIMENTAL_COREPACK=1
```

แทน YOUR-BACKEND ด้วย hostname จริง ต้องมี `/api/v3` ท้าย URL แล้วกด Deploy

## 5. ตั้ง CORS และ Redeploy

กลับไป Backend → Settings → Environment Variables ตั้ง:

```dotenv
CORS_ORIGIN=https://YOUR-FRONTEND.vercel.app
```

ใส่ origin อย่างเดียว ไม่ใส่ `/login`, `/api/v3` หรือ slash ท้าย URL หากต้องการรัน local พร้อมกัน ให้คั่นด้วย comma โดยไม่ใส่ช่องว่าง เช่น:

```dotenv
CORS_ORIGIN=https://YOUR-FRONTEND.vercel.app,http://localhost:5173
```

จาก Backend → Deployments เลือก Redeploy เพื่อใช้ค่าล่าสุด หากแก้ `VITE_API_URL` ต้อง redeploy frontend เช่นกันเพราะ Vite อ่านค่านี้ตอน build URL Preview เป็นคนละ origin กับ Production จึงต้องเพิ่ม origin Preview หากจะใช้ทดสอบ

## 6. ตรวจหลัง Deploy

1. ADMIN login แล้วเพิ่มนักศึกษาใหม่ ตรวจอีเมล/ความสนใจ แก้ชื่อโดยรหัสนักศึกษาเปลี่ยนไม่ได้
2. ลงทะเบียนให้นักศึกษาทดสอบ แล้วลบนักศึกษานั้น ตรวจว่ารายการลงทะเบียนหายด้วย
3. STUDENT login ลงวิชา เปิดเปลี่ยนวิชา ตรวจว่าปุ่มบันทึกปิดจนกว่าจะเลือก และมีเฉพาะวิชาที่ยังไม่ได้ลง
4. บันทึกแล้วตรวจวิชาใหม่ จากนั้นยกเลิกและตรวจ Dialog ยืนยัน
5. Refresh หน้า `/admin/students` หรือ `/student/enrollments` ต้องไม่ขึ้น Vercel 404
6. ตรวจ footer ชื่อและรหัส 680610725 ทั้งสอง role

## แก้ปัญหาที่พบบ่อย

| อาการ | จุดที่ตรวจ |
| --- | --- |
| Frontend เรียก localhost หลัง deploy | ตั้ง VITE_API_URL ใน frontend และ Redeploy |
| CORS error | CORS_ORIGIN ต้องตรงกับ origin ของ frontend รวม https และ Redeploy backend |
| login/CRUD ขึ้น 500 | ดู Function Logs ตรวจ DATABASE_URL, Database User, Network Access และ seed |
| login ไม่ผ่านทั้งที่ใช้บัญชีตัวอย่าง | ตรวจว่า seed ลงฐานข้อมูลเดียวกับ DATABASE_URL บน Vercel |
| Transactions are not supported | MongoDB ต้องเป็น replica set; Atlas รองรับ เหมาะกับ `$transaction` ในงานนี้ |
| Refresh แล้ว 404 | ตรวจว่า frontend/vercel.json ถูก push และ Root Directory เป็น frontend |
| pnpm version/lockfile error | ตั้ง ENABLE_EXPERIMENTAL_COREPACK=1, ใช้ packageManager และ lockfile ที่ส่งให้ |
| Prisma engine หาไม่พบ | ตรวจ build ใช้ pnpm build และ backend/vercel.json includeFiles ของ generated Prisma |

## ผลตรวจในเครื่องและขอบเขต

- Frontend build ผ่าน และ lint เฉพาะไฟล์ที่แก้ผ่าน
- Backend build/Prisma generate ผ่าน โดยใช้ DATABASE_URL จำลองสำหรับ generate ซึ่งไม่ได้เชื่อมฐานข้อมูล
- ทดสอบ 20 HTTP cases ด้วย router จริง แต่ mock Prisma และ middleware authentication ครอบคลุม validation, ownership, missing records, same course, conflict, success, partial update และ cascade delete
- ทดสอบ browser หน้า STUDENT กับ API จำลอง: ปุ่ม disabled, รายการวิชา, เปลี่ยนวิชาแล้ว state อัปเดต และยืนยันยกเลิกแล้วแถวหาย
- lint ทั้ง frontend ยังมี 7 error ในไฟล์ component/hook เดิม: theme-provider, badge, button, combobox, sidebar, tabs และ use-mobile
- ยังไม่ได้ทดสอบฐานข้อมูล MongoDB จริง, login จริงกับฐานข้อมูล หรือ deployment บนบัญชี Vercel

## เอกสารอ้างอิง

- [Vercel — Vite](https://vercel.com/docs/frameworks/frontend/vite)
- [Vercel — Package Managers](https://vercel.com/docs/package-managers)
- [Vercel — Configure a Build / Corepack](https://vercel.com/docs/builds/configure-a-build)
- [Vercel — Environment Variables](https://vercel.com/docs/environment-variables/managing-environment-variables)
- [MongoDB Atlas — IP Access List](https://www.mongodb.com/docs/atlas/security/ip-access-list/)
- [Prisma — MongoDB และ replica sets](https://www.prisma.io/docs/orm/v7/core-concepts/supported-databases/mongodb)
