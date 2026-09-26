# Part 101: Multi-threaded HTTP Server พร้อม Thread Pool (Step 801–808)

> Module I — Web Development ด้วย C/C++ | Part 101 จาก 125
> Step ที่ครอบคลุมใน Part นี้: Step 801–808
> Part ก่อนหน้า: [Part 100 — สร้าง Raw HTTP Server จาก Socket ด้วยมือ](./part-100-raw-http-server.md) | Part ถัดไป: [Part 102 — Crow Framework: พื้นฐานและ REST API แรก](./part-102-crow-basics.md)

## เป้าหมายการเรียนรู้ (Learning Objectives)

หลังจบ Part นี้ ผู้เรียนจะสามารถ:

1. อธิบายได้อย่างชัดเจนว่าทำไม Server จาก Part 100 (accept() loop เดี่ยว) รับ Client ได้
   ทีละ 1 รายเท่านั้น และผลกระทบที่เกิดขึ้นจริงเมื่อมี Client จำนวนมากพร้อมกัน
2. แก้ปัญหานั้นด้วยแนวคิดแรก **Thread-per-Connection** โดยใช้ `std::thread` (Part 81)
   สร้าง thread ใหม่ทุกครั้งที่ `accept()` connection สำเร็จ และพิสูจน์ด้วยการทดสอบจริงว่า
   ปัญหา Head-of-Line Blocking จาก Part 100 หายไปจริง
3. ค้นพบและ**วัดผลกระทบจริง**ของปัญหาใหม่ที่ตามมาจาก Thread-per-Connection: **Thread
   Creation Overhead** เมื่อจำนวน Connection สูงขึ้นมาก ด้วย Benchmark ที่เขียนขึ้นเอง
4. ออกแบบและอธิบาย **Thread Pool Pattern** ได้ด้วยตัวเอง: สร้าง Worker Thread จำนวนคงที่
   ล่วงหน้า แล้วแจกจ่ายงานผ่าน Task Queue ที่ป้องกันด้วย `std::mutex` และ
   `std::condition_variable` (Part 82)
5. Implement คลาส `ThreadPool` ที่สมบูรณ์ครบทุกส่วน: `enqueue()`, Worker Loop ที่ป้องกัน
   Spurious Wakeup อย่างถูกต้อง, และ `shutdown()` ที่ join worker thread ทุกตัวอย่างปลอดภัย
6. ประกอบ `ThreadPool` เข้ากับ HTTP Server จาก Part 100 ให้เป็น Server ที่รองรับ Concurrent
   Connection ได้จริง พร้อม Graceful Shutdown ผ่าน `SIGINT`
7. เปรียบเทียบ Performance ระหว่าง Thread-per-Connection กับ Thread Pool ด้วย**ตัวเลขจริง
   ที่วัดได้เอง**ในหลายระดับ Concurrency และอธิบายได้ว่าทำไมผลลัพธ์ถึงออกมาเป็นแบบนั้น
8. ระบุและป้องกัน Race Condition บน Task Queue, Deadlock ที่อาจเกิดจากการออกแบบ Thread
   Pool ที่ไม่รอบคอบ และปัญหาการลืม join worker thread ตอนปิดโปรแกรม

---

## 101.1 ทบทวนปัญหาจาก Part 100: รับได้ทีละ 1 Connection (Step 801)

Part 100 ปิดท้ายด้วยการสาธิตให้เห็นจริงว่า Server ที่มี `accept()` loop เดียวมีข้อจำกัด
ร้ายแรง: **Client ที่ "ช้า" เพียงรายเดียวสามารถทำให้ Client รายอื่นทั้งหมดต้องรอได้** เพราะ
loop หลักของ Server ทำงานแบบ **Synchronous ทีละขั้นตอน** — `accept()` แล้ว `recv()` แล้ว
`send()` แล้วค่อยวนกลับไป `accept()` รายถัดไป ไม่มีทางทำสองอย่างพร้อมกันได้เลยตราบใดที่
โค้ดยังรันอยู่ใน thread เดียว

```
Server (accept() loop เดียว, Single-threaded) จาก Part 100:

┌────────────────────────────────────────────────────────────────┐
│  main thread:                                                   │
│  accept() → recv() → handleRequest() → send() → close() → accept() (วนซ้ำ) │
│              ▲                                                   │
│              └── ถ้า client ช้าตรงนี้ ทุกอย่างที่เหลือต้องรอ            │
└────────────────────────────────────────────────────────────────┘
```

ทางแก้ที่ Part 100 หัวข้อ 100.8 ทิ้งท้ายไว้มี 2 แนวทางหลัก: **I/O Multiplexing**
(`select`/`poll`/`epoll` จาก Part 34) และ **Multithreading** — Part นี้เลือกแนวทางที่สอง
เพราะต่อยอดโดยตรงจาก `std::thread` (Part 81) และ `std::mutex`/`std::condition_variable`
(Part 82) ที่เรียนมาแล้ว และเป็นรากฐานสำคัญก่อนจะไปเห็นว่า Framework จริงอย่าง Crow
(Part 102 เป็นต้นไป) ผสม Multithreading เข้ากับ I/O Multiplexing ไว้ข้างในอย่างไร

### แนวคิดที่ง่ายที่สุดที่นึกถึงเป็นอันดับแรก: "สร้าง Thread ใหม่ทุก Connection"

ถ้า Client แต่ละรายทำงานอยู่ใน thread ของตัวเอง Client ที่ช้าก็จะบล็อกแค่ thread ของ
ตัวเองเท่านั้น ไม่กระทบ thread ของ Client รายอื่นเลย — นี่คือแนวคิดของ **Thread-per-
Connection** ซึ่งเป็นก้าวแรกตามธรรมชาติที่สุดที่คนเขียน Server มักจะนึกถึงก่อนเสมอ ก่อนจะ
ค้นพบข้อจำกัดของมันเองในภายหลัง (หัวข้อ 101.3) เหมือนกับที่เรากำลังจะเดินตามลำดับ
เดียวกันนี้ในบทเรียน

```
Thread-per-Connection: แนวคิดที่ 1

main thread:  accept() ──┬─▶ spawn thread ──▶ handle client 1 (ทำงานอิสระ)
              accept() ──┼─▶ spawn thread ──▶ handle client 2 (ทำงานอิสระ)
              accept() ──┴─▶ spawn thread ──▶ handle client 3 (ทำงานอิสระ)
                 │
                 └── กลับไป accept() รอบใหม่ทันที ไม่ต้องรอ client ก่อนหน้าเลย
```

---

## 101.2 สร้าง Thread-per-Connection Server เต็มรูปแบบ (Step 802)

นำ `parseRequestLine()`, `buildResponse()`, และ `handleRequest()` จาก Part 100 มาใช้ซ้ำ
ทั้งหมด (ไม่มีอะไรเปลี่ยนในส่วน HTTP parsing/response เลย) สิ่งที่เปลี่ยนมีจุดเดียวคือ
**สิ่งที่ทำหลัง `accept()` สำเร็จ**:

```cpp
// thread_per_connection_server.cpp - HTTP server: spawn thread ใหม่ทุก connection
#include <arpa/inet.h>
#include <atomic>
#include <cstdio>
#include <cstdlib>
#include <cstring>
#include <ctime>
#include <mutex>
#include <netinet/in.h>
#include <string>
#include <sys/socket.h>
#include <thread>
#include <unistd.h>
#include <vector>

#define PORT 8101
#define BACKLOG 128
#define BUFFER_SIZE 4096

struct HttpRequest {
    std::string method;
    std::string path;
    std::string version;
    bool valid = false;
};

HttpRequest parseRequestLine(const std::string& raw) {
    HttpRequest req;
    std::size_t lineEnd = raw.find("\r\n");
    if (lineEnd == std::string::npos) {
        return req;
    }
    std::string requestLine = raw.substr(0, lineEnd);
    std::size_t firstSpace = requestLine.find(' ');
    std::size_t lastSpace = requestLine.rfind(' ');
    if (firstSpace == std::string::npos || lastSpace == std::string::npos ||
        firstSpace == lastSpace) {
        return req;
    }
    req.method = requestLine.substr(0, firstSpace);
    req.path = requestLine.substr(firstSpace + 1, lastSpace - firstSpace - 1);
    req.version = requestLine.substr(lastSpace + 1);
    req.valid = true;
    return req;
}

std::string buildResponse(int statusCode, const std::string& statusText,
                           const std::string& contentType, const std::string& body) {
    std::string response;
    response += "HTTP/1.1 " + std::to_string(statusCode) + " " + statusText + "\r\n";
    response += "Content-Type: " + contentType + "\r\n";
    response += "Content-Length: " + std::to_string(body.size()) + "\r\n";
    response += "Connection: close\r\n\r\n";
    response += body;
    return response;
}

std::string handleRequest(const HttpRequest& req) {
    if (!req.valid) {
        return buildResponse(400, "Bad Request", "text/html", "<h1>400 Bad Request</h1>");
    }
    if (req.method != "GET") {
        return buildResponse(405, "Method Not Allowed", "text/html", "<h1>405</h1>");
    }
    if (req.path == "/") {
        return buildResponse(200, "OK", "text/html",
                              "<html><body><h1>Thread-per-connection server</h1></body></html>");
    }
    if (req.path == "/slow") {
        // จำลอง client/handler ที่ทำงานช้า (เช่น query database หนักๆ)
        std::this_thread::sleep_for(std::chrono::milliseconds(3000));
        return buildResponse(200, "OK", "text/plain", "slow response done");
    }
    return buildResponse(404, "Not Found", "text/html", "<h1>404</h1>");
}

// นับจำนวน thread ที่กำลังทำงานอยู่ ณ ขณะนั้น (สำหรับสาธิต/log)
std::atomic<long> g_activeThreads{0};
std::atomic<long> g_totalHandled{0};

void handleClient(int clientFd) {
    ++g_activeThreads;
    char buffer[BUFFER_SIZE];
    ssize_t bytesRead = recv(clientFd, buffer, sizeof(buffer) - 1, 0);
    if (bytesRead > 0) {
        buffer[bytesRead] = '\0';
        HttpRequest req = parseRequestLine(std::string(buffer, static_cast<std::size_t>(bytesRead)));
        std::string response = handleRequest(req);

        std::size_t totalSent = 0;
        while (totalSent < response.size()) {
            ssize_t sent = send(clientFd, response.data() + totalSent, response.size() - totalSent, 0);
            if (sent < 0) break;
            totalSent += static_cast<std::size_t>(sent);
        }
    }
    close(clientFd);
    ++g_totalHandled;
    --g_activeThreads;
}

int main(int argc, char** argv) {
    int port = PORT;
    if (argc > 1) port = std::atoi(argv[1]);

    int serverFd = socket(AF_INET, SOCK_STREAM, 0);
    if (serverFd < 0) { perror("socket"); return EXIT_FAILURE; }

    int opt = 1;
    setsockopt(serverFd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));

    struct sockaddr_in serverAddr;
    std::memset(&serverAddr, 0, sizeof(serverAddr));
    serverAddr.sin_family = AF_INET;
    serverAddr.sin_addr.s_addr = INADDR_ANY;
    serverAddr.sin_port = htons(static_cast<uint16_t>(port));

    if (bind(serverFd, reinterpret_cast<struct sockaddr*>(&serverAddr), sizeof(serverAddr)) < 0) {
        perror("bind"); close(serverFd); return EXIT_FAILURE;
    }
    if (listen(serverFd, BACKLOG) < 0) {
        perror("listen"); close(serverFd); return EXIT_FAILURE;
    }

    std::printf("[thread-per-connection] listening on port %d\n", port);

    for (;;) {
        struct sockaddr_in clientAddr;
        socklen_t clientLen = sizeof(clientAddr);
        int clientFd = accept(serverFd, reinterpret_cast<struct sockaddr*>(&clientAddr), &clientLen);
        if (clientFd < 0) {
            perror("accept");
            continue;
        }
        // spawn thread ใหม่ทุกครั้งที่ accept ได้ 1 connection แล้ว detach ทันที
        std::thread(handleClient, clientFd).detach();
    }

    close(serverFd);
    return 0;
}
```

จุดสำคัญที่สุดของโค้ดนี้คือบรรทัดเดียว:

```cpp
std::thread(handleClient, clientFd).detach();
```

หลัง `accept()` สำเร็จ เราสร้าง `std::thread` ใหม่ทันที ส่ง `clientFd` เข้าไปเป็น argument
(copy by value ตามพฤติกรรม default ของ `std::thread` ที่เรียนใน Part 81.3 — ปลอดภัยเพราะ
`clientFd` เป็นแค่ `int` ธรรมดา) แล้ว `.detach()` ทันทีเพื่อปล่อยให้ thread ทำงานเป็นอิสระ
โดยที่ loop หลักไม่ต้องรอ `.join()` เลย จึงกลับไป `accept()` รายถัดไปได้ทันที

> **ทำไมใช้ `.detach()` ไม่ใช่ `.join()` หรือเก็บใน `std::vector<std::thread>`?**
> เพราะ main loop ต้องกลับไป `accept()` ทันทีโดยไม่รอ thread ที่เพิ่งสร้างทำงานจบ ถ้าใช้
> `.join()` ตรงนี้จะกลายเป็นการรอทีละ Client เหมือนเดิมทุกประการ (แก้ปัญหาไม่ได้เลย) และ
> เก็บใน `vector` ก็ไม่มีจุดไหนใน loop ที่จะปลอดภัยพอจะ `.join()` cleanup ทีหลังได้ (Client
> อาจเชื่อมต่อเข้ามาไม่มีที่สิ้นสุด) `.detach()` จึงเป็นทางเลือกที่ตรงไปตรงมาที่สุดสำหรับ
> รูปแบบนี้ — แต่จะเห็นในหัวข้อ Common Pitfalls ว่ามันแลกมาด้วยอะไรตอน shutdown

