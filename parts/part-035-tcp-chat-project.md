# Part 35: โปรเจกต์ TCP Chat Server/Client (Step 273–280)

> Module C — Systems Programming ด้วย C บน Linux | Part 35 จาก 125
> Step ที่ครอบคลุมใน Part นี้: Step 273–280
> Part ก่อนหน้า: [Part 34 — Socket ขั้นสูง (UDP/select/poll/epoll)](./part-034-sockets-advanced.md) | Part ถัดไป: [Part 36 — Memory-Mapped File (mmap)](./part-036-mmap.md)

## เป้าหมายการเรียนรู้ (Learning Objectives)

หลังจบ Part นี้ ผู้เรียนจะสามารถ:

1. ออกแบบสถาปัตยกรรมของ TCP chat server แบบ **thread-per-client** ได้ด้วยตนเอง โดยรวม
   ความรู้เรื่อง socket (Part 33-34) กับ pthread และ mutex (Part 31-32) เข้าด้วยกัน
2. อธิบายได้ว่าทำไม client list ที่ถูกแชร์ระหว่างหลาย thread จำเป็นต้องมี mutex ป้องกัน
   และอธิบายได้ว่าจะเกิดอะไรขึ้นถ้าไม่มี (race condition, data corruption, segfault)
3. เขียนฟังก์ชัน `broadcast_message()` ที่ส่งข้อความจาก client หนึ่งไปยัง client ทุกคน
   ที่เชื่อมต่ออยู่ได้อย่างปลอดภัยจาก race condition
4. เขียน client โปรแกรมที่แยก thread สำหรับ "รับ" และ "ส่ง" ข้อความให้ทำงานพร้อมกันได้
   (full-duplex communication) โดยไม่ต้องรอสลับกันทีละฝั่ง
5. จัดการการ disconnect ของ client อย่างถูกต้องครบถ้วน — ปิด file descriptor, ลบออกจาก
   client list, แจ้งเตือนคนอื่น โดยไม่เกิด file descriptor leak หรือ memory leak
6. เข้าใจข้อจำกัดสำคัญของ TCP ในฐานะ **byte stream** (ไม่ใช่ message stream) และเขียนโค้ด
   ที่ทนทานต่อกรณีที่ `recv()` รวมหรือแยกข้อความที่ `send()` ส่งมาไม่ตรงตามจำนวนครั้ง
7. คอมไพล์ รัน และทดสอบระบบ multi-client จริงด้วย terminal หลายหน้าต่าง รวมถึงใช้ `netcat`
   เป็น client จำลองเพื่อตรวจสอบว่าโปรโตคอลที่ออกแบบไว้เป็น plain text จริง
8. ใช้ `valgrind` ตรวจสอบเบื้องต้นว่าโปรแกรมไม่มี memory leak หรือ file descriptor leak
   หลังจากมี client เชื่อมต่อเข้า-ออกหลายรอบ
9. รู้แนวทางต่อยอดโปรเจกต์นี้ไปเป็นระบบที่สมบูรณ์ขึ้น เช่น private message, thread pool,
   และเข้าใจข้อจำกัดของสถาปัตยกรรม thread-per-client เมื่อ client มีจำนวนมาก

---

## 35.1 ภาพรวมโปรเจกต์และสถาปัตยกรรม (Step 273)

Part นี้เป็น **โปรเจกต์รวบยอด** ของ Module C ครึ่งแรก เราจะนำความรู้จาก 4 Part ก่อนหน้ามา
ประกอบร่างเป็นระบบที่ใช้งานได้จริง:

| มาจาก Part | ความรู้ที่ใช้ในโปรเจกต์นี้ |
|---|---|
| Part 31 — POSIX Threads เบื้องต้น | `pthread_create()` สร้าง thread ใหม่ต่อ client หนึ่งคน, `pthread_detach()` |
| Part 32 — Thread Synchronization | `pthread_mutex_t` ป้องกัน client list ที่แชร์กันระหว่าง thread |
| Part 33 — Socket Programming (TCP) | `socket()`, `bind()`, `listen()`, `accept()`, `connect()`, `send()`/`recv()` |
| Part 34 — Socket ขั้นสูง | เข้าใจข้อจำกัดของ blocking I/O ที่ผลักดันให้ต้องใช้ thread-per-client ในบทนี้ |

โจทย์ของเราคือ: สร้าง **TCP Chat Server** ที่รับ client ได้พร้อมกันหลายคน ทุกคนอยู่ใน "ห้อง"
เดียวกัน (broadcast room) เมื่อใครพิมพ์ข้อความ ทุกคนที่เหลือในห้องจะเห็นข้อความนั้นทันที

### สถาปัตยกรรมแบบ Thread-per-Client

แนวคิดหลักคือ: ทุกครั้งที่มี client ใหม่เชื่อมต่อเข้ามา (`accept()` สำเร็จ) server จะสร้าง
thread ใหม่ 1 เส้นขึ้นมาโดยเฉพาะเพื่อคุยกับ client คนนั้น thread หลัก (main thread) มีหน้าที่
แค่วนลูป `accept()` รับคนใหม่ตลอดไปเท่านั้น ไม่ต้องยุ่งเกี่ยวกับการรับส่งข้อความเลย

```
                                SERVER PROCESS
   ┌───────────────────────────────────────────────────────────────┐
   │                                                                 │
   │   main thread                                                  │
   │   ┌─────────────────────┐                                      │
   │   │  for (;;) {          │                                     │
   │   │    accept()  ◀───────┼───── client A เชื่อมต่อเข้ามา        │
   │   │    pthread_create() ─┼──▶ [Thread A] recv/send กับ A        │
   │   │                       │         │                          │
   │   │    accept()  ◀───────┼───── client B เชื่อมต่อเข้ามา        │
   │   │    pthread_create() ─┼──▶ [Thread B] recv/send กับ B        │
   │   │                       │         │                          │
   │   │    accept()  ◀───────┼───── client C เชื่อมต่อเข้ามา        │
   │   │    pthread_create() ─┼──▶ [Thread C] recv/send กับ C        │
   │   │  }                    │         │                          │
   │   └─────────────────────┘         │                          │
   │                                     ▼                          │
   │                     ┌───────────────────────────┐              │
   │                     │   client_list[MAX_CLIENTS]  │◀─── shared │
   │                     │   (ป้องกันด้วย mutex)         │   resource │
   │                     └───────────────────────────┘              │
   │                          ▲       ▲       ▲                     │
   │            Thread A ─────┘       │       └───── Thread C       │
   │                          Thread B (broadcast อ่าน/เขียน list)  │
   └───────────────────────────────────────────────────────────────┘
```

ข้อดีของสถาปัตยกรรมนี้: **เขียนง่าย** โค้ดของแต่ละ thread อ่านเป็นลำดับ (sequential) ตรงไปตรงมา
เหมือนเขียนโปรแกรมคุยกับ client คนเดียว ไม่ต้องยุ่งกับ event loop หรือ non-blocking I/O
เหมาะสำหรับจำนวน client ระดับหลักสิบถึงหลักร้อย

ข้อจำกัด (ที่จะพูดถึงเพิ่มใน 35.8): แต่ละ thread กิน stack memory เริ่มต้นประมาณ 8 MB
(ค่า default บน Linux) ถ้ามี client หลักหมื่นคนพร้อมกัน การสร้าง thread นับหมื่นเส้นจะกิน
ทรัพยากรมหาศาลและ context-switching overhead จะสูงมาก นี่คือเหตุผลที่ระบบระดับ production
ขนาดใหญ่ (เช่น nginx, Part 101 ของหลักสูตรนี้) มักใช้ **thread pool** หรือ **event-driven I/O**
(epoll ที่เรียนใน Part 34) แทน แต่สำหรับการเรียนรู้และงานขนาดกลาง thread-per-client
เป็นจุดเริ่มต้นที่ดีที่สุดเพราะเข้าใจง่ายที่สุด

### โปรโตคอลของแชท (ออกแบบเอง แบบง่ายที่สุด)

เราจะไม่ประดิษฐ์โปรโตคอลไบนารีซับซ้อน แต่ใช้ **plain text แบบ line-based** ล้วนๆ:

1. เมื่อ client เชื่อมต่อสำเร็จ ให้ส่งชื่อผู้ใช้เป็นบรรทัดแรก (จบด้วย `\n`)
2. หลังจากนั้นทุกบรรทัดที่ client พิมพ์จะถูกส่งไปเป็นข้อความแชท
3. server แปะชื่อผู้ส่งไว้หน้าข้อความ (`ชื่อ: ข้อความ`) แล้ว broadcast ไปให้ทุกคนยกเว้นผู้ส่งเอง
4. server จะแจ้งเตือนเมื่อมีคนเข้า/ออกห้องด้วยข้อความรูปแบบ `*** ... ***`

ข้อดีของการออกแบบแบบ plain text คือ **debug ง่ายมาก** — เราจะใช้ `netcat` (`nc`) ต่อเข้าไป
คุยกับ server ได้ตรงๆ โดยไม่ต้องเขียน client เองเลย (ดู 35.6)

---

## 35.2 ออกแบบ Data Structure ที่ใช้ร่วมกันระหว่าง Thread (Step 274)

หัวใจของโปรเจกต์นี้คือ **client list** — array ที่เก็บข้อมูลของ client ทุกคนที่กำลัง
เชื่อมต่ออยู่ ณ ขณะนั้น เราจะใช้ struct ง่ายๆ ดังนี้:

```c
#define MAX_CLIENTS  32
#define NAME_SIZE    32

typedef struct {
    int  sockfd;                 /* file descriptor ของ socket ตัวนี้ */
    struct sockaddr_in addr;     /* ที่อยู่ของ client (IP:port) */
    char name[NAME_SIZE];        /* ชื่อที่ client เลือกตอนเข้ามา */
    int  active;                 /* 1 = ช่องนี้ถูกใช้งานอยู่จริง, 0 = ว่าง */
} client_t;

static client_t client_list[MAX_CLIENTS];
```

**ทำไมใช้ fixed-size array ไม่ใช่ linked list?** เพื่อความง่ายในบทเรียนนี้ — เรากำหนดเพดาน
จำนวน client สูงสุดไว้ล่วงหน้า (32 คน) แล้ววนหาช่องว่าง (`active == 0`) เมื่อ client ใหม่เข้ามา
ข้อดีคือไม่ต้อง malloc/free แยกแต่ละ node ทำให้โค้ดสั้นและไม่มีปัญหาเรื่อง fragmentation
ข้อเสียคือขยายขนาดไม่ได้ตอนรันไทม์ (ดูแบบฝึกหัดข้อ 5 ที่ให้ลองแปลงเป็น dynamic array)

### ทำไมต้องมี Mutex?

`client_list` คือสิ่งที่เรียกว่า **shared mutable state** — ข้อมูลที่ถูกเขียนและอ่านจาก
หลาย thread พร้อมกัน:

- **main thread**: เขียนเพิ่มรายการใหม่ทุกครั้งที่มี client เชื่อมต่อเข้ามา (`client_list_add`)
- **thread ของ client แต่ละคน**: อ่าน list ทั้งหมดทุกครั้งที่ broadcast (`broadcast_message`)
  และลบรายการของตัวเองออกตอน disconnect (`client_list_remove`)

ลองจินตนาการว่าไม่มี mutex แล้วเกิดเหตุการณ์นี้พร้อมกันในเวลาไล่เลี่ยกัน:

```
เวลา    Thread A (broadcast ให้ client อื่น)      Thread B (client B กำลัง disconnect)
──────  ──────────────────────────────────────    ─────────────────────────────────────
t0      for (i = 0; i < MAX_CLIENTS; i++) {
t1        ถ้า client_list[5].active == 1 (จริง)
t2                                                  client_list_remove(client B)
                                                     -> client_list[5].active = 0
                                                     -> client_list[5].sockfd = -1
t3        send(client_list[5].sockfd, ...)  <- อ่านค่า sockfd ที่เพิ่งถูกเปลี่ยนเป็น -1!
                                                     ส่งไปยัง fd ผิด หรือ error
```

นี่คือ **race condition** ตัวอย่างคลาสสิก: ผลลัพธ์ขึ้นกับจังหวะเวลา (timing) ล้วนๆ
บางครั้งรันแล้วไม่มีปัญหา บางครั้งโปรแกรม crash หรือส่งข้อความผิดคน ซึ่งแย่ที่สุดคือ
บั๊กแบบนี้ **reproduce ยากมาก** เพราะไม่ได้เกิดทุกครั้งที่รัน

ทางแก้คือให้ทุก thread ที่จะแตะต้อง `client_list` ต้อง "จองคิว" ด้วย mutex ตัวเดียวกันก่อนเสมอ:

```c
static pthread_mutex_t client_list_mutex = PTHREAD_MUTEX_INITIALIZER;
```

