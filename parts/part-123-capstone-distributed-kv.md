# Part 123: Capstone 3 — Distributed Key-Value Store (Networking + Concurrency + Persistence) (Step 977–984)

> Module K — Capstone Projects และบทสรุป | Part 123 จาก 125
> Step ที่ครอบคลุมใน Part นี้: Step 977–984
> Part ก่อนหน้า: [Part 122 — Capstone 2: Real-time Chat Server](./part-122-capstone-chat-server.md) | Part ถัดไป: [Part 124 — Capstone 4: สร้าง Web Framework ของตัวเอง](./part-124-capstone-web-framework.md)

## เป้าหมายการเรียนรู้ (Learning Objectives)

หลังจบ Part นี้ ผู้เรียนจะสามารถ:

1. อธิบายได้ว่า **Distributed System** ต่างจาก **Single-Node System** (อย่าง Mini KV Store ใน
   Part 40) อย่างไร ทั้งในแง่ความสามารถที่เพิ่มขึ้น (availability, horizontal read scaling)
   และความซับซ้อนที่แลกมา (network partition, replication lag, split-brain)
2. ออกแบบและ implement **Storage Engine** สมัยใหม่ด้วย `std::unordered_map` + `std::mutex` +
   **Write-Ahead Log (WAL)** ที่ทำให้โหนดกู้คืนสถานะได้เองหลัง process ถูก kill กลางคัน ไม่ใช่แค่
   ตอนสั่ง SAVE เหมือน Part 40
3. Implement **Snapshot** ที่บีบอัด WAL ที่ยาวขึ้นเรื่อยๆ ให้กลับมาสั้น ด้วยเทคนิค atomic
   `rename()` ต่อยอดจาก File I/O ใน Part 13
4. ออกแบบโปรโตคอล **Replication** แบบ Leader-Follower อย่างง่าย (FULLSYNC + streaming) และรู้
   อย่างซื่อสัตย์ว่านี่**ไม่ใช่** Raft/Paxos — เป็นแบบจำลองระดับการเรียนรู้ที่ไม่มี Consensus
   Algorithm ป้องกัน Split-Brain จริงจัง
5. รันหลาย process ของโปรแกรมเดียวกันบน `127.0.0.1` คนละ port พร้อมกันจริง เพื่อจำลอง "cluster"
   บนเครื่องเดียว, ยืนยันด้วยตาตัวเองว่าการ replicate ข้อมูลระหว่าง process ทำงานได้จริง
6. ทำ **Manual Failover**: kill primary, สั่ง `PROMOTE` ให้ replica กลายเป็น primary คนใหม่,
   และสั่ง `REPLICAOF` ให้ replica ที่เหลือย้ายไปเกาะ primary คนใหม่ — พร้อมทั้งได้เห็นและเข้าใจ
   ปรากฏการณ์ **Split-Brain** ด้วยข้อมูลจริงที่ diverge กันจริง
7. แยกแยะความแตกต่างที่คนส่วนใหญ่เข้าใจผิดระหว่าง "โปรเซสถูก kill -9" กับ "เครื่อง/OS พัง" และ
   อธิบายได้ว่า `fsync()` ป้องกันกรณีไหนจริงๆ ด้วยการทดลองจริงทั้งสองแบบ
8. เชื่อมโยงพฤติกรรมที่สังเกตได้ทั้งหมด (replication lag, lost write ตอน failover, retry ที่ทำให้
   ข้อมูลซ้ำ) เข้ากับทฤษฎี **CAP Theorem** และ **Consistency Model** (strong / eventual) ในระดับ
   ที่เหมาะกับการปิดหลักสูตร
9. เขียน **Client Library** เบื้องต้นที่คุยกับคลัสเตอร์ได้ พร้อมเข้าใจความหมายของ
   **At-Least-Once Delivery** และวิธีทำให้คำสั่งที่ไม่ idempotent (เช่น `INCR`) ปลอดภัยต่อการ
   retry ด้วย **Idempotency Key**

---

## 123.1 ภาพรวมโปรเจกต์และขอบเขตที่ต้องซื่อสัตย์กับตัวเอง (Step 977)

**Part 40** ปิดท้าย Module C ด้วย Mini Key-Value Store: เซิร์ฟเวอร์ตัวเดียว เก็บข้อมูลใน hash
table ป้องกันด้วย `pthread_mutex_t`, รับคำสั่งผ่าน TCP, และ SAVE/LOAD ข้อมูลลงไฟล์ด้วยมือ ระบบ
นั้นตอบโจทย์ได้ดีตราบใดที่ **เครื่องเดียวไม่เคยพัง** — แต่โลกจริงไม่ได้เป็นแบบนั้น เครื่องพัง,
ดิสก์เสีย, เครือข่ายขาด, และเมื่อข้อมูลสำคัญมากพอ การมี "สำเนา" ของมันมากกว่าหนึ่งชุดจึงไม่ใช่
ทางเลือกอีกต่อไป แต่เป็นข้อบังคับ

Capstone นี้จะเอา Mini KV Store จาก Part 40 มา **เขียนใหม่ด้วย Modern C++** (RAII, `std::mutex`,
`std::thread`, `std::optional`, `std::unordered_map` — ทบทวนจาก Part 67–68 และ 81) แล้วต่อยอดให้
เป็นระบบหลายโหนดที่มี **Replication** (สำเนาข้อมูลข้ามโหนด) และ **Write-Ahead Log** (กู้คืนสถานะ
ได้หลัง crash) จริง

### ข้อจำกัดที่ต้องพูดตรงๆ ตั้งแต่ต้น

> **นี่คือกฎทองของ Part นี้**: ทุกอย่างที่เขียนว่า "โหนด" หรือ "เครื่อง" ในบทเรียนนี้ หมายถึง
> **process แยกกันบน `127.0.0.1` คนละ port** บนเครื่องเดียวกันทั้งหมด — สภาพแวดล้อมที่ใช้เขียน
> Part นี้ไม่มีเครื่องหลายเครื่องให้ทดสอบจริง การรัน process แยกกันบน loopback คือวิธีมาตรฐานและ
> ซื่อสัตย์ที่สุดในการสอนแนวคิด distributed systems บนเครื่องเดียว **ตราบใดที่บอกไว้ตรงๆ แบบนี้**
> (ถ้าเอาไปรันข้ามเครื่องจริงในเครือข่ายจริง ต้องเปลี่ยนแค่ IP ที่ผูก socket เท่านั้น ตรรกะโค้ด
> ทั้งหมดเหมือนเดิม 100%)

และอีกข้อที่สำคัญไม่แพ้กัน:

> ระบบ replication ในโปรเจกต์นี้เป็นแบบ **Leader-Follower อย่างง่าย (async, ไม่มี consensus)**
> **ไม่ใช่** Raft, Paxos, หรือ etcd/ZooKeeper ตัวจริง ระบบจริงเหล่านั้นแก้ปัญหา **Split-Brain**
> และ **การเลือก leader คนใหม่แบบอัตโนมัติที่ปลอดภัย** ด้วยอัลกอริทึมที่ซับซ้อนมาก (เลือกเสียงข้าง
> มาก, term/epoch number, log matching property ฯลฯ) ซึ่งเกินขอบเขตของ capstone ระดับนี้มาก
> โปรเจกต์นี้จะทำ **Failover แบบ manual** (คนสั่งเอง ไม่ใช่ระบบตัดสินใจเอง) และจะ**พิสูจน์ให้เห็น
> ด้วยข้อมูลจริง**ว่าทำไมการ failover แบบไม่มี consensus ถึงเสี่ยงต่อ split-brain — เพื่อให้เข้าใจ
> ว่าของจริงอย่าง Raft มีไว้แก้ปัญหาอะไรกันแน่ ไม่ใช่แค่ท่องชื่อจำได้เฉยๆ

### สถาปัตยกรรมของระบบ

```
                       ┌───────────────────────────────────────────────┐
                       │              โหนด A (PRIMARY)                  │
   Client ──TCP──────▶ │  client-port 7001                              │
   (SET/GET/DEL/INCR)  │      │                                         │
                       │      ▼                                         │
                       │  handleClient() ──▶ Storage (hash map + mutex) │
                       │                          │                     │
                       │                          ▼                     │
                       │                    WriteAheadLog (kv.wal)      │──▶ ดิสก์
                       │                          │                     │
                       │                          ▼ (encodeEntry)       │
                       │                    ReplicaRegistry.broadcast() │
                       │                     replication-port 8001      │
                       └────────────┬───────────────────┬───────────────┘
                                    │ FULLSYNC + stream  │ FULLSYNC + stream
                                    ▼                    ▼
                       ┌────────────────────┐  ┌────────────────────┐
                       │   โหนด B (REPLICA)  │  │   โหนด C (REPLICA)  │
                       │  client-port 7002   │  │  client-port 7003   │
                       │  repl-port   8002   │  │  repl-port   8003   │
                       │  Storage (สำเนา)     │  │  Storage (สำเนา)     │
                       │  kv.wal ของตัวเอง    │  │  kv.wal ของตัวเอง    │
                       └────────────────────┘  └────────────────────┘

  ทุกกล่องคือ "process" แยกกันบนเครื่องเดียว (127.0.0.1) — ต่างกันแค่เลข port
```

**WAL/Snapshot flow ภายในแต่ละโหนด** (เหมือนกันทุกโหนดไม่ว่าจะเป็น primary หรือ replica):

```
คำสั่งเขียน (SET/DEL/INCR หรือ record ที่ replicate มา)
        │
        ▼
[1] ล็อก mutex เดียว (coarse-grained, สืบทอดปรัชญาจาก Part 40)
        │
        ▼
[2] แก้ไข std::unordered_map ในหน่วยความจำ
        │
        ▼
[3] wal_.append(...) ── write() ตรงไปยัง kernel (+ fsync() ถ้าเปิดไว้) ──▶ kv.wal
        │
        ▼
[4] ปลดล็อก แล้วค่อย broadcast ให้ replica (นอก critical section)
        │
        ▼
[5] ตอบ client (OK / VALUE ...)

เมื่อ kv.wal ยาวขึ้นเรื่อยๆ ───▶ คำสั่ง SNAPSHOT ───▶ เขียน kv.snapshot ทั้งก้อน (atomic rename)
                                                   ───▶ truncate kv.wal ให้ว่างเปล่า

ตอน process เริ่มใหม่ (startup / กู้คืนจาก crash):
   โหลด kv.snapshot (ถ้ามี) ──▶ replay kv.wal ทุก record ที่ seq ใหม่กว่า snapshot ──▶ พร้อมทำงาน
```

### ทบทวนความรู้เดิมที่โปรเจกต์นี้ใช้

| ความรู้ที่ใช้ | มาจาก Part | ใช้ทำอะไรในโปรเจกต์นี้ |
|---|---|---|
| Hash Table แนวคิด | Part 22, 40 | เปลี่ยนมาใช้ `std::unordered_map` เป็น storage engine |
| File I/O | Part 13 | WAL append, snapshot เขียน/อ่านไฟล์, `rename()` แบบ atomic |
| RAII | Part 68 | `Socket` class ห่อ file descriptor, `WriteAheadLog` ห่อ fd ของ WAL |
| Smart Pointer / Modern container | Part 67, 59 | `std::optional<std::string>` แทนการคืน `NULL`/pointer แบบ C |
| `std::thread`, `std::mutex` | Part 81–82 | Thread-per-connection, ป้องกัน race condition บน storage |
| Socket Programming (TCP) | Part 33–35 | พื้นฐาน `socket/bind/listen/accept/connect` (ห่อใน C++ แล้ว) |
| Mini KV Store (C) | Part 40 | โปรโตคอลข้อความล้วน, แนวคิด thread-per-connection, coarse locking |
| Signal/SIGPIPE (แนวคิด) | Part 28 | ต่อยอดเรื่อง MSG_NOSIGNAL ในหัวข้อ pitfalls |

### โครงสร้างไฟล์ของโปรเจกต์

```
distributed_kv/
├── src/
│   ├── common.hpp      <- Socket RAII, LineConn (อ่าน/เขียนทีละบรรทัดผ่าน TCP)
│   ├── storage.hpp      <- Storage engine: hash map + WAL + Snapshot + recovery
│   ├── node.cpp          <- โปรแกรมหลัก: รันได้ทั้ง PRIMARY/REPLICA, replication, command dispatch
│   ├── kv_client.hpp     <- Client library เบื้องต้น
│   ├── kvcli.cpp         <- เครื่องมือ command-line สำหรับคุยกับโหนด
│   └── retry_demo.cpp    <- โปรแกรมสาธิต at-least-once + idempotency key
├── fsync_demo/            <- โปรแกรมเดี่ยวๆ สาธิต pitfall เรื่อง fsync และ SIGPIPE
│   ├── buffered_wal_demo.cpp
│   ├── durable_wal_demo.cpp
│   └── sigpipe_demo.cpp
└── Makefile
```

โปรโตคอลยังคงเป็น **ข้อความล้วน (text protocol) หนึ่งคำสั่งต่อหนึ่งบรรทัด** เหมือน Part 40
(ทดสอบง่ายด้วยเครื่องมือทั่วไป) แต่เพิ่มคำสั่งใหม่สำหรับบริหารจัดการคลัสเตอร์:

| คำสั่ง | ความหมาย | ใช้ได้เมื่อไร |
|---|---|---|
| `SET <key> <value...>` | บันทึกค่า | เฉพาะ PRIMARY |
| `GET <key>` | อ่านค่า | ทุกโหนด (replica อาจตอบข้อมูลเก่ากว่าเล็กน้อย) |
| `DEL <key>` | ลบ key | เฉพาะ PRIMARY |
| `INCR <key> [REQID <id>] [DROPACK]` | บวกค่าตัวเลขทีละ 1 แบบ atomic | เฉพาะ PRIMARY |
| `EXISTS <key>` / `COUNT` | ตรวจสอบ/นับจำนวน key | ทุกโหนด |
| `EXPIRE <key> <seconds>` | ตั้ง TTL ให้ key | เฉพาะ PRIMARY |
| `SNAPSHOT` | บีบอัด WAL เป็น snapshot | ทุกโหนด (มักใช้กับ primary) |
| `INFO` | ดู role/seq/replica ที่ต่ออยู่ | ทุกโหนด |
| `PROMOTE` | เลื่อนตัวเองจาก REPLICA เป็น PRIMARY | เฉพาะ REPLICA |
| `REPLICAOF <host> <repl-port> <client-port>` | เปลี่ยน/เริ่ม replicate จาก primary ที่ระบุ | ทุกโหนด |
| `SETDELAY <ms>` | หน่วงเวลาก่อน apply record ที่ replicate เข้ามา (debug) | ทุกโหนด |
| `PING` / `QUIT` | ทดสอบการเชื่อมต่อ / ปิด connection | ทุกโหนด |

---

## 123.2 Storage Engine: Hash Map + Write-Ahead Log + Snapshot (Step 978)

### `common.hpp` — RAII Socket และการอ่านทีละบรรทัดแบบ Modern C++

ก่อนแตะ storage engine เราต้องมีเครื่องมือเครือข่ายพื้นฐานก่อน ทบทวนจาก **Part 68 (RAII)**:
ห่อ file descriptor ของ socket ด้วย class ที่ปิด fd อัตโนมัติในตอน destructor เพื่อไม่ให้มีทาง
ลืม `close()` ได้เลยไม่ว่าจะออกจาก scope ทางไหน (return กลางฟังก์ชัน, exception ฯลฯ):

