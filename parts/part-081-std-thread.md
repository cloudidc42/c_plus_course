# Part 81: std::thread และ Concurrency ใน Modern C++ (Step 641–648)

> Module G — Concurrency และ Performance Engineering | Part 81 จาก 125
> Step ที่ครอบคลุมใน Part นี้: Step 641–648
> Part ก่อนหน้า: [Part 80 — Modern C++ Best Practice และ C++ Core Guidelines](./part-080-modern-cpp-best-practices.md) | Part ถัดไป: [Part 82 — std::mutex, std::atomic, C++ Memory Model](./part-082-mutex-atomic-memory-model.md)

## เป้าหมายการเรียนรู้ (Learning Objectives)

หลังจบ Part นี้ ผู้เรียนจะสามารถ:

1. อธิบายได้ว่าทำไม C++11 ถึงเพิ่ม `std::thread` เข้ามาในมาตรฐานภาษา ทั้งที่ POSIX Threads
   (pthread) ที่เรียนไปแล้วใน Part 31-32 ก็ใช้งานได้ดีอยู่แล้ว
2. สร้าง จัดการ และรอ thread ด้วย `std::thread` ผ่าน constructor, `.join()`, `.detach()`
   และ `.joinable()` ได้อย่างถูกต้องครบทุกกรณี
3. ส่ง argument ให้ thread ได้ทั้งแบบ by value (ปลอดภัยโดย default) และ by reference
   (ผ่าน `std::ref`) พร้อมอธิบายได้ว่าทำไม `std::thread` ถึง "copy" argument ทุกตัวโดย
   default
4. ใช้ Lambda ร่วมกับ `std::thread` เขียนโค้ด concurrent ที่กระชับกว่าสไตล์ pthread มาก
5. อธิบายและสาธิตได้ว่าทำไมการปล่อยให้ `std::thread` object ที่ยัง joinable อยู่ถูก
   destruct โดยไม่ join/detach จะทำให้เกิด `std::terminate()` ทันที พร้อมรู้วิธีป้องกัน
   ด้วย RAII (`thread_guard`)
6. จัดการ Exception ที่เกิดขึ้นภายใน thread function ได้อย่างปลอดภัย และเข้าใจว่าทำไม
   exception ที่ "หลุด" ออกจาก thread function โดยไม่ถูกดักจับจะทำให้โปรแกรมทั้งหมดพัง
7. ใช้ `std::this_thread::sleep_for`, `std::this_thread::get_id`, และ
   `std::thread::hardware_concurrency()` ได้อย่างถูกต้อง
8. เปรียบเทียบโค้ด pthread จาก Part 31 กับโค้ด `std::thread` ที่ทำงานแบบเดียวกันแบบ
   side-by-side และอธิบายข้อดี-ข้อเสียของแต่ละแบบได้

---

## 81.1 ทำไม C++11 ถึงเพิ่ม `std::thread` (Step 641)

ใน Part 31 และ 32 เราใช้ **POSIX Threads (pthread)** เขียนโปรแกรม multi-thread บน Linux
มาแล้วอย่างละเอียด ทั้ง `pthread_create`, `pthread_join`, `pthread_mutex_t`,
`pthread_cond_t` ทำงานได้ดีและยังคงเป็นมาตรฐานสำคัญของ Systems Programming บน Linux
จนถึงทุกวันนี้ คำถามที่สมเหตุสมผลคือ: **แล้วทำไม C++11 (ปี 2011) ถึงต้องเพิ่ม
`std::thread` เข้ามาอีก ทั้งที่ pthread ก็ใช้งานได้อยู่แล้ว?**

คำตอบมีเหตุผลหลักอยู่ 3 ข้อ:

### 1. Portability — pthread ผูกติดกับ POSIX เท่านั้น

`pthread` เป็นส่วนหนึ่งของมาตรฐาน **POSIX** ซึ่งเป็นมาตรฐานของระบบปฏิบัติการสไตล์ Unix
(Linux, macOS, BSD) เท่านั้น **Windows ไม่รองรับ pthread โดยตรง** (แม้จะมี library
อย่าง `pthreads-win32` ให้ใช้ แต่ก็ไม่ใช่ของ native) ก่อน C++11 ถ้าต้องการเขียนโปรแกรม
C++ ที่ทำงานแบบ multi-thread ได้บนทั้ง Linux และ Windows โดยไม่พึ่งพา library ภายนอก
เพิ่มเติม แทบเป็นไปไม่ได้เลย ต้องเขียนโค้ดแยกกันคนละชุดสำหรับแต่ละแพลตฟอร์ม
(`pthread_create` บน Linux/macOS, `CreateThread` บน Windows) แล้วซ่อนความต่างไว้หลัง
`#ifdef`

`std::thread` แก้ปัญหานี้โดยตรง: มันเป็นส่วนหนึ่งของ **C++ Standard Library**
ไม่ใช่ของ OS ใดโดยเฉพาะ คอมไพเลอร์ที่รองรับ C++11 ขึ้นไป (GCC, Clang, MSVC) จะมี
implementation ของ `std::thread` ที่ **ห่อ (wrap)** กลไก thread ของแต่ละ OS ไว้ข้างใน
ให้อัตโนมัติ (บน Linux/macOS มักจะ implement ทับ pthread อยู่ดีข้างใน แต่บน Windows จะ
implement ทับ Windows Thread API แทน) โค้ดที่เขียนด้วย `std::thread` ตัวเดียวกัน
**คอมไพล์และรันได้บนทุกแพลตฟอร์มโดยไม่ต้องแก้อะไรเลย**

### 2. Type Safety — ไม่ต้องผ่าน `void*` อีกต่อไป

จำ signature ของ `pthread_create` ได้ไหม:

```c
void *(*start_routine)(void *)
```

thread function ของ pthread **ต้อง**รับและคืนค่าเป็น `void*` เท่านั้น ถ้าต้องการส่ง
argument มากกว่า 1 ค่า ต้องห่อไว้ใน struct แล้ว cast ผ่าน `void*` ไปมา ซึ่งเป็นจุดที่เสี่ยง
เกิดบั๊กจากการ cast ผิดชนิดได้ง่าย (compiler ตรวจสอบให้ไม่ได้เพราะ `void*` ไม่มีข้อมูล
เรื่องชนิด) `std::thread` ใช้ Template ภายใน ทำให้ **รับ argument ชนิดใดก็ได้โดยตรง**
ผ่าน type checking ของ C++ ตามปกติ ไม่ต้อง cast ผ่าน `void*` เลยแม้แต่ครั้งเดียว

### 3. บูรณาการกับฟีเจอร์อื่นของ Modern C++

`std::thread` ทำงานร่วมกับ RAII (Part 68), Lambda (Part 64), Exception (Part 54),
Move Semantics (Part 70) ได้อย่างเป็นธรรมชาติ เพราะมันถูกออกแบบมาเป็นส่วนหนึ่งของ
ภาษาเดียวกันตั้งแต่ต้น ไม่ใช่ C API ที่ถูก "ครอบ" มาทีหลัง สิ่งที่เราจะเห็นตลอด Part นี้คือ
โค้ด `std::thread` มักจะ**สั้นกว่าและปลอดภัยกว่า** โค้ด pthread ที่ทำงานแบบเดียวกัน
อย่างชัดเจน

### ตารางเปรียบเทียบภาพรวม: pthread vs std::thread