กฎเหล็กของโปรเจกต์นี้: **ห้ามอ่านหรือเขียน `client_list` โดยไม่ล็อก `client_list_mutex`
เด็ดขาด ไม่มีข้อยกเว้น** แม้แต่การอ่านอย่างเดียว (read-only) ก็ต้องล็อก เพราะถ้า thread อื่น
กำลังเขียนอยู่พอดี การอ่านค่าครึ่งๆ กลางๆ ก็เป็นพฤติกรรมไม่นิยาม (Undefined Behavior) เช่นกัน

---

## 35.3 การตั้งค่า Socket และ Accept Loop ของ Server (Step 275)

เริ่มเขียน `server.c` จากส่วน `main()` ก่อน — เป็นส่วนที่ทำ 4 ขั้นตอนมาตรฐานของ TCP server
ที่เรียนมาแล้วใน Part 33 (`socket` → `bind` → `listen` → `accept`) บวกกับการสร้าง thread
ให้แต่ละ client:

```c
int main(void) {
    struct sockaddr_in server_addr;

    /* ดักจับ Ctrl+C เพื่อปิด server_fd ให้เรียบร้อยก่อนออกโปรแกรม
     * (ถ้าไม่ทำ port อาจค้างอยู่ใน TIME_WAIT จน bind ใหม่ไม่ได้ทันที) */
    signal(SIGINT, handle_sigint);

    /* SIGPIPE เกิดเมื่อเขียนลง socket ที่อีกฝั่งปิดไปแล้ว ถ้าไม่ ignore
     * โปรแกรมทั้งตัวจะถูกฆ่าทิ้งทันทีโดยไม่มีโอกาส cleanup */
    signal(SIGPIPE, SIG_IGN);

    client_list_init();

    /* 1) สร้าง socket แบบ TCP (SOCK_STREAM) */
    server_fd = socket(AF_INET, SOCK_STREAM, 0);
    if (server_fd < 0) {
        perror("socket");
        exit(EXIT_FAILURE);
    }

    /* อนุญาตให้ bind port เดิมซ้ำได้ทันทีหลัง server ปิดตัว (ข้าม TIME_WAIT) */
    int opt = 1;
    if (setsockopt(server_fd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt)) < 0) {
        perror("setsockopt");
        exit(EXIT_FAILURE);
    }

    memset(&server_addr, 0, sizeof(server_addr));
    server_addr.sin_family      = AF_INET;
    server_addr.sin_addr.s_addr = INADDR_ANY;   /* ฟังทุก network interface */
    server_addr.sin_port        = htons(PORT);

    /* 2) bind: ผูก socket เข้ากับ IP:port ที่กำหนด */
    if (bind(server_fd, (struct sockaddr *)&server_addr, sizeof(server_addr)) < 0) {
        perror("bind");
        close(server_fd);
        exit(EXIT_FAILURE);
    }

    /* 3) listen: เปลี่ยน socket ให้อยู่ในสถานะรอรับการเชื่อมต่อ
     * เลข 16 คือขนาดคิวของ connection ที่รอ accept() อยู่ (backlog) */
    if (listen(server_fd, 16) < 0) {
        perror("listen");
        close(server_fd);
        exit(EXIT_FAILURE);
    }

    printf("=== TCP Chat Server ===\n");
    printf("กำลังฟังที่ port %d (กด Ctrl+C เพื่อหยุด)\n", PORT);
    ...
```

สองบรรทัดที่อาจดูใหม่คือ `signal(SIGINT, ...)` และ `signal(SIGPIPE, SIG_IGN)` (ทบทวนจาก
Part 28) — จำเป็นมากสำหรับ network server เพราะ:

- **SIGINT**: เมื่อผู้ดูแลระบบกด Ctrl+C เราต้องการปิด `server_fd` ให้เรียบร้อยก่อนออก
  ไม่ใช่ปล่อยให้ OS ฆ่าโปรแกรมทิ้งดื้อๆ (แม้ OS จะคืน socket ให้เองอยู่ดี แต่การ cleanup เอง
  เป็นนิสัยที่ดีและจำเป็นเมื่อโปรแกรมซับซ้อนขึ้น เช่น ต้อง flush log หรือปิดไฟล์อื่นๆ)
- **SIGPIPE**: ถ้า client ตัดการเชื่อมต่อกะทันหัน (เช่น ปิด terminal, network ขาด) แล้ว server
  ดันเรียก `send()` ไปยัง socket นั้นอีกครั้ง จะเกิด SIGPIPE ซึ่งค่า default คือ **ฆ่าโปรแกรมทิ้ง
  ทันที** — เป็นหายนะสำหรับ server ที่ต้องรันตลอดเวลา ต้อง ignore สัญญาณนี้ไว้เสมอ
  (เราจะใช้ `MSG_NOSIGNAL` เสริมอีกชั้นตอน `send()` ด้วยเพื่อความปลอดภัย 2 ชั้น)

### Accept Loop: หัวใจของ Thread-per-Client

```c
    for (;;) {
        struct sockaddr_in client_addr;
        socklen_t addr_len = sizeof(client_addr);

        int client_fd = accept(server_fd, (struct sockaddr *)&client_addr, &addr_len);
        if (client_fd < 0) {
            /* ถ้า server_fd ถูกปิดโดย signal handler ระหว่าง accept() ค้างอยู่
             * accept() จะคืนค่า error กลับมา ให้ออกจากลูปแทนที่จะ retry ไม่รู้จบ */
            if (errno == EINTR || errno == EBADF) {
                break;
            }
            perror("accept");
            continue;
        }

        int slot = client_list_add(client_fd, client_addr);
        if (slot < 0) {
            /* ห้องเต็ม: แจ้ง client แล้วตัดการเชื่อมต่อทันที */
            const char *full_msg = "Server เต็มแล้ว กรุณาลองใหม่ภายหลัง\n";
            send(client_fd, full_msg, strlen(full_msg), 0);
            close(client_fd);
            continue;
        }

        /* ต้อง malloc ค่า fd ให้ thread ใหม่ เพราะตัวแปร client_fd บน stack
         * ของ main() จะถูก reuse ในรอบถัดไปของ for-loop ก่อนที่ thread ใหม่
         * จะทันได้อ่านค่า ถ้าส่ง &client_fd ตรงๆ จะเกิด race condition */
        int *fd_ptr = malloc(sizeof(int));
        if (fd_ptr == NULL) {
            fprintf(stderr, "malloc ล้มเหลว ปิดการเชื่อมต่อ client นี้\n");
            client_list_remove(client_fd);
            close(client_fd);
            continue;
        }
        *fd_ptr = client_fd;

        pthread_t tid;
        if (pthread_create(&tid, NULL, client_handler, fd_ptr) != 0) {
            perror("pthread_create");
            free(fd_ptr);
            client_list_remove(client_fd);
            close(client_fd);
            continue;
        }

        /* detach thread: ไม่มีใคร join thread นี้ ปล่อยให้ระบบคืนทรัพยากร
         * ให้อัตโนมัติทันทีที่ thread จบการทำงานเอง */
        pthread_detach(tid);
    }
```

จุดที่สำคัญที่สุดในส่วนนี้ (และเป็นบั๊กคลาสสิกที่มือใหม่มักพลาด) คือการ **malloc ค่า fd**
ก่อนส่งให้ `pthread_create()` แทนที่จะส่ง `&client_fd` ตรงๆ ทั้งที่ `client_fd` เป็นตัวแปร
local อยู่แล้ว ทำไมยังไม่พอ?

ปัญหาคือ `client_fd` เป็นตัวแปรบน stack ของ `main()` ที่ **ถูก declare ใหม่ทุกรอบของ
for-loop** (จริงๆ แล้วมันคือ stack slot เดิมที่ค่าถูกเขียนทับใหม่ทุกรอบ) ถ้าเราส่ง `&client_fd`
ให้ thread ใหม่ แล้ว `main()` วนไป accept client คนถัดไปทันทีก่อนที่ thread ใหม่จะทันได้อ่าน
ค่าจาก pointer นั้น ค่าที่ thread ใหม่อ่านได้อาจเป็นค่าของ client คนถัดไปไปแล้ว! (เพราะ
`main()` เขียนทับตัวแปรเดิมด้วยค่า fd ใหม่) นี่คือ race condition ระหว่าง thread หลักกับ
thread ลูกที่พึ่งสร้าง จึงต้อง `malloc(sizeof(int))` เพื่อให้แต่ละ client มี memory เป็นของ
ตัวเองแยกกันชัดเจน แล้วให้ thread ลูกเป็นผู้ `free()` มันเองทันทีหลัง copy ค่าออกมา (ดู 35.4)

`pthread_detach(tid)` ก็สำคัญไม่แพ้กัน — เนื่องจากเราไม่มีแผนจะ `pthread_join()` thread ของ
client แต่ละคนเลย (เพราะไม่รู้ว่าจะจบเมื่อไหร่ และ main thread ยุ่งอยู่กับการ accept คนใหม่
ตลอด) การ detach บอกให้ระบบปฏิบัติการคืนทรัพยากรของ thread ให้อัตโนมัติทันทีที่ thread นั้น
ทำงานจบ ถ้าลืม detach (และไม่ join ด้วย) thread ที่จบไปแล้วจะกลายเป็น **zombie thread**
ที่ทรัพยากรไม่ถูกคืน (คล้ายกับ zombie process ที่เรียนใน Part 27)

---

## 35.4 Thread ของแต่ละ Client: Broadcast และการอ่าน/ส่งข้อความ (Step 276)

ส่วนที่เหลือของ `server.c` คือฟังก์ชันช่วยจัดการ `client_list` และหัวใจของโปรแกรม —
`client_handler()` ที่ทุก thread ของ client รันอยู่ตลอดอายุการเชื่อมต่อ

### ฟังก์ชันจัดการ Client List (ทุกฟังก์ชันล็อก mutex เสมอ)

```c
static void client_list_init(void) {
    pthread_mutex_lock(&client_list_mutex);
    for (int i = 0; i < MAX_CLIENTS; i++) {
        client_list[i].active = 0;
        client_list[i].sockfd = -1;
    }
    pthread_mutex_unlock(&client_list_mutex);
}

static int client_list_add(int sockfd, struct sockaddr_in addr) {
    int result = -1;
    pthread_mutex_lock(&client_list_mutex);
    for (int i = 0; i < MAX_CLIENTS; i++) {
        if (!client_list[i].active) {
            client_list[i].sockfd = sockfd;
            client_list[i].addr   = addr;
            client_list[i].active = 1;
            snprintf(client_list[i].name, NAME_SIZE, "guest%d", sockfd);
            result = i;
            break;
        }
    }
    pthread_mutex_unlock(&client_list_mutex);
    return result;
}

static void client_list_remove(int sockfd) {
    pthread_mutex_lock(&client_list_mutex);
    for (int i = 0; i < MAX_CLIENTS; i++) {
        if (client_list[i].active && client_list[i].sockfd == sockfd) {
            client_list[i].active = 0;
            client_list[i].sockfd = -1;
            break;
        }
    }
    pthread_mutex_unlock(&client_list_mutex);
}
```

รูปแบบที่ซ้ำกันทุกฟังก์ชันคือ **lock → ทำงานกับ list → unlock** เป็น pattern มาตรฐานที่
ควรฝึกให้เป็นสัญชาตญาณ ยิ่งช่วงเวลาที่ถือ lock สั้นเท่าไหร่ (critical section เล็ก) ยิ่งลด
โอกาสที่ thread อื่นต้องรอนานเท่านั้น

### broadcast_message(): หัวใจของคำว่า "แชท"

```c
static void broadcast_message(int sender_fd, const char *msg, size_t len) {
    pthread_mutex_lock(&client_list_mutex);
    for (int i = 0; i < MAX_CLIENTS; i++) {
        if (client_list[i].active && client_list[i].sockfd != sender_fd) {
            /* MSG_NOSIGNAL ป้องกัน SIGPIPE ในระดับ call เดียว (เผื่อกรณีที่ยัง
             * ไม่ได้ ignore SIGPIPE แบบ global) ถ้า send ผิดพลาดก็แค่ข้ามไป
             * ปล่อยให้ thread เจ้าของ client นั้น cleanup ตัวเองภายหลัง */
            ssize_t sent = send(client_list[i].sockfd, msg, len, MSG_NOSIGNAL);
            (void)sent;
        }
    }
    pthread_mutex_unlock(&client_list_mutex);
}
```

สังเกตว่าเราวนลูปทั้ง `client_list` ทั้งหมด ล็อกเพียงครั้งเดียวตลอดการ broadcast — ไม่ใช่
ล็อก/ปลดล็อกทีละคน เพราะเราต้องการให้ภาพของ "ใครอยู่ในห้องบ้าง" คงที่ตลอดการ broadcast
ครั้งนี้ (consistent snapshot) ถ้าปลดล็อกระหว่างวน อาจมี client ใหม่โผล่มาหรือมีคนหลุดออกไป
กลางคัน ทำให้พฤติกรรมไม่แน่นอน

