# Part 11: Dynamic Memory Allocation (Step 81–88)

> Module B — C ระดับกลาง & โครงสร้างข้อมูล/อัลกอริทึม | Part 11 จาก 125
> Step ที่ครอบคลุมใน Part นี้: Step 81–88
> Part ก่อนหน้า: [Part 10 — Struct, Union, Enum และ typedef](./part-010-struct-union-enum.md) | Part ถัดไป: [Part 12 — Array หลายมิติและ Pointer-to-Pointer](./part-012-multidim-arrays-pointers.md)

## เป้าหมายการเรียนรู้ (Learning Objectives)

หลังจบ Part นี้ ผู้เรียนจะสามารถ:

1. อธิบายความแตกต่างระหว่าง **Stack** และ **Heap** ได้ ทั้งในแง่ Lifetime, ขนาด, และวิธีการจัดการ
2. ใช้ `malloc` จองหน่วยความจำแบบ dynamic บน Heap และ **ตรวจสอบค่าที่คืนกลับทุกครั้ง** อย่างถูกวิธี
3. อธิบายความแตกต่างระหว่าง `malloc` กับ `calloc` และเลือกใช้ให้เหมาะกับสถานการณ์
4. ใช้ `realloc` เพื่อขยาย/ลดขนาดหน่วยความจำที่จองไว้แล้ว โดยไม่ทำให้ข้อมูลเดิมหาย และไม่ทำให้เกิด memory leak เมื่อ `realloc` ล้มเหลว
5. อธิบายว่า **Memory Leak** คืออะไร เกิดจากอะไร และเขียนโค้ดที่ทำให้เกิด Memory Leak ขึ้นจริงเพื่อสังเกตอาการ
6. อธิบายและหลีกเลี่ยง **Dangling Pointer** และ **Double Free** ซึ่งเป็นบั๊กที่อันตรายที่สุดสองอย่างของการจัดการหน่วยความจำเอง
7. ออกแบบและเขียน **Dynamic Array ที่ขยายขนาดได้เอง** (Resizable Array) ตั้งแต่ต้นจนจบ ซึ่งเป็นรากฐานของ `std::vector` ใน C++ ที่จะเรียนใน Module E
8. อธิบายแนวคิด **"ใครเป็นเจ้าของหน่วยความจำนี้" (Ownership)** และเขียนฟังก์ชันที่มี "สัญญา" (Contract) เรื่อง Ownership ที่ชัดเจน

---

## 11.1 Stack กับ Heap: สองพื้นที่หน่วยความจำที่ทุกโปรแกรม C ใช้งาน (Step 81)

ตั้งแต่ Part 8-9 เราใช้ Pointer ชี้ไปที่ตัวแปรที่ประกาศแบบปกติ (`int x; int *p = &x;`) ตัวแปร
เหล่านั้นถูกสร้างขึ้นบนพื้นที่หน่วยความจำที่เรียกว่า **Stack** ซึ่งมีคุณสมบัติเฉพาะตัวที่ต่างจาก
**Heap** อย่างสิ้นเชิง

```
หน่วยความจำของโปรเซส (Process Memory Layout) แบบง่าย

 ที่อยู่สูง
 ┌─────────────────────────┐
 │        Stack            │  ← ตัวแปร local, พารามิเตอร์ฟังก์ชัน, return address
 │   (โตลง ↓ เมื่อเรียกฟังก์ชัน)│
 ├─────────────────────────┤
 │           ↓              │
 │        (ว่าง)             │
 │           ↑              │
 ├─────────────────────────┤
 │        Heap             │  ← malloc/calloc/realloc มาจองที่ตรงนี้
 │   (โตขึ้น ↑ เมื่อ malloc)  │
 ├─────────────────────────┤
 │   BSS (global ไม่ init)  │
 ├─────────────────────────┤
 │   Data (global init แล้ว)│
 ├─────────────────────────┤
 │        Text/Code        │  ← โค้ดโปรแกรมที่คอมไพล์แล้ว
 └─────────────────────────┘
 ที่อยู่ต่ำ
```

| คุณสมบัติ | Stack | Heap |
|---|---|---|
| ใครจัดการ | Compiler จัดการอัตโนมัติ | โปรแกรมเมอร์จัดการเอง (`malloc`/`free`) |
| ความเร็ว | เร็วมาก (แค่ขยับ Stack Pointer) | ช้ากว่า (ต้องค้นหาพื้นที่ว่าง) |
| Lifetime | หายไปทันทีที่ฟังก์ชันจบ (ออกจาก scope) | อยู่ได้จนกว่าจะสั่ง `free()` เอง หรือโปรแกรมจบ |
| ขนาด | จำกัด (มักไม่กี่ MB ต่อ thread) | ใหญ่กว่ามาก (จำกัดด้วย RAM/swap ของเครื่อง) |
| ความเสี่ยง | Stack Overflow (เรียก recursion ลึกเกินไป) | Memory Leak, Dangling Pointer, Fragmentation |
| ตัวอย่างการประกาศ | `int arr[100];` | `int *arr = malloc(100 * sizeof *arr);` |

**ทำไมต้องมี Heap ทั้งที่ Stack เร็วกว่า?**

ปัญหาของตัวแปรบน Stack คือ **ขนาดต้องรู้ตอนคอมไพล์ (หรืออย่างน้อยตอนเข้า scope)** และ
**อายุผูกติดกับฟังก์ชันที่ประกาศมันเสมอ** ลองดูปัญหานี้:

```c
/* โค้ดนี้มีบั๊กร้ายแรง ห้ามทำตาม! */
int *create_array_wrong(void) {
    int local_arr[100]; /* อยู่บน Stack ของฟังก์ชันนี้ */
    for (int i = 0; i < 100; i++) {
        local_arr[i] = i;
    }
    return local_arr; /* คืน pointer ชี้ไปที่ Stack ที่กำลังจะถูกทำลาย! */
}
```

เมื่อฟังก์ชัน `create_array_wrong` จบการทำงาน พื้นที่ Stack ของมัน (รวมถึง `local_arr`) จะถูก
ถือว่า "คืนกลับ" ให้ระบบทันที ผู้เรียกที่ได้ pointer นี้ไปจะได้ **Dangling Pointer** ที่ชี้ไปยัง
หน่วยความจำที่อาจถูกฟังก์ชันอื่นเขียนทับไปแล้ว — Compiler สมัยใหม่มักจะเตือนกรณีนี้ด้วย
`-Wreturn-local-addr` แต่ก็ไม่ใช่ทุกกรณีที่จะตรวจจับได้

