# Part 7: Array และ String ใน C (Step 49–56)

> Module A — รากฐานภาษา C (Foundations of C) | Part 7 จาก 125
> Step ที่ครอบคลุมใน Part นี้: Step 49–56
> Part ก่อนหน้า: [Part 6 — ฟังก์ชันในภาษา C](./part-006-functions.md) | Part ถัดไป: [Part 8 — Pointer พื้นฐาน (ตอนที่ 1)](./part-008-pointers-basics.md)

## เป้าหมายการเรียนรู้ (Learning Objectives)

หลังจบ Part นี้ ผู้เรียนจะสามารถ:

1. ประกาศ เข้าถึง และกำหนดค่าเริ่มต้นให้ Array 1 มิติได้อย่างถูกต้อง
2. อธิบายความสัมพันธ์เบื้องต้นระหว่าง Array กับ Pointer ที่เป็นรากฐานสำคัญของภาษา C
3. ส่ง Array เป็นพารามิเตอร์เข้าไปในฟังก์ชันได้ และเข้าใจว่าทำไมพฤติกรรมนี้ต่างจาก Pass by Value
   ของตัวแปรพื้นฐาน
4. อธิบายได้ว่า String ในภาษา C คืออะไร เหตุใดจึงต้องมี Null Terminator (`'\0'`) กำกับท้ายเสมอ
5. ใช้ฟังก์ชันมาตรฐานใน `<string.h>` (`strlen`, `strcpy`, `strcat`, `strcmp`, `strncpy` ฯลฯ)
   ได้อย่างถูกต้องและปลอดภัย
6. ระบุช่องโหว่ Buffer Overflow ที่เกิดจาก `strcpy`, `strcat`, `gets` ได้ พร้อมรู้วิธีป้องกัน
7. เขียนฟังก์ชันจัดการ String ขึ้นเองได้ เพื่อเข้าใจกลไกเบื้องหลังฟังก์ชันมาตรฐาน

---

## 7.1 Array 1 มิติ: ประกาศ เข้าถึง และ Initialize (Step 49)

**Array** คือโครงสร้างข้อมูลที่เก็บค่าหลายๆ ค่าที่มี **ชนิดข้อมูลเดียวกัน** ไว้ใน **หน่วยความจำ
ที่ต่อเนื่องกัน** ภายใต้ชื่อตัวแปรเดียว แทนที่จะต้องประกาศตัวแปรแยกกันทีละตัว เช่น `score1`,
`score2`, `score3`, ... เราสามารถประกาศ Array ตัวเดียวที่เก็บคะแนนทั้งหมดได้

### การประกาศ Array

```c
ชนิดข้อมูล ชื่อ_array[ขนาด];
```

```c
int scores[5];        // Array ของ int จำนวน 5 ช่อง (ยังไม่ได้กำหนดค่าเริ่มต้น)
double prices[10];    // Array ของ double จำนวน 10 ช่อง
```

### การเข้าถึงสมาชิกด้วย Index

สมาชิกแต่ละตัวใน Array เข้าถึงผ่าน **Index** (ดัชนี) ซึ่ง **เริ่มนับจาก 0 เสมอ** ไม่ใช่ 1
นี่คือกฎที่สำคัญที่สุดข้อหนึ่งของ Array ในภาษา C (และภาษาอื่นๆ ที่ได้รับอิทธิพลจาก C แทบทั้งหมด)

```c
/*
 * array_basics.c
 * ตัวอย่างพื้นฐานของการประกาศ กำหนดค่า และเข้าถึง Array
 */
#include <stdio.h>

int main(void) {
    int scores[5];    // Array 5 ช่อง: index ที่ใช้ได้คือ 0, 1, 2, 3, 4 เท่านั้น

    scores[0] = 85;
    scores[1] = 92;
    scores[2] = 78;
    scores[3] = 88;
    scores[4] = 95;

    printf("คะแนนวิชาที่ 1 (index 0): %d\n", scores[0]);
    printf("คะแนนวิชาที่ 5 (index 4): %d\n", scores[4]);

    return 0;
}
```

Array ที่มีขนาด `5` จะมี Index ตั้งแต่ `0` ถึง `4` เท่านั้น — **ไม่มี** `scores[5]` การเข้าถึง
`scores[5]` คือการเข้าถึงหน่วยความจำนอกขอบเขตของ Array (Out-of-Bounds Access) ซึ่งเป็น
Undefined Behavior ที่อันตรายมาก (จะพูดถึงรายละเอียดในหัวข้อ Common Pitfalls)

### การกำหนดค่าเริ่มต้นตอนประกาศ (Initialization)

```c
/*
 * array_init.c
 * รูปแบบต่างๆ ของการกำหนดค่าเริ่มต้นให้ Array
 */
#include <stdio.h>

int main(void) {
    int a[5] = {10, 20, 30, 40, 50};      // กำหนดค่าครบทุกช่อง
    int b[5] = {1, 2};                    // กำหนดแค่ 2 ค่าแรก ที่เหลือถูกเติม 0 อัตโนมัติ
    int c[5] = {0};                       // เทคนิคยอดนิยม: เคลียร์ทุกช่องให้เป็น 0 ทั้งหมด
    int d[] = {7, 14, 21, 28};            // ไม่ระบุขนาด compiler จะนับให้อัตโนมัติ (ได้ขนาด 4)

    printf("b = { ");
    for (int i = 0; i < 5; i++) {
        printf("%d ", b[i]);
    }
    printf("}\n");

    printf("d มีขนาด %zu ช่อง\n", sizeof(d) / sizeof(d[0]));

    /* ป้องกัน warning ตัวแปรไม่ได้ใช้งาน สำหรับตัวอย่างสาธิตนี้ */
    printf("a[0] = %d, c[0] = %d\n", a[0], c[0]);

    return 0;
}
```

ผลลัพธ์:

```
b = { 1 2 0 0 0 }
d มีขนาด 4 ช่อง
a[0] = 10, c[0] = 0
```

> **เทคนิคสำคัญ**: `int c[5] = {0};` เป็นวิธีมาตรฐานในการเคลียร์ Array ให้เป็น 0 ทั้งหมดตั้งแต่
> ตอนประกาศ กฎของ C คือถ้าใส่ค่าเริ่มต้นให้ไม่ครบทุกช่อง **ช่องที่เหลือจะถูกเติมด้วย 0 โดย
> อัตโนมัติเสมอ** (ใช้ได้กับ Array ของชนิดข้อมูลตัวเลขทุกชนิด)

### หาขนาดของ Array ด้วย `sizeof`

`sizeof(array) / sizeof(array[0])` คือสูตรมาตรฐานสำหรับหาจำนวนสมาชิกของ Array **แต่ใช้ได้
เฉพาะตอนที่ Array นั้นยังอยู่ใน scope เดิมที่ประกาศเท่านั้น** (เมื่อส่ง Array เข้าไปในฟังก์ชัน
สูตรนี้จะใช้ไม่ได้อีกต่อไป ดูรายละเอียดในหัวข้อ 7.3)

### วนอ่านค่า Array ด้วย for loop

