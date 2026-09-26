# Part 33: Socket Programming เบื้องต้น (TCP) (Step 257–264)

> Module C — Systems Programming ด้วย C บน Linux | Part 33 จาก 125
> Step ที่ครอบคลุมใน Part นี้: Step 257–264
> Part ก่อนหน้า: [Part 32 — Thread Synchronization](./part-032-thread-sync.md) | Part ถัดไป: [Part 34 — Socket ขั้นสูง (UDP/select/poll/epoll)](./part-034-sockets-advanced.md)

## เป้าหมายการเรียนรู้ (Learning Objectives)

หลังจบ Part นี้ ผู้เรียนจะสามารถ:

1. อธิบาย Network Programming Model แบบ Client-Server ได้ และเข้าใจว่า Socket เป็นจุดเชื่อม
   ระหว่างโปรแกรมกับ Network Stack ของ OS อย่างไร
2. อธิบายความหมายของ Socket, IP Address, Port และความแตกต่างระหว่าง TCP กับ Connectionless
   Protocol ได้อย่างถูกต้อง
3. เขียน TCP Server ที่ทำงานตามลำดับ `socket()` → `bind()` → `listen()` → `accept()` ได้เอง
   ตั้งแต่ต้นจนจบ
4. เขียน TCP Client ที่ทำงานตามลำดับ `socket()` → `connect()` ได้เอง และเชื่อมต่อกับ Server
   ที่เขียนเองได้สำเร็จ
5. ใช้ `struct sockaddr_in` และฟังก์ชันแปลง Byte Order (`htons`/`htonl`/`ntohs`/`ntohl`) ได้
   อย่างถูกต้อง พร้อมอธิบายว่าทำไมต้องมีการแปลงนี้
6. สร้าง **TCP Echo Server/Client** ตัวเต็มที่รับข้อความจาก Client แล้วส่งกลับ (1 Client ต่อครั้ง)
   และรันทดสอบร่วมกันได้จริงบนเครื่องเดียวกันผ่าน `127.0.0.1`
7. จัดการปัญหา Partial Send/Receive และ Error Handling พื้นฐานของ Socket API ได้อย่างถูกต้อง

---

## 33.1 Network Programming Model: Client-Server เบื้องต้น (Step 257)

ก่อนจะเขียนโค้ด Socket ตัวแรก ต้องเข้าใจภาพรวมของการสื่อสารระหว่างโปรแกรม 2 ตัวผ่านเครือข่าย
ก่อน รูปแบบที่พบมากที่สุดในโลกจริง (และเป็นรูปแบบที่หลักสูตรนี้จะใช้ตลอด Module C และ Module
ที่เกี่ยวกับ Web Development ในภายหลัง) คือ **Client-Server Model**:

```
┌─────────────┐                                      ┌─────────────┐
│   CLIENT     │                                      │   SERVER     │
│ (ผู้ร้องขอ)   │  ───── 1. ส่ง Request ─────────────>  │ (ผู้ให้บริการ) │
│              │                                      │              │
│              │  <──── 2. ส่ง Response กลับ ─────────  │              │
└─────────────┘                                      └─────────────┘
```

- **Server**: โปรแกรมที่ "เปิดรอ" การเชื่อมต่อที่ Port ใดๆ Port หนึ่งอย่างต่อเนื่อง (ทำงาน
  ตลอดเวลา ไม่จบเอง) เปรียบเหมือนร้านค้าที่เปิดประตูรอลูกค้า
- **Client**: โปรแกรมที่ "ริเริ่ม" การเชื่อมต่อไปหา Server ที่รู้ที่อยู่ (IP + Port) อยู่แล้ว
  ทำงานเสร็จภารกิจแล้วมักจะปิดตัวเอง เปรียบเหมือนลูกค้าที่เดินเข้าร้าน สั่งของ รับของ แล้วออกไป

โมเดลนี้เป็นรากฐานของแทบทุกอย่างในโลกอินเทอร์เน็ต: เมื่อเปิดเว็บเบราว์เซอร์ไปที่เว็บไซต์ใดๆ
เบราว์เซอร์ (Client) จะเชื่อมต่อไปหา Web Server ที่รันอยู่บนเครื่องอื่น (หรือเครื่องเดียวกัน)
ส่ง HTTP Request ไป แล้วรอรับ HTTP Response กลับมาแสดงผล — กลไกเบื้องหลังทั้งหมดนี้สร้างจาก
Socket API ที่กำลังจะเรียนใน Part นี้ทั้งสิ้น

### ที่อยู่บนเครือข่าย: IP Address และ Port

การจะส่งข้อมูลไปหาโปรแกรมหนึ่งบนเครือข่ายได้ ต้องระบุ 2 อย่าง:

1. **IP Address**: ระบุว่า "เครื่อง" ไหนในเครือข่าย (เช่น `192.168.1.10` หรือ `127.0.0.1`
   สำหรับเครื่องตัวเอง — เรียกว่า **Loopback Address**)
2. **Port**: ระบุว่า "โปรแกรม" ไหนบนเครื่องนั้น (ตัวเลข 0–65535 โดย Port 0–1023 สงวนไว้สำหรับ
   บริการมาตรฐาน เช่น 80 = HTTP, 443 = HTTPS, 22 = SSH)

```
   IP Address (เครื่องไหน)      Port (โปรแกรมไหนบนเครื่องนั้น)
   ┌──────────────────┐         ┌──────┐
   │   192.168.1.10    │  :      │ 5500 │   ->  ระบุปลายทางได้ครบถ้วน
   └──────────────────┘         └──────┘
```

ตลอด Part นี้เราจะทดสอบทั้ง Server และ Client บนเครื่องเดียวกัน โดยใช้ IP `127.0.0.1`
(Loopback) ซึ่งหมายถึง "เครื่องของตัวเอง" — วิธีนี้ทำให้ทดสอบ Network Programming ได้โดย
ไม่ต้องมีเครื่องที่สองหรือต่ออินเทอร์เน็ตเลย

---

## 33.2 Socket คืออะไร (Step 258)

**Socket** คือ Abstraction ที่ระบบปฏิบัติการมอบให้โปรแกรมเมอร์ ใช้แทน "จุดปลายทาง" (endpoint)
ของการสื่อสารบนเครือข่าย เปรียบเทียบง่ายๆ Socket เหมือน **File Descriptor พิเศษ** ที่แทนที่จะ
อ่าน/เขียนไฟล์บน Disk กลับใช้อ่าน/เขียนข้อมูลผ่านเครือข่ายแทน

ในความเป็นจริง บน Linux ฟังก์ชัน `socket()` จะคืนค่าเป็น `int` (File Descriptor) เหมือนกับที่
`open()` คืนค่าให้ตอนเปิดไฟล์ทุกประการ — นี่คือหนึ่งในปรัชญาการออกแบบที่สวยงามที่สุดของ Unix
ที่เรียกว่า **"Everything is a file"**: ไม่ว่าจะเป็นไฟล์บน Disk, Pipe (Part 29), หรือ Socket
เครือข่าย ก็ใช้ File Descriptor แบบเดียวกัน และหลายฟังก์ชัน (เช่น `read()`, `write()`,
`close()`) ก็ใช้กับ Socket ได้เหมือนกับไฟล์ทั่วไป (แม้ในทางปฏิบัติมักใช้ `send()`/`recv()`
แทนเพราะมี option พิเศษสำหรับ Network ให้ใช้เพิ่มเติม)

