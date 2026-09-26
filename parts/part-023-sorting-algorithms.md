# Part 23: Sorting Algorithm (Step 177–184)

> Module B — C ระดับกลาง & โครงสร้างข้อมูล/อัลกอริทึม | Part 23 จาก 125
> Step ที่ครอบคลุมใน Part นี้: Step 177–184
> Part ก่อนหน้า: [Part 22 — Hash Table ด้วย C](./part-022-hash-tables.md) | Part ถัดไป: [Part 24 — Searching Algorithm และ Big-O](./part-024-searching-bigo.md)

## เป้าหมายการเรียนรู้ (Learning Objectives)

หลังจบ Part นี้ ผู้เรียนจะสามารถ:

1. อธิบายได้ว่า "การเรียงลำดับ (Sorting)" คืออะไร และทำไมจึงเป็นหนึ่งในปัญหาพื้นฐานที่สำคัญ
   ที่สุดของวิชาวิทยาการคอมพิวเตอร์
2. เขียน Bubble Sort, Selection Sort, Insertion Sort, Merge Sort และ Quick Sort ด้วยภาษา C
   ได้เองตั้งแต่ต้นจนจบ พร้อมอธิบายกลไกการทำงานทีละขั้นตอน
3. Trace (ไล่ดู) การทำงานของแต่ละอัลกอริทึมบนอาร์เรย์ตัวอย่างเดียวกันได้ด้วยมือ
4. อธิบายความหมายของ "Stability" ในการเรียงลำดับ และบอกได้ว่าอัลกอริทึมใดเสถียร (Stable)
   หรือไม่เสถียร (Unstable)
5. เปรียบเทียบ Time Complexity (Best/Average/Worst Case) และ Space Complexity ของอัลกอริทึม
   การเรียงลำดับทั้ง 5 ตัวได้อย่างถูกต้อง
6. เลือกใช้อัลกอริทึมการเรียงลำดับที่เหมาะสมกับสถานการณ์จริงในงานเขียนโปรแกรม
7. ใช้ `qsort()` จาก `<stdlib.h>` พร้อมเขียนฟังก์ชัน Comparator ของตัวเองเพื่อเรียงข้อมูล
   ชนิดใดก็ได้ รวมถึงข้อมูลชนิด `struct`

---

## 23.1 การเรียงลำดับคืออะไร และทำไมสำคัญ (Step 177)

**Sorting** คือกระบวนการจัดเรียงข้อมูลในคอลเลกชัน (อาร์เรย์, Linked List ฯลฯ) ให้อยู่ในลำดับ
ที่กำหนดไว้ล่วงหน้า เช่นจากน้อยไปมาก (Ascending) หรือจากมากไปน้อย (Descending)

ฟังดูเป็นปัญหาง่ายๆ แต่ Sorting คือหนึ่งในหัวข้อที่ถูกศึกษามากที่สุดในวิชา Computer Science
เพราะเหตุผลอย่างน้อย 3 ข้อ:

- **เป็นพื้นฐานของอัลกอริทึมอื่นอีกมากมาย**: Binary Search (Part 24) ต้องการข้อมูลที่เรียงแล้ว
  เท่านั้น, อัลกอริทึม Graph บางตัวต้องเรียง Edge ตามน้ำหนักก่อน (Kruskal's Algorithm), การหา
  Median หรือ Percentile ก็ง่ายขึ้นมากถ้าข้อมูลถูกเรียงไว้แล้ว
- **เกิดขึ้นทุกที่ในซอฟต์แวร์จริง**: เรียงรายชื่อผู้ใช้ตามตัวอักษร, เรียงสินค้าตามราคา, เรียง
  ผลการค้นหาตามความเกี่ยวข้อง, เรียง Log ตามเวลา
- **เป็นกรณีศึกษาชั้นเยี่ยมสำหรับการวิเคราะห์ความซับซ้อน**: อัลกอริทึมเรียงลำดับที่ต่างกัน
  แสดงให้เห็น Trade-off ระหว่างความเร็ว หน่วยความจำ และความเรียบง่าย ได้ชัดเจนมาก ซึ่งเป็น
  ทักษะที่ใช้วิเคราะห์ปัญหาอื่นๆ ได้ตลอดชีวิตการเขียนโปรแกรม

ใน Part นี้เราจะเขียนอัลกอริทึมเรียงลำดับ 5 ตัวที่เป็น "หัวใจ" ของวิชานี้ตั้งแต่ต้นจนจบ แบ่งเป็น
2 กลุ่มตามแนวคิด:

| กลุ่ม | อัลกอริทึม | แนวคิดหลัก |
|---|---|---|
| **Comparison-based แบบง่าย (O(n²))** | Bubble, Selection, Insertion Sort | เปรียบเทียบและสลับทีละคู่ ง่ายต่อการเข้าใจ |
| **Divide and Conquer (O(n log n))** | Merge Sort, Quick Sort | แบ่งปัญหาใหญ่เป็นปัญหาย่อย แก้แล้วรวมกลับ |

### ตัวอย่างอาร์เรย์มาตรฐานที่จะใช้ตลอด Part นี้

เพื่อให้เปรียบเทียบกลไกของแต่ละอัลกอริทึมได้ง่าย เราจะใช้อาร์เรย์ตัวอย่างเดียวกันตลอดทั้งบท:

```
เริ่มต้น: [8, 3, 5, 4, 9, 1]
เป้าหมาย: [1, 3, 4, 5, 8, 9]   (เรียงจากน้อยไปมาก — Ascending)
```

ก่อนไปเขียนโค้ด เรามาทำความเข้าใจแนวคิด **Stability** ให้ชัดก่อน เพราะเป็นแนวคิดที่นักเรียน
ส่วนใหญ่มองข้าม แต่สำคัญมากในงานจริง

### Stability คืออะไร

อัลกอริทึมการเรียงลำดับจะเรียกว่า **Stable (เสถียร)** ถ้าองค์ประกอบสองตัวที่มี "คีย์" การเรียง
เท่ากัน ยังคงอยู่ใน **ลำดับสัมพัทธ์เดิม** หลังการเรียงเสร็จ

ตัวอย่างที่ทำให้เห็นภาพชัด: สมมติมีรายชื่อนักเรียนพร้อมคะแนน เรียงตามชื่อ (A-Z) ไว้แล้ว:

```
[("Anan", 80), ("Bee", 75), ("Cherry", 80), ("Dan", 75)]
```

ถ้าเรียงข้อมูลนี้ใหม่ตาม **คะแนน** ด้วยอัลกอริทึมที่ Stable ผลลัพธ์จะเป็น:

```
[("Bee", 75), ("Dan", 75), ("Anan", 80), ("Cherry", 80)]
```

สังเกตว่า "Bee" ยังมาก่อน "Dan" (เพราะเดิม Bee มาก่อน Dan ในกลุ่มคะแนน 75 เท่ากัน) และ
"Anan" ยังมาก่อน "Cherry" เช่นกัน — ลำดับ A-Z เดิมของกลุ่มที่คะแนนเท่ากันถูกรักษาไว้

ถ้าอัลกอริทึมนั้น **Unstable** ลำดับสัมพัทธ์นี้อาจถูกทำลาย เช่นอาจได้
`[("Dan", 75), ("Bee", 75), ...]` แทน ซึ่งดูเหมือนไม่มีปัญหาเวลาข้อมูลเป็นแค่ตัวเลข แต่ถ้า
ข้อมูลเป็น record ที่มีหลาย field และผู้ใช้ต้องการเรียงแบบ **Multi-key** (เช่น เรียงตามคะแนนก่อน
แล้วถ้าคะแนนเท่ากันให้เรียงตามชื่อ) การใช้อัลกอริทึม Stable เรียงทีละ key จากขวาไปซ้ายจะทำให้
ได้ผลลัพธ์ที่ถูกต้องโดยไม่ต้องเขียน Comparator ที่ซับซ้อน — นี่คือเหตุผลที่ Stability สำคัญ
มากในทางปฏิบัติ (เช่น `sort()` ของ Python คือ Timsort ซึ่ง Stable โดยตั้งใจ)

เราจะกลับมาสรุปว่าอัลกอริทึมไหน Stable บ้างในหัวข้อ 23.7

---

## 23.2 Bubble Sort (Step 178)

**แนวคิด**: เปรียบเทียบสมาชิกที่อยู่ติดกันทีละคู่ ถ้าลำดับผิด (ตัวซ้ายมากกว่าตัวขวา) ให้สลับกัน
ทำแบบนี้วนไปเรื่อยๆ จนครบอาร์เรย์ — ในแต่ละรอบ ค่าที่มากที่สุดที่เหลืออยู่จะถูก "ดันขึ้น (bubble
up)" ไปอยู่ท้ายอาร์เรย์เสมอ เหมือนฟองอากาศลอยขึ้นผิวน้ำ จึงเป็นที่มาของชื่อ

