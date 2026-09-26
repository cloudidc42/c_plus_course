# Part 31: POSIX Threads (pthreads) เบื้องต้น (Step 241–248)

> Module C — Systems Programming ด้วย C บน Linux | Part 31 จาก 125
> Step ที่ครอบคลุมใน Part นี้: Step 241–248
> Part ก่อนหน้า: [Part 30 — Shared Memory และ Message Queue](./part-030-shared-memory-mq.md) | Part ถัดไป: [Part 32 — Thread Synchronization](./part-032-thread-sync.md)

## เป้าหมายการเรียนรู้ (Learning Objectives)

หลังจบ Part นี้ ผู้เรียนจะสามารถ:

1. อธิบายได้ว่า **thread** คืออะไร และแตกต่างจาก **process** อย่างไร โดยเฉพาะเรื่อง
   การแชร์ address space ร่วมกัน
2. คอมไพล์โปรแกรมที่ใช้ pthread ด้วย flag `-pthread` ได้อย่างถูกต้อง และเข้าใจว่าทำไม
   ต้องใส่ flag นี้
3. ใช้ `pthread_create()` สร้าง thread ใหม่ และ `pthread_join()` รอให้ thread ทำงานจบ
   ได้อย่างถูกต้องและเข้าใจทุกพารามิเตอร์
4. ส่ง argument ให้ thread function ได้อย่างปลอดภัย และอธิบาย/แก้ไขข้อผิดพลาดคลาสสิก
   เรื่องการส่ง address ของตัวแปร loop counter ให้ thread ได้
5. ใช้ `pthread_exit()` ส่งค่ากลับจาก thread และเข้าใจความแตกต่างจากการ `return` ปกติ
6. เขียนโปรแกรมจริงที่สร้างหลาย thread ช่วยกันคำนวณผลรวมของ array คนละส่วน และเข้าใจ
   ว่าทำไมการเขียนตัวแปรที่ใช้ร่วมกันโดยไม่ป้องกันจะทำให้ผลลัพธ์ผิดพลาด (เกริ่นสู่ Part 32)

---

## 31.1 Thread คืออะไร ต่างจาก Process อย่างไร (Step 241)

ใน Part 27-30 เราเรียนเรื่อง process มาอย่างละเอียด รู้ว่า `fork()` สร้าง process ใหม่ที่
มี address space **แยกจากกันอย่างสมบูรณ์** และต้องใช้กลไก IPC (pipe, shared memory,
message queue) เพื่อให้สื่อสารกันได้

**Thread** คือหน่วยของการทำงานอีกแบบหนึ่งที่เบากว่า process มาก จุดต่างที่สำคัญที่สุดคือ:

> **Thread หลายตัวที่อยู่ใน process เดียวกัน จะแชร์ address space เดียวกันโดยอัตโนมัติ**

นั่นหมายความว่า ตัวแปร global, heap memory (จาก `malloc`), และ static memory ทั้งหมด
ของ process จะถูกมองเห็นและเข้าถึงได้จาก**ทุก thread** ในนั้นโดยตรง ไม่ต้องใช้ shared
memory หรือ IPC ใดๆ เลย — นี่คือทั้งข้อดีมหาศาล (ง่ายและเร็วในการแชร์ข้อมูล) และเป็น
ที่มาของปัญหาที่ยากที่สุดของการเขียนโปรแกรมแบบ concurrent (การเข้าถึงพร้อมกันแบบไม่
ปลอดภัย ซึ่งเราจะเห็นในหัวข้อ 31.7 และแก้ไขอย่างจริงจังใน Part 32)

```
Process เดียว มี 3 Thread

┌──────────────────────────────────────────────────────────┐
│                         Process                             │
│  ┌────────────────────────────────────────────────────┐   │
│  │           Heap + Global/Static Data                  │   │
│  │        (Thread ทุกตัวมองเห็นและแก้ไขร่วมกันได้)          │   │
│  └────────────────────────────────────────────────────┘   │
│                                                              │
│  ┌───────────┐      ┌───────────┐      ┌───────────┐       │
│  │ Thread 1   │      │ Thread 2   │      │ Thread 3   │       │
│  │ (Stack ของ  │      │ (Stack ของ  │      │ (Stack ของ  │       │
│  │  ตัวเอง)    │      │  ตัวเอง)    │      │  ตัวเอง)    │       │
│  │ Registers   │      │ Registers   │      │ Registers   │       │
│  │ ของตัวเอง   │      │ ของตัวเอง   │      │ ของตัวเอง   │       │
│  └───────────┘      └───────────┘      └───────────┘       │
└──────────────────────────────────────────────────────────┘
```

สิ่งที่ **แชร์กัน** ระหว่าง thread ในกระบวนการเดียวกัน:

- Heap memory (ที่ได้จาก `malloc`/`calloc`)
- ตัวแปร global และ static
- File descriptor ที่เปิดไว้
- Signal handler
- Process ID, Working Directory

สิ่งที่ **แยกกัน** ในแต่ละ thread:

- **Stack** ของตัวเอง (ตัวแปร local ในแต่ละ thread จึงไม่ปนกัน)
- ชุด register ของ CPU (Program Counter, Stack Pointer ฯลฯ)
- Thread ID
- Signal mask ของตัวเอง (ในบางกรณี)

### ตารางเปรียบเทียบ Process vs Thread

| คุณสมบัติ | Process | Thread |
|---|---|---|
| Address Space | แยกกันอย่างสมบูรณ์ | แชร์กันภายใน process เดียวกัน |
| การสร้างใหม่ | หนัก (ต้อง copy page table ทั้งหมด แม้จะเป็น copy-on-write) | เบามาก (แค่สร้าง stack ใหม่ + register ใหม่) |
| การสื่อสารระหว่างกัน | ต้องใช้ IPC (pipe, shared memory, ฯลฯ) | อ่าน/เขียนตัวแปรร่วมกันได้ตรงๆ |
| หากตัวหนึ่งพัง (crash) | Process อื่นไม่ได้รับผลกระทบ | **Thread อื่นในProcess เดียวกันพังไปด้วย** (เพราะแชร์ address space) |
| Context Switch | ช้ากว่า (ต้องสลับ page table) | เร็วกว่า (ไม่ต้องสลับ page table) |
| ใช้เมื่อ | ต้องการแยกความล้มเหลวออกจากกัน หรือรันคนละโปรแกรม | ต้องการทำงานหลายอย่างพร้อมกันในโปรแกรมเดียว แชร์ข้อมูลกันบ่อยๆ |

จากตารางนี้จะเห็นว่า thread เหมาะกับงานที่ต้องทำหลายอย่างพร้อมกัน**ภายในโปรแกรม
เดียว**และต้องแชร์ข้อมูลกันบ่อยๆ เช่น web server ที่ต้องรับ connection หลายรายพร้อมกัน
หรือโปรแกรมประมวลผลภาพที่แบ่งงานคำนวณออกเป็นส่วนๆ ให้หลาย thread ช่วยกันทำ

---

## 31.2 คอมไพล์ด้วย `-pthread` Flag (Step 242)

ก่อนเขียนโค้ด มาทำความเข้าใจการคอมไพล์ก่อน เพราะเป็นจุดที่มือใหม่มักพลาดตั้งแต่ก้าวแรก

