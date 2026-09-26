# Part 18: Makefile และ Build Automation (Step 137–144)

> Module B — C ระดับกลาง & โครงสร้างข้อมูล/อัลกอริทึม | Part 18 จาก 125
> Step ที่ครอบคลุมใน Part นี้: Step 137–144
> Part ก่อนหน้า: [Part 17 — Modular Programming (Header/Linkage)](./part-017-modular-programming.md) | Part ถัดไป: [Part 19 — Linked List (Singly/Doubly/Circular)](./part-019-linked-lists.md)

## เป้าหมายการเรียนรู้ (Learning Objectives)

หลังจบ Part นี้ ผู้เรียนจะสามารถ:

1. อธิบายได้ว่าทำไมโปรเจกต์ C ที่มีหลายไฟล์ถึงต้องการเครื่องมือ Build Automation แทนการพิมพ์
   คำสั่ง `gcc` ด้วยมือ
2. เขียน Makefile พื้นฐานตาม syntax `target: dependencies` ตามด้วย recipe ได้อย่างถูกต้อง
   (รวมถึงเข้าใจว่าทำไม Tab สำคัญมาก)
3. ใช้ตัวแปรใน Makefile (`CC`, `CFLAGS`, ตัวแปรที่ตั้งเอง) เพื่อลดความซ้ำซ้อนของโค้ด
4. ใช้ Automatic Variable (`$@`, `$<`, `$^`, `$?`) เพื่อเขียน rule ที่กระชับและยืดหยุ่น
5. ใช้ Pattern Rule (`%.o: %.c`) เพื่อลดการเขียน rule ซ้ำๆ สำหรับไฟล์จำนวนมาก
6. ใช้ `.PHONY` เพื่อประกาศ target ที่ไม่ใช่ไฟล์จริง เช่น `clean`, `all`, `run`
7. เขียน Makefile ที่ scale ได้กับโปรเจกต์ที่มีไฟล์ `.c` จำนวนมากโดยใช้ `wildcard`, `patsubst`
   และการติดตาม dependency ของ header อัตโนมัติ
8. เขียน Makefile ที่สมบูรณ์สำหรับ build โปรเจกต์ `math_utils` จาก Part 17 ให้ทำงานได้ครบวงจร

---

## 18.1 ทำไมต้องใช้ Makefile (Step 137)

ใน Part 17 เราต้องพิมพ์คำสั่งแบบนี้ทุกครั้งที่ต้องการ build โปรเจกต์:

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 -c math_utils.c -o math_utils.o
gcc -Wall -Wextra -Wpedantic -std=c17 -c main.c -o main.o
gcc math_utils.o main.o -o program
```

สำหรับโปรเจกต์เล็กๆ 2-3 ไฟล์ อาจจะยังพอทนได้ แต่ลองจินตนาการโปรเจกต์ที่มี 20, 50, หรือ 200
ไฟล์ `.c` — การพิมพ์คำสั่งด้วยมือแบบนี้จะกลายเป็นปัญหาใหญ่ทันที:

- **พิมพ์ผิดง่าย**: ลืม flag บางตัว ลืมไฟล์บางไฟล์ พิมพ์ชื่อไฟล์สลับกัน
- **ไม่รู้ว่าไฟล์ไหนต้องคอมไพล์ใหม่**: ถ้าแก้แค่ `math_utils.c` แต่คอมไพล์ `main.c` ใหม่ทั้งที่
  ไม่จำเป็น เสียเวลาโดยเปล่าประโยชน์ หรือแย่กว่านั้นคือ **ลืม** คอมไพล์ไฟล์ที่แก้จริงๆ ทำให้ได้
  โปรแกรมที่รันด้วยโค้ดเก่าโดยไม่รู้ตัว (บั๊กที่หาสาเหตุยากมาก)
- **ทำซ้ำไม่ได้ในเครื่องอื่น**: เพื่อนร่วมทีมหรือเซิร์ฟเวอร์ CI/CD ต้องรู้ลำดับคำสั่งเป๊ะๆ เหมือนกัน
  ทุกตัวอักษร ถ้าใครจำผิดหรือใช้ flag ไม่ตรงกัน ก็อาจได้ผลลัพธ์ที่ต่างกัน

**Make** คือเครื่องมือ Build Automation ที่เก่าแก่ที่สุดตัวหนึ่งในโลก (สร้างขึ้นตั้งแต่ปี 1976 ที่
Bell Labs — ที่เดียวกับที่ให้กำเนิดภาษา C) แต่ยังคงเป็นเครื่องมือมาตรฐานที่ใช้กันแพร่หลายที่สุด
ตัวหนึ่งจนถึงปัจจุบัน โดยเฉพาะในโปรเจกต์ C/C++ ระดับ System Programming, Linux Kernel,
และ Open Source ทั่วโลก หลักการสำคัญของ Make คือ:

> **สร้าง target ใหม่ก็ต่อเมื่อ dependency ของมันมีการเปลี่ยนแปลง (เวลาแก้ไขไฟล์ใหม่กว่า)**

Make จะเปรียบเทียบเวลาการแก้ไขล่าสุด (timestamp) ของไฟล์ผลลัพธ์กับไฟล์ต้นทาง ถ้าไฟล์
ต้นทางใหม่กว่า จึงจะสั่งคอมไพล์ใหม่ ถ้าไม่มีอะไรเปลี่ยน Make จะข้ามขั้นตอนนั้นไปเลย ทำให้การ
build ครั้งถัดๆ ไปเร็วขึ้นมากเมื่อแก้แค่ไฟล์เดียวในโปรเจกต์ใหญ่

ติดตั้ง Make (ถ้ายังไม่มี ซึ่งปกติมากับ `build-essential` ที่ติดตั้งไปแล้วใน Part 1):

```bash
sudo apt install make -y
make --version
```

---

## 18.2 Syntax พื้นฐานของ Makefile (Step 138)

ไฟล์ที่ Make อ่านเรียกว่า `Makefile` (ตัว M ใหญ่ ไม่มีนามสกุล) วางไว้ที่ root ของโปรเจกต์
โครงสร้างพื้นฐานของหนึ่ง **rule** มีรูปแบบตายตัวดังนี้:

```makefile
target: dependencies
	recipe
