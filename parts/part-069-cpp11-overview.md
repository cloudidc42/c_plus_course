# Part 69: ภาพรวม C++11 (Step 545–552)

> Module F — Modern C++ (C++11 ถึง C++23) | Part 69 จาก 125
> Step ที่ครอบคลุมใน Part นี้: Step 545–552
> Part ก่อนหน้า: [Part 68 — RAII และ Resource Management Pattern ระดับ Production](./part-068-raii-patterns.md) | Part ถัดไป: [Part 70 — Move Semantics และ Rvalue Reference](./part-070-move-semantics.md)

## เป้าหมายการเรียนรู้ (Learning Objectives)

หลังจบ Part นี้ ผู้เรียนจะสามารถ:

1. อธิบายบริบททางประวัติศาสตร์ของ C++11 ได้ว่าทำไมมันถึงถูกเรียกว่า "C++0x" มานานกว่าทศวรรษ
   และทำไมนักพัฒนา C++ ยุคนั้นถึงมองว่ามันคือการเปลี่ยนแปลงครั้งใหญ่ที่สุดของภาษาตั้งแต่ถือกำเนิด
2. ใช้ `auto` เพื่อให้ compiler อนุมานชนิดข้อมูลแทนการเขียน type ยาวๆ เอง พร้อมรู้กฎการอนุมานชนิด
   ที่แท้จริง (ทิ้ง `const`/reference โดยอัตโนมัติ) และระบุได้ว่าเมื่อไหร่ควร/ไม่ควรใช้ `auto`
3. ใช้ `decltype` เพื่อดึงชนิดข้อมูลของนิพจน์ใดๆ มาใช้ประกาศตัวแปรหรือกำหนด return type และ
   อธิบายความแตกต่างสำคัญระหว่าง `auto` กับ `decltype` ได้อย่างชัดเจน
4. สรุปรวมกลไกเบื้องหลังของ **range-based for loop** ที่เคยใช้งานมาแล้วตั้งแต่ Part 62 ให้เข้าใจ
   อย่างเป็นทางการว่า compiler แปลงมันเป็นโค้ดแบบไหนกันแน่
5. อธิบายได้ว่า `nullptr` แก้ปัญหาอะไรของ `NULL`/`0` ที่มีมาตั้งแต่ยุค C และใช้ `nullptr` แทน
   `NULL` ในโค้ด C++ ทุกจุดตั้งแต่นี้เป็นต้นไป
6. ใช้ `enum class` (strongly-typed enum) แทน `enum` ธรรมดา และอธิบายได้ว่ามันแก้ปัญหาการรั่วไหล
   ของชื่อค่า (name leakage) และการแปลงชนิดข้อมูลแบบไม่ตั้งใจได้อย่างไร
7. ใช้ `static_assert` ตรวจสอบเงื่อนไขที่ compile-time และใช้ **uniform initialization** (`{}`)
   เพื่อเขียนโค้ดการกำหนดค่าเริ่มต้นที่สม่ำเสมอและปลอดภัยจาก narrowing conversion
8. เชื่อมโยงฟีเจอร์ทั้งหมดของ C++11 ในบทนี้เข้ากับโค้ดที่เคยเขียนมาตลอดหลักสูตร และตระหนักว่า
   หลายฟีเจอร์เหล่านี้ถูกใช้งานมาแล้วโดยไม่รู้ตัวตั้งแต่ Module D

---

## 69.1 บริบททางประวัติศาสตร์: ทำไม C++11 ถึงเรียกว่า "C++0x" (Step 545)

ภาษา C++ มาตรฐานฉบับก่อนหน้า C++11 คือ **C++98** (ปรับปรุงเล็กน้อยเป็น **C++03**) ซึ่งออกมา
ตั้งแต่ปี 1998 หลังจากนั้นวงการ C++ เข้าสู่ช่วง "ความเงียบยาวนาน" กว่า 13 ปีที่ไม่มีมาตรฐานใหม่
ออกมาเลย ในขณะที่ภาษาคู่แข่งอย่าง Java และ C# มีการอัปเดตฟีเจอร์ใหม่ๆ ออกมาต่อเนื่องเกือบทุกปี

คณะกรรมการมาตรฐาน ISO C++ (WG21) เริ่มพัฒนามาตรฐานฉบับถัดไปตั้งแต่ต้นยุค 2000s และตั้งชื่อ
โครงการชั่วคราวว่า **"C++0x"** โดยตัว `x` หมายถึง "เลขหลักหน่วยของปีที่จะออก" ซึ่งตอนตั้งชื่อ
คาดหวังกันว่าจะออกทันภายในทศวรรษ 2000s (เช่น 2008, 2009) แต่กระบวนการมาตรฐานที่ต้องผ่านการ
ตรวจสอบอย่างละเอียด การถกเถียงเรื่อง backward compatibility (โค้ด C++98 นับล้านบรรทัดทั่วโลก
ต้องยังคอมไพล์ผ่านได้) และการออกแบบฟีเจอร์ใหม่ที่ซับซ้อนมาก (เช่น move semantics ที่ต้องแก้ไข
กฎของภาษาในระดับรากฐาน) ทำให้กระบวนการยืดเยื้อออกไปเรื่อยๆ จนชื่อเล่น "C++0x" กลายเป็นมุกตลก
ในวงการเวลานั้น เพราะ `x` ดันกลายเป็น hexadecimal digit ที่มีค่ามากกว่า 9 ไปแล้ว (บางคนติดตลก
ว่า `x` ต้องเป็น `0xA` ปี 2010 หรือ `0xB` ปี 2011 กันแน่)

ในที่สุดมาตรฐานก็ได้รับการอนุมัติอย่างเป็นทางการในปี **2011** และถูกเปลี่ยนชื่อเป็น **C++11**
ตามธรรมเนียมการตั้งชื่อของ ISO (ปีที่ประกาศใช้จริง)

### ทำไม C++11 ถึงถูกเรียกว่าเป็นการเปลี่ยนแปลงครั้งใหญ่ที่สุดของภาษา

นักพัฒนา C++ รุ่นเก๋าจำนวนมากถึงกับเรียก C++11 ว่าเป็น **"the second C++"** (ภาษา C++ ตัวที่สอง)
เพราะขนาดและความลึกของการเปลี่ยนแปลงที่มันนำมา ไม่ใช่แค่การเพิ่มฟังก์ชันไลบรารีใหม่ๆ แต่เป็นการ
เปลี่ยนแปลง **กฎพื้นฐานของภาษาระดับ core language** ที่ส่งผลต่อวิธีคิดในการออกแบบโค้ดทั้งหมด:

| ด้านที่เปลี่ยนแปลง | ตัวอย่างฟีเจอร์ | ผลกระทบ |
|---|---|---|
| Type system | `auto`, `decltype` | ลดการเขียน type ซ้ำซ้อน ทำให้ template code อ่านง่ายขึ้นมาก |
| Object lifetime | rvalue reference, move semantics (Part 70) | เปลี่ยนวิธีคิดเรื่องการส่งต่อทรัพยากรทั้งหมด ประสิทธิภาพดีขึ้นแบบก้าวกระโดด |
| Concurrency | `std::thread`, `std::mutex`, `std::atomic` (Module G) | ครั้งแรกที่ C++ มี memory model และ threading library เป็นส่วนหนึ่งของมาตรฐานเอง |
| Syntax ที่ปลอดภัยขึ้น | `nullptr`, `enum class`, `static_assert` | ปิดช่องโหว่ทาง type-safety ที่มีมาตั้งแต่ยุค C |
| Library ใหม่จำนวนมาก | `unique_ptr`, `shared_ptr` (Part 67), lambda (Part 64), `unordered_map` (Part 61) | สิ่งที่ต้องเขียนเองหรือใช้ library ภายนอกมาก่อน กลายเป็นส่วนหนึ่งของ standard library |

