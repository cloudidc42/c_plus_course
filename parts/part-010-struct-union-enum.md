# Part 10: Struct, Union, Enum และ typedef (Step 73–80)

> Module A — รากฐานภาษา C (Foundations of C) | Part 10 จาก 125
> Step ที่ครอบคลุมใน Part นี้: Step 73–80
> Part ก่อนหน้า: [Part 9 — Pointer ขั้นสูง (ตอนที่ 2)](./part-009-pointers-advanced.md) | Part ถัดไป: [Part 11 — Dynamic Memory Allocation](./part-011-dynamic-memory.md)

## เป้าหมายการเรียนรู้ (Learning Objectives)

หลังจบ Part นี้ ผู้เรียนจะสามารถ:

1. ประกาศ `struct` และเข้าถึง member ของมันได้ทั้งด้วย `.` และ `->` อย่างถูกต้อง
2. ออกแบบ Nested Struct (struct ที่มี struct อื่นซ้อนอยู่ข้างใน) ได้
3. ใช้งาน struct ร่วมกับ pointer ได้อย่างคล่องแคล่ว รวมถึงส่ง struct เข้าฟังก์ชันอย่างมี
   ประสิทธิภาพ
4. อธิบายปรากฏการณ์ Memory Padding และ Alignment ของ struct ได้ พร้อมพิสูจน์ด้วย `sizeof`
   จริง และจัดเรียง member ใหม่เพื่อลด padding ได้
5. อธิบายความแตกต่างระหว่าง `union` กับ `struct` ได้อย่างชัดเจน และเลือกใช้ให้เหมาะกับงาน
6. ใช้ `enum` เพื่อทำให้โค้ดอ่านง่ายขึ้นแทนการใช้ตัวเลข "มายากล" (Magic Number)
7. ใช้ `typedef` สร้างชื่อชนิดข้อมูลของตัวเองเพื่อลดความซับซ้อนของไวยากรณ์

---

## 10.1 Struct คืออะไร: การประกาศและเข้าถึง Member (Step 73)

จนถึงตอนนี้เรารู้จักแต่ชนิดข้อมูลพื้นฐาน (`int`, `double`, `char`) และ array (ที่เก็บข้อมูล
**ชนิดเดียวกัน** หลายตัว) แต่ในโลกจริง ข้อมูลมักประกอบด้วยหลายชนิดปนกัน เช่น "จุดบนกราฟ"
ต้องมีทั้งค่า `x` และ `y`, "นักเรียนคนหนึ่ง" ต้องมีทั้งชื่อ (ข้อความ), อายุ (ตัวเลข), และเกรด
เฉลี่ย (ทศนิยม) — **`struct`** คือกลไกของ C ที่ให้เรารวมข้อมูลหลายชนิดเข้าเป็น "ก้อนเดียว"
ภายใต้ชื่อเดียวกัน

```c
/* struct_basic.c
 * สาธิตการประกาศ struct และเข้าถึง member ด้วย .
 */
#include <stdio.h>

struct point {
    int x;
    int y;
};

int main(void) {
    struct point p1;   /* ประกาศตัวแปรชนิด struct point */

    p1.x = 3;           /* เข้าถึง member ด้วยเครื่องหมายจุด (.) */
    p1.y = 4;

    printf("p1 = (%d, %d)\n", p1.x, p1.y);

    /* ประกาศพร้อมกำหนดค่าเริ่มต้นได้เลยแบบ array */
    struct point p2 = {10, 20};
    printf("p2 = (%d, %d)\n", p2.x, p2.y);

    /* กำหนดค่าด้วยชื่อ member ตรงๆ (Designated Initializer, มีตั้งแต่ C99) */
    struct point p3 = {.x = 7, .y = 9};
    printf("p3 = (%d, %d)\n", p3.x, p3.y);

    return 0;
}
```

```
p1 = (3, 4)
p2 = (10, 20)
p3 = (7, 9)
```

**อุปมา**: `struct` เปรียบเหมือนแบบฟอร์มกระดาษหนึ่งใบที่มีช่องกรอกหลายช่อง (`x`, `y`)
เวลาเราสร้างตัวแปร `struct point p1;` ก็เหมือนถ่ายเอกสารแบบฟอร์มนั้นออกมาหนึ่งใบเปล่าๆ
แล้วค่อยกรอกข้อมูลลงไปทีหลังด้วย `p1.x = 3;`

### `struct` เทียบกับ `array`: ใช้เมื่อไรกันแน่

| ลักษณะข้อมูล | ใช้ Array | ใช้ Struct |
|---|---|---|
| ชนิดข้อมูลเดียวกันหลายตัว (เช่น คะแนนสอบ 30 คน) | ✅ เหมาะ | ไม่จำเป็น |
| ข้อมูลหลายชนิดปนกันเป็นหน่วยเดียว (เช่น จุด x,y หรือนักเรียนคนหนึ่ง) | ไม่เหมาะ | ✅ เหมาะ |
| เข้าถึงด้วย index ตัวเลข | ✅ ใช้ `[i]` | ไม่มี index |
| เข้าถึงด้วยชื่อความหมาย | ไม่มี | ✅ ใช้ชื่อ member |

---

## 10.2 Struct กับ Pointer: ตัวดำเนินการ `->` (Step 74)

เมื่อมี pointer ชี้ไปยัง struct การเข้าถึง member ผ่าน `.` ตรงๆ ทำไม่ได้ (เพราะ pointer ไม่ใช่
struct โดยตรง) วิธีที่ถูกต้องคือ dereference ก่อนแล้วค่อยใช้ `.` หรือใช้ตัวดำเนินการลัด `->`
ที่ทำสองอย่างนี้ในขั้นตอนเดียว

