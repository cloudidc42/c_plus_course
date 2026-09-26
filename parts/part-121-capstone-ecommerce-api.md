# Part 121: Capstone 1 — E-Commerce Backend API เต็มรูปแบบ (C++ + Crow + PostgreSQL + Docker) (Step 961–968)

> Module K — Capstone Projects และบทสรุป | Part 121 จาก 125
> Step ที่ครอบคลุมใน Part นี้: Step 961–968
> Part ก่อนหน้า: [Part 120 — เส้นทางอาชีพ: จาก Junior สู่ Senior/Staff Engineer](./part-120-career-path.md) | Part ถัดไป: [Part 122 — Capstone 2: Real-time Multiplayer Chat Server](./part-122-capstone-chat-server.md)

## เป้าหมายการเรียนรู้ (Learning Objectives)

หลังจบ Part นี้ ผู้เรียนจะสามารถ:

1. ออกแบบ schema ฐานข้อมูลเชิงสัมพันธ์ (Relational Schema) ที่ normalize ถูกต้องสำหรับระบบ
   E-Commerce จริง — Categories, Products, Users, Orders, OrderItems — พร้อม Foreign Key,
   Constraint และ Index ที่เหมาะสม
2. จัดโครงสร้างโปรเจกต์ C++ ขนาดใหญ่แบบ Layered Architecture (Models / Data Access Layer /
   Routes / Entry Point) ที่ต่อยอดจากแพทเทิร์นของ Part 108 แต่ขยายให้รองรับ 5 resource ที่
   สัมพันธ์กันแทนที่จะเป็น resource เดียว
3. นำ JWT Authentication และ Argon2id Password Hashing จาก Part 110 มาใช้ซ้ำ (reuse) และ
   ขยายเพิ่มแนวคิด **Role-Based Access Control (RBAC)** เพื่อแยกสิทธิ์ระหว่าง `customer`
   กับ `admin`
4. เขียน endpoint ค้นหา/กรอง/แบ่งหน้าสินค้า (filter, search, pagination) ที่ query
   ฐานข้อมูลอย่างมีประสิทธิภาพด้วย JOIN เดียว แทนที่จะเกิดปัญหา N+1 Query
5. ออกแบบและ implement **transaction ที่ปลอดภัยต่อ concurrency อย่างแท้จริง** สำหรับ
   "การสั่งซื้อสินค้า" โดยใช้ `SELECT ... FOR UPDATE` ของ PostgreSQL ป้องกัน race condition
   ตอนตรวจสอบและลด stock พร้อมพิสูจน์ด้วยการยิง request จริงพร้อมกันหลายสิบตัว
6. วินิจฉัยและแก้บั๊ก concurrency ระดับ production จริงที่เกิดขึ้นระหว่างพัฒนาโปรเจกต์นี้เอง
   (การแชร์ `pqxx::connection` ตัวเดียวข้าม thread) ด้วยการสร้าง **Connection Pool** ที่
   thread-safe ด้วยมือ
7. เขียน Dockerfile แบบ Multi-stage Build และ `docker-compose.yml` ที่ประกอบ backend
   C++ เข้ากับ PostgreSQL container ตามแนวคิดที่ต่อยอดจาก Part 112
8. ทดสอบระบบทั้งหมดแบบ End-to-End ด้วย `curl` จริง ครอบคลุม customer journey เต็มรูปแบบ
   (เรียกดูสินค้า → สมัครสมาชิก → login → สั่งซื้อ → ดูประวัติการสั่งซื้อ) พร้อมทุก error
   case ที่ระบบจริงต้องรับมือ

---

## บทนำ: จุดเปลี่ยนจาก "แบบฝึกหัด" สู่ "ระบบจริง"

ตลอด 120 Part ที่ผ่านมา เราเรียนรู้ภาษา C/C++ ตั้งแต่ระดับ `Hello, World!` ไปจนถึง Concurrency,
STL ขั้นสูง, Network Programming, และ Web Development เต็มรูปแบบด้วย Crow + PostgreSQL + JWT
Module I (Part 99–112) ปิดท้ายด้วยโปรเจกต์ Task API ใน Part 108 ที่มี resource เดียว (`Task`)
และ Part 110 ที่เพิ่ม Authentication เข้าไป

**Capstone 1 ที่กำลังจะสร้างนี้คือ "ระดับถัดไป" ของ Task API นั้นโดยตรง** ความแตกต่างที่สำคัญ
ไม่ใช่แค่ "มี resource เยอะกว่า" แต่คือความซับซ้อนเชิงคุณภาพที่เปลี่ยนไปทั้งหมด:

| มิติ | Task API (Part 108/110) | E-Commerce API (Capstone 1) |
|---|---|---|
| จำนวนตาราง | 1–2 ตาราง (`tasks`, `users`) | 5 ตาราง ที่มีความสัมพันธ์ซับซ้อน (FK ข้ามกัน 4 คู่) |
| ฐานข้อมูล | SQLite (Part 108) / SQLite (Part 110) | PostgreSQL (client-server) ทั้งหมด |
| Transaction ที่ซับซ้อนที่สุด | `INSERT` เดี่ยว | **หลาย `UPDATE`+`INSERT` ใน transaction เดียว
  ที่ต้องถูกต้อง 100% แม้มี concurrent request** |
| ปัญหา concurrency ที่ต้องแก้จริง | ไม่มี (CRUD ธรรมดา) | Race condition ตอนลด stock พร้อมกันหลาย request |
| สิทธิ์ผู้ใช้ | ทุกคนที่ login แล้วทำได้เหมือนกันหมด | แยก `customer` / `admin` (RBAC) |
| การ deploy | ไม่ครอบคลุม | Dockerfile + docker-compose (backend + PostgreSQL) |

นี่คือเหตุผลที่ Capstone นี้ถูกจัดให้เป็น "บทปิดท้าย" ของทักษะ Web Development — มันบังคับให้
เอาทุกสิ่งที่เรียนมาต่อกันเป็นระบบเดียวที่ใช้งานได้จริงในโลกธุรกิจจริง ไม่ใช่แค่ตัวอย่าง
ประกอบการสอน

> **ความซื่อสัตย์ทางเทคนิคของ Part นี้**: ทุกไฟล์ C++ ทุกคำสั่ง SQL ทุกคำสั่ง `curl` และผลลัพธ์
> ที่ปรากฏใน Part นี้ **ถูก compile และรันจริงบนเครื่องที่ใช้เขียนบทเรียนนี้** ด้วย `g++ 13.3.0`,
> Crow (header-only, `/usr/local/include/crow`), `libpqxx 7.8.1`, PostgreSQL 16.15 (เปิดผ่าน
> `pg_ctlcluster 16 main start`), OpenSSL 3.0.13 และ libsodium 1.0.18 — รวมถึง **บั๊ก
> concurrency จริงที่เจอระหว่างพัฒนา** (หัวข้อ 121.6) ก็เป็นบั๊กที่เกิดขึ้นจริงกับโค้ดชุดแรกที่เขียน
> ไม่ใช่ตัวอย่างที่แต่งขึ้นมาสอน ส่วนเดียวที่ **ไม่ได้รันจริง** คือคำสั่ง `docker build` /
> `docker compose up` (หัวข้อ 121.8) เพราะสภาพแวดล้อมนี้ไม่มี Docker daemon ทำงานอยู่ (รัน
> `docker info` แล้วได้ `Cannot connect to the Docker daemon`) ซึ่งเป็นข้อจำกัดเดียวกับที่แจ้งไว้
> อย่างชัดเจนใน Part 112 — เนื้อหา Dockerfile ทุกบรรทัดยังคงถูกต้องตามหลักปฏิบัติจริงและตรวจทาน
> อย่างละเอียด เพียงแต่ไม่มีผลลัพธ์จาก terminal จริงมาแสดงในหัวข้อนั้น

