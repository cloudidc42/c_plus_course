# Part 17: Modular Programming (Header/Linkage) (Step 129–136)

> Module B — C ระดับกลาง & โครงสร้างข้อมูล/อัลกอริทึม | Part 17 จาก 125
> Step ที่ครอบคลุมใน Part นี้: Step 129–136
> Part ก่อนหน้า: [Part 16 — Error Handling ใน C](./part-016-error-handling.md) | Part ถัดไป: [Part 18 — Makefile และ Build Automation](./part-018-makefile-build.md)

## เป้าหมายการเรียนรู้ (Learning Objectives)

หลังจบ Part นี้ ผู้เรียนจะสามารถ:

1. อธิบายได้ว่า **Compilation Unit (Translation Unit)** คืออะไร และทำไมโปรแกรมขนาดใหญ่ต้อง
   แยกเป็นหลายไฟล์แทนที่จะเขียนทุกอย่างไว้ใน `main.c` ไฟล์เดียว
2. แยกโค้ดออกเป็นไฟล์ header (`.h`) และไฟล์ source (`.c`) ได้อย่างถูกต้องตามหลักปฏิบัติมาตรฐาน
3. เข้าใจความแตกต่างระหว่าง **Declaration** (ประกาศ) กับ **Definition** (นิยาม) อย่างชัดเจน
4. ใช้ `extern` เพื่อแชร์ฟังก์ชันและตัวแปรข้ามไฟล์ (**External Linkage**) ได้อย่างถูกต้อง
5. ใช้ `static` ที่ scope ระดับไฟล์เพื่อซ่อนรายละเอียดภายในของ module (**Internal Linkage**) ได้
6. วินิจฉัยและแก้ปัญหา **Multiple Definition Error** ที่เกิดจากการออกแบบ header ผิดพลาดได้ด้วยตัวเอง
7. ออกแบบ **Public API** ของ module ผ่าน header file ที่สะอาด อ่านง่าย และซ่อนรายละเอียดที่ไม่จำเป็น
8. แยกโปรแกรมจริงเป็น module `math_utils.h/.c` และ `main.c` แล้วคอมไพล์แยกกันก่อนนำมา link
   รวมเป็นโปรแกรมเดียว

---

## 17.1 ทำไมต้องแยกโค้ดเป็นหลายไฟล์ และ Compilation Unit คืออะไร (Step 129)

จนถึง Part นี้ โปรแกรมตัวอย่างทุกตัวในหลักสูตรถูกเขียนไว้ใน `.c` ไฟล์เดียว ซึ่งใช้ได้ดีกับ
โปรแกรมขนาดเล็ก แต่พอโปรเจกต์เริ่มโตขึ้น — มีฟังก์ชันหลายสิบ หลายร้อยฟังก์ชัน มีหลายคนช่วยกัน
เขียน — การเก็บทุกอย่างไว้ในไฟล์เดียวจะกลายเป็นปัญหาใหญ่:

- **คอมไพล์ช้า**: ทุกครั้งที่แก้โค้ดแม้เพียงบรรทัดเดียว compiler ต้องอ่านและแปลไฟล์ทั้งหมดใหม่
  ทั้งไฟล์ ถ้าไฟล์มีหลายหมื่นบรรทัด เวลาคอมไพล์จะยาวขึ้นเรื่อยๆ
- **แก้ไขร่วมกันยาก**: ถ้าโปรแกรมเมอร์สองคนแก้ไฟล์เดียวกันพร้อมกัน โอกาสเกิด conflict ใน Git สูงมาก
- **อ่านและดูแลรักษายาก**: หาโค้ดที่ต้องการแก้ไม่เจอง่ายๆ ในไฟล์ขนาดหลายพันบรรทัด
- **ใช้ซ้ำไม่ได้**: ถ้าฟังก์ชันคณิตศาสตร์ที่เขียนไว้ถูกฝังอยู่ใน `main.c` ของโปรเจกต์ A จะดึงไป
  ใช้ในโปรเจกต์ B ไม่ได้เลยถ้าไม่ copy-paste (ซึ่งเป็นนิสัยที่ไม่ดี)

ทางแก้คือ **แบ่งโปรแกรมออกเป็น Module** — แต่ละ module รับผิดชอบหน้าที่เฉพาะทางของตัวเอง
(เช่น module จัดการคณิตศาสตร์, module จัดการ string, module จัดการไฟล์) แล้วนำมาประกอบรวมกัน
เป็นโปรแกรมสมบูรณ์อีกที

### Compilation Unit (Translation Unit) คืออะไร

เมื่อเราสั่ง `gcc -c some_file.c` compiler จะแปลไฟล์ `.c` ไฟล์นั้น **พร้อมกับเนื้อหาทั้งหมดที่ถูก
`#include` เข้ามา** ให้กลายเป็น object file (`.o`) หนึ่งไฟล์ หน่วยที่ compiler มองเห็นและแปล
ในแต่ละครั้งนี้เรียกว่า **Compilation Unit** หรือ **Translation Unit**

```
math_utils.c  +  เนื้อหาที่ #include เข้ามา (math_utils.h, stdio.h, ...)
        │
        ▼   [gcc -c]
   math_utils.o   ← Compilation Unit 1 ที่แปลเสร็จแล้ว

main.c  +  เนื้อหาที่ #include เข้ามา (math_utils.h, stdio.h, ...)
        │
        ▼   [gcc -c]
   main.o         ← Compilation Unit 2 ที่แปลเสร็จแล้ว
```

ประเด็นสำคัญคือ **แต่ละ Compilation Unit ถูกแปลแยกกันโดยสิ้นเชิง** — ตอนที่ compiler แปล
`math_utils.c` มันไม่รู้เลยว่า `main.c` มีเนื้อหาอะไร และในทางกลับกัน สิ่งที่เชื่อมสอง
Compilation Unit เข้าด้วยกันคือขั้นตอน **Linking** (ที่เรียนไปแล้วใน Part 1) ซึ่งจะนำ `.o` ไฟล์
ทั้งหมดมารวมกันเป็น Executable เดียว

นี่คือเหตุผลที่ทำให้เราต้องมี **Header File** — เพราะเมื่อ `main.c` ต้องการเรียกใช้ฟังก์ชันที่อยู่ใน
`math_utils.c` (คนละ Compilation Unit) compiler ของ `main.c` ต้องรู้ **prototype** ของฟังก์ชันนั้น
ล่วงหน้าก่อนจะยอมให้เรียกใช้ได้ (มิฉะนั้นจะไม่รู้ว่าฟังก์ชันรับ parameter อะไร คืนค่าอะไร)

