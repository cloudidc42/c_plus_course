# Part 104: Pistache Framework: ทางเลือกสำหรับ REST API ระดับ Production (Step 825–832)

> Module I — Web Development ด้วย C/C++ | Part 104 จาก 125
> Step ที่ครอบคลุมใน Part นี้: Step 825–832
> Part ก่อนหน้า: [Part 103 — Crow ขั้นสูง: Middleware, Routing, JSON](./part-103-crow-advanced.md) | Part ถัดไป: [Part 105 — เชื่อมต่อฐานข้อมูล SQLite ด้วย C++](./part-105-sqlite-cpp.md)

## เป้าหมายการเรียนรู้ (Learning Objectives)

หลังจบ Part นี้ ผู้เรียนจะสามารถ:

1. อธิบายได้ว่า Pistache คืออะไร และต่างจาก Crow อย่างไรในเชิงสถาปัตยกรรม (async I/O model,
   ไม่ใช่ header-only, เน้น production performance)
2. ติดตั้ง Pistache ผ่าน `apt` บนเครื่องของตัวเองได้ พร้อมเข้าใจ dependency ที่เกี่ยวข้อง
   (`rapidjson-dev`)
3. เขียนและรัน HTTP Server พื้นฐานด้วย `Http::Endpoint` ได้จริง
4. ใช้ `Rest::Router` เพื่อกำหนดเส้นทาง (route) แบบ REST-style ได้
5. เขียน REST endpoint ที่รองรับ GET/POST พร้อม path parameter ด้วย syntax ของ Pistache
6. ปรับจำนวน worker thread ของ Pistache ผ่าน `Http::Endpoint::options().threads()`
7. Port endpoint บางส่วนจาก Task API ของ Part 103 (ที่เขียนด้วย Crow) มาเขียนใหม่ด้วย
   Pistache + rapidjson ได้สำเร็จ และเปรียบเทียบผลลัพธ์จริงระหว่างสองเวอร์ชัน
8. เปรียบเทียบ Crow กับ Pistache แบบตรงไปตรงมา (ความง่ายในการเขียน vs ประสิทธิภาพ/ความยืดหยุ่น
   ระดับ production) และตัดสินใจได้ว่าควรเลือกใช้ตัวไหนในสถานการณ์ต่างๆ

---

## 104.1 Pistache คืออะไร ต่างจาก Crow อย่างไร (Step 825)