```c
/* struct_pointer_arrow.c */
#include <stdio.h>

struct point {
    int x;
    int y;
};

void move_point(struct point *p, int dx, int dy) {
    p->x = p->x + dx;   /* ทางลัดของ (*p).x */
    p->y += dy;         /* ผสมกับตัวดำเนินการอื่นได้ตามปกติ */
}

int main(void) {
    struct point origin = {0, 0};
    struct point *ptr = &origin;

    printf("ก่อนขยับ: (%d, %d)\n", (*ptr).x, (*ptr).y);   /* เขียนแบบเต็มได้เช่นกัน */
    printf("ก่อนขยับ (แบบลัด): (%d, %d)\n", ptr->x, ptr->y);

    move_point(ptr, 5, 3);

    printf("หลังขยับ: (%d, %d)\n", origin.x, origin.y);

    return 0;
}
```

```
ก่อนขยับ: (0, 0)
ก่อนขยับ (แบบลัด): (0, 0)
หลังขยับ: (5, 3)
```

**กฎง่ายๆ ที่จำได้แม่นตลอดหลักสูตรนี้**:

- ถ้ามี **struct ตัวจริง** อยู่ในมือ → ใช้ `.` (เช่น `origin.x`)
- ถ้ามี **pointer ไปยัง struct** อยู่ในมือ → ใช้ `->` (เช่น `ptr->x`)
- `ptr->x` เขียนแบบเต็มคือ `(*ptr).x` เสมอ (ต้องมีวงเล็บ เพราะ `.` มี precedence สูงกว่า `*`)

### ทำไมเวลาส่ง struct เข้าฟังก์ชันมักส่งเป็น pointer

```c
/* struct_pass_by_pointer_efficiency.c */
#include <stdio.h>

struct big_record {
    char name[64];
    double history[100];
    int id;
};

/* ส่งแบบ pointer: คัดลอกแค่ address (8 ไบต์) เข้าไปในฟังก์ชัน ไม่ว่า struct จะใหญ่แค่ไหน */
void print_id(const struct big_record *record) {
    printf("id = %d\n", record->id);
}

int main(void) {
    struct big_record student = {"Somchai", {0}, 1001};
    print_id(&student);
    printf("sizeof(struct big_record) = %zu ไบต์ (ใหญ่มาก ถ้าส่งแบบ pass by value จะคัดลอกทั้งก้อน)\n",
           sizeof student);
    return 0;
}
```

```
id = 1001
sizeof(struct big_record) = 1608 ไบต์ (ใหญ่มาก ถ้าส่งแบบ pass by value จะคัดลอกทั้งก้อน)
```

ถ้าฟังก์ชัน `print_id` รับพารามิเตอร์เป็น `struct big_record record` ตรงๆ (pass by value)
โปรแกรมจะต้อง**คัดลอก struct ทั้งก้อน** (1608 ไบต์ในตัวอย่างนี้) เข้าไปในฟังก์ชันทุกครั้งที่
เรียก ซึ่งช้าและสิ้นเปลืองหน่วยความจำมาก การส่งเป็น pointer (`const struct big_record *`)
คัดลอกแค่ address เดียว (8 ไบต์) เท่านั้น พร้อมใส่ `const` เพื่อการันตีว่าฟังก์ชันจะไม่แก้ไข
ข้อมูลต้นฉบับ — นี่คือรูปแบบมาตรฐานที่โค้ด C มืออาชีพใช้เสมอเมื่อ struct มีขนาดใหญ่

---

## 10.3 Nested Struct: Struct ที่ซ้อนกัน (Step 75)

Struct หนึ่งตัวสามารถมี struct อีกตัวเป็น member ข้างในได้ เรียกว่า **Nested Struct** ซึ่งช่วย
ให้จัดกลุ่มข้อมูลที่สัมพันธ์กันเป็นชั้นๆ ได้เป็นธรรมชาติมากขึ้น

```c
/* nested_struct.c */
#include <stdio.h>

struct date {
    int day;
    int month;
    int year;
};

struct employee {
    char name[30];
    struct date hire_date;   /* struct ซ้อนอยู่ข้างใน struct */
    double salary;
};

int main(void) {
    struct employee emp = {
        .name = "Napat",
        .hire_date = {15, 6, 2023},
        .salary = 35000.0
    };

    /* เข้าถึง member ของ struct ที่ซ้อนอยู่ข้างใน ใช้ . ต่อกันเป็นทอดๆ */
    printf("ชื่อ: %s\n", emp.name);
    printf("วันที่เริ่มงาน: %d/%d/%d\n",
           emp.hire_date.day, emp.hire_date.month, emp.hire_date.year);
    printf("เงินเดือน: %.2f\n", emp.salary);

    /* แก้ไขข้อมูลชั้นในได้ตามปกติ */
    emp.hire_date.year = 2024;
    printf("ปีที่เริ่มงานหลังแก้ไข: %d\n", emp.hire_date.year);

    return 0;
}
```

```
ชื่อ: Napat
วันที่เริ่มงาน: 15/6/2023
เงินเดือน: 35000.00
ปีที่เริ่มงานหลังแก้ไข: 2024
```

ถ้า struct ที่ซ้อนอยู่ถูกเข้าถึงผ่าน pointer แทน ก็ผสม `.` และ `->` เข้าด้วยกันได้ตามชนิดของ
แต่ละชั้น:

```c
struct employee *pe = &emp;
printf("%d\n", pe->hire_date.year);   /* pe เป็น pointer ใช้ ->, hire_date เป็น struct ใช้ . */
```

