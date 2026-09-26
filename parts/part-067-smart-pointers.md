# Part 67: Smart Pointer (unique_ptr, shared_ptr, weak_ptr) แบบเจาะลึก (Step 529–536)

> Module E — Templates, Generic Programming และ STL | Part 67 จาก 125
> Step ที่ครอบคลุมใน Part นี้: Step 529–536
> Part ก่อนหน้า: [Part 66 — Custom Allocator ใน STL](./part-066-custom-allocators.md) | Part ถัดไป: [Part 68 — RAII และ Resource Management Pattern ระดับ Production](./part-068-raii-patterns.md)

## เป้าหมายการเรียนรู้ (Learning Objectives)

หลังจบ Part นี้ ผู้เรียนจะสามารถ:

1. อธิบายปัญหาที่แท้จริงของการใช้ raw pointer ร่วมกับ `new`/`delete` แบบ manual ได้ครบทั้ง 3 แบบ
   (memory leak, double free, dangling pointer) พร้อมพิสูจน์ด้วยโค้ดที่คอมไพล์และรันได้จริง
2. ใช้ `std::unique_ptr` จัดการ **exclusive ownership** ได้อย่างถูกต้อง เข้าใจว่าทำไมมัน copy
   ไม่ได้ ต้อง `std::move` เท่านั้นในการย้ายความเป็นเจ้าของ
3. ใช้ `std::make_unique` สร้าง `unique_ptr` ได้อย่างปลอดภัยและเป็นสำนวนมาตรฐาน แทนการเขียน
   `new` ตรงๆ
4. ใช้ `std::shared_ptr` จัดการ **shared ownership** ผ่านกลไก **reference counting** และอ่านค่า
   `use_count()` เพื่อทำความเข้าใจว่าใครเป็นเจ้าของ object ร่วมกันอยู่บ้าง ณ ขณะหนึ่งๆ
5. อธิบายได้ว่าทำไม `std::make_shared` มีประสิทธิภาพดีกว่า `shared_ptr(new T)` ในแง่จำนวนครั้ง
   ที่จองหน่วยความจำ และรู้ข้อยกเว้นที่ยังต้องใช้ `shared_ptr(new T)` แทน
6. วิเคราะห์และแก้ปัญหา **circular reference** ของ `shared_ptr` ด้วย `std::weak_ptr` ได้ โดยใช้
   ตัวอย่าง Parent-Child ที่พิสูจน์ว่ารั่วไหลจริงด้วย valgrind แล้วแก้ให้หายขาด
7. เขียน **custom deleter** สำหรับ smart pointer เพื่อจัดการทรัพยากรที่ไม่ใช่ memory ธรรมดา
   เช่น `FILE*`, array แบบ raw, หรือ `pthread_mutex_t` จาก Module C
8. เลือกใช้ smart pointer แต่ละชนิด (`unique_ptr` / `shared_ptr` / `weak_ptr` / raw pointer)
   ได้อย่างถูกต้องเหมาะสมกับสถานการณ์ ตามหลัก "กฎทอง" ของการจัดการ ownership ในหลักสูตรนี้

---

## 67.1 ปัญหาที่แท้จริงของ raw pointer + new/delete (Step 529)

ย้อนกลับไปที่ **Part 55 (โปรเจกต์ Library Management System)** เราตัดสินใจใช้
`std::vector<std::unique_ptr<Media>>` เก็บสื่อทั้งหมดตั้งแต่ต้น โดยบอกไว้สั้นๆ ว่า "ไม่มีการ
`new`/`delete` แบบ manual เลยแม้แต่บรรทัดเดียวในทั้งโปรเจกต์" และผัดไว้ว่าเหตุผลเชิงลึกของ
`unique_ptr`/`shared_ptr`/`weak_ptr` จะเรียนเต็มรูปแบบใน Part นี้ ถึงเวลาแล้วที่จะเข้าใจว่า
ทำไมการตัดสินใจนั้นถึงสำคัญมาก โดยเริ่มจากการดูปัญหาจริงที่เกิดขึ้นถ้า **ไม่ใช้** smart pointer

### ปัญหาที่ 1: Memory Leak เมื่อลืม delete (หรือ delete ไม่ถึงเพราะ exception)

สมมติเราเขียนโค้ดแบบ "สมัยก่อน" ที่ยังไม่มี smart pointer โดยจัดการ `Media` ด้วย raw pointer
ตรงๆ:

```cpp
// 01_leak.cpp - สาธิตปัญหาของ raw pointer + new/delete (เจตนาให้ leak)
#include <iostream>
#include <stdexcept>
#include <string>

class Media {
public:
    explicit Media(std::string title) : title_(std::move(title)) {
        std::cout << "  [สร้าง] " << title_ << '\n';
    }
    ~Media() {
        std::cout << "  [ทำลาย] " << title_ << '\n';
    }
    const std::string& title() const { return title_; }
private:
    std::string title_;
};

void processCatalog(bool triggerError) {
    Media* book = new Media("C++ Primer");   // (1) จัดสรรด้วย raw pointer

    if (triggerError) {
        // (2) สมมติว่าตรงนี้เกิด exception หรือ early return ระหว่างทาง
        throw std::runtime_error("ข้อมูลแคตตาล็อกเสียหาย");
        // ไม่มีวันไปถึง delete book; ด้านล่างเลย -> memory leak แน่นอน
    }

    std::cout << "  ประมวลผล " << book->title() << " สำเร็จ\n";
    delete book; // (3) ต้องจำ delete เอง — และบรรทัดนี้ถูกข้ามไปเมื่อ throw ก่อนหน้านี้
}

int main() {
    std::cout << "-- เรียกแบบไม่มี error --\n";
    processCatalog(false);

    std::cout << "-- เรียกแบบเกิด error --\n";
    try {
        processCatalog(true);
    } catch (const std::exception& e) {
        std::cout << "จับ error ได้: " << e.what() << '\n';
    }
    std::cout << "โปรแกรมจบการทำงาน (แต่ Media ตัวที่ 2 รั่วไหลไปแล้ว!)\n";
}
```

คอมไพล์และรัน:

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 01_leak.cpp -o 01_leak
./01_leak
```

ผลลัพธ์:

```
-- เรียกแบบไม่มี error --
  [สร้าง] C++ Primer
  ประมวลผล C++ Primer สำเร็จ
  [ทำลาย] C++ Primer
-- เรียกแบบเกิด error --
  [สร้าง] C++ Primer
จับ error ได้: ข้อมูลแคตตาล็อกเสียหาย
โปรแกรมจบการทำงาน (แต่ Media ตัวที่ 2 รั่วไหลไปแล้ว!)
```

สังเกตว่าในการเรียกครั้งที่สอง **ไม่มีข้อความ `[ทำลาย] C++ Primer` เลย** — object ตัวที่สอง
ไม่เคยถูก destroy ยืนยันด้วย **valgrind** (ทบทวนจาก Part 38):

```bash
valgrind --leak-check=full ./01_leak
```

```
==1100== HEAP SUMMARY:
==1100==     in use at exit: 32 bytes in 1 blocks
==1100==   total heap usage: 6 allocs, 5 frees, 78,123 bytes allocated
==1100==
==1100== 32 bytes in 1 blocks are definitely lost in loss record 1 of 1
==1100==    at 0x4846FA3: operator new(unsigned long)
==1100==    by 0x10A497: processCatalog(bool)
==1100==    by 0x10A6AF: main
==1100==
==1100== LEAK SUMMARY:
==1100==    definitely lost: 32 bytes in 1 blocks
```

valgrind ยืนยันชัดเจนว่ามีหน่วยความจำ **32 bytes รั่วไหลแบบ "definitely lost"** และชี้ตรงไปที่
บรรทัด `new Media(...)` ใน `processCatalog` เป๊ะๆ — นี่คือ**สาเหตุอันดับหนึ่ง**ที่ทำให้โปรแกรม
C++ ขนาดใหญ่ค่อยๆ กิน RAM มากขึ้นเรื่อยๆ จนต้อง restart (memory leak สะสม) ยิ่งโค้ดมีจุดที่
อาจ `throw`, `return` กลางทาง, หรือ `continue`/`break` ออกจาก loop มากเท่าไหร่ ก็ยิ่งมีจุดเสี่ยง
ที่บรรทัด `delete` จะ "ไปไม่ถึง" มากขึ้นเท่านั้น

### ปัญหาที่ 2: Double Free และ Dangling Pointer

ปัญหาที่อันตรายยิ่งกว่า memory leak (ซึ่งอย่างน้อยโปรแกรมยังทำงานต่อได้แค่กิน RAM มากขึ้น) คือ
**double free** — การ `delete` หน่วยความจำก้อนเดียวกันซ้ำสองครั้ง ซึ่งเป็น **Undefined Behavior**
เต็มรูปแบบ (อาจ crash ทันที, อาจทำให้ heap เสียหายแบบเงียบๆ แล้วไป crash ที่จุดอื่นในภายหลัง
ซึ่ง debug ยากมาก):

```cpp
// 02_doublefree.cpp - สาธิต double free และ dangling pointer (คอมไพล์ผ่าน แต่ runtime เป็น UB)
#include <iostream>

struct Widget {
    int value;
};