---

## 121.1 ภาพรวมโปรเจกต์และการออกแบบระบบ (Step 961)

### เลือก Domain: ทำไมต้อง E-Commerce

E-Commerce (ระบบร้านค้าออนไลน์) ถูกเลือกเป็น Capstone แรกเพราะเป็น domain ที่ทุกคนเข้าใจ
โดยสัญชาตญาณ (ทุกคนเคยซื้อของออนไลน์) แต่ซ่อนความท้าทายทางวิศวกรรมที่ลึกมากไว้ข้างใน โดยเฉพาะ
ปัญหาคลาสสิกที่ทุกระบบร้านค้าออนไลน์ในโลกจริงต้องแก้ให้ได้คือ:

> **"ทำอย่างไรไม่ให้สินค้าถูกขายเกินสต็อกที่มีอยู่จริง เมื่อมีลูกค้าหลายคนกดสั่งซื้อสินค้าตัว
> เดียวกันพร้อมกันในเสี้ยววินาทีเดียวกัน"**

นี่ไม่ใช่ปัญหาทางทฤษฎี — นี่คือปัญหาที่เกิดขึ้นจริงในระบบ e-commerce ทุกระบบ เช่น สินค้า Limited
Edition ที่มีจำนวนจำกัด หรือช่วง Flash Sale ที่คนกดพร้อมกันเป็นพันคน ถ้าระบบออกแบบผิด ร้านค้า
จะขายสินค้าที่ไม่มีอยู่จริงออกไป (Overselling) ซึ่งสร้างความเสียหายทั้งทางธุรกิจและความเชื่อมั่น
ของลูกค้า Capstone นี้จะพิสูจน์ด้วยโค้ดจริงและการทดสอบจริงว่าเราแก้ปัญหานี้ได้อย่างถูกต้อง

### ขอบเขตของระบบ (Scope)

Capstone นี้ประกอบด้วย 5 Entity หลักที่สัมพันธ์กัน:

| Entity | ความหมาย | ใครเป็นเจ้าของ/จัดการ |
|---|---|---|
| **Category** | หมวดหมู่สินค้า (เช่น อิเล็กทรอนิกส์, แฟชั่น, หนังสือ) | Admin เป็นผู้ดูแล (ในบทนี้ seed ไว้ล่วงหน้า) |
| **Product** | สินค้าแต่ละชิ้น มี SKU, ราคา, สต็อก ผูกกับ Category | Admin สร้าง/แก้ไข, ทุกคนดูได้ |
| **User** | บัญชีผู้ใช้ มี role เป็น `customer` หรือ `admin` | สมัครเองผ่าน `/auth/register` |
| **Order** | คำสั่งซื้อ 1 ครั้ง ผูกกับ User ที่สั่ง | สร้างโดย customer ที่ login แล้ว |
| **OrderItem** | รายการสินค้า 1 บรรทัดในคำสั่งซื้อหนึ่งใบ (many-to-many ระหว่าง Order กับ Product) | สร้างพร้อมกับ Order เสมอ |

### รายการ Endpoint ทั้งหมดของระบบ

กำหนด "สัญญา" (Contract) ของ API ทั้งหมดไว้ก่อนเริ่มเขียนโค้ด ตามหลักการเดียวกับ Part 108:

| Method | Path | คำอธิบาย | ต้อง Login? | ต้องเป็น Admin? |
|---|---|---|---|---|
| `GET` | `/health` | Health check | ไม่ต้อง | ไม่ต้อง |
| `GET` | `/categories` | รายการหมวดหมู่ทั้งหมด | ไม่ต้อง | ไม่ต้อง |
| `GET` | `/products` | รายการสินค้า (filter/search/pagination) | ไม่ต้อง | ไม่ต้อง |
| `GET` | `/products/<id>` | รายละเอียดสินค้าตัวเดียว | ไม่ต้อง | ไม่ต้อง |
| `POST` | `/products` | สร้างสินค้าใหม่ | ต้อง | **ต้อง** |
| `PUT` | `/products/<id>` | แก้ไขสินค้า (partial update) | ต้อง | **ต้อง** |
| `POST` | `/auth/register` | สมัครสมาชิก (ได้ role `customer` เสมอ) | ไม่ต้อง | ไม่ต้อง |
| `POST` | `/auth/login` | เข้าสู่ระบบ รับ JWT กลับมา | ไม่ต้อง | ไม่ต้อง |
| `POST` | `/orders` | สั่งซื้อสินค้า (หลายรายการในครั้งเดียว) | ต้อง | ไม่ต้อง |
| `GET` | `/orders` | ประวัติการสั่งซื้อของตัวเอง | ต้อง | ไม่ต้อง |
| `GET` | `/orders/<id>` | รายละเอียด order เดียว (ต้องเป็นเจ้าของเท่านั้น) | ต้อง | ไม่ต้อง |

สังเกตว่าโครงสร้างนี้ยึดหลัก REST เดียวกับ Part 108 ทุกประการ (Resource-based URL, HTTP method
ตรงความหมาย, HTTP status code ตรงความจริง) — Capstone นี้ไม่ได้เปลี่ยนหลักการออกแบบ API แต่
เพิ่มความซับซ้อนของ **ข้อมูลและ business logic เบื้องหลัง** endpoint เหล่านี้

### สภาพแวดล้อมที่ใช้พัฒนาและทดสอบจริง

```bash
$ g++ --version | head -1
g++ (Ubuntu 13.3.0-6ubuntu2~24.04.1) 13.3.0

$ pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
16  main    5432 online postgres /var/lib/postgresql/16/main /var/log/postgresql/postgresql-16-main.log

$ pkg-config --modversion libpqxx
7.8.1

$ dpkg -l | grep -E "nlohmann|libssl-dev|libsodium"
ii  libsodium-dev:amd64   1.0.18-1ubuntu0.24.04.1   amd64  ...headers
ii  libsodium23:amd64     1.0.18-1ubuntu0.24.04.1   amd64  ...shared library
ii  libssl-dev:amd64      3.0.13-0ubuntu3.7         amd64  ...OpenSSL headers
ii  nlohmann-json3-dev    3.11.3-1                  all    JSON for Modern C++

$ nproc
4
```

ทุกอย่างพร้อมใช้งานแบบ native บนเครื่องนี้ (ไม่ผ่าน container) — Crow เป็น header-only library
ที่ติดตั้งไว้แล้วที่ `/usr/local/include/crow` จาก Module I

---

## 121.2 ออกแบบฐานข้อมูล: Schema, Normalization และ Index (Step 962)

### ER Diagram (Entity-Relationship Diagram)