---

## 17.2 แยกโค้ดเป็น Header (.h) และ Source (.c) (Step 130)

รูปแบบมาตรฐานของการสร้าง module หนึ่งตัวใน C ประกอบด้วย 2 ไฟล์เสมอ:

| ไฟล์ | หน้าที่ |
|---|---|
| `module.h` | **Public API** — ประกาศ (declare) ว่า module นี้ "มีอะไรให้ใช้บ้าง" โดยไม่มีการ implement จริง |
| `module.c` | **Implementation** — เขียนโค้ดจริงของทุกฟังก์ชันที่ประกาศไว้ใน `.h` |

มาลองสร้าง module ชื่อ `math_utils` ที่รวมฟังก์ชันคณิตศาสตร์ที่ใช้บ่อยไว้ด้วยกัน

**`math_utils.h`**

```c
#ifndef MATH_UTILS_H
#define MATH_UTILS_H

#include <stddef.h> /* size_t */

/* ---------- Public API ของ module math_utils ---------- */

int  mu_add(int a, int b);
int  mu_subtract(int a, int b);
long mu_power(int base, unsigned int exponent);
int  mu_gcd(int a, int b);
int  mu_is_prime(int n);
double mu_average(const int *values, size_t count);

#endif /* MATH_UTILS_H */
```

**`math_utils.c`**

```c
#include "math_utils.h"

int mu_add(int a, int b) {
    return a + b;
}

int mu_subtract(int a, int b) {
    return a - b;
}

long mu_power(int base, unsigned int exponent) {
    long result = 1;
    for (unsigned int i = 0; i < exponent; i++) {
        result *= base;
    }
    return result;
}
/* ... ฟังก์ชันที่เหลือจะเติมเต็มในหัวข้อ 17.8 ... */
```

สังเกตจุดสำคัญ 2 จุด:

1. **`math_utils.c` ต้อง `#include "math_utils.h"` ของตัวเองเสมอ** — ฟังดูแปลกเพราะเหมือน
   ไฟล์รู้จักตัวเองอยู่แล้ว แต่การทำแบบนี้มีประโยชน์มาก: ถ้า `math_utils.c` implement ฟังก์ชัน
   ไม่ตรงกับ prototype ที่ประกาศไว้ใน header (เช่น สลับลำดับ parameter, พิมพ์ชนิดข้อมูลผิด)
   compiler จะจับ error ได้ทันทีตอนคอมไพล์ `math_utils.c` เอง แทนที่จะไปเจอตอน link กับ
   `main.c` ซึ่งจะบอก error ที่เข้าใจยากกว่ามาก
2. **การ `#include` ด้วยเครื่องหมายคำพูด `"..."` กับวงเล็บมุม `<...>` ต่างกัน**:
   - `#include <stdio.h>` — บอก preprocessor ให้ค้นหาใน system include path เท่านั้น
     (ใช้กับ header มาตรฐานของภาษาหรือไลบรารีที่ติดตั้งในระบบ)
   - `#include "math_utils.h"` — บอกให้ค้นหาในโฟลเดอร์เดียวกับไฟล์ปัจจุบันก่อน แล้วค่อยไปหา
     ใน system path (ใช้กับ header ที่เราเขียนเองในโปรเจกต์)

### คอมไพล์แยกกันแล้วค่อย Link

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 -c math_utils.c -o math_utils.o
gcc -Wall -Wextra -Wpedantic -std=c17 -c main.c -o main.o
gcc math_utils.o main.o -o program
./program
```

`-c` บอกให้ gcc หยุดแค่ขั้นตอน compile+assemble (สร้าง `.o`) โดยยังไม่ link คำสั่งสุดท้าย
ที่ไม่มี `-c` คือขั้นตอน link ที่รวม `.o` ทั้งสองไฟล์เข้าด้วยกัน ข้อดีของการแยกคอมไพล์แบบนี้
คือถ้าเราแก้แค่ `main.c` ก็ไม่จำเป็นต้องคอมไพล์ `math_utils.c` ใหม่เลย (จะเห็นประโยชน์เต็มๆ
เมื่อใช้ Makefile ใน Part 18)

---

## 17.3 ออกแบบ Public API ผ่าน Header ที่ดี (Step 131)

Header file คือ **สัญญา (Contract)** ระหว่าง module กับโค้ดที่มาเรียกใช้ การออกแบบ header
ที่ดีมีหลักการสำคัญดังนี้:

### 1. Header Guard เสมอ (ทบทวนจาก Part 14)

```c
#ifndef MATH_UTILS_H
#define MATH_UTILS_H
/* ... เนื้อหา ... */
#endif
```

ป้องกันไม่ให้เนื้อหาถูกแปะซ้ำเมื่อ header เดียวกันถูก `#include` มากกว่าหนึ่งครั้งใน
Compilation Unit เดียว (เช่น A.h include B.h และ C.h ที่ทั้งคู่ include D.h อีกที) ถ้าไม่มี
Header Guard และ header มีการประกาศ `struct` หรือ `typedef` จะเกิด error "redefinition"

### 2. Header ควรมีแค่สิ่งที่ "จำเป็นต้องให้คนนอกรู้" เท่านั้น

Header ที่ดีเปิดเผยเฉพาะ:

- Function prototype ของฟังก์ชันที่ต้องการให้ไฟล์อื่นเรียกใช้ได้
- Struct/typedef/enum ที่เป็นส่วนหนึ่งของ interface (จำเป็นต้องใช้ตอนเรียกฟังก์ชัน)
- Macro constant ที่เกี่ยวข้องกับการใช้งาน module (เช่น `MU_MAX_SIZE`)
- `extern` declaration ของตัวแปร global ที่ตั้งใจแชร์จริงๆ (หัวข้อ 17.4)

Header ที่ดี **ไม่ควร** มี:

- ฟังก์ชันช่วยภายใน (internal helper) ที่ไฟล์อื่นไม่ต้องรู้จัก
- ตัวแปร global ที่ใช้เฉพาะภายใน module
- Implementation ของฟังก์ชัน (ยกเว้นกรณีพิเศษ เช่น `static inline` ฟังก์ชันสั้นๆ ที่จะกล่าวถึง
  ใน Part หลังๆ เมื่อพูดถึง C++ `inline`)

