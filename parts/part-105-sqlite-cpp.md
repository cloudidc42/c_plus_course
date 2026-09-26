# Part 105: เชื่อมต่อฐานข้อมูล SQLite ด้วย C++ (Step 833–840)

> Module I — Web Development ด้วย C/C++ | Part 105 จาก 125
> Step ที่ครอบคลุมใน Part นี้: Step 833–840
> Part ก่อนหน้า: [Part 104 — Pistache Framework: ทางเลือกสำหรับ REST API ระดับ Production](./part-104-pistache.md) | Part ถัดไป: [Part 106 — เชื่อมต่อ PostgreSQL/MySQL ด้วย C++](./part-106-postgresql-mysql-cpp.md)

## เป้าหมายการเรียนรู้ (Learning Objectives)

หลังจบ Part นี้ ผู้เรียนจะสามารถ:

1. อธิบายได้ว่า SQLite คืออะไร ทำไมถึงเรียกว่า **embedded database** และเหมาะกับงานประเภทใด
   เมื่อเทียบกับฐานข้อมูลแบบ client-server อย่าง PostgreSQL/MySQL
2. ใช้ `sqlite3` C API พื้นฐาน (`sqlite3_open`, `sqlite3_close`, `sqlite3_exec`) เพื่อเปิด
   ฐานข้อมูล รันคำสั่ง SQL และอ่านผลลัพธ์ผ่าน callback ได้
3. อธิบายได้ว่า **SQL Injection** คืออะไร เกิดขึ้นได้อย่างไรจากการต่อ string SQL ตรงๆ และ
   สาธิตช่องโหว่จริงให้เห็นผลลัพธ์ที่เป็นอันตราย (รวมถึงการลบตารางทิ้งทั้งตาราง)
4. ใช้ **Prepared Statement** (`sqlite3_prepare_v2`, `sqlite3_bind_*`, `sqlite3_step`,
   `sqlite3_finalize`) เพื่อป้องกัน SQL Injection ได้อย่างสมบูรณ์ พร้อมอธิบายได้ว่าทำไม
   กลไกนี้ถึงปิดช่องโหว่ได้จริงในทางเทคนิค
5. เขียน **RAII wrapper class** ห่อ `sqlite3*` และ `sqlite3_stmt*` (ทบทวนแนวคิดจาก Part 68)
   เพื่อให้การเปิด/ปิด connection และ statement เป็นไปโดยอัตโนมัติ ไม่มีทางรั่วไหลของทรัพยากร
6. ทำ **CRUD ครบวงจร** (Create Table, Insert, Select, Update, Delete) ด้วยโค้ดที่คอมไพล์และ
   รันได้จริง ผ่าน wrapper class ของตัวเอง
7. ใช้ **Transaction** (`BEGIN`/`COMMIT`/`ROLLBACK`) ใน SQLite เพื่อเพิ่มความเร็วในการเขียน
   ข้อมูลจำนวนมาก และเพื่อรับประกันความถูกต้องของข้อมูลเมื่อมีหลายคำสั่งที่ต้องสำเร็จร่วมกัน

---

## 105.1 SQLite คืออะไร และทำไมต้องเรียนก่อนฐานข้อมูลอื่น (Step 833)

จนถึงตอนนี้เราสร้าง Web Server ด้วย Crow (Part 102–103) และ Pistache (Part 104) ได้แล้ว
แต่ทุก endpoint ที่เขียนมายังเก็บข้อมูลไว้ใน memory เท่านั้น (เช่น `std::vector` หรือ `std::map`
ที่ตายไปพร้อมกับโปรเซสเมื่อ server ปิด) ใน Web Application จริงแทบทุกระบบต้องมี **ฐานข้อมูล**
เพื่อเก็บข้อมูลให้อยู่ถาวร (persist) แม้ server จะ restart หรือ crash ไป

**SQLite** เป็นฐานข้อมูลเชิงสัมพันธ์ (Relational Database) ที่มีลักษณะพิเศษต่างจาก PostgreSQL
หรือ MySQL อย่างสิ้นเชิง:

- **Embedded Database**: ฐานข้อมูลทั้งหมดคือ **ไฟล์เดียว** บนดิสก์ (เช่น `school.db`) ไม่มี
  process แยกต่างหากที่ต้องรันอยู่เบื้องหลัง (ไม่มี "SQLite Server") โปรแกรมของเรา **ลิงก์กับ
  library `libsqlite3` โดยตรง** และอ่าน/เขียนไฟล์นั้นเองเลย
- **Zero-configuration**: ไม่ต้องติดตั้ง server, ไม่ต้องสร้าง user/role, ไม่ต้องเปิด port ใดๆ
  แค่มีไฟล์ `.db` (หรือแม้แต่ไม่มีไฟล์เลยก็สร้างใหม่ได้ทันที) ก็ใช้งานได้แล้ว
- **Serverless แต่ไม่ใช่ NoSQL**: SQLite ยังคงเป็นฐานข้อมูลเชิงสัมพันธ์เต็มรูปแบบ รองรับ SQL
  มาตรฐาน, `JOIN`, `TRANSACTION`, `INDEX`, `FOREIGN KEY` เกือบครบ
- **ใช้งานจริงมหาศาล**: เป็นฐานข้อมูลที่ **ถูก deploy มากที่สุดในโลก** — อยู่ในทุกเครื่อง Android/
  iOS (เก็บข้อมูล app), ทุกเบราว์เซอร์ (เก็บ history/cookies), Git LFS, และซอฟต์แวร์ desktop
  จำนวนมาก

### เมื่อไหร่ควรใช้ SQLite เมื่อไหร่ไม่ควร

| เหมาะกับ | ไม่เหมาะกับ |
|---|---|
| Prototype / MVP ที่ต้องการเริ่มเร็วโดยไม่ตั้ง server | ระบบที่มีผู้เขียนพร้อมกันจำนวนมาก (SQLite ล็อกทั้งไฟล์เวลาเขียน) |
| Mobile app, Desktop app ที่เก็บข้อมูลเฉพาะเครื่อง | Web application ขนาดใหญ่ที่มีหลาย server เข้าถึงฐานข้อมูลเดียวกันพร้อมกัน |
| Unit test ที่ต้องการฐานข้อมูลจริงแต่ไม่อยากตั้ง server แยก | ระบบที่ต้องการ replication, scaling แนวนอน |
| Embedded system / IoT ที่ทรัพยากรจำกัด | ระบบที่ต้องการ user/permission ระดับฐานข้อมูล (SQLite ไม่มีระบบสิทธิ์ผู้ใช้) |
| ไฟล์ config/cache ที่ต้อง query ได้ซับซ้อนกว่า JSON | งานที่ต้องเขียนพร้อมกันหนักๆ ตลอดเวลา (เช่น analytics เก็บ event ปริมาณสูง) |

Part นี้จะใช้ SQLite เป็น "ประตูแรก" สู่โลกของฐานข้อมูลใน C++ เพราะไม่ต้องเสียเวลาตั้ง server
ก่อน สามารถโฟกัสไปที่การเรียนรู้ **sqlite3 C API** และแนวคิดสำคัญอย่าง **Prepared Statement**
และ **SQL Injection** ได้เต็มที่ ก่อนจะไปเจอฐานข้อมูลแบบ client-server ใน Part 106

---

## 105.2 ติดตั้งและเปิดฐานข้อมูลครั้งแรกด้วย sqlite3 C API (Step 834)

SQLite เขียนด้วยภาษา C ล้วน และแจก API ในรูปแบบไฟล์ header เดียว `sqlite3.h` — ใน C++ เรา
เรียกใช้ฟังก์ชันเหล่านี้ได้ตรงๆ โดยไม่ต้องมี wrapper class ของภาษาอื่นมาคั่น