**Heap คือทางออก** เพราะข้อมูลบน Heap **จะไม่ถูกทำลายอัตโนมัติเมื่อฟังก์ชันจบ** มันจะอยู่
ต่อไปจนกว่าเราจะสั่ง `free()` เอง (หรือโปรแกรมจบการทำงานทั้งหมด) ทำให้เราสามารถสร้างข้อมูล
ในฟังก์ชันหนึ่งแล้วส่งต่อไปให้ฟังก์ชันอื่นใช้ต่อได้อย่างปลอดภัย — นี่คือหัวใจของ Part นี้ทั้งหมด

---

## 11.2 malloc: จองหน่วยความจำบน Heap และการตรวจสอบ NULL (Step 82)

ฟังก์ชัน `malloc` (memory allocate) ถูกประกาศไว้ใน `<stdlib.h>`:

```c
void *malloc(size_t size);
```

- รับพารามิเตอร์เป็นจำนวน **byte** ที่ต้องการจอง
- คืนค่าเป็น `void *` (generic pointer) ที่ชี้ไปยังพื้นที่ที่จองได้ — เราต้อง cast หรือกำหนดให้
  ตัวแปรชนิดที่ถูกต้องรับไว้
- **เนื้อหาข้างในเป็นค่าขยะ (Garbage/Indeterminate Value)** — `malloc` ไม่ initialize ค่าให้
- ถ้าจองไม่ได้ (หน่วยความจำในระบบไม่พอ) จะคืนค่า **`NULL`**

```c
/* mem_basic.c */
#include <stdio.h>
#include <stdlib.h>

int main(void) {
    size_t n = 5;
    int *arr = malloc(n * sizeof *arr);
    if (arr == NULL) {
        fprintf(stderr, "malloc ล้มเหลว: ไม่สามารถจองหน่วยความจำได้\n");
        return 1;
    }

    for (size_t i = 0; i < n; i++) {
        arr[i] = (int)(i * i);
    }
    for (size_t i = 0; i < n; i++) {
        printf("arr[%zu] = %d\n", i, arr[i]);
    }

    free(arr);
    arr = NULL;
    return 0;
}
```

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 mem_basic.c -o mem_basic
./mem_basic
```

ผลลัพธ์:

```
arr[0] = 0
arr[1] = 1
arr[2] = 4
arr[3] = 9
arr[4] = 16
```

### จุดสำคัญที่ต้องสังเกต

1. **`sizeof *arr` แทนที่จะเขียน `sizeof(int)`** — นี่เป็น idiom ที่มืออาชีพใช้กันเป็นมาตรฐาน
   เพราะถ้าวันหนึ่งเราเปลี่ยนชนิดของ `arr` จาก `int *` เป็น `double *` โค้ดส่วนนี้จะยังถูกต้อง
   โดยอัตโนมัติโดยไม่ต้องแก้ไขทุกจุดที่เรียก `malloc`
2. **ตรวจสอบ `NULL` เสมอ** — แม้ในเครื่องพัฒนาที่มี RAM เยอะๆ `malloc` แทบไม่มีทางล้มเหลว
   แต่ในระบบ Embedded, Server ที่รับโหลดหนัก, หรือเมื่อขอจองก้อนใหญ่ผิดปกติ (เช่นเผลอคำนวณขนาด
   ผิดจนติดลบแล้ว wrap เป็นเลขมหาศาล) `malloc` **จะ** คืน `NULL` และถ้าเราไม่เช็คแล้วเอาไป
   dereference (`arr[0] = ...` ทั้งที่ `arr == NULL`) จะได้ **Segmentation Fault** ทันที
3. **`free(arr)` เมื่อใช้เสร็จ** — ทุกการ `malloc` ต้องมี `free` คู่กันเสมอ (จะอธิบายเจาะลึกใน
   หัวข้อ 11.5)
4. **ตั้ง `arr = NULL` หลัง `free`** — เป็นนิสัยที่ดีเพื่อป้องกัน dangling pointer (หัวข้อ 11.6)

> **กฎทองของ Dynamic Memory**: ทุกครั้งที่เรียก `malloc`/`calloc`/`realloc` ต้องตรวจสอบค่าที่
> คืนกลับว่าเป็น `NULL` หรือไม่ **ก่อน** ที่จะเอา pointer นั้นไปใช้งานเสมอ ไม่มีข้อยกเว้น

---

## 11.3 calloc: malloc ที่มาพร้อมการล้างค่าเป็นศูนย์ (Step 83)

```c
void *calloc(size_t num, size_t size);
```

`calloc` (contiguous allocation) ต่างจาก `malloc` สองจุดหลัก:

1. รับ 2 พารามิเตอร์: จำนวน element และขนาดต่อ element (แทนที่จะคูณเองแบบ `malloc`)
2. **รับประกันว่าหน่วยความจำที่จองมาจะถูกล้างเป็น 0 ทั้งหมด** (`malloc` ไม่รับประกันแบบนี้)

```c
/* mem_calloc.c */
#include <stdio.h>
#include <stdlib.h>