```
                    Application (โค้ดของเรา)
                          │
                          │  read() / write() / send() / recv()
                          ▼
                  ┌───────────────┐
                  │  Socket (fd)   │   <- Abstraction ที่ OS มอบให้
                  └───────┬───────┘
                          │
                          ▼
              Kernel: TCP/IP Network Stack
                          │
                          ▼
                   Network Interface Card (NIC)
                          │
                          ▼
                    สายเคเบิล / Wi-Fi / อินเทอร์เน็ต
```

### ประเภทของ Socket ที่สำคัญ

| ค่า Constant | ความหมาย | Protocol ที่ใช้ |
|---|---|---|
| `SOCK_STREAM` | Socket แบบ **Connection-Oriented** รับประกันลำดับข้อมูลและความครบถ้วน | TCP |
| `SOCK_DGRAM` | Socket แบบ **Connectionless** ส่งเป็นก้อนๆ (datagram) ไม่รับประกันว่าจะถึงหรือมาตามลำดับ | UDP (จะเรียนละเอียดใน Part 34) |

Part นี้จะโฟกัสที่ `SOCK_STREAM` (TCP) เท่านั้น เพราะเป็น Protocol ที่ใช้มากที่สุดในโลกจริง
(HTTP, HTTPS, SSH, Database Connection ล้วนใช้ TCP) ส่วน `SOCK_DGRAM` (UDP) จะเรียนใน Part 34

### TCP รับประกันอะไรบ้าง

TCP (Transmission Control Protocol) เป็น Protocol ระดับ Transport Layer ที่มอบคุณสมบัติ
สำคัญให้แอปพลิเคชันโดยที่เราไม่ต้องเขียนโค้ดจัดการเอง:

- **Reliable (เชื่อถือได้)**: ถ้าข้อมูลหายระหว่างทาง TCP จะส่งซ้ำให้อัตโนมัติ
- **Ordered (เรียงลำดับ)**: ข้อมูลจะถึงปลายทางตามลำดับที่ส่งเสมอ แม้ในระดับ Packet จริงๆ
  อาจมาถึงไม่ตามลำดับ (Kernel จะจัดเรียงให้ก่อนส่งต่อให้แอปพลิเคชัน)
- **Connection-Oriented**: ต้อง "จับมือ" กันก่อน (TCP 3-way Handshake) ถึงจะเริ่มส่งข้อมูล
  จริงได้ และต้องปิดการเชื่อมต่ออย่างเป็นระบบ (ไม่ใช่แค่หยุดส่งเฉยๆ)
- **Byte Stream**: มองข้อมูลเป็น "สายธารของ byte" ต่อเนื่อง ไม่มีขอบเขตของ "ข้อความ" ในตัว
  Protocol เอง (นี่คือที่มาของปัญหา Partial Send/Recv ที่จะพูดถึงในหัวข้อ 33.8)

---

## 33.3 struct sockaddr_in และ Byte Order (htons/htonl) (Step 259)

การจะระบุ "IP + Port" ในโค้ด C ต้องใช้ struct ที่ชื่อ `struct sockaddr_in` (สำหรับ IPv4)
ซึ่งประกาศไว้ใน `<netinet/in.h>`:

```c
struct sockaddr_in {
    sa_family_t    sin_family;   /* address family เช่น AF_INET สำหรับ IPv4 */
    in_port_t      sin_port;     /* port number (ต้องเป็น network byte order) */
    struct in_addr sin_addr;     /* IPv4 address (ต้องเป็น network byte order) */
    /* มี padding เพิ่มเติมเพื่อให้ขนาดเท่ากับ struct sockaddr */
};
```

ฟังก์ชันที่รับ struct นี้ (เช่น `bind()`, `connect()`) จริงๆ แล้วรับ pointer เป็น
`struct sockaddr *` (แบบทั่วไปที่รองรับได้ทั้ง IPv4 และ IPv6) เราจึงต้อง cast
`(struct sockaddr *)&addr` เวลาส่งเข้าฟังก์ชันเหล่านั้นเสมอ

### ทำไมต้องมี Byte Order (Endianness)?

CPU แต่ละสถาปัตยกรรมเก็บตัวเลขหลาย byte ในหน่วยความจำไม่เหมือนกัน:

```
ตัวเลข 16-bit: 0x1234 (ทศนิยม 4660)

Big-Endian (Network Byte Order):     [0x12][0x34]   <- byte สูงอยู่ก่อน
Little-Endian (x86/x86-64 ส่วนใหญ่):  [0x34][0x12]   <- byte ต่ำอยู่ก่อน
```

ถ้าเครื่อง A เป็น Little-Endian ส่งตัวเลข Port ดิบๆ ไปให้เครื่อง B ที่เป็น Big-Endian โดยไม่มี
มาตรฐานร่วมกัน ตัวเลขจะถูกตีความผิดทันที เพื่อแก้ปัญหานี้ Internet Protocol กำหนดให้ทุกอย่าง
ที่ส่งผ่านเครือข่าย **ต้องอยู่ในรูปแบบ Big-Endian เสมอ** เรียกว่า **Network Byte Order**
ไม่ว่าเครื่องต้นทางหรือปลายทางจะเป็นสถาปัตยกรรมแบบไหนก็ตาม

### ฟังก์ชันแปลง Byte Order

| ฟังก์ชัน | ความหมาย |
|---|---|
| `htons(x)` | **h**ost **to** **n**etwork **s**hort — แปลง 16-bit (เช่น Port) จาก Host เป็น Network Byte Order |
| `htonl(x)` | **h**ost **to** **n**etwork **l**ong — แปลง 32-bit (เช่น IPv4 Address) จาก Host เป็น Network Byte Order |
| `ntohs(x)` | **n**etwork **to** **h**ost **s**hort — แปลงกลับจาก Network เป็น Host Byte Order |
| `ntohl(x)` | **n**etwork **to** **h**ost **l**ong — แปลงกลับจาก Network เป็น Host Byte Order |

> **กฎทอง**: ทุกครั้งที่ตั้งค่า `sin_port` ต้องผ่าน `htons()` เสมอ และทุกครั้งที่อ่านค่า
> `sin_port` กลับมาแสดงผล (เช่น พิมพ์ port ของ client ที่เชื่อมต่อเข้ามา) ต้องผ่าน `ntohs()`
> เพื่อแปลงกลับเป็นตัวเลขที่มนุษย์อ่านเข้าใจ **ลืมทำขั้นตอนนี้คือหนึ่งในบั๊กที่พบบ่อยที่สุด
> ของมือใหม่ที่เขียน Socket Programming**

ส่วนการแปลง IP Address จาก String (เช่น `"127.0.0.1"`) เป็นรูปแบบ Binary ที่ Socket ใช้
(และแปลงกลับ) ใช้ฟังก์ชัน:

| ฟังก์ชัน | ความหมาย |
|---|---|
| `inet_pton(AF_INET, "127.0.0.1", &addr.sin_addr)` | แปลงจาก String เป็น Binary (**p**resentation **to** **n**etwork) — คืนค่า Network Byte Order ให้อัตโนมัติ |
| `inet_ntop(AF_INET, &addr.sin_addr, buf, len)` | แปลงจาก Binary กลับเป็น String (**n**etwork **to** **p**resentation) — ใช้แสดงผล IP ให้มนุษย์อ่าน |

