# Part 9: Pointer ขั้นสูง (ตอนที่ 2) (Step 65–72)

> Module A — รากฐานภาษา C (Foundations of C) | Part 9 จาก 125
> Step ที่ครอบคลุมใน Part นี้: Step 65–72
> Part ก่อนหน้า: [Part 8 — Pointer พื้นฐาน (ตอนที่ 1)](./part-008-pointers-basics.md) | Part ถัดไป: [Part 10 — Struct, Union, Enum และ typedef](./part-010-struct-union-enum.md)

## เป้าหมายการเรียนรู้ (Learning Objectives)

หลังจบ Part นี้ ผู้เรียนจะสามารถ:

1. อธิบายและใช้งาน Pointer Arithmetic ได้อย่างถูกต้อง พร้อมเข้าใจว่าการบวก/ลบ pointer
   คำนวณตาม `sizeof` ของชนิดข้อมูลที่ชี้อยู่อย่างไร ไม่ใช่บวกลบทีละ 1 ไบต์เสมอไป
2. เปรียบเทียบและลบ pointer สองตัวเพื่อหาระยะห่างระหว่าง element ได้อย่างถูกต้องและปลอดภัย
3. เข้าใจและใช้งาน Pointer to Pointer (`**`) ได้ พร้อมอธิบายกรณีการใช้งานจริง
4. ประกาศและเรียกใช้ Function Pointer ได้ รวมถึงใช้เป็น Callback และสร้าง Dispatch Table
5. อธิบายความแตกต่างของ `const` กับ pointer ทั้ง 4 รูปแบบได้อย่างแม่นยำ และเลือกใช้ให้เหมาะสม
6. ใช้งาน `void *` (Generic Pointer) ได้อย่างถูกต้องและปลอดภัย
7. ผสมผสาน Function Pointer และ `void *` เพื่อเขียนโค้ดแบบ Generic เช่นการเรียกใช้ `qsort`

---

## 9.1 Pointer Arithmetic: บวก/ลบ Pointer คำนวณอย่างไร (Step 65)

ใน Part 8 เราเห็นแล้วว่า `ptr[i]` เทียบเท่ากับ `*(ptr + i)` แต่คำถามที่สำคัญมากคือ **"+ i" ตรงนี้
หมายถึงบวกกี่ไบต์กันแน่?** คำตอบคือ: **`ptr + i` ไม่ได้บวก `i` ไบต์ตรงๆ แต่บวก `i * sizeof(ชนิดข้อมูลที่ชี้อยู่)` ไบต์**
Compiler จะคำนวณให้อัตโนมัติตามชนิดข้อมูลที่ pointer นั้นถูกประกาศไว้

```c
/* pointer_arith_size.c
 * สาธิตว่า pointer arithmetic คำนวณตาม sizeof ของชนิดข้อมูล ไม่ใช่ทีละ 1 ไบต์
 */
#include <stdio.h>

int main(void) {
    int int_arr[4]     = {0};
    double double_arr[4] = {0};
    char  char_arr[4]  = {0};

    int    *pi = int_arr;
    double *pd = double_arr;
    char   *pc = char_arr;

    printf("int array:    pi     = %p\n", (void *)pi);
    printf("              pi + 1 = %p  (ขยับไป %zu ไบต์ = sizeof(int))\n",
           (void *)(pi + 1), sizeof(int));

    printf("double array: pd     = %p\n", (void *)pd);
    printf("              pd + 1 = %p  (ขยับไป %zu ไบต์ = sizeof(double))\n",
           (void *)(pd + 1), sizeof(double));

    printf("char array:   pc     = %p\n", (void *)pc);
    printf("              pc + 1 = %p  (ขยับไป %zu ไบต์ = sizeof(char))\n",
           (void *)(pc + 1), sizeof(char));

    return 0;
}
```

ผลลัพธ์ตัวอย่าง (address จริงเปลี่ยนไปทุกครั้งที่รัน แต่ **ระยะห่าง** จะคงที่เสมอ):

```
int array:    pi     = 0x7ffd12340000
              pi + 1 = 0x7ffd12340004  (ขยับไป 4 ไบต์ = sizeof(int))
double array: pd     = 0x7ffd12340010
              pd + 1 = 0x7ffd12340018  (ขยับไป 8 ไบต์ = sizeof(double))
char array:   pc     = 0x7ffd12340020
              pc + 1 = 0x7ffd12340021  (ขยับไป 1 ไบต์ = sizeof(char))
```

นี่คือเหตุผลสำคัญที่ทำให้ `for (int i = 0; i < n; i++) { ptr[i] = ...; }` ทำงานถูกต้องเสมอ
ไม่ว่า `ptr` จะเป็น `int *`, `double *`, หรือ struct ที่ใหญ่แค่ไหนก็ตาม — Compiler จะรู้เองว่า
"ก้าวไปหนึ่งช่อง" ต้องขยับกี่ไบต์ตามชนิดข้อมูลที่ประกาศไว้

> **ข้อยกเว้นที่ควรรู้**: `void *` ไม่มีชนิดข้อมูลที่ชัดเจนให้คำนวณขนาด ดังนั้นมาตรฐาน C
> **ไม่อนุญาต** ให้ทำ pointer arithmetic กับ `void *` โดยตรง (GCC อนุญาตเป็น extension
> ที่ถือว่า `void *` มีขนาด 1 ไบต์ แต่จะถูกเตือนภายใต้ `-Wpedantic`) เรื่องนี้จะอธิบายเพิ่มใน 9.7

### การลบ pointer ด้วยค่าคงที่ และการเดินถอยหลัง