int main(void) {
    size_t n = 5;
    int *a = malloc(n * sizeof *a);
    int *b = calloc(n, sizeof *b);
    if (a == NULL || b == NULL) {
        fprintf(stderr, "จองหน่วยความจำล้มเหลว\n");
        free(a);
        free(b);
        return 1;
    }

    printf("calloc: ");
    for (size_t i = 0; i < n; i++) {
        printf("%d ", b[i]); /* รับประกันว่าเป็น 0 ทุกตัว */
    }
    printf("\n");

    free(a);
    free(b);
    return 0;
}
```

```
calloc: 0 0 0 0 0
```

(ถ้าลอง `printf` ค่าของ `a` ที่ได้จาก `malloc` โดยไม่กำหนดค่าก่อน จะได้ค่าขยะที่ไม่แน่นอน — 
ห้ามอ่านค่านั้นและนำไปใช้ในโค้ดจริงเด็ดขาด เพราะเป็น Undefined Behavior)

### malloc vs calloc: ใช้เมื่อไหร่

| สถานการณ์ | ควรใช้ |
|---|---|
| ต้องการเขียนค่าลงทุกช่องทันทีอยู่แล้ว (เช่นวน loop ใส่ค่าเลย) | `malloc` (เร็วกว่าเล็กน้อย เพราะไม่ต้องเสียเวลาล้างค่า) |
| ต้องการค่าเริ่มต้นเป็น 0 แน่นอน (เช่น counter array, ตัวแปร flag) | `calloc` |
| จองหน่วยความจำสำหรับ struct ที่มี pointer สมาชิกข้างใน และต้องการให้เริ่มต้นเป็น `NULL` | `calloc` (เพราะ NULL บนแพลตฟอร์มส่วนใหญ่แทนด้วย bit pattern ที่เป็น 0 ทั้งหมด) |
| จองก้อนใหญ่มาก และกังวลเรื่อง `num * size` overflow | `calloc` (คอมไพเลอร์/ไลบรารีมาตรฐานจะตรวจสอบ overflow ให้ก่อนคูณ ต่างจาก `malloc(n * size)` ที่เราคูณเอง) |

---

## 11.4 realloc: ขยาย (หรือลด) ขนาดหน่วยความจำที่จองไว้แล้ว (Step 84)

```c
void *realloc(void *ptr, size_t new_size);
```

`realloc` ใช้เมื่อเราจองหน่วยความจำไว้แล้ว แต่ภายหลังพบว่าขนาดไม่พอ (หรือใหญ่เกินไป) โดย
**ข้อมูลเดิมจะถูกเก็บรักษาไว้** (เท่าที่ขนาดใหม่จะรองรับได้) พฤติกรรมของ `realloc` มี 3 กรณี:

1. ถ้ามีพื้นที่ว่างต่อจากก้อนเดิมพอ → ขยายในตำแหน่งเดิม คืน pointer เดิม
2. ถ้าไม่มีพื้นที่ว่างพอ → จองก้อนใหม่ที่อื่น, **copy ข้อมูลเดิมไปให้อัตโนมัติ**, free ก้อนเก่า,
   คืน pointer ใหม่ (ที่อยู่อาจเปลี่ยนไป!)
3. ถ้าจองไม่ได้เลย → **คืน `NULL` และ "ไม่แตะต้อง" ก้อนเดิม** (ก้อนเดิมยังใช้งานได้ปกติ)

กรณีที่ 3 นี้สำคัญมาก และเป็นสาเหตุของบั๊กที่พบบ่อยที่สุดของ `realloc`:

```c
/* ห้ามทำแบบนี้! ถ้า realloc คืน NULL เราจะ "ทำ pointer เดิมหาย" ทันที
   กลายเป็น Memory Leak เพราะไม่มีใครชี้ไปที่ก้อนเดิมอีกแล้ว */
data = realloc(data, new_size); /* ผิด: ถ้า realloc ล้มเหลว data จะกลายเป็น NULL
                                    และก้อนเดิมที่ยังไม่ถูก free ก็หายไปตลอดกาล */
```

**วิธีที่ถูกต้อง**: ใช้ตัวแปรชั่วคราวรับค่าก่อนเสมอ

```c
/* mem_realloc.c */
#include <stdio.h>
#include <stdlib.h>

int main(void) {
    size_t capacity = 4;
    int *data = malloc(capacity * sizeof *data);
    if (data == NULL) {
        fprintf(stderr, "malloc ล้มเหลว\n");
        return 1;
    }
    for (size_t i = 0; i < capacity; i++) {
        data[i] = (int)i + 1;
    }

    size_t new_capacity = capacity * 2;
    int *tmp = realloc(data, new_capacity * sizeof *tmp); /* รับด้วยตัวแปรชั่วคราว */
    if (tmp == NULL) {
        fprintf(stderr, "realloc ล้มเหลว: data เดิมยังใช้ได้อยู่\n");
        free(data); /* data เดิมยังปลอดภัย จึง free ได้ตามปกติ */
        return 1;
    }
    data = tmp; /* สลับมาใช้ก้อนใหม่ (หรือก้อนเดิมที่ถูกขยายแล้ว) ก็ต่อเมื่อสำเร็จ */
    capacity = new_capacity;

    for (size_t i = 4; i < capacity; i++) {
        data[i] = (int)i + 1;
    }

    for (size_t i = 0; i < capacity; i++) {
        printf("%d ", data[i]);
    }
    printf("\n");

    free(data);
    return 0;
}
```

```
1 2 3 4 5 6 7 8
```

สังเกตว่าค่า 4 ตัวแรก (`1 2 3 4`) ที่ตั้งไว้ก่อน `realloc` ยังอยู่ครบหลังขยายขนาด — นี่คือสิ่งที่
ทำให้ `realloc` ต่างจากการ `malloc` ก้อนใหม่แล้ว copy เองทุกอย่าง (แม้เบื้องหลังบางครั้ง
`realloc` ก็ทำแบบนั้นจริงๆ แต่มันทำให้เราโดยอัตโนมัติ)

`realloc(ptr, 0)` ในบางไลบรารีมีพฤติกรรมเหมือน `free(ptr)` และคืน `NULL` แต่พฤติกรรมนี้
เปลี่ยนแปลงได้ระหว่าง implementation ตั้งแต่ C23 เป็นต้นไปจะระบุชัดว่าเป็น Undefined ดังนั้น
**ในหลักสูตรนี้จะไม่พึ่งพฤติกรรมนี้** ถ้าต้องการคืนหน่วยความจำทั้งหมดให้เรียก `free` ตรงๆ

---

## 11.5 free และ Memory Leak (Step 85)

```c
void free(void *ptr);
```

`free` คืนหน่วยความจำที่จองด้วย `malloc`/`calloc`/`realloc` กลับให้ระบบ เพื่อให้ส่วนอื่นของ
โปรแกรม (หรือโปรแกรมอื่น) นำไปใช้ต่อได้ กฎการใช้งาน:

- `free(NULL)` **ปลอดภัยเสมอ** — มาตรฐาน C รับประกันว่าไม่ทำอะไรเลยถ้า pointer เป็น `NULL`
  (ดังนั้นไม่จำเป็นต้องเขียน `if (p != NULL) free(p);` — เขียน `free(p);` เฉยๆ ได้เลย)
- ห้าม `free` pointer ที่ไม่ได้มาจาก `malloc`/`calloc`/`realloc` (เช่น pointer ชี้ไปยังตัวแปร
  บน Stack) — เป็น Undefined Behavior ทันที
- ห้าม `free` ซ้ำสอง (Double Free — หัวข้อ 11.6)
- หลัง `free` แล้ว ห้ามเข้าถึงข้อมูลผ่าน pointer นั้นอีก (Dangling Pointer — หัวข้อ 11.6)

### Memory Leak คืออะไร

**Memory Leak (หน่วยความจำรั่ว)** เกิดขึ้นเมื่อโปรแกรม `malloc` หน่วยความจำมาแล้ว **ไม่มี
pointer ใดๆ ชี้ไปยังมันอีกต่อไป** (pointer ตัวสุดท้ายที่ชี้ไปหายไปจาก scope หรือถูกเขียนทับ)
โดยที่ยังไม่ได้ `free` — ผลคือหน่วยความจำก้อนนั้นจะ "ค้าง" อยู่ในระบบตลอดไป (จนกว่าโปรแกรม
จะจบการทำงานทั้งหมด) โดยไม่มีทางเรียกคืนมาได้อีก

```c
/* mem_leak.c — ตัวอย่างที่ทำให้เกิด Memory Leak โดยตั้งใจ (เพื่อการศึกษาเท่านั้น) */
#include <stdlib.h>