**Pistache** (https://github.com/pistacheio/pistache) คือ C++ REST Framework อีกตัวหนึ่งที่ได้
รับความนิยมในสาย production เนื่องจากออกแบบมาโดยเน้น **ประสิทธิภาพและการควบคุมระดับต่ำ**
มากกว่า Crow ตั้งแต่แรกเริ่ม

### ความแตกต่างเชิงสถาปัตยกรรมที่สำคัญ

| แง่มุม | Crow | Pistache |
|---|---|---|
| รูปแบบไลบรารี | Header-only (`#include "crow.h"` จบ) | ต้อง compile เป็น shared/static library แยก แล้ว `-lpistache` ตอน link |
| I/O Model | ใช้ asio (thread-per-connection แบบ async) | ใช้ epoll โดยตรงบน Linux (reactor pattern) ควบคุม I/O ระดับต่ำกว่า |
| Routing | Template metaprogramming, compile-time path parsing | Runtime routing ผ่าน `Rest::Router` (ยืดหยุ่นกว่าแต่ overhead มากกว่าเล็กน้อยตอน match route) |
| JSON | มี `crow::json` ในตัว (wvalue/rvalue) | ไม่มี JSON parser ในตัว ต้องพึ่ง library ภายนอก (เช่น rapidjson, nlohmann/json) |
| Syntax | คล้าย Flask/Express มาก อ่านง่าย เขียนเร็ว | ใกล้เคียง C++ ดิบมากกว่า ต้องเขียน boilerplate มากกว่า |
| Focus | Developer experience, เขียนเร็ว prototype ไว | Production robustness, ควบคุม thread/connection ได้ละเอียด |
| การติดตั้ง | Clone + CMake build/install header | ติดตั้งผ่าน package manager ได้ตรงๆ (`apt install libpistache-dev`) |

พูดง่ายๆ คือ **Crow เน้นความเร็วในการพัฒนา (Developer Velocity)** ส่วน **Pistache เน้นการควบคุม
และความทนทานระดับ production (Production Robustness)** — ทั้งสองตัวเร็วกว่า framework ภาษา
อื่นอยู่แล้วเพราะเป็น C++ ล้วน แต่ปรัชญาการออกแบบต่างกันชัดเจน

---

## 104.2 การติดตั้ง Pistache ผ่าน apt (Step 826)

ต่างจาก Crow ที่ต้อง clone จาก GitHub มา build เอง Pistache มี package สำเร็จรูปใน Ubuntu
repository ทำให้ติดตั้งง่ายกว่ามาก:

```bash
sudo apt update
sudo apt install -y libpistache-dev
```

คำสั่งนี้จะติดตั้งทั้ง:

- **`libpistache0t64`**: shared library ตัวจริง (`.so`) ที่ต้อง link ตอน runtime
- **`libpistache-dev`**: header files (`pistache/endpoint.h`, `pistache/router.h` ฯลฯ) และ
  `.so` symlink สำหรับ compile-time linking

Pistache ยังพึ่งพา **rapidjson** สำหรับงานที่เกี่ยวกับ JSON (ตัว Pistache เองไม่มี JSON parser
ในตัว) ซึ่งติดตั้งแยกต่างหาก:

```bash
sudo apt install -y rapidjson-dev
```

ตรวจสอบว่าติดตั้งสำเร็จ:

```bash
dpkg -l | grep pistache
# ii  libpistache-dev:amd64   0.0.5+ds-5.1build2   amd64   elegant C++ REST framework - development files
# ii  libpistache0t64:amd64   0.0.5+ds-5.1build2   amd64   elegant C++ REST framework
```

### คอมไพล์โปรแกรมที่ใช้ Pistache

ต่างจาก Crow ตรงนี้ชัดเจน: **ต้อง link library `-lpistache` เสมอ** (ไม่ใช่ header-only):

```bash
g++ -std=c++17 yourfile.cpp -o app -lpistache -lpthread
```

> **หมายเหตุสำคัญ**: ทุกตัวอย่างในบทนี้ถูกคอมไพล์และรันจริงด้วยคำสั่งด้านบน บนเครื่องที่มี
> `libpistache-dev` เวอร์ชัน `0.0.5+ds-5.1build2` และทดสอบด้วย `curl` จริงทุกกรณี

---

## 104.3 Http::Endpoint และ Hello World (Step 827)

Pistache มี syntax แตกต่างจาก Crow ชัดเจน แทนที่จะใช้ macro แบบ `CROW_ROUTE` เราต้อง:

1. สร้าง `Address` (IP + Port ที่จะ listen)
2. สร้าง `Http::Endpoint` แล้ว `.init(options)` ด้วยจำนวน thread ที่ต้องการ
3. เขียน handler function ที่รับ `const Rest::Request&` และ `Http::ResponseWriter`
4. `.setHandler(...)` แล้ว `.serve()` (หรือ `.serveThreaded()` สำหรับแบบ non-blocking)

```cpp
#include <pistache/endpoint.h>
#include <pistache/router.h>

using namespace Pistache;

void helloHandler(const Rest::Request&, Http::ResponseWriter response) {
    response.send(Http::Code::Ok, "Hello, Pistache!\n");
}

int main() {
    Address addr(Ipv4::any(), Port(19080));
    auto opts = Http::Endpoint::options().threads(2);

    Rest::Router router;
    Rest::Routes::Get(router, "/", Rest::Routes::bind(&helloHandler));

    Http::Endpoint server(addr);
    server.init(opts);
    server.setHandler(router.handler());
    server.serve();
}
```

คอมไพล์และรัน:

```bash
g++ -std=c++17 hello.cpp -o hello_pistache -lpistache -lpthread
./hello_pistache
```

ทดสอบ (เปิด terminal อีกอัน):

```bash
curl -s -i http://127.0.0.1:19080/
```

ผลลัพธ์จริงที่ได้:

```
HTTP/1.1 200 OK
Connection: Close
Content-Length: 17

Hello, Pistache!
```

### อธิบายทีละส่วน

- **`Address addr(Ipv4::any(), Port(19080))`**: กำหนดว่าจะ listen บนทุก network interface
  (`Ipv4::any()` เทียบเท่า `0.0.0.0`) พอร์ต 19080
- **`Http::Endpoint::options().threads(2)`**: builder pattern สำหรับกำหนด config ของ endpoint
  — ในที่นี้กำหนดให้ใช้ 2 worker thread (เทียบเท่ากับ `.concurrency(2)` ของ Crow)
- **`response.send(Http::Code::Ok, "...")`**: ส่ง response กลับพร้อม HTTP status code — สังเกต
  ว่า Pistache ใช้ enum `Http::Code::Ok` แทนตัวเลข `200` ตรงๆ ทำให้โค้ดอ่านง่ายและ type-safe
  กว่าการใช้ magic number
- **`server.serve()`**: เป็น **blocking call เหมือน `.run()` ของ Crow** — จะรันค้างจนกว่าจะถูก
  สั่งหยุด (Pistache ยังมี `serveThreaded()` สำหรับกรณีต้องการรัน server ใน background thread
  แล้วให้ main thread ทำงานอื่นต่อ)

### ข้อสังเกตสำคัญจาก response จริงที่ได้

สังเกตว่า header ที่ได้จาก Pistache คือ **`Connection: Close`** ในขณะที่ Crow ตอบ
**`Connection: Keep-Alive`** เป็น default — นี่คือความแตกต่างเชิงพฤติกรรม default ระหว่างสอง
framework ที่ควรรู้ไว้ ถ้าต้องการ Keep-Alive ใน Pistache เพื่อประสิทธิภาพที่ดีขึ้นเมื่อ client
ยิงหลาย request ติดกัน ต้องตั้งค่าเพิ่มเติมผ่าน `Http::Endpoint::options()` (เช่น
`.flags(Tcp::Options::ReuseAddr)` และการจัดการ header เอง) ซึ่งซับซ้อนกว่าฝั่ง Crow ที่เปิด
Keep-Alive ให้อัตโนมัติ

---

## 104.4 Rest::Router เบื้องต้น (Step 828)

เมื่อ endpoint มีมากกว่าหนึ่งเส้นทาง เราใช้ `Rest::Router` เพื่อจัดการ mapping ระหว่าง
(HTTP Method + Path) กับ handler function:

```cpp
Rest::Router router;

Rest::Routes::Get(router, "/tasks", Rest::Routes::bind(&getTasks));
Rest::Routes::Post(router, "/tasks", Rest::Routes::bind(&postTask));
Rest::Routes::Get(router, "/tasks/:id", Rest::Routes::bind(&getTaskById));
```

สังเกต syntax ของ Path Parameter: Pistache ใช้ **`:id`** (colon-prefix แบบเดียวกับ Express.js/
Sinatra) แทนที่จะเป็น `<int>` แบบ Crow ข้อแตกต่างสำคัญคือ **Pistache ไม่ตรวจสอบชนิดข้อมูลของ
parameter ให้อัตโนมัติ** — `:id` จับได้ทั้ง `/tasks/42` และ `/tasks/abc` เหมือนกัน (ต่างจาก
Crow ที่ `<int>` จะปฏิเสธ `/tasks/abc` ด้วย 404 ทันที) หน้าที่แปลง string เป็นตัวเลขและตรวจสอบ
ความถูกต้องจึงตกเป็นของผู้เขียนโค้ดเองทั้งหมดใน Pistache:

```cpp
void getTaskById(const Rest::Request& req, Http::ResponseWriter response) {
    auto idStr = req.param(":id").as<std::string>();
    int id;
    try {
        id = std::stoi(idStr);
    } catch (const std::exception&) {
        response.send(Http::Code::Bad_Request, "invalid id\n");
        return;
    }
    // ... ใช้ id ต่อ ...
}
```

`req.param(":id")` คืนค่าเป็น `TypedParam` ที่ต้องเรียก `.as<T>()` เพื่อแปลงเป็นชนิดที่ต้องการ —
ถ้า string ไม่ใช่ตัวเลขจริงๆ `.as<std::string>()` จะยังทำงานได้ (ได้ string ดิบมา) แต่ถ้าเราเรียก
`.as<int>()` ตรงๆ กับ string ที่แปลงไม่ได้ จะ throw exception จึงควรครอบด้วย `try/catch` เสมอ
เมื่อรับ input จาก path parameter ที่อาจไม่ตรงรูปแบบ

---

## 104.5 เขียน REST Endpoint ด้วย Pistache Syntax (Step 829)

มาดูตัวอย่างที่ผสม GET, POST และ path parameter เข้าด้วยกัน ก่อนจะไปสู่ตัวอย่างเต็มรูปแบบใน
หัวข้อ 104.7 มาดูโครงสร้างพื้นฐานของ handler ที่อ่าน JSON body ด้วย **rapidjson** กันก่อน
(เพราะ Pistache ไม่มี JSON parser ในตัวแบบ Crow):

```cpp
#include <pistache/endpoint.h>
#include <pistache/router.h>
#include <rapidjson/document.h>
#include <rapidjson/stringbuffer.h>
#include <rapidjson/writer.h>

using namespace Pistache;

void postExample(const Rest::Request& req, Http::ResponseWriter response) {
    rapidjson::Document body;
    body.Parse(req.body().c_str());

    if (body.HasParseError() || !body.IsObject() || !body.HasMember("title")) {
        response.send(Http::Code::Bad_Request, "invalid or missing title\n");
        return;
    }

    std::string title = body["title"].GetString();

    rapidjson::Document doc;
    doc.SetObject();
    auto& alloc = doc.GetAllocator();
    doc.AddMember("received_title", rapidjson::Value(title.c_str(), alloc), alloc);

    rapidjson::StringBuffer buffer;
    rapidjson::Writer<rapidjson::StringBuffer> writer(buffer);
    doc.Accept(writer);

    response.headers().add<Http::Header::ContentType>(MIME(Application, Json));
    response.send(Http::Code::Ok, buffer.GetString());
}
```

จุดที่ต้องสังเกตและเป็นความแตกต่างใหญ่จาก Crow:

1. **`req.body()`** เป็นฟังก์ชัน (มีวงเล็บ) ไม่ใช่ member field แบบ `req.body` ของ Crow
2. **ต้องเขียน rapidjson boilerplate เองทั้งหมด**: สร้าง `Document`, เรียก `.Parse()`,
   เช็ค `.HasParseError()`, สร้าง `allocator`, ใส่ค่าด้วย `.AddMember()`, serialize กลับด้วย
   `Writer` — ยาวกว่า `crow::json::wvalue`/`rvalue` มากสำหรับงานเดียวกัน นี่คือราคาที่ต้องจ่าย
   แลกกับความยืดหยุ่นและประสิทธิภาพที่สูงกว่าของ rapidjson (rapidjson เป็นหนึ่งใน JSON library
   ที่เร็วที่สุดในโลกของ C++ เพราะออกแบบมาให้ทำ zero-copy parsing ได้)
3. **ต้องเซ็ต `Content-Type` เอง** ผ่าน `response.headers().add<Http::Header::ContentType>(...)`
   — Pistache ไม่มีกลไกเดาและเซ็ต Content-Type อัตโนมัติให้แบบ `crow::json::wvalue`
   ไม่ว่าจะ return JSON หรือไม่ก็ตาม ผู้เขียนโค้ดต้องเซ็ตเองเสมอ

---

## 104.6 Thread Pool ของ Pistache (Step 830)

เหมือนกับ `.concurrency(n)` ของ Crow, Pistache กำหนดจำนวน worker thread ผ่าน
`Http::Endpoint::options().threads(n)`:

```cpp
auto opts = Http::Endpoint::options().threads(4);
server.init(opts);
```

หลักการเรื่อง thread-safety ของ shared state **เหมือนกันทุกประการกับที่เรียนใน Part 103**:
ถ้า handler เข้าถึงตัวแปร global (เช่น `std::vector` ที่เก็บข้อมูล) จากหลาย thread พร้อมกันโดย
ไม่มี `std::mutex` ป้องกัน จะเกิด Race Condition เหมือนกัน ไม่ว่าจะใช้ Crow หรือ Pistache ก็ตาม
— นี่คือหลักการพื้นฐานของ Concurrent Programming ที่ไม่เปลี่ยนไปตาม framework ที่เลือกใช้

---

## 104.7 Port Task API จาก Part 103 มาเขียนด้วย Pistache (Step 831)

ตอนนี้เรามาลอง port **Task API** (จัดการ todo item) จาก Part 103 ที่เขียนด้วย Crow มาเขียนใหม่
ทั้งหมดด้วย Pistache + rapidjson เพื่อเห็นความแตกต่างแบบเทียบเคียงกันตรงๆ

```cpp
#include <pistache/endpoint.h>
#include <pistache/router.h>
#include <rapidjson/document.h>
#include <rapidjson/stringbuffer.h>
#include <rapidjson/writer.h>
#include <vector>
#include <mutex>

using namespace Pistache;

struct Task {
    int id;
    std::string title;
    bool done;
};

std::vector<Task> g_tasks;
std::mutex g_mutex;
int g_next_id = 1;

void getTasks(const Rest::Request&, Http::ResponseWriter response) {
    rapidjson::Document doc;
    doc.SetObject();
    auto& alloc = doc.GetAllocator();
    rapidjson::Value arr(rapidjson::kArrayType);
    {
        std::lock_guard<std::mutex> lock(g_mutex);
        for (auto& t : g_tasks) {
            rapidjson::Value item(rapidjson::kObjectType);
            item.AddMember("id", t.id, alloc);
            item.AddMember("title", rapidjson::Value(t.title.c_str(), alloc), alloc);
            item.AddMember("done", t.done, alloc);
            arr.PushBack(item, alloc);
        }
    }
    doc.AddMember("tasks", arr, alloc);

    rapidjson::StringBuffer buffer;
    rapidjson::Writer<rapidjson::StringBuffer> writer(buffer);
    doc.Accept(writer);

    response.headers().add<Http::Header::ContentType>(MIME(Application, Json));
    response.send(Http::Code::Ok, buffer.GetString());
}

void postTask(const Rest::Request& req, Http::ResponseWriter response) {
    rapidjson::Document body;
    body.Parse(req.body().c_str());

    if (body.HasParseError() || !body.IsObject() || !body.HasMember("title")) {
        response.send(Http::Code::Bad_Request, "invalid or missing title\n");
        return;
    }

    Task t;
    {
        std::lock_guard<std::mutex> lock(g_mutex);
        t.id = g_next_id++;
        t.title = body["title"].GetString();
        t.done = false;
        g_tasks.push_back(t);
    }

    rapidjson::Document doc;
    doc.SetObject();
    auto& alloc = doc.GetAllocator();
    doc.AddMember("id", t.id, alloc);
    doc.AddMember("title", rapidjson::Value(t.title.c_str(), alloc), alloc);
    doc.AddMember("done", t.done, alloc);

    rapidjson::StringBuffer buffer;
    rapidjson::Writer<rapidjson::StringBuffer> writer(buffer);
    doc.Accept(writer);

    response.headers().add<Http::Header::ContentType>(MIME(Application, Json));
    response.send(Http::Code::Created, buffer.GetString());
}

void getTaskById(const Rest::Request& req, Http::ResponseWriter response) {
    auto idStr = req.param(":id").as<std::string>();
    int id = std::stoi(idStr);
    std::lock_guard<std::mutex> lock(g_mutex);
    for (auto& t : g_tasks) {
        if (t.id == id) {
            rapidjson::Document doc;
            doc.SetObject();
            auto& alloc = doc.GetAllocator();
            doc.AddMember("id", t.id, alloc);
            doc.AddMember("title", rapidjson::Value(t.title.c_str(), alloc), alloc);
            doc.AddMember("done", t.done, alloc);
            rapidjson::StringBuffer buffer;
            rapidjson::Writer<rapidjson::StringBuffer> writer(buffer);
            doc.Accept(writer);
            response.headers().add<Http::Header::ContentType>(MIME(Application, Json));
            response.send(Http::Code::Ok, buffer.GetString());
            return;
        }
    }
    response.send(Http::Code::Not_Found, "task not found\n");
}

int main() {
    Address addr(Ipv4::any(), Port(19081));
    auto opts = Http::Endpoint::options().threads(4);

    Rest::Router router;
    Rest::Routes::Get(router, "/tasks", Rest::Routes::bind(&getTasks));
    Rest::Routes::Post(router, "/tasks", Rest::Routes::bind(&postTask));
    Rest::Routes::Get(router, "/tasks/:id", Rest::Routes::bind(&getTaskById));

    Http::Endpoint server(addr);
    server.init(opts);
    server.setHandler(router.handler());
    server.serve();
}
```

### ทดสอบจริงทีละ endpoint

```bash
g++ -std=c++17 tasks_api.cpp -o tasks_api -lpistache -lpthread
./tasks_api
```

**1) GET /tasks ตอนยังไม่มีข้อมูล**