```cpp
// common.hpp — Socket RAII wrapper + line-based TCP framing ที่ใช้ร่วมกันทั้งฝั่ง
// server (node.cpp) และฝั่ง client library (kv_client.hpp)
#pragma once
#include <string>
#include <optional>
#include <stdexcept>
#include <cstring>
#include <unistd.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <netinet/tcp.h>
#include <arpa/inet.h>
#include <netdb.h>
#include <signal.h>

constexpr size_t MAX_LINE_LEN = 8192;

// Socket — ห่อ file descriptor ของ socket ด้วย RAII (ทบทวนจาก Part 68)
// ปิด fd อัตโนมัติเมื่อ object หมดอายุ ไม่ว่าจะออกจาก scope ทางไหนก็ตาม (return กลางฟังก์ชัน,
// exception, ฯลฯ) ต่างจาก C ล้วนที่ต้องเรียก close() เองทุก path ให้ครบ
class Socket {
    int fd_ = -1;
public:
    Socket() = default;
    explicit Socket(int fd) : fd_(fd) {}
    Socket(const Socket&) = delete;
    Socket& operator=(const Socket&) = delete;
    Socket(Socket&& other) noexcept : fd_(other.fd_) { other.fd_ = -1; }
    Socket& operator=(Socket&& other) noexcept {
        if (this != &other) {
            closeNow();
            fd_ = other.fd_;
            other.fd_ = -1;
        }
        return *this;
    }
    ~Socket() { closeNow(); }

    int get() const { return fd_; }
    bool valid() const { return fd_ >= 0; }
    int release() { int f = fd_; fd_ = -1; return f; }
    void closeNow() {
        if (fd_ >= 0) {
            ::close(fd_);
            fd_ = -1;
        }
    }
};

// สร้าง listening socket ผูกกับ 127.0.0.1:port (loopback เท่านั้น — ทุก "node" ในบทเรียนนี้
// คือ process แยกกันบนเครื่องเดียว จำลอง cluster ผ่าน port ที่ต่างกัน ไม่ใช่เครื่องจริงหลายเครื่อง)
inline Socket makeListener(int port, int backlog = 32) {
    int fd = ::socket(AF_INET, SOCK_STREAM, 0);
    if (fd < 0) throw std::runtime_error("socket() failed: " + std::string(strerror(errno)));
    int opt = 1;
    setsockopt(fd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));

    sockaddr_in addr{};
    addr.sin_family = AF_INET;
    addr.sin_port = htons((uint16_t)port);
    inet_pton(AF_INET, "127.0.0.1", &addr.sin_addr);

    if (bind(fd, (sockaddr*)&addr, sizeof(addr)) < 0) {
        int e = errno;
        ::close(fd);
        throw std::runtime_error("bind() failed on port " + std::to_string(port) +
                                  ": " + strerror(e));
    }
    if (listen(fd, backlog) < 0) {
        int e = errno;
        ::close(fd);
        throw std::runtime_error("listen() failed: " + std::string(strerror(e)));
    }
    return Socket(fd);
}

// เชื่อมต่อออกไปยัง 127.0.0.1:port คืน Socket ที่ไม่ valid() ถ้าเชื่อมต่อไม่สำเร็จ
inline Socket connectTo(const std::string& host, int port) {
    int fd = ::socket(AF_INET, SOCK_STREAM, 0);
    if (fd < 0) return Socket();
    sockaddr_in addr{};
    addr.sin_family = AF_INET;
    addr.sin_port = htons((uint16_t)port);
    if (inet_pton(AF_INET, host.c_str(), &addr.sin_addr) != 1) {
        ::close(fd);
        return Socket();
    }
    if (::connect(fd, (sockaddr*)&addr, sizeof(addr)) < 0) {
        ::close(fd);
        return Socket();
    }
    return Socket(fd);
}

// LineConn — อ่าน/เขียนข้อมูลผ่าน TCP socket ทีละ "บรรทัด" (คั่นด้วย \n)
// สืบทอดแนวคิด LineReader จาก Part 40 (TCP เป็น byte stream ไม่รับประกันขอบเขตบรรทัด) แต่เปลี่ยน
// จาก fixed-size char buffer เป็น std::string ที่ขยายขนาดได้เอง (Modern C++ RAII: ไม่ต้อง malloc/free
// เอง — ทบทวนจาก Part 59/67)
class LineConn {
    int fd_;
    std::string pending_; // ไบต์ที่ recv() มาแล้วแต่ยังไม่ครบบรรทัด
public:
    explicit LineConn(int fd) : fd_(fd) {}

    // คืนบรรทัด (ไม่รวม \n หรือ \r\n) ถ้าอ่านสำเร็จ, nullopt ถ้า peer ปิดการเชื่อมต่อหรือเกิด error
    std::optional<std::string> readLine() {
        for (;;) {
            auto pos = pending_.find('\n');
            if (pos != std::string::npos) {
                std::string line = pending_.substr(0, pos);
                pending_.erase(0, pos + 1);
                if (!line.empty() && line.back() == '\r') line.pop_back();
                return line;
            }
            if (pending_.size() > MAX_LINE_LEN) return std::nullopt; // ป้องกันบรรทัดยาวผิดปกติ

            char buf[4096];
            ssize_t n = recv(fd_, buf, sizeof(buf), 0);
            if (n == 0) return std::nullopt;           // ปิดการเชื่อมต่อโดยสุภาพ (FIN)
            if (n < 0) {
                if (errno == EINTR) continue;
                return std::nullopt;                    // error หรือ timeout (SO_RCVTIMEO)
            }
            pending_.append(buf, (size_t)n);
        }
    }

    // ส่งข้อความ 1 บรรทัด (เติม \n ให้อัตโนมัติ) วนส่งจนครบเพื่อรับมือ partial write
    bool sendLine(const std::string& text) {
        std::string out = text;
        out.push_back('\n');
        size_t sent = 0;
        while (sent < out.size()) {
            ssize_t n = send(fd_, out.data() + sent, out.size() - sent, MSG_NOSIGNAL);
            if (n <= 0) {
                if (n < 0 && errno == EINTR) continue;
                return false;
            }
            sent += (size_t)n;
        }
        return true;
    }
};

// ตั้ง timeout สำหรับการ recv() บน fd นี้ (ใช้ทำ client-side retry-on-timeout, ดูหัวข้อ at-least-once)
inline void setRecvTimeout(int fd, int millis) {
    timeval tv{};
    tv.tv_sec = millis / 1000;
    tv.tv_usec = (millis % 1000) * 1000;
    setsockopt(fd, SOL_SOCKET, SO_RCVTIMEO, &tv, sizeof(tv));
}
```

สังเกตว่า `sendLine()` ใช้ `MSG_NOSIGNAL` ตั้งแต่แรก — เหตุผลเต็มๆ จะอธิบายในหัวข้อ 123.7 เรื่อง
pitfall ของ `SIGPIPE`

### `storage.hpp` — หัวใจของระบบ: WAL ต้อง "เขียนก่อนตอบกลับเสมอ"

หลักการ **Write-Ahead Logging (WAL)** คือหลักการเดียวกับที่ PostgreSQL, SQLite, และ Kafka ใช้จริง:
**ก่อนจะเปลี่ยนสถานะอะไรถาวร ต้องเขียน record อธิบายการเปลี่ยนแปลงนั้นลง log แบบ append-only
ก่อนเสมอ** ถ้าระบบล่มกลางคัน แค่ replay log นี้ตามลำดับก็กู้คืนสถานะล่าสุดได้เป๊ะ

```cpp
// storage.hpp — Storage engine: in-memory hash map (std::unordered_map) + Write-Ahead Log (WAL)
// + Snapshot ต่อยอดจากแนวคิด Part 40 (KVStore ใน C) และ Part 13 (File I/O) แต่เพิ่ม WAL เพื่อให้
// กู้คืนสถานะได้แม้ process ถูก kill กลางคัน (ไม่ใช่แค่ตอนสั่ง SAVE เหมือน Part 40)
//
// จุดออกแบบสำคัญ: ทุกการเปลี่ยนแปลงสถานะ (SET/DEL/INCR) ถูกล็อกด้วย mutex เดียว (coarse-grained,
// สืบทอดปรัชญาจาก Part 40 หัวข้อ 40.4 — "Correctness ก่อน Performance เสมอ") และ "เขียนลง WAL
// ก่อนตอบกลับ" (Write-Ahead Logging: หลักการเดียวกับที่ PostgreSQL/SQLite/Kafka ใช้จริง)
#pragma once
#include <string>
#include <unordered_map>
#include <mutex>
#include <atomic>
#include <fstream>
#include <sstream>
#include <optional>
#include <vector>
#include <fcntl.h>
#include <unistd.h>
#include <cstdio>
#include <chrono>
#include <cstring>

// เข้ารหัส record หนึ่งแถวของ WAL / replication stream เป็นข้อความ 1 บรรทัด คั่นด้วย TAB:
//   "<seq>\tSET\t<key>\t<value>"
//   "<seq>\tDEL\t<key>\t"
// ข้อจำกัดที่ยอมรับได้ของโปรเจกต์ระดับนี้ (สืบทอดจาก Part 40): key/value ห้ามมี TAB หรือ newline
// ฝังอยู่ข้างใน — โปรโตคอลข้อความล้วนบรรทัดเดียวไม่รองรับกรณีนั้น
inline std::string encodeEntry(uint64_t seq, const std::string& op,
                                const std::string& key, const std::string& value) {
    std::string out;
    out.reserve(key.size() + value.size() + 32);
    out += std::to_string(seq);
    out += '\t'; out += op;
    out += '\t'; out += key;
    out += '\t'; out += value;
    return out;
}

struct DecodedEntry {
    uint64_t seq;
    std::string op;   // "SET" หรือ "DEL"
    std::string key;
    std::string value;
};

// แยก field ด้วย TAB — คืน nullopt ถ้ารูปแบบผิด (ใช้ตอน replay WAL เจอบรรทัดสุดท้ายที่เขียนไม่ครบ
// เพราะ process ถูก kill กลางคันขณะ write() กำลังทำงาน)
inline std::optional<DecodedEntry> decodeEntry(const std::string& line) {
    size_t p1 = line.find('\t');
    if (p1 == std::string::npos) return std::nullopt;
    size_t p2 = line.find('\t', p1 + 1);
    if (p2 == std::string::npos) return std::nullopt;
    size_t p3 = line.find('\t', p2 + 1);
    if (p3 == std::string::npos) return std::nullopt;

    DecodedEntry e;
    try {
        e.seq = std::stoull(line.substr(0, p1));
    } catch (...) {
        return std::nullopt;
    }
    e.op = line.substr(p1 + 1, p2 - p1 - 1);
    e.key = line.substr(p2 + 1, p3 - p2 - 1);
    e.value = line.substr(p3 + 1);
    if (e.op != "SET" && e.op != "DEL") return std::nullopt;
    return e;
}
```

`decodeEntry` คืน `std::nullopt` แทนที่จะ throw ถ้ารูปแบบผิด — นี่ไม่ใช่แค่เรื่องสไตล์
แต่เป็นการออกแบบที่**ตั้งใจ**: ตอน replay WAL หลัง crash อาจเจอ "บรรทัดสุดท้าย" ที่เขียนไม่ครบ
(เพราะ `write()` โดนขัดจังหวะกลางคันตอน process ถูก kill) เราต้องแยกให้ออกระหว่าง "รูปแบบผิดปกติ
ที่ควรหยุด replay" กับ "error ร้ายแรงที่ควร crash โปรแกรม" — `std::optional` สื่อความหมายนี้ได้
ตรงกว่า exception มาก (ทบทวนแนวคิดนี้จาก Part 16 เรื่อง Error Handling)

### `WriteAheadLog`: ทำไมต้องใช้ POSIX `write()` แทน `std::ofstream`

```cpp
// WriteAheadLog — เขียน record ผ่าน POSIX write() ตรงๆ (ไม่ใช่ std::ofstream) โดยตั้งใจ
// เหตุผลสำคัญมาก (อธิบายละเอียดในหัวข้อ pitfalls เรื่อง fsync): std::ofstream บัฟเฟอร์ข้อมูลไว้ใน
// หน่วยความจำของ "โปรเซสเรา" ก่อน — ถ้าโปรเซสถูก kill -9 ข้อมูลที่ยังไม่ flush จะหายไปทันทีเพราะ
// มันไม่เคยไปถึง kernel เลย ส่วน write() ของ POSIX จะส่งข้อมูลเข้าไปอยู่ใน page cache ของ kernel
// ทันทีที่เรียกเสร็จ (แม้จะยังไม่ถูกเขียนลงจานจริงก็ตาม) ซึ่งอยู่รอดจากการ kill โปรเซสได้เสมอ
// (แต่ไม่รอดจากไฟดับ/OS crash — นั่นคือหน้าที่ของ fsync())
class WriteAheadLog {
    int fd_ = -1;
    bool fsyncEach_ = true;
public:
    void open(const std::string& path, bool fsyncEach) {
        fsyncEach_ = fsyncEach;
        fd_ = ::open(path.c_str(), O_WRONLY | O_CREAT | O_APPEND, 0644);
        if (fd_ < 0) throw std::runtime_error("เปิด WAL ไม่สำเร็จ: " + path);
    }
    // เขียน 1 record ต่อท้ายไฟล์ (append เท่านั้น — เขียนทับของเดิมไม่ได้ ตามหลักการ WAL)
    void append(const std::string& line) {
        std::string out = line;
        out.push_back('\n');
        size_t off = 0;
        while (off < out.size()) {
            ssize_t n = ::write(fd_, out.data() + off, out.size() - off);
            if (n < 0) {
                if (errno == EINTR) continue;
                throw std::runtime_error(std::string("write() ลง WAL ล้มเหลว: ") + strerror(errno));
            }
            off += (size_t)n;
        }
        if (fsyncEach_) {
            fsync(fd_); // บังคับให้ kernel เขียนลงดิสก์จริงก่อนคืนค่า — ทนต่อไฟดับ/OS crash
        }
    }
    void closeNow() { if (fd_ >= 0) { ::close(fd_); fd_ = -1; } }
    ~WriteAheadLog() { closeNow(); }
};
```

ประเด็นนี้สำคัญพอที่จะมี**การทดลองจริงเต็มหัวข้อ** ใน 123.5 — แต่พูดสั้นๆ ไว้ตรงนี้ก่อน: มี
"เลเยอร์บัฟเฟอร์" อยู่ 2 ชั้นระหว่างโค้ดเรากับดิสก์จริง:

```
โปรแกรมของเรา ──▶ [1. Userspace buffer (เช่น std::ofstream)] ──▶ [2. Kernel page cache] ──▶ ดิสก์จริง
```

`kill -9` ฆ่า**เฉพาะโปรเซส** — หน่วยความจำของโปรเซส (รวมถึง buffer ชั้นที่ 1) หายไปทันที แต่
**เคอร์เนลยังมีชีวิตอยู่** ข้อมูลที่ไปถึงชั้นที่ 2 แล้ว (ผ่าน `write()`) จะรอดเสมอไม่ว่าโปรเซสจะ
ตายยังไง `fsync()` มีไว้ป้องกันแค่กรณี **ไฟดับ/OS crash** (ที่ชั้นที่ 2 เองก็หายไปด้วย) เท่านั้น —
รายละเอียดพร้อมผลการทดลองจริงอยู่ในหัวข้อ 123.5

### `Storage`: รวม hash map + WAL + Snapshot เป็นก้อนเดียวที่ atomic

