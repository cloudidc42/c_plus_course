# Part 124: Capstone 4 — สร้าง Web Framework ของตัวเองด้วย C++ ตั้งแต่ศูนย์ (Step 985–992)

> Module K — Capstone Projects และบทสรุป | Part 124 จาก 125
> Step ที่ครอบคลุมใน Part นี้: Step 985–992
> Part ก่อนหน้า: [Part 123 — Capstone 3: Distributed Key-Value Store](./part-123-capstone-distributed-kv.md) | Part ถัดไป: [Part 125 — บทสรุปหลักสูตร: Roadmap สู่ระดับ World-Class Engineer](./part-125-final-summary.md)

## เป้าหมายการเรียนรู้ (Learning Objectives)

หลังจบ Part นี้ ผู้เรียนจะสามารถ:

1. อธิบายได้ว่า Web Framework อย่าง Crow "ทำอะไรอยู่ข้างใน" กันแน่ โดยไม่ต้องเดา เพราะได้
   สร้างสิ่งที่ทำหน้าที่เดียวกันขึ้นมาเองตั้งแต่ศูนย์ ต่อยอดจาก Raw Socket Server (Part 100)
   และ Thread Pool (Part 101) โดยตรง
2. ออกแบบและ implement **Request/Response abstraction** ที่ parse HTTP request ดิบให้
   กลายเป็น object ที่ใช้งานสะดวก (header, query string, path parameter, JSON body)
3. ออกแบบและ implement **Router** ที่จับคู่ `(Method, Path)` เข้ากับ handler ที่ลงทะเบียนไว้
   รองรับ path parameter สไตล์ `:name` และแก้ปัญหา **route-matching ambiguity** ระหว่าง
   static route กับ parameter route ได้อย่างถูกต้อง
4. ออกแบบระบบ **Middleware Chain** ด้วย `std::function` และ Lambda (ต่อยอดจาก Part 56 และ
   Part 64 โดยตรง) ที่รองรับการ "หยุดสาย" กลางทางได้ (เช่นทำ Authentication)
5. ประกอบทุกชิ้นส่วน (Router + Middleware + Thread Pool + Raw Socket) เข้าเป็นคลาส `App`
   เดียวที่มี Fluent API แบบเดียวกับ Crow (`app.get(path, handler)`) ได้สำเร็จ
6. คอมไพล์ รัน และทดสอบ Framework ที่สร้างขึ้นเองด้วย `curl` จริง ครบทั้ง routing, path
   parameter, middleware (เห็น log ยิงจริง), และ JSON response
7. **สร้าง Task API เวอร์ชันย่อของ Part 108 ขึ้นใหม่ทั้งหมด** บน Framework ของตัวเอง (ไม่ใช้
   Crow เลยแม้แต่บรรทัดเดียว) เพื่อพิสูจน์ว่ามันใช้งานได้จริงแบบ end-to-end
8. วัดผล Benchmark เปรียบเทียบ Framework ของตัวเองกับ Crow จริง แล้ววิเคราะห์ trade-off ทาง
   วิศวกรรมได้อย่างตรงไปตรงมาว่า **ทำไมแทบไม่มีใครใช้ Framework แบบนี้ใน Production จริง**
   แม้จะสร้างได้สำเร็จก็ตาม
9. เชื่อมโยงบทเรียนทั้งหมดของ Part นี้กลับไปยังปรัชญาของ Part 99 ว่าทำไม "เข้าใจของที่อยู่
   ข้างใต้ Framework" ถึงทำให้เป็นวิศวกรที่แข็งแกร่งกว่า แม้จะไม่ได้ใช้สิ่งที่สร้างเองจริงจังก็ตาม

---

## 124.1 ทบทวนเป้าหมาย: ทำไม Capstone สุดท้ายต้องสร้าง Framework เอง (Step 985)

เดินทางมาถึงจุดนี้ ผู้เรียนผ่านทุกอย่างที่ Module I (Web Development ด้วย C/C++) เตรียมไว้ให้
ครบแล้ว:

- **Part 100–101**: สร้าง HTTP Server จาก Socket API ล้วนๆ ด้วยมือ ตั้งแต่ `parseRequestLine()`
  ไปจนถึง `ThreadPool` ที่ป้องกัน Race Condition ด้วย `std::mutex`/`std::condition_variable`
- **Part 102–103**: ใช้ **Crow** ทำ routing, path parameter, middleware, JSON response แบบ
  สำเร็จรูป เร็วและสะดวกกว่าการเขียนเองมาก
- **Part 104–111**: ต่อยอดด้วย Pistache, ฐานข้อมูล, JSON, REST API เต็มรูปแบบ (Part 108),
  WebSocket, Authentication, Server-side Rendering

คำถามที่ Capstone สุดท้ายนี้ตั้งใจถามคือ: **ถ้าเอา Crow ออกไปทั้งหมด เราจะสร้างสิ่งที่ทำหน้าที่
เดียวกันได้เองไหม?** นี่ไม่ใช่คำถามเชิงวิชาการเฉยๆ — มันคือแบบทดสอบที่จริงจังที่สุดว่าความรู้
เรื่อง Socket, HTTP Protocol, Template, Lambda, `std::function`, Thread Pool, และ JSON ที่เรียน
มาตลอด 123 Part ก่อนหน้านี้ **ประกอบร่างเข้าด้วยกันเป็นระบบที่ใช้งานได้จริง** หรือไม่

Part นี้จะสร้าง Framework ตัวจิ๋วชื่อ **"MicroWeb"** ที่มีความสามารถหลักตรงกับสิ่งที่ Crow ให้
เราใน Part 102–103 ทุกประการ (แม้จะย่อยง่ายกว่ามาก):

- Fluent route-registration API: `app.get("/users/:id", handler)`
- Router ที่จับคู่ path parameter ได้ (`:id`)
- Request/Response abstraction ที่ parse header, query string, JSON body ให้อัตโนมัติ
- Middleware chain (logging, authentication)
- Thread Pool ข้างใต้ (นำโค้ดจาก Part 101 มาต่อยอดตรงๆ ไม่ต้องเขียนใหม่)

แล้วปิดท้ายด้วยการ **สร้าง Task API เวอร์ชันย่อของ Part 108 ขึ้นใหม่บน Framework ของตัวเอง**
เพื่อพิสูจน์ว่ามันไม่ใช่แค่ของเล่นทดลอง แต่เป็นระบบที่รับ request จริง, route จริง, ตอบ JSON
จริง, ผ่าน `curl` จริงได้ทุกกรณี

### เชื่อมโยงกับปรัชญาของ Part 99

Part 99 (ภาพรวม Web Development ด้วย C/C++) วางกรอบความคิดไว้ตั้งแต่ต้นทางว่า: **การเข้าใจ
เบื้องหลังของเครื่องมือ ทำให้เราใช้เครื่องมือนั้นได้ดีขึ้นและ debug มันได้เมื่อมันพัง** Capstone
นี้คือบทพิสูจน์สุดท้ายของหลักการนั้น — หลังจบ Part นี้ เวลาเจอ Crow ทำงานแปลกๆ (เช่น route ไม่
match ตามที่คาด, middleware ทำงานผิดลำดับ, connection ค้าง) ผู้เรียนจะไม่มองมันเป็น "กล่องดำ"
อีกต่อไป เพราะรู้แล้วว่าข้างในกล่องนั้น (ไม่ว่าจะเป็น Crow, Express, Flask, หรือ framework ภาษา
ไหนก็ตาม) ล้วนทำสิ่งเดียวกันกับที่เราเพิ่งสร้างขึ้นเอง เพียงแต่ทำได้ครบถ้วนและ optimize มากกว่า
เพราะมีทีมวิศวกรจำนวนมากพัฒนามันมาหลายปี

### สถาปัตยกรรมของ MicroWeb

```
                         ┌─────────────────────────────────────────┐
                         │              App (ประกอบร่างทุกอย่าง)         │
                         │                                           │
  TCP Socket             │   accept() ──▶ ThreadPool.enqueue(fd)     │
  (Part 100-101) ────────┤                     │                     │
                         │                     ▼                     │
                         │           serviceClient(clientFd)          │
                         │                     │                     │
                         │   ┌─────────────────┴──────────────────┐  │
                         │   │        Pipeline การประมวลผล 1 Request │  │
                         │   │                                     │  │
                         │   │   [1] Router.resolve(method, path)  │  │
                         │   │        │ หา handler + path param     │  │
                         │   │        ▼                            │  │
                         │   │   [2] Middleware chain (ตามลำดับ)     │  │
                         │   │        │ ตัวไหน return false = หยุดสาย │  │
                         │   │        ▼                            │  │
                         │   │   [3] Handler (route ที่ match ได้)    │  │
                         │   │        │ เขียนผลลง Response           │  │
                         │   │        ▼                            │  │
                         │   │   [4] serialize(Response) -> bytes  │  │
                         │   └─────────────────────────────────────┘  │
                         │                     │                     │
                         └─────────────────────┼─────────────────────┘
                                                ▼
                                        send() กลับไปหา client

     Request ──▶ Router ──▶ Middleware chain ──▶ Handler ──▶ Response
```

สังเกตว่า pipeline นี้ยึด Router มาก่อน Middleware chain — เหตุผลของการออกแบบเลือกลำดับนี้จะ
อธิบายละเอียดในหัวข้อ 124.4

### โครงสร้างไฟล์ของ MicroWeb

```
microweb/
├── CMakeLists.txt
├── include/microweb/
│   ├── request.hpp      # Request: HTTP request ที่ parse แล้ว (header-only)
│   ├── response.hpp     # Response: HTTP response ที่ handler สร้าง (header-only)
│   ├── router.hpp        # Router: จับคู่ (method, path) -> handler (header-only)
│   ├── threadpool.hpp    # ThreadPool ทั่วไป ต่อยอดจาก Part 101 (header-only)
│   └── app.hpp           # App: ประกาศ interface (fluent API + run())
├── src/
│   └── app.cpp           # App: implementation จริง (parse HTTP, socket, pipeline)
└── demo/
    ├── hello.cpp          # โปรแกรมทดสอบแรก (124.6)
    └── main.cpp           # Task API เต็มรูปแบบบน MicroWeb (124.7)
```

การแบ่งเป็น header-only เกือบทั้งหมด (`request.hpp`, `response.hpp`, `router.hpp`,
`threadpool.hpp`) ยกเว้น `app.cpp` เดียวที่ต้อง compile แยก เป็นการออกแบบที่ตั้งใจเลียนแบบ
สไตล์ของ Crow เอง (Part 102.1: "Header-only ... `#include "crow.h"` ก็ใช้งานได้เลย") — ต่างกัน
ตรงที่ Crow เป็น header-only **ทั้งหมด 100%** ส่วน MicroWeb แยก `app.cpp` ออกมาเพื่อให้เห็น
ชัดเจนว่าส่วนที่ "คุยกับ Socket API ตรงๆ" (ซึ่งไม่ใช่ template และไม่จำเป็นต้องอยู่ใน header)
ควรถูกแยกออกมาเป็นไฟล์ implementation ปกติ

---

## 124.2 ออกแบบ Request และ Response Abstraction (Step 986)

### ปัญหาที่ต้องแก้: Part 100 parse ได้แค่ Request Line

จำได้จาก Part 100.2 ไหมว่า `parseRequestLine()` แยกได้แค่ `method`, `path`, `version` จาก
บรรทัดแรกของ request เท่านั้น — ไม่ได้แตะ Header เลยแม้แต่นิดเดียว (Part 100.2 บอกไว้ตรงๆ ว่า
"Parser นี้เป็นเวอร์ชันง่ายที่สุดเท่าที่จะเป็นไปได้ จงใจไม่ parse Header ทั้งหมด") ตอนนั้นเรา
ยังไม่ต้องใช้ Header เพราะ Server ยังไม่มี Route ที่ซับซ้อน แต่ Framework จริงต้องรองรับอย่าง
น้อย 4 อย่างที่ Part 100 ไม่มี:

1. **Header** (เช่น `Content-Type`, `X-API-Key` สำหรับ authentication)
2. **Query String** (`?page=2&limit=10`)
3. **Path Parameter** (`:id` จาก URL เช่น `/users/42`)
4. **Body ขนาดใหญ่ที่อาจมากกว่า 1 buffer ของ `recv()`** (เช่น JSON payload ของ POST/PUT)

`Request` และ `Response` คือสอง struct ที่ห่อหุ้มความซับซ้อนทั้งหมดนี้ไว้ ทำให้ handler ที่
ผู้ใช้ framework เขียน (เหมือน lambda ที่ส่งเข้า `CROW_ROUTE` ใน Part 102.3) ไม่ต้องแตะ raw
socket bytes เลยแม้แต่นิดเดียว

### `request.hpp`

```cpp
#pragma once
// request.hpp - Request: ตัวแทนของ HTTP Request ที่ parse แล้ว พร้อมใช้งาน
#include <algorithm>
#include <cctype>
#include <map>
#include <nlohmann/json.hpp>
#include <string>

namespace microweb {

// เก็บ header/query/param เป็น map ที่เทียบ key แบบ case-insensitive สำหรับ header
// (HTTP spec บอกว่าชื่อ header ไม่สนตัวพิมพ์เล็ก-ใหญ่ แต่ query/path param สนตัวพิมพ์ตามปกติ)
struct CaseInsensitiveLess {
    bool operator()(const std::string& a, const std::string& b) const {
        return std::lexicographical_compare(
            a.begin(), a.end(), b.begin(), b.end(),
            [](unsigned char x, unsigned char y) {
                return std::tolower(x) < std::tolower(y);
            });
    }
};

struct Request {
    std::string method;
    std::string path;          // path ล้วนๆ ไม่รวม query string เช่น "/users/42"
    std::string version;
    std::string body;

    std::map<std::string, std::string, CaseInsensitiveLess> headers;
    std::map<std::string, std::string> query;   // จาก "?key=value&..."
    std::map<std::string, std::string> params;  // จาก path parameter เช่น ":id"

    // route pattern ที่ router จับคู่ได้ (เช่น "/users/:id") ใช้สำหรับ logging middleware
    // เพื่อไม่ให้ log บวมด้วย path จริงที่มี id ต่างกันนับพันแบบ (ดูหัวข้อ 124.6)
    std::string route_pattern;

    bool valid = false;   // false = parse ไม่สำเร็จ (request line ผิดรูปแบบ) -> 400 ทันที

    // ดึงค่า header (คืน string ว่างถ้าไม่มี ไม่ throw ไม่คืน nullptr — ปลอดภัยกว่า
    // req.url_params.get() ของ Crow ที่ต้องเช็ค nullptr เองทุกครั้ง ดู Common Pitfalls ของ Part 102)
    std::string header(const std::string& key) const {
        auto it = headers.find(key);
        return it != headers.end() ? it->second : std::string();
    }

    std::string queryParam(const std::string& key, const std::string& def = "") const {
        auto it = query.find(key);
        return it != query.end() ? it->second : def;
    }

    std::string param(const std::string& key) const {
        auto it = params.find(key);
        return it != params.end() ? it->second : std::string();
    }

    // parse body เป็น JSON (throw nlohmann::json::parse_error ถ้า body ไม่ใช่ JSON ที่ถูกต้อง
    // — handler ที่เรียกต้องครอบ try/catch เอง หรือใช้ middleware ตรวจสอบล่วงหน้า)
    nlohmann::json json() const {
        return nlohmann::json::parse(body);
    }
};

}  // namespace microweb
```

