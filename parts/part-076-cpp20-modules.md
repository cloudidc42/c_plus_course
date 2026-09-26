# Part 76: C++20 Modules (Step 601–608)

> Module F — Modern C++ (C++11 ถึง C++23) | Part 76 จาก 125
> Step ที่ครอบคลุมใน Part นี้: Step 601–608
> Part ก่อนหน้า: [Part 75 — C++20 Coroutines](./part-075-cpp20-coroutines.md) | Part ถัดไป: [Part 77 — ภาพรวม C++23](./part-077-cpp23-overview.md)

## เป้าหมายการเรียนรู้ (Learning Objectives)

หลังจบ Part นี้ ผู้เรียนจะสามารถ:

1. อธิบายปัญหาของระบบ `#include`/header file แบบเดิมได้อย่างละเอียด ทั้งเรื่องความเร็วในการ
   คอมไพล์ การไม่มี encapsulation จริง และปัญหา macro รั่วไหลข้ามไฟล์
2. อธิบายได้ว่า C++20 module คืออะไร และแก้ปัญหาแต่ละข้อของ header แบบเดิมอย่างไร
3. เขียน module interface unit พื้นฐานด้วย `export module` และ `import` ได้เอง
4. แยก module interface กับ module implementation ออกจากกันได้ (คล้ายกับการแยก `.h`/`.c`
   ที่เรียนใน Part 17 แต่เป็นกลไกระดับภาษา ไม่ใช่แค่ convention)
5. คอมไพล์และรัน module จริงบนเครื่องด้วย GCC 13.3.0 ผ่านแฟล็ก `-fmodules-ts` ได้ พร้อมรู้จัก
   ข้อจำกัดเฉพาะตัวของ toolchain นี้ (เช่น เรื่องนามสกุลไฟล์)
6. อธิบายสถานะการรองรับ module จริงในคอมไพเลอร์หลักๆ (GCC/Clang/MSVC) ณ ปี 2026 ได้อย่าง
   ตรงไปตรงมา รวมถึงเหตุผลที่ header ยังคงถูกใช้เป็นหลักในโปรเจกต์ real-world จำนวนมาก
7. ตัดสินใจได้ว่าเมื่อไหร่ควรทดลองใช้ module และเมื่อไหร่ควรยึดกับ header แบบเดิมต่อไปในงานจริง

---

## 76.1 ทบทวนปัญหาของ #include/Header File แบบเดิม (Step 601)

ย้อนกลับไป **Part 1** เราเรียนรู้ว่า `#include` คือคำสั่งของ **Preprocessor** ที่แค่
"copy-paste" เนื้อหาของไฟล์ที่ระบุเข้ามาแทนที่บรรทัดนั้นตรงๆ แบบดิบๆ ไม่มีความฉลาดใดๆ
ทั้งสิ้น และ **Part 17** เราเรียนรู้การจัดระบบ header/`.c` เพื่อทำ modular programming
ด้วยกลไกนี้ ทั้งสอง Part นั้นได้ผลจริงมาตลอด 50 ปีของภาษา C/C++ แต่ก็แบกปัญหาเชิงโครงสร้าง
3 ข้อใหญ่มาตลอดเช่นกัน:

### ปัญหาที่ 1: คอมไพล์ช้า เพราะ Preprocessor แปะซ้ำๆ

ลองนึกภาพโปรเจกต์ที่มีไฟล์ `.cpp` 500 ไฟล์ และแต่ละไฟล์ `#include <vector>`,
`#include <string>`, `#include "common.h"` — **ทุกครั้งที่คอมไพล์แต่ละ `.cpp`**
Preprocessor จะต้องเปิดไฟล์ header เหล่านั้น **แปะเนื้อหาทั้งหมดเข้ามาใหม่ทุกครั้ง** แล้ว
Compiler ต้อง parse ทุกบรรทัดของ header เหล่านั้นใหม่ทั้งหมดอีกครั้ง แม้เนื้อหาของ header
จะไม่เคยเปลี่ยนแปลงเลยก็ตาม

```bash
# ทบทวนจาก Part 1: ลองดูว่า #include <iostream> ขยายออกมากี่บรรทัด
echo '#include <iostream>' | g++ -E -x c++ - | wc -l
```

บนเครื่องที่ใช้ทดสอบหลักสูตรนี้ คำสั่งข้างต้นให้ผลลัพธ์กว่า **30,000 บรรทัด** — และถ้ามี
500 ไฟล์ `.cpp` ที่ `#include <iostream>` ก็แปลว่า compiler ต้อง parse โค้ดชุดเดียวกันนี้
ซ้ำถึง 500 ครั้ง! แม้จะมี **include guard** (`#ifndef`/`#pragma once`) ป้องกันไม่ให้เนื้อหา
ถูก**นิยามซ้ำ**ภายในไฟล์เดียวกัน แต่มันป้องกันได้แค่ "ภายใน 1 translation unit" เท่านั้น
ไม่ได้ช่วยเรื่อง**การ parse ซ้ำข้าม translation unit** เลยแม้แต่น้อย — นี่คือสาเหตุหลักที่
โปรเจกต์ C++ ใหญ่ๆ ใช้เวลาคอมไพล์เป็นสิบนาทีถึงหลายชั่วโมง

### ปัญหาที่ 2: ไม่มี True Encapsulation

Header file ไม่มีกลไกซ่อนอะไรได้จริงในระดับภาษา ทุกอย่างที่เขียนไว้ใน header (แม้แต่
ฟังก์ชัน helper ภายในที่ไม่อยากให้ใครใช้) จะถูกมองเห็นได้จากทุกไฟล์ที่ `#include` มัน
ธรรมเนียมที่ใช้กันมาคือแยกไปไว้ใน `namespace detail`/`namespace internal` หรือใส่ prefix
`_` แต่ทั้งหมดนี้เป็นแค่ **ข้อตกลงทางสังคม (convention)** ไม่มีอะไรบังคับทางเทคนิคเลยว่า
โค้ดภายนอกจะเรียกใช้สิ่งเหล่านั้นไม่ได้

### ปัญหาที่ 3: Macro รั่วไหลข้ามไฟล์

ทบทวนจาก **Part 14**: macro ที่ประกาศด้วย `#define` ไม่มีขอบเขต (scope) แบบตัวแปรทั่วไป
เลย มันมีผลตั้งแต่บรรทัดที่ประกาศไปจนจบไฟล์ (หรือจนกว่าจะ `#undef`) และที่อันตรายกว่านั้นคือ
**เมื่อ header ถูก `#include` เข้าไปในไฟล์อื่น macro ของมันก็รั่วไหลเข้าไปในไฟล์นั้นด้วย**
ปัญหาคลาสสิกที่โปรแกรมเมอร์ C++ เจอกันบ่อยคือ header ของบางไลบรารี (โดยเฉพาะบน Windows)
`#define max` หรือ `#define min` ไว้ ทำให้โค้ดที่เรียก `std::max()` ในไฟล์ที่ `#include`
header นั้นพังทันทีโดยไม่รู้สาเหตุ เพราะ `max` ถูกแทนที่ด้วย macro ไปแล้วตั้งแต่ก่อน
compiler จะเห็นโค้ดจริงด้วยซ้ำ