```c
/* pointer_arith_backward.c */
#include <stdio.h>

int main(void) {
    int numbers[5] = {100, 200, 300, 400, 500};
    int *last = &numbers[4];   /* ชี้ไปที่ element สุดท้าย */

    printf("เดินจากท้าย array ถอยไปหน้า:\n");
    for (int i = 0; i < 5; i++) {
        printf("  *(last - %d) = %d\n", i, *(last - i));
    }

    return 0;
}
```

```
เดินจากท้าย array ถอยไปหน้า:
  *(last - 0) = 500
  *(last - 1) = 400
  *(last - 2) = 300
  *(last - 3) = 200
  *(last - 4) = 100
```

---

## 9.2 การเปรียบเทียบ Pointer และ Pointer Subtraction (Step 66)

Pointer สามารถเปรียบเทียบกันได้ด้วย `<`, `>`, `<=`, `>=`, `==`, `!=` **แต่การเปรียบเทียบที่มี
ความหมายตามมาตรฐานภาษา C ต้องเป็น pointer ที่ชี้อยู่ใน array เดียวกันเท่านั้น** (หรือ element
ถัดจากตัวสุดท้ายพอดี 1 ตำแหน่ง) การเปรียบเทียบ pointer จาก array คนละก้อนเป็น Undefined
Behavior ในทางเทคนิค แม้ในทางปฏิบัติจะไม่ crash ก็ตาม

การลบ pointer สองตัวที่ชี้อยู่ใน array เดียวกัน จะได้ผลลัพธ์เป็น **จำนวน element ที่ห่างกัน**
ไม่ใช่จำนวนไบต์ ชนิดข้อมูลของผลลัพธ์นี้คือ `ptrdiff_t` (นิยามอยู่ใน `<stddef.h>`)

```c
/* pointer_diff.c */
#include <stdio.h>
#include <stddef.h>

int main(void) {
    int data[6] = {10, 20, 30, 40, 50, 60};

    int *begin = &data[0];
    int *end   = &data[5];

    ptrdiff_t distance = end - begin;

    printf("begin ชี้ที่ index 0, end ชี้ที่ index 5\n");
    printf("end - begin = %td element (ไม่ใช่ไบต์!)\n", distance);

    if (begin < end) {
        printf("begin อยู่ก่อน end ใน array เดียวกัน\n");
    }

    return 0;
}
```

```
begin ชี้ที่ index 0, end ชี้ที่ index 5
end - begin = 5 element (ไม่ใช่ไบต์!)
begin อยู่ก่อน end ใน array เดียวกัน
```

สังเกตว่า Format Specifier ที่ถูกต้องสำหรับ `ptrdiff_t` คือ `%td` (t = ptrdiff_t, d = signed
decimal) — การใช้ `%d` ธรรมดากับ `ptrdiff_t` บนบางแพลตฟอร์มอาจได้ Warning เรื่อง type
mismatch ภายใต้ `-Wformat`

---

## 9.3 Pointer to Pointer (`**`) (Step 67)

Pointer to Pointer คือ pointer ที่ **ชี้ไปยัง pointer อีกตัวหนึ่ง** ไม่ใช่ชี้ไปยังตัวแปรข้อมูลทั่วไป
โดยตรง เขียนด้วยเครื่องหมาย `**` สองดอกติดกัน แนวคิดนี้ต่อยอดโดยตรงจาก diagram ในหัวข้อ 8.7

```c
/* pointer_to_pointer.c */
#include <stdio.h>

int main(void) {
    int value = 42;
    int *p = &value;     /* p ชี้ไปที่ value */
    int **pp = &p;        /* pp ชี้ไปที่ p (ซึ่งเป็น pointer) */

    printf("value  = %d\n", value);
    printf("*p     = %d   (dereference ครั้งเดียว ไปถึง value)\n", *p);
    printf("**pp   = %d   (dereference สองครั้ง: pp -> p -> value)\n", **pp);

    printf("\naddress ของแต่ละชั้น:\n");
    printf("&value = %p\n", (void *)&value);
    printf("p      = %p  (ค่าของ p คือ address ของ value)\n", (void *)p);
    printf("&p     = %p\n", (void *)&p);
    printf("pp     = %p  (ค่าของ pp คือ address ของ p)\n", (void *)pp);

    /* แก้ไขค่า value ผ่าน pp โดยตรง */
    **pp = 100;
    printf("\nหลังจาก **pp = 100, value = %d\n", value);

    return 0;
}
```

```
value  = 42
*p     = 42   (dereference ครั้งเดียว ไปถึง value)
**pp   = 42   (dereference สองครั้ง: pp -> p -> value)

address ของแต่ละชั้น:
&value = 0x7ffd1a2b3c40
p      = 0x7ffd1a2b3c40  (ค่าของ p คือ address ของ value)
&p     = 0x7ffd1a2b3c48
pp     = 0x7ffd1a2b3c48  (ค่าของ pp คือ address ของ p)

หลังจาก **pp = 100, value = 100
```

### กรณีใช้งานจริง: ฟังก์ชันที่ต้องแก้ไข pointer ของผู้เรียกเอง

เหตุผลหลักที่ต้องใช้ `**` คือเมื่อเราต้องการให้ **ฟังก์ชันแก้ไขค่าของ pointer เอง** (ไม่ใช่แก้ไข
สิ่งที่ pointer ชี้ไป) เช่นฟังก์ชันที่ต้องเปลี่ยนให้ pointer ของผู้เรียกไปชี้ที่ตำแหน่งใหม่:

