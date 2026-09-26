# Part 34: Socket ขั้นสูง (UDP/select/poll/epoll) (Step 265–272)

> Module C — Systems Programming ด้วย C บน Linux | Part 34 จาก 125
> Step ที่ครอบคลุมใน Part นี้: Step 265–272
> Part ก่อนหน้า: [Part 33 — Socket Programming เบื้องต้น (TCP)](./part-033-sockets-basics.md) | Part ถัดไป: [Part 35 — โปรเจกต์ TCP Chat Server/Client](./part-035-tcp-chat-project.md)

## เป้าหมายการเรียนรู้ (Learning Objectives)

หลังจบ Part นี้ ผู้เรียนจะสามารถ:

1. อธิบายและใช้งาน **UDP Socket** (`SOCK_DGRAM`) ด้วย `sendto()`/`recvfrom()` ได้ พร้อม
   เปรียบเทียบข้อดี-ข้อเสียกับ TCP ได้อย่างชัดเจน
2. อธิบายว่า **I/O Multiplexing** คืออะไร และทำไมการรองรับหลาย Connection พร้อมกันจึงไม่
   จำเป็นต้องใช้หลาย Thread หรือหลาย Process เสมอไป
3. ใช้ `select()` เขียน TCP Server ที่รองรับหลาย Client พร้อมกันได้จริงในโปรเซสเดียว
   Thread เดียว
4. เปรียบเทียบ `poll()` กับ `select()` และอธิบายข้อจำกัดของ `select()` ที่ `poll()` แก้ได้
5. อธิบายแนวคิดของ `epoll()` รวมถึงความแตกต่างระหว่าง **Level-Triggered** และ
   **Edge-Triggered** เบื้องต้น และเหตุผลที่ Server ประสิทธิภาพสูงบน Linux นิยมใช้ `epoll()`
6. สร้าง TCP Server ที่รองรับหลาย Client พร้อมกันด้วย `select()` ตัวเต็ม ทดสอบด้วย Client
   หลายตัวพร้อมกันได้จริง

---

## 34.1 UDP Socket: Connectionless Communication (Step 265)

ใน Part 33 เราเรียนรู้ TCP ซึ่งเป็น Protocol แบบ **Connection-Oriented** ที่ต้อง `connect()`
"จับมือ" กันก่อนถึงจะส่งข้อมูลได้ Part นี้จะเริ่มจาก Protocol อีกแบบที่ตรงข้ามกันโดยสิ้นเชิง:
**UDP (User Datagram Protocol)** ซึ่งเป็นแบบ **Connectionless**

สร้าง UDP Socket ด้วยการเปลี่ยน `SOCK_STREAM` เป็น `SOCK_DGRAM`:

```c
int sock_fd = socket(AF_INET, SOCK_DGRAM, 0); /* SOCK_DGRAM = UDP */
```

ความแตกต่างสำคัญที่สุดคือ **ไม่มี `listen()` และไม่มี `accept()`/`connect()` (ในการใช้งาน
พื้นฐาน)** — แต่ละ Packet (เรียกว่า **Datagram**) ถูกส่งแบบเอกเทศ ไม่ผูกกับ "การเชื่อมต่อ" ใดๆ
ฟังก์ชันหลักที่ใช้แทน `send()`/`recv()` คือ `sendto()`/`recvfrom()` ซึ่งต้องระบุที่อยู่ปลายทาง
(หรือรับที่อยู่ต้นทาง) ในทุกครั้งที่เรียก เพราะไม่มี "การเชื่อมต่อ" ที่จดจำปลายทางไว้ให้

```c
ssize_t sendto(int sockfd, const void *buf, size_t len, int flags,
                const struct sockaddr *dest_addr, socklen_t addrlen);

ssize_t recvfrom(int sockfd, void *buf, size_t len, int flags,
                  struct sockaddr *src_addr, socklen_t *addrlen);
```

### UDP Echo Server และ Client ตัวเต็ม

```c
/* udp_server.c - UDP Echo Server (connectionless) */
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <sys/socket.h>
#include <netinet/in.h>

#define PORT        5900
#define BUFFER_SIZE 1024

int main(void) {
    int sock_fd = socket(AF_INET, SOCK_DGRAM, 0); /* SOCK_DGRAM = UDP */
    if (sock_fd < 0) {
        perror("socket");
        exit(EXIT_FAILURE);
    }

    struct sockaddr_in server_addr;
    memset(&server_addr, 0, sizeof(server_addr));
    server_addr.sin_family = AF_INET;
    server_addr.sin_addr.s_addr = INADDR_ANY;
    server_addr.sin_port = htons(PORT);

    /* UDP ยังคงต้อง bind() เพื่อระบุ port ที่จะรอรับข้อมูล แม้จะไม่มี listen()/accept() */
    if (bind(sock_fd, (struct sockaddr *)&server_addr, sizeof(server_addr)) < 0) {
        perror("bind");
        exit(EXIT_FAILURE);
    }

    printf("[udp-server] กำลังรอข้อมูลที่ port %d ...\n", PORT);

    char buffer[BUFFER_SIZE];
    struct sockaddr_in client_addr;
    socklen_t client_len = sizeof(client_addr);

    for (int i = 0; i < 3; i++) { /* รับ 3 packet แล้วจบ (เดโม) */
        ssize_t n = recvfrom(sock_fd, buffer, sizeof(buffer) - 1, 0,
                              (struct sockaddr *)&client_addr, &client_len);
        if (n < 0) {
            perror("recvfrom");
            break;
        }
        buffer[n] = '\0';

        char client_ip[INET_ADDRSTRLEN];
        inet_ntop(AF_INET, &client_addr.sin_addr, client_ip, sizeof(client_ip));
        printf("[udp-server] ได้รับจาก %s:%d: %s\n",
               client_ip, ntohs(client_addr.sin_port), buffer);

        sendto(sock_fd, buffer, (size_t)n, 0,
               (struct sockaddr *)&client_addr, client_len);
    }

    close(sock_fd);
    return 0;
}
```

```c
/* udp_client.c - UDP Echo Client */
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <sys/socket.h>
#include <netinet/in.h>

#define SERVER_IP "127.0.0.1"
#define PORT      5900

int main(int argc, char *argv[]) {
    int sock_fd = socket(AF_INET, SOCK_DGRAM, 0);
    if (sock_fd < 0) {
        perror("socket");
        exit(EXIT_FAILURE);
    }

    struct sockaddr_in server_addr;
    memset(&server_addr, 0, sizeof(server_addr));
    server_addr.sin_family = AF_INET;
    server_addr.sin_port = htons(PORT);
    inet_pton(AF_INET, SERVER_IP, &server_addr.sin_addr);

    const char *message = (argc > 1) ? argv[1] : "Hello UDP";

    /* UDP ไม่ต้อง connect() ก่อนก็ส่งได้เลยด้วย sendto() */
    sendto(sock_fd, message, strlen(message), 0,
           (struct sockaddr *)&server_addr, sizeof(server_addr));
    printf("[udp-client] ส่งแล้ว: %s\n", message);

    char buffer[1024];
    struct sockaddr_in from_addr;
    socklen_t from_len = sizeof(from_addr);
    ssize_t n = recvfrom(sock_fd, buffer, sizeof(buffer) - 1, 0,
                          (struct sockaddr *)&from_addr, &from_len);
    if (n < 0) {
        perror("recvfrom");
        exit(EXIT_FAILURE);
    }
    buffer[n] = '\0';
    printf("[udp-client] ได้รับ echo: %s\n", buffer);

    close(sock_fd);
    return 0;
}
```

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 udp_server.c -o udp_server
gcc -Wall -Wextra -Wpedantic -std=c17 udp_client.c -o udp_client

