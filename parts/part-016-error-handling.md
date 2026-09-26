# Part 16: Error Handling ใน C (Step 121–128)

> Module B — C ระดับกลาง & โครงสร้างข้อมูล/อัลกอริทึม (Intermediate C & Data Structures/Algorithms) | Part 16 จาก 125
> Step ที่ครอบคลุมใน Part นี้: Step 121–128
> Part ก่อนหน้า: [Part 15 — Bit Manipulation](./part-015-bit-manipulation.md) | Part ถัดไป: [Part 17 — Modular Programming](./part-017-modular-programming.md)

## เป้าหมายการเรียนรู้ (Learning Objectives)

หลังจบ Part นี้ ผู้เรียนจะสามารถ:

1. อธิบายธรรมเนียม (Convention) การออกแบบ Return Code แบบ `0 = success` และเลือกใช้
   negative code หรือ enum error code ได้อย่างเหมาะสมกับสถานการณ์
2. ออกแบบ Enum Error Code ของตัวเองที่อ่านง่าย ขยายเพิ่มชนิดใหม่ได้ในอนาคต พร้อมเขียน
   ฟังก์ชันแปลง error code เป็นข้อความอธิบายที่มนุษย์อ่านเข้าใจ
3. ใช้ `errno`, `perror()`, `strerror()` เพื่อตรวจสอบสาเหตุที่แท้จริงของความล้มเหลวจาก
   ฟังก์ชันไลบรารีมาตรฐาน (เช่น `fopen` ที่เปิดไฟล์ไม่สำเร็จ)
4. อธิบายได้ว่าทำไม `errno` มีความหมาย "ถูกต้อง" แค่ทันทีหลังฟังก์ชันที่ล้มเหลวเท่านั้น
   และหลีกเลี่ยงกับดักที่พบบ่อยที่สุดของการใช้งานมันผิดวิธี
5. ใช้ `assert()` อย่างถูกต้อง เข้าใจความแตกต่างระหว่าง Debug Build และ Release Build
   ผ่าน Macro `NDEBUG` และรู้ว่าเมื่อไหร่ **ไม่ควร** ใช้ `assert`
6. เขียนโค้ดแบบ **Defensive Programming** ที่ตรวจสอบ Input, Precondition และ Postcondition
   อย่างเป็นระบบ เพื่อดักจับปัญหาให้เร็วที่สุดเท่าที่จะเป็นไปได้
7. ออกแบบ Pattern การส่งต่อ (Propagate) Error Code ขึ้นไปหลายชั้นฟังก์ชันได้อย่างถูกต้อง
   เพราะภาษา C ไม่มีกลไก Exception เหมือนภาษาสมัยใหม่อื่นๆ
8. ใช้ **`goto` Cleanup Pattern** จัดการ Resource หลายตัว (ไฟล์เปิดค้าง, หน่วยความจำที่
   จองไว้) อย่างปลอดภัยเมื่อเกิด Error กลางทางในฟังก์ชันเดียวกัน

---

## 16.1 ออกแบบ Return Code: `0 = Success` และ Negative Error Code (Step 121)

ตั้งแต่ Part 1 เราเห็นแล้วว่า `main()` คืนค่า `0` เพื่อบอกว่าโปรแกรม "สำเร็จ" — นี่ไม่ใช่
กฎบังคับของภาษา C แต่เป็น **ธรรมเนียม (Convention)** ที่สืบทอดมาจาก Unix ตั้งแต่ยุค 1970
และกลายเป็นมาตรฐานที่ทุกระบบปฏิบัติการยึดถือ: **`0` แปลว่าไม่มีอะไรผิดพลาด ส่วนค่าอื่นที่ไม่ใช่
`0` แปลว่ามีบางอย่างผิดพลาด** (ยิ่งไปกว่านั้น เชลล์สคริปต์และ CI/CD Pipeline ทั้งหมดในโลก
ตรวจสอบความสำเร็จของโปรแกรมจาก Exit Code นี้)

หลักการเดียวกันนี้ถูกนำมาใช้กับ **ฟังก์ชันของเราเองภายในโปรแกรม** ด้วย เพราะภาษา C ไม่มี
กลไก Exception (จะพูดถึงเหตุผลเบื้องหลังในหัวข้อ 16.7) วิธีเดียวที่ฟังก์ชันหนึ่งจะบอกผู้เรียก
ได้ว่า "ฉันทำงานไม่สำเร็จ" คือการ **คืนค่ากลับ (Return Value)** ที่สื่อความหมายนั้น

### รูปแบบที่ 1: Return Code เป็นตัวเลขธรรมดา

```c
/* return_code_basic.c */
#include <stdio.h>

/* คืนค่า 0 = สำเร็จ, ค่าอื่นที่ไม่ใช่ 0 = ล้มเหลว (ตามธรรมเนียม Unix/C) */
int divide_safe(int a, int b, int *result) {
    if (b == 0) {
        return -1;   /* error: หารด้วยศูนย์ */
    }
    *result = a / b;
    return 0;        /* success */
}

int main(void) {
    int result;

    if (divide_safe(10, 2, &result) == 0) {
        printf("10 / 2 = %d\n", result);
    } else {
        printf("เกิดข้อผิดพลาด: หารด้วยศูนย์\n");
    }

    if (divide_safe(10, 0, &result) == 0) {
        printf("10 / 0 = %d\n", result);
    } else {
        printf("เกิดข้อผิดพลาด: หารด้วยศูนย์\n");
    }

    return 0;
}
```

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 return_code_basic.c -o return_code_basic
./return_code_basic
```

```
10 / 2 = 5
เกิดข้อผิดพลาด: หารด้วยศูนย์
```

สังเกตแนวคิดสำคัญที่ต่างจากภาษาที่มี Exception: **ค่าที่ฟังก์ชันคำนวณได้จริง (`result`)
กับสถานะความสำเร็จ/ล้มเหลว ถูกแยกออกจากกันคนละช่องทาง** — ค่าที่คำนวณได้ส่งผ่าน
**Output Parameter** (Pointer ที่รับมาเป็นพารามิเตอร์) ในขณะที่ค่า `return` ของฟังก์ชัน
ถูกสงวนไว้สำหรับบอกสถานะเท่านั้น นี่คือรูปแบบที่พบมากที่สุดในโค้ด C ระดับ Production
(ฟังก์ชันของ POSIX อย่าง `pthread_create`, `sem_init` ก็ใช้แนวทางนี้ทั้งหมด)

### ทำไมนิยมใช้ Negative Number แทน Error แต่ละแบบ

เมื่อฟังก์ชันต้อง **คืนค่าตัวเลขที่มีความหมายจริง** (ไม่ใช่แค่สถานะ) เช่นฟังก์ชันที่คืนจำนวน
ไบต์ที่อ่านได้ (เหมือน `read()` ของ POSIX) ธรรมเนียมที่นิยมคือ: **คืนค่าบวกหรือศูนย์เมื่อสำเร็จ
(เป็นผลลัพธ์จริง) และคืนค่าลบเมื่อล้มเหลว** เพราะจำนวนไบต์/ขนาด/ตำแหน่งในโลกจริงไม่มีทาง
ติดลบอยู่แล้ว การใช้เลขลบจึงเป็น "พื้นที่ว่าง" ที่นำมาใช้บอก error ได้โดยไม่ชนกับค่าที่ถูกต้อง

```c
#include <stdio.h>

/* คืนค่า index ที่เจอ (>= 0) หรือ -1 ถ้าไม่เจอเลย — รูปแบบเดียวกับ strchr/strstr แนวคิด */
int find_char(const char *text, char target) {
    for (int i = 0; text[i] != '\0'; i++) {
        if (text[i] == target) {
            return i;   /* เจอ: คืน index (ไม่มีทางติดลบ) */
        }
    }
    return -1;          /* ไม่เจอ: คืนค่าลบที่ไม่ชนกับ index จริงใดๆ */
}

int main(void) {
    int pos = find_char("Hello, C!", 'C');
    if (pos >= 0) {
        printf("เจอที่ index %d\n", pos);
    } else {
        printf("ไม่เจอตัวอักษรนี้\n");
    }
    return 0;
}
```

### ตารางเปรียบเทียบธรรมเนียม Return Code ที่พบบ่อย

| รูปแบบ | ความหมาย | ตัวอย่างการใช้งานจริง |
|---|---|---|
| `int` แบบ 0/ไม่ใช่ 0 | 0 = สำเร็จ, อื่นๆ = ล้มเหลว (ไม่สนใจค่าที่คำนวณได้) | `main()`, `fclose()`, `pthread_mutex_lock()` |
| `int` แบบบวก/ลบ | >= 0 = ผลลัพธ์จริง, < 0 = error code | `read()`, `write()`, `printf()` (คืนจำนวนตัวอักษร หรือค่าลบถ้า error) |
| Pointer แบบ NULL | Pointer ที่ไม่ใช่ NULL = สำเร็จ, NULL = ล้มเหลว | `malloc()`, `fopen()`, `strchr()` |
| Enum Error Code | ค่าคงที่ที่สื่อความหมายชัดเจน แยกประเภท error ได้หลายแบบ | โค้ดของเราเองในหัวข้อ 16.2 |

> **กฎทองของบทนี้**: ก่อนเขียนฟังก์ชันสักตัว ให้ตัดสินใจล่วงหน้าเสมอว่า "ฟังก์ชันนี้จะบอก
> ผู้เรียกว่าล้มเหลวด้วยวิธีไหน" แล้วเขียน **comment กำกับไว้เหนือฟังก์ชัน** ให้ชัดเจน
> เพราะ C ไม่มีกลไกบังคับให้ผู้เรียกต้องตรวจสอบค่าที่คืนกลับมา (ต่างจากภาษาที่มี exception
> ซึ่งถ้าไม่ catch โปรแกรมจะหยุดทำงานทันที) — การลืมเช็ค return value ใน C จึงเป็นสาเหตุ
> อันดับต้นๆ ของบั๊กที่เงียบและอันตราย

---

## 16.2 Error Code แบบ Enum: ออกแบบให้อ่านง่ายและขยายได้ (Step 122)

การใช้ตัวเลขดิบๆ อย่าง `-1`, `-2`, `-3` มีปัญหาสำคัญ: **อ่านไม่รู้เรื่อง** เมื่อโค้ดมี error
หลายสิบแบบ คนอ่านโค้ดจะจำไม่ได้ว่า `-7` แปลว่าอะไร วิธีแก้ที่เป็นมาตรฐานคือใช้ **`enum`**
กำหนดชื่อที่สื่อความหมายให้กับ error แต่ละแบบ (เราเคยเห็นแนวคิดนี้ผ่านๆ มาแล้วใน Part 10)

```c
/* return_code_enum.c */
#include <stdio.h>