```c
/* pp_reassign.c
 * สาธิตว่าทำไมฟังก์ชันที่ต้องการเปลี่ยน "ทิศทาง" ของ pointer ต้องรับ int **
 */
#include <stdio.h>

void point_to_second(int *arr, int **ptr_ref) {
    *ptr_ref = &arr[1];   /* เปลี่ยนให้ pointer ของผู้เรียกไปชี้ที่ element ตัวที่สอง */
}

int main(void) {
    int arr[3] = {10, 20, 30};
    int *cursor = &arr[0];

    printf("ก่อนเรียกฟังก์ชัน: *cursor = %d\n", *cursor);

    point_to_second(arr, &cursor);   /* ส่ง address ของ cursor เข้าไป */

    printf("หลังเรียกฟังก์ชัน:  *cursor = %d\n", *cursor);

    return 0;
}
```

```
ก่อนเรียกฟังก์ชัน: *cursor = 10
หลังเรียกฟังก์ชัน:  *cursor = 20
```

ถ้าฟังก์ชัน `point_to_second` รับพารามิเตอร์เป็น `int *ptr_ref` ธรรมดา (ไม่ใช่ `int **`) การ
เปลี่ยนค่าใน `ptr_ref` จะเปลี่ยนแค่สำเนา pointer ในฟังก์ชันเท่านั้น `cursor` ใน `main` จะไม่
เปลี่ยนเลย — หลักการเดียวกับที่ต้องส่ง `&x` เพื่อแก้ไขค่า `int` แต่ยกระดับขึ้นไปอีกชั้นหนึ่ง
(pattern นี้จะเจออีกครั้งใน Part 11 ตอนที่ฟังก์ชันต้อง `malloc` แล้วส่ง pointer กลับผ่าน
พารามิเตอร์ที่เป็น `**`)

---

## 9.4 Function Pointer: การประกาศและเรียกใช้ (Step 68)

ใน C แม้แต่ **ฟังก์ชัน** ก็มี address ในหน่วยความจำ (เก็บอยู่ในส่วน Code/Text Segment) และเรา
สามารถเก็บ address นั้นไว้ในตัวแปรได้เช่นกัน เรียกว่า **Function Pointer** ทำให้เราส่งฟังก์ชัน
ไปมาระหว่างฟังก์ชันได้เหมือนส่งตัวแปรทั่วไป ซึ่งเป็นรากฐานของการเขียนโปรแกรมแบบ Callback

รูปแบบการประกาศ (วงเล็บรอบชื่อตัวแปรจำเป็นมาก อธิบายในหัวข้อ Pitfalls):

```c
ชนิดข้อมูลที่คืนค่า (*ชื่อตัวแปร)(ชนิดพารามิเตอร์...);
```

```c
/* function_pointer_basic.c */
#include <stdio.h>

int add(int a, int b) {
    return a + b;
}

int multiply(int a, int b) {
    return a * b;
}

int main(void) {
    int (*operation)(int, int);   /* ประกาศ function pointer ที่รับ (int,int) คืน int */

    operation = add;              /* ชื่อฟังก์ชันเปลี่ยนเป็น pointer ไปยังตัวมันเองอัตโนมัติ */
    printf("add(3, 4)      = %d\n", operation(3, 4));

    operation = multiply;
    printf("multiply(3, 4) = %d\n", operation(3, 4));

    return 0;
}
```

```
add(3, 4)      = 7
multiply(3, 4) = 12
```

สังเกตว่าเราไม่จำเป็นต้องเขียน `&add` เพราะชื่อฟังก์ชันจะ decay เป็น function pointer โดย
อัตโนมัติเช่นเดียวกับที่ array decay เป็น pointer (แต่ก็สามารถเขียน `&add` ได้เช่นกัน — ทั้งสองแบบ
ให้ผลเหมือนกัน) และการเรียก `operation(3, 4)` ก็ทำได้ตรงๆ โดยไม่ต้อง `(*operation)(3, 4)`
แม้จะเขียนแบบมีวงเล็บ dereference ก็ได้ผลเหมือนกันทุกประการ

---

## 9.5 Function Pointer เป็น Callback และ Dispatch Table (Step 69)

การใช้งานที่ทรงพลังที่สุดของ Function Pointer คือการส่งฟังก์ชันเข้าไปเป็นพารามิเตอร์ของอีก
ฟังก์ชันหนึ่ง (เรียกว่า **Callback**) เพื่อให้ฟังก์ชันแม่ "เลือก" พฤติกรรมได้โดยไม่ต้องแก้โค้ด
ฟังก์ชันแม่เลย

```c
/* function_pointer_callback.c */
#include <stdio.h>

int square(int x) { return x * x; }
int negate(int x) { return -x; }
int increment_by_one(int x) { return x + 1; }

/* ฟังก์ชันที่รับ callback: apply_to_all จะเรียก transform() กับทุก element */
void apply_to_all(int *arr, int size, int (*transform)(int)) {
    for (int i = 0; i < size; i++) {
        arr[i] = transform(arr[i]);
    }
}

void print_array(const int *arr, int size) {
    for (int i = 0; i < size; i++) {
        printf("%d ", arr[i]);
    }
    printf("\n");
}

int main(void) {
    int data[5] = {1, 2, 3, 4, 5};

    print_array(data, 5);

    apply_to_all(data, 5, square);
    printf("หลัง square:          ");
    print_array(data, 5);

    apply_to_all(data, 5, negate);
    printf("หลัง negate:          ");
    print_array(data, 5);

    apply_to_all(data, 5, increment_by_one);
    printf("หลัง increment_by_one: ");
    print_array(data, 5);

    return 0;
}
```