```
┌─────────────────────┐
│     categories      │
├─────────────────────┤
│ id            PK    │
│ name          UNIQUE │
│ slug          UNIQUE │
│ created_at           │
└──────────┬───────────┘
           │ 1
           │
           │ N
┌──────────▼───────────┐        ┌──────────────────────┐
│       products        │        │         users         │
├───────────────────────┤        ├──────────────────────┤
│ id             PK     │        │ id            PK      │
│ category_id    FK ────┼───┐    │ username      UNIQUE  │
│ sku            UNIQUE │   │    │ password_hash         │
│ name                  │   │    │ role (customer/admin) │
│ description           │   │    │ created_at            │
│ price_cents           │   │    └───────────┬───────────┘
│ stock                 │   │                │ 1
│ created_at            │   │                │
└──────────┬────────────┘   │                │ N
           │ 1               │      ┌─────────▼───────────┐
           │                 │      │        orders         │
           │ N               │      ├───────────────────────┤
┌──────────▼────────────┐    │      │ id             PK     │
│      order_items        │◄──┘      │ user_id        FK     │
├─────────────────────────┤          │ status                │
│ id              PK      │          │ total_cents           │
│ order_id        FK ─────┼──────────┤ created_at            │
│ product_id      FK      │   1    N └───────────────────────┘
│ quantity                │
│ unit_price_cents        │
└─────────────────────────┘
```

อ่านความสัมพันธ์จากไดอะแกรมนี้:

- **1 Category ↔ N Products**: หนึ่งหมวดหมู่มีสินค้าได้หลายชิ้น แต่หนึ่งสินค้าอยู่ได้แค่หมวดหมู่
  เดียว (`products.category_id` เป็น Foreign Key ชี้กลับไปที่ `categories.id`)
- **1 User ↔ N Orders**: หนึ่งผู้ใช้สั่งซื้อได้หลายครั้ง แต่หนึ่ง Order เป็นของผู้ใช้คนเดียวเสมอ
- **1 Order ↔ N OrderItems** และ **1 Product ↔ N OrderItems**: `order_items` คือตารางที่แก้ปัญหา
  ความสัมพันธ์แบบ **Many-to-Many** ระหว่าง Order กับ Product (หนึ่ง Order มีสินค้าได้หลายชิ้น
  และหนึ่งสินค้าก็ปรากฏอยู่ใน Order ได้หลายใบ) — นี่คือแพทเทิร์นมาตรฐานที่เรียกว่า
  **Junction Table** หรือ **Associative Entity** ในทฤษฎีฐานข้อมูลเชิงสัมพันธ์

### ทำไม schema นี้ผ่านการ Normalize ถูกต้อง (3NF)

หลักการ Normalization ที่สำคัญที่สุดที่ schema นี้ยึดถือคือการ **ไม่เก็บข้อมูลซ้ำซ้อนที่คำนวณ
ได้จากที่อื่น** ยกเว้นกรณีที่จำเป็นทางธุรกิจ จุดที่ควรสังเกตเป็นพิเศษ:

1. **`order_items.unit_price_cents` ไม่ใช่ข้อมูลซ้ำซ้อนที่ควรตัดทิ้ง** — แม้ราคาปัจจุบันของ
   สินค้าจะอยู่ใน `products.price_cents` อยู่แล้ว แต่เราต้อง **บันทึกราคา ณ เวลาที่สั่งซื้อ**
   แยกไว้ในแต่ละ `order_item` เพราะถ้า admin ขึ้นราคาสินค้าในวันถัดไป ใบเสร็จของคำสั่งซื้อเก่า
   ต้องยังแสดงราคาที่ลูกค้าจ่ายจริงตอนนั้น ไม่ใช่ราคาปัจจุบัน — นี่คือตัวอย่างคลาสสิกของการที่
   "การ denormalize บางจุดคือการตัดสินใจที่ถูกต้องทางธุรกิจ" ไม่ใช่ความผิดพลาดในการออกแบบ
2. **`orders.total_cents` ก็เป็นข้อมูลที่คำนวณได้จาก `SUM(order_items.quantity *
   order_items.unit_price_cents)`** แต่เราเลือกเก็บไว้ตรงๆ เพื่อไม่ต้อง `JOIN` และคำนวณใหม่
   ทุกครั้งที่แสดงประวัติคำสั่งซื้อ (ดู 121.5 ว่าค่านี้ถูกคำนวณและบันทึกพร้อมกันใน transaction
   เดียวกับการสร้าง order อย่างไร เพื่อไม่ให้ข้อมูลไม่ตรงกัน)
3. ไม่มีคอลัมน์ไหนที่เก็บค่าที่ **ไม่ขึ้นกับ primary key ทั้งหมดของแถวนั้น** (ตรงตามนิยามของ 3NF)
   เช่น `products` ไม่มีคอลัมน์ `category_name` ซ้ำอยู่ในตัวมันเอง (ต้อง `JOIN` กับ `categories`
   เพื่อเอาชื่อหมวดหมู่มาแสดงเสมอ — ดู 121.4)

### schema.sql ฉบับเต็ม

```sql
-- schema.sql — Schema ของ E-Commerce API
DROP TABLE IF EXISTS order_items CASCADE;
DROP TABLE IF EXISTS orders CASCADE;
DROP TABLE IF EXISTS products CASCADE;
DROP TABLE IF EXISTS categories CASCADE;
DROP TABLE IF EXISTS users CASCADE;

CREATE TABLE categories (
    id          SERIAL PRIMARY KEY,
    name        TEXT NOT NULL UNIQUE,
    slug        TEXT NOT NULL UNIQUE,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id            SERIAL PRIMARY KEY,
    username      TEXT NOT NULL UNIQUE,
    password_hash TEXT NOT NULL,
    role          TEXT NOT NULL DEFAULT 'customer' CHECK (role IN ('customer', 'admin')),
    created_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE products (
    id           SERIAL PRIMARY KEY,
    category_id  INTEGER NOT NULL REFERENCES categories(id) ON DELETE RESTRICT,
    sku          TEXT NOT NULL UNIQUE,
    name         TEXT NOT NULL,
    description  TEXT NOT NULL DEFAULT '',
    price_cents  BIGINT NOT NULL CHECK (price_cents >= 0),
    stock        INTEGER NOT NULL CHECK (stock >= 0),
    created_at   TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_products_category_id ON products(category_id);
CREATE INDEX idx_products_name_trgm ON products USING btree (lower(name));

CREATE TABLE orders (
    id            SERIAL PRIMARY KEY,
    user_id       INTEGER NOT NULL REFERENCES users(id) ON DELETE RESTRICT,
    status        TEXT NOT NULL DEFAULT 'placed' CHECK (status IN ('placed', 'cancelled')),
    total_cents   BIGINT NOT NULL CHECK (total_cents >= 0),
    created_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_orders_user_id ON orders(user_id);

CREATE TABLE order_items (
    id                SERIAL PRIMARY KEY,
    order_id          INTEGER NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
    product_id        INTEGER NOT NULL REFERENCES products(id) ON DELETE RESTRICT,
    quantity          INTEGER NOT NULL CHECK (quantity > 0),
    unit_price_cents  BIGINT NOT NULL CHECK (unit_price_cents >= 0)
);

CREATE INDEX idx_order_items_order_id ON order_items(order_id);
CREATE INDEX idx_order_items_product_id ON order_items(product_id);
```

### อธิบายการตัดสินใจออกแบบทีละจุด

**ทำไมเก็บราคาเป็น `price_cents` (จำนวนเต็ม หน่วยสตางค์) แทนที่จะเป็น `NUMERIC(10,2)` แบบ
Part 106**