### 3. ตั้งชื่อฟังก์ชันด้วย Prefix ของ module

สังเกตว่าทุกฟังก์ชันใน `math_utils.h` ขึ้นต้นด้วย `mu_` (ย่อจาก **M**ath **U**tils) — นี่เป็น
ธรรมเนียมปฏิบัติที่สำคัญมากในภาษา C เพราะ **C ไม่มี namespace** เหมือน C++ (จะเรียนใน
Module D) ถ้าสองไฟล์ header ต่างประกาศฟังก์ชันชื่อ `add` เหมือนกัน แล้วถูก include เข้ามา
พร้อมกัน จะเกิด error "conflicting types" หรือ "redefinition" ทันที การใส่ prefix ของ module
ไว้เสมอ (`mu_`, `str_`, `list_`, ฯลฯ) เป็นวิธีจำลอง namespace แบบง่ายๆ ที่ใช้กันทั่วไปในโค้ด C

### 4. เขียน Comment อธิบายพฤติกรรมที่ "ไม่เห็นจากชื่อฟังก์ชันอย่างเดียว"

```c
/* คืนค่าเฉลี่ยของ array `values` จำนวน `count` ตัว
 * ถ้า count == 0 จะคืนค่า 0.0 (ไม่ throw หรือ crash) */
double mu_average(const int *values, size_t count);
```

คนที่มาเรียกใช้ module ควรอ่าน header อย่างเดียวแล้วรู้ว่าจะใช้งานฟังก์ชันอย่างไร โดยไม่ต้อง
ไปเปิดดู `.c` เลย — นี่คือหัวใจของการออกแบบ API ที่ดี

---

## 17.4 extern: การแชร์ฟังก์ชันและตัวแปรข้ามไฟล์ (Step 132)

### Declaration vs Definition

นี่คือแนวคิดที่สำคัญที่สุดของ Part นี้ และเป็นสิ่งที่มือใหม่สับสนบ่อยที่สุด:

| | Declaration (การประกาศ) | Definition (การนิยาม) |
|---|---|---|
| ความหมาย | บอก compiler ว่า "สิ่งนี้มีอยู่ ชื่อนี้ ชนิดนี้" โดยไม่จองพื้นที่หน่วยความจำ | สร้างสิ่งนั้นขึ้นจริง จองพื้นที่หน่วยความจำจริง |
| ฟังก์ชัน | `int mu_add(int a, int b);` (prototype จบด้วย `;`) | `int mu_add(int a, int b) { return a + b; }` (มี body) |
| ตัวแปร | `extern int counter;` | `int counter;` หรือ `int counter = 0;` |
| ทำได้กี่ครั้ง | ประกาศซ้ำได้หลายครั้ง หลายไฟล์ | นิยามได้ **เพียงครั้งเดียว** ในทั้งโปรแกรม (One Definition Rule) |

Header file ควรมีแต่ **declaration** เท่านั้น ส่วน **definition** อยู่ใน `.c` ไฟล์เพียงที่เดียว
นี่คือเหตุผลที่ฟังก์ชันใน header จบด้วย `;` โดยไม่มี body

### คำสำคัญ `extern`

คำว่า `extern` แปลตรงตัวว่า "external" — บอก compiler ว่า "ตัวแปร/ฟังก์ชันนี้ถูกนิยามอยู่
**ที่อื่น**" สำหรับฟังก์ชัน เราแทบไม่ต้องเขียน `extern` เลยเพราะฟังก์ชันเป็น external linkage
เป็นค่า default อยู่แล้ว แต่สำหรับ **ตัวแปร global** เราต้องเขียน `extern` ให้ชัดเจนเมื่อต้องการ
ประกาศในไฟล์อื่นที่ไม่ใช่จุดที่นิยามจริง

ตัวอย่าง: สมมติเราต้องการนับจำนวนครั้งที่ฟังก์ชันใน `math_utils` ถูกเรียกทั้งหมด โดยใช้ตัวแปร
global ที่ไฟล์อื่นสามารถอ่านค่าได้ (ผ่านฟังก์ชัน getter):

**`math_utils.h`** (เพิ่มเข้าไปจากเดิม)

```c
/* ตัวแปร global ที่ให้ไฟล์อื่นมองเห็นได้ (external linkage)
 * ใน header เราแค่ "ประกาศ" (declare) ด้วย extern เท่านั้น
 * ตัวจริง (definition) อยู่ใน math_utils.c เพียงจุดเดียว */
extern long mu_call_count;

void mu_reset_call_count(void);
long mu_get_call_count(void);
```

**`math_utils.c`** (เพิ่มเข้าไปจากเดิม)

```c
/* ---------- Definition ของตัวแปร global (มีจุดเดียวในทั้งโปรแกรม) ---------- */
long mu_call_count = 0;

void mu_reset_call_count(void) {
    mu_call_count = 0;
}

long mu_get_call_count(void) {
    return mu_call_count;
}
```

ทุกไฟล์ที่ `#include "math_utils.h"` จะเห็น `extern long mu_call_count;` ซึ่งเป็นแค่การประกาศ
ว่า "มีตัวแปรชื่อนี้อยู่ที่ไหนสักแห่งในโปรแกรม ชนิด `long`" ตอน link linker จะไปค้นหาว่า
definition จริงอยู่ที่ไฟล์ object ไหน (ในที่นี้คือ `math_utils.o`) แล้วเชื่อมทุกการอ้างอิงให้ชี้ไปที่
หน่วยความจำเดียวกัน

> **แนวปฏิบัติที่ดี**: ถึง `extern` ตัวแปร global จะทำได้ แต่ในโค้ดโปรดักชันจริง มักเลี่ยงการแชร์
> ตัวแปร global ตรงๆ แบบนี้ เพราะทำให้ debug ยาก (ใครก็แก้ค่าได้จากทุกที่) แนวทางที่ดีกว่าคือ
> ทำตัวแปรเป็น `static` (หัวข้อถัดไป) ซ่อนไว้ในไฟล์เดียว แล้วเปิดแค่ฟังก์ชัน getter/setter
> ให้ไฟล์อื่นเรียกแทน (เหมือนที่ทำกับ `mu_get_call_count()` ด้านบน)

---

## 17.5 static ที่ scope ระดับไฟล์: Internal Linkage (Step 133)

