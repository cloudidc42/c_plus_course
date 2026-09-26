# Part 117: ภาพรวม Embedded Systems Programming ด้วย C/C++ (Step 929–936)

> Module J — Professional และ World-Class Practices | Part 117 จาก 125
> Step ที่ครอบคลุมใน Part นี้: Step 929–936
> Part ก่อนหน้า: [Part 116 — Cross-Platform Development: Windows/Linux/macOS](./part-116-cross-platform.md) | Part ถัดไป: [Part 118 — พื้นฐาน Game Development ด้วย C++ (SDL2/OpenGL)](./part-118-game-dev-basics.md)

## เป้าหมายการเรียนรู้ (Learning Objectives)

หลังจบ Part นี้ ผู้เรียนจะสามารถ:

1. อธิบายได้ว่า Embedded System คืออะไร ต่างจากโปรแกรมบน Desktop/Server ที่เรียนมาตลอด
   หลักสูตรอย่างไร และทำไม C/C++ ยังเป็นภาษาหลักของวงการนี้
2. อธิบายข้อจำกัดสำคัญของการเขียนโปรแกรม embedded (ไม่มี heap ไม่จำกัด, มักปิด exception/RTTI,
   ไม่มี OS คอยจัดการ memory/process ให้)
3. เข้าใจแนวคิด Cross-Compilation และใช้ `gcc-arm-none-eabi` คอมไพล์โปรแกรมสำหรับ CPU
   ตระกูล ARM Cortex-M ได้จริงบนเครื่อง x86-64
4. อธิบายโครงสร้างของโปรแกรม bare-metal (Vector Table, Reset Handler, Linker Script,
   ส่วน `.text`/`.data`/`.bss`) และเขียน Linker Script เองได้
5. ใช้ QEMU (`qemu-system-arm`) จำลองฮาร์ดแวร์ ARM เพื่อรันและทดสอบโปรแกรม bare-metal
   โดยไม่ต้องมีบอร์ดจริง
6. เขียนโค้ดเข้าถึง Memory-Mapped Register ของอุปกรณ์ต่อพ่วง (peripheral) เช่น UART ได้อย่าง
   ปลอดภัยด้วย `volatile`
7. อธิบายว่า RTOS (Real-Time Operating System) คืออะไร ให้อะไรเพิ่มจาก bare-metal superloop
   และรู้จัก FreeRTOS ในระดับภาพรวม
8. รู้แนวทางการ debug โปรแกรม embedded ในโลกจริง (JTAG/SWD, semihosting, QEMU GDB stub)

> **หมายเหตุสำคัญเรื่องสภาพแวดล้อมของ Part นี้**: เครื่องที่ใช้เขียนบทเรียนนี้ไม่มีบอร์ด ARM
> จริงต่ออยู่ (เป็น container แบบ headless) แต่มี **`gcc-arm-none-eabi` (cross-compiler จริง)**
> และ **`qemu-system-arm` (เครื่องจำลองฮาร์ดแวร์ ARM จริง)** ติดตั้งอยู่ครบ ทุกตัวอย่างในบทนี้
> ถูก **cross-compile จริงและรันบน QEMU จริง** ผลลัพธ์ที่แปะไว้คือ output จริงที่ได้จากการรัน
> ไม่ใช่การจำลองด้วยมือ — นี่คือแนวทางเดียวกับที่วิศวกร embedded มืออาชีพจำนวนมากใช้ทดสอบโค้ด
> เบื้องต้นก่อนจะไปทดสอบบนบอร์ดจริงจริงๆ (เรียกว่า "hardware-in-the-loop" มาทีหลัง)

---

## 117.1 Embedded System คืออะไร (Step 929)

ตลอดหลักสูตรนี้ เราเขียนโปรแกรมที่รันบน **Desktop/Server OS** (Linux) ซึ่งมี memory
หลาย GB, มี virtual memory, มี process scheduler, มี filesystem เต็มรูปแบบ, และมี `malloc`
ที่ขอ memory ได้แทบไม่จำกัด (จนกว่าจะเต็ม RAM จริงๆ)

**Embedded System** คือคอมพิวเตอร์ขนาดเล็กที่ถูกฝัง (embed) อยู่ในอุปกรณ์อื่นเพื่อทำงาน
เฉพาะทางอย่างใดอย่างหนึ่ง — ไม่ใช่คอมพิวเตอร์เอนกประสงค์ ตัวอย่างที่พบในชีวิตประจำวัน:

- ไมโครคอนโทรลเลอร์ในเครื่องซักผ้า, ตู้เย็น, ไมโครเวฟ
- ระบบควบคุมในรถยนต์ (ECU — Engine Control Unit, ABS, Airbag controller)
- อุปกรณ์ IoT (เซนเซอร์วัดอุณหภูมิ, smart plug, wearable device)
- Firmware ของอุปกรณ์ต่อพ่วงคอมพิวเตอร์ (คีย์บอร์ด, เมาส์, SSD controller)
- ระบบการบินและอวกาศ, อุปกรณ์การแพทย์

### เปรียบเทียบทรัพยากร: Desktop/Server vs. ไมโครคอนโทรลเลอร์ทั่วไป

| ทรัพยากร | Desktop/Server (ที่เราใช้มาตลอดหลักสูตร) | ไมโครคอนโทรลเลอร์ทั่วไป (เช่น STM32, Cortex-M) |
|---|---|---|
| RAM | หลาย GB ถึงหลายร้อย GB | ตั้งแต่ 2 KB ถึงไม่กี่ MB (ตัวที่ใช้ในบทนี้จำลอง: 64 KB) |
| Flash/Storage | หลาย GB ถึง TB (SSD/HDD) | ตั้งแต่ 16 KB ถึงไม่กี่ MB |
| CPU Clock | 2–5 GHz หลาย core | ไม่กี่ MHz ถึงร้อยกว่า MHz ส่วนใหญ่ 1 core |
| Operating System | เต็มรูปแบบ (Linux/Windows/macOS) | มักไม่มีเลย (bare-metal) หรือ RTOS ขนาดเล็กมาก |
| Virtual Memory / MMU | มี | ส่วนใหญ่ไม่มี (Cortex-M ไม่มี MMU) |
| Heap แบบไม่จำกัด | มี (จำกัดแค่ RAM จริง) | มักจำกัดมาก หรือหลีกเลี่ยงการใช้ heap เลย |
| Floating Point Unit (FPU) | มีทุกเครื่อง | มีเฉพาะบางรุ่น (ต้องเช็คก่อนใช้ `float`/`double`) |
| ราคาต่อหน่วย | หลักหมื่นถึงหลักแสนบาท | หลักสิบบาทถึงหลักร้อยบาท (ผลิตเป็นล้านตัว) |
| อายุการทำงานที่คาดหวัง | ไม่กี่ปี | 10–20 ปีขึ้นไป (เครื่องใช้ไฟฟ้า, รถยนต์, อุตสาหกรรม) |