```

- **target**: ชื่อไฟล์ที่ต้องการสร้าง (หรือชื่อ action ที่ไม่ใช่ไฟล์จริง จะพูดถึงในหัวข้อ 18.6)
- **dependencies**: รายชื่อไฟล์ที่ target นี้ต้องพึ่งพา (ถ้าไฟล์เหล่านี้ใหม่กว่า target จะสั่ง
  rebuild)
- **recipe**: คำสั่ง shell ที่จะรันเพื่อสร้าง target — **บรรทัดนี้ต้องขึ้นต้นด้วย Tab เท่านั้น**
  ห้ามใช้ Space เด็ดขาด

### ตัวอย่างแรก: Makefile อย่างง่ายที่สุด

```makefile
program: math_utils.o main.o
	gcc math_utils.o main.o -o program

math_utils.o: math_utils.c math_utils.h
	gcc -Wall -Wextra -Wpedantic -std=c17 -c math_utils.c -o math_utils.o

main.o: main.c math_utils.h
	gcc -Wall -Wextra -Wpedantic -std=c17 -c main.c -o main.o
```

รันด้วยคำสั่ง:

```bash
make
```

ผลลัพธ์:

```
gcc -Wall -Wextra -Wpedantic -std=c17 -c math_utils.c -o math_utils.o
gcc -Wall -Wextra -Wpedantic -std=c17 -c main.c -o main.o
gcc math_utils.o main.o -o program
```

Make อ่าน rule แรกสุด (`program`) เป็น **default target** เมื่อเราพิมพ์ `make` เฉยๆ โดยไม่ระบุ
target แล้วมันจะเห็นว่า `program` ต้องการ `math_utils.o` และ `main.o` ก่อน จึงไล่ไปสร้างสอง
ไฟล์นั้นก่อน (ตามลำดับที่จำเป็น ไม่จำเป็นต้องเรียงจากบนลงล่างเสมอไป) แล้วค่อยรัน recipe ของ
`program` เป็นลำดับสุดท้าย

ลองรัน `make` อีกครั้งทันทีโดยไม่แก้ไขอะไรเลย:

```bash
make
```

```
make: 'program' is up to date.
```

Make ตรวจสอบ timestamp แล้วพบว่าไม่มีไฟล์ต้นทางไหนใหม่กว่า `program` เลย จึงไม่ทำอะไร —
นี่คือหัวใจสำคัญที่ทำให้ Make มีประโยชน์มากกว่าการเขียน shell script ธรรมดาที่รันคำสั่งทุก
ครั้งไม่ว่าอะไรจะเปลี่ยนหรือไม่

### กับดักที่พบบ่อยที่สุดของ Make: Tab vs Space

```bash
printf 'hello:\n    echo hi\n' > Makefile   # ใช้ Space 4 ตัวแทน Tab โดยไม่ตั้งใจ
make
```

```
Makefile:2: *** missing separator.  Stop.
```

Make เขียนขึ้นในยุคที่ Text Editor ยังไม่ค่อยแยกความแตกต่างระหว่าง Tab กับ Space ให้เห็นชัด
กฎที่ Make ยึดถือมาตั้งแต่ต้นคือ **บรรทัด recipe ทุกบรรทัดต้องขึ้นต้นด้วยอักขระ Tab เท่านั้น**
ถ้า editor ของเรา (เช่น VS Code) ตั้งค่าให้แปลง Tab เป็น Space อัตโนมัติ (auto-indent เป็นเรื่อง
ปกติของหลาย editor) การเขียน Makefile จะพังทันที ควรตั้งค่า VS Code ให้แสดง whitespace
(`"editor.renderWhitespace": "all"`) และปิดการแปลง Tab เป็น Space เฉพาะไฟล์ `Makefile`

---

## 18.3 ตัวแปรใน Makefile: CC, CFLAGS (Step 139)

สังเกตว่า Makefile ด้านบนมีคำสั่ง `gcc -Wall -Wextra -Wpedantic -std=c17` ซ้ำอยู่หลายที่
ถ้าต้องการเปลี่ยน flag (เช่น เพิ่ม `-O2`) ต้องไปแก้ทุกจุด ซึ่งเสี่ยงตกหล่น Make รองรับ **ตัวแปร**
(Variable) เพื่อแก้ปัญหานี้:

```makefile
CC = gcc
CFLAGS = -Wall -Wextra -Wpedantic -std=c17

program: math_utils.o main.o
	$(CC) math_utils.o main.o -o program

math_utils.o: math_utils.c math_utils.h
	$(CC) $(CFLAGS) -c math_utils.c -o math_utils.o

main.o: main.c math_utils.h
	$(CC) $(CFLAGS) -c main.c -o main.o
