# Part 44: ฟังก์ชันใน C++ (Step 345–352)

> Module D — เริ่มต้น C++ และ OOP | Part 44 จาก 125
> Step ที่ครอบคลุมใน Part นี้: Step 345–352
> Part ก่อนหน้า: [Part 43 — Reference และ const](./part-043-references-const.md) | Part ถัดไป: [Part 45 — เริ่มต้น Class และ Object](./part-045-classes-objects.md)

## เป้าหมายการเรียนรู้ (Learning Objectives)

หลังจบ Part นี้ ผู้เรียนจะสามารถ:

1. เขียน **Function Overloading** (ฟังก์ชันชื่อเดียวกันหลาย signature) ได้อย่างถูกต้อง และอธิบาย
   ได้ว่า compiler เลือก overload ที่ "ตรง" ที่สุดได้อย่างไร
2. อธิบายแนวคิด **Name Mangling** เบื้องต้น และใช้ `nm`/`c++filt` เพื่อดูว่า compiler แยกฟังก์ชัน
   ชื่อเดียวกันแต่ signature ต่างกันออกจากกันได้อย่างไรในระดับ Object File
3. ใช้ **Default Argument** ได้อย่างถูกต้อง พร้อมเข้าใจกฎการวางลำดับ (ต้องอยู่ท้ายสุด) และรู้ว่า
   ควรประกาศค่า default ไว้ที่ไหนเมื่อแยก declaration/definition
4. เขียน **inline function** และอธิบายความแตกต่างจาก Macro Function (Part 14) ทั้งในแง่ความ
   ปลอดภัย (Type Checking) และพฤติกรรมเมื่อมี side-effect
5. เขียนฟังก์ชันที่รับพารามิเตอร์เป็น **reference** และ **const reference** ได้อย่างเหมาะสม
   โดยรู้ว่าเมื่อไหร่ควรใช้แบบไหน (ทบทวนและต่อยอดจาก Part 43)
6. เปรียบเทียบและเลือกใช้ **Pass by Value vs Reference vs Pointer** ได้อย่างเหมาะสมกับสถานการณ์
   จริง พร้อมอธิบายเหตุผลด้านประสิทธิภาพและความปลอดภัยของแต่ละแบบ

---

## 44.1 Function Overloading คืออะไร (Step 345)

ใน C ภาษาบังคับว่า **ฟังก์ชันแต่ละชื่อในโปรแกรมต้องมีได้แค่หนึ่งเดียว** (ถ้าตั้งชื่อซ้ำจะ error
ทันที) ทำให้เราต้องตั้งชื่อแยกกันทุกครั้งที่ต้องการทำงานคล้ายกันแต่รับ type ต่างกัน เช่น
`add_int`, `add_double`, `add_float` — น่ารำคาญและจำยาก

C++ แก้ปัญหานี้ด้วย **Function Overloading**: อนุญาตให้มีฟังก์ชัน **ชื่อเดียวกัน** ได้หลายตัว
ตราบใดที่ **signature ต่างกัน** (จำนวนพารามิเตอร์ต่างกัน หรือ type ของพารามิเตอร์ต่างกัน)
compiler จะเลือกฟังก์ชันที่ตรงกับ argument ที่เราส่งไปให้เองโดยอัตโนมัติตอน compile time

```cpp
// overload1.cpp
#include <iostream>

int add(int a, int b) {
    return a + b;
}

double add(double a, double b) {
    return a + b;
}

int add(int a, int b, int c) {
    return a + b + c;
}

int main() {
    std::cout << "add(2, 3) = " << add(2, 3) << '\n';
    std::cout << "add(2.5, 3.5) = " << add(2.5, 3.5) << '\n';
    std::cout << "add(1, 2, 3) = " << add(1, 2, 3) << '\n';
    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 overload1.cpp -o overload1
./overload1
```

ผลลัพธ์:

```
add(2, 3) = 5
add(2.5, 3.5) = 6
add(1, 2, 3) = 6
```

สังเกตว่าเราเรียก `add(...)` เหมือนกันทุกครั้ง แต่ compiler เลือกฟังก์ชันคนละตัวให้เอง
ขึ้นอยู่กับ **จำนวน** และ **ชนิด** ของ argument ที่ส่งเข้าไป กระบวนการนี้เรียกว่า
**Overload Resolution** และเกิดขึ้นทั้งหมดตอน **compile time** — ไม่มี cost ตอน runtime เลย

### signature คืออะไรกันแน่

**Signature** ของฟังก์ชันประกอบด้วยชื่อฟังก์ชัน + จำนวนพารามิเตอร์ + ชนิดของพารามิเตอร์
ตามลำดับ (ไม่นับ return type — เดี๋ยวจะอธิบายว่าทำไม)

```cpp
int    add(int a, int b);      // signature: add(int, int)
double add(double a, double b);// signature: add(double, double)
int    add(int a, int b, int c); // signature: add(int, int, int)
```

ทั้งสามฟังก์ชันนี้ signature ต่างกันหมด จึงอยู่ร่วมกันได้ในโปรแกรมเดียว

### กฎเหล็ก: จะ overload ไม่ได้ถ้า return type ต่างกันอย่างเดียว

```cpp
int get_value();     // ถูกต้อง (สมมติว่ามีแค่ตัวนี้)
double get_value();  // ERROR! ซ้ำกับตัวบนทุกอย่างยกเว้น return type
```

เหตุผลคือตอนเรียก `get_value();` เฉยๆ compiler ไม่มีทางรู้เลยว่าเราต้องการ `int` หรือ `double`
กลับมา (เพราะไม่ได้ผูกค่ากับตัวแปรหรือ context ใดๆ) จึงเลือกไม่ได้ — C++ จึงห้ามกรณีนี้ตั้งแต่
ขั้นตอน compile

---

## 44.2 กฎการ Overload Resolution และ Name Mangling เบื้องต้น (Step 346)

### Overload Resolution ทำงานอย่างไร

