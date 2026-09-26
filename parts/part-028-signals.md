# Part 28: Signal และ Signal Handling (Step 217–224)

> Module C — Systems Programming ด้วย C บน Linux | Part 28 จาก 125
> Step ที่ครอบคลุมใน Part นี้: Step 217–224
> Part ก่อนหน้า: [Part 27 — Process Management (fork/exec/wait)](./part-027-process-management.md) | Part ถัดไป: [Part 29 — Inter-Process Communication (Pipe/FIFO)](./part-029-ipc-pipes.md)

## เป้าหมายการเรียนรู้ (Learning Objectives)

หลังจบ Part นี้ ผู้เรียนจะสามารถ:

1. อธิบายได้ว่า **Signal** คืออะไรในฐานะ Asynchronous Notification และแตกต่างจากการเรียก
   ฟังก์ชันปกติหรือ Error Code อย่างไร
2. จำแนก Signal ที่พบบ่อยที่สุด (`SIGINT`, `SIGTERM`, `SIGKILL`, `SIGSEGV`, `SIGCHLD`,
   `SIGALRM`) พร้อมบอกพฤติกรรม Default และสถานการณ์ที่แต่ละตัวเกิดขึ้นจริง
3. เปรียบเทียบ **`signal()`** กับ **`sigaction()`** และอธิบายได้ว่าทำไมโค้ดระดับ Production
   ควรใช้ `sigaction()` เสมอ
4. เขียน **Signal Handler ที่ปลอดภัย** ตามหลัก Async-Signal-Safety โดยรู้ว่าฟังก์ชันใดใช้ได้
   และใช้ไม่ได้ใน Handler (เช่นทำไม `printf()`/`malloc()` ถึงอันตราย)
5. เขียนโปรแกรมที่จัดการ **Ctrl+C (`SIGINT`)** เพื่อ Cleanup ทรัพยากรอย่างปลอดภัยก่อนออก
   จากโปรแกรม (Graceful Shutdown)
6. ใช้ **`SIGCHLD`** ร่วมกับ `waitpid(WNOHANG)` เพื่อเก็บกวาด Zombie Process โดยอัตโนมัติ
   โดยไม่ต้องบล็อกรอ
7. ใช้ **`sigprocmask()`** เพื่อบล็อก Signal ชั่วคราวระหว่างช่วงวิกฤต (Critical Section)
   ที่ห้ามถูกขัดจังหวะ

---

## บทนำ: เมื่อโปรแกรมต้องรับมือกับสิ่งที่ไม่คาดฝัน

ตลอด Part 26–27 เราเขียนโปรแกรมที่ทำงานตามลำดับที่คาดเดาได้: บรรทัดต่อบรรทัด, `fork()`
แล้วรอผลด้วย `wait()` — ทุกอย่างเกิดขึ้น "ตามจังหวะที่โปรแกรมควบคุมเอง" แต่ในโลกจริง
โปรแกรมต้องรับมือกับเหตุการณ์ที่**มาถึงเมื่อไหร่ก็ได้ โดยไม่รู้ล่วงหน้า**:

- ผู้ใช้กด **Ctrl+C** กลางทางเพื่อขอให้โปรแกรมหยุด
- Process ลูกที่เรา `fork()` ไว้จบการทำงานโดยที่เรากำลังทำงานอย่างอื่นอยู่
- โปรแกรมพยายามเข้าถึงหน่วยความจำที่ไม่มีสิทธิ์ (บั๊ก Pointer ที่เรียนมาตั้งแต่ Part 8-9)
- ระบบต้องการปิดโปรแกรมอย่างสุภาพก่อน Shutdown เครื่อง

กลไกที่ Linux ใช้แจ้งเหตุการณ์เหล่านี้เข้ามาในโปรแกรมของเราเรียกว่า **Signal** — Part นี้
จะสอนวิธีดักจับและตอบสนองต่อ Signal อย่างถูกต้องและปลอดภัย

---

## 28.1 Signal คืออะไร: Asynchronous Notification (Step 217)

**Signal** คือกลไกที่ **Kernel** ใช้แจ้งเหตุการณ์บางอย่างไปยัง Process แบบ **Asynchronous**
(ไม่ประสานจังหวะ) — หมายความว่า Signal สามารถมาถึง Process ของเรา **ณ จุดใดก็ได้**ระหว่าง
การทำงาน โดยไม่สนใจว่าโปรแกรมกำลังรันอยู่บรรทัดไหน ต่างจากการเรียกฟังก์ชันปกติที่เรา
ควบคุมจังหวะได้เองทั้งหมด

```
การทำงานปกติของโปรแกรม (Synchronous)          Signal (Asynchronous)
─────────────────────────────────            ─────────────────────
บรรทัด 1                                       บรรทัด 1
บรรทัด 2                                       บรรทัด 2    ◄── Signal มาถึง "แทรก" ตรงนี้!
บรรทัด 3   (ทำตามลำดับที่เขียนไว้              (กระโดดไปรัน Handler ก่อน)
บรรทัด 4    เสมอ คาดเดาได้ 100%)                บรรทัด 3   (แล้วค่อยกลับมาทำงานต่อ)
                                               บรรทัด 4
```

Signal มาจากได้ 3 แหล่งหลัก:

1. **Kernel เอง** — เช่นตรวจพบว่าโปรแกรมเข้าถึงหน่วยความจำผิดกฎ (`SIGSEGV`), หารด้วยศูนย์
   (`SIGFPE`), หรือ Child Process จบการทำงาน (`SIGCHLD`)
2. **Process อื่น** — เรียกใช้ System Call `kill()` เพื่อส่ง Signal ไปยัง Process เป้าหมาย
   โดยตรง (เช่นคำสั่ง `kill -9 <PID>` ที่ใช้ `SIGKILL`)
3. **ตัว Process เอง** — เรียก `raise()` หรือตั้ง `alarm()` ให้ Kernel ส่ง Signal กลับมาหา
   ตัวเองเมื่อครบเวลาที่กำหนด

เมื่อ Process ได้รับ Signal มันมีทางเลือก 3 แบบ (ถ้าไม่ได้กำหนดเป็นอย่างอื่น จะใช้ **Default
Action** ของ Signal นั้นๆ):

| ทางเลือก | ความหมาย |
|---|---|
| **Default Action** | ปล่อยให้ Kernel จัดการตามพฤติกรรมมาตรฐานของ Signal นั้น (เช่นจบการทำงานทันที) |
| **Ignore** | เพิกเฉยต่อ Signal นั้นโดยสิ้นเชิง (ทำได้กับบาง Signal เท่านั้น) |
| **Custom Handler** | กำหนดฟังก์ชันของเราเองให้ Kernel เรียกทันทีที่ Signal มาถึง (หัวข้อ 28.3–28.4) |

---

## 28.2 Signal ที่พบบ่อยที่สุด (Step 218)

| Signal | เลข (ทั่วไปบน x86-64 Linux) | เกิดขึ้นเมื่อ | Default Action |
|---|---|---|---|
| `SIGINT` | 2 | ผู้ใช้กด **Ctrl+C** | จบการทำงาน (Terminate) |
| `SIGTERM` | 15 | คำขอ "ให้จบการทำงานอย่างสุภาพ" (`kill <PID>` แบบไม่ระบุ signal จะส่งตัวนี้) | จบการทำงาน (Terminate) — **ดักจับและ Cleanup ได้** |
| `SIGKILL` | 9 | ถูกบังคับฆ่าแบบไม่มีทางเลือก (`kill -9 <PID>`) | จบการทำงานทันที — **ดักจับ/เพิกเฉยไม่ได้เด็ดขาด** |
| `SIGSEGV` | 11 | เข้าถึงหน่วยความจำผิดกฎ (Dereference Wild Pointer, Buffer Overflow รุนแรง) | จบการทำงาน + สร้าง Core Dump |
| `SIGCHLD` | 17 | Child Process จบการทำงานหรือเปลี่ยนสถานะ | เพิกเฉย (Ignore) โดย Default |
| `SIGALRM` | 14 | เวลาที่ตั้งไว้ด้วย `alarm()` ครบกำหนด | จบการทำงาน |
| `SIGSTOP` | 19 | หยุด Process ชั่วคราว (เหมือน Ctrl+Z แต่ดักจับไม่ได้) | หยุดทำงาน (Stopped) |
| `SIGTSTP` | 20 | ผู้ใช้กด **Ctrl+Z** | หยุดทำงาน (Stopped) — ดักจับได้ |
| `SIGPIPE` | 13 | เขียนข้อมูลไปยัง Pipe ที่ไม่มีใครอ่านอยู่แล้ว (จะเรียนใน Part 29) | จบการทำงาน |

