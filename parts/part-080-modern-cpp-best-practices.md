# Part 80: Modern C++ Best Practice และ C++ Core Guidelines (Step 633–640)

> Module F — Modern C++ (C++11 ถึง C++23) | Part 80 จาก 125
> Step ที่ครอบคลุมใน Part นี้: Step 633–640
> Part ก่อนหน้า: [Part 79 — CRTP และ Template Pattern ขั้นสูง](./part-079-crtp-patterns.md) | Part ถัดไป: [Part 81 — std::thread และ Concurrency](./part-081-std-thread.md)

## เป้าหมายการเรียนรู้ (Learning Objectives)

หลังจบ Part นี้ ผู้เรียนจะสามารถ:

1. อธิบายได้ว่า **C++ Core Guidelines** คืออะไร ใครเป็นคนสร้าง และทำไมมันถึงเป็นเอกสารอ้างอิงที่
   สำคัญที่สุดฉบับหนึ่งของวงการ C++ ในปัจจุบัน
2. ท่อง (และเข้าใจเหตุผลเบื้องหลัง) รายการ "Prefer X over Y" ที่สำคัญที่สุด 18 ข้อ ที่สรุปรวบยอด
   ความรู้จาก Module D, E, และ F ทั้งหมดของหลักสูตรนี้
3. อธิบายหลักการ **RAII** และ **Rule of Zero/Three/Five** ในฐานะแกนกลางของการจัดการทรัพยากรใน
   Modern C++
4. Refactor โค้ดสไตล์เก่า (C-with-Classes) ให้กลายเป็นโค้ดสไตล์ Modern C++ ได้ด้วยตัวเอง พร้อมอธิบาย
   เหตุผลของการเปลี่ยนแปลงแต่ละจุด
5. รู้จักเครื่องมือที่ช่วยบังคับใช้ Guideline โดยอัตโนมัติ (clang-tidy, Compiler Explorer,
   Core Guidelines Support Library — GSL) และรู้ว่าจะเจาะลึกเครื่องมือเหล่านี้ต่อใน Part ไหน
6. อ่านและประเมินคุณภาพของโค้ด C++ สมัยใหม่ที่เขียนโดยคนอื่นได้อย่างเป็นระบบ ด้วย Checklist ที่
   นำไปใช้ได้จริงในการทำ Code Review
7. เชื่อมโยงภาพรวมทั้งหมดของ Module D (OOP), E (Template/STL), F (Modern Features) เข้าด้วยกันเป็น
   หลักคิดเดียวก่อนต่อยอดสู่ Module G (Concurrency)

---

## 80.1 C++ Core Guidelines คืออะไร (Step 633)

**C++ Core Guidelines** คือชุดแนวปฏิบัติ (Guidelines) อย่างเป็นทางการสำหรับการเขียนโค้ด C++ ที่ดี
เริ่มก่อตั้งขึ้นในปี 2015 โดย **Bjarne Stroustrup** (ผู้สร้างภาษา C++) และ **Herb Sutter** (ประธาน
คณะกรรมการมาตรฐาน ISO C++) ร่วมกับผู้เชี่ยวชาญอีกจำนวนมากในวงการ เผยแพร่แบบเปิด (Open Source) ที่
GitHub และปรับปรุงอย่างต่อเนื่องมาจนถึงปัจจุบัน (2026)

**ทำไมเอกสารนี้ถึงสำคัญขนาดนี้?** เพราะ C++ เป็นภาษาที่มีอายุมากกว่า 40 ปี สะสมทั้งฟีเจอร์ใหม่และ
"ของเก่าที่อันตราย" ไว้มากมาย (raw pointer, manual memory management, C-style array, macro) นักพัฒนา
คนหนึ่งอาจเขียนโค้ดสไตล์ปี 1998 ปนกับโค้ดสไตล์ปี 2023 ในไฟล์เดียวกันได้โดยคอมไพเลอร์ไม่ว่าอะไรเลย
C++ Core Guidelines จึงถูกสร้างขึ้นมาเพื่อตอบคำถามที่สำคัญที่สุดข้อเดียว:

> **"ในบรรดาวิธีที่ C++ อนุญาตให้ทำได้ทั้งหมด วิธีไหนคือวิธีที่ 'ถูกต้อง' ในโลกปี 2026?"**

Guideline แบ่งออกเป็นหมวดหมู่ใหญ่ๆ เช่น Philosophy, Interfaces, Functions, Classes, Resource
Management, Expressions, Performance, Concurrency แต่ละข้อมีรหัสอ้างอิง (เช่น `C.1`, `R.1`, `F.1`)
พร้อมเหตุผลและตัวอย่างประกอบ

หลักปรัชญาที่ครอบคลุมทั้งเอกสารคือ 2 ประโยคสั้นๆ ที่สรุปทุกอย่างของ Modern C++ ไว้:

> **"Type สร้างความปลอดภัยแบบ Static (Type Safety)"**
> **"Resource สร้างความปลอดภัยผ่าน RAII (Resource Safety)"**

ตลอด Part นี้เราจะนำหลักการเหล่านี้มาสรุปเป็นแนวทางที่นำไปใช้ได้จริงทันที โดยอ้างอิงกลับไปยัง
Part ต่างๆ ที่เราเรียนมาตลอด Module D, E, และ F

---

## 80.2 RAII และ Resource Management: แกนกลางของทุกอย่าง (Step 634)

ก่อนจะไปดูรายการ "Prefer X over Y" เราต้องย้ำหลักการที่สำคัญที่สุดอีกครั้ง เพราะทุกข้อในหมวด
Resource Management ล้วนต่อยอดจากหลักการเดียวนี้: **RAII (Resource Acquisition Is
Initialization)** ที่เราเรียนละเอียดไปแล้วใน **Part 68**

หลักการคือ: **ผูกอายุของทรัพยากร (memory, file handle, mutex, socket) เข้ากับอายุของ object** —
ทรัพยากรถูกจองใน Constructor และถูกคืนอัตโนมัติใน Destructor เมื่อ object หมดอายุ (ออกจาก scope,
ถูก `delete`, หรือ container ที่เก็บมันถูกทำลาย) ไม่ว่าจะออกจาก scope แบบปกติหรือผ่าน exception
ก็ตาม

```cpp
#include <iostream>
#include <memory>
#include <vector>

class Resource {
public:
    explicit Resource(int id) : id_(id) {
        std::cout << "จอง Resource #" << id_ << "\n";
    }
    ~Resource() {
        std::cout << "คืน Resource #" << id_ << "\n";
    }
    int id() const { return id_; }

private:
    int id_;
};

void process_with_raii() {
    auto r1 = std::make_unique<Resource>(1);
    std::vector<std::unique_ptr<Resource>> pool;
    pool.push_back(std::make_unique<Resource>(2));
    pool.push_back(std::make_unique<Resource>(3));

    std::cout << "กำลังใช้งาน Resource #" << r1->id() << "\n";
    // ไม่ต้องเรียก delete เองเลยแม้แต่บรรทัดเดียว
    // เมื่อออกจากฟังก์ชันนี้ r1 และ pool จะถูกทำลายอัตโนมัติ
    // ตามลำดับย้อนกลับ (Reverse Order) และคืน Resource ทั้งหมดให้เอง
}

int main() {
    process_with_raii();
    std::cout << "ออกจากฟังก์ชันแล้ว\n";
    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++20 raii_demo.cpp -o raii_demo && ./raii_demo
```

