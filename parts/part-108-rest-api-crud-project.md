# Part 108: โปรเจกต์ REST API Backend แบบ CRUD ครบวงจร (Step 857–864)

> Module I — Web Development ด้วย C/C++ | Part 108 จาก 125
> Step ที่ครอบคลุมใน Part นี้: Step 857–864
> Part ก่อนหน้า: [Part 107 — การจัดการ JSON ด้วย nlohmann/json](./part-107-json-nlohmann.md) | Part ถัดไป: [Part 109 — WebSocket Programming ด้วย C++](./part-109-websocket-cpp.md)

## เป้าหมายการเรียนรู้ (Learning Objectives)

หลังจบ Part นี้ ผู้เรียนจะสามารถ:

1. ออกแบบ REST API สำหรับ resource หนึ่งตัว (Task) ให้ครบทั้ง 5 endpoint ตามหลัก CRUD
   (Create, Read, Update, Delete) พร้อม HTTP method และ status code ที่ถูกต้องตามมาตรฐาน
2. ออกแบบรูปแบบ JSON response ("envelope") ที่สม่ำเสมอทั้ง success และ error case
   เพื่อให้ฝั่ง client เขียนโค้ดจัดการ response ได้ด้วย logic เดียวกันทุก endpoint
3. จัดโครงสร้างโปรเจกต์ C++ ขนาดกลางให้แยกเป็นชั้น (layer) อย่างเหมาะสม — Model, Data Access
   Layer (DAL), Routes — แทนที่จะยัดทุกอย่างไว้ใน `main.cpp` ไฟล์เดียว
4. เขียน Data Access Layer ที่ห่อหุ้ม (encapsulate) การคุยกับ SQLite ทั้งหมด โดยใช้ Prepared
   Statement เพื่อป้องกัน SQL Injection
5. เขียน Input Validation ที่ตรวจสอบทั้งชนิดข้อมูล (type) และขอบเขตค่า (range/length) ก่อน
   บันทึกลงฐานข้อมูล พร้อมคืน error message ที่อ่านเข้าใจง่าย
6. ตั้งค่า CMake สำหรับโปรเจกต์ที่มีหลายไฟล์ และเชื่อมกับ library ภายนอก (SQLite3, pthread)
7. ทดสอบ REST API ที่เขียนเองด้วย `curl` ครบทุก endpoint ทั้ง success case และ error case
   (400, 404, 409, 500) และอ่านผลลัพธ์จริงที่ได้กลับมา

---

Part นี้คือ "โปรเจกต์สรุป" (Capstone เล็ก) ของครึ่งแรกใน Module I เราจะนำทุกอย่างที่เรียนมา
ตั้งแต่ Part 99 ถึง Part 107 มาประกอบร่างเป็นระบบเดียวที่ใช้งานได้จริง:

- **Part 100–101**: ความเข้าใจเรื่อง HTTP server ระดับ socket → ตอนนี้เราใช้ **Crow** ครอบให้แล้ว
- **Part 102–103**: การใช้ Crow ทำ routing และ JSON response
- **Part 105**: การเชื่อมต่อ SQLite ด้วย C++ (sqlite3 C API)
- **Part 107**: การจัดการ JSON ด้วย nlohmann/json

สิ่งที่ Part นี้เพิ่มเข้ามาซึ่งไม่เคยพูดถึงมาก่อนคือ **การออกแบบระบบ (System Design) ระดับ
โปรเจกต์จริง** — ไม่ใช่แค่ "เขียนโค้ดให้รัน" แต่ต้อง "เขียนโค้ดให้คนอื่นอ่านต่อได้ ดูแลต่อได้
และขยายต่อได้" ซึ่งเป็นทักษะที่แยกโปรแกรมเมอร์มือใหม่ออกจากวิศวกรซอฟต์แวร์มืออาชีพอย่างชัดเจน

---

## 108.1 ภาพรวมโปรเจกต์: ออกแบบ Resource "Task" (Step 857)

โปรเจกต์นี้คือ **Task API** — backend ของแอปทำ To-Do List ง่ายๆ แต่สมบูรณ์แบบในเชิงวิศวกรรม
ทุกประการ resource เดียวที่เรามีคือ **Task** ซึ่งมีโครงสร้างข้อมูลดังนี้:

| ฟิลด์ | ชนิดข้อมูล | คำอธิบาย |
|---|---|---|
| `id` | integer | Primary key สร้างอัตโนมัติโดยฐานข้อมูล (auto-increment) |
| `title` | string | ชื่องาน (บังคับ, ความยาว 1-200 ตัวอักษร) |
| `description` | string | รายละเอียดงาน (ไม่บังคับ, default เป็น string ว่าง) |
| `done` | boolean | สถานะว่าทำเสร็จหรือยัง (default = false) |
| `created_at` | string (ISO-ish) | วันเวลาที่สร้าง Task นี้ (ฐานข้อมูลสร้างให้อัตโนมัติ) |

### ทำไมต้องเลือก Task เป็นตัวอย่าง

Task/To-Do List เป็น resource ที่ "ง่ายพอ" ที่จะไม่หลงประเด็นไปกับ business logic ที่ซับซ้อน
แต่ "ครบพอ" ที่จะครอบคลุมทุกแพทเทิร์นสำคัญของ REST API: text field, boolean field, ความสัมพันธ์
กับเวลา (timestamp), และการ validate ข้อมูลที่ผู้ใช้ป้อนเข้ามา สิ่งที่เรียนในนี้จะย้ายไปใช้กับ
resource อื่นๆ ที่ซับซ้อนกว่าได้ทันที (เช่น "User", "Order", "Product" ใน Capstone โปรเจกต์ใหญ่
ของ Part 121)

### กำหนดสัญญา (Contract) ของ API ทั้ง 5 Endpoint

การออกแบบ API ที่ดีต้องกำหนด "สัญญา" ให้ชัดเจนตั้งแต่ก่อนเขียนโค้ดสักบรรทัด — ทีม frontend
กับทีม backend สามารถทำงานคู่ขนานกันได้เลยถ้าตกลงสัญญานี้ไว้ก่อน:

| Method | Path | คำอธิบาย | Body ที่ต้องส่ง | Status สำเร็จ |
|---|---|---|---|---|
| `GET` | `/tasks` | ดึงรายการ Task ทั้งหมด | ไม่มี | 200 |
| `GET` | `/tasks/<id>` | ดึง Task ตัวเดียวตาม id | ไม่มี | 200 |
| `POST` | `/tasks` | สร้าง Task ใหม่ | `{title, description?, done?}` | 201 |
| `PUT` | `/tasks/<id>` | แก้ไข Task ที่มีอยู่ | `{title?, description?, done?}` | 200 |
| `DELETE` | `/tasks/<id>` | ลบ Task | ไม่มี | 200 |

สังเกตว่า `title` เป็นฟิลด์เดียวที่บังคับตอน `POST` (สร้างใหม่) ส่วน `PUT` (แก้ไข) อนุญาตให้ส่ง
มาแค่บางฟิลด์ได้ (partial update) — ฟิลด์ไหนไม่ส่งมาจะคงค่าเดิมไว้ นี่คือแพทเทิร์นที่ REST API
ในโลกจริงส่วนใหญ่ใช้กัน (คล้าย `PATCH` มากกว่า `PUT` แบบเคร่งครัดตามสเปก HTTP ดั้งเดิม
แต่เป็นแนวทางที่ปฏิบัติกันทั่วไปเพราะสะดวกกว่ามาก)

---

## 108.2 ออกแบบ JSON Response Envelope และ HTTP Status Code (Step 858)

ปัญหาที่พบบ่อยที่สุดของ REST API ที่เขียนแบบไม่ได้วางแผน คือแต่ละ endpoint คืนรูปแบบ JSON
ไม่เหมือนกัน บาง endpoint คืน object ตรงๆ บาง endpoint ห่อด้วย `{"result": ...}` บาง endpoint
คืน error เป็น string ธรรมดา ทำให้ฝั่ง client ต้องเขียนโค้ดจัดการเป็นกรณีๆ ไป

เราจะแก้ปัญหานี้ด้วยการกำหนด **Envelope** (ซองจดหมาย) รูปแบบเดียวที่ใช้ตลอดทั้ง API:

**กรณีสำเร็จ:**

```json
{
    "success": true,
    "data": { ... หรือ [ ... ] }
}
```

**กรณีล้มเหลว:**

```json
{
    "success": false,
    "error": {
        "code": "TASK_NOT_FOUND",
        "message": "ไม่พบ Task ที่มี id = 999"
    }
}
```

ข้อดีของแพทเทิร์นนี้:

- ฝั่ง client เช็คแค่ field `success` (boolean) ก็รู้ทันทีว่าจะไปอ่าน `data` หรือ `error`
- `error.code` เป็น string คงที่ (เช่น `"TASK_NOT_FOUND"`, `"VALIDATION_ERROR"`) ที่โปรแกรม
  เขียนเงื่อนไขเทียบได้โดยตรง ส่วน `error.message` เป็นข้อความสำหรับแสดงให้ผู้ใช้อ่าน
  (การแยก code กับ message ออกจากกันสำคัญมาก เพราะ message อาจเปลี่ยนคำได้ตลอดโดยไม่กระทบ
  โค้ดฝั่ง client ที่เช็คด้วย code)
- โครงสร้างเดียวกันนี้ต่อยอดได้ทันทีไม่ว่า resource จะเปลี่ยนเป็นอะไรก็ตาม

### HTTP Status Code ที่ใช้ในโปรเจกต์นี้

| Status Code | ความหมาย | ใช้เมื่อ |
|---|---|---|
| `200 OK` | สำเร็จ | GET, PUT, DELETE ที่สำเร็จ |
| `201 Created` | สร้างสำเร็จ | POST ที่สร้าง resource ใหม่สำเร็จ |
| `400 Bad Request` | คำขอผิดรูปแบบ | JSON parse ไม่ผ่าน หรือ validation ล้มเหลว |
| `404 Not Found` | ไม่พบ resource | GET/PUT/DELETE ด้วย id ที่ไม่มีอยู่จริง |
| `409 Conflict` | ข้อมูลขัดแย้งกับที่มีอยู่ | (ใช้ใน Part 110 ตอนสมัครสมาชิกด้วย username ซ้ำ) |
| `500 Internal Server Error` | เซิร์ฟเวอร์ผิดพลาดเอง | เกิด exception ที่ไม่คาดคิด (เช่น ฐานข้อมูลล่ม) |