นี่คือแพทเทิร์นที่เรียกว่า **"Never use floating point for money"** ในระดับที่เข้มงวดกว่า Part
106 อีกขั้น — Part 106 ใช้ `NUMERIC(10,2)` ซึ่งถูกต้องและปลอดภัยกว่า `FLOAT`/`DOUBLE` อยู่แล้ว
(ไม่มีปัญหา floating-point rounding error) แต่ในโปรเจกต์นี้เราเลือกเก็บเป็น **จำนวนเต็มหน่วย
สตางค์** (`BIGINT`) ไปเลย เพราะ:

- การคำนวณด้วยจำนวนเต็ม (`quantity * unit_price_cents`) ไม่มีทางเกิด rounding error ใดๆ
  เลยแม้แต่น้อย ต่างจาก `NUMERIC` ที่ยังต้องระวังเรื่อง scale ตอนคูณ/หารในบาง query ที่ซับซ้อน
- ฝั่ง client (เว็บ/มือถือ) ที่ได้ `price_cents: 129000` ไปสามารถหารด้วย 100 แล้ว format เป็น
  "1,290.00 บาท" ได้ตรงไปตรงมา ไม่ต้องกังวลเรื่อง locale การแสดงทศนิยมที่ต่างกันในแต่ละภาษา
  โปรแกรม

| แนวทางเก็บเงิน | ตัวอย่าง | ปัญหา |
|---|---|---|
| `FLOAT`/`DOUBLE` | `1290.00` | **ห้ามใช้เด็ดขาด** — `0.1 + 0.2 != 0.3` ใน floating point |
| `NUMERIC(10,2)` (Part 106) | `1290.00` | ปลอดภัย แต่ syntax การคำนวณในบาง client library ยุ่งยากกว่า |
| **จำนวนเต็มหน่วยสตางค์ (บทนี้)** | `129000` | ปลอดภัย 100%, คำนวณเป็นจำนวนเต็มล้วน, ต้องหารด้วย 100 ตอนแสดงผลเท่านั้น |

**ทำไม `ON DELETE RESTRICT` สำหรับ `products.category_id` แต่ `ON DELETE CASCADE` สำหรับ
`order_items.order_id`**

นี่คือการตัดสินใจเชิงธุรกิจที่สำคัญมากซึ่งฝังอยู่ใน schema โดยตรง:

- `products.category_id REFERENCES categories(id) ON DELETE RESTRICT`: ห้ามลบ Category ถ้ายัง
  มีสินค้าผูกอยู่ — เพราะการลบ Category ที่มีสินค้าอยู่ควรเป็นการตัดสินใจที่ทำโดยตั้งใจ (ย้าย
  สินค้าไปหมวดอื่นก่อน) ไม่ใช่เกิดขึ้นโดยไม่ตั้งใจเป็นผลข้างเคียง
- `order_items.order_id REFERENCES orders(id) ON DELETE CASCADE`: ถ้า Order ถูกลบ (กรณีพิเศษ
  เช่น ทดสอบระบบ หรือลบข้อมูลตาม policy การเก็บข้อมูล) รายการสินค้าในนั้นก็ไม่มีความหมายอีก
  ต่อไป ควรถูกลบตามไปด้วยอัตโนมัติ เพราะ `order_items` เป็นส่วนหนึ่งของ Order เสมอ ไม่มีความหมาย
  อยู่ตัวเดียวลอยๆ