```

การอ้างอิงตัวแปรใน Make ใช้ `$(ชื่อตัวแปร)` (หรือ `${ชื่อตัวแปร}` ก็ได้ ทั้งสองแบบเทียบเท่ากัน
แต่ในหลักสูตรนี้จะใช้วงเล็บกลมเป็นหลักตามธรรมเนียมที่พบบ่อยที่สุด)

`CC` และ `CFLAGS` ไม่ใช่ชื่อที่ Make บังคับตายตัว แต่เป็น **ธรรมเนียมสากล** ที่โปรเจกต์ C/C++
เกือบทุกโปรเจกต์ในโลกใช้ชื่อนี้ตรงกัน (Make เองก็มีค่า default ของตัวแปรเหล่านี้ให้ในตัวอยู่แล้ว)
ตัวแปรมาตรฐานที่ควรรู้จัก:

| ตัวแปร | ความหมาย |
|---|---|
| `CC` | คอมไพเลอร์ที่ใช้ (เช่น `gcc`, `clang`) |
| `CFLAGS` | flag สำหรับขั้นตอน compile (เช่น `-Wall -std=c17`) |
| `CXX` | คอมไพเลอร์ C++ (จะใช้ตั้งแต่ Module D) |
| `CXXFLAGS` | flag สำหรับขั้นตอน compile C++ |
| `LDFLAGS` | flag สำหรับขั้นตอน link (เช่น `-L`, `-l` สำหรับ library) |
| `LDLIBS` | รายชื่อ library ที่ต้อง link (เช่น `-lm` สำหรับ math library) |

การใช้ชื่อมาตรฐานเหล่านี้ทำให้คนอื่นที่เปิด Makefile ของเราเข้าใจได้ทันทีโดยไม่ต้องอ่านทั้งไฟล์
และยังทำให้ผู้ใช้สามารถ override ค่าได้จาก command line โดยไม่ต้องแก้ไฟล์:

```bash
make CC=clang
make CFLAGS="-Wall -Wextra -O2"
```

---

## 18.4 Automatic Variable: $@ $< $^ $? (Step 140)

สังเกตว่า recipe แต่ละอันยังคงพิมพ์ชื่อไฟล์ซ้ำกับที่เขียนไว้ใน target/dependencies อยู่ดี
(`math_utils.c` ปรากฏทั้งใน dependency และใน recipe) Make มี **Automatic Variable** ที่แทน
ค่าเหล่านี้ให้อัตโนมัติ ทำให้ไม่ต้องพิมพ์ชื่อไฟล์ซ้ำเลย:

| ตัวแปร | ความหมาย |
|---|---|
| `$@` | ชื่อของ **target** (ฝั่งซ้ายของ `:`) |
| `$<` | ชื่อของ **dependency ตัวแรก** เท่านั้น |
| `$^` | ชื่อของ **dependency ทั้งหมด** (คั่นด้วยช่องว่าง, ไม่ซ้ำ) |
| `$?` | ชื่อของ dependency ที่ **ใหม่กว่า target** เท่านั้น (ตัวที่ต้องอัปเดตจริงๆ) |

เขียน Makefile เดิมใหม่ด้วย Automatic Variable:

```makefile
CC = gcc
CFLAGS = -Wall -Wextra -Wpedantic -std=c17

program: math_utils.o main.o
	$(CC) $^ -o $@

math_utils.o: math_utils.c math_utils.h
	$(CC) $(CFLAGS) -c $< -o $@

main.o: main.c math_utils.h
	$(CC) $(CFLAGS) -c $< -o $@
```

รัน `make` จะได้ผลลัพธ์เหมือนเดิมทุกประการ:

```
gcc -Wall -Wextra -Wpedantic -std=c17 -c math_utils.c -o math_utils.o
gcc -Wall -Wextra -Wpedantic -std=c17 -c main.c -o main.o
gcc math_utils.o main.o -o program
```

สังเกตจุดสำคัญ: rule ของ `math_utils.o` มี dependency สองไฟล์ (`math_utils.c` และ
`math_utils.h`) แต่ recipe ใช้ `$<` ซึ่งหมายถึงตัวแรกเท่านั้นคือ `math_utils.c` — เราต้องการแบบ
นี้จริงๆ เพราะ `gcc -c` รับไฟล์ `.c` เป็น input การใส่ `math_utils.h` ไว้ใน dependency ไม่ได้มี
ไว้ให้ compiler compile แต่มีไว้บอก Make ว่า **ถ้า `math_utils.h` เปลี่ยน ให้ compile
`math_utils.o` ใหม่ด้วย** (เพราะ `.c` include header ตัวนี้อยู่ ถ้า header เปลี่ยนแต่ `.o` ไม่ถูก
compile ใหม่ อาจได้โค้ดที่ไม่ตรงกับ prototype ล่าสุด)

---

## 18.5 Pattern Rule: %.o: %.c (Step 141)

ถ้าโปรเจกต์มีไฟล์ `.c` 20 ไฟล์ การเขียน rule แยกทีละไฟล์แบบด้านบนจะยาวและซ้ำซากมาก
Make มี **Pattern Rule** ที่ใช้เครื่องหมาย `%` แทน "อะไรก็ได้ที่ตรงกัน" เพื่อเขียน rule เดียว
ครอบคลุมทุกไฟล์ `.c → .o` ในโปรเจกต์:

```makefile
CC = gcc
CFLAGS = -Wall -Wextra -Wpedantic -std=c17

program: math_utils.o main.o
	$(CC) $^ -o $@

