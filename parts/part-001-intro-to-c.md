# Part 1: เริ่มต้นกับภาษา C (Step 1–8)

> Module A — รากฐานภาษา C (Foundations of C) | Part 1 จาก 125
> Step ที่ครอบคลุมใน Part นี้: Step 1–8
> Part ก่อนหน้า: ไม่มี (นี่คือ Part แรก) | Part ถัดไป: [Part 2 — ตัวแปร ชนิดข้อมูล และตัวดำเนินการ](./part-002-variables-types-operators.md)

## เป้าหมายการเรียนรู้ (Learning Objectives)

หลังจบ Part นี้ ผู้เรียนจะสามารถ:

1. อธิบายได้ว่าภาษา C คืออะไร มีประวัติความเป็นมาอย่างไร และทำไมยังสำคัญมากในปี 2026
2. ติดตั้งเครื่องมือพัฒนา (Toolchain) สำหรับเขียนโปรแกรม C บนระบบปฏิบัติการที่ใช้งานอยู่ได้ด้วยตัวเอง
3. เข้าใจกระบวนการ **Preprocess → Compile → Assemble → Link → Run** ที่อยู่เบื้องหลังทุกครั้งที่กด "รัน" โปรแกรม
4. เขียน คอมไพล์ และรันโปรแกรม C โปรแกรมแรก ("Hello, World!") ได้เองตั้งแต่ต้นจนจบ
5. อ่านและเข้าใจโครงสร้างพื้นฐานของไฟล์ `.c` ทุกส่วน (header, `main`, `return`)
6. ใช้ flag พื้นฐานของคอมไพเลอร์ (`-Wall -Wextra -std=c17 -o`) เพื่อเขียนโค้ดที่ปลอดภัยตั้งแต่บรรทัดแรก
7. ตั้งค่า Editor (VS Code) ให้พร้อมสำหรับการเขียน C ตลอดหลักสูตรนี้

---

## 1.1 ภาษา C คืออะไร และทำไมต้องเรียน (Step 1)

ภาษา C ถูกออกแบบโดย **Dennis Ritchie** ที่ Bell Labs ในช่วงปี 1972 เพื่อใช้เขียนระบบปฏิบัติการ
Unix ใหม่ทั้งหมด (เดิม Unix เขียนด้วย Assembly) จุดเด่นที่ทำให้ C อยู่รอดมาเกือบ 55 ปีและยังคง
เป็นภาษาที่สำคัญที่สุดภาษาหนึ่งของโลกคือ:

- **ใกล้เคียงฮาร์ดแวร์ (Low-level)**: ควบคุมหน่วยความจำได้โดยตรงผ่าน Pointer ทำให้เขียนโปรแกรม
  ที่เร็วและมีประสิทธิภาพสูงสุดได้
- **Portable**: โค้ด C มาตรฐานสามารถคอมไพล์ให้รันบน CPU/OS แทบทุกชนิดในโลก ตั้งแต่ไมโครคอนโทรลเลอร์
  ตัวเล็กๆ ไปจนถึง Supercomputer
- **เป็นรากฐานของภาษาสมัยใหม่**: C++, Java, C#, Go, Rust, JavaScript (V8 Engine เขียนด้วย C++),
  Python (CPython เขียนด้วย C) ล้วนได้รับอิทธิพลทางไวยากรณ์หรือถูกสร้างขึ้นด้วย C/C++
- **ยังคงถูกใช้งานจริงในปี 2026**: Linux Kernel, Git, Redis, SQLite, PostgreSQL (core), ไดรเวอร์
  ฮาร์ดแวร์เกือบทั้งหมด, Firmware ของอุปกรณ์ IoT, ระบบฝังตัวในรถยนต์และเครื่องบิน ล้วนเขียนด้วย C

