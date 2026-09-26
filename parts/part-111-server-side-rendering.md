# Part 111: Server-Side Rendering ด้วย C++ (Step 881–888)

> Module I — Web Development ด้วย C/C++ | Part 111 จาก 125
> Step ที่ครอบคลุมใน Part นี้: Step 881–888
> Part ก่อนหน้า: [Part 110 — Authentication และ Security](./part-110-auth-security.md) | Part ถัดไป: [Part 112 — Deploy: Docker, Nginx, Production Server](./part-112-deploy-docker-nginx.md)

## เป้าหมายการเรียนรู้ (Learning Objectives)

หลังจบ Part นี้ ผู้เรียนจะสามารถ:

1. อธิบายความแตกต่างระหว่าง **Server-Side Rendering (SSR)** กับ REST API ที่ตอบกลับเป็น JSON ได้อย่างชัดเจน ทั้งในแง่ของ flow การทำงานและผลลัพธ์ที่ browser ได้รับ
2. วิเคราะห์ได้ว่ากรณีใดควรเลือกใช้ SSR และกรณีใดควรเลือกใช้สถาปัตยกรรม SPA (Single Page Application) + REST API แยกกัน
3. ใช้ `crow::mustache` เขียน template HTML ที่มี variable (`{{variable}}`), section (`{{#section}}`), inverted section (`{{^section}}`), comment และ partial ได้จริง
4. Render หน้าเว็บ HTML ฉบับสมบูรณ์จากข้อมูลจริงในหน่วยความจำ (เช่น รายการ Task จาก Part 108) แทนที่จะส่งเป็น JSON อย่างเดียว
5. Serve static file เช่นไฟล์ CSS ผ่าน Crow framework ได้อย่างถูกต้องและปลอดภัย
6. เขียนฟอร์ม HTML ที่ทำงานร่วมกับ backend C++ ได้จริง โดยใช้ pattern **Post/Redirect/Get (PRG)** ซึ่งเป็นรูปแบบมาตรฐานของเว็บแอปแบบ SSR
7. ประเมินข้อจำกัดของการทำ Templating ด้วย C++ อย่างตรงไปตรงมา เทียบกับภาษาที่ถูกออกแบบมาเพื่องานเว็บโดยเฉพาะ เพื่อให้ตัดสินใจเลือกเครื่องมือได้อย่างมีข้อมูลรอบด้าน ไม่ใช่ตัดสินใจตามความเคยชิน

> **หมายเหตุเรื่องสภาพแวดล้อม**: โค้ดและผลลัพธ์ทุกชิ้นใน Part นี้ถูกคอมไพล์และรันจริงด้วย
> `g++ 13.3.0 -std=c++17` กับ Crow framework ที่ติดตั้งไว้ที่ `/usr/local/include/crow`
> ผลลัพธ์ที่แสดง (เช่น output ของ `curl`) คือผลลัพธ์จริงที่ได้จากการรันคำสั่งเหล่านั้น
> ไม่ใช่ผลลัพธ์ที่แต่งขึ้น

---

## 111.1 SSR คืออะไร ต่างจาก REST API ที่ return JSON อย่างไร (Step 881)

ตลอด Part 100–110 ที่ผ่านมา ทุก endpoint ที่เราสร้างด้วย Crow ล้วนตอบกลับเป็น **JSON**
เช่น `GET /tasks` จะได้ผลลัพธ์แบบนี้:

```json
[
  {"id": 1, "title": "เรียน Server-Side Rendering", "done": true},
  {"id": 2, "title": "ทดสอบ crow::mustache", "done": false}
]
```

JSON แบบนี้ browser เปล่าๆ (ไม่มี JavaScript ทำงาน) **ไม่สามารถแสดงผลเป็นหน้าเว็บที่มนุษย์อ่านได้**
เลย ต้องมี JavaScript ฝั่ง client (เช่น React, Vue, หรือแม้แต่ `fetch()` ธรรมดา) คอยดึงข้อมูล JSON
มาแปลงเป็น HTML DOM อีกทีหนึ่ง สถาปัตยกรรมแบบนี้เรียกว่า **SPA (Single Page Application) + REST
API** — server มีหน้าที่แค่ส่งข้อมูลดิบ ส่วนการ "ประกอบร่าง" เป็นหน้าเว็บเป็นงานของฝั่ง client

**Server-Side Rendering (SSR)** คือแนวทางตรงข้าม: **server เป็นคนสร้าง HTML ที่สมบูรณ์แล้วส่งกลับ
ไปให้ browser เลย** โดย browser แค่รับ HTML มา render (แสดงผล) ตรงๆ ไม่ต้องมี JavaScript ใดๆ
ก็แสดงผลได้ครบถ้วน

### เปรียบเทียบ Flow การทำงาน

**Flow แบบ REST API + SPA:**

```
Browser                          Server
   │                                │
   │  1. GET /  (ขอหน้าเปล่าๆ)      │
   │ ──────────────────────────►   │
   │  2. ได้ index.html + JS bundle │
   │ ◄──────────────────────────   │
   │  3. JS เริ่มทำงาน, เรียก        │
   │     GET /api/tasks (JSON)     │
   │ ──────────────────────────►   │
   │  4. ได้ JSON array             │
   │ ◄──────────────────────────   │
   │  5. JS วน loop สร้าง DOM       │
   │     ผู้ใช้เห็นหน้าเว็บ (ช้ากว่า) │
```

**Flow แบบ SSR:**

```
Browser                          Server
   │                                │
   │  1. GET /tasks                 │
   │ ──────────────────────────►   │
   │     Server ดึงข้อมูล + render  │
   │     HTML เสร็จสมบูรณ์ในทีเดียว  │
   │  2. ได้ HTML ที่พร้อมแสดงผลแล้ว │
   │ ◄──────────────────────────   │
   │     Browser แสดงผลได้ทันที     │
   │     (ไม่ต้องรอ JS โหลด/ทำงาน)  │
```

จุดต่างที่สำคัญที่สุดคือ **ใครเป็นคนแปลงข้อมูล (data) ให้เป็นหน้าตา (presentation)** — ถ้าเป็น
server คือ SSR ถ้าเป็น browser (ผ่าน JavaScript) คือ Client-Side Rendering (CSR) ซึ่ง SPA ส่วนใหญ่
ใช้แนวทางนี้

### ตารางเปรียบเทียบ

| ประเด็น | REST API + SPA (CSR) | Server-Side Rendering (SSR) |
|---|---|---|
| ใครสร้าง HTML | JavaScript ฝั่ง browser | โปรแกรม C++ ฝั่ง server |
| Response ของ endpoint | JSON (`application/json`) | HTML สมบูรณ์ (`text/html`) |
| ต้องมี JavaScript หรือไม่ | ต้องมี (จำเป็น) | ไม่จำเป็น (ทำงานได้แม้ปิด JS) |
| First Paint (เห็นเนื้อหาครั้งแรก) | ช้ากว่า (ต้องรอโหลด JS + fetch data) | เร็วกว่า (ได้ HTML พร้อมเนื้อหาทันที) |
| SEO (เครื่องมือค้นหาเก็บข้อมูล) | ต้องพึ่งพา JS rendering ของ crawler (ซับซ้อนกว่า) | อ่าน HTML ได้ตรงๆ ง่ายและชัวร์กว่า |
| ความซับซ้อนของ Stack | ต้องมี frontend build pipeline แยก (webpack/vite) | มีแค่ backend เดียว ไม่ต้อง build แยก |
| Interactivity (โต้ตอบแบบ real-time) | ทำได้ลื่นไหลกว่า ไม่ต้อง reload หน้า | ต้อง reload หน้าเว็บเมื่อข้อมูลเปลี่ยน (เว้นแต่เสริม JS บางส่วน) |
| ภาระของ Server ต่อ 1 request | เบา (แค่ query DB + serialize JSON) | หนักกว่า (query DB + serialize + render template) |