```c
/*
 * array_sum.c
 * หาผลรวมและค่าเฉลี่ยของคะแนนที่เก็บใน Array
 */
#include <stdio.h>

int main(void) {
    int scores[5] = {85, 92, 78, 88, 95};
    int sum = 0;
    int count = (int)(sizeof(scores) / sizeof(scores[0]));

    for (int i = 0; i < count; i++) {
        sum += scores[i];
    }

    printf("ผลรวมคะแนน = %d\n", sum);
    printf("คะแนนเฉลี่ย = %.2f\n", (double)sum / count);

    return 0;
}
```

---

## 7.2 ความสัมพันธ์เบื้องต้นระหว่าง Array กับ Pointer (Step 50)

นี่คือหนึ่งในแนวคิดที่สำคัญที่สุดของภาษา C: **ชื่อของ Array สามารถทำหน้าที่เป็น Pointer
ชี้ไปยังสมาชิกตัวแรกของมันได้โดยอัตโนมัติ** ในเกือบทุกบริบทที่ใช้งาน (ยกเว้นตอนใช้กับ `sizeof`
และตอนที่นำ address ของตัว Array เองมาใช้)

```c
/*
 * array_pointer_intro.c
 * แสดงความสัมพันธ์เบื้องต้นระหว่าง array กับ pointer
 * (รายละเอียดเชิงลึกเรื่อง Pointer Arithmetic อยู่ใน Part 8-9)
 */
#include <stdio.h>

int main(void) {
    int numbers[5] = {10, 20, 30, 40, 50};

    printf("numbers[0]  = %d\n", numbers[0]);
    printf("*numbers    = %d\n", *numbers);        // *numbers เท่ากับ numbers[0] เสมอ

    printf("numbers[2]  = %d\n", numbers[2]);
    printf("*(numbers+2)= %d\n", *(numbers + 2));  // numbers[i] เทียบเท่ากับ *(numbers + i)

    printf("address ของ numbers[0]: %p\n", (void *)&numbers[0]);
    printf("ค่าของ numbers เอง:      %p\n", (void *)numbers);   // เท่ากันเสมอ!

    return 0;
}
```

จุดที่ต้องจำให้ขึ้นใจ (จะขยายความเต็มรูปแบบใน Part 8): **`numbers[i]` และ `*(numbers + i)`
มีความหมายเดียวกันทุกประการ** ในทางเทคนิค compiler จะแปลง `numbers[i]` ให้กลายเป็น
`*(numbers + i)` เบื้องหลังเสมอ นี่คือเหตุผลว่าทำไม Array และ Pointer ในภาษา C จึงผูกพันกัน
แนบแน่นมาก และเป็นสาเหตุที่ต้องเรียน Array ก่อนเรียน Pointer อย่างละเอียด

> **ข้อควรระวัง**: แม้ชื่อ Array จะทำหน้าที่เหมือน Pointer ได้ แต่ Array **ไม่ใช่** Pointer
> จริงๆ (ตัวแปร Array ไม่สามารถถูกกำหนดค่าใหม่ได้ เช่น `numbers = other_array;` จะ compile
> error ทันที) ความแตกต่างเชิงลึกนี้จะอธิบายอย่างละเอียดใน Part 8

---

## 7.3 ส่ง Array เข้าไปในฟังก์ชัน (Step 51)

เมื่อส่ง Array เป็นพารามิเตอร์ให้ฟังก์ชัน สิ่งที่ถูกส่งจริงๆ ไม่ใช่ "สำเนาทั้งหมดของ Array"
เหมือนที่ตัวแปรพื้นฐานถูกส่งแบบ Pass by Value แต่เป็น **address ของสมาชิกตัวแรก** (เหมือนที่
เห็นในหัวข้อ 7.2) ทำให้ฟังก์ชันสามารถแก้ไขข้อมูลใน Array ต้นฉบับได้โดยตรง — เป็นข้อยกเว้น
สำคัญของกฎ Pass by Value ที่เรียนใน Part 6

```c
/*
 * array_function_demo.c
 * ส่ง array เข้าไปในฟังก์ชัน และพิสูจน์ว่าฟังก์ชันแก้ไข array ต้นฉบับได้จริง
 */
#include <stdio.h>

void print_array(const int arr[], int size);   // const ป้องกันไม่ให้ฟังก์ชันนี้แก้ไขค่าโดยไม่ตั้งใจ
void double_all(int arr[], int size);
int sum_array(const int arr[], int size);

int main(void) {
    int data[5] = {1, 2, 3, 4, 5};
    int size = 5;

    printf("ก่อนแก้ไข: ");
    print_array(data, size);

    double_all(data, size);   // ส่ง array เข้าไป โดยไม่ต้องใช้ & เลย (ต่างจากตัวแปรพื้นฐาน)

    printf("หลังแก้ไข: ");
    print_array(data, size);

    printf("ผลรวม = %d\n", sum_array(data, size));

    return 0;
}

void print_array(const int arr[], int size) {
    for (int i = 0; i < size; i++) {
        printf("%d ", arr[i]);
    }
    printf("\n");
}

void double_all(int arr[], int size) {
    for (int i = 0; i < size; i++) {
        arr[i] *= 2;             // แก้ไขค่าใน array ต้นฉบับได้จริง เพราะได้รับ address มาโดยตรง
    }
}

int sum_array(const int arr[], int size) {
    int sum = 0;
    for (int i = 0; i < size; i++) {
        sum += arr[i];
    }
    return sum;
}
```

ผลลัพธ์:

```
ก่อนแก้ไข: 1 2 3 4 5
หลังแก้ไข: 2 4 6 8 10
ผลรวม = 30
```

### ทำไมต้องส่ง `size` เป็นพารามิเตอร์แยกต่างหากเสมอ

เมื่อ Array ถูกส่งเข้าไปในฟังก์ชัน มันจะ "เสื่อมสภาพ" (Decay) กลายเป็นเพียง Pointer ธรรมดา
สูตร `sizeof(arr) / sizeof(arr[0])` ที่ใช้ได้ในหัวข้อ 7.1 **จะใช้ไม่ได้อีกต่อไป** ภายใน
ฟังก์ชัน เพราะ `sizeof(arr)` ในบริบทนี้จะคืนขนาดของ Pointer (โดยทั่วไป 8 ไบต์บนระบบ 64-bit)
ไม่ใช่ขนาดของ Array ทั้งก้อน ด้วยเหตุนี้ **ต้องส่งขนาดของ Array เป็นพารามิเตอร์แยกต่างหากเสมอ**
ไม่เช่นนั้นฟังก์ชันจะไม่มีทางรู้เลยว่า Array ที่ได้รับมามีกี่สมาชิก

### ทำไมต้องใช้ `const` กับพารามิเตอร์ Array ที่ไม่ต้องการแก้ไข

`const int arr[]` บอก compiler (และคนอ่านโค้ด) ว่า "ฟังก์ชันนี้จะไม่แก้ไขข้อมูลใน Array นี้
เลย" ถ้าเผลอเขียนโค้ดที่พยายามแก้ไขค่าใน `arr` ภายในฟังก์ชันที่ประกาศเป็น `const` compiler
จะ error ทันที ช่วยป้องกันบั๊กที่เกิดจากการแก้ไข Array โดยไม่ตั้งใจได้ตั้งแต่ตอนคอมไพล์

---

## 7.4 String ในภาษา C คือ Array of char ที่ลงท้ายด้วย `'\0'` (Step 52)