pthread เป็นไลบรารีแยกต่างหากจาก libc มาตรฐาน (แม้ในปัจจุบัน glibc รุ่นใหม่ๆ จะรวม
pthread เข้าไปใน libc หลักแล้วก็ตาม) ต้องบอก compiler และ linker ให้รู้จักมันด้วย flag
`-pthread`:

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 -pthread program.c -o program
```

> **ทำไมใช้ `-pthread` แทน `-lpthread`?**
> วิธีเก่าคือ link ด้วย `-lpthread` ตรงๆ แต่วิธีนี้มีปัญหาเรื่อง thread-safety ของฟังก์ชัน
> อื่นๆ ใน libc (เช่น `errno` ต้องเป็น thread-local) การใช้ flag `-pthread` (ไม่มี lib
> นำหน้า) จะบอก compiler ให้ตั้งค่า preprocessor macro และ flag การคอมไพล์ที่จำเป็น
> ทั้งหมดให้ครบถ้วน ไม่ใช่แค่ link ไลบรารีเฉยๆ **ดังนั้นตลอด Part นี้และ Part 32
   ให้ใช้ `-pthread` เสมอ ไม่ใช้ `-lpthread`**

ถ้าลืมใส่ `-pthread` จะเจอ error ตอน linking แบบนี้:

```
undefined reference to `pthread_create'
undefined reference to `pthread_join'
```

ทุกตัวอย่างโค้ดใน Part นี้ต้องคอมไพล์ด้วยคำสั่งรูปแบบนี้เสมอ:

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 -pthread ชื่อไฟล์.c -o ชื่อโปรแกรม
```

---

## 31.3 และ 31.4 `pthread_create()` และ `pthread_join()` แบบละเอียด (Step 243–244)

### `pthread_create()`

```c
#include <pthread.h>

int pthread_create(pthread_t *thread,
                    const pthread_attr_t *attr,
                    void *(*start_routine)(void *),
                    void *arg);
```

| พารามิเตอร์ | ความหมาย |
|---|---|
| `thread` | pointer ไปยังตัวแปรชนิด `pthread_t` ที่จะใช้เก็บ "handle" ของ thread ที่สร้างขึ้นใหม่ (ใช้อ้างอิงตอน join ทีหลัง) |
| `attr` | คุณสมบัติพิเศษของ thread (เช่น ขนาด stack, detach state) ปกติใส่ `NULL` เพื่อใช้ค่า default |
| `start_routine` | pointer ไปยังฟังก์ชันที่ thread ใหม่จะเริ่มทำงาน ต้องมี signature แบบ `void *func(void *arg)` เท่านั้น |
| `arg` | pointer ที่จะถูกส่งเป็น argument ให้กับ `start_routine` (ถ้าไม่มีข้อมูลจะส่งได้ ใส่ `NULL`) |

คืนค่า `0` เมื่อสำเร็จ, ไม่ใช่ `0` เมื่อผิดพลาด (สังเกตว่า **ไม่ใช่** `-1` เหมือน syscall
ทั่วไป และไม่ได้ตั้งค่า `errno` — pthread ทั้งตระกูลคืนค่า error code ตรงๆ เป็นค่าที่คืน
กลับมาจากฟังก์ชันเลย)

### `pthread_join()`

```c
int pthread_join(pthread_t thread, void **retval);
```

| พารามิเตอร์ | ความหมาย |
|---|---|
| `thread` | handle ของ thread ที่ต้องการรอ (ค่าเดียวกับที่ได้จาก `pthread_create`) |
| `retval` | pointer ไปยัง `void *` ที่จะใช้เก็บค่าที่ thread คืนกลับมา (ผ่าน `return` หรือ `pthread_exit`) ถ้าไม่สนใจค่าที่คืนใส่ `NULL` ได้ |

`pthread_join()` จะ**บล็อกรอ**จนกว่า thread ที่ระบุจะทำงานจบ (คล้ายกับ `wait()` ที่ใช้
รอ child process ใน Part 27) และเป็นการปลดปล่อยทรัพยากรของ thread นั้นคืนให้ระบบ
เหมือนกับที่ `wait()` ป้องกันไม่ให้เกิด zombie process การ**ไม่ join** thread ที่สร้างขึ้น
มา (และไม่ได้ตั้งเป็น detached) จะทำให้ทรัพยากรของมันไม่ถูกคืนจนกว่า process หลักจะจบ
— คล้ายกับ zombie process ที่เรียนใน Part 27

มาดูตัวอย่างการใช้งานทั้งคู่พร้อมกัน:

```c
/* ============================================================
 * ชื่อไฟล์:     thread_basic.c
 * คำอธิบาย:     ตัวอย่าง pthread_create/pthread_join พื้นฐานที่สุด
 * ============================================================ */
#include <stdio.h>
#include <stdlib.h>
#include <pthread.h>

void *print_message(void *arg) {
    const char *msg = (const char *)arg;
    printf("[thread %lu] %s\n", (unsigned long)pthread_self(), msg);
    return NULL;
}

int main(void) {
    pthread_t t1, t2;

    printf("[main] thread หลักเริ่มทำงาน กำลังสร้าง thread ลูก 2 ตัว...\n");

    if (pthread_create(&t1, NULL, print_message, "สวัสดีจาก thread ที่ 1") != 0) {
        fprintf(stderr, "pthread_create t1 ล้มเหลว\n");
        exit(EXIT_FAILURE);
    }
    if (pthread_create(&t2, NULL, print_message, "สวัสดีจาก thread ที่ 2") != 0) {
        fprintf(stderr, "pthread_create t2 ล้มเหลว\n");
        exit(EXIT_FAILURE);
    }

    printf("[main] สร้าง thread ครบแล้ว รอ (join) ให้ทำงานจบ...\n");

    pthread_join(t1, NULL);
    pthread_join(t2, NULL);

    printf("[main] thread ทั้งสองจบการทำงานแล้ว โปรแกรมจบ\n");
    return 0;
}
```

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 -pthread thread_basic.c -o thread_basic
./thread_basic
```

ผลลัพธ์ (thread ID ตัวเลขจริงจะต่างกันไปทุกครั้งที่รัน):

```
[main] thread หลักเริ่มทำงาน กำลังสร้าง thread ลูก 2 ตัว...
[main] สร้าง thread ครบแล้ว รอ (join) ให้ทำงานจบ...
[thread 140311128962752] สวัสดีจาก thread ที่ 1
[thread 140311120570048] สวัสดีจาก thread ที่ 2
[main] thread ทั้งสองจบการทำงานแล้ว โปรแกรมจบ
```

### สังเกตสิ่งสำคัญ

- `print_message()` ถูกเรียกแบบ **asynchronous** คือ main thread ยิงคำสั่งสร้าง thread
  ทั้งสองไปแล้วก็เดินหน้าทำงานต่อทันที (พิมพ์ "สร้าง thread ครบแล้ว...") โดยไม่รอให้
  thread ลูกทำงานเสร็จก่อน ลำดับที่ข้อความจาก thread 1 และ 2 ปรากฏออกมาอาจสลับกัน
  ไปมาในแต่ละครั้งที่รัน เพราะขึ้นอยู่กับการจัดสรรเวลาของ OS scheduler
- `pthread_join()` สองบรรทัดสุดท้ายคือจุดที่ main thread **หยุดรอ** จนกว่า thread ทั้ง
  สองจะทำงานจบสมบูรณ์ ถ้าลบสองบรรทัดนี้ออก main อาจจะจบโปรแกรมไปก่อนที่ thread ลูก
  จะทันพิมพ์อะไรออกมาเลยด้วยซ้ำ (เพราะ process จะจบทันทีที่ `main()` return แม้จะยังมี
  thread อื่นทำงานค้างอยู่)
- `pthread_self()` คืนค่า thread ID ของ thread ปัจจุบันที่กำลังรันโค้ดอยู่

---

## 31.5 การส่ง Argument ให้ Thread Function (Step 245)

ฟังก์ชันที่ thread เรียกต้องมี signature ตายตัวคือ `void *func(void *arg)` — รับ
`void *` เพียงตัวเดียว และคืนค่าเป็น `void *` เท่านั้น ถ้าต้องการส่งข้อมูลมากกว่า 1 ค่า
ต้องรวมไว้ใน struct แล้วส่ง pointer ของ struct นั้นแทน

### ข้อผิดพลาดคลาสสิกที่สุดของการเขียนโปรแกรมแบบ Multithread

นี่คือบั๊กที่โปรแกรมเมอร์แทบทุกคนเคยเจอมาแล้วอย่างน้อยหนึ่งครั้งในชีวิต: การส่ง address
ของตัวแปร **loop counter** ให้ thread โดยตรง

```c
/* ============================================================
 * ชื่อไฟล์:     thread_arg_bug.c
 * คำอธิบาย:     ข้อผิดพลาดคลาสสิก — ส่ง address ของตัวแปร loop counter
 *              ให้ thread โดยตรง ทำให้ค่าที่ thread อ่านได้ผิดเพี้ยน
 * ============================================================ */
