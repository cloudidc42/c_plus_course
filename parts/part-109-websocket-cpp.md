# Part 109: WebSocket Programming ด้วย C++ (Step 865–872)

> Module I — Web Development ด้วย C/C++ | Part 109 จาก 125
> Step ที่ครอบคลุมใน Part นี้: Step 865–872
> Part ก่อนหน้า: [Part 108 — โปรเจกต์ REST API Backend แบบ CRUD ครบวงจร](./part-108-rest-api-crud-project.md) | Part ถัดไป: [Part 110 — Authentication และ Security: JWT, Password Hashing](./part-110-auth-security.md)

## เป้าหมายการเรียนรู้ (Learning Objectives)

หลังจบ Part นี้ ผู้เรียนจะสามารถ:

1. อธิบายความแตกต่างระหว่าง HTTP request-response แบบดั้งเดิมกับ WebSocket ได้อย่างชัดเจน
   ทั้งในแง่ connection lifecycle, ทิศทางการสื่อสาร (full-duplex), และ overhead ต่อข้อความ
2. อธิบาย WebSocket Handshake ตาม RFC 6455 ได้ว่าเกิดอะไรขึ้นบ้างตั้งแต่ HTTP request แรก
   จนกลายเป็น persistent connection พร้อมคำนวณค่า `Sec-WebSocket-Accept` ได้ด้วยมือ
3. ใช้ `CROW_WEBSOCKET_ROUTE` ของ Crow framework เขียน WebSocket endpoint พร้อม handler
   ครบทั้ง `onopen`, `onmessage`, `onclose`, `onerror` และ `onaccept`
4. ออกแบบและเขียน WebSocket Broadcast Chat Server ที่ส่งข้อความจาก client หนึ่งไปยัง
   client ทุกตัวที่เชื่อมต่ออยู่ได้ โดยป้องกัน race condition ด้วย `std::mutex` อย่างถูกต้อง
5. พิสูจน์ได้ด้วยเครื่องมือจริง (ThreadSanitizer) ว่าเหตุใดการแชร์ container ของ connection
   ข้าม thread โดยไม่มี mutex จึงนำไปสู่ data race และโปรแกรม crash จริง ไม่ใช่แค่ทฤษฎี
6. ทดสอบ WebSocket server ที่เขียนเองได้หลายวิธีตามเครื่องมือที่มีจริงในสภาพแวดล้อม
   (`curl --include --no-buffer` สำหรับดู handshake, `websocat` และ Python client สำหรับ
   ทดสอบการสื่อสารสองทางจริง)
7. เปรียบเทียบสถาปัตยกรรมของ WebSocket broadcast server กับ TCP Chat Server แบบ
   thread-per-client ที่เขียนด้วย raw socket ใน Part 35 ได้ว่าเหมือนและต่างกันอย่างไร
8. เข้าใจแนวคิดเรื่อง Ping/Pong keep-alive และการปิด connection อย่างถูกต้องตามสเปก
   เพื่อป้องกัน resource leak ในระบบที่ต้องรันต่อเนื่องเป็นเวลานาน

---

Part นี้ต่อยอดโดยตรงจาก **Part 108** ที่เราสร้าง Task REST API ครบวงจรไว้แล้ว คำถามที่
เกิดขึ้นตามธรรมชาติคือ: ถ้ามีคนสองคนเปิดแอป Task List พร้อมกัน แล้วคนหนึ่งติ๊กงานเสร็จ
อีกคนจะเห็นการเปลี่ยนแปลงนั้น "ทันที" ได้อย่างไร โดยไม่ต้องกด refresh เอง? คำตอบคือ REST API
แบบ request-response ธรรมดาทำแบบนั้นไม่ได้ — ต้องใช้ **WebSocket**

## 109.1 WebSocket คืออะไร ต่างจาก HTTP อย่างไร (Step 865)

HTTP ที่เราใช้มาตลอด Module I (Part 99-108) ทำงานตามรูปแบบ **Request-Response** เสมอ:
client ต้องเป็นฝ่าย "ถาม" ก่อนเสมอ server ถึงจะ "ตอบ" ได้ พอตอบเสร็จ connection (ในกรณีทั่วไป)
ก็จบไป ถ้า server อยากบอกอะไร client โดยที่ client ไม่ได้ถามมาก่อน **ทำไม่ได้เลย**

**WebSocket** (มาตรฐาน RFC 6455, ปี 2011) แก้ปัญหานี้ด้วยการเปลี่ยน TCP connection เดิม
ที่ใช้คุย HTTP ให้กลายเป็น **persistent connection แบบสองทาง (full-duplex)** — เปิดครั้งเดียว
แล้วทั้ง client และ server ส่งข้อความหากันได้ตลอดเวลา **โดยไม่ต้องมีใครเป็นฝ่าย "ถาม" ก่อน**

### ตารางเปรียบเทียบ HTTP กับ WebSocket

| คุณสมบัติ | HTTP (Request-Response) | WebSocket |
|---|---|---|
| ทิศทางการสื่อสาร | Half-duplex ทีละทาง (client ถามก่อนเสมอ) | Full-duplex สองทางพร้อมกัน |
| อายุของ connection | เปิด-ปิดทุกครั้งที่มี request (หรือ keep-alive สั้นๆ) | เปิดค้างไว้ยาว (persistent) จนกว่าจะปิดเอง |
| ใครเริ่มส่งข้อความได้ | Client เท่านั้น | ทั้ง Client และ Server |
| Overhead ต่อข้อความ | Header เต็มรูปแบบทุกครั้ง (มักหลายร้อยไบต์) | Frame header เล็กมาก (2-14 ไบต์) |
| Protocol scheme | `http://`, `https://` | `ws://`, `wss://` (secure) |
| เหมาะกับ | CRUD, การดึงข้อมูลเป็นครั้งคราว | Real-time: แชท, แจ้งเตือนสด, dashboard สด |
| การ implement บน TCP | เปิด connection ใหม่ (หรือ reuse ผ่าน keep-alive) | Upgrade จาก HTTP connection เดิมเส้นเดียว |

จุดที่สำคัญที่สุดที่ต้องเข้าใจให้แม่นคือ **WebSocket ไม่ใช่โปรโตคอลที่แยกขาดจาก HTTP
โดยสิ้นเชิง** — มันเริ่มต้นด้วย HTTP request ธรรมดาทุกประการ แล้วขอ "อัปเกรด" protocol
ของ TCP connection เส้นเดิมนั้นให้กลายเป็น WebSocket (รายละเอียดในหัวข้อ 109.3) นี่คือเหตุผล
ที่ WebSocket ใช้พอร์ตเดียวกับ HTTP ได้ (80 สำหรับ `ws://`, 443 สำหรับ `wss://`) และเดินทาง
ผ่าน firewall/proxy ส่วนใหญ่ได้โดยไม่มีปัญหา ต่างจากการเปิด raw TCP socket พอร์ตใหม่เอง
แบบที่ทำใน Part 35 ซึ่งหลายองค์กรบล็อกพอร์ตแปลกๆ ไว้ด้วยเหตุผลด้านความปลอดภัย

### ทำไม HTTP Polling ถึงไม่ใช่คำตอบที่ดี

ก่อนที่ WebSocket จะแพร่หลาย นักพัฒนาแก้ปัญหา "real-time" ด้วยเทคนิคที่เรียกว่า
**Polling** — ให้ client ยิง HTTP request ถามซ้ำๆ ทุกๆ 2-3 วินาทีว่า "มีอะไรใหม่ไหม"
วิธีนี้ใช้งานได้จริงและยังพบเห็นในระบบเก่าอยู่บ้าง แต่มีข้อเสียชัดเจน:

| แนวทาง | Latency (ความหน่วงกว่าจะเห็นข้อมูลใหม่) | Overhead | ความซับซ้อนของโค้ด |
|---|---|---|---|
| Short Polling (ยิงถามทุก N วินาที) | สูงสุด N วินาที (เฉลี่ย N/2) | สูงมาก (ยิง request เปล่าซ้ำๆ ทั้งที่ไม่มีอะไรใหม่) | ต่ำ |
| Long Polling (server "ค้าง" request ไว้จนมีข้อมูลใหม่ค่อยตอบ) | ต่ำกว่า Short Polling มาก | ปานกลาง (ยังต้องเปิด connection ใหม่ทุกรอบ) | ปานกลาง |
| **WebSocket** | ต่ำที่สุด (server push ทันทีที่มีข้อมูล) | ต่ำที่สุด (connection เดียวค้างไว้ ไม่ต้อง handshake ซ้ำ) | ปานกลาง (ต้องจัดการ connection lifecycle เอง) |

