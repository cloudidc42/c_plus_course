# Part 12: Array หลายมิติและ Pointer-to-Pointer (Step 89–96)

> Module B — C ระดับกลาง & โครงสร้างข้อมูล/อัลกอริทึม | Part 12 จาก 125
> Step ที่ครอบคลุมใน Part นี้: Step 89–96
> Part ก่อนหน้า: [Part 11 — Dynamic Memory Allocation](./part-011-dynamic-memory.md) | Part ถัดไป: [Part 13 — File I/O ใน C](./part-013-file-io.md)

## เป้าหมายการเรียนรู้ (Learning Objectives)

หลังจบ Part นี้ ผู้เรียนจะสามารถ:

1. ประกาศและใช้งาน **2D Array แบบ Static** (`int grid[ROWS][COLS]`) ได้อย่างถูกต้อง
2. อธิบายวิธีที่ 2D Array แบบ Static ถูกจัดเก็บในหน่วยความจำจริงแบบ **Row-Major Order**
   และคำนวณตำแหน่งของ element ใดๆ ด้วยสูตรได้เอง
3. ส่ง 2D Array เป็นพารามิเตอร์ให้ฟังก์ชันได้อย่างถูกต้อง ทั้งแบบขนาดคงที่และแบบระบุจำนวนแถว
   ตอนรันไทม์
4. สร้าง **Dynamic 2D Array** ด้วยเทคนิค Pointer-to-Pointer (`int **`) แบบ "array ของ pointer"
   (Ragged Array)
5. สร้าง **Dynamic 2D Array แบบจัดสรรก้อนเดียวต่อเนื่อง** (Contiguous Allocation) และอธิบาย
   ข้อดี-ข้อเสียเทียบกับแบบ Ragged Array
6. เลือกวิธีจัดสรร 2D Array ที่เหมาะสมกับสถานการณ์ต่างๆ ได้อย่างมีเหตุผล
7. เขียนโปรแกรม **Matrix Multiplication** แบบเต็มรูปแบบโดยใช้ Dynamic 2D Array
8. หลีกเลี่ยงข้อผิดพลาดที่พบบ่อยเกี่ยวกับการ `free` และการเข้าถึง 2D Array แบบ dynamic

---

## 12.1 2D Array แบบ Static และการจัดเก็บแบบ Row-Major Order (Step 89)

ใน Part 7 เราเรียน Array 1 มิติไปแล้ว ภาษา C รองรับ Array หลายมิติด้วยไวยากรณ์ที่ตรงไปตรงมา:

```c
int grid[3][4]; /* 3 แถว, 4 คอลัมน์ = 12 ช่องทั้งหมด */
```

สิ่งสำคัญที่สุดที่ต้องเข้าใจคือ: **หน่วยความจำของคอมพิวเตอร์เป็นเส้นตรงมิติเดียวเสมอ (Linear
Address Space)** ไม่มี "หน่วยความจำ 2 มิติ" อยู่จริง สิ่งที่ `int grid[3][4]` ทำคือจัดเรียง
ข้อมูล 12 ช่องนี้ต่อกันเป็นเส้นตรง โดยเรียงตาม **Row-Major Order** — คือเรียงข้อมูลของแถวที่ 0
ให้ครบก่อน แล้วต่อด้วยแถวที่ 1, 2, ... ตามลำดับ

```c
/* row_major.c */
#include <stdio.h>

#define ROWS 3
#define COLS 4

void print_matrix(int mat[ROWS][COLS]) {
    for (int i = 0; i < ROWS; i++) {
        for (int j = 0; j < COLS; j++) {
            printf("%3d ", mat[i][j]);
        }
        printf("\n");
    }
}

int main(void) {
    int grid[ROWS][COLS] = {
        {1, 2, 3, 4},
        {5, 6, 7, 8},
        {9, 10, 11, 12}
    };

    print_matrix(grid);

    printf("ที่อยู่ของ grid[0][0] = %p\n", (void *)&grid[0][0]);
    printf("ที่อยู่ของ grid[0][1] = %p\n", (void *)&grid[0][1]);
    printf("ที่อยู่ของ grid[1][0] = %p\n", (void *)&grid[1][0]);
    printf("ขนาดของ int 1 ตัว = %zu bytes\n", sizeof(int));

    return 0;
}
```

ผลลัพธ์ (ที่อยู่จริงจะต่างกันในแต่ละเครื่อง แต่ **ระยะห่างระหว่างที่อยู่** จะเหมือนกันเสมอ):

```
  1   2   3   4
  5   6   7   8
  9  10  11  12
ที่อยู่ของ grid[0][0] = 0x7ffc3f1e3b90
ที่อยู่ของ grid[0][1] = 0x7ffc3f1e3b94
ที่อยู่ของ grid[1][0] = 0x7ffc3f1e3ba0
ขนาดของ int 1 ตัว = 4 bytes
```

