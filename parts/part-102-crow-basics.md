# Part 102: Crow Framework: พื้นฐานและ REST API แรก (Step 809–816)

> Module I — Web Development ด้วย C/C++ | Part 102 จาก 125
> Step ที่ครอบคลุมใน Part นี้: Step 809–816
> Part ก่อนหน้า: [Part 101 — สร้าง Multi-threaded HTTP Server พร้อม Thread Pool](./part-101-multithreaded-http-server.md) | Part ถัดไป: [Part 103 — Crow ขั้นสูง: Middleware, Routing, JSON](./part-103-crow-advanced.md)

## เป้าหมายการเรียนรู้ (Learning Objectives)

หลังจบ Part นี้ ผู้เรียนจะสามารถ:

1. อธิบายได้ว่า Crow คืออะไร ทำไมถึงเรียกว่า "C++ microframework" และต่างจาก Flask/Express
   อย่างไรในเชิงสถาปัตยกรรมและประสิทธิภาพ
2. ติดตั้ง Crow บนเครื่องของตัวเองได้จริง ทั้งการติดตั้ง dependency (`libasio-dev`) และการ
   build/install จาก source ด้วย CMake
3. เขียนโปรแกรม Hello World ด้วย `CROW_ROUTE` คอมไพล์และรันเป็น HTTP server ได้จริง
4. ใช้ Route Parameter (Path Variable) เช่น `/user/<int>` เพื่อดึงค่าจาก URL มาใช้ในโค้ดได้
5. กำหนด HTTP Method ให้แต่ละ Route ด้วย `.methods()` และแยกแยะ GET กับ POST ได้ถูกต้อง
6. อ่านค่าจาก Query String (`?key=value`) และ Request Body ของ POST request ได้
7. คืนค่า Response เป็น JSON ด้วย `crow::json::wvalue` ได้อย่างถูกต้อง พร้อมเข้าใจว่า
   Content-Type ถูกกำหนดอัตโนมัติเมื่อไหร่
8. เปรียบเทียบโค้ดที่เขียนด้วย Crow กับ Raw Socket Server จาก Part 100-101 ได้อย่างชัดเจนว่า
   Crow ช่วยลดงานซ้ำซากส่วนไหนไปบ้าง

---

## 102.1 Crow คืออะไร และทำไมต้องเรียน (Step 809)

ใน Part 100 และ 101 เราสร้าง HTTP Server ขึ้นมาเองตั้งแต่ระดับ Socket ล้วนๆ ต้อง:

- เปิด TCP socket, `bind`, `listen`, `accept` เอง
- Parse HTTP Request แบบ raw text เอง (แยก Method, Path, Headers, Body)
- สร้าง Thread Pool เองเพื่อรองรับ concurrent connection
- ประกอบ HTTP Response string เอง (status line, headers, body) ให้ถูกต้องตาม spec

งานเหล่านี้คืองานที่ **ทุก Web Framework ในโลกทำให้อัตโนมัติ** ไม่ว่าจะเป็น Flask (Python),
Express (Node.js), Spring Boot (Java) หรือ Gin (Go) — และในโลกของ C++ หนึ่งใน Framework ที่ได้รับ
ความนิยมมากที่สุดคือ **Crow**

