# Part 26: พื้นฐานระบบปฏิบัติการสำหรับโปรแกรมเมอร์ (Step 201–208)

> Module C — Systems Programming ด้วย C บน Linux | Part 26 จาก 125
> Step ที่ครอบคลุมใน Part นี้: Step 201–208
> Part ก่อนหน้า: [Part 25 — Recursion ขั้นสูงและ Dynamic Programming](./part-025-recursion-dp.md) | Part ถัดไป: [Part 27 — Process Management (fork/exec/wait)](./part-027-process-management.md)

## เป้าหมายการเรียนรู้ (Learning Objectives)

หลังจบ Part นี้ ผู้เรียนจะสามารถ:

1. อธิบายความแตกต่างระหว่าง **Program** (ไฟล์ executable ที่นิ่งอยู่บนดิสก์) กับ **Process**
   (การทำงานจริงของโปรแกรมนั้นในหน่วยความจำ) ได้อย่างชัดเจน พร้อมยกตัวอย่างจริง
2. วาดและอธิบาย **Memory Layout ของ Process** บน Linux ได้ครบทุก Segment (Text, Data, BSS,
   Heap, Stack) พร้อมบอกได้ว่าตัวแปรแบบไหนไปอยู่ Segment ไหน
3. อธิบายแนวคิด **Virtual Memory** ได้ว่าทำไม Process แต่ละตัวถึง "เข้าใจผิด" คิดว่าตัวเอง
   เป็นเจ้าของหน่วยความจำทั้งหมดในเครื่อง ทั้งที่ RAM จริงถูกแบ่งกันใช้กับ Process อื่นอีกมากมาย
4. แยกแยะ **Kernel Space** กับ **User Space** ได้ และอธิบายได้ว่าทำไมต้องแยกสองพื้นที่นี้ออก
   จากกันอย่างเข้มงวด
5. อธิบายว่า **System Call** คืออะไร และตามรอยได้ว่าเบื้องหลังการเรียก `printf()` ธรรมดาๆ
   สุดท้ายไปจบที่ System Call ตัวไหนของ Kernel
6. อธิบายแนวคิด **Context Switching** ได้ พร้อมแยกแยะระหว่าง Voluntary กับ Involuntary
   Context Switch และวัดค่าจริงจากโปรแกรม C ได้ด้วยตัวเอง
7. ใช้เครื่องมือสำรวจ Process บน Linux ได้จริง ทั้ง `ps`, `top` และการอ่านข้อมูลจาก
   **`/proc` filesystem** โดยตรงจากโปรแกรม C ของตัวเอง

---

## บทนำ: ทำไมโปรแกรมเมอร์ C ต้องเข้าใจ OS

ตลอด 25 Part ที่ผ่านมา เราเขียนโปรแกรม C ที่ทำงาน "ในกล่องของตัวเอง" — รับ input, ประมวลผล,
พิมพ์ output แล้วจบ ไม่เคยต้องสนใจว่าเบื้องหลังโปรแกรมของเรากำลังทำงานอยู่บนอะไร ใครดูแล
หน่วยความจำให้ หรือทำไมโปรแกรมหลายตัวถึงรันพร้อมกันได้โดยไม่ชนกัน

ตั้งแต่ Part นี้เป็นต้นไป เราจะเจาะลึกเข้าไปใน **Linux Kernel** ที่อยู่เบื้องหลังทุกโปรแกรม
C ที่เราเขียน เพราะ **Systems Programming** (การเขียนโปรแกรมที่คุยกับ OS โดยตรง เช่น
จัดการ Process, Signal, Thread, Socket) คือจุดแข็งที่สุดของภาษา C และเป็นทักษะที่แยก
โปรแกรมเมอร์ทั่วไปออกจากวิศวกรระดับ Senior ที่เข้าใจว่าโปรแกรมของตัวเอง "อยู่ตรงไหน" ใน
เครื่องจริงๆ

ก่อนจะลงมือเขียน `fork()`, `exec()`, `signal()` ใน Part ถัดๆ ไป เราต้องปูพื้นฐาน 4 เรื่องนี้
ให้แน่นก่อน: **Process คืออะไร, หน่วยความจำของ Process จัดวางอย่างไร, Virtual Memory ทำงาน
อย่างไร, และโปรแกรมคุยกับ Kernel ผ่านอะไร**

---

## 26.1 Process คืออะไร เปรียบเทียบกับ Program (Step 201)

### Program vs Process

**Program** คือไฟล์ที่นิ่งอยู่บนดิสก์ (เช่นไฟล์ `hello` ที่ได้จากการคอมไพล์ `hello.c` ด้วย
`gcc`) เป็นแค่ชุดของ Byte ที่เก็บ Machine Code, ข้อมูล และ Metadata ไว้เฉยๆ ไม่ได้ "ทำงาน"
อะไรเลยจนกว่าจะถูกสั่งรัน

**Process** คือ**ตัวตนที่ Kernel สร้างขึ้น**เมื่อโปรแกรมถูกโหลดขึ้นมารันจริง Process มี
ทรัพยากรของตัวเอง (Memory Space, File Descriptor, Register ของ CPU ณ ขณะนั้น) และมี
"ชีวิต" ตั้งแต่เกิด (ถูก `fork()`/`exec()` โดย Process อื่น) จนตาย (เรียก `exit()` หรือถูก
Signal ฆ่า)

| ประเด็น | Program | Process |
|---|---|---|
| สถานะ | นิ่ง (Static) อยู่บน Disk | กำลังทำงาน (Dynamic) อยู่ใน RAM |
| จำนวน | มี 1 ชุด (ไฟล์เดียว) | รันพร้อมกันได้หลาย Process จากไฟล์เดียวกัน |
| ทรัพยากร | ไม่มี (แค่ไฟล์) | มี Memory Space, PID, File Descriptor เป็นของตัวเอง |
| ตัวอย่าง | ไฟล์ `/usr/bin/firefox` | หน้าต่าง Firefox 3 หน้าต่างที่เปิดพร้อมกัน = 3 Process (หรือมากกว่านั้นถ้านับ process ย่อยของแต่ละ tab) |

จุดสำคัญที่สุดคือ **โปรแกรมเดียวกันสามารถกลายเป็นหลาย Process พร้อมกันได้** โดยแต่ละ
Process จะมีพื้นที่หน่วยความจำของตัวเองแยกจากกันโดยสิ้นเชิง แก้ไขตัวแปรใน Process หนึ่ง
จะไม่กระทบอีก Process หนึ่งเลย แม้จะรันมาจากไฟล์ executable เดียวกันเป๊ะก็ตาม

### แต่ละ Process มี "บัตรประชาชน" ของตัวเอง: PID และ PPID

Kernel ระบุตัวตนของแต่ละ Process ด้วยเลขที่เรียกว่า **PID (Process ID)** ซึ่งไม่ซ้ำกันเลย
ในบรรดา Process ที่กำลังทำงานอยู่ ณ ขณะนั้น (แต่ **PID สามารถถูกนำกลับมาใช้ซ้ำได้** หลัง
Process เดิมตายไปแล้ว — ดูรายละเอียดในหัวข้อ Common Pitfalls) และทุก Process (ยกเว้น
Process แรกสุดของระบบ) จะมี **PPID (Parent Process ID)** บอกว่าใครเป็นคนสร้างตัวเองขึ้นมา

```c
/* ============================================================
 * ชื่อไฟล์:     process_info.c
 * คำอธิบาย:     แสดง PID และ PPID ของโปรเซสตัวเอง เปรียบเทียบ Process กับ Program
 * ============================================================ */
#define _POSIX_C_SOURCE 200809L
#include <stdio.h>
#include <unistd.h>

int main(void) {
    pid_t my_pid = getpid();
    pid_t parent_pid = getppid();

    printf("โปรแกรมนี้กำลังรันเป็น Process หมายเลข (PID) = %ld\n", (long)my_pid);
    printf("ถูกสร้างขึ้นโดย Process แม่ (PPID)          = %ld\n", (long)parent_pid);

    return 0;
}
```

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 -D_POSIX_C_SOURCE=200809L process_info.c -o process_info
./process_info
./process_info
```

```
โปรแกรมนี้กำลังรันเป็น Process หมายเลข (PID) = 2577
ถูกสร้างขึ้นโดย Process แม่ (PPID)          = 2569
โปรแกรมนี้กำลังรันเป็น Process หมายเลข (PID) = 2578
ถูกสร้างขึ้นโดย Process แม่ (PPID)          = 2569
```

สังเกตว่ารันโปรแกรม**เดียวกัน**สองครั้งติดกัน ได้ PID ที่**ต่างกัน** (2577, 2578) เพราะแต่
ละครั้งที่รัน shell จะสั่ง Kernel สร้าง Process **ใหม่**ขึ้นมาโดยเฉพาะ แต่ PPID เท่ากันเพราะ
ถูกสร้างโดย shell (bash) ตัวเดียวกัน — นี่คือหลักฐานที่จับต้องได้ว่า "Program" (ไฟล์
`process_info`) กับ "Process" (สิ่งที่มี PID) เป็นคนละสิ่งกัน

**Type `pid_t`**: สังเกตว่าเราไม่ใช้ `int` ตรงๆ กับ PID แต่ใช้ `pid_t` ซึ่งเป็น Type ที่
มาตรฐาน POSIX กำหนดไว้เฉพาะสำหรับเก็บ PID (ในทางปฏิบัติมักเป็น `int` แต่การใช้ `pid_t`
ทำให้โค้ด Portable และสื่อความหมายชัดเจนกว่า) `#define _POSIX_C_SOURCE 200809L` ที่ใส่
ไว้บนสุดของไฟล์ (ก่อน `#include` ทุกตัว) คือการบอก `glibc` ว่าเราต้องการใช้ฟีเจอร์ตาม
มาตรฐาน POSIX.1-2008 ซึ่งจำเป็นสำหรับฟังก์ชันอย่าง `getpid()`, `fork()`, `sigaction()` ที่
เราจะใช้ตลอด Module C นี้

