# Part 36: Memory-Mapped File (mmap) (Step 281–288)

> Module C — Systems Programming ด้วย C บน Linux | Part 36 จาก 125
> Step ที่ครอบคลุมใน Part นี้: Step 281–288
> Part ก่อนหน้า: [Part 35 — โปรเจกต์ TCP Chat Server/Client](./part-035-tcp-chat-project.md) | Part ถัดไป: [Part 37 — การ Debug ด้วย GDB](./part-037-gdb-debugging.md)

## เป้าหมายการเรียนรู้ (Learning Objectives)

หลังจบ Part นี้ ผู้เรียนจะสามารถ:

1. อธิบายได้ว่า `mmap()` คืออะไร และต่างจากการอ่านไฟล์ด้วย `fread()`/`read()` แบบดั้งเดิม
   อย่างไรในระดับกลไกภายในของระบบปฏิบัติการ
2. เลือกใช้ค่า `PROT_READ`, `PROT_WRITE`, `MAP_SHARED`, `MAP_PRIVATE` ได้ถูกต้องตามโจทย์
3. เปิดไฟล์ขนาดใหญ่ด้วย `mmap()` แล้วอ่าน/แก้ไขข้อมูลผ่าน pointer ได้โดยตรง
4. วัดและเปรียบเทียบความเร็วของการอ่านไฟล์ด้วย `mmap()` เทียบกับ `fread()` ด้วยตัวเองอย่าง
   ยุติธรรม (ควบคุมปัจจัยเรื่อง page cache ของเคอร์เนล)
5. ใช้ `msync()` บังคับให้ข้อมูลที่แก้ไขผ่าน mmap ถูกเขียนกลับไฟล์บนดิสก์จริงได้
6. ใช้ `mmap()` แบบ `MAP_ANONYMOUS | MAP_SHARED` สร้างหน่วยความจำที่แชร์กันระหว่าง process
   พ่อ-ลูกหลัง `fork()` ได้ เป็นทางเลือกที่ง่ายกว่า System V shared memory (Part 30)
7. อธิบายและสาธิตได้ว่าทำไมการเข้าถึง memory ที่ mmap ไว้เกินขอบเขตไฟล์จริงจะทำให้เกิด
   สัญญาณ `SIGBUS` และรู้วิธีป้องกัน/ดักจับสัญญาณนี้
8. ตัดสินใจได้ว่าเมื่อไหร่ควรใช้ `mmap()` และเมื่อไหร่ควรใช้ `read()`/`write()` แบบดั้งเดิม
   ในงานจริง

---

## 36.1 mmap คืออะไร ต่างจากการอ่านไฟล์ด้วย fread อย่างไร (Step 281)

จนถึงตอนนี้เราอ่าน/เขียนไฟล์ด้วยวิธีมาตรฐานที่เรียนใน Part 13 (File I/O): เปิดไฟล์ด้วย
`fopen()`/`open()` แล้วเรียก `fread()`/`read()` เพื่อ **copy** ข้อมูลจากไฟล์เข้ามาไว้ใน
buffer ที่เราจัดสรรเองใน memory ทุกครั้งที่ต้องการข้อมูลใหม่ ต้องเรียก syscall ซ้ำแล้วซ้ำอีก

`mmap()` (memory map) เสนอวิธีคิดที่ต่างออกไปโดยสิ้นเชิง: แทนที่จะ copy ข้อมูลมาไว้ใน buffer
`mmap()` จะ **"เอาไฟล์ทั้งไฟล์ (หรือบางส่วน) มาผูกเข้ากับช่วง address space ของโปรเซสโดยตรง"**
กลายเป็นว่าเราสามารถ "อ่าน" หรือ "เขียน" ไฟล์ได้ด้วยการเข้าถึง pointer ธรรมดา
(`data[i]`, `*ptr`) เหมือนกับกำลังทำงานกับ array ใน memory ทั่วไป โดยไม่ต้องเรียก
`read()`/`write()` ซ้ำเลยแม้แต่ครั้งเดียว

### เปรียบเทียบกลไกภายใน

```
วิธีดั้งเดิม (fread):
   ┌──────────┐  read()   ┌──────────────┐  copy   ┌─────────────────┐
   │  ไฟล์บนดิสก์  │ ───────▶ │ Kernel page   │ ──────▶ │ Buffer ใน User   │
   │             │         │ cache (RAM)   │         │ Space (ตัวแปรเรา) │
   └──────────┘           └──────────────┘         └─────────────────┘
                                                      โปรแกรมอ่านจากตรงนี้
                           (การ copy เกิดขึ้นทุกครั้งที่เรียก read/fread)

วิธี mmap:
   ┌──────────┐  page fault  ┌──────────────┐   map โดยตรง (ไม่ copy)
   │  ไฟล์บนดิสก์  │ ◀──────────▶ │ Kernel page   │ ◀═══════════════════╗
   │             │  (ตอนเข้าถึง) │ cache (RAM)   │                     ║
   └──────────┘              └──────────────┘                     ║
                                                     ┌─────────────────┐
                                                     │ Address Space   │
                                                     │ ของโปรเซสเรา     │
                                                     │ (data[i] ชี้ไป   │
                                                     │  ที่ page cache  │
                                                     │  ตรงๆ เลย)       │
                                                     └─────────────────┘
```

ความแตกต่างที่สำคัญที่สุด: วิธีดั้งเดิมมี **การ copy ข้อมูลอย่างน้อย 1 รอบ** (จาก kernel
page cache ไปยัง user buffer) ทุกครั้งที่เรียก `read()`/`fread()` แต่วิธี `mmap()` ทำให้
**pointer ของโปรแกรมเราชี้ตรงไปยัง page cache ของเคอร์เนลเลย** ไม่มีการ copy พิเศษเกิดขึ้น
(นอกเหนือจากที่จำเป็นตอน page fault ครั้งแรกที่เข้าถึงแต่ละหน้าความจำ) นี่คือที่มาของคำว่า
"zero-copy I/O" ที่มักถูกกล่าวถึงคู่กับ `mmap()`

### ตารางเปรียบเทียบ

| หัวข้อ | `fread()`/`read()` | `mmap()` |
|---|---|---|
| การเข้าถึงข้อมูล | ผ่าน buffer ที่เรา malloc/ประกาศเอง | ผ่าน pointer ที่ชี้ตรงเข้า address space |
| การ copy ข้อมูล | copy จาก kernel buffer ไปยัง user buffer เสมอ | ไม่มีการ copy พิเศษ (page cache ถูก map ตรง) |
| การเข้าถึงแบบสุ่ม (random access) | ต้อง `fseek()` ก่อนทุกครั้ง | เข้าถึง `data[offset]` ได้ตรงๆ ทันที |
| การแก้ไขไฟล์ | ต้อง `fseek()` + `fwrite()` | เขียนผ่าน pointer ได้เลย (ถ้า `PROT_WRITE`) |
| เหมาะกับไฟล์ขนาด | เล็ก-กลาง หรืออ่านตามลำดับ (sequential) | ใหญ่, เข้าถึงแบบสุ่มบ่อย, ใช้ร่วมกันหลายโปรเซส |
| ความซับซ้อนในการจัดการ error | ตรงไปตรงมา (return code ปกติ) | ซับซ้อนกว่า (ต้องระวัง `SIGBUS`, ดู 36.7) |

---

## 36.2 Anatomy ของ mmap(): Flags และพารามิเตอร์ทั้งหมด (Step 282)

Function signature เต็มของ `mmap()` (ประกาศใน `<sys/mman.h>`):

```c
void *mmap(void *addr, size_t length, int prot, int flags, int fd, off_t offset);
```

| พารามิเตอร์ | ความหมาย |
|---|---|
| `addr` | ที่อยู่ที่ต้องการให้ map (ปกติใส่ `NULL` เพื่อให้เคอร์เนลเลือกที่อยู่ที่เหมาะสมให้เอง) |
| `length` | ขนาดของ memory ที่ต้องการ map (หน่วยไบต์ แต่จะถูกปัดขึ้นเป็นทวีคูณของ page size เสมอ) |
| `prot` | สิทธิ์การเข้าถึง memory ที่จะ map (`PROT_READ`, `PROT_WRITE`, `PROT_EXEC`, `PROT_NONE`) |
| `flags` | ประเภทของ mapping (`MAP_SHARED`, `MAP_PRIVATE`, `MAP_ANONYMOUS`, ...) |
| `fd` | file descriptor ของไฟล์ที่จะ map (ใส่ `-1` ถ้าใช้ `MAP_ANONYMOUS`) |
| `offset` | ตำแหน่งเริ่มต้นในไฟล์ที่จะ map (ต้องเป็นทวีคูณของ page size เสมอ ปกติใส่ `0`) |

