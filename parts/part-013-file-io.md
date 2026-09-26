# Part 13: File I/O ใน C (Step 97–104)

> Module B — C ระดับกลาง & โครงสร้างข้อมูล/อัลกอริทึม | Part 13 จาก 125
> Step ที่ครอบคลุมใน Part นี้: Step 97–104
> Part ก่อนหน้า: [Part 12 — Array หลายมิติและ Pointer-to-Pointer](./part-012-multidim-arrays-pointers.md) | Part ถัดไป: [Part 14 — Preprocessor และ Macro](./part-014-preprocessor-macros.md)

## เป้าหมายการเรียนรู้ (Learning Objectives)

หลังจบ Part นี้ ผู้เรียนจะสามารถ:

1. เปิดและปิดไฟล์ด้วย `fopen`/`fclose` ได้อย่างถูกต้อง พร้อมเข้าใจความหมายของ mode string
   ทุกแบบ (`"r"`, `"w"`, `"a"`, `"r+"`, `"rb"` ฯลฯ) และเลือกใช้ mode ที่เหมาะกับสถานการณ์
2. เขียนและอ่านไฟล์ข้อความ (text file) ด้วย `fprintf`/`fputs` และ `fgets` ได้อย่างปลอดภัย
   โดยไม่เกิด buffer overflow
3. อธิบายความแตกต่างระหว่าง text mode กับ binary mode และรู้ว่าทำไมความแตกต่างนี้
   "แทบไม่มีผล" บน Linux แต่สำคัญมากบน Windows
4. อ่านและเขียนข้อมูล struct ทั้งก้อนลงไฟล์ binary ด้วย `fread`/`fwrite` ได้อย่างถูกต้อง
5. ตรวจสอบข้อผิดพลาดระหว่างอ่าน/เขียนไฟล์ด้วย `ferror` และแยกแยะจากการจบไฟล์ปกติด้วย `feof`
6. ใช้ `fseek`/`ftell`/`rewind` เพื่อเข้าถึงข้อมูลในไฟล์แบบสุ่ม (Random Access) โดยไม่ต้อง
   อ่านไฟล์ทั้งหมดตั้งแต่ต้น
7. เขียนโปรแกรมบันทึกและโหลดข้อมูล struct array (รายชื่อนักเรียน) ลงไฟล์ได้จริง ทั้งแบบ
   text และแบบ binary พร้อมออกแบบ interface ของโปรแกรมให้ปลอดภัยและตรวจสอบ error ครบถ้วน

---

## 13.1 fopen/fclose และความหมายของ Mode String (Step 97)

ทุกครั้งที่โปรแกรม C ต้องการอ่านหรือเขียนไฟล์บนดิสก์ จะต้องผ่าน 3 ขั้นตอนหลักเสมอ:

1. **เปิดไฟล์** ด้วย `fopen()` เพื่อขอ "หมายเลขอ้างอิง" มาจากระบบปฏิบัติการ (เก็บอยู่ในรูปแบบ
   `FILE *`)
2. **อ่าน/เขียนข้อมูล** ผ่านหมายเลขอ้างอิงนั้น ด้วยฟังก์ชันตระกูล `fprintf`, `fscanf`, `fread`,
   `fwrite`, `fgets`, `fputs` ฯลฯ
3. **ปิดไฟล์** ด้วย `fclose()` เพื่อคืนทรัพยากรกลับให้ระบบปฏิบัติการ และบังคับให้ข้อมูลที่ค้าง
   อยู่ใน buffer ถูกเขียนลงดิสก์จริง (เรียกว่า **flush**)

`FILE` เป็น struct ที่ประกาศไว้ใน `<stdio.h>` ซึ่งเก็บสถานะภายในของไฟล์ที่เปิดอยู่ (ตำแหน่ง
ปัจจุบัน, buffer, error flag ฯลฯ) โปรแกรมของเราไม่จำเป็นต้องรู้รายละเอียดข้างในของมันเลย
เพียงแค่เก็บ pointer ไปยังมัน (`FILE *`) แล้วส่งต่อให้ฟังก์ชันตระกูล `f...` ทุกตัวเท่านั้น

```c
/* fopen_basic.c
 * ตัวอย่างแรก: เปิดไฟล์เพื่อเขียน ตรวจสอบ NULL แล้วปิดไฟล์ */
#include <stdio.h>
#include <stdlib.h>

int main(void) {
    FILE *fp = fopen("greeting.txt", "w");
    if (fp == NULL) {
        perror("fopen ล้มเหลว");
        return EXIT_FAILURE;
    }

    fprintf(fp, "สวัสดี ไฟล์นี้ถูกสร้างโดยโปรแกรม C\n");

    if (fclose(fp) != 0) {
        perror("fclose ล้มเหลว");
        return EXIT_FAILURE;
    }

    printf("เขียนไฟล์ greeting.txt สำเร็จ\n");
    return EXIT_SUCCESS;
}
```

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 fopen_basic.c -o fopen_basic
./fopen_basic
cat greeting.txt
```

ผลลัพธ์:

```
เขียนไฟล์ greeting.txt สำเร็จ
สวัสดี ไฟล์นี้ถูกสร้างโดยโปรแกรม C
```

### `fopen` ต้องตรวจสอบ NULL เสมอ

`fopen()` คืนค่า `NULL` เมื่อเปิดไฟล์ไม่สำเร็จ (เช่น ไฟล์ไม่มีอยู่จริงในโหมดอ่าน, ไม่มีสิทธิ์
เข้าถึงไฟล์, ดิสก์เต็ม) การเรียกใช้ `FILE *` ที่เป็น `NULL` ต่อ (เช่นส่งให้ `fprintf`) เป็น
**Undefined Behavior** ทันที จึงต้องตรวจสอบทุกครั้งโดยไม่มีข้อยกเว้น

```c
/* fopen_fail.c
 * สาธิตกรณี fopen คืนค่า NULL เมื่อเปิดไฟล์ที่ไม่มีอยู่จริงในโหมดอ่าน */
#include <stdio.h>
#include <stdlib.h>

int main(void) {
    FILE *fp = fopen("no_such_file.txt", "r");
    if (fp == NULL) {
        perror("fopen ล้มเหลว");
        return EXIT_FAILURE;
    }

    fclose(fp);
    return EXIT_SUCCESS;
}
```

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 fopen_fail.c -o fopen_fail
./fopen_fail
echo "exit=$?"
```

ผลลัพธ์:

```
fopen ล้มเหลว: No such file or directory
exit=1
```

`perror()` เป็นฟังก์ชันที่สะดวกมากสำหรับ debug: มันจะพิมพ์ข้อความที่เราใส่ให้ ตามด้วย `: `
และคำอธิบายข้อผิดพลาดล่าสุดของระบบปฏิบัติการ (อ่านค่ามาจากตัวแปร global `errno` ซึ่งจะเรียน
อย่างละเอียดใน **Part 16 — Error Handling**)

### ตารางสรุป Mode String ทั้งหมด

Mode string เป็นพารามิเตอร์ตัวที่สองของ `fopen()` กำหนดว่าจะเปิดไฟล์เพื่อทำอะไร และไฟล์
ต้องมีอยู่แล้วหรือไม่

| Mode | ความหมาย | ไฟล์ต้องมีอยู่ก่อนไหม | ถ้าไฟล์มีอยู่แล้ว | ตำแหน่งเริ่มต้น |
|---|---|---|---|---|
| `"r"` | อ่านอย่างเดียว (text) | ต้องมี ไม่งั้น `fopen` คืน `NULL` | เปิดอ่านตามเดิม | ต้นไฟล์ |
| `"w"` | เขียนอย่างเดียว (text) | ไม่ต้องมี (สร้างให้อัตโนมัติ) | **ล้างข้อมูลเดิมทั้งหมด** | ต้นไฟล์ |
| `"a"` | เขียนต่อท้าย (text) | ไม่ต้องมี (สร้างให้อัตโนมัติ) | ข้อมูลเดิมยังอยู่ครบ | ท้ายไฟล์เสมอ |
| `"r+"` | อ่าน**และ**เขียน (text) | ต้องมี | เปิดแก้ไขได้ทั้งไฟล์ | ต้นไฟล์ |
| `"w+"` | อ่านและเขียน (text) | ไม่ต้องมี | **ล้างข้อมูลเดิมทั้งหมด** | ต้นไฟล์ |
| `"a+"` | อ่านและเขียนต่อท้าย (text) | ไม่ต้องมี | ข้อมูลเดิมยังอยู่ครบ, อ่านได้ทั้งไฟล์ | เขียนที่ท้ายไฟล์เสมอ |
| `"rb"` | เหมือน `"r"` แต่เป็น binary mode | ต้องมี | เปิดอ่านตามเดิม | ต้นไฟล์ |
| `"wb"` | เหมือน `"w"` แต่เป็น binary mode | ไม่ต้องมี | ล้างข้อมูลเดิมทั้งหมด | ต้นไฟล์ |
| `"ab"` | เหมือน `"a"` แต่เป็น binary mode | ไม่ต้องมี | ข้อมูลเดิมยังอยู่ครบ | ท้ายไฟล์เสมอ |
| `"r+b"` / `"rb+"` | อ่านและเขียน binary | ต้องมี | เปิดแก้ไขได้ทั้งไฟล์ | ต้นไฟล์ |
| `"w+b"` / `"wb+"` | อ่านและเขียน binary | ไม่ต้องมี | ล้างข้อมูลเดิมทั้งหมด | ต้นไฟล์ |
| `"a+b"` / `"ab+"` | อ่านและเขียนต่อท้าย binary | ไม่ต้องมี | ข้อมูลเดิมยังอยู่ครบ | เขียนที่ท้ายไฟล์เสมอ |

