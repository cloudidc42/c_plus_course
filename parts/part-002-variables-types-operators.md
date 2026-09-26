# Part 2: ตัวแปร ชนิดข้อมูล และตัวดำเนินการ (Step 9–16)

> Module A — รากฐานภาษา C (Foundations of C) | Part 2 จาก 125
> Step ที่ครอบคลุมใน Part นี้: Step 9–16
> Part ก่อนหน้า: [Part 1 — เริ่มต้นกับภาษา C](./part-001-intro-to-c.md) | Part ถัดไป: [Part 3 — การรับส่งข้อมูลเชิงลึก (printf/scanf)](./part-003-io-printf-scanf.md)

## เป้าหมายการเรียนรู้ (Learning Objectives)

หลังจบ Part นี้ ผู้เรียนจะสามารถ:

1. อธิบายชนิดข้อมูลพื้นฐานของ C ทั้งหมด (`int`, `float`, `double`, `char`, `short`, `long`, `unsigned`) พร้อมบอกขนาดและช่วงค่าโดยประมาณได้
2. ใช้ตัวดำเนินการ `sizeof` เพื่อตรวจสอบขนาดจริงของชนิดข้อมูลบนเครื่องที่ใช้งาน และอธิบายได้ว่าทำไมขนาดของชนิดข้อมูลใน C จึงไม่ตายตัว
3. ประกาศตัวแปรตามกฎของภาษาและ Naming Convention ที่เป็นมาตรฐานอุตสาหกรรม พร้อมหลีกเลี่ยงชื่อที่เป็นปัญหา
4. อธิบายและใช้งาน Type Conversion ทั้งแบบ Implicit (อัตโนมัติ) และ Explicit (Casting) ได้อย่างปลอดภัยและมีเจตนา
5. ใช้ตัวดำเนินการทางคณิตศาสตร์ เปรียบเทียบ ตรรกะ บิต (Bitwise) assignment และ increment/decrement ได้อย่างถูกต้อง
6. อ่านตาราง Operator Precedence และเขียนนิพจน์ (Expression) ที่ซับซ้อนได้โดยไม่เกิดบั๊กจากลำดับความสำคัญผิดพลาด
7. คอมไพล์โปรแกรมที่ใช้ตัวแปรและตัวดำเนินการทุกประเภทด้วย `gcc -Wall -Wextra -Wpedantic -std=c17` โดยไม่มี Warning แม้แต่บรรทัดเดียว

---

## 2.1 ชนิดข้อมูลพื้นฐานใน C (Step 9)

ลองนึกภาพหน่วยความจำของคอมพิวเตอร์เป็น "กล่องเก็บของ" เรียงต่อกันนับล้านกล่อง แต่ละกล่อง
มีที่อยู่ (Address) ของตัวเอง เมื่อเราประกาศตัวแปร เราสั่งให้ compiler จอง "กล่อง" จำนวนหนึ่ง
ไว้ให้เรา และ **ชนิดข้อมูล (Data Type)** คือสิ่งที่บอก compiler ว่าจะจองกี่กล่อง (กี่ Byte)
และตีความเลขฐานสองในกล่องเหล่านั้นอย่างไร

C แบ่งชนิดข้อมูลพื้นฐาน (Primitive Type) ออกเป็น 2 กลุ่มใหญ่: **จำนวนเต็ม (Integer)** และ
**จำนวนทศนิยม (Floating Point)** โดยแต่ละกลุ่มมีหลายขนาดให้เลือกใช้ตามความต้องการ

| ชนิดข้อมูล | ขนาดทั่วไป (Linux 64-bit) | ช่วงค่าโดยประมาณ | ใช้เมื่อไร |
|---|---|---|---|
| `char` | 1 byte | -128 ถึง 127 (หรือ 0–255 ถ้า unsigned) | เก็บตัวอักษร 1 ตัว หรือเลขจำนวนเต็มเล็กมาก |
| `short` | 2 bytes | -32,768 ถึง 32,767 | เลขจำนวนเต็มขนาดเล็ก ประหยัดหน่วยความจำ |
| `int` | 4 bytes | ประมาณ -2.1 พันล้าน ถึง 2.1 พันล้าน | เลขจำนวนเต็มทั่วไป (ชนิด default ที่ใช้บ่อยที่สุด) |
| `long` | 8 bytes (Linux) / 4 bytes (Windows) | ใหญ่กว่า `int` มาก | เลขจำนวนเต็มขนาดใหญ่ |
| `long long` | 8 bytes | อย่างน้อย ±9.2 ล้านล้านล้าน | เลขจำนวนเต็มขนาดใหญ่มาก รับประกันขั้นต่ำตาม C99 |
| `float` | 4 bytes | ความละเอียดประมาณ 6-7 หลัก | ทศนิยมที่ไม่ต้องการความละเอียดสูง ประหยัดหน่วยความจำ |
| `double` | 8 bytes | ความละเอียดประมาณ 15-16 หลัก | ทศนิยมทั่วไป (ชนิด default ของค่าทศนิยมใน C) |

> **สิ่งสำคัญ**: มาตรฐาน C **ไม่ได้กำหนดขนาดที่ตายตัว** ของแต่ละชนิดข้อมูล มาตรฐานกำหนด
> เพียง "ขนาดขั้นต่ำ" และความสัมพันธ์ระหว่างชนิด (เช่น `sizeof(short) <= sizeof(int) <=
> sizeof(long) <= sizeof(long long)`) ขนาดจริงขึ้นอยู่กับ Compiler และ Platform ที่คอมไพล์
> นี่คือเหตุผลที่เราต้องเรียนรู้การใช้ `sizeof` ในหัวข้อถัดไป แทนที่จะเดาขนาดเอาเอง

ลองมาดูตัวอย่างการใช้งานชนิดข้อมูลพื้นฐานและขอบเขตค่าของมันจริงๆ ผ่าน `<limits.h>` และ
`<float.h>` ซึ่งเป็น Header มาตรฐานที่นิยาม Macro บอกค่าขอบเขตของแต่ละชนิด:

