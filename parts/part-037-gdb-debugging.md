# Part 37: การ Debug ด้วย GDB (Step 289–296)

> Module C — Systems Programming ด้วย C บน Linux | Part 37 จาก 125
> Step ที่ครอบคลุมใน Part นี้: Step 289–296
> Part ก่อนหน้า: [Part 36 — Memory-Mapped File (mmap)](./part-036-mmap.md) | Part ถัดไป: [Part 38 — Valgrind และการตรวจจับ Memory Leak](./part-038-valgrind.md)

## เป้าหมายการเรียนรู้ (Learning Objectives)

หลังจบ Part นี้ ผู้เรียนจะสามารถ:

1. อธิบายได้ว่าทำไมการ debug ด้วย debugger จริงมีประสิทธิภาพกว่าการแทรก `printf()`
   เพื่อไล่หาบั๊ก โดยเฉพาะกับบั๊กที่ซับซ้อน
2. คอมไพล์โปรแกรมด้วย flag `-g` เพื่อเก็บ debug symbol และเข้าใจว่าทำไม GDB ต้องการ
   symbol เหล่านี้ถึงจะแสดงชื่อตัวแปรและเลขบรรทัดของ source code ได้
3. ใช้คำสั่งพื้นฐานของ GDB ได้คล่อง: `break`, `run`, `next`, `step`, `continue`,
   `print`, `backtrace`, `watch`, `info locals`, `info args`, `finish`
4. Debug segmentation fault จริงจนหา root cause ได้ด้วย `backtrace` และ `print`
5. ใช้ `watch` (watchpoint) ตรวจจับจุดที่ตัวแปรเปลี่ยนค่าผิดปกติแบบ off-by-one ได้
6. เข้าใจว่า core dump คืออะไร ตั้งค่าให้ระบบสร้าง core dump ได้ และเปิดวิเคราะห์
   core dump ด้วย GDB แบบ post-mortem (วิเคราะห์ย้อนหลังหลังโปรแกรม crash ไปแล้ว)
7. ใช้ GDB TUI mode (Text User Interface) เพื่อดู source code คู่กับสถานะการ debug
   แบบ real-time ในหน้าจอเดียว
8. Debug บั๊กจริง 2 ประเภทที่พบบ่อยที่สุด (null pointer dereference และ off-by-one)
   ทีละขั้นตอนด้วย GDB จนเข้าใจสาเหตุที่แท้จริง

---

## 37.1 ทำไมต้อง Debug ด้วย Debugger ไม่ใช่แค่ printf (Step 289)

วิธี debug ที่มือใหม่เกือบทุกคนเริ่มต้นด้วยคือการแทรก `printf("มาถึงจุดนี้แล้ว x = %d\n",
x);` ไปทั่วโค้ด แล้ว compile-run-อ่าน output-ลบ printf-ทำซ้ำ วิธีนี้ใช้ได้กับบั๊กง่ายๆ
แต่มีข้อจำกัดร้ายแรงเมื่อบั๊กซับซ้อนขึ้น:

| หัวข้อ | Debug ด้วย `printf()` | Debug ด้วย GDB |
|---|---|---|
| ต้องแก้โค้ดและ compile ใหม่ทุกครั้ง | ใช่ ทุกครั้งที่อยากดูค่าเพิ่ม | ไม่ต้อง — พิมพ์ `print` ดูค่าอะไรก็ได้แบบ real-time |
| ดูค่าตัวแปรที่ "คาดไม่ถึง" ว่าต้องดู | ต้องเดาล่วงหน้าว่าจะพิมพ์ตัวแปรไหน | ดูตัวแปรไหนก็ได้ทันทีที่หยุดอยู่ ณ จุดนั้น |
| หยุดโปรแกรมไว้ชั่วคราวเพื่อตรวจสอบ | ทำไม่ได้ (โปรแกรมวิ่งต่อทันที) | `break`/`watch` หยุดโปรแกรมไว้ตรงจุดที่ต้องการได้ |
| ดู call stack ทั้งหมดตอนเกิดปัญหา | ต้องเขียน printf ในทุกฟังก์ชันที่เกี่ยวข้อง | `backtrace` แสดงทั้ง call chain ในคำสั่งเดียว |
| Debug หลังโปรแกรม crash ไปแล้ว | ทำไม่ได้เลย (ข้อมูลหายไปพร้อมโปรเซส) | เปิด core dump วิเคราะห์ย้อนหลังได้ (ดู 37.7) |
| ลบ debug code ออกก่อน commit | ต้องจำลบทุกจุด (ลืมบ่อยมาก) | ไม่ต้องแก้โค้ดต้นฉบับเลยแม้แต่บรรทัดเดียว |
| Debug โค้ดคนอื่นที่ไม่คุ้นเคย | ยากมาก ไม่รู้จะแทรก printf ตรงไหน | `step`/`next` ไล่ดูการทำงานทีละบรรทัดได้ทันที |

จุดที่สำคัญที่สุดคือข้อสุดท้ายในตาราง: printf debugging ต้อง **แก้ไขโค้ดต้นฉบับ** ทุกครั้ง
ซึ่งหมายความว่าต้อง compile ใหม่ทุกรอบ (ช้าถ้าโปรเจกต์ใหญ่) และเสี่ยงลืมลบโค้ด debug
ออกก่อน commit เข้า repository จริง (เคยเจอ `printf("DEBUG: here!\n");` หลงเหลือใน
production code ไหม? นี่คือสาเหตุ) ในขณะที่ debugger อย่าง GDB ให้เรา **หยุดโปรแกรมไว้ ณ
จุดใดก็ได้ที่ต้องการ** แล้วสำรวจสถานะทั้งหมดของโปรแกรม ณ ขณะนั้นได้อย่างละเอียด โดยไม่ต้อง
แตะโค้ดต้นฉบับเลยแม้แต่บรรทัดเดียว

> ไม่ได้หมายความว่า `printf()` ไม่มีประโยชน์เลย — สำหรับบั๊กง่ายๆ ที่รู้จุดสงสัยชัดเจนแล้ว
> printf ยังคงเป็นเครื่องมือที่เร็วและตรงไปตรงมา แต่เมื่อบั๊กซับซ้อนขึ้น (โดยเฉพาะ
> segmentation fault ที่ไม่รู้ว่าเกิดจากที่ไหน) การเรียนรู้ debugger อย่างจริงจังจะ
> ประหยัดเวลาได้มหาศาลในระยะยาว

---

## 37.2 คอมไพล์ด้วย -g เพื่อเก็บ Debug Symbol (Step 290)

GDB ไม่สามารถแสดงชื่อตัวแปร ชื่อฟังก์ชัน หรือเลขบรรทัดของ source code ได้เลย **ถ้าไม่ได้
คอมไพล์ด้วย flag `-g`** ทบทวนจาก Part 1: `-g` สั่งให้ compiler ฝัง **debug symbol**
(รูปแบบมาตรฐานที่ใช้กันทั่วไปบน Linux คือ DWARF) เข้าไปในไฟล์ executable ด้วย — ข้อมูลนี้
คือแผนที่ที่บอกว่า "instruction ที่ address นี้ตรงกับบรรทัดไหนของไฟล์ .c ต้นฉบับ" และ
"ตัวแปรชื่อนี้อยู่ที่ offset ไหนของ stack frame"

มาดูความแตกต่างแบบจับต้องได้ — คอมไพล์โปรแกรมเดียวกันสองแบบ:

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 -O0 nullptr_bug.c -o nullptr_bug_nodebug   # ไม่มี -g
gcc -Wall -Wextra -Wpedantic -std=c17 -g -O0 nullptr_bug.c -o nullptr_bug        # มี -g
```

เปิด debug เวอร์ชันที่**ไม่มี** `-g`:

```
$ gdb -q ./nullptr_bug_nodebug
Reading symbols from ./nullptr_bug_nodebug...
(No debugging symbols found in ./nullptr_bug_nodebug)
(gdb) break print_person
Breakpoint 1 at 0x1238
(gdb) run
Starting program: /home/user/nullptr_bug_nodebug
[Thread debugging using libthread_db enabled]
Using host libthread_db library "/lib/x86_64-linux-gnu/libthread_db.so.1".