#include <stdio.h>
#include <stdlib.h>
#include <pthread.h>

#define NUM_THREADS 5

void *worker(void *arg) {
    int id = *(int *)arg; /* อ่านค่าจาก address ที่ได้รับมา */
    printf("[worker] เริ่มทำงานด้วย id = %d\n", id);
    return NULL;
}

int main(void) {
    pthread_t threads[NUM_THREADS];

    for (int i = 0; i < NUM_THREADS; i++) {
        /* BUG: &i คือ address ของตัวแปร i ตัวเดียวที่ใช้ร่วมกันทุกรอบ loop
         * พอ main วน loop เร็วกว่า thread ที่ถูกสร้างจะอ่านค่าทัน
         * ค่า i ที่ thread เห็นจึงอาจไม่ใช่ค่าตอนที่มันถูกสร้างเลย */
        if (pthread_create(&threads[i], NULL, worker, &i) != 0) {
            fprintf(stderr, "pthread_create ล้มเหลว\n");
            exit(EXIT_FAILURE);
        }
    }

    for (int i = 0; i < NUM_THREADS; i++) {
        pthread_join(threads[i], NULL);
    }

    printf("[main] คาดหวังว่าจะเห็น id = 0,1,2,3,4 แต่ผลจริงมักซ้ำกันหรือข้ามค่า\n");
    return 0;
}
```

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 -pthread thread_arg_bug.c -o thread_arg_bug
./thread_arg_bug
```

ผลลัพธ์จริงที่ได้ (รันซ้ำแต่ละครั้งจะได้ผลไม่เหมือนเดิม แต่จะไม่มีทางได้ `0,1,2,3,4`
เรียงกันสวยงามแบบที่คาดหวังไว้):

```
[worker] เริ่มทำงานด้วย id = 4
[worker] เริ่มทำงานด้วย id = 4
[worker] เริ่มทำงานด้วย id = 5
[worker] เริ่มทำงานด้วย id = 4
[worker] เริ่มทำงานด้วย id = 5
[main] คาดหวังว่าจะเห็น id = 0,1,2,3,4 แต่ผลจริงมักซ้ำกันหรือข้ามค่า
```

### ทำไมถึงเกิดปัญหานี้

ตัวแปร `i` ในลูป `for` เป็นตัวแปรตัวเดียวที่ถูกใช้ซ้ำไปเรื่อยๆ ในทุกรอบของลูป การส่ง
`&i` คือการส่ง **address ของตัวแปรตัวนั้น** ไม่ใช่ "ค่า" ของมัน ณ เวลานั้น เมื่อ
`pthread_create()` คืนค่ากลับมา (ซึ่งเร็วมาก) `main()` จะวนไป increment `i` ต่อทันที
**ก่อนที่** thread ใหม่จะได้เริ่มรันจริงๆ ด้วยซ้ำ (การ schedule thread ขึ้นมารันจริงเป็น
เรื่องของ OS ซึ่งไม่ได้เกิดขึ้นทันทีเสมอไป) พอ thread เริ่มอ่านค่าจาก `arg` มันจึงอาจเจอ
ค่า `i` ที่ถูก main เปลี่ยนไปแล้ว (บางค่าถูกอ่านซ้ำสองรอบ บางค่าถูกข้ามไปเลย และตัวเลข
`5` ที่ไม่ควรมีอยู่จริง — เพราะเป็นค่าของ `i` **หลัง**ลูปจบ — ก็ปรากฏขึ้นมาได้)

```
เวลา    main() (loop สร้าง thread)         Worker threads (อ่านค่า i)
─────   ──────────────────────────────    ─────────────────────────
t1      i=0, pthread_create(&i) [T0]
t2      i=1, pthread_create(&i) [T1]
t3      i=2, pthread_create(&i) [T2]
t4                                          T0 เพิ่งเริ่มรัน อ่าน *arg -> ได้ 3! (ไม่ใช่ 0)
t5      i=3, pthread_create(&i) [T3]
t6      i=4, pthread_create(&i) [T4]
t7      i=5 (loop จบแล้ว)                   T1, T2 เริ่มรัน อ่าน *arg -> ได้ 5 ทั้งคู่!
```

### วิธีแก้: ให้แต่ละ Thread มี "สำเนา" ของตัวเอง

ทางแก้ที่ถูกต้องและเป็นมาตรฐานที่สุดคือ **สร้างหน่วยความจำแยกกันคนละช่องสำหรับแต่ละ
thread** แล้วส่ง address ของช่องนั้นๆ แทนที่จะส่ง address ของตัวแปร loop counter ตรงๆ