```c
/* type_ranges.c */
#include <stdio.h>
#include <limits.h>
#include <float.h>

int main(void) {
    printf("=== จำนวนเต็มมีเครื่องหมาย (Signed Integer) ===\n");
    printf("char      : %zu byte,  range %d ถึง %d\n",
           sizeof(char), CHAR_MIN, CHAR_MAX);
    printf("short     : %zu bytes, range %d ถึง %d\n",
           sizeof(short), SHRT_MIN, SHRT_MAX);
    printf("int       : %zu bytes, range %d ถึง %d\n",
           sizeof(int), INT_MIN, INT_MAX);
    printf("long      : %zu bytes, range %ld ถึง %ld\n",
           sizeof(long), LONG_MIN, LONG_MAX);
    printf("long long : %zu bytes, range %lld ถึง %lld\n",
           sizeof(long long), LLONG_MIN, LLONG_MAX);

    printf("\n=== จำนวนเต็มไม่มีเครื่องหมาย (Unsigned Integer) ===\n");
    printf("unsigned char : %zu byte,  max %u\n",
           sizeof(unsigned char), (unsigned int)UCHAR_MAX);
    printf("unsigned int  : %zu bytes, max %u\n",
           sizeof(unsigned int), UINT_MAX);
    printf("unsigned long : %zu bytes, max %lu\n",
           sizeof(unsigned long), ULONG_MAX);

    printf("\n=== จำนวนทศนิยม (Floating Point) ===\n");
    printf("float  : %zu bytes, ความละเอียดประมาณ %d หลัก\n",
           sizeof(float), FLT_DIG);
    printf("double : %zu bytes, ความละเอียดประมาณ %d หลัก\n",
           sizeof(double), DBL_DIG);

    return 0;
}
```

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 type_ranges.c -o type_ranges
./type_ranges
```

สังเกตว่าเราใช้ `%zu` สำหรับผลลัพธ์ของ `sizeof` เพราะ `sizeof` คืนค่าชนิด `size_t`
(จำนวนเต็มไม่มีเครื่องหมายที่ compiler เลือกขนาดให้เหมาะกับระบบ) การใช้ `%d` กับ `size_t`
จะทำให้เกิด Warning เรื่อง format mismatch ทันทีเมื่อเปิด `-Wall -Wextra`

### `char` คือจำนวนเต็ม ไม่ใช่ "ตัวอักษร" จริงๆ

สิ่งที่มือใหม่มักงงคือ `char` แท้จริงแล้วเป็นชนิดจำนวนเต็มขนาด 1 byte คอมพิวเตอร์ไม่รู้จัก
ตัวอักษร รู้จักแต่ตัวเลข ตัวอักษรที่เราเห็นเป็นเพียงการตีความตัวเลขตามตาราง **ASCII**

```c
/* char_is_number.c */
#include <stdio.h>

int main(void) {
    char grade = 'A';
    printf("grade เก็บอักขระ '%c' แต่จริงๆ คือเลข %d ในหน่วยความจำ (รหัส ASCII)\n",
           grade, grade);

    int code = 66;
    printf("เลข %d ตีความเป็นอักขระ '%c'\n", code, code);

    return 0;
}
```

ผลลัพธ์:

```
grade เก็บอักขระ 'A' แต่จริงๆ คือเลข 65 ในหน่วยความจำ (รหัส ASCII)
เลข 66 ตีความเป็นอักขระ 'B'
```

### `float` กับ `double` ต่างกันตรงความละเอียด ไม่ใช่แค่ขนาด

```c
/* float_vs_double.c */
#include <stdio.h>

int main(void) {
    float  f = 1.0f / 3.0f;
    double d = 1.0  / 3.0;

    printf("float  1/3 = %.15f\n", (double)f);
    printf("double 1/3 = %.15f\n", d);

    return 0;
}
```

ผลลัพธ์ (ประมาณ):

```
float  1/3 = 0.333333343267441
double 1/3 = 0.333333333333333
```

`float` เริ่มคลาดเคลื่อนตั้งแต่หลักที่ 8 ในขณะที่ `double` แม่นยำไปได้ถึงหลักที่ 15-16
สังเกตว่าเราใส่ `f` ต่อท้ายค่าคงที่ทศนิยม (`1.0f`) เพื่อบอก compiler ว่านี่คือค่าคงที่ชนิด
`float` — ถ้าไม่ใส่ `f` ค่าคงที่ทศนิยมทุกตัวใน C จะถูกตีความเป็น `double` โดย default
และเมื่อนำไปเก็บใน `float` จะถูกลดความละเอียดลง (แม้จะไม่มี Warning ก็ตาม)

---

## 2.2 sizeof และชนิดข้อมูลที่มีขนาดแน่นอน (Step 10)

`sizeof` เป็น **Operator** (ไม่ใช่ฟังก์ชัน แม้จะเขียนคล้ายกัน) ที่ compiler คำนวณผลลัพธ์
ให้ตั้งแต่ตอนคอมไพล์ (Compile Time) โดยคืนค่าเป็นจำนวน Byte ของชนิดข้อมูลหรือตัวแปรที่ระบุ

```c
/* sizeof_demo.c */
#include <stdio.h>

int main(void) {
    int    n = 10;
    double arr[5];

    printf("sizeof(int)      = %zu\n", sizeof(int));
    printf("sizeof(n)        = %zu\n", sizeof n);        /* ไม่ต้องมีวงเล็บก็ได้กับตัวแปร */
    printf("sizeof(arr)      = %zu\n", sizeof(arr));     /* ขนาดทั้ง array = 5 * sizeof(double) */
    printf("sizeof(arr[0])   = %zu\n", sizeof(arr[0]));

    return 0;
}
```

> **ข้อสังเกตเรื่องไวยากรณ์**: `sizeof` ใช้กับ "ตัวแปร" ได้โดยไม่ต้องมีวงเล็บ (เช่น `sizeof n`)
> แต่ถ้าใช้กับ "ชื่อชนิดข้อมูล" เช่น `sizeof(int)` **ต้องมีวงเล็บเสมอ** เพราะ `int` ไม่ใช่นิพจน์
> (Expression) เพื่อความสม่ำเสมอ หลักสูตรนี้จะใส่วงเล็บให้ `sizeof` เสมอทุกกรณี

### ปัญหาของขนาดที่ไม่แน่นอน และทางแก้ด้วย `<stdint.h>`

เนื่องจาก `int`, `long` อาจมีขนาดต่างกันไปในแต่ละแพลตฟอร์ม โปรแกรมที่ต้องการควบคุมขนาด
ตัวแปรอย่างเคร่งครัด (เช่น โปรแกรมสื่อสารผ่านเครือข่าย, Embedded System, File Format)
จำเป็นต้องใช้ชนิดข้อมูลจาก `<stdint.h>` ซึ่ง**รับประกันขนาดที่แน่นอนในทุกแพลตฟอร์ม**:

| ชนิดข้อมูลจาก stdint.h | ขนาดรับประกัน | ช่วงค่า |
|---|---|---|
| `int8_t` / `uint8_t` | 1 byte | -128..127 / 0..255 |
| `int16_t` / `uint16_t` | 2 bytes | -32768..32767 / 0..65535 |
| `int32_t` / `uint32_t` | 4 bytes | ประมาณ ±2.1 พันล้าน / 0..4.2 พันล้าน |
| `int64_t` / `uint64_t` | 8 bytes | ประมาณ ±9.2 ล้านล้านล้าน / 0..1.8x10^19 |
| `size_t` | ขึ้นกับระบบ (มักเท่ากับความกว้าง address) | ใช้เก็บขนาด/ดัชนีของหน่วยความจำเสมอ |

```c
/* stdint_demo.c */
#include <stdio.h>
#include <stdint.h>
#include <inttypes.h>   /* จำเป็นสำหรับ macro PRId32, PRIu8 ฯลฯ */