ภาษา C **ไม่มีชนิดข้อมูล String แยกต่างหาก** เหมือนภาษาสมัยใหม่อื่นๆ (เช่น Python, Java)
String ในภาษา C คือเพียง **Array ของ `char`** ที่มีกฎพิเศษหนึ่งข้อ: **ต้องมี Null Terminator
(`'\0'`) กำกับอยู่ที่ตำแหน่งท้ายสุดของข้อความเสมอ** เพื่อบอกฟังก์ชันต่างๆ ว่า "ข้อความจบตรงนี้"

```c
/*
 * string_basics.c
 * ตัวอย่างพื้นฐานของ string ในภาษา C
 */
#include <stdio.h>

int main(void) {
    char greeting1[6] = {'H', 'e', 'l', 'l', 'o', '\0'};   // เขียนแบบเต็ม ระบุ '\0' เอง
    char greeting2[6] = "Hello";                           // เขียนแบบ string literal (สะดวกกว่ามาก)
    char greeting3[] = "Hello";                             // ไม่ระบุขนาด compiler นับให้ (ได้ 6: H-e-l-l-o-\0)

    printf("greeting1 = %s\n", greeting1);
    printf("greeting2 = %s\n", greeting2);
    printf("greeting3 = %s (ขนาด %zu ไบต์)\n", greeting3, sizeof(greeting3));

    return 0;
}
```

ผลลัพธ์:

```
greeting1 = Hello
greeting2 = Hello
greeting3 = Hello (ขนาด 6 ไบต์)
```

**จุดสำคัญที่สุด**: `"Hello"` มีตัวอักษรที่มองเห็นได้ 5 ตัว แต่ต้องใช้พื้นที่เก็บ **6 ไบต์**
เสมอ เพราะไบต์สุดท้ายต้องเป็น `'\0'` (Null Character ค่า ASCII เท่ากับ 0) เพื่อบอกจุดสิ้นสุด
ของข้อความ **ถ้าลืมเผื่อพื้นที่สำหรับ `'\0'` นี้ คือจุดเริ่มต้นของบั๊ก Buffer Overflow ที่
อันตรายที่สุดในภาษา C** (จะพูดถึงอย่างละเอียดในหัวข้อ 7.6)

### ทำไม printf ถึงรู้ว่า string ยาวแค่ไหน (`%s`)

```c
char message[20] = "Hi";
```

Array `message` มีขนาด 20 ไบต์ แต่ `"Hi"` ใช้ไปแค่ 3 ไบต์ (`'H'`, `'i'`, `'\0'`) ไบต์ที่เหลือ
อีก 17 ไบต์ยังเป็นค่าขยะ (Garbage Value) ที่ไม่ได้ถูกกำหนดค่า แต่เมื่อสั่ง `printf("%s", message)`
ฟังก์ชันจะพิมพ์ตัวอักษรไปเรื่อยๆ **จนกว่าจะเจอ `'\0'` เป็นตัวแรก** แล้วหยุดทันที ไม่สนใจว่า
Array ทั้งก้อนมีขนาดเท่าไร นี่คือเหตุผลที่ `'\0'` สำคัญมาก — ถ้าไม่มีมันเลย (หรือถูกเขียนทับ
โดยไม่ตั้งใจ) `printf` จะพิมพ์ค่าขยะในหน่วยความจำต่อไปเรื่อยๆ จนกว่าจะบังเอิญเจอไบต์ที่มีค่า 0
ที่ไหนสักแห่ง ซึ่งอาจไกลเกินขอบเขตของโปรแกรมจนทำให้ Crash ได้

### หา Null Terminator ด้วยตัวเอง

```c
/*
 * find_null_terminator.c
 * แสดงตำแหน่งของ '\0' ใน string ด้วยการวน loop เอง
 */
#include <stdio.h>

int main(void) {
    char word[10] = "Code";

    int i = 0;
    while (word[i] != '\0') {
        printf("word[%d] = '%c'\n", i, word[i]);
        i++;
    }
    printf("word[%d] = '\\0' (จุดสิ้นสุดของ string)\n", i);

    return 0;
}
```

---

## 7.5 ฟังก์ชันมาตรฐานใน `<string.h>` (Step 53)

ภาษา C มีไลบรารีมาตรฐาน `<string.h>` ที่รวมฟังก์ชันสำหรับจัดการ string ไว้ให้ใช้งานโดยไม่ต้อง
เขียนเอง ฟังก์ชันเหล่านี้เป็นเครื่องมือพื้นฐานที่ใช้แทบทุกวันในการเขียนโปรแกรม C

| ฟังก์ชัน | หน้าที่ | ตัวอย่าง |
|---|---|---|
| `strlen(s)` | หาความยาวของ string (ไม่นับ `'\0'`) | `strlen("Hello")` → `5` |
| `strcpy(dst, src)` | คัดลอก string จาก `src` ไปยัง `dst` | `strcpy(buf, "Hi")` |
| `strncpy(dst, src, n)` | คัดลอกแบบจำกัดจำนวนไบต์ (ปลอดภัยกว่า `strcpy`) | `strncpy(buf, "Hi", 10)` |
| `strcat(dst, src)` | ต่อ string `src` เข้าท้าย `dst` | `strcat(buf, " World")` |
| `strncat(dst, src, n)` | ต่อ string แบบจำกัดจำนวนไบต์ | `strncat(buf, s, 10)` |
| `strcmp(s1, s2)` | เปรียบเทียบ string 2 ตัว (คืน 0 ถ้าเหมือนกัน) | `strcmp("cat", "dog")` → ค่าติดลบ |
| `strncmp(s1, s2, n)` | เปรียบเทียบ string แบบจำกัดจำนวนไบต์แรก | `strncmp("cat", "car", 2)` → `0` |
| `strchr(s, c)` | หาตำแหน่งแรกที่พบตัวอักษร `c` ใน `s` | `strchr("Hello", 'l')` |
| `strstr(s1, s2)` | หาตำแหน่งแรกที่พบ string ย่อย `s2` ใน `s1` | `strstr("Hello World", "World")` |

```c
/*
 * string_h_demo.c
 * ตัวอย่างการใช้งานฟังก์ชันหลักใน <string.h>
 */
#include <stdio.h>
#include <string.h>

int main(void) {
    char name[50] = "Somchai";
    char surname[] = "Jaidee";
    char full_name[50];

    /* strlen: หาความยาว */
    printf("ความยาวของ '%s' คือ %zu ตัวอักษร\n", name, strlen(name));

    /* strcpy: คัดลอก */
    strcpy(full_name, name);
    printf("full_name หลัง strcpy = %s\n", full_name);

    /* strcat: ต่อท้าย */
    strcat(full_name, " ");     // เว้นวรรคก่อน
    strcat(full_name, surname);
    printf("full_name หลัง strcat = %s\n", full_name);

    /* strcmp: เปรียบเทียบ */
    if (strcmp(name, "Somchai") == 0) {
        printf("ชื่อตรงกับ 'Somchai'\n");
    }

    int cmp_result = strcmp("apple", "banana");
    printf("strcmp(\"apple\", \"banana\") = %d (ค่าติดลบ เพราะ 'a' < 'b' ทาง ASCII)\n", cmp_result);

    /* strncpy: คัดลอกแบบจำกัดความยาว (ปลอดภัยกว่า) */
    char short_buf[4];
    strncpy(short_buf, "Hello", sizeof(short_buf) - 1);
    short_buf[sizeof(short_buf) - 1] = '\0';   // ต้องปิดท้ายด้วย '\0' เองเสมอเมื่อใช้ strncpy
    printf("short_buf (ตัด 'Hello' ให้เหลือ 3 ตัว) = %s\n", short_buf);

    return 0;
}
```