### จุดที่สำคัญที่สุด: `SIGKILL` และ `SIGSTOP` ดักจับไม่ได้

`SIGKILL` (9) และ `SIGSTOP` (19) ถูกออกแบบมาให้เป็น **"ปุ่มฉุกเฉิน"** ที่ Kernel รับประกันว่า
จะทำงานเสมอ ไม่มีทางที่โปรแกรมจะเขียนโค้ดมาดักจับ, เพิกเฉย, หรือ Block มันได้เลย — เหตุผล
คือระบบต้องการวิธี**ฆ่า Process ที่ค้างหรือมีปัญหาหนักได้แน่นอน 100%** แม้ Process นั้นจะ
เขียน Handler ดักจับ Signal ทุกตัวไว้หมดแล้วก็ตาม (ลองนึกภาพ Process ที่ติด Bug จนดักจับ
`SIGTERM` ไว้แล้วไม่ยอมจบสักที ถ้าไม่มี `SIGKILL` ที่ฆ่าได้แน่นอน ระบบจะไม่มีทางเอา Process
นั้นออกได้เลย)

```bash
kill -l          # แสดงรายชื่อ Signal ทั้งหมดที่ระบบรองรับ พร้อมหมายเลข
kill -TERM <PID> # ส่ง SIGTERM (ขอให้จบอย่างสุภาพ — เทียบเท่า kill <PID> เฉยๆ)
kill -9 <PID>    # ส่ง SIGKILL (บังคับฆ่าทันที ไม่มีทางเลือก)
```

---

## 28.3 `signal()` vs `sigaction()`: ทำไมต้องใช้ `sigaction()` (Step 219)

### `signal()`: API ดั้งเดิม เรียบง่าย แต่มีปัญหา

```c
void (*signal(int signum, void (*handler)(int)))(int);
```

Signature ของ `signal()` อ่านยากเพราะเป็น Function Pointer ซ้อนกัน แต่ใช้งานง่ายมาก:

```c
signal(SIGINT, my_handler);   /* ตั้งให้ my_handler ถูกเรียกเมื่อได้รับ SIGINT */
```

ปัญหาของ `signal()` คือ **พฤติกรรมไม่แน่นอนข้าม Unix แต่ละสาย** (Portability ต่ำ) — เช่น
บาง Unix ยุคเก่าจะรีเซ็ต Handler กลับเป็น Default ทันทีหลังถูกเรียกครั้งแรก (ต้องเรียก
`signal()` ซ้ำใน Handler ทุกครั้งถ้าต้องการดักจับต่อ) และไม่มีทางควบคุมรายละเอียดสำคัญ
เช่น "จะ Block Signal อื่นระหว่าง Handler ทำงานหรือไม่" หรือ "System Call ที่ค้างอยู่ควร
ถูกขัดจังหวะหรือทำงานต่ออัตโนมัติ"

```c
/* ============================================================
 * ชื่อไฟล์:     signal_basic.c
 * คำอธิบาย:     ดักจับ SIGINT (Ctrl+C) ด้วย signal() แบบดั้งเดิม
 * ============================================================ */
#define _POSIX_C_SOURCE 200809L
#include <stdio.h>
#include <signal.h>
#include <unistd.h>

void handle_sigint(int signum) {
    /* คำเตือน: เรียก printf() ใน handler แบบนี้ไม่ปลอดภัยตามหลัก async-signal-safety
       (ดูรายละเอียดในหัวข้อ 28.5) แต่ในตัวอย่างนี้ใช้เพื่อสาธิตพฤติกรรมพื้นฐานก่อน */
    printf("\n[handler] ได้รับ signal หมายเลข %d (SIGINT) แล้ว\n", signum);
}

int main(void) {
    signal(SIGINT, handle_sigint);

    printf("PID=%ld กำลังทำงาน กด Ctrl+C เพื่อทดสอบ (จะรอ 5 วินาที)\n", (long)getpid());
    for (int i = 0; i < 5; i++) {
        printf("ทำงาน... (%d)\n", i + 1);
        sleep(1);
    }
    printf("จบการทำงานตามปกติ\n");
    return 0;
}
```

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 -D_POSIX_C_SOURCE=200809L signal_basic.c -o signal_basic
./signal_basic
# (กด Ctrl+C ระหว่างที่โปรแกรมกำลังนับ 1..5)
```

```
PID=4339 กำลังทำงาน กด Ctrl+C เพื่อทดสอบ (จะรอ 5 วินาที)
ทำงาน... (1)

[handler] ได้รับ signal หมายเลข 2 (SIGINT) แล้ว
ทำงาน... (2)
ทำงาน... (3)
ทำงาน... (4)
ทำงาน... (5)
จบการทำงานตามปกติ
```

สังเกตว่าโปรแกรม**ไม่จบการทำงานทันที**เมื่อกด Ctrl+C — เพราะเราติดตั้ง Handler ของเราเอง
ไปแทนที่ Default Action (ซึ่งปกติคือจบการทำงานทันที) โปรแกรมแค่พิมพ์ข้อความแล้วทำงานต่อ
จากจุดเดิม (บน Linux/glibc สมัยใหม่ `signal()` มีพฤติกรรมใกล้เคียง `sigaction()` มากขึ้น
แล้ว แต่ในทางปฏิบัติของ Production Code ยังแนะนำให้ใช้ `sigaction()` เพื่อความชัดเจนและ
Portable ข้ามระบบ)

### `sigaction()`: API มาตรฐานที่ควบคุมได้ชัดเจนและ Portable

```c
int sigaction(int signum, const struct sigaction *act, struct sigaction *oldact);
```

`struct sigaction` ให้เราควบคุมรายละเอียดที่ `signal()` ทำไม่ได้:

```c
struct sigaction {
    void (*sa_handler)(int);      /* handler ธรรมดา (แบบเดียวกับ signal()) */
    void (*sa_sigaction)(int, siginfo_t *, void *); /* handler แบบละเอียด (ดูข้อมูลผู้ส่งได้) */
    sigset_t sa_mask;             /* signal อื่นที่จะถูกบล็อกระหว่าง handler ทำงาน */
    int sa_flags;                 /* ตัวเลือกพิเศษ เช่น SA_RESTART, SA_SIGINFO */
};
```

```c
/* ============================================================
 * ชื่อไฟล์:     sigaction_demo.c
 * คำอธิบาย:     ดักจับ SIGINT ด้วย sigaction() ซึ่งเป็น API ที่ทันสมัยและปลอดภัย
 *              กว่า signal() แบบดั้งเดิม (ระบุพฤติกรรมได้ชัดเจน ไม่ขึ้นกับแพลตฟอร์ม)
 * ============================================================ */
#define _POSIX_C_SOURCE 200809L
#include <stdio.h>
#include <signal.h>
#include <string.h>
#include <unistd.h>

static volatile sig_atomic_t g_signal_count = 0;

static void handle_sigint(int signum) {
    (void)signum; /* ไม่ได้ใช้พารามิเตอร์นี้ แต่ handler ต้องมี signature นี้เสมอ */
    g_signal_count++; /* ปลอดภัย: การเขียนตัวแปร sig_atomic_t เป็น atomic operation */
}

int main(void) {
    struct sigaction sa;

    /* sa_handler: ฟังก์ชันที่จะถูกเรียกเมื่อได้รับ signal */
    sa.sa_handler = handle_sigint;

    /* sa_mask: signal อื่นที่จะถูก "บล็อกชั่วคราว" ระหว่าง handler กำลังทำงาน
       ในที่นี้ใช้ sigemptyset คือไม่บล็อก signal อื่นเพิ่มเติมเลย */
    sigemptyset(&sa.sa_mask);

    /* sa_flags: SA_RESTART สั่งให้ system call ที่ค้างอยู่ (เช่น read()) กลับมาทำงาน
       ต่ออัตโนมัติหลัง handler จบ แทนที่จะ return error EINTR ให้เราต้องเช็คเอง */
    sa.sa_flags = SA_RESTART;

    if (sigaction(SIGINT, &sa, NULL) == -1) {
        perror("sigaction ล้มเหลว");
        return 1;
    }

    printf("PID=%ld กำลังทำงาน ลองส่ง SIGINT เข้ามาหลายครั้ง (จะรอ 5 วินาที)\n",
           (long)getpid());
    for (int i = 0; i < 5; i++) {
        sleep(1);
        printf("รอบที่ %d: นับ SIGINT ที่ได้รับแล้ว = %d ครั้ง\n", i + 1, g_signal_count);
    }

    return 0;
}
```

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 -D_POSIX_C_SOURCE=200809L sigaction_demo.c -o sigaction_demo
./sigaction_demo
# (กด Ctrl+C สามครั้งระหว่างโปรแกรมทำงาน)
```