### ติดตั้งบน Ubuntu/Debian

```bash
sudo apt update
sudo apt install libsqlite3-dev -y

# ตรวจสอบเวอร์ชัน
pkg-config --modversion sqlite3
```

บนเครื่องที่ใช้เขียนบทเรียนนี้ได้ผลลัพธ์:

```
3.45.1
```

### เปิดฐานข้อมูลครั้งแรก

```cpp
// 01_sqlite_open.cpp - เปิดฐานข้อมูล SQLite ครั้งแรก
#include <sqlite3.h>
#include <cstdio>

int main(void) {
    sqlite3* db = nullptr;
    int rc = sqlite3_open("school.db", &db);

    if (rc != SQLITE_OK) {
        std::fprintf(stderr, "เปิดฐานข้อมูลไม่สำเร็จ: %s\n", sqlite3_errmsg(db));
        sqlite3_close(db);
        return 1;
    }

    std::printf("เปิดฐานข้อมูล school.db สำเร็จ (SQLite version: %s)\n", sqlite3_libversion());

    sqlite3_close(db);
    return 0;
}
```

คอมไพล์ (สังเกตว่าต้อง link `-lsqlite3` เสมอ):

```bash
g++ -Wall -Wextra -std=c++17 01_sqlite_open.cpp -o 01_sqlite_open -lsqlite3
./01_sqlite_open
```

ผลลัพธ์จริง:

```
เปิดฐานข้อมูล school.db สำเร็จ (SQLite version: 3.45.1)
```

### อธิบายทีละส่วน

- **`sqlite3* db`**: handle ที่ใช้แทน connection ไปยังฐานข้อมูล — เทียบได้กับ `FILE*` ใน C ที่
  เคยเรียนใน Part 13 คือ pointer ทึบ (opaque pointer) ที่เราไม่ควรเข้าไปยุ่งกับข้างในโดยตรง
- **`sqlite3_open("school.db", &db)`**: เปิดไฟล์ `school.db` — **ถ้าไฟล์นี้ยังไม่มีอยู่ SQLite
  จะสร้างไฟล์ใหม่ให้ทันที** (ไม่ error) จุดนี้ต่างจากฐานข้อมูล client-server ที่ต้องสร้าง
  database ไว้ล่วงหน้าเสมอ
- **`sqlite3_errmsg(db)`**: คืนข้อความ error ล่าสุดเป็น human-readable string — ใช้คู่กับการ
  ตรวจสอบ return code ทุกครั้งที่เรียก API ของ SQLite
- **`sqlite3_close(db)`**: ปิด connection และคืนทรัพยากรทั้งหมด **ต้องเรียกเสมอ** ไม่ว่าจะเปิด
  สำเร็จหรือไม่ก็ตาม (ถ้า `db` เป็น `nullptr` การเรียก `sqlite3_close` จะปลอดภัย ไม่ crash)

> สังเกตรูปแบบ **return code** ที่ SQLite ใช้ทั้งไลบรารี: เกือบทุกฟังก์ชันคืนค่า `int` โดย
> `SQLITE_OK` (ค่า 0) แปลว่าสำเร็จ ส่วนค่าอื่นคือรหัส error — นี่คือรูปแบบ error handling แบบ
> C สไตล์เดิม (ทบทวน Part 16) ที่เราต้อง `if (rc != SQLITE_OK)` ตรวจสอบเองทุกครั้ง

---

## 105.3 รันคำสั่ง SQL ด้วย sqlite3_exec และ Callback (Step 835)

วิธีที่ง่ายที่สุดในการรัน SQL แบบ "ยิงแล้วจบ" (ไม่มีการ bind parameter) คือ `sqlite3_exec`

```cpp
// 02_sqlite_exec.cpp - ใช้ sqlite3_exec สร้างตารางและ insert ข้อมูลแบบง่าย
#include <sqlite3.h>
#include <cstdio>
#include <cstdlib>

// callback ที่ sqlite3_exec จะเรียกกลับมาให้ทุกครั้งที่มี 1 row จากผลลัพธ์ SELECT
static int print_row_callback(void* /*not_used*/, int argc, char** argv, char** col_names) {
    for (int i = 0; i < argc; ++i) {
        std::printf("  %s = %s\n", col_names[i], argv[i] ? argv[i] : "NULL");
    }
    std::printf("  ---\n");
    return 0; // ต้อง return 0 เสมอ ไม่งั้น sqlite3_exec จะหยุดและถือว่าล้มเหลว
}

static void run_sql(sqlite3* db, const char* sql) {
    char* err_msg = nullptr;
    int rc = sqlite3_exec(db, sql, print_row_callback, nullptr, &err_msg);
    if (rc != SQLITE_OK) {
        std::fprintf(stderr, "SQL error: %s\n", err_msg);
        sqlite3_free(err_msg); // error message ที่ sqlite3_exec คืนมาต้อง free เองด้วย sqlite3_free
    }
}

int main(void) {
    sqlite3* db = nullptr;
    if (sqlite3_open("school.db", &db) != SQLITE_OK) {
        std::fprintf(stderr, "เปิดฐานข้อมูลไม่สำเร็จ\n");
        return 1;
    }

    const char* create_sql =
        "CREATE TABLE IF NOT EXISTS students ("
        "  id INTEGER PRIMARY KEY AUTOINCREMENT,"
        "  name TEXT NOT NULL,"
        "  score REAL NOT NULL"
        ");";
    run_sql(db, create_sql);

    const char* insert_sql =
        "INSERT INTO students (name, score) VALUES "
        "('Somchai', 85.5), ('Suda', 92.0), ('Anan', 77.25);";
    run_sql(db, insert_sql);

    std::printf("รายชื่อนักเรียนทั้งหมด:\n");
    run_sql(db, "SELECT id, name, score FROM students;");

    sqlite3_close(db);
    return 0;
}
```

```bash
g++ -Wall -Wextra -std=c++17 02_sqlite_exec.cpp -o 02_sqlite_exec -lsqlite3
./02_sqlite_exec
```

ผลลัพธ์จริง:

```
รายชื่อนักเรียนทั้งหมด:
  id = 1
  name = Somchai
  score = 85.5
  ---
  id = 2
  name = Suda
  score = 92.0
  ---
  id = 3
  name = Anan
  score = 77.25
  ---
```

`sqlite3_exec` มีข้อจำกัดสำคัญ: **มันเหมาะกับ SQL ที่ไม่มีค่าจากผู้ใช้ปนอยู่เท่านั้น** เพราะ
วิธีเดียวที่จะใส่ค่าตัวแปรเข้าไปคือการ "ต่อ string" เอง ซึ่งนำไปสู่ปัญหาใหญ่ที่สุดข้อหนึ่งของ
โลกฐานข้อมูลในหัวข้อถัดไป

---

## 105.4 SQL Injection: ช่องโหว่จริงจากการต่อ String SQL (Step 836)

**SQL Injection** คือช่องโหว่ความปลอดภัยที่เกิดขึ้นเมื่อโปรแกรมนำค่าที่รับมาจากผู้ใช้ (เช่น
จากฟอร์ม login, ช่อง search, URL parameter) ไปต่อเป็นส่วนหนึ่งของคำสั่ง SQL **โดยตรงแบบ
string concatenation** ทำให้ผู้ใช้ที่ประสงค์ร้ายสามารถ "แทรก" (inject) คำสั่ง SQL ของตัวเอง
เข้าไปปนกับคำสั่งเดิมได้ ทั้งที่ตั้งใจจะป้อนแค่ "ข้อมูล" เท่านั้น