ผลลัพธ์:

```
ความยาวของ 'Somchai' คือ 7 ตัวอักษร
full_name หลัง strcpy = Somchai
full_name หลัง strcat = Somchai Jaidee
ชื่อตรงกับ 'Somchai'
strcmp("apple", "banana") = -1 (ค่าติดลบ เพราะ 'a' < 'b' ทาง ASCII)
short_buf (ตัด 'Hello' ให้เหลือ 3 ตัว) = Hel
```

> **หมายเหตุเรื่อง `strcmp`**: ค่าที่คืนกลับมาไม่ได้การันตีว่าจะเป็น `-1`, `0`, `1` เป๊ะๆ เสมอไป
> มาตรฐานภาษา C รับประกันแค่ **เครื่องหมาย** ของค่าที่คืนกลับ (ลบ, ศูนย์, บวก) เท่านั้น
> ไม่ควรเขียนโค้ดที่ตรวจสอบค่าตรงๆ เช่น `if (strcmp(a, b) == -1)` ควรใช้ `if (strcmp(a, b) < 0)`
> แทนเสมอเพื่อความถูกต้องตามมาตรฐาน (ในตัวอย่างข้างบน glibc บนเครื่องนี้คืนค่า `-1` พอดี
> แต่ compiler/library ตัวอื่นอาจคืนค่าติดลบตัวอื่นที่ไม่ใช่ `-1` ก็ได้)

---

## 7.6 ช่องโหว่ Buffer Overflow: `strcpy`, `strcat`, `gets` (Step 54)

นี่คือหัวข้อที่สำคัญที่สุดของ Part นี้ในแง่ความปลอดภัย **Buffer Overflow** คือบั๊กที่เกิดขึ้น
เมื่อโปรแกรมเขียนข้อมูลเกินขอบเขตของพื้นที่หน่วยความจำที่จองไว้ ฟังก์ชันจัดการ string แบบ
ดั้งเดิมหลายตัวใน C **ไม่มีการตรวจสอบขนาดปลายทางเลย** ทำให้เป็นต้นเหตุของช่องโหว่ความปลอดภัย
ที่ร้ายแรงที่สุดในประวัติศาสตร์ซอฟต์แวร์ (รวมถึง Worm และ Malware ชื่อดังหลายตัว)

### ตัวอย่างช่องโหว่ที่ 1: `strcpy` ไม่ตรวจสอบขนาดปลายทาง

```c
/* buffer_overflow_strcpy.c — ตัวอย่างอันตราย ห้ามใช้รูปแบบนี้ในโค้ดจริง! */
#include <stdio.h>
#include <string.h>

int main(void) {
    char small_buffer[8];                         // จองพื้นที่แค่ 8 ไบต์
    strcpy(small_buffer, "This string is way too long!");  // ยาวกว่า 8 ไบต์มาก!

    printf("%s\n", small_buffer);
    return 0;
}
```

`"This string is way too long!"` ยาว 29 ตัวอักษร (บวก `'\0'` อีก 1 ไบต์ = 30 ไบต์) แต่
`small_buffer` จองไว้แค่ 8 ไบต์ `strcpy` **จะเขียนข้อมูลทับหน่วยความจำที่อยู่ถัดจาก
`small_buffer` ต่อไปเรื่อยๆ โดยไม่สนใจขอบเขตเลย** ซึ่งอาจเป็นตัวแปรอื่น, ค่า Return Address
ของฟังก์ชัน, หรือข้อมูลสำคัญอื่นๆ ในหน่วยความจำ ผลลัพธ์อาจเป็นได้ตั้งแต่โปรแกรม Crash
(`Segmentation fault`) ไปจนถึง **การถูกโจมตีเพื่อรันโค้ดอันตรายจากภายนอก (Code Injection)**
ซึ่งเป็นเทคนิคการโจมตีที่แฮกเกอร์ใช้จริงมานานหลายทศวรรษ

โชคดีที่ในกรณีนี้ compiler สมัยใหม่ฉลาดพอที่จะตรวจจับการ overflow แบบที่ **ขนาดรู้แน่ชัด
ตั้งแต่ตอน compile** ได้ล่วงหน้า:

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 buffer_overflow_strcpy.c -o buffer_overflow_strcpy
```

```
warning: '__builtin_memcpy' writing 29 bytes into a region of size 8 overflows the
destination [-Wstringop-overflow=]
note: destination object 'small_buffer' of size 8
```

แต่ต้องเข้าใจให้ชัดว่า **นี่เป็นกรณีพิเศษ** ที่ compiler มองเห็นทั้งขนาดปลายทางและความยาว
ของข้อความต้นทางตั้งแต่ตอน compile (เพราะเป็น string literal ตรงๆ) ในโค้ดจริงส่วนใหญ่
ขนาดของ `src` มักมาจากตัวแปรที่รับค่าตอน runtime (เช่นข้อมูลจากผู้ใช้หรือจากไฟล์)
ซึ่ง **compiler ไม่มีทางรู้ล่วงหน้าได้เลยว่าข้อมูลจะยาวแค่ไหน** จึงไม่มี Warning เตือนให้
เห็นเลย และบั๊กจะปรากฏเฉพาะตอนรันจริงเท่านั้น — นี่คือเหตุผลที่ห้ามใช้ `strcpy`/`strcat`
กับข้อมูลที่ไม่ทราบขนาดล่วงหน้าโดยเด็ดขาด ไม่ว่า compiler จะเตือนหรือไม่ก็ตาม

### ตัวอย่างช่องโหว่ที่ 2: `gets()` — อันตรายจนถูกถอดออกจากมาตรฐานภาษาแล้ว

```c
/* ห้ามใช้เด็ดขาด — gets() ถูกถอดออกจากมาตรฐาน C11 แล้ว เพราะอันตรายเกินไป */
char buffer[10];
gets(buffer);   // ไม่มีทางจำกัดจำนวนตัวอักษรที่อ่านได้เลย ไม่ว่ากรณีใดๆ ทั้งสิ้น
```

`gets()` เป็นฟังก์ชันที่อ่านข้อความจาก `stdin` **โดยไม่มีวิธีจำกัดความยาวได้เลยแม้แต่น้อย**
ไม่ว่า `buffer` จะมีขนาดเท่าไร ถ้าผู้ใช้พิมพ์ข้อความยาวกว่านั้น โปรแกรมจะเกิด Buffer Overflow
ทันที ด้วยความอันตรายที่ไม่มีทางป้องกันได้เลยนี้เอง **มาตรฐาน C11 จึงถอด `gets()` ออกจาก
ประกาศใน `<stdio.h>` ไปอย่างเป็นทางการ**

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 gets_demo.c -o gets_demo
```

```
warning: implicit declaration of function 'gets'; did you mean 'fgets'? [-Wimplicit-function-declaration]
/usr/bin/ld: gets_demo.o: in function `main':
gets_demo.c:(.text+0x28): warning: the `gets' function is dangerous and should not be used.
```

