# Part 29: Inter-Process Communication (Pipe/FIFO) (Step 225–232)

> Module C — Systems Programming ด้วย C บน Linux | Part 29 จาก 125
> Step ที่ครอบคลุมใน Part นี้: Step 225–232
> Part ก่อนหน้า: [Part 28 — Signal และ Signal Handling](./part-028-signals.md) | Part ถัดไป: [Part 30 — Shared Memory และ Message Queue](./part-030-shared-memory-mq.md)

## เป้าหมายการเรียนรู้ (Learning Objectives)

หลังจบ Part นี้ ผู้เรียนจะสามารถ:

1. อธิบายได้ว่าทำไม process ต้องมีกลไกสื่อสารกัน (IPC) และทำไมจะใช้ตัวแปร global ธรรมดา
   ร่วมกันระหว่าง process ไม่ได้เหมือนตอนใช้ thread
2. ใช้ syscall `pipe()` สร้าง anonymous pipe และเข้าใจว่ามันคือ file descriptor คู่หนึ่ง
   (read-end กับ write-end) ที่เชื่อมกันในทิศทางเดียว
3. เขียนโปรแกรมจริงที่ parent process เขียนข้อมูลผ่าน pipe ให้ child process ที่ fork มาอ่าน
4. อธิบายและป้องกันข้อผิดพลาดคลาสสิกที่สุดของ pipe ได้ นั่นคือการลืมปิด file descriptor
   ฝั่งที่ไม่ได้ใช้ ซึ่งเป็นสาเหตุอันดับหนึ่งของโปรแกรมที่ค้าง (hang) เพราะไม่มีวันเจอ EOF
5. สร้างการสื่อสารสองทาง (bidirectional) ระหว่าง parent-child ด้วย pipe สองเส้น
6. ใช้ `mkfifo()` สร้าง Named Pipe (FIFO) เพื่อให้ process ที่**ไม่ใช่**ญาติกันทาง fork
   สามารถสื่อสารกันได้ผ่านระบบไฟล์
7. เขียนโปรแกรมสองตัวที่แยกกันคอมไพล์และรันคนละ process คุยกันผ่าน FIFO ได้จริง
8. เข้าใจพฤติกรรม blocking ของ `open()` บน FIFO และออกแบบ "เซิร์ฟเวอร์" ที่รองรับ
   ผู้เขียนหลายรายทยอยเชื่อมต่อเข้ามาได้

---

## 29.1 ทำไม Process ต้องสื่อสารกัน (Step 225)

ใน Part 27 เราเรียนเรื่อง `fork()` ไปแล้วว่ามันสร้าง process ใหม่โดยการ **copy** memory
ทั้งหมดของ process เดิม (แบบ copy-on-write) นั่นหมายความว่า **หลังจาก `fork()` เสร็จ
parent กับ child จะมี address space แยกจากกันอย่างสมบูรณ์** ตัวแปรที่ชื่อเดียวกันในทั้งสอง
process จะไม่ใช่ตัวแปรตัวเดียวกันอีกต่อไป

```
ก่อน fork()                    หลัง fork()
┌─────────────┐                ┌─────────────┐    ┌─────────────┐
│   Process    │                │   Parent     │    │    Child     │
│  int x = 5;  │  ──fork()──▶   │  int x = 5;  │    │  int x = 5;  │
│              │                │  (memory A)  │    │  (memory B)  │
└─────────────┘                └─────────────┘    └─────────────┘
                                 ถ้า parent แก้ x = 99   child ยังเห็น x = 5 เหมือนเดิม
                                 (คนละหน้าหน่วยความจำกันโดยสิ้นเชิง)
```

นี่คือความแตกต่างที่สำคัญที่สุดระหว่าง **process** กับ **thread** (ซึ่งเราจะเรียนใน Part 31):
thread หลายตัวใน process เดียวกัน **แชร์ address space เดียวกัน** จึงอ่าน/เขียนตัวแปร
global ร่วมกันได้ตรงๆ แต่ process คนละตัวทำแบบนั้นไม่ได้เลย

คำถามคือ แล้วถ้าเราอยากให้ parent ส่งข้อมูลไปให้ child (หรือ process สองตัวที่ไม่เกี่ยวข้อง
กันเลยอยากคุยกัน) จะทำอย่างไร? คำตอบคือกลไกที่เรียกว่า **Inter-Process Communication (IPC)**
ซึ่งระบบปฏิบัติการ Linux มีให้เลือกใช้หลายแบบ:

| กลไก IPC | ลักษณะการทำงาน | ใช้ระหว่าง |
|---|---|---|
| **Anonymous Pipe** (`pipe()`) | ท่อทางเดียว ผ่าน kernel buffer | Process ที่เป็นญาติกัน (parent-child) เท่านั้น |
| **Named Pipe / FIFO** (`mkfifo()`) | เหมือน pipe แต่มีชื่อไฟล์ในระบบไฟล์ | Process ใดก็ได้ ไม่ต้องเป็นญาติกัน |
| **Shared Memory** | หน่วยความจำที่มองเห็นร่วมกัน | Process ใดก็ได้ (เร็วที่สุด — Part 30) |
| **Message Queue** | คิวข้อความที่ kernel จัดการให้ | Process ใดก็ได้ (Part 30) |
| **Socket** | ช่องสื่อสารแบบเครือข่าย (ใช้ในเครื่องเดียวกันได้ด้วย) | Process ใดก็ได้ แม้อยู่คนละเครื่อง (Part 33-35) |
| **Signal** | สัญญาณสั้นๆ ไม่มี payload ข้อมูลจริงจัง | Process ใดก็ได้ (Part 28 ที่เรียนไปแล้ว) |

Part นี้จะเจาะลึก 2 กลไกแรก คือ **Pipe** และ **FIFO** ซึ่งเป็นกลไก IPC ที่เก่าแก่ที่สุดและ
เรียบง่ายที่สุดในตระกูล Unix — จริงๆ แล้วเราใช้ pipe อยู่ทุกวันโดยไม่รู้ตัวทุกครั้งที่พิมพ์คำสั่ง
แบบนี้ใน shell:

```bash
ls -la | grep ".c" | wc -l
```

