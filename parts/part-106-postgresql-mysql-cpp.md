# Part 106: เชื่อมต่อ PostgreSQL/MySQL ด้วย C++ (Step 841–848)

> Module I — Web Development ด้วย C/C++ | Part 106 จาก 125
> Step ที่ครอบคลุมใน Part นี้: Step 841–848
> Part ก่อนหน้า: [Part 105 — เชื่อมต่อฐานข้อมูล SQLite ด้วย C++](./part-105-sqlite-cpp.md) | Part ถัดไป: [Part 107 — การจัดการ JSON ด้วย nlohmann/json](./part-107-json-nlohmann.md)

## เป้าหมายการเรียนรู้ (Learning Objectives)

หลังจบ Part นี้ ผู้เรียนจะสามารถ:

1. อธิบายความแตกต่างระหว่าง **embedded database** (SQLite) กับ **client-server database**
   (PostgreSQL/MySQL) ได้อย่างชัดเจน ทั้งในแง่สถาปัตยกรรม, การจัดการผู้ใช้พร้อมกัน, และการ deploy
2. ติดตั้งและตั้งค่า PostgreSQL server จริง (สร้าง role และ database) แล้วเชื่อมต่อจาก C++
   ด้วยไลบรารี **libpqxx** ได้สำเร็จ
3. ใช้ `pqxx::connection`, `pqxx::work` (transaction) และ `exec_params` เพื่อรัน
   **parameterized query** ที่ป้องกัน SQL Injection ได้อย่างสมบูรณ์
4. ทำ **CRUD ครบวงจร** บน PostgreSQL จริง พร้อมเข้าใจว่าทำไมการ "ลืม commit" ถึงทำให้ข้อมูลหาย
   ไปทั้งหมดโดยไม่มี error ใดๆ แจ้งเตือน
5. อธิบายแนวคิดพื้นฐานของ **Connection Pooling** และพิสูจน์ด้วยการวัดเวลาจริงว่าทำไมมันถึงสำคัญ
   มากสำหรับ Web Server ที่ต้องรับหลาย request พร้อมกัน
6. เขียนโค้ดเชื่อมต่อ **MySQL ด้วย MySQL Connector/C++** เบื้องต้น และเข้าใจว่า syntax ใกล้เคียง
   กับ libpqxx ในเชิงแนวคิดอย่างไร
7. เลือกใช้ SQLite, PostgreSQL หรือ MySQL ได้อย่างเหมาะสมตามลักษณะของโปรเจกต์จริง

---

## 106.1 Embedded Database vs Client-Server Database (Step 841)

Part 105 แนะนำ SQLite ในฐานะ **embedded database** — ฐานข้อมูลทั้งหมดอยู่ในไฟล์เดียว และ
โปรแกรมของเราลิงก์กับไลบรารีแล้วอ่าน/เขียนไฟล์นั้นโดยตรง Part นี้จะแนะนำสถาปัตยกรรมที่ต่างกัน
โดยสิ้นเชิงที่เรียกว่า **client-server database** ซึ่ง PostgreSQL และ MySQL ทั้งคู่ใช้

### สถาปัตยกรรมที่ต่างกัน

```
SQLite (Embedded)                    PostgreSQL/MySQL (Client-Server)
┌─────────────────────┐              ┌─────────────────────┐      ┌──────────────────┐
│   โปรแกรม C++        │              │   โปรแกรม C++ #1     │      │                  │
│   + libsqlite3       │              │   (libpqxx client)   │─────▶│                  │
│         │            │              └─────────────────────┘      │  PostgreSQL      │
│         ▼            │              ┌─────────────────────┐      │  Server Process  │
│   school.db (ไฟล์)   │              │   โปรแกรม C++ #2     │─────▶│  (พอร์ต 5432)    │
└─────────────────────┘              │   (libpqxx client)   │      │                  │
  ไม่มี process แยก                   └─────────────────────┘      │  ไฟล์ข้อมูลจริง   │
  โปรแกรมคือผู้เข้าถึงไฟล์เอง          ┌─────────────────────┐      │  อยู่ฝั่ง server  │
                                      │   psql (CLI client)  │─────▶│                  │
                                      └─────────────────────┘      └──────────────────┘
                                        ทุก client คุยผ่าน network protocol เดียวกัน
```

| ประเด็น | SQLite (Embedded) | PostgreSQL / MySQL (Client-Server) |
|---|---|---|
| ต้องมี server process รันอยู่หรือไม่ | ไม่ต้อง | **ต้องมี** (`postgres`, `mysqld`) รันอยู่เบื้องหลังตลอดเวลา |
| การเข้าถึงข้อมูล | โปรแกรมอ่าน/เขียนไฟล์ `.db` โดยตรง | โปรแกรม (client) ส่ง SQL ผ่าน network protocol ไปให้ server ประมวลผล |
| ผู้ใช้พร้อมกันหลายคน/หลายเครื่อง | ทำได้จำกัด (ล็อกทั้งไฟล์เวลาเขียน) | ออกแบบมาเพื่อรองรับ client จำนวนมากพร้อมกันโดยเฉพาะ |
| ระบบสิทธิ์ผู้ใช้ (User/Role) | ไม่มี — ใครเข้าถึงไฟล์ได้ก็ใช้ฐานข้อมูลได้ | มีระบบ role/permission ละเอียดระดับตาราง/คอลัมน์ |
| การ deploy | copy ไฟล์ `.db` ไปที่ไหนก็ใช้ได้ทันที | ต้องติดตั้งและดูแล server แยกต่างหาก (หรือใช้ managed service) |
| Scaling | ยากมาก (ไฟล์เดียว, เครื่องเดียว) | รองรับ replication, read replica, clustering |
| ความหน่วง (latency) | ต่ำมาก (ไม่มี network round-trip) | มี network round-trip เสมอ แม้จะอยู่เครื่องเดียวกัน (ผ่าน localhost) |

**ข้อสรุปสำคัญ**: PostgreSQL/MySQL ไม่ได้ "ดีกว่า" SQLite เสมอไป — มันคือเครื่องมือที่ออกแบบมา
เพื่อแก้ปัญหาคนละแบบ Web application ระดับ production ที่มีผู้ใช้หลายพัน/หมื่นคนเข้าถึงพร้อมกัน
จำเป็นต้องใช้ client-server database เพราะต้องมี server กลางที่จัดการ concurrent access, การ
ล็อกระดับแถว (row-level locking), และการรับประกัน consistency ที่ซับซ้อนกว่าที่ SQLite ทำได้

---

## 106.2 ติดตั้งและตั้งค่า PostgreSQL Server จริง (Step 842)

ก่อนเขียนโค้ด C++ ใดๆ เราต้องมี PostgreSQL server รันอยู่ก่อน และต้องสร้าง **role** (ผู้ใช้) กับ
**database** สำหรับใช้ในบทเรียนนี้

### ตรวจสอบและเปิด PostgreSQL

```bash
# ตรวจสอบว่ามี cluster ติดตั้งอยู่หรือไม่ และสถานะปัจจุบัน
pg_lsclusters
```

```
Ver Cluster Port Status Owner    Data directory              Log file
16  main    5432 online postgres /var/lib/postgresql/16/main /var/log/postgresql/postgresql-16-main.log
```

ถ้า `Status` ขึ้นเป็น `down` ให้เริ่ม service ด้วยคำสั่ง:

```bash
pg_ctlcluster 16 main start
```

### สร้าง Role และ Database สำหรับบทเรียนนี้

PostgreSQL แยก **role** (บัญชีผู้ใช้/สิทธิ์) ออกจาก **database** (ที่เก็บข้อมูล) อย่างชัดเจน
เราจะสร้างทั้งสองอย่างผ่าน `psql` โดย login ในฐานะ superuser `postgres` (บัญชีที่ถูกสร้างมาให้
ตอนติดตั้งเสมอ):