ข้อสำคัญที่ต้องเข้าใจ: **SSR ไม่ได้แปลว่าห้ามใช้ JavaScript เลย** เว็บไซต์จำนวนมากในโลกจริงใช้
แนวทางผสม (Hybrid) คือ render หน้าแรกด้วย SSR เพื่อความเร็วและ SEO แล้วค่อยเสริมความสามารถ
โต้ตอบด้วย JavaScript เล็กน้อยทีหลัง (บางครั้งเรียกว่า Progressive Enhancement) แต่ใน Part นี้
เราจะเน้นที่ SSR แบบ "ล้วนๆ" คือไม่ใช้ JavaScript ฝั่ง client เลยแม้แต่บรรทัดเดียว เพื่อให้เห็นภาพ
ชัดเจนที่สุดว่า C++ ฝั่ง server ทำอะไรได้บ้าง

---

## 111.2 เมื่อไหร่ SSR เหมาะกว่า SPA+API (Step 882)

การเลือกระหว่าง SSR กับ SPA+API ไม่ใช่เรื่องของ "อันไหนดีกว่า" แบบสัมบูรณ์ แต่เป็นเรื่องของ
**เลือกให้เหมาะกับลักษณะงาน** ต่อไปนี้คือปัจจัยหลักที่ควรพิจารณา

### 1. SEO (Search Engine Optimization)

เครื่องมือค้นหาอย่าง Google ในปัจจุบันสามารถรัน JavaScript เพื่อ index หน้าเว็บแบบ SPA ได้ในระดับ
หนึ่งก็จริง แต่:

- การ render ด้วย JS ต้องใช้เวลาและทรัพยากรของฝั่ง crawler มากกว่า ทำให้บาง crawler (โดยเฉพาะ
  ของเว็บไซต์อื่นๆ ที่ไม่ใช่ Google เช่น บอทของ social media ที่ทำ link preview) มักจะไม่รัน
  JavaScript เลย และเห็นแค่หน้าเปล่าๆ
- SSR ส่ง HTML ที่มีเนื้อหาครบถ้วนตั้งแต่ request แรก การันตีว่า **ทุก** crawler จะเห็นเนื้อหาจริง
  โดยไม่ต้องพึ่งพาว่า crawler จะรัน JS ได้ดีแค่ไหน

**สรุป**: ถ้าเว็บไซต์พึ่งพา traffic จาก Google/การค้นหา (เช่น เว็บข่าว, บล็อก, หน้าสินค้า
E-Commerce) → SSR ปลอดภัยและมั่นใจได้มากกว่า

### 2. ลักษณะของเนื้อหา (Content-Heavy vs. Interaction-Heavy)

| ลักษณะเว็บไซต์ | แนะนำ |
|---|---|
| เว็บข่าว, บล็อก, เอกสารประกอบ (documentation site) | SSR — เนื้อหาเป็นหลัก ไม่ต้องโต้ตอบซับซ้อน |
| หน้า Landing Page, เว็บบริษัท, พอร์ตโฟลิโอ | SSR — ต้องการความเร็วและ SEO เป็นหลัก |
| Dashboard ภายในองค์กร (internal tool) | SSR ก็เพียงพอ — ผู้ใช้กลุ่มเล็ก ไม่ต้อง SEO |
| แอปแชทแบบ real-time, Google Docs, เกม | SPA+API หรือ WebSocket (Part 109) — ต้องโต้ตอบทันที ไม่รีโหลดหน้า |
| แอปที่ทำงานแบบ Offline-first | SPA (Progressive Web App) — ต้องเก็บ state ฝั่ง client |

### 3. ความซับซ้อนของทีมและ Stack

ถ้าทีมมีแค่โปรแกรมเมอร์ C++ และไม่อยากดูแล frontend build pipeline แยกต่างหาก (npm, webpack,
React) การทำ SSR ด้วย C++ ล้วนๆ ทำให้ deploy ง่ายกว่ามาก — มี binary เดียว ไม่ต้องดูแล 2 โปรเจกต์
คนละภาษา คนละ deployment pipeline

### 4. งบประมาณด้าน Performance ของ Server

SSR ต้องทำงานมากกว่าต่อ 1 request (query ข้อมูล + ประกอบ template + render) เทียบกับ REST API
ที่แค่ serialize JSON ธรรมดา ถ้าเว็บไซต์มี traffic สูงมากและเนื้อหาเปลี่ยนบ่อย อาจต้องพิจารณาการทำ
caching (เช่น cache หน้า HTML ที่ render แล้วไว้ระยะหนึ่ง) ร่วมด้วย — เราจะพูดถึงเรื่อง caching
และ scaling ในระดับ production เพิ่มเติมใน Part 112 และ Module J

### ตารางสรุปการตัดสินใจ

| คำถามที่ควรถาม | ตอบ "ใช่" → เอียงไปทาง |
|---|---|
| ต้องพึ่งพา SEO/Google Search มากหรือไม่? | SSR |
| เนื้อหาเป็นหลัก การโต้ตอบน้อย (อ่าน มากกว่า คลิก)? | SSR |
| ทีมเล็ก อยากมี stack เดียว ไม่อยากดูแล frontend framework แยก? | SSR |
| ต้องการ Interactivity สูงมาก (drag-and-drop, real-time update)? | SPA + API (หรือ WebSocket) |
| ต้องรองรับ Mobile App ที่ใช้ API เดียวกับเว็บ? | SPA + API (เพราะ API เดียวใช้ได้ทั้งเว็บและแอป) |

ในทางปฏิบัติ บริษัทเทคโนโลยีจำนวนมากเลือกแนวทาง **Hybrid**: ใช้ SSR สำหรับหน้าแรกที่ต้องการ
ความเร็วและ SEO (เช่น หน้าแรกของเว็บอีคอมเมิร์ซ) และใช้ SPA+API สำหรับส่วนที่ต้องโต้ตอบซับซ้อน
(เช่น หน้าตะกร้าสินค้า, หน้า checkout) ความรู้เรื่อง SSR ใน Part นี้จึงไม่ใช่ "ทางเลือกที่ตัดกับ
REST API" ที่เรียนมาใน Part 102–108 แต่เป็น **เครื่องมืออีกชิ้นในกล่องเครื่องมือ** ที่หยิบมาใช้
เมื่อโจทย์เหมาะสม

---

## 111.3 เริ่มต้นกับ crow::mustache: Logic-less Template และ {{variable}} (Step 883)

Crow มาพร้อม template engine ในตัวชื่อ **Mustache** ({{ }} หน้าตาเหมือนหนวดแมว จึงเป็นที่มาของชื่อ)
Mustache เป็น template engine แบบ **"Logic-less"** — หมายความว่าในไฟล์ template จะ**ไม่มี**การเขียน
โค้ดโปรแกรม (ไม่มี if-else แบบเต็มรูปแบบ, ไม่มี for loop แบบเขียนเอง, ไม่มีการเรียกฟังก์ชัน) มีแค่
placeholder ง่ายๆ ไม่กี่แบบเท่านั้น ปรัชญานี้ตั้งใจบังคับให้ตรรกะทางธุรกิจ (business logic) ทั้งหมด
อยู่ในโค้ด C++ ส่วน template มีหน้าที่แค่ "จัดวาง" ข้อมูลเป็นหน้าตา HTML เท่านั้น — เป็นการแยก
concern ระหว่าง logic กับ presentation อย่างเข้มงวด

### โครงสร้างของ crow::mustache

- `crow::mustache::context` คือ alias ของ `crow::json::wvalue` ที่เราคุ้นเคยกันดีจาก Part 107
  (nlohmann/json) และ Part 102-103 — ใช้เก็บข้อมูลที่จะส่งเข้าไปแทนที่ใน template
- `crow::mustache::set_base(path)` กำหนด "โฟลเดอร์ฐาน" ที่ไฟล์ template ทั้งหมดจะถูกค้นหา (เรียก
  ครั้งเดียวตอนเริ่มโปรแกรมก็พอ)
- `crow::mustache::load(filename)` โหลดไฟล์ template จากโฟลเดอร์ฐาน แล้วคืนค่าเป็น `template_t`
- `template.render(context)` render template ด้วยข้อมูลที่ให้มา ได้ผลลัพธ์เป็น
  `crow::mustache::rendered_template` (เป็น subclass ของ `crow::returnable` — สามารถ `return`
  ออกจาก route handler ของ Crow ได้ตรงๆ โดยไม่ต้องแปลงชนิดข้อมูลเอง)

### ตัวอย่างแรก: {{variable}}

สร้างโครงสร้างโฟลเดอร์:

```bash
mkdir -p ssr_demo/templates ssr_demo/static
cd ssr_demo
```

สร้างไฟล์ `templates/hello.html`:

```html
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <title>{{page_title}}</title>
</head>
<body>
    <h1>สวัสดี, {{name}}!</h1>
    <p>วันนี้คือวันที่ {{date}}</p>
</body>
</html>
```

และไฟล์ `main.cpp`:

```cpp
#include "crow.h"
#include "crow/mustache.h"

int main() {
    // กำหนดโฟลเดอร์ฐานสำหรับ template — เรียกครั้งเดียวตอนเริ่มโปรแกรม
    crow::mustache::set_base("templates");

    crow::SimpleApp app;

    CROW_ROUTE(app, "/hello")
    ([]() {
        crow::mustache::context ctx;
        ctx["page_title"] = "หน้าทดสอบ SSR";
        ctx["name"] = "สมชาย";
        ctx["date"] = "26 กันยายน 2569";

        auto page = crow::mustache::load("hello.html");
        return page.render(ctx);
    });

    app.port(18080).run();
}
```

คอมไพล์และรัน:

```bash
g++ -std=c++17 -I/usr/local/include main.cpp -o app -lpthread
./app
```

ทดสอบด้วย `curl` (จากอีก terminal หนึ่ง):

```bash
curl http://127.0.0.1:18080/hello
```

ผลลัพธ์ที่ได้จริง:

```html
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <title>หน้าทดสอบ SSR</title>
</head>
<body>
    <h1>สวัสดี, สมชาย!</h1>
    <p>วันนี้คือวันที่ 26 กันยายน 2569</p>
</body>
</html>
```

สังเกตว่า `{{page_title}}`, `{{name}}`, `{{date}}` ถูกแทนที่ด้วยค่าจริงจาก `context` เรียบร้อย
และที่สำคัญคือ **`Content-Type` ของ response ถูกตั้งเป็น `text/html` โดยอัตโนมัติ** (ไม่ต้อง
เขียน `res.set_header("Content-Type", "text/html")` เอง) เพราะ `rendered_template` สืบทอดมาจาก
`crow::returnable` ที่ประกาศ content type ไว้ในตัวเองแล้วว่าเป็น `"text/html"`

### ความปลอดภัย: {{variable}} ป้องกัน HTML/XSS Injection ให้อัตโนมัติ

ข้อดีที่สำคัญมากของ Mustache ที่ Crow ใช้คือ **`{{variable}}` จะ escape อักขระพิเศษของ HTML ให้
อัตโนมัติ** ทดสอบได้จริงดังนี้:

```cpp
crow::mustache::context ctx;
ctx["title"] = "<script>alert(1)</script>";
auto page = crow::mustache::load("escape_test.html");
```

โดยที่ `escape_test.html` มีเนื้อหา:

```html
<p>Escaped: {{title}}</p>
<p>Unescaped: {{{title}}}</p>
```

ผลลัพธ์จริงที่ได้:

```html
<p>Escaped: &lt;script&gt;alert(1)&lt;&#x2F;script&gt;</p>
<p>Unescaped: <script>alert(1)</script></p>
```

จะเห็นว่า:

- `{{title}}` (สองปีกกา) → escape ตัวอักษร `<`, `>`, `&`, `/` ให้เป็น HTML entity อัตโนมัติ
  ทำให้ script ที่แฝงมากับข้อมูล**ไม่ถูกรันเป็นโค้ด** — ป้องกัน **Stored XSS
  (Cross-Site Scripting)** ได้ในตัว
- `{{{title}}}` (สามปีกกา, unescaped tag) → ใส่ค่าดิบๆ ตรงๆ โดยไม่ escape ใช้เฉพาะกรณีที่มั่นใจ
  100% ว่าข้อมูลนั้นปลอดภัยและตั้งใจให้เป็น HTML จริงๆ (เช่น เนื้อหาบทความที่ผ่านการกรองมาแล้ว)

> **กฎทองของ Part นี้**: ใช้ `{{variable}}` (สองปีกกา) เป็นค่าเริ่มต้นเสมอเมื่อแสดงข้อมูลที่มาจาก
> ผู้ใช้หรือฐานข้อมูล ใช้ `{{{variable}}}` (สามปีกกา) เฉพาะกรณีที่จำเป็นจริงๆ และรู้ตัวว่ากำลังทำ
> อะไรอยู่ เพราะการ unescape ข้อมูลที่ผู้ใช้ป้อนมาเองคือช่องโหว่ XSS ที่พบบ่อยที่สุดอันดับต้นๆ
> ของเว็บแอปพลิเคชันทั่วโลก

---

## 111.4 Section, Inverted Section, Comment และ Partial (Step 884)

นอกจาก `{{variable}}` แล้ว Mustache ยังมี syntax อีก 4 แบบที่สำคัญ ซึ่งครอบคลุมกรณีการใช้งาน
เกือบทั้งหมดที่จำเป็นสำหรับการ render หน้าเว็บ

### 1. Section `{{#name}} ... {{/name}}`

Section ทำหน้าที่ 2 แบบขึ้นอยู่กับชนิดของค่าใน context:

- **ถ้า `name` เป็น boolean และเป็น `true`** (หรือค่าที่ไม่ใช่ 0/ไม่ว่าง) → เนื้อหาระหว่าง
  `{{#name}}` กับ `{{/name}}` จะถูก render (แสดงผล)
- **ถ้า `name` เป็น array ของ object** → เนื้อหาระหว่างแท็กจะถูก render **ซ้ำหนึ่งครั้งต่อหนึ่ง
  element** โดยแต่ละรอบ context ภายในบล็อกจะสลับไปเป็น object ของ element นั้นๆ (เทียบเท่ากับ
  for-loop ใน template)

### 2. Inverted Section `{{^name}} ... {{/name}}`

ตรงข้ามกับ section ปกติ — เนื้อหาจะถูก render **ก็ต่อเมื่อ `name` เป็น `false`, `0`, หรือ array
ที่ว่างเปล่า** เหมาะมากสำหรับแสดงข้อความ "ไม่มีข้อมูล" เมื่อ list ว่าง

### 3. Comment `{{! ... }}`

ข้อความภายใน `{{! }}` จะถูก**ตัดทิ้งทั้งหมด** ไม่ปรากฏใน HTML ผลลัพธ์เลย ใช้สำหรับเขียนโน้ตอธิบาย
ใน template โดยไม่ให้ปนไปกับ HTML ที่ส่งออกจริง ทดสอบจริง:

Template: `Before{{! this is a comment and should not appear }}After`

ผลลัพธ์: `BeforeAfter`

### 4. Partial `{{> filename}}`

รวมไฟล์ template อื่นเข้ามาแทรก ณ จุดนั้น (คล้าย `#include` ของ C) มีประโยชน์มากสำหรับส่วนที่ใช้
ซ้ำในหลายหน้า เช่น header, footer, navigation bar ทดสอบจริงด้วยไฟล์ 2 ไฟล์:

`templates/_footer.html`:

```html
<footer>© 2026 Task App - {{site_name}}</footer>
```

`templates/partial_test.html`:

```html
<body>
<p>Content here</p>
{{> _footer.html}}
</body>
```

ผลลัพธ์จริงเมื่อ render ด้วย `ctx["site_name"] = "Demo"`:

```html
<body>
<p>Content here</p>
<footer>© 2026 Task App - Demo</footer>
</body>
```

Partial ถูกโหลดผ่าน loader เดียวกับ `crow::mustache::load` จึงอ่านจากโฟลเดอร์ฐานเดียวกันที่ตั้งไว้
ด้วย `set_base` — ไม่ต้องกำหนด path เพิ่ม

### สรุป Syntax ทั้งหมดในตารางเดียว

| Syntax | ความหมาย | ตัวอย่างค่าที่ทำให้ "แสดงผล" |
|---|---|---|
| `{{name}}` | แทรกค่า พร้อม HTML-escape อัตโนมัติ | ค่าใดๆ (string, number, bool) |
| `{{{name}}}` | แทรกค่าดิบๆ ไม่ escape (ใช้ระวัง!) | ค่าใดๆ |
| `{{&name}}` | เทียบเท่า `{{{name}}}` (unescape แบบ syntax อื่น) | ค่าใดๆ |
| `{{#name}}...{{/name}}` | แสดงเนื้อหาถ้า truthy, หรือวนซ้ำถ้าเป็น array | `true`, array ไม่ว่าง |
| `{{^name}}...{{/name}}` | แสดงเนื้อหาถ้า falsy (ตรงข้ามกับข้างบน) | `false`, `0`, array ว่าง |
| `{{! comment }}` | คอมเมนต์ ไม่แสดงผล | (ไม่มีผลต่อ output) |
| `{{> partial.html}}` | รวมไฟล์ template อื่นเข้ามา | (ไฟล์ partial ต้องมีอยู่จริง) |

