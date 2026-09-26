# Part 8: Pointer พื้นฐาน (ตอนที่ 1) (Step 57–64)

> Module A — รากฐานภาษา C (Foundations of C) | Part 8 จาก 125
> Step ที่ครอบคลุมใน Part นี้: Step 57–64
> Part ก่อนหน้า: [Part 7 — Array และ String ใน C](./part-007-arrays-strings.md) | Part ถัดไป: [Part 9 — Pointer ขั้นสูง (ตอนที่ 2)](./part-009-pointers-advanced.md)

## เป้าหมายการเรียนรู้ (Learning Objectives)

หลังจบ Part นี้ ผู้เรียนจะสามารถ:

1. อธิบายได้ว่า "Memory Address" คืออะไร และทำไม Pointer จึงเป็นแนวคิดที่ทรงพลังที่สุดอย่างหนึ่งของภาษา C
2. ใช้ตัวดำเนินการ `&` (address-of) และ `*` (dereference) ได้อย่างถูกต้องและเข้าใจความหมายจริงของทั้งสองตัว
3. ประกาศตัวแปร Pointer ได้อย่างปลอดภัย และไม่ตกหลุมพรางไวยากรณ์ที่มือใหม่พลาดบ่อยที่สุด
4. เข้าใจและใช้งาน `NULL` pointer ได้อย่างถูกต้อง พร้อมอธิบายอันตรายของ Wild Pointer และ Dangling Pointer
5. อธิบายความสัมพันธ์ระหว่าง Array กับ Pointer และปรากฏการณ์ "Array Decay" ได้
6. เขียนฟังก์ชันที่รับ Pointer เป็นพารามิเตอร์เพื่อจำลองพฤติกรรม Pass-by-Reference ได้
7. วาดและอ่าน Diagram ของ Memory Layout เพื่อ debug ปัญหาเกี่ยวกับ Pointer ได้ด้วยตัวเอง

---

## 8.1 หน่วยความจำและ Address คืออะไร (Step 57)

ก่อนจะเข้าใจ Pointer ต้องเข้าใจก่อนว่าเวลาโปรแกรม C รันอยู่ ตัวแปรทุกตัวไม่ได้ลอยอยู่เฉยๆ
ในอากาศ แต่ถูกเก็บอยู่ใน **RAM (หน่วยความจำหลัก)** จริงๆ และ RAM ก็เปรียบเสมือนตึกที่มีห้อง
เรียงกันเป็นแถวยาวนับล้านๆ ห้อง แต่ละห้องมี "เลขที่ห้อง" ของตัวเองที่ไม่ซ้ำกันเลย เลขที่ห้องนี้
เราเรียกว่า **Memory Address (แอดเดรส)**

เวลาเราประกาศตัวแปร เช่น `int age = 25;` สิ่งที่เกิดขึ้นจริงคือ:

1. Compiler จองห้องในหน่วยความจำขนาด 4 ไบต์ (สำหรับ `int` บนเครื่องส่วนใหญ่ในปัจจุบัน)
2. เขียนค่า `25` ลงในห้องนั้น
3. ตั้งชื่อเล่นให้ห้องนั้นว่า `age` เพื่อให้เราไม่ต้องจำเลขที่ห้อง (address) ที่ยุ่งยากได้เอง

พูดง่ายๆ **ตัวแปรคือ "ชื่อเล่น" ที่มนุษย์อ่านง่ายของห้องหน่วยความจำห้องหนึ่ง** ส่วน Pointer
คือกลไกที่ทำให้เรา "รู้และเก็บเลขที่ห้อง" (address) นั้นไว้ได้โดยตรง แทนที่จะรู้แค่ชื่อเล่น

ลองพิมพ์ address ของตัวแปรจริงๆ ดูก่อน:

```c
/* addr_demo.c
 * สาธิตการดู memory address ของตัวแปรแต่ละชนิด
 */
#include <stdio.h>

int main(void) {
    int age = 25;
    double price = 99.50;
    char grade = 'A';

    printf("ตัวแปร age   เก็บค่า %d    อยู่ที่ address %p\n", age, (void *)&age);
    printf("ตัวแปร price เก็บค่า %.2f อยู่ที่ address %p\n", price, (void *)&price);
    printf("ตัวแปร grade เก็บค่า %c    อยู่ที่ address %p\n", grade, (void *)&grade);

    return 0;
}
```

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 addr_demo.c -o addr_demo
./addr_demo
```

ผลลัพธ์ตัวอย่าง (address จริงจะเปลี่ยนไปทุกครั้งที่รัน เพราะระบบปฏิบัติการสมัยใหม่ใช้
**ASLR — Address Space Layout Randomization** เพื่อความปลอดภัย):

```
ตัวแปร age   เก็บค่า 25    อยู่ที่ address 0x7ffd8a3c2b1c
ตัวแปร price เก็บค่า 99.50 อยู่ที่ address 0x7ffd8a3c2b10
ตัวแปร grade เก็บค่า A     อยู่ที่ address 0x7ffd8a3c2b0f
```

สังเกต 3 จุดสำคัญ:

- ต้องใช้ Format Specifier `%p` เพื่อพิมพ์ address และต้อง **cast เป็น `(void *)` เสมอ**
  ไม่เช่นนั้นจะได้ Warning เรื่อง type mismatch ภายใต้ `-Wpedantic`
- Address ที่ได้เป็นเลขฐาน 16 (Hexadecimal) เสมอ
- Address ของแต่ละตัวแปรต่างกันไม่เท่ากันคงที่ (ในตัวอย่างนี้ต่างกัน 12 และ 1 ไบต์)
  เพราะขนาดของแต่ละชนิดข้อมูลไม่เท่ากัน และ Compiler อาจแทรก Padding ระหว่างตัวแปรด้วย
  (จะอธิบายเรื่อง Padding แบบเต็มใน Part 10)

---

## 8.2 ตัวดำเนินการ `&` และ `*`: หัวใจของ Pointer (Step 58)

C มีตัวดำเนินการ 2 ตัวที่เป็นกุญแจสำคัญของเรื่อง Pointer ทั้งหมด:

| ตัวดำเนินการ | ชื่อเรียก | ความหมาย |
|---|---|---|
| `&x` | Address-of Operator | "ขอ address ของตัวแปร x" — คืนค่าเป็น address |
| `*p` | Dereference Operator | "ไปดู/แก้ค่าที่อยู่ ณ address ที่ p ชี้ไป" |

ทั้งสองตัวเป็น **ตัวดำเนินการที่ตรงข้ามกัน (inverse)**: ถ้า `p = &x` แล้ว `*p` จะเท่ากับ `x` เสมอ

```c
/* address_dereference.c */
#include <stdio.h>

int main(void) {
    int number = 42;
    int *ptr = &number;   /* ptr เก็บ "address ของ number" */

    printf("number             = %d\n", number);
    printf("&number            = %p\n", (void *)&number);
    printf("ptr                = %p\n", (void *)ptr);
    printf("*ptr (dereference) = %d\n", *ptr);

    /* แก้ค่าผ่าน pointer ได้โดยตรง */
    *ptr = 100;
    printf("หลังจาก *ptr = 100, number = %d\n", number);

    return 0;
}
```

```
number             = 42
&number            = 0x7ffd3c1a204c
ptr                = 0x7ffd3c1a204c
*ptr (dereference) = 42
หลังจาก *ptr = 100, number = 100
```

จุดที่ต้องเข้าใจให้แม่นคือ **`ptr` และ `&number` มีค่าเท่ากัน** (คือ address เดียวกัน) แต่
`*ptr` คือ "การเดินตาม address ไปเปิดดูของข้างใน" ซึ่งเป็นคนละอย่างกับตัว address เอง

**อุปมา**: ลองนึกภาพ `number` เป็นบ้านหลังหนึ่ง `&number` คือ "เลขที่บ้าน" ที่เขียนอยู่บนกระดาษ
ส่วน `ptr` คือกระดาษที่มีเลขที่บ้านนั้นเขียนอยู่ (ตัวมันเองก็เป็นวัตถุที่มี address ของตัวเองด้วย
เพราะ `ptr` ก็เป็นตัวแปรตัวหนึ่งเหมือนกัน!) และ `*ptr` คือการ "เดินไปตามเลขที่บ้านบนกระดาษ
แล้วเปิดประตูเข้าไปดูข้างใน"

> **สังเกต**: เครื่องหมาย `*` ในภาษา C ถูกใช้ 2 ความหมายที่ต่างกันโดยสิ้นเชิงตามบริบท:
> 1. ตอน **ประกาศตัวแปร**: `int *ptr;` แปลว่า "ptr เป็นตัวแปรชนิด pointer ไปยัง int"
> 2. ตอน **ใช้งานตัวแปรที่มีอยู่แล้ว**: `*ptr` แปลว่า "dereference ไปดูค่าที่ ptr ชี้อยู่"
>
> นี่คือจุดที่มือใหม่สับสนบ่อยที่สุด ต้องดูบริบทให้ดีว่ากำลังประกาศตัวแปรใหม่ หรือใช้ตัวแปรเดิม

---

## 8.3 การประกาศตัวแปร Pointer (Step 59)

รูปแบบการประกาศ Pointer คือ:

```c
ชนิดข้อมูล *ชื่อตัวแปร;
```

เช่น `int *pa;`, `double *pd;`, `char *pc;` — ความหมายคือ "pa เป็น pointer ที่ **ชี้ไปยัง**
ตัวแปรชนิด `int`" ชนิดข้อมูลที่อยู่หน้า `*` นี้สำคัญมาก เพราะมันบอก Compiler ว่า ถ้า dereference
ตัว pointer นี้ จะได้ข้อมูลชนิดอะไร และควรอ่าน/เขียนกี่ไบต์

```c
/* pointer_declare.c */
#include <stdio.h>

int main(void) {
    int a = 10;
    double b = 3.14;
    char c = 'Z';

    int *pa = &a;
    double *pb = &b;
    char *pc = &c;

    printf("pa ชี้ไปที่ int ค่า    = %d\n", *pa);
    printf("pb ชี้ไปที่ double ค่า = %.2f\n", *pb);
    printf("pc ชี้ไปที่ char ค่า   = %c\n", *pc);

    return 0;
}
```

### กับดักไวยากรณ์ที่พบบ่อยที่สุด

หลายคนเข้าใจผิดว่า `*` เป็นส่วนหนึ่งของ "ชนิดข้อมูล" แต่จริงๆ แล้ว `*` เป็นส่วนหนึ่งของ
"ชื่อตัวแปร" แต่ละตัวต่างหาก ทำให้เขียนแบบนี้แล้วเจอบั๊กเงียบๆ:

```c
/* multi_declare_pitfall.c
 * สาธิตกับดัก: ประกาศ pointer หลายตัวในบรรทัดเดียว
 */