### ข้อมูลที่ Kernel เก็บไว้เกี่ยวกับแต่ละ Process

ในเชิงแนวคิด Kernel เก็บข้อมูลของแต่ละ Process ไว้ในโครงสร้างที่เรียกว่า **PCB (Process
Control Block)** ซึ่งมีข้อมูลสำคัญเหล่านี้ (ไม่ต้องท่องจำ แต่ควรรู้ว่ามีอะไรบ้าง):

| ข้อมูลใน PCB | ความหมาย |
|---|---|
| PID / PPID | หมายเลขประจำตัวและ PID ของพ่อ |
| State | สถานะปัจจุบัน (Running, Sleeping, Zombie, ...) — ดูหัวข้อ 26.7 |
| Program Counter | Instruction ล่าสุดที่กำลังจะรัน |
| CPU Registers | ค่าใน Register ทั้งหมด ณ ขณะที่ถูกสลับออกจาก CPU |
| Memory Management Info | Page Table ที่แปลง Virtual Address เป็น Physical Address (หัวข้อ 26.3) |
| Open File Descriptors | รายการไฟล์/socket ที่เปิดอยู่ |
| Scheduling Info | Priority, เวลาที่ใช้ CPU ไปแล้ว |

ข้อมูลนี้คือสิ่งที่ Kernel ใช้ในการทำ **Context Switching** (หัวข้อ 26.6) — เมื่อสลับจาก
Process หนึ่งไปอีก Process หนึ่ง Kernel ต้องบันทึกค่าทั้งหมดนี้ของ Process เดิมเก็บไว้ก่อน
แล้วค่อยโหลดค่าของ Process ใหม่กลับเข้า CPU

---

## 26.2 Memory Layout ของ Process (Step 202)

เมื่อ Process หนึ่งถูกสร้างขึ้น Kernel จะจัดสรรพื้นที่หน่วยความจำ (Virtual Address Space)
ให้เป็นส่วนๆ ที่เรียกว่า **Segment** แต่ละ Segment มีหน้าที่และกฎการเติบโตต่างกัน ต่อไปนี้
คือ Memory Layout แบบมาตรฐานของ Process บน Linux (x86-64):

```
ที่อยู่สูง (High Address)
┌─────────────────────────────┐
│   Command-line Args & Env   │   argv[], environ[]
├─────────────────────────────┤
│                             │
│           Stack             │   ตัวแปร local, return address
│             │               │   โตจาก "บนลงล่าง" (High -> Low)
│             ▼               │
│                             │
│      (พื้นที่ว่างตรงกลาง)      │
│                             │
│             ▲               │
│             │               │
│           Heap              │   malloc/calloc/realloc
│                             │   โตจาก "ล่างขึ้นบน" (Low -> High)
├─────────────────────────────┤
│    BSS (Uninitialized Data) │   global/static ที่ไม่มีค่าเริ่มต้น
├─────────────────────────────┤
│    Data (Initialized Data)  │   global/static ที่มีค่าเริ่มต้น
├─────────────────────────────┤
│    Text / Code Segment      │   Machine Code ของโปรแกรม (read-only)
└─────────────────────────────┘
ที่อยู่ต่ำ (Low Address)
```

### รายละเอียดแต่ละ Segment

| Segment | เก็บอะไร | ตัวอย่าง | ทิศทางการโต |
|---|---|---|---|
| **Text (Code)** | Machine Instruction ของโปรแกรม | โค้ดของทุกฟังก์ชันที่เราเขียน | คงที่ (Read-only, บาง Compiler ทำเป็น Executable แต่ห้ามเขียน) |
| **Data** | Global/Static variable ที่**มี**ค่าเริ่มต้นไม่เป็นศูนย์ | `int global_initialized = 42;` | คงที่ (กำหนดตอน Compile) |
| **BSS** (Block Started by Symbol) | Global/Static variable ที่**ไม่มี**ค่าเริ่มต้น (หรือเริ่มต้นเป็น 0) | `int global_uninitialized;` | คงที่ (แต่ Kernel เติม 0 ให้อัตโนมัติตอนโหลดโปรแกรม ไม่ต้องเก็บค่าจริงในไฟล์ executable เลย ประหยัดพื้นที่ไฟล์) |
| **Heap** | หน่วยความจำที่จองแบบพลวัต | ผลลัพธ์จาก `malloc()`, `calloc()`, `realloc()` | โตขึ้น (Low → High) เมื่อจองเพิ่ม |
| **Stack** | ตัวแปร local, พารามิเตอร์ฟังก์ชัน, Return Address | `int x = 5;` ใน `main()` | โตลง (High → Low) ทุกครั้งที่เรียกฟังก์ชัน |

### พิสูจน์ด้วยโค้ดจริง

```c
/* ============================================================
 * ชื่อไฟล์:     memory_layout.c
 * คำอธิบาย:     พิมพ์ที่อยู่ของตัวแปรแต่ละ segment เพื่อดู Memory Layout ของ Process
 * ============================================================ */
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <stdint.h>

/* ---------- global ที่มีค่าเริ่มต้น -> อยู่ใน Data Segment ---------- */
int global_initialized = 42;

/* ---------- global ที่ไม่มีค่าเริ่มต้น -> อยู่ใน BSS Segment ---------- */
int global_uninitialized;

void print_banner(void); /* ฟังก์ชัน -> เก็บอยู่ใน Text Segment */

/* หา address ของฟังก์ชันแบบ portable ตามมาตรฐาน C (ไม่แปลง function pointer
   เป็น object pointer ตรงๆ ซึ่ง ISO C ห้าม แต่ใช้ memcpy คัด bit pattern แทน) */
static uintptr_t function_address(void (*fp)(void)) {
    uintptr_t addr;
    memcpy(&addr, &fp, sizeof(addr));
    return addr;
}

int main(void) {
    /* ตัวแปร local ธรรมดา -> อยู่บน Stack Segment */
    int stack_variable = 7;

    /* หน่วยความจำที่จองแบบพลวัต -> อยู่บน Heap Segment */
    int *heap_variable = malloc(sizeof(int));
    if (heap_variable == NULL) {
        fprintf(stderr, "malloc ล้มเหลว\n");
        return 1;
    }
    *heap_variable = 99;

    printf("Text  (โค้ดฟังก์ชัน)         : 0x%-14lx (print_banner)\n",
           (unsigned long)function_address(print_banner));
    printf("Data  (global มีค่าเริ่มต้น) : %-16p (global_initialized)\n",
           (void *)&global_initialized);
    printf("BSS   (global ไม่มีค่าเริ่มต้น): %-16p (global_uninitialized)\n",
           (void *)&global_uninitialized);
    printf("Heap  (malloc)               : %-16p (heap_variable)\n",
           (void *)heap_variable);
    printf("Stack (local variable)       : %-16p (stack_variable)\n",
           (void *)&stack_variable);

    free(heap_variable);
    return 0;
}

void print_banner(void) {
    printf("=== Memory Layout Demo ===\n");
}
```

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 memory_layout.c -o memory_layout
./memory_layout
```

```
Text  (โค้ดฟังก์ชัน)         : 0x560bdc0b8357   (print_banner)
Data  (global มีค่าเริ่มต้น) : 0x560bdc0bb010   (global_initialized)
BSS   (global ไม่มีค่าเริ่มต้น): 0x560bdc0bb02c   (global_uninitialized)
Heap  (malloc)               : 0x560bfc3422a0   (heap_variable)
Stack (local variable)       : 0x7ffc8fab94ec   (stack_variable)
```

สังเกตลำดับที่อยู่จากน้อยไปมาก: **Text < Data < BSS < Heap < ... < Stack** ตรงตามแผนภาพ
ทุกประการ ที่อยู่ของ Text/Data/BSS อยู่ในช่วงหลักหมื่นล้าน (`0x560b...`) ใกล้เคียงกัน เพราะ
เป็นส่วนที่ถูกโหลดมาจากไฟล์ executable ตอนเริ่มโปรแกรม ส่วน Heap (`0x560bfc...`) อยู่ห่าง
ออกไปมาก และ Stack (`0x7ffc...`) อยู่ไกลที่สุด เพราะปกติ Kernel วาง Stack ไว้บริเวณเกือบ
บนสุดของ Virtual Address Space ของ Process

> **ทำไมต้องใช้ `memcpy` แทนการ cast function pointer เป็น `void *` ตรงๆ?**
> มาตรฐาน ISO C (`-Wpedantic`) ห้ามแปลง Function Pointer เป็น Object Pointer โดยตรง เพราะ
> ในทางทฤษฎีบางสถาปัตยกรรมอาจมีขนาดหรือการเข้ารหัสต่างกัน แม้ในทางปฏิบัติบน Linux/x86
> ทั้งสองจะมีขนาดเท่ากันเสมอ (POSIX รับประกันเรื่องนี้ไว้ด้วยซ้ำ) แต่การเขียนโค้ดที่คอมไพล์
> ผ่าน `-Wpedantic` โดยไม่มี Warning เลยคือมาตรฐานของหลักสูตรนี้ การใช้ `memcpy()` คัดลอก
> Bit Pattern ดิบๆ จึงเป็นทางออกที่ทั้ง Portable และไม่มี Warning

### ทำไมต้องแยก Data กับ BSS ทั้งที่ทั้งคู่เป็น Global Variable เหมือนกัน

เหตุผลคือ **ประหยัดพื้นที่ในไฟล์ Executable** ตัวแปรใน BSS ทุกตัวมีค่าเริ่มต้นเป็น 0 เสมอ
(ตามมาตรฐาน C) ดังนั้น Compiler/Linker ไม่จำเป็นต้องเก็บค่า 0 นับพันนับหมื่นตัวไว้ในไฟล์จริง
แค่บันทึกไว้ว่า "ตอนโหลดโปรแกรม ให้ Kernel จองพื้นที่ขนาดเท่านี้แล้วเติม 0 ให้ทั้งหมด" ก็พอ
ในขณะที่ตัวแปรใน Data Segment มีค่าเริ่มต้นที่ไม่ใช่ 0 จึงต้องเก็บค่าจริงไว้ในไฟล์ executable
ด้วย ลองทดสอบเทียบขนาดไฟล์ได้ด้วยตัวเอง:

```bash
# โปรแกรมที่มี array ขนาดใหญ่แบบไม่กำหนดค่าเริ่มต้น (ไป BSS)
echo 'int big_array[1000000]; int main(void) { return 0; }' > bss_test.c
gcc -std=c17 bss_test.c -o bss_test
ls -la bss_test          # ขนาดไฟล์เล็ก แม้ตัวแปรจะใหญ่ถึง ~4MB ตอนรัน