```
PID=4471 กำลังทำงาน ลองส่ง SIGINT เข้ามาหลายครั้ง (จะรอ 5 วินาที)
รอบที่ 1: นับ SIGINT ที่ได้รับแล้ว = 1 ครั้ง
รอบที่ 2: นับ SIGINT ที่ได้รับแล้ว = 1 ครั้ง
รอบที่ 3: นับ SIGINT ที่ได้รับแล้ว = 2 ครั้ง
รอบที่ 4: นับ SIGINT ที่ได้รับแล้ว = 2 ครั้ง
รอบที่ 5: นับ SIGINT ที่ได้รับแล้ว = 3 ครั้ง
```

### ตารางเปรียบเทียบ

| ประเด็น | `signal()` | `sigaction()` |
|---|---|---|
| ความง่ายในการใช้งาน | ง่ายมาก บรรทัดเดียว | ต้องตั้งค่า `struct sigaction` ก่อน |
| Portability ข้ามระบบ | ต่ำ (พฤติกรรมต่างกันในแต่ละ Unix) | สูง (พฤติกรรมชัดเจนตามมาตรฐาน POSIX เสมอ) |
| ควบคุม Signal Mask ระหว่าง Handler | ทำไม่ได้ | ทำได้ผ่าน `sa_mask` |
| ควบคุมพฤติกรรม System Call ที่ค้าง | ทำไม่ได้ | ทำได้ผ่าน `SA_RESTART` |
| ดูข้อมูลผู้ส่ง Signal (PID, UID) | ทำไม่ได้ | ทำได้ผ่าน `SA_SIGINFO` + `sa_sigaction` |
| คำแนะนำของหลักสูตรนี้ | ใช้เพื่อทำความเข้าใจแนวคิดพื้นฐานเท่านั้น | **ใช้ในโค้ด Production เสมอ** |

---

## 28.4 เขียน Signal Handler ที่ปลอดภัย: Async-Signal-Safety (Step 220)

นี่คือหัวข้อที่**สำคัญที่สุด**ของทั้ง Part นี้ และเป็นจุดที่โปรแกรมเมอร์มือใหม่พลาดบ่อยที่สุด

เพราะ Signal Handler ถูกเรียก**แทรกกลางการทำงานปกติของโปรแกรม ณ จุดใดก็ได้** (ตามหัวข้อ
28.1) จึงมีความเสี่ยงที่ Handler จะถูกเรียกขึ้นมา**ในขณะที่โค้ดหลักกำลังทำงานอะไรบางอย่าง
ค้างอยู่พอดี** เช่น:

- ถ้าโค้ดหลักกำลังเรียก `printf()` อยู่ (ซึ่งภายในมีการ Lock และแก้ไข Buffer ภายใน) แล้ว
  Signal มาถึงพอดี ทำให้ Handler ถูกเรียกและไป `printf()` ซ้ำอีกที — **Buffer ภายในของ
  `printf()` อาจถูกแก้ไขซ้อนกันจนข้อมูล Corrupt หรือโปรแกรม Deadlock ได้ทันที**
- ถ้าโค้ดหลักกำลังเรียก `malloc()` อยู่ (ซึ่งภายในต้องแก้ไข Linked List ของ Free Memory
  Block) แล้ว Signal มาถึงพอดี และ Handler ก็ไปเรียก `malloc()`/`free()` ต่ออีก —
  โครงสร้างข้อมูลภายในของ `malloc()` จะถูกแก้ไขซ้อนกันจนพัง

ฟังก์ชันที่ **"ปลอดภัยที่จะเรียกจาก Signal Handler"** เรียกว่า **Async-Signal-Safe
Function** ซึ่ง POSIX ระบุรายชื่อไว้ชัดเจนใน Manual Page `signal-safety(7)`

### รายชื่อฟังก์ชันที่ปลอดภัย (Async-Signal-Safe) ที่ใช้บ่อย

| ปลอดภัย (ใช้ได้ใน Handler) | **ไม่ปลอดภัย** (ห้ามใช้ใน Handler) |
|---|---|
| `write()` | `printf()`, `fprintf()`, `scanf()` (ใช้ Buffer + Lock ภายใน) |
| `_exit()` | `exit()` (Flush Buffer, เรียก `atexit()` handler) |
| `read()` | `malloc()`, `free()`, `calloc()`, `realloc()` (แก้ไข Heap Metadata) |
| `kill()`, `raise()` | `strtok()` (ใช้ Static Buffer ภายใน ไม่ Thread/Signal-safe) |
| `signal()`, `sigaction()` | ฟังก์ชันส่วนใหญ่ใน `<stdio.h>` |
| `sigprocmask()` | `sleep()` บางระบบ (พฤติกรรมไม่แน่นอนเมื่อผสมกับ signal) |
| การอ่าน/เขียนตัวแปร `volatile sig_atomic_t` | Lock ใดๆ (Mutex) — เสี่ยง Deadlock ถ้า Handler แทรกตอน Lock ถูกถืออยู่ |

### แพทเทิร์นมาตรฐาน: ตั้ง Flag ใน Handler แล้วค่อยทำงานหนักใน Main Loop

หลักการที่ปลอดภัยที่สุดคือ **ทำให้ Handler สั้นที่สุดเท่าที่จะทำได้** — แค่ตั้งค่าตัวแปร
`volatile sig_atomic_t` (ชนิดข้อมูลที่มาตรฐาน C รับประกันว่าการอ่าน/เขียนเป็น Atomic
Operation ไม่ถูกขัดจังหวะกลางคัน) แล้วปล่อยให้โค้ดหลักใน `main()` เป็นคนตรวจสอบ Flag นี้
และทำงานที่ซับซ้อน (Cleanup, `printf()`, `malloc()`) เอาเองในจังหวะที่ปลอดภัย

```c
/* ============================================================
 * ชื่อไฟล์:     safe_handler.c
 * คำอธิบาย:     ตัวอย่าง Signal Handler ที่ "ปลอดภัย" ตามหลัก async-signal-safety
 *              - ใช้ write() (async-signal-safe) แทน printf() (ไม่ปลอดภัย)
 *              - ใช้ volatile sig_atomic_t เป็น flag แทนการทำงานหนักใน handler
 *              - งานที่ซับซ้อน (เช่น cleanup, malloc, printf) ย้ายไปทำใน main loop แทน
 * ============================================================ */
#define _POSIX_C_SOURCE 200809L
#include <stdio.h>
#include <stdlib.h>
#include <signal.h>
#include <string.h>
#include <unistd.h>

/* sig_atomic_t คือชนิดข้อมูลที่มาตรฐาน C รับประกันว่าการอ่าน/เขียนเป็น atomic
   (ไม่ถูกขัดจังหวะกลางคัน) จึงปลอดภัยที่จะใช้แชร์ระหว่าง handler กับโค้ดหลัก
   ต้องเป็น volatile ด้วย เพื่อบอก compiler ว่าห้าม optimize การอ่านค่าทิ้ง
   เพราะค่าอาจถูกเปลี่ยนจาก "ภายนอก" การไหลของโค้ดปกติ (จาก signal handler) */
static volatile sig_atomic_t g_stop_requested = 0;

static void handle_sigint(int signum) {
    (void)signum;

    /* ปลอดภัย: write() อยู่ในรายการ async-signal-safe functions ตาม POSIX
       (ต่างจาก printf() ที่ใช้ internal buffer/lock ซึ่งอาจค้างหรือ corrupt ได้
       ถ้า signal มาขัดจังหวะตอน printf() ตัวอื่นกำลังทำงานอยู่พอดี) */
    const char msg[] = "\n[handler] ได้รับ SIGINT แล้ว จะขอออกอย่างปลอดภัย...\n";
    write(STDOUT_FILENO, msg, sizeof(msg) - 1);

    /* แค่ตั้ง flag เท่านั้น ไม่ทำงานหนักใน handler เลย */
    g_stop_requested = 1;
}

int main(void) {
    struct sigaction sa;
    sa.sa_handler = handle_sigint;
    sigemptyset(&sa.sa_mask);
    sa.sa_flags = 0;
    sigaction(SIGINT, &sa, NULL);

    printf("PID=%ld เริ่มทำงาน กด Ctrl+C เพื่อขอออกอย่างปลอดภัย\n", (long)getpid());

    int round = 0;
    while (!g_stop_requested) {
        printf("กำลังทำงาน... รอบที่ %d\n", ++round);
        sleep(1);
    }

    /* งานที่ "ไม่" async-signal-safe เช่น printf(), malloc()/free() ทำได้อย่างปลอดภัย
       ที่นี่ เพราะเรากลับมาอยู่ใน main flow ปกติแล้ว ไม่ได้อยู่ใน handler */
    printf("[main] ตรวจพบ g_stop_requested=1 กำลังทำ cleanup งานค้าง...\n");
    printf("[main] cleanup เสร็จสมบูรณ์ ออกโปรแกรมด้วย exit code 0\n");

    return 0;
}
```

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 -D_POSIX_C_SOURCE=200809L safe_handler.c -o safe_handler
./safe_handler
# (กด Ctrl+C ระหว่างทำงาน)
```

```
PID=4558 เริ่มทำงาน กด Ctrl+C เพื่อขอออกอย่างปลอดภัย
กำลังทำงาน... รอบที่ 1
กำลังทำงาน... รอบที่ 2