./udp_server &
sleep 0.3
./udp_client "packet one"
```

ผลลัพธ์:

```
[udp-client] ส่งแล้ว: packet one
[udp-client] ได้รับ echo: packet one
[udp-server] กำลังรอข้อมูลที่ port 5900 ...
[udp-server] ได้รับจาก 127.0.0.1:56968: packet one
```

สังเกตว่า `sendto()` ของฝั่ง Client **ไม่ต้อง `connect()` ก่อนเลย** — แค่ระบุปลายทางตรงในทุก
ครั้งที่ส่งก็เพียงพอ และฝั่ง Server ก็ยังคง `bind()` เหมือน TCP (เพื่อระบุว่าจะรอรับข้อมูลที่
Port ไหน) แต่ **ไม่มี `listen()`/`accept()`** เพราะไม่มีแนวคิดเรื่อง "การเชื่อมต่อ" ใน UDP เลย
Packet แต่ละก้อนที่มาถึงถูกจัดการเป็นเอกเทศ ไม่ขึ้นกับ Packet ก่อนหน้าหรือหลังจากนั้น

---

## 34.2 TCP vs UDP เปรียบเทียบเจาะลึก (Step 266)

| หัวข้อ | TCP | UDP |
|---|---|---|
| การเชื่อมต่อ | ต้อง `connect()`/`accept()` (3-Way Handshake) ก่อน | ไม่ต้องเชื่อมต่อ ส่งได้ทันทีด้วย `sendto()` |
| ความน่าเชื่อถือ | รับประกันข้อมูลถึงครบและไม่ซ้ำ (ส่งซ้ำอัตโนมัติถ้าหาย) | **ไม่รับประกัน** Packet อาจหาย ซ้ำ หรือไม่ถึงเลยก็ได้ |
| ลำดับข้อมูล | รับประกันมาถึงตามลำดับที่ส่งเสมอ | **ไม่รับประกัน** Packet อาจมาถึงไม่ตามลำดับที่ส่ง |
| ขอบเขตข้อความ | Byte Stream ต่อเนื่อง ไม่มีขอบเขตในตัว Protocol (ตามที่เรียนใน Part 33) | แต่ละ `sendto()` คือ 1 Datagram ที่มีขอบเขตชัดเจน ฝั่งรับ `recvfrom()` 1 ครั้งจะได้ Datagram เต็มก้อนเดียว ไม่ปนกัน |
| ความเร็ว/Overhead | ช้ากว่าเล็กน้อยเพราะมี Handshake, Acknowledgement, การจัดลำดับ | เร็วกว่า Overhead ต่ำมาก เพราะไม่มีกลไกยืนยัน/จัดลำดับใดๆ |
| Header ขนาด | 20 byte ขึ้นไป | 8 byte เท่านั้น |
| ใช้งานเมื่อไหร่ | เมื่อ**ความถูกต้องครบถ้วนสำคัญกว่าความเร็ว** เช่น โหลดหน้าเว็บ, โอนไฟล์, Database | เมื่อ**ความเร็ว/Real-time สำคัญกว่าความครบถ้วน 100%** เช่น Video Call, Online Game, DNS Query |
| ตัวอย่างการใช้งานจริง | HTTP/HTTPS, SSH, FTP, Database Connection | DNS, VoIP, Video Streaming, Online Multiplayer Game, DHCP |

### ทำไม UDP ถึง "ไม่รับประกัน" อะไรเลย แต่ยังมีคนใช้

จุดแข็งที่แท้จริงของ UDP ไม่ใช่ความเร็วอย่างเดียว แต่คือ **การควบคุมพฤติกรรมได้เต็มที่**
แอปพลิเคชันบางประเภท เช่น Video Call **ไม่ต้องการ** ให้ Packet ที่มาช้าถูกส่งซ้ำ (เพราะถึงส่ง
มาใหม่ก็สายเกินจะเอาไปแสดงผลทันเวลาอยู่ดี) การรอ Retransmission แบบ TCP กลับทำให้เกิด
อาการ "กระตุก" (lag) มากกว่าการปล่อย Frame ที่หายไปนั้นทิ้งไปเลยแล้วแสดง Frame ถัดไปต่อ —
นี่คือเหตุผลที่ Protocol ระดับสูงอย่าง WebRTC (ใช้ในวิดีโอคอล) เลือกสร้างอยู่บน UDP แทน TCP

```
TCP:  ข้อมูลหาย -> รอส่งซ้ำ -> ล่าช้าแต่ครบถ้วน   (เหมาะกับไฟล์, เว็บเพจ)
UDP:  ข้อมูลหาย -> ปล่อยผ่านไปเลย -> เร็วแต่อาจขาดหาย  (เหมาะกับ real-time)
```

---

## 34.3 สร้าง UDP Echo Server/Client (Step 267)

หัวข้อ 34.1 ได้แสดงโค้ด UDP Echo Server/Client ตัวเต็มไปแล้ว ในหัวข้อนี้เราจะเจาะลึกจุดที่
มือใหม่มักเข้าใจผิดเมื่อทำงานกับ UDP

### ทำไม UDP ยังต้อง bind() ทั้งที่ไม่มี "connection"

`bind()` ใน UDP มีจุดประสงค์เดียวกับ TCP: บอก Kernel ว่า Socket นี้จะ "รอรับข้อมูล" ที่ Port
ไหน ถ้าไม่ `bind()` ฝั่ง Server จะไม่มีทางรู้ได้เลยว่าจะให้ Client ส่งมาหาที่ Port อะไร
(สังเกตว่าฝั่ง Client ใน `udp_client.c` **ไม่ได้ `bind()` เอง** — Kernel จะเลือก Ephemeral
Port ให้อัตโนมัติตอนเรียก `sendto()` ครั้งแรก เหมือนกับที่ทำใน TCP `connect()`)

### 1 Datagram = 1 recvfrom() เสมอ (ต่างจาก TCP ตรงนี้ชัดเจน)

จุดที่ UDP ง่ายกว่า TCP อย่างเห็นได้ชัดคือ **ไม่มีปัญหา Partial Send/Recv** เพราะ UDP รักษา
ขอบเขตของ Datagram ไว้ให้เสมอ — ถ้า `sendto()` ส่งข้อมูล 100 byte ไปในครั้งเดียว ฝั่งรับ
`recvfrom()` ครั้งเดียวก็จะได้ข้อมูล 100 byte นั้นครบถ้วน (หรือไม่ได้เลยถ้า Packet หายไป)
ไม่มีทางได้แค่บางส่วนหรือได้ปนกับ Datagram อื่น — นี่คือข้อดีสำคัญที่ทำให้ Protocol ระดับสูง
บางแบบเลือกออกแบบบน UDP เพื่อความง่ายในการกำหนดขอบเขตข้อความ แม้ต้องแลกกับการไม่มี
การยืนยันความครบถ้วนก็ตาม

### ตัวอย่าง: จำลอง Packet Loss ด้วยการปิด Client ก่อน Server ตอบกลับ

ลองรัน `udp_client` โดยที่**ไม่ได้เปิด** `udp_server` ไว้เลย:

```bash
./udp_client "no one is listening"
```

ผลลัพธ์ที่น่าสนใจ: `sendto()` จะ **สำเร็จเสมอ** (ไม่มี error ทันที) ทั้งที่ไม่มีใครรับข้อมูลนั้น
เลย! เพราะ UDP ไม่มีกลไกยืนยันการรับที่ระดับ Protocol — โปรแกรมจะไปค้างที่ `recvfrom()` (รอ
ข้อมูลตอบกลับที่ไม่มีวันมาถึง) จนกว่าจะถูก Ctrl+C ยกเลิก นี่คือตัวอย่างที่จับต้องได้ของคำว่า
"UDP ไม่รับประกันอะไรเลย" — โปรแกรมที่ใช้ UDP ในโลกจริงจึงมักต้องมี **Timeout** และ **กลไก
ยืนยันของตัวเอง** (เช่น Sequence Number, Acknowledgement) ถ้าต้องการความน่าเชื่อถือระดับหนึ่ง
โดยไม่ต้องใช้ TCP ทั้งหมด

---

## 34.4 I/O Multiplexing คืออะไร ทำไมต้องใช้ (Step 268)

จาก Part 33 เราเห็นข้อจำกัดสำคัญของ TCP Server แบบง่าย: `accept()` และ `recv()` เป็น
**Blocking Call** — ถ้า Server กำลังรอ `recv()` จาก Client รายหนึ่งอยู่ Client รายอื่นที่
พยายามเชื่อมต่อเข้ามาจะต้องรอคิวจนกว่า Server จะว่าง มีวิธีแก้ปัญหานี้อยู่ 3 แนวทางหลัก:

```
แนวทางที่ 1: Multi-Process    - fork() ลูกใหม่ทุกครั้งที่มี client (สิ้นเปลือง memory มาก)
แนวทางที่ 2: Multi-Thread     - สร้าง thread ใหม่ทุกครั้งที่มี client (เรียนใน Part 31-32)
แนวทางที่ 3: I/O Multiplexing - Thread เดียว จัดการหลาย socket พร้อมกัน (หัวข้อนี้)
```

**I/O Multiplexing** คือเทคนิคที่ให้ **1 Thread เดียว** สามารถ "จับตาดู" File Descriptor
(Socket) **หลายตัวพร้อมกัน** และรู้ทันทีว่าตัวไหน "พร้อมทำงาน" (มีข้อมูลให้อ่าน หรือพร้อมให้
เขียน) โดยไม่ต้องเปิด Thread แยกสำหรับ Client แต่ละราย

```
วิธีเดิม (Blocking ตรงๆ):
   recv(client_1) -> บล็อกรอจน client_1 ส่งมา -> ระหว่างนี้ client_2 ต้องรอเฉยๆ