# โปรแกรมที่มี array ขนาดเท่ากันแต่กำหนดค่าเริ่มต้นทุกตัวเป็น 1 (ไป Data)
echo 'int big_array[1000000] = {[0 ... 999999] = 1}; int main(void) { return 0; }' > data_test.c
gcc -std=gnu17 data_test.c -o data_test
ls -la data_test          # ขนาดไฟล์ใหญ่กว่ามาก เพราะต้องเก็บค่า 1 ทุกตัวจริงๆ ในไฟล์
```

---

## 26.3 Virtual Memory เบื้องต้น (Step 203)

### ทำไม Process แต่ละตัวถึง "คิดว่า" ตัวเองเป็นเจ้าของ RAM ทั้งหมด

ถ้าเรารันโปรแกรม `memory_layout` ข้างต้นพร้อมกันหลายๆ ตัว แต่ละตัวจะรายงานที่อยู่ของ
`stack_variable` ออกมาในช่วงตัวเลขที่ใกล้เคียงกันมาก (เช่น `0x7ffc...` เสมอ) ทั้งที่ RAM จริง
มีขนาดจำกัดและถูกแบ่งกันใช้ระหว่างหลาย Process พร้อมกัน คำถามคือ **เป็นไปได้อย่างไรที่หลาย
Process จะมี "ที่อยู่" ตัวแปรอยู่ในช่วงเดียวกันได้โดยไม่ทับกัน?**

คำตอบคือที่อยู่ที่เราเห็นทั้งหมด (จาก `memory_layout.c`, `vmem_demo.c` ในหัวข้อนี้) เป็น
**Virtual Address (ที่อยู่เสมือน)** ไม่ใช่ **Physical Address (ที่อยู่จริงบนแผง RAM)**
Kernel ร่วมกับฮาร์ดแวร์ที่เรียกว่า **MMU (Memory Management Unit)** จะคอยแปล Virtual
Address ของแต่ละ Process ให้กลายเป็น Physical Address จริงที่ต่างกันไปเบื้องหลัง โดย
โปรแกรมของเราไม่มีทางรู้เลยว่าค่า "จริง" ทางฟิสิกส์คืออะไร

```
Process A (Virtual)              Process B (Virtual)
┌───────────────────┐            ┌───────────────────┐
│ 0x7ffc.... (stack) │            │ 0x7ffc.... (stack) │   <- เหมือนกันได้! เพราะเป็นแค่ Virtual
└─────────┬──────────┘            └─────────┬──────────┘
          │   MMU + Page Table แปลที่อยู่     │
          ▼                                  ▼
┌─────────────────────────────────────────────────────┐
│              Physical RAM (มีจำกัด แบ่งกันใช้)          │
│   Frame #4821 (ของ A)     Frame #9103 (ของ B)         │
└─────────────────────────────────────────────────────┘
```

การแปลที่อยู่นี้ใช้กลไกที่เรียกว่า **Paging**: หน่วยความจำถูกแบ่งเป็นบล็อกเล็กๆ ขนาดเท่ากัน
เรียกว่า **Page** (โดยทั่วไปขนาด 4KB บน x86-64) แต่ละ Process มี **Page Table** ของตัวเอง
ที่บอกว่า Virtual Page แต่ละอันแปลไปเป็น Physical Frame อันไหน

### ประโยชน์ของ Virtual Memory

1. **Isolation (แยกจากกัน)**: Process A ไม่สามารถอ่าน/เขียนหน่วยความจำของ Process B ได้
   โดยตรงเลย แม้จะรู้ค่า Virtual Address ของอีกฝั่งก็ตาม เพราะ Page Table ของแต่ละ Process
   แยกกันสิ้นเชิง — นี่คือรากฐานความปลอดภัยที่สำคัญที่สุดอย่างหนึ่งของ OS สมัยใหม่
2. **Simplicity (เขียนโปรแกรมง่ายขึ้น)**: โปรแกรมเมอร์ไม่ต้องกังวลว่า RAM เครื่องนี้เหลือ
   เท่าไหร่ หรือ Process อื่นใช้ที่อยู่ไหนไปแล้วบ้าง เพราะแต่ละ Process เห็น Address Space
   เป็นของตัวเองเต็มรูปแบบ (บน x86-64 คือ Virtual Address Space ขนาดใหญ่มาก แม้ RAM จริง
   จะมีน้อยกว่านั้นมหาศาล)
3. **Memory Overcommit และ Swap**: Kernel สามารถ "โกหก" ว่ามีหน่วยความจำเหลือเฟือ โดยแอบ
   ย้ายข้อมูลที่ไม่ได้ใช้บ่อยไปเก็บใน Disk (Swap Space) ชั่วคราว แล้วดึงกลับมาเมื่อ Process
   ต้องการใช้จริง (Page Fault กระตุ้นให้ Kernel โหลดกลับเข้า RAM)

### พิสูจน์ด้วยโค้ดและ `/proc/self/maps`

```c
/* ============================================================
 * ชื่อไฟล์:     vmem_demo.c
 * คำอธิบาย:     แสดง Virtual Address ของตัวแปร Stack และเปิด /proc/self/maps
 *              เพื่อดู Memory Mapping จริงของ Process ตัวเอง
 * ============================================================ */
#define _POSIX_C_SOURCE 200809L
#include <stdio.h>
#include <unistd.h>

int main(void) {
    int stack_variable = 123;

    printf("Virtual address ของ stack_variable ใน process นี้ (PID=%ld) คือ %p\n",
           (long)getpid(), (void *)&stack_variable);
    printf("ที่อยู่นี้เป็น \"Virtual Address\" ไม่ใช่ตำแหน่งจริงบนแผง RAM\n");
    printf("ลองรัน `cat /proc/%ld/maps` ในอีก terminal เพื่อดู mapping จริงของ process นี้\n",
           (long)getpid());

    return 0;
}
```

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 -D_POSIX_C_SOURCE=200809L vmem_demo.c -o vmem_demo
./vmem_demo
```

```
Virtual address ของ stack_variable ใน process นี้ (PID=2814) คือ 0x7ffe95d44e24
ที่อยู่นี้เป็น "Virtual Address" ไม่ใช่ตำแหน่งจริงบนแผง RAM
ลองรัน `cat /proc/2814/maps` ในอีก terminal เพื่อดู mapping จริงของ process นี้
```

`/proc/[pid]/maps` คือไฟล์พิเศษที่ Kernel สร้างขึ้นมาให้ (ไม่ใช่ไฟล์จริงบนดิสก์ — ดูหัวข้อ
26.8) แสดงรายการ Memory Mapping ทั้งหมดของ Process นั้นๆ:

```bash
# รันโปรแกรมที่ค้างไว้สักพัก แล้วดู maps ของมันจากอีก terminal
cat /proc/self/maps | head -8
```

```
561cf0fa9000-561cf0fab000 r--p 00000000 fe:00 151456   /usr/bin/cat
561cf0fab000-561cf0fb0000 r-xp 00002000 fe:00 151456   /usr/bin/cat
561cf0fb0000-561cf0fb2000 r--p 00007000 fe:00 151456   /usr/bin/cat
561cf0fb2000-561cf0fb3000 r--p 00008000 fe:00 151456   /usr/bin/cat
561cf0fb3000-561cf0fb4000 rw-p 00009000 fe:00 151456   /usr/bin/cat
561d05120000-561d05141000 rw-p 00000000 00:00 0        [heap]
7f0deda00000-7f0deda28000 r--p 00000000 fe:00 152035   /usr/lib/x86_64-linux-gnu/libc.so.6
7f0deda28000-7f0dedbb0000 r-xp 00028000 fe:00 152035   /usr/lib/x86_64-linux-gnu/libc.so.6
```