| ปัญหา | สาเหตุที่แท้จริง |
|---|---|
| คอมไพล์ช้า | Header ถูก parse ใหม่ทุกครั้งในทุก translation unit ที่ `#include` มัน |
| ไม่มี encapsulation จริง | ทุกอย่างใน header มองเห็นได้หมด ไม่มีกลไกภาษาบังคับซ่อน |
| Macro รั่วไหล | `#define` ไม่มีขอบเขต ส่งผลข้ามไฟล์ผ่าน `#include` ได้เสมอ |
| ลำดับ include สำคัญเกินไป | บาง header ต้อง include ก่อนอันอื่นเสมอ ไม่งั้น error แปลกๆ |

C++20 Modules ถูกออกแบบมาเพื่อแก้ปัญหาทั้ง 4 ข้อนี้โดยตรงในระดับภาษา ไม่ใช่แค่ convention

---

## 76.2 Module คืออะไร แก้ปัญหาอย่างไร (Step 602)

**Module** คือหน่วยการคอมไพล์ (compilation unit) รูปแบบใหม่ที่แทนที่แนวคิด "header +
`#include`" ด้วยแนวคิดใหม่ทั้งหมด: **compile หนึ่งครั้ง สร้างเป็นไฟล์ interface ที่คอมไพล์
ไว้ล่วงหน้า (เรียกทั่วไปว่า BMI — Built Module Interface) แล้วไฟล์อื่นๆ แค่ "import" ผลลัพธ์
ที่คอมไพล์ไว้แล้วนั้นเข้ามาใช้** โดยไม่ต้อง parse source code ของ module นั้นซ้ำอีกเลย

### แก้ปัญหาความเร็ว

เพราะ module ถูกคอมไพล์เป็น binary interface (บน GCC เรียกไฟล์นี้ว่า `.gcm`) เพียงครั้งเดียว
translation unit อื่นที่ `import` module นั้นจะแค่โหลดไฟล์ binary ที่คอมไพล์ไว้แล้วเข้ามา
**ไม่ต้อง parse text ซ้ำเหมือน `#include`** ในทางทฤษฎีเมื่อโปรเจกต์มีขนาดใหญ่มากและมีการ
`import` module เดียวกันจากหลายร้อยไฟล์ ผลต่างของเวลาคอมไพล์ทั้งโปรเจกต์จะเห็นชัดเจน

### แก้ปัญหา Encapsulation

Module มีกลไกที่ภาษาบังคับจริง: สิ่งใดไม่ได้ใส่ keyword `export` ไว้ **จะมองไม่เห็นจาก
ภายนอก module นั้นเลย** ไม่ใช่แค่ convention แต่เป็นกฎที่ compiler บังคับใช้ (compile error
ทันทีถ้าพยายามเรียกสิ่งที่ไม่ได้ export) — จะสาธิตให้เห็นจริงใน 76.5

### แก้ปัญหา Macro รั่วไหล

Macro ที่ประกาศภายใน module **ไม่รั่วไหลออกไปให้ไฟล์ที่ `import` module นั้น** เพราะ
macro เป็นเรื่องของ Preprocessor ซึ่งทำงานเสร็จสิ้นไปแล้วก่อนที่ module จะถูกคอมไพล์เป็น
binary interface การ `import` ไม่ใช่การ "แปะ text" แบบ `#include` จึงไม่มีทางที่ macro
จะข้ามขอบเขตของ module ไปได้ (ยกเว้นตั้งใจ `export` ผ่านกลไกอื่นซึ่งไม่แนะนำ)

### ตารางเปรียบเทียบ

| คุณสมบัติ | `#include` (Header แบบเดิม) | `import` (C++20 Module) |
|---|---|---|
| กลไกเบื้องหลัง | Preprocessor แปะ text ทั้งไฟล์ | Compiler โหลด binary interface ที่คอมไพล์ไว้แล้ว |
| ถูก parse กี่ครั้ง | ทุกครั้งที่มีการ include (ในทุก TU) | ครั้งเดียวตอนสร้าง module แล้วนำกลับมาใช้ซ้ำ |
| Encapsulation | ไม่มี (ทุกอย่างมองเห็นหมด) | มีจริง (เห็นเฉพาะที่ `export` เท่านั้น) |
| Macro รั่วไหลข้ามไฟล์ | รั่วไหลได้เสมอ | ไม่รั่วไหล (macro จบอยู่แค่ในตัว module) |
| ลำดับการ include/import สำคัญไหม | สำคัญมาก บาง header ต้องมาก่อนเสมอ | ไม่สำคัญ `import` เรียงลำดับใดก็ได้ |

---

## 76.3 Syntax พื้นฐานของ Module (Step 603)

### ประกาศ Module Interface Unit

```cpp
// ไฟล์ที่ "เป็น" module — เรียกว่า module interface unit
export module ชื่อโมดูล;   // ต้องเป็นบรรทัดแรกสุดของไฟล์ (ก่อน #include ใดๆ ด้วยซ้ำ)

// ประกาศ/นิยามอะไรก็ได้ตามปกติ แต่ต้องมี export หน้าสิ่งที่อยากให้ไฟล์อื่นมองเห็น
export int my_function(int x);

// ไม่มี export = ใช้ได้แค่ภายใน module นี้เท่านั้น มองไม่เห็นจากภายนอก
int internal_helper(int x);
```

### Import Module

```cpp
// ไฟล์ที่ "ใช้" module
import ชื่อโมดูล;

int main() {
    my_function(42);       // เรียกได้ เพราะถูก export ไว้
    // internal_helper(1); // เรียกไม่ได้! compile error ทันที
}
```

### export ทีละกลุ่มด้วย export block

ถ้ามีหลายอย่างที่อยากจะ export พร้อมกัน สามารถใช้ block `export { ... }` แทนการใส่
`export` หน้าทุกบรรทัดได้:

```cpp
export module shapes;

export {
    struct Point { double x, y; };
    double distance(Point a, Point b);
}
```

### สรุป keyword ที่เกี่ยวข้อง

| Syntax | ความหมาย |
|---|---|
| `export module X;` | ประกาศว่าไฟล์นี้คือ **primary module interface unit** ของ module ชื่อ `X` |
| `module X;` (ไม่มี `export`) | ประกาศว่าไฟล์นี้คือ **module implementation unit** — ให้ implementation เพิ่มเติมของ `X` |
| `import X;` | นำ module `X` เข้ามาใช้งาน |
| `export <declaration>` | ทำให้สิ่งที่ประกาศมองเห็นได้จากภายนอก module |
| `export import X;` | import module `X` แล้ว "ส่งต่อ" ให้คนที่ import module ของเราเห็น `X` ด้วย (re-export) |

---

## 76.4 ตัวอย่างที่คอมไพล์ได้จริงบนเครื่องนี้ (Step 604)