สังเกตว่า `grid[0][1]` อยู่ห่างจาก `grid[0][0]` แค่ 4 byte (ขนาดของ `int` 1 ตัว) แต่ `grid[1][0]`
อยู่ห่างจาก `grid[0][0]` ถึง 16 byte (`0x...ba0 - 0x...b90 = 0x10 = 16`) เพราะต้องข้ามข้อมูล
ทั้งแถวที่ 0 ไปก่อน (4 ช่อง × 4 byte = 16 byte) นี่คือหลักฐานที่แสดงว่า `grid` ถูกจัดเก็บเป็น
บล็อกต่อเนื่องกันจริงๆ ในหน่วยความจำ

### สูตรคำนวณตำแหน่งของ Element

ในความเป็นจริง คอมไพเลอร์จะแปลง `grid[i][j]` ให้กลายเป็นการคำนวณ **offset** จากจุดเริ่มต้น
ของ array ด้วยสูตร:

```
offset(i, j) = (i * COLS + j) * sizeof(element_type)
```

เช่น `grid[1][2]` จะมี offset เท่ากับ `(1 * 4 + 2) * sizeof(int) = 6 * 4 = 24` byte จากจุด
เริ่มต้นของ `grid` — การเข้าใจสูตรนี้สำคัญมากเมื่อจะย้ายไปเขียน Dynamic 2D Array แบบ
Contiguous ในหัวข้อ 12.4 เพราะเราต้องคำนวณ index เองด้วยสูตรเดียวกันนี้

| แนวคิด | ค่า |
|---|---|
| จำนวนช่องทั้งหมด | `ROWS * COLS` |
| ขนาดหน่วยความจำทั้งหมด | `ROWS * COLS * sizeof(int)` byte |
| `sizeof(grid)` | เท่ากับขนาดหน่วยความจำทั้งหมด (เพราะ `grid` เป็น array จริง ไม่ใช่ pointer) |
| `sizeof(grid[0])` | เท่ากับ `COLS * sizeof(int)` (ขนาดของ 1 แถว) |

---

## 12.2 การส่ง 2D Array เป็นพารามิเตอร์ให้ฟังก์ชัน (Step 90)

เมื่อส่ง Array (ไม่ว่ากี่มิติ) เป็นพารามิเตอร์ให้ฟังก์ชัน ภาษา C จะ **"สลาย" (decay) มิติแรก
ให้กลายเป็น pointer เสมอ** (ทบทวนจาก Part 7-8) แต่สำหรับ 2D Array **มิติที่สอง (จำนวนคอลัมน์)
ต้องระบุให้ชัดเจนเสมอ** เพราะคอมไพเลอร์ต้องใช้ค่านี้ในการคำนวณ offset ตามสูตรด้านบน

```c
void print_matrix(int mat[ROWS][COLS]) { ... }   /* ใช้ได้ */
void print_matrix(int mat[][COLS]) { ... }        /* เขียนแบบนี้ก็ได้ผลเหมือนกัน (นิยมกว่า) */
void print_matrix(int mat[][]) { ... }             /* ผิด! compile error เพราะไม่รู้ COLS */
```

ถ้าจำนวนแถวไม่ตายตัว (รู้แค่ตอนรันไทม์) ให้ส่งจำนวนแถวเป็นพารามิเตอร์แยกต่างหาก โดยจำนวน
คอลัมน์ยังต้องเป็นค่าคงที่ตอนคอมไพล์อยู่ดี (เพราะเป็น static array):

```c
/* pass_2d_array.c */
#include <stdio.h>

#define COLS 4

void print_matrix(size_t rows, int mat[][COLS]) {
    for (size_t i = 0; i < rows; i++) {
        for (size_t j = 0; j < COLS; j++) {
            printf("%3d ", mat[i][j]);
        }
        printf("\n");
    }
}

int main(void) {
    int grid[3][COLS] = {
        {1, 2, 3, 4},
        {5, 6, 7, 8},
        {9, 10, 11, 12}
    };

    print_matrix(3, grid);
    return 0;
}
```

```
  1   2   3   4
  5   6   7   8
  9  10  11  12
```

> **ข้อจำกัดสำคัญ**: static 2D array ต้องรู้จำนวนคอลัมน์ตอนคอมไพล์เสมอ (เว้นแต่จะใช้ Variable
> Length Array ของ C99 ซึ่งมีข้อจำกัดเรื่อง Stack และไม่แนะนำในโค้ด production) ถ้าต้องการ
> ทั้งจำนวนแถวและคอลัมน์ที่ไม่ทราบล่วงหน้า (กำหนดตอนรันไทม์จากผู้ใช้ เช่น อ่านขนาดจากไฟล์)
> คำตอบคือต้องใช้ **Dynamic 2D Array** ซึ่งเป็นหัวข้อถัดไป

---

## 12.3 Dynamic 2D Array แบบ Pointer-to-Pointer (Ragged Array) (Step 91–92)

เมื่อทั้งจำนวนแถวและคอลัมน์ไม่ทราบตอนคอมไพล์ วิธีที่นิยมที่สุดในภาษา C คือใช้ **Pointer-to-
Pointer** (`int **`) โดยมีแนวคิด 2 ขั้นตอน:

1. `malloc` **array ของ pointer** จำนวน `rows` ตัว (แต่ละตัวคือ `int *`)
2. วน loop `malloc` **แต่ละแถวแยกกัน** จำนวน `cols` ตัวต่อแถว

```
mat (int **)
 │
 ├──> mat[0] (int *) ──> [ ][ ][ ][ ]   <- malloc แยกก้อนที่ 1
 ├──> mat[1] (int *) ──> [ ][ ][ ][ ]   <- malloc แยกก้อนที่ 2
 └──> mat[2] (int *) ──> [ ][ ][ ][ ]   <- malloc แยกก้อนที่ 3

(แต่ละแถวเป็นก้อนหน่วยความจำที่ "แยกจากกัน" ไม่รับประกันว่าอยู่ติดกัน จึงเรียกว่า
"Ragged Array" - ในทางทฤษฎีแต่ละแถวจะมีความยาวต่างกันก็ได้ด้วย)
```

```c
/* dynamic_2d_ragged.c */
#include <stdio.h>
#include <stdlib.h>

/* จองหน่วยความจำสำหรับเมทริกซ์ขนาด rows x cols แบบ ragged array
 * Ownership Contract: ผู้เรียกเป็นเจ้าของผลลัพธ์ ต้องเรียก free_matrix_ragged() เอง
 * คืนค่า NULL ถ้าจองหน่วยความจำล้มเหลว */
int **alloc_matrix_ragged(size_t rows, size_t cols) {
    int **mat = malloc(rows * sizeof *mat);
    if (mat == NULL) {
        return NULL;
    }
    for (size_t i = 0; i < rows; i++) {
        mat[i] = malloc(cols * sizeof *mat[i]);
        if (mat[i] == NULL) {
            /* จองแถวใดแถวหนึ่งล้มเหลว ต้อง free แถวที่จองไปแล้วก่อนหน้าทั้งหมด
               เพื่อไม่ให้เกิด memory leak */
            for (size_t k = 0; k < i; k++) {
                free(mat[k]);
            }
            free(mat);
            return NULL;
        }
    }
    return mat;
}

void free_matrix_ragged(int **mat, size_t rows) {
    if (mat == NULL) {
        return;
    }
    for (size_t i = 0; i < rows; i++) {
        free(mat[i]); /* free ทีละแถวก่อนเสมอ */
    }
    free(mat); /* แล้วค่อย free array ของ pointer เอง */
}

int main(void) {
    size_t rows = 3, cols = 4;
    int **mat = alloc_matrix_ragged(rows, cols);
    if (mat == NULL) {
        fprintf(stderr, "จองหน่วยความจำล้มเหลว\n");
        return 1;
    }

    int counter = 1;
    for (size_t i = 0; i < rows; i++) {
        for (size_t j = 0; j < cols; j++) {
            mat[i][j] = counter++;
        }
    }

    for (size_t i = 0; i < rows; i++) {
        for (size_t j = 0; j < cols; j++) {
            printf("%3d ", mat[i][j]);
        }
        printf("\n");
    }

    printf("ที่อยู่ mat[0] = %p, mat[1] = %p (ไม่รับประกันว่าติดกัน)\n",
           (void *)mat[0], (void *)mat[1]);

    free_matrix_ragged(mat, rows);
    return 0;
}
```

```
  1   2   3   4
  5   6   7   8
  9  10  11  12
ที่อยู่ mat[0] = 0x555c87e922c0, mat[1] = 0x555c87e922e0 (ไม่รับประกันว่าติดกัน)
```

**จุดสำคัญเรื่องการ `free`**: ต้อง `free` **ทุกแถวก่อน** แล้วค่อย `free` array ของ pointer เอง
ในลำดับที่ตรงกันข้ามกับการจอง (จองจากนอกเข้าใน → free จากในออกนอก) ถ้าลืม `free` แถวใด
แถวหนึ่งก่อน จะเกิด Memory Leak ของก้อนนั้น (ทบทวนหัวข้อ 11.5) และถ้า `free(mat)` ก่อน
`free(mat[i])` จะทำให้เข้าถึง `mat[i]` ไม่ได้อีกต่อไป (dangling — ก้อนของแต่ละแถวจะรั่วตลอดกาล)

**ข้อดีของ Ragged Array**: แต่ละแถวสามารถมีความยาวไม่เท่ากันได้ (เช่นเก็บ adjacency list
ของกราฟที่จะเรียนใน Module ถัดๆ ไป ที่แต่ละ node มีจำนวนเพื่อนบ้านไม่เท่ากัน)

**ข้อเสีย**: การเข้าถึงข้อมูลต้อง dereference สองชั้น (`mat[i][j]` = อ่าน `mat[i]` ก่อนเพื่อได้
`int *` แล้วค่อยอ่าน `[j]`) ทำให้ **cache locality แย่กว่า** เพราะแต่ละแถวกระจายอยู่คนละที่ใน
หน่วยความจำ (จะเข้าใจผลกระทบด้านประสิทธิภาพนี้ลึกซึ้งขึ้นใน **Part 87 — Cache-Friendly Code**)