```bash
curl -s -i http://127.0.0.1:19081/tasks
```

```
HTTP/1.1 200 OK
Content-Type: application/json
Connection: Close
Content-Length: 12

{"tasks":[]}
```

**2) POST /tasks สร้าง task ใหม่**

```bash
curl -s -i -X POST -H "Content-Type: application/json" \
     -d '{"title":"เขียนโค้ด Pistache"}' http://127.0.0.1:19081/tasks
```

```
HTTP/1.1 201 Created
Content-Type: application/json
Connection: Close
Content-Length: 68

{"id":1,"title":"เขียนโค้ด Pistache","done":false}
```

**3) GET /tasks หลัง insert สำเร็จ**

```bash
curl -s http://127.0.0.1:19081/tasks
# {"tasks":[{"id":1,"title":"เขียนโค้ด Pistache","done":false}]}
```

**4) GET /tasks/1 (ดึงตาม id)**

```bash
curl -s -i http://127.0.0.1:19081/tasks/1
```

```
HTTP/1.1 200 OK
Content-Type: application/json
Connection: Close
Content-Length: 68

{"id":1,"title":"เขียนโค้ด Pistache","done":false}
```

**5) GET /tasks/999 (ไม่พบ)**

```bash
curl -s -i http://127.0.0.1:19081/tasks/999
```

