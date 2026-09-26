# Part 27: Process Management (fork/exec/wait) (Step 209–216)

> Module C — Systems Programming ด้วย C บน Linux | Part 27 จาก 125
> Step ที่ครอบคลุมใน Part นี้: Step 209–216
> Part ก่อนหน้า: [Part 26 — พื้นฐานระบบปฏิบัติการสำหรับโปรแกรมเมอร์](./part-026-os-concepts.md) | Part ถัดไป: [Part 28 — Signal และ Signal Handling](./part-028-signals.md)

## เป้าหมายการเรียนรู้ (Learning Objectives)

หลังจบ Part นี้ ผู้เรียนจะสามารถ:

1. อธิบายและใช้งาน **`fork()`** ได้อย่างถูกต้อง เข้าใจว่าทำไม Return Value ถึงต่างกันใน
   Parent และ Child พร้อมอธิบายกลไก **Copy-on-Write (COW)** ที่ทำให้ `fork()` เร็วได้จริง
2. ใช้ตระกูล **`exec()`** (`execl`, `execvp`, `execve`) เพื่อแทนที่ Process ปัจจุบันด้วย
   โปรแกรมใหม่ทั้งหมด และแยกแยะความแตกต่างของแต่ละตัวได้
3. ประกอบ **fork() + exec()** เข้าด้วยกันเป็น Pattern มาตรฐานที่ใช้สร้าง Process ลูกเพื่อ
   รันโปรแกรมอื่น ซึ่งเป็นกลไกเบื้องหลังของทุก Shell
4. ใช้ **`wait()`/`waitpid()`** เพื่อรอ Child Process และดึงค่า Exit Status ออกมาด้วย Macro
   มาตรฐาน (`WIFEXITED`, `WEXITSTATUS`, `WIFSIGNALED`, `WTERMSIG`)
5. อธิบายได้ว่า **Zombie Process** คืออะไร เกิดขึ้นได้อย่างไร และป้องกัน/แก้ไขอย่างไร
6. อธิบายได้ว่า **Orphan Process** คืออะไร และเข้าใจกลไกการ Reparent ไปยัง `init`/`reaper`
7. เขียน **Mini Shell** ของตัวเองที่รับคำสั่งจากผู้ใช้ แล้ว `fork()` + `exec()` รันคำสั่งนั้น
   จริง พร้อมรองรับ Built-in Command พื้นฐาน (`cd`, `exit`)

---

## บทนำ: จากทฤษฎีสู่การสร้าง Process ด้วยตัวเอง

ใน Part 26 เราเรียนรู้ว่า Process คืออะไรและมี Memory Layout อย่างไร แต่ยังไม่เคย**สร้าง
Process ใหม่ขึ้นมาเอง**เลยสักครั้ง — ทุกโปรแกรมที่เราเขียนมาตลอด 26 Part ถูกสร้างโดย Shell
(bash) ให้เราโดยที่เราไม่รู้ตัว

Part นี้คือจุดเปลี่ยนสำคัญ: เราจะเรียนรู้ System Call ที่ทรงพลังที่สุดตัวหนึ่งของ Unix/Linux
คือ **`fork()`** ซึ่งเป็นรากฐานของทุกสิ่งที่เกี่ยวกับ Multi-process Programming ตั้งแต่ Shell,
Web Server, Database ไปจนถึง Container Runtime อย่าง Docker ล้วนใช้ `fork()` (หรือ System
Call ที่พัฒนาต่อยอดมาจากมัน) เป็นรากฐานทั้งสิ้น

---

## 27.1 `fork()` พื้นฐาน: หนึ่ง Process กลายเป็นสอง (Step 209)

`fork()` เป็น System Call ที่แปลกและทรงพลังที่สุดตัวหนึ่งในโลกโปรแกรมมิ่ง เพราะมันทำสิ่งที่
ฟังก์ชันทั่วไป**ไม่เคยทำ**: มันถูกเรียก**ครั้งเดียว** แต่ **`return` สองครั้ง**!

เมื่อ Process A เรียก `fork()` Kernel จะสร้าง Process ใหม่ (เรียกว่า **Child**) ที่เป็น
**สำเนา (Copy) เกือบสมบูรณ์แบบ** ของ Process เดิม (เรียกว่า **Parent**) — มีตัวแปรค่า
เดียวกันทุกตัว, ตำแหน่งที่กำลังรันโค้ดอยู่ ณ บรรทัดเดียวกัน, File Descriptor ที่เปิดอยู่ชุด
เดียวกัน จากนั้นทั้งสอง Process จะทำงาน**ต่อจากบรรทัดถัดจาก `fork()` พร้อมกัน**เป็นอิสระ
จากกันโดยสิ้นเชิง

```
ก่อน fork()                          หลัง fork()
┌─────────────┐                     ┌─────────────┐      ┌─────────────┐
│  Process A  │      fork()         │  Process A  │      │  Process B  │
│  (PID=100)  │  ─────────────►     │  (PID=100)  │      │  (PID=101)  │
│             │                     │  "Parent"   │      │  "Child"    │
└─────────────┘                     │ fork()=101  │      │ fork()=0    │
                                     └─────────────┘      └─────────────┘
                                     ทำงานต่อจากบรรทัดถัดไป "พร้อมกัน" เป็นอิสระต่อกัน
```

### สิ่งที่ทำให้ Parent กับ Child แยกออกจากกันได้: Return Value ของ `fork()`

`fork()` คืนค่าที่**แตกต่างกัน**ให้ Parent และ Child เพื่อให้โค้ดของเรารู้ว่ากำลังรันอยู่ใน
ฝั่งไหน:

| ใครเรียก | `fork()` คืนค่าอะไร | ความหมาย |
|---|---|---|
| **Parent** | PID ของ Child ที่เพิ่งสร้าง (ตัวเลข > 0) | ใช้ค่านี้อ้างอิงถึง Child ตัวนี้ในภายหลัง (เช่นตอนเรียก `waitpid()`) |
| **Child** | `0` เสมอ | Child เรียก `getpid()` เอาเองได้ถ้าอยากรู้ PID ตัวเอง ไม่จำเป็นต้องพึ่งค่าที่ `fork()` คืนมา |
| **ทั้งคู่ (กรณีล้มเหลว)** | `-1` (ไม่มีการสร้าง Child เลย) | ต้องตรวจสอบเสมอ! ดู Common Pitfalls |

