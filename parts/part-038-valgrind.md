# Part 38: Valgrind และ Memory Leak Detection (Step 297–304)

> Module C — Systems Programming ด้วย C บน Linux | Part 38 จาก 125
> Step ที่ครอบคลุมใน Part นี้: Step 297–304
> Part ก่อนหน้า: [Part 37 — การ Debug ด้วย GDB](./part-037-gdb-debugging.md) | Part ถัดไป: [Part 39 — Static และ Dynamic Library](./part-039-static-dynamic-libraries.md)

## เป้าหมายการเรียนรู้ (Learning Objectives)

หลังจบ Part นี้ ผู้เรียนจะสามารถ:

1. อธิบายได้ว่า Valgrind ทำงานอย่างไรเบื้องหลัง (Dynamic Binary Instrumentation) และทำไมมันตรวจจับ
   บั๊กหน่วยความจำได้แม่นยำกว่าการอ่านโค้ดด้วยตาเปล่าหรือรอให้โปรแกรม crash เอง
2. ติดตั้งและรัน Valgrind (เครื่องมือ **Memcheck**) กับโปรแกรม C จริงได้อย่างถูกต้อง
3. ระบุและแก้ไขบั๊กหน่วยความจำ 5 ประเภทหลักที่ Valgrind ตรวจจับได้: **Memory Leak**,
   **Use-After-Free**, **Heap Buffer Overflow**, **Use of Uninitialized Value**, และ **Double Free**
4. อ่านและตีความ **Stack Trace** ที่ Valgrind รายงาน เพื่อหาตำแหน่งบรรทัดที่แท้จริงของบั๊ก
5. แยกความแตกต่างระหว่าง leak ทั้ง 4 ประเภท (definitely / indirectly / possibly lost / still
   reachable) และรู้ว่าประเภทไหนต้องแก้ก่อนเป็นอันดับแรก
6. ใช้ flag สำคัญของ Valgrind อย่างมีประสิทธิภาพ (`--leak-check=full`, `--show-leak-kinds=all`,
   `--track-origins=yes`)
7. เปรียบเทียบ Valgrind กับ AddressSanitizer (ASan) ทั้งด้านความเร็ว วิธีใช้งาน และสถานการณ์ที่
   ควรเลือกใช้แต่ละตัว
8. วางกฎเกณฑ์ส่วนตัว (Personal Workflow) ในการรัน Valgrind กับทุกโปรเจกต์ C ก่อนส่งงานหรือ
   deploy จริง

---

## 38.1 Valgrind คืออะไร และทำงานอย่างไร (Step 297)

ตลอด Part 8–36 ที่ผ่านมา เราเขียนโปรแกรม C ที่จัดการหน่วยความจำเองด้วยมือมาตลอด (`malloc`,
`free`, pointer, array) ปัญหาคือ **บั๊กหน่วยความจำส่วนใหญ่ไม่ทำให้โปรแกรม crash ทันที** — โปรแกรม
อาจรันผ่านได้ปกติ ให้ผลลัพธ์ที่ "ดูถูกต้อง" แต่แอบกิน RAM เพิ่มขึ้นเรื่อยๆ ทุกครั้งที่รัน (memory
leak) หรือแอบอ่าน/เขียนหน่วยความจำที่ไม่ใช่ของตัวเอง (undefined behavior) แล้ว **บางครั้งเท่านั้น**
ที่จะ crash แบบสุ่มในเงื่อนไขที่ทำซ้ำไม่ได้ — บั๊กประเภทนี้คือฝันร้ายของโปรแกรมเมอร์ C ทุกคน
และเป็นเหตุผลที่ทำให้ GDB (Part 37) เพียงอย่างเดียวไม่พอ เพราะ GDB ช่วยตอนโปรแกรม crash
**แล้ว** แต่ไม่ได้บอกว่า "ตอนไหนกันแน่ที่เราทำผิดกฎการใช้หน่วยความจำ"

**Valgrind** คือชุดเครื่องมือวิเคราะห์โปรแกรมที่พัฒนาโดย Julian Seward เริ่มเผยแพร่ตั้งแต่ปี 2002
เครื่องมือย่อยที่ใช้บ่อยที่สุดและเป็นค่า default คือ **Memcheck** ซึ่งจะ**จับตาดูทุกการใช้งาน
หน่วยความจำของโปรแกรมแบบ real-time ขณะรันจริง** และรายงานทันทีเมื่อพบการกระทำที่ผิดกฎ

### หลักการทำงาน: Dynamic Binary Instrumentation (DBI)

Valgrind ไม่ได้รันโปรแกรมของเราตรงๆ บน CPU เหมือนปกติ แต่ทำงานผ่านขั้นตอนที่เรียกว่า
**Dynamic Binary Instrumentation**:

```
โปรแกรมปกติ:        CPU รัน machine code ของเราโดยตรง

ภายใต้ Valgrind:     Valgrind แปลง machine code ของเราเป็นภาษากลาง (IR)
                     → แทรกโค้ดตรวจสอบเพิ่มเข้าไปรอบๆ ทุกคำสั่งที่แตะหน่วยความจำ
                     → แปลงกลับเป็น machine code แล้วค่อยรันบน CPU จริง
                     → เก็บ "shadow memory" คู่ขนานที่จำสถานะของทุกไบต์ในหน่วยความจำจริง
                       (ไบต์นี้ถูกจองหรือยัง? ถูกกำหนดค่าแล้วหรือยัง? ใครเป็นคน malloc?)
```

เพราะต้องแปลโค้ดและตรวจสอบทุกการเข้าถึงหน่วยความจำแบบนี้ **โปรแกรมภายใต้ Valgrind จะรันช้าลง
มาก** (มักช้าลง 10–50 เท่าเมื่อเทียบกับการรันปกติ — เราจะวัดตัวเลขจริงในหัวข้อ 38.8) แต่แลกมาด้วย
ความแม่นยำที่สูงมาก: Valgrind ตรวจจับได้แม้ในกรณีที่บั๊กไม่ได้ทำให้โปรแกรม crash เลยแม้แต่ครั้งเดียว

### เครื่องมือย่อยอื่นๆ ใน Valgrind Suite

Valgrind ไม่ได้มีแค่ Memcheck แต่เป็น "กรอบงาน" ที่รวมเครื่องมือหลายตัวไว้ด้วยกัน:

| เครื่องมือ | หน้าที่ | เรียนใน Part |
|---|---|---|
| **Memcheck** (default) | ตรวจจับบั๊กหน่วยความจำ (leak, use-after-free, overflow, ฯลฯ) | Part นี้ |
| **Helgrind** / **DRD** | ตรวจจับ Race Condition ในโปรแกรม multi-thread | อ้างอิงเพิ่มเติมจาก Part 32 |
| **Cachegrind** | วิเคราะห์การใช้งาน CPU Cache | อ้างอิงเพิ่มเติมจาก Part 87 |
| **Callgrind** | Profiling: นับจำนวนครั้งที่แต่ละฟังก์ชันถูกเรียกและเวลาที่ใช้ | อ้างอิงเพิ่มเติมจาก Part 86 |
| **Massif** | วัดการใช้ heap memory ตลอดอายุโปรแกรม (heap profiler) | อ้างอิงเพิ่มเติมจาก Part 86 |

Part นี้จะโฟกัสที่ **Memcheck** ล้วนๆ เพราะเป็นเครื่องมือที่ใช้บ่อยที่สุดและสำคัญที่สุดสำหรับ
โปรแกรมเมอร์ C ทุกคน

### ติดตั้ง Valgrind

```bash
sudo apt update
sudo apt install valgrind -y
valgrind --version
```

```
valgrind-3.22.0
```

> **ข้อจำกัดสำคัญ**: Valgrind รองรับ Linux และ macOS (บางเวอร์ชัน) เท่านั้น **ไม่รองรับ Windows
> โดยตรง** ผู้ใช้ Windows ต้องรันผ่าน WSL2 (ตามที่ตั้งค่าไว้ตั้งแต่ Part 1) จึงจะใช้ Valgrind ได้

---

## 38.2 บั๊กที่ 1: Memory Leak (Step 298)

**Memory Leak** คือการที่โปรแกรม `malloc`/`calloc`/`realloc` จองหน่วยความจำมา แต่ไม่เคย `free`
คืนก่อนที่ pointer ตัวสุดท้ายที่ชี้ไปยังมันจะหายไป (ทบทวนแนวคิด Ownership จาก Part 11) ทำให้
หน่วยความจำก้อนนั้น**ไม่มีทางเข้าถึงได้อีกเลย แต่ก็ไม่ถูกคืนให้ระบบด้วย** ถ้าโปรแกรมรันนาน (เช่น
server ที่รันทิ้งไว้เป็นเดือน) leak เล็กๆ ที่เกิดซ้ำๆ จะสะสมจน RAM หมดในที่สุด

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

