# MarketHub

## Overview

MarketHub คือระบบจอง-เช่าล็อคตลาดออนไลน์ ให้ผู้ค้าเลือกล็อค จอง ชำระเงิน และดูสัญญาเช่าได้เอง ส่วนแอดมินจัดการโซน ล็อค การอนุมัติจอง การจ่ายเงิน และการคืนเงินผ่านหน้า Dashboard เดียว

**ระบบนี้คือ**: ระบบจัดการการเช่าพื้นที่ล็อคในตลาด ตั้งแต่การจอง → อนุมัติ → ชำระเงิน → แจ้งเตือนต่อสัญญา

**ระบบนี้ไม่ใช่**: ระบบ POS หน้าร้าน, ระบบบัญชี/บัญชีภาษีเต็มรูปแบบ, หรือ marketplace ขายสินค้าออนไลน์

## Why

ปัญหาเดิม: การจองล็อคตลาดทำด้วยกระดาษ/แชท ทำให้ตรวจสอบล็อคว่างยาก อนุมัติช้า และไม่มีประวัติที่ตรวจสอบย้อนกลับได้

วิธีแก้: ทำระบบจองออนไลน์ที่ผูก state ของล็อค (ว่าง/จอง/เช่าอยู่) เข้ากับ booking และ payment โดยตรง พร้อม audit log ทุกการเปลี่ยนแปลง

ผลลัพธ์: ผู้ค้าจองล็อคได้เองแบบ real-time เห็นสถานะล็อคตรงกับความจริงเสมอ แอดมินลดงานเอกสารและมีหลักฐานย้อนหลังทุกธุรกรรม

## Features

- ดูล็อค/โซนตลาด พร้อมปฏิทินความว่าง (Calendar Picker) และค้นหา/บุ๊กมาร์กล็อคที่สนใจ
- จองล็อคแบบรายวัน/รายสัปดาห์/รายเดือน พร้อมคิว (Queue) กรณีล็อคไม่ว่าง
- อัปโหลดสลิปโอนเงิน ตรวจสอบด้วย OCR (Tesseract.js) ก่อนส่งให้แอดมินอนุมัติ
- แจ้งเตือนอัตโนมัติ (เตือนต่อสัญญา, สถานะจอง) ผ่าน in-app notification และอีเมล (Resend)
- Admin dashboard: จัดการโซน/ล็อค, อนุมัติ/ปฏิเสธการจอง, ตรวจสอบการชำระเงิน, คืนเงิน, จัดการพนักงาน, export ข้อมูลเป็น Excel (XLSX)
- Cron job ตรวจสัญญาใกล้หมดอายุ/ปลดล็อคอัตโนมัติ

## Quick Start

```bash
npm install
cp .env.example .env
npm run seed-admin
npm run dev
```