จุดที่น่าสนใจคือข้อสุดท้าย: firmware ของอุปกรณ์ embedded มักต้องทำงานได้ถูกต้อง **นานนับสิบปี
โดยไม่มีการอัพเดต** (อุปกรณ์การแพทย์บางตัว, ระบบควบคุมอุตสาหกรรม) ทำให้คุณภาพโค้ดและการ
ทดสอบมีความสำคัญสูงมาก — บั๊กที่แก้ทีหลังไม่ได้ (เพราะเรียกคืนอุปกรณ์นับล้านชิ้นทำไม่ได้จริง)
คือความเสี่ยงที่วิศวกร embedded ต้องคิดหนักกว่าโปรแกรมเมอร์ web/desktop ทั่วไป

---

## 117.2 ความแตกต่างจากการเขียนโปรแกรม Desktop/Server ที่เรียนมาตลอดหลักสูตร (Step 930)

การเขียนโปรแกรม embedded ใช้ภาษา C/C++ เหมือนที่เรียนมา แต่มีข้อจำกัดสำคัญที่ต้องปรับความคิด
ใหม่หลายจุด:

### 1. ไม่มี Heap ไม่จำกัด (หรือไม่มี Heap เลย)

โปรแกรมบน Desktop เรียก `new`/`malloc` ได้แทบไม่ต้องคิดมาก แต่บน microcontroller ที่มี RAM
แค่ 64 KB การ fragment ของ heap จาก `new`/`delete` ที่ทำซ้ำไปมานับล้านครั้งตลอดอายุการทำงาน
10 ปี **อาจทำให้โปรแกรม crash หลังทำงานไปแล้วหลายเดือน** เพราะ memory แตกเป็นชิ้นเล็กจน
allocate ก้อนใหญ่ไม่ได้อีก (Heap Fragmentation) — แนวทางที่นิยมในวงการ embedded คือ:

- **หลีกเลี่ยง dynamic allocation หลัง initialization เสร็จ** — allocate ทุกอย่างที่ต้องใช้
  ตอนเริ่มโปรแกรม (static allocation) แล้วไม่ปล่อย/ขอเพิ่มอีกเลยตลอดการทำงาน
- ใช้ **Memory Pool** ขนาดคงที่แทน heap ทั่วไป (เตรียม array ขนาดคงที่ไว้ล่วงหน้า)
- MISRA C++ (มาตรฐาน coding guideline สำหรับ safety-critical embedded ที่กล่าวถึงใน
  Part 113/115) หลายเวอร์ชันถึงกับ **ห้ามใช้ dynamic memory allocation หลังจากเริ่มระบบ**
  โดยสิ้นเชิงในโค้ดที่ต้องผ่านมาตรฐานความปลอดภัยสูง

### 2. หลายกรณีปิด Exception และ RTTI

`throw`/`catch` ต้องการกลไก stack unwinding ที่ใช้ทั้งพื้นที่ code เพิ่ม (ตาราง unwind
info) และเวลาประมวลผลที่ไม่แน่นอน (non-deterministic) ซึ่งขัดกับความต้องการของระบบ real-time
ที่ต้องตอบสนองภายในเวลาที่แน่นอนเป๊ะ (deterministic timing) วิศวกร embedded จำนวนมากจึง
compile โค้ดด้วย flag `-fno-exceptions -fno-rtti` (ปิด exception และ `dynamic_cast`/`typeid`)
แล้วใช้วิธี error handling แบบ return code หรือ `std::expected` (C++23) / `std::optional`
แทน `throw`

### 3. ไม่มี OS คอยจัดการให้ — ต้องเขียน Startup Code เอง

บน Linux เมื่อโปรแกรมเริ่มทำงาน มี `_start` ที่มาจาก C runtime library (`crt0`) เตรียม stack,
เรียก constructor ของ global object, แล้วค่อยเรียก `main()` — ทั้งหมดนี้ OS/loader จัดการให้
แต่บน bare-metal (ไม่มี OS เลย) **เราต้องเขียนโค้ดส่วนนี้เอง 100%**: ตั้งค่า stack pointer,
copy ค่าเริ่มต้นของตัวแปร global จาก Flash เข้า RAM, เคลียร์ค่า `0` ให้ตัวแปร global
ที่ยังไม่ initialize แล้วค่อยกระโดดไปเรียก `main()` — จะเขียนโค้ดจริงในหัวข้อ 117.4–117.5

### 4. เข้าถึงฮาร์ดแวร์ผ่าน Memory-Mapped Register โดยตรง

บน Linux เราเขียน/อ่านไฟล์ผ่าน system call (`open`, `read`, `write`) ที่ OS แปลไปเป็นการคุย
กับ driver/ฮาร์ดแวร์ให้ แต่บน embedded ที่ไม่มี OS **การควบคุมอุปกรณ์ต่อพ่วง (peripheral)
ทำผ่านการอ่าน/เขียนค่าไปยัง memory address พิเศษที่ CPU ผูกไว้กับวงจรฮาร์ดแวร์โดยตรง**
เรียกว่า **Memory-Mapped I/O** — จะสาธิตจริงในหัวข้อ 117.5–117.6

### สรุปเปรียบเทียบแนวคิดการเขียนโปรแกรม

| แนวคิด | Desktop/Server (ที่เรียนมา) | Embedded Bare-Metal |
|---|---|---|
| จุดเริ่มโปรแกรม | `main()` (OS เตรียม stack/heap ให้แล้ว) | Reset Handler (เราเขียน startup code เอง) |
| Dynamic Memory | `new`/`malloc` ใช้ได้อิสระ | หลีกเลี่ยง หรือใช้ static/pool allocation |
| Error Handling | `throw`/`catch` ใช้ได้เต็มที่ | มักปิด exception ใช้ return code แทน |
| การควบคุม I/O | เรียก system call ผ่าน OS | อ่าน/เขียน memory-mapped register โดยตรง |
| การรัน/debug | รันตรงบนเครื่อง มี `gdb` เต็มรูปแบบ | ต้อง cross-compile + flash ลงบอร์ด หรือจำลองด้วย QEMU |
| จบโปรแกรม | `return`/`exit()` กลับ OS | ไม่มีที่ให้ return ไป — วนลูปตลอดไป (infinite loop) |

---

## 117.3 Cross-Compilation ด้วย gcc-arm-none-eabi (Step 931)

**Cross-Compilation** คือการ compile โค้ดบนเครื่องหนึ่ง (**Host** — เครื่อง x86-64 ที่เราใช้
เขียนโค้ดอยู่นี้) ให้ได้ machine code ที่รันบนเครื่องอีกสถาปัตยกรรมหนึ่ง (**Target** — CPU
ตระกูล ARM ของ microcontroller) เพราะ microcontroller ตัวเล็กๆ ไม่มีกำลังพอจะรัน compiler
เองได้ (และมักไม่มี OS ให้รัน compiler อยู่แล้ว)

`gcc-arm-none-eabi` คือ GCC toolchain ที่ configure มาให้ target เป็น `arm-none-eabi`:

- **`arm`** — สถาปัตยกรรม CPU เป้าหมายคือ ARM
- **`none`** — ไม่มี OS เป้าหมาย (bare-metal — ตรงข้ามกับ `arm-linux-gnueabi` ที่ target
  เป็น Linux บน ARM เช่น Raspberry Pi)
- **`eabi`** — ใช้ Embedded Application Binary Interface (มาตรฐาน ABI สำหรับระบบฝังตัว)

ตรวจสอบว่ามี toolchain นี้พร้อมใช้งานจริงบนเครื่อง:

```bash
arm-none-eabi-gcc --version
```

ผลลัพธ์จริงที่ได้บนเครื่องที่ใช้เขียนบทเรียนนี้:

```
arm-none-eabi-gcc (15:13.2.rel1-2) 13.2.1 20231009
Copyright (C) 2023 Free Software Foundation, Inc.
```

### Flag สำคัญที่ต้องรู้เวลา Compile สำหรับ Bare-Metal

| Flag | ความหมาย |
|---|---|
| `-mcpu=cortex-m3` | ระบุ CPU core เป้าหมายเจาะจง (มีผลต่อ instruction set ที่ compiler เลือกใช้) |
| `-mthumb` | ใช้ Thumb instruction set (โค้ดกะทัดรัดกว่า ARM mode ปกติ, Cortex-M รองรับแค่ Thumb/Thumb-2) |
| `-ffreestanding` | บอก compiler ว่านี่คือ **freestanding environment** (ไม่มี OS, ไม่มี standard library เต็มรูปแบบ) compiler จะไม่สมมติว่ามี `main()` แบบปกติหรือมี libc ครบ |
| `-nostdlib` | ไม่ link กับ standard library และ startup file ปกติ (เราเขียน startup เอง) |
| `-T linker.ld` | ระบุ Linker Script กำหนดว่าแต่ละส่วนของโปรแกรมจะถูกวางที่ address ไหนใน memory จริง |

ความแตกต่างสำคัญจาก `gcc` ปกติที่ใช้ตลอดหลักสูตร: `gcc` ธรรมดา target คือ `x86_64-linux-gnu`
compile แล้วรันได้ทันทีบนเครื่องปัจจุบัน (Native Compilation) แต่ `arm-none-eabi-gcc`
compile แล้ว **ไฟล์ที่ได้รันบนเครื่อง x86-64 นี้ไม่ได้เลย** เพราะเป็น machine code ของ CPU
คนละตระกูล (ARM) ต้องเอาไป flash ลงบอร์ดจริง หรือรันผ่านตัวจำลองอย่าง QEMU เท่านั้น

---

## 117.4 กายวิภาคของโปรแกรม Bare-Metal (Step 932)

ก่อนเขียนโปรแกรมจริง ต้องเข้าใจ 3 ส่วนประกอบหลักที่ทำให้โปรแกรม bare-metal "บูท" ขึ้นมาได้:

### 1. Vector Table

CPU ตระกูล ARM Cortex-M เมื่อเปิดเครื่อง (Reset) จะไปอ่านค่า 2 ค่าแรกจาก address `0x00000000`
เสมอ:

- **Word แรก (offset 0)**: ค่าเริ่มต้นของ **Stack Pointer**
- **Word ที่สอง (offset 4)**: address ของฟังก์ชันที่จะเรียกเป็นอันดับแรก (**Reset Handler**)

ตามด้วยตารางของ address ฟังก์ชันสำหรับจัดการ interrupt/exception ต่างๆ (NMI, Hard Fault
ฯลฯ) เรียกรวมกันว่า **Vector Table** — เขียนเป็น array ของ function pointer ใน C ได้ตรงๆ

### 2. Reset Handler และการเตรียม .data/.bss

หน้าที่ของ Reset Handler คือทำสิ่งที่ C runtime (`crt0`) เคยทำให้บน Linux:

1. Copy ค่าเริ่มต้นของตัวแปร global ที่ initialize ไว้ (section `.data`) จาก Flash เข้า RAM
   (เพราะ CPU รันโค้ดจาก Flash แต่ตัวแปรต้องแก้ไขได้ ต้องอยู่ใน RAM)
2. เคลียร์ตัวแปร global ที่ไม่ได้ initialize (section `.bss`) ให้เป็น `0` ทั้งหมด (มาตรฐาน
   ภาษา C/C++ รับประกันว่าตัวแปร global uninitialized ต้องเป็น 0 — บน Linux OS/loader ทำให้
   แต่บน bare-metal เราต้องทำเอง)
3. เรียก `main()`

### 3. Linker Script

ไฟล์ที่บอก linker ว่าแต่ละ section ของโปรแกรม (`.isr_vector`, `.text`, `.data`, `.bss`)
ควรถูกวางไว้ที่ address ไหนของ memory จริง (Flash เริ่มที่ไหน, RAM เริ่มที่ไหน, มีขนาดเท่าไหร่)
ต่างจากโปรแกรม Linux ทั่วไปที่ linker ใช้ script default ที่ OS/loader เตรียมไว้ให้ โดยไม่ต้อง
เขียนเอง — บน bare-metal เราต้องรู้ **memory map ที่แท้จริงของฮาร์ดแวร์เป้าหมาย** และเขียน
Linker Script เองเสมอ

---

## 117.5 สร้างและรันโปรแกรม "Hello UART" บน QEMU จริง (Step 933)

ถึงเวลาลงมือจริง เราจะเขียนโปรแกรม bare-metal สำหรับบอร์ดจำลอง **Stellaris LM3S6965EVB**
(CPU: ARM Cortex-M3) ที่ QEMU รองรับโดยตรง แล้วให้โปรแกรมพิมพ์ข้อความผ่าน UART (การสื่อสาร
แบบอนุกรมที่ใช้กันทั่วไปในวงการ embedded สำหรับ debug/log)

### ไฟล์ที่ 1: uart.h — Memory-Mapped Register ของ UART0

```cpp
#ifndef UART_H
#define UART_H
#include <stdint.h>

#define UART0_BASE   0x4000C000u
#define UART0_DR     (*(volatile uint32_t *)(UART0_BASE + 0x000))  /* Data Register */
#define UART0_FR     (*(volatile uint32_t *)(UART0_BASE + 0x018))  /* Flag Register */
#define UART_FR_TXFF (1u << 5)  /* bit 5: Transmit FIFO Full */

static inline void uart_putc(char c) {
    while (UART0_FR & UART_FR_TXFF) { }   /* รอจนกว่า FIFO จะว่างพอให้ส่งได้ */
    UART0_DR = (uint32_t)c;
}

static inline void uart_puts(const char *s) {
    while (*s) { uart_putc(*s++); }
}

#endif
```

