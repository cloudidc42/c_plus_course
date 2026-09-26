# Part 15: Bit Manipulation (Step 113–120)

> Module B — C ระดับกลาง & โครงสร้างข้อมูล/อัลกอริทึม (Intermediate C & Data Structures/Algorithms) | Part 15 จาก 125
> Step ที่ครอบคลุมใน Part นี้: Step 113–120
> Part ก่อนหน้า: [Part 14 — Preprocessor และ Macro](./part-014-preprocessor-macros.md) | Part ถัดไป: [Part 16 — Error Handling ใน C](./part-016-error-handling.md)

## เป้าหมายการเรียนรู้ (Learning Objectives)

หลังจบ Part นี้ ผู้เรียนจะสามารถ:

1. ใช้ Bitwise Operator ทั้งหมด (`&`, `|`, `^`, `~`, `<<`, `>>`) ได้อย่างถูกต้อง พร้อมอธิบาย
   ตารางค่าความจริง (Truth Table) ของแต่ละตัวได้
2. อธิบาย Two's Complement และเข้าใจว่าทำไมเลขลบถึงถูกเก็บในหน่วยความจำแบบนั้น รวมถึงความ
   แตกต่างระหว่าง Arithmetic Shift และ Logical Shift
3. ออกแบบ Bit Flag เพื่อเก็บสถานะ (State) หลายอย่างไว้ในตัวแปรจำนวนเต็มตัวเดียว แทนการใช้
   ตัวแปร `bool` หลายตัว
4. เขียนฟังก์ชัน/Macro สำหรับ set, clear, toggle, และ check bit ได้อย่างปลอดภัยและนำกลับมาใช้ซ้ำได้
5. ใช้ Bit Field ใน `struct` เพื่อประหยัดหน่วยความจำ พร้อมรู้ข้อจำกัดเรื่อง portability
6. ประยุกต์ใช้ Bit Manipulation กับปัญหาจริง เช่นระบบ Permission Flag แบบ Unix-style
7. ใช้เทคนิค Bit Trick ที่พบบ่อยในงานจริง เช่นการนับจำนวนบิตที่เป็น 1, การตรวจสอบเลขยกกำลังสอง,
   และการสลับค่าตัวแปรด้วย XOR
8. ตัดสินใจได้ว่าเมื่อไหร่ควรใช้ Bit Manipulation ในงานจริง และเมื่อไหร่ที่ความอ่านง่ายสำคัญกว่า

---

## 15.1 Bitwise Operator: `&`, `|`, `^`, `~` (Step 113)

ทุกค่าที่เก็บในหน่วยความจำของคอมพิวเตอร์ ไม่ว่าจะเป็น `int`, `char`, หรือ `struct` สุดท้ายแล้ว
ล้วนถูกเก็บเป็น **บิต (Bit)** ที่มีค่าได้แค่ `0` หรือ `1` เท่านั้น **Bitwise Operator** คือชุดตัว
ดำเนินการที่ทำงาน "ในระดับบิต" โดยตรง แตกต่างจาก Logical Operator (`&&`, `||`, `!`) ที่มองค่า
ทั้งตัวแปรเป็น "จริง/เท็จ" เพียงค่าเดียว

| Operator | ชื่อ | ความหมาย |
|---|---|---|
| `&` | AND | บิตผลลัพธ์เป็น 1 ก็ต่อเมื่อบิตทั้งสองฝั่งเป็น 1 ทั้งคู่ |
| `\|` | OR | บิตผลลัพธ์เป็น 1 ถ้าอย่างน้อยหนึ่งฝั่งเป็น 1 |
| `^` | XOR (Exclusive OR) | บิตผลลัพธ์เป็น 1 ถ้าสองฝั่ง **ต่างกัน** เท่านั้น |
| `~` | NOT (Bitwise Complement) | กลับค่าทุกบิต (0→1, 1→0) — เป็น Unary Operator ตัวเดียว |

### ตารางค่าความจริง (Truth Table) แบบละเอียด

| A | B | A & B | A \| B | A ^ B |
|---|---|---|---|---|
| 0 | 0 | 0 | 0 | 0 |
| 0 | 1 | 0 | 1 | 1 |
| 1 | 0 | 0 | 1 | 1 |
| 1 | 1 | 1 | 1 | 0 |

และสำหรับ `~` (NOT) ซึ่งรับ operand ตัวเดียว:

| A | ~A |
|---|---|
| 0 | 1 |
| 1 | 0 |

### ตัวอย่างการคำนวณจริงทีละบิต

สมมติ `a = 12` และ `b = 10` (เป็น `unsigned char` 8 บิตเพื่อให้เห็นภาพชัด):

```
a = 12  = 0000 1100
b = 10  = 0000 1010
-----------------------
a & b   = 0000 1000  = 8
a | b   = 0000 1110  = 14
a ^ b   = 0000 0110  = 6
~a      = 1111 0011  = 243  (เมื่อมองเป็น unsigned char 8 บิต)
```

```c
/* bitwise_basics.c */
#include <stdio.h>

int main(void) {
    unsigned char a = 12;   /* 0000 1100 */
    unsigned char b = 10;   /* 0000 1010 */

    printf("a & b = %u\n", (unsigned)(a & b));   /* 8  */
    printf("a | b = %u\n", (unsigned)(a | b));   /* 14 */
    printf("a ^ b = %u\n", (unsigned)(a ^ b));   /* 6  */
    printf("~a    = %u\n", (unsigned)(unsigned char)~a);  /* 243 */

    return 0;
}
```

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 bitwise_basics.c -o bitwise_basics
./bitwise_basics
```

```
a & b = 8
a | b = 14
a ^ b = 6
~a    = 243
```

สังเกตว่าเราต้อง cast ผลลัพธ์กลับเป็น `(unsigned char)` ก่อนพิมพ์ด้วย `%u` เพราะกฎ
**Integer Promotion** ของภาษา C จะเลื่อนค่า `unsigned char` ให้กลายเป็น `int` โดยอัตโนมัติ
ก่อนคำนวณ bitwise operation เสมอ (เพื่อป้องกัน Warning และให้ผลลัพธ์ตรงกับที่ตั้งใจแสดงในขนาด
8 บิตจริงๆ)

### `&` ใช้ทำ Masking (การกรองเฉพาะบิตที่ต้องการ)

การใช้งานที่พบบ่อยที่สุดของ `&` คือการ **Mask** หรือ "กรอง" เอาเฉพาะบิตที่สนใจออกมา โดยใช้
ตัวเลขที่มีบิต 1 อยู่เฉพาะตำแหน่งที่ต้องการตรวจสอบ (เรียกว่า **Bitmask**):

```c
#include <stdio.h>

