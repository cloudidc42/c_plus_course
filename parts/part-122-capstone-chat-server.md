# Part 122: Capstone 2 — Real-time Multiplayer Chat Server (Socket + WebSocket + Multi-threading) (Step 969–976)

> Module K — Capstone Projects และบทสรุป | Part 122 จาก 125
> Step ที่ครอบคลุมใน Part นี้: Step 969–976
> Part ก่อนหน้า: [Part 121 — Capstone 1: E-Commerce Backend API เต็มรูปแบบ](./part-121-capstone-ecommerce-api.md) | Part ถัดไป: [Part 123 — Capstone 3: Distributed Key-Value Store](./part-123-capstone-distributed-kv.md)

## เป้าหมายการเรียนรู้ (Learning Objectives)

หลังจบ Part นี้ ผู้เรียนจะสามารถ:

1. ออกแบบและสร้าง **Real-time Chat Server ระดับ capstone** ที่รองรับทั้ง raw TCP socket
   (อัปเกรดจาก Part 35) และ WebSocket ผ่าน Crow (อัปเกรดจาก Part 109) **ในโปรเซสเดียวกัน**
   โดยทั้งสอง transport แชร์ห้องแชท ผู้ใช้ และประวัติแชทชุดเดียวกันได้จริง
2. ออกแบบ **thread-safe global registry** (username → connection) ที่ป้องกัน race condition
   ได้อย่างถูกต้องครบถ้วน และ **พิสูจน์ด้วย ThreadSanitizer จริง** ว่าไม่มี data race หลงเหลือ
   ในโค้ดแอปพลิเคชันที่เขียนเอง ภายใต้ concurrent stress test ที่ยิงทั้งสอง transport พร้อมกัน
3. สร้างระบบ **ห้องแชทหลายห้อง (multi-room)** ที่ client join/leave/list ได้ พร้อม
   **presence notification** (แจ้งเตือนเข้า/ออกห้องแบบ real-time) และ **private message (DM)**
   ระหว่างผู้ใช้สองคนโดยไม่ broadcast ให้คนอื่นเห็น
4. เก็บ **ประวัติแชทลง SQLite** (ต่อยอด RAII wrapper จาก Part 105) เพื่อให้ client ที่เพิ่ง
   join ห้อง (หรือเพิ่งต่อกลับมาใหม่หลัง server restart) ดึงข้อความล่าสุดย้อนหลังได้ทันที
5. ออกแบบกลไก **heartbeat ตรวจจับ connection ที่ตายแบบเงียบ** (ไม่มี FIN/RST ส่งมาจริง)
   และบังคับตัดการเชื่อมต่อของ client ที่ไม่ตอบสนองอย่างถูกต้อง โดยไม่กระทบ client อื่น
6. เขียน **load test แบบ concurrent จริง** (Python `asyncio` + `websockets`) ที่ยิง client
   นับร้อยตัวพร้อมกันกระจายในหลายห้อง แล้วพิสูจน์ทางตัวเลขว่า **ไม่มีข้อความขาด ซ้ำ หรือ
   หลุดข้ามห้อง** แม้แต่รายการเดียว
7. ระบุและแก้ **Common Pitfalls เฉพาะของระบบระดับนี้** ได้ด้วยตัวเอง โดยเฉพาะ "การถือ mutex
   ค้างไว้ระหว่างเรียก `send()` ที่อาจ block" ซึ่งจะพิสูจน์ด้วย**เวลาที่วัดได้จริง**ว่าต่างกัน
   ระหว่างเวอร์ชันที่ถูกและผิดมากกว่า 8,000 เท่า
8. ต่อยอดระบบด้วยฟีเจอร์ที่ซับซ้อนขึ้น เช่น typing indicator และ message reactions ได้ด้วย
   ตัวเอง โดยมีตัวอย่างโค้ดที่คอมไพล์และทดสอบผ่านจริงเป็นแนวทาง

---

## 122.1 ภาพรวมโปรเจกต์ สถาปัตยกรรม และจุดที่ "อัปเกรด" จาก Part 35 / Part 109 (Step 969)

Part นี้คือ **Capstone ที่สองจากสี่โปรเจกต์ปิดหลักสูตร** โจทย์ของเราคือนำสามสิ่งที่เรียนแยก
กันมาตลอดหลักสูตรมาประกอบเป็นระบบเดียวที่ใช้งานได้จริงระดับ production:

| มาจาก Part | ความรู้ที่ใช้ในโปรเจกต์นี้ |
|---|---|
| Part 35 — โปรเจกต์ TCP Chat Server/Client | สถาปัตยกรรม thread-per-client, การจัดการ TCP byte stream, การจัดการ disconnect |
| Part 82, 109 — Mutex/Atomic และ WebSocket | `std::mutex`/`std::lock_guard`, `CROW_WEBSOCKET_ROUTE`, การพิสูจน์ thread-safety ด้วย ThreadSanitizer |
| Part 105 — SQLite ด้วย C++ | RAII wrapper (`SqliteDB`/`SqliteStatement`) สำหรับเก็บประวัติแชท |
| Part 107 — nlohmann/json | โปรโตคอลข้อความแบบ JSON แทน plain text ล้วนๆ แบบ Part 35 |
| Module G ทั้งหมด | `std::thread`, `std::mutex`, การออกแบบ critical section ให้สั้นที่สุด |

สิ่งที่ทำให้ Part นี้ต่างจาก Part 35 และ Part 109 อย่างชัดเจนคือ **ไม่ได้เลือกระหว่าง raw
socket กับ WebSocket แบบใดแบบหนึ่ง แต่รองรับทั้งคู่พร้อมกันในโปรเซสเดียว** — client ที่ต่อ
เข้ามาทาง raw TCP (port 7100) กับ client ที่ต่อเข้ามาทาง WebSocket (port 7101) จะเห็นห้องแชท
เดียวกัน คุยกันได้จริง และแชร์ประวัติแชทชุดเดียวกันทั้งหมด ตรงตามชื่อ Part ที่ว่า
"Socket + WebSocket + Multi-threading"

### สถาปัตยกรรมโดยรวม

```
                         ┌─────────────────────────────────────────┐
                         │              chat_server (1 process)     │
                         │                                           │
   TCP Client ───────────┼──▶ run_tcp_server() [std::thread]         │
   (Part 35 style,       │      accept() loop                        │
    JSON บรรทัดต่อบรรทัด) │      └─▶ handle_tcp_client() [std::thread/│
                         │            client, thread-per-client]      │
                         │                 │                          │
   WebSocket Client ─────┼──▶ Crow app.run() [multithreaded()]        │
   (Part 109 style,      │      CROW_WEBSOCKET_ROUTE("/ws")           │
    JSON ต่อ text frame)  │      onopen/onmessage/onclose               │
                         │                 │                          │
                         │                 ▼                          │
                         │   ┌─────────────────────────────────┐      │
                         │   │  handle_client_message()          │◀────┼── จุดเดียวที่ทั้งสอง
                         │   │  (โปรโตคอล JSON ตัวเดียว           │     │   transport มาบรรจบกัน
                         │   │   ไม่รู้เลยว่ามาจาก TCP หรือ WS)    │      │
                         │   └───────────────┬───────────────────┘      │
                         │                   ▼                          │
                         │         ┌───────────────────┐                │
                         │         │    ChatServer       │◀── mutex_    │
                         │         │  users_ / rooms_ /   │   (ป้องกัน   │
                         │         │  user_rooms_          │   registry) │
                         │         └─────────┬─────────┘                │
                         │                   │ db_mutex_ (แยกต่างหาก)    │
                         │                   ▼                          │
                         │            ┌─────────────┐                  │
                         │            │  SQLite DB    │                  │
                         │            │ (Part 105)     │                  │
                         │            └─────────────┘                  │
                         └─────────────────────────────────────────┘
```

### เปรียบเทียบสถาปัตยกรรมกับ Part ก่อนหน้า

| แนวคิด | Part 35 (Raw TCP) | Part 109 (WebSocket) | Part 122 (Capstone — Part นี้) |
|---|---|---|---|
| Transport ที่รองรับ | TCP เท่านั้น | WebSocket เท่านั้น | **ทั้ง TCP และ WebSocket พร้อมกัน** |
| โปรโตคอล | Plain text บรรทัดต่อบรรทัด | Plain text ต่อ 1 message | **JSON structured message** (รองรับ field หลากหลาย) |
| จำนวนห้อง | 1 ห้องรวม (broadcast room เดียว) | 1 ห้องรวม | **หลายห้องพร้อมกัน (multi-room)** |
| Private message | ไม่มี (มีในแบบฝึกหัดเฉลยเท่านั้น) | ไม่มี (มีในแบบฝึกหัดเฉลยเท่านั้น) | **มีในตัวระบบหลัก** |
| ประวัติแชท | ไม่มี | ไม่มี | **เก็บลง SQLite ดึงย้อนหลังได้** |
| การตรวจจับ dead connection | ผ่าน `recv()` คืนค่า ≤ 0 เท่านั้น | ผ่าน `onclose` เท่านั้น | **เพิ่ม heartbeat ตรวจจับ connection ที่เงียบแบบไม่มี FIN/RST** |
| การจัดการ fd หลัง disconnect | ต้องเรียง "ลบออกจาก list ก่อน close()" ด้วยมือ | Crow จัดการให้ | **`shared_ptr` เป็นเจ้าของ fd — ปัญหาหมดไปโดยโครงสร้าง** |
| การพิสูจน์ thread-safety | อธิบายด้วยแผนภาพ | ThreadSanitizer กับ broadcast เดียว | **ThreadSanitizer กับ mixed TCP+WS+churn+DM stress test** |

### สภาพแวดล้อมที่ใช้พัฒนาและทดสอบจริงตลอด Part นี้

ทุกคำสั่ง ทุกผลลัพธ์ในบทเรียนนี้รันจริงบนเครื่องทดสอบที่มีสเปกดังนี้ (ตรวจสอบแล้วก่อนเริ่ม
เขียนโค้ด):

```bash
$ g++ --version
g++ (Ubuntu 13.3.0-6ubuntu2~24.04.1) 13.3.0

$ ls /usr/local/include/crow/websocket.h
/usr/local/include/crow/websocket.h        # Crow มี WebSocket จริง (CROW_WEBSOCKET_ROUTE)

$ dpkg -l | grep nlohmann
ii  nlohmann-json3-dev   3.11.3-1   all   JSON for Modern C++

$ ls /usr/include/sqlite3.h
/usr/include/sqlite3.h

$ which websocat
/root/.cargo/bin/websocat

$ python3 -c "import websockets; print(websockets.__version__)"
17.1
```

---

## 122.2 ออกแบบโปรโตคอล JSON และ `IConnection`: จุดเชื่อมกลาง TCP ↔ WebSocket (Step 970)

### ทำไมต้องเปลี่ยนจาก Plain Text (Part 35/109) มาเป็น JSON

Part 35 และ Part 109 ใช้โปรโตคอล plain text ล้วนๆ เพราะมีแค่ "ข้อความแชท" อย่างเดียวให้ส่ง
แต่ Part นี้ต้องรองรับ action หลายแบบ (login, join, leave, message, dm, history, list_rooms)
ที่แต่ละแบบมี field ไม่เหมือนกัน — ถ้ายังใช้ plain text จะต้องประดิษฐ์ format เองและ parse
เองด้วยมือ (เสี่ยง bug สูง) เราจึงใช้ **JSON object บรรทัดเดียวต่อ 1 ข้อความ** (ทบทวนจาก
Part 107) โดยทุกข้อความต้องมี field `"type"` เป็นตัวบอกว่าเป็น action อะไร

### ตารางโปรโตคอลฉบับเต็ม

**ข้อความจาก Client → Server:**

| `type` | Field ที่ต้องมี | ความหมาย |
|---|---|---|
| `login` | `username` | ล็อกอินเข้าระบบ ต้องทำเป็นข้อความแรกเสมอ |
| `join` | `room` | เข้าร่วมห้อง (สร้างห้องใหม่อัตโนมัติถ้ายังไม่มี) |
| `leave` | `room` | ออกจากห้อง |
| `list_rooms` | — | ขอรายชื่อห้องทั้งหมดพร้อมจำนวนสมาชิก |
| `message` | `room`, `text` | ส่งข้อความแชทเข้าห้อง (ต้อง join ก่อน) |
| `dm` | `to`, `text` | ส่งข้อความส่วนตัวถึงผู้ใช้คนเดียว |
| `history` | `room`, `limit` (ทางเลือก) | ขอประวัติแชทของห้องย้อนหลัง |
| `pong` | — | ตอบรับ `ping` จาก server (สำหรับ heartbeat) |

**ข้อความจาก Server → Client:**

| `type` | Field | ความหมาย |
|---|---|---|
| `welcome` | `username` | ยืนยันว่า login สำเร็จ |
| `error` | `message` | แจ้ง error เป็นข้อความอ่านง่าย |
| `joined` | `room`, `members` | ยืนยันว่า join ห้องสำเร็จ พร้อมรายชื่อสมาชิกปัจจุบัน |
| `left` | `room` | ยืนยันว่าออกจากห้องสำเร็จ |
| `room_list` | `rooms` | รายชื่อห้องทั้งหมด พร้อมจำนวนสมาชิก |
| `presence` | `room`, `event` (`join`/`leave`), `username` | แจ้งเตือนคนเข้า-ออกห้อง |
| `message` | `room`, `from`, `text`, `ts` | ข้อความแชทในห้อง |
| `dm` | `from`, `to`, `text`, `ts` | ข้อความส่วนตัว |
| `history` | `room`, `messages` | ประวัติแชทย้อนหลัง (array ของ `{from, text, ts}`) |
| `ping` | — | server ตรวจสุขภาพ connection (ต้องตอบ `pong` กลับ) |

### `IConnection`: หัวใจของการรวม TCP กับ WebSocket เข้าด้วยกัน

ปัญหาการออกแบบที่สำคัญที่สุดของ Part นี้คือ: `ChatServer` (ตัวจัดการห้อง/ผู้ใช้/ประวัติ)
ต้อง "ส่งข้อความ" ไปยัง client ได้ โดยไม่สนใจเลยว่า client คนนั้นต่อเข้ามาทาง raw TCP socket
(ที่ส่งด้วย `::send()`) หรือทาง WebSocket (ที่ส่งด้วย `conn.send_text()`) — นี่คือปัญหาคลาสสิก
ที่แก้ด้วยหลักการ **Dependency Inversion**: สร้าง interface กลางที่ทั้งสอง transport implement
แล้วให้ business logic ทั้งหมดคุยผ่าน interface นั้นอย่างเดียว

```cpp
/* ---------- 4. IConnection: จุดเชื่อมกลาง ระหว่าง TCP กับ WebSocket ----------
 * ChatServer ไม่จำเป็นต้องรู้เลยว่าอีกฝั่งเป็น raw socket หรือ WebSocket
 * connection — มันคุยผ่าน interface นี้อย่างเดียว (Dependency Inversion) */
class IConnection {
public:
    virtual ~IConnection() = default;
    virtual void send(const std::string& line) = 0;
    virtual void force_close() = 0;
    virtual std::string transport() const = 0;

    /* username ว่าง = ยังไม่ login; ตั้งไว้ที่ตัว connection เอง (ไม่ใช่ thread-local)
     * เพื่อให้ทั้งฝั่ง TCP (thread-per-client) และฝั่ง WebSocket (event callback)
     * ใช้ logic เดียวกันได้ทุกประการผ่าน handle_client_message() ตัวเดียว */
    std::string username;
};
using ConnPtr = std::shared_ptr<IConnection>;
```

จุดที่น่าสนใจคือ `username` ถูกเก็บไว้ **ที่ตัว connection object เอง** ไม่ใช่ตัวแปร
thread-local แบบ Part 35 (ที่ `client_handler()` เก็บ `char name[NAME_SIZE]` เป็นตัวแปร local
บน stack ของ thread นั้น) เพราะฝั่ง WebSocket ไม่มี "1 thread ต่อ 1 client" ตลอดอายุ
connection แบบ TCP — Crow เรียก `onopen`/`onmessage`/`onclose` จาก worker thread ที่อาจ
**ไม่ใช่ thread เดียวกัน** ในแต่ละครั้ง การเก็บ state ไว้ที่ตัว connection object (ที่มีชีวิต
ตลอดอายุการเชื่อมต่อ) จึงเป็นที่เดียวที่ถูกต้องสำหรับทั้งสอง transport

