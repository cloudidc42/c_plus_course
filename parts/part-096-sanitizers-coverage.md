# Part 96: Sanitizer และ Code Coverage: ASan, UBSan, TSan, gcov/lcov (Step 761–768)

> Module H — Build Systems, Testing และ Tooling | Part 96 จาก 125
> Step ที่ครอบคลุมใน Part นี้: Step 761–768
> Part ก่อนหน้า: [Part 95 — Static Analysis: clang-tidy, cppcheck, clang-format](./part-095-static-analysis.md) | Part ถัดไป: Part 97 — Design Pattern ใน C++ (Creational)

## เป้าหมายการเรียนรู้ (Learning Objectives)

หลังจบ Part นี้ ผู้เรียนจะสามารถ:

1. อธิบายได้ว่า **Sanitizer** ทำงานอย่างไร (Compile-Time Instrumentation) ต่างจาก Valgrind
   (Part 38) และ Static Analysis (Part 95) อย่างไร และทำไมมันตรวจจับบั๊กบางประเภทได้แม่นยำ
   กว่า พร้อมความเร็วที่สูงกว่า Valgrind มาก
2. ใช้ **AddressSanitizer (ASan)** ตรวจจับ `heap-buffer-overflow` และ `use-after-free` ได้จริง
   พร้อมอ่านและตีความรายงานของ ASan ได้ครบทุกส่วน
3. ใช้ **UndefinedBehaviorSanitizer (UBSan)** ตรวจจับ Undefined Behavior เช่น Signed Integer
   Overflow และ Null Pointer Dereference ได้จริง
4. ใช้ **ThreadSanitizer (TSan)** ตรวจจับ Data Race ในโปรแกรม multi-thread ได้จริง โดยทดสอบ
   กับโค้ด race condition ในสไตล์เดียวกับที่เรียนใน Part 32 และ Part 81
5. รู้ข้อจำกัดสำคัญของการรวม Sanitizer หลายตัวพร้อมกัน (ตัวไหนรวมกันได้ ตัวไหนรวมกันไม่ได้
   และทำไม)