วิธี I/O Multiplexing:
   select(client_1, client_2, client_3, ...) -> บอกทันทีว่า "client_2 มีข้อมูลมาแล้ว!"
   -> เราค่อยไป recv(client_2) แบบไม่บล็อก (รู้อยู่แล้วว่ามีข้อมูลจริง)
```

### ทำไมวิธีนี้ถึงสำคัญ

- **ประหยัดหน่วยความจำ**: ไม่ต้องสร้าง Thread/Process ใหม่ทุกครั้งที่มี Client (แต่ละ Thread
  กิน Stack Memory หลัก MB, แต่ละ Process กินมากกว่านั้นอีก)
- **ไม่มี Overhead ของ Context Switching**: ยิ่งมี Thread เยอะ OS ยิ่งต้องสลับ (context
  switch) ระหว่าง Thread บ่อยขึ้น ซึ่งมี cost ที่ไม่ใช่ศูนย์
- **ไม่ต้องกังวลเรื่อง Race Condition ข้าม Client**: เพราะทำงานใน Thread เดียว ไม่มี Shared
  State ที่ต้องป้องกันด้วย Mutex เหมือนที่เรียนใน Part 32 (ยกเว้นกรณีผสมกับ Multi-thread
  ในสถาปัตยกรรมที่ซับซ้อนขึ้น)
- **นี่คือสถาปัตยกรรมเบื้องหลัง Web Server ประสิทธิภาพสูงในโลกจริง**: Nginx และ Redis
  (ที่จะพูดถึงในหลาย Part ของหลักสูตรนี้) ใช้แนวคิด I/O Multiplexing (โดยเฉพาะ `epoll` บน
  Linux) เป็นหัวใจหลักในการรองรับ Connection นับหมื่นพร้อมกันด้วยทรัพยากรที่จำกัด

Part นี้จะแนะนำเครื่องมือ 3 ตัวเรียงตามยุคสมัยและความสามารถ: `select()` → `poll()` → `epoll()`

---

## 34.5 select() แบบละเอียดพร้อมตัวอย่าง (Step 269)

`select()` เป็น System Call ที่เก่าแก่ที่สุด (มีมาตั้งแต่ยุค BSD Unix) ใช้หลักการ "ส่งชุดของ
File Descriptor ที่สนใจเข้าไป แล้วรอจน Kernel บอกว่าตัวไหนพร้อมทำงานบ้าง"

```c
int select(int nfds, fd_set *readfds, fd_set *writefds,
           fd_set *exceptfds, struct timeval *timeout);