---

## 10.4 Memory Padding และ Alignment ของ Struct (Step 76)

หนึ่งในเรื่องที่มือใหม่ (และแม้แต่โปรแกรมเมอร์ที่มีประสบการณ์บางคน) มักแปลกใจคือ ขนาดของ
`struct` ที่ได้จาก `sizeof` **ไม่ได้เท่ากับผลรวมของขนาด member ทุกตัวบวกกันตรงๆ เสมอไป**
เพราะ CPU อ่าน/เขียนหน่วยความจำได้เร็วที่สุดเมื่อข้อมูลถูกวางในตำแหน่งที่ "align" กับขนาดของ
มันเอง (เช่น `int` 4 ไบต์ควรอยู่ที่ address ที่หารด้วย 4 ลงตัว) Compiler จึงแทรก **Padding**
(ช่องว่างเปล่าที่ไม่ได้ใช้งาน) เข้าไปคั่นระหว่าง member เพื่อรักษากฎ Alignment นี้

```c
/* struct_padding_demo.c
 * สาธิต memory padding ด้วยการจัดเรียง member สองแบบที่ต่างกัน
 */
#include <stdio.h>
#include <stddef.h>

/* เวอร์ชันที่ 1: เรียง member แบบ "สลับขนาด" ทำให้เกิด padding เยอะ */
struct bad_layout {
    char  flag;     /* 1 ไบต์ */
    int   number;   /* 4 ไบต์ */
    char  grade;    /* 1 ไบต์ */
    double score;   /* 8 ไบต์ */
};

/* เวอร์ชันที่ 2: เรียง member จากใหญ่ไปเล็ก ทำให้ padding น้อยลง */
struct good_layout {
    double score;   /* 8 ไบต์ */
    int   number;   /* 4 ไบต์ */
    char  flag;     /* 1 ไบต์ */
    char  grade;    /* 1 ไบต์ */
};

int main(void) {
    printf("sizeof(struct bad_layout)  = %zu ไบต์\n", sizeof(struct bad_layout));
    printf("sizeof(struct good_layout) = %zu ไบต์\n", sizeof(struct good_layout));

    printf("\noffset ของแต่ละ member ใน bad_layout:\n");
    printf("  flag   อยู่ที่ offset %zu\n", offsetof(struct bad_layout, flag));
    printf("  number อยู่ที่ offset %zu\n", offsetof(struct bad_layout, number));
    printf("  grade  อยู่ที่ offset %zu\n", offsetof(struct bad_layout, grade));
    printf("  score  อยู่ที่ offset %zu\n", offsetof(struct bad_layout, score));

    printf("\noffset ของแต่ละ member ใน good_layout:\n");
    printf("  score  อยู่ที่ offset %zu\n", offsetof(struct good_layout, score));
    printf("  number อยู่ที่ offset %zu\n", offsetof(struct good_layout, number));
    printf("  flag   อยู่ที่ offset %zu\n", offsetof(struct good_layout, flag));
    printf("  grade  อยู่ที่ offset %zu\n", offsetof(struct good_layout, grade));

    return 0;
}
```

ผลลัพธ์ตัวอย่างบนเครื่อง x86-64 ทั่วไป:

```
sizeof(struct bad_layout)  = 24 ไบต์
sizeof(struct good_layout) = 16 ไบต์

offset ของแต่ละ member ใน bad_layout:
  flag   อยู่ที่ offset 0
  number อยู่ที่ offset 4
  grade  อยู่ที่ offset 8
  score  อยู่ที่ offset 16

offset ของแต่ละ member ใน good_layout:
  score  อยู่ที่ offset 0
  number อยู่ที่ offset 8
  flag   อยู่ที่ offset 12
  grade  อยู่ที่ offset 13
```

วิเคราะห์ `bad_layout` ทีละ member: `flag` (1 ไบต์) อยู่ offset 0, แต่ `number` (4 ไบต์)
ต้อง align ที่ offset ที่หารด้วย 4 ลงตัว จึงถูกดันไปที่ offset 4 (มี **padding 3 ไบต์**
แทรกระหว่าง `flag` กับ `number`) จากนั้น `grade` อยู่ offset 8 พอดี แต่ `score` (`double`
8 ไบต์) ต้อง align ที่ offset ที่หารด้วย 8 ลงตัว จึงถูกดันจาก offset 9 ไปเป็น offset 16
(padding อีก 7 ไบต์) และท้ายสุด struct ทั้งก้อนต้องมีขนาดเป็นทวีคูณของ alignment ที่มากที่สุด
(8 ไบต์ จาก `double`) จึงปัดจาก 24 ไปเป็น 24 พอดี (ลงตัวอยู่แล้ว) — รวม padding ทั้งหมด
10 ไบต์จาก 24 ไบต์!

ในขณะที่ `good_layout` เรียง member จากใหญ่ไปเล็ก (`double` → `int` → `char` → `char`)
ทำให้แทบไม่มี padding แทรกเลย ได้ขนาดเพียง 16 ไบต์ ประหยัดไป 8 ไบต์ต่อตัวแปรหนึ่งตัว — ถ้า
มี struct แบบนี้เป็นล้านตัวใน array (เช่นในระบบฐานข้อมูลหรือเกม) ความแตกต่างนี้มีผลต่อ
ประสิทธิภาพและการใช้หน่วยความจำอย่างมีนัยสำคัญจริง

> **กฎทองในการออกแบบ struct ให้ประหยัดหน่วยความจำ**: เรียง member **จากขนาดใหญ่ไปเล็ก**
> เสมอ (เช่น `double`/`long` ก่อน แล้วค่อย `int`, ปิดท้ายด้วย `char`/`bool`) จะช่วยลด
> Padding ได้อย่างมีประสิทธิภาพโดยไม่ต้องคำนวณเองให้ซับซ้อน

