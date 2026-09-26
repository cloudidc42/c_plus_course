# Part 14: Preprocessor และ Macro (Step 105–112)

> Module B — C ระดับกลาง & โครงสร้างข้อมูล/อัลกอริทึม (Intermediate C & Data Structures/Algorithms) | Part 14 จาก 125
> Step ที่ครอบคลุมใน Part นี้: Step 105–112
> Part ก่อนหน้า: [Part 13 — File I/O ใน C](./part-013-file-io.md) | Part ถัดไป: [Part 15 — Bit Manipulation](./part-015-bit-manipulation.md)

## เป้าหมายการเรียนรู้ (Learning Objectives)

หลังจบ Part นี้ ผู้เรียนจะสามารถ:

1. อธิบายได้ว่า Preprocessor ทำงานอย่างไรในฐานะ "โปรแกรมแก้ไขข้อความ" ก่อนที่ Compiler จริงจะเริ่มทำงาน
2. ใช้ `#define` สร้างค่าคงที่ (Object-like Macro) ได้อย่างถูกต้อง และรู้ว่าเมื่อไหร่ควรใช้ `const`/`enum` แทน
3. เขียน Macro Function ได้อย่างปลอดภัย โดยเข้าใจอันตรายของการไม่ใส่วงเล็บและปัญหา Side-effect จากการส่ง expression ที่มี `++`/`--` เข้าไปเป็น argument
4. ใช้ Predefined Macro (`__FILE__`, `__LINE__`, `__func__`) และ Operator `#`/`##` เพื่อสร้าง Debug Macro ของตัวเอง
5. ควบคุมการคอมไพล์แบบมีเงื่อนไขด้วย `#ifdef`, `#ifndef`, `#if`, `#elif`, `#else`, `#endif`
6. เขียน Header Guard ได้ทั้งแบบดั้งเดิม (`#ifndef`) และแบบสมัยใหม่ (`#pragma once`) พร้อมอธิบายข้อดีข้อเสียของแต่ละแบบ
7. แยกความแตกต่างระหว่าง `#include "..."` กับ `#include <...>` และรู้ว่า Compiler ค้นหาไฟล์จากที่ไหนบ้าง
8. ใช้ `gcc -E` เพื่อ debug ปัญหาที่เกิดจาก Macro และเขียน X-Macro Pattern เบื้องต้นเพื่อลดโค้ดซ้ำซ้อน

---

## 14.1 Preprocessor คืออะไร และ `#define` สำหรับค่าคงที่ (Step 105)

ก่อนที่ Compiler จะแปลงโค้ด C เป็น Assembly (ตามที่เรียนไปใน Part 1) จะมีขั้นตอน **Preprocessing**
เกิดขึ้นก่อนเสมอ ขั้นตอนนี้ทำงานเหมือนโปรแกรม **แก้ไขข้อความ (Text Substitution)** ล้วนๆ —
มันไม่รู้จักไวยากรณ์ของภาษา C เลย ไม่รู้จัก type, ไม่รู้จัก scope มันแค่อ่านโค้ดต้นฉบับแล้ว
"แทนที่ข้อความ" ตามคำสั่งที่ขึ้นต้นด้วย `#` (เรียกว่า **Preprocessor Directive**) แล้วส่งผลลัพธ์
ที่เป็นข้อความล้วนต่อให้ Compiler จริงทำงานต่อ

รูปแบบพื้นฐานที่สุดของ `#define` คือการสร้าง **Object-like Macro** เพื่อใช้แทนค่าคงที่:

```c
/* constants_demo.c */
#include <stdio.h>

#define MAX_STUDENTS 40
#define PI 3.14159265358979
#define PROGRAM_NAME "Student Manager"

int main(void) {
    printf("โปรแกรม: %s\n", PROGRAM_NAME);
    printf("รับนักเรียนได้สูงสุด %d คน\n", MAX_STUDENTS);
    printf("พื้นที่วงกลมรัศมี 2: %.4f\n", PI * 2 * 2);
    return 0;
}
```

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 constants_demo.c -o constants_demo
./constants_demo
```

```
โปรแกรม: Student Manager
รับนักเรียนได้สูงสุด 40 คน
พื้นที่วงกลมรัศมี 2: 12.5664
```

สิ่งที่เกิดขึ้นจริงคือ Preprocessor จะเดินหาคำว่า `MAX_STUDENTS`, `PI`, `PROGRAM_NAME` ทุกจุดใน
ไฟล์ (ยกเว้นใน string literal อื่นและ comment) แล้ว **แทนที่ด้วยข้อความดิบๆ ตรงๆ** ก่อนที่
Compiler จะเริ่มอ่านโค้ดด้วยซ้ำ พูดง่ายๆ คือโค้ดข้างบนหลัง Preprocess จะกลายเป็นเหมือนเราพิมพ์
`40`, `3.14159265358979`, `"Student Manager"` ลงไปตรงๆ ทุกจุดที่เคยเขียนชื่อ Macro

### `#define` vs `const` vs `enum` — เลือกใช้อะไรดี?

Macro Constant ไม่ใช่ทางเลือกเดียวสำหรับค่าคงที่ใน C ยังมี `const` และ `enum` ที่ทำหน้าที่คล้ายกัน
แต่มีความแตกต่างสำคัญที่ต้องเข้าใจ:

| คุณสมบัติ | `#define` | `const int` | `enum` |
|---|---|---|---|
| มี Type ให้ Compiler ตรวจสอบหรือไม่ | ไม่มี (เป็นแค่ข้อความ) | มี | มี (เป็น `int`) |
| ปรากฏใน Debugger (เช่น GDB) หรือไม่ | ไม่ปรากฏ (ถูกแทนที่ไปแล้ว) | ปรากฏเป็นตัวแปรจริง | ปรากฏ (ส่วนใหญ่) |
| ใช้เป็นขนาด Array แบบ compile-time ได้หรือไม่ | ได้เสมอ | ได้เฉพาะที่ scope global/มี `static` ใน C ปกติ (ไม่ใช่ VLA) | ได้เสมอ |
| มี Scope (ขอบเขตการมองเห็น) หรือไม่ | ไม่มี — มองเห็นได้ทั้งไฟล์ตั้งแต่จุดที่ประกาศ | มี Scope ตามปกติของตัวแปร | มี Scope ตามปกติ |
| กินพื้นที่หน่วยความจำหรือไม่ | ไม่กิน (ถูกแทนที่ตอน compile) | กินพื้นที่จริง (เว้นแต่ optimize) | ไม่กิน (เป็นค่าคงที่ล้วน) |
| เหมาะกับ | ค่าคงที่ทั่วไป, ขนาด Array, Flag การคอมไพล์ | ค่าคงที่ที่มี Type ชัดเจน ต้องการ Type Safety | กลุ่มค่าคงที่ที่มีความหมายเป็นชุด (เช่นสถานะ) |