**ทำไมหลักสูตรนี้เริ่มจาก C ไม่เริ่มจาก C++ เลย?**
เพราะ C++ เป็น superset ที่สร้างต่อยอดจาก C แนวคิดเรื่อง Memory, Pointer, และการทำงานของ
Compiler ที่เรียนใน C จะทำให้เข้าใจ C++ ได้ลึกซึ้งกว่าคนที่กระโดดไปเรียน C++ เลยตั้งแต่แรก
เปรียบเหมือนต้องรู้จักโครงกระดูกก่อนจะเข้าใจกล้ามเนื้อ

### มาตรฐานของภาษา C

C มีการพัฒนามาตรฐาน (Standard) อย่างต่อเนื่องผ่าน ISO/ANSI:

| มาตรฐาน | ปีที่ออก | จุดเด่น |
|---|---|---|
| K&R C | 1978 | ต้นฉบับจากหนังสือของ Kernighan & Ritchie |
| C89 / ANSI C | 1989 | มาตรฐานแรกที่เป็นทางการ |
| C99 | 1999 | เพิ่ม `//` comment, `bool`, VLA, `inline` |
| C11 | 2011 | เพิ่ม Multithreading, Atomic, `_Generic` |
| C17 | 2018 | แก้ไขข้อบกพร่องของ C11 (ไม่มีฟีเจอร์ใหม่) |
| C23 | 2023/2024 | เพิ่ม `nullptr`, `true/false` แบบ keyword, `#embed` |

ตลอดหลักสูตรนี้เราจะใช้มาตรฐาน **C17** เป็นหลัก เพราะเป็นมาตรฐานที่คอมไพเลอร์ทุกตัวรองรับ
เต็มรูปแบบและเสถียรที่สุด และจะพูดถึง C23 เป็นครั้งคราวเมื่อมีฟีเจอร์ที่น่าสนใจ

---

## 1.2 ติดตั้ง Toolchain (Step 2)

"Toolchain" คือชุดเครื่องมือที่แปลงโค้ดต้นฉบับ (Source Code) ให้กลายเป็นโปรแกรมที่รันได้
(Executable) ประกอบด้วยอย่างน้อย 3 ส่วน: **Compiler**, **Linker**, และ **Debugger**

### บน Linux (Ubuntu/Debian) — แนะนำที่สุดสำหรับหลักสูตรนี้

```bash
sudo apt update
sudo apt install build-essential gdb git -y

# ตรวจสอบเวอร์ชัน
gcc --version
gdb --version
```

`build-essential` จะติดตั้ง `gcc` (GNU Compiler Collection), `g++`, `make` และไลบรารีพื้นฐาน
ให้ครบในคำสั่งเดียว

### บน macOS

```bash
xcode-select --install
```

คำสั่งนี้จะติดตั้ง Command Line Tools ของ Apple ซึ่งมี `clang` (compiler หลักของ macOS) มาให้
โดย macOS จะ alias คำสั่ง `gcc` ให้ชี้ไปที่ `clang` โดยอัตโนมัติ

### บน Windows

มี 2 ทางเลือกหลัก:

1. **WSL2 (Windows Subsystem for Linux)** — แนะนำที่สุด เพราะจะได้สภาพแวดล้อม Linux จริง
   ซึ่งตรงกับที่ใช้ในงานจริง (Server ส่วนใหญ่รัน Linux)

   ```powershell
   wsl --install
   ```

   จากนั้นเปิด Ubuntu ผ่าน WSL แล้วติดตั้งตามขั้นตอนของ Linux ด้านบน

2. **MSYS2 / MinGW-w64** — สำหรับผู้ที่ต้องการคอมไพล์โปรแกรม Windows native โดยตรง
   ดาวน์โหลดจาก https://www.msys2.org แล้วรัน:

   ```bash
   pacman -S mingw-w64-x86_64-gcc mingw-w64-x86_64-gdb
   ```