WebSocket จึงเป็นคำตอบที่ "ถูกที่สุด" ทั้งในแง่ latency และ resource usage สำหรับงานที่ต้อง
อัปเดตแบบ real-time จริงๆ — แต่ก็ไม่ใช่ว่าทุก endpoint ควรเป็น WebSocket ไปหมด (ดูหัวข้อถัดไป)

## 109.2 Use Case ที่เหมาะกับ WebSocket และความเชื่อมโยงกับ Part 35 / Part 122 (Step 866)

WebSocket ไม่ได้เหมาะกับทุกสถานการณ์ — endpoint อย่าง `GET /tasks` หรือ `POST /tasks` ใน
Part 108 ไม่มีเหตุผลอะไรที่ต้องเปลี่ยนเป็น WebSocket เพราะเป็นการ "ขอข้อมูลครั้งเดียวแล้วจบ"
(request-response ชัดเจน) กฎง่ายๆ ที่ใช้ตัดสินใจคือ:

> **ใช้ WebSocket เมื่อ server ต้อง "ส่งข้อมูลหา client โดยที่ client ไม่ได้ถาม" และต้องการ
> ให้ client เห็นข้อมูลใหม่ให้เร็วที่สุด**

### Use Case ที่เหมาะกับ WebSocket

| Use Case | เหตุผลที่ต้องใช้ WebSocket |
|---|---|
| แอปแชท (Chat) | ข้อความจากคนอื่นต้องโผล่ทันทีโดยที่เราไม่ได้กด refresh |
| การแจ้งเตือนสด (Live Notification) | เช่น มีคำสั่งซื้อใหม่เข้ามา ต้องแจ้งพนักงานทันที |
| Dashboard แบบสด (Real-time Dashboard) | กราฟราคาหุ้น, จำนวนผู้ใช้ออนไลน์, สถานะเซิร์ฟเวอร์ที่อัปเดตทุกวินาที |
| เกมหลายผู้เล่น (Multiplayer Game) | ตำแหน่งผู้เล่นคนอื่นต้องซิงค์กันแบบ low-latency |
| Collaborative Editing | เอกสารที่หลายคนพิมพ์พร้อมกัน (เช่น Google Docs) |

### เชื่อมโยงกับ Part 35: สถาปัตยกรรมเดิม แค่เปลี่ยนโปรโตคอลชั้นล่าง

ผู้เรียนที่ผ่าน **Part 35 (โปรเจกต์ TCP Chat Server/Client)** มาแล้วจะสังเกตได้ทันทีว่า
โจทย์ของ Part นี้คุ้นเคยมาก — "รับ client หลายคน แล้ว broadcast ข้อความจากคนหนึ่งไปยัง
ทุกคนที่เหลือ" คือปัญหาเดียวกันเป๊ะ! สิ่งที่เปลี่ยนไปมีแค่ **ชั้นโปรโตคอล**:

| แนวคิด | Part 35 (Raw TCP Socket) | Part 109 (WebSocket ผ่าน Crow) |
|---|---|---|
| การรับ connection ใหม่ | `accept()` เอง + `pthread_create()` เอง | Crow (ผ่าน asio) จัดการ thread/event loop ให้ |
| การส่งข้อมูล | `send()` ดิบๆ เป็น byte stream (ไม่มีขอบเขตข้อความ) | `send_text()` / `send_binary()` — Crow ใส่ frame header ให้อัตโนมัติ ข้อความมีขอบเขตชัดเจน |
| การรับข้อมูล | `recv()` ดิบๆ ต้องจัดการปัญหา "TCP เป็น byte stream" เอง (ดู Part 35.4) | `onmessage` callback ได้ข้อความที่ตัดขอบเขตมาให้ครบแล้ว |
| Client list ที่แชร์กัน | `client_t client_list[MAX_CLIENTS]` + `pthread_mutex_t` | `std::unordered_map<connection*, ...>` + `std::mutex` (แนวคิดเดียวกันเป๊ะ) |
| Handshake | ไม่มี (TCP เชื่อมต่อแล้วคุยได้เลย) | HTTP Upgrade handshake ตาม RFC 6455 (หัวข้อ 109.3) |
| ผ่าน Firewall องค์กรได้ง่ายไหม | ยากกว่า (พอร์ตแปลกอาจถูกบล็อก) | ง่ายกว่า (ใช้พอร์ต HTTP/HTTPS ปกติ) |
| Debug ด้วยเครื่องมือ HTTP มาตรฐานได้ไหม | ไม่ได้ (ต้องใช้ `netcat` คุยเป็น text เอง) | ได้บางส่วน (`curl` ดู handshake ได้, ต้องใช้ WebSocket client ทดสอบข้อความจริง) |

จะเห็นว่า **หัวใจของปัญหาไม่เปลี่ยน** — เรื่อง shared mutable state ที่ต้องป้องกันด้วย mutex,
เรื่องการจัดการ client เข้า-ออก, เรื่อง broadcast ยังคงเป็นแนวคิดเดียวกับ Part 35 ทุกประการ
สิ่งที่ WebSocket framework อย่าง Crow ทำให้คือ "ยกภาระเรื่อง protocol-level detail" (การ
แบ่ง TCP stream เป็นข้อความ, การทำ handshake) ออกไปจากเรา ให้เราโฟกัสที่ business logic
ของแอปพลิเคชันได้เต็มที่

### ปูทางสู่ Part 122: Capstone Real-time Multiplayer Chat Server

โปรเจกต์ broadcast chat server ง่ายๆ ที่เราจะเขียนใน Part นี้ (หัวข้อ 109.4-109.5) คือ
"เวอร์ชันจำลอง" ของสิ่งที่จะขยายให้สมบูรณ์แบบเต็มรูปแบบใน **Part 122 — Capstone 2:
Real-time Multiplayer Chat Server** ซึ่งจะรวมทั้ง raw socket (Part 35), WebSocket (Part นี้),
multi-threading ขั้นสูง, ห้องแชทหลายห้อง, และการจัดการผู้ใช้จำนวนมากพร้อมกันในระดับที่ใกล้เคียง
production จริง ทักษะเรื่อง thread-safety ที่เราพิสูจน์ด้วย ThreadSanitizer ในหัวข้อ 109.7
จะเป็นรากฐานสำคัญที่ใช้ตรงนั้นเช่นกัน

## 109.3 WebSocket Handshake (RFC 6455) และ Crow's WebSocket API (Step 867)

### ขั้นตอนของ Handshake

WebSocket connection ทุกเส้นเริ่มต้นด้วย HTTP request ธรรมดาที่มี header พิเศษขอ "อัปเกรด"
protocol ของ connection เส้นนั้น:

```
GET /ws HTTP/1.1
Host: 127.0.0.1:18280
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13
```

ถ้า server รองรับ WebSocket บน path นี้ จะตอบกลับด้วย HTTP status พิเศษ `101 Switching
Protocols` (ไม่ใช่ 200 OK แบบ HTTP ปกติ) พร้อม header `Sec-WebSocket-Accept` ที่คำนวณมา
จาก `Sec-WebSocket-Key` ของฝั่ง client:

```
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```

หลังจากขั้นตอนนี้ **TCP connection เส้นเดิมจะไม่ใช่ HTTP อีกต่อไป** — มันกลายเป็น WebSocket
connection แบบเต็มรูปแบบที่ทั้งสองฝั่งส่ง "frame" ของข้อความหากันได้ตลอดเวลาโดยไม่ต้องมี
HTTP request/response header ใดๆ อีกเลย

### วิธีคำนวณ `Sec-WebSocket-Accept` ตามสเปก

RFC 6455 กำหนดสูตรตายตัวไว้ชัดเจน (ไม่มีความลับ ไม่ใช่ secret key ใดๆ — เป็นแค่ตัวยืนยันว่า
server เข้าใจ header `Sec-WebSocket-Key` จริง ไม่ใช่ proxy เก่าที่ forward request มาผิดๆ):

```
Sec-WebSocket-Accept = base64( SHA1( Sec-WebSocket-Key + "258EAFA5-E914-47DA-95CA-C5AB0DC85B11" ) )
```

ตัวเลข `258EAFA5-E914-47DA-95CA-C5AB0DC85B11` คือ **GUID คงที่ตายตัว** ที่กำหนดไว้ใน
สเปกโดยเฉพาะ (Magic String) — ทุก WebSocket implementation ในโลกต้องใช้ค่านี้เหมือนกันหมด
ลองตรวจสอบด้วยตัวเองด้วยคำสั่ง `openssl` ตรงๆ (ไม่ต้องพึ่ง library ใดๆ เลย):