```c
#define MAX_SIZE 100          /* Macro: ไม่มี type, ไม่กิน memory */
const int max_size = 100;     /* const: มี type, Compiler ช่วยตรวจสอบได้ */
enum { MAX_ITEMS = 100 };     /* enum: ใช้เป็นค่าคงที่ระดับ compile-time ได้เหมือน macro */
```

**แนวทางปฏิบัติที่แนะนำ**: ถ้าค่าคงที่นั้นต้องใช้กำหนดขนาด Array ที่ scope local (`int arr[MAX_SIZE];`)
หรือใช้ตอน Preprocessing (เช่นใน `#if`) จำเป็นต้องใช้ `#define` หรือ `enum` เพราะ `const int`
ธรรมดาไม่ใช่ **Compile-time Constant Expression** อย่างแท้จริงใน C (ต่างจาก C++) แต่ถ้าต้องการ
Type Safety และอยากให้ Debugger มองเห็นค่าได้ ให้เลือก `const` หรือ `enum` แทน `#define` เสมอ
เมื่อทำได้

> ตามธรรมเนียมของภาษา C ชื่อ Macro Constant มักเขียนด้วย **ตัวพิมพ์ใหญ่ทั้งหมด** (UPPER_SNAKE_CASE)
> เพื่อให้ผู้อ่านโค้ดแยกออกทันทีว่านี่คือ Macro ไม่ใช่ตัวแปรหรือฟังก์ชันธรรมดา

---

## 14.2 Macro Function: พลังและกับดัก (Step 106)

`#define` ไม่ได้ใช้สร้างได้แค่ค่าคงที่ แต่ยังสร้าง **Macro Function** (หรือ Function-like Macro)
ที่รับ "พารามิเตอร์" ได้ด้วย — แต่ต้องเข้าใจให้ชัดว่ามันยังคงเป็นแค่ **Text Substitution**
ไม่ใช่ฟังก์ชันจริง และนี่คือที่มาของบั๊กที่แปลกประหลาดที่สุดหลายอย่างในภาษา C

```c
#define SQUARE(x) x * x
```

ดูเผินๆ เหมือนจะถูกต้อง ลองใช้งาน:

```c
#include <stdio.h>

#define SQUARE(x) x * x   /* อันตราย! ยังไม่ใส่วงเล็บให้ครบ */

int main(void) {
    int result = SQUARE(5);
    printf("5 ยกกำลังสอง = %d\n", result);   /* ได้ 25 ถูกต้อง... แต่บังเอิญ */

    int a = 2;
    int wrong = SQUARE(a + 3);   /* คาดหวัง (2+3)^2 = 25 */
    printf("ผลลัพธ์ที่คาดหวัง 25 แต่ได้จริง: %d\n", wrong);
    return 0;
}
```

หลังจาก Preprocessor แทนที่ข้อความแล้ว บรรทัด `SQUARE(a + 3)` จะกลายเป็น:

```c
int wrong = a + 3 * a + 3;   /* ตามลำดับ operator precedence: * ทำก่อน + */
```

เมื่อ `a = 2` จะได้ `2 + 3*2 + 3 = 2 + 6 + 3 = 11` ไม่ใช่ `25` ตามที่คาดหวังเลย! สาเหตุคือ
Preprocessor แค่ "แปะข้อความ" `a + 3` ลงไปแทนที่ `x` ทุกจุดแบบดิบๆ โดยไม่สนใจ Operator
Precedence ของนิพจน์ที่ส่งเข้ามาเลย

### วิธีแก้: ใส่วงเล็บให้ครบทุกจุด

กฎเหล็กของการเขียน Macro Function คือ **ต้องใส่วงเล็บ ครอบทั้งพารามิเตอร์แต่ละตัว และครอบ
นิพจน์ทั้งหมด**:

```c
#define SQUARE(x) ((x) * (x))
```

ทดสอบใหม่:

```c
#include <stdio.h>

#define SQUARE(x) ((x) * (x))

int main(void) {
    int a = 2;
    int correct = SQUARE(a + 3);   /* ((a + 3) * (a + 3)) = (5 * 5) = 25 */
    printf("ผลลัพธ์ที่ถูกต้อง: %d\n", correct);
    return 0;
}
```

```
ผลลัพธ์ที่ถูกต้อง: 25
```

หลัง Preprocess จะได้ `((a + 3) * (a + 3))` ซึ่งวงเล็บบังคับลำดับการคำนวณให้ถูกต้องเสมอ
ไม่ว่าผู้เรียกจะส่ง expression ที่ซับซ้อนแค่ไหนเข้ามา

### กับดักที่สอง: Side-effect จากการใช้ `++`/`--` เป็น Argument

แม้จะใส่วงเล็บครบถ้วนแล้ว Macro Function ก็ยังมีอันตรายอีกอย่างที่ **แก้ด้วยวงเล็บไม่ได้**
นั่นคือปัญหา **Side-effect** เมื่อ argument ที่ส่งเข้ามามีการเปลี่ยนค่าตัวแปร (เช่น `++x`, `x++`,
`x = f()`) เพราะพารามิเตอร์ตัวเดียวกันอาจถูก "แทนที่ซ้ำหลายครั้ง" ในนิพจน์ผลลัพธ์

```c
#include <stdio.h>

#define MAX(a, b) ((a) > (b) ? (a) : (b))

int main(void) {
    int x = 5;
    int y = 10;
    int result = MAX(x++, y);   /* ดูเหมือนจะปลอดภัยเพราะใส่วงเล็บครบแล้ว */
    printf("result = %d, x = %d\n", result, x);
    return 0;
}
```

หลัง Preprocess บรรทัดนี้จะกลายเป็น:

```c
int result = ((x++) > (y) ? (x++) : (y));
```

สังเกตว่า `x++` **ถูกแทนที่ซ้ำถึง 2 จุด** — จุดแรกใช้เปรียบเทียบ (`x++ > y` จะเพิ่มค่า `x` ครั้งที่ 1
แล้วเทียบด้วยค่าเดิม 5) เนื่องจาก `5 > 10` เป็นเท็จ โปรแกรมจะไปประเมินฝั่ง `(x++)` อีกครั้ง
(เพิ่มค่า `x` เป็นครั้งที่ 2!) ทำให้ `x` ถูกเพิ่มค่าไปทั้งหมด **2 ครั้ง** ทั้งที่โค้ดดูเหมือนเรียก
`MAX` แค่ครั้งเดียว และผลลัพธ์ `result` ก็อาจไม่ตรงกับที่ผู้เขียนตั้งใจ (ในทางเทคนิคกรณีนี้ยังถือ
เป็น Undefined Behavior เนื่องจากมีการแก้ไขค่า `x` มากกว่าหนึ่งครั้งโดยไม่มี Sequence Point คั่น
ตามมาตรฐาน C ระหว่าง sub-expression ทั้งสองของ `?:` ที่ถูกเลือกจริงเพียงฝั่งเดียว แต่ในทางปฏิบัติ
คอมไพเลอร์หลายตัวจะแสดงพฤติกรรมเพิ่มค่าซ้ำแบบนี้ให้เห็นได้จริง)