### ตัวอย่างรวมทุก syntax: หน้ารวมสถานะ

```html
<!-- templates/status.html -->
<h2>{{title}}</h2>

{{#is_online}}
<p class="status-ok">ระบบทำงานปกติ ✓</p>
{{/is_online}}

{{^is_online}}
<p class="status-error">ระบบขัดข้อง ✗</p>
{{/is_online}}

{{#has_alerts}}
<ul>
  {{#alerts}}
  <li>{{message}} (ระดับ: {{level}})</li>
  {{/alerts}}
</ul>
{{/has_alerts}}

{{^has_alerts}}
<p>ไม่มีการแจ้งเตือน</p>
{{/has_alerts}}
```

```cpp
crow::mustache::context ctx;
ctx["title"] = "สถานะระบบ";
ctx["is_online"] = true;
ctx["has_alerts"] = true;

std::vector<crow::json::wvalue> alerts;
crow::json::wvalue a1; a1["message"] = "Disk เหลือน้อย"; a1["level"] = "warning";
alerts.push_back(std::move(a1));
ctx["alerts"] = std::move(alerts);
```

โค้ดชิ้นนี้แสดงให้เห็นว่าเราสามารถผสม boolean section, array section (loop) และ inverted section
เข้าด้วยกันเพื่อสร้างหน้าเว็บที่ตอบสนองต่อสถานะข้อมูลจริงได้ครบถ้วน โดยไม่ต้องเขียน `if`
แม้แต่บรรทัดเดียวใน template — ตรรกะการตัดสินใจทั้งหมดอยู่ในโค้ด C++ (การกำหนดค่า `is_online`,
`has_alerts`) ส่วน template มีหน้าที่แค่ "จัดวาง" เท่านั้น ตรงตามปรัชญา Logic-less Template

---

## 111.5 Render หน้าเว็บจากข้อมูลจริง: Task List จาก Part 108 (Step 885)

ถึงเวลานำความรู้ทั้งหมดมาประกอบร่างเป็นตัวอย่างที่ใช้งานได้จริง เราจะนำโครงสร้างข้อมูล **Task**
แบบเดียวกับที่ออกแบบไว้ใน Part 108 (โปรเจกต์ REST API CRUD ครบวงจร) มา render เป็นหน้าเว็บ HTML
แทนที่จะตอบกลับเป็น JSON เหมือนเดิม — ข้อมูลตั้งต้นเหมือนกันทุกประการ เปลี่ยนแค่ "รูปแบบการนำเสนอ"
(presentation) เท่านั้น

### โครงสร้างโปรเจกต์

```
task_ssr/
├── main.cpp
├── templates/
│   └── tasks.html
└── static/
    └── style.css
```

### ไฟล์ template: `templates/tasks.html`

```html
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <title>{{page_title}}</title>
    <link rel="stylesheet" href="/static/style.css">
</head>
<body>
    <h1>{{page_title}}</h1>
    <p>จำนวนงานทั้งหมด: {{task_count}}</p>

    {{#has_tasks}}
    <ul class="task-list">
        {{#tasks}}
        <li class="task {{#done}}task-done{{/done}}">
            <span class="task-id">#{{id}}</span>
            <span class="task-title">{{title}}</span>
            {{#done}}<span class="badge">เสร็จแล้ว</span>{{/done}}
            {{^done}}<span class="badge badge-pending">ยังไม่เสร็จ</span>{{/done}}
        </li>
        {{/tasks}}
    </ul>
    {{/has_tasks}}

    {{^has_tasks}}
    <p class="empty">ยังไม่มีงานในระบบ</p>
    {{/has_tasks}}
</body>
</html>
```

สังเกตว่าใน `<li>` เราใช้ทั้ง section (`{{#done}}`) และ inverted section (`{{^done}}`) ของฟิลด์
เดียวกันคือ `done` ภายใน scope ของแต่ละ task — เพื่อแสดง badge คนละแบบขึ้นอยู่กับสถานะ ซึ่งเป็น
เทคนิคที่ใช้บ่อยมากในการเขียน Mustache template แบบ logic-less

### โค้ด C++: `main.cpp`

```cpp
#include "crow.h"
#include "crow/mustache.h"

// โครงสร้างข้อมูล Task — เหมือนกับที่ใช้ใน Part 108 (REST API CRUD)
struct Task {
    int id;
    std::string title;
    bool done;
};

int main() {
    // ตั้งค่าโฟลเดอร์ฐานของ template ครั้งเดียวตอนเริ่มโปรแกรม
    crow::mustache::set_base("templates");

    // ข้อมูลจำลอง (ใน Part 108 ตัวนี้จะมาจาก SQLite/PostgreSQL ผ่าน Part 105-106)
    std::vector<Task> tasks = {
        {1, "เรียน Server-Side Rendering", true},
        {2, "ทดสอบ crow::mustache", false},
        {3, "Deploy ด้วย Docker", false},
    };

    crow::SimpleApp app;

    CROW_ROUTE(app, "/tasks")
    ([&tasks]() {
        crow::mustache::context ctx;
        ctx["page_title"] = "รายการงานทั้งหมด";
        ctx["task_count"] = tasks.size();
        ctx["has_tasks"] = !tasks.empty();

        // แปลง vector<Task> ให้เป็น vector<crow::json::wvalue>
        // เพื่อป้อนเข้า context สำหรับ array section {{#tasks}}
        std::vector<crow::json::wvalue> task_list;
        for (const auto& t : tasks) {
            crow::json::wvalue item;
            item["id"] = t.id;
            item["title"] = t.title;
            item["done"] = t.done;
            task_list.push_back(std::move(item));
        }
        ctx["tasks"] = std::move(task_list);

        auto page = crow::mustache::load("tasks.html");
        return page.render(ctx);
    });

    app.port(18080).run();
}
```

คอมไพล์:

```bash
g++ -std=c++17 -I/usr/local/include main.cpp -o app -lpthread
./app
```

ทดสอบ:

```bash
curl http://127.0.0.1:18080/tasks
```

ผลลัพธ์จริงที่ได้ (คัดลอกมาจากการรันจริง ไม่มีการแก้ไข):

```html
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <title>รายการงานทั้งหมด</title>
    <link rel="stylesheet" href="/static/style.css">
</head>
<body>
    <h1>รายการงานทั้งหมด</h1>
    <p>จำนวนงานทั้งหมด: 3</p>

    <ul class="task-list">
        <li class="task task-done">
            <span class="task-id">#1</span>
            <span class="task-title">เรียน Server-Side Rendering</span>
            <span class="badge">เสร็จแล้ว</span>

        </li>
        <li class="task ">
            <span class="task-id">#2</span>
            <span class="task-title">ทดสอบ crow::mustache</span>

            <span class="badge badge-pending">ยังไม่เสร็จ</span>
        </li>
        <li class="task ">
            <span class="task-id">#3</span>
            <span class="task-title">Deploy ด้วย Docker</span>

            <span class="badge badge-pending">ยังไม่เสร็จ</span>
        </li>
    </ul>

</body>
</html>
```

ถ้าเปิดผลลัพธ์นี้ด้วย browser จริง จะเห็นรายการงาน 3 รายการ พร้อม badge สีเขียว "เสร็จแล้ว"
สำหรับงานที่ทำเสร็จ และ badge สีส้ม "ยังไม่เสร็จ" สำหรับงานที่ยังไม่เสร็จ — ทั้งหมดนี้เกิดขึ้นจาก
โค้ด C++ ล้วนๆ โดยไม่มี JavaScript แม้แต่บรรทัดเดียว

### เปรียบเทียบโค้ดกับเวอร์ชัน REST API (Part 108)

ถ้าเป็นเวอร์ชัน REST API ที่เราเขียนใน Part 108 endpoint เดียวกันนี้จะมีหน้าตาประมาณนี้:

```cpp
CROW_ROUTE(app, "/api/tasks")
([&tasks]() {
    crow::json::wvalue result;
    std::vector<crow::json::wvalue> arr;
    for (const auto& t : tasks) {
        crow::json::wvalue item;
        item["id"] = t.id;
        item["title"] = t.title;
        item["done"] = t.done;
        arr.push_back(std::move(item));
    }
    result = std::move(arr);
    return result;  // Crow แปลงเป็น JSON string + Content-Type: application/json อัตโนมัติ
});
```

จะเห็นว่า**ตรรกะการดึงข้อมูลและแปลง `Task` เป็น `crow::json::wvalue` เหมือนกันทุกประการ**
ต่างกันแค่ขั้นตอนสุดท้าย — เวอร์ชัน REST API `return result` ตรงๆ (ได้ JSON) ส่วนเวอร์ชัน SSR
เอา `crow::json::wvalue` (ในชื่อ `context`) ไปยัด `page.render(ctx)` ต่ออีกขั้นหนึ่ง (ได้ HTML)
นี่คือเหตุผลที่ `crow::mustache::context` ถูกออกแบบให้เป็น alias ของ `crow::json::wvalue`
โดยตรง — เพื่อให้ย้ายจาก REST API ไป SSR (หรือทำทั้งสองแบบพร้อมกันจากข้อมูลชุดเดียวกัน) ทำได้
ง่ายที่สุดเท่าที่จะเป็นไปได้

---

## 111.6 Serve Static File (CSS) ผ่าน Crow (Step 886)

หน้าเว็บ SSR แทบทุกหน้าต้องมีไฟล์ประกอบ เช่น CSS สำหรับจัดสไตล์ Crow มีกลไก serve static file
ในตัวโดยไม่ต้องเขียน route เพิ่มเองเลย

### พฤติกรรม Default

Crow กำหนดค่า default ไว้ 2 ค่า (เป็น macro ที่ตั้งไว้ใน `crow/settings.h`):

- `CROW_STATIC_DIRECTORY` = `"static/"` — โฟลเดอร์บนดิสก์ที่ Crow จะอ่านไฟล์มา serve
- `CROW_STATIC_ENDPOINT` = `"/static/<path>"` — URL pattern ที่ผูกกับโฟลเดอร์ข้างต้น

พูดง่ายๆ คือ **ถ้ามีโฟลเดอร์ชื่อ `static/` อยู่ในตำแหน่งเดียวกับที่รันโปรแกรม (working directory)
Crow จะ serve ไฟล์ทุกไฟล์ในนั้นผ่าน URL `/static/<ชื่อไฟล์>` ให้อัตโนมัติทันที** ไม่ต้องเขียน
`CROW_ROUTE` เพิ่มแม้แต่บรรทัดเดียว

### ทดสอบจริง

สร้างไฟล์ `static/style.css`:

```css
body { font-family: sans-serif; background: #f4f4f4; }
.task-done { text-decoration: line-through; color: #888; }
.badge { padding: 2px 6px; border-radius: 4px; background: #4caf50; color: white; }
.badge-pending { background: #ff9800; }
```

รันโปรแกรม `main.cpp` จาก 111.5 ตามปกติ (ไม่ต้องแก้โค้ดเลย) แล้วเรียก:

```bash
curl http://127.0.0.1:18080/static/style.css
```

ผลลัพธ์จริงที่ได้:

```css
body { font-family: sans-serif; background: #f4f4f4; }
.task-done { text-decoration: line-through; color: #888; }
.badge { padding: 2px 6px; border-radius: 4px; background: #4caf50; color: white; }
.badge-pending { background: #ff9800; }
```

Crow ส่งไฟล์กลับมาตรงๆ พร้อมกำหนด `Content-Type` ให้ถูกต้องตามนามสกุลไฟล์ (`text/css` สำหรับ
`.css`) โดยอัตโนมัติ ทำให้แท็ก `<link rel="stylesheet" href="/static/style.css">` ในไฟล์
`templates/tasks.html` ทำงานได้ทันทีเมื่อเปิดด้วย browser จริง

### การปรับแต่ง (Custom static directory)

ถ้าต้องการเปลี่ยนชื่อโฟลเดอร์ (เช่นใช้ `public/` แทน `static/`) โดยไม่แก้ URL prefix สามารถเรียก
เมธอด runtime ได้ตอนสร้าง app:

```cpp
crow::SimpleApp app;
app.static_directory("public");  // เปลี่ยนโฟลเดอร์บนดิสก์ แต่ URL ยังเป็น /static/<path> เหมือนเดิม
```

ส่วนถ้าต้องการเปลี่ยน**ทั้ง URL prefix** (เช่นให้เป็น `/assets/<path>` แทน `/static/<path>`)
ต้อง `#define CROW_STATIC_ENDPOINT` **ก่อน** `#include "crow.h"` เพราะค่านี้ถูกกำหนดตอน compile-time
ผ่าน preprocessor macro ไม่ใช่ค่าที่เปลี่ยนได้ตอน runtime:

```cpp
#define CROW_STATIC_ENDPOINT "/assets/<path>"
#include "crow.h"
```

### ข้อควรระวังด้านความปลอดภัย: Path Traversal

เมธอดที่ Crow ใช้ภายใน (`set_static_file_info` แบบปกติ ที่ endpoint อัตโนมัติเรียกใช้) มีการ
ตรวจสอบ (sanitize) พาธไฟล์ก่อนเปิดอ่าน เพื่อป้องกันการโจมตีแบบ **Path Traversal** เช่น การส่ง
request `GET /static/../../etc/passwd` เพื่อพยายามอ่านไฟล์นอกโฟลเดอร์ที่อนุญาต แต่ Crow ก็มีเมธอด
เวอร์ชัน `_unsafe` (เช่น `set_static_file_info_unsafe`) ให้เรียกเองได้ในกรณีที่เขียน route
static file เอง **ห้ามใช้เวอร์ชัน `_unsafe` กับพาธที่มาจาก input ของผู้ใช้โดยไม่ตรวจสอบเอง**
เด็ดขาด เพราะเป็นการปิดกลไกป้องกัน Path Traversal ทิ้งไปโดยสิ้นเชิง — ใช้ endpoint อัตโนมัติของ
Crow (ที่ตรวจสอบให้อัตโนมัติ) ให้มากที่สุดเท่าที่จะทำได้

---

## 111.7 ฟอร์ม HTML + Post/Redirect/Get Pattern: เพิ่ม Task โดยไม่ใช้ JavaScript เลย (Step 887)

จุดแข็งอย่างหนึ่งของ SSR คือสามารถสร้างหน้าเว็บที่**โต้ตอบกับผู้ใช้ได้จริง** (ไม่ใช่แค่แสดงผล
อย่างเดียว) โดยไม่ต้องพึ่ง JavaScript เลยแม้แต่น้อย ผ่านกลไกพื้นฐานที่สุดของเว็บคือ **HTML Form**

### รูปแบบ Post/Redirect/Get (PRG)

ปัญหาคลาสสิกของฟอร์ม HTML คือ ถ้า handler ของ `POST` request render หน้า HTML กลับไปตรงๆ
เมื่อผู้ใช้กด "รีเฟรช" (F5) browser จะพยายามส่ง `POST` request เดิมซ้ำอีกครั้ง (เพราะ browser
จำ request ล่าสุดไว้) ทำให้ข้อมูลถูกเพิ่มซ้ำโดยไม่ตั้งใจ วิธีแก้ที่เป็นมาตรฐานของเว็บมานานหลายสิบปี
คือ pattern **Post/Redirect/Get**:

```
1. Browser ส่ง POST /tasks (พร้อมข้อมูลฟอร์ม)
2. Server บันทึกข้อมูล แล้วตอบกลับด้วย HTTP 302/303 Redirect ไปที่ GET /tasks
3. Browser ทำตาม Redirect โดยอัตโนมัติ ส่ง GET /tasks
4. Server render หน้า HTML ปกติ (คราวนี้มีข้อมูลใหม่รวมอยู่ด้วย)
5. ถ้าผู้ใช้กดรีเฟรชตอนนี้ จะรีเฟรชแค่ GET /tasks (ปลอดภัย ไม่เพิ่มข้อมูลซ้ำ)
```

### เพิ่มฟอร์มใน template

แก้ไข `templates/tasks.html` เพิ่มฟอร์มเข้าไป:

```html
<h2>เพิ่มงานใหม่</h2>
<form action="/tasks" method="POST">
    <input type="text" name="title" placeholder="ชื่องาน" required>
    <button type="submit">เพิ่ม</button>
</form>
```

### เขียน handler รับ POST

```cpp
// เก็บข้อมูลไว้นอก lambda ด้วย static เพื่อให้ทุก request เข้าถึงชุดข้อมูลเดียวกัน
// (ในระบบจริงจุดนี้คือการเขียนลง SQLite/PostgreSQL ตาม Part 105-106)
static std::vector<Task> tasks = {
    {1, "เรียน Server-Side Rendering", true},
    {2, "ทดสอบ crow::mustache", false},
};
static int next_id = 3;

CROW_ROUTE(app, "/tasks").methods("POST"_method)
([](const crow::request& req) {
    // req.body คือ raw body ของ request เช่น "title=%E0%B8%87%E0%B8%B2%E0%B8%99..."
    // (ฟอร์ม HTML ส่งข้อมูลแบบ application/x-www-form-urlencoded โดย default)
    // เติม "?" นำหน้าเพื่อใช้ตัว parser เดียวกับ query string ใน URL แกะค่าออกมา
    auto body = crow::query_string("?" + req.body);
    const char* title = body.get("title");

    if (title != nullptr && std::string(title).size() > 0) {
        tasks.push_back({next_id++, std::string(title), false});
    }

    crow::response res;
    res.moved("/tasks");  // HTTP 302 Found + Location: /tasks
    return res;
});
```

### ทดสอบจริงแบบ end-to-end

```bash
# 1. ตรวจสอบจำนวนงานก่อนเพิ่ม
curl -s http://127.0.0.1:18081/tasks | grep "จำนวนงานทั้งหมด"
# ผลลัพธ์: <p>จำนวนงานทั้งหมด: 2</p>

# 2. ส่งฟอร์มเพิ่มงานใหม่ (จำลองสิ่งที่ browser ทำเมื่อกด submit)
curl -s -i -X POST -d "title=งานใหม่ทดสอบ" http://127.0.0.1:18081/tasks
```

ผลลัพธ์จริงของขั้นตอนที่ 2 (header ของ response):

```
HTTP/1.1 302 Found
Location: /tasks
Content-Length: 0
Server: Crow/master
Connection: Keep-Alive
```

```bash
# 3. ตรวจสอบอีกครั้งหลังเพิ่ม
curl -s http://127.0.0.1:18081/tasks | grep -E "จำนวนงานทั้งหมด|งานใหม่ทดสอบ"
```

ผลลัพธ์จริง:

```
<p>จำนวนงานทั้งหมด: 3</p>
            <span class="task-title">งานใหม่ทดสอบ</span>
```

ข้อมูลถูกเพิ่มเข้าไปจริง และแม้ข้อความเป็นภาษาไทยที่ถูก percent-encode ตอนส่งผ่าน HTTP
(`application/x-www-form-urlencoded`) `crow::query_string` ก็ decode กลับมาเป็น UTF-8 ที่ถูกต้อง
ให้อัตโนมัติ

### ทางเลือกของ HTTP Status Code สำหรับ Redirect

`crow::response` มีเมธอดสำเร็จรูปให้เลือกใช้หลายแบบ:

| เมธอด | Status Code | ความหมาย | เหมาะกับ PRG หรือไม่ |
|---|---|---|---|
| `res.redirect(url)` | 307 Temporary Redirect | บังคับให้ client ใช้ method เดิม (POST) ซ้ำ | **ไม่เหมาะ** — จะกลายเป็น POST วนซ้ำ |
| `res.moved(url)` | 302 Found | ตามธรรมเนียมเก่า browser ส่วนใหญ่เปลี่ยนเป็น GET ให้ | ใช้ได้ (เป็นที่นิยมมากที่สุด) |
| `res.code = 303; res.set_header("Location", url);` | 303 See Other | ระบุชัดเจนตามมาตรฐาน HTTP ว่า "ไปดูที่นี่ด้วย GET" | **เหมาะที่สุด** ตามหลักวิชาการ |

ทดสอบ 303 แบบ manual ก็ทำได้ผลลัพธ์ถูกต้องเช่นกัน:

```cpp
crow::response res;
res.code = 303;
res.set_header("Location", "/tasks");
return res;
```

ผลลัพธ์จริง:

```
HTTP/1.1 303 See Other
Location: /tasks
Content-Length: 0
```

ในทางปฏิบัติ ทั้ง `res.moved()` (302) และการตั้ง `res.code = 303` เอง ใช้งานได้ดีกับ browser
สมัยใหม่ทุกตัว — ที่**ต้องหลีกเลี่ยงเด็ดขาด**สำหรับ pattern PRG คือ `res.redirect()` (307) เพราะ
มันจงใจรักษา HTTP method เดิมไว้ (POST) ซึ่งเป็นพฤติกรรมที่ถูกต้องสำหรับกรณีอื่น (เช่น API ที่
ต้องการ redirect แบบคง method) แต่ผิดสำหรับ PRG pattern

---

## 111.8 ข้อจำกัดของการทำ Templating ใน C++ (Step 888)

ถึงจุดนี้เราได้เห็นแล้วว่า `crow::mustache` ทำงานได้จริงและมีประโยชน์ แต่การพูดตรงไปตรงมา
(ไม่ over-sell) เป็นสิ่งสำคัญมาก — C++ **ไม่ได้ถูกออกแบบมาเพื่องานเว็บตั้งแต่แรก** ต่างจาก
PHP, Ruby (Rails), Python (Django/Flask), JavaScript (Next.js/Express) หรือแม้แต่ Go ที่ระบบ
นิเวศน์ถูกสร้างขึ้นมาโดยคำนึงถึงงานเว็บเป็นหลักตั้งแต่ต้น

### ข้อจำกัดที่ควรรู้จริงๆ

1. **ไม่มี Hot Reload สำหรับโค้ด C++** — แม้ `crow::mustache::load()` จะอ่านไฟล์ template จาก
   ดิสก์ **ใหม่ทุกครั้ง** ที่มีการ request เข้ามา (ทดสอบแล้วในหัวข้อก่อนหน้าว่าแก้ไฟล์ `.html`
   แล้วเห็นผลทันทีโดยไม่ต้อง restart) แต่นั่นใช้ได้แค่กับไฟล์ template เท่านั้น ถ้าแก้โค้ด C++
   (เช่น เปลี่ยน logic การดึงข้อมูล, เพิ่ม route ใหม่) **ต้อง compile ใหม่และ restart server
   ทุกครั้ง** ต่างจาก Node.js/Python/Ruby ที่มักมีเครื่องมือ (nodemon, Flask debug mode,
   `rails server` แบบ development) ที่ reload โค้ด backend ให้อัตโนมัติเมื่อไฟล์เปลี่ยน
   ทำให้ development loop ของ C++ web app ช้ากว่าอย่างมีนัยสำคัญในขั้นตอนพัฒนา

2. **ไม่มี Ecosystem ของ Template Engine ที่หลากหลายและเป็นผู้ใหญ่เท่า** — ภาษาสายเว็บมี
   template engine ให้เลือกมากมายพร้อมฟีเจอร์ครบ (template inheritance, macro, filter,
   auto-escaping ที่ปรับแต่งได้ละเอียด) เช่น Jinja2 (Python), ERB/Slim (Ruby), Blade (PHP),
   Handlebars/EJS (JavaScript) ส่วน Mustache ของ Crow มีฟีเจอร์เพียงพอสำหรับงานพื้นฐาน
   (ตามที่เราทดสอบใน 111.3-111.4) แต่ **ไม่มี** template inheritance (layout ที่หน้าอื่น
   "extends" ได้), ไม่มี loop แบบระบุ index/counter ในตัว, ไม่มี filter/pipe แปลงข้อมูล
   (เช่น จัดรูปแบบวันที่, ตัดคำ) ต้องทำ logic เหล่านี้ในโค้ด C++ ก่อนส่งเข้า context เองทั้งหมด