/* Enum error code: อ่านง่ายกว่าตัวเลขดิบๆ มาก และขยายเพิ่มชนิด error ได้ในอนาคต */
typedef enum {
    ERR_OK = 0,
    ERR_DIVIDE_BY_ZERO = 1,
    ERR_OVERFLOW = 2,
    ERR_INVALID_ARGUMENT = 3
} ErrorCode;

/* ฟังก์ชันช่วยแปล ErrorCode เป็นข้อความอ่านง่าย เหมือน strerror() ของ errno */
static const char *error_code_to_string(ErrorCode code) {
    switch (code) {
        case ERR_OK:                 return "สำเร็จ";
        case ERR_DIVIDE_BY_ZERO:     return "หารด้วยศูนย์";
        case ERR_OVERFLOW:           return "ค่าล้น (overflow)";
        case ERR_INVALID_ARGUMENT:   return "พารามิเตอร์ไม่ถูกต้อง";
        default:                     return "ข้อผิดพลาดที่ไม่รู้จัก";
    }
}

static ErrorCode divide_safe(int a, int b, int *result) {
    if (result == NULL) {
        return ERR_INVALID_ARGUMENT;
    }
    if (b == 0) {
        return ERR_DIVIDE_BY_ZERO;
    }
    *result = a / b;
    return ERR_OK;
}

int main(void) {
    int result;
    ErrorCode err;

    err = divide_safe(20, 4, &result);
    if (err == ERR_OK) {
        printf("20 / 4 = %d\n", result);
    } else {
        printf("ผิดพลาด: %s\n", error_code_to_string(err));
    }

    err = divide_safe(20, 0, &result);
    if (err != ERR_OK) {
        printf("ผิดพลาด: %s\n", error_code_to_string(err));
    }

    return 0;
}
```

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 return_code_enum.c -o return_code_enum
./return_code_enum
```

```
20 / 4 = 5
ผิดพลาด: หารด้วยศูนย์
```

### หลักการออกแบบ Enum Error Code ที่ดี

1. **ให้ `ERR_OK` (หรือ `SUCCESS`) มีค่าเป็น `0` เสมอ** เพื่อให้เขียน `if (err)` เพื่อเช็คว่า
   "มี error หรือไม่" ได้สั้นๆ (แม้จะยังแนะนำให้เขียน `if (err != ERR_OK)` แบบเต็มเพื่อความ
   ชัดเจนก็ตาม) และให้สอดคล้องกับธรรมเนียม 0 = success ที่ใช้ทั่วทั้งภาษา C
2. **ตั้งชื่อด้วย prefix เดียวกันเสมอ** เช่น `ERR_` เพื่อให้กด auto-complete ใน editor แล้ว
   เห็นรายการ error ทั้งหมดได้ในที่เดียว
3. **เขียนฟังก์ชันแปลงเป็นข้อความกำกับไว้เสมอ** (เหมือน `error_code_to_string` ด้านบน)
   เพื่อให้ debug และแสดงข้อความแก่ผู้ใช้ได้ง่าย โดยไม่ต้องเปิดไฟล์ header ไปดูว่าค่า
   ตัวเลขแต่ละตัวแปลว่าอะไร
4. **ใส่ `default` case ใน `switch` เสมอ** เผื่อกรณีมีการเพิ่ม error code ใหม่ในอนาคตแต่ลืม
   อัพเดตฟังก์ชันแปลงข้อความ โปรแกรมจะไม่พังแต่จะแสดงข้อความ fallback แทน
5. **แยก Error Code ตาม "โมดูล" หรือ "ชั้นของระบบ"** เมื่อโปรเจกต์ใหญ่ขึ้น เช่น
   `FILE_ERR_*` สำหรับ error เกี่ยวกับไฟล์ และ `NET_ERR_*` สำหรับ error เกี่ยวกับ network
   แยกกันคนละ enum เพื่อไม่ให้ enum เดียวใหญ่เทอะทะเกินไป

---

## 16.3 `errno`, `perror()`, `strerror()`: รู้สาเหตุจริงจากไลบรารีมาตรฐาน (Step 123)

Return Code ที่เราออกแบบเองบอกได้แค่ "ฟังก์ชันของเราล้มเหลว" แต่เมื่อเรียกใช้ **ฟังก์ชัน
ของไลบรารีมาตรฐาน** (เช่น `fopen`, `malloc`, `fread`) ที่คืนค่าแค่ `NULL` หรือ `-1` เราจะไม่รู้
"สาเหตุที่แท้จริง" ว่าทำไมมันล้มเหลว — นี่คือหน้าที่ของตัวแปรพิเศษชื่อ **`errno`**

`errno` เป็นตัวแปร Global (ประกาศอยู่ใน `<errno.h>`) ที่ไลบรารีมาตรฐานของ C และระบบ
ปฏิบัติการ (POSIX) ใช้ร่วมกันเพื่อรายงาน "รหัสข้อผิดพลาดล่าสุด" เมื่อฟังก์ชันในกลุ่มนี้ล้มเหลว
มันจะเซ็ตค่า `errno` ให้เป็นรหัสตัวเลขที่บอกสาเหตุอย่างเจาะจง

### ตัวอย่างจริง: `fopen()` ล้มเหลวเมื่อเปิดไฟล์ที่ไม่มีอยู่จริง

```c
/* errno_perror.c */
#include <stdio.h>
#include <errno.h>

int main(void) {
    FILE *fp = fopen("/path/that/does/not/exist/data.txt", "r");

    if (fp == NULL) {
        /* เก็บค่า errno ไว้ในตัวแปรทันที ก่อนเรียกฟังก์ชันอื่นใดๆ ต่อ (แม้แต่ perror เอง) */
        int saved_errno = errno;

        /* perror พิมพ์ข้อความที่เราตั้งเอง ตามด้วย ": " แล้วตามด้วยข้อความอธิบาย errno */
        perror("fopen ล้มเหลว");
        printf("errno ที่บันทึกไว้ = %d\n", saved_errno);
        return 1;
    }

    fclose(fp);
    return 0;
}
```

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 errno_perror.c -o errno_perror
./errno_perror
echo "exit code: $?"
```

```
fopen ล้มเหลว: No such file or directory
errno ที่บันทึกไว้ = 2
exit code: 1
```

`perror("fopen ล้มเหลว")` พิมพ์ข้อความของเราตามด้วย `: ` แล้วต่อด้วยคำอธิบายของ `errno`
ปัจจุบันโดยอัตโนมัติ (ในที่นี้คือ `ENOENT` ซึ่งมีค่าตัวเลข `2` แปลว่า "No such file or directory")
สังเกตว่าเราเก็บค่า `errno` ใส่ตัวแปร `saved_errno` **ทันที** ก่อนเรียกฟังก์ชันอื่นต่อ — เหตุผล
ว่าทำไมต้องระวังเรื่องนี้ขนาดนี้ จะอธิบายละเอียดในหัวข้อถัดไป

### `strerror()`: แปลง errno เป็นข้อความโดยควบคุมรูปแบบเอง

ถ้าต้องการ **จัดรูปแบบข้อความเอง** (ไม่ใช่แค่พิมพ์ตรงๆ แบบ `perror`) ให้ใช้ `strerror()`
จาก `<string.h>` ซึ่งรับค่า errno แล้วคืน string ที่อธิบายความหมายกลับมา

```c
/* errno_strerror.c */
#include <stdio.h>
#include <string.h>
#include <errno.h>

int main(void) {
    FILE *fp = fopen("/path/that/does/not/exist/data.txt", "r");

    if (fp == NULL) {
        /* strerror(errno) คืน string ที่อธิบาย errno เป็นข้อความ
         * ให้ควบคุมการจัดรูปแบบข้อความเองได้ (ต่างจาก perror ที่พิมพ์ให้ตรงๆ) */
        fprintf(stderr, "ไม่สามารถเปิดไฟล์ได้: %s (errno=%d)\n",
                strerror(errno), errno);
        return 1;
    }

    fclose(fp);
    return 0;
}
```

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 errno_strerror.c -o errno_strerror
./errno_strerror
```

```
ไม่สามารถเปิดไฟล์ได้: No such file or directory (errno=2)
```

ตัวอย่างนี้ปลอดภัยเพราะ `strerror(errno)` และ `errno` ตัวที่สองถูก**ประเมินค่าเป็นอาร์กิวเมนต์
ก่อนที่ `fprintf` จะเริ่มทำงานจริง** จึงยังเป็นค่าเดียวกันกับตอน `fopen` ล้มเหลวอยู่ ไม่มีฟังก์ชัน
อื่นแทรกมาเปลี่ยนค่าคั่นกลาง

### รหัส errno ที่พบบ่อยที่สุด

| ชื่อ Macro | ค่า (Linux x86-64) | ความหมาย | พบบ่อยตอนไหน |
|---|---|---|---|
| `EPERM` | 1 | Operation not permitted | ไม่มีสิทธิ์ทำ operation นั้น |
| `ENOENT` | 2 | No such file or directory | เปิดไฟล์/โฟลเดอร์ที่ไม่มีอยู่จริง |
| `EBADF` | 9 | Bad file number | ใช้ file descriptor ที่ปิดไปแล้วหรือไม่ถูกต้อง |
| `ENOMEM` | 12 | Out of memory | `malloc`/`calloc` จองหน่วยความจำไม่สำเร็จ |
| `EACCES` | 13 | Permission denied | ไม่มีสิทธิ์อ่าน/เขียนไฟล์นั้น |
| `EEXIST` | 17 | File exists | สร้างไฟล์/โฟลเดอร์ที่มีอยู่แล้วซ้ำ |
| `EINVAL` | 22 | Invalid argument | ส่งพารามิเตอร์ที่ไม่ถูกต้องให้ system call |
| `EMFILE` | 24 | Too many open files | เปิดไฟล์เกินขีดจำกัดของโปรเซส |
| `ENOSPC` | 28 | No space left on device | พื้นที่ดิสก์เต็ม |