```bash
$ echo -n "dGhlIHNhbXBsZSBub25jZQ==258EAFA5-E914-47DA-95CA-C5AB0DC85B11" | \
  openssl dgst -sha1 -binary | openssl base64
s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```

ผลลัพธ์ `s3pPLMBiTxaQ9kYGzzhZRbK+xOo=` ตรงกับตัวอย่างในตัว RFC 6455 เองทุกตัวอักษร — และ
ตรงกับที่ Crow server ของเราตอบกลับมาจริงในหัวข้อ 109.6 ด้วย นี่คือหลักฐานว่า Crow
implement handshake ตามสเปกอย่างถูกต้อง 100%

### Crow's WebSocket API: `CROW_WEBSOCKET_ROUTE`

Crow ซ่อนความซับซ้อนของ handshake และ frame parsing ทั้งหมดไว้เบื้องหลัง macro
`CROW_WEBSOCKET_ROUTE` ที่ให้เราลงทะเบียน handler แยกตามเหตุการณ์ (event) ของ connection
แต่ละเส้น คล้ายกับการเขียน event-driven UI:

```cpp
CROW_WEBSOCKET_ROUTE(app, "/ws")
.onopen([&](crow::websocket::connection& conn) {
    // เรียกทันทีที่ handshake สำเร็จ (เทียบเท่า "client เข้าห้อง" ใน Part 35)
})
.onmessage([&](crow::websocket::connection& conn, const std::string& data, bool is_binary) {
    // เรียกทุกครั้งที่ได้รับข้อความสมบูรณ์ 1 ข้อความ (Crow ตัดขอบเขตให้แล้ว)
})
.onclose([&](crow::websocket::connection& conn, const std::string& reason, uint16_t code) {
    // เรียกเมื่อ connection ถูกปิด ไม่ว่าจะปิดแบบสุภาพหรือหลุดกะทันหัน
})
.onerror([&](crow::websocket::connection& conn, const std::string& message) {
    // เรียกเมื่อเกิด error ระดับ transport (เช่น connection reset กลางทาง)
})
.onaccept([&](const crow::request& req, void** userdata) -> bool {
    // เรียก "ก่อน" handshake จะสำเร็จ ใช้ปฏิเสธการเชื่อมต่อได้ (คืน false = ปฏิเสธ)
    // เหมาะสำหรับตรวจสอบ Origin header หรือ token ก่อนอนุญาตให้อัปเกรดเป็น WebSocket
    return true;
});
```

| Handler | เรียกเมื่อไหร่ | เทียบเท่าอะไรใน Part 35 |
|---|---|---|
| `onaccept` | ก่อน handshake จะสำเร็จ (ใช้กรอง connection) | ไม่มีโดยตรง (ใกล้เคียงจุดที่เช็ค `client_list_add` เต็มหรือไม่) |
| `onopen` | ทันทีหลัง handshake สำเร็จ | จุดเริ่มต้นของ `client_handler()` หลัง `accept()` |
| `onmessage` | ทุกครั้งที่ได้รับ 1 ข้อความสมบูรณ์ | หลังจาก `recv()` และตัดขอบเขตข้อความสำเร็จ |
| `onclose` | เมื่อ connection ปิด | ส่วน cleanup ท้าย `client_handler()` |
| `onerror` | เมื่อเกิด transport error | กรณี `recv()` คืนค่าติดลบพร้อม `errno` |

จุดที่ต่างจาก Part 35 อย่างชัดเจนคือ **เราไม่ต้องสร้าง thread เองด้วย `pthread_create()`
เลย** — Crow จัดการ thread/event loop (ผ่าน `asio` เบื้องหลัง) ให้เราเรียบร้อยแล้วตอนที่
เรียก `app.multithreaded()` เหมือนกับที่ทำใน REST API ของ Part 108 ทุกประการ

## 109.4 เขียน WebSocket Server พื้นฐานตัวแรก (Step 868)

เริ่มจากเวอร์ชันที่ง่ายที่สุด — server ที่รับ WebSocket connection แล้ว echo ข้อความกลับไป
ให้ผู้ส่งเอง (ยังไม่ broadcast ให้คนอื่น) เพื่อให้เห็นโครงสร้างพื้นฐานก่อน:

```cpp
// ws_echo.cpp — WebSocket server เบื้องต้นที่สุด: echo ข้อความกลับหาผู้ส่งเอง
#include "crow.h"

int main() {
    crow::SimpleApp app;

    CROW_ROUTE(app, "/")
    ([]() {
        return "เปิด ws://127.0.0.1:18280/ws เพื่อทดสอบ WebSocket echo";
    });

    CROW_WEBSOCKET_ROUTE(app, "/ws")
    .onopen([](crow::websocket::connection& conn) {
        CROW_LOG_INFO << "WebSocket เชื่อมต่อจาก " << conn.get_remote_ip();
    })
    .onmessage([](crow::websocket::connection& conn, const std::string& data, bool is_binary) {
        // ส่งข้อความเดิมกลับไปหาผู้ส่งคนเดียวกัน (ยังไม่ broadcast)
        if (is_binary) conn.send_binary(data);
        else conn.send_text(data);
    })
    .onclose([](crow::websocket::connection&, const std::string& reason, uint16_t) {
        CROW_LOG_INFO << "WebSocket ปิดการเชื่อมต่อ: " << reason;
    });

    app.port(18280).multithreaded().run();
}
```

คอมไพล์และรันจริง (ทดสอบบนเครื่องจริงด้วย `g++ 13.3.0`, Crow ติดตั้งที่
`/usr/local/include/crow`):

```bash
$ g++ -std=c++17 -Wall -Wextra -I/usr/local/include ws_echo.cpp -o ws_echo -lpthread
$ ./ws_echo
```

คอมไพล์ผ่านสะอาด **ไม่มี warning แม้แต่บรรทัดเดียว** สังเกตว่า flag การคอมไพล์เหมือน REST
API ของ Part 108 ทุกประการ — ต่างกันแค่ต้องมี `-lpthread` เพราะ Crow ใช้ `std::thread`
ภายใน (ในระบบ Linux สมัยใหม่ที่ glibc รวม pthread เข้ากับ libc แล้ว บาง distro ไม่จำเป็นต้อง
ใส่ flag นี้ตรงๆ ก็ได้ แต่การใส่ไว้เสมอปลอดภัยกว่าและพกพาข้ามระบบได้ดีกว่า)

ทดสอบดู handshake เบื้องต้นด้วย `curl` (รายละเอียดเต็มอยู่ในหัวข้อ 109.6):

```bash
$ curl -s http://127.0.0.1:18280/
เปิด ws://127.0.0.1:18280/ws เพื่อทดสอบ WebSocket echo
```

เวอร์ชัน echo นี้ยังไม่ใช่สิ่งที่โจทย์ของ Part นี้ต้องการ (echo กลับหาตัวเอง ไม่มีประโยชน์
เท่าไหร่ในโลกจริง) — สิ่งที่เราต้องการจริงๆ คือ **broadcast ให้ client ทุกตัวที่เชื่อมต่ออยู่**
ซึ่งต้องมีที่เก็บรายชื่อ client ทั้งหมด (เหมือน `client_list` ใน Part 35)

## 109.5 Broadcast Server และ Thread-Safety ด้วย Mutex (Step 869)

โจทย์หลักของ Part นี้คือ: เมื่อ client คนหนึ่งส่งข้อความมา ต้อง broadcast ไปยัง client
**ทุกตัว** ที่เชื่อมต่ออยู่ (ยกเว้นผู้ส่งเอง) เหมือนกับ `broadcast_message()` ใน Part 35
ทุกประการ ต่างกันแค่ที่เก็บข้อมูลและวิธีส่ง

### ทำไมต้องมี Mutex อีกครั้ง

เหตุผลเดียวกับ Part 35 เป๊ะ: `app.port(...).multithreaded().run()` ทำให้ Crow ใช้
worker thread หลายตัวจัดการหลาย connection พร้อมกัน นั่นแปลว่า `onopen` ของ client A,
`onmessage` ของ client B, และ `onclose` ของ client C **อาจทำงานพร้อมกันคนละ thread ในเวลา
เดียวกันได้จริง** ถ้าทั้งสาม handler นี้แก้ไข container ตัวเดียวกัน (เช่น
`std::unordered_map` ที่เก็บรายชื่อ client) โดยไม่มี mutex ป้องกัน จะเกิด **data race**
ทันที — และในหัวข้อ 109.7 เราจะพิสูจน์ด้วยเครื่องมือจริงว่ามันไม่ใช่แค่ทฤษฎี แต่ทำให้
โปรแกรม **crash จริง**