3. **การ Debug ยากกว่า** — ถ้า template ผิดพลาด (เช่น เขียน tag ผิด หรือ context ไม่มี field
   ที่ template ต้องการ) ข้อความ error ที่ได้มักไม่ชัดเจนเท่าภาษาที่ born-for-web ซึ่งมัก
   แสดง stack trace พร้อมเลขบรรทัดของ template ที่ผิดพลาดตรงๆ ผ่านหน้า error page ที่ออกแบบมา
   ให้อ่านง่าย

4. **Ecosystem ของเครื่องมือเสริมน้อยกว่ามาก** — ภาษาสายเว็บมักมี Object-Relational Mapper
   (ORM) ที่สมบูรณ์, Migration tool, Admin panel สำเร็จรูป, Authentication library ที่ plug-and-
   play ในขณะที่ C++ web development (Crow, Pistache ที่เรียนมาใน Part 102-104) ยังต้องประกอบ
   ชิ้นส่วนเหล่านี้เองเกือบทั้งหมด (เราเขียน JWT auth เองใน Part 110, เขียน SQL query เองใน
   Part 105-106)

5. **Compile Time เป็นต้นทุนที่มองข้ามไม่ได้** — ทุกครั้งที่แก้โค้ด ต้อง compile ใหม่ (แม้จะใช้
   เวลาไม่กี่วินาทีสำหรับโปรเจกต์เล็ก) แต่สำหรับโปรเจกต์ขนาดใหญ่ที่ใช้ C++ เต็มรูปแบบ compile
   time อาจกลายเป็นคอขวดของ development loop ได้จริง

### แล้วทำไมยังต้องเรียน SSR ด้วย C++?

คำตอบไม่ใช่ "เพราะ C++ เหมาะกับงานนี้ที่สุด" แต่เป็นเหตุผลเชิงบริบทที่สำคัญกว่า:

- **Performance**: สำหรับงานที่ต้องการความเร็วสูงสุดต่อ request (เช่น ระบบ embedded ที่มี HTTP
  interface, เกม server ที่ serve หน้า admin panel เล็กๆ ควบคู่ไปกับ game logic หลัก) การมี
  HTTP server และ template engine อยู่ใน binary เดียวกับระบบหลักที่เขียนด้วย C++ อยู่แล้ว
  ทำให้ไม่ต้องเพิ่มภาษาที่สองเข้ามาในระบบ ลดความซับซ้อนของ deployment
- **Resource-constrained environment**: ระบบที่มีทรัพยากรจำกัด (RAM/CPU น้อย) เช่น
  embedded system หรือ IoT device ที่ต้องมี web interface เล็กๆ สำหรับตั้งค่า C++ ใช้
  ทรัพยากรน้อยกว่า runtime ของภาษาสาย interpreted/JIT มาก
- **ทีมที่เชี่ยวชาญ C++ อยู่แล้ว**: ในองค์กรที่ core system เขียนด้วย C++ ทั้งหมด (เช่น
  ระบบ trading, ระบบควบคุมอุตสาหกรรม) การเพิ่ม endpoint HTML เล็กๆ น้อยๆ ด้วยภาษาเดียวกัน
  ง่ายกว่าการดึงทีมใหม่มาดูแล stack ภาษาที่สอง
- **การเรียนรู้แนวคิด**: การเข้าใจว่า SSR ทำงานอย่างไร "จากภายใน" (ผ่านภาษาที่ไม่มี
  magic ซ่อนอยู่มากเท่าภาษาสายเว็บ) ทำให้เข้าใจหลักการเบื้องหลัง framework เว็บอื่นๆ
  ได้ลึกซึ้งขึ้นเมื่อย้ายไปใช้ภาษาเหล่านั้นในอนาคต

**สรุปแบบตรงไปตรงมาที่สุด**: ถ้าเป้าหมายคือสร้างเว็บไซต์เนื้อหาทั่วไป (บล็อก, เว็บบริษัท,
E-Commerce ขนาดกลาง) การใช้ Next.js, Rails, Django หรือ Laravel จะให้ productivity สูงกว่า
C++ + Crow อย่างชัดเจนในแทบทุกมิติของการพัฒนา แต่ถ้าเป้าหมายคือระบบที่มี C++ เป็นแกนหลักอยู่แล้ว
และต้องการ HTTP interface แบบเบาๆ ประกอบเข้าไป ความรู้เรื่อง SSR ด้วย `crow::mustache` ใน
Part นี้คือเครื่องมือที่ใช้งานได้จริงและมีที่ทางของมันเอง

---

## ข้อผิดพลาดที่พบบ่อย (Common Pitfalls)

1. **ลืมเรียก `crow::mustache::set_base()` ก่อนเรียก `load()`** — ถ้าไม่ตั้งค่า base path
   Crow จะพยายามอ่านไฟล์จาก working directory ปัจจุบันตรงๆ (relative path เดิมที่ใส่ใน
   `load()`) ซึ่งมักทำให้หาไฟล์ template ไม่เจอถ้าโปรแกรมถูกรันจากคนละโฟลเดอร์กับตอน develop
   ควรเรียก `set_base()` ครั้งเดียวตอนต้นโปรแกรม ก่อนสร้าง route ใดๆ

2. **ใช้ `{{{variable}}}` (unescaped) กับข้อมูลที่มาจากผู้ใช้โดยตรง** — เป็นช่องโหว่ XSS ที่
   ตรวจพบได้ยากในตอน dev (เพราะมักทดสอบด้วยข้อมูลที่ "สะอาด" เอง) แต่เป็นอันตรายจริงใน production
   เมื่อผู้ใช้จริงป้อนข้อมูลที่มี `<script>` เข้ามา ให้ใช้ `{{variable}}` (escaped) เป็นค่า
   default เสมอ

3. **ลืมว่า `page.render(ctx)` คืนค่าเป็น `rendered_template` ไม่ใช่ `std::string`** —
   ถ้าต้องการ `std::string` ตรงๆ (เช่น จะเอาไปต่อกับข้อความอื่น หรือ log) ต้องเรียก `.dump()`
   เพิ่ม (`page.render(ctx).dump()`) แต่ถ้าจะ `return` จาก Crow route handler ตรงๆ ไม่ต้องแปลง
   เพราะ Crow รู้จัก `rendered_template` อยู่แล้วและจะตั้ง `Content-Type: text/html` ให้อัตโนมัติ

4. **สับสนระหว่าง `res.redirect()` (307) กับ `res.moved()` (302) ใน pattern PRG** — การใช้
   `res.redirect()` หลัง `POST` form จะทำให้ browser ส่ง `POST` ซ้ำไปยัง URL ปลายทางอีกครั้ง
   (เพราะ 307 การันตีคง method เดิม) ซึ่งมักไม่ใช่พฤติกรรมที่ต้องการ ให้ใช้ `res.moved()`
   (302) หรือกำหนด `res.code = 303` เองสำหรับ pattern "บันทึกข้อมูลแล้วพากลับไปหน้ารายการ"

5. **ลืมสร้างโฟลเดอร์ `static/` หรือวางไฟล์ผิดตำแหน่ง** — Crow serve static file จากโฟลเดอร์
   `static/` **สัมพัทธ์กับ working directory ตอนรันโปรแกรม** ไม่ใช่สัมพัทธ์กับตำแหน่งไฟล์
   source code ถ้ารันโปรแกรมจากคนละที่ (เช่นรันผ่าน CI/CD หรือ systemd ที่ set `WorkingDirectory`
   ไว้คนละที่ — ดู Part 112) ไฟล์ static อาจหาไม่เจอ ต้องตรวจสอบ working directory ให้ตรงเสมอ

6. **คาดหวัง Hot Reload ของโค้ด C++ เหมือนภาษา interpreted** — ต้อง compile ใหม่และ restart
   server ทุกครั้งที่แก้ไข logic ใน `.cpp` (ต่างจากการแก้ไฟล์ `.html` ที่ `load()` อ่านใหม่จาก
   ดิสก์ทุก request อยู่แล้วโดยไม่ต้อง restart) การไม่เข้าใจความต่างนี้ทำให้เสียเวลา debug
   ว่า "ทำไมแก้โค้ดแล้วไม่เห็นผล" ทั้งที่จริงๆ แค่ลืม compile ใหม่