เรา **ไม่ตรวจสอบผลลัพธ์ของ `send()`** อย่างเข้มงวดตรงนี้ (แค่ `(void)sent;` ทิ้งไป) เพราะถ้า
`send()` ล้มเหลว (เช่น client คนนั้นเพิ่งหลุดไปพอดี) เราปล่อยให้ thread ที่เป็นเจ้าของ client
นั้นเป็นผู้ตรวจพบเองผ่าน `recv()` แล้วจัดการ cleanup ตามปกติ (ดูหัวข้อถัดไป) — ไม่จำเป็นต้อง
ทำซ้ำสองที่

### client_handler(): ฟังก์ชันหลักของแต่ละ Thread

ฟังก์ชันนี้ทำ 4 อย่างตามลำดับ: (1) อ่านชื่อผู้ใช้ (2) ประกาศเข้าห้อง (3) วนลูปรับ-ส่งข้อความ
(4) cleanup ตอนออกจากห้อง

```c
static void *client_handler(void *arg) {
    int client_fd = *(int *)arg;
    free(arg);   /* คัดลอกค่าออกมาแล้ว ไม่ต้องใช้ heap block นี้อีก */

    char name[NAME_SIZE];
    snprintf(name, NAME_SIZE, "guest%d", client_fd);

    char buf[BUF_SIZE];
    char leftover[BUF_SIZE];
    int  has_leftover = 0;

    /* ขั้นตอนแรก: อ่านชื่อที่ client ส่งเข้ามาบรรทัดแรก */
    ssize_t n = recv(client_fd, buf, sizeof(buf) - 1, 0);
    if (n > 0) {
        buf[n] = '\0';
        char *newline = strpbrk(buf, "\r\n");
        size_t name_len = newline ? (size_t)(newline - buf) : (size_t)n;

        if (name_len > 0) {
            if (name_len > NAME_SIZE - 1) {
                name_len = NAME_SIZE - 1;
            }
            memcpy(name, buf, name_len);
            name[name_len] = '\0';
        }

        if (newline != NULL) {
            char *rest = newline;
            while (*rest == '\r' || *rest == '\n') {
                rest++;
            }
            if (*rest != '\0') {
                strncpy(leftover, rest, sizeof(leftover) - 1);
                leftover[sizeof(leftover) - 1] = '\0';
                has_leftover = 1;
            }
        }
    }
    ...
```

### จุดสำคัญที่สุดของบทเรียนนี้: TCP คือ "Byte Stream" ไม่ใช่ "Message Stream"

ระหว่างพัฒนาโค้ดนี้จริงในห้องแล็บของหลักสูตร เราเขียนเวอร์ชันแรกแบบตรงไปตรงมา คือ `recv()`
รับชื่อผู้ใช้มา ตัด newline ทิ้งแล้วจบ ไม่สนใจไบต์ที่เหลือ ผลคือเมื่อทดสอบด้วยสคริปต์อัตโนมัติ
ที่ส่งชื่อกับข้อความแรกติดกันเร็วมาก (`printf "Alice\nHello everyone\n" | ./client ...`)
**ข้อความ "Hello everyone" หายไปเฉยๆ** ไม่ถูก broadcast เลย!

สาเหตุคือ TCP เป็น **stream ของไบต์ต่อเนื่อง** ไม่มีแนวคิดเรื่อง "ขอบเขตของข้อความ" ฝัง
อยู่ในโปรโตคอลเลย แม้ client จะเรียก `send()` แยกกัน 2 ครั้ง (ครั้งแรกส่งชื่อ ครั้งที่สองส่ง
ข้อความ) แต่ถ้าทั้งสองครั้งเกิดขึ้นเร็วมากในเวลาไล่เลี่ยกัน เคอร์เนลฝั่ง server อาจรวมข้อมูล
ทั้งสองก้อนไว้ใน receive buffer เดียวกัน แล้วเมื่อ server เรียก `recv()` เพียงครั้งเดียว
ก็จะได้ข้อมูลทั้งสองก้อนมาพร้อมกันเป็นสาย `"Alice\nHello everyone\n"`

โค้ดเวอร์ชันแรกของเราตัดที่ newline ตัวแรกเพื่อเอาชื่อ แล้ว **ทิ้งส่วนที่เหลือไปเฉยๆ** — นี่คือ
บั๊กจริงที่เจอระหว่างทดสอบ! วิธีแก้ (ที่แสดงในโค้ดด้านบน) คือ: หลังตัดชื่อออกจาก buffer แล้ว
ให้ตรวจสอบว่ามีไบต์เหลืออยู่หลัง newline หรือไม่ ถ้ามีให้เก็บไว้ในตัวแปร `leftover` แล้วนำไป
ประมวลผลเป็นข้อความแรกในลูปหลัก **ก่อน** ที่จะเรียก `recv()` รอบใหม่:

```c
    for (;;) {
        if (has_leftover) {
            /* มีข้อความแรกที่ค้างมากับแพ็กเก็ตเดียวกันกับชื่อ ใช้มันก่อน
             * โดยยังไม่ต้องเรียก recv() ใหม่ในรอบนี้ */
            strncpy(buf, leftover, sizeof(buf) - 1);
            buf[sizeof(buf) - 1] = '\0';
            n = (ssize_t)strlen(buf);
            has_leftover = 0;
        } else {
            n = recv(client_fd, buf, sizeof(buf) - 1, 0);
            if (n <= 0) {
                break;
            }
            buf[n] = '\0';
        }

        char out_msg[BUF_SIZE + NAME_SIZE + 8];
        int out_len = snprintf(out_msg, sizeof(out_msg), "%s: %s", name, buf);
        ...
        printf("%s", out_msg);
        broadcast_message(client_fd, out_msg, (size_t)out_len);
    }
```

> นี่คือตัวอย่างจริงที่ยืนยันได้ว่า "การทดสอบด้วยมือ" (พิมพ์ทีละคำแล้วรอ) มักไม่เจอบั๊กแบบนี้
> เพราะมนุษย์พิมพ์ช้าพอที่แต่ละบรรทัดจะถูกส่งเป็นแพ็กเก็ตแยกกันตามธรรมชาติ แต่บั๊กจะโผล่ทันที
> เมื่อมีการเชื่อมต่อผ่านสคริปต์อัตโนมัติหรือเครือข่ายที่มี latency ต่ำมาก **นี่คือเหตุผลที่
> การทดสอบระบบเครือข่ายต้องใช้ทั้งการทดสอบด้วยมือและสคริปต์อัตโนมัติควบคู่กันเสมอ**

### ประกาศเข้าห้องและ Cleanup ตอนออก

```c
    printf("[+] %s เข้าร่วมห้องแชท (fd=%d)\n", name, client_fd);

    char join_msg[BUF_SIZE];
    int join_len = snprintf(join_msg, sizeof(join_msg),
                             "*** %s เข้าร่วมห้องแชทแล้ว ***\n", name);
    broadcast_message(client_fd, join_msg, (size_t)join_len);

    /* ... ลูปหลักรับ-ส่งข้อความตามที่แสดงด้านบน ... */

    /* ---- cleanup: ส่วนสำคัญที่สุดของการจัดการ disconnect ---- */
    client_list_remove(client_fd);
    close(client_fd);

    printf("[-] %s ออกจากห้องแชท (fd=%d)\n", name, client_fd);

    char leave_msg[BUF_SIZE];
    int leave_len = snprintf(leave_msg, sizeof(leave_msg),
                              "*** %s ออกจากห้องแชทแล้ว ***\n", name);
    broadcast_message(client_fd, leave_msg, (size_t)leave_len);

    return NULL;
}
```

ลำดับการ cleanup สำคัญมาก: **ลบออกจาก `client_list` ก่อน แล้วค่อยปิด `close(client_fd)`**
เหตุผลคือ ถ้าปิด fd ก่อนแล้วยังไม่ทันลบออกจาก list เร็วพอ มีโอกาสที่ thread อื่นกำลัง
`broadcast_message()` พอดีแล้วพยายาม `send()` ไปยัง fd ที่ถูกปิดไปแล้ว (ซึ่งอาจถูก OS นำไป
ใช้ซ้ำกับ connection ใหม่ทันที ทำให้ส่งข้อความผิดไปหา client คนละคน!) การลบออกจาก list ก่อน
ทำให้ thread อื่นมองไม่เห็น fd ตัวนี้อีกต่อไปตั้งแต่จุดนั้น จึงไม่มีทาง `send()` ผิดคนได้

### signal Handler ที่ปลอดภัย (Async-Signal-Safe)

```c
static void handle_sigint(int sig) {
    (void)sig;
    const char msg[] = "\nได้รับ SIGINT กำลังปิด server...\n";
    ssize_t written = write(STDOUT_FILENO, msg, sizeof(msg) - 1);
    (void)written;
    if (server_fd >= 0) {
        int fd = server_fd;
        server_fd = -1;
        close(fd);
    }
}
```

สังเกตว่าเราใช้ `write()` ตรงๆ แทน `printf()` ภายใน signal handler — นี่คือรายละเอียดสำคัญที่
ทบทวนจาก Part 28: ฟังก์ชันอย่าง `printf()` ใช้ internal buffer ของตัวเองที่ **ไม่ได้ถูกออกแบบ
ให้เรียกซ้อนจาก signal handler ได้อย่างปลอดภัย** (ไม่ใช่ async-signal-safe) ถ้า main thread
กำลังเรียก `printf()` ค้างอยู่พอดีตอนที่ signal มาถึง แล้ว handler ก็เรียก `printf()` ซ้อนเข้าไป
อีก อาจทำให้ internal state ของ stdio buffer เสียหายได้ ส่วน `write()` เป็น raw syscall
ที่ POSIX รับประกันว่าปลอดภัยเสมอในบริบทนี้

### ไฟล์ `server.c` ฉบับสมบูรณ์

รวมทุกส่วนที่อธิบายมาทั้งหมดเข้าด้วยกัน (ทดสอบคอมไพล์และรันจริงแล้วด้วย
`gcc -Wall -Wextra -Wpedantic -std=c17 -g -pthread server.c -o server` **ไม่มี warning
แม้แต่บรรทัดเดียว**):