---

## 10.5 Union: โครงสร้างที่ใช้หน่วยความจำร่วมกัน (Step 77)

`union` มีไวยากรณ์การประกาศคล้าย `struct` มากแทบทุกประการ แต่มีความหมายที่ต่างกันโดย
สิ้นเชิง: **สมาชิกทุกตัวใน `union` ใช้พื้นที่หน่วยความจำ "ก้อนเดียวกัน" ซ้อนทับกัน** ไม่ได้แยก
คนละพื้นที่เหมือน `struct` ดังนั้นขนาดของ `union` จะเท่ากับขนาดของ member ที่ใหญ่ที่สุดเพียง
ตัวเดียวเท่านั้น (ไม่ใช่ผลรวมของทุก member) และ **ที่เวลาใดเวลาหนึ่ง จะมีข้อมูลที่ "ถูกต้อง"
อยู่แค่ member เดียวเท่านั้น** เพราะการเขียนทับ member หนึ่งจะเขียนทับข้อมูลของ member อื่น
ไปด้วยโดยอัตโนมัติ

```c
/* union_demo.c
 * สาธิตว่า union ใช้หน่วยความจำร่วมกันจริง
 */
#include <stdio.h>

union value {
    int as_int;
    float as_float;
    char as_bytes[4];
};

int main(void) {
    union value v;

    printf("sizeof(union value) = %zu ไบต์ (เท่ากับ member ที่ใหญ่ที่สุดเท่านั้น)\n",
           sizeof(union value));

    v.as_int = 65;
    printf("\nกำหนด v.as_int = 65\n");
    printf("v.as_int   = %d\n", v.as_int);
    printf("v.as_bytes[0] = %d (คือ byte แรกของเลข 65 ในหน่วยความจำเดียวกัน)\n",
           v.as_bytes[0]);

    v.as_float = 3.14f;
    printf("\nหลังจากกำหนด v.as_float = 3.14f แล้ว\n");
    printf("v.as_float = %.2f (ค่าที่ถูกต้อง)\n", v.as_float);
    printf("v.as_int   = %d (ค่าขยะ! เพราะ bit pattern ถูกตีความใหม่เป็น int)\n", v.as_int);

    return 0;
}
```

```
sizeof(union value) = 4 ไบต์ (เท่ากับ member ที่ใหญ่ที่สุดเท่านั้น)

กำหนด v.as_int = 65
v.as_int   = 65
v.as_bytes[0] = 65 (คือ byte แรกของเลข 65 ในหน่วยความจำเดียวกัน)

หลังจากกำหนด v.as_float = 3.14f แล้ว
v.as_float = 3.14 (ค่าที่ถูกต้อง)
v.as_int   = 1078523331 (ค่าขยะ! เพราะ bit pattern ถูกตีความใหม่เป็น int)
```

ตัวเลข `1078523331` ที่ได้ไม่ใช่ "บั๊ก" แต่เป็นพฤติกรรมที่ถูกต้องตามหลักการของ `union`:
มันคือ bit pattern ของเลขทศนิยม `3.14f` แบบ IEEE-754 เพียงแต่ถูก `v.as_int` มาอ่านและ
ตีความใหม่ในฐานะเลขจำนวนเต็มธรรมดา จึงได้ค่าที่ดูเหมือนสุ่มไปจากเดิม

### ตารางเปรียบเทียบ `struct` กับ `union`

| คุณสมบัติ | `struct` | `union` |
|---|---|---|
| การจัดสรรหน่วยความจำ | แต่ละ member มีพื้นที่แยกกัน | ทุก member ใช้พื้นที่**ร่วมกัน** |
| ขนาด (`sizeof`) | ผลรวมขนาดของทุก member (+ padding) | ขนาดของ member ที่ใหญ่ที่สุด (+ padding ถ้ามี) |
| ค่าที่ใช้งานได้พร้อมกัน | อ่าน/เขียนทุก member พร้อมกันได้ทั้งหมด | ใช้งานได้ครั้งละ 1 member เท่านั้น |
| กรณีใช้งานทั่วไป | เก็บข้อมูลหลายอย่างที่ต้องอยู่ครบพร้อมกัน (เช่น struct Student) | ประหยัดหน่วยความจำเมื่อรู้ว่าจะใช้แค่ค่าเดียวในแต่ละครั้ง เช่น Variant Type, การตีความ Bit Pattern ข้ามชนิดข้อมูล |

Union มักถูกใช้ร่วมกับ `enum` (จะเรียนในหัวขัดถัดไป) เพื่อสร้าง **Tagged Union** — โครงสร้าง
ที่มี `enum` บอกว่า "ตอนนี้ member ไหนของ union กำลังถูกใช้งานอยู่จริง" ซึ่งเป็นเทคนิคพื้นฐาน
ที่ใช้จำลอง Variant Type (คล้าย `std::variant` ใน C++ ที่จะเรียนใน Module F)

---

## 10.6 Enum: ทำให้โค้ดอ่านง่ายขึ้น (Step 78)

ลองนึกภาพโค้ดที่ใช้ตัวเลข `0`, `1`, `2` แทนสถานะของออเดอร์ (`0` = รอดำเนินการ, `1` =
กำลังจัดส่ง, `2` = ส่งสำเร็จ) — ตัวเลขเหล่านี้เรียกว่า **Magic Number** เพราะอ่านโค้ดแล้วไม่มี
ทางรู้ความหมายได้เลยถ้าไม่ไปเปิดคอมเมนต์หรือเอกสารดู `enum` (enumeration) คือกลไกที่ให้
เราตั้งชื่อความหมายให้กับกลุ่มค่าคงที่จำนวนเต็มเหล่านี้แทน