int main() {
    Widget* a = new Widget{42};
    Widget* b = a;      // (1) b ชี้ไปที่ก้อนเดียวกับ a โดยไม่มีระบบนับจำนวนเจ้าของ

    std::cout << "a->value = " << a->value << ", b->value = " << b->value << '\n';

    delete a;            // (2) คืนหน่วยความจำผ่าน a
    // delete b;          // (3) ถ้าเปิดบรรทัดนี้ -> double free = Undefined Behavior ทันที

    // a และ b ตอนนี้เป็น dangling pointer ทั้งคู่ (ชี้ไปที่หน่วยความจำที่ถูกคืนแล้ว)
    // การเข้าถึง a->value หรือ b->value ต่อจากนี้คือ Undefined Behavior เช่นกัน
    std::cout << "โปรแกรมยังไม่ delete b เพื่อความปลอดภัยของตัวอย่างนี้\n";
    return 0;
}
```

```
a->value = 42, b->value = 42
โปรแกรมยังไม่ delete b เพื่อความปลอดภัยของตัวอย่างนี้
```

โค้ดข้างต้น**จงใจไม่เปิดบรรทัด `delete b;`** เพื่อไม่ให้ Undefined Behavior เกิดขึ้นจริงใน
ตัวอย่างที่แจกจ่าย แต่ประเด็นสำคัญคือ: **compiler ไม่มีทางเตือนเรื่องนี้ให้เราเลย** เพราะในสายตา
ของ compiler `a` และ `b` เป็นแค่ตัวแปร pointer ธรรมดาสองตัวที่บังเอิญเก็บค่า address เดียวกัน
มันไม่รู้ (และไม่มีทางรู้ได้จากไวยากรณ์ raw pointer) ว่า "ใครคือเจ้าของตัวจริง" ที่ควรรับผิดชอบ
การ `delete`

**Dangling pointer** คือปัญหาที่ต่อเนื่องกัน: หลัง `delete a;` ตัวแปร `a` และ `b` ทั้งคู่ยังคง
เก็บ address เดิมอยู่ (C++ ไม่ได้ตั้งค่าให้เป็น `nullptr` อัตโนมัติ) การเข้าถึง `a->value` หรือ
`b->value` ต่อจากนั้นคือการอ่าน/เขียนหน่วยความจำที่**ไม่ได้เป็นของโปรแกรมเราแล้ว** ผลลัพธ์
อาจจะ "ดูเหมือนทำงานถูกต้อง" ไปอีกหลายครั้ง (เพราะ OS ยังไม่ได้เอาหน่วยความจำนั้นไปให้คนอื่น)
ก่อนจะพังแบบไม่มีปี่มีขลุ่ยในภายหลัง ซึ่งเป็นฝันร้ายที่สุดของการ debug

### สรุปปัญหาทั้งสามของ raw pointer + manual memory management

| ปัญหา | สาเหตุ | ผลกระทบ |
|---|---|---|
| **Memory Leak** | ลืม `delete`, หรือ `delete` ไปไม่ถึงเพราะ exception/early return | โปรแกรมกิน RAM เพิ่มขึ้นเรื่อยๆ จนอาจถูก OS ฆ่าทิ้ง (OOM Kill) |
| **Double Free** | `delete` หน่วยความจำก้อนเดียวกันมากกว่าหนึ่งครั้ง | Undefined Behavior — heap เสียหาย, crash แบบสุ่ม, security vulnerability |
| **Dangling Pointer** | ใช้ pointer ต่อหลังจาก `delete` ไปแล้ว | อ่าน/เขียนหน่วยความจำที่ไม่ใช่ของเรา ผลลัพธ์คาดเดาไม่ได้ |

รากของปัญหาทั้งสามข้อคือสิ่งเดียวกัน: **raw pointer ไม่มีแนวคิดเรื่อง "ความเป็นเจ้าของ
(ownership)" ติดตัวมันมาเลย** มันเป็นแค่ตัวเลข address ที่ใครจะ `delete` เมื่อไหร่ กี่ครั้ง
ก็ได้ตามใจ ไม่มีกฎอะไรบังคับ วิธีแก้ที่ C++11 นำมาใช้คือการเอาแนวคิด **RAII (Resource
Acquisition Is Initialization)** ที่เรียนไปแล้วใน **Part 46** และขยายความอย่างจริงจังใน
**Part 54** มาผูกกับ pointer โดยตรง จนกลายเป็น **Smart Pointer** — object ขนาดเล็กที่ห่อ raw
pointer เอาไว้ และรับผิดชอบเรื่อง `delete` ให้เราอัตโนมัติผ่าน destructor ทำให้ปัญหาทั้งสามข้อ
ข้างต้นหายไปเกือบหมดโดยที่เราแทบไม่ต้องคิดเรื่อง memory เองเลย

---

## 67.2 std::unique_ptr: Exclusive Ownership (Step 530)

`std::unique_ptr<T>` (อยู่ใน header `<memory>`) คือ smart pointer ที่สื่อความหมายว่า **"ฉันคือ
เจ้าของเพียงหนึ่งเดียวของ object นี้"** — ไม่มีใครเป็นเจ้าของร่วมได้เลย เมื่อ `unique_ptr`
หมด scope หรือถูกทำลาย มันจะเรียก `delete` ให้ object ที่มันถืออยู่โดยอัตโนมัติเสมอ (ตามหลัก
RAII) ไม่ว่าจะออกจาก scope แบบปกติหรือผ่าน exception ก็ตาม

### กฎเหล็กของ unique_ptr: Copy ไม่ได้ Move ได้เท่านั้น

เพื่อรักษาสัญญาที่ว่า "มีเจ้าของเพียงหนึ่งเดียวเสมอ" `unique_ptr` จึง **ลบ copy constructor
และ copy assignment operator ทิ้งไปโดยตั้งใจ** (`= delete`) การจะย้ายความเป็นเจ้าของทำได้ทาง
เดียวเท่านั้นคือผ่าน **move semantics** (`std::move`) ซึ่งเป็นการ "โอนสิทธิ์" ไม่ใช่การ "สำเนา"
— หลัง move แล้ว ตัวต้นทางจะกลายเป็น `nullptr` ทันที ไม่มีทางมี `unique_ptr` สองตัวชี้ไปที่
object เดียวกันพร้อมกันได้เลยในเวลาใดๆ ก็ตาม (เรื่อง move semantics แบบเจาะลึกจะเรียนเต็ม
รูปแบบใน **Part 70**)

```cpp
// 03_unique_ptr.cpp
#include <iostream>
#include <memory>
#include <string>

class Media {
public:
    explicit Media(std::string title) : title_(std::move(title)) {
        std::cout << "  [สร้าง] " << title_ << '\n';
    }
    ~Media() {
        std::cout << "  [ทำลาย] " << title_ << '\n';
    }
    const std::string& title() const { return title_; }
private:
    std::string title_;
};

void describe(const std::unique_ptr<Media>& m) {
    if (m) {
        std::cout << "  กำลังดูแล: " << m->title() << '\n';
    } else {
        std::cout << "  (ไม่มี Media อยู่ในความดูแล)\n";
    }
}

std::unique_ptr<Media> createMedia(const std::string& title) {
    return std::make_unique<Media>(title); // ย้ายค่ากลับด้วย move อัตโนมัติ (RVO/NRVO)
}

int main() {
    std::cout << "-- สร้าง unique_ptr ด้วย make_unique --\n";
    std::unique_ptr<Media> owner1 = std::make_unique<Media>("Effective Modern C++");
    describe(owner1);

    std::cout << "-- ย้ายความเป็นเจ้าของด้วย std::move --\n";
    std::unique_ptr<Media> owner2 = std::move(owner1); // ownership ย้ายจาก owner1 ไป owner2
    describe(owner1); // owner1 กลายเป็น nullptr แล้ว
    describe(owner2);

    std::cout << "-- คืนค่าจากฟังก์ชันด้วย move semantics --\n";
    std::unique_ptr<Media> owner3 = createMedia("The Pragmatic Programmer");
    describe(owner3);

    std::cout << "-- reset() เพื่อคืนทรัพยากรก่อนหมด scope --\n";
    owner3.reset(); // เรียก destructor ของ Media ทันที ไม่ต้องรอจบ scope
    describe(owner3);

    std::cout << "-- จบ main: ที่เหลือถูกทำลายอัตโนมัติ --\n";
    return 0;
}
```

ผลลัพธ์:

```
-- สร้าง unique_ptr ด้วย make_unique --
  [สร้าง] Effective Modern C++
  กำลังดูแล: Effective Modern C++
-- ย้ายความเป็นเจ้าของด้วย std::move --
  (ไม่มี Media อยู่ในความดูแล)
  กำลังดูแล: Effective Modern C++
-- คืนค่าจากฟังก์ชันด้วย move semantics --
  [สร้าง] The Pragmatic Programmer
  กำลังดูแล: The Pragmatic Programmer
-- reset() เพื่อคืนทรัพยากรก่อนหมด scope --
  [ทำลาย] The Pragmatic Programmer
  (ไม่มี Media อยู่ในความดูแล)
-- จบ main: ที่เหลือถูกทำลายอัตโนมัติ --
  [ทำลาย] Effective Modern C++
```

ตรวจสอบด้วย valgrind แล้วไม่มี leak เลยแม้แต่ byte เดียว — ไม่ว่าจะสร้าง, ย้าย, หรือ `reset()`
กี่ครั้งก็ตาม เพราะทุกเส้นทางจบลงด้วยการเรียก destructor ของ `Media` ครบถ้วนเสมอ

ถ้าลองเขียนโค้ดที่พยายาม **copy** `unique_ptr` (แทนที่จะ move) compiler จะปฏิเสธทันทีตั้งแต่
ขั้นตอนคอมไพล์ — นี่คือจุดแข็งที่สุดของ `unique_ptr`: **บั๊กเรื่อง ownership ไม่ชัดเจนกลายเป็น
compile-time error แทนที่จะเป็น runtime bug**

```cpp
std::unique_ptr<int> p1 = std::make_unique<int>(10);
std::unique_ptr<int> p2 = p1; // ERROR: copy constructor ถูกลบไว้ (deleted)
```

```
error: use of deleted function 'std::unique_ptr<_Tp, _Dp>::unique_ptr(const std::unique_ptr<_Tp, _Dp>&)
       [with _Tp = int; _Dp = std::default_delete<int>]'
note: declared here
  unique_ptr(const unique_ptr&) = delete;
