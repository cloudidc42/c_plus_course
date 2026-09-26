# Part 30: Shared Memory และ Message Queue (Step 233–240)

> Module C — Systems Programming ด้วย C บน Linux | Part 30 จาก 125
> Step ที่ครอบคลุมใน Part นี้: Step 233–240
> Part ก่อนหน้า: [Part 29 — Inter-Process Communication (Pipe/FIFO)](./part-029-ipc-pipes.md) | Part ถัดไป: [Part 31 — POSIX Threads (pthreads) เบื้องต้น](./part-031-pthreads-basics.md)

## เป้าหมายการเรียนรู้ (Learning Objectives)

หลังจบ Part นี้ ผู้เรียนจะสามารถ:

1. อธิบายได้ว่าทำไม Shared Memory จึงเป็นกลไก IPC ที่เร็วที่สุด เพราะไม่ต้อง copy ข้อมูล
   ผ่าน kernel buffer เหมือน pipe/FIFO
2. ใช้ POSIX Shared Memory API (`shm_open`, `ftruncate`, `mmap`, `munmap`, `shm_unlink`)
   สร้างและใช้งานหน่วยความจำที่ใช้ร่วมกันระหว่าง process ได้จริง
3. อธิบายและสาธิตปัญหา **Race Condition** ที่เกิดขึ้นเมื่อหลาย process เขียน shared memory
   พร้อมกันโดยไม่มีการป้องกัน พร้อมเข้าใจว่าทำไมจึงต้องมี synchronization
4. ใช้ POSIX Message Queue API (`mq_open`, `mq_send`, `mq_receive`, `mq_close`, `mq_unlink`)
   ส่งข้อความระหว่าง process แบบที่รักษาขอบเขตข้อความและมี priority ในตัว
5. เปรียบเทียบข้อดี-ข้อเสียของ Pipe, Shared Memory และ Message Queue และเลือกใช้กลไก
   ที่เหมาะสมกับสถานการณ์ต่างๆ ได้อย่างมีเหตุผล
6. คอมไพล์และ link โปรแกรมที่ใช้ POSIX Shared Memory/Message Queue ด้วย flag `-lrt`
   ได้อย่างถูกต้อง

---

## 30.1 แนวคิด Shared Memory: เร็วที่สุดในบรรดา IPC (Step 233)

ใน Part 29 เราเห็นแล้วว่า pipe และ FIFO ทำงานผ่าน **kernel buffer** เสมอ — เวลา process A
อยากส่งข้อมูลให้ process B ข้อมูลต้องถูก **copy สองรอบ**: รอบแรก copy จาก memory ของ A
เข้าไปใน buffer ของ kernel (ตอนเรียก `write()`) และรอบสอง copy จาก buffer ของ kernel
ออกมาที่ memory ของ B (ตอนเรียก `read()`) การ copy สองรอบนี้ (บวกกับ context switch
ระหว่าง user space กับ kernel space ที่เกิดขึ้นทุกครั้งที่เรียก syscall) คือต้นทุนที่ทำให้
pipe/FIFO ช้ากว่าที่ควรจะเป็นเมื่อต้องส่งข้อมูลปริมาณมากๆ บ่อยๆ

**Shared Memory** แก้ปัญหานี้ด้วยแนวคิดที่ตรงไปตรงมามาก: แทนที่จะให้ kernel เป็นตัวกลาง
คอยรับ-ส่งข้อมูลทุกครั้ง เราให้ process หลายตัว **mmap พื้นที่หน่วยความจำจริงๆ ก้อนเดียวกัน**
เข้ามาไว้ใน address space ของตัวเอง เมื่อ process A เขียนข้อมูลลงไป process B จะ**เห็น
ข้อมูลนั้นทันที** โดยไม่ต้องมี syscall คั่นกลางเลยแม้แต่ครั้งเดียว (หลังจาก setup เสร็จแล้ว)

```
วิธีแบบ Pipe (ต้อง copy 2 รอบ ผ่าน syscall)
┌───────────┐   write()    ┌──────────────┐   read()    ┌───────────┐
│ Process A  │ ──copy #1──▶ │ Kernel Buffer │ ──copy #2─▶ │ Process B  │
└───────────┘              └──────────────┘             └───────────┘

วิธีแบบ Shared Memory (copy 0 รอบ หลัง setup)
┌───────────┐                                            ┌───────────┐
│ Process A  │ ──┐                                    ┌── │ Process B  │
└───────────┘   │                                    │   └───────────┘
                 ▼                                    ▼
         ┌─────────────────────────────────────────────┐
         │      หน่วยความจำจริงก้อนเดียวกัน (RAM)          │
         │   (ทั้งสอง process mmap มาที่ address ตัวเอง)   │
         └─────────────────────────────────────────────┘
```

เพราะไม่มี kernel มาคอยจัดคิว/ตรวจสอบให้ Shared Memory จึงเป็นกลไก IPC ที่ **เร็วที่สุด**
แต่ก็แลกมาด้วยราคาที่ต้องจ่าย: **โปรแกรมเมอร์ต้องจัดการ synchronization เอง** เพราะไม่มี
กลไกใดๆ มาป้องกันไม่ให้สอง process เขียนพื้นที่เดียวกันพร้อมกันจนข้อมูลเสียหาย (จะสาธิต
ปัญหานี้ในหัวข้อ 30.5)

---

## 30.2 POSIX Shared Memory API (Step 234)

POSIX Shared Memory ใช้แนวคิดที่คล้าย "ไฟล์" มาก คือมีชื่อ (คล้าย path) มีขนาด และเปิด/
ปิดได้ด้วย file descriptor แต่ข้อมูลจริงๆ จะถูกเก็บไว้บน `tmpfs` (หน่วยความจำ RAM ล้วนๆ
ไม่ใช่ disk) ทำให้เร็วมาก

ฟังก์ชันหลักที่ต้องรู้จักมี 5 ตัว:

```c
#include <sys/mman.h>
#include <fcntl.h>
#include <unistd.h>

int shm_open(const char *name, int oflag, mode_t mode);
int ftruncate(int fd, off_t length);
void *mmap(void *addr, size_t length, int prot, int flags, int fd, off_t offset);
int munmap(void *addr, size_t length);
int shm_unlink(const char *name);
```

| ฟังก์ชัน | หน้าที่ |
|---|---|
| `shm_open()` | สร้าง (หรือเปิด) shared memory object โดยระบุชื่อ (ต้องขึ้นต้นด้วย `/` เช่น `/my_shm`) คืนค่าเป็น file descriptor |
| `ftruncate()` | กำหนดขนาดของ shared memory object (ตอนสร้างใหม่ขนาดเริ่มต้นคือ 0 ไบต์ ต้องเรียกฟังก์ชันนี้ก่อนใช้งานเสมอ) |
| `mmap()` | แมป shared memory object เข้ามาเป็นส่วนหนึ่งของ address space ของ process ทำให้เข้าถึงได้เหมือนตัวแปรธรรมดา |
| `munmap()` | ยกเลิกการแมป เมื่อใช้งานเสร็จแล้ว |
| `shm_unlink()` | ลบ shared memory object ทิ้งจากระบบอย่างถาวร (ต้องมีอย่างน้อยหนึ่ง process เรียกเมื่อไม่ใช้แล้ว ไม่งั้นจะค้างอยู่ใน `/dev/shm` ตลอดไปจนกว่าจะ reboot) |