จุดที่มือใหม่สับสนบ่อยที่สุดคือ **`"w"` จะล้างไฟล์เดิมทันทีที่ `fopen` สำเร็จ** แม้จะยังไม่ได้
เขียนอะไรเลยก็ตาม ถ้าต้องการเก็บข้อมูลเดิมไว้แล้วเพิ่มต่อท้าย ต้องใช้ `"a"` เท่านั้น

การเพิ่มตัวอักษร `b` เข้าไปในตอนท้าย (เช่น `"rb"`, `"wb"`) หมายถึง **binary mode** ซึ่งจะ
อธิบายรายละเอียดในหัวข้อ 13.3

---

## 13.2 เขียน/อ่านไฟล์ข้อความด้วย fprintf, fputs และ fgets (Step 98)

### เขียนไฟล์ข้อความ

`fprintf` ทำงานเหมือน `printf` ทุกประการ เพียงแต่มีพารามิเตอร์ตัวแรกเป็น `FILE *` เพื่อระบุ
ว่าจะพิมพ์ไปที่ไฟล์ไหน (แทนที่จะพิมพ์ไปที่จอเสมอเหมือน `printf`) ส่วน `fputs` ใช้สำหรับเขียน
string ดิบๆ ลงไฟล์โดยตรงโดยไม่มีการแปลง format ใดๆ (เร็วกว่า `fprintf` เมื่อไม่ต้องการ
format specifier)

```c
/* write_lines.c
 * สร้างไฟล์ข้อความหลายบรรทัดไว้ใช้ทดสอบ fgets ในตัวอย่างถัดไป */
#include <stdio.h>
#include <stdlib.h>

int main(void) {
    FILE *fp = fopen("poem.txt", "w");
    if (fp == NULL) {
        perror("fopen ล้มเหลว");
        return EXIT_FAILURE;
    }

    fputs("บรรทัดที่หนึ่ง: ภาษา C เรียบง่ายแต่ทรงพลัง\n", fp);
    fputs("บรรทัดที่สอง: Pointer คือหัวใจของภาษานี้\n", fp);
    fputs("บรรทัดที่สาม: File I/O ทำให้ข้อมูลอยู่ได้ตลอดไป\n", fp);

    fclose(fp);
    return EXIT_SUCCESS;
}
```

สังเกตว่า `fputs` **ไม่เติม** `\n` ให้อัตโนมัติเหมือน `puts` (ฟังก์ชันคู่กันสำหรับพิมพ์ขึ้นจอ)
ต้องใส่ `\n` เองเสมอถ้าต้องการขึ้นบรรทัดใหม่

### อ่านไฟล์ทีละบรรทัดอย่างปลอดภัยด้วย fgets

`fgets(buf, size, fp)` คือวิธีที่ปลอดภัยที่สุดในการอ่านไฟล์ข้อความทีละบรรทัดในภาษา C เพราะ
มันรับพารามิเตอร์ `size` (ขนาดบัฟเฟอร์) เข้าไปด้วย ทำให้ **ไม่มีทางเขียนเกินขอบเขตบัฟเฟอร์
ได้เลย** (ต่างจาก `gets()` ที่ถูกถอดออกจากมาตรฐาน C11 ไปแล้วเพราะไม่ปลอดภัย)

พฤติกรรมสำคัญของ `fgets` ที่ต้องจำ:

- อ่านไปเรื่อยๆ จนกว่าจะเจอ `\n`, จนครบ `size - 1` ตัวอักษร, หรือจนจบไฟล์ — แล้วแต่ว่า
  อะไรมาถึงก่อน
- ถ้าอ่านเจอ `\n` จะ **เก็บ** `\n` นั้นไว้ในบัฟเฟอร์ด้วย (ต่างจาก `scanf("%s", ...)` ที่ตัด
  whitespace ทิ้งเสมอ)
- เติม `'\0'` ปิดท้าย string ให้อัตโนมัติเสมอ
- คืนค่า `NULL` เมื่อจบไฟล์ (หรือเกิด error) โดยไม่ได้อ่านตัวอักษรใดเลย — ใช้เป็นเงื่อนไข
  จบ loop ได้พอดี

```c
/* read_lines_fgets.c
 * อ่านไฟล์ข้อความทีละบรรทัดอย่างปลอดภัยด้วย fgets
 * และตัดอักขระ '\n' ท้ายบรรทัดออกเอง */
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#define LINE_BUF_SIZE 256

int main(void) {
    FILE *fp = fopen("poem.txt", "r");
    if (fp == NULL) {
        perror("fopen ล้มเหลว");
        return EXIT_FAILURE;
    }

    char line[LINE_BUF_SIZE];
    int line_no = 1;

    while (fgets(line, sizeof line, fp) != NULL) {
        /* fgets เก็บ '\n' ไว้ในบัฟเฟอร์ด้วย (ถ้าอ่านทันบรรทัดพอดี)
           เราจึงต้องตัดออกเองถ้าต้องการข้อความที่ไม่มี newline ต่อท้าย */
        size_t len = strlen(line);
        if (len > 0 && line[len - 1] == '\n') {
            line[len - 1] = '\0';
        }
        printf("[บรรทัด %d] %s\n", line_no, line);
        line_no++;
    }

    fclose(fp);
    return EXIT_SUCCESS;
}
```

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 write_lines.c -o write_lines
./write_lines
gcc -Wall -Wextra -Wpedantic -std=c17 read_lines_fgets.c -o read_lines_fgets
./read_lines_fgets
```

ผลลัพธ์:

```
[บรรทัด 1] บรรทัดที่หนึ่ง: ภาษา C เรียบง่ายแต่ทรงพลัง
[บรรทัด 2] บรรทัดที่สอง: Pointer คือหัวใจของภาษานี้
[บรรทัด 3] บรรทัดที่สาม: File I/O ทำให้ข้อมูลอยู่ได้ตลอดไป
```

> **ทำไมต้องใช้ `sizeof line` แทนตัวเลขคงที่?** เพราะถ้าวันหนึ่งเปลี่ยนขนาด `LINE_BUF_SIZE`
> โค้ดจะยังถูกต้องเสมอโดยไม่ต้องแก้ตัวเลขในหลายจุด นี่คือหลักการเดียวกับที่เคยเรียนเรื่อง
> `sizeof` กับ array ใน Part 7

### เขียนต่อท้ายไฟล์ด้วยโหมด `"a"`

```c
/* append_mode.c
 * สาธิตโหมด "a" (append): เขียนต่อท้ายไฟล์เดิมโดยไม่ลบข้อมูลเก่า
 * ต่างจากโหมด "w" ที่จะล้างไฟล์เดิมทิ้งทั้งหมดก่อนเขียนใหม่เสมอ */
#include <stdio.h>
#include <stdlib.h>