---

## 12.4 Dynamic 2D Array แบบจัดสรรก้อนเดียวต่อเนื่อง (Contiguous Allocation) (Step 93–94)

ทางเลือกที่สองคือจำลองพฤติกรรมของ static 2D array ให้ได้มากที่สุด: **จอง `malloc` เพียง
ก้อนเดียว** ที่มีขนาดใหญ่พอสำหรับข้อมูลทั้งหมด (`rows * cols` ช่อง) แล้วใช้สูตร Row-Major
Order จากหัวข้อ 12.1 คำนวณเอาเอง หรือใช้ array ของ pointer มาช่วย "ชี้" เข้าไปยังจุดเริ่มต้น
ของแต่ละแถวในก้อนเดียวกันนั้น

```
mat (int **) ──> [mat[0]][mat[1]][mat[2]]   <- array ของ pointer (จองแยก)
                    │        │        │
                    ▼        ▼        ▼
block (int *) ──> [ ][ ][ ][ ][ ][ ][ ][ ][ ][ ][ ][ ]
                  └───row0───┘└───row1───┘└───row2───┘
                  (ก้อนเดียวต่อเนื่องกันทั้งหมด rows*cols ช่อง)
```

```c
/* dynamic_2d_contiguous.c */
#include <stdio.h>
#include <stdlib.h>

/* จองหน่วยความจำสำหรับเมทริกซ์ขนาด rows x cols แบบก้อนเดียวต่อเนื่อง
 * Ownership Contract: ผู้เรียกเป็นเจ้าของผลลัพธ์ ต้องเรียก free_matrix_contiguous() เอง */
int **alloc_matrix_contiguous(size_t rows, size_t cols) {
    int **mat = malloc(rows * sizeof *mat);
    if (mat == NULL) {
        return NULL;
    }
    int *block = malloc(rows * cols * sizeof *block); /* ก้อนเดียวสำหรับข้อมูลทั้งหมด */
    if (block == NULL) {
        free(mat);
        return NULL;
    }
    for (size_t i = 0; i < rows; i++) {
        mat[i] = block + (i * cols); /* ให้ mat[i] ชี้เข้าไปยังจุดเริ่มต้นของแถว i ในก้อนเดียวกัน */
    }
    return mat;
}

void free_matrix_contiguous(int **mat) {
    if (mat == NULL) {
        return;
    }
    free(mat[0]); /* คืนก้อนข้อมูลใหญ่ก้อนเดียว (mat[0] ชี้ไปจุดเริ่มต้นของ block เสมอ) */
    free(mat);    /* คืน array ของ pointer */
}

int main(void) {
    size_t rows = 3, cols = 4;
    int **mat = alloc_matrix_contiguous(rows, cols);
    if (mat == NULL) {
        fprintf(stderr, "จองหน่วยความจำล้มเหลว\n");
        return 1;
    }

    int counter = 1;
    for (size_t i = 0; i < rows; i++) {
        for (size_t j = 0; j < cols; j++) {
            mat[i][j] = counter++;
        }
    }

    for (size_t i = 0; i < rows; i++) {
        for (size_t j = 0; j < cols; j++) {
            printf("%3d ", mat[i][j]);
        }
        printf("\n");
    }

    printf("mat[1] - mat[0] = %td ints (ต้องเท่ากับ cols=%zu เพราะติดกันจริง)\n",
           mat[1] - mat[0], cols);

    free_matrix_contiguous(mat);
    return 0;
}
```

```
  1   2   3   4
  5   6   7   8
  9  10  11  12
mat[1] - mat[0] = 4 ints (ต้องเท่ากับ cols=4 เพราะติดกันจริง)
```

สังเกตว่า `mat[1] - mat[0]` (pointer arithmetic ที่เรียนใน Part 9) ได้ค่าเท่ากับ `cols` พอดี
ซึ่งพิสูจน์ว่าทุกแถวอยู่ติดกันจริงในหน่วยความจำ ต่างจาก Ragged Array ที่ไม่รับประกันเรื่องนี้เลย

**การ `free` ต้อง `free(mat[0])` ก่อนเสมอ** (คืน block ข้อมูลใหญ่) แล้วค่อย `free(mat)` (คืน
array ของ pointer) — ลำดับสลับกันจากตอนจองไม่ได้ เพราะที่นี่มีเพียง **2 ก้อน** เท่านั้น
(ต่างจาก Ragged Array ที่มี `rows + 1` ก้อน)

### เปรียบเทียบ Ragged Array กับ Contiguous Allocation