ผลลัพธ์:

```
จอง Resource #1
จอง Resource #2
จอง Resource #3
กำลังใช้งาน Resource #1
คืน Resource #2
คืน Resource #3
คืน Resource #1
ออกจากฟังก์ชันแล้ว
```

สังเกตลำดับการคืน: ตัวแปร local สองตัว (`r1` และ `pool`) ถูกทำลายตาม **Reverse Order ของการ
ประกาศเสมอ** (เข้าทีหลัง ออกก่อน — เหมือน Stack) จึงเห็นว่า `pool` (ประกาศทีหลัง) ถูกทำลายก่อน `r1`
(ประกาศก่อน) ส่วน**ลำดับภายใน** `pool` เอง (ระหว่าง Resource #2 กับ #3) เป็นรายละเอียดการ
implement ของ `std::vector` (โดยทั่วไปจะทำลาย element ตามลำดับ index จากน้อยไปมาก) ซึ่งไม่ใช่สิ่งที่
โค้ดเราควรพึ่งพาโดยตรง สิ่งที่รับประกันแน่นอนคือ: **ทุก object ที่ได้ทรัพยากรมา จะถูกคืนครบเสมอ**
ไม่ว่าฟังก์ชันจะจบแบบปกติหรือจบเพราะ Exception ก็ตาม — นี่คือกลไกที่ Compiler จัดการให้อัตโนมัติ
และเป็นเหตุผลที่ RAII ปลอดภัยกว่าการ `free`/`delete` มือมาก เพราะไม่มีทางลืมคืนทรัพยากรเลย

จากหลักการนี้ต่อยอดไปสู่กฎการออกแบบ class ที่สำคัญที่สุดข้อหนึ่งของ Modern C++: **Rule of
Zero / Three / Five** (ที่ปูพื้นฐานไว้ตั้งแต่ Part 46 และ Part 70):

| กฎ | ความหมาย | เมื่อไหร่ใช้ |
|---|---|---|
| **Rule of Zero** | ไม่ต้องเขียน Destructor, Copy/Move Constructor, Copy/Move Assignment เองเลยแม้แต่ตัวเดียว | เมื่อ class มีแค่ member ที่เป็น type ที่จัดการตัวเองอยู่แล้ว (`std::string`, `std::vector`, `std::unique_ptr`) — **ควรเป็นเป้าหมาย default ของทุก class ใหม่** |
| **Rule of Three** (C++98 เดิม) | ถ้าต้องเขียน Destructor เอง มักต้องเขียน Copy Constructor และ Copy Assignment เองด้วย | เมื่อ class ถือทรัพยากรดิบ (raw pointer, file handle) โดยตรง — ในโค้ดสมัยใหม่ควรเลี่ยงสถานการณ์นี้ด้วยการห่อด้วย smart pointer แทน |
| **Rule of Five** (C++11+) | เพิ่ม Move Constructor และ Move Assignment เข้าไปจาก Rule of Three | เมื่อต้องการให้ class ย้ายทรัพยากรได้อย่างมีประสิทธิภาพ (Part 70) แทนที่จะ copy เสมอ |

---

## 80.3 "Prefer X over Y": 18 ข้อสำคัญที่สุด (Step 635–637)

นี่คือหัวใจของ Part นี้ — รายการสรุปแนวปฏิบัติที่สำคัญที่สุดของ Modern C++ แต่ละข้ออ้างอิงกลับไปยัง
Part ที่เกี่ยวข้องในหลักสูตรนี้ เพื่อให้เห็นว่าทุกอย่างที่เรียนมาตลอด Module D-F ประกอบกันเป็น
ปรัชญาเดียว

### หมวด Resource Management

**1. Prefer Smart Pointer over Raw Pointer + `new`/`delete`** (Part 67)

```cpp
#include <memory>

struct Widget {
    int value = 0;
};

// ไม่แนะนำ
void not_recommended() {
    Widget* w = new Widget();
    // ... ใช้งาน w ...
    delete w;  // ลืมบรรทัดนี้ = memory leak ทันที
}

// แนะนำ
void recommended() {
    auto w = std::make_unique<Widget>();
    // ไม่ต้อง delete เอง ปลอดภัยแม้เกิด exception ระหว่างทาง
    (void)w;
}

int main() {
    not_recommended();
    recommended();
    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++20 prefer_smart_pointer.cpp -o prefer_smart_pointer
```

**2. Prefer RAII over Manual Cleanup** (Part 68) — ผูกทรัพยากรทุกชนิด (ไม่ใช่แค่ memory) เข้ากับ
อายุของ object เสมอ ไม่ว่าจะเป็น file handle, mutex lock, network connection

**3. Prefer `std::array`/`std::vector` over C-style Array** (Part 59) — C-style array (`int
arr[10]`) ไม่รู้ขนาดตัวเอง (decay เป็น pointer ทันทีที่ส่งเข้าฟังก์ชัน) ในขณะที่ `std::array` และ
`std::vector` มี `.size()`, bounds-checking ผ่าน `.at()`, และทำงานร่วมกับ STL algorithm ได้ทันที

**4. Rule of Zero over Rule of Three/Five เมื่อเป็นไปได้** (80.2) — ให้ member ที่จัดการทรัพยากร
ตัวเองอยู่แล้วทำงานแทน ลดโอกาสเขียน Copy/Move ผิดพลาด

### หมวด Types & Declarations

**5. Prefer `nullptr` over `NULL` หรือ `0`** (Part 41) — `nullptr` มีชนิดข้อมูลของตัวเอง
(`std::nullptr_t`) ทำให้ Overload Resolution ไม่สับสนระหว่าง pointer กับ integer เหมือนที่ `NULL`
(ซึ่งมักเป็นแค่ macro แทน `0`) เคยทำให้เกิดปัญหา

**6. Prefer `const` ทุกที่ที่เป็นไปได้ (const-correctness)** (Part 43) — ทั้งตัวแปร, parameter
(โดยเฉพาะ `const&` สำหรับ object ขนาดใหญ่), และ member function ที่ไม่แก้ไข state ควรเป็น `const`
เสมอ ช่วยให้ compiler ตรวจจับบั๊กจากการแก้ไขค่าที่ไม่ตั้งใจได้ตั้งแต่ compile time

**7. Prefer `auto` เมื่อทำให้โค้ดอ่านง่ายขึ้น** (Part 69) — โดยเฉพาะกับ Iterator ที่มีชื่อชนิดยาว
มาก (`std::unordered_map<std::string, std::vector<int>>::const_iterator`) แต่ **ไม่ควรใช้ `auto`
พร่ำเพรื่อจนผู้อ่านเดาชนิดข้อมูลไม่ออกเลย** — สมดุลระหว่างความสั้นกับความชัดเจนคือหัวใจสำคัญ

**8. Prefer Range-Based For over Index-Based Loop** (Part 62) — เมื่อไม่จำเป็นต้องใช้ index จริงๆ
`for (const auto& x : container)` ปลอดภัยกว่า (ไม่มีทาง off-by-one) และอ่านง่ายกว่า `for (size_t i
= 0; i < container.size(); ++i)`

**9. Prefer Uniform Initialization `{}` over `()`/`=` เมื่อเหมาะสม** (Part 69) — ป้องกัน
Narrowing Conversion ที่เกิดขึ้นเงียบๆ (เช่น `int x = 3.9;` ตัดทศนิยมทิ้งโดยไม่เตือน แต่ `int x{3.9};`
จะ error ทันทีที่ compile) และแก้ปัญหา "Most Vexing Parse"

**10. Prefer `enum class` over `enum` ธรรมดา** (Part 69) — `enum class` ไม่รั่วชื่อค่าคงที่เข้า
scope ภายนอก และไม่แปลงเป็น `int` โดยอัตโนมัติ ป้องกันบั๊กจากการเปรียบเทียบ enum คนละกลุ่มกัน

### หมวด Containers & Algorithms

**11. Prefer STL Algorithm over Handwritten Raw Loop** (Part 63) — `std::sort`, `std::find`,
`std::accumulate` ผ่านการทดสอบมาอย่างละเอียด อ่านง่ายกว่า และมักเร็วกว่าลูปที่เขียนมือ (เพราะ
implementation ภายในถูก optimize เฉพาะทาง)

**12. Prefer `std::string`/`std::string_view` over C-style `char*`** (Part 65) — จัดการ memory
เอง ไม่มีปัญหา buffer overflow จาก `strcpy`, รู้ความยาวตัวเองผ่าน `.size()` โดยไม่ต้องวน `strlen`

**13. Prefer Passing by `const&` over Passing by Value สำหรับ Object ขนาดใหญ่** (Part 44) —
หลีกเลี่ยงการ copy ที่ไม่จำเป็น แต่สำหรับ type เล็กๆ (`int`, `double`, iterator) ควร pass by value
ตามปกติ เพราะการส่ง reference อาจมี overhead มากกว่าการ copy ค่าที่เล็กกว่า pointer เสียอีก

### หมวด Functions & Classes

**14. Prefer `override` เสมอเมื่อ Override Virtual Function** (Part 49) — ให้ compiler ตรวจสอบ
Signature ให้ตรงกับ Base Class จริง ป้องกันบั๊กเงียบจากการพิมพ์ signature ผิดเพี้ยนเล็กน้อย

**15. Prefer Virtual Destructor เสมอเมื่อ Class มี Virtual Function อย่างน้อยหนึ่งตัว** (Part 49)
— ป้องกัน Undefined Behavior ร้ายแรงตอน `delete` ผ่าน Base Pointer

**16. Prefer Concept (C++20) over SFINAE ดิบๆ สำหรับ Generic Code ใหม่** (Part 73, Part 78) —
อ่านง่ายกว่า Error Message ชัดเจนกว่า ใช้ SFINAE ต่อเมื่อจำเป็นต้องรองรับมาตรฐานเก่ากว่า C++20

**17. Prefer Move Semantics (`std::move`) over Unnecessary Copy สำหรับ Resource-Heavy Object**
(Part 70) — โดยเฉพาะตอน return object ขนาดใหญ่จากฟังก์ชัน หรือย้าย ownership ของ container/string

**18. Prefer Composition over Inheritance เมื่อไม่ได้ต้องการ "is-a" Relationship จริงๆ** (Part
48, Part 50) — Inheritance ควรใช้เมื่อมีความสัมพันธ์แบบ "เป็นชนิดย่อยของ" ที่แท้จริงเท่านั้น
(Liskov Substitution) การ "อยากใช้โค้ดร่วมกัน" เฉยๆ ควรใช้ Composition (มี object เป็น member)
หรือ CRTP Mixin (Part 79) แทน

### ตัวอย่างรวม: เห็นหลายข้อพร้อมกันในโค้ดเดียว

เพื่อให้เห็นภาพว่าข้อ 3, 5, 7, 8, 11 ทำงานร่วมกันในโค้ดจริงได้อย่างไร ลองดูโปรแกรมสั้นๆ ที่รวม
หลักการเหล่านี้ไว้ในที่เดียว:

```cpp
#include <algorithm>
#include <array>
#include <iostream>
#include <memory>
#include <numeric>
#include <vector>

struct Widget {
    int id;
    void greet() const { std::cout << "Widget #" << id << "\n"; }
};

int main() {
    // ข้อ 1: smart pointer แทน raw pointer + new/delete
    auto w = std::make_unique<Widget>(Widget{1});
    w->greet();

    // ข้อ 8: range-based for แทน index-based loop
    std::vector<int> numbers{1, 2, 3, 4, 5};
    int sum = 0;
    for (const int n : numbers) {
        sum += n;
    }
    std::cout << "sum (range-based for) = " << sum << "\n";

    // ข้อ 7: auto เมื่อชนิดข้อมูลยาวหรือชัดเจนจาก RHS อยู่แล้ว
    auto it = std::find(numbers.begin(), numbers.end(), 3);
    if (it != numbers.end()) {
        std::cout << "พบเลข 3 ที่ตำแหน่ง " << std::distance(numbers.begin(), it) << "\n";
    }

    // ข้อ 3: std::array แทน C array แบบ fixed-size
    std::array<int, 3> fixed{10, 20, 30};
    std::cout << "fixed.size() = " << fixed.size() << "\n";

    // ข้อ 11: STL algorithm แทนการเขียน loop เอง
    const int total = std::accumulate(numbers.begin(), numbers.end(), 0);
    std::cout << "total (accumulate) = " << total << "\n";

    // ข้อ 5: nullptr แทน NULL / 0
    Widget* maybe = nullptr;
    std::cout << "maybe == nullptr: " << std::boolalpha << (maybe == nullptr) << "\n";

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++20 prefer_examples.cpp -o prefer_examples
./prefer_examples
```

ผลลัพธ์:

```
Widget #1
sum (range-based for) = 15
พบเลข 3 ที่ตำแหน่ง 2
fixed.size() = 3
total (accumulate) = 15
maybe == nullptr: true
```

สังเกตว่าโค้ดทั้งหมดนี้ **ไม่มี `new`/`delete` แม้แต่บรรทัดเดียว ไม่มี Index-based Loop ที่ไม่จำเป็น
ไม่มี C-style Array ไม่มี `NULL`** — นี่คือสิ่งที่ "โค้ด C++ สมัยใหม่" ที่ดีหน้าตาเป็นแบบนี้จริงๆ
ในการทำงานประจำวัน ไม่ใช่แค่ตัวอย่างในหนังสือ

---

## 80.4 เครื่องมือช่วยบังคับใช้ Guideline (Step 638)

การจำ Guideline ทั้งหมดในหัวไม่ใช่วิธีที่ยั่งยืน วงการ C++ จึงมีเครื่องมือที่ช่วย **ตรวจจับการ
ละเมิด Guideline โดยอัตโนมัติ** ระหว่างพัฒนา:

| เครื่องมือ | หน้าที่ | รายละเอียดเพิ่มเติม |
|---|---|---|
| **clang-tidy** | Static Analyzer ที่มี Check ตรง `cppcoreguidelines-*` ครบเกือบทุกข้อในเอกสาร Core Guidelines เช่น ตรวจจับ raw `new`/`delete`, การใช้ `NULL`, การไม่ใส่ `override` | จะเรียนเจาะลึกวิธีตั้งค่าและใช้งานจริงใน **Part 95 (Static Analysis)** |
| **Compiler Warning (`-Wall -Wextra -Wpedantic`)** | ด่านแรกสุดที่ทุก Part ในหลักสูตรนี้บังคับใช้ตั้งแต่ Part 1 | ตรวจจับปัญหาพื้นฐาน เช่น unused variable, sign comparison, uninitialized value |
| **C++ Core Guidelines Support Library (GSL)** | ไลบรารีเสริม (`gsl::not_null`, `gsl::span`) ที่ทำให้ Guideline บางข้อ "บังคับได้จริงในโค้ด" ไม่ใช่แค่กฎที่ต้องจำ | เช่น `gsl::not_null<Widget*>` ทำให้ pointer ที่ประกาศไว้ **ห้ามเป็น `nullptr` เด็ดขาด** ตรวจสอบได้ตั้งแต่ compile/runtime |
| **Sanitizer (AddressSanitizer, UndefinedBehaviorSanitizer)** | ตรวจจับการละเมิด Memory Safety และ Undefined Behavior ที่ compiler ปกติมองไม่เห็น | จะเรียนเจาะลึกใน **Part 96 (Sanitizer และ Code Coverage)** |
| **Compiler Explorer (godbolt.org)** | เว็บไซต์ที่แสดง Assembly Output จาก Source Code แบบ real-time มีประโยชน์มากในการตรวจสอบว่า CRTP (Part 79) หรือ `constexpr` (Part 71) ถูก compiler จัดการอย่างที่คาดหวังจริงหรือไม่ | ใช้ประกอบการเรียน **Part 89 (Compiler Optimization)** |

**หลักการสำคัญ**: เครื่องมือเหล่านี้ไม่ได้มาแทนที่ความเข้าใจ แต่เป็น **ตาข่ายนิรภัย (Safety Net)**
ที่จับสิ่งที่มนุษย์พลาดได้ วิศวกรมืออาชีพใช้ทั้งความเข้าใจ (สิ่งที่เรียนใน Part นี้) **ควบคู่กับ**
เครื่องมืออัตโนมัติเสมอ ไม่ใช่พึ่งพาอย่างใดอย่างหนึ่งเพียงอย่างเดียว

---

## 80.5 Refactor จริง: จากสไตล์เก่าสู่ Modern C++ (Step 639)

มาดูตัวอย่างจริงที่รวมหลักการทั้งหมดในรายการ 80.3 เข้าด้วยกัน โดย refactor class จัดการรายชื่อแบบ
เก่า (สไตล์ "C-with-Classes" ที่พบได้บ่อยในโค้ด legacy) ให้กลายเป็น Modern C++

### ก่อน Refactor (สไตล์เก่า)

```cpp
#include <cstring>
#include <iostream>

class NameListOld {
public:
    NameListOld() : names_(nullptr), size_(0), capacity_(0) {}

    ~NameListOld() {
        for (int i = 0; i < size_; ++i) {
            delete[] names_[i];
        }
        delete[] names_;
    }

    NameListOld(const NameListOld& other) : size_(other.size_), capacity_(other.capacity_) {
        names_ = new char*[capacity_];
        for (int i = 0; i < size_; ++i) {
            names_[i] = new char[std::strlen(other.names_[i]) + 1];
            std::strcpy(names_[i], other.names_[i]);
        }
    }

    NameListOld& operator=(const NameListOld& other) {
        if (this == &other) {
            return *this;
        }
        for (int i = 0; i < size_; ++i) {
            delete[] names_[i];
        }
        delete[] names_;

        size_ = other.size_;
        capacity_ = other.capacity_;
        names_ = new char*[capacity_];
        for (int i = 0; i < size_; ++i) {
            names_[i] = new char[std::strlen(other.names_[i]) + 1];
            std::strcpy(names_[i], other.names_[i]);
        }
        return *this;
    }

    void add(const char* name) {
        if (size_ == capacity_) {
            grow();
        }
        names_[size_] = new char[std::strlen(name) + 1];
        std::strcpy(names_[size_], name);
        size_++;
    }

    void print_all() const {
        for (int i = 0; i < size_; i++) {
            std::cout << "- " << names_[i] << "\n";
        }
    }

private:
    void grow() {
        int new_capacity = (capacity_ == 0) ? 4 : capacity_ * 2;
        char** new_names = new char*[new_capacity];
        for (int i = 0; i < size_; i++) {
            new_names[i] = names_[i];
        }
        delete[] names_;
        names_ = new_names;
        capacity_ = new_capacity;
    }

    char** names_;
    int size_;
    int capacity_;
};

int main() {
    NameListOld list;
    list.add("Somchai");
    list.add("Suda");
    list.add("Anan");

    NameListOld copy = list;
    copy.add("Malee");

    std::cout << "list ต้นฉบับ:\n";
    list.print_all();
    std::cout << "copy (มีเพิ่ม Malee):\n";
    copy.print_all();

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++20 name_list_old.cpp -o name_list_old && ./name_list_old
```

ผลลัพธ์:

```
list ต้นฉบับ:
- Somchai
- Suda
- Anan
copy (มีเพิ่ม Malee):
- Somchai
- Suda
- Anan
- Malee
```

โค้ดนี้**คอมไพล์ผ่านโดยไม่มี Warning** และ**ทำงานถูกต้อง** แต่มีปัญหาเชิงคุณภาพมหาศาลซ่อนอยู่:

- ต้องเขียน Destructor, Copy Constructor, Copy Assignment เองครบ (Rule of Three) — เสี่ยงเขียนผิด
  หรือลืมอัปเดตให้ตรงกันเมื่อแก้โค้ดในอนาคต (เช่น ถ้าเพิ่ม member ใหม่แล้วลืมแก้ copy constructor)
- **ไม่มี Move Constructor/Assignment เลย** (ไม่ทำตาม Rule of Five) — ทุกครั้งที่ return
  `NameListOld` จากฟังก์ชันจะเกิดการ copy เต็มรูปแบบ (deep copy string ทุกตัว) แทนที่จะย้าย
  ownership แบบมีประสิทธิภาพ
- ใช้ raw pointer (`char**`, `char*`) และ manual memory management ทุกจุด — พื้นที่เสี่ยง memory
  leak/double free สูงมากถ้ามีคนมาแก้โค้ดในอนาคตแล้วพลาดจุดใดจุดหนึ่ง
- ใช้ index-based loop (`for (int i = 0; ...)`) ทั้งที่ไม่จำเป็นต้องใช้ index เลย
- ต้องเขียน logic การจัดการ capacity (`grow()`) เองทั้งหมด ทั้งที่ `std::vector` ทำสิ่งนี้ให้แล้ว
  อย่างผ่านการทดสอบมานับล้านครั้งทั่วโลก

### หลัง Refactor (Modern C++)

```cpp
#include <iostream>
#include <string>
#include <vector>

class NameList {
public:
    void add(const std::string& name) { names_.push_back(name); }

    void print_all() const {
        for (const auto& name : names_) {
            std::cout << "- " << name << "\n";
        }
    }

private:
    std::vector<std::string> names_;
    // ไม่ต้องเขียน destructor, copy constructor, copy assignment เอง
    // (Rule of Zero) เพราะ std::vector<std::string> จัดการหน่วยความจำให้หมดแล้ว
    // Move constructor/assignment ก็ได้มาฟรีโดยอัตโนมัติเช่นกัน
};

int main() {
    NameList list;
    list.add("Somchai");
    list.add("Suda");
    list.add("Anan");

    NameList copy = list;  // deep copy อัตโนมัติ ปลอดภัย ไม่มี double free
    copy.add("Malee");

    std::cout << "list ต้นฉบับ:\n";
    list.print_all();
    std::cout << "copy (มีเพิ่ม Malee):\n";
    copy.print_all();

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++20 name_list_modern.cpp -o name_list_modern
./name_list_modern
```

ผลลัพธ์:

```
list ต้นฉบับ:
- Somchai
- Suda
- Anan
copy (มีเพิ่ม Malee):
- Somchai
- Suda
- Anan
- Malee
```

**สรุปการเปลี่ยนแปลง** — โค้ดจาก **~85 บรรทัด ลดเหลือ ~15 บรรทัด** พร้อมข้อดีที่เพิ่มขึ้นทุกด้าน:

| ประเด็น | ก่อน (NameListOld) | หลัง (NameList) |
|---|---|---|
| จำนวนบรรทัด | ~85 บรรทัด | ~15 บรรทัด |
| Destructor/Copy/Move | เขียนเอง 3 ตัว (ขาด Move ไปเลย) | ไม่ต้องเขียนเลย (Rule of Zero) ได้ครบ 5 ตัวฟรี |
| ความเสี่ยง Memory Leak | สูง (raw pointer หลายชั้น) | แทบเป็นศูนย์ (ไม่มี raw pointer เลย) |
| ประสิทธิภาพตอน return/copy | Copy เต็มรูปแบบเสมอ (ไม่มี Move) | ใช้ Move อัตโนมัติเมื่อทำได้ |
| อ่านเข้าใจได้ง่ายแค่ไหน | ต้องอ่านทุกบรรทัดเพื่อมั่นใจว่าไม่มีบั๊ก | อ่านครั้งเดียวเข้าใจทันที (`vector<string>` บอกทุกอย่าง) |

นี่คือภาพสะท้อนที่ชัดเจนที่สุดของคำว่า **"Modern C++"**: ไม่ใช่การใช้ syntax ใหม่ๆ เพื่อความเท่ ("ฉัน
ใช้ `auto` เยอะ ฉันจึงเขียน Modern C++") แต่คือการ**มอบความรับผิดชอบด้านความถูกต้องให้กับ Type System
และ Standard Library** แทนที่จะแบกรับมันด้วยวินัยของมนุษย์ล้วนๆ เหมือนในยุค C++98

---

## 80.6 อ่านโค้ด Modern C++ อย่างมืออาชีพ: Checklist สำหรับ Code Review (Step 640)

เมื่อทำ Code Review หรืออ่านโค้ดของคนอื่น ให้ตรวจสอบตาม Checklist นี้ตามลำดับ (สรุปทุกอย่างจาก
Part นี้และ Module D-F ทั้งหมด):

**ระดับ Resource & Memory**

- [ ] มี `new`/`delete` ตรงๆ ที่ไม่ได้อยู่ใน implementation ของ smart pointer เองหรือไม่? (ถ้ามี
      ต้องมีเหตุผลชัดเจนว่าทำไมใช้ smart pointer ไม่ได้)
- [ ] Class ที่มี Destructor ที่เขียนเอง มี Copy/Move Constructor/Assignment ครบตาม Rule of
      Three/Five หรือไม่? หรือควรทำตาม Rule of Zero แทน?
- [ ] Class ที่มี Virtual Function มี Virtual Destructor หรือไม่?

**ระดับ Type & Const-Correctness**

- [ ] Parameter ที่เป็น object ขนาดใหญ่และไม่ต้องแก้ไข ผ่านมาแบบ `const&` หรือไม่?
- [ ] Member function ที่ไม่แก้ไข state ของ object ถูกทำเครื่องหมาย `const` ครบหรือไม่?
- [ ] มีการใช้ `NULL` หรือ `0` แทน `nullptr` หลงเหลืออยู่หรือไม่?
- [ ] Override ของ virtual function ทุกตัวมี `override` keyword กำกับหรือไม่?

**ระดับ Container & Algorithm**

- [ ] มี C-style array (`int arr[N]`) ที่ควรเป็น `std::array`/`std::vector` หรือไม่?
- [ ] มี raw loop ที่ทำสิ่งที่ STL algorithm ทำได้อยู่แล้ว (`sort`, `find`, `accumulate`) หรือไม่?
- [ ] Loop ที่ไม่จำเป็นต้องใช้ index ใช้ range-based for แล้วหรือยัง?

**ระดับ Generic Code (Template)**

- [ ] Generic function/class ใหม่ (โปรเจกต์ C++20+) ใช้ Concept แทน SFINAE ดิบๆ หรือไม่?
- [ ] มีการใช้ CRTP ทั้งที่ Dynamic Polymorphism ธรรมดาก็เพียงพอหรือไม่? (ตรวจสอบว่ามีเหตุผลด้าน
      Performance ที่วัดจริงรองรับหรือไม่ ตาม 79.7)

**ระดับ Warning & Tooling**

- [ ] คอมไพล์ผ่าน `-Wall -Wextra -Wpedantic` โดยไม่มี Warning หรือไม่?
- [ ] ผ่าน clang-tidy check กลุ่ม `cppcoreguidelines-*` หรือไม่? (รายละเอียดเต็มใน Part 95)

Checklist นี้ไม่ใช่กฎตายตัวที่ต้องทำตาม 100% ทุกครั้ง — **ทุก Guideline มีข้อยกเว้นที่สมเหตุสมผล**
เสมอ (เช่น Embedded System ที่มี RAM จำกัดมากอาจจำเป็นต้องใช้ raw pointer เพื่อควบคุม memory
อย่างละเอียด) สิ่งสำคัญที่สุดคือ **เมื่อจะเบี่ยงเบนจาก Guideline ต้องรู้ตัวว่าทำไมถึงเบี่ยงเบน และ
อธิบายเหตุผลนั้นได้** ไม่ใช่เบี่ยงเบนเพราะ "ไม่รู้ว่ามีทางที่ดีกว่า"

---

## ข้อผิดพลาดที่พบบ่อย (Common Pitfalls)

1. **ใช้ `auto` จนอ่านโค้ดไม่รู้เรื่องว่าตัวแปรเป็นชนิดอะไร** — `auto` ทำให้อ่านง่ายขึ้นเมื่อชนิด
   ข้อมูลชัดเจนจากบริบท (เช่น `auto w = std::make_unique<Widget>();` อ่านง่ายกว่าการเขียนชนิดยาวๆ)
   แต่ `auto result = compute();` ที่ไม่รู้ว่า `compute()` คืนอะไรกลับทำให้อ่านยากขึ้น ต้องสมดุลเสมอ

2. **เข้าใจผิดว่า Rule of Zero แปลว่า "ไม่ต้องคิดเรื่อง Copy/Move เลย"** — Rule of Zero หมายถึง
   "ไม่ต้อง**เขียน**เอง" เพราะ compiler generate ให้ถูกต้องอัตโนมัติ แต่ผู้เขียนโค้ดยังต้อง**เข้าใจ**
   ว่า class ของตัวเอง copy/move ได้ถูกต้องหรือไม่ (เช่น ถ้ามี raw pointer เป็น member แม้แต่ตัวเดียว
   Rule of Zero จะใช้ไม่ได้ทันที เพราะ compiler จะ generate shallow copy ที่ผิดพลาด)

3. **เปลี่ยนโค้ดเก่าทั้งหมดเป็น Modern C++ ในคราวเดียวโดยไม่มีการทดสอบ (Test) รองรับ** — การ
   Refactor เช่นใน 80.5 ควรทำทีละส่วนเล็กๆ พร้อม Unit Test (Part 93) คอยยืนยันพฤติกรรมเดิมไม่
   เปลี่ยนแปลง การเปลี่ยนทั้งไฟล์รวดเดียวมีความเสี่ยงสูงที่จะแอบเปลี่ยนพฤติกรรมโดยไม่ได้ตั้งใจ

4. **ใช้ Guideline เป็นข้ออ้างในการ "Over-Engineer"** — เช่น ใส่ CRTP หรือ Template ที่ซับซ้อนเกิน
   ความจำเป็นเพียงเพราะ "Guideline บอกว่าดี" ทั้งที่โค้ดธรรมดาก็เพียงพอ Guideline มีไว้เพื่อแก้ปัญหา
   จริง ไม่ใช่เป้าหมายในตัวมันเอง — โค้ดที่ดีที่สุดคือโค้ดที่**เรียบง่ายที่สุดเท่าที่ยังถูกต้อง**

5. **ไม่เปิด Warning เต็มรูปแบบ แล้วคิดว่าโค้ดตัวเอง "สะอาด" แล้ว** — Guideline จำนวนมากตรวจจับ
   ไม่ได้ด้วยสายตาเปล่า ต้องพึ่งเครื่องมือ (clang-tidy, sanitizer) เสมอ ตามที่อธิบายใน 80.4

6. **มองว่า C++ Core Guidelines เป็นกฎตายตัว 100% แล้วเถียงกับเพื่อนร่วมทีมโดยไม่ฟังบริบท** —
   Guideline ทุกข้อมีเหตุผลรองรับ ไม่ใช่ "เพราะเอกสารบอกไว้" การใช้ Guideline อย่างมืออาชีพคือ
   การเข้าใจ**เหตุผล**เบื้องหลังแต่ละข้อ แล้วนำไปปรับใช้ตามบริบทของโปรเจกต์จริง ไม่ใช่ท่องจำแล้วยึด
   ติดแบบไม่ยืดหยุ่น

---

## แบบฝึกหัดท้ายบท

1. หยิบโค้ด `NameListOld` ใน 80.5 มาทดลองคอมไพล์และรันด้วยตัวเอง จากนั้นลองเพิ่มฟังก์ชัน
   `remove_last()` ที่ลบชื่อสุดท้ายออก ทำใน **ทั้งสองเวอร์ชัน** (เก่าและใหม่) แล้วเปรียบเทียบว่า
   เวอร์ชันไหนเขียนง่ายกว่าและมีโอกาสเกิดบั๊กน้อยกว่า

2. เขียน class `Matrix` แบบ "สไตล์เก่า" ที่ใช้ `double**` (pointer-to-pointer) จัดการเมทริกซ์ 2 มิติ
   ด้วยมือทั้งหมด (constructor/destructor/copy) จากนั้น refactor เป็น "สไตล์ใหม่" โดยใช้
   `std::vector<std::vector<double>>` เปรียบเทียบจำนวนบรรทัดและความเสี่ยงบั๊ก

3. จากรายการ "Prefer X over Y" ทั้ง 18 ข้อใน 80.3 เลือกมา 5 ข้อที่คิดว่าสำคัญที่สุดสำหรับตัวเอง
   เขียนอธิบายด้วยคำพูดตัวเอง (ไม่ copy จากบทเรียน) ว่าทำไมข้อนั้นถึงสำคัญ พร้อมยกตัวอย่างโค้ด
   ประกอบทั้ง "แบบผิด" และ "แบบถูก" ของแต่ละข้อ

4. ใช้ Checklist ใน 80.6 ตรวจสอบโค้ดที่ตัวเองเคยเขียนใน Part ก่อนหน้าของหลักสูตรนี้ (เลือกมาสัก
   1-2 ไฟล์) แล้วเขียนรายงานสั้นๆ ว่าพบข้อบกพร่องอะไรบ้างตาม Checklist และจะแก้ไขอย่างไร

5. เขียนฟังก์ชัน `template <typename T> void old_style_swap(T* a, T* b)` ที่ swap ค่าด้วย pointer
   แบบสไตล์เก่า (C-style) แล้วเขียนใหม่เป็น `template <typename T> void modern_swap(T& a, T& b)`
   ที่ใช้ `std::move` (Part 70) แทน อธิบายว่าทำไมเวอร์ชันใหม่ปลอดภัยกว่า

6. อธิบาย (เป็นข้อความหรือ comment ในโค้ด) ว่าทำไม C++ Core Guidelines ถึงแนะนำ "Prefer Composition
   over Inheritance" ทั้งที่ Inheritance เป็นฟีเจอร์หลักของ OOP ที่เราเรียนมาตั้งแต่ Part 48
   ยกตัวอย่างสถานการณ์ที่ใช้ Inheritance ผิดที่มาประกอบคำอธิบาย

### แนวทางเฉลยข้อ 1

```cpp
#include <cstring>
#include <iostream>
#include <string>
#include <vector>

// ---------- สไตล์เก่า: เพิ่ม remove_last() ----------
class NameListOld {
public:
    NameListOld() : names_(nullptr), size_(0), capacity_(0) {}

    ~NameListOld() {
        for (int i = 0; i < size_; ++i) {
            delete[] names_[i];
        }
        delete[] names_;
    }

    void add(const char* name) {
        if (size_ == capacity_) {
            grow();
        }
        names_[size_] = new char[std::strlen(name) + 1];
        std::strcpy(names_[size_], name);
        size_++;
    }

    void remove_last() {
        if (size_ == 0) {
            return;  // ต้องเช็คเอง ไม่งั้น underflow
        }
        size_--;
        delete[] names_[size_];  // ต้องจำ free เอง ไม่งั้น leak
        names_[size_] = nullptr;
    }

    void print_all() const {
        for (int i = 0; i < size_; i++) {
            std::cout << "- " << names_[i] << "\n";
        }
    }

private:
    void grow() {
        int new_capacity = (capacity_ == 0) ? 4 : capacity_ * 2;
        char** new_names = new char*[new_capacity];
        for (int i = 0; i < size_; i++) {
            new_names[i] = names_[i];
        }
        delete[] names_;
        names_ = new_names;
        capacity_ = new_capacity;
    }

    char** names_;
    int size_;
    int capacity_;
};

// ---------- สไตล์ใหม่: เพิ่ม remove_last() ----------
class NameList {
public:
    void add(const std::string& name) { names_.push_back(name); }

    void remove_last() {
        if (!names_.empty()) {
            names_.pop_back();  // vector จัดการ memory ให้เองทั้งหมด
        }
    }

    void print_all() const {
        for (const auto& name : names_) {
            std::cout << "- " << name << "\n";
        }
    }

private:
    std::vector<std::string> names_;
};

int main() {
    NameListOld old_list;
    old_list.add("A");
    old_list.add("B");
    old_list.add("C");
    old_list.remove_last();
    std::cout << "NameListOld หลัง remove_last():\n";
    old_list.print_all();

    NameList new_list;
    new_list.add("A");
    new_list.add("B");
    new_list.add("C");
    new_list.remove_last();
    std::cout << "NameList หลัง remove_last():\n";
    new_list.print_all();

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++20 ex1.cpp -o ex1 && ./ex1
```

ผลลัพธ์:

```
NameListOld หลัง remove_last():
- A
- B
NameList หลัง remove_last():
- A
- B
```

**อธิบาย**: ใน `NameListOld` ฟังก์ชัน `remove_last()` ต้อง **จำเรื่อง memory management เองสองเรื่อง
พร้อมกัน**: (1) เช็คไม่ให้ `size_` ติดลบ (2) เรียก `delete[]` คืนหน่วยความจำของ string ตัวสุดท้าย
ก่อนลด `size_` ลง ถ้าลืมข้อใดข้อหนึ่งจะเกิด Memory Leak หรือ Undefined Behavior ทันที ในขณะที่
`NameList::remove_last()` ใช้ `names_.pop_back()` เพียงบรรทัดเดียว — `std::vector<std::string>`
จัดการทั้งการคืน memory ของ `std::string` ตัวสุดท้ายและการลดขนาดให้ทั้งหมดโดยอัตโนมัติ ไม่มีทางลืม
ขั้นตอนใดขั้นตอนหนึ่งได้เลย นี่คือตัวอย่างที่ชัดเจนว่าทำไม Rule of Zero (80.2) ถึงไม่ได้แค่ลดจำนวน
บรรทัดตอนเขียน constructor/destructor เท่านั้น แต่ยังลดความเสี่ยงบั๊กในทุกฟังก์ชันที่เพิ่มเข้ามาใน
อนาคตด้วย

### แนวทางเฉลยข้อ 2

```cpp
#include <iostream>
#include <vector>

// ---------- แบบเก่า: double** จัดการมือทั้งหมด ----------
class MatrixOld {
public:
    MatrixOld(int rows, int cols) : rows_(rows), cols_(cols) {
        data_ = new double*[rows_];
        for (int i = 0; i < rows_; ++i) {
            data_[i] = new double[cols_]();
        }
    }

    ~MatrixOld() {
        for (int i = 0; i < rows_; ++i) {
            delete[] data_[i];
        }
        delete[] data_;
    }

    MatrixOld(const MatrixOld& other) : rows_(other.rows_), cols_(other.cols_) {
        data_ = new double*[rows_];
        for (int i = 0; i < rows_; ++i) {
            data_[i] = new double[cols_];
            for (int j = 0; j < cols_; ++j) {
                data_[i][j] = other.data_[i][j];
            }
        }
    }

    MatrixOld& operator=(const MatrixOld&) = delete;  // ตัดทิ้งเพื่อความง่าย (โฟกัสที่ประเด็นหลัก)

    void set(int r, int c, double value) { data_[r][c] = value; }
    double get(int r, int c) const { return data_[r][c]; }

private:
    double** data_;
    int rows_;
    int cols_;
};

// ---------- แบบใหม่: vector<vector<double>> ----------
class MatrixNew {
public:
    MatrixNew(int rows, int cols) : data_(rows, std::vector<double>(cols, 0.0)) {}

    void set(int r, int c, double value) { data_[r][c] = value; }
    double get(int r, int c) const { return data_[r][c]; }
    // ไม่ต้องเขียน destructor/copy constructor/copy assignment เองเลย (Rule of Zero)

private:
    std::vector<std::vector<double>> data_;
};

int main() {
    MatrixOld old_mat(2, 2);
    old_mat.set(0, 0, 1.0);
    old_mat.set(1, 1, 2.0);
    MatrixOld old_copy = old_mat;
    std::cout << "MatrixOld copy(0,0) = " << old_copy.get(0, 0) << "\n";

    MatrixNew new_mat(2, 2);
    new_mat.set(0, 0, 1.0);
    new_mat.set(1, 1, 2.0);
    MatrixNew new_copy = new_mat;
    std::cout << "MatrixNew copy(1,1) = " << new_copy.get(1, 1) << "\n";

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++20 ex2.cpp -o ex2 && ./ex2
```

ผลลัพธ์:

```
MatrixOld copy(0,0) = 1
MatrixNew copy(1,1) = 2
```

**อธิบาย**: `MatrixOld` ต้องเขียน constructor, destructor, copy constructor เองครบ (และต้อง
ตัดสินใจจัดการ copy assignment ด้วย ในตัวอย่างนี้เลือก `= delete` เพื่อไม่ต้องเขียนโค้ดซ้ำ ซึ่งเป็น
ข้อจำกัดที่ต้องแลก) ขณะที่ `MatrixNew` ใช้ `std::vector<std::vector<double>>` เพียงบรรทัดเดียวใน
constructor แล้วได้ copy/move/destructor ที่ถูกต้องครบถ้วนฟรีทั้งหมดตาม Rule of Zero — ความเสี่ยง
บั๊กจาก `MatrixOld` มีสูงกว่ามาก เช่น ถ้าลืมเขียน bounds check ใน `set`/`get` จะเข้าถึง memory
นอกขอบเขตได้ง่ายกว่าที่คิด (แม้ `std::vector` เองก็ไม่ตรวจสอบให้ผ่าน `operator[]` เช่นกัน แต่อย่าง
น้อยการจัดการ memory เองก็ถูกตัดออกไปหมดแล้ว)

### แนวทางเฉลยข้อ 5

```cpp
#include <iostream>
#include <string>
#include <utility>

// ---------- แบบเก่า: swap ผ่าน pointer ----------
template <typename T>
void old_style_swap(T* a, T* b) {
    T temp = *a;  // copy เต็มรูปแบบ
    *a = *b;      // copy เต็มรูปแบบอีกครั้ง
    *b = temp;    // copy เต็มรูปแบบอีกครั้ง (รวม 3 ครั้ง)
}

// ---------- แบบใหม่: swap ผ่าน reference + std::move ----------
template <typename T>
void modern_swap(T& a, T& b) {
    T temp = std::move(a);  // ย้าย ไม่ copy
    a = std::move(b);       // ย้าย ไม่ copy
    b = std::move(temp);    // ย้าย ไม่ copy
}

int main() {
    int x = 1;
    int y = 2;
    old_style_swap(&x, &y);
    std::cout << "old_style_swap: x=" << x << " y=" << y << "\n";

    std::string s1 = "hello";
    std::string s2 = "world";
    modern_swap(s1, s2);
    std::cout << "modern_swap: s1=" << s1 << " s2=" << s2 << "\n";

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++20 ex5.cpp -o ex5 && ./ex5
```

ผลลัพธ์:

```
old_style_swap: x=2 y=1
modern_swap: s1=world s2=hello
```

**อธิบาย**: สำหรับ `int` ทั้งสองแบบแทบไม่ต่างกันด้าน Performance (copy เลขจำนวนเต็มถูกมาก) แต่ถ้า
`T` เป็น `std::string` ขนาดใหญ่หรือ `std::vector` ที่มีข้อมูลนับล้าน element `old_style_swap` แบบ
pointer ธรรมดา (ถ้าเขียนด้วย `T` ไม่ใช่ pointer ตั้งแต่แรก) จะต้อง **copy ข้อมูลทั้งหมดถึง 3 ครั้ง**
ในขณะที่ `modern_swap` ใช้ `std::move` ทำให้เป็นแค่การ **สลับ pointer ภายในของ string/vector**
เท่านั้น (ย้าย ownership ของ heap memory แทนที่จะ copy เนื้อหาจริง) เร็วกว่ามากและใช้ memory ชั่วคราว
น้อยกว่ามาก นี่คือเหตุผลที่ `std::swap` ใน Standard Library เองก็ implement ด้วยหลักการเดียวกับ
`modern_swap` ทุกประการ (และควรใช้ `std::swap` ที่มีให้แล้วในโค้ดจริง แทนที่จะเขียนเอง)

---

## สรุปท้ายบท

ใน Part นี้ซึ่งเป็น Part สุดท้ายของ **Module F — Modern C++ (C++11 ถึง C++23)** เราได้:

- ทำความรู้จัก **C++ Core Guidelines** เอกสารอ้างอิงอย่างเป็นทางการที่ Bjarne Stroustrup และ
  Herb Sutter ร่วมกันวางไว้ พร้อมหลักปรัชญาแกนกลาง (Type Safety และ Resource Safety)
- ย้ำหลักการ **RAII** และ **Rule of Zero/Three/Five** ในฐานะรากฐานของการจัดการทรัพยากรทั้งหมดใน
  C++ สมัยใหม่
- สรุปรายการ **"Prefer X over Y" 18 ข้อ** ที่สำคัญที่สุด ครอบคลุมตั้งแต่ Resource Management,
  Type & Declaration, Container & Algorithm, ไปจนถึง Function & Class Design — แต่ละข้ออ้างอิงกลับ
  ไปยัง Part ที่เกี่ยวข้องตลอด Module D, E, F ของหลักสูตรนี้
- รู้จักเครื่องมือที่ช่วยบังคับใช้ Guideline โดยอัตโนมัติ (clang-tidy, GSL, Sanitizer, Compiler
  Explorer) พร้อมรู้ว่าจะเจาะลึกแต่ละตัวใน Part ไหนต่อไป
- Refactor โค้ดจริงจากสไตล์เก่า (C-with-Classes, ~85 บรรทัด) ให้กลายเป็น Modern C++ (~15 บรรทัด)
  พร้อมเปรียบเทียบข้อดีทุกด้านอย่างเป็นรูปธรรม
- ได้ **Checklist สำหรับ Code Review** ที่นำไปใช้ตรวจสอบโค้ดจริงได้ทันที ทั้งของตัวเองและของเพื่อน
  ร่วมทีม

Module F ที่เราเรียนมาตั้งแต่ Part 69 (ภาพรวม C++11) จนถึง Part นี้ ได้ปูพื้นฐานฟีเจอร์และปรัชญาของ
Modern C++ ไว้ครบถ้วนสมบูรณ์แล้ว เราเข้าใจทั้ง "ฟีเจอร์คืออะไร" และ "ทำไมถึงควรใช้" ซึ่งเป็นความรู้
ที่จำเป็นอย่างยิ่งก่อนจะก้าวเข้าสู่หัวข้อที่ท้าทายที่สุดหัวข้อหนึ่งของวิศวกรรม C++ นั่นคือ
**Concurrency และ Performance Engineering** ใน **Module G**

ใน **Part 81** เราจะเริ่มต้น Module G ด้วย **`std::thread` และ Concurrency ใน Modern C++** —
วิธีเขียนโปรแกรมที่ทำงานหลายอย่างพร้อมกันอย่างถูกต้องและปลอดภัย ซึ่งจะนำหลักการ RAII และ
Resource Management ที่เราเพิ่งย้ำใน Part นี้ไปประยุกต์ใช้กับทรัพยากรที่ซับซ้อนที่สุดอย่างหนึ่งคือ
**Thread**

**ต่อไป:** [Part 81 — std::thread และ Concurrency](./part-081-std-thread.md)