```

### unique_ptr กับ Array

`unique_ptr` รองรับการห่อ array แบบ dynamic ผ่าน specialization `unique_ptr<T[]>` ซึ่งจะเรียก
`delete[]` ให้อัตโนมัติแทนที่จะเป็น `delete` ธรรมดา (สำคัญมาก — ถ้าใช้ผิดชนิดจะเป็น Undefined
Behavior):

```cpp
std::unique_ptr<int[]> buffer = std::make_unique<int[]>(5);
for (int i = 0; i < 5; ++i) buffer[i] = i * i;
for (int i = 0; i < 5; ++i) std::cout << buffer[i] << ' ';
std::cout << '\n';
```

```
0 1 4 9 16
```

### ตารางฟังก์ชันหลักของ unique_ptr

| ฟังก์ชัน/Operator | ความหมาย |
|---|---|
| `get()` | คืน raw pointer ที่ห่ออยู่ข้างใน (ไม่ transfer ownership — ใช้เพื่อส่งให้ API ที่ต้องการ raw pointer เท่านั้น) |
| `release()` | **ปลด** ownership ออกจาก `unique_ptr` แล้วคืน raw pointer กลับมา (จากนี้ผู้เรียกต้องรับผิดชอบ `delete` เอง) |
| `reset(ptr = nullptr)` | ทำลาย object เดิม (ถ้ามี) แล้วเปลี่ยนไปถือ `ptr` ใหม่ (หรือไม่ถืออะไรเลยถ้าไม่ระบุ) |
| `operator bool()` | ตรวจสอบว่ากำลังถือ object อยู่หรือไม่ (เทียบเท่า `get() != nullptr`) |
| `operator*`, `operator->` | เข้าถึง object ที่ห่ออยู่เหมือน raw pointer ปกติทุกประการ |
| `swap(other)` | สลับความเป็นเจ้าของกับ `unique_ptr` อีกตัว |

> **ข้อควรระวัง**: `get()` กับ `release()` มีความหมายต่างกันโดยสิ้นเชิง — `get()` แค่ "ยืมดู"
> raw pointer โดยที่ `unique_ptr` ยังเป็นเจ้าของเหมือนเดิม ส่วน `release()` คือ "สละสิทธิ์"
> ความเป็นเจ้าของออกไปทั้งหมด ถ้าสับสนสองตัวนี้จะกลับไปเจอ memory leak หรือ double free แบบ
> เดิมได้ทันที

---

## 67.3 std::make_unique: สร้าง unique_ptr อย่างปลอดภัย (Step 531)

`std::make_unique<T>(args...)` (เพิ่มเข้ามาใน **C++14**) เป็นฟังก์ชันมาตรฐานที่สร้าง object
ชนิด `T` ด้วย `args...` แล้วห่อด้วย `unique_ptr<T>` ให้เสร็จในขั้นตอนเดียว เทียบกับการเขียน
`std::unique_ptr<T>(new T(args...))` ตรงๆ แล้ว `make_unique` ดีกว่าด้วยเหตุผลสามข้อ:

1. **สั้นกว่าและไม่ต้องพิมพ์ชื่อ type ซ้ำสองครั้ง** (`unique_ptr<Media>` ไม่ต้องมี `new Media`
   ตามหลังให้ดูรกตา)
2. **ไม่มีคำว่า `new` ปรากฏในโค้ดผู้ใช้เลย** — สอดคล้องกับแนวทาง "Modern C++ ไม่ควรมี `new`/
   `delete` ตรงๆ ในโค้ดระดับ business logic" ที่จะเป็นหัวใจของ **Part 80 (Modern C++ Best
   Practice)**
3. **ปลอดภัยกว่าในกรณีที่ซับซ้อน** — ดูตัวอย่างต่อไปนี้ ซึ่งเป็นข้อผิดพลาดคลาสสิกที่มีการพูด
   ถึงกันมากในวงการ C++ (จากหนังสือ *Effective Modern C++* ของ Scott Meyers)

### ทำไมการเขียน new ตรงๆ ปนกับอาร์กิวเมนต์อื่นถึงเสี่ยง

```cpp
// 11_exception_unsafe_new.cpp
#include <iostream>
#include <memory>
#include <stdexcept>

struct Widget {
    Widget() { std::cout << "  [สร้าง Widget]\n"; }
    ~Widget() { std::cout << "  [ทำลาย Widget]\n"; }
};

int computePriority() {
    throw std::runtime_error("คำนวณ priority ล้มเหลว");
}

void process(std::shared_ptr<Widget> w, int priority) {
    std::cout << "  process priority=" << priority << " widget=" << (w ? "ok" : "null") << '\n';
}

// วิธีที่ "เสี่ยง" ในทางทฤษฎี: คอมไพเลอร์มีสิทธิ์ประเมินอาร์กิวเมนต์ทั้งสองของ process()
// ในลำดับใดก็ได้ (unspecified order ระหว่างอาร์กิวเมนต์ที่ต่างกัน) กล่าวคือมีสิทธิ์เรียก
// `new Widget()` "ก่อน" แล้วค่อยเรียก `computePriority()` (ที่ throw) ก่อนที่ shared_ptr<Widget>
// จะถูกสร้างเสร็จสมบูรณ์จาก raw pointer นั้น ถ้าเกิดกรณีนี้ Widget ที่เพิ่ง new จะไม่มีใคร
// เป็นเจ้าของเลย -> รั่วไหลทันทีเพราะไม่มี shared_ptr ห่อมันไว้แล้ว
void demoRiskyPattern() {
    try {
        process(std::shared_ptr<Widget>(new Widget()), computePriority());
    } catch (const std::exception& e) {
        std::cout << "จับ error ได้: " << e.what() << '\n';
    }
}

// วิธีที่ปลอดภัยเสมอ: สร้าง smart pointer ให้ "เสร็จสมบูรณ์เป็นค่า" ก่อนส่งเข้าฟังก์ชัน
// ไม่มีทางเกิดช่วงเวลาที่ raw pointer ลอยอยู่โดยไม่มีเจ้าของเลย
void demoSafePattern() {
    try {
        std::shared_ptr<Widget> w = std::make_shared<Widget>(); // ขั้นตอนนี้จบสมบูรณ์ก่อนเสมอ
        process(w, computePriority());
    } catch (const std::exception& e) {
        std::cout << "จับ error ได้: " << e.what() << '\n';
    }
}

int main() {
    std::cout << "-- รูปแบบเสี่ยง (new ตรงๆ ปนกับอาร์กิวเมนต์ที่ throw ได้) --\n";
    demoRiskyPattern();

    std::cout << "-- รูปแบบปลอดภัย (make_shared แยกเป็นค่าก่อนเสมอ) --\n";
    demoSafePattern();

    return 0;
}
```

ผลลัพธ์ (จากคอมไพเลอร์และเครื่องที่ใช้ทดสอบในบทเรียนนี้):

```
-- รูปแบบเสี่ยง (new ตรงๆ ปนกับอาร์กิวเมนต์ที่ throw ได้) --
จับ error ได้: คำนวณ priority ล้มเหลว
-- รูปแบบปลอดภัย (make_shared แยกเป็นค่าก่อนเสมอ) --
  [สร้าง Widget]
  [ทำลาย Widget]
จับ error ได้: คำนวณ priority ล้มเหลว
```

สังเกตว่าใน `demoRiskyPattern()` **ไม่มีข้อความ `[สร้าง Widget]` ปรากฏเลย** — แปลว่าคอมไพเลอร์
ที่ใช้ทดสอบเลือกประเมิน `computePriority()` (ซึ่ง throw) **ก่อน** `new Widget()` ในการรันครั้งนี้
จึงไม่มี `Widget` ถูกสร้างขึ้นมาเลยและไม่รั่วไหล — แต่นี่คือ**เรื่องบังเอิญของคอมไพเลอร์ตัวนี้
เท่านั้น** มาตรฐาน C++ **ไม่ได้การันตีลำดับการประเมินอาร์กิวเมนต์ระหว่างกัน** เอาไว้เลย
คอมไพเลอร์ตัวอื่น เวอร์ชันอื่น หรือแม้แต่ optimization level ที่ต่างกันของคอมไพเลอร์ตัวเดียวกัน
มีสิทธิ์เลือกอีกลำดับหนึ่งได้เสมอ ซึ่งถ้าเลือก `new Widget()` ก่อน จะเกิด memory leak ทันทีตาม
ที่อธิบายในคอมเมนต์ นี่คือเหตุผลที่กฎทองคือ: **อย่าเขียน `new` ปนอยู่ในอาร์กิวเมนต์ของฟังก์ชัน
เด็ดขาด ให้สร้าง smart pointer ให้เสร็จเป็นตัวแปรก่อนเสมอ** (`make_unique`/`make_shared` บังคับ
ให้เขียนแบบนี้โดยธรรมชาติอยู่แล้ว เพราะไม่มี `new` ให้แยกออกมาปนกับอะไรได้เลย)

> **หมายเหตุ**: ก่อน C++14 (ที่ยังไม่มี `make_unique`) มาตรฐานกำหนดให้ต้องเขียน
> `std::unique_ptr<T>(new T(...))` ตรงๆ เท่านั้น — บทเรียนนี้ใช้ C++17 เป็นมาตรฐานหลักของ
> หลักสูตร จึงแนะนำให้ใช้ `make_unique`/`make_shared` เสมอเมื่อทำได้ ยกเว้นกรณีพิเศษที่จะพูดถึง
> ในหัวข้อ 67.5 (custom deleter) ซึ่ง `make_unique`/`make_shared` ไม่รองรับการระบุ custom
> deleter ให้ ต้องเขียน `new` ตรงๆ ปนกับ deleter เอง

---

## 67.4 std::shared_ptr: Shared Ownership และ Reference Counting (Step 532)

ในโลกจริง ไม่ใช่ทุก object จะมีเจ้าของเดียวเสมอไป — บางครั้ง object หนึ่งต้องถูก "ใช้ร่วมกัน"
โดยหลายส่วนของโปรแกรมพร้อมกัน โดยไม่มีใครรู้แน่ชัดว่าใครจะเป็นคนสุดท้ายที่เลิกใช้มัน กรณีแบบนี้
`unique_ptr` ใช้ไม่ได้ (เพราะมันบังคับว่ามีเจ้าของเดียว) ต้องใช้ `std::shared_ptr<T>` แทน

`shared_ptr` ทำงานด้วยกลไก **Reference Counting**: มันเก็บตัวนับ (**control block**) แยกจาก
object จริง โดยทุกครั้งที่มีการ copy `shared_ptr` ตัวนับจะ **เพิ่มขึ้น 1** และทุกครั้งที่
`shared_ptr` ตัวใดตัวหนึ่งถูกทำลาย (หมด scope) ตัวนับจะ **ลดลง 1** — เมื่อตัวนับลดลงเหลือ **0**
เท่านั้น (แปลว่าไม่มีใครถือ object นี้อยู่แล้วจริงๆ) มันจึงจะเรียก `delete` object จริงให้
อัตโนมัติ

```cpp
// 05_shared_ptr.cpp
#include <iostream>
#include <memory>
#include <string>
#include <vector>

class Media {
public:
    explicit Media(std::string title) : title_(std::move(title)) {
        std::cout << "  [สร้าง] " << title_ << '\n';
    }
    ~Media() {
        std::cout << "  [ทำลาย] " << title_ << '\n';
    }
    const std::string& title() const { return title_; }
private:
    std::string title_;
};