```
HTTP/1.1 404 Not Found
Connection: Close
Content-Length: 15

task not found
```

**6) POST /tasks ด้วย body ที่ไม่ใช่ JSON**

```bash
curl -s -i -X POST -d 'garbage' http://127.0.0.1:19081/tasks
```

```
HTTP/1.1 400 Bad Request
Connection: Close
Content-Length: 25

invalid or missing title
```

ทุกกรณีข้างต้นคือผลลัพธ์จริงจากการรันบนเครื่องนี้ ยืนยันว่า API ทำงานเทียบเท่ากับเวอร์ชัน Crow
ของ Part 103 ทุกประการ — ต่างกันแค่ปริมาณโค้ดที่ต้องเขียน (rapidjson boilerplate ยาวกว่า
`crow::json::wvalue` ชัดเจนตอน serialize/deserialize) และ header `Connection: Close` ที่เป็น
default ของ Pistache (ต่างจาก Crow ที่ default เป็น `Keep-Alive`)

---

## 104.8 เปรียบเทียบ Crow vs Pistache แบบตรงไปตรงมา (Step 832)

### ตารางเปรียบเทียบโดยละเอียด

| ประเด็น | Crow | Pistache |
|---|---|---|
| ความง่ายในการเริ่มต้น | ง่ายมาก (clone + cmake install ครั้งเดียว) | ง่าย (ติดตั้งผ่าน `apt` ตรงๆ) |
| จำนวนโค้ดสำหรับ CRUD API | น้อยกว่า (JSON built-in) | มากกว่า (ต้องเขียน rapidjson boilerplate เอง) |
| Path Parameter type-checking | Compile-time (`<int>` ปฏิเสธ non-integer อัตโนมัติ) | Runtime, ไม่ตรวจสอบชนิดให้ ต้อง validate เอง |
| JSON Support | มีในตัว (`crow::json::wvalue`/`rvalue`) | ไม่มี ต้องพึ่ง library ภายนอก (rapidjson แนะนำ) |
| Default Connection Header | `Keep-Alive` | `Close` |
| ความยืดหยุ่นระดับ low-level (เช่นควบคุม TCP option) | จำกัดกว่า | ควบคุมได้ละเอียดกว่า (เข้าถึง epoll-based reactor โดยตรง) |
| Middleware system | มีในตัว (`before_handle`/`after_handle`) | ไม่มีระบบ Middleware สำเร็จรูป ต้องเขียน wrapper เอง |
| เหมาะกับ | Prototype เร็ว, API ขนาดเล็ก-กลาง, ทีมที่อยากได้ syntax คุ้นเคยจาก Flask/Express | ระบบที่ต้องการควบคุมประสิทธิภาพ/connection อย่างละเอียด, ทีมที่มีประสบการณ์ C++ สูงและต้องการความยืดหยุ่นสูงสุด |
| Community/Maturity | ใช้งานแพร่หลาย มี middleware สำเร็จรูปให้เลือกเยอะ (CORS, Session) | ใช้ในงาน production จริงหลายที่ (เช่นระบบ telecom) เน้นเสถียรภาพระยะยาว |