```cpp
struct ApplyResult {
    bool ok = true;
    uint64_t seq = 0;
    std::string line;      // ข้อความที่เขียนลง WAL / ที่จะ broadcast ให้ replica (encodeEntry ผลลัพธ์)
    std::string value;     // ใช้กับ INCR (ค่าใหม่หลังบวก) หรือ error message
};

class Storage {
    std::mutex mtx_;
    std::unordered_map<std::string, std::string> map_;
    std::unordered_map<std::string, std::chrono::steady_clock::time_point> expireAt_; // ส่วนขยาย TTL
    std::unordered_map<std::string, std::string> reqCache_; // idempotency-key cache (ส่วนขยาย)
    WriteAheadLog wal_;
    uint64_t seq_ = 0;
    std::string dataDir_, snapshotPath_, walPath_;
    bool fsyncEach_ = true;

    static bool isExpiredNoLock(std::unordered_map<std::string, std::chrono::steady_clock::time_point>& m,
                                 const std::string& key) {
        auto it = m.find(key);
        if (it == m.end()) return false;
        return std::chrono::steady_clock::now() >= it->second;
    }

public:
    void init(const std::string& dataDir, bool fsyncEach) {
        dataDir_ = dataDir;
        fsyncEach_ = fsyncEach;
        snapshotPath_ = dataDir + "/kv.snapshot";
        walPath_ = dataDir + "/kv.wal";
        recover();
        wal_.open(walPath_, fsyncEach_);
    }

    uint64_t currentSeq() {
        std::lock_guard<std::mutex> lk(mtx_);
        return seq_;
    }

    // ---------- คำสั่งที่ client เรียกตรง (เฉพาะตอน role == PRIMARY) ----------
    ApplyResult doSet(const std::string& key, const std::string& value) {
        std::lock_guard<std::mutex> lk(mtx_);
        map_[key] = value;
        expireAt_.erase(key);
        seq_++;
        std::string line = encodeEntry(seq_, "SET", key, value);
        wal_.append(line);
        return {true, seq_, line, value};
    }

    ApplyResult doDel(const std::string& key) {
        std::lock_guard<std::mutex> lk(mtx_);
        bool existed = map_.erase(key) > 0;
        expireAt_.erase(key);
        if (!existed) return {false, seq_, "", ""};
        seq_++;
        std::string line = encodeEntry(seq_, "DEL", key, "");
        wal_.append(line);
        return {true, seq_, line, ""};
    }

    // INCR ต้อง "อ่าน-บวก-เขียน-log" เป็นก้อนเดียวใน critical section เดียวกัน มิฉะนั้น 2 client
    // ที่ INCR key เดียวกันพร้อมกันจะเกิด Race Condition แบบเดียวกับที่ Part 40 เตือนไว้เรื่อง GET
    // แล้วตามด้วย SET แยกกัน 2 จังหวะ — ทางแก้คือทำทุกอย่างในนี้โดยไม่ปล่อย mutex เลย
    ApplyResult doIncr(const std::string& key) {
        std::lock_guard<std::mutex> lk(mtx_);
        return doIncrLocked(key);
    }

    // idempotency-key wrapper รอบ doIncr — ใช้แก้ปัญหา "at-least-once retry" (แบบฝึกหัดข้อ 1)
    // ล็อก mutex แค่ครั้งเดียวตลอดทั้งฟังก์ชัน (เช็ค cache + apply จริง) เพื่อไม่ให้ request-id
    // เดียวกันที่มาถึงพร้อมกันสองจังหวะ (เช่น retry ที่ยิงซ้อนกันพอดี) แซงคิวกันจน apply ซ้ำได้
    ApplyResult doIncrIdempotent(const std::string& key, const std::string& reqId) {
        std::lock_guard<std::mutex> lk(mtx_);
        auto cached = reqCache_.find(reqId);
        if (cached != reqCache_.end()) {
            // เคยเห็น request-id นี้แล้ว: คืนผลลัพธ์เดิมโดยไม่ apply ซ้ำ
            return {true, seq_, "", cached->second};
        }
        ApplyResult r = doIncrLocked(key);
        if (r.ok) reqCache_[reqId] = r.value;
        return r;
    }

private:
    // เหมือน doIncr ทุกประการ แต่สมมติว่า mtx_ ถูกล็อกไว้แล้วโดยผู้เรียก (ใช้ภายในคลาสเท่านั้น)
    ApplyResult doIncrLocked(const std::string& key) {
        long long val = 0;
        auto it = map_.find(key);
        bool alive = it != map_.end() && !isExpiredNoLock(expireAt_, key);
        if (alive) {
            try {
                size_t pos = 0;
                val = std::stoll(it->second, &pos);
                if (pos != it->second.size()) throw std::invalid_argument("not fully numeric");
            } catch (...) {
                return {false, seq_, "", "ERROR value is not an integer"};
            }
        }
        val += 1;
        std::string sval = std::to_string(val);
        map_[key] = sval;
        expireAt_.erase(key);
        seq_++;
        std::string line = encodeEntry(seq_, "SET", key, sval);
        wal_.append(line);
        return {true, seq_, line, sval};
    }

public:

    // ---------- TTL (แบบฝึกหัดข้อ 2: ทำเฉลยเต็มรูปแบบ) ----------
    void doExpire(const std::string& key, int seconds) {
        std::lock_guard<std::mutex> lk(mtx_);
        if (map_.find(key) == map_.end()) return;
        expireAt_[key] = std::chrono::steady_clock::now() + std::chrono::seconds(seconds);
    }

    // เก็บกวาด key ที่หมดอายุแบบ active (เรียกจาก background thread เป็นระยะ)
    void reapExpired() {
        std::lock_guard<std::mutex> lk(mtx_);
        auto now = std::chrono::steady_clock::now();
        for (auto it = expireAt_.begin(); it != expireAt_.end();) {
            if (now >= it->second) {
                map_.erase(it->first);
                it = expireAt_.erase(it);
            } else ++it;
        }
    }

    // ---------- อ่านข้อมูล (ทำได้ทั้ง PRIMARY และ REPLICA) ----------
    std::optional<std::string> get(const std::string& key) {
        std::lock_guard<std::mutex> lk(mtx_);
        if (isExpiredNoLock(expireAt_, key)) return std::nullopt; // lazy expiration
        auto it = map_.find(key);
        if (it == map_.end()) return std::nullopt;
        return it->second;
    }
    bool exists(const std::string& key) {
        std::lock_guard<std::mutex> lk(mtx_);
        if (isExpiredNoLock(expireAt_, key)) return false;
        return map_.find(key) != map_.end();
    }
    size_t count() {
        std::lock_guard<std::mutex> lk(mtx_);
        return map_.size();
    }

    // ---------- ใช้โดยฝั่ง replica เมื่อรับ record จาก replication stream ----------
    void applyReplicated(uint64_t seq, const std::string& op,
                          const std::string& key, const std::string& value) {
        std::lock_guard<std::mutex> lk(mtx_);
        if (op == "SET") { map_[key] = value; expireAt_.erase(key); }
        else if (op == "DEL") { map_.erase(key); expireAt_.erase(key); }
        if (seq > seq_) seq_ = seq;
        wal_.append(encodeEntry(seq, op, key, value));
    }

    // FULLSYNC: คัดลอกสถานะทั้งหมดออกมาพร้อม seq ปัจจุบัน (ใช้ตอน replica เพิ่งต่อเข้ามาใหม่)
    std::pair<uint64_t, std::vector<std::pair<std::string, std::string>>> snapshotForSync() {
        std::lock_guard<std::mutex> lk(mtx_);
        std::vector<std::pair<std::string, std::string>> out;
        out.reserve(map_.size());
        for (auto& kv : map_) {
            if (!isExpiredNoLock(expireAt_, kv.first)) out.push_back(kv);
        }
        return {seq_, out};
    }

    // ใช้ตอนฝั่ง replica ทำ FULLSYNC ครั้งแรก (แทนที่สถานะทั้งหมดด้วยข้อมูลจาก primary)
    void loadFullSync(uint64_t seq, const std::vector<std::pair<std::string, std::string>>& pairs) {
        std::lock_guard<std::mutex> lk(mtx_);
        map_.clear();
        expireAt_.clear();
        for (auto& kv : pairs) map_[kv.first] = kv.second;
        seq_ = seq;
        // เขียนเป็น snapshot ทันที (ไม่ใช่ WAL) แล้วเริ่ม WAL ใหม่ที่ว่างเปล่า — เท่ากับปฏิบัติ
        // เหมือนเพิ่งกู้คืนจาก snapshot ที่มี seq เท่ากับตอนนี้
        writeSnapshotFileNoLock();
        wal_.closeNow();
        if (::truncate(walPath_.c_str(), 0) != 0) { /* ไม่ใช่ error ร้ายแรง: ไฟล์จะถูกเปิดใหม่แบบ append ต่อทันทีด้านล่าง */ }
        wal_.open(walPath_, fsyncEach_);
    }

    // ---------- SNAPSHOT: บีบอัด WAL ที่ยาวขึ้นเรื่อยๆ ให้กลับมาสั้น (Part 13 File I/O ต่อยอด) ----------
    void snapshotNow() {
        std::lock_guard<std::mutex> lk(mtx_);
        writeSnapshotFileNoLock();
        wal_.closeNow();
        if (::truncate(walPath_.c_str(), 0) != 0) { /* ไม่ใช่ error ร้ายแรง: ไฟล์จะถูกเปิดใหม่แบบ append ต่อทันทีด้านล่าง */ } // WAL ทุก record ก่อนหน้านี้ถูกรวมเข้า snapshot แล้ว
        wal_.open(walPath_, fsyncEach_);
    }

private:
    void writeSnapshotFileNoLock() {
        std::string tmpPath = snapshotPath_ + ".tmp";
        FILE* fp = fopen(tmpPath.c_str(), "w");
        if (!fp) throw std::runtime_error("เปิดไฟล์ snapshot ชั่วคราวไม่สำเร็จ");
        fprintf(fp, "SNAPSHOT_SEQ\t%llu\n", (unsigned long long)seq_);
        for (auto& kv : map_) {
            if (isExpiredNoLock(expireAt_, kv.first)) continue;
            fprintf(fp, "%s\t%s\n", kv.first.c_str(), kv.second.c_str());
        }
        fflush(fp);
        fsync(fileno(fp));
        fclose(fp);
        // rename() บนระบบไฟล์เดียวกันเป็นปฏิบัติการ atomic ของ POSIX: ผู้อ่านจะเห็นไฟล์เก่าเต็มๆ
        // หรือไฟล์ใหม่เต็มๆ เท่านั้น ไม่มีทางเห็นไฟล์ snapshot ที่เขียนค้างอยู่ครึ่งเดียว
        if (rename(tmpPath.c_str(), snapshotPath_.c_str()) != 0) {
            throw std::runtime_error("rename() snapshot ล้มเหลว");
        }
    }

    void recover() {
        uint64_t snapSeq = 0;
        std::ifstream snap(snapshotPath_);
        if (snap.is_open()) {
            std::string line;
            bool first = true;
            while (std::getline(snap, line)) {
                if (first) {
                    first = false;
                    size_t tab = line.find('\t');
                    if (tab != std::string::npos && line.substr(0, tab) == "SNAPSHOT_SEQ") {
                        snapSeq = std::stoull(line.substr(tab + 1));
                    }
                    continue;
                }
                size_t tab = line.find('\t');
                if (tab == std::string::npos) continue;
                map_[line.substr(0, tab)] = line.substr(tab + 1);
            }
            seq_ = snapSeq;
            printf("[recover] โหลด snapshot สำเร็จ: %zu key, seq=%llu\n",
                   map_.size(), (unsigned long long)seq_);
        } else {
            printf("[recover] ไม่พบไฟล์ snapshot (เริ่มต้นด้วยสถานะว่างเปล่า)\n");
        }

        std::ifstream wal(walPath_);
        if (wal.is_open()) {
            std::string line;
            size_t applied = 0, skippedOld = 0;
            while (std::getline(wal, line)) {
                if (line.empty()) continue;
                auto decoded = decodeEntry(line);
                if (!decoded) {
                    // แถวสุดท้ายที่ parse ไม่ได้ = สัญญาณคลาสสิกของ "เขียนค้างตอนถูก kill กลางคัน"
                    // (write() ไม่ atomic ในระดับ 'บรรทัด' ถ้า process ตายกลาง write) เลือกที่จะ
                    // ข้ามแถวนั้นทิ้งและหยุด replay ต่อ (ปลอดภัยกว่าเดา) แทนที่จะ crash โปรแกรม
                    printf("[recover] คำเตือน: พบ WAL record ที่ decode ไม่ได้ ('%s') "
                           "ข้ามและหยุด replay ต่อจากจุดนี้ (สันนิษฐานว่าเขียนค้างตอน crash)\n",
                           line.c_str());
                    break;
                }
                if (decoded->seq <= seq_) { skippedOld++; continue; }
                if (decoded->op == "SET") map_[decoded->key] = decoded->value;
                else map_.erase(decoded->key);
                seq_ = decoded->seq;
                applied++;
            }
            printf("[recover] replay WAL: apply %zu record (ข้าม %zu record เก่ากว่า snapshot), "
                   "seq หลัง replay = %llu\n", applied, skippedOld, (unsigned long long)seq_);
        } else {
            printf("[recover] ไม่พบไฟล์ WAL (ปกติสำหรับโหนดที่เพิ่งสร้างใหม่)\n");
        }
    }
};
```

### จุดออกแบบที่สำคัญที่สุดของหัวข้อนี้

1. **Coarse-Grained Locking ยังคงเป็นทางเลือกที่ถูกต้อง** เหมือนใน Part 40: `Storage` มี mutex
   ตัวเดียว ล็อกทั้งการแก้ hash map และการเขียน WAL เป็นก้อนเดียวกัน — เพราะ WAL ต้องสะท้อนลำดับ
   การเปลี่ยนแปลงที่ตรงกับสิ่งที่เกิดขึ้นจริงใน memory เป๊ะๆ ถ้าแยกล็อกกันจะเกิดกรณีที่สอง thread
   เขียน WAL สลับลำดับกับที่ apply เข้า memory จริง ทำให้ replay แล้วได้ผลลัพธ์คนละอันกับที่ควรจะเป็น
2. **`doIncrLocked` ทำ "อ่าน-บวก-เขียน-log" ในก้อนเดียว** — นี่คือทางแก้ของปัญหาที่ Part 40 เตือน
   ไว้ตรงๆ (หัวข้อ 40.4 และแบบฝึกหัดข้อ 3): ถ้าแยก `GET` แล้วตามด้วย `SET` เป็นสองคำสั่งจาก
   ฝั่ง caller จะเกิด Race Condition ทันทีเมื่อมีสอง client `INCR` key เดียวกันพร้อมกัน
3. **`applyReplicated` ไม่ผ่าน `doSet`/`doDel`** เพราะ replica ไม่ได้เป็นคน "ตัดสินใจ" ค่า seq
   เอง — มันแค่รับ seq ที่ primary กำหนดมาแล้วแปะตามนั้น (`if (seq > seq_) seq_ = seq;`) ถ้าใช้
   `doSet` ปกติ replica จะเผลอสร้าง seq ของตัวเองซ้อนทับกับของ primary ทำให้ WAL ของ replica
   ไม่ตรงกับของ primary อีกต่อไป

---

## 123.3 Replication Protocol และ `node.cpp` (Step 979)

### ออกแบบโปรโตคอล Replication: FULLSYNC + Streaming

เมื่อ replica ต่อเข้ามาที่ replication port ของ primary (หรือของโหนดใดๆ ที่ถูก `PROMOTE` เป็น
primary) มันจะทำ **handshake สองขั้นตอน**:

```
Replica                                          Primary (replication port)
   │──────────────── "SYNC" ───────────────────▶│
   │                                              │ ล็อก storage, คัดลอกทุก key/value + seq ปัจจุบัน
   │◀──────── "FULLSYNC <seq> <n>" ──────────────│
   │◀──────── "key1\tvalue1" (n บรรทัด) ─────────│
   │◀──────────────── "ENDSYNC" ─────────────────│
   │  (แทนที่สถานะทั้งหมดด้วยข้อมูลที่ได้)         │
   │                                              │
   │◀─────── "seq\tSET\tkey\tvalue" (ต่อเนื่อง) ──│ ← ทุกครั้งที่มี SET/DEL/INCR ใหม่บน primary
   │  (apply เข้า storage ของตัวเอง)               │
   │      ...ต่อเนื่องไปเรื่อยๆ จนกว่าจะหลุด...      │
```

นี่คือการจำลอง**แบบง่ายมาก**ของสิ่งที่ Redis เรียกว่า RDB + replication backlog, หรือ MySQL
เรียกว่า binlog position — ระบบจริงรองรับ **Partial Resync** (ต่อจากจุดที่ค้างไว้โดยไม่ต้องขน
ข้อมูลทั้งหมดใหม่) แต่โปรเจกต์นี้ **ทำ FULLSYNC ใหม่ทุกครั้งที่เชื่อมต่อ** — จงใจทำให้ง่ายและ
ซื่อสัตย์ว่านี่คือทางลัดที่ยอมรับได้สำหรับการเรียนรู้ (ดูแบบฝึกหัดข้อ 5 สำหรับแนวทางขยาย)

### `node.cpp` — ส่วนที่ 1: `ReplicaRegistry` และสถานะของโหนด

```cpp
// node.cpp — Distributed KV Store: node เดียวที่รันได้ทั้งบทบาท PRIMARY หรือ REPLICA
// (เปลี่ยนบทบาทระหว่างรันได้จริงผ่านคำสั่ง PROMOTE / REPLICAOF — เหมือน Redis)
//
// รันหลาย process ของไฟล์นี้พร้อมกันบน 127.0.0.1 คนละ port = จำลอง "cluster" บนเครื่องเดียว
// (ไม่มีเครื่องหลายเครื่องให้ทดสอบจริงในสภาพแวดล้อมนี้ — ทุกที่ที่เขียนว่า "โหนด" หรือ "เครื่อง"
// ในบทเรียนนี้ หมายถึง process แยกกันบน localhost คนละ port เท่านั้น)
#include "common.hpp"
#include "storage.hpp"

#include <atomic>
#include <thread>
#include <mutex>
#include <vector>
#include <iostream>
#include <sstream>
#include <cstdio>
#include <cstring>
#include <cstdlib>
#include <cctype>
#include <algorithm>
#include <sys/stat.h>

// ---------- ReplicaRegistry: รายชื่อ replica ที่ต่อเข้ามาที่ replication port ของโหนดนี้ ----------
class ReplicaRegistry {
    struct Link { int fd; std::string label; };
    std::mutex mtx_;
    std::vector<Link> links_;
public:
    void add(int fd, const std::string& label) {
        std::lock_guard<std::mutex> lk(mtx_);
        links_.push_back({fd, label});
    }
    void remove(int fd) {
        std::lock_guard<std::mutex> lk(mtx_);
        links_.erase(std::remove_if(links_.begin(), links_.end(),
                     [&](const Link& l) { return l.fd == fd; }), links_.end());
    }
    size_t count() {
        std::lock_guard<std::mutex> lk(mtx_);
        return links_.size();
    }
    std::string labelsCsv() {
        std::lock_guard<std::mutex> lk(mtx_);
        std::string out;
        for (auto& l : links_) { if (!out.empty()) out += ","; out += l.label; }
        return out.empty() ? "-" : out;
    }
    // ส่ง record ให้ replica ทุกตัวที่ต่ออยู่ (fire-and-forget, ไม่รอ ack กลับ — "async replication")
    // ถ้า send() ล้มเหลว "ไม่" ปิด fd หรือลบออกจาก list ตรงนี้ทันที ปล่อยให้ subscriber handler
    // thread ของ fd นั้น (ที่ค้าง readLine() รออยู่) เป็นคนตรวจจับการหลุดและลบตัวเองออกแทน —
    // กัน race ที่สอง thread แย่งกัน close() fd เดียวกันพร้อมกัน (double-close)
    void broadcast(const std::string& line) {
        std::lock_guard<std::mutex> lk(mtx_);
        std::string out = line;
        out.push_back('\n');
        for (auto& l : links_) {
            ::send(l.fd, out.data(), out.size(), MSG_NOSIGNAL);
        }
    }
};

// ---------- สถานะรวมของโหนด ----------
struct NodeState {
    std::string id;
    int clientPort = 0;
    int replPort = 0;
    std::atomic<bool> isPrimary{true};

    std::mutex upstreamMtx;
    std::string upstreamHost;
    int upstreamReplPort = 0;
    int upstreamClientPort = 0; // เก็บไว้แค่โชว์ error message สวยๆ ตอน redirect client

    std::atomic<int> outboundFd{-1};
    std::atomic<uint64_t> replGeneration{0};
    std::atomic<uint64_t> lastAppliedSeq{0};
    std::atomic<int> replicaApplyDelayMs{0}; // หน่วงเวลาก่อน apply record ที่รับมา (จำลอง replication lag)

    std::string primaryAddrForDisplay() {
        std::lock_guard<std::mutex> lk(upstreamMtx);
        if (upstreamHost.empty()) return "(ไม่ทราบ — โหนดนี้ไม่เคยตั้งค่า upstream)";
        std::ostringstream os;
        os << upstreamHost << ":" << upstreamClientPort
           << " (replication port " << upstreamReplPort << ")";
        return os.str();
    }
};

static NodeState g_ns;
static Storage g_storage;
static ReplicaRegistry g_registry;

static void logMsg(const std::string& s) {
    printf("[%s] %s\n", g_ns.id.c_str(), s.c_str());
    fflush(stdout);
}
```

จุดสำคัญ: `broadcast()` ใช้ `MSG_NOSIGNAL` เสมอ และ**ไม่ลบ fd ออกจาก list ทันทีที่ `send()`
ล้มเหลว** — ปล่อยให้ thread ที่ดูแล connection นั้น (ซึ่งค้างอยู่ที่ `readLine()`) เป็นคนลบเอง
เพื่อป้องกัน**สอง thread แย่งกัน `close()` fd เดียวกัน** (Double-Close จะทำให้ fd หมายเลขเดียวกัน
ถูก OS นำไปใช้กับ connection ใหม่โดยไม่ได้ตั้งใจ — บั๊กประเภทนี้ตรวจจับยากมากเพราะเกิดเฉพาะช่วง
timing ที่ตรงกันพอดี)

### ส่วนที่ 2: Outbound Replication Thread (ฝั่ง REPLICA)