int main(void) {
    unsigned int flags = 0b10110110;   /* binary literal เป็นฟีเจอร์ GNU extension ของ gcc */
    unsigned int mask  = 0b00000110;   /* สนใจแค่บิตที่ 1 และ 2 (นับจาก 0) */

    unsigned int extracted = flags & mask;
    printf("บิตที่สนใจ: %u\n", extracted);
    return 0;
}
```

> **หมายเหตุ**: Binary literal (`0b...`) เป็น GNU Extension ที่ `gcc` รองรับมาตั้งแต่ C99/C11
> แต่ **ไม่ใช่ส่วนหนึ่งของมาตรฐาน ISO C จนถึง C17** (เพิ่งถูกบรรจุเป็นมาตรฐานอย่างเป็นทางการใน
> C23) ดังนั้นถ้าคอมไพล์ด้วย `-Wpedantic` จะได้ Warning ว่าเป็น extension ในตัวอย่างที่เหลือของ
> บทเรียนนี้เราจะเขียนค่าบิตด้วย Hexadecimal (`0x...`) แทนเพื่อให้คอมไพล์ผ่านแบบไม่มี Warning
> เลยตามมาตรฐาน C17 อย่างเคร่งครัด

```c
/* masking_hex.c */
#include <stdio.h>

int main(void) {
    unsigned int flags = 0xB6;    /* 1011 0110 */
    unsigned int mask  = 0x06;    /* 0000 0110 — สนใจบิตที่ 1,2 */

    unsigned int extracted = flags & mask;
    printf("flags       = 0x%02X\n", flags);
    printf("mask        = 0x%02X\n", mask);
    printf("extracted   = 0x%02X\n", extracted);
    return 0;
}
```

```
flags       = 0xB6
mask        = 0x06
extracted   = 0x06
```

---

## 15.2 Shift Operator (`<<`, `>>`) และ Two's Complement (Step 114)

### Left Shift (`<<`) และ Right Shift (`>>`)

`<<` เลื่อนบิตทั้งหมดไปทางซ้าย โดยเติม `0` เข้ามาทางขวา ส่วน `>>` เลื่อนบิตไปทางขวา (พฤติกรรม
การเติมค่าทางซ้ายขึ้นกับว่าตัวแปรเป็น signed หรือ unsigned — จะอธิบายละเอียดด้านล่าง)

```c
#include <stdio.h>

int main(void) {
    unsigned int x = 1;
    printf("1 << 0 = %u\n", x << 0);   /* 1  = 0000 0001 */
    printf("1 << 1 = %u\n", x << 1);   /* 2  = 0000 0010 */
    printf("1 << 2 = %u\n", x << 2);   /* 4  = 0000 0100 */
    printf("1 << 3 = %u\n", x << 3);   /* 8  = 0000 1000 */

    unsigned int y = 128;
    printf("128 >> 1 = %u\n", y >> 1);   /* 64 */
    printf("128 >> 4 = %u\n", y >> 4);   /* 8  */
    return 0;
}
```

**กฎสำคัญ**: `x << n` มีค่าเท่ากับ `x * 2^n` และ `x >> n` มีค่าเท่ากับ `x / 2^n` (สำหรับ
`unsigned` หรือค่า `signed` ที่ไม่ติดลบ) นี่คือเหตุผลที่ Bit Shift ถูกใช้บ่อยมากในการคำนวณ
ที่เกี่ยวกับกำลังสอง เพราะ CPU ประมวลผลคำสั่ง shift เร็วกว่าการคูณ/หารทั่วไปมาก (แม้ Compiler
สมัยใหม่จะ optimize การคูณ/หารด้วยเลขยกกำลังสองให้กลายเป็น shift อัตโนมัติอยู่แล้วก็ตาม)

### Two's Complement: ทำไมคอมพิวเตอร์เก็บเลขลบแบบนี้

ก่อนจะเข้าใจ Right Shift กับตัวแปร `signed` ต้องเข้าใจก่อนว่าคอมพิวเตอร์เก็บ **เลขลบ** ใน
หน่วยความจำอย่างไร ระบบที่ใช้กันแทบทุกสถาปัตยกรรมในปัจจุบันคือ **Two's Complement**
(ส่วนเติมเต็มสอง) ซึ่งเป็นมาตรฐานบังคับตั้งแต่ C23 และเป็นวิธีที่คอมไพเลอร์แทบทุกตัวใช้มาตั้งแต่
ก่อนหน้านั้นแล้วในทางปฏิบัติ

วิธีหาค่า Two's Complement ของเลข `n` บิต: **กลับทุกบิต (`~`) แล้วบวก 1**

ตัวอย่างเช่นเลข `-5` ใน `signed char` (8 บิต):

```
5 (บวก)         = 0000 0101
กลับทุกบิต (~5) = 1111 1010
บวก 1           = 1111 1011   <-- นี่คือค่าของ -5 ใน 8 บิต
```

ตรวจสอบด้วยโค้ด:

```c
/* twos_complement.c */
#include <stdio.h>

static void print_binary8(unsigned char value) {
    for (int i = 7; i >= 0; i--) {
        putchar((value & (1u << i)) ? '1' : '0');
        if (i == 4) {
            putchar(' ');
        }
    }
    putchar('\n');
}

int main(void) {
    signed char positive = 5;
    signed char negative = -5;

    printf("5  = ");
    print_binary8((unsigned char)positive);
    printf("-5 = ");
    print_binary8((unsigned char)negative);
    return 0;
}
```

```
5  = 0000 0101
-5 = 1111 1011
```

**ทำไมต้องเก็บแบบนี้?** เพราะ Two's Complement ทำให้ CPU ใช้วงจรบวก (Adder) ตัวเดียวกัน
คำนวณทั้งการบวกเลขบวกและการลบเลข (ซึ่งก็คือการบวกเลขลบ) ได้โดยไม่ต้องมีวงจรพิเศษแยกสำหรับ
ลบ ทำให้ Hardware เรียบง่ายและเร็วขึ้นมาก และมีข้อดีอีกอย่างคือมีค่า "ศูนย์" อยู่แบบเดียว
(ไม่เหมือนระบบ Sign-magnitude เก่าที่มีทั้ง `+0` และ `-0` แยกกัน)

**บิตซ้ายสุด (Most Significant Bit)** ของเลข signed ในระบบ Two's Complement เรียกว่า
**Sign Bit** — ถ้าเป็น `0` แปลว่าค่าเป็นบวก (หรือศูนย์), ถ้าเป็น `1` แปลว่าค่าเป็นลบ

### Arithmetic Shift vs Logical Shift

เมื่อ Shift ค่าที่เป็น `signed` ไปทางขวา (`>>`) พฤติกรรมจะต่างจาก `unsigned`:

- **Logical Right Shift** (ใช้กับ `unsigned`): เติม `0` เข้ามาทางซ้ายเสมอ ไม่สนใจ sign bit
- **Arithmetic Right Shift** (ใช้กับ `signed` บนคอมไพเลอร์ส่วนใหญ่รวมถึง `gcc`): เติมค่า
  **Sign Bit เดิม** เข้ามาทางซ้าย เพื่อรักษาเครื่องหมาย (บวก/ลบ) ของค่าไว้

```c
#include <stdio.h>