7. **ใส่ HTML ที่ไม่ปิด tag ให้ครบใน template** — เพราะ Mustache เป็น text template ธรรมดา
   ไม่ได้ parse โครงสร้าง HTML จริงๆ มันจะไม่เตือนถ้า tag เปิดแล้วไม่ปิด (เช่นลืม `</li>`)
   ผลลัพธ์คือหน้าเว็บแสดงผลเพี้ยนโดยไม่มี error message ใดๆ ให้ตรวจสอบ HTML ด้วยสายตาหรือ
   HTML validator เสมอหลังแก้ template

---

## แบบฝึกหัดท้ายบท

1. เขียนหน้า SSR `/profile` ที่แสดงข้อมูลผู้ใช้ (ชื่อ, อีเมล, จำนวนวันที่เป็นสมาชิก) โดยใช้
   `crow::mustache::context` และ `{{variable}}` เท่านั้น (ไม่ต้องใช้ section)

2. ขยายตัวอย่าง Task list ใน 111.5 ให้มี field เพิ่มเติมคือ `priority` (ค่าเป็น `"high"`,
   `"medium"`, `"low"`) แล้วใช้ section เพื่อแสดง badge สีต่างกันตาม priority (`{{#is_high}}`,
   `{{#is_medium}}`, `{{#is_low}}` เป็น boolean flag ที่คำนวณจากฝั่ง C++ ก่อนส่งเข้า context)

3. เพิ่ม endpoint `POST /tasks/<int>/toggle` ที่สลับสถานะ `done` ของ task ตาม id ที่ระบุ แล้ว
   redirect กลับไปที่ `/tasks` ด้วย pattern PRG ที่ถูกต้อง (ห้ามใช้ `res.redirect()`)

4. เขียน partial template ชื่อ `_navbar.html` ที่มีเมนู "หน้าแรก | งานทั้งหมด | โปรไฟล์" แล้ว
   include เข้าไปในทุกหน้าด้วย `{{> _navbar.html}}`

5. ทดสอบเปรียบเทียบ Content-Length ของ response ระหว่าง endpoint แบบ SSR (`/tasks` คืน HTML)
   กับ endpoint แบบ REST API (`/api/tasks` คืน JSON) ด้วยข้อมูลชุดเดียวกัน แล้ววิเคราะห์ว่า
   เพราะเหตุใด SSR จึงมีขนาด response ใหญ่กว่า JSON เสมอ

6. ลองสร้างสถานการณ์ที่ context ไม่มี field ที่ template ต้องการ (เช่น template เขียน
   `{{author}}` แต่ context ไม่ได้ set ค่า `author` ไว้) สังเกตว่า Crow แสดงผลอย่างไร
   (ไม่ crash หรือ throw exception หรือไม่) แล้วอภิปรายว่าพฤติกรรมนี้ปลอดภัยหรือไม่ในระบบจริง

### แนวทางเฉลยข้อ 1

```cpp
#include "crow.h"
#include "crow/mustache.h"

int main() {
    crow::mustache::set_base("templates");
    crow::SimpleApp app;

    CROW_ROUTE(app, "/profile")
    ([]() {
        crow::mustache::context ctx;
        ctx["name"] = "สมหญิง ใจดี";
        ctx["email"] = "somying@example.com";
        ctx["days_since_joined"] = 428;

        auto page = crow::mustache::load("profile.html");
        return page.render(ctx);
    });

    app.port(18080).run();
}
```

`templates/profile.html`:

```html
<!DOCTYPE html>
<html lang="th">
<head><meta charset="UTF-8"><title>โปรไฟล์ผู้ใช้</title></head>
<body>
    <h1>โปรไฟล์ของ {{name}}</h1>
    <p>อีเมล: {{email}}</p>
    <p>เป็นสมาชิกมาแล้ว {{days_since_joined}} วัน</p>
</body>
</html>
```

ทดสอบ:

```bash
g++ -std=c++17 -I/usr/local/include profile.cpp -o profile_app -lpthread
./profile_app &
curl http://127.0.0.1:18080/profile
```

ผลลัพธ์ที่คาดหวัง:

```html
<!DOCTYPE html>
<html lang="th">
<head><meta charset="UTF-8"><title>โปรไฟล์ผู้ใช้</title></head>
<body>
    <h1>โปรไฟล์ของ สมหญิง ใจดี</h1>
    <p>อีเมล: somying@example.com</p>
    <p>เป็นสมาชิกมาแล้ว 428 วัน</p>
</body>
</html>
```

### แนวทางเฉลยข้อ 3

```cpp
CROW_ROUTE(app, "/tasks/<int>/toggle").methods("POST"_method)
([](int task_id) {
    for (auto& t : tasks) {
        if (t.id == task_id) {
            t.done = !t.done;
            break;
        }
    }
    crow::response res;
    res.code = 303;                     // 303 See Other: ระบุชัดเจนว่าให้ตามไปด้วย GET
    res.set_header("Location", "/tasks");
    return res;
});
```

ในฟอร์ม HTML ของแต่ละแถว task สามารถแทรกปุ่ม toggle แบบนี้ในเนื้อหา `{{#tasks}}...{{/tasks}}`:

```html
<form action="/tasks/{{id}}/toggle" method="POST" style="display:inline">
    <button type="submit">สลับสถานะ</button>
</form>
```

จุดสำคัญของเฉลยนี้คือการใช้ `res.code = 303` แทน `res.redirect()` (307) ตามเหตุผลที่อธิบายไว้
ใน 111.7 — ถ้าใช้ 307 ทุกครั้งที่กด "สลับสถานะ" แล้วรีเฟรชหน้า จะเกิดการยิง `POST
/tasks/<id>/toggle` ซ้ำโดยไม่ตั้งใจ ในขณะที่ 303 บังคับให้ browser ตามไปด้วย `GET /tasks`
เสมอ ทำให้การรีเฟรชปลอดภัย

---

## สรุปท้ายบท

ใน Part นี้เราได้:

- เข้าใจความแตกต่างระหว่าง Server-Side Rendering (SSR) กับสถาปัตยกรรม REST API + SPA อย่าง
  ชัดเจนทั้งใน flow การทำงานและผลกระทบด้าน SEO, performance, และความซับซ้อนของ stack
- เรียนรู้และทดสอบ `crow::mustache` ครบทุก syntax หลัก: `{{variable}}` (พร้อม auto-escape
  ป้องกัน XSS), `{{{variable}}}` (unescaped), `{{#section}}`, `{{^section}}` (inverted),
  `{{! comment }}`, และ `{{> partial}}`
- Render หน้าเว็บ Task list จากข้อมูลจริงในหน่วยความจำ พร้อมเปรียบเทียบโค้ดกับเวอร์ชัน REST API
  ที่เคยเขียนใน Part 108 เห็นชัดว่าตรรกะการดึงข้อมูลเหมือนกัน ต่างแค่ขั้นตอนสุดท้าย
- Serve static file (CSS) ผ่านกลไก static directory ในตัวของ Crow ได้โดยไม่ต้องเขียน route
  เพิ่มเอง พร้อมเข้าใจการป้องกัน Path Traversal ในตัว
- สร้างฟอร์ม HTML ที่ทำงานร่วมกับ backend C++ ได้จริงผ่าน pattern Post/Redirect/Get (PRG)
  โดยไม่ใช้ JavaScript เลยแม้แต่บรรทัดเดียว
- ประเมินข้อจำกัดของ C++ ในงาน web templating อย่างตรงไปตรงมา ทั้งเรื่อง hot reload, ความ
  หลากหลายของ ecosystem, และต้นทุนของ compile time เทียบกับภาษาที่ born-for-web

ทุกโค้ดตัวอย่างและผลลัพธ์ใน Part นี้ผ่านการคอมไพล์และรันจริงด้วย g++ 13.3.0 และทดสอบด้วย `curl`
จริง ไม่ใช่ผลลัพธ์ที่แต่งขึ้น — เมื่อลงมือทำตามด้วยตัวเอง ควรได้ผลลัพธ์เดียวกันทุกประการ

Part สุดท้ายของ Module I ที่เหลือคือการนำแอปพลิเคชัน C++ ที่เราสร้างมาตลอด Module นี้ (REST API,
WebSocket, Authentication, และ SSR) ไป **Deploy จริง** สู่ระบบ production — เรื่องของ Docker,
Nginx reverse proxy, HTTPS, และการรันเป็น background service บน Linux server จริง

**ต่อไป:** [Part 112 — Deploy: Docker, Nginx, Production Server](./part-112-deploy-docker-nginx.md)