| หัวข้อ | pthread (Part 31-32) | `std::thread` (Part นี้) |
|---|---|---|
| มาตรฐาน | POSIX (Linux/macOS/Unix เท่านั้น) | ISO C++11 (ทุกแพลตฟอร์มที่ compiler รองรับ) |
| Header | `<pthread.h>` | `<thread>` |
| Flag ตอน compile | ต้องใส่ `-pthread` | ต้องใส่ `-pthread` เช่นกันบน Linux (เพราะ implementation ข้างในยังพึ่ง pthread library ของระบบ) |
| Signature ของ thread function | ตายตัว `void *(*)(void *)` | รับ callable object ชนิดใดก็ได้ (function, lambda, functor) พร้อม argument กี่ตัวก็ได้ |
| การส่ง argument | ต้องผ่าน `void*` + cast เอง | ส่งตรงๆ ผ่าน template, type-safe เต็มรูปแบบ |
| การรอ thread | `pthread_join(t, &retval)` | `t.join()` |
| การปล่อยทำงานอิสระ | `pthread_detach(t)` | `t.detach()` |
| ลืมจัดการ thread | ทรัพยากรรั่วไหล (คล้าย zombie) แต่โปรแกรมไม่ crash | **`std::terminate()` ทันที — โปรแกรมพังทั้งกระบวน** (ดู 81.5) |
| จำนวน CPU logic core | ไม่มี API มาตรฐานตรงๆ (ต้องใช้ `sysconf`) | `std::thread::hardware_concurrency()` |

จุดที่น่าสนใจที่สุดในตารางนี้คือแถวสุดท้ายๆ — `std::thread` **เข้มงวดกว่า** pthread ใน
เรื่องการจัดการ resource (ถ้าลืม join/detach จะพังทันทีแทนที่จะแค่รั่วไหลเงียบๆ) ซึ่งฟังดู
เหมือนข้อเสีย แต่จริงๆ แล้วเป็นการออกแบบที่ตั้งใจ — บังคับให้โปรแกรมเมอร์ต้องตัดสินใจ
อย่างชัดเจนเสมอว่าจะ join หรือ detach thread ที่สร้างขึ้นมา ไม่ปล่อยให้ "ลืม" ได้ง่ายๆ
เหมือน pthread เราจะเห็นรายละเอียดเรื่องนี้ในหัวข้อ 81.5

---

## 81.2 `std::thread`: Construction, `join()`, `detach()` (Step 642)

### สร้าง thread ด้วย constructor โดยตรง

`std::thread` ใช้ **constructor** ในการสร้างและเริ่มรัน thread ทันที (ไม่มีขั้นตอนแยก
เหมือน `pthread_create` ที่ต้องส่ง handle เข้าไปเป็น output parameter):

```cpp
#include <iostream>
#include <thread>

void print_message(const std::string& msg) {
    std::cout << "[thread " << std::this_thread::get_id() << "] " << msg << "\n";
}

int main() {
    std::cout << "[main] thread หลักเริ่มทำงาน กำลังสร้าง thread ลูก 2 ตัว...\n";

    std::thread t1(print_message, "สวัสดีจาก thread ที่ 1");
    std::thread t2(print_message, "สวัสดีจาก thread ที่ 2");

    std::cout << "[main] สร้าง thread ครบแล้ว รอ (join) ให้ทำงานจบ...\n";

    t1.join();
    t2.join();

    std::cout << "[main] thread ทั้งสองจบการทำงานแล้ว โปรแกรมจบ\n";
    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 -pthread thread_basic.cpp -o thread_basic
./thread_basic
```

ผลลัพธ์ (thread ID ตัวเลขจริงจะต่างกันไปทุกครั้งที่รัน):

```
[main] thread หลักเริ่มทำงาน กำลังสร้าง thread ลูก 2 ตัว...
[main] สร้าง thread ครบแล้ว รอ (join) ให้ทำงานจบ...
[thread 140207911335616] สวัสดีจาก thread ที่ 1
[thread 140207902942912] สวัสดีจาก thread ที่ 2
[main] thread ทั้งสองจบการทำงานแล้ว โปรแกรมจบ
```

สังเกตว่า `std::thread t1(print_message, "สวัสดีจาก thread ที่ 1");` บรรทัดเดียว
ทำหน้าที่เดียวกับ `pthread_create(&t1, NULL, print_message, arg)` ของ pthread ทุกประการ
— constructor ของ `std::thread` **เริ่มรัน thread ทันที** ไม่ต้องมีขั้นตอน "เปิดใช้งาน"
แยกต่างหาก และไม่ต้องเช็ค return value เป็น error code เอง (ถ้าสร้าง thread ไม่สำเร็จ
จริงๆ — เช่น OS ไม่มีทรัพยากรพอ — `std::thread` จะ throw `std::system_error` แทน ตาม
หลักการ Exception ที่เรียนใน Part 54 แทนที่จะคืน error code ให้เช็คเอง)

### `.join()`: รอให้ thread ทำงานจบ

`.join()` เทียบเท่ากับ `pthread_join()` ทุกประการ — **บล็อกรอ**จนกว่า thread นั้นจะทำงาน
จบ แล้วปลดปล่อยทรัพยากรของ thread คืนให้ระบบ ต่างกันแค่ `std::thread::join()` เป็น
**member function** ที่เรียกผ่าน object โดยตรง ไม่ต้องส่ง handle เป็น argument แยก
เหมือน `pthread_join(t, NULL)`

### `.detach()`: ปล่อยให้ทำงานเป็นอิสระ

บางครั้งเราไม่ต้องการรอผลลัพธ์ของ thread เลย (เช่น background logging thread ที่ทำงาน
ไปเรื่อยๆ จนกว่า process จะจบ) `.detach()` คือคำสั่งบอกว่า **"ไม่ต้องรอ thread นี้อีกแล้ว
ปล่อยให้มันทำงานเป็นอิสระไปเลย"**:

```cpp
#include <chrono>
#include <iostream>
#include <thread>

void background_logger() {
    for (int i = 0; i < 3; ++i) {
        std::this_thread::sleep_for(std::chrono::milliseconds(50));
        std::cout << "[background] log tick " << i << "\n";
    }
}

int main() {
    std::cout << "[main] เริ่มโปรแกรม\n";

    std::thread t(background_logger);
    t.detach(); // ปล่อยให้ทำงานเป็นอิสระ ไม่รอ ไม่ผูกอายุกับ t อีกต่อไป

    std::cout << "[main] t.joinable() หลัง detach = " << std::boolalpha << t.joinable() << "\n";

    // รอให้เวลาผ่านไปพอที่ background thread จะทำงานจบ (เพื่อผลลัพธ์ที่อ่านง่ายในตัวอย่างนี้)
    std::this_thread::sleep_for(std::chrono::milliseconds(200));
    std::cout << "[main] จบโปรแกรม\n";
    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 -pthread detach_demo.cpp -o detach_demo
./detach_demo
```

ผลลัพธ์:

```
[main] เริ่มโปรแกรม
[main] t.joinable() หลัง detach = false
[background] log tick 0
[background] log tick 1
[background] log tick 2
[main] จบโปรแกรม
```

### `.joinable()`: เช็คสถานะของ thread object

`.joinable()` คืน `true` ถ้า `std::thread` object นี้**ยังผูกอยู่กับ thread ของระบบจริง
ที่ยังไม่ถูก join หรือ detach** และคืน `false` ในกรณีต่อไปนี้:

- Thread object ถูกสร้างแบบ **default constructor** (`std::thread t;`) โดยยังไม่ได้ผูก
  กับงานใดๆ เลย
- Thread ถูก `.join()` ไปแล้ว
- Thread ถูก `.detach()` ไปแล้ว
- Thread object ถูก **move** ค่าออกไปให้ตัวอื่นแล้ว (`std::thread` copy ไม่ได้ แต่ move
  ได้ — ตามหลักการ Move Semantics ของ Part 70 เพราะ thread ต้องมี "เจ้าของ" เพียงหนึ่ง
  เดียวเสมอ)