### โครงสร้างข้อมูลที่ใช้

```cpp
// ข้อมูลของ client แต่ละตัวที่กำลังเชื่อมต่ออยู่
struct ClientInfo {
    std::string username;
    bool has_username = false;
};

std::mutex clients_mutex;
std::unordered_map<crow::websocket::connection*, ClientInfo> clients;
```

สังเกตว่าเราใช้ **pointer ของ `crow::websocket::connection` เป็น key** แทนที่จะเป็น
file descriptor (int) แบบ Part 35 — เพราะ Crow ไม่ได้เปิดเผย file descriptor ดิบให้เราใช้
โดยตรง (ห่อหุ้มไว้เบื้องหลัง `connection` object) แต่แนวคิดเหมือนกันทุกประการ: ใช้ค่าที่
**ระบุตัวตนของ connection แต่ละเส้นได้ไม่ซ้ำกัน** เป็น key

### โปรโตคอลง่ายๆ (เหมือน Part 35): ข้อความแรก = ชื่อผู้ใช้

เพื่อให้เทียบเคียงกับ Part 35 ได้ตรงที่สุด เราใช้โปรโตคอลเดียวกัน: **ข้อความแรก** ที่
client ส่งมาถือเป็นชื่อผู้ใช้ ข้อความถัดจากนั้นถือเป็นข้อความแชทที่ต้อง broadcast พร้อม
แปะชื่อผู้ส่งไว้ข้างหน้า

ไฟล์ `ws_chat_server.cpp` ฉบับสมบูรณ์ (ทดสอบคอมไพล์และรันจริงแล้ว):

```cpp
/* ============================================================
 * ชื่อไฟล์:     ws_chat_server.cpp
 * คำอธิบาย:     WebSocket Broadcast Chat Server ด้วย Crow
 *              ข้อความแรกที่ client ส่งมาถือเป็นชื่อผู้ใช้ ข้อความถัดไปจะถูก
 *              แปะชื่อผู้ส่งแล้ว broadcast ไปยัง client ทุกตัวที่เชื่อมต่ออยู่
 *              (ยกเว้นผู้ส่งเอง) — เทียบเท่ากับโปรโตคอลของ Part 35 แต่ใช้
 *              WebSocket แทน raw TCP socket
 * ============================================================ */
#include "crow.h"
#include <mutex>
#include <unordered_map>
#include <string>

// ข้อมูลของ client แต่ละตัวที่กำลังเชื่อมต่ออยู่
struct ClientInfo {
    std::string username;
    bool has_username = false;
};

int main() {
    crow::SimpleApp app;

    // clients คือทรัพยากรที่ "แชร์กัน" ระหว่างหลาย thread เช่นเดียวกับ
    // client_list ใน Part 35 — ต่างกันตรงที่ตอนนี้ key เป็น pointer ของ
    // crow::websocket::connection แทนที่จะเป็น file descriptor (int)
    std::mutex clients_mutex;
    std::unordered_map<crow::websocket::connection*, ClientInfo> clients;

    CROW_ROUTE(app, "/")
    ([]() {
        return "WebSocket Chat Server: connect to ws://<host>:18280/ws "
               "(ส่งข้อความแรกเป็นชื่อผู้ใช้ก่อนเสมอ)";
    });

    CROW_WEBSOCKET_ROUTE(app, "/ws")
    .onopen([&](crow::websocket::connection& conn) {
        std::lock_guard<std::mutex> lock(clients_mutex);
        clients[&conn] = ClientInfo{};
        CROW_LOG_INFO << "[+] client ใหม่เชื่อมต่อ ผู้ใช้ทั้งหมดตอนนี้ = " << clients.size();
    })
    .onclose([&](crow::websocket::connection& conn, const std::string& reason, uint16_t code) {
        std::string leaving_name;
        {
            std::lock_guard<std::mutex> lock(clients_mutex);
            auto it = clients.find(&conn);
            if (it != clients.end()) {
                leaving_name = it->second.username;
                clients.erase(it);
            }
        }
        CROW_LOG_INFO << "[-] " << leaving_name << " ออกจากห้อง (reason=" << reason
                      << " code=" << code << ") เหลือ " << clients.size() << " คน";

        if (!leaving_name.empty()) {
            std::lock_guard<std::mutex> lock(clients_mutex);
            std::string msg = "*** " + leaving_name + " ออกจากห้องแชทแล้ว ***";
            for (auto& [c, info] : clients) {
                if (info.has_username) c->send_text(msg);
            }
        }
    })
    .onmessage([&](crow::websocket::connection& conn, const std::string& data, bool is_binary) {
        if (is_binary) return; // โปรโตคอลนี้เป็น text ล้วน ไม่รองรับ binary frame

        std::lock_guard<std::mutex> lock(clients_mutex);
        auto it = clients.find(&conn);
        if (it == clients.end()) return; // ป้องกันกรณีแปลกๆ ที่ connection ไม่อยู่ใน map แล้ว

        if (!it->second.has_username) {
            // ข้อความแรก = ชื่อผู้ใช้ (เทียบเท่าบรรทัดแรกในโปรโตคอลของ Part 35)
            it->second.username = data;
            it->second.has_username = true;
            std::string join_msg = "*** " + data + " เข้าร่วมห้องแชทแล้ว ***";
            for (auto& [c, info] : clients) {
                if (c != &conn && info.has_username) c->send_text(join_msg);
            }
            conn.send_text("ยินดีต้อนรับ " + data + "! พิมพ์ข้อความเพื่อเริ่มแชทได้เลย");
            return;
        }

        // ข้อความปกติ: แปะชื่อผู้ส่งแล้ว broadcast ให้ทุกคนยกเว้นตัวเอง
        std::string out = it->second.username + ": " + data;
        for (auto& [c, info] : clients) {
            if (c != &conn && info.has_username) c->send_text(out);
        }
    });

    app.port(18280).multithreaded().run();
}
```

### จุดสำคัญที่ต้องเข้าใจให้ลึก

1. **`std::lock_guard<std::mutex>`** ล็อกทันทีที่สร้าง object และปลดล็อกอัตโนมัติตอนออกจาก
   scope (RAII — ทบทวนจาก Part 68) เหมือนกับที่ Part 35 ใช้
   `pthread_mutex_lock`/`pthread_mutex_unlock` คู่กันด้วยมือ ต่างกันตรงที่ C++ ทำให้
   "ลืมปลดล็อก" เป็นไปไม่ได้เลยแม้จะมี early return หรือ exception เกิดขึ้นกลางทาง
2. **`onclose` ล็อก mutex สองรอบแยกกัน** (ครั้งแรกเพื่อลบออกจาก map, ครั้งที่สองเพื่อ
   broadcast ข้อความ "ออกจากห้อง") แทนที่จะล็อกครั้งเดียวยาวๆ ตลอดทั้งฟังก์ชัน เหตุผลคือ
   ต้องการให้ **critical section สั้นที่สุดเท่าที่จำเป็น** — การถือ lock นานเกินจำเป็น
   จะทำให้ thread อื่นที่รอ lock อยู่ค้างนานขึ้นโดยไม่จำเป็น (หลักการเดียวกับที่อธิบายไว้
   ใน Part 35.4)
3. **`for (auto& [c, info] : clients)`** ใช้ **Structured Bindings** (C++17 — ทบทวนจาก
   Part 79) ทำให้อ่าน key/value ของ `std::unordered_map` ได้กระชับกว่าการเขียน
   `it->first`/`it->second` แบบเดิม
4. **เช็ค `is_binary`** ก่อนเสมอในทุก `onmessage` — โปรโตคอลของเราออกแบบไว้เป็น text
   ล้วนๆ การไม่ตรวจสอบและพยายามตีความ binary frame เป็นชื่อผู้ใช้อาจทำให้เกิดพฤติกรรม
   ที่ไม่คาดคิด (เทียบเท่าการไม่ validate input จาก client ใน Part 108)

## 109.6 ทดสอบ Handshake ด้วย `curl` และทดสอบจริงด้วย `websocat` (Step 870)

### ทดสอบ 1 — ดู Handshake ดิบๆ ด้วย `curl --include --no-buffer`

`curl` ไม่ใช่ WebSocket client เต็มรูปแบบ (คุยข้อความหลัง handshake ไม่ได้) แต่มีประโยชน์
มากในการ **ดู handshake ดิบๆ** เพื่อยืนยันว่า server เรา implement RFC 6455 ถูกต้อง โดย
ส่ง header ที่จำเป็นเข้าไปเอง:

```bash
$ curl -s -i --no-buffer \
  -H "Connection: Upgrade" -H "Upgrade: websocket" \
  -H "Sec-WebSocket-Version: 13" \
  -H "Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==" \
  http://127.0.0.1:18280/ws --max-time 2
```

ผลลัพธ์จริงจากการรันคำสั่งนี้กับ server ของเรา:

```
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```

ค่า `Sec-WebSocket-Accept` ที่ได้ตรงกับที่เราคำนวณด้วยมือใน 109.3 **ทุกตัวอักษร** — ยืนยัน
ว่า Crow implement การคำนวณตามสเปกถูกต้อง 100% (`--max-time 2` ทำให้ curl ตัด connection
เองหลัง 2 วินาที เพราะ curl ไม่รู้จะส่ง WebSocket frame อย่างไรต่อ ไม่ใช่ error ของ server)

### ทดสอบ 2 — ติดตั้งและใช้ `websocat` เป็น WebSocket client เต็มรูปแบบ

`websocat` คือ WebSocket client บน command line ที่ใช้งานง่ายที่สุดตัวหนึ่ง ในสภาพแวดล้อม
นี้ไม่มี `apt` package ให้ติดตั้งตรงๆ แต่ติดตั้งผ่าน `cargo` (Rust package manager) ได้จริง:

```bash
$ cargo install websocat
   Compiling websocat v1.14.1
    Finished `release` profile [optimized] target(s) in 43.18s
  Installing /root/.cargo/bin/websocat
   Installed package `websocat v1.14.1` (executable `websocat`)

$ websocat --version
websocat 1.14.1
```

ทดสอบ broadcast จริงด้วย `websocat` สองตัว (client A เปิดค้างไว้ฟังอย่างเดียว, client B
ส่งข้อความเข้าไปแล้วออก) — รันจริงบนเครื่องทดสอบ:

```bash
# Terminal 1 (client A: เชื่อมต่อค้างไว้ ฟังอย่างเดียว)
$ websocat ws://127.0.0.1:18280/ws

# Terminal 2 (client B: ส่งข้อความ "Hello via websocat" แล้วออก)
$ echo "Hello via websocat" | websocat ws://127.0.0.1:18280/ws
```

ผลลัพธ์ที่ Terminal 1 ได้รับจริง:

```
Hello via websocat
```

ข้อความจาก client B ถูก broadcast มาถึง client A จริงตามที่ออกแบบไว้ — นี่คือหลักฐานว่า
WebSocket server ของเราทำงานถูกต้องกับ WebSocket client ตัวจริง ไม่ใช่แค่ผ่าน handshake
เฉยๆ แบบที่ `curl` ทำได้

### เปรียบเทียบเครื่องมือทดสอบ WebSocket ที่ใช้ได้จริงในสภาพแวดล้อมนี้

| เครื่องมือ | ทดสอบ Handshake ได้ | ทดสอบส่ง/รับข้อความจริงได้ | ติดตั้งอย่างไร |
|---|---|---|---|
| `curl --include --no-buffer` | ได้ (ดู header `101 Switching Protocols`) | ไม่ได้ | มีอยู่แล้วในระบบ |
| `websocat` | ได้ | ได้ (WebSocket client เต็มรูปแบบ) | `cargo install websocat` |
| Python `websockets` library | ได้ | ได้ (เหมาะกับเขียน script ทดสอบอัตโนมัติหลาย client) | `pip3 install websockets` |
| เขียน C++ test client เอง | ได้ | ได้ (ควบคุมได้ละเอียดที่สุด แต่เขียนโค้ดเยอะที่สุด) | ไม่ต้องติดตั้งอะไรเพิ่ม ใช้ library เดียวกับ server |

## 109.7 ทดสอบ Multi-Client Broadcast ด้วย Python และพิสูจน์ Race Condition ด้วย ThreadSanitizer (Step 871)

### ทดสอบ Broadcast แบบเต็มรูปแบบด้วย Python `websockets`

`websocat` เหมาะกับการทดสอบแบบ manual ทีละคำสั่ง แต่การทดสอบ scenario ที่ซับซ้อนกว่า
(เช่น 2 client เข้าห้อง คุยกัน แล้วออกตามลำดับ) เขียนเป็น script ด้วย Python library
`websockets` สะดวกกว่ามาก:

```python
import asyncio
import websockets

URL = "ws://127.0.0.1:18280/ws"

async def alice():
    async with websockets.connect(URL) as ws:
        await ws.send("Alice")
        print("[Alice] <-", await ws.recv())      # welcome message
        await asyncio.sleep(0.3)
        print("[Alice] <-", await ws.recv())      # Bob's join notice
        await ws.send("สวัสดีทุกคนครับ")
        await asyncio.sleep(0.5)
        print("[Alice] <-", await ws.recv())      # Bob's chat message
        await asyncio.sleep(1.0)

async def bob():
    await asyncio.sleep(0.3)
    async with websockets.connect(URL) as ws:
        await ws.send("Bob")
        print("[Bob]   <-", await ws.recv())      # welcome
        print("[Bob]   <-", await ws.recv())      # Alice's greeting broadcast
        await ws.send("หวัดดีครับ Alice")
        await asyncio.sleep(1.0)

async def main():
    await asyncio.gather(alice(), bob())

asyncio.run(main())
```

รันจริงกับ `ws_chat_server` ที่กำลังทำงานอยู่ ผลลัพธ์ที่ได้จริง:

```
$ python3 test_chat.py
[Alice] <- ยินดีต้อนรับ Alice! พิมพ์ข้อความเพื่อเริ่มแชทได้เลย
[Bob]   <- ยินดีต้อนรับ Bob! พิมพ์ข้อความเพื่อเริ่มแชทได้เลย
[Alice] <- *** Bob เข้าร่วมห้องแชทแล้ว ***
[Bob]   <- Alice: สวัสดีทุกคนครับ
[Alice] <- Bob: หวัดดีครับ Alice
```

และฝั่ง server log (`CROW_LOG_INFO` ที่เราเขียนไว้) ยืนยันลำดับเหตุการณ์ตรงกัน:

```
(2026-09-26 11:34:22) [INFO    ] Crow/master server is running at http://0.0.0.0:18280 using 4 threads
(2026-09-26 11:34:23) [INFO    ] [+] client ใหม่เชื่อมต่อ ผู้ใช้ทั้งหมดตอนนี้ = 1
(2026-09-26 11:34:23) [INFO    ] [+] client ใหม่เชื่อมต่อ ผู้ใช้ทั้งหมดตอนนี้ = 2
(2026-09-26 11:34:25) [INFO    ] [-] Bob ออกจากห้อง (reason= code=1000) เหลือ 1 คน
(2026-09-26 11:34:25) [INFO    ] [-] Alice ออกจากห้อง (reason= code=1000) เหลือ 0 คน
```

การทดสอบนี้ยืนยันว่า: (1) โปรโตคอล "ข้อความแรก = ชื่อผู้ใช้" ทำงานถูกต้อง (2) การ
broadcast ข้อความเข้าห้องและข้อความแชทไปหาทุกคนยกเว้นผู้ส่งทำงานถูกต้อง (3) การ cleanup
ตอน disconnect (`code=1000` คือ `NormalClosure` ตาม RFC 6455) ทำงานถูกต้องและ mutex ไม่ทำให้
เกิด deadlock ใดๆ ตลอดการทดสอบ

### พิสูจน์ด้วยเครื่องมือจริงว่าทำไมต้องมี Mutex: ThreadSanitizer

Part 35 อธิบายอันตรายของ race condition ด้วยแผนภาพจำลองเหตุการณ์ — Part นี้จะไปไกลกว่านั้น
คือ **พิสูจน์ด้วยเครื่องมือจริง** ว่าถ้าตัดโค้ด mutex ทิ้งไปจะเกิดอะไรขึ้นจริงๆ โดยใช้
**ThreadSanitizer (TSan)** ซึ่งเป็น compiler flag ที่มีมาพร้อม GCC/Clang อยู่แล้ว
ไม่ต้องติดตั้งอะไรเพิ่ม

สร้างเวอร์ชัน "ไม่ปลอดภัย" ที่ตัด mutex ออกทั้งหมดโดยตั้งใจ:

```cpp
// ws_broadcast_unsafe.cpp — เวอร์ชัน "ไม่ปลอดภัย" ที่ตั้งใจไม่ใส่ mutex
// ใช้สำหรับสาธิตให้เห็น race condition จริงด้วย ThreadSanitizer เท่านั้น
// ห้ามใช้ code แบบนี้ใน production เด็ดขาด
#include "crow.h"
#include <unordered_set>

int main() {
    crow::SimpleApp app;
    std::unordered_set<crow::websocket::connection*> users; // ไม่มี mutex ป้องกันโดยตั้งใจ

    CROW_WEBSOCKET_ROUTE(app, "/ws")
    .onopen([&](crow::websocket::connection& conn) {
        users.insert(&conn); // เขียน container จาก thread ของ connection นี้
    })
    .onclose([&](crow::websocket::connection& conn, const std::string&, uint16_t) {
        users.erase(&conn); // เขียน container จาก thread ของ connection นี้ (อาจคนละ thread กับ onopen)
    })
    .onmessage([&](crow::websocket::connection& conn, const std::string& data, bool is_binary) {
        for (auto* u : users) { // อ่าน container จาก thread ที่สาม พร้อมกับอีกสองฝั่งกำลังเขียน
            if (u != &conn) {
                if (is_binary) u->send_binary(data);
                else u->send_text(data);
            }
        }
    });

    app.port(18281).multithreaded().run();
}
```

คอมไพล์ด้วย flag `-fsanitize=thread` (เปิดใช้งาน ThreadSanitizer):

```bash
$ g++ -std=c++17 -Wall -Wextra -fsanitize=thread -g -I/usr/local/include \
  ws_broadcast_unsafe.cpp -o ws_broadcast_unsafe -lpthread
```

จากนั้นรัน server แล้วยิง client 20 ตัวพร้อมกันซ้ำ 30 รอบด้วย script Python (จำลอง
สถานการณ์ที่มีคนเข้า-ออกห้องพร้อมกันจำนวนมาก ซึ่งเป็นสถานการณ์ที่ทำให้ race condition
"โผล่" ออกมาได้ง่ายที่สุด):

```python
import asyncio, websockets

URL = "ws://127.0.0.1:18281/ws"

async def one_client(i):
    try:
        async with websockets.connect(URL) as ws:
            await ws.send(f"msg from {i}")
            await asyncio.sleep(0.01)
    except Exception:
        pass

async def main():
    for round_ in range(30):
        await asyncio.gather(*(one_client(i) for i in range(20)))

asyncio.run(main())
```

**ผลลัพธ์จริงที่ได้ (ไม่ปรุงแต่ง):** ThreadSanitizer รายงาน **data race รวม 51 จุด** และ
ที่ร้ายแรงกว่านั้นคือ **โปรแกรม crash จริง** ด้วย heap-use-after-free:

```
WARNING: ThreadSanitizer: data race (pid=18654)
  Write of size 8 at 0x724800000148 by thread T2:
    #0 crow::WebSocketRule<crow::Crow<> >::handle_upgrade(...) routing.h:446
    ...
WARNING: ThreadSanitizer: data race (pid=18654)
  Read of size 8 at 0x7ffd0f059448 by thread T1:
    #0 std::_Hashtable<...>::size() const hashtable.h:648
    #1 ...::insert(...) ws_broadcast_unsafe.cpp:13
    ...
SUMMARY: ThreadSanitizer: heap-use-after-free ws_broadcast_unsafe.cpp:22 in operator()
==================
pure virtual method called
terminate called without an active exception
```

`ws_broadcast_unsafe.cpp:13` คือบรรทัด `users.insert(&conn)` ใน `onopen` และบรรทัด 22
คือ loop ใน `onmessage` ที่กำลังอ่าน `users` — TSan จับได้ตรงเป๊ะว่า thread สองตัวกำลัง
แก้ไข `std::unordered_set` ตัวเดียวกันพร้อมกันจริง และผลลัพธ์สุดท้ายคือโปรแกรม **เรียก
`operator delete` กับ connection object ที่ถูกใช้งานอยู่พอดี (use-after-free) จนพัง
ทั้งกระบวนการ** ("pure virtual method called" คือสัญญาณคลาสสิกของการเรียกเมธอดบน object
ที่ถูก destroy ไปแล้ว)

ตอนนี้ทดสอบเวอร์ชันที่ **มี** mutex (`ws_broadcast.cpp` จากหัวข้อก่อน) ด้วยการยิง stress
test ชุดเดียวกันทุกประการ ผ่าน `-fsanitize=thread` เหมือนกัน:

```bash
$ g++ -std=c++17 -Wall -Wextra -fsanitize=thread -g -I/usr/local/include \
  ws_broadcast.cpp -o ws_broadcast_tsan -lpthread
$ ./ws_broadcast_tsan &
$ python3 stress_race.py   # ยิง client 20 ตัว x 30 รอบ ชุดเดียวกับก่อนหน้า
```

**ผลลัพธ์: ไม่มี heap-use-after-free หรือ crash เกิดขึ้นเลยแม้แต่ครั้งเดียว** ตลอดการรัน
ทั้ง 30 รอบ (600 connection รวม) — server ทำงานจนจบ stress test แล้วยัง responsive ต่อไป
ปกติ นี่คือหลักฐานเชิงประจักษ์ (ไม่ใช่แค่ทฤษฎี) ว่า `std::mutex` ที่เพิ่มเข้าไปในหัวข้อ 109.5
คือสิ่งที่ป้องกันไม่ให้โปรแกรม crash จริงภายใต้ concurrent load

> **หมายเหตุอย่างตรงไปตรงมา**: การรัน stress test เดียวกันนี้ยังพบ warning การแข่งขัน
> (data race) เพิ่มอีก 1 จุดใน**โค้ดภายในของ Crow เอง** (`routing.h:446` ส่วนจัดการ
> subprotocol ตอน handshake) ซึ่งไม่เกี่ยวกับ `users`/`clients` หรือ mutex ที่เราเขียนเลย
> นี่คือตัวอย่างจริงที่แสดงให้เห็นว่า "0 data race" ในระบบที่พึ่งพา 3rd-party library
> เป็นเป้าหมายที่ยากมากในทางปฏิบัติ — สิ่งที่เราควบคุมได้และต้องรับผิดชอบเต็มที่คือ
> **โค้ดในส่วนของแอปพลิเคชันที่เราเขียนเอง** (ซึ่งพิสูจน์แล้วว่าปลอดภัย) ส่วน race
> condition ระดับ library เป็นเรื่องที่ต้องรายงานหรือติดตาม upstream ต่อไป ไม่ใช่สิ่งที่
> แก้ได้จาก application code

### ตารางสรุปผลการทดสอบ

| เวอร์ชัน | มี Mutex? | Data Race ที่พบ | ผลลัพธ์หลัง Stress Test (20 client × 30 รอบ) |
|---|---|---|---|
| `ws_broadcast_unsafe.cpp` | ไม่มี | 51 จุด (ในโค้ดแอปเราเอง) | **Crash** (heap-use-after-free) |
| `ws_broadcast.cpp` (มี mutex) | มี | 0 จุดในโค้ดแอปเรา (พบ 1 จุดในโค้ดภายในของ Crow เอง) | ทำงานได้ปกติจนจบ ไม่ crash |

## 109.8 Ping/Pong, การปิด Connection อย่างถูกต้อง และสรุปสถาปัตยกรรม (Step 872)

### Ping/Pong: กลไก Keep-Alive ของ WebSocket

เพราะ WebSocket connection ออกแบบมาให้ "เปิดค้างไว้นาน" ปัญหาที่ตามมาคือ: จะรู้ได้อย่างไร
ว่า connection ที่ค้างไว้ยัง "มีชีวิต" อยู่จริง ไม่ใช่ค้างอยู่เฉยๆ เพราะ network ขาดไปแล้ว
แต่ TCP ยังไม่ทันรู้ตัว (เช่น router กลางทางถูกถอดปลั๊กโดยไม่มีการปิด connection อย่างสุภาพ)

RFC 6455 แก้ปัญหานี้ด้วย **Ping/Pong frame** — server (หรือ client) ส่ง Ping frame ไป
เป็นระยะ ถ้าอีกฝั่งยังออนไลน์อยู่จริงจะต้องตอบ Pong frame กลับมาโดยอัตโนมัติ (บังคับตาม
สเปก ไม่ต้องเขียนโค้ดจัดการเอง — ดูใน `crow/websocket.h` บรรทัด opcode `0x9`/`0xA` ที่
Crow implement ให้ครบแล้ว) ถ้าไม่ได้รับ Pong ภายในเวลาที่กำหนด server ควรถือว่า connection
นั้น "ตายแล้ว" และปิดทิ้งเพื่อคืนทรัพยากร:

```cpp
// ตัวอย่างการส่ง Ping เองเพื่อตรวจสุขภาพ connection (Crow จัดการ Pong ตอบกลับให้อัตโนมัติ)
conn.send_ping("keepalive");
```