ลองดูโค้ดที่ **อันตรายและห้ามเขียนแบบนี้ในโปรเจกต์จริงเด็ดขาด**:

```cpp
// 03_sql_injection_vuln.cpp - ตัวอย่าง "ห้ามทำ" การต่อ string SQL ตรงๆ
// คอมไพล์และรันเพื่อดูช่องโหว่จริง (ใช้เพื่อการศึกษาเท่านั้น)
#include <sqlite3.h>
#include <iostream>
#include <string>

static int print_row_callback(void*, int argc, char** argv, char** col_names) {
    for (int i = 0; i < argc; ++i) {
        std::cout << "  " << col_names[i] << " = " << (argv[i] ? argv[i] : "NULL") << '\n';
    }
    std::cout << "  ---\n";
    return 0;
}

// ฟังก์ชันนี้ "อันตราย" เพราะรับ username จากผู้ใช้มาต่อ string ตรงๆ
static void login_vulnerable(sqlite3* db, const std::string& username) {
    std::string sql =
        "SELECT id, name FROM students WHERE name = '" + username + "';";

    std::cout << "SQL ที่ถูกรันจริง: " << sql << '\n';

    char* err_msg = nullptr;
    int rc = sqlite3_exec(db, sql.c_str(), print_row_callback, nullptr, &err_msg);
    if (rc != SQLITE_OK) {
        std::cerr << "SQL error: " << err_msg << '\n';
        sqlite3_free(err_msg);
    }
}

int main(void) {
    sqlite3* db = nullptr;
    sqlite3_open("school.db", &db);

    std::cout << "== ค้นหาปกติ (username = Somchai) ==\n";
    login_vulnerable(db, "Somchai");

    std::cout << "\n== ผู้ใช้ประสงค์ร้ายป้อน: ' OR '1'='1 ==\n";
    login_vulnerable(db, "' OR '1'='1");

    std::cout << "\n== ผู้ใช้ประสงค์ร้ายลบตารางทิ้งทั้งตาราง ==\n";
    login_vulnerable(db, "x'; DROP TABLE students; --");

    std::cout << "\n== ลองค้นหา Somchai อีกครั้งหลังโดนโจมตี ==\n";
    login_vulnerable(db, "Somchai");

    sqlite3_close(db);
    return 0;
}
```

```bash
g++ -Wall -Wextra -std=c++17 03_sql_injection_vuln.cpp -o 03_sql_injection_vuln -lsqlite3
./03_sql_injection_vuln
```

ผลลัพธ์จริงที่ได้ (น่าตกใจมาก):

```
== ค้นหาปกติ (username = Somchai) ==
SQL ที่ถูกรันจริง: SELECT id, name FROM students WHERE name = 'Somchai';
  id = 1
  name = Somchai
  ---

== ผู้ใช้ประสงค์ร้ายป้อน: ' OR '1'='1 ==
SQL ที่ถูกรันจริง: SELECT id, name FROM students WHERE name = '' OR '1'='1';
  id = 1
  name = Somchai
  ---
  id = 2
  name = Suda
  ---
  id = 3
  name = Anan
  ---

== ผู้ใช้ประสงค์ร้ายลบตารางทิ้งทั้งตาราง ==
SQL ที่ถูกรันจริง: SELECT id, name FROM students WHERE name = 'x'; DROP TABLE students; --';

== ลองค้นหา Somchai อีกครั้งหลังโดนโจมตี ==
SQL ที่ถูกรันจริง: SELECT id, name FROM students WHERE name = 'Somchai';
SQL error: no such table: students
```

### วิเคราะห์ว่าเกิดอะไรขึ้น

1. **กรณีที่ 1 (ปกติ)**: `username = "Somchai"` ถูกต่อเข้าไปในเครื่องหมายคำพูดเดี่ยวอย่างที่
   ตั้งใจ ได้ SQL ที่ถูกต้อง
2. **กรณีที่ 2 (`' OR '1'='1`)**: เมื่อค่าที่ป้อนมี `'` (single quote) ปนอยู่ มันจะ "ปิด" string
   literal ของ SQL ก่อนกำหนด ทำให้ `OR '1'='1` กลายเป็นส่วนหนึ่งของเงื่อนไข `WHERE` จริงๆ และ
   เพราะ `'1'='1'` เป็นจริงเสมอ query จึงคืน **ทุกแถวในตาราง** ออกมาหมด ทั้งที่ควรจะเห็นแค่
   ข้อมูลของ user คนเดียว — นี่คือรูปแบบคลาสสิกของการ **bypass การกรองข้อมูล/สิทธิ์การเข้าถึง**
3. **กรณีที่ 3 (`x'; DROP TABLE students; --`)**: ยิ่งอันตรายกว่านั้นคือ SQLite (เหมือน
   PostgreSQL) อนุญาตให้รันได้ **หลาย statement ในครั้งเดียว** โดยคั่นด้วย `;` เมื่อค่าที่ป้อน
   มี `;` ปนอยู่ ผู้โจมตีสามารถ "แถม" คำสั่งที่สองเข้ามาได้เลย ในที่นี้คือ `DROP TABLE students`
   ซึ่งลบตารางทิ้งไปจริงๆ ตามที่เห็นใน error `no such table: students` ตอนค้นหารอบถัดไป

> **นี่ไม่ใช่ตัวอย่างสมมติ** — SQL Injection เป็นช่องโหว่อันดับต้นๆ ใน OWASP Top 10 มาหลายปี
> ติดต่อกัน และเคยเป็นสาเหตุของการรั่วไหลข้อมูลระดับองค์กรขนาดใหญ่จำนวนมากในโลกจริง สาเหตุ
> รากฐานเสมอคือสิ่งที่โค้ดข้างบนทำ: **เอาข้อมูล (data) ไปปนกับคำสั่ง (code) โดยไม่แยกกัน**

---

## 105.5 ป้องกัน SQL Injection ด้วย Prepared Statement (Step 837)

วิธีป้องกันที่ถูกต้องและเป็นมาตรฐานอุตสาหกรรมคือ **Prepared Statement** (บางที่เรียก
**Parameterized Query**) หลักการคือ:

1. ส่ง SQL ที่มี **placeholder** (`?` ใน SQLite) แทนตำแหน่งของค่าที่จะใส่ทีหลัง ไปให้ฐานข้อมูล
   "เตรียม" (compile) ไว้ล่วงหน้า ณ จุดนี้ฐานข้อมูลรู้โครงสร้างคำสั่งครบถ้วนแล้ว **โดยยังไม่มี
   ข้อมูลผู้ใช้ปนอยู่เลย**
2. จากนั้นค่อย **bind** ค่าจริงเข้ากับแต่ละ placeholder แยกต่างหาก — ค่าที่ bind เข้าไปจะถูก
   ฐานข้อมูลปฏิบัติเป็น **ข้อมูลดิบเสมอ** ไม่ว่าข้างในจะมีอักขระ `'`, `;`, หรือคำสั่ง SQL ปนอยู่
   ก็ตาม มันจะไม่มีทางถูกตีความเป็นส่วนหนึ่งของโครงสร้างคำสั่งได้อีกต่อไป

นี่คือเหตุผลเชิงเทคนิคที่แท้จริงว่าทำไม prepared statement ถึง "ปิดช่องโหว่ได้จริง" ไม่ใช่แค่
กฎที่ต้องท่องจำ:

```cpp
// 04_prepared_statement_safe.cpp - ป้องกัน SQL Injection ด้วย Prepared Statement
#include <sqlite3.h>
#include <iostream>
#include <string>

static void login_safe(sqlite3* db, const std::string& username) {
    const char* sql = "SELECT id, name FROM students WHERE name = ?;";
    sqlite3_stmt* stmt = nullptr;

    // 1) เตรียมคำสั่ง SQL ล่วงหน้า โดยใช้ ? เป็น placeholder แทนค่าจริง
    int rc = sqlite3_prepare_v2(db, sql, -1, &stmt, nullptr);
    if (rc != SQLITE_OK) {
        std::cerr << "prepare ล้มเหลว: " << sqlite3_errmsg(db) << '\n';
        return;
    }

    // 2) bind ค่าเข้ากับ placeholder ตัวที่ 1 (นับจาก 1) — SQLite เก็บค่านี้แยกจาก SQL text
    //    เสมอ จึงไม่มีทางที่ค่านี้จะถูกตีความเป็นส่วนหนึ่งของคำสั่ง SQL ได้อีกต่อไป
    sqlite3_bind_text(stmt, 1, username.c_str(), -1, SQLITE_TRANSIENT);

    std::cout << "ค้นหาด้วย username = \"" << username << "\" (bind แบบปลอดภัย)\n";

    // 3) step ทีละแถวจนกว่าจะหมดผลลัพธ์
    while ((rc = sqlite3_step(stmt)) == SQLITE_ROW) {
        int id = sqlite3_column_int(stmt, 0);
        const unsigned char* name = sqlite3_column_text(stmt, 1);
        std::cout << "  id = " << id << ", name = " << name << '\n';
    }
    if (rc != SQLITE_DONE) {
        std::cerr << "step ล้มเหลว: " << sqlite3_errmsg(db) << '\n';
    }

    // 4) finalize ต้องเรียกเสมอเพื่อคืนทรัพยากรของ statement
    sqlite3_finalize(stmt);
    std::cout << "  (ไม่มีผลลัพธ์เพิ่มเติม)\n";
}

int main(void) {
    sqlite3* db = nullptr;
    sqlite3_open("school.db", &db);

    login_safe(db, "Somchai");

    std::cout << "\nลองโจมตีแบบเดียวกับตัวอย่างที่แล้ว:\n";
    login_safe(db, "' OR '1'='1");
    login_safe(db, "x'; DROP TABLE students; --");

    std::cout << "\nตารางยังอยู่ครบ ลองค้นหา Somchai อีกครั้ง:\n";
    login_safe(db, "Somchai");

    sqlite3_close(db);
    return 0;
}
```

```bash
g++ -Wall -Wextra -std=c++17 04_prepared_statement_safe.cpp -o 04_prepared_statement_safe -lsqlite3
./04_prepared_statement_safe
```

ผลลัพธ์จริง:

```
ค้นหาด้วย username = "Somchai" (bind แบบปลอดภัย)
  id = 1, name = Somchai
  (ไม่มีผลลัพธ์เพิ่มเติม)

ลองโจมตีแบบเดียวกับตัวอย่างที่แล้ว:
ค้นหาด้วย username = "' OR '1'='1" (bind แบบปลอดภัย)
  (ไม่มีผลลัพธ์เพิ่มเติม)
ค้นหาด้วย username = "x'; DROP TABLE students; --" (bind แบบปลอดภัย)
  (ไม่มีผลลัพธ์เพิ่มเติม)

ตารางยังอยู่ครบ ลองค้นหา Somchai อีกครั้ง:
ค้นหาด้วย username = "Somchai" (bind แบบปลอดภัย)
  id = 1, name = Somchai
  (ไม่มีผลลัพธ์เพิ่มเติม)
```

สังเกตว่าทั้งสอง payload ที่เคยเป็นอันตรายในหัวข้อก่อนหน้า ตอนนี้ถูกปฏิบัติเป็น **แค่ข้อความ
ธรรมดา** ที่ไม่มีนักเรียนคนไหนชื่อแบบนั้น จึงไม่พบผลลัพธ์ใดๆ เลย — ตารางไม่ถูกลบ ข้อมูลไม่รั่วไหล

### ฟังก์ชันสำคัญของ Prepared Statement API

| ฟังก์ชัน | หน้าที่ |
|---|---|
| `sqlite3_prepare_v2(db, sql, -1, &stmt, nullptr)` | คอมไพล์ SQL text เป็น `sqlite3_stmt*` พร้อมใช้งาน |
| `sqlite3_bind_text/int/double(stmt, index, value, ...)` | ใส่ค่าจริงลงใน placeholder ตำแหน่งที่ `index` (เริ่มนับจาก 1) |
| `sqlite3_step(stmt)` | รันคำสั่งหนึ่งขั้น — คืน `SQLITE_ROW` ถ้ามีแถวผลลัพธ์, `SQLITE_DONE` ถ้าจบ |
| `sqlite3_column_int/text/double(stmt, col_index)` | อ่านค่าจากคอลัมน์ที่ `col_index` (เริ่มนับจาก 0) ของแถวปัจจุบัน |
| `sqlite3_finalize(stmt)` | คืนทรัพยากรของ statement — ต้องเรียกเสมอ |
| `sqlite3_reset(stmt)` | รีเซ็ต statement เพื่อ bind ค่าใหม่แล้วรันซ้ำ โดยไม่ต้อง prepare ใหม่ |

> **กฎทองของ Part นี้**: ห้ามต่อ string SQL จากค่าที่มาจากผู้ใช้เด็ดขาด ไม่ว่าจะผ่านทาง
> `sqlite3_exec` หรือแม้แต่ผ่าน `std::string` ที่ดูปลอดภัย — ให้ใช้ prepared statement กับ
> `?` เสมอเมื่อค่าที่ใส่มาจากภายนอกโปรแกรม

---

## 105.6 เขียน RAII Wrapper Class ห่อ sqlite3* (ทบทวนจาก Part 68) (Step 838)

โค้ดในหัวข้อก่อนหน้าใช้งานได้ถูกต้อง แต่มีปัญหาเชิง engineering: ทุกจุดที่เปิด `sqlite3*` หรือ
`sqlite3_stmt*` ต้องคอยเรียก `sqlite3_close`/`sqlite3_finalize` เองด้วยมือ ถ้ามี `return` แทรก
กลางทาง หรือมี exception โยนออกมาก่อนถึงบรรทัด cleanup ทรัพยากรจะรั่วไหลทันที

Part 68 สอนหลักการ **RAII (Resource Acquisition Is Initialization)** ไว้ว่า: ผูกการ "เปิด"
ทรัพยากรไว้กับ constructor และผูกการ "ปิด" ไว้กับ destructor — เมื่อ object หมด scope ไม่ว่า
ด้วยวิธีใด (จบปกติ, `return` กลางทาง, หรือ exception) destructor จะถูกเรียกให้อัตโนมัติเสมอ
เราจะประยุกต์หลักการเดียวกันนี้กับ `sqlite3*` และ `sqlite3_stmt*`