เครื่องหมาย `|` (pipe character) ตรงนี้คือสิ่งเดียวกันกับที่เราจะเขียนโค้ดสร้างเองใน Part นี้
shell จะสร้าง pipe เชื่อม stdout ของ `ls` เข้ากับ stdin ของ `grep` และเชื่อม stdout ของ
`grep` เข้ากับ stdin ของ `wc` ให้อัตโนมัติ

---

## 29.2 Anonymous Pipe: syscall `pipe()` (Step 226)

`pipe()` เป็น syscall ที่สร้าง **buffer ในหน่วยความจำของ kernel** พร้อมกับ file descriptor
สองตัวที่ชี้ไปยัง buffer นั้น — ตัวหนึ่งไว้ **อ่าน** อีกตัวไว้ **เขียน**

```c
#include <unistd.h>

int pipe(int pipefd[2]);
```

- `pipefd[0]` คือ **read-end** — อ่านข้อมูลออกจาก buffer
- `pipefd[1]` คือ **write-end** — เขียนข้อมูลเข้า buffer
- คืนค่า `0` เมื่อสำเร็จ, `-1` เมื่อผิดพลาด (ต้องเช็คด้วย `perror()` เสมอ)

จำง่ายๆ ด้วยหลักที่ว่า **"0 อ่านได้ เหมือน stdin (fd 0), 1 เขียนได้ เหมือน stdout (fd 1)"**

```
                pipe() สร้าง buffer ใน kernel
        ┌───────────────────────────────────┐
        │                                     │
 write  │        Kernel Pipe Buffer           │  read
 ──────▶│        (ปกติขนาด 64 KB บน Linux)     │───────▶
fd[1]   │                                     │  fd[0]
        └───────────────────────────────────┘
```

จุดสำคัญที่ต้องเข้าใจ:

1. **pipe เป็นทิศทางเดียว (unidirectional)** — เขียนได้แค่ทาง `fd[1]` อ่านได้แค่ทาง `fd[0]`
   เท่านั้น ถ้าอยากสื่อสารสองทางต้องสร้าง pipe สองเส้น (จะสาธิตในหัวข้อ 29.5)
2. **pipe เพียงอย่างเดียวใช้ไม่ได้อะไร** — มันมีประโยชน์ก็ต่อเมื่อใช้คู่กับ `fork()` เท่านั้น
   เพราะ `fork()` จะ copy file descriptor table ทั้งหมดของ parent ไปให้ child ด้วย
   หมายความว่า **หลัง fork() ทั้ง parent และ child จะถือ fd ทั้งสองฝั่ง (`fd[0]` และ
   `fd[1]`) ของ pipe เดียวกันอยู่พร้อมกัน**
3. **pipe เป็น byte stream ไม่ใช่ message queue** — ข้อมูลที่เขียนหลายครั้งอาจถูกอ่านออกมา
   รวมกันเป็นก้อนเดียว หรือถูกแบ่งเป็นหลายก้อนก็ได้ ไม่มีการรักษาขอบเขตของแต่ละ `write()`
   ไว้ให้ (ต่างจาก message queue ใน Part 30 ที่รักษาขอบเขตข้อความให้)
4. **มี buffer จำกัด** — ถ้า buffer เต็ม (ปกติ 64 KB บน Linux) `write()` จะบล็อกรอจนกว่า
   จะมีคนมาอ่านออกไปก่อน

```
       ก่อน fork()                      หลัง fork()
┌───────────────────────┐      ┌───────────────────────┐   ┌───────────────────────┐
│        Process          │      │        Parent           │   │         Child           │
│  fd[0] ──▶ read-end     │      │  fd[0] ──▶ read-end     │   │  fd[0] ──▶ read-end     │
│  fd[1] ──▶ write-end    │─fork▶│  fd[1] ──▶ write-end    │   │  fd[1] ──▶ write-end    │
└───────────────────────┘      └───────────────────────┘   └───────────────────────┘
                                       ▲  ทั้งสองฝั่งชี้ไปที่ pipe buffer เดียวกันใน kernel  ▲
```

---

## 29.3 ตัวอย่างจริง: Parent เขียน Child อ่าน (Step 227)

มาดูโปรแกรมจริงที่ parent process ส่งข้อความหลายบรรทัดให้ child process อ่านและพิมพ์ออกมา:

```c
/* ============================================================
 * ชื่อไฟล์:     pipe_basic.c
 * คำอธิบาย:     ตัวอย่าง pipe() พื้นฐาน parent เขียน child อ่าน
 * ============================================================ */

/* ---------- 1. Header Includes ---------- */
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/wait.h>

/* ---------- 2. Macro / Constant Definitions ---------- */
#define BUF_SIZE 256

/* ---------- 4. main() ---------- */
int main(void) {
    int fd[2]; /* fd[0] = read end, fd[1] = write end */

    if (pipe(fd) == -1) {
        perror("pipe");
        exit(EXIT_FAILURE);
    }

    pid_t pid = fork();
    if (pid == -1) {
        perror("fork");
        exit(EXIT_FAILURE);
    }

    if (pid == 0) {
        /* ---------- Child: อ่านจาก pipe ---------- */
        close(fd[1]); /* child ไม่เขียน จึงปิด write-end ทิ้ง */

        char buf[BUF_SIZE];
        ssize_t n;
        while ((n = read(fd[0], buf, sizeof(buf) - 1)) > 0) {
            buf[n] = '\0';
            printf("[child] ได้รับ: %s", buf);
        }

        printf("[child] read() คืนค่า 0 แปลว่าเจอ EOF แล้ว ปิดโปรแกรม\n");
        close(fd[0]);
        exit(EXIT_SUCCESS);
    } else {
        /* ---------- Parent: เขียนลง pipe ---------- */
        close(fd[0]); /* parent ไม่อ่าน จึงปิด read-end ทิ้ง */

        const char *messages[] = {
            "สวัสดี child\n",
            "นี่คือข้อความที่สอง\n",
            "ข้อความสุดท้ายแล้วนะ\n"
        };

        for (int i = 0; i < 3; i++) {
            write(fd[1], messages[i], strlen(messages[i]));
            sleep(1);
        }

        close(fd[1]); /* สำคัญมาก: ปิด write-end เพื่อส่ง EOF ให้ child */
        wait(NULL);
        printf("[parent] child จบการทำงานแล้ว\n");
    }

    return 0;
}
```