> **กฎทองของ REST API**: ห้ามคืน `200 OK` พร้อม body ที่บอกว่า error เด็ดขาด — HTTP status
> code ต้องสะท้อนความจริงเสมอ เพราะเครื่องมือจำนวนมาก (load balancer, monitoring, HTTP client
> library) ตัดสินใจโดยดู status code เป็นหลัก ไม่ได้แกะ body ทุกครั้ง

### Idempotency: คุณสมบัติที่ต้องคำนึงถึงตอนเลือก HTTP Method

อีกแนวคิดสำคัญที่ต้องเข้าใจตอนออกแบบ API คือ **Idempotency** — การเรียก operation เดิมซ้ำๆ
หลายครั้งต้องได้ผลลัพธ์สุดท้ายเหมือนเดิมเสมอ (ไม่นับ response ที่คืนกลับมา) HTTP spec กำหนด
ไว้ชัดเจนว่า method ไหนควร idempotent:

| Method | Idempotent? | Safe? (ไม่เปลี่ยนแปลงข้อมูล) | เหตุผล |
|---|---|---|---|
| `GET` | ใช่ | ใช่ | อ่านอย่างเดียว เรียกกี่ครั้งก็ได้ผลเหมือนเดิม |
| `PUT` | ใช่ | ไม่ | แทนที่ทั้งก้อนด้วยค่าเดิมซ้ำๆ ผลลัพธ์สุดท้ายเหมือนกันเสมอ |
| `DELETE` | ใช่ | ไม่ | ลบซ้ำ id เดิม ครั้งแรกได้ 200 ครั้งต่อไปได้ 404 แต่สถานะสุดท้าย "ไม่มี resource นี้" เหมือนกัน |
| `POST` | **ไม่ใช่** | ไม่ | เรียกซ้ำ = สร้าง resource ใหม่ซ้ำอีกตัว (คนละ id) |
| `PATCH` | ไม่จำเป็นต้องเป็น | ไม่ | ขึ้นกับการ implement (เช่น "toggle" ในแบบฝึกหัดข้อ 6 **ไม่** idempotent เพราะเรียก 2 ครั้งจะสลับกลับไปกลับมา) |

เหตุผลที่ Task API นี้ใช้ `PUT` สำหรับแก้ไข (ไม่ใช่ `POST`) ก็เพราะ `PUT` สื่อความหมายว่า
"เรียกกี่ครั้งด้วยข้อมูลเดิมก็ได้ผลลัพธ์เดิม" ซึ่งตรงกับพฤติกรรมจริงของ endpoint นี้ ส่วน
`POST /tasks` ที่ไม่ idempotent ก็สมเหตุสมผลเพราะการเรียกซ้ำควรสร้าง Task ใหม่จริงๆ (ผู้ใช้
กดปุ่ม "เพิ่มงาน" สองครั้งด้วยข้อมูลเดียวกัน ควรได้ Task 2 รายการ ไม่ใช่รายการเดียว)

การเข้าใจ idempotency สำคัญมากในระบบ production จริง เพราะเครือข่ายไม่เสถียร 100% เสมอ — ถ้า
client ส่ง request แล้วไม่ได้รับ response กลับมา (timeout) การ retry ด้วย method ที่ idempotent
อย่าง `PUT`/`DELETE` ปลอดภัยกว่าการ retry `POST` มาก เพราะ `POST` ที่ retry อาจสร้างข้อมูลซ้ำ
โดยไม่ได้ตั้งใจ

### Richardson Maturity Model: API นี้อยู่ระดับไหน

Leonard Richardson เสนอโมเดลแบ่งระดับความเป็น "RESTful" ของ API ออกเป็น 4 ระดับ (0-3)
ใช้เป็นเกณฑ์คร่าวๆ ประเมินว่า API หนึ่งๆ ยึดหลัก REST มากแค่ไหน:

| ระดับ | ชื่อ | ลักษณะ |
|---|---|---|
| Level 0 | The Swamp of POX | ใช้ HTTP แค่เป็นท่อขนส่ง endpoint เดียว method เดียว (มักเป็น `POST` ทุกอย่าง) |
| Level 1 | Resources | แยก URL ตาม resource แล้ว (`/tasks`, `/tasks/1`) แต่ยังใช้ method เดียว |
| Level 2 | HTTP Verbs | ใช้ HTTP method (`GET`/`POST`/`PUT`/`DELETE`) และ status code ให้ตรงความหมาย |
| Level 3 | HATEOAS | response มี link บอก action ถัดไปที่ทำได้ (Hypermedia as the Engine of Application State) |

Task API ของเราอยู่ที่ **Level 2** — ใช้ resource-based URL, HTTP method ถูกต้อง, status code
ถูกต้อง ซึ่งเป็นระดับที่ API ในโลกจริงส่วนใหญ่ (รวมถึง API ของบริษัทใหญ่แทบทุกเจ้า) หยุดอยู่
Level 3 (HATEOAS เต็มรูปแบบ) แม้จะ "RESTful ที่สุด" ตามทฤษฎีดั้งเดิมของ Roy Fielding แต่ใน
ทางปฏิบัติมีความซับซ้อนสูงและมีประโยชน์จำกัดสำหรับ API ภายในองค์กรหรือ mobile app ที่ทีม
frontend/backend เป็นทีมเดียวกันอยู่แล้ว จึงไม่ใช่เป้าหมายของบทเรียนนี้

เราจะ implement แนวคิดของ Level 2 เป็นฟังก์ชันช่วยสองตัวคือ `api::ok()` และ `api::error()`
ในหัวข้อถัดไป

---

## 108.3 โครงสร้างโปรเจกต์แบบมืออาชีพ (Step 859)

จุดที่ frameworks ตัวอย่างส่วนใหญ่สอนแบบ "ยัดทุกอย่างใน `main.cpp`" นั้นใช้ได้กับตัวอย่างเล็กๆ
แต่พอโปรเจกต์โตขึ้น (มี resource หลายตัว, มี middleware, มี business logic ซับซ้อน) ไฟล์เดียว
จะกลายเป็นไฟล์หลักพันบรรทัดที่แก้ไขแล้วเสี่ยงพังทั้งระบบ เราจะแบ่งเป็นชั้น (layer) ตั้งแต่ต้น:

```
task_api/
├── CMakeLists.txt
├── include/
│   ├── task.hpp           # Model: struct Task
│   ├── db.hpp              # Data Access Layer: class Db
│   ├── response.hpp         # ฟังก์ชันช่วยสร้าง JSON envelope
│   └── task_routes.hpp      # ประกาศฟังก์ชันลงทะเบียน route
└── src/
    ├── main.cpp             # จุดเริ่มต้นโปรแกรม ประกอบทุกชิ้นเข้าด้วยกัน
    ├── db.cpp               # Implementation ของ Db (คุยกับ SQLite จริง)
    └── task_routes.cpp      # Implementation ของ route handler ทุกตัว
```

แนวคิดสำคัญของโครงสร้างนี้คือ **แต่ละไฟล์มีหน้าที่เดียว (Single Responsibility)**:

| ชั้น (Layer) | ไฟล์ | หน้าที่ | รู้จัก Crow หรือไม่ | รู้จัก SQLite หรือไม่ |
|---|---|---|---|---|
| Model | `task.hpp` | นิยามรูปร่างข้อมูล 1 แถว | ไม่รู้จัก | ไม่รู้จัก |
| Data Access Layer | `db.hpp` / `db.cpp` | อ่าน/เขียนฐานข้อมูล | ไม่รู้จัก | รู้จัก (รู้จักเดียว) |
| Response Helper | `response.hpp` | ห่อ JSON เป็น envelope | รู้จัก (ใช้ `crow::response`) | ไม่รู้จัก |
| Routes | `task_routes.hpp/cpp` | รับ request, เรียก DAL, คืน response | รู้จัก | ไม่รู้จักโดยตรง (เรียกผ่าน `Db`) |
| Entry point | `main.cpp` | ประกอบทุกชิ้นเข้าด้วยกัน, เปิด server | รู้จัก | รู้จัก (สร้าง instance) |

การแยกแบบนี้เรียกว่า **Layered Architecture** — ประโยชน์ที่จับต้องได้ทันทีคือ ถ้าวันหนึ่งจะ
เปลี่ยนจาก SQLite ไปเป็น PostgreSQL (ตามที่เรียนใน Part 106) เราแก้แค่ `db.cpp` ไฟล์เดียว
โดยที่ `task_routes.cpp` ไม่ต้องแตะเลยแม้แต่บรรทัดเดียว เพราะมันคุยกับ `Db` ผ่าน interface
(`list_all()`, `find_by_id()`, ...) ไม่ได้คุยกับ SQLite ตรงๆ

### เปรียบเทียบแนวทางจัดโครงสร้างโปรเจกต์

| แนวทาง | ลักษณะ | เหมาะกับ | ข้อเสียเมื่อโปรเจกต์โต |
|---|---|---|---|
| Flat (`main.cpp` ไฟล์เดียว) | ทุกอย่างอยู่ไฟล์เดียว | Demo, prototype เล็กๆ, การเรียนรู้ครั้งแรก | ไฟล์ยาวหลายพันบรรทัด, merge conflict บ่อย, ทดสอบยาก |
| Layered (แบบที่ใช้ในบทนี้) | แยกตามหน้าที่ (Model/DAL/Routes) | โปรเจกต์ขนาดกลาง, ทีมเล็ก-กลาง | ต้องวางแผนล่วงหน้าเล็กน้อยตอนเริ่ม |
| MVC เต็มรูปแบบ | เพิ่มชั้น Controller/View แยกจาก Model ชัดเจน | เว็บแอปที่มี server-side rendering (Part 111) | overhead เกินจำเป็นถ้าเป็น API-only ที่ไม่มี View |
| Hexagonal / Clean Architecture | แยก business logic ออกจาก framework ทั้งหมดผ่าน interface/port | ระบบใหญ่มาก มี business rule ซับซ้อน อายุใช้งานยาว | ซับซ้อนเกินไปสำหรับโปรเจกต์เล็กอย่าง Task API |

บทเรียนนี้เลือกใช้ **Layered Architecture** แบบเรียบง่าย เพราะเป็นจุดสมดุลที่ดีที่สุดสำหรับ
ขนาดของโปรเจกต์นี้ — ซับซ้อนพอที่จะสอนหลักการแยกชั้นที่ถูกต้อง แต่ไม่ซับซ้อนเกินจนทำให้ผู้เรียน
หลงประเด็นไปกับ abstraction ที่ไม่จำเป็น เมื่อไปถึง Capstone โปรเจกต์ใหญ่ใน Part 121-124
(E-Commerce, Chat Server, Distributed KV Store, Web Framework) แนวคิดเรื่อง Hexagonal
Architecture และ Dependency Injection แบบเต็มรูปแบบจะถูกอธิบายเพิ่มเติม