#include <stdio.h>

int main(void) {
    int *pa, b;   /* อันตราย: pa เป็น pointer แต่ b เป็น int ธรรมดา (ไม่ใช่ pointer!) */
    int value = 7;

    pa = &value;
    b = 99;

    printf("sizeof(pa) = %zu ไบต์ (นี่คือ pointer)\n", sizeof pa);
    printf("sizeof(b)  = %zu ไบต์ (นี่คือ int ธรรมดา)\n", sizeof b);
    printf("*pa = %d, b = %d\n", *pa, b);

    return 0;
}
```

โค้ดนี้คอมไพล์ผ่านโดยไม่มี Warning เลย (เพราะไวยากรณ์ถูกต้องตามภาษา) แต่ผลลัพธ์ `sizeof`
จะพิสูจน์ว่า `b` ไม่ใช่ pointer เหมือนที่ตาเห็นตอนแรก — บนเครื่อง 64 บิตทั่วไป `sizeof(pa)`
จะได้ 8 แต่ `sizeof(b)` จะได้ 4 เท่านั้น

> **แนวทางที่ปลอดภัยกว่า**: เขียน pointer แยกบรรทัดเสมอ (`int *pa;` แล้วขึ้นบรรทัดใหม่
> `int b;`) หรือถ้าจะเขียนบรรทัดเดียวจริงๆ ให้ใส่ `*` หน้าทุกตัวแปรที่ต้องการให้เป็น pointer:
> `int *pa, *pb;`

---

## 8.4 NULL Pointer และ Uninitialized Pointer (Step 60)

Pointer ที่ประกาศแล้วแต่ยังไม่ได้กำหนดค่าเริ่มต้น จะมีค่าเป็น "ขยะ" (Garbage Value) คือ
อาจชี้ไปที่ address อะไรก็ได้ในหน่วยความจำ เรียกว่า **Wild Pointer** ซึ่งอันตรายมาก เพราะถ้า
dereference ขึ้นมา อาจไปอ่าน/เขียนทับหน่วยความจำของโปรแกรมส่วนอื่นโดยไม่รู้ตัว

วิธีป้องกันคือ **กำหนดค่าเริ่มต้นให้ pointer เป็น `NULL` เสมอ** ถ้ายังไม่มี address จริงให้ชี้

```c
/* null_pointer.c */
#include <stdio.h>
#include <stddef.h>   /* สำหรับมาตรฐาน NULL */

int main(void) {
    int *ptr = NULL;    /* ยังไม่ชี้ไปที่ใดเลย — ปลอดภัยกว่า wild pointer มาก */

    if (ptr == NULL) {
        printf("ptr ยังไม่ได้ชี้ไปที่ใดเลย (เป็น NULL)\n");
    }

    int value = 10;
    ptr = &value;

    if (ptr != NULL) {
        printf("ตอนนี้ ptr ชี้ไปที่ value = %d แล้ว\n", *ptr);
    }

    return 0;
}
```

`NULL` คือค่าคงที่พิเศษ (โดยทั่วไปเทียบเท่ากับ address `0`) ที่ตามธรรมเนียมของภาษา
**ไม่มี object ใดถูกวางอยู่จริง** ดังนั้นการเปรียบเทียบ `ptr == NULL` ก่อนใช้งานทุกครั้ง
จึงเป็นวิธีป้องกันบั๊กที่นิยมที่สุด

### ทำไมห้าม dereference NULL pointer

```c
/* DANGER: ห้ามคอมไพล์แล้วรันจริงจัง เพราะจะทำให้โปรแกรม crash ทันที
 * แสดงไว้เพื่อการศึกษาเท่านั้น
 */
#include <stdio.h>
#include <stddef.h>

int main(void) {
    int *ptr = NULL;
    printf("%d\n", *ptr);   /* Undefined Behavior: อ่านค่าที่ address 0 */
    return 0;
}
```

โค้ดนี้ **คอมไพล์ผ่านได้โดยไม่มี Warning** เพราะในทางไวยากรณ์ไม่มีอะไรผิด แต่พอรันจริง
ระบบปฏิบัติการจะส่งสัญญาณ `SIGSEGV` (Segmentation Fault) มาหยุดโปรแกรมทันที เพราะ
address `0` ไม่ได้ถูกจองไว้ให้โปรแกรมของเราใช้เลย นี่คือเหตุผลที่ **การตรวจสอบ `!= NULL`
ก่อน dereference ทุกครั้งเป็นวินัยที่สำคัญที่สุดอย่างหนึ่งของโปรแกรมเมอร์ C**

---

## 8.5 Pointer กับ Array: Array Decay เบื้องต้น (Step 61)

นี่คือหนึ่งในความสัมพันธ์ที่สำคัญที่สุดของภาษา C: **ชื่อของ Array เมื่อถูกใช้ในนิพจน์ส่วนใหญ่
จะ "เสื่อมสภาพ" (Decay) กลายเป็น Pointer ที่ชี้ไปยัง Element ตัวแรกโดยอัตโนมัติ**

```c
/* array_pointer_decay.c */
#include <stdio.h>

