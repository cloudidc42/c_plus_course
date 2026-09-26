# Part 103: Crow ขั้นสูง: Middleware, Routing, JSON (Step 817–824)

> Module I — Web Development ด้วย C/C++ | Part 103 จาก 125
> Step ที่ครอบคลุมใน Part นี้: Step 817–824
> Part ก่อนหน้า: [Part 102 — Crow Framework: พื้นฐานและ REST API แรก](./part-102-crow-basics.md) | Part ถัดไป: [Part 104 — Pistache Framework: ทางเลือกสำหรับ REST API ระดับ Production](./part-104-pistache.md)

## เป้าหมายการเรียนรู้ (Learning Objectives)

หลังจบ Part นี้ ผู้เรียนจะสามารถ:

1. อธิบายได้ว่า Middleware คืออะไร ทำงานตรงไหนใน Request/Response Lifecycle ของ Crow
2. เขียน Custom Middleware ของตัวเองได้ (เช่น Logging Middleware ที่จับเวลาการประมวลผล)
3. เขียน CORS Middleware อย่างง่ายเพื่อให้ frontend จาก origin อื่นเรียก API ได้
4. Parse JSON ที่ส่งมาใน Request Body ด้วย `crow::json::rvalue` ได้อย่างปลอดภัย พร้อมตรวจสอบ
   ความถูกต้องของข้อมูลก่อนใช้งาน
5. สร้าง Custom Error Response สำหรับ 404 (ไม่พบ endpoint) และ 500 (server error จาก exception)
   ได้
6. ใช้ `crow::Blueprint` เพื่อจัดกลุ่ม route ที่เกี่ยวข้องกันภายใต้ prefix เดียวกัน
7. ปรับจำนวน thread ที่ Crow ใช้ทำงานผ่าน `.concurrency()` และเข้าใจผลกระทบต่อ Throughput
8. สร้าง REST API ที่มีหลาย endpoint ทำงานร่วมกับ in-memory data store (`std::vector` ของ
   struct) โดยป้องกัน race condition ด้วย `std::mutex` อย่างถูกต้อง

---

## 103.1 Middleware คืออะไร (Step 817)

**Middleware** คือ logic ที่ Crow รันแทรกอยู่ **ก่อน** และ **หลัง** handler ของทุก route
โดยอัตโนมัติ ไม่ว่า request จะเข้ามาที่ route ไหนก็ตาม (ยกเว้นจะปิดเฉพาะบาง route ด้วย
`CROW_MIDDLEWARES`) แนวคิดนี้เหมือนกับ Middleware ใน Express.js หรือ Flask's `before_request`/
`after_request` เพียงแต่ Crow ทำผ่าน C++ template ทำให้ตรวจสอบ type ได้ตอน compile-time

Lifecycle ของ request หนึ่งตัวเมื่อมี Middleware ผูกอยู่หลายตัว:

```
Request เข้ามา
   │
   ▼
Middleware A: before_handle()
   │
   ▼
Middleware B: before_handle()
   │
   ▼
Route Handler (โค้ดของเราที่เขียนใน CROW_ROUTE)
   │
   ▼
Middleware B: after_handle()
   │
   ▼
Middleware A: after_handle()
   │
   ▼
ส่ง Response กลับไปหา Client
```

สังเกตว่า `after_handle` รันย้อนลำดับกับ `before_handle` (คล้ายโครงสร้าง Stack) — Middleware
ที่ประกาศก่อนจะได้ "ห่อหุ้ม" ตัวที่ประกาศทีหลัง

Middleware ที่ใช้บ่อยในงานจริง:

| ประเภท | ตัวอย่างการใช้งาน |
|---|---|
| Logging | บันทึกทุก request/response พร้อมเวลาที่ใช้ประมวลผล |
| CORS | เติม header `Access-Control-Allow-Origin` ให้ทุก response |
| Authentication | เช็ค token/session ก่อนปล่อยให้ handler ทำงาน |
| Rate Limiting | นับจำนวน request ต่อ IP แล้วบล็อกถ้าเกิน threshold |
| Compression | บีบอัด response body ด้วย gzip |

---

## 103.2 เขียน Custom Middleware (Step 818)

Middleware ของ Crow คือ `struct`/`class` ที่มีโครงสร้างตายตัว 3 ส่วน:

1. `struct context { ... };` — ที่เก็บข้อมูลที่ต้องการส่งต่อจาก `before_handle` ไปยัง
   `after_handle` (เช่น เวลาเริ่มต้น)
2. `void before_handle(crow::request& req, crow::response& res, context& ctx)`
3. `void after_handle(crow::request& req, crow::response& res, context& ctx)`

มาเขียน **Logging Middleware** ที่จับเวลาว่าแต่ละ request ใช้เวลาประมวลผลกี่มิลลิวินาที:

```cpp
#include "crow.h"
#include <chrono>

struct LoggingMiddleware {
    struct context {
        std::chrono::steady_clock::time_point start;
    };

    void before_handle(crow::request& req, crow::response&, context& ctx) {
        ctx.start = std::chrono::steady_clock::now();
        CROW_LOG_INFO << ">> เริ่มรับ request: "
                      << crow::method_name(req.method) << " " << req.url;
    }

    void after_handle(crow::request&, crow::response& res, context& ctx) {
        auto end = std::chrono::steady_clock::now();
        auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(
                      end - ctx.start).count();
        CROW_LOG_INFO << "<< ตอบกลับ status=" << res.code
                      << " ใช้เวลา=" << ms << "ms";
    }
};
```