| ประเด็น | Ragged Array (`int **` + malloc ทีละแถว) | Contiguous Allocation (`int **` + malloc ก้อนเดียว) |
|---|---|---|
| จำนวนครั้งที่เรียก `malloc` | `rows + 1` ครั้ง | 2 ครั้ง |
| จำนวนครั้งที่เรียก `free` | ต้อง `free` ให้ครบ `rows + 1` ครั้ง | แค่ 2 ครั้ง |
| แต่ละแถวความยาวต่างกันได้ไหม | ได้ (true ragged array) | ไม่ได้ (ทุกแถวต้องยาวเท่ากัน) |
| Cache locality เมื่อเข้าถึงข้อมูลต่อเนื่อง | แย่กว่า (แต่ละแถวกระจายในหน่วยความจำ) | ดีกว่ามาก (ข้อมูลติดกันเหมือน static array) |
| ความเสี่ยงเรื่อง memory leak ตอน error | สูงกว่า (ต้อง cleanup หลายก้อนถ้าจองกลางทางล้มเหลว) | ต่ำกว่า (cleanup ง่ายกว่า มีแค่ 2 ก้อน) |
| ความเร็วในการจอง (allocation overhead) | ช้ากว่า (เรียก malloc หลายครั้ง) | เร็วกว่า (เรียก malloc แค่ 2 ครั้ง) |

> **คำแนะนำในทางปฏิบัติ**: ถ้าไม่มีเหตุผลเฉพาะที่ต้องการให้แต่ละแถวยาวไม่เท่ากัน (เช่น
> adjacency list) ให้เลือกใช้ **Contiguous Allocation เป็นค่าเริ่มต้นเสมอ** เพราะเร็วกว่า
> ปลอดภัยกว่า และมี cache locality ที่ดีกว่าอย่างชัดเจน โดยเฉพาะเมื่อทำงานกับเมทริกซ์ขนาดใหญ่
> ในงานคำนวณเชิงตัวเลข (Numerical Computing) หรือ Machine Learning

---

## 12.5 ตัวอย่างจริง: Matrix Multiplication (Step 95–96)

มาประยุกต์ทุกอย่างที่เรียนมาเขียนโปรแกรมคูณเมทริกซ์ (Matrix Multiplication) ซึ่งเป็นพื้นฐาน
สำคัญของ Computer Graphics, Machine Learning และการคำนวณเชิงวิทยาศาสตร์เกือบทุกแขนง

กฎการคูณเมทริกซ์: เมทริกซ์ A ขนาด `n x m` คูณกับเมทริกซ์ B ขนาด `m x p` ได้ผลลัพธ์ C ขนาด
`n x p` โดย `C[i][j] = ผลรวมของ (A[i][k] * B[k][j])` สำหรับทุก `k` ตั้งแต่ `0` ถึง `m-1`
(จำนวนคอลัมน์ของ A ต้องเท่ากับจำนวนแถวของ B เท่านั้น จึงจะคูณกันได้)

```c
/* matrix_multiply.c */
#include <stdio.h>
#include <stdlib.h>

static int **alloc_matrix(size_t rows, size_t cols) {
    int **mat = malloc(rows * sizeof *mat);
    if (mat == NULL) {
        return NULL;
    }
    int *block = calloc(rows * cols, sizeof *block);
    if (block == NULL) {
        free(mat);
        return NULL;
    }
    for (size_t i = 0; i < rows; i++) {
        mat[i] = block + (i * cols);
    }
    return mat;
}

static void free_matrix(int **mat) {
    if (mat == NULL) {
        return;
    }
    free(mat[0]);
    free(mat);
}

static void print_matrix(int **mat, size_t rows, size_t cols) {
    for (size_t i = 0; i < rows; i++) {
        for (size_t j = 0; j < cols; j++) {
            printf("%4d ", mat[i][j]);
        }
        printf("\n");
    }
}

/* คูณเมทริกซ์ a (n x m) กับ b (m2 x p) ได้ผลลัพธ์เก็บใน c (n x p)
 * คืนค่า 1 ถ้าขนาดเข้ากันและคูณสำเร็จ, 0 ถ้าขนาดไม่เข้ากัน (m != m2) */
static int multiply(int **a, size_t n, size_t m,
                     int **b, size_t m2, size_t p,
                     int **c) {
    if (m != m2) {
        return 0;
    }
    for (size_t i = 0; i < n; i++) {
        for (size_t j = 0; j < p; j++) {
            int sum = 0;
            for (size_t k = 0; k < m; k++) {
                sum += a[i][k] * b[k][j];
            }
            c[i][j] = sum;
        }
    }
    return 1;
}

int main(void) {
    size_t n = 2, m = 3, p = 2;

    int **a = alloc_matrix(n, m);
    int **b = alloc_matrix(m, p);
    int **c = alloc_matrix(n, p);
    if (a == NULL || b == NULL || c == NULL) {
        fprintf(stderr, "จองหน่วยความจำล้มเหลว\n");
        free_matrix(a);
        free_matrix(b);
        free_matrix(c);
        return 1;
    }

    int a_vals[2][3] = {{1, 2, 3}, {4, 5, 6}};
    int b_vals[3][2] = {{7, 8}, {9, 10}, {11, 12}};

    for (size_t i = 0; i < n; i++) {
        for (size_t j = 0; j < m; j++) {
            a[i][j] = a_vals[i][j];
        }
    }
    for (size_t i = 0; i < m; i++) {
        for (size_t j = 0; j < p; j++) {
            b[i][j] = b_vals[i][j];
        }
    }

    printf("A =\n");
    print_matrix(a, n, m);
    printf("B =\n");
    print_matrix(b, m, p);

    if (!multiply(a, n, m, b, m, p, c)) {
        fprintf(stderr, "ขนาดเมทริกซ์ไม่เข้ากัน คูณกันไม่ได้\n");
        free_matrix(a);
        free_matrix(b);
        free_matrix(c);
        return 1;
    }

    printf("A x B =\n");
    print_matrix(c, n, p);

    free_matrix(a);
    free_matrix(b);
    free_matrix(c);
    return 0;
}
```

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 matrix_multiply.c -o matrix_multiply
./matrix_multiply
```

ผลลัพธ์:

```
A =
   1    2    3
   4    5    6
