# Part 100: สร้าง Raw HTTP Server จาก Socket ด้วยมือ (Step 793–800)

> Module I — Web Development ด้วย C/C++ | Part 100 จาก 125
> Step ที่ครอบคลุมใน Part นี้: Step 793–800
> Part ก่อนหน้า: [Part 99 — ภาพรวม Web Development ด้วย C/C++](./part-099-web-dev-overview.md) | Part ถัดไป: [Part 101 — Multi-threaded HTTP Server พร้อม Thread Pool](./part-101-multithreaded-http-server.md)

## เป้าหมายการเรียนรู้ (Learning Objectives)

หลังจบ Part นี้ ผู้เรียนจะสามารถ:

1. สร้าง HTTP Server เปล่าๆ ด้วย Socket API ล้วนๆ (ไม่ใช้ Library ใดๆ) โดยต่อยอดจากความรู้
   `socket()`/`bind()`/`listen()`/`accept()` ที่เรียนไปแล้วใน Part 33
2. Parse HTTP Request Line ด้วยมือ (แยก Method, Path, Version ออกจาก string ดิบที่ได้จาก
   `recv()`) โดยไม่พึ่งพา Library แยก HTTP โดยเฉพาะ
3. สร้าง HTTP Response ที่ถูกต้องตาม Spec ครบทุกส่วน: Status Line, Header, Body, และ
   `Content-Length` ที่คำนวณถูกต้องเสมอ
4. เขียน Router ง่ายๆ ที่ตัดสินใจว่าจะตอบอะไรกลับไปตาม Path ที่ Client ร้องขอ
5. ทดสอบ Server ที่เขียนเองด้วย `curl` และ Web Browser จริง อ่านและตีความผลลัพธ์ที่ได้อย่าง
   ถูกต้อง
6. อธิบายและสาธิตบั๊กร้ายแรงที่พบบ่อยที่สุดของมือใหม่ที่เขียน HTTP Server เอง: การลืมใส่
   `Content-Length` ที่ทำให้ Client ค้างรอข้อมูลที่ไม่มีวันมาถึง
7. อธิบายข้อจำกัดสำคัญของ Server ที่สร้างใน Part นี้ (รับได้ทีละ 1 Connection เท่านั้น) และ
   เชื่อมโยงไปสู่เหตุผลที่ต้องเรียน Multi-threading ต่อใน Part 101

---

## 100.1 ทบทวนเป้าหมาย: ทำไมต้องเขียน HTTP Server จาก Socket ด้วยมือก่อน (Step 793)

Part 99 อธิบายไปแล้วว่า Framework อย่าง Crow, Pistache, Drogon จะทำให้เราเขียน Web Server
ได้ง่ายกว่านี้มาก แต่ Part 100-101 จงใจ**ไม่ใช้ Framework เหล่านั้นเลย** เพราะเป้าหมายไม่ใช่
"สร้าง Web Server ที่ใช้งานได้เร็วที่สุด" แต่คือ **"เข้าใจว่าเบื้องหลัง Framework ทุกตัวทำอะไร
อยู่กันแน่"**

ทุกอย่างที่เราจะเขียนใน Part นี้ต่อยอดโดยตรงจาก Part 33 (Socket Programming เบื้องต้น) แทบ
ไม่มีอะไรใหม่ในแง่ Socket API เลย — สิ่งที่ใหม่คือ **การนำ Socket มาประกอบเป็น HTTP Server
จริง** โดยเพิ่ม 2 ส่วนที่ Part 33 ยังไม่มี:

```
Part 33 (TCP Echo Server):        Part 100 (HTTP Server):

recv() ข้อมูลดิบ                    recv() ข้อมูลดิบ
   │                                   │
   ▼                                   ▼
ส่งข้อมูลเดิมกลับไป (echo)          [ใหม่] Parse เป็น HTTP Request
                                       │  (Method, Path, Version)
                                       ▼
                                   [ใหม่] ตัดสินใจตอบอะไร (Routing)
                                       │
                                       ▼
                                   [ใหม่] สร้าง HTTP Response ตาม Spec
                                       │
                                       ▼
                                   send() กลับไป
```

ผลลัพธ์สุดท้ายของ Part นี้คือโปรแกรม C++ หนึ่งตัวที่เปิด Browser หรือใช้ `curl` เข้าไปที่
`http://127.0.0.1:8100/` แล้วได้หน้าเว็บจริงกลับมา — ไม่มีอะไรวิเศษ ไม่มี "มายากล" ซ่อนอยู่
เบื้องหลังเลยแม้แต่น้อย มีแค่ `socket()`, `recv()`, `send()`, และ `std::string` ที่เราประกอบ
กันเองทั้งหมด

---

## 100.2 Parse HTTP Request Line ด้วยมือ (Step 794)

จาก Part 99.3 เราทราบแล้วว่า HTTP Request มีรูปแบบ:

```
GET /index.html HTTP/1.1\r\n
Host: 127.0.0.1:8100\r\n
User-Agent: curl/8.5.0\r\n
\r\n
```

**Request Line** (บรรทัดแรกสุด) คือส่วนที่สำคัญที่สุดสำหรับ Server ง่ายๆ ของเรา เพราะบอกว่า
Client ต้องการอะไร: `<Method> <Path> <Version>\r\n` คั่นด้วยช่องว่าง (space) 1 ตัว

การ Parse ในระดับนี้ไม่จำเป็นต้องใช้ Regular Expression หรือ Parser ที่ซับซ้อน เพราะ Format
ตายตัวชัดเจน — ใช้ `std::string::find()` หา `\r\n` ตัวแรก (จบ Request Line) แล้วหา Space
ตัวแรกกับตัวสุดท้ายภายในบรรทัดนั้นก็เพียงพอ:

```cpp
struct HttpRequest {
    std::string method;
    std::string path;
    std::string version;
    bool valid = false;
};

// แยก request line แบบง่ายๆ: "GET /path HTTP/1.1\r\n"
HttpRequest parseRequestLine(const std::string& raw) {
    HttpRequest req;
    std::size_t lineEnd = raw.find("\r\n");
    if (lineEnd == std::string::npos) {
        return req;   // ไม่เจอ \r\n เลย -> request ผิดรูปแบบ ส่ง req.valid = false กลับไป
    }
    std::string requestLine = raw.substr(0, lineEnd);

    std::size_t firstSpace = requestLine.find(' ');
    std::size_t lastSpace = requestLine.rfind(' ');
    if (firstSpace == std::string::npos || lastSpace == std::string::npos ||
        firstSpace == lastSpace) {
        return req;   // ไม่มี space ครบ 2 ตัว -> ไม่ใช่ request line ที่ถูกต้อง
    }

    req.method = requestLine.substr(0, firstSpace);
    req.path = requestLine.substr(firstSpace + 1, lastSpace - firstSpace - 1);
    req.version = requestLine.substr(lastSpace + 1);
    req.valid = true;
    return req;
}
```

