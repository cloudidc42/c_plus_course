# Part 99: ภาพรวม Web Development ด้วย C/C++ (Step 785–792)

> Module I — Web Development ด้วย C/C++ | Part 99 จาก 125
> Step ที่ครอบคลุมใน Part นี้: Step 785–792
> Part ก่อนหน้า: [Part 98 — Design Pattern ใน C++ (ตอนที่ 2): Structural และ Behavioral Patterns](./part-098-design-patterns-2.md) | Part ถัดไป: [Part 100 — สร้าง Raw HTTP Server จาก Socket ด้วยมือ](./part-100-raw-http-server.md)

## เป้าหมายการเรียนรู้ (Learning Objectives)

หลังจบ Part นี้ ผู้เรียนจะสามารถ:

1. อธิบายได้ว่าทำไมบริษัทและระบบระดับโลกบางส่วนยังคงเลือกใช้ C/C++ พัฒนา Web Backend
   ทั้งที่ภาษาอื่นอย่าง Node.js, Python, Go ดูเขียนง่ายและพัฒนาเร็วกว่ามาก
2. เปรียบเทียบจุดแข็ง-จุดอ่อนของ C/C++ กับภาษาอื่นในบริบทของ Web Development ได้อย่างเป็นกลาง
3. อธิบายภาพรวมของ HTTP Protocol (Request/Response, Method, Header, Status Code) ได้อย่าง
   ถูกต้อง แม้ไม่เคยทำ Web Development มาก่อนเลย
4. อธิบายสถาปัตยกรรม **CGI** ได้ว่าทำงานอย่างไร และทำไมถึงยังมีการใช้งานอยู่ในบางบริบทแม้จะ
   เป็นเทคโนโลยีเก่า
5. อธิบายสถาปัตยกรรม **FastCGI** และบอกได้ว่ามันแก้ปัญหาอะไรของ CGI แบบดั้งเดิม
6. อธิบายแนวทาง **Embedded HTTP Server Library** (เช่น Crow, Pistache, Drogon) และบอกได้ว่า
   เหมาะกับสถานการณ์แบบไหน
7. อธิบายรูปแบบสถาปัตยกรรม **Reverse Proxy** (nginx อยู่หน้าบ้าน + C++ backend อยู่หลังบ้าน)
   และเหตุผลที่ระบบ Production จริงแทบทั้งหมดใช้รูปแบบนี้
8. อธิบาย Roadmap ของ Module I ทั้งหมดได้ว่าจะสร้างอะไรบ้าง ตั้งแต่ Part 99 ถึง Part 112

---

## 99.1 ทำไมยังต้องใช้ C/C++ พัฒนา Web Backend ในปี 2026 (Step 785)

เมื่อพูดถึง Web Development ในปัจจุบัน ภาษาที่คนส่วนใหญ่นึกถึงเป็นอันดับแรกมักเป็น
**JavaScript/TypeScript (Node.js)**, **Python** (Django/FastAPI), หรือ **Go** เพราะเขียนง่าย,
มี Framework ที่ใช้งานสะดวก, และพัฒนาได้เร็ว คำถามที่ผู้เรียนหลายคนสงสัยเมื่อมาถึงจุดนี้คือ:
**"เรียน C/C++ มา 98 Part แล้ว จะเอาไปทำ Web ทำไม ในเมื่อภาษาอื่นทำง่ายกว่ามาก?"**

คำตอบคือ C/C++ ไม่ได้แข่งกับภาษาเหล่านั้นในทุกสถานการณ์ แต่ครองพื้นที่เฉพาะทาง (niche) ที่
สำคัญมากในอุตสาหกรรม ซึ่งมีเหตุผลหลัก 3 ข้อ:

### เหตุผลที่ 1: Performance สูงสุดเท่าที่ Hardware จะให้ได้

C/C++ ไม่มีชั้น Abstraction คั่นระหว่างโค้ดกับ CPU มากนัก (ตามที่เรียนมาตั้งแต่ Part 1 เรื่อง
กระบวนการ Compile) ไม่มี **Garbage Collector** ที่หยุดโปรแกรมเป็นระยะเพื่อเก็บขยะ (ต่างจาก
Java, Go, JavaScript ที่ทุกตัวมี GC) และไม่มี **Interpreter Overhead** เหมือน Python ผลลัพธ์คือ
C/C++ มักเร็วกว่าภาษาอื่นหลายเท่าตัวสำหรับงานที่ต้องประมวลผลหนักๆ ต่อ Request เดียว

```
เปรียบเทียบ Latency โดยประมาณ (ตัวเลขเชิงแนวคิด ขึ้นกับ workload จริง):

Python (Django)     ████████████████████████  ~20-50ms ต่อ request (งานเบา)
Node.js              ████████████              ~5-15ms ต่อ request
Go                   ██████                    ~2-8ms ต่อ request
C++ (Crow/Drogon)     ██                        ~0.1-2ms ต่อ request
```

### เหตุผลที่ 2: ควบคุม Memory และ Latency ได้ละเอียดถึงระดับ Byte และ Microsecond

Framework ภาษาอื่นมักซ่อนรายละเอียดเรื่อง Memory Allocation ไว้เบื้องหลังเพื่อความสะดวก แต่
สำหรับระบบที่ **Latency ทุก Microsecond มีความหมายทางธุรกิจจริง** (ไม่ใช่แค่ "เร็วกว่าดีกว่า"
แต่ "ช้ากว่าคู่แข่ง 1 มิลลิวินาทีเท่ากับเสียเงินจริง") การควบคุม Memory เองแบบที่เรียนมาตลอด
Module B (Memory Management), Module E (Smart Pointer), และ Module G (Performance/Cache
Friendly) กลายเป็นข้อได้เปรียบที่ภาษาอื่นทำไม่ได้ในระดับเดียวกัน

### เหตุผลที่ 3: Use Case ที่ Throughput สูงมากๆ ในโลกจริง