```

- `nfds`: ค่า File Descriptor ที่มากที่สุด **บวก 1** ใน set ที่ส่งเข้าไป (Kernel ใช้ค่านี้
  จำกัดขอบเขตการสแกน — เป็นรายละเอียดทาง Performance ที่ต้องระวังไม่ลืมใส่ให้ถูก)
- `readfds`: ชุดของ fd ที่ต้องการรู้ว่า "พร้อมอ่านหรือยัง" (เช่น `server_fd` พร้อมรับ Client
  ใหม่ หรือ `client_fd` มีข้อมูลส่งเข้ามาแล้ว)
- `writefds`: ชุดของ fd ที่ต้องการรู้ว่า "พร้อมเขียนหรือยัง" (ใช้น้อยกว่า มักปล่อย `NULL`)
- `exceptfds`: ชุดของ fd ที่ต้องการตรวจจับสถานะพิเศษ (ใช้น้อยมาก มักปล่อย `NULL`)
- `timeout`: `NULL` = รอไม่จำกัดเวลา, ค่า 0 = ตรวจสอบแล้วคืนค่าทันทีไม่รอ (Polling), หรือ
  ระบุเวลาสูงสุดที่จะรอ

### Macro สำหรับจัดการ fd_set

`fd_set` เป็นโครงสร้างข้อมูลแบบ Bitmask ภายใน ที่ **ห้ามแก้ไขตรงๆ** ต้องใช้ Macro ชุดนี้เท่านั้น:

| Macro | ความหมาย |
|---|---|
| `FD_ZERO(&set)` | เคลียร์ set ให้ว่างเปล่าทั้งหมด (ต้องเรียกก่อนใช้งานทุกครั้ง) |
| `FD_SET(fd, &set)` | เพิ่ม `fd` เข้าไปใน set |
| `FD_CLR(fd, &set)` | เอา `fd` ออกจาก set |
| `FD_ISSET(fd, &set)` | ตรวจสอบว่า `fd` อยู่ใน set (และ "พร้อม") หรือไม่ หลังจาก `select()` คืนค่ากลับมาแล้ว |

### จุดสำคัญที่มือใหม่มักพลาด: ต้องสร้าง fd_set ใหม่ทุกรอบ

`select()` จะ **แก้ไข** `fd_set` ที่ส่งเข้าไปโดยตรง (เปลี่ยนให้เหลือแค่ fd ที่พร้อมจริงๆ)
ทำให้ถ้าเราจะเรียก `select()` อีกรอบใน loop ถัดไป **ต้องเรียก `FD_ZERO`/`FD_SET` ตั้งค่าใหม่
ทั้งหมดทุกครั้งก่อนเรียก** ไม่งั้นชุดข้อมูลจะไม่ตรงกับ Client ที่ยังเชื่อมต่ออยู่จริง

### ข้อจำกัดสำคัญของ select()

- **`FD_SETSIZE`**: จำกัดจำนวน fd สูงสุดที่ใส่ใน `fd_set` ได้ (มักเป็น 1024 บน Linux) ทำให้
  ไม่เหมาะกับ Server ที่ต้องรองรับ Connection จำนวนมากจริงๆ
- **ประสิทธิภาพ O(n)**: ทุกครั้งที่เรียก `select()` Kernel ต้องสแกนทุก fd ใน set ทั้งหมด
  (แม้จะมีแค่ 1 ตัวที่พร้อมจริง) ยิ่งมี fd เยอะ ยิ่งช้าลงเป็นเส้นตรง
- **ต้อง reset set ใหม่ทุกรอบ** อย่างที่อธิบายไปข้างต้น เพิ่มความยุ่งยากในการเขียนโค้ด

---

## 34.6 poll() เทียบกับ select (Step 270)

`poll()` เป็น System Call รุ่นต่อมาที่ออกแบบมาแก้ข้อจำกัดบางอย่างของ `select()`:

```c
struct pollfd {
    int   fd;         /* file descriptor ที่จะตรวจสอบ */
    short events;     /* บอกว่าสนใจ event อะไร (เราตั้งค่าเอง) เช่น POLLIN */
    short revents;    /* kernel เติมกลับมาว่า event อะไรเกิดขึ้นจริง */
};

int poll(struct pollfd *fds, nfds_t nfds, int timeout);
```

### ข้อดีของ poll() เทียบกับ select()

| หัวข้อ | select() | poll() |
|---|---|---|
| โครงสร้างข้อมูล | `fd_set` (Bitmask ขนาดคงที่) | Array ของ `struct pollfd` (ขนาดปรับได้ตาม `malloc`) |
| จำกัดจำนวน fd | มี (`FD_SETSIZE` มักเป็น 1024) | **ไม่มีข้อจำกัดตายตัว** ปรับขนาด array ได้ตามต้องการ |
| ต้อง reset ทุกรอบ | ต้อง `FD_ZERO`/`FD_SET` ใหม่ทุกครั้ง | ตั้งค่า `events` ครั้งเดียว ไม่ต้อง reset (Kernel เขียนผลลัพธ์ลง `revents` แยกต่างหาก) |
| แยก event อ่าน/เขียนได้ชัดเจนแค่ไหน | ต้องใช้ 3 set แยกกัน (`readfds`/`writefds`/`exceptfds`) | ใช้ bitmask เดียวใน `events` (`POLLIN`, `POLLOUT`, ...) รวมกันได้ในตัวเดียว |
| ประสิทธิภาพเมื่อ fd เยอะ | O(n) เหมือนกัน | O(n) เหมือนกัน (ยังไม่ได้แก้ปัญหานี้) |

`poll()` แก้ปัญหาเรื่อง **ขนาดจำกัด** และ **ความยุ่งยากในการ reset ค่า** ได้ดีกว่า `select()`
มาก แต่ยัง**ไม่ได้แก้ปัญหาประสิทธิภาพ O(n)** — ทุกครั้งที่เรียก `poll()` Kernel ยังคงต้องวน
ตรวจสอบทุก fd ใน Array ทั้งหมดอยู่ดี ถ้ามี Connection หลักหมื่น การสแกนทุกตัวทุกรอบก็ยัง
สิ้นเปลืองอยู่ดี — นี่คือช่องว่างที่ `epoll()` เข้ามาแก้ในหัวข้อถัดไป

ค่า `events`/`revents` ที่ใช้บ่อย:

| ค่า | ความหมาย |
|---|---|
| `POLLIN` | มีข้อมูลให้อ่าน (คล้าย `readfds` ของ select) |
| `POLLOUT` | พร้อมให้เขียนข้อมูล |
| `POLLHUP` | อีกฝั่งปิดการเชื่อมต่อ (Hang Up) |
| `POLLERR` | เกิด error กับ fd นั้น |

---

## 34.7 epoll(): Edge-Triggered vs Level-Triggered เบื้องต้น (Step 271)

`epoll()` เป็นกลไกเฉพาะของ **Linux** (ไม่ portable ไปยัง macOS/BSD ที่ใช้ `kqueue` แทน)
ออกแบบมาเพื่อแก้ปัญหาประสิทธิภาพ O(n) ของ `select()`/`poll()` โดยตรง

### แนวคิดหลัก: ให้ Kernel "จำ" รายการ fd ไว้ ไม่ต้องส่งซ้ำทุกรอบ

ต่างจาก `select()`/`poll()` ที่ต้องส่ง**รายการ fd ทั้งหมด**เข้าไปให้ Kernel ตรวจทุกครั้งที่
เรียก `epoll()` ใช้วิธี **ลงทะเบียน fd ไว้กับ Kernel ล่วงหน้าเพียงครั้งเดียว** (ผ่าน
`epoll_ctl`) จากนั้น Kernel จะคอย "แจ้งเตือน" เฉพาะ fd ที่พร้อมจริงๆ เท่านั้นเมื่อเรียก
`epoll_wait` — ทำให้ประสิทธิภาพไม่ขึ้นกับจำนวน fd ทั้งหมดที่ลงทะเบียนไว้ (**O(1)** โดยประมาณ
ต่อ event ที่เกิดขึ้นจริง แทนที่จะเป็น O(n) ต่อ fd ทั้งหมด)

```c
int epoll_create1(int flags);                                    /* สร้าง epoll instance */
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *ev);  /* เพิ่ม/แก้/ลบ fd ที่สนใจ */
int epoll_wait(int epfd, struct epoll_event *events,
                int maxevents, int timeout);                     /* รอ event ที่เกิดขึ้นจริง */
```

```
select()/poll():                          epoll():
  ทุกรอบ ส่งรายการ fd ทั้งหมดไปให้ kernel      ลงทะเบียน fd ไว้ล่วงหน้าครั้งเดียว (epoll_ctl)
  kernel สแกนทุกตัว O(n) ทุกครั้ง             ทุกรอบแค่ถาม kernel ว่า "มีอะไรพร้อมบ้าง"
                                            kernel ตอบเฉพาะที่พร้อมจริง O(1) ต่อ event