คอมไพล์ด้วย Flag มาตรฐานของหลักสูตร:

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 -pthread -O2 thread_per_connection_server.cpp -o thread_per_connection_server
```

คอมไพล์ผ่านโดย**ไม่มี Warning ใดๆ เลย**

### ทดสอบว่าฟังก์ชันพื้นฐานยังทำงานถูกต้องเหมือน Part 100

```bash
./thread_per_connection_server 8101 &
curl -s -i http://127.0.0.1:8101/
```

ผลลัพธ์จริงจากการทดสอบ:

```
HTTP/1.1 200 OK
Content-Type: text/html
Content-Length: 63
Connection: close

<html><body><h1>Thread-per-connection server</h1></body></html>
```

### พิสูจน์ว่าปัญหา Head-of-Line Blocking จาก Part 100 หายไปจริง

ทำการทดสอบเดิมทุกประการกับที่ Part 100 หัวข้อ 100.8 ใช้สาธิตปัญหา: เปิด connection ไปที่
`/slow` (ค้างอยู่ 3 วินาทีฝั่ง Server) แล้วยิง Client ปกติตามไปทันที:

```bash
./thread_per_connection_server 8101 &
sleep 0.3

# client 1: เปิด connection ไปที่ /slow (server จะค้างอยู่ 3 วินาที)
curl -s -o /dev/null http://127.0.0.1:8101/slow &
sleep 0.3

# client 2: ยิง request ปกติทันที ควรได้รับ response เร็ว แม้ client 1 กำลังถูก handle อยู่
time curl -s -o /dev/null -w "HTTP %{http_code}, ใช้เวลา %{time_total}s\n" http://127.0.0.1:8101/
```

ผลลัพธ์จริงที่วัดได้:

```
HTTP 200, ใช้เวลา 0.000676s

real	0m0.009s
user	0m0.005s
sys	0m0.004s
```

เทียบกับ Part 100 ที่ Client รายที่สองต้องรอถึง **2.69 วินาที** เพราะติดอยู่หลัง Client
รายแรกใน `accept()` loop เดียว ตอนนี้ Client รายที่สองได้รับ Response ภายใน **0.68
มิลลิวินาที** แม้ Client รายแรกจะยังค้างอยู่ที่ `/slow` อีกเกือบ 3 วินาทีก็ตาม — เหตุผลคือ
ทันทีที่ `accept()` คืนค่า `clientFd` ของ Client 1 กลับมา main loop ก็สร้าง thread แยกไป
จัดการ Client 1 แล้ว**กลับไป `accept()` รอบใหม่ทันที** ไม่ต้องรอ Client 1 จบก่อนเหมือนใน
Part 100 อีกต่อไป Client 2 จึงถูก `accept()` และจัดการโดย thread ของตัวเองได้ทันทีโดยไม่มี
อะไรมาบล็อกเลย

นี่คือหลักฐานที่ชัดเจนว่า Thread-per-Connection แก้ปัญหา Head-of-Line Blocking จาก Part
100 ได้จริง 100% — แต่คำถามที่ต้องถามต่อคือ: **แนวทางนี้จะยังทำงานดีอยู่ไหมถ้ามี Client
เชื่อมต่อเข้ามาพร้อมกันเป็นจำนวนมาก (หลักร้อยหรือหลักพัน) ไม่ใช่แค่ 2 รายแบบในตัวอย่าง?**
คำตอบคือ **ไม่** และหัวข้อถัดไปจะพิสูจน์ให้เห็นด้วยตัวเลขจริง

---

## 101.3 ปัญหาใหม่ที่ตามมา: Thread Creation Overhead (Step 803)

### ทำไมการสร้าง Thread ถึงมี "ต้นทุน"

ทุกครั้งที่เรียก `std::thread(...)` ภายใน (บน Linux) จะไปเรียก `pthread_create()` ซึ่งต้อง
ขอให้ **Kernel** จัดสรรทรัพยากรใหม่ทั้งหมด: Stack memory (ปกติหลาย MB ต่อ thread),
Thread Control Block, Scheduling metadata, และลงทะเบียน thread ใหม่เข้าไปใน Scheduler ของ
OS การสลับเข้า-ออกจาก Kernel Mode (System Call) เหล่านี้มี**ต้นทุนที่วัดได้จริง** แม้จะ
เล็กมากในแต่ละครั้ง (หลักไมโครวินาที) แต่เมื่อต้องทำซ้ำเป็นพันหรือหมื่นครั้งต่อวินาที
ต้นทุนนี้จะสะสมกลายเป็นปัญหาจริงจัง

### วัดต้นทุนล้วนๆ ของการสร้าง Thread แยกออกจาก Network

ก่อนจะไปวัดผลกับ HTTP Server เต็มรูปแบบ (ที่มีต้นทุนของ Network ปนอยู่ด้วย) ลองแยกตัวแปร
ให้ชัดเจนก่อนด้วยการวัด**เฉพาะ**ต้นทุนของการสร้าง+join thread ล้วนๆ เทียบกับการใช้ pool
คงที่ทำงานจำนวนเท่ากัน (งานที่ทำในแต่ละ task คือแค่ `++counter` ตัวเดียว แทบไม่มีต้นทุนอะไร
เลยนอกจากต้นทุนของตัว thread เอง):

```cpp
// thread_creation_overhead.cpp - วัด "ต้นทุนล้วนๆ" ของการสร้าง thread ใหม่ทุกครั้ง
// เทียบกับการใช้ thread pool คงที่ทำงานจำนวนเท่ากัน (ไม่มี network เข้ามาปน เพื่อแยกตัวแปร)
#include <atomic>
#include <chrono>
#include <condition_variable>
#include <cstdio>
#include <mutex>
#include <queue>
#include <thread>
#include <vector>

constexpr int NUM_TASKS = 100000;

std::atomic<long> g_counter{0};

void trivialTask() { ++g_counter; }

double benchThreadPerTask() {
    g_counter = 0;
    auto start = std::chrono::steady_clock::now();
    for (int i = 0; i < NUM_TASKS; ++i) {
        std::thread(trivialTask).join(); // สร้าง thread ใหม่ + join ทันที ทุก task
    }
    auto end = std::chrono::steady_clock::now();
    return std::chrono::duration<double>(end - start).count();
}

class MiniPool {
public:
    explicit MiniPool(std::size_t n) : stopping_(false) {
        for (std::size_t i = 0; i < n; ++i) {
            workers_.emplace_back([this] { loop(); });
        }
    }
    ~MiniPool() {
        { std::lock_guard<std::mutex> lk(m_); stopping_ = true; }
        cv_.notify_all();
        for (auto& w : workers_) w.join();
    }
    void submit() {
        { std::lock_guard<std::mutex> lk(m_); queue_.push(1); }
        cv_.notify_one();
    }
    void waitUntilDrained() {
        for (;;) {
            {
                std::lock_guard<std::mutex> lk(m_);
                if (queue_.empty()) return;
            }
            std::this_thread::yield(); // ไม่ busy-wait หนักเกินไป พอสำหรับ micro-benchmark นี้
        }
    }

private:
    void loop() {
        for (;;) {
            std::unique_lock<std::mutex> lk(m_);
            cv_.wait(lk, [this] { return stopping_ || !queue_.empty(); });
            if (queue_.empty()) {
                if (stopping_) return;
                continue;
            }
            queue_.pop();
            lk.unlock();
            trivialTask();
        }
    }
    std::vector<std::thread> workers_;
    std::queue<int> queue_;
    std::mutex m_;
    std::condition_variable cv_;
    bool stopping_;
};

double benchThreadPool(std::size_t poolSize) {
    g_counter = 0;
    auto start = std::chrono::steady_clock::now();
    {
        MiniPool pool(poolSize);
        for (int i = 0; i < NUM_TASKS; ++i) {
            pool.submit();
        }
        pool.waitUntilDrained();
        // destructor ของ pool จะ join worker ทุกตัวตอนออกจาก scope
    }
    auto end = std::chrono::steady_clock::now();
    return std::chrono::duration<double>(end - start).count();
}