```
1 2 3 4 5
หลัง square:          1 4 9 16 25
หลัง negate:          -1 -4 -9 -16 -25
หลัง increment_by_one: 0 -3 -8 -15 -24
```

`apply_to_all` ไม่จำเป็นต้องรู้เลยว่ากำลังยกกำลังสอง หรือติดลบ หรือบวกเพิ่ม — มันแค่เรียก
`transform(arr[i])` โดยไม่สนใจว่า `transform` คือฟังก์ชันไหน นี่คือหัวใจของการเขียนโค้ดแบบ
**Generic/Reusable** ที่ใช้กันแพร่หลายในไลบรารีมาตรฐาน (เช่น `qsort` ที่จะเห็นในหัวข้อ 9.8)

### Dispatch Table: Array ของ Function Pointer

เมื่อมี "เมนูตัวเลือก" หลายแบบ แทนที่จะเขียน `if-else` หรือ `switch` ยาวๆ เราสามารถเก็บ
function pointer ไว้ใน array แล้วเลือกเรียกด้วย index ได้เลย เรียกเทคนิคนี้ว่า **Dispatch
Table** ซึ่งใช้กันมากในการเขียน Interpreter, State Machine, และ Menu-driven Program

```c
/* dispatch_table.c
 * สาธิต Dispatch Table: เลือกฟังก์ชันคำนวณจาก index แทนการเขียน switch-case ยาวๆ
 */
#include <stdio.h>

double op_add(double a, double b)      { return a + b; }
double op_subtract(double a, double b) { return a - b; }
double op_multiply(double a, double b) { return a * b; }
double op_divide(double a, double b)   { return (b != 0.0) ? a / b : 0.0; }

int main(void) {
    /* Dispatch table: index 0=+, 1=-, 2=*, 3=/ */
    double (*operations[4])(double, double) = {
        op_add, op_subtract, op_multiply, op_divide
    };
    const char *symbols[4] = { "+", "-", "*", "/" };

    double x = 10.0, y = 4.0;

    for (int i = 0; i < 4; i++) {
        printf("%.1f %s %.1f = %.2f\n", x, symbols[i], y, operations[i](x, y));
    }

    return 0;
}
```

```
10.0 + 4.0 = 14.00
10.0 - 4.0 = 6.00
10.0 * 4.0 = 40.00
10.0 / 4.0 = 2.50
```

การประกาศ `double (*operations[4])(double, double)` อ่านจากในสุดออกมาได้ว่า
"`operations` เป็น array ขนาด 4 ของ pointer ไปยังฟังก์ชันที่รับ `(double, double)` และ
คืนค่า `double`" — ไวยากรณ์นี้ซับซ้อนพอสมควร ถ้าใช้งานบ่อยแนะนำให้สร้าง `typedef` เพื่อ
ความอ่านง่าย (จะเรียนเต็มรูปแบบใน Part 10):

```c
typedef double (*BinaryOp)(double, double);
BinaryOp operations[4] = { op_add, op_subtract, op_multiply, op_divide };
```

---

## 9.6 `const` กับ Pointer: 4 รูปแบบ (Step 70)

`const` กับ pointer เป็นเรื่องที่สร้างความสับสนให้มือใหม่มากที่สุดเรื่องหนึ่ง เพราะ `const`
สามารถ "ล็อก" ได้ 2 สิ่งที่ต่างกันโดยสิ้นเชิง: **(1) ข้อมูลที่ pointer ชี้ไป** และ **(2) ตัว
pointer เอง (address ที่มันเก็บอยู่)** ทำให้เกิดชุดค่าผสมได้ทั้งหมด 4 แบบ

**เคล็ดลับการอ่าน**: อ่านประกาศจาก**ขวาไปซ้าย** โดยดูว่า `const` อยู่ข้าง `*` ฝั่งไหน — ถ้า
`const` อยู่ **ก่อน `*`** แปลว่าล็อกข้อมูลปลายทาง ถ้า `const` อยู่ **หลัง `*`** (ติดกับชื่อ
ตัวแปร) แปลว่าล็อกตัว pointer เอง

```c
/* const_pointer_four_forms.c
 * สาธิตครบทั้ง 4 รูปแบบของ const กับ pointer
 */
#include <stdio.h>

int main(void) {
    int a = 10, b = 20;

    /* 1) Pointer ธรรมดา ไม่มี const เลย: แก้ทั้งข้อมูลและเปลี่ยนทิศทางได้ */
    int *p1 = &a;
    *p1 = 11;      /* แก้ข้อมูลได้ */
    p1 = &b;       /* เปลี่ยนให้ชี้ตัวอื่นได้ */

    /* 2) Pointer to const: แก้ "ข้อมูล" ที่ชี้อยู่ไม่ได้ แต่เปลี่ยนทิศทางได้ */
    const int *p2 = &a;
    /* *p2 = 99;    ห้ามทำ: compile error เพราะข้อมูลปลายทางเป็น const */
    p2 = &b;        /* เปลี่ยนทิศทางได้ ปกติ */

    /* 3) Const pointer: แก้ "ข้อมูล" ได้ แต่เปลี่ยนทิศทางไม่ได้ */
    int *const p3 = &a;
    *p3 = 33;       /* แก้ข้อมูลได้ */
    /* p3 = &b;      ห้ามทำ: compile error เพราะตัว pointer เองเป็น const */

    /* 4) Const pointer to const: แก้ข้อมูลก็ไม่ได้ เปลี่ยนทิศทางก็ไม่ได้ (ล็อกทั้งคู่) */
    const int *const p4 = &a;
    /* *p4 = 44;     ห้ามทำ */
    /* p4 = &b;      ห้ามทำ */

    printf("a = %d, b = %d\n", a, b);
    printf("p2 ชี้ไปที่ b แล้ว, *p2 = %d\n", *p2);
    printf("p3 แก้ค่า a ผ่านตัวมันเองแล้ว: *p3 = %d\n", *p3);
    printf("p4 อ่านค่าได้อย่างเดียว: *p4 = %d\n", *p4);

    return 0;
}
```