จะเห็นว่าทั้ง compiler และ linker ต่างก็เตือนเสียงดังว่า `gets` เป็นฟังก์ชันอันตราย เพราะมันถูก
ถอดออกจาก header `<stdio.h>` แล้ว (ทำให้ compiler ไม่รู้จัก prototype ของมัน) แต่ตัว library
`glibc` บางเวอร์ชันยังคงเก็บฟังก์ชันนี้ไว้เพื่อความเข้ากันได้กับโค้ดเก่ามากๆ ทำให้โปรแกรมนี้
ยังคอมไพล์และลิงก์ผ่านได้ (พร้อม Warning เสียงดังทั้งสองชั้น) — **ห้ามใช้ `gets()` เด็ดขาด
ไม่ว่ากรณีใดก็ตาม** แม้ว่า toolchain บางชุดจะยอมให้ผ่านไปได้ก็ตาม

### วิธีป้องกันที่ถูกต้อง: ใช้ฟังก์ชันที่จำกัดขนาดเสมอ

| ฟังก์ชันอันตราย | ฟังก์ชันทดแทนที่ปลอดภัยกว่า | เหตุผล |
|---|---|---|
| `gets(buf)` | `fgets(buf, sizeof(buf), stdin)` | จำกัดจำนวนไบต์ที่อ่านได้ ป้องกัน overflow เสมอ |
| `strcpy(dst, src)` | `strncpy(dst, src, sizeof(dst) - 1)` + ปิดท้าย `'\0'` เอง | จำกัดจำนวนไบต์ที่คัดลอก |
| `strcat(dst, src)` | `strncat(dst, src, remaining_space)` | จำกัดจำนวนไบต์ที่ต่อท้าย |
| `sprintf(buf, fmt, ...)` | `snprintf(buf, sizeof(buf), fmt, ...)` | จำกัดขนาด buffer ที่เขียนได้ |

```c
/*
 * safe_input.c
 * ตัวอย่างการรับ input จากผู้ใช้อย่างปลอดภัยด้วย fgets แทน gets
 */
#include <stdio.h>
#include <string.h>

int main(void) {
    char name[20];

    printf("กรุณาป้อนชื่อของคุณ (ไม่เกิน 19 ตัวอักษร): ");
    if (fgets(name, sizeof(name), stdin) == NULL) {
        printf("ไม่สามารถอ่านข้อมูลได้\n");
        return 1;
    }

    /* fgets จะเก็บ '\n' ติดมาด้วยถ้ามีที่ว่างพอ ต้องตัดทิ้งเอง */
    size_t len = strlen(name);
    if (len > 0 && name[len - 1] == '\n') {
        name[len - 1] = '\0';
    }

    printf("สวัสดี, %s!\n", name);

    return 0;
}
```

`fgets(name, sizeof(name), stdin)` จะอ่านข้อมูลจาก `stdin` **ไม่เกินขนาดของ `name` ที่ระบุไว้
เด็ดขาด** (รวม `'\0'` ด้วย) ต่อให้ผู้ใช้พิมพ์ข้อความยาวแค่ไหนก็ตาม โปรแกรมจะไม่มีทาง Buffer
Overflow ได้เลย นี่คือรูปแบบมาตรฐานที่ควรใช้แทน `gets()` ในทุกกรณี

### ตัวอย่างช่องโหว่ที่ 3: `strncpy` ก็ยังมีข้อควรระวังที่มือใหม่มักพลาด

แม้ `strncpy` จะปลอดภัยกว่า `strcpy` แต่ก็มีพฤติกรรมที่แปลกและเป็นกับดักได้เช่นกัน:
**ถ้า `src` ยาวเท่ากับหรือมากกว่า `n` ที่ระบุไว้ `strncpy` จะ "ไม่เติม" `'\0'` ให้เลย**

```c
/* strncpy_trap.c — ตัวอย่างกับดักของ strncpy */
#include <stdio.h>
#include <string.h>

int main(void) {
    char dst[5];
    strncpy(dst, "HelloWorld", 5);   // คัดลอกแค่ 5 ตัวอักษรแรก "Hello" แต่ไม่มี '\0' เลย!

    printf("%s\n", dst);             // Undefined Behavior: printf จะอ่านเลยขอบเขตของ dst ไปเรื่อยๆ

    return 0;
}
```

เพราะ `dst` เต็มไปด้วย `'H','e','l','l','o'` พอดี 5 ไบต์ โดยไม่มีที่ว่างเหลือให้ `'\0'` เลย
เมื่อ `printf("%s", dst)` พยายามหา `'\0'` มันจะอ่านหน่วยความจำเลยขอบเขตของ `dst` ต่อไปเรื่อยๆ
จนกว่าจะบังเอิญเจอไบต์ค่า 0 ที่ไหนสักแห่ง **วิธีแก้ที่ถูกต้องคือต้องเผื่อขนาดปลายทางให้มากกว่า
ความยาวสูงสุดที่ต้องการคัดลอกอย่างน้อย 1 ไบต์เสมอ และปิดท้ายด้วย `'\0'` ด้วยตัวเอง** ดังที่
แสดงไว้แล้วในตัวอย่างหัวข้อ 7.5 (`short_buf[sizeof(short_buf) - 1] = '\0';`)

> **กฎทองของการจัดการ String ในภาษา C**: ทุกครั้งที่คัดลอก ต่อ หรือเขียนข้อมูลลงใน buffer
> ต้องถามตัวเองเสมอว่า **"buffer ปลายทางมีพื้นที่มากพอสำหรับข้อมูลบวกกับ `'\0'` จริงหรือไม่"**
> ห้ามสมมติว่าข้อมูล input จะมีความยาวไม่เกินที่คาดไว้เด็ดขาด โดยเฉพาะถ้าข้อมูลนั้นมาจาก
> ผู้ใช้หรือแหล่งภายนอกที่ควบคุมไม่ได้

---

## 7.7 เขียนฟังก์ชันจัดการ String เอง (Step 55)

การเขียนฟังก์ชันจัดการ string ขึ้นเองช่วยให้เข้าใจกลไกเบื้องหลังฟังก์ชันมาตรฐานอย่างลึกซึ้ง
และเป็นแบบฝึกหัดคลาสสิกที่ใช้ทดสอบความเข้าใจเรื่อง Array, Pointer, และ Null Terminator
พร้อมกันในคราวเดียว

### เขียน `my_strlen` เอง