### ทำไมใช้ `find(' ')` ตัวแรกกับ `rfind(' ')` ตัวสุดท้าย แทนการ split ด้วย space ทั้งหมด

Path บาง URL อาจมี space เข้ารหัสอยู่ (`%20`) แต่ไม่ควรมี space ดิบๆ ปนอยู่ใน Path เอง ตาม
Spec ของ HTTP Request Line ที่ถูกต้องจะมี Space แค่ 2 ตัวเท่านั้น (คั่น Method-Path และ
Path-Version) การหา Space ตัวแรกกับตัวสุดท้ายจึงเพียงพอและทนทานกว่าการ `split()` ด้วย Space
ทั้งหมดตรงๆ ซึ่งจะพังทันทีถ้ามีเหตุผลใดที่ Path มี Space มากกว่าที่คาด

> Parser นี้เป็นเวอร์ชัน **"ง่ายที่สุดเท่าที่จะเป็นไปได้"** จงใจไม่ Parse Header ทั้งหมด (เช่น
> `Host`, `User-Agent`) เพราะ Server ของเรายังไม่ต้องใช้ข้อมูลเหล่านั้น การ Parse HTTP แบบเต็ม
> รูปแบบที่รองรับทุก Edge Case (Header หลายบรรทัด, Chunked Transfer Encoding, ฯลฯ) คืองานที่
> Library อย่าง Crow ทำให้เราอัตโนมัติใน Part 102 เป็นต้นไป

---

## 100.3 สร้าง HTTP Response ที่ถูกต้องตาม Spec (Step 795)

นี่คือหัวใจสำคัญที่สุดของ Part นี้ HTTP Response ที่ถูกต้องต้องมีส่วนประกอบครบ 4 ส่วนตามลำดับ
เป๊ะๆ:

```
HTTP/1.1 200 OK\r\n              <- [1] Status Line
Content-Type: text/html\r\n      <- [2] Header (มีได้หลายบรรทัด)
Content-Length: 42\r\n           <- [2] Header ที่สำคัญที่สุด (อธิบายละเอียดด้านล่าง)
\r\n                             <- [3] บรรทัดว่าง คั่น Header กับ Body (ขาดไม่ได้!)
<html>...</html>                 <- [4] Body
```

```cpp
// สร้าง HTTP response ที่ถูกต้องตาม spec: status line + header + body + Content-Length
std::string buildResponse(int statusCode, const std::string& statusText,
                           const std::string& contentType, const std::string& body) {
    std::string response;
    response += "HTTP/1.1 " + std::to_string(statusCode) + " " + statusText + "\r\n";
    response += "Content-Type: " + contentType + "\r\n";
    response += "Content-Length: " + std::to_string(body.size()) + "\r\n";
    response += "Connection: close\r\n";
    response += "\r\n";
    response += body;
    return response;
}
```

### ทำไม `Content-Length` ถึงสำคัญมากจนต้องเน้นเป็นพิเศษ

จำได้จาก Part 33.8 ไหมว่า TCP เป็น **Byte Stream** ที่ไม่มีขอบเขตข้อความในตัว Protocol เอง?
ปัญหานี้กลับมาอีกครั้งใน HTTP: เมื่อ Server ส่ง Body กลับไป **Client ไม่มีทางรู้เองว่า Body
จบตรงไหน** ถ้าไม่มีการบอกไว้อย่างชัดเจน มี 2 วิธีหลักที่ HTTP ใช้บอกขอบเขตของ Body:

1. **`Content-Length: N`** — บอกจำนวน byte ของ Body ล่วงหน้าแบบตรงๆ (วิธีที่ Server ของเรา
   ใช้ เพราะเรารู้ขนาด Body ทั้งหมดตั้งแต่ก่อนส่งอยู่แล้ว)
2. **`Transfer-Encoding: chunked`** — ส่ง Body เป็นก้อนๆ (chunk) ที่แต่ละก้อนบอกขนาดตัวเอง
   ใช้เมื่อไม่รู้ขนาด Body ทั้งหมดล่วงหน้า (เช่น Streaming ข้อมูล) — Part นี้จะไม่ใช้วิธีนี้
   เพราะซับซ้อนเกินความจำเป็นสำหรับ Server เปล่าๆ ของเรา

ถ้า **ไม่มีทั้งสองอย่าง** และ Connection ไม่ได้ปิดทันที Client (โดยเฉพาะ Browser) จะ**รอข้อมูล
เพิ่มเติมต่อไปเรื่อยๆ** เพราะคิดว่า Body ยังส่งไม่ครบ — นี่คือบั๊กที่จะสาธิตให้เห็นจริงในหัวข้อ
100.7

`Connection: close` ที่ใส่เพิ่มเข้าไปด้วยเป็นการบอก Client อย่างชัดเจนว่า **หลังจากนี้ Server
จะปิด Connection ทันที** ซึ่งเป็นอีกวิธีหนึ่งที่ช่วยยืนยันขอบเขตของ Response (ถ้า Connection
ปิดไปแล้ว ก็ไม่มีข้อมูลอะไรมาเพิ่มได้อีก) — Server เวอร์ชันนี้เลือกปิด Connection ทุกครั้งหลัง
ตอบ 1 Request เพื่อความง่าย (HTTP/1.1 จริงๆ รองรับ **Persistent Connection** ที่คุยกันหลาย
Request ต่อ 1 Connection ได้ แต่นั่นเพิ่มความซับซ้อนที่ยังไม่จำเป็นสำหรับ Part นี้)

---

## 100.4 Routing ง่ายๆ ตาม Path String (Step 796)

**Router** คือส่วนที่ตัดสินใจว่า Path ไหนควรได้ Response แบบไหน สำหรับ Server ระดับนี้ การ
Routing ทำได้ง่ายๆ ด้วยการเทียบ `if`/`else` ตรงๆ กับค่า `req.path`:

```cpp
// Router ง่ายๆ: ตัดสินใจจาก path ว่าจะตอบอะไรกลับไป
std::string handleRequest(const HttpRequest& req) {
    if (!req.valid) {
        std::string body = "<h1>400 Bad Request</h1>";
        return buildResponse(400, "Bad Request", "text/html", body);
    }

    if (req.method != "GET") {
        std::string body = "<h1>405 Method Not Allowed</h1>";
        return buildResponse(405, "Method Not Allowed", "text/html", body);
    }

    if (req.path == "/") {
        std::string body =
            "<html><body><h1>Raw HTTP Server</h1>"
            "<p>สร้างจาก socket API ล้วนๆ ไม่ใช้ library ใดๆ</p></body></html>";
        return buildResponse(200, "OK", "text/html", body);
    }

    if (req.path == "/about") {
        std::string body = "This is a raw HTTP server built from scratch using socket().";
        return buildResponse(200, "OK", "text/plain", body);
    }

    if (req.path == "/api/time") {
        std::time_t now = std::time(nullptr);
        std::string body = std::string("Server time (epoch): ") + std::to_string(now);
        return buildResponse(200, "OK", "text/plain", body);
    }

    std::string body = "<h1>404 Not Found</h1><p>ไม่พบ path: " + req.path + "</p>";
    return buildResponse(404, "Not Found", "text/html", body);
}
```

สังเกตลำดับความสำคัญของการตรวจสอบ: **Bad Request (400) ก่อน แล้วค่อย Method (405) แล้วค่อย
Routing ตาม Path** — นี่คือรูปแบบที่ Framework จริงอย่าง Crow ใช้เหมือนกัน เพียงแต่ Crow มี
Syntax สะดวกกว่า (`CROW_ROUTE(app, "/about")` แทนที่จะเขียน `if` เทียบ string เอง) แต่กลไก
ภายในก็คือการเทียบ Path ตรงๆ แบบนี้เช่นกัน (บวกกับความสามารถจับ Parameter ใน Path ที่ซับซ้อน
ขึ้น ซึ่งจะเรียนใน Part 102-103)

---

## 100.5 ประกอบร่างเป็น Raw HTTP Server ตัวเต็ม (Step 797)

นำทุกส่วนที่เขียนมาประกอบเข้ากับ Socket Server แบบ `accept()` loop เดียวที่เรียนจาก Part 33
(รับ Client ได้ทีละ 1 ราย จบแล้วรับรายถัดไป):

```cpp
// raw_http_server.cpp - HTTP Server เปล่าๆ จาก raw socket (ไม่ใช้ library ใดๆ)
#include <arpa/inet.h>
#include <cstdio>
#include <cstdlib>
#include <cstring>
#include <ctime>
#include <netinet/in.h>
#include <string>
#include <sys/socket.h>
#include <unistd.h>

#define PORT 8100
#define BACKLOG 10
#define BUFFER_SIZE 4096

struct HttpRequest {
    std::string method;
    std::string path;
    std::string version;
    bool valid = false;
};

// แยก request line แบบง่ายๆ: "GET /path HTTP/1.1\r\n"
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

// สร้าง HTTP response ที่ถูกต้องตาม spec: status line + header + body + Content-Length
std::string buildResponse(int statusCode, const std::string& statusText,
                           const std::string& contentType, const std::string& body) {
    std::string response;
    response += "HTTP/1.1 " + std::to_string(statusCode) + " " + statusText + "\r\n";
    response += "Content-Type: " + contentType + "\r\n";
    response += "Content-Length: " + std::to_string(body.size()) + "\r\n";
    response += "Connection: close\r\n";
    response += "\r\n";
    response += body;
    return response;
}

// Router ง่ายๆ: ตัดสินใจจาก path ว่าจะตอบอะไรกลับไป
std::string handleRequest(const HttpRequest& req) {
    if (!req.valid) {
        std::string body = "<h1>400 Bad Request</h1>";
        return buildResponse(400, "Bad Request", "text/html", body);
    }

    if (req.method != "GET") {
        std::string body = "<h1>405 Method Not Allowed</h1>";
        return buildResponse(405, "Method Not Allowed", "text/html", body);
    }

    if (req.path == "/") {
        std::string body =
            "<html><body><h1>Raw HTTP Server</h1>"
            "<p>สร้างจาก socket API ล้วนๆ ไม่ใช้ library ใดๆ</p></body></html>";
        return buildResponse(200, "OK", "text/html", body);
    }

    if (req.path == "/about") {
        std::string body = "This is a raw HTTP server built from scratch using socket().";
        return buildResponse(200, "OK", "text/plain", body);
    }

    if (req.path == "/api/time") {
        std::time_t now = std::time(nullptr);
        std::string body = std::string("Server time (epoch): ") + std::to_string(now);
        return buildResponse(200, "OK", "text/plain", body);
    }

    std::string body = "<h1>404 Not Found</h1><p>ไม่พบ path: " + req.path + "</p>";
    return buildResponse(404, "Not Found", "text/html", body);
}

int main() {
    int serverFd = socket(AF_INET, SOCK_STREAM, 0);
    if (serverFd < 0) {
        perror("socket");
        return EXIT_FAILURE;
    }

    int opt = 1;
    if (setsockopt(serverFd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt)) < 0) {
        perror("setsockopt");
        close(serverFd);
        return EXIT_FAILURE;
    }

    struct sockaddr_in serverAddr;
    std::memset(&serverAddr, 0, sizeof(serverAddr));
    serverAddr.sin_family = AF_INET;
    serverAddr.sin_addr.s_addr = INADDR_ANY;
    serverAddr.sin_port = htons(PORT);

    if (bind(serverFd, reinterpret_cast<struct sockaddr*>(&serverAddr), sizeof(serverAddr)) < 0) {
        perror("bind");
        close(serverFd);
        return EXIT_FAILURE;
    }

    if (listen(serverFd, BACKLOG) < 0) {
        perror("listen");
        close(serverFd);
        return EXIT_FAILURE;
    }

    std::printf("[raw-http-server] กำลังฟังที่ port %d ...\n", PORT);

    for (;;) {
        struct sockaddr_in clientAddr;
        socklen_t clientLen = sizeof(clientAddr);
        int clientFd = accept(serverFd, reinterpret_cast<struct sockaddr*>(&clientAddr), &clientLen);
        if (clientFd < 0) {
            perror("accept");
            continue;
        }

        char ipStr[INET_ADDRSTRLEN];
        inet_ntop(AF_INET, &clientAddr.sin_addr, ipStr, sizeof(ipStr));

        char buffer[BUFFER_SIZE];
        ssize_t bytesRead = recv(clientFd, buffer, sizeof(buffer) - 1, 0);
        if (bytesRead <= 0) {
            close(clientFd);
            continue;
        }
        buffer[bytesRead] = '\0';

        HttpRequest req = parseRequestLine(std::string(buffer, static_cast<std::size_t>(bytesRead)));
        std::printf("[raw-http-server] %s:%d -> %s %s\n", ipStr, ntohs(clientAddr.sin_port),
                    req.method.c_str(), req.path.c_str());

        std::string response = handleRequest(req);

        std::size_t totalSent = 0;
        while (totalSent < response.size()) {
            ssize_t sent = send(clientFd, response.data() + totalSent, response.size() - totalSent, 0);
            if (sent < 0) {
                perror("send");
                break;
            }
            totalSent += static_cast<std::size_t>(sent);
        }

        close(clientFd);
    }

    close(serverFd);
    return 0;
}
```