[handler] ได้รับ SIGINT แล้ว จะขอออกอย่างปลอดภัย...
[main] ตรวจพบ g_stop_requested=1 กำลังทำ cleanup งานค้าง...
[main] cleanup เสร็จสมบูรณ์ ออกโปรแกรมด้วย exit code 0
```

สังเกตว่า Handler ทำแค่ 2 อย่าง: เรียก `write()` (ปลอดภัย) และตั้งค่า `g_stop_requested = 1`
(ปลอดภัย) — งานที่ "หนัก" กว่านั้นทั้งหมด (การพิมพ์ข้อความ Cleanup ผ่าน `printf()`) ถูกย้าย
ไปทำใน `main()` หลังจาก Handler ทำงานเสร็จและกลับมาสู่ Flow ปกติแล้วเท่านั้น — นี่คือหลักการ
ที่ต้องยึดถือเสมอเมื่อเขียน Signal Handler ในโค้ด Production

---

## 28.5 การจัดการ Ctrl+C ให้ Cleanup ก่อนออกโปรแกรม (Step 221)

มาดูตัวอย่างที่ใกล้เคียงสถานการณ์จริงมากขึ้น: โปรแกรมที่เปิด "ทรัพยากร" บางอย่างไว้ระหว่าง
ทำงาน (เช่น ไฟล์ Lock ที่บอกว่าโปรแกรมกำลังทำงานอยู่ — รูปแบบที่ใช้จริงในโปรแกรม Server
หลายตัว) แล้วเมื่อผู้ใช้กด Ctrl+C ต้องมั่นใจว่าทรัพยากรนั้นถูกปิดและลบทิ้งอย่างถูกต้องก่อน
ออกจากโปรแกรมเสมอ ไม่ปล่อยให้ค้างอยู่

```c
/* ============================================================
 * ชื่อไฟล์:     ctrlc_cleanup.c
 * คำอธิบาย:     ตัวอย่างจริง — โปรแกรมเปิดไฟล์ล็อกไว้ระหว่างทำงาน เมื่อผู้ใช้กด
 *              Ctrl+C ต้องปิดไฟล์และลบไฟล์ล็อกทิ้งก่อนออกเสมอ (graceful shutdown)
 *              ไม่ปล่อยให้ resource ค้าง เหมือนโปรแกรม server จริงที่ต้อง cleanup
 *              connection/lock file ก่อนออกทุกครั้ง
 * ============================================================ */
#define _POSIX_C_SOURCE 200809L
#include <stdio.h>
#include <stdlib.h>
#include <signal.h>
#include <unistd.h>

#define LOCK_FILE "/tmp/ctrlc_cleanup_demo.lock"

static volatile sig_atomic_t g_stop_requested = 0;

static void handle_sigint(int signum) {
    (void)signum;
    g_stop_requested = 1;
}

int main(void) {
    struct sigaction sa;
    sa.sa_handler = handle_sigint;
    sigemptyset(&sa.sa_mask);
    sa.sa_flags = 0;
    sigaction(SIGINT, &sa, NULL);
    sigaction(SIGTERM, &sa, NULL); /* รองรับทั้ง Ctrl+C และ `kill` แบบปกติ */

    /* จำลอง resource ที่ต้อง cleanup: ไฟล์ล็อกที่บอกว่าโปรแกรมนี้กำลังทำงานอยู่ */
    FILE *lock_fp = fopen(LOCK_FILE, "w");
    if (lock_fp == NULL) {
        perror("สร้างไฟล์ล็อกไม่สำเร็จ");
        return 1;
    }
    fprintf(lock_fp, "%ld\n", (long)getpid());
    fclose(lock_fp);
    printf("[main] สร้างไฟล์ล็อกที่ %s แล้ว (PID=%ld)\n", LOCK_FILE, (long)getpid());

    int round = 0;
    while (!g_stop_requested) {
        printf("[main] กำลังทำงาน... รอบที่ %d\n", ++round);
        sleep(1);
    }

    printf("[main] ได้รับสัญญาณให้หยุด กำลัง cleanup...\n");
    if (remove(LOCK_FILE) == 0) {
        printf("[main] ลบไฟล์ล็อกเรียบร้อยแล้ว ออกโปรแกรมอย่างปลอดภัย\n");
    } else {
        perror("[main] ลบไฟล์ล็อกไม่สำเร็จ");
    }

    return 0;
}
```

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 -D_POSIX_C_SOURCE=200809L ctrlc_cleanup.c -o ctrlc_cleanup
./ctrlc_cleanup
# (สังเกตว่ามีไฟล์ /tmp/ctrlc_cleanup_demo.lock ถูกสร้างขึ้นระหว่างทำงาน)
# (กด Ctrl+C)
```

```
[main] สร้างไฟล์ล็อกที่ /tmp/ctrlc_cleanup_demo.lock แล้ว (PID=4636)
[main] กำลังทำงาน... รอบที่ 1
[main] กำลังทำงาน... รอบที่ 2
[main] ได้รับสัญญาณให้หยุด กำลัง cleanup...
[main] ลบไฟล์ล็อกเรียบร้อยแล้ว ออกโปรแกรมอย่างปลอดภัย
```

ตรวจสอบได้จริงว่าไฟล์ Lock ถูกสร้างขึ้นระหว่างโปรแกรมทำงาน และถูกลบทิ้งหลัง Cleanup:

```bash
# ระหว่างโปรแกรมกำลังรัน (ก่อนกด Ctrl+C)
ls -la /tmp/ctrlc_cleanup_demo.lock
# -rw-r--r-- 1 root root 5 ... /tmp/ctrlc_cleanup_demo.lock

# หลังกด Ctrl+C และโปรแกรมจบแล้ว
ls -la /tmp/ctrlc_cleanup_demo.lock
# ls: cannot access '/tmp/ctrlc_cleanup_demo.lock': No such file or directory
```

นี่คือรูปแบบที่โปรแกรมระดับ Production ทุกตัวต้องมี — ไม่ว่าจะเป็น Web Server ที่ต้องปิด
Socket Connection ให้เรียบร้อย, Database ที่ต้อง Flush ข้อมูลค้างลง Disk ก่อนปิด, หรือ
โปรแกรมที่ต้องลบไฟล์ชั่วคราวทิ้ง — การดักจับทั้ง `SIGINT` (Ctrl+C จาก Terminal) และ
`SIGTERM` (คำสั่ง `kill` ปกติ หรือสัญญาณจาก Container Orchestrator เช่น Docker/Kubernetes
ตอนสั่งปิด Container) ด้วย Handler เดียวกัน ทำให้โปรแกรม Cleanup อย่างถูกต้องไม่ว่าจะถูก
สั่งปิดจากช่องทางไหนก็ตาม

---

## 28.6 ตั้ง Timeout ด้วย `alarm()` และ `SIGALRM` (Step 222)