```c
/* ============================================================
 * ชื่อไฟล์:     fork_basic.c
 * คำอธิบาย:     สาธิต fork() พื้นฐาน — ค่าที่ return แตกต่างกันใน Parent และ Child
 * ============================================================ */
#define _POSIX_C_SOURCE 200809L
#include <stdio.h>
#include <unistd.h>
#include <sys/wait.h>

int main(void) {
    printf("ก่อน fork(): PID=%ld\n", (long)getpid());
    fflush(stdout); /* กันข้อความซ้ำหลัง fork() — ดู Common Pitfalls ข้อ 1 */

    pid_t pid = fork();

    if (pid < 0) {
        /* fork() ล้มเหลว (เช่น หน่วยความจำไม่พอ หรือชนขีดจำกัดจำนวน process) */
        perror("fork ล้มเหลว");
        return 1;
    } else if (pid == 0) {
        /* โค้ดส่วนนี้รันเฉพาะใน Child Process เท่านั้น
           ใน Child, fork() คืนค่า 0 เสมอ */
        printf("[CHILD]  PID=%ld, PPID=%ld, fork() คืนค่า = %ld\n",
               (long)getpid(), (long)getppid(), (long)pid);
    } else {
        /* โค้ดส่วนนี้รันเฉพาะใน Parent Process เท่านั้น
           ใน Parent, fork() คืนค่า PID ของ Child ที่เพิ่งสร้าง */
        printf("[PARENT] PID=%ld, PPID=%ld, fork() คืนค่า = %ld (คือ PID ของลูก)\n",
               (long)getpid(), (long)getppid(), (long)pid);
        wait(NULL); /* รอให้ child จบก่อน (รายละเอียดเรื่อง wait() ในหัวข้อ 27.5) */
    }

    printf("บรรทัดนี้ถูกพิมพ์โดยทั้ง Parent และ Child (PID=%ld)\n", (long)getpid());
    return 0;
}
```

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 -D_POSIX_C_SOURCE=200809L fork_basic.c -o fork_basic
./fork_basic
```

```
ก่อน fork(): PID=3548
[PARENT] PID=3548, PPID=3540, fork() คืนค่า = 3549 (คือ PID ของลูก)
[CHILD]  PID=3549, PPID=3548, fork() คืนค่า = 0
บรรทัดนี้ถูกพิมพ์โดยทั้ง Parent และ Child (PID=3549)
บรรทัดนี้ถูกพิมพ์โดยทั้ง Parent และ Child (PID=3548)
```

จุดที่ต้องสังเกตให้ดี: **บรรทัดสุดท้ายถูกพิมพ์สองครั้ง** (ครั้งละ PID ต่างกัน) เพราะหลัง
`if/else if/else` จบลง โค้ดที่เหลือคือโค้ดร่วมที่ทั้ง Parent และ Child รันต่อทั้งคู่ — นี่คือ
ธรรมชาติของ `fork()` ที่ทำให้โค้ด**หนึ่งชุด**กลายเป็นการทำงาน**สองสาย**พร้อมกันโดยอัตโนมัติ
(ลำดับการพิมพ์ระหว่าง Parent กับ Child ในทางปฏิบัติไม่แน่นอน ขึ้นอยู่กับ Scheduler ว่าจะให้
ใครได้ CPU ก่อน)

---

## 27.2 Copy-on-Write (COW): ทำไม `fork()` ถึงเร็ว (Step 210)

คำถามที่น่าสงสัยคือ: ถ้า `fork()` ต้อง**คัดลอกหน่วยความจำทั้งหมด**ของ Parent ไปให้ Child
(Text, Data, BSS, Heap, Stack ทุก Segment ตามที่เรียนใน Part 26) แล้วโปรแกรมที่ใช้ RAM
หลาย GB จะ `fork()` ช้ามากไม่ใช่หรือ?

คำตอบคือ Linux **ไม่ได้คัดลอกจริงๆ ทันที** แต่ใช้เทคนิคที่เรียกว่า **Copy-on-Write (COW)**:

1. ตอน `fork()` เพิ่งเกิดขึ้น Kernel แค่สร้าง **Page Table ใหม่**ให้ Child โดยให้ทุก Entry
   **ชี้ไปที่ Physical Page เดียวกัน**กับของ Parent (ไม่มีการคัดลอกข้อมูลจริงเลยสักไบต์)
2. Page ทั้งหมดถูกทำเครื่องหมายเป็น **Read-only** ชั่วคราว (ทั้งฝั่ง Parent และ Child)
3. เมื่อฝั่งใดฝั่งหนึ่ง (Parent หรือ Child) พยายาม**เขียน**ข้อมูลลง Page ใดๆ เป็นครั้งแรก
   หลัง `fork()` — ตรงจุดนั้นเท่านั้นที่ CPU จะเกิด **Page Fault**, Kernel จะเข้ามาแทรกแซง
   คัดลอก Physical Page นั้น**เฉพาะหน้าที่ถูกเขียน**ให้กลายเป็นสำเนาแยกของฝ่ายที่เขียน แล้ว
   ค่อยอนุญาตให้เขียนต่อได้

```
ทันทีหลัง fork()                        หลังจาก Child เขียนค่าลงตัวแปร
┌─────────┐   ┌─────────┐              ┌─────────┐   ┌─────────┐
│ Parent  │   │  Child  │              │ Parent  │   │  Child  │
│PageTable│   │PageTable│              │PageTable│   │PageTable│
└────┬────┘   └────┬────┘              └────┬────┘   └────┬────┘
     │             │                        │             │
     └──────┬──────┘                        ▼             ▼
            ▼                          Physical Page   Physical Page
      Physical Page                    #A (เดิม, ของ    #B (สำเนาใหม่
      (ใช้ร่วมกัน                       Parent)          เฉพาะของ Child)
       ชั่วคราว)
```

**ประโยชน์**: `fork()` จึงเร็วมากแม้ Process จะใช้หน่วยความจำขนาดใหญ่ เพราะแทบไม่มีการ
คัดลอกข้อมูลจริงเกิดขึ้นเลยตอนเรียก และในหลายกรณี (เช่น `fork()` แล้วเรียก `exec()` ทันที —
ดูหัวข้อ 27.4) ก็ไม่จำเป็นต้องคัดลอกอะไรเพิ่มเติมเลยด้วยซ้ำ เพราะหน่วยความจำเดิมทั้งหมดถูก
ทิ้งไปตอน `exec()` แทนที่ image อยู่แล้ว

### พิสูจน์ COW ด้วยโค้ดจริง

```c
/* ============================================================
 * ชื่อไฟล์:     fork_cow.c
 * คำอธิบาย:     สาธิต Copy-on-Write — ทั้ง Parent/Child เห็นตัวแปรที่ address เดิม
 *              แต่แก้ไขค่าแล้วไม่กระทบกัน เพราะ kernel คัดลอกหน้าหน่วยความจำ
 *              (Page) จริงๆ ก็ต่อเมื่อมีการ "เขียน" เท่านั้น
 * ============================================================ */
#define _POSIX_C_SOURCE 200809L
#include <stdio.h>
#include <unistd.h>
#include <sys/wait.h>

int shared_looking_value = 100; /* อยู่ใน Data Segment */

int main(void) {
    printf("[ก่อน fork] shared_looking_value = %d ที่ address %p\n",
           shared_looking_value, (void *)&shared_looking_value);
    fflush(stdout); /* กัน buffer ซ้ำก่อน fork (ดูหัวข้อ Common Pitfalls) */

    pid_t pid = fork();

    if (pid < 0) {
        perror("fork ล้มเหลว");
        return 1;
    }

    if (pid == 0) {
        /* Child: แก้ไขค่าตัวแปร -> ตอนนี้ kernel จะ "copy" หน้าหน่วยความจำจริงๆ
           (Copy-on-Write ทำงาน ณ จุดนี้) ให้ child ก่อน แล้วค่อยแก้ */
        shared_looking_value = 999;
        printf("[CHILD ] shared_looking_value = %d ที่ address %p (คนละหน้า RAM จริงแล้ว)\n",
               shared_looking_value, (void *)&shared_looking_value);
    } else {
        wait(NULL);
        printf("[PARENT] shared_looking_value = %d ที่ address %p (ไม่ถูกกระทบเลย)\n",
               shared_looking_value, (void *)&shared_looking_value);
    }

    return 0;
}
```

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 -D_POSIX_C_SOURCE=200809L fork_cow.c -o fork_cow
./fork_cow
```

```
[ก่อน fork] shared_looking_value = 100 ที่ address 0x556ee90ef010
[CHILD ] shared_looking_value = 999 ที่ address 0x556ee90ef010 (คนละหน้า RAM จริงแล้ว)
[PARENT] shared_looking_value = 100 ที่ address 0x556ee90ef010 (ไม่ถูกกระทบเลย)
```

นี่คือหลักฐานที่ชัดเจนที่สุด: **Virtual Address เท่ากันเป๊ะ** (`0x556ee90ef010`) ทั้งใน
Parent และ Child (เพราะ Child เป็นสำเนาของ Parent จึงมี Layout ของ Virtual Address Space
เหมือนกันทุกประการ) แต่**ค่าที่อ่านได้ต่างกัน** (100 vs 999) เพราะเบื้องหลัง Physical Page
ที่ Address นี้ชี้ไปถูก Kernel แยกออกจากกันแล้วตอนที่ Child เขียนทับค่า — ตรงตามหลัก COW
ที่อธิบายไว้ข้างต้นทุกประการ

---

## 27.3 ตระกูล `exec()`: เปลี่ยนร่างเป็นโปรแกรมใหม่ทั้งหมด (Step 211)