int main(void) {
    /* รอบแรก: "w" สร้างไฟล์ใหม่ (หรือล้างของเดิมถ้ามีอยู่แล้ว) */
    FILE *fp = fopen("log.txt", "w");
    if (fp == NULL) {
        perror("fopen ล้มเหลว");
        return EXIT_FAILURE;
    }
    fprintf(fp, "[LOG] เริ่มโปรแกรม\n");
    fclose(fp);

    /* รอบที่สอง: "a" เปิดไฟล์เดิม แล้วเขียนต่อท้าย ไม่ลบของเดิม */
    fp = fopen("log.txt", "a");
    if (fp == NULL) {
        perror("fopen ล้มเหลว");
        return EXIT_FAILURE;
    }
    fprintf(fp, "[LOG] เหตุการณ์ที่ 1\n");
    fclose(fp);

    fp = fopen("log.txt", "a");
    if (fp == NULL) {
        perror("fopen ล้มเหลว");
        return EXIT_FAILURE;
    }
    fprintf(fp, "[LOG] เหตุการณ์ที่ 2\n");
    fclose(fp);

    /* อ่านกลับมาแสดงผลทั้งหมด เพื่อพิสูจน์ว่าไม่มีอะไรหายไป */
    fp = fopen("log.txt", "r");
    if (fp == NULL) {
        perror("fopen ล้มเหลว");
        return EXIT_FAILURE;
    }
    char line[128];
    while (fgets(line, sizeof line, fp) != NULL) {
        fputs(line, stdout);
    }
    fclose(fp);

    return EXIT_SUCCESS;
}
```

ผลลัพธ์:

```
[LOG] เริ่มโปรแกรม
[LOG] เหตุการณ์ที่ 1
[LOG] เหตุการณ์ที่ 2
```

จะเห็นว่าแม้เปิด-ปิดไฟล์ด้วย `"a"` ถึง 3 รอบแยกกัน ข้อมูลของทุกรอบก็ยังอยู่ครบ เพราะโหมด
`"a"` รับประกันว่าตำแหน่งเขียนจะอยู่ที่ท้ายไฟล์เสมอ ไม่ว่าจะเรียก `fseek` ไปที่ไหนก่อนหน้าก็ตาม
(พฤติกรรมนี้การันตีโดยมาตรฐาน C)

---

## 13.3 Text Mode กับ Binary Mode: ความแตกต่างและผลกระทบบน Linux (Step 99)

ภาษา C แบ่งการเปิดไฟล์ออกเป็น 2 โหมดใหญ่: **Text Mode** (ค่าเริ่มต้น เช่น `"r"`, `"w"`) และ
**Binary Mode** (เติม `b` เช่น `"rb"`, `"wb"`) ความแตกต่างในทางทฤษฎีของมาตรฐาน C มีอยู่
2 เรื่องหลัก:

1. **การแปลง newline**: ใน Text Mode ระบบปฏิบัติการบางระบบ (โดยเฉพาะ **Windows**) จะแปลง
   `\n` ที่โปรแกรมเขียน ให้กลายเป็น `\r\n` (Carriage Return + Line Feed) จริงบนดิสก์ และ
   แปลงกลับจาก `\r\n` เป็น `\n` ตอนอ่าน โดยอัตโนมัติแบบโปรแกรมเมอร์ไม่รู้ตัว ส่วน Binary
   Mode จะเขียน/อ่านไบต์ตรงๆ โดยไม่แปลงอะไรเลย
2. **จุดจบไฟล์ (EOF marker)**: ในระบบเก่าบางระบบ Text Mode อาจตีความไบต์บางค่าเป็นสัญญาณ
   จบไฟล์ก่อนถึงจุดจบจริง

**บน Linux (และ Unix อื่นๆ ทั้งหมด) ไม่มีความแตกต่างระหว่าง Text Mode กับ Binary Mode เลย**
เพราะ Linux ใช้ `\n` เป็นตัวแบ่งบรรทัดอยู่แล้วโดยไม่มีการแปลงใดๆ ทั้งสิ้น ฟังก์ชัน `fopen`
บน glibc (Linux) จะเพิกเฉยต่อตัวอักษร `b` ใน mode string โดยสิ้นเชิง — ทั้งสองแบบทำงาน
เหมือนกันทุกประการ

> **แล้วทำไมยังต้องใส่ `b` ทุกครั้งที่เปิดไฟล์ binary?**
> เพราะโค้ดของเราต้องรันได้บนทุกแพลตฟอร์ม (Portability) ถ้าเขียนโปรแกรมนี้แล้วมีคนเอาไป
> คอมไพล์บน Windows โดยลืมใส่ `b` ตอนเปิดไฟล์ binary จะเกิดบั๊กที่ตรวจจับยากมาก: ไบต์ที่มี
> ค่าเท่ากับ `0x0A` ในข้อมูล binary (ซึ่งอาจเป็นส่วนหนึ่งของตัวเลข หรือค่า struct ใดๆ ก็ได้
> ไม่เกี่ยวกับ newline เลย) จะถูกแปลงเป็น `0x0D 0x0A` ทำให้ไฟล์มีขนาดใหญ่ขึ้นและข้อมูล
> เพี้ยนทันที ดังนั้น **กฎทองคือ: เปิดไฟล์ binary ด้วย `"b"` เสมอ ไม่ว่าจะรันบนแพลตฟอร์มไหน**

### ตารางสรุปพฤติกรรมของ Text Mode vs Binary Mode

| ประเด็น | Text Mode (`"r"`, `"w"`) | Binary Mode (`"rb"`, `"wb"`) |
|---|---|---|
| การแปลง `\n` ↔ `\r\n` บน Windows | มี (อัตโนมัติ) | ไม่มี (เขียน/อ่านไบต์ดิบตรงๆ) |
| การแปลง `\n` ↔ `\r\n` บน Linux | ไม่มี (`\n` คือ `\n` เสมอ) | ไม่มี (เหมือนกับ Text Mode ทุกประการ) |
| เหมาะสำหรับข้อมูลประเภทใด | ข้อความที่มนุษย์อ่านได้ (log, CSV, config) | ข้อมูลดิบ เช่น struct, รูปภาพ, ไฟล์บีบอัด |
| ใช้ `fprintf`/`fgets`/`fputs` | ใช่ (เหมาะสมที่สุด) | ทำได้แต่ไม่นิยม (ไม่ควร parse binary ด้วย text function) |
| ใช้ `fread`/`fwrite` | ทำได้แต่ไม่นิยม (จะได้ text ดิบๆ ไม่ตรงชนิดข้อมูล) | ใช่ (เหมาะสมที่สุด) |

---

## 13.4 อ่าน/เขียนไฟล์ Binary ของ struct ด้วย fread/fwrite (Step 100)

เมื่อข้อมูลที่ต้องการเก็บเป็น struct (เช่นข้อมูลนักเรียน) การเขียนเป็น binary จะเร็วกว่าและ
กินพื้นที่น้อยกว่าการแปลงเป็นข้อความเสมอ เพราะเราคัดลอกไบต์ดิบของ struct ทั้งก้อนลงไฟล์
โดยตรง ไม่ต้องแปลงตัวเลขเป็น string ไปมา

```c
size_t fwrite(const void *ptr, size_t size, size_t nmemb, FILE *stream);
size_t fread(void *ptr, size_t size, size_t nmemb, FILE *stream);
```

- `ptr`: ที่อยู่ของข้อมูลต้นทาง (เขียน) หรือปลายทาง (อ่าน)
- `size`: ขนาดของสมาชิก 1 ตัว (byte) — มักใช้ `sizeof(TypeName)`
- `nmemb`: จำนวนสมาชิกที่ต้องการอ่าน/เขียน
- คืนค่า: **จำนวนสมาชิกที่อ่าน/เขียนสำเร็จจริง** (ไม่ใช่จำนวน byte!) ถ้าค่านี้น้อยกว่า `nmemb`
  ที่ขอไป แปลว่าเกิดปัญหา (ไฟล์เขียนไม่ครบ หรืออ่านเจอจุดจบไฟล์ก่อนกำหนด)

```c
/* binary_struct.c
 * เขียนและอ่านโครงสร้าง struct ลงไฟล์ binary ด้วย fwrite/fread */
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

typedef struct {
    int id;
    char name[32];
    double gpa;
} Student;

int main(void) {
    Student roster[3] = {
        {1, "Somchai", 3.45},
        {2, "Suda", 3.90},
        {3, "Anan", 2.75}
    };

    FILE *fp = fopen("students.bin", "wb");
    if (fp == NULL) {
        perror("fopen (เขียน) ล้มเหลว");
        return EXIT_FAILURE;
    }

    size_t written = fwrite(roster, sizeof(Student), 3, fp);
    if (written != 3) {
        fprintf(stderr, "เขียนได้ไม่ครบ: เขียนได้ %zu จาก 3 รายการ\n", written);
        fclose(fp);
        return EXIT_FAILURE;
    }
    fclose(fp);
    printf("เขียนไฟล์ binary สำเร็จ (%zu bytes)\n", 3 * sizeof(Student));

    /* อ่านกลับมาตรวจสอบ */
    Student loaded[3];
    fp = fopen("students.bin", "rb");
    if (fp == NULL) {
        perror("fopen (อ่าน) ล้มเหลว");
        return EXIT_FAILURE;
    }

    size_t read_count = fread(loaded, sizeof(Student), 3, fp);
    fclose(fp);

    printf("อ่านกลับมาได้ %zu รายการ:\n", read_count);
    for (size_t i = 0; i < read_count; i++) {
        printf("  id=%d name=%-10s gpa=%.2f\n",
               loaded[i].id, loaded[i].name, loaded[i].gpa);
    }

    return EXIT_SUCCESS;
}
```

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 binary_struct.c -o binary_struct
./binary_struct
```

ผลลัพธ์:

```
เขียนไฟล์ binary สำเร็จ (144 bytes)
อ่านกลับมาได้ 3 รายการ:
  id=1 name=Somchai    gpa=3.45
  id=2 name=Suda       gpa=3.90
  id=3 name=Anan       gpa=2.75
```

144 byte มาจากไหน? `sizeof(Student)` บนเครื่องทดสอบ (x86-64, gcc) เท่ากับ **48 byte** ไม่ใช่
`4 + 32 + 8 = 44` byte ตามที่คำนวณตรงๆ เพราะคอมไพเลอร์แทรก **padding** 4 byte ระหว่าง
`int id` กับ `char name[32]` เพื่อให้ `double gpa` เริ่มต้นที่ตำแหน่งหน่วยความจำที่หารด้วย 8
ลงตัว (ข้อกำหนดเรื่อง memory alignment ของ `double`) ดังนั้น `48 * 3 = 144` byte พอดี