เทียบกับฟังก์ชันจริง (Real Function) ที่ไม่มีปัญหานี้เลย เพราะ argument จะถูกประเมินค่าเพียง
**ครั้งเดียว** ก่อนส่งเข้าฟังก์ชัน:

```c
#include <stdio.h>

static int max_int(int a, int b) {
    return (a > b) ? a : b;
}

int main(void) {
    int x = 5;
    int y = 10;
    int result = max_int(x++, y);   /* x++ ถูกประเมินครั้งเดียวแน่นอน */
    printf("result = %d, x = %d\n", result, x);   /* result = 10, x = 6 เสมอ */
    return 0;
}
```

> **กฎทองของ Macro Function**: (1) ใส่วงเล็บครอบทุกพารามิเตอร์และครอบทั้งนิพจน์เสมอ
> (2) **ห้ามส่ง expression ที่มี side-effect** (`++`, `--`, การเรียกฟังก์ชันที่เปลี่ยน state)
> เข้าไปเป็น argument ของ Macro ที่อาจใช้พารามิเตอร์นั้นซ้ำมากกว่า 1 ครั้งในตัวมันเอง
> (3) ถ้า logic ซับซ้อนหรือมีการใช้พารามิเตอร์ซ้ำ ให้เขียนเป็น **`static inline` function** แทน
> ตั้งแต่ C99 เป็นต้นไป เพราะได้ทั้งความปลอดภัยของ type checking และมักเร็วพอๆ กับ macro
> เนื่องจาก compiler จะ inline ให้อัตโนมัติเมื่อเหมาะสม

---

## 14.3 Predefined Macro และ Operator `#` / `##` (Step 107)

Compiler มี Macro ที่กำหนดไว้ให้ใช้งานล่วงหน้า (Predefined Macro) ซึ่งมีประโยชน์มากสำหรับ
การเขียน Log และ Debug Message:

| Macro | ความหมาย |
|---|---|
| `__FILE__` | ชื่อไฟล์ปัจจุบัน (string literal) |
| `__LINE__` | เลขบรรทัดปัจจุบัน (integer) |
| `__func__` | ชื่อฟังก์ชันปัจจุบัน (เป็น identifier ตามมาตรฐาน C99 ไม่ใช่ macro แต่ใช้งานคล้ายกัน) |
| `__DATE__` | วันที่ที่ compile (string เช่น `"Sep 26 2026"`) |
| `__TIME__` | เวลาที่ compile (string เช่น `"14:32:01"`) |
| `__STDC_VERSION__` | เลขมาตรฐาน C ที่ใช้ compile (เช่น `201710L` สำหรับ C17) |

```c
/* debug_log.c */
#include <stdio.h>

#define LOG_DEBUG(msg) \
    printf("[DEBUG] %s:%d (%s): %s\n", __FILE__, __LINE__, __func__, msg)

static void process_order(void) {
    LOG_DEBUG("เริ่มประมวลผลคำสั่งซื้อ");
}

int main(void) {
    LOG_DEBUG("โปรแกรมเริ่มทำงาน");
    process_order();
    printf("Compile เมื่อ: %s %s\n", __DATE__, __TIME__);
    return 0;
}
```

```
[DEBUG] debug_log.c:9 (main): โปรแกรมเริ่มทำงาน
[DEBUG] debug_log.c:6 (process_order): เริ่มประมวลผลคำสั่งซื้อ
Compile เมื่อ: Sep 26 2026 14:32:01
```

สังเกตว่า Macro `LOG_DEBUG` ใช้เครื่องหมาย `\` ท้ายบรรทัดเพื่อบอกว่า "นิยาม Macro นี้ยังไม่จบ
ต่อบรรทัดถัดไป" — เทคนิคนี้จำเป็นเมื่อ Macro มีเนื้อหายาวเกินหนึ่งบรรทัด

### Operator `#` (Stringizing) — แปลง Argument เป็น String

Operator `#` เมื่อวางไว้หน้าพารามิเตอร์ใน Macro Function จะแปลง **token** ที่ส่งเข้ามาให้กลายเป็น
string literal ทันที:

```c
#include <stdio.h>

#define PRINT_VAR(v) printf(#v " = %d\n", v)

int main(void) {
    int score = 95;
    PRINT_VAR(score);   /* ขยายเป็น printf("score" " = %d\n", score); */
    return 0;
}
```

```
score = 95
```

`#v` แปลง identifier `score` ให้กลายเป็น string literal `"score"` (และ C จะเชื่อม string literal
สองก้อนที่วางติดกัน `"score" " = %d\n"` เป็นก้อนเดียวโดยอัตโนมัติ — เป็นกฎของภาษา C เรียกว่า
**String Literal Concatenation**) เทคนิคนี้มีประโยชน์มากเวลาเขียน Macro สำหรับ debug ที่อยาก
พิมพ์ทั้งชื่อตัวแปรและค่าของมัน

### Operator `##` (Token Pasting) — เชื่อมสอง Token เข้าด้วยกัน

Operator `##` ใช้เชื่อม token สองตัวเข้าเป็น token เดียว ณ ตอน Preprocess ซึ่งเป็นหัวใจสำคัญของ
**X-Macro Pattern** ที่จะเรียนในหัวข้อสุดท้ายของ Part นี้:

```c
#include <stdio.h>

#define MAKE_GETTER(name) int get_##name(void) { return name##_value; }

static int width_value = 1920;
static int height_value = 1080;

MAKE_GETTER(width)    /* ขยายเป็น: int get_width(void) { return width_value; } */
MAKE_GETTER(height)   /* ขยายเป็น: int get_height(void) { return height_value; } */

int main(void) {
    printf("width = %d, height = %d\n", get_width(), get_height());
    return 0;
}
```

```
width = 1920, height = 1080
```

`name##_value` เชื่อมคำว่า `width` เข้ากับ `_value` กลายเป็น identifier ใหม่ `width_value`
ตั้งแต่ตอน Preprocess — เทคนิคนี้ช่วยลดโค้ดซ้ำซ้อนได้มาก แต่ก็ทำให้อ่านยากขึ้นถ้าใช้พร่ำเพรื่อ
จึงควรใช้เฉพาะกรณีที่คุ้มค่าจริงๆ เช่นการ generate ฟังก์ชันคล้ายกันจำนวนมาก

---

## 14.4 Conditional Compilation: `#ifdef` / `#ifndef` / `#if` / `#elif` (Step 108)