int main(void) {
    int negative = -8;              /* signed: arithmetic shift */
    unsigned int as_unsigned = 0xFFFFFFF8u;   /* bit pattern เดียวกับ -8 ใน 32 บิต */

    printf("-8 >> 1 (signed)   = %d\n", negative >> 1);          /* -4 (รักษาเครื่องหมาย) */
    printf(" same bits >> 1 (unsigned) = %u\n", as_unsigned >> 1); /* เลขบวกขนาดใหญ่มาก */
    return 0;
}
```

```
-8 >> 1 (signed)   = -4
 same bits >> 1 (unsigned) = 2147483644
```

> **ข้อควรระวังสำคัญ**: มาตรฐาน C17 กำหนดว่า **การ shift ค่า signed ที่เป็นลบไปทางขวา
> (`>>`) เป็น Implementation-Defined Behavior** หมายความว่าผลลัพธ์ขึ้นอยู่กับคอมไพเลอร์/
> สถาปัตยกรรมแต่ละตัว แม้ในทางปฏิบัติเกือบทุกคอมไพเลอร์ (รวมถึง `gcc` และ `clang` บน x86/ARM)
> จะทำ Arithmetic Shift ให้ตามตัวอย่างข้างบน แต่หากต้องการโค้ดที่ portable และคาดเดาผลได้แน่นอน
> 100% ตามมาตรฐาน **ควรใช้ตัวแปร `unsigned` เสมอเมื่อทำงานกับ Bit Manipulation** และหลีกเลี่ยง
> การ shift ค่า signed ที่อาจติดลบ นอกจากนี้การ shift ด้วยจำนวนบิตที่มากกว่าหรือเท่ากับขนาดของ
> type (เช่น `int` 32 บิตแล้ว shift 32 บิตขึ้นไป) ถือเป็น **Undefined Behavior** เต็มรูปแบบ
> ต้องหลีกเลี่ยงเด็ดขาด

---

## 15.3 Bit Flag: เก็บสถานะหลายอย่างในตัวแปรเดียว (Step 115)

ปัญหาที่พบบ่อยคือการต้องเก็บ "สถานะแบบเปิด/ปิด" (Boolean State) หลายๆ อย่างพร้อมกัน เช่น
สถานะของผู้เล่นในเกม (มีชีวิต, ล่องหน, บินได้, มีเกราะ) ถ้าใช้ตัวแปร `bool` แยกทีละตัวจะกิน
หน่วยความจำมากและส่งผ่านฟังก์ชันลำบาก **Bit Flag** แก้ปัญหานี้โดยใช้ **แต่ละบิต** ของตัวแปร
จำนวนเต็มตัวเดียวแทนสถานะหนึ่งอย่าง

### กำหนดค่า Flag ด้วยเลขยกกำลังสอง (แต่ละ Flag ครองบิตของตัวเอง)

```c
/* player_flags.h */
#ifndef PLAYER_FLAGS_H
#define PLAYER_FLAGS_H

#define FLAG_ALIVE      (1u << 0)   /* 0000 0001 = 1  */
#define FLAG_INVISIBLE  (1u << 1)   /* 0000 0010 = 2  */
#define FLAG_FLYING     (1u << 2)   /* 0000 0100 = 4  */
#define FLAG_ARMORED    (1u << 3)   /* 0000 1000 = 8  */
#define FLAG_POISONED   (1u << 4)   /* 0001 0000 = 16 */

#endif /* PLAYER_FLAGS_H */
```

การเขียน `(1u << n)` แทนที่จะเขียนเลขตรงๆ (`1, 2, 4, 8, 16`) ช่วยให้อ่านง่ายและเห็นทันทีว่า
แต่ละ Flag ครองบิตตำแหน่งที่เท่าไหร่ โดยแต่ละ Flag ต้องมี **บิตเป็น 1 อยู่ตำแหน่งเดียวเท่านั้น
และไม่ซ้ำกับ Flag อื่น** เพื่อให้รวมกันแล้วไม่ชนกัน

### รวมหลาย Flag ด้วย `|` และตรวจสอบด้วย `&`

```c
/* player_flags_demo.c */
#include <stdio.h>
#include "player_flags.h"

int main(void) {
    unsigned int player_state = FLAG_ALIVE | FLAG_FLYING;   /* มีชีวิต + บินได้ */

    printf("player_state = 0x%02X\n", player_state);

    if (player_state & FLAG_ALIVE) {
        printf("- ผู้เล่นยังมีชีวิตอยู่\n");
    }
    if (player_state & FLAG_INVISIBLE) {
        printf("- ผู้เล่นล่องหน\n");
    } else {
        printf("- ผู้เล่นไม่ได้ล่องหน\n");
    }
    if (player_state & FLAG_FLYING) {
        printf("- ผู้เล่นกำลังบิน\n");
    }

    return 0;
}
```

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 player_flags_demo.c -o player_flags_demo
./player_flags_demo
```

```
player_state = 0x05
- ผู้เล่นยังมีชีวิตอยู่
- ผู้เล่นไม่ได้ล่องหน
- ผู้เล่นกำลังบิน
```

`player_state` เก็บสถานะทั้งหมด 5 อย่างได้ในตัวแปรเดียวขนาด 4 ไบต์ (หรือแม้แต่ 1 ไบต์ถ้าใช้
`unsigned char` เนื่องจากมี Flag แค่ 5 ตัว) เทียบกับการใช้ `bool` 5 ตัวแยกกันที่กิน 5 ไบต์
(หรือมากกว่านั้นเพราะ Padding ของ struct) และที่สำคัญกว่าเรื่องขนาดคือการส่งผ่านค่าทั้งหมด
ไปยังฟังก์ชันอื่นทำได้ในพารามิเตอร์ตัวเดียว

---

## 15.4 ฟังก์ชัน Set / Clear / Toggle / Check Bit (Step 116)

แทนที่จะเขียน `& |` ตรงๆ กระจายอยู่ทั่วโค้ด (ซึ่งอ่านยากและเสี่ยงพิมพ์ผิด) ควรห่อการกระทำ
พื้นฐานทั้ง 4 อย่างของ Bit Flag ไว้เป็นฟังก์ชันที่ตั้งชื่อสื่อความหมายชัดเจน:

```c
/* bitops.h */
#ifndef BITOPS_H
#define BITOPS_H

static inline unsigned int bit_set(unsigned int value, unsigned int flag) {
    return value | flag;              /* เปิดบิตที่ระบุ (ไม่ยุ่งกับบิตอื่น) */
}

static inline unsigned int bit_clear(unsigned int value, unsigned int flag) {
    return value & ~flag;             /* ปิดบิตที่ระบุ (ไม่ยุ่งกับบิตอื่น) */
}

static inline unsigned int bit_toggle(unsigned int value, unsigned int flag) {
    return value ^ flag;              /* สลับบิตที่ระบุ (0->1, 1->0) */
}

static inline int bit_check(unsigned int value, unsigned int flag) {
    return (value & flag) != 0;       /* คืน 1 ถ้าบิตที่ระบุถูกเปิดอยู่ */
}

#endif /* BITOPS_H */
```

### อธิบายกลไกของแต่ละฟังก์ชัน

**`bit_set`**: ใช้ `|` (OR) เพราะ `x | 1 = 1` เสมอ (บังคับให้บิตนั้นเป็น 1) ในขณะที่ `x | 0 = x`
(บิตอื่นที่ mask เป็น 0 จะไม่ถูกกระทบเลย) จึงเปิดเฉพาะบิตที่ต้องการโดยไม่แตะบิตอื่น

**`bit_clear`**: ใช้ `& ~flag` โดย `~flag` จะกลับบิตทุกตัวของ mask (บิตที่ต้องการปิดจะกลายเป็น
0 ส่วนบิตอื่นทั้งหมดกลายเป็น 1) แล้ว `&` กับค่าเดิม จะทำให้บิตที่ต้องการปิดกลายเป็น 0 แน่นอน
(เพราะ `x & 0 = 0`) ในขณะที่บิตอื่นยังคงค่าเดิมไว้ (เพราะ `x & 1 = x`)

**`bit_toggle`**: ใช้ `^` (XOR) เพราะ `x ^ 1` จะกลับค่าบิตนั้นเสมอ (0→1, 1→0) ในขณะที่
`x ^ 0 = x` ทำให้บิตอื่นไม่ถูกกระทบ

**`bit_check`**: ใช้ `&` เพื่อ mask เอาเฉพาะบิตที่สนใจออกมา แล้วเทียบว่าไม่เท่ากับ `0`
(สำคัญ: ต้องเทียบกับ `!= 0` ไม่ใช่ `== flag` เพราะถ้า `flag` มีหลายบิตพร้อมกัน การเทียบ `== flag`
จะเข้มงวดเกินไป)

ทดสอบใช้งานจริง:

```c
/* bitops_demo.c */
#include <stdio.h>
#include "bitops.h"
#include "player_flags.h"

int main(void) {
    unsigned int state = 0;   /* เริ่มต้นไม่มี flag ใดเปิดเลย */

    state = bit_set(state, FLAG_ALIVE);
    state = bit_set(state, FLAG_ARMORED);
    printf("หลัง set ALIVE, ARMORED: 0x%02X\n", state);

    printf("เช็ค ALIVE: %d\n", bit_check(state, FLAG_ALIVE));       /* 1 */
    printf("เช็ค FLYING: %d\n", bit_check(state, FLAG_FLYING));     /* 0 */

    state = bit_toggle(state, FLAG_FLYING);   /* เปิด FLYING */
    printf("หลัง toggle FLYING: 0x%02X\n", state);

    state = bit_clear(state, FLAG_ARMORED);   /* ปิด ARMORED */
    printf("หลัง clear ARMORED: 0x%02X\n", state);

    return 0;
}
```

```
หลัง set ALIVE, ARMORED: 0x09
เช็ค ALIVE: 1
เช็ค FLYING: 0
หลัง toggle FLYING: 0x0D
หลัง clear ARMORED: 0x05
```

การใช้ `static inline function` แทน Macro (เทียบกับที่เรียนใน Part 14 เรื่องอันตรายของ Macro
Function) ทำให้โค้ดชุดนี้ได้ทั้ง **Type Checking** จาก Compiler และปลอดภัยจากปัญหา Side-effect
ของพารามิเตอร์ เพราะพารามิเตอร์ของฟังก์ชันจริงจะถูกประเมินค่าเพียงครั้งเดียวเสมอ

---

## 15.5 Bit Field ใน `struct` (Step 117)

นอกจากการใช้ Bit Flag กับตัวแปรจำนวนเต็มธรรมดา ภาษา C ยังมีฟีเจอร์ **Bit Field** ที่ให้กำหนด
ขนาด (จำนวนบิต) ของแต่ละ Field ใน `struct` ได้โดยตรง ทำให้ Compiler จัดสรรพื้นที่หน่วยความจำ
แบบประหยัดที่สุดเท่าที่ทำได้