กฎทองที่ต้องจำ: **`std::thread` object แต่ละตัวต้องถูก `.join()` หรือ `.detach()`
อย่างใดอย่างหนึ่งเสมอ ก่อนที่ตัวมันเองจะถูก destruct** ไม่งั้นจะเกิดปัญหาร้ายแรงตามที่จะ
เห็นในหัวข้อ 81.5

---

## 81.3 การส่ง Argument ให้ Thread: By Value กับ `std::ref` (Step 643)

ต่างจาก pthread ที่ต้องส่ง argument ผ่าน `void*` เพียงตัวเดียว `std::thread` รับ
argument ได้ **กี่ตัวก็ได้** โดยส่งต่อจาก constructor ไปยังฟังก์ชันตรงๆ:

```cpp
std::thread t(function_name, arg1, arg2, arg3, /* ... */);
```

### พฤติกรรม default: `std::thread` "copy" argument ทุกตัวเสมอ

จุดสำคัญที่สุดที่ต้องเข้าใจคือ **`std::thread` จะสร้างสำเนา (copy) ของทุก argument ที่
ส่งเข้าไป** แล้วเก็บไว้ภายในตัวมันเองก่อน จากนั้นค่อยส่งสำเนานั้นให้ thread function ทำงาน
พฤติกรรมนี้ปลอดภัยโดย default เพราะไม่ว่าตัวแปรต้นทางจะเปลี่ยนแปลงหรือหมดอายุไปแล้ว
thread ก็ยังมี "สำเนาของตัวเอง" ใช้งานต่อได้เสมอ — แก้ปัญหาคลาสสิกเรื่อง loop counter
ที่เจอใน pthread (Part 31 หัวข้อ 31.5) ได้โดยอัตโนมัติ

```cpp
#include <iostream>
#include <string>
#include <thread>

// รับ argument by value: thread เก็บสำเนาของตัวเอง ปลอดภัยเสมอ
void greet_by_value(std::string name) {
    name += " (แก้ไขภายใน thread)";
    std::cout << "[by value] " << name << "\n";
}

// รับ argument by reference: ต้องการแก้ไขตัวแปรต้นทางจริงๆ
void append_suffix(std::string& text) {
    text += "_processed";
}

int main() {
    std::string original = "somchai";
    std::thread t1(greet_by_value, original);
    t1.join();
    std::cout << "[main] original หลัง by-value thread จบ ยังคงเป็น: " << original << "\n";

    std::string shared_text = "report";
    // ต้องห่อด้วย std::ref เพราะ std::thread จะ "copy" argument ทุกตัวโดย default
    // ถ้าไม่ใส่ std::ref จะ compile error เพราะ std::thread constructor พยายามสร้างสำเนา
    // ของ reference parameter จากค่าที่ decay แล้ว (ไม่ compile ผ่านเงียบๆ)
    std::thread t2(append_suffix, std::ref(shared_text));
    t2.join();
    std::cout << "[main] shared_text หลัง by-reference thread จบ: " << shared_text << "\n";

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 -pthread args_value_vs_ref.cpp -o args_value_vs_ref
./args_value_vs_ref
```

ผลลัพธ์:

```
[by value] somchai (แก้ไขภายใน thread)
[main] original หลัง by-value thread จบ ยังคงเป็น: somchai
[main] shared_text หลัง by-reference thread จบ: report_processed
```

สังเกตว่า `original` **ไม่เปลี่ยนแปลง** เพราะ `greet_by_value` แก้ไข **สำเนา** ของมันเอง
ที่ thread เก็บไว้ ในขณะที่ `shared_text` **เปลี่ยนแปลงจริง** เพราะเราห่อด้วย `std::ref`
บอก `std::thread` อย่างชัดเจนว่า "อย่า copy ตัวนี้ ให้ส่ง reference จริงๆ ไปแทน"

### ทำไมลืม `std::ref` แล้วเกิด Compile Error (ไม่ใช่แค่ Warning)

ถ้าลองส่ง `shared_text` ตรงๆ โดยไม่ห่อด้วย `std::ref`:

```cpp
std::thread t2(append_suffix, shared_text); // ลืม std::ref
```

จะได้ **compile error** ทันที (ไม่ใช่แค่ compile ผ่านแล้วพังตอนรัน):

```
error: static assertion failed: std::thread arguments must be invocable after conversion to rvalues
```

เหตุผลเชิงลึก: `std::thread` constructor จะ `decay` (แปลง type ให้เป็นชนิดพื้นฐานที่สุด)
argument ทุกตัวก่อนเก็บเป็นสำเนา เมื่อ `decay` ทำงานกับ `std::string` ธรรมดา (ไม่ใช่
`std::reference_wrapper`) จะได้ `std::string` เต็มรูปแบบ (ค่า ไม่ใช่ reference) แล้วเมื่อ
compiler ลองเรียก `append_suffix(std::string&)` ด้วยค่า rvalue ที่ได้จากการ copy นั้น
จะพบว่า **`std::string&` (non-const reference) รับ rvalue ไม่ได้** ตามกฎ reference
binding ปกติของภาษา (Part 43) จึง static_assert ล้มเหลวตั้งแต่ compile time — นี่คือ
ตัวอย่างที่ดีว่า Type System ของ C++ ช่วยจับบั๊กที่ pthread (ซึ่งผ่าน `void*` จับไม่ได้เลย)
ได้ตั้งแต่ก่อนโปรแกรมจะรันด้วยซ้ำ

`std::ref(shared_text)` สร้างค่าชนิด `std::reference_wrapper<std::string>` ที่ยังคง
"จำได้" ว่าต้องส่งเป็น reference แม้จะผ่านการ `decay`/copy ไปแล้วก็ตาม (เพราะ
`reference_wrapper` เก็บ pointer ภายใน ไม่ใช่ตัวข้อมูลจริง) เมื่อเรียกฟังก์ชันจริง C++
จะแปลง `reference_wrapper` กลับเป็น reference ให้อัตโนมัติ

---

## 81.4 Lambda กับ `std::thread`: สั้นกว่า pthread มาก (Step 644)

การใช้ Lambda (Part 64) ร่วมกับ `std::thread` คือรูปแบบที่นิยมที่สุดในโค้ด C++ สมัยใหม่
เพราะไม่ต้องประกาศฟังก์ชันแยกต่างหากเลยสำหรับงานสั้นๆ:

```cpp
#include <iostream>
#include <thread>
#include <vector>

int main() {
    std::vector<std::thread> workers;
    const int num_workers = 4;

    for (int i = 0; i < num_workers; ++i) {
        // capture by value ([i]) เพื่อให้แต่ละ thread มีสำเนา i เป็นของตัวเอง
        // (เทียบกับปัญหา address ของ loop counter ใน pthread ที่ Part 31 เจอ)
        workers.emplace_back([i]() {
            std::cout << "[worker lambda] เริ่มทำงานด้วย id = " << i << "\n";
        });
    }

    for (auto& w : workers) {
        w.join();
    }

    std::cout << "[main] worker ทั้งหมดจบการทำงานแล้ว\n";
    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 -pthread lambda_demo.cpp -o lambda_demo
./lambda_demo
```

ผลลัพธ์:

```
[worker lambda] เริ่มทำงานด้วย id = 0
[worker lambda] เริ่มทำงานด้วย id = 1
[worker lambda] เริ่มทำงานด้วย id = 2
[worker lambda] เริ่มทำงานด้วย id = 3
[main] worker ทั้งหมดจบการทำงานแล้ว
```