### `TcpConnection`: ห่อ raw socket fd ด้วย RAII — แก้ปัญหา fd-reuse race ของ Part 35 **โดยโครงสร้าง**

```cpp
/* ---------- 5. TcpConnection: ห่อ raw socket fd แบบ RAII ----------
 * ทบทวนจาก Part 35: เดิม server ต้อง "ลบออกจาก client_list ก่อน แล้วค่อย close()"
 * ด้วยมือ เพื่อป้องกัน fd-reuse race ที่อธิบายไว้ใน 35.7 ในเวอร์ชันนี้เราใช้
 * shared_ptr เป็นเจ้าของ fd แทน: close(fd) จะเกิดขึ้นก็ต่อเมื่อ shared_ptr
 * ตัวสุดท้ายที่ชี้มายัง connection นี้ถูกทำลาย (ไม่ว่าจะเป็นจาก thread ของ
 * client เอง หรือจาก snapshot ที่ broadcast_room() ถืออยู่ชั่วคราว) ทำให้ไม่มี
 * ทางที่ fd หมายเลขเดิมจะถูกนำไปใช้ซ้ำกับ connection ใหม่ในระหว่างที่ยังมี
 * ใครถืออ้างอิงอยู่ — บั๊กที่ Part 35 ต้องระวังด้วยมือ กลายเป็นสิ่งที่ "เป็นไปไม่ได้
 * โดยโครงสร้าง" ในเวอร์ชันนี้ */
class TcpConnection : public IConnection {
public:
    explicit TcpConnection(int fd) : fd_(fd) {}
    ~TcpConnection() override {
        if (fd_ >= 0) close(fd_);
    }

    void send(const std::string& line) override {
        std::string out = line;
        out.push_back('\n');
        /* MSG_NOSIGNAL: ป้องกัน SIGPIPE เหมือนที่ Part 35 สอนไว้ (35.3) */
        ssize_t sent = ::send(fd_, out.data(), out.size(), MSG_NOSIGNAL);
        (void)sent; /* ถ้าส่งไม่สำเร็จ ปล่อยให้ recv() ใน client thread เป็นผู้ตรวจพบ disconnect เอง */
    }

    void force_close() override {
        /* shutdown() ปลุก recv() ที่ค้างอยู่ใน client thread ให้คืนค่า 0 ทันที
         * (เทคนิคเดียวกับที่ Part 35.5 ใช้ปลุก receive_loop() ตอน /quit) */
        shutdown(fd_, SHUT_RDWR);
    }

    std::string transport() const override { return "tcp"; }

private:
    int fd_;
};
```

ทบทวน Part 35.7: ปัญหา **fd-reuse race** เกิดขึ้นเมื่อ `close(fd)` ถูกเรียกก่อนที่จะลบ fd นั้น
ออกจาก `client_list` — OS อาจนำเลข fd เดิมไปมอบให้ connection ใหม่ทันที ทำให้ thread อื่นที่
กำลัง broadcast อยู่พอดี `send()` ไปผิดคน Part 35 แก้ปัญหานี้ด้วย **วินัยของโปรแกรมเมอร์**
(ต้องจำให้ขึ้นใจว่า "ลบออกจาก list ก่อน แล้วค่อย close()") ซึ่งเป็นจุดที่มือใหม่พลาดบ่อยมาก

เวอร์ชันนี้แก้ปัญหาเดียวกัน **โดยไม่ต้องพึ่งวินัยของใครเลย**: `fd_` ถูก close ก็ต่อเมื่อ
destructor ของ `TcpConnection` ถูกเรียก ซึ่งจะเกิดขึ้นก็ต่อเมื่อ `shared_ptr` ตัวสุดท้ายที่ชี้
มาที่ object นี้หมดอายุ ตราบใดที่ `broadcast_room()` (หัวข้อ 122.3) ยังถือสำเนา `shared_ptr`
อยู่ระหว่างกำลังวนส่งข้อความ ต่อให้ client thread เจ้าของ connection หลุดออกจากลูปไปแล้วและ
เรียก `logout()` เสร็จแล้ว ก็ยัง**ไม่มีทาง** ที่ fd จะถูกปิดและนำกลับไปใช้ซ้ำจนกว่า broadcast
ที่กำลังดำเนินอยู่จะจบก่อน — Reference counting แก้ปัญหาที่ Part 35 ต้องแก้ด้วยการจำลำดับขั้น
ตอนให้ถูกต้องด้วยมือ

### `WsConnection`: ห่อ `crow::websocket::connection&`

```cpp
/* ---------- 6. WsConnection: ห่อ crow::websocket::connection& ---------- */
class WsConnection : public IConnection {
public:
    explicit WsConnection(crow::websocket::connection& conn) : conn_(conn) {}

    void send(const std::string& line) override { conn_.send_text(line); }
    void force_close() override { conn_.close("heartbeat timeout", 1001); }
    std::string transport() const override { return "ws"; }

private:
    crow::websocket::connection& conn_;
};
```

`WsConnection` เบากว่า `TcpConnection` มาก เพราะ Crow เป็นเจ้าของ lifecycle ของ
`crow::websocket::connection` เองอยู่แล้ว (มีชีวิตตั้งแต่ `onopen` จนถึง `onclose`) เราแค่
"ห่อ" reference ไว้ให้ตรงกับ interface `IConnection` เท่านั้น ไม่ต้องจัดการ resource เอง

---

## 122.3 `ChatServer`: Thread-safe Registry, ห้องแชท, Presence, และหัวใจของ `broadcast_room()` (Step 971)

### โครงสร้างข้อมูลที่แชร์กันข้าม thread

```cpp
/* ============================================================
 * ChatServer — หัวใจของทั้งระบบ
 *
 * เก็บ 3 อย่างที่เป็น "shared mutable state" ข้าม thread:
 *   users_       : username -> connection ที่ใช้งานอยู่ตอนนี้
 *   rooms_       : room     -> เซตของ username ที่อยู่ในห้องนั้น
 *   user_rooms_  : username -> เซตของห้องที่ user คนนั้นเข้าร่วมอยู่ (reverse index)
 * ทั้งสามตัวถูกป้องกันด้วย mutex_ ตัวเดียวกัน (ไม่ใช่คนละตัว) เพราะการ join/leave/
 * logout ต้องแก้ไขมันพร้อมกันแบบ atomic — ถ้าใช้มือ mutex คนละตัวจะเกิดหน้าต่าง
 * เวลาสั้นๆ ที่ users_ กับ rooms_ ไม่สอดคล้องกัน (ดู "ข้อผิดพลาดที่พบบ่อย" ข้อ 3)
 *
 * db_/db_mutex_ แยกออกมาต่างหากโดยตั้งใจ: การเขียนลงดิสก์ (SQLite) ช้ากว่าการแก้
 * hash map ในหน่วยความจำมาก ถ้าใช้ mutex_ ตัวเดียวกันคลุมทั้งคู่ ทุก thread ที่แค่
 * จะ join/leave ห้องจะต้องรอ thread ที่กำลังเขียนไฟล์ history อยู่ (ดู "ข้อผิดพลาด
 * ที่พบบ่อย" ข้อ 2)
 * ============================================================ */
class ChatServer {
    /* ... ดูโค้ดเต็มในหัวข้อ 122.5 ... */
private:
    std::mutex mutex_;    /* ป้องกัน users_ / rooms_ / user_rooms_ / last_seen_ ทั้งหมดพร้อมกัน */
    std::mutex db_mutex_; /* ป้องกัน db_ แยกต่างหาก จงใจไม่ใช้ตัวเดียวกับ mutex_ */

    std::unordered_map<std::string, ConnPtr> users_;
    std::unordered_map<std::string, std::set<std::string>> rooms_;
    std::unordered_map<std::string, std::set<std::string>> user_rooms_;
    std::unordered_map<std::string, std::chrono::steady_clock::time_point> last_seen_;

    SqliteDB db_;
};
```

การตัดสินใจใช้ **mutex เดียว (`mutex_`) คลุมทั้ง 4 container** (แทนที่จะแยก mutex ต่อ container
ละตัวเพื่อ "ประสิทธิภาพที่ดีกว่า") เป็นการตัดสินใจที่ตั้งใจ: `join_room()` ต้องแก้ทั้ง `rooms_`
และ `user_rooms_` พร้อมกันแบบ atomic ถ้าใช้ mutex คนละตัวสำหรับแต่ละ container จะมีช่วงเวลา
สั้นๆ ที่ thread อื่นมองเห็น `rooms_` ที่อัปเดตแล้วแต่ `user_rooms_` ยังไม่อัปเดต (หรือกลับกัน)
— นี่คือ **inconsistent snapshot** ซึ่งเป็นบั๊กที่ subtle กว่า race condition ทั่วไปมาก เพราะ
โปรแกรมจะไม่ crash แต่ให้ผลลัพธ์ผิดแบบเงียบๆ (ดูรายละเอียดใน "ข้อผิดพลาดที่พบบ่อย" ข้อ 3)

### `login()` และ `logout()`: idempotent เพื่อรองรับการเรียกซ้ำจากหลายจุด

```cpp
bool login(const ConnPtr& conn, const std::string& username) {
    std::lock_guard<std::mutex> lock(mutex_);
    if (users_.count(username) > 0) return false; /* ชื่อซ้ำ */
    conn->username = username;
    users_[username] = conn;
    last_seen_[username] = std::chrono::steady_clock::now();
    return true;
}

/* logout() ถูกออกแบบให้เรียกซ้ำได้อย่างปลอดภัย (idempotent) เพราะอาจถูกเรียก
 * จากทั้ง client thread เอง (TCP), onclose (WS), และ heartbeat_tick() พร้อมกัน
 * ในบางจังหวะเวลา — เช็ค "users_[uname] == conn ตัวนี้จริงไหม" ก่อนลบเสมอ
 * เผื่อกรณี user คนเดิม reconnect ด้วย connection ใหม่ไปแล้วก่อนที่ของเก่าจะ
 * ทันรู้ตัวว่าหลุด */
void logout(const ConnPtr& conn) {
    if (!conn || conn->username.empty()) return;
    const std::string uname = conn->username;
    std::vector<std::string> joined_rooms;
    {
        std::lock_guard<std::mutex> lock(mutex_);
        auto it = users_.find(uname);
        if (it == users_.end() || it->second != conn) return;
        users_.erase(it);

        auto urit = user_rooms_.find(uname);
        if (urit != user_rooms_.end()) {
            joined_rooms.assign(urit->second.begin(), urit->second.end());
            for (const auto& r : joined_rooms) rooms_[r].erase(uname);
            user_rooms_.erase(urit);
        }
        last_seen_.erase(uname);
    }
    conn->username.clear();
    for (const auto& room : joined_rooms) {
        broadcast_room(room,
                        json{{"type", "presence"}, {"room", room}, {"event", "leave"}, {"username", uname}}
                            .dump());
    }
}
```

ทำไม `logout()` ต้องเรียกซ้ำได้อย่างปลอดภัย? เพราะในระบบนี้มี **สามเส้นทาง** ที่อาจเรียก
`logout()` กับ connection เดียวกัน: (1) client thread ของ TCP เองตอน `recv()` คืนค่า ≤ 0,
(2) `onclose` callback ของ WebSocket, (3) `heartbeat_tick()` เมื่อตรวจพบว่า connection เงียบ
เกินเวลาที่กำหนด สามเส้นทางนี้อาจแข่งกันเรียก `logout()` เกือบพร้อมกันได้จริง (เช่น heartbeat
เพิ่งบังคับตัด connection ไปในขณะที่ client เพิ่งจะปิดตัวเองพอดี) การเช็ค
`it->second != conn` ก่อนลบทุกครั้งทำให้การเรียกครั้งที่สองเป็น no-op อย่างปลอดภัย แทนที่จะ
ลบ user คนใหม่ที่เพิ่ง reconnect ด้วยชื่อเดิมทิ้งไปโดยไม่ได้ตั้งใจ

### `join_room()`: join + ส่งประวัติ + แจ้งเตือน presence ในขั้นตอนเดียว

```cpp
void join_room(const std::string& username, const std::string& room) {
    ConnPtr conn;
    std::vector<std::string> members_snapshot;
    {
        std::lock_guard<std::mutex> lock(mutex_);
        auto uit = users_.find(username);
        if (uit == users_.end()) return;
        conn = uit->second;
        rooms_[room].insert(username);
        user_rooms_[username].insert(room);
        members_snapshot.assign(rooms_[room].begin(), rooms_[room].end());
    }
    conn->send(json{{"type", "joined"}, {"room", room}, {"members", members_snapshot}}.dump());
    send_history_to(conn, room, DEFAULT_HISTORY_LIMIT);
    broadcast_room(room,
                    json{{"type", "presence"}, {"room", room}, {"event", "join"}, {"username", username}}
                        .dump(),
                    username);
}
```

การออกแบบให้ `join_room()` ส่งประวัติแชทให้อัตโนมัติทันทีหลัง join สำเร็จ (แทนที่จะให้ client
ต้องขอ `history` เองแยกต่างหาก) จำลองพฤติกรรมของแอปแชทจริง (เช่น Slack/Discord) ที่เมื่อเข้า
ช่องไหนก็จะเห็นข้อความล่าสุดทันที — นี่คือจุดที่ requirement "client ที่เพิ่ง reconnect ดึง
ประวัติได้" ถูกตอบสนองโดยธรรมชาติ ไม่ต้องมี logic พิเศษแยกสำหรับ "reconnect" เลย เพราะ
การ reconnect ก็คือการ join ห้องใหม่อีกครั้งนั่นเอง

### `broadcast_room()`: จุดที่สำคัญที่สุดของทั้งไฟล์

```cpp
void broadcast_room(const std::string& room, const std::string& line, const std::string& exclude = "") {
    std::vector<ConnPtr> targets;
    {
        std::lock_guard<std::mutex> lock(mutex_);
        auto rit = rooms_.find(room);
        if (rit == rooms_.end()) return;
        targets.reserve(rit->second.size());
        for (const auto& uname : rit->second) {
            if (uname == exclude) continue;
            auto uit = users_.find(uname);
            if (uit != users_.end()) targets.push_back(uit->second);
        }
    } /* <-- ปลดล็อก mutex_ ตรงนี้ ก่อนเริ่มส่งข้อมูลจริงแม้แต่ตัวเดียว */

    for (const auto& conn : targets) conn->send(line);
}
```

ฟังก์ชันนี้สั้นเพียง 12 บรรทัด แต่แก้ปัญหาสำคัญ 2 อย่างพร้อมกันด้วยเทคนิคเดียว — "สแนปช็อต
แล้วปลดล็อกก่อนส่ง":

1. **Iterator invalidation ปลอดภัยโดยอัตโนมัติ**: `targets` เป็น `std::vector` ที่ copy
   ออกมาแล้วจาก `rooms_[room]` ต่อให้ client คนอื่นหลุดออกจากห้องกลางคันระหว่างที่เรากำลังวน
   ส่งข้อความอยู่พอดี (thread อื่นกำลังแก้ `rooms_` ตัวจริงในเวลาเดียวกัน) ก็ไม่มีทางกระทบกับ
   `targets` ที่เรากำลังวนอยู่เลย เพราะเป็นคนละ container กันโดยสิ้นเชิง
2. **ไม่ถือ lock ระหว่างเรียก `conn->send()`**: นี่คือจุดที่สำคัญที่สุด — จะอธิบายละเอียดพร้อม
   หลักฐานเชิงประจักษ์ (วัดเวลาจริง) ในหัวข้อ "ข้อผิดพลาดที่พบบ่อย" ข้อ 1

---

## 122.4 Private Message และประวัติแชทด้วย SQLite (ต่อยอด Part 105) (Step 972)

### Schema ของฐานข้อมูล

```sql
CREATE TABLE IF NOT EXISTS messages (
    id     INTEGER PRIMARY KEY AUTOINCREMENT,
    room   TEXT NOT NULL,
    sender TEXT NOT NULL,
    body   TEXT NOT NULL,
    ts     INTEGER NOT NULL
);
CREATE INDEX IF NOT EXISTS idx_messages_room_ts ON messages(room, ts);

CREATE TABLE IF NOT EXISTS direct_messages (
    id        INTEGER PRIMARY KEY AUTOINCREMENT,
    sender    TEXT NOT NULL,
    recipient TEXT NOT NULL,
    body      TEXT NOT NULL,
    ts        INTEGER NOT NULL
);
```