**จุดออกแบบที่ตั้งใจต่างจาก Crow อย่างชัดเจน**: `header()`, `queryParam()`, และ `param()` คืนค่า
`std::string` ว่างเปล่าเสมอถ้าไม่พบ key นั้น **ไม่คืน `nullptr`** ต่างจาก `req.url_params.get()`
ของ Crow (Part 102.6) ที่คืน `const char*` และบังคับให้ผู้ใช้ต้องเช็ค `nullptr` เองทุกครั้งก่อน
เอาไปสร้าง `std::string` (Part 102's Common Pitfall ข้อ 2 บอกตรงๆ ว่านี่เป็นจุดที่มือใหม่ลืม
เช็คแล้ว segfault บ่อยที่สุด) การออกแบบ API ของเราเองเปิดโอกาสให้แก้ปัญหานี้ตั้งแต่ต้นทาง — นี่
คือประโยชน์ที่จับต้องได้ของการ "ออกแบบ API เอง" แทนที่จะก็อปพฤติกรรมของ library อื่นมาตรงๆ

### `response.hpp`

```cpp
#pragma once
// response.hpp - Response: ตัวแทนของ HTTP Response ที่ handler สร้างขึ้น
#include <nlohmann/json.hpp>
#include <string>
#include <vector>
#include <utility>

namespace microweb {

struct Response {
    int status = 200;
    std::vector<std::pair<std::string, std::string>> headers;  // เก็บลำดับที่ set ไว้ (ต่าง
                                                                 // จาก map ที่ลำดับไม่แน่นอน)
    std::string body;

    Response() {
        headers.emplace_back("Content-Type", "text/plain; charset=utf-8");
    }

    // แทนที่ header ถ้ามีอยู่แล้ว (case-sensitive พอสำหรับ header ที่เราตั้งเอง)
    void setHeader(const std::string& key, const std::string& value) {
        for (auto& h : headers) {
            if (h.first == key) {
                h.second = value;
                return;
            }
        }
        headers.emplace_back(key, value);
    }

    // ---- Fluent helper: เขียนต่อกันเป็นสายได้ เช่น res.status(201).json(body) ----
    Response& setStatus(int code) {
        status = code;
        return *this;
    }

    Response& text(const std::string& s) {
        body = s;
        setHeader("Content-Type", "text/plain; charset=utf-8");
        return *this;
    }

    Response& html(const std::string& s) {
        body = s;
        setHeader("Content-Type", "text/html; charset=utf-8");
        return *this;
    }

    Response& json(const nlohmann::json& j) {
        body = j.dump();
        setHeader("Content-Type", "application/json");
        return *this;
    }
};

}  // namespace microweb
```

สังเกตว่า `headers` เก็บเป็น `std::vector<std::pair<...>>` **ไม่ใช่** `std::map` — เพราะเรา
ต้องการรักษาลำดับที่ header ถูก `setHeader()` ไว้ (map เรียงตาม key ตามตัวอักษรเสมอ ทำให้ลำดับ
header ใน response ดิบไม่ตรงกับที่โปรแกรมเมอร์ตั้งใจ) นี่คือรายละเอียดเล็กๆ ที่ต่างจาก
`crow::json::wvalue` ซึ่ง Part 102.7 ค้นพบด้วยการทดสอบจริงว่า **ลำดับ key ใน JSON ของ Crow ไม่
การันตี** เพราะเก็บแบบ hash-based ภายใน — MicroWeb เลือกออกแบบให้ตรงข้ามในส่วนของ HTTP header
(รักษาลำดับ) แต่ยังคงพึ่งพา `nlohmann::json` (ซึ่งเรียงตาม key เหมือนกัน) สำหรับตัว body JSON
เอง — เป็นตัวอย่างที่ดีว่าการออกแบบ data structure ต้องเลือกตาม "สิ่งที่ต้องการการันตี" ของแต่
ละส่วน ไม่ใช่ใช้ container เดียวกันไปหมดโดยไม่คิด

### Response Envelope ไม่ได้ผูกไว้ใน Framework

สังเกตว่า `Response` **ไม่มี** field `success`/`error` แบบ envelope ของ Part 108.2 ฝังอยู่เลย
— นี่คือการตัดสินใจออกแบบที่ตั้งใจ: **Framework ควรเป็นกลาง (agnostic) ต่อรูปแบบ business
response** ผู้ใช้ framework (คือเราเองตอนสร้าง Task API ใน 124.7) เป็นคนกำหนด envelope นั้นเอง
ผ่านฟังก์ชันช่วยระดับแอปพลิเคชัน (`api::ok()`/`api::error()`) เหมือนที่ Part 108.2 ทำกับ Crow
— หลักการนี้ตรงกับที่ Crow เองก็ไม่ได้บังคับรูปแบบ JSON ใดๆ ให้ผู้ใช้ (Part 102.7): Framework
ให้เครื่องมือสร้าง JSON response ได้ แต่ไม่ตัดสินใจแทนว่า JSON นั้นต้องมีหน้าตาอย่างไร

---

## 124.3 Router: จับคู่ Path พร้อม Path Parameter (Step 987)

### แนวทางที่ MicroWeb เลือก: Runtime String Matching แบบ Express.js

Part 102.3 อธิบายไว้ว่า Crow ใช้ **Template Metaprogramming** วิเคราะห์ path pattern เช่น
`"/user/<int>"` ตอน **compile-time** ผ่าน macro `CROW_ROUTE` — ทำให้ type ของ parameter (`int`,
`string`, ...) ถูกตรวจสอบตั้งแต่ compile ไม่ใช่ runtime

MicroWeb เลือกแนวทางที่ **ง่ายกว่ามาก**: เก็บ path pattern เป็น `std::string` ธรรมดา (เช่น
`"/users/:id"`) แล้วแยก **segment** ด้วย `/` ตอน runtime เทียบกับ path จริงที่ client ส่งมา
ทีละ segment — พารามิเตอร์ทุกตัวได้เป็น `std::string` เสมอ (ไม่มีการแยก type เป็น `int`/
`double` แบบ Crow) ผู้ใช้ต้อง `std::atoi()`/`std::stod()` แปลง type เองถ้าต้องการ

| แง่มุม | Crow (`<int>`, compile-time) | MicroWeb (`:id`, runtime) |
|---|---|---|
| ตรวจ type parameter | Compile-time (ผิด type = compile error) | ไม่ตรวจเลย (ได้ `std::string` เสมอ) |
| ความซับซ้อนของ implementation | สูงมาก (ต้องเขียน Template Metaprogramming) | ต่ำมาก (แค่ `split()` + เทียบ string) |
| Error ที่ path ไม่ตรง type | ไม่ match route เลย (404) ตั้งแต่ Part 102.4 | route ยัง match แต่ตัวแปรงแปลง type เองอาจ throw/ได้ 0 |
| เวลาที่ใช้พัฒนา framework | มาก (สัปดาห์-เดือน สำหรับทีมมืออาชีพ) | น้อย (ไม่กี่ชั่วโมงสำหรับเวอร์ชันพื้นฐาน) |

การเลือกแนวทาง runtime ไม่ใช่เพราะ "ดีกว่า" — มันคือ trade-off ที่ตรงไปตรงมาระหว่าง **ความ
ปลอดภัยที่ตรวจสอบได้ตอน compile** กับ **ความง่ายในการ implement** ประเด็นนี้จะกลับมาอีกครั้ง
ในบทวิเคราะห์ trade-off ท้าย Part (หัวข้อ 124.8)

### ปัญหาที่ต้องแก้ให้ถูกต้อง: Route-Matching Ambiguity

ลองนึกภาพ 2 route ที่ลงทะเบียนไว้:

```cpp
app.get("/users/:id", handlerA);   // "id" คือ path parameter รับได้ทุกค่า
app.get("/users/me", handlerB);    // "me" คือ static path ตรงตัว
```

ถ้า client ยิง `GET /users/me` เข้ามา **ทั้งสอง route match ได้ทั้งคู่** ในทางเทคนิค — `:id`
รับค่าอะไรก็ได้รวมถึง `"me"` ด้วย คำถามคือ: **route ไหนควรชนะ?**

Router ที่ implement ไม่ดี (จับคู่ route แรกที่เจอตามลำดับที่ลงทะเบียนไว้ ซึ่งเป็นพฤติกรรม
default ของหลาย framework รวมถึง Express.js) จะทำให้ผลลัพธ์**ขึ้นกับลำดับที่เขียนโค้ด** — ถ้า
สลับลำดับ `.get()` สองบรรทัดข้างบน ผลลัพธ์จะเปลี่ยนทันทีโดยไม่มี error หรือ warning ใดๆ เตือน
เลย นี่คือบั๊กที่พบได้บ่อยมากในโปรเจกต์จริงที่มี route หลายสิบ-หลายร้อยเส้น เพราะคนเขียน route
ใหม่ทีหลังมักจะไม่รู้ว่าต้องเรียงลำดับก่อน-หลังกับ route เก่าที่มีอยู่แล้วอย่างไร

**MicroWeb แก้ปัญหานี้ด้วยระบบให้คะแนน (scoring)**: segment ที่ตรงตัวอักษร (literal) แบบ
`"me"` ได้คะแนนสูงกว่า segment ที่เป็น parameter แบบ `:id` เสมอ ไม่ว่า route ไหนจะลงทะเบียน
ก่อนหรือหลังก็ตาม — Router จะไล่ตรวจ**ทุก route ที่ match ได้** แล้วเลือก route ที่มีคะแนนรวม
สูงสุดเสมอ

### `router.hpp`

```cpp
#pragma once
// router.hpp - Router: จับคู่ (Method, Path) เข้ากับ Handler ที่ลงทะเบียนไว้
// รองรับ path parameter สไตล์ ":name" เช่น "/users/:id/orders/:orderId"
#include <functional>
#include <map>
#include <sstream>
#include <string>
#include <vector>

#include "microweb/request.hpp"
#include "microweb/response.hpp"

namespace microweb {

// Handler คือ "งานจริง" ที่ผูกกับ 1 route — รับ Request (อ่านอย่างเดียวพอ) แล้วเขียนผลลง
// Response ที่ส่งเข้ามาโดยอ้างอิง เราเลือก signature นี้ (ไม่ return Response ตรงๆ) เพราะ
// สอดคล้องกับ Middleware ด้านล่างซึ่งต้อง "แก้ไข" Response เดิมต่อกันเป็นสาย (chain) ได้
using Handler = std::function<void(const Request&, Response&)>;

// Middleware คืนค่า bool: true = "ทำงานต่อได้" (ไปที่ middleware ตัวถัดไป หรือ handler ถ้าเป็น
// ตัวสุดท้าย), false = "หยุดสายทันที" (ใช้ Response ที่ middleware ตัวนี้เตรียมไว้ตอบกลับเลย
// ไม่เรียก middleware/handler ที่เหลือ) — ใช้ทำ Auth middleware ที่ปฏิเสธ request ได้ตรงจุด
using Middleware = std::function<bool(Request&, Response&)>;

// แยก segment ของ path ด้วย '/' โดยไม่เก็บ segment ว่าง (กัน "//" หรือ path ที่ลงท้ายด้วย "/")
inline std::vector<std::string> splitPath(const std::string& path) {
    std::vector<std::string> segments;
    std::stringstream ss(path);
    std::string seg;
    while (std::getline(ss, seg, '/')) {
        if (!seg.empty()) segments.push_back(seg);
    }
    return segments;
}

struct RouteMatch {
    bool found = false;
    Handler handler;
    std::string pattern;  // path pattern ดิบที่ลงทะเบียนไว้ เช่น "/users/:id" (ไว้ log)
};

class Router {
public:
    // ลงทะเบียน route ใหม่ 1 เส้นสำหรับ (method, pattern) คู่หนึ่ง
    void add(const std::string& method, const std::string& pattern, Handler handler) {
        Route r;
        r.method = method;
        r.pattern = pattern;
        r.segments = splitPath(pattern);
        r.handler = std::move(handler);
        routes_.push_back(std::move(r));
    }

    // จับคู่ path จริงที่ client ส่งมากับ route ที่ลงทะเบียนไว้ทั้งหมด
    //
    // กติกาสำคัญ (แก้ปัญหา "route-matching ambiguity" — ดู Common Pitfalls 124.9):
    // ถ้ามีมากกว่า 1 route ที่ match path เดียวกันได้ (เช่น "/users/me" แบบ static เทียบกับ
    // "/users/:id" แบบ parameter) เราจะเลือก route ที่มี "segment ตรงตัวอักษร (literal)"
    // มากที่สุดเสมอ ไม่ใช่ route แรกที่ลงทะเบียนไว้ — ทำให้ลำดับการเขียน .get(...) ในโค้ด
    // ไม่ส่งผลต่อผลการ routing เลย (ต่างจาก Express.js ที่ "ใครลงทะเบียนก่อนชนะก่อน" ซึ่งเป็น
    // บ่อเกิดบั๊กที่พบบ่อยมากถ้าคนเขียนโค้ดเผลอสลับลำดับ route แบบ static/param)
    RouteMatch resolve(const std::string& method, const std::string& path,
                        std::map<std::string, std::string>& outParams) const {
        std::vector<std::string> reqSegments = splitPath(path);

        const Route* best = nullptr;
        int bestScore = -1;
        std::map<std::string, std::string> bestParams;

        for (const auto& r : routes_) {
            if (r.method != method) continue;
            if (r.segments.size() != reqSegments.size()) continue;

            int score = 0;
            std::map<std::string, std::string> params;
            bool ok = true;
            for (std::size_t i = 0; i < r.segments.size(); ++i) {
                const std::string& pat = r.segments[i];
                const std::string& real = reqSegments[i];
                if (!pat.empty() && pat[0] == ':') {
                    params[pat.substr(1)] = real;
                    score += 1;  // param match ได้คะแนนน้อยกว่า literal match
                } else if (pat == real) {
                    score += 10;  // literal match ตรงเป๊ะ ได้คะแนนเยอะกว่าเสมอ
                } else {
                    ok = false;
                    break;
                }
            }
            if (!ok) continue;

            if (score > bestScore) {
                bestScore = score;
                best = &r;
                bestParams = std::move(params);
            }
        }

        RouteMatch result;
        if (best != nullptr) {
            result.found = true;
            result.handler = best->handler;
            result.pattern = best->pattern;
            outParams = std::move(bestParams);
        }
        return result;
    }

    std::size_t routeCount() const { return routes_.size(); }

private:
    struct Route {
        std::string method;
        std::string pattern;
        std::vector<std::string> segments;
        Handler handler;
    };

    std::vector<Route> routes_;
};

}  // namespace microweb
```

จุดสำคัญที่ต้องเข้าใจในอัลกอริทึม `resolve()`:

1. **กรองด้วยจำนวน segment ก่อน** (`r.segments.size() != reqSegments.size()`) — `/users/:id`
   (2 segment) จะไม่มีวัน match กับ `/users/1/orders` (3 segment) เลย ไม่ว่ากรณีใด ทำให้ตัด
   route ที่ไม่เกี่ยวข้องออกไปได้เร็วตั้งแต่ต้น
2. **ให้คะแนนทีละ segment**: literal match = 10 คะแนน, parameter match = 1 คะแนน — ตัวเลข
   เหล่านี้เลือกให้ต่างกันมากพอที่ literal segment แม้แค่ 1 ตำแหน่งก็ชนะ parameter segment ได้
   หลายตำแหน่งรวมกัน (เช่น `/a/:b/:c` ให้คะแนน 1+1=2 ในขณะที่ `/a/x/:c` ให้คะแนน 10+1=11 —
   route หลังชนะเสมอถ้า path จริงคือ `/a/x/y`)
3. **เลือก route คะแนนสูงสุดจาก route ที่ match ได้ทั้งหมด** ไม่ใช่ route แรกที่เจอ — แก้ปัญหา
   ambiguity ตามที่อธิบายไว้ข้างต้นได้อย่างสมบูรณ์