Breakpoint 1, 0x0000555555555238 in print_person ()
(gdb) backtrace
#0  0x0000555555555238 in print_person ()
#1  0x0000555555555332 in main ()
```

สังเกตว่า GDB แจ้งเตือนทันทีว่า **"No debugging symbols found"** และเมื่อหยุดที่
breakpoint ก็บอกได้แค่ **ตำแหน่ง memory address ดิบๆ** (`0x0000555555555238`) ไม่รู้ว่า
`print_person` รับ parameter อะไร ไม่รู้ว่าอยู่บรรทัดไหนของไฟล์ .c เลย — ใช้งานแทบไม่ได้จริง

เทียบกับเวอร์ชันที่ **มี** `-g`:

```
$ gdb -q ./nullptr_bug
Reading symbols from ./nullptr_bug...
(gdb) break print_person
Breakpoint 1 at 0x1240: file nullptr_bug.c, line 30.
(gdb) run
Starting program: /home/user/nullptr_bug

Breakpoint 1, print_person (p=0x7fffffffc934) at nullptr_bug.c:30
30	    printf("Name: %s, Age: %d\n", p->name, p->age);
```

ตอนนี้ GDB บอกได้ครบถ้วน: ไฟล์ต้นฉบับคือ `nullptr_bug.c` บรรทัด 30 พารามิเตอร์ `p`
มีค่าเป็น `0x7fffffffc934` และแสดง source code บรรทัดนั้นให้ดูตรงๆ เลย — **นี่คือเหตุผล
ที่กฎทองของหลักสูตรนี้ (ตั้งแต่ Part 1) คือต้องใส่ `-g` เสมอระหว่างพัฒนา**

> **ข้อควรรู้สำหรับงานจริง**: โปรแกรมที่ build สำหรับ production (release build) มักไม่ใส่
> `-g` เพื่อลดขนาดไฟล์ executable แต่ในทางปฏิบัติ ทีมวิศวกรรมมืออาชีพมักใช้เทคนิค **แยก
> debug symbol ออกเป็นไฟล์ต่างหาก** (`objcopy --only-keep-debug`) เก็บไว้สำหรับ debug
> ในอนาคตได้โดยไม่ทำให้ไฟล์ executable ที่ deploy จริงมีขนาดใหญ่ขึ้น

---

## 37.3 คำสั่งพื้นฐานของ GDB (Step 291)

### ตารางสรุปคำสั่งที่ใช้บ่อยที่สุด

| คำสั่ง | ย่อ | ความหมาย |
|---|---|---|
| `break <ที่ตำแหน่ง>` | `b` | ตั้ง breakpoint หยุดโปรแกรมที่ฟังก์ชันหรือบรรทัดที่ระบุ |
| `break <ตำแหน่ง> if <เงื่อนไข>` | | ตั้ง breakpoint ที่หยุดเฉพาะเมื่อเงื่อนไขเป็นจริง (conditional breakpoint) |
| `run` | `r` | เริ่มรันโปรแกรมจากต้นจนกว่าจะเจอ breakpoint หรือจบ |
| `next` | `n` | รันบรรทัดถัดไป 1 บรรทัด โดย **ไม่** เข้าไปข้างในฟังก์ชันที่ถูกเรียก |
| `step` | `s` | รันบรรทัดถัดไป 1 บรรทัด โดย **เข้าไปข้างใน** ฟังก์ชันที่ถูกเรียกด้วย |
| `continue` | `c` | รันโปรแกรมต่อจนกว่าจะเจอ breakpoint ถัดไปหรือจบ |
| `finish` | | รันจนกว่าฟังก์ชันปัจจุบันจะ return แล้วแสดงค่าที่ return ออกมา |
| `print <expr>` | `p` | แสดงค่าของตัวแปรหรือนิพจน์ใดๆ |
| `backtrace` | `bt` | แสดง call stack ทั้งหมด ณ จุดที่หยุดอยู่ |
| `watch <ตัวแปร>` | | ตั้ง watchpoint หยุดโปรแกรมทันทีที่ค่าตัวแปรนี้เปลี่ยนแปลง |
| `info locals` | | แสดงตัวแปร local ทั้งหมดในฟังก์ชันปัจจุบัน |
| `info args` | | แสดง parameter ทั้งหมดของฟังก์ชันปัจจุบัน |
| `list` | `l` | แสดง source code รอบๆ บรรทัดปัจจุบัน |
| `quit` | `q` | ออกจาก GDB |

### ตัวอย่างจริง: ทดลองคำสั่งพื้นฐานทั้งหมด

ใช้โปรแกรม `nullptr_bug.c` เดียวกัน (จะดูโค้ดเต็มใน 37.4) ทดลองตั้ง breakpoint ที่
`main`, ไล่ทีละบรรทัดด้วย `next`, แล้ว `step` เข้าไปดูข้างในฟังก์ชัน `find_person`:

```bash
gdb -q ./nullptr_bug
```

```
Reading symbols from ./nullptr_bug...
(gdb) break main
Breakpoint 1 at 0x1271: file nullptr_bug.c, line 33.
(gdb) run
Starting program: /home/user/nullptr_bug