คอมไพล์และรัน:

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 pipe_basic.c -o pipe_basic
./pipe_basic
```

ผลลัพธ์:

```
[child] ได้รับ: สวัสดี child
[child] ได้รับ: นี่คือข้อความที่สอง
[child] ได้รับ: ข้อความสุดท้ายแล้วนะ
[child] read() คืนค่า 0 แปลว่าเจอ EOF แล้ว ปิดโปรแกรม
[parent] child จบการทำงานแล้ว
```

### อธิบายทีละส่วน

- **`pipe(fd)`** เรียกก่อน `fork()` เสมอ — ถ้าเรียกหลัง fork() ทั้งสอง process จะได้ pipe
  คนละเส้นกัน ซึ่งไม่มีประโยชน์อะไรเลย เพราะสื่อสารกันไม่ได้
- **child เข้าเงื่อนไข `pid == 0`** จะปิด `fd[1]` (write-end) ทิ้งทันที เพราะ child มีหน้าที่
  อ่านอย่างเดียว การถือ fd ที่ไม่ได้ใช้ไว้เป็นความเสี่ยง (จะอธิบายในหัวข้อถัดไป)
- **`read()` แบบวนลูป** เป็น pattern มาตรฐานสำหรับอ่านจาก pipe เพราะข้อมูลอาจไม่มาครบ
  ในครั้งเดียว ต้องอ่านไปเรื่อยๆ จนกว่า `read()` จะคืนค่า `0` (แปลว่าเจอ EOF — ไม่มีใคร
  ถือ write-end ไว้แล้ว)
- **parent เรียก `wait(NULL)`** เพื่อรอให้ child จบการทำงานก่อน ป้องกันไม่ให้ child กลาย
  เป็น zombie process (ทบทวนได้จาก Part 27)

---

## 29.4 การปิด File Descriptor ที่ไม่ใช้ให้ถูกต้อง (Step 228)

นี่คือกฎที่สำคัญที่สุดของการใช้งาน pipe: **ทุก process ที่ไม่ได้ใช้ฝั่งใดของ pipe ต้อง
`close()` ฝั่งนั้นทิ้งทันที** ไม่งั้นจะเกิดปัญหาที่ตรวจจับยากมาก

### ทำไมการปิดถึงสำคัญขนาดนั้น

`read()` จากท่อ pipe จะคืนค่า `0` (สัญญาณ EOF) **ก็ต่อเมื่อไม่มี process ใดในระบบถือ
write-end ของ pipe เส้นนั้นค้างอยู่เลย** ปัญหาคือ หลัง `fork()` ทั้ง parent และ child ต่างก็
ถือสำเนาของ `fd[1]` (write-end) อยู่คนละชุด แม้ parent จะ `close(fd[1])` ไปแล้ว
**ถ้า child ยังถือ `fd[1]` ของตัวเองไว้โดยไม่ปิด** kernel จะยังคิดว่า "ยังมีคนอาจเขียนเพิ่ม
ได้อยู่นะ" และจะไม่ส่ง EOF ให้ — ผลคือ `read()` จะบล็อกรอตลอดกาล

มาดูโค้ดที่มีบั๊กนี้ชัดๆ:

```c
/* ============================================================
 * ชื่อไฟล์:     pipe_bug.c
 * คำอธิบาย:     ตัวอย่าง "บั๊กคลาสสิก" ของ pipe — ลืมปิด write-end ที่ไม่ใช้
 *              ทำให้ child ค้าง (block) ที่ read() ตลอดกาล เพราะไม่มีวันเจอ EOF
 * ============================================================ */
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/wait.h>

#define BUF_SIZE 256

int main(void) {
    int fd[2];

    if (pipe(fd) == -1) {
        perror("pipe");
        exit(EXIT_FAILURE);
    }

    pid_t pid = fork();
    if (pid == -1) {
        perror("fork");
        exit(EXIT_FAILURE);
    }

    if (pid == 0) {
        /* BUG: child ลืม close(fd[1]) ทำให้ child เองถือ write-end ค้างไว้ */
        char buf[BUF_SIZE];
        ssize_t n;
        while ((n = read(fd[0], buf, sizeof(buf) - 1)) > 0) {
            buf[n] = '\0';
            printf("[child] ได้รับ: %s", buf);
            fflush(stdout);
        }
        printf("[child] จะไม่มีทาง printf บรรทัดนี้ได้เลย เพราะ read() ค้างตลอดกาล\n");
        close(fd[0]);
        exit(EXIT_SUCCESS);
    } else {
        const char *msg = "ข้อความเดียว\n";
        write(fd[1], msg, strlen(msg));
        close(fd[1]); /* parent ปิดของตัวเองแล้ว แต่ child ยังถือ write-end อีกชุดอยู่! */
        wait(NULL);
        printf("[parent] จบแล้ว (แต่จริงๆ โปรแกรมนี้จะไม่มีวันมาถึงบรรทัดนี้ถ้า child ค้าง)\n");
    }

    return 0;
}
```

ลองคอมไพล์และรันโดยจำกัดเวลาไว้ด้วย `timeout` (ป้องกันไม่ให้ terminal ค้างจริงๆ):

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 pipe_bug.c -o pipe_bug
timeout 3 ./pipe_bug
echo "exit code: $?"
```

ผลลัพธ์:

```
[child] ได้รับ: ข้อความเดียว
exit code: 124
```

จะเห็นว่า child อ่านข้อความแรกได้ปกติ แต่หลังจากนั้น `read()` รอบถัดไปจะ**ค้างตลอดกาล**
เพราะยังมี write-end (ของ child เอง) เปิดค้างอยู่ในระบบ `exit code 124` คือรหัสมาตรฐาน
ที่คำสั่ง `timeout` ใช้บอกว่า "โปรแกรมไม่จบเองภายในเวลาที่กำหนด เลยต้องฆ่าทิ้ง"

> **กฎทองของการใช้ pipe**: ทันทีที่ทราบว่า process ไหนมีบทบาทอ่านหรือเขียน ให้ `close()`
> ฝั่งตรงข้ามที่ไม่ได้ใช้**ทันที** ก่อนเริ่ม logic อื่นใดๆ ทั้งฝั่ง parent และ child ต้องปิดให้ครบ
> ไม่ใช่แค่ฝั่งเดียว