```cpp
// 05_sqlite_raii.hpp - RAII wrapper รอบ sqlite3* (ทบทวนแนวคิดจาก Part 68)
#pragma once
#include <sqlite3.h>
#include <stdexcept>
#include <string>
#include <utility>

// ---------- SqliteStatement: ห่อ sqlite3_stmt* แบบ RAII ----------
class SqliteStatement {
public:
    SqliteStatement(sqlite3* db, const std::string& sql) {
        if (sqlite3_prepare_v2(db, sql.c_str(), -1, &stmt_, nullptr) != SQLITE_OK) {
            throw std::runtime_error("prepare ล้มเหลว: " + std::string(sqlite3_errmsg(db)));
        }
    }

    ~SqliteStatement() {
        if (stmt_) sqlite3_finalize(stmt_); // คืนทรัพยากรอัตโนมัติเสมอ ไม่ว่าจะจบแบบไหน
    }

    // ห้าม copy เพราะ sqlite3_stmt* เป็น resource เดี่ยว ห้ามมีเจ้าของซ้ำ
    SqliteStatement(const SqliteStatement&) = delete;
    SqliteStatement& operator=(const SqliteStatement&) = delete;

    SqliteStatement(SqliteStatement&& other) noexcept : stmt_(other.stmt_) {
        other.stmt_ = nullptr;
    }
    SqliteStatement& operator=(SqliteStatement&& other) noexcept {
        if (this != &other) {
            if (stmt_) sqlite3_finalize(stmt_);
            stmt_ = other.stmt_;
            other.stmt_ = nullptr;
        }
        return *this;
    }

    void bind_text(int index, const std::string& value) {
        sqlite3_bind_text(stmt_, index, value.c_str(), -1, SQLITE_TRANSIENT);
    }
    void bind_double(int index, double value) {
        sqlite3_bind_double(stmt_, index, value);
    }
    void bind_int(int index, int value) {
        sqlite3_bind_int(stmt_, index, value);
    }

    // step() คืนค่า true ถ้ายังมีแถวอยู่ (SQLITE_ROW), false เมื่อจบ (SQLITE_DONE)
    bool step() {
        int rc = sqlite3_step(stmt_);
        if (rc == SQLITE_ROW) return true;
        if (rc == SQLITE_DONE) return false;
        throw std::runtime_error("step ล้มเหลว: " + std::string(sqlite3_errmsg(sqlite3_db_handle(stmt_))));
    }

    int column_int(int col) const { return sqlite3_column_int(stmt_, col); }
    double column_double(int col) const { return sqlite3_column_double(stmt_, col); }
    std::string column_text(int col) const {
        const unsigned char* text = sqlite3_column_text(stmt_, col);
        return text ? reinterpret_cast<const char*>(text) : "";
    }

private:
    sqlite3_stmt* stmt_ = nullptr;
};

// ---------- SqliteDB: ห่อ sqlite3* แบบ RAII ----------
class SqliteDB {
public:
    explicit SqliteDB(const std::string& path) {
        if (sqlite3_open(path.c_str(), &db_) != SQLITE_OK) {
            std::string msg = sqlite3_errmsg(db_);
            sqlite3_close(db_);
            throw std::runtime_error("เปิดฐานข้อมูลไม่สำเร็จ: " + msg);
        }
        // เปิด foreign key constraint (SQLite ปิดไว้เป็นค่า default)
        sqlite3_exec(db_, "PRAGMA foreign_keys = ON;", nullptr, nullptr, nullptr);
    }

    ~SqliteDB() {
        if (db_) sqlite3_close(db_); // ปิด connection อัตโนมัติเมื่อ SqliteDB หมดอายุ (Rule of Zero ฝั่งผู้ใช้)
    }

    SqliteDB(const SqliteDB&) = delete;
    SqliteDB& operator=(const SqliteDB&) = delete;
    SqliteDB(SqliteDB&&) = delete;
    SqliteDB& operator=(SqliteDB&&) = delete;

    void exec(const std::string& sql) {
        char* err_msg = nullptr;
        if (sqlite3_exec(db_, sql.c_str(), nullptr, nullptr, &err_msg) != SQLITE_OK) {
            std::string msg = err_msg ? err_msg : "unknown error";
            sqlite3_free(err_msg);
            throw std::runtime_error("SQL error: " + msg);
        }
    }

    SqliteStatement prepare(const std::string& sql) {
        return SqliteStatement(db_, sql);
    }

    sqlite3_int64 last_insert_rowid() const {
        return sqlite3_last_insert_rowid(db_);
    }

    int changes() const {
        return sqlite3_changes(db_);
    }

private:
    sqlite3* db_ = nullptr;
};
```

จุดสำคัญที่ทบทวนและขยายจาก Part 68:

- **`SqliteDB`** ทำตาม **Rule of Zero ที่ผู้ใช้ class เห็น**: ผู้เรียกใช้ไม่ต้องเรียก
  `sqlite3_close` เองเลยสักครั้ง — แค่ให้ `SqliteDB` หมด scope ก็พอ เราปิดการ copy/move ทั้งคู่
  ไว้เพราะการมี `sqlite3*` สองตัวชี้ไปที่ connection เดียวกันแล้ว close ซ้ำจะเป็นอันตราย
  (Double-Close คล้ายกับ Double-Free ที่เคยเรียนใน Module B)
- **`SqliteStatement`** ทำตาม **Rule of Five** เต็มรูปแบบเหมือน `FileGuard` ใน Part 68 เพราะ
  ถือ raw resource (`sqlite3_stmt*`) เองโดยตรง จึงต้องเขียน move constructor/assignment ให้
  ถูกต้อง เพื่อให้ `db.prepare(sql)` คืนค่าเป็น object กลับมาได้โดยไม่ copy
- ทั้งสอง class **โยน exception** (`std::runtime_error`) แทนการคืน error code แบบ C เดิม ทำให้
  โค้ดฝั่งผู้ใช้เขียนด้วย `try`/`catch` ครั้งเดียวครอบคลุมได้ทั้งฟังก์ชัน แทนที่จะต้องเช็ค
  `if (rc != SQLITE_OK)` ทุกบรรทัดเหมือนใน C API ดิบ

---

## 105.7 CRUD ครบวงจรด้วย Wrapper Class ของตัวเอง (Step 839)

ตอนนี้เรามีเครื่องมือครบแล้ว มาลองเขียนโปรแกรมที่ทำ **CRUD (Create, Read, Update, Delete)**
ครบทั้ง 4 การกระทำผ่าน `SqliteDB`/`SqliteStatement`:

```cpp
// 06_crud_full.cpp - CRUD ครบวงจร (Create table, Insert, Select, Update, Delete)
// โดยใช้ SqliteDB / SqliteStatement RAII wrapper จาก 05_sqlite_raii.hpp
#include "05_sqlite_raii.hpp"
#include <iostream>
#include <iomanip>

static void print_all_students(SqliteDB& db) {
    auto stmt = db.prepare("SELECT id, name, score FROM students ORDER BY id;");
    std::cout << std::left << std::setw(4) << "id" << std::setw(12) << "name" << "score\n";
    std::cout << "----------------------\n";
    while (stmt.step()) {
        std::cout << std::left << std::setw(4) << stmt.column_int(0)
                   << std::setw(12) << stmt.column_text(1)
                   << stmt.column_double(2) << '\n';
    }
}

int main(void) {
    try {
        SqliteDB db("crud_demo.db");

        // ----- CREATE TABLE -----
        db.exec("DROP TABLE IF EXISTS students;");
        db.exec(
            "CREATE TABLE students ("
            "  id INTEGER PRIMARY KEY AUTOINCREMENT,"
            "  name TEXT NOT NULL,"
            "  score REAL NOT NULL"
            ");"
        );
        std::cout << "[CREATE] สร้างตาราง students สำเร็จ\n\n";

        // ----- INSERT (ผ่าน prepared statement ป้องกัน injection) -----
        {
            auto stmt = db.prepare("INSERT INTO students (name, score) VALUES (?, ?);");
            stmt.bind_text(1, "Somchai");
            stmt.bind_double(2, 85.5);
            stmt.step();
            std::cout << "[INSERT] เพิ่ม Somchai, id ที่ได้ = " << db.last_insert_rowid() << '\n';
        }
        {
            auto stmt = db.prepare("INSERT INTO students (name, score) VALUES (?, ?);");
            stmt.bind_text(1, "Suda");
            stmt.bind_double(2, 92.0);
            stmt.step();
            std::cout << "[INSERT] เพิ่ม Suda, id ที่ได้ = " << db.last_insert_rowid() << '\n';
        }
        {
            auto stmt = db.prepare("INSERT INTO students (name, score) VALUES (?, ?);");
            stmt.bind_text(1, "Anan");
            stmt.bind_double(2, 77.25);
            stmt.step();
            std::cout << "[INSERT] เพิ่ม Anan, id ที่ได้ = " << db.last_insert_rowid() << "\n\n";
        }

        // ----- SELECT -----
        std::cout << "[SELECT] รายชื่อทั้งหมดหลัง insert:\n";
        print_all_students(db);
        std::cout << '\n';

        // ----- UPDATE -----
        {
            auto stmt = db.prepare("UPDATE students SET score = ? WHERE name = ?;");
            stmt.bind_double(1, 88.0);
            stmt.bind_text(2, "Somchai");
            stmt.step();
            std::cout << "[UPDATE] แก้คะแนน Somchai เป็น 88.0 (แถวที่เปลี่ยน = " << db.changes() << ")\n\n";
        }
        std::cout << "[SELECT] รายชื่อทั้งหมดหลัง update:\n";
        print_all_students(db);
        std::cout << '\n';

        // ----- DELETE -----
        {
            auto stmt = db.prepare("DELETE FROM students WHERE name = ?;");
            stmt.bind_text(1, "Anan");
            stmt.step();
            std::cout << "[DELETE] ลบ Anan (แถวที่ถูกลบ = " << db.changes() << ")\n\n";
        }
        std::cout << "[SELECT] รายชื่อทั้งหมดหลัง delete:\n";
        print_all_students(db);

    } catch (const std::exception& e) {
        std::cerr << "เกิดข้อผิดพลาด: " << e.what() << '\n';
        return 1;
    }

    return 0;
}
```

```bash
g++ -Wall -Wextra -std=c++17 06_crud_full.cpp -o 06_crud_full -lsqlite3
./06_crud_full
```

ผลลัพธ์จริง:

```
[CREATE] สร้างตาราง students สำเร็จ

[INSERT] เพิ่ม Somchai, id ที่ได้ = 1
[INSERT] เพิ่ม Suda, id ที่ได้ = 2
[INSERT] เพิ่ม Anan, id ที่ได้ = 3

[SELECT] รายชื่อทั้งหมดหลัง insert:
id  name        score
----------------------
1   Somchai     85.5
2   Suda        92
3   Anan        77.25

[UPDATE] แก้คะแนน Somchai เป็น 88.0 (แถวที่เปลี่ยน = 1)

[SELECT] รายชื่อทั้งหมดหลัง update:
id  name        score
----------------------
1   Somchai     88
2   Suda        92
3   Anan        77.25

[DELETE] ลบ Anan (แถวที่ถูกลบ = 1)

[SELECT] รายชื่อทั้งหมดหลัง delete:
id  name        score
----------------------
1   Somchai     88
2   Suda        92
```

สังเกตว่าโค้ดในหัวข้อนี้ **ไม่มีการเรียก `sqlite3_close`, `sqlite3_finalize` หรือตรวจสอบ
`if (rc != SQLITE_OK)` แม้แต่บรรทัดเดียว** — ทั้งหมดถูกซ่อนอยู่หลัง RAII wrapper และ exception
ที่เขียนไว้ในหัวข้อก่อนหน้า นี่คือประโยชน์ที่จับต้องได้ของการลงทุนเขียน wrapper ที่ดีตั้งแต่ต้น

---

## 105.8 Transaction ใน SQLite: ความเร็วและความถูกต้องของข้อมูล (Step 840)

เมื่อต้อง insert ข้อมูลจำนวนมาก หรือต้องการให้หลายคำสั่งสำเร็จ **พร้อมกันทั้งหมดหรือไม่สำเร็จ
เลยสักคำสั่ง** (all-or-nothing) เราต้องใช้ **Transaction** ผ่านคำสั่ง `BEGIN` / `COMMIT` /
`ROLLBACK`

โดย default แล้ว SQLite เปิด **auto-commit mode**: ทุกคำสั่งที่รันจะถูกเขียนลงดิสก์ (fsync)
ทันทีทีละคำสั่ง ซึ่งช้ามากเมื่อต้อง insert หลายพันแถวติดกัน เพราะฐานข้อมูลต้อง "ล็อก + เขียน +
ปลดล็อก" ไฟล์ซ้ำทุกครั้ง การครอบด้วย `BEGIN`/`COMMIT` จะรวมทุกคำสั่งข้างในเป็นการเขียนดิสก์
เพียงครั้งเดียวตอน `COMMIT`

```cpp
// 07_transaction.cpp - BEGIN/COMMIT/ROLLBACK ใน SQLite เพื่อความเร็วและความถูกต้อง
#include "05_sqlite_raii.hpp"
#include <iostream>
#include <chrono>

static void insert_many(SqliteDB& db, int count, bool use_transaction) {
    auto start = std::chrono::steady_clock::now();

    if (use_transaction) db.exec("BEGIN TRANSACTION;");

    for (int i = 0; i < count; ++i) {
        auto stmt = db.prepare("INSERT INTO logs (message) VALUES (?);");
        stmt.bind_text(1, "log entry " + std::to_string(i));
        stmt.step();
    }

    if (use_transaction) db.exec("COMMIT;");

    auto end = std::chrono::steady_clock::now();
    std::chrono::duration<double, std::milli> ms = end - start;
    std::cout << (use_transaction ? "มี transaction: " : "ไม่มี transaction: ")
               << count << " inserts ใช้เวลา " << ms.count() << " ms\n";
}

int main(void) {
    {
        SqliteDB db("tx_demo.db");
        db.exec("DROP TABLE IF EXISTS logs;");
        db.exec("CREATE TABLE logs (id INTEGER PRIMARY KEY AUTOINCREMENT, message TEXT);");
        insert_many(db, 1000, false);
    }
    {
        SqliteDB db("tx_demo.db");
        db.exec("DROP TABLE IF EXISTS logs;");
        db.exec("CREATE TABLE logs (id INTEGER PRIMARY KEY AUTOINCREMENT, message TEXT);");
        insert_many(db, 1000, true);
    }

    // สาธิต ROLLBACK: เริ่ม transaction แล้วยกเลิกกลางทาง
    SqliteDB db("tx_demo.db");
    db.exec("BEGIN TRANSACTION;");
    db.exec("INSERT INTO logs (message) VALUES ('will be rolled back');");
    std::cout << "\n[ก่อน ROLLBACK] จำนวนแถวที่เปลี่ยนใน transaction นี้ = " << db.changes() << '\n';
    db.exec("ROLLBACK;");

    auto stmt = db.prepare("SELECT COUNT(*) FROM logs WHERE message = 'will be rolled back';");
    stmt.step();
    std::cout << "[หลัง ROLLBACK] จำนวนแถวที่พบข้อความนี้จริงในตาราง = " << stmt.column_int(0) << '\n';

    return 0;
}
```

```bash
g++ -Wall -Wextra -std=c++17 -O2 07_transaction.cpp -o 07_transaction -lsqlite3
./07_transaction
```

ผลลัพธ์จริง (ตัวเลขเวลาจะแตกต่างกันไปตามเครื่อง แต่สัดส่วนความต่างจะยังมหาศาลเสมอ):