**Conditional Compilation** คือความสามารถของ Preprocessor ในการ "เลือก" ว่าจะรวมส่วนไหนของ
โค้ดเข้าไปคอมไพล์บ้าง โดยตัดสินใจจากเงื่อนไขที่ประเมินผลได้ตั้งแต่ตอน Preprocess (ก่อน compile
จริง) ทำให้เราเขียนโค้ดที่ต่างกันสำหรับแต่ละแพลตฟอร์ม, แต่ละโหมด (debug/release), หรือแต่ละ
config ได้ในไฟล์เดียวกัน

### `#ifdef` / `#ifndef` — ตรวจสอบว่า Macro ถูก define ไว้หรือไม่

```c
/* platform_demo.c */
#include <stdio.h>

#define ENABLE_LOGGING

int main(void) {
#ifdef ENABLE_LOGGING
    printf("[LOG] โปรแกรมเริ่มทำงานแล้ว\n");
#endif

#ifndef DISABLE_GREETING
    printf("สวัสดี ผู้ใช้งาน!\n");
#endif

    printf("โปรแกรมทำงานเสร็จสิ้น\n");
    return 0;
}
```

`#ifdef ENABLE_LOGGING` แปลว่า "ถ้า Macro ชื่อ `ENABLE_LOGGING` ถูก `#define` ไว้ (ไม่สนใจค่า
ที่ define แม้จะ define เป็นค่าว่างก็ยังถือว่า defined)" ส่วน `#ifndef` คือตรงข้าม — "ถ้า **ยังไม่**
ถูก define" เนื้อหาระหว่าง `#ifdef`/`#ifndef` กับ `#endif` จะถูกรวมเข้าไปคอมไพล์ก็ต่อเมื่อเงื่อนไข
เป็นจริงเท่านั้น ถ้าเงื่อนไขเป็นเท็จ โค้ดส่วนนั้นจะถูก **ตัดทิ้งไปเลยตั้งแต่ก่อน compile**
(ไม่ใช่แค่ข้ามการรัน เหมือน `if` ปกติ)

### กำหนดค่า Macro จาก Command Line โดยไม่แก้โค้ด

จุดที่มีประโยชน์มากคือสามารถกำหนดค่า Macro ผ่าน flag `-D` ของ `gcc` ได้โดยไม่ต้องแก้ไฟล์
source เลย:

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 -DENABLE_LOGGING platform_demo.c -o platform_demo
```

เทคนิคนี้ใช้บ่อยมากในระบบ Build จริง (จะเรียนละเอียดใน Part 18 เรื่อง Makefile) เพื่อสลับโหมด
Debug/Release หรือเปิด/ปิดฟีเจอร์โดยไม่ต้องแก้โค้ด

### `#if` / `#elif` / `#else` — ประเมินนิพจน์ค่าคงที่

`#if` ทรงพลังกว่า `#ifdef` เพราะประเมิน **นิพจน์ค่าคงที่จำนวนเต็ม (Integer Constant Expression)**
ได้ ไม่ใช่แค่เช็คว่า define หรือไม่:

```c
/* version_check.c */
#include <stdio.h>

#define APP_VERSION 3

int main(void) {
#if APP_VERSION >= 3
    printf("ใช้ฟีเจอร์เวอร์ชันใหม่ (v3 ขึ้นไป)\n");
#elif APP_VERSION == 2
    printf("ใช้ฟีเจอร์เวอร์ชัน 2\n");
#else
    printf("ใช้ฟีเจอร์เวอร์ชันเก่า (v1)\n");
#endif
    return 0;
}
```

ในนิพจน์ของ `#if` เราสามารถใช้ operator เชิงตรรกะ (`&&`, `||`, `!`), operator เปรียบเทียบ,
และ operator พิเศษ `defined(NAME)` ได้ (ซึ่งเทียบเท่าการเขียน `#ifdef NAME` แต่ใช้รวมกับ
เงื่อนไขอื่นในนิพจน์เดียวกันได้):

```c
#if defined(__linux__) && !defined(ANDROID)
    /* โค้ดเฉพาะ Linux แท้ๆ (ไม่ใช่ Android) */
#elif defined(_WIN32)
    /* โค้ดเฉพาะ Windows */
#elif defined(__APPLE__)
    /* โค้ดเฉพาะ macOS */
#else
    #error "ไม่รองรับแพลตฟอร์มนี้"
#endif
```

`#error` เป็นอีกหนึ่ง Directive ที่มีประโยชน์ — เมื่อ Preprocessor เจอ `#error` จะหยุดการคอมไพล์
ทันทีพร้อมแสดงข้อความที่กำหนด เหมาะกับการป้องกันไม่ให้โค้ดถูกคอมไพล์ในสภาพแวดล้อมที่ไม่รองรับ

| Directive | ใช้ตรวจสอบ |
|---|---|
| `#ifdef NAME` | Macro `NAME` defined อยู่หรือไม่ |
| `#ifndef NAME` | Macro `NAME` **ไม่** defined อยู่หรือไม่ |
| `#if EXPR` | นิพจน์ค่าคงที่ `EXPR` เป็นจริง (ไม่เท่ากับ 0) หรือไม่ |
| `#elif EXPR` | เงื่อนไขถัดไป ถ้า `#if`/`#elif` ก่อนหน้าเป็นเท็จทั้งหมด |
| `#else` | กรณีอื่นๆ ที่เหลือ |
| `#endif` | ปิดบล็อกเงื่อนไข (บังคับมีคู่กับ `#if`/`#ifdef`/`#ifndef` เสมอ) |

---

## 14.5 Header Guard: `#ifndef` แบบดั้งเดิม vs `#pragma once` (Step 109)

ปัญหาคลาสสิกที่เกิดขึ้นเมื่อโปรเจกต์มีหลายไฟล์ (จะเรียนละเอียดเรื่องการแบ่งไฟล์ header/source
ใน Part 17) คือการที่ header ไฟล์เดียวกันถูก `#include` เข้ามาซ้ำมากกว่าหนึ่งครั้งในหน่วยคอมไพล์
เดียวกัน (เช่น ไฟล์ A include ทั้ง B และ C แต่ทั้ง B และ C ต่าง include D เหมือนกัน) ทำให้
เนื้อหาของ D ถูกแปะซ้ำสองครั้ง เกิด Error แบบ "redefinition"

### วิธีที่ 1: `#ifndef` Header Guard (มาตรฐาน ใช้ได้ทุก Compiler)

```c
/* shapes.h */
#ifndef SHAPES_H
#define SHAPES_H

typedef struct {
    double width;
    double height;
} Rectangle;

double rectangle_area(Rectangle r);

#endif /* SHAPES_H */
```