```c
/*
 * my_string_functions.c
 * เขียนฟังก์ชันจัดการ string พื้นฐานขึ้นเอง เพื่อเข้าใจกลไกเบื้องหลัง <string.h>
 */
#include <stdio.h>

size_t my_strlen(const char *s);
void my_strcpy(char *dst, const char *src, size_t dst_size);
int my_strcmp(const char *s1, const char *s2);
void my_reverse(char *s);

int main(void) {
    const char *text = "Programming";

    printf("my_strlen(\"%s\") = %zu\n", text, my_strlen(text));

    char copy[20];
    my_strcpy(copy, text, sizeof(copy));
    printf("my_strcpy ได้ผลลัพธ์ = %s\n", copy);

    printf("my_strcmp(\"apple\", \"apple\") = %d\n", my_strcmp("apple", "apple"));
    printf("my_strcmp(\"apple\", \"banana\") = %d\n", my_strcmp("apple", "banana"));

    char word[] = "Hello";
    my_reverse(word);
    printf("my_reverse(\"Hello\") = %s\n", word);

    return 0;
}

/* หาความยาวของ string โดยนับตัวอักษรจนกว่าจะเจอ '\0' */
size_t my_strlen(const char *s) {
    size_t length = 0;
    while (s[length] != '\0') {
        length++;
    }
    return length;
}

/* คัดลอก string อย่างปลอดภัย โดยจำกัดไม่ให้เกินขนาดปลายทาง (dst_size) */
void my_strcpy(char *dst, const char *src, size_t dst_size) {
    size_t i = 0;
    while (i < dst_size - 1 && src[i] != '\0') {
        dst[i] = src[i];
        i++;
    }
    dst[i] = '\0';   // ปิดท้ายด้วย '\0' เสมอ ไม่ว่า src จะยาวแค่ไหน
}

/* เปรียบเทียบ string ทีละตัวอักษร คืนค่าตามส่วนต่างของ ASCII code ตัวแรกที่ไม่ตรงกัน */
int my_strcmp(const char *s1, const char *s2) {
    size_t i = 0;
    while (s1[i] != '\0' && s2[i] != '\0') {
        if (s1[i] != s2[i]) {
            return (unsigned char)s1[i] - (unsigned char)s2[i];
        }
        i++;
    }
    return (unsigned char)s1[i] - (unsigned char)s2[i];
}

/* กลับด้าน string ในหน่วยความจำเดิม (in-place) โดยสลับตัวอักษรหัว-ท้ายเข้าหากัน */
void my_reverse(char *s) {
    size_t length = my_strlen(s);
    for (size_t i = 0; i < length / 2; i++) {
        char temp = s[i];
        s[i] = s[length - 1 - i];
        s[length - 1 - i] = temp;
    }
}
```

ผลลัพธ์:

```
my_strlen("Programming") = 11
my_strcpy ได้ผลลัพธ์ = Programming
my_strcmp("apple", "apple") = 0
my_strcmp("apple", "banana") = -1
my_reverse("Hello") = olleH
```

### สิ่งที่เรียนรู้จากการเขียนฟังก์ชันเหล่านี้เอง

- **`my_strlen`** แสดงให้เห็นชัดเจนว่า `strlen` ของจริงก็ทำงานแบบเดียวกัน คือวน loop นับ
  ตัวอักษรไปเรื่อยๆ จนกว่าจะเจอ `'\0'` — ความยาวของ string จึงเป็นการคำนวณแบบ O(n) เสมอ
  ไม่ใช่การอ่านค่าที่เก็บไว้แล้วแบบทันที (ต่างจากภาษาสมัยใหม่บางภาษาที่เก็บความยาวไว้ล่วงหน้า)
- **`my_strcpy`** แสดงเทคนิคการป้องกัน Buffer Overflow ด้วยตัวเอง โดยรับพารามิเตอร์
  `dst_size` เพิ่มเข้ามา (ปรัชญาเดียวกับ `strncpy` แต่รับประกันการปิดท้าย `'\0'` เสมอ
  ซึ่งเป็นข้อดีกว่า `strncpy` มาตรฐานที่มีกับดักตามหัวข้อ 7.6)
- **`my_strcmp`** แสดงให้เห็นว่าการเปรียบเทียบ string แท้จริงแล้วคือการเปรียบเทียบค่า ASCII
  ของตัวอักษรทีละตัว ไม่ใช่การเปรียบเทียบ "ความหมาย" ใดๆ ทั้งสิ้น
- **`my_reverse`** แสดงเทคนิค **In-place Algorithm** (แก้ไขข้อมูลในหน่วยความจำเดิมโดยตรง
  ไม่สร้าง Array ใหม่) ซึ่งเป็นเทคนิคสำคัญด้านประสิทธิภาพที่จะเจอบ่อยมากในวิชา Algorithm

---

## 7.8 ตัวอย่างจริง: โปรแกรมประมวลผลข้อความจากผู้ใช้ (Step 56)

มาผสมทุกอย่างที่เรียนมาใน Part นี้เข้าด้วยกัน: รับข้อความจากผู้ใช้อย่างปลอดภัย, วิเคราะห์ด้วย
ฟังก์ชันจาก `<string.h>`, และประมวลผลด้วย Array

```c
/*
 * text_analyzer.c
 * โปรแกรมวิเคราะห์ข้อความ: นับตัวอักษร คำ สระ และตรวจสอบว่าเป็น Palindrome หรือไม่
 */
#include <stdio.h>
#include <string.h>
#include <ctype.h>

int count_words(const char *text);
int count_vowels(const char *text);
int is_palindrome(const char *text);

int main(void) {
    char input[100];

    printf("กรุณาป้อนประโยคที่ต้องการวิเคราะห์: ");
    if (fgets(input, sizeof(input), stdin) == NULL) {
        printf("ไม่สามารถอ่านข้อมูลได้\n");
        return 1;
    }

    /* ตัด '\n' ที่ fgets เก็บติดมาด้วยออก */
    size_t len = strlen(input);
    if (len > 0 && input[len - 1] == '\n') {
        input[len - 1] = '\0';
    }

    printf("\n=== ผลการวิเคราะห์ ===\n");
    printf("ข้อความ           : \"%s\"\n", input);
    printf("จำนวนตัวอักษร      : %zu ตัว\n", strlen(input));
    printf("จำนวนคำ           : %d คำ\n", count_words(input));
    printf("จำนวนสระ (a,e,i,o,u): %d ตัว\n", count_vowels(input));

    if (is_palindrome(input)) {
        printf("เป็น Palindrome    : ใช่ (อ่านย้อนกลับเหมือนเดิม)\n");
    } else {
        printf("เป็น Palindrome    : ไม่ใช่\n");
    }

    return 0;
}

/* นับจำนวนคำ โดยนับช่วงของตัวอักษรที่ไม่ใช่ช่องว่างซึ่งต่อเนื่องกัน */
int count_words(const char *text) {
    int words = 0;
    int inside_word = 0;

    for (int i = 0; text[i] != '\0'; i++) {
        if (!isspace((unsigned char)text[i])) {
            if (!inside_word) {
                words++;
                inside_word = 1;
            }
        } else {
            inside_word = 0;
        }
    }

    return words;
}

/* นับจำนวนสระในข้อความ (ไม่สนใจตัวพิมพ์เล็ก-ใหญ่) */
int count_vowels(const char *text) {
    int count = 0;

    for (int i = 0; text[i] != '\0'; i++) {
        char c = (char)tolower((unsigned char)text[i]);
        if (c == 'a' || c == 'e' || c == 'i' || c == 'o' || c == 'u') {
            count++;
        }
    }

    return count;
}

/* ตรวจสอบว่าข้อความ (เฉพาะตัวอักษร ไม่สนใจช่องว่างและตัวพิมพ์) เป็น Palindrome หรือไม่ */
int is_palindrome(const char *text) {
    char cleaned[100];
    int j = 0;

    /* คัดกรองเฉพาะตัวอักษร/ตัวเลข และแปลงเป็นตัวพิมพ์เล็กทั้งหมด */
    for (int i = 0; text[i] != '\0' && j < (int)sizeof(cleaned) - 1; i++) {
        if (isalnum((unsigned char)text[i])) {
            cleaned[j] = (char)tolower((unsigned char)text[i]);
            j++;
        }
    }
    cleaned[j] = '\0';

    int left = 0;
    int right = j - 1;

    while (left < right) {
        if (cleaned[left] != cleaned[right]) {
            return 0;
        }
        left++;
        right--;
    }

    return 1;
}
```