`0x4000C000` คือ **base address** ของอุปกรณ์ UART0 บนชิป LM3S6965 — นี่ไม่ใช่ address
ของ RAM ปกติ แต่เป็น address พิเศษที่วงจรของชิปผูกไว้กับวงจร UART โดยตรง เมื่อ CPU เขียนค่า
ไปที่ address `UART0_DR` วงจรฮาร์ดแวร์จะส่งค่านั้นออกทางสาย TX ทันที ไม่ใช่แค่เก็บใน RAM

สังเกตคำว่า **`volatile`** ที่อยู่หน้า `uint32_t*` ทุกจุด — นี่คือคีย์เวิร์ดที่สำคัญที่สุด
คำหนึ่งของการเขียนโปรแกรม embedded: มันบอก compiler ว่า **ค่าที่ address นี้อาจเปลี่ยนแปลง
ได้จากภายนอกโปรแกรม (จากฮาร์ดแวร์) โดยที่ compiler มองไม่เห็น** ห้าม optimize การอ่าน/เขียน
ทิ้งหรือเรียงลำดับใหม่เด็ดขาด ถ้าลืมใส่ `volatile` compiler อาจมองว่า `while (UART0_FR &
UART_FR_TXFF)` เป็น infinite loop ที่ไม่มีประโยชน์ (เพราะมันไม่เห็นว่าค่าจะเปลี่ยนจากไหน)
แล้ว optimize จนโปรแกรม hang จริงหรือพฤติกรรมผิดเพี้ยนไปเลย

### ไฟล์ที่ 2: startup.c — Vector Table และ Reset Handler

```cpp
/* startup.c - minimal Cortex-M3 startup + vector table for QEMU lm3s6965evb */
#include <stdint.h>

extern uint32_t _sidata, _sdata, _edata, _sbss, _ebss, _estack;
void Reset_Handler(void);
void Default_Handler(void);
int main(void);

/* Vector table: ต้องอยู่ที่ address 0x00000000 (จุดเริ่มโหลดของ Cortex-M) */
__attribute__((section(".isr_vector")))
void (* const vector_table[])(void) = {
    (void (*)(void))&_estack,   /* [0] Initial Stack Pointer */
    Reset_Handler,              /* [1] Reset Handler */
    Default_Handler,            /* [2] NMI */
    Default_Handler,            /* [3] Hard Fault */
};

void Reset_Handler(void) {
    uint32_t *src = &_sidata;
    uint32_t *dst = &_sdata;
    while (dst < &_edata) { *dst++ = *src++; }  /* copy .data จาก Flash เข้า RAM */
    dst = &_sbss;
    while (dst < &_ebss) { *dst++ = 0; }        /* เคลียร์ .bss เป็น 0 */
    main();
    while (1) { }   /* ถ้า main() return กลับมา ก็วนลูปเฉยๆ ไม่มีที่ให้ไปต่อ */
}

void Default_Handler(void) {
    while (1) { }
}
```

ตัวแปร `_sidata`, `_sdata`, `_edata`, `_sbss`, `_ebss`, `_estack` ไม่ได้ประกาศไว้ที่ไหนในไฟล์
C เลย — มันคือ **symbol ที่ linker script จะกำหนดค่าให้** (คือ address จริงในหน่วยความจำ
ไม่ใช่ตัวแปรที่มีค่าเก็บอยู่จริง) เทคนิคนี้เป็นวิธีมาตรฐานที่ startup code ของบอร์ด embedded
เกือบทุกยี่ห้อใช้

### ไฟล์ที่ 3: main.c

```cpp
#include "uart.h"

int main(void) {
    uart_puts("Hello from bare-metal ARM Cortex-M3 on QEMU!\r\n");
    uart_puts("Module J - Embedded Systems demo\r\n");
    while (1) { }   /* embedded program ไม่มีที่ให้ return ไป จึงวนลูปตลอดไป */
    return 0;
}
```

### ไฟล์ที่ 4: linker.ld

```
ENTRY(Reset_Handler)

MEMORY
{
    FLASH (rx)  : ORIGIN = 0x00000000, LENGTH = 256K
    RAM   (rwx) : ORIGIN = 0x20000000, LENGTH = 64K
}

_estack = ORIGIN(RAM) + LENGTH(RAM);

SECTIONS
{
    .isr_vector :
    {
        KEEP(*(.isr_vector))
    } > FLASH

    .text :
    {
        *(.text*)
        *(.rodata*)
    } > FLASH

    _sidata = LOADADDR(.data);

    .data :
    {
        _sdata = .;
        *(.data*)
        _edata = .;
    } > RAM AT> FLASH

    .bss :
    {
        _sbss = .;
        *(.bss*)
        *(COMMON)
        _ebss = .;
    } > RAM
}
```

จุดที่ควรสังเกต:

- `MEMORY { FLASH ... RAM ... }` กำหนด memory map จริงของชิป LM3S6965: Flash เริ่มที่
  `0x00000000` ขนาด 256 KB, RAM เริ่มที่ `0x20000000` ขนาด 64 KB (ตัวเลขนี้มาจาก datasheet
  ของชิปจริง ไม่ใช่ค่าที่เดาเอง)
- `_estack = ORIGIN(RAM) + LENGTH(RAM);` — กำหนดให้ stack เริ่มจากปลายสุดของ RAM แล้วโตลง
  มาหา address ต่ำ (ธรรมเนียมมาตรฐานของ ARM: stack โตจากบนลงล่าง)
- `.data : { ... } > RAM AT> FLASH` — บอกว่าตอน **รันจริง (VMA)** section นี้อยู่ใน RAM
  แต่ตอน **โหลดจริง (LMA)** ค่าเริ่มต้นถูกเก็บไว้ใน Flash แล้ว Reset Handler เป็นคนย้ายมา
  ตอนบูท (ตรงกับโค้ดใน `startup.c` ที่ copy จาก `_sidata` ไปยัง `_sdata`)

### Compile และ Link จริง

```bash
CPU_FLAGS="-mcpu=cortex-m3 -mthumb"
arm-none-eabi-gcc $CPU_FLAGS -Wall -Wextra -ffreestanding -O2 -c startup.c -o startup.o
arm-none-eabi-gcc $CPU_FLAGS -Wall -Wextra -ffreestanding -O2 -c main.c -o main.o
arm-none-eabi-gcc $CPU_FLAGS -ffreestanding -nostdlib -T linker.ld startup.o main.o \
    -o firmware.elf -Wl,-Map=firmware.map
arm-none-eabi-size firmware.elf
```

ผลลัพธ์จริงที่ได้ (compile ผ่านสะอาด ไม่มี warning เลย):

```
   text	   data	    bss	    dec	    hex	filename
    251	      0	      0	    251	     fb	firmware.elf
```

`firmware.elf` มีขนาดโค้ดทั้งหมดเพียง **251 bytes** — เทียบกับโปรแกรม "Hello, World!" บน
Linux ที่ link กับ dynamic library แล้วมักมีขนาดหลายสิบ KB ความต่างของขนาดนี้สะท้อนข้อจำกัด
เรื่อง Flash/RAM ที่จำกัดมากของ embedded ได้ชัดเจน

### รันบน QEMU จริง