ก่อนอื่นต้องพูดตรงๆ ตามหลักความซื่อสัตย์ของหลักสูตรนี้: **GCC 13.3.0 บนเครื่องที่ใช้เขียน
บทเรียนนี้ยังไม่มีแฟล็กมาตรฐานชื่อ `-fmodules`** (แฟล็กนี้ถูกเพิ่มในบางเวอร์ชันของ GCC
รุ่นหลังๆ) สิ่งที่มีอยู่คือ **`-fmodules-ts`** ซึ่งเป็นการรองรับตาม **Technical
Specification (TS)** ที่ออกมาก่อนมาตรฐาน C++20 ตัวจริง — ทดสอบยืนยันแล้วว่า:

```bash
$ g++ -std=c++20 -fmodules -c math.cc -o math.o
g++: error: unrecognized command-line option '-fmodules'; did you mean '-Mmodules'?
```

แต่ข่าวดีคือ **`-fmodules-ts` ใช้งานได้จริง** และรองรับ syntax หลักของ module ตาม C++20
เพียงพอสำหรับตัวอย่างพื้นฐานถึงระดับกลาง ลองมาสร้าง module แรกกัน

### ข้อควรระวังเรื่องนามสกุลไฟล์ (พบจากการทดสอบจริงบนเครื่องนี้)

มาตรฐาน C++ **ไม่ได้กำหนดนามสกุลไฟล์ตายตัว** สำหรับ module interface unit — แต่ละ
คอมไพเลอร์เลือกไม่เหมือนกัน (Clang นิยม `.cppm`, MSVC นิยม `.ixx`) เมื่อทดสอบบนเครื่องนี้
พบพฤติกรรมที่ **อันตรายมาก** ถ้าไม่รู้ล่วงหน้า:

```bash
$ g++ -std=c++20 -fmodules-ts -c math.cppm -o math.o
g++: warning: math.cppm: linker input file unused because linking not done
$ echo $?
0
$ ls math.o
ls: cannot access 'math.o': No such file or directory
```

**สังเกตให้ดี: exit code เป็น 0 (สำเร็จ!) แต่ไม่มีไฟล์ `math.o` ถูกสร้างขึ้นมาเลย**
เพราะ GCC ไม่รู้จักนามสกุล `.cppm` ว่าเป็นซอร์สโค้ด C++ จึงปฏิบัติกับมันเหมือนเป็นไฟล์
สำหรับ linker (เช่น `.o` หรือ `.a`) ที่ **ไม่ได้ถูกใช้เพราะเราสั่ง `-c` (ไม่ link)** ผลคือ
มันเงียบๆ ไม่ทำอะไรเลยแต่รายงานว่าสำเร็จ — เป็นกับดักที่ทำให้เข้าใจผิดว่าคอมไพล์ผ่านทั้งที่
จริงไม่ได้คอมไพล์อะไรเลยสักบรรทัด (จะพูดถึงอีกครั้งใน Common Pitfalls)

**ทางแก้ที่ทดสอบแล้วใช้ได้จริง 2 วิธี:**

```bash
# วิธีที่ 1: บังคับภาษาด้วย -x c++ (เก็บนามสกุล .cppm ไว้ได้ตามธรรมเนียมข้ามคอมไพเลอร์)
g++ -std=c++20 -fmodules-ts -x c++ -c math.cppm -o math.o

# วิธีที่ 2: เปลี่ยนนามสกุลเป็น .cc/.cpp ธรรมดา ที่ GCC รู้จักอยู่แล้ว
g++ -std=c++20 -fmodules-ts -c math.cc -o math.o
```

บทเรียนนี้จะใช้วิธีที่ 2 (นามสกุล `.cc`) ในตัวอย่างต่อจากนี้ เพื่อให้คำสั่งคอมไพล์สั้นและ
ไม่ต้องจำแฟล็กเพิ่ม แต่ผู้เรียนควรรู้ทั้งสองวิธีไว้ เพราะโค้ด module ที่เจอในโลกจริง (เช่น
ตัวอย่างจาก Clang/MSVC) มักใช้นามสกุล `.cppm`/`.ixx` เป็นค่าเริ่มต้น

### ตัวอย่างที่ 1: Module พื้นฐานที่สุด

```cpp
// math.cc  (module interface unit)
export module math;

export int add(int a, int b) {
    return a + b;
}

export int multiply(int a, int b) {
    return a * b;
}
```

```cpp
// main.cpp  (ไฟล์ที่ import module)
import math;
#include <iostream>

int main() {
    std::cout << add(2, 3) << " " << multiply(4, 5) << std::endl;
}
```

คอมไพล์เป็น 2 ขั้นตอน — ต้องสร้าง (compile) module ให้เสร็จก่อนเสมอ ก่อนจะคอมไพล์ไฟล์ที่
`import` มัน (นี่คือกฎสำคัญ: **module ต้องถูก build ก่อนถูก import เสมอ** ต่างจาก
`#include` ที่ลำดับไฟล์ไม่สำคัญ):

```bash
g++ -std=c++20 -fmodules-ts -c math.cc -o math.o     # (1) build module ก่อน
g++ -std=c++20 -fmodules-ts -c main.cpp -o main.o    # (2) ค่อยคอมไพล์ไฟล์ที่ import
g++ -std=c++20 -fmodules-ts math.o main.o -o main    # (3) link ตามปกติ
./main
```

ผลลัพธ์ (ทดสอบจริงบนเครื่องนี้):

```
5 20
```

ถ้าลองสลับลำดับ compile ไฟล์ที่ 2 ก่อนไฟล์ที่ 1 (import ก่อน build module) จะได้ error
ที่ชัดเจนมาก (ข้อความจริงจาก GCC):

```
In module imported at main.cpp:1:1:
math: error: failed to read compiled module: No such file or directory
math: note: compiled module file is 'gcm.cache/math.gcm'
math: note: imports must be built before being imported
math: fatal error: returning to the gate for a mechanical issue
compilation terminated.
```

สังเกตว่า GCC เก็บ binary interface ของ module ไว้ในโฟลเดอร์ชื่อ `gcm.cache/` ที่ถูกสร้าง
ขึ้นอัตโนมัติในโฟลเดอร์ที่รันคำสั่งคอมไพล์ (ไฟล์ `.gcm` คือสิ่งที่มาตรฐานเรียกทั่วไปว่า
BMI — Built Module Interface — เนื้อหาของมันเป็น binary เฉพาะของ GCC เวอร์ชันนั้นๆ
ไม่สามารถใช้ข้ามคอมไพเลอร์หรือบางครั้งข้ามเวอร์ชันได้เลย ซึ่งเป็นข้อจำกัดสำคัญที่จะพูดถึงใน
76.6)

### ตัวอย่างที่ 2: แยก Module Interface กับ Implementation (เทียบกับ Part 17)

จุดที่น่าสนใจคือ module รองรับการแยก "ประกาศ" กับ "implementation" ออกจากกันได้ คล้ายกับ
การแยก `.h`/`.c` ใน Part 17 แต่คราวนี้เป็นกลไกภาษาโดยตรง ไม่ใช่แค่ convention:

```cpp
// shapes.cc  (module interface unit — มีแค่ "ประกาศ")
export module shapes;

export double circle_area(double radius);   // แค่ประกาศ ยังไม่มี implementation
```

```cpp
// shapes_impl.cc  (module implementation unit — ให้ implementation จริง)
module shapes;   // สังเกต: ไม่มี "export" ตรงนี้ เพราะนี่คือ implementation unit
                  // ไม่ใช่ primary interface unit

double circle_area(double radius) {
    return 3.14159265358979 * radius * radius;
}
```