void leaky_function(void) {
    int *p = malloc(100 * sizeof *p);
    if (p == NULL) {
        return;
    }
    p[0] = 42;
    /* ลืม free(p) ก่อนออกจากฟังก์ชัน -> เกิด memory leak ทุกครั้งที่ฟังก์ชันนี้ถูกเรียก
       เพราะ p เป็นตัวแปร local บน Stack ที่หายไปเมื่อฟังก์ชันจบ
       แต่หน่วยความจำที่ p เคยชี้ไปอยู่บน Heap ไม่ได้หายตามไปด้วย! */
}

int main(void) {
    for (int i = 0; i < 1000; i++) {
        leaky_function(); /* รั่วซ้ำ 1000 ครั้ง = รั่วรวม 1000 * 100 * sizeof(int) bytes */
    }
    return 0;
}
```

โค้ดนี้คอมไพล์ผ่านโดยไม่มี Warning เลย (`-Wall -Wextra -Wpedantic` จับ Memory Leak **ไม่ได้**
ในระดับการคอมไพล์ เพราะการวิเคราะห์ ownership ของหน่วยความจำต้องใช้เครื่องมือพิเศษ) — 
นี่คือเหตุผลที่ Memory Leak เป็นบั๊กที่อันตราย เพราะโปรแกรมยังทำงาน "ถูกต้อง" ในสายตาที่มองแค่
ผลลัพธ์ แต่ค่อยๆ กิน RAM มากขึ้นเรื่อยๆ จนวันหนึ่งระบบอาจช้าลงหรือ crash เพราะ RAM หมด

ใน **Part 38 (Valgrind)** เราจะเรียนเครื่องมือที่ตรวจจับ Memory Leak แบบนี้ได้อัตโนมัติ แต่
ตอนนี้อยากให้จำหลักการง่ายๆ ไว้ก่อน:

> **หลักการ "1 malloc ต้องมี 1 free"**: ทุกครั้งที่เขียน `malloc`/`calloc`/`realloc` ให้ถามตัวเอง
> ทันทีว่า "แล้วใครจะเป็นคน `free` มัน และ `free` ที่ไหน" ถ้าตอบไม่ได้ทันที มีโอกาสสูงที่จะลืม

---

## 11.6 Dangling Pointer และ Double Free (Step 86)

### Dangling Pointer

**Dangling Pointer** คือ pointer ที่ยังคง "ชี้ไปยังที่อยู่" ของหน่วยความจำที่ **ถูกคืนกลับไปแล้ว**
(ผ่าน `free`) การเข้าถึงข้อมูลผ่าน dangling pointer เป็น **Undefined Behavior** — อาจได้ค่าขยะ
เก่า, ค่าที่ถูกโปรแกรมส่วนอื่นเขียนทับไปแล้ว, หรือโปรแกรม crash ทันที

```c
/* mem_dangling.c — ตัวอย่างบั๊กที่ตั้งใจสร้างขึ้นเพื่อการศึกษา ห้ามทำตามในโค้ดจริง! */
#include <stdio.h>
#include <stdlib.h>

int main(void) {
    int *p = malloc(sizeof *p);
    if (p == NULL) {
        return 1;
    }
    *p = 10;
    free(p);
    /* จากบรรทัดนี้ไป p คือ Dangling Pointer เพราะหน่วยความจำถูกคืนแล้ว */
    printf("%d\n", *p); /* Undefined Behavior: ห้ามทำแบบนี้! */
    return 0;
}
```

ลองคอมไพล์โค้ดนี้ด้วย GCC เวอร์ชันใหม่ (13 ขึ้นไป) จะเห็นว่า compiler ฉลาดพอจะจับได้เลยว่า
เราใช้ pointer หลัง free:

```
mem_dangling.c: In function 'main':
mem_dangling.c:13:5: warning: pointer 'p' used after 'free' [-Wuse-after-free]
   13 |     printf("%d\n", *p); /* Undefined Behavior: ห้ามทำแบบนี้! */
      |     ^~~~~~~~~~~~~~~~~~
mem_dangling.c:10:5: note: call to 'free' here
   10 |     free(p);
      |     ^~~~~~~