int main() {
    unsigned int hw = std::thread::hardware_concurrency();
    if (hw == 0) hw = 4;

    double t1 = benchThreadPerTask();
    std::printf("[thread-per-task]  %d tasks, spawn+join ทุกครั้ง: %.3f วินาที (%.1f task/s), counter=%ld\n",
                NUM_TASKS, t1, NUM_TASKS / t1, g_counter.load());

    double t2 = benchThreadPool(hw);
    std::printf("[thread-pool %2u]  %d tasks, ใช้ queue+%u worker คงที่: %.3f วินาที (%.1f task/s), counter=%ld\n",
                hw, NUM_TASKS, hw, t2, NUM_TASKS / t2, g_counter.load());

    std::printf("[สรุป] thread pool เร็วกว่า %.1f เท่า สำหรับงานเล็กๆ จำนวนมากแบบนี้\n", t1 / t2);
    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 -pthread -O2 thread_creation_overhead.cpp -o thread_creation_overhead
./thread_creation_overhead
```

ผลลัพธ์จริงจากการทดสอบ (เครื่องทดสอบมี 4 logical core):

```
[thread-per-task]  100000 tasks, spawn+join ทุกครั้ง: 3.964 วินาที (25225.5 task/s), counter=100000
[thread-pool  4]  100000 tasks, ใช้ queue+4 worker คงที่: 1.018 วินาที (98276.2 task/s), counter=100000
[สรุป] thread pool เร็วกว่า 3.9 เท่า สำหรับงานเล็กๆ จำนวนมากแบบนี้
```

รันซ้ำอีกสองครั้งเพื่อยืนยันว่าไม่ใช่ความบังเอิญ:

```
[thread-per-task]  100000 tasks, spawn+join ทุกครั้ง: 4.160 วินาที (24040.3 task/s), counter=100000
[thread-pool  4]  100000 tasks, ใช้ queue+4 worker คงที่: 0.952 วินาที (105031.5 task/s), counter=100000
[สรุป] thread pool เร็วกว่า 4.4 เท่า สำหรับงานเล็กๆ จำนวนมากแบบนี้
```

**ผลลัพธ์คงที่ทุกครั้งที่รัน: thread pool เร็วกว่าประมาณ 4 เท่า** สำหรับงานเล็กๆ จำนวนมาก
แบบนี้ — และนี่คือการวัดที่**ไม่มี network เข้ามาปนเลยแม้แต่นิดเดียว** (ทั้งสองแบบทำงาน
เดียวกันทุกประการคือ `++counter`) ตัวเลขที่ต่างกันมหาศาลนี้มาจาก**ต้นทุนของการสร้าง thread
ใหม่ 100,000 ครั้ง**ล้วนๆ เทียบกับการใช้ thread ที่มีอยู่แล้ว 4 ตัวหยิบงานจาก queue ซ้ำๆ

### วัดผลกระทบกับ HTTP Server จริงภายใต้โหลดสูง

ทีนี้มาดูว่าผลกระทบนี้ปรากฏชัดแค่ไหนเมื่อรวมกับ Network I/O จริง เขียนโปรแกรม Client
ทดสอบโหลดของตัวเอง (จะอธิบายเต็มรูปแบบในหัวข้อ 101.7) แล้วยิงไปที่
`thread_per_connection_server` ด้วยระดับ Concurrency ที่ต่างกัน:

```bash
./thread_per_connection_server 8101 &

# concurrency=100 (100 thread ยิงพร้อมกัน x 20 request ต่อ thread = 2000 request รวม)
./bench_client 8101 100 20 /

# concurrency=300 (300 thread ยิงพร้อมกัน x 10 request ต่อ thread = 3000 request รวม)
./bench_client 8101 300 10 /
```

ผลลัพธ์จริงจากการทดสอบ:

```
concurrency=100 requests_per_thread=20 total=2000 success=2000 failed=0 time=0.184s throughput=10896.1 req/s
concurrency=300 requests_per_thread=10 total=3000 success=3000 failed=0 time=2.109s throughput=1422.8 req/s
```

สังเกตตัวเลขให้ดี: เมื่อเพิ่ม concurrency จาก 100 เป็น 300 (เพิ่มขึ้น 3 เท่า) throughput
**ลดลง**จาก 10,896 req/s เหลือเพียง 1,423 req/s (**ลดลงเกือบ 8 เท่า**) ทั้งที่ทั้งสอง
การทดสอบยิง request ทั้งหมดจำนวนใกล้เคียงกัน (2,000 กับ 3,000) — นี่ไม่ใช่พฤติกรรมที่ควร
เกิดขึ้นถ้า Server ขยายขนาด (scale) ได้ดี เหตุผลคือที่ concurrency สูงขึ้น Server ต้อง
**สร้าง OS thread ใหม่พร้อมกันเป็นจำนวนมากในเวลาสั้นๆ** ทำให้เกิดการแย่งชิง CPU และ
Scheduler overhead ระหว่าง thread ที่เพิ่งสร้างใหม่จำนวนมากพร้อมกัน ซ้อนทับกับต้นทุนของ
การสร้าง thread เองที่วัดแยกไว้แล้วข้างต้น

### ตารางสรุปตัวเลขที่วัดได้ในหัวข้อนี้

| การทดสอบ | ผลลัพธ์ |
|---|---|
| Micro-benchmark: 100,000 spawn+join เปล่าๆ | 3.96s (25,225 task/s) |
| Micro-benchmark: 100,000 งานผ่าน thread pool (4 worker) | 1.02s (98,276 task/s) — **เร็วกว่า ~3.9 เท่า** |
| HTTP server จริง, concurrency=100 (2,000 request) | 10,896 req/s |
| HTTP server จริง, concurrency=300 (3,000 request) | 1,423 req/s (**ลดลง ~8 เท่า** เมื่อ concurrency เพิ่ม 3 เท่า) |

บทสรุปของหัวข้อนี้: Thread-per-Connection **แก้ปัญหา Head-of-Line Blocking ได้จริง** แต่
**ไม่ scale ได้ดีเมื่อจำนวน Connection พร้อมกันสูงขึ้นมาก** เพราะต้นทุนของการสร้าง thread
ใหม่ทุกครั้งไม่หายไปไหน ยิ่ง Connection เข้ามาถี่เท่าไหร่ ต้นทุนนี้ก็ยิ่งเด่นชัดขึ้นเท่านั้น
— นี่คือจุดที่ **Thread Pool Pattern** เข้ามาแก้ปัญหาทั้งสองอย่างพร้อมกัน

---

## 101.4 แนวคิด Thread Pool Pattern (Step 804)

### หลักการ: สร้าง Thread "ล่วงหน้า" ครั้งเดียว แล้วนำกลับมาใช้ซ้ำ

Thread Pool แก้ปัญหา Thread Creation Overhead ด้วยไอเดียที่ตรงไปตรงมา: **สร้าง Worker
Thread จำนวนคงที่ (เช่น เท่ากับจำนวน CPU core) เพียงครั้งเดียวตอนโปรแกรมเริ่มทำงาน** แล้ว
ให้ Worker Thread เหล่านั้น**วนรับงานจาก Task Queue ไปเรื่อยๆ** ไม่มีการสร้าง thread ใหม่
อีกเลยตลอดอายุของโปรแกรม (ยกเว้นตอน resize pool เอง ซึ่งไม่ได้ทำใน Part นี้)

```
Thread Pool Pattern:

┌──────────────────────────────────────────────────────────────────┐
│                                                                    │
│   main thread:                     Task Queue                    │
│   accept() ──▶ enqueue(fd) ──▶  [fd1][fd2][fd3][fd4]...           │
│   accept() ──▶ enqueue(fd) ──▶       ▲                            │
│   accept() ──▶ enqueue(fd) ──▶       │ ป้องกันด้วย mutex+condition_variable │
│      │                               │                            │
│      └── กลับไป accept() ทันที        │                            │
│                                       │                            │
│                    ┌──────────────────┼──────────────────┐        │
│                    ▼                  ▼                  ▼        │
│              Worker Thread 1    Worker Thread 2    Worker Thread 3│
│              (สร้างไว้ล่วงหน้า)   (สร้างไว้ล่วงหน้า)   (สร้างไว้ล่วงหน้า)│
│              รอ→หยิบงาน→ทำ→รอ    รอ→หยิบงาน→ทำ→รอ     รอ→หยิบงาน→ทำ→รอ│
│              (วนซ้ำตลอดอายุโปรแกรม ไม่สร้าง thread ใหม่อีกเลย)         │
└──────────────────────────────────────────────────────────────────┘
```

สังเกตความแตกต่างที่สำคัญที่สุดจาก Thread-per-Connection: **จำนวน Thread คงที่เสมอ ไม่ว่า
จะมี Connection เข้ามากี่รายก็ตาม** — ถ้า Connection เข้ามาเร็วกว่าที่ Worker จะประมวลผล
ทัน งานเหล่านั้นจะไป**รอในคิว**แทนที่จะไปสร้าง thread ใหม่เพิ่ม (ต่างจาก Thread-per-
Connection ที่จำนวน thread โตไม่มีขอบเขตตามจำนวน Connection)

### ทำไมต้องใช้ `std::mutex` + `std::condition_variable` คู่กันเสมอ

Task Queue เป็น **Shared State** ที่ทั้ง Main Thread (ผู้เขียน/Producer) และ Worker
Thread ทุกตัว (ผู้อ่าน/Consumer) เข้าถึงพร้อมกัน — ตรงกับรูปแบบคลาสสิกที่เรียกว่า
**Producer-Consumer Pattern** ที่ต้องการเครื่องมือ 2 อย่างร่วมกันเสมอ:

- **`std::mutex`** (Part 82.1): ป้องกัน Race Condition ตอนอ่าน/เขียน `std::queue`
  พร้อมกันจากหลาย thread — `std::queue` **ไม่ thread-safe เองเลยแม้แต่นิดเดียว**
  (เหมือน `std::vector`, `std::string` และ container มาตรฐานเกือบทั้งหมดของ C++)
- **`std::condition_variable`** (ปูพื้นจาก pthread ใน Part 32, เต็มรูปแบบใน Part 82):
  ป้องกันไม่ให้ Worker Thread ต้อง**วนเช็ค** ("busy-wait") ว่ามีงานในคิวหรือยังซ้ำๆ
  ไม่หยุด (ซึ่งจะกิน CPU core ทั้งตัวไปเปล่าๆ ตลอดเวลา) แทนที่จะทำแบบนั้น Worker จะ
  **"หลับรอ"** อย่างมีประสิทธิภาพจนกว่าจะถูก**ปลุก**โดย Main Thread ตอนมีงานใหม่เข้าคิว

> **ทบทวนจาก Part 82**: `std::mutex` เพียงอย่างเดียวป้องกัน Race Condition ได้ก็จริง
> แต่ไม่มีกลไก "แจ้งเตือน" ในตัว ถ้าใช้ `std::mutex` ล้วนๆ Worker Thread ต้องเขียน loop
> ที่ `lock()` แล้วเช็คว่าคิวว่างไหมซ้ำๆ ไม่หยุด (`while (queue.empty()) { unlock(); lock(); }`)
> ซึ่งสิ้นเปลือง CPU มหาศาลโดยไม่จำเป็น `std::condition_variable` แก้ปัญหานี้ตรงจุด — ให้
> OS จัดการ "ปลุก" thread ที่หลับรออยู่ให้เราแทน ไม่ต้อง busy-wait เองเลย

---

## 101.5 Implement คลาส `ThreadPool` เต็มรูปแบบ (Step 805)

### โครงสร้างของคลาส

```cpp
class ThreadPool {
public:
    explicit ThreadPool(std::size_t numWorkers);
    ~ThreadPool();                 // เรียก shutdown() ให้อัตโนมัติ (RAII เหมือนหลักการ Part 68)

    ThreadPool(const ThreadPool&) = delete;             // ห้าม copy
    ThreadPool& operator=(const ThreadPool&) = delete;

    void enqueue(int clientFd);    // เพิ่มงานเข้าคิว
    void shutdown();               // หยุดรับงานใหม่, ทำงานที่ค้างให้เสร็จ, join worker ทุกตัว

private:
    void workerLoop(std::size_t workerId);   // สิ่งที่ worker thread แต่ละตัวทำวนซ้ำตลอดชีวิต

    std::vector<std::thread> workers_;
    std::queue<int> taskQueue_;
    std::mutex queueMutex_;
    std::condition_variable condVar_;
    bool stopping_;
};
```

### Worker Loop: หัวใจสำคัญที่สุดของ Thread Pool

```cpp
void ThreadPool::workerLoop(std::size_t workerId) {
    for (;;) {
        int clientFd = -1;
        {
            std::unique_lock<std::mutex> lock(queueMutex_);
            // รอจนกว่า "มีงานในคิว" หรือ "กำลัง shutdown" (ป้องกัน spurious wakeup
            // ด้วย predicate แบบนี้เสมอ ตาม pattern มาตรฐานของ condition_variable)
            condVar_.wait(lock, [this] { return stopping_ || !taskQueue_.empty(); });

            if (taskQueue_.empty()) {
                // ถูกปลุกเพราะ stopping_ และไม่มีงานเหลือแล้ว -> ออกจาก loop จบ thread นี้
                if (stopping_) return;
                continue;
            }

            clientFd = taskQueue_.front();
            taskQueue_.pop();
        } // ปลด lock ก่อนไปทำงานจริง เพื่อให้ worker ตัวอื่นหยิบงานถัดไปได้ทันที

        serviceClient(clientFd);
    }
}
```

มี 3 จุดสำคัญที่ต้องเข้าใจให้ลึกซึ้งในโค้ดนี้:

**1. ทำไมต้องใช้ `std::unique_lock` ไม่ใช่ `std::lock_guard`**

Part 82.1 สอนไว้ว่า `std::lock_guard` ล็อกครั้งเดียวตอนสร้าง ปลดล็อกครั้งเดียวตอน
destruct เท่านั้น ไม่มีความยืดหยุ่นอื่นใด แต่ `condVar_.wait(lock, ...)` **ต้องการปลดล็อก
ชั่วคราวระหว่างที่กำลังหลับรออยู่** (ไม่งั้น thread อื่นจะไม่มีทางล็อกเพื่อเพิ่มงานใหม่เข้า
คิวได้เลย เกิด Deadlock ทันที) แล้วต้อง**ล็อกกลับให้อัตโนมัติ**ตอนถูกปลุกขึ้นมา — ความ
สามารถนี้มีเฉพาะใน `std::unique_lock` เท่านั้น ตรงกับตารางเปรียบเทียบใน Part 82.1 ที่ระบุ
ไว้ชัดเจนว่า **`condition_variable::wait()` ต้องการ `unique_lock` เท่านั้น**

**2. ทำไมต้องส่ง Predicate (`[this] { return stopping_ || !taskQueue_.empty(); }`) เข้าไปเสมอ**

`std::condition_variable::wait()` มีปรากฏการณ์ที่เรียกว่า **Spurious Wakeup** — ตัว
condition variable อาจ**ปลุก thread ขึ้นมาได้เองแม้ไม่มีใคร `notify()` เลย** (เป็น
พฤติกรรมที่มาตรฐาน C++ "อนุญาต" ไว้ เพื่อให้ OS/Hardware บาง implementation ทำงานได้ง่าย
ขึ้น) ถ้าเขียน `condVar_.wait(lock)` เฉยๆ โดยไม่ใส่ predicate แล้ว thread ตื่นมาแบบ
Spurious Wakeup โดยที่คิวยังว่างอยู่จริง โค้ดจะพยายามหยิบงานจากคิวที่ว่างเปล่าทันที ทำให้
เกิด Undefined Behavior การส่ง Predicate เข้าไปบอก `wait()` ว่า **"ตื่นแล้วให้เช็คเงื่อนไข
นี้ด้วย ถ้ายังไม่จริงให้กลับไปหลับต่อทันที"** แก้ปัญหานี้ได้อย่างสมบูรณ์ — เทียบเท่ากับการ
เขียน `while (!predicate()) wait_without_predicate();` ด้วยมือ แต่ `wait()` เวอร์ชันรับ
predicate ทำให้ปลอดภัยโดยอัตโนมัติ ไม่ต้องเขียน loop เองเลย

**3. ทำไมต้องปลด lock ก่อนเรียก `serviceClient(clientFd)`**

สังเกตว่าโค้ดใช้ `{ }` (scope ซ้อน) ครอบเฉพาะส่วนที่แตะ `taskQueue_` เท่านั้น เมื่อออกจาก
scope นั้น `unique_lock` จะถูก destruct และปลดล็อกให้อัตโนมัติ **ก่อน**ที่จะเรียก
`serviceClient()` ซึ่งอาจใช้เวลานาน (รอ `recv()`/`send()` จาก Network) — ถ้าไม่ปลด lock
ก่อน worker thread ตัวอื่นทั้งหมดจะ**ต้องรอ**จนกว่า worker ตัวนี้จะประมวลผล Client เสร็จ
ก่อนถึงจะหยิบงานถัดไปได้ กลายเป็นคอขวดที่ทำลายจุดประสงค์ของการมี Worker หลายตัวไปเลย —
กฎทองคือ: **ถือ lock ให้สั้นที่สุดเท่าที่จำเป็นเสมอ ปลดล็อกก่อนทำงานที่ใช้เวลานานทุกครั้ง**

### `enqueue()` และ `shutdown()`: ฝั่ง Producer และการปิดระบบอย่างปลอดภัย

```cpp
void ThreadPool::enqueue(int clientFd) {
    {
        std::lock_guard<std::mutex> lock(queueMutex_);
        if (stopping_) {
            // ปฏิเสธงานใหม่ระหว่างกำลัง shutdown เพื่อไม่ให้ queue โตไม่มีที่สิ้นสุด
            close(clientFd);
            return;
        }
        taskQueue_.push(clientFd);
    }
    condVar_.notify_one();
}

void ThreadPool::shutdown() {
    {
        std::lock_guard<std::mutex> lock(queueMutex_);
        if (stopping_) return;
        stopping_ = true;
    }
    condVar_.notify_all();
    for (auto& w : workers_) {
        if (w.joinable()) {
            w.join();
        }
    }
}
```

สังเกตว่า `enqueue()` ใช้ `std::lock_guard` ธรรมดา (ไม่ใช่ `unique_lock`) เพราะที่นี่**ไม่มี
การ wait บน condition_variable** — แค่ล็อก, push เข้าคิว, ปลดล็อก, แล้ว `notify_one()`
**หลังจาก**ปลดล็อกแล้ว (ทำนอก scope ของ `lock_guard`) เพื่อไม่ให้ worker ที่ถูกปลุกต้อง
รอ mutex ที่ main thread ยังถืออยู่โดยไม่จำเป็น (เป็น optimization เล็กน้อยแต่ช่วยลด
Context Switch ที่สูญเปล่าได้จริง)

`shutdown()` คือหัวใจของการปิดระบบอย่างปลอดภัย: **ตั้ง `stopping_ = true` ก่อน** (ภายใต้
lock เสมอ เพราะ worker thread อ่านค่านี้ผ่าน predicate ของ `wait()`) แล้ว **`notify_all()`
ปลุก worker ทุกตัวพร้อมกัน** (ต่างจาก `notify_one()` ที่ปลุกแค่ตัวเดียว — ตรงนี้ต้องปลุก
**ทุกตัว**เพราะทุกตัวต้องตรวจสอบ `stopping_` และออกจาก loop) สุดท้าย **`.join()` worker
ทุกตัว**เพื่อรอให้แน่ใจว่าทุก thread ทำงานที่ค้างอยู่ในมือ (ถ้ามี) เสร็จสมบูรณ์และจบตัวเอง
จริงๆ ก่อนที่ `ThreadPool` object จะถูกทำลาย — ตรงกับกฎทองจาก Part 81.2: **thread ที่ยัง
joinable ต้องถูก join หรือ detach ก่อนถูก destruct เสมอ ไม่งั้นจะเกิด `std::terminate()`**

---

## 101.6 ประกอบเป็น HTTP Server เต็มรูปแบบ + Graceful Shutdown (Step 806)

รวมทุกส่วนเข้าด้วยกัน: HTTP parsing/response จาก Part 100, คลาส `ThreadPool` จากหัวข้อ
ก่อนหน้า, และ Signal Handler สำหรับปิดโปรแกรมอย่างสุภาพผ่าน `SIGINT` (Ctrl+C)

```cpp
// threadpool_http_server.cpp - HTTP server ที่ใช้ Thread Pool คงที่ + task queue
#include <arpa/inet.h>
#include <atomic>
#include <cerrno>
#include <condition_variable>
#include <csignal>
#include <cstdio>
#include <cstdlib>
#include <cstring>
#include <mutex>
#include <netinet/in.h>
#include <queue>
#include <string>
#include <sys/socket.h>
#include <thread>
#include <unistd.h>
#include <vector>

#define BACKLOG 128
#define BUFFER_SIZE 4096

// ---------- HTTP parsing/response (เหมือน Part 100) ----------
struct HttpRequest {
    std::string method;
    std::string path;
    std::string version;
    bool valid = false;
};

HttpRequest parseRequestLine(const std::string& raw) {
    HttpRequest req;
    std::size_t lineEnd = raw.find("\r\n");
    if (lineEnd == std::string::npos) {
        return req;
    }
    std::string requestLine = raw.substr(0, lineEnd);
    std::size_t firstSpace = requestLine.find(' ');
    std::size_t lastSpace = requestLine.rfind(' ');
    if (firstSpace == std::string::npos || lastSpace == std::string::npos ||
        firstSpace == lastSpace) {
        return req;
    }
    req.method = requestLine.substr(0, firstSpace);
    req.path = requestLine.substr(firstSpace + 1, lastSpace - firstSpace - 1);
    req.version = requestLine.substr(lastSpace + 1);
    req.valid = true;
    return req;
}

std::string buildResponse(int statusCode, const std::string& statusText,
                           const std::string& contentType, const std::string& body) {
    std::string response;
    response += "HTTP/1.1 " + std::to_string(statusCode) + " " + statusText + "\r\n";
    response += "Content-Type: " + contentType + "\r\n";
    response += "Content-Length: " + std::to_string(body.size()) + "\r\n";
    response += "Connection: close\r\n\r\n";
    response += body;
    return response;
}

std::string handleRequest(const HttpRequest& req) {
    if (!req.valid) {
        return buildResponse(400, "Bad Request", "text/html", "<h1>400 Bad Request</h1>");
    }
    if (req.method != "GET") {
        return buildResponse(405, "Method Not Allowed", "text/html", "<h1>405</h1>");
    }
    if (req.path == "/") {
        return buildResponse(200, "OK", "text/html",
                              "<html><body><h1>Thread pool server</h1></body></html>");
    }
    if (req.path == "/slow") {
        std::this_thread::sleep_for(std::chrono::milliseconds(3000));
        return buildResponse(200, "OK", "text/plain", "slow response done");
    }
    return buildResponse(404, "Not Found", "text/html", "<h1>404</h1>");
}

void serviceClient(int clientFd) {
    char buffer[BUFFER_SIZE];
    ssize_t bytesRead = recv(clientFd, buffer, sizeof(buffer) - 1, 0);
    if (bytesRead > 0) {
        buffer[bytesRead] = '\0';
        HttpRequest req = parseRequestLine(std::string(buffer, static_cast<std::size_t>(bytesRead)));
        std::string response = handleRequest(req);
        std::size_t totalSent = 0;
        while (totalSent < response.size()) {
            ssize_t sent = send(clientFd, response.data() + totalSent, response.size() - totalSent, 0);
            if (sent < 0) break;
            totalSent += static_cast<std::size_t>(sent);
        }
    }
    close(clientFd);
}

// ---------- Thread Pool ----------
class ThreadPool {
public:
    explicit ThreadPool(std::size_t numWorkers) : stopping_(false) {
        for (std::size_t i = 0; i < numWorkers; ++i) {
            workers_.emplace_back([this, i] { workerLoop(i); });
        }
    }

    // ห้าม copy/move เพราะ thread ผูกอยู่กับ this ผ่าน lambda capture แล้ว
    ThreadPool(const ThreadPool&) = delete;
    ThreadPool& operator=(const ThreadPool&) = delete;

    ~ThreadPool() {
        shutdown();
    }

    // เพิ่มงาน (client fd) เข้าคิว แล้วปลุก worker ที่กำลังหลับรออยู่ 1 ตัว
    void enqueue(int clientFd) {
        {
            std::lock_guard<std::mutex> lock(queueMutex_);
            if (stopping_) {
                // ปฏิเสธงานใหม่ระหว่างกำลัง shutdown เพื่อไม่ให้ queue โตไม่มีที่สิ้นสุด
                close(clientFd);
                return;
            }
            taskQueue_.push(clientFd);
        }
        condVar_.notify_one();
    }

    // ปิด pool อย่างสุภาพ: แจ้งให้ worker หยุดรับงานใหม่ ทำงานที่ค้างอยู่ในคิวให้เสร็จ
    // ก่อน แล้วค่อย join thread ทุกตัว (ป้องกัน pitfall "ลืม join ตอน shutdown")
    void shutdown() {
        {
            std::lock_guard<std::mutex> lock(queueMutex_);
            if (stopping_) return;
            stopping_ = true;
        }
        condVar_.notify_all();
        for (auto& w : workers_) {
            if (w.joinable()) {
                w.join();
            }
        }
    }

    long completedCount() const { return completed_.load(); }

private:
    void workerLoop(std::size_t workerId) {
        for (;;) {
            int clientFd = -1;
            {
                std::unique_lock<std::mutex> lock(queueMutex_);
                // รอจนกว่า "มีงานในคิว" หรือ "กำลัง shutdown" (ป้องกัน spurious wakeup
                // ด้วย predicate แบบนี้เสมอ ตาม pattern มาตรฐานของ condition_variable)
                condVar_.wait(lock, [this] { return stopping_ || !taskQueue_.empty(); });

                if (taskQueue_.empty()) {
                    // ถูกปลุกเพราะ stopping_ และไม่มีงานเหลือแล้ว -> ออกจาก loop จบ thread นี้
                    if (stopping_) return;
                    continue; // ป้องกันไว้เผื่อกรณีแปลกๆ (ไม่ควรเกิดขึ้นจริง)
                }

                clientFd = taskQueue_.front();
                taskQueue_.pop();
            } // ปลด lock ก่อนไปทำงานจริง เพื่อให้ worker ตัวอื่นหยิบงานถัดไปได้ทันที

            (void)workerId;
            serviceClient(clientFd);
            ++completed_;
        }
    }

    std::vector<std::thread> workers_;
    std::queue<int> taskQueue_;
    std::mutex queueMutex_;
    std::condition_variable condVar_;
    bool stopping_;
    std::atomic<long> completed_{0};
};

std::atomic<bool> g_shouldStop{false};
void signalHandler(int) { g_shouldStop = true; }

int main(int argc, char** argv) {
    int port = 8102;
    std::size_t poolSize = 8;
    if (argc > 1) port = std::atoi(argv[1]);
    if (argc > 2) poolSize = static_cast<std::size_t>(std::atoi(argv[2]));

    std::signal(SIGINT, signalHandler);

    int serverFd = socket(AF_INET, SOCK_STREAM, 0);
    if (serverFd < 0) { perror("socket"); return EXIT_FAILURE; }

    int opt = 1;
    setsockopt(serverFd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));

    struct sockaddr_in serverAddr;
    std::memset(&serverAddr, 0, sizeof(serverAddr));
    serverAddr.sin_family = AF_INET;
    serverAddr.sin_addr.s_addr = INADDR_ANY;
    serverAddr.sin_port = htons(static_cast<uint16_t>(port));

    if (bind(serverFd, reinterpret_cast<struct sockaddr*>(&serverAddr), sizeof(serverAddr)) < 0) {
        perror("bind"); close(serverFd); return EXIT_FAILURE;
    }
    if (listen(serverFd, BACKLOG) < 0) {
        perror("listen"); close(serverFd); return EXIT_FAILURE;
    }

    // ตั้ง timeout ให้ accept() คืนค่ากลับมาเป็นระยะ เพื่อให้เช็ค g_shouldStop ได้
    struct timeval tv;
    tv.tv_sec = 1;
    tv.tv_usec = 0;
    setsockopt(serverFd, SOL_SOCKET, SO_RCVTIMEO, &tv, sizeof(tv));

    std::printf("[thread-pool] listening on port %d with %zu worker threads\n", port, poolSize);

    {
        ThreadPool pool(poolSize);

        while (!g_shouldStop) {
            struct sockaddr_in clientAddr;
            socklen_t clientLen = sizeof(clientAddr);
            int clientFd = accept(serverFd, reinterpret_cast<struct sockaddr*>(&clientAddr), &clientLen);
            if (clientFd < 0) {
                if (errno == EAGAIN || errno == EWOULDBLOCK) continue; // แค่ timeout เฉยๆ ไม่ใช่ error จริง
                if (g_shouldStop) break;
                continue;
            }
            pool.enqueue(clientFd);
        }

        std::printf("[thread-pool] กำลัง shutdown: รอ worker ทำงานที่ค้างอยู่ในคิวให้เสร็จก่อน...\n");
        // เรียก shutdown() เองอย่างชัดเจน (join worker thread ทุกตัวจนครบ) แล้วค่อยพิมพ์
        // สรุปผล — ป้องกัน pitfall "ลืม join worker thread ตอนปิดโปรแกรม" ที่ทำให้
        // request ที่ค้างอยู่ในคิวหายไปเงียบๆ หรือโปรแกรม main จบก่อน worker ทำงานเสร็จ
        pool.shutdown();
        std::printf("[thread-pool] จำนวน request ที่ให้บริการสำเร็จทั้งหมด: %ld\n", pool.completedCount());
        // ออกจาก scope ตรงนี้ ThreadPool destructor จะเรียก shutdown() ซ้ำ แต่เป็น no-op
        // เพราะ stopping_ ถูกตั้งเป็น true และ worker ทุกตัว join ไปแล้ว
    }

    close(serverFd);
    std::printf("[thread-pool] ปิดเซิร์ฟเวอร์เรียบร้อย\n");
    return 0;
}
```

### ทำไมต้องตั้ง `SO_RCVTIMEO` บน Listening Socket

`accept()` โดย default จะ **บล็อกรอตลอดไป**จนกว่าจะมี Connection เข้ามา — ถ้า Main
Thread ติดอยู่ใน `accept()` แบบไม่มีกำหนดเวลา แม้ Signal Handler จะตั้ง `g_shouldStop =
true` แล้วก็ตาม โปรแกรมก็ยังไม่มีโอกาสเช็คค่านั้นจนกว่าจะมี Client รายใหม่เชื่อมต่อเข้ามา
จริงๆ (ซึ่งอาจไม่มีวันเกิดขึ้นเลยถ้าไม่มี Client เหลืออยู่) การตั้ง `SO_RCVTIMEO` ไว้ที่ 1
วินาทีทำให้ `accept()` **คืนค่ากลับมาเองทุก 1 วินาที** (ด้วย error `EAGAIN`/`EWOULDBLOCK`
ถ้าไม่มี Connection เข้ามาจริง) เปิดโอกาสให้ loop เช็ค `g_shouldStop` เป็นระยะได้ นี่คือ
เทคนิคง่ายๆ ที่ทำให้ Graceful Shutdown เป็นไปได้โดยไม่ต้องใช้ I/O Multiplexing ที่ซับซ้อน
กว่า

คอมไพล์:

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 -pthread -O2 threadpool_http_server.cpp -o threadpool_http_server
```