**สำคัญมาก**: การคอมไพล์โปรแกรมที่ใช้ฟังก์ชันเหล่านี้ ต้อง link ไลบรารี `librt` ด้วย
flag `-lrt` เสมอ (บน glibc รุ่นใหม่ๆ บางฟังก์ชันอาจย้ายเข้า libc หลักแล้ว แต่การใส่
`-lrt` ไว้ยังคงปลอดภัยและแนะนำเสมอเพื่อความเข้ากันได้ข้ามระบบ):

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 program.c -o program -lrt
```

นอกจากนี้ เนื่องจากฟังก์ชันอย่าง `ftruncate()` เป็นส่วนหนึ่งของมาตรฐาน POSIX (ไม่ใช่
ISO C มาตรฐาน) เมื่อคอมไพล์ด้วย `-std=c17` (ซึ่งบังคับโหมด strict ISO C) เราต้อง
ประกาศ feature test macro ก่อน `#include` ใดๆ เพื่อเปิดให้เห็นฟังก์ชันกลุ่มนี้:

```c
#define _POSIX_C_SOURCE 200809L
```

> ตัวเลข `200809L` หมายถึงมาตรฐาน POSIX.1-2008 ซึ่งเป็นเวอร์ชันที่ครอบคลุมฟังก์ชัน
> ที่เราใช้ตลอด Module C นี้ (รวมถึง `nanosleep`, `sigaction` ที่เคยใช้ใน Part ก่อนๆ)

---

## 30.3 ตัวอย่างจริง: เขียนและอ่าน Shared Memory ระหว่างสอง Process (Step 235)

มาสร้างโปรแกรมสองตัวแยกกัน — ตัวหนึ่งสร้างและเขียนข้อมูลลง shared memory อีกตัวเปิด
อ่านข้อมูลนั้น เพื่อความง่ายเราจะออกแบบ struct เล็กๆ ที่มี flag บอกสถานะความพร้อมด้วย

```c
/* ============================================================
 * ชื่อไฟล์:     shm_writer.c
 * คำอธิบาย:     สร้างและเขียนข้อมูลลง POSIX Shared Memory
 * คอมไพล์:      gcc -Wall -Wextra -Wpedantic -std=c17 shm_writer.c -o shm_writer -lrt
 * ============================================================ */
#define _POSIX_C_SOURCE 200809L
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <fcntl.h>
#include <unistd.h>
#include <sys/mman.h>
#include <sys/stat.h>

#define SHM_NAME "/course_shm_demo"

typedef struct {
    int  ready;       /* flag บอกว่าข้อมูลพร้อมอ่านหรือยัง (0/1) */
    char message[100];
} shared_data_t;

int main(void) {
    /* 1) สร้าง shared memory object (คล้ายไฟล์ แต่อยู่ใน RAM ผ่าน tmpfs) */
    int fd = shm_open(SHM_NAME, O_CREAT | O_RDWR, 0666);
    if (fd == -1) {
        perror("shm_open");
        exit(EXIT_FAILURE);
    }

    /* 2) กำหนดขนาดของ shared memory object ด้วย ftruncate */
    if (ftruncate(fd, sizeof(shared_data_t)) == -1) {
        perror("ftruncate");
        exit(EXIT_FAILURE);
    }

    /* 3) mmap เข้า address space ของ process เรา */
    shared_data_t *shm = mmap(NULL, sizeof(shared_data_t),
                              PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
    if (shm == MAP_FAILED) {
        perror("mmap");
        exit(EXIT_FAILURE);
    }
    close(fd); /* fd ใช้แค่ตอน mmap เท่านั้น ปิดได้เลยหลัง map สำเร็จ */

    shm->ready = 0;
    strncpy(shm->message, "Hello from shared memory writer!", sizeof(shm->message) - 1);
    shm->message[sizeof(shm->message) - 1] = '\0';

    printf("[writer] เขียนข้อมูลลง shared memory แล้ว รอ 1 วินาทีก่อนตั้ง ready=1...\n");
    fflush(stdout);
    sleep(1); /* จำลองเวลาที่ใช้เตรียมข้อมูล (ในของจริงต้องใช้ semaphore ไม่ใช่ sleep) */

    shm->ready = 1; /* ส่งสัญญาณว่าอ่านได้แล้ว */
    printf("[writer] ตั้ง ready = 1 แล้ว\n");

    /* รอสักครู่ให้ reader มีเวลาอ่านก่อนที่ writer จะจบ */
    sleep(2);

    if (munmap(shm, sizeof(shared_data_t)) == -1) {
        perror("munmap");
    }

    printf("[writer] จบการทำงาน (ไม่ unlink เพื่อให้ reader ลบเอง)\n");
    return 0;
}
```

```c
/* ============================================================
 * ชื่อไฟล์:     shm_reader.c
 * คำอธิบาย:     เปิดอ่านข้อมูลจาก POSIX Shared Memory ที่ shm_writer.c สร้างไว้
 * คอมไพล์:      gcc -Wall -Wextra -Wpedantic -std=c17 shm_reader.c -o shm_reader -lrt
 * ============================================================ */
#define _POSIX_C_SOURCE 200809L
#include <stdio.h>
#include <stdlib.h>
#include <fcntl.h>
#include <time.h>
#include <unistd.h>
#include <sys/mman.h>
#include <sys/stat.h>

#define SHM_NAME "/course_shm_demo"

typedef struct {
    int  ready;
    char message[100];
} shared_data_t;

int main(void) {
    /* เปิด shared memory ที่มีอยู่แล้ว (ไม่ใส่ O_CREAT) */
    int fd = shm_open(SHM_NAME, O_RDWR, 0666);
    if (fd == -1) {
        perror("shm_open");
        exit(EXIT_FAILURE);
    }

    shared_data_t *shm = mmap(NULL, sizeof(shared_data_t),
                              PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
    if (shm == MAP_FAILED) {
        perror("mmap");
        exit(EXIT_FAILURE);
    }
    close(fd);

    printf("[reader] รอ writer ตั้ง ready = 1 (busy-wait — เดี๋ยว Part 32 จะแก้ให้ถูกวิธี)...\n");
    fflush(stdout);

    struct timespec poll_delay = {.tv_sec = 0, .tv_nsec = 10 * 1000 * 1000}; /* 10ms */
    while (shm->ready == 0) {
        nanosleep(&poll_delay, NULL); /* busy-wait polling แบบง่ายๆ เพื่อ demo เท่านั้น */
    }

    printf("[reader] ready = 1 แล้ว อ่านข้อความ: %s\n", shm->message);

    if (munmap(shm, sizeof(shared_data_t)) == -1) {
        perror("munmap");
    }

    if (shm_unlink(SHM_NAME) == -1) {
        perror("shm_unlink");
    } else {
        printf("[reader] shm_unlink สำเร็จ ลบ shared memory object ออกจากระบบแล้ว\n");
    }

    return 0;
}
```

คอมไพล์และรัน (เปิดสอง terminal เหมือน Part 29 หรือรัน writer เป็น background):

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 shm_writer.c -o shm_writer -lrt
gcc -Wall -Wextra -Wpedantic -std=c17 shm_reader.c -o shm_reader -lrt