int main(void) {
    int32_t  a = 100000;
    uint8_t  b = 250;
    int64_t  c = 9000000000LL;

    printf("int32_t a = %" PRId32 "\n", a);
    printf("uint8_t b = %" PRIu8  "\n", b);
    printf("int64_t c = %" PRId64 "\n", c);

    return 0;
}
```

Macro อย่าง `PRId32` และ `PRIu8` มาจาก `<inttypes.h>` ถูกออกแบบมาเพื่อขยายเป็น format
specifier ที่ถูกต้องสำหรับแต่ละแพลตฟอร์มโดยอัตโนมัติ (เพราะ `int32_t` อาจเป็น `int` บนบาง
ระบบ แต่เป็น `long` บนบางระบบ) ทำให้โค้ดพกพาได้แม้เปลี่ยนแพลตฟอร์ม — เราจะกลับมาใช้ชนิด
ข้อมูลกลุ่มนี้อย่างจริงจังใน Module ที่เกี่ยวกับ Systems Programming และ Networking

---

## 2.3 การประกาศตัวแปรและ Naming Convention (Step 11)

การประกาศตัวแปรใน C มีรูปแบบ `ชนิดข้อมูล ชื่อตัวแปร;` หรือกำหนดค่าเริ่มต้นพร้อมกันได้เลย
`ชนิดข้อมูล ชื่อตัวแปร = ค่าเริ่มต้น;`

```c
int    age = 25;
double price = 199.50;
char   initial = 'K';
int    x, y, z;             /* ประกาศหลายตัวแปรชนิดเดียวกันในบรรทัดเดียวได้ */
int    width = 10, height = 20;  /* ประกาศพร้อมกำหนดค่าหลายตัวก็ได้ */
```

### กฎของภาษา (บังคับ)

1. ชื่อตัวแปรประกอบด้วยตัวอักษร (a-z, A-Z), ตัวเลข (0-9), และ underscore (`_`) เท่านั้น
2. **ห้ามขึ้นต้นด้วยตัวเลข** เช่น `2nd_place` ผิดกฎ
3. **Case-sensitive** — `age`, `Age`, `AGE` ถือเป็นตัวแปรคนละตัวกัน
4. ห้ามใช้ชื่อซ้ำกับ **Keyword สงวน** ของภาษา เช่น `int`, `return`, `for`, `if`, `while`,
   `struct`, `const`, `static`, `void`, `sizeof` (มี Keyword ทั้งหมด 32 คำใน C89 และเพิ่มขึ้น
   ในมาตรฐานหลัง เช่น `_Bool`, `_Generic`, `_Atomic`)
5. ชื่อที่ขึ้นต้นด้วย underscore ตามด้วยตัวพิมพ์ใหญ่ (เช่น `_Foo`) หรือ underscore สองตัว
   (`__foo`) เป็น**ชื่อที่มาตรฐานสงวนไว้ใช้ภายใน Compiler/Library เอง** ห้ามผู้เขียนโปรแกรม
   ตั้งชื่อแบบนี้

### Convention (ธรรมเนียมปฏิบัติ ไม่ใช่กฎบังคับของภาษา แต่เป็นมาตรฐานอุตสาหกรรม)

| แนวทาง | ตัวอย่าง | เหตุผล |
|---|---|---|
| ใช้ `snake_case` สำหรับตัวแปรและฟังก์ชัน | `total_price`, `is_valid` | เป็นธรรมเนียมของ C แทบทุกโครงการใหญ่ (Linux Kernel, PostgreSQL) |
| ใช้ `UPPER_SNAKE_CASE` สำหรับค่าคงที่ (`#define`/`const`) | `MAX_BUFFER_SIZE` | แยกให้เห็นชัดว่าเป็นค่าคงที่ ไม่ใช่ตัวแปรที่เปลี่ยนได้ |
| ตั้งชื่อให้สื่อความหมาย | `student_count` ไม่ใช่ `sc` | โค้ดถูกอ่านมากกว่าถูกเขียน — ชื่อดีคือเอกสารในตัวเอง |
| ใช้ตัวแปรชื่อสั้น (`i`, `j`, `k`) เฉพาะตัวนับ loop เท่านั้น | `for (int i = 0; ...)` | เป็นธรรมเนียมสากลที่อ่านแล้วเข้าใจทันที |
| หลีกเลี่ยงชื่อที่คล้ายกันเกินไป | ไม่ใช้ `data1`, `data2`, `data3` | ทำให้สลับใช้ผิดตัวได้ง่าย ควรตั้งชื่อสื่อความหมายต่างกันจริง |

```c
/* naming_demo.c */
#include <stdio.h>

#define MAX_STUDENTS 40   /* ค่าคงที่ระดับโปรแกรม ใช้ UPPER_SNAKE_CASE */

int main(void) {
    int  student_count = 32;          /* snake_case, สื่อความหมาย */
    double average_score = 78.5;
    const int max_score = 100;        /* ค่าคงที่ระดับตัวแปร ใช้ const */

    printf("นักเรียน %d จาก %d คน คะแนนเฉลี่ย %.1f จากเต็ม %d\n",
           student_count, MAX_STUDENTS, average_score, max_score);

    return 0;
}
```

> **ทำไมต้องมี `const` ทั้งที่มี `#define` แล้ว?** ทั้งสองใช้สร้างค่าคงที่ได้ แต่ทำงานต่างกัน:
> `#define` เป็นคำสั่งของ Preprocessor (แทนที่ Text ล้วนๆ ก่อน compile) ส่วน `const` เป็น
> ตัวแปรจริงที่มีชนิดข้อมูลและอยู่ในหน่วยความจำ ทำให้ debugger มองเห็นและตรวจสอบชนิดได้
> เราจะเจาะลึกเรื่องนี้อีกครั้งใน Part 14 (Preprocessor และ Macro)

---

## 2.4 Type Conversion: Implicit และ Explicit Casting (Step 12)

### Implicit Conversion (การแปลงชนิดอัตโนมัติ)

เมื่อนิพจน์มีตัวถูกดำเนินการ (Operand) คนละชนิดกัน compiler จะแปลงชนิดให้ตรงกันโดย
อัตโนมัติตามกฎที่เรียกว่า **Usual Arithmetic Conversions** โดยหลักการคร่าวๆ คือ
"แปลงชนิดที่มีข้อมูลน้อยกว่าให้กลายเป็นชนิดที่มีข้อมูลมากกว่า" (Type Promotion) เพื่อไม่ให้
สูญเสียข้อมูล:

```
char/short  →  int          (Integer Promotion — เกิดก่อนการคำนวณเสมอ)
int         →  unsigned int (ถ้ามีฝั่งใดฝั่งหนึ่งเป็น unsigned int)
int/unsigned int → long     (ถ้ามีฝั่งใดฝั่งหนึ่งเป็น long)
int/long    →  float/double (ถ้ามีฝั่งใดฝั่งหนึ่งเป็นทศนิยม)
float       →  double       (ถ้าอีกฝั่งเป็น double)
```

```c
/* implicit_conversion.c */
#include <stdio.h>

int main(void) {
    int    a = 7;
    double b = 2.0;
    double result = a / b;   /* a ถูกแปลงเป็น double โดยอัตโนมัติก่อนหาร */

    printf("7 / 2.0 = %.2f (แปลง int -> double อัตโนมัติ)\n", result);

    char c1 = 'A';
    char c2 = 'B';
    int  sum = c1 + c2;       /* c1, c2 ถูก promote เป็น int ก่อนบวกเสมอ */

    printf("'A' + 'B' = %d (บวกกันในรูปเลข ASCII หลัง promote เป็น int)\n", sum);

    return 0;
}
```

### กับดักคลาสสิก: Integer Division

```c
/* integer_division_trap.c */
#include <stdio.h>

int main(void) {
    int a = 7, b = 2;

    printf("7 / 2 (int หาร int)     = %d\n", a / b);          /* ได้ 3 ไม่ใช่ 3.5 */
    printf("7 / 2.0 (int หาร double) = %.1f\n", a / 2.0);      /* ได้ 3.5 */
    printf("(double)7 / 2           = %.1f\n", (double)a / b); /* ได้ 3.5 */

    return 0;
}
```