### ตารางสรุป: ใครควรปิดอะไร

| Process | บทบาท | ต้องปิด |
|---|---|---|
| Parent (เขียน) | Writer | `close(fd[0])` ปิด read-end |
| Child (อ่าน) | Reader | `close(fd[1])` ปิด write-end |

---

## 29.5 การสื่อสารสองทาง (Bidirectional) ด้วย Pipe สองเส้น (Step 229)

เพราะ pipe หนึ่งเส้นสื่อสารได้ทางเดียวเท่านั้น ถ้าต้องการให้ parent ส่งงานให้ child แล้ว
child ส่งผลลัพธ์กลับมา ต้องสร้าง pipe สองเส้นแยกกัน — เส้นหนึ่งสำหรับ parent→child
อีกเส้นสำหรับ child→parent

```
              to_child pipe
   Parent  ─────────────────▶  Child
   (เขียน to_child[1])          (อ่าน to_child[0])

              to_parent pipe
   Parent  ◀─────────────────  Child
   (อ่าน to_parent[0])          (เขียน to_parent[1])
```

โค้ดตัวอย่างแบบเต็มอยู่ในหัวข้อ "แบบฝึกหัดท้ายบท" ข้อ 2 ด้านล่าง (เฉลยข้อ 2) ซึ่งสาธิต
parent ส่งเลขจำนวนเต็ม 5 ตัวให้ child บวก แล้ว child ส่งผลรวมกลับมาให้ parent

ข้อจำกัดสำคัญของทั้ง anonymous pipe และวิธีสื่อสารสองทางแบบนี้คือ **ใช้ได้เฉพาะระหว่าง
process ที่เป็นญาติกันผ่าน `fork()` เท่านั้น** เพราะ file descriptor ของ pipe จะถูกส่งต่อ
ผ่านการ copy ตอน `fork()` เท่านั้น ถ้าอยากให้ process สองตัวที่รันแยกกันตั้งแต่ต้น (คนละ
`./program` คนละครั้ง) คุยกัน ต้องใช้กลไกอีกแบบที่เรียกว่า **Named Pipe (FIFO)**

---

## 29.6 Named Pipe (FIFO): `mkfifo()` (Step 230)

FIFO (First In First Out) หรือ **Named Pipe** ทำงานเหมือน pipe ทุกประการ (byte stream,
ทิศทางเดียว, ผ่าน kernel buffer) แต่มีจุดต่างสำคัญคือ **มันมีชื่อปรากฏอยู่ในระบบไฟล์จริง**
ทำให้ process ใดๆ ก็ตามที่รู้ path ของมันสามารถเปิดมันขึ้นมาอ่าน/เขียนได้ โดยไม่ต้องเป็น
ญาติกันทาง fork เลย

```c
#include <sys/stat.h>

int mkfifo(const char *pathname, mode_t mode);
```

- `pathname` คือ path ของไฟล์ FIFO ที่จะสร้าง (เช่น `/tmp/my_fifo`)
- `mode` คือสิทธิ์การเข้าถึง (คล้าย `chmod`) เช่น `0666` ให้ทุกคนอ่านเขียนได้
- คืนค่า `0` เมื่อสำเร็จ, `-1` เมื่อผิดพลาด — ถ้า FIFO มีอยู่แล้วจะได้ `errno == EEXIST`
  (ไม่ใช่ error ร้ายแรง แค่แปลว่ามีอยู่แล้วจาก process ก่อนหน้า)

เมื่อสร้างแล้ว ลองดูด้วยคำสั่ง `ls -l` จะเห็นตัวอักษร `p` (pipe) นำหน้าสิทธิ์:

```bash
mkfifo /tmp/my_fifo
ls -l /tmp/my_fifo
# prw-rw-r-- 1 user user 0 Jan  1 00:00 /tmp/my_fifo
```

### พฤติกรรม Blocking ที่สำคัญของ FIFO

จุดที่แตกต่างจาก anonymous pipe อย่างชัดเจนคือพฤติกรรมตอน `open()`:

- `open(path, O_RDONLY)` จะ**บล็อก**รอจนกว่าจะมี process อื่นมา `open()` แบบ `O_WRONLY`
- `open(path, O_WRONLY)` จะ**บล็อก**รอจนกว่าจะมี process อื่นมา `open()` แบบ `O_RDONLY`

พูดง่ายๆ คือ FIFO จะไม่ยอมให้ฝั่งใดฝั่งหนึ่งเปิดสำเร็จ จนกว่าจะมี "คู่สนทนา" มาปรากฏตัว
ครบทั้งสองฝั่งก่อน — เหมือนการนัดเจอกัน ถ้าไปถึงคนเดียวต้องยืนรอจนกว่าอีกฝ่ายจะมา

```
Writer                          Reader
  │                                │
  │ open(O_WRONLY)                  │ open(O_RDONLY)
  │ ...บล็อกรอ...                   │ ...บล็อกรอ...
  │                                │
  └──────────── จับคู่กันสำเร็จ ─────┘
  │                                │
  │ write() ──────────────────────▶│ read()
```

---

## 29.7 ตัวอย่างจริง: สองโปรแกรมคุยกันผ่าน FIFO (Step 231)

มาสร้างโปรแกรมสองตัวแยกกันจริงๆ ที่คอมไพล์เป็นคนละไฟล์ executable และรันคนละ
process กัน โดยคุยกันผ่าน FIFO

**ไฟล์ที่ 1: `fifo_writer.c`**