```cpp
// use_shapes.cpp
import shapes;
#include <iostream>

int main() {
    std::cout << circle_area(2.0) << "\n";
}
```

```bash
g++ -std=c++20 -fmodules-ts -c shapes.cc -o shapes.o
g++ -std=c++20 -fmodules-ts -c shapes_impl.cc -o shapes_impl.o
g++ -std=c++20 -fmodules-ts -c use_shapes.cpp -o use_shapes.o
g++ -std=c++20 -fmodules-ts shapes.o shapes_impl.o use_shapes.o -o use_shapes
./use_shapes
```

ทดสอบจริงแล้ว ได้ผลลัพธ์:

```
12.5664
```

โน้ตสำคัญ: module implementation unit (`shapes_impl.cc`) เขียน `module shapes;` (ไม่มี
`export` นำหน้า) เพื่อบอกว่า "ฉันคือส่วนขยายของ module `shapes`" ไฟล์นี้**เข้าถึงทุกอย่าง
ใน module `shapes` ได้หมด แม้จะไม่ได้ export** (มองเห็นทั้ง export และ non-export จากมุมมอง
ภายใน module เดียวกัน) แต่ตัวมันเองก็ไม่สามารถ export อะไรเพิ่มให้คนภายนอกเห็นได้อีก
(สิทธิ์ export ทำได้เฉพาะใน primary interface unit เท่านั้น)

---

## 76.5 Encapsulation จริงของ Module: พิสูจน์ด้วยการคอมไพล์ (Step 605)

นี่คือจุดที่ทำให้เห็นว่า module ต่างจาก header อย่างเป็นรูปธรรม ลองเขียน module ที่มี
ฟังก์ชันช่วยภายในที่ไม่ export:

```cpp
// geometry.cc
export module geometry;

// ฟังก์ชันช่วยภายใน ไม่ export -> มองไม่เห็นจากนอก module (encapsulation จริง)
namespace {
    double square(double x) {
        return x * x;
    }
}

export struct Point {
    double x;
    double y;
};

export double distance(Point a, Point b) {
    return __builtin_sqrt(square(a.x - b.x) + square(a.y - b.y));
}
```

```cpp
// use_geometry.cpp — ใช้งานปกติ (เรียกเฉพาะสิ่งที่ export)
import geometry;
#include <iostream>

int main() {
    Point p1{0.0, 0.0};
    Point p2{3.0, 4.0};
    std::cout << "distance = " << distance(p1, p2) << std::endl;
}
```

```bash
g++ -std=c++20 -fmodules-ts -c geometry.cc -o geometry.o
g++ -std=c++20 -fmodules-ts -c use_geometry.cpp -o use_geometry.o
g++ -std=c++20 -fmodules-ts geometry.o use_geometry.o -o use_geometry
./use_geometry
# distance = 5
```

ทดสอบจริง ได้ผลลัพธ์ `distance = 5` ตามคาด แต่ที่น่าสนใจคือลองสร้างไฟล์ที่ **พยายาม
แอบเรียก `square()` ที่ไม่ได้ export**:

```cpp
// leak_test.cpp
import geometry;

int main() {
    return (int)square(2.0);   // พยายามเรียกฟังก์ชันที่ไม่ได้ export
}
```

```bash
g++ -std=c++20 -fmodules-ts -c leak_test.cpp -o leak_test.o
```

ผลลัพธ์ (ทดสอบจริง — compile error ทันที):

```
leak_test.cpp: In function 'int main()':
leak_test.cpp:3:17: error: 'square' was not declared in this scope
    3 |     return (int)square(2.0);
      |                 ^~~~~~
```

**นี่คือความแตกต่างที่สำคัญที่สุดจาก header แบบเดิม** ถ้า `square()` เขียนไว้ใน header
ธรรมดาที่ `#include` เข้ามา ไม่ว่าจะใส่ไว้ใน `namespace` ไม่ระบุชื่อ (anonymous namespace)
หรือพยายามซ่อนด้วยวิธีใดในระดับ convention มันก็ยังจะถูกมองเห็นได้ในบางกรณี (เช่น ถ้าไม่ได้
ใส่ `static`/anonymous namespace ให้ถูกต้อง) แต่กับ module compiler **บังคับ**ไม่ให้เห็น
สิ่งที่ไม่ได้ `export` เลย ไม่มีทางเลี่ยงในระดับภาษา

---

## 76.6 สถานะการรองรับจริงในคอมไพเลอร์ปัจจุบัน ปี 2026 (Step 606)

แม้ตัวอย่างข้างบนจะคอมไพล์และรันได้จริงบนเครื่องนี้ แต่ต้องพูดตรงไปตรงมาว่า **สถานะการ
รองรับ module ในระบบนิเวศ C++ โดยรวม ณ ปี 2026 ยังตามหลัง header file แบบเดิมอยู่มาก**
ในโปรเจกต์ real-world ส่วนใหญ่ ด้วยเหตุผลหลายข้อ:

| ประเด็น | สถานะโดยรวม |
|---|---|
| **GCC** | รองรับผ่าน `-fmodules-ts` (TS-based) ใช้งานได้ในระดับพื้นฐานถึงกลางอย่างที่เห็นในบทนี้ แต่ยังไม่มี flag `-fmodules` มาตรฐานตาม ISO C++20 เต็มรูปแบบบนเวอร์ชัน 13.x ที่ทดสอบ |
| **Clang** | รองรับดีกว่าในแง่ tooling (แฟล็ก `-fmodules`, ใช้นามสกุล `.cppm` ได้ตรงๆ) และมีการพัฒนา `import std;` (standard library module) ต่อเนื่อง แต่ก็ยังต้องระบุ flag การ scan dependency เพิ่มเติมในหลายกรณี |
| **MSVC** | มักถูกมองว่ารองรับ module ได้เสถียรที่สุดในบรรดาสามค่าย เพราะ Visual Studio integrate การ scan/build module เข้ากับระบบ build โดยตรง (นามสกุล `.ixx`) |
| **Build System (CMake, Ninja, Bazel)** | การรองรับ module ต้องอาศัย "dependency scanning" พิเศษ (รู้ว่าไฟล์ไหน export module ชื่ออะไร ต้อง build ก่อนไฟล์ไหน) ซึ่งเพิ่งเริ่มเสถียรใน CMake รุ่นใหม่ๆ (ต้องใช้ CMake และ generator เวอร์ชันที่รองรับ `CXX_MODULES` โดยเฉพาะ) โปรเจกต์เก่าจำนวนมากยังไม่ได้ปรับมาใช้ |
| **Standard Library เป็น Module** (`import std;`) | เป็นแนวคิดที่น่าตื่นเต้นมาก (import STL ทั้งชุดในบรรทัดเดียว แทน `#include` เป็นสิบไฟล์) แต่การรองรับยังไม่แพร่หลายเท่ากันในทุกคอมไพเลอร์/เวอร์ชันไลบรารีมาตรฐานที่ใช้งานทั่วไป |
| **Header Unit** (`import <header>;` การ import header เดิมแบบ module) | ตามมาตรฐานมีแนวคิดนี้อยู่ แต่ต้อง precompile header เป็น BMI ก่อนใช้งานเช่นกัน (ทดสอบบนเครื่องนี้พบว่ายังต้องตั้งค่าเพิ่มเติมพอสมควร ดู 76.7) |
| **Binary Compatibility ของ BMI ข้ามคอมไพเลอร์/เวอร์ชัน** | ไม่มีมาตรฐานกลาง — ไฟล์ `.gcm` ของ GCC ใช้กับ Clang ไม่ได้ และบางครั้งก็ใช้ข้าม GCC คนละเวอร์ชันไม่ได้ด้วยซ้ำ ทำให้การแจกจ่ายไลบรารีเป็น "precompiled module" ข้าม toolchain แทบเป็นไปไม่ได้ในทางปฏิบัติ ต่างจาก header ที่เป็น text ธรรมดาที่คอมไพล์ได้ทุกที่เสมอ |