`idx_messages_room_ts` สำคัญมาก — ทุกครั้งที่ client join ห้อง เราจะ query
`WHERE room = ? ORDER BY ts DESC LIMIT ?` (ดูโค้ดด้านล่าง) ถ้าไม่มี index บนคอลัมน์ `room`
SQLite จะต้องสแกนทั้งตารางทุกครั้ง ซึ่งช้าลงเรื่อยๆ เมื่อประวัติแชทสะสมมากขึ้น

### RAII Wrapper: ต่อยอดจาก Part 105 เพิ่ม `bind_int64`/`column_int64`

Part 105 สอนการเขียน `SqliteDB`/`SqliteStatement` เป็น RAII wrapper รอบ `sqlite3*` และ
`sqlite3_stmt*` ไว้แล้ว (105.6) เราเอาโค้ดนั้นมาใช้ตรงๆ โดยเพิ่มแค่ 2 เมธอดที่ตอนนั้นยังไม่ใช้:
`bind_int64`/`column_int64` สำหรับเก็บ **timestamp แบบ Unix epoch** (ค่า `time_t` ของปี 2026
เกินขอบเขตของ `int` 32-bit ที่ Part 105 ใช้ตอนนั้นไปแล้ว):

```cpp
// sqlite_raii.hpp — ต่อยอดจาก Part 105: 05_sqlite_raii.hpp
#pragma once
#include <sqlite3.h>
#include <stdexcept>
#include <string>
#include <utility>

class SqliteStatement {
public:
    SqliteStatement(sqlite3* db, const std::string& sql) {
        if (sqlite3_prepare_v2(db, sql.c_str(), -1, &stmt_, nullptr) != SQLITE_OK) {
            throw std::runtime_error("prepare ล้มเหลว: " + std::string(sqlite3_errmsg(db)));
        }
    }
    ~SqliteStatement() {
        if (stmt_) sqlite3_finalize(stmt_);
    }
    SqliteStatement(const SqliteStatement&) = delete;
    SqliteStatement& operator=(const SqliteStatement&) = delete;
    SqliteStatement(SqliteStatement&& other) noexcept : stmt_(other.stmt_) { other.stmt_ = nullptr; }
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
    void bind_int(int index, int value) { sqlite3_bind_int(stmt_, index, value); }
    // เพิ่มจาก Part 105: bind ค่า 64-bit สำหรับ timestamp (เกินขอบเขต int 32-bit)
    void bind_int64(int index, long long value) {
        sqlite3_bind_int64(stmt_, index, static_cast<sqlite3_int64>(value));
    }

    bool step() {
        int rc = sqlite3_step(stmt_);
        if (rc == SQLITE_ROW) return true;
        if (rc == SQLITE_DONE) return false;
        throw std::runtime_error("step ล้มเหลว: " + std::string(sqlite3_errmsg(sqlite3_db_handle(stmt_))));
    }

    int column_int(int col) const { return sqlite3_column_int(stmt_, col); }
    long long column_int64(int col) const { return static_cast<long long>(sqlite3_column_int64(stmt_, col)); }
    std::string column_text(int col) const {
        const unsigned char* text = sqlite3_column_text(stmt_, col);
        return text ? reinterpret_cast<const char*>(text) : "";
    }

private:
    sqlite3_stmt* stmt_ = nullptr;
};

class SqliteDB {
public:
    explicit SqliteDB(const std::string& path) {
        // SQLITE_OPEN_FULLMUTEX: ขอโหมด "Serialized" ของ SQLite เอง (thread-safe ภายในตัวมัน)
        // แม้เราจะล็อก db_mutex_ ของแอปคลุมทุกการเรียกอยู่แล้วก็ตาม การขอ flag นี้ตรงๆ
        // ทำให้ชัดเจนไม่ต้องพึ่ง "ค่า default ของแต่ละแพ็กเกจ" ซึ่งอาจต่างกันไปในแต่ละระบบ
        int rc = sqlite3_open_v2(path.c_str(), &db_,
                                  SQLITE_OPEN_READWRITE | SQLITE_OPEN_CREATE | SQLITE_OPEN_FULLMUTEX,
                                  nullptr);
        if (rc != SQLITE_OK) {
            std::string msg = db_ ? sqlite3_errmsg(db_) : "ไม่ทราบสาเหตุ";
            sqlite3_close(db_);
            throw std::runtime_error("เปิดฐานข้อมูลไม่สำเร็จ: " + msg);
        }
        sqlite3_exec(db_, "PRAGMA foreign_keys = ON;", nullptr, nullptr, nullptr);
    }
    ~SqliteDB() { if (db_) sqlite3_close(db_); }
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
    SqliteStatement prepare(const std::string& sql) { return SqliteStatement(db_, sql); }

private:
    sqlite3* db_ = nullptr;
};
```

### `send_dm()`: private message ที่ไม่ broadcast ให้ใครเห็น

```cpp
bool send_dm(const std::string& from, const std::string& to, const std::string& text) {
    ConnPtr to_conn, from_conn;
    {
        std::lock_guard<std::mutex> lock(mutex_);
        auto tit = users_.find(to);
        if (tit == users_.end()) return false;
        to_conn = tit->second;
        auto fit = users_.find(from);
        if (fit != users_.end()) from_conn = fit->second;
    }
    const long long ts = now_epoch_seconds();
    persist_dm(from, to, text, ts);

    json payload = {{"type", "dm"}, {"from", from}, {"to", to}, {"text", text}, {"ts", ts}};
    to_conn->send(payload.dump());
    if (from_conn && from_conn != to_conn) from_conn->send(payload.dump()); /* echo ให้ผู้ส่งเห็นด้วย */
    return true;
}
```

สังเกตว่า `send_dm()` **ไม่แตะ `rooms_` เลยแม้แต่บรรทัดเดียว** — มันมองหา connection ของผู้รับ
โดยตรงจาก `users_` แล้วส่งข้อความหาแค่ 2 คน (ผู้ส่งกับผู้รับ) เท่านั้น ไม่มีใครในห้องไหนเห็น
ข้อความนี้เลย ตรงตาม requirement "private message ระหว่างผู้ใช้โดยไม่ broadcast ให้คนอื่นเห็น"

### `fetch_room_history()`: ดึงประวัติย้อนหลัง เรียงเก่า → ใหม่

```cpp
std::vector<HistoryRow> fetch_room_history(const std::string& room, int limit) {
    std::lock_guard<std::mutex> lock(db_mutex_);
    auto stmt = db_.prepare(
        "SELECT sender, body, ts FROM messages WHERE room = ? ORDER BY ts DESC, id DESC LIMIT ?;");
    stmt.bind_text(1, room);
    stmt.bind_int(2, limit);
    std::vector<HistoryRow> rows;
    while (stmt.step()) {
        rows.push_back(HistoryRow{stmt.column_text(0), stmt.column_text(1), stmt.column_int64(2)});
    }
    std::reverse(rows.begin(), rows.end()); /* เรียงเก่า -> ใหม่ ให้อ่านเหมือนแชทจริง */
    return rows;
}
```

Query ใช้ `ORDER BY ts DESC ... LIMIT ?` เพื่อดึง "N ข้อความล่าสุด" ได้อย่างมีประสิทธิภาพ
(ใช้ index ที่สร้างไว้) แล้วค่อย `std::reverse()` กลับเป็นลำดับเก่า → ใหม่ในหน่วยความจำ
เพื่อให้ client แสดงผลได้ตรงกับที่มนุษย์คาดหวัง (ข้อความเก่าอยู่บน ข้อความใหม่อยู่ล่าง)

---

## 122.5 Dual Transport เต็มรูปแบบ: Raw TCP Socket + WebSocket ในโปรเซสเดียว (Step 973)

### ฝั่ง Raw TCP: อัปเกรดจาก Part 35 สองจุดสำคัญ

```cpp
/* ============================================================
 * ฝั่ง Raw TCP Socket (อัปเกรดจาก Part 35)
 *
 * ต่างจาก Part 35 สองจุดสำคัญ:
 *   1) ใช้ std::thread (Module G) แทน pthread เพราะเป็นโค้ด C++ ล้วนของ Module K
 *   2) ใช้ recv_buffer สะสม + วนดึงบรรทัดออกมาเรื่อยๆ แทนเทคนิค "leftover ครั้งเดียว"
 *      ของ Part 35.4 — เวอร์ชันนี้ถูกต้องทั่วไปกว่า เพราะรองรับทั้งกรณีที่ recv()
 *      ได้หลายข้อความ (หลาย \n) รวมกันมาในครั้งเดียว และกรณีที่ข้อความหนึ่งถูกตัด
 *      แบ่งมาหลาย recv() ก็ยังประกอบกลับเป็นบรรทัดที่สมบูรณ์ได้ถูกต้องเสมอ
 * ============================================================ */
void handle_tcp_client(ChatServer& server, int fd) {
    auto conn = std::make_shared<TcpConnection>(fd);
    std::string buffer;
    char chunk[4096];

    for (;;) {
        ssize_t n = recv(fd, chunk, sizeof(chunk), 0);
        if (n <= 0) break; /* n==0: ปิดแบบสุภาพ, n<0: error/ECONNRESET — cleanup เหมือนกันทั้งคู่ */
        buffer.append(chunk, static_cast<size_t>(n));

        size_t pos;
        while ((pos = buffer.find('\n')) != std::string::npos) {
            std::string line = buffer.substr(0, pos);
            buffer.erase(0, pos + 1);
            if (line.empty()) continue;
            try {
                json msg = json::parse(line);
                handle_client_message(server, conn, msg);
            } catch (const json::parse_error& e) {
                conn->send(err_json(std::string("invalid JSON: ") + e.what()));
            }
        }
    }

    server.logout(conn);
    /* conn (shared_ptr) หมด scope ตรงนี้ — ถ้าไม่มี broadcast snapshot ไหนถืออยู่
     * ณ ขณะนี้พอดี ~TcpConnection() จะถูกเรียกทันทีและปิด fd ให้อัตโนมัติ */
}
```

จุดสำคัญที่ต่างจากเทคนิค "leftover" ของ Part 35.4: ตอนนั้นเราจัดการปัญหา "TCP เป็น byte
stream" แค่ครั้งเดียวตอนอ่านชื่อผู้ใช้ (เก็บส่วนที่เหลือไว้ในตัวแปร `leftover` ตัวเดียว)
เวอร์ชันนี้ใช้ `buffer` เป็น **accumulator ถาวรตลอดอายุการเชื่อมต่อ**: ทุกครั้งที่ `recv()`
ได้ข้อมูลมา จะต่อท้ายเข้า `buffer` ก่อนเสมอ แล้ววนดึง `\n` ออกมาทีละบรรทัดจนกว่าจะไม่เหลือ
บรรทัดสมบูรณ์ในนั้นแล้ว — วิธีนี้ถูกต้องทั่วไปกว่าและใช้ได้กับทุกข้อความ ไม่ใช่แค่บรรทัดแรก

### `run_tcp_server()`: accept loop ด้วย `std::thread`

```cpp
void run_tcp_server(ChatServer& server, int port) {
    int server_fd = socket(AF_INET, SOCK_STREAM, 0);
    int opt = 1;
    setsockopt(server_fd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));

    sockaddr_in addr{};
    addr.sin_family = AF_INET;
    addr.sin_addr.s_addr = INADDR_ANY;
    addr.sin_port = htons(static_cast<uint16_t>(port));

    if (bind(server_fd, reinterpret_cast<sockaddr*>(&addr), sizeof(addr)) < 0) {
        perror("[tcp] bind");
        return;
    }
    if (listen(server_fd, 128) < 0) {
        perror("[tcp] listen");
        return;
    }
    std::cout << "[tcp] กำลังฟังที่ port " << port << std::endl;

    for (;;) {
        sockaddr_in client_addr{};
        socklen_t len = sizeof(client_addr);
        int client_fd = accept(server_fd, reinterpret_cast<sockaddr*>(&client_addr), &len);
        if (client_fd < 0) {
            if (errno == EINTR) continue;
            break;
        }
        std::thread(handle_tcp_client, std::ref(server), client_fd).detach();
    }
    close(server_fd);
}
```

สังเกตว่าเราไม่ต้อง `malloc(sizeof(int))` เพื่อส่ง `client_fd` ให้ thread ใหม่แบบที่ Part 35.3
ต้องทำ (ปัญหา "ตัวแปร local ถูก reuse ก่อน thread ใหม่จะทันอ่านค่า") เพราะ `std::thread`
constructor **copy ค่า argument เข้าไปเก็บเป็นของตัวเองทันที** (ผ่าน `std::decay`/perfect
forwarding ภายใน) ไม่ใช่ส่ง pointer ไปยัง stack ของ caller เหมือน `pthread_create()` — นี่คือ
อีกหนึ่งจุดที่ Modern C++ ทำให้บั๊กคลาสสิกของ Part 35 "หายไปโดยธรรมชาติ" โดยไม่ต้องจำอะไรเป็น
พิเศษ

### `main()`: ผูกทุกอย่างเข้าด้วยกัน — TCP thread, Heartbeat thread, และ Crow WebSocket

```cpp
int main(int argc, char** argv) {
    std::string db_path = (argc > 1) ? argv[1] : "chat_history.db";

    signal(SIGPIPE, SIG_IGN); /* ทบทวนจาก Part 35.3: จำเป็นสำหรับฝั่ง raw TCP */

    ChatServer server(db_path);
    std::cout << "=== Capstone 2: Real-time Multiplayer Chat Server ===" << std::endl;
    std::cout << "ฐานข้อมูลประวัติแชท: " << db_path << std::endl;

    std::thread tcp_thread(run_tcp_server, std::ref(server), TCP_PORT);
    tcp_thread.detach();

    std::thread heartbeat_thread([&server]() {
        for (;;) {
            std::this_thread::sleep_for(HEARTBEAT_INTERVAL);
            server.heartbeat_tick();
        }
    });
    heartbeat_thread.detach();

    /* ---------------- ฝั่ง WebSocket (อัปเกรดจาก Part 109) ---------------- */
    crow::SimpleApp app;
    std::mutex ws_map_mutex; /* ป้องกัน ws_conns เท่านั้น — เป็นคนละเรื่องกับ mutex_ ภายใน ChatServer */
    std::unordered_map<crow::websocket::connection*, ConnPtr> ws_conns;

    CROW_ROUTE(app, "/")
    ([]() {
        return "Capstone Chat Server: WebSocket ws://<host>:7101/ws, Raw TCP <host>:7100";
    });

    CROW_WEBSOCKET_ROUTE(app, "/ws")
        .onopen([&](crow::websocket::connection& c) {
            auto conn = std::make_shared<WsConnection>(c);
            std::lock_guard<std::mutex> lock(ws_map_mutex);
            ws_conns[&c] = conn;
        })
        .onmessage([&](crow::websocket::connection& c, const std::string& data, bool is_binary) {
            if (is_binary) return;
            ConnPtr conn;
            {
                std::lock_guard<std::mutex> lock(ws_map_mutex);
                auto it = ws_conns.find(&c);
                if (it == ws_conns.end()) return;
                conn = it->second;
            }
            try {
                json msg = json::parse(data);
                handle_client_message(server, conn, msg);
            } catch (const json::parse_error& e) {
                conn->send(err_json(std::string("invalid JSON: ") + e.what()));
            }
        })
        .onclose([&](crow::websocket::connection& c, const std::string&, uint16_t) {
            ConnPtr conn;
            {
                std::lock_guard<std::mutex> lock(ws_map_mutex);
                auto it = ws_conns.find(&c);
                if (it == ws_conns.end()) return;
                conn = it->second;
                ws_conns.erase(it);
            }
            server.logout(conn);
        });

    app.port(WS_PORT).multithreaded().run();
    return 0;
}
```

สังเกตว่า `ws_conns` (แผนที่ raw pointer ของ `crow::websocket::connection*` ไปยัง `ConnPtr`)
ใช้ mutex **แยกต่างหาก** (`ws_map_mutex`) จาก `mutex_` ภายใน `ChatServer` — เพราะมันเป็นเรื่อง
คนละชั้น: `ws_conns` มีหน้าที่แค่ "แปล raw pointer ของ Crow ให้เป็น `ConnPtr` ของเรา" เท่านั้น
ไม่เกี่ยวกับ business logic เรื่องห้อง/ผู้ใช้เลย การแยก mutex ตามความรับผิดชอบ (separation of
concerns) แบบนี้ทำให้ critical section ของแต่ละที่สั้นและเข้าใจง่ายขึ้น