### โค้ดสมบูรณ์

```c
/* ============================================================
 * ชื่อไฟล์:     bubble_sort.c
 * คำอธิบาย:     สาธิต Bubble Sort พร้อม Optimization ตรวจจับว่าเรียงเสร็จแล้ว
 * ============================================================ */

/* ---------- 1. Header Includes ---------- */
#include <stdio.h>
#include <stddef.h>

/* ---------- 2. Macro / Constant Definitions ---------- */
#define ARRAY_SIZE 6

/* ---------- 3. Function Prototypes ---------- */
void bubble_sort(int arr[], size_t n);
void print_array(const int arr[], size_t n);

/* ---------- 4. main() ---------- */
int main(void) {
    int data[ARRAY_SIZE] = {8, 3, 5, 4, 9, 1};

    printf("ก่อนเรียง: ");
    print_array(data, ARRAY_SIZE);

    bubble_sort(data, ARRAY_SIZE);

    printf("หลังเรียง: ");
    print_array(data, ARRAY_SIZE);

    return 0;
}

/* ---------- 5. Function Implementations ---------- */
void bubble_sort(int arr[], size_t n) {
    if (n < 2) {
        return;
    }

    for (size_t i = 0; i < n - 1; i++) {
        int swapped = 0;

        for (size_t j = 0; j < n - 1 - i; j++) {
            if (arr[j] > arr[j + 1]) {
                int temp = arr[j];
                arr[j] = arr[j + 1];
                arr[j + 1] = temp;
                swapped = 1;
            }
        }

        /* ถ้ารอบนี้ไม่มีการสลับเลย แปลว่าเรียงเสร็จแล้ว หยุดได้ทันที */
        if (!swapped) {
            break;
        }
    }
}

void print_array(const int arr[], size_t n) {
    for (size_t i = 0; i < n; i++) {
        printf("%d ", arr[i]);
    }
    printf("\n");
}
```

คอมไพล์และรัน:

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 bubble_sort.c -o bubble_sort
./bubble_sort
```

```
ก่อนเรียง: 8 3 5 4 9 1
หลังเรียง: 1 3 4 5 8 9
```

### Trace ทีละรอบบนอาร์เรย์ตัวอย่าง

```
เริ่มต้น:            8 3 5 4 9 1

รอบที่ 1 (i=0):
  เทียบ 8,3 -> สลับ    3 8 5 4 9 1
  เทียบ 8,5 -> สลับ    3 5 8 4 9 1
  เทียบ 8,4 -> สลับ    3 5 4 8 9 1
  เทียบ 8,9 -> ไม่สลับ 3 5 4 8 9 1
  เทียบ 9,1 -> สลับ    3 5 4 8 1 9   <- 9 (ค่ามากสุด) ลอยไปท้ายสุดแล้ว

รอบที่ 2 (i=1):
  เทียบ 3,5 / 5,4->สลับ / 5,8 / 8,1->สลับ
  ผลลัพธ์:             3 4 5 1 8 9

รอบที่ 3 (i=2):
  เทียบ 3,4 / 4,5 / 5,1->สลับ
  ผลลัพธ์:             3 4 1 5 8 9

รอบที่ 4 (i=3):
  เทียบ 3,4 / 4,1->สลับ
  ผลลัพธ์:             3 1 4 5 8 9

รอบที่ 5 (i=4):
  เทียบ 3,1->สลับ
  ผลลัพธ์:             1 3 4 5 8 9   <- เรียงเสร็จสมบูรณ์
```

สำหรับอาร์เรย์ 6 ตัว Bubble Sort ใช้ทั้งหมด **5 รอบ (n-1 รอบ)** ในกรณีเลวร้ายที่สุด แต่ละรอบ
จะเปรียบเทียบน้อยลงเรื่อยๆ เพราะส่วนท้ายที่เรียงเสร็จแล้วไม่ต้องแตะอีก

### Stability ของ Bubble Sort

Bubble Sort เป็น **Stable** เพราะเราสลับก็ต่อเมื่อ `arr[j] > arr[j+1]` (ใช้ `>` ไม่ใช่ `>=`)
ถ้าสองค่าเท่ากัน จะไม่มีการสลับ จึงคงลำดับสัมพัทธ์เดิมไว้เสมอ

---

## 23.3 Selection Sort (Step 179)

**แนวคิด**: ในแต่ละรอบ ค้นหาค่า **น้อยที่สุด** จากส่วนที่ยังไม่ได้เรียง แล้วสลับให้มาอยู่ตำแหน่ง
แรกสุดของส่วนที่ยังไม่เรียงนั้น ทำซ้ำไปเรื่อยๆ จนครบทุกตำแหน่ง — ต่างจาก Bubble Sort ตรงที่
Selection Sort **สลับน้อยครั้งกว่ามาก** (สลับสูงสุดแค่ n-1 ครั้งทั้งหมด) แต่ยังคงต้อง
**เปรียบเทียบ** ทุกคู่เหมือนเดิม

### โค้ดสมบูรณ์

```c
/* ============================================================
 * ชื่อไฟล์:     selection_sort.c
 * คำอธิบาย:     สาธิต Selection Sort — หา min แล้วสลับมาไว้หน้าสุด
 * ============================================================ */
#include <stdio.h>
#include <stddef.h>

#define ARRAY_SIZE 6

void selection_sort(int arr[], size_t n);
void print_array(const int arr[], size_t n);

int main(void) {
    int data[ARRAY_SIZE] = {8, 3, 5, 4, 9, 1};

    printf("ก่อนเรียง: ");
    print_array(data, ARRAY_SIZE);

    selection_sort(data, ARRAY_SIZE);

    printf("หลังเรียง: ");
    print_array(data, ARRAY_SIZE);

    return 0;
}

void selection_sort(int arr[], size_t n) {
    if (n < 2) {
        return;
    }

    for (size_t i = 0; i < n - 1; i++) {
        size_t min_idx = i;

        /* หาตำแหน่งของค่าน้อยที่สุดในส่วนที่ยังไม่เรียง [i..n-1] */
        for (size_t j = i + 1; j < n; j++) {
            if (arr[j] < arr[min_idx]) {
                min_idx = j;
            }
        }

        if (min_idx != i) {
            int temp = arr[i];
            arr[i] = arr[min_idx];
            arr[min_idx] = temp;
        }
    }
}

void print_array(const int arr[], size_t n) {
    for (size_t i = 0; i < n; i++) {
        printf("%d ", arr[i]);
    }
    printf("\n");
}
```

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 selection_sort.c -o selection_sort
./selection_sort
```