B =
   7    8
   9   10
  11   12
A x B =
  58   64
 139  154
```

ตรวจทานด้วยมือ: `C[0][0] = 1*7 + 2*9 + 3*11 = 7 + 18 + 33 = 58` ตรงกับผลลัพธ์ที่ได้ ✓

สังเกตว่าเราใช้ `calloc` แทน `malloc` ในฟังก์ชัน `alloc_matrix` ของตัวอย่างนี้ (ทบทวนหัวข้อ
11.3) เพื่อให้ `c` เริ่มต้นเป็น 0 ทั้งหมดโดยอัตโนมัติ แม้ในโค้ดนี้เราจะเขียนทับทุกช่องอยู่แล้วก็ตาม
แต่การใช้ `calloc` เป็นนิสัยที่ปลอดภัยกว่าเมื่อไม่แน่ใจ 100% ว่าทุกช่องจะถูกเขียนทับแน่นอน

ฟังก์ชัน `multiply` ในตัวอย่างนี้ยังสาธิตการตรวจสอบเงื่อนไข **ก่อนคำนวณ** (`m != m2`) ซึ่งเป็น
ตัวอย่างของ **Defensive Programming** ที่จะเรียนอย่างละเอียดใน **Part 16 (Error Handling)**

---

## ข้อผิดพลาดที่พบบ่อย (Common Pitfalls)

1. **ลืมระบุจำนวนคอลัมน์เมื่อรับ 2D array เป็นพารามิเตอร์** เช่นเขียน `void f(int mat[][])`
   ซึ่งจะ compile error ทันที เพราะคอมไพเลอร์ต้องรู้จำนวนคอลัมน์เพื่อคำนวณ offset ของแต่ละแถว
2. **สับสนระหว่าง `int **` กับ `int (*)[COLS]`** — `int **` คือ pointer ไปยัง pointer (ใช้กับ
   dynamic 2D array แบบ ragged/contiguous ที่มี array ของ pointer จริง) ส่วน `int (*)[COLS]`
   คือ pointer ไปยัง array ของ `int` ขนาด `COLS` (ใช้ตอนส่ง static 2D array ให้ฟังก์ชันแบบ
   `int mat[][COLS]`) ทั้งสองแบบ **ไม่สามารถใช้แทนกันได้** และการ cast ผิดชนิดจะทำให้ผลลัพธ์
   ผิดเพี้ยนโดยไม่มี warning ชัดเจนเสมอไป
3. **`free` ไม่ครบทุกแถวของ Ragged Array** — ลืมวน loop `free(mat[i])` ก่อน `free(mat)` ทำให้
   แต่ละแถวรั่ว (memory leak) ทีละก้อน ยิ่งเมทริกซ์มีจำนวนแถวมาก ยิ่งรั่วมาก
   (ทบทวนหัวข้อ 12.3)
4. **`free(mat[0])` ผิดจุดใน Contiguous Allocation** — ถ้าไปเผลอ `free(mat[1])` หรือ
   `free(mat[2])` แทน จะเป็น Undefined Behavior ทันที เพราะ `free` ต้องได้รับ pointer ที่ตรงกับ
   จุดเริ่มต้นของก้อนที่ `malloc` คืนมาเป๊ะๆ เท่านั้น (มีแค่ `mat[0]` ที่ตรงกับจุดเริ่มต้นของ `block`)
5. **สลับลำดับการ `free`** — เผลอ `free(mat)` (array ของ pointer) ก่อน `free(mat[0])`
   (ก้อนข้อมูล) ใน Contiguous Allocation ทำให้ pointer ที่ชี้ไปยัง `block` หายไป ก่อนจะได้
   `free` มัน กลายเป็น memory leak ของก้อนข้อมูลทั้งก้อน
6. **จองแบบ Ragged Array แต่หวังว่าแต่ละแถวจะอยู่ติดกัน** แล้วใช้ pointer arithmetic ข้ามแถว
   เช่น `mat[0][cols]` เพื่อหวังว่าจะได้ค่าเดียวกับ `mat[1][0]` — เป็น Undefined Behavior เพราะ
   Ragged Array **ไม่รับประกัน** ว่าแถวต่างๆ จะอยู่ติดกันในหน่วยความจำ (ต่างจาก Contiguous
   Allocation ที่รับประกันเรื่องนี้)
7. **คำนวณขนาดสำหรับ `malloc` ผิดสูตร** เช่นเขียน `malloc(rows + cols)` แทนที่จะเป็น
   `malloc(rows * cols * sizeof(int))` — ทำให้จองหน่วยความจำน้อยเกินไปมาก และเขียนข้อมูล
   เกินขอบเขตทันที (Buffer Overflow)

---

## แบบฝึกหัดท้ายบท

1. เขียนฟังก์ชัน `int **transpose(int **src, size_t rows, size_t cols)` ที่คืนเมทริกซ์ใหม่
   (ขนาด `cols x rows`) ซึ่งเป็น transpose ของ `src` โดยใช้ Dynamic 2D Array แบบ Contiguous
   Allocation พร้อมเขียน Ownership Contract กำกับให้ชัดเจน
2. โค้ดด้านล่างนี้จองหน่วยความจำแบบ Ragged Array ได้ถูกต้อง แต่ฟังก์ชัน `free_matrix_ragged`
   มีบั๊ก memory leak ซ่อนอยู่ ให้หาและแก้ไข:
   ```c
   void free_matrix_ragged(int **mat, size_t rows) {
       (void)rows;
       free(mat); /* บั๊ก: free แค่ array ของ pointer ไม่ได้ free แต่ละแถวเลย! */
   }
   ```
3. เขียนฟังก์ชัน `long sum_matrix(int **mat, size_t rows, size_t cols)` ที่คืนผลรวมของสมาชิก
   ทั้งหมดในเมทริกซ์ (ใช้ได้กับทั้ง Ragged Array และ Contiguous Allocation เพราะ interface
   เป็น `int **` เหมือนกัน)
4. อธิบายด้วยคำพูดของตัวเองว่าทำไม `mat[1] - mat[0]` ในตัวอย่าง Contiguous Allocation
   (หัวข้อ 12.4) ถึงได้ค่าเท่ากับ `cols` เสมอ แต่ในตัวอย่าง Ragged Array (หัวข้อ 12.3) ค่านี้
   ไม่สามารถคาดเดาได้เลย
5. ขยายโปรแกรม `matrix_multiply.c` ในหัวข้อ 12.5 ให้รับขนาดเมทริกซ์ (`n`, `m`, `p`) และค่า
   ในแต่ละช่องจากผู้ใช้ผ่าน `scanf` แทนการกำหนดค่าตายตัวในโค้ด (คำใบ้: จะได้ใช้ `fscanf`/
   `scanf` เจาะลึกใน Part 13)
6. เขียนฟังก์ชัน `void fill_identity(int **mat, size_t n)` ที่เติมค่าให้ `mat` (ขนาด `n x n`)
   กลายเป็น Identity Matrix (เส้นทแยงมุมเป็น 1 ที่เหลือเป็น 0)

### แนวทางเฉลยข้อ 1

```c
#include <stdio.h>
#include <stdlib.h>