```bash
sudo -u postgres psql <<'EOF'
DROP DATABASE IF EXISTS coursedb;
DROP ROLE IF EXISTS courseuser;
CREATE ROLE courseuser WITH LOGIN PASSWORD 'course_pass123';
CREATE DATABASE coursedb OWNER courseuser;
EOF
```

ผลลัพธ์จริงที่ได้:

```
NOTICE:  database "coursedb" does not exist, skipping
DROP DATABASE
NOTICE:  role "courseuser" does not exist, skipping
DROP ROLE
CREATE ROLE
CREATE DATABASE
```

จากนั้นทดสอบว่า role ใหม่นี้เชื่อมต่อผ่าน TCP (`127.0.0.1`) ด้วย password ได้จริงหรือไม่ —
ขั้นตอนนี้สำคัญเพราะโปรแกรม C++ ของเราจะเชื่อมต่อผ่าน TCP เสมอ (ไม่ใช่ผ่าน Unix socket แบบที่
`sudo -u postgres psql` ใช้):

```bash
PGPASSWORD=course_pass123 psql -h 127.0.0.1 -U courseuser -d coursedb \
    -c "SELECT current_user, current_database();"
```

ผลลัพธ์จริง:

```
current_user | current_database
--------------+------------------
 courseuser   | coursedb
(1 row)
```

เมื่อทดสอบผ่านแล้วแปลว่าโปรแกรม C++ จะเชื่อมต่อด้วย connection string
`host=127.0.0.1 port=5432 dbname=coursedb user=courseuser password=course_pass123` ได้แน่นอน

> **หมายเหตุด้านความปลอดภัย**: ในตัวอย่างนี้ hard-code password ไว้ในโค้ดเพื่อความง่ายต่อการ
> เรียนรู้เท่านั้น ในโปรเจกต์จริงต้องอ่าน credential จาก **environment variable** หรือไฟล์
> config ที่ไม่ถูก commit เข้า Git repository เด็ดขาด (จะเจาะลึกเรื่อง secret management ใน
> Part 112 และ 115)

### ติดตั้งไลบรารี libpqxx

```bash
sudo apt install libpqxx-dev -y
pkg-config --modversion libpqxx
```

ผลลัพธ์บนเครื่องที่ใช้เขียนบทเรียนนี้:

```
7.8.1
```

`libpqxx` เป็นไลบรารี C++ อย่างเป็นทางการสำหรับ PostgreSQL (ต่างจาก `libpq` ซึ่งเป็น C API
ระดับต่ำที่ `libpqxx` ห่ออยู่อีกชั้นหนึ่งให้ใช้งานสไตล์ C++ ที่ปลอดภัยกว่า มี RAIIในตัว และมี
exception handling ครบ) เวลาคอมไพล์ต้อง link ทั้ง `-lpqxx` และ `-lpq`

---

## 106.3 เชื่อมต่อ PostgreSQL ครั้งแรกด้วย libpqxx (Step 843)

```cpp
// 01_pqxx_connect.cpp - เชื่อมต่อ PostgreSQL ครั้งแรกด้วย libpqxx
#include <pqxx/pqxx>
#include <iostream>

int main(void) {
    try {
        // Connection string แบบ key=value (libpqxx รองรับ format เดียวกับ libpq)
        pqxx::connection conn(
            "host=127.0.0.1 port=5432 dbname=coursedb "
            "user=courseuser password=course_pass123"
        );

        std::cout << "เชื่อมต่อสำเร็จ! ฐานข้อมูล: " << conn.dbname() << '\n';
        std::cout << "PostgreSQL server version: " << conn.server_version() << '\n';

    } catch (const std::exception& e) {
        std::cerr << "เชื่อมต่อล้มเหลว: " << e.what() << '\n';
        return 1;
    }
    return 0;
}
```

```bash
g++ -Wall -Wextra -std=c++17 01_pqxx_connect.cpp -o 01_pqxx_connect -lpqxx -lpq
./01_pqxx_connect
```

ผลลัพธ์จริง:

```
เชื่อมต่อสำเร็จ! ฐานข้อมูล: coursedb
PostgreSQL server version: 160015
```

(`160015` คือรูปแบบตัวเลขเวอร์ชันของ PostgreSQL ที่ libpqxx คืนมา หมายถึงเวอร์ชัน 16.15)

### สังเกตความต่างจาก SQLite ตั้งแต่บรรทัดแรก

- `pqxx::connection` เป็น **RAII class ที่มากับไลบรารีเลย** — ไม่ต้องเขียน wrapper เองเหมือน
  `sqlite3*` ใน Part 105 เพราะ libpqxx ออกแบบมาสไตล์ C++ ตั้งแต่ต้น (ต่างจาก `sqlite3.h` ที่เป็น
  C API ดิบ)
- การเชื่อมต่อล้มเหลว (เช่น server ไม่ได้เปิด, password ผิด, database ไม่มีอยู่) จะ **โยน
  exception ออกมาทันที** ไม่ใช่การคืน return code แบบ `sqlite3_open` — เราจึงต้องครอบด้วย
  `try`/`catch` ตั้งแต่บรรทัดสร้าง `pqxx::connection`
- Connection string เป็นรูปแบบ `key=value key=value ...` ซึ่งเป็นมาตรฐานของ `libpq` ที่ไลบรารี
  PostgreSQL client แทบทุกภาษาใช้ร่วมกัน (Python's psycopg2, Node's `pg`, Go's `pq` ก็ใช้
  รูปแบบใกล้เคียงกัน)

---

## 106.4 Transaction และ Parameterized Query ด้วย pqxx::work (Step 844)

libpqxx บังคับให้ทุกคำสั่ง SQL ต้องรันผ่าน **transaction object** เสมอ (แม้แต่ `SELECT` ธรรมดา
ก็ต้องอยู่ใน transaction) คลาสที่ใช้บ่อยที่สุดคือ `pqxx::work` ซึ่งเป็น transaction แบบ
read-write มาตรฐาน

```cpp
pqxx::work txn(conn);              // เปิด transaction
pqxx::result r = txn.exec("...");  // รันคำสั่ง SQL ภายใน transaction นี้
txn.commit();                       // ยืนยันการเปลี่ยนแปลงให้ถาวร (สำคัญมาก - ดูหัวข้อ 106.6)
```

สำหรับคำสั่งที่มีค่าจากผู้ใช้ปนอยู่ ให้ใช้ **`exec_params`** แทน `exec` เสมอ — มันทำหน้าที่
เดียวกับ prepared statement + bind parameter ใน SQLite (Part 105.5) โดย libpqxx จัดการส่งค่า
แยกจาก SQL text ให้อัตโนมัติ ผ่าน placeholder แบบ `$1`, `$2`, `$3`, ...

```cpp
pqxx::result r = txn.exec_params(
    "INSERT INTO employees (name, department, salary) VALUES ($1, $2, $3) RETURNING id;",
    "Somchai", "Engineering", 45000.00
);
int new_id = r[0]["id"].as<int>();
```

จุดที่ต่างจาก SQLite อย่างเห็นได้ชัด: PostgreSQL รองรับ `RETURNING` clause ทำให้ดึงค่า
`id` ที่เพิ่ง insert กลับมาได้ในคำสั่งเดียว โดยไม่ต้องเรียกฟังก์ชันแยกแบบ
`sqlite3_last_insert_rowid()`

---

## 106.5 CRUD ครบวงจรบน PostgreSQL จริง (Step 845)

มาลองทำ CRUD ครบทั้ง 4 การกระทำบนฐานข้อมูล `coursedb` ที่ตั้งค่าไว้ในหัวข้อ 106.2:

```cpp
// 02_pqxx_crud.cpp - CRUD ครบวงจรบน PostgreSQL ด้วย pqxx::work (transaction)
#include <pqxx/pqxx>
#include <iostream>
#include <iomanip>

static void print_all_employees(pqxx::connection& conn) {
    pqxx::work txn(conn); // เปิด transaction สำหรับอ่านข้อมูล (read-only ก็ยังต้องอยู่ใน transaction)
    pqxx::result rows = txn.exec("SELECT id, name, department, salary FROM employees ORDER BY id;");
    txn.commit();

    std::cout << std::left << std::setw(4) << "id" << std::setw(12) << "name"
               << std::setw(14) << "department" << "salary\n";
    std::cout << "--------------------------------------\n";
    for (const auto& row : rows) {
        std::cout << std::left << std::setw(4) << row["id"].as<int>()
                   << std::setw(12) << row["name"].as<std::string>()
                   << std::setw(14) << row["department"].as<std::string>()
                   << row["salary"].as<double>() << '\n';
    }
}

int main(void) {
    try {
        pqxx::connection conn(
            "host=127.0.0.1 port=5432 dbname=coursedb "
            "user=courseuser password=course_pass123"
        );

        // ----- CREATE TABLE -----
        {
            pqxx::work txn(conn);
            txn.exec("DROP TABLE IF EXISTS employees;");
            txn.exec(
                "CREATE TABLE employees ("
                "  id SERIAL PRIMARY KEY,"
                "  name TEXT NOT NULL,"
                "  department TEXT NOT NULL,"
                "  salary NUMERIC(10,2) NOT NULL"
                ");"
            );
            txn.commit(); // ต้อง commit() เสมอ ไม่งั้น DDL/DML จะถูก rollback ตอน txn หมด scope
            std::cout << "[CREATE] สร้างตาราง employees สำเร็จ\n\n";
        }

        // ----- INSERT ด้วย exec_params (parameterized query ป้องกัน SQL injection) -----
        {
            pqxx::work txn(conn);
            pqxx::result r1 = txn.exec_params(
                "INSERT INTO employees (name, department, salary) VALUES ($1, $2, $3) RETURNING id;",
                "Somchai", "Engineering", 45000.00
            );
            int id1 = r1[0]["id"].as<int>();

            pqxx::result r2 = txn.exec_params(
                "INSERT INTO employees (name, department, salary) VALUES ($1, $2, $3) RETURNING id;",
                "Suda", "Marketing", 38000.00
            );
            int id2 = r2[0]["id"].as<int>();

            pqxx::result r3 = txn.exec_params(
                "INSERT INTO employees (name, department, salary) VALUES ($1, $2, $3) RETURNING id;",
                "Anan", "Engineering", 52000.00
            );
            int id3 = r3[0]["id"].as<int>();

            txn.commit();
            std::cout << "[INSERT] เพิ่มพนักงาน 3 คน id = " << id1 << ", " << id2 << ", " << id3 << "\n\n";
        }

        std::cout << "[SELECT] รายชื่อทั้งหมดหลัง insert:\n";
        print_all_employees(conn);
        std::cout << '\n';

        // ----- UPDATE -----
        {
            pqxx::work txn(conn);
            pqxx::result r = txn.exec_params(
                "UPDATE employees SET salary = $1 WHERE name = $2;",
                48000.00, "Somchai"
            );
            txn.commit();
            std::cout << "[UPDATE] ปรับเงินเดือน Somchai (แถวที่เปลี่ยน = "
                       << r.affected_rows() << ")\n\n";
        }
        std::cout << "[SELECT] รายชื่อทั้งหมดหลัง update:\n";
        print_all_employees(conn);
        std::cout << '\n';

        // ----- DELETE -----
        {
            pqxx::work txn(conn);
            pqxx::result r = txn.exec_params(
                "DELETE FROM employees WHERE name = $1;", "Anan"
            );
            txn.commit();
            std::cout << "[DELETE] ลบ Anan (แถวที่ถูกลบ = " << r.affected_rows() << ")\n\n";
        }
        std::cout << "[SELECT] รายชื่อทั้งหมดหลัง delete:\n";
        print_all_employees(conn);

    } catch (const std::exception& e) {
        std::cerr << "เกิดข้อผิดพลาด: " << e.what() << '\n';
        return 1;
    }
    return 0;
}
```

```bash
g++ -Wall -Wextra -std=c++17 02_pqxx_crud.cpp -o 02_pqxx_crud -lpqxx -lpq
./02_pqxx_crud
```

ผลลัพธ์จริง:

```
[CREATE] สร้างตาราง employees สำเร็จ

[INSERT] เพิ่มพนักงาน 3 คน id = 1, 2, 3

[SELECT] รายชื่อทั้งหมดหลัง insert:
id  name        department    salary
--------------------------------------
1   Somchai     Engineering   45000
2   Suda        Marketing     38000
3   Anan        Engineering   52000

[UPDATE] ปรับเงินเดือน Somchai (แถวที่เปลี่ยน = 1)

[SELECT] รายชื่อทั้งหมดหลัง update:
id  name        department    salary
--------------------------------------
1   Somchai     Engineering   48000
2   Suda        Marketing     38000
3   Anan        Engineering   52000

[DELETE] ลบ Anan (แถวที่ถูกลบ = 1)

[SELECT] รายชื่อทั้งหมดหลัง delete:
id  name        department    salary
--------------------------------------
1   Somchai     Engineering   48000
2   Suda        Marketing     38000
```

### ตารางฟังก์ชันสำคัญของ libpqxx

| ฟังก์ชัน/คลาส | หน้าที่ |
|---|---|
| `pqxx::connection(connection_string)` | เปิด connection ไปยัง PostgreSQL server |
| `pqxx::work txn(conn)` | เปิด transaction แบบอ่าน-เขียน |
| `txn.exec(sql)` | รัน SQL ที่ไม่มี parameter (หรือมีค่าคงที่ที่ปลอดภัยอยู่แล้ว) |
| `txn.exec_params(sql, args...)` | รัน SQL พร้อม parameter ที่ bind แยกจาก text (ป้องกัน injection) |
| `txn.commit()` | ยืนยันการเปลี่ยนแปลงทั้งหมดใน transaction ให้ถาวร |
| `row["column_name"].as<T>()` | อ่านค่าจากคอลัมน์ แปลงเป็นชนิด `T` |
| `result.affected_rows()` | จำนวนแถวที่ได้รับผลกระทบจาก `INSERT`/`UPDATE`/`DELETE` |

---

## 106.6 Pitfall สำคัญ: SQL Injection และการลืม Commit (Step 846)

### SQL Injection บน PostgreSQL

แม้จะเป็นฐานข้อมูลคนละตัวกับ SQLite แต่ช่องโหว่ SQL Injection ที่เจอใน Part 105 เกิดขึ้นได้
เหมือนกันทุกประการ ถ้ายังต่อ string SQL ตรงๆ:

```cpp
// 04_pqxx_injection.cpp - SQL Injection บน PostgreSQL: string concat (อันตราย) vs exec_params (ปลอดภัย)
#include <pqxx/pqxx>
#include <iostream>

static void search_vulnerable(pqxx::connection& conn, const std::string& name) {
    pqxx::work txn(conn);
    // อันตราย: ต่อ string SQL ตรงๆ จากค่าที่ผู้ใช้ป้อน
    std::string sql = "SELECT id, name FROM employees WHERE name = '" + name + "';";
    std::cout << "SQL: " << sql << '\n';
    try {
        pqxx::result r = txn.exec(sql);
        txn.commit();
        for (const auto& row : r) {
            std::cout << "  พบ: id=" << row["id"].as<int>() << " name=" << row["name"].as<std::string>() << '\n';
        }
        std::cout << "  จำนวนแถวที่พบ: " << r.size() << "\n\n";
    } catch (const std::exception& e) {
        std::cout << "  เกิด error: " << e.what() << "\n\n";
    }
}

static void search_safe(pqxx::connection& conn, const std::string& name) {
    pqxx::work txn(conn);
    // ปลอดภัย: ค่าที่ผู้ใช้ป้อนถูกส่งแยกจาก SQL text ผ่าน parameter $1 เสมอ
    pqxx::result r = txn.exec_params(
        "SELECT id, name FROM employees WHERE name = $1;", name
    );
    txn.commit();
    std::cout << "parameterized query ด้วยค่า: \"" << name << "\"\n";
    for (const auto& row : r) {
        std::cout << "  พบ: id=" << row["id"].as<int>() << " name=" << row["name"].as<std::string>() << '\n';
    }
    std::cout << "  จำนวนแถวที่พบ: " << r.size() << "\n\n";
}

int main(void) {
    pqxx::connection conn(
        "host=127.0.0.1 port=5432 dbname=coursedb "
        "user=courseuser password=course_pass123"
    );

    std::cout << "== แบบอันตราย (string concat) ==\n";
    search_vulnerable(conn, "Somchai");
    search_vulnerable(conn, "' OR '1'='1");

    std::cout << "== แบบปลอดภัย (exec_params) ==\n";
    search_safe(conn, "Somchai");
    search_safe(conn, "' OR '1'='1");

    return 0;
}
```

```bash
g++ -Wall -Wextra -std=c++17 04_pqxx_injection.cpp -o 04_pqxx_injection -lpqxx -lpq
./04_pqxx_injection
```

ผลลัพธ์จริง:

```
== แบบอันตราย (string concat) ==
SQL: SELECT id, name FROM employees WHERE name = 'Somchai';
  พบ: id=1 name=Somchai
  จำนวนแถวที่พบ: 1

SQL: SELECT id, name FROM employees WHERE name = '' OR '1'='1';
  พบ: id=2 name=Suda
  พบ: id=1 name=Somchai
  จำนวนแถวที่พบ: 2

== แบบปลอดภัย (exec_params) ==
parameterized query ด้วยค่า: "Somchai"
  พบ: id=1 name=Somchai
  จำนวนแถวที่พบ: 1

parameterized query ด้วยค่า: "' OR '1'='1"
  จำนวนแถวที่พบ: 0
```

เห็นได้ชัดว่าการค้นหาแบบ string concat ด้วย payload `' OR '1'='1` คืนพนักงาน **ทุกคน**
กลับมา (2 คนที่เหลือในตาราง) ทั้งที่ผู้ใช้ตั้งใจค้นหาคนเดียว ในขณะที่ `exec_params` ปฏิบัติค่า
เดียวกันนี้เป็น **ชื่อพนักงานตามตัวอักษร** (ไม่มีพนักงานคนไหนชื่อแบบนั้นจริงๆ) จึงคืนผลลัพธ์
ว่างเปล่าอย่างถูกต้อง

### Pitfall: ลืมเรียก commit()

ปัญหาที่พบบ่อยมากสำหรับมือใหม่ที่ใช้ libpqxx คือ **ลืมเรียก `txn.commit()`** เมื่อ `pqxx::work`
หมด scope โดยยังไม่ได้ commit **destructor จะสั่ง rollback ให้อัตโนมัติแบบเงียบๆ** (ไม่มี error
หรือ warning ใดๆ แจ้งเตือนเลย) ทำให้ข้อมูลที่เขียนไปดูเหมือนหายไปอย่างลึกลับ:

```cpp
// 03_pqxx_forgot_commit.cpp - สาธิต pitfall: ลืม commit() ทำให้ transaction rollback อัตโนมัติ
#include <pqxx/pqxx>
#include <iostream>

int main(void) {
    pqxx::connection conn(
        "host=127.0.0.1 port=5432 dbname=coursedb "
        "user=courseuser password=course_pass123"
    );

    {
        pqxx::work txn(conn);
        txn.exec_params(
            "INSERT INTO employees (name, department, salary) VALUES ($1, $2, $3);",
            "Malee", "Sales", 30000.00
        );
        std::cout << "INSERT ทำงานแล้ว... แต่ยังไม่ได้เรียก txn.commit()\n";
        // *** ลืมเรียก txn.commit() ตรงนี้ ***
    } // <-- txn ถูกทำลายที่นี่ (จบ scope) โดยยังไม่ commit

    std::cout << "หลุดออกจาก scope ของ txn แล้ว (destructor ของ pqxx::work จะ rollback ให้อัตโนมัติ)\n\n";

    // ตรวจสอบว่า Malee ถูกบันทึกจริงหรือไม่
    pqxx::work check(conn);
    pqxx::result r = check.exec("SELECT name FROM employees WHERE name = 'Malee';");
    check.commit();

    if (r.empty()) {
        std::cout << "ผลลัพธ์: ไม่พบ Malee ในตาราง -> ข้อมูลหายไปเพราะไม่ได้ commit!\n";
    } else {
        std::cout << "ผลลัพธ์: พบ Malee (ไม่ควรเกิดขึ้นถ้า pitfall นี้เป็นจริง)\n";
    }

    return 0;
}
```

```bash
g++ -Wall -Wextra -std=c++17 03_pqxx_forgot_commit.cpp -o 03_pqxx_forgot_commit -lpqxx -lpq
./03_pqxx_forgot_commit
```

ผลลัพธ์จริง:

```
INSERT ทำงานแล้ว... แต่ยังไม่ได้เรียก txn.commit()
หลุดออกจาก scope ของ txn แล้ว (destructor ของ pqxx::work จะ rollback ให้อัตโนมัติ)

ผลลัพธ์: ไม่พบ Malee ในตาราง -> ข้อมูลหายไปเพราะไม่ได้ commit!
```

การออกแบบนี้ของ libpqxx จริงๆ แล้ว **ปลอดภัยกว่าการ auto-commit** ในแง่หนึ่ง เพราะถ้าเกิด
exception กลางทางระหว่างทำงานหลายคำสั่งใน transaction เดียวกัน การ rollback อัตโนมัติจะ
ป้องกันไม่ให้ข้อมูลเสียหายครึ่งๆ กลางๆ แต่ก็หมายความว่า **ต้องจำให้ขึ้นใจว่าต้องเรียก
`commit()` เองเสมอเมื่อต้องการให้การเปลี่ยนแปลงมีผลจริง**

---

## 106.7 แนวคิดเบื้องต้นของ Connection Pooling (Step 847)

การเปิด `pqxx::connection` ใหม่แต่ละครั้งไม่ใช่การกระทำที่ถูกและเร็ว เพราะเบื้องหลังต้องมีการ
**TCP handshake, PostgreSQL authentication, และการจัดสรร process/memory ฝั่ง server** ทุกครั้ง
ถ้า Web Server ของเราเปิด connection ใหม่ทุกครั้งที่มี HTTP request เข้ามา (ซึ่งอาจเกิดขึ้น
หลายร้อยครั้งต่อวินาที) ต้นทุนนี้จะกลายเป็นคอขวดสำคัญของระบบ

**Connection Pool** คือแนวคิดง่ายๆ: **เปิด connection ไว้ล่วงหน้าจำนวนหนึ่งตั้งแต่ตอน server
เริ่มทำงาน แล้วนำ connection เหล่านั้นมา "ใช้ซ้ำ" (reuse)** สำหรับทุก request แทนที่จะเปิดใหม่
ทุกครั้ง

มาพิสูจน์ด้วยการวัดเวลาจริงว่าความต่างมากแค่ไหน:

```cpp
// 06_pool_concept.cpp - แนวคิดเบื้องต้นของ Connection Pool (เปรียบเทียบเวลาที่ใช้)
#include <pqxx/pqxx>
#include <iostream>
#include <chrono>
#include <vector>
#include <memory>
#include <mutex>

// เปิด connection ใหม่ทุกครั้งที่มี request เข้ามา (ไม่มี pool) -- ไม่ควรทำแบบนี้ใน production
static void without_pool(int n_requests) {
    auto start = std::chrono::steady_clock::now();
    for (int i = 0; i < n_requests; ++i) {
        pqxx::connection conn(
            "host=127.0.0.1 port=5432 dbname=coursedb "
            "user=courseuser password=course_pass123"
        );
        pqxx::work txn(conn);
        txn.exec("SELECT 1;");
        txn.commit();
    } // conn ปิดทันทีที่จบแต่ละ iteration แล้วต้องเปิดใหม่รอบถัดไป
    auto end = std::chrono::steady_clock::now();
    std::chrono::duration<double, std::milli> ms = end - start;
    std::cout << "ไม่มี pool: " << n_requests << " requests ใช้เวลา "
               << ms.count() << " ms\n";
}

// ------- SimpleConnectionPool: ตัวอย่างแนวคิดง่ายๆ (ไม่ thread-safe เต็มรูปแบบ, เพื่อการสาธิต) -------
class SimpleConnectionPool {
public:
    SimpleConnectionPool(const std::string& conn_str, std::size_t size) {
        for (std::size_t i = 0; i < size; ++i) {
            pool_.push_back(std::make_unique<pqxx::connection>(conn_str));
        }
    }

    // ยืม connection ออกจาก pool (แบบง่าย: หมุนวน round-robin)
    pqxx::connection& acquire() {
        std::lock_guard<std::mutex> lock(mutex_);
        pqxx::connection& conn = *pool_[next_];
        next_ = (next_ + 1) % pool_.size();
        return conn;
    }

private:
    std::vector<std::unique_ptr<pqxx::connection>> pool_;
    std::size_t next_ = 0;
    std::mutex mutex_;
};

static void with_pool(int n_requests) {
    SimpleConnectionPool pool(
        "host=127.0.0.1 port=5432 dbname=coursedb "
        "user=courseuser password=course_pass123",
        4 // เปิด connection ไว้ล่วงหน้าแค่ 4 ตัว ใช้ซ้ำตลอด
    );

    auto start = std::chrono::steady_clock::now();
    for (int i = 0; i < n_requests; ++i) {
        pqxx::connection& conn = pool.acquire();
        pqxx::work txn(conn);
        txn.exec("SELECT 1;");
        txn.commit();
    }
    auto end = std::chrono::steady_clock::now();
    std::chrono::duration<double, std::milli> ms = end - start;
    std::cout << "มี pool (4 connections): " << n_requests << " requests ใช้เวลา "
               << ms.count() << " ms\n";
}

int main(void) {
    const int n = 50;
    without_pool(n);
    with_pool(n);
    return 0;
}
```

```bash
g++ -Wall -Wextra -std=c++17 -O2 06_pool_concept.cpp -o 06_pool_concept -lpqxx -lpq -pthread
./06_pool_concept
```

ผลลัพธ์จริง:

```
ไม่มี pool: 50 requests ใช้เวลา 672.269 ms
มี pool (4 connections): 50 requests ใช้เวลา 12.9843 ms
```

การมี pool ทำให้ 50 requests เร็วขึ้นกว่า **50 เท่า** ในตัวอย่างนี้ เพราะต้นทุนการเปิด
connection ใหม่ (TCP handshake + authentication) ถูกจ่ายแค่ 4 ครั้งตอนสร้าง pool เท่านั้น
ไม่ใช่ 50 ครั้งเหมือนกรณีแรก

### ทำไม Connection Pooling ถึงสำคัญมากสำหรับ Web Server

- **Web Server รับหลาย request พร้อมกัน**: จาก Part 101 เราเรียนเรื่อง Thread Pool สำหรับ
  HTTP request ไปแล้ว — แต่ละ thread ที่ประมวลผล request หนึ่งอันมักต้องคุยกับฐานข้อมูล ถ้าทุก
  thread เปิด connection ใหม่ทุกครั้ง จำนวน connection ที่ server ฐานข้อมูลต้องรับมืออาจพุ่งสูง
  จนเกิน limit ที่ตั้งไว้ (PostgreSQL มีค่า default `max_connections = 100`)
- **Pool จำกัดจำนวน connection สูงสุด**: การกำหนดขนาด pool ที่แน่นอน (เช่น 10-20 connection)
  ช่วยป้องกันไม่ให้ traffic ที่พุ่งสูงกะทันหันทำให้ฐานข้อมูลล่มจาก connection ล้น
- **ในโปรเจกต์จริงไม่ต้องเขียน pool เอง**: ตัวอย่าง `SimpleConnectionPool` ข้างต้นมีไว้เพื่อ
  สาธิตแนวคิดเท่านั้น (ยังขาดการจัดการ connection ที่หลุด, การรอคิวเมื่อ pool เต็ม, ฯลฯ) ใน
  งานจริงมักใช้ไลบรารี pool สำเร็จรูป หรือใช้ pooler ระดับ infrastructure อย่าง **PgBouncer**
  (สำหรับ PostgreSQL) วางไว้หน้าฐานข้อมูลแทน

---

## 106.8 MySQL Connector/C++ เบื้องต้น และตารางเปรียบเทียบ (Step 848)

### ติดตั้งและทดสอบ MySQL

สำหรับบทเรียนนี้ได้ติดตั้งและทดสอบ MySQL Connector/C++ กับ **MySQL server จริงที่รันอยู่บน
เครื่องเดียวกัน** (ไม่ใช่แค่โค้ดอ้างอิงเฉยๆ):

```bash
sudo apt install libmysqlcppconn-dev mysql-server -y
service mysql start

mysql -u root <<'EOF'
CREATE DATABASE coursedb;
CREATE USER 'courseuser'@'localhost' IDENTIFIED BY 'course_pass123';
GRANT ALL PRIVILEGES ON coursedb.* TO 'courseuser'@'localhost';
FLUSH PRIVILEGES;
EOF
```

ไลบรารี `libmysqlcppconn-dev` ที่ใช้ในบทเรียนนี้เป็น **MySQL Connector/C++ เวอร์ชัน 1.1**
(legacy) ซึ่งมี API สไตล์ **JDBC** (คล้ายกับ Java Database Connectivity) — ต่างจาก libpqxx ที่
เป็น API สไตล์ C++ ล้วนๆ แต่แนวคิดหลักเรื่อง prepared statement และ parameterized query
เหมือนกันทุกประการ