### ทดสอบจริง: พิสูจน์ว่า Ambiguity ถูกแก้แล้ว

เขียนโปรแกรมทดสอบเล็กๆ ที่ **จงใจลงทะเบียน `/users/:id` (parameter) ก่อน `/users/me`
(static)** เพื่อพิสูจน์ว่าลำดับการลงทะเบียนไม่มีผลต่อผลลัพธ์เลย:

```cpp
// ambiguity_demo.cpp - สาธิต route-matching ambiguity: static route ("/users/me") ปะทะ
// parameter route ("/users/:id") ควรจะ match "/users/me" เข้ากับ static route เสมอ
// ไม่ว่าจะลงทะเบียน route ไหนก่อนก็ตาม
#include <cstdio>
#include <csignal>
#include "microweb/app.hpp"

using microweb::App;
using microweb::Request;
using microweb::Response;

App* g_app = nullptr;
void onSigint(int) { if (g_app) g_app->requestStop(); }

int main() {
    App app;

    // จงใจลงทะเบียน "/users/:id" (parameter) ก่อน "/users/me" (static) เพื่อพิสูจน์ว่า
    // ลำดับการลงทะเบียนไม่มีผลต่อผลการ routing เลย
    app.get("/users/:id", [](const Request& req, Response& res) {
        res.json({{"matched", "param-route"}, {"id", req.param("id")}});
    });
    app.get("/users/me", [](const Request&, Response& res) {
        res.json({{"matched", "static-route"}});
    });

    g_app = &app;
    std::signal(SIGINT, onSigint);
    app.run(8125, 2);
    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 -pthread -O2 -Iinclude \
    demo/ambiguity_demo.cpp src/app.cpp -o ambiguity_demo
./ambiguity_demo &
curl -s http://127.0.0.1:8125/users/me
curl -s http://127.0.0.1:8125/users/42
```

ผลลัพธ์จริงจากการทดสอบ:

```
{"matched":"static-route"}
{"id":"42","matched":"param-route"}
```

`GET /users/me` match เข้ากับ **static route** แม้จะลงทะเบียน parameter route ไว้ก่อนก็ตาม
ส่วน `GET /users/42` (ที่ไม่ตรงกับ `"me"` แบบ literal) จึง fallback ไปที่ parameter route ได้
ถูกต้อง — พิสูจน์ว่าระบบให้คะแนนทำงานตามที่ออกแบบไว้ 100%

---

## 124.4 ออกแบบ Middleware Chain ด้วย `std::function` (Step 988)

### ทำไมต้องใช้ `std::function` ไม่ใช่ Function Pointer ธรรมดา

Part 64.7 สอนไว้ว่า `std::function` คือ **Type Erasure** ที่เก็บ Callable object ได้ทุกชนิด
(function pointer, functor, lambda) ไว้ในตัวแปรชนิดเดียวกัน — นี่คือกลไกที่ทำให้ `Router` และ
`App` ของเรารับ **lambda ที่ capture ตัวแปรภายนอกได้** (เช่น `[&store](const Request& req,
Response& res) { ... }` ที่จะเห็นใน Task API หัวข้อ 124.7) ถ้าใช้ function pointer ธรรมดา
(`void (*)(const Request&, Response&)`) จะ**ไม่สามารถ capture ตัวแปรใดๆ ได้เลย** เพราะ
function pointer ชี้ไปที่โค้ดล้วนๆ ไม่มีที่เก็บ state ส่วนตัว (closure) ต่างจาก lambda ที่มี
capture list

```cpp
using Handler = std::function<void(const Request&, Response&)>;
using Middleware = std::function<bool(Request&, Response&)>;
```

สองบรรทัดนี้คือหัวใจของความยืดหยุ่นทั้งหมดของ MicroWeb — เชื่อมโยงตรงกับ Part 56 (Function
Template ที่ทำให้ `std::function` เองเป็น class template ที่ instantiate ได้กับ signature
ใดก็ได้) และ Part 64 (Lambda + `std::function` สำหรับ Callback Pattern) ที่เรียนมาก่อนหน้านี้

### ทำไม Middleware คืนค่า `bool` แทนที่จะเป็น `void`

Middleware ของ MicroWeb มี signature `bool(Request&, Response&)` — คืนค่า:

- **`true`**: "ทำงานเสร็จแล้ว ไปต่อได้" — middleware ตัวถัดไป (หรือ handler ถ้าเป็นตัวสุดท้าย)
  จะถูกเรียกต่อ
- **`false`**: "หยุดสายทันที" — Response ที่ middleware ตัวนี้เตรียมไว้แล้วจะถูกส่งกลับไปหา
  client เลย ไม่เรียก middleware/handler ตัวที่เหลือ

การออกแบบนี้ทำให้เขียน **Authentication Middleware** ได้ตรงไปตรงมามาก — ถ้า credential ผิด
middleware แค่ set `res.status = 401` แล้ว `return false;` ก็หยุด pipeline ได้ทันที ไม่ต้อง
โยน exception หรือใช้กลไกซับซ้อนอื่นใด

### เปรียบเทียบกับ Middleware แบบ "Onion Model" ของ Express.js

Framework บางตัว (เช่น Express.js, Koa.js) ใช้ middleware แบบ **onion model**: middleware แต่
ละตัวรับ callback ชื่อ `next()` และเรียกมันเพื่อ "ส่งต่อ" ไปยัง middleware ถัดไป — จุดเด่นของ
แบบนี้คือ middleware **ทำงานได้ทั้งก่อนและหลัง** handler (เช่น middleware วัดเวลาที่ handler
ใช้ทั้งหมด ต้องมีโค้ดทั้งก่อนเรียก `next()` และหลัง `next()` คืนค่ากลับมา)

```
Onion Model (Express.js):        Linear Model (MicroWeb):

middleware1 { ก่อน                middleware1 -> true
  middleware2 { ก่อน                  │
    handler()                          ▼
  } หลัง                          middleware2 -> true
} หลัง                                 │
                                       ▼
                                    handler()
                                  (จบเลย ไม่มีจังหวะ "หลัง" middleware)
```

MicroWeb เลือก **Linear Model** ที่ง่ายกว่ามาก (middleware ทำงานแค่ "ก่อน" handler เท่านั้น)
เพราะครอบคลุม use case ที่พบบ่อยที่สุด (logging ตอนรับ request, authentication) ได้เพียงพอแล้ว
โดยไม่ต้องเขียน callback-passing ที่ซับซ้อนกว่า — **นี่คือ trade-off ที่ตั้งใจเลือกความง่ายใน
การ implement เหนือความสามารถเต็มรูปแบบ** ซึ่งเป็นตัวอย่างที่ดีของ Extension Exercise ที่ 4 ใน
ท้ายบท (เพิ่ม Onion Model เต็มรูปแบบ)

### `threadpool.hpp`: นำ Part 101 มาต่อยอดแบบทั่วไป

ก่อนจะประกอบ `App` เข้าด้วยกัน เราต้องมี Thread Pool ที่รับงานได้ **หลายชนิด** ไม่ใช่แค่
`int clientFd` ตรงๆ แบบ Part 101.5 — `ThreadPool` เวอร์ชันนี้ generalize ให้ `enqueue()` รับ
`std::function<void()>` ใดก็ได้ (ผูก closure ของงานไว้ในตัวมันเองทั้งหมด):

```cpp
#pragma once
// threadpool.hpp - ThreadPool ทั่วไป ต่อยอดตรงๆ จากคลาส ThreadPool ของ Part 101
// (threadpool_http_server.cpp) ต่างกันแค่จุดเดียว: แทนที่จะรับ "int clientFd" ตรงๆ
// เราให้ enqueue() รับ std::function<void()> ใดๆ ก็ได้ (Type Erasure ผ่าน std::function
// ที่เรียนใน Part 64) ทำให้ ThreadPool ตัวนี้นำไปใช้กับงานอะไรก็ได้ ไม่ผูกกับ HTTP โดยตรง
// เหมือนตัวเดิมของ Part 101 — App ของ MicroWeb จะเป็นผู้ผูก "task" ให้กลายเป็น
// "อ่าน+ประมวลผล+ตอบ 1 connection" อีกที (ดูหัวข้อ 124.5)
#include <condition_variable>
#include <functional>
#include <mutex>
#include <queue>
#include <thread>
#include <vector>

namespace microweb {

class ThreadPool {
public:
    explicit ThreadPool(std::size_t numWorkers) : stopping_(false) {
        for (std::size_t i = 0; i < numWorkers; ++i) {
            workers_.emplace_back([this] { workerLoop(); });
        }
    }

    ThreadPool(const ThreadPool&) = delete;
    ThreadPool& operator=(const ThreadPool&) = delete;

    ~ThreadPool() { shutdown(); }

    void enqueue(std::function<void()> task) {
        {
            std::lock_guard<std::mutex> lock(queueMutex_);
            if (stopping_) return;  // ปฏิเสธงานใหม่ระหว่าง shutdown เหมือน Part 101.5
            taskQueue_.push(std::move(task));
        }
        condVar_.notify_one();
    }

    void shutdown() {
        {
            std::lock_guard<std::mutex> lock(queueMutex_);
            if (stopping_) return;
            stopping_ = true;
        }
        condVar_.notify_all();
        for (auto& w : workers_) {
            if (w.joinable()) w.join();
        }
    }

private:
    void workerLoop() {
        for (;;) {
            std::function<void()> task;
            {
                std::unique_lock<std::mutex> lock(queueMutex_);
                condVar_.wait(lock, [this] { return stopping_ || !taskQueue_.empty(); });
                if (taskQueue_.empty()) {
                    if (stopping_) return;
                    continue;
                }
                task = std::move(taskQueue_.front());
                taskQueue_.pop();
            }
            task();  // ปลด lock แล้วค่อยทำงานจริงเสมอ (กฎทองจาก Part 101.5)
        }
    }

    std::vector<std::thread> workers_;
    std::queue<std::function<void()>> taskQueue_;
    std::mutex queueMutex_;
    std::condition_variable condVar_;
    bool stopping_;
};

}  // namespace microweb
```

ทุกกฎทองจาก Part 101.5 ยังคงอยู่ครบถ้วน: predicate ป้องกัน Spurious Wakeup, ปลด lock ก่อนเรียก
`task()` (เพื่อไม่ให้ worker อื่นต้องรอ), และ `shutdown()` ที่ join worker ทุกตัวก่อน destruct
เสมอ — สิ่งที่เปลี่ยนไปมีแค่ชนิดของสิ่งที่อยู่ในคิว (`std::function<void()>` แทน `int`)

---

## 124.5 ประกอบร่างเป็น App: Router + Middleware + Socket Server (Step 989)

### `app.hpp`: ประกาศ Interface แบบ Fluent

```cpp
#pragma once
// app.hpp - App: จุดประกอบร่างทุกอย่างของ MicroWeb เข้าด้วยกัน
// (Router + Middleware chain + ThreadPool + Raw Socket Server จาก Part 100-101)
#include <atomic>
#include <cstddef>
#include <string>

#include "microweb/request.hpp"
#include "microweb/response.hpp"
#include "microweb/router.hpp"
#include "microweb/threadpool.hpp"

namespace microweb {

class App {
public:
    App();

    // --- Fluent route-registration API: app.get("/users/:id", handler) ---
    App& get(const std::string& path, Handler handler);
    App& post(const std::string& path, Handler handler);
    App& put(const std::string& path, Handler handler);
    App& del(const std::string& path, Handler handler);

    // ลงทะเบียน middleware — ทำงานตามลำดับที่ .use() ถูกเรียก (ลงทะเบียนก่อน = รันก่อนเสมอ)
    App& use(Middleware mw);

    // เริ่มรัน server (blocking call เหมือน crow::SimpleApp::run() ใน Part 102)
    // numWorkers ค่าเริ่มต้น 0 หมายถึง "ใช้ hardware_concurrency() ของเครื่อง"
    void run(int port, std::size_t numWorkers = 0);

    // ให้ Signal Handler เรียกเพื่อขอ shutdown อย่างสุภาพ (ดูหัวข้อ 124.5)
    void requestStop();

    std::size_t routeCount() const { return router_.routeCount(); }

private:
    void serviceClient(int clientFd);

    Router router_;
    std::vector<Middleware> middlewares_;
    std::atomic<bool> stopping_{false};
};

}  // namespace microweb
```

สังเกตว่า `get()`, `post()`, `put()`, `del()`, `use()` คืนค่า `App&` (reference กลับไปที่ตัวเอง)
— ทำให้เขียนต่อกันเป็นสายได้ (`app.use(...).get(...).post(...)`) แม้ในตัวอย่างของบทนี้จะไม่ได้
เขียนแบบต่อสายกันเสมอไป (เพื่อความอ่านง่าย) แต่ API ก็เปิดโอกาสให้ทำได้ นี่คือ Fluent Interface
Pattern เดียวกับที่ `crow::SimpleApp` ใช้ตอน `app.port(N).run()` (Part 102.3)

### `app.cpp`: หัวใจของ Pipeline

`app.cpp` มี 3 ส่วนใหญ่: (1) ฟังก์ชันช่วย parse HTTP request แบบเต็มรูปแบบ (ต่อยอดจาก Part
100 แต่เพิ่ม header + body), (2) `serviceClient()` ที่รัน pipeline `Router -> Middleware ->
Handler -> Response`, และ (3) `run()` ที่เปิด socket + thread pool (ต่อยอดจาก Part 101.6
โดยตรงเกือบทั้งหมด)