```cpp
// ---------- Outbound replication thread: ฝั่ง REPLICA เชื่อมต่อออกไปหา primary ----------
static void runReplicaLink(std::string host, int replPort, int clientPortHint, uint64_t myGen) {
    Socket sock = connectTo(host, replPort);
    if (!sock.valid()) {
        logMsg("เชื่อมต่อ upstream " + host + ":" + std::to_string(replPort) + " ไม่สำเร็จ");
        return;
    }
    if (g_ns.replGeneration.load() != myGen) return; // ถูกแทนที่ไปแล้วก่อนแม้แต่จะเชื่อมต่อเสร็จ
    g_ns.outboundFd = sock.get();

    LineConn conn(sock.get());
    std::string req = "SYNC";
    req.push_back('\n');
    ::send(sock.get(), req.data(), req.size(), MSG_NOSIGNAL);

    auto header = conn.readLine();
    if (!header || header->rfind("FULLSYNC", 0) != 0) {
        logMsg("FULLSYNC handshake ล้มเหลว จาก " + host + ":" + std::to_string(replPort));
        g_ns.outboundFd = -1;
        return;
    }
    uint64_t seq; long n;
    sscanf(header->c_str(), "FULLSYNC %llu %ld", (unsigned long long*)&seq, &n);

    std::vector<std::pair<std::string, std::string>> pairs;
    pairs.reserve((size_t)n);
    for (long i = 0; i < n; i++) {
        auto line = conn.readLine();
        if (!line) { logMsg("การเชื่อมต่อหลุดระหว่าง FULLSYNC"); g_ns.outboundFd = -1; return; }
        size_t tab = line->find('\t');
        pairs.push_back({line->substr(0, tab), line->substr(tab + 1)});
    }
    auto end = conn.readLine();
    if (!end || *end != "ENDSYNC") {
        logMsg("รูปแบบ FULLSYNC ผิดพลาด (ไม่พบ ENDSYNC)");
    }

    g_storage.loadFullSync(seq, pairs);
    g_ns.lastAppliedSeq = seq;
    logMsg("FULLSYNC จาก " + host + ":" + std::to_string(replPort) +
           " สำเร็จ: " + std::to_string(pairs.size()) + " key, seq เริ่มต้น=" + std::to_string(seq));

    {
        std::lock_guard<std::mutex> lk(g_ns.upstreamMtx);
        g_ns.upstreamHost = host;
        g_ns.upstreamReplPort = replPort;
        g_ns.upstreamClientPort = clientPortHint;
    }

    // สตรีมรับ record ต่อเนื่องหลัง FULLSYNC เสร็จ (บทบาทเดียวกับ "replication backlog" ของ Redis
    // แบบย่อส่วน — ในระบบจริง replica ที่หลุดสั้นๆ จะขอ "resume จากจุดที่ค้างไว้" ได้ (partial resync)
    // แต่ในโปรเจกต์นี้ทุกครั้งที่เชื่อมต่อใหม่จะ FULLSYNC ใหม่หมดเสมอ — จงใจทำให้ง่าย)
    for (;;) {
        if (g_ns.replGeneration.load() != myGen) break; // ถูก REPLICAOF/PROMOTE แทนที่แล้ว
        auto line = conn.readLine();
        if (!line) {
            logMsg("*** การเชื่อมต่อไปยัง primary (" + host + ":" + std::to_string(replPort) +
                   ") หลุด — primary อาจ crash หรือถูกปิด ***");
            break;
        }
        auto decoded = decodeEntry(*line);
        if (!decoded) continue;
        int delay = g_ns.replicaApplyDelayMs.load();
        if (delay > 0) std::this_thread::sleep_for(std::chrono::milliseconds(delay));
        g_storage.applyReplicated(decoded->seq, decoded->op, decoded->key, decoded->value);
        g_ns.lastAppliedSeq = decoded->seq;
    }
    if (g_ns.outboundFd.load() == sock.get()) g_ns.outboundFd = -1;
}

// เริ่ม/เปลี่ยนสาย replication ไปยัง upstream ใหม่ (ใช้ตอน startup ด้วยและตอนรับคำสั่ง REPLICAOF)
static void startReplicationFrom(const std::string& host, int replPort, int clientPortHint) {
    uint64_t newGen = ++g_ns.replGeneration;
    int oldFd = g_ns.outboundFd.exchange(-1);
    if (oldFd >= 0) {
        ::shutdown(oldFd, SHUT_RDWR); // ปลุกให้ recv() ที่ค้างอยู่ใน thread เก่าคืนค่าแล้วจบ thread ไปเอง
    }
    g_ns.isPrimary = false;
    std::thread(runReplicaLink, host, replPort, clientPortHint, newGen).detach();
}
```

**`replGeneration`** คือกลไกสำคัญที่ทำให้ `REPLICAOF`/`PROMOTE` เปลี่ยนเป้าหมาย replication
ระหว่างรันได้อย่างปลอดภัย: ทุกครั้งที่เริ่มสาย replication ใหม่ ตัวเลขนี้จะถูกเพิ่มขึ้น thread
เก่าที่ยังค้างอยู่ (ถ้ามี) จะตรวจสอบค่านี้แล้วรู้ว่าตัวเอง "ล้าสมัย" แล้วหยุดทำงานเอง — และ
`::shutdown(oldFd, SHUT_RDWR)` เป็นเทคนิคมาตรฐานสำหรับ**ปลุกให้ `recv()` ที่ค้างบล็อกอยู่ใน
thread อื่นคืนค่าออกมาทันที** โดยไม่ต้องรอ timeout

### ส่วนที่ 3: Replication Listener (ฝั่งรับ replica ใหม่)

```cpp
// ---------- Replication listener: ฝั่ง PRIMARY (หรือโหนดใดๆ ก็ตามที่ถูก promote) รับ replica ใหม่ ----------
static void handleSubscriber(Socket sock, std::string peerLabel) {
    int fd = sock.get();
    LineConn conn(fd);
    auto first = conn.readLine();
    if (!first || *first != "SYNC") { return; }

    auto [seq, pairs] = g_storage.snapshotForSync();
    std::ostringstream header;
    header << "FULLSYNC " << seq << " " << pairs.size();
    if (!conn.sendLine(header.str())) return;
    for (auto& kv : pairs) {
        if (!conn.sendLine(kv.first + "\t" + kv.second)) return;
    }
    if (!conn.sendLine("ENDSYNC")) return;

    g_registry.add(fd, peerLabel);
    logMsg("replica เชื่อมต่อเข้ามาใหม่: " + peerLabel + " (FULLSYNC " +
           std::to_string(pairs.size()) + " key ที่ seq=" + std::to_string(seq) + ")");

    // replica ฝั่งนี้จะไม่ส่งอะไรกลับมาอีกใน protocol ที่ออกแบบไว้ — เราแค่ block รอ readLine()
    // เพื่อใช้ตรวจจับตอนที่การเชื่อมต่อหลุด (สำคัญมาก มิฉะนั้นจะไม่รู้เลยว่า replica หายไปแล้ว)
    conn.readLine();
    g_registry.remove(fd);
    logMsg("replica หลุดการเชื่อมต่อ: " + peerLabel);
}

static void replicationListenerLoop(int replPort) {
    Socket listener = makeListener(replPort);
    logMsg("replication listener ฟังอยู่ที่ 127.0.0.1:" + std::to_string(replPort));
    for (;;) {
        sockaddr_in addr{};
        socklen_t len = sizeof(addr);
        int fd = accept(listener.get(), (sockaddr*)&addr, &len);
        if (fd < 0) { if (errno == EINTR) continue; break; }
        char ip[INET_ADDRSTRLEN];
        inet_ntop(AF_INET, &addr.sin_addr, ip, sizeof(ip));
        std::string label = std::string(ip) + ":" + std::to_string(ntohs(addr.sin_port));
        std::thread(handleSubscriber, Socket(fd), label).detach();
    }
}
```

โน้ตสำคัญ: **replication listener ทำงานตลอดเวลาไม่ว่าโหนดจะมี role เป็นอะไร** ไม่ใช่แค่ตอนเป็น
PRIMARY — เหตุผลคือเมื่อโหนดถูก `PROMOTE` กลางอากาศ มันต้องพร้อมรับ replica ใหม่ทันทีโดยไม่ต้อง
restart process (จะเห็นสถานการณ์นี้จริงในหัวข้อ 123.6 ตอน failover)

### ส่วนที่ 4: Command Dispatch ฝั่ง Client

```cpp
// ---------- Client-facing command dispatch ----------
static std::string trimLeft(const std::string& s) {
    size_t i = 0;
    while (i < s.size() && (s[i] == ' ' || s[i] == '\t')) i++;
    return s.substr(i);
}

// แยก "คำสั่งแรก" ออกจากส่วนที่เหลือของบรรทัด คืน {CMD (ตัวใหญ่ทั้งหมด), rest}
static std::pair<std::string, std::string> splitCommand(const std::string& lineIn) {
    std::string line = trimLeft(lineIn);
    size_t sp = line.find_first_of(" \t");
    std::string cmd = (sp == std::string::npos) ? line : line.substr(0, sp);
    std::string rest = (sp == std::string::npos) ? "" : trimLeft(line.substr(sp + 1));
    for (auto& c : cmd) c = (char)toupper((unsigned char)c);
    return {cmd, rest};
}

static void handleClient(Socket sock, std::string peerLabel) {
    LineConn conn(sock.get());
    logMsg("client เชื่อมต่อ: " + peerLabel);
    for (;;) {
        auto lineOpt = conn.readLine();
        if (!lineOpt) { logMsg("client ปิดการเชื่อมต่อ: " + peerLabel); return; }
        auto [cmd, rest] = splitCommand(*lineOpt);

        if (cmd == "PING") {
            conn.sendLine("PONG");
        } else if (cmd == "INFO") {
            std::ostringstream os;
            os << "INFO role=" << (g_ns.isPrimary ? "PRIMARY" : "REPLICA")
               << " node=" << g_ns.id
               << " seq=" << g_storage.currentSeq()
               << " last_applied=" << g_ns.lastAppliedSeq.load()
               << " replicas=" << g_registry.count()
               << " replica_addrs=" << g_registry.labelsCsv()
               << " upstream=" << (g_ns.isPrimary ? "-" : g_ns.primaryAddrForDisplay());
            conn.sendLine(os.str());
        } else if (cmd == "COUNT") {
            conn.sendLine("COUNT " + std::to_string(g_storage.count()));
        } else if (cmd == "GET") {
            if (rest.empty()) { conn.sendLine("ERROR usage: GET <key>"); continue; }
            auto v = g_storage.get(rest);
            conn.sendLine(v ? ("VALUE " + *v) : "NOT_FOUND");
        } else if (cmd == "EXISTS") {
            if (rest.empty()) { conn.sendLine("ERROR usage: EXISTS <key>"); continue; }
            conn.sendLine(g_storage.exists(rest) ? "1" : "0");
        } else if (cmd == "SET") {
            if (!g_ns.isPrimary) {
                conn.sendLine("ERROR readonly replica; primary is at " + g_ns.primaryAddrForDisplay());
                continue;
            }
            size_t sp = rest.find_first_of(" \t");
            if (sp == std::string::npos) { conn.sendLine("ERROR usage: SET <key> <value>"); continue; }
            std::string key = rest.substr(0, sp);
            std::string value = trimLeft(rest.substr(sp + 1));
            if (key.empty() || value.empty()) { conn.sendLine("ERROR usage: SET <key> <value>"); continue; }
            auto r = g_storage.doSet(key, value);
            g_registry.broadcast(r.line);
            conn.sendLine("OK");
        } else if (cmd == "DEL") {
            if (!g_ns.isPrimary) {
                conn.sendLine("ERROR readonly replica; primary is at " + g_ns.primaryAddrForDisplay());
                continue;
            }
            if (rest.empty()) { conn.sendLine("ERROR usage: DEL <key>"); continue; }
            auto r = g_storage.doDel(rest);
            if (r.ok) { g_registry.broadcast(r.line); conn.sendLine("DELETED"); }
            else conn.sendLine("NOT_FOUND");
        } else if (cmd == "INCR") {
            if (!g_ns.isPrimary) {
                conn.sendLine("ERROR readonly replica; primary is at " + g_ns.primaryAddrForDisplay());
                continue;
            }
            std::istringstream is(rest);
            std::string key, opt1, reqId, opt2;
            is >> key >> opt1;
            bool dropAck = false;
            if (opt1 == "REQID") { is >> reqId; is >> opt2; if (opt2 == "DROPACK") dropAck = true; }
            else if (opt1 == "DROPACK") dropAck = true;
            if (key.empty()) { conn.sendLine("ERROR usage: INCR <key> [REQID <id>] [DROPACK]"); continue; }

            ApplyResult r = reqId.empty() ? g_storage.doIncr(key)
                                           : g_storage.doIncrIdempotent(key, reqId);
            if (r.ok && !r.line.empty()) g_registry.broadcast(r.line);
            if (dropAck) {
                // *** DROPACK คือ debug hook สำหรับสาธิตบทเรียนเท่านั้น: apply สำเร็จจริง แต่
                // ตั้งใจไม่ตอบกลับ client เพื่อจำลอง "ack หายระหว่างทาง" แบบควบคุมได้บนเครื่อง
                // เดียว (ของจริงมักเกิดจากแพ็กเก็ต reply สูญหายกลางเครือข่าย ซึ่งเราจำลองบน
                // loopback ตรงๆ ไม่ได้ — นี่คือ honest simplification ที่ตั้งใจบอกไว้ตรงๆ) ***
                logMsg("[DEBUG] จงใจไม่ส่ง response กลับให้ client (จำลอง ack หาย) หลัง INCR " + key);
                continue;
            }
            if (!r.ok) conn.sendLine(r.value);
            else conn.sendLine("VALUE " + r.value);
        } else if (cmd == "EXPIRE") {
            if (!g_ns.isPrimary) {
                conn.sendLine("ERROR readonly replica; primary is at " + g_ns.primaryAddrForDisplay());
                continue;
            }
            std::istringstream is(rest);
            std::string key; int secs = 0;
            is >> key >> secs;
            if (key.empty() || !is) { conn.sendLine("ERROR usage: EXPIRE <key> <seconds>"); continue; }
            g_storage.doExpire(key, secs);
            conn.sendLine("OK");
        } else if (cmd == "SETDELAY") {
            int ms = atoi(rest.c_str());
            g_ns.replicaApplyDelayMs = ms;
            conn.sendLine("OK delay=" + std::to_string(ms) + "ms");
        } else if (cmd == "SNAPSHOT") {
            g_storage.snapshotNow();
            conn.sendLine("SNAPSHOTTED seq=" + std::to_string(g_storage.currentSeq()));
        } else if (cmd == "PROMOTE") {
            if (g_ns.isPrimary) { conn.sendLine("ALREADY_PRIMARY"); continue; }
            int oldFd = g_ns.outboundFd.exchange(-1);
            if (oldFd >= 0) ::shutdown(oldFd, SHUT_RDWR);
            g_ns.replGeneration++;
            g_ns.isPrimary = true;
            logMsg("*** PROMOTE: โหนดนี้กลายเป็น PRIMARY แล้ว (manual failover) ***");
            conn.sendLine("PROMOTED");
        } else if (cmd == "REPLICAOF") {
            std::istringstream is(rest);
            std::string host; int replPort = 0, clientPort = 0;
            is >> host >> replPort >> clientPort;
            if (host.empty() || replPort == 0) {
                conn.sendLine("ERROR usage: REPLICAOF <host> <repl-port> <client-port>");
                continue;
            }
            logMsg("*** REPLICAOF: เริ่ม replicate จาก " + host + ":" + std::to_string(replPort) + " ***");
            startReplicationFrom(host, replPort, clientPort);
            conn.sendLine("REPLICATING");
        } else if (cmd == "QUIT") {
            conn.sendLine("BYE");
            return;
        } else if (cmd.empty()) {
            conn.sendLine("ERROR empty command");
        } else {
            conn.sendLine("ERROR unknown command: " + cmd);
        }
    }
}

static void clientListenerLoop(int clientPort) {
    Socket listener = makeListener(clientPort);
    logMsg("client listener ฟังอยู่ที่ 127.0.0.1:" + std::to_string(clientPort));
    for (;;) {
        sockaddr_in addr{};
        socklen_t len = sizeof(addr);
        int fd = accept(listener.get(), (sockaddr*)&addr, &len);
        if (fd < 0) { if (errno == EINTR) continue; break; }
        char ip[INET_ADDRSTRLEN];
        inet_ntop(AF_INET, &addr.sin_addr, ip, sizeof(ip));
        std::string label = std::string(ip) + ":" + std::to_string(ntohs(addr.sin_port));
        std::thread(handleClient, Socket(fd), label).detach();
    }
}

static void ttlReaperLoop() {
    for (;;) {
        std::this_thread::sleep_for(std::chrono::milliseconds(300));
        g_storage.reapExpired();
    }
}
```

สังเกตลำดับการทำงานของ `SET`: **`doSet()` (apply + log WAL) เกิดก่อน `broadcast()` (ส่งให้
replica) เสมอ** — นี่คือลำดับที่ถูกต้องของ Write-Ahead Logging: primary ต้องมั่นใจว่าตัวเอง
บันทึกสำเร็จก่อน ถึงจะกระจายข้อมูลออกไป

### ส่วนที่ 5: `main()`