แต่ละบรรทัดคือ 1 ช่วง Virtual Address ที่ Map อยู่กับบางสิ่ง (ไฟล์ executable, library, heap,
stack) รูปแบบคือ `start-end permission offset device inode path` โดย `permission` บอกว่า
`r`=read, `w`=write, `x`=execute, `p`=private (Copy-on-Write ถ้าใช้ร่วมกับ process อื่นแล้ว
มีการเขียน — จะเรียนละเอียดใน Part 27) ลองรันโปรแกรมที่ค้างไว้ 2 วินาทีแล้วกรองเฉพาะแถว
`heap` กับ `stack`:

```bash
./vmem_sleep &        # โปรแกรมที่ sleep(2) หลังพิมพ์ PID/address
PID=$!
sleep 1
cat /proc/$PID/maps | grep -E "heap|stack"
```

```
56298a415000-56298a436000 rw-p 00000000 00:00 0   [heap]
7fffe4a65000-7fffe4a87000 rw-p 00000000 00:00 0   [stack]
```

สังเกตว่าบรรทัด `[heap]` และ `[stack]` ไม่มี `path` ไปยังไฟล์ใดเลย (เป็น `00:00 0`) เพราะ
เป็นหน่วยความจำที่สร้างขึ้นมาสดๆ ตอนรัน ไม่ได้มาจากไฟล์บนดิสก์เหมือน Text/Data Segment

### เกร็ดความรู้: ASLR (Address Space Layout Randomization)

หากลองคอมไพล์และรัน `memory_layout.c` ซ้ำหลายครั้ง จะสังเกตว่าค่า Address ที่ได้**ไม่เท่า
เดิม**ในแต่ละครั้งที่รัน (แม้จะเป็นโปรแกรมเดียวกัน) นี่คือฟีเจอร์ความปลอดภัยของ Linux
ที่เรียกว่า **ASLR** — Kernel จะสุ่มตำแหน่งเริ่มต้นของ Stack, Heap และ Shared Library ทุก
ครั้งที่โหลดโปรแกรม เพื่อป้องกันการโจมตีที่อาศัยการเดา Address ล่วงหน้า (เช่น Buffer
Overflow Exploit ที่ต้องรู้ Address ที่แน่นอนถึงจะโจมตีสำเร็จ — จะเรียนเชิงลึกใน Part 115)

---

## 26.4 Kernel Space vs User Space (Step 204)

### ทำไมต้องแบ่งพื้นที่

CPU สมัยใหม่รองรับ **Privilege Level** (หรือเรียกว่า **Ring** บนสถาปัตยกรรม x86) หลายระดับ
Linux ใช้แค่ 2 ระดับหลักคือ:

```
┌─────────────────────────────────────────────┐
│              User Space (Ring 3)             │
│  - โปรแกรมทั่วไปทั้งหมด: your_app, firefox,   │
│    gcc, python, bash, ...                    │
│  - เข้าถึงฮาร์ดแวร์โดยตรง "ไม่ได้"              │
│  - เขียนหน่วยความจำของ process อื่น "ไม่ได้"    │
└──────────────────┬────────────────────────────┘
                    │  System Call (ประตูเดียวที่เชื่อมสองฝั่ง)
                    ▼
┌─────────────────────────────────────────────┐
│             Kernel Space (Ring 0)            │
│  - ตัว Linux Kernel เอง                       │
│  - Device Driver                             │
│  - เข้าถึงฮาร์ดแวร์ได้เต็มรูปแบบ                │
│  - ควบคุม Memory, CPU Scheduling, ไฟล์ระบบ    │
└─────────────────────────────────────────────┘
```

| ประเด็น | User Space | Kernel Space |
|---|---|---|
| Privilege | ต่ำ (Ring 3) — ถูกจำกัดสิทธิ์ | สูงสุด (Ring 0) — ทำอะไรก็ได้กับฮาร์ดแวร์ |
| ใครอยู่ตรงนี้ | โปรแกรมแอปพลิเคชันทั่วไปทั้งหมด รวมถึงโปรแกรม C ที่เราเขียน | Kernel, Device Driver |
| เข้าถึงฮาร์ดแวร์โดยตรง | ทำไม่ได้ | ทำได้ |
| เข้าถึงหน่วยความจำของ process อื่น | ทำไม่ได้ (ถูก MMU/Kernel กัน) | ทำได้ (ควบคุมทุก Page Table) |
| ถ้าเกิด bug/crash | Process นั้นล่มไปตัวเดียว (`Segmentation Fault`) ระบบยังทำงานต่อได้ | ทั้งระบบอาจล่ม (Kernel Panic) |

### ทำไมต้องแยกอย่างเข้มงวด

ลองจินตนาการว่าถ้าโปรแกรม User Space ธรรมดาสามารถเขียนข้อมูลลง Disk Controller โดยตรง
โดยไม่ผ่าน Kernel ได้ — บั๊กเล็กๆ ในโปรแกรม Text Editor อาจเขียนทับข้อมูลของทั้งเครื่องได้
เลยโดยไม่มีใครห้าม การบังคับให้ทุกการเข้าถึงฮาร์ดแวร์ต้อง "ขอผ่าน Kernel" เท่านั้น ทำให้
Kernel สามารถตรวจสอบสิทธิ์ ป้องกันการชนกัน และรักษาเสถียรภาพของทั้งระบบไว้ได้

**คำถามคือ ถ้าโปรแกรม User Space ทำอะไรกับฮาร์ดแวร์โดยตรงไม่ได้เลย แล้วเวลาโปรแกรมเรา
เรียก `printf()` เพื่อพิมพ์ข้อความออกหน้าจอ มันไปถึงหน้าจอได้อย่างไร?** คำตอบคือผ่านกลไก
ที่เรียกว่า **System Call** ซึ่งเป็นหัวข้อถัดไป

---

## 26.5 System Call คืออะไร (Step 205)

**System Call** คือ "ประตู" ที่เป็นทางเดียวที่โปรแกรมใน User Space จะขอให้ Kernel ทำงานบาง
อย่างแทนตัวเองได้ (เช่น อ่าน/เขียนไฟล์, จองหน่วยความจำเพิ่ม, สร้าง Process ใหม่, ส่งข้อมูล
ผ่านเครือข่าย) เบื้องหลังการเรียก System Call คือคำสั่งพิเศษของ CPU ที่จะ "สลับโหมด" จาก
User Mode (Ring 3) ไปเป็น Kernel Mode (Ring 0) ชั่วคราว ให้ Kernel เข้ามาทำงานที่ขอไว้ให้
เสร็จ แล้วค่อยสลับกลับมา User Mode พร้อมผลลัพธ์

```
User Space                    Kernel Space
─────────────                 ──────────────
write(1, "hi", 2)
    │
    │  (1) เรียก syscall instruction
    ▼
                    ──────►    (2) CPU สลับเป็น Kernel Mode
                                (3) Kernel ตรวจสอบสิทธิ์ ทำงานให้
                                (4) เขียนข้อมูลจริงไปที่ device
    ▲
    │  (5) CPU สลับกลับ User Mode พร้อมผลลัพธ์
    │
โค้ดถัดไปทำงานต่อ
```

### `printf()` vs `write()`: ใครคุยกับ Kernel จริงๆ

`printf()` เป็นฟังก์ชันของ **Standard Library (libc)** ที่ทำงานอยู่ใน **User Space**
ทั้งหมด — มันไม่ใช่ System Call! สิ่งที่ `printf()` ทำคือจัดรูปแบบข้อความ (แปลง `%d`,
`%s` ฯลฯ) แล้วเก็บผลลัพธ์ไว้ใน **Buffer ภายใน (User Space Buffer)** ก่อน และจะเรียก
System Call `write()` ให้เราจริงๆ ก็ต่อเมื่อ Buffer เต็ม, เจอ `\n` (กรณี stdout เป็น
Line-buffered เช่นตอนเชื่อมกับ Terminal), หรือโปรแกรมจบการทำงาน