> ค่าตัวเลขจริงของแต่ละ Macro เป็น **Implementation-Defined** (อาจต่างกันไปในแต่ละระบบ
> ปฏิบัติการ) ค่าที่แสดงในตารางนี้มาจากการตรวจสอบ `<asm-generic/errno-base.h>` บน Linux
> จริง — **ห้าม hardcode ตัวเลขเหล่านี้ในโค้ดเด็ดขาด ให้ใช้ชื่อ Macro เสมอ** (เช่น
> `if (errno == ENOENT)` ไม่ใช่ `if (errno == 2)`) เพื่อให้โค้ด portable ข้ามระบบ

---

## 16.4 กับดักของ `errno`: ใช้ได้แค่ "ทันที" หลังฟังก์ชันล้มเหลวเท่านั้น (Step 124)

นี่คือกฎที่สำคัญที่สุดข้อเดียวเกี่ยวกับ `errno` ที่ถ้าลืมจะทำให้ debug ยากมาก:

> **`errno` มีค่าที่เชื่อถือได้ก็ต่อเมื่ออ่านทันทีหลังจากฟังก์ชันที่รายงานว่าล้มเหลว (คืนค่า
> `NULL`/`-1`/ค่าที่บ่งบอก error) เท่านั้น** มาตรฐาน C รับประกันแค่ว่า `errno` **จะถูกเซ็ต**
> เมื่อฟังก์ชันล้มเหลว แต่ **ไม่รับประกันว่าจะถูกล้างกลับเป็น 0 เมื่อฟังก์ชันตัวต่อไปสำเร็จ**
> และที่อันตรายกว่านั้นคือ **ฟังก์ชันไลบรารีตัวอื่นที่ดูเหมือน "ทำงานสำเร็จ" ก็อาจเปลี่ยนค่า
> `errno` เป็นผลข้างเคียงภายในได้เช่นกัน** แม้จะไม่ได้รายงานว่าตัวเองล้มเหลวก็ตาม

ตัวอย่างต่อไปนี้พิสูจน์กับดักนี้ด้วยการรันจริงบนเครื่อง — ผลลัพธ์ที่เห็นคือของจริง ไม่ใช่
การจำลอง:

```c
/* errno_pitfall.c
 * สาธิตข้อผิดพลาดที่พบบ่อยและ "เห็นผลจริง" บนระบบนี้: การอ่าน errno อีกครั้งหลังจากเรียก
 * ฟังก์ชันไลบรารีตัวอื่นคั่นกลาง แม้แต่ perror() เอง ก็เป็นฟังก์ชันที่ "อาจ" เปลี่ยนค่า errno
 * ได้ระหว่างทำงานภายใน (เช่น glibc บางเวอร์ชันต้องเปิดไฟล์ locale message catalog)
 */
#include <stdio.h>
#include <errno.h>

int main(void) {
    FILE *fp = fopen("/path/that/does/not/exist/data.txt", "r");

    if (fp == NULL) {
        int errno_right_after_fopen = errno;   /* ถูกต้อง: อ่านทันที */

        perror("เปิดไฟล์ไม่สำเร็จ");            /* พิมพ์ข้อความถูกต้อง เพราะอ่าน errno ตอนต้น */

        int errno_after_perror = errno;        /* ผิดพลาด: เชื่อว่า errno ยังเป็นค่าเดิม */

        printf("errno ทันทีหลัง fopen()  = %d\n", errno_right_after_fopen);
        printf("errno หลังเรียก perror() = %d  <-- อาจไม่ใช่ค่าจาก fopen() อีกต่อไปแล้ว!\n",
               errno_after_perror);
    }

    if (fp != NULL) {
        fclose(fp);
    }
    return 0;
}
```

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 errno_pitfall.c -o errno_pitfall
./errno_pitfall
```

```
เปิดไฟล์ไม่สำเร็จ: No such file or directory
errno ทันทีหลัง fopen()  = 2
errno หลังเรียก perror() = 22  <-- อาจไม่ใช่ค่าจาก fopen() อีกต่อไปแล้ว!
```

ผลลัพธ์จริงบนเครื่องที่ใช้เขียนบทเรียนนี้แสดงให้เห็นชัดเจน: `errno` มีค่า `2` (`ENOENT`)
ทันทีหลัง `fopen()` ล้มเหลว ตรงกับข้อความ "No such file or directory" ที่ `perror()` พิมพ์
ออกมาถูกต้อง (เพราะ `perror` อ่านค่า `errno` ที่ถูกต้องไปใช้ตอนต้นของมันเอง) **แต่หลังจาก
`perror()` ทำงานเสร็จแล้ว ค่า `errno` กลับกลายเป็น `22` (`EINVAL`) ไปแล้ว** — สาเหตุคือ
`perror` ในบาง implementation ของ glibc ต้องเรียกฟังก์ชันภายในเพิ่มเติม (เช่นเกี่ยวกับ locale
message catalog) ซึ่งอาจล้มเหลวเงียบๆ และเซ็ต `errno` ทับค่าเดิมไปโดยที่เราไม่รู้ตัว แม้ตัว
`perror` เองจะทำงาน "สำเร็จ" ตามหน้าที่หลักของมัน (พิมพ์ข้อความถูกต้อง) ก็ตาม

### ข้อสรุปเชิงปฏิบัติ

**เก็บค่า `errno` ใส่ตัวแปรท้องถิ่นทันทีที่รู้ว่าฟังก์ชันล้มเหลว ก่อนเรียกฟังก์ชันอื่นใดๆ
ต่อ (รวมถึง `perror` หรือแม้แต่ `printf` ธรรมดาที่ปลอดภัยกว่าแต่ก็ไม่ควรเสี่ยง) แล้วใช้
ตัวแปรที่เก็บไว้นั้นในการตัดสินใจหรือรายงานผลต่อไป** นี่คือเหตุผลที่ตัวอย่าง `errno_perror.c`
ในหัวข้อ 16.3 เขียนแบบเก็บ `saved_errno` ไว้ก่อนเรียก `perror()` เสมอ

| สถานการณ์ | errno เชื่อถือได้หรือไม่ |
|---|---|
| อ่าน `errno` ทันทีบรรทัดถัดไปหลังฟังก์ชันล้มเหลว | เชื่อถือได้ |
| เก็บ `errno` ใส่ตัวแปรทันที แล้วใช้ตัวแปรนั้นต่อไปเรื่อยๆ | เชื่อถือได้ |
| อ่าน `errno` หลังเรียกฟังก์ชันไลบรารีอื่นคั่นกลาง (แม้จะดู "ไม่เกี่ยวกับ error") | **ไม่เชื่อถือได้** |
| อ่าน `errno` เพื่อเช็คว่า "ฟังก์ชันก่อนหน้าสำเร็จหรือไม่" โดยไม่เช็ค return value ก่อน | **ผิดหลักการเลย** — ต้องเช็ค return value ก่อนเสมอ `errno` เป็นแค่ตัวบอก "สาเหตุ" ไม่ใช่ตัวบอก "สำเร็จ/ล้มเหลว" |

---

## 16.5 `assert()`: ใช้เมื่อไหร่ และทำไมหายไปใน Release Build (Step 125)

`assert()` จาก `<assert.h>` เป็น Macro ที่ตรวจสอบว่าเงื่อนไขที่ระบุเป็นจริงหรือไม่ **ถ้าเป็น
เท็จ โปรแกรมจะพิมพ์ข้อความบอกไฟล์/บรรทัด/เงื่อนไขที่ผิดพลาด แล้วเรียก `abort()` หยุดโปรแกรม
ทันที** มันถูกออกแบบมาเพื่อดักจับ **บั๊กของโปรแกรมเมอร์เอง** (เช่นละเมิด precondition ที่
ตัวเองกำหนดไว้) ไม่ใช่เพื่อจัดการ error ที่มาจากภายนอก (เช่น input จากผู้ใช้ หรือไฟล์ที่หายไป)

```c
/* assert_demo.c */
#include <stdio.h>
#include <assert.h>

/* precondition: n ต้องไม่ติดลบ (เป็น "สัญญา" ระหว่างผู้เรียกกับฟังก์ชันนี้) */
static long factorial(int n) {
    assert(n >= 0 && "factorial: n ต้องไม่ติดลบ");

    long result = 1;
    for (int i = 2; i <= n; i++) {
        result *= i;
    }
    return result;
}

int main(void) {
    printf("5! = %ld\n", factorial(5));
    printf("0! = %ld\n", factorial(0));

    /* ลองปลดคอมเมนต์บรรทัดล่างนี้เพื่อดู assert ทำงานตอน n ติดลบ:
     * printf("%ld\n", factorial(-3));
     * โปรแกรมจะหยุดทันทีพร้อมข้อความบอกไฟล์/บรรทัด/เงื่อนไขที่ผิดพลาด
     */

    return 0;
}
```

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 assert_demo.c -o assert_demo
./assert_demo
```

```
5! = 120
0! = 1
```

เทคนิค `assert(n >= 0 && "factorial: n ต้องไม่ติดลบ")` เป็นลูกเล่นที่นิยมกันมาก: เพราะ
`"ข้อความ"` (string literal) มีค่าความจริงเป็น "จริง" เสมอ (pointer ที่ไม่ใช่ NULL) การเขียน
`เงื่อนไข && "ข้อความ"` จึงไม่เปลี่ยนผลลัพธ์ของเงื่อนไขเลย แต่เมื่อ assert ล้มเหลว ข้อความ
จะถูกพิมพ์ออกมาด้วย ทำให้ debug ง่ายขึ้นมากโดยไม่ต้องเปิดโค้ดไปดูว่าบรรทัดนั้นตรวจสอบอะไร

### ดูข้อความจริงตอน assert ล้มเหลว

```c
/* assert_trigger.c — จงใจให้ assert ล้มเหลว เพื่อดูข้อความจริงที่ได้ */
#include <stdio.h>
#include <assert.h>

static long factorial(int n) {
    assert(n >= 0 && "factorial: n ต้องไม่ติดลบ");

    long result = 1;
    for (int i = 2; i <= n; i++) {
        result *= i;
    }
    return result;
}

int main(void) {
    /* stdout เป็น buffered stream ตามปกติ (ไม่ใช่ terminal แบบ interactive เสมอไป) และ
     * abort() ที่ assert() เรียกภายในจะไม่ flush buffer ให้ก่อนหยุดโปรแกรม จึงต้อง fflush
     * เองเพื่อให้ข้อความที่พิมพ์ไปก่อนหน้าปรากฏออกมาจริงก่อนโปรแกรมหยุดทำงาน */
    printf("กำลังเรียก factorial(-3)...\n");
    fflush(stdout);
    printf("%ld\n", factorial(-3));
    return 0;
}
```

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 assert_trigger.c -o assert_trigger
./assert_trigger
echo "exit code: $?"
```

```
กำลังเรียก factorial(-3)...
assert_trigger: assert_trigger.c:6: factorial: Assertion `n >= 0 && "factorial: n ต้องไม่ติดลบ"' failed.
exit code: 134
```