typedef struct {
    char *name;
    int   age;
} Person;

static Person *make_person(const char *name, int age) {
    Person *p = malloc(sizeof(*p));
    if (p == NULL) {
        return NULL;
    }
    p->name = malloc(strlen(name) + 1);
    if (p->name == NULL) {
        free(p);
        return NULL;
    }
    strcpy(p->name, name);
    p->age = age;
    return p;
}

/* บั๊ก: free(p) แต่ลืม free(p->name) ก่อน -> ตัว string หลุดลอยไปตลอดกาล */
static void free_person_buggy(Person *p) {
    free(p);
}

int main(void) {
    Person *alice = make_person("Alice", 30);
    printf("สร้าง %s อายุ %d ปี\n", alice->name, alice->age);

    free_person_buggy(alice);

    printf("จบโปรแกรม\n");
    return 0;
}
```

`free_person_buggy` ลืมว่า `p->name` เป็นหน่วยความจำอีกก้อนหนึ่งที่จองแยกต่างหากด้วย `malloc`
เมื่อ `free(p)` เพียงอย่างเดียว struct `Person` ถูกคืน แต่ **string ที่ `p->name` เคยชี้ไปหายไปกับ
ตา** — ไม่มี pointer ตัวไหนในโปรแกรมชี้ไปที่มันอีกแล้ว คอมไพล์ปกติจะไม่มี warning ใดๆ เลย
(compiler ไม่รู้ semantics ของ ownership) และโปรแกรมรันจบได้ปกติไม่ crash:

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 -g -O0 leak_demo.c -o leak_demo
./leak_demo
```

```
สร้าง Alice อายุ 30 ปี
จบโปรแกรม
```

ดูเผินๆ เหมือนไม่มีอะไรผิดพลาด แต่ลองรันผ่าน Valgrind:

```bash
valgrind --leak-check=full --show-leak-kinds=all ./leak_demo
```

ผลลัพธ์จริงที่ได้ (รันจริงบนเครื่องที่ใช้เขียนบทเรียนนี้):

```
==2597== Memcheck, a memory error detector
==2597== Copyright (C) 2002-2022, and GNU GPL'd, by Julian Seward et al.
==2597== Using Valgrind-3.22.0 and LibVEX; rerun with -h for copyright info
==2597== Command: ./leak_demo
==2597==
สร้าง Alice อายุ 30 ปี
จบโปรแกรม
==2597==
==2597== HEAP SUMMARY:
==2597==     in use at exit: 6 bytes in 1 blocks
==2597==   total heap usage: 3 allocs, 2 frees, 4,118 bytes allocated
==2597==
==2597== 6 bytes in 1 blocks are definitely lost in loss record 1 of 1
==2597==    at 0x4846828: malloc (in /usr/libexec/valgrind/vgpreload_memcheck-amd64-linux.so)
==2597==    by 0x10922F: make_person (leak_demo.c:15)
==2597==    by 0x1092BD: main (leak_demo.c:31)
==2597==
==2597== LEAK SUMMARY:
==2597==    definitely lost: 6 bytes in 1 blocks
==2597==    indirectly lost: 0 bytes in 0 blocks
==2597==      possibly lost: 0 bytes in 0 blocks
==2597==    still reachable: 0 bytes in 0 blocks
==2597==         suppressed: 0 bytes in 0 blocks
==2597==
==2597== For lists of detected and suppressed errors, rerun with: -s
==2597== ERROR SUMMARY: 1 errors from 1 contexts (suppressed: 0 from 0)
```

### อ่านผลลัพธ์ทีละส่วน

- **`HEAP SUMMARY`**: สรุปว่าโปรแกรมเรียก malloc/calloc/realloc ไปทั้งหมด 3 ครั้ง (struct
  `Person`, string `"Alice"` และ buffer ภายในของ `printf` เอง) และเรียก `free` แค่ 2 ครั้ง
  เหลือหน่วยความจำค้างอยู่ ณ ตอนจบโปรแกรม 6 bytes ใน 1 block (คือ `"Alice\0"` ความยาว 6 ไบต์)
- **`6 bytes in 1 blocks are definitely lost`**: นี่คือหัวใจของรายงาน — บอกชัดเจนว่าหน่วยความจำ
  ก้อนนี้ **"lost แน่นอน"** (จะอธิบายความหมายของ "definitely" เทียบกับคำอื่นในหัวข้อถัดไป)
- **Stack Trace ใต้บรรทัด leak**: นี่คือส่วนที่มีค่าที่สุด — บอกว่าหน่วยความจำก้อนนี้ถูกจองด้วย
  `malloc` ที่ **`leak_demo.c:15`** ภายในฟังก์ชัน `make_person` ซึ่งถูกเรียกจาก **`main`** ที่
  **`leak_demo.c:31`** — Valgrind ชี้เป้าตำแหน่งที่ **จองหน่วยความจำ** ให้ตรงๆ (ไม่ใช่ตำแหน่งที่
  ควร free ซึ่งเราต้องหาเอง แต่ก็ทำให้ตามรอยกลับไปดูโค้ดได้ง่ายมาก)
- **`ERROR SUMMARY: 1 errors`**: สรุปว่าพบปัญหา 1 รายการทั้งหมด

### ประเภทของ Leak ทั้ง 4 แบบ

Valgrind แยกประเภท leak ออกเป็น 4 แบบตามความรุนแรง:

| ประเภท | ความหมาย | ต้องแก้หรือไม่ |
|---|---|---|
| **definitely lost** | ไม่มี pointer ใดๆ เหลืออยู่เลยที่ชี้ไปยังก้อนนี้ (หรือแม้แต่ชี้เข้าไปตรงกลางก้อน) | **ต้องแก้เสมอ** — คือ leak จริงๆ 100% |
| **indirectly lost** | ก้อนนี้ยังมี pointer ชี้ถึงจาก struct อื่นที่ตัวมันเองเป็น "definitely lost" (เช่น node ลูกของ linked list ที่ node หัว leak ไปแล้ว) | **ต้องแก้เสมอ** — เป็นผลพวงจาก definitely lost ตัวอื่น |
| **possibly lost** | มี pointer เหลืออยู่ แต่ชี้ไปที่ "กลาง" ก้อนหน่วยความจำ ไม่ใช่จุดเริ่มต้นพอดี (มักเกิดจากเทคนิค pointer arithmetic ที่ผิดปกติ) | ควรตรวจสอบ อาจเป็น false positive ก็ได้ |
| **still reachable** | หน่วยความจำยังไม่ได้ `free` ตอนจบโปรแกรม แต่ **ยังมี pointer ที่เข้าถึงได้อยู่จริง** (เช่นตัวแปร global ที่ยังชี้ถึงอยู่จนจบโปรแกรม) | ไม่ใช่บั๊กเสมอไป แต่ควร free ให้ครบเพื่อความสะอาดของโค้ด |

ตัวอย่าง "still reachable" ที่พบบ่อย:

```c
#include <stdlib.h>

static int *global_cache = NULL;

int main(void) {
    global_cache = malloc(sizeof(int) * 10);
    global_cache[0] = 1;
    /* จบโปรแกรมโดยไม่ free(global_cache) แต่ pointer ยัง "เข้าถึงได้" ผ่านตัวแปร global
     * จนถึงบรรทัดสุดท้ายของโปรแกรม -> valgrind จัดเป็น "still reachable" ไม่ใช่ "definitely lost" */
    return 0;
}
```

```
==25303== HEAP SUMMARY:
==25303==     in use at exit: 40 bytes in 1 blocks
==25303==   total heap usage: 1 allocs, 0 frees, 40 bytes allocated
==25303==
==25303== 40 bytes in 1 blocks are still reachable in loss record 1 of 1
==25303==    at 0x4846828: malloc (in /usr/libexec/valgrind/vgpreload_memcheck-amd64-linux.so)
==25303==    by 0x10915A: main (still_reachable.c:6)
==25303==
==25303== LEAK SUMMARY:
==25303==    definitely lost: 0 bytes in 0 blocks
==25303==    indirectly lost: 0 bytes in 0 blocks
==25303==      possibly lost: 0 bytes in 0 blocks
==25303==    still reachable: 40 bytes in 1 blocks
==25303==         suppressed: 0 bytes in 0 blocks
```

สังเกตว่า `ERROR SUMMARY` ของกรณีนี้จะเป็น **`0 errors`** — ค่า default ของ Valgrind ไม่นับ
"still reachable" เป็น error (ต้องเปิดด้วย `--show-leak-kinds=all` ถึงจะเห็นรายละเอียด แต่มันจะยัง
ไม่ทำให้ exit code ของ valgrind เปลี่ยนเป็น error เว้นแต่จะตั้ง flag เพิ่มเติม) เหตุผลคือหน่วยความจำ
แบบนี้ **OS จะคืนให้อัตโนมัติเมื่อโปรแกรมจบการทำงาน** ไม่ได้เป็นอันตรายเท่า "definitely lost"
ที่เกิดขึ้นซ้ำๆ ระหว่างที่โปรแกรมยังรันอยู่ (เช่นใน loop หรือใน server ที่รับ request ไม่หยุด)