ตัวอย่างการประกาศและตั้งค่า `struct sockaddr_in` ให้ครบถ้วน:

```c
#include <string.h>
#include <arpa/inet.h>
#include <netinet/in.h>

struct sockaddr_in addr;
memset(&addr, 0, sizeof(addr));        /* เคลียร์ค่าขยะให้หมดก่อนเสมอ */
addr.sin_family = AF_INET;             /* IPv4 */
addr.sin_port = htons(5500);           /* port 5500 -> ต้องผ่าน htons() */
inet_pton(AF_INET, "127.0.0.1", &addr.sin_addr); /* แปลง string เป็น binary */
```

หมายเหตุ: ค่าพิเศษ `INADDR_ANY` (ใช้ตอนฝั่ง Server เท่านั้น) หมายถึง "รับการเชื่อมต่อจากทุก
Network Interface ของเครื่อง" ไม่ต้องระบุ IP ตายตัว:

```c
addr.sin_addr.s_addr = INADDR_ANY; /* ไม่ต้องผ่าน htonl() เพราะ INADDR_ANY เป็น 0 อยู่แล้ว */
```

---

## 33.4 TCP Server Flow: socket() → bind() → listen() → accept() (Step 260)

ฝั่ง Server ต้องทำ 4 ขั้นตอนตามลำดับนี้เสมอก่อนจะเริ่มรับส่งข้อมูลกับ Client ได้:

```
┌──────────┐    ┌────────┐    ┌─────────┐    ┌──────────┐    ┌───────────────┐
│ socket() │ -> │ bind() │ -> │listen() │ -> │ accept() │ -> │ recv()/send() │
└──────────┘    └────────┘    └─────────┘    └──────────┘    └───────────────┘
 สร้าง socket    ผูก IP+Port   เริ่มฟัง        รอรับการเชื่อม   คุยกับ client
                 ให้ socket    (เข้าคิว)       ต่อจริง         (client_fd ใหม่!)
```

### ขั้นตอนที่ 1: socket() — สร้าง Socket

```c
int server_fd = socket(AF_INET, SOCK_STREAM, 0);
```

- `AF_INET`: ใช้ IPv4 (ถ้าต้องการ IPv6 ใช้ `AF_INET6`)
- `SOCK_STREAM`: ใช้ TCP (Connection-Oriented)
- `0`: Protocol ให้ระบบเลือกอัตโนมัติตามค่า 2 ตัวข้างต้น (สำหรับ `SOCK_STREAM` + `AF_INET`
  จะได้ TCP โดยอัตโนมัติ)

ฟังก์ชันคืนค่าเป็น File Descriptor (`int`) หรือ `-1` ถ้าล้มเหลว (ต้องเช็คเสมอและใช้ `perror`
รายงาน error)

### ขั้นตอนที่ 2: bind() — ผูก Socket กับ IP+Port

```c
struct sockaddr_in server_addr;
memset(&server_addr, 0, sizeof(server_addr));
server_addr.sin_family = AF_INET;
server_addr.sin_addr.s_addr = INADDR_ANY;
server_addr.sin_port = htons(PORT);

bind(server_fd, (struct sockaddr *)&server_addr, sizeof(server_addr));
```

`bind()` บอก Kernel ว่า "Socket ตัวนี้จะรับข้อมูลที่ Port นี้บน Interface นี้" ถ้า Port นั้น
ถูกใช้อยู่แล้วโดยโปรแกรมอื่น (หรือ TCP connection เก่าที่ยังอยู่ในสถานะ `TIME_WAIT`) จะได้
error `EADDRINUSE` ("Address already in use") ซึ่งแก้ได้ด้วย `setsockopt` และ `SO_REUSEADDR`
(จะแสดงในโค้ดตัวเต็มหัวข้อ 33.6)

### ขั้นตอนที่ 3: listen() — เริ่มรอรับการเชื่อมต่อ

```c
listen(server_fd, BACKLOG);
```

`listen()` เปลี่ยนสถานะ socket จาก "active" (สำหรับเชื่อมต่อออก) เป็น "passive" (สำหรับรอรับ
การเชื่อมต่อเข้า) พารามิเตอร์ `BACKLOG` คือขนาดคิวสูงสุดของ connection ที่รอ `accept()`
อยู่ (ถ้า Client เชื่อมต่อเข้ามาเร็วกว่าที่ Server เรียก `accept()` ทัน คิวนี้จะเก็บพักไว้ก่อน
สูงสุดตามค่านี้ ถ้าคิวเต็ม connection ใหม่จะถูกปฏิเสธ)

### ขั้นตอนที่ 4: accept() — รับการเชื่อมต่อจริง

```c
struct sockaddr_in client_addr;
socklen_t client_len = sizeof(client_addr);
int client_fd = accept(server_fd, (struct sockaddr *)&client_addr, &client_len);
```

นี่คือจุดที่สำคัญที่สุดที่มือใหม่มักสับสน: `accept()` **จะบล็อก (block)** จนกว่าจะมี Client
เชื่อมต่อเข้ามาจริง และเมื่อสำเร็จจะ**คืน File Descriptor ตัวใหม่** (`client_fd`) ที่**ไม่ใช่**
ตัวเดียวกับ `server_fd` — `server_fd` ยังคงใช้สำหรับรอรับ Client รายต่อไป (เรียก `accept()`
ซ้ำได้เรื่อยๆ) ในขณะที่ `client_fd` ใช้สำหรับคุยกับ Client รายที่เพิ่งเชื่อมต่อเข้ามาโดยเฉพาะ

```
                     server_fd (ตัวเดิม ใช้ accept() ซ้ำได้เรื่อยๆ)
                            │
              accept() ─────┼───────────> client_fd_1 (คุยกับ client รายที่ 1)
                            │
              accept() ─────┴───────────> client_fd_2 (คุยกับ client รายที่ 2)
```

ใน Part นี้เราจะเขียน Server แบบง่ายที่สุดที่รับ Client ได้ **ทีละราย** (accept แล้วคุยจนจบ
แล้วค่อย accept รายถัดไป) ส่วนการรับหลาย Client พร้อมกันจะเรียนใน Part 34 และ 35

### เบื้องหลัง connect()/accept(): TCP 3-Way Handshake

แม้โปรแกรมเมอร์จะไม่ต้องเขียนโค้ดจัดการเอง แต่ควรเข้าใจว่าเบื้องหลังทุกครั้งที่ Client เรียก
`connect()` สำเร็จ (และ Server เรียก `accept()` ได้ Client ตัวใหม่) จริงๆ แล้ว Kernel ทั้งสอง
ฝั่งได้แลกเปลี่ยน Packet กันไปแล้ว 3 รอบ เรียกว่า **TCP 3-Way Handshake**:

```
   Client                                          Server
   (หลัง connect())                                 (หลัง listen())
      │                                                │
      │  1) SYN (Synchronize) ─────────────────────>   │   "ขอเริ่มคุยกันได้ไหม?"
      │                                                │
      │  <───────────────────────── 2) SYN-ACK ─────   │   "ได้ ฉันพร้อมแล้ว คุณพร้อมไหม?"
      │                                                │
      │  3) ACK ────────────────────────────────────>  │   "พร้อม เริ่มคุยกันได้เลย"
      │                                                │
      │ <========== เชื่อมต่อสำเร็จ พร้อมส่งข้อมูล ==========> │
```