### CMakeLists.txt

```cmake
cmake_minimum_required(VERSION 3.16)
project(task_api CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

find_package(Threads REQUIRED)
find_package(SQLite3 REQUIRED)

add_executable(task_api
    src/main.cpp
    src/db.cpp
    src/task_routes.cpp
)

target_include_directories(task_api PRIVATE
    include
    /usr/local/include     # ที่อยู่ของ Crow (header-only library)
)

target_link_libraries(task_api PRIVATE
    Threads::Threads
    SQLite::SQLite3
)
```

`find_package(SQLite3 REQUIRED)` ใช้ module `FindSQLite3.cmake` ที่มากับ CMake ตั้งแต่เวอร์ชัน
3.14 เป็นต้นไป จะหา header และ library ของ SQLite3 ที่ติดตั้งในระบบให้อัตโนมัติ ไม่ต้องเขียน
path ตรงๆ เอง ส่วน Crow เป็น **header-only library** (ไม่มีไฟล์ `.so`/`.a` ให้ link) จึงแค่เพิ่ม
`/usr/local/include` เข้า include path ก็พอ

Build ด้วยคำสั่งมาตรฐาน:

```bash
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build -j4
```

ผลลัพธ์จริงจากการรันคำสั่งนี้บนเครื่องที่ติดตั้ง Crow ไว้ที่ `/usr/local/include/crow`:

```
-- The CXX compiler identification is GNU 13.3.0
-- Found Threads: TRUE
-- Found SQLite3: /usr/include (found version "3.45.1")
-- Configuring done
-- Generating done
-- Build files have been written to: .../task_api/build
[ 25%] Building CXX object CMakeFiles/task_api.dir/src/main.cpp.o
[ 50%] Building CXX object CMakeFiles/task_api.dir/src/db.cpp.o
[ 75%] Building CXX object CMakeFiles/task_api.dir/src/task_routes.cpp.o
[100%] Linking CXX executable task_api
[100%] Built target task_api
```

Build ผ่านสะอาด ไม่มี warning แม้แต่บรรทัดเดียว (ทุกไฟล์ในบทเรียนนี้คอมไพล์ผ่านจริงด้วย
`g++ 13.3.0` มาตรฐาน C++17)

---

## 108.4 Model Layer: struct Task (Step 860)

Model คือชั้นที่ "เบาที่สุด" ในระบบ — เป็นแค่ struct ธรรมดาที่แทนข้อมูล 1 แถว ไม่ผูกกับทั้ง
Crow และ SQLite โดยตรง เพื่อให้ทดสอบ (unit test) และใช้ซ้ำได้ง่ายที่สุด

`include/task.hpp`:

```cpp
#pragma once
#include <string>
#include <nlohmann/json.hpp>

// Model: ตัวแทนของแถวข้อมูล 1 แถวในตาราง tasks
// แยกเป็น struct ตรงๆ ไม่ผูกกับ Crow หรือ SQLite เพื่อให้ทดสอบและใช้ซ้ำได้ง่าย
struct Task {
    int id = 0;
    std::string title;
    std::string description;
    bool done = false;
    std::string created_at;   // เก็บเป็น ISO-8601 string ที่ SQLite สร้างให้อัตโนมัติ

    // แปลง Task -> nlohmann::json เพื่อส่งกลับเป็น response
    nlohmann::json to_json() const {
        return nlohmann::json{
            {"id", id},
            {"title", title},
            {"description", description},
            {"done", done},
            {"created_at", created_at}
        };
    }
};
```

สังเกตว่า `Task` **รู้วิธีแปลงตัวเองเป็น JSON** (`to_json()`) แต่ **ไม่รู้เรื่อง HTTP หรือ SQL
เลย** — นี่คือหลักการ **Separation of Concerns**: struct นี้ตอบคำถามแค่ "Task หน้าตาเป็น
อย่างไร" ไม่ตอบคำถาม "Task ถูกส่งผ่าน HTTP อย่างไร" หรือ "Task ถูกเก็บในฐานข้อมูลอย่างไร"

### Response Envelope Helper

`include/response.hpp`:

```cpp
#pragma once
#include <string>
#include <nlohmann/json.hpp>
#include "crow.h"

// ---------------------------------------------------------------------------
// รูปแบบ JSON response "envelope" ที่สม่ำเสมอตลอดทั้ง API
//
// กรณีสำเร็จ:
//   { "success": true, "data": <อะไรก็ได้> }
//
// กรณีล้มเหลว:
//   { "success": false, "error": { "code": "TASK_NOT_FOUND", "message": "..." } }
//
// การบังคับใช้รูปแบบเดียวกันทุก endpoint ทำให้ฝั่ง client (เว็บ/มือถือ) เขียนโค้ด
// จัดการ response ได้ด้วย logic เดียว ไม่ต้องเดาว่า endpoint ไหนคืนรูปแบบไหน
// ---------------------------------------------------------------------------
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

การประกาศเป็น `inline` function ใน header ทำให้ include ไฟล์นี้จากหลาย `.cpp` ได้โดยไม่เกิด
"multiple definition error" ตอน link (ทบทวนเรื่อง linkage จาก Part 17) — เป็นแพทเทิร์นมาตรฐาน
สำหรับฟังก์ชันช่วยเล็กๆ ที่อยากประกาศพร้อม implementation ในไฟล์เดียว

---

## 108.5 Data Access Layer: เชื่อมต่อ SQLite ผ่านคลาส Db (Step 861)

นี่คือหัวใจของโปรเจกต์ในแง่การจัดการข้อมูล คลาส `Db` ห่อหุ้มทุกการคุยกับ SQLite ไว้ที่เดียว
ชั้นอื่นจะไม่เห็น `sqlite3_stmt*` หรือ SQL string เลยแม้แต่นิดเดียว

`include/db.hpp`:

```cpp
#pragma once
#include <sqlite3.h>
#include <string>
#include <optional>
#include <vector>
#include "task.hpp"

// Db คือชั้น "Data Access Layer" (DAL) — ห่อหุ้มการคุยกับ SQLite ทั้งหมดไว้ที่นี่ที่เดียว
// ชั้นอื่น (routes) จะไม่รู้จัก sqlite3_stmt หรือ SQL เลยแม้แต่นิดเดียว
class Db {
public:
    explicit Db(const std::string& path);
    ~Db();

    Db(const Db&) = delete;
    Db& operator=(const Db&) = delete;

    void init_schema();

    std::vector<Task> list_all();
    std::optional<Task> find_by_id(int id);
    Task create(const std::string& title, const std::string& description, bool done);
    bool update(int id, const std::string& title, const std::string& description, bool done);
    bool remove(int id);

private:
    sqlite3* conn_ = nullptr;
};
```

สังเกตว่า `Db(const Db&) = delete;` และ `Db& operator=(const Db&) = delete;` — เราลบ copy
constructor/assignment ทิ้งไปโดยตั้งใจ เพราะ `Db` ถือ resource ระบบปฏิบัติการ (file handle
ของ SQLite connection ผ่าน `sqlite3*`) การก็อปปี้ struct ที่ถือ raw pointer แบบนี้จะทำให้เกิด
**double-free** ตอน destructor ถูกเรียกสองครั้งกับ pointer เดียวกัน (ทบทวนแนวคิด Rule of
Three/Five จาก Part 46 และ RAII จาก Part 68)

`src/db.cpp`:

```cpp
#include "db.hpp"
#include <stdexcept>
#include <cstring>

Db::Db(const std::string& path) {
    int rc = sqlite3_open(path.c_str(), &conn_);
    if (rc != SQLITE_OK) {
        std::string msg = "เปิดฐานข้อมูลไม่สำเร็จ: ";
        msg += sqlite3_errmsg(conn_);
        throw std::runtime_error(msg);
    }
    // เปิด foreign_keys และ WAL mode ไว้เป็นนิสัยที่ดี แม้ตารางเดียวจะยังไม่จำเป็นนัก
    sqlite3_exec(conn_, "PRAGMA journal_mode=WAL;", nullptr, nullptr, nullptr);
}

Db::~Db() {
    if (conn_) sqlite3_close(conn_);
}

void Db::init_schema() {
    const char* sql = R"SQL(
        CREATE TABLE IF NOT EXISTS tasks (
            id          INTEGER PRIMARY KEY AUTOINCREMENT,
            title       TEXT NOT NULL,
            description TEXT NOT NULL DEFAULT '',
            done        INTEGER NOT NULL DEFAULT 0,
            created_at  TEXT NOT NULL DEFAULT (datetime('now'))
        );
    )SQL";
    char* err_msg = nullptr;
    int rc = sqlite3_exec(conn_, sql, nullptr, nullptr, &err_msg);
    if (rc != SQLITE_OK) {
        std::string msg = "สร้างตารางไม่สำเร็จ: ";
        msg += err_msg;
        sqlite3_free(err_msg);
        throw std::runtime_error(msg);
    }
}

static Task row_to_task(sqlite3_stmt* stmt) {
    Task t;
    t.id = sqlite3_column_int(stmt, 0);
    t.title = reinterpret_cast<const char*>(sqlite3_column_text(stmt, 1));
    const unsigned char* desc = sqlite3_column_text(stmt, 2);
    t.description = desc ? reinterpret_cast<const char*>(desc) : "";
    t.done = sqlite3_column_int(stmt, 3) != 0;
    t.created_at = reinterpret_cast<const char*>(sqlite3_column_text(stmt, 4));
    return t;
}

std::vector<Task> Db::list_all() {
    std::vector<Task> result;
    const char* sql = "SELECT id, title, description, done, created_at "
                       "FROM tasks ORDER BY id ASC;";
    sqlite3_stmt* stmt = nullptr;
    if (sqlite3_prepare_v2(conn_, sql, -1, &stmt, nullptr) != SQLITE_OK) {
        throw std::runtime_error(sqlite3_errmsg(conn_));
    }
    while (sqlite3_step(stmt) == SQLITE_ROW) {
        result.push_back(row_to_task(stmt));
    }
    sqlite3_finalize(stmt);
    return result;
}

std::optional<Task> Db::find_by_id(int id) {
    const char* sql = "SELECT id, title, description, done, created_at "
                       "FROM tasks WHERE id = ?;";
    sqlite3_stmt* stmt = nullptr;
    if (sqlite3_prepare_v2(conn_, sql, -1, &stmt, nullptr) != SQLITE_OK) {
        throw std::runtime_error(sqlite3_errmsg(conn_));
    }
    sqlite3_bind_int(stmt, 1, id);

    std::optional<Task> result;
    if (sqlite3_step(stmt) == SQLITE_ROW) {
        result = row_to_task(stmt);
    }
    sqlite3_finalize(stmt);
    return result;
}