> **กฎทองของ Part นี้**: ให้ความสำคัญกับ **definitely lost** และ **indirectly lost** เป็นอันดับ
> แรกเสมอ เพราะเป็น leak ที่ยืนยันแล้ว 100% ส่วน "possibly lost" ให้ตรวจสอบเพิ่มเติม และ
> "still reachable" ไม่จำเป็นต้องแก้ด่วนแต่ควรทำความสะอาดให้ครบเมื่อมีเวลา

---

## 38.3 บั๊กที่ 2: Use-After-Free (Step 299)

**Use-After-Free (UAF)** คือการใช้งาน pointer ที่ **`free` ไปแล้ว** — เป็นหนึ่งในบั๊กที่อันตราย
ที่สุดในภาษา C/C++ เพราะหน่วยความจำที่ถูก free อาจถูก allocator เอาไปให้คนอื่นใช้ต่อได้ทันที
ทำให้การอ่าน/เขียนผ่าน pointer เดิมไปกระทบข้อมูลของคนละส่วนของโปรแกรมโดยไม่รู้ตัว (ในโลกความ
ปลอดภัยไซเบอร์ UAF เป็นช่องโหว่ยอดนิยมที่ถูกใช้โจมตีจริงมากที่สุดประเภทหนึ่ง)

```c
#include <stdio.h>
#include <stdlib.h>

typedef struct {
    int balance;
} Account;

int main(void) {
    Account *acc = malloc(sizeof(*acc));
    if (acc == NULL) {
        return 1;
    }
    acc->balance = 1000;
    printf("ยอดเงินก่อนปิดบัญชี: %d\n", acc->balance);

    free(acc); /* ปิดบัญชี: คืนหน่วยความจำ */

    /* บั๊ก: ใช้ acc ต่อทั้งที่ free ไปแล้ว (Use-After-Free) */
    acc->balance += 500;
    printf("ยอดเงินหลังฝากเพิ่ม (บั๊ก!): %d\n", acc->balance);

    return 0;
}
```

> **หมายเหตุ**: GCC เวอร์ชันใหม่ๆ (13 ขึ้นไป) ฉลาดพอที่จะจับ pattern แบบตรงไปตรงมานี้ได้ตอน
> compile เองแล้ว ด้วย warning `-Wuse-after-free` ซึ่งเป็นข่าวดี — แต่ในโค้ดจริงที่ pointer
> เดินทางผ่านหลายฟังก์ชัน หลาย struct หรือถูก free แบบมีเงื่อนไข (เช่น free ใน branch หนึ่งแต่ไม่
> ใช่อีก branch) compiler มักตามรอยไม่ทันอีกต่อไป **Valgrind ตรวจจับได้ในทุกกรณีเพราะดูจาก
> พฤติกรรมจริงตอนรัน ไม่ใช่การวิเคราะห์โค้ดแบบ static** และนี่คือเหตุผลที่ยังต้องเรียนรู้เครื่องมือ
> นี้แม้ compiler จะเก่งขึ้นเรื่อยๆ ก็ตาม

รันผ่าน Valgrind:

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 -g -O0 uaf_demo.c -o uaf_demo
valgrind ./uaf_demo
```

```
==2598== Memcheck, a memory error detector
==2598== Copyright (C) 2002-2022, and GNU GPL'd, by Julian Seward et al.
==2598== Using Valgrind-3.22.0 and LibVEX; rerun with -h for copyright info
==2598== Command: ./uaf_demo
==2598==
==2598== Invalid read of size 4
==2598==    at 0x1091E7: main (uaf_demo.c:19)
==2598==  Address 0x4a78040 is 0 bytes inside a block of size 4 free'd
==2598==    at 0x484988F: free (in /usr/libexec/valgrind/vgpreload_memcheck-amd64-linux.so)
==2598==    by 0x1091E2: main (uaf_demo.c:16)
==2598==  Block was alloc'd at
==2598==    at 0x4846828: malloc (in /usr/libexec/valgrind/vgpreload_memcheck-amd64-linux.so)
==2598==    by 0x10919E: main (uaf_demo.c:9)
==2598==
==2598== Invalid write of size 4
==2598==    at 0x1091F3: main (uaf_demo.c:19)
==2598==  Address 0x4a78040 is 0 bytes inside a block of size 4 free'd
==2598==    at 0x484988F: free (in /usr/libexec/valgrind/vgpreload_memcheck-amd64-linux.so)
==2598==    by 0x1091E2: main (uaf_demo.c:16)
==2598==  Block was alloc'd at
==2598==    at 0x4846828: malloc (in /usr/libexec/valgrind/vgpreload_memcheck-amd64-linux.so)
==2598==    by 0x10919E: main (uaf_demo.c:9)
==2598==
==2598== Invalid read of size 4
==2598==    at 0x1091F9: main (uaf_demo.c:20)
==2598==  Address 0x4a78040 is 0 bytes inside a block of size 4 free'd
==2598==    at 0x484988F: free (in /usr/libexec/valgrind/vgpreload_memcheck-amd64-linux.so)
==2598==    by 0x1091E2: main (uaf_demo.c:16)
==2598==  Block was alloc'd at
==2598==    at 0x4846828: malloc (in /usr/libexec/valgrind/vgpreload_memcheck-amd64-linux.so)
==2598==    by 0x10919E: main (uaf_demo.c:9)
==2598==
ยอดเงินก่อนปิดบัญชี: 1000
ยอดเงินหลังฝากเพิ่ม (บั๊ก!): 1500
==2598==
==2598== HEAP SUMMARY:
==2598==     in use at exit: 0 bytes in 0 blocks
==2598==   total heap usage: 2 allocs, 2 frees, 4,100 bytes allocated
==2598==
==2598== All heap blocks were freed -- no leaks are possible
==2598==
==2598== For lists of detected and suppressed errors, rerun with: -s
==2598== ERROR SUMMARY: 3 errors from 3 contexts (suppressed: 0 from 0)
```

### จุดสำคัญที่ต้องสังเกต

- Valgrind รายงาน **3 error แยกกัน**: `Invalid read` ตอนอ่าน `acc->balance` เพื่อบวก 500,
  `Invalid write` ตอนเขียนค่าใหม่กลับเข้าไป, และ `Invalid read` อีกครั้งตอน `printf` อ่านค่ามา
  แสดงผล — **ทุกครั้งที่แตะหน่วยความจำที่ free ไปแล้วนับเป็น error แยกกันหมด**
- แต่ละ error มาพร้อม **3 stack trace ซ้อนกัน**:
  1. ตำแหน่งปัจจุบันที่เกิดปัญหา (`at 0x1091E7: main (uaf_demo.c:19)`)
  2. ตำแหน่งที่ `free()` ก้อนนี้ (`by ... free (uaf_demo.c:16)`)
  3. ตำแหน่งที่ `malloc()` ก้อนนี้ตอนแรก (`Block was alloc'd at ... (uaf_demo.c:9)`)

  โครงสร้างนี้ทรงพลังมาก เพราะมันให้ **ประวัติทั้งหมด** ของหน่วยความจำก้อนนี้ในรายงานเดียว —
  ไม่ต้องเดาว่า pointer นี้เคยถูก free ที่ไหน หรือถูกจองมาจากที่ไหน
- ที่น่าสนใจคือโปรแกรมนี้ **รันจบและให้ผลลัพธ์ที่ "ดูเหมือนถูกต้อง"** (`1500`) โดยไม่ crash เลย
  เพราะหน่วยความจำก้อนเล็กๆ ขนาด 4 bytes ที่เพิ่ง free ไปมักยังไม่ถูก allocator เอาไปใช้ซ้ำทันที
  — นี่คือธรรมชาติอันตรายของ Undefined Behavior: **"ดูเหมือนใช้ได้" ไม่ได้แปลว่า "ถูกต้อง"**
  ในโปรแกรมจริงที่ซับซ้อนกว่านี้ หน่วยความจำก้อนเดียวกันอาจถูก allocator เอาไปให้ตัวแปรอื่นใช้
  พอดี ทำให้ค่าที่อ่าน/เขียนผิดเพี้ยนไปเป็นค่าขยะโดยสมบูรณ์ หรือ crash ทันที แล้วแต่จังหวะ

---

## 38.4 บั๊กที่ 3: Heap Buffer Overflow (Step 300)