int main(void) {
    int scores[5] = {10, 20, 30, 40, 50};
    int *ptr = scores;   /* array "decay" กลายเป็น pointer ไปยัง element แรกทันที */

    printf("scores[0] = %d, *ptr       = %d\n", scores[0], *ptr);
    printf("scores[2] = %d, *(ptr + 2) = %d\n", scores[2], *(ptr + 2));

    printf("เข้าถึงทีละตัวด้วย ptr[i]:\n");
    for (size_t i = 0; i < 5; i++) {
        printf("  ptr[%zu] = %d\n", i, ptr[i]);
    }

    printf("sizeof(scores) = %zu ไบต์ (ทั้ง array 5 ตัว x 4 ไบต์)\n", sizeof scores);
    printf("sizeof(ptr)    = %zu ไบต์ (แค่ pointer ตัวเดียว)\n", sizeof ptr);

    return 0;
}
```

```
scores[0] = 10, *ptr       = 10
scores[2] = 30, *(ptr + 2) = 30
เข้าถึงทีละตัวด้วย ptr[i]:
  ptr[0] = 10
  ptr[1] = 20
  ptr[2] = 30
  ptr[3] = 40
  ptr[4] = 50
sizeof(scores) = 20 ไบต์ (ทั้ง array 5 ตัว x 4 ไบต์)
sizeof(ptr)    = 8 ไบต์ (แค่ pointer ตัวเดียว)
```

จากผลลัพธ์นี้เห็นได้ชัดว่า `scores[i]` และ `ptr[i]` (รวมถึง `*(ptr + i)`) ให้ผลลัพธ์เหมือนกัน
ทุกประการ เพราะในความเป็นจริงแล้ว **`arr[i]` คือ syntactic sugar (ไวยากรณ์ที่เขียนให้สวยขึ้น)
ของ `*(arr + i)` เท่านั้นเอง** — Compiler แปลงให้เหมือนกันเป๊ะทั้งสองแบบ

แต่ `sizeof` ฟ้องความแตกต่างที่สำคัญมาก: `sizeof(scores)` ได้ขนาดของ **array ทั้งก้อน**
(20 ไบต์) ในขณะที่ `sizeof(ptr)` ได้ขนาดของ **pointer เพียงตัวเดียว** (8 ไบต์บนเครื่อง 64 บิต)
เพราะ `ptr` ไม่ได้ "เป็น" array มันแค่ "ชี้ไปที่" element แรกของ array เท่านั้น

> **ข้อควรระวัง**: Array Decay เกิดขึ้นเกือบทุกครั้งที่ชื่อ array ถูกใช้ในนิพจน์ ยกเว้น 2 กรณี
> คือตอนใช้กับ `sizeof` และตอนใช้กับ `&` (เช่น `&scores` จะได้ pointer ไปยัง "ทั้ง array"
> ซึ่งเป็นชนิดข้อมูลคนละแบบกับ `scores` เฉยๆ — รายละเอียดเต็มจะอยู่ใน Part 12)

---

## 8.6 ส่ง Pointer เป็น Argument: จำลอง Pass-by-Reference (Step 62)

จาก Part 6 เรารู้ว่า C ส่ง argument แบบ **Pass by Value** เสมอ คือฟังก์ชันจะได้ "สำเนา" ของค่า
ไปใช้งาน ไม่ใช่ตัวแปรต้นฉบับ ทำให้ฟังก์ชันแก้ไขค่าตัวแปรของผู้เรียกไม่ได้โดยตรง — **แต่ถ้าเรา
ส่ง "address" ของตัวแปรไปแทน (ผ่าน pointer) ฟังก์ชันก็จะสามารถ dereference กลับไปแก้ไขค่า
ตัวแปรต้นฉบับได้** นี่คือวิธีที่ C ใช้จำลองพฤติกรรม Pass-by-Reference

```c
/* swap_with_pointer.c */
#include <stdio.h>

void increment(int *value) {
    (*value)++;    /* ต้องมีวงเล็บ! ถ้าเขียน *value++ จะหมายถึงคนละอย่าง (ดู Pitfalls) */
}

void swap(int *a, int *b) {
    int temp = *a;
    *a = *b;
    *b = temp;
}

int main(void) {
    int x = 5;
    increment(&x);
    printf("x หลังจาก increment = %d\n", x);

    int p = 1, q = 2;
    printf("ก่อน swap: p = %d, q = %d\n", p, q);
    swap(&p, &q);
    printf("หลัง swap: p = %d, q = %d\n", p, q);

    return 0;
}
```

```
x หลังจาก increment = 6
ก่อน swap: p = 1, q = 2
หลัง swap: p = 2, q = 1
```

### เปรียบเทียบ: ทำไม Pass by Value ธรรมดาถึงใช้ swap ไม่ได้

```c
/* swap_broken.c
 * เวอร์ชันที่ swap ไม่สำเร็จ เพราะรับค่าแบบ pass by value ธรรมดา
 */