```
ก่อนเรียง: 8 3 5 4 9 1
หลังเรียง: 1 3 4 5 8 9
```

### Trace ทีละรอบ

```
เริ่มต้น:                8 3 5 4 9 1

i=0: ค่าน้อยสุดใน [0..5] คือ 1 (index 5) -> สลับ arr[0],arr[5]
     ผลลัพธ์:             1 3 5 4 9 8

i=1: ค่าน้อยสุดใน [1..5] คือ 3 (index 1, อยู่ตำแหน่งถูกแล้ว) -> ไม่สลับ
     ผลลัพธ์:             1 3 5 4 9 8

i=2: ค่าน้อยสุดใน [2..5] คือ 4 (index 3) -> สลับ arr[2],arr[3]
     ผลลัพธ์:             1 3 4 5 9 8

i=3: ค่าน้อยสุดใน [3..5] คือ 5 (index 3, อยู่ตำแหน่งถูกแล้ว) -> ไม่สลับ
     ผลลัพธ์:             1 3 4 5 9 8

i=4: ค่าน้อยสุดใน [4..5] คือ 8 (index 5) -> สลับ arr[4],arr[5]
     ผลลัพธ์:             1 3 4 5 8 9   <- เรียงเสร็จสมบูรณ์
```

สังเกตว่า Selection Sort สลับข้อมูลจริงแค่ **3 ครั้ง** เท่านั้น (ที่ i=0, i=2, i=4) ในขณะที่
Bubble Sort สลับไปแล้วหลายครั้งกว่านั้นมากในตัวอย่างเดียวกัน — นี่คือข้อดีของ Selection Sort
เวลาการ "เขียน (write)" ข้อมูลมีต้นทุนสูง (เช่นเขียนลง EEPROM/Flash Memory ที่มีอายุการเขียน
จำกัด) เพราะจำนวนครั้งของการเขียนจะเป็น O(n) เสมอไม่ว่ากรณีใด

### Stability ของ Selection Sort

Selection Sort เป็น **Unstable** โดยธรรมชาติของอัลกอริทึม เพราะการสลับตำแหน่งไกลๆ กัน
(swap แบบกระโดดข้าม ไม่ใช่สลับติดกัน) อาจทำให้ค่าที่เท่ากันสลับลำดับสัมพัทธ์กัน ตัวอย่างเช่น
อาร์เรย์ `[5a, 3, 5b]` (5a และ 5b คือค่า 5 สองตัวที่มาจากคนละที่) เมื่อหา min ของรอบแรกเจอ 3
ที่ index 1 แล้วสลับกับ index 0 จะได้ `[3, 5a, 5b]` — กรณีนี้ยังคงลำดับ 5a ก่อน 5b ไว้ได้
แต่ถ้าตัวอย่างซับซ้อนกว่านี้ (เช่นมี 3 ค่าเท่ากันปนกัน) การสลับข้ามตำแหน่งจะทำให้ลำดับสัมพัทธ์
เสียได้ง่าย จึงถือว่า Selection Sort เป็น Unstable โดย default (แม้จะปรับให้ Stable ได้ด้วย
เทคนิคพิเศษ แต่จะซับซ้อนขึ้นและเสียข้อดีเรื่องจำนวนการเขียนน้อยไป)

---

## 23.4 Insertion Sort (Step 180)

**แนวคิด**: เลียนแบบวิธีที่คนเรียงไพ่บนมือ — หยิบไพ่ทีละใบจากกอง แล้วแทรก (insert) เข้าไปใน
ตำแหน่งที่ถูกต้องของไพ่ที่ถืออยู่ในมือ (ซึ่งเรียงอยู่แล้ว) ทำแบบนี้ไปเรื่อยๆ จนหยิบไพ่ครบทุกใบ

### โค้ดสมบูรณ์

```c
/* ============================================================
 * ชื่อไฟล์:     insertion_sort.c
 * คำอธิบาย:     สาธิต Insertion Sort — แทรกค่าใหม่เข้าตำแหน่งที่ถูกต้องในส่วนที่เรียงแล้ว
 * ============================================================ */
#include <stdio.h>
#include <stddef.h>

#define ARRAY_SIZE 6

void insertion_sort(int arr[], size_t n);
void print_array(const int arr[], size_t n);

int main(void) {
    int data[ARRAY_SIZE] = {8, 3, 5, 4, 9, 1};

    printf("ก่อนเรียง: ");
    print_array(data, ARRAY_SIZE);

    insertion_sort(data, ARRAY_SIZE);

    printf("หลังเรียง: ");
    print_array(data, ARRAY_SIZE);

    return 0;
}

void insertion_sort(int arr[], size_t n) {
    for (size_t i = 1; i < n; i++) {
        int key = arr[i];
        size_t j = i;

        /* เลื่อนสมาชิกที่มากกว่า key ไปทางขวาทีละตัว เพื่อเปิดช่องว่างให้ key แทรก */
        while (j > 0 && arr[j - 1] > key) {
            arr[j] = arr[j - 1];
            j--;
        }
        arr[j] = key;
    }
}

void print_array(const int arr[], size_t n) {
    for (size_t i = 0; i < n; i++) {
        printf("%d ", arr[i]);
    }
    printf("\n");
}
```

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 insertion_sort.c -o insertion_sort
./insertion_sort
```

```
ก่อนเรียง: 8 3 5 4 9 1
หลังเรียง: 1 3 4 5 8 9
```

> **หมายเหตุเรื่อง `while (j > 0 && arr[j - 1] > key)`**: C ใช้ **Short-circuit Evaluation**
> เสมอ ถ้า `j > 0` เป็นเท็จ นิพจน์ `arr[j - 1]` จะไม่ถูกประเมินเลย จึงปลอดภัยแม้ `j` เป็น
> `size_t` (unsigned) เพราะเราไม่มีทางเข้าไปคำนวณ `j - 1` ตอนที่ `j == 0`

### Trace ทีละรอบ ("มือที่ถือไพ่ที่เรียงแล้ว" คือส่วนซ้ายของ `|`)

```
เริ่มต้น:                 8 | 3 5 4 9 1

i=1, key=3: 8>3 เลื่อน 8 ไปขวา, แทรก 3 ที่ index 0
     ผลลัพธ์:            3 8 | 5 4 9 1

i=2, key=5: 8>5 เลื่อน, 3<5 หยุด, แทรก 5 ที่ index 1
     ผลลัพธ์:            3 5 8 | 4 9 1

i=3, key=4: 8>4 เลื่อน, 5>4 เลื่อน, 3<4 หยุด, แทรก 4 ที่ index 1
     ผลลัพธ์:            3 4 5 8 | 9 1

i=4, key=9: 8<9 หยุดทันที ไม่ต้องเลื่อนเลย แทรก 9 ที่ index 4 (ตำแหน่งเดิม)
     ผลลัพธ์:            3 4 5 8 9 | 1

i=5, key=1: 9,8,5,4,3 มากกว่า 1 ทั้งหมด เลื่อนหมด แทรก 1 ที่ index 0
     ผลลัพธ์:            1 3 4 5 8 9 |   <- เรียงเสร็จสมบูรณ์