**Heap Buffer Overflow** คือการอ่านหรือเขียนหน่วยความจำ **เกินขอบเขต** ของก้อนที่ `malloc` จองไว้
ทบทวนจาก Part 7 และ Part 11: array ใน C **ไม่มีการตรวจสอบขอบเขตอัตโนมัติ** เขียนเลยขอบไปเท่าไร
ก็ได้โดยไม่มี error ใดๆ ในทันที (compiler ก็ไม่เตือนเสมอไปโดยเฉพาะเมื่อขนาด array เป็นตัวแปรที่
รู้ค่าตอนรัน ไม่ใช่ค่าคงที่ตอน compile)

```c
#include <stdio.h>
#include <stdlib.h>

int main(void) {
    int n = 5;
    int *scores = malloc(n * sizeof(int));
    if (scores == NULL) {
        return 1;
    }

    /* บั๊ก: วนลูปถึง i <= n แทนที่จะเป็น i < n -> เขียนเกินขอบเขต heap block ไป 1 ช่อง */
    for (int i = 0; i <= n; i++) {
        scores[i] = i * 10;
    }

    int total = 0;
    for (int i = 0; i < n; i++) {
        total += scores[i];
    }
    printf("ผลรวม (ไม่รวมช่องที่เขียนเกิน) = %d\n", total);

    free(scores);
    return 0;
}
```

บั๊กคลาสสิกแบบ **off-by-one**: `for (int i = 0; i <= n; i++)` ทำให้ `i` วิ่งถึง `5` ด้วย ทั้งที่
`scores` มีที่ให้เก็บแค่ index `0` ถึง `4` (5 ช่อง) การเขียน `scores[5]` จึงเขียนเลยขอบไป 1 `int`
(4 bytes บนระบบทั่วไป) พอดี

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 -g -O0 overflow_demo.c -o overflow_demo
valgrind ./overflow_demo
```

```
==2623== Memcheck, a memory error detector
==2623== Copyright (C) 2002-2022, and GNU GPL'd, by Julian Seward et al.
==2623== Using Valgrind-3.22.0 and LibVEX; rerun with -h for copyright info
==2623== Command: ./overflow_demo
==2623==
==2623== Invalid write of size 4
==2623==    at 0x1091EC: main (overflow_demo.c:13)
==2623==  Address 0x4a78054 is 0 bytes after a block of size 20 alloc'd
==2623==    at 0x4846828: malloc (in /usr/libexec/valgrind/vgpreload_memcheck-amd64-linux.so)
==2623==    by 0x1091AC: main (overflow_demo.c:6)
==2623==
ผลรวม (ไม่รวมช่องที่เขียนเกิน) = 100
==2623==
==2623== HEAP SUMMARY:
==2623==     in use at exit: 0 bytes in 0 blocks
==2623==   total heap usage: 2 allocs, 2 frees, 4,116 bytes allocated
==2623==
==2623== All heap blocks were freed -- no leaks are possible
==2623==
==2623== For lists of detected and suppressed errors, rerun with: -s
==2623== ERROR SUMMARY: 1 errors from 1 contexts (suppressed: 0 from 0)
```

ข้อความ **`Address 0x4a78054 is 0 bytes after a block of size 20 alloc'd`** อ่านตรงตัวเลยว่า:
"ตำแหน่งหน่วยความจำที่เขียนอยู่ห่างจาก **ท้าย** ก้อนที่จองไว้ (ขนาด 20 bytes = 5 × `int`)
พอดี 0 bytes" พูดง่ายๆ คือ **เขียนติดกับขอบพอดีแต่เลยไปแล้ว 1 ไบต์แรก** — สั้น กระชับ และชี้เป้า
ตรงจุดกว่าการงมหาด้วยการอ่านโค้ดเองมาก โดยเฉพาะใน array ที่มีขนาดหลักพันหรือหลักล้าน element

> **สังเกต**: โปรแกรมนี้**ไม่ crash และให้ผลลัพธ์ที่ถูกต้องด้วยซ้ำ** (`ผลรวม = 100` ถูกต้องตาม
> ที่ `scores[0..4]` ควรเป็น) เพราะการเขียนเกิน 1 int นั้นตกอยู่ใน "ช่องว่างกันชน" (red zone /
> padding) ที่ allocator เผื่อไว้ ไม่ได้ไปเขียนทับข้อมูลสำคัญของก้อนอื่นพอดี — นี่คือเหตุผลที่บั๊ก
> ประเภทนี้อันตรายมาก เพราะทดสอบแบบธรรมดา (แค่รันดูผลลัพธ์) จะผ่านตลอด แต่พอ deploy ไปนานๆ
> ในสภาพแวดล้อมที่ layout หน่วยความจำต่างออกไปเล็กน้อย (เช่น compile ด้วย flag optimize ต่างกัน,
> ขนาด array ใหญ่ขึ้น) อาจเขียนทับข้อมูลสำคัญจริงๆ แล้ว crash หรือให้ผลลัพธ์ผิดแบบประหลาดทันที

---

## 38.5 บั๊กที่ 4: Use of Uninitialized Value (Step 301)

ย้อนกลับไปที่ Part 1 หัวข้อ 1.5 เราเคยเห็น warning `'x' is used uninitialized` จาก compiler มา
แล้วสำหรับกรณีง่ายๆ ในฟังก์ชันเดียว แต่ในสถานการณ์ที่ซับซ้อนกว่านั้น เช่น ค่าที่มาจาก `malloc`
(ซึ่งไม่ได้เซ็ตค่าเริ่มต้นให้เหมือน `calloc`) แล้วถูกส่งผ่านไปยังอีกฟังก์ชันหนึ่ง compiler มักตามรอย
ไม่ทันอีกต่อไป

```c
#include <stdio.h>
#include <stdlib.h>

static int classify(int score) {
    if (score >= 50) {
        return 1; /* ผ่าน */
    }
    return 0; /* ไม่ผ่าน */
}

int main(void) {
    int *score = malloc(sizeof(int));
    if (score == NULL) {
        return 1;
    }
    /* บั๊ก: ลืมกำหนดค่าเริ่มต้นให้ *score ก่อนใช้งาน */

    if (classify(*score)) {
        printf("นักเรียนสอบผ่าน\n");
    } else {
        printf("นักเรียนสอบไม่ผ่าน\n");
    }

    free(score);
    return 0;
}
```

`malloc` **ไม่รับประกันว่าหน่วยความจำที่คืนมาจะเป็น 0** (ต่างจาก `calloc` ที่รับประกัน — ทบทวนจาก
Part 11) ค่าที่อยู่ในนั้นคือ "ขยะ" ที่หลงเหลือจากการใช้งานหน่วยความจำก้อนนี้ครั้งก่อนหน้า การอ่าน
`*score` ไปใช้ตัดสินใจ (`if (score >= 50)`) โดยไม่กำหนดค่าก่อนคือ **Undefined Behavior** เต็มรูปแบบ

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 -g -O0 uninit_demo.c -o uninit_demo
valgrind --track-origins=yes ./uninit_demo
```

```
==2624== Memcheck, a memory error detector
==2624== Copyright (C) 2002-2022, and GNU GPL'd, by Julian Seward et al.
==2624== Using Valgrind-3.22.0 and LibVEX; rerun with -h for copyright info
==2624== Command: ./uninit_demo
==2624==
==2624== Conditional jump or move depends on uninitialised value(s)
==2624==    at 0x109198: classify (uninit_demo.c:8)
==2624==    by 0x1091DC: main (uninit_demo.c:21)
==2624==  Uninitialised value was created by a heap allocation
==2624==    at 0x4846828: malloc (in /usr/libexec/valgrind/vgpreload_memcheck-amd64-linux.so)
==2624==    by 0x1091BD: main (uninit_demo.c:15)
==2624==
นักเรียนสอบไม่ผ่าน
==2624==
==2624== HEAP SUMMARY:
==2624==     in use at exit: 0 bytes in 0 blocks
==2624==   total heap usage: 2 allocs, 2 frees, 4,100 bytes allocated
==2624==
==2624== All heap blocks were freed -- no leaks are possible
==2624==
==2624== For lists of detected and suppressed errors, rerun with: -s
==2624== ERROR SUMMARY: 1 errors from 1 contexts (suppressed: 0 from 0)
```

ข้อความ **`Conditional jump or move depends on uninitialised value(s)`** คือลายเซ็นคลาสสิกของ
บั๊กประเภทนี้ — แปลว่า "โปรแกรมกำลังจะตัดสินใจ (เช่น `if`) โดยใช้ค่าที่ยังไม่ได้ถูกกำหนด" มันชี้ไปที่
`classify` บรรทัด 8 (`if (score >= 50)`) ซึ่งถูกเรียกจาก `main` บรรทัด 21 พอดี

### ทำไมต้องใช้ `--track-origins=yes`

ลองรันแบบไม่ใส่ flag นี้ดูความแตกต่าง:

```bash
valgrind ./uninit_demo
```

ผลลัพธ์จะแสดงแค่:

```
==2624== Conditional jump or move depends on uninitialised value(s)
==2624==    at 0x109198: classify (uninit_demo.c:8)
==2624==    by 0x1091DC: main (uninit_demo.c:21)
```

**หายไปทั้งบล็อก `Uninitialised value was created by a heap allocation ...`** — โดยค่า default
Valgrind รู้แค่ว่า "ค่าที่ใช้อยู่ยังไม่ได้ถูกกำหนด" แต่ไม่รู้ว่า **มันมาจากไหนตั้งแต่แรก** เพราะการ
ตามรอยที่มาของทุกไบต์ทุกครั้งมี**ต้นทุนด้าน performance สูงมาก** (Valgrind ทำให้โปรแกรมช้าลง
อยู่แล้ว การเปิดฟีเจอร์นี้เพิ่มยิ่งทำให้ช้าขึ้นไปอีกหลายเท่า) จึงถูกปิดเป็นค่า default ไว้ แต่เวลา
debug จริง การรู้ว่า "ค่านี้มาจาก heap allocation ที่บรรทัดไหน" มีประโยชน์มากจนคุ้มที่จะเปิดเพิ่ม
โดยเฉพาะเมื่อค่าที่ยังไม่ถูกกำหนดถูกส่งผ่านไปหลายฟังก์ชันจนยากจะตามรอยด้วยตาเปล่า

> **กฎทองของ Part นี้**: เวลา debug ปัญหา "uninitialised value" ให้เปิด `--track-origins=yes`
> เสมอตั้งแต่แรก จะได้ไม่ต้องรันซ้ำสองรอบ

---

## 38.6 บั๊กที่ 5: Double Free และ Invalid Free (Step 302)

**Double Free** คือการเรียก `free()` กับ pointer ตัวเดียวกัน **สองครั้ง** ซึ่งเป็น Undefined
Behavior ตามมาตรฐาน C ทันที เพราะ allocator ภายในอาจตีความหน่วยความจำที่ free ไปแล้วซ้ำเป็น
คำสั่งทำลายโครงสร้างข้อมูลภายในของตัว allocator เอง (metadata corruption) ซึ่งอาจทำให้ malloc/
free ครั้งต่อๆ ไปในโปรแกรม **ทั้งโปรแกรม** พังตามไปด้วยอย่างไม่คาดคิด

```c
#include <stdio.h>
#include <stdlib.h>