```c
/* ============================================================
 * ชื่อไฟล์:     syscall_demo.c
 * คำอธิบาย:     เปรียบเทียบ printf() (User Space, ผ่าน libc buffer)
 *              กับ write() (เรียก System Call ตรงๆ ทันที)
 * ============================================================ */
#define _POSIX_C_SOURCE 200809L
#include <stdio.h>
#include <unistd.h>
#include <string.h>

int main(void) {
    const char *msg = "ข้อความนี้ถูกส่งออกผ่าน write() system call โดยตรง\n";

    /* printf() เป็นฟังก์ชันใน User Space (libc) ที่ทำ Buffering ก่อน
       แล้วค่อยเรียก write() ให้เราอีกที (มักจะเรียกตอนบัฟเฟอร์เต็ม หรือเจอ '\n'
       ในกรณี stdout ที่เป็น terminal / line-buffered) */
    printf("นี่คือข้อความจาก printf() — อยู่ใน User Space memory buffer ก่อน\n");

    /* write() คือ wrapper ของ system call เบอร์ SYS_write เรียก kernel โดยตรง
       ไม่มีการ buffer ใดๆ ทั้งสิ้น ข้อมูลถูกส่งเข้า kernel ทันที */
    write(STDOUT_FILENO, msg, strlen(msg));

    return 0;
}
```

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 -D_POSIX_C_SOURCE=200809L syscall_demo.c -o syscall_demo
./syscall_demo
```

```
นี่คือข้อความจาก printf() — อยู่ใน User Space memory buffer ก่อน
ข้อความนี้ถูกส่งออกผ่าน write() system call โดยตรง
```

เมื่อรันปกติผ่าน Terminal ผลลัพธ์ออกมาตามลำดับโค้ดพอดี เพราะ stdout ที่เชื่อมกับ Terminal
เป็นแบบ **Line-buffered** (เจอ `\n` ก็ flush ทันที) แต่ลองสังเกตผ่านเครื่องมือ **`strace`**
ซึ่งดักฟัง System Call ทุกตัวที่โปรแกรมเรียก:

```bash
strace -e trace=write ./syscall_demo
```

```
write(1, "\340\270\202...(bytes ของข้อความ printf)", 119) = 119
นี่คือข้อความจาก printf() — อยู่ใน User Space memory buffer ก่อน
write(1, "\340\270\202...(bytes ของข้อความ write)", 109) = 109
ข้อความนี้ถูกส่งออกผ่าน write() system call โดยตรง
+++ exited with 0 +++
```

เมื่อ stdout ไม่ได้เชื่อมกับ Terminal โดยตรง (เช่นถูก `strace` ครอบอยู่ หรือถูก pipe ไปที่
โปรแกรมอื่น) stdout จะเปลี่ยนเป็นโหมด **Fully-buffered** แทน (ประสิทธิภาพดีกว่าสำหรับข้อมูล
จำนวนมาก) ทำให้ `printf()` **ไม่** flush ทันทีที่เจอ `\n` แต่รอจนกว่า Buffer จะเต็มหรือ
โปรแกรมจบ — นี่คือเหตุผลที่จำนวนครั้งของ `write()` ที่เห็นใน `strace` (2 ครั้ง ตัวหนึ่งจาก
`printf()` อีกตัวจาก `write()` ที่เราเรียกเอง) ยืนยันว่าสุดท้ายแล้ว**ไม่ว่าจะผ่าน `printf()`
หรือเรียก `write()` ตรงๆ ก็ต้องจบลงที่ System Call ตัวเดียวกันเสมอ** — นี่คือหัวใจสำคัญของ
หัวข้อนี้: **`printf()` เป็นแค่ Wrapper ที่สะดวกกว่า แต่สุดท้ายก็ต้องพึ่ง System Call เพื่อ
คุยกับ Kernel อยู่ดี**

### System Call ที่พบบ่อย

| System Call | หน้าที่ | จะเรียนละเอียดใน |
|---|---|---|
| `read()`, `write()` | อ่าน/เขียนข้อมูลผ่าน File Descriptor | Part 13 (พื้นฐาน), Part 33 (Socket) |
| `open()`, `close()` | เปิด/ปิดไฟล์ | Part 13 |
| `fork()` | สร้าง Process ใหม่ | **Part 27** |
| `execve()` | รันโปรแกรมใหม่ทับ Process ปัจจุบัน (`execl`/`execvp` เป็น wrapper ของตัวนี้) | **Part 27** |
| `wait()`, `waitpid()` | รอ Process ลูกให้ทำงานจบ | **Part 27** |
| `kill()` | ส่ง Signal ไปยัง Process อื่น | **Part 28** |
| `mmap()` | Map หน่วยความจำ/ไฟล์เข้ากับ Address Space | Part 36 |
| `socket()`, `connect()`, `bind()` | สร้างและจัดการ Network Connection | Part 33–34 |
| `brk()` / `sbrk()` | ขยายขนาด Heap Segment (ที่ `malloc()` เรียกใช้ภายใน) | Part 11 (ทบทวน) |

---

## 26.6 Context Switching เบื้องต้น (Step 206)

เครื่องคอมพิวเตอร์ทั่วไปมี CPU Core จำนวนจำกัด (เช่น 4, 8, 16 core) แต่ระบบมักมี Process
ทำงานพร้อมกันเป็นร้อยเป็นพัน Process วิธีที่ Linux ทำให้ทุก Process "ดูเหมือน" ทำงานพร้อม
กันได้คือการ**สลับ CPU ไปมาระหว่าง Process อย่างรวดเร็ว** (เร็วจนมนุษย์รู้สึกเหมือนทำงาน
พร้อมกันจริงๆ) กระบวนการสลับนี้เรียกว่า **Context Switching**

### Context Switch ทำงานอย่างไร

เมื่อ Kernel ตัดสินใจสลับจาก Process A ไป Process B, Kernel (ผ่านส่วนที่เรียกว่า
**Scheduler**) จะทำ 3 ขั้นตอนหลัก:

1. **บันทึก (Save) Context ของ A**: ค่า Register ทั้งหมดของ CPU ขณะนั้น (รวมถึง Program
   Counter ที่บอกว่า A รันมาถึงบรรทัดไหนแล้ว) ถูกเก็บลงใน PCB ของ A (ดูหัวข้อ 26.1)
2. **โหลด (Load) Context ของ B**: ดึงค่า Register ที่เคยบันทึกไว้ของ B กลับเข้า CPU
   (รวมถึงสลับ Page Table ให้ MMU ใช้ของ B แทน — นี่คือส่วนที่มีค่าใช้จ่ายสูงเพราะทำให้
   Cache และ TLB ที่เคย "จำ" ข้อมูลของ A ไว้ใช้ไม่ได้อีกต่อไป ต้องเริ่มโหลดใหม่)
3. **กระโดดไปทำงานต่อที่ B**: CPU เริ่มรันคำสั่งของ B ต่อจากจุดที่ B เคยถูกสลับออกไปครั้งก่อน

```
เวลา:  0ms      10ms      20ms      30ms      40ms
       │─────────│─────────│─────────│─────────│
CPU:   [ Process A ][ Process B ][ Process A ][ Process C ]
              ▲            ▲            ▲
        Context Switch (บันทึก A, โหลด B, บันทึก B, โหลด A, ...)
```

### เมื่อไหร่ที่เกิด Context Switch

| ประเภท | ทริกเกอร์ | ตัวอย่าง |
|---|---|---|
| **Voluntary (สมัครใจ)** | Process เรียก System Call ที่ต้องรอ (Blocking) เช่น รอ I/O, `sleep()` | โปรแกรมเรียก `read()` รอข้อมูลจาก network แล้วยังไม่มีข้อมูลมา -> สละ CPU ให้ process อื่นระหว่างรอ |
| **Involuntary (ถูกบังคับ)** | Timer Interrupt บอกว่า Process ใช้เวลาครบโควตา (Time Slice) แล้ว scheduler แย่งคืน | โปรแกรมคำนวณหนักๆ ใน loop รันไม่หยุดนานเกินไป |
| **Preemption จาก Process สำคัญกว่า** | Process ที่มี Priority สูงกว่าพร้อมทำงาน | Interrupt Handler ของฮาร์ดแวร์ต้องการ CPU ด่วน |

### วัดค่าจริงด้วย `getrusage()`

```c
/* ============================================================
 * ชื่อไฟล์:     ctxswitch_demo.c
 * คำอธิบาย:     เปรียบเทียบจำนวน Context Switch แบบ Voluntary (สมัครใจ, เช่น sleep)
 *              กับแบบ Involuntary (ถูกบังคับ, เช่นถูก scheduler แย่ง CPU) โดยใช้
 *              getrusage() อ่านค่าสถิติจาก kernel
 * ============================================================ */
#define _POSIX_C_SOURCE 200809L
#include <stdio.h>
#include <sys/resource.h>
#include <time.h>

static void print_ctxswitch(const char *label) {
    struct rusage usage;
    getrusage(RUSAGE_SELF, &usage);
    printf("%-28s: voluntary=%-6ld involuntary=%-6ld\n",
           label, usage.ru_nvcsw, usage.ru_nivcsw);
}

int main(void) {
    struct timespec req = {.tv_sec = 0, .tv_nsec = 50000000L}; /* 50 ms */

    print_ctxswitch("ก่อนเริ่มทำงาน");

    /* Voluntary Context Switch: process "สมัครใจ" สละ CPU เอง เพราะรอ I/O (nanosleep
       เป็นการขอให้ kernel ปลุกเราในอนาคต ระหว่างนี้ยกเลิก CPU ให้ process อื่นไปก่อน) */
    for (int i = 0; i < 3; i++) {
        nanosleep(&req, NULL);
    }
    print_ctxswitch("หลังเรียก nanosleep() 3 ครั้ง");

    /* Involuntary Context Switch: ทำงานหนัก (busy loop) ต่อเนื่องนานพอที่ scheduler
       ของ kernel จะตัดสินใจแย่ง CPU คืนไปให้ process อื่นตาม time slice (นับแม้แต่ตอน
       ทำงานอยู่ก็ยังถูกสลับได้ ถ้าใช้เวลานานเกินโควตาที่ scheduler กำหนด) */
    volatile long counter = 0;
    for (long i = 0; i < 1200000000L; i++) {
        counter += i % 7;
    }
    print_ctxswitch("หลัง busy loop หนักๆ");

    return 0;
}
```

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 -D_POSIX_C_SOURCE=200809L -O0 ctxswitch_demo.c -o ctxswitch_demo
./ctxswitch_demo
```