%.o: %.c
	$(CC) $(CFLAGS) -c $< -o $@
```

`%.o: %.c` อ่านว่า "สำหรับไฟล์ `.o` ใดๆ ก็ตามที่ต้องการสร้าง ให้หาไฟล์ `.c` ชื่อเดียวกัน (ตัด
นามสกุลออกแล้วแทนด้วย `.c`) มาเป็น dependency" เช่นเมื่อ Make ต้องการสร้าง `main.o` มันจะ
จับคู่กับ `main.c` โดยอัตโนมัติ และเมื่อต้องการ `math_utils.o` ก็จับคู่กับ `math_utils.c` — เพียง
rule เดียวรองรับทุกไฟล์ `.c` ในโปรเจกต์โดยไม่ต้องเขียนซ้ำเลย

รัน `make` ได้ผลลัพธ์เดียวกันทุกประการกับก่อนหน้า:

```bash
rm -f program *.o
make
```

```
gcc -Wall -Wextra -Wpedantic -std=c17 -c math_utils.c -o math_utils.o
gcc -Wall -Wextra -Wpedantic -std=c17 -c main.c -o main.o
gcc math_utils.o main.o -o program
```

> **หมายเหตุ**: ในตัวอย่างนี้ Pattern Rule ไม่ได้ระบุ `math_utils.h` เป็น dependency อีกต่อไป
> (เพราะ pattern rule จับคู่ได้แค่ตามรูปแบบชื่อไฟล์ ไม่รู้เรื่อง `#include` ภายใน) นี่คือข้อจำกัดที่
> เราจะแก้ไขอย่างเป็นระบบในหัวข้อ 18.7 ด้วยเทคนิค automatic dependency generation

---

## 18.6 Phony Target: .PHONY, clean, run (Step 142)

บางครั้งเราต้องการ target ที่ไม่ได้สร้างไฟล์จริง แต่เป็นแค่ **action** ที่อยากให้ Make รันให้
เช่น ลบไฟล์ผลลัพธ์การ build ทั้งหมด (`clean`) หรือ build แล้วรันทันที (`run`):

```makefile
CC = gcc
CFLAGS = -Wall -Wextra -Wpedantic -std=c17

program: math_utils.o main.o
	$(CC) $^ -o $@

%.o: %.c
	$(CC) $(CFLAGS) -c $< -o $@

.PHONY: clean run

run: program
	./program

clean:
	rm -f program *.o
```

```bash
make run
```

```
gcc -Wall -Wextra -Wpedantic -std=c17 -c math_utils.c -o math_utils.o
gcc -Wall -Wextra -Wpedantic -std=c17 -c main.c -o main.o
gcc math_utils.o main.o -o program
./program
mu_add(2, 3)      = 5
mu_subtract(10,4) = 6
mu_power(2, 10)   = 1024
mu_gcd(48, 18)    = 6
mu_is_prime(17)   = 1
mu_average(...)   = 80.00
เรียกฟังก์ชันใน module ไปทั้งหมด 6 ครั้ง
```

### ทำไมต้อง `.PHONY`

ลองจินตนาการว่าในโฟลเดอร์โปรเจกต์ดันมีไฟล์ชื่อ `clean` อยู่จริง (อาจเป็นไฟล์ที่สร้างขึ้นโดย
บังเอิญ) Make จะเปรียบเทียบ timestamp ของไฟล์ `clean` กับ dependency ของมัน (ในที่นี้ไม่มี
dependency เลย) แล้วสรุปว่า "ไฟล์ `clean` มีอยู่แล้วและไม่มี dependency ให้ต้องอัปเดต" จึง
**ข้าม** ไม่รัน recipe เลย! `.PHONY` คือการบอก Make อย่างชัดเจนว่า "target เหล่านี้ไม่ใช่ไฟล์
จริง อย่าไปเช็ค timestamp ให้รัน recipe ทุกครั้งที่ถูกเรียก"

> **กฎทองของ Part นี้**: ทุก target ที่ไม่ใช่ชื่อไฟล์ผลลัพธ์จริง (เช่น `clean`, `all`, `run`,
> `test`, `install`) ต้องถูกประกาศใน `.PHONY` เสมอ เป็นแนวปฏิบัติมาตรฐานที่ Makefile
> คุณภาพดีทุกไฟล์ในโลกทำกัน

### Target `all` — ธรรมเนียมของ default target

โปรเจกต์ส่วนใหญ่นิยมสร้าง target ชื่อ `all` ไว้เป็น target แรกสุดของไฟล์ (เพื่อให้เป็น default
เมื่อพิมพ์ `make` เฉยๆ) แม้จะมี target เดียวก็ตาม เพราะเป็นชื่อที่คาดเดาได้และเป็นสากล:

```makefile
.PHONY: all clean run

all: program
```

### โบนัส: ทำให้ Makefile "เงียบ" ด้วย `@` และ build เร็วขึ้นด้วย `-j`

โดยปกติ Make จะพิมพ์คำสั่งทุกคำสั่งที่มันรันออกทาง terminal ก่อนรันจริง (อย่างที่เห็นในทุก
ตัวอย่างด้านบน) ถ้าต้องการให้ Makefile ดูสะอาดขึ้นโดยพิมพ์แค่ข้อความสรุปสั้นๆ แทน สามารถ
ใส่เครื่องหมาย `@` ไว้หน้าคำสั่งใน recipe เพื่อ "ปิดเสียง" การพิมพ์คำสั่งนั้นได้:

```makefile
%.o: %.c
	@echo "Compiling $<..."
	@$(CC) $(CFLAGS) -c $< -o $@
```

```bash
$ make
Compiling main.c...
Compiling math_utils.c...
```

อีกเทคนิคหนึ่งที่มีประโยชน์มากเมื่อโปรเจกต์มีไฟล์จำนวนมากคือ flag `-j` (jobs) ที่สั่งให้ Make
คอมไพล์หลายไฟล์ **พร้อมกัน** แทนที่จะทำทีละไฟล์ตามลำดับ:

```bash
make -j4      # ใช้ 4 process คอมไพล์พร้อมกัน
make -j$(nproc)   # ใช้จำนวน core ของ CPU ทั้งหมดที่มี (nproc คือคำสั่ง Linux ที่คืนจำนวน core)
```

เนื่องจากไฟล์ `.o` แต่ละไฟล์ในโปรเจกต์เราไม่ได้ขึ้นกับกันและกัน (ต่างคนต่าง compile จาก `.c`
ของตัวเอง) Make จึงสามารถแจกงานคอมไพล์แต่ละไฟล์ไปให้ CPU หลาย core ทำงานพร้อมกันได้
อย่างปลอดภัย ในโปรเจกต์ขนาดใหญ่ที่มีไฟล์เป็นร้อยเป็นพัน (เช่นตอน compile Linux Kernel)
`-j` สามารถลดเวลา build ลงได้หลายเท่าตัว

---

## 18.7 Scale ให้รองรับหลายไฟล์: wildcard, patsubst, Auto-Dependency (Step 143)

Pattern Rule ช่วยลดการเขียน rule ซ้ำสำหรับ `.c → .o` ไปมากแล้ว แต่ target หลัก (`program`)
ยังต้องพิมพ์รายชื่อไฟล์ `.o` ทั้งหมดด้วยมืออยู่ดี (`math_utils.o main.o`) ถ้าโปรเจกต์มี 30 ไฟล์
`.c` นี่จะเป็นภาระอีกจุดหนึ่งที่ลืมอัปเดตได้ง่าย Make มีฟังก์ชัน built-in ช่วยแก้ปัญหานี้:

### `$(wildcard pattern)` — ค้นหาไฟล์ที่ตรงกับ pattern

```makefile
SRCS = $(wildcard *.c)
```

`$(wildcard *.c)` จะขยายเป็นรายชื่อไฟล์ `.c` ทั้งหมดในโฟลเดอร์ปัจจุบันโดยอัตโนมัติ (เช่น
`math_utils.c main.c`) ทำให้เมื่อเพิ่มไฟล์ `.c` ใหม่เข้าโปรเจกต์ ไม่ต้องแก้ Makefile เลย

### `$(patsubst pattern,replacement,text)` — แปลงชื่อไฟล์

```makefile
OBJS = $(patsubst %.c,%.o,$(SRCS))
```

แปลงรายชื่อไฟล์ `.c` ใน `SRCS` ให้เป็นรายชื่อไฟล์ `.o` ที่สอดคล้องกัน (เช่น
`math_utils.c main.c` → `math_utils.o main.o`) โดยอัตโนมัติเช่นกัน

### Auto-Dependency Generation — แก้ปัญหา header ไม่ถูกติดตาม

จากหัวข้อ 18.5 เราพบข้อจำกัดว่า Pattern Rule ไม่รู้เรื่องความสัมพันธ์ระหว่าง `.c` กับ header
ที่มัน `#include` วิธีแก้มาตรฐานคือให้ **compiler เป็นคนสร้างไฟล์ dependency ให้เอง** ด้วย flag
`-MMD -MP`:

```makefile
CFLAGS = -Wall -Wextra -Wpedantic -std=c17 -MMD -MP
```

เมื่อคอมไพล์ `math_utils.c` ด้วย flag นี้ นอกจาก `math_utils.o` แล้ว gcc จะสร้างไฟล์
`math_utils.d` ที่มีเนื้อหาประมาณนี้ให้อัตโนมัติ:

```makefile
math_utils.o: math_utils.c math_utils.h
```

ซึ่งคือ rule ที่บอก Make ว่า `math_utils.o` ขึ้นกับ `math_utils.h` ด้วย — ตรงกับที่เราเคยเขียน
เองด้วยมือในหัวข้อ 18.2! เราแค่สั่งให้ Makefile `-include` ไฟล์ `.d` เหล่านี้เข้ามา:

```makefile
DEPS = $(patsubst %.c,%.d,$(SRCS))

-include $(DEPS)
```

เครื่องหมาย `-` หน้า `include` บอกให้ Make **ไม่ error** ถ้าไฟล์ `.d` ยังไม่มีอยู่ (เช่นตอน build
ครั้งแรกที่ยังไม่เคยคอมไพล์เลย)

---

## 18.8 Makefile ฉบับสมบูรณ์สำหรับโปรเจกต์ math_utils (Step 144)

รวมทุกเทคนิคที่เรียนมาใน Part นี้เข้าด้วยกัน เพื่อ build โปรเจกต์ `math_utils.h/.c` +
`main.c` จาก Part 17 ให้สมบูรณ์:

```makefile
# ============================================================
# Makefile — โปรเจกต์ math_utils (สืบทอดจาก Part 17)
# ============================================================

CC      = gcc
CFLAGS  = -Wall -Wextra -Wpedantic -std=c17 -MMD -MP
LDFLAGS =

TARGET  = program
SRCS    = $(wildcard *.c)
OBJS    = $(patsubst %.c,%.o,$(SRCS))
DEPS    = $(patsubst %.c,%.d,$(SRCS))

.PHONY: all run clean

# target แรกสุด = default target เมื่อพิมพ์ "make" เฉยๆ
all: $(TARGET)

$(TARGET): $(OBJS)
	$(CC) $(OBJS) -o $(TARGET) $(LDFLAGS)

# Pattern Rule: คอมไพล์ไฟล์ .c ใดๆ เป็น .o ที่ตรงกัน
%.o: %.c
	$(CC) $(CFLAGS) -c $< -o $@

run: $(TARGET)
	./$(TARGET)

clean:
	rm -f $(TARGET) $(OBJS) $(DEPS)

# นำ dependency ของ header ที่ compiler สร้างให้มาใช้ (ถ้ามี)
-include $(DEPS)
```

### ทดสอบทุก target

```bash
$ make
gcc -Wall -Wextra -Wpedantic -std=c17 -MMD -MP -c main.c -o main.o
gcc -Wall -Wextra -Wpedantic -std=c17 -MMD -MP -c math_utils.c -o math_utils.o
gcc main.o math_utils.o -o program

$ ls
Makefile  main.c  main.d  main.o  math_utils.c  math_utils.d  math_utils.h  math_utils.o  program

$ make
make: Nothing to be done for 'all'.

$ touch math_utils.h
$ make
gcc -Wall -Wextra -Wpedantic -std=c17 -MMD -MP -c main.c -o main.o
gcc -Wall -Wextra -Wpedantic -std=c17 -MMD -MP -c math_utils.c -o math_utils.o
gcc main.o math_utils.o -o program

$ make run
./program
mu_add(2, 3)      = 5
mu_subtract(10,4) = 6
mu_power(2, 10)   = 1024
mu_gcd(48, 18)    = 6
mu_is_prime(17)   = 1
mu_average(...)   = 80.00
เรียกฟังก์ชันใน module ไปทั้งหมด 6 ครั้ง

$ make clean
rm -f program main.o math_utils.o main.d math_utils.d

$ ls
Makefile  main.c  math_utils.c  math_utils.h
```

สังเกตพฤติกรรมสำคัญ: เมื่อสั่ง `touch math_utils.h` (จำลองการแก้ไข header) Make คอมไพล์
**ทั้งสองไฟล์ `.o` ใหม่** แม้เราจะแก้แค่ header ตัวเดียว เพราะทั้ง `main.c` และ `math_utils.c`
ต่าง `#include "math_utils.h"` และไฟล์ `.d` ที่ compiler สร้างให้บอก Make ไว้แล้วว่าทั้งคู่ขึ้นกับ
header นี้ — นี่คือสิ่งที่ Pattern Rule เพียวๆ (ไม่มี auto-dependency) จะพลาดไป

Makefile ฉบับนี้ยังมีคุณสมบัติสำคัญคือ **scale ได้ทันทีเมื่อโปรเจกต์โตขึ้น** — ถ้าเพิ่มไฟล์ `.c`
ใหม่เข้าโฟลเดอร์ (เช่น `string_utils.c` ในแบบฝึกหัดข้อถัดไป) ไม่ต้องแก้ Makefile แม้แต่บรรทัด
เดียว เพราะ `$(wildcard *.c)` จะเห็นไฟล์ใหม่โดยอัตโนมัติ

---

## ข้อผิดพลาดที่พบบ่อย (Common Pitfalls)

1. **ใช้ Space แทน Tab ในบรรทัด recipe** — เกิด error `missing separator. Stop.` ทันที
   ตรวจสอบโดยเปิด "Render Whitespace" ใน editor หรือใช้คำสั่ง `cat -A Makefile` เพื่อดู
   อักขระ Tab (`^I`) กับ Space ให้ชัดเจน
2. **ลืม `.PHONY`** — ถ้ามีไฟล์ในโฟลเดอร์ชื่อตรงกับ phony target (เช่น มีไฟล์ชื่อ `clean` หรือ
   `test` อยู่จริงโดยบังเอิญ) Make จะคิดว่า target นั้น "up to date" แล้วไม่รัน recipe เลย
3. **Header ไม่ถูกใส่เป็น dependency (หรือไม่ได้ใช้ auto-dependency)** — แก้ header แล้ว
   `make` ไม่คอมไพล์ไฟล์ที่ include header นั้นใหม่ ทำให้รันโปรแกรมด้วยโค้ดที่ไม่ตรงกับ header
   ล่าสุดโดยไม่รู้ตัว (บั๊กที่ตรวจจับยากมาก) แนวทางที่ดีที่สุดคือใช้ `-MMD -MP` เสมอตามที่สอนใน
   หัวข้อ 18.7
4. **ลืม `$(CC)` แล้วเขียน `gcc` ตรงๆ ปนกับที่ใช้ `$(CC)`** — ทำให้เปลี่ยน compiler ผ่าน
   `make CC=clang` ไม่ครบทุกจุด ควรใช้ตัวแปรให้สม่ำเสมอตลอดทั้งไฟล์
5. **ลืม `clean` ไฟล์ `.d` ที่เกิดจาก `-MMD`** — ทำให้โฟลเดอร์โปรเจกต์เกลื่อนไปด้วยไฟล์ที่ไม่ได้
   ใช้งานจริง ควรใส่ `$(DEPS)` ไว้ใน `rm -f` ของ target `clean` เสมอเมื่อใช้เทคนิคนี้