static void cleanup(int *buf) {
    printf("กำลังคืนหน่วยความจำ...\n");
    free(buf);
}

int main(void) {
    int *buf = malloc(10 * sizeof(int));
    if (buf == NULL) {
        return 1;
    }
    buf[0] = 42;
    printf("buf[0] = %d\n", buf[0]);

    cleanup(buf);

    /* บั๊ก: free ซ้ำอีกครั้งที่ main โดยไม่รู้ว่า cleanup() free ไปแล้ว */
    free(buf);

    return 0;
}
```

โค้ดนี้จำลองสถานการณ์ที่พบบ่อยในโปรเจกต์จริง: `cleanup()` เป็นฟังก์ชันที่รับผิดชอบคืนหน่วยความจำ
ให้แล้ว แต่ `main` **ไม่รู้** ว่า `cleanup()` ทำหน้าที่นั้นไปแล้ว จึง `free(buf)` ซ้ำอีกครั้งด้วยความ
สับสนเรื่อง ownership (ยิ่งโค้ดมีหลายไฟล์ หลายฟังก์ชัน ยิ่งเกิดบั๊กแบบนี้ได้ง่าย)

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 -g -O0 doublefree_demo.c -o doublefree_demo
valgrind ./doublefree_demo
```

```
==2648== Memcheck, a memory error detector
==2648== Copyright (C) 2002-2022, and GNU GPL'd, by Julian Seward et al.
==2648== Using Valgrind-3.22.0 and LibVEX; rerun with -h for copyright info
==2648== Command: ./doublefree_demo
==2648==
==2648== Invalid free() / delete / delete[] / realloc()
==2648==    at 0x484988F: free (in /usr/libexec/valgrind/vgpreload_memcheck-amd64-linux.so)
==2648==    by 0x10923C: main (doublefree_demo.c:20)
==2648==  Address 0x4a78040 is 0 bytes inside a block of size 40 free'd
==2648==    at 0x484988F: free (in /usr/libexec/valgrind/vgpreload_memcheck-amd64-linux.so)
==2648==    by 0x1091D3: cleanup (doublefree_demo.c:6)
==2648==    by 0x109230: main (doublefree_demo.c:17)
==2648==  Block was alloc'd at
==2648==    at 0x4846828: malloc (in /usr/libexec/valgrind/vgpreload_memcheck-amd64-linux.so)
==2648==    by 0x1091EC: main (doublefree_demo.c:10)
==2648==
buf[0] = 42
กำลังคืนหน่วยความจำ...
==2648==
==2648== HEAP SUMMARY:
==2648==     in use at exit: 0 bytes in 0 blocks
==2648==   total heap usage: 2 allocs, 3 frees, 4,136 bytes allocated
==2648==
==2648== All heap blocks were freed -- no leaks are possible
==2648==
==2648== For lists of detected and suppressed errors, rerun with: -s
==2648== ERROR SUMMARY: 1 errors from 1 contexts (suppressed: 0 from 0)
```

สังเกต **`HEAP SUMMARY`** อีกครั้ง: `total heap usage: 2 allocs, 3 frees` — จำนวนครั้งที่ free
มากกว่าจำนวนครั้งที่ malloc! ตัวเลขนี้ที่ไม่ matched กันเป็นสัญญาณเตือนเบื้องต้นที่สังเกตได้ทันที
แม้ยังไม่อ่าน error รายละเอียดเลยด้วยซ้ำ ส่วนรายงานหลักก็ให้ประวัติครบเหมือนกรณี Use-After-Free:
ตำแหน่งที่ `free()` ครั้งที่สอง (`main` บรรทัด 20), ตำแหน่งที่ `free()` ครั้งแรก (`cleanup`
บรรทัด 6 เรียกจาก `main` บรรทัด 17), และตำแหน่งที่ `malloc()` ตอนแรก (`main` บรรทัด 10)

### แนวทางป้องกัน Double Free ที่ใช้กันจริง

รูปแบบที่นิยมใช้ป้องกันบั๊กนี้คือ **ตั้ง pointer เป็น `NULL` ทันทีหลัง `free`** เพราะมาตรฐาน C
รับประกันว่า **`free(NULL)` เป็น no-op ที่ปลอดภัยเสมอ** (ไม่ error ไม่ crash):

```c
free(ptr);
ptr = NULL;   /* free(NULL) ในอนาคตจะไม่เกิดอันตรายใดๆ */
```

จะสาธิตรูปแบบนี้แบบเต็มในแบบฝึกหัดข้อ 2 ท้ายบทนี้

---

## 38.7 อ่าน Stack Trace และ Flag สำคัญของ Valgrind (Step 303)

จากตัวอย่างทั้ง 5 แบบที่ผ่านมา จะเห็น pattern ของรายงาน Valgrind ซ้ำๆ กัน มาสรุปวิธีอ่านให้เป็น
ระบบ และรวม flag ที่สำคัญที่สุดไว้ในที่เดียว

### โครงสร้างของ Stack Trace

```
==PID== <ประเภทของปัญหา>
==PID==    at 0xADDRESS: <ฟังก์ชัน> (<ไฟล์>:<บรรทัด>)     <- จุดที่เกิดปัญหาจริง (อ่านก่อนเสมอ)
==PID==    by 0xADDRESS: <ฟังก์ชันที่เรียก> (<ไฟล์>:<บรรทัด>)  <- ผู้เรียกฟังก์ชันข้างบน
==PID==    by 0xADDRESS: ...                                <- ไล่ขึ้นไปเรื่อยๆ จนถึง main
```

อ่านจากบนลงล่าง: บรรทัดแรก (`at`) คือตำแหน่งที่เกิด error จริงๆ บรรทัดถัดๆ ไป (`by`) คือ **call
chain** ไล่ย้อนกลับไปว่าใครเรียกใครมาจนถึงจุดนั้น (เหมือน `backtrace` ใน GDB ที่เรียนจาก Part 37
ทุกประการ — ถ้าคุ้นเคยกับการอ่าน backtrace ของ GDB มาแล้ว จะอ่านของ Valgrind ได้ทันทีโดยไม่ต้อง
เรียนรู้อะไรใหม่)

`==PID==` (เช่น `==2597==`) คือ **Process ID** ของโปรแกรมที่กำลังถูกตรวจสอบ — สังเกตว่าเวลา
copy stack trace ไปแปะที่อื่น (เช่นถามเพื่อนร่วมทีม หรือแปะใน bug report) ไม่จำเป็นต้องสนใจตัวเลข
นี้ เพราะเปลี่ยนไปทุกครั้งที่รันใหม่