static int **alloc_matrix(size_t rows, size_t cols) {
    int **mat = malloc(rows * sizeof *mat);
    if (mat == NULL) {
        return NULL;
    }
    int *block = malloc(rows * cols * sizeof *block);
    if (block == NULL) {
        free(mat);
        return NULL;
    }
    for (size_t i = 0; i < rows; i++) {
        mat[i] = block + (i * cols);
    }
    return mat;
}

static void free_matrix(int **mat) {
    if (mat == NULL) {
        return;
    }
    free(mat[0]);
    free(mat);
}

static void print_matrix(int **mat, size_t rows, size_t cols) {
    for (size_t i = 0; i < rows; i++) {
        for (size_t j = 0; j < cols; j++) {
            printf("%3d ", mat[i][j]);
        }
        printf("\n");
    }
}

/* คืนเมทริกซ์ใหม่ที่เป็น transpose ของ src (rows x cols) -> ผลลัพธ์ (cols x rows)
 * Ownership Contract: ผู้เรียกเป็นเจ้าของหน่วยความจำที่คืนมา ต้อง free_matrix() เอง
 * คืนค่า NULL ถ้าจองหน่วยความจำล้มเหลว */
static int **transpose(int **src, size_t rows, size_t cols) {
    int **dst = alloc_matrix(cols, rows);
    if (dst == NULL) {
        return NULL;
    }
    for (size_t i = 0; i < rows; i++) {
        for (size_t j = 0; j < cols; j++) {
            dst[j][i] = src[i][j];
        }
    }
    return dst;
}

int main(void) {
    size_t rows = 2, cols = 3;
    int **m = alloc_matrix(rows, cols);
    if (m == NULL) {
        return EXIT_FAILURE;
    }

    int values[2][3] = {{1, 2, 3}, {4, 5, 6}};
    for (size_t i = 0; i < rows; i++) {
        for (size_t j = 0; j < cols; j++) {
            m[i][j] = values[i][j];
        }
    }

    printf("เมทริกซ์ต้นฉบับ (%zux%zu):\n", rows, cols);
    print_matrix(m, rows, cols);

    int **t = transpose(m, rows, cols);
    if (t == NULL) {
        free_matrix(m);
        return EXIT_FAILURE;
    }

    printf("เมทริกซ์ transpose (%zux%zu):\n", cols, rows);
    print_matrix(t, cols, rows);

    free_matrix(m);
    free_matrix(t);
    return EXIT_SUCCESS;
}
```

ผลลัพธ์:

```
เมทริกซ์ต้นฉบับ (2x3):
  1   2   3
  4   5   6