```

### Level-Triggered (ค่า default) vs Edge-Triggered

นี่คือแนวคิดที่สำคัญที่สุดของ `epoll()` และมักสร้างความสับสนให้มือใหม่:

- **Level-Triggered (LT)** — ค่า default: `epoll_wait` จะแจ้งเตือนซ้ำๆ **ตราบใดที่ยังมี
  ข้อมูลค้างอยู่ให้อ่าน** แม้เราจะยังอ่านไม่หมดในรอบก่อนหน้าก็ตาม พฤติกรรมนี้**เหมือนกับ**
  `select()`/`poll()` ทุกประการ ทำให้ Migrate โค้ดเดิมมาใช้ `epoll()` ได้ง่าย โดยไม่ต้องเปลี่ยน
  ตรรกะการอ่านข้อมูลเลย
- **Edge-Triggered (ET)**: เปิดใช้งานด้วย Flag `EPOLLET` — `epoll_wait` จะแจ้งเตือน **เพียง
  ครั้งเดียว** ตอนที่สถานะ "เปลี่ยนจากไม่พร้อมเป็นพร้อม" เท่านั้น ถ้าเราไม่รีบอ่านข้อมูลให้
  หมดในรอบนั้น (`recv()` วนจนกว่าจะได้ `EAGAIN`/`EWOULDBLOCK`) ข้อมูลที่เหลือค้างจะไม่ถูก
  แจ้งเตือนซ้ำอีกจนกว่าจะมีข้อมูลใหม่เข้ามาเพิ่ม — จำเป็นต้องใช้ Non-blocking Socket คู่กัน
  เสมอ ไม่งั้นโปรแกรมจะค้างที่ `recv()` ตอนพยายามอ่านให้หมด

```
Level-Triggered:  มีข้อมูล 100 byte ค้างอยู่ -> แจ้งเตือนทุกรอบจนกว่าจะอ่านหมด (ง่ายกว่า)
Edge-Triggered:   มีข้อมูล 100 byte เข้ามาใหม่ -> แจ้งเตือน "ครั้งเดียว"
                  ถ้าอ่านไม่หมดในรอบนั้น ต้องรอข้อมูลใหม่มาอีกถึงจะถูกแจ้งเตือนอีกครั้ง
```

| หัวข้อ | Level-Triggered | Edge-Triggered |
|---|---|---|
| ความง่ายในการเขียนโค้ด | ง่ายกว่า ใกล้เคียง select/poll | ยากกว่า ต้องอ่าน/เขียนจนหมดทุกครั้งและใช้ Non-blocking Socket |
| ความเสี่ยง Bug | ต่ำกว่า | สูงกว่าถ้าอ่านไม่หมดจะพลาด event ในรอบถัดไป |
| ประสิทธิภาพ | ดี | ดีกว่าเล็กน้อยในงานที่ throughput สูงมาก (ลด syscall ที่ไม่จำเป็น) |
| ใช้ในโปรเจกต์นี้ | ✅ (เพื่อความเข้าใจง่ายก่อน) | จะกล่าวถึงเพิ่มเติมเมื่อพูดถึง High-Performance Server ในภายหลัง |

สำหรับหลักสูตรนี้ ตัวอย่างในหัวข้อถัดไปและใน Part 35 จะใช้ **Level-Triggered** (ค่า default
ไม่ใส่ `EPOLLET`) เพราะเข้าใจง่ายกว่าและเพียงพอสำหรับการเรียนรู้แนวคิดพื้นฐาน ส่วน
Edge-Triggered จะกลับมาพูดถึงอีกครั้งเมื่อไปถึงหัวข้อ Multi-threaded HTTP Server (Part 101)
ที่ต้องการประสิทธิภาพสูงสุด

### ตัวอย่าง epoll() แบบย่อ (Level-Triggered)

```c
int epfd = epoll_create1(0);

struct epoll_event ev;
ev.events = EPOLLIN;             /* level-triggered โดย default (ไม่ใส่ EPOLLET) */
ev.data.fd = server_fd;
epoll_ctl(epfd, EPOLL_CTL_ADD, server_fd, &ev);

struct epoll_event events[MAX_EVENTS];
int n_ready = epoll_wait(epfd, events, MAX_EVENTS, -1);

for (int i = 0; i < n_ready; i++) {
    int fd = events[i].data.fd;
    if (fd == server_fd) {
        /* มี client ใหม่มาเชื่อมต่อ -> accept() แล้วลงทะเบียนด้วย EPOLL_CTL_ADD */
    } else {
        /* fd นี้มีข้อมูลให้อ่าน -> recv() ได้เลย */
    }
}
```

### เปรียบเทียบสรุปทั้ง 3 ตัว

| หัวข้อ | select() | poll() | epoll() |
|---|---|---|---|
| แพลตฟอร์ม | ทุก Unix/POSIX | ทุก Unix/POSIX | **เฉพาะ Linux** |
| จำกัดจำนวน fd | มี (`FD_SETSIZE`) | ไม่มี | ไม่มี |
| ประสิทธิภาพเมื่อ fd เยอะมาก | แย่ O(n) ต่อการเรียกทุกครั้ง | แย่ O(n) ต่อการเรียกทุกครั้ง | **ดีมาก O(1) ต่อ event ที่เกิดจริง** |
| ต้องส่งรายการ fd ซ้ำทุกรอบ | ต้อง | ไม่ต้อง (แต่ยังสแกน array ทั้งหมด) | **ไม่ต้อง** (ลงทะเบียนครั้งเดียว) |
| เหมาะกับ | โปรแกรมเล็กๆ, code ที่ต้อง portable ข้าม OS | โปรแกรมขนาดกลาง ที่ยังต้อง portable | Server ประสิทธิภาพสูงบน Linux (Nginx, Redis ใช้แนวทางนี้) |

---

## 34.8 สร้าง TCP Server รองรับหลาย Client พร้อมกันด้วย select() ตัวเต็ม (Step 272)

มาถึงเป้าหมายสำคัญของ Part นี้: แก้ข้อจำกัดของ TCP Server จาก Part 33 ที่รับได้ทีละ 1 Client
ให้กลายเป็น Server ที่รองรับ **หลาย Client พร้อมกันได้จริง** โดยยังคงเป็น Thread เดียว

```c
/* select_server.c - TCP server รองรับหลาย client พร้อมกันด้วย select() */
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <sys/socket.h>
#include <sys/select.h>
#include <netinet/in.h>