สังเกตว่าโครงสร้างของ `main()` แทบจะเหมือนกับ `tcp_server_loop.c` (เฉลยข้อ 2 ของ Part 33)
ทุกประการ — ต่างกันแค่ 2 จุด: **Parse ข้อมูลที่ได้เป็น HTTP Request** แทนที่จะพิมพ์ตรงๆ และ
**ใช้ `handleRequest()`/`buildResponse()` สร้างข้อมูลตอบกลับที่ถูก Format ตาม HTTP Spec**
แทนที่จะ Echo ข้อมูลเดิมกลับไปตรงๆ

คอมไพล์ด้วย Flag มาตรฐานของหลักสูตร:

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 raw_http_server.cpp -o raw_http_server -pthread
```

คอมไพล์ผ่านโดย**ไม่มี Warning ใดๆ เลย** (แม้ยังไม่ได้ใช้ Multithreading ใน Part นี้ แต่ใส่
`-pthread` ไว้ตั้งแต่ตอนนี้เพื่อความเคยชิน เพราะ Part 101 จะต้องใช้ทันที)

---

## 100.6 ทดสอบด้วย curl และ Browser จริง (Step 798)

รัน Server ค้างไว้ใน Terminal หนึ่ง:

```bash
./raw_http_server
```

```
[raw-http-server] กำลังฟังที่ port 8100 ...
```

เปิด Terminal อีกหน้าต่างมาทดสอบด้วย `curl -i` (Flag `-i` ให้แสดง HTTP Header ที่ได้กลับมา
ด้วย ไม่ใช่แค่ Body):

```bash
curl -s -i http://127.0.0.1:8100/
```

ผลลัพธ์จริงที่ได้จากการทดสอบ:

```
HTTP/1.1 200 OK
Content-Type: text/html
Content-Length: 145
Connection: close

<html><body><h1>Raw HTTP Server</h1><p>สร้างจาก socket API ล้วนๆ ไม่ใช้ library ใดๆ</p></body></html>
```

ทดสอบ Route อื่นๆ ที่เขียนไว้:

```bash
curl -s -i http://127.0.0.1:8100/about
curl -s -i http://127.0.0.1:8100/api/time
curl -s -i http://127.0.0.1:8100/nope
curl -s -i -X POST http://127.0.0.1:8100/
```

ผลลัพธ์จริงตามลำดับ:

```
HTTP/1.1 200 OK
Content-Type: text/plain
Content-Length: 60
Connection: close

This is a raw HTTP server built from scratch using socket().
HTTP/1.1 200 OK
Content-Type: text/plain
Content-Length: 31
Connection: close

Server time (epoch): 1790413895
HTTP/1.1 404 Not Found
Content-Type: text/html
Content-Length: 56
Connection: close

<h1>404 Not Found</h1><p>ไม่พบ path: /nope</p>
HTTP/1.1 405 Method Not Allowed
Content-Type: text/html
Content-Length: 31
Connection: close

<h1>405 Method Not Allowed</h1>
```

ฝั่ง Server (Terminal ที่รันค้างไว้) จะพิมพ์ Log ของทุก Request ที่เข้ามาตามลำดับ:

```
[raw-http-server] กำลังฟังที่ port 8100 ...
[raw-http-server] 127.0.0.1:44720 -> GET /
[raw-http-server] 127.0.0.1:44728 -> GET /about
[raw-http-server] 127.0.0.1:44738 -> GET /api/time
[raw-http-server] 127.0.0.1:44742 -> GET /nope
[raw-http-server] 127.0.0.1:44752 -> POST /
```

สังเกตว่า **Port ของ Client เปลี่ยนไปทุกครั้ง** (`44720`, `44728`, `44738`, ...) แม้จะยิง
Request จากเครื่องเดียวกันไปยัง Server เดียวกัน — นี่คือ Ephemeral Port ที่ OS สุ่มให้ทุกครั้ง
ที่เปิด Connection ใหม่ ตามที่เรียนไปแล้วใน Part 33 (เพราะ `curl` แต่ละคำสั่งเปิด Connection
TCP ใหม่ทั้งหมด ไม่ได้ใช้ Connection เดิมซ้ำ สอดคล้องกับ `Connection: close` ที่ Server ตอบไป)

### ทดสอบด้วย Browser จริง

เปิด Web Browser (Chrome, Firefox, หรือตัวใดก็ได้) แล้วพิมพ์ `http://127.0.0.1:8100/` ใน
Address Bar จะเห็นหน้าเว็บ **"Raw HTTP Server"** แสดงผลจริงตามที่ HTML ใน `body` กำหนดไว้ —
นี่คือหลักฐานที่ชัดเจนที่สุดว่า Browser ไม่ได้สนใจว่า Server เขียนด้วยภาษาอะไรหรือใช้ Library
ตัวไหน สนใจแค่ว่า **Byte Stream ที่ได้กลับมาตรงตาม HTTP Spec หรือไม่** ตราบใดที่ Format ถูก
ต้อง Browser ก็แสดงผลได้เหมือนกับที่แสดงผลเว็บไซต์จริงทุกประการ