ตัวอย่างการรัน:

```
กรุณาป้อนประโยคที่ต้องการวิเคราะห์: A man a plan a canal Panama

=== ผลการวิเคราะห์ ===
ข้อความ           : "A man a plan a canal Panama"
จำนวนตัวอักษร      : 28 ตัว
จำนวนคำ           : 7 คำ
จำนวนสระ (a,e,i,o,u): 11 ตัว
เป็น Palindrome    : ใช่ (อ่านย้อนกลับเหมือนเดิม)
```

โปรแกรมนี้รวมทักษะทั้งหมดของ Part 7 เข้าด้วยกัน: การรับ input อย่างปลอดภัยด้วย `fgets`,
การใช้ `<string.h>` (`strlen`) ร่วมกับ `<ctype.h>` (`isspace`, `isalnum`, `tolower`),
การเขียนฟังก์ชันประมวลผล string เอง, และการใช้ Array (`cleaned[100]`) เป็นพื้นที่ทำงานชั่วคราว

---

## ข้อผิดพลาดที่พบบ่อย (Common Pitfalls)

1. **Off-by-one เมื่อจองพื้นที่ให้ String ไม่พอสำหรับ `'\0'`** — ลืมนึกว่า string ต้องการ
   พื้นที่ `ความยาวข้อความ + 1` เสมอ

   ```c
   char name[5];
   strcpy(name, "Alice");   // "Alice" มี 5 ตัวอักษร + '\0' = ต้องการ 6 ไบต์ แต่จองไว้แค่ 5!
   ```

   นี่คือ Buffer Overflow ทันที แม้จำนวนตัวอักษรจะดู "พอดี" กับขนาด Array ก็ตาม เพราะลืม
   เผื่อพื้นที่ให้ `'\0'`

2. **เข้าถึง Index เกินขอบเขตของ Array (Out-of-Bounds Access)** — ภาษา C **ไม่มีการตรวจสอบ
   ขอบเขตของ Array ให้อัตโนมัติ** ต่างจากภาษาสมัยใหม่หลายภาษา

   ```c
   int arr[5] = {1, 2, 3, 4, 5};
   printf("%d\n", arr[5]);   // บั๊ก: index ที่ใช้ได้คือ 0-4 เท่านั้น arr[5] คือ Undefined Behavior
   ```

   โค้ดนี้อาจคอมไพล์ผ่านได้โดยไม่มี Warning เลย (โดยเฉพาะถ้า index มาจากตัวแปรที่คำนวณ
   ตอน runtime) แต่การรันจะอ่านค่าขยะจากหน่วยความจำที่ไม่ใช่ของ Array นี้ อาจได้ค่าประหลาด
   หรือโปรแกรม Crash แล้วแต่กรณี

3. **ใช้ `gets()` หรือ `strcpy`/`strcat` กับข้อมูลจากผู้ใช้โดยไม่จำกัดขนาด** — ดังที่อธิบาย
   อย่างละเอียดในหัวข้อ 7.6 ควรใช้ `fgets`, `strncpy`, `strncat`, `snprintf` แทนเสมอเมื่อ
   ขนาดข้อมูลไม่แน่นอนหรือมาจากภายนอก

4. **เปรียบเทียบ String ด้วย `==` แทน `strcmp`** — เป็นบั๊กที่พบบ่อยมากในหมู่มือใหม่ที่ย้าย
   มาจากภาษาอื่น

   ```c
   char s1[] = "hello";
   char s2[] = "hello";

   if (s1 == s2) {              // บั๊ก: เปรียบเทียบ address ของ array ไม่ใช่เนื้อหาข้างใน!
       printf("เหมือนกัน\n");    // บรรทัดนี้จะไม่ทำงาน แม้เนื้อหาจะเหมือนกันทุกตัวอักษร
   }

   if (strcmp(s1, s2) == 0) {   // ถูกต้อง: เปรียบเทียบเนื้อหาข้างในทีละตัวอักษร
       printf("เหมือนกันจริง\n");
   }
   ```

   `s1` และ `s2` เป็น Array คนละก้อนที่อยู่คนละตำแหน่งในหน่วยความจำ แม้เนื้อหาข้างในจะ
   เหมือนกันทุกตัวอักษร `s1 == s2` จะเปรียบเทียบ **address** ของทั้งสอง (เพราะชื่อ Array
   เสื่อมสภาพเป็น Pointer ตามหัวข้อ 7.2) ซึ่งต่างกันเสมอ ต้องใช้ `strcmp` เพื่อเปรียบเทียบ
   **เนื้อหา** ข้างในจริงๆ เท่านั้น

5. **ลืมว่า `strncpy` อาจไม่เติม `'\0'` ให้** — ดังตัวอย่างในหัวข้อ 7.6 ต้องปิดท้ายด้วย
   `dst[n-1] = '\0';` ด้วยตัวเองเสมอหลังเรียก `strncpy`

6. **ส่ง Array ไปฟังก์ชันแล้วใช้ `sizeof` หาขนาดข้างในฟังก์ชันนั้น** — ใช้ไม่ได้ผลตามที่คาดหวัง
   เพราะ Array เสื่อมสภาพเป็น Pointer แล้ว (ดูรายละเอียดในหัวข้อ 7.3)

   ```c
   void print_size(int arr[]) {
       printf("%zu\n", sizeof(arr));   // ได้ขนาดของ pointer (มักเป็น 8) ไม่ใช่ขนาดของ array จริง!
   }
   ```

---

## แบบฝึกหัดท้ายบท

1. เขียนโปรแกรมที่ประกาศ Array ของ `int` ขนาด 10 ช่อง รับค่าจากผู้ใช้ทีละตัวจนครบ แล้วหา
   ค่ามากที่สุดและค่าน้อยที่สุดใน Array นั้น

2. เขียนฟังก์ชัน `int count_char(const char *s, char target)` ที่นับว่าตัวอักษร `target`
   ปรากฏใน string `s` กี่ครั้ง แล้วทดสอบด้วยคำว่า `"mississippi"` กับตัวอักษร `'s'`
   (คำตอบที่ถูกต้องคือ 4)

3. เขียนโปรแกรมที่รับชื่อและนามสกุลจากผู้ใช้แยกกัน (คนละบรรทัด) ด้วย `fgets` แล้วต่อเป็นชื่อเต็ม
   อย่างปลอดภัยด้วย `strncat` (ห้ามใช้ `strcat` เปล่าๆ โดยเด็ดขาด)

4. เขียนฟังก์ชัน `int is_anagram(const char *s1, const char *s2)` ที่ตรวจสอบว่า string
   สองตัวเป็น Anagram ของกันหรือไม่ (มีตัวอักษรชุดเดียวกัน แค่เรียงลำดับต่างกัน เช่น
   `"listen"` กับ `"silent"`) คำใบ้: นับจำนวนครั้งที่แต่ละตัวอักษร (`'a'`-`'z'`) ปรากฏใน
   string ทั้งสอง แล้วเปรียบเทียบว่าเท่ากันทุกตัวหรือไม่