```bash
qemu-system-arm -M lm3s6965evb -nographic -kernel firmware.elf
```

`-M lm3s6965evb` บอก QEMU ให้จำลองบอร์ด Stellaris LM3S6965EVB ทั้งบอร์ด (ไม่ใช่แค่ CPU
เฉยๆ — รวมถึงวงจร UART ที่ mapped ไว้ที่ `0x4000C000` ตรงกับที่เราเขียนโค้ดไว้พอดี)
`-nographic` บอกให้ QEMU ไม่เปิดหน้าต่างกราฟิก แต่ต่อ serial console (UART) เข้ากับ terminal
ที่รันคำสั่งอยู่แทน `-kernel firmware.elf` บอกให้โหลดไฟล์ ELF ของเราเข้าไปรันเป็น firmware

**ผลลัพธ์จริงที่ได้จากการรันบน QEMU** (capture จริงจาก terminal, ไม่ใช่การจำลอง):

```
Timer with period zero, disabling
Hello from bare-metal ARM Cortex-M3 on QEMU!
Module J - Embedded Systems demo
```

(บรรทัด "Timer with period zero, disabling" เป็นข้อความ debug จาก QEMU เองเกี่ยวกับ
watchdog timer ของบอร์ดจำลองที่เรายังไม่ได้ตั้งค่า ไม่ใช่ error และไม่กระทบการทำงานของ
โปรแกรม) ข้อความสองบรรทัดถัดมาคือสิ่งที่โปรแกรมของเราพิมพ์ผ่าน UART จริง — พิสูจน์ว่า
Vector Table ทำงานถูกต้อง, Reset Handler เตรียม memory สำเร็จ, `main()` ถูกเรียกจริง และ
การเขียนค่าไปยัง memory-mapped register ของ UART ทำให้ QEMU (ซึ่งจำลองวงจร UART จริง)
ส่งข้อความออกมาที่ terminal ได้จริง เพราะโปรแกรมมี `while(1)` ที่ท้าย `main()` จึงต้อง
กด `Ctrl+A` แล้วตามด้วย `X` เพื่อออกจาก QEMU (หรือใช้ `timeout` เวลาทดสอบอัตโนมัติ)

นี่คือ pipeline แบบเดียวกับที่วิศวกร embedded มืออาชีพหลายคนใช้ในการพัฒนาเบื้องต้นก่อนจะ
flash ลงบอร์ดจริง: เขียนโค้ด → cross-compile → รันบน QEMU → ถ้าถูกต้องค่อยเอาไป flash ลง
ฮาร์ดแวร์จริงและทดสอบซ้ำอีกครั้ง (เพราะ QEMU จำลองพฤติกรรมได้ไม่ครบ 100% กับฮาร์ดแวร์จริง
โดยเฉพาะ timing และ peripheral แปลกๆ ที่ QEMU ไม่รองรับ)

---

## 117.6 รูปแบบการเข้าถึง Memory-Mapped Register ที่ใช้จริงในวงการ (Step 934)

ตัวอย่างในหัวข้อ 117.5 ใช้ macro กำหนด address ตรงๆ ซึ่งใช้ได้ผลแต่เมื่อ peripheral มี
register หลายตัวเรียงติดกัน (เช่น UART จริงมักมีมากกว่า 10 register) การเขียน macro แยกทีละ
ตัวจะซ้ำซ้อนมาก วงการ embedded นิยมใช้ `struct` แทนเพื่อให้โค้ดอ่านง่ายและตรงกับที่ datasheet
ของชิปอธิบายไว้:

```cpp
#include <stdint.h>

/* จำลอง layout ของ UART peripheral (เรียงตาม offset จริงในสเปคของชิป) */
typedef struct {
    volatile uint32_t DR;      /* offset 0x000: Data Register */
    volatile uint32_t RSR_ECR; /* offset 0x004: Receive Status / Error Clear */
    uint32_t RESERVED0[4];     /* offset 0x008-0x014: ช่องว่างตามสเปคของชิป */
    volatile uint32_t FR;      /* offset 0x018: Flag Register */
} UART_TypeDef;

#define UART0  ((UART_TypeDef *)0x4000C000u)

/* บิตในรีจิสเตอร์ FR (Flag Register) */
#define UART_FR_TXFF  (1u << 5)

void uart_send_char(char c) {
    while (UART0->FR & UART_FR_TXFF) { }
    UART0->DR = (uint32_t)c;
}
```

รูปแบบนี้ (`struct` ที่แต่ละ field ตรงกับ register ตาม offset ในสเปคของชิปเป๊ะ แล้ว cast
address ของฮาร์ดแวร์เป็น pointer ไปยัง struct นั้น) คือรูปแบบมาตรฐานที่ **CMSIS** (Cortex
Microcontroller Software Interface Standard — มาตรฐานที่ ARM กำหนดให้ผู้ผลิตชิปทุกเจ้าที่ใช้
core Cortex-M ต้อง provide header ไฟล์แบบนี้ให้) ใช้จริงในทุก vendor (STMicroelectronics,
Nordic Semiconductor, Texas Instruments ฯลฯ) — เวลาผู้เรียนไปเปิด header ไฟล์ของบอร์ด
STM32 จริงๆ (เช่น `stm32f4xx.h`) จะเห็น pattern แบบนี้ทุก peripheral (GPIO, UART, SPI, I2C,
Timer) นับสิบตัว

**ข้อควรระวังเรื่อง `RESERVED0[4]`**: ไม่ได้ใส่ `volatile` เพราะไม่มีการอ่าน/เขียนจริง มันมี
ไว้แค่ "กินพื้นที่" ใน struct ให้ offset ของ field ถัดไป (`FR`) ตรงกับตำแหน่งจริงในฮาร์ดแวร์
เท่านั้น — และต้องระวังเรื่อง **struct padding** ที่ compiler อาจแทรกเพิ่มโดยไม่รู้ตัว ถ้า
field มีขนาดไม่เท่ากัน (ในตัวอย่างนี้ทุก field เป็น `uint32_t` เท่ากันหมดจึงไม่มีปัญหา แต่ถ้า
ผสม `uint8_t`/`uint32_t` ต้องใช้ `#pragma pack` หรือ `__attribute__((packed))` ระวังเรื่องนี้
เสมอเวลาออกแบบ struct สำหรับ memory-mapped I/O)

---

## 117.7 RTOS คืออะไร และทำไมบางครั้งจึงต้องใช้ (Step 935)

โปรแกรมที่เขียนในหัวข้อ 117.5 เป็นรูปแบบที่เรียกว่า **Super Loop** (หรือ Bare-Metal Loop):
`main()` ทำงานเป็น loop เดียวไม่มีที่สิ้นสุด จัดการทุกอย่างตามลำดับ (ทีละ task) เหมาะกับงาน
ง่ายๆ ที่ไม่ต้องทำหลายอย่างพร้อมกันจริงจัง