| Use Case | ทำไมถึงเลือก C/C++ |
|---|---|
| **Trading System / High-Frequency Trading** | Order ต้องส่งถึงตลาดหุ้นเร็วที่สุดเท่าที่จะเป็นไปได้ ทุก Microsecond คือเงินจริง Latency ที่ไม่สม่ำเสมอจาก Garbage Collector เป็นสิ่งที่ยอมรับไม่ได้เลย |
| **Ad Server** | ต้องตัดสินใจว่าจะแสดงโฆษณาตัวไหนภายในเวลาไม่กี่ Millisecond ต่อ Request นับพันล้าน Request ต่อวัน ต้นทุน Server ต่อ Request ที่ลดลงแม้เพียงเล็กน้อยคูณด้วยปริมาณมหาศาลกลายเป็นเงินจำนวนมาก |
| **Game Backend (Real-time Multiplayer)** | ต้องประมวลผล State ของผู้เล่นหลายพันคนพร้อมกันแบบ Real-time ด้วย Latency ต่ำและสม่ำเสมอ (จำ UDP ที่เรียนใน Part 34 ได้ไหม? เกม Online ส่วนใหญ่เลือก UDP เพราะเหตุผลเดียวกันนี้) |
| **CDN / Proxy ระดับสูง (เช่น แกนหลักของ nginx)** | ต้องรองรับ Connection นับล้านพร้อมกันด้วยทรัพยากรจำกัด นี่คือเหตุผลที่ nginx (เขียนด้วย C) ยังคงเป็นมาตรฐานอุตสาหกรรมมาหลายสิบปี |
| **Database Engine (PostgreSQL, Redis, SQLite)** | Layer ที่อยู่ใต้ Web Application ต้องเร็วและเสถียรที่สุด เพราะทุกอย่างข้างบนพึ่งพา Layer นี้อยู่ |

> **ข้อควรระวัง**: การใช้ C/C++ ทำ Web Backend ไม่ได้แปลว่า "ดีกว่าเสมอ" — สำหรับ Web
> Application ทั่วไปที่ไม่มี Requirement ด้าน Performance สุดขั้ว การใช้ Python/Node.js/Go
> มักคุ้มค่ากว่ามากเพราะพัฒนาเร็วกว่า หา Developer ง่ายกว่า และ Ecosystem ของ Library
> (เช่น ORM, Admin Panel สำเร็จรูป) สมบูรณ์กว่ามาก **C/C++ เหมาะกับกรณีที่ Performance
> เป็นข้อกำหนดทางธุรกิจจริงๆ ไม่ใช่แค่ "อยากได้เร็วๆ ไว้ก่อน"**

---

## 99.2 เปรียบเทียบ C/C++ กับภาษา Web สมัยใหม่อย่างเป็นกลาง (Step 786)

| หัวข้อ | C/C++ | Node.js (JavaScript) | Python | Go |
|---|---|---|---|---|
| ความเร็วในการประมวลผล (Runtime Performance) | เร็วที่สุด | ปานกลาง-เร็ว (V8 JIT) | ช้าที่สุด (Interpreter) | เร็ว (Compiled + goroutine เบา) |
| ความเร็วในการพัฒนา (Developer Velocity) | ช้าที่สุด | เร็วมาก | เร็วมาก | เร็ว |
| Memory Safety โดย Default | ไม่มี (ต้องจัดการเอง ตามที่เรียนใน Module B/E) | มี (GC) | มี (GC) | มี (GC) |
| Ecosystem/Library สำหรับ Web (ORM, Auth, Admin) | จำกัด ต้องสร้างเองมาก | สมบูรณ์มาก (npm) | สมบูรณ์มาก (PyPI) | ค่อนข้างสมบูรณ์ |
| ความสม่ำเสมอของ Latency (Latency Jitter) | ต่ำมาก (ไม่มี GC pause) | มี GC pause เป็นระยะ | มี GC pause เป็นระยะ | มี GC pause แต่สั้นมาก |
| เหมาะกับทีมขนาดเล็ก/Startup ที่ต้องออกของเร็ว | ไม่เหมาะ | เหมาะมาก | เหมาะมาก | เหมาะ |
| เหมาะกับระบบที่ Performance คือ Requirement ทางธุรกิจ | เหมาะที่สุด | ไม่เหมาะ | ไม่เหมาะ | พอใช้ได้ |
| จำนวน Developer ในตลาดที่เขียนเป็น | น้อยกว่ามาก | เยอะมาก | เยอะมาก | ปานกลาง |

### บทเรียนสำคัญ: เลือกเครื่องมือให้เหมาะกับปัญหา ไม่ใช่เลือกเพราะ "ชอบ"

ข้อสรุปของหัวข้อนี้ไม่ใช่ "C/C++ ดีที่สุดเสมอ" แต่เป็นการฝึกทักษะที่สำคัญที่สุดอย่างหนึ่งของ
วิศวกรซอฟต์แวร์ระดับโลก: **การเลือกเทคโนโลยีให้เหมาะกับปัญหาจริง** ไม่ใช่เพราะกระแสหรือความ
คุ้นเคยส่วนตัว บริษัทขนาดใหญ่จำนวนมากใช้ทั้งสองแนวทางผสมกัน เช่น ใช้ Python/Go เขียน Web
Application ทั่วไปเป็นส่วนใหญ่ของระบบ แต่แยกเฉพาะส่วนที่ต้องการ Performance สุดขั้ว (เช่น
Matching Engine, Real-time Bidding, Media Transcoding) มาเขียนด้วย C++ ต่างหาก แล้วเชื่อมต่อ
กันผ่าน API หรือ Message Queue — แนวคิดนี้เรียกว่า **Polyglot Architecture**

---

## 99.3 ภาพรวม HTTP Protocol โดยย่อ สำหรับผู้ที่ไม่เคยทำ Web มาก่อน (Step 787)

ก่อนจะลงลึกเรื่องสถาปัตยกรรมต่างๆ ต้องแน่ใจว่าเข้าใจพื้นฐานของ **HTTP (HyperText Transfer
Protocol)** ก่อน เพราะทุกอย่างใน Module นี้สร้างอยู่บนพื้นฐานนี้ทั้งหมด

### HTTP คือ Protocol ระดับ Application ที่วิ่งอยู่บน TCP