```c
/* enum_demo.c
 * สาธิตว่า enum ทำให้โค้ดอ่านง่ายขึ้นกว่าการใช้ magic number
 */
#include <stdio.h>

enum order_status {
    STATUS_PENDING,     /* ค่าเริ่มต้นคือ 0 */
    STATUS_SHIPPING,    /* อัตโนมัติเป็น 1 */
    STATUS_DELIVERED,   /* อัตโนมัติเป็น 2 */
    STATUS_CANCELLED    /* อัตโนมัติเป็น 3 */
};

const char *status_to_text(enum order_status status) {
    switch (status) {
        case STATUS_PENDING:   return "รอดำเนินการ";
        case STATUS_SHIPPING:  return "กำลังจัดส่ง";
        case STATUS_DELIVERED: return "ส่งสำเร็จ";
        case STATUS_CANCELLED: return "ยกเลิกแล้ว";
        default:                return "ไม่ทราบสถานะ";
    }
}

int main(void) {
    enum order_status current = STATUS_SHIPPING;

    printf("สถานะปัจจุบัน (ค่าตัวเลขจริง) = %d\n", current);
    printf("สถานะปัจจุบัน (อ่านง่าย)      = %s\n", status_to_text(current));

    if (current == STATUS_SHIPPING) {
        printf("พัสดุกำลังเดินทางอยู่\n");
    }

    return 0;
}
```

```
สถานะปัจจุบัน (ค่าตัวเลขจริง) = 1
สถานะปัจจุบัน (อ่านง่าย)      = กำลังจัดส่ง
พัสดุกำลังเดินทางอยู่
```

โดยดีฟอลต์ ค่าของสมาชิก `enum` แต่ละตัวจะเริ่มจาก `0` และเพิ่มทีละ `1` ตามลำดับที่เขียนไว้
แต่สามารถกำหนดค่าเริ่มต้นเองได้ (และค่าที่เหลือจะนับต่อจากค่านั้น):

```c
enum http_status {
    HTTP_OK = 200,
    HTTP_NOT_FOUND = 404,
    HTTP_SERVER_ERROR = 500,
    HTTP_SERVER_ERROR_2 = 501   /* จะนับต่อจากตัวก่อนหน้าถ้าไม่ได้กำหนดเอง เช่น 501 ถ้าไม่เขียนไว้ */
};
```

> **ข้อควรรู้**: ในภาษา C (ต่างจาก C++) ตัวแปรชนิด `enum` โดยพื้นฐานคือ `int` และค่า enum
> ต่างๆ ถือเป็นค่าคงที่จำนวนเต็มธรรมดา ทำให้เปรียบเทียบหรือใช้ใน `switch` ได้อย่างเป็น
> ธรรมชาติ แต่ก็หมายความว่า compiler จะ**ไม่บังคับ** ให้ตัวแปร enum มีค่าอยู่ในกลุ่มที่
> ประกาศไว้เท่านั้น (สามารถ assign เลขอื่นที่ไม่ตรงกับสมาชิกใดเลยได้โดยไม่มี error)
> ต้องอาศัยวินัยของโปรแกรมเมอร์เองในการใช้ค่าที่ถูกต้อง

---

## 10.7 `typedef`: สร้างชื่อ Type ของตัวเอง (Step 79)

การเขียน `struct point` หรือ `enum order_status` ซ้ำๆ ทุกครั้งที่ประกาศตัวแปรค่อนข้างยาว
`typedef` ช่วยให้เราตั้ง **ชื่อเล่นให้กับชนิดข้อมูล** เพื่อให้เขียนโค้ดกระชับและอ่านง่ายขึ้น

```c
/* typedef_demo.c */
#include <stdio.h>

/* แบบที่ 1: typedef ให้กับ struct ที่มีอยู่แล้ว */
struct point {
    int x;
    int y;
};
typedef struct point Point;

/* แบบที่ 2: ประกาศ struct พร้อม typedef ในคำสั่งเดียว (นิยมมากในโค้ดจริง) */
typedef struct {
    char name[30];
    int  age;
    double gpa;
} Student;

/* typedef กับ enum ก็ทำได้เช่นกัน */
typedef enum { MONDAY, TUESDAY, WEDNESDAY, THURSDAY, FRIDAY, SATURDAY, SUNDAY } Weekday;

int main(void) {
    Point p1 = {3, 4};              /* ไม่ต้องเขียน struct point อีกต่อไป */
    printf("p1 = (%d, %d)\n", p1.x, p1.y);

    Student s1 = {"Malee", 20, 3.75};
    printf("นักเรียน: %s อายุ %d ปี GPA %.2f\n", s1.name, s1.age, s1.gpa);

    Weekday today = WEDNESDAY;
    printf("วันนี้คือวันที่ %d ของสัปดาห์ (นับจาก 0 = จันทร์)\n", today);

    return 0;
}
```

```
p1 = (3, 4)
นักเรียน: Malee อายุ 20 ปี GPA 3.75
วันนี้คือวันที่ 2 ของสัปดาห์ (นับจาก 0 = จันทร์)
```