เทียบกับ pthread ใน Part 31 (`ex_factorial_threads.c`) ที่ต้องประกาศฟังก์ชันแยก รับ
`void*` แล้ว cast เอง โค้ดชุดนี้ใช้ **capture list ของ lambda (`[i]`)** ทำหน้าที่แทนการ
สร้าง struct + `void*` cast ได้เลยในบรรทัดเดียว ปลอดภัยกว่าเพราะ `[i]` capture by value
ให้แต่ละ lambda มีสำเนา `i` เป็นของตัวเองโดยอัตโนมัติ **แก้ปัญหา loop counter ของ pthread
(Part 31 หัวข้อ 31.5) ได้ตั้งแต่ระดับภาษาเลย** ไม่ต้องสร้าง array แยกช่องเองเหมือนที่
pthread ต้องทำ

> **ข้อควรระวัง**: ถ้า capture ด้วย `[&i]` (by reference) แทน `[i]` จะกลับไปเจอปัญหา
> เดียวกับ pthread ทุกประการ เพราะทุก lambda จะอ้างอิงตัวแปร `i` ตัวเดียวกันในลูป
> ทำให้ค่าที่อ่านได้อาจไม่ตรงกับตอนที่ thread ถูกสร้าง — **capture by value เสมอเมื่อส่ง
> ค่าที่เปลี่ยนแปลงในลูปให้ thread**

### `std::vector<std::thread>`: เก็บ thread หลายตัวอย่างเป็นระบบ

สังเกตว่าโค้ดข้างต้นใช้ `std::vector<std::thread>` เก็บ thread ทั้งหมด แล้ววนลูป `.join()`
ทีหลัง — รูปแบบนี้เป็นมาตรฐานที่ใช้บ่อยมากเมื่อจำนวน thread ไม่คงที่ (ขึ้นกับ
`hardware_concurrency()` หรือขนาดของงาน) เพราะ `std::thread` **move ได้แต่ copy ไม่ได้**
(ตามหลักการ Rule of Five ใน Part 80) `.emplace_back()` จึงสร้าง thread ใหม่เข้าไปใน
vector โดยตรงโดยไม่ต้อง copy เลย

---

## 81.5 `std::terminate()`: อันตรายที่สุดของ `std::thread` (Step 645)

นี่คือความแตกต่างที่สำคัญที่สุดระหว่าง `std::thread` กับ pthread ที่ต้องเข้าใจให้ลึกซึ้ง
ก่อนใช้งานจริง

### กฎเหล็ก: Thread ที่ Joinable ต้องถูก join/detach ก่อน Destruct เสมอ

```cpp
#include <iostream>
#include <thread>

void worker() {
    std::cout << "[worker] เริ่มทำงาน\n";
}

int main() {
    std::cout << "[main] สร้าง thread แต่ไม่ join และไม่ detach\n";
    std::thread t(worker);
    // ตั้งใจไม่เรียก t.join() หรือ t.detach() เพื่อสาธิต std::terminate
    std::cout << "[main] ออกจาก main โดยไม่จัดการ t เลย\n";
    return 0;
} // t ถูก destruct ตรงนี้ ขณะยัง joinable() == true -> std::terminate ถูกเรียก
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 -pthread terminate_demo.cpp -o terminate_demo
./terminate_demo; echo "EXIT CODE: $?"
```

ผลลัพธ์จริงที่ได้ — โปรแกรม**พังทันที**:

```
terminate called without an active exception
Aborted (core dumped)
EXIT CODE: 134
```

Exit code `134` คือ `128 + 6` (สัญญาณ `SIGABRT` เลข 6) — `std::terminate()` เรียก
`std::abort()` ภายใน ทำให้ process ถูกฆ่าทิ้งทันทีโดยไม่มีโอกาส cleanup ใดๆ เลย (ไม่มี
destructor ของตัวแปรอื่นถูกเรียก ไม่มีโอกาส catch exception ใดๆ ทั้งสิ้น)

### ทำไม C++ ถึงออกแบบให้ "พังแรง" ขนาดนี้

คำถามที่ตามมาคือทำไม Committee ของ C++ ไม่ออกแบบให้ destructor ของ `std::thread`
เรียก `.join()` ให้อัตโนมัติไปเลยล่ะ ดูจะปลอดภัยกว่า? เหตุผลคือ:

- **การ join อัตโนมัติอาจทำให้โปรแกรมค้างเงียบๆ**: ถ้า destructor เรียก join ให้
  อัตโนมัติ โปรแกรมเมอร์อาจไม่รู้ตัวเลยว่าโค้ดของตนกำลังบล็อกรอ thread ที่ทำงานช้าอยู่
  ตรงไหน (เพราะ join เกิดขึ้น "เงียบๆ" ตอนออกจาก scope) ทำให้ debug ยากกว่าเดิมมาก
- **การ detach อัตโนมัติอาจทำให้ resource เข้าถึงหลังหมดอายุ**: ถ้า destructor เรียก
  detach ให้อัตโนมัติแทน thread อาจยังทำงานต่อไปโดยอ้างอิงตัวแปรที่ scope นั้นถือไว้
  (ผ่าน reference หรือ pointer) ซึ่งกำลังจะถูกทำลายพอดี กลายเป็น dangling
  reference/pointer ทันที (ดูตัวอย่างจริงในแบบฝึกหัดท้ายบท)

Committee จึงเลือกทางที่ **"บังคับให้โปรแกรมเมอร์ต้องตัดสินใจเองอย่างชัดเจน"** แทนที่จะ
เดาให้ — ถ้าลืมตัดสินใจ โปรแกรมจะพังทันทีแบบชัดเจนที่สุดเท่าที่จะทำได้ (ไม่ใช่บั๊กที่แอบ
ซ่อนอยู่เงียบๆ เหมือนทรัพยากร pthread รั่วไหล) นี่คือปรัชญาเดียวกับที่ Part 80 อธิบายไว้:
**Type/Resource Safety ต้องชัดเจนและตรวจสอบได้ ไม่ใช่การเดาที่อาจผิดพลาด**

### วิธีป้องกันที่ถูกต้อง: RAII `thread_guard`

ทางแก้ที่เป็นมาตรฐานที่สุด (มาจากหนังสือ *C++ Concurrency in Action* ของ Anthony
Williams ซึ่งเป็นหนังสืออ้างอิงสำคัญของวงการ) คือเขียน RAII wrapper ที่ผูกอายุของ
`std::thread` เข้ากับอายุของ object ตามหลักการที่เรียนใน Part 68:

```cpp
#include <iostream>
#include <stdexcept>
#include <thread>
#include <utility>

// RAII wrapper: ผูกอายุของ std::thread เข้ากับอายุของ object นี้
// ถ้าออกจาก scope ไม่ว่าจะทางปกติหรือผ่าน exception จะ join() ให้อัตโนมัติเสมอ
class thread_guard {
public:
    explicit thread_guard(std::thread t) : t_(std::move(t)) {}

    ~thread_guard() {
        if (t_.joinable()) {
            t_.join();
        }
    }

    thread_guard(const thread_guard&) = delete;
    thread_guard& operator=(const thread_guard&) = delete;

private:
    std::thread t_;
};

void worker() {
    std::cout << "[worker] กำลังทำงาน...\n";
}

void do_work_that_might_throw() {
    thread_guard guard{std::thread(worker)};
    std::cout << "[main] กำลังทำงานอย่างอื่นระหว่างรอ worker...\n";
    throw std::runtime_error("เกิดข้อผิดพลาดระหว่างทำงาน");
    // ไม่มีทางมาถึงบรรทัดนี้ แต่ guard's destructor จะยัง join() ให้เสมอ
    // ระหว่างที่ exception กำลัง unwind stack ออกไป
}

int main() {
    try {
        do_work_that_might_throw();
    } catch (const std::exception& e) {
        std::cout << "[main] ดักจับ exception ได้: " << e.what() << "\n";
        std::cout << "[main] thread ถูก join ไปแล้วโดย thread_guard destructor "
                     "แม้จะเกิด exception ก็ตาม (ไม่มี std::terminate)\n";
    }
    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 -pthread thread_guard.cpp -o thread_guard
./thread_guard
```

