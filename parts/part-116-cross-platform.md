# Part 116: Cross-Platform Development: Windows/Linux/macOS (Step 921–928)

> Module J — Professional และ World-Class Practices | Part 116 จาก 125
> Step ที่ครอบคลุมใน Part นี้: Step 921–928
> Part ก่อนหน้า: [Part 115 — Security ใน C/C++ และ Secure Coding Practice](./part-115-security-cpp.md) | Part ถัดไป: [Part 117 — ภาพรวม Embedded Systems Programming ด้วย C/C++](./part-117-embedded-systems.md)

## เป้าหมายการเรียนรู้ (Learning Objectives)

หลังจบ Part นี้ ผู้เรียนจะสามารถ:

1. อธิบายได้ว่าทำไมโค้ด C/C++ ที่ "compile ผ่านบนเครื่องฉัน" ถึงอาจใช้งานไม่ได้เลยบน OS หรือ compiler
   อีกตัวหนึ่ง และระบุแหล่งที่มาของปัญหาแต่ละประเภทได้ (compiler, ABI, filesystem, endianness)
2. ใช้ preprocessor macro มาตรฐาน (`_WIN32`, `__APPLE__`, `__linux__`, `__GNUC__`, `_MSC_VER`,
   `__clang__`) เพื่อเขียนโค้ดที่ปรับพฤติกรรมตาม platform ได้อย่างถูกต้อง
3. อธิบายปัญหา ABI (Application Binary Interface) ที่ทำให้ library ที่ compile ด้วยคนละ compiler/
   คนละเวอร์ชันมักใช้ร่วมกันไม่ได้ตรงๆ และรู้วิธีป้องกันเบื้องต้น (extern "C", stable C ABI)
4. เข้าใจ Endianness และเขียนโค้ดตรวจสอบ/แปลงค่าให้ทำงานถูกต้องข้าม architecture
5. ใช้ `std::filesystem` (C++17) แทนการเขียน path handling แบบ platform-specific เอง
6. เขียน `CMakeLists.txt` ที่ตรวจจับ platform และปรับ compiler flag ให้เหมาะสมกับแต่ละ OS โดยอัตโนมัติ
7. ออกแบบโค้ดที่ "แยกส่วน platform-specific" ออกจาก business logic ด้วยรูปแบบ wrapper/abstraction
8. เข้าใจแนวทางการทดสอบโค้ด cross-platform ด้วย CI matrix build แม้ไม่มีเครื่องจริงครบทุก OS

> **หมายเหตุสำคัญเรื่องสภาพแวดล้อมของ Part นี้**: เครื่องที่ใช้เขียนและทดสอบเนื้อหานี้เป็น
> container แบบ headless ที่รัน **Linux (Ubuntu 24.04) เท่านั้น** ทุกตัวอย่างโค้ดในบทนี้ถูก
> compile และรันจริงด้วย `g++` 13.3.0 และ `clang++` 18.1 บน Linux เครื่องนี้ — ผลลัพธ์ที่แปะไว้
> คือผลลัพธ์จริงที่ได้จากการรันจริง ไม่ใช่การจำลอง ส่วนพฤติกรรมบน Windows/macOS ที่อธิบายในบทนี้
> อ้างอิงจากมาตรฐานภาษา C++, เอกสารทางการของ Microsoft/Apple/Windows SDK และพฤติกรรมที่ทราบกันดี
> ในวงการ ผู้เรียนที่มีเครื่อง Windows/macOS จริงควรลองรันโค้ดเหล่านี้ซ้ำบนเครื่องตัวเองเพื่อ
> เปรียบเทียบผลลัพธ์

---

## 116.1 ทำไม Cross-Platform Development ถึงยาก (Step 921)

จนถึง Part นี้ เราเขียนโค้ด C/C++ บน Linux เป็นหลักมาตลอดหลักสูตร แต่ในโลกความเป็นจริง
ซอฟต์แวร์จำนวนมาก — ตั้งแต่เกม ไปจนถึงโปรแกรม CAD, IDE, หรือ Library ที่คนทั้งโลกใช้ —
ต้องรันได้ทั้งบน **Windows, Linux และ macOS** พร้อมกัน คำถามคือ ทำไมการเขียนโค้ดให้ "รันได้
ทุกที่" ถึงไม่ใช่เรื่องง่าย ทั้งที่มาตรฐานภาษา C++ เป็นมาตรฐานเดียวกันทั่วโลก?

คำตอบคือ **มาตรฐานภาษา C++ นิยามแค่พฤติกรรมของภาษา ไม่ได้นิยามว่า compiler/OS/hardware
ต้อง implement รายละเอียดเบื้องหลังอย่างไร** ความแตกต่างที่พบบ่อยที่สุดมี 4 กลุ่มใหญ่:

| หมวด | ตัวอย่างปัญหา |
|---|---|
| **Compiler ต่างกัน** | MSVC (Windows), GCC (Linux/MinGW), Clang (macOS/LLVM) implement extension, warning, และ optimization ต่างกัน |
| **Filesystem** | Windows ใช้ `\` เป็น path separator, Linux/macOS ใช้ `/`; Windows ไม่ case-sensitive กับชื่อไฟล์ (ปกติ) แต่ Linux เป็น |
| **ABI (Application Binary Interface)** | การจัด layout ของ struct, calling convention, name mangling ต่างกันระหว่าง compiler/platform |
| **Endianness / Data representation** | CPU บาง architecture เก็บ byte แบบ little-endian บางตัว big-endian; ขนาด `long` ต่างกันระหว่าง Windows (32-bit) กับ Linux/macOS (64-bit) แม้เป็นเครื่อง 64-bit เหมือนกัน |

### ตารางเปรียบเทียบ Toolchain หลักของแต่ละ OS

| หัวข้อ | Windows | Linux | macOS |
|---|---|---|---|
| Compiler หลัก | MSVC (`cl.exe`) หรือ MinGW-w64 (GCC) | GCC / Clang | Clang (Apple Clang) |
| C++ ABI | Microsoft C++ ABI (ไม่เสถียรข้ามเวอร์ชัน MSVC ก่อน 2015, เสถียรตั้งแต่ VS2015 เป็นต้นมา) | Itanium C++ ABI (มาตรฐานที่ GCC/Clang บน Linux ใช้ร่วมกัน) | Itanium C++ ABI (เหมือน Linux เพราะ Clang ใช้ ABI เดียวกัน) |
| Dynamic Library | `.dll` | `.so` | `.dylib` |
| Static Library | `.lib` | `.a` | `.a` |
| Executable | `.exe` | ไม่มีนามสกุล (หรือ ELF) | ไม่มีนามสกุล (หรือ Mach-O) |
| Path separator | `\` (แต่ยอมรับ `/` ในหลาย API) | `/` | `/` |
| Newline (text mode) | `\r\n` (CRLF) | `\n` (LF) | `\n` (LF) |
| Case sensitivity ของไฟล์ | ไม่ (โดย default) | ใช่ | ปกติไม่ (APFS default case-insensitive แต่ตั้งเป็น case-sensitive ได้) |
| `sizeof(long)` (64-bit) | 4 bytes (LLP64) | 8 bytes (LP64) | 8 bytes (LP64) |

แถวสุดท้ายเป็นกับดักที่โปรแกรมเมอร์มือใหม่มักไม่รู้: บน **Windows 64-bit**, `long` ยังคงเป็น
**4 bytes** (data model แบบ LLP64) แต่บน **Linux/macOS 64-bit** `long` เป็น **8 bytes**
(data model แบบ LP64) นี่คือเหตุผลที่โค้ดที่ทำ bit-shift หรือ arithmetic กับ `long` แล้วสมมติ
ขนาดตายตัว มักพังทันทีเมื่อย้ายไป compile บน Windows — ทางแก้คือใช้ type ที่ระบุขนาดชัดเจน
จาก `<cstdint>` เช่น `int32_t`, `int64_t`, `uint64_t` แทนการใช้ `long` เมื่อขนาดมีผลต่อ logic

---

## 116.2 ตรวจจับ Platform ด้วย Preprocessor Macro (Step 922)

Compiler แต่ละตัวจะ define macro พิเศษไว้ล่วงหน้า (predefined macro) เพื่อบอกว่ากำลัง compile
บน OS ไหนและด้วย compiler ตัวไหน เราใช้ `#ifdef`/`#if defined(...)` ตรวจสอบ macro เหล่านี้
เพื่อเขียนโค้ดที่ปรับพฤติกรรมตาม platform ได้