```c
/* ============================================================
 * ชื่อไฟล์:     fifo_writer.c
 * คำอธิบาย:     เขียนข้อความลง Named Pipe (FIFO) เพื่อส่งให้ process อื่น
 *              ที่ไม่ใช่ parent-child กัน (คนละโปรแกรม รันแยกกันได้)
 * ============================================================ */
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <fcntl.h>
#include <unistd.h>
#include <sys/stat.h>
#include <errno.h>

#define FIFO_PATH "/tmp/course_chat_fifo"

int main(void) {
    /* สร้าง FIFO ถ้ายังไม่มี (ถ้ามีอยู่แล้วจาก process อื่น mkfifo จะคืน EEXIST) */
    if (mkfifo(FIFO_PATH, 0666) == -1 && errno != EEXIST) {
        perror("mkfifo");
        exit(EXIT_FAILURE);
    }

    printf("[writer] กำลังเปิด FIFO เพื่อเขียน (จะรอจนกว่ามี reader มาเปิดอ่าน)...\n");
    fflush(stdout);

    /* open() แบบ O_WRONLY จะ "บล็อก" รอจนกว่าจะมีอีกฝั่งเปิดแบบ O_RDONLY */
    int fd = open(FIFO_PATH, O_WRONLY);
    if (fd == -1) {
        perror("open");
        exit(EXIT_FAILURE);
    }

    printf("[writer] มี reader เชื่อมต่อแล้ว เริ่มส่งข้อความ\n");

    const char *messages[] = {
        "Hello from writer process!\n",
        "FIFO ทำให้ process คนละต้นไม้สื่อสารกันได้\n",
        "ข้อความสุดท้ายแล้ว\n"
    };

    for (int i = 0; i < 3; i++) {
        write(fd, messages[i], strlen(messages[i]));
        printf("[writer] ส่งแล้ว: %s", messages[i]);
        fflush(stdout);
    }

    close(fd); /* ปิด write-end -> ทำให้ reader ฝั่งโน้นเจอ EOF */
    printf("[writer] ปิด FIFO แล้ว จบการทำงาน\n");

    return 0;
}
```

**ไฟล์ที่ 2: `fifo_reader.c`**

```c
/* ============================================================
 * ชื่อไฟล์:     fifo_reader.c
 * คำอธิบาย:     อ่านข้อความจาก Named Pipe (FIFO) ที่ fifo_writer.c เขียนมา
 * ============================================================ */
#include <stdio.h>
#include <stdlib.h>
#include <fcntl.h>
#include <unistd.h>
#include <sys/stat.h>
#include <errno.h>

#define FIFO_PATH "/tmp/course_chat_fifo"
#define BUF_SIZE 256

int main(void) {
    if (mkfifo(FIFO_PATH, 0666) == -1 && errno != EEXIST) {
        perror("mkfifo");
        exit(EXIT_FAILURE);
    }

    printf("[reader] กำลังเปิด FIFO เพื่ออ่าน (จะรอจนกว่ามี writer มาเชื่อมต่อ)...\n");
    fflush(stdout);

    int fd = open(FIFO_PATH, O_RDONLY);
    if (fd == -1) {
        perror("open");
        exit(EXIT_FAILURE);
    }

    printf("[reader] มี writer เชื่อมต่อแล้ว เริ่มรับข้อความ\n");

    char buf[BUF_SIZE];
    ssize_t n;
    while ((n = read(fd, buf, sizeof(buf) - 1)) > 0) {
        buf[n] = '\0';
        printf("[reader] ได้รับ: %s", buf);
        fflush(stdout);
    }

    printf("[reader] เจอ EOF แล้ว (writer ปิดการเชื่อมต่อ) จบการทำงาน\n");
    close(fd);

    /* ลบไฟล์ FIFO ทิ้งเมื่อใช้งานเสร็จ */
    unlink(FIFO_PATH);

    return 0;
}
```

คอมไพล์ทั้งสองไฟล์แยกกัน แล้วเปิด **สอง terminal** รันคนละหน้าต่าง (หรือรัน reader
เป็น background process แบบด้านล่างก็ได้):

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 fifo_writer.c -o fifo_writer
gcc -Wall -Wextra -Wpedantic -std=c17 fifo_reader.c -o fifo_reader

# Terminal 1
./fifo_reader

# Terminal 2 (เปิดแยกอีกหน้าต่าง)
./fifo_writer
```

ผลลัพธ์ฝั่ง writer:

```
[writer] กำลังเปิด FIFO เพื่อเขียน (จะรอจนกว่ามี reader มาเปิดอ่าน)...
[writer] มี reader เชื่อมต่อแล้ว เริ่มส่งข้อความ
[writer] ส่งแล้ว: Hello from writer process!
[writer] ส่งแล้ว: FIFO ทำให้ process คนละต้นไม้สื่อสารกันได้
[writer] ส่งแล้ว: ข้อความสุดท้ายแล้ว
[writer] ปิด FIFO แล้ว จบการทำงาน
```

ผลลัพธ์ฝั่ง reader:

```
[reader] กำลังเปิด FIFO เพื่ออ่าน (จะรอจนกว่ามี writer มาเชื่อมต่อ)...
[reader] มี writer เชื่อมต่อแล้ว เริ่มรับข้อความ
[reader] ได้รับ: Hello from writer process!
FIFO ทำให้ process คนละต้นไม้สื่อสารกันได้
ข้อความสุดท้ายแล้ว
[reader] เจอ EOF แล้ว (writer ปิดการเชื่อมต่อ) จบการทำงาน
```

สังเกตว่า **reader อ่านทั้ง 3 ข้อความออกมาในการเรียก `read()` ครั้งเดียว** (printf
แสดงรวมกันเป็นก้อนเดียว) นี่คือหลักฐานยืนยันสิ่งที่บอกไว้ในหัวข้อ 29.2 — pipe/FIFO เป็น
**byte stream** ไม่ใช่ message queue จึงไม่รักษาขอบเขตของแต่ละ `write()` ให้ ถ้าต้องการ
ส่งเป็น "ข้อความ" ที่แยกขอบเขตชัดเจน ต้องออกแบบ protocol เอง (เช่น คั่นด้วย `\n` แล้ว
parse ทีละบรรทัด) หรือใช้ message queue ใน Part 30 แทน

---

## 29.8 Blocking Server Pattern สำหรับ FIFO (Step 232)

ปัญหาหนึ่งของโปรแกรม `fifo_reader.c` ด้านบนคือ มันอ่านได้แค่รอบเดียวแล้วจบ ถ้ามี writer
รายใหม่มาเชื่อมต่ออีกครั้ง reader ตัวเดิมจะไม่รับรู้อะไรเลยเพราะปิดตัวไปแล้ว ในโลกจริง
เรามักอยากให้โปรแกรมฝั่งอ่าน (เปรียบเหมือน "เซิร์ฟเวอร์") **ทำงานค้างไว้ตลอด** และรับ
ข้อความจาก writer หลายๆ รายที่ผลัดกันเข้ามาเชื่อมต่อได้เรื่อยๆ

หลักการคือ: เมื่อ `read()` คืนค่า `0` (EOF เพราะ writer ตัวก่อนหน้าปิดไปแล้ว) แทนที่จะ
จบโปรแกรม ให้ `close()` fd เดิมแล้ววนกลับไป `open()` ใหม่อีกรอบ ซึ่งจะไปบล็อกรอ writer
รายถัดไปโดยอัตโนมัติ

```
   ┌─────────────────────────────────────────────┐
   │  while (ไม่ได้รับสัญญาณให้หยุด) {                │
   │      open(FIFO, O_RDONLY);   // บล็อกรอ writer  │
   │      while (read() > 0) { ... }  // อ่านจนเจอ EOF │
   │      close(fd);              // writer ตัวนี้จบแล้ว│
   │  }  // วนกลับไปรอ writer ตัวถัดไป                  │
   └─────────────────────────────────────────────┘