```
ก่อนเริ่มทำงาน               : voluntary=0      involuntary=0
หลังเรียก nanosleep() 3 ครั้ง  : voluntary=3      involuntary=0
หลัง busy loop หนักๆ          : voluntary=3      involuntary=3
```

สังเกตว่า `nanosleep()` 3 ครั้งทำให้ `voluntary` เพิ่มขึ้น 3 พอดี (สอดคล้องกับจำนวนครั้งที่
สละ CPU โดยสมัครใจ) แต่ `involuntary` ไม่เพิ่มเลยในช่วงนั้น ในทางกลับกัน Busy Loop ที่ใช้
เวลาประมวลผลต่อเนื่องนานหลายร้อยมิลลิวินาที ทำให้ `involuntary` เพิ่มขึ้น เพราะ Scheduler
ของ Kernel แย่ง CPU คืนไปให้ Process อื่นตาม Time Slice ที่กำหนดไว้ (ค่าที่ได้อาจแตกต่างกัน
ไปในแต่ละเครื่อง ขึ้นกับจำนวน Process อื่นที่รันแข่งอยู่ ณ ขณะนั้นด้วย)

> **ทำไม Context Switch ถึงมี "ต้นทุน" (Overhead)?** นอกจากเวลาที่ใช้บันทึก/โหลด Register
> แล้ว สิ่งที่แพงกว่านั้นคือ **CPU Cache และ TLB (Translation Lookaside Buffer)** ที่เคย
> "จำ" ข้อมูลและ Address Translation ของ Process เดิมไว้ จะถูกล้างทิ้งไปเมื่อสลับไป Process
> ใหม่ ทำให้ Process ใหม่ต้องเริ่มโหลดข้อมูลจาก RAM ใหม่ทั้งหมด (Cache Miss สูงขึ้นชั่วคราว)
> — นี่คือเหตุผลที่โปรแกรมที่มี Thread/Process จำนวนมากเกินความจำเป็นอาจช้าลงจริง แม้จะ
> ดูเหมือน "ขนานกันมากขึ้น" ก็ตาม (จะเจาะลึกเรื่องนี้ต่อใน Module G)

---

## 26.7 เครื่องมือสำรวจ Process บน Linux: `ps` และ `top` (Step 207)

### `ps` — ดู Snapshot ของ Process ณ ขณะนั้น

```bash
ps aux | head -6
```

```
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root         1  0.1  0.0  23692  4672 ?        SLl  05:51   0:01 /sbin/init
root         2  0.0  0.0      0     0 ?        S    05:51   0:00 [kthreadd]
root         3  0.0  0.0      0     0 ?        S    05:51   0:00 [pool_workqueue_release]
root         4  0.0  0.0      0     0 ?        I<   05:51   0:00 [kworker/R-rcu_gp]
```

| คอลัมน์ | ความหมาย |
|---|---|
| `USER` | เจ้าของ Process |
| `PID` | Process ID |
| `%CPU` | เปอร์เซ็นต์ CPU ที่ใช้ (เฉลี่ยตั้งแต่เริ่ม) |
| `%MEM` | เปอร์เซ็นต์ RAM ที่ใช้ |
| `VSZ` | Virtual Memory Size (KB) — ขนาด Virtual Address Space ทั้งหมดที่จองไว้ (รวมส่วนที่ยังไม่ได้ใช้จริง) |
| `RSS` | Resident Set Size (KB) — หน่วยความจำจริงที่อยู่ใน RAM ณ ขณะนี้ (ไม่รวมส่วนที่ถูก swap ออกไป) |
| `TTY` | Terminal ที่ผูกอยู่ (`?` = ไม่มี เช่น background service) |
| `STAT` | สถานะของ Process (ดูตารางด้านล่าง) |
| `START` | เวลาที่เริ่มทำงาน |
| `TIME` | เวลา CPU สะสมที่ใช้ไปจริง |
| `COMMAND` | คำสั่งที่ใช้เริ่ม Process นี้ |

```bash
ps -ef | head -6
```

```
UID        PID  PPID  C STIME TTY          TIME CMD
root         1     0  0 05:51 ?        00:00:01 /sbin/init
root         2     0  0 05:51 ?        00:00:00 [kthreadd]
```

`ps -ef` เน้นแสดง `PPID` (ซึ่ง `ps aux` ไม่มี) เหมาะเวลาต้องการดูว่า Process ไหนเป็นลูกของ
ใคร ส่วน `ps aux` เน้นแสดงการใช้ CPU/RAM — เลือกใช้ตามสิ่งที่ต้องการดู

### Process State (คอลัมน์ `STAT`)

| ตัวอักษร | สถานะ | ความหมาย |
|---|---|---|
| `R` | Running | กำลังรันอยู่บน CPU หรือพร้อมรัน (Runnable) |
| `S` | Sleeping | รอ Event บางอย่าง (Interruptible — รับ Signal ได้ระหว่างรอ) |
| `D` | Uninterruptible Sleep | รอ I/O (มักเป็น Disk) — รับ Signal ไม่ได้ระหว่างรอ |
| `Z` | Zombie | จบการทำงานแล้ว แต่ Parent ยังไม่มา `wait()` รับ exit status (**Part 27**) |
| `T` | Stopped | ถูกหยุดไว้ชั่วคราว (เช่นโดน `SIGSTOP` หรือ Ctrl+Z) |
| `<` | High Priority | ต่อท้าย STAT ตัวอื่น หมายถึง Priority สูงกว่าปกติ |

### `ps` แบบกำหนดคอลัมน์เอง

```bash
ps -o pid,ppid,stat,pcpu,pmem,cmd -p $$
```

```
  PID  PPID STAT %CPU %MEM CMD
 3445  3403 Ss   20.0  0.0 /bin/bash
```

`-o` ให้เราเลือกเฉพาะคอลัมน์ที่สนใจ มีประโยชน์มากเวลาเขียน Script อัตโนมัติที่ต้องการดึง
ข้อมูลเฉพาะเจาะจง เช่น `ps -o pid= -p 1234` จะพิมพ์แค่ตัวเลข PID ล้วนๆ ไม่มี Header
(เครื่องหมาย `=` หลังชื่อคอลัมน์คือการบอกให้ไม่ต้องพิมพ์หัวตาราง)

### `top` — ดูแบบ Real-time

```bash
top
```

`top` แสดงข้อมูล Process ทั้งหมดแบบ**อัปเดตสด** (ค่า Default คือทุก 3 วินาที) เรียงตามการ
ใช้ CPU มากไปน้อยเป็นค่าเริ่มต้น ส่วนหัวของหน้าจอแสดงข้อมูลรวมของทั้งระบบ:

```
top - 14:32:10 up 2 days,  3:14,  2 users,  load average: 0.52, 0.58, 0.61
Tasks: 215 total,   1 running, 213 sleeping,   0 stopped,   1 zombie
%Cpu(s):  3.2 us,  1.1 sy,  0.0 ni, 95.4 id,  0.2 wa,  0.0 hi,  0.1 si,  0.0 st
MiB Mem :  16019.5 total,   3204.1 free,   6822.3 used,   5993.1 buff/cache
```

| ปุ่มลัดใน `top` | ทำอะไร |
|---|---|
| `q` | ออกจากโปรแกรม |
| `k` | ส่ง signal (ฆ่า) process ตาม PID ที่ระบุ |
| `M` | เรียงตามการใช้ Memory |
| `P` | เรียงตามการใช้ CPU (ค่า default) |
| `1` | แสดงการใช้งานแยกตามแต่ละ CPU Core |

> **`htop`**: ทางเลือกที่เป็นมิตรกว่า `top` มาก (มีสี, เมาส์คลิกได้, กราฟการใช้ CPU/RAM
> แบบเห็นภาพชัดเจน) ติดตั้งด้วย `sudo apt install htop` — แนะนำให้ใช้ในการทำงานประจำวัน
> แต่ `top` ยังสำคัญเพราะมีติดตั้งในแทบทุก Linux Server โดยไม่ต้องลงเพิ่ม

---

## 26.8 `/proc` Filesystem (Step 208)

`/proc` เป็น **Virtual Filesystem** ที่ไม่ได้เก็บข้อมูลจริงอยู่บน Disk เลย — ไฟล์ทุกไฟล์ใน
`/proc` ถูก Kernel "สร้างขึ้นมาสดๆ" ทุกครั้งที่มีการอ่าน โดยดึงข้อมูลจาก Data Structure
ภายใน Kernel เอง (เช่น PCB ที่พูดถึงในหัวข้อ 26.1) นี่คือวิธีที่เครื่องมืออย่าง `ps`, `top`
ใช้ดึงข้อมูล Process จริงๆ เบื้องหลัง — พวกมันก็แค่**อ่านไฟล์ใน `/proc` แล้วจัดรูปแบบให้
สวยงาม**เท่านั้นเอง

### ไฟล์สำคัญใน `/proc` ที่ควรรู้จัก

