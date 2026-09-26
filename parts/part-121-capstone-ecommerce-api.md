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

## 121.5 Catalog API: Category, Product, Filter/Search/Pagination และปัญหา N+1 (Step 965)

### `include/connection_pool.hpp`: เกริ่นนำก่อนเข้ารายละเอียด

`db.cpp` ในหัวข้อนี้จะเรียกใช้ `ConnectionPool` ซึ่งเป็นคลาสที่แก้ปัญหา concurrency bug จริงที่
เจอระหว่างพัฒนาโปรเจกต์นี้ (รายละเอียดเต็มอยู่ใน 121.6) ในหัวข้อนี้ให้เข้าใจแค่ว่า
`pool_.acquire()` คืน RAII guard ที่ให้ยืม `pqxx::connection` หนึ่งตัวมาใช้ชั่วคราว แล้วคืนกลับ
อัตโนมัติเมื่อหมด scope — ใช้งานเหมือนมี `pqxx::connection` ตัวเดียวตรงๆ แทบทุกประการ

### `include/db.hpp`: Data Access Layer เต็มรูปแบบ

```cpp
#pragma once
#include <pqxx/pqxx>
#include <string>
#include <vector>
#include <optional>
#include <stdexcept>
#include "models.hpp"
#include "connection_pool.hpp"

// โยน exception ชนิดนี้เมื่อสินค้าตัวใดตัวหนึ่งใน order มีสต็อกไม่พอ
// route layer จะจับ exception นี้แยกจาก exception ทั่วไป เพื่อคืน 409 Conflict แทน 500
struct InsufficientStockError : public std::runtime_error {
    int product_id;
    std::string product_name;
    int requested;
    int available;
    InsufficientStockError(int pid, std::string pname, int req, int avail)
        : std::runtime_error("สินค้า \"" + pname + "\" มีไม่พอ (ต้องการ " +
                              std::to_string(req) + " มีในสต็อก " + std::to_string(avail) + ")"),
          product_id(pid), product_name(std::move(pname)), requested(req), available(avail) {}
};

struct ProductNotFoundError : public std::runtime_error {
    int product_id;
    explicit ProductNotFoundError(int pid)
        : std::runtime_error("ไม่พบสินค้า id = " + std::to_string(pid)), product_id(pid) {}
};

struct OrderLineRequest {
    int product_id;
    int quantity;
};

// รายการค้นหา/กรองสินค้า
struct ProductQuery {
    std::optional<int> category_id;
    std::optional<std::string> search;
    int page = 1;
    int limit = 20;
};

struct ProductPage {
    std::vector<Product> items;
    long long total_count = 0;
    int page = 1;
    int limit = 20;
    long long total_pages = 0;
};

// Db คือชั้น Data Access Layer (Repository) — ห่อหุ้มการคุยกับ PostgreSQL ทั้งหมดผ่าน libpqxx
// ชั้น routes จะไม่เห็น pqxx::connection หรือ SQL string เลยแม้แต่นิดเดียว
class Db {
public:
    // pool_size คือจำนวน connection ที่เปิดค้างไว้ล่วงหน้า ควรตั้งให้ >= จำนวน worker thread
    // ของ Crow (ค่า default ของ .multithreaded() คือ std::thread::hardware_concurrency())
    // เพื่อไม่ให้ request ต้องรอคิวกันเปล่าๆ ทั้งที่ CPU core ยังว่างอยู่
    explicit Db(const std::string& conn_string, std::size_t pool_size = 8);

    // --- Categories ---
    std::vector<Category> list_categories();

    // --- Products ---
    ProductPage list_products(const ProductQuery& q);
    std::optional<Product> find_product_by_id(int id);
    Product create_product(int category_id, const std::string& sku, const std::string& name,
                            const std::string& description, long long price_cents, int stock);
    bool update_product(int id, const std::string& name, const std::string& description,
                         long long price_cents, int stock);

    // --- Users ---
    std::optional<User> create_user(const std::string& username, const std::string& password_hash,
                                     const std::string& role);
    std::optional<User> find_user_by_username(const std::string& username);

    // --- Orders (หัวใจของโปรเจกต์: transaction ที่ปลอดภัยต่อ concurrency) ---
    Order place_order(int user_id, const std::vector<OrderLineRequest>& lines);
    std::vector<Order> list_orders_by_user(int user_id);
    std::optional<Order> find_order_by_id(int order_id, int user_id);

private:
    ConnectionPool pool_;

    Order load_order_with_items(pqxx::work& txn, int order_id);
};
```

### `src/db.cpp`: ส่วน Categories และ Products

```cpp
#include "db.hpp"
#include <algorithm>
#include <sstream>

Db::Db(const std::string& conn_string, std::size_t pool_size) : pool_(conn_string, pool_size) {}

// ---------------------------------------------------------------------------
// Categories
// ---------------------------------------------------------------------------
std::vector<Category> Db::list_categories() {
    auto conn = pool_.acquire();
    pqxx::work txn(*conn);
    pqxx::result rows = txn.exec("SELECT id, name, slug FROM categories ORDER BY id;");
    txn.commit();

    std::vector<Category> result;
    for (const auto& row : rows) {
        Category c;
        c.id = row["id"].as<int>();
        c.name = row["name"].as<std::string>();
        c.slug = row["slug"].as<std::string>();
        result.push_back(c);
    }
    return result;
}

// ---------------------------------------------------------------------------
// Products
// ---------------------------------------------------------------------------
static Product row_to_product(const pqxx::row& row) {
    Product p;
    p.id = row["id"].as<int>();
    p.category_id = row["category_id"].as<int>();
    p.category_name = row["category_name"].as<std::string>();
    p.sku = row["sku"].as<std::string>();
    p.name = row["name"].as<std::string>();
    p.description = row["description"].as<std::string>();
    p.price_cents = row["price_cents"].as<long long>();
    p.stock = row["stock"].as<int>();
    return p;
}

ProductPage Db::list_products(const ProductQuery& q) {
    auto conn = pool_.acquire();
    pqxx::work txn(*conn);

    // เราต่อ WHERE clause แบบมีเงื่อนไข แต่ "ค่า" ทุกตัวยังคงส่งผ่าน exec_params เสมอ
    // (ต่อแค่โครงสร้าง SQL ที่เป็นค่าคงที่ ไม่เคยต่อค่าที่มาจากผู้ใช้ตรงๆ)
    std::string where = "WHERE 1=1";
    int param_idx = 1;
    std::vector<std::string> params;

    if (q.category_id.has_value()) {
        where += " AND p.category_id = $" + std::to_string(param_idx++);
        params.push_back(std::to_string(*q.category_id));
    }
    if (q.search.has_value() && !q.search->empty()) {
        where += " AND lower(p.name) LIKE lower($" + std::to_string(param_idx++) + ")";
        params.push_back("%" + *q.search + "%");
    }

    // นับจำนวนทั้งหมดก่อน (สำหรับ pagination metadata)
    std::string count_sql = "SELECT COUNT(*) AS cnt FROM products p " + where + ";";
    pqxx::result count_result;
    {
        pqxx::params pp;
        for (auto& v : params) pp.append(v);
        count_result = txn.exec_params(count_sql, pp);
    }
    long long total_count = count_result[0]["cnt"].as<long long>();

    int page = std::max(1, q.page);
    int limit = std::max(1, std::min(100, q.limit)); // จำกัดสูงสุด 100 ต่อหน้า ป้องกัน DoS
    int offset = (page - 1) * limit;

    // เขียนแยกทีละบรรทัดโดยตั้งใจ (ไม่ใช้ param_idx++ สองครั้งในนิพจน์เดียว) เพราะลำดับการ
    // ประเมินผลของ operator+ ต่อเนื่องกันไม่ได้ถูกกำหนดไว้ตายตัวใน C++ (unsequenced) — ถ้าเขียน
    // ...+ std::to_string(param_idx++) + ... + std::to_string(param_idx++) ... ในนิพจน์เดียว
    // คอมไพเลอร์มีสิทธิ์ประเมินฝั่งขวาก่อนฝั่งซ้ายก็ได้ ทำให้ได้เลข placeholder สลับกันโดยไม่รู้ตัว
    // (ดู Common Pitfalls ข้อ "Sequence Point" ท้ายบท)
    int limit_placeholder = param_idx++;
    int offset_placeholder = param_idx++;
    std::string list_sql =
        "SELECT p.id, p.category_id, c.name AS category_name, p.sku, p.name, "
        "       p.description, p.price_cents, p.stock "
        "FROM products p "
        "JOIN categories c ON c.id = p.category_id " +
        where +
        " ORDER BY p.id "
        "LIMIT $" + std::to_string(limit_placeholder) +
        " OFFSET $" + std::to_string(offset_placeholder) + ";";
    params.push_back(std::to_string(limit));
    params.push_back(std::to_string(offset));

    pqxx::result rows;
    {
        pqxx::params pp;
        for (auto& v : params) pp.append(v);
        rows = txn.exec_params(list_sql, pp);
    }
    txn.commit();

    ProductPage page_result;
    for (const auto& row : rows) page_result.items.push_back(row_to_product(row));
    page_result.total_count = total_count;
    page_result.page = page;
    page_result.limit = limit;
    page_result.total_pages = (total_count + limit - 1) / limit;
    return page_result;
}

std::optional<Product> Db::find_product_by_id(int id) {
    auto conn = pool_.acquire();
    pqxx::work txn(*conn);
    pqxx::result rows = txn.exec_params(
        "SELECT p.id, p.category_id, c.name AS category_name, p.sku, p.name, "
        "       p.description, p.price_cents, p.stock "
        "FROM products p JOIN categories c ON c.id = p.category_id "
        "WHERE p.id = $1;",
        id
    );
    txn.commit();
    if (rows.empty()) return std::nullopt;
    return row_to_product(rows[0]);
}

Product Db::create_product(int category_id, const std::string& sku, const std::string& name,
                            const std::string& description, long long price_cents, int stock) {
    auto conn = pool_.acquire();
    pqxx::work txn(*conn);
    pqxx::result r = txn.exec_params(
        "INSERT INTO products (category_id, sku, name, description, price_cents, stock) "
        "VALUES ($1, $2, $3, $4, $5, $6) RETURNING id;",
        category_id, sku, name, description, price_cents, stock
    );
    txn.commit();
    int new_id = r[0]["id"].as<int>();
    return *find_product_by_id(new_id);
}

bool Db::update_product(int id, const std::string& name, const std::string& description,
                         long long price_cents, int stock) {
    auto conn = pool_.acquire();
    pqxx::work txn(*conn);
    pqxx::result r = txn.exec_params(
        "UPDATE products SET name = $1, description = $2, price_cents = $3, stock = $4 "
        "WHERE id = $5;",
        name, description, price_cents, stock, id
    );
    txn.commit();
    return r.affected_rows() > 0;
}
```