หลักการทำงาน: ครั้งแรกที่ไฟล์นี้ถูก include, Macro `SHAPES_H` ยังไม่ถูก define จึงทำให้เงื่อนไข
`#ifndef SHAPES_H` เป็นจริง เนื้อหาทั้งหมดจึงถูกรวมเข้าไป **และ** `SHAPES_H` ถูก define ไว้ทันที
ด้วยบรรทัด `#define SHAPES_H` ครั้งต่อไปที่ไฟล์เดียวกันถูก include ซ้ำ (ไม่ว่าจะจากที่ไหน)
เงื่อนไข `#ifndef SHAPES_H` จะเป็นเท็จ (เพราะ define ไปแล้ว) ทำให้เนื้อหาทั้งหมดถูกข้ามไป —
ป้องกันการประกาศซ้ำได้อย่างสมบูรณ์

> **ธรรมเนียมการตั้งชื่อ Macro guard**: มักตั้งตามชื่อไฟล์แบบตัวพิมพ์ใหญ่ทั้งหมด คั่นด้วย
> underscore เช่นไฟล์ `shapes.h` → `SHAPES_H`, ไฟล์ `linked_list.h` → `LINKED_LIST_H`
> ในโปรเจกต์ใหญ่อาจเติม prefix ชื่อโปรเจกต์เพื่อลดโอกาสชนกัน เช่น `MYAPP_SHAPES_H`

### วิธีที่ 2: `#pragma once` (สมัยใหม่ ไม่ใช่มาตรฐาน ISO C แต่รองรับแทบทุก Compiler)

```c
/* shapes_modern.h */
#pragma once

typedef struct {
    double width;
    double height;
} Rectangle;

double rectangle_area(Rectangle r);
```

`#pragma once` สั่ง Compiler ตรงๆ ว่า "ไฟล์นี้ให้ include เข้าไปแค่ครั้งเดียวเท่านั้นต่อหนึ่ง
หน่วยคอมไพล์" โดย Compiler จะจำ "ไฟล์" นี้จาก path จริงบน disk แทนที่จะอาศัย Macro name

| หัวข้อเปรียบเทียบ | `#ifndef` Guard | `#pragma once` |
|---|---|---|
| เป็นส่วนหนึ่งของมาตรฐาน ISO C หรือไม่ | ใช่ (มาตรฐานเต็มรูปแบบ) | ไม่ใช่ (เป็น Compiler extension แต่รองรับกว้างขวางมาก) |
| ความเสี่ยงชื่อ Macro ชนกัน | มี (ถ้าตั้งชื่อ guard ซ้ำกันโดยไม่ตั้งใจ) | ไม่มี — อิงจากไฟล์จริง ไม่ใช่ชื่อ Macro |
| ความเร็วในการคอมไพล์ | เปิดไฟล์และอ่านทุกครั้งแต่ข้ามเนื้อหาด้วย `#ifndef` | บาง Compiler ข้ามการเปิดไฟล์ซ้ำได้เลย เร็วกว่าเล็กน้อยในโปรเจกต์ใหญ่ |
| ปัญหา Edge case | แทบไม่มี ถ้าตั้งชื่อไม่ชนกัน | อาจมีปัญหากับไฟล์ที่ถูก mount/copy เป็นหลาย path ในระบบไฟล์แปลกๆ (พบน้อยมาก) |
| ความนิยมในโปรเจกต์จริง | ยังใช้กันแพร่หลายมาก โดยเฉพาะโปรเจกต์ที่เน้น portability สูงสุด | ได้รับความนิยมมากขึ้นเรื่อยๆ ในโปรเจกต์สมัยใหม่เพราะเขียนสั้นกว่า |

หลักสูตรนี้จะสอนทั้งสองแบบ และในโปรเจกต์ตั้งแต่ Part 17 เป็นต้นไปจะใช้ `#ifndef` guard เป็นหลัก
เพื่อความเข้ากันได้สูงสุดกับ Compiler ทุกตัว แต่ผู้เรียนสามารถเลือกใช้ `#pragma once` ในโปรเจกต์
ส่วนตัวได้ตามความสะดวก เพราะ `gcc`, `clang` และ `MSVC` รองรับทั้งหมด

---

## 14.6 `#include "..."` กับ `#include <...>` ต่างกันอย่างไร (Step 110)

`#include` เป็น Directive ที่บอก Preprocessor ให้นำเนื้อหาทั้งหมดของไฟล์ที่ระบุมา "แปะ" ณ จุดนั้น
แบบตรงๆ (เหมือนก็อปปี้-วางเนื้อหาไฟล์นั้นเข้ามาทั้งดุ้น) แต่มีความแตกต่างสำคัญระหว่างการเขียน
ด้วยเครื่องหมาย `"..."` และ `<...>` อยู่ที่ **ลำดับการค้นหาไฟล์**

```c
#include <stdio.h>     /* ค้นหาจาก System Include Path ก่อน */
#include "myheader.h"  /* ค้นหาจาก Local/Project Path ก่อน */
```

### ลำดับการค้นหาของ `<...>`

ใช้สำหรับ **Header มาตรฐานหรือ Header ของ Library ภายนอก** ที่ไม่ได้เป็นส่วนหนึ่งของโปรเจกต์
เรา Compiler จะค้นหาจาก **System Include Path** เท่านั้น (เช่น `/usr/include`,
`/usr/lib/gcc/.../include` บน Linux) โดยไม่สนใจโฟลเดอร์ของไฟล์ปัจจุบันเลย

### ลำดับการค้นหาของ `"..."`

ใช้สำหรับ **Header ที่เราเขียนเองในโปรเจกต์** Compiler จะค้นหาตามลำดับนี้:

1. ค้นหาใน **โฟลเดอร์เดียวกับไฟล์ที่กำลัง include** (ไฟล์ที่มีบรรทัด `#include` นี้อยู่) ก่อน
2. ถ้าไม่เจอ จะค้นหาต่อในโฟลเดอร์ที่ระบุด้วย flag `-I` ตอนคอมไพล์
3. ถ้ายังไม่เจออีก จะ fallback ไปค้นหาใน System Include Path เหมือน `<...>` เป็นลำดับสุดท้าย

ตัวอย่างโครงสร้างโปรเจกต์:

```
myproject/
├── main.c
├── include/
│   └── mathutils.h
└── src/
    └── mathutils.c
```

```c
/* main.c */
#include <stdio.h>          /* Header มาตรฐาน */
#include "include/mathutils.h"   /* Header ของเราเอง — ใช้ path สัมพัทธ์ได้ */
```

หรือถ้าอยากอ้างอิงแบบสั้นโดยไม่ต้องพิมพ์ path เต็มทุกครั้ง ให้ใช้ flag `-I` บอก compiler ว่า
มีโฟลเดอร์ header เพิ่มเติมอยู่ที่ไหน:

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 -Iinclude main.c src/mathutils.c -o myprogram
```

จากนั้นใน `main.c` จะเขียนสั้นลงเหลือแค่ `#include "mathutils.h"` ได้เลย เพราะ `-Iinclude`
บอกให้ compiler มองหาไฟล์ในโฟลเดอร์ `include/` เพิ่มด้วย