เราเคยเรียน `static` ใน context ของตัวแปร local ภายในฟังก์ชัน (ทำให้ค่าคงอยู่ข้ามการเรียก
ฟังก์ชัน) แต่ `static` ยังมีความหมายที่ **สำคัญกว่ามาก** เมื่อใช้กับฟังก์ชันหรือตัวแปร **ที่ scope
ระดับไฟล์** (global scope, นอกฟังก์ชันใดๆ)

เมื่อฟังก์ชันหรือตัวแปร global ถูกประกาศด้วย `static`:

- มันจะมองเห็นได้ **เฉพาะภายใน Compilation Unit (ไฟล์ `.c`) เดียวกันเท่านั้น**
- ไฟล์ `.c` อื่นจะเรียกใช้ไม่ได้เลย แม้จะพยายาม `extern` ประกาศชื่อเดียวกันก็ตาม
- เรียกพฤติกรรมนี้ว่า **Internal Linkage**

ตัวอย่างจาก `math_utils.c`:

```c
#include "math_utils.h"

/* static: internal linkage — ฟังก์ชันช่วยเหล่านี้เป็นรายละเอียดภายในของ module
 * ไฟล์อื่นมองไม่เห็นและเรียกใช้ไม่ได้ แม้จะไม่มี header guard ป้องกันก็ตาม */
static void mu_track_call(void) {
    mu_call_count++;
}

static int mu_abs_int(int x) {
    return (x < 0) ? -x : x;
}

int mu_gcd(int a, int b) {
    mu_track_call();
    a = mu_abs_int(a);
    b = mu_abs_int(b);
    while (b != 0) {
        int temp = b;
        b = a % b;
        a = temp;
    }
    return a;
}
```

`mu_track_call` และ `mu_abs_int` เป็นแค่ฟังก์ชันช่วยภายในที่ `math_utils.c` ใช้เอง ไม่มีเหตุผล
ที่ `main.c` หรือไฟล์อื่นต้องรู้จักหรือเรียกใช้ตรงๆ การประกาศเป็น `static` มีประโยชน์ 3 อย่าง:

1. **ซ่อนรายละเอียดการ implement** — คนใช้ module เห็นแค่ public API ใน header ทำให้
   module เปลี่ยนวิธี implement ภายในได้อย่างอิสระโดยไม่กระทบโค้ดที่เรียกใช้
2. **ป้องกันชื่อชนกัน (name collision)** — ถ้าอีกไฟล์หนึ่งในโปรเจกต์ก็มีฟังก์ชันช่วยชื่อ
   `mu_abs_int` เหมือนกัน (หรือสั้นกว่านั้นเช่น `abs_int`) จะไม่มีปัญหาเลยเพราะแต่ละไฟล์เห็น
   แค่ของตัวเอง
3. **ช่วยให้ compiler optimize ได้ดีขึ้น** — เพราะ compiler รู้แน่นอนว่าฟังก์ชันนี้ถูกเรียกจากที่ไหน
   บ้างภายในไฟล์เดียวกัน (ไม่ต้องกันไว้เผื่อไฟล์อื่นเรียก) จึงสามารถ inline หรือปรับแต่งโค้ดให้
   เร็วขึ้นได้อย่างเต็มที่

> **กฎทองของ Part นี้**: ฟังก์ชันหรือตัวแปร global ใดที่ไม่ได้ถูกประกาศไว้ใน header (ไม่ได้
> ตั้งใจเป็นส่วนหนึ่งของ public API) ควรใส่ `static` เสมอ นี่คือแนวปฏิบัติที่โค้ด C คุณภาพสูง
> ในโลกจริง (เช่น Redis, SQLite) ใช้กันอย่างเคร่งครัด

---

## 17.6 สรุป Linkage ทั้ง 3 แบบในภาษา C (Step 134)

ภาษา C มีแนวคิดเรื่อง **Linkage** (การเชื่อมโยงชื่อข้าม Compilation Unit) อยู่ 3 แบบ:

| Linkage | ความหมาย | ตัวอย่าง |
|---|---|---|
| **External Linkage** | ชื่อเดียวกันในไฟล์ต่างๆ อ้างถึงสิ่งเดียวกัน มองเห็นได้ข้ามไฟล์ | ฟังก์ชันธรรมดา, ตัวแปร global ที่ไม่ใส่ `static` |
| **Internal Linkage** | มองเห็นได้เฉพาะภายใน Compilation Unit เดียว | ตัวแปร/ฟังก์ชัน global ที่ใส่ `static` |
| **No Linkage** | แต่ละที่ที่ประกาศเป็นคนละตัวกันอิสระ ไม่เกี่ยวข้องกันเลย | ตัวแปร local ภายในฟังก์ชัน (รวมถึง local `static`) |

แผนภาพช่วยจำ:

```
┌─────────────────────────────────────────────────────────────┐
│  ระดับไฟล์ (File Scope / Global Scope)                        │
│                                                               │
│   int    g_shared    = 0;   →  External Linkage (default)   │
│   static int g_hidden = 0;  →  Internal Linkage              │
│                                                               │
│   int shared_func(void) { ... }   →  External Linkage        │
│   static int hidden_func(void) { ... }  →  Internal Linkage  │
│                                                               │
│   void some_function(void) {                                │
│       int local_var = 0;        →  No Linkage (แยกกันทุก call)│
│       static int persist = 0;   →  No Linkage (แต่ค่าคงอยู่)  │
│   }                                                          │
└─────────────────────────────────────────────────────────────┘
```

ข้อสังเกตที่มือใหม่มักสับสน: `static int persist = 0;` ที่อยู่**ภายในฟังก์ชัน** มีพฤติกรรมเรื่อง
**อายุ (storage duration)** เหมือนตัวแปร global (ค่าคงอยู่ตลอดโปรแกรม ไม่ถูกสร้างใหม่ทุกครั้งที่
เรียกฟังก์ชัน) แต่เรื่อง **linkage** มันคือ **No Linkage** เพราะชื่อ `persist` มองเห็นได้แค่ภายใน
ฟังก์ชันนั้นเท่านั้น ไม่มีไฟล์ไหนอ้างอิงชื่อนี้ข้ามมาได้ — **Storage Duration** กับ **Linkage** เป็น
คนละแนวคิดกัน แม้จะใช้ keyword เดียวกันคือ `static` ก็ตาม

---

## 17.7 ปัญหา Multiple Definition และวิธีป้องกัน (Step 135)

### กรณีที่ 1: ตัวแปร global ถูก define ซ้ำในหลายไฟล์