int main() {
    std::cout << "-- สร้าง shared_ptr ตัวแรก --\n";
    std::shared_ptr<Media> p1 = std::make_shared<Media>("Design Patterns");
    std::cout << "use_count = " << p1.use_count() << '\n';

    {
        std::cout << "-- copy ไปยัง p2 (เข้า scope ใหม่) --\n";
        std::shared_ptr<Media> p2 = p1; // เพิ่ม reference count เป็น 2
        std::cout << "use_count = " << p1.use_count() << '\n';

        std::vector<std::shared_ptr<Media>> shelf;
        shelf.push_back(p1); // เพิ่มเป็น 3
        std::cout << "use_count = " << p1.use_count() << '\n';
    } // p2 และ shelf ออกจาก scope -> count ลดกลับเหลือ 1

    std::cout << "-- ออกจาก scope ด้านใน --\n";
    std::cout << "use_count = " << p1.use_count() << '\n';

    std::cout << "-- จบ main --\n";
    return 0;
}
```

ผลลัพธ์:

```
-- สร้าง shared_ptr ตัวแรก --
  [สร้าง] Design Patterns
use_count = 1
-- copy ไปยัง p2 (เข้า scope ใหม่) --
use_count = 2
use_count = 3
-- ออกจาก scope ด้านใน --
use_count = 1
-- จบ main --
  [ทำลาย] Design Patterns
```

สังเกตว่า `Media` ถูกทำลายที่บรรทัดสุดท้ายเท่านั้น (ตอน `p1` ออกจาก `main`) เพราะนั่นคือจุดที่
`use_count()` ลดลงเหลือ **0** ในที่สุด — ไม่ว่าจะมีใครมา copy `shared_ptr` ไปกี่ตัว ไปเก็บใน
`vector` กี่ที่ ก็ไม่ต้องกังวลเรื่องว่า "ใครจะเป็นคนตัดสินใจ `delete`" อีกต่อไป เพราะระบบนับ
จำนวนจะจัดการให้เองโดยอัตโนมัติ ต่างจาก `unique_ptr` ที่ **copy ได้ตามปกติ** (ไม่ถูกลบเหมือน
`unique_ptr`) และ copy แต่ละครั้งไม่ได้สร้าง object ใหม่ แต่เป็นการ "เพิ่มผู้ถือครองร่วม"
เข้าไปในระบบนับ

> **ข้อควรระวัง**: `use_count()` มีไว้เพื่อ **debug และทำความเข้าใจ** เป็นหลัก ไม่ควรใช้เป็น
> ส่วนหนึ่งของ logic การตัดสินใจในโปรแกรม production (เช่น `if (p.use_count() == 1) { ... }`)
> เพราะในโปรแกรมที่มีหลาย thread ค่านี้อาจเปลี่ยนไปได้ทันทีหลังอ่านเสร็จ (race condition
> ทบทวน Part 32) ทำให้ logic ที่พึ่งพาค่านี้ไม่น่าเชื่อถือ

---

## 67.5 std::make_shared และเหตุผลที่มีประสิทธิภาพกว่า shared_ptr(new T) (Step 533)

เช่นเดียวกับ `make_unique`, `std::make_shared<T>(args...)` คือวิธีมาตรฐานในการสร้าง
`shared_ptr` — แต่ในกรณีของ `shared_ptr` มี**เหตุผลด้านประสิทธิภาพที่หนักแน่นกว่ามาก**ที่ควร
ใช้ `make_shared` แทนการเขียน `shared_ptr<T>(new T(...))` ตรงๆ

### โครงสร้างภายในของ shared_ptr: Object + Control Block

`shared_ptr` ไม่ได้เก็บแค่ raw pointer ไปยัง object แต่ยังต้องเก็บ **control block** ที่มี
ตัวนับ reference count (และ weak count ที่จะพูดถึงในหัวข้อถัดไป) ด้วยเสมอ คำถามคือ: object
กับ control block นี้ถูกจัดสรรในหน่วยความจำแยกกันหรือรวมกัน?

- **`std::shared_ptr<T> p(new T(...))`**: `new T(...)` จองหน่วยความจำสำหรับ object **ครั้งที่
  หนึ่ง** จากนั้น constructor ของ `shared_ptr` จะจองหน่วยความจำสำหรับ control block แยกต่างหาก
  **อีกครั้งที่สอง** — รวมเป็น **2 ครั้งของการจัดสรรหน่วยความจำ (heap allocation)**
- **`std::make_shared<T>(args...)`**: จะจอง**หน่วยความจำก้อนเดียว** ที่มีขนาดใหญ่พอสำหรับทั้ง
  object และ control block รวมกัน — **1 ครั้งเท่านั้น**

การจัดสรรหน่วยความจำ (`malloc`/`new`) เป็นหนึ่งใน operation ที่ "แพง" ที่สุดในโปรแกรม เพราะ
ต้องติดต่อกับ memory allocator ของระบบ (ทบทวน Part 66 เรื่อง custom allocator) การลดจำนวนครั้ง
ที่ต้องจัดสรรจาก 2 เหลือ 1 จึงส่งผลต่อประสิทธิภาพจริง โดยเฉพาะเมื่อสร้าง `shared_ptr` จำนวนมาก
ในลูปที่ทำงานหนัก:

```cpp
// 06_make_shared_perf.cpp
#include <chrono>
#include <iostream>
#include <memory>

struct Data {
    int values[8];
};

constexpr int kIterations = 2'000'000;

int main() {
    using clock = std::chrono::steady_clock;

    // วิธีที่ 1: shared_ptr(new T) -> จอง 2 ครั้ง (control block แยกจาก object)
    auto start1 = clock::now();
    for (int i = 0; i < kIterations; ++i) {
        std::shared_ptr<Data> p(new Data{});
        (void)p;
    }
    auto end1 = clock::now();

    // วิธีที่ 2: make_shared<T>() -> จองครั้งเดียว (object + control block รวมกัน)
    auto start2 = clock::now();
    for (int i = 0; i < kIterations; ++i) {
        auto p = std::make_shared<Data>();
        (void)p;
    }
    auto end2 = clock::now();

    auto ms1 = std::chrono::duration_cast<std::chrono::milliseconds>(end1 - start1).count();
    auto ms2 = std::chrono::duration_cast<std::chrono::milliseconds>(end2 - start2).count();

    std::cout << "shared_ptr(new T):     " << ms1 << " ms\n";
    std::cout << "make_shared<T>():      " << ms2 << " ms\n";
    return 0;
}
```

คอมไพล์ด้วย optimization (สำคัญมากเวลาวัด performance จริง — ทบทวนหลักการนี้จะเจาะลึกเต็ม
รูปแบบใน Part 86):

```bash
g++ -O2 -Wall -Wextra -Wpedantic -std=c++17 06_make_shared_perf.cpp -o 06_perf
./06_perf
```

ผลลัพธ์ตัวอย่าง (จากเครื่องที่ใช้ทดสอบ ตัวเลขจริงจะต่างกันไปตาม CPU/allocator ของแต่ละเครื่อง
แต่ทิศทางที่ `make_shared` เร็วกว่าจะคงที่เสมอ):

```
shared_ptr(new T):     62 ms
make_shared<T>():      32 ms
```

`make_shared` เร็วกว่าเกือบ 2 เท่าในตัวอย่างนี้ ซึ่งมาจากการลดจำนวนครั้งการจัดสรรหน่วยความจำ
ลงครึ่งหนึ่งพอดี นอกจากความเร็วแล้ว การรวม object กับ control block ไว้ก้อนเดียวกันยังทำให้
**cache locality** ดีขึ้นด้วย (ข้อมูลทั้งสองส่วนอยู่ใกล้กันใน RAM มากกว่า) ซึ่งจะเจาะลึกเรื่อง
cache-friendly code ใน Part 87

### ข้อยกเว้น: เมื่อไหร่ที่ยังต้องใช้ shared_ptr(new T) แทน

แม้ `make_shared` จะเร็วกว่าและควรใช้เป็นค่าเริ่มต้นเสมอ แต่มีสองกรณีที่ยังจำเป็นต้องเขียน
`shared_ptr<T>(new T(...), deleter)` ตรงๆ:

1. **ต้องการ custom deleter** — `make_shared` ไม่มีพารามิเตอร์ให้ระบุ deleter เอง (จะพูดถึง
   ในหัวข้อ 67.7)
2. **หน่วยความจำอาจไม่ถูกคืนทันทีที่ object หมดประโยชน์** — เพราะ object และ control block
   อยู่ก้อนเดียวกัน ถ้ามี `weak_ptr` เหลืออยู่แม้แต่ตัวเดียว (จะพูดถึงในหัวข้อ 67.6) หน่วยความจำ
   ทั้งก้อน (รวมพื้นที่ของ object ด้วย) จะยังไม่ถูกคืนจนกว่า `weak_ptr` ตัวสุดท้ายจะหมดไปด้วย
   ซึ่งถ้า object มีขนาดใหญ่มากและ `weak_ptr` อยู่ได้นาน อาจทำให้หน่วยความจำถูกกักไว้นานเกิน
   ความจำเป็น — กรณีนี้พบไม่บ่อยในโค้ดทั่วไป แต่เป็นสิ่งที่ควรรู้ไว้เมื่อทำงานกับระบบที่มี
   ข้อจำกัดด้านหน่วยความจำสูง

---

## 67.6 std::weak_ptr: แก้ปัญหา Circular Reference (Step 534)

นี่คือหัวข้อที่สำคัญที่สุดของ Part นี้ ย้อนกลับไปที่ **Part 55** อีกครั้ง — ตอนนั้นเราออกแบบ
ให้ `Member::borrowedMedia_` เป็น **raw pointer แบบไม่เป็นเจ้าของ (non-owning observer
pointer)** ไปยัง `Media` ที่ `Library` เป็นเจ้าของตัวจริงผ่าน `unique_ptr` และบอกไว้ว่า "สมาร์ท
พอยน์เตอร์แบบเจาะลึกทุกชนิด รวมถึง `weak_ptr` สำหรับกรณีแบบนี้โดยเฉพาะ จะเรียนเต็มรูปแบบใน
Part 67" ถึงเวลาแล้วที่จะเข้าใจว่าทำไม `weak_ptr` ถึงจำเป็นสำหรับสถานการณ์แบบนี้

### ปัญหา: Circular Reference ทำให้ shared_ptr Leak

`shared_ptr` แก้ปัญหาเรื่อง "ใครจะ `delete`" ได้ด้วยการนับจำนวนผู้ถือครอง แต่มันมีจุดอ่อน
สำคัญหนึ่งข้อ: **ถ้า object สองตัวถือ `shared_ptr` ชี้กลับไปหากันเป็นวงกลม (circular
reference) ตัวนับของทั้งคู่จะไม่มีวันลดลงเหลือ 0 เลย แม้จะไม่มีใครจากภายนอกอ้างถึงมันแล้วก็ตาม**
— ผลคือ memory leak ที่ `valgrind` ก็จับได้ (มันไม่ใช่ Undefined Behavior แต่เป็นการออกแบบที่
ผิดพลาด)

ลองดูตัวอย่าง `Parent` ที่ถือ `Child` และ `Child` ก็ถือ `Parent` กลับด้วย `shared_ptr` ทั้งคู่:

```cpp
// 07_circular_leak.cpp - สาธิต memory leak จาก shared_ptr วนกลับหากัน (circular reference)
#include <iostream>
#include <memory>
#include <string>