---

## 100.7 บั๊กร้ายแรง: ลืมใส่ Content-Length (Step 799)

มาถึงจุดสำคัญที่สุดของ Part นี้: การสาธิตให้เห็นจริงว่าเกิดอะไรขึ้นถ้าลืม `Content-Length`
ลองเขียน Server จำลองที่ **จงใจลืม** Header นี้ (และไม่ใส่ `Connection: close` ด้วย):

```cpp
// buggy_no_content_length.cpp - สาธิตบั๊ก: ลืมใส่ Content-Length ทำให้ client รอไม่จบ
#include <arpa/inet.h>
#include <cstdio>
#include <cstring>
#include <netinet/in.h>
#include <string>
#include <sys/socket.h>
#include <unistd.h>

#define PORT 8111

int main() {
    int serverFd = socket(AF_INET, SOCK_STREAM, 0);
    int opt = 1;
    setsockopt(serverFd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));

    struct sockaddr_in addr;
    std::memset(&addr, 0, sizeof(addr));
    addr.sin_family = AF_INET;
    addr.sin_addr.s_addr = INADDR_ANY;
    addr.sin_port = htons(PORT);
    bind(serverFd, reinterpret_cast<struct sockaddr*>(&addr), sizeof(addr));
    listen(serverFd, 5);
    std::printf("[buggy-server] listening on %d (ไม่ใส่ Content-Length โดยตั้งใจ)\n", PORT);

    struct sockaddr_in clientAddr;
    socklen_t clientLen = sizeof(clientAddr);
    int clientFd = accept(serverFd, reinterpret_cast<struct sockaddr*>(&clientAddr), &clientLen);

    char buffer[4096];
    recv(clientFd, buffer, sizeof(buffer) - 1, 0);   // อ่านทิ้ง ไม่สนใจเนื้อหา request

    // บั๊ก: ไม่ใส่ Content-Length และไม่ใส่ Connection: close -> client ไม่รู้ว่าเมื่อไหร่ body จบ
    std::string response =
        "HTTP/1.1 200 OK\r\n"
        "Content-Type: text/plain\r\n"
        "\r\n"
        "Hello without Content-Length";
    send(clientFd, response.data(), response.size(), 0);

    // จงใจไม่ close(clientFd) ทันที เพื่อจำลองพฤติกรรมที่ client มองไม่เห็นจุดจบของ body
    sleep(3);
    close(clientFd);
    close(serverFd);
    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 buggy_no_content_length.cpp -o buggy_no_content_length
./buggy_no_content_length &
curl -s -i --max-time 2 http://127.0.0.1:8111/
```

ผลลัพธ์จริงจากการทดสอบ (ใส่ `--max-time 2` ป้องกันไม่ให้ `curl` ค้างตลอดไประหว่างทดสอบ เพราะ
โดยปกติถ้าไม่ใส่ Flag นี้ `curl` จะรอไปเรื่อยๆ ไม่มีกำหนด):

```
HTTP/1.1 200 OK
Content-Type: text/plain

Hello without Content-Length

real	0m2.010s
user	0m0.008s
sys	0m0.000s
curl exit code: 28
```

### วิเคราะห์สิ่งที่เกิดขึ้น

สังเกตว่า `curl` **ได้รับ Body ข้อความ "Hello without Content-Length" ครบถูกต้องแล้ว** (แสดง
ผลออกมาให้เห็นในผลลัพธ์ด้านบน) แต่ยัง**ไม่ยอมจบการทำงาน** — มันรอต่อไปอีกจนครบเวลา 2 วินาที
ที่กำหนดไว้ใน `--max-time` แล้วจบด้วย **Exit Code 28** ซึ่งเป็นรหัส Error มาตรฐานของ `curl`
ที่แปลว่า **"Operation timeout"**

เหตุผลคือ HTTP/1.1 **สันนิษฐานว่า Connection เป็นแบบ Persistent (เปิดค้างไว้ใช้ซ้ำได้) โดย
Default** เว้นแต่จะมีการบอกไว้ชัดเจนว่าจะปิด (`Connection: close`) หรือบอกขอบเขตของ Body ไว้
ชัดเจน (`Content-Length`) เมื่อ Server ของเราไม่ใส่ทั้งสองอย่าง `curl` จึงตีความว่า **"บางที
Server อาจจะส่งข้อมูลเพิ่มมาอีกในภายหลังผ่าน Connection เดิมนี้"** จึงรอต่อไปเรื่อยๆ โดยไม่รู้
ว่า Body ที่ได้มาแล้วนั้นคือทั้งหมดที่จะได้รับ

ถ้าเป็น Web Browser จริง (ไม่ใช่ `curl` ที่ทดสอบ) ผู้ใช้จะเห็นวงล้อหมุน (Loading Spinner) ค้าง
อยู่ตลอดไป หรือหน้าเว็บโหลดไม่เสร็จสักที แม้ข้อมูลทั้งหมดจะถึงเครื่องผู้ใช้แล้วจริงๆ ก็ตาม —
นี่คือเหตุผลที่ `Content-Length` (หรือ `Connection: close` เป็นทางเลือกสำรอง) **ไม่ใช่รายละเอียด
เล็กน้อยที่มองข้ามได้** แต่เป็นส่วนที่ขาดไม่ได้เลยของ HTTP Response ที่ถูกต้อง

---

## 100.8 ข้อจำกัดของ Raw Implementation นี้ (Step 800)

Server ที่สร้างเสร็จใน Part นี้ทำงานได้ถูกต้องตาม HTTP Spec ทุกประการ แต่ยังมีข้อจำกัดใหญ่
ที่สืบทอดมาจากโครงสร้าง `accept()` loop แบบง่ายที่สุดของ Part 33: **รับได้ทีละ 1 Connection
เท่านั้น** ถ้ามี Client รายที่สองพยายามเชื่อมต่อเข้ามาระหว่างที่ Server กำลังจัดการ Client
รายแรกอยู่ รายที่สองต้องรอในคิว (Backlog) จนกว่า Server จะว่าง

### สาธิตปัญหาให้เห็นจริง

ลองจำลอง Client ที่ "ช้า" (เปิด Connection ค้างไว้โดยไม่ส่งข้อมูลอะไรเลยเป็นเวลา 3 วินาที)
แล้ววัดเวลาที่ Client รายที่สองต้องรอกว่าจะได้รับ Response:

```bash
./raw_http_server &

# client 1: เปิด connection ค้างไว้โดยไม่ส่งอะไรเลย เป็นเวลา 3 วินาที (จำลอง client ช้า)
( exec 3<>/dev/tcp/127.0.0.1/8100; sleep 3; exec 3<&- ) &
sleep 0.3

time curl -s -o /dev/null -w "HTTP %{http_code}, ใช้เวลา %{time_total}s\n" http://127.0.0.1:8100/
```

ผลลัพธ์จริงจากการทดสอบ:

```
HTTP 200, ใช้เวลา 2.690013s

real	0m2.702s
user	0m0.002s
sys	0m0.012s
```

### วิเคราะห์ผลลัพธ์

Client รายที่สอง (คำสั่ง `curl` ปกติ) ควรจะได้รับ Response แทบจะทันที (ในหลัก Millisecond)
เพราะเป็นแค่การขอหน้าเว็บง่ายๆ แต่กลับต้อง**รอถึง 2.69 วินาที** — เหตุผลคือ:

```
เวลา 0.0s:  Client 1 (ช้า) เชื่อมต่อเข้ามา -> Server accept() สำเร็จ -> เรียก recv() รอข้อมูล
เวลา 0.3s:  Client 2 (curl) พยายามเชื่อมต่อ -> TCP handshake สำเร็จ (เข้าคิว backlog)
            แต่ Server ยังไม่เรียก accept() รอบใหม่ เพราะติดอยู่ที่ recv() ของ Client 1
เวลา 3.0s:  Client 1 ปิด connection (ครบ sleep 3 วินาที) -> recv() คืนค่า 0 -> Server ปิด
            client_fd แล้ววน loop กลับไป accept() รอบใหม่ -> ได้ Client 2 จากคิว -> ตอบกลับ
เวลา ~3.0s: Client 2 ได้รับ Response ในที่สุด (จริงๆ ตัวเลขที่วัดได้คือ 2.69s เพราะนับจากตอนที่
            curl เริ่มทำงาน ซึ่งช้ากว่า client 1 เริ่มไปแล้วประมาณ 0.3 วินาที)
```

Client รายที่สอง**เชื่อมต่อ TCP สำเร็จทันที** (เพราะ `listen()` มี Backlog Queue รองรับ) แต่
**ต้องรอ HTTP Response** จนกว่า Server จะว่างจาก Client รายแรก — นี่คือข้อจำกัดที่ร้ายแรงมาก
สำหรับ Web Server ในโลกจริง เพราะ Client ที่ช้าเพียงรายเดียว (เช่น Connection ที่มีปัญหา
เครือข่าย หรือแม้แต่ Client ที่จงใจโจมตีแบบ **Slowloris**) สามารถทำให้ Client รายอื่นทั้งหมด
ต้องรอได้

```
┌─────────────────────────────────────────────────────────────┐
│  Server (accept() loop เดียว, Single-threaded)                │
│                                                                │
│  Client 1 (ช้า) ──▷ กำลังถูกจัดการอยู่ (recv() ค้างรอ)          │
│                                                                │
│  Client 2 ──▷ [รอในคิว Backlog] ──▷ ต้องรอ Client 1 จบก่อน      │
│  Client 3 ──▷ [รอในคิว Backlog] ──▷ ต้องรอ Client 1 และ 2 จบก่อน │
│  Client 4 ──▷ [รอในคิว Backlog] ──▷ ...                        │
└─────────────────────────────────────────────────────────────┘
```

การแก้ปัญหานี้มี 2 แนวทางหลักตามที่ปูพื้นไว้ใน Part 34: **I/O Multiplexing** (`select`/`poll`/
`epoll` — เหมาะกับ Connection จำนวนมากที่ส่วนใหญ่ไม่ได้ใช้ CPU หนัก) และ **Multithreading**
(สร้าง Thread แยกจัดการแต่ละ Connection — เหมาะกับงานที่ต้องประมวลผลหนักต่อ Request) **Part
101** จะเลือกแนวทางที่สอง เพราะต่อยอดจาก `std::thread` (Part 81) และ `std::mutex`/
`std::condition_variable` (Part 82) ที่เรียนมาแล้วโดยตรง และเป็นรากฐานสำคัญก่อนจะไปเรียน
Framework ที่ใช้ Multithreading ผสมกับ I/O Multiplexing ในภายหลัง

---

## ข้อผิดพลาดที่พบบ่อย (Common Pitfalls)

1. **ลืมใส่ `Content-Length` หรือคำนวณผิด** — ตามที่สาธิตในหัวข้อ 100.7 ทำให้ Client/Browser
   ค้างรอข้อมูลที่ไม่มีวันมาถึง ต้องคำนวณจากขนาด **Body จริง** (`body.size()`) เสมอ ไม่ใช่
   ขนาดของ string ทั้งก้อนที่รวม Header เข้าไปด้วย
2. **ลืมบรรทัดว่าง (`\r\n\r\n`) คั่นระหว่าง Header กับ Body** — ถ้าขาดบรรทัดว่างนี้ไป Client
   จะตีความ Body บรรทัดแรกเป็น Header เพิ่มเติมแทน ทำให้ Parse ผิดพลาดทั้งหมด
3. **ใช้ `\n` เดี่ยวๆ แทน `\r\n`** — HTTP Spec กำหนดให้ใช้ **CRLF (`\r\n`)** เป็นตัวคั่นบรรทัด
   เสมอ Client บางตัว (โดยเฉพาะ Browser บางรุ่นหรือ Library HTTP บางตัว) อาจ Parse ผิดพลาด
   ถ้าใช้แค่ `\n` แม้จะดูเหมือนทำงานได้ปกติกับ `curl` บางเวอร์ชันที่ผ่อนปรนกว่า Spec
4. **ไม่ตรวจสอบว่า `recv()` คืนค่า `<= 0`** — ถ้า Client ปิด Connection ก่อนส่งข้อมูลมาครบ
   (หรือส่งมาแค่บางส่วน) แล้วโค้ดพยายาม Parse Request ที่ไม่สมบูรณ์อยู่ดี อาจทำให้เกิด
   พฤติกรรมที่คาดเดาไม่ได้ ต้องเช็คก่อนเสมอแล้ว `continue`/`close` ทันทีถ้าค่าที่ได้ไม่ถูกต้อง