```

สังเกตจุดสำคัญ: ที่ `i=4` (key=9) อัลกอริทึมหยุดทันทีโดยไม่ต้องเลื่อนอะไรเลย เพราะ 9 มากกว่า
สมาชิกทุกตัวที่เรียงไว้แล้วอยู่แล้ว — นี่คือเหตุผลที่ Insertion Sort **เร็วมากเมื่อข้อมูล
ใกล้เรียงอยู่แล้ว (Nearly Sorted)** โดย Best Case คือ O(n) เท่านั้น (เทียบกับ Bubble/Selection
ที่ยังคงวน O(n²) แม้ข้อมูลจะเรียงอยู่แล้วก็ตาม ยกเว้น Bubble Sort ที่ใส่ optimization ตรวจจับ
`swapped` ไว้แล้วเช่นในโค้ดข้างต้น)

### Stability ของ Insertion Sort

Insertion Sort เป็น **Stable** เพราะเงื่อนไขการเลื่อนคือ `arr[j-1] > key` (ใช้ `>` ไม่ใช่
`>=`) หมายความว่าถ้าค่าเท่ากัน เราจะหยุดแทรกทันทีโดยไม่เลื่อนสมาชิกตัวเดิมออกไป ทำให้สมาชิก
ที่เท่ากันคงลำดับการมาก่อน-หลังเดิมไว้

---

## 23.5 Merge Sort (Step 181)

Bubble, Selection, Insertion Sort ทั้ง 3 ตัวข้างต้นล้วนเป็นอัลกอริทึม **O(n²)** ซึ่งช้าลงอย่าง
รวดเร็วเมื่อข้อมูลมีขนาดใหญ่ (อาร์เรย์ล้านตัวจะใช้เวลาประมาณ 10¹² operation!) ตอนนี้เราจะเข้าสู่
กลุ่มอัลกอริทึมที่เร็วกว่ามาก โดยใช้แนวคิด **Divide and Conquer (แบ่งแยกและเอาชนะ)**

**แนวคิดของ Merge Sort**:

1. **Divide**: แบ่งอาร์เรย์ออกเป็นสองครึ่งซ้าย-ขวา
2. **Conquer**: เรียกตัวเองแบบ Recursive เพื่อเรียงแต่ละครึ่งย่อยจนเหลือ 1 ตัว (ซึ่งถือว่า
   เรียงแล้วโดยอัตโนมัติ)
3. **Combine**: ผสาน (Merge) สองครึ่งที่เรียงแล้วเข้าด้วยกันให้กลายเป็นอาร์เรย์เดียวที่เรียง
   สมบูรณ์

ขั้นตอนที่ยากและสำคัญที่สุดคือ "Merge" — การผสานสองอาร์เรย์ย่อยที่เรียงแล้วให้กลายเป็นอาร์เรย์
เดียวที่เรียง ทำได้โดยเทียบสมาชิกหัวแถวของทั้งสองฝั่งทีละคู่ แล้วหยิบตัวที่น้อยกว่าออกมาก่อนเสมอ

### โค้ดสมบูรณ์

```c
/* ============================================================
 * ชื่อไฟล์:     merge_sort.c
 * คำอธิบาย:     สาธิต Merge Sort แบบ Divide and Conquer
 * ============================================================ */
#include <stdio.h>
#include <stdlib.h>
#include <stddef.h>

#define ARRAY_SIZE 6

void merge_sort(int arr[], size_t left, size_t right);
void merge(int arr[], size_t left, size_t mid, size_t right);
void print_array(const int arr[], size_t n);

int main(void) {
    int data[ARRAY_SIZE] = {8, 3, 5, 4, 9, 1};

    printf("ก่อนเรียง: ");
    print_array(data, ARRAY_SIZE);

    merge_sort(data, 0, ARRAY_SIZE - 1);

    printf("หลังเรียง: ");
    print_array(data, ARRAY_SIZE);

    return 0;
}

void merge_sort(int arr[], size_t left, size_t right) {
    if (left >= right) {
        return; /* ฐาน: อาร์เรย์เหลือ 0 หรือ 1 ตัว ถือว่าเรียงแล้ว */
    }

    size_t mid = left + (right - left) / 2;

    merge_sort(arr, left, mid);        /* เรียงครึ่งซ้าย */
    merge_sort(arr, mid + 1, right);   /* เรียงครึ่งขวา */
    merge(arr, left, mid, right);      /* ผสานทั้งสองครึ่งเข้าด้วยกัน */
}

void merge(int arr[], size_t left, size_t mid, size_t right) {
    size_t n1 = mid - left + 1;
    size_t n2 = right - mid;

    int *left_arr = malloc(n1 * sizeof(int));
    int *right_arr = malloc(n2 * sizeof(int));

    if (left_arr == NULL || right_arr == NULL) {
        fprintf(stderr, "จองหน่วยความจำไม่สำเร็จ\n");
        free(left_arr);
        free(right_arr);
        exit(EXIT_FAILURE);
    }

    for (size_t i = 0; i < n1; i++) {
        left_arr[i] = arr[left + i];
    }
    for (size_t j = 0; j < n2; j++) {
        right_arr[j] = arr[mid + 1 + j];
    }

    size_t i = 0, j = 0, k = left;

    /* เทียบหัวแถวของสองฝั่ง หยิบตัวที่น้อยกว่าออกมาก่อนเสมอ */
    while (i < n1 && j < n2) {
        if (left_arr[i] <= right_arr[j]) {
            arr[k] = left_arr[i];
            i++;
        } else {
            arr[k] = right_arr[j];
            j++;
        }
        k++;
    }

    /* คัดลอกสมาชิกที่เหลือ (ถ้ามี) ของแต่ละฝั่ง */
    while (i < n1) {
        arr[k] = left_arr[i];
        i++;
        k++;
    }
    while (j < n2) {
        arr[k] = right_arr[j];
        j++;
        k++;
    }

    free(left_arr);
    free(right_arr);
}

void print_array(const int arr[], size_t n) {
    for (size_t i = 0; i < n; i++) {
        printf("%d ", arr[i]);
    }
    printf("\n");
}
```

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 merge_sort.c -o merge_sort
./merge_sort
```

```
ก่อนเรียง: 8 3 5 4 9 1
หลังเรียง: 1 3 4 5 8 9
```

> **สังเกตเงื่อนไข `left_arr[i] <= right_arr[j]`**: การใช้ `<=` (ไม่ใช่ `<`) ตรงนี้สำคัญมาก
> เพราะถ้าสองค่าเท่ากัน เราให้ความสำคัญกับฝั่งซ้าย (`left_arr`) ก่อนเสมอ ซึ่งฝั่งซ้ายคือส่วน
> ที่อยู่ก่อนในอาร์เรย์ต้นฉบับ — นี่คือกลไกที่ทำให้ Merge Sort เป็น **Stable**

### Trace การแบ่งและผสาน (Recursion Tree)

```
                         [8,3,5,4,9,1]
                        /             \
                 [8,3,5]               [4,9,1]
                /       \              /       \
             [8]      [3,5]         [4]      [9,1]
                      /    \                  /    \
                   [3]    [5]              [9]    [1]

--- ขั้น Merge (ไล่จากล่างขึ้นบน) ---

merge([3],[5])       -> [3,5]
merge([8],[3,5])     -> [3,5,8]     (เทียบ 8,3->3 | 8,5->5 | เหลือ 8)
merge([9],[1])       -> [1,9]       (เทียบ 9,1->1 | เหลือ 9)
merge([4],[1,9])     -> [1,4,9]     (เทียบ 4,1->1 | 4,9->4 | เหลือ 9)
merge([3,5,8],[1,4,9]) -> [1,3,4,5,8,9]
    เทียบ 3,1 -> 1
    เทียบ 3,4 -> 3
    เทียบ 5,4 -> 4
    เทียบ 5,9 -> 5
    เทียบ 8,9 -> 8
    เหลือ    -> 9
    ผลลัพธ์สุดท้าย: [1,3,4,5,8,9]
```

Merge Sort แบ่งปัญหาลง log₂(n) ระดับเสมอ (สำหรับ n=6 คือประมาณ 3 ระดับ) และในแต่ละระดับ
ใช้เวลารวม O(n) สำหรับการ merge ทั้งหมด — จึงได้ความซับซ้อนรวมเป็น **O(n log n)**