ทดสอบยืนยันประเด็น `import std;` โดยตรงบนเครื่องนี้ (ทั้ง `-std=c++20` และ `-std=c++23`):

```cpp
import std;
int main() { std::cout << "test\n"; }
```

```
In module imported at import_std_test.cpp:1:1:
std: error: failed to read compiled module: No such file or directory
std: note: compiled module file is 'gcm.cache/std.gcm'
std: note: imports must be built before being imported
```

ผลลัพธ์เดียวกันทั้งสองมาตรฐาน — เพราะ GCC 13.3.0 ไม่ได้แจก standard library ในรูปแบบ module
สำเร็จรูป (precompiled `std.gcm`) มาให้ใช้ทันที ผู้ใช้ต้อง build module `std` เองจาก source
ของ libstdc++ ก่อน (ซึ่งซับซ้อนและอยู่นอกเหนือขอบเขตของบทเรียนนี้) ยืนยันชัดเจนว่า
`import std;` ยังไม่ใช่ทางเลือกที่ใช้งานได้ทันทีบน toolchain นี้ ต่างจากที่บางบทความออนไลน์
อาจพูดถึงในเชิงทฤษฎีล้วนๆ

**สรุปตรงไปตรงมา**: ในโปรเจกต์ open-source และองค์กรขนาดใหญ่จำนวนมาก ณ ปี 2026 **ยังคง
ใช้ header file (`#include`) เป็นวิธีหลักในการจัดโครงสร้างโค้ด** เหตุผลหลักไม่ใช่เพราะ
module แย่ในทางทฤษฎี (ตรงกันข้าม มันแก้ปัญหาที่แท้จริงตามที่อธิบายใน 76.1–76.2) แต่เป็น
เพราะ:

1. **ต้นทุนการย้าย (migration cost) สูง** — โค้ดเบสขนาดหลักล้านบรรทัดที่มีอยู่แล้วเปลี่ยน
   มาใช้ module ทั้งหมดเป็นงานใหญ่มาก
2. **Build system ยังไม่เสถียรสมบูรณ์ในทุก toolchain** ทำให้ทีมที่ต้อง build ข้าม
   compiler/platform (Linux + Windows + macOS) เสี่ยงเจอปัญหาที่ทำนายไม่ได้
3. **ไลบรารีบุคคลที่สาม (third-party) ส่วนใหญ่ยังแจกจ่ายเป็น header/source ธรรมดา**
   ไม่ได้แจกเป็น module ทำให้โปรเจกต์ที่พึ่งพาไลบรารีเหล่านั้นได้ประโยชน์จาก module
   เฉพาะโค้ดของตัวเองบางส่วนเท่านั้น
4. **ทีมพัฒนาจำนวนมากยังไม่คุ้นเคย** กับ syntax และ mental model ใหม่ของ module รวมถึง
   เครื่องมือ (IDE, static analyzer, linter) บางตัวก็ยังตามการรองรับไม่ทัน 100%

โปรเจกต์ใหม่ (greenfield) ขนาดเล็กถึงกลางที่ยึด toolchain เดียวตายตัว (เช่น ใช้ MSVC
เท่านั้น หรือ Clang เท่านั้น) คือกลุ่มที่ได้ประโยชน์จาก module มากที่สุดในตอนนี้ ส่วน
โปรเจกต์ใหญ่ที่ต้อง cross-platform และพึ่งพาไลบรารีเก่าจำนวนมาก ส่วนใหญ่ยังรอให้ ecosystem
เสถียรกว่านี้ก่อนย้ายเต็มรูปแบบ

---

## 76.7 ข้อจำกัดที่พบจากการทดสอบจริงบนเครื่องนี้ (Step 607)

เพื่อความโปร่งใส ขอสรุปข้อจำกัดที่พบจริงระหว่างทดสอบตัวอย่างในบทนี้บน GCC 13.3.0 พร้อม
`-fmodules-ts`:

1. **นามสกุลไฟล์ `.cppm` ไม่ถูกจดจำโดยอัตโนมัติ** — ต้องใช้ `-x c++` หรือเปลี่ยนนามสกุลเป็น
   `.cc`/`.cpp` ตามที่อธิบายใน 76.4 (พฤติกรรมนี้อาจต่างกันไปในแต่ละเวอร์ชัน/distro ของ GCC)

2. **การ import standard library header เป็น header unit ยังไม่ทำงานทันทีแบบ out-of-the-box**
   ทดสอบแล้วพบว่า:
   ```cpp
   import <iostream>;   // header unit — พยายาม import <iostream> แบบ module
   ```
   ให้ผลลัพธ์:
   ```
   In module imported at headerunit.cpp:1:1:
   /usr/include/c++/13/iostream: error: failed to read compiled module: No such file or directory
   /usr/include/c++/13/iostream: note: compiled module file is 'gcm.cache/./usr/include/c++/13/iostream.gcm'
   /usr/include/c++/13/iostream: note: imports must be built before being imported
   ```
   กล่าวคือ ต้อง precompile `<iostream>` เป็น header unit ก่อนแยกต่างหาก (ด้วยคำสั่งเพิ่มเติม
   เช่น `-fmodules-ts -x c++-header --precompile`) ซึ่งซับซ้อนกว่าการใช้ `#include <iostream>`
   ธรรมดามาก จึงไม่คุ้มค่าสำหรับตัวอย่างในหลักสูตรนี้ และนี่คือเหตุผลที่ทุกตัวอย่างข้างต้น
   ยังคง `#include <iostream>` ธรรมดาคู่ไปกับ `import` module ของเราเอง — เป็นแนวทางที่ใช้
   งานจริงได้และเป็นสิ่งที่แนะนำในช่วงเปลี่ยนผ่านนี้ (ผสม module ของตัวเองกับ header ของ
   standard library ได้ตามปกติ ไม่ขัดแย้งกัน)

3. **ไม่มี dependency scanning อัตโนมัติ** — ในตัวอย่างทั้งหมดของบทนี้ เราต้อง**สั่งคอมไพล์
   module interface ก่อนไฟล์ที่ import มันเองด้วยมือ** ทีละคำสั่ง ถ้าลืมลำดับจะเจอ error
   ตามที่แสดงใน 76.4 — ในโปรเจกต์จริงที่ใช้ module เยอะๆ จำเป็นต้องพึ่ง build system ที่รองรับ
   การ scan dependency ของ module โดยเฉพาะ (เช่น CMake รุ่นใหม่ที่รองรับ `CXX_MODULES`) ไม่ควร
   ไล่คอมไพล์ทีละไฟล์ด้วยมือแบบในบทเรียนนี้เมื่อโปรเจกต์ใหญ่ขึ้น