```
ไม่มี transaction: 1000 inserts ใช้เวลา 891.474 ms
มี transaction: 1000 inserts ใช้เวลา 4.08375 ms

[ก่อน ROLLBACK] จำนวนแถวที่เปลี่ยนใน transaction นี้ = 1
[หลัง ROLLBACK] จำนวนแถวที่พบข้อความนี้จริงในตาราง = 0
```

การครอบด้วย transaction ทำให้ insert เร็วขึ้นกว่า **200 เท่า** ในตัวอย่างนี้ และ `ROLLBACK`
พิสูจน์ว่าคำสั่งที่อยู่ใน transaction ที่ยังไม่ `COMMIT` สามารถ "ยกเลิกทั้งหมด" ได้จริง —
แถวที่ insert ไปแล้วหายไปราวกับไม่เคยเกิดขึ้น

### หลักการเลือกใช้ transaction

| สถานการณ์ | ควรใช้ transaction หรือไม่ |
|---|---|
| Insert/Update ทีละแถวเดี่ยวๆ ไม่บ่อย | ไม่จำเป็น (auto-commit ก็เพียงพอ) |
| Insert ข้อมูลจำนวนมากรวดเดียว (bulk insert) | ควรใช้เสมอ เพื่อความเร็ว |
| หลายคำสั่งที่ต้องสำเร็จร่วมกันเป็นหน่วยเดียว (เช่น โอนเงินระหว่าง 2 บัญชี: หักบัญชี A แล้วเติมบัญชี B) | ต้องใช้เสมอ เพื่อความถูกต้องของข้อมูล (ป้องกันเงินหายระหว่างทางถ้า crash กลางทาง) |

---

## ข้อผิดพลาดที่พบบ่อย (Common Pitfalls)

1. **ต่อ string SQL จากค่าที่มาจากผู้ใช้** — เป็นสาเหตุของ SQL Injection ตามที่สาธิตในหัวข้อ
   105.4 ห้ามใช้ `"... WHERE name = '" + user_input + "'"` เด็ดขาด ให้ใช้ prepared statement
   กับ `?` เสมอ
2. **ลืม `sqlite3_finalize` หรือ `sqlite3_close`** — ถ้าเขียน C API ดิบโดยไม่มี RAII wrapper
   การลืม cleanup ในบาง code path (เช่น `return` กลางทางตอนเจอ error) จะทำให้ resource รั่วไหล
   สะสมจนถึง limit ของระบบ วิธีป้องกันคือเขียน wrapper class อย่างที่ทำในหัวข้อ 105.6 เสมอ
3. **ลืมตรวจสอบ return value ของ `sqlite3_open`** — `sqlite3_open` จะคืน handle ที่ใช้งานได้
   (ไม่ใช่ `nullptr`) แม้เปิดไม่สำเร็จก็ตาม (เพื่อให้เรียก `sqlite3_errmsg` ได้) การไม่เช็ค
   return code จะทำให้โค้ดพยายามใช้ฐานข้อมูลที่เปิดไม่สำเร็จต่อไปโดยไม่รู้ตัว
4. **ใช้ `sqlite3_exec` แทน prepared statement เมื่อมีค่าจากผู้ใช้ปนอยู่** — แม้จะดูสะดวกกว่า
   แต่ทันทีที่ต้องใส่ตัวแปรลงใน SQL ต้องเปลี่ยนไปใช้ `sqlite3_prepare_v2` + `sqlite3_bind_*`
   เท่านั้น
5. **Insert ทีละแถวจำนวนมากโดยไม่ครอบ transaction** — จะช้ากว่าที่ควรมาก (ตามที่วัดได้ในหัวข้อ
   105.8) โดยเฉพาะเมื่อต้อง seed ข้อมูลทดสอบหรือ import ไฟล์ CSV ขนาดใหญ่
6. **ลืมว่า `SQLITE_TRANSIENT` vs `SQLITE_STATIC` สำคัญตอน bind text** — การ bind
   `std::string` ต้องใช้ `SQLITE_TRANSIENT` เพื่อบอกให้ SQLite **copy ข้อมูลเก็บไว้เอง**
   เพราะ `.c_str()` ของ `std::string` ชั่วคราวอาจถูกทำลายไปก่อนที่ `sqlite3_step` จะถูกเรียก
   ถ้าใช้ `SQLITE_STATIC` ผิดที่จะเกิด Undefined Behavior จากการอ่าน memory ที่ถูกคืนไปแล้ว
7. **คิดว่า SQLite รองรับการเขียนพร้อมกันจากหลาย process/thread ได้ดีเหมือน PostgreSQL** —
   SQLite ล็อกทั้งไฟล์เวลาเขียน (ในโหมด rollback journal ปกติ) ถ้าต้องการระบบที่มีผู้ใช้เขียน
   พร้อมกันจำนวนมาก ควรย้ายไปใช้ PostgreSQL/MySQL ตามที่จะเรียนใน Part 106

---

## แบบฝึกหัดท้ายบท

1. เขียนโปรแกรมสร้างตาราง `books` (`id`, `title`, `author`, `price`) แล้ว insert หนังสือ 5 เล่ม
   ด้วย prepared statement จากนั้นเขียนฟังก์ชัน `find_by_author(SqliteDB&, const std::string&)`
   ที่ใช้ prepared statement ค้นหาหนังสือทั้งหมดของนักเขียนคนหนึ่ง แล้วพิมพ์ผลลัพธ์
2. ดัดแปลง `login_vulnerable` ในหัวข้อ 105.4 ให้กลายเป็นฟังก์ชันค้นหาหนังสือด้วยชื่อ (title) แบบ
   ต่อ string ตรงๆ แล้วทดลองหา input ที่ทำให้มันคืนหนังสือทุกเล่มออกมาทั้งที่ผู้ใช้ตั้งใจค้นหา
   แค่เล่มเดียว (ห้ามใช้ `DROP TABLE` ในข้อนี้ แค่แสดงให้เห็นการ bypass เงื่อนไข `WHERE`)
3. เพิ่มเมธอด `SqliteStatement::bind_null(int index)` ให้กับ wrapper class ในหัวข้อ 105.6
   (ใช้ `sqlite3_bind_null`) แล้วทดสอบ insert แถวที่มีค่า `NULL` ในคอลัมน์ที่อนุญาตให้เป็น
   `NULL` ได้
4. เขียนฟังก์ชัน `transfer_score(SqliteDB&, const std::string& from, const std::string& to, double amount)`
   ที่ลดคะแนนของนักเรียนคนหนึ่งและเพิ่มคะแนนให้อีกคนหนึ่งภายใน **transaction เดียวกัน** — ถ้า
   นักเรียนคนใดคนหนึ่งไม่มีอยู่จริงให้ `ROLLBACK` ทั้งหมด (ห้ามให้เกิดสถานะที่คะแนนถูกลดไปแล้ว
   แต่ไม่มีใครได้รับเพิ่ม)
5. เขียนโปรแกรมวัดเวลาเปรียบเทียบการ `SELECT` ข้อมูล 100,000 แถวด้วย `sqlite3_exec` +
   callback เทียบกับการใช้ prepared statement วนลูป `sqlite3_step` เอง (ไม่ต้องมี callback)
   บันทึกผลว่าวิธีไหนเร็วกว่าและอธิบายเหตุผล
6. อ่านเอกสาร SQLite เรื่อง `PRAGMA journal_mode = WAL;` แล้วทดลองเปิดโหมดนี้กับฐานข้อมูลของ
   ข้อ 4 อธิบายด้วยคำพูดตัวเองว่า WAL (Write-Ahead Logging) ช่วยเรื่อง concurrent read/write
   ได้อย่างไรเมื่อเทียบกับโหมด rollback journal ปกติ