ผลลัพธ์:

```
[main] กำลังทำงานอย่างอื่นระหว่างรอ worker...
[worker] กำลังทำงาน...
[main] ดักจับ exception ได้: เกิดข้อผิดพลาดระหว่างทำงาน
[main] thread ถูก join ไปแล้วโดย thread_guard destructor แม้จะเกิด exception ก็ตาม (ไม่มี std::terminate)
```

นี่คือการนำ RAII (Part 68) มาประยุกต์ใช้กับทรัพยากรที่ซับซ้อนที่สุดตัวหนึ่งคือ **thread**
— ไม่ว่า `do_work_that_might_throw()` จะจบแบบปกติหรือจบเพราะ exception, destructor ของ
`guard` จะถูกเรียกเสมอระหว่างการ unwind stack และ `.join()` จะถูกเรียกให้อัตโนมัติทุก
ครั้ง ไม่มีทาง `std::terminate()` เกิดขึ้นได้เลย

> **สังเกต syntax**: บรรทัด `thread_guard guard{std::thread(worker)};` ใช้วงเล็บปีกกา
> `{}` แทนวงเล็บกลม `()` เพราะถ้าเขียน `thread_guard guard(std::thread(worker));`
> compiler จะตีความ (ผิด) ว่าเป็นการ **ประกาศฟังก์ชัน** ชื่อ `guard` ที่รับ parameter
> เป็น `std::thread(*)()` แทนที่จะเป็นการสร้างตัวแปร — ปัญหานี้เรียกว่า **"Most Vexing
> Parse"** ที่ Part 80 (ข้อ 9 ของ "Prefer X over Y") แนะนำให้ใช้ Uniform Initialization
> `{}` แก้ไขได้พอดี (compiler เตือนด้วย `-Wvexing-parse` ถ้าใช้ `()` แบบผิด)

### Exception ที่หลุดออกจาก Thread Function โดยตรง

อีกกรณีหนึ่งที่ทำให้เกิด `std::terminate()` คือการปล่อยให้ **exception หลุดออกจาก thread
function เอง** โดยไม่ถูกดักจับเลย:

```cpp
#include <iostream>
#include <stdexcept>
#include <thread>

void risky_worker() {
    throw std::runtime_error("เกิดปัญหาภายใน worker thread");
}

int main() {
    std::cout << "[main] สร้าง thread ที่ throw exception โดยไม่ดักจับ\n";
    std::thread t(risky_worker);
    t.join();
    std::cout << "[main] ไม่มีทางมาถึงบรรทัดนี้ได้\n";
    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 -pthread exception_terminate.cpp -o exception_terminate
./exception_terminate; echo "EXIT: $?"
```

ผลลัพธ์:

```
terminate called after throwing an instance of 'std::runtime_error'
  what():  เกิดปัญหาภายใน worker thread
Aborted (core dumped)
EXIT: 134
```

เหตุผลคือ **exception mechanism ของ C++ ทำงานเฉพาะภายใน call stack ของ thread เดียวกัน
เท่านั้น** — `try`/`catch` ใน `main()` **ไม่สามารถดักจับ exception ที่ throw จาก thread
อื่นได้เลย** ไม่ว่าจะห่อ `t.join()` ด้วย `try`/`catch` แน่นหนาแค่ไหนก็ตาม เพราะเมื่อ
exception ไต่ขึ้นไปจนสุด call stack ของ `risky_worker()` (ซึ่งรันอยู่บน stack ของ
thread ใหม่ คนละ stack กับ `main()`) โดยไม่เจอ handler เลย มาตรฐาน C++ กำหนดให้เรียก
`std::terminate()` ทันที นี่คือเหตุผลที่ Part 83 (`std::future`/`std::async`) จะมี
ประโยชน์มาก เพราะมันมีกลไก "ส่ง exception ข้าม thread กลับมาให้ผู้เรียกจัดการทีหลัง" ให้
อัตโนมัติ

**วิธีแก้ที่ถูกต้องสำหรับ Part นี้**: ต้องดักจับ exception **ภายใน** thread function
เองให้ครบทุกกรณีก่อนที่มันจะจบการทำงาน:

```cpp
#include <exception>
#include <iostream>
#include <stdexcept>
#include <thread>

void risky_worker() {
    try {
        throw std::runtime_error("เกิดปัญหาภายใน worker thread");
    } catch (const std::exception& e) {
        // ต้องดักจับให้ครบภายใน thread function เอง ห้ามปล่อยให้ exception
        // "หลุด" ออกจากฟังก์ชันนี้ไปเด็ดขาด
        std::cerr << "[worker] ดักจับ exception ได้: " << e.what() << "\n";
    }
}

int main() {
    std::thread t(risky_worker);
    t.join();
    std::cout << "[main] โปรแกรมทำงานต่อได้ปกติ เพราะ exception ถูกดักจับใน thread แล้ว\n";
    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 -pthread exception_safe.cpp -o exception_safe
./exception_safe
```

ผลลัพธ์:

```
[worker] ดักจับ exception ได้: เกิดปัญหาภายใน worker thread
[main] โปรแกรมทำงานต่อได้ปกติ เพราะ exception ถูกดักจับใน thread แล้ว
```

> **กฎทองของ Part นี้**: ทุก thread function ต้อง **ดักจับ exception ทุกชนิดที่อาจ
> เกิดขึ้นภายในตัวมันเองให้ครบ** ห้ามปล่อยให้หลุดออกไปนอกฟังก์ชันเด็ดขาด ไม่มีทางให้
> `main()` หรือ thread อื่นมาช่วยดักจับแทนได้เลย

---

## 81.6 `this_thread::sleep_for`, `this_thread::get_id`, `hardware_concurrency()` (Step 646)

Namespace `std::this_thread` มีฟังก์ชัน utility ที่ใช้ได้จาก**ภายใน thread ปัจจุบัน**
ที่กำลังรันโค้ดอยู่:

```cpp
#include <chrono>
#include <iostream>
#include <thread>

void show_info(int worker_id) {
    std::cout << "[worker " << worker_id << "] thread id ของฉันคือ "
              << std::this_thread::get_id() << "\n";
    std::this_thread::sleep_for(std::chrono::milliseconds(30));
    std::cout << "[worker " << worker_id << "] ตื่นแล้ว ทำงานต่อ\n";
}

int main() {
    unsigned int cores = std::thread::hardware_concurrency();
    std::cout << "[main] จำนวน hardware thread ที่เครื่องนี้รองรับ (โดยประมาณ): "
              << cores << "\n";
    std::cout << "[main] main thread id = " << std::this_thread::get_id() << "\n";

    std::thread t(show_info, 1);
    t.join();

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 -pthread sleep_id_hw.cpp -o sleep_id_hw
./sleep_id_hw
```

ผลลัพธ์ (ตัวเลขจริงขึ้นกับเครื่องที่รัน):

```
[main] จำนวน hardware thread ที่เครื่องนี้รองรับ (โดยประมาณ): 4
[main] main thread id = 140432691029824
[worker 1] thread id ของฉันคือ 140432684086976
[worker 1] ตื่นแล้ว ทำงานต่อ
```

### รายละเอียดของแต่ละฟังก์ชัน