จำได้จาก Part 33 ไหมว่า TCP คือ Byte Stream ที่ไม่มีขอบเขตข้อความในตัว Protocol เอง? HTTP
คือ **Application Protocol** ที่กำหนดกฎเกณฑ์เพิ่มเติมข้างบน TCP เพื่อให้ทั้งสองฝั่ง (Browser/
Client และ Web Server) เข้าใจตรงกันว่าข้อความที่ส่งไปมาคืออะไร มีขอบเขตแค่ไหน และควรตอบสนอง
อย่างไร

```
                    Application Layer:  HTTP    <- Part นี้เรียนเรื่องนี้
                          │
                    Transport Layer:    TCP     <- เรียนไปแล้วใน Part 33
                          │
                    Network Layer:      IP
                          │
                    Link Layer:         Ethernet/Wi-Fi
```

### รูปแบบของ HTTP Request

```
GET /index.html HTTP/1.1          <- Request Line: Method + Path + Version
Host: www.example.com             <- Header (key: value)
User-Agent: curl/8.5.0            <- Header
Accept: */*                       <- Header
                                   <- บรรทัดว่าง (คั่นระหว่าง Header กับ Body)
(Body - อาจไม่มีก็ได้ เช่น GET request มักไม่มี body)
```

### รูปแบบของ HTTP Response

```
HTTP/1.1 200 OK                          <- Status Line: Version + Status Code + Status Text
Content-Type: text/html                  <- Header
Content-Length: 137                      <- Header (สำคัญมาก จะเรียนละเอียดใน Part 100)
                                          <- บรรทัดว่าง
<html>...</html>                         <- Body (เนื้อหาจริงที่ Client จะเอาไปแสดงผล)
```

### HTTP Method ที่ใช้บ่อยที่สุด

| Method | ความหมาย | ตัวอย่างการใช้งาน |
|---|---|---|
| `GET` | ขอข้อมูล ไม่ควรมีผลข้างเคียง (Side Effect) กับข้อมูลฝั่ง Server | เปิดหน้าเว็บ, ดึงรายการสินค้า |
| `POST` | ส่งข้อมูลไปสร้างสิ่งใหม่ หรือทำ Action ที่มีผลข้างเคียง | สมัครสมาชิก, สร้างออเดอร์ใหม่ |
| `PUT` | แก้ไขข้อมูลที่มีอยู่แล้วทั้งก้อน (Replace) | อัปเดตข้อมูลโปรไฟล์ทั้งหมด |
| `DELETE` | ลบข้อมูล | ลบโพสต์, ลบบัญชีผู้ใช้ |
| `PATCH` | แก้ไขข้อมูลบางส่วน | อัปเดตแค่ชื่อผู้ใช้อย่างเดียว |

### HTTP Status Code ที่ใช้บ่อยที่สุด

| กลุ่ม | ความหมาย | ตัวอย่าง |
|---|---|---|
| `2xx` | สำเร็จ | `200 OK`, `201 Created`, `204 No Content` |
| `3xx` | Redirect (ให้ไปที่อื่นต่อ) | `301 Moved Permanently`, `302 Found` |
| `4xx` | Client ทำผิดพลาด | `400 Bad Request`, `404 Not Found`, `401 Unauthorized` |
| `5xx` | Server มีปัญหา | `500 Internal Server Error`, `503 Service Unavailable` |

> Part 100 จะให้เขียนโค้ดสร้าง HTTP Response ที่ถูกต้องตาม Spec เหล่านี้ด้วยมือทั้งหมดจาก
> raw socket ตรงๆ ไม่มี Library ช่วยเลย เพื่อให้เข้าใจอย่างลึกซึ้งว่าเบื้องหลัง Framework ทุกตัว
> (Crow, Pistache, Django, Express) จริงๆ แล้วทำอะไรอยู่

---

## 99.4 สถาปัตยกรรม CGI (Common Gateway Interface) (Step 788)

**CGI** คือสถาปัตยกรรมที่เก่าแก่ที่สุดในการเชื่อมต่อ Web Server เข้ากับโปรแกรมที่ประมวลผล
Business Logic (เกิดขึ้นตั้งแต่ต้นทศวรรษ 1990) แนวคิดหลักคือ: **Web Server รับ Request มา
แล้ว "รันโปรแกรมใหม่" (fork + exec) หนึ่งตัวต่อหนึ่ง Request** ส่งข้อมูล Request ผ่าน
**Environment Variable** และ **stdin** แล้วอ่านผลลัพธ์กลับจาก **stdout** ของโปรแกรมนั้น

```
┌──────────┐  Request   ┌─────────────┐   fork() + exec()   ┌──────────────────┐
│  Browser  │───────────▷│  Web Server  │─────────────────────▷│  CGI Program      │
│ (Client)  │            │ (Apache/nginx)│                     │ (โปรแกรมใหม่ทุกครั้ง)│
│           │            │             │                     │ - อ่าน env vars    │
│           │◁───────────│             │◁────────────────────│ - พิมพ์ผลออก stdout │
└──────────┘  Response   └─────────────┘   stdout ของ process  └──────────────────┘
```

Web Server จะตั้งค่า Environment Variable ที่สำคัญให้โปรแกรม CGI อ่านได้ก่อนรัน เช่น:

| Environment Variable | ความหมาย |
|---|---|
| `REQUEST_METHOD` | Method ของ Request เช่น `GET`, `POST` |
| `QUERY_STRING` | ส่วนที่อยู่หลัง `?` ใน URL เช่น `name=somchai&age=25` |
| `CONTENT_LENGTH` | ความยาวของ body (สำหรับ POST) |
| `REMOTE_ADDR` | IP Address ของ Client |
| `SERVER_PROTOCOL` | เวอร์ชันของ HTTP ที่ใช้ |

ตัวอย่างโปรแกรม CGI ที่เขียนด้วย C ง่ายๆ ตัวหนึ่ง:

```c
/* hello_cgi.c - ตัวอย่าง CGI program ง่ายๆ ที่รันผ่าน Web Server ทั่วไป */
#include <stdio.h>
#include <stdlib.h>

int main(void) {
    const char *query = getenv("QUERY_STRING");
    if (query == NULL) {
        query = "";
    }

    /* CGI ต้องพิมพ์ HTTP Header ก่อนเสมอ ตามด้วยบรรทัดว่าง 1 บรรทัด แล้วค่อยตามด้วย body
     * Web Server จะอ่าน stdout ของโปรแกรมนี้ แล้วต่อ Status Line ("HTTP/1.1 200 OK") ให้เอง */
    printf("Content-Type: text/html\r\n");
    printf("\r\n");
    printf("<html><body>\n");
    printf("<h1>Hello from CGI (C program)</h1>\n");
    printf("<p>Query string: %s</p>\n", query);
    printf("</body></html>\n");

    return 0;
}
```

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 hello_cgi.c -o hello_cgi.cgi
```

ในการใช้งานจริงจะเอาไฟล์ `hello_cgi.cgi` นี้ไปวางไว้ในโฟลเดอร์ `cgi-bin/` ของ Web Server
(เช่น Apache) ให้ทดสอบพฤติกรรมของโปรแกรมได้ตรงๆ ด้วยการจำลอง Environment Variable ที่ Web
Server จะตั้งให้ก่อนเรียกโปรแกรมทุกครั้ง:

```bash
REQUEST_METHOD=GET QUERY_STRING="name=somchai" ./hello_cgi.cgi
```

ผลลัพธ์ที่ได้จริง (รันทดสอบแล้วบนเครื่องจริง):

```
Content-Type: text/html

<html><body>
<h1>Hello from CGI (C program)</h1>
<p>Query string: name=somchai</p>
</body></html>
```

สังเกตว่าผลลัพธ์เริ่มด้วย Header (`Content-Type`) ตามด้วยบรรทัดว่าง แล้วตามด้วย Body — Web
Server จะอ่านเอาต์พุตนี้ แล้วเติม Status Line (`HTTP/1.1 200 OK`) ต่อหน้าให้เองก่อนส่งกลับไป
หา Browser

### ข้อดีของ CGI

- **เรียบง่ายที่สุด**: เขียนโปรแกรมอะไรก็ได้ ภาษาอะไรก็ได้ (Perl, C, Python, Shell Script)
  ขอแค่อ่าน stdin/Environment Variable และพิมพ์ผลลัพธ์ที่ถูก Format ออก stdout ได้ก็พอ
- **แยกส่วนกันชัดเจน**: แต่ละ Request เป็น Process ใหม่ทั้งหมด ถ้าโปรแกรม crash ก็กระทบแค่
  Request นั้น ไม่ลาม Process อื่น (Fault Isolation ที่ดีมาก)

### ข้อเสียของ CGI ที่ทำให้ถูกแทนที่ในระบบสมัยใหม่ส่วนใหญ่

- **สร้าง Process ใหม่ทุก Request**: `fork()` + `exec()` มี Overhead สูงมากเมื่อเทียบกับ
  จำนวน Request ต่อวินาทีที่ระบบสมัยใหม่ต้องรองรับ (นี่คือปัญหาแบบเดียวกับที่จะเจอใน Part 101
  เรื่อง "สร้าง Thread ใหม่ทุก Connection" เพียงแต่ CGI สร้างทั้ง Process ซึ่งหนักกว่า Thread
  มาก)
- **ไม่มี State ระหว่าง Request**: เพราะแต่ละ Request เป็นคนละ Process กัน ทำให้เชื่อมต่อ
  Database ใหม่ทุกครั้ง (Connection Pool ทำไม่ได้ง่ายๆ) และไม่มี Memory Cache ที่ใช้ร่วมกัน
  ข้าม Request ได้เลย

### CGI ยังใช้อยู่ที่ไหนในปี 2026

แม้จะเป็นเทคโนโลยีเก่า CGI ยังคงพบได้ในบางบริบท: ระบบ Legacy ขององค์กรขนาดใหญ่ที่ยังไม่ได้
Migrate, Shared Hosting ราคาถูกบางเจ้าที่รองรับแค่ CGI/Perl, และงาน Script อัตโนมัติเบื้องหลัง
บางประเภทที่ไม่สนใจ Performance สูง เพราะรันไม่บ่อย — เข้าใจ CGI จึงยังมีประโยชน์แม้จะไม่ได้
เอาไปสร้างระบบใหม่ในปัจจุบัน

---

## 99.5 สถาปัตยกรรม FastCGI: แก้ปัญหา Overhead ของ CGI (Step 789)

**FastCGI** ถูกออกแบบมาเพื่อแก้จุดอ่อนที่ใหญ่ที่สุดของ CGI โดยตรง: **การสร้าง Process ใหม่
ทุก Request** แนวคิดหลักคือเปลี่ยนจาก "รัน Process ใหม่ทุกครั้ง" เป็น **"รัน Process (หรือ
Thread) ไว้ล่วงหน้าถาวร แล้วให้ Web Server ส่ง Request หลายๆ ตัวต่อเนื่องมาให้ Process
เดียวกันจัดการ ผ่าน Protocol พิเศษ"**

```
CGI (แบบเดิม):                              FastCGI (แบบใหม่):

Request 1 -> fork() process ใหม่ -> ตาย       Process ถาวร (เปิดค้างไว้)
Request 2 -> fork() process ใหม่ -> ตาย            │
Request 3 -> fork() process ใหม่ -> ตาย       Request 1 ─┤
   (สร้าง-ทำลาย process ทุกครั้ง                Request 2 ─┼─▷ จัดการโดย process เดิม
    overhead สูงมาก)                          Request 3 ─┤    (ไม่ fork ใหม่ทุกครั้ง)