### Macro หลักที่ต้องรู้

| Macro | ความหมาย |
|---|---|
| `_WIN32` | กำลัง compile บน Windows (ใช่ทั้ง 32-bit และ 64-bit ชื่อนี้ค้างมาจากอดีต) |
| `_WIN64` | กำลัง compile บน Windows แบบ 64-bit โดยเฉพาะ |
| `__APPLE__` | กำลัง compile บน macOS/iOS (ของ Apple ทั้งหมด ต้องเช็ค `<TargetConditionals.h>` เพิ่มถ้าต้องแยก iOS/macOS) |
| `__linux__` | กำลัง compile บน Linux |
| `__unix__` | กำลัง compile บนระบบตระกูล Unix (Linux, macOS ก็ define ตัวนี้ด้วย) |
| `__GNUC__` | กำลัง compile ด้วย GCC (หรือ compiler ที่เลียนแบบ GCC เช่น Clang บางเวอร์ชันก็ define ด้วย!) |
| `__clang__` | กำลัง compile ด้วย Clang โดยเฉพาะ (ต้องเช็คตัวนี้ **ก่อน** `__GNUC__` เสมอ) |
| `_MSC_VER` | กำลัง compile ด้วย MSVC ค่าตัวเลขบอกเวอร์ชัน (เช่น 1939 = VS2022) |

ลองเขียนโปรแกรมตรวจสอบ platform และ compiler จริง สร้างไฟล์ `platform_detect.cpp`:

```cpp
#include <iostream>
#include <cstdint>

int main() {
#if defined(_WIN32)
    std::cout << "Platform: Windows (_WIN32 defined)\n";
#elif defined(__APPLE__)
    std::cout << "Platform: macOS/iOS (__APPLE__ defined)\n";
#elif defined(__linux__)
    std::cout << "Platform: Linux (__linux__ defined)\n";
#else
    std::cout << "Platform: Unknown\n";
#endif

    // สำคัญ: ต้องเช็ค __clang__ ก่อน __GNUC__ เสมอ
    // เพราะ Clang define __GNUC__ ด้วยเพื่อความเข้ากันได้กับโค้ดที่เช็คแค่ GCC
#if defined(__GNUC__) && !defined(__clang__)
    std::cout << "Compiler: GCC " << __GNUC__ << "." << __GNUC_MINOR__ << "\n";
#elif defined(__clang__)
    std::cout << "Compiler: Clang " << __clang_major__ << "." << __clang_minor__ << "\n";
#elif defined(_MSC_VER)
    std::cout << "Compiler: MSVC " << _MSC_VER << "\n";
#endif

    std::cout << "sizeof(void*) = " << sizeof(void*) << " bytes ("
              << (sizeof(void*) * 8) << "-bit)\n";

    union { std::uint32_t i; unsigned char c[4]; } u;
    u.i = 0x01020304u;
    if (u.c[0] == 4) std::cout << "Endianness: Little-endian\n";
    else if (u.c[0] == 1) std::cout << "Endianness: Big-endian\n";
    return 0;
}
```

คอมไพล์และรันจริงด้วยทั้ง GCC และ Clang บนเครื่องทดสอบ:

```bash
g++ -Wall -Wextra -std=c++17 platform_detect.cpp -o platform_detect
./platform_detect

clang++ -Wall -Wextra -std=c++17 platform_detect.cpp -o platform_detect_clang
./platform_detect_clang
```

ผลลัพธ์จริงที่ได้ (Linux, GCC 13.3.0):

```
Platform: Linux (__linux__ defined)
Compiler: GCC 13.3
sizeof(void*) = 8 bytes (64-bit)
Endianness: Little-endian
```

ผลลัพธ์จริงที่ได้ (Linux, Clang 18.1.3):

```
Platform: Linux (__linux__ defined)
Compiler: Clang 18.1
sizeof(void*) = 8 bytes (64-bit)
Endianness: Little-endian
```

สังเกตว่าโค้ดเดียวกันทุกตัวอักษร แค่เปลี่ยน compiler ก็ยังให้ผลตรงกันบน Linux — นี่คือสิ่งที่
เราคาดหวังเวลาเขียนโค้ดพกพาได้ ถ้าเอาโค้ดนี้ไป compile บน Windows ด้วย MSVC ผลลัพธ์ที่
**คาดว่าจะได้** (อ้างอิงจากเอกสาร Microsoft และพฤติกรรมมาตรฐานของ x86/x86-64 ทุกวันนี้) คือ:

```
Platform: Windows (_WIN32 defined)
Compiler: MSVC 1939
sizeof(void*) = 8 bytes (64-bit)
Endianness: Little-endian
```

(CPU สถาปัตยกรรม x86/x86-64 และ ARM ที่ macOS/Windows/Linux ใช้งานทั่วไปในปี 2026 ล้วนเป็น
little-endian ทั้งหมด ปัญหา endianness จะเจอจริงๆ ตอนทำงานกับ hardware แบบ embedded บางตัว
หรือข้อมูลที่ส่งผ่านเครือข่ายซึ่งกำหนดเป็น big-endian ตามธรรมเนียม — จะพูดถึงในหัวข้อ 116.4)

---