`alarm(seconds)` เป็นฟังก์ชันที่ขอให้ Kernel ส่ง `SIGALRM` กลับมาหา Process ตัวเองเมื่อผ่าน
ไปตามเวลาที่กำหนด — มีประโยชน์มากในการใส่ **Timeout** ให้กับ Operation ที่อาจค้างรอไม่มี
กำหนด เช่นรอ Input จากผู้ใช้ หรือรอการเชื่อมต่อ Network (จะเจอบ่อยขึ้นใน Part 33-34)

```c
/* ============================================================
 * ชื่อไฟล์:     alarm_timeout.c
 * คำอธิบาย:     ใช้ alarm() + SIGALRM ตั้ง timeout ให้กับการรอ input จากผู้ใช้
 *              (fgets ที่ค้างรอไม่จำกัดเวลาโดยปกติ)
 * ============================================================ */
#define _POSIX_C_SOURCE 200809L
#include <stdio.h>
#include <string.h>
#include <signal.h>
#include <unistd.h>

static volatile sig_atomic_t g_timed_out = 0;

static void handle_sigalrm(int signum) {
    (void)signum;
    g_timed_out = 1;
    const char msg[] = "\n[SIGALRM] หมดเวลาแล้ว!\n";
    write(STDOUT_FILENO, msg, sizeof(msg) - 1);
}

int main(void) {
    struct sigaction sa;
    sa.sa_handler = handle_sigalrm;
    sigemptyset(&sa.sa_mask);
    /* ไม่ใส่ SA_RESTART เพราะเราต้องการให้ fgets() ที่ค้างอยู่ถูกขัดจังหวะ
       และ return NULL กลับมาทันทีเมื่อ SIGALRM มาถึง ไม่ใช่ทำงานต่ออัตโนมัติ */
    sa.sa_flags = 0;
    sigaction(SIGALRM, &sa, NULL);

    printf("คุณมีเวลา 3 วินาทีในการพิมพ์อะไรก็ได้แล้วกด Enter: ");
    fflush(stdout);

    alarm(3); /* ตั้งเวลา: ถ้าผ่านไป 3 วินาที kernel จะส่ง SIGALRM มาให้เราเอง */

    char buffer[128];
    char *result = fgets(buffer, sizeof(buffer), stdin);

    alarm(0); /* ยกเลิก alarm ที่ตั้งไว้ (ถ้า fgets สำเร็จก่อนหมดเวลา) */

    if (result != NULL && !g_timed_out) {
        buffer[strcspn(buffer, "\n")] = '\0';
        printf("คุณพิมพ์ว่า: \"%s\"\n", buffer);
    } else {
        printf("ไม่ได้รับ input ทันเวลา ใช้ค่า default แทน\n");
    }

    return 0;
}
```

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 -D_POSIX_C_SOURCE=200809L alarm_timeout.c -o alarm_timeout
./alarm_timeout
# กรณีที่ 1: ไม่พิมพ์อะไรเลยภายใน 3 วินาที
```

```
คุณมีเวลา 3 วินาทีในการพิมพ์อะไรก็ได้แล้วกด Enter: 
[SIGALRM] หมดเวลาแล้ว!
ไม่ได้รับ input ทันเวลา ใช้ค่า default แทน
```

```bash
./alarm_timeout
# กรณีที่ 2: พิมพ์ทันเวลา
```

```
คุณมีเวลา 3 วินาทีในการพิมพ์อะไรก็ได้แล้วกด Enter: hello
คุณพิมพ์ว่า: "hello"
```

จุดสำคัญคือ `fgets()` เป็น System Call ที่ **Blocking** (ค้างรอโดยไม่มีกำหนดถ้าไม่มีข้อมูล
เข้ามา) แต่เมื่อ `SIGALRM` มาถึงระหว่างที่ `fgets()` กำลังค้างรออยู่พอดี ตัว `fgets()` จะ
**ถูกขัดจังหวะและคืนค่า `NULL` ทันที** (เพราะเราไม่ได้ใส่ `SA_RESTART` ไว้) ทำให้โปรแกรม
สามารถตรวจจับ Timeout และทำงานต่อได้โดยไม่ต้องค้างรอตลอดไป

---

## 28.7 บล็อก Signal ชั่วคราวด้วย `sigprocmask()` (Step 223)

บางครั้งโปรแกรมมี **ช่วงวิกฤต (Critical Section)** ที่ไม่ต้องการให้ถูก Signal มาขัดจังหวะ
เด็ดขาด เช่นกำลังเขียนข้อมูลสำคัญลงไฟล์อยู่ครึ่งทาง ถ้าถูก `SIGINT` แทรกและ Handler ไป
แก้ไขข้อมูลเดียวกันพร้อมกัน อาจทำให้ไฟล์เสียหายได้ วิธีป้องกันคือ **บล็อก (Block)** Signal
นั้นไว้ชั่วคราวด้วย `sigprocmask()` — Signal ที่ถูกบล็อกจะไม่หายไปไหน แต่ Kernel จะ **"พัก"**
ไว้ก่อน (เรียกว่า Pending) แล้วค่อยส่งเข้ามาจริงทันทีที่ถูกปลดบล็อก

```c
/* ============================================================
 * ชื่อไฟล์:     mask_demo.c
 * คำอธิบาย:     ใช้ sigprocmask() บล็อก SIGINT ชั่วคราวระหว่าง "ช่วงวิกฤต"
 *              (Critical Section) ที่ห้ามถูกขัดจังหวะ เช่นระหว่างกำลังเขียนไฟล์
 *              สำคัญอยู่ แล้วค่อยปลดบล็อกหลังจบช่วงวิกฤต
 * ============================================================ */
#define _POSIX_C_SOURCE 200809L
#include <stdio.h>
#include <signal.h>
#include <unistd.h>

static volatile sig_atomic_t g_got_sigint = 0;

static void handle_sigint(int signum) {
    (void)signum;
    g_got_sigint = 1;
}

int main(void) {
    struct sigaction sa;
    sa.sa_handler = handle_sigint;
    sigemptyset(&sa.sa_mask);
    sa.sa_flags = 0;
    sigaction(SIGINT, &sa, NULL);

    sigset_t block_set;
    sigemptyset(&block_set);
    sigaddset(&block_set, SIGINT);

    printf("[main] เข้าสู่ช่วงวิกฤต (SIGINT ถูกบล็อกไว้ชั่วคราว) ลองส่ง SIGINT ตอนนี้ได้\n");
    sigprocmask(SIG_BLOCK, &block_set, NULL); /* บล็อก SIGINT ตั้งแต่จุดนี้ */

    for (int i = 0; i < 3; i++) {
        printf("[main] กำลังทำงานสำคัญ... ขั้นตอนที่ %d (SIGINT ที่ส่งมาจะถูกพักไว้ก่อน)\n",
               i + 1);
        sleep(1);
    }

    printf("[main] จบช่วงวิกฤตแล้ว g_got_sigint ก่อนปลดบล็อก = %d\n", g_got_sigint);
    sigprocmask(SIG_UNBLOCK, &block_set, NULL); /* ปลดบล็อก -> handler จะถูกเรียกทันทีถ้ามี signal ค้างอยู่ */
    printf("[main] ปลดบล็อกแล้ว g_got_sigint หลังปลดบล็อก = %d\n", g_got_sigint);

    return 0;
}
```

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 -D_POSIX_C_SOURCE=200809L mask_demo.c -o mask_demo
./mask_demo
# (กด Ctrl+C ระหว่างที่โปรแกรมกำลังพิมพ์ "กำลังทำงานสำคัญ")
```

```
[main] เข้าสู่ช่วงวิกฤต (SIGINT ถูกบล็อกไว้ชั่วคราว) ลองส่ง SIGINT ตอนนี้ได้
[main] กำลังทำงานสำคัญ... ขั้นตอนที่ 1 (SIGINT ที่ส่งมาจะถูกพักไว้ก่อน)
[main] กำลังทำงานสำคัญ... ขั้นตอนที่ 2 (SIGINT ที่ส่งมาจะถูกพักไว้ก่อน)
[main] กำลังทำงานสำคัญ... ขั้นตอนที่ 3 (SIGINT ที่ส่งมาจะถูกพักไว้ก่อน)
[main] จบช่วงวิกฤตแล้ว g_got_sigint ก่อนปลดบล็อก = 0
[main] ปลดบล็อกแล้ว g_got_sigint หลังปลดบล็อก = 1
```