```c
/* status_register.c */
#include <stdio.h>

typedef struct {
    unsigned int is_alive    : 1;   /* ใช้แค่ 1 บิต: 0 หรือ 1 */
    unsigned int is_flying   : 1;
    unsigned int is_armored  : 1;
    unsigned int health_level : 4;  /* ใช้ 4 บิต: เก็บค่า 0-15 ได้ */
    unsigned int reserved    : 1;   /* บิตสำรอง ยังไม่ใช้งาน */
} PlayerStatus;

int main(void) {
    PlayerStatus p = {0};   /* กำหนดค่าเริ่มต้นทุก field เป็น 0 */

    p.is_alive = 1;
    p.is_flying = 0;
    p.health_level = 12;

    printf("is_alive    = %u\n", p.is_alive);
    printf("is_flying   = %u\n", p.is_flying);
    printf("health_level = %u\n", p.health_level);
    printf("ขนาดของ struct ทั้งหมด = %zu ไบต์\n", sizeof(PlayerStatus));

    return 0;
}
```

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 status_register.c -o status_register
./status_register
```

```
is_alive    = 1
is_flying   = 0
health_level = 12
ขนาดของ struct ทั้งหมด = 4 ไบต์
```

จะเห็นว่าทั้ง 5 Field รวมกันใช้แค่ `1+1+1+4+1 = 8` บิต แต่ `struct` ทั้งก้อนใช้พื้นที่จริง 4 ไบต์
(32 บิต) เพราะ Compiler จัดสรรพื้นที่เป็นหน่วย `unsigned int` (ประเภทที่ใช้ประกาศ Bit Field)
เต็มก้อนเสมอ ไม่ได้ใช้แค่ 1 ไบต์ตามที่หลายคนเข้าใจผิด — การประหยัดพื้นที่ที่แท้จริงของ Bit
Field จะเห็นผลชัดเมื่อมีหลาย instance จำนวนมาก หรือเมื่อรวม field เข้าไปในพื้นที่เดียวกันได้
มากกว่าจำนวนไบต์ที่ต้องใช้หากประกาศแยกเป็นตัวแปรปกติ

### ข้อจำกัดสำคัญของ Bit Field ที่ต้องรู้ก่อนใช้งานจริง

1. **ลำดับการจัดเรียงบิต (Bit Order) เป็น Implementation-Defined** — มาตรฐาน C ไม่ได้กำหนด
   ตายตัวว่า Compiler จะจัดเรียง field แรกไว้ที่บิตซ้ายสุดหรือขวาสุดของหน่วยความจำ ทำให้
   โครงสร้างเดียวกันอาจมี memory layout ต่างกันเมื่อ compile ด้วยคอมไพเลอร์หรือสถาปัตยกรรม
   ต่างกัน **ห้ามใช้ Bit Field เพื่อจับคู่กับ Binary Format ที่ต้องตรงกันข้ามเครื่อง** (เช่น
   Network Protocol หรือไฟล์ Binary ที่ต้องอ่านข้ามแพลตฟอร์ม) ให้ใช้ Bitwise Operator กับ
   ตัวแปรปกติแทนสำหรับกรณีนั้น เพราะเราควบคุมตำแหน่งบิตได้แน่นอน 100%
2. **ไม่สามารถใช้ `&` (Address-of) กับ Bit Field ได้** — เพราะ Bit Field ไม่มี address ที่
   ชี้ได้แบบไบต์ปกติ (มันอาจไม่ได้ตรงกับขอบเขตไบต์เลย)
3. **การใช้ `int` (signed) เป็น Bit Field มีความหมายไม่ชัดเจนพอ** — ควรใช้ `unsigned int`
   เสมอสำหรับ Bit Field เว้นแต่ต้องการค่าติดลบจริงๆ และเข้าใจผลกระทบของ Sign Bit ในพื้นที่
   บิตที่จำกัดเป็นอย่างดี
4. **Padding ระหว่าง Field อาจเกิดขึ้นได้** ถ้า field ถัดไปมีขนาดใหญ่จนไม่พอดีกับพื้นที่ที่เหลือ
   ของหน่วยจัดเก็บปัจจุบัน Compiler อาจแทรกช่องว่างหรือขึ้นหน่วยใหม่ ทำให้ `sizeof` ไม่เท่ากับ
   ผลรวมบิตที่คำนวณเองอย่างที่คาดหวัง

> **แนวทางเลือกใช้**: Bit Field เหมาะกับการจัดกลุ่ม flag/สถานะภายในโปรแกรมเดียวที่ compile
> ด้วย compiler เดียวกันเสมอ (เช่น struct เก็บ config ภายใน หรือ struct เก็บสถานะเกม) ส่วน
> Bitwise Operator กับตัวแปรปกติ (`unsigned int` + `#define FLAG_X (1u << n)`) เหมาะกับกรณีที่
> ต้องการควบคุมตำแหน่งบิตอย่างแน่นอน พกพาข้ามระบบได้ 100% เช่น Network Protocol, File Format,
> หรือการสื่อสารกับ Hardware/Register โดยตรง

---

## 15.6 ตัวอย่างจริง: ระบบ Permission Flag แบบ Unix-Style (Step 118)

ระบบไฟล์ของ Unix/Linux ใช้ Bit Flag ในการเก็บสิทธิ์การเข้าถึงไฟล์ (Permission) มาตั้งแต่ยุคแรก
เช่นคำสั่ง `chmod 754 file.txt` ที่ทุกคนคุ้นเคย แท้จริงแล้ว `7`, `5`, `4` แต่ละหลักคือค่า
Bit Flag ของ **read (4) + write (2) + execute (1)** รวมกัน — มาลองสร้างระบบคล้ายกันขึ้นเอง

```c
/* permission_system.c */
#include <stdio.h>

#define PERM_READ    (1u << 2)   /* 100 = 4 */
#define PERM_WRITE   (1u << 1)   /* 010 = 2 */
#define PERM_EXECUTE (1u << 0)   /* 001 = 1 */

typedef unsigned int Permission;

static Permission perm_grant(Permission current, Permission flag) {
    return current | flag;
}

static Permission perm_revoke(Permission current, Permission flag) {
    return current & ~flag;
}

static int perm_has(Permission current, Permission flag) {
    return (current & flag) == flag;
}

static void perm_print(Permission p) {
    printf("%c%c%c (0%o)\n",
           perm_has(p, PERM_READ)    ? 'r' : '-',
           perm_has(p, PERM_WRITE)   ? 'w' : '-',
           perm_has(p, PERM_EXECUTE) ? 'x' : '-',
           p);
}

int main(void) {
    Permission owner_perm = 0;

    owner_perm = perm_grant(owner_perm, PERM_READ);
    owner_perm = perm_grant(owner_perm, PERM_WRITE);
    owner_perm = perm_grant(owner_perm, PERM_EXECUTE);
    printf("สิทธิ์เริ่มต้นของเจ้าของไฟล์: ");
    perm_print(owner_perm);   /* rwx (07) */

    owner_perm = perm_revoke(owner_perm, PERM_WRITE);
    printf("หลังถอนสิทธิ์เขียน:         ");
    perm_print(owner_perm);   /* r-x (05) */

    printf("มีสิทธิ์อ่านหรือไม่: %s\n",
           perm_has(owner_perm, PERM_READ) ? "มี" : "ไม่มี");
    printf("มีสิทธิ์เขียนหรือไม่: %s\n",
           perm_has(owner_perm, PERM_WRITE) ? "มี" : "ไม่มี");

    /* จำลอง permission เต็มรูปแบบ 3 ระดับแบบ chmod: owner, group, other */
    Permission group_perm = PERM_READ;
    Permission other_perm = PERM_READ;
    unsigned int combined_octal =
        (owner_perm << 6) | (group_perm << 3) | other_perm;
    printf("รวมแบบ chmod-style: 0%o\n", combined_octal);   /* 0544 */

    return 0;
}
```

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 permission_system.c -o permission_system
./permission_system
```

```
สิทธิ์เริ่มต้นของเจ้าของไฟล์: rwx (07)
หลังถอนสิทธิ์เขียน:         r-x (05)
มีสิทธิ์อ่านหรือไม่: มี
มีสิทธิ์เขียนหรือไม่: ไม่มี
รวมแบบ chmod-style: 0544
```

ตัวอย่างนี้แสดงให้เห็นว่า **แนวคิดเดียวกัน** ที่เรียนมาตลอด Part นี้ (`|` สำหรับให้สิทธิ์,
`& ~` สำหรับถอนสิทธิ์, `&` สำหรับตรวจสอบ) ถูกใช้งานจริงในระบบปฏิบัติการระดับโลกมาหลายสิบปี
และการรวม permission ของ 3 กลุ่ม (owner/group/other) เข้าเป็นเลขฐานแปดตัวเดียวก็ใช้เพียง
Left Shift (`<<`) เพื่อจัดตำแหน่งบิตของแต่ละกลุ่มให้ไม่ทับกัน แล้ว `|` รวมเข้าด้วยกัน

---

## 15.7 Bit Trick ที่พบบ่อยในงานจริง (Step 119)

นอกจาก set/clear/toggle/check พื้นฐาน ยังมีเทคนิคระดับบิตที่ใช้บ่อยมากในงาน Systems
Programming และการแข่งขันเขียนโปรแกรม (Competitive Programming) ที่ควรรู้จักไว้

### ตรวจสอบว่าเป็นเลขคู่/คี่ด้วย `&` แทน `%`

```c
#include <stdio.h>
#include <stdbool.h>