`fork()` สร้าง Process ใหม่ที่เป็น**สำเนาของโปรแกรมเดิม** แต่ในทางปฏิบัติเรามักต้องการให้
Child รัน**โปรแกรมอื่น**ไปเลย (เช่น Shell ที่รับคำสั่ง `ls` แล้วต้องไปรันโปรแกรม `/bin/ls`
จริงๆ ไม่ใช่รันโค้ดของ Shell เอง) นี่คือหน้าที่ของตระกูลฟังก์ชัน **`exec()`**

`exec()` ทำสิ่งที่ตรงข้ามกับ `fork()` โดยสิ้นเชิง: แทนที่จะ**สร้าง** Process ใหม่ มันจะ
**แทนที่ (Replace)** Image ทั้งหมดของ Process ปัจจุบัน (Text, Data, BSS, Heap, Stack —
ทุก Segment ที่เรียนใน Part 26) ด้วยโปรแกรมใหม่ทั้งหมด **โดยที่ PID เดิมยังคงเดิม** (ไม่มี
การสร้าง Process ใหม่เลย)

```
ก่อน exec()                              หลัง exec("/bin/ls") สำเร็จ
┌───────────────────┐                   ┌───────────────────┐
│  Process (PID=200) │                   │  Process (PID=200) │  <- PID เดิม!
│  Text: exec_demo   │    execl(...)     │  Text: ls          │  <- โค้ดถูกแทนที่
│  Data: ตัวแปรเดิม   │  ─────────────►   │  Data: ตัวแปรของ ls │  <- ข้อมูลเดิมหายหมด
│  Stack: เดิม        │                   │  Stack: ใหม่หมด     │
└───────────────────┘                   └───────────────────┘
```

### `execl()` vs `execvp()`: ต่างกันตรงไหน

ตระกูล `exec()` มีหลายตัวแปร (`execl`, `execle`, `execlp`, `execv`, `execvp`, `execve`)
แต่ทั้งหมดเป็น Wrapper ของ System Call เดียวกันคือ `execve()` ความต่างกันอยู่ที่**วิธีรับ
Argument** เท่านั้น จำง่ายๆ จากตัวอักษรท้ายชื่อ:

| ตัวอักษร | ความหมาย |
|---|---|
| `l` (list) | รับ Argument แบบแยกทีละตัว จบด้วย `(char *)NULL` |
| `v` (vector) | รับ Argument เป็น Array ของ `char *` (ปิดท้ายด้วย `NULL`) |
| `p` (path) | ค้นหาโปรแกรมจาก Environment Variable `$PATH` ให้อัตโนมัติ (ไม่ต้องระบุ path เต็ม) |
| `e` (environment) | รับ Environment Variable ชุดใหม่แยกต่างหาก แทนที่จะสืบทอดจาก Process เดิม |

```c
/* ============================================================
 * ชื่อไฟล์:     exec_demo.c
 * คำอธิบาย:     สาธิต execl() และ execvp() — แทนที่ image ของ process ปัจจุบัน
 *              ด้วยโปรแกรมใหม่ทั้งหมด
 * ============================================================ */
#define _POSIX_C_SOURCE 200809L
#include <stdio.h>
#include <unistd.h>

int main(int argc, char *argv[]) {
    printf("[ก่อน exec] PID=%ld กำลังจะกลายร่างเป็นโปรแกรมอื่น...\n", (long)getpid());
    fflush(stdout); /* สำคัญมาก! ดู Common Pitfalls ข้อ 2 */

    if (argc > 1 && argv[1][0] == 'v') {
        /* execvp: รับ argument เป็น array (v = vector), หา path ให้เองจาก $PATH (p = path) */
        char *args[] = {"ls", "-l", "/tmp", NULL};
        execvp("ls", args);
    } else {
        /* execl: รับ argument แบบ list ทีละตัว จบด้วย NULL, ต้องระบุ path เต็ม */
        execl("/bin/ls", "ls", "-l", "/tmp", (char *)NULL);
    }

    /* บรรทัดนี้ (และหลังจากนี้) จะ "ไม่มีวันถูกรัน" ถ้า exec() สำเร็จ
       เพราะ image ของ process ถูกแทนที่ไปแล้วทั้งหมด รวมถึง instruction pointer
       จะมาถึงบรรทัดนี้ได้ก็ต่อเมื่อ exec() ล้มเหลวเท่านั้น */
    perror("exec ล้มเหลว");
    return 1;
}
```

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 -D_POSIX_C_SOURCE=200809L exec_demo.c -o exec_demo
./exec_demo
```

```
[ก่อน exec] PID=3720 กำลังจะกลายร่างเป็นโปรแกรมอื่น...
total 2172
drwxr-xr-x 3 root root    4096 Mar 31 13:31 147.0.7727.24
-rw------- 1 root root   27486 Sep 26 05:51 claude-append-system-prompt.txt
... (รายการไฟล์ใน /tmp ต่อ)
```

สังเกตว่าผลลัพธ์คือรายการไฟล์จาก `ls -l /tmp` **จริงๆ** ไม่ใช่ข้อความอะไรจาก `exec_demo.c`
เองเลยหลังจากบรรทัด "[ก่อน exec]" — เพราะ Process นี้ได้ "กลายร่าง" เป็น `ls` ไปแล้วอย่าง
สมบูรณ์ และบรรทัด `perror("exec ล้มเหลว")` ก็ไม่เคยถูกรันเลย เพราะ `exec()` สำเร็จ

> **ทำไมต้อง `fflush(stdout)` ก่อน `exec()` เสมอ?** เพราะ `exec()` ไม่ได้แค่ "แทนที่โค้ด"
> แต่ล้าง **Memory Space ทั้งหมด** ของ Process เดิมทิ้งไปด้วย รวมถึง Buffer ภายในของ
> `printf()` ที่ยังไม่ได้ Flush! ถ้าลืม `fflush(stdout)` ข้อความที่ `printf()` ไว้ก่อนหน้า
> (ที่ยังค้างอยู่ใน Buffer เพราะยังไม่เจอ `\n` ที่กระตุ้นการ Flush หรือ stdout เป็นแบบ
> Fully-buffered) จะ**หายไปเลยโดยไม่มีการเตือน** — นี่คือ Pitfall จริงที่พบเจอตอนพัฒนา
> เนื้อหาบทนี้ (ดูรายละเอียดเพิ่มใน Common Pitfalls ข้อ 2)

---

## 27.4 รวมร่าง fork() + exec(): Pattern มาตรฐานของ Systems Programming (Step 212)

เมื่อรวม `fork()` กับ `exec()` เข้าด้วยกัน เราจะได้ Pattern ที่ **ทุก Shell บนโลกนี้ใช้**
ในการรันคำสั่ง: **`fork()` สร้าง Process ลูกก่อน แล้วให้ Child เรียก `exec()` กลายร่างเป็น
โปรแกรมที่ต้องการ ส่วน Parent รอด้วย `wait()`**

ทำไมต้องแยก 2 ขั้นตอนแบบนี้ ทำไมไม่มี System Call เดียวที่ "สร้าง Process ใหม่แล้วรัน
โปรแกรมอื่นเลย"? เหตุผลคือ**ความยืดหยุ่น**: การแยก `fork()` ออกจาก `exec()` ทำให้ Child มี
โอกาส "แก้ไขสภาพแวดล้อมของตัวเอง" ก่อนกลายร่าง เช่น เปลี่ยนทิศทาง Input/Output (จะเรียน
เรื่อง Redirection และ Pipe ใน Part 29), เปลี่ยน Working Directory, หรือปรับ Environment
Variable — สิ่งเหล่านี้ทำได้ง่ายเพราะยังอยู่ในโค้ดของเราเอง (ก่อน `exec()` จะแทนที่ทุกอย่าง)

```c
/* ============================================================
 * ชื่อไฟล์:     fork_exec_pattern.c
 * คำอธิบาย:     รูปแบบมาตรฐานที่สุดของ Systems Programming: fork() แยก process ใหม่
 *              แล้วให้ child เรียก exec() รันโปรแกรมอื่น ส่วน parent รอด้วย wait()
 *              (นี่คือกลไกเบื้องหลังของทุก shell รวมถึง Part 27.8 ที่จะสร้าง mini shell)
 * ============================================================ */