> ตลอดหลักสูตรนี้ ตัวอย่างคำสั่งทั้งหมดจะอ้างอิงสภาพแวดล้อม Linux/WSL เป็นหลัก
> เพราะ Part หลังๆ ที่เกี่ยวกับ Systems Programming และ Web Development
> (Socket, pthread, Docker) จะรันบน Linux เป็นหลักเหมือนกับใน Production จริง

### ติดตั้ง VS Code และ Extension

1. ดาวน์โหลด VS Code จาก https://code.visualstudio.com
2. ติดตั้ง Extension ที่จำเป็น:
   - **C/C++** (จาก Microsoft) — IntelliSense, Debugging
   - **C/C++ Extension Pack**
   - **Code Runner** (ทางเลือก สำหรับรันโค้ดเร็วๆ)
   - **CMake Tools** (จะใช้ตั้งแต่ Module H เป็นต้นไป)

---

## 1.3 กระบวนการเบื้องหลังการคอมไพล์ (Step 3)

สิ่งที่มือใหม่มักเข้าใจผิดคือคิดว่า "คอมไพล์" เป็นขั้นตอนเดียว แต่จริงๆ แล้วมี 4 ขั้นตอนซ่อนอยู่:

```
main.c
   │
   ▼
[1] Preprocessing   →  main.i   (ขยาย #include, #define, ลบ comment)
   │
   ▼
[2] Compilation      →  main.s   (แปลง C เป็น Assembly)
   │
   ▼
[3] Assembly         →  main.o   (แปลง Assembly เป็น Machine Code / Object File)
   │
   ▼
[4] Linking          →  main    (รวม Object File + Library ให้เป็น Executable)
```

เราสามารถสั่งให้ `gcc` หยุดที่แต่ละขั้นตอนเพื่อดูผลลัพธ์จริงได้ ลองสร้างไฟล์ `demo.c`:

```c
#include <stdio.h>

#define GREETING "Hello"

int main(void) {
    printf("%s, World!\n", GREETING);
    return 0;
}
```

**ขั้นตอนที่ 1: Preprocessing** — ดูว่า `#include` และ `#define` ถูกขยายอย่างไร

```bash
gcc -E demo.c -o demo.i
cat demo.i | tail -n 10
```

ผลลัพธ์จะโชว์เนื้อหาทั้งหมดของ `stdio.h` ถูกแปะเข้ามาในไฟล์ (หลายพันบรรทัด!) และ
`GREETING` ถูกแทนที่ด้วย `"Hello"` ตรงๆ (Macro เป็นแค่ Text Substitution ไม่ใช่ตัวแปร)

**ขั้นตอนที่ 2: Compilation** — แปลงเป็น Assembly

```bash
gcc -S demo.c -o demo.s
cat demo.s
```

จะเห็นคำสั่ง Assembly ของ CPU (เช่น `mov`, `call`, `push`) ซึ่งเป็นภาษาที่ใกล้ฮาร์ดแวร์มาก
นี่คือสิ่งที่ทำให้ C เร็ว เพราะมันแปลงตรงไปเป็นคำสั่งระดับ CPU โดยแทบไม่มีชั้น Abstraction คั่น

**ขั้นตอนที่ 3: Assembly → Object File**

```bash
gcc -c demo.c -o demo.o
file demo.o
```

`demo.o` เป็น Binary ที่มี Machine Code แล้ว แต่ยังขาดการเชื่อมโยงกับฟังก์ชัน `printf`
ที่มาจากไลบรารีมาตรฐาน (`libc`) — ยังรันไม่ได้

**ขั้นตอนที่ 4: Linking**

```bash
gcc demo.o -o demo
./demo
# Hello, World!
```

**Linker** จะนำ `demo.o` ไปรวมกับโค้ดของ `printf` ที่อยู่ใน `libc.so` (Dynamic Library ของระบบ)
ให้กลายเป็นไฟล์ Executable ที่รันได้จริง