```

Web Server (เช่น nginx) จะคุยกับ FastCGI Process ผ่าน Protocol ที่นิยามไว้เฉพาะ (มักผ่าน
Unix Domain Socket หรือ TCP Socket ภายในเครื่องเดียวกัน) แทนที่จะสื่อสารผ่าน Environment
Variable + stdin/stdout เหมือน CGI ดั้งเดิม ทำให้:

- **ไม่มี Overhead ของการสร้าง Process ใหม่ทุก Request** — Process ทำงานต่อเนื่องได้ยาวนาน
  เหมือน Web Server ทั่วไป
- **รักษา State ข้าม Request ได้** — เชื่อมต่อ Database ค้างไว้ (Connection Pool) หรือ Cache
  ข้อมูลไว้ใน Memory ระหว่าง Request ได้ตามปกติ
- **ยังคงแยก Process จาก Web Server หลัก** — ถ้า FastCGI Process crash ก็ไม่ทำให้ nginx/Apache
  ล่มไปด้วย (Fault Isolation ยังคงมีอยู่บางส่วน ต่างจากการฝัง Logic ไว้ใน Module ของ Web
  Server โดยตรง)

### ตัวอย่างการใช้งานจริงของ FastCGI

- **PHP-FPM (FastCGI Process Manager)**: วิธีมาตรฐานที่ nginx ใช้รัน PHP ในปัจจุบัน (แทนที่
  `mod_php` แบบเก่าที่ฝัง PHP Interpreter ไว้ใน Process ของ Apache โดยตรง)
  ```
  nginx  <--FastCGI Protocol-->  php-fpm (process pool ที่เปิดค้างไว้ถาวร)
  ```
- **fcgiwrap**: ตัวห่อ (wrapper) ที่ทำให้โปรแกรม CGI แบบเดิม (รวมถึงโปรแกรม C/C++ ที่เขียนตาม
  Spec ของ CGI) ทำงานผ่าน FastCGI Protocol ได้โดยไม่ต้องเขียนโปรแกรมใหม่ทั้งหมด

FastCGI จึงเป็นสถาปัตยกรรม **"ตัวกลาง"** ระหว่างความง่ายของ CGI กับ Performance ที่ดีขึ้นมาก
โดยไม่ต้องเปลี่ยนวิธีคิดเรื่องการแยก Business Logic ออกจาก Web Server หลัก

---

## 99.6 Embedded HTTP Server Library: Crow, Pistache, Drogon (Step 790)

สถาปัตยกรรมที่สามคือแนวทางที่หลักสูตรนี้จะใช้เป็นหลักตั้งแต่ **Part 102** เป็นต้นไป: แทนที่
จะพึ่งพา Web Server ภายนอก (Apache/nginx) มาส่ง Request ให้โปรแกรมของเราผ่าน CGI/FastCGI
**โปรแกรม C++ ของเราเองจะทำหน้าที่เป็น HTTP Server เต็มรูปแบบในตัวมันเอง** (Embedded HTTP
Server) โดยใช้ Library ที่สร้างอยู่บน Socket API แบบเดียวกับที่เรียนใน Part 33-34

```
สถาปัตยกรรม CGI/FastCGI:              สถาปัตยกรรม Embedded HTTP Server:

Browser -> nginx -> CGI/FastCGI       Browser -> โปรแกรม C++ ของเราเอง
           (Web Server แยกต่างหาก)              (เป็น HTTP Server ในตัวเอง
                                                  ใช้ Crow/Pistache/Drogon)
```

Library กลุ่มนี้ทำหน้าที่จัดการรายละเอียดที่ซับซ้อนของ HTTP Protocol และ I/O Multiplexing
(select/poll/epoll ที่เรียนใน Part 34) ให้เราโดยอัตโนมัติ เหลือแค่ให้เราเขียน **Route Handler**
(ฟังก์ชันที่บอกว่า path ไหนควรตอบอะไร) เท่านั้น — ซึ่ง Part 100-101 จะให้เราสร้างเวอร์ชัน
**ดิบที่สุด** ของสิ่งนี้ด้วยมือตัวเองก่อน เพื่อให้เข้าใจว่า Library เหล่านี้ทำอะไรอยู่เบื้องหลัง
กันแน่ ก่อนจะไปใช้ Library จริงใน Part 102 เป็นต้นไป

| Library | จุดเด่น | จะเรียนใน |
|---|---|---|
| **Crow** | Syntax คล้าย Flask (Python) เขียนง่าย เรียนรู้เร็ว เหมาะเป็นจุดเริ่มต้น | Part 102-103 |
| **Pistache** | เน้น REST API โดยเฉพาะ ออกแบบมาให้ทำงานแบบ Asynchronous ได้ดี | Part 104 |
| **Drogon** | Framework ที่ครบเครื่องที่สุดในกลุ่มนี้ มี ORM, WebSocket, Coroutine (C++20) ในตัว เน้น Performance สูงสุด | กล่าวถึงเปรียบเทียบใน Part 104 |

การใช้ Embedded HTTP Server Library เป็นแนวทางที่นิยมมากขึ้นเรื่อยๆ ในระบบ Microservice
สมัยใหม่ เพราะแต่ละ Service เป็นโปรแกรมเดี่ยวที่รันได้ทันที (Self-contained) เหมาะกับการ
Deploy ผ่าน Container (Docker) ที่จะเรียนใน Part 112

---

## 99.7 Reverse Proxy Pattern: nginx หน้าบ้าน + C++ Backend หลังบ้าน (Step 791)

แม้โปรแกรม C++ ของเราจะเป็น HTTP Server เต็มรูปแบบในตัวเองได้แล้ว (จากหัวข้อ 99.6) ระบบ
Production จริงแทบทั้งหมดกลับยังคงวาง **nginx** (หรือ Web Server อื่นที่คล้ายกัน) ไว้ **หน้าบ้าน**
เสมอ แทนที่จะให้ Browser คุยกับโปรแกรม C++ ของเราตรงๆ รูปแบบนี้เรียกว่า **Reverse Proxy
Pattern**

```
                    ┌─────────────────────────────────────────┐
                    │              nginx (Reverse Proxy)         │