**กฎทอง**: ถ้าตัวถูกดำเนินการ (Operand) ทั้งสองฝั่งของ `/` เป็นจำนวนเต็มทั้งคู่ ผลลัพธ์จะเป็น
จำนวนเต็มเสมอ (ปัดเศษทิ้ง ไม่ใช่ปัดขึ้น-ลง) ไม่ว่าตัวแปรที่รับผลลัพธ์จะเป็น `double` ก็ตาม
เพราะ compiler ดูที่ชนิดของ **ตัวถูกดำเนินการ** ไม่ใช่ชนิดของตัวแปรฝั่งซ้ายของ `=`

### Explicit Conversion (Casting)

เราสามารถบังคับแปลงชนิดข้อมูลเองได้ด้วย Cast Operator: `(ชนิดข้อมูลใหม่)นิพจน์`

```c
/* explicit_cast.c */
#include <stdio.h>

int main(void) {
    double pi = 3.14159;
    int    truncated = (int)pi;             /* ตัดทศนิยมทิ้ง ไม่ใช่ปัดเศษ! ได้ 3 */

    printf("(int)3.14159 = %d\n", truncated);

    double negative = -3.9;
    printf("(int)-3.9 = %d\n", (int)negative); /* ได้ -3 (ตัดเข้าหาศูนย์ ไม่ใช่ปัดลง) */

    long big = 300L;
    unsigned char narrowed = (unsigned char)big;  /* บังคับแปลงลงชนิดเล็กกว่า */
    printf("(unsigned char)300L = %u (เกิด wraparound เพราะ 300 > 255)\n",
           (unsigned int)narrowed);

    return 0;
}
```

ผลลัพธ์:

```
(int)3.14159 = 3
(int)-3.9 = -3
(unsigned char)300L = 44 (เกิด wraparound เพราะ 300 > 255)
```

สังเกตว่า `(int)3.14159` ได้ `3` (**ตัดทศนิยมทิ้งเข้าหาศูนย์** — Truncation toward zero)
ไม่ใช่การปัดเศษ (Rounding) ถ้าต้องการปัดเศษต้องใช้ฟังก์ชัน `round()` จาก `<math.h>`
ซึ่งจะพูดถึงในบทที่เกี่ยวกับฟังก์ชันคณิตศาสตร์ต่อไป

ส่วนกรณี `(unsigned char)300L` ค่า 300 ใหญ่เกินกว่าที่ `unsigned char` (0-255) จะเก็บได้
เกิดพฤติกรรมที่เรียกว่า **wraparound**: ผลลัพธ์คือ `300 mod 256 = 44` การแปลงจากชนิดใหญ่
ไปชนิดเล็กกว่าสำหรับ unsigned type มีนิยามชัดเจนในมาตรฐาน (modulo 2^n) จึงไม่ใช่ Undefined
Behavior แต่ก็เป็นแหล่งบั๊กที่พบบ่อยมากถ้าไม่ระวังขนาดของชนิดข้อมูลที่ใช้

---

## 2.5 ตัวดำเนินการทางคณิตศาสตร์ Assignment และ Increment/Decrement (Step 13)

### ตัวดำเนินการทางคณิตศาสตร์พื้นฐาน

| Operator | ความหมาย | ตัวอย่าง (a=7, b=2) |
|---|---|---|
| `+` | บวก | `a + b` = 9 |
| `-` | ลบ | `a - b` = 5 |
| `*` | คูณ | `a * b` = 14 |
| `/` | หาร (int/int = int, ปัดทศนิยมทิ้ง) | `a / b` = 3 |
| `%` | หารเอาเศษ (Modulo, ใช้ได้กับจำนวนเต็มเท่านั้น) | `a % b` = 1 |

```c
/* arithmetic_ops.c */
#include <stdio.h>

int main(void) {
    int a = 7, b = 2;

    printf("%d + %d = %d\n", a, b, a + b);
    printf("%d - %d = %d\n", a, b, a - b);
    printf("%d * %d = %d\n", a, b, a * b);
    printf("%d / %d = %d\n", a, b, a / b);
    printf("%d %% %d = %d\n", a, b, a % b);   /* %% คือการพิมพ์เครื่องหมาย % ตรงๆ */

    printf("%d %% %d = %d (เศษติดลบตามเครื่องหมายตัวตั้ง)\n", -7, 2, -7 % 2);

    return 0;
}
```

ตั้งแต่มาตรฐาน C99 เป็นต้นไป ผลลัพธ์ของ `%` กับเลขลบมีนิยามชัดเจน: **เครื่องหมายของผลลัพธ์
`%` จะตรงกับเครื่องหมายของตัวตั้ง (Dividend)** เช่น `-7 % 2` ได้ `-1` (ก่อน C99 พฤติกรรมนี้
เป็น Implementation-defined)

### Assignment Operator แบบย่อ (Compound Assignment)

```c
/* compound_assignment.c */
#include <stdio.h>

int main(void) {
    int score = 10;

    score += 5;   /* เทียบเท่า score = score + 5;  -> 15 */
    printf("หลัง += 5  : %d\n", score);

    score -= 3;   /* -> 12 */
    printf("หลัง -= 3  : %d\n", score);

    score *= 2;   /* -> 24 */
    printf("หลัง *= 2  : %d\n", score);

    score /= 4;   /* -> 6 */
    printf("หลัง /= 4  : %d\n", score);

    score %= 4;   /* -> 2 */
    printf("หลัง %%= 4 : %d\n", score);

    return 0;
}
```

Compound Assignment ไม่ใช่แค่ย่อการพิมพ์ แต่ยังสื่อเจตนา (Intent) ให้คนอ่านโค้ดเข้าใจ
ทันทีว่า "กำลังปรับปรุงค่าเดิม" ซึ่งเป็นธรรมเนียมที่ใช้กันแพร่หลายในโค้ดมืออาชีพ

### Increment / Decrement: Prefix เทียบกับ Postfix

`++x` (Prefix) กับ `x++` (Postfix) ทั้งคู่เพิ่มค่า `x` ขึ้น 1 เหมือนกัน แต่ **ค่าที่นิพจน์นั้น
คืนกลับมาต่างกัน**:

- `++x` — เพิ่มค่าก่อน แล้วคืนค่าใหม่ (ที่เพิ่มแล้ว)
- `x++` — คืนค่าเดิมก่อน แล้วค่อยเพิ่มค่าทีหลัง

```c
/* prefix_vs_postfix.c */
#include <stdio.h>

int main(void) {
    int a = 5;
    int b = ++a;   /* a เป็น 6 ก่อน แล้วค่อยเอาไปให้ b -> a=6, b=6 */
    printf("prefix : a=%d, b=%d\n", a, b);

    int c = 5;
    int d = c++;   /* d ได้ค่าเดิมของ c (5) ก่อน แล้ว c ค่อยกลายเป็น 6 */
    printf("postfix: c=%d, d=%d\n", c, d);

    return 0;
}
```

ผลลัพธ์:

```
prefix : a=6, b=6
postfix: c=6, d=5
```