#define PORT        6000
#define MAX_CLIENTS FD_SETSIZE
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
    if (listen(server_fd, 10) < 0) {
        perror("listen"); exit(EXIT_FAILURE);
    }

    printf("[select-server] ฟังที่ port %d\n", PORT);

    int client_fds[MAX_CLIENTS];
    for (int i = 0; i < MAX_CLIENTS; i++) client_fds[i] = -1;

    fd_set read_fds;
    int running = 1;
    int total_messages_handled = 0;

    while (running) {
        /* ต้อง reset fd_set ใหม่ทุกรอบ เพราะ select() แก้ไข set ที่ส่งเข้าไปโดยตรง */
        FD_ZERO(&read_fds);
        FD_SET(server_fd, &read_fds);
        int max_fd = server_fd;

        for (int i = 0; i < MAX_CLIENTS; i++) {
            if (client_fds[i] != -1) {
                FD_SET(client_fds[i], &read_fds);
                if (client_fds[i] > max_fd) max_fd = client_fds[i];
            }
        }

        int activity = select(max_fd + 1, &read_fds, NULL, NULL, NULL);
        if (activity < 0) { perror("select"); break; }

        /* 1) มี client ใหม่มาเชื่อมต่อ */
        if (FD_ISSET(server_fd, &read_fds)) {
            struct sockaddr_in client_addr;
            socklen_t client_len = sizeof(client_addr);
            int new_fd = accept(server_fd, (struct sockaddr *)&client_addr, &client_len);
            if (new_fd >= 0) {
                int slot = -1;
                for (int i = 0; i < MAX_CLIENTS; i++) {
                    if (client_fds[i] == -1) { slot = i; break; }
                }
                if (slot == -1) {
                    printf("[select-server] client เต็ม ปฏิเสธการเชื่อมต่อ\n");
                    close(new_fd);
                } else {
                    client_fds[slot] = new_fd;
                    printf("[select-server] client ใหม่ fd=%d (slot %d)\n", new_fd, slot);
                }
            }
        }

        /* 2) client เดิมส่งข้อมูลมา */
        for (int i = 0; i < MAX_CLIENTS; i++) {
            int fd = client_fds[i];
            if (fd != -1 && FD_ISSET(fd, &read_fds)) {
                char buffer[BUFFER_SIZE];
                ssize_t n = recv(fd, buffer, sizeof(buffer) - 1, 0);
                if (n <= 0) {
                    printf("[select-server] client fd=%d หลุดการเชื่อมต่อ\n", fd);
                    close(fd);
                    client_fds[i] = -1;
                } else {
                    buffer[n] = '\0';
                    printf("[select-server] fd=%d ส่ง: %s\n", fd, buffer);
                    total_messages_handled++;
                    send(fd, buffer, (size_t)n, 0); /* echo กลับ */
                    if (strncmp(buffer, "quit", 4) == 0) {
                        running = 0;
                    }
                }
            }
        }
    }

    printf("[select-server] จบการทำงาน จัดการไปทั้งหมด %d ข้อความ\n", total_messages_handled);
    for (int i = 0; i < MAX_CLIENTS; i++) {
        if (client_fds[i] != -1) close(client_fds[i]);
    }
    close(server_fd);
    return 0;
}
```

### ทดสอบด้วย 2 Client พร้อมกันจริง

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 select_server.c -o select_server
./select_server &
sleep 0.3

# เปิด client 2 ตัวพร้อมกัน (background) ก่อน server จะได้จัดการทั้งคู่ในรอบ select เดียวกัน
./tcp_client "hello-from-A" &
./tcp_client "hello-from-B" &
wait
./tcp_client "quit"   # ส่งคำสั่งพิเศษให้ server ปิดตัวเอง (ตามที่ออกแบบไว้ในโค้ด)
```

ผลลัพธ์ฝั่ง Server (fd ตัวเลขจริงอาจต่างกันไปในแต่ละเครื่อง):

```
[select-server] ฟังที่ port 6000
[select-server] client ใหม่ fd=4 (slot 0)
[select-server] client ใหม่ fd=5 (slot 1)
[select-server] fd=4 ส่ง: hello-from-A
[select-server] fd=5 ส่ง: hello-from-B
[select-server] client fd=4 หลุดการเชื่อมต่อ
[select-server] client fd=5 หลุดการเชื่อมต่อ
[select-server] client ใหม่ fd=4 (slot 0)
[select-server] fd=4 ส่ง: quit
[select-server] จบการทำงาน จัดการไปทั้งหมด 3 ข้อความ
```

สังเกตให้ดี: **Server ตัวนี้เป็น Thread เดียว Process เดียว** แต่สามารถรับ Client A และ B
ที่เชื่อมต่อเข้ามา**พร้อมกัน**ได้ทั้งคู่ในรอบ `select()` เดียว (ทั้งสอง fd ถูก
`FD_ISSET` เป็นจริงพร้อมกัน) แล้ววนจัดการทีละตัวตามลำดับใน `for` loop — นี่คือหัวใจของ I/O
Multiplexing: **ดูเหมือนทำงานพร้อมกัน (concurrent) แต่จริงๆ ประมวลผลทีละ Event ใน Thread
เดียว** ไม่มี Race Condition ให้ต้องกังวลเหมือนตอนใช้ Multi-thread ใน Part 32 เพราะไม่มี
Shared State ที่ถูกแก้ไขจากหลาย Thread พร้อมกันเลย

> โค้ดตัวเต็มของ `poll()` และ `epoll()` เวอร์ชันเดียวกันนี้ (โครงสร้างคล้ายกันมาก เปลี่ยน
> แค่ API การรอ event) เป็นแบบฝึกหัดข้อ 4-5 ท้ายบท เพื่อให้เห็นความแตกต่างของโค้ดจริงด้วยมือ
> ตัวเอง

---

## ข้อผิดพลาดที่พบบ่อย (Common Pitfalls)

1. **ลืม `bind()` ฝั่ง UDP Server** — เข้าใจผิดว่า UDP ไม่มี "connection" เลยไม่ต้อง bind
   ทำให้ Server ไม่มีทางรู้ว่าจะรอรับข้อมูลที่ Port ไหน (`bind()` ยังจำเป็นเสมอฝั่งที่ต้อง
   "รอรับ" ไม่ว่าจะเป็น TCP หรือ UDP)
2. **คาดหวังว่า UDP รับประกันการส่งถึง** — เขียนโค้ดที่ไม่มี Timeout หรือกลไกยืนยันใดๆ ทำให้
   โปรแกรมค้างตลอดไปที่ `recvfrom()` ถ้า Packet หายไปจริง (ตามที่สาธิตในหัวข้อ 34.3)
3. **ลืม reset `fd_set` ก่อนเรียก `select()` ทุกรอบ** — เพราะ `select()` แก้ไขค่าใน set
   โดยตรง (ตัด fd ที่ไม่พร้อมออกไป) ถ้าไม่เรียก `FD_ZERO`/`FD_SET` ใหม่ทุกรอบ ใน Loop ถัดไป
   จะได้ค่าที่ไม่ตรงกับรายการ Client ที่ยังเชื่อมต่ออยู่จริง
4. **คำนวณ `nfds` (ตัวแรกของ `select()`) ผิด** — ต้องเป็นค่า fd สูงสุด **บวก 1** เสมอ ถ้าลืม
   บวก 1 หรือใช้ค่าคงที่ที่ไม่ได้อัปเดตตามจำนวน Client จริง `select()` อาจมองข้าม fd บางตัว
   ไปโดยไม่แจ้ง error ใดๆ
5. **ใช้ Edge-Triggered `epoll` (`EPOLLET`) โดยไม่เปลี่ยน Socket เป็น Non-blocking และไม่
   อ่านข้อมูลจนหมดในแต่ละรอบ** — ทำให้พลาด event ที่เหลือค้างอยู่ในรอบถัดไป และถ้า Socket
   ยังเป็น Blocking อาจทำให้ Thread ค้างที่ `recv()` เมื่อพยายามอ่านให้หมดแต่ข้อมูลหมดไปก่อน