```cpp
// app.cpp - Implementation ของ App: raw socket server (Part 100-101) + Router +
// Middleware chain ประกอบเข้าด้วยกันเป็น "Web Framework" ตัวจิ๋ว
#include "microweb/app.hpp"

#include <arpa/inet.h>
#include <netinet/in.h>
#include <sys/socket.h>
#include <unistd.h>

#include <cerrno>
#include <cstdio>
#include <cstring>
#include <thread>

namespace microweb {

namespace {

constexpr std::size_t kMaxHeaderBytes = 8192;    // กัน client ส่ง header ยาวไม่มีที่สิ้นสุด
constexpr std::size_t kMaxBodyBytes = 10 * 1024 * 1024;  // 10 MB (ดูแบบฝึกหัดข้อ 1)
constexpr std::size_t kRecvChunk = 4096;

// ---------- ขั้นที่ 1: parse request line (เหมือน Part 100 เป๊ะ) ----------
struct RequestLine {
    std::string method, fullPath, version;
    bool valid = false;
};

RequestLine parseRequestLine(const std::string& line) {
    RequestLine rl;
    std::size_t firstSpace = line.find(' ');
    std::size_t lastSpace = line.rfind(' ');
    if (firstSpace == std::string::npos || lastSpace == std::string::npos ||
        firstSpace == lastSpace) {
        return rl;
    }
    rl.method = line.substr(0, firstSpace);
    rl.fullPath = line.substr(firstSpace + 1, lastSpace - firstSpace - 1);
    rl.version = line.substr(lastSpace + 1);
    rl.valid = true;
    return rl;
}

// ---------- ขั้นที่ 2: parse header block (ใหม่เทียบกับ Part 100 ที่ไม่ parse header เลย)
// headerBlock คือทุกอย่างระหว่าง request line กับบรรทัดว่างคู่ ("\r\n\r\n")
void parseHeaders(const std::string& headerBlock, Request& req) {
    std::size_t pos = 0;
    while (pos < headerBlock.size()) {
        std::size_t lineEnd = headerBlock.find("\r\n", pos);
        if (lineEnd == std::string::npos) lineEnd = headerBlock.size();
        std::string line = headerBlock.substr(pos, lineEnd - pos);
        pos = lineEnd + 2;

        std::size_t colon = line.find(':');
        if (colon == std::string::npos) continue;  // header บรรทัดแปลก ข้ามไปเงียบๆ
        std::string key = line.substr(0, colon);
        std::size_t valueStart = colon + 1;
        while (valueStart < line.size() && line[valueStart] == ' ') ++valueStart;
        std::string value = line.substr(valueStart);
        req.headers[key] = value;
    }
}

// ---------- ขั้นที่ 3: แยก query string ออกจาก path + parse "k=v&k2=v2" ----------
void splitPathAndQuery(const std::string& fullPath, Request& req) {
    std::size_t q = fullPath.find('?');
    std::string queryStr;
    if (q == std::string::npos) {
        req.path = fullPath;
    } else {
        req.path = fullPath.substr(0, q);
        queryStr = fullPath.substr(q + 1);
    }

    std::size_t pos = 0;
    while (pos < queryStr.size()) {
        std::size_t amp = queryStr.find('&', pos);
        if (amp == std::string::npos) amp = queryStr.size();
        std::string pair = queryStr.substr(pos, amp - pos);
        std::size_t eq = pair.find('=');
        if (eq != std::string::npos) {
            req.query[pair.substr(0, eq)] = pair.substr(eq + 1);
        } else if (!pair.empty()) {
            req.query[pair] = "";
        }
        pos = amp + 1;
    }
}

// อ่าน request ทั้งก้อนจาก socket จนกว่าจะได้ header ครบ + body ครบตาม Content-Length
// (ต่างจาก Part 100 ที่ recv() ครั้งเดียวแล้วสมมติว่าได้ข้อมูลครบ — สมมติฐานนั้นใช้ไม่ได้อีก
// ต่อไปแล้วตอนต้อง parse body ของ POST/PUT ที่อาจมีขนาดใหญ่กว่า 1 buffer ของ recv())
bool readFullRequest(int fd, std::string& out) {
    char chunk[kRecvChunk];
    std::size_t headerEnd = std::string::npos;
    std::size_t contentLength = 0;
    bool haveContentLength = false;

    for (;;) {
        ssize_t n = recv(fd, chunk, sizeof(chunk), 0);
        if (n < 0) return false;
        if (n == 0) break;  // client ปิด connection
        out.append(chunk, static_cast<std::size_t>(n));

        if (headerEnd == std::string::npos) {
            headerEnd = out.find("\r\n\r\n");
            if (headerEnd != std::string::npos) {
                // เจอจุดจบ header แล้ว หา Content-Length เพื่อรู้ว่าต้องอ่าน body อีกกี่ byte
                std::string headerPart = out.substr(0, headerEnd);
                std::size_t clPos = headerPart.find("Content-Length:");
                if (clPos == std::string::npos) clPos = headerPart.find("content-length:");
                if (clPos != std::string::npos) {
                    std::size_t numStart = headerPart.find(':', clPos) + 1;
                    contentLength = static_cast<std::size_t>(std::stoul(headerPart.substr(numStart)));
                    haveContentLength = true;
                    if (contentLength > kMaxBodyBytes) return false;  // ป้องกัน body ถล่ม memory
                }
            } else if (out.size() > kMaxHeaderBytes) {
                return false;  // header ยาวเกินไป ไม่ใช่ HTTP request ที่สมเหตุสมผล
            }
        }

        if (headerEnd != std::string::npos) {
            std::size_t bodySoFar = out.size() - (headerEnd + 4);
            if (!haveContentLength || bodySoFar >= contentLength) {
                break;  // ไม่มี body ต้องอ่าน หรืออ่านครบตาม Content-Length แล้ว
            }
        }
    }
    return headerEnd != std::string::npos || !out.empty();
}

std::string statusText(int code) {
    switch (code) {
        case 200: return "OK";
        case 201: return "Created";
        case 204: return "No Content";
        case 400: return "Bad Request";
        case 401: return "Unauthorized";
        case 403: return "Forbidden";
        case 404: return "Not Found";
        case 405: return "Method Not Allowed";
        case 413: return "Payload Too Large";
        case 500: return "Internal Server Error";
        default: return "Unknown";
    }
}

std::string serialize(const Response& res) {
    std::string out;
    out += "HTTP/1.1 " + std::to_string(res.status) + " " + statusText(res.status) + "\r\n";
    for (const auto& h : res.headers) {
        out += h.first + ": " + h.second + "\r\n";
    }
    out += "Content-Length: " + std::to_string(res.body.size()) + "\r\n";
    out += "Connection: close\r\n";
    out += "\r\n";
    out += res.body;
    return out;
}

void sendAll(int fd, const std::string& data) {
    std::size_t sent = 0;
    while (sent < data.size()) {
        ssize_t n = send(fd, data.data() + sent, data.size() - sent, 0);
        if (n < 0) break;
        sent += static_cast<std::size_t>(n);
    }
}

}  // namespace

App::App() = default;

App& App::get(const std::string& path, Handler handler) {
    router_.add("GET", path, std::move(handler));
    return *this;
}
App& App::post(const std::string& path, Handler handler) {
    router_.add("POST", path, std::move(handler));
    return *this;
}
App& App::put(const std::string& path, Handler handler) {
    router_.add("PUT", path, std::move(handler));
    return *this;
}
App& App::del(const std::string& path, Handler handler) {
    router_.add("DELETE", path, std::move(handler));
    return *this;
}

App& App::use(Middleware mw) {
    middlewares_.push_back(std::move(mw));
    return *this;
}

void App::requestStop() { stopping_ = true; }

// ---------- หัวใจของ Pipeline: Request -> Router -> Middleware chain -> Handler -> Response
void App::serviceClient(int clientFd) {
    std::string raw;
    if (!readFullRequest(clientFd, raw)) {
        close(clientFd);
        return;
    }

    std::size_t headerEnd = raw.find("\r\n\r\n");
    std::size_t lineEnd = raw.find("\r\n");

    Request req;
    Response res;

    if (lineEnd == std::string::npos) {
        res.status = 400;
        res.json({{"success", false},
                   {"error", {{"code", "BAD_REQUEST"}, {"message", "malformed request line"}}}});
        sendAll(clientFd, serialize(res));
        close(clientFd);
        return;
    }

    RequestLine rl = parseRequestLine(raw.substr(0, lineEnd));
    if (!rl.valid) {
        res.status = 400;
        res.json({{"success", false},
                   {"error", {{"code", "BAD_REQUEST"}, {"message", "malformed request line"}}}});
        sendAll(clientFd, serialize(res));
        close(clientFd);
        return;
    }

    req.method = rl.method;
    req.version = rl.version;
    req.valid = true;
    splitPathAndQuery(rl.fullPath, req);

    if (headerEnd != std::string::npos) {
        std::string headerBlock = raw.substr(lineEnd + 2, headerEnd - (lineEnd + 2));
        parseHeaders(headerBlock, req);
        req.body = raw.substr(headerEnd + 4);
    }

    // [1] Router: หา handler ที่ตรงกับ (method, path) ก่อนสิ่งอื่นใด — ตามผังใน 124.1
    std::map<std::string, std::string> params;
    RouteMatch match = router_.resolve(req.method, req.path, params);
    req.params = params;

    Handler finalHandler;
    if (match.found) {
        req.route_pattern = match.pattern;
        finalHandler = match.handler;
    } else {
        req.route_pattern = "(unmatched)";
        finalHandler = [](const Request& r, Response& out) {
            out.status = 404;
            out.json({{"success", false},
                       {"error", {{"code", "NOT_FOUND"},
                                  {"message", "ไม่พบ path: " + r.path}}}});
        };
    }

    // [2] Middleware chain: รันตามลำดับที่ .use() ไว้ ตัวไหน return false คือ "หยุดสายทันที"
    bool shortCircuited = false;
    for (auto& mw : middlewares_) {
        if (!mw(req, res)) {
            shortCircuited = true;
            break;
        }
    }

    // [3] Handler: เรียกเฉพาะตอนไม่มี middleware ตัวไหนหยุดสายไว้ก่อน
    if (!shortCircuited) {
        finalHandler(req, res);
    }

    // [4] Response: serialize ตาม HTTP spec แล้วส่งกลับ (เหมือน buildResponse() ของ Part 100)
    sendAll(clientFd, serialize(res));
    close(clientFd);
}

void App::run(int port, std::size_t numWorkers) {
    if (numWorkers == 0) {
        numWorkers = std::thread::hardware_concurrency();
        if (numWorkers == 0) numWorkers = 4;
    }

    int serverFd = socket(AF_INET, SOCK_STREAM, 0);
    if (serverFd < 0) {
        perror("socket");
        return;
    }
    int opt = 1;
    setsockopt(serverFd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));

    struct sockaddr_in addr;
    std::memset(&addr, 0, sizeof(addr));
    addr.sin_family = AF_INET;
    addr.sin_addr.s_addr = INADDR_ANY;
    addr.sin_port = htons(static_cast<uint16_t>(port));

    if (bind(serverFd, reinterpret_cast<struct sockaddr*>(&addr), sizeof(addr)) < 0) {
        perror("bind");
        close(serverFd);
        return;
    }
    if (listen(serverFd, 128) < 0) {
        perror("listen");
        close(serverFd);
        return;
    }

    struct timeval tv;
    tv.tv_sec = 1;
    tv.tv_usec = 0;
    setsockopt(serverFd, SOL_SOCKET, SO_RCVTIMEO, &tv, sizeof(tv));

    std::printf("[microweb] listening on port %d with %zu worker threads, %zu route(s) registered\n",
                port, numWorkers, router_.routeCount());

    {
        ThreadPool pool(numWorkers);
        while (!stopping_) {
            struct sockaddr_in clientAddr;
            socklen_t clientLen = sizeof(clientAddr);
            int clientFd = accept(serverFd, reinterpret_cast<struct sockaddr*>(&clientAddr), &clientLen);
            if (clientFd < 0) {
                if (errno == EAGAIN || errno == EWOULDBLOCK) continue;
                if (stopping_) break;
                continue;
            }
            pool.enqueue([this, clientFd] { serviceClient(clientFd); });
        }
        std::printf("[microweb] กำลัง shutdown: รอ worker ทำงานที่ค้างอยู่ให้เสร็จก่อน...\n");
    }

    close(serverFd);
    std::printf("[microweb] ปิดเซิร์ฟเวอร์เรียบร้อย\n");
}

}  // namespace microweb
```

### จุดสำคัญที่ต้องเข้าใจให้ลึก

1. **`readFullRequest()` แก้ข้อจำกัดของ Part 100 ที่ `recv()` ครั้งเดียวแล้วจบ** — ตอนนี้เรา
   ต้องรองรับ body ของ POST/PUT ที่อาจใหญ่กว่า 1 buffer (`kRecvChunk` = 4096 byte) ฟังก์ชันนี้
   จึง `recv()` วนซ้ำจนกว่าจะเจอ `\r\n\r\n` (จบ header) แล้วอ่านต่อจนครบตาม `Content-Length`
   ที่ header บอกไว้ — นี่คือ pattern เดียวกับที่ Part 100.3 อธิบายว่าทำไม `Content-Length`
   ถึงสำคัญ เพียงแต่คราวนี้เราเป็น**ฝั่งที่ต้องอ่าน** Content-Length แทนที่จะเป็นฝั่งที่ต้อง
   *เขียน* มันแบบ Part 100
2. **`App::run()` แทบจะเหมือน `threadpool_http_server.cpp` ของ Part 101.6 ทุกประการ** — ใช้
   `SO_RCVTIMEO` แบบเดียวกันเพื่อให้ `accept()` คืนค่ากลับมาเป็นระยะและเช็ค `stopping_` ได้
   (Graceful Shutdown) และใช้ `ThreadPool` ตัวเดียวกันโครงสร้าง เพียงแต่ตอนนี้ enqueue เป็น
   lambda ที่ capture `this` และ `clientFd` แทนที่จะ enqueue `clientFd` ตรงๆ
3. **Pipeline `[1] Router -> [2] Middleware -> [3] Handler`**: การให้ Router ทำงานก่อน
   Middleware (ตรงข้ามกับ intuition แรกที่หลายคนคิดว่า middleware ควรทำงานก่อนเสมอ) เป็นการ
   ตัดสินใจออกแบบที่ตั้งใจ — เหตุผลคือ **Middleware ต้องรู้ `req.route_pattern` เพื่อ log ได้
   อย่างมีความหมาย** (ดูหัวข้อถัดไป) ถ้า Middleware ทำงานก่อน Router มันจะไม่รู้เลยว่า path
   นี้จะ match route ไหน (หรือไม่ match เลย) ทำให้ log บอกได้แค่ path ดิบเท่านั้น — การให้
   Router ทำงานก่อนแก้ปัญหานี้ได้ และเป็นเหตุผลที่ built-in 404 handler ก็ถูกสร้างเป็น
   `finalHandler` **ก่อน** middleware chain จะรัน (middleware ยังคง log ได้แม้ request จะจบ
   ด้วย 404 ก็ตาม เพราะ middleware chain รันทุกครั้งไม่ว่า route จะ match หรือไม่)

---

## 124.6 คอมไพล์ครั้งแรกและทดสอบ Hello World + Middleware Logging (Step 990)

### `CMakeLists.txt`

```cmake
cmake_minimum_required(VERSION 3.16)
project(microweb CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

find_package(Threads REQUIRED)

# MicroWeb เป็น static library เล็กๆ: router + app + threadpool (request/response เป็น
# header-only ล้วนจึงไม่ต้อง compile แยก)
add_library(microweb STATIC
    src/app.cpp
)
target_include_directories(microweb PUBLIC include)
target_link_libraries(microweb PUBLIC Threads::Threads)

# Demo: Task API ที่สร้างขึ้นบน MicroWeb
add_executable(task_api_microweb demo/main.cpp)
target_link_libraries(task_api_microweb PRIVATE microweb)
```

```bash
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build -j4
```

ผลลัพธ์จริงจากการรันคำสั่งนี้:

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
-- Configuring done (0.3s)
-- Generating done (0.0s)
-- Build files have been written to: .../microweb/build
[ 25%] Building CXX object CMakeFiles/microweb.dir/src/app.cpp.o
[ 50%] Linking CXX static library libmicroweb.a
[ 50%] Built target microweb
[ 75%] Building CXX object CMakeFiles/task_api_microweb.dir/demo/main.cpp.o
[100%] Linking CXX executable task_api_microweb
[100%] Built target task_api_microweb
```

Build ผ่านสะอาดโดยไม่มี warning เลยแม้แต่บรรทัดเดียว เช่นเดียวกับตอนคอมไพล์ทุกไฟล์ในบทนี้ด้วย
`g++` ตรงๆ:

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 -pthread -O2 -Iinclude -c src/app.cpp -o /tmp/app.o
# (ไม่มี output ใดๆ ออกมาเลย = compile ผ่านโดยไม่มี warning สักตัว)
```

### โปรแกรมทดสอบแรก: 1 route ธรรมดา + 1 route มี path parameter + logging middleware

ก่อนจะไปสร้าง Task API เต็มรูปแบบ มาทดสอบ pipeline พื้นฐานก่อนด้วยโปรแกรมเล็กๆ:

```cpp
// hello.cpp - โปรแกรม MicroWeb โปรแกรมแรก: 1 route ธรรมดา + 1 route มี path parameter
// + 1 middleware แบบ logging เพื่อพิสูจน์ pipeline เต็มรูปแบบก่อนไปสร้าง Task API ใน 124.7
#include <csignal>
#include <cstdio>
#include "microweb/app.hpp"

using microweb::App;
using microweb::Request;
using microweb::Response;

App* g_app = nullptr;
void onSigint(int) {
    if (g_app != nullptr) g_app->requestStop();
}

int main() {
    App app;

    app.use([](Request& req, Response&) {
        std::printf("[log] %s %s (route=%s)\n", req.method.c_str(), req.path.c_str(),
                    req.route_pattern.c_str());
        return true;
    });

    app.get("/", [](const Request&, Response& res) {
        res.text("Hello, MicroWeb!");
    });

    app.get("/greet/:name", [](const Request& req, Response& res) {
        res.json({{"message", "สวัสดี " + req.param("name") + "!"}});
    });

    g_app = &app;
    std::signal(SIGINT, onSigint);
    app.run(8123, 4);
    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 -pthread -O2 -Iinclude \
    demo/hello.cpp src/app.cpp -o hello_microweb
./hello_microweb &
curl -s -i http://127.0.0.1:8123/
curl -s -i http://127.0.0.1:8123/greet/Somchai
```

ผลลัพธ์จริงจากการทดสอบ:

```
HTTP/1.1 200 OK
Content-Type: text/plain; charset=utf-8
Content-Length: 16
Connection: close

Hello, MicroWeb!
HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 41
Connection: close

{"message":"สวัสดี Somchai!"}
```

และฝั่ง server (terminal ที่รันค้างไว้) พิมพ์ log ของ middleware ออกมาจริงทุก request:

```
[microweb] listening on port 8123 with 4 worker threads, 2 route(s) registered
[log] GET / (route=/)
[log] GET /greet/Somchai (route=/greet/:name)
[microweb] กำลัง shutdown: รอ worker ทำงานที่ค้างอยู่ให้เสร็จก่อน...
[microweb] ปิดเซิร์ฟเวอร์เรียบร้อย
```

สังเกตว่า `route=` ใน log บอก **pattern ที่ลงทะเบียนไว้** (`/greet/:name`) ไม่ใช่ path จริงที่
client ส่งมา (`/greet/Somchai`) — นี่คือประโยชน์ของการให้ Router ทำงานก่อน Middleware ตามที่
อธิบายไว้ในหัวข้อ 124.5: ถ้ามี client ยิง `/greet/Somchai`, `/greet/Somsri`, `/greet/John` มา
หลายพันครั้ง log จะรวมกันเป็นบรรทัดเดียวที่มีความหมาย (`route=/greet/:name`) แทนที่จะกลายเป็น
หลายพันบรรทัดที่ต่างกันแค่ชื่อ ทำให้วิเคราะห์ traffic ของ endpoint ได้ง่ายกว่ามาก — นี่คือ
เทคนิคที่ Framework จริงระดับ production (รวมถึง APM tool อย่าง Datadog, New Relic) ใช้กันจริง
เวลารายงานสถิติของ endpoint

### ทดสอบว่า URL Encoding ยังไม่ถูก decode (ข้อจำกัดที่ควรรู้ตัวไว้)

ลองยิง path parameter ที่เป็นข้อความไทยผ่าน percent-encoding (`%E0%B9%84...`):

```bash
curl -s -i http://127.0.0.1:8123/greet/%E0%B9%84%E0%B8%97%E0%B8%A2
```

ผลลัพธ์จริง:

```
HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 61
Connection: close

{"message":"สวัสดี %E0%B9%84%E0%B8%97%E0%B8%A2!"}
```

**MicroWeb ไม่ decode percent-encoding ใน path เลย** — ค่าที่ `req.param("name")` ได้คือ string
ดิบตามที่ client ส่งมาตรงๆ ต่างจาก Framework จริงอย่าง Crow/Express ที่ decode `%XX` ให้อัตโนมัติ
ก่อนส่งเข้า handler นี่เป็นข้อจำกัดจริงที่ค้นพบจากการทดสอบจริง (ไม่ใช่แค่ทฤษฎี) และเป็นตัวอย่าง
ที่ดีว่าทำไมการสร้าง Framework เองถึงเผยข้อจำกัดที่ผู้ใช้ Framework สำเร็จรูปไม่เคยรู้ตัวว่ามัน
เป็นงานที่ Framework ทำให้อัตโนมัติอยู่เบื้องหลัง — ถ้าจะนำ MicroWeb ไปใช้จริง ต้องเพิ่มฟังก์ชัน
`urlDecode()` เข้าไปใน `splitPathAndQuery()` ก่อน (ไม่ได้รวมไว้ในบทนี้เพื่อโฟกัสที่สถาปัตยกรรม
หลักของ framework แต่ผู้เรียนที่สนใจสามารถลองเขียนเพิ่มเป็นแบบฝึกหัดต่อยอดได้)

---

## 124.7 สร้าง Task API เวอร์ชันย่อของ Part 108 ขึ้นใหม่บน MicroWeb (Step 991)

นี่คือบทพิสูจน์สุดท้ายว่า MicroWeb ใช้งานได้จริงแบบ end-to-end — เราจะสร้าง Task API (จาก
Part 108) ขึ้นใหม่ทั้งหมดโดย **ไม่แตะ Crow เลยแม้แต่บรรทัดเดียว** ครบทั้ง 5 endpoint ตามหลัก
CRUD, JSON envelope pattern เดียวกับ Part 108.2, และเพิ่ม **Authentication Middleware** สำหรับ
endpoint ที่ลบข้อมูล

เพื่อโฟกัสที่ตัว Framework เอง (ไม่ใช่ Data Access Layer ที่ Part 108.5 สอนไปแล้วอย่างละเอียด)
เวอร์ชันนี้เก็บข้อมูลใน memory ด้วย `std::vector<Task>` ที่ป้องกัน Race Condition ด้วย
`std::mutex` แทน SQLite — หลักการเชื่อมต่อฐานข้อมูลจริงยังคงเหมือนกับ Part 108.5 ทุกประการ
(ห่อหุ้มไว้ในคลาสเดียว ป้องกันด้วย mutex หรือใช้ connection pool) เพียงแต่ตัดความซับซ้อนของ
SQL ออกเพื่อให้เห็นโครงสร้างของ Framework ชัดเจนที่สุด

### `demo/main.cpp`

```cpp
// main.cpp - Demo: สร้าง Task API เวอร์ชันย่อของ Part 108 ใหม่ทั้งหมดบน MicroWeb
// ที่เราเขียนขึ้นเอง (ไม่ใช้ Crow เลยแม้แต่บรรทัดเดียว) เพื่อพิสูจน์ว่า Framework ที่สร้าง
// เองใช้งานได้จริงแบบ end-to-end: routing + path parameter + middleware + JSON response
#include <csignal>
#include <cstdio>
#include <ctime>
#include <mutex>
#include <optional>
#include <vector>

#include "microweb/app.hpp"

using microweb::App;
using microweb::Request;
using microweb::Response;
using nlohmann::json;

// ---------- Model (โครงสร้างเดียวกับ Part 108 แต่เก็บใน memory แทน SQLite เพื่อให้ตัวอย่าง
// นี้โฟกัสที่ตัว Framework เอง ไม่ใช่ตัว Data Access Layer ที่ Part 108 สอนไปแล้ว) ----------
struct Task {
    int id;
    std::string title;
    std::string description;
    bool done;
    std::string created_at;

    json to_json() const {
        return json{{"id", id}, {"title", title}, {"description", description},
                    {"done", done}, {"created_at", created_at}};
    }
};

// ---------- "Data layer" ง่ายๆ ป้องกัน Race Condition ด้วย std::mutex เพราะ MicroWeb
// เรียก handler จากหลาย worker thread พร้อมกันได้เสมอ (เหมือน Db ของ Part 108.5) ----------
class TaskStore {
public:
    std::vector<Task> listAll() {
        std::lock_guard<std::mutex> lock(mutex_);
        return tasks_;
    }

    std::optional<Task> findById(int id) {
        std::lock_guard<std::mutex> lock(mutex_);
        for (auto& t : tasks_) {
            if (t.id == id) return t;
        }
        return std::nullopt;
    }

    Task create(const std::string& title, const std::string& description, bool done) {
        std::lock_guard<std::mutex> lock(mutex_);
        Task t;
        t.id = nextId_++;
        t.title = title;
        t.description = description;
        t.done = done;
        t.created_at = currentTimestamp();
        tasks_.push_back(t);
        return t;
    }

    bool update(int id, const std::string& title, const std::string& description, bool done) {
        std::lock_guard<std::mutex> lock(mutex_);
        for (auto& t : tasks_) {
            if (t.id == id) {
                t.title = title;
                t.description = description;
                t.done = done;
                return true;
            }
        }
        return false;
    }

    bool remove(int id) {
        std::lock_guard<std::mutex> lock(mutex_);
        for (auto it = tasks_.begin(); it != tasks_.end(); ++it) {
            if (it->id == id) {
                tasks_.erase(it);
                return true;
            }
        }
        return false;
    }

private:
    static std::string currentTimestamp() {
        std::time_t now = std::time(nullptr);
        char buf[32];
        std::strftime(buf, sizeof(buf), "%Y-%m-%d %H:%M:%S", std::gmtime(&now));
        return buf;
    }

    std::vector<Task> tasks_;
    std::mutex mutex_;
    int nextId_ = 1;
};

App* g_app = nullptr;
void signalHandler(int) {
    if (g_app != nullptr) g_app->requestStop();
}

// ---------- Response envelope helper (แนวคิดเดียวกับ api::ok()/api::error() ของ Part 108.2)
namespace api {
void ok(Response& res, const json& data, int status = 200) {
    res.status = status;
    res.json({{"success", true}, {"data", data}});
}
void error(Response& res, int status, const std::string& code, const std::string& message) {
    res.status = status;
    res.json({{"success", false}, {"error", {{"code", code}, {"message", message}}}});
}
}  // namespace api

int main() {
    App app;
    TaskStore store;

    // ---------- Middleware #1: Logging (รันทุก request รวมถึงที่ 404) ----------
    app.use([](Request& req, Response&) {
        std::printf("[log] %s %s (route=%s)\n", req.method.c_str(), req.path.c_str(),
                    req.route_pattern.c_str());
        return true;  // เดินหน้าต่อเสมอ ไม่เคยหยุดสาย
    });

    // ---------- Middleware #2: Auth เฉพาะ DELETE ----------
    // ตรวจสอบ header "X-API-Key" เฉพาะตอน method เป็น DELETE เท่านั้น (route อื่นผ่านได้เสมอ)
    // นี่คือตัวอย่างที่ตรงกับ Part 102/103 ที่ Crow ก็ทำ auth middleware แบบเดียวกันนี้
    app.use([](Request& req, Response& res) {
        if (req.method != "DELETE") return true;
        std::string key = req.header("X-API-Key");
        if (key != "secret123") {
            api::error(res, 401, "UNAUTHORIZED", "ต้องแนบ header X-API-Key ที่ถูกต้อง");
            return false;  // หยุดสายทันที ไม่เรียก handler ลบข้อมูลจริง
        }
        return true;
    });

    // ---------- Routes ----------
    app.get("/tasks", [&store](const Request&, Response& res) {
        json arr = json::array();
        for (auto& t : store.listAll()) arr.push_back(t.to_json());
        api::ok(res, arr);
    });

    app.get("/tasks/:id", [&store](const Request& req, Response& res) {
        int id = std::atoi(req.param("id").c_str());
        auto found = store.findById(id);
        if (!found) {
            api::error(res, 404, "TASK_NOT_FOUND", "ไม่พบ Task ที่มี id = " + req.param("id"));
            return;
        }
        api::ok(res, found->to_json());
    });

    app.post("/tasks", [&store](const Request& req, Response& res) {
        json body;
        try {
            body = req.json();
        } catch (const json::parse_error&) {
            api::error(res, 400, "INVALID_JSON", "body ไม่ใช่ JSON ที่ถูกต้อง");
            return;
        }
        if (!body.contains("title") || !body["title"].is_string() ||
            body["title"].get<std::string>().empty()) {
            api::error(res, 400, "VALIDATION_ERROR", "ต้องมี field title ที่ไม่ว่างเปล่า");
            return;
        }
        std::string description = body.value("description", "");
        bool done = body.value("done", false);
        Task created = store.create(body["title"].get<std::string>(), description, done);
        api::ok(res, created.to_json(), 201);
    });

    app.put("/tasks/:id", [&store](const Request& req, Response& res) {
        int id = std::atoi(req.param("id").c_str());
        auto existing = store.findById(id);
        if (!existing) {
            api::error(res, 404, "TASK_NOT_FOUND", "ไม่พบ Task ที่มี id = " + req.param("id"));
            return;
        }
        json body;
        try {
            body = req.json();
        } catch (const json::parse_error&) {
            api::error(res, 400, "INVALID_JSON", "body ไม่ใช่ JSON ที่ถูกต้อง");
            return;
        }
        std::string title = body.value("title", existing->title);
        std::string description = body.value("description", existing->description);
        bool done = body.value("done", existing->done);
        store.update(id, title, description, done);
        auto updated = store.findById(id);
        api::ok(res, updated->to_json());
    });

    app.del("/tasks/:id", [&store](const Request& req, Response& res) {
        int id = std::atoi(req.param("id").c_str());
        if (!store.remove(id)) {
            api::error(res, 404, "TASK_NOT_FOUND", "ไม่พบ Task ที่มี id = " + req.param("id"));
            return;
        }
        api::ok(res, json{{"deleted_id", id}});
    });

    g_app = &app;
    std::signal(SIGINT, signalHandler);

    app.run(8124, 4);
    return 0;
}
```

### Build และทดสอบทุก Endpoint จริงด้วย curl

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 -pthread -O2 -Iinclude \
    demo/main.cpp src/app.cpp -o task_api_microweb
./task_api_microweb &
```

**GET /tasks (ก่อนมีข้อมูล):**

```bash
curl -s -i http://127.0.0.1:8124/tasks
```

```
HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 26
Connection: close

{"data":[],"success":true}
```

**POST /tasks (สร้าง 2 รายการ):**

```bash
curl -s -i -X POST -H "Content-Type: application/json" \
    -d '{"title":"เขียนบทเรียน Part 124","description":"สร้าง MicroWeb framework"}' \
    http://127.0.0.1:8124/tasks

curl -s -i -X POST -H "Content-Type: application/json" \
    -d '{"title":"ทดสอบ curl"}' http://127.0.0.1:8124/tasks
```

```
HTTP/1.1 201 Created
Content-Type: application/json
Content-Length: 187
Connection: close

{"data":{"created_at":"2026-09-26 12:21:48","description":"สร้าง MicroWeb framework","done":false,"id":1,"title":"เขียนบทเรียน Part 124"},"success":true}
HTTP/1.1 201 Created
Content-Type: application/json
Content-Length: 128
Connection: close

{"data":{"created_at":"2026-09-26 12:21:48","description":"","done":false,"id":2,"title":"ทดสอบ curl"},"success":true}
```

สังเกตว่า `description` ที่ไม่ได้ส่งมาใน request ที่สอง ถูก default เป็น string ว่างอัตโนมัติ
ผ่าน `body.value("description", "")` ของ nlohmann/json (เทคนิคเดียวกับที่ Part 107 สอน)

**GET /tasks/1 (path parameter):**

```bash
curl -s -i http://127.0.0.1:8124/tasks/1
```

```
HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 187
Connection: close

{"data":{"created_at":"2026-09-26 12:21:48","description":"สร้าง MicroWeb framework","done":false,"id":1,"title":"เขียนบทเรียน Part 124"},"success":true}
```

**GET /tasks/999 (ไม่พบ -> 404 ตาม envelope pattern):**

```bash
curl -s -i http://127.0.0.1:8124/tasks/999
```

```
HTTP/1.1 404 Not Found
Content-Type: application/json
Content-Length: 109
Connection: close