แต่เมื่อระบบซับซ้อนขึ้น (ต้องอ่านเซนเซอร์ทุก 10ms **พร้อมกับ** รับคำสั่งผ่าน UART **พร้อมกับ**
ควบคุมมอเตอร์แบบ real-time) การจัดการทุกอย่างในลูปเดียวจะยุ่งยากและเสี่ยงต่อการที่งานหนึ่งทำ
นานเกินไปจนงานอื่น "ไม่ได้รับการดูแล" ทันเวลา — นี่คือจุดที่ **RTOS (Real-Time Operating
System)** เข้ามาช่วย

RTOS ไม่ใช่ OS เต็มรูปแบบแบบ Linux (ไม่มี virtual memory, ไม่มี filesystem เต็มรูปแบบ, ไม่มี
process แยก address space) แต่ให้บริการหลักที่จำเป็น:

- **Task Scheduling**: สลับการทำงานระหว่างหลาย "task" (คล้าย thread) ตาม priority ที่กำหนด
  รับประกันว่า task สำคัญจะได้ CPU time ตรงเวลาที่ต้องการ (deterministic timing)
- **Synchronization Primitives**: Semaphore, Mutex, Queue สำหรับสื่อสารและป้องกัน race
  condition ระหว่าง task (แนวคิดเดียวกับที่เรียนใน Part 32 เรื่อง Thread Synchronization
  แต่ implement ในสเกลที่เล็กกว่ามาก เหมาะกับ RAM หลัก KB)
- **Timing Services**: delay, timeout, periodic task ที่แม่นยำระดับ millisecond

### FreeRTOS โดยย่อ

**FreeRTOS** คือ RTOS แบบ open-source ที่ได้รับความนิยมสูงที่สุดตัวหนึ่งในโลก embedded
(ปัจจุบันดูแลโดย Amazon Web Services ในชื่อ AWS FreeRTOS) จุดเด่นคือ footprint เล็กมาก
(kernel หลักใช้ RAM แค่ไม่กี่ KB) รองรับ CPU หลากหลายตระกูล (ARM Cortex-M, RISC-V, ESP32
ฯลฯ) และมี ecosystem ของ library เสริมเยอะ (TCP/IP stack, security)

โครงสร้างโค้ดที่ใช้ FreeRTOS หน้าตาต่างจาก bare-metal loop ชัดเจน (โค้ดตัวอย่างนี้เป็นแนวคิด
โดยสังเขป — ไม่ได้ compile ทดสอบจริงในบทเรียนนี้ เพราะต้องมี FreeRTOS source/config เพิ่มเติม
และ port layer เฉพาะบอร์ด ซึ่งเกินขอบเขต "ภาพรวม" ของ Part นี้):

```cpp
#include "FreeRTOS.h"
#include "task.h"

void sensor_task(void *pvParameters) {
    (void)pvParameters;
    for (;;) {
        /* อ่านค่าเซนเซอร์ ... */
        vTaskDelay(pdMS_TO_TICKS(10));  /* หน่วงเวลา 10ms แบบไม่กิน CPU ทิ้งเปล่า */
    }
}

void uart_task(void *pvParameters) {
    (void)pvParameters;
    for (;;) {
        /* รับคำสั่งผ่าน UART ... */
        vTaskDelay(pdMS_TO_TICKS(5));
    }
}

int main(void) {
    xTaskCreate(sensor_task, "Sensor", 128, NULL, 2, NULL); /* priority 2 */
    xTaskCreate(uart_task,   "UART",   128, NULL, 1, NULL); /* priority 1 */
    vTaskStartScheduler();  /* เริ่ม scheduler — จุดนี้ไป "ไม่กลับ" เหมือน bare-metal loop */
    for (;;) { }            /* ไม่ควรมาถึงจุดนี้ถ้า scheduler ทำงานปกติ */
}
```

### เมื่อไหร่ควรใช้ Bare-Metal, RTOS หรือ Linux เต็มรูปแบบ

| สถานการณ์ | ทางเลือกที่เหมาะสม |
|---|---|
| งานง่าย ทำตามลำดับได้ ไม่ต้องแยก task | Bare-Metal Super Loop |
| ต้องทำหลายงานพร้อมกันแบบมี timing เข้มงวด, RAM จำกัดมาก (< 1 MB) | RTOS (เช่น FreeRTOS, Zephyr) |
| ต้องการ networking เต็มรูปแบบ, filesystem, GUI ซับซ้อน, RAM เหลือเฟือ (หลาย MB ขึ้นไป) | Embedded Linux (เช่น Raspberry Pi, Yocto Project) |

---

## 117.8 การ Debug โปรแกรม Embedded ในโลกจริง (Step 936)

Debug โปรแกรม embedded ยากกว่าโปรแกรม Desktop เพราะเรามักไม่มีจอ, คีย์บอร์ด, หรือแม้แต่
`printf` ที่ใช้งานได้ตรงๆ (เพราะ `printf` มาตรฐานต้องพึ่ง `stdout` ที่ผูกกับ OS) เทคนิคหลักที่
ใช้กันจริงในวงการมีดังนี้:

### 1. JTAG / SWD (Serial Wire Debug)

บอร์ดจริงส่วนใหญ่มีพอร์ต JTAG หรือ SWD (มาตรฐานของ ARM ที่ใช้สาย 2 เส้น) ให้ต่อกับ debug
probe (เช่น ST-Link, J-Link) แล้วใช้ GDB ควบคุม CPU ได้เหมือนกับ debug โปรแกรม Desktop
เกือบทุกอย่าง (ตั้ง breakpoint, step, ดูค่า register/memory แบบ real-time) — นี่คือวิธี
มาตรฐานที่ใช้กับฮาร์ดแวร์จริง

### 2. QEMU GDB Stub — Debug โดยไม่ต้องมีบอร์ดจริง

QEMU มีความสามารถพิเศษคือเปิด **GDB server ในตัว** ทำให้ debug โปรแกรมที่รันอยู่ใน QEMU ได้
เหมือนมีฮาร์ดแวร์จริงต่ออยู่ วิธีใช้:

```bash
# เปิด QEMU แล้วหยุดรอที่คำสั่งแรกทันที (-S) พร้อมเปิด gdbserver ที่ port 1234 (-s)
qemu-system-arm -M lm3s6965evb -nographic -kernel firmware.elf -s -S
```

จากนั้นในอีก terminal หนึ่ง เชื่อมต่อด้วย GDB (ต้องใช้ GDB ที่รองรับสถาปัตยกรรม ARM เช่น
`gdb-multiarch` บน Linux หรือ `arm-none-eabi-gdb` ที่มากับ toolchain):

```
(gdb) target remote localhost:1234
(gdb) break main
(gdb) continue
(gdb) info registers
(gdb) print/x _estack
```

ก็จะ debug โค้ดที่รันอยู่ใน QEMU ได้เหมือนโปรแกรมปกติทุกประการ — วิธีนี้มีประโยชน์มากตอน
พัฒนาเบื้องต้นเพราะไม่ต้องมี debug probe หรือบอร์ดจริงเลย