```c
/* ============================================================
 * ชื่อไฟล์:     thread_arg_fixed.c
 * คำอธิบาย:     แก้บั๊กจาก thread_arg_bug.c โดยให้แต่ละ thread มี "สำเนา" ของค่า id
 *              เป็นของตัวเอง ไม่ใช้ address ของ loop counter ร่วมกัน
 * ============================================================ */
#include <stdio.h>
#include <stdlib.h>
#include <pthread.h>

#define NUM_THREADS 5

void *worker(void *arg) {
    int id = *(int *)arg;
    printf("[worker] เริ่มทำงานด้วย id = %d\n", id);
    return NULL;
}

int main(void) {
    pthread_t threads[NUM_THREADS];
    int thread_ids[NUM_THREADS]; /* หน่วยความจำแยกกันคนละช่องต่อ thread */

    for (int i = 0; i < NUM_THREADS; i++) {
        thread_ids[i] = i; /* copy ค่าลงช่องของตัวเองก่อนสร้าง thread */
        if (pthread_create(&threads[i], NULL, worker, &thread_ids[i]) != 0) {
            fprintf(stderr, "pthread_create ล้มเหลว\n");
            exit(EXIT_FAILURE);
        }
    }

    for (int i = 0; i < NUM_THREADS; i++) {
        pthread_join(threads[i], NULL);
    }

    printf("[main] ตอนนี้แต่ละ thread มี address ของตัวเอง (&thread_ids[i]) ผลลัพธ์เลยถูกต้องเสมอ\n");
    return 0;
}
```

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 -pthread thread_arg_fixed.c -o thread_arg_fixed
./thread_arg_fixed
```

ผลลัพธ์ (ลำดับที่ปรากฏอาจสลับกันได้ เพราะ thread ทำงานพร้อมกัน แต่**ค่าทั้ง 5 ค่าจะ
ถูกต้องเสมอ** คือมี 0, 1, 2, 3, 4 ครบไม่ซ้ำไม่ขาด):

```
[worker] เริ่มทำงานด้วย id = 1
[worker] เริ่มทำงานด้วย id = 3
[worker] เริ่มทำงานด้วย id = 2
[worker] เริ่มทำงานด้วย id = 0
[worker] เริ่มทำงานด้วย id = 4
[main] ตอนนี้แต่ละ thread มี address ของตัวเอง (&thread_ids[i]) ผลลัพธ์เลยถูกต้องเสมอ
```

เพราะตอนนี้แต่ละ thread ได้รับ address ของ **ช่องหน่วยความจำที่แยกจากกันโดยเฉพาะ**
(`&thread_ids[0]`, `&thread_ids[1]`, ...) ไม่มีใครมาเขียนทับค่าที่ thread อื่นกำลังจะอ่าน
อีกต่อไป แม้ main จะวนลูปเร็วแค่ไหนก็ไม่กระทบกับค่าที่ thread แต่ละตัวจะอ่านได้

> **ทางเลือกอื่นที่ใช้ได้เช่นกัน**: ในระบบ 64-bit ที่ `sizeof(void*) >= sizeof(int)`
> บางคนนิยม "แอบยัด" ค่า `int` ลงไปใน `void*` ตรงๆ ด้วยการ cast ผ่าน `intptr_t` เช่น
> `pthread_create(&threads[i], NULL, worker, (void*)(intptr_t)i)` แล้วอ่านค่ากลับด้วย
> `int id = (int)(intptr_t)arg;` วิธีนี้หลีกเลี่ยงปัญหาเรื่อง address ได้เพราะไม่ได้ส่ง
> pointer จริงๆ แต่ซ่อนตัวเลขไว้ในค่าของ pointer เอง — ใช้ได้ผลแต่ควรระวังเรื่องความ
> ชัดเจนของโค้ด (readability) เพราะทำให้คนอ่านโค้ดสับสนได้ง่ายกว่าวิธี array ด้านบน

---

## 31.6 `pthread_exit()` (Step 246)

thread function สามารถจบการทำงานได้สองวิธี: **`return`** ค่ากลับไปตามปกติ (เหมือน
ฟังก์ชันทั่วไป) หรือเรียก **`pthread_exit()`** อย่างชัดเจน

```c
void pthread_exit(void *retval);
```

`pthread_exit(retval)` ทำสิ่งเดียวกับ `return retval;` ที่ท้ายฟังก์ชัน thread ทุกประการ
— คือจบการทำงานของ thread ปัจจุบันและส่งค่า `retval` กลับไปให้ผู้ที่เรียก
`pthread_join()` มารับ แต่ข้อดีของ `pthread_exit()` คือ **เรียกได้จากจุดใดก็ได้ในโปรแกรม**
ไม่จำเป็นต้องรอให้ control flow ไหลไปถึงบรรทัดสุดท้ายของฟังก์ชัน แม้จะอยู่ลึกในฟังก์ชัน
ย่อยที่ thread function เรียกใช้ต่อไปอีกหลายชั้นก็ยังเรียก `pthread_exit()` เพื่อจบ
thread ปัจจุบันได้ทันที (คล้ายกับความสัมพันธ์ระหว่าง `exit()` กับ `return` ใน `main()`
ที่เรียนไปตั้งแต่ Part 1)

ตัวอย่างการใช้ `pthread_exit()` ส่งค่ากลับที่จัดสรรด้วย `malloc()`:

```c
void *sum_range(void *arg) {
    range_t *r = (range_t *)arg;

    long *partial_sum = malloc(sizeof(long));
    if (partial_sum == NULL) {
        pthread_exit(NULL); /* จบ thread ทันทีถ้า malloc ล้มเหลว ไม่ต้องรันโค้ดต่อ */
    }
    *partial_sum = 0;

    for (int i = r->start; i < r->end; i++) {
        *partial_sum += g_array[i];
    }

    pthread_exit(partial_sum); /* ส่งค่ากลับผ่าน pthread_exit เทียบเท่า return partial_sum; */
}
```

> **ข้อควรระวัง**: ค่าที่ส่งกลับผ่าน `pthread_exit()` หรือ `return` **ห้ามเป็น address
> ของตัวแปร local** ในฟังก์ชัน thread นั้นเด็ดขาด เพราะตัวแปร local จะถูกทำลายทิ้งทันที
> ที่ thread จบการทำงาน (stack ของ thread ถูกคืนกลับให้ระบบ) ทำให้ pointer ที่ส่งกลับไป
> กลายเป็น **dangling pointer** ทันที ต้องใช้ `malloc()` จัดสรรหน่วยความจำบน heap แทน
> เสมอ (heap ไม่ได้ผูกกับ stack ของ thread ใดๆ จึงยังอยู่ต่อได้หลัง thread จบ) แล้วให้
> ฝั่งที่เรียก `pthread_join()` เป็นคน `free()` มันทิ้งเมื่อใช้งานเสร็จ

**สำคัญ**: การเรียก `pthread_exit()` ใน **main thread** (thread ที่รัน `main()`) มี
พฤติกรรมพิเศษ — มันจะทำให้ main thread จบ แต่ **thread อื่นๆ ที่ยังทำงานอยู่จะไม่ถูก
ฆ่าตาม** (ต่างจากการเรียก `exit()` หรือ `return` ปกติใน `main()` ที่จะทำให้ทั้ง process
จบทันที รวมถึง thread อื่นๆ ด้วย) รายละเอียดนี้มีประโยชน์ในบางกรณี เช่นเมื่อต้องการให้
main thread จบไปก่อนแต่ยังให้ worker thread อื่นๆ ทำงานต่อจนเสร็จ แต่โดยทั่วไปในโค้ด
ระดับ Part นี้เราจะใช้ `pthread_join()` รอทุก thread ให้จบก่อนที่ `main()` จะ `return`
เสมอ เพื่อความชัดเจนและปลอดภัยที่สุด

---

## 31.7 ตัวอย่างจริง: หลาย Thread ช่วยกันคำนวณผลรวมของ Array (Step 247)

มาถึงตัวอย่างที่ครบวงจรที่สุดของ Part นี้: ใช้ 4 thread ช่วยกันคำนวณผลรวมของ array
ขนาดใหญ่ โดยแบ่งงานให้แต่ละ thread รับผิดชอบคนละช่วง (range) ของ array

```c
/* ============================================================
 * ชื่อไฟล์:     thread_sum_race.c
 * คำอธิบาย:     ใช้หลาย thread ช่วยกันคำนวณผลรวมของ array คนละส่วน
 *              (ยังไม่ป้องกัน race condition — ดูว่าผลลัพธ์ผิดได้อย่างไร
 *              Part 32 จะแนะนำ mutex เพื่อแก้ปัญหานี้อย่างถูกวิธี)
 * ============================================================ */