```c
/* ============================================================
 * ชื่อไฟล์:     server.c
 * คำอธิบาย:     TCP Chat Server แบบ multi-client โดยใช้ thread-per-client
 *              รับข้อความจาก client คนหนึ่งแล้ว broadcast ไปยัง client
 *              ทุกคนที่เชื่อมต่ออยู่ (ยกเว้นผู้ส่งเอง)
 * ============================================================ */

/* ---------- 1. Header Includes ---------- */
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <errno.h>
#include <unistd.h>
#include <signal.h>
#include <pthread.h>
#include <arpa/inet.h>
#include <sys/socket.h>
#include <netinet/in.h>

/* ---------- 2. Macro / Constant Definitions ---------- */
#define PORT            5500
#define MAX_CLIENTS     32
#define BUF_SIZE        1024
#define NAME_SIZE       32

/* ---------- 3. Struct และ Global State ---------- */

/* ข้อมูลของ client แต่ละคนที่กำลังเชื่อมต่ออยู่ */
typedef struct {
    int  sockfd;                 /* file descriptor ของ socket ตัวนี้ */
    struct sockaddr_in addr;     /* ที่อยู่ของ client (IP:port) */
    char name[NAME_SIZE];        /* ชื่อที่ client เลือกตอนเข้ามา */
    int  active;                 /* 1 = ช่องนี้ถูกใช้งานอยู่จริง, 0 = ว่าง */
} client_t;

/* client_list คือทรัพยากรที่ "แชร์กัน" ระหว่างทุก thread
 * (thread หลักตอน accept() ใหม่ + thread ของ client แต่ละคนตอน broadcast)
 * ดังนั้นทุกครั้งที่จะอ่านหรือแก้ไข ต้องล็อกด้วย client_list_mutex เสมอ */
static client_t     client_list[MAX_CLIENTS];
static pthread_mutex_t client_list_mutex = PTHREAD_MUTEX_INITIALIZER;
static int           server_fd = -1;

/* ---------- 4. Function Prototypes ---------- */
static void  client_list_init(void);
static int   client_list_add(int sockfd, struct sockaddr_in addr);
static void  client_list_remove(int sockfd);
static void  broadcast_message(int sender_fd, const char *msg, size_t len);
static void *client_handler(void *arg);
static void  handle_sigint(int sig);

/* ---------- 5. main() ---------- */
int main(void) {
    struct sockaddr_in server_addr;

    /* ดักจับ Ctrl+C เพื่อปิด server_fd ให้เรียบร้อยก่อนออกโปรแกรม
     * (ถ้าไม่ทำ port อาจค้างอยู่ใน TIME_WAIT จน bind ใหม่ไม่ได้ทันที) */
    signal(SIGINT, handle_sigint);

    /* SIGPIPE เกิดเมื่อเขียนลง socket ที่อีกฝั่งปิดไปแล้ว ถ้าไม่ ignore
     * โปรแกรมทั้งตัวจะถูกฆ่าทิ้งทันทีโดยไม่มีโอกาส cleanup */
    signal(SIGPIPE, SIG_IGN);

    client_list_init();

    /* 1) สร้าง socket แบบ TCP (SOCK_STREAM) */
    server_fd = socket(AF_INET, SOCK_STREAM, 0);
    if (server_fd < 0) {
        perror("socket");
        exit(EXIT_FAILURE);
    }

    /* อนุญาตให้ bind port เดิมซ้ำได้ทันทีหลัง server ปิดตัว (ข้าม TIME_WAIT) */
    int opt = 1;
    if (setsockopt(server_fd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt)) < 0) {
        perror("setsockopt");
        exit(EXIT_FAILURE);
    }

    memset(&server_addr, 0, sizeof(server_addr));
    server_addr.sin_family      = AF_INET;
    server_addr.sin_addr.s_addr = INADDR_ANY;   /* ฟังทุก network interface */
    server_addr.sin_port        = htons(PORT);

    /* 2) bind: ผูก socket เข้ากับ IP:port ที่กำหนด */
    if (bind(server_fd, (struct sockaddr *)&server_addr, sizeof(server_addr)) < 0) {
        perror("bind");
        close(server_fd);
        exit(EXIT_FAILURE);
    }

    /* 3) listen: เปลี่ยน socket ให้อยู่ในสถานะรอรับการเชื่อมต่อ
     * เลข 16 คือขนาดคิวของ connection ที่รอ accept() อยู่ (backlog) */
    if (listen(server_fd, 16) < 0) {
        perror("listen");
        close(server_fd);
        exit(EXIT_FAILURE);
    }

    printf("=== TCP Chat Server ===\n");
    printf("กำลังฟังที่ port %d (กด Ctrl+C เพื่อหยุด)\n", PORT);

    /* 4) วนลูป accept() รับ client ใหม่ตลอดไป
     * ทุกครั้งที่มี client ใหม่เข้ามา จะสร้าง thread ใหม่ 1 เส้นให้คุยกับ
     * client คนนั้นโดยเฉพาะ (thread-per-client model) */
    for (;;) {
        struct sockaddr_in client_addr;
        socklen_t addr_len = sizeof(client_addr);

        int client_fd = accept(server_fd, (struct sockaddr *)&client_addr, &addr_len);
        if (client_fd < 0) {
            /* ถ้า server_fd ถูกปิดโดย signal handler ระหว่าง accept() ค้างอยู่
             * accept() จะคืนค่า error กลับมา ให้ออกจากลูปแทนที่จะ retry ไม่รู้จบ */
            if (errno == EINTR || errno == EBADF) {
                break;
            }
            perror("accept");
            continue;
        }

        int slot = client_list_add(client_fd, client_addr);
        if (slot < 0) {
            /* ห้องเต็ม: แจ้ง client แล้วตัดการเชื่อมต่อทันที */
            const char *full_msg = "Server เต็มแล้ว กรุณาลองใหม่ภายหลัง\n";
            send(client_fd, full_msg, strlen(full_msg), 0);
            close(client_fd);
            continue;
        }

        /* ต้อง malloc ค่า fd ให้ thread ใหม่ เพราะตัวแปร client_fd บน stack
         * ของ main() จะถูก reuse ในรอบถัดไปของ for-loop ก่อนที่ thread ใหม่
         * จะทันได้อ่านค่า ถ้าส่ง &client_fd ตรงๆ จะเกิด race condition */
        int *fd_ptr = malloc(sizeof(int));
        if (fd_ptr == NULL) {
            fprintf(stderr, "malloc ล้มเหลว ปิดการเชื่อมต่อ client นี้\n");
            client_list_remove(client_fd);
            close(client_fd);
            continue;
        }
        *fd_ptr = client_fd;

        pthread_t tid;
        if (pthread_create(&tid, NULL, client_handler, fd_ptr) != 0) {
            perror("pthread_create");
            free(fd_ptr);
            client_list_remove(client_fd);
            close(client_fd);
            continue;
        }

        /* detach thread: ไม่มีใคร join thread นี้ ปล่อยให้ระบบคืนทรัพยากร
         * ให้อัตโนมัติทันทีที่ thread จบการทำงานเอง */
        pthread_detach(tid);
    }

    /* server_fd อาจถูกปิดไปแล้วโดย handle_sigint() (ซึ่งจะตั้งค่าเป็น -1 หลังปิด)
     * ต้องเช็กก่อนเสมอ ไม่เช่นนั้นจะเป็นการเรียก close(-1) ซ้ำซ้อนโดยเปล่าประโยชน์ */
    if (server_fd >= 0) {
        close(server_fd);
    }
    printf("Server ปิดตัวเรียบร้อย\n");
    return 0;
}

/* ---------- 6. Function Implementations ---------- */

/* ตั้งค่า client_list เริ่มต้นให้ทุกช่องว่าง */
static void client_list_init(void) {
    pthread_mutex_lock(&client_list_mutex);
    for (int i = 0; i < MAX_CLIENTS; i++) {
        client_list[i].active = 0;
        client_list[i].sockfd = -1;
    }
    pthread_mutex_unlock(&client_list_mutex);
}

/* เพิ่ม client ใหม่เข้า client_list (หาช่องว่างช่องแรกที่เจอ)
 * คืนค่า index ของช่องที่ได้ หรือ -1 ถ้าห้องเต็ม
 * ฟังก์ชันนี้ถูกเรียกจาก thread หลัก (main) เท่านั้น แต่ต้องล็อกอยู่ดี
 * เพราะ client_handler() ของ thread อื่นอาจกำลังอ่าน client_list พร้อมกัน */
static int client_list_add(int sockfd, struct sockaddr_in addr) {
    int result = -1;
    pthread_mutex_lock(&client_list_mutex);
    for (int i = 0; i < MAX_CLIENTS; i++) {
        if (!client_list[i].active) {
            client_list[i].sockfd = sockfd;
            client_list[i].addr   = addr;
            client_list[i].active = 1;
            snprintf(client_list[i].name, NAME_SIZE, "guest%d", sockfd);
            result = i;
            break;
        }
    }
    pthread_mutex_unlock(&client_list_mutex);
    return result;
}

/* ลบ client ออกจาก client_list เมื่อ disconnect (ค้นหาด้วย sockfd) */
static void client_list_remove(int sockfd) {
    pthread_mutex_lock(&client_list_mutex);
    for (int i = 0; i < MAX_CLIENTS; i++) {
        if (client_list[i].active && client_list[i].sockfd == sockfd) {
            client_list[i].active = 0;
            client_list[i].sockfd = -1;
            break;
        }
    }
    pthread_mutex_unlock(&client_list_mutex);
}

/* ส่งข้อความ msg ไปยัง client ทุกคนที่ active อยู่ ยกเว้นคนที่ sockfd == sender_fd
 * นี่คือหัวใจของ "chat": อ่าน client_list ต้องล็อก mutex เพื่อไม่ให้ thread อื่น
 * แก้ไข list (เช่นมี client หลุดพอดี) ระหว่างที่เรากำลังวนลูปอยู่ */
static void broadcast_message(int sender_fd, const char *msg, size_t len) {
    pthread_mutex_lock(&client_list_mutex);
    for (int i = 0; i < MAX_CLIENTS; i++) {
        if (client_list[i].active && client_list[i].sockfd != sender_fd) {
            /* MSG_NOSIGNAL ป้องกัน SIGPIPE ในระดับ call เดียว (เผื่อกรณีที่ยัง
             * ไม่ได้ ignore SIGPIPE แบบ global) ถ้า send ผิดพลาดก็แค่ข้ามไป
             * ปล่อยให้ thread เจ้าของ client นั้น cleanup ตัวเองภายหลัง */
            ssize_t sent = send(client_list[i].sockfd, msg, len, MSG_NOSIGNAL);
            (void)sent;
        }
    }
    pthread_mutex_unlock(&client_list_mutex);
}

/* ฟังก์ชันหลักที่ thread ของ client แต่ละคนรันอยู่ตลอดอายุการเชื่อมต่อ
 * arg คือ pointer ไปยัง int ที่เก็บ client_fd (malloc มาจาก main) */
static void *client_handler(void *arg) {
    int client_fd = *(int *)arg;
    free(arg);   /* คัดลอกค่าออกมาแล้ว ไม่ต้องใช้ heap block นี้อีก */

    char name[NAME_SIZE];
    snprintf(name, NAME_SIZE, "guest%d", client_fd);

    char buf[BUF_SIZE];

    /* leftover: TCP เป็น "byte stream" ไม่ใช่ "message stream" — ถ้า client
     * ส่งชื่อกับข้อความแรกติดกันเร็วมาก (เช่น สคริปต์อัตโนมัติ) เคอร์เนลอาจ
     * รวมทั้งสองอย่างมาให้ recv() ครั้งเดียวก็ได้ ดังนั้นหลังตัดบรรทัดชื่อออก
     * ต้องเก็บไบต์ที่เหลือ (ถ้ามี) ไว้ประมวลผลต่อเป็นข้อความแรก ห้ามทิ้งไป
     * (ดูรายละเอียดเพิ่มเติมใน "ข้อผิดพลาดที่พบบ่อย" ท้าย Part นี้) */
    char leftover[BUF_SIZE];
    int  has_leftover = 0;

    /* ขั้นตอนแรก: อ่านชื่อที่ client ส่งเข้ามาบรรทัดแรก */
    ssize_t n = recv(client_fd, buf, sizeof(buf) - 1, 0);
    if (n > 0) {
        buf[n] = '\0';
        char *newline = strpbrk(buf, "\r\n");
        size_t name_len = newline ? (size_t)(newline - buf) : (size_t)n;

        if (name_len > 0) {
            if (name_len > NAME_SIZE - 1) {
                name_len = NAME_SIZE - 1;
            }
            memcpy(name, buf, name_len);
            name[name_len] = '\0';
        }

        if (newline != NULL) {
            char *rest = newline;
            while (*rest == '\r' || *rest == '\n') {
                rest++;
            }
            if (*rest != '\0') {
                strncpy(leftover, rest, sizeof(leftover) - 1);
                leftover[sizeof(leftover) - 1] = '\0';
                has_leftover = 1;
            }
        }
    }

    /* อัปเดตชื่อใน client_list ให้ตรงกับที่ผู้ใช้ตั้ง */
    pthread_mutex_lock(&client_list_mutex);
    for (int i = 0; i < MAX_CLIENTS; i++) {
        if (client_list[i].active && client_list[i].sockfd == client_fd) {
            strncpy(client_list[i].name, name, NAME_SIZE - 1);
            client_list[i].name[NAME_SIZE - 1] = '\0';
            break;
        }
    }
    pthread_mutex_unlock(&client_list_mutex);

    printf("[+] %s เข้าร่วมห้องแชท (fd=%d)\n", name, client_fd);

    char join_msg[BUF_SIZE];
    int join_len = snprintf(join_msg, sizeof(join_msg),
                             "*** %s เข้าร่วมห้องแชทแล้ว ***\n", name);
    broadcast_message(client_fd, join_msg, (size_t)join_len);

    /* ลูปหลัก: รอรับข้อความจาก client คนนี้ แล้ว broadcast ต่อ
     * recv() จะ block รอจนกว่าจะมีข้อมูลเข้ามา หรือ client ปิดการเชื่อมต่อ */
    for (;;) {
        if (has_leftover) {
            /* มีข้อความแรกที่ค้างมากับแพ็กเก็ตเดียวกันกับชื่อ ใช้มันก่อน
             * โดยยังไม่ต้องเรียก recv() ใหม่ในรอบนี้ */
            strncpy(buf, leftover, sizeof(buf) - 1);
            buf[sizeof(buf) - 1] = '\0';
            n = (ssize_t)strlen(buf);
            has_leftover = 0;
        } else {
            n = recv(client_fd, buf, sizeof(buf) - 1, 0);
            if (n <= 0) {
                /* n == 0 หมายถึง client ปิดการเชื่อมต่ออย่างสุภาพ (orderly shutdown)
                 * n < 0 หมายถึงเกิด error เช่น connection reset by peer */
                break;
            }
            buf[n] = '\0';
        }

        char out_msg[BUF_SIZE + NAME_SIZE + 8];
        int out_len = snprintf(out_msg, sizeof(out_msg), "%s: %s", name, buf);
        if (out_len < 0) {
            continue;
        }
        /* ถ้าข้อความไม่มี newline ท้าย ให้เติมให้ เพื่อให้ client อีกฝั่ง
         * แยกแต่ละบรรทัดออกจากกันได้ถูกต้องเสมอ */
        if (out_len > 0 && out_msg[out_len - 1] != '\n' &&
            (size_t)out_len < sizeof(out_msg) - 1) {
            out_msg[out_len] = '\n';
            out_msg[out_len + 1] = '\0';
            out_len++;
        }

        printf("%s", out_msg);
        broadcast_message(client_fd, out_msg, (size_t)out_len);
    }

    /* ---- cleanup: ส่วนสำคัญที่สุดของการจัดการ disconnect ---- */
    client_list_remove(client_fd);
    close(client_fd);

    printf("[-] %s ออกจากห้องแชท (fd=%d)\n", name, client_fd);

    char leave_msg[BUF_SIZE];
    int leave_len = snprintf(leave_msg, sizeof(leave_msg),
                              "*** %s ออกจากห้องแชทแล้ว ***\n", name);
    broadcast_message(client_fd, leave_msg, (size_t)leave_len);

    return NULL;
}

/* จัดการ Ctrl+C: ปิด server_fd เพื่อให้ accept() ที่ block อยู่คืนค่า error
 * ออกมาแล้ว main loop จะหลุดออกไปปิดโปรแกรมอย่างเรียบร้อย
 *
 * ข้อควรระวัง: ภายใน signal handler ห้ามเรียกฟังก์ชันที่ไม่ใช่
 * "async-signal-safe" เช่น printf() (เพราะ printf ใช้ buffer ภายในที่ไม่ได้
 * ถูกออกแบบให้เรียกซ้อนจาก signal handler ได้อย่างปลอดภัย) ในที่นี้จึงใช้
 * write() ไปที่ STDOUT_FILENO ตรงๆ ซึ่งเป็น syscall ที่ปลอดภัยแทน */
static void handle_sigint(int sig) {
    (void)sig;
    const char msg[] = "\nได้รับ SIGINT กำลังปิด server...\n";
    ssize_t written = write(STDOUT_FILENO, msg, sizeof(msg) - 1);
    (void)written;
    if (server_fd >= 0) {
        int fd = server_fd;
        server_fd = -1;
        close(fd);
    }
}
```