หลังจาก Handshake ครบ 3 รอบ `connect()` ฝั่ง Client จึงจะ return กลับมาสำเร็จ และ `accept()`
ฝั่ง Server ก็จะได้ `client_fd` ตัวใหม่กลับมาในจังหวะเดียวกัน (ในทางเทคนิค Kernel ฝั่ง Server
จะรับ connection ที่ handshake เสร็จแล้วเข้าคิว — ค่า `BACKLOG` ที่ตั้งใน `listen()` คือขนาด
คิวนี้ — ส่วน `accept()` แค่ดึง connection ที่เสร็จสมบูรณ์แล้วออกจากคิวมาใช้งานเท่านั้น)

การปิดการเชื่อมต่อก็มีกลไกคล้ายกัน เรียกว่า **4-Way Termination** (FIN/ACK สลับกันทั้งสอง
ฝั่ง) ซึ่งเกิดขึ้นอัตโนมัติเมื่อเราเรียก `close()` บน Socket — และนี่คือที่มาของสถานะ
`TIME_WAIT` ที่ทำให้ต้องใช้ `SO_REUSEADDR` ตอนพัฒนา/ทดสอบตามที่จะเห็นในหัวข้อ 33.6

---

## 33.5 TCP Client Flow: socket() → connect() (Step 261)

ฝั่ง Client ง่ายกว่าฝั่ง Server มาก เพราะไม่ต้อง `bind()`/`listen()`/`accept()` — แค่ 2 ขั้นตอน:

```
┌──────────┐    ┌────────────┐    ┌───────────────┐
│ socket() │ -> │ connect()  │ -> │ send()/recv() │
└──────────┘    └────────────┘    └───────────────┘
 สร้าง socket    เชื่อมต่อไปหา       คุยกับ server
                 server ที่รู้จัก
                 IP+Port อยู่แล้ว
```

```c
int sock_fd = socket(AF_INET, SOCK_STREAM, 0);

struct sockaddr_in server_addr;
memset(&server_addr, 0, sizeof(server_addr));
server_addr.sin_family = AF_INET;
server_addr.sin_port = htons(PORT);
inet_pton(AF_INET, "127.0.0.1", &server_addr.sin_addr);

connect(sock_fd, (struct sockaddr *)&server_addr, sizeof(server_addr));
```

`connect()` จะทำการ **TCP 3-way Handshake** กับ Server เบื้องหลังให้อัตโนมัติ (SYN → SYN-ACK
→ ACK) โดยที่โปรแกรมเมอร์ไม่ต้องยุ่งกับรายละเอียดนี้เอง ถ้า Server ยังไม่ได้ `listen()` อยู่ที่
Port นั้น หรือ Port/IP ผิด จะได้ error `ECONNREFUSED` ("Connection refused") กลับมาทันที

หลังจาก `connect()` สำเร็จ `sock_fd` ตัวเดียวนี้แหละที่ใช้ทั้ง `send()` และ `recv()` คุยกับ
Server ไปจนจบการสนทนา แล้วค่อย `close()` เมื่อเสร็จภารกิจ

```
เปรียบเทียบภาพรวม Server vs Client:

Server:  socket() -> bind() -> listen() -> accept() -> recv()/send() -> close()
Client:  socket() -----------------------> connect() -> send()/recv() -> close()
```

---

## 33.6 สร้าง TCP Echo Server ตัวเต็ม (Step 262)

"Echo Server" คือ Server ที่ง่ายที่สุดในการฝึกเข้าใจ Socket: รับข้อความอะไรมาก็ส่งข้อความ
เดิมกลับไป ไม่มี Business Logic ซับซ้อน แต่ครอบคลุมทุกขั้นตอนของ TCP Server ที่เรียนมา

```c
/* tcp_server.c - TCP Echo Server (รับ 1 client ต่อครั้ง) */
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <sys/socket.h>
#include <netinet/in.h>

#define PORT        5500
#define BACKLOG     5
#define BUFFER_SIZE 1024

int main(void) {
    int server_fd = socket(AF_INET, SOCK_STREAM, 0);
    if (server_fd < 0) {
        perror("socket");
        exit(EXIT_FAILURE);
    }

    /* อนุญาตให้ bind port ซ้ำได้ทันทีหลังปิดโปรแกรม (แก้ปัญหา "Address already in use") */
    int opt = 1;
    if (setsockopt(server_fd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt)) < 0) {
        perror("setsockopt");
        exit(EXIT_FAILURE);
    }

    struct sockaddr_in server_addr;
    memset(&server_addr, 0, sizeof(server_addr));
    server_addr.sin_family = AF_INET;
    server_addr.sin_addr.s_addr = INADDR_ANY;   /* รับทุก network interface */
    server_addr.sin_port = htons(PORT);          /* แปลง host byte order -> network byte order */

    if (bind(server_fd, (struct sockaddr *)&server_addr, sizeof(server_addr)) < 0) {
        perror("bind");
        close(server_fd);
        exit(EXIT_FAILURE);
    }

    if (listen(server_fd, BACKLOG) < 0) {
        perror("listen");
        close(server_fd);
        exit(EXIT_FAILURE);
    }

    printf("[server] กำลังฟังที่ port %d ...\n", PORT);

    struct sockaddr_in client_addr;
    socklen_t client_len = sizeof(client_addr);
    int client_fd = accept(server_fd, (struct sockaddr *)&client_addr, &client_len);
    if (client_fd < 0) {
        perror("accept");
        close(server_fd);
        exit(EXIT_FAILURE);
    }

    char client_ip[INET_ADDRSTRLEN];
    inet_ntop(AF_INET, &client_addr.sin_addr, client_ip, sizeof(client_ip));
    printf("[server] client เชื่อมต่อจาก %s:%d\n", client_ip, ntohs(client_addr.sin_port));

    char buffer[BUFFER_SIZE];
    for (;;) {
        ssize_t bytes_received = recv(client_fd, buffer, sizeof(buffer) - 1, 0);
        if (bytes_received < 0) {
            perror("recv");
            break;
        }
        if (bytes_received == 0) {
            printf("[server] client ปิดการเชื่อมต่อ\n");
            break;
        }
        buffer[bytes_received] = '\0';
        printf("[server] ได้รับ: %s\n", buffer);

        /* ส่งข้อความเดิมกลับไป (echo) - ต้องระวัง partial send */
        ssize_t total_sent = 0;
        while (total_sent < bytes_received) {
            ssize_t sent = send(client_fd, buffer + total_sent,
                                 (size_t)(bytes_received - total_sent), 0);
            if (sent < 0) {
                perror("send");
                break;
            }
            total_sent += sent;
        }
    }

    close(client_fd);
    close(server_fd);
    return 0;
}
```

### อธิบายจุดสำคัญเพิ่มเติม

- **`SO_REUSEADDR`**: ถ้าไม่ใส่ตัวเลือกนี้ เวลาปิด Server แล้วรันใหม่ทันที อาจได้ error
  `EADDRINUSE` เพราะ Kernel ยังจอง Port นั้นไว้ในสถานะ `TIME_WAIT` ของ Connection เก่าอยู่
  (เป็นกลไกของ TCP ที่รอให้แน่ใจว่า Packet ที่ค้างอยู่ในเครือข่ายหมดไปจริงๆ ก่อน) ในระหว่าง
  พัฒนา/ทดสอบ เราจึงเปิด `SO_REUSEADDR` เพื่อข้ามข้อจำกัดนี้ไป (Production จริงก็นิยมเปิดไว้
  เช่นกันสำหรับกรณี Restart Service)