#define _POSIX_C_SOURCE 200809L
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/wait.h>

int main(void) {
    printf("[PARENT %ld] จะสั่งรันคำสั่ง 'echo Hello from child' ผ่าน fork+exec\n",
           (long)getpid());
    fflush(stdout);

    pid_t pid = fork();

    if (pid < 0) {
        perror("fork ล้มเหลว");
        exit(EXIT_FAILURE);
    }

    if (pid == 0) {
        /* --- Child: เปลี่ยนร่างเป็น 'echo' --- */
        execlp("echo", "echo", "Hello from child", (char *)NULL);
        /* มาถึงตรงนี้ได้แปลว่า exec ล้มเหลว */
        perror("execlp ล้มเหลว");
        exit(EXIT_FAILURE);
    }

    /* --- Parent: รอ child ทำงานจนเสร็จ --- */
    int status;
    pid_t finished = waitpid(pid, &status, 0);

    if (finished == pid && WIFEXITED(status)) {
        printf("[PARENT] child PID=%ld จบแล้ว ด้วย exit code = %d\n",
               (long)finished, WEXITSTATUS(status));
    }

    return 0;
}
```

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 -D_POSIX_C_SOURCE=200809L fork_exec_pattern.c -o fork_exec_pattern
./fork_exec_pattern
```

```
[PARENT 3798] จะสั่งรันคำสั่ง 'echo Hello from child' ผ่าน fork+exec
Hello from child
[PARENT] child PID=3799 จบแล้ว ด้วย exit code = 0
```

Pattern นี้สำคัญมากจนควรจดจำให้ขึ้นใจ เพราะเราจะใช้มันซ้ำแล้วซ้ำเล่าตลอดหลักสูตรที่เหลือ
(รวมถึงตอนสร้าง Mini Shell ในหัวข้อ 27.8): **`fork()` → เช็ค return value → ถ้าเป็น Child
ให้ `exec()` → ถ้าเป็น Parent ให้ `wait()`/`waitpid()`**

---

## 27.5 `wait()` / `waitpid()` และการอ่าน Exit Status (Step 213)

เมื่อ Child Process จบการทำงาน (ไม่ว่าจะด้วย `return`, `exit()`, หรือถูก Signal ฆ่า) Kernel
จะเก็บ **Exit Status** ของมันไว้ชั่วคราว รอให้ Parent มา "รับ" ค่านี้ด้วย `wait()` หรือ
`waitpid()` — ถ้า Parent ไม่มารับ Child จะกลายเป็น **Zombie** (หัวข้อ 27.6)

| ฟังก์ชัน | พฤติกรรม |
|---|---|
| `wait(&status)` | รอ Child **ตัวใดก็ได้**ตัวแรกที่จบ (ไม่เลือก) เหมาะเมื่อมี Child เดียว |
| `waitpid(pid, &status, options)` | รอ Child ที่ระบุ `pid` ชัดเจน (หรือ `-1` = ตัวใดก็ได้เหมือน `wait()`) ควบคุมพฤติกรรมได้ด้วย `options` เช่น `WNOHANG` (ไม่บล็อกรอ ถ้ายังไม่มี child จบ ก็ return ทันที) |

ค่า `status` ที่ได้กลับมาเป็นตัวเลขที่**เข้ารหัสข้อมูลหลายอย่างปนกัน** (ทั้งว่าจบแบบปกติหรือ
ถูกฆ่า, และค่า Exit Code หรือหมายเลข Signal) ห้ามอ่านค่านี้ตรงๆ แต่ต้องผ่าน Macro มาตรฐาน
ที่ประกาศใน `<sys/wait.h>` เท่านั้น:

| Macro | ใช้ตรวจสอบ |
|---|---|
| `WIFEXITED(status)` | จริงถ้า Child จบแบบปกติ (เรียก `exit()`/`return` จาก `main`) |
| `WEXITSTATUS(status)` | ค่า Exit Code (ใช้ได้ก็ต่อเมื่อ `WIFEXITED` เป็นจริงเท่านั้น) |
| `WIFSIGNALED(status)` | จริงถ้า Child ถูก Signal ฆ่าตาย (เช่น Segmentation Fault, `kill -9`) |
| `WTERMSIG(status)` | หมายเลข Signal ที่ฆ่า Child (ใช้ได้ก็ต่อเมื่อ `WIFSIGNALED` เป็นจริง) |

```c
/* ============================================================
 * ชื่อไฟล์:     wait_status.c
 * คำอธิบาย:     สาธิตการอ่านค่า exit status ของ child ด้วย macro มาตรฐาน
 *              WIFEXITED / WEXITSTATUS / WIFSIGNALED / WTERMSIG
 * ============================================================ */
#define _POSIX_C_SOURCE 200809L
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <signal.h>
#include <string.h>
#include <sys/wait.h>

static void run_child_and_report(int exit_code, int send_signal) {
    pid_t pid = fork();

    if (pid < 0) {
        perror("fork ล้มเหลว");
        return;
    }

    if (pid == 0) {
        if (send_signal != 0) {
            raise(send_signal); /* จำลอง process ถูกฆ่าด้วย signal */
        }
        _exit(exit_code); /* ใช้ _exit() ไม่ใช่ exit() ในลูก ดู Common Pitfalls */
    }

    int status;
    waitpid(pid, &status, 0);

    if (WIFEXITED(status)) {
        printf("child PID=%ld จบแบบปกติ ด้วย exit code = %d\n",
               (long)pid, WEXITSTATUS(status));
    } else if (WIFSIGNALED(status)) {
        printf("child PID=%ld ถูกฆ่าด้วย signal หมายเลข %d (%s)\n",
               (long)pid, WTERMSIG(status), strsignal(WTERMSIG(status)));
    }
}

int main(void) {
    run_child_and_report(0, 0);       /* exit(0) ปกติ */
    run_child_and_report(42, 0);      /* exit(42) */
    run_child_and_report(0, SIGKILL); /* ถูกฆ่าด้วย SIGKILL */
    return 0;
}
```

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 -D_POSIX_C_SOURCE=200809L wait_status.c -o wait_status
./wait_status
```

```
child PID=3995 จบแบบปกติ ด้วย exit code = 0
child PID=3996 จบแบบปกติ ด้วย exit code = 42
child PID=3997 ถูกฆ่าด้วย signal หมายเลข 9 (Killed)
```

โน้ตสำคัญ: `exit(42)` ทำให้ `WEXITSTATUS(status)` อ่านได้ `42` พอดี — นี่คือกลไกเดียวกับที่
เราเรียนใน Part 1 เรื่อง `echo $?` เพียงแต่ครั้งนี้ **Parent Process ของเราเอง**เป็นผู้อ่าน
ค่านี้โดยตรงผ่าน `waitpid()` แทนที่จะเป็น Shell

---

## 27.6 Zombie Process: ผีที่ยังไม่ถูกฝัง (Step 214)

**Zombie Process** คือ Child Process ที่**จบการทำงานไปแล้ว** (โค้ดหยุดรันแล้วจริงๆ) แต่
Kernel ยัง**เก็บ Entry ของมันไว้ในตาราง Process** เพราะ Exit Status ของมันยังไม่ถูก Parent
มา "รับ" ด้วย `wait()`/`waitpid()` — ในทางเทคนิค Zombie **ไม่ได้ใช้ CPU หรือ RAM ของโปรแกรม
เดิมแล้ว** (Memory ทุก Segment ถูกคืนกลับไปหมดแล้ว) มันเหลือแค่ Entry เล็กๆ ในตาราง Process
ของ Kernel (PID, PPID, Exit Status) รอให้ใครมาอ่าน

```c
/* ============================================================
 * ชื่อไฟล์:     zombie_demo.c
 * คำอธิบาย:     สาธิตการเกิด Zombie Process — child จบการทำงานแล้ว แต่ parent
 *              ไม่ยอมเรียก wait()/waitpid() มารับ exit status ทำให้ entry ของ
 *              child ยังค้างอยู่ในตาราง process ของ kernel (สถานะ Z)
 * ============================================================ */