### Stability ของ Merge Sort

Merge Sort เป็น **Stable** ตามที่อธิบายไว้ข้างต้น (ใช้ `<=` เลือกฝั่งซ้ายก่อนเมื่อค่าเท่ากัน)

---

## 23.6 Quick Sort (Step 182)

**แนวคิด**: เลือกสมาชิกตัวหนึ่งเป็น **Pivot** แล้ว "จัดกลุ่ม (Partition)" สมาชิกที่เหลือทั้งหมด
ออกเป็น 2 กลุ่ม: กลุ่มที่น้อยกว่า Pivot อยู่ทางซ้าย และกลุ่มที่มากกว่า Pivot อยู่ทางขวา จากนั้น
เรียก Quick Sort แบบ Recursive กับสองกลุ่มย่อยนั้นต่อไป — ต่างจาก Merge Sort ตรงที่ Quick Sort
ทำงานหนักตอน **Partition** (ก่อนแบ่ง) ในขณะที่ Merge Sort ทำงานหนักตอน **Merge** (หลังแบ่ง)
และ Quick Sort เรียงแบบ **In-place** ไม่ต้องใช้อาร์เรย์เสริมเหมือน Merge Sort

เทคนิค Partition ที่นิยมและเข้าใจง่ายที่สุดคือ **Lomuto Partition Scheme** ซึ่งเลือกสมาชิก
**ตัวสุดท้าย** ของช่วงเป็น Pivot เสมอ

### โค้ดสมบูรณ์

```c
/* ============================================================
 * ชื่อไฟล์:     quick_sort.c
 * คำอธิบาย:     สาธิต Quick Sort ด้วย Lomuto Partition Scheme
 * ============================================================ */
#include <stdio.h>
#include <stddef.h>

#define ARRAY_SIZE 6

void quick_sort(int arr[], long low, long high);
long partition(int arr[], long low, long high);
void swap_int(int *a, int *b);
void print_array(const int arr[], size_t n);

int main(void) {
    int data[ARRAY_SIZE] = {8, 3, 5, 4, 9, 1};

    printf("ก่อนเรียง: ");
    print_array(data, ARRAY_SIZE);

    quick_sort(data, 0, (long)ARRAY_SIZE - 1);

    printf("หลังเรียง: ");
    print_array(data, ARRAY_SIZE);

    return 0;
}

void quick_sort(int arr[], long low, long high) {
    if (low < high) {
        long pivot_idx = partition(arr, low, high);
        quick_sort(arr, low, pivot_idx - 1);   /* เรียงกลุ่มซ้ายของ pivot */
        quick_sort(arr, pivot_idx + 1, high);  /* เรียงกลุ่มขวาของ pivot */
    }
}

long partition(int arr[], long low, long high) {
    int pivot = arr[high]; /* เลือกตัวสุดท้ายของช่วงเป็น pivot เสมอ */
    long i = low - 1;      /* i คือขอบเขตของกลุ่ม "น้อยกว่า pivot" */

    for (long j = low; j < high; j++) {
        if (arr[j] < pivot) {
            i++;
            swap_int(&arr[i], &arr[j]);
        }
    }

    /* ย้าย pivot มาไว้ตรงกลาง ระหว่างกลุ่มน้อยกว่าและกลุ่มมากกว่า */
    swap_int(&arr[i + 1], &arr[high]);
    return i + 1;
}

void swap_int(int *a, int *b) {
    int temp = *a;
    *a = *b;
    *b = temp;
}

void print_array(const int arr[], size_t n) {
    for (size_t i = 0; i < n; i++) {
        printf("%d ", arr[i]);
    }
    printf("\n");
}
```

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 quick_sort.c -o quick_sort
./quick_sort
```

```
ก่อนเรียง: 8 3 5 4 9 1
หลังเรียง: 1 3 4 5 8 9
```

> **ทำไมใช้ `long` แทน `size_t` สำหรับ index ใน Quick Sort?** เพราะระหว่างการ Partition
> ตัวแปร `i` เริ่มต้นที่ `low - 1` ซึ่งอาจติดลบได้เมื่อ `low == 0` ถ้าใช้ `size_t` (unsigned)
> การลบจนติดลบจะ wrap around กลายเป็นเลขบวกมหาศาลทันที (Undefined Behavior เชิงตรรกะที่ร้ายแรง)
> จึงต้องใช้ชนิดข้อมูลที่เป็น signed แทนสำหรับงานลักษณะนี้

### Trace การ Partition (Lomuto Scheme)

```
QuickSort(low=0, high=5) บน [8,3,5,4,9,1]
  pivot = arr[5] = 1
  i = -1
  j=0: arr[0]=8, ไม่ < 1, ข้าม
  j=1: arr[1]=3, ไม่ < 1, ข้าม
  j=2: arr[2]=5, ไม่ < 1, ข้าม
  j=3: arr[3]=4, ไม่ < 1, ข้าม
  j=4: arr[4]=9, ไม่ < 1, ข้าม
  สลับ arr[i+1]=arr[0] กับ arr[high]=arr[5]  ->  [1,3,5,4,9,8]
  pivot_idx = 0  (1 เข้าที่แล้ว: ทุกอย่างทางซ้ายไม่มี, ทุกอย่างทางขวามากกว่า 1 หมด)

  ซ้าย QuickSort(0,-1)  -> ว่างเปล่า ไม่ทำอะไร
  ขวา QuickSort(1,5) บน [3,5,4,9,8] (index 1..5)
      pivot = arr[5] = 8
      i = 0
      j=1: arr[1]=3 < 8 -> i=1, สลับ arr[1],arr[1] (ไม่เปลี่ยน)
      j=2: arr[2]=5 < 8 -> i=2, สลับ arr[2],arr[2] (ไม่เปลี่ยน)
      j=3: arr[3]=4 < 8 -> i=3, สลับ arr[3],arr[3] (ไม่เปลี่ยน)
      j=4: arr[4]=9, ไม่ < 8, ข้าม
      สลับ arr[i+1]=arr[4] กับ arr[high]=arr[5]  ->  [1,3,5,4,8,9]
      pivot_idx = 4

      ซ้าย QuickSort(1,3) บน [3,5,4] (index 1..3)
          pivot = arr[3] = 4
          i = 0
          j=1: arr[1]=3 < 4 -> i=1, สลับ arr[1],arr[1] (ไม่เปลี่ยน)
          j=2: arr[2]=5, ไม่ < 4, ข้าม
          สลับ arr[i+1]=arr[2] กับ arr[high]=arr[3]  ->  [1,3,4,5,8,9]
          pivot_idx = 2
          ซ้าย QuickSort(1,1) -> 1 ตัว, จบ
          ขวา QuickSort(3,3) -> 1 ตัว, จบ

      ขวา QuickSort(5,5) -> 1 ตัว, จบ