4. **โฟลเดอร์ `gcm.cache/` ถูกสร้างขึ้นในทุกจุดที่คอมไพล์** — ถ้าไม่จัดการให้ดี (เช่น เพิ่มใน
   `.gitignore` ตามที่เรียนใน Part 1) อาจมีไฟล์ binary แปลกปลอมหลุดเข้าไปใน git repository ได้

5. **การดึง header ของ standard library เข้ามาใน module (แม้ผ่าน "global module fragment"
   ที่ถูกต้องตามมาตรฐาน) ทำให้เกิด conflict รุนแรงเมื่อไฟล์อื่นที่ import module นั้นก็
   `#include` header ตัวเดียวกันแบบปกติด้วย** — นี่คือข้อจำกัดที่สำคัญที่สุดที่พบระหว่างทดสอบ
   บทเรียนนี้ ลองดูตัวอย่าง (เขียนตามมาตรฐานถูกต้องทุกอย่าง โดยใช้ `module;` เป็น global
   module fragment เพื่อใส่ `#include` ก่อนเข้าสู่ module purview):
   ```cpp
   // stringutils.cc
   module;
   #include <string>
   #include <algorithm>
   export module stringutils;

   export std::string to_upper(std::string s) { /* ... */ }
   ```
   ```cpp
   // main.cpp
   import stringutils;
   #include <iostream>   // <-- include header มาตรฐานตามปกติ

   int main() { /* ... */ }
   ```
   เมื่อคอมไพล์ `main.cpp` (ที่ `#include <iostream>` ซึ่งดึง `<string>`, `<type_traits>`
   และอื่นๆ เข้ามาแบบ textual ตามปกติ ซ้อนทับกับสิ่งที่ module `stringutils` ดึงเข้าไปแล้ว)
   จะพังยับด้วย error หลายสิบบรรทัด ตัวอย่างบางส่วนจาก g++ 13.3.0 จริง:
   ```
   error: redefinition of 'void* operator new(std::size_t, void*)'
   error: redefinition of default argument for 'class _Up'
   error: conflicting global module declaration 'template<class ... _Tp>
          using std::common_type_t = typename std::common_type@stringutils::type'
   ```
   สาเหตุคือ `-fmodules-ts` (ซึ่งอิงตาม Technical Specification ก่อนมาตรฐานตัวจริง) ยัง
   **ไม่สามารถ merge ประกาศเดียวกันที่มาจากสองทาง** (ทางหนึ่งผ่าน module, อีกทางผ่าน
   `#include` ธรรมดา) ให้เป็นสิ่งเดียวกันได้อย่างถูกต้องเสมอไป โดยเฉพาะกับ header ขนาดใหญ่
   ของ standard library อย่าง `<string>`/`<type_traits>` ที่มีการประกาศซับซ้อนจำนวนมาก
   **บทเรียนนี้จึงจงใจเลือกตัวอย่างที่ module ใช้แค่ fundamental type (`int`, `double`)
   ล้วนๆ ไม่ดึง standard library header เข้าไปใน module เลย** ซึ่งทดสอบแล้วว่าเสถียรและ
   ใช้งานได้จริง 100% ตามที่แสดงตลอดบทเรียนนี้ นี่คือตัวอย่างที่ชัดเจนที่สุดว่าทำไม 76.6
   ถึงบอกว่าการรองรับ module ในทางปฏิบัติยังไม่นิ่งพอสำหรับโค้ดที่พึ่งพา standard library
   หนักๆ บน toolchain รุ่นนี้

จุดยืนของหลักสูตรนี้: ตัวอย่างทั้งหมดที่แสดงในบทนี้ **คอมไพล์และรันได้จริง ตรวจสอบแล้วบน
เครื่องจริง** ไม่ใช่โค้ดที่เขียนขึ้นลอยๆ ตามทฤษฎี แต่ก็ต้องยอมรับตรงๆ ว่านี่คือการทดสอบใน
ระดับ "ตัวอย่างเดี่ยว" เท่านั้น ยังไม่ใช่การพิสูจน์ว่า module พร้อมใช้งานแบบเต็มรูปแบบใน
โปรเจกต์ขนาดใหญ่บน toolchain นี้

---

## 76.8 แนวทางเลือกใช้จริงในปี 2026: Module vs Header (Step 608)

สรุปเป็นแนวทางตัดสินใจที่ใช้งานได้จริง:

| สถานการณ์ | คำแนะนำ |
|---|---|
| โปรเจกต์ใหม่ ขนาดเล็ก-กลาง ใช้ toolchain เดียวตายตัว (เช่น MSVC หรือ Clang อย่างเดียว) | ลองใช้ module ได้ ประโยชน์ด้าน encapsulation และความเร็วคอมไพล์คุ้มค่าที่จะทดลอง |
| โปรเจกต์ที่ต้อง cross-platform ข้าม GCC/Clang/MSVC | ยังคงใช้ header แบบเดิมเป็นหลัก จนกว่า ecosystem จะเสถียรกว่านี้ |
| โปรเจกต์เก่าขนาดใหญ่ที่มีโค้ดหลักล้านบรรทัด | ไม่คุ้มที่จะ migrate ทั้งหมด อาจพิจารณาใช้ module เฉพาะโค้ดใหม่ที่เขียนเพิ่มเข้าไปเท่านั้น |
| ไลบรารีที่ต้องแจกจ่ายให้คนอื่นใช้ผ่านหลาย toolchain | แจกเป็น header ตามเดิม เพราะ BMI ไม่ compatible ข้ามคอมไพเลอร์ |
| งานเรียนรู้/ทดลองเพื่อเข้าใจอนาคตของภาษา | คุ้มค่ามากที่จะเรียนรู้ไว้ เพราะ module คือทิศทางระยะยาวของ C++ อย่างชัดเจน |

### ใช้ Makefile บังคับลำดับการ build module ให้ถูกต้อง (ทบทวนจาก Part 18)

ทบทวนจาก **Part 18**: เราใช้ `make` เพื่อจัดการลำดับการ build ผ่านการประกาศ dependency
ระหว่างไฟล์ หลักการเดียวกันนี้ใช้แก้ปัญหา "ต้อง build module ก่อนไฟล์ที่ import" ได้พอดี
ทดสอบแล้วว่า Makefile นี้ทำงานถูกต้องบนเครื่องนี้:

```makefile
CXX = g++
CXXFLAGS = -std=c++20 -fmodules-ts -Wall -Wextra

main: main.o numeric.o
	$(CXX) $(CXXFLAGS) numeric.o main.o -o main

# กฎสำคัญ: ไฟล์ที่ import module ต้องขึ้นกับไฟล์ .o ของ module นั้นเสมอ
# เพื่อบังคับให้ make คอมไพล์ module ให้เสร็จก่อนเสมอ ไม่ว่าจะสั่ง make จากลำดับไหน
main.o: main.cpp numeric.o
	$(CXX) $(CXXFLAGS) -c main.cpp -o main.o

numeric.o: numeric.cc
	$(CXX) $(CXXFLAGS) -c numeric.cc -o numeric.o

clean:
	rm -f *.o main
	rm -rf gcm.cache
```