```cpp
int main(int argc, char** argv) {
    std::string id = "node", role = "primary", dataDir = "./data";
    int clientPort = 7000, replPort = 8000;
    std::string primaryHost; int primaryReplPort = 0, primaryClientPort = 0;
    bool fsyncEach = true;

    for (int i = 1; i < argc; i++) {
        std::string a = argv[i];
        auto val = [&](const std::string& key) -> std::optional<std::string> {
            if (a.rfind(key, 0) == 0) return a.substr(key.size());
            return std::nullopt;
        };
        if (auto v = val("--id=")) id = *v;
        else if (auto v = val("--role=")) role = *v;
        else if (auto v = val("--port=")) clientPort = atoi(v->c_str());
        else if (auto v = val("--replport=")) replPort = atoi(v->c_str());
        else if (auto v = val("--data=")) dataDir = *v;
        else if (auto v = val("--fsync=")) fsyncEach = (*v == "on" || *v == "1" || *v == "true");
        else if (auto v = val("--primary-host=")) primaryHost = *v;
        else if (auto v = val("--primary-replport=")) primaryReplPort = atoi(v->c_str());
        else if (auto v = val("--primary-port=")) primaryClientPort = atoi(v->c_str());
    }

    g_ns.id = id;
    g_ns.clientPort = clientPort;
    g_ns.replPort = replPort;
    g_ns.isPrimary = (role != "replica");

    mkdir(dataDir.c_str(), 0755);
    logMsg("กำลังเริ่มโหนด: role=" + role + " data=" + dataDir +
           " fsync=" + (fsyncEach ? "on" : "off"));
    g_storage.init(dataDir, fsyncEach);

    std::thread(replicationListenerLoop, replPort).detach();
    std::this_thread::sleep_for(std::chrono::milliseconds(50)); // ให้ listener ผูก port เสร็จก่อน

    if (!g_ns.isPrimary) {
        if (primaryHost.empty() || primaryReplPort == 0) {
            fprintf(stderr, "role=replica ต้องระบุ --primary-host และ --primary-replport ด้วย\n");
            return 1;
        }
        startReplicationFrom(primaryHost, primaryReplPort, primaryClientPort);
    }

    std::thread(ttlReaperLoop).detach();

    logMsg("พร้อมทำงาน (client-port=" + std::to_string(clientPort) +
           ", repl-port=" + std::to_string(replPort) + ")");
    clientListenerLoop(clientPort); // block ตลอดชีวิตโปรเซส (main thread)
    return 0;
}
```

โหนดเดียวกัน (binary เดียวกัน) เปิดได้ทั้งเป็น primary หรือ replica ขึ้นกับ `--role` — นี่คือ
เหตุผลที่ `PROMOTE`/`REPLICAOF` ทำได้แบบ **runtime โดยไม่ต้อง restart process**: มันแค่เปลี่ยนค่า
`isPrimary` และ thread ที่เกี่ยวข้อง ไม่ได้เปลี่ยนโปรแกรมที่รันอยู่เลย

---

## 123.4 Build และรัน Cluster 3 โหนดจริงบนเครื่องเดียว (Step 980)

### Makefile

```makefile
CXX      = g++
CXXFLAGS = -std=c++20 -O2 -Wall -Wextra -pthread
LDFLAGS  = -pthread

.PHONY: all clean

all: node kvcli retry_demo

node: src/node.cpp src/common.hpp src/storage.hpp
	$(CXX) $(CXXFLAGS) src/node.cpp -o node $(LDFLAGS)

kvcli: src/kvcli.cpp src/kv_client.hpp src/common.hpp
	$(CXX) $(CXXFLAGS) src/kvcli.cpp -o kvcli $(LDFLAGS)

retry_demo: src/retry_demo.cpp src/kv_client.hpp src/common.hpp
	$(CXX) $(CXXFLAGS) src/retry_demo.cpp -o retry_demo $(LDFLAGS)

clean:
	rm -f node kvcli retry_demo
```

รันจริงบนเครื่องที่ใช้เขียนบทเรียนนี้ (g++ 13.3.0 บน Ubuntu 24.04):

```bash
$ make clean && make
```

```
rm -f node kvcli retry_demo
g++ -std=c++20 -O2 -Wall -Wextra -pthread src/node.cpp -o node -pthread
g++ -std=c++20 -O2 -Wall -Wextra -pthread src/kvcli.cpp -o kvcli -pthread
g++ -std=c++20 -O2 -Wall -Wextra -pthread src/retry_demo.cpp -o retry_demo -pthread
```

**คอมไพล์ผ่านทั้ง 3 ไฟล์โดยไม่มี warning แม้แต่บรรทัดเดียว** ยืนยันว่าโค้ดทั้งหมด (`common.hpp`,
`storage.hpp`, `node.cpp` รวมกว่า 550 บรรทัด) ผ่านมาตรฐาน `-Wall -Wextra` อย่างเคร่งครัด

### เริ่มคลัสเตอร์ 3 โหนด: A (primary), B และ C (replica)

เปิด terminal 3 บาน (หรือรันเป็น background process แบบด้านล่างนี้) — **นี่คือ 3 process แยกกัน
บน `127.0.0.1` คนละ port จำลอง cluster บนเครื่องเดียว**:

```bash
$ ./node --id=A --role=primary --port=7001 --replport=8001 --data=data/A --fsync=on
```

```
[A] กำลังเริ่มโหนด: role=primary data=data/A fsync=on
[recover] ไม่พบไฟล์ snapshot (เริ่มต้นด้วยสถานะว่างเปล่า)
[recover] ไม่พบไฟล์ WAL (ปกติสำหรับโหนดที่เพิ่งสร้างใหม่)
[A] replication listener ฟังอยู่ที่ 127.0.0.1:8001
[A] พร้อมทำงาน (client-port=7001, repl-port=8001)
[A] client listener ฟังอยู่ที่ 127.0.0.1:7001
```

```bash
$ ./node --id=B --role=replica --port=7002 --replport=8002 --data=data/B --fsync=on \
    --primary-host=127.0.0.1 --primary-replport=8001 --primary-port=7001
$ ./node --id=C --role=replica --port=7003 --replport=8003 --data=data/C --fsync=on \
    --primary-host=127.0.0.1 --primary-replport=8001 --primary-port=7001
```

```
# log ของ B
[B] กำลังเริ่มโหนด: role=replica data=data/B fsync=on
[recover] ไม่พบไฟล์ snapshot (เริ่มต้นด้วยสถานะว่างเปล่า)
[recover] ไม่พบไฟล์ WAL (ปกติสำหรับโหนดที่เพิ่งสร้างใหม่)
[B] replication listener ฟังอยู่ที่ 127.0.0.1:8002
[B] พร้อมทำงาน (client-port=7002, repl-port=8002)
[B] client listener ฟังอยู่ที่ 127.0.0.1:7002
[B] FULLSYNC จาก 127.0.0.1:8001 สำเร็จ: 0 key, seq เริ่มต้น=0

# log ของ A (หลัง replica ทั้งสองต่อเข้ามา)
[A] replica เชื่อมต่อเข้ามาใหม่: 127.0.0.1:48572 (FULLSYNC 0 key ที่ seq=0)
[A] replica เชื่อมต่อเข้ามาใหม่: 127.0.0.1:48558 (FULLSYNC 0 key ที่ seq=0)
```

ทั้งสาม process ขึ้นจริง เชื่อมต่อกันจริงผ่าน TCP บน loopback

### เขียนที่ primary แล้วยืนยันว่า replica ได้รับข้อมูลจริง

ใช้ `kvcli` (client library เล็กๆ ที่จะอธิบายเต็มในหัวข้อ 123.7) คุยกับแต่ละโหนด:

```bash
$ ./kvcli 127.0.0.1 7001 "SET fruit apple"
$ ./kvcli 127.0.0.1 7001 "SET city Bangkok"
$ ./kvcli 127.0.0.1 7001 "INCR orders"
$ ./kvcli 127.0.0.1 7001 "INCR orders"
```

```
OK
OK
VALUE 1
VALUE 2
```

```bash
$ ./kvcli 127.0.0.1 7001 "GET fruit"; ./kvcli 127.0.0.1 7001 "INFO"
```

```
VALUE apple
INFO role=PRIMARY node=A seq=4 last_applied=0 replicas=2 replica_addrs=127.0.0.1:48572,127.0.0.1:48558 upstream=-
```

```bash
$ ./kvcli 127.0.0.1 7002 "GET fruit"; ./kvcli 127.0.0.1 7002 "GET orders"; ./kvcli 127.0.0.1 7002 "INFO"
```

```
VALUE apple
VALUE 2
INFO role=REPLICA node=B seq=4 last_applied=4 replicas=0 replica_addrs=- upstream=127.0.0.1:7001 (replication port 8001)
```

```bash
$ ./kvcli 127.0.0.1 7003 "GET city"; ./kvcli 127.0.0.1 7003 "INFO"
```

```
VALUE Bangkok
INFO role=REPLICA node=C seq=4 last_applied=4 replicas=0 replica_addrs=- upstream=127.0.0.1:7001 (replication port 8001)
```

**ข้อมูลถูก replicate ข้าม process จริง** — `fruit`, `city`, `orders` ที่เขียนที่ A (port 7001)
ปรากฏครบถ้วนที่ B (port 7002) และ C (port 7003) โดยที่เราไม่เคยเขียนอะไรที่ B/C โดยตรงเลย และ
`seq=4` ตรงกันทั้งสามโหนดพอดี (2 SET + 2 INCR)

### ยืนยันว่า replica ปฏิเสธคำสั่งเขียน

```bash
$ ./kvcli 127.0.0.1 7002 "SET x y"
```

```
ERROR readonly replica; primary is at 127.0.0.1:7001 (replication port 8001)
```

### ยืนยันว่า replica ตายไม่ทำให้ primary ตายตาม

```bash
$ kill -9 $(cat logs/C.pid)
$ ./kvcli 127.0.0.1 7001 "SET orders 3"
$ ./kvcli 127.0.0.1 7001 "PING"
```

```
OK
PONG
```

primary A ยังตอบสนองปกติทุกอย่างแม้ replica C ตายไปกลางอากาศ log ของ A ยืนยันว่ามันตรวจพบการ
หลุดของ C ได้เองด้วย:

```
[A] replica หลุดการเชื่อมต่อ: 127.0.0.1:48558
```

และ B ที่ยังเชื่อมต่ออยู่ก็ยังได้รับข้อมูลต่อเนื่องตามปกติ (`GET orders` ที่ B ได้ `VALUE 3`)
ไม่ได้รับผลกระทบจากการที่ C ตายไปแต่อย่างใด

---

## 123.5 Crash & Recovery: WAL Replay และ Snapshot (Step 981)

### ทดสอบ 1: `kill -9` primary กลางอากาศ แล้วดูว่ากู้คืนได้จริงไหม

```bash
$ ./kvcli 127.0.0.1 7001 "COUNT"
```

```
COUNT 4
```

```bash
$ cat data/A/kv.wal
```

```
1	SET	fruit	apple
2	SET	city	Bangkok
3	SET	orders	1
4	SET	orders	2
5	SET	orders	3
6	SET	flash_sale	sold_out
```

(บรรทัดสุดท้าย `flash_sale` มาจากการสาธิต replication lag ที่จะอธิบายด้านล่าง — ทำก่อนตรงนี้แล้ว)

```bash
$ kill -9 $(cat logs/A.pid)
```

process A ตายทันทีโดยไม่มีโอกาส "ปิดอย่างสุภาพ" ใดๆ เลย ไม่มี destructor ทำงาน ไม่มีโอกาส flush
อะไรเพิ่มเติม — จำลอง crash แบบสมบูรณ์ที่สุดเท่าที่ทำได้โดยไม่ทำให้ทั้งเครื่องพัง

```bash
$ ./node --id=A --role=primary --port=7001 --replport=8001 --data=data/A --fsync=on
```

```
[A] กำลังเริ่มโหนด: role=primary data=data/A fsync=on
[recover] ไม่พบไฟล์ snapshot (เริ่มต้นด้วยสถานะว่างเปล่า)
[recover] replay WAL: apply 6 record (ข้าม 0 record เก่ากว่า snapshot), seq หลัง replay = 6
[A] replication listener ฟังอยู่ที่ 127.0.0.1:8001
[A] พร้อมทำงาน (client-port=7001, repl-port=8001)
[A] client listener ฟังอยู่ที่ 127.0.0.1:7001
```

```bash
$ ./kvcli 127.0.0.1 7001 "COUNT"; ./kvcli 127.0.0.1 7001 "GET fruit"
$ ./kvcli 127.0.0.1 7001 "GET orders"; ./kvcli 127.0.0.1 7001 "GET flash_sale"
```

```
COUNT 4
VALUE apple
VALUE 3
VALUE sold_out
```

**ข้อมูลกลับมาครบทุกตัวเป๊ะ** แม้ process จะถูกฆ่าแบบไม่มีการเตือนล่วงหน้าเลย — `replay WAL:
apply 6 record` พิสูจน์ว่าโปรแกรม replay ทุก record ที่เคย `write()` ไปแล้วสำเร็จก่อน crash

### ทดสอบ 2: SNAPSHOT แล้วดูว่า WAL ถูกบีบอัดจริง + กู้คืนจาก snapshot+tail ถูกต้อง

```bash
$ ./kvcli 127.0.0.1 7001 "SNAPSHOT"
```

```
SNAPSHOTTED seq=6
```

```bash
$ cat data/A/kv.snapshot
```

```
SNAPSHOT_SEQ	6
flash_sale	sold_out
orders	3
city	Bangkok
fruit	apple
```

```bash
$ wc -l data/A/kv.wal
```

```
0 data/A/kv.wal
```

WAL ถูก truncate เหลือว่างเปล่าจริง เพราะข้อมูลทั้งหมดถูกรวมเข้า snapshot แล้ว ลองเขียนเพิ่มแล้ว
crash+restart อีกรอบ:

```bash
$ ./kvcli 127.0.0.1 7001 "SET after_snapshot yes"
$ cat data/A/kv.wal
```

```
7	SET	after_snapshot	yes
```

```bash
$ kill -9 $(cat logs/A.pid)
$ ./node --id=A --role=primary --port=7001 --replport=8001 --data=data/A --fsync=on
```

```
[A] กำลังเริ่มโหนด: role=primary data=data/A fsync=on
[recover] โหลด snapshot สำเร็จ: 4 key, seq=6
[recover] replay WAL: apply 1 record (ข้าม 0 record เก่ากว่า snapshot), seq หลัง replay = 7
[A] replication listener ฟังอยู่ที่ 127.0.0.1:8001
[A] พร้อมทำงาน (client-port=7001, repl-port=8001)
[A] client listener ฟังอยู่ที่ 127.0.0.1:7001
```

```bash
$ ./kvcli 127.0.0.1 7001 "COUNT"; ./kvcli 127.0.0.1 7001 "GET after_snapshot"
```

```
COUNT 5
VALUE yes
```

โหลด 4 key จาก snapshot (seq=6) แล้ว replay ต่ออีก 1 record จาก WAL (seq=7) — **ผสาน snapshot
กับ WAL tail ได้ถูกต้องเป๊ะ** นี่คือกลไกเดียวกับที่ PostgreSQL เรียกว่า checkpoint + WAL replay

### ทดสอบ 3: `std::ofstream` ที่ไม่ `flush()` VS POSIX `write()` — พิสูจน์ pitfall เรื่อง fsync ด้วยของจริง

นี่คือการทดลองที่สำคัญที่สุดของหัวข้อนี้ ใช้โปรแกรมเดี่ยวๆ 2 ตัวที่ตัดเฉพาะประเด็นนี้ออกมา
ทดสอบให้เห็นชัดๆ (ไม่ปนกับความซับซ้อนของ `node.cpp` ทั้งระบบ):

```cpp
// buffered_wal_demo.cpp — ตั้งใจเขียน WAL ผิดวิธี: ใช้ std::ofstream แบบมี buffer โดยไม่ flush()
// เพื่อพิสูจน์ว่าถ้า process ถูก kill -9 ระหว่างที่ยังไม่ flush ข้อมูลจะหายจริง (เพราะ buffer
// อยู่ในหน่วยความจำของโปรเซสเราเอง ไม่เคยไปถึง kernel เลย)
#include <fstream>
#include <cstdio>
#include <unistd.h>
#include <string>

int main(int argc, char** argv) {
    if (argc < 2) { fprintf(stderr, "usage: %s <wal-file>\n", argv[0]); return 1; }
    std::ofstream ofs(argv[1], std::ios::app); // std::ofstream: มี internal buffer (libstdc++ filebuf)
    for (int i = 1; i <= 5; i++) {
        ofs << i << "\tSET\tkey" << i << "\tvalue" << i << "\n"; // *** ไม่เรียก flush() ***
        printf("[buffered] เขียน record %d ลง ofstream buffer แล้ว (ยังไม่ flush ไป kernel)\n", i);
        fflush(stdout); // อันนี้ flush stdout เพื่อให้เห็น log ทันที ไม่เกี่ยวกับ ofs
        usleep(200000);
    }
    ofs.flush();
    printf("[buffered] เขียนครบ 5 record และ flush() แล้วตามปกติ (โปรแกรมจบโดยไม่ถูก kill)\n");
    return 0;
}
```

```cpp
// durable_wal_demo.cpp — เขียน WAL ด้วย POSIX write() ตรงๆ (ไม่ผ่าน buffer ของโปรเซสเราเอง)
// เปรียบเทียบกับ buffered_wal_demo.cpp: แม้ "ไม่" เรียก fsync() เลย ข้อมูลที่ write() ไปแล้วก็จะ
// รอดจากการถูก kill -9 เสมอ เพราะมันอยู่ใน page cache ของ kernel ไปแล้ว ไม่ใช่ของโปรเซส
// fsync() มีไว้ป้องกันกรณี "เครื่องดับ/OS crash" เท่านั้น ซึ่งจำลองบนเครื่องเดียวกันนี้ไม่ได้จริง
#include <fcntl.h>
#include <unistd.h>
#include <cstdio>
#include <cstring>
#include <string>

int main(int argc, char** argv) {
    if (argc < 2) { fprintf(stderr, "usage: %s <wal-file> [fsync]\n", argv[0]); return 1; }
    bool doFsync = argc > 2 && std::string(argv[2]) == "fsync";
    int fd = open(argv[1], O_WRONLY | O_CREAT | O_APPEND, 0644);
    if (fd < 0) { perror("open"); return 1; }
    for (int i = 1; i <= 5; i++) {
        std::string line = std::to_string(i) + "\tSET\tkey" + std::to_string(i) +
                            "\tvalue" + std::to_string(i) + "\n";
        ssize_t n = write(fd, line.data(), line.size());
        (void)n;
        if (doFsync) fsync(fd);
        printf("[durable%s] write()%s record %d ให้ kernel แล้ว\n",
               doFsync ? "+fsync" : "", doFsync ? "+fsync()" : "", i);
        fflush(stdout);
        usleep(200000);
    }
    close(fd);
    printf("[durable] เขียนครบ 5 record ตามปกติ (โปรแกรมจบโดยไม่ถูก kill)\n");
    return 0;
}
```