### ทำไม `list_products()` ใช้ `JOIN` แทนที่จะ query แยก: การหลีกเลี่ยงปัญหา N+1

นี่คือจุดสำคัญที่แยกโค้ด production-quality ออกจากโค้ดตัวอย่างทั่วไป ลองจินตนาการถ้าเขียนแบบ
"ง่ายแต่ผิด" ดังนี้:

```cpp
// ❌ ตัวอย่างโค้ดที่มีปัญหา N+1 — อย่าทำแบบนี้
std::vector<Product> list_products_naive() {
    auto products = query("SELECT * FROM products;");        // Query 1 ครั้ง ได้สินค้า N ตัว
    for (auto& p : products) {
        auto cat = query("SELECT name FROM categories WHERE id = " + p.category_id);
        // ⚠️ Query เพิ่มอีก 1 ครั้งต่อสินค้า 1 ตัว!
        p.category_name = cat.name;
    }
    return products;
}
```

ถ้ามีสินค้า 100 ตัว โค้ดข้างต้นจะยิง query ไปที่ฐานข้อมูลทั้งหมด **1 + 100 = 101 ครั้ง** (Query
แรกดึงสินค้าทั้งหมด บวก Query ที่ N สำหรับดึงชื่อหมวดหมู่ของสินค้าแต่ละตัว) — นี่คือปัญหาที่
เรียกว่า **N+1 Query Problem** ซึ่งเป็นสาเหตุอันดับต้นๆ ของ API ที่ช้าลงอย่างรุนแรงเมื่อข้อมูล
มีจำนวนมากขึ้น (Latency ยิ่งเพิ่มตามจำนวนแถวเชิงเส้น ทั้งที่ผู้ใช้แค่อยากได้ "รายการสินค้า 1
หน้า")

`list_products()` ในโปรเจกต์นี้แก้ปัญหานี้ด้วย **`JOIN` เดียว** ที่ดึงทั้งข้อมูลสินค้าและชื่อ
หมวดหมู่มาพร้อมกันในเที่ยวเดียว (`FROM products p JOIN categories c ON c.id = p.category_id`)
ไม่ว่าจะมีสินค้ากี่ตัว ก็ใช้แค่ **1 query เดียวเสมอ** — นี่คือความแตกต่างเชิงคุณภาพระหว่าง
"เขียนโค้ดให้รันได้" กับ "เขียนโค้ดที่ scale ได้จริงเมื่อข้อมูลโต"

| แนวทาง | จำนวน Query เมื่อมีสินค้า 100 ตัว | จำนวน Query เมื่อมีสินค้า 100,000 ตัว |
|---|---|---|
| แยก query ต่อแถว (N+1) | 101 ครั้ง | 100,001 ครั้ง |
| **JOIN เดียว (ที่ใช้ในบทนี้)** | **1 ครั้ง** | **1 ครั้ง** |

### `src/routes_catalog.cpp`: Endpoint ทั้งหมดของ Category และ Product

```cpp
#include "routes_catalog.hpp"
#include "response.hpp"
#include <nlohmann/json.hpp>
#include <pqxx/pqxx>
#include <cstdlib>
#include <algorithm>

using json = nlohmann::json;

// validate payload สำหรับ POST/PUT /products (ใช้ร่วมกันทั้งสอง endpoint)
static std::string validate_product_payload(const json& body, bool require_all_fields) {
    if (require_all_fields && !body.contains("category_id")) return "ต้องมีฟิลด์ \"category_id\"";
    if (require_all_fields && !body.contains("sku")) return "ต้องมีฟิลด์ \"sku\"";
    if (require_all_fields && !body.contains("name")) return "ต้องมีฟิลด์ \"name\"";
    if (require_all_fields && !body.contains("price_cents")) return "ต้องมีฟิลด์ \"price_cents\"";
    if (require_all_fields && !body.contains("stock")) return "ต้องมีฟิลด์ \"stock\"";

    if (body.contains("category_id") && !body["category_id"].is_number_integer())
        return "ฟิลด์ \"category_id\" ต้องเป็นจำนวนเต็ม";
    if (body.contains("sku") && (!body["sku"].is_string() || body["sku"].get<std::string>().empty()))
        return "ฟิลด์ \"sku\" ต้องเป็น string ที่ไม่ว่าง";
    if (body.contains("name") && (!body["name"].is_string() || body["name"].get<std::string>().empty()))
        return "ฟิลด์ \"name\" ต้องเป็น string ที่ไม่ว่าง";
    if (body.contains("description") && !body["description"].is_string())
        return "ฟิลด์ \"description\" ต้องเป็น string";
    if (body.contains("price_cents") &&
        (!body["price_cents"].is_number_integer() || body["price_cents"].get<long long>() < 0))
        return "ฟิลด์ \"price_cents\" ต้องเป็นจำนวนเต็มไม่ติดลบ (หน่วยสตางค์)";
    if (body.contains("stock") &&
        (!body["stock"].is_number_integer() || body["stock"].get<int>() < 0))
        return "ฟิลด์ \"stock\" ต้องเป็นจำนวนเต็มไม่ติดลบ";
    return "";
}

void register_catalog_routes(crow::App<AuthMiddleware>& app, Db& db) {

    // GET /categories — สาธารณะ ไม่ต้อง login
    CROW_ROUTE(app, "/categories")
    .methods(crow::HTTPMethod::GET)
    ([&db]() {
        try {
            auto cats = db.list_categories();
            json arr = json::array();
            for (const auto& c : cats) arr.push_back(c.to_json());
            return api::ok(arr);
        } catch (const std::exception& e) {
            return api::error(500, "INTERNAL_ERROR", e.what());
        }
    });

    // GET /products — รายการสินค้า (สาธารณะ) พร้อม filter/search/pagination
    // query params: category_id, search, page, limit
    CROW_ROUTE(app, "/products")
    .methods(crow::HTTPMethod::GET)
    ([&db](const crow::request& req) {
        try {
            ProductQuery q;
            if (const char* cat = req.url_params.get("category_id")) {
                q.category_id = std::atoi(cat);
            }
            if (const char* s = req.url_params.get("search")) {
                q.search = std::string(s);
            }
            if (const char* p = req.url_params.get("page")) {
                q.page = std::max(1, std::atoi(p));
            }
            if (const char* l = req.url_params.get("limit")) {
                q.limit = std::atoi(l);
            }

            ProductPage result = db.list_products(q);
            json arr = json::array();
            for (const auto& p : result.items) arr.push_back(p.to_json());

            json data{
                {"items", arr},
                {"page", result.page},
                {"limit", result.limit},
                {"total_count", result.total_count},
                {"total_pages", result.total_pages}
            };
            return api::ok(data);
        } catch (const std::exception& e) {
            return api::error(500, "INTERNAL_ERROR", e.what());
        }
    });

    // GET /products/<id> — รายละเอียดสินค้าตัวเดียว (สาธารณะ)
    CROW_ROUTE(app, "/products/<int>")
    .methods(crow::HTTPMethod::GET)
    ([&db](int id) {
        try {
            auto p = db.find_product_by_id(id);
            if (!p) {
                return api::error(404, "PRODUCT_NOT_FOUND", "ไม่พบสินค้า id = " + std::to_string(id));
            }
            return api::ok(p->to_json());
        } catch (const std::exception& e) {
            return api::error(500, "INTERNAL_ERROR", e.what());
        }
    });

    // POST /products — สร้างสินค้าใหม่ (admin เท่านั้น)
    CROW_ROUTE(app, "/products")
    .methods(crow::HTTPMethod::POST)
    .CROW_MIDDLEWARES(app, AuthMiddleware)
    ([&app, &db](const crow::request& req) {
        auto& ctx = app.get_context<AuthMiddleware>(req);
        if (ctx.role != "admin") {
            return api::error(403, "FORBIDDEN", "เฉพาะผู้ดูแลระบบ (admin) เท่านั้นที่สร้างสินค้าได้");
        }

        json body;
        try {
            body = json::parse(req.body);
        } catch (const json::parse_error&) {
            return api::error(400, "INVALID_JSON", "Body ไม่ใช่ JSON ที่ถูกต้อง");
        }
        if (!body.is_object()) {
            return api::error(400, "INVALID_JSON", "Body ต้องเป็น JSON object");
        }
        std::string err = validate_product_payload(body, /*require_all_fields=*/true);
        if (!err.empty()) {
            return api::error(400, "VALIDATION_ERROR", err);
        }

        try {
            Product created = db.create_product(
                body["category_id"].get<int>(),
                body["sku"].get<std::string>(),
                body["name"].get<std::string>(),
                body.value("description", ""),
                body["price_cents"].get<long long>(),
                body["stock"].get<int>()
            );
            return api::ok(created.to_json(), 201);
        } catch (const pqxx::foreign_key_violation&) {
            return api::error(400, "INVALID_CATEGORY", "ไม่พบ category_id ที่ระบุ");
        } catch (const pqxx::unique_violation&) {
            return api::error(409, "SKU_TAKEN", "sku นี้ถูกใช้ไปแล้ว");
        } catch (const std::exception& e) {
            return api::error(500, "INTERNAL_ERROR", e.what());
        }
    });

    // PUT /products/<id> — แก้ไขสินค้า (admin เท่านั้น) — partial update
    CROW_ROUTE(app, "/products/<int>")
    .methods(crow::HTTPMethod::PUT)
    .CROW_MIDDLEWARES(app, AuthMiddleware)
    ([&app, &db](const crow::request& req, int id) {
        auto& ctx = app.get_context<AuthMiddleware>(req);
        if (ctx.role != "admin") {
            return api::error(403, "FORBIDDEN", "เฉพาะผู้ดูแลระบบ (admin) เท่านั้นที่แก้ไขสินค้าได้");
        }

        auto existing = db.find_product_by_id(id);
        if (!existing) {
            return api::error(404, "PRODUCT_NOT_FOUND", "ไม่พบสินค้า id = " + std::to_string(id));
        }

        json body;
        try {
            body = json::parse(req.body);
        } catch (const json::parse_error&) {
            return api::error(400, "INVALID_JSON", "Body ไม่ใช่ JSON ที่ถูกต้อง");
        }
        if (!body.is_object()) {
            return api::error(400, "INVALID_JSON", "Body ต้องเป็น JSON object");
        }
        std::string err = validate_product_payload(body, /*require_all_fields=*/false);
        if (!err.empty()) {
            return api::error(400, "VALIDATION_ERROR", err);
        }

        try {
            std::string name = body.value("name", existing->name);
            std::string description = body.value("description", existing->description);
            long long price_cents = body.value("price_cents", existing->price_cents);
            int stock = body.value("stock", existing->stock);
            db.update_product(id, name, description, price_cents, stock);
            auto updated = db.find_product_by_id(id);
            return api::ok(updated->to_json());
        } catch (const std::exception& e) {
            return api::error(500, "INTERNAL_ERROR", e.what());
        }
    });
}
```

จุดที่ควรสังเกตเป็นพิเศษ: การ catch `pqxx::foreign_key_violation` และ `pqxx::unique_violation`
แยกจาก `std::exception` ทั่วไป — libpqxx แปล PostgreSQL error code (เช่น `23503` สำหรับ foreign
key violation, `23505` สำหรับ unique violation) เป็น exception class เฉพาะให้อัตโนมัติ ทำให้
เราแปล error ของฐานข้อมูลเป็น HTTP status code ที่ถูกต้องได้โดยไม่ต้อง parse error message
เป็น string เอง (ซึ่งเปราะบางกว่ามากถ้า PostgreSQL เปลี่ยนข้อความ error ในเวอร์ชันใหม่)

---

## 121.6 Orders: Transaction ที่ปลอดภัยต่อ Concurrency (หัวใจของโปรเจกต์) (Step 966)

นี่คือหัวข้อที่สำคัญที่สุดของทั้ง Capstone — ทุกอย่างที่เรียนมาตั้งแต่ต้น Part นี้ (schema
design, layered architecture, RBAC) เป็นแค่ "โครงสร้างรองรับ" ให้กับปัญหาแกนกลางข้อเดียว:
**ทำอย่างไรให้การสั่งซื้อสินค้าถูกต้อง 100% แม้มีลูกค้าหลายคนสั่งซื้อสินค้าตัวเดียวกันพร้อมกัน**

### ทำความเข้าใจ Race Condition ด้วยภาพ

ลองจินตนาการสินค้าตัวหนึ่งมี `stock = 1` (เหลือชิ้นสุดท้าย) และมีลูกค้า 2 คนกด "สั่งซื้อ" ในเวลา
ใกล้เคียงกันมากจนสอง request ถูกประมวลผลพร้อมกัน ถ้าเขียนโค้ดแบบ "เช็คก่อนแล้วค่อยลด"
(Check-Then-Act) โดยไม่ป้องกันอะไรเลย จะเกิดสถานการณ์นี้:

```
เวลา     Thread A (ลูกค้า 1)              Thread B (ลูกค้า 2)
──────────────────────────────────────────────────────────────
t1       SELECT stock FROM products
         WHERE id=5;  -> ได้ stock=1
t2                                        SELECT stock FROM products
                                          WHERE id=5;  -> ได้ stock=1
t3       ตรวจ: 1 >= 1 (พอ) ✓
t4                                        ตรวจ: 1 >= 1 (พอ) ✓
t5       UPDATE stock = stock - 1
         (stock กลายเป็น 0)
t6                                        UPDATE stock = stock - 1
                                          (stock กลายเป็น -1) ⚠️ ติดลบ!
```

ทั้งสอง Thread อ่านค่า `stock = 1` **ก่อน** ที่ฝั่งใดฝั่งหนึ่งจะเขียนค่าลดลง ทำให้ทั้งคู่คิดว่า
"มีของพอ" และปล่อยให้คำสั่งซื้อผ่านทั้งสองรายการ ผลคือร้านค้าขายสินค้าที่ไม่มีอยู่จริงออกไป
(Overselling) — นี่คือ **Race Condition** แบบคลาสสิกที่เรียกว่า **Check-Then-Act** race
(ทบทวนแนวคิดพื้นฐานของ Race Condition จาก Part 73 เรื่อง Concurrency ใน C++)

### ทางแก้: `SELECT ... FOR UPDATE`

PostgreSQL มีกลไก **Row-Level Locking** ที่แก้ปัญหานี้ได้โดยตรงผ่านคำสั่ง
`SELECT ... FOR UPDATE` — เมื่อ transaction หนึ่งอ่านแถวด้วยคำสั่งนี้ PostgreSQL จะ **ล็อกแถว
นั้นไว้ทันที** ไม่ให้ transaction อื่นอ่านแบบ `FOR UPDATE` แถวเดียวกันได้จนกว่า transaction แรก
จะ `COMMIT` หรือ `ROLLBACK` — transaction ที่สองจะ**ต้องรอ** (block) ไม่ใช่ผ่านไปพร้อมกันแบบ
ในภาพด้านบน

```
เวลา     Thread A (ลูกค้า 1)                       Thread B (ลูกค้า 2)
────────────────────────────────────────────────────────────────────────
t1       BEGIN;
t2       SELECT stock FROM products
         WHERE id=5 FOR UPDATE;  -> stock=1
         (แถวนี้ถูกล็อกแล้ว)
t3                                                 BEGIN;
t4                                                 SELECT stock FROM products
                                                    WHERE id=5 FOR UPDATE;
                                                    -> ⏸ ต้องรอ! (แถวถูกล็อกอยู่)
t5       ตรวจ: 1 >= 1 (พอ) ✓
t6       UPDATE stock = stock - 1;  (stock=0)
t7       COMMIT;  (ปลดล็อก)
t8                                                 -> ตื่นจากการรอ ได้ stock=0 (ค่าล่าสุดหลัง commit)
t9                                                 ตรวจ: 0 >= 1 (ไม่พอ!) ✗
t10                                                ROLLBACK; คืน 409 INSUFFICIENT_STOCK
```

Thread B ไม่ได้อ่านค่า stock "เก่า" (`1`) แต่ต้องรอจน Thread A ทำงานเสร็จและ commit ก่อน แล้ว
จึงอ่านค่า **ล่าสุดที่แท้จริง** (`0`) ทำให้ตรวจพบว่าสินค้าหมดได้อย่างถูกต้อง — นี่คือหลักการ
เดียวกับที่แบบฝึกหัดข้อ 3 ของ Part 106 ใช้แก้ปัญหาการโอนเงินระหว่างบัญชี (`transfer_money`)
เพียงแต่ตอนนี้เราขยายให้ทำงานกับสินค้าหลายรายการในคำสั่งซื้อเดียว

### ทำไมต้องเรียง `product_id` ก่อนล็อกเสมอ: ป้องกัน Deadlock

เมื่อคำสั่งซื้อหนึ่งใบมีสินค้าหลายรายการ (เช่น สั่งทั้งสินค้า id=6 และ id=7 พร้อมกัน) จะต้องล็อก
หลายแถวในคำสั่งซื้อเดียว ถ้าลูกค้าสองคนสั่งสินค้าชุดเดียวกันแต่ **ส่งลำดับ item ในคนละแบบ**
กัน (คนแรกส่ง `[6, 7]` คนที่สองส่ง `[7, 6]`) แล้วเราล็อกตามลำดับที่ client ส่งมาตรงๆ อาจเกิด
**Deadlock**:

```
เวลา     Order A (items: [6, 7])            Order B (items: [7, 6])
──────────────────────────────────────────────────────────────────
t1       ล็อกสินค้า id=6 สำเร็จ
t2                                          ล็อกสินค้า id=7 สำเร็จ
t3       พยายามล็อกสินค้า id=7
         -> ต้องรอ Order B ปล่อยล็อก
t4                                          พยายามล็อกสินค้า id=6
                                            -> ต้องรอ Order A ปล่อยล็อก
         ⚠️ ทั้งสองฝั่งรอกันไปมาไม่มีวันจบ (Deadlock)
```

PostgreSQL มีกลไกตรวจจับ deadlock อัตโนมัติและจะยกเลิก transaction ใดตัวหนึ่งด้วย error
`deadlock detected` แต่นั่นก็แปลว่า order ของลูกค้าคนหนึ่งล้มเหลวไปโดยไม่จำเป็น ทางแก้ที่ถูกต้อง
คือ **บังคับให้ทุก transaction ล็อกทรัพยากรตามลำดับเดียวกันเสมอ** ไม่ว่า client จะส่งลำดับ
มาแบบไหนก็ตาม — ในโค้ดของเราคือการ `std::sort` รายการสินค้าตาม `product_id` ก่อนวนล็อกเสมอ
ทำให้ทั้ง Order A และ Order B ล็อกตามลำดับ `6` ก่อน `7` เสมอ ไม่มีทางเกิดการรอกันเป็นวงกลมได้อีก
ต่อไป

### `src/db.cpp`: `place_order()` เต็มรูปแบบ

```cpp
// ---------------------------------------------------------------------------
// Orders — จุดที่สำคัญที่สุดของทั้งโปรเจกต์: transaction ที่ปลอดภัยต่อ concurrency
// ---------------------------------------------------------------------------
Order Db::place_order(int user_id, const std::vector<OrderLineRequest>& lines) {
    auto conn = pool_.acquire();
    pqxx::work txn(*conn);

    // ---- ขั้นตอนที่ 1: เรียง product_id จากน้อยไปมากก่อนล็อกเสมอ ----
    // เหตุผล: ถ้า order A ล็อก (1,2) และ order B ล็อก (2,1) พร้อมกันโดยไม่เรียงลำดับ
    // อาจเกิด deadlock ได้ (แต่ละฝั่งถือ lock ของอีกฝั่งต้องการ รอกันไปมาไม่จบ)
    // การบังคับล็อกตามลำดับ id เดียวกันเสมอทำให้ไม่มีทางเกิด deadlock แบบวนรอบได้เลย
    std::vector<OrderLineRequest> sorted_lines = lines;
    std::sort(sorted_lines.begin(), sorted_lines.end(),
              [](const OrderLineRequest& a, const OrderLineRequest& b) {
                  return a.product_id < b.product_id;
              });

    long long total_cents = 0;
    std::vector<OrderItemLine> item_lines;

    // ---- ขั้นตอนที่ 2: ล็อกแถวสินค้าทุกตัวด้วย SELECT ... FOR UPDATE ----
    // FOR UPDATE บอก PostgreSQL ว่า "ห้าม transaction อื่นแตะแถวนี้จนกว่าเราจะ commit/rollback"
    // นี่คือกลไกที่ป้องกัน race condition ตอนเช็ค-แล้ว-ลด stock (check-then-act)
    for (const auto& line : sorted_lines) {
        pqxx::result rows = txn.exec_params(
            "SELECT id, name, price_cents, stock FROM products WHERE id = $1 FOR UPDATE;",
            line.product_id
        );
        if (rows.empty()) {
            throw ProductNotFoundError(line.product_id);
        }
        int current_stock = rows[0]["stock"].as<int>();
        std::string pname = rows[0]["name"].as<std::string>();
        long long price = rows[0]["price_cents"].as<long long>();

        if (current_stock < line.quantity) {
            // txn จะ rollback อัตโนมัติเมื่อ exception หลุดออกจากฟังก์ชันนี้ (destructor ของ pqxx::work)
            throw InsufficientStockError(line.product_id, pname, line.quantity, current_stock);
        }

        // ---- ขั้นตอนที่ 3: ลด stock ทันทีในทรานแซกชันเดียวกัน (ยังไม่ commit) ----
        txn.exec_params(
            "UPDATE products SET stock = stock - $1 WHERE id = $2;",
            line.quantity, line.product_id
        );

        OrderItemLine oil;
        oil.product_id = line.product_id;
        oil.product_name = pname;
        oil.quantity = line.quantity;
        oil.unit_price_cents = price;
        item_lines.push_back(oil);
        total_cents += price * line.quantity;
    }

    // ---- ขั้นตอนที่ 4: สร้าง order + order_items ----
    pqxx::result order_row = txn.exec_params(
        "INSERT INTO orders (user_id, status, total_cents) VALUES ($1, 'placed', $2) RETURNING id, created_at;",
        user_id, total_cents
    );
    int order_id = order_row[0]["id"].as<int>();

    for (const auto& oil : item_lines) {
        txn.exec_params(
            "INSERT INTO order_items (order_id, product_id, quantity, unit_price_cents) "
            "VALUES ($1, $2, $3, $4);",
            order_id, oil.product_id, oil.quantity, oil.unit_price_cents
        );
    }

    // ---- ขั้นตอนที่ 5: commit ทั้งหมดในครั้งเดียว ----
    // ถ้า process ล่มก่อนบรรทัดนี้ (ไฟดับ, exception) transaction ทั้งก้อนจะถูก rollback
    // stock ที่ลดไปตอน SELECT...FOR UPDATE + UPDATE ข้างต้นจะกลับคืนสภาพเดิมโดยอัตโนมัติ
    txn.commit();

    Order order;
    order.id = order_id;
    order.user_id = user_id;
    order.status = "placed";
    order.total_cents = total_cents;
    order.created_at = order_row[0]["created_at"].as<std::string>();
    order.items = item_lines;
    return order;
}
```

ทุกขั้นตอน (ล็อก → ตรวจสอบ → ลด stock → สร้าง order → สร้าง order_items → commit) อยู่ใน
**transaction เดียว** (`pqxx::work txn`) — ถ้าขั้นตอนใดขั้นตอนหนึ่งล้มเหลว (สินค้าไม่พอ, สินค้า
ไม่มีอยู่จริง, connection หลุด) exception จะทำให้ `txn` ถูกทำลายโดยไม่ได้ `commit()` และ
PostgreSQL จะ **rollback ทุกอย่างที่ทำไปในทรานแซกชันนี้กลับสู่สภาพเดิมโดยอัตโนมัติ** (หลักการ
เดียวกับที่อธิบายไว้ใน Part 106.6 เรื่องการลืม commit) — นี่คือเหตุผลที่คำสั่งซื้อที่ล้มเหลวไม่
ทำให้ stock ของสินค้าตัวอื่นในออเดอร์เดียวกันถูกลดไปครึ่งๆ กลางๆ

`load_order_with_items()` (ใช้ร่วมกันระหว่าง `list_orders_by_user()` และ `find_order_by_id()`)
และ `list_orders_by_user()`/`find_order_by_id()` เต็มรูปแบบ:

```cpp
Order Db::load_order_with_items(pqxx::work& txn, int order_id) {
    pqxx::result order_rows = txn.exec_params(
        "SELECT id, user_id, status, total_cents, created_at FROM orders WHERE id = $1;",
        order_id
    );
    Order order;
    order.id = order_rows[0]["id"].as<int>();
    order.user_id = order_rows[0]["user_id"].as<int>();
    order.status = order_rows[0]["status"].as<std::string>();
    order.total_cents = order_rows[0]["total_cents"].as<long long>();
    order.created_at = order_rows[0]["created_at"].as<std::string>();

    pqxx::result item_rows = txn.exec_params(
        "SELECT oi.product_id, p.name AS product_name, oi.quantity, oi.unit_price_cents "
        "FROM order_items oi JOIN products p ON p.id = oi.product_id "
        "WHERE oi.order_id = $1 ORDER BY oi.id;",
        order_id
    );
    for (const auto& row : item_rows) {
        OrderItemLine oil;
        oil.product_id = row["product_id"].as<int>();
        oil.product_name = row["product_name"].as<std::string>();
        oil.quantity = row["quantity"].as<int>();
        oil.unit_price_cents = row["unit_price_cents"].as<long long>();
        order.items.push_back(oil);
    }
    return order;
}

std::vector<Order> Db::list_orders_by_user(int user_id) {
    auto conn = pool_.acquire();
    pqxx::work txn(*conn);
    pqxx::result rows = txn.exec_params(
        "SELECT id FROM orders WHERE user_id = $1 ORDER BY id DESC;", user_id
    );
    std::vector<Order> orders;
    for (const auto& row : rows) {
        orders.push_back(load_order_with_items(txn, row["id"].as<int>()));
    }
    txn.commit();
    return orders;
}

std::optional<Order> Db::find_order_by_id(int order_id, int user_id) {
    auto conn = pool_.acquire();
    pqxx::work txn(*conn);
    pqxx::result check = txn.exec_params(
        "SELECT id FROM orders WHERE id = $1 AND user_id = $2;", order_id, user_id
    );
    if (check.empty()) {
        txn.commit();
        return std::nullopt;
    }
    Order order = load_order_with_items(txn, order_id);
    txn.commit();
    return order;
}
```

สังเกตว่า `find_order_by_id()` เช็ค `WHERE id = $1 AND user_id = $2` **ในคำสั่งเดียว** แทนที่
จะ query แค่ `id` แล้วเอามาเช็ค `user_id` ทีหลังในโค้ด C++ — นี่คือแพทเทิร์นความปลอดภัยที่สำคัญ
มาก เพราะทำให้ **ฐานข้อมูลเป็นผู้บังคับกฎ "ownership" ให้เราโดยตรง** ไม่มีทางที่บั๊กใน C++ (เช่น
ลืมเช็ค `if` เงื่อนไข) จะทำให้ผู้ใช้คนหนึ่งเห็นข้อมูล order ของอีกคนหลุดออกไปได้เลย

`src/routes_orders.cpp` — Route layer ที่แปล exception จาก DAL เป็น HTTP status code ที่ถูกต้อง:

```cpp
#include "routes_orders.hpp"
#include "response.hpp"
#include <nlohmann/json.hpp>

using json = nlohmann::json;

// validate body ของ POST /orders: {"items":[{"product_id":1,"quantity":2}, ...]}
static std::string validate_order_payload(const json& body, std::vector<OrderLineRequest>& out) {
    if (!body.contains("items") || !body["items"].is_array() || body["items"].empty()) {
        return "ต้องมีฟิลด์ \"items\" เป็น array ที่ไม่ว่าง";
    }
    for (const auto& item : body["items"]) {
        if (!item.is_object() || !item.contains("product_id") || !item.contains("quantity")) {
            return "แต่ละรายการใน \"items\" ต้องมี \"product_id\" และ \"quantity\"";
        }
        if (!item["product_id"].is_number_integer() || item["product_id"].get<int>() <= 0) {
            return "\"product_id\" ต้องเป็นจำนวนเต็มบวก";
        }
        if (!item["quantity"].is_number_integer() || item["quantity"].get<int>() <= 0) {
            return "\"quantity\" ต้องเป็นจำนวนเต็มบวก";
        }
        out.push_back(OrderLineRequest{item["product_id"].get<int>(), item["quantity"].get<int>()});
    }
    return "";
}

void register_order_routes(crow::App<AuthMiddleware>& app, Db& db) {

    // POST /orders — สั่งซื้อ (ต้อง login) หัวใจของระบบ: ตรวจ+ลด stock แบบ atomic ในทรานแซกชันเดียว
    CROW_ROUTE(app, "/orders")
    .methods(crow::HTTPMethod::POST)
    .CROW_MIDDLEWARES(app, AuthMiddleware)
    ([&app, &db](const crow::request& req) {
        auto& ctx = app.get_context<AuthMiddleware>(req);

        json body;
        try {
            body = json::parse(req.body);
        } catch (const json::parse_error&) {
            return api::error(400, "INVALID_JSON", "Body ไม่ใช่ JSON ที่ถูกต้อง");
        }
        if (!body.is_object()) {
            return api::error(400, "INVALID_JSON", "Body ต้องเป็น JSON object");
        }

        std::vector<OrderLineRequest> lines;
        std::string err = validate_order_payload(body, lines);
        if (!err.empty()) {
            return api::error(400, "VALIDATION_ERROR", err);
        }

        try {
            Order order = db.place_order(ctx.user_id, lines);
            return api::ok(order.to_json(), 201);
        } catch (const InsufficientStockError& e) {
            // 409 Conflict: คำขอขัดแย้งกับสถานะปัจจุบันของทรัพยากร (สินค้าในสต็อกไม่พอ)
            return api::error(409, "INSUFFICIENT_STOCK", e.what());
        } catch (const ProductNotFoundError& e) {
            return api::error(400, "PRODUCT_NOT_FOUND", e.what());
        } catch (const std::exception& e) {
            return api::error(500, "INTERNAL_ERROR", e.what());
        }
    });

    // GET /orders — ประวัติการสั่งซื้อของผู้ใช้ที่ login อยู่เท่านั้น (ต้อง login)
    CROW_ROUTE(app, "/orders")
    .methods(crow::HTTPMethod::GET)
    .CROW_MIDDLEWARES(app, AuthMiddleware)
    ([&app, &db](const crow::request& req) {
        auto& ctx = app.get_context<AuthMiddleware>(req);
        try {
            auto orders = db.list_orders_by_user(ctx.user_id);
            json arr = json::array();
            for (const auto& o : orders) arr.push_back(o.to_json());
            return api::ok(arr);
        } catch (const std::exception& e) {
            return api::error(500, "INTERNAL_ERROR", e.what());
        }
    });

    // GET /orders/<id> — รายละเอียด order เดียว (ต้องเป็นเจ้าของ order เท่านั้น)
    CROW_ROUTE(app, "/orders/<int>")
    .methods(crow::HTTPMethod::GET)
    .CROW_MIDDLEWARES(app, AuthMiddleware)
    ([&app, &db](const crow::request& req, int id) {
        auto& ctx = app.get_context<AuthMiddleware>(req);
        try {
            auto order = db.find_order_by_id(id, ctx.user_id);
            if (!order) {
                // จงใจคืน 404 (ไม่ใช่ 403) ไม่ว่า order จะไม่มีอยู่จริง หรือมีแต่เป็นของคนอื่น
                // เพื่อไม่ให้ผู้ใช้เดา id ของ order คนอื่นแล้วรู้ได้ว่า id นั้น "มีอยู่จริง"
                return api::error(404, "ORDER_NOT_FOUND", "ไม่พบ order id = " + std::to_string(id));
            }
            return api::ok(order->to_json());
        } catch (const std::exception& e) {
            return api::error(500, "INTERNAL_ERROR", e.what());
        }
    });
}
```

---

## 121.7 บั๊ก Concurrency จริงที่เจอระหว่างพัฒนา และการแก้ด้วย Connection Pool (Step 967)

หัวข้อนี้คือส่วนที่ตรงไปตรงมาที่สุดของทั้งบทเรียน — มันคือเรื่องราวของบั๊กจริงที่เกิดขึ้นจริง
ระหว่างพัฒนาโปรเจกต์นี้ ไม่ใช่ตัวอย่างที่แต่งขึ้นมาสอน ผู้เขียนบทเรียนนี้เขียนโค้ด `Db` เวอร์ชัน
แรกให้ถือ `pqxx::connection conn_` เป็นสมาชิกตัวเดียว (แบบเดียวกับที่ Part 108 ทำกับ
`sqlite3* conn_`) แล้วรันการทดสอบ concurrency ตามที่วางแผนไว้ — ผลลัพธ์คือระบบพังจริง

### ขั้นที่ 1: เขียนโค้ดเวอร์ชันแรก (มีบั๊กซ่อนอยู่)

โค้ดเวอร์ชันแรกของ `Db` เหมือนกับที่แสดงในหัวข้อ 121.5–121.6 ทุกประการ **ยกเว้น** ส่วนเดียวคือ
สมาชิกส่วนตัว:

```cpp
// ❌ เวอร์ชันแรก (มีบั๊ก) — เหมือน Part 108 ทุกประการ แต่ต่างฐานข้อมูล
class Db {
    // ...
private:
    pqxx::connection conn_;   // connection ตัวเดียว แชร์กันทุก request
};

Db::Db(const std::string& conn_string) : conn_(conn_string) {}

// ทุกเมธอดเรียก pqxx::work txn(conn_); ตรงๆ
```

โค้ดนี้ **compile ผ่านสะอาด ไม่มี warning** และทำงานถูกต้อง 100% เมื่อทดสอบทีละ request ด้วย
`curl` (ทุกอย่างในหัวข้อ 121.1–121.6 ผ่านหมด) — นี่คือจุดที่อันตรายที่สุดของบั๊กประเภทนี้: **มัน
ไม่มีทางถูกจับได้เลยถ้าไม่ทดสอบภายใต้ concurrent load จริง**

### ขั้นที่ 2: ยิง Concurrent Request จริงเพื่อทดสอบ — ระบบพัง

สร้างสินค้า "limited edition" ที่มี `stock = 10` แล้วยิง `POST /orders` พร้อมกัน 30 request
(คนละ 1 ชิ้น) ด้วย `xargs -P`:

```bash
#!/bin/bash
# concurrency_test.sh
source /tmp/tokens.env
BASE="http://127.0.0.1:18480"
PRODUCT_ID=5
N=30

order_once() {
    curl -s -o /dev/null -w "%{http_code}\n" -X POST "$BASE/orders" \
        -H "Authorization: Bearer $CUST_TOKEN" \
        -H "Content-Type: application/json" \
        -d "{\"items\":[{\"product_id\":$PRODUCT_ID,\"quantity\":1}]}"
}
export -f order_once
export BASE CUST_TOKEN PRODUCT_ID

seq 1 "$N" | xargs -P "$N" -I{} bash -c order_once > /tmp/concurrency_results.txt
sort /tmp/concurrency_results.txt | uniq -c
```

ผลลัพธ์จริงจากการรันครั้งแรก (ก่อนแก้บั๊ก):

```
$ ./concurrency_test.sh
ยิง 30 คำขอ POST /orders พร้อมกัน (แข่งกันซื้อสินค้า id=5 ที่มี stock เริ่มต้น = 10)...
--- สรุปผลลัพธ์ HTTP status code จากทั้ง 30 คำขอ ---
     10 201
      4 409
     16 500
สำเร็จ (201): 10 ครั้ง
ถูกปฏิเสธเพราะสต็อกไม่พอ (409): 4 ครั้ง
```

**16 จาก 30 request คืน `500 Internal Server Error`** — ผิดปกติทันที (ที่ถูกต้องควรมีแค่ `201`
กับ `409` เท่านั้น ไม่ควรมี `500` เลย) ตรวจสอบ log ของ server เพิ่มเติม:

```
(2026-09-26) [INFO] Request: ... HTTP/1.1 POST /orders
(2026-09-26) [INFO] Response: ... /orders 500 0
(2026-09-26) [INFO] Request: ... HTTP/1.1 POST /orders
(2026-09-26) [INFO] Response: ... /orders 500 0
...
```

และเมื่อลอง `curl` เรียกดูสินค้าตัวเดิมหลังจากนั้น:

```bash
$ curl -s http://127.0.0.1:18480/products/5
{"error":{"code":"INTERNAL_ERROR","message":"Lost connection to the database server."},"success":false}
```

**connection ทั้งตัวพังไปเลย** — แม้แต่ `GET` ธรรมดาที่ไม่เกี่ยวกับการสั่งซื้อก็ใช้ไม่ได้อีกต่อไป
จนกว่าจะ restart server ใหม่ นี่คือสัญญาณชัดเจนว่าปัญหาไม่ใช่แค่ business logic ผิด แต่เป็นปัญหา
ระดับ connection/protocol ที่เสียหายจริง

### ขั้นที่ 3: วินิจฉัยสาเหตุที่แท้จริง

**`pqxx::connection` ไม่ใช่ thread-safe** — นี่คือข้อเท็จจริงที่ต้องเข้าใจให้ชัดเจน และมันคือ
**ความแตกต่างที่สำคัญที่สุดข้อหนึ่งระหว่าง libpqxx กับ SQLite** ที่เรียนไปใน Part 108:

| ประเด็น | `sqlite3*` (Part 108) | `pqxx::connection` (Capstone นี้) |
|---|---|---|
| Thread-safety ของ connection เดียว | **ปลอดภัย** เมื่อคอมไพล์แบบ `SQLITE_THREADSAFE=1` (Serialized mode — ค่า default บน Ubuntu/Debian) | **ไม่ปลอดภัย** — libpqxx ไม่มี mutex ป้องกันภายในให้ |
| สิ่งที่เกิดขึ้นถ้าหลาย thread ใช้ connection เดียวพร้อมกัน | SQLite คิว request ให้อัตโนมัติ (ช้าลงแต่ไม่พัง) | **Protocol state ของ PostgreSQL wire protocol เสียหาย** เพราะสอง thread เขียน/อ่าน socket เดียวกันสลับกันแบบไม่ประสานงาน |
| อาการที่เจอจริงเมื่อพัง | ไม่มี (ปลอดภัยโดย design) | `Lost connection to the database server`, error ที่คาดเดาไม่ได้, บาง request ค้าง |

Crow ทำงานแบบ `.multithreaded()` โดย default (ใช้ `std::thread::hardware_concurrency()` = 4
thread บนเครื่องนี้) เราสร้าง `Db db(conn_string);` เพียงครั้งเดียวใน `main()` แล้วส่ง
**reference เดียวกัน** ให้ทุก route handler ใช้ร่วมกัน (ตามแพทเทิร์นเดียวกับ Part 108 ทุกประการ)
— เมื่อ 4 worker thread ของ Crow ประมวลผล `POST /orders` พร้อมกัน 4 คำขอ ทั้ง 4 thread เรียก
`pqxx::work txn(conn_);` บน **connection object เดียวกัน** พร้อมกัน แต่ละ thread พยายามส่ง
คำสั่ง SQL ผ่าน socket เดียวกันในเวลาเดียวกัน ทำให้ byte stream ของ PostgreSQL wire protocol
ปนกันจนอ่านไม่ออก — เมื่อ protocol เสียหายขนาดนี้ PostgreSQL server ก็ปิด connection ทิ้ง
(สอดคล้องกับข้อความ "Lost connection to the database server" ที่เห็น)

> **บทเรียนสำคัญที่สุดของหัวข้อนี้**: อย่าสมมติว่า pattern ที่ปลอดภัยกับ library หนึ่ง (SQLite ใน
> Part 108) จะปลอดภัยกับ library อื่นโดยอัตโนมัติ (libpqxx ในบทนี้) แม้ทั้งคู่จะเป็น "database
> driver" เหมือนกัน แต่ threading guarantee ของแต่ละ library ต้องตรวจสอบเอกสารแยกกันเสมอ ห้าม
> เดา

### ขั้นที่ 4: แก้ปัญหาด้วย Connection Pool ที่ Thread-Safe จริง

ทางแก้คือให้แต่ละ request "ยืม" connection ของตัวเองจาก pool ระหว่างทำงาน ไม่มี thread สอง
ตัวแตะ connection เดียวกันพร้อมกันเด็ดขาด — สร้างคลาส `ConnectionPool` ต่อยอดจากแนวคิด
`SimpleConnectionPool` ของ Part 106.7 แต่เพิ่ม **mutex + condition_variable** ให้ thread-safe
จริง (Part 106.7 บอกไว้ชัดเจนแล้วว่าตัวอย่างเดิม "ยังไม่ thread-safe เต็มรูปแบบ เพื่อการสาธิต
เท่านั้น" — Capstone นี้คือจุดที่ต้องทำให้สมบูรณ์จริง)

`include/connection_pool.hpp`:

```cpp
#pragma once
#include <pqxx/pqxx>
#include <memory>
#include <mutex>
#include <condition_variable>
#include <vector>
#include <string>

// ---------------------------------------------------------------------------
// ConnectionPool — pool ของ pqxx::connection ที่ thread-safe จริง
//
// เหตุผลที่ต้องมีคลาสนี้: pqxx::connection ตัวเดียว **ไม่ใช่ thread-safe** ต่างจาก
// sqlite3* ใน Part 108 ที่ SQLite คอมไพล์มาแบบ SERIALIZED (มี mutex ในตัวป้องกันให้
// อัตโนมัติ) ถ้าหลาย thread ของ Crow (.multithreaded()) เรียกเมธอดของ connection เดียวกัน
// พร้อมกัน จะเกิด corruption ของ protocol state กับ PostgreSQL ทันที (อาการจริงที่เจอ:
// "Lost connection to the database server" และ error อื่นๆ ที่คาดเดาไม่ได้ — ดู 121.7)
//
// ทางแก้คือให้แต่ละ "งาน" (request handler หนึ่งครั้ง) ยืม connection ของตัวเองจาก pool
// ระหว่างการยืม ไม่มี thread อื่นแตะ connection ตัวนั้นเลย เมื่องานเสร็จ (guard หมด scope)
// connection จะถูกคืนกลับ pool โดยอัตโนมัติผ่าน RAII
// ---------------------------------------------------------------------------
class ConnectionPool {
public:
    ConnectionPool(const std::string& conn_string, std::size_t size) {
        for (std::size_t i = 0; i < size; ++i) {
            available_.push_back(std::make_unique<pqxx::connection>(conn_string));
        }
    }

    ConnectionPool(const ConnectionPool&) = delete;
    ConnectionPool& operator=(const ConnectionPool&) = delete;

    // RAII guard: ยืม connection ตอน construct คืนกลับ pool ตอน destruct เสมอ
    // (แม้จะออกจากฟังก์ชันด้วย exception ก็ตาม — เหมือนหลักการเดียวกับ std::lock_guard)
    class Guard {
    public:
        Guard(ConnectionPool& pool, std::unique_ptr<pqxx::connection> conn)
            : pool_(pool), conn_(std::move(conn)) {}

        ~Guard() {
            if (conn_) pool_.release(std::move(conn_));
        }

        Guard(const Guard&) = delete;
        Guard& operator=(const Guard&) = delete;
        Guard(Guard&&) = default;

        pqxx::connection& operator*() { return *conn_; }

    private:
        ConnectionPool& pool_;
        std::unique_ptr<pqxx::connection> conn_;
    };

    // ยืม connection ออกจาก pool — ถ้า pool ว่างพอดี (ทุก connection ถูกยืมหมด) thread
    // ที่เรียกจะ "รอ" (block) จน thread อื่นคืน connection กลับมาก่อน ผ่าน condition_variable
    // (ไม่ใช่ busy-wait) นี่คือกลไกที่จำกัดจำนวน concurrent database access สูงสุด = ขนาด pool
    Guard acquire() {
        std::unique_lock<std::mutex> lock(mutex_);
        cv_.wait(lock, [this] { return !available_.empty(); });
        std::unique_ptr<pqxx::connection> conn = std::move(available_.back());
        available_.pop_back();
        return Guard(*this, std::move(conn));
    }

private:
    void release(std::unique_ptr<pqxx::connection> conn) {
        {
            std::lock_guard<std::mutex> lock(mutex_);
            available_.push_back(std::move(conn));
        }
        cv_.notify_one();
    }

    std::vector<std::unique_ptr<pqxx::connection>> available_;
    std::mutex mutex_;
    std::condition_variable cv_;
};
```

จุดที่ต้องเข้าใจให้ลึก:

1. **`std::unique_ptr<pqxx::connection>`**: เก็บเป็น pointer ไม่ใช่ value ตรงๆ เพราะ
   `pqxx::connection` ไม่รองรับการย้าย (move) แบบง่ายในบางเวอร์ชัน และเราต้องการ "ย้าย
   ความเป็นเจ้าของ" connection ไปมาระหว่าง pool กับ Guard โดยไม่ก็อปปี้
2. **`cv_.wait(lock, predicate)`**: ถ้า pool ว่าง (`available_.empty()`) thread ที่เรียก
   `acquire()` จะถูก "พักการทำงาน" (block แบบไม่กิน CPU) จนกว่าจะมี thread อื่นเรียก
   `release()` แล้วปลุก (`notify_one()`) — นี่คือกลไกที่ทำให้ pool ทำหน้าที่เป็น **Semaphore
   จำกัดจำนวน concurrent database access สูงสุดเท่ากับขนาด pool** ไปพร้อมกันในตัว
3. **RAII (`Guard` destructor)**: ไม่ว่าฟังก์ชันที่ยืม connection จะจบแบบปกติ หรือจบเพราะ
   exception หลุดออกมา (เช่น `InsufficientStockError` ใน `place_order()`) `Guard` จะคืน
   connection กลับ pool เสมอ — ไม่มีทาง "connection รั่ว" (leak) ค้างอยู่นอก pool ได้เลย

แก้ `Db` ให้ใช้ pool แทน connection ตัวเดียว (`db.hpp` ในหัวข้อ 121.5 คือเวอร์ชันที่แก้แล้ว):

```cpp
// db.hpp — ก่อนแก้: pqxx::connection conn_;
// db.hpp — หลังแก้:
private:
    ConnectionPool pool_;
```

และทุกเมธอดของ `Db` เปลี่ยนจาก:

```cpp
pqxx::work txn(conn_);   // ❌ ก่อนแก้: ใช้ connection เดียวกันทุก thread
```

เป็น:

```cpp
auto conn = pool_.acquire();  // ✅ หลังแก้: ยืม connection ของตัวเองมาใช้
pqxx::work txn(*conn);
```

(ครบทุกเมธอดตามที่แสดงเต็มรูปแบบใน 121.5–121.6)

`main.cpp` ส่ง `pool_size` เข้าไปให้ `Db` และอ่านค่าจาก environment variable ได้ (สำหรับปรับ
ขนาด pool ตอน deploy จริงโดยไม่ต้อง compile ใหม่):

```cpp
std::size_t pool_size = std::stoul(get_env_or("DB_POOL_SIZE", "8"));
Db db(conn_string, pool_size);
```

ตั้งค่า default ไว้ที่ 8 (มากกว่าจำนวน worker thread ของ Crow บนเครื่องนี้ที่มี 4 thread) เพื่อ
ไม่ให้ request ต้องรอคิวกันเปล่าๆ ทั้งที่ CPU core ยังว่างอยู่ — กฎทั่วไปคือตั้ง pool size ให้
**มากกว่าหรือเท่ากับ** จำนวน worker thread เสมอ

### ขั้นที่ 5: Build ใหม่และพิสูจน์ว่าแก้ได้จริง

```bash
$ cmake --build build -j4
[ 37%] Building CXX object CMakeFiles/ecommerce_api.dir/src/routes_catalog.cpp.o
[ 37%] Building CXX object CMakeFiles/ecommerce_api.dir/src/routes_auth.cpp.o
[ 37%] Building CXX object CMakeFiles/ecommerce_api.dir/src/db.cpp.o
[ 50%] Building CXX object CMakeFiles/ecommerce_api.dir/src/main.cpp.o
[ 62%] Building CXX object CMakeFiles/ecommerce_api.dir/src/routes_orders.cpp.o
[ 75%] Building CXX object CMakeFiles/ecommerce_api.dir/src/jwt.cpp.o
[ 87%] Building CXX object CMakeFiles/ecommerce_api.dir/src/password.cpp.o
[100%] Linking CXX executable ecommerce_api
[100%] Built target ecommerce_api
```

Build ผ่านสะอาด ไม่มี warning เช่นเดิม รีเซ็ตฐานข้อมูล สร้างสินค้าทดสอบใหม่ (stock=10) แล้วรัน
`concurrency_test.sh` **ตัวเดิมทุกบรรทัด** อีกครั้ง:

```bash
$ ./concurrency_test.sh
ยิง 30 คำขอ POST /orders พร้อมกัน (แข่งกันซื้อสินค้า id=5 ที่มี stock เริ่มต้น = 10)...
--- สรุปผลลัพธ์ HTTP status code จากทั้ง 30 คำขอ ---
     10 201
     20 409
สำเร็จ (201): 10 ครั้ง
ถูกปฏิเสธเพราะสต็อกไม่พอ (409): 20 ครั้ง
```

**ไม่มี `500` แม้แต่ตัวเดียวอีกต่อไป** — จำนวน `201` (สำเร็จ) เท่ากับจำนวนสต็อกเป๊ะ (10) และ
`409` (ถูกปฏิเสธ) ครบ 20 ที่เหลือพอดี (30 - 10 = 20) ตรวจสอบสถานะฐานข้อมูลจริงยืนยันอีกชั้น:

```bash
$ curl -s http://127.0.0.1:18480/products/5
{"data":{"category_id":1,"category_name":"อุปกรณ์อิเล็กทรอนิกส์","description":"ผลิตจำนวนจำกัดสำหรับทดสอบ concurrency","id":5,"name":"หูฟังรุ่นลิมิเต็ด Edition","price_cents":199000,"sku":"LTD-001","stock":0},"success":true}

$ PGPASSWORD=ecom_pass123 psql -h 127.0.0.1 -U ecomuser -d ecommercedb \
  -c "SELECT count(*) orders_count, sum(quantity) units_sold FROM order_items WHERE product_id=5;"
 orders_count | units_sold
--------------+------------
           10 |         10
(1 row)
```

**`stock = 0` พอดี ไม่ติดลบแม้แต่หน่วยเดียว** และมี order_item ที่สั่งสินค้าตัวนี้จริง **10
รายการ 10 ชิ้นพอดี** ตรงกับจำนวน `201` ที่นับได้ทุกประการ — ไม่มีการ Oversell เกิดขึ้นเลย

### ทดสอบซ้ำด้วยขนาดที่ใหญ่ขึ้น (N=50) เพื่อความมั่นใจ

เพื่อให้แน่ใจว่าผลลัพธ์ก่อนหน้าไม่ใช่ความบังเอิญ สร้างสินค้าใหม่ที่มี `stock = 20` แล้วยิง
**50 concurrent request** พร้อมกัน:

```bash
$ ./concurrency_test_50.sh
ยิง 50 คำขอ POST /orders พร้อมกัน (แข่งกันซื้อสินค้า id=8 ที่มี stock เริ่มต้น = 20)...
--- สรุปผลลัพธ์ HTTP status code จากทั้ง 50 คำขอ ---
     20 201
     30 409
```

```bash
$ curl -s http://127.0.0.1:18480/products/8
{"data":{...,"id":8,"name":"นาฬิกาสมาร์ทวอทช์รุ่นพิเศษ","sku":"LTD-002","stock":0},"success":true}

$ PGPASSWORD=ecom_pass123 psql -h 127.0.0.1 -U ecomuser -d ecommercedb \
  -c "SELECT count(*), sum(quantity) FROM order_items WHERE product_id=8;"
 count | sum
-------+-----
    20 |  20
(1 row)
```

ผลลัพธ์เหมือนเดิมทุกประการ (ไม่มี `500`, `201` เท่ากับ stock เป๊ะ, สต็อกสุดท้ายเป็น `0` ไม่ติดลบ)
— พิสูจน์ว่าการแก้ปัญหาถูกต้องและสามารถทำซ้ำได้ (reproducible) ไม่ใช่ผลบังเอิญจากการรันครั้งเดียว

### ทดสอบการป้องกัน Deadlock ด้วย Multi-Item Orders

สุดท้าย พิสูจน์ว่าการเรียง `product_id` ก่อนล็อก (หัวข้อ 121.6) ป้องกัน deadlock ได้จริง โดย
สร้างสินค้า 2 ตัวใหม่ (id=6, id=7 สต็อกตัวละ 500) แล้วยิง 40 order พร้อมกัน สลับลำดับ item ใน
body ไปมา (บางอันส่ง `[6,7]` บางอันส่ง `[7,6]`):

```bash
#!/bin/bash
# deadlock_test.sh
order_forward() {
    curl -s -o /dev/null -w "%{http_code} %{time_total}\n" -X POST "$BASE/orders" \
        -H "Authorization: Bearer $CUST_TOKEN" -H "Content-Type: application/json" \
        -d '{"items":[{"product_id":6,"quantity":1},{"product_id":7,"quantity":1}]}'
}
order_reverse() {
    curl -s -o /dev/null -w "%{http_code} %{time_total}\n" -X POST "$BASE/orders" \
        -H "Authorization: Bearer $CUST_TOKEN" -H "Content-Type: application/json" \
        -d '{"items":[{"product_id":7,"quantity":1},{"product_id":6,"quantity":1}]}'
}
# ยิงสลับ forward/reverse 40 ครั้งพร้อมกันทั้งหมด แล้ว wait
```

ผลลัพธ์จริง (ตัดมาบางส่วน):

```
$ ./deadlock_test.sh
ยิง 40 คำขอพร้อมกัน สลับลำดับ product_id ใน body ไปมา (ทดสอบ deadlock avoidance)...
201 0.007550
201 0.006961
201 0.007776
...
201 0.013021

real	0m0.152s
user	0m0.220s
sys	0m0.114s
--- สรุป HTTP status ---
     40 201
```

ทั้ง 40 request สำเร็จหมด (`201` ทุกตัว) ภายในเวลารวมแค่ **0.152 วินาที** โดยแต่ละ request ใช้
เวลาไม่เกิน 41 มิลลิวินาที — ไม่มี request ไหน "ค้าง" รอกันเป็นเวลานานผิดปกติ (ซึ่งจะเป็น
สัญญาณของ deadlock ที่รอจน PostgreSQL ตรวจจับและยกเลิกเอง ปกติใช้เวลาหลายวินาทีถึงจะ timeout)
ยืนยัน stock สุดท้ายถูกต้อง:

```bash
$ curl -s http://127.0.0.1:18480/products/6; echo
{"data":{...,"stock":460},"success":true}
$ curl -s http://127.0.0.1:18480/products/7; echo
{"data":{...,"stock":460},"success":true}
```

เริ่มต้นสต็อก 500 แต่ละตัว ถูกสั่งซื้อไป 40 ชิ้น (500 - 40 = 460) ตรงกันทั้งสองสินค้า — การล็อก
ตามลำดับ `product_id` ทำงานถูกต้อง ไม่มี deadlock เกิดขึ้นแม้แต่ครั้งเดียวตลอด 40 request ที่
จงใจส่งลำดับสลับกันไปมา

### สรุปตารางเปรียบเทียบ ก่อน/หลังแก้บั๊ก

| รายการ | ก่อนแก้ (connection เดียว) | หลังแก้ (Connection Pool) |
|---|---|---|
| จำนวน `500 Internal Server Error` จาก 30 concurrent request | 16 | **0** |
| จำนวน `201 Created` (ตรงกับ stock=10 หรือไม่) | 10 (ตรง แต่ปนกับ error) | **10 (ตรงเป๊ะ)** |
| สถานะ connection หลังทดสอบ | พัง ("Lost connection") ต้อง restart server | ปกติ ใช้งานต่อได้ทันที |
| ผลลัพธ์ทดสอบซ้ำ (reproducibility) | ไม่แน่นอน (race condition ขึ้นกับ timing) | สม่ำเสมอทุกครั้งที่รันซ้ำ (ทดสอบแล้วที่ N=30 และ N=50) |

---