> **ข้อควรระวังเรื่อง Portability ของไฟล์ binary struct**: ขนาด padding, การจัดเรียง byte
> ของตัวเลข (endianness), และแม้แต่ขนาดของชนิดข้อมูลพื้นฐาน (เช่น `int` อาจเป็น 2 หรือ 4
> byte ในบางระบบฝังตัว) **ไม่รับประกันว่าจะเหมือนกันทุกแพลตฟอร์ม/ทุกคอมไพเลอร์** ไฟล์
> binary ของ struct ที่เขียนจากเครื่องหนึ่ง อาจอ่านผิดเพี้ยนถ้าย้ายไปเปิดบนสถาปัตยกรรม
> ที่ต่างกัน (เช่น จาก x86 ไป ARM ที่มี endianness ต่างกัน) นี่คือเหตุผลที่ระบบระดับ
> Production จริง (เช่น protocol เครือข่าย หรือไฟล์ที่ต้องแชร์ข้ามแพลตฟอร์ม) มักไม่เขียน
> struct ดิบๆ แบบนี้ แต่จะ **serialize** ข้อมูลด้วยรูปแบบที่กำหนด byte order ชัดเจน (เช่น
> JSON, Protocol Buffers) ซึ่งจะเรียนในภายหลังของหลักสูตร (Module I) สำหรับตอนนี้ การเขียน
> struct ดิบเหมาะกับกรณีที่ไฟล์ถูกอ่าน-เขียนโดยโปรแกรมเดียวกันบนเครื่องเดียวกันเท่านั้น

---

## 13.5 ตรวจสอบข้อผิดพลาดด้วย ferror และ feof (Step 101)

เมื่อฟังก์ชันอ่านไฟล์ (เช่น `fread`, `fscanf`, `fgetc`) คืนค่าที่บ่งบอกว่า "อ่านไม่สำเร็จ" มี
สาเหตุที่เป็นไปได้ 2 แบบ ซึ่งต้องแยกออกจากกันให้ชัดเจน:

- **จบไฟล์ตามปกติ (End-Of-File)**: ไม่ใช่ข้อผิดพลาด เป็นสถานะปกติที่บอกว่าอ่านมาถึงจุดจบ
  ของไฟล์แล้ว ตรวจสอบด้วย `feof(fp)`
- **เกิดข้อผิดพลาดจริง (I/O Error)**: เช่น ดิสก์เสีย, ถอด USB กะทันหันระหว่างอ่าน ตรวจสอบ
  ด้วย `ferror(fp)`

```c
/* error_check.c
 * สาธิตการแยกแยะ "จบไฟล์ปกติ" (feof) กับ "เกิดข้อผิดพลาดระหว่างอ่าน" (ferror)
 * โดยอ่านไฟล์ตัวเลขทีละบรรทัดด้วย fscanf */
#include <stdio.h>
#include <stdlib.h>

int main(void) {
    FILE *fp = fopen("numbers.txt", "w");
    if (fp == NULL) {
        perror("fopen (เขียน) ล้มเหลว");
        return EXIT_FAILURE;
    }
    fprintf(fp, "10\n20\n30\n");
    fclose(fp);

    fp = fopen("numbers.txt", "r");
    if (fp == NULL) {
        perror("fopen (อ่าน) ล้มเหลว");
        return EXIT_FAILURE;
    }

    int value;
    int sum = 0;
    int count = 0;

    /* fscanf คืนจำนวน field ที่แปลงสำเร็จ, หรือ EOF เมื่อจบไฟล์/error */
    while (fscanf(fp, "%d", &value) == 1) {
        sum += value;
        count++;
    }

    if (ferror(fp)) {
        fprintf(stderr, "เกิดข้อผิดพลาดขณะอ่านไฟล์\n");
        fclose(fp);
        return EXIT_FAILURE;
    }

    if (feof(fp)) {
        printf("อ่านจนจบไฟล์ตามปกติ อ่านได้ %d ตัวเลข ผลรวม = %d\n", count, sum);
    }

    fclose(fp);
    return EXIT_SUCCESS;
}
```

ผลลัพธ์:

```
อ่านจนจบไฟล์ตามปกติ อ่านได้ 3 ตัวเลข ผลรวม = 60
```

> **จุดที่ต้องระวังมาก**: ถ้า `fscanf` คืนค่าน้อยกว่าจำนวนที่ขอ (ในตัวอย่างนี้คือน้อยกว่า 1)
> **ไม่ได้แปลว่าเกิด `feof` หรือ `ferror` เสมอไป** ลองสมมติว่าไฟล์มีเนื้อหา `"10 abc 30"`
> — เมื่อ `fscanf` เจอ `"abc"` ในตำแหน่งที่คาดหวังตัวเลข มันจะคืนค่า `0` (แปลงไม่สำเร็จ)
> ทันที โดยที่ **ทั้ง `feof(fp)` และ `ferror(fp)` จะเป็นเท็จทั้งคู่** เพราะตัวไฟล์เองยังไม่ได้
> จบและไม่ได้เกิด I/O error แต่อย่างใด ปัญหาคือ **รูปแบบข้อมูลไม่ตรงกับที่คาดหวัง** ดังนั้น
> เวลาตรวจสอบผลลัพธ์ของ `fscanf` ควรตรวจค่าที่มันคืนกลับมาก่อนเป็นอันดับแรกเสมอ แล้วค่อยใช้
> `feof`/`ferror` เพื่อวินิจฉัยว่า "ทำไม" มันถึงหยุดอ่าน ไม่ใช่ใช้แทนกันได้

---

## 13.6 fseek, ftell และ rewind: การเข้าถึงไฟล์แบบสุ่ม (Random Access) (Step 102)

ที่ผ่านมาเราอ่านไฟล์แบบ "เรียงลำดับ" (Sequential Access) เสมอ คืออ่านตั้งแต่ต้นไปจนจบ แต่ไฟล์
บนดิสก์ (ต่างจากการอ่านจากเครือข่ายหรือ terminal) รองรับการ **กระโดด** ไปอ่าน/เขียนที่ตำแหน่ง
ใดก็ได้โดยตรง เรียกว่า **Random Access** ซึ่งมีประโยชน์มากเมื่อข้อมูลแต่ละ record มีขนาด
คงที่ (เช่น struct binary) เพราะเราคำนวณตำแหน่งของ record ที่ต้องการได้ล่วงหน้าโดยไม่ต้อง
อ่านทุก record ก่อนหน้าเลย

| ฟังก์ชัน | หน้าที่ |
|---|---|
| `long ftell(FILE *fp)` | คืนตำแหน่งปัจจุบันของเคอร์เซอร์ในไฟล์ (นับจากต้นไฟล์ = 0, หน่วยเป็น byte) |
| `int fseek(FILE *fp, long offset, int whence)` | ย้ายเคอร์เซอร์ไปยังตำแหน่งใหม่ |
| `void rewind(FILE *fp)` | ย้ายเคอร์เซอร์กลับไปต้นไฟล์ และล้าง error/EOF flag ให้ด้วย |

พารามิเตอร์ `whence` ของ `fseek` มี 3 ค่า:

| ค่า | ความหมาย |
|---|---|
| `SEEK_SET` | นับ `offset` จาก **ต้นไฟล์** |
| `SEEK_CUR` | นับ `offset` จาก **ตำแหน่งปัจจุบัน** |
| `SEEK_END` | นับ `offset` จาก **ท้ายไฟล์** (มักใช้ `offset = 0` เพื่อหาความยาวไฟล์ทั้งหมด) |

```c
/* random_access.c
 * สาธิต fseek/ftell/rewind สำหรับเข้าถึงไฟล์ binary แบบสุ่ม (Random Access)
 * โดยใช้ไฟล์ struct Student จากตัวอย่างก่อนหน้า */
#include <stdio.h>
#include <stdlib.h>

typedef struct {
    int id;
    char name[32];
    double gpa;
} Student;

int main(void) {
    Student roster[3] = {
        {1, "Somchai", 3.45},
        {2, "Suda", 3.90},
        {3, "Anan", 2.75}
    };

    FILE *fp = fopen("students2.bin", "wb+");
    if (fp == NULL) {
        perror("fopen ล้มเหลว");
        return EXIT_FAILURE;
    }

    fwrite(roster, sizeof(Student), 3, fp);

    /* ftell บอกตำแหน่งปัจจุบันของ "เคอร์เซอร์" ในไฟล์ (นับจากต้นไฟล์ = 0) */
    long pos_after_write = ftell(fp);
    printf("ตำแหน่งหลังเขียนครบ 3 records = %ld bytes\n", pos_after_write);

    /* ต้องการอ่าน record ที่ 2 (index 1) โดยไม่ต้องอ่านตั้งแต่ต้น
       ใช้ fseek กระโดดไปตำแหน่ง index * sizeof(Student) จากต้นไฟล์ (SEEK_SET) */
    long target_offset = 1L * (long)sizeof(Student);
    if (fseek(fp, target_offset, SEEK_SET) != 0) {
        perror("fseek ล้มเหลว");
        fclose(fp);
        return EXIT_FAILURE;
    }

    Student second;
    if (fread(&second, sizeof(Student), 1, fp) != 1) {
        fprintf(stderr, "อ่าน record ที่ 2 ไม่สำเร็จ\n");
        fclose(fp);
        return EXIT_FAILURE;
    }
    printf("record ที่ 2 (สุ่มเข้าถึงโดยตรง): id=%d name=%s gpa=%.2f\n",
           second.id, second.name, second.gpa);

    /* rewind กลับไปต้นไฟล์ เทียบเท่า fseek(fp, 0, SEEK_SET) แต่ยังล้าง error flag ให้ด้วย */
    rewind(fp);
    Student first;
    fread(&first, sizeof(Student), 1, fp);
    printf("record ที่ 1 (หลัง rewind): id=%d name=%s gpa=%.2f\n",
           first.id, first.name, first.gpa);

    /* ใช้ SEEK_END เพื่อหาความยาวไฟล์ทั้งหมด (byte) */
    fseek(fp, 0, SEEK_END);
    long file_size = ftell(fp);
    printf("ขนาดไฟล์ทั้งหมด = %ld bytes (เท่ากับ %ld records)\n",
           file_size, file_size / (long)sizeof(Student));

    fclose(fp);
    return EXIT_SUCCESS;
}
```