สังเกตว่าแม้จะกด Ctrl+C ระหว่างที่โปรแกรมกำลังพิมพ์ขั้นตอนที่ 1-3 อยู่ ค่า `g_got_sigint`
ก็ยังเป็น `0` ทันทีหลังจบ Loop — เพราะ Handler **ยังไม่ถูกเรียกเลย** ในระหว่างนั้น (Signal
ถูกพักไว้ที่ Kernel) จนกระทั่งเรียก `sigprocmask(SIG_UNBLOCK, ...)` เท่านั้นที่ Handler
ถูกเรียกทันที ทำให้ค่าเปลี่ยนเป็น `1` — นี่คือเครื่องมือสำคัญเวลาต้องปกป้องช่วงเวลาที่
ข้อมูลอยู่ในสถานะไม่สมบูรณ์ (Inconsistent State) จากการถูก Signal แทรกกลางทาง

---

## 28.8 `SIGCHLD` + `waitpid()`: ป้องกัน Zombie แบบอัตโนมัติ (Step 224)

กลับมาที่ปัญหา **Zombie Process** จาก Part 27.6 — วิธีป้องกันที่ Production-grade ที่สุด
คือดักจับ **`SIGCHLD`** (Signal ที่ Kernel ส่งให้ Parent อัตโนมัติทุกครั้งที่ Child จบการ
ทำงาน) แล้วให้ Handler เรียก `waitpid()` แบบ `WNOHANG` (ไม่บล็อกรอ) เพื่อเก็บกวาด Child
ที่จบไปแล้วทันที โดยที่โค้ดหลักไม่ต้องหยุดรอ `wait()` ตรงๆ เลยแม้แต่น้อย