ในทางปฏิบัติเราไม่จำเป็นต้องแยกทำทีละขั้นตอนแบบนี้ทุกครั้ง คำสั่งเดียว `gcc demo.c -o demo`
จะทำทั้ง 4 ขั้นตอนให้อัตโนมัติ แต่การเข้าใจเบื้องหลังจะช่วยตอนที่เจอ **Error** เพราะ Error
แต่ละประเภทจะเกิดในขั้นตอนที่ต่างกัน:

| ประเภท Error | เกิดในขั้นตอน | ตัวอย่าง |
|---|---|---|
| Preprocessing Error | Preprocess | `#include` ไฟล์ที่ไม่มีอยู่จริง |
| Syntax/Type Error | Compilation | ลืมใส่ `;`, ใช้ตัวแปรผิดชนิด |
| Undefined Reference | Linking | เรียกฟังก์ชันที่ประกาศแต่ไม่ได้ implement |

---

## 1.4 โปรแกรมแรก: Hello, World! (Step 4)

สร้างโฟลเดอร์สำหรับหลักสูตรนี้และไฟล์แรก:

```bash
mkdir -p ~/c_course/part01
cd ~/c_course/part01
touch hello.c
```

เปิดไฟล์ `hello.c` ด้วย VS Code แล้วพิมพ์โค้ดนี้ **ด้วยตัวเอง** (ไม่ copy-paste):

```c
/*
 * hello.c
 * โปรแกรมแรกในหลักสูตร C/C++ ฉบับสมบูรณ์
 */
#include <stdio.h>   // ไลบรารีมาตรฐานสำหรับ Input/Output เช่น printf, scanf

int main(void) {
    printf("Hello, World!\n");
    return 0;
}
```

คอมไพล์และรัน:

```bash
gcc -Wall -Wextra -std=c17 hello.c -o hello
./hello
```

ผลลัพธ์:

```
Hello, World!
```

### อธิบายทีละบรรทัด

- **`#include <stdio.h>`**: บอก Preprocessor ให้แปะเนื้อหาของไฟล์ `stdio.h` (Standard I/O)
  เข้ามาในจุดนี้ เพื่อให้เรามีสิทธิ์ใช้ฟังก์ชัน `printf` ได้ (ประกาศ prototype ไว้ในไฟล์นี้)
- **`int main(void)`**: จุดเริ่มต้นของทุกโปรแกรม C — เมื่อ OS โหลดโปรแกรมขึ้นมารัน จะเรียก
  ฟังก์ชันชื่อ `main` เสมอ คำว่า `int` หมายถึงฟังก์ชันนี้จะคืนค่าเป็นเลขจำนวนเต็มกลับไปให้ OS
  (เรียกว่า **Exit Code**) ส่วน `void` ในวงเล็บหมายถึง "ไม่รับพารามิเตอร์ใดๆ"
- **`printf("Hello, World!\n");`**: เรียกฟังก์ชันพิมพ์ข้อความออกทาง `stdout`
  - `\n` คือ **Escape Sequence** หมายถึงการขึ้นบรรทัดใหม่ (Newline)
  - ทุกคำสั่งใน C จบด้วย `;` (Semicolon) เสมอ — นี่คือกฎที่เข้มงวดที่สุดข้อหนึ่งของภาษา
- **`return 0;`**: ส่งค่า `0` กลับไปให้ระบบปฏิบัติการ ตามธรรมเนียมสากล **`0` แปลว่าโปรแกรม
  ทำงานสำเร็จ** ค่าอื่นที่ไม่ใช่ 0 (เช่น 1, 2, -1) หมายถึงเกิดข้อผิดพลาด

### ตรวจสอบ Exit Code

```bash
./hello
echo $?     # แสดง Exit Code ของโปรแกรมล่าสุดที่รัน — จะได้ 0
```