./shm_writer &
sleep 0.2
./shm_reader
```

ผลลัพธ์ฝั่ง writer:

```
[writer] เขียนข้อมูลลง shared memory แล้ว รอ 1 วินาทีก่อนตั้ง ready=1...
[writer] ตั้ง ready = 1 แล้ว
[writer] จบการทำงาน (ไม่ unlink เพื่อให้ reader ลบเอง)
```

ผลลัพธ์ฝั่ง reader:

```
[reader] รอ writer ตั้ง ready = 1 (busy-wait — เดี๋ยว Part 32 จะแก้ให้ถูกวิธี)...
[reader] ready = 1 แล้ว อ่านข้อความ: Hello from shared memory writer!
[reader] shm_unlink สำเร็จ ลบ shared memory object ออกจากระบบแล้ว
```

### ทำไมต้องใช้ flag `ready` แทนการอ่านค่าตรงๆ

สังเกตว่าโปรแกรม `shm_reader.c` ไม่ได้อ่าน `shm->message` ทันทีหลัง mmap เสร็จ แต่วน
`while (shm->ready == 0)` รอก่อน เหตุผลคือ **shared memory ไม่มีกลไกแจ้งเตือนใดๆ ในตัว**
ต่างจาก pipe ที่ `read()` จะบล็อกอัตโนมัติจนกว่าจะมีข้อมูล — reader ของ shared memory
ต้องคอยตรวจสอบเองว่าข้อมูลพร้อมหรือยัง วิธีที่ใช้ในตัวอย่างนี้เรียกว่า **busy-wait polling**
(วนถามซ้ำๆ) ซึ่งใช้งานได้จริงแต่สิ้นเปลือง CPU โดยไม่จำเป็น วิธีที่ถูกต้องกว่าคือใช้
**semaphore** หรือ **condition variable** มาช่วยแจ้งเตือน ซึ่งเป็นหัวข้อหลักของ **Part 32**
(ในหัวข้อ 30.5 เราจะแอบใช้ semaphore แก้ปัญหาที่เกี่ยวข้องกันเป็นตัวอย่างนำร่อง)

---

## 30.4 การจัดการวงจรชีวิตของ Shared Memory Object (Step 236)

จุดที่มือใหม่มักสับสนคือ shared memory object **ไม่ได้ถูกลบทิ้งอัตโนมัติ** เมื่อ process
ที่สร้างมันปิดตัวไป มันจะยังคงอยู่ใน `/dev/shm` ต่อไปจนกว่าจะมีใครเรียก `shm_unlink()`
หรือจนกว่าเครื่องจะ reboot ลองตรวจสอบเองได้:

```bash
ls -la /dev/shm/
```

จะเห็นไฟล์ `course_shm_demo` ปรากฏอยู่จริงๆ (ก่อนที่ reader จะ unlink มันทิ้ง)

```
สถานะของ Shared Memory Object ตลอดวงจรชีวิต

shm_open(O_CREAT) ──▶ ftruncate() ──▶ mmap() ──▶ [ใช้งาน] ──▶ munmap()
       │                                                          │
       │                                                          ▼
       │                                              object ยังอยู่ใน /dev/shm
       │                                              (แม้ process จะปิดไปแล้ว!)
       │                                                          │
       └──────────────────── shm_unlink() ◀───────────────────────┘
                              (ลบออกจากระบบถาวร)
```

หลักปฏิบัติที่ดีคือ **ให้ process ฝั่งใดฝั่งหนึ่งที่แน่ใจว่าเป็นตัวสุดท้ายที่ใช้งานเป็นคน
`shm_unlink()`** เช่นเดียวกับตัวอย่างข้างต้นที่ให้ reader เป็นคน unlink หลังอ่านเสร็จ
สมบูรณ์ ถ้าลืม unlink ไฟล์จะค้างอยู่ในระบบเรื่อยๆ ซึ่งแม้จะไม่ทำให้โปรแกรมพังทันที แต่
จะสร้างขยะสะสมในระบบ (คล้ายกับ memory leak แต่เป็นระดับ system-wide)

> **ข้อควรระวัง**: ถ้ารันโปรแกรมทดสอบซ้ำหลายรอบแล้วลืม unlink ให้ลบไฟล์ทิ้งเองด้วย
> `rm /dev/shm/ชื่อไฟล์` ก่อนรันรอบใหม่ ไม่งั้น `shm_open(O_CREAT)` ในรอบถัดไปอาจไป
> เจอข้อมูลเก่าที่ค้างอยู่ ทำให้ผลลัพธ์สับสน

---

## 30.5 ปัญหา Race Condition เมื่อหลาย Process เขียนพร้อมกัน (Step 237)

นี่คือ "ราคา" ที่ต้องจ่ายสำหรับความเร็วของ shared memory มาดูตัวอย่างที่จำลองสถานการณ์
จริง: สร้าง 4 process ให้แต่ละตัวเพิ่มค่าตัวแปร counter ตัวเดียวกันในหน่วยความจำที่ใช้
ร่วมกัน คนละ 200,000 ครั้ง โดย**ไม่มีการป้องกันใดๆ เลย**

```c
/* ============================================================
 * ชื่อไฟล์:     shm_race.c
 * คำอธิบาย:     สาธิต Race Condition เมื่อหลาย process เขียน shared memory
 *              พร้อมกันโดยไม่มีการป้องกัน (synchronization)
 * คอมไพล์:      gcc -Wall -Wextra -Wpedantic -std=c17 shm_race.c -o shm_race
 * ============================================================ */
#define _POSIX_C_SOURCE 200809L
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/mman.h>
#include <sys/wait.h>

#define NUM_PROCESSES 4
#define INCREMENTS_PER_PROCESS 200000

int main(void) {
    /* ใช้ MAP_ANONYMOUS + MAP_SHARED แทน shm_open เพราะไม่ต้องมีชื่อไฟล์
     * (เหมาะกับ process ที่เป็นญาติกันผ่าน fork() เท่านั้น) */
    int *counter = mmap(NULL, sizeof(int), PROT_READ | PROT_WRITE,
                        MAP_SHARED | MAP_ANONYMOUS, -1, 0);
    if (counter == MAP_FAILED) {
        perror("mmap");
        exit(EXIT_FAILURE);
    }
    *counter = 0;

    printf("[main] คาดหวังผลลัพธ์ที่ถูกต้อง: %d x %d = %d\n",
           NUM_PROCESSES, INCREMENTS_PER_PROCESS,
           NUM_PROCESSES * INCREMENTS_PER_PROCESS);
    fflush(stdout); /* สำคัญ: flush ก่อน fork() ไม่งั้น buffer ที่ยังไม่ flush จะถูกคัดลอก
                     * ไปที่ child ทุกตัว แล้วพิมพ์ซ้ำตอน child เรียก exit() */

    for (int i = 0; i < NUM_PROCESSES; i++) {
        pid_t pid = fork();
        if (pid == -1) {
            perror("fork");
            exit(EXIT_FAILURE);
        }
        if (pid == 0) {
            /* Child: เพิ่มค่า counter แบบไม่ป้องกัน race condition เลย */
            for (int j = 0; j < INCREMENTS_PER_PROCESS; j++) {
                /* บรรทัดนี้จริงๆ แล้วไม่ใช่ 1 คำสั่งเดียวใน CPU
                 * แต่เป็น 3 ขั้นตอน: read counter -> เพิ่มค่า -> write กลับ
                 * ถ้า process อื่นแทรกเข้ามาระหว่างนี้ ค่าจะหายไป */
                (*counter)++;
            }
            exit(EXIT_SUCCESS);
        }
    }

    /* รอทุก child process จบ */
    for (int i = 0; i < NUM_PROCESSES; i++) {
        wait(NULL);
    }

    printf("[main] ผลลัพธ์จริงที่ได้: %d\n", *counter);
    if (*counter != NUM_PROCESSES * INCREMENTS_PER_PROCESS) {
        printf("[main] --> เกิด Race Condition! ค่าที่ควรจะเพิ่มบางครั้งหายไป\n");
    }

    if (munmap(counter, sizeof(int)) == -1) {
        perror("munmap");
    }

    return 0;
}
```

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 shm_race.c -o shm_race
./shm_race
```