## 116.3 ปัญหา ABI (Application Binary Interface) (Step 923)

**ABI** คือ "สัญญา" ระดับ binary ว่าฟังก์ชัน, struct, class จะถูกจัดวางในหน่วยความจำและเรียก
ใช้งานอย่างไรในระดับเครื่อง (register ไหนใช้ส่ง parameter, stack เรียงแบบไหน, struct มี
padding เท่าไหร่) ต่างจาก **API** ที่เป็นสัญญาระดับ source code (`signature` ของฟังก์ชัน)

ปัญหาคือ **C++ ไม่มี ABI มาตรฐานสากลที่ทุก compiler ใช้ร่วมกัน** ต่างจาก C ที่ ABI ค่อนข้าง
เสถียรและเหมือนกันในแต่ละ platform วิธีที่ compiler แต่ละตัวจัดการเรื่องต่อไปนี้ **ไม่รับประกัน**
ว่าจะตรงกัน:

- **Name Mangling**: C++ รองรับ function overloading จึงต้องเข้ารหัสชื่อฟังก์ชัน (mangle)
  ให้มี type ของ parameter ฝังอยู่ด้วย แต่ GCC/Clang กับ MSVC ใช้วิธี mangle ชื่อ **ต่างกัน**
- **Virtual Table (vtable) Layout**: การจัดเรียง vtable ของ class ที่มี virtual function
  อาจต่างกันระหว่าง compiler
- **Struct Padding/Alignment**: แม้กฎการ align จะคล้ายกัน แต่ compiler flag บางตัว
  (`#pragma pack`, `-fpack-struct`) ทำให้ผลต่างกันได้
- **Exception Handling Mechanism**: วิธี unwind stack ตอน throw exception ต่างกันระหว่าง
  MSVC (SEH-based) กับ GCC/Clang (DWARF-based บน Linux, Itanium ABI)

ลองดูตัวอย่าง Name Mangling จริงบนเครื่องนี้:

```bash
cat > mangle_demo.cpp << 'EOF'
int add(int a, int b) { return a + b; }
double add(double a, double b) { return a + b; }
EOF
g++ -c mangle_demo.cpp -o mangle_demo.o
nm mangle_demo.o
```

ผลลัพธ์จริงที่ได้:

```
0000000000000000 T _Z3addii
0000000000000014 T _Z3adddd
```

`_Z3addii` คือชื่อ `add(int, int)` ที่ถูก mangle ตามกฎของ Itanium C++ ABI (`_Z` = เริ่มต้น
mangled name, `3add` = ชื่อยาว 3 ตัวอักษรคือ "add", `ii` = parameter สอง `int`) ถ้าไปดูโค้ด
เดียวกันที่ compile ด้วย MSVC บน Windows ชื่อที่ mangle ออกมาจะหน้าตาไม่เหมือนกันเลย เช่น
`?add@@YAHHH@Z` (รูปแบบของ MSVC) — **นี่คือเหตุผลที่ .dll ที่ compile ด้วย MSVC กับ .so ที่
compile ด้วย GCC เอามาแปะกันตรงๆ ในระดับ C++ symbol ไม่ได้** ถ้าจะทำ Library ที่ต้องใช้ร่วมกัน
ข้าม compiler (โดยเฉพาะ plugin system หรือ dynamic library ที่แจกเป็น binary) วิธีแก้มาตรฐาน
คือ **ห่อ interface ด้วย `extern "C"`** เพื่อบังคับให้ใช้กฎการตั้งชื่อแบบ C (ไม่มีการ mangle
ตาม type) ซึ่งเสถียรและเข้ากันได้ข้าม compiler เกือบทุกตัว:

```cpp
// stable_api.h — header สำหรับ public API ของ library ที่จะแจกเป็น .dll/.so ข้าม compiler
#ifdef __cplusplus
extern "C" {
#endif

int stable_add(int a, int b);
void* stable_create_handle(void);
void stable_destroy_handle(void* handle);

#ifdef __cplusplus
}
#endif
```

```cpp
// stable_api.cpp — implementation ภายในใช้ C++ เต็มรูปแบบได้ตามปกติ
#include "stable_api.h"

class InternalEngine {
public:
    int compute(int a, int b) { return a + b; }
};

extern "C" int stable_add(int a, int b) {
    InternalEngine engine;
    return engine.compute(a, b);
}

extern "C" void* stable_create_handle(void) {
    return new InternalEngine();
}

extern "C" void stable_destroy_handle(void* handle) {
    delete static_cast<InternalEngine*>(handle);
}
```