ผลลัพธ์:

```
ตำแหน่งหลังเขียนครบ 3 records = 144 bytes
record ที่ 2 (สุ่มเข้าถึงโดยตรง): id=2 name=Suda gpa=3.90
record ที่ 1 (หลัง rewind): id=1 name=Somchai gpa=3.45
ขนาดไฟล์ทั้งหมด = 144 bytes (เท่ากับ 3 records)
```

สังเกตว่าเราเปิดไฟล์ด้วยโหมด `"wb+"` (เขียนและอ่านได้ในไฟล์เดียวกัน) เพื่อให้เขียนแล้ว
`fseek` กลับไปอ่านได้ทันทีโดยไม่ต้องปิดแล้วเปิดใหม่ สูตรคำนวณตำแหน่งของ record ที่ `i`
คือ `i * sizeof(RecordType)` — หลักการเดียวกับสูตร Row-Major Order ที่เรียนใน Part 12
เพียงแต่คราวนี้ "แถว" คือ record หนึ่งตัวในไฟล์แทนที่จะเป็นแถวใน array ในหน่วยความจำ

### แก้ไข record บางตัวโดยไม่กระทบ record อื่น

เทคนิคนี้มีประโยชน์มากเมื่อต้องการอัปเดตข้อมูลเพียงบางส่วนของไฟล์ขนาดใหญ่ โดยไม่ต้องอ่าน
ทั้งไฟล์เข้าหน่วยความจำแล้วเขียนทับใหม่ทั้งหมด

```c
/* update_inplace.c
 * สาธิตโหมด "r+b": เปิดไฟล์ binary ที่มีอยู่แล้วเพื่อ "แก้ไขบางส่วน" โดยไม่ต้อง
 * เขียนไฟล์ใหม่ทั้งหมด ใช้ fseek กระโดดไปตำแหน่ง record ที่ต้องการแล้วเขียนทับ */
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

typedef struct {
    int id;
    char name[32];
    double gpa;
} Student;

int main(void) {
    Student roster[3] = {
        {1, "Somchai", 3.45},
        {2, "Suda", 3.90},
        {3, "Anan", 2.75}
    };

    FILE *fp = fopen("update_demo.bin", "wb");
    if (fp == NULL) {
        perror("fopen ล้มเหลว");
        return EXIT_FAILURE;
    }
    fwrite(roster, sizeof(Student), 3, fp);
    fclose(fp);

    /* เปิดไฟล์เดิมด้วย "r+b" (อ่าน+เขียน, ไฟล์ต้องมีอยู่แล้ว, ไม่ล้างข้อมูลเดิม) */
    fp = fopen("update_demo.bin", "r+b");
    if (fp == NULL) {
        perror("fopen ล้มเหลว");
        return EXIT_FAILURE;
    }

    /* แก้ไข gpa ของนักเรียนคนที่ 2 (index 1) เป็น 4.00 โดยไม่แตะ record อื่นเลย */
    long target = 1L * (long)sizeof(Student);
    fseek(fp, target, SEEK_SET);

    Student updated = {2, "Suda", 4.00};
    fwrite(&updated, sizeof(Student), 1, fp);
    fclose(fp);

    /* อ่านทั้งหมดกลับมาตรวจสอบ */
    fp = fopen("update_demo.bin", "rb");
    if (fp == NULL) {
        perror("fopen ล้มเหลว");
        return EXIT_FAILURE;
    }
    Student check[3];
    fread(check, sizeof(Student), 3, fp);
    fclose(fp);

    for (int i = 0; i < 3; i++) {
        printf("id=%d name=%-8s gpa=%.2f\n", check[i].id, check[i].name, check[i].gpa);
    }

    return EXIT_SUCCESS;
}
```

ผลลัพธ์:

```
id=1 name=Somchai  gpa=3.45
id=2 name=Suda     gpa=4.00
id=3 name=Anan     gpa=2.75
```

จะเห็นว่ามีแค่ record ของ `id=2` เท่านั้นที่เปลี่ยนแปลง (gpa จาก 3.90 เป็น 4.00) ส่วน record
อื่นไม่ถูกกระทบเลย — นี่คือพลังของ Random Access ที่ `fseek` มอบให้

---

## 13.7 ตัวอย่างจริง: บันทึกและโหลดรายชื่อนักเรียน (Text vs Binary) (Step 103–104)

มารวมทุกอย่างที่เรียนมาในบทนี้ เขียนโปรแกรมที่ใช้งานได้จริง: ระบบบันทึก/โหลดรายชื่อนักเรียน
(`Student` struct array) ลงไฟล์ทั้งสองแบบ — **text** (มนุษย์อ่านได้, แก้ไขด้วย text editor
ได้ง่าย, portable ข้ามแพลตฟอร์ม 100%) และ **binary** (เร็วกว่า, ขนาดไฟล์เล็กกว่าและคงที่
ต่อ record, เหมาะกับข้อมูลจำนวนมาก) โดยออกแบบ interface ของทั้งสองแบบให้หน้าตาคล้ายกัน
เพื่อให้เลือกใช้แบบไหนก็ได้ตามความต้องการ

```c
/* student_roster.c
 * โปรแกรมตัวอย่างจริง: บันทึก/โหลดรายชื่อนักเรียน (struct array) ลงไฟล์
 * ทั้งแบบ text (มนุษย์อ่านได้, portable) และแบบ binary (เร็ว, ขนาดคงที่)
 */
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#define MAX_STUDENTS 100
#define NAME_LEN 32

typedef struct {
    int id;
    char name[NAME_LEN];
    double gpa;
} Student;

/* ---------- บันทึก/โหลดแบบ TEXT ---------- */

/* บันทึก roster เป็นไฟล์ข้อความ หนึ่งบรรทัดต่อหนึ่งคน คั่นด้วยช่องว่าง
 * คืนค่า 1 ถ้าสำเร็จ, 0 ถ้าเปิดไฟล์ไม่ได้ */
int save_students_text(const char *filename, const Student *list, size_t count) {
    FILE *fp = fopen(filename, "w");
    if (fp == NULL) {
        perror("save_students_text: fopen ล้มเหลว");
        return 0;
    }
    for (size_t i = 0; i < count; i++) {
        fprintf(fp, "%d %s %.2f\n", list[i].id, list[i].name, list[i].gpa);
    }
    fclose(fp);
    return 1;
}

/* โหลด roster จากไฟล์ข้อความ เขียนผลลัพธ์ลงใน out (ขนาดสูงสุด max_count)
 * คืนจำนวนรายการที่โหลดได้จริง (0 ถ้าเปิดไฟล์ไม่ได้) */
size_t load_students_text(const char *filename, Student *out, size_t max_count) {
    FILE *fp = fopen(filename, "r");
    if (fp == NULL) {
        perror("load_students_text: fopen ล้มเหลว");
        return 0;
    }

    size_t count = 0;
    while (count < max_count &&
           fscanf(fp, "%d %31s %lf", &out[count].id, out[count].name, &out[count].gpa) == 3) {
        count++;
    }

    if (ferror(fp)) {
        fprintf(stderr, "load_students_text: เกิดข้อผิดพลาดขณะอ่านไฟล์\n");
    }

    fclose(fp);
    return count;
}

/* ---------- บันทึก/โหลดแบบ BINARY ---------- */

/* บันทึก roster เป็นไฟล์ binary ดิบๆ ของ struct ทั้งก้อน
 * คืนค่า 1 ถ้าสำเร็จ, 0 ถ้าล้มเหลว (เปิดไฟล์ไม่ได้ หรือเขียนไม่ครบ) */
int save_students_binary(const char *filename, const Student *list, size_t count) {
    FILE *fp = fopen(filename, "wb");
    if (fp == NULL) {
        perror("save_students_binary: fopen ล้มเหลว");
        return 0;
    }

    size_t written = fwrite(list, sizeof(Student), count, fp);
    fclose(fp);

    if (written != count) {
        fprintf(stderr, "save_students_binary: เขียนได้ %zu จาก %zu รายการ\n",
                written, count);
        return 0;
    }
    return 1;
}

/* โหลด roster จากไฟล์ binary เขียนผลลัพธ์ลงใน out (ขนาดสูงสุด max_count)
 * คืนจำนวนรายการที่โหลดได้จริง */
size_t load_students_binary(const char *filename, Student *out, size_t max_count) {
    FILE *fp = fopen(filename, "rb");
    if (fp == NULL) {
        perror("load_students_binary: fopen ล้มเหลว");
        return 0;
    }

    size_t count = fread(out, sizeof(Student), max_count, fp);

    if (ferror(fp)) {
        fprintf(stderr, "load_students_binary: เกิดข้อผิดพลาดขณะอ่านไฟล์\n");
    }

    fclose(fp);
    return count;
}

/* ---------- ฟังก์ชันช่วยแสดงผล ---------- */

void print_students(const Student *list, size_t count) {
    printf("%-4s %-10s %s\n", "ID", "Name", "GPA");
    for (size_t i = 0; i < count; i++) {
        printf("%-4d %-10s %.2f\n", list[i].id, list[i].name, list[i].gpa);
    }
}

int main(void) {
    Student roster[3] = {
        {101, "Somchai", 3.45},
        {102, "Suda", 3.90},
        {103, "Anan", 2.75}
    };
    size_t roster_count = 3;

    printf("=== ข้อมูลต้นฉบับ ===\n");
    print_students(roster, roster_count);

    /* --- ทดสอบรอบ text --- */
    if (!save_students_text("roster.txt", roster, roster_count)) {
        return EXIT_FAILURE;
    }
    printf("\n=== โหลดกลับจาก roster.txt (text) ===\n");
    Student loaded_text[MAX_STUDENTS];
    size_t n_text = load_students_text("roster.txt", loaded_text, MAX_STUDENTS);
    print_students(loaded_text, n_text);

    /* --- ทดสอบรอบ binary --- */
    if (!save_students_binary("roster.bin", roster, roster_count)) {
        return EXIT_FAILURE;
    }
    printf("\n=== โหลดกลับจาก roster.bin (binary) ===\n");
    Student loaded_bin[MAX_STUDENTS];
    size_t n_bin = load_students_binary("roster.bin", loaded_bin, MAX_STUDENTS);
    print_students(loaded_bin, n_bin);

    return EXIT_SUCCESS;
}
```

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 student_roster.c -o student_roster
./student_roster
```

ผลลัพธ์:

```
=== ข้อมูลต้นฉบับ ===
ID   Name       GPA
101  Somchai    3.45
102  Suda       3.90
103  Anan       2.75