จุดที่ต้องระวัง (และเป็นสิ่งที่พบตอนทดสอบจริงในเครื่อง): Crow เวอร์ชันที่ใช้ในบทเรียนนี้
**ไม่มีเมธอด `req.method_string()`** อย่างที่บาง tutorial เก่าเขียนไว้ — ต้องใช้ฟังก์ชัน
`crow::method_name(req.method)` แทน (รับ `crow::HTTPMethod` แล้วคืนชื่อ method เป็น
`std::string`) ถ้าเขียนผิดจะเจอ compile error ทันที:

```
error: 'struct crow::request' has no member named 'method_string'
```

เพื่อผูก Middleware เข้ากับ App ต้องประกาศ type ของ App ให้รู้จัก Middleware list ผ่าน
`crow::App<Middleware1, Middleware2, ...>` แทนที่จะใช้ `crow::SimpleApp` แบบ Part 102:

```cpp
crow::App<LoggingMiddleware> app;
```

---

## 103.3 CORS Middleware อย่างง่าย (Step 819)

**CORS (Cross-Origin Resource Sharing)** คือกลไกความปลอดภัยของ browser ที่บล็อก JavaScript
จาก origin หนึ่ง (เช่น `http://localhost:3000`) ไม่ให้เรียก API จาก origin อื่น (เช่น
`http://localhost:18083`) เว้นแต่ server จะอนุญาตผ่าน header `Access-Control-Allow-Origin`
อย่างชัดเจน มาเขียน Middleware ง่ายๆ ที่เติม header นี้ให้ทุก response:

```cpp
struct CorsMiddleware {
    struct context {};
    void before_handle(crow::request&, crow::response&, context&) {}
    void after_handle(crow::request&, crow::response& res, context&) {
        res.add_header("Access-Control-Allow-Origin", "*");
    }
};
```

> Crow เองก็มี middleware สำเร็จรูปชื่อ `crow::CORSHandler` อยู่ใน
> `crow/middlewares/cors.h` ที่ยืดหยุ่นกว่านี้มาก (กำหนด origin เฉพาะ, กำหนด method ที่อนุญาต,
> จัดการ preflight `OPTIONS` request ให้อัตโนมัติ) แต่การเขียน Middleware เองแบบข้างต้นช่วยให้
> เข้าใจกลไกเบื้องหลังก่อนไปใช้ของสำเร็จรูปในงานจริง

### รวม Middleware สองตัวเข้าด้วยกันและทดสอบจริง

```cpp
#include "crow.h"
#include <chrono>

struct LoggingMiddleware {
    struct context {
        std::chrono::steady_clock::time_point start;
    };

    void before_handle(crow::request& req, crow::response&, context& ctx) {
        ctx.start = std::chrono::steady_clock::now();
        CROW_LOG_INFO << ">> เริ่มรับ request: "
                      << crow::method_name(req.method) << " " << req.url;
    }

    void after_handle(crow::request&, crow::response& res, context& ctx) {
        auto end = std::chrono::steady_clock::now();
        auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(
                      end - ctx.start).count();
        CROW_LOG_INFO << "<< ตอบกลับ status=" << res.code
                      << " ใช้เวลา=" << ms << "ms";
    }
};

struct CorsMiddleware {
    struct context {};
    void before_handle(crow::request&, crow::response&, context&) {}
    void after_handle(crow::request&, crow::response& res, context&) {
        res.add_header("Access-Control-Allow-Origin", "*");
    }
};

int main() {
    crow::App<LoggingMiddleware, CorsMiddleware> app;

    CROW_ROUTE(app, "/")([](){
        return "middleware demo";
    });

    // custom 404
    CROW_CATCHALL_ROUTE(app)
    ([](crow::response& res){
        res.code = 404;
        crow::json::wvalue err;
        err["error"] = "not_found";
        err["message"] = "ไม่พบ endpoint ที่ร้องขอ";
        res.body = err.dump();
        res.set_header("Content-Type", "application/json");
    });

    app.port(18083).run();
}
```

คอมไพล์และรันจริง:

```bash
g++ -std=c++17 -I/usr/local/include middleware.cpp -o middleware -lpthread
./middleware
```

ทดสอบ:

```bash
curl -s -i http://127.0.0.1:18083/
```

ผลลัพธ์จริง:

```
HTTP/1.1 200 OK
Access-Control-Allow-Origin: *
Content-Length: 15
Server: Crow/master
Connection: Keep-Alive

middleware demo
```

Log ฝั่ง server ที่แสดงว่า Middleware ทำงานจริงก่อนและหลัง handler:

```
(2026-09-26 09:12:11) [INFO    ] Request: 127.0.0.1:50452 ... GET /
(2026-09-26 09:12:11) [INFO    ] >> เริ่มรับ request: GET /
(2026-09-26 09:12:11) [INFO    ] Response: 0x55ab88fe5fb0 / 200 0
(2026-09-26 09:12:11) [INFO    ] << ตอบกลับ status=200 ใช้เวลา=0ms
```

และทดสอบ route ที่ไม่มีอยู่จริง (ครอบคลุมหัวข้อ Error Handling ในหัวข้อถัดไปไปในตัว):

```bash
curl -s -i http://127.0.0.1:18083/doesnotexist
```

ผลลัพธ์จริง:

```
HTTP/1.1 404 Not Found
Access-Control-Allow-Origin: *
Content-Type: application/json
Content-Length: 86
Server: Crow/master
Connection: Keep-Alive

{"message":"ไม่พบ endpoint ที่ร้องขอ","error":"not_found"}
```

สังเกตว่า `Access-Control-Allow-Origin: *` ถูกเติมเข้ามาแม้กระทั่งใน response 404 ที่มาจาก
`CROW_CATCHALL_ROUTE` — นี่คือพลังของ Middleware: มันครอบคลุมทุก response โดยไม่ต้องเขียนโค้ด
ซ้ำในทุก handler

---

## 103.4 crow::json::rvalue: Parse JSON Request Body (Step 820)

Part 102 เราคืนค่า JSON ด้วย `crow::json::wvalue` ("writable") ไปแล้ว ในหัวข้อนี้จะพูดถึงฝั่ง
ตรงข้าม: การ **อ่าน** JSON ที่ client ส่งมาใน POST body ด้วย `crow::json::rvalue`
("readable value")

```cpp
CROW_ROUTE(app, "/tasks").methods(crow::HTTPMethod::POST)
([](const crow::request& req){
    auto body = crow::json::load(req.body);
    if (!body) {
        return crow::response(400, "invalid json");
    }
    if (!body.has("title")) {
        return crow::response(422, "missing field: title");
    }
    std::string title = body["title"].s();
    // ... ใช้ title ต่อ ...
    return crow::response(201, "created");
});
```

ขั้นตอนสำคัญที่ต้องทำเสมอเมื่อ parse JSON จาก client:

1. `crow::json::load(req.body)` คืนค่า `crow::json::rvalue` — ถ้า string ที่ส่งมาไม่ใช่ JSON
   ที่ valid เลย ค่าที่ได้จะประเมินเป็น `false` เมื่อเช็คใน `if` (operator `bool` overload)
   ต้องเช็คก่อนเสมอ ไม่งั้นการเรียก `.has(...)` หรือ `[...]` บนค่าที่ไม่ valid จะเป็น
   Undefined Behavior
2. `body.has("title")` เช็คว่ามี field นี้อยู่จริงหรือไม่ ก่อนจะเรียก `body["title"]` ตรงๆ
   (ถ้าไม่เช็คแล้ว field ไม่มีจริง จะได้ error แบบ runtime ที่ debug ยาก)
3. `.s()` แปลงค่าเป็น `std::string` — Crow ยังมี `.i()` (int64_t), `.d()` (double),
   `.b()` (bool) สำหรับแปลงเป็นชนิดอื่น ถ้าชนิดจริงใน JSON ไม่ตรงกับที่เรียก จะ throw
   `std::runtime_error`

---

## 103.5 Error Handling: Custom 404/500 (Step 821)

เราเห็น Custom 404 ไปแล้วในหัวข้อ 103.3 (`CROW_CATCHALL_ROUTE`) มาดู Custom 500 กัน

### พฤติกรรม Default เมื่อไม่มี Custom Handler

ก่อนจะเขียน custom handler ลองดูก่อนว่า Crow ทำอะไรให้เราโดย default เมื่อ handler throw
exception โดยไม่มีการดักจับเพิ่มเติมเลย:

```cpp
CROW_ROUTE(app, "/crash")
([](){
    throw std::runtime_error("boom without custom handler!");
    return crow::response(200);
});
```

ทดสอบจริง:

```bash
curl -s -i http://127.0.0.1:18089/crash
```

```
HTTP/1.1 500 Internal Server Error
Content-Length: 27
Server: Crow/master
Connection: Keep-Alive

500 Internal Server Error
```

และ log ฝั่ง server แสดงข้อความ error พร้อมรายละเอียด exception จริงที่ throw ออกมา:

```
(2026-09-26 09:29:01) [ERROR   ] An uncaught exception occurred: boom without custom handler!
```

ข้อสังเกตสำคัญ: Crow **ไม่ปล่อยให้โปรแกรม crash ทั้งตัวเมื่อ handler throw exception**
(ต่างจากโปรแกรม C++ ทั่วไปที่ uncaught exception จะเรียก `std::terminate()` ทำให้โปรแกรมทั้ง
process ตายทันที) แต่ Crow ดักจับ exception ไว้ในระดับ connection handler แล้วแปลงเป็น
`500 Internal Server Error` ให้อัตโนมัติ พร้อม log รายละเอียดไว้ฝั่ง server — request อื่นๆ ที่
กำลังทำงานอยู่ใน thread อื่นยังคงทำงานต่อไปตามปกติไม่ได้รับผลกระทบ นี่คือคุณสมบัติสำคัญของ
Web Framework ที่ดี: **exception ใน request หนึ่งต้องไม่ทำให้ทั้ง server ล่ม**

อย่างไรก็ตาม body ของ response ที่ได้ (`"500 Internal Server Error"`) เป็นข้อความทั่วไปที่ไม่มี
ประโยชน์กับ client มากนัก (ไม่บอกว่าเกิดอะไรขึ้นจริง ไม่ใช่ JSON ที่ frontend จะ parse ต่อได้ง่าย)
จึงมักต้องเขียน custom `exception_handler` เพื่อควบคุม format ของ response ให้เหมาะกับ API ของ
เราเอง:

```cpp
#include "crow.h"

int main() {
    crow::SimpleApp app;

    CROW_ROUTE(app, "/crash")
    ([](){
        throw std::runtime_error("boom!");
        return crow::response(200);
    });

    // custom exception handler
    app.exception_handler([](crow::response& res){
        crow::json::wvalue err;
        err["error"] = "internal_server_error";
        try {
            throw;
        } catch (const std::exception& e) {
            err["message"] = e.what();
        }
        res.code = 500;
        res.body = err.dump();
        res.set_header("Content-Type", "application/json");
        res.end();
    });

    app.port(18085).run();
}
```

ทดสอบจริง:

```bash
curl -s -i http://127.0.0.1:18085/crash
```

ผลลัพธ์จริง:

```
HTTP/1.1 500 Internal Server Error
Content-Type: application/json
Content-Length: 51
Server: Crow/master
Connection: Keep-Alive

{"message":"boom!","error":"internal_server_error"}
```

สังเกตเทคนิคที่ใช้ใน `exception_handler`: การเรียก `throw;` **โดยไม่มี argument** ภายใน
`catch` block เปล่าที่ครอบด้วย `try` อีกชั้น เป็นวิธีมาตรฐานของ C++ ในการ "re-throw exception
ปัจจุบัน" เพื่อดักจับ type ที่แท้จริงของมันอีกครั้ง (`app.exception_handler` เรียก callback นี้
ตอนที่ยังอยู่ใน exception context เดิม การเขียนแบบนี้ทำให้เราดึง `.what()` ออกมาได้โดยไม่ต้อง
เปลี่ยน signature ของ `exception_handler` ให้ซับซ้อน)

> **ข้อควรระวังระดับ production**: การ echo `e.what()` กลับไปให้ client โดยตรง (เหมือน
> ตัวอย่างข้างบน) เหมาะสำหรับ debug เท่านั้น ในระบบจริงควรซ่อนรายละเอียด exception ภายใน
> ไม่ให้หลุดไปถึง client (เพราะอาจเปิดเผยโครงสร้างภายในของระบบให้ผู้ไม่หวังดี) แล้ว log
> รายละเอียดจริงไว้ฝั่ง server แทน

---

## 103.6 Blueprint: จัดกลุ่ม Route (Step 822)

เมื่อ API มีจำนวน endpoint มากขึ้น การประกาศทุก route ไว้ใน `main()` เดียวจะทำให้โค้ดอ่านยาก
`crow::Blueprint` ช่วยให้เรา **จัดกลุ่ม route ที่เกี่ยวข้องกัน** ไว้ภายใต้ prefix เดียวกัน
(คล้าย `Blueprint` ของ Flask หรือ `Router` ของ Express)

จุดที่ต้องระวังมาก (และเป็นสิ่งที่พบจากการทดสอบจริงในเครื่อง): **Blueprint ใช้ macro คนละตัวกับ
App ปกติ** ถ้าใช้ `CROW_ROUTE` กับ `Blueprint` โดยตรงจะเจอ compile error:

```cpp
crow::Blueprint api("api", "api");

CROW_ROUTE(api, "/tasks").methods(crow::HTTPMethod::GET)   // ผิด!
(...)
```

```
error: 'class crow::Blueprint' has no member named 'route'
```

เหตุผลคือ `CROW_ROUTE` ถูก define ไว้ให้เรียก `.route<...>()` ของ `App` เท่านั้น ส่วน
`Blueprint` ต้องใช้ macro เฉพาะของมันคือ **`CROW_BP_ROUTE`**:

```cpp
CROW_BP_ROUTE(api, "/tasks").methods(crow::HTTPMethod::GET)   // ถูกต้อง
(...)
```

หลังประกาศ route ทั้งหมดใน Blueprint แล้ว ต้องนำไป register เข้ากับ app ด้วย
`app.register_blueprint(...)` โดย path ทุกอันภายใต้ Blueprint จะถูก mount ไว้ที่ prefix ตามชื่อ
ที่ตั้งไว้ตอนสร้าง (ในตัวอย่างนี้คือ `/api`)

### Blueprint แบบซ้อนกัน (Nested Blueprint)

Crow ยังรองรับการ register Blueprint หนึ่งเข้าไปใน Blueprint อีกตัวหนึ่ง ทำให้จัดกลุ่ม route
เป็นลำดับชั้นได้ เช่นการทำ API versioning (`/api/v1/...`):

```cpp
#include "crow.h"

int main() {
    crow::SimpleApp app;

    crow::Blueprint api("api");
    crow::Blueprint v1("v1");

    CROW_BP_ROUTE(v1, "/ping")
    ([](){ return "pong from v1"; });

    api.register_blueprint(v1);
    app.register_blueprint(api);

    app.port(18088).run();
}
```

ทดสอบจริง:

```bash
curl -s -i http://127.0.0.1:18088/api/v1/ping
```

```
HTTP/1.1 200 OK
Content-Length: 12
Server: Crow/master
Connection: Keep-Alive

pong from v1
```

สังเกตว่า path สุดท้ายกลายเป็น `/api/v1/ping` ตาม prefix ของทั้งสองชั้นต่อกัน (`api` +
`v1` + `/ping`) เทคนิคนี้มีประโยชน์มากเมื่อ API ของเราต้องรองรับหลายเวอร์ชันพร้อมกัน
(`/api/v1/...` และ `/api/v2/...`) โดยแยกโค้ดของแต่ละเวอร์ชันออกจากกันเป็นคนละไฟล์/คนละ
Blueprint ได้อย่างเป็นระเบียบ