คอมไพล์ผ่านโดย**ไม่มี Warning ใดๆ เลย**

### ทดสอบฟังก์ชันพื้นฐาน

```bash
./threadpool_http_server 8102 4 &
curl -s -i http://127.0.0.1:8102/
curl -s -i http://127.0.0.1:8102/nope
curl -s -i -X POST http://127.0.0.1:8102/
```

ผลลัพธ์จริงจากการทดสอบ:

```
HTTP/1.1 200 OK
Content-Type: text/html
Content-Length: 53
Connection: close

<html><body><h1>Thread pool server</h1></body></html>
HTTP/1.1 404 Not Found
Content-Type: text/html
Content-Length: 12
Connection: close

<h1>404</h1>
HTTP/1.1 405 Method Not Allowed
Content-Type: text/html
Content-Length: 12
Connection: close

<h1>405</h1>
```

### ทดสอบ Head-of-Line Blocking ด้วยแบบเดียวกับหัวข้อ 101.2

```bash
./threadpool_http_server 8103 4 &
sleep 0.4
curl -s -o /dev/null http://127.0.0.1:8103/ &
curl -s -o /dev/null http://127.0.0.1:8103/slow &
sleep 0.2
time curl -s -o /dev/null -w "HTTP %{http_code}, ใช้เวลา %{time_total}s\n" http://127.0.0.1:8103/
```