```bash
make
./main
```

ทดสอบจริง `make` เรียกคำสั่งตามลำดับที่ถูกต้องให้อัตโนมัติ (`numeric.cc` ก่อน `main.cpp`
เสมอ เพราะประกาศ dependency ไว้) และรันได้ผลลัพธ์ถูกต้องครบถ้วน — วิธีนี้ปลอดภัยกว่าการจำ
ลำดับคำสั่งด้วยตัวเองมาก และเป็นแนวทางเบื้องต้นก่อนจะขยับไปใช้ build system ที่รองรับ
module โดยเฉพาะอย่าง CMake รุ่นใหม่ (Part 91) เมื่อโปรเจกต์ใหญ่ขึ้น

module คือ**อนาคต**ของการจัดการโค้ด C++ อย่างไม่ต้องสงสัย เพราะแก้ปัญหาเชิงโครงสร้างที่
`#include` มีมาตั้งแต่ยุค C แรกเริ่มได้จริงในระดับภาษา แต่ ณ ปี 2026 มันยังอยู่ในช่วง
"เปลี่ยนผ่าน" ที่ header แบบเดิมยังจำเป็นต้องอยู่คู่กันไปอีกระยะหนึ่ง ผู้เรียนที่เข้าใจทั้ง
สองระบบ (และรู้ข้อจำกัดจริงของแต่ละ toolchain อย่างในบทนี้) จะได้เปรียบกว่าคนที่รู้แค่
ทฤษฎีจาก slide โดยไม่เคยลองคอมไพล์จริง

---

## ข้อผิดพลาดที่พบบ่อย (Common Pitfalls)

1. **ใช้นามสกุล `.cppm` กับ GCC โดยไม่ระบุ `-x c++` แล้วเข้าใจผิดว่าคอมไพล์ผ่าน** — ตามที่
   แสดงใน 76.4 คำสั่งจะจบด้วย exit code 0 (ดูเหมือนสำเร็จ) แต่ไม่มีไฟล์ `.o` ถูกสร้างขึ้นจริง
   เป็นกับดักที่อันตรายเพราะไม่มี error ชัดเจน มีแค่ warning เบาๆ ที่มือใหม่มักมองข้าม
   วิธีป้องกัน: ตรวจสอบว่าไฟล์ output ถูกสร้างขึ้นจริงเสมอหลังคอมไพล์ (`ls -la` หรือเช็คใน
   build script) อย่าเชื่อ exit code เพียงอย่างเดียว

2. **ลืม build module interface ก่อน compile ไฟล์ที่ import มัน** — จะได้ error
   "failed to read compiled module" ตามที่แสดงใน 76.4 กฎสำคัญที่ต่างจาก `#include` โดย
   สิ้นเชิงคือ **ลำดับการคอมไพล์มีผลกับ module** ต้องคอมไพล์ module interface ก่อนเสมอ

3. **ลืมว่า `export module X;` ต้องอยู่บรรทัดแรกสุดของไฟล์** (ก่อน `#include` ใดๆ ด้วยซ้ำ
   ในกรณีทั่วไป) การใส่ comment ธรรมดาไว้ก่อนได้ แต่ preprocessor directive อื่นที่มีผลก่อน
   บรรทัดนี้อาจทำให้เกิดปัญหาได้ ควรทำให้ `export module X;` เป็นสิ่งแรกสุดที่มีความหมายจริง
   ในไฟล์เสมอ

4. **สับสนระหว่าง `export module X;` กับ `module X;`** — ตัวแรกคือ primary interface unit
   (ประกาศ module ใหม่ ต้องมีแค่ 1 ไฟล์ต่อ 1 module) ส่วนตัวหลังคือ implementation unit
   (ให้ implementation เพิ่มเติม มีได้หลายไฟล์) ถ้าเขียนผิดจะได้ผลลัพธ์ที่ต่างกันโดยสิ้นเชิง

5. **คาดหวังว่า `import <ไลบรารีมาตรฐาน>;` (header unit) จะใช้งานได้ทันทีแบบ `#include`**
   ตามที่ทดสอบใน 76.7 บน GCC 13.3.0 + `-fmodules-ts` ยังต้องมีการ precompile เพิ่มเติม
   ไม่ใช่ drop-in replacement ของ `#include` ในทันที ควรทดสอบบน toolchain จริงของตัวเองก่อน
   เชื่อว่าใช้งานได้เสมอ

6. **ไม่ได้เพิ่ม `gcm.cache/` ลงใน `.gitignore`** — ทำให้ไฟล์ binary interface ที่สร้างจาก
   เครื่องของตัวเอง (ซึ่งใช้ข้ามเครื่อง/เวอร์ชัน compiler ไม่ได้อยู่แล้ว) หลุดเข้าไปใน git
   repository โดยไม่ตั้งใจ เพิ่มบรรทัด `gcm.cache/` ในไฟล์ `.gitignore` เสมอเมื่อเริ่มใช้ module

7. **คาดหวัง binary compatibility ข้าม compiler หรือข้ามเวอร์ชัน** — ไฟล์ `.gcm`/BMI ที่
   คอมไพล์ด้วย GCC เวอร์ชันหนึ่ง ใช้กับ Clang หรือแม้แต่ GCC คนละเวอร์ชันไม่ได้เสมอไป ถ้า
   ทีมมีสมาชิกใช้คนละ compiler ต้อง build module ใหม่ในเครื่องของแต่ละคนเสมอ ไม่สามารถแชร์
   ไฟล์ `.gcm` ที่ build ไว้แล้วข้ามเครื่องได้เหมือนแชร์ header ไฟล์ธรรมดา

---

## แบบฝึกหัดท้ายบท

1. สร้าง module ชื่อ `numeric` ที่ export ฟังก์ชัน `clamp_int(int value, int lo, int hi)`
   (บีบค่าให้อยู่ในช่วง `[lo, hi]`) และ `lerp(double a, double b, double t)` (คำนวณค่า
   ระหว่าง `a` กับ `b` ที่สัดส่วน `t`) แล้วเขียนไฟล์ `main.cpp` ที่ `import numeric;` มาใช้งาน
   คอมไพล์และรันให้ผ่านจริงด้วย `-fmodules-ts` (ใช้แต่ fundamental type ล้วนๆ ตามที่แนะนำใน
   76.7 ข้อ 5 เพื่อหลีกเลี่ยงปัญหาการดึง standard library header เข้ามาใน module)

2. ทดลองทำผิดกฎโดยตั้งใจ: สร้างฟังก์ชัน helper ที่ไม่ export ใน module จากข้อ 1 แล้วลอง
   เรียกมันจากไฟล์ `main.cpp` ดูว่า error message ที่ได้ตรงกับที่คาดไว้ตามที่เรียนใน 76.5
   หรือไม่ บันทึก error message ที่ได้จริง