#define _POSIX_C_SOURCE 200809L
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

int main(void) {
    pid_t pid = fork();

    if (pid < 0) {
        perror("fork ล้มเหลว");
        exit(EXIT_FAILURE);
    }

    if (pid == 0) {
        /* Child จบการทำงานทันที */
        printf("[CHILD %ld] จบการทำงานแล้ว\n", (long)getpid());
        _exit(0);
    }

    /* Parent "ไม่" เรียก wait() เลย -- แกล้งยุ่งอยู่ 5 วินาที
       ระหว่างนี้ child ที่จบไปแล้วจะกลายเป็น Zombie (สถานะ Z) เพราะ kernel ยังเก็บ
       exit status ของมันไว้รอให้ parent มา "รับศพ" (wait) แต่ parent ไม่มารับสักที */
    printf("[PARENT %ld] child PID=%ld คือ Zombie แล้วตอนนี้ ลองรัน "
           "'ps -o pid,ppid,stat,cmd -p %ld' ในอีก terminal ภายใน 5 วินาที\n",
           (long)getpid(), (long)pid, (long)pid);
    sleep(5);

    printf("[PARENT] ตอนนี้ parent จบการทำงานแล้ว -- init/reaper จะเก็บกวาด Zombie ให้เอง\n");
    return 0;
}
```

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 -D_POSIX_C_SOURCE=200809L zombie_demo.c -o zombie_demo
./zombie_demo &
sleep 1
ps -o pid,ppid,stat,cmd -p <PID_ของ_child>
```

```
  PID  PPID STAT CMD
 4038  4036 Z    [zombie_demo] <defunct>
```

สังเกตคอลัมน์ `STAT` เป็น **`Z`** (Zombie) และ `CMD` แสดง **`<defunct>`** ต่อท้ายชื่อโปรแกรม
— นี่คือสัญญาณที่ `ps` ใช้บอกว่า Process นี้ตายแล้วแต่ยังไม่ถูก "เก็บศพ" (คำว่า Zombie
มาจากภาพลักษณ์นี้เอง: ร่างกายตายแล้วแต่ยังเดินได้ในสายตาของระบบ)

### วิธีป้องกัน Zombie

1. **เรียก `wait()`/`waitpid()` เสมอ** หลัง `fork()` ทุกครั้งที่รู้ว่าจะมี Child เกิดขึ้น
2. **ใช้ `SA_NOCLDWAIT`** ใน `sigaction()` เพื่อบอก Kernel ว่า "ไม่สนใจ Zombie เลย เก็บกวาด
   ให้อัตโนมัติ" (เหมาะกับโปรแกรมที่ไม่สนใจ Exit Status ของ Child เลยจริงๆ)
3. **ใช้ SIGCHLD Handler ร่วมกับ `waitpid(WNOHANG)`** เพื่อเก็บกวาด Child ที่จบแล้วโดย
   อัตโนมัติในเบื้องหลัง โดยไม่ต้องบล็อกรอ — วิธีนี้เป็นวิธีที่ Production-grade มากที่สุด
   และจะเรียนละเอียดพร้อมเขียนโค้ดจริงใน **Part 28.8**

> Zombie ไม่ได้อันตรายทันที (เพราะไม่กิน CPU/RAM มาก) แต่ถ้าโปรแกรม `fork()` บ่อยๆ โดยไม่
> `wait()` เลย (เช่น Server ที่รับ Connection ใหม่ตลอดเวลาแล้ว `fork()` ทุกครั้ง) Zombie
> จะสะสมจนเต็มตาราง Process ของ Kernel ทำให้ `fork()` ครั้งใหม่ล้มเหลว (เพราะชนขีดจำกัด
> จำนวน Process สูงสุดของระบบ) — สุดท้ายทั้งระบบอาจสร้าง Process ใหม่ไม่ได้เลย

---

## 27.7 Orphan Process: ลูกที่พ่อตายก่อน (Step 215)

**Orphan Process** คือสถานการณ์ตรงข้ามกับ Zombie: Child ยังทำงานอยู่ แต่ **Parent ตายไป
ก่อน** เมื่อเกิดเหตุการณ์นี้ Kernel จะ **"Reparent"** (ย้ายสังกัด) Child ตัวนั้นให้ไปเป็นลูก
ของ Process พิเศษที่เรียกว่า **`init`** (PID 1 เสมอบนระบบ Linux แบบดั้งเดิม) หรือ **"Subreaper"**
(Process ที่ตั้งค่าพิเศษไว้ให้รับ Orphan แทน `init` เช่นในบาง Container Runtime) โดย
อัตโนมัติทันที ไม่มี Orphan ตัวไหนถูก "ลอย" อยู่โดยไม่มีพ่อเลย

```c
/* ============================================================
 * ชื่อไฟล์:     orphan_demo.c
 * คำอธิบาย:     สาธิตการเกิด Orphan Process — parent จบการทำงานก่อน child
 *              ทำให้ child ถูก "รับเลี้ยง" (reparent) โดย init/reaper process
 *              (PID 1 หรือ subreaper) โดยอัตโนมัติ
 * ============================================================ */
#define _POSIX_C_SOURCE 200809L
#include <stdio.h>
#include <unistd.h>

int main(void) {
    pid_t pid = fork();

    if (pid == 0) {
        /* Child: รอสักครู่ให้ parent ตายไปก่อน แล้วค่อยเช็ค PPID ของตัวเอง */
        printf("[CHILD  %ld] PPID ตอนเพิ่งเกิด = %ld\n", (long)getpid(), (long)getppid());
        sleep(2);
        printf("[CHILD  %ld] PPID หลัง parent ตายไปแล้ว = %ld (ถูก reparent แล้ว!)\n",
               (long)getpid(), (long)getppid());
    } else {
        /* Parent: จบการทำงานทันทีโดยไม่รอ child เลย */
        printf("[PARENT %ld] จะจบการทำงานทันที ปล่อยให้ child PID=%ld กลายเป็น orphan\n",
               (long)getpid(), (long)pid);
    }

    return 0;
}
```

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 -D_POSIX_C_SOURCE=200809L orphan_demo.c -o orphan_demo
./orphan_demo
```

```
[PARENT 4105] จะจบการทำงานทันที ปล่อยให้ child PID=4106 กลายเป็น orphan
[CHILD  4106] PPID ตอนเพิ่งเกิด = 4105
[CHILD  4106] PPID หลัง parent ตายไปแล้ว = 1 (ถูก reparent แล้ว!)
```

สังเกตว่า `PPID` ของ Child เปลี่ยนจาก `4105` (Parent ตัวจริง) เป็น **`1`** (init) ทันทีที่
Parent ตายไป — นี่คือกลไก Reparent ที่ทำงานอัตโนมัติโดย Kernel และ `init`/`reaper` ที่รับ
เลี้ยง Orphan จะทำหน้าที่ **`wait()`** ให้เองเมื่อ Orphan ตัวนั้นจบการทำงานในที่สุด (ป้องกัน
ไม่ให้ Orphan กลายเป็น Zombie ค้างตลอดไปโดยไม่มีใคร `wait()`) — Orphan Process **ไม่ใช่
ปัญหาที่ต้องแก้ไข** เป็นพฤติกรรมปกติและถูกออกแบบมาให้ปลอดภัยอยู่แล้ว ต่างจาก Zombie ที่
เป็นสัญญาณของบั๊กในโปรแกรม (ลืม `wait()`) ที่ควรแก้ไข

### Zombie vs Orphan สรุปเปรียบเทียบ

| ประเด็น | Zombie | Orphan |
|---|---|---|
| ใครตายไปแล้ว | Child | Parent |
| ใครยังทำงานอยู่ | Parent (แต่ไม่ยอม wait) | Child |
| เป็นปัญหาหรือไม่ | ใช่ ถ้าสะสมมาก (บั๊กที่ควรแก้) | ไม่ เป็นพฤติกรรมปกติที่ Kernel จัดการให้ |
| แก้ไข/ป้องกันอย่างไร | เรียก `wait()`/`waitpid()` หรือใช้ SIGCHLD handler | ไม่ต้องทำอะไร — Kernel reparent ให้อัตโนมัติ |

---

## 27.8 ตัวอย่างจริง: เขียน Mini Shell ของตัวเอง (Step 216)

ถึงเวลารวมทุกอย่างที่เรียนมาใน Part นี้เข้าด้วยกัน: **Mini Shell** ที่อ่านคำสั่งจากผู้ใช้
แล้ว `fork()` + `exec()` รันคำสั่งนั้นจริง พร้อม `waitpid()` รอผลลัพธ์ และรองรับ Built-in
Command พื้นฐาน 2 ตัวคือ `cd` และ `exit` (ซึ่งต้องทำใน Process ของ Shell เอง **ห้าม**
`fork()` ไปทำ เพราะการเปลี่ยน Working Directory ใน Child จะไม่กระทบ Shell เลย — ลอง
พิจารณาว่าทำไมถึงเป็นเช่นนั้นจากความรู้เรื่อง Process แยกกันโดยสิ้นเชิงในหัวข้อ 27.1)

```c
/* ============================================================
 * ชื่อไฟล์:     mini_shell.c
 * คำอธิบาย:     Mini Shell อย่างง่าย — อ่านคำสั่งจากผู้ใช้ แล้ว fork() + exec()
 *              รันคำสั่งนั้น พร้อม waitpid() รอผลลัพธ์ รองรับ built-in "cd" และ "exit"
 * ============================================================ */