- **`recv()` คืนค่า 0**: หมายถึง Client **ปิดการเชื่อมต่ออย่างสุภาพ** (ส่ง FIN packet มา)
  ไม่ใช่ error — ต้องแยกแยะให้ออกจากกรณี `recv()` คืนค่า `-1` ที่หมายถึงเกิด error จริงๆ
- **`buffer[bytes_received] = '\0';`**: `recv()` **ไม่ได้ใส่ null terminator ให้อัตโนมัติ**
  เพราะข้อมูลที่รับมาอาจเป็น Binary Data ไม่ใช่ String เสมอไป เราต้องใส่เองถ้าต้องการใช้เป็น
  C-String และต้องเผื่อขนาด buffer ไว้ 1 byte เสมอ (`sizeof(buffer) - 1` ตอนเรียก `recv`)

---

## 33.7 สร้าง TCP Echo Client ตัวเต็ม + ทดสอบร่วมกัน (Step 263)

```c
/* tcp_client.c - TCP Echo Client */
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <sys/socket.h>
#include <netinet/in.h>

#define SERVER_IP   "127.0.0.1"
#define PORT        5500
#define BUFFER_SIZE 1024

int main(int argc, char *argv[]) {
    int sock_fd = socket(AF_INET, SOCK_STREAM, 0);
    if (sock_fd < 0) {
        perror("socket");
        exit(EXIT_FAILURE);
    }

    struct sockaddr_in server_addr;
    memset(&server_addr, 0, sizeof(server_addr));
    server_addr.sin_family = AF_INET;
    server_addr.sin_port = htons(PORT);

    if (inet_pton(AF_INET, SERVER_IP, &server_addr.sin_addr) <= 0) {
        fprintf(stderr, "ที่อยู่ IP ไม่ถูกต้อง: %s\n", SERVER_IP);
        exit(EXIT_FAILURE);
    }

    if (connect(sock_fd, (struct sockaddr *)&server_addr, sizeof(server_addr)) < 0) {
        perror("connect");
        exit(EXIT_FAILURE);
    }

    printf("[client] เชื่อมต่อสำเร็จ\n");

    const char *message = (argc > 1) ? argv[1] : "Hello from client";
    size_t msg_len = strlen(message);

    ssize_t total_sent = 0;
    while ((size_t)total_sent < msg_len) {
        ssize_t sent = send(sock_fd, message + total_sent, msg_len - (size_t)total_sent, 0);
        if (sent < 0) {
            perror("send");
            exit(EXIT_FAILURE);
        }
        total_sent += sent;
    }
    printf("[client] ส่งแล้ว: %s\n", message);

    char buffer[BUFFER_SIZE];
    ssize_t bytes_received = recv(sock_fd, buffer, sizeof(buffer) - 1, 0);
    if (bytes_received < 0) {
        perror("recv");
        exit(EXIT_FAILURE);
    }
    buffer[bytes_received] = '\0';
    printf("[client] ได้รับ echo: %s\n", buffer);

    close(sock_fd);
    return 0;
}
```

### คอมไพล์และรันทดสอบร่วมกันจริง

เปิด Terminal 2 หน้าต่าง (หรือ 2 tab): หน้าต่างแรกรัน Server ค้างไว้ หน้าต่างที่สองรัน Client

**Terminal 1 (Server):**

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 tcp_server.c -o tcp_server
./tcp_server
```

```
[server] กำลังฟังที่ port 5500 ...
```

(โปรแกรมจะค้างรออยู่ตรงนี้จนกว่าจะมี Client เชื่อมต่อเข้ามา — เป็นพฤติกรรมปกติของ `accept()`)

**Terminal 2 (Client):**

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 tcp_client.c -o tcp_client
./tcp_client "Hello, TCP World!"
```

ผลลัพธ์ฝั่ง Client:

```
[client] เชื่อมต่อสำเร็จ
[client] ส่งแล้ว: Hello, TCP World!
[client] ได้รับ echo: Hello, TCP World!
```

ผลลัพธ์ฝั่ง Server (Terminal 1 ที่รอค้างอยู่):

```
[server] กำลังฟังที่ port 5500 ...
[server] client เชื่อมต่อจาก 127.0.0.1:41068
[server] ได้รับ: Hello, TCP World!
[server] client ปิดการเชื่อมต่อ
```

(เลข Port ฝั่ง Client เช่น `41068` เป็น **Ephemeral Port** ที่ OS สุ่มเลือกให้อัตโนมัติเวลา
เรียก `connect()` โดยไม่ต้อง `bind()` เอง — Client ไม่จำเป็นต้องมี Port ตายตัวเหมือน Server
เพราะเป็นฝ่ายเริ่มการเชื่อมต่อ ไม่ใช่ฝ่ายรอ)

หลังจาก Client `close(sock_fd)` เซิร์ฟเวอร์จะเห็น `recv()` คืนค่า `0` (client ปิดการเชื่อมต่อ)
แล้ว loop จะ `break` ออกมา ปิด `client_fd` และ `server_fd` แล้วจบโปรแกรม (เพราะโค้ดตัวอย่างนี้
ออกแบบให้รับได้แค่ 1 Client แล้วจบ — ถ้าต้องการรับ Client รายต่อไปเรื่อยๆ ต้องเอาส่วน
`accept()` ไปไว้ใน loop ครอบอีกชั้น ซึ่งจะฝึกในแบบฝึกหัดท้ายบทข้อ 2)

---

## 33.8 Partial Send/Recv และการจัดการ Error ใน Socket (Step 264)

หนึ่งในความเข้าใจผิดที่พบบ่อยที่สุดของมือใหม่คือคิดว่า `send()`/`recv()` หนึ่งครั้งจะส่ง/รับ
ข้อมูลได้ครบตามที่ขอเสมอ — **ความจริงไม่ใช่แบบนั้น** เพราะ TCP มองข้อมูลเป็น **Byte Stream**
ต่อเนื่อง ไม่ใช่ "ข้อความ" ที่มีขอบเขตชัดเจนในตัว Protocol เอง

### ทำไมถึงเกิด Partial Send

`send()` **อาจส่งได้น้อยกว่า** ที่ขอในครั้งเดียว แล้วคืนค่าจำนวน byte ที่ส่งจริงกลับมา
(ไม่ใช่ error) สาเหตุที่พบบ่อย:

- **Kernel Buffer เต็ม**: Socket มี Send Buffer ภายใน Kernel ขนาดจำกัด ถ้าฝั่งรับอ่านข้อมูล
  ช้ากว่าที่เราส่ง Buffer อาจเต็มชั่วคราว
- **ข้อมูลก้อนใหญ่**: ถ้าส่งข้อมูลขนาดหลาย MB ในครั้งเดียว Kernel อาจแบ่งส่งเป็นหลายรอบ
  ภายใน โดยคืนค่าจำนวนที่ส่งได้ในรอบนั้นๆ กลับมาก่อน

วิธีแก้คือ **loop ส่งจนกว่าจะครบ** ตามที่แสดงในโค้ดหัวข้อ 33.6 และ 33.7:

```c
ssize_t total_sent = 0;
while (total_sent < (ssize_t)msg_len) {
    ssize_t sent = send(sock_fd, message + total_sent, msg_len - (size_t)total_sent, 0);
    if (sent < 0) {
        perror("send");
        break;
    }
    total_sent += sent;
}
```