### เมื่อไหร่ควรเลือก Crow

- ต้องการสร้าง REST API หรือ prototype ให้เสร็จเร็วที่สุด
- ทีมคุ้นเคยกับ syntax แบบ Flask/Express มาก่อน อยากเรียนรู้ได้เร็ว
- งานส่วนใหญ่เป็นการรับ-ส่ง JSON ธรรมดา ไม่ต้องการควบคุม TCP/HTTP ระดับลึก
- ต้องการ Middleware system ที่ใช้งานง่ายในตัว (logging, CORS ฯลฯ)

### เมื่อไหร่ควรเลือก Pistache

- ระบบต้องรองรับ throughput สูงมากและต้องการ tuning ระดับ thread/connection อย่างละเอียด
- ทีมมีความเชี่ยวชาญ C++ สูงอยู่แล้ว ไม่กังวลกับการเขียน boilerplate เพิ่ม เพื่อแลกกับการควบคุม
  ที่มากกว่า
- ต้องการเลือก JSON library เอง (เช่นใช้ nlohmann/json แทน rapidjson เพราะ syntax คุ้นเคยกว่า)
  แทนที่จะถูกผูกกับ JSON implementation ของ framework
- งานเป็นระบบ production ระยะยาวที่ต้องการความสามารถในการ debug/profile ที่ระดับ low-level
  (เพราะ Pistache ไม่ซ่อนรายละเอียดของ I/O มากเท่า Crow)