ลองดูตัวอย่างโค้ดที่ผิด — สมมติมีสองไฟล์ต่างประกาศตัวแปร global ชื่อเดียวกันโดยไม่ใช้ `extern`:

**`a.c`**
```c
int counter;   /* ตั้งใจจะเป็น definition */

void set_counter(int v) {
    counter = v;
}
```

**`b.c`**
```c
#include <stdio.h>

int counter;   /* ก็เป็น definition อีกตัว! ผิดพลาด */

void set_counter(int v);

int main(void) {
    set_counter(5);
    printf("%d\n", counter);
    return 0;
}
```

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 -c a.c -o a.o
gcc -Wall -Wextra -Wpedantic -std=c17 -c b.c -o b.o
gcc a.o b.o -o dup
```

ทั้งสองไฟล์คอมไพล์ผ่านโดยไม่มี warning เลย (เพราะแต่ละไฟล์แยกกันดูก็ถูกต้องตามหลักไวยากรณ์)
แต่จะเกิด error ตอนขั้นตอน **link**:

```
/usr/bin/ld: b.o:(.bss+0x0): multiple definition of `counter'; a.o:(.bss+0x0): first defined here
collect2: error: ld returned 1 exit status
```

นี่คือตัวอย่างชัดเจนของ **Undefined Reference / Multiple Definition Error** ที่เกิดในขั้นตอน
Linking ตามตารางที่เคยแสดงใน Part 1 — ทั้งสองไฟล์นิยาม `counter` เป็นของตัวเอง เมื่อ linker
พยายามรวมทั้งสองเข้าด้วยกันในโปรแกรมเดียว มันไม่รู้ว่าจะใช้ตัวไหน จึงฟ้อง error

**วิธีแก้ที่ถูกต้อง**: นิยามตัวแปรใน `.c` ไฟล์เดียวเท่านั้น แล้วประกาศด้วย `extern` ในไฟล์อื่น
(หรือใน header ที่ทุกไฟล์ include ร่วมกัน)

```c
/* a.c — นิยามจริงมีจุดเดียว */
int counter;

void set_counter(int v) {
    counter = v;
}
```

```c
/* b.c */
#include <stdio.h>

extern int counter;   /* แค่ declaration ไม่ใช่ definition */

void set_counter(int v);

int main(void) {
    set_counter(5);
    printf("%d\n", counter);
    return 0;
}
```

### กรณีที่ 2: ใส่ implementation ของฟังก์ชันไว้ใน Header โดยตรง

นี่คือข้อผิดพลาดที่มือใหม่ทำบ่อยมาก — เขียน body ของฟังก์ชันไว้ใน `.h` แทนที่จะไว้แค่ prototype:

**`util.h`** (ผิด!)
```c
#ifndef UTIL_H
#define UTIL_H

int square(int x) { return x * x; }   /* มี body อยู่ใน header! */

#endif
```

ถ้า header นี้ถูก `#include` เข้าไปในมากกว่าหนึ่ง Compilation Unit ที่ถูกนำมา link รวมกัน
(เช่น `a.c` และ `main.c` ต่างก็ `#include "util.h"`) — แม้ Header Guard จะป้องกันการแปะซ้ำ
**ภายในไฟล์เดียว** ได้ แต่ Header Guard **ป้องกันข้ามไฟล์ไม่ได้เลย** เพราะแต่ละ Compilation
Unit ถูกแปลแยกกัน (ตามที่อธิบายในหัวข้อ 17.1) ผลคือฟังก์ชัน `square` จะถูกนิยามซ้ำใน
`a.o` และ `main.o` ทั้งคู่:

```
/usr/bin/ld: main.o: in function `square':
main.c:(.text+0x0): multiple definition of `square'; a.o:a.c:(.text+0x0): first defined here
collect2: error: ld returned 1 exit status
```

**วิธีแก้**: header เก็บแค่ prototype (`int square(int x);`) ส่วน body ไปอยู่ใน `.c` ไฟล์เพียง
ไฟล์เดียว (เช่น `util.c`) เท่านั้น ตามหลักการที่อธิบายไปตลอด Part นี้

> **หมายเหตุ**: ในภาษา C ยุคหลัง (และโดยเฉพาะ C++) มีวิธีใส่ฟังก์ชันไว้ใน header ได้อย่าง
> ปลอดภัยด้วยการเติม keyword `static inline` หน้าฟังก์ชัน (`static inline int square(int x) {...}`)
> ซึ่งจะทำให้แต่ละ Compilation Unit ได้สำเนาของตัวเองแบบ internal linkage ไม่ชนกัน — เทคนิคนี้
> มักใช้กับฟังก์ชันสั้นๆ ที่อยากให้ compiler inline ได้ในทุกไฟล์ที่เรียก แต่สำหรับ Part นี้ให้ยึด
> หลักการพื้นฐานคือ "header = ประกาศ, .c = implement" ไปก่อน เพื่อให้เข้าใจรากฐานที่แท้จริง

### เช็คลิสต์ป้องกัน Multiple Definition

1. Header ควรมีแต่ prototype ของฟังก์ชัน (`;` ไม่มี body) ยกเว้นกรณีพิเศษ `static inline`
2. ตัวแปร global ที่แชร์ข้ามไฟล์ ต้องประกาศด้วย `extern` ใน header และนิยามจริงใน `.c`
   ไฟล์เดียวเท่านั้น
3. Header ต้องมี Header Guard เสมอ (ป้องกันปัญหาภายในไฟล์เดียว แต่ไม่ใช่ตัวแก้ปัญหา
   multiple-definition ข้ามไฟล์)
4. ฟังก์ชัน/ตัวแปรที่ไม่ต้องการให้ไฟล์อื่นเห็น ให้ใส่ `static` เสมอ

---

## 17.8 ตัวอย่างจริง: math_utils Module ฉบับสมบูรณ์ (Step 136)

มาประกอบทุกสิ่งที่เรียนมาใน Part นี้เป็นโปรเจกต์เล็กๆ ที่สมบูรณ์ ประกอบด้วย 3 ไฟล์:

### โครงสร้างโปรเจกต์

```
part17_project/
├── math_utils.h
├── math_utils.c
└── main.c
```

### `math_utils.h`