ผลลัพธ์สุดท้าย: [1,3,4,5,8,9]
```

### Stability ของ Quick Sort

Quick Sort (แบบมาตรฐานที่สลับตำแหน่งข้ามช่วงระหว่าง Partition) เป็น **Unstable** เพราะการ
สลับตำแหน่งของสมาชิกในขั้น Partition อาจย้ายสมาชิกที่มีค่าเท่ากันข้ามกันไปมาได้ ตัวอย่างเช่น
`[3a, 1, 3b, 2]` เมื่อ Partition ด้วย pivot = 2 (ตัวสุดท้าย) สมาชิก 3a และ 1 จะถูกเทียบกับ
pivot ก่อน 3b แต่กระบวนการสลับข้ามตำแหน่งอาจทำให้ 3b ย้ายไปอยู่ก่อน 3a ได้ในบางกรณี

---

## 23.7 ตารางเปรียบเทียบ Time/Space Complexity และ Stability (Step 183)

นี่คือตารางสรุปที่สำคัญที่สุดของ Part นี้ ควรจดจำให้ขึ้นใจ เพราะจะถูกถามในการสัมภาษณ์งาน
โปรแกรมเมอร์แทบทุกครั้งที่มีหัวข้อ Data Structures & Algorithms:

| อัลกอริทึม | Best Case | Average Case | Worst Case | Space | Stable? | In-place? |
|---|---|---|---|---|---|---|
| **Bubble Sort** | O(n) * | O(n²) | O(n²) | O(1) | ใช่ | ใช่ |
| **Selection Sort** | O(n²) | O(n²) | O(n²) | O(1) | ไม่ | ใช่ |
| **Insertion Sort** | O(n) | O(n²) | O(n²) | O(1) | ใช่ | ใช่ |
| **Merge Sort** | O(n log n) | O(n log n) | O(n log n) | O(n) | ใช่ | ไม่ |
| **Quick Sort** | O(n log n) | O(n log n) | O(n²) ** | O(log n) *** | ไม่ | ใช่ |

\* เฉพาะเมื่อใส่ optimization ตรวจจับ `swapped` แบบในโค้ดข้างต้น (ถ้าไม่ใส่ Best Case จะเป็น
O(n²) เหมือนกัน เพราะ loop ยังวนครบทุกคู่โดยไม่สนใจว่าสลับหรือไม่)

\*\* เกิดขึ้นเมื่อเลือก Pivot แล้วได้กลุ่มที่ไม่สมดุลซ้ำๆ กัน (เช่นข้อมูลเรียงอยู่แล้วและเลือก
Pivot เป็นตัวสุดท้ายเสมอแบบ Lomuto Scheme — จะทำให้ทุกรอบ Partition แบ่งได้กลุ่มเดียวขนาด n-1
กับอีกกลุ่มขนาด 0 กลายเป็นเหมือน Selection Sort ที่ช้าที่สุด)

\*\*\* Space ของ Quick Sort คือขนาดของ Call Stack จาก Recursion ไม่ใช่อาร์เรย์เสริม (เพราะ
Partition ทำแบบ in-place) ในกรณีเลวร้ายที่สุดที่แบ่งไม่สมดุล Stack อาจลึกถึง O(n)

### ทำไม Quick Sort ถึงชื่อว่า "Quick" ทั้งที่ Worst Case แย่กว่า Merge Sort

เพราะในทางปฏิบัติ (Average Case กับข้อมูลสุ่มทั่วไป) Quick Sort มัก **เร็วกว่า Merge Sort จริง**
แม้ทั้งคู่จะเป็น O(n log n) เหมือนกัน เหตุผลหลักคือ:

1. Quick Sort ทำงาน **In-place** ไม่ต้องจองหน่วยความจำเพิ่มและคัดลอกข้อมูลไปมา (Merge Sort
   ต้อง `malloc`/`free` ทุกครั้งที่ merge) จึงมี **Constant Factor** ที่เล็กกว่า
2. Quick Sort เข้าถึงหน่วยความจำแบบต่อเนื่อง (Cache-friendly) มากกว่า
3. ในทางปฏิบัติสามารถลด Worst Case ลงได้ด้วยการสุ่มเลือก Pivot (Randomized Quick Sort) หรือ
   เลือก Median-of-Three ทำให้โอกาสเจอ Worst Case ในข้อมูลจริงต่ำมาก

---

## 23.8 qsort() จาก stdlib.h และการเลือกใช้อัลกอริทึมในทางปฏิบัติ (Step 184)

ในงานจริงแทบไม่มีใครเขียนอัลกอริทึมเรียงลำดับเองอีกแล้ว (ยกเว้นเพื่อการศึกษา หรือกรณีพิเศษ
เฉพาะทางมากๆ) เพราะภาษา C มีฟังก์ชัน **`qsort()`** ให้ใช้งานได้ทันทีจาก `<stdlib.h>` ซึ่งใน
คอมไพเลอร์ส่วนใหญ่ implement ด้วยอัลกอริทึมแบบผสม (เช่น Introsort ที่รวม Quick Sort + Heap
Sort + Insertion Sort เข้าด้วยกัน เพื่อป้องกัน Worst Case ของ Quick Sort ล้วนๆ)

Signature ของ `qsort()`:

```c
void qsort(void *base, size_t nmemb, size_t size,
           int (*compar)(const void *, const void *));
```

- `base`: pointer ไปยังจุดเริ่มต้นของอาร์เรย์
- `nmemb`: จำนวนสมาชิกในอาร์เรย์
- `size`: ขนาด (byte) ของสมาชิกแต่ละตัว (ใช้ `sizeof(...)`)
- `compar`: **Comparator function** ที่เราต้องเขียนเอง — รับ pointer สองตัว (แบบ `void *`)
  แล้วคืนค่า:
  - ค่าน้อยกว่า 0 ถ้า element แรกควรมาก่อน
  - ค่า 0 ถ้าเท่ากัน
  - ค่ามากกว่า 0 ถ้า element แรกควรมาหลัง

### โค้ดสมบูรณ์: เรียงตัวเลข และเรียง struct ด้วย qsort()

```c
/* ============================================================
 * ชื่อไฟล์:     qsort_demo.c
 * คำอธิบาย:     สาธิตการใช้ qsort() จาก stdlib.h กับข้อมูลตัวเลขและ struct
 * ============================================================ */
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#define ARRAY_SIZE 6

typedef struct {
    char name[20];
    int score;
} Student;

int compare_int_asc(const void *a, const void *b);
int compare_student_by_score_desc(const void *a, const void *b);

int main(void) {
    int numbers[ARRAY_SIZE] = {8, 3, 5, 4, 9, 1};

    qsort(numbers, ARRAY_SIZE, sizeof(int), compare_int_asc);

    printf("ตัวเลขเรียงจากน้อยไปมาก: ");
    for (int i = 0; i < ARRAY_SIZE; i++) {
        printf("%d ", numbers[i]);
    }
    printf("\n");

    Student students[3] = {
        {"Somchai", 75},
        {"Suda", 92},
        {"Anan", 60}
    };

    qsort(students, 3, sizeof(Student), compare_student_by_score_desc);

    printf("นักเรียนเรียงตามคะแนนมาก->น้อย:\n");
    for (int i = 0; i < 3; i++) {
        printf("  %-10s %d\n", students[i].name, students[i].score);
    }

    return 0;
}

int compare_int_asc(const void *a, const void *b) {
    int x = *(const int *)a;
    int y = *(const int *)b;
    /* วิธีนี้ปลอดภัยกว่าการ return (x - y) ตรงๆ เพราะ (x - y) อาจ overflow ได้
       ถ้า x, y มีค่าห่างกันมาก (เช่น x = INT_MAX, y = INT_MIN) */
    return (x > y) - (x < y);
}

int compare_student_by_score_desc(const void *a, const void *b) {
    const Student *s1 = (const Student *)a;
    const Student *s2 = (const Student *)b;
    return s2->score - s1->score; /* เรียงจากมากไปน้อย: เอาของ s2 ลบ s1 */
}
```

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 qsort_demo.c -o qsort_demo
./qsort_demo
```

```
ตัวเลขเรียงจากน้อยไปมาก: 1 3 4 5 8 9
นักเรียนเรียงตามคะแนนมาก->น้อย:
  Suda       92
  Somchai    75
  Anan       60
```