เปิด [http://localhost:3000](http://localhost:3000)

รันด้วย Docker (MongoDB ในตัว):

```bash
docker compose up --build
```

## Tech Stack

- **Framework**: Next.js 16 (App Router) + TypeScript + React 19
- **UI**: React Bootstrap, Sass, Framer Motion, SweetAlert2
- **Database**: MongoDB + Mongoose
- **Auth**: NextAuth.js v5 (JWT)
- **Forms/Validation**: React Hook Form + Zod
- **Files/OCR**: Cloudinary (เก็บสลิป), Tesseract.js (อ่านสลิปโอนเงิน)
- **Email**: Resend
- **Export**: xlsx
- **Testing**: Vitest + Testing Library

## Architecture

```
Client (React) → Next.js API Routes (src/app/api) → Service/Validation (src/lib) → Mongoose Models (src/models) → MongoDB
```

- Route groups แยกตามสิทธิ์: `(auth)` หน้า login/register, `(user)` หน้าผู้เช่า, `(admin)` หน้าแอดมิน
- `src/lib/auth` — NextAuth config และ helper ตรวจสิทธิ์
- `src/lib/db` — การเชื่อมต่อ MongoDB
- `src/lib/notification`, `src/lib/email` — ส่งแจ้งเตือน/อีเมล
- `src/lib/ocr` — อ่านสลิปโอนเงินด้วย Tesseract.js
- `src/app/api/cron` — endpoint สำหรับ scheduled job (ตรวจ CRON_SECRET)

## Workflow / Lifecycle

การจองล็อค:

```
เลือกล็อค → จอง (PENDING) → อัปโหลดสลิป+OCR → แอดมินตรวจสอบ/อนุมัติ → ชำระเงินยืนยัน (APPROVED) → เช่าอยู่ (ACTIVE) → ใกล้หมดสัญญา (แจ้งเตือน) → ต่อสัญญา / สิ้นสุด
```

ถ้าล็อคไม่ว่าง ผู้ใช้เข้าคิว (Queue) และรับแจ้งเตือนเมื่อว่าง ("สนใจ" ผ่าน Interest List)

## User Roles

- **User (ผู้เช่า)**: ดูล็อค จอง ชำระเงิน ดูประวัติ/สัญญา บุ๊กมาร์กล็อค รับแจ้งเตือน
- **Admin**: จัดการโซน/ล็อค อนุมัติ/ปฏิเสธการจอง ตรวจสอบการชำระเงินและคืนเงิน จัดการพนักงาน export รายงาน

## Business Rules

- สถานะล็อคเปลี่ยนได้ผ่าน booking/payment flow เท่านั้น ห้ามแก้ state ล็อคตรงในฐานข้อมูล
- ทุกการเปลี่ยนสถานะสำคัญ (อนุมัติ, คืนเงิน, ปลดล็อค) ถูกบันทึกใน `AuditLog`
- Endpoint ใต้ `src/app/api/cron` ต้องมี `CRON_SECRET` ที่ถูกต้อง ไม่งั้นปฏิเสธคำขอ
- การชำระเงินต้องมีสลิปแนบและผ่านการตรวจสอบก่อนอนุมัติจอง

## Project Structure

```
src/
├── app/
│   ├── (auth)/       # login, register
│   ├── (user)/       # หน้าผู้เช่า: locks, bookmarks, my-bookings, notifications, profile
│   ├── (admin)/      # หน้าแอดมิน
│   └── api/          # API routes: auth, bookings, locks, payments, admin, cron, ฯลฯ
├── components/       # UI components แยกตามโดเมน (admin, locks, forms, layout, ...)
├── lib/              # auth, db, email, notification, ocr, utils, validations
├── models/           # Mongoose schemas (User, Lock, Booking, Payment, Zone, Queue, ...)
├── styles/
└── types/
```

## Environment Variables

| ตัวแปร | ใช้ทำอะไร |
|---|---|
| `MONGODB_URI` | connection string ของ MongoDB |
| `NEXTAUTH_URL` | URL ของแอป สำหรับ NextAuth |
| `NEXTAUTH_SECRET` | secret เข้ารหัส session/JWT (อย่างน้อย 32 ตัวอักษร) |
| `CLOUDINARY_CLOUD_NAME` / `CLOUDINARY_API_KEY` / `CLOUDINARY_API_SECRET` | เก็บไฟล์สลิปการชำระเงิน |
| `ADMIN_NAME` / `ADMIN_USERNAME` / `ADMIN_PASSWORD` | ข้อมูล admin เริ่มต้น ใช้กับ `npm run seed-admin` |
| `CRON_SECRET` | ป้องกัน endpoint ใน `src/app/api/cron` |
| `RESEND_API_KEY` / `EMAIL_FROM` | ส่งอีเมลแจ้งเตือนผ่าน Resend |

ดูตัวอย่างเต็มที่ [.env.example](.env.example)

## Commands

```bash
npm run dev            # dev server
npm run build           # build production
npm run start            # start production build
npm run lint              # eslint
npm run test               # vitest
npm run seed-admin          # สร้าง admin เริ่มต้นจาก .env
npm run seed-market-data     # seed ข้อมูลตลาดตัวอย่าง
```

## Database

MongoDB ผ่าน Mongoose โมเดลหลักอยู่ใน [src/models](src/models): `User`, `Zone`, `Lock`, `Booking`, `Payment`, `Refund`, `Queue`, `InterestList`, `Notification`, `AuditLog`

Local dev แนะนำใช้ MongoDB ผ่าน Docker (`docker-compose.yml`), production ใช้ MongoDB Atlas

## Testing

```bash
npm run test
```

ใช้ Vitest + Testing Library ดูรายละเอียดผลทดสอบเพิ่มเติมที่ [TEST_RESULTS.md](TEST_RESULTS.md) และ checklist ที่ [TESTING_CHECKLIST.md](TESTING_CHECKLIST.md)

## Developer Guide

- เริ่มอ่านโค้ดที่ `src/app/api` (business logic ฝั่ง server) และ `src/models` (โครงสร้างข้อมูล) ก่อน แล้วค่อยดู UI ใน `src/app/(user)` / `src/app/(admin)`
- เพิ่ม feature ใหม่: เพิ่ม model ใน `src/models` (ถ้าจำเป็น) → เพิ่ม API route ใน `src/app/api` → validation ด้วย Zod ใน `src/lib/validations` → หน้า UI ใน route group ที่เกี่ยวข้อง
- ดูรายละเอียดเชิงลึกเพิ่มเติมที่ [developer_implementation_guide.md](developer_implementation_guide.md) และแผนออกแบบระบบที่ [market_lock_rental_system_plan.md](market_lock_rental_system_plan.md)

## Deployment

- **Vercel**: เชื่อม repo แล้ว deploy ได้ทันที (ดู [vercel.json](vercel.json))
- **Docker**: `docker compose up --build` รันแอป + MongoDB local ดูรายละเอียดที่ [DOCKER_GUIDE.md](DOCKER_GUIDE.md)

## License

Private project — ไม่เปิด public license