### 3. Semihosting

กลไกพิเศษที่ให้โปรแกรมบน ARM ที่ต่อกับ debugger (หรือรันบน QEMU) เรียก system call พื้นฐาน
บนเครื่อง host ได้ผ่าน debug interface (เช่น `printf` ไปโผล่ที่ terminal ของเครื่อง host จริง
โดยไม่ต้องมี UART) สะดวกมากตอนพัฒนา แต่ **ช้ามาก** (ทุกครั้งที่เรียกต้องหยุด CPU คุยกับ
debugger) จึงไม่ใช้ใน production firmware จริง เหมาะแค่ตอน debug ระหว่างพัฒนาเท่านั้น

### 4. Logic Analyzer / Oscilloscope

เมื่อปัญหาเกี่ยวข้องกับ timing ของสัญญาณไฟฟ้าจริง (เช่น protocol การสื่อสารกับเซนเซอร์ทำงาน
ผิดจังหวะ) เครื่องมือ software อย่างเดียวไม่พอ ต้องใช้ Logic Analyzer จับสัญญาณดิจิทัลจริงบน
ขาไฟฟ้าของชิปมาดูเทียบกับสิ่งที่โค้ดควรทำ — เป็นทักษะเฉพาะทางที่ต้องเรียนรู้เพิ่มเมื่อทำงาน
กับฮาร์ดแวร์จริง

### 5. UART Logging (วิธีที่ใช้บ่อยที่สุดในทางปฏิบัติ)

ท้ายที่สุดแล้ว วิธีที่ใช้บ่อยและง่ายที่สุดในการพัฒนาจริงคือสิ่งที่เราทำไปแล้วในหัวข้อ 117.5:
พิมพ์ log ผ่าน UART ออกไปให้เครื่อง host อ่าน (ผ่านสาย USB-to-Serial adapter บนบอร์ดจริง หรือ
ผ่าน `-nographic` บน QEMU) — เรียบง่าย เชื่อถือได้ และไม่ต้องพึ่งเครื่องมือพิเศษเพิ่มเติม

---

## ข้อผิดพลาดที่พบบ่อย (Common Pitfalls)

1. **ลืมใส่ `volatile` กับ memory-mapped register** — compiler จะ optimize การอ่าน/เขียนทิ้ง
   หรือ cache ค่าไว้ใน register ของ CPU แทนที่จะอ่าน/เขียนจริงทุกครั้ง ทำให้โปรแกรม hang
   หรือพฤติกรรมผิดเพี้ยนแบบที่ debug ยากมาก (โดยเฉพาะเมื่อเปิด optimization ระดับสูง `-O2`
   ขึ้นไป — โค้ดที่ไม่มี `volatile` มักทำงาน "ถูกโดยบังเอิญ" ตอน `-O0` แต่พังทันทีตอน `-O2`)
2. **ใช้ dynamic allocation (`new`/`malloc`) แบบไม่ระวังใน loop หลัก** — เสี่ยง heap
   fragmentation ที่ทำให้โปรแกรม crash หลังทำงานไปนานเป็นเดือน ทั้งที่ตอน test ช่วงแรกดูปกติดี
3. **สมมติว่า stack ใหญ่พอเสมอ** — RAM มีจำกัดมาก การเรียก recursive function ลึกๆ หรือ
   ประกาศ local array ขนาดใหญ่ในฟังก์ชันอาจทำให้ stack overflow ทับข้อมูลส่วนอื่นของ RAM
   โดยไม่มี OS คอย detect ให้เหมือนบน Linux (ไม่มี segmentation fault ให้เห็น แค่พฤติกรรม
   ประหลาดเกิดขึ้นเงียบๆ)
4. **ลืมว่า Cortex-M บางรุ่นไม่มี FPU** — การใช้ `float`/`double` บนชิปที่ไม่มี Hardware FPU
   จะถูกแปลงเป็นการเรียกฟังก์ชัน software floating point ที่ช้ามาก (ช้ากว่าฮาร์ดแวร์ FPU
   หลายสิบเท่า) ควรตรวจสอบสเปคชิปก่อนออกแบบ algorithm ที่ใช้ floating point หนักๆ
5. **เขียน Linker Script ผิดจน section ทับกัน** — ถ้า `.text` ใหญ่เกินขนาด Flash จริง หรือ
   `.data`/`.bss` รวมกันเกินขนาด RAM จริง โปรแกรมจะ corrupt memory โดยไม่มี error ชัดเจน
   ตอน compile (linker บางครั้งเตือน แต่บางครั้งก็ไม่ ต้องเช็คด้วย `size` เทียบกับสเปคจริงเสมอ)
6. **ทดสอบบน QEMU แล้วมั่นใจว่าใช้ได้กับฮาร์ดแวร์จริงแน่นอน** — QEMU จำลอง CPU และ
   peripheral หลักได้ดี แต่ไม่ครอบคลุมทุก edge case ของฮาร์ดแวร์จริง (timing แปลกๆ, สัญญาณ
   รบกวนทางไฟฟ้า, ความคลาดเคลื่อนของ clock) ต้องทดสอบบนฮาร์ดแวร์จริงเสมอก่อนขึ้น production

---

## แบบฝึกหัดท้ายบท

1. แก้ไข `main.c` จากหัวข้อ 117.5 ให้พิมพ์ตัวเลขนับ 1 ถึง 5 ผ่าน UART (ต้องเขียนฟังก์ชันแปลง
   `int` เป็น string เอง เพราะไม่มี `printf`/`itoa` ให้ใช้ใน environment แบบ freestanding นี้)
   แล้ว cross-compile และรันบน QEMU จริงเพื่อยืนยันผลลัพธ์

2. เพิ่ม delay function อย่างง่าย (busy-wait loop นับรอบ) ระหว่างการพิมพ์แต่ละบรรทัดใน
   `main.c` แล้วอธิบายว่าทำไมวิธีนี้ (busy-wait) ไม่เหมาะกับระบบที่ต้องทำงานหลายอย่างพร้อมกัน
   เมื่อเทียบกับการใช้ RTOS

3. เขียน struct แบบ CMSIS-style (ตามหัวข้อ 117.6) สำหรับ peripheral สมมติที่มี register
   3 ตัว: `CTRL` (offset 0x00), `STATUS` (offset 0x04), `DATA` (offset 0x08) แล้วเขียน
   ฟังก์ชันเปิดใช้งาน peripheral นี้โดยตั้งบิตที่ 0 ของ `CTRL`

4. อธิบายด้วยคำพูดตัวเองว่าทำไม `Reset_Handler` ต้อง copy `.data` จาก Flash มา RAM ก่อน
   แต่ไม่ต้องทำแบบเดียวกันกับ `.text` (โค้ดของโปรแกรม)