นอกจากนี้ยังสังเกตได้ว่า Constructor ของ `Blueprint` รับ argument เดียว (`Blueprint("api")`)
ก็เพียงพอแล้ว ไม่จำเป็นต้องส่ง argument ที่สองซ้ำ (`Blueprint("api", "api")`) เหมือนตัวอย่าง
ก่อนหน้า เว้นแต่ต้องการกำหนด static file directory ที่ต่างจากชื่อ prefix โดยเฉพาะ

---

## 103.7 Multi-threading ของ Crow: concurrency() (Step 823)

Part 102 เราเห็น log บอกว่า Crow รันด้วย **2 threads** โดย default (ตามจำนวน core ที่ตรวจจับ
ได้) เราสามารถกำหนดจำนวน thread เองได้ด้วย `.concurrency(n)` ก่อนเรียก `.run()`:

```cpp
app.concurrency(4);
app.port(18084).run();
```

ทดสอบจริง log ที่ได้:

```
(2026-09-26 09:12:49) [INFO    ] Crow/master server is running at http://0.0.0.0:18084 using 4 threads
```

**ข้อควรเข้าใจให้ถูกต้อง**: จำนวน thread ที่มากขึ้นไม่ได้แปลว่าเร็วขึ้นเสมอไป — ถ้า handler ของ
เราทำงานแบบ CPU-bound (คำนวณหนัก) การเพิ่ม thread เกินจำนวน CPU core จริงจะทำให้เกิด context
switching overhead มากขึ้นแทนที่จะเร็วขึ้น ในทางกลับกัน ถ้า handler ส่วนใหญ่รอ I/O (เช่น query
database ระยะไกล) การมี thread มากพอจะช่วยให้ throughput สูงขึ้นเพราะ thread ที่ว่างสามารถรับ
request ใหม่ระหว่างที่ thread อื่นรอ I/O อยู่ กฎทั่วไปคือเริ่มจากค่า default
(`hardware_concurrency()`) แล้ว benchmark จริงก่อนปรับเปลี่ยน

จุดสำคัญที่สุดของหัวข้อนี้เชื่อมโยงไปสู่หัวข้อถัดไป: เพราะ handler ของแต่ละ request **อาจถูก
เรียกพร้อมกันจากหลาย thread ในเวลาเดียวกันจริงๆ** ถ้า handler เหล่านั้นเข้าถึง shared state
(เช่นตัวแปร global หรือ `std::vector` ที่เก็บข้อมูลใน memory) โดยไม่มีการป้องกัน จะเกิด
**Race Condition** ทันที — นี่คือเหตุผลที่หัวข้อถัดไปต้องใช้ `std::mutex` อย่างเคร่งครัด

---

## 103.8 ตัวอย่างจริง: Task API พร้อม In-memory Data Store + Mutex (Step 824)

มาสร้าง REST API ที่สมบูรณ์แบบขึ้น: API จัดการ "Task" (todo item) ที่เก็บข้อมูลไว้ใน memory
(`std::vector<Task>`) รวมทุกหัวข้อที่เรียนมาในบทนี้เข้าด้วยกัน — Blueprint, JSON parsing,
Error handling, และ Thread-safety ด้วย mutex:

```cpp
#include "crow.h"
#include <vector>
#include <mutex>

struct Task {
    int id;
    std::string title;
    bool done;
};

std::vector<Task> g_tasks;
std::mutex g_mutex;
int g_next_id = 1;

int main() {
    crow::SimpleApp app;

    crow::Blueprint api("api", "api");

    // GET /api/tasks — คืนรายการ task ทั้งหมด
    CROW_BP_ROUTE(api, "/tasks").methods(crow::HTTPMethod::GET)
    ([](){
        std::lock_guard<std::mutex> lock(g_mutex);
        crow::json::wvalue result;
        std::vector<crow::json::wvalue> items;
        for (auto& t : g_tasks) {
            crow::json::wvalue item;
            item["id"] = t.id;
            item["title"] = t.title;
            item["done"] = t.done;
            items.push_back(std::move(item));
        }
        result["tasks"] = std::move(items);
        return result;
    });

    // POST /api/tasks — สร้าง task ใหม่
    CROW_BP_ROUTE(api, "/tasks").methods(crow::HTTPMethod::POST)
    ([](const crow::request& req){
        auto body = crow::json::load(req.body);
        if (!body) {
            return crow::response(400, "invalid json");
        }
        if (!body.has("title")) {
            return crow::response(422, "missing field: title");
        }
        Task t;
        {
            std::lock_guard<std::mutex> lock(g_mutex);
            t.id = g_next_id++;
            t.title = body["title"].s();
            t.done = false;
            g_tasks.push_back(t);
        }
        crow::json::wvalue result;
        result["id"] = t.id;
        result["title"] = t.title;
        result["done"] = t.done;
        return crow::response(201, result);
    });

    app.register_blueprint(api);

    CROW_ROUTE(app, "/")([](){ return "blueprint demo"; });

    app.concurrency(4);
    app.port(18084).run();
}
```

### ทดสอบจริงทีละ endpoint

```bash
g++ -std=c++17 -I/usr/local/include blueprint_json.cpp -o blueprint_json -lpthread
./blueprint_json
```

**1) เรียก root path**

```bash
curl -s http://127.0.0.1:18084/
# blueprint demo
```

**2) GET /api/tasks ตอนยังไม่มีข้อมูล**

```bash
curl -s -i http://127.0.0.1:18084/api/tasks
```

```
HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 12
Server: Crow/master
Connection: Keep-Alive

{"tasks":[]}
```