```

นี่คือตัวอย่างที่ดีว่าทำไม `-Wall -Wextra` ถึงสำคัญมาก — คอมไพเลอร์สมัยใหม่เริ่มตรวจจับ
Use-After-Free บางกรณีได้แล้วในระดับ static analysis แต่ **ไม่ใช่ทุกกรณี** (เช่นถ้า `free`
กับการใช้งานอยู่คนละฟังก์ชันกัน compiler มักตรวจไม่พบ) ดังนั้นห้ามพึ่งพา compiler
เพียงอย่างเดียว ต้องเข้าใจหลักการและระวังด้วยตัวเองเสมอ

**วิธีป้องกัน**: ตั้งค่า pointer เป็น `NULL` ทันทีหลัง `free` เสมอ

```c
free(p);
p = NULL; /* ตอนนี้ p ไม่ใช่ dangling pointer อีกต่อไป มันคือ NULL pointer ที่ปลอดภัย */
```

การเข้าถึง `*p` ตอนนี้จะได้ Segmentation Fault ทันที (crash ทันทีอย่างชัดเจน) ซึ่ง **ดีกว่า**
Undefined Behavior แบบเงียบๆ มาก เพราะเราจะเจอบั๊กทันทีตอนทดสอบ แทนที่จะซ่อนอยู่ลึกๆ
แล้วมาแสดงอาการตอนโปรแกรม deploy ไปแล้ว

### Double Free

**Double Free** คือการเรียก `free` กับ pointer ตัวเดียวกันซ้ำสองครั้ง (หรือมากกว่า) โดยไม่ได้
`malloc` ใหม่คั่นกลาง — เป็น Undefined Behavior ที่มักทำให้ heap ของโปรแกรมเสียหาย (heap
corruption) และเป็นช่องโหว่ด้านความปลอดภัยที่ถูกใช้โจมตีจริงในโลก security บ่อยครั้ง

```c
/* mem_doublefree.c — ตัวอย่างบั๊กที่ตั้งใจสร้างขึ้นเพื่อการศึกษา ห้ามทำตามในโค้ดจริง! */
#include <stdlib.h>

int main(void) {
    int *p = malloc(sizeof *p);
    if (p == NULL) {
        return 1;
    }
    free(p);
    free(p); /* Double Free: Undefined Behavior ห้ามทำแบบนี้! */
    return 0;
}
```

GCC จะเตือนกรณีง่ายๆ แบบนี้เช่นกัน (`-Wuse-after-free`) แต่ในโปรแกรมจริงที่ซับซ้อนกว่านี้
(เช่น `free` ผ่านสองเส้นทางของโค้ดที่ไม่คาดคิดว่าจะมาเจอกัน) compiler มักตรวจจับไม่ได้

**วิธีป้องกัน**: หลักการเดียวกับ dangling pointer — ตั้งเป็น `NULL` หลัง `free` เสมอ เพราะ
`free(NULL)` ปลอดภัย ต่อให้โค้ดพลาดเรียก `free` ซ้ำ ก็จะไม่เกิดอะไรขึ้น

| บั๊ก | เกิดจาก | อาการที่พบ | วิธีป้องกันหลัก |
|---|---|---|---|
| Memory Leak | จองแล้วไม่ปล่อย | RAM ใช้เพิ่มขึ้นเรื่อยๆ ไม่ crash ทันที | ทุก malloc ต้องคู่กับ free เสมอ ตรวจสอบด้วย Valgrind |
| Dangling Pointer | ใช้ pointer หลัง free | อ่าน/เขียนค่าขยะ, crash แบบสุ่ม | ตั้ง `p = NULL` หลัง free ทันที |
| Double Free | free ซ้ำ pointer เดิม | Heap corruption, crash, ช่องโหว่ security | ตั้ง `p = NULL` หลัง free (เพราะ `free(NULL)` ปลอดภัย) |

---

## 11.7 เขียน Dynamic Array ที่ขยายขนาดได้เอง (Step 87)

ตอนนี้เรามีความรู้ครบพอที่จะเขียนโครงสร้างข้อมูลที่ทรงพลังที่สุดอย่างหนึ่งของ C: **Resizable
Array** (บางครั้งเรียก "Dynamic Array" หรือใน C++ คือ `std::vector` ที่จะเรียนใน Part 59)
แนวคิดคือ:

1. เก็บ pointer ไปยังข้อมูล, จำนวนข้อมูลปัจจุบัน (`size`), และความจุที่จองไว้ (`capacity`)
2. เมื่อจะเพิ่มข้อมูลแต่ `size == capacity` (พื้นที่เต็มแล้ว) ให้ `realloc` ขยาย `capacity`
   (มักจะขยายเป็น 2 เท่าของเดิม เพื่อลดจำนวนครั้งที่ต้อง `realloc` โดยรวม)

```c
/* int_vector.c — resizable dynamic array (คล้าย std::vector แบบง่าย) */
#include <stdio.h>
#include <stdlib.h>

typedef struct {
    int *data;
    size_t size;      /* จำนวนข้อมูลที่ใช้งานจริงตอนนี้ */
    size_t capacity;  /* จำนวนช่องที่จองไว้ทั้งหมด (>= size เสมอ) */
} IntVector;

void iv_init(IntVector *v) {
    v->data = NULL;
    v->size = 0;
    v->capacity = 0;
}

/* คืน 1 ถ้าสำเร็จ, 0 ถ้า realloc ล้มเหลว (v ยังอยู่ในสถานะที่ใช้งานได้ปกติ) */
int iv_push_back(IntVector *v, int value) {
    if (v->size == v->capacity) {
        size_t new_capacity = (v->capacity == 0) ? 4 : v->capacity * 2;
        int *tmp = realloc(v->data, new_capacity * sizeof *tmp);
        if (tmp == NULL) {
            return 0; /* v->data เดิมยังปลอดภัย ไม่มีอะไรเสียหาย */
        }
        v->data = tmp;
        v->capacity = new_capacity;
    }
    v->data[v->size] = value;
    v->size++;
    return 1;
}

void iv_free(IntVector *v) {
    free(v->data);
    v->data = NULL;
    v->size = 0;
    v->capacity = 0;
}