```
a = 33, b = 20
p2 ชี้ไปที่ b แล้ว, *p2 = 20
p3 แก้ค่า a ผ่านตัวมันเองแล้ว: *p3 = 33
p4 อ่านค่าได้อย่างเดียว: *p4 = 33
```

### ตารางสรุปทั้ง 4 รูปแบบ

| รูปแบบการประกาศ | แก้ไข "ข้อมูล" ที่ชี้อยู่ได้ไหม | เปลี่ยน "ทิศทาง" (ให้ชี้ตัวอื่น) ได้ไหม | ใช้เมื่อไร |
|---|---|---|---|
| `int *p;` | ✅ ได้ | ✅ ได้ | pointer ทั่วไปที่ต้องแก้ทั้งข้อมูลและย้ายทิศทางได้อิสระ |
| `const int *p;` (Pointer to const) | ❌ ไม่ได้ | ✅ ได้ | รับ array/ข้อมูลเข้าฟังก์ชันแบบ "อ่านอย่างเดียว" แต่ยังอยากเลื่อน pointer เดินไปเรื่อยๆ ได้ |
| `int *const p;` (Const pointer) | ✅ ได้ | ❌ ไม่ได้ | pointer ที่ต้อง "ตรึง" ให้ชี้ที่เดิมตลอดชีวิตของมัน แต่ยังแก้ข้อมูลปลายทางได้ |
| `const int *const p;` (Const pointer to const) | ❌ ไม่ได้ | ❌ ไม่ได้ | ค่าคงที่แบบเต็มรูปแบบ ทั้ง address และข้อมูลห้ามเปลี่ยนแม้แต่นิดเดียว |

### กรณีใช้งานจริงที่พบบ่อยที่สุด: `const` ในพารามิเตอร์ฟังก์ชัน

รูปแบบที่สองในตาราง (`const int *`) เป็นรูปแบบที่เจอบ่อยที่สุดในโค้ดจริง โดยเฉพาะเวลาส่ง
array หรือ string เข้าไปให้ฟังก์ชัน "อ่าน" ค่าอย่างเดียวโดยไม่ต้องการให้ฟังก์ชันแก้ไขต้นฉบับ
ได้เลย (เคยเห็นแล้วในฟังก์ชัน `print_array` ของหัวข้อ 9.5 ที่ใช้ `const int *arr`):

```c
/* const_param_protection.c */
#include <stdio.h>

/* ประกาศว่าฟังก์ชันนี้จะไม่แก้ไขข้อมูลใน arr เด็ดขาด - compiler ช่วยการันตีให้ */
int sum_array(const int *arr, int size) {
    int total = 0;
    for (int i = 0; i < size; i++) {
        total += arr[i];
        /* arr[i] = 0;   ถ้าลองแก้ค่า จะเป็น compile error ทันที เพราะ arr เป็น const */
    }
    return total;
}

int main(void) {
    int numbers[4] = {5, 10, 15, 20};
    printf("ผลรวม = %d\n", sum_array(numbers, 4));
    return 0;
}
```

การใส่ `const` ในพารามิเตอร์แบบนี้เรียกว่า **const-correctness** — เป็นวินัยที่โปรแกรมเมอร์
มืออาชีพยึดถือ เพราะทำให้ compiler ช่วยตรวจจับบั๊กที่อาจแก้ไขข้อมูลผิดจุดโดยไม่ตั้งใจได้
ล่วงหน้า ตั้งแต่ตอนคอมไพล์ ไม่ต้องรอไปเจอตอนรันจริง

---

## 9.7 Void Pointer: Pointer ที่ไม่ระบุชนิด (Step 71)

`void *` คือ **Generic Pointer** หรือ pointer ที่ "ไม่ผูกกับชนิดข้อมูลใดๆ" มันเก็บได้แค่ address
เฉยๆ โดยไม่รู้ว่าถ้า dereference แล้วจะได้ข้อมูลชนิดอะไร ด้วยเหตุนี้ **`void *` จึง dereference
ตรงๆ ไม่ได้** ต้อง cast ไปเป็น pointer ชนิดที่ถูกต้องก่อนเสมอ

```c
/* void_pointer_basic.c */
#include <stdio.h>

int main(void) {
    int number = 42;
    double price = 3.14;

    void *generic_ptr;

    generic_ptr = &number;
    /* printf("%d\n", *generic_ptr);   ห้ามทำ: compile error เพราะไม่รู้ชนิดข้อมูล */
    printf("number ผ่าน void*: %d\n", *(int *)generic_ptr);   /* ต้อง cast ก่อน */

    generic_ptr = &price;
    printf("price ผ่าน void*:  %.2f\n", *(double *)generic_ptr);

    return 0;
}
```

```
number ผ่าน void*: 42
price ผ่าน void*:  3.14
```

### ทำไม `void *` ถึงมีประโยชน์