สิ่งที่น่าสนใจคือ **หลายฟีเจอร์ที่จะเรียนใน Module F นี้ ผู้เรียนได้ใช้งานมาแล้วตลอดหลักสูตร
โดยไม่รู้ตัว** — `auto` ในทุก range-based for loop ตั้งแต่ Part 62, `nullptr` ในการเช็ค pointer
ตั้งแต่ Part 67, `unique_ptr`/`shared_ptr` ทั้ง Part 67 ล้วนเป็นฟีเจอร์ของ C++11 ทั้งสิ้น Part นี้
และอีกสาม Part ถัดไป (70, 71) จะทำให้ความรู้ที่ใช้แบบสัญชาตญาณเหล่านั้นกลายเป็นความเข้าใจอย่าง
เป็นทางการและลึกซึ้ง

### มาตรฐาน C++ หลังจาก C++11

หลังจาก C++11 ออกมา คณะกรรมการมาตรฐานตัดสินใจเปลี่ยนจังหวะการออกมาตรฐานใหม่ จากที่เคยทิ้งช่วง
นานเป็นทศวรรษ มาเป็น **ออกทุก 3 ปี** (three-year release cycle) เพื่อไม่ให้เกิดปัญหาแบบเดิมอีก:

| มาตรฐาน | ปีที่ออก | ธีมหลัก |
|---|---|---|
| C++98/03 | 1998/2003 | มาตรฐานแรก, templates, STL |
| **C++11** | **2011** | **การเปลี่ยนแปลงครั้งใหญ่ที่สุด**: auto, move semantics, lambda, threading |
| C++14 | 2014 | ฟีเจอร์เสริมและแก้ไขข้อบกพร่องของ C++11 (Part 72) |
| C++17 | 2017 | structured bindings, `if constexpr`, `std::optional`/`variant` (Part 72) |
| C++20 | 2020 | concepts, ranges, coroutines, modules (Part 73-76) |
| C++23 | 2023/2024 | ปรับปรุงเพิ่มเติม (Part 77) |

ตลอด Module F นี้เราจะไล่เรียงตามลำดับเวลานี้ เริ่มจาก C++11 (Part 69-70-71) ก่อน แล้วค่อยไป
C++14/17 (Part 72) และ C++20/23 (Part 73-77) ตามลำดับ

---

## 69.2 auto: ให้ Compiler อนุมานชนิดข้อมูลแทนเรา (Step 546)

ก่อน C++11 ทุกตัวแปรต้องระบุชนิดข้อมูลตรงๆ เสมอ ซึ่งไม่มีปัญหาอะไรกับชนิดง่ายๆ อย่าง `int` หรือ
`double` แต่กลายเป็นภาระหนักมากเมื่อทำงานกับ template และ STL ที่ชนิดข้อมูลมักซับซ้อนและยาวมาก
(เช่น `std::map<std::string, std::vector<int>>::iterator`)

`auto` แก้ปัญหานี้โดยบอกให้ compiler **อนุมานชนิดข้อมูลจากค่าเริ่มต้นที่กำหนดให้** โดยอัตโนมัติ
ข้อควรเข้าใจให้ชัดคือ `auto` ไม่ใช่ "ชนิดข้อมูลแบบไดนามิก" เหมือนใน Python/JavaScript — มันยังคง
เป็น **static typing** เหมือนเดิมทุกประการ เพียงแต่ compiler เป็นคนเขียน type ให้เราแทนที่จะต้อง
พิมพ์เอง ชนิดข้อมูลที่ได้ยังคงถูกกำหนดตายตัวตั้งแต่ compile-time เหมือน `int`/`double` ทุกประการ

```cpp
// 01_auto_basics.cpp
#include <iostream>
#include <map>
#include <string>
#include <vector>

int main() {
    auto i = 42;                       // int
    auto d = 3.14;                     // double
    auto s = std::string("hello");     // std::string
    auto vec = std::vector<int>{1, 2, 3, 4, 5};

    std::cout << "i = " << i << ", d = " << d << ", s = " << s << '\n';
    std::cout << "vec.size() = " << vec.size() << '\n';

    std::map<std::string, std::vector<int>> scoreBoard;
    scoreBoard["Somchai"] = {80, 90, 100};
    scoreBoard["Somsri"] = {70, 60};

    // ก่อน C++11: ต้องเขียนชนิดของ iterator แบบเต็มยาวเป๊ะ
    for (std::map<std::string, std::vector<int>>::iterator it = scoreBoard.begin();
         it != scoreBoard.end(); ++it) {
        std::cout << "[แบบเก่า] " << it->first << '\n';
    }

    // C++11: ให้ compiler อนุมานชนิดของ iterator แทน
    for (auto it = scoreBoard.begin(); it != scoreBoard.end(); ++it) {
        std::cout << "[auto]   " << it->first << '\n';
    }

    for (auto& kv : scoreBoard) {
        std::cout << kv.first << " -> ";
        for (auto val : kv.second) {
            std::cout << val << ' ';
        }
        std::cout << '\n';
    }

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 01_auto_basics.cpp -o 01_auto
./01_auto
```

ผลลัพธ์:

```
i = 42, d = 3.14, s = hello
vec.size() = 5
[แบบเก่า] Somchai
[แบบเก่า] Somsri
[auto]   Somchai
[auto]   Somsri
Somchai -> 80 90 100 
Somsri -> 70 60
```

สังเกตว่าสองบล็อกของ for loop (แบบเก่ากับ `auto`) ให้ผลลัพธ์เหมือนกันทุกประการ — `auto` **ไม่ได้
เปลี่ยนพฤติกรรมของโปรแกรม** เลยแม้แต่นิดเดียว มันแค่ทำให้ compiler เขียนชนิดข้อมูลที่ถูกต้องแทน
เราเท่านั้น ประโยชน์ที่ได้คือความอ่านง่ายและลดโอกาสพิมพ์ชนิดข้อมูลผิด โดยเฉพาะเมื่อทำงานกับ
nested template type ที่ซับซ้อนแบบในตัวอย่างนี้

### กฎการอนุมานชนิดข้อมูลของ auto

กฎของ `auto` ยึดตามกฎเดียวกับการอนุมานชนิดข้อมูลของ template parameter (จะเจาะลึกเต็มรูปแบบใน
Part 78) แต่สรุปสั้นๆ ที่ต้องจำให้ขึ้นใจตอนนี้คือ:

> **`auto` ทิ้ง `const` และ reference โดยอัตโนมัติเสมอ** เว้นแต่จะเขียน `const auto&`, `auto&`,
> หรือ `auto&&` ระบุไว้ชัดเจนด้วยตัวเอง

```cpp
// 02_auto_pitfalls.cpp
#include <iostream>
#include <vector>

int main() {
    const std::vector<int> numbers = {10, 20, 30};

    // (1) auto ทิ้ง const และ reference โดยอัตโนมัติเสมอ
    for (auto n : numbers) {          // n เป็น int ธรรมดา (สำเนา) ไม่ใช่ const int&
        n += 1;                       // แก้ค่าสำเนาได้ ไม่กระทบ numbers ต้นฉบับ
        std::cout << n << ' ';
    }
    std::cout << '\n';

    for (const auto& n : numbers) {   // ต้องเติม const และ & เองเสมอถ้าต้องการ reference แบบอ่านอย่างเดียว
        std::cout << n << ' ';
    }
    std::cout << '\n';

    // (2) auto กับ vector<bool>: กับดักคลาสสิก เพราะ vector<bool> ไม่คืน bool& จริง
    // แต่คืน std::vector<bool>::reference ซึ่งเป็น proxy object
    std::vector<bool> flags = {true, false, true};
    auto proxy = flags[0];   // ชนิดจริงคือ std::vector<bool>::reference ไม่ใช่ bool!
    flags[0] = false;        // แก้ค่าใน flags แล้ว proxy (ที่อ้างอิงตำแหน่งเดียวกัน) ก็เปลี่ยนตาม
    std::cout << "proxy (หลังแก้ flags[0]) = " << std::boolalpha << proxy << '\n';

    bool realCopy = flags[0]; // ใช้ bool ตรงๆ เมื่อรู้ว่าต้องการค่าจริงแบบสำเนา ไม่ใช่ proxy
    std::cout << "realCopy = " << realCopy << '\n';

    return 0;
}
```