{"error":{"code":"TASK_NOT_FOUND","message":"ไม่พบ Task ที่มี id = 999"},"success":false}
```

**PUT /tasks/1 (partial update — ส่งแค่ `done`):**

```bash
curl -s -i -X PUT -d '{"done":true}' http://127.0.0.1:8124/tasks/1
```

```
HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 186
Connection: close

{"data":{"created_at":"2026-09-26 12:21:48","description":"สร้าง MicroWeb framework","done":true,"id":1,"title":"เขียนบทเรียน Part 124"},"success":true}
```

**POST /tasks (validation error — ไม่มี title):**

```bash
curl -s -i -X POST -d '{}' http://127.0.0.1:8124/tasks
```

```
HTTP/1.1 400 Bad Request
Content-Type: application/json
Content-Length: 142
Connection: close

{"error":{"code":"VALIDATION_ERROR","message":"ต้องมี field title ที่ไม่ว่างเปล่า"},"success":false}
```

**DELETE /tasks/2 โดยไม่มี API Key (Middleware หยุดสายที่ 401):**

```bash
curl -s -i -X DELETE http://127.0.0.1:8124/tasks/2
```

```
HTTP/1.1 401 Unauthorized
Content-Type: application/json
Content-Length: 131
Connection: close

{"error":{"code":"UNAUTHORIZED","message":"ต้องแนบ header X-API-Key ที่ถูกต้อง"},"success":false}
```

**DELETE /tasks/2 พร้อม API Key ที่ถูกต้อง:**

```bash
curl -s -i -X DELETE -H "X-API-Key: secret123" http://127.0.0.1:8124/tasks/2
```

```
HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 40
Connection: close

{"data":{"deleted_id":2},"success":true}
```

### Log ของ Middleware ที่ยิงจริงตลอดการทดสอบ

ฝั่ง server พิมพ์ log ของทุก request ที่ผ่านเข้ามา (ทั้งจาก Middleware #1 logging) ตามลำดับที่
เกิดขึ้นจริง:

```
[microweb] listening on port 8124 with 4 worker threads, 5 route(s) registered
[log] GET /tasks (route=/tasks)
[log] POST /tasks (route=/tasks)
[log] POST /tasks (route=/tasks)
[log] GET /tasks (route=/tasks)
[log] GET /tasks/1 (route=/tasks/:id)
[log] GET /tasks/999 (route=/tasks/:id)
[log] PUT /tasks/1 (route=/tasks/:id)
[log] POST /tasks (route=/tasks)
[log] DELETE /tasks/2 (route=/tasks/:id)
[log] DELETE /tasks/2 (route=/tasks/:id)
[log] GET /tasks (route=/tasks)
[microweb] กำลัง shutdown: รอ worker ทำงานที่ค้างอยู่ให้เสร็จก่อน...
[microweb] ปิดเซิร์ฟเวอร์เรียบร้อย
```

สังเกตว่า **DELETE /tasks/2 ปรากฏใน log ทั้งสองครั้ง** (ครั้งแรกที่ถูก middleware auth ปฏิเสธ
ด้วย 401 และครั้งที่สองที่ผ่านด้วย API key ที่ถูกต้อง) — พิสูจน์ว่า Middleware #1 (logging)
ทำงาน**ก่อน** Middleware #2 (auth) เสมอ ตามลำดับที่ `.use()` ถูกเรียกไว้ในโค้ด แม้ Middleware
#2 จะหยุดสายกลางทาง (`return false`) ก็ไม่กระทบ Middleware #1 ที่ทำงานไปก่อนหน้าแล้ว — นี่คือ
พฤติกรรมที่ถูกต้องตามที่ pipeline design ในหัวข้อ 124.4 ตั้งใจไว้ทุกประการ

### ทดสอบ Robustness: Malformed Request, Large Body, Invalid JSON

**Request ที่ผิดรูปแบบสิ้นเชิง (ไม่มี space เลยสักตัว) — ต้องได้ 400:**

```bash
( exec 3<>/dev/tcp/127.0.0.1/8124; printf 'GARBAGE\r\n\r\n' >&3; timeout 1 cat <&3; exec 3<&- )
```

```
HTTP/1.1 400 Bad Request
Content-Type: application/json
Content-Length: 83
Connection: close

{"error":{"code":"BAD_REQUEST","message":"malformed request line"},"success":false}
```

**Body ขนาดใหญ่กว่า 1 buffer ของ `recv()` (9,041 byte เทียบกับ `kRecvChunk` = 4,096):**

```bash
python3 -c "
import json
desc = 'x' * 9000
print(json.dumps({'title': 'big task', 'description': desc}))
" > bigbody.json
wc -c bigbody.json
# 9041 bigbody.json

curl -s -i -X POST -H "Content-Type: application/json" --data @bigbody.json \
    http://127.0.0.1:8124/tasks | head -c 200
```

```
HTTP/1.1 201 Created
Content-Type: application/json
Content-Length: 9116
Connection: close