> **คำเตือนสำคัญ**: ห้ามใช้ตัวแปรเดียวกันมากกว่าหนึ่งครั้งในนิพจน์ที่มีการแก้ไขค่า (side
> effect) โดยไม่มีจุด **Sequence Point** คั่นกลาง เช่น `printf("%d %d", i++, i++);` หรือ
> `int r = i++ + i++;` เป็น **Undefined Behavior** — compiler มีสิทธิ์ประเมินค่าลำดับใดก่อน
> ก็ได้ และผลลัพธ์อาจต่างกันไปในแต่ละ compiler/optimization level แม้โค้ดจะดู "เดาได้"
> ก็ตาม กฎง่ายๆ ที่ปลอดภัยที่สุดคือ **แก้ไขตัวแปรเดียวได้แค่ครั้งเดียวต่อหนึ่ง statement**

---

## 2.6 ตัวดำเนินการเปรียบเทียบและตรรกะ (Step 14)

### ตัวดำเนินการเปรียบเทียบ (Relational Operators)

ผลลัพธ์ของตัวดำเนินการเปรียบเทียบใน C คือ `int` ที่มีค่า `1` (จริง) หรือ `0` (เท็จ) เท่านั้น
(ก่อน C99 ไม่มีชนิด `bool` แท้ๆ ในภาษา แต่ตั้งแต่ C99 มี `<stdbool.h>` ให้ใช้ `bool`,
`true`, `false` เป็น Macro/Alias ของ `_Bool`, `1`, `0`)

| Operator | ความหมาย |
|---|---|
| `==` | เท่ากับ |
| `!=` | ไม่เท่ากับ |
| `<` | น้อยกว่า |
| `>` | มากกว่า |
| `<=` | น้อยกว่าหรือเท่ากับ |
| `>=` | มากกว่าหรือเท่ากับ |

```c
/* relational_ops.c */
#include <stdio.h>
#include <stdbool.h>

int main(void) {
    int x = 10, y = 20;
    bool is_equal = (x == y);
    bool is_less  = (x < y);

    printf("x == y -> %d (%s)\n", is_equal, is_equal ? "true" : "false");
    printf("x < y  -> %d (%s)\n", is_less,  is_less  ? "true" : "false");

    return 0;
}
```

### ตัวดำเนินการตรรกะ (Logical Operators) และ Short-Circuit Evaluation

| Operator | ความหมาย |
|---|---|
| `&&` | AND (จริงทั้งคู่ถึงจะจริง) |
| `\|\|` | OR (จริงข้างใดข้างหนึ่งก็จริง) |
| `!` | NOT (กลับค่าความจริง) |

C ใช้กลไก **Short-Circuit Evaluation**: `&&` และ `\|\|` จะประเมินฝั่งขวาก็ต่อเมื่อจำเป็น
เท่านั้น — ถ้า `&&` ฝั่งซ้ายเป็นเท็จ ผลลัพธ์ต้องเป็นเท็จแน่นอนแล้ว compiler จะ**ไม่ประเมิน
ฝั่งขวาเลย**; ถ้า `||` ฝั่งซ้ายเป็นจริง ก็จะไม่ประเมินฝั่งขวาเช่นกัน

```c
/* short_circuit.c */
#include <stdio.h>

int print_and_return_zero(void) {
    printf("  -> ฟังก์ชันนี้ถูกเรียก!\n");
    return 0;
}

int main(void) {
    int x = 0;

    printf("ทดสอบ && กับฝั่งซ้ายเป็นเท็จ:\n");
    if (x != 0 && print_and_return_zero() == 0) {
        printf("เข้า if\n");
    } else {
        printf("ไม่เข้า if (ฟังก์ชันฝั่งขวาไม่ถูกเรียกเลย เพราะ x != 0 เป็นเท็จแล้ว)\n");
    }

    printf("\nทดสอบ || กับฝั่งซ้ายเป็นจริง:\n");
    if (x == 0 || print_and_return_zero() == 0) {
        printf("เข้า if (ฟังก์ชันฝั่งขวาไม่ถูกเรียกเลย เพราะ x == 0 เป็นจริงแล้ว)\n");
    }

    return 0;
}
```

Short-circuit ไม่ใช่แค่เรื่อง performance แต่เป็นเทคนิคที่ใช้ป้องกันบั๊กร้ายแรงได้ เช่น
การตรวจสอบ pointer ว่าไม่ใช่ `NULL` ก่อนจะเข้าถึงข้อมูลที่มันชี้ไป (จะได้เห็นตัวอย่างจริง
เมื่อเรียนเรื่อง Pointer ใน Part 8):

```c
/* ตัวอย่างแนวคิด (จะใช้จริงหลังเรียน Pointer) */
if (ptr != NULL && ptr->value > 0) {
    /* ปลอดภัย เพราะถ้า ptr เป็น NULL, ptr->value จะไม่ถูกประเมินเลย */
}
```

### กับดัก: `=` ปะปนกับ `==`

```c
/* assignment_vs_equality.c */
#include <stdio.h>

int main(void) {
    int flag = 0;

    if (flag = 1) {   /* ตั้งใจพิมพ์ == แต่พิมพ์ = ผิด! */
        printf("เข้า if เสมอ เพราะ flag = 1 คือนิพจน์ที่มีค่า 1 (จริงเสมอ)\n");
    }

    return 0;
}
```

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 assignment_vs_equality.c -o assignment_vs_equality
```

`-Wall` จะเตือนทันที:

```
warning: suggest parentheses around assignment used as truth value [-Wparentheses]
```

Compiler ไม่ถือว่านี่เป็น Error เพราะ `if (flag = 1)` เป็นไวยากรณ์ที่ถูกต้องตามกฎภาษา
(นิพจน์ assignment มีค่ากลับมาเสมอ) แต่ **แทบจะไม่มีทางที่ผู้เขียนตั้งใจเขียนแบบนี้จริงๆ**
ฉะนั้น Warning นี้สำคัญมาก และเป็นเหตุผลสำคัญข้อหนึ่งที่กฎทองของหลักสูตรนี้บังคับให้เปิด
`-Wall` เสมอ

---

## 2.7 Bitwise Operators เบื้องต้น (Step 15)

Bitwise Operator ทำงานกับข้อมูลใน**ระดับบิต** (0 และ 1) โดยตรง ซึ่งสำคัญมากในงาน Systems
Programming, Embedded, Networking Protocol และการเขียนโปรแกรมที่ต้องประหยัดหน่วยความจำ
สุดขีด (จะเจาะลึกอีกครั้งใน Part 15 — Bit Manipulation)

| Operator | ชื่อ | ความหมาย |
|---|---|---|
| `&` | AND | บิตเป็น 1 เมื่อทั้งสองฝั่งเป็น 1 |
| `\|` | OR | บิตเป็น 1 เมื่ออย่างน้อยฝั่งใดฝั่งหนึ่งเป็น 1 |
| `^` | XOR | บิตเป็น 1 เมื่อทั้งสองฝั่ง**ต่างกัน** |
| `~` | NOT (Complement) | กลับทุกบิต (0↔1) |
| `<<` | Left Shift | เลื่อนบิตไปทางซ้าย (เท่ากับคูณด้วย 2 ยกกำลัง n) |
| `>>` | Right Shift | เลื่อนบิตไปทางขวา (เท่ากับหารด้วย 2 ยกกำลัง n สำหรับ unsigned) |

```c
/* bitwise_demo.c */
#include <stdio.h>
#include <stdint.h>