Task Db::create(const std::string& title, const std::string& description, bool done) {
    const char* sql = "INSERT INTO tasks (title, description, done) VALUES (?, ?, ?);";
    sqlite3_stmt* stmt = nullptr;
    if (sqlite3_prepare_v2(conn_, sql, -1, &stmt, nullptr) != SQLITE_OK) {
        throw std::runtime_error(sqlite3_errmsg(conn_));
    }
    // SQLITE_TRANSIENT บอกให้ SQLite "คัดลอก" string ไปเก็บเอง เพราะ std::string
    // ต้นทางอาจถูกทำลายก่อน statement จะถูก step จริง
    sqlite3_bind_text(stmt, 1, title.c_str(), -1, SQLITE_TRANSIENT);
    sqlite3_bind_text(stmt, 2, description.c_str(), -1, SQLITE_TRANSIENT);
    sqlite3_bind_int(stmt, 3, done ? 1 : 0);

    if (sqlite3_step(stmt) != SQLITE_DONE) {
        sqlite3_finalize(stmt);
        throw std::runtime_error(sqlite3_errmsg(conn_));
    }
    sqlite3_finalize(stmt);

    int new_id = static_cast<int>(sqlite3_last_insert_rowid(conn_));
    auto created = find_by_id(new_id);
    return *created; // ปลอดภัย เพราะเพิ่ง insert สำเร็จแน่นอน
}

bool Db::update(int id, const std::string& title, const std::string& description, bool done) {
    const char* sql = "UPDATE tasks SET title = ?, description = ?, done = ? WHERE id = ?;";
    sqlite3_stmt* stmt = nullptr;
    if (sqlite3_prepare_v2(conn_, sql, -1, &stmt, nullptr) != SQLITE_OK) {
        throw std::runtime_error(sqlite3_errmsg(conn_));
    }
    sqlite3_bind_text(stmt, 1, title.c_str(), -1, SQLITE_TRANSIENT);
    sqlite3_bind_text(stmt, 2, description.c_str(), -1, SQLITE_TRANSIENT);
    sqlite3_bind_int(stmt, 3, done ? 1 : 0);
    sqlite3_bind_int(stmt, 4, id);

    if (sqlite3_step(stmt) != SQLITE_DONE) {
        sqlite3_finalize(stmt);
        throw std::runtime_error(sqlite3_errmsg(conn_));
    }
    int changed = sqlite3_changes(conn_);
    sqlite3_finalize(stmt);
    return changed > 0;
}

bool Db::remove(int id) {
    const char* sql = "DELETE FROM tasks WHERE id = ?;";
    sqlite3_stmt* stmt = nullptr;
    if (sqlite3_prepare_v2(conn_, sql, -1, &stmt, nullptr) != SQLITE_OK) {
        throw std::runtime_error(sqlite3_errmsg(conn_));
    }
    sqlite3_bind_int(stmt, 1, id);
    if (sqlite3_step(stmt) != SQLITE_DONE) {
        sqlite3_finalize(stmt);
        throw std::runtime_error(sqlite3_errmsg(conn_));
    }
    int changed = sqlite3_changes(conn_);
    sqlite3_finalize(stmt);
    return changed > 0;
}
```

### จุดสำคัญที่ต้องเข้าใจให้ลึก

1. **Prepared Statement ทุกที่ ไม่มีการต่อ string SQL ด้วยมือเลยสักบรรทัด** — ค่าจากผู้ใช้
   (`title`, `description`) เข้าสู่ query ผ่าน `sqlite3_bind_*` เท่านั้น นี่คือการป้องกัน
   **SQL Injection** ที่ถูกต้อง 100% ถ้าเขียน `"INSERT INTO tasks VALUES ('" + title + "')"`
   ตรงๆ ผู้ใช้ที่ส่ง `title` เป็น `'); DROP TABLE tasks; --` จะทำลายฐานข้อมูลได้ทันที
2. **`SQLITE_TRANSIENT` vs `SQLITE_STATIC`**: `SQLITE_TRANSIENT` บอก SQLite ให้คัดลอกข้อมูล
   ไปเก็บเองทันที ปลอดภัยเสมอแต่ช้ากว่าเล็กน้อย ส่วน `SQLITE_STATIC` บอกว่า "ข้อมูลนี้จะยังอยู่
   ตลอดไป ไม่ต้องคัดลอก" ซึ่งเสี่ยงมากถ้าตัวแปรต้นทางเป็น local variable ที่จะถูกทำลายก่อน
   `sqlite3_step()` ถูกเรียก ในโปรเจกต์นี้เราใช้ `SQLITE_TRANSIENT` เสมอเพื่อความปลอดภัย
3. **`sqlite3_changes()`** คืนจำนวนแถวที่ถูกกระทบจากคำสั่งล่าสุด — เราใช้สิ่งนี้เพื่อรู้ว่า
   `UPDATE`/`DELETE` เจอแถวที่ตรงเงื่อนไขจริงหรือไม่ (ถ้าเป็น 0 แปลว่า id ที่ส่งมาไม่มีอยู่จริง
   → ต้องคืน 404)

### ประเด็นสำคัญที่มักถูกมองข้าม: Thread-Safety ของ Db

`main.cpp` เรียก `app.port(18180).multithreaded().run();` — `.multithreaded()` บอก Crow ให้ใช้
worker thread หลายตัวรับ request พร้อมกัน (จำนวน thread เท่ากับ `std::thread::hardware_concurrency()`
โดย default) นั่นแปลว่า **หลาย thread อาจเรียกเมธอดของ `Db` object ตัวเดียวกันพร้อมกันได้จริง**
เพราะเราสร้าง `Db db("tasks.db");` เพียงตัวเดียวใน `main()` แล้วส่ง reference เดียวกันเข้าไปให้
ทุก route handler ใช้ร่วมกัน

คำถามที่ต้องตอบให้ได้คือ: โค้ดใน `db.cpp` ปลอดภัยจาก **Data Race** หรือไม่?

คำตอบคือ **ปลอดภัย** ในกรณีนี้ เพราะเหตุผลเฉพาะของ SQLite ไม่ใช่เพราะโค้ดเราเขียน
synchronization เอง:

- ไลบรารี SQLite บน Ubuntu/Debian (และแทบทุก distro) ถูกคอมไพล์มาโดยเปิด
  `SQLITE_THREADSAFE=1` ซึ่งหมายถึงโหมด **Serialized** — SQLite ใส่ mutex ป้องกันไว้ภายใน
  ตัวมันเองรอบทุกการเรียก API จึงปลอดภัยที่จะให้หลาย thread ใช้ `sqlite3*` connection ตัวเดียว
  ร่วมกันได้โดยไม่ทำให้ข้อมูลเสียหาย (แต่จะมีการ "รอคิว" กันภายใน ทำให้ throughput ไม่ได้เพิ่ม
  ตามจำนวน thread เสมอไป)
- ตรวจสอบโหมด thread-safety ของ SQLite ที่ติดตั้งอยู่ได้ด้วยคำสั่งนี้:

```bash
$ echo "PRAGMA compile_options;" | sqlite3 :memory: 2>/dev/null | grep -i thread
# หรือถ้าไม่มี sqlite3 CLI ติดตั้งไว้ ตรวจผ่านโค้ด C++ ได้ด้วย sqlite3_threadsafe()
```

```cpp
#include <sqlite3.h>
#include <cstdio>
int main() {
    printf("SQLite threadsafe mode = %d\n", sqlite3_threadsafe());
    // 0 = Single-thread, 1 = Serialized (ปลอดภัยแชร์ connection ข้าม thread), 2 = Multi-thread
}
```

> **ข้อควรระวังสำหรับโปรเจกต์จริง**: อย่าพึ่งพา default ของระบบเสมอไป ถ้าจะ deploy จริงควร
> เรียก `sqlite3_threadsafe()` ตรวจสอบตอน startup หรือดีที่สุดคือใช้ **connection pool**
> (สร้าง `Db` หลายตัว หนึ่งตัวต่อ thread หรือใช้ pool ขนาดคงที่) แทนการแชร์ connection เดียว
> เพื่อทั้งความปลอดภัยที่ชัดเจนกว่าและ throughput ที่ดีกว่า ซึ่งเป็นแนวทางที่ระบบ production
> จริงส่วนใหญ่เลือกใช้ โดยเฉพาะเมื่อเปลี่ยนไปใช้ PostgreSQL (Part 106) ที่ connection pool
> เป็นเรื่องจำเป็นมากกว่า SQLite เสียอีก

---

## 108.6 Routes Layer ตอนที่ 1: GET /tasks และ GET /tasks/<id> (Step 862)

`include/task_routes.hpp`:

```cpp
#pragma once
#include "crow.h"
#include "db.hpp"

// ลงทะเบียน route ทั้งหมดที่เกี่ยวกับ resource "Task" เข้ากับ Crow app
// รับ Db มาโดยอ้างอิง (reference) เพื่อให้ handler ทุกตัวใช้ connection เดียวกัน
void register_task_routes(crow::SimpleApp& app, Db& db);
```

เริ่มต้น `src/task_routes.cpp` ด้วยสอง endpoint แรกที่เป็น "อ่านอย่างเดียว" (read-only)
จึงไม่ต้องกังวลเรื่อง validation มากนัก:

```cpp
#include "task_routes.hpp"
#include "response.hpp"
#include <nlohmann/json.hpp>

using json = nlohmann::json;

void register_task_routes(crow::SimpleApp& app, Db& db) {

    // GET /tasks — ดึงรายการ Task ทั้งหมด
    CROW_ROUTE(app, "/tasks")
    .methods(crow::HTTPMethod::GET)
    ([&db]() {
        try {
            auto tasks = db.list_all();
            json arr = json::array();
            for (const auto& t : tasks) arr.push_back(t.to_json());
            return api::ok(arr);
        } catch (const std::exception& e) {
            return api::error(500, "INTERNAL_ERROR", e.what());
        }
    });

    // GET /tasks/<id> — ดึง Task ตัวเดียวตาม id
    CROW_ROUTE(app, "/tasks/<int>")
    .methods(crow::HTTPMethod::GET)
    ([&db](int id) {
        try {
            auto task = db.find_by_id(id);
            if (!task) {
                return api::error(404, "TASK_NOT_FOUND",
                    "ไม่พบ Task ที่มี id = " + std::to_string(id));
            }
            return api::ok(task->to_json());
        } catch (const std::exception& e) {
            return api::error(500, "INTERNAL_ERROR", e.what());
        }
    });

    // (POST, PUT, DELETE จะเพิ่มในหัวข้อถัดไป)
}
```

### เส้นทางของ Request หนึ่งตัว ผ่านทุกชั้นของระบบ