Browser ──HTTPS──▷  │  - จัดการ TLS/HTTPS certificate            │
                    │  - Serve static file (รูปภาพ, CSS, JS)     │
                    │  - Rate limiting / บล็อก IP ที่น่าสงสัย     │
                    │  - Load balance ไปยัง backend หลายตัว      │
                    └──────────────────┬──────────────────────┘
                                       │ HTTP (internal network)
                     ┌─────────────────┼─────────────────┐
                     ▼                 ▼                 ▼
              ┌────────────┐   ┌────────────┐   ┌────────────┐
              │ C++ Backend │   │ C++ Backend │   │ C++ Backend │
              │  Instance 1 │   │  Instance 2 │   │  Instance 3 │
              └────────────┘   └────────────┘   └────────────┘
```

### ทำไมต้องมี nginx อยู่หน้าบ้าน ทั้งที่โปรแกรม C++ ของเราเป็น HTTP Server อยู่แล้ว

1. **TLS/HTTPS Termination**: การตั้งค่า HTTPS Certificate และการเข้ารหัส/ถอดรหัสที่ถูกต้อง
   ปลอดภัย และอัปเดตตามมาตรฐานความปลอดภัยล่าสุดอยู่เสมอเป็นงานเฉพาะทางที่ nginx ทำมาอย่าง
   เชี่ยวชาญนานหลายสิบปี การเขียนเองใหม่มีความเสี่ยงด้านความปลอดภัยสูงโดยไม่จำเป็น
2. **Serve Static File ได้เร็วกว่า**: nginx ถูก Optimize มาเฉพาะสำหรับการส่งไฟล์นิ่งๆ (รูปภาพ,
   CSS, JavaScript) ให้ปล่อยงานนี้ไว้ที่ nginx แล้วให้โปรแกรม C++ ของเราโฟกัสแค่ Business
   Logic (Dynamic Content) เท่านั้น
3. **Load Balancing**: ถ้า Backend หนึ่งตัวรองรับ Traffic ไม่พอ nginx กระจาย Request ไปยัง
   หลาย Instance ของโปรแกรม C++ ได้ทันทีโดยไม่ต้องแก้โค้ด Backend เลย
4. **Zero-downtime Deployment**: อัปเดตโปรแกรม C++ backend ทีละ Instance ได้ในขณะที่ nginx
   ยังคงส่ง Traffic ไปยัง Instance อื่นที่ยังทำงานอยู่ตามปกติ ผู้ใช้ไม่รู้สึกถึง Downtime เลย
5. **ป้องกันชั้นแรก (Defense in Depth)**: Rate Limiting, บล็อก Pattern การโจมตีที่รู้จักแล้ว,
   และการกรอง Request ที่ผิดปกติ ทำได้สะดวกกว่าที่ Layer นี้ก่อนที่จะถึงโปรแกรม Backend เลย

รูปแบบสถาปัตยกรรมนี้จะเป็นหัวข้อเจาะลึกใน **Part 112** (Part สุดท้ายของ Module I) ซึ่งจะสอน
การตั้งค่า nginx เป็น Reverse Proxy จริง พร้อม Deploy โปรแกรม C++ ทั้งหมดที่สร้างมาตลอด Module
นี้ผ่าน Docker

---

## 99.8 Roadmap ของ Module I ทั้งหมด (Step 792)

Module I มีทั้งหมด 14 Part (Part 99-112) โดยแบ่งเป็น 4 ช่วงใหญ่ๆ ตามลำดับความลึก:

```
ช่วงที่ 1: รากฐาน (From Scratch)                Part 99-101
   │  Part 99  - ภาพรวม + สถาปัตยกรรม (Part นี้)
   │  Part 100 - Raw HTTP Server จาก socket ด้วยมือ (รับได้ทีละ 1 client)
   │  Part 101 - Multi-threaded HTTP Server + Thread Pool (รับหลาย client พร้อมกัน)
   │
ช่วงที่ 2: Web Framework จริง                   Part 102-104
   │  Part 102 - Crow Framework: พื้นฐานและ REST API แรก
   │  Part 103 - Crow ขั้นสูง: Middleware, Routing, JSON Response
   │  Part 104 - Pistache Framework: ทางเลือกสำหรับ REST API ระดับ Production
   │
ช่วงที่ 3: ระบบ Backend ที่ใช้งานได้จริง         Part 105-109
   │  Part 105 - เชื่อมต่อฐานข้อมูล SQLite
   │  Part 106 - เชื่อมต่อฐานข้อมูล PostgreSQL/MySQL
   │  Part 107 - จัดการ JSON ด้วย nlohmann/json
   │  Part 108 - โปรเจกต์: REST API Backend แบบ CRUD ครบวงจร
   │  Part 109 - WebSocket Programming ด้วย C++
   │
ช่วงที่ 4: Production-Ready                     Part 110-112
      Part 110 - Authentication/Security: JWT, bcrypt, HTTPS
      Part 111 - Server-Side Rendering: Serve HTML/Template จาก C++
      Part 112 - Deploy: Docker, Nginx Reverse Proxy, Linux Production Server