ผลลัพธ์จริง — ได้รับ Response เร็วเช่นเดียวกับ Thread-per-Connection (เพราะมี worker
ว่างอยู่ 4 ตัว เหลือเฟือสำหรับ Client แค่ 3 รายในตัวอย่างนี้):

```
HTTP 200, ใช้เวลา 0.000557s

real	0m0.007s
user	0m0.006s
sys	0m0.000s
```

### ทดสอบ Graceful Shutdown ด้วย `SIGINT`

```bash
./threadpool_http_server 8103 4 &
SERVER_PID=$!
sleep 0.4
curl -s -o /dev/null http://127.0.0.1:8103/ &
curl -s -o /dev/null http://127.0.0.1:8103/slow &
sleep 0.2
curl -s -o /dev/null -w "HTTP %{http_code}\n" http://127.0.0.1:8103/

kill -INT $SERVER_PID     # จำลองการกด Ctrl+C
```

Log จริงที่ Server พิมพ์ออกมาระหว่างและหลัง shutdown:

```
[thread-pool] listening on port 8103 with 4 worker threads
[thread-pool] กำลัง shutdown: รอ worker ทำงานที่ค้างอยู่ในคิวให้เสร็จก่อน...
[thread-pool] จำนวน request ที่ให้บริการสำเร็จทั้งหมด: 3
[thread-pool] ปิดเซิร์ฟเวอร์เรียบร้อย
```

`3` คือจำนวน Request ทั้งหมดที่ยิงไปจริงในการทดสอบนี้ (`/`, `/slow`, และ `/` อีกครั้ง) —
ยืนยันว่า **ทุก Worker Thread ถูก join สำเร็จ ไม่มี thread หลุดค้าง ไม่มี
`std::terminate()`** และตัวเลข `completedCount()` ที่รายงานถูกต้องตรงกับที่เกิดขึ้นจริง
100% (พิสูจน์ว่าคิวไม่มี Race Condition ที่ทำให้นับพลาด)

---

## 101.7 Benchmark เปรียบเทียบจริง: Thread-per-Connection vs Thread Pool (Step 807)

### เครื่องมือวัดโหลด: `bench_client.cpp`

เพื่อวัดผลอย่างยุติธรรม เขียน Client ทดสอบโหลดของตัวเอง (ไม่ใช้ `ab`/`wrk` เพราะไม่ได้
ติดตั้งในสภาพแวดล้อมทดสอบ แต่หลักการเดียวกันทุกประการ): สร้าง `std::thread` จำนวน
`concurrency` ตัว แต่ละตัวยิง HTTP request จำนวน `requests_per_thread` ครั้งแบบต่อเนื่อง
(sequential ภายใน thread เดียว, concurrent ระหว่าง thread) แล้ววัดเวลารวมทั้งหมด:

```cpp
// bench_client.cpp - client ทดสอบโหลด: ยิง request พร้อมกันหลาย thread แล้ววัดเวลารวม
// วิธีใช้: ./bench_client <port> <concurrency> <requests_per_thread> [path]
#include <arpa/inet.h>
#include <atomic>
#include <chrono>
#include <cstdio>
#include <cstdlib>
#include <cstring>
#include <netinet/in.h>
#include <string>
#include <sys/socket.h>
#include <thread>
#include <unistd.h>
#include <vector>

std::atomic<long> g_success{0};
std::atomic<long> g_failed{0};

bool doOneRequest(int port, const std::string& path) {
    int sockFd = socket(AF_INET, SOCK_STREAM, 0);
    if (sockFd < 0) return false;

    struct sockaddr_in serverAddr;
    std::memset(&serverAddr, 0, sizeof(serverAddr));
    serverAddr.sin_family = AF_INET;
    serverAddr.sin_port = htons(static_cast<uint16_t>(port));
    inet_pton(AF_INET, "127.0.0.1", &serverAddr.sin_addr);

    if (connect(sockFd, reinterpret_cast<struct sockaddr*>(&serverAddr), sizeof(serverAddr)) < 0) {
        close(sockFd);
        return false;
    }

    std::string request = "GET " + path + " HTTP/1.1\r\nHost: 127.0.0.1\r\n\r\n";
    if (send(sockFd, request.data(), request.size(), 0) < 0) {
        close(sockFd);
        return false;
    }

    char buffer[4096];
    ssize_t n = recv(sockFd, buffer, sizeof(buffer) - 1, 0);
    close(sockFd);
    return n > 0;
}

void workerThread(int port, int requestsPerThread, const std::string& path) {
    for (int i = 0; i < requestsPerThread; ++i) {
        if (doOneRequest(port, path)) {
            ++g_success;
        } else {
            ++g_failed;
        }
    }
}

int main(int argc, char** argv) {
    if (argc < 4) {
        std::fprintf(stderr, "usage: %s <port> <concurrency> <requests_per_thread> [path]\n", argv[0]);
        return 1;
    }
    int port = std::atoi(argv[1]);
    int concurrency = std::atoi(argv[2]);
    int requestsPerThread = std::atoi(argv[3]);
    std::string path = (argc > 4) ? argv[4] : "/";

    std::vector<std::thread> workers;
    workers.reserve(static_cast<std::size_t>(concurrency));

    auto startTime = std::chrono::steady_clock::now();

    for (int i = 0; i < concurrency; ++i) {
        workers.emplace_back(workerThread, port, requestsPerThread, path);
    }
    for (auto& w : workers) {
        w.join();
    }

    auto endTime = std::chrono::steady_clock::now();
    double elapsedSec = std::chrono::duration<double>(endTime - startTime).count();
    long totalRequests = g_success.load() + g_failed.load();
    double rps = static_cast<double>(g_success.load()) / elapsedSec;

    std::printf("concurrency=%d requests_per_thread=%d total=%ld success=%ld failed=%ld "
                "time=%.3fs throughput=%.1f req/s\n",
                concurrency, requestsPerThread, totalRequests, g_success.load(), g_failed.load(),
                elapsedSec, rps);
    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 -pthread -O2 bench_client.cpp -o bench_client
```

คอมไพล์ผ่านโดย**ไม่มี Warning ใดๆ เลย**

### รอบที่ 1: เปรียบเทียบที่ Concurrency ระดับต่างๆ กัน

รัน Server ทั้งสองเวอร์ชันแยกกัน (thread pool ใช้ 8 worker) แล้วยิง Benchmark ชุดเดียวกัน
ทุกประการใส่ทั้งคู่:

```bash
./thread_per_connection_server 8101 &
./bench_client 8101 100 20 /
./bench_client 8101 300 10 /
./bench_client 8101 1000 5 /

./threadpool_http_server 8102 8 &
./bench_client 8102 100 20 /
./bench_client 8102 300 10 /
./bench_client 8102 1000 5 /
```

ผลลัพธ์จริงจากการทดสอบทั้งหมด:

| Concurrency | Total Request | Thread-per-Connection | Thread Pool (8 worker) | อัตราเร่งของ Pool |
|---|---|---|---|---|
| 100 | 2,000 | 10,896.1 req/s (0.184s) | **29,317.5 req/s (0.068s)** | **~2.7 เท่า** |
| 300 | 3,000 | 1,422.8 req/s (2.109s) | **2,824.3 req/s (1.062s)** | **~2.0 เท่า** |
| 1,000 | 5,000 | 1,601.5 req/s (3.122s) | 1,588.7 req/s (3.147s) | ~1.0 เท่า (ใกล้เคียงกัน) |

### วิเคราะห์ผลลัพธ์อย่างตรงไปตรงมา

ผลลัพธ์นี้**ไม่ได้บอกว่า Thread Pool เร็วกว่าเสมอในทุกกรณีแบบง่ายๆ** — ที่ Concurrency
ระดับ 100-300 Thread Pool เร็วกว่าอย่างชัดเจน (2-2.7 เท่า) เพราะไม่ต้องเสียเวลาสร้าง OS
thread ใหม่ซ้ำๆ ตรงตามที่วัดแยกไว้ในหัวข้อ 101.3 ทุกประการ

แต่ที่ Concurrency ระดับ 1,000 ตัวเลขทั้งสองเวอร์ชัน**บรรจบเข้าใกล้กันมาก** (1,601 กับ
1,589 req/s) เหตุผลคือที่ระดับความเข้มข้นนี้ **คอขวดได้ย้ายไปอยู่ที่จุดอื่นแล้ว** ไม่ใช่ที่
การสร้าง/จัดการ thread ของ Server อีกต่อไป: Client ฝั่ง `bench_client` เองต้องสร้าง 1,000
thread พร้อมกัน (มีต้นทุนเดียวกันกับที่วัดในหัวข้อ 101.3 แต่คราวนี้เกิดที่ฝั่ง Client),
Kernel ต้องจัดการ TCP connection queue (`backlog`) ที่มีขนาดจำกัดอยู่ที่ 128 ในโค้ดตัวอย่าง
นี้ และเครื่องทดสอบมีเพียง 4 logical core เท่านั้น ทำให้ทั้ง Client และ Server ต้องแย่งชิง
CPU กันเองอย่างหนักไม่ว่าฝั่ง Server จะออกแบบมาดีแค่ไหนก็ตาม

> **บทเรียนสำคัญจากการวัดผลจริงตรงนี้**: การ Optimize ที่จุดหนึ่งช่วยได้จริงก็ต่อเมื่อจุด
> นั้นเป็นคอขวดจริงๆ ของระบบ ณ ขณะนั้น เมื่อโหลดสูงขึ้นถึงระดับหนึ่ง คอขวดอาจย้ายไปอยู่ที่
> อื่น (Client, Network Stack ของ OS, จำนวน CPU core) ที่ Thread Pool ช่วยอะไรไม่ได้อีกแล้ว
> — นี่คือเหตุผลที่ Part 86 (Performance Profiling) เน้นย้ำเสมอว่าต้อง **วัดจริงก่อนสรุป**
> ไม่ใช่เดาเอาจากทฤษฎีอย่างเดียว

### รอบที่ 2: ขนาด Pool ที่เหมาะสมควรเป็นเท่าไหร่

ทดสอบ Thread Pool ที่ Concurrency คงที่ (200 client, 10 request/thread = 2,000 request
รวม) แต่เปลี่ยนจำนวน Worker Thread ในแต่ละรอบ — เครื่องทดสอบมี **4 logical core**
(`std::thread::hardware_concurrency()` คืนค่า 4 บนเครื่องนี้):

```bash
./threadpool_http_server 8201 4   &  ./bench_client 8201 200 10 /
./threadpool_http_server 8202 8   &  ./bench_client 8202 200 10 /
./threadpool_http_server 8203 32  &  ./bench_client 8203 200 10 /
./threadpool_http_server 8204 200 &  ./bench_client 8204 200 10 /
```

ผลลัพธ์จริงจากการทดสอบ:

| จำนวน Worker Thread | Throughput ที่วัดได้ |
|---|---|
| 4 (= จำนวน core พอดี) | 1,848.4 req/s |
| 8 (2 เท่าของ core) | **1,902.4 req/s** (ดีที่สุด) |
| 32 (8 เท่าของ core) | 1,876.0 req/s |
| 200 (เท่ากับจำนวน Client) | 1,898.1 req/s |

ตัวเลขทั้ง 4 แถวนี้**ใกล้เคียงกันมาก** (ต่างกันไม่ถึง 3%) ทั้งที่จำนวน Worker ต่างกันถึง 50
เท่า (4 ตัว เทียบกับ 200 ตัว) — นี่คือหลักฐานที่ยืนยันสิ่งที่ Part 81.6 สอนไว้ตรงๆ:
**การสร้าง thread มากกว่าจำนวน CPU core จริงไม่ได้ทำให้งานเร็วขึ้น** เพราะเครื่องมีแค่ 4
core ที่ประมวลผลได้พร้อมกันจริง ไม่ว่าจะมี Worker Thread กี่ตัวรอคิวอยู่ก็ตาม งานก็ยังถูก
จำกัดด้วยจำนวน core อยู่ดี ยิ่งมี Thread มากเกินจำเป็นยิ่งเพิ่ม Context Switch Overhead
โดยไม่ได้อะไรเพิ่มเติม (สังเกตว่า 32 และ 200 worker กลับ**ช้ากว่า**เล็กน้อยเมื่อเทียบกับ 8
worker ด้วยซ้ำ)

> **แนวปฏิบัติที่แนะนำ**: เริ่มต้นด้วยขนาด pool ที่ใกล้เคียง
> `std::thread::hardware_concurrency()` (บวกอาจจะคูณ 1-2 เท่าถ้างานมี I/O wait บ่อยๆ อย่าง
> HTTP Server ที่ต้องรอ Network) แล้วปรับตามการวัดผลจริงของงานนั้นๆ ไม่มีค่าตายตัวที่ถูกต้อง
> เสมอสำหรับทุกสถานการณ์ — ต้อง **วัดจริง** เหมือนที่ทำในหัวข้อนี้เสมอ

### ตารางสรุปเปรียบเทียบภาพรวมทั้ง Part