ลองเปลี่ยนเป็น `return 42;` คอมไพล์ใหม่ แล้วรัน `echo $?` อีกครั้ง จะเห็นว่าได้ `42`
กลไกนี้สำคัญมากในการเขียน Script อัตโนมัติและ CI/CD ในภายหลัง เพราะระบบภายนอกจะตรวจสอบ
ว่าโปรแกรมเราสำเร็จหรือล้มเหลวจากค่านี้

---

## 1.5 Flag สำคัญของ GCC ที่ต้องใช้ตั้งแต่วันแรก (Step 5)

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 -g -O0 hello.c -o hello
```

| Flag | ความหมาย |
|---|---|
| `-Wall` | เปิดคำเตือน (Warning) ชุดพื้นฐานที่สำคัญเกือบทั้งหมด |
| `-Wextra` | เปิดคำเตือนเพิ่มเติมที่ `-Wall` ไม่ครอบคลุม |
| `-Wpedantic` | บังคับตรวจสอบตามมาตรฐานภาษาอย่างเคร่งครัด ไม่ยอมรับ GNU extension |
| `-std=c17` | กำหนดให้คอมไพล์ตามมาตรฐาน C17 |
| `-g` | ใส่ Debug Symbol เพื่อให้ใช้ GDB debug ได้ (จะเรียนใน Part 37) |
| `-O0` | ปิดการ Optimize (Optimization Level 0) เหมาะกับตอนพัฒนา/debug |
| `-o hello` | กำหนดชื่อไฟล์ผลลัพธ์ (ถ้าไม่ใส่ จะได้ `a.out` เป็นค่า default) |

> **กฎทองของหลักสูตรนี้**: ห้ามคอมไพล์โค้ดโดยไม่เปิด `-Wall -Wextra` เด็ดขาด
> Warning ที่ compiler เตือน มักจะบอกบั๊กที่ซ่อนอยู่ล่วงหน้าก่อนที่โปรแกรมจะรันแล้วพัง

ลองทำโค้ดที่มีปัญหาเพื่อดู Warning:

```c
#include <stdio.h>

int main(void) {
    int x;                 // ประกาศแต่ไม่ได้กำหนดค่าเริ่มต้น
    printf("%d\n", x);     // ใช้ค่าที่ยังไม่ถูกกำหนด (Undefined Behavior)
    return 0;
}
```

```bash
gcc -Wall -Wextra -std=c17 uninitialized.c -o uninitialized
```

จะได้ Warning ประมาณนี้:

```
warning: 'x' is used uninitialized in this function [-Wuninitialized]
```

Compiler เตือนล่วงหน้าว่าเรากำลังใช้ค่าขยะ (Garbage Value) จากหน่วยความจำ — นี่คือ
ตัวอย่างของ **Undefined Behavior (UB)** ซึ่งเป็นแนวคิดสำคัญที่จะพูดถึงซ้ำๆ ตลอดหลักสูตรนี้
เพราะเป็นสาเหตุของบั๊กที่แปลกประหลาดและอันตรายที่สุดในภาษา C/C++

---

## 1.6 ตั้งค่า VS Code สำหรับหลักสูตร (Step 6)

สร้างไฟล์ `.vscode/tasks.json` ในโฟลเดอร์โปรเจกต์ เพื่อคอมไพล์ด้วยปุ่ม `Ctrl+Shift+B`:

```json
{
    "version": "2.0.0",
    "tasks": [
        {
            "label": "Build C file",
            "type": "shell",
            "command": "gcc",
            "args": [
                "-Wall", "-Wextra", "-Wpedantic",
                "-std=c17", "-g",
                "${file}",
                "-o", "${fileDirname}/${fileBasenameNoExtension}"
            ],
            "group": {
                "kind": "build",
                "isDefault": true
            }
        }
    ]
}
```

และ `.vscode/launch.json` สำหรับ Debug ด้วย GDB (จะใช้งานเต็มรูปแบบใน Part 37):

```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Debug C file (GDB)",
            "type": "cppdbg",
            "request": "launch",
            "program": "${fileDirname}/${fileBasenameNoExtension}",
            "args": [],
            "cwd": "${workspaceFolder}",
            "MIMode": "gdb",
            "preLaunchTask": "Build C file"
        }
    ]
}
```

---

## 1.7 โครงสร้างมาตรฐานของไฟล์ C ทุกไฟล์ (Step 7)

ตั้งแต่ Part นี้เป็นต้นไป ทุกไฟล์ `.c` ในหลักสูตรจะเขียนตามโครงสร้างมาตรฐานนี้:

```c
/* ============================================================
 * ชื่อไฟล์:     ชื่อไฟล์.c
 * คำอธิบาย:     อธิบายสั้นๆ ว่าโปรแกรมนี้ทำอะไร
 * ผู้เขียน/วันที่: (ถ้าต้องการ)
 * ============================================================ */