```

สังเกตว่าโครงสร้างของ Module I เดินตามปรัชญาเดียวกับทั้งหลักสูตรนี้ตั้งแต่ Part 1: **เข้าใจ
กลไกดิบๆ เบื้องหลังก่อนเสมอ แล้วค่อยขยับขึ้นไปใช้เครื่องมือระดับสูงที่ทำงานเดียวกันแต่สะดวก
กว่า** — เราจะไม่กระโดดไปเปิด Crow แล้วเขียน `CROW_ROUTE(app, "/")` ทันทีโดยไม่รู้ว่าเบื้องหลัง
มันทำอะไรอยู่ แต่จะสร้าง HTTP Server ดิบๆ ด้วย `socket()`/`bind()`/`listen()`/`accept()` เอง
ก่อนใน Part 100-101 เพื่อให้เมื่อไปถึง Part 102 และเห็น `CROW_ROUTE` ทำงาน จะเข้าใจทันทีว่า
มันคือ Abstraction ของสิ่งที่เราเพิ่งเขียนเองมาด้วยมือ ไม่ใช่ "มายากล" ที่ไม่รู้ว่าทำงานอย่างไร

> ทักษะที่สั่งสมมาตลอด 98 Part ก่อนหน้านี้ทั้งหมดจะถูกใช้งานจริงใน Module นี้: Socket
> Programming (Part 33-35), Multithreading และ Thread Pool (Part 31-32, 81-82), Smart
> Pointer (Module E) สำหรับจัดการ Resource ของ Connection, Design Pattern (Part 97-98)
> โดยเฉพาะ Observer สำหรับระบบ Event-driven และ Strategy สำหรับ Middleware, RAII (Module D)
> สำหรับปิด Socket/File ให้ปลอดภัยเสมอแม้เกิด Exception, และ Build System/Testing (Module H)
> สำหรับจัดการโปรเจกต์ขนาดใหญ่ที่มีหลายไฟล์และ Dependency ภายนอก

---

## ข้อผิดพลาดที่พบบ่อย (Common Pitfalls)

1. **คิดว่า "C/C++ เร็วกว่า จึงควรใช้ทำ Web เสมอ"** — เป็นความเข้าใจผิดที่พบบ่อยที่สุด
   Performance สูงมาพร้อมต้นทุนด้าน Development Velocity ที่สูงตามไปด้วยเสมอ (เขียนช้ากว่า,
   Debug ยากกว่า, บั๊กด้าน Memory ที่ไม่มีใน GC Language) ควรใช้ C/C++ เมื่อ Requirement ทาง
   ธุรกิจต้องการ Performance จริงๆ เท่านั้น ไม่ใช่เพราะ "อยากเขียนเร็วไว้ก่อน"
2. **สับสนระหว่าง CGI กับ FastCGI ว่าเป็นสิ่งเดียวกัน** — ทั้งสองมีชื่อคล้ายกันแต่สถาปัตยกรรม
   ต่างกันโดยพื้นฐาน CGI สร้าง Process ใหม่ทุก Request ส่วน FastCGI ใช้ Process ถาวรที่รับ
   หลาย Request ต่อเนื่องกัน
3. **เข้าใจผิดว่า Embedded HTTP Server Library ทำให้ไม่ต้องมี nginx อีกต่อไป** — แม้โปรแกรม
   C++ จะเป็น HTTP Server สมบูรณ์ในตัวเองได้ แต่ระบบ Production จริงยังคงต้องการ nginx หรือ
   เทียบเท่าอยู่หน้าบ้านเสมอ ด้วยเหตุผลด้าน TLS, Load Balancing, และความปลอดภัยตามที่อธิบาย
   ในหัวข้อ 99.7
4. **มองข้าม HTTP Header `Content-Length`** เพราะดูเป็นรายละเอียดเล็กน้อย — Header นี้สำคัญ
   มากจนต้องใช้เวลาทั้ง Part 100 อธิบายและสาธิตบั๊กที่เกิดจากการลืมใส่มันโดยเฉพาะ
5. **เริ่มต้นด้วยการเปิด Framework ทันทีโดยไม่เข้าใจ HTTP/Socket เบื้องหลัง** — ทำให้เมื่อเจอ
   ปัญหาที่ Framework จัดการไม่ได้ตรงตามที่ต้องการ (เช่น Custom Header, Performance Tuning)
   จะไม่รู้ว่าจะแก้ปัญหาจากตรงไหน เพราะไม่เข้าใจกลไกที่แท้จริงข้างใต้

---

## แบบฝึกหัดท้ายบท

1. เขียนตารางเปรียบเทียบข้อดี-ข้อเสียของ C/C++ กับภาษาที่คุณคุ้นเคยที่สุด (เช่น Python หรือ
   JavaScript) สำหรับการสร้าง Web Backend ในบริบทของโปรเจกต์ที่คุณสนใจจริง (เช่น ระบบ
   E-commerce ขนาดเล็ก, ระบบ Chat, หรือ Game Backend)
2. เขียนโปรแกรม CGI ด้วยภาษา C ที่พิมพ์ **Environment Variable ทั้งหมด** ที่ Web Server
   ส่งมาให้ (ใช้ `extern char **environ;`) แล้วทดสอบด้วยการตั้งค่า Environment Variable
   จำลองก่อนรัน (`REQUEST_METHOD=GET QUERY_STRING=... ./program`)
3. อธิบายด้วยคำพูดตัวเอง (ไม่ต้องเขียนโค้ด) ว่าทำไม FastCGI ถึงยังคง "แยก Process" ออกจาก
   Web Server หลัก ทั้งที่มีเป้าหมายเพื่อ Performance ที่ดีขึ้น ไม่ทำไมไม่รวมเป็น Process
   เดียวกันไปเลยเหมือน Embedded HTTP Server
4. วาดสถาปัตยกรรม Reverse Proxy ของระบบสมมติที่มี Backend 3 ตัว, Redis สำหรับ Cache, และ
   PostgreSQL สำหรับ Database หลัก (ใช้ ASCII Diagram แบบในบทเรียนหรือวาดด้วยมือก็ได้)
5. เขียนโปรแกรม CGI ที่ตรวจสอบค่า `REQUEST_METHOD` แล้วตอบข้อความต่างกันสำหรับ `GET`,
   `POST`, และ Method อื่นๆ ที่ไม่รู้จัก
6. อธิบายว่าทำไม Module I ถึงเลือกสอน Raw Socket HTTP Server (Part 100-101) ก่อนสอน
   Framework จริง (Part 102 เป็นต้นไป) ทั้งที่ในงานจริงแทบไม่มีใครเขียน HTTP Server จาก
   Socket ตรงๆ อีกแล้ว

### แนวทางเฉลยข้อ 2

```c
/* env_cgi.c - เฉลยข้อ 2: CGI program ที่พิมพ์ environment variable ทั้งหมดที่ web server ส่งให้ */
#include <stdio.h>

extern char **environ;   /* ตัวแปร global มาตรฐานที่ทุกโปรแกรม C มี ชี้ไปยัง array
                           * ของ environment variable ทั้งหมดของ process (จบด้วย NULL) */

int main(void) {
    printf("Content-Type: text/plain\r\n");
    printf("\r\n");
    printf("=== Environment Variables ที่ Web Server ส่งให้ CGI Program ===\n");
    for (char **env = environ; *env != NULL; env++) {
        printf("%s\n", *env);
    }
    return 0;
}
```

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 env_cgi.c -o env_cgi.cgi
REQUEST_METHOD=GET QUERY_STRING="q=1" SERVER_PROTOCOL=HTTP/1.1 REMOTE_ADDR=127.0.0.1 ./env_cgi.cgi
```