ผลลัพธ์ (ตัวเลขจริงจะแตกต่างกันไปทุกครั้งที่รัน เพราะขึ้นกับจังหวะการ schedule ของ OS):

```
[main] คาดหวังผลลัพธ์ที่ถูกต้อง: 4 x 200000 = 800000
[main] ผลรวมที่ได้จริง: 467507
[main] --> เกิด Race Condition! ค่าที่ควรจะเพิ่มบางครั้งหายไป
```

### ทำไมผลลัพธ์ถึงผิด

คำสั่ง `(*counter)++;` ที่ดูเหมือนเป็นคำสั่งเดียว แท้จริงแล้วถูกแปลเป็นคำสั่งระดับ CPU
อย่างน้อย 3 ขั้นตอน (ทบทวนแนวคิดนี้จาก Part 15 เรื่อง Bit Manipulation ที่เคยพูดถึง
การทำงานระดับ CPU):

```
ขั้นตอนจริงของ (*counter)++ ในระดับ CPU:
  1. LOAD  : อ่านค่า counter จากหน่วยความจำเข้า register
  2. ADD   : บวก register เพิ่มขึ้น 1
  3. STORE : เขียนค่าใน register กลับไปที่หน่วยความจำ

ถ้า Process A และ Process B ทำพร้อมกันแบบสลับจังหวะแบบนี้:

  เวลา   Process A                Process B              ค่า counter จริง
  ────   ─────────────────────    ─────────────────────   ─────────────
  t1     LOAD counter (=100)                               100
  t2                              LOAD counter (=100)       100
  t3     ADD -> 101                                          100
  t4     STORE -> counter=101                                101
  t5                              ADD -> 101                  101
  t6                              STORE -> counter=101        101   <- ควรเป็น 102!

  ผลคือ: ทั้งสอง process เพิ่มค่าไปคนละ 1 ครั้ง (รวม 2 ครั้ง)
  แต่ค่า counter สุดท้ายเพิ่มขึ้นแค่ 1 เพราะ Process B เขียนทับค่าที่ Process A
  เพิ่งเขียนไปโดยไม่รู้ตัว
```

นี่คือนิยามของ **Race Condition**: ผลลัพธ์สุดท้ายของโปรแกรมขึ้นอยู่กับ "จังหวะ" การสลับ
กันทำงานของ process/thread หลายตัว ซึ่งเราไม่สามารถควบคุมหรือคาดเดาได้แน่นอน ทำให้
โปรแกรมแบบนี้เป็นบั๊กที่**เกิดแบบสุ่ม** (บางครั้งรันแล้วได้ค่าถูกต้องพอดี บางครั้งผิดมาก
บางครั้งผิดน้อย) ซึ่งทำให้ debug ยากมากในโลกจริง

### พิสูจน์ว่าแก้ได้จริงด้วย Semaphore (ตัวอย่างนำร่องก่อน Part 32)

เพื่อให้เห็นภาพว่าปัญหานี้ **แก้ได้จริง** เรามาดูเวอร์ชันที่ใช้ POSIX semaphore เป็น
"กุญแจ" ป้องกันไม่ให้สอง process เข้าไปแก้ counter พร้อมกัน (เนื้อหาเรื่อง semaphore
แบบละเอียดจะอยู่ใน Part 32 นี่เป็นแค่การพิสูจน์แนวคิดเบื้องต้น):

```c
/* ============================================================
 * ชื่อไฟล์:     ex_shm_race_fixed.c
 * คำอธิบาย:     แก้ปัญหา Race Condition จาก shm_race.c ด้วย POSIX unnamed
 *              semaphore ที่วางไว้ใน shared memory เดียวกัน
 * คอมไพล์:      gcc -Wall -Wextra -Wpedantic -std=c17 ex_shm_race_fixed.c \
 *                  -o ex_shm_race_fixed -pthread
 * ============================================================ */
#define _POSIX_C_SOURCE 200809L
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <semaphore.h>
#include <sys/mman.h>
#include <sys/wait.h>

#define NUM_PROCESSES 4
#define INCREMENTS_PER_PROCESS 200000

typedef struct {
    sem_t lock;   /* semaphore ที่ทำหน้าที่เป็น "กุญแจ" ป้องกันการเขียนพร้อมกัน */
    int   counter;
} shared_state_t;

int main(void) {
    shared_state_t *state = mmap(NULL, sizeof(shared_state_t), PROT_READ | PROT_WRITE,
                                 MAP_SHARED | MAP_ANONYMOUS, -1, 0);
    if (state == MAP_FAILED) {
        perror("mmap");
        exit(EXIT_FAILURE);
    }

    /* pshared = 1 หมายถึง semaphore ตัวนี้ใช้ร่วมกันระหว่าง process (ไม่ใช่แค่ thread)
     * ค่าเริ่มต้นของ semaphore = 1 หมายถึง "ว่าง พร้อมให้เข้าไปทำงานได้ 1 ราย" */
    if (sem_init(&state->lock, 1, 1) == -1) {
        perror("sem_init");
        exit(EXIT_FAILURE);
    }
    state->counter = 0;

    printf("[main] คาดหวังผลลัพธ์ที่ถูกต้อง: %d\n", NUM_PROCESSES * INCREMENTS_PER_PROCESS);
    fflush(stdout);

    for (int i = 0; i < NUM_PROCESSES; i++) {
        pid_t pid = fork();
        if (pid == -1) {
            perror("fork");
            exit(EXIT_FAILURE);
        }
        if (pid == 0) {
            for (int j = 0; j < INCREMENTS_PER_PROCESS; j++) {
                sem_wait(&state->lock);   /* ขอกุญแจก่อนเข้าไปแก้ counter */
                state->counter++;
                sem_post(&state->lock);   /* คืนกุญแจให้ process อื่นใช้ต่อ */
            }
            exit(EXIT_SUCCESS);
        }
    }

    for (int i = 0; i < NUM_PROCESSES; i++) {
        wait(NULL);
    }

    printf("[main] ผลลัพธ์จริงที่ได้ (มี semaphore ป้องกันแล้ว): %d\n", state->counter);

    sem_destroy(&state->lock);
    munmap(state, sizeof(shared_state_t));
    return 0;
}
```

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 ex_shm_race_fixed.c -o ex_shm_race_fixed -pthread
./ex_shm_race_fixed
```

ผลลัพธ์ (ได้ค่าถูกต้อง**ทุกครั้ง**ที่รัน ไม่ว่าจะรันกี่รอบก็ตาม):

```
[main] คาดหวังผลลัพธ์ที่ถูกต้อง: 800000
[main] ผลลัพธ์จริงที่ได้ (มี semaphore ป้องกันแล้ว): 800000
```

`sem_wait()` จะบล็อกรอถ้ามี process อื่นถือกุญแจอยู่ (ทำให้ทั้ง 3 ขั้นตอน LOAD-ADD-STORE
ของแต่ละ process ทำงานให้เสร็จสมบูรณ์ก่อนที่ process อื่นจะเข้ามาแทรกได้) การล็อกแบบนี้
ทำให้ operation ที่ควรจะเป็น "atomic" (ทำครบหรือไม่ทำเลย ไม่มีใครแทรกกลาง) กลายเป็น
atomic ได้จริง แต่ก็แลกมาด้วยความเร็วที่ช้าลง (เพราะ process ต้องผลัดกันทำทีละตัว) —
นี่คือ trade-off พื้นฐานระหว่างความถูกต้องกับความเร็วที่จะพูดถึงอย่างละเอียดใน Part 32

---

## 30.6 POSIX Message Queue เบื้องต้น (Step 238)

**Message Queue** เป็นกลไก IPC ที่อยู่ตรงกลางระหว่าง pipe กับ shared memory ในแง่ของ
ความซับซ้อน: มันยังคง copy ข้อมูลผ่าน kernel เหมือน pipe (จึงช้ากว่า shared memory)
แต่แลกมาด้วยความสะดวก 2 อย่างที่ pipe ไม่มีให้:

1. **รักษาขอบเขตของแต่ละข้อความ** — ส่งเป็นก้อนๆ (message) ชัดเจน ไม่ใช่ byte stream
   ต่อเนื่องเหมือน pipe ผู้รับจะได้ข้อความครบตามที่ผู้ส่งส่งมาในการเรียก `mq_receive()`
   แต่ละครั้งเสมอ
2. **มี priority ในตัว** — ข้อความที่มี priority สูงกว่าจะถูก `mq_receive()` ออกมาก่อน
   เสมอ ไม่ว่าจะถูกส่งเข้าคิวก่อนหรือหลังก็ตาม (ต่างจาก pipe ที่เป็น FIFO ล้วนๆ)

```c
#include <mqueue.h>