```
11 21 31 
10 20 30 
proxy (หลังแก้ flags[0]) = false
realCopy = false
```

ตัวอย่างที่สองแสดงกับดักคลาสสิกที่แม้แต่โปรแกรมเมอร์ C++ ที่มีประสบการณ์ก็เคยพลาด:
`std::vector<bool>` เป็น specialization พิเศษที่เก็บข้อมูลแบบ **bit-packed** (1 bit ต่อ 1
`bool` แทนที่จะเป็น 1 byte ตามปกติ) เพื่อประหยัดหน่วยความจำ ผลคือ `operator[]` ของมัน **ไม่
สามารถคืน `bool&` จริงได้** (ไม่มี byte ที่ addressable ให้ชี้) จึงต้องคืน **proxy object** ชนิด
`std::vector<bool>::reference` แทน เมื่อใช้ `auto` รับค่านี้มา จะได้ proxy ที่ยังคงอ้างอิงตำแหน่ง
เดิมใน `flags` อยู่ ไม่ใช่ค่า `bool` ที่ copy ออกมาแล้วอย่างที่คาดหวัง

### ตารางสรุป: auto รูปแบบต่างๆ

| รูปแบบ | ความหมาย | ใช้เมื่อ |
|---|---|---|
| `auto x = expr` | copy ค่า, ทิ้ง `const`/reference | ต้องการสำเนาอิสระของค่า |
| `auto& x = expr` | reference, เขียนแก้ค่าต้นฉบับได้ | ต้องการแก้ไข object ต้นฉบับผ่าน x |
| `const auto& x = expr` | reference แบบอ่านอย่างเดียว | ต้องการหลีกเลี่ยงการ copy แต่ไม่ต้องการแก้ไข (ใช้บ่อยที่สุดใน range-based for) |
| `auto&& x = expr` | forwarding reference (Part 70.8) | ใช้ใน template หรือ generic lambda ที่ต้องรับได้ทั้ง lvalue/rvalue |

---

## 69.3 auto: เมื่อไหร่ควรใช้ และเมื่อไหร่ไม่ควรใช้ (Step 547)

`auto` เป็นเครื่องมือที่ทรงพลัง แต่เหมือนเครื่องมือทุกชนิดในโปรแกรมมิ่ง มันมีทั้งสถานการณ์ที่
เหมาะสมและไม่เหมาะสม การใช้พร่ำเพรื่อโดยไม่คิดสามารถทำให้โค้ดอ่านยากขึ้นแทนที่จะง่ายขึ้น

### ควรใช้ auto เมื่อ

1. **ชนิดข้อมูลยาวและซับซ้อน** โดยเฉพาะ iterator ของ container ซ้อนกันหลายชั้น หรือ return type
   ของ lambda/template function ที่บางครั้งไม่มีทางเขียนชื่อ type ได้เลยด้วยซ้ำ (จะเห็นชัดเจนใน
   Part 78 เรื่อง template metaprogramming)
2. **ชนิดข้อมูลชัดเจนอยู่แล้วจากด้านขวาของนิพจน์** เช่น `auto ptr = std::make_unique<Widget>();`
   — การเขียน `std::unique_ptr<Widget> ptr = std::make_unique<Widget>();` คือการเขียนคำว่า
   `Widget` ซ้ำสองครั้งโดยไม่จำเป็น
3. **ป้องกันการเขียน type ผิดพลาดจากการ copy ที่ไม่ตั้งใจ** เช่น การวนลูปด้วย `int` ทั้งที่ขนาด
   ของ container จริงๆ เป็น `std::size_t` (unsigned) ทำให้เกิดคำเตือนเรื่อง sign comparison
   หรือแย่กว่านั้นคือ undefined behavior จาก integer overflow ในบางกรณี
4. **range-based for loop แทบทุกกรณี** — เขียน `auto`, `auto&`, หรือ `const auto&` เกือบเสมอ

### ไม่ควรใช้ (หรือควรระวังเป็นพิเศษ) เมื่อ

1. **ชนิดข้อมูลไม่ชัดเจนจากบริบท** เช่น `auto result = compute();` ถ้าไม่เห็น signature ของ
   `compute()` เลย ผู้อ่านโค้ดจะไม่รู้เลยว่า `result` เป็นชนิดอะไร ทำให้ต้องเปิดไฟล์อื่นตามหา
   ในกรณีแบบนี้การเขียนชนิดข้อมูลตรงๆ (เช่น `double result = compute();`) ช่วยให้อ่านโค้ดง่ายกว่า
2. **ต้องการชนิดข้อมูลที่เฉพาะเจาะจง ไม่ใช่ชนิดที่ compiler อนุมานให้โดยธรรมชาติ** เช่นต้องการ
   `double` แต่ initializer เป็น `int` (`auto x = 5;` จะได้ `int` ไม่ใช่ `double`) ต้องเขียน
   `double x = 5;` หรือ `auto x = 5.0;` ให้ชัดเจนแทน
3. **การประกาศ interface สาธารณะของ class/function** (เช่น return type ของ public method หรือ
   ชนิดของ parameter) ควรเขียนชนิดข้อมูลตรงๆ เสมอ เพื่อให้ผู้ใช้ library เห็น contract ที่ชัดเจน
   จาก signature โดยไม่ต้องไปเปิดดู implementation
4. **กับดักของ `vector<bool>` และ proxy type อื่นๆ** ตามที่แสดงในหัวข้อก่อนหน้า

> **กฎทองที่ใช้ได้จริง**: ใช้ `auto` เมื่อมันทำให้โค้ด **อ่านง่ายขึ้น** (ลดความซ้ำซ้อน, ลด
> โอกาสพิมพ์ type ผิด) หลีกเลี่ยงเมื่อมันทำให้โค้ด **อ่านยากขึ้น** (ซ่อนข้อมูลสำคัญที่ผู้อ่าน
> ควรเห็นทันทีโดยไม่ต้องเดา) — นี่ไม่ใช่กฎที่ตายตัว 100% แต่เป็นหลักการที่ทีมพัฒนา C++
> ระดับ production ส่วนใหญ่ยึดถือ (จะเจาะลึกเพิ่มเติมใน Part 80 เรื่อง Core Guidelines)

---

## 69.4 decltype: ดึงชนิดข้อมูลจากนิพจน์ใดๆ ก็ได้ (Step 548)

`decltype` เป็นอีกฟีเจอร์ของ C++11 ที่ทำงานคู่กับ `auto` แต่มีเป้าหมายต่างกัน: `auto` อนุมาน
ชนิดข้อมูล **จากค่าที่กำหนดให้** ในขณะที่ `decltype` ดึงชนิดข้อมูล **ของนิพจน์ (expression)
ใดๆ ก็ได้** โดยไม่ต้องประเมินค่าของนิพจน์นั้นจริงๆ (compiler แค่ "มองดู" ชนิดของมันเท่านั้น)

ความแตกต่างที่สำคัญที่สุดคือ **`decltype` รักษา reference และ `const` ไว้ตามชนิดที่ประกาศจริง
เสมอ** ต่างจาก `auto` ที่ทิ้งทั้งสองอย่างนี้โดยอัตโนมัติ