### ไฟล์ `chat_server.cpp` ฉบับสมบูรณ์

รวมทุกส่วนที่อธิบายมาทั้งหมดเข้าด้วยกัน (คอมไพล์และรันจริงแล้วด้วย `g++ 13.3.0`
**ไม่มี warning แม้แต่บรรทัดเดียว** แม้จะเปิด `-Wall -Wextra -Wpedantic`):

```cpp
/* ============================================================
 * ชื่อไฟล์:     chat_server.cpp
 * คำอธิบาย:     Capstone 2 — Real-time Multiplayer Chat Server
 *              รองรับทั้ง raw TCP socket (อัปเกรดจาก Part 35) และ WebSocket
 *              ผ่าน Crow (อัปเกรดจาก Part 109) ในโปรเซสเดียวกัน แชร์
 *              "ห้องแชท" (room) เดียวกัน, รองรับ private message, ประวัติ
 *              แชทเก็บลง SQLite (ต่อยอด Part 105), presence notification,
 *              heartbeat ตรวจจับ connection ที่ตายแล้ว และ thread-safety
 *              เต็มรูปแบบที่พิสูจน์ด้วย ThreadSanitizer
 * ============================================================ */

/* ---------- 1. Header Includes ---------- */
#include "crow.h"
#include "sqlite_raii.hpp"
#include <nlohmann/json.hpp>

#include <arpa/inet.h>
#include <netinet/in.h>
#include <sys/socket.h>
#include <unistd.h>
#include <csignal>

#include <algorithm>
#include <chrono>
#include <iostream>
#include <map>
#include <memory>
#include <mutex>
#include <set>
#include <string>
#include <thread>
#include <unordered_map>
#include <vector>

using json = nlohmann::json;

/* ---------- 2. ค่าคงที่ของระบบ ---------- */
constexpr int   TCP_PORT               = 7100;
constexpr int   WS_PORT                = 7101;
constexpr int   DEFAULT_HISTORY_LIMIT  = 20;
constexpr int   MAX_USERNAME_LEN       = 32;
constexpr int   MAX_MESSAGE_LEN        = 2000;

/* ค่า heartbeat ตั้งสั้นเพื่อให้สาธิต/ทดสอบได้เร็วในบทเรียนนี้
 * ในระบบจริงควรตั้ง INTERVAL ~ 30 วินาที, TIMEOUT ~ 90 วินาที */
constexpr auto HEARTBEAT_INTERVAL = std::chrono::seconds(3);
constexpr auto HEARTBEAT_TIMEOUT  = std::chrono::seconds(8);

/* ---------- 3. ฟังก์ชันช่วยเล็กๆ ---------- */
static long long now_epoch_seconds() {
    return std::chrono::duration_cast<std::chrono::seconds>(
               std::chrono::system_clock::now().time_since_epoch())
        .count();
}

static std::string err_json(const std::string& message) {
    return json{{"type", "error"}, {"message", message}}.dump();
}

/* ---------- 4. IConnection: จุดเชื่อมกลาง ระหว่าง TCP กับ WebSocket ---------- */
class IConnection {
public:
    virtual ~IConnection() = default;
    virtual void send(const std::string& line) = 0;
    virtual void force_close() = 0;
    virtual std::string transport() const = 0;
    std::string username;
};
using ConnPtr = std::shared_ptr<IConnection>;

/* ---------- 5. TcpConnection: ห่อ raw socket fd แบบ RAII ---------- */
class TcpConnection : public IConnection {
public:
    explicit TcpConnection(int fd) : fd_(fd) {}
    ~TcpConnection() override {
        if (fd_ >= 0) close(fd_);
    }

    void send(const std::string& line) override {
        std::string out = line;
        out.push_back('\n');
        ssize_t sent = ::send(fd_, out.data(), out.size(), MSG_NOSIGNAL);
        (void)sent;
    }

    void force_close() override { shutdown(fd_, SHUT_RDWR); }
    std::string transport() const override { return "tcp"; }

private:
    int fd_;
};

/* ---------- 6. WsConnection: ห่อ crow::websocket::connection& ---------- */
class WsConnection : public IConnection {
public:
    explicit WsConnection(crow::websocket::connection& conn) : conn_(conn) {}

    void send(const std::string& line) override { conn_.send_text(line); }
    void force_close() override { conn_.close("heartbeat timeout", 1001); }
    std::string transport() const override { return "ws"; }

private:
    crow::websocket::connection& conn_;
};

/* ---------- 7. โครงสร้างข้อมูลของ 1 ข้อความในประวัติแชท ---------- */
struct HistoryRow {
    std::string sender;
    std::string body;
    long long   ts;
};

class ChatServer {
public:
    explicit ChatServer(const std::string& db_path) : db_(db_path) {
        db_.exec(
            "CREATE TABLE IF NOT EXISTS messages ("
            "  id     INTEGER PRIMARY KEY AUTOINCREMENT,"
            "  room   TEXT NOT NULL,"
            "  sender TEXT NOT NULL,"
            "  body   TEXT NOT NULL,"
            "  ts     INTEGER NOT NULL"
            ");");
        db_.exec("CREATE INDEX IF NOT EXISTS idx_messages_room_ts ON messages(room, ts);");
        db_.exec(
            "CREATE TABLE IF NOT EXISTS direct_messages ("
            "  id        INTEGER PRIMARY KEY AUTOINCREMENT,"
            "  sender    TEXT NOT NULL,"
            "  recipient TEXT NOT NULL,"
            "  body      TEXT NOT NULL,"
            "  ts        INTEGER NOT NULL"
            ");");
    }

    bool login(const ConnPtr& conn, const std::string& username) {
        std::lock_guard<std::mutex> lock(mutex_);
        if (users_.count(username) > 0) return false;
        conn->username = username;
        users_[username] = conn;
        last_seen_[username] = std::chrono::steady_clock::now();
        return true;
    }

    void logout(const ConnPtr& conn) {
        if (!conn || conn->username.empty()) return;
        const std::string uname = conn->username;
        std::vector<std::string> joined_rooms;
        {
            std::lock_guard<std::mutex> lock(mutex_);
            auto it = users_.find(uname);
            if (it == users_.end() || it->second != conn) return;
            users_.erase(it);

            auto urit = user_rooms_.find(uname);
            if (urit != user_rooms_.end()) {
                joined_rooms.assign(urit->second.begin(), urit->second.end());
                for (const auto& r : joined_rooms) rooms_[r].erase(uname);
                user_rooms_.erase(urit);
            }
            last_seen_.erase(uname);
        }
        conn->username.clear();
        for (const auto& room : joined_rooms) {
            broadcast_room(room,
                            json{{"type", "presence"}, {"room", room}, {"event", "leave"}, {"username", uname}}
                                .dump());
        }
        std::cout << "[-] " << uname << " ออกจากระบบ (transport=" << conn->transport() << ")" << std::endl;
    }

    void touch(const std::string& username) {
        if (username.empty()) return;
        std::lock_guard<std::mutex> lock(mutex_);
        last_seen_[username] = std::chrono::steady_clock::now();
    }

    void join_room(const std::string& username, const std::string& room) {
        ConnPtr conn;
        std::vector<std::string> members_snapshot;
        {
            std::lock_guard<std::mutex> lock(mutex_);
            auto uit = users_.find(username);
            if (uit == users_.end()) return;
            conn = uit->second;
            rooms_[room].insert(username);
            user_rooms_[username].insert(room);
            members_snapshot.assign(rooms_[room].begin(), rooms_[room].end());
        }
        conn->send(json{{"type", "joined"}, {"room", room}, {"members", members_snapshot}}.dump());
        send_history_to(conn, room, DEFAULT_HISTORY_LIMIT);
        broadcast_room(room,
                        json{{"type", "presence"}, {"room", room}, {"event", "join"}, {"username", username}}
                            .dump(),
                        username);
        std::cout << "[+] " << username << " เข้าร่วมห้อง '" << room << "' (สมาชิกทั้งหมด "
                  << members_snapshot.size() << " คน)" << std::endl;
    }

    void leave_room(const std::string& username, const std::string& room) {
        ConnPtr conn;
        bool was_member = false;
        {
            std::lock_guard<std::mutex> lock(mutex_);
            auto uit = users_.find(username);
            if (uit == users_.end()) return;
            conn = uit->second;
            auto rit = rooms_.find(room);
            if (rit != rooms_.end()) was_member = rit->second.erase(username) > 0;
            auto urit = user_rooms_.find(username);
            if (urit != user_rooms_.end()) urit->second.erase(room);
        }
        if (!was_member) {
            conn->send(err_json("คุณไม่ได้อยู่ในห้อง '" + room + "'"));
            return;
        }
        conn->send(json{{"type", "left"}, {"room", room}}.dump());
        broadcast_room(room,
                        json{{"type", "presence"}, {"room", room}, {"event", "leave"}, {"username", username}}
                            .dump());
        std::cout << "[-] " << username << " ออกจากห้อง '" << room << "'" << std::endl;
    }

    json list_rooms() {
        std::lock_guard<std::mutex> lock(mutex_);
        json arr = json::array();
        for (const auto& [room, members] : rooms_) {
            arr.push_back(json{{"room", room}, {"members", members.size()}});
        }
        return arr;
    }

    bool send_room_message(const std::string& from, const std::string& room, const std::string& text) {
        bool member;
        {
            std::lock_guard<std::mutex> lock(mutex_);
            auto rit = rooms_.find(room);
            member = (rit != rooms_.end()) && rit->second.count(from) > 0;
        }
        if (!member) return false;

        const long long ts = now_epoch_seconds();
        persist_room_message(room, from, text, ts);
        broadcast_room(room,
                        json{{"type", "message"}, {"room", room}, {"from", from}, {"text", text}, {"ts", ts}}
                            .dump());
        return true;
    }

    bool send_dm(const std::string& from, const std::string& to, const std::string& text) {
        ConnPtr to_conn, from_conn;
        {
            std::lock_guard<std::mutex> lock(mutex_);
            auto tit = users_.find(to);
            if (tit == users_.end()) return false;
            to_conn = tit->second;
            auto fit = users_.find(from);
            if (fit != users_.end()) from_conn = fit->second;
        }
        const long long ts = now_epoch_seconds();
        persist_dm(from, to, text, ts);

        json payload = {{"type", "dm"}, {"from", from}, {"to", to}, {"text", text}, {"ts", ts}};
        to_conn->send(payload.dump());
        if (from_conn && from_conn != to_conn) from_conn->send(payload.dump());
        return true;
    }

    void send_history_to(const ConnPtr& conn, const std::string& room, int limit) {
        std::vector<HistoryRow> rows = fetch_room_history(room, limit);
        json arr = json::array();
        for (const auto& r : rows) {
            arr.push_back(json{{"from", r.sender}, {"text", r.body}, {"ts", r.ts}});
        }
        conn->send(json{{"type", "history"}, {"room", room}, {"messages", arr}}.dump());
    }

    void heartbeat_tick() {
        std::vector<ConnPtr> to_ping;
        std::vector<ConnPtr> to_kill;
        const auto now = std::chrono::steady_clock::now();
        {
            std::lock_guard<std::mutex> lock(mutex_);
            for (const auto& [uname, conn] : users_) {
                auto it = last_seen_.find(uname);
                const auto last = (it != last_seen_.end()) ? it->second : now;
                if (now - last > HEARTBEAT_TIMEOUT) {
                    to_kill.push_back(conn);
                } else {
                    to_ping.push_back(conn);
                }
            }
        }
        for (const auto& conn : to_ping) conn->send(R"({"type":"ping"})");
        for (const auto& conn : to_kill) {
            std::cout << "[heartbeat] " << conn->username
                      << " ไม่ตอบ pong ภายใน " << HEARTBEAT_TIMEOUT.count()
                      << " วินาที บังคับตัดการเชื่อมต่อ (transport=" << conn->transport() << ")" << std::endl;
            conn->force_close();
            logout(conn);
        }
    }

private:
    void broadcast_room(const std::string& room, const std::string& line, const std::string& exclude = "") {
        std::vector<ConnPtr> targets;
        {
            std::lock_guard<std::mutex> lock(mutex_);
            auto rit = rooms_.find(room);
            if (rit == rooms_.end()) return;
            targets.reserve(rit->second.size());
            for (const auto& uname : rit->second) {
                if (uname == exclude) continue;
                auto uit = users_.find(uname);
                if (uit != users_.end()) targets.push_back(uit->second);
            }
        }
        for (const auto& conn : targets) conn->send(line);
    }

    void persist_room_message(const std::string& room, const std::string& sender, const std::string& body,
                               long long ts) {
        std::lock_guard<std::mutex> lock(db_mutex_);
        auto stmt = db_.prepare("INSERT INTO messages(room, sender, body, ts) VALUES (?, ?, ?, ?);");
        stmt.bind_text(1, room);
        stmt.bind_text(2, sender);
        stmt.bind_text(3, body);
        stmt.bind_int64(4, ts);
        stmt.step();
    }

    void persist_dm(const std::string& from, const std::string& to, const std::string& body, long long ts) {
        std::lock_guard<std::mutex> lock(db_mutex_);
        auto stmt =
            db_.prepare("INSERT INTO direct_messages(sender, recipient, body, ts) VALUES (?, ?, ?, ?);");
        stmt.bind_text(1, from);
        stmt.bind_text(2, to);
        stmt.bind_text(3, body);
        stmt.bind_int64(4, ts);
        stmt.step();
    }

    std::vector<HistoryRow> fetch_room_history(const std::string& room, int limit) {
        std::lock_guard<std::mutex> lock(db_mutex_);
        auto stmt = db_.prepare(
            "SELECT sender, body, ts FROM messages WHERE room = ? ORDER BY ts DESC, id DESC LIMIT ?;");
        stmt.bind_text(1, room);
        stmt.bind_int(2, limit);
        std::vector<HistoryRow> rows;
        while (stmt.step()) {
            rows.push_back(HistoryRow{stmt.column_text(0), stmt.column_text(1), stmt.column_int64(2)});
        }
        std::reverse(rows.begin(), rows.end());
        return rows;
    }

    std::mutex mutex_;
    std::mutex db_mutex_;

    std::unordered_map<std::string, ConnPtr> users_;
    std::unordered_map<std::string, std::set<std::string>> rooms_;
    std::unordered_map<std::string, std::set<std::string>> user_rooms_;
    std::unordered_map<std::string, std::chrono::steady_clock::time_point> last_seen_;

    SqliteDB db_;
};

void handle_client_message(ChatServer& server, const ConnPtr& conn, const json& msg) {
    if (!msg.contains("type") || !msg["type"].is_string()) {
        conn->send(err_json("ข้อความต้องมี field 'type' เป็น string เสมอ"));
        return;
    }
    const std::string type = msg["type"];

    if (type == "pong") {
        server.touch(conn->username);
        return;
    }

    if (type == "login") {
        std::string username = msg.value("username", "");
        if (username.empty() || username.size() > MAX_USERNAME_LEN) {
            conn->send(err_json("username ต้องไม่ว่างและยาวไม่เกิน " +
                                 std::to_string(MAX_USERNAME_LEN) + " ตัวอักษร"));
            return;
        }
        if (!conn->username.empty()) {
            conn->send(err_json("login ไปแล้ว (ชื่อ " + conn->username + ")"));
            return;
        }
        if (!server.login(conn, username)) {
            conn->send(err_json("ชื่อผู้ใช้ '" + username + "' ถูกใช้งานอยู่แล้ว"));
            return;
        }
        conn->send(json{{"type", "welcome"}, {"username", username}}.dump());
        std::cout << "[login] " << username << " (transport=" << conn->transport() << ")" << std::endl;
        return;
    }

    if (conn->username.empty()) {
        conn->send(err_json("กรุณา login ก่อน (ส่ง {\"type\":\"login\",\"username\":...})"));
        return;
    }
    server.touch(conn->username);

    if (type == "join") {
        std::string room = msg.value("room", "");
        if (room.empty()) { conn->send(err_json("ต้องระบุ room")); return; }
        server.join_room(conn->username, room);
        return;
    }
    if (type == "leave") {
        std::string room = msg.value("room", "");
        if (room.empty()) { conn->send(err_json("ต้องระบุ room")); return; }
        server.leave_room(conn->username, room);
        return;
    }
    if (type == "list_rooms") {
        conn->send(json{{"type", "room_list"}, {"rooms", server.list_rooms()}}.dump());
        return;
    }
    if (type == "message") {
        std::string room = msg.value("room", "");
        std::string text = msg.value("text", "");
        if (room.empty() || text.empty()) { conn->send(err_json("ต้องระบุ room และ text")); return; }
        if (text.size() > MAX_MESSAGE_LEN) { conn->send(err_json("ข้อความยาวเกินไป")); return; }
        if (!server.send_room_message(conn->username, room, text)) {
            conn->send(err_json("คุณยังไม่ได้เข้าร่วมห้อง '" + room + "'"));
        }
        return;
    }
    if (type == "dm") {
        std::string to = msg.value("to", "");
        std::string text = msg.value("text", "");
        if (to.empty() || text.empty()) { conn->send(err_json("ต้องระบุ to และ text")); return; }
        if (text.size() > MAX_MESSAGE_LEN) { conn->send(err_json("ข้อความยาวเกินไป")); return; }
        if (!server.send_dm(conn->username, to, text)) {
            conn->send(err_json("ไม่พบผู้ใช้ '" + to + "' ในระบบขณะนี้"));
        }
        return;
    }
    if (type == "history") {
        std::string room = msg.value("room", "");
        int limit = msg.value("limit", DEFAULT_HISTORY_LIMIT);
        if (room.empty()) { conn->send(err_json("ต้องระบุ room")); return; }
        server.send_history_to(conn, room, limit);
        return;
    }

    conn->send(err_json("ไม่รู้จัก type: " + type));
}

void handle_tcp_client(ChatServer& server, int fd) {
    auto conn = std::make_shared<TcpConnection>(fd);
    std::string buffer;
    char chunk[4096];

    for (;;) {
        ssize_t n = recv(fd, chunk, sizeof(chunk), 0);
        if (n <= 0) break;
        buffer.append(chunk, static_cast<size_t>(n));

        size_t pos;
        while ((pos = buffer.find('\n')) != std::string::npos) {
            std::string line = buffer.substr(0, pos);
            buffer.erase(0, pos + 1);
            if (line.empty()) continue;
            try {
                json msg = json::parse(line);
                handle_client_message(server, conn, msg);
            } catch (const json::parse_error& e) {
                conn->send(err_json(std::string("invalid JSON: ") + e.what()));
            }
        }
    }

    server.logout(conn);
}

void run_tcp_server(ChatServer& server, int port) {
    int server_fd = socket(AF_INET, SOCK_STREAM, 0);
    int opt = 1;
    setsockopt(server_fd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));

    sockaddr_in addr{};
    addr.sin_family = AF_INET;
    addr.sin_addr.s_addr = INADDR_ANY;
    addr.sin_port = htons(static_cast<uint16_t>(port));

    if (bind(server_fd, reinterpret_cast<sockaddr*>(&addr), sizeof(addr)) < 0) {
        perror("[tcp] bind");
        return;
    }
    if (listen(server_fd, 128) < 0) {
        perror("[tcp] listen");
        return;
    }
    std::cout << "[tcp] กำลังฟังที่ port " << port << std::endl;

    for (;;) {
        sockaddr_in client_addr{};
        socklen_t len = sizeof(client_addr);
        int client_fd = accept(server_fd, reinterpret_cast<sockaddr*>(&client_addr), &len);
        if (client_fd < 0) {
            if (errno == EINTR) continue;
            break;
        }
        std::thread(handle_tcp_client, std::ref(server), client_fd).detach();
    }
    close(server_fd);
}

int main(int argc, char** argv) {
    std::string db_path = (argc > 1) ? argv[1] : "chat_history.db";

    signal(SIGPIPE, SIG_IGN);

    ChatServer server(db_path);
    std::cout << "=== Capstone 2: Real-time Multiplayer Chat Server ===" << std::endl;
    std::cout << "ฐานข้อมูลประวัติแชท: " << db_path << std::endl;

    std::thread tcp_thread(run_tcp_server, std::ref(server), TCP_PORT);
    tcp_thread.detach();

    std::thread heartbeat_thread([&server]() {
        for (;;) {
            std::this_thread::sleep_for(HEARTBEAT_INTERVAL);
            server.heartbeat_tick();
        }
    });
    heartbeat_thread.detach();

    crow::SimpleApp app;
    std::mutex ws_map_mutex;
    std::unordered_map<crow::websocket::connection*, ConnPtr> ws_conns;

    CROW_ROUTE(app, "/")
    ([]() {
        return "Capstone Chat Server: WebSocket ws://<host>:7101/ws, Raw TCP <host>:7100";
    });

    CROW_WEBSOCKET_ROUTE(app, "/ws")
        .onopen([&](crow::websocket::connection& c) {
            auto conn = std::make_shared<WsConnection>(c);
            std::lock_guard<std::mutex> lock(ws_map_mutex);
            ws_conns[&c] = conn;
        })
        .onmessage([&](crow::websocket::connection& c, const std::string& data, bool is_binary) {
            if (is_binary) return;
            ConnPtr conn;
            {
                std::lock_guard<std::mutex> lock(ws_map_mutex);
                auto it = ws_conns.find(&c);
                if (it == ws_conns.end()) return;
                conn = it->second;
            }
            try {
                json msg = json::parse(data);
                handle_client_message(server, conn, msg);
            } catch (const json::parse_error& e) {
                conn->send(err_json(std::string("invalid JSON: ") + e.what()));
            }
        })
        .onclose([&](crow::websocket::connection& c, const std::string&, uint16_t) {
            ConnPtr conn;
            {
                std::lock_guard<std::mutex> lock(ws_map_mutex);
                auto it = ws_conns.find(&c);
                if (it == ws_conns.end()) return;
                conn = it->second;
                ws_conns.erase(it);
            }
            server.logout(conn);
        });

    app.port(WS_PORT).multithreaded().run();
    return 0;
}
```