5. **ลืมว่า `send()` อาจส่งได้ไม่ครบในครั้งเดียว (Partial Send)** — เหมือนที่เรียนใน Part 33.8
   ต้อง loop ส่งจนกว่าจะครบเสมอ โดยเฉพาะเมื่อ Response มีขนาดใหญ่ (เช่น ไฟล์ HTML ที่มีรูปภาพ
   Base64 ฝังอยู่) ยิ่งมีโอกาสเกิด Partial Send สูงขึ้น
6. **มองข้ามข้อจำกัดเรื่องรับได้ทีละ 1 Connection** — คิดว่า Server ทำงาน "เร็วพอ" แล้วไม่มี
   ปัญหา ทั้งที่ในโลกจริงมี Client จำนวนมากพร้อมกันเสมอ แม้แต่ Client รายเดียวที่ช้าผิดปกติ
   (Network มีปัญหา หรือถูกโจมตีแบบ Slowloris) ก็ทำให้ Client รายอื่นทั้งหมดต้องรอได้ตามที่
   สาธิตในหัวข้อ 100.8
7. **ไม่ปิด Header ด้วยการระบุ `Content-Type` ที่ถูกต้อง** — เช่น ส่ง HTML กลับไปแต่ระบุ
   `Content-Type: text/plain` ทำให้ Browser แสดงผลเป็นข้อความดิบ (แสดง Tag `<html>` ให้เห็น
   ตรงๆ) แทนที่จะ Render เป็นหน้าเว็บ

---

## แบบฝึกหัดท้ายบท

1. เพิ่ม Route ใหม่ `/api/echo` ที่รับ Query String ต่อท้าย Path (เช่น `/api/echo?msg=hello`)
   แล้วส่งค่า Query String นั้นกลับไปเป็น Body ของ Response (ต้องแยก Path จริงออกจาก Query
   String ก่อน เพราะตอนนี้ Router เทียบ `req.path` แบบตรงเป๊ะทั้งก้อน)
2. แก้ไข Server ให้รองรับ HTTP Method `HEAD` (เหมือน `GET` ทุกประการแต่**ไม่ส่ง Body กลับ**
   ส่งแค่ Status Line กับ Header เท่านั้น รวมถึง `Content-Length` ที่ต้องเป็นขนาดของ Body
   ที่ **จะ** ส่งถ้าเป็น `GET` แม้จะไม่ได้ส่งจริงก็ตาม — นี่คือพฤติกรรมมาตรฐานของ HTTP Spec)
3. รัน Server เวอร์ชันในหัวข้อ 100.7 (ที่ลืม `Content-Length`) แล้วลองสังเกตพฤติกรรมผ่าน
   Browser จริง (ไม่ใช่ `curl`) ว่าเกิดอะไรขึ้นกับหน้าเว็บที่เปิดค้างไว้
4. เพิ่ม Route `/api/headers` ที่ตอบกลับ Path และ Method ของ Request ปัจจุบันในรูปแบบ JSON
   ง่ายๆ (เช่น `{"method": "GET", "path": "/api/headers"}`) โดยยังไม่ต้องใช้ JSON Library
   ใดๆ (แค่ต่อ string เอง ก็เพียงพอสำหรับแบบฝึกหัดนี้)
5. แก้ไข `buildResponse()` ให้รองรับการส่ง Custom Header เพิ่มเติมได้ (เช่น
   `Cache-Control: no-cache`) โดยไม่ต้องแก้ Signature ของฟังก์ชันเดิมทั้งหมด (ใบ้: ใช้
   `std::vector<std::pair<std::string, std::string>>` เป็นพารามิเตอร์เสริม)
6. เขียนโปรแกรมทดสอบ (Client) ของตัวเองที่เชื่อมต่อไปยัง Server แล้ว **จงใจส่ง Request ที่
   ผิดรูปแบบ** (เช่น ส่งแค่ `"GARBAGE DATA"` โดยไม่มี `\r\n` เลย) แล้วตรวจสอบว่า Server ตอบ
   กลับด้วย `400 Bad Request` ตามที่ออกแบบไว้จริงหรือไม่

### แนวทางเฉลยข้อ 1

```cpp
// raw_http_server_ex1.cpp - เฉลยข้อ 1: เพิ่ม route /api/echo?msg=... ที่คืนค่า query string กลับไป
#include <arpa/inet.h>
#include <cstdio>
#include <cstdlib>
#include <cstring>
#include <netinet/in.h>
#include <string>
#include <sys/socket.h>
#include <unistd.h>

#define PORT 8110
#define BACKLOG 10
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

// ดึงค่า query string ส่วนที่อยู่หลัง '?' ออกจาก path (แยกจาก path จริงที่ใช้ routing)
std::string extractQuery(const std::string& fullPath, std::string& pathOut) {
    std::size_t qPos = fullPath.find('?');
    if (qPos == std::string::npos) {
        pathOut = fullPath;
        return "";
    }
    pathOut = fullPath.substr(0, qPos);
    return fullPath.substr(qPos + 1);
}

std::string handleRequest(const HttpRequest& req) {
    if (!req.valid) return buildResponse(400, "Bad Request", "text/html", "<h1>400</h1>");

    std::string cleanPath;
    std::string query = extractQuery(req.path, cleanPath);

    if (cleanPath == "/api/echo") {
        std::string body = "you sent query: " + (query.empty() ? "(none)" : query);
        return buildResponse(200, "OK", "text/plain", body);
    }

    if (cleanPath == "/") {
        return buildResponse(200, "OK", "text/html", "<h1>OK</h1>");
    }

    return buildResponse(404, "Not Found", "text/html", "<h1>404</h1>");
}

int main() {
    int serverFd = socket(AF_INET, SOCK_STREAM, 0);
    int opt = 1;
    setsockopt(serverFd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));

    struct sockaddr_in serverAddr;
    std::memset(&serverAddr, 0, sizeof(serverAddr));
    serverAddr.sin_family = AF_INET;
    serverAddr.sin_addr.s_addr = INADDR_ANY;
    serverAddr.sin_port = htons(PORT);
    bind(serverFd, reinterpret_cast<struct sockaddr*>(&serverAddr), sizeof(serverAddr));
    listen(serverFd, BACKLOG);
    std::printf("[echo-server] listening on %d\n", PORT);

    for (;;) {
        struct sockaddr_in clientAddr;
        socklen_t clientLen = sizeof(clientAddr);
        int clientFd = accept(serverFd, reinterpret_cast<struct sockaddr*>(&clientAddr), &clientLen);
        if (clientFd < 0) continue;

        char buffer[BUFFER_SIZE];
        ssize_t n = recv(clientFd, buffer, sizeof(buffer) - 1, 0);
        if (n <= 0) { close(clientFd); continue; }
        buffer[n] = '\0';

        HttpRequest req = parseRequestLine(std::string(buffer, static_cast<std::size_t>(n)));
        std::string response = handleRequest(req);

        std::size_t sentTotal = 0;
        while (sentTotal < response.size()) {
            ssize_t sent = send(clientFd, response.data() + sentTotal, response.size() - sentTotal, 0);
            if (sent < 0) break;
            sentTotal += static_cast<std::size_t>(sent);
        }
        close(clientFd);
    }
    close(serverFd);
    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 raw_http_server_ex1.cpp -o raw_http_server_ex1
./raw_http_server_ex1 &
curl -s -i "http://127.0.0.1:8110/api/echo?msg=hello&name=somchai"
```