Breakpoint 1, main () at nullptr_bug.c:33
33	int main(void) {
(gdb) next
34	    person_t people[3] = {
(gdb) next
40	    printf("ค้นหา Bob:\n");
(gdb) next
41	    print_person(find_person(people, 3, "Bob"));
(gdb) step
find_person (people=0x7fffffffc910, count=3, name=0x555555556030 "Bob")
    at nullptr_bug.c:18
18	    for (int i = 0; i < count; i++) {
(gdb) backtrace
#0  find_person (people=0x7fffffffc910, count=3, name=0x555555556030 "Bob")
    at nullptr_bug.c:18
#1  0x000055555555532a in main () at nullptr_bug.c:41
(gdb) info args
people = 0x7fffffffc910
count = 3
name = 0x555555556030 "Bob"
(gdb) print name
$1 = 0x555555556030 "Bob"
(gdb) finish
Run till exit from #0  find_person (people=0x7fffffffc910, count=3, 
    name=0x555555556030 "Bob") at nullptr_bug.c:18
0x000055555555532a in main () at nullptr_bug.c:41
41	    print_person(find_person(people, 3, "Bob"));
Value returned is $2 = (person_t *) 0x7fffffffc934
(gdb) quit
```

สังเกตความแตกต่างที่สำคัญระหว่าง `next` กับ `step`: ตอนอยู่ที่บรรทัด 40 (`printf(...)`)
เราใช้ `next` เพื่อ **ข้าม** การเข้าไปดูข้างในของ `printf` (ฟังก์ชันของ library ที่เรา
ไม่สนใจรายละเอียด) แต่พอถึงบรรทัด 41 ที่มีการเรียก `find_person()` ซึ่งเป็นฟังก์ชันที่
**เราเขียนเอง** และต้องการไล่ดูข้างใน เราใช้ `step` แทน เพื่อกระโดดเข้าไปในฟังก์ชันนั้น

`finish` มีประโยชน์มากเมื่ออยู่ข้างในฟังก์ชันแล้วอยากรู้ว่าค่าที่ return ออกไปคืออะไร
โดยไม่ต้องกด `next` ไล่ทีละบรรทัดจนจบฟังก์ชันเอง — จากตัวอย่างข้างต้นเห็นชัดว่า
`find_person("Bob")` คืนค่า pointer `0x7fffffffc934` กลับไป

> **เคล็ดลับที่ควรรู้**: การกด Enter เฉยๆ (ไม่พิมพ์คำสั่งใหม่) ใน GDB จะเป็นการ **สั่งซ้ำ
> คำสั่งล่าสุดที่พิมพ์ไป** อัตโนมัติ เช่น ถ้าเพิ่งพิมพ์ `next` แล้วกด Enter เปล่าๆ ไปเรื่อยๆ
> โปรแกรมจะไล่บรรทัดทีละบรรทัดโดยไม่ต้องพิมพ์ `next` ซ้ำทุกครั้ง ช่วยประหยัดเวลาการพิมพ์
> ได้มากเวลาต้องไล่โค้ดยาวๆ

---

## 37.4 Debug Segmentation Fault จริงด้วย Backtrace หา Root Cause (Step 292)

มาดูโค้ดเต็มของบั๊กตัวแรก — **null pointer dereference** ซึ่งเป็นสาเหตุของ segmentation
fault ที่พบบ่อยที่สุดในโปรแกรม C:

```c
/* ============================================================
 * ชื่อไฟล์:     nullptr_bug.c
 * คำอธิบาย:     โปรแกรมสาธิตบั๊ก null pointer dereference สำหรับฝึก
 *              การ debug ด้วย GDB — find_person() คืนค่า NULL เมื่อ
 *              หาไม่เจอ แต่ print_person() ไม่ได้ตรวจสอบก่อนใช้งาน
 * ============================================================ */
#include <stdio.h>
#include <string.h>

typedef struct {
    char name[32];
    int  age;
} person_t;

/* ค้นหาคนชื่อ name ใน array people คืนค่า pointer ไปยัง record นั้น
 * หรือคืนค่า NULL ถ้าหาไม่เจอ (พฤติกรรมปกติของฟังก์ชันค้นหาทั่วไป) */
person_t *find_person(person_t *people, int count, const char *name) {
    for (int i = 0; i < count; i++) {
        if (strcmp(people[i].name, name) == 0) {
            return &people[i];
        }
    }
    return NULL;
}

/* บั๊ก: ฟังก์ชันนี้ไม่ตรวจสอบว่า p เป็น NULL หรือไม่ก่อนใช้งาน
 * ถ้า caller ส่ง NULL เข้ามา (เช่น ผลลัพธ์จาก find_person ที่หาไม่เจอ)
 * บรรทัด p->name จะพยายามอ่าน memory ที่ address ใกล้ 0 -> SIGSEGV */
void print_person(person_t *p) {
    printf("Name: %s, Age: %d\n", p->name, p->age);
}

int main(void) {
    person_t people[3] = {
        {"Alice", 30},
        {"Bob",   25},
        {"Carol", 35}
    };

    printf("ค้นหา Bob:\n");
    print_person(find_person(people, 3, "Bob"));

    printf("ค้นหา Dave (ไม่มีในลิสต์):\n");
    print_person(find_person(people, 3, "Dave"));   /* จุดที่จะ crash */

    printf("บรรทัดนี้จะไม่มีวันถูกพิมพ์ เพราะโปรแกรม crash ไปก่อนแล้ว\n");
    return 0;
}
```

รันตรงๆ ก่อนเพื่อยืนยันว่า crash จริง:

```bash
$ gcc -Wall -Wextra -Wpedantic -std=c17 -g -O0 nullptr_bug.c -o nullptr_bug
$ ./nullptr_bug
ค้นหา Bob:
Name: Bob, Age: 25
ค้นหา Dave (ไม่มีในลิสต์):
Segmentation fault (core dumped)
$ echo $?
139
```

`exit code 139` ยืนยันว่าโปรแกรมถูกฆ่าด้วยสัญญาณ (`128 + สัญญาณ`) — `139 - 128 = 11`
ซึ่งตรงกับหมายเลขของ `SIGSEGV` พอดี (ทบทวนความสัมพันธ์นี้จาก Part 28)

### Debug ด้วย GDB: ไล่หา Root Cause ทีละขั้นตอน

```bash
gdb -q ./nullptr_bug
```

```
Reading symbols from ./nullptr_bug...
(gdb) break print_person
Breakpoint 1 at 0x1240: file nullptr_bug.c, line 30.
(gdb) run
Starting program: /home/user/nullptr_bug

Breakpoint 1, print_person (p=0x7fffffffc934) at nullptr_bug.c:30
30	    printf("Name: %s, Age: %d\n", p->name, p->age);
(gdb) print p
$1 = (person_t *) 0x7fffffffc934
(gdb) backtrace
#0  print_person (p=0x7fffffffc934) at nullptr_bug.c:30
#1  0x0000555555555332 in main () at nullptr_bug.c:41
(gdb) next
31	}
(gdb) print *p
$2 = {name = "Bob", '\000' <repeats 28 times>, age = 25}
(gdb) continue
Continuing.

Breakpoint 1, print_person (p=0x0) at nullptr_bug.c:30
30	    printf("Name: %s, Age: %d\n", p->name, p->age);
(gdb) print p
$3 = (person_t *) 0x0
(gdb) next

Program received signal SIGSEGV, Segmentation fault.
0x0000555555555244 in print_person (p=0x0) at nullptr_bug.c:30
30	    printf("Name: %s, Age: %d\n", p->name, p->age);
(gdb) backtrace
#0  0x0000555555555244 in print_person (p=0x0) at nullptr_bug.c:30
#1  0x0000555555555361 in main () at nullptr_bug.c:44
(gdb) quit
```

นี่คือขั้นตอนการหา root cause แบบเป็นระบบ:

1. ตั้ง breakpoint ที่ `print_person()` แล้ว `run` — เจอ breakpoint ครั้งแรกตอนเรียก
   ด้วย "Bob" ซึ่ง `p` มีค่าเป็น address จริง (`0x7fffffffc934`) `print *p` ยืนยันว่า
   ข้อมูลถูกต้อง (`name = "Bob", age = 25`)
2. `continue` ไปยังการเรียกครั้งที่สอง (ด้วย "Dave") — คราวนี้ `print p` แสดงชัดเจนว่า
   **`p` มีค่าเป็น `0x0` (NULL)** ตั้งแต่ก่อนเข้าฟังก์ชันด้วยซ้ำ
3. กด `next` เพื่อรันบรรทัดที่มีปัญหา (`printf(...)` ที่เข้าถึง `p->name`) — ตรงนี้เอง
   ที่ **`SIGSEGV`** เกิดขึ้นจริง gdb แสดงให้เห็นชัดเจนว่าเกิดที่บรรทัด 30
4. `backtrace` ยืนยัน call chain ทั้งหมด: `print_person` (บรรทัด 30) ถูกเรียกจาก `main`
   บรรทัด 44 — ตรงกับบรรทัดที่เรียก `print_person(find_person(people, 3, "Dave"))`
   ใน source code จริง

**Root cause ชัดเจน**: `find_person()` คืนค่า `NULL` เมื่อหา "Dave" ไม่เจอ (ถูกต้องตาม
design) แต่ `print_person()` ไม่ได้ตรวจสอบ `p == NULL` ก่อนเข้าถึง `p->name` เลย —
วิธีแก้คือเพิ่มการตรวจสอบ:

```c
void print_person(person_t *p) {
    if (p == NULL) {
        printf("ไม่พบข้อมูลบุคคลนี้\n");
        return;
    }
    printf("Name: %s, Age: %d\n", p->name, p->age);
}
```

---

## 37.5 Debug Off-by-One Bug ทีละขั้นตอนด้วย GDB (Step 293)

บั๊กประเภทที่สองที่พบบ่อยไม่แพ้กันคือ **off-by-one** — เงื่อนไขของลูปผิดไปแค่ 1 หน่วย
แต่ทำให้เข้าถึง array นอกขอบเขต ต่างจาก null pointer dereference ตรงที่ off-by-one
**มักไม่ crash ทันที** (อ่านค่าขยะจาก memory ที่ยังคง valid อยู่แต่ไม่ใช่ของเรา) ทำให้
ตรวจจับยากกว่ามาก เพราะโปรแกรม "ดูเหมือนทำงานได้" แต่ผลลัพธ์ผิด

```c
/* ============================================================
 * ชื่อไฟล์:     offbyone_bug.c
 * คำอธิบาย:     โปรแกรมสาธิตบั๊ก off-by-one สำหรับฝึกการ debug ด้วย GDB
 *              sum_array() ใช้เงื่อนไข i <= size แทนที่จะเป็น i < size
 *              ทำให้อ่านค่านอกขอบเขต array ไป 1 ตำแหน่ง
 * ============================================================ */
#include <stdio.h>

/* บั๊ก: เงื่อนไขวนลูปควรเป็น i < size ไม่ใช่ i <= size
 * เมื่อ size = 5 ลูปนี้จะพยายามเข้าถึง arr[5] ซึ่งอยู่นอกขอบเขตของ
 * array ที่มีแค่ index 0-4 (อ่านค่าขยะจาก stack memory ที่อยู่ถัดไป) */
int sum_array(int *arr, int size) {
    int sum = 0;
    for (int i = 0; i <= size; i++) {
        sum += arr[i];
    }
    return sum;
}

int main(void) {
    int numbers[5] = {10, 20, 30, 40, 50};

    int expected = 10 + 20 + 30 + 40 + 50;   /* ค่าที่ถูกต้องคือ 150 */
    int total = sum_array(numbers, 5);

    printf("ผลรวมที่คำนวณได้: %d\n", total);
    printf("ผลรวมที่ถูกต้อง:  %d\n", expected);

    if (total != expected) {
        printf("*** พบความผิดปกติ: ผลรวมไม่ตรงกับที่คาดหวัง! ***\n");
    }

    return 0;
}
```

รันตรงๆ ก่อน:

```bash
$ gcc -Wall -Wextra -Wpedantic -std=c17 -g -O0 offbyone_bug.c -o offbyone_bug
$ ./offbyone_bug
ผลรวมที่คำนวณได้: 32917
ผลรวมที่ถูกต้อง:  150
*** พบความผิดปกติ: ผลรวมไม่ตรงกับที่คาดหวัง! ***
```

โปรแกรม **ไม่ crash** แต่ให้ผลลัพธ์ผิด (`32917` แทนที่จะเป็น `150`) นี่คือลักษณะเฉพาะ
ของ Undefined Behavior จากการอ่าน memory นอกขอบเขต — ค่าที่ได้ไม่ใช่ค่าคงที่ตายตัวเสมอไป
(ขึ้นกับ layout ของ stack ที่อาจเปลี่ยนแปลงเล็กน้อยระหว่างการรันแต่ละครั้ง เพราะมีปัจจัย
เช่น ASLR หรือ environment variable เข้ามาเกี่ยวข้อง)

### ใช้ `watch` ตรวจจับจุดที่ตัวแปรเปลี่ยนค่า

แทนที่จะเดาว่าปัญหาอยู่ตรงไหน เราจะใช้ **watchpoint** เฝ้าดูตัวแปร `i` ทุกครั้งที่มันมี
การเปลี่ยนค่า เพื่อดูว่าลูปวิ่งไปถึงค่าไหนบ้างจริงๆ:

```bash
gdb -q ./offbyone_bug
```

```
Reading symbols from ./offbyone_bug...
(gdb) break sum_array
Breakpoint 1 at 0x1198: file offbyone_bug.c, line 13.
(gdb) run
Starting program: /home/user/offbyone_bug

Breakpoint 1, sum_array (arr=0x7fffffffc970, size=5) at offbyone_bug.c:13
13	    int sum = 0;
(gdb) next
14	    for (int i = 0; i <= size; i++) {
(gdb) watch i
Hardware watchpoint 2: i
(gdb) continue
Continuing.

Hardware watchpoint 2: i

Old value = 0
New value = 1
0x00005555555551c5 in sum_array (arr=0x7fffffffc970, size=5)
    at offbyone_bug.c:14
14	    for (int i = 0; i <= size; i++) {
(gdb) continue
Continuing.

Hardware watchpoint 2: i

Old value = 1
New value = 2
...
(gdb) continue
Continuing.

Hardware watchpoint 2: i

Old value = 4
New value = 5
0x00005555555551c5 in sum_array (arr=0x7fffffffc970, size=5)
    at offbyone_bug.c:14
14	    for (int i = 0; i <= size; i++) {
(gdb) info locals
i = 5
sum = 150
(gdb) print arr[5]
$1 = 32767
(gdb) print arr[4]
$2 = 50
(gdb) print size
$3 = 5
(gdb) quit
```

(ผลลัพธ์ของ `continue` รอบที่ 3 และ 4 ถูกย่อไว้ในเอกสารนี้เพื่อความกระชับ — มีรูปแบบ
เดียวกันทุกประการ คือ `Old value` เพิ่มขึ้นทีละ 1 จนถึง `New value = 5`)

**นี่คือหลักฐานที่ชัดเจนที่สุด**: `info locals` แสดงว่า `i = 5` และ `size = 5` —
เงื่อนไขลูป `i <= size` จึงยังเป็นจริงอยู่ (`5 <= 5`) ทำให้ลูปวิ่งเข้าไปประมวลผล
`arr[5]` ซึ่ง **อยู่นอกขอบเขตของ array ที่มีแค่ index 0-4** `print arr[5]` ยืนยันว่า
มันคือค่าขยะ (`32767`) ที่ไม่เกี่ยวข้องกับข้อมูลจริงเลย ในขณะที่ `print arr[4]`
(องค์ประกอบตัวสุดท้ายที่ถูกต้อง) ให้ค่า `50` ตามที่คาดหวัง — `sum = 150` ตรงนี้ยังถูกต้อง
อยู่ (เป็นผลรวมของ index 0-4) แต่หลังจากนี้ลูปจะบวก `arr[5]` (ค่าขยะ) เข้าไปอีกหนึ่งรอบ
ก่อนที่ `i` จะเพิ่มเป็น 6 และเงื่อนไข `6 <= 5` เป็นเท็จ ทำให้ลูปหยุด

**Root cause ชัดเจน**: เงื่อนไขลูปควรเป็น `i < size` ไม่ใช่ `i <= size`

```c
int sum_array(int *arr, int size) {
    int sum = 0;
    for (int i = 0; i < size; i++) {   /* แก้จาก <= เป็น < */
        sum += arr[i];
    }
    return sum;
}
```

---

## 37.6 Watchpoint แบบเจาะลึก: watch, rwatch, awatch (Step 294)

GDB มี watchpoint 3 ประเภทที่ควรรู้จัก:

| คำสั่ง | หยุดเมื่อ |
|---|---|
| `watch <expr>` | ค่าของ `expr` **ถูกเขียน** (write) เท่านั้น (ที่ใช้ใน 37.5) |
| `rwatch <expr>` | ค่าของ `expr` **ถูกอ่าน** (read) เท่านั้น |
| `awatch <expr>` | ค่าของ `expr` ถูก **อ่านหรือเขียน** (access) อย่างใดอย่างหนึ่ง |

`rwatch` มีประโยชน์มากเมื่อสงสัยว่ามีการอ่านตัวแปรตัวหนึ่งจากที่ไม่คาดคิด (เช่น สงสัยว่า
ฟังก์ชันไหนแอบมาอ่านค่านี้บ่อยผิดปกติ) ทดลองใช้กับ `sum` ในโปรแกรม `offbyone_bug.c`:

```
(gdb) break sum_array
(gdb) run
(gdb) next
14	    for (int i = 0; i <= size; i++) {
(gdb) rwatch sum
Hardware read watchpoint 2: sum
(gdb) continue
Continuing.

Hardware read watchpoint 2: sum

Value = 32917
sum_array (arr=0x7fffffffc970, size=5) at offbyone_bug.c:18
18	}
(gdb) continue
Continuing.

Watchpoint 2 deleted because the program has left the block in
which its expression is valid.
0x000055555555522f in main () at offbyone_bug.c:28
28	    int total = sum_array(numbers, 5);
```

สังเกตข้อความสุดท้าย **"Watchpoint 2 deleted because the program has left the block
in which its expression is valid"** — นี่คือพฤติกรรมสำคัญที่ต้องรู้: watchpoint ที่ตั้ง
กับตัวแปร **local** (เช่น `sum` ที่ประกาศอยู่ข้างใน `sum_array()`) จะถูกลบทิ้งอัตโนมัติ
ทันทีที่โปรแกรมออกจาก scope ของฟังก์ชันนั้น (เพราะตัวแปรนั้นไม่มีตัวตนอยู่บน stack อีก
ต่อไปแล้ว) ต่างจาก watchpoint บนตัวแปร **global** ที่จะคงอยู่ตลอดการรันโปรแกรม

### Hardware Watchpoint คืออะไร

สังเกตว่า GDB แสดงข้อความ **"Hardware watchpoint"** ไม่ใช่แค่ "watchpoint" เฉยๆ —
บน CPU สถาปัตยกรรมสมัยใหม่ (x86-64, ARM64) มี **debug register ระดับฮาร์ดแวร์** ที่
ออกแบบมาสำหรับตรวจจับการเข้าถึง memory address เฉพาะจุดโดยตรง ทำให้ watchpoint ทำงาน
**เร็วมาก** (CPU ตรวจสอบเองในระดับฮาร์ดแวร์ ไม่ต้องพึ่งพา GDB คอย single-step ทีละ
instruction) ถ้าจำนวน watchpoint เกินกว่าที่ debug register ของ CPU รองรับ (ปกติมีแค่
2-4 ช่องขึ้นกับสถาปัตยกรรม) GDB จะ fallback ไปใช้ **software watchpoint** ซึ่งทำงาน
ช้ากว่ามากเพราะต้อง single-step ทุก instruction แล้วตรวจค่าเองตลอดเวลา

---

## 37.7 Core Dump คืออะไรและวิธีเปิดดูด้วย GDB (Step 295)

จนถึงตอนนี้เรา debug โปรแกรมแบบ **"สด"** (live) คือรันผ่าน GDB ตั้งแต่ต้น แต่ในโลกจริง
บั๊กมักเกิดขึ้นตอนที่โปรแกรมรันบน production server โดยไม่มี GDB attach อยู่เลย เมื่อ
โปรแกรม crash ไปแล้ว เราจะสูญเสียโอกาส debug แบบสดไปตลอดกาล — **core dump** คือทางออก
สำหรับสถานการณ์นี้

**Core dump** คือไฟล์ที่ระบบปฏิบัติการสร้างขึ้นโดยอัตโนมัติเมื่อโปรแกรม crash ด้วย
สัญญาณบางประเภท (เช่น `SIGSEGV`, `SIGABRT`) โดยไฟล์นี้จะบันทึก **สถานะทั้งหมดของ
โปรเซส ณ วินาทีที่ crash** ไว้ ทั้ง memory, register, call stack — ทำให้เราสามารถเปิด
วิเคราะห์ย้อนหลังด้วย GDB ได้ทีหลัง แม้โปรเซสจะตายไปแล้วก็ตาม

### เปิดใช้งาน Core Dump

โดย default หลาย Linux distribution จะปิดการสร้าง core dump ไว้ (`ulimit -c` = 0)
เพื่อประหยัดพื้นที่ดิสก์ ต้องเปิดใช้งานก่อน:

```bash
$ ulimit -c
0
$ ulimit -c unlimited
$ ulimit -c
unlimited
```

จากนั้นรันโปรแกรมที่มีบั๊กให้ crash:

```bash
$ ./nullptr_bug
ค้นหา Bob:
Name: Bob, Age: 25
ค้นหา Dave (ไม่มีในลิสต์):
Segmentation fault (core dumped)
```

ข้อความ **"(core dumped)"** ยืนยันว่ามีการสร้างไฟล์ core dump จริง ตรวจสอบไฟล์:

```bash
$ ls -la core*
-rw------- 1 root root 462848 Sep 26 06:44 core
```

(ชื่อไฟล์ core dump ขึ้นกับการตั้งค่า `/proc/sys/kernel/core_pattern` ของระบบ — บาง
ระบบตั้งชื่อเป็น `core.<PID>` แทน ตรวจสอบ pattern ปัจจุบันด้วย `cat
/proc/sys/kernel/core_pattern`)

### เปิด Core Dump ด้วย GDB

```bash
gdb -q ./nullptr_bug core
```

```
Reading symbols from ./nullptr_bug...
[New LWP 28159]
[Thread debugging using libthread_db enabled]
Using host libthread_db library "/lib/x86_64-linux-gnu/libthread_db.so.1".
Core was generated by `./nullptr_bug'.
Program terminated with signal SIGSEGV, Segmentation fault.
#0  0x0000563811af2244 in print_person (p=0x0) at nullptr_bug.c:30
30	    printf("Name: %s, Age: %d\n", p->name, p->age);
(gdb) backtrace
#0  0x0000563811af2244 in print_person (p=0x0) at nullptr_bug.c:30
#1  0x0000563811af2361 in main () at nullptr_bug.c:44
(gdb) print *p
Cannot access memory at address 0x0
(gdb) frame 1
#1  0x0000563811af2361 in main () at nullptr_bug.c:44
44	    print_person(find_person(people, 3, "Dave"));   /* จุดที่จะ crash */
(gdb) list
39	
40	    printf("ค้นหา Bob:\n");
41	    print_person(find_person(people, 3, "Bob"));
42	
43	    printf("ค้นหา Dave (ไม่มีในลิสต์):\n");
44	    print_person(find_person(people, 3, "Dave"));   /* จุดที่จะ crash */
45	
46	    printf("บรรทัดนี้จะไม่มีวันถูกพิมพ์ เพราะโปรแกรม crash ไปก่อนแล้ว\n");
47	    return 0;
48	}
(gdb) quit
```

GDB บอกทันทีว่า **"Program terminated with signal SIGSEGV"** และแสดง `backtrace` ได้
เหมือนกับตอน debug สดทุกประการ — สังเกตว่า `print *p` ให้ error **"Cannot access
memory at address 0x0"** เพราะ `p` เป็น NULL อยู่แล้ว (การพยายาม dereference มันจึง
ล้มเหลวเหมือนตอนโปรแกรมจริง crash) `frame 1` ใช้เปลี่ยนไปดู stack frame ของ `main`
(frame ที่เรียก `print_person` มา) และ `list` แสดง source code รอบๆ บรรทัดนั้นให้ดู
บริบทเพิ่มเติม

จุดสำคัญที่สุดของ core dump คือ **เราไม่จำเป็นต้อง reproduce บั๊กใหม่เลย** ถ้า production
server ส่งไฟล์ core dump กลับมาให้ทีม dev วิเคราะห์ สามารถหา root cause ได้ทันทีโดยไม่
ต้องพยายามทำให้บั๊กเกิดซ้ำในเครื่องตัวเอง (ซึ่งบางบั๊กอาจ reproduce ยากมาก เช่น บั๊กที่
เกิดจาก timing หรือข้อมูล input เฉพาะของลูกค้ารายนั้น)

---

## 37.8 GDB TUI Mode เบื้องต้น (Step 296)

จนถึงตอนนี้เราใช้ GDB ในโหมด command-line ธรรมดา ซึ่งต้องพิมพ์ `list` ทุกครั้งที่อยาก
ดู source code รอบๆ บรรทัดปัจจุบัน — **TUI mode** (Text User Interface) แก้ปัญหานี้
โดยแสดง source code ควบคู่กับ command prompt ในหน้าจอเดียวกันแบบ real-time

เปิดใช้งาน TUI mode ได้ 2 วิธี:

1. เริ่ม gdb ด้วย flag `-tui`: `gdb -tui ./nullptr_bug`
2. หรือกด `Ctrl+X` แล้วตามด้วย `A` ระหว่างอยู่ใน GDB ปกติ (สลับเข้า/ออก TUI mode ได้
   ตลอดเวลา)

ตัวอย่างหน้าจอจริงที่ได้ (capture จากการรันจริง หลังตั้ง `break print_person` และ `run`):

```
┌─nullptr_bug.c────────────────────────────────────────────────────────────────────────────────────┐
│       23     return NULL;                                                                        │
│       24 }                                                                                       │
│       25                                                                                         │
│       26 /* บั๊ก: ฟังก์ชันนี้ไม่ตรวจสอบว่า p เป็น NULL หรือไม่ก่อนใช้งาน                                       │
│       27  * ถ้า caller ส่ง NULL เข้ามา (เช่น ผลลัพธ์จาก find_person ที่หาไม่เจอ)                          │
│       28  * บรรทัด p->name จะพยายามอ่าน memory ที่ address ใกล้ 0 -> SIGSEGV */                       │
│       29 void print_person(person_t *p) {                                                        │
│B+>    30     printf("Name: %s, Age: %d\n", p->name, p->age);                                     │
│       31 }                                                                                       │
│       32                                                                                         │
│       33 int main(void) {                                                                        │
│       34     person_t people[3] = {                                                              │
│       35         {"Alice", 30},                                                                  │
│       36         {"Bob",   25},                                                                  │
│       37         {"Carol", 35}                                                                   │
│       38     };                                                                                  │
│       39                                                                                         │
└──────────────────────────────────────────────────────────────────────────────────────────────────┘
multi-thre Thread 0x7ffff7faf7 (src) In: print_person                      L30   PC: 0x555555555240

[Thread debugging using libthread_db enabled]
Using host libthread_db library "/lib/x86_64-linux-gnu/libthread_db.so.1".
ค้นหา Bob:
Breakpoint 1, print_person (p=0x7fffffffbf74) at nullptr_bug.c:30
(gdb)
```

สังเกตสัญลักษณ์ `B+>` ที่หน้าบรรทัด 30 — `B` หมายถึงมี breakpoint ตั้งอยู่ที่บรรทัดนี้
ส่วน `>` คือ **ตำแหน่งปัจจุบัน** ที่โปรแกรมหยุดอยู่ (program counter) แถบด้านล่างของ
กรอบ source แสดงข้อมูลสถานะ: ชื่อ thread, ฟังก์ชันปัจจุบัน (`print_person`), เลขบรรทัด
(`L30`), และ address ของ instruction (`PC: 0x555555555240`)

เมื่อพิมพ์คำสั่ง `next` ในโหมด TUI หน้าจอ source จะเลื่อนตำแหน่ง `>` ตามไปด้วยทันที
โดยไม่ต้องพิมพ์ `list` ซ้ำเลย:

```
┌─nullptr_bug.c────────────────────────────────────────────────────────────────────────────────────┐
│       26 /* บั๊ก: ฟังก์ชันนี้ไม่ตรวจสอบว่า p เป็น NULL หรือไม่ก่อนใช้งาน                                       │
│       27  * ถ้า caller ส่ง NULL เข้ามา (เช่น ผลลัพธ์จาก find_person ที่หาไม่เจอ)                          │
│       28  * บรรทัด p->name จะพยายามอ่าน memory ที่ address ใกล้ 0 -> SIGSEGV */                       │
│       29 void print_person(person_t *p) {                                                        │
│B+     30     printf("Name: %s, Age: %d\n", p->name, p->age);                                     │
│  >    31 }                                                                                       │
│       32                                                                                         │
│       33 int main(void) {                                                                        │
...
multi-thre Thread 0x7ffff7faf7 (src) In: print_person                      L31   PC: 0x555555555262

Name: Bob, Age: 25
(gdb) next
```

ตำแหน่ง `>` ขยับจากบรรทัด 30 ไปบรรทัด 31 พอดี และมีข้อความ "Name: Bob, Age: 25" ปรากฏ
ในส่วน command window ด้านล่าง (ผลลัพธ์จริงจาก `printf()` ที่เพิ่งรันผ่านไป)

### คำสั่งที่ใช้บ่อยใน TUI Mode

| คีย์ลัด/คำสั่ง | ความหมาย |
|---|---|
| `Ctrl+X` แล้วตาม `A` | สลับเข้า/ออก TUI mode |
| `Ctrl+X` แล้วตาม `2` | สลับ layout แสดง 2 หน้าต่าง (เช่น source + assembly) |
| `Ctrl+L` | วาดหน้าจอใหม่ (เผื่อจอเพี้ยนจาก output อื่นที่แทรกเข้ามา) |
| `layout src` | แสดงเฉพาะหน้าต่าง source code |
| `layout asm` | แสดงหน้าต่าง assembly instruction |
| `layout regs` | แสดงหน้าต่าง CPU register เพิ่มเติม |
| ลูกศรขึ้น/ลง (เมื่อ focus อยู่ที่ source window) | เลื่อนดู source code ขึ้น/ลง |

TUI mode มีประโยชน์มากตอน `step`/`next` ไล่โค้ดยาวๆ เพราะเห็นบริบทของโค้ดรอบข้าง
ตลอดเวลาโดยไม่ต้องพิมพ์ `list` ซ้ำ แต่สำหรับงานที่ต้องพิมพ์คำสั่งซับซ้อนเยอะๆ (เช่น
ตรวจสอบ struct ใหญ่ๆ หลายชั้น) โหมดปกติที่มีพื้นที่ output กว้างกว่าอาจอ่านง่ายกว่า
เลือกใช้ตามความเหมาะสมของงาน

---

## ข้อผิดพลาดที่พบบ่อย (Common Pitfalls)

1. **ลืมคอมไพล์ด้วย `-g` แล้วสงสัยว่าทำไม GDB บอกได้แค่ address ดิบๆ** — อาการที่พบคือ
   `(No debugging symbols found in ...)` และ backtrace แสดงแต่ `0x0000...` ไม่มีชื่อ
   ฟังก์ชันหรือเลขบรรทัดเลย ตามที่สาธิตจริงใน 37.2 ทางแก้คือคอมไพล์ใหม่ด้วย `-g` เสมอ
   ระหว่างพัฒนา

2. **ใช้ `-O2` หรือ optimization level สูงระหว่าง debug** — compiler ที่ optimize
   โค้ดอาจจัดเรียงลำดับการทำงานใหม่ ลบตัวแปรที่ไม่จำเป็นทิ้ง หรือ inline ฟังก์ชันเข้า
   ด้วยกัน ทำให้ GDB แสดงผลที่ดู "แปลกๆ" เช่น `next` ข้ามหลายบรรทัดพร้อมกัน หรือตัวแปร
   บางตัว `print` ไม่ได้เพราะถูก optimize ทิ้งไปแล้ว ควรใช้ `-O0` เสมอระหว่าง debug
   (ตามที่ใช้ตลอด Part นี้)

3. **สับสนระหว่าง `next` กับ `step`** — ใช้ `step` ตอนที่ควรใช้ `next` แล้วหลงเข้าไปใน
   ฟังก์ชันของ library (เช่น `printf`, `strcmp`) ที่ไม่มี debug symbol หรือ source code
   ให้ดู ทำให้เห็นแต่ assembly หรือ error แปลกๆ วิธีแก้ถ้าหลงเข้าไปแล้วคือใช้ `finish`
   เพื่อออกมาที่ระดับเดิม

4. **ตั้ง watchpoint บนตัวแปร local แล้วแปลกใจว่าทำไมมันหายไปเอง** — ตามที่สาธิตใน 37.6
   watchpoint บนตัวแปร local จะถูกลบอัตโนมัติเมื่อโปรแกรมออกจาก scope ของฟังก์ชันนั้น
   (ข้อความ "Watchpoint deleted because the program has left the block...") นี่ไม่ใช่
   บั๊กของ GDB แต่เป็นพฤติกรรมที่ถูกต้องตามหลักการของ scope ในภาษา C

5. **ลืมว่า `ulimit -c` เป็นการตั้งค่าเฉพาะ shell session ปัจจุบัน** — ถ้าเปิด terminal
   ใหม่หรือรันผ่านสคริปต์ที่ spawn shell ใหม่ ค่า `ulimit -c unlimited` ที่ตั้งไว้ก่อน
   หน้าจะไม่มีผลอีกต่อไป (กลับไปเป็นค่า default ซึ่งมักเป็น 0) ต้องตั้งใหม่ทุกครั้งที่
   เปิด session ใหม่ หรือกำหนดถาวรผ่านไฟล์ configuration ของระบบถ้าต้องการให้มีผลเสมอ

6. **พยายาม debug segmentation fault ด้วยการอ่านโค้ดเฉยๆ โดยไม่ใช้ `backtrace`** — สำหรับ
   บั๊กที่เกิดจาก call chain ยาวหลายชั้น การไล่อ่านโค้ดด้วยตาเปล่าโดยไม่รู้ว่าฟังก์ชันไหน
   เรียกฟังก์ชันไหนตามลำดับจริงๆ เสียเวลามากกว่าการใช้ `backtrace` ดู call stack จริง
   ในคำสั่งเดียวมาก โดยเฉพาะเมื่อโค้ดมีหลายจุดที่เรียกฟังก์ชันเดียวกัน (ต้องรู้แน่ชัดว่า
   ครั้งนี้ crash มาจากจุดเรียกไหน)

7. **จำ exit code ผิดว่าหมายถึงอะไร** — `echo $?` หลังโปรแกรม crash ด้วยสัญญาณจะได้ค่า
   `128 + เลขสัญญาณ` เช่น `139` คือ `SIGSEGV` (128+11), `136` คือ `SIGFPE` (128+8)
   ไม่ใช่แค่ "error code ทั่วไป" — เลขนี้เป็นเบาะแสสำคัญที่บอกได้ทันทีว่าควรมองหาบั๊ก
   ประเภทไหนก่อนแม้จะยังไม่ได้เปิด GDB เลยด้วยซ้ำ

---

## แบบฝึกหัดท้ายบท

1. เขียนโปรแกรมที่มีบั๊ก "ลืม initialize ตัวแปร" (ทบทวนจาก Part 1) แล้วใช้ GDB `print`
   ดูค่าขยะที่ตัวแปรนั้นมีอยู่จริงก่อนถูกใช้งาน
2. ใช้ **conditional breakpoint** (`break <ตำแหน่ง> if <เงื่อนไข>`) กับโปรแกรมที่มี
   ลูปวนหลายรอบ แต่บั๊กเกิดขึ้นเฉพาะรอบที่ตัวแปรมีค่าตรงตามเงื่อนไขที่กำหนดเท่านั้น
   (ดูแนวทางเฉลยข้อ 2)
3. เขียน recursive function ที่ไม่มี base case ที่ถูกต้อง (ลืมลดค่าพารามิเตอร์ในการ
   เรียกซ้ำ) ทำให้เกิด **stack overflow** แล้วใช้ `backtrace <N>` วิเคราะห์ pattern
   ของ call stack ที่ผิดปกติ (ดูแนวทางเฉลยข้อ 3)
4. ใช้คำสั่ง `x` (examine memory) ของ GDB เพื่อดูค่า raw bytes ของ struct ใน 37.4
   เปรียบเทียบกับผลลัพธ์ที่ได้จาก `print *p` แบบปกติ
5. เขียนไฟล์ `.gdbinit` ในโฟลเดอร์โปรเจกต์ที่ตั้ง breakpoint และคำสั่งเริ่มต้นอัตโนมัติ
   ทุกครั้งที่เปิด GDB กับโปรแกรมนั้น (ค้นคว้าเพิ่มเติมเกี่ยวกับไฟล์ `.gdbinit`)
6. ใช้ TUI mode ผสมกับ `layout regs` เพื่อดูค่า CPU register เปลี่ยนแปลงทีละบรรทัดขณะ
   ใช้ `stepi` (step ทีละ 1 instruction แทนที่จะเป็น 1 บรรทัดของ source code)

### แนวทางเฉลยข้อ 2 (Conditional Breakpoint)

โปรแกรมที่บั๊กเกิดขึ้นเฉพาะตอน `i = 7` เท่านั้น (จำลองสถานการณ์ที่ record ตัวหนึ่งใน
ชุดข้อมูลมีค่าที่ทำให้คำนวณผิดพลาด):

```c
/* ตัวอย่างเฉลย: โปรแกรมที่มีบั๊กเกิดขึ้นเฉพาะตอน i = 7 เท่านั้น (จำลอง
 * สถานการณ์ที่พบบ่อยในงานจริง เช่น record ที่ 7 ในชุดข้อมูลมีค่าที่ทำให้
 * คำนวณผิดพลาด) ถ้าตั้ง breakpoint ธรรมดาที่ process_item() แล้วกด
 * continue ทีละครั้งจนกว่าจะถึง i=7 จะต้องกด 7 ครั้ง (ช้าและน่ารำคาญ
 * ถ้าลูปมีหลักพันหรือหลักหมื่นรอบ) ควรใช้ conditional breakpoint แทน */
#include <stdio.h>

int process_item(int i) {
    int result = 100 / (i - 7);   /* บั๊ก: หารด้วย 0 เมื่อ i == 7 -> SIGFPE */
    return result;
}

int main(void) {
    for (int i = 0; i < 10; i++) {
        printf("กำลังประมวลผล item ที่ %d\n", i);
        int r = process_item(i);
        printf("  ผลลัพธ์ = %d\n", r);
    }
    printf("เสร็จสิ้น\n");
    return 0;
}
```

```bash
$ gcc -Wall -Wextra -Wpedantic -std=c17 -g -O0 cond_break.c -o cond_break
$ ./cond_break
กำลังประมวลผล item ที่ 0
  ผลลัพธ์ = -14
...
กำลังประมวลผล item ที่ 7
Floating point exception (core dumped)
$ echo $?
136
```

`exit code 136` (`128 + 8`) ยืนยันว่าเป็น `SIGFPE` (Arithmetic Exception — ในกรณีนี้
คือหารด้วยศูนย์ ซึ่งภาษา C ไม่ตรวจสอบเองเลย ต้องพึ่งฮาร์ดแวร์ตรวจจับแล้วส่งสัญญาณนี้
กลับมา) ทดสอบด้วย conditional breakpoint แทนที่จะกด `continue` 7 ครั้ง:

```bash
gdb -q ./cond_break
```

```
Reading symbols from ./cond_break...
(gdb) break process_item if i == 7
Breakpoint 1 at 0x1174: file cond_break.c, line 9.
(gdb) run
Starting program: /home/user/cond_break
กำลังประมวลผล item ที่ 0
  ผลลัพธ์ = -14
กำลังประมวลผล item ที่ 1
  ผลลัพธ์ = -16
กำลังประมวลผล item ที่ 2
  ผลลัพธ์ = -20
กำลังประมวลผล item ที่ 3
  ผลลัพธ์ = -25
กำลังประมวลผล item ที่ 4
  ผลลัพธ์ = -33
กำลังประมวลผล item ที่ 5
  ผลลัพธ์ = -50
กำลังประมวลผล item ที่ 6
  ผลลัพธ์ = -100
กำลังประมวลผล item ที่ 7

Breakpoint 1, process_item (i=7) at cond_break.c:9
9	    int result = 100 / (i - 7);   /* บั๊ก: หารด้วย 0 เมื่อ i == 7 -> SIGFPE */
(gdb) print i
$1 = 7
(gdb) backtrace
#0  process_item (i=7) at cond_break.c:9
#1  0x00005555555551c2 in main () at cond_break.c:16
(gdb) continue
Continuing.

Program received signal SIGFPE, Arithmetic exception.
0x0000555555555180 in process_item (i=7) at cond_break.c:9
9	    int result = 100 / (i - 7);   /* บั๊ก: หารด้วย 0 เมื่อ i == 7 -> SIGFPE */
(gdb) quit
```

สังเกตว่า GDB ปล่อยให้โปรแกรมรันผ่าน `i = 0` ถึง `i = 6` ไปเองโดยอัตโนมัติ (ไม่หยุด
เลยแม้แต่ครั้งเดียว) แล้วหยุดตรง `i = 7` **ในการรันครั้งเดียว** — ประหยัดเวลามากเมื่อ
เทียบกับการตั้ง breakpoint ธรรมดาแล้วกด `continue` 7 รอบ ยิ่งถ้าบั๊กเกิดที่รอบที่
1,000 หรือ 100,000 ความแตกต่างจะยิ่งชัดเจนขึ้นไปอีกมาก

### แนวทางเฉลยข้อ 3 (Stack Overflow จาก Recursive Function)

```c
/* ตัวอย่างเฉลย: recursive function ที่ลืมลดค่า n ในการเรียกซ้ำ (ลืมเขียน
 * n - 1 เขียนแค่ n เฉยๆ) ทำให้เงื่อนไขหยุด n == 0 ไม่มีวันเป็นจริงถ้า n
 * เริ่มจากค่าที่ไม่ใช่ 0 เรียกตัวเองไม่รู้จบจนกว่า stack จะเต็ม (stack
 * overflow) แล้วโปรแกรม crash ด้วย SIGSEGV */
#include <stdio.h>

/* บั๊ก: บรรทัดสุดท้ายควรเรียก countdown_sum(n - 1) ไม่ใช่ countdown_sum(n) */
long countdown_sum(int n) {
    if (n == 0) {
        return 0;
    }
    return n + countdown_sum(n);
}

int main(void) {
    printf("เริ่มคำนวณ countdown_sum(10)\n");
    long result = countdown_sum(10);
    printf("ผลลัพธ์ = %ld\n", result);
    return 0;
}
```

```bash
$ gcc -Wall -Wextra -Wpedantic -std=c17 -g -O0 stack_overflow.c -o stack_overflow
$ ./stack_overflow
เริ่มคำนวณ countdown_sum(10)
Segmentation fault (core dumped)
```

Debug ด้วย GDB:

```
$ gdb -q ./stack_overflow
Reading symbols from ./stack_overflow...
(gdb) run
Starting program: /home/user/stack_overflow
เริ่มคำนวณ countdown_sum(10)

Program received signal SIGSEGV, Segmentation fault.
0x0000555555555191 in countdown_sum (n=10) at stack_overflow.c:12
12	    return n + countdown_sum(n);
(gdb) backtrace 5
#0  0x0000555555555191 in countdown_sum (n=10) at stack_overflow.c:12
#1  0x0000555555555196 in countdown_sum (n=10) at stack_overflow.c:12
#2  0x0000555555555196 in countdown_sum (n=10) at stack_overflow.c:12
#3  0x0000555555555196 in countdown_sum (n=10) at stack_overflow.c:12
#4  0x0000555555555196 in countdown_sum (n=10) at stack_overflow.c:12
(More stack frames follow...)
(gdb) print n
$1 = 10
(gdb) quit
```

`backtrace 5` (จำกัดแสดงแค่ 5 frame แรก แทนที่จะพยายามแสดงทั้งหมดหลายหมื่น frame ซึ่ง
จะท่วมหน้าจอ) เผยให้เห็น pattern ที่ผิดปกติทันที: **ทุก frame มีค่า `n = 10` เหมือนกัน
หมดเลย ไม่มีการลดค่าลงแม้แต่ครั้งเดียว** ข้อความ "(More stack frames follow...)"
ยืนยันว่ามี stack frame อีกมหาศาลที่ไม่ได้แสดง (จำนวนหลักพันถึงหลักหมื่นก่อนที่ stack
จะเต็มจริง) นี่คือหลักฐานที่ฟ้องชัดเจนว่า **การเรียกซ้ำไม่เคยลดค่า `n` ลงเลย** — root
cause คือบรรทัด `return n + countdown_sum(n);` ที่ควรเป็น `countdown_sum(n - 1)`

เทคนิคการอ่าน backtrace ที่มี frame ซ้ำกันจำนวนมากแบบนี้ (ค่าพารามิเตอร์เหมือนกันทุก
frame) คือสัญญาณเตือนคลาสสิกของ **infinite recursion** เสมอ ไม่ต้องรอดูจนครบทุก frame
ก็สรุปสาเหตุได้จากแค่ 2-3 frame แรกที่แสดงออกมา

---

## สรุปท้ายบท

ใน Part นี้เราได้:

- เข้าใจข้อจำกัดของการ debug ด้วย `printf()` และเหตุผลที่ debugger อย่าง GDB มี
  ประสิทธิภาพเหนือกว่าอย่างชัดเจนสำหรับบั๊กที่ซับซ้อน
- เห็นความแตกต่างที่จับต้องได้ระหว่างการคอมไพล์ด้วยและไม่มี `-g` และเข้าใจว่าทำไม
  debug symbol จึงจำเป็นต่อการใช้งาน GDB อย่างมีประสิทธิภาพ
- ใช้คำสั่งพื้นฐานของ GDB ได้ครบชุด: `break`, `run`, `next`, `step`, `continue`,
  `print`, `backtrace`, `finish`, `info locals`, `info args`
- Debug บั๊ก null pointer dereference จริงจนหา root cause ได้ด้วย `backtrace` และ
  `print` ทีละขั้นตอน
- ใช้ `watch`/`rwatch` ตรวจจับบั๊ก off-by-one ที่ไม่ได้ทำให้โปรแกรม crash แต่ให้ผลลัพธ์
  ผิดเงียบๆ พร้อมเข้าใจความแตกต่างระหว่าง hardware watchpoint กับ software watchpoint
- เปิดใช้งานและวิเคราะห์ core dump ได้ด้วยตัวเอง เข้าใจว่าทำไมมันสำคัญมากสำหรับการ
  debug ปัญหาที่เกิดบน production server
- ใช้ GDB TUI mode ดู source code คู่กับสถานะการ debug แบบ real-time ในหน้าจอเดียว
- ฝึกใช้ conditional breakpoint และวิเคราะห์ stack overflow จาก infinite recursion
  ผ่านแบบฝึกหัดที่มีการรันและตรวจสอบจริงทุกขั้นตอน

GDB เป็นเครื่องมือที่ต้องฝึกฝนบ่อยๆ ถึงจะคล่อง ยิ่งเจอบั๊กหลากหลายรูปแบบมากเท่าไหร่
ยิ่งพัฒนาสัญชาตญาณในการเลือกคำสั่งที่เหมาะสมได้เร็วขึ้นเท่านั้น ใน **Part 38** เราจะ
เรียนรู้เครื่องมืออีกตัวหนึ่งที่สำคัญไม่แพ้กันคือ **Valgrind** ซึ่งเชี่ยวชาญเฉพาะทางด้าน
การตรวจจับ memory leak และ undefined behavior ที่ GDB ตรวจจับได้ยากหรือตรวจจับไม่ได้เลย
(เช่น การใช้ memory หลัง `free()` ไปแล้ว หรือการอ่าน memory ที่ยังไม่ได้ initialize)

**ต่อไป:** [Part 38 — Valgrind และการตรวจจับ Memory Leak](./part-038-valgrind.md)