### ทำไมถึงเกิด Partial Recv

ในทางกลับกัน `recv()` ก็ **อาจได้ข้อมูลน้อยกว่า** ที่ขอในครั้งเดียวเช่นกัน แม้ฝั่งส่งจะส่งมา
ครบในครั้งเดียวก็ตาม เพราะ Network Packet อาจถูกแบ่งเป็นหลายก้อนระหว่างทาง (Fragmentation)
หรือ Kernel Buffer ฝั่งรับยังไม่มีข้อมูลมาครบตามที่ Application ขออ่าน ปัญหานี้ยิ่งเห็นชัดถ้า
ข้อความที่ส่งมีขนาดใหญ่กว่า Buffer ที่ใช้อ่านในแต่ละรอบ

**ปัญหาที่ลึกกว่านั้น**: เพราะ TCP เป็น Byte Stream ไม่มีขอบเขตข้อความในตัว Protocol ถ้า
Client ส่ง 2 ข้อความติดกันเร็วมาก (`send("AAA"); send("BBB");`) ฝั่ง Server อาจได้รับเป็น
`recv()` ครั้งเดียวที่มีค่า `"AAABBB"` รวมกัน หรืออาจถูกแบ่งเป็น `"AA"` แล้วตามด้วย `"ABBB"`
ก็ได้ — **ไม่มีการรับประกันว่า 1 ครั้งของ `send()` จะตรงกับ 1 ครั้งของ `recv()` เสมอไป**

```
ฝั่งส่ง:   send("AAA")   send("BBB")
                │             │
                ▼             ▼
         ┌─────────────────────────┐
         │   TCP Byte Stream       │   <- ไม่มีขอบเขตข้อความ รวมเป็นสายเดียว
         │   A A A B B B           │
         └─────────────────────────┘
                │
                ▼
ฝั่งรับ:   recv() อาจได้ "AAABBB" ทีเดียว
           หรือ "AA" แล้วตามด้วย "ABBB"
           หรือรูปแบบอื่นที่ไม่แน่นอน!
```

ในการออกแบบ Protocol ระดับ Application จริง (เช่น HTTP) จึงต้องมีวิธี**กำหนดขอบเขตข้อความ**
เอง เช่น ใส่ตัวคั่น (`\n` หรือ `\r\n\r\n`) หรือใส่ความยาวข้อมูลไว้ที่ header ก่อนตัวข้อมูลจริง
(**Length-Prefixed Framing**) — เรื่องนี้จะเรียนเจาะลึกในโปรเจกต์ TCP Chat Server (Part 35)
สำหรับ Part นี้เราจึงจงใจออกแบบ Echo Server/Client ให้ส่ง-รับกันแค่ 1 รอบต่อการเชื่อมต่อ
เพื่อไม่ต้องแก้ปัญหา Framing ตั้งแต่ตอนนี้

### Error Handling พื้นฐานของ Socket API

ทุกฟังก์ชัน Socket หลัก (`socket`, `bind`, `listen`, `accept`, `connect`, `send`, `recv`)
คืนค่า `-1` เมื่อล้มเหลว และตั้งค่า `errno` ไว้บอกสาเหตุ ควรเช็คทุกครั้งและใช้ `perror()`
รายงาน error ให้อ่านง่าย (ตามที่ทำในโค้ดทุกตัวอย่างของ Part นี้):

| ฟังก์ชัน | Error ที่พบบ่อย | ความหมาย |
|---|---|---|
| `bind()` | `EADDRINUSE` | Port ถูกใช้อยู่แล้ว (แก้ด้วย `SO_REUSEADDR` หรือรอ/เปลี่ยน Port) |
| `connect()` | `ECONNREFUSED` | ปลายทางไม่มี Server ฟังอยู่ที่ Port นั้น (ยังไม่ได้รัน Server หรือ Port ผิด) |
| `connect()` | `ETIMEDOUT` | เชื่อมต่อไม่สำเร็จภายในเวลาที่กำหนด (เครือข่ายมีปัญหา หรือ Firewall บล็อก) |
| `send()`/`recv()` | `ECONNRESET` | อีกฝั่งปิดการเชื่อมต่อกะทันหัน (เช่น โปรแกรม crash) |
| `accept()` | `EMFILE` | เปิด File Descriptor มากเกิน limit ของ process |

---

## ข้อผิดพลาดที่พบบ่อย (Common Pitfalls)

1. **ลืมใส่ `htons()` ตอนตั้งค่า Port** — ถ้าเขียน `server_addr.sin_port = PORT;` ตรงๆ โดยไม่
   ผ่าน `htons()` บนเครื่อง Little-Endian (เช่น x86/x86-64 ทั่วไป) จะได้ Port ผิดโดยสิ้นเชิง
   (เช่น Port 5500 อาจกลายเป็น Port อื่นที่ไม่มีใครฟังอยู่) ทำให้ `bind()`/`connect()` ใช้
   ไม่ได้อย่างเงียบๆ (compile ผ่าน ไม่มี warning แต่พฤติกรรมผิด)
2. **ไม่ตรวจสอบค่าที่คืนจาก `recv()`** — สับสนระหว่างค่า `0` (client ปิดการเชื่อมต่ออย่าง
   สุภาพ) กับค่า `-1` (เกิด error) ถ้าไม่แยกให้ถูก loop อาจวนไม่รู้จบหรือ crash
3. **ไม่ใส่ null terminator หลัง `recv()`** — `recv()` ไม่ทำให้อัตโนมัติ ถ้านำ buffer ไปใช้
   เป็น C-String ต่อ (เช่น `printf("%s", buffer)`) โดยไม่ใส่ `buffer[n] = '\0';` ก่อน จะได้
   Undefined Behavior (อ่านเลยขอบเขตข้อมูลจริงไปจนกว่าจะเจอ byte 0 ในหน่วยความจำโดยบังเอิญ)
4. **คิดว่า `send()`/`recv()` 1 ครั้งเท่ากับ 1 "ข้อความ" เสมอ** — อย่างที่อธิบายในหัวขัอ 33.8
   ต้องเขียน Application Protocol ที่มีขอบเขตข้อความชัดเจนเองเสมอ ถ้าข้อมูลมีความซับซ้อนกว่า
   การส่ง-รับครั้งเดียวจบ
5. **ไม่เปิด `SO_REUSEADDR`** — ทำให้ระหว่างพัฒนา ต้องรอสถานะ `TIME_WAIT` หมดอายุ (มักหลายสิบ
   วินาที) ก่อนจะรัน Server ตัวใหม่ที่ Port เดิมได้ทุกครั้งที่แก้โค้ดแล้วรันใหม่
6. **ลืม `close()` Socket** — ทั้ง `server_fd` และ `client_fd` ต้องปิดเมื่อเลิกใช้งาน ไม่งั้น
   จะรั่วไหล File Descriptor (คล้าย Memory Leak) จนถึงจุดที่โปรแกรมเปิด Socket ใหม่ไม่ได้อีก
   เมื่อรันเป็นเวลานาน (โดยเฉพาะ Server ที่รับ Client จำนวนมากต่อเนื่อง)
7. **สับสนระหว่าง `server_fd` กับ `client_fd`** — พยายามใช้ `server_fd` ในการ `send()`/`recv()`
   คุยกับ Client โดยตรง (ที่ถูกต้องคือต้องใช้ `client_fd` ที่ได้จาก `accept()` เท่านั้น
   `server_fd` มีหน้าที่แค่ `accept()` การเชื่อมต่อใหม่ๆ เท่านั้น)