{"data":{"created_at":"2026-09-26 12:22:59","description":"xxxxxxxxxxxxxx...
```

Task ถูกสร้างสำเร็จ (`201 Created`) พร้อม `description` ครบทั้ง 9,000 ตัวอักษร — พิสูจน์ว่า
`readFullRequest()` วน `recv()` ต่อเนื่องจนอ่านครบตาม `Content-Length` จริง ไม่ตัดข้อมูลทิ้ง
แม้ body จะใหญ่กว่า buffer ของ `recv()` แต่ละครั้งกว่า 2 เท่าก็ตาม

**JSON ที่ผิดรูปแบบ:**

```bash
curl -s -i -X POST -d 'not valid json {{{' http://127.0.0.1:8124/tasks
```

```
HTTP/1.1 400 Bad Request
Content-Type: application/json
Content-Length: 121
Connection: close

{"error":{"code":"INVALID_JSON","message":"body ไม่ใช่ JSON ที่ถูกต้อง"},"success":false}
```

`req.json()` throw `nlohmann::json::parse_error` ตามที่ออกแบบไว้ใน `request.hpp` และ handler
`try/catch` มันได้ถูกต้อง ตอบ `400 Bad Request` แทนที่จะปล่อยให้โปรแกรม crash — หลักการเดียวกับ
ที่ Part 102's แบบฝึกหัดข้อ 3 สอนไว้: **ไม่เชื่อ input จากภายนอกเด็ดขาด**

---

## 124.8 เปรียบเทียบกับ Crow จริง: Trade-off ทางวิศวกรรมที่ต้องเข้าใจ (Step 992)

### Feature Comparison: MicroWeb vs Crow

| Feature | MicroWeb (Part นี้) | Crow (Part 102–103) |
|---|---|---|
| Fluent routing API | ✅ `app.get(path, handler)` | ✅ `CROW_ROUTE(app, path)` |
| Path parameter | ✅ `:id` (runtime, ได้ `std::string` เสมอ) | ✅ `<int>`/`<string>`/`<path>` (compile-time, type-safe) |
| Query string | ✅ `req.queryParam("key")` | ✅ `req.url_params.get("key")` |
| JSON request/response | ✅ ผ่าน `nlohmann::json` | ✅ ผ่าน `crow::json` ในตัว |
| Middleware | ✅ Linear model (ก่อน handler เท่านั้น) | ✅ Onion model เต็มรูปแบบ (`before_handle`/`after_handle`) |
| Thread Pool ภายใน | ✅ คงที่ตาม `numWorkers` ที่ตั้งค่า | ✅ ปรับอัตโนมัติ + ใช้ asio async I/O |
| Persistent Connection (Keep-Alive) | ❌ ปิดทุก connection หลัง 1 response | ✅ รองรับเต็มรูปแบบ |
| I/O Model | Blocking (`recv()`/`send()` บล็อก thread ทั้งเส้น) | Non-blocking (asio event loop, epoll-based) |
| URL Decoding | ❌ ไม่ decode `%XX` เลย | ✅ decode อัตโนมัติ |
| HTTPS/TLS | ❌ ไม่รองรับ | ✅ รองรับผ่าน asio SSL |
| WebSocket | ❌ ไม่รองรับ | ✅ รองรับในตัว |
| จำนวนบรรทัดโค้ด (framework เอง) | ~450 บรรทัด | หลายหมื่นบรรทัด (รวม asio) |
| ระยะเวลาพัฒนา | ไม่กี่ชั่วโมง (บทเรียนนี้) | หลายปี โดยทีม/ชุมชน open source |
| ผ่านการทดสอบใน production จริง | ❌ ไม่เคย | ✅ ใช้งานจริงหลายบริษัททั่วโลก |

### Benchmark จริง: MicroWeb vs Crow ที่ระดับ Concurrency ต่างกัน

เพื่อให้เห็นผลกระทบของความแตกต่างเรื่อง **I/O Model** (Blocking vs Non-blocking) อย่างเป็น
รูปธรรม ทดสอบทั้งสอง server ด้วยเครื่องมือ `bench_client` ตัวเดียวกับ Part 101.7 (ยิง `GET /`
plain text ธรรมดา ไม่มี business logic ปนเพื่อวัด "ต้นทุนของตัว framework เอง" ล้วนๆ):

```bash
./microweb_hello 8130 4 &
./bench_client 8130 100 20 /
./bench_client 8130 300 10 /
./bench_client 8130 1000 5 /

./crow_hello 8131 &
./bench_client 8131 100 20 /
./bench_client 8131 300 10 /
./bench_client 8131 1000 5 /
```

ผลลัพธ์จริงจากการทดสอบทั้งหมด (เครื่องทดสอบมี 4 logical core เท่ากับที่ใช้ทดสอบใน Part 101):

| Concurrency | Total Request | MicroWeb (4 worker, blocking) | Crow (asio, non-blocking) |
|---|---|---|---|
| 100 | 2,000 | **34,718.1 req/s** | 27,720.3 req/s |
| 300 | 3,000 | 2,381.5 req/s | **38,535.8 req/s** |
| 1,000 | 5,000 | 2,367.2 req/s | **28,831.0 req/s** |

### วิเคราะห์ผลลัพธ์อย่างตรงไปตรงมา

ผลลัพธ์นี้**น่าประหลาดใจในแวบแรก**: ที่ concurrency ต่ำ (100) MicroWeb กลับ**เร็วกว่า Crow**
ด้วยซ้ำ! เหตุผลคือที่ concurrency ระดับนี้ยังไม่มากพอที่จะทำให้ thread ทั้ง 4 ตัวของ MicroWeb
อิ่มตัว (saturate) และ MicroWeb ไม่มี overhead ของ asio's event loop machinery ที่ซับซ้อนกว่า
(scheduling callback, buffer management ภายใน) ทำให้งานง่ายๆ อย่าง "ตอบ string คงที่กลับไป"
เสร็จเร็วกว่าในกรณีนี้

แต่พอ concurrency เพิ่มเป็น 300 ตัวเลขของ MicroWeb **ร่วงลงมาเกือบ 15 เท่า** (จาก 34,718 เหลือ
2,381 req/s) ในขณะที่ Crow กลับ**เร็วขึ้น**เป็น 38,535 req/s — ที่ concurrency 1,000 MicroWeb
ยังคงติดอยู่ที่ระดับ ~2,300 req/s ในขณะที่ Crow ยังคงรักษาระดับเกือบ 30,000 req/s ไว้ได้

เหตุผลคือ **ข้อจำกัดที่ Part 101.8 เตือนไว้แล้วล่วงหน้า**: "Worker Thread แต่ละตัวยังคง
'บล็อก' ตลอดเวลาที่จัดการ 1 Connection" — MicroWeb มี worker thread แค่ 4 ตัว (เท่ากับจำนวน
core) แต่ละตัว `recv()`/`send()` แบบ **blocking** เต็มรูปแบบ เมื่อ connection พร้อมกันมากกว่า
จำนวน worker หลายเท่า (300 หรือ 1,000 เทียบกับ worker แค่ 4 ตัว) connection ส่วนใหญ่ต้อง**รอคิว
อยู่ใน `taskQueue_`** จนกว่า worker ตัวใดตัวหนึ่งจะว่าง — ในขณะที่ Crow ใช้ **asio (non-blocking,
event-driven I/O)** ที่ thread จำนวนน้อยสามารถ "สลับดูแล" connection หลายพันตัวพร้อมกันได้โดย
ไม่ต้องบล็อกรอ connection ไหนตัวหนึ่งนานๆ (ใช้ `epoll` ของ Linux ข้างใต้ — เทคนิคที่ Part 34
ปูพื้นไว้ในชื่อ "I/O Multiplexing")

### ตารางสรุป Trade-off

| ประเด็น | Blocking I/O + Thread Pool คงที่ (MicroWeb) | Non-blocking I/O + Event Loop (Crow/asio) |
|---|---|---|
| ความซับซ้อนในการ implement | ต่ำ (เข้าใจง่าย debug ง่าย) | สูงมาก (ต้องจัดการ callback, state machine ของแต่ละ connection) |
| Throughput ที่ concurrency ต่ำ | ดี (อาจดีกว่าด้วยซ้ำ) | ดี |
| Throughput ที่ concurrency สูง | **แย่ลงมาก** (ติดคอขวดที่จำนวน thread) | **คงที่หรือดีขึ้น** (ไม่ผูกกับจำนวน thread) |
| จำนวน connection พร้อมกันสูงสุดที่รองรับได้ดี | จำกัดด้วยจำนวน worker thread (มักหลักสิบ-ร้อย) | จำกัดด้วย memory/file descriptor (มักหลักหมื่น-แสน) |
| เหมาะกับ | งานเรียนรู้, prototype, internal tool ที่ traffic ต่ำ | Production web service ที่ต้องรองรับผู้ใช้จำนวนมาก |

### คำตอบตรงไปตรงมา: "ควรเอา Framework นี้ไปใช้ Production จริงไหม?"

**คำตอบคือแทบจะไม่ควรเลย** และนี่คือเหตุผลที่ตรงไปตรงมาที่สุด:

1. **ไม่รองรับ Non-blocking I/O** — ตามที่ Benchmark ข้างต้นพิสูจน์แล้วว่า throughput ร่วงลง
   อย่างรุนแรงเมื่อ concurrency สูง ซึ่งเป็นสถานการณ์ปกติของเว็บ production จริงที่มีผู้ใช้
   พร้อมกันหลายร้อย-หลายพันคน
2. **ไม่รองรับ HTTPS/TLS** — ปี 2026 เว็บไซต์แทบทุกแห่งต้องใช้ HTTPS เป็นค่าเริ่มต้น (แม้แต่
   Browser สมัยใหม่ก็เตือน "ไม่ปลอดภัย" ถ้าไม่มี HTTPS) การเพิ่ม TLS support เข้าไปเองเป็นงาน
   ที่ซับซ้อนและเสี่ยงเรื่องความปลอดภัยสูงมากถ้าไม่มีทีมผู้เชี่ยวชาญด้าน cryptography ตรวจสอบ
3. **ไม่มี Test Suite ที่ครอบคลุมทุก Edge Case ของ HTTP Spec** — เช่น Chunked Transfer
   Encoding, HTTP/2, multipart form data, cookie parsing — Crow ผ่านการใช้งานจริงและแก้บั๊ก
   จากผู้ใช้งานหลายพันคนมาแล้วหลายปี ในขณะที่ MicroWeb เพิ่งถูกทดสอบด้วยมือไม่กี่สิบครั้งใน
   บทเรียนนี้เท่านั้น
4. **ไม่มี Security Audit** — ช่องโหว่ด้านความปลอดภัย (Buffer Overflow, Request Smuggling,
   Denial of Service) ที่ Framework ระดับ production ผ่านการตรวจสอบมาแล้วนับครั้งไม่ถ้วน ยังไม่
   เคยถูกตรวจสอบใน MicroWeb เลย

**แต่คำตอบนี้ไม่ได้แปลว่า Part นี้เสียเวลาเปล่า** — ตรงข้ามเลย นี่คือประเด็นสำคัญที่สุดของทั้ง
บทเรียน: **คุณค่าของการสร้าง Framework เองไม่ได้อยู่ที่ "จะเอาไปใช้จริงหรือไม่" แต่อยู่ที่
"ตอนนี้คุณเข้าใจว่า Crow (หรือ framework ภาษาไหนก็ตาม) ทำงานอย่างไรข้างใน"** เมื่อวันหนึ่ง Crow
ใน production ทำงานช้าผิดปกติ หรือ route ไม่ match ตามที่คาดหวัง หรือ middleware ทำงานผิดลำดับ
— วิศวกรที่เข้าใจว่า router ต้องมีระบบให้คะแนนแก้ ambiguity, middleware เป็น chain ที่ short-
circuit ได้, และ thread pool มีขีดจำกัดเรื่อง blocking I/O จะ **debug ปัญหาได้เร็วกว่าและลึก
กว่า** วิศวกรที่มองมันเป็นกล่องดำมาก — นี่คือหัวใจของปรัชญาที่ Part 99 วางไว้ตั้งแต่ต้นทาง และ
เป็นบทสรุปที่แท้จริงของ Capstone 4 นี้

---

## ข้อผิดพลาดที่พบบ่อย (Common Pitfalls)

1. **Route-Matching Ambiguity ที่ไม่ได้แก้** — ถ้า Router implement แบบง่ายที่สุด (จับคู่ route
   แรกที่ match ได้ตามลำดับที่ลงทะเบียน) ผลลัพธ์การ routing จะ**ขึ้นกับลำดับที่เขียนโค้ด** ซึ่ง
   เป็นบั๊กที่ตรวจจับยากมากเพราะไม่มี compile error หรือ warning ใดๆ เตือนเลย ลองดูโค้ดที่ผิด
   ต่อไปนี้ (Router แบบไม่มีระบบให้คะแนน):

   ```cpp
   // บั๊ก: จับคู่ route แรกที่เจอ ไม่สนว่ามี route อื่นที่ "ตรงกว่า"
   for (const auto& r : routes_) {
       if (r.method != method) continue;
       if (matchesLoosely(r, path)) {   // ไม่แยกคะแนน literal vs param
           return r;                     // คืนค่าทันทีตัวแรกที่เจอ!
       }
   }
   ```

   ถ้าลงทะเบียน `/users/:id` ก่อน `/users/me` request `GET /users/me` จะ match เข้า
   `/users/:id` เสมอ (ได้ `id = "me"`) — ทางแก้คือระบบให้คะแนนตามที่ Router ของ MicroWeb ใช้
   (หัวข้อ 124.3): เก็บ**ทุก route ที่ match ได้** แล้วเลือกตัวที่คะแนนสูงสุด ไม่ใช่ตัวแรกที่เจอ

2. **Middleware Ordering Bug: auth middleware ทำงานหลัง handler ที่ควรถูกป้องกัน** — ถ้าเขียน
   `.get()` ก่อน `.use()` โดยไม่ระวังลำดับความหมาย (แม้ MicroWeb จะออกแบบให้ middleware ทำงาน
   ก่อน handler เสมอไม่ว่าจะ `.use()` ก่อนหรือหลัง `.get()` ในโค้ดก็ตาม เพราะ pipeline แยก
   ขั้นตอนชัดเจน) แต่ปัญหาที่แท้จริงที่พบบ่อยกว่าคือ **ลำดับระหว่าง middleware หลายตัวสลับกัน
   โดยไม่ได้ตั้งใจ**:

   ```cpp
   // บั๊ก: middleware ตรวจสอบ rate limit ทำงาน "หลัง" middleware auth
   // ทำให้ request ที่ auth ไม่ผ่านยังถูกนับเข้า rate limit counter อยู่ดี (เปลืองโดยไม่จำเป็น)
   app.use(authMiddleware);       // ควรทำงานทีหลัง ไม่ใช่ก่อน
   app.use(rateLimitMiddleware);  // ควรทำงานก่อน (กันโหลดก่อนเสียเวลาตรวจ auth)
   ```

   ทางแก้คือคิดลำดับ middleware ให้ชัดเจนตั้งแต่ต้น: **สิ่งที่ควรกรอง request ออกเร็วที่สุด
   (rate limiting, IP blacklist) ควรอยู่ก่อนสุด** ตามด้วย authentication แล้วค่อยเป็น
   business logic เฉพาะ route — MicroWeb ไม่มีกลไกตรวจสอบลำดับนี้ให้อัตโนมัติ เป็นหน้าที่ของ
   ผู้เขียนแอปพลิเคชันที่ต้องคิดลำดับ `.use()` ให้ถูกต้องเอง

3. **Thread-Safety ของ Router ถ้าเปิดให้ลงทะเบียน Route ระหว่างที่ Server รันอยู่** —
   MicroWeb ในบทเรียนนี้ **ลงทะเบียน route ทั้งหมดก่อนเรียก `app.run()` เท่านั้น** (single-
   threaded ตอน setup, ไม่มี thread อื่นแตะ `Router::routes_` พร้อมกันเลย) แต่ถ้าในอนาคตอยาก
   เพิ่มความสามารถ "hot reload route ระหว่างที่ server กำลังรับ request อยู่" (เช่น plugin
   system ที่โหลด route ใหม่แบบ dynamic) จะเกิด **Data Race ทันที** เพราะ worker thread หลาย
   ตัวเรียก `Router::resolve()` (อ่าน `routes_`) พร้อมกับอีก thread หนึ่งเรียก `Router::add()`
   (เขียน `routes_`) — `std::vector` ไม่ thread-safe เลยแม้แต่นิดเดียว (ทบทวนจาก Part 101.5)
   ทางแก้คือห่อ `routes_` ด้วย `std::shared_mutex` (อ่านพร้อมกันได้หลาย thread ด้วย
   `std::shared_lock` แต่เขียนต้องล็อกแบบ exclusive ด้วย `std::unique_lock`) — ดูแบบฝึกหัดข้อ
   4 ท้ายบทสำหรับแนวทางที่ควรทำถ้าต้องการความสามารถนี้จริง

4. **ลืมว่า `Content-Length` เป็นสิ่งที่ต้อง "อ่าน" ไม่ใช่แค่ "เขียน"** — Part 100 สอนให้เรา
   ระวังเรื่องเขียน `Content-Length` ให้ถูกต้องตอนสร้าง response แต่ตอนสร้าง framework ที่รับ
   request เข้ามา เราต้อง**อ่าน** `Content-Length` จาก client ให้ถูกต้องด้วย ถ้า
   `readFullRequest()` หยุดอ่านทันทีที่เจอ `\r\n\r\n` โดยไม่สนใจว่ายังมี body เหลืออยู่อีกกี่
   byte ตาม `Content-Length` ที่ประกาศไว้ POST/PUT request ที่มี body ขนาดใหญ่กว่า 1 buffer
   ของ `recv()` (ตามที่ทดสอบจริงในหัวข้อ 124.7 ด้วย body 9,041 byte) จะถูกตัดข้อมูลทิ้งกลาง
   คัน ทำให้ `nlohmann::json::parse()` ล้มเหลวด้วย error ที่งงว่า "ทำไม JSON ถึงขาดๆ หายๆ"

5. **ปล่อยให้ Client กำหนด `Content-Length` ได้อย่างไม่จำกัด (Denial of Service)** — ถ้าไม่มี
   `kMaxBodyBytes` จำกัดไว้ client ที่จงใจส่ง header `Content-Length: 999999999999` (เกือบ 1
   Terabyte) จะทำให้ server พยายามจัดสรร memory มหาศาลเพื่อรอรับ body ที่อาจไม่มีวันมาครบ
   (หรือมาจริงแต่ทำให้ memory ของเครื่องหมดจนกระทบ request อื่นทั้งหมด) — นี่คือรูปแบบหนึ่งของ
   **Denial of Service Attack** ที่ framework ระดับ production ทุกตัวต้องป้องกันไว้ตั้งแต่ต้น
   ดูแบบฝึกหัดข้อ 1 (เฉลยเต็มรูปแบบ) สำหรับวิธีแก้ที่ถูกต้อง

6. **Path Traversal ถ้าทำ Static File Serving แบบไม่ระวัง** — ถ้าจะเพิ่มความสามารถส่งไฟล์
   กลับไป (เช่น `/static/*filepath`) และต่อ path ด้วยการ concatenate string ตรงๆ โดยไม่ตรวจ
   สอบ `..` ก่อน client ที่ส่ง `filepath = "../../../etc/passwd"` จะสามารถอ่านไฟล์ **นอก**
   โฟลเดอร์ที่ตั้งใจให้เข้าถึงได้ทั้งหมด — ดูแบบฝึกหัดข้อ 5 (เฉลยเต็มรูปแบบ) สำหรับวิธีป้องกัน
   ที่ถูกต้องด้วย `std::filesystem::weakly_canonical()`

---

## แบบฝึกหัดท้ายบท

1. **เพิ่มขีดจำกัดขนาด POST/PUT Body** — ตอนนี้ `readFullRequest()` ปิด connection เงียบๆ
   (ไม่ตอบอะไรกลับไปเลย) ถ้า `Content-Length` เกิน `kMaxBodyBytes` ให้แก้ไขให้ตอบกลับด้วย
   `413 Payload Too Large` พร้อม JSON envelope ที่บอกเหตุผลชัดเจน (เฉลยเต็มรูปแบบด้านล่าง)
2. **เพิ่ม Static File Serving** — เพิ่ม route `/static/*filepath` ที่ส่งไฟล์จากโฟลเดอร์
   `public/` กลับไป พร้อมป้องกัน Path Traversal Attack อย่างถูกต้อง (เฉลยเต็มรูปแบบด้านล่าง)
3. **แยกแยะ `404 Not Found` กับ `405 Method Not Allowed`** — ตอนนี้ `Router::resolve()` คืน
   `found = false` เหมือนกันหมดไม่ว่าจะเป็นเพราะ path ไม่มีอยู่จริง หรือ path มีอยู่แต่ method
   ไม่ตรง (เช่นมี `GET /tasks` แต่ client ยิง `PATCH /tasks`) ลองแก้ `RouteMatch` ให้แยกสอง
   กรณีนี้ออกจากกัน แล้วตอบ `405 Method Not Allowed` แทน `404` เมื่อ path ตรงแต่ method ไม่ตรง
   (ใบ้: ต้องค้นหาก่อนว่ามี route ที่ path ตรงกันหรือไม่ โดยไม่สนใจ method ก่อน แล้วค่อยกรองด้วย
   method อีกที)
4. **เพิ่ม Thread-Safety ให้ Router รองรับการลงทะเบียน Route ระหว่าง Server กำลังรันอยู่** —
   ห่อ `routes_` ของ `Router` ด้วย `std::shared_mutex` ให้ `resolve()` ใช้ `std::shared_lock`
   (อ่านพร้อมกันได้หลาย thread) และ `add()` ใช้ `std::unique_lock` (เขียนต้องล็อกคนเดียว)
   ทดสอบด้วยการเรียก `app.get(...)` เพิ่ม route ใหม่จาก thread แยก **ระหว่างที่** `bench_client`
   กำลังยิง request เข้ามาพร้อมกันอยู่ (รันด้วย ThreadSanitizer `-fsanitize=thread` เพื่อยืนยัน
   ว่าไม่มี Data Race หลงเหลืออยู่)
5. **เขียน CORS Middleware** — เพิ่ม middleware ที่ใส่ header
   `Access-Control-Allow-Origin: *` ให้กับทุก response (จำเป็นถ้าจะให้เว็บหน้าบ้านที่รันคนละ
   origin เรียก API นี้ผ่าน `fetch()` ของ JavaScript ได้) และรองรับ `OPTIONS` preflight request
   ที่ browser ส่งมาก่อน request จริงโดยอัตโนมัติ
6. **เพิ่มการรองรับ Persistent Connection (`Keep-Alive`)** — ตอนนี้ MicroWeb ปิด connection
   ทุกครั้งหลังตอบ 1 response (`Connection: close`) เหมือน Part 100-101 ลองแก้ให้ตรวจสอบ header
   `Connection: keep-alive` จาก request แล้ววน loop รับ request ใหม่ต่อจาก connection เดิมแทน
   การปิดทันที (ระวัง: ต้องคิดเรื่อง timeout ของ connection ที่เปิดค้างไว้นานเกินไปด้วย ไม่งั้น
   worker thread จะถูกยึดไว้ตลอดไปถ้า client ไม่ส่ง request ใหม่มาเลย)

### เฉลยข้อ 1: จำกัดขนาด Body ด้วย 413 Payload Too Large

แก้ `readFullRequest()` ให้คืนค่า `enum class` แทน `bool` เดี่ยว เพื่อให้ผู้เรียกรู้ **เหตุผล**
ที่อ่านไม่สำเร็จ และเลือกตอบ status code ที่ถูกต้อง:

```cpp
// เฉลยข้อ 1: แยกผลลัพธ์ของการอ่าน request ออกเป็น enum แทนที่จะเป็น bool เดียว เพราะตอนนี้
// เรามี "เหตุผลที่อ่านไม่สำเร็จ" มากกว่า 1 แบบ และแต่ละแบบควรตอบกลับด้วย status code ต่างกัน
enum class ReadResult { Ok, ClientClosed, HeaderTooLarge, BodyTooLarge };

ReadResult readFullRequest(int fd, std::string& out) {
    char chunk[kRecvChunk];
    std::size_t headerEnd = std::string::npos;
    std::size_t contentLength = 0;
    bool haveContentLength = false;

    for (;;) {
        ssize_t n = recv(fd, chunk, sizeof(chunk), 0);
        if (n < 0) return ReadResult::ClientClosed;
        if (n == 0) break;
        out.append(chunk, static_cast<std::size_t>(n));

        if (headerEnd == std::string::npos) {
            headerEnd = out.find("\r\n\r\n");
            if (headerEnd != std::string::npos) {
                std::string headerPart = out.substr(0, headerEnd);
                std::size_t clPos = headerPart.find("Content-Length:");
                if (clPos == std::string::npos) clPos = headerPart.find("content-length:");
                if (clPos != std::string::npos) {
                    std::size_t numStart = headerPart.find(':', clPos) + 1;
                    contentLength = static_cast<std::size_t>(std::stoul(headerPart.substr(numStart)));
                    haveContentLength = true;
                    // จุดสำคัญ: เช็คจาก header "Content-Length" ที่ client ประกาศไว้ *ก่อน*
                    // จะยอมอ่าน body จริงเลยสักไบต์เดียว — ปฏิเสธได้เร็วที่สุด ไม่ต้องเสียเวลา
                    // รับข้อมูลกิกะไบต์มาเก็บใน std::string ก่อนค่อยรู้ว่ามันเกิน limit
                    if (contentLength > kMaxBodyBytes) return ReadResult::BodyTooLarge;
                }
            } else if (out.size() > kMaxHeaderBytes) {
                return ReadResult::HeaderTooLarge;
            }
        }

        if (headerEnd != std::string::npos) {
            std::size_t bodySoFar = out.size() - (headerEnd + 4);
            // เผื่อกรณี client โกหก Content-Length (ประกาศน้อยกว่าที่ส่งจริง) เรายังต้องเช็ค
            // ขนาดจริงที่ทยอยรับเข้ามาด้วย ไม่ใช่เชื่อแค่ค่าที่ประกาศไว้ตอนเดียว
            if (bodySoFar > kMaxBodyBytes) return ReadResult::BodyTooLarge;
            if (!haveContentLength || bodySoFar >= contentLength) break;
        }
    }
    if (headerEnd == std::string::npos && out.empty()) return ReadResult::ClientClosed;
    return ReadResult::Ok;
}
```

และแก้ `serviceClient()` ให้ตอบ status code ที่ถูกต้องตาม `ReadResult`:

```cpp
ReadResult rr = readFullRequest(clientFd, raw);
if (rr == ReadResult::ClientClosed) {
    close(clientFd);
    return;
}
if (rr == ReadResult::HeaderTooLarge) {
    Response tooLarge;
    tooLarge.status = 431;  // 431 Request Header Fields Too Large
    tooLarge.json({{"success", false},
                    {"error", {{"code", "HEADER_TOO_LARGE"}, {"message", "header ยาวเกินกำหนด"}}}});
    sendAll(clientFd, serialize(tooLarge));
    close(clientFd);
    return;
}
if (rr == ReadResult::BodyTooLarge) {
    Response tooLarge;
    tooLarge.status = 413;
    tooLarge.json({{"success", false},
                    {"error", {{"code", "PAYLOAD_TOO_LARGE"},
                               {"message", "body ใหญ่เกิน " + std::to_string(kMaxBodyBytes) + " byte"}}}});
    sendAll(clientFd, serialize(tooLarge));
    close(clientFd);
    return;
}
```

ทดสอบจริงด้วยการตั้ง `kMaxBodyBytes = 200` (เพื่อทดสอบง่าย) และ route `/echo` ที่คืนจำนวน byte
ของ body ที่ได้รับ:

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 -pthread -O2 -Iinclude \
    main_ex1.cpp app_ex1.cpp -o ex1_server
./ex1_server &

# body เล็ก (21 byte) < 200 byte -> ผ่านปกติ
curl -s -i -X POST -d '{"msg":"hello world"}' http://127.0.0.1:8140/echo

# body ใหญ่ (311 byte) > 200 byte -> 413
curl -s -i -X POST --data @big.json http://127.0.0.1:8140/echo
```

ผลลัพธ์จริงจากการทดสอบ:

```
HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 21
Connection: close

{"received_bytes":21}
HTTP/1.1 413 Payload Too Large
Content-Type: application/json
Content-Length: 105
Connection: close

{"error":{"code":"PAYLOAD_TOO_LARGE","message":"body ใหญ่เกิน 200 byte"},"success":false}
```

body ขนาด 311 byte ถูกปฏิเสธด้วย `413` ตามที่ตั้งใจไว้ทันที **ก่อน**จะยอมอ่าน body ทั้งก้อนเข้า
memory ด้วยซ้ำ (เช็คจาก `Content-Length` ตั้งแต่จบ header) — นี่คือจุดสำคัญที่ทำให้การป้องกันนี้
มีประสิทธิภาพจริง ไม่ใช่แค่ตรวจสอบหลังรับข้อมูลมาเก็บไว้เต็มๆ แล้ว

### เฉลยข้อ 2: Static File Serving พร้อมป้องกัน Path Traversal

ขั้นแรกต้องเพิ่มความสามารถ **wildcard segment** ให้ `Router` ก่อน (segment ที่ขึ้นต้นด้วย `*`
จับ path ส่วนที่เหลือทั้งหมดไว้ในพารามิเตอร์เดียว คล้าย `<path>` ของ Crow ใน Part 102.4):

```cpp
// ส่วนที่แก้ไขใน Router::resolve() เพื่อรองรับ wildcard segment
for (const auto& r : routes_) {
    if (r.method != method) continue;

    // segment ที่ขึ้นต้นด้วย '*' คือ "wildcard" ต้องเป็น segment สุดท้ายของ pattern เท่านั้น
    // และยอมให้จำนวน segment จริงมากกว่าจำนวน segment ของ pattern ได้
    bool isWildcard = !r.segments.empty() && r.segments.back()[0] == '*';
    if (!isWildcard && r.segments.size() != reqSegments.size()) continue;
    if (isWildcard && reqSegments.size() < r.segments.size() - 1) continue;

    int score = 0;
    std::map<std::string, std::string> params;
    bool ok = true;
    std::size_t fixedCount = isWildcard ? r.segments.size() - 1 : r.segments.size();
    for (std::size_t i = 0; i < fixedCount; ++i) {
        const std::string& pat = r.segments[i];
        const std::string& real = reqSegments[i];
        if (!pat.empty() && pat[0] == ':') {
            params[pat.substr(1)] = real;
            score += 1;
        } else if (pat == real) {
            score += 10;
        } else {
            ok = false;
            break;
        }
    }
    if (ok && isWildcard) {
        std::string rest;
        for (std::size_t i = fixedCount; i < reqSegments.size(); ++i) {
            if (!rest.empty()) rest += "/";
            rest += reqSegments[i];
        }
        params[r.segments.back().substr(1)] = rest;
        score += 1;  // wildcard นับเป็น match ที่ "กว้าง" ที่สุด ให้คะแนนน้อยสุดเสมอ
    }
    if (!ok) continue;
    // ... ส่วนที่เหลือเหมือนเดิม (เลือก route คะแนนสูงสุด)
}
```

จากนั้นเขียน route `/static/*filepath` ที่อ่านไฟล์จากโฟลเดอร์ `public/` **พร้อมป้องกัน Path
Traversal Attack ด้วย `std::filesystem::weakly_canonical()`**:

```cpp
// main_ex2.cpp - เฉลยข้อ 2: Static File Serving ผ่าน wildcard route "/static/*filepath"
// พร้อมป้องกัน Path Traversal Attack (client ขอไฟล์นอก public/ เช่น "../../../etc/passwd")
#include <csignal>
#include <filesystem>
#include <fstream>
#include <sstream>
#include "microweb/app.hpp"

namespace fs = std::filesystem;
using microweb::App; using microweb::Request; using microweb::Response;

std::string guessContentType(const std::string& path) {
    if (path.size() >= 5 && path.substr(path.size() - 5) == ".html") return "text/html; charset=utf-8";
    if (path.size() >= 4 && path.substr(path.size() - 4) == ".css") return "text/css";
    if (path.size() >= 3 && path.substr(path.size() - 3) == ".js") return "application/javascript";
    return "application/octet-stream";
}

int main() {
    App app;
    const fs::path publicDir = fs::canonical("public");

    app.get("/static/*filepath", [publicDir](const Request& req, Response& res) {
        std::string requested = req.param("filepath");

        // ประกอบ path เต็มแล้วบังคับ resolve ให้เป็น canonical (แก้ ".." และ symlink ให้หมด)
        // ก่อนจะเทียบว่า path ที่ resolve แล้วยังอยู่ "ภายใต้" publicDir จริงหรือไม่
        fs::path requestedPath = publicDir / requested;
        std::error_code ec;
        fs::path canonicalPath = fs::weakly_canonical(requestedPath, ec);

        // เช็คว่า canonicalPath ขึ้นต้นด้วย publicDir จริง (กัน "../" หลุดออกไปนอกโฟลเดอร์)
        auto mismatch = std::mismatch(publicDir.begin(), publicDir.end(), canonicalPath.begin());
        bool insidePublicDir = (mismatch.first == publicDir.end());

        if (ec || !insidePublicDir || !fs::exists(canonicalPath) || !fs::is_regular_file(canonicalPath)) {
            res.status = 404;
            res.json({{"success", false},
                       {"error", {{"code", "FILE_NOT_FOUND"}, {"message", "ไม่พบไฟล์: " + requested}}}});
            return;
        }

        std::ifstream file(canonicalPath, std::ios::binary);
        std::ostringstream ss;
        ss << file.rdbuf();
        res.body = ss.str();
        res.setHeader("Content-Type", guessContentType(canonicalPath.string()));
        res.status = 200;
    });

    app.run(8150, 2);
    return 0;
}
```

ทดสอบการทำงานปกติก่อน (โฟลเดอร์ `public/` มี `index.html` และ `css/style.css`):

```bash
curl -s -i http://127.0.0.1:8150/static/index.html
curl -s -i http://127.0.0.1:8150/static/css/style.css
```

```
HTTP/1.1 200 OK
Content-Type: text/html; charset=utf-8
Content-Length: 27
Connection: close

<h1>Hello Static File</h1>

HTTP/1.1 200 OK
Content-Type: text/css
Content-Length: 21
Connection: close

body { color: red; }
```

ทดสอบ Path Traversal Attack จริง — ใช้ raw socket แทน `curl` ตรงๆ เพราะ `curl` เองมีการ
normalize `..` ใน URL ให้อัตโนมัติก่อนส่ง request ออกไปด้วยซ้ำ (ค้นพบจากการทดสอบจริง) ทำให้ต้อง
ส่ง raw bytes ผ่าน `/dev/tcp` เพื่อพิสูจน์ว่า **การป้องกันฝั่ง server ทำงานจริง** ไม่ได้ปลอดภัย
เพราะบังเอิญ client ช่วย normalize ให้:

```bash
( exec 3<>/dev/tcp/127.0.0.1/8150
  printf 'GET /static/css/../../secret.txt HTTP/1.1\r\nHost: x\r\n\r\n' >&3
  timeout 1 cat <&3
  exec 3<&- )
```

ผลลัพธ์จริง:

```
HTTP/1.1 404 Not Found
Content-Type: application/json
Content-Length: 113
Connection: close

{"error":{"code":"FILE_NOT_FOUND","message":"ไม่พบไฟล์: css/../../secret.txt"},"success":false}
```

`secret.txt` ที่อยู่**นอก** โฟลเดอร์ `public/` (ระดับเดียวกับตัวโปรแกรม) ไม่ถูกเปิดเผยออกมา —
`fs::weakly_canonical()` แปลง `.../css/../../secret.txt` ให้กลายเป็น path เต็มที่มี `secret.txt`
อยู่**นอก** `publicDir` แล้วเช็คด้วย `std::mismatch()` พบว่าไม่ได้ขึ้นต้นด้วย `publicDir` จึง
ปฏิเสธด้วย `404` ทันที (ตั้งใจตอบ `404` แทน `403` เพื่อไม่เปิดเผยแม้กระทั่งว่าไฟล์นั้น "มีอยู่
จริงแต่เข้าไม่ได้" — เป็นหลักการความปลอดภัยพื้นฐานที่เรียกว่า **Information Hiding**)

---

## สรุปท้ายบท

ใน Part นี้เราได้:

- สร้าง Web Framework ตัวจิ๋วชื่อ **MicroWeb** ขึ้นมาเองตั้งแต่ศูนย์ ต่อยอดโดยตรงจาก Raw
  Socket Server (Part 100), Thread Pool (Part 101), Template/`std::function`/Lambda (Part 56,
  64), และ JSON (Part 107) — ครบทุกความสามารถหลักที่ Crow มี: Fluent routing, path parameter,
  middleware chain, JSON request/response
- ออกแบบ `Request`/`Response` abstraction ที่ parse HTTP request แบบเต็มรูปแบบ (header, query
  string, body ตาม `Content-Length` ที่อาจใหญ่กว่า 1 buffer ของ `recv()`)
- ออกแบบ `Router` ที่แก้ปัญหา **route-matching ambiguity** ด้วยระบบให้คะแนน literal-vs-
  parameter segment อย่างถูกต้อง และพิสูจน์ด้วยการทดสอบจริงว่าลำดับการลงทะเบียน route ไม่มีผล
  ต่อผลการ routing เลย
- ออกแบบ **Middleware Chain** แบบ Linear model ด้วย `std::function<bool(Request&, Response&)>`
  ที่รองรับการหยุดสายกลางทางสำหรับทำ Authentication ได้
- ประกอบทุกชิ้นส่วนเข้าเป็นคลาส `App` เดียว compile ผ่านสะอาดด้วย `g++ -Wall -Wextra
  -Wpedantic -std=c++17` และ CMake โดยไม่มี warning เลยแม้แต่บรรทัดเดียว
- **สร้าง Task API เวอร์ชันย่อของ Part 108 ขึ้นใหม่ทั้งหมดบน MicroWeb** (ไม่ใช้ Crow เลย)
  ทดสอบครบทุก endpoint ด้วย `curl` จริง ทั้ง CRUD, validation, authentication middleware,
  malformed request, body ขนาดใหญ่ที่ข้าม buffer boundary, และ invalid JSON
- วัด Benchmark เปรียบเทียบกับ Crow จริง พบว่า MicroWeb เร็วกว่าที่ concurrency ต่ำ แต่ throughput
  ร่วงลงเกือบ 15 เท่าที่ concurrency สูง (300+) เพราะเป็น Blocking I/O ในขณะที่ Crow ใช้ asio
  (Non-blocking, epoll-based) รักษาระดับ throughput ไว้ได้แม้ concurrency สูงถึง 1,000
- วิเคราะห์ trade-off ทางวิศวกรรมอย่างตรงไปตรงมาว่า **ทำไมแทบไม่ควรเอา Framework แบบนี้ไปใช้
  Production จริง** (ไม่รองรับ Non-blocking I/O, HTTPS, ไม่ผ่าน Security Audit) แต่คุณค่าที่
  แท้จริงของบทเรียนนี้คือการเข้าใจว่า Framework ที่ใช้จริงทำงานอย่างไรข้างใน

### ภาพรวมของ Capstone ทั้ง 4 โปรเจกต์ใน Module K

| Capstone | โฟกัสหลัก | เทคนิคที่ผสมผสาน |
|---|---|---|
| Part 121: E-Commerce Backend | REST API เต็มรูปแบบระดับ production | Crow + PostgreSQL + Docker |
| Part 122: Real-time Chat Server | Concurrency และ Real-time Communication | Socket + WebSocket + Multi-threading |
| Part 123: Distributed Key-Value Store | Networking และ Data Consistency | Networking + Concurrency + Persistence |
| **Part 124: Web Framework ของตัวเอง** | **เข้าใจรากฐานของทุกอย่างที่ใช้มาตลอดหลักสูตร** | **Socket + Template + std::function + Thread Pool** |

Capstone นี้ปิดท้ายด้วยการหวนกลับไปหารากฐานที่สุด — ไม่ใช่เพราะมันคือโปรเจกต์ที่ "ยากที่สุด" ใน
เชิงเทคนิค (Part 122 และ 123 อาจซับซ้อนกว่าในแง่ distributed system) แต่เพราะมันคือโปรเจกต์ที่
**เชื่อมทุกอย่างที่เรียนมาตลอด 124 Part เข้าด้วยกันเป็นภาพเดียว** ตั้งแต่ `printf("Hello,
World!")` ใน Part 1 ไปจนถึง Thread Pool ที่ป้องกัน Race Condition ด้วย `std::mutex` ใน Part
101 มาจนถึง Template และ Lambda ใน Module E — ทั้งหมดนี้ไม่ใช่บทเรียนแยกส่วนกันอีกต่อไป แต่คือ
ส่วนประกอบของระบบเดียวที่ทำงานร่วมกันได้จริง

ใน **Part 125** ซึ่งเป็น Part สุดท้ายของหลักสูตรทั้งหมด เราจะสรุปภาพรวมของการเดินทางทั้ง 124
Part ที่ผ่านมา วาง Roadmap สำหรับก้าวต่อไปสู่ระดับ Senior/Staff Engineer และแนะนำแหล่งเรียนรู้
ต่อยอดสำหรับสายงาน C/C++ ในโลกจริงต่อไป

**ต่อไป:** [Part 125 — บทสรุปหลักสูตร: Roadmap สู่ระดับ World-Class Engineer และแหล่งเรียนรู้ต่อยอด](./part-125-final-summary.md)