| ฟังก์ชัน | ความหมาย | เทียบเท่า pthread |
|---|---|---|
| `std::this_thread::get_id()` | คืนค่า thread id ของ thread **ปัจจุบัน** ที่กำลังรันโค้ดอยู่ ชนิด `std::thread::id` (เปรียบเทียบเท่ากันได้ด้วย `==`) | `pthread_self()` |
| `std::this_thread::sleep_for(duration)` | หยุดรอ thread ปัจจุบันเป็นระยะเวลาหนึ่ง รับค่าจาก `<chrono>` (`std::chrono::milliseconds`, `seconds` ฯลฯ) | `nanosleep()` (Part 32) |
| `std::this_thread::sleep_until(time_point)` | หยุดรอจนถึงเวลาที่กำหนดไว้แบบ absolute (ไม่ใช่ duration แบบสัมพัทธ์) | ไม่มีเทียบเท่าตรงๆ ใน pthread พื้นฐาน |
| `std::thread::hardware_concurrency()` | คืนค่าประมาณจำนวน thread ที่ CPU รันพร้อมกันได้จริง (logical core) — เป็น **static member function** เรียกผ่านชื่อ class ได้เลยไม่ต้องมี object | ต้องใช้ `sysconf(_SC_NPROCESSORS_ONLN)` ซึ่งเป็นของ Linux เฉพาะ |

### ทำไม `hardware_concurrency()` ถึงสำคัญมากใน Module G

ค่านี้จะถูกใช้ซ้ำๆ ตลอด Module G ที่เหลือ (โดยเฉพาะ Part 85 เรื่อง Parallel Algorithm
และ Part 90 เรื่อง Benchmarking) เพราะเป็นตัวเลขที่บอกว่า **ควรแบ่งงานออกเป็นกี่ส่วน**
เพื่อใช้ CPU ให้คุ้มค่าที่สุด — การสร้าง thread มากกว่าจำนวน core จริงไม่ได้ทำให้งานเร็ว
ขึ้น (มีแต่จะเพิ่ม overhead จากการสลับ context ไปมา) มาตรฐานกำหนดให้ค่านี้เป็นแค่
"คำแนะนำ" (hint) เท่านั้น — **อาจคืนค่า `0` ได้ถ้าระบบไม่สามารถระบุค่าที่แน่นอนได้**
ดังนั้นโค้ด Production ที่ดีควรเช็คเสมอ:

```cpp
unsigned int num_threads = std::thread::hardware_concurrency();
if (num_threads == 0) {
    num_threads = 4; // ค่า fallback ที่สมเหตุสมผลถ้าระบบตอบไม่ได้
}
```

---

## 81.7 เทียบโค้ดจริง: pthread (Part 31) กับ `std::thread` แบบ Side-by-Side (Step 647)

มาดูตัวอย่างที่ครบวงจรที่สุด — โปรแกรมที่หลาย thread ช่วยกันคำนวณผลรวมของ array คนละ
ส่วน (`thread_sum_safe.c` จาก Part 31 หัวข้อ 31.7) เขียนใหม่ด้วย `std::thread` แล้ว
เปรียบเทียบกันทีละจุด

### เวอร์ชัน pthread (ทบทวนจาก Part 31)

```c
/* thread_sum_safe.c จาก Part 31 (ย่อเฉพาะส่วนสำคัญเพื่อเทียบ) */
typedef struct { int start; int end; } range_t;

void *sum_range(void *arg) {
    range_t *r = (range_t *)arg;
    long *partial_sum = malloc(sizeof(long));   /* ต้อง malloc เอง */
    *partial_sum = 0;
    for (int i = r->start; i < r->end; i++) {
        *partial_sum += g_array[i];
    }
    pthread_exit(partial_sum);                  /* คืนค่าผ่าน void* */
}

/* ใน main: */
pthread_create(&threads[i], NULL, sum_range, &ranges[i]);
/* ... */
void *retval;
pthread_join(threads[i], &retval);
long *partial_sum = (long *)retval;
total += *partial_sum;
free(partial_sum);                              /* ต้อง free เอง ไม่งั้น leak */
```

### เวอร์ชัน `std::thread`

```cpp
#include <iostream>
#include <thread>
#include <vector>

constexpr int ARRAY_SIZE = 4'000'000;
constexpr int NUM_THREADS = 4;

int g_array[ARRAY_SIZE];

void sum_range(int start, int end, long& result) {
    long local_sum = 0;
    for (int i = start; i < end; ++i) {
        local_sum += g_array[i];
    }
    result = local_sum; // เขียนลงตัวแปรของตัวเองที่ main เตรียมไว้ให้คนละช่อง
}

int main() {
    for (int i = 0; i < ARRAY_SIZE; ++i) {
        g_array[i] = 1;
    }

    std::vector<std::thread> threads;
    std::vector<long> partial_sums(NUM_THREADS, 0);
    const int chunk = ARRAY_SIZE / NUM_THREADS;

    for (int i = 0; i < NUM_THREADS; ++i) {
        const int start = i * chunk;
        const int end = (i == NUM_THREADS - 1) ? ARRAY_SIZE : (i + 1) * chunk;
        // ส่ง partial_sums[i] แบบ reference ผ่าน std::ref เพื่อให้ thread เขียนผลลัพธ์
        // กลับมาที่ vector ของ main โดยตรง (คนละ index กัน ไม่มี race condition)
        threads.emplace_back(sum_range, start, end, std::ref(partial_sums[i]));
    }

    for (auto& t : threads) {
        t.join();
    }

    long total = 0;
    for (long partial : partial_sums) {
        total += partial;
    }

    std::cout << "[main] ผลรวมที่คาดหวัง: " << ARRAY_SIZE << "\n";
    std::cout << "[main] ผลรวมที่ได้จริง: " << total << "\n";

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 -pthread sum_std_thread.cpp -o sum_std_thread
./sum_std_thread
```

ผลลัพธ์:

```
[main] ผลรวมที่คาดหวัง: 4000000
[main] ผลรวมที่ได้จริง: 4000000
```

### ตารางสรุปความแตกต่างที่เห็นได้จริงจากโค้ดคู่นี้

| ประเด็น | pthread | `std::thread` |
|---|---|---|
| การส่งผลลัพธ์กลับ | ต้อง `malloc()` heap memory เอง ส่งผ่าน `void*`, แล้ว `free()` เองทีหลัง | ส่ง `std::ref(partial_sums[i])` ตรงๆ ให้เขียนกลับเข้า `vector` ของ main ได้เลย ไม่มี heap allocation ที่ต้องจัดการเอง |
| การ cast ชนิดข้อมูล | ต้อง cast `void*` เป็น `range_t*`, `long*` ด้วยมือหลายจุด | ไม่มีการ cast แม้แต่จุดเดียว — type-safe ทั้งหมดผ่าน template |
| ความเสี่ยง memory leak | มี (ถ้าลืม `free(partial_sum)`) | ไม่มีเลย (ไม่มี `new`/`malloc` ในโค้ดนี้เลย) |
| ความเสี่ยงลืม join | รั่วไหลทรัพยากรเงียบๆ ไม่ crash | โปรแกรม crash ทันที (`std::terminate`) — เห็นบั๊กได้เร็วกว่า |
| จำนวนบรรทัดของ core logic | ยาวกว่า (ต้องเขียน struct, cast, malloc/free) | สั้นกว่าอย่างชัดเจน |
| Portability | Linux/macOS เท่านั้น | ทุกแพลตฟอร์มที่มี C++11 compiler |

นี่คือบทสรุปที่ชัดเจนของ Part นี้: `std::thread` ไม่ได้ "แทนที่" ความรู้เรื่อง thread
ที่เรียนจาก pthread ใน Part 31-32 เลย (แนวคิดเรื่อง Race Condition, Critical Section,
Deadlock ที่เรียนไปยังคงใช้ได้ทั้งหมดกับ `std::thread`) แต่มันคือ **เครื่องมือที่ดีกว่า
ในการเขียนแนวคิดเดียวกันนั้น** ให้ปลอดภัยกว่า สั้นกว่า และพกพาข้ามแพลตฟอร์มได้

---

## ข้อผิดพลาดที่พบบ่อย (Common Pitfalls)