---

## แบบฝึกหัดท้ายบท

1. แก้ไข `tcp_server.c` ให้พิมพ์ Port ของ Client ที่เชื่อมต่อเข้ามาด้วย `%d` **โดยไม่ผ่าน**
   `ntohs()` แล้วเปรียบเทียบผลลัพธ์กับตอนที่ผ่าน `ntohs()` อธิบายว่าทำไมตัวเลขถึงต่างกันมาก
2. แก้ไข `tcp_server.c` ให้ครอบส่วน `accept()` ไว้ใน `while (1)` loop เพื่อรับ Client ได้
   ต่อเนื่องหลายรายทีละคน (accept รายหนึ่งจนจบการสนทนา แล้วค่อย accept รายถัดไป) โดยยังคง
   `server_fd` ตัวเดิมไว้ตลอด
3. เขียน Client ตัวใหม่ที่เชื่อมต่อไปยัง Server แล้วส่งข้อความ **5 ข้อความติดกัน** (แต่ละ
   ข้อความคั่นด้วย `\n`) จากนั้นสังเกตว่าฝั่ง Server ที่ `recv()` เพียงครั้งเดียวได้รับข้อมูล
   มาเป็นก้อนเดียวกันทั้งหมดหรือแยกเป็นหลายครั้ง (ผลลัพธ์อาจไม่แน่นอนในแต่ละรอบที่รัน)
4. แก้ไข `tcp_client.c` ให้รับ Argument ที่ 2 เป็น Port แทนที่จะ hardcode `PORT` ไว้ในโค้ด
   ใช้ `atoi()` แปลง string เป็นตัวเลข แล้วทดสอบเชื่อมต่อ Server ที่ Port อื่นที่ไม่ใช่ 5500
5. เขียนโปรแกรมที่พยายาม `connect()` ไปยัง Port ที่**ไม่มี Server ฟังอยู่** (เช่น Port 9999
   ที่ไม่ได้รันอะไรไว้) แล้วจับ error ด้วย `perror()` สังเกตข้อความ error ที่ได้ (ควรเห็น
   "Connection refused")
6. ออกแบบและ implement Protocol ง่ายๆ ที่ใส่ **ความยาวข้อความ 4 byte** (เป็นตัวเลข binary)
   ไว้หน้าข้อมูลจริงเสมอ (Length-Prefixed Framing) แล้วแก้ทั้ง Server และ Client ให้อ่าน
   ความยาวก่อน แล้วค่อยอ่านข้อมูลตามความยาวนั้นให้ครบ (เป็นการแก้ปัญหา Partial Recv อย่างเป็น
   ระบบ ซึ่งเป็นรากฐานสำคัญของโปรเจกต์ TCP Chat ใน Part 35)

### แนวทางเฉลยข้อ 1

```c
/* server_port_demo.c - เฉลยข้อ 1: เปรียบเทียบ port ก่อน-หลังผ่าน ntohs() */
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <sys/socket.h>
#include <netinet/in.h>

#define PORT 5501

int main(void) {
    int server_fd = socket(AF_INET, SOCK_STREAM, 0);
    int opt = 1;
    setsockopt(server_fd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));

    struct sockaddr_in server_addr;
    memset(&server_addr, 0, sizeof(server_addr));
    server_addr.sin_family = AF_INET;
    server_addr.sin_addr.s_addr = INADDR_ANY;
    server_addr.sin_port = htons(PORT);

    bind(server_fd, (struct sockaddr *)&server_addr, sizeof(server_addr));
    listen(server_fd, 5);
    printf("[server] ฟังที่ port %d (รอ client จาก tcp_client.c)\n", PORT);

    struct sockaddr_in client_addr;
    socklen_t client_len = sizeof(client_addr);
    int client_fd = accept(server_fd, (struct sockaddr *)&client_addr, &client_len);

    /* ไม่ผ่าน ntohs(): จะได้ค่าที่ถูกตีความผิด (Network Byte Order ดิบๆ) */
    printf("Port แบบไม่แปลง (ผิด):  %d\n", client_addr.sin_port);
    /* ผ่าน ntohs(): จะได้ค่า port จริงที่มนุษย์อ่านเข้าใจ */
    printf("Port แบบแปลงแล้ว (ถูก): %d\n", ntohs(client_addr.sin_port));

    close(client_fd);
    close(server_fd);
    return 0;
}
```

ผลลัพธ์ตัวอย่าง (ตัวเลขจริงจะต่างกันไปตาม Ephemeral Port ที่ OS สุ่มให้ Client แต่ละครั้ง):

```
[server] ฟังที่ port 5501 (รอ client จาก tcp_client.c)
Port แบบไม่แปลง (ผิด):  12189
Port แบบแปลงแล้ว (ถูก): 47661
```

เหตุผลที่ตัวเลขต่างกันมาก: ค่าที่เก็บใน `sin_port` เป็น 16-bit Big-Endian (Network Byte
Order) เสมอ แต่ CPU สถาปัตยกรรม x86-64 เป็น Little-Endian การพิมพ์ด้วย `%d` ตรงๆ โดยไม่แปลง
คือการเอา byte 2 ตัวมาอ่านสลับตำแหน่งกัน ได้ตัวเลขที่ผิดไปจากความเป็นจริงโดยสิ้นเชิง

### แนวทางเฉลยข้อ 2