5. ลองรัน `qemu-system-arm -M lm3s6965evb -nographic -kernel firmware.elf -s -S` แล้วต่อ
   ด้วย `gdb-multiarch` (หรือ debugger ที่มี) ตั้ง breakpoint ที่ `main` แล้วดูค่า register
   `pc` ตอนหยุด อธิบายว่าค่าที่เห็นสอดคล้องกับ Vector Table ที่เขียนไว้อย่างไร

6. เปรียบเทียบให้เห็นภาพ: เขียนตารางสรุป (RAM, มี OS หรือไม่, มี exception หรือไม่, ตัวอย่าง
   การใช้งานจริง) ของ 3 ระดับ: (ก) bare-metal microcontroller อย่างในบทนี้ (ข) Raspberry Pi
   ที่รัน embedded Linux (ค) เครื่อง server ที่เราใช้เขียนโค้ดมาตลอดหลักสูตร

### แนวทางเฉลยข้อ 1

```cpp
/* main.c ฉบับแก้ไข */
#include "uart.h"

static void uart_put_digit(int d) {
    uart_putc((char)('0' + d));
}

int main(void) {
    uart_puts("Counting from 1 to 5:\r\n");
    for (int i = 1; i <= 5; ++i) {
        uart_put_digit(i);
        uart_puts("\r\n");
    }
    uart_puts("Done!\r\n");
    while (1) { }
    return 0;
}
```

Compile และ link ใหม่ด้วยคำสั่งเดิมจากหัวข้อ 117.5 (ใช้ `startup.c`, `uart.h`, `linker.ld`
ชุดเดิม เปลี่ยนแค่ `main.c`):

```bash
CPU_FLAGS="-mcpu=cortex-m3 -mthumb"
arm-none-eabi-gcc $CPU_FLAGS -Wall -Wextra -ffreestanding -O2 -c startup.c -o startup.o
arm-none-eabi-gcc $CPU_FLAGS -Wall -Wextra -ffreestanding -O2 -c main.c -o main.o
arm-none-eabi-gcc $CPU_FLAGS -ffreestanding -nostdlib -T linker.ld startup.o main.o -o firmware.elf
qemu-system-arm -M lm3s6965evb -nographic -kernel firmware.elf
```

ผลลัพธ์จริงที่ได้จากการรันบน QEMU (ทดสอบจริงด้วย pipeline เดียวกับหัวข้อ 117.5):

```
Timer with period zero, disabling
Counting from 1 to 5:
1
2
3
4
5
Done!
```

### แนวทางเฉลยข้อ 3

```cpp
#include <stdint.h>

typedef struct {
    volatile uint32_t CTRL;
    volatile uint32_t STATUS;
    volatile uint32_t DATA;
} MyPeripheral_TypeDef;

#define MY_PERIPH  ((MyPeripheral_TypeDef *)0x40020000u)

#define MY_PERIPH_CTRL_ENABLE  (1u << 0)

void my_peripheral_enable(void) {
    MY_PERIPH->CTRL |= MY_PERIPH_CTRL_ENABLE;
}
```

จุดสำคัญของเฉลยนี้: ใช้ `|=` (OR แล้วเซ็ตกลับ) แทนการเขียนทับค่าทั้ง register ตรงๆ
(`MY_PERIPH->CTRL = MY_PERIPH_CTRL_ENABLE;`) เพราะ register ควบคุมของฮาร์ดแวร์จริงมักมี
หลายบิตที่ควบคุมฟีเจอร์ต่างกันอยู่ใน register เดียวกัน การเขียนทับทั้ง register จะไปลบล้าง
การตั้งค่าบิตอื่นที่อาจถูกตั้งไว้ก่อนหน้าโดยไม่ได้ตั้งใจ — เป็นข้อควรระวังที่สำคัญมากใน
โลกการเขียนโปรแกรมควบคุมฮาร์ดแวร์จริง

---

## สรุปท้ายบท

ใน Part นี้เราได้:

- เข้าใจว่า Embedded System คืออะไร ต่างจาก Desktop/Server อย่างไรในแง่ทรัพยากร (RAM,
  Flash, การมี/ไม่มี OS)
- เข้าใจข้อจำกัดสำคัญของการเขียนโปรแกรม embedded: หลีกเลี่ยง dynamic allocation, มักปิด
  exception/RTTI, ต้องเขียน startup code เอง, ควบคุมฮาร์ดแวร์ผ่าน memory-mapped register
- ใช้ `gcc-arm-none-eabi` cross-compile โปรแกรมสำหรับ ARM Cortex-M ได้จริงบนเครื่อง x86-64
- เข้าใจกายวิภาคของโปรแกรม bare-metal: Vector Table, Reset Handler, Linker Script และ
  ความสัมพันธ์ระหว่าง section `.text`/`.data`/`.bss` กับ Flash/RAM จริง
- **สร้างและรันโปรแกรม bare-metal จริงบน QEMU สำเร็จ** — เห็นข้อความ "Hello from bare-metal
  ARM Cortex-M3 on QEMU!" ปรากฏจริงผ่าน UART ที่จำลองโดย QEMU
- เข้าใจรูปแบบ CMSIS-style struct สำหรับเข้าถึง peripheral register ที่ใช้จริงในวงการ
  และเหตุผลของ `volatile`
- รู้จัก RTOS (โดยเฉพาะ FreeRTOS) ในภาพรวม และรู้ว่าเมื่อไหร่ควรเลือก bare-metal, RTOS,
  หรือ embedded Linux
- รู้จักเทคนิค debug โปรแกรม embedded ในโลกจริง ทั้ง JTAG/SWD, QEMU GDB stub, semihosting,
  และ UART logging

Part นี้เป็นบท "เปิดโลก" ให้เห็นภาพรวมของสายงาน embedded เท่านั้น — การเป็นวิศวกร embedded
มืออาชีพต้องเรียนรู้เพิ่มอีกมาก (RTOS แบบเจาะลึก, low-power design, communication protocol
เช่น I2C/SPI/CAN, safety standard เช่น MISRA C/C++ ที่กล่าวถึงใน Part 115, และการทำงานกับ
ฮาร์ดแวร์จริงที่มีข้อจำกัดทางไฟฟ้า) แหล่งเรียนรู้ต่อยอดที่แนะนำ ได้แก่ เอกสาร ARM Cortex-M
Programming Manual, คอร์ส FreeRTOS อย่างเป็นทางการจาก AWS, และการทดลองกับบอร์กราคาประหยัด
จริงอย่าง STM32 Nucleo หรือ Raspberry Pi Pico

ใน **Part 118** เราจะเปลี่ยนโหมดไปอีกด้าน — จากโลกที่ทรัพยากรจำกัดสุดขีด สู่โลกของ
**Game Development** ที่ต้องการประสิทธิภาพสูงสุดในอีกรูปแบบหนึ่ง (real-time rendering ที่
60+ เฟรมต่อวินาที) โดยใช้ SDL2 และเกริ่นนำ OpenGL

**ต่อไป:** [Part 118 — พื้นฐาน Game Development ด้วย C++ (SDL2/OpenGL)](./part-118-game-dev-basics.md)