ก่อนลงรายละเอียดโค้ด ควรเห็นภาพรวมว่า request หนึ่งตัวเดินทางผ่านชั้นไหนบ้างกว่าจะได้
response กลับไป — สมมติ client ยิง `GET /tasks/1` เข้ามา:

```
Client (curl)
   │  HTTP GET /tasks/1
   ▼
Crow HTTP Server (จัดการ socket, parse HTTP request ให้เป็น crow::request)
   │
   ▼
Crow Router (จับคู่ path "/tasks/<int>" แปลง "1" เป็น int ให้อัตโนมัติ)
   │
   ▼
Route Handler ใน task_routes.cpp  (ชั้น "Routes")
   │  เรียก db.find_by_id(1)
   ▼
Db::find_by_id()  ใน db.cpp  (ชั้น "Data Access Layer")
   │  sqlite3_prepare_v2 → sqlite3_bind_int → sqlite3_step
   ▼
SQLite Engine (อ่านไฟล์ tasks.db บนดิสก์ หรือจาก page cache)
   │  คืนแถวข้อมูลกลับมาเป็น sqlite3_stmt
   ▼
row_to_task()  แปลงแถว SQLite เป็น struct Task  (ชั้น "Model")
   │
   ▼
Route Handler เรียก task->to_json() แล้วห่อด้วย api::ok()  (ชั้น "Response Helper")
   │
   ▼
Crow ส่ง crow::response กลับเป็น HTTP response จริง
   │
   ▼
Client ได้รับ {"success":true,"data":{...}}
```

การเห็นภาพรวมนี้ช่วยตอบคำถามสำคัญ: **ถ้าอยากรู้ว่าบั๊กอยู่ชั้นไหน ให้ไล่ตามลูกศรทีละขั้น** —
ถ้า response ผิดรูปแบบ ปัญหาอยู่ที่ Response Helper หรือ Route Handler, ถ้าข้อมูลผิด ปัญหาอยู่ที่
Data Access Layer หรือ Model, ถ้า route ไม่ถูกเรียกเลย ปัญหาอยู่ที่ Router การแยกชั้นทำให้
"พื้นที่สงสัย" แคบลงทันทีเมื่อเจอบั๊ก แทนที่จะต้องไล่อ่านทั้งไฟล์เป็นพันบรรทัด

จุดที่ควรสังเกต: `CROW_ROUTE(app, "/tasks/<int>")` ใช้ **path parameter แบบมี type** ของ
Crow — `<int>` บอก Crow ว่าต้องแปลงส่วนนี้ของ URL เป็น `int` ให้อัตโนมัติก่อนเรียก handler
ถ้า URL เป็น `/tasks/abc` (ไม่ใช่ตัวเลข) Crow จะไม่ match route นี้เลย (คืน 404 แบบ built-in)
โดยที่เราไม่ต้องเขียน parsing เอง — ต่างจากการอ่าน string แล้ว `std::stoi` เองซึ่งต้อง
`try-catch` การแปลงที่ผิดพลาดด้วยตัวเอง

การจับ `try/catch` รอบ logic ทุก handler เป็นแพทเทิร์นที่ตั้งใจทำซ้ำในทุก endpoint — ป้องกัน
ไม่ให้ exception ที่ไม่คาดคิด (เช่น ฐานข้อมูลถูกลบไปกลางทาง) ทำให้ทั้ง process ล่ม (crash)
กลายเป็นแค่ response `500` ที่ client จัดการต่อได้ นี่คือความแตกต่างสำคัญระหว่างโค้ดตัวอย่าง
กับโค้ด production

---

## 108.7 Routes Layer ตอนที่ 2: POST /tasks พร้อม Input Validation (Step 863)

ก่อนเขียน handler ของ `POST` เราต้องมีฟังก์ชัน validate ข้อมูลที่ผู้ใช้ส่งเข้ามาก่อน เพราะ
input จากภายนอกคือสิ่งที่ **ไม่มีวันไว้ใจได้** เสมอ (กฎทองข้อแรกของ Security ที่จะเรียนลึกใน
Part 110 และ Part 115):

```cpp
// ---------------------------------------------------------------------------
// ฟังก์ชันช่วย validate input ที่ผู้ใช้ส่งเข้ามาสำหรับ create/update
// คืนค่า std::string ว่าง = ผ่าน, ถ้าไม่ผ่านจะคืนข้อความ error
// ---------------------------------------------------------------------------
static std::string validate_task_payload(const json& body, bool require_all_fields) {
    if (require_all_fields && !body.contains("title")) {
        return "ต้องมีฟิลด์ \"title\"";
    }
    if (body.contains("title")) {
        if (!body["title"].is_string()) {
            return "ฟิลด์ \"title\" ต้องเป็น string";
        }
        std::string title = body["title"].get<std::string>();
        if (title.empty() || title.size() > 200) {
            return "ฟิลด์ \"title\" ต้องมีความยาว 1-200 ตัวอักษร";
        }
    }
    if (body.contains("description") && !body["description"].is_string()) {
        return "ฟิลด์ \"description\" ต้องเป็น string";
    }
    if (body.contains("done") && !body["done"].is_boolean()) {
        return "ฟิลด์ \"done\" ต้องเป็น boolean (true/false)";
    }
    return "";
}
```

ฟังก์ชันนี้รับพารามิเตอร์ `require_all_fields` เพื่อใช้ซ้ำได้ทั้งกรณี `POST` (ต้องมี `title`
เสมอ) และกรณี `PUT` (ทุกฟิลด์เป็นทางเลือก แต่ถ้าส่งมาต้อง valid) — หลีกเลี่ยงการเขียนโค้ด
validate ซ้ำสองชุด

Handler ของ `POST /tasks`:

```cpp
    // POST /tasks — สร้าง Task ใหม่
    CROW_ROUTE(app, "/tasks")
    .methods(crow::HTTPMethod::POST)
    ([&db](const crow::request& req) {
        json body;
        try {
            body = json::parse(req.body);
        } catch (const json::parse_error&) {
            return api::error(400, "INVALID_JSON", "Body ไม่ใช่ JSON ที่ถูกต้อง");
        }

        if (!body.is_object()) {
            return api::error(400, "INVALID_JSON", "Body ต้องเป็น JSON object");
        }

        std::string err = validate_task_payload(body, /*require_all_fields=*/true);
        if (!err.empty()) {
            return api::error(400, "VALIDATION_ERROR", err);
        }

        try {
            std::string title = body["title"].get<std::string>();
            std::string description = body.value("description", "");
            bool done = body.value("done", false);
            Task created = db.create(title, description, done);
            return api::ok(created.to_json(), /*status=*/201);
        } catch (const std::exception& e) {
            return api::error(500, "INTERNAL_ERROR", e.what());
        }
    });
```

ลำดับการตรวจสอบสำคัญมาก — เรียงจาก "ถูกที่สุด" ไปหา "แพงที่สุด" เสมอ:

1. **Parse JSON ก่อน** — ถ้า body ไม่ใช่ JSON เลยด้วยซ้ำ ไม่มีประโยชน์จะตรวจอะไรต่อ
2. **ตรวจว่าเป็น object** — ป้องกันกรณีส่ง JSON ที่ valid แต่เป็น array หรือตัวเลขเปล่าๆ มา
3. **Validate ฟิลด์และชนิดข้อมูล** — ก่อนจะแตะฐานข้อมูลเลย
4. **สุดท้ายค่อยเขียนลงฐานข้อมูล** ซึ่งเป็นการดำเนินการที่ "แพง" ที่สุดในบรรดาทั้งหมด

`body.value("description", "")` เป็นเมธอดของ nlohmann/json ที่เรียนไปแล้วใน Part 107 —
คืนค่า default (`""`) ทันทีถ้า key ไม่มีอยู่ แทนที่จะต้องเขียน `if (body.contains(...))`
ทุกครั้ง ทำให้โค้ดกระชับขึ้นมาก

---

## 108.8 Routes Layer ตอนที่ 3, main.cpp, Build & ทดสอบทุก Endpoint (Step 864)

เติม `PUT` และ `DELETE` ให้ครบ 5 endpoint:

```cpp
    // PUT /tasks/<id> — แก้ไข Task ที่มีอยู่ (แทนที่ทั้งฟิลด์ title/description/done)
    CROW_ROUTE(app, "/tasks/<int>")
    .methods(crow::HTTPMethod::PUT)
    ([&db](const crow::request& req, int id) {
        auto existing = db.find_by_id(id);
        if (!existing) {
            return api::error(404, "TASK_NOT_FOUND",
                "ไม่พบ Task ที่มี id = " + std::to_string(id));
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

        // PUT อนุญาตให้ส่งมาบางฟิลด์ (partial update) เพื่อความสะดวกใช้งานจริง
        // แต่ทุกฟิลด์ที่ส่งมาต้อง valid
        std::string err = validate_task_payload(body, /*require_all_fields=*/false);
        if (!err.empty()) {
            return api::error(400, "VALIDATION_ERROR", err);
        }

        try {
            std::string title = body.value("title", existing->title);
            std::string description = body.value("description", existing->description);
            bool done = body.value("done", existing->done);
            db.update(id, title, description, done);
            auto updated = db.find_by_id(id);
            return api::ok(updated->to_json());
        } catch (const std::exception& e) {
            return api::error(500, "INTERNAL_ERROR", e.what());
        }
    });

    // DELETE /tasks/<id> — ลบ Task
    CROW_ROUTE(app, "/tasks/<int>")
    .methods(crow::HTTPMethod::DELETE)
    ([&db](int id) {
        try {
            bool removed = db.remove(id);
            if (!removed) {
                return api::error(404, "TASK_NOT_FOUND",
                    "ไม่พบ Task ที่มี id = " + std::to_string(id));
            }
            return api::ok(json{{"deleted_id", id}});
        } catch (const std::exception& e) {
            return api::error(500, "INTERNAL_ERROR", e.what());
        }
    });
}
```

สังเกตว่า `PUT` เช็คว่า Task มีอยู่จริงก่อน (`find_by_id`) ก่อนจะไป parse body ด้วยซ้ำ —
เพราะถ้า id ไม่มีอยู่จริง เราอยากคืน `404` ทันทีโดยไม่ต้องเสียเวลา validate body ที่ไม่มี
ประโยชน์อะไรแล้ว (fail fast)

### main.cpp — จุดประกอบร่างทุกชิ้นเข้าด้วยกัน