#include <stdio.h>

void swap_wrong(int a, int b) {
    int temp = a;
    a = b;
    b = temp;
    /* a, b ที่ถูกสลับ เป็นแค่ "สำเนา" ในฟังก์ชันนี้เท่านั้น หายไปทันทีที่ฟังก์ชันจบ */
}

int main(void) {
    int p = 1, q = 2;
    swap_wrong(p, q);
    printf("หลังเรียก swap_wrong: p = %d, q = %d (ไม่เปลี่ยนเลย!)\n", p, q);
    return 0;
}
```

```
หลังเรียก swap_wrong: p = 1, q = 2 (ไม่เปลี่ยนเลย!)
```

โค้ดทั้งสองเวอร์ชันคอมไพล์ผ่านโดยไม่มี Warning เพราะไม่มีอะไรผิดไวยากรณ์ — แต่ `swap_wrong`
ไม่ทำงานตามที่ตั้งใจ นี่คือเหตุผลว่าทำไม "การส่ง address" จึงเป็นเทคนิคที่ขาดไม่ได้เวลาต้องการ
ให้ฟังก์ชันแก้ไขค่าของตัวแปรที่อยู่นอกฟังก์ชันนั้น

---

## 8.7 ภาพรวม Memory Layout ด้วย ASCII Diagram (Step 63)

มาดูภาพรวมทั้งหมดของสิ่งที่เกิดขึ้นในหน่วยความจำ (Stack) ระหว่างรันโค้ดตัวอย่างนี้:

```c
int   x   = 10;
int  *p   = &x;
int **pp  = &p;
```

```
                Address (ตัวอย่าง)      ชื่อตัวแปร     ค่าที่เก็บอยู่
              +------------------+
 0x7ffd...30  |        10        |  <-- x            (int ธรรมดา)
              +------------------+
 0x7ffd...28  |    0x7ffd...30   |  <-- p            (pointer ชี้ไปที่ x)
              +------------------+                        │
 0x7ffd...20  |    0x7ffd...28   |  <-- pp           (pointer ชี้ไปที่ p)
              +------------------+                        │
                                                            │
   pp ────────────────────────────────────────────────────┘
    │  (pp ชี้ไปที่ address ของ p)
    ▼
    p ─────────────► (p ชี้ไปที่ address ของ x)
                       │
                       ▼
                       x = 10
```

อ่าน diagram นี้จากล่างขึ้นบน: `x` คือห้องเก็บเลข `10` ธรรมดา, `p` คือห้องที่เก็บ "เลขที่ห้อง"
ของ `x` เอาไว้ (คือ address `0x7ffd...30`), และ `pp` (Pointer to Pointer ที่จะเรียนเต็มใน
Part 9) คือห้องที่เก็บ "เลขที่ห้อง" ของ `p` อีกที เหมือนจดหมายที่ส่งต่อกันเป็นทอดๆ:
`pp` บอกว่าไปหา `p`, `p` บอกว่าไปหา `x`, และ `x` คือปลายทางที่มีค่าจริงอยู่

การฝึกวาด diagram แบบนี้ด้วยมือเวลาเขียนโปรแกรมที่มี pointer ซับซ้อน (โดยเฉพาะ pointer to
pointer, array of pointer, หรือ struct ที่มี pointer เป็น member) เป็นทักษะที่ช่วยลด
บั๊กเรื่อง pointer ได้มากที่สุดวิธีหนึ่ง — เมื่อ debug แล้วงงว่า pointer ไหนชี้ไปที่ไหน ให้หยิบ
กระดาษมาวาด diagram แบบนี้ก่อนเสมอ

---

## 8.8 `sizeof` ของ Pointer และข้อผิดพลาดเรื่องขนาด (Step 64)

ประเด็นสำคัญที่มือใหม่มักเข้าใจผิดคือ **ขนาดของ pointer ไม่ได้ขึ้นกับชนิดข้อมูลที่มันชี้ไป**
แต่ขึ้นกับสถาปัตยกรรมของเครื่อง (32 บิต หรือ 64 บิต) เท่านั้น — Pointer ทุกชนิดบนเครื่องเดียวกัน
จะมีขนาดเท่ากันหมด เพราะโดยแก่นแท้แล้ว pointer ก็คือ "ตัวเลข address" ตัวหนึ่งเท่านั้น

```c
/* pointer_sizeof.c */
#include <stdio.h>

struct point { int x; int y; };