```

รูปแบบนี้คือ pattern พื้นฐานที่โปรแกรมเซิร์ฟเวอร์แบบง่ายๆ จำนวนมากใช้ (ก่อนที่จะไปเรียน
เรื่อง socket server เต็มรูปแบบใน Part 33-35) โค้ดตัวอย่างแบบเต็มของ pattern นี้อยู่ใน
เฉลยแบบฝึกหัดข้อ 4 ด้านล่าง ซึ่งใช้ `sigaction()` (ทบทวนจาก Part 28) ร่วมกับการวนลูป
`open()`/`read()`/`close()` เพื่อสร้างโปรแกรมรับข้อความที่หยุดได้อย่างปลอดภัยด้วย Ctrl+C

---

## ข้อผิดพลาดที่พบบ่อย (Common Pitfalls)

1. **ลืมปิด file descriptor ฝั่งที่ไม่ได้ใช้** — คือบั๊กอันดับหนึ่งของการใช้ pipe ตามที่
   สาธิตในหัวข้อ 29.4 ผลคือโปรแกรมค้าง (hang) เพราะ `read()` ไม่มีวันเจอ EOF ให้ตรวจสอบ
   ทุกครั้งหลัง `fork()` ว่า process ฝั่งไหนใช้ fd ไหน แล้ว `close()` ฝั่งตรงข้ามทันที
2. **เรียก `pipe()` หลัง `fork()`** — ทำให้ parent กับ child ได้ pipe คนละเส้นกัน ไม่มีทาง
   สื่อสารกันได้เลย ต้องเรียก `pipe()` **ก่อน** `fork()` เสมอ
3. **คาดหวังว่า pipe จะรักษาขอบเขตของแต่ละ `write()`** — pipe เป็น byte stream ล้วนๆ
   การเขียน 3 ครั้งฝั่งหนึ่งอาจถูกอ่านออกมาเป็นการเรียก `read()` เพียงครั้งเดียวฝั่งรับ
   (ดังที่เห็นในหัวข้อ 29.7) ถ้าต้องการแยกขอบเขตข้อความ ต้องออกแบบ protocol เอง เช่น
   ใส่ length-prefix หรือ delimiter อย่าง `\n`
4. **ไม่ตรวจสอบค่าที่คืนจาก `read()`/`write()`** — ทั้งสองฟังก์ชันอาจคืนค่าน้อยกว่าที่ขอ
   (partial read/write) โดยเฉพาะเมื่อข้อมูลมีขนาดใหญ่ ต้องวนลูปอ่าน/เขียนจนครบ หรือ
   อย่างน้อยต้องเช็คค่าที่คืนกลับมาทุกครั้ง (compiler จะเตือนด้วย `-Wunused-result` ถ้า
   เปิด optimization และไม่ได้เช็คค่า)
5. **ลืมว่า pipe ใช้ได้แค่ระหว่างญาติกัน** — พยายามใช้ `int fd[2]` จาก `pipe()` ข้าม
   process ที่รันแยกกันตั้งแต่ต้น (คนละคำสั่ง `./program`) จะไม่มีทางทำงานได้ เพราะ
   file descriptor เป็นของ process นั้นๆ เท่านั้น ไม่ได้ถูกแชร์ข้าม process โดยอัตโนมัติ
   ถ้าต้องการแบบนั้นต้องใช้ FIFO (`mkfifo`) แทน
6. **ลืม `unlink()` ไฟล์ FIFO หลังใช้งานเสร็จ** — ไฟล์ FIFO จะยังค้างอยู่ในระบบไฟล์แม้
   โปรแกรมจะปิดไปแล้ว ถ้ารันโปรแกรมซ้ำและลืมลบไฟล์เดิม อาจได้ `errno == EEXIST` ตอน
   เรียก `mkfifo()` ซ้ำ (ไม่ใช่ error ร้ายแรงถ้าเช็ค `errno` ให้ถูก แต่ควรทำความสะอาดเสมอ)

---

## แบบฝึกหัดท้ายบท

1. แก้ไข `pipe_basic.c` ให้ parent ส่ง **เลขจำนวนเต็ม** (ไม่ใช่ string) ทีละตัวจำนวน 10 ตัว
   ให้ child อ่านแล้วคำนวณผลรวมออกมาพิมพ์
2. เขียนโปรแกรมที่ใช้ pipe **สองเส้น** ให้ parent ส่งเลขจำนวนเต็ม 5 ตัวให้ child คำนวณ
   ผลรวม แล้ว child ส่งผลลัพธ์กลับมาให้ parent พิมพ์ออกทางหน้าจอ (สื่อสารสองทาง)
3. ทดลองรันโปรแกรม `pipe_bug.c` ด้วยตัวเองผ่าน `timeout 3 ./pipe_bug` แล้วอธิบายด้วย
   คำพูดตัวเองว่าทำไม `read()` ถึงไม่คืนค่า 0 สักที ทั้งที่ parent ปิด write-end ของตัวเอง
   ไปแล้ว
4. เขียนโปรแกรมฝั่ง "เซิร์ฟเวอร์" ที่เปิด FIFO รออ่านข้อความ **ไม่รู้จบ** (วนลูปเปิดใหม่
   ทุกครั้งที่เจอ EOF) และหยุดได้อย่างปลอดภัยเมื่อกด Ctrl+C (ใช้ `sigaction` จับ `SIGINT`)
   จากนั้นเขียนโปรแกรมฝั่ง "ไคลเอนต์" ที่รับ argument จาก command line แล้วส่งข้อความ
   นั้นไปให้เซิร์ฟเวอร์ ทดสอบรันไคลเอนต์หลายครั้งติดกันว่าเซิร์ฟเวอร์รับข้อความได้ครบทุกครั้ง
5. อธิบายว่าทำไมคำสั่ง shell `cmd1 | cmd2 | cmd3` ถึงทำงานได้เร็วโดยที่ `cmd2` ไม่ต้องรอ
   `cmd1` ทำงานเสร็จสมบูรณ์ก่อนถึงจะเริ่มทำงาน (คำใบ้: เกี่ยวกับ buffer ขนาด 64 KB และ
   การที่ process ทั้งสามรันพร้อมกัน)
6. ลองใช้คำสั่ง `strace -f -e trace=pipe,fork,clone,close,read,write ./pipe_basic` (ถ้า
   เครื่องมี `strace` ติดตั้งอยู่) เพื่อดู syscall จริงที่เกิดขึ้นเบื้องหลังโปรแกรม `pipe_basic.c`
   สังเกตลำดับการเรียก `pipe()`, `fork()`/`clone()`, `close()`, และ `read()`/`write()`

### แนวทางเฉลยข้อ 2 (สื่อสารสองทางด้วย pipe คู่)

```c
/* ============================================================
 * ชื่อไฟล์:     ex_two_way_pipe.c
 * คำอธิบาย:     เฉลยแบบฝึกหัด — สื่อสารสองทางระหว่าง parent-child ด้วย pipe 2 เส้น
 *              parent ส่งเลขจำนวนเต็ม N ตัวให้ child, child บวกผลรวมแล้วส่งกลับ
 * ============================================================ */
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/wait.h>