รันจริง: ปล่อยให้แต่ละโปรแกรมเขียนไปได้สัก 2-3 record แล้ว `kill -9` กลางคัน:

```bash
$ ./buffered_wal_demo /tmp/wal_buffered.demo &
$ sleep 0.45; kill -9 $!
$ cat /tmp/wal_buffered.demo
```

**ผลลัพธ์จริง**:

```
[buffered] เขียน record 1 ลง ofstream buffer แล้ว (ยังไม่ flush ไป kernel)
[buffered] เขียน record 2 ลง ofstream buffer แล้ว (ยังไม่ flush ไป kernel)
[buffered] เขียน record 3 ลง ofstream buffer แล้ว (ยังไม่ flush ไป kernel)
Killed
```

```bash
$ cat /tmp/wal_buffered.demo
$ wc -c /tmp/wal_buffered.demo
```

```
(ไม่มีเนื้อหาอะไรเลย)
0 /tmp/wal_buffered.demo
```

**ไฟล์ว่างเปล่าสนิท 0 ไบต์** — แม้โปรแกรมจะ "รายงาน" ว่าเขียนไปแล้วถึง 3 record ก็ตาม! ข้อมูล
ทั้งหมดหายไปเพราะมันติดอยู่ใน buffer ของ `std::ofstream` ที่อยู่ใน**หน่วยความจำของโปรเซส** และ
ไม่เคยไปถึง kernel เลยก่อนที่โปรเซสจะถูกฆ่า

```bash
$ ./durable_wal_demo /tmp/wal_durable.demo &
$ sleep 0.45; kill -9 $!
$ cat /tmp/wal_durable.demo
```

**ผลลัพธ์จริง**:

```
[durable] write() record 1 ให้ kernel แล้ว
[durable] write() record 2 ให้ kernel แล้ว
[durable] write() record 3 ให้ kernel แล้ว
Killed
```

```bash
$ cat /tmp/wal_durable.demo
```

```
1	SET	key1	value1
2	SET	key2	value2
3	SET	key3	value3
```

**ข้อมูลทั้ง 3 record รอดครบ** แม้จะไม่ได้เรียก `fsync()` เลยสักครั้ง! เพราะ `write()` ส่งข้อมูล
เข้า kernel page cache ทันที ซึ่งเป็นหน่วยความจำของ**เคอร์เนล** ไม่ใช่ของโปรเซสเรา `kill -9`
ทำลายแค่โปรเซส เคอร์เนลไม่ได้รับผลกระทบใดๆ

> **สรุปที่ต้องจำให้แม่น**: `fsync()` **ไม่ได้** มีไว้ป้องกัน "โปรแกรมถูก kill" — การใช้ `write()`
> ธรรมดา (แทน buffered stream ที่ไม่ flush) ก็ทนต่อกรณีนั้นได้อยู่แล้ว `fsync()` มีไว้ป้องกัน
> กรณี **ไฟดับกะทันหันหรือ OS crash** ที่แม้แต่ kernel page cache ก็จะหายไปด้วย (เพราะ RAM ทั้งหมด
> เป็น volatile) ซึ่งเป็นสถานการณ์ที่**จำลองบนเครื่องเดียวกันนี้ไม่ได้จริง** เพราะจะทำให้ทั้ง
> session รวมถึงเครื่องมือที่กำลังทดสอบอยู่พังไปด้วย — เป็นข้อจำกัดที่ต้องยอมรับตรงๆ และเป็นเหตุผล
> ที่ `node.cpp` ยังคง `fsync()` ทุกครั้งเป็นค่า default (ปลอดภัยไว้ก่อนสำหรับ production จริง)
> แม้บนเครื่องทดสอบนี้เราจะพิสูจน์ไม่ได้ว่ามันต่างจากไม่ fsync จริงๆ ก็ตาม

---

## 123.6 Manual Failover, Replication Lag และ Split-Brain (Step 982)

### Replication Lag: "Eventual Consistency Window" ที่จับต้องได้จริง

คำสั่ง `SETDELAY` ทำให้ replica จงใจหน่วงเวลาก่อน apply record ที่ replicate เข้ามา — จำลอง
เครือข่ายที่ช้าหรือ replica ที่ทำงานหนักอยู่:

```bash
$ ./kvcli 127.0.0.1 7002 "SETDELAY 1500"
$ ./kvcli 127.0.0.1 7001 "SET flash_sale sold_out"
$ ./kvcli 127.0.0.1 7002 "GET flash_sale"     # อ่านทันที
```

```
OK delay=1500ms
OK
NOT_FOUND
```

```bash
$ sleep 2
$ ./kvcli 127.0.0.1 7002 "GET flash_sale"     # อ่านอีกครั้งหลังรอ
```

```
VALUE sold_out
```

**นี่คือ "Eventual Consistency Window" ตัวจริง**: ทันทีที่ primary ตอบ `OK` กลับไปให้ client
ข้อมูลนั้น**ยังไม่ถึง** replica เลย — ถ้ามี client อีกตัวอ่านจาก replica ในช่วงเวลานั้นพอดี
(หรือแม้แต่ในสถานการณ์จริงที่ไม่ได้ตั้งใจหน่วงเลย เพียงแค่เครือข่ายมี latency ตามธรรมชาติ) จะ
เห็นข้อมูลเก่า — Redis, MySQL replication, DynamoDB (โหมด eventually consistent read) ทุกระบบที่
ใช้ async replication มีหน้าต่างแบบนี้เหมือนกันหมด ต่างกันแค่ขนาดของหน้าต่าง (มักเป็นมิลลิวินาที
ไม่ใช่วินาทีแบบที่จงใจทำให้เห็นชัดในการสาธิตนี้)

### Manual Failover: `PROMOTE` และ `REPLICAOF`

สถานการณ์: primary A ตายจริง (kill -9 แบบถาวร) ตอนนี้ B เป็น replica ที่ **หยุด replicate มา
สักพักแล้ว** (เพราะ A เคย restart ไปสองรอบก่อนหน้านี้ และ replica **ไม่ reconnect อัตโนมัติ** —
เป็นข้อจำกัดที่ตั้งใจปล่อยไว้ ดูหัวข้อ pitfalls) B จึงมีข้อมูลถึงแค่ `seq=6` ไม่มี
`after_snapshot` (`seq=7`) เลย:

```bash
$ ./kvcli 127.0.0.1 7001 "COUNT"          # A (ตัวจริงล่าสุด) มี 5 key
```

```
COUNT 5
```

```bash
$ kill -9 $(cat logs/A.pid)                # primary ตายจริง (ถาวร)
$ ./kvcli 127.0.0.1 7002 "COUNT"           # B มีแค่ข้อมูลเท่าที่เคยได้รับ
$ ./kvcli 127.0.0.1 7002 "GET after_snapshot"
```

```
COUNT 4
NOT_FOUND
```

```bash
$ ./kvcli 127.0.0.1 7002 "PROMOTE"
$ ./kvcli 127.0.0.1 7002 "INFO"
```

```
PROMOTED
INFO role=PRIMARY node=B seq=6 last_applied=6 replicas=0 replica_addrs=- upstream=-
```

> **`after_snapshot` หายไปจากคลัสเตอร์แล้วจริงๆ ไม่มีโหนดไหนเหลือมันอยู่เลย** — นี่คือ
> **Lost Write ของจริง** ไม่ใช่แค่ทฤษฎี: client ได้รับ `OK` กลับไปตอน `SET after_snapshot yes`
> (primary apply + log WAL สำเร็จ) แต่เพราะ primary ตอบ client **โดยไม่รอ ack จาก replica ก่อน**
> (async replication, at-most-1-copy durability) เมื่อ primary ตายไปและไม่มี replica ตัวไหนเคย
> ได้รับ record นั้นเลย ข้อมูลก็หายไปพร้อมกับ primary ตัวเก่าอย่างถาวร — นี่คือ trade-off ตรงๆ
> ของการเลือก **Availability/Latency เหนือ Durability สูงสุด** ซึ่งเชื่อมโยงกับ CAP Theorem ที่
> จะอธิบายด้านล่าง

ตอนนี้ให้ C (ที่ตายไปก่อนหน้านี้) กลับมา — restart ด้วยค่าตั้งต้นเดิม (ชี้ไปที่ A ที่ตายแล้ว)
เพื่อดูว่าจะเกิดอะไรขึ้น แล้วค่อยแก้ด้วย `REPLICAOF`:

```bash
$ ./node --id=C --role=replica --port=7003 --replport=8003 --data=data/C --fsync=on \
    --primary-host=127.0.0.1 --primary-replport=8001 --primary-port=7001
```

```
[C] กำลังเริ่มโหนด: role=replica data=data/C fsync=on
[recover] โหลด snapshot สำเร็จ: 0 key, seq=0
[recover] replay WAL: apply 4 record (ข้าม 0 record เก่ากว่า snapshot), seq หลัง replay = 4
[C] replication listener ฟังอยู่ที่ 127.0.0.1:8003
[C] พร้อมทำงาน (client-port=7003, repl-port=8003)
[C] client listener ฟังอยู่ที่ 127.0.0.1:7003
[C] เชื่อมต่อ upstream 127.0.0.1:8001 ไม่สำเร็จ
```

**ตามคาด**: C กู้คืนข้อมูลของตัวเองจาก WAL ได้ปกติ (4 record จากก่อนหน้านี้) แต่เชื่อมต่อ upstream
เดิม (port 8001 ของ A ที่ตายแล้ว) ไม่สำเร็จ ต้องสั่งย้ายไปเกาะ B ด้วยมือ:

```bash
$ ./kvcli 127.0.0.1 7003 "REPLICAOF 127.0.0.1 8002 7002"
```

```
REPLICATING
```

```
[C] FULLSYNC จาก 127.0.0.1:8002 สำเร็จ: 4 key, seq เริ่มต้น=6
```

```bash
$ ./kvcli 127.0.0.1 7002 "SET post_failover ok"
$ sleep 0.2
$ ./kvcli 127.0.0.1 7003 "GET post_failover"
```

```
OK
VALUE ok
```

**Failover สำเร็จสมบูรณ์**: B เป็น primary คนใหม่, C ตามทันและรับข้อมูลใหม่ต่อเนื่องได้ปกติ ทั้ง
กระบวนการทำผ่านคำสั่ง `PROMOTE`/`REPLICAOF` โดย**ไม่ต้อง restart process ใดเลยแม้แต่ตัวเดียว**

### Split-Brain: ผลลัพธ์จริงเมื่อ Primary เก่า "ฟื้นคืนชีพ" โดยไม่รู้ว่าโดน Failover ไปแล้ว

นี่คือสถานการณ์ operational ที่เกิดขึ้นจริงบ่อยมาก: มีคน (หรือ script อัตโนมัติ) restart เครื่อง
ของ primary เก่าโดยไม่รู้ว่าคลัสเตอร์ทำ failover ไปแล้ว:

```bash
$ ./node --id=A --role=primary --port=7001 --replport=8001 --data=data/A --fsync=on
```

```
[A] กำลังเริ่มโหนด: role=primary data=data/A fsync=on
[recover] โหลด snapshot สำเร็จ: 4 key, seq=6
[recover] replay WAL: apply 1 record (ข้าม 0 record เก่ากว่า snapshot), seq หลัง replay = 7
[A] replication listener ฟังอยู่ที่ 127.0.0.1:8001
[A] พร้อมทำงาน (client-port=7001, repl-port=8001)
[A] client listener ฟังอยู่ที่ 127.0.0.1:7001
```

```bash
$ ./kvcli 127.0.0.1 7001 "INFO"
```

```
INFO role=PRIMARY node=A seq=7 last_applied=0 replicas=0 replica_addrs=- upstream=-
```

**A เชื่อว่าตัวเองยังเป็น PRIMARY** — เพราะมันถูกสั่งให้ start ด้วย `--role=primary` เหมือนเดิม
ไม่มีกลไกอะไรบอกมันว่า "มีคนอื่นถูกเลื่อนเป็น primary แทนคุณไปแล้ว" (นี่คือสิ่งที่ Raft/Paxos
แก้ด้วย **term number** ที่ทุกโหนดเห็นตรงกัน — ระบบของเราไม่มีกลไกแบบนั้นเลย):

```bash
$ ./kvcli 127.0.0.1 7001 "GET orders"        # A
$ ./kvcli 127.0.0.1 7002 "GET orders"        # B (primary ตัวจริง)
```

```
VALUE 3
VALUE 3
```

ตอนนี้ยังตรงกันอยู่ (เพราะ orders ไม่ได้ถูกแก้ตั้งแต่ก่อน failover) แต่พอมี client (พลาด/ไม่รู้
สถานการณ์) เขียนตรงไปที่ A:

```bash
$ ./kvcli 127.0.0.1 7001 "SET orders 999"
$ ./kvcli 127.0.0.1 7001 "SET rogue_write from_zombie_A"
```

```
OK
OK
```

```bash
$ ./kvcli 127.0.0.1 7001 "GET orders"; ./kvcli 127.0.0.1 7002 "GET orders"
$ ./kvcli 127.0.0.1 7001 "GET rogue_write"
$ ./kvcli 127.0.0.1 7002 "GET rogue_write"; ./kvcli 127.0.0.1 7003 "GET rogue_write"
```

**ผลลัพธ์จริง**:

```
VALUE 999
VALUE 3
VALUE from_zombie_A
NOT_FOUND
NOT_FOUND
```

**นี่คือ Split-Brain ตัวจริง ข้อมูล diverge กันจริงๆ**: `orders` เป็น `999` บน A แต่เป็น `3` บน
B (primary ตัวจริงของคลัสเตอร์) ส่วน `rogue_write` มีอยู่เฉพาะบน A เท่านั้น — ไม่มีทางไหนเลยที่
ระบบจะ "รวม" ข้อมูลทั้งสองสายกลับเป็นหนึ่งเดียวได้โดยอัตโนมัติ เพราะทั้งสองสายไม่รู้จักกันเลย
(A ไม่มี replica ต่ออยู่แม้แต่ตัวเดียว, B/C ไม่รู้ด้วยซ้ำว่า A ฟื้นคืนชีพมาแล้ว) วิธีแก้ในโลกจริง
คือ**ต้องมีคนตัดสินใจด้วยมือ**ว่าจะทิ้งข้อมูลฝั่งไหน (มักทิ้งฝั่ง minority/stale) แล้ว
reconcile ด้วยมือ — งานที่น่าเบื่อและเสี่ยงต่อการเข้าใจผิดที่สุดของ operator ทุกคนที่เคยเจอ
เหตุการณ์นี้จริง

### เชื่อมโยงกับ CAP Theorem

**CAP Theorem** (Eric Brewer, 2000) บอกว่าระบบแบบกระจาย (distributed system) ไม่สามารถมีครบ
ทั้ง 3 สมบัตินี้พร้อมกันได้เมื่อเกิด **Network Partition (P)**:

- **Consistency (C)**: ทุกโหนดเห็นข้อมูลชุดเดียวกันเสมอ (เหมือนมีสำเนาเดียว)
- **Availability (A)**: ทุก request ที่ไม่ตาย ต้องได้รับ response กลับเสมอ (แม้บางโหนดจะขาด
  การเชื่อมต่อกับโหนดอื่น)
- **Partition Tolerance (P)**: ระบบยังทำงานต่อได้แม้เครือข่ายระหว่างบางโหนดจะขาด

เนื่องจาก network partition เกิดขึ้นได้เสมอในระบบจริง (สาย LAN หลุด, packet หาย) จึงต้องเลือก
ระหว่าง **C** กับ **A** เมื่อ partition เกิดขึ้นจริง — โปรเจกต์นี้เลือก **AP** (เอียงไปทาง
Availability): primary ตอบ client ทันทีโดยไม่รอให้ replica ยืนยันก่อน (จึงมี availability/latency
ดี) แลกมาด้วยการที่ระหว่างเกิด partition หรือ failover ข้อมูลอาจไม่ตรงกันข้ามโหนด (สูญเสีย
Consistency ชั่วคราว หรืออย่างที่เห็นในการทดลอง คือสูญเสียถาวรถ้าไม่ reconcile)

| Consistency Model | นิยาม | ระบบตัวอย่าง | ระบบของเราตอนนี้เป็นแบบนี้ไหม |
|---|---|---|---|
| **Strong Consistency** | อ่านค่าล่าสุดเสมอทุกโหนด | Google Spanner, etcd (Raft) | ไม่ — replica อาจตอบข้อมูลเก่ากว่า |
| **Eventual Consistency** | ถ้าหยุดเขียน สักพักทุกโหนดจะตรงกันเอง (ถ้าไม่มี split-brain) | DynamoDB, Cassandra (default) | ใช่ — ตราบใดที่ไม่มี split-brain |
| **Causal Consistency** | เห็นลำดับเหตุ-ผลตรงกัน แต่ไม่จำเป็นต้อง real-time | บาง config ของ MongoDB | ไม่ได้ implement |
| **Read-Your-Writes** | client เห็นค่าที่ตัวเองเพิ่งเขียนเสมอ | ต้อง route read ไป primary หรือ sticky session | ไม่ได้ implement (ดูแบบฝึกหัดข้อ 4) |