เมื่อเราเรียกฟังก์ชันที่มีชื่อซ้ำกันหลาย overload, compiler จะไล่หา candidate ที่ "แม่นที่สุด"
ตามลำดับความสำคัญคร่าวๆ ดังนี้ (จากดีที่สุดไปแย่ที่สุด):

1. **Exact Match** — type ตรงเป๊ะ (หรือต่างแค่ `const`/reference ผิวนอก)
2. **Promotion** — เช่น `char` → `int`, `float` → `double`
3. **Standard Conversion** — เช่น `int` → `double`, `double` → `int` (มีการปัดค่า/ตัดทศนิยม)
4. **User-defined Conversion** — ผ่าน constructor หรือ conversion operator ที่เราเขียนเอง
   (จะเรียนในภายหลัง)

ถ้ามี candidate มากกว่า 1 ตัวที่ "ดีเท่ากัน" ในระดับสูงสุด compiler จะรายงาน **ambiguous call**
ทันที (ไม่เดาให้เอง) ลองดูตัวอย่างที่ทำให้เกิด ambiguous:

```cpp
// ambiguous.cpp — ตัวอย่างนี้ "ผิดโดยตั้งใจ" เพื่อสาธิต error การ overload ที่คลุมเครือ
#include <iostream>

void describe(int a, double b) {
    std::cout << "int, double: " << a << ", " << b << '\n';
}

void describe(double a, int b) {
    std::cout << "double, int: " << a << ", " << b << '\n';
}

int main() {
    describe(1, 2); // ambiguous! ทั้งสอง overload ต้อง convert พารามิเตอร์ตัวหนึ่งเท่ากันพอดี
    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 ambiguous.cpp -o ambiguous
```

```
error: call of overloaded 'describe(int, int)' is ambiguous
note: candidate: 'void describe(int, double)'
note: candidate: 'void describe(double, int)'
```

เหตุผล: `describe(1, 2)` เรียกด้วย `int, int` ทั้งคู่ ถ้าเลือก `describe(int, double)` ต้อง
convert พารามิเตอร์ตัวที่ 2 จาก `int` เป็น `double` (1 conversion) ถ้าเลือก `describe(double, int)`
ต้อง convert พารามิเตอร์ตัวที่ 1 จาก `int` เป็น `double` (1 conversion เหมือนกัน) — ทั้งสองทาง
"เสียค่าใช้จ่ายในการแปลง" เท่ากันพอดี compiler จึงตัดสินใจแทนเราไม่ได้ และปฏิเสธที่จะเดา

### Name Mangling เบื้องต้น: ทำไม Overload ถึงทำงานได้จริงในระดับ Linker

คำถามที่มือใหม่มักสงสัย: ในเมื่อ Linker (ที่เรียนใน Part 1) ทำงานกับชื่อ symbol เป็น string ธรรมดา
แล้ว C++ ทำให้ `add(int, int)` กับ `add(double, double)` ไม่ชนกันใน Object File ได้อย่างไร
ในเมื่อทั้งคู่ชื่อ `add` เหมือนกัน?

คำตอบคือ compiler ของ C++ จะไม่เก็บชื่อฟังก์ชันตรงๆ ลงใน Object File แต่จะเข้ารหัสชื่อ +
signature ทั้งหมดให้กลายเป็นชื่อ symbol ที่ไม่ซ้ำกัน เทคนิคนี้เรียกว่า **Name Mangling**

ลองดูของจริงด้วยคำสั่ง `nm` (แสดง symbol table ของ Object File) และ `c++filt`
(แปลงชื่อที่ mangle แล้วกลับเป็นชื่อที่อ่านง่าย):

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 -c overload1.cpp -o overload1.o
nm overload1.o | grep ' T '
```

ผลลัพธ์ (ชื่อ symbol ที่ mangle แล้ว):

```
0000000000000018 T _Z3adddd
0000000000000000 T _Z3addii
0000000000000036 T _Z3addiii
0000000000000056 T main
```

แปลงกลับให้อ่านง่ายด้วย `c++filt`:

```bash
nm overload1.o | c++filt | grep ' T '
```

```
0000000000000018 T add(double, double)
0000000000000000 T add(int, int)
0000000000000036 T add(int, int, int)
0000000000000056 T main
```

จะเห็นว่า `_Z3addii` คือ `add(int, int)`, `_Z3adddd` คือ `add(double, double)` — ชื่อ symbol
ในระดับ machine code ถูกเข้ารหัสให้มีทั้งชื่อฟังก์ชันและ type ของพารามิเตอร์ครบถ้วน ทำให้
Linker มองเห็นว่าเป็นคนละ symbol กันจริงๆ ไม่ได้ชนกัน

> **สิ่งที่ควรรู้**: รูปแบบของ Name Mangling **ไม่ได้เป็นมาตรฐานสากล** — แต่ละคอมไพเลอร์
> (GCC/Clang ใช้ Itanium C++ ABI, MSVC ใช้รูปแบบของตัวเอง) mangle ชื่อไม่เหมือนกัน นี่คือ
> เหตุผลหลักที่ไฟล์ `.o` ที่คอมไพล์จาก GCC กับ MSVC เอามาลิงก์ร่วมกันตรงๆ ไม่ได้ และเป็น
> เหตุผลที่เวลาเขียนโค้ด C++ ให้ C เรียกใช้ (หรือกลับกัน) ต้องใช้ `extern "C"` เพื่อบอก compiler
> ว่า "อย่า mangle ชื่อนี้" (เรื่องนี้จะเจาะลึกใน Part ที่เกี่ยวกับการเชื่อม C กับ C++)

---

## 44.3 Default Argument และกฎการวางลำดับ (Step 347)

**Default Argument** คือการกำหนดค่าเริ่มต้นให้พารามิเตอร์ไว้ล่วงหน้า ถ้าผู้เรียกไม่ส่ง argument
ตัวนั้นมา compiler จะใช้ค่า default แทนให้อัตโนมัติ

```cpp
// default_args.cpp
#include <iostream>
#include <string>