#include <stdio.h>
#include <stdlib.h>
#include <pthread.h>

#define ARRAY_SIZE 4000000
#define NUM_THREADS 4

static int g_array[ARRAY_SIZE];
static long g_total = 0; /* ตัวแปร global ที่ทุก thread จะแก้ไขร่วมกัน (ไม่มีการป้องกัน!) */

typedef struct {
    int start; /* index เริ่มต้นของ chunk ที่ thread นี้รับผิดชอบ */
    int end;   /* index สิ้นสุด (ไม่รวม) */
} range_t;

void *sum_range(void *arg) {
    range_t *r = (range_t *)arg;

    for (int i = r->start; i < r->end; i++) {
        /* บั๊ก: g_total += g_array[i] ไม่ใช่ operation เดียวในระดับ CPU
         * แต่คือ read-modify-write 3 ขั้นตอน ถ้า thread อื่นแทรกเข้ามา
         * ระหว่างขั้นตอนเหล่านี้ ผลบวกบางค่าจะหายไปจากผลรวมสุดท้าย */
        g_total += g_array[i];
    }

    return NULL;
}

int main(void) {
    /* เติมข้อมูล: ให้ทุกช่องมีค่า 1 เพื่อให้คำนวณผลลัพธ์ที่ถูกต้องได้ง่ายๆ */
    for (int i = 0; i < ARRAY_SIZE; i++) {
        g_array[i] = 1;
    }

    pthread_t threads[NUM_THREADS];
    range_t ranges[NUM_THREADS];
    int chunk = ARRAY_SIZE / NUM_THREADS;

    for (int i = 0; i < NUM_THREADS; i++) {
        ranges[i].start = i * chunk;
        ranges[i].end = (i == NUM_THREADS - 1) ? ARRAY_SIZE : (i + 1) * chunk;
        if (pthread_create(&threads[i], NULL, sum_range, &ranges[i]) != 0) {
            fprintf(stderr, "pthread_create ล้มเหลว\n");
            exit(EXIT_FAILURE);
        }
    }

    for (int i = 0; i < NUM_THREADS; i++) {
        pthread_join(threads[i], NULL);
    }

    printf("[main] ผลรวมที่คาดหวัง: %d\n", ARRAY_SIZE);
    printf("[main] ผลรวมที่ได้จริง: %ld\n", g_total);
    if (g_total != ARRAY_SIZE) {
        printf("[main] --> เกิด Race Condition! ผลรวมผิดเพราะหลาย thread เขียน g_total พร้อมกัน\n");
        printf("[main] --> ใน Part 32 เราจะใช้ mutex ป้องกันปัญหานี้อย่างถูกวิธี\n");
    }

    return 0;
}
```

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 -pthread thread_sum_race.c -o thread_sum_race
./thread_sum_race
```

ผลลัพธ์จริงที่ได้ (ตัวเลขจะแตกต่างกันไปทุกครั้งที่รัน):

```
[main] ผลรวมที่คาดหวัง: 4000000
[main] ผลรวมที่ได้จริง: 1120516
[main] --> เกิด Race Condition! ผลรวมผิดเพราะหลาย thread เขียน g_total พร้อมกัน
[main] --> ใน Part 32 เราจะใช้ mutex ป้องกันปัญหานี้อย่างถูกวิธี
```

ทั้งที่แต่ละช่อง `g_array[i]` มีค่า `1` ทั้งหมด และผลรวมที่ถูกต้องควรเป็น `4000000`
พอดี แต่เพราะทั้ง 4 thread เขียน `g_total += g_array[i]` เข้าตัวแปร **global ตัวเดียวกัน**
พร้อมๆ กันโดยไม่มีการป้องกันเลย ผลลัพธ์จึงผิดพลาดไปมาก (เหตุผลเชิงลึกเหมือนกับที่อธิบาย
ไว้ใน Part 30 หัวข้อ 30.5 ทุกประการ เพียงแค่คราวนี้เป็น thread ไม่ใช่ process แล้ว)

### วิธีหลีกเลี่ยงปัญหานี้แบบไม่ต้องใช้ Lock เลย: ให้แต่ละ Thread สะสมผลของตัวเอง

ก่อนที่ Part 32 จะสอนวิธีใช้ mutex ป้องกัน shared state อย่างถูกต้อง มีหลักการออกแบบ
ที่สำคัญกว่านั้นอีกขั้นที่ควรรู้ไว้ก่อน: **วิธีที่ดีที่สุดในการหลีกเลี่ยง race condition คือ
การไม่มี shared mutable state ตั้งแต่แรก** ถ้าออกแบบให้แต่ละ thread สะสมผลลัพธ์ไว้ใน
หน่วยความจำของตัวเอง (ไม่แตะตัวแปร global ระหว่างทำงานเลย) แล้วให้ main thread เป็น
ผู้รวมผลทีหลัง**หลังจาก** `pthread_join()` สำเร็จแล้วเท่านั้น ก็จะไม่มีการเขียนพร้อมกัน
เกิดขึ้นเลย จึงไม่มีทางเกิด race condition ได้ตั้งแต่ต้น:

```c
/* ============================================================
 * ชื่อไฟล์:     thread_sum_safe.c
 * คำอธิบาย:     เวอร์ชันที่ "หลีกเลี่ยง" race condition โดยให้แต่ละ thread
 *              สะสมผลรวมไว้ใน "หน่วยความจำของตัวเอง" แล้วส่งค่ากลับผ่าน
 *              pthread_exit() แทนที่จะเขียนแก้ตัวแปร global ร่วมกัน
 * ============================================================ */
#include <stdio.h>
#include <stdlib.h>
#include <pthread.h>

#define ARRAY_SIZE 4000000
#define NUM_THREADS 4

static int g_array[ARRAY_SIZE];

typedef struct {
    int start;
    int end;
} range_t;

void *sum_range(void *arg) {
    range_t *r = (range_t *)arg;

    /* ใช้หน่วยความจำที่จัดสรรเฉพาะของ thread นี้ ไม่มี thread ตัวอื่นแตะต้องได้ */
    long *partial_sum = malloc(sizeof(long));
    if (partial_sum == NULL) {
        pthread_exit(NULL);
    }
    *partial_sum = 0;

    for (int i = r->start; i < r->end; i++) {
        *partial_sum += g_array[i];
    }

    /* ส่งค่ากลับผ่าน pthread_exit() — เทียบเท่ากับการ return จากฟังก์ชัน
     * แต่เรียกได้จากทุกจุดใน thread function ไม่ต้องรอไหลมาถึงท้ายฟังก์ชัน */
    pthread_exit(partial_sum);
}

int main(void) {
    for (int i = 0; i < ARRAY_SIZE; i++) {
        g_array[i] = 1;
    }

    pthread_t threads[NUM_THREADS];
    range_t ranges[NUM_THREADS];
    int chunk = ARRAY_SIZE / NUM_THREADS;

    for (int i = 0; i < NUM_THREADS; i++) {
        ranges[i].start = i * chunk;
        ranges[i].end = (i == NUM_THREADS - 1) ? ARRAY_SIZE : (i + 1) * chunk;
        if (pthread_create(&threads[i], NULL, sum_range, &ranges[i]) != 0) {
            fprintf(stderr, "pthread_create ล้มเหลว\n");
            exit(EXIT_FAILURE);
        }
    }

    long total = 0;
    for (int i = 0; i < NUM_THREADS; i++) {
        void *retval;
        pthread_join(threads[i], &retval); /* ได้ pointer ที่ thread ส่งผ่าน pthread_exit() */
        long *partial_sum = (long *)retval;
        total += *partial_sum; /* บวกทีละตัว "หลัง" join สำเร็จเท่านั้น ไม่มีการแข่งกันเขียน */
        free(partial_sum);
    }

    printf("[main] ผลรวมที่คาดหวัง: %d\n", ARRAY_SIZE);
    printf("[main] ผลรวมที่ได้จริง: %ld (ถูกต้องเสมอ เพราะไม่มี shared write ระหว่าง thread)\n", total);

    return 0;
}
```

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 -pthread thread_sum_safe.c -o thread_sum_safe
./thread_sum_safe
```

ผลลัพธ์ (ได้ค่าถูกต้อง **ทุกครั้ง** ไม่ว่าจะรันกี่รอบก็ตาม):

```
[main] ผลรวมที่คาดหวัง: 4000000
[main] ผลรวมที่ได้จริง: 4000000 (ถูกต้องเสมอ เพราะไม่มี shared write ระหว่าง thread)
```

จุดสำคัญที่ทำให้เวอร์ชันนี้ปลอดภัย คือ **ไม่มีตัวแปรใดๆ ที่ถูกเขียนพร้อมกันโดยหลาย
thread เลยตลอดช่วงที่ thread กำลังทำงานอยู่** การรวมผล (`total += *partial_sum;`)
เกิดขึ้นใน main thread เพียงตัวเดียว **หลังจาก** `pthread_join()` ยืนยันแล้วว่า thread
นั้นทำงานจบสมบูรณ์แล้วเท่านั้น — เป็นการรวมแบบ**เรียงลำดับ (sequential)** ไม่ใช่แบบ
**พร้อมกัน (concurrent)**

> **ข้อคิดสำคัญ**: การหลีกเลี่ยง shared mutable state (เท่าที่ทำได้) มักจะดีกว่าการพึ่งพา
> lock เสมอ เพราะไม่มี overhead ของการล็อก/ปลดล็อก และไม่มีความเสี่ยงเรื่อง deadlock
> เลย แต่ในโลกจริงมีหลายสถานการณ์ที่ไม่สามารถหลีกเลี่ยง shared state ได้ (เช่น
> ตัวนับจำนวนคำขอที่เข้ามาในเซิร์ฟเวอร์แบบ real-time) กรณีเหล่านั้นจำเป็นต้องใช้
> mutex, semaphore หรือ atomic operation ซึ่งเป็นเนื้อหาเต็มรูปแบบของ **Part 32**

---

## 31.8 สรุปวงจรชีวิตของ Thread (Step 248)

มาสรุปภาพรวมทั้งหมดของวงจรชีวิต thread หนึ่งตัวตั้งแต่เกิดจนตาย:

```
┌──────────────────────────────────────────────────────────────────┐
│  main() หรือ thread อื่น เรียก pthread_create(&t, NULL, func, arg)   │
└──────────────────────────────────────────────────────────────────┘
                              │
                              ▼
              ┌───────────────────────────────┐
              │  Thread ใหม่เกิดขึ้น เริ่มรัน func(arg) │
              │  (มี stack ของตัวเอง, แชร์ heap/     │
              │   global ร่วมกับ thread อื่นๆ)         │
              └───────────────────────────────┘
                              │
                    ทำงานไปเรื่อยๆ จนกว่าจะ...
                              │
              ┌───────────────┴───────────────┐
              ▼                               ▼
    ┌─────────────────────┐       ┌─────────────────────┐
    │  return retval;       │       │  pthread_exit(retval); │
    │  (ไหลถึงท้ายฟังก์ชัน)   │       │  (เรียกจากจุดใดก็ได้)   │
    └─────────────────────┘       └─────────────────────┘
              │                               │
              └───────────────┬───────────────┘
                              ▼
              ┌───────────────────────────────┐
              │  Thread จบการทำงาน แต่ทรัพยากร   │
              │  ยังไม่ถูกคืนจนกว่าจะถูก join     │
              └───────────────────────────────┘
                              │
                              ▼
              ┌───────────────────────────────┐
              │  ผู้สร้างเรียก pthread_join(t, &r)│
              │  ได้ค่า retval กลับมา + ทรัพยากร │
              │  ของ thread ถูกคืนให้ระบบครบถ้วน  │
              └───────────────────────────────┘