| กลยุทธ์ Replication | ความเร็วเขียน | ความปลอดภัยของข้อมูล | ระบบของเราใช้แบบไหน |
|---|---|---|---|
| **Async single-leader** (ที่เราใช้) | เร็วที่สุด (ไม่รอ replica) | เสี่ยง lost write ตอน failover มากที่สุด | ✅ ใช้แบบนี้ |
| **Sync single-leader** (รอทุก replica ยืนยันก่อนตอบ) | ช้าที่สุด (รอ replica ช้าที่สุด) | ไม่มี lost write เลย แต่ replica ตายตัวเดียวทำให้เขียนไม่ได้เลย | ไม่ได้ implement |
| **Quorum (W+R > N)** | กลางๆ (รอแค่เสียงข้างมาก) | ปลอดภัยขึ้นมาก โดยไม่ต้องรอทุกตัว | ไม่ได้ implement (ดูแบบฝึกหัดข้อ 3) |
| **Multi-leader** | เขียนได้จากหลายจุดพร้อมกัน | ต้องมี conflict resolution (ซับซ้อนมาก) | ไม่ได้ implement |
| **Consensus-based** (Raft/Paxos) | ช้ากว่า async แต่ไม่มี split-brain | ปลอดภัยสูงสุด, ป้องกัน split-brain ด้วย term/quorum | ไม่ได้ implement (คือสิ่งที่ Part นี้ตั้งใจ**ไม่ทำ**) |

---

## 123.7 Client Library, SIGPIPE Pitfall และ At-Least-Once Delivery (Step 983)

### `kv_client.hpp` — Client Library เบื้องต้น

```cpp
// kv_client.hpp — client library เล็กๆ สำหรับคุยกับ node.cpp ผ่านโปรโตคอลข้อความล้วน
// ใช้เป็นฐานของ kvcli.cpp (เครื่องมือ command-line) และ retry_demo.cpp (สาธิต at-least-once)
#pragma once
#include "common.hpp"
#include <optional>
#include <string>

class KVClient {
    Socket sock_;
    std::optional<LineConn> conn_;
    std::string host_;
    int port_ = 0;
public:
    bool connect(const std::string& host, int port, int timeoutMs = 0) {
        sock_ = connectTo(host, port);
        if (!sock_.valid()) return false;
        if (timeoutMs > 0) setRecvTimeout(sock_.get(), timeoutMs);
        conn_.emplace(sock_.get());
        host_ = host; port_ = port;
        return true;
    }
    bool connected() const { return sock_.valid(); }

    // ส่งคำสั่งหนึ่งบรรทัด รอ response หนึ่งบรรทัดกลับมา คืน nullopt ถ้า timeout/หลุด
    std::optional<std::string> command(const std::string& cmd) {
        if (!conn_) return std::nullopt;
        if (!conn_->sendLine(cmd)) return std::nullopt;
        return conn_->readLine();
    }

    void disconnect() { conn_.reset(); sock_.closeNow(); }
};
```

```cpp
// kvcli.cpp — client CLI ง่ายๆ: kvcli <host> <port> "<CMD>"  หรือไม่ใส่ CMD เพื่อเข้าโหมด
// interactive (อ่านคำสั่งทีละบรรทัดจาก stdin จนกว่าจะ QUIT/EOF)
#include "kv_client.hpp"
#include <iostream>
#include <cstdlib>

int main(int argc, char** argv) {
    if (argc < 3) {
        fprintf(stderr, "usage: %s <host> <port> [command...]\n", argv[0]);
        return 1;
    }
    std::string host = argv[1];
    int port = atoi(argv[2]);

    KVClient client;
    if (!client.connect(host, port)) {
        fprintf(stderr, "เชื่อมต่อ %s:%d ไม่สำเร็จ\n", host.c_str(), port);
        return 1;
    }

    if (argc > 3) {
        std::string cmd;
        for (int i = 3; i < argc; i++) { if (i > 3) cmd += " "; cmd += argv[i]; }
        auto r = client.command(cmd);
        if (!r) { fprintf(stderr, "ไม่ได้รับ response (timeout หรือหลุด)\n"); return 1; }
        printf("%s\n", r->c_str());
        return 0;
    }

    std::string line;
    while (std::getline(std::cin, line)) {
        if (line.empty()) continue;
        auto r = client.command(line);
        if (!r) { printf("(connection closed)\n"); break; }
        printf("%s\n", r->c_str());
        if (line == "QUIT" || line == "quit") break;
    }
    return 0;
}
```

นี่คือ client library ที่ใช้ทดสอบทุกอย่างตลอด Part นี้มาแล้ว — เรียบง่ายโดยตั้งใจ (ต่อ TCP,
ส่งบรรทัด, รับบรรทัด) เพื่อให้เน้นไปที่แนวคิดของระบบ ไม่ใช่ความซับซ้อนของ client SDK

### Pitfall: `SIGPIPE` ฆ่า Primary ทั้งกระบวนการเมื่อ Replica หลุดกลางคัน

พฤติกรรม default ของสัญญาณ `SIGPIPE` ใน POSIX คือ **terminate โปรเซสทันที** สัญญาณนี้เกิดขึ้น
เมื่อโปรแกรมเรียก `send()`/`write()` ไปยัง socket ที่อีกฝั่งปิดไปแล้ว — สถานการณ์ที่เกิดขึ้นได้
บ่อยมากเวลา replica หลุดกลางคันแล้ว primary ยัง `broadcast()` ต่อไปโดยไม่รู้ตัว ลองพิสูจน์ด้วย
โปรแกรมเดี่ยวๆ:

```cpp
// sigpipe_demo.cpp — สาธิต pitfall เรื่อง SIGPIPE ตอน replica หลุดกลางคัน:
// รับ 1 connection แล้ว send() ข้อมูลให้เรื่อยๆ ทุก 300ms ถ้า client ปิดการเชื่อมต่อไปแล้วแต่เรายัง
// send() ต่อโดยไม่ป้องกัน จะโดน SIGPIPE ฆ่าทั้งโปรเซสทันที (พฤติกรรม default ของ SIGPIPE คือ
// terminate) — รันด้วย argv[1]="fixed" เพื่อใช้ MSG_NOSIGNAL แล้วเทียบผลลัพธ์จริง
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <unistd.h>
#include <cstdio>
#include <cstring>
#include <cerrno>
#include <string>
#include <thread>
#include <chrono>

int main(int argc, char** argv) {
    bool fixed = argc > 1 && std::string(argv[1]) == "fixed";
    int listenFd = socket(AF_INET, SOCK_STREAM, 0);
    int opt = 1;
    setsockopt(listenFd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));
    sockaddr_in addr{};
    addr.sin_family = AF_INET;
    addr.sin_port = htons(9999);
    inet_pton(AF_INET, "127.0.0.1", &addr.sin_addr);
    if (bind(listenFd, (sockaddr*)&addr, sizeof(addr)) < 0) { perror("bind"); return 1; }
    listen(listenFd, 1);
    printf("[sigpipe-demo] mode=%s: listening on 127.0.0.1:9999\n",
           fixed ? "fixed(MSG_NOSIGNAL)" : "broken(no-protection)");
    fflush(stdout);

    int fd = accept(listenFd, nullptr, nullptr);
    printf("[sigpipe-demo] accepted connection (fd=%d)\n", fd);
    fflush(stdout);

    for (int i = 0; i < 15; i++) {
        std::this_thread::sleep_for(std::chrono::milliseconds(300));
        std::string line = "ping " + std::to_string(i) + "\n";
        ssize_t n = fixed ? send(fd, line.data(), line.size(), MSG_NOSIGNAL)
                           : send(fd, line.data(), line.size(), 0);
        printf("[sigpipe-demo] send() #%d -> %zd (errno=%s)\n",
               i, n, n < 0 ? strerror(errno) : "-");
        fflush(stdout);
    }
    printf("[sigpipe-demo] วนลูปครบโดยไม่ถูกฆ่า\n");
    return 0;
}
```

รันเวอร์ชัน "พัง" (ไม่ป้องกัน) แล้วให้ client ปิดการเชื่อมต่อทันทีหลังต่อเสร็จ:

```bash
$ ./sigpipe_demo broken &
$ exec 3<>/dev/tcp/127.0.0.1/9999; exec 3<&-; exec 3>&-   # ต่อแล้วปิดทันที
```

**ผลลัพธ์จริง**:

```
[sigpipe-demo] mode=broken(no-protection): listening on 127.0.0.1:9999
[sigpipe-demo] accepted connection (fd=4)
[sigpipe-demo] send() #0 -> 7 (errno=-)
[1]+  Broken pipe             ./sigpipe_demo broken
```

**โปรเซสถูกฆ่าทันทีตั้งแต่ `send()` ครั้งที่สอง** (ครั้งแรกยังสำเร็จเพราะ kernel ยังไม่ทันรู้ว่า
อีกฝั่งปิดไปแล้ว) shell รายงาน `Broken pipe` ซึ่งคือสัญญาณบอกว่าโปรเซสตายด้วย `SIGPIPE`
(exit code 141 = 128+13, 13 คือเลขของ `SIGPIPE`)

รันเวอร์ชัน "แก้แล้ว" (ใช้ `MSG_NOSIGNAL`) แบบเดียวกัน:

```bash
$ ./sigpipe_demo fixed &
$ exec 3<>/dev/tcp/127.0.0.1/9999; exec 3<&-; exec 3>&-
```

**ผลลัพธ์จริง**:

```
[sigpipe-demo] mode=fixed(MSG_NOSIGNAL): listening on 127.0.0.1:9999
[sigpipe-demo] accepted connection (fd=4)
[sigpipe-demo] send() #0 -> 7 (errno=-)
[sigpipe-demo] send() #1 -> -1 (errno=Broken pipe)
[sigpipe-demo] send() #2 -> -1 (errno=Broken pipe)
[sigpipe-demo] send() #3 -> -1 (errno=Broken pipe)
```

**process ไม่ตายแล้ว** — `send()` แค่คืนค่า `-1` พร้อม `errno = EPIPE` (ข้อความ "Broken pipe")
ให้เราจัดการเองตามปกติ (ในกรณีของ `ReplicaRegistry::broadcast()` คือถูกเพิกเฉยแล้วปล่อยให้
subscriber handler thread ตรวจจับและลบออกจาก registry เอง) นี่คือเหตุผลที่ `common.hpp` และ
`node.cpp` ใช้ `MSG_NOSIGNAL` ในทุกจุดที่ `send()` มาตั้งแต่ต้น

### At-Least-Once Delivery: เมื่อ Client Retry แล้วคำสั่งไม่ Idempotent

โปรโตคอลของเราเป็น **At-Least-Once**: ถ้า client ไม่ได้รับ response (timeout) มันไม่รู้ว่า
คำสั่งไปถึง server จริงหรือเปล่า — วิธีที่ปลอดภัยที่สุดสำหรับ client คือ **retry ซ้ำ** ซึ่งถูกต้อง
สำหรับคำสั่งที่ **idempotent** (ทำซ้ำกี่ครั้งผลลัพธ์เหมือนเดิม เช่น `SET key value`) แต่จะ
**อันตรายมาก** สำหรับคำสั่งที่ไม่ idempotent เช่น `INCR`

`node.cpp` มี debug hook ชื่อ `DROPACK` ที่จำลอง "ack หายระหว่างทาง" แบบควบคุมได้บนเครื่องเดียว
(server apply คำสั่งจริง แต่ตั้งใจไม่ตอบกลับ) — เป็นวิธีจำลอง fault-injection ที่ซื่อสัตย์
(บอกตรงๆ ว่าเป็นเครื่องมือทดสอบ ไม่ใช่ความล้มเหลวของเครือข่ายจริง เพราะ loopback ทำให้แพ็กเก็ต
หายเองตามธรรมชาติแทบไม่ได้เลย):

```cpp
// retry_demo.cpp — สาธิต "at-least-once delivery" ของ client library เมื่อ ack หายไป:
// 1) ใช้ INCR ธรรมดา + DROPACK (debug hook ฝั่ง server) จำลอง ack หาย -> client timeout -> retry
//    -> นับซ้ำ (ไม่ idempotent) -> ค่าสุดท้ายผิดจากที่ควรจะเป็น
// 2) ใช้ INCR ... REQID <id> แบบเดียวกัน แต่ใส่ idempotency key -> retry ปลอดภัย ไม่นับซ้ำ
#include "kv_client.hpp"
#include <cstdio>
#include <cstdlib>

// ส่งคำสั่งพร้อม retry-on-timeout อัตโนมัติ 1 ครั้ง คืนค่า response สุดท้ายที่ได้ (จาก retry หรือครั้งแรก)
static std::string sendWithRetry(KVClient& c, const std::string& firstCmd, const std::string& retryCmd) {
    auto r1 = c.command(firstCmd);
    if (r1) return *r1; // ได้ ack ปกติ ไม่ต้อง retry
    printf("  -> ไม่ได้รับ response ภายในเวลาที่กำหนด (สันนิษฐานว่า ack หาย) : กำลัง retry...\n");
    auto r2 = c.command(retryCmd);
    return r2 ? *r2 : std::string("(retry ก็ timeout อีก)");
}

int main(int argc, char** argv) {
    std::string host = argc > 1 ? argv[1] : "127.0.0.1";
    int port = argc > 2 ? atoi(argv[2]) : 7001;

    printf("=== ส่วนที่ 1: INCR ธรรมดา + retry (ไม่ปลอดภัย) ===\n");
    {
        KVClient c;
        c.connect(host, port, /*timeoutMs=*/500);
        c.command("DEL unsafe_counter");
        printf("ส่ง: INCR unsafe_counter DROPACK\n");
        std::string resp = sendWithRetry(c, "INCR unsafe_counter DROPACK", "INCR unsafe_counter");
        printf("  -> response จาก retry: %s\n", resp.c_str());
        auto final = c.command("GET unsafe_counter");
        printf("ค่าสุดท้ายของ unsafe_counter = %s  (ควรจะเป็น VALUE 1 ถ้า retry ปลอดภัย)\n\n",
               final ? final->c_str() : "(ไม่ทราบ)");
    }

    printf("=== ส่วนที่ 2: INCR + REQID (idempotency key) + retry (ปลอดภัย) ===\n");
    {
        KVClient c;
        c.connect(host, port, /*timeoutMs=*/500);
        c.command("DEL safe_counter");
        printf("ส่ง: INCR safe_counter REQID demo-req-1 DROPACK\n");
        std::string resp = sendWithRetry(c, "INCR safe_counter REQID demo-req-1 DROPACK",
                                             "INCR safe_counter REQID demo-req-1");
        printf("  -> response จาก retry: %s\n", resp.c_str());
        auto final = c.command("GET safe_counter");
        printf("ค่าสุดท้ายของ safe_counter = %s  (ควรจะเป็น VALUE 1 เสมอ ไม่ว่าจะ retry กี่ครั้ง)\n",
               final ? final->c_str() : "(ไม่ทราบ)");
    }
    return 0;
}
```

รันจริง:

```bash
$ ./retry_demo 127.0.0.1 7002
```

```
=== ส่วนที่ 1: INCR ธรรมดา + retry (ไม่ปลอดภัย) ===
ส่ง: INCR unsafe_counter DROPACK
  -> ไม่ได้รับ response ภายในเวลาที่กำหนด (สันนิษฐานว่า ack หาย) : กำลัง retry...
  -> response จาก retry: VALUE 2
ค่าสุดท้ายของ unsafe_counter = VALUE 2  (ควรจะเป็น VALUE 1 ถ้า retry ปลอดภัย)

=== ส่วนที่ 2: INCR + REQID (idempotency key) + retry (ปลอดภัย) ===
ส่ง: INCR safe_counter REQID demo-req-1 DROPACK
  -> ไม่ได้รับ response ภายในเวลาที่กำหนด (สันนิษฐานว่า ack หาย) : กำลัง retry...
  -> response จาก retry: VALUE 1
ค่าสุดท้ายของ safe_counter = VALUE 1  (ควรจะเป็น VALUE 1 เสมอ ไม่ว่าจะ retry กี่ครั้ง)
```

**ผลลัพธ์ตรงตามที่ทฤษฎีทำนายเป๊ะ**: `unsafe_counter` ถูกนับซ้ำกลายเป็น `2` ทั้งที่ผู้ใช้ตั้งใจ
INCR แค่ครั้งเดียว เพราะ server apply สำเร็จทั้งสองครั้ง (ครั้งแรกที่ถูก `DROPACK` และครั้งที่
retry) แต่ `safe_counter` ยังคงเป็น `1` เพราะ `doIncrIdempotent()` (จากหัวข้อ 123.2) จำ
`REQID` ที่เคยเห็นแล้ว และคืนผลลัพธ์เดิมโดยไม่ apply ซ้ำ — **Idempotency Key คือรูปแบบมาตรฐาน
ที่ระบบชำระเงินจริง (เช่น Stripe API) ใช้แก้ปัญหานี้เป๊ะๆ**

---

## 123.8 Common Pitfalls, แบบฝึกหัด และสรุปท้ายบท (Step 984)

### ข้อผิดพลาดที่พบบ่อย (Common Pitfalls)

1. **เข้าใจผิดว่า `fsync()` ป้องกัน "โปรเซสถูก kill"** — ตามที่พิสูจน์แล้วในหัวข้อ 123.5:
   `write()` ธรรมดา (ไม่มี `fsync()`) ก็ทนต่อ `kill -9` ได้อยู่แล้ว เพราะข้อมูลไปถึง kernel page
   cache แล้ว `fsync()` มีไว้ป้องกันไฟดับ/OS crash เท่านั้น — ความเข้าใจผิดนี้นำไปสู่การเขียน
   ระบบที่ปลอดภัยเกินจำเป็น (fsync ทุก write ทำให้ throughput ต่ำ) โดยไม่ได้ป้องกันอะไรเพิ่ม
   จากที่ `write()` ให้ไว้แล้วในสถานการณ์ที่พบบ่อยที่สุด