**3) POST /api/tasks สร้าง task ใหม่**

```bash
curl -s -i -X POST -H "Content-Type: application/json" \
     -d '{"title":"ซื้อของ"}' http://127.0.0.1:18084/api/tasks
```

```
HTTP/1.1 201 Created
Content-Type: application/json
Content-Length: 53
Server: Crow/master
Connection: Keep-Alive

{"done":false,"title":"ซื้อของ","id":1}
```

**4) POST /api/tasks โดยไม่ส่ง title (ต้อง reject)**

```bash
curl -s -i -X POST -H "Content-Type: application/json" \
     -d '{}' http://127.0.0.1:18084/api/tasks
```

```
HTTP/1.1 422 Unprocessable Entity
Content-Length: 20
Server: Crow/master
Connection: Keep-Alive

missing field: title
```

**5) POST /api/tasks ด้วย body ที่ไม่ใช่ JSON เลย**

```bash
curl -s -i -X POST -d 'not json at all' http://127.0.0.1:18084/api/tasks
```

```
HTTP/1.1 400 Bad Request
Content-Length: 12
Server: Crow/master
Connection: Keep-Alive

invalid json
```

**6) GET /api/tasks อีกครั้ง หลัง insert สำเร็จไป 1 รายการ**

```bash
curl -s http://127.0.0.1:18084/api/tasks
# {"tasks":[{"done":false,"title":"ซื้อของ","id":1}]}
```

ทุกกรณีข้างต้นคือผลลัพธ์จริงจากการรันเครื่องนี้ ยืนยันว่าลำดับการ validate ทำงานถูกต้องตามที่
ออกแบบไว้: เช็ค JSON valid ก่อน (400) แล้วค่อยเช็ค field ที่จำเป็น (422) ก่อนจะสร้างข้อมูลจริง

### ทำไมต้องมี std::mutex ตรงนี้

`g_tasks` และ `g_next_id` เป็นตัวแปร **global** ที่ handler ทุก request เข้าถึงร่วมกัน เมื่อ
`app.concurrency(4)` ทำให้มี worker thread 4 ตัวพร้อมกัน ถ้ามีสอง request ยิง `POST /api/tasks`
เข้ามาพร้อมกันในเวลาเดียวกันโดยไม่มี mutex ป้องกัน อาจเกิดเหตุการณ์นี้:

```
Thread 1: อ่านค่า g_next_id = 5
Thread 2: อ่านค่า g_next_id = 5   (ยังไม่ทันถูก increment)
Thread 1: t.id = 5, g_next_id++ → 6
Thread 2: t.id = 5, g_next_id++ → 6   (ได้ id ซ้ำกับ Thread 1!)
```

ผลคือ Task สองตัวได้ `id` เดียวกัน (ข้อมูลเสียหาย/Data Race) หรือแย่กว่านั้นคือ
`g_tasks.push_back(...)` จากสอง thread พร้อมกันอาจทำให้ internal state ของ `std::vector`
เสียหาย (เพราะ `push_back` ไม่ thread-safe เมื่อเรียกพร้อมกันจากหลาย thread) จนโปรแกรม crash
`std::lock_guard<std::mutex>` แก้ปัญหานี้โดยบังคับให้ทีละ thread เท่านั้นที่เข้าไปแก้ไข
`g_tasks`/`g_next_id` ได้ในแต่ละครั้ง (Critical Section) — เนื้อหาเรื่อง `std::mutex` และ
Race Condition แบบละเอียดอยู่ใน Part 82 หากต้องการทบทวนพื้นฐาน

---

## ตารางสรุป: Crow vs Raw Server เมื่อพูดถึง Concurrency

| ประเด็น | Raw Thread Pool (Part 101) | Crow |
|---|---|---|
| การสร้าง Thread Pool | เขียนเอง (queue + worker threads) | มีในตัว ปรับด้วย `.concurrency(n)` |
| ความรับผิดชอบเรื่อง Thread-safety ของ shared state | ยังคงเป็นหน้าที่ผู้เขียนโค้ด | **ยังคงเป็นหน้าที่ผู้เขียนโค้ดเหมือนเดิม** — framework ไม่ช่วยตรงนี้ |
| Handler ถูกเรียกจาก thread ไหน | thread ที่เรากำหนดเอง | thread pool ภายในของ asio ที่เรามองไม่เห็นโดยตรง |
| ผลกระทบถ้าลืม mutex | Race condition | Race condition (เหมือนกันทุกประการ) |

จุดที่ต้องเน้นย้ำ: **การใช้ Framework ไม่ได้ทำให้ปัญหาเรื่อง Concurrency หายไป** มันแค่ซ่อน
รายละเอียดของการสร้าง thread ไว้เท่านั้น ความรับผิดชอบเรื่อง thread-safety ของโค้ดที่เราเขียน
ยังคงอยู่ที่ตัวเราเสมอ

---

## ข้อผิดพลาดที่พบบ่อย (Common Pitfalls)

1. **ใช้ `CROW_ROUTE` กับ `Blueprint` แทนที่จะใช้ `CROW_BP_ROUTE`** — เป็น compile error ที่พบ
   บ่อยที่สุดตอนเริ่มใช้ Blueprint (`'class crow::Blueprint' has no member named 'route'`)
   ต้องจำให้ขึ้นใจว่า Blueprint ใช้ macro คนละตัว