```cpp
// 03_decltype.cpp
#include <iostream>
#include <type_traits>
#include <vector>

int add(int a, int b) { return a + b; }

template <typename T, typename U>
auto multiply(T a, U b) -> decltype(a * b) {   // trailing return type: ต้องใช้ก่อน C++14
    return a * b;
}

int main() {
    int x = 10;
    int& refX = x;
    const int cx = 20;

    decltype(x) a = 1;        // int (decltype ของตัวแปรธรรมดา = ชนิดที่ประกาศไว้)
    decltype(refX) b = x;     // int& (decltype รักษา reference ไว้ ต่างจาก auto)
    decltype(cx) c = 5;       // const int (decltype รักษา const ไว้ด้วย)
    decltype(add(1, 2)) d = 99; // int (decltype ของนิพจน์เรียกฟังก์ชัน = ชนิดค่าที่ฟังก์ชันคืน)

    std::cout << "a=" << a << " b=" << b << " c=" << c << " d=" << d << '\n';

    static_assert(std::is_same_v<decltype(x), int>, "x ควรเป็น int");
    static_assert(std::is_same_v<decltype((x)), int&>, "decltype((x)) ที่มีวงเล็บคู่ จะกลายเป็น int&");

    std::vector<int> v = {1, 2, 3};
    auto byAuto = multiply(2, 3.5);      // auto: ชนิดถูกอนุมานจาก return statement จริง (ต้อง C++14 ขึ้นไปถ้าไม่ใช้ trailing return)
    decltype(multiply(2, 3.5)) byDecltype = multiply(2, 3.5);

    std::cout << "byAuto=" << byAuto << " byDecltype=" << byDecltype << '\n';
    std::cout << "v.size()=" << v.size() << '\n';

    return 0;
}
```

```
a=1 b=10 c=5 d=99
byAuto=7 byDecltype=7
v.size()=3
```

จุดที่น่าสนใจและมักทำให้สับสนคือ `decltype(x)` กับ `decltype((x))` (มีวงเล็บคู่) **ให้ผลลัพธ์
ต่างกัน**: `decltype(x)` มองที่ **ตัวแปร `x` ที่ถูกประกาศไว้** จึงได้ชนิดตามที่ประกาศตรงๆ
(`int`) แต่ `decltype((x))` มองที่ **นิพจน์ `(x)`** ซึ่งเป็น lvalue expression (จะอธิบายเรื่อง
lvalue/rvalue อย่างละเอียดใน Part 70.1) ทำให้ได้ผลลัพธ์เป็น `int&` เสมอ — นี่คือกับดักที่ทำให้
แม้แต่ผู้เชี่ยวชาญ C++ พลาดได้ง่ายๆ ถ้าไม่ระวังเรื่องวงเล็บ

### ตารางเปรียบเทียบ auto กับ decltype

| คุณสมบัติ | `auto` | `decltype` |
|---|---|---|
| อนุมานจาก | ค่าที่กำหนดให้ตัวแปร (initializer) | ชนิดของนิพจน์ใดๆ โดยไม่ต้องประเมินค่าจริง |
| ทิ้ง `const`/reference | ทิ้งเสมอ (ต้องเติม `&`/`const` เอง) | **รักษาไว้เสมอ** ตามชนิดที่แท้จริงของนิพจน์ |
| ต้องมี initializer | ต้องมี (compiler อนุมานจากค่าที่ให้) | ไม่ต้องมี (แค่ต้องมีนิพจน์ให้ตรวจสอบชนิด) |
| ใช้บ่อยที่สุดกับ | การประกาศตัวแปรทั่วไป, range-based for | Generic programming, trailing return type, template metaprogramming (Part 78) |

> **หมายเหตุสำหรับอนาคต**: C++14 เพิ่ม `decltype(auto)` ที่รวมพฤติกรรมของทั้งสองเข้าด้วยกัน
> (อนุมานจาก initializer เหมือน `auto` แต่รักษา `const`/reference เหมือน `decltype`) จะพูดถึง
> อย่างละเอียดใน **Part 72** เมื่อเข้าสู่เรื่องฟีเจอร์ C++14

---

## 69.5 Range-based For Loop: สรุปรวมกลไกเบื้องหลัง (Step 549)

Part 62 ได้แนะนำการใช้งาน range-based for loop ไปแล้วในบริบทของ STL iterator แต่ในหัวข้อนี้
เราจะสรุปรวมกลไกทั้งหมดอย่างเป็นทางการ เพราะมันเป็นหนึ่งในฟีเจอร์ที่ใช้งานบ่อยที่สุดของ C++11

```cpp
// 04_range_for.cpp
#include <iostream>
#include <vector>

struct Rectangle {
    double width;
    double height;
};

int main() {
    std::vector<int> nums = {1, 2, 3, 4, 5};

    // รูปแบบที่เขียนทุกวัน (Part 62 เคยใช้งานมาแล้ว)
    for (int n : nums) {
        std::cout << n << ' ';
    }
    std::cout << '\n';

    // สิ่งที่ compiler แปลงให้เบื้องหลัง (เทียบเท่ากันทุกประการ)
    {
        auto&& range = nums;
        auto it = range.begin();
        auto end = range.end();
        for (; it != end; ++it) {
            int n = *it;
            std::cout << n << ' ';
        }
        std::cout << '\n';
    }

    // ใช้ auto& เพื่อแก้ไข element จริงในภาชนะ (ไม่ copy)
    for (auto& n : nums) {
        n *= 10;
    }
    for (auto n : nums) {
        std::cout << n << ' ';
    }
    std::cout << '\n';

    // range-based for ใช้ได้กับทุกชนิดที่มี begin()/end() รวมถึง struct/class ของเราเอง
    std::vector<Rectangle> rects = {{2.0, 3.0}, {4.0, 5.0}};
    for (const auto& r : rects) {
        std::cout << "area = " << (r.width * r.height) << '\n';
    }

    // และใช้กับ raw array ได้ด้วย (compiler รู้ขนาด array ที่ compile-time)
    int rawArr[] = {7, 8, 9};
    for (auto n : rawArr) {
        std::cout << n << ' ';
    }
    std::cout << '\n';

    return 0;
}
```

```
1 2 3 4 5 
1 2 3 4 5 
10 20 30 40 50 
area = 6
area = 20
7 8 9
```

### กลไกเบื้องหลังอย่างเป็นทางการ

เมื่อ compiler เจอ `for (auto& elem : container) { ... }` มันจะแปลง (desugar) ให้เทียบเท่ากับ
โค้ดต่อไปนี้เสมอ (มาตรฐานภาษากำหนดไว้ตายตัว):

```cpp
{
    auto&& __range = container;              // เก็บ reference ไปยัง container (ไม่ copy)
    auto __begin = __range.begin();          // หรือ begin(__range) ถ้าเป็น raw array/มี free function begin()
    auto __end = __range.end();
    for (; __begin != __end; ++__begin) {
        auto& elem = *__begin;               // ชนิดของ elem ตรงตามที่เราเขียนไว้ (auto, auto&, const auto&)
        { ... }                              // เนื้อ loop body ที่เราเขียน
    }
}
```

จุดสำคัญที่ต้องเข้าใจ:

1. **`container` ถูกประเมินแค่ครั้งเดียว** ตอนเริ่ม loop และผูกกับ `__range` (เขียนเป็น `auto&&`
   เพื่อรองรับได้ทั้ง lvalue container และ rvalue container ชั่วคราว) — ถ้า `container` เป็น
   นิพจน์ที่มี side effect (เช่นเรียกฟังก์ชันที่คืน container ใหม่ทุกครั้ง) นิพจน์นั้นจะถูกเรียก
   แค่ครั้งเดียวเท่านั้น ไม่ใช่ทุกรอบของ loop