6. **ลืมว่า `select()`/`poll()` มีข้อจำกัดด้านจำนวน/ประสิทธิภาพ** — เลือกใช้กับระบบที่ต้อง
   รองรับ Connection จำนวนมาก (หลักพันขึ้นไป) ทำให้ Server ช้าลงเรื่อยๆ เมื่อ Client เพิ่มขึ้น
   ทั้งที่ตอนออกแบบทดสอบด้วย Client จำนวนน้อยแล้วดูเหมือนไม่มีปัญหา
7. **ปิด fd แล้วลืมเอาออกจาก Array/Set ที่ติดตามอยู่** — เช่นใน `select_server.c` ถ้าลืมตั้ง
   `client_fds[i] = -1` หลัง `close(fd)` รอบถัดไปจะพยายาม `FD_SET` บน fd ที่ปิดไปแล้ว ซึ่ง
   อาจกลายเป็น fd ใหม่ที่ระบบเอาเลขเดิมมาใช้ซ้ำ (fd number ถูก reuse ได้) ทำให้เกิดพฤติกรรม
   ที่คาดเดาไม่ได้

---

## แบบฝึกหัดท้ายบท

1. แก้ไข `udp_server.c` ให้รับ Packet **ไม่จำกัดจำนวน** (เอาเงื่อนไข `for (int i = 0; i < 3;
   i++)` ออก ใช้ `while (1)` แทน) แล้วทดสอบส่ง Packet จาก `udp_client.c` หลายๆ ครั้งติดกัน
2. เขียนโปรแกรม UDP Client ที่ส่ง Packet ไปยัง Server ที่**ไม่มีใครฟังอยู่จริง** (เช่น Port
   ที่ไม่ได้เปิด Server ไว้) แล้วสังเกตว่า `sendto()` ยัง "สำเร็จ" ตามที่คาดหรือไม่ อธิบายว่า
   ทำไมถึงต่างจากพฤติกรรมของ TCP `connect()` ที่ Part 33 (ที่จะได้ `ECONNREFUSED` ทันที)
3. แก้ไข `select_server.c` ให้เพิ่ม Feature: เมื่อ Client คนหนึ่งพิมพ์ข้อความอะไรมา ให้
   Server ส่งข้อความนั้นไปยัง **ทุก Client ที่เชื่อมต่ออยู่** (Broadcast) ไม่ใช่ echo กลับไป
   หาแค่คนที่ส่งมาเท่านั้น (นี่คือรากฐานสำคัญของโปรเจกต์ TCP Chat Server ใน Part 35)
4. เขียน TCP Server ที่รองรับหลาย Client พร้อมกันด้วย `poll()` แทน `select()` (โครงสร้าง
   คล้ายกับ `select_server.c` มาก) แล้วเปรียบเทียบจำนวนบรรทัดโค้ดและความซับซ้อนของทั้งสอง
   เวอร์ชัน
5. เขียน TCP Server ที่รองรับหลาย Client พร้อมกันด้วย `epoll()` (Level-Triggered) แล้วทดสอบ
   กับ Client หลายตัวพร้อมกันเหมือนข้อ 3-4
6. วัดความแตกต่างของประสิทธิภาพเชิงแนวคิด: เขียนคำอธิบาย (ไม่ต้องเขียนโค้ดวัดจริง) ว่าถ้ามี
   Client เชื่อมต่อพร้อมกัน 10,000 ราย เหตุใด Server ที่ใช้ `select()`/`poll()` ถึงมีแนวโน้ม
   ช้ากว่า Server ที่ใช้ `epoll()` อย่างมีนัยสำคัญ

### แนวทางเฉลยข้อ 4

```c
/* poll_server.c - เฉลยข้อ 4: TCP server รองรับหลาย client ด้วย poll() */
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <sys/socket.h>
#include <poll.h>
#include <netinet/in.h>

#define PORT        6100
#define MAX_CLIENTS 64
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
    if (listen(server_fd, 10) < 0) { perror("listen"); exit(EXIT_FAILURE); }

    printf("[poll-server] ฟังที่ port %d\n", PORT);

    /* ต่างจาก select(): ใช้ array ของ struct pollfd แทน fd_set ไม่มีข้อจำกัด FD_SETSIZE */
    struct pollfd fds[MAX_CLIENTS + 1];
    int nfds = 1;
    fds[0].fd = server_fd;
    fds[0].events = POLLIN;
    for (int i = 1; i <= MAX_CLIENTS; i++) fds[i].fd = -1;

    int running = 1;
    int total_messages = 0;

    while (running) {
        /* ต่างจาก select(): ไม่ต้อง reset events ทุกรอบ (ตั้งไว้ตอน add ก็พอ) */
        int ret = poll(fds, (nfds_t)nfds, -1); /* -1 = รอไม่จำกัดเวลา */
        if (ret < 0) { perror("poll"); break; }

        if (fds[0].revents & POLLIN) {
            struct sockaddr_in client_addr;
            socklen_t client_len = sizeof(client_addr);
            int new_fd = accept(server_fd, (struct sockaddr *)&client_addr, &client_len);
            if (new_fd >= 0) {
                if (nfds > MAX_CLIENTS) {
                    printf("[poll-server] เต็ม ปฏิเสธ client\n");
                    close(new_fd);
                } else {
                    fds[nfds].fd = new_fd;
                    fds[nfds].events = POLLIN;
                    printf("[poll-server] client ใหม่ fd=%d (index %d)\n", new_fd, nfds);
                    nfds++;
                }
            }
        }

        for (int i = 1; i < nfds; i++) {
            if (fds[i].fd == -1) continue;
            if (fds[i].revents & (POLLIN | POLLHUP | POLLERR)) {
                char buffer[BUFFER_SIZE];
                ssize_t n = recv(fds[i].fd, buffer, sizeof(buffer) - 1, 0);
                if (n <= 0) {
                    printf("[poll-server] client fd=%d หลุด\n", fds[i].fd);
                    close(fds[i].fd);
                    fds[i].fd = -1;
                } else {
                    buffer[n] = '\0';
                    printf("[poll-server] fd=%d ส่ง: %s\n", fds[i].fd, buffer);
                    total_messages++;
                    send(fds[i].fd, buffer, (size_t)n, 0);
                    if (strncmp(buffer, "quit", 4) == 0) running = 0;
                }
            }
        }
    }

    printf("[poll-server] จบการทำงาน จัดการไป %d ข้อความ\n", total_messages);
    for (int i = 0; i < nfds; i++) if (fds[i].fd != -1) close(fds[i].fd);
    close(server_fd);
    return 0;
}
```

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 poll_server.c -o poll_server
./poll_server &
sleep 0.3
./tcp_client "poll-A" &
./tcp_client "poll-B" &
wait
./tcp_client "quit"
```

ผลลัพธ์:

```
[poll-server] ฟังที่ port 6100
[poll-server] client ใหม่ fd=4 (index 1)
[poll-server] client ใหม่ fd=5 (index 2)
[poll-server] fd=4 ส่ง: poll-B
[poll-server] fd=5 ส่ง: poll-A
[poll-server] client fd=4 หลุด
[poll-server] client fd=5 หลุด
[poll-server] client ใหม่ fd=4 (index 3)
[poll-server] fd=4 ส่ง: quit
[poll-server] จบการทำงาน จัดการไป 3 ข้อความ
```

สังเกตว่าโครงสร้างโค้ดคล้ายกับ `select_server.c` มาก ความต่างหลักคือไม่ต้อง `FD_ZERO`/
`FD_SET` ใหม่ทุกรอบ (แค่ตั้งค่า `events` ตอนเพิ่ม Client เข้ามาครั้งเดียวก็พอ) และไม่มีข้อจำกัด
เรื่องจำนวน Client สูงสุดที่ตายตัวจาก `FD_SETSIZE` เหมือน `select()`

### แนวทางเฉลยข้อ 5

```c
/* epoll_server.c - เฉลยข้อ 5: TCP server รองรับหลาย client ด้วย epoll() (level-triggered) */
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <sys/socket.h>
#include <sys/epoll.h>
#include <netinet/in.h>