ผลลัพธ์ตัวอย่าง (ตัดเหลือเฉพาะ Environment Variable ที่เกี่ยวข้องกับ CGI โดยตรง — เมื่อรันจริง
จะเห็น Environment Variable ของระบบปฏิบัติการปนอยู่ด้วยจำนวนมาก เพราะ `environ` แสดงค่าทั้งหมด
ของ process ไม่ได้กรองเฉพาะที่ Web Server ตั้งให้):

```
Content-Type: text/plain

=== Environment Variables ที่ Web Server ส่งให้ CGI Program ===
REQUEST_METHOD=GET
QUERY_STRING=q=1
REMOTE_ADDR=127.0.0.1
SERVER_PROTOCOL=HTTP/1.1
...
(ตามด้วย environment variable อื่นๆ ของระบบปฏิบัติการที่ process สืบทอดมา)
```

จุดสำคัญ: เวลารันผ่าน Web Server จริง (เช่น Apache + `mod_cgi`) ตัวแปรอย่าง `REQUEST_METHOD`,
`QUERY_STRING`, `REMOTE_ADDR`, `SERVER_PROTOCOL` จะถูกตั้งค่าให้อัตโนมัติตาม Request จริงที่
เข้ามา — การจำลองด้วยการตั้ง Environment Variable เองก่อนรัน (`REQUEST_METHOD=GET ...`) คือ
วิธีทดสอบพฤติกรรมของโปรแกรม CGI โดยไม่ต้องติดตั้ง Web Server เต็มรูปแบบระหว่างพัฒนา

### แนวทางเฉลยข้อ 5

```c
/* method_cgi.c - เฉลยข้อ 5: CGI program ที่ตอบต่างกันตาม REQUEST_METHOD */
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

int main(void) {
    const char *method = getenv("REQUEST_METHOD");
    if (method == NULL) {
        method = "UNKNOWN";
    }

    printf("Content-Type: text/plain\r\n");
    printf("\r\n");

    if (strcmp(method, "GET") == 0) {
        printf("ได้รับ GET request - ส่งข้อมูลกลับไปแสดงผลตามปกติ\n");
    } else if (strcmp(method, "POST") == 0) {
        printf("ได้รับ POST request - ควรอ่านข้อมูลจาก stdin ตามความยาวใน CONTENT_LENGTH\n");
    } else {
        printf("Method ที่ไม่รองรับ: %s\n", method);
    }
    return 0;
}
```

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 method_cgi.c -o method_cgi.cgi
echo "--- GET ---";    REQUEST_METHOD=GET    ./method_cgi.cgi
echo "--- POST ---";   REQUEST_METHOD=POST   ./method_cgi.cgi
echo "--- DELETE ---"; REQUEST_METHOD=DELETE ./method_cgi.cgi
```

ผลลัพธ์จริงจากการรันทดสอบ:

```
--- GET ---
Content-Type: text/plain

ได้รับ GET request - ส่งข้อมูลกลับไปแสดงผลตามปกติ
--- POST ---
Content-Type: text/plain

ได้รับ POST request - ควรอ่านข้อมูลจาก stdin ตามความยาวใน CONTENT_LENGTH
--- DELETE ---
Content-Type: text/plain

Method ที่ไม่รองรับ: DELETE
```

(ข้อ 1, 3, 4 และ 6 เป็นคำถามเชิงอธิบาย/ออกแบบที่ไม่มีโค้ดตายตัว ให้ผู้เรียนลองเขียนคำตอบด้วย
คำพูดตัวเองตามแนวทางที่อธิบายไว้ในเนื้อหาบทเรียนแต่ละหัวข้อ)

---

## สรุปท้ายบท

ใน Part นี้เราได้:

- เข้าใจเหตุผลที่แท้จริงที่ทำให้ C/C++ ยังคงเป็นตัวเลือกสำคัญสำหรับ Web Backend บางประเภท
  แม้ภาษาอื่นจะพัฒนาได้เร็วกว่า — Performance สูงสุด, การควบคุม Memory/Latency ละเอียด,
  และ Use Case ที่ Throughput สูงมาก เช่น Trading System, Ad Server, Game Backend
- เปรียบเทียบ C/C++ กับ Node.js, Python, Go อย่างเป็นกลาง และเข้าใจหลักการเลือกเทคโนโลยี
  ให้เหมาะกับปัญหา ไม่ใช่เลือกเพราะกระแส
- ทบทวนภาพรวมของ HTTP Protocol (Request/Response, Method, Status Code) สำหรับผู้ที่ยังไม่
  เคยทำ Web Development มาก่อน
- เข้าใจสถาปัตยกรรม **CGI** (สร้าง Process ใหม่ทุก Request), **FastCGI** (Process ถาวรที่
  แก้ปัญหา Overhead ของ CGI), **Embedded HTTP Server Library** (โปรแกรมของเราเป็น Server
  เต็มรูปแบบในตัวเอง), และ **Reverse Proxy Pattern** (nginx หน้าบ้าน + C++ backend หลังบ้าน)
- เห็นภาพรวม Roadmap ของ Module I ทั้ง 14 Part ตั้งแต่ Raw Socket ไปจนถึง Production Deploy

ใน **Part 100** เราจะเริ่มลงมือจริงด้วยการสร้าง **HTTP Server เปล่าๆ จาก Socket API ด้วยมือ**
ที่เรียนมาจาก Part 33 ทั้งหมด ไม่ใช้ Library ใดๆ เลย เพื่อให้เข้าใจอย่างถ่องแท้ว่า HTTP ทำงาน
อย่างไรจริงๆ เบื้องหลัง Framework ทุกตัวที่เราจะได้ใช้ในภายหลัง

**ต่อไป:** [Part 100 — สร้าง Raw HTTP Server จาก Socket ด้วยมือ](./part-100-raw-http-server.md)