```c
#ifndef MATH_UTILS_H
#define MATH_UTILS_H

#include <stddef.h> /* size_t */

/* ============================================================
 * math_utils — module รวมฟังก์ชันคณิตศาสตร์ที่ใช้บ่อย
 * Public API: ฟังก์ชันและตัวแปรทั้งหมดในไฟล์นี้คือสัญญาที่
 * ไฟล์อื่นในโปรเจกต์เรียกใช้ได้อย่างปลอดภัย
 * ============================================================ */

int  mu_add(int a, int b);
int  mu_subtract(int a, int b);
long mu_power(int base, unsigned int exponent);
int  mu_gcd(int a, int b);
int  mu_is_prime(int n);

/* คืนค่าเฉลี่ยของ array `values` จำนวน `count` ตัว
 * ถ้า count == 0 จะคืนค่า 0.0 (ไม่ crash) */
double mu_average(const int *values, size_t count);

/* ตัวนับจำนวนครั้งที่ฟังก์ชันสาธารณะของ module นี้ถูกเรียก
 * (external linkage ผ่าน extern — definition จริงอยู่ใน math_utils.c) */
extern long mu_call_count;

void mu_reset_call_count(void);
long mu_get_call_count(void);

#endif /* MATH_UTILS_H */
```

### `math_utils.c`

```c
/* ============================================================
 * math_utils.c — implementation ของ module math_utils
 * ============================================================ */
#include "math_utils.h"

/* ---------- Definition ของตัวแปร global (มีจุดเดียวในทั้งโปรแกรม) ---------- */
long mu_call_count = 0;

/* ---------- static: internal linkage ----------
 * ฟังก์ชันช่วยเหล่านี้เป็นรายละเอียดภายในของ module
 * ไฟล์อื่นมองไม่เห็นและเรียกใช้ไม่ได้เลย */
static void mu_track_call(void) {
    mu_call_count++;
}

static int mu_abs_int(int x) {
    return (x < 0) ? -x : x;
}

int mu_add(int a, int b) {
    mu_track_call();
    return a + b;
}

int mu_subtract(int a, int b) {
    mu_track_call();
    return a - b;
}

long mu_power(int base, unsigned int exponent) {
    mu_track_call();
    long result = 1;
    for (unsigned int i = 0; i < exponent; i++) {
        result *= base;
    }
    return result;
}

int mu_gcd(int a, int b) {
    mu_track_call();
    a = mu_abs_int(a);
    b = mu_abs_int(b);
    while (b != 0) {
        int temp = b;
        b = a % b;
        a = temp;
    }
    return a;
}

int mu_is_prime(int n) {
    mu_track_call();
    if (n < 2) {
        return 0;
    }
    for (int i = 2; (long)i * i <= n; i++) {
        if (n % i == 0) {
            return 0;
        }
    }
    return 1;
}

double mu_average(const int *values, size_t count) {
    mu_track_call();
    if (count == 0) {
        return 0.0;
    }
    long sum = 0;
    for (size_t i = 0; i < count; i++) {
        sum += values[i];
    }
    return (double)sum / (double)count;
}

void mu_reset_call_count(void) {
    mu_call_count = 0;
}

long mu_get_call_count(void) {
    return mu_call_count;
}
```

### `main.c`

```c
/* ============================================================
 * main.c — โปรแกรมที่ใช้งาน module math_utils
 * ============================================================ */
#include <stdio.h>
#include "math_utils.h"

int main(void) {
    printf("mu_add(2, 3)      = %d\n", mu_add(2, 3));
    printf("mu_subtract(10,4) = %d\n", mu_subtract(10, 4));
    printf("mu_power(2, 10)   = %ld\n", mu_power(2, 10));
    printf("mu_gcd(48, 18)    = %d\n", mu_gcd(48, 18));
    printf("mu_is_prime(17)   = %d\n", mu_is_prime(17));

    int scores[] = {80, 90, 70, 100, 60};
    size_t n = sizeof(scores) / sizeof(scores[0]);
    printf("mu_average(...)   = %.2f\n", mu_average(scores, n));

    printf("เรียกฟังก์ชันใน module ไปทั้งหมด %ld ครั้ง\n", mu_get_call_count());

    return 0;
}
```

### คอมไพล์แยกกันแล้ว Link รวม

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 -c math_utils.c -o math_utils.o
gcc -Wall -Wextra -Wpedantic -std=c17 -c main.c -o main.o
gcc math_utils.o main.o -o program
./program
```

ผลลัพธ์:

```
mu_add(2, 3)      = 5
mu_subtract(10,4) = 6
mu_power(2, 10)   = 1024
mu_gcd(48, 18)    = 6
mu_is_prime(17)   = 1
mu_average(...)   = 80.00
เรียกฟังก์ชันใน module ไปทั้งหมด 6 ครั้ง
```

ลองสังเกตว่า `main.o` **ไม่รู้เลย** ว่า `mu_add` ถูก implement อย่างไรข้างใน มันรู้แค่ prototype
จาก header ว่า "รับ int สองตัว คืน int หนึ่งตัว" — นี่คือแก่นของ Modular Programming: **แยก
"สิ่งที่ต้องรู้เพื่อใช้งาน" (interface) ออกจาก "วิธีการทำงานภายใน" (implementation)** อย่าง
เด็ดขาด ทำให้ทีมพัฒนาโปรแกรมขนาดใหญ่สามารถทำงานคู่ขนานกันได้ — คนหนึ่งเขียน
`math_utils.c` อีกคนเขียน `main.c` โดยตกลงกันแค่หน้าตาของ `math_utils.h` ก็เพียงพอแล้ว
ที่จะทำงานพร้อมกันได้โดยไม่ชนกัน

> ใน Part 18 ที่กำลังจะถึง เราจะเรียนรู้วิธีใช้ **Makefile** เพื่อจัดการขั้นตอนคอมไพล์-link
> หลายไฟล์แบบนี้โดยอัตโนมัติ แทนที่จะพิมพ์คำสั่ง `gcc` ยาวๆ ซ้ำมือทุกครั้ง

---

## ข้อผิดพลาดที่พบบ่อย (Common Pitfalls)

1. **ใส่ implementation ของฟังก์ชันไว้ใน header โดยตรง** — ทำให้เกิด `multiple definition`
   error ทันทีที่ header นั้นถูก include เข้าไปมากกว่าหนึ่ง Compilation Unit ที่ link รวมกัน
   Header Guard **ป้องกันปัญหานี้ไม่ได้** เพราะมันป้องกันแค่การแปะซ้ำ*ภายในไฟล์เดียว*
2. **ประกาศตัวแปร global โดยไม่ใช้ `extern` ในไฟล์ที่สอง** — กลายเป็นการ define ซ้ำ เกิด
   `multiple definition of` ตอน link แม้แต่ละไฟล์จะคอมไพล์ผ่านแยกกันโดยไม่มี warning เลยก็ตาม
3. **ลืม `#include` header ของ module ตัวเองใน `.c` ไฟล์** — เสียโอกาสให้ compiler ตรวจสอบ
   ว่า implementation ตรงกับ prototype ที่ประกาศไว้หรือไม่ ถ้าพิมพ์ signature ผิด (เช่น
   ลำดับ parameter สลับกัน) จะไม่มี error ให้เห็นจนกว่าจะไป link กับไฟล์อื่นที่เรียกใช้ผิดแบบ