1. **ปล่อยให้ `std::thread` ที่ยัง joinable อยู่ถูก destruct โดยไม่ join/detach** —
   ทำให้เกิด `std::terminate()` ทันที (ดูสาธิตเต็มรูปแบบใน 81.5) เป็นข้อผิดพลาดที่พบบ่อย
   ที่สุดของผู้เริ่มต้นใช้ `std::thread` วิธีป้องกันที่ดีที่สุดคือใช้ RAII wrapper อย่าง
   `thread_guard` เสมอ แทนที่จะเรียก `.join()`/`.detach()` ตรงๆ ในหลายจุดของโค้ด

2. **ลืม `std::ref` ตอนต้องการส่ง argument แบบ reference** — จะได้ compile error ที่
   อ่านยาก (`static assertion failed`) เพราะ `std::thread` copy argument ทุกตัวโดย
   default เสมอ ต้องห่อด้วย `std::ref()` อย่างชัดเจนทุกครั้งที่ต้องการแก้ไขตัวแปรต้นทาง
   จริงๆ

3. **Detach thread ที่อ้างอิง (by reference/pointer) ตัวแปร local ของฟังก์ชันที่จะ
   return ไปแล้ว** — เป็น **dangling reference** ที่อันตรายกว่าเดิม เพราะ `std::thread`
   ที่ detach แล้วอาจยังทำงานอยู่นานหลังจากฟังก์ชันต้นทาง return ไปแล้ว ทำให้เข้าถึง
   หน่วยความจำที่ถูกทำลายไปแล้ว (stack ของฟังก์ชันที่ return ไปแล้วถูกนำกลับมาใช้ซ้ำ)
   ตัวอย่างคลาสสิกและวิธีแก้ไขอยู่ในแบบฝึกหัดข้อ 4

4. **ปล่อยให้ exception หลุดออกจาก thread function โดยไม่ดักจับ** — ทำให้เกิด
   `std::terminate()` ทันที เพราะ `try`/`catch` ของ thread อื่น (รวมถึง `main()`)
   ไม่มีทางดักจับ exception ข้าม thread ได้เลย ต้องดักจับให้ครบ**ภายใน**thread function
   เอง (Part 83 จะแนะนำ `std::future`/`std::async` ที่ส่ง exception ข้าม thread กลับมา
   ให้อัตโนมัติ ซึ่งสะดวกกว่ามาก)

5. **ใช้วงเล็บกลม `()` แทนวงเล็บปีกกา `{}` ตอนสร้าง object ที่มี `std::thread` เป็น
   argument** — ทำให้เกิด "Most Vexing Parse" ที่ compiler ตีความเป็นการประกาศฟังก์ชัน
   แทนการสร้างตัวแปร (ดูตัวอย่างและ warning `-Wvexing-parse` ใน 81.5) แก้ได้ด้วยการใช้
   Uniform Initialization `{}` ตามที่ Part 80 แนะนำ

6. **เรียก `.join()` ซ้ำสองครั้งบน thread เดียวกัน** — `std::thread::join()` throw
   `std::system_error` ถ้าเรียกตอนที่ `.joinable() == false` แล้ว (เช่นเคย join หรือ
   detach ไปแล้ว) ต่างจาก pthread ที่ `pthread_join()` ซ้ำเป็น Undefined Behavior เฉยๆ
   โดยไม่มีการแจ้งเตือนใดๆ

7. **สร้าง thread จำนวนมากเกินความจำเป็นโดยไม่อ้างอิง `hardware_concurrency()`** —
   การสร้าง thread มากกว่าจำนวน CPU core จริงไม่ได้ทำให้งานเร็วขึ้น มีแต่จะเพิ่ม
   overhead จากการสลับ context (จะเห็นผลกระทบชัดเจนขึ้นเมื่อเรียน Performance
   Profiling ใน Part 86)

---

## แบบฝึกหัดท้ายบท

1. เขียนโปรแกรมสร้างหลาย `std::thread` ที่แต่ละตัวคำนวณ factorial ของตัวเลขคนละตัว
   (พอร์ตจากแบบฝึกหัดข้อ 1 ของ Part 31 ที่ใช้ pthread) โดยใช้ Lambda และ
   `std::vector<std::thread>` แทนการประกาศฟังก์ชันแยกและ `void*`

2. ทดลองลบ `t1.join()` และ `t2.join()` ออกจาก `thread_basic.cpp` ในหัวข้อ 81.2 แล้ว
   สังเกตว่าเกิดอะไรขึ้น (ควรได้ `std::terminate()`) อธิบายด้วยคำพูดตัวเองว่าทำไม

3. เขียน object อีกตัวที่ใช้ `thread_guard` (จาก 81.5) จัดการ thread หลายตัวพร้อมกัน
   (เช่นเก็บ `std::vector<thread_guard>`) แล้วทดสอบว่ายังทำงานถูกต้องเมื่อเกิด
   exception กลางทาง

4. ศึกษาโค้ดต่อไปนี้ที่มีบั๊ก **dangling reference** ซ่อนอยู่ (จงใจเขียนผิดเพื่อการศึกษา)
   แล้วแก้ไขให้ปลอดภัย:

   ```cpp
   struct func {
       int& i;
       explicit func(int& i_) : i(i_) {}
       void operator()() {
           for (int j = 0; j < 3; ++j) {
               std::this_thread::sleep_for(std::chrono::milliseconds(50));
               std::cout << "[detached] i = " << i << "\n"; // อ่าน reference ที่อาจ dangling
           }
       }
   };

   void oops() {
       int some_local_state = 42;
       func my_func(some_local_state);
       std::thread my_thread(my_func);
       my_thread.detach(); // BUG: ไม่รอ thread นี้เลย
   } // some_local_state ถูกทำลายทันทีที่ออกจากฟังก์ชันนี้ แต่ thread อาจยังทำงานอยู่
   ```

5. เขียนโปรแกรมที่พิสูจน์ว่าเรียก `.join()` ซ้ำสองครั้งบน `std::thread` เดียวกันจะ throw
   `std::system_error` (ใบ้: ใช้ `try`/`catch` ห่อการเรียกครั้งที่สอง)

6. เขียนโปรแกรมที่ใช้ `std::thread::hardware_concurrency()` ตัดสินใจว่าจะสร้างกี่
   thread สำหรับแบ่งงานคำนวณผลรวมของ array ขนาดใหญ่ (ปรับจาก 81.7 ให้จำนวน thread ไม่
   ตายตัวที่ 4 อีกต่อไป แต่ขึ้นกับเครื่องที่รันจริง พร้อมมี fallback ถ้าค่าที่ได้เป็น 0)

### แนวทางเฉลยข้อ 1

```cpp
#include <iostream>
#include <thread>
#include <vector>

struct Task {
    int number;
    unsigned long long result = 0;
};

int main() {
    std::vector<Task> tasks = {{5}, {7}, {10}, {12}, {3}, {15}};
    std::vector<std::thread> threads;

    for (auto& task : tasks) {
        // capture by reference ([&task]) เพื่อให้ thread เขียนผลลัพธ์กลับเข้า
        // struct ตัวเดียวกันใน vector ได้โดยตรง (คนละ element กัน ไม่มี race condition)
        threads.emplace_back([&task]() {
            unsigned long long result = 1;
            for (int i = 2; i <= task.number; ++i) {
                result *= static_cast<unsigned long long>(i);
            }
            task.result = result;
        });
    }

    for (auto& t : threads) {
        t.join();
    }

    for (const auto& task : tasks) {
        std::cout << task.number << "! = " << task.result << "\n";
    }

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 -pthread ex1_factorial.cpp -o ex1_factorial
./ex1_factorial
```

ผลลัพธ์:

```
5! = 120
7! = 5040
10! = 3628800
12! = 479001600
3! = 6
15! = 1307674368000
```