ผลลัพธ์จริงจากการทดสอบ:

```
HTTP/1.1 200 OK
Content-Type: text/plain
Content-Length: 38
Connection: close

you sent query: msg=hello&name=somchai
```

### แนวทางเฉลยข้อ 6

```cpp
// bad_request_client.cpp - เฉลยข้อ 6: client ที่จงใจส่งข้อมูลผิดรูปแบบไปยัง server
#include <arpa/inet.h>
#include <cstdio>
#include <cstring>
#include <netinet/in.h>
#include <string>
#include <sys/socket.h>
#include <unistd.h>

int main() {
    int sockFd = socket(AF_INET, SOCK_STREAM, 0);

    struct sockaddr_in serverAddr;
    std::memset(&serverAddr, 0, sizeof(serverAddr));
    serverAddr.sin_family = AF_INET;
    serverAddr.sin_port = htons(8100);
    inet_pton(AF_INET, "127.0.0.1", &serverAddr.sin_addr);

    if (connect(sockFd, reinterpret_cast<struct sockaddr*>(&serverAddr), sizeof(serverAddr)) < 0) {
        perror("connect");
        return 1;
    }

    // จงใจส่งข้อมูลที่ไม่มี \r\n เลย -> parseRequestLine() ของ server จะหา "\r\n" ไม่เจอ
    // -> req.valid = false -> server ควรตอบกลับด้วย 400 Bad Request
    const char* garbage = "GARBAGE DATA WITHOUT PROPER HTTP FORMAT";
    send(sockFd, garbage, std::strlen(garbage), 0);

    char buffer[4096];
    ssize_t n = recv(sockFd, buffer, sizeof(buffer) - 1, 0);
    if (n > 0) {
        buffer[n] = '\0';
        std::printf("ได้รับกลับมา:\n%s\n", buffer);
    }

    close(sockFd);
    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 bad_request_client.cpp -o bad_request_client
./raw_http_server &
./bad_request_client
```

ผลลัพธ์ที่คาดหวัง (ตรงตามที่ออกแบบไว้ใน `handleRequest()`):

```
ได้รับกลับมา:
HTTP/1.1 400 Bad Request
Content-Type: text/html
Content-Length: 24
Connection: close

<h1>400 Bad Request</h1>
```

(ข้อ 2, 3, 4 และ 5 ให้ผู้เรียนลองทำเองตามแนวทางของตัวอย่างในบทเรียน — สิ่งสำคัญคือทุกคำตอบ
ต้องคอมไพล์ผ่านด้วย `-Wall -Wextra -Wpedantic -std=c++17` โดยไม่มี Warning ใดๆ เลย และต้อง
ทดสอบด้วย `curl` หรือ Browser จริงก่อนถือว่าทำเสร็จ)

---

## สรุปท้ายบท

ใน Part นี้เราได้:

- สร้าง HTTP Server เปล่าๆ จาก Socket API ล้วนๆ โดยไม่ใช้ Library ใดๆ ต่อยอดจาก
  `socket()`/`bind()`/`listen()`/`accept()` ที่เรียนไปแล้วใน Part 33
- Parse HTTP Request Line ด้วยมือ แยก Method, Path, และ Version ออกจาก string ดิบได้อย่าง
  ถูกต้อง
- สร้าง HTTP Response ที่ถูกต้องตาม Spec ครบทุกส่วน โดยเฉพาะ `Content-Length` ที่ต้อง
  คำนวณถูกต้องเสมอ
- เขียน Router ง่ายๆ ที่ตัดสินใจตอบกลับตาม Path และทดสอบ Server ที่สร้างเองได้จริงผ่าน
  `curl` และ Web Browser
- สาธิตให้เห็นจริงว่าการลืม `Content-Length` ทำให้ Client ค้างรอข้อมูลที่ไม่มีวันมาถึงอย่างไร
  (ทดสอบจริงจนได้ Exit Code 28 = Timeout จาก `curl`)
- เข้าใจข้อจำกัดสำคัญของ Server ที่สร้างขึ้น: รับ Client ได้ **ทีละ 1 รายเท่านั้น** และวัด
  ผลกระทบจริงจากการทดสอบ (Client รายที่สองต้องรอเกือบ 3 วินาทีเพราะ Client รายแรกช้า)

Server ของเราตอนนี้ทำงานถูกต้องตาม HTTP Spec ทุกประการ แต่ยังไม่พร้อมใช้งานจริงเลย เพราะรับ
โหลดพร้อมกันไม่ได้ ใน **Part 101** เราจะแก้ปัญหานี้ด้วย **Multithreading** เริ่มจากวิธีง่าย
ที่สุด (สร้าง Thread ใหม่ทุก Connection) แล้วค้นพบปัญหาใหม่ที่ตามมา (Thread Creation Overhead)
ก่อนจะแก้ด้วย **Thread Pool Pattern** ที่ใช้ `std::mutex` และ `std::condition_variable` จาก
Part 82 มาสร้างระบบคิวงานที่ปลอดภัยจาก Race Condition

**ต่อไป:** [Part 101 — Multi-threaded HTTP Server พร้อม Thread Pool](./part-101-multithreaded-http-server.md)