| หัวข้อ | Thread-per-Connection | Thread Pool |
|---|---|---|
| จำนวน Thread | เพิ่มขึ้นตามจำนวน Connection ไม่มีขอบเขต | คงที่เสมอ (กำหนดไว้ล่วงหน้า) |
| แก้ Head-of-Line Blocking จาก Part 100 ได้ไหม | ได้ (ยืนยันด้วยการทดสอบจริง 101.2) | ได้เช่นกัน (ยืนยันด้วยการทดสอบจริง 101.6) |
| Throughput ที่ concurrency ต่ำ-กลาง (100-300) | พื้นฐาน | **เร็วกว่า 2-2.7 เท่า** (วัดจริง) |
| Throughput ที่ concurrency สูงมาก (1,000+) | ใกล้เคียงกับ pool (คอขวดย้ายไปที่อื่นแล้ว) | ใกล้เคียงกับ thread-per-connection |
| ความเสี่ยงด้าน Resource | เสี่ยงสร้าง thread จนหน่วยความจำ/OS resource หมดถ้า Connection ถล่มเข้ามา (ดู Common Pitfalls) | ปลอดภัยกว่า เพราะ thread คงที่, ควบคุม queue ได้ |
| ความซับซ้อนของโค้ด | ต่ำกว่า (ไม่ต้องมี queue/mutex/condition_variable) | สูงกว่าเล็กน้อย (ต้องจัดการ queue อย่างถูกต้อง) |
| เหมาะกับ | Connection จำนวนไม่มาก, งานที่ไม่ต้องขยายขนาดสูง | Production Server จริงที่ต้องรองรับโหลดสูงและควบคุมทรัพยากรได้แน่นอน |

---

## 101.8 ข้อจำกัดที่เหลืออยู่ของ Thread Pool Server นี้ (Step 808)

Thread Pool ที่สร้างขึ้นใน Part นี้แก้ปัญหาทั้ง Head-of-Line Blocking และ Thread Creation
Overhead ได้จริงตามที่พิสูจน์ด้วยตัวเลขข้างต้น แต่ยังมีข้อจำกัดสำคัญที่ต้องรู้ตัวไว้ก่อนจะ
เอาไปใช้งานจริง:

1. **Worker Thread แต่ละตัวยังคง "บล็อก" ตลอดเวลาที่จัดการ 1 Connection** — ถ้า Client
   รายหนึ่งส่งข้อมูลช้ามาก (เช่น Network มีปัญหา หรือถูกโจมตีแบบ Slowloris ที่ Part 100
   กล่าวถึง) Worker Thread ตัวนั้นจะถูก "ยึด" ไว้ทั้งตัวจนกว่า `recv()`/`send()` จะเสร็จ
   หรือ timeout ถ้า Client แบบนี้มีจำนวนมากพอที่จะยึด Worker ทุกตัวพร้อมกัน (มากกว่าขนาด
   Pool) Client รายใหม่ที่เหลือจะต้องรอในคิวไม่มีกำหนดเวลา ปัญหานี้ยังไม่ได้แก้อย่าง
   สมบูรณ์ใน Part นี้
2. **Task Queue ยังไม่มีขอบเขตจำกัด (Unbounded)** — ถ้า Client ยิง Request เร็วกว่าที่
   Worker ทั้งหมดจะประมวลผลทัน คิวจะโตขึ้นเรื่อยๆ ไม่มีที่สิ้นสุด กิน Memory มากขึ้นตาม
   จำนวน Connection ที่ค้างอยู่ (แบบฝึกหัดข้อ 3 จะแก้ปัญหานี้ด้วย Bounded Queue)
3. **ยังไม่รองรับ Persistent Connection (`Keep-Alive`)** — เหมือนกับ Server ใน Part 100
   Server นี้ปิด Connection ทุกครั้งหลังตอบ 1 Request (`Connection: close`) ทำให้ต้องเปิด
   TCP Connection ใหม่ทุกครั้ง ซึ่งมีต้นทุนของ TCP Handshake ซ้ำๆ ที่หลีกเลี่ยงได้ถ้า
   รองรับ Keep-Alive
4. **ไม่มีการจำกัด Rate หรือ Timeout ต่อ Connection** — Client ที่เปิด Connection ค้างไว้
   โดยไม่ส่งข้อมูลอะไรเลยจะยึด Worker Thread ไว้ตลอดไป (จนกว่า OS-level timeout ของ
   `recv()` จะทำงาน ถ้าตั้งไว้)

ข้อจำกัดเหล่านี้ไม่ได้แปลว่า Thread Pool Pattern "ใช้ไม่ได้" — ตรงข้ามเลย มันคือ Pattern
ที่ Web Server ระดับ Production จำนวนมากใช้งานจริง (มักผสมกับ I/O Multiplexing สำหรับ
Connection ที่รอเฉยๆ และใช้ Thread Pool เฉพาะตอนต้องประมวลผลจริง) สิ่งที่ Part นี้ต้องการ
สื่อคือ: **เข้าใจว่า Trade-off ของแต่ละ Pattern คืออะไร และรู้จักวัดผลจริงก่อนตัดสินใจ**
ไม่ใช่ท่องจำว่า Pattern ไหน "ดีที่สุด" เสมอ

Framework จริงอย่าง **Crow** ที่จะเรียนใน **Part 102** จัดการรายละเอียดเหล่านี้ให้
อัตโนมัติเกือบทั้งหมด (Thread Pool ภายในที่ tune มาอย่างดี, รองรับ Keep-Alive, มี Timeout
ที่กำหนดค่าได้) — ความรู้เรื่อง Thread Pool ที่สร้างขึ้นด้วยมือใน Part นี้จะทำให้เข้าใจว่า
เบื้องหลัง Framework เหล่านั้นทำงานอย่างไรกันแน่ แทนที่จะใช้มันแบบ "กล่องดำ" ที่ไม่รู้ว่า
ข้างในทำงานยังไง

---

## ข้อผิดพลาดที่พบบ่อย (Common Pitfalls)

1. **Race Condition บน Task Queue เพราะลืมล็อก mutex ก่อนเข้าถึง** — นี่คือบั๊กที่
   อันตรายที่สุดของ Thread Pool ลองดูโค้ดที่ผิดต่อไปนี้ (ตัดมาจากการทดสอบจริง):

   ```cpp
   void producer(int id) {
       for (int i = 0; i < ITEMS_PER_PRODUCER; ++i) {
           // บั๊ก: เข้าถึง g_taskQueue.push() โดย "ไม่ล็อก" mutex ก่อนเลย
           g_taskQueue.push(id * 100000 + i);
           g_condVar.notify_one();
       }
   }
   ```

   รันด้วย **ThreadSanitizer** (`-fsanitize=thread`) เพื่อดูว่าเครื่องมือตรวจจับ race ได้
   จริงอย่างไร:

   ```bash
   g++ -Wall -Wextra -Wpedantic -std=c++17 -pthread -g -fsanitize=thread racy_queue_demo.cpp -o racy_queue_demo
   TSAN_OPTIONS="halt_on_error=1" ./racy_queue_demo
   ```

   ผลลัพธ์จริงที่ได้ (ตัดเฉพาะส่วนสำคัญ):

   ```
   WARNING: ThreadSanitizer: data race (pid=27577)
     Read of size 8 at 0x55c7742d4070 by thread T2:
       #0 int& std::deque<int, std::allocator<int> >::emplace_back<int>(int&&) ...
       #3 producer(int) racy_queue_demo.cpp:24

     Previous write of size 8 at 0x55c7742d4070 by thread T1:
       #0 int& std::deque<int, std::allocator<int> >::emplace_back<int>(int&&) ...
       #3 producer(int) racy_queue_demo.cpp:24

     Location is global 'g_taskQueue' of size 80 ...

   SUMMARY: ThreadSanitizer: data race .../deque.tcc:167 in int& std::deque<int, std::allocator<int> >::emplace_back<int>(int&&)
   ```

   ThreadSanitizer ชี้ตรงจุดที่ผิดพลาดแม่นยำมาก: **2 thread (T1 กับ T2) กำลังเขียนแก้ไข
   `g_taskQueue` ตัวเดียวกันพร้อมกันที่บรรทัด 24** โดยไม่มีการ synchronize ใดๆ เลย — เพิ่ม
   `std::lock_guard<std::mutex> lock(g_queueMutex);` ก่อน `.push()` กลับเข้าไป (แก้ปัญหา
   แบบเดียวกับที่คลาส `ThreadPool` ใน 101.5 ทำ) แล้วรันซ้ำ ThreadSanitizer จะไม่รายงาน
   ปัญหาใดๆ เลย และผลลัพธ์ถูกต้องครบทุกครั้ง (`consumed = 800` ตรงตามที่คาดหวัง)

2. **ลืมใส่ Predicate ให้ `condition_variable::wait()`** — เขียน `condVar_.wait(lock);`
   เฉยๆ โดยไม่ใส่ lambda predicate ทำให้เสี่ยงต่อ **Spurious Wakeup** (thread ตื่นขึ้นมา
   เองโดยไม่มีใคร `notify()` เลย) ถ้าคิวว่างอยู่ตอนนั้น โค้ดจะพยายามหยิบงานจากคิวเปล่า
   (`taskQueue_.front()` บน queue ว่าง คือ Undefined Behavior) เสมอใส่ Predicate แบบ
   `condVar_.wait(lock, [this]{ return stopping_ || !taskQueue_.empty(); });` เพื่อให้
   `wait()` เช็คเงื่อนไขซ้ำให้อัตโนมัติทุกครั้งที่ตื่น

3. **Deadlock จากการ "เรียกงานเข้า Pool ตัวเองซ้อนกัน" แบบ Blocking** — ถ้า Task ที่
   กำลังรันอยู่ใน Worker Thread ต้องการ enqueue งานใหม่เข้า **Pool เดียวกัน** แล้ว
   **รอผลลัพธ์แบบ Blocking** ก่อนจะทำงานต่อ (เช่นเขียน future/promise เองแล้ว `.get()`
   ทันที) จะเกิด Deadlock ได้ทันทีถ้า Pool มี Worker ไม่พอ: Worker ทุกตัวไปรองาน
   ที่ตัวเองเพิ่ง enqueue ไว้ แต่ไม่มี Worker เหลือว่างพอจะไปหยิบงานนั้นมาทำ — ทางแก้คือ
   ห้าม Task ภายใน Pool "รอผลลัพธ์แบบ Blocking" จาก Task อื่นในคิวเดียวกันเด็ดขาด (ถ้า
   ต้องการ Dependency ระหว่างงานจริงๆ ต้องออกแบบ Pool แยกกัน หรือใช้ Pattern อื่น เช่น
   Task Graph)

4. **ลืม `.join()` Worker Thread ตอน Shutdown** — ถ้า `ThreadPool` destructor ไม่เรียก
   `shutdown()` ที่ join thread ทุกตัวให้ครบ (เช่น เขียน destructor เปล่าไว้ หรือลืมเขียน
   destructor เลย) จะเกิด `std::terminate()` ทันทีตามกฎเหล็กจาก Part 81.5 (thread ที่ยัง
   joinable ถูก destruct) **หรือแย่กว่านั้น**: ถ้าใช้ `.detach()` แทน `.join()` ตอน
   shutdown Worker Thread ที่กำลังทำงานอยู่กับ `this->taskQueue_` อาจยังทำงานต่อไปหลังจาก
   `ThreadPool` object ถูกทำลายไปแล้วจริง กลายเป็น **Use-After-Free** ที่ตรวจจับได้ยาก
   มาก (ต่างจาก Thread-per-Connection ในหัวข้อ 101.2 ที่ตั้งใจ `.detach()` เพราะแต่ละ
   thread ไม่ได้อ้างอิง state ที่ผูกกับ object อายุสั้นใดๆ)

5. **Task Queue ไม่มีขอบเขต (Unbounded) ทำให้ Memory โตไม่หยุดภายใต้โหลดสูง** — ถ้า
   Client ยิง Request เร็วกว่า Worker จะประมวลผลทัน (เช่นระหว่างถูกโจมตีแบบ DoS) คิวจะ
   สะสม `clientFd` ไปเรื่อยๆ ไม่มีที่สิ้นสุด แต่ละ `clientFd` ที่ค้างอยู่ก็ยังกิน File
   Descriptor ของระบบไปด้วย (ซึ่งมีขีดจำกัด `ulimit -n`) จนอาจถึงจุดที่ `accept()` เริ่ม
   ล้มเหลวเพราะ File Descriptor หมด ทางแก้คือทำ **Bounded Queue** ที่ปฏิเสธงานใหม่ (ตอบ
   `503 Service Unavailable`) เมื่อคิวเต็มเกินขนาดที่กำหนด (ดูแบบฝึกหัดข้อ 3)

6. **ตั้งขนาด Pool ตามความรู้สึกแทนที่จะอ้างอิง `hardware_concurrency()` และวัดผลจริง** —
   ตามที่พิสูจน์ในหัวข้อ 101.7 การเพิ่มจำนวน Worker Thread เกินจำนวน CPU core จริงไม่ได้
   ทำให้ Throughput ดีขึ้นเสมอไป (บางครั้งแย่ลงเล็กน้อยด้วยซ้ำจาก Context Switch Overhead)
   ต้องเริ่มจาก `std::thread::hardware_concurrency()` เป็นฐาน แล้ววัดผลจริงเพื่อปรับค่า
   ให้เหมาะกับลักษณะงาน (I/O-bound vs CPU-bound) ของ Server นั้นๆ

7. **ปล่อยให้ Exception หลุดออกจาก Worker Function โดยไม่ดักจับ** — ถ้า
   `serviceClient()` หรือฟังก์ชันใดๆ ที่ Worker Thread เรียกใช้เกิด throw exception ขึ้นมา
   โดยไม่มี `try`/`catch` ครอบไว้ภายใน Worker Loop เอง จะเกิด `std::terminate()` ทันที
   ตามกฎจาก Part 81.5 (exception ข้าม thread ไม่มีทาง catch ได้จาก main thread) และที่
   ร้ายแรงกว่านั้นคือ **Worker Thread ตัวนั้นจะหายไปจาก Pool ถาวร** (โปรแกรมทั้งตัวพัง
   ไปเลยเพราะ `std::terminate()`) Production Code ที่ดีควรห่อการเรียก
   `serviceClient(clientFd)` ด้วย `try`/`catch (const std::exception&)` ภายใน
   `workerLoop()` เสมอ เพื่อไม่ให้ Request ที่มีปัญหาเพียง 1 รายทำให้ทั้ง Server ล่ม