```c
/* tcp_server_loop.c - เฉลยข้อ 2: accept loop รับ client หลายรายต่อเนื่อง (ทีละคน) */
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <sys/socket.h>
#include <netinet/in.h>

#define PORT        5502
#define BACKLOG     5
#define BUFFER_SIZE 1024

int main(void) {
    int server_fd = socket(AF_INET, SOCK_STREAM, 0);
    if (server_fd < 0) { perror("socket"); exit(EXIT_FAILURE); }

    int opt = 1;
    setsockopt(server_fd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));

    struct sockaddr_in server_addr;
    memset(&server_addr, 0, sizeof(server_addr));
    server_addr.sin_family = AF_INET;
    server_addr.sin_addr.s_addr = INADDR_ANY;
    server_addr.sin_port = htons(PORT);

    if (bind(server_fd, (struct sockaddr *)&server_addr, sizeof(server_addr)) < 0) {
        perror("bind"); exit(EXIT_FAILURE);
    }
    if (listen(server_fd, BACKLOG) < 0) { perror("listen"); exit(EXIT_FAILURE); }

    printf("[server] ฟังที่ port %d (รับ client ต่อเนื่องทีละคน กด Ctrl+C เพื่อหยุด)\n", PORT);

    /* หัวใจของเฉลยข้อนี้: ครอบ accept() ไว้ใน for(;;) ชั้นนอก
     * server_fd ตัวเดิมถูกใช้ accept() ซ้ำได้เรื่อยๆ ไม่มีวันหมดอายุ */
    for (;;) {
        struct sockaddr_in client_addr;
        socklen_t client_len = sizeof(client_addr);
        int client_fd = accept(server_fd, (struct sockaddr *)&client_addr, &client_len);
        if (client_fd < 0) {
            perror("accept");
            continue; /* accept ล้มเหลวรายนี้ ไม่ใช่เหตุผลให้ทั้ง server ตาย ลองรายถัดไปต่อ */
        }

        char client_ip[INET_ADDRSTRLEN];
        inet_ntop(AF_INET, &client_addr.sin_addr, client_ip, sizeof(client_ip));
        printf("[server] client ใหม่จาก %s:%d\n", client_ip, ntohs(client_addr.sin_port));

        /* loop ชั้นในคุยกับ client รายนี้จนกว่าจะปิดการเชื่อมต่อ */
        char buffer[BUFFER_SIZE];
        for (;;) {
            ssize_t n = recv(client_fd, buffer, sizeof(buffer) - 1, 0);
            if (n <= 0) {
                if (n < 0) perror("recv");
                printf("[server] client หลุดการเชื่อมต่อ\n");
                break;
            }
            buffer[n] = '\0';
            printf("[server] ได้รับ: %s\n", buffer);

            ssize_t total_sent = 0;
            while (total_sent < n) {
                ssize_t sent = send(client_fd, buffer + total_sent,
                                     (size_t)(n - total_sent), 0);
                if (sent < 0) { perror("send"); break; }
                total_sent += sent;
            }
        }
        close(client_fd); /* ปิดเฉพาะ client_fd ของรายนี้ server_fd ยังใช้งานต่อได้ */
    }

    close(server_fd); /* ไม่มีวันมาถึงจุดนี้ในเดโมนี้ เพราะ loop ไม่จบเอง (หยุดด้วย Ctrl+C) */
    return 0;
}
```

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 tcp_server_loop.c -o tcp_server_loop
./tcp_server_loop &   # รันเป็น background เพื่อทดสอบหลาย client ติดกัน
sleep 0.3
./tcp_client "first client"
./tcp_client "second client"
```

ผลลัพธ์ฝั่ง Server (สังเกตว่า `server_fd` ตัวเดิมรับ Client ได้ต่อเนื่อง 2 รายโดยไม่ต้อง
รันโปรแกรมใหม่เลย):

```
[server] ฟังที่ port 5502 (รับ client ต่อเนื่องทีละคน กด Ctrl+C เพื่อหยุด)
[server] client ใหม่จาก 127.0.0.1:42292
[server] ได้รับ: first client
[server] client หลุดการเชื่อมต่อ
[server] client ใหม่จาก 127.0.0.1:42300
[server] ได้รับ: second client
[server] client หลุดการเชื่อมต่อ
```

ข้อสังเกตสำคัญ: โซลูชันนี้ยังคง**รับได้ทีละ 1 Client เท่านั้นในเวลาเดียวกัน** — ถ้า Client
รายที่สองพยายามเชื่อมต่อเข้ามา**ระหว่าง**ที่ Server กำลังคุยกับรายแรกอยู่ (ยังไม่ปิดการ
เชื่อมต่อ) รายที่สองจะต้องรอในคิว `BACKLOG` จนกว่า Server จะว่างมา `accept()` ใหม่ ปัญหานี้
จะได้รับการแก้ไขอย่างแท้จริงด้วย I/O Multiplexing ใน Part 34

### แนวทางเฉลยข้อ 5

```c
/* connection_refused_demo.c - เฉลยข้อ 5: ทดสอบ connect ไป port ที่ไม่มี server ฟังอยู่ */
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <sys/socket.h>
#include <netinet/in.h>

#define UNUSED_PORT 9999

int main(void) {
    int sock_fd = socket(AF_INET, SOCK_STREAM, 0);
    if (sock_fd < 0) {
        perror("socket");
        exit(EXIT_FAILURE);
    }

    struct sockaddr_in addr;
    memset(&addr, 0, sizeof(addr));
    addr.sin_family = AF_INET;
    addr.sin_port = htons(UNUSED_PORT);
    inet_pton(AF_INET, "127.0.0.1", &addr.sin_addr);

    printf("กำลังพยายามเชื่อมต่อไปยัง port %d ที่ไม่มี server ฟังอยู่...\n", UNUSED_PORT);

    if (connect(sock_fd, (struct sockaddr *)&addr, sizeof(addr)) < 0) {
        perror("connect ล้มเหลวตามที่คาดไว้");
        close(sock_fd);
        return EXIT_FAILURE;
    }

    printf("เชื่อมต่อสำเร็จ (ไม่ควรมาถึงจุดนี้ในการทดสอบนี้)\n");
    close(sock_fd);
    return EXIT_SUCCESS;
}
```

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 connection_refused_demo.c -o connection_refused_demo
./connection_refused_demo
```

ผลลัพธ์ที่คาดหวัง:

```
กำลังพยายามเชื่อมต่อไปยัง port 9999 ที่ไม่มี server ฟังอยู่...
connect ล้มเหลวตามที่คาดไว้: Connection refused
```

ข้อความ "Connection refused" มาจาก `perror()` ที่แปลงค่า `errno` (`ECONNREFUSED`) เป็น
ข้อความที่มนุษย์อ่านเข้าใจให้อัตโนมัติ — สาเหตุคือ Kernel ของเครื่องปลายทาง (ในกรณีนี้คือ
เครื่องตัวเอง เพราะใช้ `127.0.0.1`) ตอบกลับด้วย TCP RST Packet ทันทีเมื่อไม่มีโปรแกรมใด
`listen()` อยู่ที่ Port นั้นเลย

---

## สรุปท้ายบท

ใน Part นี้เราได้:

- เข้าใจ Network Programming Model แบบ Client-Server และบทบาทของ IP Address/Port
- เข้าใจว่า Socket คือ Abstraction ที่ต่อยอดจากปรัชญา "Everything is a file" ของ Unix
- ใช้ `struct sockaddr_in` และฟังก์ชันแปลง Byte Order (`htons`/`htonl`/`ntohs`/`ntohl`)
  ได้อย่างถูกต้อง พร้อมเข้าใจสาเหตุที่ต้องมีการแปลงนี้ (Endianness)
- เขียน TCP Server ตามลำดับ `socket()` → `bind()` → `listen()` → `accept()` และ TCP Client
  ตามลำดับ `socket()` → `connect()` ได้เองตั้งแต่ต้นจนจบ
- สร้าง TCP Echo Server/Client ตัวเต็ม ทดสอบรันร่วมกันจริงผ่าน `127.0.0.1` ได้สำเร็จ
- เข้าใจปัญหา Partial Send/Recv ที่มาจากธรรมชาติของ TCP Byte Stream และรู้วิธีจัดการเบื้องต้น

Server ที่เขียนใน Part นี้ยังมีข้อจำกัดใหญ่: รับ Client ได้ **ทีละ 1 รายเท่านั้น** ถ้า Client
รายที่สองพยายามเชื่อมต่อเข้ามาระหว่างที่ Server กำลังคุยกับรายแรกอยู่ จะต้องรอในคิว (Backlog)
จนกว่า Server จะว่างเรียก `accept()` ใหม่ — ซึ่งไม่เพียงพอสำหรับ Server ในโลกจริงที่ต้องรองรับ
ผู้ใช้จำนวนมากพร้อมกัน ใน **Part 34** เราจะเรียนรู้ **UDP Socket** (ทางเลือกที่ไม่ต้องเชื่อมต่อ
ก่อน) และเทคนิค **I/O Multiplexing** (`select()`, `poll()`, `epoll()`) ที่ทำให้ Server ตัวเดียว
รองรับหลาย Client พร้อมกันได้โดยไม่ต้องใช้หลาย Thread

**ต่อไป:** [Part 34 — Socket ขั้นสูง (UDP/select/poll/epoll)](./part-034-sockets-advanced.md)