/* ---------- 1. Header Includes ---------- */
#define _POSIX_C_SOURCE 200809L
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/wait.h>

/* ---------- 2. Macro / Constant Definitions ---------- */
#define MAX_LINE_LEN 1024
#define MAX_ARGS 64

/* ---------- 3. Function Prototypes ---------- */
static void print_prompt(void);
static int read_line(char *buffer, size_t size);
static int parse_line(char *line, char *args[], int max_args);
static void execute_command(char *args[]);

/* ---------- 4. main() ---------- */
int main(void) {
    char line[MAX_LINE_LEN];
    char *args[MAX_ARGS];

    printf("=== Mini Shell (พิมพ์ 'exit' เพื่อออก) ===\n");

    while (1) {
        print_prompt();

        if (read_line(line, sizeof(line)) == 0) {
            /* เจอ EOF (เช่นกด Ctrl+D) ให้ออกจาก shell อย่างสุภาพ */
            printf("\n");
            break;
        }

        int argc = parse_line(line, args, MAX_ARGS);
        if (argc == 0) {
            continue; /* บรรทัดว่าง ไม่ต้องทำอะไร */
        }

        /* --- Built-in command: exit --- */
        if (strcmp(args[0], "exit") == 0) {
            printf("ออกจาก Mini Shell แล้ว\n");
            break;
        }

        /* --- Built-in command: cd (ต้องทำใน process ของ shell เอง
               จะ fork ไปทำใน child ไม่ได้ เพราะ cwd ของ child ไม่กระทบ shell) --- */
        if (strcmp(args[0], "cd") == 0) {
            const char *target = (argc >= 2) ? args[1] : getenv("HOME");
            if (target == NULL || chdir(target) != 0) {
                fprintf(stderr, "mini_shell: cd: ไม่สามารถเข้าไปที่ '%s' ได้\n",
                        target ? target : "(HOME ไม่ถูกตั้งค่า)");
            }
            continue;
        }

        /* --- คำสั่งภายนอก: fork + exec + wait --- */
        execute_command(args);
    }

    return 0;
}

/* ---------- 5. Function Implementations ---------- */
static void print_prompt(void) {
    char cwd[MAX_LINE_LEN];
    if (getcwd(cwd, sizeof(cwd)) != NULL) {
        printf("mini_shell:%s$ ", cwd);
    } else {
        printf("mini_shell$ ");
    }
    fflush(stdout);
}

static int read_line(char *buffer, size_t size) {
    if (fgets(buffer, (int)size, stdin) == NULL) {
        return 0; /* EOF หรือ error */
    }
    /* ตัด newline ท้ายบรรทัดทิ้ง */
    buffer[strcspn(buffer, "\n")] = '\0';
    return 1;
}

static int parse_line(char *line, char *args[], int max_args) {
    int argc = 0;
    char *token = strtok(line, " \t");

    while (token != NULL && argc < max_args - 1) {
        args[argc++] = token;
        token = strtok(NULL, " \t");
    }
    args[argc] = NULL; /* execvp ต้องการ array ที่ปิดท้ายด้วย NULL เสมอ */
    return argc;
}

static void execute_command(char *args[]) {
    pid_t pid = fork();

    if (pid < 0) {
        perror("mini_shell: fork ล้มเหลว");
        return;
    }

    if (pid == 0) {
        /* --- Child: กลายร่างเป็นโปรแกรมที่ผู้ใช้สั่ง --- */
        execvp(args[0], args);
        /* มาถึงตรงนี้ได้แปลว่าหาโปรแกรมไม่เจอ หรือ exec ล้มเหลว */
        fprintf(stderr, "mini_shell: ไม่พบคำสั่ง '%s'\n", args[0]);
        _exit(127); /* ธรรมเนียมของ shell: 127 = command not found */
    }

    /* --- Parent: รอ child ให้ทำงานจนเสร็จก่อนจะรับ prompt คำสั่งถัดไป --- */
    int status;
    waitpid(pid, &status, 0);
}
```

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 -D_POSIX_C_SOURCE=200809L mini_shell.c -o mini_shell
./mini_shell
```

ทดลองพิมพ์คำสั่งต่างๆ:

```
=== Mini Shell (พิมพ์ 'exit' เพื่อออก) ===
mini_shell:/home/user$ echo hello from mini shell
hello from mini shell
mini_shell:/home/user$ pwd
/home/user
mini_shell:/home/user$ cd /tmp
mini_shell:/tmp$ pwd
/tmp
mini_shell:/tmp$ ls -d /tmp
/tmp
mini_shell:/tmp$ nonexistentcmd123
mini_shell: ไม่พบคำสั่ง 'nonexistentcmd123'
mini_shell:/tmp$ exit
ออกจาก Mini Shell แล้ว
```

Shell ตัวนี้แม้จะเรียบง่ายมาก (ไม่รองรับ Pipe `|`, Redirection `>`, Background Job `&`,
Environment Variable Expansion, หรือ Quoting) แต่โครงสร้างหลักคือสิ่งเดียวกันเป๊ะกับที่
`bash`, `zsh`, หรือ Shell ระดับ Production ใช้: **อ่านบรรทัดคำสั่ง → แยก Argument →
ตรวจสอบ Built-in → ถ้าไม่ใช่ Built-in ให้ `fork()` + `exec()` + `waitpid()`** — เมื่อเรียน
เรื่อง Pipe ใน **Part 29** เราจะกลับมาต่อยอด Mini Shell ตัวนี้ให้รองรับ `cmd1 | cmd2` ได้ด้วย

---

## ข้อผิดพลาดที่พบบ่อย (Common Pitfalls)

1. **ลืม `fflush(stdout)` ก่อน `fork()` ทำให้ข้อความซ้ำ** — ถ้า stdout เป็นแบบ Fully-buffered
   (เช่นตอน Redirect ออกไฟล์หรือ Pipe) ข้อความที่ `printf()` ไว้แต่ยังไม่ถูก Flush จะอยู่ใน
   Buffer ที่ **ถูกคัดลอกไปให้ Child ด้วย** (เพราะ Buffer ก็เป็นส่วนหนึ่งของ Memory ที่
   `fork()` สำเนาไป) เมื่อทั้ง Parent และ Child ต่างก็ Flush Buffer เดียวกันตอนจบโปรแกรม
   ข้อความเดิมจะถูกพิมพ์ซ้ำสองครั้ง แก้ไขด้วยการ `fflush(stdout)` ทุกครั้งก่อนเรียก `fork()`