| Path | ข้อมูล |
|---|---|
| `/proc/[pid]/status` | สรุปข้อมูลของ process (State, VmRSS, Threads, ...) แบบอ่านง่าย |
| `/proc/[pid]/cmdline` | คำสั่งเต็มที่ใช้เริ่ม process นี้ |
| `/proc/[pid]/maps` | Memory Mapping ทั้งหมด (ดูหัวข้อ 26.3) |
| `/proc/[pid]/fd/` | รายการ File Descriptor ที่ process นี้เปิดอยู่ (เป็น symlink ไปยังไฟล์จริง) |
| `/proc/[pid]/environ` | Environment Variable ของ process นี้ |
| `/proc/self` | Symlink พิเศษที่ชี้ไปยัง `/proc/[pid]` ของ**ตัวเอง**เสมอ (สะดวกมากเวลาเขียนโปรแกรมอ่านข้อมูลตัวเอง) |
| `/proc/meminfo` | ข้อมูลหน่วยความจำของทั้งระบบ |
| `/proc/cpuinfo` | ข้อมูล CPU ของเครื่อง |
| `/proc/loadavg` | ค่า Load Average ของระบบ (ตัวเลขเดียวกับที่ `top` แสดงบรรทัดบนสุด) |

```bash
head -5 /proc/meminfo
```

```
MemTotal:       16481980 kB
MemFree:        15675608 kB
MemAvailable:   15893000 kB
Buffers:            7388 kB
Cached:           423616 kB
```

### อ่าน `/proc/self/status` จากโปรแกรม C โดยตรง

เพราะ `/proc/[pid]/status` เป็นแค่ไฟล์ข้อความธรรมดา (Text File) เราจึงเปิดอ่านมันได้ด้วย
`fopen()`/`fgets()` แบบเดียวกับไฟล์ทั่วไปที่เรียนใน Part 13 ทุกประการ ไม่ต้องใช้ API พิเศษใดๆ
เลย — นี่คือความสวยงามของปรัชญา Unix ที่ว่า **"Everything is a file"**

```c
/* ============================================================
 * ชื่อไฟล์:     proc_reader.c
 * คำอธิบาย:     อ่านข้อมูลของ process ตัวเองจาก /proc/self/status โดยตรง
 *              (ตัวอย่างการเข้าถึงข้อมูล kernel ผ่าน /proc filesystem)
 * ============================================================ */
#define _POSIX_C_SOURCE 200809L
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#define LINE_MAX_LEN 256

int main(void) {
    FILE *fp = fopen("/proc/self/status", "r");
    if (fp == NULL) {
        perror("fopen /proc/self/status ล้มเหลว");
        return 1;
    }

    /* เรากันหน่วยความจำก้อนใหญ่ไว้บน Heap เพื่อไม่ให้กินพื้นที่ Stack มาก */
    char *line = malloc(LINE_MAX_LEN);
    if (line == NULL) {
        fclose(fp);
        return 1;
    }

    printf("ข้อมูลบางส่วนของ process ตัวเองจาก /proc/self/status:\n\n");

    while (fgets(line, LINE_MAX_LEN, fp) != NULL) {
        /* เลือกพิมพ์เฉพาะบรรทัดที่น่าสนใจ */
        if (strncmp(line, "Name:", 5) == 0 ||
            strncmp(line, "State:", 6) == 0 ||
            strncmp(line, "Pid:", 4) == 0 ||
            strncmp(line, "PPid:", 5) == 0 ||
            strncmp(line, "VmSize:", 7) == 0 ||
            strncmp(line, "VmRSS:", 6) == 0 ||
            strncmp(line, "Threads:", 8) == 0) {
            printf("  %s", line);
        }
    }

    free(line);
    fclose(fp);
    return 0;
}
```

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 -D_POSIX_C_SOURCE=200809L proc_reader.c -o proc_reader
./proc_reader
```

```
ข้อมูลบางส่วนของ process ตัวเองจาก /proc/self/status:

  Name:	proc_reader
  State:	R (running)
  Pid:	3418
  PPid:	3403
  VmSize:	    2692 kB
  VmRSS:	    1488 kB
  Threads:	1
```

`VmSize` คือ Virtual Memory ทั้งหมดที่จองไว้ (เทียบเท่า `VSZ` ใน `ps`) ส่วน `VmRSS` คือ
หน่วยความจำจริงที่อยู่ใน RAM ตอนนี้ (เทียบเท่า `RSS` ใน `ps`) — ตัวเลขนี้มักน้อยกว่า `VmSize`
มาก เพราะโปรแกรมมักจอง Virtual Address Space ไว้เผื่อ แต่ยังไม่ได้ใช้จริงทั้งหมด (Kernel
จะ Map Physical Page ให้จริงๆ ก็ต่อเมื่อมีการอ่าน/เขียนหน่วยความจำส่วนนั้นครั้งแรกเท่านั้น
เรียกว่า **Demand Paging** ซึ่งต่อยอดจากแนวคิด Virtual Memory ในหัวข้อ 26.3)

---

## ข้อผิดพลาดที่พบบ่อย (Common Pitfalls)

1. **คิดว่า Address ที่ `printf("%p", ...)` แสดงคือตำแหน่งจริงบนแผง RAM** — ตามที่อธิบายใน
   หัวข้อ 26.3 ค่าที่เห็นเป็น**เพียง Virtual Address** เท่านั้น ตำแหน่งจริงทางฟิสิกส์ถูกซ่อน
   ไว้หลัง MMU และ Page Table เสมอ โปรแกรม User Space ไม่มีทางรู้หรือควบคุมมันได้โดยตรง

2. **คาดหวังว่า Address จะเหมือนเดิมทุกครั้งที่รันโปรแกรม** — เพราะ **ASLR** (หัวข้อ 26.3)
   สุ่มตำแหน่งเริ่มต้นของ Stack/Heap/Library ใหม่ทุกครั้งที่โหลดโปรแกรม การเขียนโค้ดหรือ
   ทดสอบที่ไปฝังค่า Address ตายตัวไว้ (Hardcoded Address) จะพังทันทีเมื่อรันคนละครั้ง

3. **สมมติว่า PID จะไม่ถูกใช้ซ้ำ** — Linux คืนค่า PID ที่ตายไปแล้วกลับมาใช้ใหม่ได้เสมอ (วน
   กลับมาใช้ตั้งแต่ PID เล็กๆ เมื่อ PID สูงสุดที่อนุญาต หรือ `/proc/sys/kernel/pid_max`
   ถูกใช้จนหมด) ถ้าเขียนโปรแกรมที่เก็บ PID ไว้อ้างอิง Process ในอนาคต (เช่นเก็บ PID ไว้ใน
   ไฟล์ Lock) ต้องระวัง Race Condition ที่ Process เดิมตายไปแล้ว แต่ PID เดียวกันถูกนำไปใช้
   กับ Process อื่นที่ไม่เกี่ยวข้องกันเลย

4. **อ่าน `/proc/[pid]/...` โดยไม่ตรวจสอบว่า Process นั้นยังมีชีวิตอยู่หรือไม่** — Process
   อาจจบการทำงานไปแล้วระหว่างที่โปรแกรมของเรากำลังจะเปิดไฟล์ `/proc/[pid]/status` (Race
   Condition ระหว่างตรวจสอบกับใช้งานจริง) ทำให้ `fopen()` คืนค่า `NULL` ต้องเช็ค Return
   Value และจัดการ Error เสมอ อย่าสมมติว่า PID ที่เพิ่งเห็นจาก `ps` จะยังอยู่แน่ๆ

5. **เข้าใจผิดว่า Stack กับ Heap ต้อง "โตชนกัน" เสมอถ้าใช้เยอะพอ** — แม้แผนภาพในหัวข้อ 26.2
   จะวาดให้ Stack กับ Heap โตเข้าหากันตรงกลาง แต่ในความเป็นจริงบน Linux สมัยใหม่ที่ใช้ 64-bit
   Address Space ขนาดมหาศาล โอกาสที่ทั้งสองจะโตมาชนกันจริงมีน้อยมาก (ปัญหาที่พบบ่อยกว่าคือ
   **Stack Overflow** จากการเรียก Recursion ลึกเกินไป หรือประกาศ Array ขนาดใหญ่บน Stack
   โดยตรง ซึ่งชนกับ**ขีดจำกัดของ Stack เอง** ไม่ใช่ชนกับ Heap — ทบทวนได้จาก Common Pitfall
   ข้อ 2 ใน Part 25)

6. **ลืมว่า `printf()` ไม่ได้พิมพ์ออกจอทันที** — เพราะ `printf()` ทำ Buffering ก่อนเสมอ
   (หัวข้อ 26.5) ถ้าโปรแกรม Crash กลางทาง (เช่นเจอ Segmentation Fault) ข้อความที่ `printf()`
   ไว้แต่ยังไม่ทัน Flush อาจ**หายไปเลย ไม่ถูกพิมพ์ออกมา** ทำให้ Debug สับสนว่าโปรแกรม Crash
   ตรงไหนกันแน่ — ควรใช้ `fflush(stdout)` หลัง `printf()` ที่สำคัญ หรือใช้ `stderr` (ซึ่ง
   Unbuffered ตามมาตรฐาน) สำหรับข้อความ Debug/Error

7. **ใช้ผลลัพธ์จาก `ps`/`top` แบบ Single-snapshot มาสรุปพฤติกรรมระยะยาวของโปรแกรม** — ค่า
   `%CPU` ใน `ps aux` เป็นค่าเฉลี่ยตั้งแต่โปรแกรมเริ่มทำงาน ไม่ใช่ค่า ณ วินาทีนั้นเป๊ะๆ ถ้า
   ต้องการดูพฤติกรรมสดควรใช้ `top`/`htop` ที่อัปเดตต่อเนื่อง หรือเก็บค่าจาก `/proc` เองเป็น
   ช่วงเวลาแล้วคำนวณผลต่าง (Delta) เอา

---

## แบบฝึกหัดท้ายบท

1. รันโปรแกรม `process_info.c` จากบทเรียนสองครั้งพร้อมกันในสอง Terminal แล้วใช้คำสั่ง
   `ps -o pid,ppid,stat,cmd -p <PID>` ตรวจสอบ PID/PPID ที่ได้ ว่าตรงกับที่โปรแกรมพิมพ์
   ออกมาหรือไม่

2. ดัดแปลง `memory_layout.c` ให้เรียก `malloc()` ติดกัน 3 ครั้ง แล้วพิมพ์ Address ของแต่ละ
   ครั้ง สังเกตว่า Heap "โต" ไปในทิศทางไหน (Address เพิ่มขึ้นหรือลดลง) เทียบกับที่อธิบายไว้
   ในหัวข้อ 26.2

3. ใช้คำสั่ง `cat /proc/self/maps` นับดูว่า Process ของ Shell ที่คุณใช้อยู่ตอนนี้มี Memory
   Mapping ทั้งหมดกี่บรรทัด และมีบรรทัดไหนที่เป็น Shared Library (`.so`) บ้าง

4. เขียนโปรแกรม `proc_status_of_pid.c` ที่รับ PID จาก `argv[1]` แล้วอ่านค่า `VmRSS` ของ
   Process นั้นจาก `/proc/<pid>/status` มาพิมพ์ (คำใบ้: ใช้ `snprintf()` สร้าง path แบบ
   Dynamic จาก PID ที่รับมา)

5. ดัดแปลง `ctxswitch_demo.c` ให้เปรียบเทียบจำนวน Involuntary Context Switch ระหว่างการรัน
   Busy Loop ตัวเดียว กับการรันโปรแกรมเดียวกัน 4 ชุดพร้อมกัน (เปิด 4 Terminal รันพร้อมกัน)
   อธิบายว่าทำไมตัวเลขถึงเปลี่ยนไป

6. อธิบายด้วยคำพูดของตัวเอง (ไม่ต้องเขียนโค้ด) ว่าทำไม Kernel Space กับ User Space ถึงต้อง
   แยกจากกัน และยกตัวอย่างสถานการณ์ที่ถ้าไม่มีการแยกนี้ อาจเกิดปัญหาอะไรกับระบบ

### แนวทางเฉลยข้อ 1

```bash
# Terminal 1
./process_info
```

```
โปรแกรมนี้กำลังรันเป็น Process หมายเลข (PID) = 15234
ถูกสร้างขึ้นโดย Process แม่ (PPID)          = 15100
```

ระหว่างโปรแกรมกำลังรันอยู่ (ถ้าต้องการเวลาตรวจสอบ อาจเติม `sleep(3);` ก่อน `return 0;`
ชั่วคราว) เปิด Terminal 2 แล้วตรวจสอบ:

```bash
# Terminal 2
ps -o pid,ppid,stat,cmd -p 15234
```

```
  PID  PPID STAT CMD