int main(void) {
    int *pi = NULL;
    char *pc = NULL;
    double *pd = NULL;
    struct point *pp = NULL;

    printf("sizeof(int *)          = %zu ไบต์\n", sizeof pi);
    printf("sizeof(char *)         = %zu ไบต์\n", sizeof pc);
    printf("sizeof(double *)       = %zu ไบต์\n", sizeof pd);
    printf("sizeof(struct point *) = %zu ไบต์\n", sizeof pp);

    printf("--- เทียบกับขนาดของชนิดข้อมูลที่ถูกชี้ไป ---\n");
    printf("sizeof(int)          = %zu ไบต์\n", sizeof(int));
    printf("sizeof(char)         = %zu ไบต์\n", sizeof(char));
    printf("sizeof(double)       = %zu ไบต์\n", sizeof(double));
    printf("sizeof(struct point) = %zu ไบต์\n", sizeof(struct point));

    return 0;
}
```

ผลลัพธ์บนเครื่อง 64 บิตทั่วไป:

```
sizeof(int *)          = 8 ไบต์
sizeof(char *)         = 8 ไบต์
sizeof(double *)       = 8 ไบต์
sizeof(struct point *) = 8 ไบต์
--- เทียบกับขนาดของชนิดข้อมูลที่ถูกชี้ไป ---
sizeof(int)          = 4 ไบต์
sizeof(char)         = 1 ไบต์
sizeof(double)       = 8 ไบต์
sizeof(struct point) = 8 ไบต์
```

จะเห็นว่า pointer ทุกตัว (`int *`, `char *`, `double *`, `struct point *`) มีขนาด **8 ไบต์
เท่ากันหมด** ไม่ว่าจะชี้ไปยังข้อมูลชนิดใด เพราะ pointer เก็บแค่ "เลขที่ห้อง" เดียวกันในความหมาย
ของระบบปฏิบัติการ ส่วนชนิดข้อมูลที่อยู่หน้า `*` (เช่น `int` ใน `int *`) มีไว้บอก Compiler ว่า
**ถ้า dereference แล้วจะอ่าน/เขียนกี่ไบต์ และตีความ bit pattern นั้นเป็นอะไร** ไม่ได้มีผลต่อ
ขนาดของตัว pointer เองเลย

> หมายเหตุ: ขนาด 8 ไบต์นี้เป็นค่าทั่วไปบนระบบ 64 บิต (x86-64) บนเครื่อง 32 บิตรุ่นเก่า
> pointer ทุกชนิดจะมีขนาด 4 ไบต์เท่ากันหมดแทน — กฎ "pointer ทุกชนิดขนาดเท่ากันในเครื่อง
> เดียวกัน" ยังคงเป็นจริงเสมอ

---

## ข้อผิดพลาดที่พบบ่อย (Common Pitfalls)

1. **Dereference NULL หรือ Wild Pointer** — การเข้าถึง `*ptr` โดยไม่ตรวจสอบก่อนว่า `ptr`
   มีค่าเป็น address ที่ถูกต้องแล้วหรือยัง เป็นสาเหตุอันดับหนึ่งของ Segmentation Fault
   ในโปรแกรม C แก้โดยกำหนดค่าเริ่มต้นเป็น `NULL` เสมอ และตรวจสอบ `if (ptr != NULL)`
   ก่อน dereference ทุกครั้งที่ไม่แน่ใจ

2. **Uninitialized Pointer (Wild Pointer)** — การประกาศ `int *p;` แล้วใช้งาน `*p` ทันที
   โดยไม่ได้กำหนดค่าให้มันชี้ไปที่ใดก่อน เป็น Undefined Behavior เพราะ `p` มีค่าขยะที่ไม่รู้
   ว่าชี้ไปที่ใด อาจไปเขียนทับข้อมูลสำคัญของโปรแกรมโดยไม่มี error ใดๆ เตือนล่วงหน้าเลย

3. **สับสนการประกาศ pointer หลายตัวบรรทัดเดียว** — `int *pa, pb;` ทำให้ `pb` เป็น `int`
   ธรรมดา ไม่ใช่ pointer ทั้งที่ดูตาเปล่าเหมือนควรจะเป็น ควรเขียน `int *pa, *pb;` หรือแยก
   คนละบรรทัดเสมอ

4. **สับสน `sizeof(array)` กับ `sizeof(pointer)`** — เมื่อ array ถูกส่งเข้าไปในฟังก์ชันเป็น
   parameter มันจะ decay กลายเป็น pointer เสมอ ทำให้ `sizeof` ที่เรียกข้างในฟังก์ชันได้ขนาด
   ของ pointer (เช่น 8) แทนที่จะเป็นขนาดของ array ทั้งก้อน (เช่น 20) เป็นบั๊กคลาสสิกที่พบ
   บ่อยมากในฟังก์ชันที่พยายามหาความยาว array จาก parameter โดยไม่ส่งขนาดมาด้วยตรงๆ

5. **Dangling Pointer จากการ return address ของตัวแปร local** — ตัวแปร local (ที่ประกาศ
   ในฟังก์ชันโดยไม่ใช้ `static`) จะถูกทำลายทันทีที่ฟังก์ชันจบการทำงาน (อยู่บน Stack) ถ้า
   return address ของมันออกไป pointer นั้นจะกลายเป็น "Dangling Pointer" ที่ชี้ไปยังหน่วย
   ความจำที่ไม่ได้เป็นของตัวแปรนั้นแล้ว:

   ```c
   /* dangling_pointer.c — ตัวอย่าง UB ที่ compiler เตือนให้ล่วงหน้า */
   #include <stdio.h>

   int *get_local_address(void) {
       int local_value = 99;
       return &local_value;   /* อันตราย: local_value จะถูกทำลายทันทีที่ฟังก์ชันจบ */
   }

   int main(void) {
       int *p = get_local_address();
       printf("%d\n", *p);    /* Undefined Behavior: อ่านค่าจาก stack ที่ถูกคืนแล้ว */
       return 0;
   }
   ```

   คอมไพล์ด้วย `gcc -Wall -Wextra -Wpedantic -std=c17` จะได้ Warning ทันที:

   ```
   warning: function returns address of local variable [-Wreturn-local-addr]
   ```

   วิธีแก้คือห้าม return address ของตัวแปร local เด็ดขาด ถ้าต้องการให้ข้อมูลอยู่รอดหลัง
   ฟังก์ชันจบ ต้องใช้ Dynamic Memory Allocation (`malloc`) ซึ่งจะเรียนใน Part 11 หรือส่ง
   pointer ไปยังตัวแปรที่ผู้เรียกเป็นเจ้าของอยู่แล้วแทน

6. **ลืมใส่วงเล็บตอนใช้ `*` ร่วมกับตัวดำเนินการอื่น** — `*ptr++` กับ `(*ptr)++` มีความหมาย
   ต่างกันโดยสิ้นเชิง เพราะ `++` มีลำดับความสำคัญ (precedence) สูงกว่า `*` แบบ dereference
   `*ptr++` จะแปลว่า "dereference ค่าปัจจุบันก่อน แล้วค่อยขยับ pointer ไปข้างหน้า" (เทียบเท่า
   `*(ptr++)`) ในขณะที่ `(*ptr)++` แปลว่า "เพิ่มค่าที่ pointer ชี้อยู่ขึ้นหนึ่ง" ซึ่งเป็นคนละเรื่อง
   กันเลย ต้องใส่วงเล็บให้ชัดเจนเสมอเมื่อผสมตัวดำเนินการเหล่านี้เข้าด้วยกัน (รายละเอียดเต็มเรื่อง
   ลำดับความสำคัญของ pointer arithmetic จะอยู่ใน Part 9)

---

## แบบฝึกหัดท้ายบท

1. เขียนฟังก์ชัน `void double_value(int *p)` ที่รับ pointer ไปยัง `int` แล้วคูณค่าที่ pointer
   นั้นชี้อยู่ด้วย 2 (แก้ไขค่าต้นฉบับโดยตรงผ่าน pointer) แล้วเรียกใช้ใน `main`

2. เขียนโปรแกรมที่ประกาศ `int arr[5]` แล้วพิมพ์ทุก element ออกมาโดย **ห้ามใช้เครื่องหมาย `[]`
   เลยแม้แต่ครั้งเดียว** (ใช้ pointer arithmetic กับ `*(ptr + i)` เท่านั้น)

3. เขียนโปรแกรมพิสูจน์ปัญหาข้อ 4 ใน Common Pitfalls: ประกาศ array ใน `main` แล้วส่งเข้า
   ฟังก์ชัน แล้วพิมพ์ `sizeof` ของ array ทั้งใน `main` และในฟังก์ชันนั้น เปรียบเทียบผลลัพธ์
   ว่าต่างกันจริงหรือไม่ พร้อมอธิบายด้วยคำพูดตัวเอง

4. เขียนฟังก์ชัน `int *find_max(int *arr, int size)` ที่รับ array (ผ่าน pointer) และขนาดของมัน
   แล้ว **คืนค่าเป็น pointer** ที่ชี้ไปยัง element ที่มีค่ามากที่สุดใน array นั้น (ไม่ใช่คืนค่าตัวเลข
   ธรรมดา) จากนั้นในฟังก์ชัน `main` ให้ dereference pointer ที่ได้เพื่อพิมพ์ค่ามากที่สุดออกมา

5. หา bug จากฟังก์ชัน `get_local_address` ในหัวข้อ Common Pitfalls ข้อ 5 แล้วแก้ไขให้ถูกต้อง
   โดยเปลี่ยนวิธีการออกแบบฟังก์ชันใหม่ (ไม่ใช้ `malloc` เพราะยังไม่ได้เรียน — ให้เปลี่ยนวิธี
   ส่งค่าแทน เช่น ส่ง pointer จาก `main` เข้าไปให้ฟังก์ชันเขียนค่าใส่แทน)

6. เขียนโปรแกรมที่มีตัวแปร `int x = 5;`, `int *p = &x;`, `int **pp = &p;` แล้ววาด ASCII
   diagram แบบในหัวข้อ 8.7 ด้วยมือของตัวเอง (บนกระดาษหรือ text editor ก็ได้) จากนั้นพิมพ์
   `*p`, `**pp`, `*pp` ออกมาเทียบกับ diagram ที่วาดไว้ว่าตรงกันหรือไม่

### แนวทางเฉลยข้อ 1

```c
/* double_value.c */
#include <stdio.h>