5. แก้ไขโค้ดต่อไปนี้ให้ปลอดภัยจาก Buffer Overflow โดยไม่เปลี่ยนพฤติกรรมหลักของโปรแกรม
   (ยังคงต้องรับชื่อจากผู้ใช้และทักทายได้เหมือนเดิม)

   ```c
   #include <stdio.h>

   int main(void) {
       char name[10];
       printf("ชื่อของคุณ: ");
       gets(name);
       printf("สวัสดี %s\n", name);
       return 0;
   }
   ```

6. เขียนฟังก์ชัน `void to_uppercase(char *s)` ที่แปลงตัวอักษรทั้งหมดใน string ให้เป็น
   ตัวพิมพ์ใหญ่ **โดยแก้ไขในหน่วยความจำเดิม (in-place)** โดยไม่สร้าง string ใหม่เลย

### แนวทางเฉลยข้อ 2

```c
/*
 * count_char.c
 * นับจำนวนครั้งที่ตัวอักษรหนึ่งปรากฏใน string
 */
#include <stdio.h>

int count_char(const char *s, char target);

int main(void) {
    const char *word = "mississippi";
    char target = 's';

    printf("ตัวอักษร '%c' ปรากฏใน \"%s\" จำนวน %d ครั้ง\n",
           target, word, count_char(word, target));

    return 0;
}

int count_char(const char *s, char target) {
    int count = 0;

    for (int i = 0; s[i] != '\0'; i++) {
        if (s[i] == target) {
            count++;
        }
    }

    return count;
}
```

ผลลัพธ์: `ตัวอักษร 's' ปรากฏใน "mississippi" จำนวน 4 ครั้ง`

โค้ดนี้แสดงรูปแบบมาตรฐานของการวน loop ผ่าน string ใน C: ใช้ `for` loop ที่ตรวจสอบเงื่อนไข
`s[i] != '\0'` แทนการระบุจำนวนรอบตายตัว เพราะเราไม่รู้ความยาวของ `s` ล่วงหน้า (หรือจะเรียก
`strlen(s)` มาเก็บไว้ก่อนก็ได้ แต่การเช็ค `'\0'` ตรงๆ ในเงื่อนไข loop เป็นวิธีที่ประหยัดกว่า
เพราะไม่ต้องวน loop อ่านความยาวถึง 2 รอบ)

### แนวทางเฉลยข้อ 5 (แก้ไข Buffer Overflow)

```c
/*
 * safe_greeting.c
 * แก้ไขจากโค้ดเดิมที่ใช้ gets() ให้ปลอดภัยด้วย fgets()
 */
#include <stdio.h>
#include <string.h>

int main(void) {
    char name[10];

    printf("ชื่อของคุณ: ");
    if (fgets(name, sizeof(name), stdin) == NULL) {
        printf("ไม่สามารถอ่านข้อมูลได้\n");
        return 1;
    }

    /* ตัด '\n' ที่ fgets อาจเก็บติดมาด้วยออก (ถ้ามีที่ว่างพอให้เก็บ) */
    size_t len = strlen(name);
    if (len > 0 && name[len - 1] == '\n') {
        name[len - 1] = '\0';
    }

    printf("สวัสดี %s\n", name);

    return 0;
}
```

การแก้ไขหลักคือเปลี่ยนจาก `gets(name)` เป็น `fgets(name, sizeof(name), stdin)` ซึ่งบังคับ
ให้อ่านข้อมูลได้ไม่เกินขนาดของ `name` เด็ดขาด (นับรวม `'\0'` ด้วย) ไม่ว่าผู้ใช้จะพิมพ์ข้อความ
ยาวแค่ไหนก็ตาม โปรแกรมจะไม่มีทาง Buffer Overflow อีกต่อไป ส่วนขั้นตอนตัด `'\n'` ท้าย string
เป็นผลข้างเคียงที่ต้องจัดการเพิ่มเติมเสมอเมื่อเปลี่ยนมาใช้ `fgets` (เพราะต่างจาก `gets` ตรงที่
`fgets` จะเก็บตัวอักษร Enter ที่ผู้ใช้กดติดมาด้วยถ้ามีพื้นที่เหลือพอ)

---

## สรุปท้ายบท

ใน Part นี้เราได้:

- ประกาศ เข้าถึง และกำหนดค่าเริ่มต้นให้ Array 1 มิติได้อย่างถูกต้อง พร้อมเข้าใจกฎ Index
  เริ่มจาก 0 เสมอ
- เห็นความสัมพันธ์เบื้องต้นระหว่าง Array กับ Pointer ผ่านสมการ `arr[i]` เทียบเท่ากับ
  `*(arr + i)` ซึ่งเป็นรากฐานสำคัญที่จะขยายความเต็มรูปแบบใน Part 8-9
- ส่ง Array เข้าไปในฟังก์ชันได้ และเข้าใจว่าทำไมฟังก์ชันแก้ไข Array ต้นฉบับได้โดยตรง
  (ต่างจากตัวแปรพื้นฐานที่เป็น Pass by Value ล้วนๆ)
- เข้าใจว่า String ในภาษา C คือ Array of char ที่ต้องมี `'\0'` กำกับท้ายเสมอ และทำไม
  รายละเอียดเล็กๆ นี้จึงสำคัญมาก
- ใช้ฟังก์ชันมาตรฐานใน `<string.h>` (`strlen`, `strcpy`, `strcat`, `strcmp`, `strncpy`)
  ได้อย่างถูกต้อง
- ระบุและป้องกันช่องโหว่ Buffer Overflow จาก `strcpy`, `strcat`, `gets` ได้ พร้อมรู้จัก
  ฟังก์ชันทดแทนที่ปลอดภัยกว่า (`fgets`, `strncpy`, `strncat`, `snprintf`)
- เขียนฟังก์ชันจัดการ string ขึ้นเอง (`my_strlen`, `my_strcpy`, `my_strcmp`, `my_reverse`)
  เพื่อเข้าใจกลไกเบื้องหลังฟังก์ชันมาตรฐานอย่างลึกซึ้ง

Array และ String คือโครงสร้างข้อมูลพื้นฐานที่สุดที่ทำให้เราจัดการข้อมูลจำนวนมากได้ในคราวเดียว
แต่ตลอด Part นี้เราได้เห็นเงาของแนวคิดสำคัญที่สุดของภาษา C ปรากฏขึ้นซ้ำๆ นั่นคือ **Pointer**
ทั้งตอนที่ชื่อ Array ทำหน้าที่ชี้ไปยังสมาชิกตัวแรก และตอนที่ฟังก์ชันรับ Array เป็นพารามิเตอร์
ใน **Part 8** เราจะหยุดพักจาก Array ชั่วคราว แล้วเจาะลึกเรื่อง **Pointer พื้นฐาน** อย่างเต็ม
รูปแบบ ตั้งแต่แนวคิดเรื่อง Address และ Memory ไปจนถึงความสัมพันธ์ระหว่าง Pointer กับ Array
ที่จะทำให้ทุกอย่างที่เรียนมาใน Part นี้ชัดเจนขึ้นไปอีกขั้น

**ต่อไป:** [Part 8 — Pointer พื้นฐาน (ตอนที่ 1)](./part-008-pointers-basics.md)