`typedef` ไม่ได้สร้างชนิดข้อมูลใหม่ขึ้นมาจริงๆ มันแค่ตั้ง **ชื่อเล่น (Alias)** ให้กับชนิดข้อมูล
ที่มีอยู่แล้วเท่านั้น `Point` กับ `struct point` จึงเป็นสิ่งเดียวกันทุกประการในสายตาของ
compiler — ประโยชน์หลักคือทำให้โค้ดอ่านง่ายขึ้นและพิมพ์สั้นลง โดยเฉพาะเมื่อใช้กับชนิดข้อมูล
ที่ซับซ้อน เช่น function pointer ที่เคยเห็นใน Part 9:

```c
typedef double (*BinaryOp)(double, double);   /* อ่านง่ายกว่า double (*)(double, double) มาก */
```

> **ข้อตกลงในหลักสูตรนี้**: `typedef` สำหรับ struct/enum มักตั้งชื่อขึ้นต้นด้วยตัวใหญ่ (เช่น
> `Point`, `Student`) เพื่อแยกให้เห็นชัดจากตัวแปรทั่วไปที่มักตั้งชื่อขึ้นต้นด้วยตัวเล็ก
> (เช่น `student_count`) — เป็นเพียง Convention ไม่ใช่กฎบังคับของภาษา แต่ช่วยให้อ่านโค้ด
> ทีมง่ายขึ้นมาก

---

## 10.8 ตัวอย่างจริงแบบครบวงจร: `struct Student` และ Array of Struct (Step 80)

มาประกอบทุกอย่างที่เรียนมาใน Part นี้เข้าด้วยกัน: `struct`, pointer, `enum`, และ `typedef`
ในโปรแกรมจัดการข้อมูลนักเรียนที่ใกล้เคียงกับงานจริงมากขึ้น

```c
/* student_management_demo.c
 * ตัวอย่างครบวงจร: struct + enum + typedef + pointer + array of struct
 */
#include <stdio.h>
#include <string.h>

typedef enum {
    GRADE_A,
    GRADE_B,
    GRADE_C,
    GRADE_D,
    GRADE_F
} Grade;

typedef struct {
    char  name[30];
    int   age;
    double gpa;
    Grade grade;
} Student;

Grade gpa_to_grade(double gpa) {
    if (gpa >= 3.5) return GRADE_A;
    if (gpa >= 3.0) return GRADE_B;
    if (gpa >= 2.5) return GRADE_C;
    if (gpa >= 2.0) return GRADE_D;
    return GRADE_F;
}

const char *grade_to_text(Grade g) {
    static const char *labels[] = {"A", "B", "C", "D", "F"};
    return labels[g];
}

/* รับ pointer ไปยัง array ของ Student (const เพราะแค่อ่านอย่างเดียว) เพื่อไม่ต้องคัดลอกทั้งก้อน */
void print_report(const Student *students, int count) {
    printf("%-10s %-5s %-6s %-6s\n", "ชื่อ", "อายุ", "GPA", "เกรด");
    for (int i = 0; i < count; i++) {
        printf("%-10s %-5d %-6.2f %-6s\n",
               students[i].name,
               students[i].age,
               students[i].gpa,
               grade_to_text(students[i].grade));
    }
}

/* ฟังก์ชันที่แก้ไขข้อมูลจริง ต้องรับ pointer ที่ไม่ใช่ const */
void assign_grades(Student *students, int count) {
    for (int i = 0; i < count; i++) {
        students[i].grade = gpa_to_grade(students[i].gpa);
    }
}

int main(void) {
    Student class_room[3];

    strcpy(class_room[0].name, "Somchai");
    class_room[0].age = 19;
    class_room[0].gpa = 3.8;

    strcpy(class_room[1].name, "Malee");
    class_room[1].age = 20;
    class_room[1].gpa = 2.9;

    strcpy(class_room[2].name, "Anan");
    class_room[2].age = 18;
    class_room[2].gpa = 1.8;

    assign_grades(class_room, 3);
    print_report(class_room, 3);

    return 0;
}
```

```
ชื่อ         อายุ  GPA    เกรด
Somchai    19    3.80   A
Malee      20    2.90   C
Anan       18    1.80   F
```

โปรแกรมนี้สรุปแนวคิดสำคัญของทั้ง Part 10: `typedef struct { ... } Student;` สร้างชนิด
ข้อมูลของตัวเอง, `enum Grade` แทนที่ Magic Number ด้วยชื่อที่มีความหมาย, `Student
class_room[3]` คือ array ของ struct, และฟังก์ชันทั้งสอง (`print_report`, `assign_grades`)
สาธิตหลักการ const-correctness จาก Part 9: ฟังก์ชันที่แค่ "อ่าน" ข้อมูลควรรับ `const
Student *` ส่วนฟังก์ชันที่ต้อง "แก้ไข" ข้อมูลจริงจึงรับ `Student *` แบบไม่มี `const`

---

## ข้อผิดพลาดที่พบบ่อย (Common Pitfalls)

1. **ใช้ `.` กับ pointer หรือ `->` กับ struct ตัวจริงสลับกัน** — `ptr.member` เมื่อ `ptr`
   เป็น pointer จะเป็น **Compile Error** ทันที (ไม่ใช่แค่ Warning) เพราะ `.` ใช้กับ struct
   โดยตรงเท่านั้น เช่นเดียวกัน `s->member` เมื่อ `s` เป็น struct ตัวจริง (ไม่ใช่ pointer) ก็
   จะ Compile Error เช่นกัน จำกฎ "pointer ใช้ `->`, struct ตัวจริงใช้ `.`" ให้แม่น