```c
/* ============================================================
 * ชื่อไฟล์:     sigchld_reaper.c
 * คำอธิบาย:     ใช้ SIGCHLD handler ร่วมกับ waitpid(WNOHANG) เพื่อ "เก็บกวาด"
 *              child ที่จบการทำงานโดยอัตโนมัติ ป้องกันไม่ให้เกิด Zombie Process
 *              เลย แม้จะสร้าง child หลายตัวและไม่ได้เรียก wait() ตรงๆ ใน main
 * ============================================================ */
#define _POSIX_C_SOURCE 200809L
#include <stdio.h>
#include <stdlib.h>
#include <signal.h>
#include <errno.h>
#include <unistd.h>
#include <sys/wait.h>

static volatile sig_atomic_t g_reaped_count = 0;

static void handle_sigchld(int signum) {
    (void)signum;

    /* errno เป็น global (จริงๆ คือ thread-local) ที่ signal handler อาจไปเปลี่ยนค่า
       โดยไม่ตั้งใจ ถ้า handler เรียกฟังก์ชันที่ set errno (เช่น waitpid) ขณะที่ main
       flow กำลังเช็ค errno ของ system call ตัวอื่นค้างอยู่พอดี ต้องเซฟ/คืนค่าเสมอ */
    int saved_errno = errno;

    /* วนเรียก waitpid() แบบ WNOHANG (ไม่บล็อกรอ) จนกว่าจะไม่มี child ที่จบแล้ว
       ค้างอยู่เลย จำเป็นต้องวนเป็น loop เพราะถ้า child หลายตัวจบพร้อมกัน อาจมี
       SIGCHLD มาถึงเราแค่ครั้งเดียว (signal ธรรมดาไม่ถูก "นับสะสม" หรือ queue ไว้) */
    pid_t pid;
    while ((pid = waitpid(-1, NULL, WNOHANG)) > 0) {
        g_reaped_count++;
    }

    errno = saved_errno;
}

int main(void) {
    struct sigaction sa;
    sa.sa_handler = handle_sigchld;
    sigemptyset(&sa.sa_mask);
    sa.sa_flags = SA_RESTART; /* กัน system call ที่ค้าง (เช่น sleep) ไม่ให้ถูกขัดจังหวะ */
    sigaction(SIGCHLD, &sa, NULL);

    printf("[main] PID=%ld กำลังสร้าง child 5 ตัว แต่ละตัวจบไม่พร้อมกัน\n", (long)getpid());

    for (int i = 0; i < 5; i++) {
        pid_t pid = fork();
        if (pid < 0) {
            perror("fork ล้มเหลว");
            continue;
        }
        if (pid == 0) {
            /* child แต่ละตัวหลับไม่เท่ากัน แล้วจบไปเงียบๆ */
            sleep((unsigned int)(i % 3) + 1);
            _exit(0);
        }
    }

    printf("[main] ทำงานอื่นต่อได้ตามปกติ โดยไม่ต้องรอ child เลยสักตัว\n");
    for (int i = 0; i < 6; i++) {
        printf("[main] ทำงานอยู่... วินาทีที่ %d, เก็บกวาด child ไปแล้ว %d ตัว\n",
               i + 1, g_reaped_count);
        sleep(1);
    }

    printf("[main] จบการทำงาน: เก็บกวาด child ไปทั้งหมด %d ตัว (ควรเท่ากับ 5)\n",
           g_reaped_count);
    return 0;
}
```

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 -D_POSIX_C_SOURCE=200809L sigchld_reaper.c -o sigchld_reaper
./sigchld_reaper
```

```
[main] PID=4816 กำลังสร้าง child 5 ตัว แต่ละตัวจบไม่พร้อมกัน
[main] ทำงานอื่นต่อได้ตามปกติ โดยไม่ต้องรอ child เลยสักตัว
[main] ทำงานอยู่... วินาทีที่ 1, เก็บกวาด child ไปแล้ว 0 ตัว
[main] ทำงานอยู่... วินาทีที่ 2, เก็บกวาด child ไปแล้ว 2 ตัว
[main] ทำงานอยู่... วินาทีที่ 3, เก็บกวาด child ไปแล้ว 3 ตัว
[main] ทำงานอยู่... วินาทีที่ 4, เก็บกวาด child ไปแล้ว 4 ตัว
[main] ทำงานอยู่... วินาทีที่ 5, เก็บกวาด child ไปแล้ว 5 ตัว
[main] ทำงานอยู่... วินาทีที่ 6, เก็บกวาด child ไปแล้ว 5 ตัว
[main] จบการทำงาน: เก็บกวาด child ไปทั้งหมด 5 ตัว (ควรเท่ากับ 5)
```

ตรวจสอบด้วย `ps aux | grep sigchld_reaper` ระหว่างโปรแกรมกำลังทำงานจะพบว่า**ไม่มี Process
ไหนอยู่ในสถานะ `Z` (Zombie) เลย** แม้แต่ตัวเดียว ต่างจาก `zombie_demo.c` ใน Part 27 ที่ปล่อย
ให้ Zombie ค้างอยู่จนกว่า Parent จะจบการทำงาน

### ทำไมต้องวนเป็น `while` loop แทนที่จะเรียก `waitpid()` แค่ครั้งเดียว

จุดที่พลาดง่ายที่สุดคือคิดว่า Handler ถูกเรียก 1 ครั้งต่อ Child ที่จบ 1 ตัวเสมอ — แต่ความ
จริงคือ **Signal ธรรมดา (Standard Signal) ไม่ถูก "นับสะสม" หรือเข้าคิว (Queue)** ถ้า Child
หลายตัวจบพร้อมกันในช่วงเวลาสั้นๆ ก่อนที่ Handler จะมีโอกาสได้ทำงาน อาจมี `SIGCHLD` เข้ามา
รวมกันเหลือแค่ครั้งเดียว การวน `while ((pid = waitpid(-1, NULL, WNOHANG)) > 0)` จนกว่าจะ
ไม่เหลือ Child ที่จบแล้วค้างอยู่เลย จึงเป็นวิธีเดียวที่รับประกันว่าจะไม่มี Zombie หลงเหลือ
แม้ Child หลายตัวจะจบพร้อมกันพอดี

---

## ข้อผิดพลาดที่พบบ่อย (Common Pitfalls)

1. **เรียกฟังก์ชันที่ไม่ Async-Signal-Safe ใน Handler** — โดยเฉพาะ `printf()` และ
   `malloc()`/`free()` เพราะทั้งสองใช้ Buffer/Heap Metadata ภายในที่ไม่ปลอดภัยถ้าถูก
   Signal แทรกกลางทาง (ดูหัวข้อ 28.4) แม้ในทางปฏิบัติจะ "ดูเหมือนทำงานได้" บ่อยครั้งเพราะ
   โอกาสที่ Signal จะมาถึงตรงจังหวะที่พังพอดีมีน้อย แต่นี่คือ **Undefined Behavior** ที่จะ
   แสดงอาการเป็นบั๊กที่สุ่มเกิดขึ้นนานๆ ครั้ง (Heisenbug) และ Debug ได้ยากมาก

2. **ใช้ `exit()` แทน `_exit()` ใน Signal Handler** — เช่นเดียวกับใน Child หลัง `fork()`
   (Part 27) `exit()` จะ Flush Buffer และเรียก `atexit()` Handler ซึ่งไม่ Async-Signal-Safe
   ถ้าจำเป็นต้องออกโปรแกรมทันทีจาก Handler ให้ใช้ `_exit()` เสมอ

3. **ลืมว่า `SIGKILL` และ `SIGSTOP` ดักจับหรือ Block ไม่ได้เด็ดขาด** — เขียน `sigaction()`
   ดักจับ `SIGKILL` แล้วสงสัยว่าทำไม Handler ไม่เคยถูกเรียกเลย เพราะ Kernel ปฏิเสธการตั้ง
   Handler สำหรับ 2 Signal นี้ตั้งแต่แรก (`sigaction()` จะคืนค่า Error ถ้าพยายามทำ) — ถ้า
   ต้องการให้โปรแกรม Cleanup ก่อนตาย ต้องใช้ `SIGTERM` ไม่ใช่ `SIGKILL`

4. **ลืมใช้ `volatile sig_atomic_t`** สำหรับตัวแปรที่ใช้แชร์ระหว่าง Handler กับโค้ดหลัก —
   ถ้าใช้ `int` ธรรมดาโดยไม่ใส่ `volatile` Compiler อาจ Optimize โค้ดในลูปที่เช็คค่าตัวแปร
   นั้น (เช่น `while (!g_stop_requested)`) ให้อ่านค่าแค่ครั้งเดียวแล้วเก็บไว้ใน Register
   (เพราะมองไม่เห็นว่ามีอะไรมาเปลี่ยนค่าจาก "ภายนอก" การไหลของโค้ดปกติ) ทำให้ Loop ไม่มีวัน
   จบแม้ Handler จะตั้งค่าไปแล้วก็ตาม — นี่คือ Pitfall ที่ทำให้ Debug งงมากเพราะโค้ด
   "ดูถูกต้อง" ทุกอย่างแต่ทำงานผิดเมื่อเปิด Optimization (`-O2` ขึ้นไป)

5. **ไม่วน `waitpid(WNOHANG)` เป็น loop ใน `SIGCHLD` Handler** — ถ้าเรียก `waitpid()` แค่
   ครั้งเดียวต่อการเรียก Handler 1 ครั้ง จะพลาด Case ที่ Child หลายตัวจบพร้อมกันแต่ Signal
   ถูกรวมเหลือครั้งเดียว (ตามที่อธิบายในหัวข้อ 28.8) ทำให้ยังมี Zombie หลงเหลืออยู่บางส่วน

6. **ไม่เซฟ/คืนค่า `errno` ใน Handler** — ถ้า Handler เรียกฟังก์ชันที่อาจเปลี่ยนค่า `errno`
   (เช่น `waitpid()`) แล้วดันไปแทรกกลาง System Call อื่นที่โค้ดหลักกำลังจะเช็ค `errno`
   อยู่พอดี ค่า `errno` ที่โค้ดหลักอ่านได้หลัง Handler จบอาจไม่ใช่ค่าที่มาจาก System Call
   ที่โค้ดหลักเรียกจริง แต่เป็นค่าที่ Handler ไปเปลี่ยนทับไว้ ต้อง `int saved = errno;`
   ก่อน แล้ว `errno = saved;` หลังทำงานใน Handler เสร็จเสมอ

7. **ลืมว่า `SA_RESTART` มีทั้งข้อดีและข้อเสีย** — การใส่ `SA_RESTART` ทำให้ System Call ที่
   ค้างอยู่ (เช่น `read()`) กลับมาทำงานต่ออัตโนมัติหลัง Handler จบ สะดวกสำหรับหลายกรณี แต่
   ถ้าต้องการให้ System Call ที่ค้างอยู่ถูก "ขัดจังหวะจริง" (เช่นใน `alarm_timeout.c` ที่
   ต้องการให้ `fgets()` คืนค่า `NULL` ทันทีเมื่อ Timeout) การใส่ `SA_RESTART` จะทำให้
   พฤติกรรมที่ต้องการไม่เกิดขึ้น ต้องเลือกใช้ Flag นี้ตามจุดประสงค์ให้ถูกต้อง

---

## แบบฝึกหัดท้ายบท

1. ดัดแปลง `signal_basic.c` ให้ดักจับทั้ง `SIGINT` และ `SIGTERM` ด้วย Handler เดียวกัน แล้ว
   ทดสอบด้วยการกด Ctrl+C และใช้คำสั่ง `kill <PID>` (ไม่ใส่ `-9`) จากอีก Terminal

2. อธิบายด้วยคำพูดของตัวเอง (ไม่ต้องรันจริงเพราะ Reproduce ยาก) ว่าทำไมการเรียก `malloc()`
   ใน Signal Handler ขณะที่โค้ดหลักกำลังเรียก `malloc()` อยู่พอดี อาจทำให้โปรแกรม Deadlock
   หรือ Heap เสียหาย

3. ดัดแปลง `safe_handler.c` ให้ดักจับทั้ง `SIGINT` และ `SIGTERM` ด้วย `sigaction()` เดียวกัน
   (คำใบ้: เรียก `sigaction()` สองครั้งด้วย `struct sigaction` ตัวเดียวกัน แต่คนละ
   Signal Number)

4. เขียนโปรแกรม `alarm_timeout.c` เวอร์ชันของตัวเอง ที่เปลี่ยนจากรอ Input ผู้ใช้ เป็นการ
   จำกัดเวลาทำงานของ Busy Loop (เช่นคำนวณอะไรบางอย่างที่อาจใช้เวลานาน) ให้หยุดอัตโนมัติ
   ถ้าเกิน 3 วินาที

5. ดัดแปลง `sigchld_reaper.c` ให้พิมพ์ **PID และ Exit Code** ของ Child แต่ละตัวที่ถูกเก็บ
   กวาด แทนที่จะนับจำนวนเฉยๆ (คำใบ้: ต้องใช้ `write()` ในการพิมพ์ ไม่ใช่ `printf()` เพราะ
   ยังอยู่ใน Handler — ใช้ `snprintf()` แปลงตัวเลขเป็น String ก่อน เพราะ `snprintf()` เข้า
   ข่าย Async-Signal-Safe ตาม POSIX)

6. เขียนโปรแกรมที่ใช้ `sigaction()` พร้อม `SA_SIGINFO` เพื่อรับรู้ **PID ของ Process ที่ส่ง
   `SIGINT`** มาหาเรา (คำใบ้: ใช้ `sa_sigaction` แทน `sa_handler` และอ่านค่าจาก
   `siginfo_t->si_pid`) ทดสอบด้วยการเปิด 2 Terminal แล้วใช้ `kill -INT <PID>` ส่งจากอีก
   Terminal มาหา

### แนวทางเฉลยข้อ 4

```c
/* ============================================================
 * ชื่อไฟล์:     alarm_timeout_compute.c
 * คำอธิบาย:     เฉลยแบบฝึกหัด — จำกัดเวลาทำงานของ Busy Loop ด้วย alarm()+SIGALRM
 *              ให้หยุดอัตโนมัติถ้าคำนวณนานเกิน 3 วินาที แทนการรอ input ผู้ใช้
 * ============================================================ */
#define _POSIX_C_SOURCE 200809L
#include <stdio.h>
#include <signal.h>
#include <unistd.h>

static volatile sig_atomic_t g_time_is_up = 0;

static void handle_sigalrm(int signum) {
    (void)signum;
    g_time_is_up = 1;
}