=== โหลดกลับจาก roster.txt (text) ===
ID   Name       GPA
101  Somchai    3.45
102  Suda       3.90
103  Anan       2.75

=== โหลดกลับจาก roster.bin (binary) ===
ID   Name       GPA
101  Somchai    3.45
102  Suda       3.90
103  Anan       2.75
```

เนื้อหาจริงของ `roster.txt` ที่ได้ (เปิดด้วย text editor ทั่วไปอ่านได้เลย):

```
101 Somchai 3.45
102 Suda 3.90
103 Anan 2.75
```

ในขณะที่ `roster.bin` มีขนาด 144 byte พอดี (3 × 48 byte ต่อ record ตามที่คำนวณไว้ในหัวข้อ
13.4) และเปิดด้วย text editor จะเห็นเป็นตัวอักษรแปลกๆ ปนกัน เพราะเป็นไบต์ดิบของ struct
ไม่ใช่ข้อความ

### เปรียบเทียบแนวทาง Text vs Binary สำหรับเก็บข้อมูล struct

| ประเด็น | บันทึกแบบ Text | บันทึกแบบ Binary |
|---|---|---|
| มนุษย์เปิดอ่าน/แก้ไขด้วยมือได้ไหม | ได้ (เปิดด้วย text editor ทั่วไป) | ไม่ได้ (ต้องเขียนโปรแกรมอ่านโดยเฉพาะ) |
| ขนาดไฟล์ | มักใหญ่กว่า (ตัวเลขถูกเก็บเป็นตัวอักษร) | เล็กกว่าและคงที่ต่อ record เสมอ |
| ความเร็วในการอ่าน/เขียน | ช้ากว่า (ต้องแปลงข้อความ ↔ ตัวเลขทุกครั้ง) | เร็วกว่ามาก (คัดลอกไบต์ดิบตรงๆ) |
| รองรับ Random Access (`fseek` ไป record ที่ N) | ยาก (แต่ละบรรทัดยาวไม่เท่ากัน คำนวณตำแหน่งล่วงหน้าไม่ได้) | ง่ายมาก (`N * sizeof(Struct)` คำนวณตำแหน่งได้ทันที) |
| Portable ข้าม platform/compiler | สูงมาก (เป็นข้อความล้วนๆ) | ต่ำกว่า (ขึ้นกับ padding, endianness ตามที่อธิบายในหัวข้อ 13.4) |
| ใช้ควบคุมด้วย version control (git diff) | อ่าน diff ได้ง่าย | อ่าน diff ไม่ได้เลย |

> **คำแนะนำในทางปฏิบัติ**: ถ้าข้อมูลมีขนาดเล็ก ต้องการแก้ไขด้วยมือได้ หรือต้องแชร์ข้าม
> ระบบที่ไม่รู้จักกัน ให้เลือก **text** (หรือรูปแบบมาตรฐานอย่าง JSON/CSV ที่จะเรียนภายหลัง)
> แต่ถ้าข้อมูลมีจำนวนมาก ต้องการความเร็วสูงสุด หรือต้องทำ Random Access บ่อยๆ (เช่นฐานข้อมูล
> ขนาดเล็กของตัวเอง) ให้เลือก **binary** — หัวข้อนี้จะกลับมาเจออีกครั้งอย่างจริงจังใน
> **Part 40 (Mini Key-Value Store Engine)**

---

## ข้อผิดพลาดที่พบบ่อย (Common Pitfalls)

1. **ไม่ตรวจสอบค่าที่ `fopen` คืนกลับมา** — ถ้า `fopen` คืน `NULL` (ไฟล์ไม่มีอยู่, ไม่มีสิทธิ์
   เข้าถึง, ดิสก์เต็ม) แล้วยังส่ง pointer นั้นให้ `fprintf`/`fread`/`fwrite` ต่อ จะเป็น
   Undefined Behavior ทันที (โปรแกรม crash หรือแย่กว่านั้นคือทำงานผิดแบบเงียบๆ) ต้องเช็ค
   `if (fp == NULL)` ทุกครั้งหลัง `fopen`
2. **ลืมเรียก `fclose` ทำให้ข้อมูลไม่ถูกเขียนลงดิสก์จริง** — stdio buffer ข้อมูลที่เขียนไว้
   ใน memory ก่อนเสมอ (เพื่อความเร็ว) แล้วค่อย flush ลงดิสก์เป็นก้อนใหญ่ๆ การเรียก `fclose`
   (หรือ `exit`/`return` จาก `main` ตามปกติ) จะบังคับ flush buffer ให้ แต่ถ้าโปรแกรมจบแบบ
   ผิดปกติ (เช่นถูก `kill` หรือเรียก `_exit()` ตรงๆ) ข้อมูลที่ค้างอยู่ใน buffer จะ **หายไป
   ทันทีโดยไม่มีการเตือน** ลองดูตัวอย่างพิสูจน์:

   ```c
   /* no_fclose_demo.c
    * DEMO บั๊กโดยตั้งใจ: เขียนข้อมูลแต่ "ลืม" fclose แล้วจบโปรแกรมด้วย _exit()
    * ทันที (จำลองโปรแกรม crash หรือถูก kill กะทันหัน) */
   #include <stdio.h>
   #include <unistd.h>

   int main(void) {
       FILE *fp = fopen("no_flush.txt", "w");
       if (fp == NULL) {
           return 1;
       }

       fprintf(fp, "ข้อความนี้อาจไม่ถูกเขียนลงไฟล์จริงถ้าไม่ fclose/fflush\n");

       /* _exit() จบโปรแกรมทันที "โดยไม่" เรียก stdio cleanup เลย
          ต่างจาก exit()/return จาก main ที่มาตรฐาน C รับประกันว่าจะ flush
          บัฟเฟอร์ของทุกไฟล์ที่เปิดค้างไว้ให้อัตโนมัติก่อนจบโปรแกรมเสมอ */
       _exit(0);
   }
   ```

   ```bash
   gcc -Wall -Wextra -Wpedantic -std=c17 no_fclose_demo.c -o no_fclose_demo
   ./no_fclose_demo
   wc -c no_flush.txt
   ```

   ผลลัพธ์:

   ```
   0 no_flush.txt
   ```

   ไฟล์มีอยู่จริง แต่ **มีขนาด 0 byte** — ข้อความที่ `fprintf` เขียนไปทั้งหมดหายไปเพราะยัง
   ค้างอยู่ใน buffer ตอนที่โปรแกรมถูกตัดจบ เทียบกับเวอร์ชันที่เรียก `fclose` ก่อนเสมอ:

   ```c
   /* with_fclose_demo.c — เวอร์ชันที่ถูกต้อง */
   #include <stdio.h>
   #include <stdlib.h>
   #include <unistd.h>

   int main(void) {
       FILE *fp = fopen("with_flush.txt", "w");
       if (fp == NULL) {
           return EXIT_FAILURE;
       }

       fprintf(fp, "ข้อความนี้จะถูกเขียนลงไฟล์แน่นอน เพราะเรียก fclose\n");

       fclose(fp); /* บังคับ flush + ปิด file descriptor */

       _exit(0); /* ต่อให้ยังจบแบบดุดันด้วย _exit ข้อมูลก็ปลอดภัยแล้ว */
   }
   ```

   ```
   135 with_flush.txt
   ```

   คราวนี้ไฟล์มีข้อมูลครบ 135 byte เพราะ `fclose` ถูกเรียกไปแล้วก่อนที่ `_exit` จะทำงาน —
   **บทเรียน: `fclose` ทุกไฟล์ที่เปิดเสมอ ไม่ว่าโปรแกรมจะจบแบบไหนก็ตาม**
3. **บัฟเฟอร์ของ `fgets` เล็กเกินไป** — ถ้าบรรทัดในไฟล์ยาวกว่าบัฟเฟอร์ที่กำหนด `fgets` จะ
   **ไม่ overflow** (ปลอดภัยกว่า `gets` มาก) แต่จะตัดบรรทัดเดียวออกเป็นหลายส่วนโดยไม่บอก
   เราเลย ทำให้ logic ที่คาดหวังว่า "1 ครั้งของ fgets = 1 บรรทัดเต็มๆ" ผิดพลาดไปโดยไม่รู้ตัว

   ```c
   /* fgets_small_buffer.c — DEMO บั๊กโดยตั้งใจ: บัฟเฟอร์เล็กเกินไป (8 byte) */
   #define TINY_BUF_SIZE 8
   char buf[TINY_BUF_SIZE];
   /* ไฟล์มีบรรทัดเดียว: "This line is definitely longer than eight bytes\n" */
   while (fgets(buf, sizeof buf, fp) != NULL) {
       printf("chunk: \"%s\"\n", buf);
   }
   ```

   ผลลัพธ์ (บรรทัดเดียวถูกตัดออกเป็น 7 ชิ้น):

   ```
   chunk 1: "This li"
   chunk 2: "ne is d"
   chunk 3: "efinite"
   chunk 4: "ly long"
   chunk 5: "er than"
   chunk 6: " eight "
   chunk 7: "bytes
   "
   ```

   วิธีแก้: กำหนดขนาดบัฟเฟอร์ให้ใหญ่พอสำหรับข้อมูลจริงเสมอ (เช่น 256 หรือ 1024 byte สำหรับ
   ไฟล์ config/log ทั่วไป) หรือถ้าความยาวบรรทัดไม่แน่นอนเลย ให้ตรวจสอบว่าอ่านได้ครบบรรทัด
   หรือไม่ (เช็คว่ามี `'\n'` อยู่ในบัฟเฟอร์ที่อ่านมาหรือเปล่า) แล้ววนอ่านเพิ่มถ้ายังไม่ครบ
4. **สับสนระหว่าง `fscanf`/`fprintf` (text) กับ `fread`/`fwrite` (binary)** — การเปิดไฟล์
   ด้วย `"wb"` แล้วเขียนด้วย `fprintf` ทำได้ (เพราะบน Linux ไม่มีผลต่างจาก text mode) แต่
   ผลลัพธ์จะเป็นข้อความ ไม่ใช่ binary ของ struct ทำให้ตอนอ่านกลับด้วย `fread` จะได้ข้อมูล
   ผิดเพี้ยนทันที ต้องใช้คู่ให้ตรงกันเสมอ: text file คู่กับ `fprintf`/`fgets`/`fscanf`,
   binary file คู่กับ `fwrite`/`fread`
5. **ลืมตรวจสอบค่าที่ `fread`/`fwrite` คืนกลับมา** — ถ้าดิสก์เต็มระหว่างเขียน หรือไฟล์สั้น
   กว่าที่คาดระหว่างอ่าน ฟังก์ชันจะคืนจำนวนที่ทำสำเร็จจริง (น้อยกว่าที่ขอ) โดยไม่ throw
   error หรือ crash ให้เห็นชัดๆ ถ้าไม่เช็คค่าที่คืนมา โปรแกรมจะเดินหน้าทำงานต่อด้วยข้อมูล
   ที่ไม่ครบโดยไม่รู้ตัว
6. **ใช้ `feof(fp)` เป็นเงื่อนไขของ loop โดยตรง** (เช่น `while (!feof(fp)) { fscanf(...); }`)
   — เป็นบั๊กคลาสสิกที่พบบ่อยมาก เพราะ `feof` จะเป็นจริง **หลังจาก** พยายามอ่านแล้วเจอจุดจบ
   ไฟล์เท่านั้น ทำให้เกิดการอ่านครั้งสุดท้ายที่ล้มเหลว (ได้ค่าขยะ) แต่ loop ยังทำงานต่ออีก
   1 รอบก่อนจะออก วิธีที่ถูกต้องคือใช้ **ค่าที่ฟังก์ชันอ่านคืนกลับมาโดยตรง** เป็นเงื่อนไข
   (`while (fscanf(fp, "%d", &x) == 1)` หรือ `while (fgets(...) != NULL)`) ตามที่ใช้ใน
   ทุกตัวอย่างของบทนี้
7. **เปิดไฟล์ binary โดยลืมใส่ `b` ใน mode string** — บน Linux จะไม่มีอาการอะไรให้เห็นเลย
   (เพราะ Linux เพิกเฉยต่อ `b` อยู่แล้ว) แต่ถ้าโค้ดเดียวกันถูกคอมไพล์บน Windows จะเกิดบั๊ก
   ข้อมูลเพี้ยนทันทีตามที่อธิบายในหัวข้อ 13.3 ให้ใส่ `b` เป็นนิสัยเสมอเมื่อทำงานกับข้อมูล
   binary เพื่อความปลอดภัยข้ามแพลตฟอร์ม

---

## แบบฝึกหัดท้ายบท

1. เขียนโปรแกรม `line_word_char_count.c` ที่นับจำนวนบรรทัด จำนวนคำ และจำนวนตัวอักษรทั้งหมด
   ในไฟล์ข้อความ (ทำงานคล้ายคำสั่ง `wc` ของ Linux) โดยอ่านไฟล์ทีละบรรทัดด้วย `fgets`
2. โค้ดในหัวข้อ Common Pitfalls ข้อ 3 (`fgets_small_buffer.c`) มีบัฟเฟอร์เล็กเกินไปทำให้
   บรรทัดยาวถูกตัดออกเป็นหลายชิ้น ให้แก้ไขโปรแกรมนั้นให้อ่านบรรทัดได้ครบถ้วนในครั้งเดียว
   (คำใบ้: เปลี่ยนขนาดบัฟเฟอร์ให้ใหญ่พอ แล้วพิสูจน์ด้วยการรันโปรแกรมว่าได้ผลลัพธ์เป็น
   1 chunk เท่านั้น)
3. เขียนฟังก์ชัน `int append_student_binary(const char *filename, const Student *s)` ที่
   เพิ่ม record นักเรียนคนใหม่ต่อท้ายไฟล์ binary ที่มีอยู่แล้ว โดยไม่ทำลายข้อมูลเดิมที่มีอยู่
   (คำใบ้: เลือก mode string ที่เหมาะสมจากตารางในหัวข้อ 13.1)
4. เขียนฟังก์ชัน `int find_and_update_gpa(const char *filename, int id, double new_gpa)`
   ที่ค้นหานักเรียนตาม `id` ในไฟล์ binary แล้วอัปเดตค่า `gpa` ที่ตำแหน่งเดิมในไฟล์โดยตรง
   (ห้ามโหลดทั้งไฟล์เข้า array ในหน่วยความจำ ต้องใช้ `fseek` วนอ่านทีละ record แทน)
5. เขียนโปรแกรมที่อ่านไฟล์ตัวเลข (บรรทัดละ 1 ค่า) แล้วคำนวณค่าเฉลี่ย ค่าน้อยสุด และค่ามากสุด
   พร้อมตรวจสอบข้อผิดพลาดด้วย `ferror` ให้ครบถ้วนตามที่เรียนในหัวข้อ 13.5
6. เขียนโปรแกรม `text_to_binary.c` ที่แปลงไฟล์ `roster.txt` (รูปแบบข้อความจากหัวข้อ 13.7)
   ให้กลายเป็น `roster.bin` (รูปแบบ binary) โดยใช้ฟังก์ชัน `load_students_text` และ
   `save_students_binary` ที่เขียนไว้แล้วในบทเรียนนี้มาประกอบกัน

### แนวทางเฉลยข้อ 1

```c
/* ex1_wc.c
 * เฉลยข้อ 1: นับจำนวนบรรทัด คำ และตัวอักษรทั้งหมดในไฟล์ข้อความ (คล้าย wc) */