```cpp
// 05_mysql_connector.cpp - CRUD เบื้องต้นด้วย MySQL Connector/C++ (legacy 1.1, JDBC-style API)
#include <mysql_driver.h>
#include <mysql_connection.h>
#include <cppconn/statement.h>
#include <cppconn/prepared_statement.h>
#include <cppconn/resultset.h>
#include <iostream>
#include <memory>

int main(void) {
    try {
        sql::mysql::MySQL_Driver* driver = sql::mysql::get_mysql_driver_instance();

        std::unique_ptr<sql::Connection> conn(
            driver->connect("tcp://127.0.0.1:3306", "courseuser", "course_pass123")
        );
        conn->setSchema("coursedb");
        std::cout << "เชื่อมต่อ MySQL สำเร็จ\n";

        std::unique_ptr<sql::Statement> stmt(conn->createStatement());
        stmt->execute("DROP TABLE IF EXISTS products;");
        stmt->execute(
            "CREATE TABLE products ("
            "  id INT AUTO_INCREMENT PRIMARY KEY,"
            "  name VARCHAR(100) NOT NULL,"
            "  price DECIMAL(10,2) NOT NULL"
            ");"
        );
        std::cout << "[CREATE] สร้างตาราง products สำเร็จ\n";

        // INSERT ด้วย PreparedStatement (parameterized -> ป้องกัน injection เหมือนกับ pqxx/sqlite)
        std::unique_ptr<sql::PreparedStatement> ins(
            conn->prepareStatement("INSERT INTO products (name, price) VALUES (?, ?);")
        );
        ins->setString(1, "Keyboard");
        ins->setDouble(2, 890.00);
        ins->execute();

        ins->setString(1, "Mouse");
        ins->setDouble(2, 450.00);
        ins->execute();
        std::cout << "[INSERT] เพิ่มสินค้า 2 รายการสำเร็จ\n";

        // SELECT
        std::unique_ptr<sql::Statement> qstmt(conn->createStatement());
        std::unique_ptr<sql::ResultSet> res(
            qstmt->executeQuery("SELECT id, name, price FROM products ORDER BY id;")
        );
        std::cout << "[SELECT] รายการสินค้า:\n";
        while (res->next()) {
            std::cout << "  id=" << res->getInt("id")
                       << " name=" << res->getString("name")
                       << " price=" << res->getDouble("price") << '\n';
        }

        // UPDATE
        std::unique_ptr<sql::PreparedStatement> upd(
            conn->prepareStatement("UPDATE products SET price = ? WHERE name = ?;")
        );
        upd->setDouble(1, 990.00);
        upd->setString(2, "Keyboard");
        int affected = upd->executeUpdate();
        std::cout << "[UPDATE] ปรับราคา Keyboard (แถวที่เปลี่ยน = " << affected << ")\n";

        // DELETE
        std::unique_ptr<sql::PreparedStatement> del(
            conn->prepareStatement("DELETE FROM products WHERE name = ?;")
        );
        del->setString(1, "Mouse");
        int deleted = del->executeUpdate();
        std::cout << "[DELETE] ลบ Mouse (แถวที่ถูกลบ = " << deleted << ")\n";

        std::unique_ptr<sql::ResultSet> res2(
            qstmt->executeQuery("SELECT id, name, price FROM products ORDER BY id;")
        );
        std::cout << "[SELECT] รายการสินค้าหลัง update/delete:\n";
        while (res2->next()) {
            std::cout << "  id=" << res2->getInt("id")
                       << " name=" << res2->getString("name")
                       << " price=" << res2->getDouble("price") << '\n';
        }

    } catch (const sql::SQLException& e) {
        std::cerr << "MySQL error: " << e.what() << " (code " << e.getErrorCode() << ")\n";
        return 1;
    }
    return 0;
}
```

```bash
g++ -Wall -Wextra -std=c++17 05_mysql_connector.cpp -o 05_mysql_connector -lmysqlcppconn
./05_mysql_connector
```

ผลลัพธ์จริง (รันจริงกับ MySQL server บนเครื่องนี้):

```
เชื่อมต่อ MySQL สำเร็จ
[CREATE] สร้างตาราง products สำเร็จ
[INSERT] เพิ่มสินค้า 2 รายการสำเร็จ
[SELECT] รายการสินค้า:
  id=1 name=Keyboard price=890
  id=2 name=Mouse price=450
[UPDATE] ปรับราคา Keyboard (แถวที่เปลี่ยน = 1)
[DELETE] ลบ Mouse (แถวที่ถูกลบ = 1)
[SELECT] รายการสินค้าหลัง update/delete:
  id=1 name=Keyboard price=990
```

### เทียบ syntax ระหว่าง libpqxx กับ MySQL Connector/C++

| การกระทำ | libpqxx (PostgreSQL) | MySQL Connector/C++ |
|---|---|---|
| เชื่อมต่อ | `pqxx::connection conn("host=... user=...")` | `driver->connect("tcp://host:port", user, pass)` |
| เลือก database | รวมอยู่ใน connection string (`dbname=...`) | เรียกแยก `conn->setSchema("dbname")` |
| รัน SQL ไม่มี parameter | `txn.exec(sql)` | `stmt->execute(sql)` |
| Prepared/Parameterized | `txn.exec_params(sql, args...)` (placeholder `$1,$2,...`) | `conn->prepareStatement(sql)` แล้ว `->setString/setInt/setDouble(index, value)` (placeholder `?`) |
| อ่านผลลัพธ์ | `for (const auto& row : result)` แล้ว `row["col"].as<T>()` | `while (res->next())` แล้ว `res->getInt("col")` |
| Transaction | `pqxx::work` + `txn.commit()` (บังคับทุกคำสั่งต้องอยู่ใน transaction) | `conn->setAutoCommit(false)` แล้ว `conn->commit()` (default เป็น auto-commit) |

**ความซื่อสัตย์ทางเทคนิค**: ตัวอย่าง MySQL ด้านบนได้รับการทดสอบจริงกับ MySQL server 8.0 ที่
ติดตั้งและรันบนเครื่องเดียวกับที่ใช้เขียนบทเรียนนี้ตลอดทั้ง Part ไม่ใช่แค่โค้ดอ้างอิงที่ไม่ได้
รัน — แต่เนื่องจากเวลาที่ใช้ทดสอบ MySQL มีจำกัดกว่า PostgreSQL (ซึ่งเป็นฐานข้อมูลหลักของ
หลักสูตรนี้ตั้งแต่ Module I เป็นต้นไป และจะถูกใช้ต่อใน Part 108 และ Capstone 1) หากพบว่า MySQL
server ไม่ได้ติดตั้ง/เปิดอยู่ในสภาพแวดล้อมของผู้เรียน ให้ทำตามขั้นตอนติดตั้งด้านบนก่อน — syntax
ของ MySQL Connector/C++ ที่แสดงในหัวข้อนี้เป็นไปตาม API เวอร์ชัน 1.1.12 อย่างถูกต้องตามเอกสาร
ทางการ

---

## ตารางเปรียบเทียบ: SQLite vs PostgreSQL vs MySQL

| หัวข้อ | SQLite | PostgreSQL | MySQL |
|---|---|---|---|
| ประเภท | Embedded (ไฟล์เดียว) | Client-Server | Client-Server |
| ต้องมี server แยก | ไม่ต้อง | ต้อง | ต้อง |
| Concurrent write | จำกัดมาก (ล็อกทั้งไฟล์) | ดีมาก (MVCC, row-level lock) | ดี (ขึ้นกับ storage engine, InnoDB รองรับ row-level lock) |
| SQL Standard Compliance | ค่อนข้างสูงแต่ขาดบางฟีเจอร์ | สูงมาก ครบเครื่องที่สุดในบรรดา open source DB | ปานกลาง มี extension เฉพาะตัวหลายจุด |
| Data Type ที่โดดเด่น | Dynamic typing (ยืดหยุ่นแต่เสี่ยง) | เข้มงวด + type พิเศษเยอะ (JSONB, Array, UUID, Range) | เข้มงวดปานกลาง มี type เฉพาะ เช่น `ENUM` |
| C++ Library หลัก | `sqlite3` (C API ดิบ) | `libpqxx` (C++ native, RAII เต็มรูปแบบ) | MySQL Connector/C++ (JDBC-style API) |
| Use case ที่เหมาะ | Prototype, mobile, desktop, unit test, config เก็บข้อมูลน้อย | Web app ระดับ production, ระบบที่ต้องการ data integrity สูง, งานที่ query ซับซ้อน | Web app ทั่วไป, ระบบที่ทีมคุ้นเคยกับ ecosystem ของ MySQL (WordPress, phpMyAdmin ฯลฯ) |
| Replication/Scaling | แทบไม่มี built-in | Streaming replication, logical replication ครบเครื่อง | Master-replica replication เป็นมาตรฐานมานาน |
| ความนิยมใน Cloud (managed service) | ไม่ค่อยมี managed service (เพราะไม่ใช่ client-server) | สูงมาก (AWS RDS/Aurora, GCP Cloud SQL, Supabase) | สูงมาก (AWS RDS, GCP Cloud SQL, PlanetScale) |