`void *` ถูกใช้เมื่อเราต้องการเขียนฟังก์ชันที่ **ทำงานได้กับข้อมูลชนิดใดก็ได้** โดยไม่รู้ล่วงหน้า
ว่าจะเป็นชนิดอะไร ตัวอย่างที่คุ้นเคยที่สุดคือ `malloc` (จะเรียนเต็มใน Part 11) ที่คืนค่าเป็น
`void *` เสมอ เพราะมันไม่รู้ว่าผู้เรียกจะเอาหน่วยความจำนั้นไปเก็บ `int`, `double`, หรือ struct
ใดๆ ก็ตาม, และ `memcpy`, `memset` ที่รับพารามิเตอร์เป็น `void *` เพื่อคัดลอก/เซ็ตค่าหน่วยความจำ
แบบดิบๆ ได้กับข้อมูลทุกชนิด

> **ข้อจำกัดสำคัญของ `void *`**: มาตรฐาน C **ไม่อนุญาต** ให้ทำ pointer arithmetic
> (`generic_ptr + 1`) หรือ dereference (`*generic_ptr`) กับ `void *` โดยตรง เพราะไม่มี
> ขนาดข้อมูลให้อ้างอิง ต้อง cast เป็น pointer ชนิดที่รู้ขนาดแน่นอนก่อนเสมอ (เช่น `(int *)`,
> `(char *)`) — โค้ดที่เขียน `(char *)generic_ptr + n` เพื่อขยับทีละไบต์เป็น pattern ที่นิยม
> ใช้ในโค้ดระดับ low-level เช่น memory allocator เอง

---

## 9.8 ผสมผสาน void Pointer และ Function Pointer: `qsort` (Step 72)

ไลบรารีมาตรฐาน C มีฟังก์ชัน `qsort` ใน `<stdlib.h>` ที่เป็นตัวอย่างชั้นยอดของการผสมผสาน
`void *` และ Function Pointer เข้าด้วยกันเพื่อสร้างฟังก์ชันเรียงลำดับที่ **ใช้ได้กับข้อมูลชนิด
ใดก็ได้** โดยไม่ต้องเขียนฟังก์ชัน sort ใหม่ทุกครั้งที่ชนิดข้อมูลเปลี่ยน

Signature ของ `qsort` คือ:

```c
void qsort(void *base, size_t nmemb, size_t size,
           int (*compar)(const void *, const void *));
```

- `base`: pointer ไปยัง array ที่จะเรียง (เป็น `void *` เพราะไม่รู้ล่วงหน้าว่าเป็น array ชนิดใด)
- `nmemb`: จำนวน element ทั้งหมด
- `size`: ขนาด (ไบต์) ของ element แต่ละตัว (ได้จาก `sizeof`)
- `compar`: **Function Pointer แบบ Callback** ที่บอกวิธีเปรียบเทียบ element สองตัว

```c
/* qsort_demo.c
 * สาธิตการใช้ qsort กับ array ของ int โดยผสม void* และ function pointer
 */
#include <stdio.h>
#include <stdlib.h>

/* ฟังก์ชันเปรียบเทียบ: qsort กำหนด signature ตายตัวว่าต้องรับ const void* สองตัว */
int compare_ints(const void *a, const void *b) {
    int int_a = *(const int *)a;   /* ต้อง cast void* กลับเป็น int* ก่อน dereference */
    int int_b = *(const int *)b;

    if (int_a < int_b) return -1;
    if (int_a > int_b) return 1;
    return 0;
}

int compare_ints_descending(const void *a, const void *b) {
    return compare_ints(b, a);   /* สลับลำดับ argument เพื่อเรียงจากมากไปน้อย */
}

void print_array(const int *arr, int size) {
    for (int i = 0; i < size; i++) {
        printf("%d ", arr[i]);
    }
    printf("\n");
}

int main(void) {
    int numbers[7] = {42, 7, 19, 3, 88, 25, 1};

    printf("ก่อนเรียง:        ");
    print_array(numbers, 7);

    qsort(numbers, 7, sizeof(int), compare_ints);
    printf("เรียงจากน้อยไปมาก: ");
    print_array(numbers, 7);

    qsort(numbers, 7, sizeof(int), compare_ints_descending);
    printf("เรียงจากมากไปน้อย: ");
    print_array(numbers, 7);

    return 0;
}
```

```
ก่อนเรียง:        42 7 19 3 88 25 1
เรียงจากน้อยไปมาก: 1 3 7 19 25 42 88
เรียงจากมากไปน้อย: 88 42 25 19 7 3 1
```

สิ่งที่เกิดขึ้นเบื้องหลัง: `qsort` เองไม่รู้เลยว่ากำลังเรียง `int`, `double`, หรือแม้แต่ struct
ที่ซับซ้อน มันแค่เดินไล่ผ่านหน่วยความจำดิบๆ ทีละ `size` ไบต์ (ใช้ `void *` + arithmetic แบบ
`(char *)base + i * size` ภายใน) แล้วเรียก `compar` (function pointer ที่เราส่งเข้าไป) เพื่อถาม
ว่า "element สองตัวนี้ ใครมาก่อนกัน" เท่านั้นเอง — นี่คือรูปแบบการออกแบบที่เรียกว่า **Generic
Programming ผ่าน Runtime Polymorphism แบบ C** ซึ่งเป็นแนวคิดเดียวกับที่ C++ จะทำให้สวยงาม
ขึ้นด้วย Template ใน Module E ของหลักสูตรนี้

---

## ข้อผิดพลาดที่พบบ่อย (Common Pitfalls)