void double_value(int *p) {
    *p = *p * 2;
}

int main(void) {
    int number = 21;
    printf("ก่อนเรียก double_value: number = %d\n", number);

    double_value(&number);
    printf("หลังเรียก double_value: number = %d\n", number);

    return 0;
}
```

```
ก่อนเรียก double_value: number = 21
หลังเรียก double_value: number = 42
```

จุดสำคัญคือต้องส่ง `&number` (address ของ number) เข้าไป ไม่ใช่ `number` เฉยๆ เพราะฟังก์ชัน
รับพารามิเตอร์เป็น `int *p` (pointer) ถ้าส่ง `number` ตรงๆ จะเกิด Compile Error ทันที เพราะ
ชนิดข้อมูลไม่ตรงกัน (ส่ง `int` ให้พารามิเตอร์ที่ต้องการ `int *`)

### แนวทางเฉลยข้อ 4

```c
/* find_max.c */
#include <stdio.h>

int *find_max(int *arr, int size) {
    int *max_ptr = &arr[0];   /* เริ่มต้นสมมติว่า element แรกคือค่ามากสุดก่อน */

    for (int i = 1; i < size; i++) {
        if (arr[i] > *max_ptr) {
            max_ptr = &arr[i];
        }
    }

    return max_ptr;
}