```cpp
#include "crow.h"
#include "db.hpp"
#include "task_routes.hpp"
#include "response.hpp"
#include <nlohmann/json.hpp>

int main() {
    crow::SimpleApp app;

    // เปิดฐานข้อมูล SQLite แบบไฟล์ (persist ข้ามการรีสตาร์ท)
    Db db("tasks.db");
    db.init_schema();

    register_task_routes(app, db);

    // Health check endpoint — เอาไว้ให้ load balancer / uptime monitor เรียกเช็คสถานะ
    CROW_ROUTE(app, "/health")
    ([]() {
        return api::ok(nlohmann::json{{"status", "up"}}, 200);
    });

    app.port(18080).multithreaded().run();
}
```

`main.cpp` มีความยาวแค่ 20 กว่าบรรทัด แต่ทำหน้าที่สำคัญคือ "ประกอบร่าง" (composition) —
สร้าง `Db`, ส่ง reference เข้าไปให้ `register_task_routes()` ลงทะเบียน route ทั้งหมด แล้วสั่ง
รัน server นี่คือประโยชน์ที่จับต้องได้ของ Layered Architecture: `main.cpp` อ่านแล้วเข้าใจ
"ภาพรวม" ของทั้งระบบได้ในหน้าเดียว โดยไม่ต้องไล่อ่านรายละเอียดของแต่ละ route

> **หมายเหตุเรื่องพอร์ต**: ในบทเรียนนี้ตัวอย่างจริงที่ทดสอบบนเครื่องใช้พอร์ต 18180 แทน 18080
> เพราะพอร์ต 18080 ถูกใช้งานโดยโปรเซสอื่นอยู่ก่อนแล้วบนเครื่องทดสอบ (เจอ error
> `Address already in use` ตอนรันจริง) — ในเครื่องของผู้เรียนเองสามารถใช้พอร์ตใดก็ได้ที่ว่าง
> นี่คือเหตุผลที่ "Health Check" และ "การอ่าน error message จาก log ให้เป็น" สำคัญมากในงาน
> จริง เพราะเซิร์ฟเวอร์ไม่ยอมเปิดพอร์ตแต่ไม่มีข้อความ error ให้อ่านเลยเป็นสถานการณ์ที่พบบ่อย
> ที่สุดอย่างหนึ่งตอน deploy จริง

### Build และทดสอบทุก Endpoint ด้วย curl จริง

Build:

```bash
$ cmake -S . -B build -DCMAKE_BUILD_TYPE=Release
$ cmake --build build -j4
[100%] Built target task_api
```

รัน server (พอร์ต 18180):

```bash
$ ./build/task_api
```

**ทดสอบ 1 — Health check:**

```bash
$ curl -s http://127.0.0.1:18180/health
{"data":{"status":"up"},"success":true}
```

**ทดสอบ 2 — GET /tasks ตอนฐานข้อมูลยังว่าง:**

```bash
$ curl -s http://127.0.0.1:18180/tasks
{"data":[],"success":true}
```

**ทดสอบ 3 — POST /tasks สร้าง Task ใหม่ 2 รายการ:**

```bash
$ curl -s -i -X POST http://127.0.0.1:18180/tasks -H "Content-Type: application/json" \
  -d '{"title":"เรียนเขียน REST API ด้วย Crow","description":"ทำ Part 108 ให้จบ"}'

HTTP/1.1 201 Created
Content-Type: application/json
Content-Length: 196
Server: Crow/master
Connection: Keep-Alive

{"data":{"created_at":"2026-09-26 09:12:56","description":"ทำ Part 108 ให้จบ","done":false,"id":1,"title":"เรียนเขียน REST API ด้วย Crow"},"success":true}
```

```bash
$ curl -s -X POST http://127.0.0.1:18180/tasks -H "Content-Type: application/json" \
  -d '{"title":"ซื้อของเข้าบ้าน","description":"นม ไข่ ขนมปัง","done":false}'

{"data":{"created_at":"2026-09-26 09:12:56","description":"นม ไข่ ขนมปัง","done":false,"id":2,"title":"ซื้อของเข้าบ้าน"},"success":true}
```

สังเกต status code `201 Created` (ไม่ใช่ 200) และค่า `created_at` ที่ฐานข้อมูลสร้างให้
อัตโนมัติ — client ไม่ต้องส่งเวลามาเอง ป้องกันปัญหา clock ไม่ตรงกันระหว่าง client กับ server

**ทดสอบ 4 — GET /tasks ดูรายการทั้งหมด:**

```bash
$ curl -s http://127.0.0.1:18180/tasks
{"data":[{"created_at":"2026-09-26 09:12:56","description":"ทำ Part 108 ให้จบ","done":false,"id":1,"title":"เรียนเขียน REST API ด้วย Crow"},{"created_at":"2026-09-26 09:12:56","description":"นม ไข่ ขนมปัง","done":false,"id":2,"title":"ซื้อของเข้าบ้าน"}],"success":true}
```

**ทดสอบ 5 — GET /tasks/1 (พบ) กับ GET /tasks/999 (ไม่พบ):**

```bash
$ curl -s -i http://127.0.0.1:18180/tasks/1
HTTP/1.1 200 OK
...
{"data":{"created_at":"2026-09-26 09:12:56","description":"ทำ Part 108 ให้จบ","done":false,"id":1,"title":"เรียนเขียน REST API ด้วย Crow"},"success":true}

$ curl -s -i http://127.0.0.1:18180/tasks/999
HTTP/1.1 404 Not Found
...
{"error":{"code":"TASK_NOT_FOUND","message":"ไม่พบ Task ที่มี id = 999"},"success":false}
```

**ทดสอบ 6 — PUT /tasks/2 (mark done) กับ PUT ของ id ที่ไม่มีอยู่จริง:**

```bash
$ curl -s -i -X PUT http://127.0.0.1:18180/tasks/2 -H "Content-Type: application/json" \
  -d '{"done":true}'

HTTP/1.1 200 OK
...
{"data":{"created_at":"2026-09-26 09:12:56","description":"นม ไข่ ขนมปัง","done":true,"id":2,"title":"ซื้อของเข้าบ้าน"},"success":true}

$ curl -s -i -X PUT http://127.0.0.1:18180/tasks/999 -H "Content-Type: application/json" \
  -d '{"done":true}'

HTTP/1.1 404 Not Found
...
{"error":{"code":"TASK_NOT_FOUND","message":"ไม่พบ Task ที่มี id = 999"},"success":false}
```

ส่งแค่ `{"done":true}` โดยไม่ระบุ `title`/`description` เลย และ Task ยังคงค่าฟิลด์เดิมไว้ครบ
— นี่คือ partial update ที่ทำงานถูกต้องตามที่ออกแบบไว้

**ทดสอบ 7 — DELETE /tasks/2:**

```bash
$ curl -s -i -X DELETE http://127.0.0.1:18180/tasks/2
HTTP/1.1 200 OK
...
{"data":{"deleted_id":2},"success":true}

$ curl -s http://127.0.0.1:18180/tasks
{"data":[{"created_at":"2026-09-26 09:12:56","description":"ทำ Part 108 ให้จบ","done":false,"id":1,"title":"เรียนเขียน REST API ด้วย Crow"}],"success":true}
```

**ทดสอบ 8 — Validation Error ทุกกรณี:**

```bash
$ curl -s -i -X POST http://127.0.0.1:18180/tasks -H "Content-Type: application/json" \
  -d '{"description":"ไม่มี title"}'

HTTP/1.1 400 Bad Request
...
{"error":{"code":"VALIDATION_ERROR","message":"ต้องมีฟิลด์ \"title\""},"success":false}

$ curl -s -i -X POST http://127.0.0.1:18180/tasks -H "Content-Type: application/json" \
  -d 'not-json-at-all'

HTTP/1.1 400 Bad Request
...
{"error":{"code":"INVALID_JSON","message":"Body ไม่ใช่ JSON ที่ถูกต้อง"},"success":false}

$ curl -s -i -X POST http://127.0.0.1:18180/tasks -H "Content-Type: application/json" \
  -d '{"title":"ok","done":"yes"}'

HTTP/1.1 400 Bad Request
...
{"error":{"code":"VALIDATION_ERROR","message":"ฟิลด์ \"done\" ต้องเป็น boolean (true/false)"},"success":false}
```

ทั้งหมดนี้คือผลลัพธ์จริงที่รันได้บนเครื่องจริง (g++ 13.3.0, Crow master, SQLite 3.45.1) —
ทุก endpoint, ทุก status code, และทุก error case ทำงานตรงตามที่ออกแบบไว้ในหัวข้อ 108.1
และ 108.2 ทุกประการ

### สคริปต์ทดสอบอัตโนมัติ

การพิมพ์คำสั่ง `curl` ทีละคำสั่งเวลาทดสอบทุกครั้งไม่ใช่แนวทางที่ยั่งยืน โปรเจกต์จริงควรมี
สคริปต์ทดสอบที่รันซ้ำได้ (repeatable) เก็บไว้ในโปรเจกต์ ตัวอย่าง `test_api.sh` ง่ายๆ ที่ไล่
ทดสอบทุก endpoint และเช็ค HTTP status code อัตโนมัติด้วย `curl -o /dev/null -w "%{http_code}"`:

```bash
#!/bin/bash
# test_api.sh — ทดสอบ Task API ทุก endpoint แบบอัตโนมัติ
BASE="http://127.0.0.1:18180"
PASS=0
FAIL=0

check() {
    local desc="$1" expected="$2" actual="$3"
    if [ "$expected" = "$actual" ]; then
        echo "[PASS] $desc (status=$actual)"
        PASS=$((PASS+1))
    else
        echo "[FAIL] $desc (expected=$expected, got=$actual)"
        FAIL=$((FAIL+1))
    fi
}

status() { curl -s -o /dev/null -w "%{http_code}" "$@"; }

check "GET /health"               200 "$(status $BASE/health)"
check "GET /tasks (list)"         200 "$(status $BASE/tasks)"
check "POST /tasks (valid)"       201 "$(status -X POST $BASE/tasks -d '{"title":"t1"}')"
check "GET /tasks/1 (found)"      200 "$(status $BASE/tasks/1)"
check "GET /tasks/9999 (missing)" 404 "$(status $BASE/tasks/9999)"
check "PUT /tasks/1 (valid)"      200 "$(status -X PUT $BASE/tasks/1 -d '{"done":true}')"
check "PUT /tasks/9999 (missing)" 404 "$(status -X PUT $BASE/tasks/9999 -d '{"done":true}')"
check "POST /tasks (no title)"    400 "$(status -X POST $BASE/tasks -d '{}')"
check "DELETE /tasks/1"           200 "$(status -X DELETE $BASE/tasks/1)"
check "DELETE /tasks/1 (again)"   404 "$(status -X DELETE $BASE/tasks/1)"

echo "----------------------------------------"
echo "ผ่าน $PASS เคส, ไม่ผ่าน $FAIL เคส"
[ "$FAIL" -eq 0 ] && exit 0 || exit 1
```