/* ฟังก์ชันช่วยพิมพ์เลขฐาน 2 ของ uint8_t เพื่อให้เห็นภาพบิตชัดเจน */
void print_bits(uint8_t value) {
    for (int i = 7; i >= 0; i--) {
        putchar((value & (1u << i)) ? '1' : '0');
    }
    putchar('\n');
}

int main(void) {
    uint8_t a = 0x0C;   /* 0x0C = 12  = 0000 1100 ในเลขฐานสอง */
    uint8_t b = 0x0A;   /* 0x0A = 10  = 0000 1010 ในเลขฐานสอง */

    printf("a       = ");  print_bits(a);
    printf("b       = ");  print_bits(b);

    printf("a & b   = ");  print_bits((uint8_t)(a & b));
    printf("a | b   = ");  print_bits((uint8_t)(a | b));
    printf("a ^ b   = ");  print_bits((uint8_t)(a ^ b));
    printf("~a      = ");  print_bits((uint8_t)~a);
    printf("a << 2  = ");  print_bits((uint8_t)(a << 2));
    printf("a >> 2  = ");  print_bits((uint8_t)(a >> 2));

    return 0;
}
```

> **หมายเหตุเรื่อง Binary Literal**: หลายคนอยากเขียนค่าคงที่เป็นเลขฐานสองตรงๆ เช่น
> `0b00001100` ซึ่ง gcc/clang รองรับมานานในฐานะ GNU Extension และเพิ่งถูกบรรจุเป็นมาตรฐาน
> ทางการใน **C23** แต่ **ยังไม่ใช่ส่วนหนึ่งของ C17** ถ้าคอมไพล์ด้วย `-std=c17 -Wpedantic`
> แบบเคร่งครัด จะได้ Warning `binary constants are a C2X feature or GCC extension` ทันที
> เพื่อให้โค้ดในหลักสูตรนี้ compile ผ่านแบบ Standard-compliant 100% โดยไม่มี Warning เราจึง
> ใช้เลขฐานสิบหก (`0x0C`) แทน ซึ่งอ่านค่าฐานสองได้ไม่ยาก เพราะเลขฐานสิบหก 1 หลักตรงกับ
> เลขฐานสองพอดี 4 บิตเสมอ (`0x0C` = `1100`, `0x0A` = `1010`)

### เหตุผลที่ต้อง cast กลับเป็น `uint8_t` ทุกครั้ง

สังเกตว่าทุกผลลัพธ์ของ Bitwise Operator ถูก cast กลับเป็น `(uint8_t)` ก่อนส่งเข้า
`print_bits()` เพราะกฎ **Integer Promotion**: ตัวแปรชนิดเล็กกว่า `int` (เช่น `uint8_t`,
`char`, `short`) จะถูก promote เป็น `int` โดยอัตโนมัติก่อนเข้าสู่ทุกตัวดำเนินการทางคณิตศาสตร์
และ bitwise ผลลัพธ์ของ `a & b` จึงมีชนิดเป็น `int` (ไม่ใช่ `uint8_t`) เราจึงต้อง cast กลับ
เพื่อไม่ให้ compiler แจ้งเตือนเรื่อง narrowing conversion เมื่อส่งค่าให้พารามิเตอร์ชนิด
`uint8_t`

### ตัวอย่างการใช้งานจริง: Bit Flags

```c
/* bit_flags_intro.c */
#include <stdio.h>

#define FLAG_READ    (1 << 0)   /* 0001 */
#define FLAG_WRITE   (1 << 1)   /* 0010 */
#define FLAG_EXECUTE (1 << 2)   /* 0100 */

int main(void) {
    int permissions = FLAG_READ | FLAG_WRITE;   /* เปิดสิทธิ์อ่านและเขียน */

    printf("มีสิทธิ์อ่าน?    %s\n", (permissions & FLAG_READ)    ? "มี" : "ไม่มี");
    printf("มีสิทธิ์เขียน?   %s\n", (permissions & FLAG_WRITE)   ? "มี" : "ไม่มี");
    printf("มีสิทธิ์รัน?     %s\n", (permissions & FLAG_EXECUTE) ? "มี" : "ไม่มี");

    permissions |= FLAG_EXECUTE;  /* เพิ่มสิทธิ์รัน */
    printf("\nหลังเพิ่มสิทธิ์รัน:\n");
    printf("มีสิทธิ์รัน?     %s\n", (permissions & FLAG_EXECUTE) ? "มี" : "ไม่มี");

    return 0;
}
```

เทคนิคการเก็บหลายสถานะ (Flags) ไว้ในตัวแปรเดียวโดยใช้แต่ละบิตแทนแต่ละสถานะ เป็นเทคนิค
คลาสสิกที่ใช้จริงในระบบไฟล์ (File Permission แบบ Unix เช่น `chmod 755`), Network Protocol
Header และ Hardware Register — เราจะกลับมาเจาะลึกใน Part 15

---

## 2.8 Operator Precedence และ Associativity (Step 16)

เมื่อนิพจน์มีตัวดำเนินการหลายตัวปนกัน compiler ต้องมีกฎตัดสินว่าจะประมวลผลอันไหนก่อน
กฎนี้เรียกว่า **Precedence (ลำดับความสำคัญ)** และเมื่อตัวดำเนินการมีลำดับเท่ากัน จะใช้กฎ
**Associativity (ทิศทางการจับกลุ่ม)** ตัดสินว่าจะจับกลุ่มจากซ้ายไปขวา หรือขวาไปซ้าย

ตารางด้านล่างเรียงจากลำดับความสำคัญ**สูงสุด**ไปต่ำสุด (เฉพาะตัวดำเนินการที่เรียนถึงตอนนี้):

| ลำดับ | Operator | คำอธิบาย | Associativity |
|---|---|---|---|
| 1 (สูงสุด) | `()` `[]` | เรียกฟังก์ชัน, เข้าถึง array | ซ้าย → ขวา |
| 2 | `!` `~` `++` `--` `(type)` (unary +/-) | ตัวดำเนินการเอกภาค (Unary) | ขวา → ซ้าย |
| 3 | `*` `/` `%` | คูณ หาร มอด | ซ้าย → ขวา |
| 4 | `+` `-` | บวก ลบ | ซ้าย → ขวา |
| 5 | `<<` `>>` | Bit shift | ซ้าย → ขวา |
| 6 | `<` `<=` `>` `>=` | เปรียบเทียบ | ซ้าย → ขวา |
| 7 | `==` `!=` | เท่ากับ/ไม่เท่ากับ | ซ้าย → ขวา |
| 8 | `&` | Bitwise AND | ซ้าย → ขวา |
| 9 | `^` | Bitwise XOR | ซ้าย → ขวา |
| 10 | `\|` | Bitwise OR | ซ้าย → ขวา |
| 11 | `&&` | Logical AND | ซ้าย → ขวา |
| 12 | `\|\|` | Logical OR | ซ้าย → ขวา |
| 13 | `?:` | Ternary (จะเรียนใน Part 4) | ขวา → ซ้าย |
| 14 (ต่ำสุด) | `=` `+=` `-=` ฯลฯ | Assignment | ขวา → ซ้าย |

```c
/* precedence_demo.c */
#include <stdio.h>