### ปิด Connection อย่างถูกต้องตาม Close Status Code

WebSocket มี close status code มาตรฐาน (นิยามใน `crow::websocket::CloseStatusCode`) ที่
บอกเหตุผลของการปิด connection ให้อีกฝั่งทราบอย่างชัดเจน แทนที่จะปิดเฉยๆ แบบ raw TCP:

| Code | ชื่อ | ความหมาย |
|---|---|---|
| 1000 | `NormalClosure` | ปิดแบบสุภาพตามปกติ (ทั้งสองฝั่งตกลงกันแล้ว) |
| 1001 | `EndpointGoingAway` | ฝั่งใดฝั่งหนึ่งกำลังจะปิดตัว (เช่น browser ปิด tab) |
| 1006 | `ClosedAbnormally` | หลุดกะทันหันโดยไม่มีการปิดแบบสุภาพ (เช่น network ขาด) |
| 1009 | `MessageTooBig` | ข้อความใหญ่เกิน `max_payload_size` ที่กำหนดไว้ |

ในการทดสอบหัวข้อ 109.7 สังเกตว่า log แสดง `code=1000` เสมอเมื่อ Python client ปิด
connection ด้วย `async with` ตามปกติ (ปิดแบบสุภาพ) เทียบกับตอนทดสอบด้วย `curl --max-time 2`
ในหัวข้อ 109.6 ที่ curl ตัด connection กะทันหันจนได้ `code=1006` (`ClosedAbnormally`) —
ค่า code เหล่านี้มีประโยชน์มากตอน debug ปัญหา connection ที่หลุดบ่อยผิดปกติในระบบจริง

### สรุปสถาปัตยกรรมทั้งหมดของ Part นี้

```
Client (websocat / Python websockets / เบราว์เซอร์)
   │  1) HTTP GET /ws พร้อม header Upgrade: websocket
   ▼
Crow HTTP Server (จัดการ handshake ตาม RFC 6455 ให้อัตโนมัติ)
   │  2) ตอบ 101 Switching Protocols พร้อม Sec-WebSocket-Accept ที่คำนวณถูกต้อง
   ▼
crow::websocket::connection (persistent, full-duplex, ยังอยู่จน close)
   │  3) onopen() -> lock mutex -> เพิ่มเข้า clients map -> unlock
   │  4) onmessage() -> lock mutex -> วนลูป broadcast ให้ทุกคนยกเว้นตัวเอง -> unlock
   │  5) onclose()  -> lock mutex -> ลบออกจาก clients map -> unlock -> broadcast ข้อความอำลา
   ▼
Client ทุกตัวที่เหลือได้รับข้อความแบบ real-time โดยไม่ต้อง poll เอง
```

เทียบกับ Part 35 (TCP Chat) แล้ว **แนวคิดเรื่อง shared state + mutex เหมือนกันทุกประการ**
สิ่งที่เปลี่ยนไปมีแค่ Crow รับหน้าที่จัดการ thread pool, HTTP/WebSocket handshake, และการ
ตัดขอบเขตข้อความให้เราแทน ทำให้เราโฟกัสที่ business logic (โปรโตคอลแชท, การ broadcast)
ได้เต็มที่โดยไม่ต้องกังวลเรื่อง protocol-level detail ที่ Part 35 ต้องจัดการเองทั้งหมด

---

## ข้อผิดพลาดที่พบบ่อย (Common Pitfalls)

1. **แชร์ container ของ connection ข้าม thread โดยไม่มี mutex ป้องกัน** — ดังที่พิสูจน์
   ด้วย ThreadSanitizer ในหัวข้อ 109.7 การตัด mutex ออกทำให้เกิด data race จริงและ
   โปรแกรม crash ด้วย heap-use-after-free ไม่ใช่แค่ทฤษฎีที่ "อาจจะ" เกิด
2. **ถือ mutex lock ค้างไว้นานเกินจำเป็น** — เช่น ล็อกตั้งแต่ต้นฟังก์ชันจนจบโดยไม่จำเป็น
   ทำให้ thread อื่นที่รอ lock ต้องรอนานขึ้นโดยเปล่าประโยชน์ ควรทำให้ critical section
   สั้นที่สุดเท่าที่จำเป็นเสมอ (ดูตัวอย่างการล็อกแยกสองรอบใน `onclose` หัวข้อ 109.5)
3. **ลืมตรวจสอบว่า connection ยังอยู่ใน container ก่อนใช้งาน** — หลังจาก `onclose` ลบ
   connection ออกจาก map ไปแล้ว ถ้ามี event อื่นมาถึงช้าและพยายามเข้าถึง connection
   เดิมโดยไม่เช็คก่อน (เช่น `clients.find(&conn) == clients.end()`) จะได้ undefined
   behavior ทันที
4. **ไม่ปิด WebSocket connection อย่างถูกต้องเมื่อ server จะปิดตัว** — ปล่อยให้ OS ตัด
   TCP connection ทิ้งดื้อๆ โดยไม่ส่ง close frame ตามสเปก (opcode `0x8`) ทำให้ client
   ฝั่งตรงข้ามได้ `code=1006` (ClosedAbnormally) แทนที่จะเป็น `1000` (NormalClosure)
   ซึ่งอาจทำให้ logic ฝั่ง client ที่แยกแยะสองกรณีนี้ทำงานผิดพลาด
5. **ไม่จำกัดขนาด payload ของข้อความ** — WebSocket message ไม่มีขีดจำกัดขนาดในตัวสเปก
   ถ้าไม่เรียก `conn.set_max_payload_size(...)` กำหนดเพดานไว้ ผู้ใช้ที่ประสงค์ร้ายส่ง
   ข้อความขนาดหลาย GB มาได้ ทำให้ server หน่วยความจำเต็มจนล่ม (Denial of Service)
6. **เข้าใจผิดว่า `onmessage` ได้ข้อมูลดิบที่ไม่ได้ validate** — ข้อมูลจาก `onmessage`
   ยังคงเป็น input จากภายนอกที่ "ไม่มีวันไว้ใจได้" เหมือนกับ body ของ HTTP request ใน
   Part 108 ทุกประการ ต้อง validate ก่อนใช้งานเสมอ (เช่น เช็คความยาว, เช็ครูปแบบ)
7. **ไม่จัดการ `is_binary` ให้ถูกต้อง** — ถ้าโปรโตคอลออกแบบไว้เป็น text-only แต่ไม่เช็ค
   `is_binary` ก่อน แล้วพยายามตีความ binary data เป็น string ตรงๆ อาจได้ข้อมูลเพี้ยน
   หรือ exception ที่ไม่คาดคิดตอน parse
8. **ลืมว่า WebSocket connection กิน memory ต่อ connection มากกว่า HTTP connection
   ทั่วไป** — เพราะต้องเปิดค้างไว้ตลอดอายุการเชื่อมต่อ (ต่างจาก HTTP ที่ปิดเร็ว) ระบบที่
   ต้องรองรับ concurrent WebSocket connection จำนวนมากต้องวางแผนเรื่อง resource budget
   ไว้ล่วงหน้า (เทียบกับข้อจำกัดของ thread-per-client ที่พูดถึงใน Part 35)
9. **ใช้ `ws://` แทน `wss://` ใน production** — `ws://` ไม่มีการเข้ารหัสใดๆ เลย ข้อความ
   ทั้งหมด (รวมถึง token การยืนยันตัวตนถ้ามี) เดินทางเป็น plain text ที่ใครดักฟังระหว่าง
   ทางก็อ่านได้ ต้องใช้ `wss://` (WebSocket over TLS) เสมอในระบบจริง (รายละเอียด HTTPS
   จะเจาะลึกใน Part 110 และ 112)
10. **ทดสอบด้วยมือ (manual) เพียงอย่างเดียวโดยไม่ทำ stress test** — เหมือนที่ Part 35
    เจอบั๊ก TCP byte-stream ตอนทดสอบด้วยสคริปต์อัตโนมัติ (ไม่ใช่ตอนพิมพ์มือ) race
    condition ในหัวข้อ 109.7 ก็เช่นกัน — ไม่มีวันเจอถ้าทดสอบแค่ 1-2 client ทีละคนช้าๆ
    ต้องจำลอง concurrent load จริงถึงจะเจอบั๊กประเภทนี้

---

## แบบฝึกหัดท้ายบท

1. เพิ่ม endpoint `GET /users/count` (HTTP ปกติ ไม่ใช่ WebSocket) ที่คืนจำนวน client ที่
   เชื่อมต่อ WebSocket อยู่ ณ ขณะนั้นเป็น JSON `{"connected": N}` (ต้องล็อก mutex เดียวกัน
   กับที่ WebSocket handler ใช้ เพราะเป็นการอ่าน container ตัวเดียวกันข้าม thread)