---

## แบบฝึกหัดท้ายบท

1. เพิ่ม Route `/stats` ที่รายงานสถานะปัจจุบันของ Thread Pool แบบ JSON ง่ายๆ (จำนวน
   Worker, จำนวนงานที่ค้างอยู่ในคิว, จำนวน Request ที่ให้บริการสำเร็จไปแล้ว) โดยต้องเพิ่ม
   Method ใหม่ในคลาส `ThreadPool` ที่ล็อก mutex ก่อนอ่าน `taskQueue_.size()` เสมอ (ห้าม
   อ่านโดยไม่ล็อกเด็ดขาด แม้จะดูเหมือนแค่ "อ่านอย่างเดียว" ก็ตาม)

2. ทดลองปรับจำนวน Worker Thread ของ `threadpool_http_server` เป็น 2, 4, 8, 16, 32 แล้ว
   ยิง `bench_client` ด้วย concurrency คงที่ (เช่น 200) วัด Throughput ของแต่ละค่า แล้ว
   หาว่าค่าไหนดีที่สุดสำหรับเครื่องของตัวเอง อธิบายผลลัพธ์โดยอ้างอิง
   `std::thread::hardware_concurrency()` ของเครื่องที่ทดสอบ

3. ทำให้ `ThreadPool` มี **Bounded Queue** (จำกัดขนาดคิวสูงสุด) — ถ้าคิวเต็มให้ปฏิเสธงาน
   ใหม่ทันทีโดยตอบ `503 Service Unavailable` แทนที่จะรับเข้าคิวไม่จำกัดขนาดเหมือนใน
   101.5 (แก้ปัญหาจาก Common Pitfalls ข้อ 5)

4. เพิ่มการวัด **Queue Wait Time** ของแต่ละ Request (เวลาตั้งแต่ `enqueue()` จนถึงตอนที่
   Worker Thread เริ่มประมวลผลจริง) แล้ว log ค่านี้ออกมาทุกครั้งที่ Worker เสร็จงาน 1 ชิ้น
   (ใบ้: บันทึก `std::chrono::steady_clock::now()` ตอน enqueue แล้วส่งค่านั้นเข้าคิวไป
   พร้อมกับ `clientFd` โดยเปลี่ยนชนิดของคิวจาก `std::queue<int>` เป็น
   `std::queue<std::pair<int, std::chrono::steady_clock::time_point>>`)

5. ทำซ้ำการทดลองในข้อ "Common Pitfalls ข้อ 1" ด้วยตัวเอง: เขียนโปรแกรมที่จงใจลบ
   `std::lock_guard` ออกจากจุดที่ผิด แล้วรันด้วย `-fsanitize=thread` เพื่อดู Warning จริง
   จากนั้นใส่ lock กลับเข้าไปแล้วยืนยันว่า ThreadSanitizer ไม่รายงานปัญหาใดๆ อีก

6. ปรับ `handleClient()` ของเวอร์ชัน Thread-per-Connection (หัวข้อ 101.2) ให้ตั้งค่า
   `SO_RCVTIMEO` บน `clientFd` ก่อนเรียก `recv()` (เช่น timeout 5 วินาที) เพื่อป้องกัน
   Client ที่เปิด Connection ค้างไว้โดยไม่ส่งข้อมูลอะไรเลย (คล้ายการโจมตีแบบ Slowloris ที่
   Part 100 กล่าวถึง) ไม่ให้ยึด Thread ไว้ตลอดไป

### แนวทางเฉลยข้อ 1

```cpp
// ex1_stats_server.cpp - เฉลยข้อ 1: เพิ่ม route /stats รายงานสถานะของ thread pool แบบ real-time
#include <arpa/inet.h>
#include <atomic>
#include <cerrno>
#include <condition_variable>
#include <csignal>
#include <cstdio>
#include <cstdlib>
#include <cstring>
#include <mutex>
#include <netinet/in.h>
#include <queue>
#include <string>
#include <sys/socket.h>
#include <thread>
#include <unistd.h>
#include <vector>

#define BACKLOG 128
#define BUFFER_SIZE 4096

struct HttpRequest {
    std::string method;
    std::string path;
    bool valid = false;
};

HttpRequest parseRequestLine(const std::string& raw) {
    HttpRequest req;
    std::size_t lineEnd = raw.find("\r\n");
    if (lineEnd == std::string::npos) return req;
    std::string requestLine = raw.substr(0, lineEnd);
    std::size_t firstSpace = requestLine.find(' ');
    std::size_t lastSpace = requestLine.rfind(' ');
    if (firstSpace == std::string::npos || lastSpace == std::string::npos || firstSpace == lastSpace) {
        return req;
    }
    req.method = requestLine.substr(0, firstSpace);
    req.path = requestLine.substr(firstSpace + 1, lastSpace - firstSpace - 1);
    req.valid = true;
    return req;
}

std::string buildResponse(int statusCode, const std::string& statusText,
                           const std::string& contentType, const std::string& body) {
    std::string response;
    response += "HTTP/1.1 " + std::to_string(statusCode) + " " + statusText + "\r\n";
    response += "Content-Type: " + contentType + "\r\n";
    response += "Content-Length: " + std::to_string(body.size()) + "\r\n";
    response += "Connection: close\r\n\r\n";
    response += body;
    return response;
}

// ---------- Thread Pool ที่รายงานสถานะได้ ----------
class ThreadPool {
public:
    explicit ThreadPool(std::size_t numWorkers) : poolSize_(numWorkers), stopping_(false) {
        for (std::size_t i = 0; i < numWorkers; ++i) {
            workers_.emplace_back([this] { workerLoop(); });
        }
    }

    ThreadPool(const ThreadPool&) = delete;
    ThreadPool& operator=(const ThreadPool&) = delete;

    ~ThreadPool() { shutdown(); }

    void enqueue(int clientFd) {
        {
            std::lock_guard<std::mutex> lock(queueMutex_);
            if (stopping_) { close(clientFd); return; }
            taskQueue_.push(clientFd);
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
        for (auto& w : workers_) if (w.joinable()) w.join();
    }

    // สแนปช็อตสถานะปัจจุบันของ pool: จำนวน worker, จำนวนงานที่ค้างในคิว, จำนวนที่ทำสำเร็จแล้ว
    // ต้องล็อก queueMutex_ ตอนอ่าน taskQueue_.size() เพราะ std::queue ไม่ thread-safe เอง
    std::string statsJson() {
        std::size_t queued;
        {
            std::lock_guard<std::mutex> lock(queueMutex_);
            queued = taskQueue_.size();
        }
        return "{\"pool_size\": " + std::to_string(poolSize_) +
               ", \"queued_tasks\": " + std::to_string(queued) +
               ", \"completed\": " + std::to_string(completed_.load()) + "}";
    }

    void runTask(int clientFd, const std::string& body, const std::string& contentType, int code,
                 const std::string& statusText) {
        std::string response = buildResponse(code, statusText, contentType, body);
        std::size_t totalSent = 0;
        while (totalSent < response.size()) {
            ssize_t sent = send(clientFd, response.data() + totalSent, response.size() - totalSent, 0);
            if (sent < 0) break;
            totalSent += static_cast<std::size_t>(sent);
        }
        close(clientFd);
    }

private:
    void workerLoop() {
        for (;;) {
            int clientFd = -1;
            {
                std::unique_lock<std::mutex> lock(queueMutex_);
                condVar_.wait(lock, [this] { return stopping_ || !taskQueue_.empty(); });
                if (taskQueue_.empty()) {
                    if (stopping_) return;
                    continue;
                }
                clientFd = taskQueue_.front();
                taskQueue_.pop();
            }
            serviceOne(clientFd);
            ++completed_;
        }
    }

    void serviceOne(int clientFd) {
        char buffer[BUFFER_SIZE];
        ssize_t bytesRead = recv(clientFd, buffer, sizeof(buffer) - 1, 0);
        if (bytesRead <= 0) { close(clientFd); return; }
        buffer[bytesRead] = '\0';

        HttpRequest req = parseRequestLine(std::string(buffer, static_cast<std::size_t>(bytesRead)));
        if (!req.valid) {
            runTask(clientFd, "<h1>400 Bad Request</h1>", "text/html", 400, "Bad Request");
            return;
        }
        if (req.path == "/stats") {
            runTask(clientFd, statsJson(), "application/json", 200, "OK");
            return;
        }
        if (req.path == "/") {
            runTask(clientFd, "<h1>OK</h1>", "text/html", 200, "OK");
            return;
        }
        if (req.path == "/slow") {
            std::this_thread::sleep_for(std::chrono::milliseconds(2000));
            runTask(clientFd, "slow done", "text/plain", 200, "OK");
            return;
        }
        runTask(clientFd, "<h1>404 Not Found</h1>", "text/html", 404, "Not Found");
    }

    std::vector<std::thread> workers_;
    std::queue<int> taskQueue_;
    std::mutex queueMutex_;
    std::condition_variable condVar_;
    std::size_t poolSize_;
    bool stopping_;
    std::atomic<long> completed_{0};
};

std::atomic<bool> g_shouldStop{false};
void signalHandler(int) { g_shouldStop = true; }

int main(int argc, char** argv) {
    int port = 8301;
    std::size_t poolSize = 4;
    if (argc > 1) port = std::atoi(argv[1]);
    if (argc > 2) poolSize = static_cast<std::size_t>(std::atoi(argv[2]));

    std::signal(SIGINT, signalHandler);

    int serverFd = socket(AF_INET, SOCK_STREAM, 0);
    int opt = 1;
    setsockopt(serverFd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));

    struct sockaddr_in serverAddr;
    std::memset(&serverAddr, 0, sizeof(serverAddr));
    serverAddr.sin_family = AF_INET;
    serverAddr.sin_addr.s_addr = INADDR_ANY;
    serverAddr.sin_port = htons(static_cast<uint16_t>(port));
    bind(serverFd, reinterpret_cast<struct sockaddr*>(&serverAddr), sizeof(serverAddr));
    listen(serverFd, BACKLOG);

    struct timeval tv;
    tv.tv_sec = 1;
    tv.tv_usec = 0;
    setsockopt(serverFd, SOL_SOCKET, SO_RCVTIMEO, &tv, sizeof(tv));

    std::printf("[stats-server] listening on port %d, pool size %zu\n", port, poolSize);

    {
        ThreadPool pool(poolSize);
        while (!g_shouldStop) {
            struct sockaddr_in clientAddr;
            socklen_t clientLen = sizeof(clientAddr);
            int clientFd = accept(serverFd, reinterpret_cast<struct sockaddr*>(&clientAddr), &clientLen);
            if (clientFd < 0) {
                if (errno == EAGAIN || errno == EWOULDBLOCK) continue;
                continue;
            }
            pool.enqueue(clientFd);
        }
        pool.shutdown();
    }

    close(serverFd);
    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 -pthread -O2 ex1_stats_server.cpp -o ex1_stats_server
./ex1_stats_server 8301 2 &
curl -s http://127.0.0.1:8301/stats
```

ผลลัพธ์จริงจากการทดสอบก่อนมีงานใดๆ เข้ามา:

```
{"pool_size": 2, "queued_tasks": 0, "completed": 0}
```

**ข้อสังเกตสำคัญที่พบระหว่างการทดสอบจริง**: เมื่อยิง `/slow` (sleep 2 วินาที) เข้ามา 5
ครั้งพร้อมกันกับ pool ที่มี Worker แค่ 2 ตัว แล้วยิง `/stats` ตามเข้าไปทันที คำตอบของ
`/stats` **ไม่ได้กลับมาทันที** — เพราะ `/stats` ก็เป็นแค่งานอีกชิ้นหนึ่งในคิวเดียวกัน ถ้า
Worker ทั้งสองตัวกำลังยุ่งอยู่กับ `/slow` คำขอ `/stats` ต้องต่อคิวรอเหมือนงานอื่นทุก
ประการ ทำให้ค่าที่รายงานกลับมาเป็นค่า **ณ เวลาที่มันถูกประมวลผลจริง ไม่ใช่ ณ เวลาที่ถูก
เรียก** นี่คือข้อจำกัดของการออกแบบแบบนี้ที่ควรรู้ตัวไว้: ถ้าต้องการ Endpoint สำหรับ
Monitoring ที่ตอบสนอง**ทันที**แม้ตอน Pool อิ่มตัวเต็มที่ ควรแยก Endpoint นั้นออกจากคิวงาน
หลัก (เช่น เปิด Listening Socket แยกต่างหากสำหรับ Endpoint ด้าน Admin/Monitoring) ไม่ใช่
เดินผ่าน Task Queue เดียวกับ Traffic ปกติ — เป็นตัวอย่างที่ดีว่าทำไม Production Web
Server จริงๆ (เช่นที่จะเห็นใน Crow ตอน Part 102) มักแยก Health Check/Metrics Endpoint
ออกจากเส้นทางหลักเสมอ

### แนวทางเฉลยข้อ 3