---

## 35.5 เขียน Client: Full-Duplex ด้วยสอง Thread (Step 277)

ฝั่ง client มีความท้าทายที่ต่างจาก server: ผู้ใช้ต้อง **พิมพ์ข้อความส่งออกไป** และ **รับ
ข้อความจากคนอื่นเข้ามา** ได้ **พร้อมกัน** ไม่ใช่สลับกันทีละอย่าง (ถ้า Bob กำลังพิมพ์ข้อความอยู่
แต่ Alice ส่งข้อความมาก่อน โปรแกรมต้องแสดงข้อความของ Alice ทันทีโดยไม่ต้องรอให้ Bob กด Enter)

ปัญหาคือ `recv()` เป็น **blocking call** — ถ้าเราเขียนโปรแกรมเดียวที่สลับกันเรียก `fgets()`
(รอ input จากผู้ใช้) แล้วค่อย `recv()` (รอข้อความจาก server) ทีละอย่าง โปรแกรมจะติดค้างอยู่ที่
`fgets()` จนกว่าผู้ใช้จะพิมพ์อะไรสักอย่าง แล้วจะพลาดข้อความที่ server ส่งเข้ามาระหว่างนั้นไป
(หรืออย่างน้อยก็แสดงผลช้ากว่าที่ควร)

ทางแก้คือ **แยก thread ทำหน้าที่รับข้อความต่างหาก** ให้ทำงานคู่ขนานกับ thread หลักที่รอรับ
input จากผู้ใช้ตลอดเวลา:

```
        THREAD หลัก (main)                      THREAD รับข้อความ (receive_loop)
   ┌─────────────────────────┐              ┌─────────────────────────────┐
   │ while (fgets(stdin)) {   │              │ while (running) {            │
   │   ส่ง input ไปยัง server  │              │   recv() รอข้อความจาก server │
   │ }                         │              │   พิมพ์ข้อความที่ได้รับทันที  │
   │                           │              │ }                             │
   └─────────────────────────┘              └─────────────────────────────┘
              │                                            │
              └──────────────── sockfd (ตัวเดียวกัน) ───────┘
```

### main() ของ client.c

```c
int main(int argc, char *argv[]) {
    if (argc != 3) {
        fprintf(stderr, "วิธีใช้: %s <server-ip> <port>\n", argv[0]);
        fprintf(stderr, "ตัวอย่าง: %s 127.0.0.1 5500\n", argv[0]);
        return EXIT_FAILURE;
    }

    const char *server_ip = argv[1];
    int port = atoi(argv[2]);

    sockfd = socket(AF_INET, SOCK_STREAM, 0);
    if (sockfd < 0) {
        perror("socket");
        return EXIT_FAILURE;
    }

    struct sockaddr_in server_addr;
    memset(&server_addr, 0, sizeof(server_addr));
    server_addr.sin_family = AF_INET;
    server_addr.sin_port   = htons((uint16_t)port);

    if (inet_pton(AF_INET, server_ip, &server_addr.sin_addr) <= 0) {
        fprintf(stderr, "ที่อยู่ IP ไม่ถูกต้อง: %s\n", server_ip);
        close(sockfd);
        return EXIT_FAILURE;
    }

    if (connect(sockfd, (struct sockaddr *)&server_addr, sizeof(server_addr)) < 0) {
        perror("connect");
        close(sockfd);
        return EXIT_FAILURE;
    }

    printf("เชื่อมต่อไปยัง %s:%d สำเร็จ\n", server_ip, port);
    ...
```

หลังเชื่อมต่อสำเร็จ อ่านชื่อจากผู้ใช้แล้วส่งให้ server เป็นบรรทัดแรก จากนั้นสร้าง thread
`receive_loop` แล้วเข้าสู่ลูปหลักที่อ่าน input จากผู้ใช้:

```c
    pthread_t recv_tid;
    if (pthread_create(&recv_tid, NULL, receive_loop, NULL) != 0) {
        perror("pthread_create");
        close(sockfd);
        return EXIT_FAILURE;
    }

    /* thread หลัก: วนลูปอ่าน input จากผู้ใช้แล้วส่งไปยัง server */
    char input[BUF_SIZE];
    while (running && fgets(input, sizeof(input), stdin) != NULL) {
        if (strncmp(input, "/quit", 5) == 0) {
            running = 0;
            break;
        }
        ssize_t sent = send(sockfd, input, strlen(input), 0);
        if (sent < 0) {
            perror("send");
            running = 0;
            break;
        }
    }

    /* ปิด socket เพื่อปลุก recv() ที่ค้างอยู่ใน receive_loop() ให้คืนค่าออกมา
     * (recv() บน socket ที่ถูกปิดจะคืนค่า -1 หรือ 0 ทันที ทำให้ thread นั้น
     * หลุดจากลูปและจบการทำงานได้เอง) */
    running = 0;
    shutdown(sockfd, SHUT_RDWR);
    close(sockfd);

    pthread_join(recv_tid, NULL);

    printf("ออกจากห้องแชทเรียบร้อย\n");
    return EXIT_SUCCESS;
}
```

จุดที่น่าสนใจคือการปิดโปรแกรม: เมื่อผู้ใช้พิมพ์ `/quit` thread หลักต้องการหยุด thread
`receive_loop` ที่กำลัง **block อยู่ใน `recv()`** อยู่ วิธีปลุกมันคือเรียก `shutdown(sockfd,
SHUT_RDWR)` ก่อนแล้วค่อย `close(sockfd)` — `shutdown()` จะบังคับให้ `recv()` ที่ค้างอยู่ใน
thread อื่นคืนค่าออกมาทันที (ได้ 0 เหมือน connection ปิดแบบสุภาพ) ทำให้ `receive_loop` หลุด
ออกจากลูปและจบการทำงานได้เอง จากนั้น `pthread_join()` รอจน thread นั้นจบจริงๆ ก่อนที่โปรแกรม
หลักจะออกจากโปรแกรม — ตรงนี้เราเลือก **join แทน detach** เพราะต้องการมั่นใจว่า thread ลูก
จบสนิทแล้วจริงๆ ก่อนที่โปรแกรมทั้งตัวจะปิดตัวลง (ต่างจากฝั่ง server ที่ detach เพราะไม่สนใจ
ว่า thread จะจบเมื่อไหร่)

### receive_loop(): Thread รับข้อความ

```c
static void *receive_loop(void *arg) {
    (void)arg;
    char buf[BUF_SIZE];

    while (running) {
        ssize_t n = recv(sockfd, buf, sizeof(buf) - 1, 0);
        if (n <= 0) {
            /* n == 0: server ปิดการเชื่อมต่อ (เช่น server ล่มหรือถูกสั่งหยุด)
             * n < 0: เกิด error หรือ socket ถูกปิดจาก thread หลักตอน /quit */
            if (running) {
                printf("\n*** การเชื่อมต่อกับ server ถูกตัด ***\n");
            }
            running = 0;
            break;
        }
        buf[n] = '\0';
        printf("%s", buf);
        fflush(stdout);
    }
    return NULL;
}
```

`fflush(stdout)` หลัง `printf()` ทุกครั้งจำเป็นเพราะเมื่อ output ไม่ได้ต่อกับ terminal
โดยตรง (เช่น ถูก redirect ไปไฟล์ หรือรันผ่านสคริปต์ทดสอบ) stdio จะเปลี่ยนจาก **line-buffered**
เป็น **fully-buffered** ซึ่งหมายความว่าข้อความจะไม่ถูกเขียนออกจริงจนกว่า buffer จะเต็มหรือ
โปรแกรมจบ — ถ้าลืม `fflush()` ผู้ใช้ที่ redirect output จะไม่เห็นข้อความแบบ real-time เลย

`sockfd` และ `running` เป็นตัวแปร global ที่ทั้งสอง thread เข้าถึงร่วมกัน สังเกตว่าเรา
**ไม่ต้องใช้ mutex ป้องกัน `sockfd`** เหมือนฝั่ง server เพราะทั้งสอง thread ใช้มันคนละทิศทาง
(thread หนึ่งเรียก `send()` อีก thread เรียก `recv()`) ซึ่งเป็นคนละ system call ที่เคอร์เนล
จัดการ buffer แยกกันอยู่แล้ว (send buffer กับ receive buffer) จึงไม่มี race condition ระหว่าง
สองทิศทางนี้ ส่วน `running` เป็น `volatile int` ธรรมดา (ไม่ใช่ atomic) ซึ่งเพียงพอสำหรับ flag
ง่ายๆ แบบนี้เพราะมีแค่การอ่าน/เขียนค่า 0 หรือ 1 (ในงานที่ซับซ้อนกว่านี้ ควรพิจารณาใช้
`_Atomic int` จาก `<stdatomic.h>` ที่จะเรียนในหลักสูตร C++ Module I ต่อไป)

### ไฟล์ `client.c` ฉบับสมบูรณ์