#define COUNT 5

int main(void) {
    int to_child[2];   /* parent เขียน -> child อ่าน */
    int to_parent[2];  /* child เขียน  -> parent อ่าน */

    if (pipe(to_child) == -1 || pipe(to_parent) == -1) {
        perror("pipe");
        exit(EXIT_FAILURE);
    }

    pid_t pid = fork();
    if (pid == -1) {
        perror("fork");
        exit(EXIT_FAILURE);
    }

    if (pid == 0) {
        /* ---------- Child ---------- */
        close(to_child[1]);  /* child ไม่เขียนฝั่งนี้ */
        close(to_parent[0]); /* child ไม่อ่านฝั่งนี้ */

        int numbers[COUNT];
        ssize_t n = read(to_child[0], numbers, sizeof(numbers));
        if (n != (ssize_t)sizeof(numbers)) {
            fprintf(stderr, "[child] อ่านข้อมูลไม่ครบ\n");
            exit(EXIT_FAILURE);
        }
        close(to_child[0]);

        int sum = 0;
        for (int i = 0; i < COUNT; i++) {
            sum += numbers[i];
        }
        printf("[child] คำนวณผลรวมได้ %d ส่งกลับไปให้ parent\n", sum);

        write(to_parent[1], &sum, sizeof(sum));
        close(to_parent[1]);
        exit(EXIT_SUCCESS);
    } else {
        /* ---------- Parent ---------- */
        close(to_child[0]);  /* parent ไม่อ่านฝั่งนี้ */
        close(to_parent[1]); /* parent ไม่เขียนฝั่งนี้ */

        int numbers[COUNT] = {10, 20, 30, 40, 50};
        write(to_child[1], numbers, sizeof(numbers));
        close(to_child[1]); /* ปิดหลังส่งครบ */

        int result;
        ssize_t n = read(to_parent[0], &result, sizeof(result));
        if (n != (ssize_t)sizeof(result)) {
            fprintf(stderr, "[parent] อ่านผลลัพธ์ไม่ครบ\n");
            exit(EXIT_FAILURE);
        }
        close(to_parent[0]);

        printf("[parent] ได้รับผลรวมจาก child: %d\n", result);
        wait(NULL);
    }

    return 0;
}
```

คอมไพล์และรัน:

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 ex_two_way_pipe.c -o ex_two_way_pipe
./ex_two_way_pipe
```

ผลลัพธ์ที่ได้จริง:

```
[child] คำนวณผลรวมได้ 150 ส่งกลับไปให้ parent
[parent] ได้รับผลรวมจาก child: 150
```

สังเกตว่าตัวอย่างนี้ส่ง `int` ดิบๆ ผ่าน `write()`/`read()` โดยตรง (ไม่ใช่ string) ซึ่งทำได้
เพราะ parent กับ child รันบนเครื่องเดียวกัน ใช้ compiler เดียวกัน จึงมี memory layout ของ
`int` เหมือนกันทุกประการ — ถ้าเป็นการสื่อสารข้ามเครื่อง (เช่นผ่าน socket ใน Part 33) จะ
ต้องระวังเรื่อง byte order (endianness) เพิ่มเติม

### แนวทางเฉลยข้อ 4 (เซิร์ฟเวอร์ FIFO ที่รับหลาย client)

**ไฟล์เซิร์ฟเวอร์:**