| | `#include <...>` | `#include "..."` |
|---|---|---|
| จุดเริ่มค้นหา | System Include Path โดยตรง | โฟลเดอร์ของไฟล์ปัจจุบัน ก่อน |
| ใช้กับ | Standard Library, Library ภายนอกที่ติดตั้งในระบบ | Header ที่เขียนเองในโปรเจกต์ |
| ตัวอย่าง | `<stdio.h>`, `<stdlib.h>`, `<pthread.h>` | `"config.h"`, `"linked_list.h"` |

> **ข้อควรระวัง**: ทั้ง `<...>` และ `"..."` เป็นแค่ "ธรรมเนียมและลำดับการค้นหาที่แนะนำ" การใช้
> `"..."` กับ System Header ก็ยังคอมไพล์ผ่านได้ในทางเทคนิค (เพราะสุดท้ายจะ fallback ไปค้นหาที่
> System Path เหมือนกัน) แต่ **ไม่ควรทำ** เพราะทำให้โค้ดอ่านยากและสับสนว่าไฟล์ไหนเป็นของเราเอง

---

## 14.7 Debug ปัญหาจาก Macro ด้วย `gcc -E` (Step 111)

เพราะ Macro เป็นเพียง Text Substitution ที่ทำงาน "มองไม่เห็น" ก่อนที่ Compiler จริงจะเริ่มทำงาน
เวลาเกิด Error หรือพฤติกรรมแปลกๆ ที่เกี่ยวกับ Macro ข้อความ error ที่ได้มักจะสับสนมาก เพราะ
Compiler รายงาน Error โดยอ้างอิงจากโค้ด **หลังขยาย Macro แล้ว** ไม่ใช่โค้ดต้นฉบับที่เราเขียน

เครื่องมือสำคัญที่สุดสำหรับ debug ปัญหานี้คือ flag `-E` ของ `gcc` ที่เราเคยเห็นสั้นๆ ใน Part 1
แล้ว — มันสั่งให้ `gcc` **หยุดแค่ขั้นตอน Preprocessing** แล้วพิมพ์ผลลัพธ์ที่ขยาย Macro ครบแล้ว
ออกมาให้ดู โดยไม่ทำการ compile ต่อ

```c
/* macro_bug.c */
#include <stdio.h>

#define AREA(w, h) w * h
#define DOUBLE(x) x + x

int main(void) {
    int a = AREA(2 + 3, 4);      /* คาดหวัง 5*4=20 แต่จะพังเพราะไม่ใส่วงเล็บ */
    int b = DOUBLE(3) * 2;       /* คาดหวัง (3+3)*2=12 แต่จะพังด้วยเหตุผลเดียวกัน */
    printf("a = %d, b = %d\n", a, b);
    return 0;
}
```

ลอง preprocess ดูเพื่อดูว่า Macro ถูกขยายเป็นอะไรกันแน่:

```bash
gcc -E macro_bug.c
```

ส่วนที่เกี่ยวข้องในผลลัพธ์ (ตัดส่วนเนื้อหาของ `stdio.h` ที่ยาวมากออกไป) จะแสดงบรรทัด `main`
ที่ถูกขยายแล้ว:

```c
int main(void) {
    int a = 2 + 3 * 4;      /* AREA(2+3, 4) ถูกแทนที่แบบดิบๆ ตรงๆ */
    int b = 3 + 3 * 2;      /* DOUBLE(3) * 2 ถูกแทนที่แบบดิบๆ */
    printf("a = %d, b = %d\n", a, b);
    return 0;
}
```

พอเห็นแบบนี้ก็เข้าใจทันทีว่าทำไมค่าที่ได้ถึงผิด: `2 + 3 * 4 = 14` (ไม่ใช่ 20) และ
`3 + 3 * 2 = 9` (ไม่ใช่ 12) — เพราะ operator precedence ของ `*` มาก่อน `+` เสมอ ทั้งที่ตอนเขียน
Macro เราตั้งใจให้ `w`, `h`, `x` เป็น "หน่วยเดียวที่แยกจากกันไม่ได้" แต่ Text Substitution ไม่รู้
เจตนานั้นเลย

วิธีแก้คือใส่วงเล็บให้ครบตามกฎที่เรียนไปในหัวข้อ 14.2:

```c
#define AREA(w, h) ((w) * (h))
#define DOUBLE(x) ((x) + (x))
```

```bash
gcc -E -DAREA_FIXED macro_bug_fixed.c | tail -8
```

> **เคล็ดลับการใช้ `gcc -E` ในชีวิตจริง**: เวลาเจอ Error ที่ชี้ไปยังบรรทัดที่มีการเรียก Macro
> ซับซ้อน หรือ Error message ดูไม่สัมพันธ์กับโค้ดที่เราเห็นเลย ให้รัน `gcc -E ไฟล์.c -o ไฟล์.i`
> แล้วเปิดไฟล์ `.i` ขึ้นมาดูตรงบรรทัดที่เกี่ยวข้อง (ใช้ `grep` หรือค้นหาชื่อฟังก์ชัน/ตัวแปรที่
> เกี่ยวข้องเพื่อข้ามเนื้อหาของ Header มาตรฐานที่ยาวมาก) จะเห็นโค้ด "ตัวจริง" ที่ Compiler
> กำลังพยายามคอมไพล์อยู่ทันที

---

## 14.8 X-Macro Pattern เบื้องต้น (Step 112)

**X-Macro** เป็นเทคนิคขั้นสูงที่ใช้ Preprocessor เพื่อ **สร้างข้อมูลหลายชุดจากแหล่งความจริง
เดียว (Single Source of Truth)** ลดปัญหาการต้องแก้ไขข้อมูลชุดเดียวกันในหลายที่ (ซึ่งเสี่ยงลืมแก้
บางจุดจนข้อมูลไม่ตรงกัน)

สมมติว่าต้องการเขียนโปรแกรมที่มี Error Code หลายแบบ และในหลายจุดของโค้ดต้องการทั้ง
(1) enum ของ error code, (2) array ของข้อความอธิบาย error, และ (3) ฟังก์ชันแปลง error code
เป็น string — ปกติต้องเขียนรายการ error ซ้ำถึง 3 ที่ ถ้าเพิ่ม error ใหม่ต้องจำให้แก้ครบทุกที่

### ขั้นตอนที่ 1: นิยาม "รายการข้อมูล" ไว้ที่เดียวด้วย Macro