ผลลัพธ์จริงจากการรันสคริปต์นี้กับเซิร์ฟเวอร์ที่กำลังทำงานอยู่:

```
[PASS] GET /health (status=200)
[PASS] GET /tasks (list) (status=200)
[PASS] POST /tasks (valid) (status=201)
[PASS] GET /tasks/1 (found) (status=200)
[PASS] GET /tasks/9999 (missing) (status=404)
[PASS] PUT /tasks/1 (valid) (status=200)
[PASS] PUT /tasks/9999 (missing) (status=404)
[PASS] POST /tasks (no title) (status=400)
[PASS] DELETE /tasks/1 (status=200)
[PASS] DELETE /tasks/1 (again) (status=404)
----------------------------------------
ผ่าน 10 เคส, ไม่ผ่าน 0 เคส
```

สคริปต์แบบนี้ (แม้จะเรียบง่ายกว่า unit test framework อย่าง Google Test ที่จะเรียนใน
Part 93 มาก) มีค่ามหาศาลในทางปฏิบัติ เพราะรันได้ใน CI/CD pipeline (Part 94) ทุกครั้งที่มีการ
แก้โค้ด ทำให้รู้ทันทีถ้า endpoint ไหนพังจากการแก้ไขครั้งล่าสุด โดยไม่ต้องพิมพ์ `curl` ทีละ
คำสั่งด้วยมือซ้ำๆ

### CORS: เมื่อ Client เป็นเว็บเบราว์เซอร์

ถ้า Task API นี้จะถูกเรียกจากเว็บแอปที่รันอยู่คนละ origin (เช่น frontend อยู่ที่
`http://localhost:3000` แต่ API อยู่ที่ `http://localhost:18180`) เบราว์เซอร์จะบล็อก request
ด้วยนโยบาย **Same-Origin Policy** ทันที ต้องเปิด **CORS (Cross-Origin Resource Sharing)**
โดยเพิ่ม header ที่เหมาะสมในทุก response:

```cpp
// เพิ่มใน main.cpp ก่อน register route หรือทำเป็น middleware แยก (Part 103 สอน middleware ไว้แล้ว)
CROW_ROUTE(app, "/tasks").methods(crow::HTTPMethod::OPTIONS)
([]() {
    crow::response res(204);
    res.set_header("Access-Control-Allow-Origin", "http://localhost:3000");
    res.set_header("Access-Control-Allow-Methods", "GET, POST, PUT, DELETE, OPTIONS");
    res.set_header("Access-Control-Allow-Headers", "Content-Type, Authorization");
    return res;
});
```

Browser จะส่ง **Preflight Request** (HTTP method `OPTIONS`) มาก่อนเสมอสำหรับ request ที่ไม่ใช่
`GET`/`POST` แบบง่าย (เช่น มี header `Content-Type: application/json`) เพื่อถาม server ว่า
"อนุญาตให้ origin นี้เรียกด้วย method นี้ พร้อม header เหล่านี้หรือไม่" ก่อนจะส่ง request จริง
ตามมา ในบทเรียนนี้เราทดสอบด้วย `curl` เป็นหลักซึ่งไม่ผ่านกลไก CORS ของเบราว์เซอร์ (CORS เป็น
กลไกที่บังคับใช้ฝั่งเบราว์เซอร์เท่านั้น ไม่ใช่ข้อจำกัดของ HTTP เอง) แต่ผู้เรียนที่จะต่อ Task API
นี้เข้ากับเว็บ frontend จริงต้องเปิด CORS header ให้ถูกต้องเสมอ

---

## ข้อผิดพลาดที่พบบ่อย (Common Pitfalls)

1. **ต่อ string SQL ด้วยมือแทนที่จะใช้ Prepared Statement** — เช่นเขียน
   `"SELECT * FROM tasks WHERE title = '" + title + "'"` ตรงๆ เปิดช่องให้เกิด SQL Injection
   ทันที ต้องใช้ `sqlite3_bind_*` เสมอไม่มีข้อยกเว้น
2. **ลืมเช็ค `body.is_object()` ก่อนเรียก `body["field"]`** — ถ้า client ส่ง JSON ที่ valid
   แต่เป็น `[1,2,3]` หรือ `"hello"` (ไม่ใช่ object) การเรียก `body["title"]` ตรงๆ จะโยน
   exception ที่ไม่ได้ตั้งใจ ต้องตรวจ `is_object()` ก่อนเสมอ
3. **คืน `200 OK` พร้อม body ที่บอกว่า error** — ทำให้ HTTP client library และเครื่องมือ
   monitoring ทั้งหมดเข้าใจผิดว่า request สำเร็จ ต้องใช้ status code ที่ตรงกับความจริงเสมอ
4. **ไม่แยก error `code` กับ `message` ออกจากกัน** — ถ้าฝั่ง client ต้อง parse ข้อความ
   ภาษาไทยเพื่อตัดสินใจ logic (เช่น เช็คว่า string มีคำว่า "ไม่พบ" หรือไม่) โค้ดจะพังทันทีที่
   เปลี่ยนคำ ต้องใช้ `code` (string คงที่) สำหรับ logic และ `message` สำหรับแสดงผลเท่านั้น
5. **ยัดทุกอย่างไว้ใน `main.cpp` ไฟล์เดียว** — ใช้ได้กับ demo 50 บรรทัด แต่พอ resource เพิ่ม
   เป็น 5-10 ตัว ไฟล์จะยาวหลายพันบรรทัดจนแก้ไขแล้วเสี่ยงพังทั้งระบบ ต้องแยกเป็น layer ตั้งแต่
   เริ่มโปรเจกต์
6. **ไม่ปิด SQLite connection อย่างถูกต้อง** — ถ้าลืมเรียก `sqlite3_close()` ใน destructor
   (หรือลืมเขียน destructor เลย) จะเกิด resource leak ทุกครั้งที่สร้าง `Db` object ใหม่ ควร
   ใช้หลัก RAII เสมอ (สร้าง connection ใน constructor ปิดใน destructor)
7. **ไม่ validate ชนิดข้อมูลของ field ที่ผู้ใช้ส่งมา ตรวจแต่ว่ามี key อยู่หรือไม่** — เช่น
   ตรวจว่ามี `"done"` แต่ไม่ตรวจว่าเป็น boolean จริง ถ้าผู้ใช้ส่ง `"done": "yes"` มา
   `body["done"].get<bool>()` จะโยน `type_error` exception ทันที ต้องตรวจ `is_boolean()`
   ก่อนดึงค่าเสมอ
8. **แชร์ connection/object ที่ไม่ thread-safe ข้าม thread โดยไม่ตรวจสอบก่อน** — Crow ทำงาน
   แบบ multithreaded โดย default (`.multithreaded()`) การใช้ตัวแปร global หรือ object ที่ไม่มี
   การป้องกัน (เช่น `std::vector` ธรรมดาที่ไม่มี mutex) ร่วมกันข้าม request ที่มาพร้อมกันจะทำให้
   เกิด Data Race ทันที (โปรเจกต์นี้ปลอดภัยเพราะ SQLite serialized mode จัดการให้ แต่ถ้าเปลี่ยน
   ไปเก็บข้อมูลใน container ของ C++ เองต้องเพิ่ม `std::mutex` ป้องกันเองเสมอ)
9. **ไม่จำกัดขนาด request body** — ถ้าไม่ตั้งค่า limit ผู้ใช้ที่ประสงค์ร้ายสามารถส่ง JSON body
   ขนาดหลาย GB มาทำให้ server หน่วยความจำเต็มจนล่มได้ (Denial of Service) Crow มี
   `app.bindaddr(...)` และการตั้งค่า `CROW_MAX_PAYLOAD_SIZE` หรือใช้ reverse proxy อย่าง nginx
   (Part 112) จำกัดขนาด body ไว้เป็นด่านแรกเสมอ
10. **เข้าใจผิดว่า `PUT` ต้องบังคับส่งครบทุกฟิลด์เสมอ** — ตามสเปก HTTP ดั้งเดิม `PUT` หมายถึง
    "แทนที่ resource ทั้งก้อน" (จึงควรบังคับส่งครบทุกฟิลด์) แต่ในทางปฏิบัติ API ส่วนใหญ่ในโลก
    จริงใช้ `PUT` แบบผ่อนปรนอนุญาต partial update เหมือนที่บทเรียนนี้ทำ ประเด็นสำคัญคือทีมต้อง
    ตกลงและเอกสารพฤติกรรมนี้ให้ตรงกันชัดเจน ไม่ใช่ปล่อยให้แต่ละ endpoint ตีความไม่เหมือนกัน

---

## แบบฝึกหัดท้ายบท

1. เพิ่ม endpoint ใหม่ `GET /tasks?done=true` (query parameter) ที่กรองเฉพาะ Task ที่ทำเสร็จ
   แล้ว และ `GET /tasks?done=false` สำหรับ Task ที่ยังไม่เสร็จ (ใช้ `req.url_params.get("done")`
   ของ Crow)
2. เพิ่มฟิลด์ `priority` เข้าไปใน Task (ค่าเป็น `"low"`, `"medium"`, `"high"` เท่านั้น) พร้อม
   validation ที่ปฏิเสธค่าอื่นที่ไม่อยู่ในสามค่านี้
3. เขียน endpoint ใหม่ `DELETE /tasks` (ไม่มี id) ที่ลบ Task ที่ `done = true` ทั้งหมดในครั้ง
   เดียว (bulk delete) และคืนจำนวนที่ถูกลบกลับไปใน response
4. เปลี่ยนจาก SQLite เป็นการเก็บข้อมูลใน `std::vector<Task>` ในหน่วยความจำล้วนๆ (ไม่ persist)
   แล้วสังเกตว่าต้องแก้โค้ดกี่ไฟล์ — คำตอบควรมีแค่ไฟล์เดียวคือ `db.hpp`/`db.cpp` ถ้าออกแบบชั้น
   ไว้ถูกต้อง
5. เพิ่ม pagination ให้ `GET /tasks` ด้วย query parameter `?page=1&limit=10` และคืนข้อมูล
   เพิ่มเติมใน response เช่น `total_count`, `total_pages`
6. เขียน endpoint `PATCH /tasks/<id>/done` ที่สลับสถานะ `done` ของ Task ไปมา (toggle) โดยไม่
   ต้องส่ง body มาเลย (ใช้ HTTP method `PATCH` ที่ยังไม่เคยใช้ในบทนี้)

### แนวทางเฉลยข้อ 1