2. **ลืม `fflush(stdout)` ก่อน `exec()` ทำให้ข้อความหาย** — ตรงข้ามกับข้อ 1: `exec()` ล้าง
   Memory Space ทั้งหมดทิ้ง (ไม่ใช่แค่ล้างโค้ด) ข้อความที่ค้างอยู่ใน Buffer ของ `printf()`
   ที่ยังไม่ Flush จะ**หายไปเลยไม่มีวันถูกพิมพ์** เพราะ Buffer นั้นก็ถูกทำลายไปพร้อมกับ
   Process Image เดิม

3. **ใช้ `exit()` แทน `_exit()` ใน Child หลัง `fork()`** — `exit()` (ใน `<stdlib.h>`) จะ
   Flush stdio Buffer ทั้งหมดและเรียก Handler ที่ลงทะเบียนด้วย `atexit()` ก่อนออกโปรแกรม
   ถ้า Buffer ของ Parent ที่ยังไม่ Flush ถูกคัดลอกมาให้ Child ตอน `fork()` (ตามข้อ 1) แล้ว
   Child ดันไปเรียก `exit()` การ Flush นั้นจะทำให้ข้อความเดิมของ Parent ถูกพิมพ์ซ้ำผ่าน
   Child ด้วย! ใน Systems Programming จึงมีธรรมเนียมว่า **Child ควรใช้ `_exit()`** (System
   Call ตรงๆ ไม่ Flush อะไรเลย ไม่เรียก Handler ใดๆ) เพื่อออกจากโปรแกรมทันทีอย่างสะอาด

4. **ไม่ตรวจสอบค่า Return ของ `fork()` ว่าเป็น `-1` หรือไม่** — ถ้า `fork()` ล้มเหลว (เช่น
   ระบบมี Process มากเกินขีดจำกัด `RLIMIT_NPROC`) มันจะคืนค่า `-1` **ไม่ใช่ Process ใหม่เกิด
   ขึ้นเลย** โค้ดที่ไม่เช็คเงื่อนไขนี้อาจเข้าใจผิดว่า `pid == 0` (ทึกทักว่าเป็น Child) หรือ
   ใช้ `pid` (ที่จริงคือ `-1`) ไปเรียก `waitpid(-1, ...)` ซึ่งมีความหมายพิเศษ (รอ Child
   ตัวใดก็ได้ ไม่ใช่รอ Child ที่ไม่มีอยู่จริง) ทำให้พฤติกรรมผิดเพี้ยนไปโดยไม่มี Error ชัดเจน

5. **ลืม `wait()`/`waitpid()` ทำให้เกิด Zombie สะสม** — โดยเฉพาะโปรแกรมที่ `fork()` บ่อยๆ
   (เช่น Server ที่สร้าง Process ใหม่รับทุก Connection) ถ้าลืม `wait()` เพียงจุดเดียว
   Zombie จะสะสมไปเรื่อยๆ จนวันหนึ่งชนขีดจำกัดจำนวน Process ของระบบ ทำให้ `fork()` ครั้ง
   ใหม่ล้มเหลวไปด้วย (ดูหัวข้อ 27.6)

6. **เข้าใจผิดว่าโค้ดหลัง `exec()` ที่สำเร็จจะยังถูกรันต่อ** — เมื่อ `exec()` สำเร็จ Process
   Image ทั้งหมดถูกแทนที่แล้ว **ไม่มีทางย้อนกลับมารันโค้ดเดิมได้อีก** โค้ดที่เขียนไว้หลัง
   เรียก `exec()` (เช่น `perror()` ใน `exec_demo.c`) จะถูกรันก็ต่อเมื่อ **`exec()` ล้มเหลว
   เท่านั้น** (คืนค่า `-1`) — ถ้าเห็นข้อความ Error หลัง `exec()` แปลว่ามันล้มเหลวแน่นอน แต่
   ถ้าไม่เห็นอะไรเลยและโปรแกรมกลายเป็นโปรแกรมอื่นไปแล้ว นั่นคือ `exec()` ทำงานสำเร็จ

7. **ลืมว่า Built-in Command อย่าง `cd` ต้องทำใน Shell เอง ห้าม `fork()`** — ถ้าเผลอ
   `fork()` แล้วให้ Child เรียก `chdir()` การเปลี่ยน Directory จะเกิดขึ้นใน Child เท่านั้น
   (เพราะ Working Directory เป็นส่วนหนึ่งของ Process แต่ละตัว แยกจากกันตาม Memory Layout
   ที่เรียนใน Part 26) พอ Child จบการทำงาน การเปลี่ยนแปลงนั้นก็หายไปพร้อมกับ Child ทันที
   Shell (Parent) จะยังอยู่ที่ Directory เดิมเหมือนไม่มีอะไรเกิดขึ้น

---

## แบบฝึกหัดท้ายบท

1. ดัดแปลง `fork_basic.c` ให้ `fork()` สองครั้งซ้อนกัน สร้าง Process ต้นไม้ 3 ชั้น
   (Grandparent → Parent → Grandchild) แล้วพิมพ์ PID/PPID ของแต่ละชั้นให้ครบ

2. เขียนโปรแกรม `run_util.c` ที่รับคำสั่งและ Argument จาก `argv[1]` เป็นต้นไป แล้ว
   `fork()` + `execvp()` รันคำสั่งนั้น พร้อมพิมพ์ Exit Code สุดท้ายออกมา (ใช้งานแบบ
   `./run_util ls -l /tmp`)

3. ดัดแปลง `wait_status.c` ให้ทดสอบกรณี Child ถูกฆ่าด้วย `SIGSEGV` แทน `SIGKILL` (ใช้
   `raise(SIGSEGV)`) สังเกตข้อความที่ `strsignal()` แสดงว่าต่างจาก `SIGKILL` อย่างไร

4. เขียนโปรแกรมที่จงใจสร้าง Orphan Process (เหมือน `orphan_demo.c`) แล้วตรวจสอบว่าในเครื่อง
   ของคุณ Orphan ถูก Reparent ไปเป็นลูกของ PID อะไร (ควรเป็น `1` บนเครื่อง Linux ทั่วไป แต่
   อาจเป็นเลขอื่นถ้ารันอยู่ใน Container ที่มี Subreaper ของตัวเอง)

5. ดัดแปลง `mini_shell.c` ให้เพิ่ม Built-in Command ใหม่ชื่อ `status` ที่พิมพ์ Exit Code
   ของคำสั่งล่าสุดที่รันไป (คำใบ้: ต้องเก็บค่า `WEXITSTATUS()` ไว้ในตัวแปร Global หลังจาก
   `execute_command()` ทุกครั้ง)

6. อธิบายด้วยคำพูดของตัวเอง (พร้อมยกตัวอย่างโค้ดสั้นๆ ประกอบ) ว่าทำไมการไม่ตรวจสอบค่า
   Return ของ `fork()` ถึงเป็นอันตราย และยกตัวอย่างสถานการณ์ที่บั๊กนี้อาจทำให้โปรแกรม
   ทำงานผิดพลาดแบบที่ Debug ได้ยาก

### แนวทางเฉลยข้อ 1