```c
/* ============================================================
 * ชื่อไฟล์:     ex_fifo_persistent_reader.c
 * คำอธิบาย:     เฉลยแบบฝึกหัด — โปรแกรมอ่าน FIFO ที่ไม่ปิดตัวเมื่อเจอ EOF
 *              แต่วนกลับไปเปิดอ่านใหม่ เพื่อรองรับ writer หลายตัวที่ผลัดกัน
 *              เชื่อมต่อเข้ามาเรื่อยๆ (คล้าย pattern ของ logging server จริง)
 * ============================================================ */
#define _POSIX_C_SOURCE 200809L
#include <stdio.h>
#include <stdlib.h>
#include <fcntl.h>
#include <unistd.h>
#include <sys/stat.h>
#include <errno.h>
#include <signal.h>

#define FIFO_PATH "/tmp/course_ex_fifo"
#define BUF_SIZE 256

static volatile sig_atomic_t g_stop = 0;

static void handle_sigint(int signo) {
    (void)signo;
    g_stop = 1;
}

int main(void) {
    if (mkfifo(FIFO_PATH, 0666) == -1 && errno != EEXIST) {
        perror("mkfifo");
        exit(EXIT_FAILURE);
    }

    struct sigaction sa;
    sa.sa_handler = handle_sigint;
    sigemptyset(&sa.sa_mask);
    sa.sa_flags = 0;
    sigaction(SIGINT, &sa, NULL);

    printf("[server] เริ่มรอรับข้อความ กด Ctrl+C เพื่อหยุด\n");

    while (!g_stop) {
        int fd = open(FIFO_PATH, O_RDONLY); /* บล็อกรอ writer ตัวถัดไป */
        if (fd == -1) {
            if (errno == EINTR) {
                continue; /* ถูก interrupt ด้วย signal ระหว่างรอ ให้ลองใหม่/เช็ค g_stop */
            }
            perror("open");
            break;
        }

        char buf[BUF_SIZE];
        ssize_t n;
        while ((n = read(fd, buf, sizeof(buf) - 1)) > 0) {
            buf[n] = '\0';
            printf("[server] ได้รับ: %s", buf);
        }

        close(fd); /* writer ตัวนี้ปิดแล้ว (EOF) กลับไปวน loop เปิดรอ writer ตัวถัดไป */
        printf("[server] writer ตัวก่อนหน้าตัดการเชื่อมต่อแล้ว รอ writer ตัวถัดไป...\n");
    }

    printf("[server] ได้รับสัญญาณหยุด ปิดโปรแกรม\n");
    unlink(FIFO_PATH);
    return 0;
}
```

**ไฟล์ไคลเอนต์:**

```c
#define _POSIX_C_SOURCE 200809L
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <fcntl.h>
#include <unistd.h>

#define FIFO_PATH "/tmp/course_ex_fifo"

int main(int argc, char *argv[]) {
    if (argc != 2) {
        fprintf(stderr, "usage: %s <message>\n", argv[0]);
        exit(EXIT_FAILURE);
    }
    int fd = open(FIFO_PATH, O_WRONLY);
    if (fd == -1) {
        perror("open");
        exit(EXIT_FAILURE);
    }
    char line[300];
    snprintf(line, sizeof(line), "%s\n", argv[1]);
    write(fd, line, strlen(line));
    close(fd);
    return 0;
}
```

คอมไพล์และทดสอบ (รัน server ใน background ด้วย `&` เพื่อทดสอบในเทอร์มินัลเดียว
สำหรับเดโม จริงๆ ควรเปิดคนละหน้าต่าง):

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 ex_fifo_persistent_reader.c -o ex_fifo_server
gcc -Wall -Wextra -Wpedantic -std=c17 ex_fifo_client.c -o ex_fifo_client

./ex_fifo_server &
sleep 0.3
./ex_fifo_client "ข้อความจาก client ตัวที่ 1"
./ex_fifo_client "ข้อความจาก client ตัวที่ 2"
```

ผลลัพธ์ฝั่งเซิร์ฟเวอร์:

```
[server] เริ่มรอรับข้อความ กด Ctrl+C เพื่อหยุด
[server] ได้รับ: ข้อความจาก client ตัวที่ 1
[server] writer ตัวก่อนหน้าตัดการเชื่อมต่อแล้ว รอ writer ตัวถัดไป...
[server] ได้รับ: ข้อความจาก client ตัวที่ 2
[server] writer ตัวก่อนหน้าตัดการเชื่อมต่อแล้ว รอ writer ตัวถัดไป...
```

เซิร์ฟเวอร์รับข้อความจาก client ทั้งสองครั้งได้สำเร็จโดยไม่ต้องปิดตัวเองระหว่างนั้นเลย
เพราะ pattern การวน `open()`-`read()`-`close()` ทำให้มันพร้อมรับ writer รายใหม่เสมอ

---

## สรุปท้ายบท

ใน Part นี้เราได้:

- เข้าใจว่าทำไม process ต้องมีกลไก IPC เพราะ `fork()` ทำให้ address space แยกจากกัน
  อย่างสมบูรณ์ ต่างจาก thread ที่แชร์ memory กัน
- ใช้ `pipe()` สร้าง anonymous pipe และเข้าใจว่ามันคือ file descriptor คู่ที่เชื่อมกันทาง
  เดียวผ่าน kernel buffer
- เขียนโปรแกรมจริงให้ parent-child คุยกันผ่าน pipe พร้อมเรียนรู้กฎทองที่สำคัญที่สุด: ต้อง
  ปิด file descriptor ฝั่งที่ไม่ได้ใช้เสมอ ไม่งั้นโปรแกรมจะค้างเพราะไม่มีวันเจอ EOF
- ขยายไปสู่การสื่อสารสองทางด้วย pipe สองเส้น
- ใช้ `mkfifo()` สร้าง Named Pipe เพื่อให้ process ที่ไม่ใช่ญาติกันสื่อสารกันได้ผ่านระบบไฟล์
- เขียนโปรแกรมสองตัวแยกกันจริงๆ คุยกันผ่าน FIFO ได้สำเร็จ พร้อมเข้าใจพฤติกรรม blocking
  ของ `open()` และออกแบบ pattern เซิร์ฟเวอร์ที่รองรับ client หลายรายต่อเนื่อง

Pipe และ FIFO เป็นกลไก IPC ที่เรียบง่ายแต่มีข้อจำกัดสำคัญ 2 อย่างคือ **ความเร็ว** (ข้อมูล
ต้อง copy ผ่าน kernel buffer สองรอบ คือ write เข้า kernel แล้ว read ออกมา) และ **การไม่
รักษาขอบเขตข้อความ** (เป็น byte stream ล้วนๆ) ใน **Part 30** เราจะเรียนกลไก IPC ที่
เร็วกว่ามาก — **Shared Memory** ซึ่งไม่ต้อง copy ข้อมูลผ่าน kernel เลย และ **Message
Queue** ซึ่งรักษาขอบเขตของแต่ละข้อความให้อัตโนมัติ พร้อมทั้งเรียนรู้ปัญหาใหม่ที่ตามมาเมื่อ
หลาย process เข้าถึงหน่วยความจำเดียวกันพร้อมกัน นั่นคือ **Race Condition**

**ต่อไป:** [Part 30 — Shared Memory และ Message Queue](./part-030-shared-memory-mq.md)