ค่าคืนกลับ: pointer ไปยังจุดเริ่มต้นของ memory ที่ map สำเร็จ หรือค่าคงที่พิเศษ
`MAP_FAILED` (ไม่ใช่ `NULL`!) เมื่อล้มเหลว — นี่คือจุดพลาดคลาสสิกจุดแรก (ดู "ข้อผิดพลาด
ที่พบบ่อย" ข้อ 1)

### PROT_READ / PROT_WRITE — สิทธิ์การเข้าถึง

กำหนดว่าโปรแกรมมีสิทธิ์ **อ่าน** และ/หรือ **เขียน** memory ที่ map ไว้ได้หรือไม่ (คล้ายกับ
permission ของไฟล์ แต่เป็นระดับ memory page) ค่าที่ใช้บ่อยที่สุด:

- `PROT_READ` เท่านั้น: เหมาะกับการอ่านไฟล์อย่างเดียว (read-only) เช่น โหลดไฟล์ config
- `PROT_READ | PROT_WRITE`: เหมาะกับการแก้ไขไฟล์โดยตรง

ถ้าเปิดไฟล์ด้วย `open(path, O_RDONLY)` แล้วพยายาม `mmap()` ด้วย `PROT_WRITE` จะได้ error
`EACCES` ทันที เพราะสิทธิ์ของ file descriptor ต้องครอบคลุมสิทธิ์ที่ขอใน `mmap()` เสมอ

### MAP_SHARED กับ MAP_PRIVATE — ความแตกต่างที่สำคัญที่สุด

นี่คือ flag ที่มือใหม่สับสนบ่อยที่สุด และเข้าใจผิดแล้วเป็นอันตรายมากที่สุด:

| | `MAP_SHARED` | `MAP_PRIVATE` |
|---|---|---|
| การเปลี่ยนแปลงสะท้อนกลับไฟล์จริงหรือไม่ | **ใช่** เขียนแล้วมีผลกับไฟล์บนดิสก์จริง | **ไม่** เขียนแล้วเป็นแค่สำเนาของโปรเซสตัวเอง (copy-on-write) |
| การแชร์กับ process อื่นที่ map ไฟล์เดียวกัน | เห็นการเปลี่ยนแปลงของกันและกัน | ไม่เห็น แต่ละโปรเซสมีสำเนาของตัวเอง |
| ใช้เมื่อไหร่ | ต้องการแก้ไขไฟล์จริง หรือทำ shared memory ระหว่างโปรเซส | อ่านอย่างเดียว หรือต้องการ "ทดลองแก้ไข" โดยไม่กระทบต้นฉบับ |

ตัวอย่างที่ชัดเจน: ถ้าเปิดไฟล์ config ด้วย `mmap()` แบบ `PROT_READ | PROT_WRITE` กับ
`MAP_PRIVATE` แล้วเขียนทับค่าบางตัวเพื่อทดสอบ การเปลี่ยนแปลงนั้นจะอยู่แค่ใน memory ของ
โปรเซสเราเท่านั้น ไฟล์จริงบนดิสก์จะไม่ถูกแตะต้องเลย เหมาะมากสำหรับการทดลองแก้ไขข้อมูลชั่วคราว

### MAP_ANONYMOUS — mmap โดยไม่ผูกกับไฟล์ใดๆ

เมื่อรวมกับ `MAP_SHARED` และ `fd = -1` จะได้หน่วยความจำที่ **ไม่ผูกกับไฟล์บนดิสก์เลย**
เต็มไปด้วยไบต์ 0 ตอนเริ่มต้น ใช้เป็น shared memory ระหว่าง process พ่อ-ลูกได้ (ดู 36.6)

---

## 36.3 ตัวอย่างจริง: อ่านไฟล์ขนาดใหญ่ด้วย mmap เทียบกับ fread (Step 283)

มาวัดผลจริงกัน — เขียนโปรแกรมสร้างไฟล์ทดสอบขนาด 300 MB ก่อน:

```c
/* gen_testfile.c: สร้างไฟล์ทดสอบขนาดใหญ่ที่เต็มไปด้วยข้อมูล
 * pseudo-random แบบทำซ้ำได้ (deterministic) */
#include <stdio.h>
#include <stdlib.h>

int main(int argc, char *argv[]) {
    if (argc != 3) {
        fprintf(stderr, "usage: %s <filename> <size_mb>\n", argv[0]);
        return 1;
    }
    const char *filename = argv[1];
    long size_mb = atol(argv[2]);
    long total_bytes = size_mb * 1024L * 1024L;

    FILE *fp = fopen(filename, "wb");
    if (!fp) {
        perror("fopen");
        return 1;
    }

    unsigned char buf[65536];
    unsigned int seed = 12345;
    long written = 0;
    while (written < total_bytes) {
        for (size_t i = 0; i < sizeof(buf); i++) {
            seed = seed * 1103515245u + 12345u;
            buf[i] = (unsigned char)(seed >> 16);
        }
        size_t to_write = sizeof(buf);
        if ((long)to_write > total_bytes - written) {
            to_write = (size_t)(total_bytes - written);
        }
        fwrite(buf, 1, to_write, fp);
        written += (long)to_write;
    }
    fclose(fp);
    printf("สร้างไฟล์ %s ขนาด %ld MB สำเร็จ\n", filename, size_mb);
    return 0;
}
```

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 -O2 gen_testfile.c -o gen_testfile
./gen_testfile testfile.bin 300
```

### โปรแกรมเปรียบเทียบความเร็ว

```c
/* ============================================================
 * ชื่อไฟล์:     compare_read.c
 * คำอธิบาย:     เปรียบเทียบความเร็วการอ่านไฟล์ขนาดใหญ่ด้วย fread()
 *              (อ่านผ่าน buffer ตามปกติ) กับ mmap() (map ไฟล์เข้า
 *              address space โดยตรงแล้วเข้าถึงผ่าน pointer)
 *
 *              รับโหมดเป็น argument เพื่อให้เรียกทีละโหมด แยกโปรเซส
 *              กัน — จำเป็นสำหรับการวัดที่ยุติธรรม เพราะถ้ารันสองวิธี
 *              ในโปรเซสเดียวติดกัน วิธีที่รันทีหลังจะได้เปรียบเพราะ
 *              ข้อมูลถูกแคชไว้ใน page cache ของเคอร์เนลจากรอบแรกแล้ว
 * ============================================================ */
#define _POSIX_C_SOURCE 200809L

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <time.h>
#include <fcntl.h>
#include <unistd.h>
#include <sys/mman.h>
#include <sys/stat.h>

static double now_seconds(void) {
    struct timespec ts;
    clock_gettime(CLOCK_MONOTONIC, &ts);
    return (double)ts.tv_sec + (double)ts.tv_nsec / 1e9;
}

/* อ่านไฟล์ทั้งหมดด้วย fread() ผ่าน buffer ขนาด 64KB วนซ้ำ แล้วรวมค่า
 * ไบต์ทั้งหมด (checksum อย่างง่าย) เพื่อบังคับให้ compiler ไม่ optimize
 * การอ่านทิ้งไปเฉยๆ (ต้องมีการใช้ผลลัพธ์จริง) */
static unsigned long sum_with_fread(const char *path, double *elapsed) {
    FILE *fp = fopen(path, "rb");
    if (!fp) {
        perror("fopen");
        exit(EXIT_FAILURE);
    }

    unsigned char buf[65536];
    unsigned long checksum = 0;
    size_t n;

    double start = now_seconds();
    while ((n = fread(buf, 1, sizeof(buf), fp)) > 0) {
        for (size_t i = 0; i < n; i++) {
            checksum += buf[i];
        }
    }
    *elapsed = now_seconds() - start;

    fclose(fp);
    return checksum;
}

/* mmap() ไฟล์ทั้งไฟล์เข้ามาเป็น pointer เดียว แล้ววน sum ไบต์ทั้งหมด
 * ตรงๆ ผ่าน pointer โดยไม่มีการ copy ข้อมูลผ่าน buffer กลางเหมือน
 * fread() (kernel จัดการ page fault ให้อัตโนมัติเมื่อเข้าถึงแต่ละหน้า) */
static unsigned long sum_with_mmap(const char *path, double *elapsed) {
    int fd = open(path, O_RDONLY);
    if (fd < 0) {
        perror("open");
        exit(EXIT_FAILURE);
    }

    struct stat st;
    if (fstat(fd, &st) < 0) {
        perror("fstat");
        close(fd);
        exit(EXIT_FAILURE);
    }
    size_t filesize = (size_t)st.st_size;

    double start = now_seconds();

    /* PROT_READ: ขอสิทธิ์อ่านอย่างเดียว, MAP_PRIVATE: การเปลี่ยนแปลง
     * (ถ้ามี) จะไม่ถูกเขียนกลับไฟล์จริง เหมาะกับกรณีอ่านอย่างเดียวแบบนี้ */
    unsigned char *data = mmap(NULL, filesize, PROT_READ, MAP_PRIVATE, fd, 0);
    if (data == MAP_FAILED) {
        perror("mmap");
        close(fd);
        exit(EXIT_FAILURE);
    }

    unsigned long checksum = 0;
    for (size_t i = 0; i < filesize; i++) {
        checksum += data[i];
    }

    *elapsed = now_seconds() - start;

    munmap(data, filesize);
    close(fd);
    return checksum;
}

int main(int argc, char *argv[]) {
    if (argc != 3 || (strcmp(argv[2], "fread") != 0 && strcmp(argv[2], "mmap") != 0)) {
        fprintf(stderr, "usage: %s <filename> <fread|mmap>\n", argv[0]);
        return EXIT_FAILURE;
    }

    double elapsed = 0.0;
    unsigned long checksum;

    if (strcmp(argv[2], "fread") == 0) {
        checksum = sum_with_fread(argv[1], &elapsed);
        printf("[fread] checksum=%lu  เวลา=%.4f วินาที\n", checksum, elapsed);
    } else {
        checksum = sum_with_mmap(argv[1], &elapsed);
        printf("[mmap ] checksum=%lu  เวลา=%.4f วินาที\n", checksum, elapsed);
    }

    return EXIT_SUCCESS;
}
```

สังเกตบรรทัดแรกสุดของไฟล์: `#define _POSIX_C_SOURCE 200809L` **ก่อน** `#include` ใดๆ
ทั้งสิ้น — จำเป็นมากเมื่อคอมไพล์ด้วย `-std=c17` เพราะ `-std=c17` บังคับให้ compiler ใช้
มาตรฐาน ISO C ล้วนๆ ซึ่ง **ไม่รวม** ฟังก์ชันของ POSIX อย่าง `clock_gettime()`,
`CLOCK_MONOTONIC`, หรือแม้แต่ `mmap()` เองเข้ามาให้โดยอัตโนมัติ ถ้าลืมบรรทัดนี้จะเจอ error
`'CLOCK_MONOTONIC' undeclared` (พบจริงระหว่างพัฒนาตัวอย่างนี้)

### ทดสอบให้ยุติธรรม: ต้องควบคุม Page Cache

ปัญหาสำคัญของการวัดความเร็ว I/O คือ **เคอร์เนลจะแคชเนื้อหาไฟล์ไว้ใน RAM (page cache)**
หลังจากอ่านครั้งแรก ถ้าเรารันทั้งสองวิธีติดกันในโปรแกรมเดียว วิธีที่รันทีหลังจะได้เปรียบ
อย่างไม่เป็นธรรมเพราะข้อมูลถูกแคชไว้แล้วจากรอบแรก เพื่อวัดผลอย่างยุติธรรม เราต้อง **ล้าง
page cache ก่อนวัดแต่ละวิธี** (ต้องมีสิทธิ์ root):

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 -g -O2 compare_read.c -o compare_read

sync; echo 3 > /proc/sys/vm/drop_caches
./compare_read testfile.bin fread

sync; echo 3 > /proc/sys/vm/drop_caches
./compare_read testfile.bin mmap
```

ผลลัพธ์จริงจากการทดสอบ (ไฟล์ขนาด 300 MB, cold cache ทั้งสองรอบ):

```
[fread] checksum=40107877504  เวลา=0.2603 วินาที
[mmap ] checksum=40107877504  เวลา=0.1481 วินาที
```

`checksum` ตรงกันทั้งสองวิธี ยืนยันว่าอ่านข้อมูลได้ถูกต้องเหมือนกัน ในการทดสอบครั้งนี้
`mmap()` เร็วกว่าประมาณ 1.75 เท่า และเมื่อรันซ้ำหลายรอบ (cold cache ทุกครั้ง) ผลลัพธ์
แกว่งอยู่ในช่วง **เร็วกว่าประมาณ 1.2–2 เท่า** ขึ้นกับสภาพของดิสก์และระบบ ณ ขณะนั้น
และเมื่อทดสอบด้วย **warm cache** (ไม่ล้าง cache เลย รันซ้ำหลายรอบ) ผลต่างจะแคบลงเหลือ
ประมาณ **1.15–1.3 เท่า** เพราะทั้งสองวิธีอ่านจาก RAM เหมือนกัน ต่างกันแค่ที่ `fread()`
ยังต้อง copy ข้อมูลจาก page cache มาที่ user buffer อีกชั้นหนึ่งอยู่ดี ในขณะที่ `mmap()`
ไม่ต้อง copy เลย

> **ทำไมไม่เร็วกว่ากันมากอย่างที่คาดหวัง?** เพราะ `fread()` ก็มี internal buffering
> ของตัวเองที่ค่อนข้างมีประสิทธิภาพอยู่แล้ว ความได้เปรียบของ `mmap()` จะเห็นชัดกว่านี้มาก
> ในสถานการณ์อื่น เช่น การเข้าถึงข้อมูลแบบสุ่ม (random access) ในไฟล์ขนาดใหญ่ที่ไม่ต้อง
> `fseek()` ไปมา หรือเมื่อหลายโปรเซสต้องการอ่านไฟล์เดียวกันพร้อมกัน (page cache ถูกแชร์กัน
> ในระดับเคอร์เนลอยู่แล้ว ไม่ต้องแต่ละโปรเซส copy ข้อมูลซ้ำเป็นของตัวเอง)

---

## 36.4 แก้ไขไฟล์ผ่าน mmap โดยตรง (Step 284)

ตัวอย่างนี้เปิดไฟล์ที่มีอยู่แล้ว แล้วแก้ไขเนื้อหาผ่าน pointer โดยตรง ไม่มีการเรียก
`fseek()`/`fwrite()` เลยแม้แต่บรรทัดเดียว:

```c
/* ============================================================
 * ชื่อไฟล์:     mmap_edit.c
 * คำอธิบาย:     เปิดไฟล์แล้ว mmap ด้วย PROT_WRITE + MAP_SHARED เพื่อ
 *              แก้ไขเนื้อหาไฟล์โดยตรงผ่าน pointer (ไม่ต้อง fseek/fwrite)
 *              จากนั้นใช้ msync() บังคับให้ข้อมูลถูกเขียนกลับไฟล์จริง
 *              บนดิสก์ทันที แทนที่จะรอให้เคอร์เนล flush เองภายหลัง
 * ============================================================ */
#define _POSIX_C_SOURCE 200809L

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <fcntl.h>
#include <unistd.h>
#include <sys/mman.h>
#include <sys/stat.h>

int main(int argc, char *argv[]) {
    if (argc != 2) {
        fprintf(stderr, "usage: %s <filename>\n", argv[0]);
        return EXIT_FAILURE;
    }

    /* เปิดไฟล์แบบอ่าน-เขียน (O_RDWR) — ต้องเปิดแบบนี้เพราะเราจะ mmap
     * ด้วย PROT_WRITE ด้วย ถ้าเปิดแค่ O_RDONLY แล้ว mmap ขอ PROT_WRITE
     * จะได้ error EACCES ทันที */
    int fd = open(argv[1], O_RDWR);
    if (fd < 0) {
        perror("open");
        return EXIT_FAILURE;
    }

    struct stat st;
    if (fstat(fd, &st) < 0) {
        perror("fstat");
        close(fd);
        return EXIT_FAILURE;
    }
    size_t filesize = (size_t)st.st_size;

    if (filesize == 0) {
        fprintf(stderr, "ไฟล์ว่างเปล่า ไม่มีอะไรให้แก้ไข\n");
        close(fd);
        return EXIT_FAILURE;
    }

    /* PROT_READ | PROT_WRITE: ขอสิทธิ์ทั้งอ่านและเขียน
     * MAP_SHARED: การเปลี่ยนแปลงที่เขียนผ่าน pointer จะสะท้อนกลับไปยัง
     * ไฟล์จริงบนดิสก์ (ต่างจาก MAP_PRIVATE ที่ใช้ copy-on-write เฉพาะ
     * ใน memory ของโปรเซสตัวเอง ไม่กระทบไฟล์ต้นฉบับเลย) */
    char *data = mmap(NULL, filesize, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
    if (data == MAP_FAILED) {
        perror("mmap");
        close(fd);
        return EXIT_FAILURE;
    }

    printf("แก้ไขไฟล์ %s (ขนาด %zu ไบต์) ผ่าน mmap โดยตรง\n", argv[1], filesize);
    printf("10 ไบต์แรกก่อนแก้ไข: ");
    for (size_t i = 0; i < 10 && i < filesize; i++) {
        printf("%02x ", (unsigned char)data[i]);
    }
    printf("\n");

    /* แก้ไขข้อมูล "ผ่าน pointer โดยตรง" — ไม่มี fseek(), ไม่มี fwrite()
     * เลยแม้แต่นิดเดียว เหมือนกำลังแก้ไข array ธรรมดาใน memory เท่านั้น */
    size_t n = filesize < 10 ? filesize : 10;
    for (size_t i = 0; i < n; i++) {
        data[i] = (char)('A' + (int)i);
    }

    printf("10 ไบต์แรกหลังแก้ไข: ");
    for (size_t i = 0; i < 10 && i < filesize; i++) {
        printf("%02x ", (unsigned char)data[i]);
    }
    printf("\n");

    /* msync(): ดูรายละเอียดใน 36.5 */
    if (msync(data, filesize, MS_SYNC) < 0) {
        perror("msync");
    } else {
        printf("msync() สำเร็จ ข้อมูลถูกเขียนกลับไฟล์บนดิสก์แล้ว\n");
    }

    /* munmap(): เลิก map หน่วยความจำนี้ (ควรทำเสมอเมื่อเลิกใช้งาน
     * เหมือนกับ free() คู่กับ malloc() — ถ้าไม่ munmap โปรแกรมที่รัน
     * นานๆ จะเสีย virtual address space ไปเรื่อยๆ ถ้ามีการ mmap ซ้ำๆ) */
    if (munmap(data, filesize) < 0) {
        perror("munmap");
    }

    close(fd);
    return EXIT_SUCCESS;
}
```

ทดสอบจริง:

```bash
$ echo "Hello, mmap world! This is a test file." > small.txt
$ od -c small.txt | head -3
0000000   H   e   l   l   o   ,       m   m   a   p       w   o   r   l
0000020   d   !       T   h   i   s       i   s       a       t   e   s
0000040   t       f   i   l   e   .  \n

$ ./mmap_edit small.txt
แก้ไขไฟล์ small.txt (ขนาด 40 ไบต์) ผ่าน mmap โดยตรง
10 ไบต์แรกก่อนแก้ไข: 48 65 6c 6c 6f 2c 20 6d 6d 61
10 ไบต์แรกหลังแก้ไข: 41 42 43 44 45 46 47 48 49 4a
msync() สำเร็จ ข้อมูลถูกเขียนกลับไฟล์บนดิสก์แล้ว

$ cat small.txt
ABCDEFGHIJp world! This is a test file.

$ od -c small.txt | head -3
0000000   A   B   C   D   E   F   G   H   I   J   p       w   o   r   l
0000020   d   !       T   h   i   s       i   s       a       t   e   s
0000040   t       f   i   l   e   .  \n
```

ไฟล์บนดิสก์ถูกแก้ไขจริง 10 ไบต์แรกโดยไม่มีการเรียก `fwrite()` แม้แต่ครั้งเดียว — นี่คือ
พลังของ `MAP_SHARED`: pointer `data` **คือ** ไฟล์นั้นเอง (ในมุมมองของโปรแกรม) การเขียน
ทับค่าใน `data[i]` จึงเหมือนกับการเขียนทับไฟล์โดยตรง

---

## 36.5 msync(): บังคับ Flush การเปลี่ยนแปลงกลับไฟล์ (Step 285)

เมื่อแก้ไขข้อมูลผ่าน `mmap()` ด้วย `MAP_SHARED` เคอร์เนลจะทำเครื่องหมายหน้าความจำ
(memory page) ที่ถูกแก้ไขว่าเป็น **"dirty page"** และจะ flush ข้อมูลกลับไฟล์บนดิสก์
**เองโดยอัตโนมัติเป็นระยะๆ** (ตาม timer ภายในของเคอร์เนล หรือเมื่อหน่วยความจำเริ่มขาดแคลน)
แต่ถ้าโปรแกรมของเราต้องการ **รับประกันว่าข้อมูลถูกเขียนลงดิสก์แน่นอน ณ จุดใดจุดหนึ่ง**
(เช่น ก่อนที่จะแจ้งโปรแกรมอื่นว่า "เขียนเสร็จแล้ว" หรือก่อนโปรแกรมจะปิดตัวลง) ต้องเรียก
`msync()` เอง — คล้ายกับ `fsync()` ของ file descriptor ธรรมดา แต่ทำงานกับหน้าความจำที่ map ไว้

```c
int msync(void *addr, size_t length, int flags);
```

| Flag | ความหมาย |
|---|---|
| `MS_SYNC` | รอจนกว่าการเขียนข้อมูลกลับดิสก์จะเสร็จสมบูรณ์จริงก่อนคืนค่าออกมา (blocking) |
| `MS_ASYNC` | สั่งให้เริ่ม flush แต่คืนค่าออกมาทันทีโดยไม่รอให้เขียนเสร็จ (non-blocking) |
| `MS_INVALIDATE` | ล้าง cache อื่นๆ ของ mapping นี้ที่อาจไม่ตรงกัน (ใช้ร่วมกับ flag ข้างต้น) |

ในตัวอย่าง 36.4 เราใช้ `MS_SYNC` เพราะต้องการความมั่นใจสูงสุดว่าข้อมูลลงดิสก์แล้วจริงก่อน
จะพิมพ์ข้อความยืนยันออกมา ถ้าโปรแกรมของเราไม่สนใจเวลาที่แน่นอนที่ข้อมูลจะลงดิสก์ (เช่น
เขียน log ที่ไม่ critical มาก) การใช้ `MS_ASYNC` จะให้ประสิทธิภาพที่ดีกว่าเพราะไม่ต้อง
block รอ

> **ข้อควรรู้**: ถ้าไม่เรียก `msync()` เลย ข้อมูลก็ **ยังจะถูกเขียนกลับไฟล์ในที่สุด** อยู่ดี
> เมื่อ `munmap()` หรือเมื่อโปรแกรมจบการทำงานอย่างปกติ (เคอร์เนลจะ flush dirty page ให้
> โดยอัตโนมัติ) `msync()` จำเป็นเฉพาะเมื่อต้องการควบคุม **จังหวะเวลา** ที่แน่นอนของการ
> เขียนลงดิสก์เท่านั้น ไม่ใช่เงื่อนไขบังคับสำหรับให้ข้อมูลถูกบันทึกจริง

---

## 36.6 mmap สำหรับ Shared Memory ระหว่าง Process (Step 286)

ทบทวนจาก Part 30: เราเคยใช้ System V shared memory (`shmget()`/`shmat()`) และ POSIX
shared memory (`shm_open()`) เพื่อแชร์หน่วยความจำระหว่าง process ที่ไม่มีความสัมพันธ์กัน
โดยตรง (unrelated processes) แต่เมื่อ process ทั้งสองมีความสัมพันธ์แบบ **parent-child**
(ผ่าน `fork()`) มีวิธีที่ **ง่ายกว่ามาก**: ใช้ `mmap()` กับ `MAP_ANONYMOUS | MAP_SHARED`

```c
/* ============================================================
 * ชื่อไฟล์:     mmap_shared_fork.c
 * คำอธิบาย:     ใช้ mmap() แบบ MAP_ANONYMOUS | MAP_SHARED สร้างหน่วย
 *              ความจำที่ "ไม่ผูกกับไฟล์ใดๆ" (anonymous) แต่แชร์กันได้
 *              ระหว่าง process พ่อ-ลูกหลัง fork() — ใช้แทน System V
 *              shared memory (shmget/shmat จาก Part 30) แบบง่ายกว่ามาก
 *              สำหรับกรณี parent/child ที่มีความสัมพันธ์กันโดยตรง
 * ============================================================ */
#define _POSIX_C_SOURCE 200809L

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/mman.h>
#include <sys/wait.h>

/* struct ที่จะแชร์กันระหว่าง parent กับ child ผ่าน memory เดียวกัน */
typedef struct {
    int counter;
    char message[64];
} shared_data_t;

int main(void) {
    /* MAP_ANONYMOUS: ไม่ผูกกับไฟล์จริง (ใส่ fd เป็น -1 และ offset เป็น 0)
     * MAP_SHARED: การเปลี่ยนแปลงมองเห็นร่วมกันได้ระหว่างโปรเซสที่ map
     * หน่วยความจำนี้ (ในที่นี้คือ parent และ child ที่ได้ mapping มา
     * จาก fork() ซึ่งจะสืบทอด memory mapping ทั้งหมดของ parent ไปด้วย) */
    shared_data_t *shared = mmap(NULL, sizeof(shared_data_t),
                                  PROT_READ | PROT_WRITE,
                                  MAP_SHARED | MAP_ANONYMOUS,
                                  -1, 0);
    if (shared == MAP_FAILED) {
        perror("mmap");
        return EXIT_FAILURE;
    }

    shared->counter = 0;
    strcpy(shared->message, "ยังไม่มีใครแก้ไข");

    pid_t pid = fork();
    if (pid < 0) {
        perror("fork");
        munmap(shared, sizeof(shared_data_t));
        return EXIT_FAILURE;
    }

    if (pid == 0) {
        /* ---- Child process ---- */
        sleep(1);   /* หน่วงให้ parent พิมพ์ค่าเริ่มต้นก่อน อ่านง่ายขึ้น */
        printf("[child ] เห็นค่า counter = %d, message = \"%s\"\n",
               shared->counter, shared->message);

        shared->counter = 42;
        strcpy(shared->message, "แก้ไขโดย child process");
        printf("[child ] แก้ไขค่าแล้ว: counter = %d, message = \"%s\"\n",
               shared->counter, shared->message);

        munmap(shared, sizeof(shared_data_t));
        _exit(EXIT_SUCCESS);
    }

    /* ---- Parent process ---- */
    printf("[parent] ค่าเริ่มต้น: counter = %d, message = \"%s\"\n",
           shared->counter, shared->message);

    int status;
    waitpid(pid, &status, 0);   /* รอให้ child ทำงานเสร็จก่อน */

    printf("[parent] หลัง child จบการทำงาน เห็นค่า counter = %d, message = \"%s\"\n",
           shared->counter, shared->message);
    printf("[parent] (ถ้าใช้ MAP_PRIVATE แทน MAP_SHARED ค่านี้จะยังเป็น 0 "
           "และข้อความเดิม เพราะ child จะแก้ไขแค่สำเนาของตัวเองเท่านั้น)\n");

    munmap(shared, sizeof(shared_data_t));
    return EXIT_SUCCESS;
}
```

ผลลัพธ์จริงจากการรัน:

```
[parent] ค่าเริ่มต้น: counter = 0, message = "ยังไม่มีใครแก้ไข"
[parent] หลัง child จบการทำงาน เห็นค่า counter = 42, message = "แก้ไขโดย child process"
[parent] (ถ้าใช้ MAP_PRIVATE แทน MAP_SHARED ค่านี้จะยังเป็น 0 และข้อความเดิม เพราะ child จะแก้ไขแค่สำเนาของตัวเองเท่านั้น)
```

`parent` เห็นค่าที่ `child` แก้ไขจริง (`counter = 42`) ยืนยันว่า memory ถูกแชร์จริง
ระหว่างสอง process แยกกัน (คนละ address space, คนละ PID) ไม่ใช่แค่ตัวแปรธรรมดาที่ถูก
copy ไปตอน `fork()` (ทบทวนจาก Part 27: หลัง `fork()` โดยปกติตัวแปรทุกตัวจะถูก **copy**
แยกกันคนละชุดตาม copy-on-write แต่ memory ที่ `mmap()` ไว้แบบ `MAP_SHARED` **ไม่ถูก
copy** — ทั้ง parent และ child ยังคงชี้ไปยัง physical memory หน้าเดียวกัน)

เมื่อทดสอบเปลี่ยนเป็น `MAP_PRIVATE | MAP_ANONYMOUS` (ทุกอย่างเหมือนเดิมทุกประการ
เปลี่ยนแค่ flag เดียว) ผลลัพธ์เปลี่ยนไปทันที:

```
[parent] ค่าเริ่มต้น: counter = 0, message = "ยังไม่มีใครแก้ไข"
[parent] หลัง child จบการทำงาน เห็นค่า counter = 0, message = "ยังไม่มีใครแก้ไข"
```

`parent` **ไม่เห็น** การเปลี่ยนแปลงของ `child` เลย เพราะ `MAP_PRIVATE` ทำให้ทันทีที่
`child` เขียนทับข้อมูล เคอร์เนลจะทำ copy-on-write สร้างสำเนาหน้าความจำใหม่ให้ `child`
โดยเฉพาะ ไม่กระทบกับสำเนาที่ `parent` ถืออยู่เลย — นี่คือหลักฐานเชิงประจักษ์ที่ยืนยันตาราง
เปรียบเทียบ `MAP_SHARED` vs `MAP_PRIVATE` ใน 36.2

### เทียบกับ System V Shared Memory (Part 30)

| | System V (`shmget`/`shmat`) | POSIX `shm_open` | `mmap` + `MAP_ANONYMOUS` |
|---|---|---|---|
| ใช้ระหว่าง process ที่ไม่เกี่ยวข้องกัน | ได้ (ผ่าน key) | ได้ (ผ่านชื่อใน `/dev/shm`) | **ไม่ได้โดยตรง** (ต้องมี fd ร่วมกันผ่าน `fork()` หรือส่ง fd ผ่าน Unix socket) |
| ใช้ระหว่าง parent-child หลัง `fork()` | ได้ แต่ setup ซับซ้อนกว่า | ได้ | **ง่ายที่สุด** ไม่ต้องตั้งชื่อ ไม่ต้องจัดการ cleanup ไฟล์ใดๆ |
| Cleanup เมื่อเลิกใช้ | ต้อง `shmctl(IPC_RMID)` เอง | ต้อง `shm_unlink()` เอง | ไม่มีทรัพยากรค้างเลย เพราะไม่ผูกกับไฟล์/key ใดๆ หายไปพร้อม process |

สรุป: ถ้า process ที่ต้องการแชร์ memory มีความสัมพันธ์แบบ parent-child (สร้างผ่าน
`fork()` โดยตรง) `mmap()` แบบ anonymous คือทางเลือกที่ **เรียบง่ายที่สุด** ไม่ต้องจัดการ
key หรือชื่อใดๆ เลย

---

## 36.7 ข้อควรระวัง: SIGBUS เมื่อเข้าถึงเกินขอบเขตไฟล์ (Step 287)

นี่คืออันตรายที่สำคัญที่สุดของ `mmap()` ที่ต่างจาก `read()`/`write()` โดยสิ้นเชิง: การ
`mmap()` ทำงานเป็น **หน่วยหน้าความจำ (page)** เสมอ ไม่ใช่หน่วยไบต์ ถ้าไฟล์มีขนาด 10 ไบต์
แต่ page size ของระบบคือ 4096 ไบต์ การ `mmap()` ไฟล์นั้นจะได้ memory ทั้ง page (4096 ไบต์)
มา โดยส่วนที่เกินจากไฟล์จริง (ไบต์ที่ 10-4095) จะถูกเคอร์เนล **zero-fill** ให้อัตโนมัติ
อ่านได้ตามปกติไม่มีปัญหา

แต่ถ้าเรา `mmap()` ขอพื้นที่มากกว่า 1 page และเข้าถึง page ที่ **เกินกว่าที่ไฟล์มีข้อมูล
รองรับไปเลย** (ไม่ใช่แค่ปลาย page สุดท้าย แต่เป็น page ถัดไปทั้ง page) จะเกิดสัญญาณ
**`SIGBUS`** (Bus Error) ทันทีที่เข้าถึง — ไม่ใช่ตอนเรียก `mmap()` แต่เป็นตอนที่โปรแกรม
**เข้าถึง** address นั้นจริงๆ (เพราะ `mmap()` ทำงานแบบ lazy — ไม่มีการตรวจสอบใดๆ ล่วงหน้า)

### สาธิตให้เห็นจริง

```c
/* ============================================================
 * ชื่อไฟล์:     mmap_sigbus_demo.c
 * คำอธิบาย:     สาธิตให้เห็นจริงว่าการเข้าถึง memory ที่ mmap ไว้เกิน
 *              ขอบเขตของไฟล์จริงจะทำให้เกิดสัญญาณ SIGBUS (ไม่ใช่
 *              SIGSEGV) และสาธิตวิธี "ดักจับ" สัญญาณนี้ด้วย sigaction
 *              + sigsetjmp/siglongjmp เพื่อให้โปรแกรมไม่ตายทันที
 *              (ในโค้ด production จริง ควรป้องกันไม่ให้เกิดตั้งแต่ต้น
 *              ด้วยการตรวจสอบขนาดไฟล์ก่อนเข้าถึงเสมอ ไม่ใช่พึ่งการ
 *              ดักจับสัญญาณแบบนี้เป็นทางออกหลัก)
 * ============================================================ */
#define _POSIX_C_SOURCE 200809L

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <signal.h>
#include <setjmp.h>
#include <fcntl.h>
#include <unistd.h>
#include <sys/mman.h>
#include <sys/stat.h>

static sigjmp_buf jump_buffer;
static volatile sig_atomic_t sigbus_caught = 0;

/* signal handler สำหรับ SIGBUS โดยเฉพาะ — เมื่อเกิดสัญญาณนี้ เราจะ
 * "กระโดด" กลับไปยังจุดที่ตั้งไว้ด้วย sigsetjmp() แทนที่จะปล่อยให้
 * โปรแกรมถูกฆ่าทิ้งตามพฤติกรรม default ของสัญญาณนี้ */
static void handle_sigbus(int sig) {
    (void)sig;
    sigbus_caught = 1;
    siglongjmp(jump_buffer, 1);
}

int main(void) {
    const char *filename = "sigbus_test.dat";

    /* สร้างไฟล์ทดสอบขนาดเล็กมากโดยตั้งใจ (แค่ 10 ไบต์) */
    int fd = open(filename, O_RDWR | O_CREAT | O_TRUNC, 0644);
    if (fd < 0) {
        perror("open");
        return EXIT_FAILURE;
    }
    if (write(fd, "0123456789", 10) != 10) {
        perror("write");
        close(fd);
        return EXIT_FAILURE;
    }

    /* mmap ขอ map ขนาด 3 หน้าความจำเต็ม (โดยทั่วไป 4096 ไบต์บน x86_64
     * ต่อหน้า รวม 3 หน้า = 12288 ไบต์) ทั้งที่ไฟล์จริงมีแค่ 10 ไบต์
     *
     * ข้อควรเข้าใจสำคัญ: ไบต์ที่อยู่ "ในหน้าแรก" แต่เกิน 10 ไบต์ของไฟล์
     * (คือ byte ที่ 10-4095) จะถูกเคอร์เนล zero-fill ให้อัตโนมัติและ
     * อ่านได้ตามปกติ ไม่เกิด SIGBUS เพราะหน้านั้นยังถือว่า "มีไฟล์รองรับ"
     * อยู่ (แค่ท้ายหน้าเกินขอบไฟล์เฉยๆ) แต่ถ้าเข้าถึงหน้าที่ 2 หรือ 3
     * ซึ่งอยู่ "เกินกว่าหน้าสุดท้ายที่ไฟล์มีข้อมูลรองรับไปเลย" จะเกิด
     * SIGBUS ทันทีที่เข้าถึง (ไม่ใช่ตอนเรียก mmap) */
    long page_size = sysconf(_SC_PAGESIZE);
    size_t map_size = (size_t)page_size * 3;
    char *data = mmap(NULL, map_size, PROT_READ | PROT_WRITE,
                       MAP_SHARED, fd, 0);
    if (data == MAP_FAILED) {
        perror("mmap");
        close(fd);
        return EXIT_FAILURE;
    }

    printf("ไฟล์มีขนาดจริง 10 ไบต์ แต่ mmap ไว้ %zu ไบต์ (3 pages, page=%ld)\n",
           map_size, page_size);
    printf("อ่าน byte แรกๆ (อยู่ในขอบเขตไฟล์จริง): ");
    for (int i = 0; i < 10; i++) {
        putchar(data[i]);
    }
    printf("  <- อ่านได้ปกติ ไม่มีปัญหา\n");

    printf("อ่าน byte ที่ตำแหน่ง %ld (เกินไฟล์ แต่ยังอยู่ใน page แรก): %d "
           "<- ได้ 0 เพราะเคอร์เนล zero-fill ส่วนท้ายของ page ให้\n",
           page_size - 1, data[page_size - 1]);

    /* ติดตั้ง signal handler สำหรับ SIGBUS ก่อนพยายามเข้าถึงพื้นที่
     * นอกขอบเขตไฟล์จริง (เกิน 10 ไบต์แรกไปแล้ว) */
    struct sigaction sa;
    memset(&sa, 0, sizeof(sa));
    sa.sa_handler = handle_sigbus;
    sigemptyset(&sa.sa_mask);
    sigaction(SIGBUS, &sa, NULL);

    long beyond_offset = page_size * 2;   /* อยู่ใน page ที่ 3 ซึ่งไฟล์ไม่มีข้อมูลรองรับเลย */
    printf("กำลังพยายามอ่าน byte ที่ตำแหน่ง %ld (อยู่ใน page ที่เกินไฟล์ทั้งหมด)...\n",
           beyond_offset);

    if (sigsetjmp(jump_buffer, 1) == 0) {
        /* ตำแหน่งนี้อยู่ใน page ที่ 3 (index 2) ซึ่งไม่มีส่วนใดของไฟล์
         * จริงมารองรับเลยแม้แต่ไบต์เดียว (ไฟล์มีแค่ 10 ไบต์ ซึ่งอยู่ใน
         * page แรกเท่านั้น) เมื่อเข้าถึง page นี้ เคอร์เนลไม่มีข้อมูลจะ
         * เอามาทำ page fault handling ให้ได้ จึงยิง SIGBUS ออกมาทันที */
        char boom = data[beyond_offset];
        printf("อ่านได้ค่า: %d (ไม่ควรมาถึงบรรทัดนี้ถ้า SIGBUS เกิดขึ้นจริง)\n",
               boom);
    } else {
        printf("*** ดักจับ SIGBUS ได้สำเร็จ! ***\n");
        printf("สาเหตุ: เข้าถึง offset ที่เกินขนาดไฟล์จริงบนดิสก์\n");
        printf("บทเรียน: การ mmap() ไม่ได้ตรวจสอบขอบเขตไฟล์ตอนเรียกเลย\n");
        printf("         มันจะพังตอนที่เรา *เข้าถึง* memory นั้นจริงๆ เท่านั้น\n");
    }

    munmap(data, map_size);
    close(fd);
    unlink(filename);

    if (!sigbus_caught) {
        fprintf(stderr, "คาดหวังว่าจะเกิด SIGBUS แต่ไม่เกิด — ผิดปกติ\n");
        return EXIT_FAILURE;
    }

    printf("โปรแกรมจบการทำงานอย่างปลอดภัย (ไม่ crash) แม้เจอ SIGBUS\n");
    return EXIT_SUCCESS;
}
```

ผลลัพธ์จริงจากการรัน (ยืนยันแล้วว่า `SIGBUS` เกิดขึ้นจริงและถูกดักจับได้สำเร็จ):

```
ไฟล์มีขนาดจริง 10 ไบต์ แต่ mmap ไว้ 12288 ไบต์ (3 pages, page=4096)
อ่าน byte แรกๆ (อยู่ในขอบเขตไฟล์จริง): 0123456789  <- อ่านได้ปกติ ไม่มีปัญหา
อ่าน byte ที่ตำแหน่ง 4095 (เกินไฟล์ แต่ยังอยู่ใน page แรก): 0 <- ได้ 0 เพราะเคอร์เนล zero-fill ส่วนท้ายของ page ให้
กำลังพยายามอ่าน byte ที่ตำแหน่ง 8192 (อยู่ใน page ที่เกินไฟล์ทั้งหมด)...
*** ดักจับ SIGBUS ได้สำเร็จ! ***
สาเหตุ: เข้าถึง offset ที่เกินขนาดไฟล์จริงบนดิสก์
บทเรียน: การ mmap() ไม่ได้ตรวจสอบขอบเขตไฟล์ตอนเรียกเลย
         มันจะพังตอนที่เรา *เข้าถึง* memory นั้นจริงๆ เท่านั้น
โปรแกรมจบการทำงานอย่างปลอดภัย (ไม่ crash) แม้เจอ SIGBUS
```

ถ้า **ไม่** ติดตั้ง signal handler ดักจับ `SIGBUS` ไว้เลย พฤติกรรม default ของสัญญาณนี้คือ
**ฆ่าโปรแกรมทิ้งทันที** (คล้ายกับ `SIGSEGV`) ต่างจาก `SIGSEGV` ตรงที่ `SIGBUS` มักเกิดจาก
สาเหตุเฉพาะของ memory-mapped I/O เช่น เข้าถึงเกินขอบเขตไฟล์ที่ map ไว้ หรือไฟล์ที่ map
ไว้ถูกไฟล์อื่นทำให้เล็กลง (เช่นถูก `truncate()` ระหว่างที่ยัง map อยู่) ในขณะที่อีกโปรเซส
ยังพยายามเข้าถึง memory ส่วนนั้นอยู่

### แนวทางป้องกันที่ถูกต้อง (ดีกว่าการดักจับ signal)

การดักจับ `SIGBUS` ด้วย `sigsetjmp`/`siglongjmp` ที่แสดงข้างต้นเป็น **เทคนิคสำหรับ
การศึกษาและกรณีฉุกเฉินเท่านั้น** ในโค้ด production จริง แนวทางที่ถูกต้องกว่าคือ:

1. **ตรวจสอบขนาดไฟล์ด้วย `fstat()` ก่อนเสมอ** และจำกัดการเข้าถึงให้อยู่ในขอบเขตนั้น
2. **`mmap()` ด้วยขนาดที่ตรงกับขนาดไฟล์จริงเท่านั้น** ไม่ map เกินความจำเป็น
3. ถ้าจำเป็นต้องขยายไฟล์ก่อนเขียนเพิ่ม ให้ `ftruncate()` ขยายไฟล์ก่อน แล้วค่อย `mmap()`
   ใหม่ตามขนาดที่ต้องการ (ดูแบบฝึกหัดข้อ 2 ที่สาธิตเทคนิคนี้)

---

## 36.8 เมื่อไหร่ควรใช้ mmap และเมื่อไหร่ควรใช้ read/write (Step 288)

จากทุกอย่างที่เรียนมา สรุปเป็นแนวทางตัดสินใจได้ดังนี้:

| สถานการณ์ | แนะนำให้ใช้ |
|---|---|
| อ่านไฟล์ config ขนาดเล็ก (ไม่กี่ KB) ครั้งเดียวตอนเริ่มโปรแกรม | `fread()` ธรรมดา (ความต่างของความเร็วไม่มีนัยสำคัญ, โค้ดง่ายกว่า) |
| อ่านไฟล์ขนาดใหญ่มาก (หลาย GB) ตามลำดับครั้งเดียว | ทั้งสองวิธีใช้ได้ แต่ `mmap()` ให้ประสิทธิภาพดีกว่าเล็กน้อยและใช้ memory คุ้มกว่า (ไม่ต้องจอง buffer เอง) |
| เข้าถึงข้อมูลในไฟล์ขนาดใหญ่แบบสุ่ม (random access) บ่อยๆ เช่น ฐานข้อมูล, ดัชนีค้นหา | **`mmap()`** ชนะขาด เพราะไม่ต้อง `fseek()` ไปมา อ่าน `data[offset]` ได้ตรงๆ |
| หลายโปรเซสต้องอ่านไฟล์เดียวกันพร้อมกันบ่อยๆ | **`mmap()`** เพราะ page cache ถูกแชร์กันในระดับเคอร์เนลอยู่แล้ว ประหยัด memory รวม |
| แชร์ข้อมูลระหว่าง parent-child process | **`mmap()`** แบบ `MAP_ANONYMOUS` (ดู 36.6) ง่ายกว่า System V/POSIX shared memory มาก |
| เขียนไฟล์แบบ append (เพิ่มข้อมูลต่อท้ายเรื่อยๆ ไม่รู้ขนาดสุดท้ายล่วงหน้า) | `write()`/`fwrite()` ธรรมดา เพราะ `mmap()` ต้องรู้ขนาดล่วงหน้าและ resize ยุ่งยากกว่า |
| ไฟล์ที่มีขนาดเปลี่ยนแปลงบ่อยระหว่างการใช้งาน | `read()`/`write()` ธรรมดา ปลอดภัยกว่า (หลีกเลี่ยงความเสี่ยง `SIGBUS`) |
| ต้องการควบคุม error handling อย่างละเอียด ไม่อยากยุ่งกับ signal | `read()`/`write()` ธรรมดา (error กลับมาเป็น return code ปกติ ไม่ใช่ signal) |

กฎง่ายๆ ที่จำได้เสมอ: **`mmap()` คุ้มค่าที่สุดเมื่อไฟล์มีขนาดใหญ่และมีการเข้าถึงซ้ำๆ
หรือแบบสุ่ม** ถ้าเป็นการอ่าน/เขียนไฟล์เล็กๆ ครั้งเดียวแบบเรียงลำดับ ความซับซ้อนที่เพิ่มขึ้น
ของ `mmap()` (ต้องระวัง `SIGBUS`, ต้องจัดการ page alignment, ต้อง `munmap()` เอง) มักไม่
คุ้มกับประโยชน์ที่ได้

---

## ข้อผิดพลาดที่พบบ่อย (Common Pitfalls)

1. **เช็ก error ของ `mmap()` ด้วย `== NULL` แทนที่จะเป็น `== MAP_FAILED`** — `mmap()`
   ที่ล้มเหลวจะคืนค่าคงที่พิเศษ `MAP_FAILED` (ซึ่งเป็น `(void *)-1` ไม่ใช่ `(void *)0`)
   ถ้าเช็กผิดเป็น `if (data == NULL)` โปรแกรมจะไม่ตรวจพบว่า `mmap()` ล้มเหลว แล้วเอา
   pointer ที่ไม่ถูกต้องไปใช้ต่อ ทำให้เกิด crash ที่จุดอื่นซึ่ง debug ยากกว่ามาก

2. **ลืมว่า `mmap()` ไฟล์ขนาด 0 ไบต์ทำไม่ได้** — จะได้ error `EINVAL` ทันที (ยืนยันแล้ว
   จากการทดสอบจริง) ถ้าต้องการสร้างไฟล์ใหม่ทั้งหมดผ่าน `mmap()` ต้อง `ftruncate()`
   ขยายขนาดไฟล์ให้มากกว่า 0 ก่อนเสมอ (ดูแบบฝึกหัดข้อ 2)

3. **ลืมใส่ `#define _POSIX_C_SOURCE 200809L` ก่อน `#include` เมื่อคอมไพล์ด้วย
   `-std=c17`** — มาตรฐาน ISO C ล้วนๆ ไม่รวมฟังก์ชันของ POSIX อย่าง `mmap()`,
   `clock_gettime()`, `CLOCK_MONOTONIC` เข้ามาให้อัตโนมัติ ถ้าลืมจะเจอ warning
   `implicit declaration of function` หรือ error `undeclared` ตามที่พบจริงระหว่างพัฒนา
   ตัวอย่างในบทเรียนนี้

4. **สับสนระหว่าง `MAP_SHARED` กับ `MAP_PRIVATE`** — ถ้าต้องการแก้ไขไฟล์จริงหรือแชร์
   memory ระหว่าง process แต่ดันใช้ `MAP_PRIVATE` การเปลี่ยนแปลงจะหายไปเงียบๆ (ไม่มี
   error ใดๆ แจ้งเตือนเลย) เพราะ copy-on-write ทำให้แต่ละ process เห็นแค่สำเนาของ
   ตัวเอง เป็นบั๊กที่ตรวจจับยากมากเพราะโปรแกรม "ดูเหมือนทำงานได้" ไม่ crash แต่ผลลัพธ์
   ผิดจากที่คาดหวัง

5. **เข้าถึง memory ที่ mmap ไว้เกินขอบเขตไฟล์จริง (ในหน้าที่ไฟล์ไม่รองรับเลย)** — ทำให้
   เกิด `SIGBUS` ตามที่สาธิตจริงใน 36.7 อาการมักเกิดตอนที่ไฟล์มีขนาดเล็กกว่าที่คาดไว้
   (เช่น ไฟล์ที่ถูกโปรแกรมอื่น truncate ระหว่างที่เรายัง map อยู่) ทางป้องกันคือตรวจสอบ
   ขนาดไฟล์จริงด้วย `fstat()` ก่อนกำหนดขนาดที่จะ `mmap()` เสมอ

6. **ลืม `munmap()` เมื่อเลิกใช้งาน** — คล้ายกับการลืม `free()` คู่กับ `malloc()` หรือ
   ลืม `close()` คู่กับ `open()` ถ้าโปรแกรมมีการ `mmap()`/`munmap()` ซ้ำๆ ตลอดอายุการ
   ทำงาน (เช่น เปิดไฟล์ใหม่มาประมวลผลทีละไฟล์) การลืม `munmap()` จะทำให้ virtual address
   space ของโปรเซสถูกใช้ไปเรื่อยๆ จนอาจถึงขีดจำกัดในที่สุด

7. **แก้ไข memory ที่ map แบบ `PROT_READ` อย่างเดียว (ไม่มี `PROT_WRITE`)** — จะได้
   `SIGSEGV` ทันที (ไม่ใช่ `SIGBUS`) เพราะนี่คือการละเมิดสิทธิ์การเข้าถึง memory
   โดยตรง ไม่เกี่ยวกับขอบเขตไฟล์เลย ต้องแยกความแตกต่างระหว่างสองสัญญาณนี้ให้ออก:
   `SIGSEGV` = ละเมิดสิทธิ์การเข้าถึง (permission), `SIGBUS` = เข้าถึง address ที่ไม่มี
   physical memory หรือไฟล์รองรับจริง (alignment/backing store)

---

## แบบฝึกหัดท้ายบท

1. เขียนโปรแกรมนับจำนวนบรรทัด (newline) ในไฟล์ขนาดใหญ่ด้วย `mmap()` แทนการอ่านทีละ
   บรรทัดด้วย `fgets()` แล้วเปรียบเทียบความเร็วกับ `wc -l`
2. สร้างไฟล์ใหม่ทั้งหมด (ไม่ใช่แก้ไขไฟล์เดิม) ผ่าน `mmap()` เท่านั้น โดยใช้ `ftruncate()`
   ขยายขนาดไฟล์ก่อน `mmap()` (แก้ปัญหา "mmap ไฟล์ขนาด 0 ไบต์ไม่ได้" จากข้อผิดพลาดที่
   พบบ่อยข้อ 2)
3. ปรับ `mmap_shared_fork.c` ให้มี child process มากกว่า 1 ตัว (เช่น 3 ตัว) ที่แต่ละตัว
   เพิ่มค่า `counter` ขึ้น 1000 ครั้ง แล้วสังเกตว่าเกิด race condition หรือไม่ถ้าไม่มี
   การป้องกัน จากนั้นแก้ไขด้วย `pthread_mutex_t` ที่สร้างใน shared memory เดียวกัน
   (ต้องตั้งค่า `pthread_mutexattr_setpshared()` เป็น `PTHREAD_PROCESS_SHARED`)
4. เขียนโปรแกรมเปรียบเทียบความเร็วการ **เขียน** ไฟล์ขนาดใหญ่ด้วย `mmap()` (แก้ไขผ่าน
   pointer แล้ว `msync()`) เทียบกับการเขียนด้วย `fwrite()` ตามปกติ
5. จงทดลองเขียนโปรแกรมที่ทำให้เกิด `SIGBUS` โดย **ไม่** ดักจับสัญญาณเลย สังเกต exit
   code ที่ได้จาก `echo $?` หลังโปรแกรม crash แล้วอธิบายว่าทำไมได้ค่านั้น (ใบ้: ดู
   ความสัมพันธ์ระหว่างเลข signal กับ exit code ที่เคยเรียนใน Part 28)
6. เขียนโปรแกรมที่เปิดไฟล์ขนาดใหญ่มาก (เช่น 1 GB) ด้วย `mmap()` แล้วใช้
   `madvise(addr, length, MADV_SEQUENTIAL)` บอกใบ้เคอร์เนลว่าจะเข้าถึงข้อมูลตามลำดับ
   เปรียบเทียบความเร็วกับกรณีที่ไม่เรียก `madvise()` เลย (ค้นคว้าเพิ่มเติมเกี่ยวกับ
   `madvise()` และ flag อื่นๆ เช่น `MADV_RANDOM`, `MADV_WILLNEED`)

### แนวทางเฉลยข้อ 1 (นับจำนวนบรรทัดด้วย mmap)

```c
/* ตัวอย่างเฉลย: นับจำนวนบรรทัด (newline) ในไฟล์ด้วย mmap
 * เปรียบเทียบกับการอ่านทีละบรรทัดด้วย fgets ที่ต้อง copy ข้อมูลผ่าน
 * buffer ซ้ำไปมา วิธีนี้แค่วน pointer หา '\n' ตรงๆ ใน memory ที่ map ไว้ */
#define _POSIX_C_SOURCE 200809L

#include <stdio.h>
#include <stdlib.h>
#include <fcntl.h>
#include <unistd.h>
#include <sys/mman.h>
#include <sys/stat.h>

int main(int argc, char *argv[]) {
    if (argc != 2) {
        fprintf(stderr, "usage: %s <filename>\n", argv[0]);
        return EXIT_FAILURE;
    }

    int fd = open(argv[1], O_RDONLY);
    if (fd < 0) {
        perror("open");
        return EXIT_FAILURE;
    }

    struct stat st;
    if (fstat(fd, &st) < 0) {
        perror("fstat");
        close(fd);
        return EXIT_FAILURE;
    }
    size_t filesize = (size_t)st.st_size;

    if (filesize == 0) {
        printf("0\n");
        close(fd);
        return EXIT_SUCCESS;
    }

    const char *data = mmap(NULL, filesize, PROT_READ, MAP_PRIVATE, fd, 0);
    if (data == MAP_FAILED) {
        perror("mmap");
        close(fd);
        return EXIT_FAILURE;
    }

    long line_count = 0;
    for (size_t i = 0; i < filesize; i++) {
        if (data[i] == '\n') {
            line_count++;
        }
    }
    /* ถ้าไฟล์ไม่ได้จบด้วย newline แต่ยังมีเนื้อหาค้างอยู่ ให้นับเป็น
     * อีกหนึ่งบรรทัด (ต่างจาก wc -l เล็กน้อยที่นับเฉพาะบรรทัดที่จบด้วย
     * newline สมบูรณ์เท่านั้น) */
    if (data[filesize - 1] != '\n') {
        line_count++;
    }

    printf("%ld\n", line_count);

    munmap((void *)data, filesize);
    close(fd);
    return EXIT_SUCCESS;
}
```

ทดสอบจริง:

```bash
$ printf "line1\nline2\nline3\n" > test1.txt
$ ./ex_linecount test1.txt
3
$ wc -l < test1.txt
3

$ printf "line1\nline2\nline3" > test2.txt   # ไม่มี newline ท้ายสุด
$ ./ex_linecount test2.txt
3
$ wc -l < test2.txt
2
```

สังเกตว่าผลลัพธ์ตรงกับ `wc -l` เมื่อไฟล์จบด้วย newline อย่างถูกต้อง แต่ต่างกันเล็กน้อย
เมื่อไฟล์ไม่มี newline ท้ายสุด (บรรทัดสุดท้ายที่ไม่สมบูรณ์) ซึ่งเป็นพฤติกรรมที่ตั้งใจ
ออกแบบไว้ตามที่ comment อธิบาย ไม่ใช่บั๊ก

### แนวทางเฉลยข้อ 2 (สร้างไฟล์ใหม่ด้วย ftruncate + mmap)

```c
/* ตัวอย่างเฉลย: สร้างไฟล์ใหม่ทั้งหมด (ไม่ใช่แก้ไขไฟล์เดิม) ด้วย mmap
 * ประเด็นสำคัญคือไฟล์ใหม่มีขนาด 0 ไบต์ ซึ่ง mmap "ห้าม" map ไฟล์ขนาด
 * 0 ไบต์ (จะได้ EINVAL ทันที) จึงต้องขยายขนาดไฟล์ด้วย ftruncate() ก่อน
 * เสมอ แล้วค่อย mmap ตามขนาดที่ต้องการใช้งาน */
#define _POSIX_C_SOURCE 200809L

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <fcntl.h>
#include <unistd.h>
#include <sys/mman.h>

int main(int argc, char *argv[]) {
    if (argc != 2) {
        fprintf(stderr, "usage: %s <output-filename>\n", argv[0]);
        return EXIT_FAILURE;
    }

    const char message[] = "สร้างไฟล์นี้ทั้งหมดผ่าน mmap ไม่มีการเรียก fwrite() เลย\n";
    size_t message_len = sizeof(message) - 1;   /* ไม่นับ '\0' ท้ายสุด */

    /* O_CREAT: สร้างไฟล์ใหม่ถ้ายังไม่มี, O_RDWR: เปิดแบบอ่าน-เขียน
     * เพราะจะ mmap ด้วย PROT_WRITE ต่อ, โหมด 0644 = rw-r--r-- */
    int fd = open(argv[1], O_RDWR | O_CREAT | O_TRUNC, 0644);
    if (fd < 0) {
        perror("open");
        return EXIT_FAILURE;
    }

    /* ftruncate(): ขยาย (หรือย่อ) ขนาดไฟล์ให้เป็นตามที่ต้องการ โดยยังไม่
     * ต้องเขียนข้อมูลใดๆ ลงไปเลย พื้นที่ที่ขยายใหม่จะถูกเติมด้วยไบต์ 0
     * โดยอัตโนมัติ ขั้นตอนนี้จำเป็นเพราะ mmap() ไม่สามารถ map ไฟล์ที่มี
     * ขนาด 0 ไบต์ได้เลย และไม่สามารถขยายขนาดไฟล์ให้ใหญ่ขึ้นเองระหว่างที่
     * map ค้างอยู่ได้ด้วย */
    if (ftruncate(fd, (off_t)message_len) < 0) {
        perror("ftruncate");
        close(fd);
        return EXIT_FAILURE;
    }

    char *data = mmap(NULL, message_len, PROT_READ | PROT_WRITE,
                       MAP_SHARED, fd, 0);
    if (data == MAP_FAILED) {
        perror("mmap");
        close(fd);
        return EXIT_FAILURE;
    }

    memcpy(data, message, message_len);

    if (msync(data, message_len, MS_SYNC) < 0) {
        perror("msync");
    }

    munmap(data, message_len);
    close(fd);

    printf("สร้างไฟล์ %s ขนาด %zu ไบต์ผ่าน mmap สำเร็จ\n", argv[1], message_len);
    return EXIT_SUCCESS;
}
```

ทดสอบจริง (ไฟล์ `newfile.txt` ไม่มีอยู่ก่อนเลย):

```bash
$ ./ex_create_via_mmap newfile.txt
สร้างไฟล์ newfile.txt ขนาด 134 ไบต์ผ่าน mmap สำเร็จ

$ cat newfile.txt
สร้างไฟล์นี้ทั้งหมดผ่าน mmap ไม่มีการเรียก fwrite() เลย

$ wc -c newfile.txt
134 newfile.txt
```

และถ้าลองข้าม `ftruncate()` แล้ว `mmap()` ไฟล์ที่เพิ่งสร้างด้วย `O_CREAT` ตรงๆ (ขนาด
ยังเป็น 0 ไบต์) จะได้ error ทันที ยืนยันจากการทดสอบจริง:

```
mmap on 0-byte file: Invalid argument
```

นี่คือหลักฐานที่ยืนยันว่า `ftruncate()` เป็นขั้นตอนที่ **จำเป็น** ไม่ใช่ทางเลือก เมื่อ
ต้องการสร้างไฟล์ใหม่ทั้งหมดผ่าน `mmap()`

---

## สรุปท้ายบท

ใน Part นี้เราได้:

- เข้าใจกลไกเบื้องหลังของ `mmap()` ว่าต่างจาก `fread()`/`read()` อย่างไรในระดับ
  page cache ของเคอร์เนล และทำไมจึงเรียกว่าเป็นเทคนิค "zero-copy I/O"
- เรียนรู้ความหมายและการเลือกใช้ `PROT_READ`/`PROT_WRITE` และ `MAP_SHARED`/`MAP_PRIVATE`
  อย่างถูกต้อง พร้อมหลักฐานเชิงประจักษ์ที่ยืนยันความแตกต่างของทั้งสอง flag
- เขียนโปรแกรมอ่านไฟล์ขนาดใหญ่ด้วย `mmap()` และวัดผลเปรียบเทียบกับ `fread()` อย่าง
  ยุติธรรมโดยควบคุมปัจจัยเรื่อง page cache ของเคอร์เนล
- แก้ไขไฟล์โดยตรงผ่าน pointer ด้วย `mmap()` และ `MS_SYNC`/`MS_ASYNC` ผ่าน `msync()`
- ใช้ `mmap()` แบบ `MAP_ANONYMOUS | MAP_SHARED` สร้าง shared memory ระหว่าง
  parent-child process ได้ง่ายกว่า System V shared memory จาก Part 30 มาก
- สาธิตและดักจับ `SIGBUS` ได้จริงเมื่อเข้าถึง memory เกินขอบเขตไฟล์ พร้อมเข้าใจ
  แนวทางป้องกันที่ถูกต้องกว่าการพึ่งพา signal handler
- มีเกณฑ์ตัดสินใจที่ชัดเจนว่าเมื่อไหร่ควรใช้ `mmap()` และเมื่อไหร่ควรใช้ `read()`/`write()`
  แบบดั้งเดิมในงานจริง

ใน **Part 37** เราจะเปลี่ยนโฟกัสไปที่เครื่องมือที่สำคัญที่สุดอย่างหนึ่งของโปรแกรมเมอร์ C
นั่นคือ **GDB (GNU Debugger)** — วิธี debug โปรแกรมที่มีบั๊กซับซ้อน (null pointer
dereference, off-by-one, segmentation fault) แบบทีละขั้นตอนด้วยเครื่องมือระดับมืออาชีพ
แทนการเดาบั๊กด้วย `printf()` เพียงอย่างเดียว

**ต่อไป:** [Part 37 — การ Debug ด้วย GDB](./part-037-gdb-debugging.md)