```c
/* error_codes.h */
#ifndef ERROR_CODES_H
#define ERROR_CODES_H

/* X-Macro table: แต่ละแถวคือ (ชื่อ enum, ข้อความอธิบาย) */
#define ERROR_CODE_TABLE \
    X(ERR_OK,             "สำเร็จ ไม่มีข้อผิดพลาด") \
    X(ERR_FILE_NOT_FOUND, "ไม่พบไฟล์ที่ระบุ") \
    X(ERR_OUT_OF_MEMORY,  "หน่วยความจำไม่เพียงพอ") \
    X(ERR_INVALID_INPUT,  "ข้อมูลนำเข้าไม่ถูกต้อง")

#endif /* ERROR_CODES_H */
```

### ขั้นตอนที่ 2: ใช้ตาราง X-Macro เดิม สร้างทั้ง `enum` และ `array` โดยไม่พิมพ์ชื่อ error ซ้ำเลย

```c
/* error_demo.c */
#include <stdio.h>
#include "error_codes.h"

/* --- สร้าง enum จากตาราง X-Macro --- */
typedef enum {
#define X(name, text) name,
    ERROR_CODE_TABLE
#undef X
    ERR_COUNT   /* จำนวน error ทั้งหมด (trick ที่ใช้บ่อย: ใส่เป็นตัวสุดท้ายเสมอ) */
} ErrorCode;

/* --- สร้าง array ของข้อความจากตาราง X-Macro เดิม --- */
static const char *const error_messages[ERR_COUNT] = {
#define X(name, text) [name] = text,
    ERROR_CODE_TABLE
#undef X
};

const char *error_to_string(ErrorCode code) {
    if (code < 0 || code >= ERR_COUNT) {
        return "รหัส error ไม่รู้จัก";
    }
    return error_messages[code];
}

int main(void) {
    for (int i = 0; i < ERR_COUNT; i++) {
        printf("[%d] %s\n", i, error_to_string((ErrorCode)i));
    }
    return 0;
}
```

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 error_demo.c -o error_demo
./error_demo
```

```
[0] สำเร็จ ไม่มีข้อผิดพลาด
[1] ไม่พบไฟล์ที่ระบุ
[2] หน่วยความจำไม่เพียงพอ
[3] ข้อมูลนำเข้าไม่ถูกต้อง
```

### เกิดอะไรขึ้นจริงๆ

`ERROR_CODE_TABLE` คือ Macro ที่เก็บ "รายการ" เอาไว้ โดยแต่ละแถวเรียก `X(...)` ซึ่งเรายังไม่ได้
นิยามว่า `X` คืออะไร — เราจะนิยาม `X` ใหม่ทุกครั้งก่อนใช้ `ERROR_CODE_TABLE` ตามจุดประสงค์ที่
ต้องการ (สร้าง enum ก็นิยาม `X` แบบหนึ่ง, สร้าง array ก็นิยาม `X` อีกแบบหนึ่ง) แล้วใช้
`#undef X` ทันทีหลังใช้เสร็จเพื่อไม่ให้ชื่อ `X` ไปรบกวนส่วนอื่นของโค้ด

ประโยชน์ชัดเจนคือ **ถ้าต้องการเพิ่ม Error Code ใหม่ แก้แค่จุดเดียว** ในไฟล์ `error_codes.h`
ทั้ง `enum ErrorCode` และ `error_messages[]` จะอัปเดตตามโดยอัตโนมัติเมื่อ compile ใหม่ ไม่มี
ความเสี่ยงที่ enum กับ string message จะไม่ตรงกันอีกต่อไป — เทคนิคนี้ใช้จริงในโปรเจกต์ขนาดใหญ่
หลายแห่ง เช่นการ generate opcode table ของ Virtual Machine หรือ state table ของ Protocol

> X-Macro เป็นเทคนิคที่ทรงพลังแต่ก็ทำให้โค้ดอ่านยากขึ้นสำหรับคนที่ไม่คุ้นเคย จึงควรใช้เฉพาะ
> กรณีที่มีข้อมูลชุดเดียวกันที่ต้องนำไปสร้างเป็นหลายรูปแบบจริงๆ (เช่น enum + string + จำนวน)
> ไม่ควรใช้พร่ำเพรื่อในโค้ดทั่วไปที่มีแค่รายการเดียว

---

## ข้อผิดพลาดที่พบบ่อย (Common Pitfalls)

1. **ลืมใส่วงเล็บใน Macro Function** — `#define SQUARE(x) x*x` ดูใช้ได้ตอนทดสอบง่ายๆ แต่พังทันที
   เมื่อส่ง expression ที่มี operator บวก/ลบเข้ามา ให้ใส่วงเล็บครอบทั้งพารามิเตอร์แต่ละตัวและ
   ครอบทั้งนิพจน์เสมอ: `#define SQUARE(x) ((x) * (x))`
2. **ส่ง Expression ที่มี Side-effect เข้า Macro** — เช่น `MAX(i++, j)` เมื่อ Macro ใช้พารามิเตอร์
   ซ้ำมากกว่าหนึ่งครั้งในตัวมันเอง ค่าที่มี side-effect จะถูกประเมินซ้ำ ทำให้พฤติกรรมผิดเพี้ยนหรือ
   เป็น Undefined Behavior ให้เขียนเป็น `static inline` function แทนเมื่อ logic ซับซ้อน
3. **ลืม `#undef` ชื่อ Macro ชั่วคราวหลังใช้ X-Macro** — โดยเฉพาะชื่อสั้นๆ อย่าง `X` ที่อาจไปชนกับ
   Macro อื่นในไฟล์เดียวกันถ้าไม่ `#undef` ทันทีหลังใช้งานเสร็จ
4. **ลืมปิด `#endif`** หรือปิดผิดคู่กับ `#if`/`#ifdef`/`#ifndef` — ทำให้ Compiler อ่านโค้ดผิดเพี้ยน
   ไปทั้งไฟล์ ควรเขียน comment ท้าย `#endif` บอกว่าปิดคู่กับอะไร เช่น `#endif /* SHAPES_H */`
   โดยเฉพาะเมื่อมี `#if` ซ้อนกันหลายชั้น
5. **ตั้งชื่อ Header Guard ซ้ำกันโดยไม่ตั้งใจ** — ถ้าสองไฟล์ header ต่างกันแต่ใช้ Macro guard
   ชื่อเดียวกัน (เช่นทั้งคู่ใช้ `UTILS_H`) ไฟล์ที่สองจะไม่ถูกรวมเข้ามาเลยเมื่อ include ทั้งคู่
   ในไฟล์เดียวกัน เพราะ Preprocessor คิดว่า guard ถูก define ไปแล้ว ทำให้เกิด error แปลกๆ
   อย่าง "undeclared function/type" ที่ดูไม่เกี่ยวกับ Header Guard เลย