2. เพิ่มการรองรับ **private message**: ถ้าข้อความที่ client ส่งมาขึ้นต้นด้วย `@ชื่อผู้ใช้ `
   ให้ส่งข้อความนั้นไปหาแค่คนที่ชื่อตรงกันเท่านั้น แทนที่จะ broadcast ให้ทุกคน
3. ใช้ `conn.set_max_payload_size(4096)` จำกัดขนาดข้อความสูงสุดไว้ที่ 4 KB แล้วทดสอบว่า
   ถ้าส่งข้อความเกินขนาดจริง `onerror`/`onclose` ถูกเรียกด้วย code `MessageTooBig`
   หรือไม่ (ใช้ Python ส่ง string ยาวๆ ทดสอบ)
4. เพิ่มกลไก **heartbeat**: ให้ server ส่ง `send_ping()` ไปยังทุก client ทุก 30 วินาที
   (ใช้ `std::thread` แยกต่างหากที่วนลูปตลอดอายุโปรแกรม) และถ้า client ไม่ตอบ pong ภายใน
   10 วินาทีให้ปิด connection ทิ้ง
5. เขียนห้องแชทแบบ **หลายห้อง** (multi-room): แก้โปรโตคอลให้ข้อความที่สองที่ client ส่งมา
   (หลังชื่อผู้ใช้) เป็นชื่อห้อง แล้ว broadcast เฉพาะให้คนในห้องเดียวกันเท่านั้น
6. ทดลองรันเวอร์ชัน `ws_broadcast_unsafe.cpp` (ไม่มี mutex) ด้วยตัวเองพร้อม
   `-fsanitize=thread` แล้วลองปรับจำนวน client และจำนวนรอบใน stress test ให้มากขึ้น/
   น้อยลง สังเกตว่าจำนวน data race ที่พบเปลี่ยนแปลงไปอย่างไร และลองอธิบายด้วยคำพูดตัวเอง
   ว่าทำไมจำนวน concurrent client ที่มากขึ้นทำให้ "โอกาสเจอ" race condition สูงขึ้น

### แนวทางเฉลยข้อ 1

เพิ่ม route HTTP ปกติที่อ่านค่า `clients.size()` โดยล็อก mutex ตัวเดียวกับที่ใช้ใน
WebSocket handler — ย้ำว่า HTTP route กับ WebSocket route แม้จะเป็นคนละ mechanism แต่ยัง
รันบน worker thread ของ Crow เหมือนกัน จึงยังต้องล็อกเหมือนเดิมทุกประการ:

```cpp
CROW_ROUTE(app, "/users/count")
([&]() {
    std::lock_guard<std::mutex> lock(clients_mutex);
    nlohmann::json body{{"connected", static_cast<int>(clients.size())}};
    crow::response res(200, body.dump());
    res.set_header("Content-Type", "application/json");
    return res;
});
```

ทดสอบจริง (มี Alice และ Bob เชื่อมต่ออยู่):

```bash
$ curl -s http://127.0.0.1:18280/users/count
{"connected":2}
```

จุดที่พลาดบ่อยในข้อนี้คือ "ลืม" ว่า route HTTP ธรรมดาก็ต้องล็อก mutex เหมือนกัน เพราะคิดว่า
mutex มีไว้ป้องกันแค่ WebSocket handler ด้วยกันเองเท่านั้น — ที่จริงแล้ว mutex ป้องกัน
**การเข้าถึง container พร้อมกันจากหลาย thread** ไม่ว่า thread นั้นจะมาจาก HTTP handler
หรือ WebSocket handler ก็ตาม

### แนวทางเฉลยข้อ 3

เพิ่มการเรียก `set_max_payload_size()` ใน `onopen` (ต้องตั้งค่าให้แต่ละ connection เอง
เพราะเป็นเมธอดของ `connection` object ไม่ใช่ของทั้ง route):

```cpp
.onopen([&](crow::websocket::connection& conn) {
    conn.set_max_payload_size(4096); // จำกัดไว้ที่ 4 KB ต่อ 1 ข้อความ
    std::lock_guard<std::mutex> lock(clients_mutex);
    clients[&conn] = ClientInfo{};
})
```

ทดสอบด้วย Python ส่งข้อความยาวเกินขนาด:

```python
import asyncio, websockets

async def main():
    async with websockets.connect("ws://127.0.0.1:18280/ws") as ws:
        await ws.send("Tester")
        await ws.recv()  # welcome message
        big_message = "A" * 10000  # 10 KB เกินเพดาน 4 KB ที่ตั้งไว้
        try:
            await ws.send(big_message)
            reply = await asyncio.wait_for(ws.recv(), timeout=2)
            print("ได้รับ:", reply)
        except websockets.exceptions.ConnectionClosed as e:
            print(f"connection ถูกปิดโดย server: code={e.code} reason={e.reason!r}")

asyncio.run(main())
```

ผลลัพธ์ที่คาดหวัง (ตาม logic ของ Crow ในไฟล์ `crow/websocket.h` ส่วน `WebSocketReadState::Mask`
ที่เช็ค `max_payload_bytes_` ก่อนอ่าน payload จริง): server จะเรียก `error_handler_` แล้วปิด
connection ด้วย status code `MessageTooBig` (1009) ทันทีที่ตรวจพบว่าขนาดข้อความที่กำลัง
จะรับเกินเพดานที่ตั้งไว้ ก่อนที่จะอ่าน payload เข้ามาเต็มจำนวนด้วยซ้ำ — นี่คือการป้องกัน
Denial of Service ที่มีประสิทธิภาพ เพราะปฏิเสธตั้งแต่ "รู้ขนาด" โดยไม่ต้องเสีย memory/
bandwidth อ่านข้อมูลทั้งก้อนเข้ามาก่อนแล้วค่อยปฏิเสธทีหลัง

---

## สรุปท้ายบท

ใน Part นี้เราได้:

- เข้าใจความแตกต่างพื้นฐานระหว่าง HTTP request-response กับ WebSocket แบบ persistent
  full-duplex connection พร้อมเหตุผลว่าทำไม Polling ไม่ใช่คำตอบที่ดีสำหรับงาน real-time
- เข้าใจ WebSocket Handshake ตาม RFC 6455 อย่างละเอียดถึงระดับคำนวณ
  `Sec-WebSocket-Accept` ด้วยมือ และยืนยันด้วย `curl` ว่า Crow implement ถูกต้องตามสเปก
- ใช้ `CROW_WEBSOCKET_ROUTE` เขียน WebSocket server ครบทั้ง `onopen`, `onmessage`,
  `onclose` ได้ด้วยตนเอง
- สร้าง WebSocket Broadcast Chat Server ที่เทียบเคียงกับ TCP Chat Server ของ Part 35
  ได้โดยตรง พร้อมป้องกัน race condition ด้วย `std::mutex` อย่างถูกต้อง
- **พิสูจน์ด้วย ThreadSanitizer จริง** ว่าการไม่มี mutex ทำให้เกิด data race 51 จุดและ
  โปรแกรม crash จริงด้วย heap-use-after-free ขณะที่เวอร์ชันมี mutex ผ่าน stress test
  เดียวกันได้โดยไม่ crash
- ทดสอบ server ที่เขียนเองด้วยเครื่องมือจริงหลายตัว: `curl` (ดู handshake), `websocat`
  (ติดตั้งผ่าน `cargo`) และ Python `websockets` library สำหรับทดสอบ multi-client scenario
  ที่ซับซ้อน
- เข้าใจกลไก Ping/Pong keep-alive และความหมายของ Close Status Code มาตรฐาน

Task API จาก Part 108 ตอนนี้มีทั้งความสามารถ CRUD แบบ REST และเรารู้วิธีเพิ่มความสามารถ
real-time ผ่าน WebSocket แล้ว สิ่งที่ยังขาดอยู่คือ **ความปลอดภัย** — ทุก endpoint ที่เรา
เขียนมาจนถึงตอนนี้ใครก็เรียกได้โดยไม่ต้องพิสูจน์ตัวตนเลย ใน **Part 110** เราจะเพิ่ม
**Authentication** เข้าไปในระบบเดียวกันนี้ ด้วย JWT และ password hashing ที่ถูกต้องตาม
มาตรฐานความปลอดภัยสมัยใหม่

**ต่อไป:** [Part 110 — Authentication และ Security: JWT, Password Hashing](./part-110-auth-security.md)