**คำแนะนำสำหรับหลักสูตรนี้**: ตั้งแต่ Part 108 (โปรเจกต์ REST API CRUD) เป็นต้นไป เราจะยึด
**PostgreSQL** เป็นฐานข้อมูลหลักของหลักสูตร เพราะมี feature set ที่ครบและเข้มงวดที่สุด เหมาะกับ
การเรียนรู้แนวคิดฐานข้อมูลเชิงลึกในระดับ professional และจะถูกใช้ต่อเนื่องไปจนถึง Capstone 1
(Part 121)

---

## ข้อผิดพลาดที่พบบ่อย (Common Pitfalls)

1. **ลืมเริ่ม PostgreSQL/MySQL service ก่อนรันโปรแกรม** — ถ้า server ไม่ได้เปิดอยู่
   `pqxx::connection` หรือ `driver->connect()` จะโยน exception ทันที พร้อมข้อความประมาณ
   "Connection refused" ตรวจสอบด้วย `pg_lsclusters` (PostgreSQL) หรือ `service mysql status`
   (MySQL) ก่อนเสมอเมื่อเจอปัญหาเชื่อมต่อไม่ได้
2. **ต่อ string SQL ตรงๆ เหมือนที่เตือนไว้ใน Part 105** — ช่องโหว่เดียวกันเกิดขึ้นได้กับทุก
   ฐานข้อมูล ให้ใช้ `exec_params` (libpqxx) หรือ `PreparedStatement` (MySQL Connector/C++) เสมอ
   เมื่อมีค่าจากผู้ใช้ปนอยู่ใน query
3. **ลืมเรียก `txn.commit()`** — ตามที่สาธิตในหัวข้อ 106.6 การลืม commit จะทำให้ transaction
   ถูก rollback แบบเงียบๆ ไม่มี error ใดๆ เตือน ข้อมูลจะดูเหมือนหายไปอย่างไร้ร่องรอย ควรตรวจสอบ
   ทุกจุดที่เขียนข้อมูลว่ามี `commit()` ครบถ้วน
4. **เปิด connection ใหม่ทุกครั้งที่มี request เข้ามาในเว็บเซิร์ฟเวอร์** — เป็นสาเหตุของ
   performance bottleneck ที่พบบ่อยที่สุดอย่างหนึ่งในระบบที่เพิ่งเริ่มเขียน ให้ใช้ connection
   pool ตามแนวคิดในหัวข้อ 106.7 เสมอในโปรเจกต์ที่มีการรับ request พร้อมกัน
5. **Hard-code connection string (โดยเฉพาะ password) ลงในโค้ดแล้ว commit เข้า Git** —
   credential ต้องอ่านจาก environment variable หรือไฟล์ config ที่ไม่ถูก track โดย version
   control (จะเจาะลึกใน Part 112 และ 115)
6. **ไม่ตรวจสอบว่า `pqxx::result` ว่างเปล่าก่อนเรียก `r[0]`** — ถ้า `INSERT ... RETURNING`
   หรือ `SELECT` ไม่คืนแถวใดเลย (เช่นเงื่อนไข `WHERE` ไม่ตรงกับข้อมูลใดเลย) การเข้าถึง
   `r[0]` จะโยน exception `pqxx::usage_error` ทันที ควรเช็ค `r.empty()` ก่อนเสมอ
7. **สับสนระหว่าง auto-commit ของ MySQL กับ libpqxx ที่บังคับ transaction เสมอ** — MySQL
   Connector/C++ ค่า default เป็น auto-commit (ทุกคำสั่งมีผลทันที) ในขณะที่ libpqxx ต้อง
   `commit()` เองเสมอ นักพัฒนาที่สลับไปมาระหว่างสองไลบรารีนี้มักลืมจุดนี้จนเกิดพฤติกรรมที่ไม่
   คาดคิด

---

## แบบฝึกหัดท้ายบท

1. เขียนโปรแกรมสร้างตาราง `orders` (`id`, `customer_name`, `total_amount`, `status`) บน
   PostgreSQL แล้ว insert ออเดอร์ 4 รายการด้วย `exec_params` จากนั้นเขียนฟังก์ชัน
   `find_orders_by_status(pqxx::connection&, const std::string&)` ที่ค้นหาออเดอร์ตามสถานะ
2. ดัดแปลงตัวอย่าง `04_pqxx_injection.cpp` ให้ทดลองโจมตีด้วย payload ที่มี `;` ปนอยู่ (คล้ายกับ
   ตัวอย่าง `DROP TABLE` ใน Part 105) แล้วสังเกตว่า `txn.exec()` ของ libpqxx อนุญาตให้รันหลาย
   statement พร้อมกันหรือไม่ (คำใบ้: ลองเทียบพฤติกรรมนี้กับ `sqlite3_exec` ใน Part 105)
3. เขียนฟังก์ชัน `transfer_money(pqxx::connection&, int from_account, int to_account, double amount)`
   ที่หักเงินจากบัญชีหนึ่งและเพิ่มให้อีกบัญชีหนึ่งภายใน transaction เดียวกัน โดยต้อง `ROLLBACK`
   ทันทีถ้าบัญชีต้นทางมีเงินไม่พอ (ใช้แนวคิดเดียวกับแบบฝึกหัดข้อ 4 ของ Part 105 แต่เปลี่ยนมาใช้
   PostgreSQL)
4. ขยาย `SimpleConnectionPool` ในหัวข้อ 106.7 ให้รองรับการ "คืน" connection กลับเข้า pool อย่าง
   ชัดเจนผ่าน RAII (เขียน class `PooledConnectionGuard` ที่ยืม connection ตอน constructor และ
   คืนกลับ pool ตอน destructor ทบทวนแนวคิด RAII จาก Part 68 และ 105.6)
5. เขียนโปรแกรมเชื่อมต่อ MySQL แล้วสร้างตาราง `reviews` (`id`, `product_name`, `rating`,
   `comment`) ทำ CRUD ครบวงจรด้วย `PreparedStatement` เทียบเวลาที่ใช้เขียนโค้ดกับตัวอย่าง
   PostgreSQL ในหัวข้อ 106.5 แล้วสรุปว่า syntax ไหนที่ผู้เรียนรู้สึกว่าอ่านง่ายกว่า พร้อมเหตุผล
6. อ่านเอกสารของ **PgBouncer** แล้วอธิบายด้วยคำพูดตัวเองว่ามันแตกต่างจาก
   `SimpleConnectionPool` ที่เขียนในบทเรียนนี้อย่างไร และทำไมระบบระดับ production มักเลือกใช้
   connection pooler แบบ infrastructure-level แทนการเขียน pool เองในโค้ดแอปพลิเคชัน

### แนวทางเฉลยข้อ 1