int main(void) {
    int result1 = 2 + 3 * 4;         /* * ก่อน + -> 2 + 12 = 14 */
    int result2 = (2 + 3) * 4;       /* วงเล็บบังคับให้ + ก่อน -> 5 * 4 = 20 */

    printf("2 + 3 * 4   = %d\n", result1);
    printf("(2+3) * 4   = %d\n", result2);

    int a = 5, b = 3, c = 2;
    int result3 = a > b && b > c;    /* > ก่อน && เสมอ -> (5>3) && (3>2) -> 1 && 1 = 1 */
    printf("a>b && b>c  = %d\n", result3);

    int x = 1;
    int y = x << 1 + 1;              /* + มีลำดับสูงกว่า << ! -> x << (1+1) -> 1 << 2 = 4 */
    printf("x << 1 + 1  = %d (คนละความหมายกับที่อาจคาดหวังว่า (x<<1)+1 = 3!)\n", y);

    return 0;
}
```

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 precedence_demo.c -o precedence_demo
```

น่าสนใจว่าบรรทัดสุดท้ายจะมี Warning โผล่ขึ้นมาทันที:

```
warning: suggest parentheses around '+' inside '<<' [-Wparentheses]
```

Compiler เองก็รู้ว่านิพจน์แบบนี้มักทำให้คนอ่านสับสน จึงเตือนให้ใส่วงเล็บกำกับความหมายให้
ชัดเจน แม้ผลลัพธ์ที่ได้ (`x << (1+1)`) จะ**ถูกต้องตามกฎภาษา** ก็ตาม นี่คือกับดักที่พบได้บ่อย
มาก: หลายคนคิดว่า `<<` มีลำดับความสำคัญสูงเหมือนตัวดำเนินการทางคณิตศาสตร์อื่นๆ แต่ตาม
ตารางด้านบน **`+` มีลำดับสูงกว่า `<<`** ดังนั้น `1 + 1` จะถูกคำนวณก่อนกลายเป็น `x << 2`
(ได้ 4) ไม่ใช่ `(x << 1) + 1` (ที่จะได้ 3) — Warning นี้เองคือหลักฐานว่าทำไมกฎ "ใส่วงเล็บ
เสมอเมื่อผสม operator ต่างกลุ่ม" ถึงสำคัญ เพราะแม้แต่ compiler ยังต้องเตือน

> **แนวปฏิบัติที่ดีที่สุด**: เมื่อผสมตัวดำเนินการต่างกลุ่มกันในนิพจน์เดียว (โดยเฉพาะ Bitwise
> ปนกับ Arithmetic หรือ Logical ปนกับ Bitwise) **ให้ใส่วงเล็บเสมอ แม้จะรู้ลำดับความสำคัญ
> ดีอยู่แล้วก็ตาม** เพราะวงเล็บทำให้โค้ดสื่อเจตนาชัดเจนกับคนอ่านคนอื่น (รวมถึงตัวเราเองใน
> อนาคต) โดยไม่ต้องท่องตารางลำดับความสำคัญทุกครั้งที่อ่านโค้ด นี่คือหลักการเดียวกับที่บริษัท
> ซอฟต์แวร์ชั้นนำกำหนดไว้ใน Coding Standard ของตนแทบทุกแห่ง

---

## ข้อผิดพลาดที่พบบ่อย (Common Pitfalls)

1. **Integer Division ที่ไม่ได้ตั้งใจ** — `average = total_score / count;` เมื่อทั้งสองตัวเป็น
   `int` จะได้ผลลัพธ์เป็นจำนวนเต็มเสมอ ทำให้ค่าเฉลี่ยผิดเพี้ยน (เช่น 7/2 ได้ 3 ไม่ใช่ 3.5)
   ทางแก้คือ cast ตัวใดตัวหนึ่งเป็น `double` ก่อนหาร: `(double)total_score / count`

2. **เปรียบเทียบ signed กับ unsigned** — เมื่อเปรียบเทียบ `int` กับ `unsigned int` ในนิพจน์
   เดียวกัน compiler จะแปลง `int` เป็น `unsigned int` ก่อนเปรียบเทียบ ทำให้ค่าลบกลายเป็นเลข
   บวกมหาศาลโดยไม่รู้ตัว:

   ```c
   int a = -1;
   unsigned int b = 1;
   if (a < b) {   /* คาดหวังว่าจริง แต่จริงๆ ได้ "เท็จ"! */
       printf("a < b\n");
   } else {
       printf("a >= b (a=-1 ถูกแปลงเป็น unsigned กลายเป็นเลขบวกมหาศาล)\n");
   }
   ```

   `-Wextra` จะเตือนด้วย `-Wsign-compare` ว่า `comparison of integer expressions of
   different signedness` — **อย่าเพิกเฉยต่อ Warning นี้เด็ดขาด** ให้แปลงชนิดให้ตรงกันก่อน
   เปรียบเทียบเสมอ

3. **Unsigned Underflow ใน Loop** — คลาสสิกมาก: `for (unsigned int i = 5; i >= 0; i--)`
   เป็น Infinite Loop เพราะ `unsigned int` ไม่มีค่าติดลบ เมื่อ `i` เป็น 0 แล้วลดค่าอีกครั้ง
   จะ wrap กลับไปเป็นค่าสูงสุด (เช่น 4,294,967,295) แทนที่จะเป็น -1 ทำให้เงื่อนไข `i >= 0`
   เป็นจริงตลอดไป ทางแก้คือใช้ `int` แทน หรือปรับโครงสร้าง loop ให้ไม่ต้องเทียบกับ 0 โดยตรง

4. **ลืมใส่ `f` ต่อท้าย float literal** — `float x = 3.14;` ค่าคงที่ `3.14` เป็น `double`
   โดย default แล้วถูกลดขนาดลงมาเป็น `float` แบบเงียบๆ (ไม่มี Warning) ทำให้ความละเอียด
   คลาดเคลื่อนโดยไม่รู้ตัว ควรเขียน `float x = 3.14f;` เสมอเมื่อกำหนดค่าคงที่ให้ตัวแปร `float`

5. **แก้ไขตัวแปรเดียวกันหลายครั้งในนิพจน์เดียว** — เช่น `arr[i++] = i;` หรือ
   `printf("%d %d", i++, i++);` เป็น Undefined Behavior เพราะไม่มี Sequence Point กำหนด
   ลำดับการประเมินที่ชัดเจน ผลลัพธ์อาจต่างกันไปตาม compiler และ optimization level —
   ให้แยกเป็นคนละ statement เสมอเมื่อมีการแก้ไขค่าตัวแปรมากกว่าหนึ่งครั้ง

6. **สับสนระหว่าง `=` (assignment) กับ `==` (comparison)** — `if (flag = 1)` compile ผ่าน
   และมักทำให้เงื่อนไขเป็นจริงเสมอโดยไม่มี Error แต่ `-Wall` จะเตือนด้วย `-Wparentheses`
   วิธีป้องกันเพิ่มเติมที่บางคนใช้คือเขียนค่าคงที่ไว้ฝั่งซ้าย (`if (1 == flag)`) เพื่อให้พิมพ์
   ผิดเป็น `if (1 = flag)` ซึ่งจะเป็น **Compile Error ทันที** (เพราะกำหนดค่าให้ตัวเลขไม่ได้)