class Child; // forward declaration

class Parent {
public:
    explicit Parent(std::string name) : name_(std::move(name)) {
        std::cout << "  [สร้าง Parent] " << name_ << '\n';
    }
    ~Parent() {
        std::cout << "  [ทำลาย Parent] " << name_ << '\n';
    }
    void setChild(std::shared_ptr<Child> child) { child_ = std::move(child); }
    const std::string& name() const { return name_; }

private:
    std::string name_;
    std::shared_ptr<Child> child_; // Parent เป็นเจ้าของ Child
};

class Child {
public:
    explicit Child(std::string name) : name_(std::move(name)) {
        std::cout << "  [สร้าง Child] " << name_ << '\n';
    }
    ~Child() {
        std::cout << "  [ทำลาย Child] " << name_ << '\n';
    }
    void setParent(std::shared_ptr<Parent> parent) { parent_ = std::move(parent); }
    const std::string& name() const { return name_; }

private:
    std::string name_;
    std::shared_ptr<Parent> parent_; // Child ก็ถือ shared_ptr กลับไปหา Parent ด้วย! (ปัญหา)
};

int main() {
    std::cout << "-- สร้าง Parent และ Child แล้วให้ถือกันและกัน --\n";
    {
        std::shared_ptr<Parent> parent = std::make_shared<Parent>("บ้านเลขที่ 1");
        std::shared_ptr<Child> child = std::make_shared<Child>("ลูกคนโต");

        parent->setChild(child);   // parent -> child   (use_count ของ child = 2)
        child->setParent(parent);  // child  -> parent  (use_count ของ parent = 2)

        std::cout << "parent use_count = " << parent.use_count() << '\n'; // 2
        std::cout << "child  use_count = " << child.use_count() << '\n';  // 2
        std::cout << "-- ออกจาก scope: parent และ child (ตัวแปรท้องถิ่น) จะถูกทำลาย --\n";
    } // parent, child (ตัวแปรในสแต็ก) หมด scope ตรงนี้ -> use_count ของแต่ละฝั่งลดลงเหลือ 1
      // แต่ไม่มีทางลดเหลือ 0 เพราะยังมี "อีกฝั่ง" ถืออยู่เสมอ -> ไม่มี destructor ไหนถูกเรียกเลย!

    std::cout << "-- จบ main (สังเกตว่าไม่มีข้อความ [ทำลาย] เลยแม้แต่บรรทัดเดียว = LEAK) --\n";
    return 0;
}
```

ผลลัพธ์:

```
-- สร้าง Parent และ Child แล้วให้ถือกันและกัน --
  [สร้าง Parent] บ้านเลขที่ 1
  [สร้าง Child] ลูกคนโต
parent use_count = 2
child  use_count = 2
-- ออกจาก scope: parent และ child (ตัวแปรท้องถิ่น) จะถูกทำลาย --
-- จบ main (สังเกตว่าไม่มีข้อความ [ทำลาย] เลยแม้แต่บรรทัดเดียว = LEAK) --
```

**ไม่มีข้อความ `[ทำลาย]` ปรากฏเลย** — ยืนยันด้วย valgrind:

```bash
valgrind --leak-check=full ./07_circular_leak
```

```
==1762== 183 (64 direct, 119 indirect) bytes in 1 blocks are definitely lost in loss record 4 of 4
==1762==    at 0x4846FA3: operator new(unsigned long)
==1762==    by ... std::make_shared<Parent, char const (&) [33]>(char const (&) [33])
==1762==    by 0x10A473: main
```

valgrind ยืนยันชัดเจน — ทั้ง `Parent` และ `Child` รั่วไหลทั้งคู่ (183 bytes รวมทั้งสอง object
และ control block ของมัน) สาเหตุคือหลัง `parent` และ `child` (ตัวแปรใน `main`) ออกจาก scope
`use_count` ของแต่ละฝั่งลดลงจาก 2 เหลือ 1 เท่านั้น — เพราะยังมี **"อีกฝั่ง" ถืออยู่เสมอ**
(`Parent::child_` ยังถือ `Child` อยู่, `Child::parent_` ยังถือ `Parent` อยู่) ทั้งคู่เลย
**ไม่มีทางที่ตัวนับจะลดลงเหลือ 0 ได้เลย** ต่อให้ไม่มีใครจากโลกภายนอก (นอก object ทั้งสอง)
อ้างถึงมันแล้วก็ตาม — นี่คือกับดักที่อันตรายที่สุดของ `shared_ptr` เพราะโค้ดคอมไพล์ผ่านสนิท
รันไม่ crash แต่รั่วไหลไปเรื่อยๆ อย่างเงียบๆ

### ทางแก้: std::weak_ptr

`std::weak_ptr<T>` คือ smart pointer ที่ **"สังเกตการณ์" object ที่ shared_ptr ถืออยู่ โดยไม่
นับเป็นเจ้าของ** (ไม่เพิ่ม `use_count()` เลย) มันตอบคำถามได้ว่า "object นี้ยังมีชีวิตอยู่ไหม"
แต่ไม่มีสิทธิ์ตัดสินใจว่า object ควรมีชีวิตอยู่ต่อหรือไม่

หลักการแก้ปัญหา circular reference คือ: ต้องมี**ฝั่งหนึ่งเท่านั้น**ที่เป็นเจ้าของจริง (ใช้
`shared_ptr`) ส่วนอีกฝั่งที่แค่ "ต้องการอ้างอิงกลับ" ให้ใช้ `weak_ptr` แทน ในตัวอย่าง
Parent-Child ความสัมพันธ์ที่เป็นธรรมชาติคือ `Parent` เป็นเจ้าของ `Child` (parent ตายแล้ว
child ควรตายตาม) แต่ `Child` ไม่ควรเป็นเจ้าของ `Parent` กลับ — มันแค่ต้องการ "รู้จัก" parent
ของมันเพื่อไต่กลับขึ้นไปได้เท่านั้น:

```cpp
// 08_weak_fix.cpp - แก้ circular reference ด้วย weak_ptr
#include <iostream>
#include <memory>
#include <string>

class Child;

class Parent {
public:
    explicit Parent(std::string name) : name_(std::move(name)) {
        std::cout << "  [สร้าง Parent] " << name_ << '\n';
    }
    ~Parent() {
        std::cout << "  [ทำลาย Parent] " << name_ << '\n';
    }
    void setChild(std::shared_ptr<Child> child) { child_ = std::move(child); }
    const std::string& name() const { return name_; }

private:
    std::string name_;
    std::shared_ptr<Child> child_; // Parent ยังคงเป็น "เจ้าของ" Child ตัวจริง (strong)
};

class Child {
public:
    explicit Child(std::string name) : name_(std::move(name)) {
        std::cout << "  [สร้าง Child] " << name_ << '\n';
    }
    ~Child() {
        std::cout << "  [ทำลาย Child] " << name_ << '\n';
    }
    // Child แค่ "รู้จัก" Parent เพื่อไต่กลับขึ้นไปดูข้อมูลได้ ไม่ได้เป็นเจ้าของ -> ใช้ weak_ptr
    void setParent(std::shared_ptr<Parent> parent) { parent_ = parent; }
    const std::string& name() const { return name_; }

    void greetParent() const {
        // ต้อง lock() ก่อนใช้งานเสมอ เพราะ Parent อาจถูกทำลายไปแล้วก็ได้
        if (std::shared_ptr<Parent> p = parent_.lock()) {
            std::cout << "  " << name_ << " ทักทาย parent: " << p->name() << '\n';
        } else {
            std::cout << "  " << name_ << " ไม่มี parent อยู่แล้ว (ถูกทำลายไปแล้ว)\n";
        }
    }

private:
    std::string name_;
    std::weak_ptr<Parent> parent_; // ไม่นับ reference count -> ตัด cycle ได้
};

int main() {
    std::cout << "-- สร้าง Parent/Child โดย Child ถือ Parent ผ่าน weak_ptr --\n";
    std::weak_ptr<Parent> observer; // ใช้เฝ้าดูจากภายนอกด้วย
    {
        std::shared_ptr<Parent> parent = std::make_shared<Parent>("บ้านเลขที่ 1");
        std::shared_ptr<Child> child = std::make_shared<Child>("ลูกคนโต");

        parent->setChild(child);
        child->setParent(parent); // เก็บเป็น weak_ptr ภายใน ไม่เพิ่ม use_count ของ parent

        std::cout << "parent use_count = " << parent.use_count() << '\n'; // 1 (ไม่ใช่ 2!)
        std::cout << "child  use_count = " << child.use_count() << '\n';  // 2 (parent ยังถือ child)

        child->greetParent();
        observer = parent;
        std::cout << "-- ออกจาก scope --\n";
    } // parent, child ถูกทำลายตามลำดับที่ถูกต้องทันทีที่ scope จบ

    std::cout << "-- หลังออกจาก scope: observer.expired() = " << std::boolalpha
              << observer.expired() << '\n';
    std::cout << "-- จบ main --\n";
    return 0;
}
```

ผลลัพธ์:

```
-- สร้าง Parent/Child โดย Child ถือ Parent ผ่าน weak_ptr --
  [สร้าง Parent] บ้านเลขที่ 1
  [สร้าง Child] ลูกคนโต
parent use_count = 1
child  use_count = 2
  ลูกคนโต ทักทาย parent: บ้านเลขที่ 1
-- ออกจาก scope --
  [ทำลาย Parent] บ้านเลขที่ 1
  [ทำลาย Child] ลูกคนโต