รูปแบบนี้เรียกว่า **"C ABI as a boundary"** — ใช้ C++ เต็มรูปแบบภายใน library แต่ expose
หน้าตา (interface) ที่ผู้ใช้มองเห็นเป็น C ล้วนๆ เป็นเทคนิคที่ library ข้าม-platform ชื่อดัง
จำนวนมาก (เช่น SQLite, libcurl, Vulkan SDK) ใช้กันจริง เพราะ C ABI เสถียรกว่า C++ ABI มาก
และทำให้ภาษาอื่น (Python, Rust, C#) เรียกใช้ library ของเราผ่าน FFI ได้ง่ายด้วย

มาทดสอบ compile และ run จริง:

```bash
g++ -Wall -Wextra -std=c++17 -c stable_api.cpp -o stable_api.o
nm stable_api.o | grep stable_
```

ผลลัพธ์จริง — สังเกตว่าชื่อฟังก์ชันที่ `extern "C"` **ไม่ถูก mangle** เลย (ไม่มี `_Z` นำหน้า):

```
0000000000000000 T stable_add
0000000000000030 T stable_create_handle
0000000000000050 T stable_destroy_handle
```

### Calling Convention: อีกมุมหนึ่งของปัญหา ABI

นอกจาก Name Mangling แล้ว **Calling Convention** (ข้อตกลงว่าพารามิเตอร์ของฟังก์ชันจะถูกส่ง
ผ่าน register หรือ stack แบบไหน ใครเป็นผู้ล้าง stack หลังเรียกฟังก์ชันเสร็จ) ก็เป็นส่วนหนึ่ง
ของ ABI ที่ต่างกันได้ระหว่าง compiler/platform โดยเฉพาะบน **Windows 32-bit** ที่ยังมีการ
ใช้ calling convention หลายแบบผสมกันในโค้ดเก่า:

| Calling Convention | ใครล้าง stack | ใช้ที่ไหนบ่อย |
|---|---|---|
| `__cdecl` (ค่า default ของ C/C++ บน Windows) | ผู้เรียก (caller) | โค้ด C/C++ ทั่วไป |
| `__stdcall` | ฟังก์ชันที่ถูกเรียก (callee) | Windows API (`WINAPI` macro คือ `__stdcall`) |
| `__fastcall` | callee (ส่งพารามิเตอร์แรกๆ ผ่าน register) | โค้ดที่เน้นความเร็วเป็นพิเศษบางกรณี |

```cpp
// ตัวอย่างการระบุ calling convention อย่างชัดเจนบน Windows (MSVC/MinGW)
// โค้ดนี้ compile ได้เฉพาะเมื่อ target เป็น Windows เพราะ __stdcall เป็น
// extension เฉพาะของ compiler สำหรับ Windows เท่านั้น ไม่ใช่มาตรฐาน C++
#if defined(_WIN32)
int __stdcall windows_style_function(int a, int b) {
    return a + b;
}
#endif
```

บน **Linux/macOS (x86-64)** ปัญหานี้แทบไม่เกิดขึ้นแล้ว เพราะ System V AMD64 ABI (มาตรฐาน
ABI ที่ GCC/Clang บน Linux/macOS ใช้ร่วมกัน) กำหนด calling convention ไว้เป็นหนึ่งเดียว
ไม่มีทางเลือกหลายแบบผสมกันแบบ Windows 32-bit ยุคเก่า — นี่คืออีกเหตุผลที่ทำให้ ABI ของ
Linux/macOS "เรียบง่ายและทำนายได้ง่ายกว่า" Windows ในหลายกรณี

---

## 116.4 Endianness: ลำดับไบต์ในหน่วยความจำ (Step 924)

**Endianness** คือวิธีที่ CPU เก็บ byte ของค่าตัวเลขที่มีขนาดมากกว่า 1 byte ในหน่วยความจำ
มี 2 แบบหลัก:

- **Little-endian**: เก็บ byte ที่มีนัยสำคัญน้อยที่สุด (Least Significant Byte) ไว้ที่ address
  ต่ำสุดก่อน — เป็นแบบที่ CPU ตระกูล x86/x86-64 และ ARM ส่วนใหญ่ในปี 2026 ใช้ (โหมด default)
- **Big-endian**: เก็บ byte ที่มีนัยสำคัญมากที่สุด (Most Significant Byte) ไว้ที่ address
  ต่ำสุดก่อน — ใช้ในโปรโตคอลเครือข่ายเป็นธรรมเนียม (เรียกว่า **"Network Byte Order"**) และ
  CPU บางตระกูล (เช่น PowerPC แบบดั้งเดิม, MIPS แบบ BE)

```
ค่า 0x01020304 เก็บใน memory address 0x1000-0x1003:

Little-endian:  [0x1000]=04  [0x1001]=03  [0x1002]=02  [0x1003]=01
Big-endian:     [0x1000]=01  [0x1001]=02  [0x1002]=03  [0x1003]=04
```

โค้ดตรวจสอบ endianness ที่เราทดสอบไปแล้วในหัวข้อ 116.2 ใช้เทคนิค `union` (มอง memory
เดียวกันเป็นทั้ง `uint32_t` และ array ของ `unsigned char`) — เทคนิคนี้เป็นวิธีมาตรฐานที่ปลอดภัย
ใน C/C++ (ต่างจากการ cast pointer ข้าม type ที่อาจผิดกฎ strict aliasing)

### เมื่อไหร่ที่ Endianness มีผลจริงกับงานของเรา

1. **ส่งข้อมูล binary ผ่านเครือข่าย**: โปรโตคอลมาตรฐาน (TCP/IP header, หลาย binary protocol)
   กำหนดให้ตัวเลขหลาย byte ต้องส่งแบบ **Big-endian (Network Byte Order)** เสมอ ไม่ว่าเครื่อง
   ต้นทาง/ปลายทางจะเป็น endianness แบบไหน ภาษา C มีฟังก์ชันมาตรฐานจาก `<arpa/inet.h>`
   (Linux/macOS) หรือ `<winsock2.h>` (Windows) ช่วยแปลงให้อัตโนมัติ:

   ```cpp
   #include <arpa/inet.h>  // Linux/macOS; บน Windows ใช้ <winsock2.h> แทน
   #include <cstdint>

   uint32_t host_value = 0x01020304;
   uint32_t network_value = htonl(host_value); // Host TO Network, Long (32-bit)
   uint32_t back_to_host = ntohl(network_value); // Network TO Host, Long
   ```

   ฟังก์ชันกลุ่มนี้ (`htonl`, `htons`, `ntohl`, `ntohs`) จะ **ไม่ทำอะไรเลย** ถ้าเครื่องเป็น
   big-endian อยู่แล้ว และจะ **สลับ byte จริง** ถ้าเครื่องเป็น little-endian — เขียนโค้ด
   เครือข่ายควรเรียกฟังก์ชันนี้เสมอโดยไม่ต้องเช็ค endianness เองก่อน (จะพูดถึงลึกกว่านี้
   ตอนทำ Socket Programming ใน Part 33–34 ที่เรียนไปแล้ว)

2. **อ่าน/เขียนไฟล์ binary format ที่กำหนด byte order ตายตัว**: เช่นไฟล์ภาพ, ไฟล์เสียง,
   หรือ protocol เฉพาะทางบางแบบ ต้องรู้ endianness ที่ format นั้นกำหนดไว้และแปลงให้ตรง

3. **ทำงานกับ embedded system บาง architecture**: จะพูดถึงเพิ่มเติมใน Part 117

> ข่าวดีคือ ในปี 2026 คอมพิวเตอร์ desktop/server/มือถือแทบทั้งหมด (x86-64, ARM64 ที่ใช้ใน
> Apple Silicon, Windows on ARM, Android, iOS) เป็น **little-endian** ทั้งหมด ปัญหา
> endianness ระหว่าง Windows/Linux/macOS บนเครื่องทั่วไปจึง **แทบไม่เกิดขึ้นแล้ว** สิ่งที่
> ยังสำคัญคือความเข้าใจแนวคิดนี้ไว้สำหรับตอนทำงานกับเครือข่ายหรือ embedded (Part 117)

---

## 116.5 std::filesystem: ทางออกสมัยใหม่แทน Path Handling แบบ Manual (Step 925)

ก่อน C++17 การจัดการ path (สร้างโฟลเดอร์, ต่อ path, แยกชื่อไฟล์กับนามสกุล) ต้องเขียนโค้ด
แยกตาม platform เอง เช่น เช็ค `_WIN32` แล้วใช้ `_mkdir()` จาก `<direct.h>` บน Windows
กับ `mkdir()` จาก `<sys/stat.h>` บน Linux/macOS พร้อมกับต้อง hardcode ตัวคั่น path (`\` หรือ
`/`) เอง — เต็มไปด้วยจุดที่พลาดได้ง่าย

**`std::filesystem`** (เพิ่มเข้ามาใน C++17, header `<filesystem>`) แก้ปัญหานี้ทั้งหมดโดย
ให้ library เป็นตัวจัดการ path separator, การเข้าถึงไฟล์ระบบ ฯลฯ ให้เอง โค้ดเดียวกัน
compile แล้วรันได้ถูกต้องทั้ง 3 OS

### ตารางเปรียบเทียบ: แบบเก่า (Platform-specific) กับ std::filesystem

| งาน | แบบเก่า (ต้องเขียนแยก platform) | std::filesystem (เขียนครั้งเดียว) |
|---|---|---|
| สร้างโฟลเดอร์ | `mkdir()` (POSIX) vs `_mkdir()` (Windows) | `std::filesystem::create_directory(path)` |
| ต่อ path | ต่อ string เอง + ระวัง `/` vs `\` | `path / "subfolder" / "file.txt"` (operator `/` ฉลาดพอ) |
| แยกนามสกุลไฟล์ | parse string เอง (`strrchr` หา `.`) | `path.extension()` |
| listไฟล์ในโฟลเดอร์ | `opendir()`/`readdir()` (POSIX) vs `FindFirstFile()` (Windows) | `std::filesystem::directory_iterator` |
| เช็คว่าไฟล์มีอยู่จริงไหม | `stat()` (POSIX) vs `GetFileAttributes()` (Windows) | `std::filesystem::exists(path)` |
| หาขนาดไฟล์ | `stat()`/`GetFileSizeEx()` | `std::filesystem::file_size(path)` |

### ตัวอย่างจริง: ทดสอบ compile และรันบนเครื่องนี้

```cpp
#include <iostream>
#include <filesystem>
#include <fstream>

namespace fs = std::filesystem;

int main() {
    fs::path config_dir = fs::path("myapp_data") / "config";
    fs::create_directories(config_dir);
    std::cout << "สร้างโฟลเดอร์: " << config_dir << "\n";

    fs::path config_file = config_dir / "settings.txt";
    {
        std::ofstream out(config_file);
        out << "volume=80\n";
    }
    std::cout << "ไฟล์ที่สร้าง: " << config_file << "\n";
    std::cout << "  - เป็น absolute path: " << fs::absolute(config_file) << "\n";
    std::cout << "  - extension: " << config_file.extension() << "\n";
    std::cout << "  - stem: " << config_file.stem() << "\n";
    std::cout << "  - parent_path: " << config_file.parent_path() << "\n";
    std::cout << "  - ขนาดไฟล์: " << fs::file_size(config_file) << " bytes\n";

    std::cout << "\nรายการไฟล์ใน myapp_data (แบบ recursive):\n";
    for (const auto& entry : fs::recursive_directory_iterator("myapp_data")) {
        std::cout << "  " << entry.path().generic_string()
                  << (entry.is_directory() ? " [dir]" : " [file]") << "\n";
    }

    fs::remove_all("myapp_data");
    std::cout << "\nลบโฟลเดอร์ทดสอบเรียบร้อย (cleanup)\n";
    return 0;
}
```

คอมไพล์ (บาง compiler เก่ามากอาจต้องลิงก์ `-lstdc++fs` แต่ GCC 13/Clang 18 สมัยใหม่ไม่ต้องแล้ว):

```bash
g++ -Wall -Wextra -std=c++17 filesystem_demo.cpp -o filesystem_demo
./filesystem_demo
```

ผลลัพธ์จริงที่ได้:

```
สร้างโฟลเดอร์: "myapp_data/config"
ไฟล์ที่สร้าง: "myapp_data/config/settings.txt"
  - เป็น absolute path: "/home/user/.../myapp_data/config/settings.txt"
  - extension: ".txt"
  - stem: "settings"
  - parent_path: "myapp_data/config"
  - ขนาดไฟล์: 10 bytes

รายการไฟล์ใน myapp_data (แบบ recursive):
  myapp_data/config [dir]
  myapp_data/config/settings.txt [file]

ลบโฟลเดอร์ทดสอบเรียบร้อย (cleanup)
```

จุดที่น่าสนใจคือ `config_file.generic_string()` และการพิมพ์ `path` ผ่าน `operator<<` จะแสดง
path ด้วย `/` เสมอ (แม้บน Windows การพิมพ์ปกติของ `path` อาจแสดงเป็น `\` แต่ `generic_string()`
บังคับให้เป็น `/` ทุกกรณี) — ถ้าต้องการ path ที่ "เป็นธรรมชาติ" ของแต่ละ OS (เช่นจะเอาไปแสดง
ให้ user เห็น) ให้ใช้ `path.string()` ธรรมดา ซึ่งจะปรับ separator ให้เข้ากับ OS ปัจจุบัน
โดยอัตโนมัติ — นี่คือหัวใจของ `std::filesystem`: **เราไม่ต้องรู้เลยว่ากำลังรันบน OS ไหน**
library จัดการ separator, การเข้าถึงไฟล์ระบบ และ edge case ให้ทั้งหมด

---

## 116.6 CMake สำหรับจัดการ Cross-Platform Build (Step 926)

ใน Part 91 เราเรียน CMake ไปแบบละเอียดแล้วในฐานะ build system ทั่วไป ใน Part นี้จะเจาะมุมมอง
เฉพาะ **cross-platform**: CMake ถูกออกแบบมาตั้งแต่ต้นเพื่อแก้ปัญหานี้โดยตรง — เขียน
`CMakeLists.txt` ไฟล์เดียว แล้ว CMake จะ generate build system ที่เหมาะกับแต่ละ OS ให้เอง
(Makefile บน Linux, Xcode project บน macOส, Visual Studio solution บน Windows)

### ตัวอย่าง CMakeLists.txt ที่ตรวจจับ platform และปรับ flag อัตโนมัติ

```cmake
cmake_minimum_required(VERSION 3.15)
project(CrossPlatDemo LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

add_executable(crossplat_demo platform_detect.cpp)

# ตรวจจับ platform ผ่านตัวแปรที่ CMake set ให้อัตโนมัติ (ไม่ต้องเขียน #ifdef เอง)
if (WIN32)
    message(STATUS "กำลัง config บน Windows")
    target_compile_definitions(crossplat_demo PRIVATE PLATFORM_WINDOWS)
elseif (APPLE)
    message(STATUS "กำลัง config บน macOS")
    target_compile_definitions(crossplat_demo PRIVATE PLATFORM_MACOS)
elseif (UNIX)
    message(STATUS "กำลัง config บน Linux/UNIX")
    target_compile_definitions(crossplat_demo PRIVATE PLATFORM_LINUX)
endif()

# generator expression: เลือก flag ตาม compiler ID โดยไม่ต้องเขียน if/else ซ้อนเยอะ
target_compile_options(crossplat_demo PRIVATE
    $<$<CXX_COMPILER_ID:GNU,Clang>:-Wall -Wextra>
    $<$<CXX_COMPILER_ID:MSVC>:/W4>
)
```

สิ่งที่เกิดขึ้นในไฟล์นี้:

- `if (WIN32)` / `elseif (APPLE)` / `elseif (UNIX)` — CMake set ตัวแปรพวกนี้ให้อัตโนมัติตาม
  OS ที่กำลัง configure โดยไม่ต้องเขียน preprocessor macro เอง ระดับ build-system
- `target_compile_definitions(... PLATFORM_WINDOWS)` — ส่ง macro ของเราเองเข้าไปใน compiler
  (เทียบเท่ากับ `-DPLATFORM_WINDOWS`) เพื่อให้โค้ด C++ เช็คด้วย `#ifdef PLATFORM_WINDOWS` ได้
  ถ้าอยากใช้ macro ที่เราตั้งชื่อเองแทนการเช็ค `_WIN32` ตรงๆ ในทุกไฟล์
- generator expression `$<$<CXX_COMPILER_ID:GNU,Clang>:...>` — เลือก flag ที่เหมาะสมกับ
  MSVC (`/W4` = ระดับ warning สูงสุด) เทียบเท่ากับ GCC/Clang (`-Wall -Wextra`) เพราะ MSVC
  ใช้รูปแบบ command-line flag คนละแบบกับ GCC/Clang โดยสิ้นเชิง (`/` แทน `-`)

ทดสอบ configure และ build จริงบนเครื่องนี้:

```bash
cmake -S . -B build
cmake --build build
./build/crossplat_demo
```

ผลลัพธ์จริงที่ได้:

```
-- The CXX compiler identification is GNU 13.3.0
-- กำลัง config บน Linux/UNIX
-- Configuring done (0.2s)
-- Generating done (0.0s)
-- Build files have been written to: .../build
[ 50%] Building CXX object CMakeFiles/crossplat_demo.dir/platform_detect.cpp.o
[100%] Linking CXX executable crossplat_demo
[100%] Built target crossplat_demo
Platform: Linux (__linux__ defined)
Compiler: GCC 13.3
sizeof(void*) = 8 bytes (64-bit)
Endianness: Little-endian
```

สังเกตบรรทัด `-- กำลัง config บน Linux/UNIX` ที่ CMake พิมพ์ระหว่าง configure — พิสูจน์ว่า
`if (UNIX)` ทำงานถูกต้องจริง บนเครื่อง Windows คำสั่งเดียวกันนี้ (`cmake -S . -B build` แล้ว
`cmake --build build`) จะสร้าง Visual Studio solution (หรือ Ninja build ถ้าเลือก generator
นั้น) โดยอัตโนมัติ และ branch `if (WIN32)` จะทำงานแทน — **ผู้ใช้ไม่ต้องแก้ CMakeLists.txt เลย**
นี่คือคุณค่าหลักของการใช้ CMake สำหรับโปรเจกต์ cross-platform ตั้งแต่ต้น

---

## 116.7 ออกแบบโค้ด: แยก Platform-Specific ออกจาก Business Logic (Step 927)

รูปแบบที่ดีที่สุดสำหรับโค้ด cross-platform ไม่ใช่การโรย `#ifdef` กระจายไปทั่วทั้งโปรเจกต์
(เพราะจะอ่านยากและ maintain ยากมากเมื่อโปรเจกต์โต) แต่คือการ **ห่อ (wrap) โค้ด
platform-specific ไว้ในจุดเดียว** แล้วให้ส่วนอื่นของโปรแกรมเรียกผ่าน interface กลางที่ไม่ขึ้น
กับ platform เลย — คล้ายกับหลักการ **Dependency Inversion** ที่เรียนใน Part 114
(Software Architecture)

### ตัวอย่าง: ฟังก์ชันหา Home Directory ของผู้ใช้แบบ Cross-Platform

```cpp
// platform_utils.h — public interface ไม่มี #ifdef เลยสักบรรทัด
#pragma once
#include <string>

namespace platform {
    // คืนค่า path ของ home directory ผู้ใช้ปัจจุบัน ไม่ว่าจะรันบน OS ไหน
    std::string get_home_directory();
}
```

```cpp
// platform_utils.cpp — จุดเดียวในโปรเจกต์ทั้งหมดที่มี #ifdef เรื่อง platform
#include "platform_utils.h"
#include <cstdlib>

#if defined(_WIN32)
    #include <windows.h>
    #include <shlobj.h>
#endif

namespace platform {

std::string get_home_directory() {
#if defined(_WIN32)
    char path[MAX_PATH];
    if (SUCCEEDED(SHGetFolderPathA(nullptr, CSIDL_PROFILE, nullptr, 0, path))) {
        return std::string(path);
    }
    return "";
#else
    // Linux และ macOS ต่างก็ใช้ environment variable HOME เหมือนกัน (มาตรฐาน POSIX)
    const char* home = std::getenv("HOME");
    return home ? std::string(home) : std::string("");
#endif
}

} // namespace platform
```

```cpp
// main.cpp — ส่วนที่เหลือของโปรแกรมทั้งหมดไม่ต้องรู้จัก #ifdef เลย
#include "platform_utils.h"
#include <iostream>

int main() {
    std::string home = platform::get_home_directory();
    std::cout << "Home directory: " << home << "\n";
    return 0;
}
```

ทดสอบ compile และรันจริงบนเครื่องนี้ (Linux เดินเข้าเส้นทาง `#else` ที่ใช้ `HOME`):

```bash
g++ -Wall -Wextra -std=c++17 -c platform_utils.cpp -o platform_utils.o
g++ -Wall -Wextra -std=c++17 main.cpp platform_utils.o -o platform_demo
./platform_demo
```

ผลลัพธ์จริงที่ได้:

```
Home directory: /home/user
```

ข้อดีของแนวทางนี้:

1. **โค้ด business logic (`main.cpp` และไฟล์อื่นทั้งหมด) ไม่มี `#ifdef` เลยแม้แต่บรรทัดเดียว**
   อ่านง่าย ทดสอบง่าย ย้ายไปใช้ platform ใหม่ในอนาคตแค่แก้ไฟล์เดียว
2. **จุดที่ผูกกับ platform ถูกจำกัดไว้แคบมาก** (ไฟล์เดียว) ทำให้ตอน debug ปัญหาเฉพาะ OS
   รู้ทันทีว่าต้องไปดูที่ไหน
3. ถ้าต้อง mock/test บน CI ที่รันแค่ Linux แต่ต้องทดสอบ logic ที่เกี่ยวกับ "home directory"
   ก็ทำ unit test แยกจาก `platform_utils.cpp` ได้ (inject ค่า string ปลอมแทนการเรียกจริง)

---

## 116.8 การทดสอบโค้ด Cross-Platform โดยไม่มีเครื่องครบทุก OS (Step 928)

ปัญหาที่นักพัฒนาแทบทุกคนเจอ: มักไม่มีเครื่อง Windows, Linux, macOS ครบทั้ง 3 เครื่องบน
โต๊ะทำงาน (เหมือนกับ container ที่ใช้เขียนบทเรียนนี้ ที่มีแค่ Linux เท่านั้น) แนวทางที่วงการ
ใช้จริงเพื่อมั่นใจว่าโค้ดทำงานถูกต้องทุก platform มีดังนี้:

### 1. CI Matrix Build (สำคัญที่สุด — ทบทวนจาก Part 94)

GitHub Actions (และ CI ระบบอื่น) รองรับการรัน job เดียวกันบน runner หลาย OS พร้อมกันผ่าน
`matrix` โดยไม่ต้องมีเครื่องจริงเลย:

```yaml
name: Cross-Platform Build
on: [push, pull_request]

jobs:
  build:
    strategy:
      matrix:
        os: [ubuntu-latest, windows-latest, macos-latest]
    runs-on: ${{ matrix.os }}
    steps:
      - uses: actions/checkout@v4
      - name: Configure
        run: cmake -S . -B build
      - name: Build
        run: cmake --build build --config Release
      - name: Test
        run: ctest --test-dir build --output-on-failure
```

Workflow นี้จะ compile และรัน test บนเครื่อง Windows, Linux, macOS จริงที่ GitHub เตรียมไว้ให้
(ฟรีสำหรับ public repository) ทุกครั้งที่ push โค้ด — คือวิธีที่ทีมพัฒนาซอฟต์แวร์ cross-platform
ระดับโลก (เช่น ทีมของ LLVM, CMake เอง, หรือ SDL2) ใช้ยืนยันความถูกต้องจริงตลอดเวลา ไม่ต้องรอ
ให้มีคนซื้อ Mac มาทดสอบเอง

### 2. Static Analysis หลายตัว + หลาย Compiler

การ compile ด้วย GCC, Clang **และ** MSVC (ผ่าน CI) ช่วยจับ error ที่ compiler ตัวหนึ่งอนุญาต
แต่อีกตัวไม่ยอม (บ่อยครั้งเป็นสัญญาณของ Undefined Behavior หรือการพึ่งพา extension เฉพาะ
compiler ที่ไม่ portable) — Clang และ MSVC มักเข้มงวดกับมาตรฐานมากกว่า GCC ในบาง edge case

### 3. Docker และ Virtual Machine

สำหรับทดสอบ Linux distro ต่างๆ (Ubuntu, Alpine, Fedora ฯลฯ) โดยไม่ต้องมีเครื่องจริงหลายเครื่อง
Docker คือเครื่องมือมาตรฐาน (ทบทวนจาก Part 112) ส่วนการทดสอบ macOS จริง มักจำเป็นต้องใช้
เครื่อง Mac จริงหรือ CI ของ GitHub เพราะ macOS ไม่อนุญาตให้ virtualize บน hardware ที่ไม่ใช่
ของ Apple ตามเงื่อนไข license (ยกเว้นบางบริการ cloud ที่ได้รับอนุญาตพิเศษ)

### 4. ความจริงของบทเรียนนี้

ทุกตัวอย่างโค้ดใน Part นี้ **compile และรันจริงบน Linux container ที่ไม่มีจอ** ที่ใช้เขียน
บทเรียนนี้ — เราไม่มี Windows หรือ macOS ให้ทดสอบจริงในสภาพแวดล้อมนี้ ดังนั้นสิ่งที่ทำได้อย่าง
ซื่อสัตย์ที่สุดคือ: (1) ยืนยันว่าโค้ดคอมไพล์และรันถูกต้องบน Linux ด้วย compiler สองตัว (GCC,
Clang) (2) อธิบายพฤติกรรมบน Windows/macOS โดยอ้างอิงจากมาตรฐานภาษาและเอกสารทางการ ไม่ใช่
การเดา และ (3) แนะนำให้ผู้เรียนที่มีเครื่อง Windows/macOS จริงทดลองรันโค้ดเดียวกันด้วยตัวเอง
เพื่อเห็นผลลัพธ์จริงเทียบกัน — นี่คือทักษะสำคัญของวิศวกร cross-platform ตัวจริง: **เขียนโค้ด
ที่ตนเองไม่สามารถ compile ทดสอบได้ครบทุก target แต่มั่นใจในความถูกต้องได้ผ่านมาตรฐานภาษา,
เครื่องมือ CI, และหลักการออกแบบที่ดี**

---

## ข้อผิดพลาดที่พบบ่อย (Common Pitfalls)

1. **เช็ค `__GNUC__` ก่อน `__clang__`** — Clang define `__GNUC__` ไว้ด้วยเพื่อความเข้ากันได้
   กับโค้ดเก่า ถ้าเขียน `#ifdef __GNUC__` ไว้ก่อนแล้วค่อยเช็ค `__clang__` ทีหลัง โค้ดจะเข้าใจ
   ผิดว่า Clang คือ GCC เสมอ ต้องเช็ค `__clang__` **ก่อน** ทุกครั้ง
2. **Hardcode path separator เอง** — เขียน `folder + "\\" + filename` หรือ
   `folder + "/" + filename` ตรงๆ แทนที่จะใช้ `std::filesystem::path` ทำให้โค้ดพังทันทีเมื่อ
   ย้าย OS
3. **สมมติขนาด type ตายตัว** — ใช้ `long`, `int` แล้วคาดหวังขนาดคงที่ข้าม platform (`long`
   เป็น 4 หรือ 8 bytes ขึ้นกับ OS/data model) ควรใช้ `<cstdint>` (`int32_t`, `int64_t`,
   `uint64_t`) เมื่อขนาดมีผลต่อความถูกต้องของโปรแกรม (เช่น serialize ข้อมูล, ทำ binary format)
4. **ลืมว่า newline ต่างกัน** — ไฟล์ text ที่สร้างบน Windows มักมี `\r\n` แต่บน Linux/macOS
   มีแค่ `\n` ถ้าอ่านไฟล์ด้วย mode ข้อความ (`"r"`) ข้าม platform โดยไม่ระวัง อาจเจอ `\r`
   ปนอยู่ในสตริงที่ parse ออกมา (โดยเฉพาะเวลาอ่านไฟล์ config/CSV ที่ผู้ใช้ Windows แก้ไขมา)
5. **ทึกทักว่า `.dll`/`.so` ที่ compile ด้วยคนละ compiler ใช้แทนกันได้เลย** — โดยเฉพาะถ้า
   ส่ง C++ class/STL container ข้ามขอบเขต library ตรงๆ (ABI ไม่ตรงกันแน่นอน) ควรใช้ C ABI
   เป็นขอบเขตเสมอถ้าต้องแจก binary ให้คนอื่นที่ compiler อาจไม่ตรงกับเรา
6. **ไม่เคยรันบน CI matrix เลยจนกว่าจะถึงวัน release** — ทำให้เจอบั๊กเฉพาะ OS ช้าเกินไป ควร
   ตั้ง CI ให้ build ทุก OS เป้าหมายตั้งแต่ commit แรกๆ ของโปรเจกต์

---

## แบบฝึกหัดท้ายบท

1. เขียนฟังก์ชัน `std::string get_path_separator_style()` ที่คืนค่า `"backslash"` ถ้าอยู่บน
   Windows หรือ `"forward-slash"` ถ้าอยู่บน Linux/macOS โดยใช้ preprocessor macro (ไม่ใช้
   `std::filesystem`) แล้ว compile รันบนเครื่องของคุณเพื่อยืนยันผลลัพธ์

2. ใช้ `std::filesystem` เขียนโปรแกรมที่รับชื่อโฟลเดอร์จาก command line argument แล้วพิมพ์
   รายชื่อไฟล์ทั้งหมดในโฟลเดอร์นั้นพร้อมขนาดไฟล์แต่ละไฟล์ (ไม่ต้อง recursive) เรียงจากไฟล์
   ใหญ่ไปเล็ก

3. เขียน `CMakeLists.txt` สำหรับโปรเจกต์ที่มี 2 ไฟล์ source โดยให้ compile flag เป็น
   `-O2 -Wall -Wextra` เมื่อใช้ GCC/Clang และ `/O2 /W4` เมื่อใช้ MSVC (ใช้ generator
   expression ตามที่เรียนในหัวข้อ 116.6)

4. ออกแบบและเขียน library เล็กๆ ที่มี public interface เป็น `extern "C"` (มีฟังก์ชันอย่างน้อย
   2 ฟังก์ชัน) ภายในใช้ C++ class ได้ตามสะดวก แล้วตรวจสอบด้วย `nm` ว่าชื่อฟังก์ชันไม่ถูก
   mangle จริง

5. เขียนฟังก์ชันแปลงค่า `uint16_t` จาก host byte order เป็น network byte order เอง (ไม่ใช้
   `htons`) โดยใช้ bit-shift และ mask แล้วเปรียบเทียบผลลัพธ์กับ `htons()` จริงว่าตรงกัน

6. ลองเขียน GitHub Actions workflow (`.yml`) ที่ build โปรเจกต์ C++ ด้วย CMake บน 3 OS
   พร้อมกัน (อ้างอิงจากหัวข้อ 116.8) แม้ไม่มี repository จริงให้ push ก็ให้เขียนไฟล์ให้ถูก
   syntax และอธิบายว่าแต่ละส่วนทำหน้าที่อะไร

### แนวทางเฉลยข้อ 1

```cpp
#include <iostream>
#include <string>

std::string get_path_separator_style() {
#if defined(_WIN32)
    return "backslash";
#else
    return "forward-slash";
#endif
}

int main() {
    std::cout << "Path separator style: " << get_path_separator_style() << "\n";
    return 0;
}
```

คอมไพล์และรันบนเครื่อง Linux ที่ใช้ทดสอบเนื้อหานี้:

```bash
g++ -Wall -Wextra -std=c++17 ex1.cpp -o ex1
./ex1
```

ผลลัพธ์จริง:

```
Path separator style: forward-slash
```

### แนวทางเฉลยข้อ 4

```cpp
// mylib.h
#pragma once
#ifdef __cplusplus
extern "C" {
#endif

int mylib_square(int x);
int mylib_add(int a, int b);

#ifdef __cplusplus
}
#endif
```

```cpp
// mylib.cpp
#include "mylib.h"

class MathEngine {
public:
    int square(int x) { return x * x; }
    int add(int a, int b) { return a + b; }
};

extern "C" int mylib_square(int x) {
    MathEngine engine;
    return engine.square(x);
}

extern "C" int mylib_add(int a, int b) {
    MathEngine engine;
    return engine.add(a, b);
}
```

ทดสอบและตรวจสอบด้วย `nm`:

```bash
g++ -Wall -Wextra -std=c++17 -c mylib.cpp -o mylib.o
nm mylib.o | grep mylib_
```

ผลลัพธ์จริงที่ได้ (ยืนยันว่าไม่มี mangling — ชื่อฟังก์ชันตรงกับที่เขียนในโค้ดเป๊ะ):

```
0000000000000000 T mylib_add
0000000000000020 T mylib_square
```

เทียบกับถ้าไม่ใส่ `extern "C"` (ลอง comment ออกแล้ว compile ใหม่) ชื่อใน `nm` จะกลายเป็น
`_Z10mylib_addii` และ `_Z12mylib_squarei` ทันที ซึ่งพิสูจน์ให้เห็นชัดเจนว่า `extern "C"`
คือกลไกเดียวที่ควบคุมว่า compiler จะ mangle ชื่อหรือไม่

---

## สรุปท้ายบท

ใน Part นี้เราได้:

- เข้าใจว่าความยากของ cross-platform C++ มาจาก 4 แหล่งหลัก: compiler ต่างกัน, filesystem
  ต่างกัน, ABI ต่างกัน, และ endianness/data model ต่างกัน
- ใช้ preprocessor macro (`_WIN32`, `__APPLE__`, `__linux__`, `__clang__`, `__GNUC__`,
  `_MSC_VER`) ตรวจจับ platform/compiler ได้อย่างถูกต้อง พร้อมข้อควรระวังเรื่องลำดับการเช็ค
- เข้าใจปัญหา ABI โดยเฉพาะ Name Mangling และรู้จักเทคนิค "C ABI as a boundary" ด้วย
  `extern "C"` เพื่อทำ library ที่ใช้ข้าม compiler ได้จริง
- เข้าใจ Endianness และรู้ว่าเมื่อไหร่ที่มันสำคัญจริง (เครือข่าย, binary format, embedded)
- ใช้ `std::filesystem` แทนการเขียน path handling แบบ platform-specific เอง ซึ่งลดโค้ด
  `#ifdef` ที่กระจัดกระจายลงไปมาก
- เขียน CMake ที่ตรวจจับ platform และปรับ flag ให้อัตโนมัติ พร้อมเห็นผลจริงจากการ build
- ออกแบบโค้ดแบบแยก platform-specific ออกจาก business logic ด้วย wrapper/interface pattern
- รู้จักวิธีทดสอบโค้ด cross-platform ในทางปฏิบัติแม้ไม่มีเครื่องครบทุก OS ผ่าน CI matrix build

ใน **Part 117** เราจะก้าวเข้าสู่โลกที่ต่างออกไปอีกขั้ว — **Embedded Systems** ที่ไม่มี OS
เต็มรูปแบบให้พึ่งพาเลย ต้องเขียนโปรแกรมที่คุมฮาร์ดแวร์โดยตรง และเราจะได้ทดลอง
cross-compile โปรแกรมจริงด้วย `gcc-arm-none-eabi` แล้วรันบน QEMU เพื่อดูผลลัพธ์จริงบนจอ
"ฮาร์ดแวร์จำลอง"

**ต่อไป:** [Part 117 — ภาพรวม Embedded Systems Programming ด้วย C/C++](./part-117-embedded-systems.md)
