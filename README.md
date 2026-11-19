# AskFile

เว็บจัดเก็บและแชร์ไฟล์งาน (HTML + JavaScript ล้วน) ใช้ **Supabase** เก็บข้อมูลและไฟล์ และ **Firebase Auth** สำหรับล็อกอินด้วย Google
มีหน้า **การ์ดสมาชิก** (`card.html`) ที่แชร์เป็นลิงก์สาธารณะได้ เช่น วางในไบโอ Instagram

## โครงสร้างไฟล์
| ไฟล์ | หน้าที่ |
|---|---|
| `index.html` | โครงสร้างหน้าหลัก (หน้าแรก, พื้นที่ทำงานไฟล์, โน้ต, หน้าต่างดูตัวอย่าง) |
| `app.js` | ตรรกะหลัก: ล็อกอิน สิทธิ์ ไฟล์ โน้ต การดาวน์โหลด การอัปโหลดหลายไฟล์ |
| `card.html` | หน้าการ์ดสมาชิก: แสดง แชร์ลิงก์ และแก้ไขการ์ดของตัวเอง |
| `config.js` | **ค่าตั้งค่าทั้งหมด** (URL และคีย์ของ Supabase, ค่า Firebase) แก้ที่ไฟล์นี้ไฟล์เดียว |
| `i18n.js` | ข้อความไทยและอังกฤษ |
| `theme.js` | สลับธีมมืด/สว่าง |
| `dots.js` | พื้นหลังจุดของหน้าแรก |
| `tailwind-config.js` | การตั้งค่าสีและฟอนต์ |
| `styles.css` | สไตล์ (กระจก พื้นหลังไล่สี อนิเมชัน) |

## ขั้นตอนตั้งค่าก่อนใช้งาน

### 1. Firebase
- เปิด **Google** ใน Authentication → Sign-in method
- ใน Authentication → Settings → **Authorized domains** เพิ่มโดเมนที่ใช้งานจริง (เช่น `askfile-web.vercel.app`) และคง `localhost` ไว้สำหรับทดสอบ
- คัดลอกค่า Web config มาใส่ใน `config.js` (ถ้าเปลี่ยนโปรเจกต์ Firebase)

### 2. Supabase
- ไปที่ Authentication → Sign In / Providers → **Third-party Auth** แล้วเพิ่ม Firebase โดยใส่ Project ID เป็น `fen-board`
  ถ้าไม่ตั้งค่านี้ คำขอที่แนบโทเคนจาก Firebase จะถูกปฏิเสธด้วยสถานะ 401
- ตั้งค่า URL และคีย์ใน `config.js`

### 3. ฐานข้อมูล
- รัน SQL ของตาราง `files`, `projects`, `share_links`, `notes`, `allowed_emails`, `storage bucket project-files` และ policy ที่เกี่ยวข้อง (ชุดนี้ของโปรเจกต์เดิม ไม่ได้รวมอยู่ในไฟล์ของโปรเจกต์)
- ฟังก์ชันที่แอปเรียกใช้ต้องมีอยู่ เช่น `is_allowed()`, `my_role()`, `can_view()`

### 4. สิทธิ์ผู้ใช้
เพิ่มอีเมลที่อนุญาตใน `allowed_emails` พร้อมระดับ `admin` หรือ `uploader`

### 5. ตารางการ์ดสมาชิก (`member_cards`)
รัน SQL นี้ใน SQL Editor ของ Supabase

```sql
create table if not exists member_cards (
  email text primary key,
  name text not null default '',
  handle text not null default '',
  role text not null default '',
  bio text not null default '',
  tags text[] not null default '{}',
  contact_email text not null default '',
  discord text not null default '',
  instagram text not null default '',
  avatar_url text,
  slug text unique,
  is_public boolean not null default true,
  updated_at timestamptz not null default now()
);
alter table member_cards enable row level security;

-- ทุกคนที่ล็อกอิน (และผู้ใช้ทั่วไป) อ่านการ์ดที่เปิดสาธารณะได้ ใช้กับลิงก์แชร์
create policy "member_cards public read" on member_cards
  for select to anon, authenticated using (is_public);

-- เจ้าของเพิ่มและแก้ไขได้เฉพาะการ์ดของอีเมลตัวเอง และต้องอยู่ในรายการอนุญาต
create policy "member_cards insert own" on member_cards
  for insert to authenticated
  with check (lower(email) = lower(auth.jwt() ->> 'email') and public.is_allowed());

create policy "member_cards update own" on member_cards
  for update to authenticated
  using (lower(email) = lower(auth.jwt() ->> 'email') and public.is_allowed())
  with check (lower(email) = lower(auth.jwt() ->> 'email') and public.is_allowed());

grant select, insert, update on member_cards to anon, authenticated;
```

ไม่มีนโยบายลบ จึงไม่มีใครลบการ์ดได้

## การรัน
เปิดผ่าน `https://` หรือ `http://localhost` เท่านั้น เพราะใช้ ES module และ `crypto.subtle`
หลังดีพลอยบน Vercel ให้ใส่โดเมนนั้นใน Firebase → Authorized domains ด้วย

## การใช้หน้าการ์ดและลิงก์แชร์
- เปิด `card.html` แล้วล็อกอิน ระบบจะให้ตั้งชื่อลิงก์ (slug) ตัวอักษร a–z, 0–9 และขีดกลาง
- ลิงก์แชร์คือ `https://โดเมนของคุณ/card.html?u=ชื่อลิงก์` นำไปวางในไบโอ Instagram ได้
- ถ้าปิด "เปิดให้ทุกคนดูการ์ดนี้" ลิงก์จะไม่แสดงการ์ดให้คนอื่น
- ผู้ที่ไม่ได้ล็อกอินเห็นการ์ดแบบอ่านอย่างเดียว

## ความปลอดภัย
- คีย์ใน `config.js` เป็นคีย์สาธารณะ การป้องกันข้อมูลอยู่ที่ RLS ในฐานข้อมูล
- **ห้ามใส่ service role key หรือ secret key ในโค้ดฝั่งเบราว์เซอร์**
- การเข้าถึงไฟล์และข้อมูลทั้งหมดขึ้นกับ RLS และรายการ `allowed_emails`