#include <stdio.h>
#include <stdlib.h>
#include <ctype.h>

#define LINE_BUF_SIZE 512

int main(void) {
    const char *filename = "wc_input.txt";

    /* สร้างไฟล์ตัวอย่างไว้ทดสอบก่อน */
    FILE *out = fopen(filename, "w");
    if (out == NULL) {
        perror("fopen ล้มเหลว");
        return EXIT_FAILURE;
    }
    fputs("Hello world from C\n", out);
    fputs("File I/O is fun\n", out);
    fputs("Goodbye\n", out);
    fclose(out);

    FILE *fp = fopen(filename, "r");
    if (fp == NULL) {
        perror("fopen ล้มเหลว");
        return EXIT_FAILURE;
    }

    long line_count = 0;
    long word_count = 0;
    long char_count = 0;
    char line[LINE_BUF_SIZE];

    while (fgets(line, sizeof line, fp) != NULL) {
        line_count++;

        int in_word = 0;
        for (const char *p = line; *p != '\0'; p++) {
            char_count++;
            if (isspace((unsigned char)*p)) {
                in_word = 0;
            } else if (!in_word) {
                in_word = 1;
                word_count++;
            }
        }
    }

    if (ferror(fp)) {
        fprintf(stderr, "เกิดข้อผิดพลาดขณะอ่านไฟล์\n");
        fclose(fp);
        return EXIT_FAILURE;
    }

    fclose(fp);

    printf("ไฟล์ %s: %ld บรรทัด, %ld คำ, %ld ตัวอักษร\n",
           filename, line_count, word_count, char_count);

    return EXIT_SUCCESS;
}
```

ผลลัพธ์ (ตรวจสอบตรงกับคำสั่ง `wc` ของระบบจริง):

```
ไฟล์ wc_input.txt: 3 บรรทัด, 9 คำ, 43 ตัวอักษร
```

จุดสำคัญของเฉลยนี้คือการใช้ `isspace((unsigned char)*p)` แทน `isspace(*p)` ตรงๆ — ฟังก์ชัน
ใน `<ctype.h>` ทุกตัวคาดหวังพารามิเตอร์เป็นค่าที่แทนได้ด้วย `unsigned char` หรือ `EOF`
เท่านั้น ถ้า `char` บนแพลตฟอร์มนั้นเป็น signed (ซึ่งพบได้บ่อย) และตัวอักษรมีค่าติดลบ (เช่น
ไบต์ของ multi-byte character อย่างภาษาไทยที่เข้ารหัส UTF-8) การส่งค่าติดลบเข้า `isspace`
ตรงๆ จะเป็น Undefined Behavior การ cast เป็น `unsigned char` ก่อนเสมอจึงเป็นนิสัยที่ถูกต้อง

### แนวทางเฉลยข้อ 3

```c
/* ex3_append_binary.c
 * เฉลยข้อ 3: เพิ่ม record นักเรียนคนใหม่ต่อท้ายไฟล์ binary เดิม โดยไม่ทำลาย
 * ข้อมูลเดิมที่มีอยู่แล้ว (ใช้โหมด "ab" ซึ่งเขียนต่อท้ายเสมอไม่ว่าจะ fseek
 * ไปที่ใดก่อนหน้าก็ตาม) */
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