int main(void) {
    IntVector v;
    iv_init(&v);

    for (int i = 1; i <= 10; i++) {
        if (!iv_push_back(&v, i * i)) {
            fprintf(stderr, "push_back ล้มเหลว: หน่วยความจำไม่พอ\n");
            iv_free(&v);
            return 1;
        }
        printf("push %d -> size=%zu capacity=%zu\n", i * i, v.size, v.capacity);
    }

    printf("ค่าทั้งหมด: ");
    for (size_t i = 0; i < v.size; i++) {
        printf("%d ", v.data[i]);
    }
    printf("\n");

    iv_free(&v);
    return 0;
}
```

ผลลัพธ์:

```
push 1 -> size=1 capacity=4
push 4 -> size=2 capacity=4
push 9 -> size=3 capacity=4
push 16 -> size=4 capacity=4
push 25 -> size=5 capacity=8
push 36 -> size=6 capacity=8
push 49 -> size=7 capacity=8
push 64 -> size=8 capacity=8
push 81 -> size=9 capacity=16
push 100 -> size=10 capacity=16
ค่าทั้งหมด: 1 4 9 16 25 36 49 64 81 100
```

สังเกตว่า `capacity` เพิ่มเป็น 4 → 8 → 16 (เพิ่มเป็น 2 เท่าเสมอ ไม่ใช่ +1 ทีละครั้ง) — นี่คือ
เทคนิคสำคัญที่เรียกว่า **Amortized Growth** ถ้าเราขยาย capacity ทีละ 1 ทุกครั้งที่ push
(`capacity = size + 1`) จะต้อง `realloc` (ซึ่งอาจต้อง copy ข้อมูลทั้งหมด) ทุกครั้งที่เพิ่มข้อมูล
1 ตัว ทำให้การเพิ่มข้อมูล n ตัวใช้เวลารวม O(n²) แต่การเพิ่มเป็น 2 เท่าทำให้การ `realloc`
เกิดขึ้นเพียง O(log n) ครั้ง และเวลารวมทั้งหมดลดเหลือ O(n) — หลักการนี้คือสิ่งเดียวกับที่
`std::vector` ของ C++ ใช้จริงในทุก implementation

> ในทางปฏิบัติ โครงสร้างข้อมูลแบบ Resizable Array นี้จะถูกนำไปประยุกต์ใช้ซ้ำๆ ตลอดหลักสูตร
> โดยเฉพาะใน **Part 19-22 (Linked List, Stack/Queue, Tree, Hash Table)** ที่ต้องออกแบบว่า
> "จะเก็บข้อมูลจำนวนไม่แน่นอนได้อย่างไร" — คำตอบเกือบทุกครั้งเกี่ยวข้องกับ Dynamic Memory
> ที่เรียนใน Part นี้ทั้งสิ้น

---

## 11.8 แนวคิด "ใครเป็นเจ้าของหน่วยความจำนี้" (Ownership) (Step 88)

ปัญหาที่ยากที่สุดของการเขียนโปรแกรม C ขนาดใหญ่ที่มีหลายฟังก์ชันเรียกกันไปมา **ไม่ใช่**
วิธีเรียก `malloc`/`free` (ซึ่งเป็นแค่ syntax) แต่คือการตอบคำถามว่า **"ใครมีหน้าที่ `free`
หน่วยความจำก้อนนี้"** — คำถามนี้เรียกว่า **Ownership (ความเป็นเจ้าของ)**

ถ้าไม่มีการตกลง (contract) ที่ชัดเจน จะเกิดปัญหาสองแบบ:

- **ไม่มีใคร free เลย** เพราะทุกฝ่ายคิดว่าอีกฝ่าย free ให้ → Memory Leak
- **มีมากกว่าหนึ่งฝ่าย free** เพราะทุกฝ่ายคิดว่าตัวเองต้อง free → Double Free

### รูปแบบ Ownership ที่พบบ่อยที่สุดใน C

**แบบที่ 1: ฟังก์ชันคืนหน่วยความจำใหม่ → ผู้เรียกเป็นเจ้าของ**

```c
/* ownership_demo.c */
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

/* Ownership Contract: ฟังก์ชันนี้ malloc หน่วยความจำใหม่และคืนกลับไป
 * ผู้เรียก (caller) เป็นเจ้าของหน่วยความจำนี้ และมีหน้าที่ free() เอง
 * คืนค่า NULL ถ้า malloc ล้มเหลว */
char *make_greeting(const char *name) {
    size_t len = strlen(name);
    char *buf = malloc(len + 32);
    if (buf == NULL) {
        return NULL;
    }
    snprintf(buf, len + 32, "สวัสดี, %s!", name);
    return buf;
}