3. เขียน module ที่แยก interface (`.cc` ที่มี `export module`) กับ implementation
   (`.cc` ที่มี `module X;` เฉยๆ) ออกจากกัน คล้ายกับตัวอย่าง `shapes` ใน 76.4 แต่เปลี่ยนเป็น
   ฟังก์ชันคำนวณพื้นที่สี่เหลี่ยมผืนผ้าแทน

4. ลองคอมไพล์ตัวอย่างจากข้อ 1 โดย**ไม่ build module ก่อน** (คอมไพล์ `main.cpp` ก่อน
   `numeric.cc`) แล้วบันทึก error message ที่ได้ เทียบกับที่แสดงใน 76.4

5. เขียนอธิบายด้วยคำพูดตัวเอง (เป็นคอมเมนต์ในไฟล์หรือเอกสารสั้นๆ) ว่าทำไมการแจกจ่ายไลบรารี
   เป็น pre-compiled module (`.gcm`) ให้ทีมอื่นใช้ข้าม compiler จึงทำได้ยากกว่าการแจกจ่ายเป็น
   header file ธรรมดา

6. (ท้าทาย) ค้นคว้าเพิ่มเติม (จากเอกสารของ GCC/Clang ที่ติดตั้งบนเครื่องของตัวเอง หรือ
   release note ล่าสุด) ว่าเวอร์ชันของ compiler ที่ผู้เรียนใช้อยู่รองรับ `import std;`
   (standard library module) หรือยัง แล้วถ้ารองรับ ลองทดสอบเขียนโปรแกรมง่ายๆ ที่ใช้มันดู
   ถ้าไม่รองรับ ให้บันทึก error message ที่ได้ไว้เป็นหลักฐาน

### แนวทางเฉลยข้อ 1

```cpp
// numeric.cc
export module numeric;

export int clamp_int(int value, int lo, int hi) {
    if (value < lo) return lo;
    if (value > hi) return hi;
    return value;
}

export double lerp(double a, double b, double t) {
    return a + (b - a) * t;
}
```

```cpp
// main.cpp
import numeric;
#include <iostream>

int main() {
    std::cout << clamp_int(15, 0, 10) << "\n";   // คาดหวัง: 10 (เกินขอบบน ถูกบีบลงมา)
    std::cout << lerp(0.0, 10.0, 0.25) << "\n";  // คาดหวัง: 2.5 (25% ของระยะทางจาก 0 ถึง 10)
}
```

```bash
g++ -std=c++20 -fmodules-ts -c numeric.cc -o numeric.o
g++ -std=c++20 -fmodules-ts -c main.cpp -o main.o
g++ -std=c++20 -fmodules-ts numeric.o main.o -o main
./main
```

ทดสอบคอมไพล์และรันจริงบนเครื่องที่ใช้เขียนบทเรียนนี้ ได้ผลลัพธ์ตามที่คาด:

```
10
2.5
```

(หมายเหตุสำคัญ: ตัวอย่างนี้จงใจใช้แค่ `int`/`double` ไม่ดึง standard library header ใดๆ
เข้ามาใน module เลย เพราะตามที่พิสูจน์ไว้ใน 76.7 ข้อ 5 การดึง header อย่าง `<string>` เข้ามา
ใน module แล้วให้ไฟล์อื่นที่ `import` module นั้น `#include` header ตัวเดียวกันซ้ำอีกที
จะทำให้เกิด conflict รุนแรงบน `-fmodules-ts` ของ GCC 13.3.0 — ผู้เรียนที่อยากลองใช้
`std::string` ใน module จริงๆ ควรทดสอบบน toolchain ของตัวเองก่อน และเตรียมใจว่าอาจเจอ
ปัญหาแบบเดียวกัน)

### แนวทางเฉลยข้อ 4

```bash
# ลืม build module ก่อน — คอมไพล์ main.cpp ก่อน numeric.cc
g++ -std=c++20 -fmodules-ts -c main.cpp -o main.o
```

ผลลัพธ์ที่ได้จริง (error message มีรูปแบบเดียวกับที่แสดงใน 76.4 แต่เปลี่ยนชื่อ module):

```
In module imported at main.cpp:1:1:
numeric: error: failed to read compiled module: No such file or directory
numeric: note: compiled module file is 'gcm.cache/numeric.gcm'
numeric: note: imports must be built before being imported
numeric: fatal error: returning to the gate for a mechanical issue
compilation terminated.
```

นี่คือหลักฐานยืนยันกฎสำคัญที่สุดของการทำงานกับ module: **ต้อง build module interface ให้
เสร็จก่อนเสมอ ก่อนคอมไพล์ไฟล์ใดๆ ที่ `import` มัน** ไม่มีข้อยกเว้น และเป็นความแตกต่างที่
ชัดเจนที่สุดจาก `#include` ที่ไม่สนใจลำดับไฟล์เลย

---

## สรุปท้ายบท

ใน Part นี้เราได้:

- ทบทวนและเจาะลึกปัญหาเชิงโครงสร้าง 3 ข้อของระบบ `#include`/header แบบเดิม: คอมไพล์ช้า
  เพราะ parse ซ้ำ, ไม่มี encapsulation จริง, และ macro รั่วไหลข้ามไฟล์
- เข้าใจว่า C++20 module แก้ปัญหาทั้งสามข้อนี้อย่างไรในระดับภาษา ผ่านกลไก compiled binary
  interface และการบังคับ export
- เขียนและคอมไพล์ module จริงบนเครื่อง (GCC 13.3.0 + `-fmodules-ts`) ได้สำเร็จหลายตัวอย่าง
  ทั้งแบบพื้นฐาน แบบแยก interface/implementation และแบบพิสูจน์ encapsulation จริงด้วยการ
  ทำให้ compile error เมื่อพยายามเรียกสิ่งที่ไม่ได้ export
- รู้ข้อจำกัดเฉพาะของ toolchain นี้อย่างตรงไปตรงมา ทั้งเรื่องนามสกุลไฟล์ `.cppm`,
  การ import header unit ที่ยังไม่สะดวก, และการไม่มี dependency scanning อัตโนมัติ
- เข้าใจสถานะการรองรับ module จริงในอุตสาหกรรมปี 2026 ว่าเหตุใด header แบบเดิมจึงยังคง
  เป็นวิธีหลักในโปรเจกต์ real-world จำนวนมาก แม้ module จะเหนือกว่าในทางทฤษฎี
- ได้แนวทางตัดสินใจเชิงปฏิบัติว่าเมื่อไหร่ควรทดลองใช้ module และเมื่อไหร่ควรยึด header เดิม

module คือทิศทางระยะยาวของภาษา C++ อย่างชัดเจน แต่การเปลี่ยนผ่านยังต้องใช้เวลาอีกพักใหญ่
ใน **Part 77** เราจะขยับไปดูภาพรวมของฟีเจอร์ใหม่ใน **C++23** ทั้ง `std::expected`,
`std::print`, deducing this และอื่นๆ พร้อมทดสอบจริงว่าฟีเจอร์ไหนใช้งานได้แล้วบนเครื่องนี้

**ต่อไป:** [Part 77 — ภาพรวมฟีเจอร์ใหม่ C++23](./part-077-cpp23-overview.md)