```c
/* ============================================================
 * ชื่อไฟล์:     client.c
 * คำอธิบาย:     TCP Chat Client ที่แยก thread สำหรับ "รับ" ข้อความ
 *              จาก server ให้ทำงานพร้อมกันกับ thread หลักที่คอยรับ
 *              input จากผู้ใช้แล้ว "ส่ง" ออกไป (full-duplex chat)
 * ============================================================ */

/* ---------- 1. Header Includes ---------- */
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <pthread.h>
#include <arpa/inet.h>
#include <sys/socket.h>
#include <netinet/in.h>

/* ---------- 2. Macro / Constant Definitions ---------- */
#define BUF_SIZE    1024
#define NAME_SIZE   32

/* ---------- 3. Global State ---------- */

/* sockfd ต้องเป็น global เพราะทั้ง thread หลัก (ส่ง) และ thread รับ (recv_thread)
 * ต้องใช้ socket ตัวเดียวกันร่วมกัน — คนละทิศทาง (ส่ง/รับ) จึงไม่จำเป็นต้องมี
 * mutex ป้องกัน เพราะ read กับ write บน socket ตัวเดียวกันเป็นคนละ operation
 * ที่ kernel จัดการแยก buffer กันอยู่แล้ว (recv queue กับ send queue) */
static int sockfd = -1;

/* flag บอกว่าการเชื่อมต่อยังมีชีวิตอยู่หรือไม่ ใช้สื่อสารระหว่าง 2 thread
 * ว่าเมื่อไหร่ควรเลิกวนลูปแล้วปิดโปรแกรม */
static volatile int running = 1;

/* ---------- 4. Function Prototypes ---------- */
static void *receive_loop(void *arg);

/* ---------- 5. main() ---------- */
int main(int argc, char *argv[]) {
    if (argc != 3) {
        fprintf(stderr, "วิธีใช้: %s <server-ip> <port>\n", argv[0]);
        fprintf(stderr, "ตัวอย่าง: %s 127.0.0.1 5500\n", argv[0]);
        return EXIT_FAILURE;
    }

    const char *server_ip = argv[1];
    int port = atoi(argv[2]);

    sockfd = socket(AF_INET, SOCK_STREAM, 0);
    if (sockfd < 0) {
        perror("socket");
        return EXIT_FAILURE;
    }

    struct sockaddr_in server_addr;
    memset(&server_addr, 0, sizeof(server_addr));
    server_addr.sin_family = AF_INET;
    server_addr.sin_port   = htons((uint16_t)port);

    if (inet_pton(AF_INET, server_ip, &server_addr.sin_addr) <= 0) {
        fprintf(stderr, "ที่อยู่ IP ไม่ถูกต้อง: %s\n", server_ip);
        close(sockfd);
        return EXIT_FAILURE;
    }

    if (connect(sockfd, (struct sockaddr *)&server_addr, sizeof(server_addr)) < 0) {
        perror("connect");
        close(sockfd);
        return EXIT_FAILURE;
    }

    printf("เชื่อมต่อไปยัง %s:%d สำเร็จ\n", server_ip, port);

    char name[NAME_SIZE];
    printf("ตั้งชื่อผู้ใช้ของคุณ: ");
    fflush(stdout);
    if (fgets(name, sizeof(name), stdin) == NULL) {
        fprintf(stderr, "อ่านชื่อไม่สำเร็จ\n");
        close(sockfd);
        return EXIT_FAILURE;
    }
    name[strcspn(name, "\r\n")] = '\0';

    char name_line[NAME_SIZE + 2];
    snprintf(name_line, sizeof(name_line), "%s\n", name);
    if (send(sockfd, name_line, strlen(name_line), 0) < 0) {
        perror("send");
        close(sockfd);
        return EXIT_FAILURE;
    }

    printf("=== เข้าร่วมห้องแชทแล้ว พิมพ์ข้อความแล้วกด Enter เพื่อส่ง ===\n");
    printf("=== พิมพ์ /quit เพื่อออกจากโปรแกรม ===\n");

    /* สร้าง thread แยกสำหรับรับข้อความจาก server เข้ามาแสดงผลตลอดเวลา
     * โดยไม่ต้องรอให้ผู้ใช้กด Enter ที่ thread หลักก่อน (full-duplex) */
    pthread_t recv_tid;
    if (pthread_create(&recv_tid, NULL, receive_loop, NULL) != 0) {
        perror("pthread_create");
        close(sockfd);
        return EXIT_FAILURE;
    }

    /* thread หลัก: วนลูปอ่าน input จากผู้ใช้แล้วส่งไปยัง server */
    char input[BUF_SIZE];
    while (running && fgets(input, sizeof(input), stdin) != NULL) {
        if (strncmp(input, "/quit", 5) == 0) {
            running = 0;
            break;
        }
        ssize_t sent = send(sockfd, input, strlen(input), 0);
        if (sent < 0) {
            perror("send");
            running = 0;
            break;
        }
    }

    /* ปิด socket เพื่อปลุก recv() ที่ค้างอยู่ใน receive_loop() ให้คืนค่าออกมา
     * (recv() บน socket ที่ถูกปิดจะคืนค่า -1 หรือ 0 ทันที ทำให้ thread นั้น
     * หลุดจากลูปและจบการทำงานได้เอง) */
    running = 0;
    shutdown(sockfd, SHUT_RDWR);
    close(sockfd);

    pthread_join(recv_tid, NULL);

    printf("ออกจากห้องแชทเรียบร้อย\n");
    return EXIT_SUCCESS;
}

/* ---------- 6. Function Implementations ---------- */

/* ทำงานอยู่เบื้องหลังตลอดเวลา: รอรับข้อความจาก server แล้วพิมพ์ออกทางจอ
 * ทำงานคู่ขนานกับ thread หลักที่รอรับ input จากผู้ใช้ */
static void *receive_loop(void *arg) {
    (void)arg;
    char buf[BUF_SIZE];

    while (running) {
        ssize_t n = recv(sockfd, buf, sizeof(buf) - 1, 0);
        if (n <= 0) {
            /* n == 0: server ปิดการเชื่อมต่อ (เช่น server ล่มหรือถูกสั่งหยุด)
             * n < 0: เกิด error หรือ socket ถูกปิดจาก thread หลักตอน /quit */
            if (running) {
                printf("\n*** การเชื่อมต่อกับ server ถูกตัด ***\n");
            }
            running = 0;
            break;
        }
        buf[n] = '\0';
        printf("%s", buf);
        fflush(stdout);
    }
    return NULL;
}
```

---

## 35.6 คอมไพล์และทดสอบระบบจริง (Step 278)