**อธิบาย**: เทียบกับ `ex_factorial_threads.c` ของ Part 31 ที่ต้องประกาศ struct `task_t`,
ฟังก์ชัน `compute_factorial` แยก, และ cast `void*` เป็น `task_t*` เอง เวอร์ชันนี้ใช้
Lambda capture `[&task]` ทำหน้าที่แทนทั้งหมด — แต่ละ Lambda อ้างอิง element คนละตัวใน
`vector<Task>` (ผ่าน range-based for loop ที่รับ reference `auto& task`) จึงไม่มี
Race Condition เกิดขึ้นเลย แม้จะ capture by reference ก็ตาม เพราะ**แต่ละ thread เขียน
คนละ element ที่ไม่ทับซ้อนกัน**

### แนวทางเฉลยข้อ 4

```cpp
#include <chrono>
#include <iostream>
#include <thread>

struct func {
    int i; // แก้จาก int& เป็น int -- เก็บ "สำเนา" ของค่า ไม่ใช่ reference
    explicit func(int i_) : i(i_) {}
    void operator()() {
        for (int j = 0; j < 3; ++j) {
            std::this_thread::sleep_for(std::chrono::milliseconds(50));
            std::cout << "[detached] i = " << i << "\n";
        }
    }
};

void fixed() {
    int some_local_state = 42;
    func my_func(some_local_state); // copy ค่าเข้า struct ตั้งแต่ตรงนี้
    std::thread my_thread(my_func);
    my_thread.detach(); // ปลอดภัยแล้ว เพราะ my_func ไม่ได้อ้างอิงกลับไปที่ some_local_state
}

int main() {
    std::cout << "[main] เรียก fixed()\n";
    fixed();
    std::cout << "[main] fixed() คืนค่าแล้ว แต่ thread ยังมีสำเนาของค่าเป็นของตัวเอง\n";
    std::this_thread::sleep_for(std::chrono::milliseconds(300));
    std::cout << "[main] จบโปรแกรม\n";
    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 -pthread ex4_fixed_ref.cpp -o ex4_fixed_ref
./ex4_fixed_ref
```

ผลลัพธ์:

```
[main] เรียก fixed()
[main] fixed() คืนค่าแล้ว แต่ thread ยังมีสำเนาของค่าเป็นของตัวเอง
[detached] i = 42
[detached] i = 42
[detached] i = 42
[main] จบโปรแกรม
```

**อธิบาย**: การแก้ไขคือเปลี่ยน `int& i;` เป็น `int i;` ธรรมดาใน struct `func` ทำให้
constructor `func(int i_) : i(i_)` **copy ค่า** เข้ามาเก็บไว้ตั้งแต่ตอนสร้าง object แทนที่
จะเก็บแค่ reference ที่ชี้กลับไปยัง `some_local_state` เมื่อ `fixed()` return และ
`some_local_state` ถูกทำลาย `my_func` (ซึ่งเป็นสำเนาที่ถูก copy เข้าไปใน `std::thread`
อีกที) ก็ไม่ได้รับผลกระทบเลย เพราะมันมีค่า `42` เป็นของตัวเองอย่างสมบูรณ์แล้ว

ถ้าทดสอบโค้ด "เวอร์ชันบั๊ก" ในแบบฝึกหัดจริงด้วย **AddressSanitizer** (เครื่องมือที่จะ
เรียนเต็มรูปแบบใน Part 96) จะเห็นการแจ้งเตือนที่ชัดเจนมาก:

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 -pthread -fsanitize=address -g dangling_ref.cpp -o dangling_ref_asan
./dangling_ref_asan
```

```
==13566==ERROR: AddressSanitizer: stack-use-after-return on address ...
READ of size 4 at ... thread T1
    #0 ... in func::operator()() dangling_ref.cpp:12
    ...
Address ... is located in stack of thread T0 at offset 48 in frame
    #0 ... in oops() dangling_ref.cpp:17
  This frame has 3 object(s):
    [48, 52) 'some_local_state' (line 18) <== Memory access at offset 48 is inside this variable
```

ข้อความ `stack-use-after-return` ยืนยันตรงตามที่วิเคราะห์ไว้ทุกประการ: thread T1 (worker
ที่ detach ไป) พยายามอ่าน memory ของตัวแปร `some_local_state` ที่อยู่บน stack ของ
thread T0 (main) **หลังจาก** frame ของ `oops()` return ไปแล้ว — เครื่องมือ Sanitizer
แบบนี้จะมีประโยชน์อย่างยิ่งเมื่อโค้ด concurrent ซับซ้อนขึ้นในบท Module G ต่อๆ ไป

---

## สรุปท้ายบท

ใน Part นี้ ซึ่งเป็น Part แรกของ **Module G — Concurrency และ Performance
Engineering** เราได้:

- เข้าใจเหตุผลที่ C++11 เพิ่ม `std::thread` เข้ามา แม้ pthread จาก Part 31-32 จะใช้งาน
  ได้ดีอยู่แล้ว: Portability ข้ามแพลตฟอร์ม, Type Safety ที่ไม่ต้องผ่าน `void*`, และการ
  บูรณาการกับฟีเจอร์อื่นของ Modern C++
- สร้าง จัดการ และรอ thread ด้วย `.join()`/`.detach()`/`.joinable()` ได้อย่างครบถ้วน
- ส่ง argument ให้ thread ทั้งแบบ by value (default ปลอดภัย) และ by reference ผ่าน
  `std::ref` พร้อมเข้าใจ compile error ที่เกิดขึ้นถ้าลืม
- ใช้ Lambda ร่วมกับ `std::thread` เขียนโค้ดที่กระชับกว่าสไตล์ pthread มาก และแก้ปัญหา
  loop counter แบบเดียวกับที่เจอใน Part 31 ได้ตั้งแต่ระดับภาษา
- **สาธิตจริง** ว่าการปล่อยให้ `std::thread` ที่ joinable อยู่ถูก destruct โดยไม่จัดการ
  จะทำให้เกิด `std::terminate()` ทันที และเรียนรู้วิธีป้องกันด้วย RAII (`thread_guard`)
  ตามหลักการที่ Part 68 และ 80 ปูพื้นฐานไว้
- เข้าใจว่า exception ข้าม thread ทำงานอย่างไร (หรือไม่ทำงานเลยถ้าไม่ดักจับให้ถูกที่)
- ใช้ `this_thread::sleep_for/get_id` และ `hardware_concurrency()` ได้อย่างถูกต้อง
- เปรียบเทียบโค้ด pthread กับ `std::thread` แบบ side-by-side เห็นข้อดีที่จับต้องได้จริง

สิ่งที่ Part นี้**ยังไม่ได้แก้**คือปัญหาเดิมที่ Part 32 เคยแก้ด้วย `pthread_mutex_t`:
**Race Condition** ยังคงเกิดขึ้นได้เหมือนเดิมถ้าหลาย `std::thread` เข้าถึงตัวแปรร่วมกัน
โดยไม่มีการป้องกัน (แนวคิดพื้นฐานเรื่อง Critical Section จาก Part 32 ยังใช้ได้ทั้งหมด)
ใน **Part 82** เราจะเรียนรู้เครื่องมือของ Modern C++ ที่ใช้แก้ปัญหานี้:
**`std::mutex`** (คู่กับ RAII wrapper `std::lock_guard`/`std::unique_lock`) และ
**`std::atomic`** (ทางเลือกที่เร็วกว่าสำหรับข้อมูลพื้นฐาน) พร้อมทำความเข้าใจ **C++
Memory Model** ที่อยู่เบื้องหลังการทำงานของทั้งสองอย่างลึกซึ้งยิ่งขึ้น

**ต่อไป:** [Part 82 — std::mutex, std::atomic, C++ Memory Model](./part-082-mutex-atomic-memory-model.md)