---

## แบบฝึกหัดท้ายบท

1. เขียนโปรแกรม `bmi_calculator.c` รับน้ำหนัก (kg) และส่วนสูง (m) ที่กำหนดในซอร์สโค้ดเป็น
   `double` แล้วคำนวณค่า BMI ตามสูตร `BMI = น้ำหนัก / (ส่วนสูง * ส่วนสูง)` พิมพ์ผลลัพธ์ทศนิยม
   2 ตำแหน่ง

2. เขียนโปรแกรมที่ประกาศตัวแปร `int a = 5, b = 10;` แล้วสลับค่าทั้งสองโดย**ไม่ใช้ตัวแปรที่
   สาม** (ใช้เทคนิคทางคณิตศาสตร์หรือ Bitwise XOR ก็ได้) พิมพ์ค่าก่อนและหลังสลับ

3. เขียนฟังก์ชัน `print_bits` (คล้ายในบทเรียน) แล้วทดลองพิมพ์ค่าบิตของเลข `uint8_t` หลายๆ
   ค่า (0, 1, 128, 255, 170) สังเกตรูปแบบบิตของแต่ละค่า

4. โปรแกรมด้านล่างมี Warning เรื่อง sign-compare จงแก้ไขให้คอมไพล์ผ่านโดยไม่มี Warning เลย
   แม้เปิด `-Wall -Wextra -Wpedantic` และยังคงพฤติกรรมที่ถูกต้องตามตรรกะเดิม (ตรวจสอบว่า
   `a` น้อยกว่า `b` จริง):

   ```c
   #include <stdio.h>
   int main(void) {
       int a = -1;
       unsigned int b = 1;
       if (a < b) {
           printf("a < b\n");
       } else {
           printf("a >= b\n");
       }
       return 0;
   }
   ```

5. เขียนโปรแกรมคำนวณดอกเบี้ยทบต้นอย่างง่าย 1 ปี ด้วยสูตร
   `เงินรวม = เงินต้น * (1 + อัตราดอกเบี้ย)` โดยกำหนดเงินต้น 10000.0 และอัตราดอกเบี้ย 0.03
   ให้ความสำคัญกับการใส่วงเล็บให้ operator precedence ถูกต้อง แล้วพิมพ์ผลลัพธ์ทศนิยม 2 ตำแหน่ง

6. เขียนโปรแกรมที่มี `#define FLAG_A (1 << 0)`, `FLAG_B (1 << 1)`, `FLAG_C (1 << 2)`
   สร้างตัวแปร `int settings = 0;` แล้วเปิดใช้งาน FLAG_A กับ FLAG_C ด้วย Bitwise OR
   จากนั้นตรวจสอบและพิมพ์ว่า FLAG_B เปิดอยู่หรือไม่ (ควรได้คำตอบว่า "ปิด")

### แนวทางเฉลยข้อ 1

```c
/* bmi_calculator.c */
#include <stdio.h>

int main(void) {
    double weight_kg = 65.0;
    double height_m  = 1.70;

    double bmi = weight_kg / (height_m * height_m);

    printf("น้ำหนัก %.1f kg ส่วนสูง %.2f m -> BMI = %.2f\n",
           weight_kg, height_m, bmi);

    return 0;
}
```

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 bmi_calculator.c -o bmi_calculator
./bmi_calculator
# น้ำหนัก 65.0 kg ส่วนสูง 1.70 m -> BMI = 22.49
```

จุดสำคัญของข้อนี้คือการใส่วงเล็บรอบ `height_m * height_m` ให้ชัดเจน แม้ `*` จะมีลำดับ
ความสำคัญสูงกว่า `/` อยู่แล้วตามธรรมชาติ (ซ้ายไปขวา `weight_kg / height_m * height_m` จะ
ได้ผลลัพธ์ผิดทันที เพราะจะกลายเป็น `(weight_kg / height_m) * height_m`) การเขียนวงเล็บ
ให้ชัดเจนแบบนี้คือการป้องกันบั๊กเชิงคณิตศาสตร์ที่พบบ่อยที่สุดอย่างหนึ่ง

### แนวทางเฉลยข้อ 4

```c
/* fixed_sign_compare.c */
#include <stdio.h>

int main(void) {
    int a = -1;
    unsigned int b = 1;

    /* วิธีที่ 1: แปลง b เป็น int ก่อนเปรียบเทียบ (ปลอดภัยเพราะรู้ว่า b เล็กพอ) */
    if (a < (int)b) {
        printf("a < b\n");
    } else {
        printf("a >= b\n");
    }

    return 0;
}
```

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 fixed_sign_compare.c -o fixed_sign_compare
./fixed_sign_compare
# a < b
```

วิธีแก้คือ**ตัดสินใจอย่างมีเจตนา**ว่าจะเปรียบเทียบในโดเมนไหน ในที่นี้เรารู้ว่าค่า `b` ที่ใช้
งานจริงจะไม่เกินขอบเขตของ `int` จึง cast `b` เป็น `int` ก่อนเปรียบเทียบได้อย่างปลอดภัย
(ถ้า `b` อาจมีค่าเกิน `INT_MAX` จริงๆ ในสถานการณ์อื่น จะต้องเปลี่ยนไป cast `a` เป็น
`unsigned int` แทน และต้องมั่นใจว่า `a` ไม่ติดลบก่อนเสมอ — การเลือกทิศทาง cast ต้องพิจารณา
ตามบริบทของข้อมูลจริงเสมอ ไม่มีคำตอบเดียวที่ถูกต้องตายตัว)

---

## สรุปท้ายบท

ใน Part นี้เราได้:

- รู้จักชนิดข้อมูลพื้นฐานของ C ทั้งหมด พร้อมขนาดและช่วงค่า และเข้าใจว่าทำไมขนาดจึงไม่ตายตัว
- ใช้ `sizeof` ตรวจสอบขนาดจริง และรู้จักชนิดข้อมูลขนาดคงที่จาก `<stdint.h>` สำหรับงานที่
  ต้องการความแน่นอนข้ามแพลตฟอร์ม
- เข้าใจกฎการตั้งชื่อตัวแปรทั้งที่เป็นกฎบังคับของภาษาและ Convention ของอุตสาหกรรม
- แยกแยะ Type Conversion แบบ Implicit (อัตโนมัติ) และ Explicit (Casting) พร้อมรู้จักกับดัก
  อย่าง Integer Division และ Wraparound
- ใช้ตัวดำเนินการทางคณิตศาสตร์ Assignment แบบย่อ Increment/Decrement เปรียบเทียบ ตรรกะ
  และ Bitwise ได้อย่างถูกต้องและปลอดภัย
- อ่านตาราง Operator Precedence และรู้ว่าเมื่อไรควรใส่วงเล็บเพื่อความชัดเจนของโค้ด

ใน **Part 3** เราจะเจาะลึกการรับส่งข้อมูลผ่าน `printf` และ `scanf` อย่างละเอียด ตั้งแต่
Format Specifier ทุกรูปแบบ ไปจนถึงปัญหา Buffer ค้างใน `stdin` ที่มือใหม่แทบทุกคนต้องเจอ

**ต่อไป:** [Part 3 — การรับส่งข้อมูลเชิงลึก (printf/scanf)](./part-003-io-printf-scanf.md)