int main(void) {
    char *msg = make_greeting("สมชาย");
    if (msg == NULL) {
        fprintf(stderr, "จองหน่วยความจำล้มเหลว\n");
        return 1;
    }
    printf("%s\n", msg);

    free(msg); /* main คือเจ้าของ จึงเป็นผู้ free */
    msg = NULL;
    return 0;
}
```

**แบบที่ 2: ฟังก์ชันแค่ "ยืมดู" (Borrow) ไม่ได้เป็นเจ้าของ**

ฟังก์ชันแบบ `void print_matrix(int **mat, size_t rows, size_t cols)` ที่จะเรียนใน Part 12
รับ pointer เข้ามาแค่ **อ่านหรือแก้ไขข้อมูล** แต่ **ไม่มีสิทธิ์ `free`** หน่วยความจำนั้น — ฟังก์ชัน
แบบนี้ถือว่า "ยืม" (borrow) หน่วยความจำมาใช้ชั่วคราวเท่านั้น เจ้าของตัวจริงยังเป็นคนเดิม

**แบบที่ 3: ฟังก์ชันรับ pointer มาแล้ว "รับช่วงเป็นเจ้าของ" (Transfer Ownership)**

พบได้ในฟังก์ชันอย่าง `iv_free(&v)` ในหัวข้อที่แล้ว — เมื่อเรียกแล้ว หน่วยความจำจะถูก `free`
ข้างในทันที ผู้เรียกห้ามใช้ `v.data` อีกต่อไปหลังเรียกฟังก์ชันนี้

### วิธีสื่อสาร Ownership ในโค้ดจริง

เพราะภาษา C **ไม่มีกลไกทางภาษาที่บังคับเรื่อง Ownership** (ต่างจาก Rust ที่มี Borrow Checker
หรือ C++ ที่มี Smart Pointer ซึ่งจะเรียนใน Part 67) วิธีเดียวที่ทำได้ในภาษา C คือ **เขียน
comment เป็นสัญญา (contract) กำกับไว้ที่ prototype ของฟังก์ชันทุกตัวที่เกี่ยวข้องกับ dynamic
memory เสมอ** ดังตัวอย่างด้านบนที่เขียนกำกับด้วยคำว่า "Ownership Contract"

| รูปแบบ | ตัวอย่าง | ใครต้อง free |
|---|---|---|
| ฟังก์ชันคืน pointer ใหม่ | `char *make_greeting(...)` | ผู้เรียก (caller) |
| ฟังก์ชันรับ pointer มาแค่อ่าน/แก้ไข | `void print_matrix(int **mat, ...)` | ไม่มีใคร free ในฟังก์ชันนี้ เจ้าของเดิมยัง free เอง |
| ฟังก์ชันรับ pointer มาเพื่อทำลาย | `void iv_free(IntVector *v)` | ฟังก์ชันนี้เอง (caller ห้ามใช้ต่อ) |

การฝึกคิดเรื่อง Ownership ให้เป็นนิสัยตั้งแต่ Part นี้ จะทำให้การเขียน Linked List (Part 19),
Tree (Part 21), Hash Table (Part 22) และโปรเจกต์ใหญ่ใน Module C ง่ายขึ้นมาก เพราะโครงสร้าง
ข้อมูลเหล่านั้นล้วนสร้างขึ้นจาก `malloc`/`free` ที่กระจายอยู่หลายฟังก์ชันทั้งสิ้น

---

## ข้อผิดพลาดที่พบบ่อย (Common Pitfalls)

1. **ลืมตรวจสอบค่าที่ `malloc`/`calloc`/`realloc` คืนกลับว่าเป็น `NULL` หรือไม่** — ทำให้เมื่อ
   ระบบขาดหน่วยความจำจริง โปรแกรมจะ Segmentation Fault ทันทีที่ dereference pointer ที่เป็น
   `NULL` แทนที่จะแจ้ง error อย่างสุภาพ
2. **เขียน `p = realloc(p, new_size);` ตรงๆ โดยไม่ใช้ตัวแปรชั่วคราว** — ถ้า `realloc` ล้มเหลว
   `p` จะกลายเป็น `NULL` ทันที ทำให้ pointer ก้อนเดิมหายไปตลอดกาล (Memory Leak) ทั้งที่
   ก้อนเดิมยังใช้งานได้อยู่
3. **ลืม `free` ในบางเส้นทางของโค้ด (โดยเฉพาะ early return ตอน error)** — เป็นสาเหตุอันดับหนึ่ง
   ของ Memory Leak ในโค้ดจริง ต้องตรวจสอบทุก `return` ในฟังก์ชันว่าถ้ามี `malloc` ค้างอยู่ก่อนหน้า
   ต้อง `free` ก่อน return ด้วยหรือไม่
4. **ใช้ pointer ต่อหลัง `free` (Use-After-Free / Dangling Pointer)** — โดยเฉพาะเมื่อมี
   หลาย pointer ชี้ไปยังก้อนเดียวกัน (aliasing) แล้ว `free` ผ่าน pointer ตัวหนึ่ง แต่ลืมว่า pointer
   ตัวอื่นก็ชี้ไปที่เดียวกัน
5. **`free` ซ้ำสอง (Double Free)** — มักเกิดจากโครงสร้างโค้ดที่ซับซ้อน มีหลายเส้นทางเรียก
   cleanup function ซ้ำกัน วิธีป้องกันที่ง่ายที่สุดคือตั้ง pointer เป็น `NULL` ทันทีหลัง `free` ทุกครั้ง
6. **สับสนขนาดที่ส่งให้ `malloc`** — เช่นเผลอเขียน `malloc(n)` แทน `malloc(n * sizeof(int))`
   (จองน้อยไปมาก ทำให้เขียนข้อมูลเกินขอบเขต — Buffer Overflow) หรือคำนวณขนาดจนเกิด
   integer overflow เมื่อ `n` มีค่ามากผิดปกติ
7. **`free` pointer ที่ไม่ได้มาจาก `malloc`/`calloc`/`realloc`** เช่น `free(&local_var)` ที่ `local_var`
   อยู่บน Stack — เป็น Undefined Behavior ทันที

---

## แบบฝึกหัดท้ายบท

1. เขียนฟังก์ชัน `double *make_squares(size_t n)` ที่ `malloc` array ของ `double` ขนาด `n`
   แล้วเติมค่ากำลังสองของ index แต่ละตัว (`arr[i] = i * i`) จากนั้นคืน pointer กลับไป โดยเขียน
   comment กำกับ Ownership Contract ให้ชัดเจนว่าใครต้อง `free`
2. จากโครงสร้าง `IntVector` ในหัวข้อ 11.7 ให้เขียนฟังก์ชันเพิ่ม `int iv_shrink_to_fit(IntVector *v)`
   ที่ใช้ `realloc` ลด `capacity` ให้เท่ากับ `size` พอดี (คืนหน่วยความจำส่วนเกินกลับให้ระบบ)
3. โค้ดด้านล่างนี้มีบั๊กเรื่อง Memory Leak ซ่อนอยู่ ให้หาและแก้ไข:
   ```c
   void process_and_cleanup(size_t n) {
       int *buf = malloc(n * sizeof *buf);
       if (buf == NULL) {
           fprintf(stderr, "malloc ล้มเหลว\n");
           return;
       }
       for (size_t i = 0; i < n; i++) {
           buf[i] = (int)i;
       }
       long sum = 0;
       for (size_t i = 0; i < n; i++) {
           sum += buf[i];
       }
       printf("ผลรวม = %ld\n", sum);
       /* บั๊ก: โค้ดจบตรงนี้โดยไม่มี free(buf) เลย! */
   }
   ```
4. เขียนฟังก์ชัน `int iv_insert_front(IntVector *v, int value)` ที่แทรกค่าใหม่ไว้ที่ตำแหน่งแรกสุด
   ของ `IntVector` (ต้องขยับข้อมูลเดิมทั้งหมดไปทางขวา 1 ตำแหน่งก่อน โดยอาจใช้ `memmove`
   จาก `<string.h>`)
5. อธิบายด้วยคำพูดของตัวเองว่าทำไม `calloc(n, size)` ปลอดภัยกว่า `malloc(n * size)` ในแง่ของ
   ความเสี่ยงเรื่อง integer overflow เมื่อ `n` และ `size` มีค่ามากทั้งคู่
6. เขียนโปรแกรมทดสอบที่จงใจสร้าง Dangling Pointer ขึ้นมา (คล้ายตัวอย่างในหัวข้อ 11.6) แล้ว
   สังเกตว่า GCC เตือน `-Wuse-after-free` หรือไม่ในเครื่องของผู้เรียนเอง (ขึ้นกับเวอร์ชัน GCC)

### แนวทางเฉลยข้อ 1

```c
#include <stdio.h>
#include <stdlib.h>