-- หลังออกจาก scope: observer.expired() = true
-- จบ main --
```

คราวนี้ทั้ง `Parent` และ `Child` ถูกทำลายเรียบร้อย (ยืนยันด้วย valgrind แล้วว่า **ไม่มี leak
เลย**) สังเกตรายละเอียดสองจุดที่สำคัญ:

1. **`parent.use_count()` เป็น 1 ไม่ใช่ 2** — เพราะ `Child::setParent` เก็บเป็น `weak_ptr` ซึ่ง
   ไม่เพิ่มตัวนับ ต่างจากตัวอย่างที่ leak ก่อนหน้าที่เป็น 2
2. **ลำดับการทำลาย**: `~Parent()` ถูกเรียกก่อน (เพราะ `parent` ในสแต็กหมด scope ทำให้
   `use_count` ของ `Parent` ลดเหลือ 0) แล้ว**ระหว่าง** ที่ `~Parent()` กำลังทำลายสมาชิกของมัน
   สมาชิก `child_` (ซึ่งเป็น `shared_ptr<Child>`) จึงถูกทำลายตามไปด้วย ทำให้เห็น
   `[ทำลาย Child]` ตามหลังทันที — นี่คือพฤติกรรมที่ถูกต้องของ **destructor ที่ทำลาย
   member ตามลำดับย้อนกลับของการประกาศ** หลังจาก destructor body ของตัวเองรันเสร็จ

`lock()` คือหัวใจของการใช้ `weak_ptr` อย่างปลอดภัย — มันพยายาม "เลื่อนขั้น" `weak_ptr` กลับเป็น
`shared_ptr` ชั่วคราว ถ้า object ยังไม่ถูกทำลาย จะได้ `shared_ptr` ที่ใช้งานได้ปกติ (และเพิ่ม
`use_count` ชั่วคราวระหว่างที่ใช้งานอยู่ ป้องกันไม่ให้ object ถูกทำลายกลางคันขณะกำลังใช้)
แต่ถ้า object ถูกทำลายไปแล้ว จะได้ `shared_ptr` ที่เป็น `nullptr` แทน — `expired()` คือทางลัด
ในการเช็คว่า object ยังอยู่ไหมโดยไม่ต้องสร้าง `shared_ptr` ชั่วคราว แต่ **ไม่ควรใช้ `expired()`
แล้วค่อย `lock()` แยกกันสองขั้นตอน** เพราะระหว่างสองบรรทัดนั้น object อาจถูกทำลายพอดี (โดยเฉพาะ
ในโปรแกรมที่มีหลาย thread) — ให้ `lock()` แล้วเช็คผลลัพธ์ในขั้นตอนเดียวเสมอแบบในตัวอย่างข้างต้น

> **เชื่อมโยงกับ Part 55**: ด้วยความรู้นี้ ถ้าจะปรับปรุงระบบ Library ให้ `Member`
> ต้องการ "รู้ว่ากำลังยืม `Media` ตัวไหนอยู่" โดยไม่ต้องเป็นเจ้าของ (ซึ่งเดิมใช้ raw pointer
> ที่เสี่ยงต่อ dangling ถ้า `Media` ถูกลบออกจากระบบไปแล้ว) เราสามารถเปลี่ยน
> `Member::borrowedMedia_` จาก `std::vector<Media*>` เป็น `std::vector<std::weak_ptr<Media>>`
> แทนได้ — วิธีนี้ปลอดภัยกว่า raw pointer มาก เพราะก่อนใช้งานทุกครั้งต้อง `lock()` ก่อนเสมอ
> ถ้า `Media` ตัวนั้นถูกลบออกจากระบบไปแล้วจริงๆ `lock()` จะคืน `nullptr` ให้แทนที่จะเป็น
> dangling pointer ที่ใช้แล้ว crash หรือแย่กว่านั้นคือทำงาน "ดูเหมือนถูก" ทั้งที่จริงๆ ไม่ปลอดภัย

---

## 67.7 Custom Deleter (Step 535)

โดยปกติ smart pointer จะเรียก `delete` (หรือ `delete[]` สำหรับ array) ให้อัตโนมัติ แต่บาง
ทรัพยากรไม่ได้ถูกคืนด้วย `delete` เช่น `FILE*` ต้องคืนด้วย `fclose()`, array ที่จองด้วย `new T[]`
โดยตรงต้องคืนด้วย `delete[]`, หรือทรัพยากรของระบบปฏิบัติการอย่าง `pthread_mutex_t` ต้องคืนด้วย
`pthread_mutex_destroy()` — ทั้งหมดนี้ทำได้ด้วยการระบุ **custom deleter** ให้ smart pointer

Custom deleter สามารถเป็นได้ทั้ง **functor (struct ที่มี `operator()`)**, **lambda expression**,
หรือ **function pointer** โดยสำหรับ `unique_ptr` ต้องระบุชนิดของ deleter เป็น **template
parameter ตัวที่สอง** ด้วย (เพราะ deleter เป็นส่วนหนึ่งของ type) ส่วน `shared_ptr` **ไม่ต้อง**
ระบุ type ของ deleter ใน template parameter เลย (ส่งผ่าน constructor ตรงๆ ได้) เพราะ
`shared_ptr` เก็บ deleter ไว้ใน control block แบบ type-erased อยู่แล้ว

```cpp
// 09_custom_deleter.cpp
#include <cstdio>
#include <iostream>
#include <memory>

struct FileCloser {
    void operator()(std::FILE* fp) const {
        if (fp) {
            std::cout << "  [ปิดไฟล์ผ่าน custom deleter]\n";
            std::fclose(fp);
        }
    }
};

int main() {
    // 1) unique_ptr + custom deleter แบบ functor สำหรับ FILE*
    {
        std::unique_ptr<std::FILE, FileCloser> file(std::fopen("/tmp/smart_ptr_demo.txt", "w"));
        if (file) {
            std::fputs("hello from unique_ptr custom deleter\n", file.get());
            std::cout << "  เขียนไฟล์สำเร็จ\n";
        }
    } // ไฟล์ถูกปิดอัตโนมัติผ่าน FileCloser::operator() ตรงนี้

    // 2) unique_ptr + custom deleter แบบ lambda
    {
        auto deleter = [](std::FILE* fp) {
            if (fp) {
                std::cout << "  [ปิดไฟล์ผ่าน lambda deleter]\n";
                std::fclose(fp);
            }
        };
        std::unique_ptr<std::FILE, decltype(deleter)> file(
            std::fopen("/tmp/smart_ptr_demo.txt", "r"), deleter);
        if (file) {
            char buf[64] = {0};
            if (std::fgets(buf, sizeof(buf), file.get())) {
                std::cout << "  อ่านได้: " << buf;
            }
        }
    }

    // 3) shared_ptr + custom deleter สำหรับ array (สมัย C++17 shared_ptr<T[]> ยังไม่มี ต้องทำเอง)
    {
        std::shared_ptr<int> arr(new int[5]{1, 2, 3, 4, 5}, [](int* p) {
            std::cout << "  [คืน array ผ่าน custom deleter]\n";
            delete[] p;
        });
        for (int i = 0; i < 5; ++i) std::cout << arr.get()[i] << ' ';
        std::cout << '\n';
    }

    std::remove("/tmp/smart_ptr_demo.txt");
    return 0;
}
```

ผลลัพธ์:

```
  เขียนไฟล์สำเร็จ
  [ปิดไฟล์ผ่าน custom deleter]
  อ่านได้: hello from unique_ptr custom deleter
  [ปิดไฟล์ผ่าน lambda deleter]
1 2 3 4 5
  [คืน array ผ่าน custom deleter]
```

> **หมายเหตุ**: ตั้งแต่ C++17 มี `std::shared_ptr<T[]>` ให้ใช้ได้แล้วโดยไม่ต้องเขียน custom
> deleter เอง (มันจะเรียก `delete[]` ให้อัตโนมัติ เหมือน `unique_ptr<T[]>`) ตัวอย่างข้างต้น
> เขียนแบบ custom deleter เพื่อสาธิตหลักการที่นำไปประยุกต์กับทรัพยากรชนิดอื่นที่ไม่ใช่ memory
> ได้ด้วย (เช่น `FILE*`) ซึ่งไม่มี built-in support ให้

### ผลกระทบของ custom deleter ต่อขนาดของ unique_ptr

จุดที่ต่างกันชัดเจนระหว่าง `unique_ptr` กับ `shared_ptr` คือ deleter ของ `unique_ptr` เป็นส่วน
หนึ่งของ**ชนิดข้อมูล (type)** ทำให้ขนาด (`sizeof`) ของมันอาจเปลี่ยนไปตามชนิดของ deleter ที่ใช้
ในขณะที่ `shared_ptr` มีขนาดคงที่เสมอไม่ว่าจะใช้ deleter แบบไหน:

```cpp
// 10_sizeof.cpp
#include <memory>
#include <iostream>