---

## 122.6 คอมไพล์ รันจริง และทดสอบ Cross-Transport Chat (TCP ↔ WebSocket ↔ websocat) (Step 974)

### คอมไพล์ด้วย g++

```bash
$ g++ -std=c++17 -Wall -Wextra -Wpedantic -I/usr/local/include \
  chat_server.cpp -o chat_server -lpthread -lsqlite3
$
```

คอมไพล์ผ่านสะอาด **ไม่มี warning แม้แต่บรรทัดเดียว** (ตรวจสอบ exit code แล้วเป็น 0)

### หรือคอมไพล์ด้วย CMake (ทบทวนจาก Part 91)

สำหรับโปรเจกต์ที่จะขยายเป็นหลายไฟล์ในอนาคต (เช่นแยก `ChatServer` ออกเป็น `.hpp`/`.cpp`)
การตั้ง CMake ไว้ตั้งแต่ต้นสะดวกกว่า:

```cmake
cmake_minimum_required(VERSION 3.16)
project(capstone_chat_server CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

find_package(Threads REQUIRED)
find_path(CROW_INCLUDE_DIR crow.h PATHS /usr/local/include)
find_library(SQLITE3_LIB sqlite3 REQUIRED)

add_executable(chat_server chat_server.cpp)
target_include_directories(chat_server PRIVATE ${CROW_INCLUDE_DIR})
target_link_libraries(chat_server PRIVATE Threads::Threads ${SQLITE3_LIB})
target_compile_options(chat_server PRIVATE -Wall -Wextra -Wpedantic)
```

ทดสอบจริง:

```bash
$ cmake -S . -B build
-- The CXX compiler identification is GNU 13.3.0
-- Found Threads: TRUE
-- Configuring done (0.3s)
-- Generating done (0.0s)
-- Build files have been written to: .../cmake_build/build

$ cmake --build build
[ 50%] Building CXX object CMakeFiles/chat_server.dir/chat_server.cpp.o
[100%] Linking CXX executable chat_server
[100%] Built target chat_server
```

Build สำเร็จและได้ไฟล์ executable ตัวเดียวกันกับที่คอมไพล์ด้วย `g++` ตรงๆ ทุกประการ — จากนี้
ไปในบทเรียนจะใช้คำสั่ง `g++` ตรงๆ เพื่อความกระชับ แต่ใครที่ต้องการจัดโครงสร้างโปรเจกต์ให้เป็น
ระเบียบมากขึ้นสามารถใช้ CMake ไฟล์ข้างต้นได้ทันที

### รันเซิร์ฟเวอร์จริง

```bash
$ ./chat_server
=== Capstone 2: Real-time Multiplayer Chat Server ===
ฐานข้อมูลประวัติแชท: chat_history.db
[tcp] กำลังฟังที่ port 7100
(2026-09-26 12:26:57) [INFO    ] Crow/master server is running at http://0.0.0.0:7101 using 4 threads
(2026-09-26 12:26:57) [INFO    ] Call `app.loglevel(crow::LogLevel::Warning)` to hide Info level logs.
```

Server เดียวเปิดพร้อมกัน 2 ports: **7100 สำหรับ raw TCP** (แบบ Part 35) และ **7101 สำหรับ
WebSocket** (แบบ Part 109) — ทั้งสอง endpoint แชร์ `ChatServer` instance เดียวกัน

### ทดสอบ 1: TCP Client คุยกับ TCP Client (join, broadcast, history, presence, ping)

เขียน Python script สั้นๆ เป็น TCP client ทดสอบ (ใช้ raw `socket` module ไม่ต้องพึ่ง library
พิเศษใดๆ เพราะโปรโตคอลของเราเป็นแค่ JSON บรรทัดต่อบรรทัดผ่าน TCP ธรรมดา):

```bash
$ python3 tcp_client.py 127.0.0.1 7100 alice general "Hello from TCP Alice" 2 &
$ python3 tcp_client.py 127.0.0.1 7100 bob general "Hi from TCP Bob" 1
```

ผลลัพธ์จริง:

```
[alice/tcp] <- {"type":"welcome","username":"alice"}
[alice/tcp] <- {"members":["alice"],"room":"general","type":"joined"}
[alice/tcp] <- {"messages":[],"room":"general","type":"history"}
[bob/tcp] <- {"type":"welcome","username":"bob"}
[bob/tcp] <- {"members":["alice","bob"],"room":"general","type":"joined"}
[bob/tcp] <- {"messages":[{"from":"alice","text":"Hello from TCP Alice","ts":1790425551}],"room":"general","type":"history"}
[alice/tcp] <- {"from":"alice","room":"general","text":"Hello from TCP Alice","ts":1790425551,"type":"message"}
[alice/tcp] <- {"event":"join","room":"general","type":"presence","username":"bob"}
[alice/tcp] <- {"from":"bob","room":"general","text":"Hi from TCP Bob","ts":1790425551,"type":"message"}
[bob/tcp] <- {"from":"bob","room":"general","text":"Hi from TCP Bob","ts":1790425551,"type":"message"}
[alice/tcp] <- {"type":"ping"}
[bob/tcp] <- {"type":"ping"}
[bob/tcp] <- {"event":"leave","room":"general","type":"presence","username":"alice"}
```

สังเกตสิ่งที่พิสูจน์ได้จากผลลัพธ์นี้: (1) alice join ก่อน bob จึงเห็น history ว่างเปล่า
(2) bob join ทีหลังจึงเห็น history มีข้อความของ alice อยู่แล้ว — พิสูจน์ requirement
"client ที่เพิ่ง join ดึงประวัติล่าสุดได้ทันที" (3) alice เห็นข้อความของตัวเอง (echo กลับ)
เพื่อ confirm ว่าส่งสำเร็จ (4) ทั้งคู่ได้รับ `ping` จาก heartbeat thread ระหว่างทดสอบ
(5) เมื่อ alice ปิด connection bob เห็น presence `leave` ทันที

### ทดสอบ 2: Cross-Transport — WebSocket คุยกับ TCP ในห้องเดียวกันจริง

นี่คือการพิสูจน์หัวใจของ Part นี้: client สองคนที่ต่อผ่านคนละ transport กันสามารถแชทกันได้
จริงในห้องเดียวกัน — Carol ต่อผ่าน **WebSocket** (Python `websockets` library), Dave ต่อผ่าน
**raw TCP** (Python `socket` module ตรงๆ):

```bash
$ python3 ws_client.py ws://127.0.0.1:7101/ws carol general "" 3.5 &
$ python3 tcp_client.py 127.0.0.1 7100 dave general "สวัสดี Carol จากฝั่ง TCP!" 1.5
```

ผลลัพธ์ฝั่ง Carol (WebSocket):

```
[carol/ws] <- {"type":"welcome","username":"carol"}
[carol/ws] <- {"members":["carol"],"room":"general","type":"joined"}
[carol/ws] <- {"messages":[],"room":"general","type":"history"}
[carol/ws] <- {"type":"ping"}
[carol/ws] <- {"event":"join","room":"general","type":"presence","username":"dave"}
[carol/ws] <- {"from":"dave","room":"general","text":"สวัสดี Carol จากฝั่ง TCP!","ts":1790425629,"type":"message"}
[carol/ws] <- {"type":"ping"}
```

ผลลัพธ์ฝั่ง Dave (raw TCP):

```
[dave/tcp] <- {"type":"welcome","username":"dave"}
[dave/tcp] <- {"members":["carol","dave"],"room":"general","type":"joined"}
[dave/tcp] <- {"messages":[],"room":"general","type":"history"}
[dave/tcp] <- {"from":"dave","room":"general","text":"สวัสดี Carol จากฝั่ง TCP!","ts":1790425629,"type":"message"}
[dave/tcp] <- {"type":"ping"}
[dave/tcp] <- {"event":"leave","room":"general","type":"presence","username":"carol"}
```

ข้อความของ Dave (TCP) ถูกส่งไปถึง Carol (WebSocket) จริง ด้วย `ts` ตรงกันเป๊ะ
(`1790425629`) — พิสูจน์ว่า `ChatServer` มองไม่เห็นความแตกต่างระหว่างสอง transport เลย ตรง
ตามที่การออกแบบ `IConnection` ตั้งใจไว้

### ทดสอบ 3: ยืนยันด้วย `websocat` จริง (ไม่ใช่แค่ library ของ Python)

เพื่อความมั่นใจว่าระบบใช้งานได้กับ WebSocket client ตัวจริงทั่วไป ไม่ใช่แค่ library เฉพาะ
ทดสอบซ้ำด้วย `websocat` (ติดตั้งผ่าน `cargo install websocat` เหมือนใน Part 109):

```bash
$ ( sleep 0.4; printf '{"type":"login","username":"wanda"}\n{"type":"join","room":"general"}\n'; sleep 1.5 ) \
  | websocat ws://127.0.0.1:7101/ws &
$ python3 tcp_client.py 127.0.0.1 7100 zack general "สวัสดี Wanda จาก websocat test!" 1
```

ผลลัพธ์ที่ wanda (websocat) ได้รับจริง:

```
{"type":"welcome","username":"wanda"}
{"members":["wanda"],"room":"general","type":"joined"}
{"messages":[],"room":"general","type":"history"}
{"event":"join","room":"general","type":"presence","username":"zack"}
{"from":"zack","room":"general","text":"สวัสดี Wanda จาก websocat test!","ts":1790426643,"type":"message"}
{"type":"ping"}
```

ยืนยันอีกครั้งด้วยเครื่องมือคนละตัว: raw TCP client (zack) กับ WebSocket client ตัวจริงผ่าน
`websocat` (wanda) คุยกันได้ในห้องเดียวกันสมบูรณ์

### ทดสอบ 4: Feature Demo — Duplicate Login, Error Handling, List Rooms, DM

```python
# --- 1) ชื่อผู้ใช้ซ้ำต้องถูกปฏิเสธ ---
await w1.send(json.dumps({"type": "login", "username": "eve"}))
await w2.send(json.dumps({"type": "login", "username": "eve"}))

# --- 2) ส่งข้อความเข้าห้องที่ยังไม่ได้ join ---
await w1.send(json.dumps({"type": "message", "room": "random", "text": "hi"}))

# --- 3) join แล้วขอ list_rooms ---
await w1.send(json.dumps({"type": "join", "room": "random"}))
await w1.send(json.dumps({"type": "list_rooms"}))

# --- 4) DM ระหว่างผู้ใช้ และ DM ไปหาคนที่ไม่มีตัวตน ---
await a.send(json.dumps({"type": "dm", "to": "grace", "text": "แอบส่งข้อความส่วนตัวนะ"}))
await a.send(json.dumps({"type": "dm", "to": "nobody", "text": "hello?"}))
```

ผลลัพธ์จริงทั้งหมด:

```
w1 login -> {"type":"welcome","username":"eve"}
w2 login (duplicate name) -> {"message":"ชื่อผู้ใช้ 'eve' ถูกใช้งานอยู่แล้ว","type":"error"}
w1 send to unjoined room -> {"message":"คุณยังไม่ได้เข้าร่วมห้อง 'random'","type":"error"}
w1 joined -> {"members":["eve"],"room":"random","type":"joined"}
w1 history -> {"messages":[],"room":"random","type":"history"}
w1 list_rooms -> {"rooms":[{"members":1,"room":"random"},{"members":0,"room":"general"}],"type":"room_list"}
frank dm echo -> {"from":"frank","text":"แอบส่งข้อความส่วนตัวนะ","to":"grace","ts":1790425646,"type":"dm"}
grace received dm -> {"from":"frank","text":"แอบส่งข้อความส่วนตัวนะ","to":"grace","ts":1790425646,"type":"dm"}
dm to nonexistent user -> {"message":"ไม่พบผู้ใช้ 'nobody' ในระบบขณะนี้","type":"error"}
```

ทุก error case ทำงานตามที่ออกแบบไว้ครบถ้วน — สังเกตว่า `list_rooms` แสดงห้อง `general`
ที่มีสมาชิก 0 คน (เพราะ `rooms_` ไม่ลบ entry ของห้องทิ้งแม้จะว่างเปล่า เพื่อให้ห้องที่เคย
ถูกสร้างยังคงอยู่ในรายการเสมอ)

### ทดสอบ 5: ประวัติแชทอยู่รอดข้าม Server Restart จริง

นี่คือการพิสูจน์ requirement ที่สำคัญที่สุดข้อหนึ่ง: "ประวัติแชทต้องอยู่รอดแม้ server จะถูก
restart" (ไม่ใช่แค่อยู่รอดระหว่างที่ process เดิมยังรันอยู่)

```bash
# ขั้นที่ 1: มี dave ส่งข้อความเข้าห้อง general แล้ว server ยังรันอยู่
$ python3 -c "
import sqlite3
con = sqlite3.connect('chat_history.db')
for row in con.execute('SELECT room, sender, body, ts FROM messages ORDER BY id;'):
    print(row)
"
('general', 'dave', 'สวัสดี Carol จากฝั่ง TCP!', 1790425629)

# ขั้นที่ 2: ปิด server (kill process) แล้วเปิดใหม่จาก process ID ใหม่ทั้งหมด
$ pkill -f "./chat_server"
$ ./chat_server
=== Capstone 2: Real-time Multiplayer Chat Server ===
ฐานข้อมูลประวัติแชท: chat_history.db
[tcp] กำลังฟังที่ port 7100
(2026-09-26 12:27:53) [INFO    ] Crow/master server is running at http://0.0.0.0:7101 using 4 threads

# ขั้นที่ 3: client ใหม่ (heidi) join ห้อง general เป็นคนแรกใน process ใหม่นี้
$ python3 -c "
import asyncio, websockets, json
async def main():
    async with websockets.connect('ws://127.0.0.1:7101/ws') as ws:
        await ws.send(json.dumps({'type':'login','username':'heidi'}))
        print(await ws.recv())
        await ws.send(json.dumps({'type':'join','room':'general'}))
        print(await ws.recv())
        print(await ws.recv())  # history
asyncio.run(main())
"
```

ผลลัพธ์จริง:

```
{"type":"welcome","username":"heidi"}
{"members":["heidi"],"room":"general","type":"joined"}
{"messages":[{"from":"dave","text":"สวัสดี Carol จากฝั่ง TCP!","ts":1790425629}],"room":"general","type":"history"}
```

Heidi เป็นคนแรกที่ join ห้อง `general` ใน **process ใหม่ทั้งหมด** (server ถูกฆ่าและเปิดใหม่
ระหว่างนั้น ไม่มี state ใดๆ ในหน่วยความจำเหลืออยู่เลย) แต่ยังเห็นข้อความของ dave จาก
**process เดิม** ได้ครบถ้วน — เพราะมันถูกเก็บลง `chat_history.db` บนดิสก์จริง ไม่ใช่แค่
ในหน่วยความจำ

---

## 122.7 Heartbeat ตรวจจับ Connection ตาย, Load Test พิสูจน์ Concurrency, และ ThreadSanitizer (Step 975)

### ทดสอบ Heartbeat: แยกแยะ Client "เงียบ" กับ Client "ปกติ" ได้จริง

โจทย์คือ: server ต้องตรวจจับ client ที่ connection ยังไม่ขาดในระดับ TCP (ไม่มี FIN/RST ส่งมา)
แต่ไม่ตอบสนองอะไรเลย (เช่น แอปค้าง, เบราว์เซอร์ถูก suspend) แล้วบังคับตัดทิ้งเพื่อคืน
ทรัพยากร โดย**ไม่กระทบ client คนอื่นที่ยังทำงานปกติ**

เขียน client จำลอง 2 แบบ: `stubborn` (รับข้อความปกติแต่**ไม่เคย**ตอบ `pong` กลับเมื่อได้รับ
`ping`) และ `goodcitizen` (ตอบ `pong` ทันทีทุกครั้งที่ได้รับ `ping`) ให้ทั้งคู่เชื่อมต่อพร้อมกัน
เข้าห้องเดียวกัน:

```python
# stubborn_client.py — จงใจไม่ตอบ pong เลย
async for msg in websocket_messages():
    print("stubborn received (แต่จะไม่ตอบ pong):", msg)

# good_client.py — ตอบ pong ทุกครั้ง
async for msg in websocket_messages():
    if json.loads(msg).get("type") == "ping":
        await ws.send(json.dumps({"type": "pong"}))
```

ผลลัพธ์จริงฝั่ง `stubborn`:

```
login -> {"type":"welcome","username":"stubborn"}
joined -> {"members":["stubborn"],"room":"general","type":"joined"}
history -> {"messages":[...],"room":"general","type":"history"}
stubborn received (แต่จะไม่ตอบ pong): {"event":"join","room":"general","type":"presence","username":"goodcitizen"}
stubborn received (แต่จะไม่ตอบ pong): {"type":"ping"}
stubborn received (แต่จะไม่ตอบ pong): {"type":"ping"}
connection ended: ConnectionClosedOK(Close(code=1001, reason='heartbeat timeout'), Close(code=1001, reason='heartbeat timeout'), True)
```

ผลลัพธ์จริงฝั่ง `goodcitizen`:

```
login -> {"type":"welcome","username":"goodcitizen"}
joined -> {"members":["goodcitizen","stubborn"],"room":"general","type":"joined"}
goodcitizen received: {"event":"join","room":"general","type":"presence","username":"stubborn"}
goodcitizen received: {"messages":[...],"room":"general","type":"history"}
goodcitizen ได้รับ ping -> ตอบ pong กลับทันที
goodcitizen ได้รับ ping -> ตอบ pong กลับทันที
goodcitizen ได้รับ ping -> ตอบ pong กลับทันที
goodcitizen received: {"event":"leave","room":"general","type":"presence","username":"stubborn"}
goodcitizen ได้รับ ping -> ตอบ pong กลับทันที
goodcitizen ได้รับ ping -> ตอบ pong กลับทันที
goodcitizen จบการทดสอบโดยยังเชื่อมต่ออยู่ปกติ (ไม่ถูกบังคับตัด)
```

และ server log ยืนยันตรงกันทุกจุด:

```
[login] goodcitizen (transport=ws)
[login] stubborn (transport=ws)
[+] stubborn เข้าร่วมห้อง 'general' (สมาชิกทั้งหมด 1 คน)
[+] goodcitizen เข้าร่วมห้อง 'general' (สมาชิกทั้งหมด 2 คน)
[heartbeat] stubborn ไม่ตอบ pong ภายใน 8 วินาที บังคับตัดการเชื่อมต่อ (transport=ws)
[-] stubborn ออกจากระบบ (transport=ws)
[-] goodcitizen ออกจากระบบ (transport=ws)
```

**stubborn ถูกตัดการเชื่อมต่อด้วย close code 1001 หลังเงียบไปครบ `HEARTBEAT_TIMEOUT` (8
วินาที) พอดี** ในขณะที่ **goodcitizen ที่ตอบ pong ทุกครั้งไม่ถูกแตะต้องเลยแม้แต่ครั้งเดียว**
ตลอด 5 รอบ heartbeat — พิสูจน์ว่า heartbeat mechanism ทำงานถูกต้องแม่นยำ และไม่มีผลกระทบต่อ
client ที่ยังทำงานปกติ

> **หมายเหตุ**: ค่า `HEARTBEAT_INTERVAL`/`HEARTBEAT_TIMEOUT` ตั้งไว้สั้นมาก (3/8 วินาที)
> เพื่อให้สาธิตและทดสอบได้เร็วในบทเรียนนี้เท่านั้น ระบบจริงควรตั้งไว้ยาวกว่านี้มาก (เช่น
> interval 30 วินาที, timeout 90 วินาที) เพื่อไม่ให้ตัด connection ที่แค่มี latency สูงชั่วคราว
> ทิ้งโดยไม่จำเป็น

### Load Test: พิสูจน์ความถูกต้องภายใต้ Concurrency สูงด้วยตัวเลขจริง

หัวใจของการพิสูจน์ "ไม่มีข้อความขาด ซ้ำ หรือหลุดข้ามห้อง" คือสคริปต์ `load_test.py` ที่:

1. สร้าง client จำนวนมาก กระจายเข้า **8 ห้อง** ห้องละ **15 client**
2. รอจน**สมาชิกครบทุกคนในห้อง join เสร็จจริง** (ใช้ `asyncio.Event` เป็น barrier) ก่อนเริ่มส่ง
   ข้อความ — ป้องกันไม่ให้นับ "ข้อความที่ส่งก่อนผู้รับจะ join ทัน" เป็นข้อความที่หายไปผิดๆ
3. แต่ละ client ส่งข้อความ **20 ข้อความ** ที่มีเลข sequence กำกับชัดเจน (`username:seq`)
4. ทุก client (รวมทั้งผู้ส่งเองที่ได้รับ echo กลับ) นับว่าได้รับข้อความของทุกคนในห้องครบไหม
   มีข้อความจากห้องอื่นหลุดเข้ามาไหม และมีข้อความซ้ำไหม

```python
async def sender():
    await room_ready_events[room].wait()  # รอสมาชิกครบห้องก่อนเริ่มยิงข้อความจริง
    for seq in range(MESSAGES_PER_CLIENT):
        await ws.send(json.dumps({"type": "message", "room": room, "text": f"{username}:{seq}"}))
        await asyncio.sleep(0.005)

async def receiver(stop_after):
    ...
    if data["room"] != room:
        cross_room_leaks.append((username, room, data["room"], data))
        continue
    sender_name, seq = data["text"].split(":")
    key = (room, sender_name, int(seq))
    if key in seen_local:
        per_client_dupes.append((username, key))
    seen_local.add(key)
    received_counter[key] += 1
```

รันจริงกับ server ที่กำลังทำงานอยู่ (120 concurrent WebSocket client พร้อมกัน):

```bash
$ python3 load_test.py
เวลาที่ใช้ทดสอบทั้งหมด: 15.11 วินาที
จำนวนห้อง: 8, client ต่อห้อง: 15, ข้อความต่อ client: 20
ข้อความที่ถูกส่งทั้งหมด (unique message): 2400
จำนวนครั้งที่ควรถูกส่งมอบทั้งหมด (ทุกคนในห้อง x ทุกข้อความ): 36000
จำนวนครั้งที่ถูกส่งมอบจริง (นับจากฝั่งผู้รับทั้งหมด): 36000
ข้อความที่ 'ข้ามห้อง' หลุดเข้ามาผิดห้อง: 0 รายการ
ข้อความที่ client คนเดียวกันเห็น 'ซ้ำ' (duplicate ในฝั่งเดียว): 0 รายการ
ข้อความที่ 'ขาดหาย' (ได้รับน้อยกว่าจำนวนสมาชิกในห้อง): 0 รายการ
ข้อความที่ 'เกิน' จำนวนสมาชิกในห้อง (broadcast ซ้ำ): 0 รายการ
ผลสรุป: PASS — ไม่มีข้อความขาด/ซ้ำ/ข้ามห้องเลยแม้แต่รายการเดียว
```

**36,000 การส่งมอบข้อความ ถูกต้อง 100% ทั้งหมด** — ไม่มีข้อความขาดหาย ไม่มีข้อความซ้ำ และ
ไม่มีข้อความหลุดข้ามห้องแม้แต่รายการเดียว ภายใต้ 120 concurrent client ที่ยิงพร้อมกันจริง

> **หมายเหตุจากการพัฒนาจริง**: รอบทดสอบแรกที่เขียนสคริปต์นี้ (ก่อนใส่ barrier
> `room_ready_events`) รายงานว่ามีข้อความ "ขาดหาย" 17 รายการจาก 6,000 การส่งมอบ ตอนแรก
> ดูน่าตกใจมาก แต่เมื่อตรวจสอบแล้วพบว่า**ไม่ใช่บั๊กของ server เลย** — เป็นเพราะ client ที่
> join เสร็จเร็วเริ่มส่งข้อความก่อนที่ client คนอื่นในห้องเดียวกันจะ join ทัน ทำให้ข้อความ
> ช่วงแรกๆ ถูกส่งไปตอนที่ผู้รับยังไม่ได้เป็นสมาชิกจริงๆ (ซึ่งเป็นพฤติกรรมที่ถูกต้องแล้ว
> ของ server — ห้ามส่งข้อความให้คนที่ยังไม่ join) หลังแก้สคริปต์ทดสอบให้รอสมาชิกครบก่อน
> เริ่มส่งข้อความ ผลลัพธ์กลายเป็น 100% ถูกต้องทันที **นี่คือบทเรียนสำคัญ**: ตัวเลขที่ผิดจาก
> การทดสอบไม่ได้แปลว่า production code ผิดเสมอไป — ต้องตรวจสอบ test harness เองด้วยเสมอ
> ก่อนสรุปว่าเป็นบั๊กของระบบ

### ThreadSanitizer: พิสูจน์ Thread-safety ของ Registry จริงภายใต้ Mixed-Transport Stress Test

คอมไพล์ด้วย `-fsanitize=thread` (เหมือน Part 82/109):

```bash
$ g++ -std=c++17 -Wall -Wextra -fsanitize=thread -g -O1 -I/usr/local/include \
  chat_server.cpp -o chat_server_tsan -lpthread -lsqlite3
```

คอมไพล์ผ่านสำเร็จ (0 errors) — มี warning ระดับ `-Wtsan` จาก header ของ `asio` ที่ Crow ใช้
ภายใน (`atomic_thread_fence` ไม่รองรับเต็มรูปแบบภายใต้ TSan) ซึ่งเป็นเรื่องปกติของไลบรารี
ภายนอก ไม่ใช่ปัญหาของโค้ดเรา:

```
warning: 'atomic_thread_fence' is not supported with '-fsanitize=thread' [-Wtsan]
```

รัน server ตัวที่คอมไพล์ด้วย TSan แล้วยิง **`tsan_stress.py`** — สคริปต์ที่จงใจยิงโหลด
ผสมทุกรูปแบบพร้อมกันเพื่อบีบให้เกิด race มากที่สุดเท่าที่จะเป็นไปได้:

- **WebSocket chat client** 18 ตัว (6 ตัวต่อห้อง × 3 ห้อง) ส่งข้อความ 12 ข้อความ/ตัว
- **Raw TCP chat client** 18 ตัว (6 ตัวต่อห้อง × 3 ห้อง) ส่งข้อความ 12 ข้อความ/ตัว
- **Churn client** 10 ตัว ที่ join/leave ห้องสลับไปมาเร็วๆ ตลอด 5 วินาที (บีบให้
  `rooms_`/`user_rooms_` ถูกแก้ไขถี่ที่สุด)
- **DM client** 8 ตัว ที่ส่ง direct message หากันสลับคู่ตลอดเวลา

รวม **54 concurrent connection ผสมทั้งสอง transport** ยิงพร้อมกันจริง:

```bash
$ python3 tsan_stress.py
กำลังยิง 54 concurrent client (WS chat + TCP chat + churn + DM) ...
เสร็จสิ้นภายใน 6.36 วินาที
```

**ผลลัพธ์จาก ThreadSanitizer (คัดลอกจริง ไม่ปรุงแต่ง):**