2. **ต้องมี `begin()`/`end()`** ไม่ว่าจะเป็น member function ของ container เอง (`vector`,
   `map`, `string`, ฯลฯ) หรือ free function `begin(x)`/`end(x)` ที่หาเจอผ่าน ADL (Argument-
   Dependent Lookup) — นี่คือเหตุผลที่ raw array ใช้งานได้ด้วย (มี free function `begin`/`end`
   สำหรับ array ให้อยู่แล้วใน `<iterator>`)
3. **ชนิดของ `elem` ขึ้นอยู่กับสิ่งที่เราเขียนไว้เอง** — `auto` (copy), `auto&` (แก้ไขได้),
   `const auto&` (อ่านอย่างเดียว ไม่ copy) — เหมือนกับการใช้ `auto` ในบริบทอื่นๆ ทุกประการ
   ตามกฎในหัวข้อ 69.2

---

## 69.6 nullptr: แก้ปัญหาที่ NULL/0 ทิ้งไว้ตั้งแต่ยุค C (Step 550)

ในภาษา C และ C++98/03 การเช็คว่า pointer เป็น "ไม่มีค่า" ใช้ macro `NULL` ซึ่งในทางเทคนิคแล้ว
มักถูก define ไว้เป็นแค่ `0` หรือ `((void*)0)` เท่านั้น — พูดง่ายๆ คือ **`NULL` ไม่ใช่ชนิดข้อมูล
พิเศษ มันเป็นแค่ตัวเลข 0 ที่ปลอมตัวมา** ปัญหานี้สร้างความกำกวมในการเลือก overload ของฟังก์ชัน

```cpp
// 05_nullptr.cpp
#include <iostream>

void handle(int value) {
    std::cout << "handle(int) called with " << value << '\n';
}

void handle(const char* text) {
    std::cout << "handle(const char*) called with "
              << (text ? text : "(null)") << '\n';
}

int main() {
    int* p1 = nullptr;         // ชัดเจนว่าเป็น null pointer ไม่ใช่ตัวเลข 0
    if (p1 == nullptr) {
        std::cout << "p1 เป็น null pointer\n";
    }

    // ปัญหาคลาสสิกของ NULL/0: เลือก overload ผิดเพราะ NULL มักเป็นแค่ 0 (int) ไม่ใช่ pointer
    handle(0);          // เรียก handle(int) เสมอ ไม่ว่าเจตนาจะเป็น pointer หรือไม่
    handle(nullptr);    // เรียก handle(const char*) เสมอ เพราะ nullptr มีชนิดของตัวเองคือ std::nullptr_t
                         // ซึ่งแปลงเป็น pointer type ใดก็ได้ แต่แปลงเป็นชนิดตัวเลขไม่ได้เลย

    std::cout << std::boolalpha;
    std::cout << "nullptr == 0 ในเชิงการเปรียบเทียบ pointer: "
              << (p1 == 0) << '\n'; // ยังเปรียบเทียบได้ แต่ nullptr ป้องกันการ "เลือก overload ผิด" ตอนส่งอาร์กิวเมนต์

    return 0;
}
```

```
p1 เป็น null pointer
handle(int) called with 0
handle(const char*) called with (null)
nullptr == 0 ในเชิงการเปรียบเทียบ pointer: true
```

สังเกตว่า `handle(0)` เรียก `handle(int)` เสมอ แม้ในใจเราอาจตั้งใจส่ง "null pointer" เข้าไปก็ตาม
เพราะ `0` มีชนิดเป็น `int` ตามตัวอักษรที่เขียน ไม่ใช่ pointer — นี่คือบั๊กเงียบที่เคยเกิดขึ้นจริง
ในโค้ด C++98/03 จำนวนมาก โดยเฉพาะเมื่อ overload ทั้งสองแบบมีอยู่จริงในโค้ดเบส (เช่น library ที่
รับได้ทั้งค่าตัวเลขและ pointer) โปรแกรมเมอร์อาจตั้งใจเรียก overload ที่รับ pointer แต่ compiler
กลับเลือก overload ที่รับตัวเลขให้แทนอย่างเงียบๆ โดยไม่มี warning ใดๆ เลย

C++11 แก้ปัญหานี้ด้วยการเพิ่มคีย์เวิร์ด `nullptr` ที่มีชนิดข้อมูลเป็นของตัวเองคือ
**`std::nullptr_t`** ซึ่งถูกออกแบบให้ **แปลง (convert) เป็น pointer type ใดก็ได้โดยปริยาย
(implicit conversion)** แต่ **ไม่สามารถแปลงเป็นชนิดตัวเลขใดๆ ได้เลย** ทำให้ `handle(nullptr)`
เลือก overload ที่รับ `const char*` ได้อย่างไม่กำกวมเสมอ

> **กฎทองของหลักสูตรนี้ตั้งแต่ Part นี้เป็นต้นไป**: ใช้ `nullptr` แทน `NULL` หรือ `0` เพื่อแทน
> "pointer ที่ไม่ชี้ไปที่อะไรเลย" ในโค้ด C++ ทุกจุด ไม่มีข้อยกเว้น (ยกเว้นตอนเขียนโค้ด C ล้วนใน
> Module A-C ที่ยังไม่มี `nullptr` ให้ใช้)

---

## 69.7 enum class: Strongly-Typed Enum (Step 551)

ทบทวนจาก **Part 10**: `enum` ธรรมดาในภาษา C/C++ มีปัญหาสำคัญสองข้อที่สร้างความรำคาญและบั๊กใน
โค้ดเบสขนาดใหญ่มาตลอด:

1. **ชื่อค่ารั่วไหลเข้า scope รอบข้าง (name leakage)**: ค่าคงที่ที่ประกาศใน `enum` จะกลายเป็น
   ชื่อในระดับเดียวกับ scope ที่ประกาศ `enum` นั้น (มักเป็น global หรือ namespace scope) ทำให้
   ถ้าประกาศ `enum` สองตัวที่มีชื่อค่าซ้ำกัน (เช่น `GREEN` ใน enum สี กับ `GREEN` ใน enum
   สัญญาณไฟจราจร) จะเกิด **compile error ทันที** เพราะชื่อชนกัน
2. **แปลงเป็น `int` ได้โดยอัตโนมัติ (implicit conversion)**: ทำให้เผลอนำค่าจาก `enum` คนละตัว
   มาเปรียบเทียบหรือคำนวณกันได้โดยไม่มี warning ใดๆ ทั้งที่ในทางความหมายแล้วไม่ควรเปรียบเทียบกัน
   เลย (เช่นเปรียบเทียบสีกับสถานะออเดอร์)

`enum class` (บางครั้งเรียก **scoped enum**) ที่เพิ่มเข้ามาใน C++11 แก้ปัญหาทั้งสองข้อนี้พร้อมกัน

```cpp
// 06_enum_class.cpp
#include <iostream>

// enum ธรรมดา (C-style, ทบทวน Part 10): ชื่อค่ารั่วไหลเข้าสู่ scope รอบข้างทั้งหมด
enum Color { RED, GREEN, BLUE };
enum TrafficLight { GREEN_LIGHT, YELLOW_LIGHT, RED_LIGHT };
// ถ้าเผลอตั้งชื่อ GREEN ซ้ำกันสอง enum จะ "ชนกัน" ทันทีตอน compile (ประกาศซ้ำในโครงสร้างเดียวกัน)

// enum class (C++11): strongly-typed, ชื่อค่าต้องอ้างผ่านชื่อ enum เสมอ ไม่รั่วไหล
enum class Direction { North, South, East, West };
enum class Suit { North, South, East, West }; // ชื่อซ้ำกับ Direction ได้สบายๆ เพราะอยู่คนละ scope

// enum class ยังกำหนด underlying type ได้ชัดเจน (ค่าเริ่มต้นคือ int แต่กำหนดเองได้)
enum class StatusCode : unsigned char { Ok = 0, NotFound = 4, ServerError = 5 };

int main() {
    Color c = RED;             // ใช้ชื่อ RED ตรงๆ ได้เลย (รั่วไหลเข้า global scope)
    if (c == RED) {
        std::cout << "c คือ RED (enum ธรรมดา)\n";
    }

    Direction d = Direction::North;   // ต้องระบุ Direction:: เสมอ ป้องกันชื่อชนกัน
    if (d == Direction::North) {
        std::cout << "d คือ Direction::North\n";
    }

    // enum class ไม่แปลงเป็น int โดยอัตโนมัติ (ต่างจาก enum ธรรมดา) ต้อง static_cast ชัดเจน
    int code = static_cast<int>(StatusCode::NotFound);
    std::cout << "StatusCode::NotFound as int = " << code << '\n';

    // enum ธรรมดา แปลงเป็น int ได้ทันทีโดยไม่ต้อง cast (จุดอ่อนที่ enum class แก้)
    int rawColor = RED;
    std::cout << "RED as int (implicit) = " << rawColor << '\n';

    return 0;
}
```