4. **ลืมใส่ `static` ให้ฟังก์ชัน/ตัวแปรช่วยภายใน** — ทำให้ทุกฟังก์ชันกลายเป็น external
   linkage โดยไม่ตั้งใจ เสี่ยงชื่อชนกับไฟล์อื่นในโปรเจกต์ใหญ่ และเปิดเผยรายละเอียดภายในที่
   ไม่ควรให้ใครเห็น
5. **สับสนระหว่าง `#include <...>` กับ `#include "..."`** — ใช้ `<...>` กับ header ของตัวเอง
   จะทำให้ compiler หา system path ก่อน ซึ่งอาจหาไม่เจอ (หรือแย่กว่านั้นคือไปเจอไฟล์ชื่อ
   เดียวกันจาก library อื่นโดยไม่ตั้งใจ) ให้จำไว้ว่า header ที่เขียนเองในโปรเจกต์ใช้ `"..."` เสมอ
6. **Header ไม่มี Header Guard** — ถ้า module อื่นดันไป `#include` header เดียวกันสองครั้งใน
   Compilation Unit เดียว (ผ่านการ include ทางอ้อมของหลาย header) จะเกิด error
   "redefinition" ของทุก struct/typedef ที่ประกาศไว้ในนั้น

---

## แบบฝึกหัดท้ายบท

1. สร้าง module ใหม่ชื่อ `string_utils.h` / `string_utils.c` ที่มีฟังก์ชัน `su_str_reverse`
   (คืน string ที่กลับด้าน โดยใช้ buffer ที่ผู้เรียกจัดสรรมาให้) และ `su_str_is_palindrome`
   (ตรวจว่า string เป็นพาลินโดรมหรือไม่) แล้วเขียน `main.c` มาทดสอบทั้งสองฟังก์ชัน
2. เพิ่มฟังก์ชัน `mu_lcm` (หา ค.ร.น. โดยใช้ `mu_gcd` ที่มีอยู่แล้ว) เข้าไปใน module
   `math_utils` ทั้ง header และ implementation แล้ว compile ทดสอบ
3. ทดลองลบ `static` ออกจาก `mu_track_call` ใน `math_utils.c` แล้วสร้างไฟล์ `extra.c` อีก
   ไฟล์ที่มีฟังก์ชันชื่อ `mu_track_call` เหมือนกัน (implement ต่างกันก็ได้) แล้วลองคอมไพล์และ
   link ทั้งสามไฟล์เข้าด้วยกัน สังเกต error ที่ได้ และอธิบายว่าทำไม `static` ถึงป้องกันปัญหานี้ได้
4. ออกแบบ module ชื่อ `shape_utils` ที่มี struct `Rectangle` (เก็บ `width`, `height`) และ
   ฟังก์ชัน `shape_rect_area`, `shape_rect_perimeter` โดยให้ struct และฟังก์ชันทั้งหมดเป็น
   public API ที่เหมาะสม (ตัดสินใจเองว่าอะไรควรอยู่ใน header อะไรควรเป็น `static` ใน `.c`)
5. เขียนโปรแกรมที่มีตัวแปร `extern int g_error_count;` ที่ใช้ร่วมกันระหว่าง module สองตัว
   (เช่น `validator.c` และ `logger.c`) โดยทุกครั้งที่ `validator.c` เจอข้อมูลผิดพลาด ให้เพิ่มค่า
   ตัวแปรนี้ และ `logger.c` มีฟังก์ชัน `print_error_summary()` ที่พิมพ์ค่าตัวแปรนี้ออกมา
6. จงอธิบายด้วยคำพูดของตัวเอง (เขียนเป็นคอมเมนต์ในไฟล์ `.c` ก็ได้) ว่าทำไมการใส่ body ของ
   ฟังก์ชันไว้ใน header โดยตรงถึงทำให้เกิด multiple definition error เมื่อ header นั้นถูก include
   จากมากกว่าหนึ่งไฟล์ แม้จะมี Header Guard แล้วก็ตาม

### แนวทางเฉลยข้อ 1

**`string_utils.h`**

```c
#ifndef STRING_UTILS_H
#define STRING_UTILS_H

#include <stddef.h> /* size_t */

/* กลับด้าน string `src` แล้วเขียนผลลัพธ์ลงใน `dest`
 * dest ต้องมีขนาดเพียงพอ (อย่างน้อยเท่ากับ strlen(src) + 1)
 * dest และ src ห้ามเป็น buffer เดียวกัน */
void su_str_reverse(const char *src, char *dest);

/* คืนค่า 1 ถ้า `s` เป็นพาลินโดรม (ไม่สนตัวพิมพ์ใหญ่/เล็ก) มิฉะนั้นคืน 0 */
int su_str_is_palindrome(const char *s);

#endif /* STRING_UTILS_H */
```

**`string_utils.c`**

```c
#include "string_utils.h"
#include <string.h>
#include <ctype.h>

void su_str_reverse(const char *src, char *dest) {
    size_t len = strlen(src);
    for (size_t i = 0; i < len; i++) {
        dest[i] = src[len - 1 - i];
    }
    dest[len] = '\0';
}

static int su_normalize_char(unsigned char c) {
    return tolower(c);
}

int su_str_is_palindrome(const char *s) {
    size_t len = strlen(s);
    if (len == 0) {
        return 1;
    }
    size_t left = 0;
    size_t right = len - 1;
    while (left < right) {
        int c1 = su_normalize_char((unsigned char)s[left]);
        int c2 = su_normalize_char((unsigned char)s[right]);
        if (c1 != c2) {
            return 0;
        }
        left++;
        right--;
    }
    return 1;
}
```

**`main.c`**