```cpp
// exercise1.cpp
#include <pqxx/pqxx>
#include <iostream>

static void find_orders_by_status(pqxx::connection& conn, const std::string& status) {
    pqxx::work txn(conn);
    pqxx::result rows = txn.exec_params(
        "SELECT id, customer_name, total_amount FROM orders WHERE status = $1 ORDER BY id;",
        status
    );
    txn.commit();

    std::cout << "ออเดอร์สถานะ \"" << status << "\":\n";
    if (rows.empty()) {
        std::cout << "  ไม่พบออเดอร์ในสถานะนี้\n";
        return;
    }
    for (const auto& row : rows) {
        std::cout << "  [" << row["id"].as<int>() << "] "
                   << row["customer_name"].as<std::string>()
                   << " ยอดรวม " << row["total_amount"].as<double>() << " บาท\n";
    }
}

int main(void) {
    try {
        pqxx::connection conn(
            "host=127.0.0.1 port=5432 dbname=coursedb "
            "user=courseuser password=course_pass123"
        );

        {
            pqxx::work txn(conn);
            txn.exec("DROP TABLE IF EXISTS orders;");
            txn.exec(
                "CREATE TABLE orders ("
                "  id SERIAL PRIMARY KEY,"
                "  customer_name TEXT NOT NULL,"
                "  total_amount NUMERIC(10,2) NOT NULL,"
                "  status TEXT NOT NULL"
                ");"
            );
            txn.exec_params("INSERT INTO orders (customer_name, total_amount, status) VALUES ($1,$2,$3);",
                             "Somchai", 1200.00, "pending");
            txn.exec_params("INSERT INTO orders (customer_name, total_amount, status) VALUES ($1,$2,$3);",
                             "Suda", 850.00, "paid");
            txn.exec_params("INSERT INTO orders (customer_name, total_amount, status) VALUES ($1,$2,$3);",
                             "Anan", 2300.00, "pending");
            txn.exec_params("INSERT INTO orders (customer_name, total_amount, status) VALUES ($1,$2,$3);",
                             "Malee", 500.00, "cancelled");
            txn.commit();
        }

        find_orders_by_status(conn, "pending");
        find_orders_by_status(conn, "shipped"); // ไม่มีออเดอร์สถานะนี้

    } catch (const std::exception& e) {
        std::cerr << "เกิดข้อผิดพลาด: " << e.what() << '\n';
        return 1;
    }
    return 0;
}
```

ผลลัพธ์ที่คาดหวังเมื่อคอมไพล์และรัน (`g++ -Wall -Wextra -std=c++17 exercise1.cpp -o exercise1 -lpqxx -lpq`):

```
ออเดอร์สถานะ "pending":
  [1] Somchai ยอดรวม 1200 บาท
  [3] Anan ยอดรวม 2300 บาท
ออเดอร์สถานะ "shipped":
  ไม่พบออเดอร์ในสถานะนี้
```

### แนวทางเฉลยข้อ 3

```cpp
// exercise3.cpp
#include <pqxx/pqxx>
#include <iostream>

static void transfer_money(pqxx::connection& conn, int from_account,
                            int to_account, double amount) {
    pqxx::work txn(conn);
    try {
        pqxx::result balance_rows = txn.exec_params(
            "SELECT balance FROM accounts WHERE id = $1 FOR UPDATE;", from_account
        );
        if (balance_rows.empty()) {
            throw std::runtime_error("ไม่พบบัญชีต้นทาง id=" + std::to_string(from_account));
        }
        double current_balance = balance_rows[0]["balance"].as<double>();
        if (current_balance < amount) {
            throw std::runtime_error("ยอดเงินในบัญชีต้นทางไม่พอ (มี " +
                                       std::to_string(current_balance) + " ต้องการ " +
                                       std::to_string(amount) + ")");
        }

        txn.exec_params("UPDATE accounts SET balance = balance - $1 WHERE id = $2;",
                          amount, from_account);
        txn.exec_params("UPDATE accounts SET balance = balance + $1 WHERE id = $2;",
                          amount, to_account);

        txn.commit();
        std::cout << "โอนเงิน " << amount << " จากบัญชี " << from_account
                   << " ไปบัญชี " << to_account << " สำเร็จ\n";

    } catch (const std::exception& e) {
        // txn จะถูก rollback อัตโนมัติเมื่อหมด scope โดยไม่เรียก commit()
        std::cout << "ยกเลิกการโอนเงิน: " << e.what() << '\n';
    }
}

int main(void) {
    pqxx::connection conn(
        "host=127.0.0.1 port=5432 dbname=coursedb "
        "user=courseuser password=course_pass123"
    );

    {
        pqxx::work setup(conn);
        setup.exec("DROP TABLE IF EXISTS accounts;");
        setup.exec("CREATE TABLE accounts (id SERIAL PRIMARY KEY, balance NUMERIC(10,2) NOT NULL);");
        setup.exec("INSERT INTO accounts (balance) VALUES (1000.00), (200.00);"); // id 1, id 2
        setup.commit();
    }

    transfer_money(conn, 1, 2, 300.00);   // ควรสำเร็จ (บัญชี 1 มี 1000 พอโอน)
    transfer_money(conn, 2, 1, 999999.00); // ควรล้มเหลว (บัญชี 2 มีเงินไม่พอ)

    return 0;
}
```

ผลลัพธ์ที่คาดหวัง:

```
โอนเงิน 300 จากบัญชี 1 ไปบัญชี 2 สำเร็จ
ยกเลิกการโอนเงิน: ยอดเงินในบัญชีต้นทางไม่พอ (มี 500.00 ต้องการ 999999.00)
```

จุดสำคัญของเฉลยนี้คือการใช้ `SELECT ... FOR UPDATE` เพื่อล็อกแถวของบัญชีต้นทางไว้ระหว่าง
ตรวจสอบยอดเงิน ป้องกันไม่ให้มีคำสั่งโอนเงินอื่นมาแทรกกลางระหว่างที่เรากำลังตรวจสอบและอัปเดต
ยอดเงิน (race condition) — เป็นฟีเจอร์ของ PostgreSQL ที่ SQLite ไม่มีให้ใช้ในระดับนี้ เพราะ
SQLite ล็อกทั้งไฟล์อยู่แล้วเวลาเขียน

---

## สรุปท้ายบท

ใน Part นี้เราได้:

- เข้าใจความแตกต่างระหว่าง embedded database (SQLite) กับ client-server database
  (PostgreSQL/MySQL) ทั้งในแง่สถาปัตยกรรมและการใช้งานจริง
- ติดตั้งและตั้งค่า PostgreSQL server จริง สร้าง role/database และเชื่อมต่อจาก C++ ด้วย
  **libpqxx** ผ่าน `pqxx::connection`, `pqxx::work`, และ `exec_params`
- ทำ CRUD ครบวงจรบน PostgreSQL จริง พร้อมเข้าใจ pitfall สำคัญเรื่อง SQL Injection และการลืม
  `commit()` ที่ทำให้ข้อมูลหายไปแบบเงียบๆ
- พิสูจน์ด้วยตัวเลขจริงว่า **Connection Pooling** ทำให้เร็วขึ้นกว่า 50 เท่าเมื่อเทียบกับการเปิด
  connection ใหม่ทุกครั้ง
- เชื่อมต่อและทำ CRUD บน **MySQL** จริงด้วย MySQL Connector/C++ และเห็นความคล้ายคลึงและความ
  ต่างของ syntax เมื่อเทียบกับ libpqxx
- ได้ตารางเปรียบเทียบ SQLite vs PostgreSQL vs MySQL เพื่อใช้เลือกฐานข้อมูลที่เหมาะสมกับ
  โปรเจกต์ในอนาคต

ใน **Part 107** เราจะเรียนรู้การจัดการ **JSON** ด้วยไลบรารี nlohmann/json ซึ่งเป็นรูปแบบข้อมูล
มาตรฐานที่ REST API แทบทุกตัวใช้ส่งข้อมูลกลับไปให้ client — และจะนำผลลัพธ์จากการ query
ฐานข้อมูลใน Part นี้มาแปลงเป็น JSON response จริง เพื่อเตรียมพร้อมสำหรับโปรเจกต์ REST API CRUD
เต็มรูปแบบใน Part 108

**ต่อไป:** [Part 107 — การจัดการ JSON ด้วย nlohmann/json](./part-107-json-nlohmann.md)