int main() {
    auto lambdaDeleter = [](int* p) { delete p; };

    std::cout << "sizeof(int*)                                     = " << sizeof(int*) << '\n';
    std::cout << "sizeof(unique_ptr<int>)                          = " << sizeof(std::unique_ptr<int>) << '\n';
    std::cout << "sizeof(unique_ptr<int, decltype(lambdaDeleter)>) = "
              << sizeof(std::unique_ptr<int, decltype(lambdaDeleter)>) << '\n';
    std::cout << "sizeof(unique_ptr<int, void(*)(int*)>)           = "
              << sizeof(std::unique_ptr<int, void (*)(int*)>) << '\n';
    std::cout << "sizeof(shared_ptr<int>)                          = " << sizeof(std::shared_ptr<int>) << '\n';
    std::cout << "sizeof(weak_ptr<int>)                            = " << sizeof(std::weak_ptr<int>) << '\n';
    return 0;
}
```

ผลลัพธ์ (บนระบบ 64-bit):

```
sizeof(int*)                                     = 8
sizeof(unique_ptr<int>)                          = 8
sizeof(unique_ptr<int, decltype(lambdaDeleter)>) = 8
sizeof(unique_ptr<int, void(*)(int*)>)           = 16
sizeof(shared_ptr<int>)                          = 16
sizeof(weak_ptr<int>)                            = 16
```

สังเกตว่า `unique_ptr` ที่ใช้ lambda deleter แบบไม่ capture อะไรเลย (**stateless**) ยังคงมีขนาด
เท่ากับ raw pointer เดิม (8 bytes) เพราะ compiler ทำ **Empty Base Optimization** ให้ แต่ถ้าใช้
**function pointer** เป็น deleter ขนาดจะเพิ่มเป็น 16 bytes ทันที (เพราะต้องเก็บ address ของ
ฟังก์ชันนั้นไว้ด้วยจริงๆ) ส่วน `shared_ptr` และ `weak_ptr` มีขนาด 16 bytes เสมอ (raw pointer +
pointer ไปยัง control block) ไม่ว่าจะใช้ deleter แบบไหนก็ตาม เพราะ deleter ถูกเก็บใน control
block ที่อยู่บน heap แยกออกไปแล้ว — นี่คือรายละเอียดเล็กๆ ที่ช่วยอธิบายว่าทำไมบางครั้งการเลือก
ชนิดของ deleter จึงมีผลต่อประสิทธิภาพและขนาดของโครงสร้างข้อมูลโดยรวม

---

## 67.8 กฎทองในการเลือกใช้ Smart Pointer แต่ละแบบ (Step 536)

หลังจากเรียนรู้ smart pointer ทั้งสามชนิดแล้ว คำถามที่สำคัญที่สุดคือ "แล้วเมื่อไหร่ควรใช้ตัว
ไหน" ต่อไปนี้คือตารางตัดสินใจที่ควรยึดเป็นกฎทองตลอดหลักสูตรที่เหลือ (และตลอดการทำงานจริง):

### ตารางตัดสินใจ: unique_ptr vs shared_ptr vs weak_ptr vs raw pointer

| สถานการณ์ | ควรใช้ | เหตุผล |
|---|---|---|
| Object มีเจ้าของเพียงหนึ่งเดียวชัดเจน (เช่น `Library` เป็นเจ้าของ `Media` แต่ละชิ้น) | `std::unique_ptr` | สื่อความหมาย ownership ชัดเจนที่สุด, overhead ต่ำสุด (เท่า raw pointer) |
| Object ต้องถูกใช้ร่วมกันโดยหลายส่วนของโปรแกรม โดยไม่รู้ว่าใครจะเป็นเจ้าของคนสุดท้าย | `std::shared_ptr` | reference counting จัดการช่วงชีวิตให้อัตโนมัติ |
| ต้องการ "อ้างอิงกลับ" ไปยัง object ที่ `shared_ptr` อีกฝั่งเป็นเจ้าของอยู่ (ป้องกัน circular reference) | `std::weak_ptr` | ไม่เพิ่ม reference count, ตรวจสอบว่า object ยังอยู่ไหมได้ด้วย `lock()` |
| ต้องการ cache หรือ observer ที่ "ดูเฉยๆ" ไม่อยากมีผลต่อช่วงชีวิตของ object | `std::weak_ptr` | เหมาะกับ event listener, observer pattern ที่จะเจาะลึกใน Part 97-98 |
| ฟังก์ชันแค่ต้องการ "ยืมดู" object ชั่วคราว ไม่ต้องการเป็นเจ้าของหรือขยายอายุมันเลย | **raw pointer หรือ reference ธรรมดา** (ไม่ใช่ smart pointer) | การส่ง `shared_ptr` เข้าฟังก์ชันที่แค่ "ยืมดู" โดยไม่จำเป็นจะเพิ่ม/ลด reference count โดยใช่เหตุ เสียประสิทธิภาพฟรีๆ |
| ทำงานกับ C API เก่า หรือ array แบบ fixed-size ที่ไม่ต้องจัดการ ownership เลย | raw pointer / `std::array` / `std::vector` | ไม่มี dynamic ownership ให้ต้องจัดการ |

### กฎทองสรุปสั้นๆ

> **1. เริ่มต้นด้วย `unique_ptr` เสมอ** — มันคือ default ที่ถูกต้องที่สุดสำหรับเกือบทุก
> สถานการณ์ที่ต้องมี dynamic ownership เพราะ overhead เป็นศูนย์เมื่อเทียบกับ raw pointer
> และสื่อความหมายชัดเจนว่า "มีเจ้าของเดียว"
>
> **2. เปลี่ยนเป็น `shared_ptr` เฉพาะเมื่อพิสูจน์ได้จริงๆ ว่าต้องมีเจ้าของร่วมหลายฝ่าย** —
> อย่าใช้ `shared_ptr` เป็น default เพราะ "เผื่อไว้ปลอดภัยกว่า" การมี reference counting
> ทุกจุดมี overhead ทั้งด้านหน่วยความจำ (control block) และเวลา (atomic increment/decrement
> ที่ปลอดภัยกับหลาย thread เสมอ แม้โปรแกรมจะเป็น single-thread ก็ตาม)
>
> **3. ใช้ `weak_ptr` ทุกครั้งที่มีความสัมพันธ์แบบ "อ้างอิงกลับ" หรือ "ไม่เป็นเจ้าของ"
> ระหว่าง object ที่ใช้ `shared_ptr`** — เพื่อป้องกัน circular reference ตั้งแต่ตอนออกแบบ
> ไม่ต้องรอให้ valgrind เจอ leak ก่อนแล้วค่อยแก้
>
> **4. ฟังก์ชันที่แค่ "ยืมใช้" object ชั่วคราว ไม่ต้องรับ smart pointer เป็นพารามิเตอร์เลย**
> — ให้รับ raw pointer หรือ reference ธรรมดาแทน (เช่น `void process(const Media& m)` หรือ
> `void process(Media* m)`) แล้วให้ผู้เรียกเป็นคนตัดสินใจเรื่อง ownership เอง วิธีนี้ทั้ง
> เร็วกว่าและยืดหยุ่นกว่า (ฟังก์ชันรับได้ทั้ง object ที่มาจาก `unique_ptr`, `shared_ptr`,
> หรือแม้แต่ตัวแปรบน stack ธรรมดา)
>
> **5. ห้ามเขียน `new`/`delete` ตรงๆ ในโค้ดระดับ business logic อีกต่อไป** — ใช้
> `make_unique`/`make_shared` เสมอ ยกเว้นกรณีต้องใช้ custom deleter เท่านั้น

---

## ข้อผิดพลาดที่พบบ่อย (Common Pitfalls)

1. **ลืม `std::move` ตอนย้าย ownership ของ `unique_ptr`** — เขียน
   `std::unique_ptr<T> p2 = p1;` ตรงๆ จะเจอ compile error ทันที (copy constructor ถูกลบ) ซึ่ง
   ถือว่า "โชคดี" เพราะ compiler จับได้เอง แต่บางคนแก้ปัญหาผิดทางด้วยการเปลี่ยนไปใช้
   `shared_ptr` ทั้งที่ควรจะแค่เติม `std::move(p1)` ให้ถูกต้อง
2. **ใช้ `shared_ptr` เป็น default โดยไม่จำเป็น** — ทำให้โปรแกรมช้าลงจาก atomic reference
   counting ที่ไม่จำเป็น และซ่อนบั๊กเรื่อง ownership ที่ไม่ชัดเจนไว้ (เพราะ `shared_ptr` copy
   ได้ง่ายเกินไป ทำให้ไม่มีใครรู้ว่า "ใครควรเป็นเจ้าของตัวจริง")
3. **circular reference ระหว่าง `shared_ptr` สองฝั่ง** — โดยเฉพาะในโครงสร้างข้อมูลที่มีการ
   อ้างอิงกลับตามธรรมชาติ เช่น parent-child, doubly linked list, observer pattern ต้องคอย
   ถามตัวเองเสมอว่า "ฝั่งไหนควรเป็นเจ้าของจริง ฝั่งไหนแค่สังเกตการณ์" แล้วใช้ `weak_ptr` กับ
   ฝั่งที่สังเกตการณ์
4. **เรียก `->` หรือ `*` บน `weak_ptr` โดยตรง** — `weak_ptr` **ไม่มี** `operator->` หรือ
   `operator*` ให้ใช้ (compile error ทันที) เพราะมันอาจ "หมดอายุ" ได้ทุกเมื่อ ต้อง `lock()`
   ก่อนเสมอแล้วเช็คผลลัพธ์
5. **เขียน `new` ปนอยู่ในอาร์กิวเมนต์ของฟังก์ชันที่มีอาร์กิวเมนต์อื่นที่อาจ throw ได้** เช่น
   `f(std::shared_ptr<T>(new T()), mayThrow())` — ให้แยกสร้าง smart pointer เป็นตัวแปรก่อน
   เสมอ หรือใช้ `make_shared`/`make_unique` ซึ่งบังคับให้เขียนถูกโดยธรรมชาติอยู่แล้ว
6. **ใช้ `get()` แล้วเผลอ `delete` raw pointer ที่ได้มาเอง** — `get()` แค่ "ยืมดู" ไม่ได้โอน
   ownership การ `delete` มันตรงๆ จะกลายเป็น double free ทันทีเมื่อ smart pointer ตัวเดิม
   หมด scope แล้วพยายาม `delete` ซ้ำอีกครั้ง
7. **ผสม `unique_ptr<T[]>` กับ `unique_ptr<T>` สลับกัน** — ถ้าจองด้วย `new T[]` ต้องใช้
   `unique_ptr<T[]>` เท่านั้น (ไม่ใช่ `unique_ptr<T>`) มิเช่นนั้นตอนทำลายจะเรียก `delete`
   (ตัวเดียว) แทนที่จะเป็น `delete[]` ซึ่งเป็น Undefined Behavior

---

## แบบฝึกหัดท้ายบท

1. เขียนฟังก์ชัน `archiveMedia(Catalog& from, Catalog& to, std::size_t index)` ที่ย้าย
   `unique_ptr<Media>` จาก vector หนึ่งไปอีก vector หนึ่งตาม index ที่กำหนด **โดยไม่ copy
   object เลยแม้แต่ครั้งเดียว** (ใช้ `std::move`)
2. เขียนโปรแกรมที่สร้าง `shared_ptr<Media>` หนึ่งตัว แล้วเก็บสำเนาของมันไว้ใน `std::vector`
   สามที่ต่างกัน พิมพ์ `use_count()` หลังจากเพิ่มแต่ละครั้ง และหลังจากลบออกจากแต่ละ vector
   ทีละที่ (สังเกตว่าตัวเลขเปลี่ยนถูกต้องตามที่คาดหวังหรือไม่)
3. กำหนดโค้ด `Parent`/`Child` ที่มี circular reference ให้ (เหมือนตัวอย่าง 07 ในบทเรียน) ให้
   แก้ไขโดยเปลี่ยนความสัมพันธ์ฝั่งใดฝั่งหนึ่งเป็น `weak_ptr` แล้วพิสูจน์ด้วย valgrind ว่า
   ไม่มี memory leak เหลืออยู่อีกต่อไป
4. เขียน custom deleter สำหรับ `pthread_mutex_t` (ทบทวนจาก Module C, Part 31-32) โดยใช้
   `std::unique_ptr<pthread_mutex_t, Deleter>` ที่เรียก `pthread_mutex_destroy()` อัตโนมัติ
   เมื่อหมด scope
5. เปรียบเทียบ `sizeof` ของ `unique_ptr<int>` เมื่อใช้ deleter สามแบบ: default deleter,
   lambda ที่ไม่ capture ตัวแปรใดๆ, และ function pointer ธรรมดา อธิบายด้วยคำพูดตัวเองว่าทำไม
   ผลลัพธ์ถึงต่างกัน
6. ให้จัดหมวดหมู่สถานการณ์ต่อไปนี้ว่าควรใช้ smart pointer ชนิดใด (`unique_ptr`/`shared_ptr`/
   `weak_ptr`/raw pointer) พร้อมให้เหตุผลสั้นๆ: (ก) Node ของ Binary Tree ที่แต่ละ node มี
   เจ้าของเดียวคือ parent ของมัน (ข) Cache ที่เก็บ observer ไปยัง object ที่อาจถูกลบไปแล้ว
   เมื่อไหร่ก็ได้ (ค) ฟังก์ชัน `printReport(const Report& r)` ที่แค่อ่านค่าไปแสดงผลอย่างเดียว

### แนวทางเฉลยข้อ 1

```cpp
// ex1_transfer.cpp
#include <iostream>
#include <memory>
#include <string>
#include <vector>