### สรุปสั้นๆ ด้วยคำเดียว

**Crow = Productivity ก่อน, Pistache = Control ก่อน** — ทั้งสองตัวเร็วกว่า framework จากภาษา
อื่นในระดับเดียวกันอยู่แล้ว (เพราะเป็น C++ ทั้งคู่) ความแตกต่างที่แท้จริงอยู่ที่ **developer
experience** และ **ระดับความยืดหยุ่นที่ framework ยอมให้เราควบคุม** ไม่ใช่ที่ตัวภาษาเบื้องหลัง

---

## ข้อผิดพลาดที่พบบ่อย (Common Pitfalls)

1. **ลืม `-lpistache` ตอน link** — เพราะ Pistache ไม่ใช่ header-only เหมือน Crow ถ้าลืม flag
   นี้จะได้ linker error จำนวนมาก (`undefined reference to Pistache::...`)
2. **สับสน `req.body` (Crow, member field) กับ `req.body()` (Pistache, member function)** —
   ถ้าเขียนโค้ด Pistache แต่ลืมใส่วงเล็บเพราะเคยชินจาก Crow จะได้ compile error ทันที
3. **ลืมว่า `:id` ของ Pistache ไม่ตรวจสอบชนิดข้อมูลให้อัตโนมัติ** — ต่างจาก `<int>` ของ Crow
   ที่ปฏิเสธ non-integer path ด้วย 404 ทันที ถ้าไม่ validate เอง (`std::stoi` แล้ว catch
   exception) โปรแกรมอาจ crash เมื่อ client ส่ง path parameter ที่แปลงเป็นตัวเลขไม่ได้
4. **ลืมเซ็ต `Content-Type` เองสำหรับ JSON response** — Pistache ไม่มีกลไกเดา Content-Type
   ให้แบบ `crow::json::wvalue` ของ Crow ถ้าลืมเซ็ต client ฝั่งที่คาดหวัง JSON (เช่น JavaScript
   `fetch().then(r => r.json())`) อาจ parse ผิดพลาดเพราะไม่รู้ว่า response เป็น JSON
5. **คาดหวัง Keep-Alive โดย default เหมือน Crow** — Pistache ตอบ `Connection: Close` เป็น
   default ถ้าระบบต้องการ Keep-Alive เพื่อลด overhead ของการเปิด TCP connection ใหม่ทุกครั้ง
   ต้องตั้งค่าเพิ่มเติมเอง ไม่ใช่พฤติกรรมที่ได้มาฟรีๆ เหมือน Crow