int main(void) {
    int scores[6] = {55, 82, 91, 47, 68, 90};
    int size = 6;

    int *result = find_max(scores, size);

    printf("ค่ามากที่สุดคือ %d (อยู่ที่ index %ld ของ array)\n",
           *result, (long)(result - scores));

    return 0;
}
```

```
ค่ามากที่สุดคือ 91 (อยู่ที่ index 2 ของ array)
```

จุดที่น่าสนใจในเฉลยนี้คือบรรทัดสุดท้าย: `result - scores` คือการนำ pointer สองตัวมาลบกัน
เพื่อหาว่า element ที่ `result` ชี้อยู่ ห่างจาก element แรกของ array กี่ตำแหน่ง (คือหา index
นั่นเอง) นี่คือตัวอย่างเล็กๆ ของ **Pointer Subtraction** ซึ่งจะอธิบายอย่างละเอียดใน Part 9

### แนวทางเฉลยข้อ 2

```c
/* print_array_no_brackets.c */
#include <stdio.h>

int main(void) {
    int arr[5] = {11, 22, 33, 44, 55};
    int *ptr = arr;   /* array decay เป็น pointer ไปยัง element แรก */

    printf("พิมพ์ array โดยห้ามใช้ [] เลย:\n");
    for (int i = 0; i < 5; i++) {
        printf("  element ที่ %d = %d\n", i, *(ptr + i));
    }

    return 0;
}
```

```
พิมพ์ array โดยห้ามใช้ [] เลย:
  element ที่ 0 = 11
  element ที่ 1 = 22
  element ที่ 2 = 33
  element ที่ 3 = 44
  element ที่ 4 = 55
```

เฉลยนี้ใช้ `*(ptr + i)` แทน `arr[i]` โดยตรง ซึ่งให้ผลลัพธ์เหมือนกันทุกประการ เพราะอย่างที่
อธิบายไว้ในหัวข้อ 8.5 ทั้งสองรูปแบบเป็นสิ่งเดียวกันในสายตาของ Compiler อีกวิธีหนึ่งที่ทำได้
เช่นกันคือขยับตัว `ptr` เองไปเรื่อยๆ ด้วย `ptr++` ในลูป แทนที่จะคำนวณ `ptr + i` ใหม่ทุกรอบ
(ทั้งสองวิธีให้ผลลัพธ์เหมือนกัน แต่ต้องระวังว่าถ้าขยับ `ptr` เองแล้ว ตัวแปร `ptr` จะไม่ได้ชี้ไปที่
element แรกของ array อีกต่อไปหลังจากลูปจบ)

---

## สรุปท้ายบท

ใน Part นี้เราได้:

- เข้าใจว่า Memory Address คือ "เลขที่ห้อง" ของหน่วยความจำ และตัวแปรคือ "ชื่อเล่น" ของห้องนั้น
- ใช้ตัวดำเนินการ `&` (address-of) และ `*` (dereference) ได้อย่างถูกต้อง
- ประกาศ pointer ได้อย่างปลอดภัย และรู้จักหลีกเลี่ยงกับดักไวยากรณ์ `int *a, b;`
- เข้าใจ `NULL` pointer และอันตรายของการ dereference wild pointer หรือ NULL pointer
- เข้าใจปรากฏการณ์ Array Decay ที่ทำให้ array กลายเป็น pointer โดยอัตโนมัติ และผลกระทบ
  ต่อ `sizeof`
- เขียนฟังก์ชันที่รับ pointer เพื่อจำลอง Pass-by-Reference ได้ (เช่น `swap`, `increment`)
- อ่านและวาด ASCII diagram อธิบาย memory layout ของ pointer ได้ด้วยตัวเอง

Pointer คือรากฐานที่สำคัญที่สุดอย่างหนึ่งของภาษา C — แนวคิดที่เรียนใน Part นี้จะถูกใช้ซ้ำแล้ว
ซ้ำเล่าตลอดทั้งหลักสูตร ตั้งแต่ Dynamic Memory, Data Structure, ไปจนถึง Modern C++ (Smart
Pointer ก็สร้างอยู่บนความเข้าใจพื้นฐานนี้เช่นกัน) ใน **Part 9** เราจะเจาะลึกต่อไปในเรื่อง
**Pointer Arithmetic, Pointer to Pointer, Function Pointer และ const กับ Pointer**
ซึ่งเป็นทักษะระดับที่โปรแกรมเมอร์ C มืออาชีพต้องใช้งานเป็นประจำ

**ต่อไป:** [Part 9 — Pointer ขั้นสูง (ตอนที่ 2)](./part-009-pointers-advanced.md)