```

### Checklist ก่อนใช้งาน pthread ในโค้ดจริง

1. คอมไพล์ด้วย `-pthread` เสมอ (ไม่ใช่ `-lpthread`)
2. `pthread_create()` ทุกครั้งต้องเช็คค่าที่คืนกลับ (ไม่ใช่ `-1` แต่คืน error code ตรงๆ)
3. `pthread_join()` ทุก thread ที่สร้างขึ้นมาเสมอ ไม่งั้นทรัพยากรจะรั่วไหล (เว้นแต่ตั้งใจ
   ทำ detached thread ด้วย `pthread_detach()` ซึ่งเป็นเนื้อหาขั้นสูงกว่าที่จะพูดถึงใน
   Part หลังๆ)
4. อย่าส่ง address ของตัวแปร local ที่กำลังจะถูกทำลาย (เช่น loop counter หรือตัวแปร
   ในฟังก์ชันที่ return ไปแล้ว) ให้ thread ใช้งานต่อ
5. อย่าคืนค่า address ของตัวแปร local ของ thread function ผ่าน `return`/`pthread_exit`
   ต้องใช้ `malloc()` เสมอถ้าต้องการส่งข้อมูลกลับ
6. ถ้าหลาย thread ต้องเขียนตัวแปรร่วมกัน ให้คิดก่อนเสมอว่า "หลีกเลี่ยงได้ไหม" (ให้แต่ละ
   thread สะสมผลของตัวเองแล้วรวมทีหลัง) ถ้าหลีกเลี่ยงไม่ได้จริงๆ ต้องใช้กลไก
   synchronization ที่เหมาะสม (Part 32)

---

## ข้อผิดพลาดที่พบบ่อย (Common Pitfalls)

1. **ลืม `-pthread` ตอนคอมไพล์** — ได้ error `undefined reference to 'pthread_create'`
   ตอน linking แม้โค้ดจะถูกต้องทุกอย่าง ต้องใส่ flag `-pthread` ทุกครั้งที่ใช้ pthread
2. **ส่ง address ของตัวแปร loop counter ให้หลาย thread โดยตรง** — บั๊กคลาสสิกที่สาธิต
   ในหัวข้อ 31.5 ทำให้ thread อ่านค่าที่ผิดเพี้ยนไปเพราะตัวแปรถูกเปลี่ยนไปแล้วก่อนที่
   thread จะทันอ่าน ต้องสร้างหน่วยความจำแยกกันคนละช่องให้แต่ละ thread เสมอ
3. **ไม่ `pthread_join()` thread ที่สร้างขึ้นมา** — ทำให้ทรัพยากรของ thread ไม่ถูกคืน
   (คล้ายกับ zombie process) และถ้า `main()` จบไปก่อนที่ thread ลูกจะทำงานเสร็จ
   thread ลูกจะถูกฆ่าทิ้งกลางคันทันทีโดยไม่มีการเตือนใดๆ
4. **คืนค่า address ของตัวแปร local (บน stack ของ thread) ผ่าน `return`/`pthread_exit`**
   — ทันทีที่ thread จบ stack ของมันจะถูกทำลาย ทำให้ pointer ที่ส่งกลับไปกลายเป็น
   dangling pointer การใช้งานค่านั้นต่อคือ Undefined Behavior ต้องใช้ `malloc()` เสมอ
5. **เข้าใจผิดว่า thread สองตัวจะรันเรียงลำดับตามที่สร้าง** — thread ที่ถูกสร้างก่อนไม่
   ได้แปลว่าจะรันเสร็จหรือแม้แต่เริ่มรันก่อน การ schedule เป็นหน้าที่ของ OS ทั้งหมด
   ลำดับการทำงานจริงจึงไม่แน่นอน (non-deterministic) ห้ามเขียนโค้ดที่พึ่งพาลำดับการ
   ทำงานของ thread โดยไม่มีกลไก synchronization รองรับ
6. **เขียนตัวแปร global/shared ร่วมกันจากหลาย thread โดยไม่ป้องกัน** — สาเหตุของ
   race condition ดังที่สาธิตในหัวข้อ 31.7 ผลลัพธ์จะผิดพลาดแบบสุ่มและ debug ได้ยาก
   มาก เพราะบั๊กแบบนี้อาจไม่แสดงอาการทุกครั้งที่รัน (เรียกว่า "Heisenbug")
7. **ลืมว่า thread ทั้งหมดจะถูกฆ่าทันทีถ้า process หลักจบ (หรือ crash)** — ต่างจาก
   process ที่ถ้าตัวหนึ่ง crash ตัวอื่นยังทำงานต่อได้ปกติ ถ้า thread ใดตัวหนึ่งทำให้เกิด
   segmentation fault (เช่น เข้าถึง memory ที่ไม่ถูกต้อง) **ทั้ง process จะพังไปด้วย
   ทันที** รวมถึง thread อื่นๆ ทั้งหมดที่กำลังทำงานอยู่ในเวลานั้น

---

## แบบฝึกหัดท้ายบท

1. เขียนโปรแกรมสร้าง 5 thread ที่แต่ละตัวรับตัวเลข `n` ที่แตกต่างกัน แล้วคำนวณ
   `factorial(n)` เก็บผลลัพธ์ลง array คนละ index (ไม่มี race condition เพราะเขียนคนละ
   ช่อง) แล้วให้ main พิมพ์ผลลัพธ์ทั้งหมดออกมาหลัง join ครบทุก thread
2. ทดลองลบบรรทัด `pthread_join()` ออกจาก `thread_basic.c` ทั้งหมด แล้วสังเกตว่า
   ข้อความจาก thread ลูกยังปรากฏขึ้นมาบนหน้าจอหรือไม่ ลองรันซ้ำหลายครั้งเพื่อดูว่า
   ผลลัพธ์คงที่หรือไม่คงที่ อธิบายเหตุผลด้วยคำพูดตัวเอง
3. เขียนโปรแกรมวัดเวลาเปรียบเทียบระหว่างการสร้าง+รอ 2000 thread กับการสร้าง+รอ
   2000 process (ด้วย `fork()`/`wait()`) แล้วรายงานว่า process หนักกว่า thread กี่เท่า
4. แก้ไข `thread_arg_bug.c` ด้วยวิธีที่ **ไม่ใช้ array แยกช่อง** แต่ใช้การ cast ตัวเลข
   ผ่าน `intptr_t` แทน (ดูคำอธิบายท้ายหัวข้อ 31.5 เป็นแนวทาง) แล้วทดสอบว่าได้ผลลัพธ์
   ถูกต้องเหมือนกับวิธี array หรือไม่
5. เขียนโปรแกรมที่ thread function เรียก `pthread_exit()` จากภายในฟังก์ชันย่อยอีกชั้น
   หนึ่ง (ไม่ใช่เรียกตรงๆ ใน thread function หลัก) เพื่อพิสูจน์ว่า `pthread_exit()`
   สามารถเรียกได้จากทุกความลึกของ call stack
6. อธิบายว่าทำไมโปรแกรมที่ thread หนึ่งเกิด Segmentation Fault จะทำให้ทั้งโปรแกรม
   ปิดตัวลงทันที ทั้งที่ thread อื่นๆ ยังทำงานปกติดีอยู่ (เทียบกับกรณี `fork()` ที่ child
   process หนึ่ง crash แต่ parent process ยังทำงานต่อได้)

### แนวทางเฉลยข้อ 1 (Factorial หลาย Thread)

```c
/* ============================================================
 * ชื่อไฟล์:     ex_factorial_threads.c
 * คำอธิบาย:     เฉลยแบบฝึกหัด — สร้างหลาย thread คำนวณ factorial ของตัวเลข
 *              คนละตัวพร้อมกัน แล้วเก็บผลลัพธ์ลง array คนละ index (ไม่มี
 *              การเขียนซ้อนทับกัน จึงไม่มี race condition)
 * ============================================================ */
#include <stdio.h>
#include <stdlib.h>
#include <pthread.h>

#define NUM_TASKS 6

typedef struct {
    int number;                /* ตัวเลขที่ต้องคำนวณ factorial */
    unsigned long long result; /* ที่เก็บผลลัพธ์ (คนละช่องต่อ thread) */
} task_t;

void *compute_factorial(void *arg) {
    task_t *task = (task_t *)arg;
    unsigned long long result = 1;
    for (int i = 2; i <= task->number; i++) {
        result *= (unsigned long long)i;
    }
    task->result = result; /* เขียนลงช่องของตัวเองเท่านั้น ไม่ชนกับ thread อื่น */
    return NULL;
}

int main(void) {
    pthread_t threads[NUM_TASKS];
    task_t tasks[NUM_TASKS] = {
        {.number = 5}, {.number = 7}, {.number = 10},
        {.number = 12}, {.number = 3}, {.number = 15}
    };

    for (int i = 0; i < NUM_TASKS; i++) {
        if (pthread_create(&threads[i], NULL, compute_factorial, &tasks[i]) != 0) {
            fprintf(stderr, "pthread_create ล้มเหลวที่ index %d\n", i);
            exit(EXIT_FAILURE);
        }
    }

    for (int i = 0; i < NUM_TASKS; i++) {
        pthread_join(threads[i], NULL);
    }

    for (int i = 0; i < NUM_TASKS; i++) {
        printf("%d! = %llu\n", tasks[i].number, tasks[i].result);
    }

    return 0;
}
```

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 -pthread ex_factorial_threads.c -o ex_factorial_threads
./ex_factorial_threads
```

ผลลัพธ์ที่ได้จริง:

```
5! = 120
7! = 5040
10! = 3628800
12! = 479001600
3! = 6
15! = 1307674368000
```

สังเกตว่าผลลัพธ์พิมพ์ออกมาตามลำดับ index ใน array เสมอ (5!, 7!, 10!, ...) แม้ว่า
thread แต่ละตัวจะคำนวณเสร็จไม่พร้อมกันก็ตาม เพราะการพิมพ์ผลลัพธ์เกิดขึ้น **หลัง**
`pthread_join()` ครบทุกตัวแล้วในลูปแยกต่างหาก ซึ่งเป็นการวนแบบเรียงลำดับปกติ

### แนวทางเฉลยข้อ 3 (เปรียบเทียบความเร็ว Thread vs Process)

```c
/* ============================================================
 * ชื่อไฟล์:     ex_thread_vs_process_benchmark.c
 * คำอธิบาย:     เฉลยแบบฝึกหัด — วัดเวลาที่ใช้สร้างและรอ (join/wait) thread
 *              จำนวนมากเทียบกับ process จำนวนเท่ากัน เพื่อพิสูจน์ด้วยตัวเลข
 *              จริงว่า thread นั้น "เบา" กว่า process มากแค่ไหน
 * ============================================================ */
#define _POSIX_C_SOURCE 200809L
#include <stdio.h>
#include <stdlib.h>
#include <time.h>
#include <unistd.h>
#include <pthread.h>
#include <sys/wait.h>

#define NUM_WORKERS 2000

static double elapsed_seconds(struct timespec start, struct timespec end) {
    return (double)(end.tv_sec - start.tv_sec) +
           (double)(end.tv_nsec - start.tv_nsec) / 1e9;
}

void *noop_thread(void *arg) {
    (void)arg;
    return NULL;
}

int main(void) {
    struct timespec t1, t2;

    /* ---------- Benchmark 1: สร้าง thread 2000 ตัว ---------- */
    pthread_t threads[NUM_WORKERS];
    clock_gettime(CLOCK_MONOTONIC, &t1);

    for (int i = 0; i < NUM_WORKERS; i++) {
        if (pthread_create(&threads[i], NULL, noop_thread, NULL) != 0) {
            fprintf(stderr, "pthread_create ล้มเหลว\n");
            exit(EXIT_FAILURE);
        }
    }
    for (int i = 0; i < NUM_WORKERS; i++) {
        pthread_join(threads[i], NULL);
    }

    clock_gettime(CLOCK_MONOTONIC, &t2);
    double thread_time = elapsed_seconds(t1, t2);

    /* ---------- Benchmark 2: fork() process 2000 ตัว ---------- */
    clock_gettime(CLOCK_MONOTONIC, &t1);

    for (int i = 0; i < NUM_WORKERS; i++) {
        pid_t pid = fork();
        if (pid == -1) {
            perror("fork");
            exit(EXIT_FAILURE);
        }
        if (pid == 0) {
            _exit(0); /* child ไม่ต้องทำอะไร แค่จบทันที */
        }
    }
    for (int i = 0; i < NUM_WORKERS; i++) {
        wait(NULL);
    }

    clock_gettime(CLOCK_MONOTONIC, &t2);
    double process_time = elapsed_seconds(t1, t2);

    printf("สร้าง+รอ %d thread   ใช้เวลา: %.4f วินาที\n", NUM_WORKERS, thread_time);
    printf("สร้าง+รอ %d process  ใช้เวลา: %.4f วินาที\n", NUM_WORKERS, process_time);
    printf("process หนักกว่า thread ประมาณ %.1f เท่า\n", process_time / thread_time);

    return 0;
}
```

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 -pthread ex_thread_vs_process_benchmark.c \
    -o ex_thread_vs_process_benchmark
./ex_thread_vs_process_benchmark
```

ผลลัพธ์ตัวอย่างจริงที่วัดได้ (ตัวเลขจะแตกต่างกันไปตามสเปกและภาระของเครื่อง แต่แนวโน้ม
จะเหมือนกันเสมอ คือ process ช้ากว่า thread ชัดเจน):

```
สร้าง+รอ 2000 thread   ใช้เวลา: 0.0650 วินาที
สร้าง+รอ 2000 process  ใช้เวลา: 0.1815 วินาที
process หนักกว่า thread ประมาณ 2.8 เท่า
```

ผลลัพธ์นี้ยืนยันสิ่งที่อธิบายไว้ในตารางเปรียบเทียบของหัวข้อ 31.1 อย่างเป็นรูปธรรม: การ
สร้าง process ต้องมีการเตรียม page table และโครงสร้างข้อมูลของ process ใหม่ทั้งชุด
(แม้จะใช้เทคนิค copy-on-write ช่วยลดงานไปมากแล้วก็ตาม) ในขณะที่การสร้าง thread เป็น
เพียงการจัดสรร stack ใหม่และตั้งค่า register ชุดใหม่ภายใน process เดิม ซึ่งเบากว่ามาก

---

## สรุปท้ายบท

ใน Part นี้เราได้:

- เข้าใจความแตกต่างพื้นฐานที่สุดระหว่าง process กับ thread: thread แชร์ address space
  กันโดยอัตโนมัติ ในขณะที่ process แยกจากกันอย่างสมบูรณ์
- เรียนรู้วิธีคอมไพล์โปรแกรมที่ใช้ pthread ด้วย flag `-pthread` อย่างถูกต้อง
- ใช้ `pthread_create()`/`pthread_join()` สร้างและรอ thread ได้อย่างครบถ้วนทุก
  พารามิเตอร์
- เจอและแก้ไขข้อผิดพลาดคลาสสิกที่สุดของการเขียนโปรแกรมแบบ multithread — การส่ง
  address ของตัวแปร loop counter ให้ thread โดยตรง
- ใช้ `pthread_exit()` ส่งค่ากลับจาก thread อย่างปลอดภัยด้วย heap memory
- เขียนโปรแกรมจริงที่หลาย thread ช่วยกันคำนวณผลรวมของ array และได้เห็นทั้งเวอร์ชัน
  ที่มี race condition และเวอร์ชันที่ออกแบบให้ไม่มี shared mutable state เลย

เราเห็นแล้วว่าปัญหา race condition ที่เจอครั้งแรกใน Part 30 (ระหว่าง process ผ่าน
shared memory) เกิดขึ้นได้ง่ายยิ่งกว่าเดิมระหว่าง thread เพราะ thread แชร์ตัวแปร global
กันโดยอัตโนมัติโดยไม่ต้องตั้งใจทำอะไรเป็นพิเศษเลย ใน **Part 32** เราจะเจาะลึกวิธีป้องกัน
ปัญหานี้อย่างถูกต้องและครบถ้วนด้วยเครื่องมือ synchronization หลักสามตัวของ pthread:
**Mutex** (กุญแจล็อกพื้นฐาน), **Semaphore** (กุญแจที่นับจำนวนได้), และ **Condition
Variable** (กลไกแจ้งเตือนระหว่าง thread) พร้อมทั้งเรียนรู้ปัญหาคลาสสิกอีกตัวที่ตามมาคือ
**Deadlock** และวิธีออกแบบโปรแกรมให้หลีกเลี่ยงมันได้

**ต่อไป:** [Part 32 — Thread Synchronization](./part-032-thread-sync.md)