static bool is_odd(unsigned int n) {
    return (n & 1u) == 1u;   /* บิตขวาสุดเป็น 1 แปลว่าเป็นเลขคี่ เร็วกว่า n % 2 */
}

int main(void) {
    for (unsigned int i = 0; i < 6; i++) {
        printf("%u เป็นเลข%s\n", i, is_odd(i) ? "คี่" : "คู่");
    }
    return 0;
}
```

### ตรวจสอบว่าเป็นเลขยกกำลังสองหรือไม่

เลขยกกำลังสอง (`1, 2, 4, 8, 16, ...`) จะมีบิตที่เป็น 1 อยู่เพียง **บิตเดียว** เสมอ เทคนิคคลาสสิก
คือใช้ `n & (n - 1)` ซึ่งจะ "ลบบิต 1 ตัวขวาสุด" ออกไป ถ้าผลลัพธ์เป็น `0` แปลว่าเดิมมีบิต 1
อยู่แค่ตัวเดียว จึงเป็นเลขยกกำลังสอง

```c
#include <stdio.h>
#include <stdbool.h>

static bool is_power_of_two(unsigned int n) {
    return n != 0 && (n & (n - 1)) == 0;
}

int main(void) {
    unsigned int values[] = {1, 2, 3, 4, 15, 16, 1024, 1023};
    size_t count = sizeof(values) / sizeof(values[0]);

    for (size_t i = 0; i < count; i++) {
        printf("%u %s เลขยกกำลังสอง\n",
               values[i], is_power_of_two(values[i]) ? "เป็น" : "ไม่เป็น");
    }
    return 0;
}
```

```
1 เป็น เลขยกกำลังสอง
2 เป็น เลขยกกำลังสอง
3 ไม่เป็น เลขยกกำลังสอง
4 เป็น เลขยกกำลังสอง
15 ไม่เป็น เลขยกกำลังสอง
16 เป็น เลขยกกำลังสอง
1024 เป็น เลขยกกำลังสอง
1023 ไม่เป็น เลขยกกำลังสอง
```

### นับจำนวนบิตที่เป็น 1 (Population Count) ด้วย Brian Kernighan's Algorithm

```c
#include <stdio.h>

static int count_set_bits(unsigned int n) {
    int count = 0;
    while (n != 0) {
        n &= (n - 1);   /* ลบบิต 1 ตัวขวาสุดออกทีละครั้ง */
        count++;
    }
    return count;
}

int main(void) {
    unsigned int value = 0x2F;   /* 0010 1111 มีบิต 1 อยู่ 5 ตัว */
    printf("0x%X มีบิตที่เป็น 1 จำนวน %d บิต\n", value, count_set_bits(value));
    return 0;
}
```

```
0x2F มีบิตที่เป็น 1 จำนวน 5 บิต
```

เทคนิค `n &= (n - 1)` ทำงานเพราะ `n - 1` จะเปลี่ยนบิต 1 ตัวขวาสุดของ `n` ให้เป็น 0 และเปลี่ยน
บิตทั้งหมดทางขวาของมัน (ที่เดิมเป็น 0) ให้กลายเป็น 1 ทั้งหมด เมื่อนำมา `&` กับ `n` เดิม บิตที่
เพิ่งกลายเป็น 1 เหล่านั้นจะถูกตัดทิ้งไป เหลือแค่การลบบิต 1 ตัวขวาสุดออกไปตัวเดียว ทำให้วน
ลูปเพียง "จำนวนบิตที่เป็น 1" ครั้งเท่านั้น (เร็วกว่าการวนตรวจสอบทีละบิตทั้ง 32 บิตเสมอไป)

### สลับค่าตัวแปรด้วย XOR โดยไม่ใช้ตัวแปรชั่วคราว (และทำไมไม่ควรใช้ในโค้ดจริง)

```c
#include <stdio.h>