```
c คือ RED (enum ธรรมดา)
d คือ Direction::North
StatusCode::NotFound as int = 4
RED as int (implicit) = 0
```

สังเกตว่า `enum class Direction` และ `enum class Suit` ประกาศชื่อค่าซ้ำกันทุกตัว (`North`,
`South`, `East`, `West`) ได้อย่างสบายๆ โดยไม่มี compile error เลย เพราะชื่อเหล่านั้นถูก "ขัง"
ไว้ใน scope ของ enum แต่ละตัว ต้องเรียกผ่าน `Direction::North` หรือ `Suit::North` เสมอ ต่างจาก
`enum` ธรรมดาที่ถ้าลองประกาศ `GREEN` ซ้ำในสอง enum จะเกิด compile error ทันที (ลองแก้โค้ดข้างบน
เพิ่ม `enum AnotherColor { GREEN };` ดูจะเห็น error `redeclaration of 'GREEN'`)

### ตารางเปรียบเทียบ enum กับ enum class

| คุณสมบัติ | `enum` ธรรมดา | `enum class` |
|---|---|---|
| ชื่อค่ารั่วไหลเข้า scope รอบข้าง | ใช่ (มักก่อปัญหาชื่อชนกัน) | **ไม่** ต้องอ้างผ่าน `EnumName::Value` เสมอ |
| แปลงเป็น `int` โดยอัตโนมัติ | ใช่ (จุดอ่อนด้าน type-safety) | **ไม่** ต้อง `static_cast` ชัดเจนเสมอ |
| กำหนด underlying type เอง | ทำได้ (C++11 ขึ้นไป) เช่น `enum Color : char` | ทำได้เหมือนกัน และเป็นค่าเริ่มต้นที่แนะนำ |
| Forward declaration (ประกาศล่วงหน้าไม่ต้องนิยามค่า) | ทำได้เฉพาะกรณีระบุ underlying type | ทำได้เสมอ แม้ไม่ระบุ underlying type (default เป็น `int`) |

> **กฎทองของหลักสูตรนี้ตั้งแต่ Part นี้เป็นต้นไป**: ใช้ `enum class` แทน `enum` ธรรมดาในโค้ด
> C++ ใหม่ทุกจุด ไม่มีข้อยกเว้น เพราะได้ความปลอดภัยด้าน type-safety เพิ่มขึ้นมาก โดยแทบไม่มี
> ข้อเสียเลย (ข้อเสียเดียวคือต้องพิมพ์ `EnumName::` นำหน้าเสมอ ซึ่งถือเป็นราคาที่คุ้มค่ามาก)

---

## 69.8 ฟีเจอร์เล็กๆ ของ C++11 ที่ควรรู้: static_assert และ Uniform Initialization (Step 552)

นอกจากฟีเจอร์ใหญ่ๆ ที่ผ่านมา C++11 ยังเพิ่มฟีเจอร์เล็กๆ อีกจำนวนมากที่ใช้งานกันแทบทุกวันโดย
ไม่รู้สึกว่าเป็นฟีเจอร์ "ใหม่" อีกต่อไป สองตัวที่สำคัญที่สุดคือ `static_assert` และ **uniform
initialization** (การใช้ `{}` กำหนดค่าเริ่มต้น)

### static_assert: ตรวจสอบเงื่อนไขที่ compile-time

`static_assert(condition, message)` ตรวจสอบว่า `condition` เป็นจริงหรือไม่ **ตั้งแต่ตอน
compile** ถ้าเป็นเท็จ compiler จะหยุด compile ทันทีพร้อมแสดง `message` ที่เราเขียนไว้ ต่างจาก
`assert()` (จาก `<cassert>`, ทบทวน Part 16) ที่ตรวจสอบตอน **runtime** เท่านั้น

ประโยชน์สำคัญคือมันจับข้อผิดพลาดได้ **ก่อนที่โปรแกรมจะถูกส่งไปรันจริงเสียอีก** เหมาะมากกับการ
ตรวจสอบสมมติฐานเกี่ยวกับขนาดของ type, ค่าคงที่ที่ไม่ควรเปลี่ยนโดยไม่ตั้งใจ หรือเงื่อนไขของ
template parameter (จะใช้งานหนักมากใน Part 71 เรื่อง constexpr และ Part 78 เรื่อง
metaprogramming)

### Uniform Initialization: ใช้ {} กำหนดค่าเริ่มต้นแบบเดียวกันทุกที่

ก่อน C++11 การกำหนดค่าเริ่มต้นให้ตัวแปรมีหลายรูปแบบไม่สอดคล้องกัน (`int x = 5;`,
`int arr[3] = {1, 2, 3};`, `MyClass obj(1, 2);`) C++11 เสนอไวยากรณ์ `{}` ที่ใช้ได้กับแทบทุก
สถานการณ์แบบเดียวกันหมด เรียกว่า **uniform initialization** หรือ **brace initialization**

```cpp
// 07_misc.cpp
#include <iostream>
#include <string>
#include <vector>

static_assert(sizeof(int) >= 4, "โปรแกรมนี้ต้องการ int อย่างน้อย 4 byte");

struct Point {
    int x;
    int y;
};

class Wallet {
public:
    Wallet(int coins, int bills) : coins_(coins), bills_(bills) {}
    void print() const { std::cout << "coins=" << coins_ << " bills=" << bills_ << '\n'; }

private:
    int coins_;
    int bills_;
};

int main() {
    // Uniform initialization: ใช้ {} ได้กับแทบทุกชนิดข้อมูล เขียนสม่ำเสมอเป็นแบบเดียวกันหมด
    int a{5};
    double d{3.14};
    std::string s{"hello"};
    std::vector<int> v{1, 2, 3};
    Point p{10, 20};
    Wallet w{3, 2};

    std::cout << "a=" << a << " d=" << d << " s=" << s << '\n';
    std::cout << "v.size()=" << v.size() << " p=(" << p.x << ", " << p.y << ")\n";
    w.print();

    // {} ยังป้องกัน narrowing conversion ที่ทำให้ข้อมูลเสียหายโดยไม่รู้ตัว
    int okValue{100};              // ผ่าน: 100 เก็บใน int ได้พอดี ไม่มีข้อมูลสูญหาย
    // int bad{3.14};              // ผิด: compile error ทันที เพราะ 3.14 ใส่ใน int จะเสียเศษทศนิยม (narrowing)
    int badButAllowed = 3.14;      // แบบเก่า (ไม่ใช้ {}) ยอมให้ผ่านเงียบๆ ได้เศษทศนิยมหายไปโดยไม่เตือน

    std::cout << "okValue=" << okValue << " badButAllowed=" << badButAllowed << '\n';

    // static_assert ตรวจสอบเงื่อนไขที่ compile-time เอง ไม่ต้องรอรันโปรแกรมแล้วพังทีหลัง
    static_assert(sizeof(Point) == 2 * sizeof(int), "Point ควรมีขนาดเท่ากับ int สองตัวติดกัน");

    return 0;
}
```