```
WARNING: ThreadSanitizer: data race (pid=9856)
  Write of size 8 at 0x7248000008c8 by thread T5:
    #0 crow::WebSocketRule<crow::Crow<> >::handle_upgrade(...) routing.h:446
    #1 void crow::Router::handle_upgrade<crow::SocketAdaptor&>(...) routing.h:1527
    #2 void crow::Crow<>::handle_upgrade<crow::SocketAdaptor>(...) app.h:268
    ...
SUMMARY: ThreadSanitizer: data race /usr/local/include/crow/routing.h:446 in
  crow::WebSocketRule<crow::Crow<> >::handle_upgrade(...)

WARNING: ThreadSanitizer: data race (pid=9856)
  ... (เหตุการณ์เดียวกัน เกิดซ้ำอีกครั้งจาก handshake ครั้งถัดไป)
SUMMARY: ThreadSanitizer: data race /usr/local/include/crow/routing.h:446 in
  crow::WebSocketRule<crow::Crow<> >::handle_upgrade(...)
```

**พบ data race แค่ 2 รายการเท่านั้นตลอดการทดสอบทั้งหมด และทั้งคู่ชี้ไปที่จุดเดียวกันเป๊ะ:**
`crow/routing.h:446` ในเมธอด `handle_upgrade()` ของ Crow เอง ตรวจสอบ source บรรทัดนั้นแล้วพบว่า
คือบรรทัด `max_payload_ = max_payload_override_ ? max_payload_ : app_->websocket_max_payload();`
— การเขียนทับตัวแปรสมาชิก `max_payload_` ของ **route object ตัวเดียวที่ใช้ร่วมกันทุก
connection** ตอนทำ WebSocket handshake พร้อมกันหลาย connection คือ**โค้ดภายในของ Crow เอง**
ไม่เกี่ยวข้องกับ `users_`, `rooms_`, `user_rooms_`, `ws_conns`, หรือ `db_` ที่เราเขียนเองแม้แต่
น้อย — และเป็น**ปัญหาเดียวกันเป๊ะ**กับที่ Part 109 หัวข้อ 109.7 เคยพบมาแล้วตอนทดสอบ
`ws_chat_server.cpp` ธรรมดา (ไม่ได้เกี่ยวกับความซับซ้อนที่เพิ่มขึ้นใน Part นี้เลย)

```bash
$ grep -c "chat_server.cpp" tsan_server.log     # เช็คว่ามี stack frame ไหนชี้เข้าโค้ดเราไหม
5   # ทั้งหมดเป็นแค่ stack frame ของ main()/CROW_WEBSOCKET_ROUTE()/app.run() (entry point)
    # ไม่มีจุดไหนเลยที่ตัว "Read/Write" จริงเกิดขึ้นในโค้ดแอปพลิเคชันของเรา
```

**สรุปผล TSan ของโค้ดที่เราเขียนเอง: 0 data race** ตลอดการทดสอบที่ผสมทั้ง TCP, WebSocket,
room churn, และ DM พร้อมกัน 54 connection — ตรงตาม requirement "thread-safe global registry
... verified with ThreadSanitizer against a concurrent stress test" ทุกประการ

### เปรียบเทียบ: ถ้าตัด Mutex ออกจาก Registry จะเกิดอะไรขึ้น

เพื่อให้เห็นภาพชัดว่า `mutex_` ที่เราใส่ไว้มีความหมายจริง ลองสร้างเวอร์ชันจำลองที่ตัด
`std::mutex` ออกทั้งหมดจาก registry ที่มีโครงสร้างเดียวกัน (`users_`/`rooms_` แบบเดียวกัน)
โดยจงใจไม่ล็อกอะไรเลย:

```cpp
// registry_unsafe_demo.cpp — สาธิต "ทำไมต้องมี mutex_" จาก ChatServer จริง
// เวอร์ชันนี้ตัด lock_guard ออกจากทุกจุดที่แตะ users_/rooms_ โดยตั้งใจ
#include <iostream>
#include <set>
#include <string>
#include <thread>
#include <unordered_map>
#include <vector>

std::unordered_map<std::string, int> users_;
std::unordered_map<std::string, std::set<std::string>> rooms_;

void join_room(const std::string& username, int fd, const std::string& room) {
    users_[username] = fd;
    rooms_[room].insert(username);
}
void broadcast(const std::string& room) {
    for (const auto& username : rooms_[room]) {
        volatile int fd = users_[username];
        (void)fd;
    }
}
void leave_room(const std::string& username, const std::string& room) {
    rooms_[room].erase(username);
    users_.erase(username);
}

int main() {
    constexpr int N = 40;
    std::vector<std::thread> threads;
    for (int i = 0; i < N; ++i) {
        std::string uname = "user" + std::to_string(i);
        threads.emplace_back(join_room, uname, i, "general");
        threads.emplace_back(broadcast, "general");
        threads.emplace_back(leave_room, uname, "general");
    }
    for (auto& t : threads) t.join();
    std::cout << "จบการทำงาน\n";
    return 0;
}
```

```bash
$ g++ -std=c++17 -Wall -Wextra -fsanitize=thread -g -O1 registry_unsafe_demo.cpp \
  -o registry_unsafe_demo -lpthread
$ ./registry_unsafe_demo
```

**ผลลัพธ์จริง: ThreadSanitizer รายงาน data race 38 รายการ** กระจายอยู่ทั่วภายในของ
`std::unordered_map`/`std::set` (`hashtable.h`, `stl_tree.h`) รวมถึงชี้ตรงมาที่โค้ดของเราเอง
โดยตรงด้วย:

```
SUMMARY: ThreadSanitizer: data race registry_unsafe_demo.cpp:22 in broadcast(std::string const&)
ThreadSanitizer: reported 38 warnings
```

(บรรทัด 22 คือ `volatile int fd = users_[username];` ใน `broadcast()` — TSan จับได้ตรงเป๊ะว่า
มี thread อื่นกำลังเขียน `users_` พร้อมกันในเวลาเดียวกันจริง) เทียบกับ `chat_server.cpp` ตัวจริง
ที่ผ่าน stress test หนักกว่านี้มากโดยไม่มี data race ในโค้ดแอปเลยแม้แต่จุดเดียว — **มี `mutex_`
กับ ไม่มี `mutex_` คือความต่างระหว่าง 0 กับ 38 data race**

---

## 122.8 ข้อผิดพลาดที่พบบ่อย (Common Pitfalls)

### ข้อ 1 (สำคัญที่สุด): ถือ Mutex ค้างไว้ระหว่างเรียก `send()` ที่อาจ Block — พิสูจน์ด้วยเวลาจริง

นี่คือ pitfall เฉพาะของระบบระดับนี้ที่ Part 35/109 ไม่เคยแสดงให้เห็นเป็นตัวเลขมาก่อน:
ถ้า broadcast loop **ถือ mutex ค้างไว้ตลอดขณะเรียก `send()`** และมี client สักตัวที่อ่านช้า
(หรือไม่อ่านเลย) จน kernel send buffer เต็ม `send()` จะ **block รอ** — และถ้าตอนนั้นยังถือ
mutex อยู่ **ทุก thread อื่นในระบบที่ต้องการ mutex ตัวเดียวกัน (ไม่ว่าจะเกี่ยวกับ client ที่
ค้างอยู่หรือไม่ก็ตาม) จะค้างตามไปด้วยหมดทั้งระบบ**

สร้างเซิร์ฟเวอร์ทดสอบขนาดเล็ก 2 เวอร์ชันที่ต่างกันแค่จุดเดียว:

```cpp
// เวอร์ชัน "ผิด" — ถือ lock ตลอดลูป broadcast รวมถึงตอน send()
void broadcast_payload(const std::string& payload) {
    std::lock_guard<std::mutex> lock(mtx);       // <-- ล็อกตรงนี้
    for (int fd : fds) {
        send(fd, payload.data(), payload.size(), MSG_NOSIGNAL);  // อาจ block ขณะยังถือ lock!
    }
}
```

```cpp
// เวอร์ชัน "ถูก" — สแนปช็อตแล้วปลดล็อกก่อนส่ง (เทคนิคเดียวกับ chat_server.cpp)
void broadcast_payload(const std::string& payload) {
    std::vector<int> snapshot;
    { std::lock_guard<std::mutex> lock(mtx); snapshot = fds; }   // ปลดล็อกตรงนี้
    for (int fd : snapshot) {
        send(fd, payload.data(), payload.size(), MSG_NOSIGNAL);  // ไม่มีใครถือ lock ระหว่างนี้
    }
}
```

ทดสอบจริง: เปิด client "ช้า" (connect แล้วไม่อ่านอะไรเลย) 1 ตัว, client "ยิงข้อความก้อนใหญ่
ถี่ๆ" 1 ตัว (จำลอง load), และ client "ยิง PING" 1 ตัวที่วัดเวลาว่ากว่าจะได้รับ PONG กลับใช้
เวลาเท่าไหร่ (PING ต้องใช้ mutex ตัวเดียวกันสั้นๆ เพื่ออ่านจำนวน client ปัจจุบัน):

```bash
$ python3 measure_stall.py 7300 UNSAFE     # เวอร์ชันถือ lock ตลอด send()
[UNSAFE] PING #0 -> TIMEOUT หลังรอ 8.000 วินาที (ไม่ได้รับ PONG เลย)
[UNSAFE] PING #1 -> TIMEOUT หลังรอ 8.000 วินาที (ไม่ได้รับ PONG เลย)
[UNSAFE] PING #2 -> TIMEOUT หลังรอ 8.000 วินาที (ไม่ได้รับ PONG เลย)
[UNSAFE] PING latency สูงสุดที่วัดได้: 8.000 วินาที

$ python3 measure_stall.py 7301 FIXED      # เวอร์ชันสแนปช็อตแล้วปลดล็อกก่อนส่ง
[FIXED] PING #0 -> ได้รับ PONG แล้ว ใช้เวลา 0.001 วินาที
[FIXED] PING #1 -> ได้รับ PONG แล้ว ใช้เวลา 0.000 วินาที
[FIXED] PING #2 -> ได้รับ PONG แล้ว ใช้เวลา 0.000 วินาที
[FIXED] PING latency สูงสุดที่วัดได้: 0.001 วินาที
```

**เวอร์ชันผิดค้างอยู่นานเกิน 8 วินาทีเต็ม (จริงๆ แล้วค้างต่อไปไม่มีกำหนดจนกว่า client ช้าจะ
ถูกบังคับปิดหรือ buffer ระบายออก) ในขณะที่เวอร์ชันถูกตอบกลับภายใน 1 มิลลิวินาที** — ต่างกัน
มากกว่า **8,000 เท่า** นี่คือเหตุผลที่ `broadcast_room()` ใน `chat_server.cpp` (122.3) ต้อง
สแนปช็อตแล้วปลดล็อกก่อนส่งเสมอ ไม่ใช่แค่ "ทฤษฎีที่ฟังดูดี" แต่เป็นตัวเลขที่วัดได้จริงและ
ต่างกันมหาศาล

### ข้อ 2: Iterator Invalidation ขณะ Broadcast ถ้า Client หลุดกลางคัน

ถ้า `broadcast_room()` วนลูปบน `rooms_[room]` (container จริง) โดยตรงแทนที่จะ copy ออกมาก่อน
แล้วมี thread อื่นเรียก `leave_room()`/`logout()` ที่แก้ไข `rooms_[room]` ตัวเดียวกันพร้อมกัน
(เช่น `erase()`) จะเกิด **iterator invalidation** — undefined behavior ที่อาจทำให้ crash หรือ
วนลูปข้ามสมาชิกบางคนไปเฉยๆ โดยไม่มี error ใดๆ เตือน `chat_server.cpp` หลีกเลี่ยงปัญหานี้
**โดยการออกแบบ ไม่ใช่โดยความระมัดระวัง**: `targets` เป็น `std::vector` ที่ copy ค่าออกมาแล้ว
ก่อนปลดล็อก จึงเป็นคนละ container จาก `rooms_[room]` ตัวจริงโดยสิ้นเชิง ไม่มีทางถูก invalidate
จากการแก้ไข `rooms_` ในภายหลังได้เลย

### ข้อ 3: ลืมป้องกัน Room-Membership Map ด้วย Mutex ตัวเดียวกับ User Registry

ข้อผิดพลาดที่ดูสมเหตุสมผลแต่อันตรายมาก: คิดว่า "`rooms_` กับ `users_` เป็นข้อมูลคนละเรื่อง
ใช้ mutex คนละตัวก็น่าจะเร็วกว่า" ปัญหาคือ `join_room()`/`leave_room()`/`broadcast_room()`
ต้องอ่าน/เขียนทั้งสอง container **พร้อมกันแบบ atomic** ถ้าใช้ mutex คนละตัว จะมีหน้าต่าง
เวลาสั้นๆ ที่ thread หนึ่งเห็น `rooms_` อัปเดตแล้วแต่ `users_`/`user_rooms_` ยังไม่ทัน (หรือ
สลับกัน) ทำให้เกิด **inconsistent snapshot** — เช่น `broadcast_room()` อาจเจอ username ใน
`rooms_[room]` ที่ไม่มีอยู่ใน `users_` แล้ว (เพราะเพิ่ง logout ไปพร้อมๆ กัน) บั๊กแบบนี้ไม่ทำให้
crash และไม่ถูก ThreadSanitizer จับ (เพราะแต่ละ container ถูกป้องกันด้วย mutex ของตัวเองอย่าง
ถูกต้องในทางเทคนิค) แต่ให้ผลลัพธ์ผิดแบบเงียบๆ ซึ่งตรวจจับยากกว่า data race ปกติมาก
`chat_server.cpp` แก้ปัญหานี้จากรากด้วยการใช้ **mutex เดียว (`mutex_`) คลุมทั้ง 4 container**
ที่เกี่ยวข้องกัน

### ข้อ 4: ถือ Mutex ตัวเดียวกันคลุมทั้ง In-Memory State และ Disk I/O

ถ้าใช้ `mutex_` ตัวเดียวกันคลุมทั้งการแก้ไข `rooms_`/`users_` **และ** การเขียนข้อความลง
SQLite (`persist_room_message()`) ทุก thread ที่แค่ต้องการ join/leave ห้องธรรมดาจะต้องรอ
thread ที่กำลังเขียนไฟล์ลงดิสก์อยู่ ซึ่งช้ากว่าการแก้ไข hash map ในหน่วยความจำหลายเท่าตัว
(disk I/O มี latency หลัก millisecond ขึ้นไป เทียบกับการแก้ hash map ที่เป็นหลัก nanosecond)
`chat_server.cpp` แยก `db_mutex_` ออกจาก `mutex_` อย่างชัดเจน ทำให้การเขียนประวัติลง SQLite
ไม่เคยไปบล็อกการ join/leave/broadcast ของ thread อื่นเลย

### ข้อ 5: fd-Reuse Race ตอน Disconnect (ปัญหาคลาสสิกจาก Part 35 ที่แก้ได้ด้วยโครงสร้าง)

Part 35.7 อธิบายว่าต้อง "ลบออกจาก client_list ก่อน แล้วค่อย `close()`" ด้วยมือ ไม่เช่นนั้น
OS อาจนำ fd หมายเลขเดิมไปใช้กับ connection ใหม่ก่อนที่ thread อื่นจะทันรู้ว่า fd นั้นไม่ใช่
เจ้าของเดิมแล้ว ใน `chat_server.cpp` ปัญหานี้แก้ด้วย **reference counting ผ่าน `shared_ptr`**
(`TcpConnection` เก็บ fd ไว้ และ `close()` เกิดขึ้นเฉพาะใน destructor) — ทำให้ fd ไม่มีทาง
ถูกปิดตราบใดที่ยังมี `shared_ptr` (เช่น snapshot ของ `broadcast_room()`) ถืออ้างอิงอยู่ ไม่ว่า
โปรแกรมเมอร์จะเขียนโค้ด cleanup ผิดลำดับแค่ไหนก็ตาม

### ข้อ 6: ลืม `MSG_NOSIGNAL`/`signal(SIGPIPE, SIG_IGN)` สำหรับ Raw TCP (ทบทวนจาก Part 35)

เหมือน Part 35.3 ทุกประการ: ถ้า client ฝั่ง raw TCP หลุดกะทันหันแล้ว server เรียก `send()`
ไปยัง socket นั้นอีกครั้งโดยไม่ป้องกันไว้ ค่า default ของ `SIGPIPE` คือฆ่าโปรแกรมทั้งตัวทันที
`chat_server.cpp` ป้องกันสองชั้นเหมือนเดิม: `signal(SIGPIPE, SIG_IGN)` ใน `main()` และ
`MSG_NOSIGNAL` ทุกครั้งที่เรียก `::send()` ใน `TcpConnection::send()`