### Flag สำคัญที่ต้องรู้จัก

| Flag | ความหมาย |
|---|---|
| `--leak-check=full` | ตรวจสอบ memory leak แบบละเอียด พร้อม stack trace ของแต่ละก้อนที่ leak (ถ้าไม่ใส่ flag นี้ จะเห็นแค่สรุปตัวเลขรวมใน `LEAK SUMMARY` โดยไม่มีรายละเอียด) |
| `--show-leak-kinds=all` | แสดงครบทั้ง 4 ประเภท (definitely/indirectly/possibly lost + still reachable) ปกติ default จะซ่อน "still reachable" ไว้ |
| `--track-origins=yes` | ตามรอยที่มาของ uninitialised value (ตามที่เห็นในหัวข้อ 38.5) — ทำให้ช้าลงมาก ใช้เฉพาะตอน debug ปัญหานี้โดยเฉพาะ |
| `-s` (หรือ `--error-exitcode=N`) | `-s` แสดงสรุป error/suppression เพิ่มเติม, `--error-exitcode=N` ทำให้ Valgrind คืน exit code `N` เมื่อพบ error (มีประโยชน์มากใน CI/CD Pipeline เพื่อทำให้ build fail อัตโนมัติเมื่อพบบั๊ก — จะใช้จริงใน Part 94) |
| `--num-callers=N` | จำนวนบรรทัดใน stack trace ที่จะแสดง (default 12 ชั้น บางครั้งไม่พอสำหรับโค้ดที่เรียกฟังก์ชันซ้อนกันลึกมาก) |
| `--log-file=FILE` | เขียนผลลัพธ์ทั้งหมดลงไฟล์แทนการพิมพ์ออกหน้าจอ (สะดวกเมื่อ output ยาวมาก) |

คำสั่งที่แนะนำให้ใช้เป็นค่าเริ่มต้นสำหรับทุกโปรเจกต์ตลอดหลักสูตรนี้:

```bash
valgrind --leak-check=full --show-leak-kinds=all --track-origins=yes ./program
```

### รันร่วมกับ GDB: `--vgdb=yes`

Valgrind และ GDB (Part 37) ทำงานร่วมกันได้ด้วย โดย Valgrind มีโหมด **vgdb** ที่ทำตัวเป็น "GDB
server" ให้ต่อ GDB เข้ามา debug แบบ interactive ได้ทันทีที่ Valgrind เจอ error (แทนที่จะรอให้
โปรแกรม crash จริงแล้วค่อย attach):

```bash
valgrind --vgdb=yes --vgdb-error=0 ./program
```

flag `--vgdb-error=0` บอกให้ Valgrind **หยุดโปรแกรมทันทีที่เจอ error ตัวแรก** และรอ GDB มา attach
(เปิด terminal อีกบานแล้วรัน `gdb -ex "target remote | vgdb"` ตามที่ Valgrind พิมพ์คำสั่งแนะนำไว้
บนหน้าจอ) เทคนิคนี้มีประโยชน์มากเมื่อต้องการตรวจสอบ **สถานะของตัวแปรทั้งหมด ณ ขณะที่เกิดบั๊ก**
ไม่ใช่แค่ดู stack trace เฉยๆ

### Suppression File เบื้องต้น

บางครั้งไลบรารีระบบ (เช่น driver กราฟิกบางตัว หรือไลบรารีปิดซอร์ส) มี "leak" ที่เป็นความตั้งใจของ
ผู้พัฒนาเอง (เช่น cache ที่ตั้งใจไม่ free จนจบโปรแกรม) ซึ่งไม่ใช่ความรับผิดชอบของเรา และจะทำให้
รายงาน Valgrind ของโปรแกรมเรารกไปด้วย error ที่ไม่เกี่ยวข้อง วิธีแก้คือเขียน **Suppression File**
เพื่อบอก Valgrind ว่า "รายการ error แบบนี้ไม่ต้องรายงาน":

```bash
valgrind --gen-suppressions=all --leak-check=full ./program 2> suppressions.txt
# แก้ไข suppressions.txt เอาเฉพาะรายการที่ไม่เกี่ยวกับโค้ดของเรา แล้วใช้:
valgrind --suppressions=suppressions.txt --leak-check=full ./program
```

ในหลักสูตรนี้ยังไม่จำเป็นต้องใช้ suppression file (เพราะโค้ดทุกตัวอย่างเขียนขึ้นเองทั้งหมด) แต่
เป็นเทคนิคที่จะพบเจอแน่นอนเมื่อทำงานกับโปรเจกต์ระดับ production ที่พึ่งพา third-party library

---

## 38.8 Valgrind vs AddressSanitizer (ASan) (Step 304)

**AddressSanitizer (ASan)** เป็นเครื่องมือตรวจจับบั๊กหน่วยความจำอีกตัวหนึ่งที่ฝังอยู่ใน compiler
โดยตรง (ทั้ง GCC และ Clang) เปิดใช้งานง่ายๆ ด้วย flag `-fsanitize=address` ตอน compile — เราจะ
เรียนรู้ ASan และเพื่อนของมัน (UBSan, ThreadSanitizer) แบบละเอียดใน **Part 96** แต่ในที่นี้จะ
เปรียบเทียบแนวคิดเบื้องต้นเพื่อให้เห็นภาพรวมของระบบนิเวศเครื่องมือ debug หน่วยความจำในโลก C/C++

ลองคอมไพล์ `uaf_demo.c` จากหัวข้อ 38.3 ด้วย ASan แทน Valgrind:

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 -g -O0 -fsanitize=address uaf_demo.c -o uaf_asan
./uaf_asan
```

ผลลัพธ์ (ตัดบางส่วนที่ยาวเกินไปออก):

```
=================================================================
==2684==ERROR: AddressSanitizer: heap-use-after-free on address 0x502000000010 at pc 0x559b3468e314 bp 0x7ffe623e8cd0 sp 0x7ffe623e8cc0
READ of size 4 at 0x502000000010 thread T0
    #0 0x559b3468e313 in main uaf_demo.c:19
    #1 0x7f4f1c62a1c9 in __libc_start_call_main ../sysdeps/nptl/libc_start_call_main.h:58

0x502000000010 is located 0 bytes inside of 4-byte region [0x502000000010,0x502000000014)
freed by thread T0 here:
    #0 0x7f4f1cafc4d8 in free ../../../../src/libsanitizer/asan/asan_malloc_linux.cpp:52
    #1 0x559b3468e2dc in main uaf_demo.c:16

previously allocated by thread T0 here:
    #0 0x7f4f1cafd9c7 in malloc ../../../../src/libsanitizer/asan/asan_malloc_linux.cpp:69
    #1 0x559b3468e25e in main uaf_demo.c:9