#define PORT        6200
#define MAX_EVENTS  32
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
    if (listen(server_fd, 10) < 0) { perror("listen"); exit(EXIT_FAILURE); }

    printf("[epoll-server] ฟังที่ port %d\n", PORT);

    int epfd = epoll_create1(0);
    if (epfd < 0) { perror("epoll_create1"); exit(EXIT_FAILURE); }

    struct epoll_event ev;
    ev.events = EPOLLIN;             /* level-triggered โดย default (ไม่ใส่ EPOLLET) */
    ev.data.fd = server_fd;
    epoll_ctl(epfd, EPOLL_CTL_ADD, server_fd, &ev);

    struct epoll_event events[MAX_EVENTS];
    int running = 1;
    int total_messages = 0;

    while (running) {
        int n_ready = epoll_wait(epfd, events, MAX_EVENTS, -1);
        if (n_ready < 0) { perror("epoll_wait"); break; }

        for (int i = 0; i < n_ready; i++) {
            int fd = events[i].data.fd;

            if (fd == server_fd) {
                struct sockaddr_in client_addr;
                socklen_t client_len = sizeof(client_addr);
                int client_fd = accept(server_fd, (struct sockaddr *)&client_addr, &client_len);
                if (client_fd >= 0) {
                    struct epoll_event client_ev;
                    client_ev.events = EPOLLIN;
                    client_ev.data.fd = client_fd;
                    epoll_ctl(epfd, EPOLL_CTL_ADD, client_fd, &client_ev);
                    printf("[epoll-server] client ใหม่ fd=%d\n", client_fd);
                }
            } else {
                char buffer[BUFFER_SIZE];
                ssize_t r = recv(fd, buffer, sizeof(buffer) - 1, 0);
                if (r <= 0) {
                    printf("[epoll-server] client fd=%d หลุด\n", fd);
                    epoll_ctl(epfd, EPOLL_CTL_DEL, fd, NULL);
                    close(fd);
                } else {
                    buffer[r] = '\0';
                    printf("[epoll-server] fd=%d ส่ง: %s\n", fd, buffer);
                    total_messages++;
                    send(fd, buffer, (size_t)r, 0);
                    if (strncmp(buffer, "quit", 4) == 0) running = 0;
                }
            }
        }
    }

    printf("[epoll-server] จบการทำงาน จัดการไป %d ข้อความ\n", total_messages);
    close(epfd);
    close(server_fd);
    return 0;
}
```

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 epoll_server.c -o epoll_server
./epoll_server &
sleep 0.3
./tcp_client "epoll-A" &
./tcp_client "epoll-B" &
wait
./tcp_client "quit"
```

ผลลัพธ์:

```
[epoll-server] ฟังที่ port 6200
[epoll-server] client ใหม่ fd=5
[epoll-server] fd=5 ส่ง: epoll-B
[epoll-server] client fd=5 หลุด
[epoll-server] client ใหม่ fd=5
[epoll-server] fd=5 ส่ง: epoll-A
[epoll-server] client fd=5 หลุด
[epoll-server] client ใหม่ fd=5
[epoll-server] fd=5 ส่ง: quit
[epoll-server] จบการทำงาน จัดการไป 3 ข้อความ
```

สังเกตว่า `epoll_ctl(EPOLL_CTL_ADD, ...)` ถูกเรียกแค่ครั้งเดียวต่อ fd แต่ละตัว (ตอนเพิ่มเข้า
ระบบครั้งแรก) ไม่ต้องส่งรายการ fd ทั้งหมดซ้ำในทุกรอบเหมือน `select()`/`poll()` เลย นี่คือ
เหตุผลที่ `epoll()` มีประสิทธิภาพดีกว่ามากเมื่อจำนวน Connection สูงขึ้น เพราะต้นทุนของการ
"ถาม Kernel ว่ามีอะไรพร้อมบ้าง" ไม่ได้ขึ้นกับจำนวน fd ทั้งหมดที่ลงทะเบียนไว้อีกต่อไป

---

## สรุปท้ายบท

ใน Part นี้เราได้:

- ใช้งาน UDP Socket ด้วย `sendto()`/`recvfrom()` และเข้าใจความแตกต่างเชิงลึกจาก TCP ทั้ง
  เรื่องความน่าเชื่อถือ ขอบเขตข้อความ และการใช้งานที่เหมาะสมของแต่ละ Protocol
- เข้าใจว่า I/O Multiplexing คืออะไร และทำไมการรองรับหลาย Connection พร้อมกันไม่จำเป็นต้อง
  ใช้หลาย Thread หรือ Process เสมอไป
- ใช้ `select()` เขียน TCP Server ที่รองรับหลาย Client พร้อมกันได้จริงใน Thread เดียว
- เปรียบเทียบ `poll()` กับ `select()` และเข้าใจว่า `poll()` แก้ปัญหาเรื่องขนาดจำกัดและความ
  ยุ่งยากในการ reset ค่าได้ แต่ยังคงมีข้อจำกัดด้านประสิทธิภาพ O(n) เหมือนกัน
- เข้าใจแนวคิดของ `epoll()` ทั้งเรื่องการลงทะเบียน fd ล่วงหน้าที่ทำให้ได้ประสิทธิภาพ O(1)
  ต่อ event และความแตกต่างระหว่าง Level-Triggered กับ Edge-Triggered

เราตอนนี้มีเครื่องมือครบทั้งชุดสำหรับสร้าง Network Server ที่รองรับผู้ใช้จำนวนมากพร้อมกันได้
แล้ว ทั้ง `select()`, `poll()`, และ `epoll()` — สิ่งที่ยังขาดคือการนำความรู้ทั้งหมดจาก Part 31
ถึง 34 มารวมกันเป็นโปรเจกต์จริง ใน **Part 35** เราจะสร้าง **TCP Chat Server/Client** แบบ
Multi-client เต็มรูปแบบ ที่ผู้ใช้หลายคนสามารถเชื่อมต่อเข้ามาคุยกันพร้อมกันได้จริง โดยนำเทคนิค
Broadcast (จากแบบฝึกหัดข้อ 3), Length-Prefixed Framing (จาก Part 33), และ I/O Multiplexing
(จาก Part นี้) มาประกอบร่างเป็นระบบที่ใช้งานได้จริง

**ต่อไป:** [Part 35 — โปรเจกต์ TCP Chat Server/Client](./part-035-tcp-chat-project.md)