6. **ใช้ Macro แทน `const`/`enum`/`inline function` ทั้งที่ไม่จำเป็น** — Macro ไม่มี Type Checking
   ไม่มี Scope และไม่ปรากฏใน Debugger ทำให้ debug ยากกว่ามาก ควรใช้ Macro เฉพาะกรณีที่จำเป็น
   จริงๆ (เช่นค่าคงที่ระดับ compile-time, conditional compilation, X-Macro)

---

## แบบฝึกหัดท้ายบท

1. เขียน Macro Function ชื่อ `CUBE(x)` ที่คำนวณค่ายกกำลังสาม โดยต้องใส่วงเล็บให้ปลอดภัยตามกฎ
   ที่เรียนไป แล้วทดสอบด้วยค่า `CUBE(2 + 1)` ต้องได้ผลลัพธ์ `27` ไม่ใช่ค่าอื่น
2. เขียนโปรแกรมที่มี Macro `#define IS_EVEN(n) ((n) % 2 == 0)` และใช้ตรวจสอบเลขคู่/คี่ของตัวเลข
   ที่ผู้ใช้ป้อนเข้ามา 5 ค่า
3. เขียน Header Guard ให้ไฟล์ `point.h` ที่มี `struct Point { double x, y; };` โดยใช้ทั้งสองแบบ
   คือแบบ `#ifndef` และแบบ `#pragma once` (ทำเป็นไฟล์แยกกัน) แล้วอธิบายว่าทั้งสองแบบทำงาน
   ต่างกันอย่างไร
4. ใช้ `gcc -E` ตรวจสอบว่า Macro ต่อไปนี้ขยายเป็นอะไร แล้วอธิบายว่าทำไมถึงให้ผลลัพธ์ผิดพลาด:
   `#define TRIPLE(x) x + x + x` เมื่อเรียก `int r = TRIPLE(2) * 5;`
5. เขียนโปรแกรมที่ใช้ `#if defined(__linux__)`, `#elif defined(_WIN32)`, `#else` เพื่อพิมพ์ข้อความ
   บอกว่ากำลังคอมไพล์อยู่บนระบบปฏิบัติการอะไร
6. ขยายตัวอย่าง X-Macro ในหัวข้อ 14.8 ให้เพิ่ม Error Code ใหม่ชื่อ `ERR_PERMISSION_DENIED`
   พร้อมข้อความอธิบาย แล้วยืนยันว่าโปรแกรมยังคอมไพล์และรันได้ถูกต้องโดยแก้แค่จุดเดียว

### แนวทางเฉลยข้อ 1

```c
#include <stdio.h>

#define CUBE(x) ((x) * (x) * (x))

int main(void) {
    int result = CUBE(2 + 1);   /* ต้องได้ ((2+1)*(2+1)*(2+1)) = (3*3*3) = 27 */
    printf("CUBE(2 + 1) = %d\n", result);
    return 0;
}
```

```
CUBE(2 + 1) = 27
```

จุดสำคัญคือถ้าเขียน `#define CUBE(x) x * x * x` โดยไม่ใส่วงเล็บ ผลลัพธ์ของ `CUBE(2 + 1)` จะ
ขยายเป็น `2 + 1 * 2 + 1 * 2 + 1` ซึ่งตาม operator precedence จะคำนวณเป็น `2 + 2 + 2 + 1 = 7`
ผิดไปจากที่ตั้งใจอย่างสิ้นเชิง

### แนวทางเฉลยข้อ 4

```bash
gcc -E triple_bug.c
```

ผลลัพธ์ส่วนที่เกี่ยวข้องจะแสดง:

```c
int r = 2 + 2 + 2 * 5;
```

เพราะ `TRIPLE(2)` ถูกแทนที่แบบตรงๆ เป็น `2 + 2 + 2` (ไม่มีวงเล็บ) แล้วตามด้วย `* 5` จากโค้ด
เดิม ทำให้นิพจน์ทั้งหมดกลายเป็น `2 + 2 + 2 * 5` ซึ่งตาม operator precedence คูณจะถูกคำนวณก่อน
บวก ได้ `2 + 2 + 10 = 14` ไม่ใช่ `(2+2+2) * 5 = 30` ตามที่ผู้เขียนคาดหวัง วิธีแก้คือต้องเขียน
`#define TRIPLE(x) ((x) + (x) + (x))` เพื่อบังคับให้ทั้งก้อนถูกคำนวณรวมกันก่อนคูณด้วย `5`
เสมอไม่ว่าจะถูกใช้ในบริบทไหนก็ตาม

---

## สรุปท้ายบท

ใน Part นี้เราได้:

- เข้าใจว่า Preprocessor เป็นเพียงขั้นตอน Text Substitution ที่ทำงานก่อน Compiler จริง
- สร้างค่าคงที่ด้วย `#define` และรู้ว่าเมื่อไหร่ควรเลือก `const`/`enum` แทน
- เขียน Macro Function อย่างปลอดภัย เข้าใจอันตรายของการไม่ใส่วงเล็บ และปัญหา Side-effect
  จากการใช้ `++`/`--` เป็น argument
- ใช้ Predefined Macro (`__FILE__`, `__LINE__`, `__func__`) และ Operator `#`/`##` สร้าง Debug
  Macro และ Code Generation Pattern ของตัวเอง
- ควบคุมการคอมไพล์แบบมีเงื่อนไขด้วย `#ifdef`/`#ifndef`/`#if`/`#elif`/`#else`/`#endif`
- เขียน Header Guard ได้ทั้งแบบ `#ifndef` และ `#pragma once` พร้อมรู้ข้อดีข้อเสียของแต่ละแบบ
- แยกความแตกต่างระหว่าง `#include "..."` กับ `#include <...>`
- ใช้ `gcc -E` เพื่อ debug ปัญหาที่เกิดจาก Macro ได้อย่างมั่นใจ
- เขียน X-Macro Pattern เบื้องต้นเพื่อลดโค้ดซ้ำซ้อนจากข้อมูลชุดเดียวกัน

Preprocessor เป็นเครื่องมือที่ทรงพลังแต่ก็อันตรายถ้าใช้ไม่ระวัง — หลักการสำคัญที่สุดที่ควรจำไว้
คือ Macro เป็นแค่ "การแทนที่ข้อความ" ไม่ใช่ฟังก์ชันจริง ทุกครั้งที่สงสัยพฤติกรรมของ Macro
ให้กลับมาที่ `gcc -E` เสมอ ใน **Part 15** เราจะย้ายไปเรียนรู้ **Bit Manipulation** ซึ่งเป็นทักษะ
สำคัญสำหรับงาน Systems Programming, Embedded Systems และการเขียนโปรแกรมที่ต้องการ
ประสิทธิภาพสูงสุด

**ต่อไป:** [Part 15 — Bit Manipulation](./part-015-bit-manipulation.md)