int main(void) {
    int a = 5;
    int b = 9;

    a = a ^ b;
    b = a ^ b;   /* b = (a^b) ^ b = a (เดิม) */
    a = a ^ b;   /* a = (a^b) ^ a(เดิม) = b (เดิม) */

    printf("a = %d, b = %d\n", a, b);   /* a = 9, b = 5 */
    return 0;
}
```

เทคนิคนี้เป็นที่นิยมในการสัมภาษณ์งานและ Competitive Programming เพราะดูฉลาดหลักแหลม แต่ใน
**โค้ดที่ใช้งานจริงไม่ควรใช้เทคนิคนี้** เพราะ (1) อ่านยากกว่า `int tmp = a; a = b; b = tmp;`
มาก โดยไม่ได้เร็วกว่าจริงในทางปฏิบัติ (Compiler สมัยใหม่ optimize การสลับค่าแบบใช้ตัวแปร
ชั่วคราวได้ดีอยู่แล้ว) และ (2) **ถ้า `a` กับ `b` เป็นตัวแปรตัวเดียวกัน (aliasing) เช่นเป็น
element เดียวกันในอาเรย์ที่ถูกอ้างถึงด้วย index ต่างกันแต่ชี้ไปที่ตำแหน่งเดียวกัน** ค่าจะกลาย
เป็น `0` ผิดพลาดทันที (เพราะ `a ^ a = 0` เสมอ) เทคนิคนี้จึงเหมาะสำหรับ "รู้ไว้เพื่อเข้าใจ" มากกว่า
"นำไปใช้จริง"

| เทคนิค | Time Complexity | เหมาะกับ |
|---|---|---|
| `n & 1` เช็คคู่/คี่ | O(1) | แทน `n % 2` เมื่อทำงานกับ `unsigned` |
| `n & (n-1) == 0` เช็คเลขยกกำลังสอง | O(1) | ตรวจสอบขนาด buffer, hash table |
| Brian Kernighan popcount | O(จำนวนบิตที่เป็น 1) | นับ flag ที่เปิดอยู่, Hamming weight |
| XOR swap | O(1) แต่ไม่แนะนำให้ใช้จริง | ใช้เพื่อความเข้าใจเท่านั้น |

---

## 15.8 เมื่อไหร่ควร (และไม่ควร) ใช้ Bit Manipulation ในงานจริง (Step 120)

Bit Manipulation เป็นเครื่องมือที่ทรงพลังมาก แต่การใช้พร่ำเพรื่อในจุดที่ไม่จำเป็นจะทำให้โค้ด
อ่านยากขึ้นโดยไม่ได้ประโยชน์ด้านประสิทธิภาพที่วัดผลได้จริง ควรพิจารณาปัจจัยต่อไปนี้ก่อนตัดสินใจ

### ควรใช้เมื่อ

- **ต้องเก็บ Flag/สถานะจำนวนมากอย่างกระชับ** เช่น permission system, feature flag,
  configuration option ที่มีหลายสิบตัวเลือก
- **ทำงานใกล้ Hardware โดยตรง** เช่นการเขียนโปรแกรมควบคุม register ของ microcontroller,
  การ parse binary protocol/file format ที่กำหนดตำแหน่งบิตไว้ตายตัว
- **ต้องการประสิทธิภาพสูงสุดในจุดที่วัดผลแล้วว่าเป็นคอขวดจริง** (Hot Path) เช่นใน Hash Function,
  Compression Algorithm, Cryptography
- **ทำงานกับข้อมูลจำนวนมหาศาลที่ต้องประหยัดหน่วยความจำทุกไบต์** เช่น Bitmap Index ใน Database,
  Bloom Filter

### ไม่ควรใช้ (หรือควรระวังเป็นพิเศษ) เมื่อ

- **โค้ดทำงานกับข้อมูลจำนวนน้อยและความอ่านง่ายสำคัญกว่า** — การเขียน `if (count % 2 == 0)`
  ชัดเจนกว่า `if ((count & 1) == 0)` มากสำหรับคนอ่านทั่วไป ควรใช้ Bit Trick เฉพาะเมื่อวัดผล
  (Profile) แล้วพบว่าเป็นคอขวดจริงๆ เท่านั้น
- **ต้องการโค้ดที่ Portable ข้ามสถาปัตยกรรมแบบเข้มงวด** — ต้องระวังเรื่อง **Endianness**
  (ลำดับไบต์ในหน่วยความจำ ซึ่งจะเรียนละเอียดใน Part ถัดๆ ไปที่เกี่ยวกับ Networking) และเรื่อง
  Implementation-Defined Behavior ของการ shift ค่า signed ตามที่กล่าวไปในหัวข้อ 15.2
- **ทีมงานส่วนใหญ่ไม่คุ้นเคยกับ Bit Manipulation** — ควรเขียนฟังก์ชันห่อ (Wrapper Function)
  ที่ตั้งชื่อสื่อความหมายชัดเจนเสมอ (เหมือนตัวอย่าง `bit_set`/`bit_clear`/`perm_grant` ในบทนี้)
  แทนการเขียน `&`/`|`/`^` ตรงๆ กระจายอยู่ทั่วโค้ด เพื่อให้คนอื่นอ่านโค้ดแล้วเข้าใจเจตนาได้ทันที
  โดยไม่ต้องถอดรหัสตรรกะระดับบิตเอง

> หลักคิดสำคัญ: **"เขียนโค้ดให้อ่านง่ายก่อน แล้วค่อย optimize เฉพาะจุดที่วัดผลแล้วจำเป็นจริงๆ"**
> Bit Manipulation เป็นเครื่องมือที่ควรมีติดตัวและเข้าใจอย่างลึกซึ้ง แต่ไม่ใช่สิ่งที่ต้องใช้
> ทุกครั้งที่ทำได้

---

## ข้อผิดพลาดที่พบบ่อย (Common Pitfalls)

1. **สับสนระหว่าง `&`/`|` (Bitwise) กับ `&&`/`||` (Logical)** — `if (flags & FLAG_READ)`
   ตรวจสอบบิตอย่างถูกต้อง แต่ `if (flags && FLAG_READ)` จะตรวจสอบแค่ว่า `flags` ไม่เท่ากับ 0
   **และ** `FLAG_READ` ไม่เท่ากับ 0 (ซึ่งมักเป็นจริงเสมอถ้า `FLAG_READ` ไม่ใช่ 0) ทำให้เงื่อนไข
   ผิดพลาดแบบเนียนมากและหา bug ยาก
2. **ใช้ `signed int` เป็นตัวแปรเก็บ Bit Flag** — เมื่อ shift บิตซ้ายสุด (sign bit) อาจเกิด
   Undefined Behavior หรือพฤติกรรมที่คาดเดายาก ควรใช้ `unsigned int` เสมอสำหรับงาน Bit
   Manipulation
3. **Shift เกินขนาดของ Type** — เช่น `1 << 32` บน `int` 32 บิต เป็น Undefined Behavior เต็ม
   รูปแบบ (ไม่ใช่แค่ได้ผลลัพธ์ผิด) ต้องตรวจสอบให้แน่ใจว่าจำนวนบิตที่ shift น้อยกว่าขนาดของ
   type เสมอ และถ้าต้องการ 64 บิตให้ใช้ `1ULL << n` แทน `1 << n`
4. **เทียบผลลัพธ์ `&` กับค่าที่ไม่ใช่ mask เดิม** — เขียน `if ((flags & MULTI_BIT_MASK) == 1)`
   ทั้งที่ `MULTI_BIT_MASK` มีหลายบิต ต้องเทียบกับ `MULTI_BIT_MASK` เอง (`== MULTI_BIT_MASK`)
   ไม่ใช่ `1`
5. **คาดหวังว่า Bit Field มี Memory Layout ตรงกันทุกคอมไพเลอร์** — ตามที่อธิบายในหัวข้อ 15.5
   ห้ามใช้ Bit Field เพื่อจับคู่กับ Binary Format ข้ามระบบหรือ Network Protocol
6. **ลืมว่า `~` กลับบิตทุกบิตของทั้ง type ไม่ใช่แค่ mask เดิม** — `~FLAG_READ` เมื่อ `FLAG_READ`
   เป็น `unsigned char` ที่ถูก promote เป็น `int` จะได้ผลลัพธ์กลับบิตของ `int` ทั้ง 32 บิต
   ถ้าเอาไปใช้กับตัวแปรที่มีขนาดเล็กกว่าโดยไม่ cast ให้ถูกต้อง อาจได้ผลลัพธ์ที่ไม่ตรงกับที่คาดหวัง

---

## แบบฝึกหัดท้ายบท

1. เขียนฟังก์ชัน `void print_binary32(unsigned int value)` ที่พิมพ์เลขจำนวนเต็ม 32 บิตออกมา
   เป็นเลขฐานสองครบทุกบิต (คั่นทุก 4 บิตด้วยช่องว่างเพื่อให้อ่านง่าย)
2. ออกแบบระบบ Bit Flag สำหรับสถานะออเดอร์ในร้านค้าออนไลน์ ที่มีสถานะ `PAID`, `SHIPPED`,
   `DELIVERED`, `CANCELLED`, `REFUNDED` แล้วเขียนฟังก์ชันตรวจสอบว่าออเดอร์หนึ่งๆ "จ่ายเงินแล้ว
   แต่ยังไม่ถูกยกเลิก" หรือไม่
3. เขียนฟังก์ชัน `unsigned int reverse_bits(unsigned char value)` ที่กลับลำดับบิตทั้ง 8 บิต
   ของค่า `value` (เช่น `0b10110000` กลายเป็น `0b00001101`)
4. ใช้เทคนิค `n & (n-1)` เขียนฟังก์ชันตรวจสอบว่าตัวเลขที่ผู้ใช้ป้อนเข้ามาเป็นเลขยกกำลังสอง
   หรือไม่ พร้อมทดสอบกับค่าอย่างน้อย 5 ค่า
5. เขียน `struct` ที่ใช้ Bit Field เก็บข้อมูลวันที่แบบกระชับ (`day : 5`, `month : 4`,
   `year : 12` โดยเก็บปี ค.ศ. แบบ offset จาก 2000) แล้วพิมพ์ `sizeof` ของ struct นั้นออกมา
6. เขียนโปรแกรมที่รับตัวเลข 2 จำนวนจากผู้ใช้ แล้วนับว่าตัวเลขทั้งสองมี "บิตที่ต่างกัน" กี่ตำแหน่ง
   (Hamming Distance) โดยใช้ `^` ร่วมกับเทคนิคนับบิตที่เรียนในหัวข้อ 15.7

### แนวทางเฉลยข้อ 1

```c
/* print_binary32.c */
#include <stdio.h>