```
a=5 d=3.14 s=hello
v.size()=3 p=(10, 20)
coins=3 bills=2
okValue=100 badButAllowed=3
```

จุดที่สำคัญที่สุดของ uniform initialization ที่ควรจำคือเรื่อง **narrowing conversion**: ถ้าลอง
uncomment บรรทัด `int bad{3.14};` จะได้ compile error ทันที เพราะ `{}` **ห้าม**การแปลงชนิดที่
ทำให้ข้อมูลสูญหาย (แปลง `double` ที่มีเศษทศนิยมเป็น `int` จะทิ้งเศษทศนิยมไป) ในขณะที่การกำหนด
ค่าแบบเก่าด้วย `=` (`int badButAllowed = 3.14;`) **ยอมให้ผ่านไปเงียบๆ** โดยตัดเศษทศนิยมทิ้งโดย
ไม่มีคำเตือนใดๆ เลย (แม้เปิด `-Wall -Wextra -Wpedantic` แล้วก็ตาม เพราะนี่เป็น standard
conversion ที่ภาษาอนุญาตไว้แต่ต้น) นี่คือเหตุผลสำคัญที่ทำให้หลายทีมเลือกใช้ `{}` เป็นค่า
เริ่มต้นสำหรับการประกาศตัวแปรทุกชนิดตั้งแต่ C++11 เป็นต้นมา

---

## ข้อผิดพลาดที่พบบ่อย (Common Pitfalls)

1. **คิดว่า `auto` คือ dynamic typing แบบ Python/JavaScript** — `auto` ยังคงเป็น static typing
   เต็มรูปแบบ ชนิดข้อมูลถูกกำหนดตายตัวตั้งแต่ compile-time เพียงแค่ compiler เขียนให้แทนเราเอง
   ตัวแปรที่ประกาศด้วย `auto` **ไม่สามารถเปลี่ยนชนิดข้อมูลได้ในภายหลัง** เหมือนภาษา dynamic
2. **ลืมว่า `auto` ทิ้ง `const`/reference โดยอัตโนมัติ** ทำให้ range-based for loop ที่เขียน
   `for (auto x : container)` กับ container ขนาดใหญ่ที่เก็บ object หนักๆ (เช่น `std::string`
   หรือ struct ใหญ่ๆ) เกิดการ copy โดยไม่จำเป็นทุกรอบของ loop โดยไม่รู้ตัว ควรใช้
   `const auto&` เกือบทุกครั้งเมื่อไม่จำเป็นต้องแก้ไขหรือ copy element
3. **สับสน `decltype(x)` กับ `decltype((x))`** — วงเล็บคู่เปลี่ยนความหมายจาก "ชนิดตามที่ประกาศ"
   เป็น "ชนิดของนิพจน์ lvalue" ซึ่งมักได้ reference type กลับมาโดยไม่ตั้งใจ
4. **ยังใช้ `NULL` หรือ `0` แทน `nullptr` ในโค้ด C++ ใหม่** — นอกจากปัญหาเรื่อง overload
   resolution ที่กำกวมแล้ว การอ่านโค้ดที่มี `nullptr` ยังสื่อความหมายชัดเจนกว่ามากว่ากำลังทำงาน
   กับ pointer ไม่ใช่ตัวเลข
5. **ใช้ `enum` ธรรมดาต่อไปในโค้ดใหม่โดยไม่มีเหตุผล** — นอกจากปัญหาชื่อรั่วไหลแล้ว การแปลงเป็น
   `int` โดยอัตโนมัติยังเปิดช่องให้เกิดบั๊กจากการเปรียบเทียบ enum คนละความหมายกันโดยไม่มี
   คำเตือนจาก compiler เลย ควรใช้ `enum class` เป็นค่าเริ่มต้นเสมอในโค้ดใหม่
6. **ใช้ uniform initialization (`{}`) กับ `std::vector` แล้วได้ผลลัพธ์ไม่ตรงกับที่คาดหวัง**
   เช่น `std::vector<int> v{5};` จะได้ vector ที่มี **1 element ค่าเท่ากับ 5** (เพราะมี
   constructor แบบ `initializer_list` ที่ compiler เลือกก่อนเสมอถ้ามีให้เลือก) ไม่ใช่ vector
   ที่มี **5 elements ค่าเป็น 0** อย่างที่หลายคนคาดหวัง (ต้องใช้วงเล็บกลม `std::vector<int>
   v(5);` แทนถ้าต้องการความหมายหลัง)
7. **เข้าใจว่า `static_assert` ทำงานเหมือน `assert()` ธรรมดา** — `static_assert` ตรวจสอบที่
   compile-time เท่านั้น เงื่อนไขที่ใส่เข้าไปต้องเป็นค่าที่ compiler รู้ได้ตอน compile (compile-
   time constant expression) ถ้าใส่เงื่อนไขที่ขึ้นกับค่า runtime (เช่นค่าที่มาจากผู้ใช้ป้อน)
   จะเกิด compile error ทันที ไม่ใช่การตรวจสอบตอนรันโปรแกรมแบบ `assert()`

---

## แบบฝึกหัดท้ายบท

1. แปลง `enum` ธรรมดาต่อไปนี้ให้เป็น `enum class` ที่กำหนด underlying type เป็น `int` อย่าง
   ชัดเจน พร้อมเขียนฟังก์ชัน `toString()` ที่แปลงค่าแต่ละตัวเป็นข้อความ และใช้ `static_assert`
   ยืนยันว่าลำดับค่าตรงตามที่กำหนด:
   ```cpp
   enum OrderStatus { PENDING, SHIPPED, DELIVERED, CANCELLED };
   ```
2. เขียนโปรแกรมที่มี `struct Student { std::string name; std::vector<int> scores; };` แล้วเขียน
   ฟังก์ชัน `average()` ที่ใช้ `auto` เป็น return type และฟังก์ชัน `findTopStudent()` ที่คืนค่า
   เป็น pointer (ใช้ `nullptr` แทนการคืนค่าเมื่อ list ว่าง) โดยใช้ `decltype` ประกาศตัวแปรภายใน
   ให้มีชนิดตรงกับตัวแปรอื่นอย่างชัดเจน
3. เขียนโปรแกรมที่ใช้ `auto` วนลูปผ่าน `std::vector<bool>` แล้วพยายามแก้ไขค่าที่ได้จาก `auto`
   ตัวแปรนั้น สังเกตว่าเกิดอะไรขึ้นกับ vector ต้นฉบับ (เทียบกับตอนที่ใช้ `bool` ตรงๆ แทน `auto`)
   อธิบายด้วยคำพูดตัวเองว่าทำไมถึงเกิดพฤติกรรมนี้
4. ทดลองเขียน `for (auto x : container)` เทียบกับ `for (const auto& x : container)` กับ
   `std::vector<std::string>` ที่มีสมาชิกจำนวนมาก (เช่น 100,000 ตัว) แล้วนับจำนวนครั้งที่
   copy constructor ของ `std::string` ถูกเรียก (เพิ่ม `std::cout` ใน wrapper class ที่ห่อ
   `std::string` ไว้เพื่อนับ) เพื่อยืนยันด้วยตาตัวเองว่า `const auto&` ประหยัดกว่าจริง
5. เขียนโปรแกรมที่แสดงกับดักของ `std::vector<int> v{5};` เทียบกับ `std::vector<int> v(5);`
   พร้อมพิมพ์ `v.size()` และเนื้อหาทั้งหมดของทั้งสองออกมาเปรียบเทียบกัน
6. ใช้ `static_assert` ตรวจสอบว่า `sizeof(long long) >= sizeof(int)` และเขียนอธิบายว่าทำไมการ
   ตรวจสอบแบบนี้ที่ compile-time ถึงมีประโยชน์มากกว่าการตรวจสอบตอน runtime ด้วย `assert()`