> **ข้อควรระวังสำคัญ**: `qsort()` ของมาตรฐาน C **ไม่รับประกัน Stability** เพราะ Implementation
> ขึ้นกับแต่ละคอมไพเลอร์ ถ้าต้องการผลลัพธ์ที่ Stable แน่นอน (เช่นเรียง multi-key) ควรเขียน
> Comparator ให้เทียบทุก field ที่ต้องการในฟังก์ชันเดียว หรือเขียนอัลกอริทึม Merge Sort เองแทน

### ตารางเลือกใช้อัลกอริทึมในทางปฏิบัติ

| สถานการณ์ | อัลกอริทึมที่ควรเลือก | เหตุผล |
|---|---|---|
| งานทั่วไป ไม่มีข้อจำกัดพิเศษ | `qsort()` / library ที่มีให้ | ผ่านการทดสอบมาอย่างดี เร็ว และปลอดภัยกว่าเขียนเอง |
| ข้อมูลขนาดเล็กมาก (< 10-20 ตัว) | Insertion Sort | Overhead ของ Recursion ใน Merge/Quick Sort ทำให้ช้ากว่าจริงๆ ในทางปฏิบัติ (คอมไพเลอร์ standard library หลายตัวจึงสลับไปใช้ Insertion Sort เองเมื่อ partition เล็กพอ) |
| ข้อมูลที่ใกล้เรียงอยู่แล้ว (Nearly Sorted) | Insertion Sort | Best Case O(n) เร็วมากในกรณีนี้ |
| ต้องการ Stability (เรียง multi-key) | Merge Sort หรือ Stable sort ของภาษานั้น | รับประกันลำดับสัมพัทธ์ของค่าที่เท่ากัน |
| หน่วยความจำจำกัดมาก (Embedded System) | Quick Sort หรือ Heap Sort | เรียงแบบ in-place ไม่ต้องจองหน่วยความจำเสริม |
| ต้องการ Worst Case ที่รับประกันแน่นอน (Real-time System) | Merge Sort หรือ Heap Sort | ไม่มีความเสี่ยงตกไปเป็น O(n²) เหมือน Quick Sort ล้วนๆ |
| ข้อมูลมีจำนวนค่าที่ซ้ำกันมาก (เช่น 0/1/2 เท่านั้น) | Counting Sort (นอกเหนือขอบเขต Part นี้) | ไม่ใช่ comparison-based เร็วกว่า O(n log n) ได้ |
| สอนหรือเรียนพื้นฐานอัลกอริทึม | Bubble/Selection Sort | เข้าใจง่ายที่สุด เหมาะเป็นจุดเริ่มต้น แต่ไม่ควรใช้จริงในโปรดักชัน |

---

## ข้อผิดพลาดที่พบบ่อย (Common Pitfalls)

1. **ใช้ `int` แทน `size_t` เป็น loop counter แล้วเทียบกับขนาดอาร์เรย์ที่เป็น `size_t`**
   จะได้ Warning `comparison of integer expressions of different signedness` เวลาคอมไพล์
   ด้วย `-Wextra` ควรใช้ `size_t` ให้สอดคล้องกันตลอด หรือใช้ `long`/`int` แบบตั้งใจเมื่อ index
   อาจติดลบได้ (เช่นใน Quick Sort ด้านบน)

2. **Off-by-one ในเงื่อนไข loop ของ Bubble/Selection/Insertion Sort** เช่นเขียน
   `for (size_t j = 0; j <= n - 1 - i; j++)` (ใช้ `<=` แทน `<`) จะทำให้เข้าถึง `arr[j+1]`
   เกินขอบเขตอาร์เรย์ (Out-of-bounds Access) ซึ่งเป็น Undefined Behavior — ควรวาดรูป
   index ด้วยมือทุกครั้งก่อนเขียนเงื่อนไข loop ที่เกี่ยวกับขอบเขตอาร์เรย์

3. **ลืม `free()` อาร์เรย์ชั่วคราวใน Merge Sort** ทำให้เกิด Memory Leak ทุกครั้งที่เรียก
   `merge()` — ควรรัน Valgrind (จะเรียนใน Part 38) ตรวจสอบเสมอเมื่อเขียนโค้ดที่มี `malloc`
   ซ้อนอยู่ใน Recursive Function

4. **เขียน Quick Sort ด้วย `size_t` สำหรับ index `i = low - 1` โดยตรง** ทำให้เกิด Integer
   Underflow กลายเป็นเลขบวกมหาศาลทันทีเมื่อ `low == 0` เพราะ `size_t` เป็น unsigned — ต้องใช้
   ชนิดข้อมูล signed (`long` หรือ `int`) สำหรับ index ที่มีโอกาสติดลบระหว่างการคำนวณ

5. **เขียน Comparator ของ `qsort()` แบบ `return *(int*)a - *(int*)b;` ตรงๆ** อาจเกิด Integer
   Overflow ถ้าค่าที่เทียบห่างกันมาก (เช่น `INT_MAX - INT_MIN` overflow แน่นอน) ควรใช้รูปแบบ
   `(x > y) - (x < y)` ที่ปลอดภัยกว่าเสมอ

6. **ลืม cast `void *` กลับเป็นชนิดข้อมูลจริงก่อนใช้งานใน Comparator ของ `qsort()`** จะทำให้
   compiler แจ้ง error หรือ Undefined Behavior ถ้า dereference `void *` ตรงๆ โดยไม่ cast ก่อน

7. **เข้าใจผิดว่าอัลกอริทึมที่ Time Complexity เท่ากันจะเร็วเท่ากันเสมอในทางปฏิบัติ** เช่น
   คิดว่า Merge Sort กับ Quick Sort เร็วเท่ากันเพราะทั้งคู่ O(n log n) — แต่ Constant Factor,
   Cache Locality, และ Memory Allocation Overhead ทำให้ความเร็วจริงต่างกันได้มาก ต้อง
   **Benchmark จริง** เสมอเมื่อประสิทธิภาพสำคัญ (จะเรียนเครื่องมือ Benchmark ใน Part 90)

---

## แบบฝึกหัดท้ายบท

1. เขียนโปรแกรม `bubble_sort_desc.c` ที่ดัดแปลง Bubble Sort ให้เรียงจาก **มากไปน้อย**
   (Descending) แทน

2. แก้ไข `insertion_sort.c` ให้พิมพ์อาร์เรย์ทั้งหมดออกมาหลังจบแต่ละรอบ `i` (คือหลังแทรก `key`
   แต่ละตัวเสร็จ) เพื่อยืนยัน Trace ที่แสดงในบทเรียนด้วยตัวเอง

3. เขียนโปรแกรมที่นับจำนวนครั้งของการ **เปรียบเทียบ (comparison)** และการ **สลับ (swap)**
   จริงของ Bubble Sort, Selection Sort และ Insertion Sort เมื่อรันบนอาร์เรย์ตัวอย่าง
   `{8, 3, 5, 4, 9, 1}` แล้วเปรียบเทียบตัวเลขที่ได้กับที่วิเคราะห์ไว้ในบทเรียน

4. เขียนฟังก์ชัน `is_sorted(const int arr[], size_t n)` ที่คืนค่า `1` ถ้าอาร์เรย์เรียงจากน้อย
   ไปมากแล้ว หรือ `0` ถ้ายังไม่เรียง แล้วใช้ตรวจสอบผลลัพธ์ของทั้ง 5 อัลกอริทึมในบทนี้

5. ดัดแปลง `quick_sort.c` ให้ใช้ **สมาชิกตัวแรก** ของช่วงเป็น Pivot แทนตัวสุดท้าย (ปรับ
   Partition scheme ให้เหมาะสม) แล้วทดสอบว่ายังได้ผลลัพธ์ถูกต้องหรือไม่