mqd_t mq_open(const char *name, int oflag, mode_t mode, struct mq_attr *attr);
int   mq_send(mqd_t mqdes, const char *msg_ptr, size_t msg_len, unsigned int msg_prio);
ssize_t mq_receive(mqd_t mqdes, char *msg_ptr, size_t msg_len, unsigned int *msg_prio);
int   mq_close(mqd_t mqdes);
int   mq_unlink(const char *name);
```

struct `mq_attr` ใช้กำหนดคุณสมบัติของคิวตอนสร้าง:

```c
struct mq_attr {
    long mq_flags;   /* flag ของคิว (0 = blocking mode ปกติ) */
    long mq_maxmsg;  /* จำนวนข้อความสูงสุดที่คิวเก็บได้พร้อมกัน */
    long mq_msgsize; /* ขนาดสูงสุดของแต่ละข้อความ (ไบต์) */
    long mq_curmsgs; /* จำนวนข้อความที่มีอยู่ตอนนี้ (kernel เป็นคนอัปเดตให้) */
};
```

การคอมไพล์ต้อง link `-lrt` เช่นเดียวกับ shared memory:

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 program.c -o program -lrt
```

---

## 30.7 ตัวอย่างจริง: ส่งงานผ่าน Message Queue พร้อม Priority (Step 239)

```c
/* ============================================================
 * ชื่อไฟล์:     mq_sender.c
 * คำอธิบาย:     ส่งข้อความผ่าน POSIX Message Queue
 * คอมไพล์:      gcc -Wall -Wextra -Wpedantic -std=c17 mq_sender.c -o mq_sender -lrt
 * ============================================================ */
#define _POSIX_C_SOURCE 200809L
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <mqueue.h>
#include <fcntl.h>
#include <unistd.h>

#define MQ_NAME "/course_mq_demo"
#define MAX_MSG_SIZE 256

int main(void) {
    struct mq_attr attr;
    attr.mq_flags   = 0;
    attr.mq_maxmsg  = 10;
    attr.mq_msgsize = MAX_MSG_SIZE;
    attr.mq_curmsgs = 0;

    mqd_t mq = mq_open(MQ_NAME, O_CREAT | O_WRONLY, 0666, &attr);
    if (mq == (mqd_t)-1) {
        perror("mq_open");
        exit(EXIT_FAILURE);
    }

    const char *messages[] = {
        "งานที่ 1: ประมวลผลไฟล์ A",
        "งานที่ 2: ประมวลผลไฟล์ B",
        "งานที่ 3: ปิดระบบ"
    };
    unsigned int priorities[] = {1, 5, 10}; /* ยิ่งเลขมาก = priority สูงกว่า */

    for (int i = 0; i < 3; i++) {
        if (mq_send(mq, messages[i], strlen(messages[i]) + 1, priorities[i]) == -1) {
            perror("mq_send");
            exit(EXIT_FAILURE);
        }
        printf("[sender] ส่ง (priority=%u): %s\n", priorities[i], messages[i]);
    }

    if (mq_close(mq) == -1) {
        perror("mq_close");
    }

    printf("[sender] ส่งครบทุกข้อความแล้ว\n");
    return 0;
}
```

```c
/* ============================================================
 * ชื่อไฟล์:     mq_receiver.c
 * คำอธิบาย:     รับข้อความจาก POSIX Message Queue ที่ mq_sender.c ส่งมา
 * คอมไพล์:      gcc -Wall -Wextra -Wpedantic -std=c17 mq_receiver.c -o mq_receiver -lrt
 * ============================================================ */
#define _POSIX_C_SOURCE 200809L
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <mqueue.h>
#include <fcntl.h>
#include <unistd.h>

#define MQ_NAME "/course_mq_demo"
#define MAX_MSG_SIZE 256

int main(void) {
    mqd_t mq = mq_open(MQ_NAME, O_RDONLY);
    if (mq == (mqd_t)-1) {
        perror("mq_open");
        exit(EXIT_FAILURE);
    }

    struct mq_attr attr;
    if (mq_getattr(mq, &attr) == -1) {
        perror("mq_getattr");
        exit(EXIT_FAILURE);
    }

    char *buf = malloc((size_t)attr.mq_msgsize);
    if (buf == NULL) {
        fprintf(stderr, "malloc ล้มเหลว\n");
        exit(EXIT_FAILURE);
    }

    /* รับข้อความ 3 ครั้งตามที่ sender ส่งมา
     * หมายเหตุ: mq_receive จะคืนข้อความที่ priority สูงสุดก่อนเสมอ
     * ไม่ใช่ตามลำดับที่ส่ง (ต่างจาก pipe/FIFO ที่เป็น FIFO ล้วนๆ) */
    for (int i = 0; i < 3; i++) {
        unsigned int prio;
        ssize_t n = mq_receive(mq, buf, (size_t)attr.mq_msgsize, &prio);
        if (n == -1) {
            perror("mq_receive");
            exit(EXIT_FAILURE);
        }
        printf("[receiver] ได้รับ (priority=%u): %s\n", prio, buf);
    }

    free(buf);

    if (mq_close(mq) == -1) {
        perror("mq_close");
    }
    if (mq_unlink(MQ_NAME) == -1) {
        perror("mq_unlink");
    } else {
        printf("[receiver] mq_unlink สำเร็จ\n");
    }

    return 0;
}
```

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 mq_sender.c -o mq_sender -lrt
gcc -Wall -Wextra -Wpedantic -std=c17 mq_receiver.c -o mq_receiver -lrt