6. **วาง recipe ของสอง target ติดกันโดยไม่มีบรรทัดว่างคั่น แล้วอ่านยาก** — แม้จะไม่ error
   แต่ทำให้ Makefile อ่านยากขึ้นมากเมื่อมี rule จำนวนมาก ควรเว้นบรรทัดว่างระหว่างแต่ละ rule
   เสมอเพื่อความเป็นระเบียบ

---

## แบบฝึกหัดท้ายบท

1. เขียน Makefile สำหรับโปรเจกต์ `string_utils` จาก Part 17 (แบบฝึกหัดข้อ 1) ที่มี target
   `all`, `run`, `clean` ครบถ้วน โดยใช้ `wildcard` และ `patsubst` (ห้ามพิมพ์ชื่อไฟล์ `.c`/`.o`
   ตรงๆ ใน Makefile เลย)
2. เพิ่มความสามารถให้ Makefile ของโปรเจกต์ `math_utils` รองรับตัวแปร `MODE` โดยที่
   `make` (ไม่ระบุ `MODE`) หรือ `make MODE=release` จะคอมไพล์ด้วย `-O2` ในขณะที่
   `make MODE=debug` จะคอมไพล์ด้วย `-g -O0 -DDEBUG_BUILD` แทน (ใช้ `ifeq`/`endif`)
3. ทดลองสร้างไฟล์ `Makefile` ที่มี recipe เยื้องด้วย space แทน tab ด้วยตัวเอง สังเกต error
   message ที่ได้ แล้วอธิบายว่าทำไม Make ถึงต้องการ Tab เป็นการเฉพาะ (ค้นคว้าเพิ่มเติมว่า
   เวอร์ชันใหม่ๆ ของ GNU Make มีวิธีเปลี่ยน separator character ได้หรือไม่)
4. เขียน Makefile สำหรับโปรเจกต์ validator/logger/main จาก Part 17 (แบบฝึกหัดข้อ 5) ที่ใช้
   auto-dependency generation (`-MMD -MP`) แล้วทดสอบว่าเมื่อแก้ไข `error_common.h`
   ไฟล์ `.o` ทั้งสามไฟล์ถูกคอมไพล์ใหม่ทั้งหมดจริงหรือไม่
5. เพิ่ม target ชื่อ `install` เข้าไปใน Makefile ของโปรเจกต์ `math_utils` ที่ copy ไฟล์
   `program` ไปไว้ที่โฟลเดอร์ `~/bin/` ของผู้ใช้ (สร้างโฟลเดอร์นี้ก่อนถ้ายังไม่มี) พร้อมประกาศ
   `.PHONY` ให้ถูกต้อง
6. อธิบายด้วยคำพูดตัวเอง (เขียนเป็นคอมเมนต์ในไฟล์ Makefile) ว่าทำไม Pattern Rule
   (`%.o: %.c`) เพียงอย่างเดียวถึงไม่เพียงพอที่จะรับประกันว่าไฟล์ `.o` จะถูกคอมไพล์ใหม่เมื่อ
   header ที่มันขึ้นกับมีการเปลี่ยนแปลง และเทคนิค `-MMD -MP` แก้ปัญหานี้อย่างไร

### แนวทางเฉลยข้อ 1

```makefile
# ============================================================
# Makefile — โปรเจกต์ string_utils
# ============================================================

CC      = gcc
CFLAGS  = -Wall -Wextra -Wpedantic -std=c17 -MMD -MP

TARGET  = program
SRCS    = $(wildcard *.c)
OBJS    = $(patsubst %.c,%.o,$(SRCS))
DEPS    = $(patsubst %.c,%.d,$(SRCS))

.PHONY: all run clean

all: $(TARGET)

$(TARGET): $(OBJS)
	$(CC) $(OBJS) -o $(TARGET)

%.o: %.c
	$(CC) $(CFLAGS) -c $< -o $@

run: $(TARGET)
	./$(TARGET)

clean:
	rm -f $(TARGET) $(OBJS) $(DEPS)

-include $(DEPS)
```

ทดสอบ:

```bash
$ make run
gcc -Wall -Wextra -Wpedantic -std=c17 -MMD -MP -c main.c -o main.o
gcc -Wall -Wextra -Wpedantic -std=c17 -MMD -MP -c string_utils.c -o string_utils.o
gcc main.o string_utils.o -o program
./program
reverse("Hello") = "olleH"
is_palindrome("level") = 1
is_palindrome("Racecar") = 1
is_palindrome("Hello") = 0
is_palindrome("A") = 1

$ make clean
rm -f program main.o string_utils.o main.d string_utils.d
```

Makefile นี้ **เหมือนกันทุกตัวอักษร** กับ Makefile ของโปรเจกต์ `math_utils` ในหัวข้อ 18.8
(ยกเว้นคอมเมนต์หัวไฟล์) — นี่คือพลังของการเขียน Makefile แบบ generic ด้วย `wildcard` และ
`patsubst`: เราสามารถ copy Makefile ตัวเดียวไปใช้กับโปรเจกต์ C หลายไฟล์แบบง่ายๆ (single
executable, ไฟล์ `.c` ทั้งหมดอยู่โฟลเดอร์เดียวกัน) ได้เกือบทุกโปรเจกต์โดยไม่ต้องแก้ไขอะไรเลย

### แนวทางเฉลยข้อ 2