2. **เข้าถึง shared state (global vector, global counter) จากหลาย handler โดยไม่ใช้ mutex**
   — เพราะ Crow เรียก handler จากหลาย worker thread พร้อมกันจริง ถ้าไม่ป้องกันจะเกิด Race
   Condition ที่ทำให้ข้อมูลเสียหายแบบสุ่ม (บางครั้งรันแล้วไม่มีปัญหา บางครั้งพัง — เป็น
   ลักษณะเฉพาะของ bug ประเภทนี้ที่ debug ยากมาก)
3. **ถือ mutex lock นานเกินจำเป็น** — เช่น lock ครอบคลุมการแปลง JSON ทั้งหมด (ซึ่งไม่ได้แตะ
   shared state) แทนที่จะ lock เฉพาะช่วงที่แก้ไข `g_tasks` จริงๆ ทำให้ throughput ลดลงเพราะ
   thread อื่นต้องรอนานโดยไม่จำเป็น
4. **ลืมเช็ค `crow::json::load()` ว่า valid ก่อนเรียก `.has()`/`[...]`** — ถ้า client ส่ง body
   ที่ parse ไม่ได้ (invalid JSON) แล้วเราไม่เช็คก่อนใช้งาน จะเป็น Undefined Behavior
5. **echo ข้อความ exception ดิบๆ กลับไปให้ client ใน production** — เปิดเผยรายละเอียดภายในของ
   ระบบให้ผู้ไม่หวังดี ควรแยก error message ที่ปลอดภัยสำหรับ client ออกจาก log รายละเอียดจริง
   ที่เก็บไว้ฝั่ง server เท่านั้น
6. **ลืมว่า Middleware `after_handle` รันย้อนลำดับกับ `before_handle`** — ถ้ามีหลาย Middleware
   ที่พึ่งพาลำดับการทำงานกัน (เช่น Middleware ที่ตรวจ auth ต้องรันก่อน Middleware ที่ log
   ข้อมูล user) ต้องเรียงลำดับใน `crow::App<...>` ให้ถูกต้อง

---

## แบบฝึกหัดท้ายบท

1. เขียน Middleware ที่นับจำนวน request ทั้งหมดที่เข้ามาตั้งแต่ server เริ่มทำงาน (เก็บใน
   `std::atomic<int>` แบบ global) แล้วเพิ่ม route `/stats` ที่คืนค่าจำนวนนั้นเป็น JSON
2. เพิ่ม endpoint `DELETE /api/tasks/<int>` เข้าไปในตัวอย่าง Task API ของหัวข้อ 103.8 ที่ลบ
   task ตาม id ที่ระบุ (ถ้าไม่พบ id ให้ตอบ 404) อย่าลืม lock mutex ก่อนแก้ไข `g_tasks`
3. เพิ่ม endpoint `PUT /api/tasks/<int>` ที่รับ JSON `{"done": true}` แล้วอัปเดตสถานะของ task
   ตาม id
4. เขียน Middleware ตรวจสอบ Authentication แบบง่าย: ถ้า request ไม่มี header `X-API-Key` ที่
   ตรงกับค่าที่กำหนดไว้ล่วงหน้า ให้ตอบกลับ `401 Unauthorized` ทันทีโดยไม่เรียก handler จริง
   (ใบ้: ใช้ `res.end()` ใน `before_handle` เพื่อหยุด pipeline)
5. ทดลองยิง `POST /api/tasks` พร้อมกันหลายๆ ครั้งด้วยเครื่องมือ เช่น `ab` (Apache Bench) หรือ
   loop ของ `curl` ใน shell script แล้วตรวจสอบว่า `id` ของแต่ละ task ที่สร้างขึ้นไม่ซ้ำกันเลย
   (พิสูจน์ว่า mutex ทำงานถูกต้อง)
6. ลองแยก route จากหัวข้อ 103.8 ออกเป็นสอง Blueprint (เช่น `/api/tasks` กับ `/api/users`)
   register ทั้งสองเข้ากับ app เดียวกัน แล้วตรวจสอบว่าทั้งสองกลุ่ม path ทำงานถูกต้อง

### แนวทางเฉลยข้อ 1

```cpp
#include "crow.h"
#include <atomic>

std::atomic<int> g_request_count{0};

struct CounterMiddleware {
    struct context {};
    void before_handle(crow::request&, crow::response&, context&) {
        g_request_count++;
    }
    void after_handle(crow::request&, crow::response&, context&) {}
};

int main() {
    crow::App<CounterMiddleware> app;

    CROW_ROUTE(app, "/")([](){
        return "hello";
    });

    CROW_ROUTE(app, "/stats")
    ([](){
        crow::json::wvalue result;
        result["total_requests"] = g_request_count.load();
        return result;
    });

    app.port(18092).run();
}
```

ทดสอบ:

```bash
g++ -std=c++17 -I/usr/local/include ex1.cpp -o ex1 -lpthread
./ex1 &
curl -s http://127.0.0.1:18092/ > /dev/null
curl -s http://127.0.0.1:18092/ > /dev/null
curl -s http://127.0.0.1:18092/stats
# {"total_requests":2}
```

ใช้ `std::atomic<int>` แทน `std::mutex` เพราะการ increment ค่าจำนวนเต็มตัวเดียวเป็นกรณีที่
`std::atomic` เหมาะสมและเร็วกว่า mutex มาก (ไม่มี critical section ที่ซับซ้อนเกินกว่าการ
บวกเลขค่าเดียว) — หลักการเลือกระหว่าง `atomic` กับ `mutex` คือ: ถ้าปกป้องแค่ตัวแปรเดี่ยว
ใช้ `atomic`, ถ้าต้องปกป้องหลายค่าที่ต้องแก้พร้อมกันแบบ atomic (เช่น `vector` + `counter`
ในตัวอย่าง 103.8) ต้องใช้ `mutex`