15234 15100 S+   ./process_info
```

จะเห็นว่า `PID` และ `PPID` ที่ `ps` รายงานตรงกับที่โปรแกรมพิมพ์ออกมาทุกประการ เพราะทั้งคู่
ดึงข้อมูลมาจากแหล่งเดียวกันคือ Kernel (โปรแกรมของเราเรียกผ่าน `getpid()`/`getppid()`
ในขณะที่ `ps` เดินอ่านข้อมูลผ่าน `/proc/15234/status` ดังที่อธิบายไว้ในหัวข้อ 26.8) — นี่
คือหลักฐานที่ยืนยันว่าไม่ว่าจะถามผ่านช่องทางไหน คำตอบก็มาจากแหล่งความจริงเดียวกันเสมอ
คือข้อมูลที่ Kernel เก็บไว้ใน Process Control Block ของแต่ละ Process

### แนวทางเฉลยข้อ 4

```c
/* ============================================================
 * ชื่อไฟล์:     proc_status_of_pid.c
 * คำอธิบาย:     เฉลยแบบฝึกหัด — อ่าน /proc/<pid>/status ของ process ใดๆ ที่ระบุผ่าน
 *              argv[1] แล้วพิมพ์ค่า VmRSS (หน่วยความจำจริงที่ process นั้นใช้อยู่)
 * ============================================================ */
#define _POSIX_C_SOURCE 200809L
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#define PATH_LEN 64
#define LINE_LEN 256

int main(int argc, char *argv[]) {
    if (argc != 2) {
        fprintf(stderr, "วิธีใช้: %s <PID>\n", argv[0]);
        return 1;
    }

    char path[PATH_LEN];
    snprintf(path, sizeof(path), "/proc/%s/status", argv[1]);

    FILE *fp = fopen(path, "r");
    if (fp == NULL) {
        fprintf(stderr, "ไม่พบ process PID=%s (อาจจบการทำงานไปแล้ว)\n", argv[1]);
        return 1;
    }

    char line[LINE_LEN];
    int found = 0;
    while (fgets(line, sizeof(line), fp) != NULL) {
        if (strncmp(line, "VmRSS:", 6) == 0) {
            printf("PID %s ใช้หน่วยความจำจริง (VmRSS) = %s", argv[1], line + 6);
            found = 1;
            break;
        }
    }

    if (!found) {
        printf("PID %s ไม่มีข้อมูล VmRSS (อาจเป็น kernel thread)\n", argv[1]);
    }

    fclose(fp);
    return 0;
}
```

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 -D_POSIX_C_SOURCE=200809L proc_status_of_pid.c -o proc_status_of_pid
./proc_status_of_pid $$        # $$ คือ PID ของ shell ปัจจุบันใน bash
./proc_status_of_pid 1         # PID 1 คือ init/systemd เสมอบนระบบ Linux
```

```
PID 5062 ใช้หน่วยความจำจริง (VmRSS) = 	    6216 kB
PID 1 ใช้หน่วยความจำจริง (VmRSS) = 	    4672 kB
```

**คำอธิบาย**: จุดสำคัญของเฉลยนี้คือการสร้าง Path แบบ Dynamic ด้วย `snprintf()` แทนการ
Hardcode PID ไว้ในโค้ด ทำให้โปรแกรมนี้ใช้ตรวจสอบ Process **ใดก็ได้**ในระบบที่เรามีสิทธิ์
อ่าน (Process ของ User อื่นหรือ Process ระดับ Kernel บางตัวอาจอ่านไม่ได้เพราะสิทธิ์ไม่พอ
ซึ่งเป็นพฤติกรรมที่ถูกต้องตามหลัก Kernel Space vs User Space ในหัวข้อ 26.4 — Kernel จะ
ปฏิเสธการอ่านถ้าเราไม่มีสิทธิ์เพียงพอ) การใช้ `argc != 2` ตรวจสอบจำนวน Argument ก่อนเสมอ
เป็นนิสัยการเขียนโปรแกรมที่ปลอดภัยที่ควรทำทุกครั้งที่รับ Input จากผู้ใช้

---

## สรุปท้ายบท

ใน Part นี้เราได้:

- แยกแยะความแตกต่างระหว่าง **Program** (ไฟล์นิ่งบน Disk) กับ **Process** (การทำงานจริง
  ที่มี PID, PPID และทรัพยากรของตัวเอง) พร้อมพิสูจน์ด้วยโปรแกรมจริง
- เข้าใจ **Memory Layout ของ Process** ครบทั้ง 5 Segment (Text, Data, BSS, Heap, Stack)
  พร้อมพิสูจน์ลำดับ Address จริงด้วยโปรแกรม C
- เข้าใจแนวคิด **Virtual Memory** ว่าทำไมแต่ละ Process ถึงมี Address Space เป็นของตัวเอง
  โดยไม่ชนกัน ผ่านกลไก MMU และ Page Table พร้อมรู้จัก ASLR
- แยกแยะ **Kernel Space vs User Space** และเหตุผลด้านความปลอดภัย/เสถียรภาพที่ต้องแยกกัน
- เข้าใจว่า **System Call** คือประตูเดียวที่เชื่อม User Space กับ Kernel Space และตามรอย
  ได้ว่า `printf()` สุดท้ายก็ต้องพึ่ง `write()` System Call เหมือนกัน
- เข้าใจ **Context Switching** ทั้งแบบ Voluntary และ Involuntary พร้อมวัดค่าจริงด้วย
  `getrusage()`
- ใช้เครื่องมือสำรวจ Process ระดับมืออาชีพ: `ps`, `top`, `htop`, และเขียนโปรแกรม C อ่าน
  ข้อมูลจาก **`/proc` filesystem** โดยตรง

ความรู้ทั้งหมดใน Part นี้คือรากฐานที่จำเป็นสำหรับก้าวต่อไป เพราะใน **Part 27** เราจะเริ่ม
**สร้าง Process ใหม่ด้วยตัวเอง** ผ่าน System Call `fork()` และ `exec()` ซึ่งเป็นหัวใจของ
Systems Programming — ทุกอย่างที่เรียนไปวันนี้ (Memory Layout, Virtual Memory, PID/PPID)
จะกลับมาเกี่ยวข้องโดยตรงทันทีที่เราเริ่มเขียน `fork()` เป็นครั้งแรก

**ต่อไป:** [Part 27 — Process Management (fork/exec/wait)](./part-027-process-management.md)