6. **ลืม thread-safety ของ shared state เหมือนเดิม** — หลักการเรื่อง `std::mutex` ป้องกัน
   Race Condition จาก Part 103 ยังคงใช้ได้ทุกประการกับ Pistache เพราะ handler ก็ถูกเรียกจาก
   หลาย worker thread พร้อมกันเช่นเดียวกัน

---

## แบบฝึกหัดท้ายบท

1. เขียนโปรแกรม Pistache ที่มี route `/square/:n` รับตัวเลขแล้วคืนค่ากำลังสองของมันกลับไปเป็น
   ข้อความธรรมดา (ต้อง validate ว่า `:n` แปลงเป็นตัวเลขได้จริงก่อน ถ้าแปลงไม่ได้ให้ตอบ
   `400 Bad Request`)
2. เพิ่ม endpoint `DELETE /tasks/:id` เข้าไปในตัวอย่าง Task API ของหัวข้อ 104.7 ที่ลบ task
   ตาม id ที่ระบุ (ถ้าไม่พบ id ให้ตอบ `404 Not Found`) อย่าลืม lock mutex ก่อนแก้ไข `g_tasks`
3. เพิ่ม endpoint `PUT /tasks/:id` ที่รับ JSON `{"done": true}` แล้วอัปเดตสถานะของ task ตาม id
   โดยใช้ rapidjson parse body เหมือนเดิม
4. เปลี่ยนจำนวน thread ของ server จาก 4 เป็น 1 แล้วสังเกตว่าโปรแกรมยังทำงานถูกต้องหรือไม่
   (ควรยังถูกต้องเสมอเพราะ mutex ป้องกันไว้แล้ว — จำนวน thread ไม่ควรกระทบความถูกต้องของข้อมูล
   มีผลแค่ throughput)
5. เขียนโปรแกรมเปรียบเทียบขนาดไฟล์ executable ที่ได้จาก Crow (header-only, static link
   ผ่าน asio ที่ถูก include เข้ามาทั้งหมด) กับ Pistache (dynamic link กับ `.so`) ด้วยคำสั่ง
   `ls -lh` และ `ldd` แล้วอธิบายความแตกต่างที่สังเกตเห็น
6. ลองใช้ `Http::Endpoint::options().flags(Tcp::Options::ReuseAddr)` เพิ่มเข้าไปใน config
   ของ server แล้วอธิบายว่า flag นี้มีประโยชน์อย่างไรตอน restart server บ่อยๆ ระหว่างพัฒนา
   (ใบ้: เกี่ยวข้องกับสถานะ `TIME_WAIT` ของ TCP socket ที่เรียนใน Part 33-34)

### แนวทางเฉลยข้อ 1

```cpp
#include <pistache/endpoint.h>
#include <pistache/router.h>

using namespace Pistache;

void squareHandler(const Rest::Request& req, Http::ResponseWriter response) {
    auto nStr = req.param(":n").as<std::string>();
    long long n;
    try {
        n = std::stoll(nStr);
    } catch (const std::exception&) {
        response.send(Http::Code::Bad_Request, "invalid number\n");
        return;
    }
    long long result = n * n;
    response.send(Http::Code::Ok, std::to_string(result) + "\n");
}

int main() {
    Address addr(Ipv4::any(), Port(19090));
    auto opts = Http::Endpoint::options().threads(2);

    Rest::Router router;
    Rest::Routes::Get(router, "/square/:n", Rest::Routes::bind(&squareHandler));

    Http::Endpoint server(addr);
    server.init(opts);
    server.setHandler(router.handler());
    server.serve();
}
```

ทดสอบ:

```bash
g++ -std=c++17 ex1.cpp -o ex1 -lpistache -lpthread
./ex1 &
curl -s http://127.0.0.1:19090/square/5
# 25

curl -s -i http://127.0.0.1:19090/square/abc
# HTTP/1.1 400 Bad Request ...
# invalid number
```

ใช้ `std::stoll` (แปลงเป็น `long long`) แทน `std::stoi` เพื่อรองรับตัวเลขที่มีขนาดใหญ่กว่า
`int` ปกติได้ และครอบด้วย `try/catch` เสมอเพราะ path parameter ของ Pistache ไม่ผ่านการ
ตรวจสอบชนิดใดๆ มาก่อนถึงมือเรา

### แนวทางเฉลยข้อ 2