```c
#include <stdio.h>
#include <string.h>
#include "string_utils.h"

int main(void) {
    const char *word = "Hello";
    char reversed[32];

    su_str_reverse(word, reversed);
    printf("reverse(\"%s\") = \"%s\"\n", word, reversed);

    const char *tests[] = {"level", "Racecar", "Hello", "A"};
    size_t n = sizeof(tests) / sizeof(tests[0]);
    for (size_t i = 0; i < n; i++) {
        printf("is_palindrome(\"%s\") = %d\n",
               tests[i], su_str_is_palindrome(tests[i]));
    }

    return 0;
}
```

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 -c string_utils.c -o string_utils.o
gcc -Wall -Wextra -Wpedantic -std=c17 -c main.c -o main.o
gcc string_utils.o main.o -o program
./program
```

ผลลัพธ์:

```
reverse("Hello") = "olleH"
is_palindrome("level") = 1
is_palindrome("Racecar") = 1
is_palindrome("Hello") = 0
is_palindrome("A") = 1
```

### แนวทางเฉลยข้อ 5

**`error_common.h`** (header กลางที่ทั้งสอง module include ร่วมกัน)

```c
#ifndef ERROR_COMMON_H
#define ERROR_COMMON_H

extern int g_error_count;

#endif /* ERROR_COMMON_H */
```

**`validator.h`**

```c
#ifndef VALIDATOR_H
#define VALIDATOR_H

/* ตรวจว่า x อยู่ในช่วง [0, 100] หรือไม่ ถ้าไม่ ให้เพิ่ม g_error_count
 * คืนค่า 1 ถ้าข้อมูลถูกต้อง, 0 ถ้าผิดพลาด */
int validate_score(int x);

#endif /* VALIDATOR_H */
```

**`validator.c`**

```c
#include "validator.h"
#include "error_common.h"

int validate_score(int x) {
    if (x < 0 || x > 100) {
        g_error_count++;
        return 0;
    }
    return 1;
}
```

**`logger.h`**

```c
#ifndef LOGGER_H
#define LOGGER_H

void print_error_summary(void);

#endif /* LOGGER_H */
```

**`logger.c`**

```c
#include <stdio.h>
#include "logger.h"
#include "error_common.h"

void print_error_summary(void) {
    printf("พบข้อมูลผิดพลาดทั้งหมด %d รายการ\n", g_error_count);
}
```

**`main.c`**

```c
#include "validator.h"
#include "logger.h"
#include "error_common.h"

/* นิยามจริงของตัวแปร global — มีจุดเดียวในทั้งโปรแกรม */
int g_error_count = 0;

int main(void) {
    int inputs[] = {85, -5, 101, 42, 200};
    int n = (int)(sizeof(inputs) / sizeof(inputs[0]));

    for (int i = 0; i < n; i++) {
        validate_score(inputs[i]);
    }

    print_error_summary();
    return 0;
}
```

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 -c validator.c -o validator.o
gcc -Wall -Wextra -Wpedantic -std=c17 -c logger.c -o logger.o
gcc -Wall -Wextra -Wpedantic -std=c17 -c main.c -o main.o
gcc validator.o logger.o main.o -o program
./program
```

ผลลัพธ์:

```
พบข้อมูลผิดพลาดทั้งหมด 3 รายการ
```

สังเกตว่า `g_error_count` **นิยามจริงอยู่ใน `main.c`** เพียงจุดเดียว ส่วน `validator.c` และ
`logger.c` เห็นแค่ `extern int g_error_count;` ที่มาจาก `error_common.h` — นี่คือรูปแบบทั่วไป
ของการแชร์ state ข้าม module หลายตัวในโปรแกรม C แม้จะใช้งานได้จริง แต่ในทางปฏิบัติของ
โปรเจกต์ใหญ่มักนิยมส่งค่าผ่าน struct/parameter แทนตัวแปร global เพื่อให้ทดสอบและ debug
ง่ายกว่า (เป็นหัวข้อที่จะกลับมาเจาะลึกอีกครั้งเมื่อพูดถึง Software Architecture ใน Part 114)

---

## สรุปท้ายบท

ใน Part นี้เราได้:

- เข้าใจว่า **Compilation Unit** คือหน่วยที่ compiler แปลแยกกันในแต่ละครั้ง และทำไมโปรแกรม
  ขนาดใหญ่ต้องถูกแบ่งเป็นหลายไฟล์
- แยกโค้ดเป็น **header (.h)** สำหรับประกาศ public API และ **source (.c)** สำหรับ
  implementation จริง ตามหลักปฏิบัติมาตรฐานของภาษา C
- เข้าใจความแตกต่างระหว่าง **Declaration** กับ **Definition** อย่างถ่องแท้
- ใช้ `extern` เพื่อแชร์ตัวแปร/ฟังก์ชันข้ามไฟล์ (**External Linkage**)
- ใช้ `static` ที่ scope ระดับไฟล์เพื่อซ่อนรายละเอียดภายในของ module (**Internal Linkage**)
- วินิจฉัยและป้องกันปัญหา **Multiple Definition Error** ที่เกิดจาก header ออกแบบผิดพลาด
- ออกแบบ Public API ที่ดีผ่าน header file ที่สะอาดและอ่านง่าย
- สร้างโปรเจกต์จริงที่แยกเป็น `math_utils.h/.c` และ `main.c` คอมไพล์แยกกันแล้ว link รวม
  เป็นโปรแกรมเดียว

การแยกโค้ดเป็นหลายไฟล์แบบนี้ยังมีข้อจำกัดหนึ่งที่เห็นได้ชัดจากตัวอย่างใน Part นี้:
ทุกครั้งที่จะคอมไพล์โปรเจกต์ เราต้องพิมพ์คำสั่ง `gcc` ยาวๆ หลายบรรทัดด้วยมือ และถ้าลืม
คอมไพล์ไฟล์ที่แก้ไขใหม่ ก็อาจได้โปรแกรมที่ใช้โค้ดเก่าโดยไม่รู้ตัว ใน **Part 18** เราจะเรียนรู้
เครื่องมือที่แก้ปัญหานี้โดยตรง — **Makefile** — ซึ่งจะทำให้การ build โปรเจกต์หลายไฟล์เป็นไป
โดยอัตโนมัติ เร็วขึ้น และปลอดภัยจากความผิดพลาดของมนุษย์

**ต่อไป:** [Part 18 — Makefile และ Build Automation](./part-018-makefile-build.md)