เมทริกซ์ transpose (3x2):
  1   4
  2   5
  3   6
```

### แนวทางเฉลยข้อ 2

บั๊กคือฟังก์ชันไม่ได้ `free` แต่ละแถวก่อนที่จะ `free(mat)` เลย ทำให้ทุกแถวรั่วหมด แก้ไขโดยวน
loop `free` แต่ละแถวก่อนเสมอ:

```c
#include <stdio.h>
#include <stdlib.h>

int **alloc_matrix_ragged(size_t rows, size_t cols) {
    int **mat = malloc(rows * sizeof *mat);
    if (mat == NULL) {
        return NULL;
    }
    for (size_t i = 0; i < rows; i++) {
        mat[i] = malloc(cols * sizeof *mat[i]);
        if (mat[i] == NULL) {
            for (size_t k = 0; k < i; k++) {
                free(mat[k]);
            }
            free(mat);
            return NULL;
        }
    }
    return mat;
}

/* เวอร์ชันแก้ไขแล้ว: ต้อง free ทีละแถวก่อนเสมอ */
void free_matrix_ragged(int **mat, size_t rows) {
    if (mat == NULL) {
        return;
    }
    for (size_t i = 0; i < rows; i++) {
        free(mat[i]); /* จุดที่แก้ไข: เพิ่ม loop นี้เข้ามา */
    }
    free(mat);
}

int main(void) {
    size_t rows = 3, cols = 4;
    int **mat = alloc_matrix_ragged(rows, cols);
    if (mat == NULL) {
        return EXIT_FAILURE;
    }
    for (size_t i = 0; i < rows; i++) {
        for (size_t j = 0; j < cols; j++) {
            mat[i][j] = (int)(i * cols + j);
        }
    }
    for (size_t i = 0; i < rows; i++) {
        for (size_t j = 0; j < cols; j++) {
            printf("%3d ", mat[i][j]);
        }
        printf("\n");
    }
    free_matrix_ragged(mat, rows);
    return EXIT_SUCCESS;
}
```

ผลลัพธ์:

```
  0   1   2   3
  4   5   6   7
  8   9  10  11
```

(สามารถตรวจสอบได้ด้วย **Valgrind** ซึ่งจะเรียนใน Part 38 ว่าเวอร์ชันเดิมมี "definitely lost"
เท่ากับขนาดของทุกแถวรวมกัน แต่เวอร์ชันที่แก้แล้วจะไม่มี leak เหลืออยู่เลย — ข้อ 3-6 ให้ผู้เรียน
ลองทำเองก่อน โดยนำโครงสร้างฟังก์ชันจากหัวข้อ 12.3-12.5 มาปรับใช้ได้โดยตรง)

---

## สรุปท้ายบท

ใน Part นี้เราได้:

- เข้าใจว่า Static 2D Array ถูกจัดเก็บในหน่วยความจำแบบ Row-Major Order และคำนวณตำแหน่ง
  ของแต่ละ element ด้วยสูตร `(i * COLS + j) * sizeof(element_type)` ได้ด้วยตัวเอง
- ส่ง 2D Array เป็นพารามิเตอร์ให้ฟังก์ชันได้อย่างถูกต้อง ทั้งแบบขนาดตายตัวและแบบระบุจำนวน
  แถวตอนรันไทม์
- สร้าง Dynamic 2D Array ด้วยเทคนิค Pointer-to-Pointer ได้ทั้งสองแบบ: **Ragged Array**
  (จองแยกแต่ละแถว) และ **Contiguous Allocation** (จองก้อนเดียวต่อเนื่อง) พร้อมเข้าใจข้อดี
  ข้อเสียของแต่ละแบบ
- เขียนโปรแกรม Matrix Multiplication เต็มรูปแบบที่ใช้ Dynamic 2D Array ได้จริง
- ตระหนักถึงข้อผิดพลาดเรื่องการ `free` ที่พบบ่อยเมื่อทำงานกับ 2D Array แบบ dynamic

ทักษะเรื่องการจัดการหน่วยความจำสำหรับข้อมูลที่มีโครงสร้าง (ทั้ง 1D และ 2D) ที่เราสะสมมา
ตลอด Part 11-12 นี้ จะถูกนำไปใช้ต่อยอดใน **Part 13** ซึ่งเราจะเรียนวิธี **อ่านและเขียนข้อมูล
ลงไฟล์** เพื่อให้ข้อมูลที่โปรแกรมสร้างขึ้นสามารถเก็บไว้ใช้ต่อได้แม้ปิดโปรแกรมไปแล้ว

**ต่อไป:** [Part 13 — File I/O ใน C](./part-013-file-io.md)