int main(void) {
    struct sigaction sa;
    sa.sa_handler = handle_sigalrm;
    sigemptyset(&sa.sa_mask);
    sa.sa_flags = 0;
    sigaction(SIGALRM, &sa, NULL);

    printf("เริ่มคำนวณ จะหยุดอัตโนมัติถ้าเกิน 3 วินาที...\n");
    alarm(3);

    long iterations = 0;
    volatile long checksum = 0; /* volatile กันไม่ให้ compiler optimize loop ทิ้ง */

    /* ทำงานหนักไปเรื่อยๆ จนกว่า flag จาก SIGALRM handler จะถูกตั้งค่า
       ต้องเช็ค flag บ่อยพอ (ทุกรอบ loop) เพื่อให้หยุดได้ทันทีที่หมดเวลา */
    while (!g_time_is_up) {
        checksum += iterations % 97;
        iterations++;
    }

    alarm(0);
    printf("หยุดทำงานแล้ว: คำนวณไปทั้งหมด %ld รอบ (checksum=%ld) ก่อนหมดเวลา\n",
           iterations, checksum);

    return 0;
}
```

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 -D_POSIX_C_SOURCE=200809L alarm_timeout_compute.c -o alarm_timeout_compute
./alarm_timeout_compute
```

```
เริ่มคำนวณ จะหยุดอัตโนมัติถ้าเกิน 3 วินาที...
หยุดทำงานแล้ว: คำนวณไปทั้งหมด 3021847293 รอบ (checksum=145048164) ก่อนหมดเวลา
```

**คำอธิบาย**: หัวใจของเฉลยนี้คือการเช็ค `g_time_is_up` เป็นเงื่อนไขของ `while` loop โดยตรง
แทนที่จะเช็คแค่ครั้งเดียวก่อนเริ่ม Loop — เพราะ `SIGALRM` อาจมาถึง **ระหว่าง** Loop กำลัง
ทำงานอยู่ (ซึ่งเป็นสถานการณ์ปกติที่คาดหวังไว้) การเช็ค Flag ทุกรอบทำให้โปรแกรมตอบสนองต่อ
Timeout ได้เร็วที่สุดเท่าที่จะทำได้ (ไม่เกิน 1 รอบ Loop หลังจากหมดเวลาจริง) ต่างจาก
`alarm_timeout.c` ต้นฉบับที่ใช้กับ `fgets()` (System Call ที่ถูกขัดจังหวะได้โดยตรงจาก
Signal) เฉลยนี้ใช้กับ Busy Loop ธรรมดาที่ไม่มี System Call ให้ขัดจังหวะ จึงต้องอาศัยการ
เช็ค Flag ในโค้ดของเราเองแทน

### แนวทางเฉลยข้อ 6

```c
/* ============================================================
 * ชื่อไฟล์:     sa_siginfo_demo.c
 * คำอธิบาย:     เฉลยแบบฝึกหัด — ใช้ SA_SIGINFO เพื่อรับข้อมูลเพิ่มเติมเกี่ยวกับ
 *              signal เช่น PID ของ process ที่ส่ง signal มา (si_pid) ผ่าน
 *              handler แบบ sa_sigaction แทน sa_handler ธรรมดา
 * ============================================================ */
#define _POSIX_C_SOURCE 200809L
#include <stdio.h>
#include <signal.h>
#include <unistd.h>
#include <stdlib.h>

static void handle_sigint_info(int signum, siginfo_t *info, void *ucontext) {
    (void)signum;
    (void)ucontext;
    char buf[128];
    int len = snprintf(buf, sizeof(buf),
                        "\n[handler] ได้รับ SIGINT จาก PID=%ld (UID ผู้ส่ง=%ld)\n",
                        (long)info->si_pid, (long)info->si_uid);
    write(STDOUT_FILENO, buf, (size_t)len);
    _exit(0);
}

int main(void) {
    struct sigaction sa;
    sa.sa_sigaction = handle_sigint_info;
    sigemptyset(&sa.sa_mask);
    sa.sa_flags = SA_SIGINFO; /* บอก kernel ว่าเราจะใช้ sa_sigaction ไม่ใช่ sa_handler */

    if (sigaction(SIGINT, &sa, NULL) == -1) {
        perror("sigaction ล้มเหลว");
        return 1;
    }

    printf("PID=%ld รอรับ SIGINT พร้อมข้อมูลผู้ส่ง...\n", (long)getpid());
    pause(); /* หลับรอ signal ใดๆ มาปลุก */

    return 0;
}
```

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 -D_POSIX_C_SOURCE=200809L sa_siginfo_demo.c -o sa_siginfo_demo
./sa_siginfo_demo &
# (จากอีก terminal หรือ shell เดียวกัน)
kill -INT <PID_ของ_sa_siginfo_demo>
```

```
PID=4991 รอรับ SIGINT พร้อมข้อมูลผู้ส่งข้อมูล...

[handler] ได้รับ SIGINT จาก PID=25433 (UID ผู้ส่ง=0)
```

**คำอธิบาย**: จุดต่างที่สำคัญที่สุดของเฉลยนี้คือการใช้ `sa.sa_sigaction` (รับพารามิเตอร์ 3
ตัว: หมายเลข Signal, `siginfo_t *` ที่มีรายละเอียดเพิ่มเติม, และ `void *` สำหรับ Context
ระดับ CPU) แทน `sa.sa_handler` (รับแค่หมายเลข Signal ตัวเดียว) และต้องตั้ง `sa.sa_flags =
SA_SIGINFO` เพื่อบอก Kernel ว่าให้เรียก Handler แบบนี้แทน — `info->si_pid` บอกว่า Process
ไหนเป็นคนเรียก `kill()` ส่ง Signal มาหาเรา ซึ่งเป็นข้อมูลที่ `sa_handler` แบบธรรมดาไม่มีทาง
รู้ได้เลย ประโยชน์จริงของ Field นี้คือการสร้างระบบที่ต้องการตรวจสอบสิทธิ์ว่า "ใครเป็นคนส่ง
Signal มา" ก่อนตัดสินใจตอบสนอง (เช่นปฏิเสธ Signal จาก Process ที่ไม่ได้รับอนุญาต)

---

## สรุปท้ายบท

ใน Part นี้เราได้:

- เข้าใจ **Signal** ในฐานะ Asynchronous Notification ที่มาถึงโปรแกรมได้ทุกเมื่อ พร้อมรู้จัก
  Signal สำคัญที่พบบ่อยที่สุด (`SIGINT`, `SIGTERM`, `SIGKILL`, `SIGSEGV`, `SIGCHLD`,
  `SIGALRM`) และพฤติกรรม Default ของแต่ละตัว
- เปรียบเทียบ **`signal()`** กับ **`sigaction()`** และเข้าใจว่าทำไมโค้ด Production ต้องใช้
  `sigaction()` เพื่อความชัดเจนและ Portable
- เขียน **Signal Handler ที่ปลอดภัย** ตามหลัก Async-Signal-Safety โดยใช้ `write()` แทน
  `printf()` และแพทเทิร์น "ตั้ง Flag แล้วทำงานหนักใน Main Loop"
- จัดการ **Ctrl+C** ให้ Cleanup ทรัพยากร (ไฟล์ Lock) อย่างปลอดภัยก่อนออกโปรแกรมจริง
  (Graceful Shutdown) ซึ่งเป็นแพทเทิร์นที่โปรแกรม Production ทุกตัวต้องมี
- ใช้ **`alarm()` + `SIGALRM`** ตั้ง Timeout ให้กับ Operation ที่อาจค้างรอไม่มีกำหนด
- ใช้ **`sigprocmask()`** บล็อก Signal ชั่วคราวเพื่อปกป้องช่วงวิกฤต (Critical Section)
- ใช้ **`SIGCHLD` + `waitpid(WNOHANG)`** เก็บกวาด Zombie Process โดยอัตโนมัติ แก้ปัญหา
  ที่ค้างมาตั้งแต่ Part 27 ได้อย่างสมบูรณ์และ Production-grade

นี่คือ Part สุดท้ายที่เราทำงานกับ Process **เดี่ยวๆ** — ตั้งแต่ **Part 29** เป็นต้นไป เราจะ
เริ่มให้ Process หลายตัว**สื่อสารกัน** ผ่านกลไกที่เรียกว่า **Inter-Process Communication
(IPC)** เริ่มจาก **Pipe** และ **FIFO (Named Pipe)** ซึ่งจะนำ Mini Shell จาก Part 27 กลับมา
ต่อยอดให้รองรับคำสั่งแบบ `cmd1 | cmd2` ได้จริงเป็นครั้งแรก

**ต่อไป:** [Part 29 — Inter-Process Communication (Pipe/FIFO)](./part-029-ipc-pipes.md)