**Crow** (https://github.com/CrowCpp/Crow) คือ C++ microframework สำหรับสร้าง Web
Application/REST API ที่มีจุดเด่นคือ:

- **Header-only**: ไม่ต้อง compile เป็น `.so`/`.a` แยก เพียง `#include "crow.h"` ก็ใช้งานได้เลย
  (ถึงแม้จะเป็น header-only แต่ internally ใช้ `libasio` สำหรับ async I/O)
- **Syntax คล้าย Flask/Express มาก**: ใช้ decorator-style route ผ่าน macro `CROW_ROUTE`
  ทำให้นักพัฒนาที่มาจากภาษาอื่นเรียนรู้ได้เร็ว
- **เร็วกว่า Framework ภาษาอื่นมาก**: เพราะเป็น C++ ล้วน ไม่มี interpreter หรือ garbage collector
  มาคั่นกลาง benchmark สาธารณะหลายที่ (เช่น TechEmpower Framework Benchmarks) จัด Crow และ
  framework losertype เดียวกัน (Drogon, Pistache) อยู่ในกลุ่มเร็วที่สุดของโลก แซงหน้า Node.js/Django
  หลายเท่าตัวในสถานการณ์ CPU-bound
- **Built-in Routing, Middleware, JSON parsing** — ไม่ต้องเขียนเอง

### เปรียบเทียบ Crow กับ Flask/Express แบบคร่าวๆ

| แง่มุม | Crow (C++) | Flask (Python) | Express (Node.js) |
|---|---|---|---|
| ภาษา | C++ (compiled) | Python (interpreted) | JavaScript (V8, JIT) |
| Concurrency model | Multi-thread (asio) | Single-thread + WSGI/gunicorn | Event loop (single-thread) |
| Startup time | เร็วมาก (native binary) | ช้ากว่า (ต้อง import interpreter) | ปานกลาง |
| Memory footprint | ต่ำมาก | สูงกว่ามาก | สูงกว่า Crow |
| Throughput (requests/sec) | สูงมาก (แข่งกับ Go/Rust) | ต่ำกว่า Crow หลายเท่า | ต่ำกว่า Crow |
| ความง่ายในการเขียน | ง่าย (คล้าย Flask) | ง่ายที่สุด | ง่าย |
| Deploy | Binary เดียว ไม่ต้องมี runtime | ต้องมี Python runtime | ต้องมี Node.js runtime |

จุดที่ต้องเข้าใจให้ชัด: Crow ไม่ได้ "แทนที่" ความรู้เรื่อง Socket ที่เราเรียนใน Part 100-101 — มันแค่
**automate งานที่ซ้ำซาก** ให้เรา ความเข้าใจเรื่อง TCP, HTTP protocol, thread pool ที่ผ่านมายังจำเป็น
มากเวลาต้อง debug ปัญหาที่ลึกกว่าระดับ framework (เช่น connection leak, keep-alive ทำงานผิดปกติ)

---

## 102.2 การติดตั้ง Crow (Step 810)

Crow เป็น header-only library แต่ยังต้องอาศัย **Asio** (C++ networking library) เป็น dependency
เบื้องหลัง และแนะนำให้ build ผ่าน CMake เพื่อให้ CMake จัดการ dependency และ install header
ไปยังตำแหน่งมาตรฐานของระบบให้อัตโนมัติ

### ขั้นตอนที่ 1: ติดตั้ง Asio (dependency หลัก)

```bash
sudo apt update
sudo apt install -y libasio-dev
```

`libasio-dev` คือไลบรารี networking แบบ header-only ของ C++ ที่ Crow ใช้เป็นเครื่องยนต์ async I/O
เบื้องหลัง (คนละตัวกับ Boost.Asio แต่ API หน้าตาคล้ายกันมาก เพราะ Asio คือต้นฉบับที่ Boost.Asio
เอาไปใช้)

### ขั้นตอนที่ 2: Clone และ Build Crow จาก Source

```bash
git clone https://github.com/CrowCpp/Crow.git
cd Crow
mkdir build && cd build
cmake .. -DCROW_BUILD_EXAMPLES=OFF -DCROW_BUILD_TESTS=OFF
sudo cmake --install .
```

อธิบายทีละคำสั่ง:

- `git clone ...`: ดึง source code ของ Crow ล่าสุดจาก GitHub
- `cmake .. -DCROW_BUILD_EXAMPLES=OFF -DCROW_BUILD_TESTS=OFF`: configure โปรเจกต์ โดยปิดการ
  build ตัวอย่างและ test ของตัว Crow เอง (เราต้องการแค่ header กับไฟล์ config ของ CMake)
- `sudo cmake --install .`: ติดตั้ง header ไปที่ `/usr/local/include/crow/` และไฟล์รวม
  `/usr/local/include/crow.h` พร้อมไฟล์ `CrowConfig.cmake` สำหรับโปรเจกต์ที่ใช้ CMake เรียกใช้
  Crow ผ่าน `find_package(Crow)` ได้ในอนาคต

ตรวจสอบว่าติดตั้งสำเร็จ:

```bash
ls /usr/local/include/crow.h
ls /usr/local/include/crow/ | head
```

ถ้าเห็นไฟล์ `crow.h` และโฟลเดอร์ `crow/` ที่มีไฟล์อย่าง `app.h`, `routing.h`, `json.h` แสดงว่า
ติดตั้งสำเร็จแล้ว

### คอมไพล์โปรแกรมที่ใช้ Crow

เนื่องจาก Crow เป็น header-only เราไม่ต้อง link library ของ Crow เอง (ไม่มี `-lcrow`) แต่ยังต้อง
link `pthread` เพราะ Crow ใช้ multi-threading ภายใน:

```bash
g++ -std=c++17 -I/usr/local/include yourfile.cpp -o app -lpthread
```

> **หมายเหตุสำคัญ**: ทุกตัวอย่างในบทนี้ถูกคอมไพล์และรันจริงด้วยคำสั่งด้านบน บน g++ 13.3.0
> (Ubuntu 24.04) และทดสอบด้วย `curl` จริงทุกกรณี ผลลัพธ์ที่แสดงในบทเรียนคือ output จริงที่ได้
> จากการรัน ไม่ใช่ตัวอย่างสมมติ

---

## 102.3 Hello World ด้วย CROW_ROUTE (Step 811)

มาเขียนโปรแกรม Crow โปรแกรมแรกกัน สร้างไฟล์ `hello.cpp`:

```cpp
#include "crow.h"

int main() {
    crow::SimpleApp app;

    CROW_ROUTE(app, "/")([](){
        return "Hello, Crow!";
    });

    app.port(18080).run();
}
```

คอมไพล์และรัน:

```bash
g++ -std=c++17 -I/usr/local/include hello.cpp -o hello_crow -lpthread
./hello_crow
```

ผลลัพธ์บน terminal (log ของ Crow เอง):

```
(2026-09-26 09:10:23) [INFO    ] Crow/master server is running at http://0.0.0.0:18080 using 2 threads
(2026-09-26 09:10:23) [INFO    ] Call `app.loglevel(crow::LogLevel::Warning)` to hide Info level logs.
```

เปิด terminal อีกอันแล้วยิง request ด้วย `curl`:

```bash
curl -s -i http://127.0.0.1:18080/
```

ผลลัพธ์จริงที่ได้:

```
HTTP/1.1 200 OK
Content-Length: 12
Server: Crow/master
Connection: Keep-Alive

Hello, Crow!
```

และในฝั่ง server จะเห็น log อัตโนมัติ:

```
(2026-09-26 09:10:24) [INFO    ] Request: 127.0.0.1:49306 0x559a644dcfb0 HTTP/1.1 GET /
(2026-09-26 09:10:24) [INFO    ] Response: 0x559a644dcfb0 / 200 0
```

### อธิบายทีละส่วน

- **`crow::SimpleApp app;`**: สร้าง instance ของ application หลัก `SimpleApp` คือ alias ของ
  `Crow<>` ที่ไม่มี middleware ใดๆ ผูกอยู่ (จะพูดถึง middleware ใน Part 103)
- **`CROW_ROUTE(app, "/")`**: เป็น macro ที่ขยายออกมาเป็นการเรียก `app.route<...>("/")` โดย
  ใช้ template metaprogramming วิเคราะห์ path string ตอน compile-time เพื่อดึงชื่อ parameter
  ออกมาอัตโนมัติ (เดี๋ยวจะเห็นตอนใช้ route parameter) นี่คือเหตุผลที่ macro รูปแบบนี้เร็วกว่า
  framework ที่ทำ routing แบบ runtime string matching ล้วนๆ
- **`([](){ return "Hello, Crow!"; })`**: Lambda function ที่ทำหน้าที่เป็น Handler — ฟังก์ชันที่
  Crow จะเรียกเมื่อมี request เข้ามาตรง route นี้ ค่าที่ `return` จะถูกแปลงเป็น HTTP Response
  body โดยอัตโนมัติ (ในที่นี้ Crow แปลง `const char*`/`std::string` เป็น response แบบ `200 OK`
  ให้ทันที)
- **`app.port(18080).run()`**: กำหนด port ที่จะ listen แล้วเริ่มรัน server (blocking call —
  จะรันค้างอยู่ตรงนี้จนกว่าจะถูก interrupt)

สังเกตว่า log บอกว่า server รันด้วย **"2 threads"** โดย default (จำนวนขึ้นกับจำนวน core ของ
เครื่องที่ตรวจจับได้ผ่าน `std::thread::hardware_concurrency()`) — เทียบกับ Part 101 ที่เราต้องเขียน
Thread Pool เองทั้งหมด ตรงนี้ Crow จัดการให้อัตโนมัติทันทีที่เรียก `.run()`

สังเกตอีกจุดที่สำคัญ: **response ไม่มี header `Content-Type`** เลย เพราะเรา `return` เป็น
plain string ธรรมดา Crow จะไม่เดา MIME type ให้ ประเด็นนี้จะสำคัญมากตอนที่เราคืน JSON
(ดู 102.7)

### CROW_ROUTE ขยายออกมาเป็นอะไรกันแน่

การอ่านไฟล์ header จริงของ Crow (`/usr/local/include/crow/app.h`) ช่วยให้เข้าใจว่า macro
`CROW_ROUTE` ไม่ได้มีเวทมนตร์ซ่อนอยู่ — มันเป็นแค่ตัวช่วยย่อโค้ดที่ยาวให้สั้นลง:

```cpp
#define CROW_ROUTE(app, url) \
    app.template route<crow::black_magic::get_parameter_tag(url)>(url)
```

พูดง่ายๆ คือเมื่อเราเขียน `CROW_ROUTE(app, "/user/<int>")` มันจะถูกแทนที่ (ตอน preprocessing)
ด้วยโค้ดประมาณ `app.template route<TAG>("/user/<int>")` โดย `TAG` คือค่าที่คำนวณได้ตอน
**compile-time** จากการวิเคราะห์ string `"/user/<int>"` ว่ามี parameter กี่ตัว ชนิดอะไรบ้าง
(ฟังก์ชัน `get_parameter_tag` เป็น `constexpr` function ที่ทำงานตอน compile ไม่ใช่ runtime)
นี่คือเหตุผลที่ Crow ตรวจจับความไม่ตรงกันระหว่าง path parameter กับ lambda parameter ได้ตั้งแต่
ตอน compile แทนที่จะเป็น runtime error เหมือน framework ที่ทำ routing แบบ string parsing ล้วนๆ
ตอน request เข้ามาจริง — เป็นตัวอย่างที่ดีของการใช้ Template Metaprogramming (เนื้อหาที่จะเรียน
ละเอียดใน Part 56-57 และ Part 78) เพื่อย้ายงานตรวจสอบจาก runtime ไปเป็น compile-time

---

## 102.4 Route Parameter (Path Variable) (Step 812)

ในเว็บแอปจริง เรามักต้องดึงค่าจาก URL เช่น `/user/42` เพื่อดึงข้อมูล user ที่มี id เท่ากับ 42
Crow รองรับสิ่งนี้ผ่าน syntax พิเศษในตัว path string โดยตรง:

```cpp
#include "crow.h"

int main() {
    crow::SimpleApp app;

    // Route parameter: /user/<int>
    CROW_ROUTE(app, "/user/<int>")
    ([](int id){
        return "User ID: " + std::to_string(id);
    });

    // Route parameter: string
    CROW_ROUTE(app, "/greet/<string>")
    ([](std::string name){
        return "Hello, " + name + "!";
    });

    // Multiple parameters
    CROW_ROUTE(app, "/add/<int>/<int>")
    ([](int a, int b){
        return crow::response(std::to_string(a + b));
    });

    app.port(18081).run();
}
```

จุดสำคัญ: **ลำดับ parameter ใน path ต้องตรงกับลำดับ parameter ของ lambda เป๊ะๆ** และชนิดของ
parameter (`<int>`, `<string>`, `<double>`, `<uint>`, `<path>`) ต้องตรงกับ type ที่ lambda รับ —
Crow ใช้ compile-time template magic ตรวจสอบสิ่งนี้ ถ้าไม่ตรง จะได้ **compile error** ทันที
ไม่ใช่ runtime error แบบ framework ภาษาอื่น นี่คือข้อได้เปรียบใหญ่ของการใช้ static-typed language

ทดสอบจริง:

```bash
curl -s http://127.0.0.1:18081/user/42
# User ID: 42

curl -s http://127.0.0.1:18081/greet/Somchai
# Hello, Somchai!

curl -s http://127.0.0.1:18081/add/3/4
# 7

curl -s -o /dev/null -w "%{http_code}\n" http://127.0.0.1:18081/user/notanumber
# 404
```

สังเกตกรณีสุดท้าย: เมื่อยิง `/user/notanumber` (ที่ตำแหน่ง `<int>` ใส่ string มา) Crow จะ
**ไม่ match route นี้เลย** และตอบกลับ `404 Not Found` แทนที่จะเป็น error 500 หรือ crash — เพราะ
`<int>` ในตัว routing table ทำหน้าที่เป็นทั้ง type constraint และ pattern matching ไปพร้อมกัน

### ตัวเลือก Path Parameter ทั้งหมดของ Crow

| Syntax | ชนิดข้อมูลที่ได้ | ตัวอย่าง |
|---|---|---|
| `<int>` | `int64_t` | `/user/<int>` |
| `<uint>` | `uint64_t` | `/count/<uint>` |
| `<double>` | `double` | `/price/<double>` |
| `<string>` | `std::string` (segment เดียว ไม่มี `/`) | `/greet/<string>` |
| `<path>` | `std::string` (จับ path ที่เหลือทั้งหมดรวม `/`) | `/files/<path>` |

---

## 102.5 HTTP Method ต่างๆ ด้วย .methods() (Step 813)

ที่ผ่านมาทุก route รับเฉพาะ `GET` (ค่า default ของ `CROW_ROUTE`) ถ้าต้องการรับ Method อื่น เช่น
`POST`, `PUT`, `DELETE` ต้องเรียก `.methods(...)` ต่อท้าย:

```cpp
CROW_ROUTE(app, "/echo").methods(crow::HTTPMethod::POST)
([](const crow::request& req){
    return "You sent: " + req.body;
});
```

หรือรับได้หลาย Method พร้อมกันในตัวเดียว โดยใช้ comma แยก:

```cpp
CROW_ROUTE(app, "/resource").methods(crow::HTTPMethod::GET, crow::HTTPMethod::POST)
([](const crow::request& req){
    if (req.method == crow::HTTPMethod::GET) {
        return crow::response("GET request");
    }
    return crow::response("POST request");
});
```

ถ้ายิง Method ที่ route ไม่รองรับ (เช่นยิง `DELETE` ไปที่ route ที่รับแค่ `GET`/`POST`) Crow จะ
ตอบกลับ `405 Method Not Allowed` โดยอัตโนมัติ โดยไม่ต้องเขียน logic เช็คเองเลย ลองทดสอบจริง:

```bash
curl -s -i -X DELETE http://127.0.0.1:18087/resource
```

ผลลัพธ์จริงที่ได้จากเครื่องนี้:

```
HTTP/1.1 405 Method Not Allowed
Content-Length: 24
Server: Crow/master
Connection: Keep-Alive

405 Method Not Allowed
```

**ข้อสังเกตที่ควรรู้ (จากการทดสอบจริง ไม่ใช่จากเอกสาร)**: เวอร์ชัน Crow ที่ติดตั้งบนเครื่องนี้
**ไม่ได้แนบ header `Allow`** มาบอกว่า Method ไหนที่ route นี้รองรับจริงๆ (บาง tutorial หรือ
เอกสารเก่าของ Crow อาจกล่าวถึง header นี้ แต่พฤติกรรมจริงที่ทดสอบได้คือ body เป็นข้อความ
`405 Method Not Allowed` เปล่าๆ เท่านั้น) ถ้าต้องการให้ client รู้ว่า Method ไหนที่ใช้ได้จริง
ต้อง**เขียนเอกสาร API แยกต่างหาก** หรือทำ custom error handler เพิ่มเติม — นี่เป็นตัวอย่างที่ดี
ว่าเหตุใดการทดสอบพฤติกรรมจริงของ library เวอร์ชันที่ใช้อยู่จึงสำคัญกว่าการเชื่อเอกสารหรือ
ความจำล้วนๆ เสมอ

---

## 102.6 การอ่าน Query String และ Request Body (Step 814)

ใน Handler ทุกตัว เราสามารถรับ `const crow::request& req` เป็น parameter เพื่อเข้าถึงข้อมูลของ
request ปัจจุบันได้ (path parameter จาก URL ยังคงส่งเป็น parameter แยกตามปกติ ถ้ามี)

### อ่าน Query String

```cpp
CROW_ROUTE(app, "/search")
([](const crow::request& req){
    auto q = req.url_params.get("q");
    std::string keyword = q ? q : "(ไม่ได้ระบุ)";
    return "ค้นหา: " + keyword;
});
```

`req.url_params.get("q")` คืนค่า `const char*` — ถ้าไม่มี query parameter ชื่อนั้นจริงๆ จะได้
`nullptr` กลับมา จึงต้องเช็ค `nullptr` ก่อนใช้งานเสมอ (เป็นจุดที่มือใหม่มักลืมแล้วโดน segfault
ถ้าไปเรียก `.get("q")` เข้ากับ `std::string` ตรงๆ โดยไม่เช็คก่อน)

### อ่าน Request Body

```cpp
CROW_ROUTE(app, "/echo").methods(crow::HTTPMethod::POST)
([](const crow::request& req){
    return "Body length: " + std::to_string(req.body.size())
         + ", content: " + req.body;
});
```

`req.body` คือ `std::string` ที่เก็บ raw body ทั้งหมดของ request (Crow อ่านมาให้ครบตาม
`Content-Length` แล้ว ไม่ต้องอ่านจาก socket เองทีละ chunk แบบใน Part 100)

---

## 102.7 คืนค่า JSON Response ด้วย crow::json::wvalue (Step 815)

REST API ยุคใหม่แทบทั้งหมดสื่อสารด้วย JSON Crow มี type พิเศษชื่อ `crow::json::wvalue`
("writable value") สำหรับสร้าง JSON object ขึ้นมาแล้วส่งกลับเป็น response ได้โดยตรง

มาดูตัวอย่างที่ผสมทั้ง query string, POST body, และ JSON response เข้าด้วยกัน:

```cpp
#include "crow.h"

int main() {
    crow::SimpleApp app;

    // GET กับ query string
    CROW_ROUTE(app, "/search")
    ([](const crow::request& req){
        auto q = req.url_params.get("q");
        std::string keyword = q ? q : "(ไม่ได้ระบุ)";
        crow::json::wvalue result;
        result["query"] = keyword;
        result["status"] = "ok";
        return result;
    });

    // POST อ่าน body และคืน JSON
    CROW_ROUTE(app, "/echo").methods(crow::HTTPMethod::POST)
    ([](const crow::request& req){
        crow::json::wvalue result;
        result["received"] = req.body;
        result["length"] = req.body.size();
        return crow::response(200, result);
    });

    // GET คืน JSON object/array ผสมกัน
    CROW_ROUTE(app, "/users")
    ([](){
        crow::json::wvalue result;
        result["users"][0]["id"] = 1;
        result["users"][0]["name"] = "Somchai";
        result["users"][1]["id"] = 2;
        result["users"][1]["name"] = "Somsri";
        result["count"] = 2;
        return result;
    });

    app.port(18082).run();
}
```

ทดสอบจริงด้วย `curl -i` (แสดง header ด้วย เพื่อดู `Content-Type`):

```bash
curl -s -i "http://127.0.0.1:18082/search?q=crow"
```

ผลลัพธ์จริง:

```
HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 30
Server: Crow/master
Connection: Keep-Alive

{"status":"ok","query":"crow"}
```

```bash
curl -s -i -X POST -d "hello from client" http://127.0.0.1:18082/echo
```

ผลลัพธ์จริง:

```
HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 44
Server: Crow/master
Connection: Keep-Alive

{"length":17,"received":"hello from client"}
```

```bash
curl -s -i http://127.0.0.1:18082/users
```

ผลลัพธ์จริง:

```
HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 72
Server: Crow/master
Connection: Keep-Alive

{"count":2,"users":[{"id":1,"name":"Somchai"},{"name":"Somsri","id":2}]}
```

### ข้อสังเกตสำคัญจากผลลัพธ์จริง

1. **`Content-Type: application/json` ถูกเซ็ตให้อัตโนมัติ** เมื่อ handler `return` เป็น
   `crow::json::wvalue` โดยตรง (ไม่ว่าจะ return ตรงๆ หรือห่อด้วย `crow::response(200, result)`)
   ต่างจากตอน return `std::string`/`const char*` ที่จะไม่มี header นี้เลย (ดู 102.3)
2. **ลำดับ key ใน JSON ที่ได้ไม่การันตีว่าจะตรงกับลำดับที่เราใส่ในโค้ด** — สังเกตจาก
   `/users` ที่ user ตัวแรก (`id` มาก่อน `name`) แต่ user ตัวที่สองกลับได้ `name` มาก่อน `id`
   เพราะ `crow::json::wvalue` เก็บ field แบบ hash-based internally การเขียนโค้ดที่พึ่งพา
   ลำดับ key ของ JSON output จึงเป็นความคิดที่ผิด (ฝั่ง client ที่ดีต้องอ่าน JSON โดยใช้ชื่อ
   key เสมอ ไม่ใช่ตำแหน่ง)
3. **`req.body.size()` นับเป็น byte** ในตัวอย่างข้างบน ข้อความ `"hello from client"` มี 18
   ตัวอักษร แต่ผลลัพธ์บอก `length: 17` เพราะ `curl -d` จะตัดตัวอักษรสุดท้ายที่เป็น
   whitespace/newline ตามพฤติกรรม default ของ `curl` เอง (ไม่ใช่บั๊กของ Crow) — เป็นตัวอย่าง
   ที่ดีว่าเวลา debug เรื่อง byte count ต้องเข้าใจเครื่องมือที่ใช้ยิง request ด้วย ไม่ใช่โทษ
   framework ทันที

---

### เจาะลึก crow::response: กำหนด Header เอง, Redirect, และ Query Parameter หลายตัว

นอกจาก `return` ค่าที่เป็น string หรือ `wvalue` ตรงๆ เรายังสร้าง `crow::response` ขึ้นมาเองแบบ
เต็มรูปแบบเพื่อควบคุม status code, body, และ header ได้ทีละส่วนอย่างละเอียด:

```cpp
#include "crow.h"

int main() {
    crow::SimpleApp app;

    // สร้าง response แบบเจาะลึก: กำหนด code, body, header เอง
    CROW_ROUTE(app, "/custom")
    ([](){
        crow::response res;
        res.code = 201;
        res.body = "created!";
        res.set_header("X-Custom-Header", "hello-world");
        res.set_header("Content-Type", "text/plain; charset=utf-8");
        return res;
    });

    // Redirect
    CROW_ROUTE(app, "/old-page")
    ([](){
        crow::response res;
        res.redirect("/new-page");
        return res;
    });

    CROW_ROUTE(app, "/new-page")
    ([](){
        return "นี่คือหน้าใหม่";
    });

    // หลาย query parameter พร้อมกัน + default value
    CROW_ROUTE(app, "/paginate")
    ([](const crow::request& req){
        auto page_p = req.url_params.get("page");
        auto limit_p = req.url_params.get("limit");
        int page = page_p ? std::stoi(page_p) : 1;
        int limit = limit_p ? std::stoi(limit_p) : 10;
        crow::json::wvalue result;
        result["page"] = page;
        result["limit"] = limit;
        return result;
    });

    app.port(18086).run();
}
```

ทดสอบจริงทุกกรณี:

```bash
curl -s -i http://127.0.0.1:18086/custom
```

```
HTTP/1.1 201 Created
Content-Type: text/plain; charset=utf-8
X-Custom-Header: hello-world
Content-Length: 8
Server: Crow/master
Connection: Keep-Alive

created!
```

```bash
curl -s -i http://127.0.0.1:18086/old-page
```

```
HTTP/1.1 307 Temporary Redirect
Location: /new-page
Content-Length: 0
Server: Crow/master
Connection: Keep-Alive
```

**ข้อสังเกตจริงที่ทดสอบได้**: `res.redirect(...)` ของ Crow ใช้ status code **`307 Temporary
Redirect`** (ไม่ใช่ `301 Moved Permanently` หรือ `302 Found` ที่บาง framework อื่นใช้เป็น
default) ความแตกต่างนี้สำคัญในทางปฏิบัติเพราะ `307` บอก client (และ browser) ว่า **ห้าม
เปลี่ยน HTTP Method ตอน follow redirect** (ต่างจาก `302` ที่ browser บางตัวจะเปลี่ยน POST
เป็น GET โดยอัตโนมัติ) ถ้าต้องการ `301`/`302` แทน ต้องตั้งค่า `res.code` เองหลังเรียก
`redirect()` แล้วค่อยเซ็ต header `Location` เพิ่มเติม

ทดสอบตาม redirect ด้วย `curl -L`:

```bash
curl -s -i -L http://127.0.0.1:18086/old-page
```

```
HTTP/1.1 307 Temporary Redirect
Location: /new-page
Content-Length: 0
Server: Crow/master
Connection: Keep-Alive

HTTP/1.1 200 OK
Content-Length: 42
Server: Crow/master
Connection: Keep-Alive

นี่คือหน้าใหม่
```

และทดสอบ query parameter หลายตัวพร้อม default value:

```bash
curl -s http://127.0.0.1:18086/paginate
# {"limit":10,"page":1}

curl -s "http://127.0.0.1:18086/paginate?page=2&limit=20"
# {"limit":20,"page":2}
```

รูปแบบ "อ่าน query param แล้วถ้าไม่มีให้ใช้ค่า default" แบบนี้ (`page_p ? std::stoi(page_p) : 1`)
เป็น pattern ที่พบบ่อยมากในการเขียน REST API สำหรับ endpoint ที่รองรับ pagination
(`page`, `limit`, `offset` ฯลฯ) ควรจำรูปแบบนี้ไว้ใช้ซ้ำ

---

## 102.8 เทียบกับ Raw Server จาก Part 100-101 (Step 816)

ตอนนี้เราเขียนโปรแกรมที่ทำสิ่งเดียวกับ Part 100-101 ได้ (รับ HTTP request, parse, ตอบกลับ) แต่
โค้ดสั้นกว่ามาก มาสรุปเป็นตารางว่า Crow ช่วยประหยัดงานตรงไหนบ้าง:

| งานที่ต้องทำ | Raw Socket Server (Part 100-101) | Crow (Part 102) |
|---|---|---|
| เปิด TCP socket, bind, listen | เขียนเองด้วย `socket()`, `bind()`, `listen()` | `app.port(N).run()` ทำให้หมด |
| Accept connection | `accept()` แบบ loop เอง | จัดการภายใน asio event loop |
| Concurrency / Thread Pool | สร้าง Thread Pool เอง, จัดการ queue งานเอง | มี thread pool ในตัว ปรับจำนวนด้วย `.concurrency(n)` |
| Parse HTTP Request (method, path, headers, body) | Parse string ทีละ byte เอง เสี่ยง edge case เยอะ | `crow::request` แยกให้ครบ (`method`, `url`, `headers`, `body`) |
| Routing (จับคู่ path กับ handler) | เขียน `if/else` หรือ string matching เอง | `CROW_ROUTE` + path parameter จับคู่ให้อัตโนมัติ |
| Path parameter (`/user/42`) | ต้อง parse string เอง แล้วแปลง type เอง | ประกาศ type ใน path ตรงๆ (`<int>`) ได้ type-safe ตั้งแต่ compile-time |
| สร้าง HTTP Response (status line, headers) | ประกอบ string เองให้ตรง HTTP spec | `crow::response`/`return` string หรือ JSON — Crow จัดรูปแบบให้ |
| Method Not Allowed (405) | ต้องเช็คเองและ generate response เอง | อัตโนมัติเมื่อใช้ `.methods()` |
| JSON encode/decode | ต้องพึ่ง JSON library เองแล้วเขียน glue code | มี `crow::json::wvalue`/`crow::json::rvalue` ในตัว |
| Logging request/response | เขียนเอง | มี logger ในตัว (`CROW_LOG_INFO` ฯลฯ) |

สิ่งที่ **Crow ไม่ได้ทำแทนความเข้าใจพื้นฐาน**: TCP handshake, HTTP protocol เบื้องหลัง,
การจัดการ connection แบบ keep-alive, และปัญหาเรื่อง thread-safety ของ shared state (เมื่อหลาย
request เข้ามาพร้อมกันในหลาย thread) — เรื่องเหล่านี้ยังคงเป็นความรู้ที่ต้องใช้ตลอดเวลาที่เขียน
Web Backend ด้วย C++ ไม่ว่าจะใช้ framework ไหนก็ตาม และจะกลับมาเป็นประเด็นสำคัญใน Part 103
ตอนที่เราต้องแชร์ state (เช่น in-memory database) ระหว่าง worker thread ของ Crow

---

## ข้อผิดพลาดที่พบบ่อย (Common Pitfalls)

1. **ลืม `-lpthread` ตอนคอมไพล์** — Crow ใช้ multi-threading ภายใน (ผ่าน asio) หลักการที่
   ถูกต้องคือควร link `pthread` เข้าไปด้วยเสมอเมื่อ compile โปรแกรมที่ใช้ thread
   ในทางปฏิบัติบน Ubuntu 24.04 (glibc 2.39) ทดสอบจริงแล้วพบว่าโปรแกรม Crow **ยัง compile และ
   รันได้ปกติแม้ไม่ใส่ `-lpthread`** เพราะ glibc ตั้งแต่เวอร์ชัน 2.34 เป็นต้นมาได้รวมฟังก์ชัน
   ของ `libpthread` เข้าไปอยู่ใน `libc` หลักแล้ว ทำให้ linker หา symbol เจอโดยไม่ต้องระบุ flag
   นี้ตรงๆ อย่างไรก็ตาม **ควรใส่ `-lpthread` ไว้เสมอเป็นนิสัยที่ดี** เพราะโค้ดเดียวกันอาจ
   compile ไม่ผ่านบนระบบอื่นที่ใช้ glibc เวอร์ชันเก่ากว่า 2.34 หรือใช้ C library อื่น (เช่น
   musl libc บน Alpine Linux) ซึ่งยังคงแยก `pthread` ออกจาก `libc` อยู่ — การพึ่งพาพฤติกรรม
   เฉพาะของระบบปัจจุบันโดยไม่ระบุ flag ให้ชัดเจนคือความเสี่ยงเรื่อง portability ที่ควรหลีกเลี่ยง
2. **ลืมเช็ค `nullptr` จาก `req.url_params.get(...)`** — ถ้า query parameter ไม่มีอยู่จริง
   ฟังก์ชันนี้คืน `nullptr` การเอาไปสร้าง `std::string` ตรงๆ โดยไม่เช็คก่อนจะทำให้เกิด
   Undefined Behavior (segfault) ทันที
3. **ลำดับ/ชนิดของ Path Parameter ไม่ตรงกับ Lambda parameter** — เช่นประกาศ
   `"/user/<int>/<string>"` แต่ lambda เขียน `[](std::string name, int id)` (สลับลำดับ)
   จะทำให้ compile error หรือได้ค่าผิดประเภทโดยไม่รู้ตัว ต้องเช็คลำดับให้ตรงกันเสมอ
4. **คิดว่า return `std::string` จะได้ `Content-Type: text/plain` อัตโนมัติ** — ความจริงคือ
   Crow **ไม่เซ็ต Content-Type ให้เลย** ถ้า client (เช่น browser หรือ JS `fetch`) คาดหวัง
   `text/plain`/`application/json` ต้องเซ็ตเองผ่าน `res.set_header("Content-Type", ...)`
   หรือคืนเป็น `crow::json::wvalue` (ซึ่ง Crow จะเซ็ต `application/json` ให้อัตโนมัติเฉพาะกรณีนี้)
5. **ลืมว่า `.run()` เป็น blocking call** — โค้ดหลังบรรทัด `app.port(N).run()` จะไม่ถูกเรียกจนกว่า
   server จะถูกปิด (เช่นกด Ctrl+C) ถ้าต้องการรัน logic อื่นควบคู่ไปด้วยต้องใช้ `.run_async()`
   หรือรัน logic นั้นใน thread แยก
6. **สร้าง route ซ้ำ path เดิมโดยไม่ระวัง method** — ถ้าประกาศ `CROW_ROUTE(app, "/x")` สองครั้ง
   (โดยไม่ระบุ method ต่างกัน) จะเกิด error ตอน runtime เพราะ Crow จะตรวจพบว่ามี rule ชนกัน
   ในตำแหน่งเดียวกัน
7. **เชื่อว่า `405 Method Not Allowed` จะมี header `Allow` แนบมาด้วยเสมอ** — จากการทดสอบจริง
   บนเวอร์ชันนี้ Crow ตอบแค่ body ข้อความเปล่าๆ ไม่มี header `Allow` บอก Method ที่รองรับจริง
   ถ้า client ต้องการทราบข้อมูลนี้ ต้องพึ่งเอกสาร API หรือเขียน custom handler เพิ่มเติมเอง
8. **ใช้ `std::stoi`/`std::stod` กับ query parameter โดยไม่ครอบ try/catch** — แม้ path
   parameter แบบ `<int>` จะ type-safe ตั้งแต่ต้น แต่ query parameter (`req.url_params.get(...)`)
   เป็น `const char*` ดิบๆ เสมอ ถ้า client ส่งค่าที่แปลงเป็นตัวเลขไม่ได้ (เช่น `?page=abc`)
   `std::stoi` จะ throw `std::invalid_argument` ทำให้โปรแกรม crash ถ้าไม่ได้ครอบด้วย
   `try/catch` หรือ `exception_handler` ไว้
9. **เข้าใจผิดว่า `res.redirect(...)` ใช้ `302 Found`** — จากการทดสอบจริง Crow ใช้
   `307 Temporary Redirect` เป็นค่า default ซึ่งมีความหมายต่างจาก `302`/`301` ในแง่การรักษา
   HTTP Method เดิมตอน follow redirect ถ้าต้องการพฤติกรรมอื่นต้องกำหนด `res.code` เอง

---

## แบบฝึกหัดท้ายบท

1. เขียนโปรแกรม Crow ที่มี route `/square/<int>` รับตัวเลขแล้วคืนค่ากำลังสองของมันกลับไปเป็น
   ข้อความธรรมดา (เช่น `/square/5` ตอบ `25`)
2. เขียน route `/hello` ที่รับ query parameter ชื่อ `name` (เช่น `/hello?name=Somchai`) แล้วคืน
   JSON `{"message": "Hello, Somchai!"}` ถ้าไม่มี `name` ให้ตอบ `{"message": "Hello, Guest!"}`
3. เขียน route `POST /calc` ที่รับ body เป็นตัวเลขสองตัวคั่นด้วย comma (เช่น `"3,4"`) แล้วคืน
   JSON ที่มี field `sum`, `diff`, `product` ของสองจำนวนนั้น
4. ทดลองสร้าง route ที่รับได้ทั้ง `GET` และ `POST` ในตัวเดียว (`/info`) แล้วให้ตอบข้อความต่างกัน
   ตาม Method ที่ใช้ยิงเข้ามา ทดสอบด้วย `curl -X GET` และ `curl -X POST` ว่าตอบต่างกันจริง
5. ทดลองยิง Method ที่ route ไม่รองรับ (เช่น `curl -X DELETE` ไปที่ route ที่รับแค่ GET) แล้ว
   สังเกตด้วย `curl -i` ว่า status code ที่ได้คืออะไร และมี header `Allow` แนบมาด้วยจริงหรือไม่
   ในเวอร์ชัน Crow ที่ติดตั้งอยู่บนเครื่องของตัวเอง
6. เขียน route `/files/<path>` แล้วทดลองยิง URL ที่มี `/` หลายชั้น (เช่น `/files/a/b/c.txt`)
   เปรียบเทียบกับการใช้ `<string>` แทน `<path>` ว่าพฤติกรรมต่างกันอย่างไร
7. เขียน route `/redirect-demo` ที่ใช้ `res.redirect(...)` ไปยัง route อื่น แล้วใช้
   `curl -i` (ไม่ใส่ `-L`) ตรวจสอบว่า status code ที่ได้คือ `307` จริงหรือไม่ ตามที่บทเรียน
   อธิบายไว้

### แนวทางเฉลยข้อ 1

```cpp
#include "crow.h"

int main() {
    crow::SimpleApp app;

    CROW_ROUTE(app, "/square/<int>")
    ([](int n){
        long long result = static_cast<long long>(n) * n;
        return std::to_string(result);
    });

    app.port(18090).run();
}
```

ทดสอบ:

```bash
g++ -std=c++17 -I/usr/local/include ex1.cpp -o ex1 -lpthread
./ex1 &
curl -s http://127.0.0.1:18090/square/5
# 25
curl -s http://127.0.0.1:18090/square/-3
# 9
```

ใช้ `long long` เก็บผลคูณเพื่อป้องกัน overflow กรณีมีคนยิงเลขจำนวนเต็มขนาดใหญ่เข้ามา
(`<int>` ของ Crow map เป็น `int64_t` ภายใน แต่ผลคูณของสองค่า int64_t อาจ overflow ได้เช่นกัน
ในระบบจริงควรเช็ค range ก่อนคูณด้วย)

### แนวทางเฉลยข้อ 3

```cpp
#include "crow.h"
#include <sstream>

int main() {
    crow::SimpleApp app;

    CROW_ROUTE(app, "/calc").methods(crow::HTTPMethod::POST)
    ([](const crow::request& req){
        std::stringstream ss(req.body);
        std::string a_str, b_str;
        if (!std::getline(ss, a_str, ',') || !std::getline(ss, b_str)) {
            return crow::response(400, "รูปแบบต้องเป็น \"a,b\"");
        }

        double a, b;
        try {
            a = std::stod(a_str);
            b = std::stod(b_str);
        } catch (const std::exception&) {
            return crow::response(400, "ตัวเลขไม่ถูกต้อง");
        }

        crow::json::wvalue result;
        result["sum"] = a + b;
        result["diff"] = a - b;
        result["product"] = a * b;
        return crow::response(200, result);
    });

    app.port(18091).run();
}
```

ทดสอบ:

```bash
g++ -std=c++17 -I/usr/local/include ex3.cpp -o ex3 -lpthread
./ex3 &
curl -s -X POST -d "3,4" http://127.0.0.1:18091/calc
# {"diff":-1.0,"product":12.0,"sum":7.0}

curl -s -i -X POST -d "abc" http://127.0.0.1:18091/calc
# HTTP/1.1 400 Bad Request
# รูปแบบต้องเป็น "a,b"
```

จุดสำคัญของเฉลยนี้: เราตรวจสอบทั้งกรณี body ไม่มี comma และกรณีตัวเลขแปลงไม่ได้ (`std::stod`
throw exception) แล้วตอบ `400 Bad Request` แทนที่จะปล่อยให้โปรแกรม crash — นี่คือหลักการพื้นฐาน
ของการเขียน REST API ที่ปลอดภัย: **ไม่เชื่อ input จากภายนอกเด็ดขาด** ต้อง validate ก่อนใช้งานเสมอ

### แนวทางเฉลยข้อ 4

```cpp
#include "crow.h"

int main() {
    crow::SimpleApp app;

    CROW_ROUTE(app, "/info").methods(crow::HTTPMethod::GET, crow::HTTPMethod::POST)
    ([](const crow::request& req){
        if (req.method == crow::HTTPMethod::GET) {
            return crow::response("นี่คือ GET request");
        }
        return crow::response("นี่คือ POST request พร้อม body: " + req.body);
    });

    app.port(18095).run();
}
```

ทดสอบจริง:

```bash
g++ -std=c++17 -I/usr/local/include ex4.cpp -o ex4 -lpthread
./ex4 &

curl -s -X GET http://127.0.0.1:18095/info
# นี่คือ GET request

curl -s -X POST -d "test data" http://127.0.0.1:18095/info
# นี่คือ POST request พร้อม body: test data
```

จุดสำคัญของเฉลยนี้: ใช้ `req.method` เปรียบเทียบกับค่า enum `crow::HTTPMethod::GET` เพื่อแยก
พฤติกรรมของ handler ตัวเดียวตาม Method ที่ใช้เรียกเข้ามา แสดงให้เห็นว่า route เดียวสามารถทำ
หน้าที่ต่างกันโดยสิ้นเชิงตาม Method ได้ — เทคนิคนี้มีประโยชน์เมื่อต้องการให้ path เดียวกัน
(เช่น `/resource`) ทำหน้าที่ทั้งอ่านข้อมูล (GET) และสร้างข้อมูลใหม่ (POST) ตามหลักการออกแบบ
RESTful API

---

## สรุปท้ายบท

ใน Part นี้เราได้:

- เข้าใจว่า Crow คือ C++ microframework แบบ header-only ที่เร็วกว่า framework ภาษาอื่นมาก
  เพราะไม่มี interpreter/GC มาคั่น
- ติดตั้ง Crow บนเครื่องจริงผ่าน `libasio-dev` + CMake build/install และคอมไพล์โปรแกรมที่ใช้
  Crow ได้สำเร็จด้วย `g++ -std=c++17 -I/usr/local/include ... -lpthread`
- เขียนและรัน Hello World Server ด้วย `CROW_ROUTE` ได้จริง พร้อมทดสอบด้วย `curl` และอ่าน log
  ของ Crow ได้
- ใช้ Route Parameter (`<int>`, `<string>`, `<path>`) เพื่อดึงค่าจาก URL แบบ type-safe
- กำหนด HTTP Method ด้วย `.methods()` และเข้าใจพฤติกรรม `405 Method Not Allowed` อัตโนมัติ
- อ่าน Query String และ Request Body จาก `crow::request` ได้
- สร้าง JSON Response ด้วย `crow::json::wvalue` และเข้าใจว่า `Content-Type` ถูกเซ็ตอัตโนมัติ
  เฉพาะกรณีคืนค่าเป็น JSON เท่านั้น
- เปรียบเทียบได้ชัดเจนว่า Crow ช่วยลดงานซ้ำซากจาก Raw Socket Server ตรงไหนบ้าง และอะไรที่
  ยังคงต้องเข้าใจเองเสมอไม่ว่าจะใช้ framework ไหน

ใน **Part 103** เราจะเจาะลึก Crow ต่อในระดับที่ซับซ้อนขึ้น: **Middleware** (logic ที่รันก่อน/หลัง
handler ทุก request เช่น logging, CORS), การ parse JSON request body ด้วย `crow::json::rvalue`,
การทำ custom error response, `Blueprint` สำหรับจัดกลุ่ม route, และการสร้าง REST API ที่มีหลาย
endpoint ทำงานร่วมกับ in-memory data store ที่ป้องกันด้วย mutex อย่างถูกต้อง

**ต่อไป:** [Part 103 — Crow ขั้นสูง: Middleware, Routing, JSON](./part-103-crow-advanced.md)