void greet(const std::string& name, const std::string& greeting = "สวัสดี") {
    std::cout << greeting << ", " << name << "!\n";
}

double calc_price(double base_price, double discount_percent = 0.0, double tax_percent = 7.0) {
    double after_discount = base_price * (1.0 - discount_percent / 100.0);
    return after_discount * (1.0 + tax_percent / 100.0);
}

int main() {
    greet("สมชาย");             // ใช้ greeting default = "สวัสดี"
    greet("Somchai", "Hello");  // override greeting เอง

    std::cout << "ราคาปกติ: "        << calc_price(100.0) << '\n';
    std::cout << "ลด 10%: "          << calc_price(100.0, 10.0) << '\n';
    std::cout << "ลด 10% ภาษี 0%: " << calc_price(100.0, 10.0, 0.0) << '\n';
    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 default_args.cpp -o default_args
./default_args
```

```
สวัสดี, สมชาย!
Hello, Somchai!
ราคาปกติ: 107
ลด 10%: 96.3
ลด 10% ภาษี 0%: 90
```

`calc_price` แสดงประโยชน์ชัดเจน: ผู้เรียกส่งแค่ `base_price` ก็ใช้งานได้ทันที (ไม่มีส่วนลด
ภาษี 7% ตามปกติ) แต่ถ้าต้องการ override ก็ทำได้ทีละตัวจากซ้ายไปขวา

### กฎการวางลำดับ: Default Argument ต้องอยู่ "ท้ายสุด" เท่านั้น

กฎที่สำคัญที่สุดของ Default Argument คือ **เมื่อพารามิเตอร์ตัวใดตัวหนึ่งมีค่า default แล้ว
พารามิเตอร์ทุกตัวที่อยู่ถัดจากมัน (ทางขวา) ต้องมีค่า default ด้วยเช่นกัน** เหตุผลคือผู้เรียก
ต้องส่ง argument **ต่อเนื่องจากซ้ายไปขวาเสมอ** ห้ามข้าม — ถ้าอยากข้ามพารามิเตอร์ตัวกลาง
โดยใช้ default ของตัวสุดท้ายเอง ก็จะทำไม่ได้เพราะภาษาไม่รองรับการระบุ argument แบบ "ข้าม"
(ไม่เหมือนบาง keyword argument ใน Python)

```cpp
// bad_default.cpp — ตัวอย่างนี้ "ผิดโดยตั้งใจ" เพื่อสาธิต compile error
void bad_function(int a = 5, int b) {  // ERROR: parameter 'b' ไม่มี default
    (void)a;
    (void)b;
}

int main() {
    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 bad_default.cpp -o bad_default
```

```
error: default argument missing for parameter 2 of 'void bad_function(int, int)'
note: ...following parameter 1 which has a default argument
```

compiler อธิบายชัดเจนว่า: พารามิเตอร์ตัวที่ 1 (`a`) มี default แล้ว ดังนั้นพารามิเตอร์ตัวที่ 2
(`b`) ก็ต้องมี default ตามไปด้วย — วิธีแก้คือสลับตำแหน่งให้พารามิเตอร์ที่ "ไม่มี default"
มาอยู่ก่อนเสมอ:

```cpp
void good_function(int b, int a = 5); // ถูกต้อง: พารามิเตอร์ไม่มี default อยู่ก่อน
```

### ประกาศ Default Argument ที่ไหนเมื่อแยก Header/Implementation

เมื่อแยก declaration (ใน `.h`) กับ definition (ใน `.cpp`) ตามที่เรียนใน Part 17 (Modular
Programming) กฎคือ **ระบุค่า default ไว้ที่ declaration เท่านั้น** (ปกติคือใน header)
ห้ามระบุซ้ำอีกที่ definition มิฉะนั้นจะเป็น error เพราะเหมือนกำหนด default ให้พารามิเตอร์
เดียวกันสองครั้งในหน่วยแปลเดียวกัน

```cpp
// box.h
#ifndef BOX_H
#define BOX_H

double box_volume(double length, double width, double height = 1.0);

#endif
```

```cpp
// box.cpp
#include "box.h"

// ห้ามใส่ `= 1.0` ซ้ำตรงนี้อีก เพราะ default argument ประกาศได้ที่เดียวใน translation unit เดียวกัน
double box_volume(double length, double width, double height) {
    return length * width * height;
}
```

```cpp
// box_main.cpp
#include <iostream>
#include "box.h"

int main() {
    std::cout << "box_volume(2, 3) = "    << box_volume(2, 3)    << '\n';
    std::cout << "box_volume(2, 3, 4) = " << box_volume(2, 3, 4) << '\n';
    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 box_main.cpp box.cpp -o box_main
./box_main
```

```
box_volume(2, 3) = 6
box_volume(2, 3, 4) = 24
```

เหตุผลที่กฎนี้สมเหตุสมผล: ผู้ที่เรียกใช้ฟังก์ชันจะ `#include "box.h"` เท่านั้น ไม่เคยเห็น
`box.cpp` เลย ดังนั้นค่า default **ต้องอยู่ในที่ที่ผู้เรียกมองเห็น** คือ header — ถ้าใส่ไว้ใน
`.cpp` แทน ผู้เรียกที่มีแค่ header จะไม่รู้จักค่า default นั้นเลย

---

## 44.4 inline Function เทียบกับ Macro Function (Step 348)

ใน Part 14 เราเรียนเรื่อง **Macro Function** (`#define SQUARE(x) ((x) * (x))`) ซึ่งทำงานด้วย
การแทนที่ข้อความล้วนๆ ตอน preprocessing ก่อน compiler จริงจะเริ่มทำงาน แม้จะเร็วเพราะไม่มี
การเรียกฟังก์ชันจริง แต่ก็มีอันตรายมากมาย (ไม่มี type checking, ปัญหา side-effect เวลาส่ง
expression ที่มี `++`/`--` เข้าไป)

C++ เสนอทางเลือกที่ปลอดภัยกว่ามาก คือ **`inline` function**:

```cpp
inline int square_inline(int x) {
    return x * x;
}
```

`inline` เป็นเพียง **คำแนะนำ (hint)** ที่บอก compiler ว่า "ถ้าเป็นไปได้ ให้แทรกโค้ดของฟังก์ชัน
นี้เข้าไปตรงจุดที่เรียกใช้เลย (เหมือน macro) แทนที่จะสร้าง call จริง เพื่อลด overhead ของการ
เรียกฟังก์ชัน (function call overhead)" แต่ต่างจาก macro ตรงที่ `inline` function ยังคงเป็น
**ฟังก์ชันจริง** ที่ผ่าน type checking ของ compiler ทุกประการ

### เปรียบเทียบพฤติกรรมเมื่อมี side-effect

```cpp
// inline_demo.cpp
#include <iostream>

#define SQUARE_MACRO(x) ((x) * (x))

inline int square_inline(int x) {
    return x * x;
}

int main() {
    int a = 5;
    std::cout << "macro: "  << SQUARE_MACRO(a)   << '\n';
    std::cout << "inline: " << square_inline(a)  << '\n';

    // ตัวอย่างต่อไปนี้ "จงใจ" ส่ง ++i เข้า macro เพื่อสาธิตปัญหา side-effect ที่ Part 14 เคยเตือนไว้
    // บรรทัดนี้จะได้ warning -Wsequence-point จาก compiler ด้วย เพราะพฤติกรรมเป็น Undefined Behavior
    int i = 1;
    std::cout << "macro with ++i: " << SQUARE_MACRO(++i) << '\n';

    int j = 1;
    std::cout << "inline with ++j: " << square_inline(++j) << '\n';

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 inline_demo.cpp -o inline_demo
./inline_demo
```

```
macro: 25
inline: 25
macro with ++i: 9
inline with ++j: 4
```

`SQUARE_MACRO(++i)` ถูกขยายเป็น `((++i) * (++i))` ตอน preprocessing — `i` ถูกเพิ่มค่าถึง
**สองครั้ง** ในนิพจน์เดียว ซึ่งเป็น **Undefined Behavior** ตามมาตรฐาน (ลำดับการประเมินไม่ชัดเจน)
compiler เองก็เตือนด้วย warning `-Wsequence-point` ในบรรทัดนี้ ผลลัพธ์ `9` ที่ได้ (`i` เพิ่ม
จาก 1 เป็น 3 แล้ว `3 * 3 = 9`) เป็นแค่พฤติกรรมที่บังเอิญเกิดขึ้นกับ compiler ตัวนี้เท่านั้น
ไม่ควรพึ่งพา ในขณะที่ `square_inline(++j)` ปลอดภัยสมบูรณ์: `++j` ถูกประเมินค่า **ครั้งเดียว**
ก่อนถูกส่งเข้าไปเป็น argument ของฟังก์ชัน (พฤติกรรมเหมือนฟังก์ชันธรรมดาทุกประการ) ได้ผลลัพธ์
`4` (`j` เป็น 2 แล้ว `2 * 2 = 4`) ตรงตามที่คาดหวัง

### ตารางเปรียบเทียบ Macro Function vs inline Function

| คุณสมบัติ | Macro Function (`#define`) | `inline` Function |
|---|---|---|
| ทำงานตอนไหน | Preprocessing (Text Substitution) | Compile time (เป็นฟังก์ชันจริง) |
| Type Checking | ไม่มีเลย | มีครบเหมือนฟังก์ชันปกติ |
| ปัญหา side-effect (`++x`) | มี (ประเมินซ้ำได้) | ไม่มี (ประเมิน argument ครั้งเดียว) |
| Debug ด้วย Debugger | ยาก (ไม่มีชื่อฟังก์ชันให้ตั้ง breakpoint) | ง่าย (เป็นฟังก์ชันจริง ตั้ง breakpoint ได้) |
| Scope | ไม่มี scope (มีผลทั้งไฟล์หลังจุด define) | มี scope ตามปกติของภาษา (namespace, class ได้) |
| การควบคุมของ compiler | บังคับแทรกเสมอ (ไม่มีทางเลือก) | เป็นแค่คำแนะนำ compiler มีสิทธิ์ไม่ทำตามก็ได้ |

> **คำแนะนำของหลักสูตรนี้**: ตั้งแต่ C++ เป็นต้นไป ให้ใช้ `inline` function หรือ `constexpr`
> function (จะเรียนใน Module F) แทน Macro Function สำหรับ logic ทุกชนิดเสมอ เก็บ `#define`
> ไว้ใช้เฉพาะงานที่ preprocessor ทำได้อย่างเดียวจริงๆ เช่น header guard หรือ conditional
> compilation (`#ifdef`)

### `inline` ไม่ได้แปลว่า "บังคับ inline เสมอ"

จุดที่มือใหม่เข้าใจผิดบ่อยคือคิดว่า `inline` สั่งให้ compiler แทรกโค้ดเข้าไปเสมอ แต่จริงๆ แล้ว
compiler สมัยใหม่ (GCC, Clang) **มีอิสระที่จะไม่ทำตาม** คำแนะนำนี้ก็ได้ ถ้าฟังก์ชันมีขนาดใหญ่
เกินไปหรือถูกเรียกแบบ recursive — และในทางกลับกัน compiler ก็สามารถ inline ฟังก์ชัน
**ที่ไม่ได้ใส่ `inline` เลย** ได้เองถ้าเปิด optimization (`-O2` เป็นต้นไป) และเห็นว่าคุ้มค่า

ความหมายที่แท้จริงและสำคัญที่สุดของ `inline` ในภาษา C++ สมัยใหม่ (ตั้งแต่การเขียนฟังก์ชันใน
header) จึงเป็นเรื่อง **linkage** มากกว่าประสิทธิภาพ: `inline` อนุญาตให้ฟังก์ชันตัวเดียวกันถูก
นิยามซ้ำได้ในหลาย translation unit (เช่นเมื่อ header ถูก `#include` เข้าไปในหลายไฟล์ `.cpp`)
โดยไม่ทำให้ linker ฟ้อง "multiple definition error" — รายละเอียดเรื่องนี้จะเจาะลึกอีกครั้งเมื่อ
เขียน class ที่มี member function นิยามอยู่ใน header (Part 45 เป็นต้นไป)

---

## 44.5 ฟังก์ชันที่รับ Reference เป็น Parameter (Step 349)

Part 43 แนะนำให้รู้จัก **Reference** (`int&`) ในฐานะ "ชื่อเล่นของตัวแปรเดิม" ในบริบทนี้เราจะ
ทบทวนและเจาะลึกว่าทำไม reference parameter ถึงสำคัญมากในการออกแบบฟังก์ชัน

### ปัญหาของ pass by value เมื่อต้องการแก้ไขค่าต้นฉบับ

```cpp
void increment_by_value(int x) {
    x = x + 1; // แก้ไขแค่ "สำเนา" ในฟังก์ชัน ไม่มีผลต่อต้นฉบับ
}
```

ถ้าเรียก `increment_by_value(num)` ค่า `num` ข้างนอกจะ **ไม่เปลี่ยนเลย** เพราะพารามิเตอร์
`x` เป็นแค่สำเนา (copy) ของ `num` เท่านั้น การแก้ไข `x` จึงไม่มีผลย้อนกลับไปที่ `num`

### แก้ปัญหาด้วย reference parameter

```cpp
void increment_by_reference(int& x) {
    x = x + 1; // x คือ "ชื่อเล่น" ของตัวแปรต้นฉบับโดยตรง แก้ x = แก้ต้นฉบับ
}
```

เมื่อพารามิเตอร์เป็น `int&` การเรียก `increment_by_reference(num)` จะทำให้ `x` ภายในฟังก์ชัน
**ผูกติดกับ `num` โดยตรง** ไม่มีการคัดลอกใดๆ เกิดขึ้น การแก้ไข `x` จึงเท่ากับแก้ไข `num` จริง

ตัวอย่างที่ใช้งานได้จริง: ฟังก์ชัน `swap` แบบคลาสสิก

```cpp
// swap_demo.cpp
#include <iostream>

void swap_by_pointer(int* a, int* b) {
    int temp = *a;
    *a = *b;
    *b = temp;
}

void swap_by_reference(int& a, int& b) {
    int temp = a;
    a = b;
    b = temp;
}

int main() {
    int x = 1, y = 2;
    swap_by_pointer(&x, &y);
    std::cout << "after swap_by_pointer: x=" << x << " y=" << y << '\n';

    swap_by_reference(x, y);
    std::cout << "after swap_by_reference: x=" << x << " y=" << y << '\n';

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 swap_demo.cpp -o swap_demo
./swap_demo
```

```
after swap_by_pointer: x=2 y=1
after swap_by_reference: x=1 y=2
```

สังเกตว่า `swap_by_reference(x, y)` เรียกใช้งานได้ **สะอาดกว่า** `swap_by_pointer(&x, &y)`
มาก — ไม่ต้องใส่ `&` ตอนเรียก และภายในฟังก์ชันก็ไม่ต้อง dereference ด้วย `*` เลย นี่คือเหตุผล
หลักที่โค้ด C++ สมัยใหม่นิยมใช้ reference parameter แทน pointer parameter เมื่อ "รู้แน่นอนว่า
argument มีอยู่จริง ไม่มีทาง null" (จะเจาะลึกเกณฑ์การเลือกในหัวข้อ 44.7 และ 44.8)

---

## 44.6 const Reference Parameter: ปลอดภัยและมีประสิทธิภาพ (Step 350)

ปัญหาถัดมา: ถ้าเราแค่ต้องการ "อ่าน" ค่าของ object ขนาดใหญ่ (เช่น `std::string` ที่มีตัวอักษร
เป็นพันตัว) โดยไม่ต้องการแก้ไขมันเลย เราควรใช้ pass by value หรือ reference ดี?

- **Pass by value**: ปลอดภัย (ไม่มีทางแก้ต้นฉบับโดยไม่ตั้งใจ) แต่ **เปลืองมาก** เพราะต้อง
  คัดลอกข้อมูลทั้งหมดทุกครั้งที่เรียกฟังก์ชัน
- **Pass by reference (non-const)**: ไม่คัดลอกข้อมูล เร็ว แต่ **ไม่ปลอดภัย** เพราะเปิดช่องให้
  ฟังก์ชันแก้ไขต้นฉบับได้โดยไม่ตั้งใจ (ผู้เรียกอ่านโค้ดแล้วอาจไม่รู้ด้วยซ้ำว่าค่าตัวเองจะถูกแก้)

คำตอบที่ดีที่สุดคือ **const reference** (`const std::string&`) — ได้ทั้งสองข้อดี: ไม่คัดลอก
ข้อมูล (เร็วเท่า reference) และ compiler รับประกันว่าฟังก์ชันจะไม่แก้ไขต้นฉบับ (ปลอดภัยเท่า
pass by value)

```cpp
#include <string>

std::size_t length_by_const_ref(const std::string& s) {
    return s.size(); // อ่านได้อย่างเดียว ห้ามแก้ไข s
}
```

ถ้าลองแก้ไข `s` ภายในฟังก์ชันที่รับ `const std::string&` compiler จะฟ้อง error ทันทีตอน
compile time — นี่คือพลังของ **const correctness** ที่เรียนใน Part 43: มันเปลี่ยนกฎที่ควร
เป็นแค่ "ข้อตกลงในใจ" ให้กลายเป็นกฎที่ compiler ช่วยตรวจสอบให้จริง

### กฎง่ายๆ ในการเลือก: value, reference หรือ const reference

| สถานการณ์ | ควรใช้ |
|---|---|
| type เล็ก (`int`, `double`, `char`, `bool`) และไม่ต้องแก้ไขต้นฉบับ | pass by value (คัดลอกถูกกว่าสร้าง reference) |
| type ใหญ่ (`std::string`, `std::vector`, class ที่เขียนเอง) และแค่ต้องการ "อ่าน" | `const T&` |
| ต้องการแก้ไขต้นฉบับ ไม่ว่า type จะเล็กหรือใหญ่ | `T&` (non-const reference) |
| อาจไม่มี object อยู่จริง (nullable) | pointer (`T*`) — จะเจาะลึกในหัวข้อถัดไป |

> **เกร็ดสำคัญ**: type เล็กอย่าง `int` การส่งด้วย reference (`int&`) จริงๆ แล้ว **ไม่ได้เร็วกว่า**
> pass by value เลย เพราะ reference ในทางเทคนิคมักถูก implement ด้วย pointer ภายใน (กิน
> พื้นที่เท่ากับ pointer คือ 8 ไบต์บนเครื่อง 64-bit) ในขณะที่ `int` กินแค่ 4 ไบต์ — การส่งแบบ
> `const int&` จึงอาจ "ช้ากว่า" การส่ง `int` ตรงๆ ด้วยซ้ำ! กฎทองคือ: **type เล็กและ trivial
> (คัดลอกเร็ว) ให้ pass by value เสมอ ส่วน type ใหญ่ที่คัดลอกแพง (มี dynamic memory ข้างใน
> เช่น string/vector/class) ให้ใช้ const reference**

---

## 44.7 Pointer Parameter ใน C++ และเมื่อไหร่ควรใช้ (Step 351)

C++ ยังรองรับ pointer parameter เหมือน C ทุกประการ (`int*`, `std::string*` ฯลฯ) คำถามคือ
ในเมื่อมี reference ที่ใช้งานง่ายกว่าแล้ว ทำไมยังต้องใช้ pointer อยู่?

คำตอบคือ **pointer สื่อความหมายบางอย่างที่ reference ทำไม่ได้**: pointer สามารถเป็น
`nullptr` ได้ (แปลว่า "ไม่มี object อยู่จริง" หรือ "optional") ในขณะที่ reference **ต้องผูก
กับ object ที่มีอยู่จริงเสมอ** ตั้งแต่ตอนสร้าง ไม่มีทางเป็น "reference ที่ไม่ชี้ไปที่ไหนเลย" ได้
(ไม่มี "null reference" ในภาษา)

```cpp
#include <string>

void shout_by_pointer(std::string* s) {
    if (s == nullptr) {   // pointer ตรวจสอบได้ว่า "มีของจริงหรือไม่"
        return;
    }
    for (char& c : *s) {
        c = static_cast<char>(std::toupper(static_cast<unsigned char>(c)));
    }
}
```

เรียกใช้ได้ทั้งกรณีมี object จริงและกรณีไม่มี:

```cpp
std::string d = "pointer";
shout_by_pointer(&d);       // ปกติ
shout_by_pointer(nullptr);  // ปลอดภัย เพราะฟังก์ชันเช็ค nullptr ไว้แล้ว
```

ถ้าเขียนฟังก์ชันเดียวกันด้วย reference (`std::string&`) จะ **ไม่มีทางส่ง "ไม่มีอะไรเลย"** เข้า
ไปได้เลย เพราะ reference บังคับให้ต้องผูกกับตัวแปรจริงเสมอตั้งแต่ตอนเรียก — นี่คือข้อดีสำคัญ
ของ reference จากมุมมองความปลอดภัย (ไม่มีทาง "ลืมเช็ค null" เพราะไม่มี null ให้เช็คตั้งแต่แรก)
แต่ก็เป็นข้อจำกัดถ้าเราต้องการสื่อความหมาย "อาจไม่มีค่า" จริงๆ

> **หมายเหตุสำหรับ C++ สมัยใหม่**: ตั้งแต่ C++17 เป็นต้นไป มี `std::optional<T>` (Module F)
> ที่สื่อความหมาย "อาจไม่มีค่า" ได้ปลอดภัยกว่า raw pointer มาก (ไม่มีปัญหา dangling pointer)
> แนวทางปฏิบัติในโปรเจกต์จริงจึงมักสงวน raw pointer ไว้สำหรับกรณีเฉพาะ (เช่น optional
> parameter ที่ไม่ต้องเป็นเจ้าของหน่วยความจำ) และใช้ reference/`std::optional` เป็นค่าเริ่มต้น

---

## 44.8 สรุปเปรียบเทียบ: Pass by Value vs Reference vs Pointer (Step 352)

มาดูทั้งสามแบบเทียบกันในโปรแกรมเดียว เพื่อเห็นความแตกต่างของพฤติกรรมชัดเจน:

```cpp
// passing.cpp
#include <cctype>
#include <iostream>
#include <string>

// 1) Pass by value: คัดลอกทั้งก้อน ปลอดภัยแต่เปลืองถ้าข้อมูลใหญ่
std::string shout_by_value(std::string s) {
    for (char& c : s) {
        c = static_cast<char>(std::toupper(static_cast<unsigned char>(c)));
    }
    return s;
}

// 2) Pass by reference (non-const): แก้ไขต้นฉบับได้โดยตรง ไม่มีการคัดลอก
void shout_by_reference(std::string& s) {
    for (char& c : s) {
        c = static_cast<char>(std::toupper(static_cast<unsigned char>(c)));
    }
}

// 3) Pass by const reference: อ่านอย่างเดียว ไม่คัดลอก ไม่แก้ไขต้นฉบับ
std::size_t length_by_const_ref(const std::string& s) {
    return s.size();
}

// 4) Pass by pointer: ต้องเช็ค nullptr เอง แต่สื่อความหมาย "อาจไม่มีของจริง" (optional) ได้
void shout_by_pointer(std::string* s) {
    if (s == nullptr) {
        return;
    }
    for (char& c : *s) {
        c = static_cast<char>(std::toupper(static_cast<unsigned char>(c)));
    }
}

int main() {
    std::string a = "hello";
    std::string result = shout_by_value(a);
    std::cout << "by value: a ยังเป็น \"" << a << "\", result = \"" << result << "\"\n";

    std::string b = "world";
    shout_by_reference(b);
    std::cout << "by reference: b กลายเป็น \"" << b << "\"\n";

    std::string c = "const test";
    std::cout << "by const&: length = " << length_by_const_ref(c) << '\n';

    std::string d = "pointer";
    shout_by_pointer(&d);
    std::cout << "by pointer: d กลายเป็น \"" << d << "\"\n";

    shout_by_pointer(nullptr); // ปลอดภัย เพราะเช็ค nullptr ไว้แล้ว

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 passing.cpp -o passing
./passing
```

```
by value: a ยังเป็น "hello", result = "HELLO"
by reference: b กลายเป็น "WORLD"
by const&: length = 10
by pointer: d กลายเป็น "POINTER"
```

### ตารางสรุปสุดท้าย

| ลักษณะ | Pass by Value (`T`) | Pass by Reference (`T&`) | Pass by const Reference (`const T&`) | Pass by Pointer (`T*`) |
|---|---|---|---|---|
| มีการคัดลอกข้อมูลหรือไม่ | มี (คัดลอกทั้งก้อน) | ไม่มี | ไม่มี | ไม่มี (คัดลอกแค่ที่อยู่ 8 ไบต์) |
| แก้ไขต้นฉบับได้หรือไม่ | ไม่ได้ | ได้ | ไม่ได้ (compiler บังคับ) | ได้ (ถ้าไม่ใช่ `const T*`) |
| รับ `nullptr` / "ไม่มีค่า" ได้หรือไม่ | ไม่ได้ | ไม่ได้ | ไม่ได้ | ได้ |
| syntax ตอนเรียกใช้ | ปกติ `f(x)` | ปกติ `f(x)` (ไม่ต้องใส่ `&`) | ปกติ `f(x)` | ต้องใส่ `&x` หรือส่ง pointer ที่มีอยู่แล้ว |
| syntax ภายในฟังก์ชัน | ใช้ตรงๆ | ใช้ตรงๆ | ใช้ตรงๆ (แก้ไม่ได้) | ต้อง dereference ด้วย `*` |
| เหมาะกับ | type เล็ก (`int`, `double`, `char`) | ต้องแก้ไขค่าต้นฉบับ (เช่น `swap`) | type ใหญ่ที่แค่ต้องการอ่าน (ค่า default ที่ควรใช้เมื่อไม่แน่ใจ) | ค่าที่อาจไม่มีอยู่จริง หรือทำงานร่วมกับ C-style API |

**หลักการเลือกอย่างรวดเร็ว (Rule of Thumb) ที่ใช้ได้ตลอดหลักสูตรนี้:**

1. type เล็ก ไม่ต้องแก้ไข → **pass by value**
2. type ใหญ่ ไม่ต้องแก้ไข → **`const T&`**
3. ต้องแก้ไขต้นฉบับ (ไม่ว่า type เล็กหรือใหญ่) → **`T&`**
4. ค่าที่อาจไม่มีอยู่จริง (optional) → **`T*`** (หรือ `std::optional<T>` ในโค้ดสมัยใหม่)

---

## ข้อผิดพลาดที่พบบ่อย (Common Pitfalls)

1. **สร้าง overload ที่ต่างกันแค่ return type** — ภาษาไม่อนุญาต เพราะตอนเรียกฟังก์ชันเฉยๆ
   compiler ไม่มีทางรู้ว่าอยากได้ type ไหนกลับมา ต้องทำให้ signature (จำนวน/ชนิดพารามิเตอร์)
   ต่างกันจริงเสมอ
2. **ลืมว่า default argument ต้องอยู่ท้ายสุด** — ถ้าใส่ default ให้พารามิเตอร์ตัวกลางโดยที่
   ตัวขวาสุดไม่มี default จะ compile error ทันที ต้องจัดลำดับพารามิเตอร์ใหม่เสมอ
3. **ใส่ default argument ซ้ำทั้งใน header และ `.cpp`** — จะได้ error "redefinition of default
   argument" เพราะ default ถูกกำหนดได้แค่ครั้งเดียวต่อ translation unit ให้ใส่ไว้ที่ header
   (declaration) เท่านั้น
4. **คิดว่า `inline` บังคับให้ compiler inline เสมอ** — จริงๆ แล้วเป็นแค่คำแนะนำ compiler มี
   สิทธิ์ปฏิเสธได้ถ้าฟังก์ชันใหญ่เกินไปหรือ recursive และ compiler ก็ inline ฟังก์ชันที่ไม่มี
   `inline` เองได้เช่นกันเมื่อเปิด optimization
5. **ใช้ pass by value กับ object ขนาดใหญ่โดยไม่จำเป็น** — ทำให้เสีย performance จากการ
   คัดลอกข้อมูลทุกครั้งที่เรียกฟังก์ชันโดยไม่จำเป็น ทั้งที่แค่ต้องการ "อ่าน" ค่าเท่านั้น ควรใช้
   `const T&` แทนสำหรับ type ที่มี dynamic memory ข้างใน
6. **ใช้ reference parameter (non-const) ทั้งที่จริงๆ ไม่ได้ต้องการแก้ไขต้นฉบับ** — ทำให้
   ผู้เรียกอ่านโค้ดแล้วเข้าใจผิดว่าค่าของตัวเองอาจถูกฟังก์ชันแก้ไข ควรใส่ `const` เสมอถ้าไม่ได้
   ตั้งใจแก้ไขจริงๆ (const correctness ตาม Part 43)
7. **ลืมเช็ค `nullptr` ก่อน dereference pointer parameter** — ถ้าฟังก์ชันรับ `T*` แล้วมีโอกาส
   ถูกเรียกด้วย `nullptr` (เช่น optional parameter) ต้องเช็คก่อนใช้งานทุกครั้ง มิฉะนั้นจะเกิด
   Undefined Behavior (segmentation fault) ทันทีที่ dereference pointer ที่เป็น null

---

## แบบฝึกหัดท้ายบท

1. เขียนฟังก์ชัน `area` แบบ overload 2 รูปแบบ: `double area(double side)` สำหรับพื้นที่สี่เหลี่ยม
   จัตุรัส และ `double area(double width, double height)` สำหรับพื้นที่สี่เหลี่ยมผืนผ้า พร้อม
   เขียน `main` ทดสอบเรียกใช้ทั้งสองแบบ
2. เขียนฟังก์ชัน `void print_info(const std::string& name, int age = 0, const std::string& city
   = "ไม่ระบุ")` ที่พิมพ์ข้อมูลออกมา แล้วทดลองเรียกใช้ 3 แบบ: ส่งแค่ `name`, ส่ง `name` กับ `age`,
   และส่งครบทั้ง 3 ตัว
3. ใช้ `nm` และ `c++filt` ตรวจสอบว่าฟังก์ชัน overload ที่คุณเขียนในข้อ 1 ถูก mangle เป็นชื่ออะไร
   บ้างในระดับ Object File
4. เขียนฟังก์ชัน `void reset_to_zero(int& x)` ที่รับ reference แล้วตั้งค่าตัวแปรที่ส่งเข้ามาเป็น 0
   เขียน `main` ทดสอบว่าตัวแปรต้นฉบับเปลี่ยนค่าจริง
5. เขียนฟังก์ชัน `double sum_of_vector(const std::vector<double>& numbers)` ที่บวกค่าทุกตัวใน
   `vector` แล้วอธิบายด้วยคำพูดตัวเองว่าทำไมต้องใช้ `const std::vector<double>&` แทนที่จะเป็น
   `std::vector<double>` เฉยๆ
6. เขียนฟังก์ชัน `void update_score(int* score, int delta)` ที่รับ pointer และเพิ่มค่าตาม `delta`
   โดยถ้า `score` เป็น `nullptr` ให้ฟังก์ชัน return ทันทีโดยไม่ทำอะไร ทดสอบทั้งกรณีส่ง pointer
   จริงและกรณีส่ง `nullptr`

### แนวทางเฉลยข้อ 1

```cpp
#include <iostream>

double area(double side) {
    return side * side;
}

double area(double width, double height) {
    return width * height;
}

int main() {
    std::cout << "พื้นที่สี่เหลี่ยมจัตุรัสด้าน 4: " << area(4.0) << '\n';
    std::cout << "พื้นที่สี่เหลี่ยมผืนผ้า 3x5: "     << area(3.0, 5.0) << '\n';
    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 exercise1.cpp -o exercise1
./exercise1
```

```
พื้นที่สี่เหลี่ยมจัตุรัสด้าน 4: 16
พื้นที่สี่เหลี่ยมผืนผ้า 3x5: 15
```

`area(4.0)` ตรงกับ overload ตัวแรก (พารามิเตอร์เดียว) ส่วน `area(3.0, 5.0)` ตรงกับ overload
ตัวที่สอง (สองพารามิเตอร์) — compiler เลือกให้ถูกต้องเองตาม signature โดยที่เราไม่ต้องตั้งชื่อ
ฟังก์ชันแยกกันเลย

### แนวทางเฉลยข้อ 4

```cpp
#include <iostream>

void reset_to_zero(int& x) {
    x = 0;
}

int main() {
    int counter = 42;
    std::cout << "ก่อนเรียก: counter = " << counter << '\n';
    reset_to_zero(counter);
    std::cout << "หลังเรียก: counter = " << counter << '\n';
    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 exercise4.cpp -o exercise4
./exercise4
```

```
ก่อนเรียก: counter = 42
หลังเรียก: counter = 0
```

เพราะพารามิเตอร์ `x` เป็น `int&` ซึ่งเป็นชื่อเล่นของ `counter` โดยตรง (ไม่มีการคัดลอก) การ
เขียน `x = 0;` ภายในฟังก์ชันจึงเท่ากับเขียน `counter = 0;` โดยตรง — ค่าต้นฉบับจึงเปลี่ยนจริง
ต่างจาก pass by value ที่การแก้ไขพารามิเตอร์จะไม่มีผลย้อนกลับไปที่ตัวแปรต้นฉบับเลย

---

## สรุปท้ายบท

ใน Part นี้เราได้:

- เข้าใจ **Function Overloading** และกฎ Overload Resolution ที่ compiler ใช้เลือกฟังก์ชันที่
  ตรงที่สุดตาม signature
- เห็นของจริงว่า **Name Mangling** ทำให้ overload ทำงานได้ในระดับ Object File/Linker ผ่าน
  เครื่องมือ `nm` และ `c++filt`
- ใช้ **Default Argument** ได้อย่างถูกต้อง พร้อมเข้าใจกฎการวางลำดับ (ต้องอยู่ท้ายสุด) และรู้ว่า
  ต้องประกาศค่า default ไว้ที่ declaration/header เท่านั้น
- เข้าใจว่า **inline function** ปลอดภัยกว่า Macro Function ของ Part 14 อย่างไร ทั้งในแง่
  type checking และปัญหา side-effect
- ทบทวนและต่อยอด **reference/const reference parameter** จาก Part 43 ในบริบทของการออกแบบ
  ฟังก์ชัน
- สรุปเปรียบเทียบ **Pass by Value vs Reference vs Pointer** ครบทั้ง 3 รูปแบบ พร้อมหลักการ
  เลือกใช้ที่นำไปใช้ได้จริงตลอดหลักสูตรที่เหลือ

พื้นฐานเรื่องฟังก์ชันใน C++ ที่เราวางไว้ใน Part นี้ — โดยเฉพาะเรื่อง overloading, default
argument และการเลือกวิธีส่งพารามิเตอร์อย่างเหมาะสม — จะถูกใช้ซ้ำตลอดไปเมื่อเราเริ่มเขียน
**member function** ของ class ใน Part ถัดไป เพราะ member function ก็คือฟังก์ชันที่ทำตามกฎ
เดียวกันทุกประการ เพียงแต่ผูกอยู่กับ object

**ต่อไป:** [Part 45 — เริ่มต้น Class และ Object](./part-045-classes-objects.md)