1. **คิดว่า pointer arithmetic บวกทีละ 1 ไบต์เสมอ** — `ptr + 1` ขยับตาม `sizeof` ของชนิด
   ข้อมูลที่ pointer ชี้อยู่ ไม่ใช่ 1 ไบต์ตรงๆ (ยกเว้น `char *` ที่บังเอิญ `sizeof(char) == 1`)
   ความเข้าใจผิดนี้มักนำไปสู่บั๊ก "off-by-N ไบต์" เวลาผสม pointer arithmetic กับการคำนวณ
   ขนาดหน่วยความจำเอง

2. **ลืมวงเล็บตอนประกาศ Function Pointer** — `int *f(int, int);` กับ `int (*f)(int, int);`
   เป็นคนละความหมายกันโดยสิ้นเชิง! แบบแรกคือ "ฟังก์ชันชื่อ `f` ที่คืนค่าเป็น `int *`" (function
   declaration ธรรมดา) ส่วนแบบหลังคือ "ตัวแปร `f` ที่เป็น pointer ไปยังฟังก์ชัน" เพราะ `()` มี
   precedence สูงกว่า `*` ต้องใส่วงเล็บรอบ `*f` เสมอเมื่อประกาศ function pointer

3. **เปรียบเทียบ pointer จาก array คนละก้อน** — การเขียน `if (ptr_a < ptr_b)` โดยที่ `ptr_a`
   และ `ptr_b` ชี้ไปยัง array คนละตัวกัน เป็น Undefined Behavior ตามมาตรฐาน แม้ในทางปฏิบัติ
   จะได้ผลลัพธ์ (ที่ไม่มีความหมาย) กลับมาโดยไม่ crash ก็ตาม ควรเปรียบเทียบ pointer ภายใน
   array เดียวกันเท่านั้น

4. **สับสน `const int *p` กับ `int *const p`** — นี่คือกับดักคลาสสิกของ const-pointer จำ
   หลักการ "อ่านจากใกล้ชื่อตัวแปรออกไป": `const` ที่อยู่ **ติดกับชื่อตัวแปร** (`*const p`)
   ล็อกตัว pointer เอง ส่วน `const` ที่อยู่ **ห่างจากชื่อตัวแปร** (`const int *p`) ล็อกข้อมูล
   ปลายทาง ถ้าจำสับสนให้กลับไปดูตารางสรุปในหัวข้อ 9.6 ทุกครั้ง

5. **Dereference `void *` โดยตรงโดยไม่ cast ก่อน** — `*generic_ptr` เมื่อ `generic_ptr`
   เป็น `void *` จะเป็น **Compile Error** ทันที (ไม่ใช่แค่ Warning) เพราะ compiler ไม่รู้ว่าจะ
   อ่านกี่ไบต์และตีความอย่างไร ต้อง cast เป็นชนิดที่ถูกต้องก่อนเสมอ เช่น `*(int *)generic_ptr`

6. **ลืมว่า function pointer callback (เช่นใน `qsort`) ต้องมี signature ตรงกันเป๊ะ** — ถ้า
   ฟังก์ชันเปรียบเทียบที่ส่งให้ `qsort` มี signature ไม่ตรงกับที่กำหนด (`int (*)(const void *,
   const void *)`) เช่นลืมใส่ `const` หรือใช้ชนิดพารามิเตอร์ผิด จะได้ Warning เรื่อง
   incompatible pointer type ภายใต้ `-Wall -Wextra` ทันที ควรอ่าน signature ที่ต้องการจาก
   man page (`man 3 qsort`) ให้ตรงเป๊ะก่อนเขียนฟังก์ชันเปรียบเทียบเสมอ

---

## แบบฝึกหัดท้ายบท

1. เขียนโปรแกรมที่มี `double arr[4]` แล้วใช้ pointer arithmetic (`ptr + i`) พิมพ์ address ของ
   แต่ละ element ออกมา พิสูจน์ด้วยตาตัวเองว่าแต่ละ address ห่างกัน 8 ไบต์จริง (บนเครื่อง
   ที่ `sizeof(double) == 8`)

2. เขียนฟังก์ชัน `void swap_via_pp(int **a, int **b)` ที่รับ pointer to pointer สองตัว แล้ว
   สลับให้ `*a` กับ `*b` ชี้สลับที่กัน (ไม่ใช่สลับค่าข้อมูล แต่สลับ "ทิศทาง" ที่แต่ละตัวชี้ไป)
   แล้วทดสอบกับ `int x = 1, y = 2; int *p = &x, *q = &y;`

3. เขียน Dispatch Table ของฟังก์ชันเปรียบเทียบตัวเลข 3 แบบ: `is_positive`, `is_negative`,
   `is_zero` (แต่ละฟังก์ชันรับ `int` คืนค่า `int` แบบ boolean 0/1) แล้ววนลูปทดสอบทุกฟังก์ชัน
   กับค่าตัวเลขชุดหนึ่ง

4. ให้เติมคำว่า `const` ในตำแหน่งที่ถูกต้องสำหรับแต่ละสถานการณ์ต่อไปนี้ (เขียนโค้ดจริงมาทดสอบ
   ด้วย): (ก) pointer ที่ต้องอ่าน array อย่างเดียวแต่เลื่อนตำแหน่งได้, (ข) pointer ที่ต้องชี้ไปที่
   ตัวแปรตัวเดิมตลอดฟังก์ชัน แต่แก้ไขค่าของมันได้เรื่อยๆ

5. เขียนโปรแกรมที่มี `struct { int id; char name[20]; } students[3]` แล้วใช้ `qsort` เรียง
   นักเรียนตาม `id` จากน้อยไปมาก (ต้องเขียนฟังก์ชันเปรียบเทียบเองที่ cast `const void *`
   กลับเป็น pointer ของ struct นักเรียนก่อน)