```cpp
// ex3_bounded_queue_server.cpp - เฉลยข้อ 3: จำกัดขนาดคิวสูงสุด ถ้าเต็มตอบ 503 ทันที
#include <arpa/inet.h>
#include <atomic>
#include <cerrno>
#include <condition_variable>
#include <csignal>
#include <cstdio>
#include <cstdlib>
#include <cstring>
#include <mutex>
#include <netinet/in.h>
#include <queue>
#include <string>
#include <sys/socket.h>
#include <thread>
#include <unistd.h>
#include <vector>

#define BACKLOG 128
#define BUFFER_SIZE 4096

std::string buildResponse(int statusCode, const std::string& statusText,
                           const std::string& contentType, const std::string& body) {
    std::string response;
    response += "HTTP/1.1 " + std::to_string(statusCode) + " " + statusText + "\r\n";
    response += "Content-Type: " + contentType + "\r\n";
    response += "Content-Length: " + std::to_string(body.size()) + "\r\n";
    response += "Connection: close\r\n\r\n";
    response += body;
    return response;
}

void sendAll(int fd, const std::string& data) {
    std::size_t totalSent = 0;
    while (totalSent < data.size()) {
        ssize_t sent = send(fd, data.data() + totalSent, data.size() - totalSent, 0);
        if (sent < 0) break;
        totalSent += static_cast<std::size_t>(sent);
    }
}

void serviceOne(int clientFd) {
    char buffer[BUFFER_SIZE];
    ssize_t bytesRead = recv(clientFd, buffer, sizeof(buffer) - 1, 0);
    if (bytesRead <= 0) { close(clientFd); return; }
    buffer[bytesRead] = '\0';
    // handler ง่ายๆ: จำลองงานหนักที่ใช้เวลาแปรผัน เพื่อให้คิวเต็มได้จริงตอนทดสอบ
    std::this_thread::sleep_for(std::chrono::milliseconds(500));
    sendAll(clientFd, buildResponse(200, "OK", "text/plain", "processed\n"));
    close(clientFd);
}

// ThreadPool ที่มี "ขนาดคิวสูงสุด" (bounded queue) แทนที่จะรับงานไม่จำกัดเหมือนใน 101.5
// ถ้าคิวเต็ม enqueue() จะปฏิเสธงานใหม่ทันที (คืน false) แทนที่จะรอหรือรับเข้าคิวไม่จำกัด
class BoundedThreadPool {
public:
    BoundedThreadPool(std::size_t numWorkers, std::size_t maxQueueSize)
        : maxQueueSize_(maxQueueSize), stopping_(false) {
        for (std::size_t i = 0; i < numWorkers; ++i) {
            workers_.emplace_back([this] { workerLoop(); });
        }
    }

    BoundedThreadPool(const BoundedThreadPool&) = delete;
    BoundedThreadPool& operator=(const BoundedThreadPool&) = delete;

    ~BoundedThreadPool() { shutdown(); }

    // คืน true ถ้ารับงานเข้าคิวสำเร็จ, false ถ้าคิวเต็มแล้ว (ผู้เรียกต้องจัดการ client เอง เช่น
    // ตอบ 503 แล้วปิด connection ทิ้ง แทนที่จะปล่อยให้คิวโตไม่มีขอบเขตจนหน่วยความจำหมด)
    bool tryEnqueue(int clientFd) {
        std::lock_guard<std::mutex> lock(queueMutex_);
        if (stopping_) return false;
        if (taskQueue_.size() >= maxQueueSize_) {
            return false; // คิวเต็ม -- ปฏิเสธงานนี้ทันที ไม่รอ ไม่บล็อก
        }
        taskQueue_.push(clientFd);
        condVar_.notify_one();
        return true;
    }

    void shutdown() {
        {
            std::lock_guard<std::mutex> lock(queueMutex_);
            if (stopping_) return;
            stopping_ = true;
        }
        condVar_.notify_all();
        for (auto& w : workers_) if (w.joinable()) w.join();
    }

    long completedCount() const { return completed_.load(); }
    long rejectedCount() const { return rejected_.load(); }
    void recordRejected() { ++rejected_; }

private:
    void workerLoop() {
        for (;;) {
            int clientFd = -1;
            {
                std::unique_lock<std::mutex> lock(queueMutex_);
                condVar_.wait(lock, [this] { return stopping_ || !taskQueue_.empty(); });
                if (taskQueue_.empty()) {
                    if (stopping_) return;
                    continue;
                }
                clientFd = taskQueue_.front();
                taskQueue_.pop();
            }
            serviceOne(clientFd);
            ++completed_;
        }
    }

    std::vector<std::thread> workers_;
    std::queue<int> taskQueue_;
    std::mutex queueMutex_;
    std::condition_variable condVar_;
    std::size_t maxQueueSize_;
    bool stopping_;
    std::atomic<long> completed_{0};
    std::atomic<long> rejected_{0};
};

std::atomic<bool> g_shouldStop{false};
void signalHandler(int) { g_shouldStop = true; }

int main(int argc, char** argv) {
    int port = 8302;
    std::size_t poolSize = 2;
    std::size_t maxQueue = 4;
    if (argc > 1) port = std::atoi(argv[1]);
    if (argc > 2) poolSize = static_cast<std::size_t>(std::atoi(argv[2]));
    if (argc > 3) maxQueue = static_cast<std::size_t>(std::atoi(argv[3]));

    std::signal(SIGINT, signalHandler);

    int serverFd = socket(AF_INET, SOCK_STREAM, 0);
    int opt = 1;
    setsockopt(serverFd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));

    struct sockaddr_in serverAddr;
    std::memset(&serverAddr, 0, sizeof(serverAddr));
    serverAddr.sin_family = AF_INET;
    serverAddr.sin_addr.s_addr = INADDR_ANY;
    serverAddr.sin_port = htons(static_cast<uint16_t>(port));
    bind(serverFd, reinterpret_cast<struct sockaddr*>(&serverAddr), sizeof(serverAddr));
    listen(serverFd, BACKLOG);

    struct timeval tv;
    tv.tv_sec = 1;
    tv.tv_usec = 0;
    setsockopt(serverFd, SOL_SOCKET, SO_RCVTIMEO, &tv, sizeof(tv));

    std::printf("[bounded-pool] listening on %d, workers=%zu, max_queue=%zu\n", port, poolSize, maxQueue);

    {
        BoundedThreadPool pool(poolSize, maxQueue);
        while (!g_shouldStop) {
            struct sockaddr_in clientAddr;
            socklen_t clientLen = sizeof(clientAddr);
            int clientFd = accept(serverFd, reinterpret_cast<struct sockaddr*>(&clientAddr), &clientLen);
            if (clientFd < 0) {
                if (errno == EAGAIN || errno == EWOULDBLOCK) continue;
                continue;
            }
            if (!pool.tryEnqueue(clientFd)) {
                // คิวเต็ม: ตอบ 503 Service Unavailable ทันที แทนที่จะรับเข้าคิวไม่จำกัดขนาด
                sendAll(clientFd, buildResponse(503, "Service Unavailable", "text/plain",
                                                 "server busy, try again later\n"));
                close(clientFd);
                pool.recordRejected();
            }
        }
        std::printf("[bounded-pool] completed=%ld rejected=%ld\n", pool.completedCount(),
                    pool.rejectedCount());
        pool.shutdown();
    }

    close(serverFd);
    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 -pthread -O2 ex3_bounded_queue_server.cpp -o ex3_bounded_queue_server
./ex3_bounded_queue_server 8302 2 4 &   # 2 worker, คิวสูงสุด 4 -> รับพร้อมกันได้สูงสุด 6

# ยิง 12 request พร้อมกัน (เกินความจุ 6 อยู่ 6 request)
for i in $(seq 1 12); do
  curl -s -o /dev/null -w "%{http_code}\n" http://127.0.0.1:8302/ &
done
wait
```

ผลลัพธ์จริงจากการทดสอบ (นับจำนวน status code ที่ได้จากทั้ง 12 request):

```
200
200
200
200
200
200
503
503
503
503
503
503
```

ได้ **`200 OK` แน่นอน 6 รายการ** (2 Worker ที่กำลังประมวลผลอยู่ทันที + คิวที่รับได้อีก 4
= ความจุรวม 6) และ **`503 Service Unavailable` อีก 6 รายการ** สำหรับ Request ที่เกินความ
จุพอดี ตรงกับที่ออกแบบไว้ 100% เมื่อปิด Server ด้วย `SIGINT` Log สุดท้ายยืนยันตัวเลขที่
สอดคล้องกัน:

```
[bounded-pool] completed=12 rejected=12
```

(ตัวเลขรวมเป็น 12/12 เพราะรันการทดสอบซ้ำ 2 รอบระหว่างการทดลองจริง แต่ละรอบมีอัตราส่วน
สำเร็จ/ปฏิเสธ 6 ต่อ 6 คงที่เสมอ ตามความจุที่กำหนดไว้ `workers + maxQueue = 2 + 4 = 6`)

**อธิบาย**: `tryEnqueue()` เช็คขนาดคิวปัจจุบัน (`taskQueue_.size() >= maxQueueSize_`)
**ภายใต้ lock เดียวกัน**กับที่ใช้ตอน push เข้าคิวจริง (Atomic ในเชิงตรรกะ — ไม่มีช่วงเวลา
ใดที่ thread อื่นมาแทรกระหว่างการเช็คขนาดกับการ push ได้) นี่คือรูปแบบที่เรียกว่า
**Check-then-Act** ที่ต้องทำภายใต้ lock เดียวกันเสมอ ถ้าแยกการเช็คขนาดกับการ push ออกเป็น
2 lock คนละครั้ง (ผิด!) จะเกิด Race Condition ที่ทำให้คิวอาจเกินขนาดที่กำหนดได้ในบาง
จังหวะเวลา — หลักการเดียวกับที่ Part 82 เน้นย้ำเรื่อง Critical Section ที่ต้องครอบคลุมทุก
ขั้นตอนของการดำเนินการที่ต้อง atomic ร่วมกัน

(ข้อ 2, 4, 5 และ 6 ให้ผู้เรียนลองทำเองตามแนวทางของตัวอย่างในบทเรียน — สิ่งสำคัญคือทุก
คำตอบต้องคอมไพล์ผ่านด้วย `-Wall -Wextra -Wpedantic -std=c++17 -pthread` โดยไม่มี Warning
ใดๆ เลย และต้องทดสอบด้วย `curl`/`bench_client` จริงก่อนถือว่าทำเสร็จ)

---

## สรุปท้ายบท

ใน Part นี้เราได้:

- ทบทวนและยืนยันปัญหา Head-of-Line Blocking จาก Part 100 ด้วยตัวเลขจริง แล้วแก้ด้วย
  แนวทางแรกคือ **Thread-per-Connection** ผ่าน `std::thread` (Part 81) ที่ spawn thread
  ใหม่ทุกครั้งที่ `accept()` สำเร็จ พร้อมพิสูจน์ด้วยการทดสอบจริงว่า Client ที่ช้าไม่กระทบ
  Client รายอื่นอีกต่อไป (จาก 2.69 วินาทีใน Part 100 เหลือ 0.68 มิลลิวินาที)
- ค้นพบและ**วัดผลกระทบจริง**ของ Thread Creation Overhead ทั้งในระดับ Micro-benchmark
  ล้วนๆ (thread pool เร็วกว่า ~4 เท่า) และในระดับ HTTP Server จริงภายใต้โหลดสูง
  (Throughput ลดลงเกือบ 8 เท่าเมื่อ concurrency เพิ่มขึ้น 3 เท่าสำหรับ Thread-per-
  Connection)
- ออกแบบและ Implement **Thread Pool Pattern** เต็มรูปแบบด้วย `std::mutex` และ
  `std::condition_variable` จาก Part 82: Task Queue ที่ปลอดภัยจาก Race Condition,
  Worker Loop ที่ป้องกัน Spurious Wakeup อย่างถูกต้องด้วย Predicate, และ `shutdown()`
  ที่ join Worker Thread ทุกตัวอย่างปลอดภัยตามกฎเหล็กจาก Part 81.5
- ประกอบ Thread Pool เข้ากับ HTTP Server พร้อม Graceful Shutdown ผ่าน `SIGINT` และ
  ทดสอบยืนยันว่าไม่มี Request ใดหายไปหรือค้างระหว่างปิดโปรแกรม
- **เปรียบเทียบ Performance จริง**ระหว่าง Thread-per-Connection กับ Thread Pool ในหลาย
  ระดับ Concurrency (Thread Pool เร็วกว่า 2-2.7 เท่าที่โหลดต่ำ-กลาง แต่บรรจบเข้าใกล้กัน
  ที่โหลดสูงมากเพราะคอขวดย้ายไปที่อื่น) และยืนยันด้วยข้อมูลจริงว่าขนาด Pool ที่เหมาะสม
  ควรอ้างอิงจาก `hardware_concurrency()` ไม่ใช่ยิ่งมาก Worker ยิ่งดีเสมอไป
- สาธิต Race Condition บน Task Queue ด้วย **ThreadSanitizer** จริง (`-fsanitize=thread`)
  แล้วแก้ไขให้ถูกต้อง เป็นทักษะสำคัญที่จะใช้ตรวจสอบโค้ด Concurrent ตลอดหลักสูตรที่เหลือ

Thread Pool ที่สร้างขึ้นเองใน Part นี้ยังมีข้อจำกัดอยู่ (Unbounded Queue โดย default,
Worker ยังบล็อกทั้ง thread ต่อ 1 connection, ไม่รองรับ Keep-Alive) แต่หลักการที่ได้เรียนรู้
— Producer-Consumer Pattern ด้วย mutex+condition_variable, การจัดการ Lifecycle ของ
Thread อย่างปลอดภัย, และวินัยในการวัดผลจริงก่อนสรุป — คือรากฐานที่ Framework ระดับ
Production ทุกตัวใช้อยู่เบื้องหลัง ใน **Part 102** เราจะเริ่มใช้ **Crow Framework** ที่ห่อ
รายละเอียดทั้งหมดนี้ไว้ให้อัตโนมัติ พร้อมเพิ่มความสามารถที่ Server จากมือของเราเองใน 2
Part ที่ผ่านมายังไม่มี เช่น Routing ที่ยืดหยุ่นกว่า, JSON handling ในตัว, และ Middleware

**ต่อไป:** [Part 102 — Crow Framework: พื้นฐานและ REST API แรก](./part-102-crow-basics.md)