6. เขียนโปรแกรมที่ใช้ `qsort()` เรียง array ของ `struct` ที่มี field `char name[30]` และ
   `double gpa` โดยเรียงตาม `gpa` จากมากไปน้อย ถ้า `gpa` เท่ากันให้เรียงตามชื่อ (`name`)
   จากน้อยไปมาก (A-Z) — โจทย์นี้ฝึกการเขียน Comparator แบบ Multi-key ในฟังก์ชันเดียว

### แนวทางเฉลยข้อ 1

```c
/* ============================================================
 * ชื่อไฟล์:     bubble_sort_desc.c
 * คำอธิบาย:     Bubble Sort แบบเรียงจากมากไปน้อย (Descending)
 * ============================================================ */
#include <stdio.h>
#include <stddef.h>

#define ARRAY_SIZE 6

void bubble_sort_desc(int arr[], size_t n);
void print_array(const int arr[], size_t n);

int main(void) {
    int data[ARRAY_SIZE] = {8, 3, 5, 4, 9, 1};

    printf("ก่อนเรียง: ");
    print_array(data, ARRAY_SIZE);

    bubble_sort_desc(data, ARRAY_SIZE);

    printf("หลังเรียง (มาก->น้อย): ");
    print_array(data, ARRAY_SIZE);

    return 0;
}

void bubble_sort_desc(int arr[], size_t n) {
    if (n < 2) {
        return;
    }

    for (size_t i = 0; i < n - 1; i++) {
        int swapped = 0;

        for (size_t j = 0; j < n - 1 - i; j++) {
            /* เปลี่ยนจาก > เป็น < เพียงจุดเดียว ก็สลับทิศทางการเรียงได้ทันที */
            if (arr[j] < arr[j + 1]) {
                int temp = arr[j];
                arr[j] = arr[j + 1];
                arr[j + 1] = temp;
                swapped = 1;
            }
        }

        if (!swapped) {
            break;
        }
    }
}

void print_array(const int arr[], size_t n) {
    for (size_t i = 0; i < n; i++) {
        printf("%d ", arr[i]);
    }
    printf("\n");
}
```

ผลลัพธ์ที่ควรได้:

```
ก่อนเรียง: 8 3 5 4 9 1
หลังเรียง (มาก->น้อย): 9 8 5 4 3 1
```

จุดสำคัญของเฉลยนี้คือการสังเกตว่า **จุดเดียว** ที่ต้องแก้จากโค้ด Ascending เดิมคือเปลี่ยน
เงื่อนไขการสลับจาก `arr[j] > arr[j + 1]` เป็น `arr[j] < arr[j + 1]` เท่านั้น — หลักการนี้
ใช้ได้กับ Selection Sort และ Insertion Sort เช่นกัน (แค่กลับทิศทางเงื่อนไขเปรียบเทียบ)

### แนวทางเฉลยข้อ 4

```c
/* ============================================================
 * ชื่อไฟล์:     is_sorted.c
 * คำอธิบาย:     ฟังก์ชันตรวจสอบว่าอาร์เรย์เรียงจากน้อยไปมากแล้วหรือยัง
 * ============================================================ */
#include <stdio.h>
#include <stddef.h>

int is_sorted(const int arr[], size_t n);

int main(void) {
    int sorted_arr[6]   = {1, 3, 4, 5, 8, 9};
    int unsorted_arr[6] = {8, 3, 5, 4, 9, 1};
    int single[1]       = {42};
    int empty_check[1]  = {0}; /* ใช้ n=0 เพื่อทดสอบกรณี array ว่าง */

    printf("sorted_arr   is_sorted = %d (คาดว่า 1)\n",
           is_sorted(sorted_arr, 6));
    printf("unsorted_arr is_sorted = %d (คาดว่า 0)\n",
           is_sorted(unsorted_arr, 6));
    printf("single       is_sorted = %d (คาดว่า 1, มีตัวเดียวถือว่าเรียงแล้ว)\n",
           is_sorted(single, 1));
    printf("empty        is_sorted = %d (คาดว่า 1, array ว่างถือว่าเรียงแล้ว)\n",
           is_sorted(empty_check, 0));

    return 0;
}

int is_sorted(const int arr[], size_t n) {
    /* array ที่มี 0 หรือ 1 สมาชิก ถือว่าเรียงแล้วโดยนิยาม (ไม่มีคู่ให้ผิดลำดับได้) */
    for (size_t i = 0; i + 1 < n; i++) {
        if (arr[i] > arr[i + 1]) {
            return 0;
        }
    }
    return 1;
}
```

ผลลัพธ์:

```
sorted_arr   is_sorted = 1 (คาดว่า 1)
unsorted_arr is_sorted = 0 (คาดว่า 0)
single       is_sorted = 1 (คาดว่า 1, มีตัวเดียวถือว่าเรียงแล้ว)
empty        is_sorted = 1 (คาดว่า 1, array ว่างถือว่าเรียงแล้ว)
```

จุดที่ต้องระวังในเฉลยนี้คือเงื่อนไข `i + 1 < n` แทนที่จะเขียน `i < n - 1` — เพราะ `n` เป็น
`size_t` (unsigned) ถ้า `n == 0` การคำนวณ `n - 1` จะ underflow กลายเป็นเลขบวกมหาศาลทันที
แต่ `i + 1 < n` ปลอดภัยกว่าเพราะ `i` เริ่มจาก 0 เสมอ ไม่มีทาง overflow ในทางปฏิบัติ

---

## สรุปท้ายบท

ใน Part นี้เราได้:

- เข้าใจว่า Sorting คือปัญหาพื้นฐานที่สำคัญที่สุดปัญหาหนึ่งในวิทยาการคอมพิวเตอร์ และเป็น
  รากฐานของอัลกอริทึมอื่นอีกมากมาย
- เขียนและ Trace การทำงานของ **Bubble Sort, Selection Sort, Insertion Sort** ซึ่งเป็นกลุ่ม
  อัลกอริทึม O(n²) ที่เข้าใจง่ายแต่ไม่เหมาะกับข้อมูลขนาดใหญ่
- เขียนและ Trace การทำงานของ **Merge Sort และ Quick Sort** ซึ่งใช้แนวคิด Divide and Conquer
  ทำให้ได้ความซับซ้อนเฉลี่ย O(n log n)
- เข้าใจแนวคิด **Stability** และรู้ว่าอัลกอริทึมไหน Stable (Bubble, Insertion, Merge) หรือ
  Unstable (Selection, Quick) พร้อมเหตุผลเบื้องหลัง
- จดจำตารางเปรียบเทียบ Time/Space Complexity ของอัลกอริทึมทั้ง 5 ตัว ซึ่งเป็นความรู้พื้นฐาน
  ที่ต้องใช้ตลอดเส้นทางอาชีพโปรแกรมเมอร์
- ใช้ `qsort()` จาก `<stdlib.h>` พร้อมเขียน Comparator function เองสำหรับข้อมูลตัวเลขและ
  `struct` และรู้แนวทางเลือกอัลกอริทึมที่เหมาะสมกับสถานการณ์จริง

ใน **Part 24** เราจะไปเรียนรู้อีกฝั่งหนึ่งของปัญหาพื้นฐาน: **การค้นหา (Searching)** ทั้ง
Linear Search และ Binary Search พร้อมปูพื้นฐาน **Big-O Notation** อย่างเป็นทางการ ซึ่งเป็น
เครื่องมือที่เราใช้ในการวิเคราะห์อัลกอริทึมทุกตัวที่ผ่านมาแล้ว (รวมถึงอัลกอริทึมเรียงลำดับ
ใน Part นี้) และจะใช้ต่อไปตลอดทั้งหลักสูตร

**ต่อไป:** [Part 24 — Searching Algorithm และ Big-O](./part-024-searching-bigo.md)