### แนวทางเฉลยข้อ 2

```cpp
#include "crow.h"
#include <vector>
#include <mutex>
#include <algorithm>

struct Task {
    int id;
    std::string title;
    bool done;
};

std::vector<Task> g_tasks;
std::mutex g_mutex;
int g_next_id = 1;

int main() {
    crow::SimpleApp app;
    crow::Blueprint api("api", "api");

    CROW_BP_ROUTE(api, "/tasks").methods(crow::HTTPMethod::POST)
    ([](const crow::request& req){
        auto body = crow::json::load(req.body);
        if (!body || !body.has("title")) {
            return crow::response(400, "invalid request");
        }
        Task t;
        {
            std::lock_guard<std::mutex> lock(g_mutex);
            t.id = g_next_id++;
            t.title = body["title"].s();
            t.done = false;
            g_tasks.push_back(t);
        }
        crow::json::wvalue result;
        result["id"] = t.id;
        return crow::response(201, result);
    });

    CROW_BP_ROUTE(api, "/tasks/<int>").methods(crow::HTTPMethod::DELETE)
    ([](int id){
        std::lock_guard<std::mutex> lock(g_mutex);
        auto it = std::find_if(g_tasks.begin(), g_tasks.end(),
                                [id](const Task& t){ return t.id == id; });
        if (it == g_tasks.end()) {
            return crow::response(404, "task not found");
        }
        g_tasks.erase(it);
        return crow::response(204, "");
    });

    app.register_blueprint(api);
    app.port(18093).run();
}
```

ทดสอบ:

```bash
g++ -std=c++17 -I/usr/local/include ex2.cpp -o ex2 -lpthread
./ex2 &
curl -s -X POST -H "Content-Type: application/json" \
     -d '{"title":"งานทดสอบ"}' http://127.0.0.1:18093/api/tasks
# {"id":1}

curl -s -i -X DELETE http://127.0.0.1:18093/api/tasks/1
# HTTP/1.1 204 No Content ...

curl -s -i -X DELETE http://127.0.0.1:18093/api/tasks/999
# HTTP/1.1 404 Not Found ...
# task not found
```

จุดสำคัญของเฉลยนี้: ใช้ `CROW_BP_ROUTE(api, "/tasks/<int>")` เพื่อรับ path parameter ภายใน
Blueprint ได้เหมือนกับ route ปกติทุกประการ (Blueprint ไม่ได้จำกัดความสามารถของ route parameter
แต่อย่างใด) และ lock mutex ครอบคลุมเฉพาะช่วง search + erase เท่านั้น ไม่ครอบคลุมการสร้าง
`crow::response` ซึ่งไม่แตะ shared state

---

## สรุปท้ายบท

ใน Part นี้เราได้:

- เข้าใจ Middleware Lifecycle ของ Crow (`before_handle` → handler → `after_handle`)
  และเขียน Custom Middleware สำหรับ Logging และ CORS ได้จริง
- แก้ปัญหา compile error จริงที่พบระหว่างพัฒนา (`method_string()` ไม่มีในเวอร์ชันนี้ ต้องใช้
  `crow::method_name()`)
- Parse JSON Request Body ด้วย `crow::json::rvalue` พร้อม validate ก่อนใช้งานทุกครั้ง
- สร้าง Custom Error Response ทั้ง 404 (ผ่าน `CROW_CATCHALL_ROUTE`) และ 500 (ผ่าน
  `app.exception_handler`)
- ใช้ `crow::Blueprint` จัดกลุ่ม route และแก้ปัญหาจริงที่พบว่าต้องใช้ `CROW_BP_ROUTE` แทน
  `CROW_ROUTE`
- ปรับจำนวน worker thread ด้วย `.concurrency(n)` และเข้าใจว่าทำไม thread-safety ยังคงเป็น
  ความรับผิดชอบของผู้เขียนโค้ดเสมอ ไม่ว่า framework จะจัดการ thread pool ให้แค่ไหนก็ตาม
- สร้าง REST API ที่สมบูรณ์ (Task API) ที่มีหลาย endpoint ทำงานร่วมกับ in-memory data store
  พร้อมป้องกัน Race Condition ด้วย `std::mutex` อย่างถูกต้อง และทดสอบจริงด้วย `curl` ครบทุก
  เส้นทาง (สำเร็จ, JSON ผิดรูปแบบ, field ขาดหาย)

ใน **Part 104** เราจะเปลี่ยนไปรู้จัก **Pistache** ซึ่งเป็นอีกหนึ่ง C++ REST Framework ที่ออกแบบ
มาเน้นประสิทธิภาพระดับ production มากกว่า Crow โดยใช้สถาปัตยกรรม async I/O ที่ต่างออกไป และไม่ใช่
header-only (ต้อง link library จริง) เราจะเปรียบเทียบ Crow กับ Pistache แบบตรงไปตรงมา และลอง
port endpoint บางส่วนจาก Task API ของ Part นี้ไปเขียนด้วย Pistache

**ต่อไป:** [Part 104 — Pistache Framework: ทางเลือกสำหรับ REST API ระดับ Production](./part-104-pistache.md)