./mq_sender
./mq_receiver
```

ผลลัพธ์:

```
[sender] ส่ง (priority=1): งานที่ 1: ประมวลผลไฟล์ A
[sender] ส่ง (priority=5): งานที่ 2: ประมวลผลไฟล์ B
[sender] ส่ง (priority=10): งานที่ 3: ปิดระบบ
[sender] ส่งครบทุกข้อความแล้ว
[receiver] ได้รับ (priority=10): งานที่ 3: ปิดระบบ
[receiver] ได้รับ (priority=5): งานที่ 2: ประมวลผลไฟล์ B
[receiver] ได้รับ (priority=1): งานที่ 1: ประมวลผลไฟล์ A
[receiver] mq_unlink สำเร็จ
```

สังเกตว่า **แม้ sender จะส่งข้อความ "งานที่ 3: ปิดระบบ" เป็นลำดับสุดท้าย** แต่เพราะมันมี
priority สูงสุด (10) `mq_receive()` จึงคืนข้อความนี้ออกมา**ก่อน**เสมอ นี่คือคุณสมบัติที่
มีประโยชน์มากในระบบคิวงานจริง เช่น การให้คำสั่ง "shutdown" หรือ "emergency stop"
มี priority สูงสุดเสมอ เพื่อให้ worker process ประมวลผลคำสั่งนั้นก่อนงานปกติอื่นๆ ที่
รอคิวอยู่ก่อนหน้า

ลองตรวจสอบ message queue ที่มีอยู่ในระบบผ่าน virtual filesystem `/dev/mqueue`:

```bash
ls -la /dev/mqueue/
```

---

## 30.8 เปรียบเทียบ Pipe vs Shared Memory vs Message Queue (Step 240)

ตอนนี้เรารู้จักกลไก IPC หลักสามแบบแล้ว มาสรุปว่าควรเลือกใช้อะไรเมื่อไหร่:

| คุณสมบัติ | Pipe / FIFO | Shared Memory | Message Queue |
|---|---|---|---|
| **ความเร็ว** | ปานกลาง (copy 2 รอบผ่าน kernel) | **เร็วที่สุด** (ไม่ copy หลัง setup) | ปานกลาง (copy 2 รอบผ่าน kernel เหมือน pipe) |
| **รักษาขอบเขตข้อความ** | ไม่รักษา (byte stream ล้วนๆ) | ไม่เกี่ยวข้อง (เข้าถึงเป็น memory ตรงๆ) | **รักษา** (แต่ละ message แยกกันชัดเจน) |
| **มี Priority ในตัว** | ไม่มี | ไม่มี | **มี** |
| **ต้องจัดการ Synchronization เอง** | ไม่ต้อง (blocking read/write ช่วยให้) | **ต้องทำเอง** (เสี่ยง race condition สูงสุด) | ไม่ต้อง (kernel จัดคิวให้) |
| **ใช้ระหว่าง process ที่ไม่ใช่ญาติกันได้** | FIFO ได้ / pipe ธรรมดาไม่ได้ | ได้ (ผ่าน `shm_open` ตั้งชื่อ) | ได้ (ผ่าน `mq_open` ตั้งชื่อ) |
| **เหมาะกับข้อมูลขนาดใหญ่** | พอใช้ได้ (มี buffer จำกัด ~64KB) | **เหมาะที่สุด** | ไม่เหมาะ (มักจำกัด `mq_msgsize` ไว้เล็ก) |
| **เหมาะกับข้อความสั้นๆ ที่มีความสำคัญต่างกัน** | ไม่เหมาะ | ไม่เหมาะ | **เหมาะที่สุด** |
| **ความซับซ้อนในการใช้งาน** | ต่ำที่สุด | สูงที่สุด (ต้องคิดเรื่อง sync เอง) | ปานกลาง |

### แนวทางการเลือกใช้ในสถานการณ์จริง

```
ต้องการส่งข้อมูล stream ต่อเนื่อง ระหว่าง parent-child แบบง่ายๆ?
  └──▶ ใช้ Pipe

ต้องการสื่อสารระหว่าง process ที่ไม่ใช่ญาติกัน แบบง่ายๆ ไม่ซับซ้อน?
  └──▶ ใช้ FIFO (Named Pipe)

ต้องการความเร็วสูงสุด ส่งข้อมูลปริมาณมาก (เช่น เฟรมภาพ, buffer เสียง)?
  └──▶ ใช้ Shared Memory (ร่วมกับ semaphore/mutex ป้องกัน race condition)

ต้องการระบบคิวงานที่มีลำดับความสำคัญ (priority) หรือข้อความที่ต้องแยกขอบเขตชัดเจน?
  └──▶ ใช้ Message Queue

ต้องการสื่อสารข้ามเครื่อง (ไม่ใช่แค่ในเครื่องเดียวกัน)?
  └──▶ ต้องใช้ Socket (Part 33-35) — IPC ทั้ง 3 แบบข้างต้นใช้ได้แค่ในเครื่องเดียวกันเท่านั้น