```cpp
#include <pistache/endpoint.h>
#include <pistache/router.h>
#include <vector>
#include <mutex>
#include <algorithm>

using namespace Pistache;

struct Task {
    int id;
    std::string title;
    bool done;
};

std::vector<Task> g_tasks = {{1, "งานตัวอย่าง", false}};
std::mutex g_mutex;

void deleteTask(const Rest::Request& req, Http::ResponseWriter response) {
    auto idStr = req.param(":id").as<std::string>();
    int id;
    try {
        id = std::stoi(idStr);
    } catch (const std::exception&) {
        response.send(Http::Code::Bad_Request, "invalid id\n");
        return;
    }

    std::lock_guard<std::mutex> lock(g_mutex);
    auto it = std::find_if(g_tasks.begin(), g_tasks.end(),
                            [id](const Task& t){ return t.id == id; });
    if (it == g_tasks.end()) {
        response.send(Http::Code::Not_Found, "task not found\n");
        return;
    }
    g_tasks.erase(it);
    response.send(Http::Code::No_Content, "");
}

int main() {
    Address addr(Ipv4::any(), Port(19091));
    auto opts = Http::Endpoint::options().threads(4);

    Rest::Router router;
    Rest::Routes::Delete(router, "/tasks/:id", Rest::Routes::bind(&deleteTask));

    Http::Endpoint server(addr);
    server.init(opts);
    server.setHandler(router.handler());
    server.serve();
}
```

ทดสอบ:

```bash
g++ -std=c++17 ex2.cpp -o ex2 -lpistache -lpthread
./ex2 &
curl -s -i -X DELETE http://127.0.0.1:19091/tasks/1
# HTTP/1.1 204 No Content ...

curl -s -i -X DELETE http://127.0.0.1:19091/tasks/1
# HTTP/1.1 404 Not Found ...
# task not found (เพราะถูกลบไปแล้วในการเรียกครั้งก่อน)
```

จุดสำคัญของเฉลยนี้: ใช้ `Rest::Routes::Delete(...)` (ตัวช่วยของ Pistache สำหรับ HTTP DELETE
method โดยเฉพาะ คู่กับ `Get`/`Post`/`Put` ที่มีให้ครบ) และปฏิบัติตามหลักการเดียวกับ Task API
ของ Part 103 ทุกประการ: validate input ก่อน (แปลง id เป็นตัวเลขได้จริงไหม), lock mutex เฉพาะ
ช่วงที่แก้ไข shared state, และตอบ status code ที่สื่อความหมายถูกต้องตาม REST convention
(`204 No Content` สำหรับ delete สำเร็จ, `404 Not Found` ถ้าไม่พบ resource)

---

## สรุปท้ายบท

ใน Part นี้เราได้:

- เข้าใจว่า Pistache คือ C++ REST Framework ที่เน้น production robustness และการควบคุม
  ระดับต่ำ ต่างจาก Crow ที่เน้น developer experience และความเร็วในการพัฒนา
- ติดตั้ง Pistache ผ่าน `apt install libpistache-dev` (พร้อม `rapidjson-dev`) ได้จริง ซึ่งง่าย
  กว่าการ build Crow จาก source
- เขียนและรัน HTTP Server ด้วย `Http::Endpoint` + `Rest::Router` ได้จริง พร้อมทดสอบด้วย
  `curl` และสังเกตความแตกต่างของ default header (`Connection: Close`) เทียบกับ Crow
- เขียน REST endpoint ที่รองรับ GET/POST/DELETE พร้อม path parameter (`:id`) ด้วย syntax ของ
  Pistache และเข้าใจว่าต้อง validate ชนิดข้อมูลของ path parameter เองเสมอ
- Port Task API เต็มรูปแบบจาก Part 103 (Crow) มาเขียนใหม่ด้วย Pistache + rapidjson ได้สำเร็จ
  และทดสอบเทียบผลลัพธ์จริงว่าทำงานเทียบเท่ากันทุกกรณี
- เปรียบเทียบ Crow กับ Pistache แบบตรงไปตรงมาในทุกมิติ (ความง่าย, ประสิทธิภาพ, ความยืดหยุ่น)
  และสรุปเป็นหลักการเลือกใช้: **Crow = Productivity ก่อน, Pistache = Control ก่อน**

ตลอด Part 102-104 เราได้เห็นสอง Framework หลักของ C++ สำหรับสร้าง REST API ซึ่งทั้งคู่ยังคง
ต้องอาศัยความรู้พื้นฐานเรื่อง HTTP, Concurrency และ Thread-safety ที่เรียนมาตั้งแต่ Part 100-101
เสมอ — ไม่มี framework ไหนแทนที่ความเข้าใจพื้นฐานเหล่านี้ได้ ใน **Part 105** เราจะเริ่มเชื่อมต่อ
REST API ที่เราสร้างเข้ากับฐานข้อมูลจริงเป็นครั้งแรก โดยเริ่มจาก **SQLite** ซึ่งเป็นฐานข้อมูล
แบบ embedded ที่ไม่ต้องตั้ง server แยก เหมาะสำหรับการเรียนรู้พื้นฐานการเชื่อมต่อฐานข้อมูลด้วย
C++ ก่อนไปสู่ PostgreSQL/MySQL ใน Part 106

**ต่อไป:** [Part 105 — เชื่อมต่อฐานข้อมูล SQLite ด้วย C++](./part-105-sqlite-cpp.md)