class Media {
public:
    explicit Media(std::string title) : title_(std::move(title)) {}
    const std::string& title() const { return title_; }
private:
    std::string title_;
};

using Catalog = std::vector<std::unique_ptr<Media>>;

// ย้าย Media จาก "from" ไปยัง "to" ตาม index ที่กำหนด โดยไม่ copy object เลยแม้แต่ครั้งเดียว
bool archiveMedia(Catalog& from, Catalog& to, std::size_t index) {
    if (index >= from.size() || !from[index]) return false;

    to.push_back(std::move(from[index])); // ย้าย ownership ออกจาก from
    from.erase(from.begin() + static_cast<Catalog::difference_type>(index));
    return true;
}

void printCatalog(const std::string& name, const Catalog& c) {
    std::cout << name << " (" << c.size() << " รายการ): ";
    for (const auto& m : c) std::cout << "[" << m->title() << "] ";
    std::cout << '\n';
}

int main() {
    Catalog active;
    active.push_back(std::make_unique<Media>("Clean Code"));
    active.push_back(std::make_unique<Media>("The Mythical Man-Month"));
    active.push_back(std::make_unique<Media>("Refactoring"));

    Catalog archived;

    printCatalog("active ก่อน", active);
    archiveMedia(active, archived, 1); // ย้าย "The Mythical Man-Month" ไป archived
    printCatalog("active หลัง", active);
    printCatalog("archived หลัง", archived);

    return 0;
}
```

ผลลัพธ์:

```
active ก่อน (3 รายการ): [Clean Code] [The Mythical Man-Month] [Refactoring]
active หลัง (2 รายการ): [Clean Code] [Refactoring]
archived หลัง (1 รายการ): [The Mythical Man-Month]
```

ตรวจสอบด้วย valgrind แล้วไม่มี leak — `std::move(from[index])` ย้าย ownership ของ
`unique_ptr<Media>` ออกจาก slot เดิมโดยตรง (slot เดิมกลายเป็น `nullptr`) ก่อนจะถูก `erase()`
ออกจาก vector ไม่มีการสร้าง `Media` ตัวใหม่หรือ copy ใดๆ เกิดขึ้นเลยตลอดกระบวนการย้าย

### แนวทางเฉลยข้อ 3

การแก้ต้องระบุก่อนว่า **ฝั่งไหนควรเป็นเจ้าของจริง** — ในความสัมพันธ์ Parent-Child ตามธรรมชาติ
แล้ว `Parent` ควรเป็นเจ้าของ `Child` (parent หายไป child ก็ควรหายไปด้วย) ส่วน `Child` แค่
ต้องการ "รู้จัก" parent ของมันเพื่อไต่กลับขึ้นไปได้ จึงเปลี่ยน `Child::parent_` จาก
`std::shared_ptr<Parent>` เป็น `std::weak_ptr<Parent>`:

```cpp
// ex3_fix_cycle.cpp (โครงสร้างเดียวกับ 08_weak_fix.cpp ในบทเรียน)
#include <iostream>
#include <memory>
#include <string>

class Child;

class Parent {
public:
    explicit Parent(std::string name) : name_(std::move(name)) {
        std::cout << "  [สร้าง Parent] " << name_ << '\n';
    }
    ~Parent() { std::cout << "  [ทำลาย Parent] " << name_ << '\n'; }
    void setChild(std::shared_ptr<Child> child) { child_ = std::move(child); }
    const std::string& name() const { return name_; }
private:
    std::string name_;
    std::shared_ptr<Child> child_; // ฝั่งเจ้าของจริง: ยังเป็น shared_ptr เหมือนเดิม
};

class Child {
public:
    explicit Child(std::string name) : name_(std::move(name)) {
        std::cout << "  [สร้าง Child] " << name_ << '\n';
    }
    ~Child() { std::cout << "  [ทำลาย Child] " << name_ << '\n'; }
    void setParent(std::shared_ptr<Parent> parent) { parent_ = parent; } // เก็บเป็น weak_ptr
    const std::string& name() const { return name_; }
private:
    std::string name_;
    std::weak_ptr<Parent> parent_; // เปลี่ยนจาก shared_ptr -> weak_ptr เพื่อตัด cycle
};

int main() {
    std::shared_ptr<Parent> parent = std::make_shared<Parent>("บ้านเลขที่ 1");
    std::shared_ptr<Child> child = std::make_shared<Child>("ลูกคนโต");
    parent->setChild(child);
    child->setParent(parent);
    std::cout << "parent use_count = " << parent.use_count() << " (ต้องเป็น 1)\n";
    return 0;
}
```

```
  [สร้าง Parent] บ้านเลขที่ 1
  [สร้าง Child] ลูกคนโต
parent use_count = 1 (ต้องเป็น 1)
  [ทำลาย Parent] บ้านเลขที่ 1
  [ทำลาย Child] ลูกคนโต
```

รัน `valgrind --leak-check=full ./ex3_fix_cycle` แล้วจะเห็น
`All heap blocks were freed -- no leaks are possible` ยืนยันว่าการเปลี่ยนไปใช้ `weak_ptr`
ตัดวงจร circular reference ได้สำเร็จ

### แนวทางเฉลยข้อ 4

```cpp
// ex4_mutex_deleter.cpp
#include <iostream>
#include <memory>
#include <pthread.h>

struct MutexDeleter {
    void operator()(pthread_mutex_t* m) const {
        std::cout << "  [pthread_mutex_destroy ถูกเรียกอัตโนมัติ]\n";
        pthread_mutex_destroy(m);
        delete m;
    }
};

using MutexPtr = std::unique_ptr<pthread_mutex_t, MutexDeleter>;

MutexPtr makeMutex() {
    MutexPtr m(new pthread_mutex_t);
    pthread_mutex_init(m.get(), nullptr);
    return m;
}

int main() {
    MutexPtr mtx = makeMutex();

    pthread_mutex_lock(mtx.get());
    std::cout << "  อยู่ใน critical section\n";
    pthread_mutex_unlock(mtx.get());

    std::cout << "-- จบ main: mutex จะถูก destroy อัตโนมัติผ่าน MutexDeleter --\n";
    return 0;
}
```

คอมไพล์ (ต้อง link `-lpthread`):

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 ex4_mutex_deleter.cpp -o ex4 -lpthread
./ex4
```

```
  อยู่ใน critical section
-- จบ main: mutex จะถูก destroy อัตโนมัติผ่าน MutexDeleter --
  [pthread_mutex_destroy ถูกเรียกอัตโนมัติ]
```

ตัวอย่างนี้เป็นการปูทางไปสู่ **Part 68** โดยตรง — การห่อทรัพยากรของระบบปฏิบัติการ (ที่ไม่มี
smart pointer มาตรฐานรองรับโดยตรง) ด้วย custom deleter คือหนึ่งในรูปแบบของ RAII pattern ที่
จะเจาะลึกอย่างเป็นระบบใน Part ถัดไป

---

## สรุปท้ายบท

ใน Part นี้เราได้:

- เข้าใจปัญหาที่แท้จริงสามข้อของ raw pointer + `new`/`delete` แบบ manual (memory leak, double
  free, dangling pointer) และพิสูจน์ด้วยโค้ดจริงร่วมกับ valgrind
- ใช้ `std::unique_ptr` จัดการ exclusive ownership พร้อมเข้าใจว่าทำไมมัน copy ไม่ได้ ต้อง
  `std::move` เท่านั้น และใช้ `std::make_unique` เป็นสำนวนมาตรฐานในการสร้างมัน
- ใช้ `std::shared_ptr` จัดการ shared ownership ผ่านกลไก reference counting และรู้จัก
  `use_count()` เพื่อทำความเข้าใจสถานะของ object ที่ถูกใช้ร่วมกัน
- เข้าใจว่าทำไม `std::make_shared` มีประสิทธิภาพดีกว่า `shared_ptr(new T)` (จัดสรรหน่วยความจำ
  ครั้งเดียวแทนสองครั้ง) และรู้ข้อยกเว้นที่ยังต้องใช้ `shared_ptr(new T)` แทน
- แก้ปัญหา **circular reference** ของ `shared_ptr` ด้วย `std::weak_ptr` ผ่านตัวอย่าง
  Parent-Child ที่พิสูจน์ leak จริงด้วย valgrind แล้วแก้ให้หายขาด — ปิดช่องโหว่ที่ค้างไว้ตั้งแต่
  Part 55
- เขียน custom deleter สำหรับทรัพยากรที่ไม่ใช่ memory ธรรมดา เช่น `FILE*`, array, และปูทางไปสู่
  การห่อทรัพยากรของ OS อย่าง `pthread_mutex_t`
- สรุปกฎทองในการเลือกใช้ smart pointer แต่ละชนิดให้เหมาะสมกับสถานการณ์ ซึ่งจะเป็นหลักการที่ใช้
  ตลอดหลักสูตรที่เหลือ

ใน **Part 68** เราจะขยายแนวคิด RAII ที่เป็นรากฐานของ smart pointer ทั้งหมดใน Part นี้ให้
กว้างขึ้นไปอีก ไปสู่ **RAII และ Resource Management Pattern ระดับ Production** — ครอบคลุมทั้ง
การเขียน RAII wrapper class ของตัวเองสำหรับทรัพยากรที่ไม่มี smart pointer ให้, `std::lock_guard`/
`std::unique_lock` สำหรับ mutex, แนวคิด **Rule of Zero**, และระดับของ **Exception Safety
Guarantee**

**ต่อไป:** [Part 68 — RAII และ Resource Management Pattern ระดับ Production](./part-068-raii-patterns.md)