6. อธิบายด้วยคำพูดของตัวเองว่าทำไม `qsort` ถึงต้องรับพารามิเตอร์ `size` (ขนาดของ element)
   เพิ่มเข้ามาด้วย ทั้งที่มันก็รับ `void *base` มาแล้ว (ใบ้: คิดถึงคำตอบของ Pitfall ข้อ 1)

### แนวทางเฉลยข้อ 2

```c
/* swap_via_pp.c */
#include <stdio.h>

void swap_via_pp(int **a, int **b) {
    int *temp = *a;
    *a = *b;
    *b = temp;
}

int main(void) {
    int x = 1, y = 2;
    int *p = &x;
    int *q = &y;

    printf("ก่อน swap: *p = %d, *q = %d\n", *p, *q);

    swap_via_pp(&p, &q);

    printf("หลัง swap: *p = %d, *q = %d\n", *p, *q);
    printf("(x และ y ไม่ได้เปลี่ยนค่าเลย x = %d, y = %d, แต่ p กับ q สลับทิศทางกัน)\n", x, y);

    return 0;
}
```

```
ก่อน swap: *p = 1, *q = 2
หลัง swap: *p = 2, *q = 1
(x และ y ไม่ได้เปลี่ยนค่าเลย x = 1, y = 2, แต่ p กับ q สลับทิศทางกัน)
```

จุดสำคัญของเฉลยนี้คือ `x` และ `y` **ไม่ได้ถูกแก้ไขค่าเลย** — สิ่งที่เปลี่ยนคือ `p` ที่แต่เดิมชี้
ไปที่ `x` ตอนนี้ไปชี้ที่ `y` แทน (และในทางกลับกันสำหรับ `q`) นี่คือความแตกต่างสำคัญระหว่าง
"สลับข้อมูล" (`swap` ใน Part 8) กับ "สลับทิศทางของ pointer" (`swap_via_pp` ข้อนี้)

### แนวทางเฉลยข้อ 5

```c
/* qsort_students.c */
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

struct student {
    int id;
    char name[20];
};

int compare_by_id(const void *a, const void *b) {
    const struct student *sa = (const struct student *)a;
    const struct student *sb = (const struct student *)b;

    if (sa->id < sb->id) return -1;
    if (sa->id > sb->id) return 1;
    return 0;
}

int main(void) {
    struct student students[3] = {
        {103, "Chai"},
        {101, "Bee"},
        {102, "Ann"}
    };

    qsort(students, 3, sizeof(struct student), compare_by_id);

    for (int i = 0; i < 3; i++) {
        printf("id=%d name=%s\n", students[i].id, students[i].name);
    }

    return 0;
}
```

```
id=101 name=Bee
id=102 name=Ann
id=103 name=Chai
```

`sizeof(struct student)` บอก `qsort` ว่า element แต่ละตัวกินพื้นที่กี่ไบต์ (รวม padding ถ้ามี
ซึ่งจะอธิบายเต็มใน Part 10) ทำให้มันเดินหน่วยความจำแบบดิบๆ ได้ถูกต้องโดยไม่ต้องรู้เลยว่า
`struct student` มีหน้าตาเป็นอย่างไรข้างใน — นี่คือพลังของการผสม `void *` กับ `size` และ
function pointer เข้าด้วยกัน

---

## สรุปท้ายบท

ใน Part นี้เราได้:

- เข้าใจ Pointer Arithmetic อย่างละเอียด และรู้ว่าการบวก/ลบ pointer คำนวณตาม `sizeof`
  ของชนิดข้อมูลเสมอ ไม่ใช่ทีละ 1 ไบต์
- เปรียบเทียบและลบ pointer เพื่อหาระยะห่างระหว่าง element ด้วย `ptrdiff_t`
- เข้าใจและใช้งาน Pointer to Pointer (`**`) รวมถึงกรณีที่ต้องใช้จริงเมื่อฟังก์ชันต้องเปลี่ยน
  ทิศทางของ pointer ของผู้เรียก
- ประกาศและใช้งาน Function Pointer เป็น Callback และสร้าง Dispatch Table ได้
- แยกแยะ `const` กับ pointer ได้ครบทั้ง 4 รูปแบบอย่างแม่นยำ พร้อมรู้ว่าจะใช้แบบไหนเมื่อไร
- ใช้งาน `void *` เป็น Generic Pointer ได้อย่างปลอดภัย
- ผสมผสาน Function Pointer และ `void *` เพื่อใช้ `qsort` เรียงข้อมูลชนิดใดก็ได้

ทักษะเรื่อง Pointer ทั้งหมดที่เรียนใน Part 8 และ 9 คือรากฐานที่สำคัญที่สุดของภาษา C
ตั้งแต่นี้ไปแทบทุก Part ในหลักสูตรจะใช้แนวคิดเหล่านี้ ไม่ว่าจะเป็น Dynamic Memory (Part 11),
Linked List (Part 19), หรือแม้แต่ Smart Pointer ใน C++ (Part 67) ที่ก็สร้างอยู่บนความเข้าใจ
พื้นฐานเรื่อง pointer ที่เพิ่งเรียนไปนี้เอง ใน **Part 10** เราจะเปลี่ยนไปเรียนวิธีสร้าง
**ชนิดข้อมูลของตัวเอง** ผ่าน Struct, Union, Enum และ typedef ซึ่งจะทำงานร่วมกับ Pointer
ที่เรียนมาอย่างแนบแน่น

**ต่อไป:** [Part 10 — Struct, Union, Enum และ typedef](./part-010-struct-union-enum.md)