```makefile
# ============================================================
# Makefile — โปรเจกต์ math_utils รองรับ MODE=debug|release
# ============================================================

CC         = gcc
WARN_FLAGS = -Wall -Wextra -Wpedantic -std=c17

ifeq ($(MODE),debug)
CFLAGS = $(WARN_FLAGS) -g -O0 -DDEBUG_BUILD -MMD -MP
else
CFLAGS = $(WARN_FLAGS) -O2 -MMD -MP
endif

TARGET  = program
SRCS    = $(wildcard *.c)
OBJS    = $(patsubst %.c,%.o,$(SRCS))
DEPS    = $(patsubst %.c,%.d,$(SRCS))

.PHONY: all clean

all: $(TARGET)

$(TARGET): $(OBJS)
	$(CC) $(OBJS) -o $(TARGET)

%.o: %.c
	$(CC) $(CFLAGS) -c $< -o $@

clean:
	rm -f $(TARGET) $(OBJS) $(DEPS)

-include $(DEPS)
```

ทดสอบทั้งสองโหมด (ต้อง `make clean` ระหว่างสลับโหมด เพราะไฟล์ `.o` เดิมถูกคอมไพล์ด้วย
flag ต่างกัน Make จะไม่รู้ว่าต้อง rebuild เนื่องจากตรวจสอบแค่ timestamp ของไฟล์ ไม่ได้ตรวจสอบ
ว่าค่าตัวแปรเปลี่ยนไปหรือไม่):

```bash
$ make
gcc -Wall -Wextra -Wpedantic -std=c17 -O2 -MMD -MP -c main.c -o main.o
gcc -Wall -Wextra -Wpedantic -std=c17 -O2 -MMD -MP -c math_utils.c -o math_utils.o
gcc main.o math_utils.o -o program

$ make clean
rm -f program main.o math_utils.o main.d math_utils.d

$ make MODE=debug
gcc -Wall -Wextra -Wpedantic -std=c17 -g -O0 -DDEBUG_BUILD -MMD -MP -c main.c -o main.o
gcc -Wall -Wextra -Wpedantic -std=c17 -g -O0 -DDEBUG_BUILD -MMD -MP -c math_utils.c -o math_utils.o
gcc main.o math_utils.o -o program
```

`ifeq (arg1,arg2)` ... `else` ... `endif` เป็น **Conditional Directive** ของ Make เอง (ไม่ใช่
`#ifdef` ของ preprocessor ภาษา C) ทำงานตอนที่ Make กำลังอ่านไฟล์ Makefile ก่อนที่จะรัน
recipe ใดๆ เลย — เป็นเครื่องมือสำคัญที่ทำให้ Makefile หนึ่งไฟล์รองรับหลาย configuration
ของการ build ได้ ซึ่งจะได้ใช้งานอย่างเข้มข้นขึ้นอีกเมื่อโปรเจกต์โตขึ้นในภายหลัง (และในที่สุดจะ
เปลี่ยนไปใช้เครื่องมือที่ทรงพลังกว่าอย่าง **CMake** ใน Part 91 สำหรับโปรเจกต์ C++ ขนาดใหญ่)

---

## สรุปท้ายบท

ใน Part นี้เราได้:

- เข้าใจว่าทำไมโปรเจกต์ C หลายไฟล์ถึงต้องใช้ Build Automation แทนการพิมพ์คำสั่ง `gcc`
  ด้วยมือทุกครั้ง และหลักการพื้นฐานของ Make: rebuild เฉพาะเมื่อ dependency เปลี่ยนแปลง
- เขียน Makefile ตาม syntax `target: dependencies` + recipe (และจำได้แม่นว่า recipe ต้อง
  ขึ้นต้นด้วย Tab เท่านั้น)
- ใช้ตัวแปรมาตรฐาน `CC`, `CFLAGS` เพื่อลดความซ้ำซ้อนและทำให้ override ค่าได้จาก command
  line
- ใช้ Automatic Variable `$@`, `$<`, `$^` เพื่อเขียน rule ที่กระชับ
- ใช้ Pattern Rule `%.o: %.c` เพื่อรองรับไฟล์ `.c` จำนวนมากด้วย rule เดียว
- ใช้ `.PHONY` เพื่อประกาศ target ที่ไม่ใช่ไฟล์จริงอย่าง `all`, `run`, `clean`
- ประกอบเทคนิคทั้งหมดเป็น Makefile ที่ scale ได้จริงด้วย `wildcard`, `patsubst`, และ
  auto-dependency generation ผ่าน `-MMD -MP`
- เขียน Makefile ที่สมบูรณ์สำหรับโปรเจกต์ `math_utils` จาก Part 17 ที่ build, run, และ clean
  ได้ครบวงจรด้วยคำสั่งเดียว

จนถึงตอนนี้เราเน้นเรื่องการเขียนโค้ดและจัดระเบียบโปรเจกต์ C มาโดยตลอด ใน **Part 19**
เราจะเริ่มเข้าสู่โลกของ **โครงสร้างข้อมูล (Data Structures)** อย่างจริงจัง โดยเริ่มจาก
**Linked List** — โครงสร้างข้อมูลแบบไดนามิกตัวแรกที่อาศัย Pointer และ Dynamic Memory
Allocation ที่เรียนมาใน Part 9 และ Part 11 มาประกอบกันเป็นโครงสร้างที่ยืดหยุ่นกว่า Array
ธรรมดามาก

**ต่อไป:** [Part 19 — Linked List (Singly/Doubly/Circular)](./part-019-linked-lists.md)