- `order_items.product_id REFERENCES products(id) ON DELETE RESTRICT`: ห้ามลบสินค้าที่เคยถูก
  สั่งซื้อไปแล้ว เพราะจะทำให้ประวัติคำสั่งซื้อเก่าขาดข้อมูลอ้างอิงไป (ในระบบจริงมักใช้ "soft
  delete" คือมีคอลัมน์ `is_active` แทนการลบจริง ซึ่งเป็นแบบฝึกหัดข้อ 5 ท้ายบท)

**Index ที่เพิ่มเข้ามาและเหตุผล**

| Index | เหตุผล |
|---|---|
| `idx_products_category_id` | `GET /products?category_id=X` filter บ่อยมาก ถ้าไม่มี index PostgreSQL ต้อง Sequential Scan ทั้งตาราง |
| `idx_products_name_trgm` (บน `lower(name)`) | รองรับการค้นหาสินค้าแบบ case-insensitive (`ILIKE` / `LIKE lower(...)`) ให้เร็วขึ้น |
| `idx_orders_user_id` | `GET /orders` (ประวัติของผู้ใช้คนหนึ่ง) เป็น query ที่เรียกบ่อยที่สุดของระบบ |
| `idx_order_items_order_id` | โหลดรายการสินค้าในแต่ละ order (JOIN บ่อยมากตอนแสดงรายละเอียด order) |
| `idx_order_items_product_id` | ใช้เมื่อต้องการดูว่าสินค้าตัวหนึ่งถูกสั่งซื้อไปในออเดอร์ไหนบ้าง (เช่น รายงานยอดขาย) |

Primary Key ทุกตัว (`id SERIAL PRIMARY KEY`) มี index มาให้อัตโนมัติอยู่แล้วจาก PostgreSQL
เช่นเดียวกับ Foreign Key แต่ **PostgreSQL ไม่ได้สร้าง index ให้อัตโนมัติที่ฝั่ง FK เอง** (ต่างจาก
Primary Key) นี่คือเหตุผลที่เราต้องสร้าง `idx_products_category_id`, `idx_orders_user_id` และ
Index อื่นๆ ที่อยู่บนคอลัมน์ FK เองตรงๆ

### สร้าง Role, Database และรัน Schema จริง

```bash
$ pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
16  main    5432 down   postgres /var/lib/postgresql/16/main /var/log/postgresql/postgresql-16-main.log

$ pg_ctlcluster 16 main start
$ pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
16  main    5432 online postgres /var/lib/postgresql/16/main /var/log/postgresql/postgresql-16-main.log
```

```bash
$ sudo -u postgres psql <<'EOF'
DROP DATABASE IF EXISTS ecommercedb;
DROP ROLE IF EXISTS ecomuser;
CREATE ROLE ecomuser WITH LOGIN PASSWORD 'ecom_pass123';
CREATE DATABASE ecommercedb OWNER ecomuser;
EOF
```

ผลลัพธ์จริง:

```
NOTICE:  database "ecommercedb" does not exist, skipping
DROP DATABASE
NOTICE:  role "ecomuser" does not exist, skipping
DROP ROLE
CREATE ROLE
CREATE DATABASE
```

รัน `schema.sql` จริงผ่าน TCP (แบบเดียวกับที่โปรแกรม C++ จะเชื่อมต่อ):

```bash
$ PGPASSWORD=ecom_pass123 psql -h 127.0.0.1 -U ecomuser -d ecommercedb -f schema.sql
```

ผลลัพธ์จริง:

```
NOTICE:  table "order_items" does not exist, skipping
DROP TABLE
NOTICE:  table "orders" does not exist, skipping
DROP TABLE
NOTICE:  table "products" does not exist, skipping
DROP TABLE
NOTICE:  table "categories" does not exist, skipping
DROP TABLE
NOTICE:  table "users" does not exist, skipping
DROP TABLE
CREATE TABLE
CREATE TABLE
CREATE TABLE
CREATE INDEX
CREATE INDEX
CREATE TABLE
CREATE INDEX
CREATE TABLE
CREATE INDEX
CREATE INDEX
```

Schema ถูกสร้างสำเร็จครบทั้ง 5 ตารางและ 5 Index บน PostgreSQL 16.15 จริง

---

## 121.3 โครงสร้างโปรเจกต์แบบ Layered Architecture (Step 963)

### โครงสร้างไฟล์ทั้งหมด

ต่อยอดจากแพทเทิร์น Layered Architecture ของ Part 108 แต่ขยายเป็นหลาย Model/Route เพราะมีหลาย
resource ที่สัมพันธ์กัน:

```
ecommerce_api/
├── CMakeLists.txt
├── Dockerfile
├── docker-compose.yml
├── .dockerignore
├── schema.sql
├── seed_categories.sql
├── include/
│   ├── models.hpp             # struct Category, Product, User, Order, OrderItemLine
│   ├── connection_pool.hpp    # ConnectionPool: pool ของ pqxx::connection ที่ thread-safe
│   ├── db.hpp                 # Data Access Layer: class Db (repository pattern)
│   ├── response.hpp           # ฟังก์ชันช่วยสร้าง JSON envelope (สืบทอดจาก Part 108)
│   ├── jwt.hpp                # JWT sign/verify (สืบทอดจาก Part 110)
│   ├── password.hpp           # Argon2id hash/verify (สืบทอดจาก Part 110)
│   ├── auth_middleware.hpp    # AuthMiddleware: ตรวจ JWT + เก็บ role ไว้ใน context
│   ├── routes_catalog.hpp     # ประกาศ register_catalog_routes() (categories + products)
│   ├── routes_auth.hpp        # ประกาศ register_auth_routes() (register/login)
│   └── routes_orders.hpp      # ประกาศ register_order_routes() (orders)
└── src/
    ├── main.cpp                # จุดเริ่มต้นโปรแกรม ประกอบทุกชิ้นเข้าด้วยกัน
    ├── db.cpp                  # Implementation ของ Db (คุยกับ PostgreSQL จริงผ่าน libpqxx)
    ├── jwt.cpp                 # Implementation ของ JWT HS256
    ├── password.cpp             # Implementation ของ Argon2id ผ่าน libsodium
    ├── routes_catalog.cpp       # Implementation ของ /categories, /products/*
    ├── routes_auth.cpp          # Implementation ของ /auth/*
    └── routes_orders.cpp        # Implementation ของ /orders/*
```

### ตารางความรับผิดชอบของแต่ละชั้น

| ชั้น (Layer) | ไฟล์ | หน้าที่ | รู้จัก Crow? | รู้จัก libpqxx? |
|---|---|---|---|---|
| Model | `models.hpp` | นิยามรูปร่างข้อมูลของแต่ละ entity + แปลงเป็น JSON | ไม่รู้จัก | ไม่รู้จัก |
| Connection Pool | `connection_pool.hpp` | จัดการ `pqxx::connection` หลายตัวให้ thread-safe | ไม่รู้จัก | รู้จัก (รู้จักเดียว) |
| Data Access Layer | `db.hpp`/`db.cpp` | อ่าน/เขียน PostgreSQL, business logic เชิงข้อมูล | ไม่รู้จัก | รู้จัก |
| Auth (JWT + Password) | `jwt.*`, `password.*` | ตรรกะ crypto ล้วนๆ | ไม่รู้จัก | ไม่รู้จัก |
| Middleware | `auth_middleware.hpp` | เชื่อม JWT เข้ากับ HTTP request/response ของ Crow | รู้จัก | ไม่รู้จัก |
| Routes | `routes_*.hpp/cpp` | รับ request, validate, เรียก DAL, คืน response | รู้จัก | ไม่รู้จักโดยตรง (ผ่าน `Db`) |
| Entry point | `main.cpp` | ประกอบทุกชิ้น, อ่าน config จาก env, เปิด server | รู้จัก | รู้จัก (สร้าง `Db`) |

การแยกชั้นนี้ทำให้ **ถ้าจะเปลี่ยนจาก PostgreSQL ไปเป็นฐานข้อมูลอื่นในอนาคต แก้แค่ `db.cpp` กับ
`connection_pool.hpp` เท่านั้น** โดย `routes_*.cpp` ทั้งหมดไม่ต้องแตะเลย เพราะคุยกับ `Db` ผ่าน
interface (`list_products()`, `place_order()`, ...) ไม่เคยเห็น `pqxx::` ตรงๆ เลย

### CMakeLists.txt

```cmake
cmake_minimum_required(VERSION 3.16)
project(ecommerce_api CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
add_compile_options(-Wall -Wextra)

find_package(Threads REQUIRED)
find_package(OpenSSL REQUIRED)
find_package(PkgConfig REQUIRED)
pkg_check_modules(SODIUM REQUIRED libsodium)
pkg_check_modules(PQXX REQUIRED libpqxx)

add_executable(ecommerce_api
    src/main.cpp
    src/db.cpp
    src/routes_catalog.cpp
    src/routes_auth.cpp
    src/routes_orders.cpp
    src/jwt.cpp
    src/password.cpp
)

target_include_directories(ecommerce_api PRIVATE
    include
    /usr/local/include
    ${SODIUM_INCLUDE_DIRS}
    ${PQXX_INCLUDE_DIRS}
)

target_link_libraries(ecommerce_api PRIVATE
    Threads::Threads
    OpenSSL::SSL
    OpenSSL::Crypto
    ${SODIUM_LIBRARIES}
    ${PQXX_LIBRARIES}
)
```

สังเกตว่าเราใช้ `pkg_check_modules(REQUIRED libpqxx)` แทน `find_package(PQXX)` เพราะ libpqxx
เวอร์ชันที่ apt ติดตั้งให้บนเครื่องนี้ไม่มี CMake config file มาให้ (`PQXXConfig.cmake`) แต่มี
`.pc` file ให้ `pkg-config` ใช้แทน — นี่คือเหตุผลที่ต้อง `find_package(PkgConfig REQUIRED)`
ก่อนเสมอเมื่อ library เป้าหมายไม่รองรับ CMake โดยตรง (ทบทวนแนวคิดนี้จาก Part 89 เรื่อง CMake
ขั้นสูง)

**Build ครั้งแรก:**

```bash
$ cmake -S . -B build -DCMAKE_BUILD_TYPE=Release
```

ผลลัพธ์จริง:

```
-- The CXX compiler identification is GNU 13.3.0
-- Detecting CXX compiler ABI info
-- Detecting CXX compiler ABI info - done
-- Check for working CXX compiler: /usr/bin/c++ - skipped
-- Detecting CXX compile features
-- Detecting CXX compile features - done
-- Performing Test CMAKE_HAVE_LIBC_PTHREAD
-- Performing Test CMAKE_HAVE_LIBC_PTHREAD - Success
-- Found Threads: TRUE
-- Found OpenSSL: /usr/lib/x86_64-linux-gnu/libcrypto.so (found version "3.0.13")
-- Found PkgConfig: /usr/bin/pkg-config (found version "1.8.1")
-- Checking for module 'libsodium'
--   Found libsodium, version 1.0.18
-- Checking for module 'libpqxx'
--   Found libpqxx, version 7.8.1
-- Configuring done (0.4s)
-- Generating done (0.0s)
-- Build files have been written to: .../ecommerce_api/build
```

```bash
$ cmake --build build -j4
```

ผลลัพธ์จริง:

```
[ 12%] Building CXX object CMakeFiles/ecommerce_api.dir/src/main.cpp.o
[ 25%] Building CXX object CMakeFiles/ecommerce_api.dir/src/routes_catalog.cpp.o
[ 37%] Building CXX object CMakeFiles/ecommerce_api.dir/src/routes_auth.cpp.o
[ 50%] Building CXX object CMakeFiles/ecommerce_api.dir/src/db.cpp.o
[ 62%] Building CXX object CMakeFiles/ecommerce_api.dir/src/routes_orders.cpp.o
[ 75%] Building CXX object CMakeFiles/ecommerce_api.dir/src/jwt.cpp.o
[ 87%] Building CXX object CMakeFiles/ecommerce_api.dir/src/password.cpp.o
[100%] Linking CXX executable ecommerce_api
[100%] Built target ecommerce_api
```

Build ผ่านสะอาด **ไม่มี warning แม้แต่บรรทัดเดียว** ด้วย `-Wall -Wextra` เปิดอยู่ตลอด — ทุกไฟล์
ในโปรเจกต์นี้ผ่านมาตรฐานเดียวกับที่กำหนดไว้ใน Part 1 ("กฎทองของหลักสูตร: ห้ามคอมไพล์โค้ดโดยไม่
เปิด `-Wall -Wextra`") ตลอดทั้งบทเรียน

### Model Layer: `models.hpp`

```cpp
#pragma once
#include <string>
#include <vector>
#include <nlohmann/json.hpp>

// ---------------------------------------------------------------------------
// Model layer: struct ธรรมดา ไม่รู้จัก Crow และไม่รู้จัก pqxx เลย
// ---------------------------------------------------------------------------

struct Category {
    int id = 0;
    std::string name;
    std::string slug;

    nlohmann::json to_json() const {
        return nlohmann::json{{"id", id}, {"name", name}, {"slug", slug}};
    }
};

struct Product {
    int id = 0;
    int category_id = 0;
    std::string category_name; // เติมตอน JOIN เพื่อลด N+1 query (ดู 121.4)
    std::string sku;
    std::string name;
    std::string description;
    long long price_cents = 0;
    int stock = 0;

    nlohmann::json to_json() const {
        return nlohmann::json{
            {"id", id},
            {"category_id", category_id},
            {"category_name", category_name},
            {"sku", sku},
            {"name", name},
            {"description", description},
            {"price_cents", price_cents},
            {"stock", stock}
        };
    }
};

struct User {
    int id = 0;
    std::string username;
    std::string password_hash;
    std::string role; // "customer" หรือ "admin"
};

struct OrderItemLine {
    int product_id = 0;
    std::string product_name;
    int quantity = 0;
    long long unit_price_cents = 0;

    nlohmann::json to_json() const {
        return nlohmann::json{
            {"product_id", product_id},
            {"product_name", product_name},
            {"quantity", quantity},
            {"unit_price_cents", unit_price_cents},
            {"line_total_cents", unit_price_cents * quantity}
        };
    }
};

struct Order {
    int id = 0;
    int user_id = 0;
    std::string status;
    long long total_cents = 0;
    std::string created_at;
    std::vector<OrderItemLine> items;

    nlohmann::json to_json() const {
        nlohmann::json items_json = nlohmann::json::array();
        for (const auto& it : items) items_json.push_back(it.to_json());
        return nlohmann::json{
            {"id", id},
            {"user_id", user_id},
            {"status", status},
            {"total_cents", total_cents},
            {"created_at", created_at},
            {"items", items_json}
        };
    }
};
```

จุดที่สำคัญที่สุดในไฟล์นี้คือ `line_total_cents` ใน `OrderItemLine::to_json()` — ค่านี้
**ไม่ได้ถูกเก็บในฐานข้อมูลเลย** แต่คำนวณสดตอนแปลงเป็น JSON (`unit_price_cents * quantity`)
เพราะเป็นค่าที่คำนวณได้เสมอจากสองคอลัมน์ที่มีอยู่แล้ว การเก็บมันซ้ำในฐานข้อมูลจะทำให้เกิดความ
เสี่ยงที่ข้อมูลไม่ตรงกัน (ถ้าแก้ `quantity` แล้วลืมแก้ `line_total_cents` ตาม) — นี่คือหลักการ
เดียวกับที่อธิบายไว้ใน 121.2 เรื่องการเลือกว่าอะไรควรเก็บจริง อะไรควรคำนวณสด

### Response Envelope: `response.hpp` (สืบทอดจาก Part 108 ตรงๆ)

```cpp
#pragma once
#include <string>
#include <nlohmann/json.hpp>
#include "crow.h"

namespace api {

inline crow::response ok(const nlohmann::json& data, int status = 200) {
    nlohmann::json body{{"success", true}, {"data", data}};
    crow::response res(status, body.dump());
    res.set_header("Content-Type", "application/json");
    return res;
}

inline crow::response error(int status, const std::string& code, const std::string& message) {
    nlohmann::json body{
        {"success", false},
        {"error", {{"code", code}, {"message", message}}}
    };
    crow::response res(status, body.dump());
    res.set_header("Content-Type", "application/json");
    return res;
}

} // namespace api
```

รูปแบบ envelope นี้เหมือนกับ Part 108 ทุกประการ — พิสูจน์ว่าแพทเทิร์นที่ดีสามารถนำไปใช้ซ้ำได้
ข้ามโปรเจกต์โดยไม่ต้องออกแบบใหม่ทุกครั้ง

---

## 121.4 Authentication ที่ใช้ซ้ำจาก Part 110 + ขยายเป็น Role-Based Access Control (Step 964)

### หลักการ: Reuse ไม่ใช่ Rewrite

Part 110 สอนการ implement JWT (HS256) ด้วยมือผ่าน OpenSSL และ Argon2id password hashing ผ่าน
libsodium ไว้อย่างละเอียดแล้ว Capstone นี้ **ไม่เขียนโค้ดส่วนนั้นใหม่** แต่คัดลอกไฟล์
`jwt.hpp`/`jwt.cpp` และ `password.hpp`/`password.cpp` มาใช้ตรงๆ — นี่คือพฤติกรรมที่ถูกต้องของ
วิศวกรมืออาชีพจริง: เมื่อมี building block ที่ผ่านการทดสอบแล้วว่าถูกต้องและปลอดภัย ไม่มีเหตุผลใด
ที่ต้องเขียนใหม่จากศูนย์ ความเสี่ยงจะเพิ่มขึ้นเปล่าๆ โดยไม่ได้ประโยชน์อะไรเพิ่ม

`include/password.hpp` (เหมือน Part 110 ทุกตัวอักษร):

```cpp
#pragma once
#include <string>

// ห่อหุ้มการ hash/verify password ด้วย libsodium (Argon2id) — สืบทอดจาก Part 110 ตรงๆ
namespace password {
std::string hash(const std::string& plain);
bool verify(const std::string& hashed, const std::string& plain);
} // namespace password
```

`src/password.cpp`:

```cpp
#include "password.hpp"
#include <sodium.h>
#include <stdexcept>
#include <array>

namespace password {

std::string hash(const std::string& plain) {
    if (sodium_init() < 0) {
        throw std::runtime_error("เริ่มต้น libsodium ไม่สำเร็จ");
    }
    std::array<char, crypto_pwhash_STRBYTES> out{};
    int rc = crypto_pwhash_str(
        out.data(),
        plain.c_str(), plain.size(),
        crypto_pwhash_OPSLIMIT_INTERACTIVE,
        crypto_pwhash_MEMLIMIT_INTERACTIVE
    );
    if (rc != 0) {
        throw std::runtime_error("hash password ไม่สำเร็จ (อาจเป็นเพราะหน่วยความจำไม่พอ)");
    }
    return std::string(out.data());
}

bool verify(const std::string& hashed, const std::string& plain) {
    if (sodium_init() < 0) {
        throw std::runtime_error("เริ่มต้น libsodium ไม่สำเร็จ");
    }
    return crypto_pwhash_str_verify(hashed.c_str(), plain.c_str(), plain.size()) == 0;
}

} // namespace password
```

`include/jwt.hpp`:

```cpp
#pragma once
#include <string>
#include <optional>
#include <nlohmann/json.hpp>

// JWT (HS256) — implement เองด้วย OpenSSL เป็น crypto primitive สืบทอดจาก Part 110 ตรงๆ
namespace jwt {
std::string sign(nlohmann::json payload, const std::string& secret);
std::optional<nlohmann::json> verify(const std::string& token, const std::string& secret);
} // namespace jwt
```

`src/jwt.cpp` เหมือนกับที่ implement ไว้ใน Part 110.4 ทุกประการ (Base64URL encode/decode,
HMAC-SHA256 ผ่าน OpenSSL, constant-time comparison ป้องกัน timing attack) จึงไม่ยกมาซ้ำในบทนี้
— ผู้เรียนที่ยังไม่คุ้นกับรายละเอียดควรย้อนกลับไปอ่าน Part 110.3–110.4 ก่อน

### สิ่งใหม่ที่ Capstone นี้เพิ่มเข้ามา: Role-Based Access Control (RBAC)

Part 110 มีแค่แนวคิด "login แล้วหรือยัง" (Authentication) แต่ไม่มีแนวคิด "login แล้วทำอะไรได้
บ้าง" (Authorization) — Capstone นี้ต้องแยกสิทธิ์ระหว่างลูกค้าทั่วไป (`customer`) กับผู้ดูแลระบบ
(`admin`) เพราะ endpoint อย่าง `POST /products` (สร้างสินค้าใหม่) ต้องสงวนไว้ให้ admin เท่านั้น

**หลักการสำคัญ**: role ของผู้ใช้ต้องถูกฝังไว้ใน JWT payload (`claims`) ตั้งแต่ตอน login แล้วให้
Middleware ถอดค่านี้ออกมาเก็บไว้ใน context ให้ route handler ตรวจสอบได้ทันทีโดยไม่ต้อง query
ฐานข้อมูลซ้ำ — สอดคล้องกับหลักการ **Stateless Authentication** ของ JWT ที่อธิบายไว้ใน Part 110.3

`users` table มีคอลัมน์ `role TEXT NOT NULL DEFAULT 'customer' CHECK (role IN ('customer',
'admin'))` (ดู 121.2) — การใช้ `CHECK` constraint ที่ระดับฐานข้อมูลป้องกันไม่ให้ค่า role ที่ผิด
เพี้ยน (เช่น พิมพ์ผิดเป็น `"admni"`) หลุดเข้าไปในฐานข้อมูลได้เลย แม้แต่จาก bug ในโค้ด C++ เอง

### `include/auth_middleware.hpp`

```cpp
#pragma once
#include "crow.h"
#include "jwt.hpp"
#include <cstdlib>
#include <string>
#include <nlohmann/json.hpp>

inline std::string jwt_secret() {
    const char* env = std::getenv("JWT_SECRET");
    if (env && std::string(env).size() > 0) {
        return env;
    }
    return "dev-only-secret-change-me-in-production";
}

// AuthMiddleware — ตรวจสอบ JWT จาก header "Authorization: Bearer <token>"
// เก็บทั้ง user_id และ role ไว้ใน context เพื่อให้ route ปลายทางตรวจสอบสิทธิ์ admin ได้
struct AuthMiddleware : crow::ILocalMiddleware {
    struct context {
        int user_id = 0;
        std::string username;
        std::string role;
    };

    void before_handle(crow::request& req, crow::response& res, context& ctx) {
        std::string auth_header = req.get_header_value("Authorization");
        const std::string prefix = "Bearer ";

        if (auth_header.size() <= prefix.size() ||
            auth_header.compare(0, prefix.size(), prefix) != 0) {
            reject(res, "MISSING_TOKEN", "ต้องแนบ header Authorization: Bearer <token>");
            return;
        }

        std::string token = auth_header.substr(prefix.size());
        auto payload = jwt::verify(token, jwt_secret());
        if (!payload) {
            reject(res, "INVALID_TOKEN", "Token ไม่ถูกต้องหรือหมดอายุแล้ว");
            return;
        }

        // เก็บข้อมูลผู้ใช้ไว้ใน context เพื่อให้ handler ปลายทางใช้ต่อได้
        ctx.user_id = (*payload).value("sub", 0);
        ctx.username = (*payload).value("username", "");
        ctx.role = (*payload).value("role", "customer");
    }

    void after_handle(crow::request&, crow::response&, context&) {}

private:
    void reject(crow::response& res, const std::string& code, const std::string& message) {
        nlohmann::json body{
            {"success", false},
            {"error", {{"code", code}, {"message", message}}}
        };
        res.code = 401;
        res.set_header("Content-Type", "application/json");
        res.write(body.dump());
        res.end();
    }
};
```

ต่างจาก Part 110 ตรงที่ `context` มีฟิลด์ `role` เพิ่มเข้ามา — เมื่อ route handler เรียก
`app.get_context<AuthMiddleware>(req).role` จะได้ค่า role ที่ถอดออกมาจาก JWT ทันที โดยไม่ต้อง
query ตาราง `users` ซ้ำเลย (ราคาที่ต้องจ่ายคือถ้า admin ถูกลดสิทธิ์กลางคัน token เก่าที่ออกไป
แล้วจะยัง "จำ" สิทธิ์เดิมไว้จนกว่าจะหมดอายุ — ดู Common Pitfalls ข้อ 6 ท้ายบท)

### `src/routes_auth.cpp`: `/auth/register` และ `/auth/login`

```cpp
#include "routes_auth.hpp"
#include "response.hpp"
#include "password.hpp"
#include "jwt.hpp"
#include <nlohmann/json.hpp>
#include <chrono>

using json = nlohmann::json;

static std::string validate_credentials(const json& body) {
    if (!body.contains("username") || !body["username"].is_string()) {
        return "ต้องมีฟิลด์ \"username\" เป็น string";
    }
    if (!body.contains("password") || !body["password"].is_string()) {
        return "ต้องมีฟิลด์ \"password\" เป็น string";
    }
    std::string username = body["username"].get<std::string>();
    std::string pw = body["password"].get<std::string>();
    if (username.empty() || username.size() > 50) {
        return "username ต้องมีความยาว 1-50 ตัวอักษร";
    }
    if (pw.size() < 8) {
        return "password ต้องมีความยาวอย่างน้อย 8 ตัวอักษร";
    }
    return "";
}

void register_auth_routes(crow::App<AuthMiddleware>& app, Db& db) {

    // POST /auth/register — สมัครสมาชิกใหม่
    // หมายเหตุความปลอดภัย: endpoint สาธารณะนี้สร้างได้แค่บัญชี role="customer" เท่านั้น
    // ห้ามให้ client กำหนด role ของตัวเองมาใน request เด็ดขาด (ดู Common Pitfalls ข้อ
    // Privilege Escalation) บัญชี admin ตัวแรกต้องสร้างผ่านการ seed ฐานข้อมูลโดยตรงเท่านั้น
    CROW_ROUTE(app, "/auth/register")
    .methods(crow::HTTPMethod::POST)
    ([&db](const crow::request& req) {
        json body;
        try {
            body = json::parse(req.body);
        } catch (const json::parse_error&) {
            return api::error(400, "INVALID_JSON", "Body ไม่ใช่ JSON ที่ถูกต้อง");
        }

        std::string err = validate_credentials(body);
        if (!err.empty()) {
            return api::error(400, "VALIDATION_ERROR", err);
        }

        std::string username = body["username"].get<std::string>();
        std::string plain_password = body["password"].get<std::string>();

        try {
            std::string hashed = password::hash(plain_password);
            auto user = db.create_user(username, hashed, "customer");
            if (!user) {
                return api::error(409, "USERNAME_TAKEN",
                    "ชื่อผู้ใช้ \"" + username + "\" ถูกใช้ไปแล้ว");
            }
            return api::ok(json{{"id", user->id}, {"username", user->username},
                                 {"role", user->role}}, 201);
        } catch (const std::exception& e) {
            return api::error(500, "INTERNAL_ERROR", e.what());
        }
    });

    // POST /auth/login — ตรวจสอบ username/password แล้วออก JWT (ฝัง role ไว้ใน claims)
    CROW_ROUTE(app, "/auth/login")
    .methods(crow::HTTPMethod::POST)
    ([&db](const crow::request& req) {
        json body;
        try {
            body = json::parse(req.body);
        } catch (const json::parse_error&) {
            return api::error(400, "INVALID_JSON", "Body ไม่ใช่ JSON ที่ถูกต้อง");
        }

        std::string err = validate_credentials(body);
        if (!err.empty()) {
            return api::error(400, "VALIDATION_ERROR", err);
        }

        std::string username = body["username"].get<std::string>();
        std::string plain_password = body["password"].get<std::string>();

        try {
            auto user = db.find_user_by_username(username);
            // จงใจคืน error message เดียวกันเสมอ ป้องกัน User Enumeration Attack (Part 110)
            if (!user || !password::verify(user->password_hash, plain_password)) {
                return api::error(401, "INVALID_CREDENTIALS", "username หรือ password ไม่ถูกต้อง");
            }

            long long now = std::chrono::duration_cast<std::chrono::seconds>(
                std::chrono::system_clock::now().time_since_epoch()).count();

            json claims{
                {"sub", user->id},
                {"username", user->username},
                {"role", user->role},
                {"exp", now + 3600}
            };
            std::string token = jwt::sign(claims, jwt_secret());

            return api::ok(json{
                {"access_token", token},
                {"token_type", "Bearer"},
                {"expires_in", 3600},
                {"role", user->role}
            });
        } catch (const std::exception& e) {
            return api::error(500, "INTERNAL_ERROR", e.what());
        }
    });
}
```

### สร้าง Admin คนแรก: ทำไมต้อง Seed ผ่านฐานข้อมูลโดยตรง ไม่ใช่ผ่าน API

จุดที่ต้องระวังมากที่สุดจุดหนึ่งของระบบ RBAC คือ **"ไข่กับไก่อะไรเกิดก่อนกัน"** — ถ้า
`POST /auth/register` อนุญาตให้ client ส่ง `{"role": "admin"}` มาเองได้ ใครก็สามารถสมัครเป็น
admin ได้ทันทีโดยไม่ต้องรับอนุญาตจากใคร (นี่คือช่องโหว่ **Privilege Escalation** ที่ร้ายแรงมาก)

ทางแก้ในบทเรียนนี้ (และในระบบ production จำนวนมาก) คือ **บังคับให้ `/auth/register` สร้างได้แค่
`role = "customer"` เสมอ** ไม่มีทางเลือกอื่น ส่วนบัญชี admin คนแรกต้องถูกสร้างผ่านการเข้าถึง
ฐานข้อมูลโดยตรง (สิทธิ์ที่มีแค่ทีม DevOps/DBA เท่านั้นที่เข้าถึงได้) — ทดสอบจริง:

```bash
$ curl -s -X POST http://127.0.0.1:18480/auth/register -H "Content-Type: application/json" \
  -d '{"username":"adminuser","password":"AdminP@ss1"}'
{"data":{"id":2,"role":"customer","username":"adminuser"},"success":true}
```

สังเกตว่าแม้จะตั้งชื่อผู้ใช้ว่า `adminuser` แต่ role ที่ได้กลับมาคือ `"customer"` เสมอ — ชื่อ
ไม่มีผลต่อสิทธิ์เลย จากนั้นโปรโมทเป็น admin ผ่าน `psql` โดยตรง:

```bash
$ PGPASSWORD=ecom_pass123 psql -h 127.0.0.1 -U ecomuser -d ecommercedb \
  -c "UPDATE users SET role='admin' WHERE username='adminuser';"
UPDATE 1

$ PGPASSWORD=ecom_pass123 psql -h 127.0.0.1 -U ecomuser -d ecommercedb \
  -c "SELECT id, username, role FROM users;"
 id | username  |   role
----+-----------+----------
  1 | somchai   | customer
  2 | adminuser | admin
(2 rows)
```

จากนั้น login ใหม่อีกครั้งเพื่อรับ JWT ที่มี `"role":"admin"` ฝังอยู่ (JWT เก่าที่ออกไปก่อนหน้า
ยังมี role เดิมค้างอยู่จนกว่าจะหมดอายุ — นี่คือเหตุผลที่ระบบ production จริงมักตั้ง `exp` ของ
JWT ให้สั้น เช่น 15–60 นาที ไม่ใช่หลายวัน):

```bash
$ curl -s -X POST http://127.0.0.1:18480/auth/login -H "Content-Type: application/json" \
  -d '{"username":"adminuser","password":"AdminP@ss1"}'
```

ผลลัพธ์จริง:

```json
{"data":{"access_token":"eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJleHAiOjE3OTA0Mjk0MzEsImlhdCI6MTc5MDQyNTgzMSwicm9sZSI6ImFkbWluIiwic3ViIjoyLCJ1c2VybmFtZSI6ImFkbWludXNlciJ9.Ha0DjBnoDoxM5Uq6yZKNyY1xn64uQ7J-u7u5MwC569o","expires_in":3600,"role":"admin","token_type":"Bearer"},"success":true}
```

ถอดรหัส payload ของ JWT นี้ (Base64URL decode ส่วนกลาง) จะได้
`{"exp":1790429431,"iat":1790425831,"role":"admin","sub":2,"username":"adminuser"}` — เห็นได้
ชัดว่า `"role":"admin"` ถูกฝังอยู่ในนั้นแล้ว พร้อมให้ `AuthMiddleware` ถอดออกมาใช้ตรวจสอบสิทธิ์
ในทุก request ถัดไป

---