SUMMARY: AddressSanitizer: heap-use-after-free uaf_demo.c:19 in main
```

สังเกตว่าข้อมูลที่ได้**คล้ายกับ Valgrind มาก** (ตำแหน่งที่เกิดปัญหา, ตำแหน่งที่ free, ตำแหน่งที่
malloc) เพราะเป้าหมายคือปัญหาเดียวกัน แต่วิธีการเบื้องหลังต่างกันโดยสิ้นเชิง

### ความแตกต่างเชิงเทคนิค

| ประเด็น | Valgrind (Memcheck) | AddressSanitizer (ASan) |
|---|---|---|
| วิธีทำงาน | Dynamic Binary Instrumentation ตอน**รัน** (ไม่ต้อง compile ใหม่) | แทรกโค้ดตรวจสอบตอน**compile** (ต้อง compile ใหม่ด้วย `-fsanitize=address`) |
| ความเร็ว (ทดสอบจริงในบทนี้) | ช้ากว่าปกติราว **40 เท่า** | ช้ากว่าปกติราว **3 เท่า** เท่านั้น |
| การใช้หน่วยความจำ | เพิ่มขึ้นมาก (ต้องเก็บ shadow memory คู่ขนาน) | เพิ่มขึ้นเช่นกัน แต่โดยทั่วไปน้อยกว่า Valgrind |
| ต้องแก้ไข build หรือไม่ | **ไม่ต้อง** — รันกับ binary ที่ compile ปกติได้เลย | **ต้อง** compile ใหม่ด้วย flag พิเศษเสมอ |
| Stack Overflow / Global Buffer Overflow | ตรวจจับได้จำกัดกว่า | ตรวจจับได้ครอบคลุมกว่า (รวม stack และ global variable overflow) |
| รองรับแพลตฟอร์ม | Linux/macOS (ไม่รองรับ Windows โดยตรง) | Linux/macOS/Windows (ผ่าน MSVC หรือ Clang) |
| ความนิยมใน production/CI | นิยมใช้รันแบบ ad-hoc ตอน debug | นิยมฝังเป็นส่วนหนึ่งของ CI pipeline เพราะเร็วพอจะรันทุก commit ได้ |

**ตัวเลขจริงที่วัดได้** จากโปรแกรมทดสอบที่รัน loop คำนวณ 20 ล้านรอบ:

| โหมดการรัน | เวลาที่ใช้ |
|---|---|
| รันปกติ (native) | 0.018 วินาที |
| รันด้วย ASan | 0.058 วินาที (ช้ากว่า ~3 เท่า) |
| รันด้วย Valgrind | 0.753 วินาที (ช้ากว่า ~42 เท่า) |

> **หมายเหตุ**: ตัวเลขนี้ขึ้นกับลักษณะงานของแต่ละโปรแกรมมาก โปรแกรมที่เข้าถึงหน่วยความจำบ่อย
> (memory-intensive) จะเห็น overhead ของ Valgrind สูงกว่าโปรแกรมที่เน้นคำนวณตัวเลขล้วนๆ
> (compute-bound) แต่แนวโน้มที่ **ASan เร็วกว่า Valgrind อย่างชัดเจน** จะเป็นจริงเสมอในทุกกรณี

### เลือกใช้ตัวไหนดี

- **ใช้ Valgrind เมื่อ**: ต้องการตรวจสอบ binary ที่มีอยู่แล้วโดยไม่ต้อง compile ใหม่ (เช่นตรวจสอบ
  โปรแกรมของคนอื่นที่ไม่มี source code), ต้องการเครื่องมือที่ครบชุดในตัวเดียว (Memcheck +
  Helgrind + Cachegrind), หรือทำงานในสภาพแวดล้อมที่ยังไม่รองรับ sanitizer ของ compiler รุ่นใหม่
- **ใช้ ASan เมื่อ**: ต้องการรันเทสต์บ่อยๆ ระหว่างพัฒนา (เร็วกว่ามาก รบกวน workflow น้อยกว่า),
  ต้องการฝังไว้ใน CI/CD ให้รันทุกครั้งที่ push โค้ด (ตามที่จะเรียนใน Part 94), หรือต้องการตรวจจับ
  stack/global buffer overflow ที่ Valgrind ตรวจจับได้จำกัดกว่า

ในทางปฏิบัติ ทีมพัฒนาระดับมืออาชีพจำนวนมาก**ใช้ทั้งสองตัวร่วมกัน**: รัน ASan เป็นส่วนหนึ่งของ
CI ทุก commit เพราะเร็ว และรัน Valgrind เป็นระยะๆ (เช่นก่อน release ใหญ่ หรือตอน debug ปัญหา
ที่ ASan ตรวจไม่พบ) เพื่อความละเอียดที่มากกว่า — ไม่มีเครื่องมือใดที่ "ดีกว่า" อีกตัวในทุกมิติ
เราจะเจาะลึก ASan, UBSan และ ThreadSanitizer พร้อมวิธีติดตั้งใน CI แบบเต็มรูปแบบใน **Part 96**

---

## ข้อผิดพลาดที่พบบ่อย (Common Pitfalls)

1. **ลืมคอมไพล์ด้วย `-g`** — ถ้าไม่มี debug symbol Valgrind จะรายงานแค่ address ดิบๆ
   (`at 0x109198: ??? (in ./program)`) โดยไม่มีชื่อฟังก์ชันหรือเลขบรรทัดให้เลย ทำให้อ่าน stack
   trace แทบไม่ได้ประโยชน์อะไรเลย **ต้องคอมไพล์ด้วย `-g` เสมอเมื่อจะรัน Valgrind** (ไม่จำเป็นต้อง
   ปิด optimization ด้วย `-O0` เสมอไป แต่แนะนำให้ทำระหว่าง debug เพราะ `-O2` ขึ้นไปอาจทำให้
   compiler จัดเรียงโค้ดใหม่จนเลขบรรทัดที่รายงานคลาดเคลื่อนจากโค้ดต้นฉบับได้)
2. **เข้าใจผิดว่า Valgrind แสดง exit code เป็น error เสมอเมื่อเจอปัญหา** — โดย default exit code
   ของ Valgrind คือ exit code ของ**โปรแกรมที่ถูกรัน** ไม่ใช่ของ Valgrind เอง ถ้าต้องการให้
   Valgrind คืนค่า error code ต่างหากเมื่อเจอบั๊ก (สำคัญมากสำหรับ CI/CD) ต้องใส่ flag
   `--error-exitcode=1` (หรือเลขอื่นที่ไม่ใช่ 0) อย่างชัดเจน
3. **ไม่อ่าน `HEAP SUMMARY` ก่อนอ่านรายละเอียด** — ตัวเลข `X allocs, Y frees` เป็นสัญญาณเตือน
   เบื้องต้นที่เร็วที่สุด ถ้า `X != Y` แปลว่ามีบางอย่างผิดปกติแน่นอน (leak หรือ double free) ควร
   เช็คตัวเลขนี้เป็นอันดับแรกก่อนไล่อ่าน error ทีละรายการ
4. **สับสนระหว่าง "definitely lost" กับ "still reachable"** — ไล่แก้ "still reachable" ก่อนโดย
   ไม่สนใจ "definitely lost" ที่ร้ายแรงกว่ามาก ทั้งที่ควรทำตรงกันข้าม (ทบทวนตารางในหัวข้อ 38.2)
5. **รัน Valgrind กับโปรแกรมที่ทำงานหนักมาก (เช่น loop คำนวณหลักพันล้านรอบ) โดยไม่คาดหวังว่ามัน
   จะช้าลงหลายสิบเท่า** — ถ้าโปรแกรมใช้เวลานานผิดปกติภายใต้ Valgrind อย่าเพิ่งตกใจคิดว่า
   Valgrind ค้าง ให้รอ หรือลดขนาด input ลงให้เล็กพอที่จะ debug ได้ในเวลาที่สมเหตุสมผลก่อน
6. **ลืมว่า Valgrind ตรวจจับได้เฉพาะ code path ที่ถูกรันจริงเท่านั้น** — ถ้าโค้ดมีบั๊กอยู่ใน branch
   ที่ input ทดสอบไม่เคยพาไปถึง Valgrind จะไม่มีทางรู้เรื่องบั๊กนั้นเลย การรัน Valgrind ไม่ได้
   แปลว่าโปรแกรม "ปลอดภัย 100%" มันแค่ยืนยันว่า **เท่าที่ทดสอบมา ยังไม่พบปัญหา** เท่านั้น
   (หลักการเดียวกับ Unit Testing ที่จะเรียนใน Part 93 — coverage ที่ไม่ครบคือจุดบอดเสมอ)

---

## แบบฝึกหัดท้ายบท

1. แก้บั๊ก memory leak ในโปรแกรม `leak_demo.c` จากหัวข้อ 38.2 ให้ถูกต้อง (ต้อง `free(p->name)`
   ก่อน `free(p)` เสมอ) แล้วรันผ่าน `valgrind --leak-check=full` ยืนยันว่าได้
   `All heap blocks were freed -- no leaks are possible`
2. แก้บั๊ก double free ในโปรแกรม `doublefree_demo.c` จากหัวข้อ 38.6 ด้วยเทคนิค "ตั้งเป็น NULL
   หลัง free" ที่แนะนำไว้ท้ายหัวข้อนั้น (ต้องแก้ให้ `cleanup()` รับ `int **` แทน `int *` เพื่อ
   แก้ไข pointer ของผู้เรียกได้จริง) แล้วยืนยันด้วย Valgrind ว่า error หายไปหมด
3. เขียนโปรแกรมที่จงใจทำ **Invalid Read** จากการอ่านเลยขอบเขต array บน heap (ตรงข้ามกับตัวอย่าง
   Invalid Write ในหัวข้อ 38.4) แล้วรัน Valgrind ดูว่าข้อความที่ได้ต่างจาก Invalid Write อย่างไร
4. เขียนโปรแกรมที่มี "possibly lost" (คำใบ้: ลองสร้าง pointer ที่ชี้ไปยัง**กลาง** array ที่
   malloc มา แล้วปล่อยให้ pointer ตัวเดียวที่ชี้ไปยัง**จุดเริ่มต้น**ของ array นั้นหลุดขอบเขตไป)
   แล้วสังเกตความแตกต่างจาก "definitely lost" ใน `LEAK SUMMARY`
5. ทดลอง compile โปรแกรม `uninit_demo.c` จากหัวข้อ 38.5 ด้วย `-fsanitize=address
   -fsanitize=undefined` แทน Valgrind แล้วเปรียบเทียบว่า ASan/UBSan ตรวจจับปัญหานี้ได้หรือไม่
   (คำใบ้: ASan เน้นเรื่อง memory bounds และ use-after-free เป็นหลัก การตรวจจับ uninitialized
   value โดยเฉพาะต้องใช้ **MemorySanitizer (MSan)** ซึ่งเป็นเครื่องมือคนละตัว ลองค้นคว้าเพิ่มเติม
   ว่าทำไม ASan ธรรมดาถึงตรวจจับปัญหานี้ได้ไม่ครบเท่า Valgrind หรือ MSan)
6. รันโปรแกรม `overflow_demo.c` จากหัวข้อ 38.4 ด้วย `--num-callers=2` เทียบกับค่า default
   แล้วอธิบายว่า flag นี้ส่งผลต่อรายงานอย่างไรในกรณีนี้ (คำใบ้: ลองนึกถึงโปรแกรมที่มี call chain
   ลึกกว่านี้มาก เช่น 20 ชั้น ว่า flag นี้จะมีผลชัดเจนกว่านี้อย่างไร)

### แนวทางเฉลยข้อ 1

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

typedef struct {
    char *name;
    int   age;
} Person;

static Person *make_person(const char *name, int age) {
    Person *p = malloc(sizeof(*p));
    if (p == NULL) {
        return NULL;
    }
    p->name = malloc(strlen(name) + 1);
    if (p->name == NULL) {
        free(p);
        return NULL;
    }
    strcpy(p->name, name);
    p->age = age;
    return p;
}

/* แก้บั๊ก: free(p->name) ก่อนเสมอ แล้วค่อย free(p) */
static void free_person(Person *p) {
    free(p->name);
    free(p);
}

int main(void) {
    Person *alice = make_person("Alice", 30);
    printf("สร้าง %s อายุ %d ปี\n", alice->name, alice->age);

    free_person(alice);

    printf("จบโปรแกรม\n");
    return 0;
}
```

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 -g -O0 leak_fixed.c -o leak_fixed
valgrind --leak-check=full --show-leak-kinds=all ./leak_fixed
```

ผลลัพธ์จริงที่ได้:

```
==25244== HEAP SUMMARY:
==25244==     in use at exit: 0 bytes in 0 blocks
==25244==   total heap usage: 3 allocs, 3 frees, 4,118 bytes allocated
==25244==
==25244== All heap blocks were freed -- no leaks are possible
==25244==
==25244== For lists of detected and suppressed errors, rerun with: -s
==25244== ERROR SUMMARY: 0 errors from 0 contexts (suppressed: 0 from 0)
```

`3 allocs, 3 frees` ตรงกันพอดี (struct `Person`, string `"Alice"`, และ buffer ภายในของ `printf`
ที่ถูก free เองอัตโนมัติ) และ `0 errors` ยืนยันว่าไม่มีบั๊กหลงเหลือ — จุดสำคัญของเฉลยนี้คือ **ต้อง
free หน่วยความจำตามลำดับย้อนกลับกับที่ malloc เสมอเมื่อมี ownership ซ้อนกันเป็นชั้นๆ**: จองจาก
นอกเข้าใน (struct ก่อน แล้วค่อยจอง field ข้างใน) แต่ free จากในออกนอก (field ข้างในก่อน แล้วค่อย
free struct) มิฉะนั้นจะเข้าถึง field ที่ต้องใช้ตัดสินใจว่าต้อง free อะไรบ้างไม่ได้อีกต่อไป

### แนวทางเฉลยข้อ 2

```c
#include <stdio.h>
#include <stdlib.h>