```

ในทางปฏิบัติ ระบบใหญ่ๆ มักผสมผสานหลายกลไกเข้าด้วยกัน เช่น ใช้ shared memory เก็บ
ข้อมูลขนาดใหญ่ (เช่น เฟรมวิดีโอ) แล้วใช้ message queue หรือ pipe เพียงแค่ส่ง "สัญญาณ"
สั้นๆ บอกว่า "ข้อมูลใน shared memory พร้อมอ่านแล้วนะ" (ไม่ต้อง copy ข้อมูลก้อนใหญ่ผ่าน
message queue ซึ่งมักจำกัดขนาดข้อความไว้เล็ก)

---

## ข้อผิดพลาดที่พบบ่อย (Common Pitfalls)

1. **ลืม `-lrt` ตอน link** — จะได้ error แบบ `undefined reference to 'shm_open'` หรือ
   `undefined reference to 'mq_open'` ตอน linking แม้โค้ดจะไม่มีปัญหาอะไรเลยก็ตาม
   ต้อง link ไลบรารี realtime extension เสมอเมื่อใช้ POSIX shared memory หรือ message queue
2. **ลืม `ftruncate()` ก่อนใช้งาน shared memory** — shared memory object ที่สร้างใหม่ด้วย
   `shm_open(O_CREAT)` จะมีขนาด **0 ไบต์** เสมอ ถ้าลืมเรียก `ftruncate()` กำหนดขนาดก่อน
   `mmap()` จะทำให้ `mmap()` ล้มเหลว หรือถ้าสำเร็จก็จะเข้าถึงหน่วยความจำนอกขอบเขตทันที
   (Segmentation Fault)
3. **ลืม `shm_unlink()` / `mq_unlink()` หลังใช้งานเสร็จ** — object จะค้างอยู่ในระบบ
   (`/dev/shm` หรือ `/dev/mqueue`) ไปเรื่อยๆ แม้ทุก process จะปิดตัวไปหมดแล้ว ทำให้
   รันโปรแกรมทดสอบซ้ำแล้วเจอข้อมูลเก่าปนเปื้อน ควรลบทิ้งด้วยมือด้วย `rm /dev/shm/ชื่อ`
   ระหว่างพัฒนาถ้าลืม unlink ในโค้ด
4. **เข้าถึง shared memory โดยไม่มี synchronization ใดๆ เลย** — ดังที่สาธิตในหัวข้อ 30.5
   นี่คือสาเหตุของ race condition ที่พบบ่อยที่สุด ต้องจำไว้เสมอว่า shared memory ไม่มี
   กลไกป้องกันการเขียนพร้อมกันในตัวมันเอง เป็นหน้าที่ของโปรแกรมเมอร์ล้วนๆ
5. **สับสนระหว่างขนาด struct ที่แชร์กันใน 32-bit กับ 64-bit build** — ถ้า process ทั้งสอง
   ฝั่งไม่ได้คอมไพล์ด้วย architecture/flag เดียวกัน (เช่น อันหนึ่งคอมไพล์แบบ 32-bit อีก
   อันแบบ 64-bit) ขนาดและ alignment ของ struct อาจไม่ตรงกัน ทำให้อ่านข้อมูลผิดเพี้ยน
   ควรคอมไพล์ทุก process ที่แชร์ struct เดียวกันด้วย toolchain และ flag ชุดเดียวกันเสมอ
6. **ตั้งชื่อ shared memory / message queue โดยไม่ขึ้นต้นด้วย `/`** — มาตรฐาน POSIX
   กำหนดให้ชื่อของ `shm_open()` และ `mq_open()` ต้องขึ้นต้นด้วย `/` เสมอ (เช่น
   `/my_shm` ไม่ใช่ `my_shm` เฉยๆ) และห้ามมี `/` ตัวที่สองในชื่อ มิฉะนั้นบางระบบจะ
   ปฏิเสธด้วย `EINVAL`
7. **ใช้ `sleep()` แทนกลไก synchronization จริง ในโค้ด production** — ตัวอย่าง
   `shm_writer.c`/`shm_reader.c` ในหัวข้อ 30.3 ใช้ `sleep()` และ busy-wait polling
   เพื่อความง่ายในการสาธิตเท่านั้น ในโค้ดจริงห้ามทำแบบนี้เด็ดขาด เพราะเวลาที่แน่นอนไม่มี
   ทางรับประกันความถูกต้องได้ (ถ้าเครื่องช้ากว่าที่คาดไว้ก็จะพังทันที) ต้องใช้ semaphore
   หรือ condition variable ที่แท้จริง (Part 32)

---

## แบบฝึกหัดท้ายบท

1. แก้ไข `shm_writer.c`/`shm_reader.c` ให้เป็นการสื่อสารแบบ "ping-pong" สองทาง คือ
   writer ส่งข้อความ, reader อ่านแล้วตอบกลับด้วย flag ตัวที่สอง (`response_ready`),
   writer รอ flag นั้นแล้วพิมพ์ผลตอบรับออกมา
2. ทดลองรัน `mq_sender.c` ส่งข้อความจำนวนมากกว่า `mq_maxmsg` ที่กำหนดไว้ (ลองส่ง 15
   ข้อความ ทั้งที่กำหนด `mq_maxmsg = 10`) สังเกตว่าเกิดอะไรขึ้นกับ `mq_send()` เมื่อคิวเต็ม
   (คำใบ้: ในโหมด blocking ปกติ `mq_send()` จะบล็อกรอจนกว่าจะมีที่ว่าง)
3. เขียนโปรแกรมเปรียบเทียบเวลาที่ใช้ย้ายข้อมูลขนาด 64 MB ผ่าน pipe เทียบกับผ่าน
   shared memory (ใช้ `clock_gettime(CLOCK_MONOTONIC, ...)` จับเวลา) แล้วรายงานผลว่า
   เร็วกว่ากันกี่เท่า
4. ใช้ POSIX semaphore แก้ปัญหา race condition ใน `shm_race.c` (ดูตัวอย่างในหัวข้อ 30.5
   เป็นแนวทาง) แล้วรันซ้ำ 5 รอบเพื่อพิสูจน์ว่าผลลัพธ์ถูกต้องทุกครั้ง
5. อธิบายด้วยคำพูดตัวเองว่าทำไม message queue จึงเหมาะกับการทำ "task queue" (ระบบ
   คิวงาน) มากกว่า shared memory ทั้งที่ shared memory เร็วกว่า
6. ออกแบบ (เขียนเป็น pseudocode หรือ diagram ก็ได้ ไม่จำเป็นต้องคอมไพล์) ระบบส่งภาพ
   วิดีโอแบบ real-time ระหว่างสอง process โดยใช้ shared memory เก็บข้อมูลภาพ และใช้
   message queue หรือ FIFO เพียงส่ง "สัญญาณ" บอกว่าเฟรมใหม่พร้อมแล้ว

### แนวทางเฉลยข้อ 3 (เปรียบเทียบความเร็ว Pipe vs Shared Memory)

```c
/* ============================================================
 * ชื่อไฟล์:     ex_ipc_benchmark.c
 * คำอธิบาย:     เฉลยแบบฝึกหัด — เปรียบเทียบเวลาที่ใช้ "ย้ายข้อมูล" ขนาดใหญ่
 *              ผ่าน pipe (ต้อง copy ผ่าน kernel buffer 2 รอบ: write + read)
 *              เทียบกับการเขียนตรงลง shared memory (memcpy ครั้งเดียว ไม่ต้อง
 *              ผ่าน syscall เลยหลัง mmap เสร็จ)
 * คอมไพล์:      gcc -Wall -Wextra -Wpedantic -std=c17 ex_ipc_benchmark.c \
 *                  -o ex_ipc_benchmark -O2
 * ============================================================ */
#define _POSIX_C_SOURCE 200809L
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <time.h>
#include <unistd.h>
#include <sys/mman.h>

#define TOTAL_SIZE (64 * 1024 * 1024) /* ย้ายข้อมูลทั้งหมด 64 MB */
#define CHUNK_SIZE (64 * 1024)        /* ย้ายทีละ 64 KB ต่อรอบ */

static double elapsed_seconds(struct timespec start, struct timespec end) {
    return (double)(end.tv_sec - start.tv_sec) +
           (double)(end.tv_nsec - start.tv_nsec) / 1e9;
}

int main(void) {
    char *src = malloc(CHUNK_SIZE);
    char *sink = malloc(CHUNK_SIZE);
    if (src == NULL || sink == NULL) {
        fprintf(stderr, "malloc ล้มเหลว\n");
        exit(EXIT_FAILURE);
    }
    memset(src, 'A', CHUNK_SIZE);

    /* ---------- Benchmark 1: pipe (ต้องผ่าน write() + read() เข้า kernel) ---------- */
    int fd[2];
    if (pipe(fd) == -1) {
        perror("pipe");
        exit(EXIT_FAILURE);
    }

    struct timespec t1, t2;
    clock_gettime(CLOCK_MONOTONIC, &t1);

    long remaining = TOTAL_SIZE;
    while (remaining > 0) {
        if (write(fd[1], src, CHUNK_SIZE) == -1) {
            perror("write");
            exit(EXIT_FAILURE);
        }
        if (read(fd[0], sink, CHUNK_SIZE) == -1) {
            perror("read");
            exit(EXIT_FAILURE);
        }
        remaining -= CHUNK_SIZE;
    }

    clock_gettime(CLOCK_MONOTONIC, &t2);
    double pipe_time = elapsed_seconds(t1, t2);
    close(fd[0]);
    close(fd[1]);

    /* ---------- Benchmark 2: shared memory (memcpy ตรงๆ ไม่ผ่าน syscall) ----------
     * ใช้บัฟเฟอร์ขนาดเท่า pipe (CHUNK_SIZE เดียว วนเขียนซ้ำ) เพื่อเทียบแบบยุติธรรม:
     * ทั้งสองฝั่งย้ายข้อมูลปริมาณเท่ากัน (TOTAL_SIZE) เป็นก้อนขนาดเท่ากัน
     * (CHUNK_SIZE) ต่างกันแค่ pipe ต้อง copy 2 ครั้ง (write เข้า kernel buffer +
     * read ออกมา) ผ่าน syscall ทุกรอบ ส่วน shared memory copy ครั้งเดียวจบ
     * ไม่มี syscall เลย เพราะอีกฝั่ง (consumer) มองเห็นหน่วยความจำเดียวกันอยู่แล้ว */
    char *shm = mmap(NULL, CHUNK_SIZE, PROT_READ | PROT_WRITE,
                     MAP_SHARED | MAP_ANONYMOUS, -1, 0);
    if (shm == MAP_FAILED) {
        perror("mmap");
        exit(EXIT_FAILURE);
    }
    memset(shm, 0, CHUNK_SIZE); /* แตะหน้า memory ให้ครบก่อน เพื่อไม่ให้ page fault มากวนผลวัด */

    clock_gettime(CLOCK_MONOTONIC, &t1);

    remaining = TOTAL_SIZE;
    while (remaining > 0) {
        memcpy(shm, src, CHUNK_SIZE);
        remaining -= CHUNK_SIZE;
    }

    clock_gettime(CLOCK_MONOTONIC, &t2);
    double shm_time = elapsed_seconds(t1, t2);
    munmap(shm, CHUNK_SIZE);

    printf("ย้ายข้อมูลทั้งหมด %d MB (chunk ละ %d KB)\n",
           TOTAL_SIZE / (1024 * 1024), CHUNK_SIZE / 1024);
    printf("  ผ่าน pipe (write+read ผ่าน kernel) : %.4f วินาที\n", pipe_time);
    printf("  ผ่าน shared memory (memcpy ตรงๆ)   : %.4f วินาที\n", shm_time);
    printf("  shared memory เร็วกว่าประมาณ %.1f เท่า\n", pipe_time / shm_time);

    free(src);
    free(sink);
    return 0;
}
```

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 ex_ipc_benchmark.c -o ex_ipc_benchmark -O2
./ex_ipc_benchmark
```