```c
/* ============================================================
 * ชื่อไฟล์:     process_tree.c
 * คำอธิบาย:     เฉลยแบบฝึกหัด — fork() สองครั้งซ้อนกันเพื่อสร้าง Process 3 ชั้น
 *              (Grandparent -> Parent -> Grandchild) แล้วพิมพ์ PID/PPID ของแต่ละชั้น
 * ============================================================ */
#define _POSIX_C_SOURCE 200809L
#include <stdio.h>
#include <unistd.h>
#include <sys/wait.h>

int main(void) {
    printf("[GRANDPARENT] PID=%ld เริ่มต้น\n", (long)getpid());
    fflush(stdout);

    pid_t pid1 = fork();

    if (pid1 == 0) {
        /* --- ชั้นที่ 2: PARENT --- */
        printf("[PARENT]      PID=%ld, PPID=%ld (grandparent)\n",
               (long)getpid(), (long)getppid());
        fflush(stdout);

        pid_t pid2 = fork();

        if (pid2 == 0) {
            /* --- ชั้นที่ 3: GRANDCHILD --- */
            printf("[GRANDCHILD]  PID=%ld, PPID=%ld (parent)\n",
                   (long)getpid(), (long)getppid());
            fflush(stdout); /* สำคัญ! _exit() ไม่ flush ให้ ดู Common Pitfalls */
            _exit(0);
        }

        waitpid(pid2, NULL, 0);
        _exit(0);
    }

    waitpid(pid1, NULL, 0);
    printf("[GRANDPARENT] ทุก process ลูกจบการทำงานหมดแล้ว\n");
    return 0;
}
```

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 -D_POSIX_C_SOURCE=200809L process_tree.c -o process_tree
./process_tree
```

```
[GRANDPARENT] PID=25226 เริ่มต้น
[PARENT]      PID=25227, PPID=25226 (grandparent)
[GRANDCHILD]  PID=25228, PPID=25227 (parent)
[GRANDPARENT] ทุก process ลูกจบการทำงานหมดแล้ว
```

**คำอธิบาย**: จุดที่พลาดได้ง่ายที่สุดในเฉลยนี้ (และเป็นสิ่งที่เกิดขึ้นจริงระหว่างพัฒนา
เนื้อหาบทนี้) คือการลืม `fflush(stdout)` ก่อน `_exit(0)` ใน Grandchild — ถ้าลืม ข้อความ
`[GRANDCHILD]` จะ**หายไปทั้งบรรทัด** เพราะ `_exit()` ไม่ Flush Buffer ให้ (ตรงตาม Common
Pitfall ข้อ 3) นี่คือเหตุผลที่โค้ดข้างต้นเรียก `fflush(stdout)` ก่อน `_exit(0)` ทุกจุดที่
มีการพิมพ์ข้อความสำคัญไว้ก่อนหน้า — เป็นวินัยที่ต้องฝึกให้ติดเป็นนิสัยเวลาเขียนโปรแกรมที่
เกี่ยวกับ `fork()`

### แนวทางเฉลยข้อ 2

```c
/* ============================================================
 * ชื่อไฟล์:     run_util.c
 * คำอธิบาย:     เฉลยแบบฝึกหัด — โปรแกรม "run" อย่างง่าย รับคำสั่งจาก argv[1..]
 *              แล้ว fork+execvp+waitpid รันคำสั่งนั้น พร้อมพิมพ์ exit code สุดท้าย
 *              ใช้งาน: ./run_util ls -l /tmp
 * ============================================================ */
#define _POSIX_C_SOURCE 200809L
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/wait.h>

int main(int argc, char *argv[]) {
    if (argc < 2) {
        fprintf(stderr, "วิธีใช้: %s <คำสั่ง> [argument...]\n", argv[0]);
        return 1;
    }

    pid_t pid = fork();
    if (pid < 0) {
        perror("fork ล้มเหลว");
        return 1;
    }

    if (pid == 0) {
        /* argv[1..argc-1] คือคำสั่งและ argument ที่ผู้ใช้ต้องการรัน
           execvp ต้องการ array ที่ปิดท้ายด้วย NULL ซึ่ง argv ของ main() มีให้อยู่แล้ว
           (argv[argc] รับประกันว่าเป็น NULL เสมอตามมาตรฐาน C) */
        execvp(argv[1], &argv[1]);
        perror("execvp ล้มเหลว");
        _exit(127);
    }

    int status;
    waitpid(pid, &status, 0);

    if (WIFEXITED(status)) {
        printf("[run_util] คำสั่งจบด้วย exit code = %d\n", WEXITSTATUS(status));
        return WEXITSTATUS(status);
    } else if (WIFSIGNALED(status)) {
        printf("[run_util] คำสั่งถูกฆ่าด้วย signal = %d\n", WTERMSIG(status));
        return 128 + WTERMSIG(status);
    }

    return 0;
}
```

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 -D_POSIX_C_SOURCE=200809L run_util.c -o run_util
./run_util echo "hello from run_util"
./run_util false; echo "exit=$?"
./run_util nonexistent_cmd_xyz; echo "exit=$?"
```

```
hello from run_util
[run_util] คำสั่งจบด้วย exit code = 0
[run_util] คำสั่งจบด้วย exit code = 1
exit=1
execvp ล้มเหลว: No such file or directory
[run_util] คำสั่งจบด้วย exit code = 127
exit=127
```

**คำอธิบาย**: จุดที่น่าสนใจที่สุดในเฉลยนี้คือการใช้ `&argv[1]` เป็น Argument ตัวที่สองของ
`execvp()` โดยตรง แทนที่จะสร้าง Array ใหม่เอง — เพราะ `argv` ของ `main()` เป็น Array ของ
`char *` ที่ปิดท้ายด้วย `NULL` อยู่แล้วตามมาตรฐานภาษา C (`argv[argc] == NULL` เสมอ) การส่ง
`&argv[1]` จึงเท่ากับส่ง Array ย่อยที่ตัด `argv[0]` (ชื่อโปรแกรม `run_util` เอง) ออกไป และ
ยังคง `NULL` Terminator เดิมไว้ครบถ้วน ประหยัดทั้งโค้ดและหน่วยความจำเมื่อเทียบกับการสร้าง
Array ใหม่ นอกจากนี้ยังสังเกตว่าโปรแกรมนี้คืนค่า Exit Code เดียวกับที่คำสั่งภายในคืนมา
(`128 + signal` เป็นธรรมเนียมมาตรฐานของ Shell เมื่อ Process ถูก Signal ฆ่า) ทำให้ `run_util`
ทำงานโปร่งใสเหมือนเรียกคำสั่งนั้นตรงๆ ทุกประการ

---

## สรุปท้ายบท

ใน Part นี้เราได้:

- เข้าใจ **`fork()`** อย่างละเอียด ทั้งกลไก Return Value ที่ต่างกันระหว่าง Parent/Child
  และเทคนิค **Copy-on-Write** ที่ทำให้ `fork()` เร็วแม้ Process จะใช้หน่วยความจำมาก
- ใช้ตระกูล **`exec()`** (`execl`, `execvp`) เพื่อแทนที่ Process Image ด้วยโปรแกรมใหม่
  ทั้งหมด และเข้าใจว่าทำไมต้อง `fflush(stdout)` ก่อนเรียกเสมอ
- ประกอบ **fork() + exec() + wait()** เป็น Pattern มาตรฐานที่ใช้สร้าง Process ลูกเพื่อรัน
  โปรแกรมอื่น ซึ่งเป็นรากฐานของทุก Shell
- อ่านค่า **Exit Status** ของ Child ด้วย Macro มาตรฐาน (`WIFEXITED`, `WEXITSTATUS`,
  `WIFSIGNALED`, `WTERMSIG`) ได้อย่างถูกต้อง
- แยกแยะและป้องกัน **Zombie Process** (ลูกตายแต่พ่อไม่ยอมรับศพ) และเข้าใจ **Orphan
  Process** (พ่อตายก่อนลูก ถูก Reparent ไปหา `init` อัตโนมัติ) ได้อย่างชัดเจน
- สร้าง **Mini Shell** ของตัวเองที่ใช้งานได้จริง ผสมผสานทุกแนวคิดของ Part นี้เข้าด้วยกัน

Process ที่เราสร้างและควบคุมได้แล้วในวันนี้ ยังขาดความสามารถสำคัญอย่างหนึ่ง: **การรับมือกับ
เหตุการณ์ที่มาถึงแบบไม่คาดฝัน** เช่น ผู้ใช้กด Ctrl+C กลางทาง หรือ Process ลูกจบการทำงานโดย
ที่เราไม่ได้ตั้งใจรอ — นี่คือหน้าที่ของ **Signal** ซึ่งเป็นหัวข้อของ **Part 28** ที่จะสอน
วิธีดักจับและตอบสนองต่อเหตุการณ์เหล่านี้อย่างปลอดภัยและมืออาชีพ

**ต่อไป:** [Part 28 — Signal และ Signal Handling](./part-028-signals.md)