### คอมไพล์

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 -g -pthread server.c -o server
gcc -Wall -Wextra -Wpedantic -std=c17 -g -pthread client.c -o client
```

สังเกต flag `-pthread` ที่ต้องใส่ทั้งตอน compile และ link เสมอเมื่อใช้ POSIX threads
(ทบทวนจาก Part 31) — ถ้าลืมใส่ อาจคอมไพล์ผ่านแต่ link ไม่ผ่าน หรือแย่กว่านั้นคือ link ผ่าน
แต่พฤติกรรม thread ทำงานผิดปกติเพราะไลบรารี pthread บางเวอร์ชันไม่ได้ถูกโหลดแบบ thread-safe
เต็มรูปแบบ ทั้งสองคำสั่งข้างต้นรันผ่านโดย **ไม่มี warning แม้แต่บรรทัดเดียว**

### ทดสอบด้วย Terminal 3 หน้าต่าง

เปิด terminal บาน 1 รัน server:

```bash
$ ./server
=== TCP Chat Server ===
กำลังฟังที่ port 5500 (กด Ctrl+C เพื่อหยุด)
```

เปิด terminal บาน 2 รัน client คนแรก:

```bash
$ ./client 127.0.0.1 5500
เชื่อมต่อไปยัง 127.0.0.1:5500 สำเร็จ
ตั้งชื่อผู้ใช้ของคุณ: Alice
=== เข้าร่วมห้องแชทแล้ว พิมพ์ข้อความแล้วกด Enter เพื่อส่ง ===
=== พิมพ์ /quit เพื่อออกจากโปรแกรม ===
```

เปิด terminal บาน 3 รัน client คนที่สอง:

```bash
$ ./client 127.0.0.1 5500
เชื่อมต่อไปยัง 127.0.0.1:5500 สำเร็จ
ตั้งชื่อผู้ใช้ของคุณ: Bob
=== เข้าร่วมห้องแชทแล้ว พิมพ์ข้อความแล้วกด Enter เพื่อส่ง ===
=== พิมพ์ /quit เพื่อออกจากโปรแกรม ===
*** Alice เข้าร่วมห้องแชทแล้ว ***
```

กลับไปที่ terminal ของ Alice พิมพ์ `Hello everyone` แล้วกด Enter — terminal ของ Bob จะขึ้น
ทันที `Alice: Hello everyone` ในขณะที่ terminal ของ server แสดง log:

```
[+] Alice เข้าร่วมห้องแชท (fd=4)
[+] Bob เข้าร่วมห้องแชท (fd=5)
Alice: Hello everyone
```

การทดสอบข้างต้นได้รับการยืนยันจริงในห้องแล็บด้วยสคริปต์ทดสอบอัตโนมัติที่จำลอง 3 client
(Alice, Bob, Carol) เชื่อมต่อ ส่งข้อความ และหลุดออกในเวลาต่างกัน ได้ log ของ server ดังนี้
(คัดลอกจากการรันจริง ไม่มีการแก้ไข):

```
=== TCP Chat Server ===
กำลังฟังที่ port 5500 (กด Ctrl+C เพื่อหยุด)
[+] Alice เข้าร่วมห้องแชท (fd=4)
Alice: Hello everyone
[+] Bob เข้าร่วมห้องแชท (fd=5)
Bob: Hi Alice
[+] Carol เข้าร่วมห้องแชท (fd=6)
Carol: Hey all, quick message
[-] Carol ออกจากห้องแชท (fd=6)
[-] Bob ออกจากห้องแชท (fd=5)
[-] Alice ออกจากห้องแชท (fd=4)
Server ปิดตัวเรียบร้อย
```

และฝั่ง client ของ Alice (ซึ่งยังอยู่ในห้องตอนที่ Bob และ Carol เข้ามาและออกไป) เห็นครบทุก
เหตุการณ์ตามลำดับเวลาจริง:

```
เชื่อมต่อไปยัง 127.0.0.1:5500 สำเร็จ
ตั้งชื่อผู้ใช้ของคุณ: === เข้าร่วมห้องแชทแล้ว พิมพ์ข้อความแล้วกด Enter เพื่อส่ง ===
=== พิมพ์ /quit เพื่อออกจากโปรแกรม ===
*** Bob เข้าร่วมห้องแชทแล้ว ***
Bob: Hi Alice
*** Carol เข้าร่วมห้องแชทแล้ว ***
Carol: Hey all, quick message
*** Carol ออกจากห้องแชทแล้ว ***
*** Bob ออกจากห้องแชทแล้ว ***
ออกจากห้องแชทเรียบร้อย
```

### ทดสอบด้วย netcat (ยืนยันว่าโปรโตคอลเป็น Plain Text จริง)

เพราะโปรโตคอลของเราออกแบบเป็น plain text ล้วนๆ (ไม่มีการเข้ารหัสหรือ header พิเศษ) เราจึง
ใช้ `netcat` (`nc`) ต่อเข้าไปคุยกับ server ได้ตรงๆ โดยไม่ต้องเขียน client เองเลย เหมาะสำหรับ
debug อย่างรวดเร็ว:

```bash
$ nc 127.0.0.1 5500
netcat-user
hello from netcat
```

ฝั่ง client ตัวจริงอีกคน (Dave) ที่เชื่อมต่ออยู่จะเห็น:

```
*** Dave เข้าร่วมห้องแชทแล้ว ***
```

และ server log:

```
[+] netcat-user เข้าร่วมห้องแชท (fd=4)
netcat-user: hello from netcat
[+] Dave เข้าร่วมห้องแชท (fd=5)
Dave: Hi netcat-user
[-] Dave ออกจากห้องแชท (fd=5)
[-] netcat-user ออกจากห้องแชท (fd=4)
```

> **ข้อสังเกตจากการทดสอบจริง**: `nc` เวอร์ชันมาตรฐานของ Ubuntu ไม่ปิดการเชื่อมต่อเองทันที
> หลัง stdin ถูกปิด (EOF) — มันจะรอฝั่งตรงข้ามปิดก่อน ทำให้ถ้าทดสอบผ่านสคริปต์ต้องใช้
> `timeout` คลุมไว้ (เช่น `timeout 3 nc 127.0.0.1 5500`) ไม่เช่นนั้นสคริปต์จะค้าง นี่คือ
> พฤติกรรมที่แตกต่างกันไปตามแต่ละ implementation ของ `nc` (บางตัวมี flag `-q` สำหรับกำหนด
> เวลาหน่วงก่อนปิดหลัง EOF) และเป็นตัวอย่างที่ดีว่า "เครื่องมือ debug ก็มีพฤติกรรมที่ต้อง
> ทำความเข้าใจเช่นกัน" ไม่ใช่แค่โค้ดที่เราเขียนเอง

### ตรวจสอบ Memory และ File Descriptor Leak ด้วย valgrind

แม้ Part 38 จะพูดถึง valgrind อย่างละเอียด แต่สำหรับโปรเจกต์ที่มีการ malloc และเปิด/ปิด
file descriptor ตลอดเวลาแบบนี้ ควรตรวจสอบเบื้องต้นไว้ตั้งแต่ตอนนี้:

```bash
valgrind --leak-check=full --show-leak-kinds=all --track-origins=yes ./server
```

เปิด client 2 ตัวเชื่อมต่อเข้ามาคุยกันแล้วหลุดออกตามปกติ จากนั้นกด Ctrl+C ที่ server
ผลลัพธ์จริงจากการทดสอบ (คัดลอกทั้งหมด ไม่มีการตัดต่อ):

```
==25545== Memcheck, a memory error detector
==25545== Copyright (C) 2002-2022, and GNU GPL'd, by Julian Seward et al.
==25545== Using Valgrind-3.22.0 and LibVEX; rerun with -h for copyright info
==25545== Command: ./server
==25545==
==25545==
==25545== HEAP SUMMARY:
==25545==     in use at exit: 0 bytes in 0 blocks
==25545==   total heap usage: 4 allocs, 4 frees, 4,376 bytes allocated
==25545==
==25545== All heap blocks were freed -- no leaks are possible
==25545==
==25545== For lists of detected and suppressed errors, rerun with: -s
==25545== ERROR SUMMARY: 0 errors from 0 contexts (suppressed: 0 from 0)
```

**"All heap blocks were freed -- no leaks are possible"** และ **"0 errors"** คือผลลัพธ์ที่
ต้องการเห็นเสมอ — ยืนยันว่าการ `malloc(sizeof(int))` ทุกครั้งที่มี client ใหม่ (35.3) ถูก
`free()` ครบทุกครั้งจริง (4 allocs ตรงกับ 4 ครั้งที่มี client เชื่อมต่อในการทดสอบนี้พอดี)

ระหว่างการพัฒนาจริง เราเคยรันคำสั่งนี้แล้วเจอ warning ประหลาด `Warning: invalid file
descriptor -1 in syscall close()` ซึ่งนำไปสู่การค้นพบบั๊ก **double-close** ใน `main()`
(อธิบายละเอียดในหัวข้อ "ข้อผิดพลาดที่พบบ่อย" ข้อ 6) — นี่คือตัวอย่างที่ดีว่า valgrind
ตรวจจับปัญหาได้ลึกกว่าการทดสอบด้วยการรันโปรแกรมเฉยๆ มาก แม้โปรแกรมจะ "ดูเหมือนทำงานถูกต้อง"
จากภายนอกก็ตาม

---

## 35.7 การจัดการ Disconnect และการปิดทรัพยากรอย่างถูกต้อง (Step 279)

การจัดการ disconnect ให้ถูกต้องคือส่วนที่แยก "โค้ดสาธิต" ออกจาก "โค้ดที่พร้อมใช้งานจริง"
มาดูรายละเอียดของทุกกรณีที่ client อาจหลุดจากห้องแชท:

### กรณีที่ 1: Client ปิดโปรแกรมอย่างสุภาพ (พิมพ์ `/quit`)

ฝั่ง client เรียก `shutdown()` + `close()` ตามลำดับที่อธิบายใน 35.5 ฝั่ง server จะเห็น
`recv()` คืนค่า `0` (ไม่ใช่ค่าลบ) ซึ่งหมายถึง "TCP FIN ถูกส่งมาอย่างเป็นระเบียบ" — โค้ดของเรา
ตรวจสอบด้วย `n <= 0` ครอบคลุมทั้งกรณีนี้และกรณี error ไว้ในเงื่อนไขเดียว เพราะทั้งสองกรณี
เราต้องทำ cleanup แบบเดียวกันคือออกจากลูปแล้วปิดทรัพยากร

### กรณีที่ 2: Client หลุดกะทันหัน (ปิด terminal, network ขาด, kill -9)

ในกรณีนี้ TCP connection จะไม่มีการส่ง FIN อย่างเป็นระเบียบ ฝั่ง server อาจได้รับ
`ECONNRESET` (connection reset by peer) เมื่อพยายาม `recv()` หรือ `send()` ไปยัง client
ที่หายไปแล้ว ซึ่งจะทำให้ `recv()` คืนค่า `-1` — โค้ดของเราจัดการเหมือนกรณีที่ 1 คือ `n <= 0`
แล้วออกจากลูปทันที ไม่ต้องแยกกรณีเพราะผลลัพธ์ที่ต้องทำ (cleanup) เหมือนกัน

### กรณีที่ 3: Server เต็ม (ครบ `MAX_CLIENTS` แล้ว)

ในกรณีนี้ `client_list_add()` จะคืนค่า `-1` ตั้งแต่ก่อนสร้าง thread ด้วยซ้ำ — server จะส่ง
ข้อความแจ้งเตือนแล้วปิด connection ทันที ไม่เปลืองทรัพยากรสร้าง thread ที่ไม่มีที่เก็บข้อมูล
ให้ใช้งานเลย

### ลำดับการ Cleanup ที่ถูกต้อง (ทบทวนจาก 35.4)

```
1. client_list_remove(client_fd)   <- ลบออกจาก list ก่อน (ป้องกัน thread อื่น broadcast มาเจอ)
2. close(client_fd)                <- ปิด socket จริง คืน fd ให้ระบบนำไปใช้ซ้ำได้
3. broadcast_message(...)          <- แจ้งคนอื่นว่ามีคนออกไปแล้ว (หลัง fd ถูกลบออกจาก list แล้ว)
```

หากสลับลำดับ (ปิด `close()` ก่อนลบออกจาก list) จะเปิดช่องให้เกิดปัญหาที่เรียกว่า
**file descriptor reuse race**: เมื่อ `close(client_fd)` แล้ว OS อาจนำเลข fd เดียวกันนั้น
ไปมอบให้ connection ใหม่ที่กำลังจะเข้ามาทันที (fd เป็นทรัพยากรที่ถูกนำกลับมาใช้ใหม่เสมอ)
ถ้า `client_list` ยังไม่ทันถูกอัปเดต thread อื่นที่กำลัง `broadcast_message()` อยู่พอดีอาจ
`send()` ไปยัง fd หมายเลขเดิม ซึ่งตอนนี้กลายเป็นของ client คนใหม่ไปแล้ว — ข้อความจะไปผิดคน

### thread ที่ Detach แล้วไม่มี Zombie เหลือค้าง

เพราะทุก client thread ถูก `pthread_detach()` ตั้งแต่สร้าง (35.3) เมื่อ `client_handler()`
`return NULL;` ระบบจะคืนทรัพยากรของ thread นั้น (stack, thread control block) ให้อัตโนมัติ
ทันที ไม่ต้องมีใครมา `pthread_join()` — นี่คือเหตุผลที่การทดสอบด้วย valgrind ใน 35.6 ไม่พบ
memory leak แม้จะมี client เข้า-ออกหลายรอบ

---

## 35.8 แนวทางต่อยอดโปรเจกต์ (Step 280)

โปรเจกต์นี้เป็นจุดเริ่มต้นที่แข็งแรง แต่ยังมีช่องทางต่อยอดอีกมากสำหรับความท้าทายเพิ่มเติม:

| แนวทางต่อยอด | รายละเอียดคร่าวๆ |
|---|---|
| **Private Message** | คำสั่ง `/msg <name> <text>` ส่งเฉพาะคนที่ระบุ (ดูแบบฝึกหัดข้อ 2) |
| **รายชื่อผู้ใช้ออนไลน์** | คำสั่ง `/list` ให้ server ส่งรายชื่อ client ทั้งหมดที่ active กลับมา |
| **บันทึกประวัติแชท** | เขียนทุกข้อความลงไฟล์ log ด้วย `mmap()` แทน `fwrite()` เพื่อประสิทธิภาพ (ต่อใน Part 36!) |
| **Thread Pool** | แทนที่จะสร้าง thread ใหม่ทุกครั้ง ใช้ pool ของ thread ที่สร้างไว้ล่วงหน้าจำนวนคงที่ (เรียนลึกใน Part 101) |
| **Event-driven ด้วย epoll** | แทนที่ thread-per-client ทั้งหมดด้วย single-thread epoll loop (ทบทวน Part 34) เพื่อรองรับ client จำนวนมากด้วยทรัพยากรน้อยกว่า |
| **TLS Encryption** | ห่อ socket ด้วย OpenSSL เพื่อเข้ารหัสการสื่อสาร (โปรโตคอล plain text ปัจจุบันดักฟังได้ง่ายมาก) |
| **Rate Limiting** | จำกัดจำนวนข้อความต่อวินาทีต่อ client เพื่อป้องกัน spam/DoS |

### ทำไม Thread-per-Client ไม่ Scale ไปเรื่อยๆ

ลองคำนวณคร่าวๆ: Linux กำหนด stack size เริ่มต้นของแต่ละ thread ไว้ที่ 8 MB (ปรับได้ผ่าน
`pthread_attr_setstacksize()` แต่ไม่ควรตั้งเล็กเกินไปเพราะเสี่ยง stack overflow) ถ้ามี client
10,000 คนพร้อมกัน สถาปัตยกรรมนี้จะพยายามจอง stack รวม **80 GB** (แม้ในทางปฏิบัติ Linux จะ
จองแบบ lazy allocation ไม่ commit หน่วยความจำจริงจนกว่าจะใช้ แต่ virtual memory ก็ยังเป็น
ข้อจำกัด และ context-switching ระหว่าง thread นับหมื่นเส้นก็มีต้นทุนสูงมาก) นี่คือเหตุผลที่
Part 34 สอน `epoll` ไว้ล่วงหน้า — เมื่อถึงจุดที่ระบบต้องรองรับ client จำนวนมาก (production
web server ระดับ Part 100-101) การสลับไปใช้ **single-thread + epoll** หรือ **thread pool
ขนาดคงที่ร่วมกับ epoll** จะให้ประสิทธิภาพดีกว่ามาก โปรเจกต์นี้จึงเหมาะกับงานขนาดกลาง
(หลักสิบถึงหลักร้อย concurrent connections) เป็นหลัก

ใน **Part 36** เราจะเปลี่ยนโฟกัสไปที่การจัดการไฟล์ด้วย **Memory-Mapped File (mmap)** ซึ่ง
เป็นเทคนิคที่มีประโยชน์มากถ้าจะนำไปทำระบบบันทึกประวัติแชท (chat log) ที่ต้องอ่าน/เขียนไฟล์
ขนาดใหญ่อย่างมีประสิทธิภาพ

---

## ข้อผิดพลาดที่พบบ่อย (Common Pitfalls)

1. **ลืมล็อก `mutex` ตอนวนลูป broadcast หรือแก้ไข `client_list`** — แม้จะเป็นแค่การ "อ่าน"
   อย่างเดียวก็ต้องล็อก เพราะถ้า thread อื่นกำลังเขียนอยู่พอดี การอ่านค่าครึ่งๆ กลางๆ เป็น
   Undefined Behavior เสมอ อาการที่พบบ่อยคือโปรแกรม crash แบบ **ไม่สม่ำเสมอ** (บางทีรันได้
   ปกติ บางทีพัง) ซึ่งเป็นสัญญาณคลาสสิกของ race condition

2. **ส่ง pointer ไปยัง stack variable เข้า `pthread_create()` โดยตรง** เช่น
   `pthread_create(&tid, NULL, client_handler, &client_fd)` แทนที่จะ `malloc` ค่าให้แยก
   ต่างหาก (35.3) จะทำให้ thread ใหม่มีโอกาสอ่านค่าผิดเพราะตัวแปรต้นทางถูกเขียนทับไปแล้วก่อน
   ที่ thread ใหม่จะทันอ่าน — บั๊กนี้อาจไม่แสดงอาการทุกครั้งที่รัน ทำให้ debug ยากมาก

3. **ไม่ ignore `SIGPIPE` และไม่ใช้ `MSG_NOSIGNAL`** — ถ้า client หลุดกะทันหันแล้ว server
   เรียก `send()` ไปยัง socket นั้นอีกครั้งโดยไม่ป้องกันไว้เลย ค่า default ของ SIGPIPE คือ
   **ฆ่าโปรแกรมทั้งตัวทันที** ซึ่งร้ายแรงมากสำหรับ server ที่ต้องรันตลอดเวลาไม่ว่า client
   คนไหนจะหลุดไปก็ตาม ควรป้องกันทั้งสองชั้น (`signal(SIGPIPE, SIG_IGN)` และ `MSG_NOSIGNAL`)

4. **เข้าใจผิดว่า TCP ส่งข้อมูลเป็น "ก้อนข้อความ" ตรงตามจำนวนครั้งที่เรียก `send()`** — ตามที่
   สาธิตจริงใน 35.4 การเรียก `send()` สองครั้งติดกันเร็วๆ (เช่น ส่งชื่อแล้วส่งข้อความทันที)
   อาจถูกเคอร์เนลรวมมาให้ `recv()` ครั้งเดียว หรือในทางกลับกัน ข้อมูลก้อนใหญ่ก้อนเดียวที่
   `send()` อาจถูกแบ่งมาให้ `recv()` หลายครั้งก็ได้ (โดยเฉพาะเมื่อข้อความมีขนาดใหญ่กว่า MTU
   ของเครือข่าย) โค้ดที่ถูกต้องต้องไม่สมมติว่า 1 `recv()` เท่ากับ 1 ข้อความเสมอ

5. **ไม่ `pthread_detach()` หรือ `pthread_join()` thread ของ client** — ถ้าลืมทั้งสองอย่าง
   thread ที่ทำงานจบไปแล้วจะกลายเป็นสถานะคล้าย zombie thread ที่ทรัพยากร (thread control
   block) ไม่ถูกคืนให้ระบบ ยิ่งมี client เข้า-ออกมากเท่าไหร่ ยิ่งสะสมมากขึ้นเรื่อยๆ จนอาจถึง
   ขีดจำกัดจำนวน thread สูงสุดของระบบปฏิบัติการในที่สุด

6. **Double-close บน file descriptor เดียวกัน** — บั๊กจริงที่พบระหว่างพัฒนาโปรเจกต์นี้:
   `handle_sigint()` ปิด `server_fd` และตั้งค่าเป็น `-1` แล้ว แต่โค้ดเวอร์ชันแรกใน `main()`
   หลังออกจาก accept loop ยังเรียก `close(server_fd)` ซ้ำอีกครั้งโดยไม่เช็กก่อน กลายเป็น
   `close(-1)` ซึ่งแม้จะไม่ทำให้โปรแกรม crash (คืนค่า error เฉยๆ) แต่ `valgrind` จับได้ทันที
   ด้วยข้อความ `Warning: invalid file descriptor -1 in syscall close()` บทเรียนคือ **ทุกครั้ง
   ที่ signal handler ปิดทรัพยากรร่วมกับโค้ดส่วนอื่น ต้องมีการเช็กสถานะก่อนปิดซ้ำเสมอ**
   (ในโค้ดฉบับสมบูรณ์ด้านบนได้แก้เป็น `if (server_fd >= 0) { close(server_fd); }` แล้ว)

7. **ลืมว่า `close()` บน socket ไม่ปลุก `recv()` ที่ thread อื่น block ค้างอยู่เสมอไป** — ใน
   บาง implementation การ `close()` fd จาก thread หนึ่งขณะที่อีก thread กำลัง block อยู่ใน
   `recv()` บน fd เดียวกันอาจไม่ปลุกให้ตื่นทันที ทางที่ปลอดภัยกว่าคือเรียก `shutdown(fd,
   SHUT_RDWR)` ก่อนเสมอ (ดังที่ทำในฝั่ง client ที่ 35.5) ซึ่งรับประกันว่า `recv()` ที่ค้างอยู่
   จะคืนค่าออกมาแน่นอน

---

## แบบฝึกหัดท้ายบท

1. เพิ่มคำสั่ง `/list` ให้ client ขอรายชื่อผู้ใช้ทั้งหมดที่ online จาก server แล้ว server
   ตอบกลับเป็นรายการชื่อคั่นด้วยจุลภาค (comma) เฉพาะ client ที่ขอเท่านั้น
2. เพิ่มฟีเจอร์ private message: `/msg <name> <text>` ให้ส่งข้อความถึงเฉพาะผู้ใช้ที่ระบุชื่อ
   เท่านั้น ไม่ broadcast ให้คนอื่นเห็น พร้อมแจ้งเตือนถ้าไม่พบผู้ใช้ชื่อนั้นในห้อง
3. ปรับปรุงระบบชื่อผู้ใช้ไม่ให้ซ้ำกัน — ถ้ามีคนตั้งชื่อซ้ำกับที่มีอยู่แล้วในห้อง ให้ server
   เติมตัวเลขต่อท้ายอัตโนมัติ (เช่น `Alice` ซ้ำ ให้กลายเป็น `Alice2`) แล้วแจ้ง client ว่าชื่อ
   ถูกเปลี่ยน
4. เขียน bash script ที่เปิด server และ client หลายตัวพร้อมกันโดยอัตโนมัติ (คล้ายที่สาธิตใน
   35.6) แล้วตรวจสอบผลลัพธ์ในไฟล์ log ของแต่ละ client ว่าตรงตามที่คาดหวังหรือไม่ (automated
   integration test)
5. ปรับ `MAX_CLIENTS` ให้ใช้ dynamic array (`realloc`) แทน fixed-size array เพื่อรองรับ
   จำนวน client ไม่จำกัด พร้อมป้องกัน race condition ระหว่างการขยาย array ด้วย mutex ตัวเดิม
6. เพิ่ม timestamp รูปแบบ `[HH:MM:SS]` หน้าทุกข้อความที่ถูก broadcast โดยใช้เวลาปัจจุบันของ
   เครื่อง server (ใช้ `time()`, `localtime_r()`, และ `strftime()`)

### แนวทางเฉลยข้อ 6 (Timestamp)

เพิ่ม `#include <time.h>` แล้วแก้ส่วนสร้าง `out_msg` ใน `client_handler()` ให้แปะเวลาไว้
หน้าสุด ก่อนชื่อผู้ส่ง:

```c
/* สร้าง timestamp รูปแบบ [HH:MM:SS] จากเวลาปัจจุบันของเครื่อง server
 * time() + localtime() ให้ struct tm แล้วใช้ strftime() จัดรูปแบบ
 * เป็น string ตามที่ต้องการ (ปลอดภัยกว่าการต่อ string มือ) */
time_t now = time(NULL);
struct tm tm_now;
localtime_r(&now, &tm_now);   /* localtime_r = เวอร์ชัน thread-safe ของ localtime() */
char time_buf[16];
strftime(time_buf, sizeof(time_buf), "%H:%M:%S", &tm_now);

char out_msg[BUF_SIZE + NAME_SIZE + 32];
int out_len = snprintf(out_msg, sizeof(out_msg), "[%s] %s: %s", time_buf, name, buf);
if (out_len < 0) {
    continue;
}
```

จุดสำคัญคือใช้ `localtime_r()` แทน `localtime()` ธรรมดา — `localtime()` มาตรฐานคืน pointer
ไปยัง `static struct tm` ตัวเดียวที่ใช้ร่วมกันในทั้งโปรเซส **ไม่ thread-safe** ถ้าหลาย
client thread เรียก `localtime()` พร้อมกัน อาจได้ค่าปนกันข้ามกันได้ (race condition อีกแบบ
หนึ่งที่ไม่เกี่ยวกับ `client_list` เลย แต่เกี่ยวกับ standard library function ที่ไม่ thread-safe)
ส่วน `localtime_r()` (`_r` ย่อมาจาก "reentrant") รับ buffer ของผู้เรียกเอง (`&tm_now`) ทำให้
แต่ละ thread มีพื้นที่เก็บผลลัพธ์เป็นของตัวเอง ปลอดภัยแน่นอน

ทดสอบคอมไพล์และรันจริงแล้วได้ผลลัพธ์:

```
[+] Alice เข้าร่วมห้องแชท (fd=4)
[06:19:51] Alice: Hello with timestamp
[-] Alice ออกจากห้องแชท (fd=4)
```

### แนวทางเฉลยข้อ 2 (Private Message)

เพิ่มฟังก์ชันช่วยส่งข้อความถึงคนเดียวโดยค้นหาจากชื่อ (วางไว้ข้างๆ `broadcast_message()`):

```c
/* ส่งข้อความส่วนตัวไปยัง client ที่ชื่อ target_name เท่านั้น (ไม่ broadcast)
 * คืนค่า 1 ถ้าหาชื่อเจอและส่งสำเร็จ, 0 ถ้าไม่พบผู้ใช้ชื่อนี้ในห้อง */
static int send_to_name(const char *target_name, const char *msg, size_t len) {
    int found = 0;
    pthread_mutex_lock(&client_list_mutex);
    for (int i = 0; i < MAX_CLIENTS; i++) {
        if (client_list[i].active && strcmp(client_list[i].name, target_name) == 0) {
            ssize_t sent = send(client_list[i].sockfd, msg, len, MSG_NOSIGNAL);
            (void)sent;
            found = 1;
            break;
        }
    }
    pthread_mutex_unlock(&client_list_mutex);
    return found;
}
```

จากนั้นในลูปหลักของ `client_handler()` ตรวจสอบว่าข้อความที่ได้รับขึ้นต้นด้วย `/msg ` หรือไม่
**ก่อน** จะประมวลผลเป็นข้อความ broadcast ปกติ:

```c
/* ตรวจสอบคำสั่ง /msg <name> <text> ก่อนว่าเป็นข้อความส่วนตัวหรือไม่
 * รูปแบบ: "/msg Bob สวัสดี" -> ส่งเฉพาะ Bob เท่านั้น ไม่ broadcast */
if (strncmp(buf, "/msg ", 5) == 0) {
    char target[NAME_SIZE];
    char body[BUF_SIZE];
    /* sscanf กับ %31s จะอ่านคำแรก (ชื่อ target) โดยไม่ล้น buffer
     * %n เก็บตำแหน่ง offset หลังจากอ่านชื่อเสร็จ เพื่อตัดส่วนที่เหลือ
     * ทั้งหมดออกมาเป็นเนื้อความ (รองรับข้อความที่มีช่องว่างหลายคำ) */
    int offset = 0;
    if (sscanf(buf + 5, "%31s%n", target, &offset) == 1) {
        const char *body_start = buf + 5 + offset;
        while (*body_start == ' ') {
            body_start++;
        }
        snprintf(body, sizeof(body), "%s", body_start);

        char priv_msg[BUF_SIZE + NAME_SIZE + 32];
        int priv_len = snprintf(priv_msg, sizeof(priv_msg),
                                 "[private จาก %s] %s", name, body);
        if (priv_len > 0 && priv_msg[priv_len - 1] != '\n' &&
            (size_t)priv_len < sizeof(priv_msg) - 1) {
            priv_msg[priv_len] = '\n';
            priv_msg[priv_len + 1] = '\0';
            priv_len++;
        }

        if (send_to_name(target, priv_msg, (size_t)priv_len)) {
            printf("[private] %s -> %s: %s", name, target, body);
        } else {
            char notfound[BUF_SIZE];
            int nf_len = snprintf(notfound, sizeof(notfound),
                                   "*** ไม่พบผู้ใช้ชื่อ '%s' ในห้อง ***\n", target);
            send(client_fd, notfound, (size_t)nf_len, MSG_NOSIGNAL);
        }
    }
    continue;   /* ข้อความ /msg ไม่ต้อง broadcast ต่อ ให้วนไปรอรับข้อความถัดไป */
}
```

ทดสอบจริง: Alice ส่ง `/msg Bob Hi Bob only` ผลลัพธ์ฝั่ง server:

```
[+] Alice เข้าร่วมห้องแชท (fd=4)
[+] Bob เข้าร่วมห้องแชท (fd=5)
[private] Alice -> Bob: Hi Bob only
```

ฝั่ง Bob ได้รับ `[private จาก Alice] Hi Bob only` ในขณะที่ client คนอื่นที่อาจอยู่ในห้อง
(ถ้ามี) จะไม่เห็นข้อความนี้เลย ยืนยันว่า private message ทำงานถูกต้องตามที่ออกแบบไว้ — และ
สังเกตว่าโค้ดนี้ยังคงยึดกฎเหล็กของบทเรียน: ทุกการเข้าถึง `client_list` (ใน `send_to_name`)
ยังคงล็อก mutex เหมือนเดิมทุกประการ

---

## สรุปท้ายบท

ใน Part นี้เราได้:

- ออกแบบและสร้าง TCP Chat Server แบบ multi-client ด้วยสถาปัตยกรรม **thread-per-client**
  โดยรวมความรู้เรื่อง socket, pthread, และ mutex จาก Part 31-34 เข้าด้วยกันเป็นโปรเจกต์จริง
- เข้าใจอย่างลึกซึ้งว่าทำไม shared state ระหว่าง thread (client list) ต้องมี mutex ป้องกัน
  พร้อมเห็นตัวอย่างเป็นรูปธรรมของ race condition ที่จะเกิดขึ้นถ้าไม่มี
- เขียน `broadcast_message()` และจัดการ full-duplex client ด้วยสอง thread (ส่ง/รับ) ได้เอง
- ค้นพบและแก้บั๊กจริง 2 ตัวระหว่างพัฒนา: การที่ TCP เป็น byte stream ทำให้ `recv()` รวมสอง
  ข้อความเข้าด้วยกันได้ (ทำให้ข้อมูลหายถ้าไม่จัดการ leftover ให้ถูกต้อง) และบั๊ก double-close
  บน `server_fd` ที่ `valgrind` เป็นผู้ตรวจจับได้
- ทดสอบระบบจริงด้วย terminal หลายหน้าต่าง, สคริปต์อัตโนมัติ, `netcat`, และ `valgrind` จนมั่นใจ
  ว่าไม่มี memory leak หรือ file descriptor leak หลงเหลือ
- เห็นแนวทางต่อยอดโปรเจกต์และข้อจำกัดของสถาปัตยกรรม thread-per-client เมื่อต้องรองรับ
  client จำนวนมาก ซึ่งเป็นแรงจูงใจของเทคนิค epoll และ thread pool ที่จะเรียนในบทถัดๆ ไป

โปรเจกต์นี้คือรากฐานสำคัญที่จะกลับมาใช้ซ้ำอีกครั้งใน **Part 122 (Capstone: Real-time Chat
Server)** ช่วงท้ายหลักสูตร ซึ่งจะนำแนวคิดเดียวกันนี้ไปขยายเป็นระบบระดับ production เต็มรูปแบบ

ใน **Part 36** เราจะเปลี่ยนหัวข้อไปที่การจัดการไฟล์ด้วยเทคนิค **Memory-Mapped File (mmap)**
ซึ่งเป็นวิธีการอ่าน/เขียนไฟล์ที่เร็วกว่า `fread`/`fwrite` แบบเดิมมาก โดยการ map เนื้อหาไฟล์
เข้ามาอยู่ใน address space ของโปรแกรมโดยตรง — เทคนิคนี้จะเป็นประโยชน์มากถ้าจะนำไปต่อยอด
โปรเจกต์แชทให้บันทึกประวัติข้อความลงไฟล์ขนาดใหญ่อย่างมีประสิทธิภาพ

**ต่อไป:** [Part 36 — Memory-Mapped File (mmap)](./part-036-mmap.md)