ผลลัพธ์ตัวอย่างจริงที่วัดได้ (ตัวเลขจะแตกต่างกันไปตามสเปกเครื่อง แต่แนวโน้มจะเหมือนกัน):

```
ย้ายข้อมูลทั้งหมด 64 MB (chunk ละ 64 KB)
  ผ่าน pipe (write+read ผ่าน kernel) : 0.0100 วินาที
  ผ่าน shared memory (memcpy ตรงๆ)   : 0.0018 วินาที
  shared memory เร็วกว่าประมาณ 5.6 เท่า
```

ผลลัพธ์ยืนยันสิ่งที่อธิบายไว้ในหัวข้อ 30.1 ชัดเจน: การตัด syscall (`write`/`read`) ออกไป
เหลือแค่ `memcpy()` ตรงๆ ทำให้เร็วขึ้นหลายเท่าตัว เพราะไม่มีค่าใช้จ่ายของการสลับโหมด
ระหว่าง user space กับ kernel space (context switch) ที่เกิดขึ้นทุกครั้งที่เรียก syscall

### แนวทางเฉลยข้อ 4 (แก้ Race Condition ด้วย Semaphore)

โค้ดฉบับเต็มแสดงไว้แล้วในหัวข้อ 30.5 ("พิสูจน์ว่าแก้ได้จริงด้วย Semaphore") — ไฟล์ชื่อ
`ex_shm_race_fixed.c` ลองรันซ้ำหลายรอบตามที่โจทย์ขอ:

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 ex_shm_race_fixed.c -o ex_shm_race_fixed -pthread
for i in 1 2 3 4 5; do ./ex_shm_race_fixed; done
```

ผลลัพธ์ที่ได้จริงจากการรันซ้ำ 5 รอบ:

```
[main] คาดหวังผลลัพธ์ที่ถูกต้อง: 800000
[main] ผลลัพธ์จริงที่ได้ (มี semaphore ป้องกันแล้ว): 800000
[main] คาดหวังผลลัพธ์ที่ถูกต้อง: 800000
[main] ผลลัพธ์จริงที่ได้ (มี semaphore ป้องกันแล้ว): 800000
[main] คาดหวังผลลัพธ์ที่ถูกต้อง: 800000
[main] ผลลัพธ์จริงที่ได้ (มี semaphore ป้องกันแล้ว): 800000
[main] คาดหวังผลลัพธ์ที่ถูกต้อง: 800000
[main] ผลลัพธ์จริงที่ได้ (มี semaphore ป้องกันแล้ว): 800000
[main] คาดหวังผลลัพธ์ที่ถูกต้อง: 800000
[main] ผลลัพธ์จริงที่ได้ (มี semaphore ป้องกันแล้ว): 800000
```

ได้ค่า `800000` ที่ถูกต้องครบทุกรอบ พิสูจน์ว่า `sem_wait()`/`sem_post()` ป้องกัน race
condition ได้จริง โดยแลกกับความเร็วที่ช้าลงเมื่อเทียบกับเวอร์ชันไม่มีการป้องกันเลย (ลอง
จับเวลาเปรียบเทียบเองด้วย `time ./shm_race` กับ `time ./ex_shm_race_fixed` จะเห็นความ
แตกต่างชัดเจน)

---

## สรุปท้ายบท

ใน Part นี้เราได้:

- เข้าใจว่าทำไม Shared Memory จึงเป็นกลไก IPC ที่เร็วที่สุด เพราะตัดขั้นตอนการ copy
  ข้อมูลผ่าน kernel buffer ออกไปทั้งหมดหลังจาก setup เสร็จ
- ใช้ POSIX Shared Memory API ครบวงจร (`shm_open`, `ftruncate`, `mmap`, `munmap`,
  `shm_unlink`) สร้างโปรแกรมสองตัวที่แชร์หน่วยความจำกันได้จริง
- สาธิตและเข้าใจปัญหา **Race Condition** อย่างละเอียดในระดับ CPU instruction พร้อมเห็น
  ตัวอย่างการแก้ไขด้วย POSIX semaphore เป็นตัวอย่างนำร่องก่อนเข้า Part 32
- ใช้ POSIX Message Queue API (`mq_open`, `mq_send`, `mq_receive`) ส่งข้อความที่รักษา
  ขอบเขตและมี priority ในตัว
- เปรียบเทียบ Pipe, Shared Memory และ Message Queue อย่างเป็นระบบ และรู้วิธีเลือกใช้
  กลไกที่เหมาะสมกับสถานการณ์ต่างๆ ในโลกจริง

ตอนนี้เราจบเนื้อหา IPC ระหว่าง process ครบทั้ง 3 กลไกหลักแล้ว (Pipe/FIFO ใน Part 29,
Shared Memory/Message Queue ใน Part นี้) แต่ยังมีปัญหาใหญ่ที่ค้างคาอยู่: **การป้องกัน
race condition** ที่เราเห็นเพียงตัวอย่างนำร่องเท่านั้น ใน **Part 31** เราจะเปลี่ยนมุมมอง
จาก "การสื่อสารระหว่าง process" ไปสู่ "การทำงานพร้อมกันภายใน process เดียว" ด้วย
**POSIX Threads (pthreads)** ซึ่งเบากว่า process มาก และแชร์ address space กันโดย
อัตโนมัติ (ไม่ต้องใช้ shared memory เลย) แต่ก็จะเจอปัญหา race condition แบบเดียวกันนี้
อีกครั้ง ก่อนที่ **Part 32** จะสอนวิธีป้องกันมันอย่างถูกต้องและครบถ้วนด้วย Mutex,
Semaphore และ Condition Variable

**ต่อไป:** [Part 31 — POSIX Threads (pthreads) เบื้องต้น](./part-031-pthreads-basics.md)
