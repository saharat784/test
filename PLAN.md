# Project Plan: Meeting Room Booking System

## 1. Problem Understanding & Assumptions
- **เป้าหมาย:** สร้างระบบจัดการการจองห้องประชุมที่ป้องกันการจองเวลาทับซ้อนกันได้อย่างแม่นยำ
- **Assumptions (สมมติฐานเพิ่มเติมจากโจทย์):**
  1. มีห้องประชุมตั้งต้น 3 ห้อง (Room A, Room B, Room C) โดย Seed ข้อมูลไว้ในระบบทันที
  2. เวลาจองต้องเป็นช่วงเวลาในอนาคต และ `end_time` ต้องมากกว่า `start_time` เสมอ
  3. ข้ามระบบ Authentication จริง โดยให้ผู้จองกรอกชื่อผู้จองในฟอร์ม directly เพื่อโฟกัสที่ Core Booking Logic

## 2. Tech Stack & Architectural Decisions
- **Backend:** Python + FastAPI
  - *เหตุผล:* จัดการ Request/Response Schema และ Data Validation ผ่าน Pydantic ได้รวดเร็ว ลดโค้ด Boilerplate
- **Database:** SQLite
  - *เหตุผล:* ทำงานแบบ File-based ไม่ต้องเสียเวลา Config Database Server และสามารถรันทดสอบได้ทันทีที่ Clone repo
- **Frontend:** React (Vite) + Tailwind CSS
  - *เหตุผล:* ขึ้นโครง UI และจัดการ State ของฟอร์ม/ตารางรายการได้เร็วที่สุด

## 3. Data Model & API Design
### Database Schema (`bookings`)
- `id`: Integer (PK, Auto-increment)
- `room_name`: String (Not Null) — เช่น "Room A"
- `title`: String (Not Null) — หัวข้อการประชุม
- `booked_by`: String (Not Null) — ชื่อผู้จอง
- `start_time`: DateTime (ISO 8601, Not Null)
- `end_time`: DateTime (ISO 8601, Not Null)
- `status`: String (Default: `"CONFIRMED"`) — `CONFIRMED` | `CANCELLED`

### API Endpoints
- `GET /api/bookings` — ดึงรายการจองทั้งหมด (เรียงตาม `start_time` ล่าสุด)
- `POST /api/bookings` — สร้างการจองใหม่ (ตรวจสอบ Overlap ก่อนบันทึก)
- `PATCH /api/bookings/{id}/cancel` — เปลี่ยนสถานะเป็น `CANCELLED` (คืนช่วงเวลาให้ห้องว่าง)

## 4. Task Prioritization & Execution Plan

### P0: Must-Have (Core MVP — เป้าหมาย: เสร็จสมบูรณ์ 100%)
- [ ] **Setup:** ขึ้นโครงโปรเจกต์ Backend (FastAPI + SQLite) และ Frontend (React + Tailwind) พร้อมตั้งค่า CORS
- [ ] **Backend Core:** สร้างตาราง `bookings` และ API `GET`, `POST`, `PATCH /cancel`
- [ ] **Core Business Logic (Overlap Detection):** เขียนเงื่อนไขตรวจสอบเวลาทับซ้อนสำหรับห้องเดียวกันเฉพาะรายการที่ `status == 'CONFIRMED'` (`new_start < existing_end AND new_end > existing_start`)
- [ ] **Frontend Core:** สร้างหน้าจอหลัก 2 ส่วน (ฟอร์มสร้างการจอง + ตารางแสดงรายการจองและปุ่มยกเลิก) เชื่อมต่อ API ให้ใช้งานได้จริง

### P1: Should-Have (Validation & UX Resilience)
- [ ] **Input & Boundary Validation:**
  - ดักกรณี `start_time >= end_time` (แจ้ง Error 400)
  - ดักกรณีจองเวลาย้อนหลังในอดีต
  - แสดงข้อความ Error ที่ชัดเจนบนหน้าเว็บเมื่อจองชนเวลาผู้อื่น (HTTP 409 Conflict)
- [ ] **UI States:** เพิ่ม Loading state ป้องกันการกดปุ่มจองซ้ำ และแสดง Empty state เมื่อยังไม่มีรายการจอง

### P2: Nice-to-Have (ทำเมื่อ P0 และ P1 เสร็จแล้ว)
- [ ] เพิ่มตัวกรอง (Filter) ดูรายการจองแยกตามชื่อห้องประชุม หรือวันที่
- [ ] เพิ่ม Confirmation Modal ก่อนกดยกเลิกการจอง
- [ ] เขียน Unit Test ทดสอบฟังก์ชัน Overlap Detection (ครอบคลุมเคสเวลาชนบางส่วน, เวลาครอบคลุมทั้งหมด, เวลาต่อกันพอดี)

## 5. Edge Cases & Risk Mitigation
1. **เวลาต่อกันพอดี (Adjacent Time Slots):** เช่น จอง 10:00–11:00 และอีกคนจอง 11:00–12:00 ต้อง **อนุญาตให้จองได้** (ใช้เครื่องหมาย `<` และ `>` ไม่ใช่ `<=` และ `>=`)
2. **การจองซ้ำในเวลาที่เคยถูกยกเลิกไปแล้ว:** หากรายการเดิมมีสถานะ `CANCELLED` ระบบต้องอนุญาตให้คนใหม่จองช่วงเวลานั้นได้
3. **การจัดการ Timezone:** กำหนดมาตรฐานการส่งข้อมูลระหว่าง Frontend-Backend เป็น ISO 8601 String ที่ชัดเจน เพื่อป้องกันเวลาคลาดเคลื่อน