โปรแกรมหยุดทันทีด้วย Signal `SIGABRT` (exit code `134` = `128 + 6`, เลข `6` คือหมายเลขของ
`SIGABRT`) พร้อมบอกชัดเจนว่าไฟล์ไหน บรรทัดไหน ฟังก์ชันไหน และเงื่อนไขอะไรที่ผิดพลาด — มี
ประโยชน์มากตอนพัฒนา เพราะเรารู้ทันทีว่าโค้ดส่วนไหนละเมิดสมมติฐานของตัวเอง (สังเกตว่าเราต้อง
`fflush(stdout)` ก่อนบรรทัดที่อาจ crash ด้วย เพราะ `abort()` ที่ `assert` เรียกใช้ภายในจะไม่
flush buffer ของ `stdout` ให้ก่อนหยุดโปรแกรม ถ้าไม่ flush เอง ข้อความที่ print ไปก่อนหน้าอาจ
หายไปเงียบๆ เมื่อโปรแกรมไม่ได้รันอยู่บน terminal แบบ interactive)

### `NDEBUG`: ทำไม assert หายไปในโปรแกรม Release

จุดที่ต้องเข้าใจให้ลึกที่สุดคือ: **ถ้า compile ด้วย macro `NDEBUG` ถูกกำหนดไว้ ทุก `assert()`
ในโปรแกรมจะถูกลบออกไปทั้งหมดโดย Preprocessor ราวกับไม่เคยเขียนมันเลย** (ไม่ใช่แค่ "ปิด
การทำงาน" แต่เนื้อโค้ดหายไปจริงๆ) นี่คือพฤติกรรมที่มาตรฐาน C กำหนดไว้ตายตัว

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 -DNDEBUG assert_trigger.c -o assert_trigger_ndebug
./assert_trigger_ndebug
echo "exit code: $?"
```

```
กำลังเรียก factorial(-3)...
1
exit code: 0
```

ผลลัพธ์นี้คือหัวใจของปัญหา: เมื่อ compile ด้วย `-DNDEBUG` โปรแกรมไม่หยุดทำงานอีกต่อไป
แต่กลับ **คืนค่า `1` แบบเงียบๆ** (เพราะลูป `for (i = 2; i <= n; i++)` ไม่ทำงานเลยเมื่อ
`n = -3` จึงคืนค่าเริ่มต้น `result = 1` ซึ่งเป็นค่าที่ **ผิดทางคณิตศาสตร์โดยสิ้นเชิง** สำหรับ
`(-3)!`) โปรแกรมไม่พัง แต่ให้ผลลัพธ์ผิดแบบเงียบๆ — อันตรายกว่าการพังตรงๆ เสียอีก

โดยทั่วไป **Debug Build** (ที่ใช้ระหว่างพัฒนา, ทดสอบ) จะ **ไม่กำหนด** `NDEBUG` เพื่อให้
`assert` ทำงานเต็มที่คอยจับบั๊ก ส่วน **Release Build** (ที่จะส่งให้ผู้ใช้จริง) มักกำหนด
`NDEBUG` เพื่อ (1) ตัดค่าใช้จ่ายด้าน performance จากการเช็คเงื่อนไขซ้ำๆ ทุกครั้งที่ฟังก์ชัน
ทำงาน และ (2) ป้องกันไม่ให้โปรแกรม "หยุดกะทันหัน" ต่อหน้าผู้ใช้จากเงื่อนไขที่เป็นแค่การ
debug ภายใน — Makefile/Build System ระดับ production (จะเรียนละเอียดใน Part 18) มักตั้ง
`CFLAGS` ต่างกันสองชุดสำหรับสองโหมดนี้โดยเฉพาะ

### ตาราง: `assert()` เทียบกับการเช็คเงื่อนไขด้วย `if`

| ประเด็น | `assert(condition)` | `if (condition) { ... }` |
|---|---|---|
| หายไปเมื่อ compile ด้วย `NDEBUG` | ใช่ — หายไปทั้งหมด | ไม่ — ยังทำงานเสมอไม่ว่า build แบบไหน |
| เหมาะกับตรวจสอบอะไร | บั๊กของโปรแกรมเมอร์เอง (precondition, invariant ภายใน) | ข้อมูลจากภายนอกโปรแกรม (input ผู้ใช้, ไฟล์, network, ค่าที่คำนวณแล้วอาจผิดพลาดได้จริง) |
| พฤติกรรมเมื่อเงื่อนไขเป็นเท็จ | หยุดโปรแกรมทันทีด้วย `abort()` | โปรแกรมยังทำงานต่อได้ตามที่เขียนไว้ใน branch |
| ใช้กับ Input ของผู้ใช้ได้ไหม | **ห้ามใช้เด็ดขาด** — จะหายไปใน release แล้วเปิดช่องให้ input ที่ผิดพลาดหลุดเข้าไปโดยไม่ถูกตรวจสอบเลย | ใช่ — เป็นวิธีที่ถูกต้องเสมอ |

> **กฎทองข้อที่สองของบทนี้**: **ห้ามใช้ `assert()` ตรวจสอบ Input จากผู้ใช้, ไฟล์, Network,
> หรือค่าที่ระบบภายนอกส่งมาเด็ดขาด** เพราะเมื่อ compile แบบ Release ด้วย `NDEBUG` เงื่อนไข
> เหล่านั้นจะไม่ถูกตรวจสอบเลย โปรแกรมจะยอมรับข้อมูลผิดๆ เข้าไปประมวลผลต่อโดยไม่รู้ตัว ซึ่ง
> อาจกลายเป็นช่องโหว่ด้านความปลอดภัยร้ายแรงได้ ให้ใช้ `assert` เฉพาะกับสิ่งที่ **ถ้าเกิดขึ้น
> แปลว่าโค้ดของเราเองมีบั๊กแน่นอน** เท่านั้น ส่วนการตรวจสอบข้อมูลจากภายนอกให้ใช้ `if` และ
> คืน error code แบบปกติเสมอ (ตามที่จะเรียนต่อในหัวข้อ 16.6)

---

## 16.6 Defensive Programming: ตรวจ Input, Precondition, Postcondition (Step 126)

**Defensive Programming** คือแนวคิดการเขียนโค้ดโดย "ไม่ไว้ใจ" ข้อมูลที่ไหลเข้ามาในฟังก์ชัน
เลย ไม่ว่าจะมาจากผู้ใช้, ไฟล์, เครือข่าย, หรือแม้แต่จากฟังก์ชันอื่นในโปรแกรมเดียวกัน — ตรวจสอบ
ทุกอย่างที่ตรวจสอบได้ **ก่อน** ที่จะนำไปใช้งานจริง เพื่อดักจับปัญหาให้เร็วที่สุดเท่าที่จะเป็นไปได้
(ยิ่งดักได้เร็ว ยิ่งง่ายต่อการหาสาเหตุ เทียบกับปล่อยให้ข้อมูลผิดพลาดไหลลึกเข้าไปในระบบแล้วค่อย
พังที่จุดอื่นซึ่งไกลจากต้นเหตุจริงมาก)

แนวคิดสำคัญสองคำที่ต้องเข้าใจ:

- **Precondition (เงื่อนไขก่อนทำงาน)**: สิ่งที่ต้องเป็นจริง **ก่อน** ฟังก์ชันเริ่มทำงาน
  เช่น "พารามิเตอร์ต้องไม่เป็น NULL", "ค่าที่ส่งเข้ามาต้องอยู่ในช่วงที่กำหนด"
- **Postcondition (เงื่อนไขหลังทำงาน)**: สิ่งที่ฟังก์ชัน **รับประกัน** ว่าจะเป็นจริงเสมอ
  หลังทำงานเสร็จ (ถ้าคืนค่าสำเร็จ) เช่น "string ปลายทางจะเป็น null-terminated เสมอ"

```c
/* defensive_programming.c */
#include <stdio.h>
#include <string.h>

typedef enum {
    ERR_OK = 0,
    ERR_NULL_POINTER = 1,
    ERR_BUFFER_TOO_SMALL = 2,
    ERR_EMPTY_INPUT = 3
} ErrorCode;

/*
 * copy_trimmed: คัดลอกข้อความจาก src ไปยัง dest โดยตัดช่องว่างหน้า-หลังออก
 *
 * precondition:
 *   - dest และ src ต้องไม่เป็น NULL
 *   - dest_size ต้องมากพอสำหรับความยาวข้อความ + 1 (สำหรับ '\0')
 * postcondition:
 *   - ถ้าคืนค่า ERR_OK, dest จะเป็น null-terminated string เสมอ
 */
static ErrorCode copy_trimmed(char *dest, size_t dest_size, const char *src) {
    /* Defensive check 1: ตรวจ NULL pointer ก่อนใช้งานเสมอ */
    if (dest == NULL || src == NULL) {
        return ERR_NULL_POINTER;
    }
    if (dest_size == 0) {
        return ERR_BUFFER_TOO_SMALL;
    }

    /* หาจุดเริ่มต้น (ข้ามช่องว่างหน้า) */
    while (*src == ' ' || *src == '\t') {
        src++;
    }

    size_t len = strlen(src);
    /* หาจุดสิ้นสุด (ตัดช่องว่างท้าย) */
    while (len > 0 && (src[len - 1] == ' ' || src[len - 1] == '\t')) {
        len--;
    }

    if (len == 0) {
        return ERR_EMPTY_INPUT;
    }
    /* Defensive check 2: ตรวจสอบว่า buffer ปลายทางใหญ่พอ ก่อน copy จริง */
    if (len + 1 > dest_size) {
        return ERR_BUFFER_TOO_SMALL;
    }

    memcpy(dest, src, len);
    dest[len] = '\0';   /* postcondition: รับประกันว่า null-terminated เสมอ */
    return ERR_OK;
}

static void print_result(ErrorCode err, const char *dest) {
    switch (err) {
        case ERR_OK:
            printf("ผลลัพธ์: \"%s\"\n", dest);
            break;
        case ERR_NULL_POINTER:
            printf("ผิดพลาด: pointer เป็น NULL\n");
            break;
        case ERR_BUFFER_TOO_SMALL:
            printf("ผิดพลาด: buffer เล็กเกินไป\n");
            break;
        case ERR_EMPTY_INPUT:
            printf("ผิดพลาด: ข้อความว่างเปล่าหลังตัดช่องว่าง\n");
            break;
        default:
            printf("ผิดพลาดที่ไม่รู้จัก\n");
            break;
    }
}

int main(void) {
    char buffer[16];
    ErrorCode err;

    err = copy_trimmed(buffer, sizeof(buffer), "  Hello C  ");
    print_result(err, buffer);

    err = copy_trimmed(buffer, sizeof(buffer), "ข้อความยาวเกินไปสำหรับ buffer เล็กๆ นี้แน่นอน");
    print_result(err, buffer);

    err = copy_trimmed(buffer, sizeof(buffer), "   ");
    print_result(err, buffer);

    err = copy_trimmed(buffer, sizeof(buffer), NULL);
    print_result(err, buffer);

    return 0;
}
```

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 defensive_programming.c -o defensive_programming
./defensive_programming
```

```
ผลลัพธ์: "Hello C"
ผิดพลาด: buffer เล็กเกินไป
ผิดพลาด: ข้อความว่างเปล่าหลังตัดช่องว่าง
ผิดพลาด: pointer เป็น NULL
```

สังเกตว่าโค้ดนี้ **ไม่ใช้ `assert` เลยแม้แต่จุดเดียว** เพราะทุกเงื่อนไขที่ตรวจสอบ (`NULL`,
ขนาด buffer, ข้อความว่าง) ล้วนเป็นสิ่งที่ **อาจเกิดขึ้นได้จริงจากข้อมูลภายนอก** ไม่ใช่บั๊กของ
โปรแกรมเมอร์ จึงต้องใช้ `if` และคืน `ErrorCode` เพื่อให้ผู้เรียกจัดการต่อได้อย่างเหมาะสมเสมอ
ไม่ว่าจะ compile แบบ Debug หรือ Release ก็ตาม — นี่คือความแตกต่างสำคัญที่สุดระหว่างหัวข้อ
16.5 กับ 16.6

### หลักปฏิบัติของ Defensive Programming ที่ควรทำเป็นนิสัย

1. **ตรวจสอบ Pointer parameter ทุกตัวว่าไม่เป็น `NULL` ก่อนใช้งาน** เว้นแต่ฟังก์ชันนั้นจะ
   ระบุชัดเจนในเอกสาร (comment) ว่ารับประกันว่าผู้เรียกจะไม่ส่ง `NULL` มา (แล้วอาจใช้
   `assert` แทนได้ในกรณีนั้น เพราะกลายเป็น "สัญญาระหว่างนักพัฒนา" ไม่ใช่ input ภายนอก)
2. **ตรวจสอบขอบเขตของ Array/Buffer ก่อนเขียนหรืออ่านเสมอ** อย่าไว้ใจว่าขนาดที่คำนวณมา
   จะพอดีเสมอไป
3. **ตรวจสอบค่าที่ได้จาก `scanf`, `fgets`, `atoi` และฟังก์ชันแปลงข้อมูลอื่นๆ ทุกครั้ง**
   เพราะ input จากผู้ใช้ไม่มีทางเชื่อถือได้ 100%
4. **เขียน comment บอก Precondition/Postcondition ไว้เหนือฟังก์ชันเสมอ** (เหมือนตัวอย่าง
   `copy_trimmed` ด้านบน) เพื่อให้คนอื่น (หรือตัวเราเองในอนาคต) เข้าใจ "สัญญา" ของฟังก์ชัน
   นี้ได้ทันทีโดยไม่ต้องอ่านทั้ง implementation
5. **Fail Fast**: เมื่อเจอเงื่อนไขที่ผิดพลาด ให้คืน error code หรือหยุดทำงาน **ทันที** อย่า
   ปล่อยให้โค้ดทำงานต่อด้วยข้อมูลที่รู้อยู่แล้วว่าผิดพลาด เพราะจะทำให้ปัญหาลุกลามไปไกลจาก
   ต้นเหตุ ยากต่อการสืบสาเหตุภายหลัง

---

## 16.7 Propagate Error ขึ้นหลายชั้นฟังก์ชัน: เพราะ C ไม่มี Exception (Step 127)

ภาษาสมัยใหม่อย่าง C++, Java, Python มีกลไก **Exception** ที่ให้ error "โยน" (throw) ขึ้นไป
ข้ามหลายชั้นฟังก์ชันได้เองโดยอัตโนมัติ จนกว่าจะมีจุดใดจุดหนึ่ง `catch` มันไว้ แต่ **ภาษา C
ไม่มีกลไกนี้เลย** — ทุกฟังก์ชันในห่วงโซ่การเรียก (Call Chain) ต้อง **ตรวจสอบ return value
ของฟังก์ชันที่ตัวเองเรียก แล้วส่งต่อ (Propagate) สถานะ error นั้นขึ้นไปให้ผู้เรียกของตัวเองเอง
ด้วยมือทุกชั้น** ไม่มีทางลัด

```c
/* propagate_error.c
 * สาธิตการ "ส่งต่อ error ขึ้นไปหลายชั้นฟังก์ชัน" ใน C
 * เพราะ C ไม่มี exception การจะแจ้งให้ผู้เรียกชั้นบนสุดรู้ว่ามีปัญหาเกิดขึ้นที่ชั้นล่างสุด
 * ต้อง "ส่งต่อ (propagate)" ค่า error code ผ่านทุกชั้นของฟังก์ชันที่เรียกต่อกันมา
 */
#include <stdio.h>
#include <string.h>

#define PERSON_NAME_MAX 32

typedef enum {
    ERR_OK = 0,
    ERR_INVALID_AGE = 1,
    ERR_NAME_TOO_LONG = 2,
    ERR_NULL_POINTER = 3
} ErrorCode;

typedef struct {
    char name[PERSON_NAME_MAX];
    int age;
} Person;

/* ชั้นที่ 1 (ล่างสุด): ตรวจสอบอายุ */
static ErrorCode validate_age(int age) {
    if (age < 0 || age > 150) {
        return ERR_INVALID_AGE;
    }
    return ERR_OK;
}

/* ชั้นที่ 1 (ล่างสุด): ตรวจสอบชื่อ */
static ErrorCode validate_name(const char *name) {
    if (name == NULL) {
        return ERR_NULL_POINTER;
    }
    if (strlen(name) >= PERSON_NAME_MAX) {
        return ERR_NAME_TOO_LONG;
    }
    return ERR_OK;
}

/* ชั้นที่ 2 (กลาง): เรียกใช้ทั้งสองฟังก์ชันตรวจสอบ แล้ว "ส่งต่อ" error ตัวแรกที่เจอ */
static ErrorCode create_person(Person *out, const char *name, int age) {
    if (out == NULL) {
        return ERR_NULL_POINTER;
    }

    ErrorCode err = validate_name(name);
    if (err != ERR_OK) {
        return err;   /* ส่งต่อ error ขึ้นไปทันที ไม่ทำงานต่อ */
    }

    err = validate_age(age);
    if (err != ERR_OK) {
        return err;   /* ส่งต่อ error ขึ้นไปทันที */
    }

    strcpy(out->name, name);
    out->age = age;
    return ERR_OK;
}

/* ชั้นที่ 3 (บนสุด): register_person เรียก create_person และส่งต่อ error ต่อไปอีกชั้น */
static ErrorCode register_person(Person *out, const char *name, int age) {
    ErrorCode err = create_person(out, name, age);
    if (err != ERR_OK) {
        /* ชั้นนี้อาจจะ log หรือแปะบริบทเพิ่มเติมได้ก่อนส่งต่อ */
        fprintf(stderr, "[register_person] สร้าง Person ไม่สำเร็จ (code=%d)\n", err);
        return err;
    }
    printf("[register_person] ลงทะเบียนสำเร็จ: %s (อายุ %d)\n", out->name, out->age);
    return ERR_OK;
}

int main(void) {
    Person p;
    ErrorCode err;

    /* ปิด buffering ของ stdout เพื่อให้ลำดับข้อความ stdout/stderr ตรงกับลำดับการทำงานจริง */
    setvbuf(stdout, NULL, _IONBF, 0);

    /* main คือชั้นบนสุดที่ตัดสินใจว่าจะทำอย่างไรกับ error ที่ถูกส่งต่อขึ้นมาจากชั้นล่างสุด */
    err = register_person(&p, "Somchai", 30);
    if (err != ERR_OK) {
        printf("main: การลงทะเบียนล้มเหลว (code=%d)\n", err);
    }

    err = register_person(&p, "Somchai", 999);   /* อายุไม่ถูกต้อง เกิดจากชั้นล่างสุด */
    if (err != ERR_OK) {
        printf("main: การลงทะเบียนล้มเหลว (code=%d)\n", err);
    }

    return 0;
}
```

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 propagate_error.c -o propagate_error
./propagate_error
```

```
[register_person] ลงทะเบียนสำเร็จ: Somchai (อายุ 30)
[register_person] สร้าง Person ไม่สำเร็จ (code=1)
main: การลงทะเบียนล้มเหลว (code=1)
```

จะเห็นว่า error `ERR_INVALID_AGE` (code=1) เกิดขึ้นที่ **ชั้นล่างสุด** (`validate_age`)
แล้วถูกส่งต่อขึ้นมาทีละชั้น: `validate_age` → `create_person` → `register_person` → `main`
โดยแต่ละชั้น **ตรวจสอบ return value ทันทีหลังเรียกฟังก์ชันถัดไปเสมอ** (สังเกต pattern
`if (err != ERR_OK) { return err; }` ที่ซ้ำกันทุกชั้น) นี่คือรูปแบบที่พบมากที่สุดในโค้ด C
คุณภาพสูงระดับ Production ทั่วโลก — บางครั้งเรียก pattern นี้ว่า **"Check and Propagate"**

### หลักการออกแบบการ Propagate Error ที่ดี

1. **ตรวจสอบ return value ของทุกฟังก์ชันที่อาจล้มเหลว ทันทีหลังเรียก** อย่าปล่อยผ่านแม้แต่
   ครั้งเดียว เพราะไม่มีกลไกใดบังคับให้ C หยุดทำงานอัตโนมัติเหมือน exception
2. **ส่งต่อ error code เดิมขึ้นไป** เว้นแต่ชั้นกลางต้องการแปลความหมายใหม่ให้เหมาะกับบริบท
   ของตัวเอง (เช่นแปล error จาก "การเปิดไฟล์ config" ให้กลายเป็น "การตั้งค่าระบบล้มเหลว"
   ที่ชั้นบนเข้าใจง่ายกว่า)
3. **Log หรือแปะบริบทเพิ่มเติมได้ที่จุดที่ตรวจพบ** (เหมือน `fprintf(stderr, ...)` ใน
   `register_person`) เพราะยิ่งอยู่ใกล้ต้นเหตุ ยิ่งมีข้อมูลบริบทมากกว่าชั้นบนสุด
4. **ให้ชั้นบนสุด (เช่น `main` หรือ Handler ระดับบนของระบบ) เป็นผู้ตัดสินใจสุดท้าย** ว่าจะ
   ทำอย่างไรกับ error (แสดงข้อความแก่ผู้ใช้, retry, จบโปรแกรม) ส่วนชั้นกลางมีหน้าที่แค่
   ส่งต่อข้อมูลให้ครบถ้วนเท่านั้น ไม่ควรตัดสินใจแทนชั้นบนโดยพลการ

---

## 16.8 `goto` Cleanup Pattern: จัดการ Resource หลายตัวตอน Error (Step 128)

ปัญหาคลาสสิกของ C คือ: เมื่อฟังก์ชันหนึ่งต้อง **เปิด Resource หลายตัวติดกัน** (เปิดไฟล์,
`malloc` หลายก้อน) แล้วเกิด Error ขึ้นกลางทาง เราต้อง **ปิด/คืน Resource ทุกตัวที่เปิดสำเร็จ
ไปแล้วก่อนหน้านั้น** ก่อนจะ `return` ออกจากฟังก์ชัน ไม่เช่นนั้นจะเกิด **Resource Leak**
(หน่วยความจำรั่วหรือไฟล์ค้าง) — ถ้าเขียนด้วย `if` ซ้อนกันหลายชั้น โค้ดจะเยิ่นเย้อและอ่านยาก
มาก วิธีแก้ที่เป็นมาตรฐานในวงการ C (ใช้ในโค้ด Linux Kernel เองด้วย) คือการใช้ **`goto`
สำหรับ Cleanup** ซึ่งเป็นหนึ่งในการใช้งาน `goto` ไม่กี่แบบที่ถือว่าเป็น Best Practice
(ต่างจากการใช้ `goto` แบบอื่นที่มักถูกมองว่าทำให้โค้ดอ่านยาก)

### รูปแบบพื้นฐาน: หนึ่ง Resource หนึ่ง Label

```c
/* goto_cleanup_single.c
 * สาธิต goto cleanup pattern แบบพื้นฐาน: หนึ่งจุดออก หนึ่ง label cleanup
 */
#include <stdio.h>
#include <stdlib.h>

static int process_file(const char *path) {
    int status = 0;
    FILE *fp = fopen(path, "r");

    if (fp == NULL) {
        perror("fopen ล้มเหลว");
        return -1;
    }

    char *buffer = malloc(256);
    if (buffer == NULL) {
        fprintf(stderr, "malloc ล้มเหลว\n");
        status = -1;
        goto cleanup_file;   /* ต้องปิดไฟล์ก่อนออกจากฟังก์ชัน */
    }

    if (fgets(buffer, 256, fp) != NULL) {
        printf("อ่านได้: %s", buffer);
    }

    free(buffer);

cleanup_file:
    fclose(fp);
    return status;
}

int main(void) {
    /* ใช้ตัวมันเองเป็น input ที่มีอยู่จริง เพื่อให้ตัวอย่างรันได้จริงและ deterministic */
    int rc = process_file(__FILE__);
    printf("process_file คืนค่า %d\n", rc);
    return 0;
}
```

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 goto_cleanup_single.c -o goto_cleanup_single
./goto_cleanup_single
```

```
อ่านได้: /* goto_cleanup_single.c
process_file คืนค่า 0
```

### รูปแบบเต็ม: หลาย Resource พร้อมกัน (ไฟล์ 2 ไฟล์ + `malloc` 2 ก้อน)

เมื่อมี Resource มากกว่าหนึ่งตัว หลักการสำคัญคือ **เรียง label cleanup (หรือลำดับการ
free/close ภายใน label เดียว) จาก "resource ที่เปิดล่าสุด" กลับไปหา "resource ที่เปิดก่อน
สุด" เสมอ (Last-In-First-Out — เหมือนหลักการเดียวกับ Stack)** และต้องเช็คว่า resource
แต่ละตัว **เปิดสำเร็จแล้วจริงๆ** ก่อน cleanup มัน (เช่นเช็ค `!= NULL` ก่อน `fclose`/`free`)

```c
/* goto_cleanup_multi.c
 * สาธิต goto cleanup pattern เมื่อต้องจัดการหลาย resource พร้อมกัน
 * (เปิดไฟล์ 2 ไฟล์ + malloc 2 ก้อน) โดยต้อง cleanup เฉพาะ resource ที่เปิดสำเร็จแล้วเท่านั้น
 * ถ้า error เกิดกลางทาง — เรียงลำดับ label จาก "หลังสุด" ไป "แรกสุด" (reverse order)
 */
#include <stdio.h>
#include <stdlib.h>

#define BUFFER_SIZE 128

typedef enum {
    ERR_OK = 0,
    ERR_OPEN_INPUT = 1,
    ERR_OPEN_OUTPUT = 2,
    ERR_ALLOC_READ_BUF = 3,
    ERR_ALLOC_WRITE_BUF = 4
} ErrorCode;

static ErrorCode copy_file_buffered(const char *in_path, const char *out_path) {
    ErrorCode status = ERR_OK;

    FILE *in_fp = NULL;
    FILE *out_fp = NULL;
    char *read_buf = NULL;
    char *write_buf = NULL;
    size_t lines_copied = 0;

    in_fp = fopen(in_path, "r");
    if (in_fp == NULL) {
        perror("เปิดไฟล์ต้นทางล้มเหลว");
        status = ERR_OPEN_INPUT;
        goto cleanup;   /* ยังไม่มี resource ใดเปิดสำเร็จเลย ไป cleanup (ไม่ทำอะไร) ได้เลย */
    }

    out_fp = fopen(out_path, "w");
    if (out_fp == NULL) {
        perror("เปิดไฟล์ปลายทางล้มเหลว");
        status = ERR_OPEN_OUTPUT;
        goto cleanup;   /* ต้องปิด in_fp ที่เปิดไปแล้ว */
    }

    read_buf = malloc(BUFFER_SIZE);
    if (read_buf == NULL) {
        fprintf(stderr, "จัดสรร read_buf ล้มเหลว\n");
        status = ERR_ALLOC_READ_BUF;
        goto cleanup;   /* ต้องปิดทั้ง in_fp และ out_fp */
    }

    write_buf = malloc(BUFFER_SIZE);
    if (write_buf == NULL) {
        fprintf(stderr, "จัดสรร write_buf ล้มเหลว\n");
        status = ERR_ALLOC_WRITE_BUF;
        goto cleanup;   /* ต้อง free read_buf และปิดทั้ง in_fp, out_fp */
    }

    /* งานหลัก: คัดลอกข้อมูลทีละบรรทัดโดยใช้ buffer ทั้งสอง */
    while (fgets(read_buf, BUFFER_SIZE, in_fp) != NULL) {
        snprintf(write_buf, BUFFER_SIZE, "%s", read_buf);
        fputs(write_buf, out_fp);
        lines_copied++;
    }
    printf("คัดลอกสำเร็จ %zu บรรทัด\n", lines_copied);

cleanup:
    /* เรียง free/close ตามลำดับย้อนกลับจากที่ acquire มา (Last-In-First-Out)
     * และทุกตัวปลอดภัยกับค่า NULL: free(NULL) ไม่ทำอะไร, ส่วน fclose ต้องเช็ค NULL เอง */
    free(write_buf);
    free(read_buf);
    if (out_fp != NULL) {
        fclose(out_fp);
    }
    if (in_fp != NULL) {
        fclose(in_fp);
    }

    return status;
}

int main(void) {
    /* ปิด buffering ของ stdout เพื่อให้ลำดับข้อความ stdout/stderr ที่พิมพ์ออกมาตรงกับ
     * ลำดับการทำงานจริงของโปรแกรมเป๊ะๆ (ปกติแล้ว stderr จะไม่ buffer อยู่แล้ว) */
    setvbuf(stdout, NULL, _IONBF, 0);

    /* เตรียมไฟล์ต้นทางตัวอย่างที่มีเนื้อหาแน่นอน 3 บรรทัด เพื่อให้ผลลัพธ์ deterministic */
    const char *sample_path = "/tmp/goto_cleanup_multi_input.txt";
    FILE *sample = fopen(sample_path, "w");
    if (sample == NULL) {
        perror("สร้างไฟล์ตัวอย่างล้มเหลว");
        return 1;
    }
    fputs("บรรทัดที่ 1\n", sample);
    fputs("บรรทัดที่ 2\n", sample);
    fputs("บรรทัดที่ 3\n", sample);
    fclose(sample);

    ErrorCode err = copy_file_buffered(sample_path, "/tmp/goto_cleanup_multi_output.txt");
    if (err == ERR_OK) {
        printf("คัดลอกไฟล์สำเร็จ\n");
    } else {
        printf("คัดลอกไฟล์ล้มเหลว (code=%d)\n", err);
    }

    /* ทดสอบเคส error: เปิดไฟล์ต้นทางที่ไม่มีอยู่จริง */
    err = copy_file_buffered("/path/that/does/not/exist.txt", "/tmp/goto_cleanup_multi_output2.txt");
    printf("ทดสอบไฟล์ต้นทางไม่มีอยู่จริง: code=%d\n", err);

    return 0;
}
```

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 goto_cleanup_multi.c -o goto_cleanup_multi
./goto_cleanup_multi
```

```
คัดลอกสำเร็จ 3 บรรทัด
คัดลอกไฟล์สำเร็จ
เปิดไฟล์ต้นทางล้มเหลว: No such file or directory
ทดสอบไฟล์ต้นทางไม่มีอยู่จริง: code=1
```

โครงสร้างนี้มีจุดเด่นสำคัญคือ **มีจุดออกจากฟังก์ชันแค่จุดเดียว (single point of exit)**
ที่ท้ายฟังก์ชัน (`return status;`) ไม่ว่าจะสำเร็จหรือล้มเหลวที่จุดไหนก็ตาม ทุกเส้นทางจะไหล
ผ่าน label `cleanup:` เดียวกันเสมอ ทำให้มั่นใจได้ว่า **ไม่มีทางลืม cleanup resource ใดๆ**
(ต่างจากการเขียน `if` ซ้อนกันหลายชั้นที่ต้องคัดลอกโค้ด cleanup ซ้ำในทุก branch ซึ่งเสี่ยง
ลืมหรือเขียนไม่ครบมาก) และการเช็ค `!= NULL` ก่อน `fclose`/การใช้ `free(NULL)` (ซึ่ง
มาตรฐาน C รับประกันว่าปลอดภัย ไม่ทำอะไรเลย) ทำให้ label เดียวนี้ **ปลอดภัยสำหรับทุก
สถานการณ์ error ที่เป็นไปได้** ไม่ว่า error จะเกิดที่จุดไหนในฟังก์ชัน

### หลักการเขียน `goto` Cleanup Pattern ให้ถูกต้อง

1. **ประกาศตัวแปร resource ทุกตัวเป็น `NULL` ตั้งแต่ต้นฟังก์ชัน** เพื่อให้ label cleanup
   เช็ค `!= NULL` ได้อย่างปลอดภัยเสมอ ไม่ว่าโค้ดจะ `goto` มาจากจุดไหนก็ตาม
2. **`goto` ได้เฉพาะ "ไปข้างหน้า" สู่ label cleanup ที่ท้ายฟังก์ชันเท่านั้น** ห้ามใช้ `goto`
   กระโดดข้าม scope แบบซับซ้อนหรือย้อนกลับขึ้นไปข้างบน (จะทำให้อ่านโค้ดยากขึ้นทันที และ
   เสียจุดเด่นเรื่อง "single point of exit" ไป)
3. **เรียงลำดับ cleanup แบบย้อนกลับ (LIFO)** จาก resource ที่ได้มาล่าสุดไปหาแรกสุดเสมอ
   เพราะ resource ที่ได้มาทีหลังมักขึ้นอยู่กับ resource ก่อนหน้า (เช่น buffer ที่ใช้คู่กับ
   ไฟล์ที่เปิดไว้)
4. **ใช้ label เดียวพอสำหรับกรณีส่วนใหญ่** (อย่างตัวอย่างข้างบน) แต่ถ้า Resource มีความ
   ซับซ้อนมาก (เช่นต้องปลด mutex lock ก่อนปิดไฟล์เสมอ) อาจใช้หลาย label เรียงกัน เช่น
   `cleanup_buffer:`, `cleanup_output:`, `cleanup_input:` แล้วปล่อยให้โค้ด "ไหลผ่าน" ลงมา
   ทีละ label ตามลำดับปกติของภาษา C (ไม่มี `break` คั่นระหว่าง label)
5. **ตั้งชื่อ label ให้สื่อความหมาย** เช่น `cleanup`, `cleanup_file`, `error_out` แทนชื่อ
   กำกวมอย่าง `done` หรือ `end` เพื่อให้เจตนาชัดเจนตั้งแต่อ่านครั้งแรก

---

## ข้อผิดพลาดที่พบบ่อย (Common Pitfalls)

1. **ลืมเช็ค Return Value ของฟังก์ชันที่อาจล้มเหลว** — เพราะ C ไม่มี exception บังคับให้
   หยุดทำงาน การละเลยตรวจสอบค่าที่คืนกลับมาทำให้โปรแกรมทำงานต่อไปด้วยข้อมูลที่ผิดพลาด
   แบบเงียบๆ จนพังที่จุดอื่นซึ่งไกลจากต้นเหตุมาก (สืบสาเหตุยากมาก)
2. **อ่าน `errno` หลังเรียกฟังก์ชันอื่นคั่นกลาง แม้จะดูเหมือน "ไม่เกี่ยวกับ error" เลยก็ตาม**
   — ตามที่พิสูจน์ด้วยการรันจริงในหัวข้อ 16.4 แม้แต่ `perror()` เองก็ยังเปลี่ยนค่า `errno`
   เป็นผลข้างเคียงได้ ต้องเก็บค่า `errno` ใส่ตัวแปรทันทีเสมอ
3. **ใช้ `assert()` ตรวจสอบ Input จากผู้ใช้หรือข้อมูลจากภายนอก** — เมื่อ compile แบบ
   Release ด้วย `-DNDEBUG` เงื่อนไขเหล่านั้นจะถูกลบออกไปทั้งหมด ทำให้ข้อมูลที่ผิดพลาด
   หลุดรอดเข้าไปประมวลผลต่อโดยไม่มีการตรวจสอบใดๆ เลย อาจกลายเป็นช่องโหว่ความปลอดภัย
4. **เขียน Error Code ด้วยตัวเลขดิบๆ กระจายอยู่ทั่วโค้ดแทนที่จะใช้ `enum`** — ทำให้อ่านยาก
   และเสี่ยงพิมพ์ตัวเลขผิดหรือใช้ค่าซ้ำกันโดยไม่ตั้งใจ
5. **ลืม Propagate Error ขึ้นไปให้ครบทุกชั้น** — เช่นชั้นกลางเรียกฟังก์ชันที่อาจล้มเหลว แต่
   ไม่ตรวจสอบค่าที่คืนกลับมา หรือตรวจสอบแล้วแต่ไม่ `return` ค่า error ขึ้นไปต่อ ทำให้ชั้น
   บนสุดไม่รู้เลยว่ามีปัญหาเกิดขึ้นที่ชั้นล่าง
6. **เรียงลำดับ Label ใน `goto` Cleanup ผิด (ไม่เป็น LIFO)** — เช่นปิดไฟล์ต้นทางก่อนที่
   ยังต้องใช้อ่านข้อมูลต่อ หรือ `free()` buffer ที่ resource อื่นยังอ้างอิงอยู่ ทำให้เกิด
   Use-After-Free หรือปิด/คืน resource ผิดลำดับจนโปรแกรม crash
7. **ลืมกำหนดค่าเริ่มต้นของตัวแปร resource เป็น `NULL` ก่อนใช้ `goto` Cleanup** — ถ้า
   `goto` ไปที่ label cleanup ก่อนตัวแปรบางตัวถูกกำหนดค่า (เช่นประกาศไว้เฉยๆ ไม่ initialize)
   การเช็ค `!= NULL` ที่ label จะอ่านค่าขยะ (Garbage Value) แล้วอาจเรียก `fclose`/`free`
   กับ pointer ที่ไม่ถูกต้อง กลายเป็น Undefined Behavior ทันที

---

## แบบฝึกหัดท้ายบท

1. เขียนฟังก์ชัน `ErrorCode safe_sqrt(double x, double *result)` ที่คำนวณรากที่สองของ `x`
   โดยใช้ `sqrt()` จาก `<math.h>` แต่คืน enum error code `ERR_NEGATIVE_INPUT` ถ้า `x` ติดลบ
   แทนที่จะปล่อยให้ได้ค่า `NaN` แบบเงียบๆ
2. เขียนโปรแกรมที่พยายามเปิดไฟล์ที่ไม่มีสิทธิ์เขียน (ลอง `fopen("/root/test.txt", "w")`
   บนเครื่อง Linux ทั่วไปที่ไม่ได้รันด้วยสิทธิ์ root) แล้วใช้ `strerror(errno)` แสดงข้อความ
   สาเหตุ พร้อมเปรียบเทียบกับ `errno` ที่ได้จากตัวอย่างในบทนี้ว่าตรงกับ `EACCES` หรือไม่
3. เขียนฟังก์ชันที่มี precondition ชัดเจน (เช่น `array` ต้องมีอย่างน้อย 1 element) แล้วใช้
   `assert()` ตรวจสอบ precondition นั้น จากนั้นลอง compile ทั้งแบบมีและไม่มี `-DNDEBUG`
   เพื่อสังเกตความแตกต่างของพฤติกรรมด้วยตัวเอง
4. ออกแบบระบบ error code 3 ชั้นฟังก์ชัน (คล้ายตัวอย่าง `validate_age`/`create_person`/
   `register_person` ในหัวข้อ 16.7) สำหรับสถานการณ์ "การสมัครสมาชิกเว็บไซต์" ที่ต้อง
   ตรวจสอบทั้งความยาวรหัสผ่าน (`password`) และรูปแบบอีเมล (`email`) แบบง่ายๆ (มี `@`
   อย่างน้อยหนึ่งตัว)
5. เขียนฟังก์ชันที่จองหน่วยความจำด้วย `malloc` 3 ก้อนติดกัน (ขนาดต่างกันได้) โดยใช้ `goto`
   cleanup pattern จัดการกรณีที่ก้อนใดก้อนหนึ่งจองไม่สำเร็จกลางทาง ให้แน่ใจว่า `free()`
   เฉพาะก้อนที่จองสำเร็จไปแล้วเท่านั้น
6. รวมทุกแนวคิดในบทนี้: เขียนฟังก์ชัน `ErrorCode load_config(const char *path, Config *out)`
   ที่เปิดไฟล์ config, `malloc` buffer สำหรับอ่านไฟล์, ตรวจสอบ input ทุกจุดแบบ Defensive
   Programming, ใช้ enum error code, และใช้ `goto` cleanup pattern จัดการ resource ทั้งหมด

### แนวทางเฉลยข้อ 3

```c
/* assert_precondition.c */
#include <stdio.h>
#include <assert.h>

/* precondition: arr ต้องไม่เป็น NULL และ count ต้องมากกว่า 0 */
static int array_sum(const int *arr, size_t count) {
    assert(arr != NULL && "array_sum: arr ต้องไม่เป็น NULL");
    assert(count > 0 && "array_sum: count ต้องมากกว่า 0");

    int total = 0;
    for (size_t i = 0; i < count; i++) {
        total += arr[i];
    }
    return total;
}

int main(void) {
    int numbers[] = {10, 20, 30, 40, 50};
    size_t count = sizeof(numbers) / sizeof(numbers[0]);

    printf("ผลรวม = %d\n", array_sum(numbers, count));

    return 0;
}
```

```bash
# คอมไพล์แบบ Debug (assert ทำงานเต็มที่)
gcc -Wall -Wextra -Wpedantic -std=c17 assert_precondition.c -o assert_precondition_debug
./assert_precondition_debug

# คอมไพล์แบบ Release (assert ถูกลบออกทั้งหมด)
gcc -Wall -Wextra -Wpedantic -std=c17 -DNDEBUG assert_precondition.c -o assert_precondition_release
./assert_precondition_release
```

```
ผลรวม = 150
```

ทั้งสองแบบให้ผลลัพธ์เดียวกันเมื่อ input ถูกต้องตาม precondition (ซึ่งเป็นกรณีปกติ) ความ
แตกต่างจะเห็นชัดก็ต่อเมื่อลองเรียก `array_sum(NULL, 5)` หรือ `array_sum(numbers, 0)` —
Debug Build จะหยุดทันทีพร้อมข้อความชี้บั๊ก ในขณะที่ Release Build (มี `-DNDEBUG`) จะข้าม
การตรวจสอบไปเลย แล้วปล่อยให้ `arr[i]` dereference pointer ที่เป็น `NULL` ซึ่งเป็น
Undefined Behavior (มักจะได้ Segmentation Fault บนระบบส่วนใหญ่) — นี่คือเหตุผลที่ยืนยัน
อีกครั้งว่า `assert` เหมาะกับการจับบั๊กตอนพัฒนาเท่านั้น ไม่ใช่กลไกป้องกันข้อผิดพลาดสำหรับ
โปรแกรมที่ส่งถึงมือผู้ใช้จริง

### แนวทางเฉลยข้อ 5

```c
/* goto_cleanup_three_mallocs.c */
#include <stdio.h>
#include <stdlib.h>

typedef enum {
    ERR_OK = 0,
    ERR_ALLOC_A = 1,
    ERR_ALLOC_B = 2,
    ERR_ALLOC_C = 3
} ErrorCode;

static ErrorCode allocate_three_buffers(size_t size_a, size_t size_b, size_t size_c) {
    ErrorCode status = ERR_OK;

    int *buf_a = NULL;
    int *buf_b = NULL;
    int *buf_c = NULL;

    buf_a = malloc(size_a * sizeof(int));
    if (buf_a == NULL) {
        fprintf(stderr, "จองหน่วยความจำ buf_a ล้มเหลว\n");
        status = ERR_ALLOC_A;
        goto cleanup;
    }
    printf("จอง buf_a สำเร็จ (%zu ints)\n", size_a);

    buf_b = malloc(size_b * sizeof(int));
    if (buf_b == NULL) {
        fprintf(stderr, "จองหน่วยความจำ buf_b ล้มเหลว\n");
        status = ERR_ALLOC_B;
        goto cleanup;   /* ต้อง free buf_a ที่จองไปแล้ว */
    }
    printf("จอง buf_b สำเร็จ (%zu ints)\n", size_b);

    buf_c = malloc(size_c * sizeof(int));
    if (buf_c == NULL) {
        fprintf(stderr, "จองหน่วยความจำ buf_c ล้มเหลว\n");
        status = ERR_ALLOC_C;
        goto cleanup;   /* ต้อง free ทั้ง buf_a และ buf_b */
    }
    printf("จอง buf_c สำเร็จ (%zu ints)\n", size_c);

    printf("จองหน่วยความจำครบทั้ง 3 ก้อนสำเร็จ\n");

cleanup:
    /* เรียงย้อนกลับ (LIFO): free(NULL) ปลอดภัยเสมอตามมาตรฐาน C ไม่ต้องเช็ค NULL ก่อนก็ได้ */
    free(buf_c);
    free(buf_b);
    free(buf_a);

    return status;
}

int main(void) {
    ErrorCode err;

    /* ปิด buffering ของ stdout เพื่อให้ลำดับข้อความ stdout/stderr ตรงกับลำดับการทำงานจริง */
    setvbuf(stdout, NULL, _IONBF, 0);

    printf("--- ทดสอบกรณีสำเร็จทั้งหมด ---\n");
    err = allocate_three_buffers(10, 20, 30);
    printf("ผลลัพธ์: code=%d\n\n", err);

    printf("--- ทดสอบกรณีจองก้อนใหญ่เกินจริง (คาดว่าจะล้มเหลวที่ก้อนใดก้อนหนึ่ง) ---\n");
    /* ขนาดใหญ่ระดับ petabyte (2^48 ints = 2^50 bytes) เพื่อบังคับให้ malloc ล้มเหลวแน่นอน
     * บนเครื่องทั่วไป โดยยังไม่ใหญ่พอที่จะทำให้ size_b * sizeof(int) overflow กลับมาเป็น 0 */
    err = allocate_three_buffers(10, (size_t)1 << 48, 30);
    printf("ผลลัพธ์: code=%d\n", err);

    return 0;
}
```

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 goto_cleanup_three_mallocs.c -o goto_cleanup_three_mallocs
./goto_cleanup_three_mallocs
```

```
--- ทดสอบกรณีสำเร็จทั้งหมด ---
จอง buf_a สำเร็จ (10 ints)
จอง buf_b สำเร็จ (20 ints)
จอง buf_c สำเร็จ (30 ints)
จองหน่วยความจำครบทั้ง 3 ก้อนสำเร็จ
ผลลัพธ์: code=0

--- ทดสอบกรณีจองก้อนใหญ่เกินจริง (คาดว่าจะล้มเหลวที่ก้อนใดก้อนหนึ่ง) ---
จอง buf_a สำเร็จ (10 ints)
จองหน่วยความจำ buf_b ล้มเหลว
ผลลัพธ์: code=2
```

> **สังเกต**: ต้องเลือกขนาดที่ขอจองอย่างระมัดระวัง — ลองขอ `(size_t)1 << 62` ตัว `int`
> ดูก่อน จะพบว่า `malloc` กลับ "สำเร็จ" อย่างน่าประหลาดใจ เพราะ `size_b * sizeof(int)`
> เกิด **Integer Overflow** จน wrap กลับมาเป็นค่าเล็กๆ (หรือ `0`) โดยไม่มีการเตือนใดๆ นี่คือ
> อีกหนึ่งกับดักคลาสสิกของ C: **ต้องตรวจสอบ overflow ของการคำนวณขนาดก่อนส่งให้ `malloc`
> เสมอ** โดยเฉพาะเมื่อขนาดมาจากการคูณจำนวนสองค่าเข้าด้วยกัน (จะเรียนเทคนิคป้องกันเรื่องนี้
> อย่างละเอียดใน Part ที่เกี่ยวกับ Dynamic Memory และ Security ในโมดูลถัดๆ ไป)

ในกรณีที่สอง `buf_a` จองสำเร็จ แต่ `buf_b` (ขอขนาดใหญ่เกินหน่วยความจำที่มีจริงในระบบ)
ล้มเหลว โค้ดจึงกระโดดไปที่ `cleanup:` ทันที ซึ่งจะ `free(buf_c)` (เป็น `NULL` เพราะไม่เคย
ถูกจอง — ปลอดภัย ไม่ทำอะไร), `free(buf_b)` (เป็น `NULL` เช่นกัน), และ `free(buf_a)` (ปลด
หน่วยความจำที่จองสำเร็จไปจริงคืนให้ระบบ) ครบถ้วนทุกก้อนโดยไม่มีการรั่วไหลเลย แม้จะจอง
สำเร็จแค่ 1 ใน 3 ก้อนก็ตาม

---

## สรุปท้ายบท

ใน Part นี้เราได้:

- เข้าใจธรรมเนียมการออกแบบ Return Code แบบ `0 = success` และการใช้ Negative Code หรือ
  `enum` Error Code แทนตัวเลขดิบๆ เพื่อให้โค้ดอ่านง่ายและขยายได้
- ใช้ `errno`, `perror()`, และ `strerror()` เพื่อรู้สาเหตุที่แท้จริงของความล้มเหลวจากฟังก์ชัน
  ไลบรารีมาตรฐาน พร้อมพิสูจน์ด้วยการรันจริงว่าทำไม `errno` ต้องอ่านทันทีหลังฟังก์ชันล้มเหลว
  เท่านั้น
- เข้าใจ `assert()` อย่างลึกซึ้ง ทั้งการใช้งานที่ถูกต้อง (ดักบั๊กของโปรแกรมเมอร์เอง) และ
  ผลกระทบของ `NDEBUG` ที่ทำให้ assert หายไปทั้งหมดใน Release Build
- เขียนโค้ดแบบ Defensive Programming ที่ตรวจสอบ Input, Precondition, และ Postcondition
  อย่างเป็นระบบ และรู้ความแตกต่างสำคัญระหว่างสิ่งที่ควรใช้ `assert` กับสิ่งที่ควรใช้ `if`
- ออกแบบ Pattern การ Propagate Error ขึ้นไปหลายชั้นฟังก์ชันได้ด้วยมือ เพราะ C ไม่มีกลไก
  Exception เหมือนภาษาสมัยใหม่อื่นๆ
- ใช้ `goto` Cleanup Pattern จัดการ Resource หลายตัว (ไฟล์, หน่วยความจำ) ได้อย่างปลอดภัย
  แม้เกิด Error กลางทางในสถานการณ์ที่ซับซ้อน

ทักษะการจัดการ Error อย่างเป็นระบบที่เรียนไปใน Part นี้เป็นรากฐานสำคัญที่จะกลับมาใช้ซ้ำ
ตลอดหลักสูตร โดยเฉพาะใน Module C (Systems Programming) ที่ต้องจัดการ Error จาก System
Call จำนวนมาก และแม้แต่ใน C++ (Module D เป็นต้นไป) ที่แม้จะมี Exception ให้ใช้ แต่แนวคิด
เรื่อง Defensive Programming และการออกแบบ Error Code ที่ดียังคงสำคัญเสมอ ใน **Part 17**
เราจะเรียนรู้ **Modular Programming** การจัดโครงสร้างโปรแกรมขนาดใหญ่ให้เป็นระเบียบด้วย
Header File, Compilation Unit, และ Linkage ซึ่งจะทำให้เราเขียนโปรแกรมหลายไฟล์ที่ทำงาน
ร่วมกันได้อย่างเป็นระบบ

**ต่อไป:** [Part 17 — Modular Programming](./part-017-modular-programming.md)