### แนวทางเฉลยข้อ 1

```cpp
// exercise1.cpp
#include "05_sqlite_raii.hpp"
#include <iostream>

static void find_by_author(SqliteDB& db, const std::string& author) {
    auto stmt = db.prepare("SELECT id, title, price FROM books WHERE author = ?;");
    stmt.bind_text(1, author);

    std::cout << "หนังสือของ " << author << ":\n";
    bool found = false;
    while (stmt.step()) {
        found = true;
        std::cout << "  [" << stmt.column_int(0) << "] "
                   << stmt.column_text(1) << " ราคา " << stmt.column_double(2) << " บาท\n";
    }
    if (!found) {
        std::cout << "  ไม่พบหนังสือของนักเขียนคนนี้\n";
    }
}

int main(void) {
    try {
        SqliteDB db("books.db");
        db.exec("DROP TABLE IF EXISTS books;");
        db.exec(
            "CREATE TABLE books ("
            "  id INTEGER PRIMARY KEY AUTOINCREMENT,"
            "  title TEXT NOT NULL,"
            "  author TEXT NOT NULL,"
            "  price REAL NOT NULL"
            ");"
        );

        struct BookSeed { std::string title, author; double price; };
        std::vector<BookSeed> seeds = {
            {"C++ Primer", "Lippman", 1200.0},
            {"Effective C++", "Meyers", 890.0},
            {"More Effective C++", "Meyers", 950.0},
            {"The C Programming Language", "Kernighan", 700.0},
            {"Design Patterns", "Gang of Four", 1500.0},
        };
        for (const auto& b : seeds) {
            auto stmt = db.prepare("INSERT INTO books (title, author, price) VALUES (?, ?, ?);");
            stmt.bind_text(1, b.title);
            stmt.bind_text(2, b.author);
            stmt.bind_double(3, b.price);
            stmt.step();
        }

        find_by_author(db, "Meyers");
        find_by_author(db, "Somchai"); // ไม่มีในฐานข้อมูล

    } catch (const std::exception& e) {
        std::cerr << "เกิดข้อผิดพลาด: " << e.what() << '\n';
        return 1;
    }
    return 0;
}
```

ผลลัพธ์ที่คาดหวังเมื่อคอมไพล์และรัน (`g++ -Wall -Wextra -std=c++17 exercise1.cpp -o exercise1 -lsqlite3`):

```
หนังสือของ Meyers:
  [2] Effective C++ ราคา 890 บาท
  [3] More Effective C++ ราคา 950 บาท
หนังสือของ Somchai:
  ไม่พบหนังสือของนักเขียนคนนี้
```

### แนวทางเฉลยข้อ 4

```cpp
// exercise4.cpp
#include "05_sqlite_raii.hpp"
#include <iostream>

static bool student_exists(SqliteDB& db, const std::string& name) {
    auto stmt = db.prepare("SELECT COUNT(*) FROM students WHERE name = ?;");
    stmt.bind_text(1, name);
    stmt.step();
    return stmt.column_int(0) > 0;
}

static void transfer_score(SqliteDB& db, const std::string& from,
                            const std::string& to, double amount) {
    db.exec("BEGIN TRANSACTION;");
    try {
        if (!student_exists(db, from) || !student_exists(db, to)) {
            throw std::runtime_error("ไม่พบนักเรียน \"" + from + "\" หรือ \"" + to + "\"");
        }

        {
            auto stmt = db.prepare("UPDATE students SET score = score - ? WHERE name = ?;");
            stmt.bind_double(1, amount);
            stmt.bind_text(2, from);
            stmt.step();
        }
        {
            auto stmt = db.prepare("UPDATE students SET score = score + ? WHERE name = ?;");
            stmt.bind_double(1, amount);
            stmt.bind_text(2, to);
            stmt.step();
        }

        db.exec("COMMIT;");
        std::cout << "โอนคะแนน " << amount << " จาก " << from << " ไปยัง " << to << " สำเร็จ\n";

    } catch (const std::exception& e) {
        db.exec("ROLLBACK;");
        std::cout << "ยกเลิกการโอนคะแนน: " << e.what() << '\n';
    }
}

int main(void) {
    SqliteDB db("crud_demo.db"); // ใช้ต่อจากตัวอย่าง 06_crud_full.cpp (มี Somchai, Suda)

    transfer_score(db, "Somchai", "Suda", 5.0);   // ควรสำเร็จ
    transfer_score(db, "Somchai", "NoSuchPerson", 5.0); // ควร rollback ทั้งหมด

    return 0;
}
```

ผลลัพธ์ที่คาดหวัง:

```
โอนคะแนน 5 จาก Somchai ไปยัง Suda สำเร็จ
ยกเลิกการโอนคะแนน: ไม่พบนักเรียน "Somchai" หรือ "NoSuchPerson"
```

จุดสำคัญของเฉลยนี้คือ `db.exec("BEGIN TRANSACTION;")` ครอบทั้งการตรวจสอบและการ `UPDATE` ทั้ง
สองคำสั่งไว้ด้วยกัน หากพบว่านักเรียนคนใดไม่มีอยู่จริง จะโยน exception ออกมาแล้วถูกจับด้วย
`catch` เพื่อสั่ง `ROLLBACK` ทันที รับประกันว่าจะไม่มีสถานะกลางๆ ที่คะแนนถูกหักไปแล้วแต่ไม่มี
ใครได้รับเพิ่มเกิดขึ้นเลย

---

## สรุปท้ายบท

ใน Part นี้เราได้:

- เข้าใจว่า SQLite เป็น embedded database ที่ไม่ต้องมี server แยก เหมาะกับ prototype, mobile,
  desktop app และงานขนาดเล็กถึงกลาง
- ใช้ `sqlite3` C API พื้นฐาน (`sqlite3_open`, `sqlite3_exec`) และเห็นช่องโหว่ **SQL Injection**
  ที่เกิดจากการต่อ string SQL ตรงๆ พร้อมผลลัพธ์จริงที่อันตรายถึงขั้นลบตารางทิ้งได้
- ป้องกัน SQL Injection ได้อย่างสมบูรณ์ด้วย **Prepared Statement** (`sqlite3_prepare_v2`,
  `sqlite3_bind_*`, `sqlite3_step`, `sqlite3_finalize`)
- เขียน **RAII wrapper class** (`SqliteDB`, `SqliteStatement`) ทบทวนหลักการจาก Part 68 เพื่อ
  จัดการทรัพยากรอัตโนมัติ ปลอดภัยจาก resource leak
- ทำ **CRUD ครบวงจร** และใช้ **Transaction** เพื่อเพิ่มความเร็วในการเขียนข้อมูลจำนวนมากกว่า
  200 เท่า พร้อมรับประกันความถูกต้องของข้อมูลด้วย `ROLLBACK`

ใน **Part 106** เราจะก้าวจาก embedded database ไปสู่ฐานข้อมูลระดับ **client-server** อย่าง
PostgreSQL และ MySQL ซึ่งเป็นสิ่งที่ Web Application ระดับ production ส่วนใหญ่ใช้งานจริง
พร้อมเรียนรู้แนวคิด **Connection Pooling** ที่สำคัญมากเมื่อ server ต้องรับหลาย request พร้อมกัน

**ต่อไป:** [Part 106 — เชื่อมต่อ PostgreSQL/MySQL ด้วย C++](./part-106-postgresql-mysql-cpp.md)