/* Ownership Contract: ฟังก์ชันนี้คืน pointer ไปยัง array ของ double ที่ malloc
 * ไว้ขนาด n ตัว ผู้เรียกฟังก์ชันนี้เป็นเจ้าของหน่วยความจำ และต้อง free() เอง
 * คืนค่า NULL ถ้า malloc ล้มเหลว */
double *make_squares(size_t n) {
    double *arr = malloc(n * sizeof *arr);
    if (arr == NULL) {
        return NULL;
    }
    for (size_t i = 0; i < n; i++) {
        arr[i] = (double)i * (double)i;
    }
    return arr;
}

int main(void) {
    size_t n = 6;
    double *squares = make_squares(n);
    if (squares == NULL) {
        fprintf(stderr, "จองหน่วยความจำล้มเหลว\n");
        return EXIT_FAILURE;
    }

    for (size_t i = 0; i < n; i++) {
        printf("%.1f ", squares[i]);
    }
    printf("\n");

    free(squares); /* main เป็นเจ้าของ จึงเป็นผู้ free */
    return EXIT_SUCCESS;
}
```

ผลลัพธ์: `0.0 1.0 4.0 9.0 16.0 25.0`

### แนวทางเฉลยข้อ 2

```c
#include <stdio.h>
#include <stdlib.h>

typedef struct {
    int *data;
    size_t size;
    size_t capacity;
} IntVector;

void iv_init(IntVector *v) {
    v->data = NULL;
    v->size = 0;
    v->capacity = 0;
}

int iv_push_back(IntVector *v, int value) {
    if (v->size == v->capacity) {
        size_t new_capacity = (v->capacity == 0) ? 4 : v->capacity * 2;
        int *tmp = realloc(v->data, new_capacity * sizeof *tmp);
        if (tmp == NULL) {
            return 0;
        }
        v->data = tmp;
        v->capacity = new_capacity;
    }
    v->data[v->size] = value;
    v->size++;
    return 1;
}

/* ลดขนาด capacity ให้เท่ากับ size พอดี เพื่อคืนหน่วยความจำส่วนเกินให้ระบบ
 * คืนค่า 1 ถ้าสำเร็จ, 0 ถ้า realloc ล้มเหลว (v ยังใช้งานได้ตามปกติ) */
int iv_shrink_to_fit(IntVector *v) {
    if (v->size == v->capacity) {
        return 1; /* ไม่มีอะไรต้องทำ */
    }
    if (v->size == 0) {
        free(v->data);
        v->data = NULL;
        v->capacity = 0;
        return 1;
    }
    int *tmp = realloc(v->data, v->size * sizeof *tmp);
    if (tmp == NULL) {
        return 0; /* realloc ล้มเหลว v->data เดิมยังใช้ได้ */
    }
    v->data = tmp;
    v->capacity = v->size;
    return 1;
}

void iv_free(IntVector *v) {
    free(v->data);
    v->data = NULL;
    v->size = 0;
    v->capacity = 0;
}

int main(void) {
    IntVector v;
    iv_init(&v);
    for (int i = 0; i < 5; i++) {
        iv_push_back(&v, i * 10);
    }
    printf("ก่อน shrink: size=%zu capacity=%zu\n", v.size, v.capacity);

    if (!iv_shrink_to_fit(&v)) {
        fprintf(stderr, "shrink_to_fit ล้มเหลว\n");
    }
    printf("หลัง shrink: size=%zu capacity=%zu\n", v.size, v.capacity);

    for (size_t i = 0; i < v.size; i++) {
        printf("%d ", v.data[i]);
    }
    printf("\n");

    iv_free(&v);
    return 0;
}
```

ผลลัพธ์:

```
ก่อน shrink: size=5 capacity=8
หลัง shrink: size=5 capacity=5
0 10 20 30 40
```

(ข้อ 3-6 ให้ผู้เรียนลองทำเองก่อน แนวทางคร่าวๆ ของข้อ 3 คือเพิ่ม `free(buf);` ก่อน
`return;` ทุกจุดที่ฟังก์ชันจะจบการทำงานหลังจาก `malloc` สำเร็จแล้ว — ลองเทียบกับแนวทาง
ในหัวข้อ 11.5 เรื่อง "1 malloc ต้องมี 1 free")

---

## สรุปท้ายบท

ใน Part นี้เราได้:

- เข้าใจความแตกต่างระหว่าง Stack และ Heap และรู้ว่าทำไม Heap ถึงจำเป็นสำหรับข้อมูลที่ต้อง
  มีอายุยืนกว่าฟังก์ชันที่สร้างมันขึ้นมา
- ใช้ `malloc`, `calloc`, `realloc` ได้อย่างถูกต้อง พร้อมตรวจสอบ `NULL` ทุกครั้ง
- เข้าใจว่า Memory Leak, Dangling Pointer และ Double Free คืออะไร เกิดจากอะไร และป้องกัน
  อย่างไร (โดยเฉพาะนิสัย "ตั้ง pointer เป็น `NULL` ทันทีหลัง `free`")
- เขียน Resizable Dynamic Array (`IntVector`) ตั้งแต่ต้นจนจบ ซึ่งเป็นรากฐานสำคัญของ
  โครงสร้างข้อมูลเกือบทุกตัวที่จะเรียนใน Module B ต่อจากนี้
- เข้าใจแนวคิด Ownership และรู้วิธีเขียน comment เป็นสัญญาที่ชัดเจนเพื่อป้องกัน Memory Leak
  และ Double Free ในโค้ดที่มีหลายฟังก์ชันเกี่ยวข้องกับหน่วยความจำเดียวกัน

ความรู้เรื่อง Dynamic Memory นี้จะถูกใช้ทันทีใน **Part 12** ซึ่งเราจะนำ `malloc` ไปสร้าง
**Array หลายมิติแบบ Dynamic** (Dynamic 2D Array) เพื่อทำงานกับข้อมูลแบบตาราง/เมทริกซ์
ที่ขนาดไม่ทราบล่วงหน้าตอนคอมไพล์

**ต่อไป:** [Part 12 — Array หลายมิติและ Pointer-to-Pointer](./part-012-multidim-arrays-pointers.md)