2. **ใช้ `std::ofstream`/`FILE*` แบบ buffered เขียน WAL โดยไม่ `flush()`** — ดูเผินๆ เหมือน
   ทำงานถูกต้อง (โปรแกรมทดสอบปกติจะจบแบบ flush ให้อัตโนมัติตอน destructor) แต่จะเสียข้อมูลจริง
   ทันทีที่ process ถูก kill ก่อนถึงจุด flush ตามที่พิสูจน์แล้วด้วย `buffered_wal_demo`
3. **`send()` โดยไม่ใส่ `MSG_NOSIGNAL` (หรือไม่ `signal(SIGPIPE, SIG_IGN)`)** — เมื่อ replica
   หรือ client หลุดกลางคัน `send()` ครั้งถัดไปจะโดน `SIGPIPE` ฆ่าทั้งโปรเซสทันทีตามค่า default
   (พิสูจน์แล้วใน `sigpipe_demo`) นี่คือบั๊กที่อันตรายมากเพราะทำให้ **primary ทั้งตัวล่มเพียง
   เพราะ replica ตัวเดียวหลุด** — ทางแก้คือใส่ `MSG_NOSIGNAL` ทุกจุดที่ `send()`
4. **Split-Brain จากการ failover แบบไม่มี consensus** — ตามที่พิสูจน์แล้ว: ถ้า primary เก่า
   "ฟื้นคืนชีพ" โดยไม่รู้ว่าถูก failover ไปแล้ว มันจะรับ write ต่อไปตามปกติ ทำให้ข้อมูล diverge
   จากคลัสเตอร์จริงอย่างถาวร ระบบ production ต้องมีกลไกป้องกัน เช่น **fencing token** (เลข
   generation ที่เพิ่มทุกครั้งที่ failover แล้วปฏิเสธ write จากโหนดที่ generation เก่ากว่า) หรือ
   ใช้ consensus algorithm เต็มรูปแบบ (Raft/Paxos) ซึ่งเกินขอบเขตของ capstone นี้
5. **Lost Write จาก Async Replication ที่ไม่รอ ack** — พิสูจน์แล้วว่า write ที่ primary ตอบ
   `OK` ไปแล้วอาจหายไปถาวรถ้า primary ตายก่อนที่ replica จะได้รับมัน (และไม่มี replica ตัวอื่น
   ได้รับด้วย) นี่คือ trade-off ของการเลือก Availability/Latency เหนือ Durability — ถ้าข้อมูล
   สำคัญมากพอ ต้องพิจารณา synchronous replication หรือ quorum write (ดูแบบฝึกหัดข้อ 3)
6. **ไม่มี Auto-Reconnect ทำให้ replica "หยุดตามทัน" อย่างเงียบๆ** — ในโปรเจกต์นี้ ถ้า
   connection ระหว่าง replica กับ primary หลุด (ไม่ว่าเพราะ primary crash หรือแค่เครือข่ายสะดุด
   ชั่วคราว) replica จะ**ไม่พยายามเชื่อมต่อใหม่เองเลย** ต้องสั่ง `REPLICAOF` ซ้ำด้วยมือเสมอ —
   พบเจอจริงระหว่างการทดสอบ Part นี้เอง (ดูหัวข้อ 123.6 ที่ B ไม่เคย resync กลับหลัง A restart)
   ระบบ production จริงอย่าง Redis จะ retry เชื่อมต่อใหม่อัตโนมัติพร้อม exponential backoff
   (ดูแบบฝึกหัดข้อ 6)
7. **Retry คำสั่งที่ไม่ Idempotent โดยไม่มี Idempotency Key** — พิสูจน์แล้วว่า `INCR` ที่ retry
   หลัง ack หายจะนับซ้ำ ทุกคำสั่งที่ไม่ idempotent (INCR, APPEND, "โอนเงิน", "ส่งอีเมล") ต้องมี
   กลไกป้องกันการ apply ซ้ำ ถ้า client library มี retry logic
8. **ล็อก mutex แยกกันระหว่าง "แก้ข้อมูล" กับ "เขียน WAL"** — ถ้าแยกล็อกสองจุดนี้ (ต่างจากที่
   `Storage` ทำในหัวข้อ 123.2) จะเกิดความเสี่ยงที่สอง thread apply เข้า memory กับเขียน WAL
   คนละลำดับกัน ทำให้ replay WAL แล้วได้ผลลัพธ์ไม่ตรงกับสถานะจริงที่เคยมีในหน่วยความจำ

### แบบฝึกหัดท้ายบท

1. เพิ่มคำสั่ง `DECR <key>` ที่ลดค่าตัวเลขทีละ 1 แบบ atomic เหมือน `INCR` (คำใบ้: เพิ่ม
   `doDecrLocked` ใน `Storage` คล้าย `doIncrLocked` แต่ `val -= 1`)
2. **(โจทย์ที่มีเฉลยเต็มรูปแบบด้านล่าง)** เพิ่ม TTL/expiry ให้ key ผ่านคำสั่ง `EXPIRE <key>
   <seconds>` ให้ key นั้นหายไปเองเมื่อครบเวลา ทั้งแบบ lazy (ตรวจตอน GET) และ active (background
   thread กวาดล้างเป็นระยะ)
3. เพิ่ม **Quorum Read**: ให้ client library อ่านค่าจากทั้ง primary และ replica ทุกตัวพร้อมกัน
   แล้วเลือกค่าที่มาจากโหนดที่รายงาน `seq` สูงที่สุด (ต้องแก้ response ของ `GET` ให้ส่ง seq
   กลับมาด้วย เช่น `VALUE <seq> <value>`) วิเคราะห์ว่าวิธีนี้แก้ปัญหา stale read ได้แค่ไหน
   และยังไม่แก้ปัญหาอะไรบ้าง (คำใบ้: ถ้าทุกโหนดที่ตอบยังไม่ทันได้ record ล่าสุดเลย quorum read
   ก็ยังอ่านค่าเก่าอยู่ดี)
4. เพิ่ม **Read-Your-Writes Consistency**: ให้ client library จำ `seq` ล่าสุดที่ตัวเองเคยเขียน
   สำเร็จไว้ แล้วเวลา `GET` จาก replica ให้ตรวจสอบก่อนว่า replica นั้น `last_applied >= seq`
   ที่จำไว้หรือไม่ ถ้ายังไม่ถึงให้รอหรือ fallback ไปอ่านจาก primary แทน
5. ออกแบบและ implement **Partial Resync**: แทนที่จะ FULLSYNC ใหม่ทุกครั้งที่ replica เชื่อมต่อ
   ให้ primary เก็บ "replication backlog" (WAL ช่วงล่าสุด เช่น 1000 record ล่าสุด) ไว้ในหน่วยความจำ
   ถ้า replica แจ้ง seq ล่าสุดที่ตัวเองมีมาตอน handshake และ seq นั้นยังอยู่ใน backlog ให้ส่งแค่
   ส่วนต่างแทนการ FULLSYNC ทั้งหมด
6. เพิ่ม **Auto-Reconnect** ให้ replica: เมื่อ `runReplicaLink` หลุดจาก loop (upstream ตาย)
   ให้ลองเชื่อมต่อใหม่เองอัตโนมัติทุกๆ N วินาที (แนะนำใช้ exponential backoff: เริ่มจาก 1 วินาที
   แล้วเพิ่มเป็น 2, 4, 8 วินาที จนถึงเพดานที่กำหนด) แทนที่จะต้องรอให้ operator สั่ง `REPLICAOF`
   ซ้ำด้วยมือทุกครั้ง

### แนวทางเฉลยข้อ 2: TTL/Expiry เต็มรูปแบบ (implement และทดสอบจริงแล้ว)

การ implement นี้อยู่ใน `storage.hpp` ที่แสดงในหัวข้อ 123.2 แล้ว (ฟังก์ชัน `doExpire`,
`reapExpired`, และการเช็ค `isExpiredNoLock` ใน `get`/`exists`/`snapshotForSync`) มี 2 กลไก
ทำงานร่วมกัน:

- **Lazy Expiration**: ทุกครั้งที่ `get()`/`exists()` ถูกเรียก จะเช็คก่อนว่า key นั้นหมดอายุ
  หรือยัง ถ้าหมดแล้วให้ตอบเหมือนไม่มี key นั้นอยู่เลย (แม้จะยังไม่ถูกลบออกจาก `map_` จริงๆ ก็ตาม)
- **Active Expiration**: background thread (`ttlReaperLoop` ใน `node.cpp`) เรียก
  `reapExpired()` ทุก 300ms เพื่อลบ key ที่หมดอายุออกจาก memory จริงๆ ป้องกันไม่ให้ key
  ที่หมดอายุแล้วแต่ไม่มีใครมา `get()` ค้างอยู่ในหน่วยความจำตลอดไป

ผูกเข้ากับ dispatcher ใน `node.cpp` ด้วยคำสั่ง `EXPIRE`:

```cpp
} else if (cmd == "EXPIRE") {
    if (!g_ns.isPrimary) {
        conn.sendLine("ERROR readonly replica; primary is at " + g_ns.primaryAddrForDisplay());
        continue;
    }
    std::istringstream is(rest);
    std::string key; int secs = 0;
    is >> key >> secs;
    if (key.empty() || !is) { conn.sendLine("ERROR usage: EXPIRE <key> <seconds>"); continue; }
    g_storage.doExpire(key, secs);
    conn.sendLine("OK");
}
```

ทดสอบจริง:

```bash
$ ./kvcli 127.0.0.1 7002 "SET session_token abc123"
$ ./kvcli 127.0.0.1 7002 "EXPIRE session_token 2"
$ ./kvcli 127.0.0.1 7002 "GET session_token"      # ทันที
$ sleep 2.5
$ ./kvcli 127.0.0.1 7002 "GET session_token"      # หลัง 2.5 วิ
$ ./kvcli 127.0.0.1 7002 "EXISTS session_token"
```

```
OK
OK
VALUE abc123
NOT_FOUND
0
```

ตรงตามที่ออกแบบไว้ทุกประการ: `session_token` ยังอ่านได้ปกติทันทีหลัง `EXPIRE`, แต่หายไปสนิท
(`NOT_FOUND`, `EXISTS` คืน `0`) หลังผ่านไป 2.5 วินาทีตามที่ตั้ง TTL ไว้ 2 วินาที

> **ข้อจำกัดที่ยอมรับไว้ตรงๆ**: TTL ในเฉลยนี้ **ไม่ถูก replicate ไปยัง replica เลย** (คำสั่ง
> `EXPIRE` ไม่ผ่าน `encodeEntry`/`broadcast`) ระบบจริงอย่าง Redis แก้ปัญหานี้ด้วยการแปลง
> `EXPIRE key seconds` (relative time) เป็น `PEXPIREAT key <absolute-unix-ms>` ก่อน replicate
> เพื่อไม่ให้ replica คำนวณเวลาหมดอายุผิดเพราะ clock ไม่ตรงกันหรือ network delay — เป็นรายละเอียด
> ที่ตั้งใจละไว้เพื่อไม่ให้โจทย์ซับซ้อนเกินไป

### แนวทางเฉลยข้อ (Idempotency Key สำหรับ At-Least-Once) — เฉลยเต็มรูปแบบ

นี่คือเฉลยเต็มรูปแบบข้อที่สองที่โจทย์กำหนด (ผูกกับปัญหา at-least-once ในหัวข้อ 123.7 โดยตรง)
Implementation อยู่ใน `storage.hpp` แล้ว (`reqCache_` และ `doIncrIdempotent`) หลักการคือ:

```cpp
ApplyResult doIncrIdempotent(const std::string& key, const std::string& reqId) {
    std::lock_guard<std::mutex> lk(mtx_);
    auto cached = reqCache_.find(reqId);
    if (cached != reqCache_.end()) {
        // เคยเห็น request-id นี้แล้ว: คืนผลลัพธ์เดิมโดยไม่ apply ซ้ำ
        return {true, seq_, "", cached->second};
    }
    ApplyResult r = doIncrLocked(key);
    if (r.ok) reqCache_[reqId] = r.value;
    return r;
}
```

จุดสำคัญของเฉลยนี้:

1. **`reqCache_` เก็บ mapping จาก request-id → ผลลัพธ์** ที่เคย apply ไปแล้ว ทำให้ครั้งที่สอง
   ที่ request-id เดิมมาถึง (จาก retry) ระบบคืนผลลัพธ์เดิมโดยไม่แตะ `map_` เลย
2. **เช็ค cache และ apply จริงอยู่ใน critical section เดียวกัน** (ล็อก `mtx_` ครั้งเดียว
   ตลอดฟังก์ชัน) — ถ้าแยกเป็นสองขั้นตอน (เช็ค cache ก่อน ปลดล็อก แล้วค่อย apply) จะเกิด Race
   Condition แบบเดียวกับที่ Part 40 เตือนเรื่อง GET-then-SET: ถ้า retry สองอันมาพร้อมกันพอดี
   (เช่น client timeout แล้วยิง retry ซ้อนกับ connection เดิมที่ยัง in-flight อยู่) ทั้งคู่อาจ
   เห็นว่า cache ยังไม่มี แล้ว apply ซ้ำทั้งคู่ได้
3. ผลการทดสอบจริงอยู่ในหัวข้อ 123.7 แล้ว: `safe_counter` คงค่า `1` แม้จะถูก retry ก็ตาม
   ต่างจาก `unsafe_counter` ที่กลายเป็น `2`

> **ข้อจำกัดที่ยอมรับไว้ตรงๆ**: `reqCache_` ในเฉลยนี้เก็บไว้ตลอดไปไม่มีวันลบ (ไม่มี TTL/LRU
> eviction) ในระบบ production จริงต้องจำกัดอายุของ cache นี้ (เช่น เก็บแค่ 24 ชั่วโมง) มิฉะนั้น
> หน่วยความจำจะโตไม่มีที่สิ้นสุดเมื่อใช้งานนานๆ — การผสม TTL จากแบบฝึกหัดข้อ 2 เข้ากับ
> `reqCache_` นี้เป็นส่วนขยายที่ทำได้ไม่ยากและควรลองทำเป็นแบบฝึกหัดเพิ่มเติมด้วยตัวเอง

---

## สรุปท้ายบท

ใน Part นี้เราได้:

- เขียน **Distributed Key-Value Store** เต็มรูปแบบด้วย Modern C++ ต่อยอดจาก Mini KV Store (C)
  ใน Part 40 ให้กลายเป็นระบบหลายโหนดที่มี Replication และ Write-Ahead Log จริง
- ออกแบบ Storage Engine ที่ **เขียน WAL ก่อนตอบกลับเสมอ (Write-Ahead Logging)** และ Snapshot
  ที่บีบอัด WAL ด้วยเทคนิค atomic `rename()` ต่อยอดจาก Part 13
- รันหลาย process บน `127.0.0.1` คนละ port จริง เพื่อจำลอง cluster บนเครื่องเดียว, พิสูจน์ด้วย
  ข้อมูลจริงว่า **replication ทำงานได้จริง** ข้าม process ผ่านโปรโตคอล FULLSYNC + streaming
- ทดสอบ **Crash & Recovery จริง**: `kill -9` process กลางคันแล้วยืนยันว่า WAL replay กู้คืน
  สถานะได้ครบถ้วนทุกครั้ง รวมถึงพิสูจน์ด้วยการทดลองจริงว่า `fsync()` ป้องกันอะไรบ้าง (และไม่
  ป้องกันอะไรบ้าง) ต่างจากความเข้าใจผิดที่พบบ่อย
- ทำ **Manual Failover** (`PROMOTE`/`REPLICAOF`) สำเร็จจริง และได้เห็น **Split-Brain** เกิดขึ้น
  จริงพร้อมข้อมูลที่ diverge กันจริงเมื่อ primary เก่าฟื้นคืนชีพโดยไม่รู้ตัว — เชื่อมโยงเข้ากับ
  CAP Theorem และตารางเปรียบเทียบ Consistency Model/Replication Strategy ต่างๆ
- สร้าง **Client Library** เบื้องต้นและพิสูจน์แนวคิด **At-Least-Once Delivery** พร้อมทางแก้ด้วย
  **Idempotency Key** ด้วยการทดลองที่จับต้องได้จริง ไม่ใช่แค่ทฤษฎี
- พูดตรงๆ และซื่อสัตย์ตลอดทั้งบทว่าโปรเจกต์นี้**ไม่ใช่** Raft/Paxos และไม่มี Consensus Algorithm
  ป้องกัน Split-Brain จริงจัง — เข้าใจว่าระบบ production จริงต้องการอะไรเพิ่มเติมจากนี้อีกมาก

Capstone นี้พิสูจน์ให้เห็นว่าแนวคิดหลักของ Distributed Systems (Replication, Consistency
Trade-off, Failure Detection, At-Least-Once Semantics) สามารถเข้าใจได้อย่างลึกซึ้งด้วยการ
**ลงมือเขียนโค้ดจริง รันจริง และทำให้มันพังในแบบที่ควบคุมได้จริง** — ไม่ใช่แค่ท่องทฤษฎีเฉยๆ

ใน **Part 124** ซึ่งเป็น Capstone สุดท้ายก่อนบทสรุปหลักสูตร เราจะเปลี่ยนทิศทางกลับมาที่การสร้าง
**Web Framework ของตัวเองด้วย C++ ตั้งแต่ศูนย์** — นำความรู้เรื่อง Template, Routing, HTTP
Parsing จาก Module I มาประกอบร่างเป็นเฟรมเวิร์กระดับ mini-Express/mini-Flask ที่เข้าใจการทำงาน
ภายในได้ทุกบรรทัด

**ต่อไป:** [Part 124 — Capstone 4: สร้าง Web Framework ของตัวเอง](./part-124-capstone-web-framework.md)