/* ---------- 1. Header Includes ---------- */
#include <stdio.h>
#include <stdlib.h>

/* ---------- 2. Macro / Constant Definitions ---------- */
#define MAX_SIZE 100

/* ---------- 3. Function Prototypes (ถ้ามีหลายฟังก์ชัน) ---------- */
void print_banner(void);

/* ---------- 4. main() ---------- */
int main(void) {
    print_banner();
    return 0;
}

/* ---------- 5. Function Implementations ---------- */
void print_banner(void) {
    printf("=== เริ่มต้นหลักสูตร C/C++ ===\n");
}
```

การจัดโครงสร้างแบบนี้ไม่ใช่กฎบังคับของภาษา แต่เป็น **Convention** ที่ทำให้อ่านโค้ดคนอื่น
ได้ง่ายขึ้นมาก และเป็นมาตรฐานที่ใช้ในบริษัทซอฟต์แวร์จริงแทบทุกแห่ง

---

## 1.8 ตั้งค่า Git สำหรับเก็บโค้ดของหลักสูตร (Step 8)

ตลอดหลักสูตรนี้ควร commit โค้ดทุก Part ลง Git repository ของตัวเอง เพื่อฝึกความเคยชิน
แบบมืออาชีพตั้งแต่ต้น:

```bash
cd ~/c_course
git init
cat > .gitignore << 'EOF'
# Compiled binaries
*.o
*.out
*.exe
hello
demo
# ไฟล์ที่ไม่มีนามสกุล ที่เป็นผลลัพธ์การคอมไพล์ (ระวังอย่า ignore โฟลเดอร์สำคัญ)
EOF

git add .
git commit -m "Part 1: Hello World และตั้งค่า Toolchain"
```

> เคล็ดลับ: อย่า commit ไฟล์ Binary ที่คอมไพล์แล้ว (`.o`, executable) เข้า Git เด็ดขาด
> เพราะไฟล์เหล่านี้สร้างใหม่จาก Source Code ได้เสมอ และจะทำให้ repository บวมโดยไม่จำเป็น

---

## ข้อผิดพลาดที่พบบ่อย (Common Pitfalls)

1. **ลืมใส่ `#include <stdio.h>`** — จะได้ Warning "implicit declaration of function 'printf'"
   ในมาตรฐานเก่า หรือ Error ตรงๆ ในคอมไพเลอร์ใหม่บางตัว เพราะ compiler ไม่รู้จัก signature
   ของฟังก์ชัน
2. **ลืม `;` ท้ายคำสั่ง** — เป็น Syntax Error ที่พบบ่อยที่สุดสำหรับมือใหม่ อ่าน error message
   ที่ compiler แจ้งเลขบรรทัดเสมอ (บางครั้ง error จริงอยู่บรรทัดก่อนหน้าที่ compiler ชี้)
3. **สับสนระหว่าง `main()` กับ `main(void)`** — ใน C ทั้งสองแบบต่างกัน! `main()` แปลว่า
   "ไม่ระบุพารามิเตอร์" (ยอมรับพารามิเตอร์อะไรก็ได้แบบไม่ตรวจสอบ) ส่วน `main(void)` แปลว่า
   "ไม่รับพารามิเตอร์ใดๆ เด็ดขาด" ควรใช้ `main(void)` เสมอเพื่อความชัดเจนและปลอดภัย