static void cleanup(int **buf_ptr) {
    printf("กำลังคืนหน่วยความจำ...\n");
    free(*buf_ptr);
    *buf_ptr = NULL; /* กันบั๊ก: ตั้งเป็น NULL ทันทีหลัง free เพื่อให้ free(NULL) ครั้งถัดไปปลอดภัย */
}

int main(void) {
    int *buf = malloc(10 * sizeof(int));
    if (buf == NULL) {
        return 1;
    }
    buf[0] = 42;
    printf("buf[0] = %d\n", buf[0]);

    cleanup(&buf);

    /* free ซ้ำอีกครั้ง แต่ตอนนี้ buf เป็น NULL แล้ว -> free(NULL) คือ no-op ตามมาตรฐาน C */
    free(buf);

    return 0;
}
```

จุดสำคัญของเฉลยนี้คือการเปลี่ยน parameter ของ `cleanup` จาก `int *` เป็น **`int **`**
(pointer-to-pointer ทบทวนจาก Part 9 และ Part 12) เพราะถ้ายังรับแค่ `int *` เหมือนเดิม การตั้ง
`buf_ptr = NULL` ภายในฟังก์ชันจะแก้แค่ **สำเนา local** ของ pointer เท่านั้น ไม่ส่งผลย้อนกลับไปยัง
ตัวแปร `buf` ใน `main` เลย (ทบทวนหลักการ pass-by-value ของ C จาก Part 6) ต้องส่ง **ที่อยู่ของ
ตัวแปร pointer** (`&buf`) เข้าไป แล้ว dereference สองชั้น (`*buf_ptr`) ถึงจะแก้ค่าตัวจริงได้

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 -g -O0 doublefree_fixed.c -o doublefree_fixed
valgrind ./doublefree_fixed
```

ผลลัพธ์จริงที่ได้:

```
buf[0] = 42
กำลังคืนหน่วยความจำ...
==25278== HEAP SUMMARY:
==25278==     in use at exit: 0 bytes in 0 blocks
==25278==   total heap usage: 2 allocs, 2 frees, 4,136 bytes allocated
==25278==
==25278== All heap blocks were freed -- no leaks are possible
==25278==
==25278== ERROR SUMMARY: 0 errors from 0 contexts (suppressed: 0 from 0)
```

`2 allocs, 2 frees` ตรงกันพอดี (เทียบกับ `2 allocs, 3 frees` ในเวอร์ชันที่มีบั๊ก) และ
`0 errors` ยืนยันว่า double free หายไปแล้วอย่างสมบูรณ์ — เทคนิค "NULL หลัง free" นี้เป็นแนวปฏิบัติ
มาตรฐานที่โปรเจกต์ C คุณภาพดีทั่วโลกใช้กัน แม้จะไม่ได้ป้องกันทุกกรณีของ double free (เช่นกรณีที่
มี pointer สองตัวชี้ไปยังก้อนเดียวกันคนละตัวแปร - aliasing - เทคนิคนี้ก็ยังช่วยไม่ได้ ต้องอาศัย
การออกแบบ ownership ที่ชัดเจนตั้งแต่ต้น) แต่ก็ครอบคลุมกรณีที่พบบ่อยที่สุดได้อย่างมีประสิทธิภาพ

---

## สรุปท้ายบท

ใน Part นี้เราได้:

- เข้าใจหลักการทำงานของ Valgrind ผ่าน Dynamic Binary Instrumentation และรู้จักเครื่องมือย่อย
  ต่างๆ ในชุด (Memcheck, Helgrind, Cachegrind, Callgrind, Massif)
- รัน Valgrind (Memcheck) จริงกับโปรแกรมที่มีบั๊กหน่วยความจำ 5 ประเภทหลัก และอ่านผลลัพธ์จริง
  ของแต่ละแบบ: **Memory Leak**, **Use-After-Free**, **Heap Buffer Overflow**, **Use of
  Uninitialized Value**, และ **Double Free**
- แยกแยะ leak ทั้ง 4 ระดับ (definitely / indirectly / possibly lost / still reachable) และรู้ว่า
  ประเภทไหนต้องแก้ก่อน
- อ่าน Stack Trace ของ Valgrind ได้อย่างเป็นระบบ และรู้จัก flag สำคัญ (`--leak-check=full`,
  `--show-leak-kinds=all`, `--track-origins=yes`, `--error-exitcode`)
- เปรียบเทียบ Valgrind กับ AddressSanitizer ทั้งด้านหลักการทำงาน ความเร็ว (วัดตัวเลขจริง) และ
  สถานการณ์ที่ควรเลือกใช้แต่ละตัว พร้อมรู้ว่าจะได้เจาะลึก ASan แบบเต็มรูปแบบใน Part 96

Valgrind คือเครื่องมือที่ควรรันกับ**ทุกโปรเจกต์ C** ก่อนที่จะมั่นใจว่าโค้ดพร้อมใช้งานจริง —
ทักษะนี้จะติดตัวไปตลอดเส้นทางสายงาน Systems Programming ใน **Part 39** เราจะเปลี่ยนโฟกัสจาก
การตรวจจับบั๊กไปสู่การจัดระเบียบโค้ด: **Static และ Dynamic Library** วิธีการแพ็กโค้ด C ที่เขียน
ขึ้นให้กลายเป็นไลบรารีที่โปรแกรมอื่นเรียกใช้ได้โดยไม่ต้องคอมไพล์ source code ซ้ำทุกครั้ง

**ต่อไป:** [Part 39 — Static และ Dynamic Library](./part-039-static-dynamic-libraries.md)