typedef struct {
    int id;
    char name[32];
    double gpa;
} Student;

/* เพิ่ม new_student ต่อท้ายไฟล์ binary ที่ filename ชี้อยู่
 * ถ้าไฟล์ยังไม่มีอยู่ โหมด "ab" จะสร้างไฟล์ใหม่ให้อัตโนมัติ
 * คืนค่า 1 ถ้าสำเร็จ, 0 ถ้าล้มเหลว */
int append_student_binary(const char *filename, const Student *new_student) {
    FILE *fp = fopen(filename, "ab");
    if (fp == NULL) {
        perror("append_student_binary: fopen ล้มเหลว");
        return 0;
    }

    size_t written = fwrite(new_student, sizeof *new_student, 1, fp);
    fclose(fp);

    if (written != 1) {
        fprintf(stderr, "append_student_binary: เขียนไม่สำเร็จ\n");
        return 0;
    }
    return 1;
}

size_t load_all_students(const char *filename, Student *out, size_t max_count) {
    FILE *fp = fopen(filename, "rb");
    if (fp == NULL) {
        return 0;
    }
    size_t count = fread(out, sizeof *out, max_count, fp);
    fclose(fp);
    return count;
}

int main(void) {
    const char *filename = "append_roster.bin";
    remove(filename); /* เริ่มจากไฟล์ว่างเสมอเพื่อให้ตัวอย่างนี้ทำซ้ำได้ผลเหมือนเดิม */

    Student a = {1, "Somchai", 3.45};
    Student b = {2, "Suda", 3.90};
    Student c = {3, "Anan", 2.75};

    append_student_binary(filename, &a);
    append_student_binary(filename, &b);
    append_student_binary(filename, &c);

    Student all[10];
    size_t n = load_all_students(filename, all, 10);

    printf("มีนักเรียนทั้งหมด %zu คนในไฟล์:\n", n);
    for (size_t i = 0; i < n; i++) {
        printf("  id=%d name=%-8s gpa=%.2f\n", all[i].id, all[i].name, all[i].gpa);
    }

    return EXIT_SUCCESS;
}
```

ผลลัพธ์:

```
มีนักเรียนทั้งหมด 3 คนในไฟล์:
  id=1 name=Somchai  gpa=3.45
  id=2 name=Suda     gpa=3.90
  id=3 name=Anan     gpa=2.75
```

สังเกตว่าเราเรียก `append_student_binary` แยกกัน 3 ครั้ง โดยแต่ละครั้ง `fopen`/`fclose`
ใหม่หมด แต่ข้อมูลจากรอบก่อนหน้าไม่หายไปเลย เพราะโหมด `"ab"` รับประกันว่าตำแหน่งเขียนจะอยู่
ที่ท้ายไฟล์เสมอ — พฤติกรรมเดียวกับที่พิสูจน์ไว้แล้วในตัวอย่าง `append_mode.c` ของหัวข้อ 13.2
เพียงแต่คราวนี้ใช้กับข้อมูล binary แทนข้อความล้วนๆ (ข้อ 2, 4, 5, 6 ให้ผู้เรียนลองนำโครงสร้าง
ฟังก์ชันจากหัวข้อ 13.2–13.7 มาปรับใช้เอง)

---

## สรุปท้ายบท

ใน Part นี้เราได้:

- เปิด-ปิดไฟล์ด้วย `fopen`/`fclose` อย่างถูกต้อง พร้อมเข้าใจความหมายของ mode string ทุกแบบ
  ผ่านตารางสรุปที่ครอบคลุมทั้ง text และ binary mode
- เขียนและอ่านไฟล์ข้อความด้วย `fprintf`/`fputs`/`fgets` ได้อย่างปลอดภัย โดยเข้าใจพฤติกรรม
  ของ `fgets` เรื่องการเก็บ `\n` และขนาดบัฟเฟอร์อย่างละเอียด
- แยกแยะความแตกต่างระหว่าง text mode กับ binary mode ได้ทั้งในทางทฤษฎีและในทางปฏิบัติจริง
  บน Linux เทียบกับ Windows
- อ่าน/เขียนข้อมูล struct ทั้งก้อนลงไฟล์ binary ด้วย `fread`/`fwrite` พร้อมเข้าใจเรื่อง
  struct padding ที่ส่งผลต่อขนาดไฟล์จริง
- แยกแยะ `feof` (จบไฟล์ปกติ) ออกจาก `ferror` (ข้อผิดพลาดจริง) ได้อย่างถูกต้อง และรู้จัก
  กับดักคลาสสิกของการใช้ `feof` เป็นเงื่อนไข loop โดยตรง
- ใช้ `fseek`/`ftell`/`rewind` เพื่อเข้าถึงและแก้ไขข้อมูลในไฟล์แบบสุ่มได้โดยไม่ต้องอ่าน
  ทั้งไฟล์ตั้งแต่ต้น
- เขียนโปรแกรมบันทึก/โหลดรายชื่อนักเรียนที่ใช้งานได้จริง ทั้งแบบ text และแบบ binary พร้อม
  เข้าใจข้อดี-ข้อเสียของแต่ละแนวทางในการเลือกใช้งานจริง

ทักษะเรื่อง File I/O ที่เรียนใน Part นี้จะถูกใช้ซ้ำตลอดหลักสูตรที่เหลือ ตั้งแต่การอ่าน config
ไฟล์ ไปจนถึงการสร้างฐานข้อมูลขนาดเล็กของตัวเองใน Part 40 ใน **Part 14** เราจะเรียนรู้เรื่อง
**Preprocessor และ Macro** อย่างละเอียด ซึ่งเป็นกลไกเบื้องหลัง `#include` และ `#define` ที่
เราใช้มาตั้งแต่ Part 1 โดยยังไม่เคยเจาะลึกว่ามันทำงานอย่างไรจริงๆ

**ต่อไป:** [Part 14 — Preprocessor และ Macro](./part-014-preprocessor-macros.md)