void print_binary32(unsigned int value) {
    for (int i = 31; i >= 0; i--) {
        putchar((value & (1u << i)) ? '1' : '0');
        if (i % 4 == 0 && i != 0) {
            putchar(' ');
        }
    }
    putchar('\n');
}

int main(void) {
    print_binary32(0);
    print_binary32(1);
    print_binary32(255);
    print_binary32(0xDEADBEEFu);
    return 0;
}
```

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 print_binary32.c -o print_binary32
./print_binary32
```

```
0000 0000 0000 0000 0000 0000 0000 0000
0000 0000 0000 0000 0000 0000 0000 0001
0000 0000 0000 0000 0000 0000 1111 1111
1101 1110 1010 1101 1011 1110 1110 1111
```

### แนวทางเฉลยข้อ 6

```c
/* hamming_distance.c */
#include <stdio.h>

static int count_set_bits(unsigned int n) {
    int count = 0;
    while (n != 0) {
        n &= (n - 1);
        count++;
    }
    return count;
}

static int hamming_distance(unsigned int a, unsigned int b) {
    return count_set_bits(a ^ b);
}

int main(void) {
    unsigned int a, b;

    printf("ป้อนเลขจำนวนเต็มไม่ติดลบ 2 ค่า (คั่นด้วยช่องว่าง): ");
    if (scanf("%u %u", &a, &b) != 2) {
        printf("รูปแบบข้อมูลไม่ถูกต้อง\n");
        return 1;
    }

    printf("%u กับ %u มีบิตที่ต่างกัน %d ตำแหน่ง\n",
           a, b, hamming_distance(a, b));
    return 0;
}
```

```
ป้อนเลขจำนวนเต็มไม่ติดลบ 2 ค่า (คั่นด้วยช่องว่าง): 9 15
9 กับ 15 มีบิตที่ต่างกัน 2 ตำแหน่ง
```

อธิบาย: `9 = 1001`, `15 = 1111`, `9 ^ 15 = 0110` ซึ่งมีบิตที่เป็น 1 อยู่ 2 ตำแหน่ง (Hamming
Distance = 2) หลักการนี้ใช้จริงในหลายวงการ เช่นการตรวจสอบ Error Correction Code และการ
วัดความคล้ายกันของข้อมูลในบางอัลกอริทึม Machine Learning

---

## สรุปท้ายบท

ใน Part นี้เราได้:

- เข้าใจ Bitwise Operator ทั้ง 6 ตัว (`&`, `|`, `^`, `~`, `<<`, `>>`) พร้อมตารางค่าความจริงเต็มรูปแบบ
- เข้าใจ Two's Complement และที่มาของการเก็บเลขลบในหน่วยความจำ รวมถึงความแตกต่างระหว่าง
  Arithmetic Shift และ Logical Shift
- ออกแบบระบบ Bit Flag เพื่อเก็บสถานะหลายอย่างในตัวแปรเดียวได้อย่างมีประสิทธิภาพ
- เขียนฟังก์ชัน set/clear/toggle/check bit ที่ปลอดภัยและนำกลับมาใช้ซ้ำได้
- ใช้ Bit Field ใน struct พร้อมเข้าใจข้อจำกัดเรื่อง portability
- ประยุกต์ใช้ทุกแนวคิดสร้างระบบ Permission Flag แบบ Unix-style ได้จริง
- รู้จัก Bit Trick ที่ใช้บ่อย เช่นการนับบิต, ตรวจสอบเลขยกกำลังสอง, และเข้าใจข้อจำกัดของ XOR swap
- ตัดสินใจได้ว่าเมื่อไหร่ควรและไม่ควรใช้ Bit Manipulation ในงานจริง

ทักษะ Bit Manipulation ที่เรียนไปใน Part นี้จะกลับมาใช้ซ้ำอีกหลายครั้งตลอดหลักสูตร ตั้งแต่การ
เขียน Hash Table (Part 22), การเขียนโปรแกรมระดับ Systems บน Linux (Module C), ไปจนถึงงาน
Network Programming และ Embedded Systems ในช่วงท้ายหลักสูตร ใน **Part 16** เราจะเรียนรู้
**Error Handling ใน C** ซึ่งเป็นทักษะสำคัญที่จะทำให้โปรแกรมของเรา "พังอย่างสง่างาม" แทนที่จะ
พังแบบไม่มีคำเตือนเมื่อเจอสถานการณ์ที่ไม่คาดคิด

**ต่อไป:** [Part 16 — Error Handling ใน C](./part-016-error-handling.md)