4. **ไม่เปิด Warning ตอน compile** — ทำให้พลาดบั๊กที่ compiler เตือนไว้ล่วงหน้า
5. **ลืม `return 0;`** — แม้ในทางเทคนิค C99 ขึ้นไปจะ "ยอมให้" ไม่ต้องเขียน `return 0;`
   ท้าย `main` (compiler จะแทรกให้อัตโนมัติ) แต่การเขียนไว้ชัดๆ ทำให้โค้ดอ่านง่ายกว่าเสมอ

---

## แบบฝึกหัดท้ายบท

1. เขียนโปรแกรม `hello_name.c` ที่พิมพ์ข้อความ `"Hello, <ชื่อคุณ>! Welcome to C Programming."`
2. ลองคอมไพล์โปรแกรมในข้อ 1 โดย **ไม่ใส่** `-Wall -Wextra` แล้วลองใส่กลับเข้าไป สังเกตความ
   แตกต่างของ Warning ที่ได้ (ลองสร้างบั๊กเล็กๆ เช่น ตัวแปรไม่ได้ใช้ เพื่อดู Warning)
3. ทดลองคำสั่ง `gcc -E`, `gcc -S`, `gcc -c` กับโปรแกรม `hello.c` แล้วเปิดดูไฟล์ผลลัพธ์แต่ละ
   ขั้นตอน (`.i`, `.s`, `.o`) ด้วยตัวเอง อธิบายด้วยคำพูดตัวเองว่าแต่ละไฟล์ต่างกันอย่างไร
4. เขียนโปรแกรมที่ `return` ค่า Exit Code เป็นเลขวันเกิดของคุณ (เช่นเกิดวันที่ 15 ก็ `return 15;`)
   แล้วตรวจสอบด้วย `echo $?` ว่าตรงกันจริง
5. ตั้งค่า Git repository สำหรับหลักสูตรนี้ และ commit งานของ Part 1 ทั้งหมด

### แนวทางเฉลยข้อ 1

```c
#include <stdio.h>

int main(void) {
    printf("Hello, Somchai! Welcome to C Programming.\n");
    return 0;
}
```

(แทนที่ "Somchai" ด้วยชื่อของผู้เรียนเอง — จุดสำคัญคือต้องคอมไพล์ผ่านและรันได้จริง
พร้อมเปิด `-Wall -Wextra -std=c17` โดยไม่มี Warning ใดๆ เลย)

---

## สรุปท้ายบท

ใน Part นี้เราได้:

- เข้าใจที่มาและความสำคัญของภาษา C ในโลกซอฟต์แวร์ปัจจุบัน
- ติดตั้ง Toolchain (GCC/Clang, GDB, VS Code) พร้อมใช้งานตลอดหลักสูตร
- เข้าใจกระบวนการ Preprocess → Compile → Assemble → Link อย่างละเอียด
- เขียนและรันโปรแกรม C โปรแกรมแรกได้สำเร็จ พร้อมเข้าใจทุกบรรทัดของโค้ด
- รู้จัก Flag สำคัญของ GCC ที่ต้องใช้ตลอดการเรียน (`-Wall -Wextra -std=c17`)
- ตั้งค่า Git เพื่อเก็บผลงานตลอดหลักสูตร

ใน **Part 2** เราจะเจาะลึกเรื่อง **ตัวแปร ชนิดข้อมูล และตัวดำเนินการ** ซึ่งเป็นอิฐก้อนถัดไป
ที่จะนำไปสู่การเขียนโปรแกรมที่ประมวลผลข้อมูลได้จริง

**ต่อไป:** [Part 2 — ตัวแปร ชนิดข้อมูล และตัวดำเนินการ](./part-002-variables-types-operators.md)