### แนวทางเฉลยข้อ 1

```cpp
// ex1_enum_class.cpp
#include <iostream>
#include <string_view>

enum class OrderStatus : int { Pending = 0, Shipped = 1, Delivered = 2, Cancelled = 3 };

std::string_view toString(OrderStatus status) {
    switch (status) {
        case OrderStatus::Pending:   return "Pending";
        case OrderStatus::Shipped:   return "Shipped";
        case OrderStatus::Delivered: return "Delivered";
        case OrderStatus::Cancelled: return "Cancelled";
    }
    return "Unknown";
}

int main() {
    OrderStatus s = OrderStatus::Shipped;
    std::cout << "สถานะออเดอร์: " << toString(s) << '\n';

    static_assert(static_cast<int>(OrderStatus::Cancelled) == 3,
                  "ลำดับค่าของ OrderStatus ต้องตรงตามที่ตกลงกับฝั่ง frontend");

    int raw = static_cast<int>(s);
    std::cout << "ค่าตัวเลขของสถานะ = " << raw << '\n';

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 ex1_enum_class.cpp -o ex1
./ex1
```

```
สถานะออเดอร์: Shipped
ค่าตัวเลขของสถานะ = 1
```

สังเกตว่า `toString` ใช้ `switch` แบบ **ไม่มี `default` case** แต่ compiler ไม่เตือนอะไรเลย
เพราะเราครอบคลุมทุกค่าของ `enum class` ไว้ครบแล้ว (`-Wswitch` จะเตือนถ้าลืมกรณีใดกรณีหนึ่งไป
ซึ่งเป็นประโยชน์เพิ่มเติมของการใช้ `enum class` ที่ compiler ช่วยตรวจสอบความครบถ้วนให้ได้)

### แนวทางเฉลยข้อ 2

```cpp
// ex2_auto_decltype_student.cpp
#include <iostream>
#include <string>
#include <vector>

struct Student {
    std::string name;
    std::vector<int> scores;
};

// คืนค่าเฉลี่ยของนักเรียนคนหนึ่ง ใช้ auto เป็น return type (คอมไพเลอร์อนุมานจาก return statement)
auto average(const Student& st) {
    double sum = 0.0;
    for (auto score : st.scores) {   // auto ทำสำเนา int ธรรมดา เหมาะกับ int ที่เบามาก
        sum += score;
    }
    return st.scores.empty() ? 0.0 : sum / static_cast<double>(st.scores.size());
}

// คืน pointer ไปยังนักเรียนที่คะแนนเฉลี่ยสูงสุด หรือ nullptr ถ้า list ว่าง
const Student* findTopStudent(const std::vector<Student>& students) {
    const Student* best = nullptr;
    double bestAvg = -1.0;
    for (const auto& st : students) {
        decltype(bestAvg) avg = average(st);  // decltype ยืนยันว่า avg ชนิดเดียวกับ bestAvg เป๊ะ
        if (avg > bestAvg) {
            bestAvg = avg;
            best = &st;
        }
    }
    return best;
}

int main() {
    std::vector<Student> students = {
        {"Somchai", {80, 90, 100}},
        {"Somsri", {70, 60, 65}},
        {"Suda", {95, 92, 98}},
    };

    for (const auto& st : students) {
        std::cout << st.name << " เฉลี่ย = " << average(st) << '\n';
    }

    const Student* top = findTopStudent(students);
    if (top != nullptr) {
        std::cout << "นักเรียนคะแนนเฉลี่ยสูงสุดคือ: " << top->name << '\n';
    } else {
        std::cout << "ไม่มีนักเรียนในระบบ\n";
    }

    std::vector<Student> empty;
    const Student* none = findTopStudent(empty);
    std::cout << "ผลลัพธ์เมื่อ list ว่าง: " << (none == nullptr ? "nullptr ตามที่คาด" : "ผิดพลาด") << '\n';

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 ex2_auto_decltype_student.cpp -o ex2
./ex2
```

```
Somchai เฉลี่ย = 90
Somsri เฉลี่ย = 65
Suda เฉลี่ย = 95
นักเรียนคะแนนเฉลี่ยสูงสุดคือ: Suda
ผลลัพธ์เมื่อ list ว่าง: nullptr ตามที่คาด
```

จุดสำคัญของเฉลยนี้คือการใช้ `decltype(bestAvg)` แทนที่จะเขียน `double` ตรงๆ — แม้ในกรณีนี้จะ
ให้ผลเหมือนกัน แต่ `decltype` ทำให้โค้ดยืนยัน "ความสัมพันธ์" ระหว่างสองตัวแปรไว้ชัดเจนในระดับ
compile-time: ถ้าใครในอนาคตเปลี่ยนชนิดของ `bestAvg` จาก `double` เป็น `float` (เช่นเพื่อประหยัด
หน่วยความจำ) ตัวแปร `avg` จะเปลี่ยนชนิดตามไปโดยอัตโนมัติ ไม่ต้องตามไปแก้ทุกจุดที่อ้างอิงเอง

---

## สรุปท้ายบท

ใน Part นี้เราได้:

- เข้าใจบริบททางประวัติศาสตร์ของ **C++11** ว่าทำไมถึงถูกเรียกว่า "C++0x" มานานกว่าทศวรรษ และ
  ทำไมมันถึงถูกยกย่องว่าเป็นการเปลี่ยนแปลงครั้งใหญ่ที่สุดของภาษาตั้งแต่ถือกำเนิดขึ้นมา
- ใช้ **`auto`** เพื่อลดการเขียนชนิดข้อมูลที่ซ้ำซ้อน พร้อมเข้าใจกฎการอนุมานชนิด (ทิ้ง
  `const`/reference เสมอ) และรู้ว่าเมื่อไหร่ควร/ไม่ควรใช้มัน
- ใช้ **`decltype`** ดึงชนิดข้อมูลจากนิพจน์ใดๆ ได้ และเข้าใจความแตกต่างสำคัญจาก `auto` (รักษา
  `const`/reference ไว้เสมอ)
- สรุปรวมกลไกเบื้องหลังของ **range-based for loop** อย่างเป็นทางการ (การ desugar เป็น
  `begin()`/`end()` และ iterator)
- ใช้ **`nullptr`** แทน `NULL`/`0` เพื่อแก้ปัญหาการเลือก overload ผิดที่มีมาตั้งแต่ยุค C
- ใช้ **`enum class`** แทน `enum` ธรรมดา เพื่อป้องกันชื่อรั่วไหลและการแปลงชนิดข้อมูลแบบไม่
  ตั้งใจ
- ใช้ **`static_assert`** ตรวจสอบเงื่อนไขที่ compile-time และใช้ **uniform initialization**
  (`{}`) เพื่อความสม่ำเสมอและความปลอดภัยจาก narrowing conversion

ฟีเจอร์ทั้งหมดที่เรียนใน Part นี้เป็นแค่ "ยอดภูเขาน้ำแข็ง" ของ C++11 เท่านั้น — สิ่งที่ทรงพลัง
และเปลี่ยนวิธีคิดของโปรแกรมเมอร์ C++ มากที่สุดยังไม่ได้พูดถึงเลย นั่นคือ **Move Semantics** ซึ่ง
เป็นกลไกที่ทำให้ C++ ส่งต่อทรัพยากรขนาดใหญ่ (เช่น `std::vector` ที่มีข้อมูลนับล้านตัว) ระหว่าง
ฟังก์ชันได้โดยแทบไม่มีค่าใช้จ่ายด้านประสิทธิภาพเลย ต่างจากการ copy แบบเดิมที่ต้องคัดลอกข้อมูล
ทั้งหมดทุกครั้ง

**ต่อไป:** [Part 70 — Move Semantics และ Rvalue Reference](./part-070-move-semantics.md)