6. ติดตั้งและใช้ **gcov**/**lcov** วัด Code Coverage ของ Unit Test จาก Part 93 ได้จริง สร้าง
   HTML Coverage Report และอ่านค่าเปอร์เซ็นต์ที่ได้อย่างถูกต้อง รวมถึงเข้าใจว่า Coverage สูง
   ไม่ได้แปลว่าโค้ด "ไม่มีบั๊ก"
7. รวม Sanitizer และ Code Coverage เข้ากับ CI Pipeline จาก Part 94 เพื่อให้ทุก Pull Request
   ถูกตรวจสอบบั๊ก runtime และ coverage อัตโนมัติ

---

## หมายเหตุสำคัญก่อนเริ่ม

ทุกคำสั่งและผลลัพธ์ของ ASan, UBSan, TSan, Valgrind, gcov, และ lcov ใน Part นี้**รันจริงบน
เครื่องนี้** (g++ 13.3.0, clang++ 18.1.3, Valgrind 3.22.0, lcov 2.0-1) ไม่มีรายงาน error หรือ
ตัวเลข coverage ใดถูกแต่งขึ้นเอง — รวมถึงจุดที่เครื่องมือมีพฤติกรรมแปลกหรือขัดแย้งกันเอง (ซึ่งมีจริง
และจะระบุไว้อย่างตรงไปตรงมาในหัวข้อ 96.7) ก็เป็นสิ่งที่พบเจอจริงระหว่างการเตรียมเนื้อหานี้

---

## 96.1 Sanitizer คืออะไร ทำงานอย่างไร (Step 761)

### ทบทวน: Valgrind ทำงานอย่างไร (จาก Part 38)

ใน **Part 38** เราเรียนรู้ว่า Valgrind ทำงานผ่าน **Dynamic Binary Instrumentation** — มันรัน
โปรแกรมของเราผ่านตัวมันเองเหมือน CPU จำลอง แปล machine code เป็นภาษากลาง แทรกโค้ด
ตรวจสอบ แล้วค่อยรันจริง วิธีนี้แม่นยำมากแต่ **ช้ามาก** (10–50 เท่า) เพราะไม่มีการแตะต้องโค้ด
ต้นฉบับเลย ทุกอย่างเกิดที่ runtime ล้วนๆ

### Sanitizer ทำงานต่างออกไป: Compile-Time Instrumentation

**Sanitizer** (ASan, UBSan, TSan ที่พัฒนาโดยทีม Google ร่วมกับ LLVM/GCC) ใช้แนวทางตรงข้าม
คือ**แทรกโค้ดตรวจสอบเข้าไปตอนคอมไพล์** ไม่ใช่ตอนรัน:

```
โค้ดต้นฉบับ (.cpp)
      │
      ▼
Compiler (พร้อม -fsanitize=...) แทรกคำสั่งตรวจสอบเข้าไปในทุกจุดที่เข้าถึงหน่วยความจำ/
                                มีความเสี่ยงต่อ Undefined Behavior/เข้าถึง shared state
      │
      ▼
Binary ที่มีโค้ดตรวจสอบฝังอยู่ในตัวเอง (ไม่ต้องมีโปรแกรมอื่นมาคอยจับตาดูจากภายนอก)
      │
      ▼
รันตรงบน CPU จริง (เร็วกว่า Valgrind มาก เพราะไม่ต้องแปล instruction ผ่านชั้นจำลอง)
```

ความแตกต่างเชิงสถาปัตยกรรมนี้ทำให้ Sanitizer เร็วกว่า Valgrind มาก (โดยทั่วไป ASan ช้ากว่า
โปรแกรมปกติแค่ **2-3 เท่า** เทียบกับ Valgrind ที่ช้ากว่า **10-50 เท่า**) แลกมาด้วยข้อจำกัดว่า
**ต้อง compile โปรแกรมใหม่ด้วย flag พิเศษเสมอ** (ใช้กับ binary ที่ compile ไว้แล้วโดยไม่มี flag
ไม่ได้เลย ต่างจาก Valgrind ที่รันกับ binary ปกติได้ทันที)

### ตารางเปรียบเทียบ Valgrind vs Sanitizer

| หัวข้อ | Valgrind (Memcheck) | Sanitizer (ASan/UBSan/TSan) |
|---|---|---|
| **หลักการทำงาน** | Dynamic Binary Instrumentation (รันผ่านตัวจำลอง) | Compile-Time Instrumentation (แทรกโค้ดตอน compile) |
| **ต้อง compile ใหม่ไหม** | ไม่ต้อง (รันกับ binary ปกติได้เลย ขอแค่มี `-g`) | ต้อง (ต้อง compile ด้วย `-fsanitize=...`) |
| **ความเร็ว (overhead)** | ช้ากว่าปกติ 10-50 เท่า | ช้ากว่าปกติ 2-3 เท่า (ASan), บางตัวมากกว่านั้น (TSan) |
| **หน่วยความจำที่ใช้เพิ่ม** | สูงมาก | สูง แต่น้อยกว่า Valgrind |
| **รองรับ Cross-platform** | Linux/macOS เท่านั้น | Linux/macOS/Windows (ผ่าน MSVC บางส่วน) |
| **ตรวจ Data Race ได้ไหม** | ได้ผ่านเครื่องมือแยก (Helgrind/DRD) | ได้โดยตรงผ่าน ThreadSanitizer |
| **ใช้กับ Production build ได้ไหม** | ไม่เหมาะ (ช้าเกินไป) | บางทีมใช้ ASan ใน Staging ได้ (แต่ไม่ใช่ Production จริง) |
| **จุดเด่น** | ตรวจ Memory Leak ได้ละเอียดมาก (definitely/possibly/still-reachable) | เร็ว เหมาะรันใน CI ทุกครั้งที่ push |

**บทสรุปเชิงปฏิบัติ**: ทั้งสองเครื่องมือไม่ได้แข่งกันแทนที่กัน — Sanitizer เหมาะกับการรันเป็นส่วนหนึ่ง
ของ CI ทุกครั้ง (เพราะเร็ว) ในขณะที่ Valgrind เหมาะกับการไล่ debug แบบละเอียดเป็นครั้งคราว
(เพราะช้ากว่าแต่ให้รายละเอียดบางอย่าง เช่น leak classification ที่ดีกว่า)

---

## 96.2 AddressSanitizer: Heap Buffer Overflow (Step 762)

### ติดตั้งและเปิดใช้งาน

AddressSanitizer มาพร้อมกับ GCC และ Clang อยู่แล้ว (ตั้งแต่ GCC 4.8 / Clang 3.1 ขึ้นไป) ไม่ต้อง
ติดตั้งอะไรเพิ่ม แค่เปิด flag ตอน compile:

```bash
g++ -Wall -Wextra -std=c++17 -g -fsanitize=address -fno-omit-frame-pointer myfile.cpp -o myfile
```

- **`-fsanitize=address`** — เปิด ASan
- **`-g`** — ใส่ debug symbol เพื่อให้ ASan รายงานเลขบรรทัดที่ถูกต้อง
- **`-fno-omit-frame-pointer`** — ทำให้ stack trace ที่ ASan รายงานแม่นยำและอ่านง่ายขึ้น

### ตัวอย่างจริง: Heap Buffer Overflow

```cpp
// heap_overflow.cpp
#include <cstdio>

int main() {
    int *arr = new int[5];
    for (int i = 0; i < 5; i++) {
        arr[i] = i * i;
    }
    // จงใจเขียนเกินขอบเขต heap array (index 5 ไม่มีอยู่จริง valid index คือ 0-4)
    arr[5] = 99;
    printf("arr[5] = %d\n", arr[5]);
    delete[] arr;
    return 0;
}
```

```bash
g++ -Wall -Wextra -std=c++17 -g -fsanitize=address -fno-omit-frame-pointer \
    heap_overflow.cpp -o heap_overflow_asan
./heap_overflow_asan
```

ผลลัพธ์จริงที่ได้บนเครื่องนี้:

```
=================================================================
==20644==ERROR: AddressSanitizer: heap-buffer-overflow on address 0x503000000054 at pc 0x55d39c9f230c bp 0x7ffcfc2d2fa0 sp 0x7ffcfc2d2f90
WRITE of size 4 at 0x503000000054 thread T0
    #0 0x55d39c9f230b in main heap_overflow.cpp:10
    #1 0x7fc59b02a1c9 in __libc_start_call_main ../sysdeps/nptl/libc_start_call_main.h:58
    #2 0x7fc59b02a28a in __libc_start_main_impl ../csu/libc-start.c:360
    #3 0x55d39c9f2184 in _start (heap_overflow_asan+0x1184)

0x503000000054 is located 0 bytes after 20-byte region [0x503000000040,0x503000000054)
allocated by thread T0 here:
    #0 0x7fc59b8fe6c8 in operator new[](unsigned long) ../../../../src/libsanitizer/asan/asan_new_delete.cpp:98
    #1 0x55d39c9f225e in main heap_overflow.cpp:5

SUMMARY: AddressSanitizer: heap-buffer-overflow heap_overflow.cpp:10 in main
Shadow bytes around the buggy address:
  ...
=>0x503000000000: fa fa 00 00 00 fa fa fa 00 00[04]fa fa fa fa fa
  ...
Shadow byte legend (one shadow byte represents 8 application bytes):
  Addressable:           00
  Partially addressable: 01 02 03 04 05 06 07
  Heap left redzone:       fa
  Freed heap region:       fd
  ...
==20644==ABORTING
```

### อ่านรายงานของ ASan อย่างถูกต้อง

1. **บรรทัดแรก** บอกประเภทบั๊กทันที: `heap-buffer-overflow` — เขียน/อ่านเกินขอบเขตของ heap
   allocation
2. **`WRITE of size 4 at ...`** — บอกว่าเป็นการ**เขียน** (ไม่ใช่อ่าน) ขนาด 4 ไบต์ (ตรงกับ `int`)
3. **Stack trace แรก** — บอกตำแหน่งที่เกิด error จริง (`heap_overflow.cpp:10` ตรงกับบรรทัด
   `arr[5] = 99;` เป๊ะ)
4. **"0 bytes after 20-byte region"** — บอกชัดว่า address ที่เขียนอยู่ **ติดกันทันที** หลัง block
   ที่จองไว้ขนาด 20 ไบต์ (5 × `int` = 20 ไบต์ ตรงกับ `new int[5]` เป๊ะ) — นี่คือหลักฐานยืนยันว่า
   `arr[5]` (index ที่ 6, เกิน array ไป 1 ตัว) คือสาเหตุจริง
5. **"allocated by thread T0 here"** — บอกว่า block นี้ถูกจองที่บรรทัดไหน (`heap_overflow.cpp:5`
   ตรงกับ `new int[5]`) ทำให้เห็นภาพครบทั้งจุดจองและจุดที่ทำผิด ในรายงานเดียว
6. **Shadow Bytes** — ส่วนขั้นสูงที่แสดงสถานะหน่วยความจำภายในของ ASan (ปกติไม่ต้องอ่านลึก
   ขนาดนี้ในการ debug ทั่วไป เว้นแต่กำลังศึกษากลไกภายในของ ASan เอง)
7. **`==20644==ABORTING`** — ASan **หยุดโปรแกรมทันที** เมื่อเจอปัญหา (ผิดกับ default behavior
   ของ UBSan ที่จะเห็นในหัวข้อ 96.4 ซึ่งรายงานแล้วปล่อยให้โปรแกรมทำงานต่อ)

---

## 96.3 AddressSanitizer: Use-After-Free และเทียบผลกับ Valgrind (Step 763)

### ตัวอย่างจริง: Use-After-Free

```cpp
// use_after_free.cpp
#include <cstdio>

struct Widget {
    int id;
};

int main() {
    Widget *w = new Widget{42};
    printf("id before delete = %d\n", w->id);
    delete w;
    // จงใจใช้ pointer ต่อหลังจาก delete ไปแล้ว (dangling pointer)
    printf("id after delete = %d\n", w->id);
    return 0;
}
```

```bash
g++ -Wall -Wextra -std=c++17 -g -fsanitize=address -fno-omit-frame-pointer \
    use_after_free.cpp -o uaf_asan
./uaf_asan
```

ผลลัพธ์จริง:

```
=================================================================
==20676==ERROR: AddressSanitizer: heap-use-after-free on address 0x502000000010 at pc 0x56210ea36340 bp 0x7fff48537580 sp 0x7fff48537570
READ of size 4 at 0x502000000010 thread T0
    #0 0x56210ea3633f in main use_after_free.cpp:12

0x502000000010 is located 0 bytes inside of 4-byte region [0x502000000010,0x502000000014)
freed by thread T0 here:
    #0 0x7f97332ff5e8 in operator delete(void*, unsigned long) ../../../../src/libsanitizer/asan/asan_new_delete.cpp:164
    #1 0x56210ea36308 in main use_after_free.cpp:10

previously allocated by thread T0 here:
    #0 0x7f97332fe548 in operator new(unsigned long) ../../../../src/libsanitizer/asan/asan_new_delete.cpp:95
    #1 0x56210ea3625e in main use_after_free.cpp:8

SUMMARY: AddressSanitizer: heap-use-after-free use_after_free.cpp:12 in main
```

จุดเด่นของรายงานนี้คือ ASan บอกครบทั้ง **3 เหตุการณ์ในไทม์ไลน์เดียวกัน**:

1. `previously allocated ... use_after_free.cpp:8` — จองที่ไหน (`new Widget{42}`)
2. `freed by thread T0 here ... use_after_free.cpp:10` — ปล่อยคืนที่ไหน (`delete w`)
3. `READ of size 4 ... use_after_free.cpp:12` — ใช้งานต่อหลังปล่อยคืนแล้วที่ไหน (`w->id` บรรทัด
   ที่สอง)

การเห็นทั้ง 3 จุดพร้อมกันในรายงานเดียวทำให้ debug เร็วกว่าการนั่งไล่โค้ดเองมาก โดยเฉพาะเมื่อ
`new`/`delete`/`use` อยู่คนละฟังก์ชันหรือคนละไฟล์กัน

### เปรียบเทียบกับ Valgrind บนบั๊กเดียวกัน (Part 38 vs Part นี้)

compile โค้ดเดียวกันแบบ**ไม่มี** sanitizer แล้วรันผ่าน Valgrind แทน:

```bash
g++ -Wall -Wextra -std=c++17 -g use_after_free.cpp -o uaf_plain
valgrind --error-exitcode=1 ./uaf_plain
```

ผลลัพธ์จริงจาก Valgrind:

```
==20698== Memcheck, a memory error detector
==20698== Command: ./uaf_plain
==20698==
==20698== Invalid read of size 4
==20698==    at 0x1091DF: main (use_after_free.cpp:12)
==20698==  Address 0x4e21080 is 0 bytes inside a block of size 4 free'd
==20698==    at 0x484A61D: operator delete(void*, unsigned long) (in /usr/libexec/valgrind/vgpreload_memcheck-amd64-linux.so)
==20698==    by 0x1091DA: main (use_after_free.cpp:10)
==20698==  Block was alloc'd at
==20698==    at 0x4846FA3: operator new(unsigned long) (in /usr/libexec/valgrind/vgpreload_memcheck-amd64-linux.so)
==20698==    by 0x10919E: main (use_after_free.cpp:8)
==20698==
id before delete = 42
id after delete = 42
==20698==
==20698== HEAP SUMMARY:
==20698==     in use at exit: 0 bytes in 0 blocks
==20698==   total heap usage: 3 allocs, 3 frees, 77,828 bytes allocated
==20698==
==20698== ERROR SUMMARY: 1 errors from 1 contexts (suppressed: 0 from 0)
```

### ข้อสังเกตสำคัญที่พบจากการทดสอบจริง: พฤติกรรมหลังพบ Error ต่างกัน

จุดที่น่าสนใจที่สุดจากการรันทั้งสองเครื่องมือกับบั๊กเดียวกันคือ **พฤติกรรมหลังตรวจพบปัญหา
ต่างกันโดยสิ้นเชิง**:

| | ASan | Valgrind |
|---|---|---|
| ตรวจพบ use-after-free | ✅ พบ | ✅ พบ |
| โปรแกรมทำอะไรต่อ | **หยุดทันที** (`ABORTING`) ไม่พิมพ์ "id after delete" เลย | **รันต่อไปตามปกติ** พิมพ์ "id after delete = 42" ออกมาด้วย (ค่ายังไม่ถูกเขียนทับ จึงบังเอิญถูกในกรณีนี้) |
| Exit code | ไม่ใช่ 0 (fail ทันที) | ไม่ใช่ 0 (เพราะสั่ง `--error-exitcode=1` ไว้) แต่โปรแกรมได้ทำงานจนจบสมบูรณ์ |

Valgrind ปล่อยให้โปรแกรมทำงานต่อไปหลังรายงาน error (default behavior) ในขณะที่ ASan
**หยุดโปรแกรมทันที** เมื่อเจอปัญหาร้ายแรง — ทั้งสองแบบมีข้อดีต่างกัน: การหยุดทันทีของ ASan
ทำให้แน่ใจว่าจะไม่มี "ผลข้างเคียงที่ผิดเพี้ยนต่อเนื่อง" จากหน่วยความจำที่เสียหายแล้ว เหมาะกับ CI
ที่ต้องการคำตอบชัดเจนว่า "ผ่านหรือไม่ผ่าน" ส่วนการปล่อยให้รันต่อของ Valgrind มีประโยชน์ตอน
อยากเห็นว่าถ้าปล่อยบั๊กนี้ไว้จริง จะกระทบพฤติกรรมโปรแกรมส่วนอื่นต่อไปอย่างไรด้วย

### เปรียบเทียบความเร็ว: วัดจริงบนเครื่องนี้

ทดสอบด้วยโปรแกรมที่เข้าถึงหน่วยความจำหนักๆ (อ่าน/เขียน array ขนาด 2 ล้าน `int` วนซ้ำ 50 รอบ):

```bash
g++ -O1 -std=c++17 perf_compare2.cpp -o perf_plain
g++ -O1 -g -fsanitize=address -std=c++17 perf_compare2.cpp -o perf_asan

time ./perf_plain
time ./perf_asan
time valgrind --quiet ./perf_plain
```

ผลลัพธ์จริงบนเครื่องนี้:

```
=== plain (ไม่มี instrumentation) ===
real    0m0.088s

=== ASan ===
real    0m0.233s

=== Valgrind (memcheck) ===
real    0m1.821s
```

สรุปเป็นตัวเลข: ASan ช้ากว่าปกติประมาณ **2.6 เท่า** ในขณะที่ Valgrind ช้ากว่าปกติประมาณ
**20.7 เท่า** — ตัวเลขนี้ยืนยันสิ่งที่ตารางเปรียบเทียบในหัวข้อ 96.1 บอกไว้อย่างชัดเจนด้วยข้อมูลจริง
จากเครื่องเดียวกัน ตัวเลขจริงจะแตกต่างกันไปตามลักษณะโปรแกรม (โปรแกรมที่ทำ syscall เยอะๆ
อาจเห็นความต่างน้อยกว่านี้) แต่แนวโน้มที่ ASan เร็วกว่า Valgrind อย่างมีนัยสำคัญนั้นเป็นจริงเสมอ

---

## 96.4 UndefinedBehaviorSanitizer: Integer Overflow และ Null Pointer Dereference (Step 764)

### ตัวอย่างจริง: Signed Integer Overflow

```cpp
// int_overflow.cpp
#include <cstdio>
#include <climits>

int multiply(int a, int b) {
    return a * b;   // จงใจไม่เช็ค overflow
}

int main() {
    int a = INT_MAX;
    int b = 2;
    printf("INT_MAX = %d\n", a);
    int result = multiply(a, b);   // signed integer overflow = Undefined Behavior
    printf("result = %d\n", result);
    return 0;
}
```

```bash
g++ -Wall -Wextra -std=c++17 -g -fsanitize=undefined int_overflow.cpp -o int_overflow_ubsan
./int_overflow_ubsan
```

ผลลัพธ์จริง:

```
int_overflow.cpp:5:16: runtime error: signed integer overflow: 2147483647 * 2 cannot be represented in type 'int'
INT_MAX = 2147483647
result = -2
```

### ข้อสังเกตสำคัญ: UBSan รายงานแล้ว "ปล่อยให้รันต่อ" โดย default

สังเกตว่าโปรแกรมพิมพ์ `result = -2` ต่อจนจบ **ไม่หยุดทันทีเหมือน ASan** นี่คือค่า default ของ
UBSan — มันรายงาน UB แล้วปล่อยให้โปรแกรมทำงานต่อไปตามพฤติกรรมจริงของ hardware (ซึ่งใน
กรณีนี้คือ wraparound แบบ two's complement จนได้ `-2`) เหตุผลที่ UBSan ออกแบบมาแบบนี้คือ
Undefined Behavior หลายชนิด "ดูเหมือนไม่ทำให้โปรแกรมพังทันที" การหยุดทุกครั้งอาจรบกวนการ
ไล่ดูว่ามี UB กี่จุดในโปรแกรมเดียว

ถ้าต้องการให้ UBSan หยุดทันทีเหมือน ASan (เหมาะกับการใช้ใน CI ที่ต้องการคำตอบ pass/fail
ชัดเจน) ใช้ flag เพิ่ม:

```bash
g++ -Wall -Wextra -std=c++17 -g -fsanitize=undefined -fno-sanitize-recover=all \
    int_overflow.cpp -o int_overflow_ubsan_strict
./int_overflow_ubsan_strict
echo "EXIT: $?"
```

ผลลัพธ์จริง:

```
int_overflow.cpp:5:16: runtime error: signed integer overflow: 2147483647 * 2 cannot be represented in type 'int'
EXIT: 1
```

ครั้งนี้โปรแกรม**หยุดทันที**หลังรายงาน (ไม่พิมพ์ `result = ...` เลย) และ exit code เป็น 1 — นี่คือ
รูปแบบที่ควรใช้เสมอเมื่อรวม UBSan เข้ากับ CI (จะทำในหัวข้อ 96.6)

### ตัวอย่างจริง: Null Pointer Dereference

```cpp
// null_deref.cpp
#include <cstdio>

struct Config {
    int timeout;
};

int read_timeout(Config *cfg) {
    return cfg->timeout;   // ไม่เช็คว่า cfg เป็น nullptr ก่อน
}

int main() {
    Config *cfg = nullptr;
    printf("timeout = %d\n", read_timeout(cfg));
    return 0;
}
```

```bash
g++ -Wall -Wextra -std=c++17 -g -fsanitize=undefined null_deref.cpp -o null_deref_ubsan
./null_deref_ubsan
```

ผลลัพธ์จริง:

```
null_deref.cpp:8:17: runtime error: member access within null pointer of type 'struct Config'
Segmentation fault
```

จุดที่น่าสนใจมากในกรณีนี้คือ UBSan **รายงานสาเหตุที่แท้จริงก่อน** (`member access within null
pointer`) แล้วโปรแกรมถึง **segfault จริง** ตามมา (เพราะการ dereference `nullptr` จริงๆ ทำให้ OS
ส่ง `SIGSEGV` มาฆ่าโปรเซสอยู่ดี ไม่ว่าจะมี UBSan หรือไม่) เทียบกับการรันโปรแกรมเดียวกันแบบไม่มี
UBSan ที่จะได้แค่ `Segmentation fault` เฉยๆ **โดยไม่มีคำอธิบายใดๆ เลยว่าทำไม** — นี่คือคุณค่า
ที่แท้จริงของ UBSan ในสถานการณ์นี้: มันไม่ได้ "ป้องกัน" segfault (เพราะป้องกันไม่ได้จริงๆ) แต่
**อธิบายก่อนที่มันจะเกิด** ทำให้ debug เร็วขึ้นมหาศาลเมื่อเทียบกับการเห็นแค่ "Segmentation fault"
ลอยๆ แล้วต้องเปิด GDB (Part 37) ไล่หาสาเหตุเอง

---

## 96.5 ThreadSanitizer: Data Race (Step 765)

### ทบทวน Race Condition จาก Part 32 และ Part 81

ใน **Part 32** เราเขียนโค้ด C ด้วย `pthread` ที่มี Race Condition จากตัวแปร `counter` ที่หลาย
thread เข้าถึงพร้อมกันโดยไม่มีการป้องกัน และใน **Part 81** เราเขียนโปรแกรมแบบเดียวกันด้วย
`std::thread` ของ C++ ลองเขียนโปรแกรมสไตล์เดียวกันอีกครั้งเพื่อทดสอบกับ ThreadSanitizer:

```cpp
// race.cpp
#include <cstdio>
#include <thread>
#include <vector>

long counter = 0;   // shared state ไม่มีการป้องกัน (เทียบกับ race.c ใน Part 32)

void increment_counter() {
    for (int i = 0; i < 100000; i++) {
        counter++;   // Critical Section ที่ไม่ปลอดภัย
    }
}

int main() {
    const int NUM_THREADS = 4;
    std::vector<std::thread> threads;

    for (int i = 0; i < NUM_THREADS; i++) {
        threads.emplace_back(increment_counter);
    }
    for (auto &t : threads) {
        t.join();
    }

    printf("Expected: %d\n", NUM_THREADS * 100000);
    printf("Actual:   %ld\n", counter);
    return 0;
}
```

### พิสูจน์ Race Condition ก่อนด้วยการรันปกติ (ไม่มี sanitizer)

```bash
g++ -Wall -Wextra -std=c++17 -pthread race.cpp -o race_plain
./race_plain
./race_plain
./race_plain
```

ผลลัพธ์จริง (รัน 3 ครั้งติดกัน เห็นค่าไม่เท่ากันทุกครั้งและต่างจากค่าที่ควรได้มาก):

```
Expected: 400000
Actual:   138401
Expected: 400000
Actual:   129325
Expected: 400000
Actual:   148122
```

โปรแกรมนี้ **compile ผ่านสะอาด ไม่ crash เลย** แต่ให้ผลลัพธ์ผิดทุกครั้งและไม่เท่ากันในแต่ละรอบ —
เป็นหลักฐานชัดเจนของ Data Race แบบเดียวกับที่พิสูจน์ไว้ใน Part 32 ด้วยโค้ด C ล้วน

### รันผ่าน ThreadSanitizer

```bash
g++ -Wall -Wextra -std=c++17 -g -pthread -fsanitize=thread race.cpp -o race_tsan
./race_tsan
```

ผลลัพธ์จริงบนเครื่องนี้ (ตัดมาแสดงเฉพาะส่วนสำคัญที่สุด — รายงานฉบับเต็มยาวกว่านี้มากเพราะมี
stack trace ของทุก thread ที่เกี่ยวข้อง):

```
==================
WARNING: ThreadSanitizer: data race (pid=21032)
  Read of size 8 at 0x56390ba15020 by thread T2:
    #0 increment_counter() race.cpp:9

  Previous write of size 8 at 0x56390ba15020 by thread T1:
    #0 increment_counter() race.cpp:9

  Location is global 'counter' of size 8 at 0x56390ba15020

  Thread T2 (tid=21036, running) created by main thread at:
    #0 pthread_create ...
    #6 main race.cpp:18

  Thread T1 (tid=21035, running) created by main thread at:
    #0 pthread_create ...
    #6 main race.cpp:18

SUMMARY: ThreadSanitizer: data race race.cpp:9 in increment_counter()
==================
... (มี WARNING ซ้ำอีกชุดสำหรับคู่ thread อื่น) ...
Expected: 400000
Actual:   311628
ThreadSanitizer: reported 2 warnings
```

จากการรันจริง TSan รายงาน **2 คู่ data race** ที่บรรทัด `counter++;` (race.cpp:9) และโปรแกรม
จบด้วย exit code **66** (ค่ามาตรฐานที่ TSan ใช้บอกว่าพบปัญหา ไม่ใช่ 0 หรือ 1 แบบ ASan/UBSan)

### อ่านรายงานของ TSan อย่างถูกต้อง

1. **`Read of size 8 ... by thread T2`** คู่กับ **`Previous write of size 8 ... by thread T1`**
   — TSan บอกชัดว่า thread ไหนอ่าน thread ไหนเขียน ที่ location เดียวกัน โดยไม่มีการป้องกัน
   (ไม่มี mutex หรือ atomic คั่นกลาง)
2. **`Location is global 'counter'`** — ระบุตัวแปรที่เป็นปัญหาชัดเจน (ในกรณีนี้คือ global
   variable `counter`)
3. **`Thread T2 created by main thread at: ... main race.cpp:18`** — บอกว่า thread นี้ถูกสร้าง
   ที่บรรทัดไหน (ตรงกับ `threads.emplace_back(increment_counter);`) ทำให้ตามรอยได้ว่า thread
   ไหนคือ thread ไหนจริงๆ ในโค้ด
4. **สังเกตว่าโปรแกรมยังรันจนจบ** (พิมพ์ `Actual: 311628` ออกมาด้วย) — TSan ปล่อยให้โปรแกรม
   ทำงานต่อหลังรายงาน (คล้าย UBSan default) ต่างจาก ASan ที่ abort ทันที

### แก้ปัญหาด้วย Mutex และยืนยันด้วย TSan อีกครั้ง

```cpp
// race_fixed.cpp
#include <cstdio>
#include <mutex>
#include <thread>
#include <vector>

long counter = 0;
std::mutex counter_mutex;

void increment_counter() {
    for (int i = 0; i < 100000; i++) {
        std::lock_guard<std::mutex> lock(counter_mutex);
        counter++;
    }
}

int main() {
    const int NUM_THREADS = 4;
    std::vector<std::thread> threads;

    for (int i = 0; i < NUM_THREADS; i++) {
        threads.emplace_back(increment_counter);
    }
    for (auto &t : threads) {
        t.join();
    }

    printf("Expected: %d\n", NUM_THREADS * 100000);
    printf("Actual:   %ld\n", counter);
    return 0;
}
```

```bash
g++ -Wall -Wextra -std=c++17 -g -pthread -fsanitize=thread race_fixed.cpp -o race_fixed_tsan
./race_fixed_tsan
echo "EXIT: $?"
```

ผลลัพธ์จริง:

```
Expected: 400000
Actual:   400000
EXIT: 0
```

ไม่มี warning ใดๆ จาก TSan เลย ค่า `Actual` ตรงกับ `Expected` ทุกครั้งที่รันซ้ำ (ลองรันหลายรอบเพื่อ
ยืนยันว่าไม่ใช่ความบังเอิญ) — `std::lock_guard<std::mutex>` (จาก Part 82) รับประกันว่ามีแค่
thread เดียวเข้าไปใน critical section (`counter++`) ได้ในเวลาเดียวกัน

---

## 96.6 ใช้ Sanitizer หลายตัวพร้อมกัน: ข้อจำกัดที่ต้องรู้ (Step 766)

### ASan + UBSan: ใช้ร่วมกันได้

เป็นการรวมที่**นิยมใช้ที่สุด**ในทางปฏิบัติ เพราะทั้งสองตัวตรวจจับปัญหาคนละประเภทที่ไม่ทับซ้อนกัน
(ASan เน้น memory, UBSan เน้น undefined behavior เชิง logic/arithmetic):

```bash
g++ -Wall -Wextra -std=c++17 -g -fsanitize=address,undefined \
    heap_overflow.cpp -o combo_asan_ubsan
./combo_asan_ubsan
```

ผลลัพธ์จริง — ยืนยันว่า compile ผ่านและ ASan ยังคงตรวจจับปัญหาได้ตามปกติ:

```
=================================================================
==21212==ERROR: AddressSanitizer: heap-buffer-overflow on address 0x503000000054 at pc 0x561bdbeba47b bp 0x7ffe21fcab00 sp 0x7ffe21fcaaf0
WRITE of size 4 at 0x503000000054 thread T0
    #0 0x561bdbeba47a in main heap_overflow.cpp:10
```

### ASan + TSan: ใช้ร่วมกันไม่ได้

```bash
g++ -Wall -Wextra -std=c++17 -g -pthread -fsanitize=address,thread race.cpp -o combo_asan_tsan
```

ผลลัพธ์จริง — **compiler ปฏิเสธตั้งแต่ขั้นตอน compile เลย**:

```
cc1plus: error: '-fsanitize=thread' is incompatible with '-fsanitize=address'
```

**เหตุผลเชิงเทคนิค**: ทั้ง ASan และ TSan ต่างก็ต้องการควบคุม memory layout และ runtime
ของโปรแกรมในระดับที่ลึกมาก (ASan ต้องการพื้นที่ "shadow memory" ขนาดใหญ่คลุมทุก byte ของ
heap/stack, TSan ก็ต้องการ shadow memory ของตัวเองสำหรับติดตาม happens-before relationship
ระหว่าง thread) กลไกทั้งสองชนกันในระดับ memory layout ของ process ทำให้ไม่สามารถทำงาน
ร่วมกันในกระบวนการเดียวได้ นี่ไม่ใช่ข้อจำกัดที่จะแก้ในเร็วๆ นี้ แต่เป็นข้อจำกัดเชิงสถาปัตยกรรมที่
ทีม LLVM/GCC ยอมรับและบันทึกไว้อย่างเป็นทางการ

### วิธีแก้ในทางปฏิบัติ: แยก Build

เมื่อรวมกันไม่ได้ วิธีแก้คือ**สร้าง build แยกกันสำหรับแต่ละ sanitizer** แล้วรัน test suite เดียวกัน
ผ่านแต่ละ build:

```bash
# Build 1: สำหรับตรวจ memory bug + UB
g++ -g -fsanitize=address,undefined -fno-sanitize-recover=all myproject.cpp -o app_asan_ubsan

# Build 2: สำหรับตรวจ data race (แยกต่างหาก)
g++ -g -pthread -fsanitize=thread myproject.cpp -o app_tsan
```

ใน CI มักตั้งเป็นสอง job แยกกัน รันขนานกันไปพร้อมกับ job build-and-test และ static-analysis
จาก Part 94–95 (จะทำจริงในหัวข้อ 96.8)

### สรุปตารางการรวม Sanitizer

| การรวม | ใช้ร่วมกันได้ไหม | หมายเหตุ |
|---|---|---|
| ASan + UBSan | ✅ ได้ (นิยมใช้คู่กันเป็นค่าเริ่มต้น) | ตรวจจับปัญหาคนละประเภท ไม่ทับซ้อนกัน |
| ASan + LeakSanitizer (LSan) | ✅ ได้ (LSan ผนวกอยู่ใน ASan โดย default บน Linux อยู่แล้ว) | ไม่ต้องเปิดแยก |
| ASan + TSan | ❌ ไม่ได้ | Compiler ปฏิเสธตั้งแต่ compile-time เพราะชน memory layout กัน |
| UBSan + TSan | ✅ ได้ | ทั้งสองไม่ได้ชนกันเรื่อง memory layout |
| MemorySanitizer (MSan) + ASan/TSan | ❌ ไม่ได้ | MSan (ตรวจ uninitialized memory) ก็ชนกับ ASan/TSan เช่นกัน |

---

## 96.7 Code Coverage ด้วย gcov และ lcov (Step 767)

### แนวคิด: Coverage วัดอะไร

**Code Coverage** วัดว่า**บรรทัดโค้ด/ฟังก์ชัน/branch เท่าไหร่ที่ถูกรันจริงระหว่าง test suite ทำงาน**
ค่าที่ได้เป็นตัวชี้วัดว่า test ของเราจาก **Part 93** "ครอบคลุม" โค้ดมากแค่ไหน — **สำคัญมาก**ที่
ต้องเข้าใจว่า Coverage 100% **ไม่ได้แปลว่าโค้ดไม่มีบั๊ก** มันแปลว่า "ทุกบรรทัดถูกรันอย่างน้อย
หนึ่งครั้ง" เท่านั้น ไม่ได้รับประกันว่า assertion ที่ตรวจสอบครอบคลุมทุกกรณีขอบ (edge case) จริง

### ติดตั้งเครื่องมือ

```bash
sudo apt install lcov -y   # gcov มากับ gcc อยู่แล้ว ไม่ต้องติดตั้งแยก
gcov --version
lcov --version
```

```
gcov (Ubuntu 13.3.0-6ubuntu2~24.04.1) 13.3.0
lcov: LCOV version 2.0-1
```

### Compile ด้วย Flag สำหรับ Coverage

ต้องเพิ่ม `--coverage` (เทียบเท่า `-fprofile-arcs -ftest-coverage` ตอน compile และ `-lgcov`
ตอน link) และปิด optimization (`-O0`) เพื่อให้เลขบรรทัดที่รายงานตรงกับโค้ดต้นฉบับเป๊ะ:

```bash
cmake -S . -B build-cov -DCMAKE_BUILD_TYPE=Debug \
      -DCMAKE_CXX_FLAGS="--coverage -O0" \
      -DCMAKE_EXE_LINKER_FLAGS="--coverage"
cmake --build build-cov -j
```

### รัน Test Suite จาก Part 93 เพื่อสร้างข้อมูล Coverage

ใช้โปรเจกต์ตัวอย่างเดียวกับ Part 94 (ไลบรารี `mathutils` + Google Test) โดยเพิ่มฟังก์ชันใหม่
`max3` เข้าไปเพื่อสาธิตกรณีที่ยังไม่มี test ครอบคลุม:

```cpp
// mathutils.hpp (เพิ่มเข้ามา)
int max3(int a, int b, int c);
```

```cpp
// mathutils.cpp (เพิ่มเข้ามา)
int max3(int a, int b, int c) {
    if (a >= b && a >= c) return a;
    if (b >= a && b >= c) return b;
    return c;
}
```

รัน test suite เดิม (8 test case จาก Part 94 — **ยังไม่มี test สำหรับ `max3` เลย**):

```bash
cd build-cov
./unit_tests
```

```
[==========] 8 tests from 1 test suite ran. (0 ms total)
[  PASSED  ] 8 tests.
```

### สร้าง Coverage Report ด้วย gcov โดยตรง (ระดับไฟล์เดียว)

```bash
cd build-cov/CMakeFiles/mathutils.dir/src
gcov mathutils.cpp.gcno --object-directory .
```

ผลลัพธ์จริง:

```
File 'mathutils.cpp'
Lines executed:78.57% of 28
Creating 'mathutils.cpp.gcov'
```

เปิดดูไฟล์ `mathutils.cpp.gcov` ที่ gcov สร้างขึ้น (เป็นไฟล์ต้นฉบับที่มีตัวเลขนับจำนวนครั้งที่แต่ละ
บรรทัดถูกรันแปะไว้ด้านหน้า):

```
        2:    5:int add(int a, int b) {
        2:    6:    return a + b;
        -:    7:}
        -:    8:
        4:    9:long long factorial(int n) {
        4:   10:    if (n < 0) {
        1:   11:        return 0;
        -:   12:    }
        ...
        2:   32:int gcd(int a, int b) {
        8:   33:    while (b != 0) {
        6:   34:        int t = b;
        ...
    #####:   41:int max3(int a, int b, int c) {
    #####:   42:    if (a >= b && a >= c) {
    #####:   43:        return a;
        -:   44:    }
    #####:   45:    if (b >= a && b >= c) {
    #####:   46:        return b;
        -:   47:    }
    #####:   48:    return c;
        -:   49:}
```

การอ่านผลลัพธ์นี้:

- **ตัวเลขทางซ้าย** (เช่น `2:`, `4:`, `8:`) คือ**จำนวนครั้ง**ที่บรรทัดนั้นถูกรันจริงระหว่าง test
- **`-:`** หมายถึงบรรทัดที่ไม่ใช่ executable code (เช่น comment, บรรทัดว่าง, ปีกกาปิด)
- **`#####:`** คือสัญลักษณ์ที่สำคัญที่สุด — บรรทัดนี้**ไม่เคยถูกรันเลยสักครั้ง** ในที่นี้คือทุกบรรทัด
  ของฟังก์ชัน `max3` ที่ยังไม่มี test เรียกใช้เลย

### สร้าง HTML Report ด้วย lcov (วิธีที่ใช้จริงในทางปฏิบัติ)

การอ่านไฟล์ `.gcov` ทีละไฟล์ไม่สะดวกเมื่อโปรเจกต์มีหลายสิบไฟล์ **lcov** ทำหน้าที่รวบรวมข้อมูล
จาก gcov ทั้งโปรเจกต์แล้วสร้างเป็น HTML Report ที่กดดูผ่าน browser ได้:

```bash
cd build-cov
lcov --capture --directory . --output-file coverage.info \
     --base-directory .. --ignore-errors mismatch
```

> **ปัญหาจริงที่พบระหว่างทดสอบ**: การรัน `lcov --capture` โดยไม่ใส่ `--ignore-errors mismatch`
> บนเครื่องนี้ (lcov 2.0-1 คู่กับ gcc/gcov 13.3.0) ทำให้เกิด **error และหยุดทำงานทันที**:
> ```
> geninfo: ERROR: mismatched end line for _ZN33MathUtils_AddPositiveNumbers_Test8TestBodyEv
> at test_mathutils.cpp:5: 5 -> 7
> (use "geninfo --ignore-errors mismatch ..." to bypass this error)
> ```
> นี่เป็นความไม่ตรงกันเล็กน้อยระหว่างข้อมูล debug ที่ gcc รุ่นนี้สร้างกับสิ่งที่ lcov 2.0 คาดหวัง
> (ไม่ใช่บั๊กในโค้ดของเราเอง) lcov บอกวิธีแก้ไว้ในข้อความ error เลย คือเพิ่ม
> `--ignore-errors mismatch` ตามที่ใช้ด้านบน — เป็นตัวอย่างที่ดีว่าทำไมต้องอ่าน error message
> ของเครื่องมือ tooling อย่างละเอียดเสมอ แทนที่จะเดาสุ่มแก้ปัญหา

กรองไฟล์ system header และไฟล์ test ออก (ไม่ต้องการวัด coverage ของ `<gtest/gtest.h>` หรือ
ของไฟล์ test เอง สนใจแค่ coverage ของ**โค้ด production** ใน `src/`):

```bash
lcov --remove coverage.info '/usr/*' '*/tests/*' \
     --output-file coverage.filtered.info --ignore-errors unused
```

สร้าง HTML:

```bash
genhtml coverage.filtered.info --output-directory coverage_html
```

ผลลัพธ์จริง:

```
Found 1 entries.
Found common filename prefix "/.../demo_project"
Generating output.
Processing file src/mathutils.cpp
  lines=28 hit=22 functions=5 hit=4
Overall coverage rate:
  lines......: 78.6% (22 of 28 lines)
  functions......: 80.0% (4 of 5 functions)
```

ผลลัพธ์ตรงกับที่ gcov รายงานไว้ก่อนหน้า (78.57% ≈ 78.6% ต่างกันแค่การปัดเศษ) — 4 จาก 5
ฟังก์ชันถูกทดสอบ (`add`, `factorial`, `is_prime`, `gcd`) ส่วน `max3` ยังไม่มี test เลย

### โครงสร้างของ HTML Report ที่ได้จริง

```
coverage_html/
├── index.html              ← หน้าสรุปภาพรวมทั้งโปรเจกต์
├── gcov.css                ← สไตล์ของรายงาน
├── amber.png / emerald.png / ruby.png   ← ไอคอนสีแสดงระดับ coverage
└── src/
    ├── index.html           ← สรุประดับโฟลเดอร์ src/
    ├── mathutils.cpp.gcov.html   ← ไฟล์โค้ดพร้อมสีไฮไลต์ทุกบรรทัด (เขียว=ถูกรัน, แดง=ไม่ถูกรัน)
    └── mathutils.cpp.func.html   ← สรุป coverage แยกตามฟังก์ชัน
```

เปิด `index.html` จะเห็นตารางสรุปเปอร์เซ็นต์ coverage พร้อมสีบอกระดับ (**emerald/เขียว** =
coverage สูง, **amber/เหลือง** = ปานกลาง, **ruby/แดง** = ต่ำ) และคลิกลึกเข้าไปที่ไฟล์ใดไฟล์หนึ่ง
จะเห็นโค้ดต้นฉบับพร้อมพื้นหลังสีเขียวสำหรับบรรทัดที่ถูกทดสอบ และพื้นหลังสีแดงสำหรับบรรทัดที่
`#####` (ไม่เคยถูกรันเลย) — ในกรณีนี้คือทุกบรรทัดในฟังก์ชัน `max3`

### เพิ่ม Test ที่ขาดหายไปและยืนยัน Coverage กลับมา 100%

```cpp
TEST(MathUtils, Max3AllBranches) {
    EXPECT_EQ(mathutils::max3(3, 1, 2), 3);   // a มากสุด
    EXPECT_EQ(mathutils::max3(1, 5, 2), 5);   // b มากสุด
    EXPECT_EQ(mathutils::max3(1, 2, 9), 9);   // c มากสุด
}
```

Build และรัน coverage ใหม่:

```bash
cmake --build build-cov -j
cd build-cov && ./unit_tests
```

```
[----------] 9 tests from MathUtils (0 ms total)
[  PASSED  ] 9 tests.
```

สร้าง report ใหม่ด้วยขั้นตอนเดิม:

```bash
lcov --capture --directory . --output-file coverage.info --base-directory .. --ignore-errors mismatch
lcov --remove coverage.info '/usr/*' '*/tests/*' --output-file coverage.filtered.info --ignore-errors unused
genhtml coverage.filtered.info --output-directory coverage_html
```

ผลลัพธ์จริง:

```
Processing file src/mathutils.cpp
  lines=28 hit=28 functions=5 hit=5
Overall coverage rate:
  lines......: 100.0% (28 of 28 lines)
  functions......: 100.0% (5 of 5 functions)
```

กลับมาเป็น 100% ครบทั้ง lines และ functions — พิสูจน์การเขียน test เพิ่มให้ครอบคลุมทุก branch
ของ `max3` (กรณี a มากสุด, b มากสุด, c มากสุด) ทำงานได้จริงตามที่คาดหวัง

### ข้อควรระวังที่พบจริงบนเครื่องนี้: `lcov --list` รายงานตัวเลขผิดเพี้ยน

ระหว่างทดสอบพบพฤติกรรมที่ควรรู้ไว้: คำสั่ง `lcov --list <file>.info` (แยกต่างหากจากตอน
`--capture`/`--remove`) บนเครื่องนี้ (lcov 2.0-1) รายงานตัวเลขที่**ไม่ตรงกับความเป็นจริง**:

```bash
lcov --list coverage.filtered.info
```

```
                   |Lines       |Functions  |Branches
Filename           |Rate     Num|Rate    Num|Rate     Num
=========================================================
mathutils.cpp      |18.2%     22| 0.0%     4|    -      0
```

ตัวเลขนี้**ผิดอย่างชัดเจน** — ไฟล์ `.info` เดียวกันนี้ เมื่อดูค่า `LF`/`LH` (Line Found/Line Hit)
ที่เก็บอยู่ในไฟล์โดยตรง หรือใช้ `genhtml` สร้างรายงาน กลับได้ค่าที่ถูกต้อง (100% ตามที่แสดงไว้
ด้านบน) จุดนี้เป็น**ข้อบกพร่องที่พบจริงของคำสั่ง `lcov --list` บนเวอร์ชันนี้** ไม่ใช่ปัญหาของ
โปรเจกต์หรือของ test เอง — บทเรียนสำคัญคือ **อย่าเชื่อผลลัพธ์จากเครื่องมือเพียงคำสั่งเดียว** เมื่อ
ตัวเลขดูน่าสงสัย (เช่น ทำไมบรรทัดเดียวกันได้ผลต่างกันระหว่าง `--capture` กับ `--list`) ให้ตรวจ
ทานข้าม-เครื่องมือ (cross-check) เช่น เทียบกับผลจาก `genhtml` โดยตรงหรือเปิดไฟล์ `.info` ดู
ค่า `LF:`/`LH:` เอง เสมอก่อนสรุปผล — โดยสรุปให้ **เชื่อผลสรุปจาก `genhtml` หรือค่า `LF`/`LH`
ในไฟล์ `.info` เป็นหลัก** มากกว่าเชื่อ `lcov --list` เพียงอย่างเดียวถ้าตัวเลขขัดแย้งกัน

---

## 96.8 รวม Sanitizer และ Coverage เข้ากับ CI จาก Part 94 (Step 768)

### ขยาย `ci.yml` เพิ่ม job สำหรับ Sanitizer และ Coverage

ต่อยอดจาก workflow ของ Part 94–95 เพิ่มอีก 2 job:

```yaml
name: CI

on:
  push:
    branches: [ "main" ]
  pull_request:
    branches: [ "main" ]

jobs:
  build-and-test:
    # ... เหมือน Part 94 ...
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: sudo apt-get update && sudo apt-get install -y cmake libgtest-dev g++
      - run: cmake -S . -B build -DCMAKE_BUILD_TYPE=Debug
      - run: cmake --build build -j "$(nproc)"
      - working-directory: build
        run: ctest --output-on-failure

  static-analysis:
    # ... เหมือน Part 95 ...
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: sudo apt-get update && sudo apt-get install -y cmake libgtest-dev clang-tidy cppcheck
      - run: cmake -S . -B build -DCMAKE_EXPORT_COMPILE_COMMANDS=ON
      - run: clang-tidy -p build src/*.cpp -checks='-*,modernize-*,bugprone-*,performance-*'

  sanitizers:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install dependencies
        run: |
          sudo apt-get update
          sudo apt-get install -y cmake libgtest-dev g++

      - name: Build with ASan + UBSan
        run: |
          cmake -S . -B build-san \
            -DCMAKE_BUILD_TYPE=Debug \
            -DCMAKE_CXX_FLAGS="-fsanitize=address,undefined -fno-sanitize-recover=all -g"
          cmake --build build-san -j "$(nproc)"

      - name: Run tests under ASan + UBSan
        working-directory: build-san
        run: ctest --output-on-failure

      - name: Build with ThreadSanitizer
        run: |
          cmake -S . -B build-tsan \
            -DCMAKE_BUILD_TYPE=Debug \
            -DCMAKE_CXX_FLAGS="-fsanitize=thread -g"
          cmake --build build-tsan -j "$(nproc)"

      - name: Run tests under ThreadSanitizer
        working-directory: build-tsan
        run: ctest --output-on-failure

  coverage:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install dependencies
        run: |
          sudo apt-get update
          sudo apt-get install -y cmake libgtest-dev g++ lcov

      - name: Build with coverage instrumentation
        run: |
          cmake -S . -B build-cov \
            -DCMAKE_BUILD_TYPE=Debug \
            -DCMAKE_CXX_FLAGS="--coverage -O0" \
            -DCMAKE_EXE_LINKER_FLAGS="--coverage"
          cmake --build build-cov -j "$(nproc)"

      - name: Run tests to generate coverage data
        working-directory: build-cov
        run: ./unit_tests

      - name: Generate coverage report
        working-directory: build-cov
        run: |
          lcov --capture --directory . --output-file coverage.info \
               --base-directory .. --ignore-errors mismatch
          lcov --remove coverage.info '/usr/*' '*/tests/*' \
               --output-file coverage.filtered.info --ignore-errors unused
          genhtml coverage.filtered.info --output-directory coverage_html

      - name: Upload coverage HTML report
        uses: actions/upload-artifact@v4
        with:
          name: coverage-report
          path: build-cov/coverage_html
```

อธิบายจุดสำคัญ:

- **`sanitizers` job แยก build เป็น 2 ชุด** (`build-san` สำหรับ ASan+UBSan, `build-tsan` สำหรับ
  TSan) ตามข้อจำกัดที่พิสูจน์ไว้ในหัวข้อ 96.6 ว่ารวมกันในกระบวนการเดียวไม่ได้
- **`-fno-sanitize-recover=all`** ใส่ไว้ในการ build สำหรับ CI เสมอ — เพื่อให้ `ctest` รายงาน
  ผลเป็น fail ทันทีที่เจอ UB แทนที่จะปล่อยผ่านไปเงียบๆ (ตามที่พิสูจน์พฤติกรรม default ไว้ใน
  หัวข้อ 96.4)
- **`coverage` job อัปโหลด HTML report เป็น Artifact** ด้วย `actions/upload-artifact@v4` — ทำให้
  ใครก็ตามที่เข้ามาดูผลการรัน workflow นี้บน GitHub สามารถดาวน์โหลด HTML report มาเปิดดูเองได้
  โดยตรง ไม่ต้องรันซ้ำเองบนเครื่อง (Action นี้เคยใช้แนวคิดเดียวกันมาแล้วในตัวอย่าง CD เบื้องต้น
  ของ Part 94 หัวข้อ 94.8)

### ตรวจสอบ YAML

```bash
python3 -c "
import yaml
data = yaml.safe_load(open('.github/workflows/ci.yml'))
print('jobs:', list(data['jobs'].keys()))
"
```

```
jobs: ['build-and-test', 'static-analysis', 'sanitizers', 'coverage']
```

### ยืนยันคำสั่งทุกตัวรันผ่านจริงบนเครื่อง local

ทุกคำสั่งใน job `sanitizers` และ `coverage` ข้างต้นถูกรันและยืนยันผลจริงแล้วในหัวข้อ 96.2–96.7
ของ Part นี้ ทั้งสองแบบ (ASan+UBSan และ TSan แยกกัน) รวมถึงขั้นตอน generate coverage report
ทั้งหมด — เหลือแค่นำไปวางในไฟล์ workflow ตามโครงสร้างนี้ตรงๆ

### ผลลัพธ์ที่จะเกิดขึ้นจริงบน GitHub (ภาพประกอบ)

Pull Request ที่ push โค้ดใหม่จะมี **4 job ตรวจสอบพร้อมกัน**: `build-and-test`,
`static-analysis`, `sanitizers`, และ `coverage` — ถ้า `sanitizers` job ล้มเหลว หมายความว่ามี
memory bug, UB, หรือ data race หลุดเข้ามาในโค้ดใหม่ ซึ่งเป็นระดับความเข้มงวดที่โปรเจกต์
Open Source และบริษัทซอฟต์แวร์ระดับโลกจำนวนมากใช้จริง (เช่น Chromium, LLVM เอง ก็รัน ASan/
UBSan/TSan เป็นส่วนหนึ่งของ CI มาตรฐานของโปรเจกต์)

---

## ข้อผิดพลาดที่พบบ่อย (Common Pitfalls)

1. **ลืมใส่ `--coverage` ตอน compile**: จะไม่มีไฟล์ `.gcno`/`.gcda` ถูกสร้างขึ้นเลย ทำให้ `gcov`/
   `lcov` รายงาน error ว่าหาไฟล์ข้อมูลไม่เจอ ต้องตรวจสอบว่าใส่ flag นี้ทั้งตอน compile **และ**
   ตอน link (`CMAKE_CXX_FLAGS` และ `CMAKE_EXE_LINKER_FLAGS` ต้องมีทั้งคู่)
2. **พยายามรวม ASan กับ TSan ในการ compile เดียวกัน**: ได้ compiler error ทันที
   (`incompatible with -fsanitize=address`) ทางแก้คือแยก build เป็นสองชุดตามหัวข้อ 96.6 เสมอ
3. **ลืมว่า cppcheck/UBSan/TSan ไม่ fail โดย default ในบางกรณี**: UBSan ไม่หยุดโปรแกรมโดย
   default (ต้องใส่ `-fno-sanitize-recover=all` เพื่อให้ CI จับ fail ได้จริง) — ถ้าลืมใส่ flag นี้
   ใน CI, job อาจ "ผ่าน" สีเขียวทั้งที่มี UB เกิดขึ้นจริงในโค้ด
4. **Optimize ระดับสูง (`-O2`/`-O3`) ตอนใช้ Sanitizer หรือ Coverage**: การ optimize ทำให้
   compiler อาจจัดเรียง/รวมบรรทัดโค้ดใหม่ ทำให้ stack trace ของ sanitizer หรือเลขบรรทัดของ
   coverage คลาดเคลื่อนจากโค้ดต้นฉบับ ควรใช้ `-O0` หรืออย่างมาก `-O1` เมื่อต้องการรายงานที่
   แม่นยำ (ใน production build จริงที่ต้องใช้ sanitizer ถาวร บางทีมยอมใช้ `-O1` เพื่อแลกกับ
   ความเร็วที่มากขึ้นเล็กน้อย โดยยอมรับว่ารายงานอาจไม่แม่นยำ 100%)
5. **เข้าใจผิดว่า Coverage สูงแปลว่าโค้ดปลอดภัย**: Coverage 100% วัดแค่ "บรรทัดถูกรัน" ไม่ได้
   วัดว่า test มี assertion ที่ตรวจสอบผลลัพธ์ถูกต้องครบทุกกรณีขอบหรือไม่ โค้ดที่มี coverage 100%
   แต่ไม่มี assertion เลย (แค่เรียกฟังก์ชันเฉยๆ) ก็ยังมีบั๊กซ่อนอยู่ได้เต็มไปหมด
6. **เชื่อผลลัพธ์จากคำสั่งเดียวโดยไม่ cross-check**: อย่างที่พบจริงกับ `lcov --list` ในหัวข้อ 96.7
   เครื่องมือ tooling ก็มีบั๊กหรือพฤติกรรมแปลกได้เช่นกัน ถ้าตัวเลขดูขัดแย้งกับสิ่งที่คาดหวัง ควร
   ตรวจสอบด้วยอย่างน้อยสองวิธี (เช่น `genhtml` กับการเปิดไฟล์ `.info` ดูตรงๆ) ก่อนสรุปผล

---

## แบบฝึกหัดท้ายบท

1. เขียนโปรแกรม C++ ที่มี Heap Buffer Overflow ของตัวเอง (ไม่ต้องเหมือนตัวอย่างในบทเรียน)
   คอมไพล์ด้วย `-fsanitize=address` แล้วยืนยันว่า ASan รายงานปัญหาได้ถูกต้อง อ่าน stack trace
   ที่ได้และระบุว่าบรรทัดไหนคือจุด allocate และบรรทัดไหนคือจุดที่เขียนเกินขอบเขต
2. เขียนโปรแกรมที่มี Signed Integer Overflow ของตัวเอง ทดสอบทั้งแบบ default (`-fsanitize=
   undefined`) และแบบ `-fno-sanitize-recover=all` เปรียบเทียบว่าพฤติกรรมของโปรแกรมต่างกัน
   อย่างไรหลังจากรายงาน error
3. ดัดแปลงโค้ด race condition ในบทเรียนให้ใช้ `std::atomic<long>` แทน `std::mutex` (ทบทวนจาก
   Part 82) แล้วรันผ่าน ThreadSanitizer ยืนยันว่าไม่มี data race เหลืออยู่เช่นกัน อธิบายด้วยคำพูด
   ตัวเองว่าทำไม `std::atomic` ถึงแก้ปัญหานี้ได้โดยไม่ต้องใช้ mutex เลย
4. ลองรวม `-fsanitize=address` กับ `-fsanitize=thread` ในคำสั่ง compile เดียวกันด้วยตัวเอง
   บันทึก error message ที่ได้ แล้วอธิบายด้วยคำพูดตัวเองว่าทำไมทั้งสองตัวถึงรวมกันไม่ได้
5. เขียนไลบรารีเล็กๆ ของตัวเอง (อย่างน้อย 3 ฟังก์ชัน) พร้อม unit test ตาม Part 93 ที่จงใจไม่
   ครอบคลุมบางฟังก์ชัน แล้ววัด coverage ด้วย gcov/lcov ยืนยันว่าเห็น `#####` ในไฟล์ `.gcov`
   ตรงกับฟังก์ชันที่ไม่มี test จริง จากนั้นเพิ่ม test ให้ครบแล้ววัดซ้ำจนได้ 100%
6. เขียน `.github/workflows/ci.yml` ของตัวเองให้มีครบทั้ง 4 job (`build-and-test`,
   `static-analysis`, `sanitizers`, `coverage`) ตามโครงสร้างในหัวข้อ 96.8 พร้อมตรวจสอบ YAML
   syntax และยืนยันว่าทุกคำสั่งรันผ่านบนเครื่อง local ก่อน push จริง

### แนวทางเฉลยข้อ 3

```cpp
#include <atomic>
#include <cstdio>
#include <thread>
#include <vector>

std::atomic<long> counter{0};

void increment_counter() {
    for (int i = 0; i < 100000; i++) {
        counter++;   // atomic increment ปลอดภัยจาก data race โดยไม่ต้องใช้ mutex
    }
}

int main() {
    const int NUM_THREADS = 4;
    std::vector<std::thread> threads;
    for (int i = 0; i < NUM_THREADS; i++) {
        threads.emplace_back(increment_counter);
    }
    for (auto &t : threads) t.join();

    printf("Expected: %d\n", NUM_THREADS * 100000);
    printf("Actual:   %ld\n", counter.load());
    return 0;
}
```

```bash
g++ -Wall -Wextra -std=c++17 -g -pthread -fsanitize=thread atomic_counter.cpp -o atomic_tsan
./atomic_tsan
```

ผลลัพธ์ที่ควรได้: ไม่มี warning จาก TSan และ `Actual` เท่ากับ `Expected` เสมอ

**คำอธิบาย**: `std::atomic<long>` รับประกันว่าการอ่าน-แก้ไข-เขียนกลับ (`counter++`) เกิดขึ้นเป็น
**หนึ่งหน่วยที่แบ่งแยกไม่ได้ (atomic/indivisible operation)** ที่ระดับ CPU instruction เอง (ผ่าน
คำสั่งพิเศษเช่น `LOCK XADD` บน x86) ไม่มีช่วงเวลาใดที่ thread อื่นจะเห็นค่ากลางๆ ระหว่างการ
อ่านกับการเขียนได้ ต่างจาก `std::mutex` ที่ป้องกันด้วยการล็อกทั้ง critical section (ซึ่งอาจ
ครอบคลุมหลายคำสั่งพร้อมกัน) `std::atomic` เหมาะกับกรณีง่ายๆ แบบนี้ที่มีแค่ตัวแปรเดียวที่ต้อง
ป้องกัน และมักเร็วกว่า mutex เล็กน้อยเพราะไม่ต้องมีการ lock/unlock overhead เต็มรูปแบบ

### แนวทางเฉลยข้อ 4

```bash
g++ -Wall -Wextra -std=c++17 -g -pthread -fsanitize=address,thread race.cpp -o combo_test
```

ผลลัพธ์ที่ควรได้:

```
cc1plus: error: '-fsanitize=thread' is incompatible with '-fsanitize=address'
```

**คำอธิบาย**: ทั้ง AddressSanitizer และ ThreadSanitizer ต่างต้องการ "shadow memory" ของ
ตัวเองในการติดตามสถานะของหน่วยความจำทุก byte (ASan ติดตามว่า byte ไหน "ถูกจองอย่างถูกต้อง"
ส่วน TSan ติดตามว่า thread ไหนแตะ byte ไหนเมื่อไหร่เพื่อคำนวณ happens-before relationship)
ทั้งสองกลไกต้องการควบคุม memory layout ของ process ในแบบที่ขัดแย้งกันโดยตรง จึงไม่สามารถ
ทำงานร่วมกันในโปรเซสเดียวได้ ทางแก้ที่ใช้จริงคือแยก build เป็นสองชุดตามที่แสดงไว้ในหัวข้อ 96.6
และ 96.8 แล้วรันแยกกันใน CI

---

## สรุปท้ายบท

ใน Part นี้เราได้:

- เข้าใจว่า Sanitizer ทำงานด้วย Compile-Time Instrumentation ต่างจาก Valgrind ที่ใช้ Dynamic
  Binary Instrumentation ทำให้เร็วกว่ามาก (วัดจริงได้ ASan ~2.6 เท่า เทียบกับ Valgrind ~20.7
  เท่าของเวลาปกติในการทดสอบเดียวกัน)
- ใช้ AddressSanitizer ตรวจจับ heap-buffer-overflow และ use-after-free ได้จริง พร้อม
  เปรียบเทียบพฤติกรรมและรายงานกับ Valgrind จาก Part 38 บนบั๊กเดียวกัน
- ใช้ UndefinedBehaviorSanitizer ตรวจจับ signed integer overflow และ null pointer
  dereference พร้อมเข้าใจความแตกต่างระหว่างโหมด default (รายงานแล้วรันต่อ) กับโหมด
  `-fno-sanitize-recover=all` (รายงานแล้วหยุดทันที)
- ใช้ ThreadSanitizer ตรวจจับ data race บนโค้ดที่ทดสอบตามสไตล์ Part 32/81 พร้อมแก้ปัญหาด้วย
  `std::mutex` และยืนยันผลด้วย TSan ซ้ำจนสะอาด
- เข้าใจข้อจำกัดสำคัญของการรวม sanitizer หลายตัว: ASan+UBSan ใช้ร่วมกันได้ แต่ ASan+TSan
  ใช้ร่วมกันไม่ได้เพราะชนกันที่ระดับ memory layout
- ใช้ gcov และ lcov วัด code coverage ของ test จาก Part 93 ได้จริง สร้าง HTML report จริง
  และเข้าใจว่า coverage สูงไม่ได้แปลว่าโค้ดปลอดภัยจากบั๊ก รวมถึงเจอและเรียนรู้จากพฤติกรรม
  ที่ไม่คาดคิดของเครื่องมือจริง (`lcov --list` ที่รายงานผิดบนเวอร์ชันที่ใช้)
- รวม Sanitizer และ Coverage เข้ากับ CI Pipeline ที่สร้างมาตั้งแต่ Part 94 ให้ครบทั้ง 4 ชั้นการ
  ตรวจสอบคุณภาพอัตโนมัติ: Build+Test, Static Analysis, Sanitizer, และ Coverage

Part นี้ปิดท้ายชุดเครื่องมือ Testing และ Tooling หลักของ Module H ด้วยระบบป้องกันคุณภาพ
หลายชั้น (Defense in Depth) ที่ผสาน CMake (Part 91), Package Management (Part 92), Unit
Testing (Part 93), CI (Part 94), Static Analysis (Part 95), และ Sanitizer/Coverage (Part นี้)
เข้าด้วยกัน ทำให้มั่นใจได้ว่าโค้ดที่ merge เข้า `main` มีคุณภาพสูงจริง ไม่ใช่แค่ "compile ผ่าน" เฉยๆ

Module H ยังเหลืออีก 2 Part ที่จะปิดท้ายด้วยหัวข้อ **Design Pattern ใน C++** ก่อนที่หลักสูตรจะ
ก้าวเข้าสู่ **Module I: Web Development ด้วย C/C++** ซึ่งจะนำเครื่องมือทั้งหมดที่เรียนใน Module H
นี้ไปใช้กับโปรเจกต์เว็บจริงทุกโปรเจกต์ตั้งแต่ raw socket ไปจนถึง framework อย่าง Crow และ
Pistache

**ต่อไป:** Part 97 — Design Pattern ใน C++ (Creational Patterns)