### ข้อ 7: ไม่มีกลไกตรวจจับ Connection ที่ "เงียบ" แบบไม่มี FIN/RST

`recv()` คืนค่า `0`/`-1` (TCP) หรือ `onclose` ถูกเรียก (WebSocket) ครอบคลุมแค่กรณีที่
connection ปิดแบบที่ระดับ transport รู้ตัว แต่ถ้า network ขาดแบบเงียบๆ (เช่น เราเตอร์กลางทาง
ถูกถอดปลั๊กโดยไม่มีการส่ง FIN/RST ใดๆ) ทั้ง TCP และ WebSocket จะไม่รู้ตัวเลยว่า connection
ตายแล้ว จนกว่าจะพยายาม `send()` แล้ว timeout (ซึ่งอาจใช้เวลานานมากตาม TCP retransmission
timeout ของ OS) — heartbeat mechanism (122.7) แก้ปัญหานี้โดยตรงด้วยการเช็คที่ application
layer เอง ไม่พึ่งพา TCP/WebSocket ให้บอกเรา

### ข้อ 8: `logout()` ที่ไม่ Idempotent จะลบ User คนใหม่ที่เพิ่ง Reconnect ทิ้งผิดคน

ถ้า `logout(conn)` ลบ `users_[username]` ทิ้งโดยไม่เช็คก่อนว่า connection ที่ถืออยู่ตรงกับ
connection ที่บันทึกไว้จริงหรือไม่ จะเกิดปัญหาเมื่อ: user คนหนึ่ง reconnect เร็วมากด้วยชื่อ
เดิม (เช่น auto-reconnect ของ client) ก่อนที่ `logout()` ของ connection เก่าจะทันถูกเรียก —
`logout()` เก่าที่มาทีหลังอาจลบ connection **ใหม่** ที่เพิ่งเข้ามาทิ้งไปผิดๆ `chat_server.cpp`
ป้องกันด้วยการเช็ค `it->second != conn` ก่อนลบทุกครั้ง (122.3)

### ข้อ 9: ไม่จำกัดความยาวข้อความ/username (ช่องโหว่ DoS)

ถ้าไม่ตรวจสอบ `MAX_MESSAGE_LEN`/`MAX_USERNAME_LEN` ผู้ใช้ที่ประสงค์ร้ายสามารถส่งข้อความยาว
หลาย MB หรือ username ยาวเป็นพันตัวอักษรได้ ซึ่งกิน memory ทั้งฝั่ง server (เก็บลง SQLite),
ฝั่ง broadcast (ส่งซ้ำไปทุกคนในห้อง), และฝั่ง client ทุกคนที่ต้อง parse JSON ก้อนใหญ่นั้น
`chat_server.cpp` validate ทั้งสองค่าใน `handle_client_message()` ก่อนประมวลผลเสมอ

---

## แบบฝึกหัดท้ายบท

1. **Typing Indicator**: เพิ่ม message type `"typing"` ที่ broadcast บอกสมาชิกคนอื่นในห้องว่า
   "username กำลังพิมพ์อยู่" โดยไม่ต้อง persist ลง SQLite เลย (ดูเฉลยเต็มด้านล่าง)
2. **Message Reactions**: เพิ่มความสามารถกดรีแอกชัน (emoji) ใส่ข้อความที่มีอยู่แล้ว โดยต้อง
   เก็บ `id` ของแต่ละข้อความไว้ตั้งแต่ตอน insert ลง SQLite เพื่อให้ client อ้างอิงกลับมาได้
   (ดูเฉลยเต็มด้านล่าง)
3. **Read Receipts**: เพิ่ม message type `"read"` ที่ client ส่งมาบอกว่า "อ่านข้อความ id นี้
   แล้ว" แล้ว broadcast แจ้งผู้ส่งเดิมว่า "username คนนี้อ่านแล้ว" (ต้องคิดว่าจะเก็บสถานะ
   "ใครอ่านอะไรไปแล้วบ้าง" ไว้ที่ไหน — ในหน่วยความจำหรือ SQLite ก็ได้ ลองเปรียบเทียบข้อดี
   ข้อเสียทั้งสองแบบ)
4. **Rate Limiting ต่อผู้ใช้**: จำกัดจำนวนข้อความที่ 1 คนส่งได้ต่อวินาที (เช่น ไม่เกิน 5
   ข้อความ/วินาที) ถ้าเกินให้ตอบ `error` กลับแทนที่จะ broadcast (ใบ้: เก็บ
   `std::deque<time_point>` ของเวลาที่ส่งข้อความล่าสุดต่อ username แล้วนับจำนวนที่อยู่ใน
   หน้าต่างเวลา 1 วินาทีล่าสุด)
5. **ห้องที่มีรหัสผ่าน (Private Room)**: เพิ่ม field `password` ตอน `join` ห้องที่ถูกสร้างไว้
   ด้วยรหัสผ่าน ถ้ารหัสไม่ตรงให้ปฏิเสธการ join (ใบ้: ต้องเก็บ hash ของรหัสผ่านต่อห้อง ไม่ใช่
   plain text — ทบทวนการ hash password จาก Part 110)
6. **Multi-Server Scaling ผ่าน Pub/Sub** (โจทย์ขั้นสูง เชื่อมโยงไปสู่ Part 123): ระบบปัจจุบัน
   รองรับได้แค่ 1 process — ถ้าต้องรองรับผู้ใช้นับล้านคนจะต้องมีหลาย server instance ทำงาน
   ขนานกัน ลองออกแบบคร่าวๆ ว่าจะใช้ระบบ pub/sub กลาง (เช่น Redis Pub/Sub หรือระบบที่เขียนเอง
   แบบ Part 123 — Distributed Key-Value Store) เพื่อ broadcast ข้อความข้าม server instance
   ได้อย่างไร โดยที่ `ChatServer`/`IConnection` ที่ออกแบบไว้ตอนนี้เปลี่ยนแปลงน้อยที่สุด

### เฉลยข้อ 1: Typing Indicator (ทดสอบคอมไพล์และรันจริงแล้ว)

เพิ่มเมธอดใน `ChatServer`:

```cpp
/* ---------------- แบบฝึกหัด: Typing Indicator ----------------
 * ไม่ persist ลง SQLite เลย (เป็นสถานะชั่วคราวมาก ไม่มีประโยชน์ที่จะเก็บ)
 * และไม่ต้องมี id ใดๆ — แค่ broadcast บอกคนอื่นในห้องว่า "username กำลังพิมพ์" */
bool broadcast_typing(const std::string& username, const std::string& room) {
    bool member;
    {
        std::lock_guard<std::mutex> lock(mutex_);
        auto rit = rooms_.find(room);
        member = (rit != rooms_.end()) && rit->second.count(username) > 0;
    }
    if (!member) return false;
    broadcast_room(room, json{{"type", "typing"}, {"room", room}, {"username", username}}.dump(),
                   username /* ไม่ต้องส่งกลับไปหาตัวเอง */);
    return true;
}
```

เพิ่ม dispatch case ใน `handle_client_message()`:

```cpp
if (type == "typing") {
    std::string room = msg.value("room", "");
    if (room.empty()) { conn->send(err_json("ต้องระบุ room")); return; }
    server.broadcast_typing(conn->username, room);
    return;
}
```

ทดสอบจริง:

```python
await a.send(json.dumps({"type": "typing", "room": "general"}))
print("bob sees alice typing ->", await b.recv())
```

ผลลัพธ์จริง:

```
bob sees alice typing -> {"room":"general","type":"typing","username":"alice"}
```

### เฉลยข้อ 2: Message Reactions (ทดสอบคอมไพล์และรันจริงแล้ว)

ขั้นแรกต้องแก้ `send_room_message()` ให้ **คืนค่า `id`** ของข้อความที่เพิ่ง insert (ใช้
`sqlite3_last_insert_rowid`) แทนที่จะคืนแค่ `bool`:

```cpp
// เพิ่มใน SqliteDB (sqlite_raii.hpp)
long long last_insert_rowid() const {
    return static_cast<long long>(sqlite3_last_insert_rowid(db_));
}
```

```cpp
// เพิ่มตาราง reactions ตอนสร้าง ChatServer
db_.exec(
    "CREATE TABLE IF NOT EXISTS reactions ("
    "  id         INTEGER PRIMARY KEY AUTOINCREMENT,"
    "  message_id INTEGER NOT NULL,"
    "  room       TEXT NOT NULL,"
    "  username   TEXT NOT NULL,"
    "  emoji      TEXT NOT NULL,"
    "  ts         INTEGER NOT NULL"
    ");");
```

```cpp
// แก้ send_room_message() ให้คืน id แทน bool
long long send_room_message(const std::string& from, const std::string& room, const std::string& text) {
    bool member;
    {
        std::lock_guard<std::mutex> lock(mutex_);
        auto rit = rooms_.find(room);
        member = (rit != rooms_.end()) && rit->second.count(from) > 0;
    }
    if (!member) return -1;

    const long long ts = now_epoch_seconds();
    long long msg_id = persist_room_message(room, from, text, ts); /* เปลี่ยนให้คืน id */
    broadcast_room(room, json{{"type", "message"}, {"room", room}, {"from", from},
                               {"text", text}, {"ts", ts}, {"id", msg_id}}.dump());
    return msg_id;
}
```

เมธอดใหม่สำหรับกดรีแอกชัน:

```cpp
/* ---------------- แบบฝึกหัด: Message Reactions ----------------
 * ตรวจสอบแค่ว่าผู้กดรีแอกชันเป็นสมาชิกห้องอยู่จริง (ไม่ตรวจว่า message_id
 * มีอยู่จริงในห้องนั้นหรือไม่ เพื่อความง่าย — ในระบบจริงควร JOIN กับตาราง
 * messages เพื่อยืนยันว่า message_id นั้นอยู่ในห้อง room จริง ก่อนบันทึก) */
bool react_to_message(const std::string& username, const std::string& room, long long message_id,
                       const std::string& emoji) {
    bool member;
    {
        std::lock_guard<std::mutex> lock(mutex_);
        auto rit = rooms_.find(room);
        member = (rit != rooms_.end()) && rit->second.count(username) > 0;
    }
    if (!member) return false;

    const long long ts = now_epoch_seconds();
    persist_reaction(message_id, room, username, emoji, ts);
    broadcast_room(room, json{{"type", "reaction"}, {"room", room}, {"message_id", message_id},
                               {"username", username}, {"emoji", emoji}}.dump());
    return true;
}
```

เพิ่ม dispatch case:

```cpp
if (type == "react") {
    std::string room = msg.value("room", "");
    std::string emoji = msg.value("emoji", "");
    long long message_id = msg.value("message_id", -1LL);
    if (room.empty() || emoji.empty() || message_id < 0) {
        conn->send(err_json("ต้องระบุ room, message_id และ emoji"));
        return;
    }
    if (!server.react_to_message(conn->username, room, message_id, emoji)) {
        conn->send(err_json("คุณยังไม่ได้เข้าร่วมห้อง '" + room + "'"));
    }
    return;
}
```

คอมไพล์ผ่านสะอาด ไม่มี warning และทดสอบจริงครบทุก scenario:

```python
await a.send(json.dumps({"type": "message", "room": "general", "text": "ทดสอบ reaction กันหน่อย"}))
msg_for_b = json.loads(await b.recv())
message_id = msg_for_b["id"]
await b.send(json.dumps({"type": "react", "room": "general", "message_id": message_id, "emoji": "👍"}))
```

ผลลัพธ์จริงทั้งหมด:

```
alice เห็นข้อความตัวเอง -> {'from': 'alice', 'id': 1, 'room': 'general', 'text': 'ทดสอบ reaction กันหน่อย', 'ts': 1790426550, 'type': 'message'}
bob เห็นข้อความของ alice -> {'from': 'alice', 'id': 1, 'room': 'general', 'text': 'ทดสอบ reaction กันหน่อย', 'ts': 1790426550, 'type': 'message'}
alice เห็น reaction ของ bob -> {"emoji":"👍","message_id":1,"room":"general","type":"reaction","username":"bob"}
bob เห็น reaction ของตัวเอง (echo กลับ) -> {"emoji":"👍","message_id":1,"room":"general","type":"reaction","username":"bob"}
react ในห้องที่ไม่ได้ join -> {"message":"คุณยังไม่ได้เข้าร่วมห้อง 'not-joined-room'","type":"error"}
```

และตรวจสอบด้วยว่า reaction ถูกบันทึกลง SQLite จริง:

```bash
$ python3 -c "
import sqlite3
con = sqlite3.connect('chat_ext.db')
for row in con.execute('SELECT message_id, room, username, emoji, ts FROM reactions;'):
    print(row)
"
(1, 'general', 'bob', '👍', 1790426550)
```

ฟีเจอร์ทั้งสองข้อทำงานถูกต้องครบถ้วน คอมไพล์ผ่าน 100% โดยไม่มี warning และผ่านการทดสอบจริง
ทุก scenario รวมถึง error case (react ในห้องที่ยังไม่ได้ join)

---

## สรุปท้ายบท

ใน Part นี้เราได้สร้าง **Real-time Multiplayer Chat Server ระดับ capstone** ที่:

- รองรับทั้ง **raw TCP socket** (อัปเกรดจาก Part 35) และ **WebSocket** (อัปเกรดจาก Part 109)
  พร้อมกันในโปรเซสเดียว ผ่านการออกแบบ `IConnection` interface ที่แยกชั้น transport ออกจาก
  business logic อย่างสมบูรณ์ — พิสูจน์จริงด้วยการให้ client TCP กับ WebSocket คุยกันในห้อง
  เดียวกันสำเร็จ ด้วยเครื่องมือหลายตัว (Python `socket`, Python `websockets`, `websocat`)
- มี **ห้องแชทหลายห้อง** พร้อม join/leave/list, **presence notification**, และ
  **private message (DM)** ที่ทดสอบครบทุก scenario รวมถึง error case
- เก็บ **ประวัติแชทลง SQLite** (ต่อยอด RAII wrapper จาก Part 105) และพิสูจน์แล้วว่าอยู่รอด
  ได้จริงแม้ server จะถูกฆ่าและเปิดใหม่ทั้ง process
- มี **heartbeat mechanism** ที่แยกแยะ client เงียบกับ client ปกติได้แม่นยำ พิสูจน์ด้วยการ
  ทดสอบ client จำลองสองแบบพร้อมกันจริง
- ผ่าน **load test 120 concurrent client, 36,000 message deliveries โดยไม่มีข้อความขาด
  ซ้ำ หรือหลุดข้ามห้องแม้แต่รายการเดียว**
- ผ่าน **ThreadSanitizer กับ mixed-transport stress test 54 connection** โดยไม่มี data race
  ในโค้ดแอปพลิเคชันที่เขียนเองแม้แต่จุดเดียว (พบแค่ race ที่รู้จักอยู่แล้วในโค้ดภายในของ Crow
  เอง ซึ่งไม่เกี่ยวกับ registry ของเรา)
- พิสูจน์ pitfall ที่อันตรายที่สุดของระบบระดับนี้ด้วย**ตัวเลขเวลาจริง**: การถือ mutex ค้างไว้
  ระหว่างเรียก `send()` ทำให้ระบบทั้งระบบค้างนานกว่าเวอร์ชันที่ถูกต้องมากกว่า **8,000 เท่า**

ทักษะสำคัญที่สุดที่ Part นี้ต้องการปลูกฝังไม่ใช่แค่ "เขียนแชทเซิร์ฟเวอร์ได้" แต่คือ
**วินัยในการพิสูจน์ความถูกต้องของระบบ concurrent ด้วยหลักฐานเชิงประจักษ์** — ไม่ว่าจะเป็น
ตัวเลขจาก load test, รายงานจาก ThreadSanitizer, หรือเวลาที่วัดได้จริงจากการทดลอง แทนที่จะ
เชื่อแค่ "โค้ดดูน่าจะถูก" เพียงอย่างเดียว ซึ่งเป็นทักษะที่จะติดตัวไปใช้ได้กับระบบ distributed
ที่ซับซ้อนกว่านี้มากใน **Part 123 — Capstone 3: Distributed Key-Value Store** ที่จะเจาะลึก
เรื่อง networking, concurrency, และ persistence ในระดับที่ต้องรับมือกับหลาย node พร้อมกัน

**ต่อไป:** [Part 123 — Capstone 3: Distributed Key-Value Store](./part-123-capstone-distributed-kv.md)