2. **สมมติว่า `sizeof(struct)` เท่ากับผลรวมขนาดของทุก member เป๊ะๆ** — อย่างที่เห็นในหัวข้อ
   10.4 Memory Padding ทำให้ `sizeof` ของ struct มักมากกว่าผลรวมตรงๆ เสมอ ห้ามเขียนโค้ด
   ที่สมมติขนาดของ struct เอง (เช่น คำนวณ offset ด้วยมือ) ควรใช้ `sizeof` และ `offsetof`
   จาก `<stddef.h>` เสมอเพื่อความถูกต้องข้ามแพลตฟอร์ม

3. **ใช้ `union` แล้วอ่าน member ที่ไม่ได้เพิ่งเขียนล่าสุด** — เนื่องจาก `union` ใช้พื้นที่ร่วมกัน
   การเขียนค่าให้ member หนึ่งแล้วไปอ่าน member อื่นที่ไม่เกี่ยวข้องกัน (โดยไม่มีกลไก tag
   บอกว่า member ไหนถูกต้อง) จะได้ค่าที่ไม่มีความหมาย เป็นแหล่งบั๊กที่ debug ยากมาก ถ้าต้อง
   ใช้ `union` ควรใช้คู่กับ `enum` เป็น "tag" บอกเสมอว่า member ไหนกำลังถูกใช้งานจริง

4. **ลืมว่า `enum` ใน C ไม่ได้บังคับขอบเขตค่า** — สามารถ assign ตัวเลขที่ไม่ตรงกับสมาชิกใด
   เลยให้ตัวแปร enum ได้โดย compiler ไม่ error หรือแม้แต่ warning (เช่น `enum order_status
   s = 999;` ก็คอมไพล์ผ่าน) การใช้ `switch` ที่มี `default` case เผื่อไว้เสมอจึงเป็นวินัยที่ดี
   เพื่อดักค่าที่ผิดปกติได้ (ดังตัวอย่างในฟังก์ชัน `status_to_text` ที่หัวข้อ 10.6)

5. **สับสนระหว่างการ initialize array of struct กับการใช้ `strcpy` ผิดที่** — Member ที่เป็น
   `char name[30]` ไม่สามารถกำหนดค่าด้วย `=` ตรงๆ ได้หลังประกาศแล้ว (เช่น
   `class_room[0].name = "Somchai";` เป็น **Compile Error** เพราะ array ไม่สามารถ assign
   ด้วย `=` ได้โดยตรงนอกช่วง initialization) ต้องใช้ `strcpy(class_room[0].name,
   "Somchai");` เท่านั้น (ทบทวนเรื่อง string จาก Part 7)

6. **ลืมว่า `typedef` ไม่ได้สร้างชนิดข้อมูลใหม่จริงๆ** — `typedef struct point Point;`
   ทำให้ `Point` และ `struct point` เป็นสิ่งเดียวกันทุกประการ ไม่ใช่ชนิดข้อมูลที่ปลอดภัยกว่า
   หรือแตกต่างในเชิงพฤติกรรมแต่อย่างใด มันเป็นเพียงเรื่องของความสะดวกในการอ่าน/เขียนโค้ด
   เท่านั้น ห้ามคาดหวังว่า `typedef` จะช่วยตรวจสอบชนิดข้อมูลเข้มงวดขึ้นแบบที่ C++ ทำได้

---

## แบบฝึกหัดท้ายบท

1. เขียน `struct rectangle` ที่มี member `width` และ `height` (เป็น `double`) พร้อมเขียน
   ฟังก์ชัน `double area(const struct rectangle *r)` ที่คืนค่าพื้นที่ของสี่เหลี่ยมนั้น

2. เขียนโปรแกรมพิสูจน์ Memory Padding ด้วยตัวเอง: สร้าง struct ของตัวเองที่มี `char`, `int`,
   `char`, `double` เรียงกันแบบสลับขนาด แล้วพิมพ์ `sizeof` และ `offsetof` ของแต่ละ member
   ออกมา จากนั้นลองจัดเรียง member ใหม่ให้ padding น้อยที่สุดแล้วเปรียบเทียบขนาดทั้งสองแบบ

3. เขียน `union` ชื่อ `Number` ที่มี member `as_int` (`int`), `as_float` (`float`), และ
   `as_bytes` (`char[4]`) แล้วทดลองกำหนดค่าให้ `as_int` แล้วพิมพ์ `as_bytes` ทีละ byte
   ออกมาเพื่อดู bit pattern ของเลขจำนวนเต็มนั้น

4. เขียน `enum Direction` ที่มีค่า `NORTH`, `EAST`, `SOUTH`, `WEST` แล้วเขียนฟังก์ชัน
   `Direction turn_right(Direction current)` ที่คืนค่าทิศทางถัดไปเมื่อหมุนตัวไปทางขวา 90 องศา
   (เช่น จาก `NORTH` ต้องได้ `EAST`, จาก `WEST` ต้องวนกลับไปที่ `NORTH`)

5. ใช้ `typedef` สร้างชื่อเล่นให้กับ `struct rectangle` จากข้อ 1 เป็น `Rect` แล้วเขียนฟังก์ชัน
   `Rect make_rect(double w, double h)` ที่สร้างและคืนค่า struct ใหม่กลับมา (ทดลอง return
   struct ทั้งก้อนแทนที่จะ return ผ่าน pointer ดูว่ายังคอมไพล์ผ่านได้ปกติ)

6. ต่อยอดจากตัวอย่าง `student_management_demo.c` ในหัวข้อ 10.8: เพิ่มฟังก์ชัน
   `Student *find_student_by_name(Student *students, int count, const char *name)`
   ที่คืนค่า pointer ไปยัง Student ที่มีชื่อตรงกับที่ค้นหา (ใช้ `strcmp` เปรียบเทียบ) หรือคืนค่า
   `NULL` ถ้าไม่พบ