เพิ่ม logic กรองใน handler ของ `GET /tasks` โดยอ่าน query parameter ผ่าน
`req.url_params.get("done")` ซึ่งคืนค่าเป็น `const char*` (เป็น `nullptr` ถ้าไม่มี parameter
นี้มา):

```cpp
CROW_ROUTE(app, "/tasks")
.methods(crow::HTTPMethod::GET)
([&db](const crow::request& req) {
    try {
        auto tasks = db.list_all();
        const char* done_param = req.url_params.get("done");

        json arr = json::array();
        for (const auto& t : tasks) {
            if (done_param != nullptr) {
                std::string filter(done_param);
                bool want_done = (filter == "true");
                if (t.done != want_done) continue; // ข้าม Task ที่ไม่ตรงเงื่อนไข
            }
            arr.push_back(t.to_json());
        }
        return api::ok(arr);
    } catch (const std::exception& e) {
        return api::error(500, "INTERNAL_ERROR", e.what());
    }
});
```

ทดสอบ:

```bash
$ curl -s "http://127.0.0.1:18180/tasks?done=true"
{"data":[{"...":"...", "done":true, ...}],"success":true}

$ curl -s "http://127.0.0.1:18180/tasks?done=false"
{"data":[{"...":"...", "done":false, ...}],"success":true}
```

การกรองแบบนี้ทำใน C++ หลังจากดึงข้อมูลทั้งหมดมาแล้ว (in-memory filter) ซึ่งเหมาะกับข้อมูล
จำนวนไม่มาก ถ้าข้อมูลมีเป็นแสน-ล้านแถว ควรย้าย logic การกรองไปที่ SQL query โดยตรง (เพิ่ม
`WHERE done = ?` ใน `db.cpp`) เพื่อให้ฐานข้อมูลทำหน้าที่กรองแทน ซึ่งเร็วกว่ามากเพราะมี Index
ช่วยได้

### แนวทางเฉลยข้อ 3

เพิ่มเมธอด `remove_completed()` ใน `Db` ที่รัน `DELETE FROM tasks WHERE done = 1;` แล้วคืน
จำนวนแถวที่ถูกลบ:

```cpp
// เพิ่มใน db.hpp
int remove_completed();

// เพิ่มใน db.cpp
int Db::remove_completed() {
    const char* sql = "DELETE FROM tasks WHERE done = 1;";
    char* err_msg = nullptr;
    int rc = sqlite3_exec(conn_, sql, nullptr, nullptr, &err_msg);
    if (rc != SQLITE_OK) {
        std::string msg = err_msg;
        sqlite3_free(err_msg);
        throw std::runtime_error(msg);
    }
    return sqlite3_changes(conn_);
}
```

Route handler:

```cpp
// DELETE /tasks — ลบ Task ที่ done = true ทั้งหมด (bulk delete)
CROW_ROUTE(app, "/tasks")
.methods(crow::HTTPMethod::DELETE)
([&db]() {
    try {
        int deleted_count = db.remove_completed();
        return api::ok(json{{"deleted_count", deleted_count}});
    } catch (const std::exception& e) {
        return api::error(500, "INTERNAL_ERROR", e.what());
    }
});
```

ทดสอบ (สมมติมี Task ที่ `done = true` อยู่ 3 รายการ):

```bash
$ curl -s -i -X DELETE http://127.0.0.1:18180/tasks
HTTP/1.1 200 OK
...
{"data":{"deleted_count":3},"success":true}
```

ข้อควรระวัง: route `DELETE /tasks` (ไม่มี id) กับ `DELETE /tasks/<int>` (มี id) เป็นคนละ route
กัน Crow จะแยก match ให้อัตโนมัติตาม path pattern จึงไม่ชนกัน แต่ผู้ออกแบบ API ต้องเอกสารให้
ชัดเจนว่าการเรียก `DELETE /tasks` แบบไม่มี id หมายถึง "ลบ Task ที่เสร็จแล้วทั้งหมด" ไม่ใช่
"ลบทุก Task ที่มีอยู่ในระบบ" เพราะเป็นการดำเนินการที่มีผลกระทบสูง (destructive operation)
ต้องระบุความหมายให้ชัดเจนเสมอ

### แนวทางเฉลยข้อ 2

เพิ่ม validation function เฉพาะสำหรับฟิลด์ `priority` ที่ยอมรับแค่ 3 ค่าที่กำหนดไว้ล่วงหน้า
(เทคนิคนี้เรียกว่า **Enum-like validation** — แม้ JSON จะไม่มี enum ในตัว แต่เราบังคับด้วย
`std::set` ของค่าที่อนุญาตแทน):

```cpp
#include <set>

static std::string validate_priority(const json& body) {
    static const std::set<std::string> allowed = {"low", "medium", "high"};
    if (body.contains("priority")) {
        if (!body["priority"].is_string()) {
            return "ฟิลด์ \"priority\" ต้องเป็น string";
        }
        std::string p = body["priority"].get<std::string>();
        if (allowed.find(p) == allowed.end()) {
            return "ฟิลด์ \"priority\" ต้องเป็นหนึ่งใน low, medium, high เท่านั้น";
        }
    }
    return "";
}
```

เรียกใช้ต่อจาก `validate_task_payload()` เดิมในทั้ง handler ของ `POST` และ `PUT` เพิ่มฟิลด์
`priority TEXT NOT NULL DEFAULT 'medium'` ในตาราง SQLite และเพิ่ม `priority` ใน `Task::to_json()`
ทดสอบจริง:

```bash
$ curl -s -i -X POST http://127.0.0.1:18381/check -d '{"priority":"high"}'
HTTP/1.1 200 OK
...
{"data":{"priority":"high"},"success":true}

$ curl -s -i -X POST http://127.0.0.1:18381/check -d '{"priority":"urgent"}'
HTTP/1.1 400 Bad Request
...
{"error":{"code":"VALIDATION_ERROR","message":"ฟิลด์ \"priority\" ต้องเป็นหนึ่งใน low, medium, high เท่านั้น"},"success":false}
```

ค่า `"urgent"` ที่ไม่อยู่ในเซตที่อนุญาตถูกปฏิเสธด้วย `400 Bad Request` ทันที ตามที่ออกแบบไว้

### แนวทางเฉลยข้อ 6

`PATCH /tasks/<id>/done` ไม่ต้องรับ body เลย เพราะความหมายของมันคือ "สลับสถานะจากค่าปัจจุบัน"
ไม่ใช่ "ตั้งค่าใหม่" — อ่านค่าปัจจุบันจากฐานข้อมูลก่อน แล้วเขียนค่ากลับที่เป็นค่าตรงข้าม:

```cpp
// PATCH /tasks/<id>/toggle — สลับสถานะ done ไปมา ไม่ต้องส่ง body
CROW_ROUTE(app, "/tasks/<int>/toggle")
.methods(crow::HTTPMethod::PATCH)
([&db](int id) {
    auto existing = db.find_by_id(id);
    if (!existing) {
        return api::error(404, "TASK_NOT_FOUND",
            "ไม่พบ Task ที่มี id = " + std::to_string(id));
    }
    bool new_done = !existing->done; // สลับค่าจาก true เป็น false หรือ false เป็น true
    db.update(id, existing->title, existing->description, new_done);
    auto updated = db.find_by_id(id);
    return api::ok(updated->to_json());
});
```

ทดสอบจริง (สร้าง Task id=1 ด้วย `done=false` ไว้ก่อน):

```bash
$ curl -s -i -X PATCH http://127.0.0.1:18380/tasks/1/toggle
HTTP/1.1 200 OK
...
{"data":{"created_at":"2026-09-26 09:24:59","description":"desc","done":true,"id":1,"title":"Test Task"},"success":true}

$ curl -s -i -X PATCH http://127.0.0.1:18380/tasks/1/toggle
HTTP/1.1 200 OK
...
{"data":{"created_at":"2026-09-26 09:24:59","description":"desc","done":false,"id":1,"title":"Test Task"},"success":true}

$ curl -s -i -X PATCH http://127.0.0.1:18380/tasks/999/toggle
HTTP/1.1 404 Not Found
...
{"error":{"code":"TASK_NOT_FOUND","message":"ไม่พบ Task ที่มี id = 999"},"success":false}
```

ครั้งแรก `done` สลับจาก `false` เป็น `true` ครั้งที่สองสลับกลับเป็น `false` — สังเกตว่า
endpoint นี้ **ไม่ idempotent** ตามตารางในหัวข้อ 108.2 (เรียกซ้ำ 2 ครั้งได้ผลต่างจากเรียก
ครั้งเดียว) ซึ่งเป็นตัวอย่างที่ดีว่าทำไมต้องเข้าใจ idempotency ก่อนออกแบบ endpoint ใหม่ๆ
เสมอ ไม่ใช่ทุก endpoint ที่ควร idempotent

---

## สรุปท้ายบท

ใน Part นี้เราได้:

- ออกแบบ REST API ครบวงจรสำหรับ resource "Task" ตามหลัก CRUD ทั้ง 5 endpoint พร้อม HTTP
  method และ status code ที่ถูกต้อง
- ออกแบบและ implement JSON Response Envelope ที่สม่ำเสมอทั้ง success และ error case
- จัดโครงสร้างโปรเจกต์แบบ Layered Architecture แยก Model, Data Access Layer, Response
  Helper, Routes และ Entry point ออกจากกันอย่างชัดเจน
- เขียน Data Access Layer ที่คุยกับ SQLite ผ่าน Prepared Statement ป้องกัน SQL Injection
  100%
- เขียน Input Validation ที่ตรวจทั้งชนิดข้อมูลและขอบเขตค่าก่อนบันทึกลงฐานข้อมูล
- ตั้งค่า CMake สำหรับโปรเจกต์หลายไฟล์ที่เชื่อมกับ SQLite3 และ Crow (header-only)
- ทดสอบทุก endpoint จริงด้วย `curl` ครบทั้ง success case และ error case (400, 404) พร้อม
  ผลลัพธ์จริงจากเซิร์ฟเวอร์ที่รันอยู่

โปรเจกต์นี้คือรากฐานสำคัญที่จะถูกต่อยอดต่อไปอีก 2 Part ข้างหน้า — **Part 109** จะเพิ่ม
ความสามารถแบบ real-time ผ่าน **WebSocket** (การสื่อสารสองทางที่ไม่ต้องรอ client มา request
ก่อน) และ **Part 110** จะเพิ่ม **Authentication** เข้าไปในระบบ Task API ตัวเดียวกันนี้ ทำให้
เฉพาะผู้ใช้ที่ login แล้วเท่านั้นถึงจะเข้าถึง endpoint ต่างๆ ได้

**ต่อไป:** [Part 109 — WebSocket Programming ด้วย C++](./part-109-websocket-cpp.md)