### แนวทางเฉลยข้อ 1

```c
/* rectangle_area.c */
#include <stdio.h>

struct rectangle {
    double width;
    double height;
};

double area(const struct rectangle *r) {
    return r->width * r->height;
}

int main(void) {
    struct rectangle box = {5.0, 3.0};

    printf("สี่เหลี่ยมขนาด %.1f x %.1f มีพื้นที่ = %.2f\n",
           box.width, box.height, area(&box));

    return 0;
}
```

```
สี่เหลี่ยมขนาด 5.0 x 3.0 มีพื้นที่ = 15.00
```

สังเกตว่าฟังก์ชัน `area` รับพารามิเตอร์เป็น `const struct rectangle *r` (pointer แบบ const)
ตามหลัก const-correctness ที่เรียนมาจาก Part 9 — ฟังก์ชันนี้แค่ "คำนวณ" พื้นที่ ไม่มีเหตุผล
ต้องแก้ไขข้อมูลต้นฉบับเลย การใส่ `const` จึงเป็นการสื่อสารเจตนาที่ชัดเจนและให้ compiler
ช่วยตรวจสอบให้ด้วย

### แนวทางเฉลยข้อ 4

```c
/* direction_turn.c */
#include <stdio.h>

typedef enum { NORTH, EAST, SOUTH, WEST } Direction;

Direction turn_right(Direction current) {
    /* วนกลับไปที่ NORTH (0) หลังจาก WEST (3) ด้วยเลขคณิต modulo */
    return (Direction)((current + 1) % 4);
}

const char *direction_to_text(Direction d) {
    static const char *names[] = {"NORTH", "EAST", "SOUTH", "WEST"};
    return names[d];
}

int main(void) {
    Direction facing = NORTH;

    for (int i = 0; i < 5; i++) {
        printf("กำลังหันหน้าไปทาง %s\n", direction_to_text(facing));
        facing = turn_right(facing);
    }

    return 0;
}
```

```
กำลังหันหน้าไปทาง NORTH
กำลังหันหน้าไปทาง EAST
กำลังหันหน้าไปทาง SOUTH
กำลังหันหน้าไปทาง WEST
กำลังหันหน้าไปทาง NORTH
```

เทคนิคสำคัญในเฉลยนี้คือการใช้ `(current + 1) % 4` เพื่อ "วนกลับ" จาก `WEST` (ค่า 3) ไปเป็น
`NORTH` (ค่า 0) โดยอัตโนมัติ และต้อง cast ผลลัพธ์กลับเป็น `(Direction)` อย่างชัดเจน เพราะ
ผลลัพธ์ของ `current + 1` ในทาง C จะถูกเลื่อนชนิด (promote) เป็น `int` ธรรมดาก่อนเสมอ
ถ้าไม่ cast กลับ compiler บางตัวภายใต้ `-Wconversion` (ซึ่งเข้มกว่า `-Wall -Wextra` ที่ใช้
ในหลักสูตรนี้) จะเตือนเรื่องการแปลงชนิดข้อมูลแบบนัย

---

## สรุปท้ายบท

ใน Part นี้เราได้:

- ประกาศและใช้งาน `struct` เข้าถึง member ได้ทั้งด้วย `.` (struct ตัวจริง) และ `->` (pointer)
- ออกแบบ Nested Struct เพื่อจัดกลุ่มข้อมูลที่สัมพันธ์กันเป็นชั้นๆ
- เข้าใจ Memory Padding และ Alignment อย่างลึกซึ้ง พร้อมพิสูจน์ด้วย `sizeof`/`offsetof` จริง
  และรู้วิธีจัดเรียง member เพื่อลด padding
- แยกแยะความแตกต่างระหว่าง `struct` กับ `union` ได้ พร้อมรู้ว่าจะเลือกใช้แบบไหนเมื่อไร
- ใช้ `enum` แทน Magic Number เพื่อให้โค้ดอ่านง่ายและสื่อความหมายชัดเจนขึ้น
- ใช้ `typedef` สร้างชื่อเล่นให้กับชนิดข้อมูลของตัวเองเพื่อลดความซับซ้อนของไวยากรณ์
- ประกอบทุกแนวคิดเข้าด้วยกันในตัวอย่างโปรแกรมจัดการข้อมูลนักเรียนแบบครบวงจร

Struct, Union, Enum และ typedef ที่เรียนใน Part นี้ คือเครื่องมือสุดท้ายที่ทำให้เรา **ออกแบบ
ชนิดข้อมูลของตัวเองได้อย่างสมบูรณ์** ซึ่งเป็นทักษะที่จำเป็นอย่างยิ่งสำหรับ Data Structure
ทุกชนิดที่กำลังจะได้เรียนต่อจากนี้ (Linked List, Stack, Queue, Tree, Hash Table ใน Module B)
และยังเป็นรากฐานโดยตรงของแนวคิด `class` ใน C++ (Module D) ที่จริงๆ แล้วก็คือ `struct`
ที่ถูกขยายความสามารถให้มีฟังก์ชันของตัวเองนั่นเอง จบ Module A ที่รากฐานภาษา C ครบสมบูรณ์
แล้ว ใน **Part 11** เราจะเริ่มต้น Module B ด้วยเรื่องที่สำคัญที่สุดอย่างหนึ่งของการเขียน C
ระดับกลาง: **Dynamic Memory Allocation** — การขอและคืนหน่วยความจำระหว่างโปรแกรมทำงาน
จริงด้วย `malloc`, `calloc`, `realloc`, และ `free`

**ต่อไป:** [Part 11 — Dynamic Memory Allocation](./part-011-dynamic-memory.md)
