# Part 77: ภาพรวมฟีเจอร์ใหม่ C++23 (Step 609–616)

> Module F — Modern C++ (C++11 ถึง C++23) | Part 77 จาก 125
> Step ที่ครอบคลุมใน Part นี้: Step 609–616
> Part ก่อนหน้า: [Part 76 — C++20 Modules](./part-076-cpp20-modules.md) | Part ถัดไป: [Part 78 — Metaprogramming](./part-078-metaprogramming.md)

## เป้าหมายการเรียนรู้ (Learning Objectives)

หลังจบ Part นี้ ผู้เรียนจะสามารถ:

1. อธิบายภาพรวมของมาตรฐาน C++23 ได้ว่าเน้นปรับปรุงด้านใดเป็นหลัก และต่างจาก C++20 อย่างไร
2. ใช้ `std::expected` เขียน error handling แบบไม่ throw exception ได้ พร้อมเข้าใจว่าต่างจาก
   `std::optional` อย่างไรและควรเลือกใช้ตัวไหนเมื่อใด
3. เขียนและอธิบาย multidimensional subscript operator (`operator[]` ที่รับหลาย argument
   เช่น `matrix[i, j]`) ได้
4. ใช้ `if consteval` แยกพฤติกรรมของฟังก์ชันระหว่างตอนที่ทำงานใน compile-time กับ runtime ได้
5. อธิบายแนวคิดของ deducing this (explicit object parameter) ได้ว่าคืออะไร แก้ปัญหาอะไร
   แม้จะยังทดสอบรันจริงไม่ได้บนคอมไพเลอร์หลักของหลักสูตรนี้
6. อธิบายแนวคิดของ `std::print`/`std::println` (`<print>`) ได้ว่าแก้ปัญหาอะไรของ
   `printf`/`std::cout` แม้จะยังทดสอบรันจริงไม่ได้บนคอมไพเลอร์หลักของหลักสูตรนี้เช่นกัน
7. ระบุได้อย่างชัดเจนและมีหลักฐานว่าฟีเจอร์ C++23 ตัวไหนใช้งานได้จริงบน g++ 13.3.0 (คอมไพเลอร์
   หลักของหลักสูตรนี้) และตัวไหนต้องรอคอมไพเลอร์เวอร์ชันใหม่กว่า

---

## 77.1 ภาพรวม C++23: มาตรฐานนี้เน้นอะไร (Step 609)

หลังจาก C++20 ที่เพิ่มฟีเจอร์ระดับภาษาขนาดใหญ่ 4 ตัวรวด (Concepts, Ranges, Coroutines,
Modules — ที่เราเรียนมาตั้งแต่ Part 73) คณะกรรมการมาตรฐาน ISO ตั้งใจให้ **C++23 เป็น
"การอัปเดตขนาดเล็ก" (a smaller release)** เมื่อเทียบกับ C++20 โดยเน้นไปที่:

1. **แก้ไขและเติมเต็มช่องว่างของ C++20** — ฟีเจอร์ใหญ่ของ C++20 หลายตัวยังขาดเครื่องมือ
   สนับสนุนที่ครบถ้วน (เช่น Ranges ยังขาด view หลายตัวที่ควรมี) C++23 เข้ามาเติมส่วนที่ขาด
2. **ปรับปรุง Standard Library เป็นหลัก** มากกว่าเพิ่ม syntax ใหม่ในระดับภาษา (แม้จะมี
   ฟีเจอร์ระดับภาษาใหม่อยู่บ้าง เช่น deducing this, `if consteval`, multidimensional
   subscript operator)
3. **แก้ปัญหาคลาสสิกที่ค้างคามานาน** เช่น การ handle error โดยไม่ throw exception
   (`std::expected`) และการพิมพ์ข้อความที่ปลอดภัยและเร็วกว่า (`std::print`)

| | C++20 | C++23 |
|---|---|---|
| ขนาดของการเปลี่ยนแปลง | ใหญ่มาก (เทียบเท่า C++11) | เล็กกว่า เน้นเติมเต็ม |
| จุดเน้นหลัก | ฟีเจอร์ภาษาใหม่ (Concepts/Ranges/Coroutines/Modules) | Library ใหม่ + แก้ปัญหาการใช้งานเดิม |
| ตัวอย่างฟีเจอร์เด่น | `concept`, `co_await`, `import` | `std::expected`, `std::print`, deducing this |

**ข้อควรระวังที่สำคัญที่สุดสำหรับบทเรียนนี้**: การรองรับ C++23 ในคอมไพเลอร์แต่ละตัว ณ
ปี 2026 **ไม่เท่ากันเลย** แม้แต่ในคอมไพเลอร์ตัวเดียวกันคนละเวอร์ชันก็รองรับไม่เท่ากัน
บทเรียนนี้จะ**ทดสอบทุกฟีเจอร์จริงบนเครื่องที่ใช้เขียนหลักสูตร** ซึ่งมี **g++ 13.3.0**
เป็นคอมไพเลอร์หลัก (ตรวจสอบด้วย `g++ --version`) และจะบอกตรงไปตรงมาว่าฟีเจอร์ไหนใช้ได้จริง
ฟีเจอร์ไหนยังใช้ไม่ได้ — ไม่มีการอ้างว่าโค้ดคอมไพล์ผ่านทั้งที่ไม่เคยทดสอบจริง

```bash
g++ --version
# g++ (Ubuntu 13.3.0-6ubuntu2~24.04.1) 13.3.0
```

วิธีตรวจสอบว่าคอมไพเลอร์รองรับฟีเจอร์ไหนบ้างอย่างเป็นระบบ (แทนที่จะเดา) คือใช้
**Feature-Test Macro** จาก header `<version>` ที่มาตรฐานกำหนดไว้ให้:

```cpp
#include <version>
#include <iostream>

int main() {
#ifdef __cpp_lib_expected
    std::cout << "std::expected: " << __cpp_lib_expected << "\n";
#else
    std::cout << "std::expected: NOT SUPPORTED\n";
#endif
}
```

จะใช้เทคนิคนี้ตรวจสอบทีละฟีเจอร์ตลอดทั้งบทเรียนนี้

---

## 77.2 std::expected: Error Handling แบบไม่ Throw Exception (Step 610)

### ปัญหาที่ std::expected แก้

ทบทวนจาก **Part 54** (Exception Handling) และ **Part 72** (`std::optional`/`variant`):
เรามีสองแนวทางหลักในการรายงานความล้มเหลวของฟังก์ชันมาตลอด:

1. **Exception** (`throw`/`catch`) — ทรงพลังแต่มีต้นทุนด้าน performance เมื่อเกิดขึ้นจริง
   และทำให้ control flow ของโปรแกรม "มองไม่เห็น" จาก signature ของฟังก์ชัน (ต้องอ่านเอกสาร
   หรือโค้ดข้างในถึงจะรู้ว่าฟังก์ชันนี้ throw อะไรได้บ้าง)
2. **`std::optional<T>`** — บอกได้แค่ว่า "มีค่าหรือไม่มีค่า" แต่**บอกไม่ได้ว่าทำไมถึงไม่มี
   ค่า** เช่น ฟังก์ชันแปลงข้อความเป็นตัวเลขที่ล้มเหลว จะไม่รู้เลยว่าล้มเหลวเพราะสตริงว่าง
   เพราะมีตัวอักษรปน หรือเพราะตัวเลขเกินขอบเขต

`std::expected<T, E>` (จาก `<expected>`) แก้ปัญหาข้อ 2 ได้ตรงจุด: มันคือ type ที่**เก็บ
ได้ทั้งค่าที่สำเร็จ (type `T`) หรือค่า error พร้อมรายละเอียด (type `E`)** โดยไม่ต้อง throw
exception เลย เปรียบเทียบง่ายๆ คือ `Result<T, E>` ใน Rust หรือ `Either` ใน Haskell

### ตัวอย่างที่ทดสอบคอมไพล์และรันจริงแล้ว

```cpp
// expected_demo.cpp
#include <expected>
#include <iostream>
#include <string>

std::expected<int, std::string> parse_positive(int x) {
    if (x < 0) {
        return std::unexpected("negative number");   // รายงาน error พร้อมรายละเอียด
    }
    return x * 2;   // ค่าที่สำเร็จ ไม่ต้องห่ออะไรพิเศษ
}

int main() {
    auto r1 = parse_positive(5);
    auto r2 = parse_positive(-3);

    if (r1) {
        std::cout << "r1 = " << *r1 << "\n";              // เข้าถึงค่าด้วย * เหมือน optional
    }
    if (!r2) {
        std::cout << "r2 error: " << r2.error() << "\n";  // เข้าถึง error ด้วย .error()
    }
}
```

```bash
g++ -Wall -Wextra -std=c++23 expected_demo.cpp -o expected_demo
./expected_demo
```

ผลลัพธ์ (ทดสอบจริงบนเครื่องนี้):

```
r1 = 10
r2 error: negative number
```

### เปรียบเทียบ std::optional กับ std::expected

| | `std::optional<T>` | `std::expected<T, E>` |
|---|---|---|
| บอกได้ว่ามีค่าหรือไม่ | ใช่ | ใช่ |
| บอกได้ว่า**ทำไม**ถึงไม่มีค่า | ไม่ได้ | ได้ (ผ่าน type `E`) |
| เหมาะกับ | ค่าที่ "ไม่มี" เป็นเรื่องปกติ ไม่ต้องอธิบายเหตุผล (เช่น หา element ใน container ไม่เจอ) | การดำเนินการที่ล้มเหลวได้หลายสาเหตุ และผู้เรียกควรรู้สาเหตุ (เช่น parse, network, I/O) |
| ต้นทุน | เบา | เบา (ไม่มีการจอง heap หรือ throw เหมือน exception) |

### Monadic Operations (C++23 เพิ่มให้ทั้ง optional และ expected)

C++23 เพิ่มเมธอดแบบ "monadic" ให้ทั้ง `std::optional` และ `std::expected` ทำให้ต่อ
pipeline การประมวลผลได้โดยไม่ต้องเขียน `if` ตรวจสอบทีละขั้น ทดสอบแล้วว่าใช้งานได้จริงบน
g++ 13.3.0:

```cpp
#include <expected>
#include <iostream>
#include <string>

std::expected<int, std::string> half(int x) {
    if (x % 2 != 0) return std::unexpected("odd number");
    return x / 2;
}

int main() {
    std::expected<int, std::string> e = 10;

    // and_then: ถ้าสำเร็จ ส่งค่าต่อให้ฟังก์ชันถัดไป (ซึ่งคืน expected เหมือนกัน)
    // transform: ถ้าสำเร็จ แปลงค่าธรรมดา (ไม่ต้องคืน expected)
    auto r = e.and_then(half).transform([](int x) { return x + 1; });
    std::cout << (r ? std::to_string(*r) : r.error()) << "\n";   // 6

    std::expected<int, std::string> e2 = 7;
    auto r2 = e2.and_then(half);
    std::cout << (r2 ? std::to_string(*r2) : r2.error()) << "\n";   // odd number
}
```

ทดสอบจริงได้ผลลัพธ์:

```
6
odd number
```

`std::optional` ก็มี `.and_then()`, `.transform()`, `.or_else()` แบบเดียวกัน — ทดสอบแล้ว
ว่าคอมไพล์และทำงานถูกต้องบนเครื่องนี้เช่นกัน

---

## 77.3 Multidimensional Subscript Operator (Step 611)

### ปัญหาเดิม

ก่อน C++23 `operator[]` **รับได้แค่ 1 argument เท่านั้น** ทำให้คลาสที่ต้องการทำ index
แบบหลายมิติ (เช่น matrix, tensor) ต้องใช้ทางเลือกที่ไม่สวยงามนัก เช่น
`operator()(int r, int c)` (ใช้วงเล็บกลมแทนวงเล็บเหลี่ยม), หรือ `operator[](int r)`
คืน proxy object แล้วเรียก `operator[](int c)` ซ้อนอีกที (`matrix[r][c]`) ซึ่งมี overhead
และเขียน implementation ยุ่งยากกว่าที่ควรจะเป็น

### C++23 แก้ปัญหานี้โดยตรง

C++23 (P2128) อนุญาตให้ `operator[]` **รับหลาย argument พร้อมกันได้** ทำให้เขียน
`matrix[i, j]` ได้ตรงๆ ทดสอบแล้วว่าคอมไพล์และรันได้จริงบน g++ 13.3.0:

```cpp
// matrix_demo.cpp
#include <iostream>
#include <vector>

struct Matrix {
    std::vector<double> data;
    int rows, cols;

    Matrix(int r, int c) : data(r * c, 0.0), rows(r), cols(c) {}

    // operator[] รับ 2 argument พร้อมกัน -- ฟีเจอร์ใหม่ของ C++23
    double& operator[](int r, int c) {
        return data[r * cols + c];
    }
    double operator[](int r, int c) const {
        return data[r * cols + c];
    }
};

int main() {
    Matrix m(2, 2);
    m[0, 0] = 1.0;
    m[0, 1] = 2.0;
    m[1, 0] = 3.0;
    m[1, 1] = 4.0;
    std::cout << m[0, 0] << " " << m[0, 1] << " "
              << m[1, 0] << " " << m[1, 1] << "\n";
}
```

```bash
g++ -Wall -Wextra -std=c++23 matrix_demo.cpp -o matrix_demo
./matrix_demo
```

ผลลัพธ์ (ทดสอบจริง):

```
1 2 3 4
```

### ตรวจสอบด้วย Feature-Test Macro

```cpp
#ifdef __cpp_multidimensional_subscript
    std::cout << __cpp_multidimensional_subscript << "\n";   // 202211
#endif
```

ทดสอบแล้วว่า g++ 13.3.0 กำหนด macro นี้เป็น `202211` ยืนยันว่ารองรับฟีเจอร์นี้เต็มรูปแบบ
ตามมาตรฐาน

**ข้อควรระวัง**: `m[0, 0]` ในโค้ด **ก่อน** C++23 (เช่นคอมไพล์ด้วย `-std=c++17`) จะยังคอมไพล์
ผ่านได้เหมือนกัน แต่ความหมายจะ**ต่างกันโดยสิ้นเชิง**! เพราะ `,` ใน `[0, 0]` จะถูกตีความเป็น
**comma operator** (ประเมินค่าซ้ายทิ้งแล้วคืนค่าขวา) ทำให้ `m[0, 0]` เท่ากับ `m[0]` เฉยๆ
(ถ้า `operator[]` รับ 1 argument อยู่แล้ว) นี่คือเหตุผลที่มาตรฐานต้อง**เปลี่ยนความหมายของ
comma ใน `[]` โดยเฉพาะ** เมื่อเปิดใช้ C++23 — เป็นหนึ่งใน breaking change เล็กๆ ที่ควรรู้ไว้

---

## 77.4 if consteval (Step 612)

### ปัญหาที่ต้องแก้

ทบทวนจาก **Part 71** (`constexpr`): ฟังก์ชันที่เป็น `constexpr` สามารถถูกเรียกได้ทั้งตอน
**compile-time** และ **runtime** แต่บางครั้งเราอยากให้ฟังก์ชันทำงาน**ต่างกัน**ระหว่างสอง
บริบทนี้ เช่น ตอน compile-time อยากใช้อัลกอริทึมที่ปลอดภัยแต่ช้ากว่า (เพราะไม่มีต้นทุนจริงที่
runtime) ส่วนตอน runtime อยากใช้อัลกอริทึมที่เร็วกว่า (เช่น ใช้ SIMD intrinsic ที่ใช้ตอน
compile-time ไม่ได้)

ก่อนหน้านี้มีฟังก์ชัน `std::is_constant_evaluated()` (C++20) ให้ตรวจสอบได้ แต่การใช้งาน
ร่วมกับ `if` ธรรมดามีข้อผิดพลาดที่พบบ่อย (compiler บาง optimize path ผิดพลาดได้ในบางกรณี
ขอบเขต) C++23 จึงเพิ่ม keyword `if consteval` ให้ใช้งานตรงไปตรงมาและปลอดภัยกว่า

### ตัวอย่างที่ทดสอบคอมไพล์และรันจริงแล้ว

```cpp
// if_consteval_demo.cpp
#include <iostream>

constexpr int compute(int x) {
    if consteval {
        return x * 2;   // ทำงานตอน compile-time
    } else {
        return x * 3;   // ทำงานตอน runtime
    }
}

int main() {
    constexpr int a = compute(5);   // ประเมินตอน compile-time -> ใช้ path "* 2"
    int y = 5;
    int b = compute(y);             // y ไม่ใช่ constant -> ทำงานตอน runtime -> ใช้ path "* 3"
    std::cout << a << " " << b << "\n";
}
```

```bash
g++ -Wall -Wextra -std=c++23 if_consteval_demo.cpp -o if_consteval_demo
./if_consteval_demo
```

ผลลัพธ์ (ทดสอบจริง):

```
10 15
```

สังเกตว่า `a` ได้ `5 * 2 = 10` (compile-time path) ส่วน `b` ได้ `5 * 3 = 15` (runtime path)
ทั้งที่เรียกฟังก์ชันเดียวกันด้วยค่า `5` เหมือนกัน — พิสูจน์ว่า `if consteval` เลือก branch
ต่างกันจริงตามบริบทที่ถูกเรียก ยืนยันด้วย feature-test macro `__cpp_if_consteval` ที่ g++
13.3.0 กำหนดเป็น `202106` (รองรับเต็มรูปแบบ)

**ข้อควรจำ**: เขียน `if consteval { ... }` (ไม่มีวงเล็บครอบเงื่อนไขแบบ `if (consteval)`)
เพราะ `consteval` ในบริบทนี้เป็นส่วนหนึ่งของ syntax ใหม่ ไม่ใช่ expression ที่ประเมินค่า
`true`/`false` แบบ `if` ปกติ และ `else` (ถ้ามี) จะหมายถึง "ตอน runtime" เสมอ

---

## 77.5 Deducing This (Explicit Object Parameter) (Step 613)

### แนวคิด

Deducing this (P0847) คือฟีเจอร์ที่อนุญาตให้ **ประกาศพารามิเตอร์ตัวแรกของ member function
เป็น `this` แบบชัดเจน** แทนที่จะให้ compiler แอบส่ง `this` ให้โดยอัตโนมัติแบบที่เป็นมาตลอด
ตั้งแต่ Part 45:

```cpp
struct Widget {
    int value = 42;

    // รูปแบบเดิม (ก่อน C++23): this ถูกส่งให้โดยนัย มองไม่เห็นใน parameter list
    int get_old() const { return value; }

    // รูปแบบใหม่ (C++23): ระบุ "explicit object parameter" ตรงๆ เป็นพารามิเตอร์ตัวแรก
    auto get_new(this const Widget& self) { return self.value; }
};
```

### ปัญหาที่แก้ได้

1. **เลิกต้องเขียน const/non-const overload ซ้ำสองรอบ** — เดิมทีถ้าอยากให้ method ทำงาน
   ได้ทั้งกับ object ที่เป็น `const` และไม่ใช่ `const` (คืนค่า reference ที่ตรงกัน) ต้อง
   เขียน overload แยกกัน 2 ตัว (`T& get()` กับ `const T& get() const`) แต่ด้วย deducing
   this เขียน template ตัวเดียวให้ compiler deduce type ของ `self` ให้เองได้เลย:
   ```cpp
   template <typename Self>
   auto&& get(this Self&& self) { return std::forward<Self>(self).value; }
   ```
2. **เขียน recursive lambda ได้ตรงๆ** — ก่อนหน้านี้ lambda เรียกตัวเองไม่ได้โดยตรง (เพราะ
   lambda ไม่มีชื่อให้เรียกตัวเองจากข้างในตอนที่ยังนิยามไม่เสร็จ) ต้องใช้ `std::function`
   หรือ Y-combinator วนอ้อม แต่ deducing this ทำให้ lambda รับ "ตัวมันเอง" เป็นพารามิเตอร์
   แรกได้ตรงๆ:
   ```cpp
   auto factorial = [](this auto self, int n) -> int {
       return n <= 1 ? 1 : n * self(n - 1);
   };
   ```
3. **ใช้แทน CRTP (Curiously Recurring Template Pattern) ได้ในหลายกรณี** ที่เดิมต้องใช้
   template ซับซ้อนเพื่อให้ base class เรียก method ของ derived class ได้ (จะกล่าวถึง CRTP
   แบบเต็มใน Part 79)

### สถานะการรองรับ — ทดสอบจริงแล้ว

**ต้องพูดตรงไปตรงมาที่สุด: g++ 13.3.0 (คอมไพเลอร์หลักของหลักสูตรนี้) ยังไม่รองรับฟีเจอร์นี้**
ทดสอบโค้ดนี้:

```cpp
#include <iostream>

struct Widget {
    int value = 42;
    auto get(this const Widget& self) {
        return self.value;
    }
};

int main() {
    Widget w;
    std::cout << w.get() << "\n";
}
```

```bash
g++ -std=c++23 -Wall -Wextra -o deducing_this deducing_this.cpp
```

ผลลัพธ์จริงบนเครื่องนี้คือ **compile error** ทันที:

```
deducing_this.cpp:6:14: error: expected identifier before 'this'
    6 |     auto get(this const Widget& self) {
      |              ^~~~
```

ยืนยันด้วย feature-test macro เช่นกัน: `__cpp_explicit_this_parameter` **ไม่ถูกกำหนด**
บน g++ 13.3.0 เลย

อย่างไรก็ตาม เพื่อให้เห็นภาพว่าฟีเจอร์นี้ทำงานได้จริงในคอมไพเลอร์ที่รองรับ ได้ทดลองคอมไพล์
โค้ดเดียวกันนี้ด้วย **Clang 18.1.3** ที่มีอยู่บนเครื่องเดียวกัน (`clang++-18 --version`)
ผลปรากฏว่า **คอมไพล์ผ่านและรันได้จริง**:

```bash
clang++-18 -std=c++23 -Wall -Wextra -o deducing_this deducing_this.cpp
./deducing_this
# 42
```

**หมายเหตุที่น่าสนใจจากการทดสอบ**: แม้ Clang 18.1.3 จะรองรับ syntax นี้จริง (โค้ดคอมไพล์
และรันได้) แต่เมื่อตรวจสอบด้วย feature-test macro `__cpp_explicit_this_parameter` ผ่าน
`<version>` header กลับพบว่า **ไม่ถูกกำหนดเช่นกัน** บนการตั้งค่านี้ (Clang บนเครื่องนี้ใช้
libstdc++ ของ GCC เป็น standard library ซึ่งตรวจสอบ compiler capability บางส่วนผ่าน
`__GNUC__`) — นี่คือบทเรียนสำคัญที่ควรจำ: **feature-test macro ที่มาจาก header ของ
standard library หนึ่งๆ ไม่ได้รับประกันเสมอไปว่าจะสะท้อนความสามารถจริงของ compiler ทุกตัว
ที่ใช้ standard library นั้น** ทางที่ปลอดภัยที่สุดคือ**ทดลองคอมไพล์โค้ดจริงดู** ไม่ใช่เชื่อ
macro เพียงอย่างเดียวเสมอไป

**สรุปสำหรับผู้เรียน**: เข้าใจ syntax และแนวคิดของ deducing this ไว้ให้แม่น เพราะเป็น
ทิศทางสำคัญของ C++ ยุคหน้า (โดยเฉพาะเรื่องลด boilerplate ของ const/non-const overload)
แต่ถ้าใช้ g++ เป็นคอมไพเลอร์หลักในโปรเจกต์จริง ต้องรอ GCC เวอร์ชันใหม่กว่าที่ทดสอบบน
เครื่องนี้ก่อนถึงจะใช้ฟีเจอร์นี้ในโค้ด production ได้

---

## 77.6 std::print และ std::println (Step 614)

### ปัญหาที่ต้องการแก้

ทบทวนจาก **Part 3** และ **Part 42**: C++ มีสองวิธีหลักในการพิมพ์ข้อความมาตลอด —
`printf` (สืบทอดจาก C) กับ `std::cout` (C++ streams) ทั้งคู่มีข้อเสียคนละแบบ:

| | `printf` | `std::cout` |
|---|---|---|
| Type safety | ไม่มี — ใส่ `%d` กับตัวแปร `double` ผิด type แล้ว Undefined Behavior ทันที | มี (compiler ตรวจ type ให้) |
| ความเร็ว | เร็ว | ช้ากว่า (มี overhead ของ stream state, locale) |
| Syntax เขียนสั้นแค่ไหน | สั้น กระชับ | ต้องเขียน `<<` ต่อกันยาวๆ อ่านยากเมื่อมีตัวแปรเยอะ |

`std::print`/`std::println` จาก header `<print>` (C++23) ออกแบบมาให้ได้**ทั้งความปลอดภัย
ของ type และความเร็ว** โดยใช้ syntax แบบ format string คล้าย `std::format` (C++20) ที่ใช้
`{}` แทน placeholder และ compiler ตรวจสอบ type ให้ตั้งแต่ compile-time:

```cpp
// รูปแบบไวยากรณ์ตามมาตรฐาน (ยังไม่ได้ทดสอบรันจริงบนเครื่องนี้ ดูเหตุผลด้านล่าง)
#include <print>

int main() {
    std::print("Hello, {}!\n", "world");
    std::println("value = {}", 42);   // println เติม newline ให้อัตโนมัติ ไม่ต้องใส่ \n เอง
}
```

### สถานะการรองรับ — ทดสอบจริงแล้ว

**g++ 13.3.0 ยังไม่มี header `<print>` เลย** ทดสอบแล้วได้ผลลัพธ์:

```
$ g++ -std=c++23 -Wall -Wextra -o print_test print_test.cpp
print_test.cpp:1:10: fatal error: print: No such file or directory
    1 | #include <print>
      |          ^~~~~~~~
compilation terminated.
```

ทดสอบซ้ำกับ **Clang 18.1.3** บนเครื่องเดียวกัน (ซึ่งใช้ libstdc++ ของระบบเป็น standard
library ไม่ใช่ libc++ ของตัวเอง) ก็ได้ผลเดียวกัน:

```
$ clang++-18 -std=c++23 -o print_test print_test.cpp
print_test.cpp:1:10: fatal error: 'print' file not found
    1 | #include <print>
      |          ^~~~~~~~
```

ยืนยันด้วย feature-test macro `__cpp_lib_print` — **ไม่ถูกกำหนดทั้งสองคอมไพเลอร์** บน
เครื่องนี้ สาเหตุคือ `<print>` ต้องพึ่งพา implementation ของ standard library (libstdc++
หรือ libc++) ที่รองรับมันโดยเฉพาะ ซึ่งบน libstdc++ เวอร์ชันที่มากับ GCC 13.3.0 ยังไม่มี
การ implement header นี้ไว้เลย (ต่างจาก compiler frontend ล้วนๆ อย่าง deducing this ที่
Clang รองรับได้แม้ standard library จะเป็นของ GCC — เพราะ `<print>` เป็นเรื่องของ library
ไม่ใช่เรื่องของภาษา)

**สรุปสำหรับผู้เรียน**: เข้าใจแนวคิดและ syntax ของ `std::print`/`std::println` ไว้ เพราะ
เป็นทิศทางที่ชัดเจนว่าจะมาแทนที่ `printf`/`std::cout` ในระยะยาว (ปลอดภัยกว่า `printf`
เร็วกว่า `std::cout`) แต่ในทางปฏิบัติกับ g++ เวอร์ชันนี้ ยังต้องใช้ `std::cout` หรือ
`printf` ต่อไปก่อน จนกว่าจะอัปเกรด libstdc++ เป็นเวอร์ชันที่รองรับ `<print>`

---

## 77.7 ภาพรวมฟีเจอร์อื่นๆ ของ C++23 (Step 615)

นอกจากฟีเจอร์หลักที่อธิบายไปแล้ว C++23 ยังมีของใหม่อีกจำนวนมาก ต่อไปนี้คือฟีเจอร์ที่
**ทดสอบคอมไพล์และรันจริงแล้วบน g++ 13.3.0** พร้อมทั้งฟีเจอร์ที่ยังใช้ไม่ได้:

### ฟีเจอร์ที่ใช้งานได้จริงบน g++ 13.3.0 (ทดสอบแล้ว)

**`std::stacktrace`** (`<stacktrace>`) — เก็บและแสดง call stack ได้โดยไม่ต้องพึ่ง debugger
ภายนอก มีประโยชน์มากสำหรับ logging error ใน production:

```cpp
#include <stacktrace>
#include <iostream>

void inner() {
    auto st = std::stacktrace::current();
    std::cout << "frames: " << st.size() << "\n";
    if (!st.empty()) std::cout << "top: " << st[0].description() << "\n";
}
void outer() { inner(); }
int main() { outer(); }
```

**ข้อควรระวังที่พบจากการทดสอบจริง**: header `<stacktrace>` compile ผ่านปกติ แต่ตอน
**link** จะ error `undefined reference to '__glibcxx_backtrace_create_state'` ถ้าไม่เพิ่ม
library เสริมให้ครบ ต้อง link เพิ่มด้วย `-lstdc++_libbacktrace`:

```bash
g++ -std=c++23 -g -o stacktrace_demo stacktrace_demo.cpp -lstdc++_libbacktrace
./stacktrace_demo
```

ทดสอบจริงได้ผลลัพธ์ (จำนวน frame และชื่อฟังก์ชันอาจต่างกันไปตามโครงสร้างโค้ดจริง):

```
frames: 7
top: inner()
```

**Ranges Views ใหม่** (`std::views::zip`, `std::views::chunk`, `std::views::enumerate`)
— ต่อยอดจาก C++20 Ranges ที่เรียนใน Part 74 ทดสอบแล้วว่าใช้งานได้จริง:

```cpp
#include <ranges>
#include <vector>
#include <iostream>

int main() {
    std::vector<int> a{1, 2, 3};
    std::vector<char> b{'x', 'y', 'z'};

    // zip: จับคู่ 2 range เข้าด้วยกันทีละตำแหน่ง
    for (auto [x, y] : std::views::zip(a, b)) {
        std::cout << x << y << " ";
    }
    std::cout << "\n";   // 1x 2y 3z

    // chunk: แบ่ง range เป็นกลุ่มย่อยขนาดเท่าๆ กัน
    for (auto chunk : std::views::chunk(a, 2)) {
        for (int v : chunk) std::cout << v << " ";
        std::cout << "| ";
    }
    std::cout << "\n";   // 1 2 | 3 |

    // enumerate: จับคู่แต่ละ element กับ index ของมัน
    for (auto [i, v] : std::views::enumerate(a)) {
        std::cout << i << ":" << v << " ";
    }
    std::cout << "\n";   // 0:1 1:2 2:3
}
```

ทดสอบจริง คอมไพล์ผ่านและรันได้ผลลัพธ์ตรงตามคอมเมนต์ทุกบรรทัด

**Static `operator()`** — เดิมทีทุก operator overload (รวมถึง `operator()`) ต้องเป็น
non-static member function เสมอ (เพราะต้องมี `this` โดยปริยาย) C++23 อนุญาตให้
`operator()` เป็น `static` ได้ถ้ามันไม่ต้องใช้ state ของ object เลย (ลด overhead การส่ง
`this` โดยไม่จำเป็น):

```cpp
struct Adder {
    static int operator()(int a, int b) { return a + b; }
};
int main() {
    Adder add;
    return add(2, 3);   // เรียกได้ปกติ แม้ operator() จะเป็น static
}
```

ทดสอบแล้วคอมไพล์และรันได้ผลลัพธ์ถูกต้อง (`add(2, 3)` ได้ `5`)

**Literal Suffix `uz`/`z`** สำหรับ `size_t` — ก่อนหน้านี้การเขียนตัวเลขคงที่ชนิด `size_t`
ตรงๆ ทำได้ไม่สะดวก (ต้อง cast หรือใช้ `sizeof` มาเทียบ) C++23 เพิ่ม suffix `uz` ให้ใช้ได้ตรงๆ

```cpp
auto x = 10uz;   // x มี type เป็น size_t (unsigned) ทดสอบแล้วว่าคอมไพล์ผ่านจริง
```

**`std::to_underlying`, `std::byteswap`, `std::unreachable`** (จาก `<utility>` และ
`<bit>`) — ฟังก์ชัน utility เล็กๆ ที่ช่วยลดโค้ดซ้ำซากที่เคยต้องเขียนเอง: แปลง `enum class`
กลับเป็น underlying type, สลับ byte order, และบอก compiler ว่า "จุดนี้ไม่มีทางถูกรันถึง"
เพื่อช่วย optimization ทดสอบยืนยันแล้วว่า feature-test macro ของทั้งสามตัวนี้ถูกกำหนดครบ
บน g++ 13.3.0

### ฟีเจอร์ที่ยังใช้ไม่ได้บน g++ 13.3.0 (ทดสอบแล้ว — Header ไม่มีอยู่จริง)

| ฟีเจอร์ | Header | ผลการทดสอบ |
|---|---|---|
| `std::mdspan` (multidimensional array view) | `<mdspan>` | `fatal error: mdspan: No such file or directory` |
| `std::flat_map`/`std::flat_set` | `<flat_map>`/`<flat_set>` | `fatal error: flat_map: No such file or directory` |
| `std::print`/`std::println` | `<print>` | ตามที่แสดงใน 77.6 |
| Deducing this | (ภาษา ไม่ใช่ library) | ตามที่แสดงใน 77.5 |

### ตารางสรุปฟีเจอร์ทั้งหมดที่ทดสอบในบทเรียนนี้

| ฟีเจอร์ | g++ 13.3.0 (`-std=c++23`) | หมายเหตุ |
|---|---|---|
| `std::expected` + monadic ops | ใช้งานได้ | ทดสอบเต็มรูปแบบใน 77.2 |
| Multidimensional `operator[]` | ใช้งานได้ | `__cpp_multidimensional_subscript` = 202211 |
| `if consteval` | ใช้งานได้ | `__cpp_if_consteval` = 202106 |
| `std::optional` monadic ops | ใช้งานได้ | `and_then`/`transform`/`or_else` |
| `std::stacktrace` | ใช้งานได้ (ต้อง `-lstdc++_libbacktrace`) | ระวังเรื่อง link flag |
| `views::zip`/`chunk`/`enumerate` | ใช้งานได้ | ต่อยอด Ranges จาก Part 74 |
| Static `operator()` | ใช้งานได้ | |
| Literal suffix `uz` | ใช้งานได้ | |
| `std::to_underlying`/`byteswap`/`unreachable` | ใช้งานได้ | |
| Deducing this | **ใช้ไม่ได้** (compile error) | ใช้ได้บน Clang 18.1.3 ที่ทดสอบเทียบ |
| `std::print`/`std::println` | **ใช้ไม่ได้** (header ไม่มี) | ใช้ไม่ได้บน Clang 18.1.3 เช่นกัน (ขึ้นกับ libstdc++) |
| `std::mdspan` | **ใช้ไม่ได้** (header ไม่มี) | |
| `std::flat_map`/`flat_set` | **ใช้ไม่ได้** (header ไม่มี) | |

---

## 77.8 สรุปสถานะและคำแนะนำเชิงปฏิบัติ (Step 616)

จากการทดสอบทั้งหมดในบทเรียนนี้ สรุปเป็นข้อคิดสำหรับการทำงานจริงได้ดังนี้:

1. **อย่าเชื่อว่า `-std=c++23` แปลว่าใช้ฟีเจอร์ C++23 ได้ทุกตัว** — แฟล็กนี้แค่บอก compiler
   ว่า "รองรับ syntax ระดับภาษาตามมาตรฐานนี้ให้มากที่สุดเท่าที่ทำได้" ไม่ได้รับประกันว่า
   standard library (libstdc++/libc++) ที่มากับคอมไพเลอร์เวอร์ชันนั้นจะ implement ทุก
   header/class ที่มาตรฐานกำหนดไว้ครบถ้วน อย่างที่เห็นจาก `<print>`, `<mdspan>`,
   `<flat_map>` ที่ขาดหายไปเลยบน g++ 13.3.0 ทั้งที่ compiler ตัวเดียวกันรองรับฟีเจอร์ระดับ
   ภาษาอื่นๆ ของมาตรฐานเดียวกันได้

2. **แยกให้ออกระหว่าง "ฟีเจอร์ภาษา" (compiler frontend) กับ "ฟีเจอร์ library"** — deducing
   this เป็นเรื่องของ compiler frontend ล้วนๆ (parser ต้องเข้าใจ syntax ใหม่) ส่วน
   `std::print`/`std::mdspan` เป็นเรื่องของ standard library implementation ทั้งสองส่วนนี้
   พัฒนาไปคนละความเร็วกัน แม้จะอยู่ใน compiler suite เดียวกัน (สังเกตได้จากที่ Clang 18.1.3
   รองรับ deducing this ได้ แต่ก็ยังไม่มี `<print>` เพราะพึ่ง standard library ของ GCC)

3. **ใช้ feature-test macro ควบคู่กับการทดลองคอมไพล์จริงเสมอ** อย่างที่พบใน 77.5:
   บางครั้ง macro ก็ไม่ตรงกับความสามารถจริงของ compiler โดยเฉพาะเมื่อผสม compiler
   frontend ตัวหนึ่งกับ standard library ของอีกค่ายหนึ่ง (เช่น Clang + libstdc++)

4. **ก่อนใช้ฟีเจอร์ C++23 ใดๆ ในโปรเจกต์จริง ให้ทดสอบคอมไพล์บน toolchain ที่ทีมใช้จริงก่อน
   เสมอ** อย่าเชื่อจากเอกสารหรือบทความออนไลน์อย่างเดียว เพราะสถานะการรองรับเปลี่ยนแปลงเร็ว
   มากในแต่ละเวอร์ชันของ GCC/Clang/MSVC และอาจต่างจากที่บทความเขียนไว้ ณ เวลาที่เขียน

5. **โปรเจกต์ที่ยึด g++ เวอร์ชันเก่ากว่านี้เป็นหลัก** ควรใช้ฟีเจอร์ที่ยืนยันว่าใช้ได้แน่นอน
   ก่อน (`std::expected`, multidimensional subscript, `if consteval`, ranges views ใหม่)
   ส่วนฟีเจอร์ที่ยังไม่รองรับ (deducing this, `std::print`, `std::mdspan`, `std::flat_map`)
   ให้เขียนโค้ดแบบเดิมไปพลางก่อน (const/non-const overload คู่, `std::cout`, เขียน matrix
   view เอง, ใช้ `std::map`/`std::vector<std::pair<...>>` แทน) แล้วค่อยย้ายมาใช้เมื่อ
   อัปเกรด toolchain ในอนาคต

---

## ข้อผิดพลาดที่พบบ่อย (Common Pitfalls)

1. **คิดว่า `-std=c++23` รับประกันว่าฟีเจอร์ C++23 ทุกตัวใช้ได้** — ตามที่พิสูจน์ในบทนี้
   `<print>`, `<mdspan>`, `<flat_map>` และ deducing this ล้วนคอมไพล์ไม่ผ่านบน g++ 13.3.0
   แม้จะเปิด `-std=c++23` แล้วก็ตาม ต้องตรวจสอบทีละฟีเจอร์เสมอ

2. **ใช้ `matrix[i, j]` แล้วคาดหวังผลแบบ C++23 ทั้งที่ยังคอมไพล์ด้วย `-std=c++17`/`c++20`
   อยู่** — จะไม่ error แต่ความหมายเปลี่ยนไปเป็น comma operator ตามที่อธิบายใน 77.3
   ทำให้ได้ผลลัพธ์ที่ผิดแบบเงียบๆ โดยไม่มี error หรือ warning เตือนชัดเจน (นี่คือหนึ่งใน
   กรณีอันตรายที่สุด เพราะโค้ดคอมไพล์ผ่านแต่ทำงานผิด)

3. **ใช้ `std::stacktrace` แล้วลืม link `-lstdc++_libbacktrace`** — จะคอมไพล์ผ่านปกติ
   (เพราะ header ประกาศทุกอย่างไว้ครบ) แต่ error ตอน link ด้วย
   `undefined reference to '__glibcxx_backtrace_create_state'` มือใหม่มักงงว่าทำไม
   header compile ผ่านแต่ link ไม่ผ่าน (ทบทวนความแตกต่างระหว่าง compile error กับ link
   error ได้จาก Part 1)

4. **สับสนระหว่าง `std::optional` กับ `std::expected`** — ใช้ `std::optional<T>` ในกรณีที่
   ต้องการรายงาน**สาเหตุ**ของความล้มเหลว ทำให้ผู้เรียกโค้ดไม่รู้ว่าเกิดอะไรขึ้นจริง ควรเปลี่ยน
   ไปใช้ `std::expected<T, E>` เมื่อความล้มเหลวมีได้หลายสาเหตุที่ผู้เรียกควรรู้และจัดการ
   ต่างกัน

5. **เชื่อ feature-test macro 100% โดยไม่ทดลองคอมไพล์จริง** — ตามที่พบใน 77.5 ว่า Clang
   ที่ใช้ libstdc++ ของ GCC ไม่ได้กำหนด `__cpp_explicit_this_parameter` ทั้งที่ syntax
   ใช้งานได้จริง การพึ่ง macro เพียงอย่างเดียวโดยไม่เคยลองคอมไพล์อาจทำให้ปิดการใช้ฟีเจอร์ที่
   จริงๆ ใช้ได้ (หรือกลับกัน คิดว่าใช้ได้ทั้งที่จริงไม่ได้) อย่างผิดพลาด

6. **ลืมว่า `if consteval` ต้องเขียนแบบ block `{ }` ไม่ใช่ expression ในวงเล็บ** — เขียน
   `if (consteval)` แบบ `if` ปกติจะเป็น syntax error ทันที เพราะ `consteval` ในบริบทนี้
   ไม่ใช่ identifier ที่ประเมินค่าได้ แต่เป็นส่วนหนึ่งของ keyword compound `if consteval`

---

## แบบฝึกหัดท้ายบท

1. เขียนฟังก์ชัน `std::expected<double, std::string> safe_divide(double a, double b)` ที่
   คืนค่า error `"division by zero"` เมื่อ `b == 0` แล้วทดสอบเรียกทั้งกรณีสำเร็จและล้มเหลว

2. เขียน class `Grid3D` ที่เก็บข้อมูล 3 มิติ (`x`, `y`, `z`) โดยใช้ multidimensional
   subscript operator `operator[](int x, int y, int z)` ให้เข้าถึงข้อมูลได้แบบ
   `grid[1, 2, 3]`

3. เขียนฟังก์ชัน `constexpr` ที่ใช้ `if consteval` เพื่อเลือกวิธีคำนวณค่า factorial ต่างกัน
   ระหว่าง compile-time (ใช้ recursion ธรรมดา) กับ runtime (ใช้ loop) แล้วพิสูจน์ด้วยการ
   print ผลลัพธ์ทั้งสองกรณี

4. ใช้ `std::views::enumerate` (หรือเขียนแบบ manual ถ้าคอมไพเลอร์ที่ใช้ไม่รองรับ) เพื่อ
   พิมพ์ index คู่กับค่าของ `std::vector<std::string>` ที่มีชื่อผลไม้ 5 ชนิด

5. เขียน pipeline โดยใช้ monadic operations ของ `std::expected` (`.and_then()`,
   `.transform()`) ที่ต่อกัน 3 ขั้นตอน (เช่น parse string เป็นตัวเลข → ตรวจสอบว่าเป็นค่าบวก
   → คูณด้วย 2) โดยแต่ละขั้นตอนสามารถ fail และรายงานข้อความ error ต่างกันได้

6. (ท้าทาย) ตรวจสอบบนเครื่องของผู้เรียนเองว่า compiler ที่ติดตั้งอยู่ (ไม่ว่าจะเป็น GCC,
   Clang, หรือ MSVC เวอร์ชันใดก็ตาม) รองรับ deducing this และ `std::print` หรือยัง โดยใช้
   ทั้ง feature-test macro และการทดลองคอมไพล์โค้ดจริง เปรียบเทียบผลกับที่รายงานไว้ในบทเรียน
   นี้ (g++ 13.3.0 และ Clang 18.1.3)

### แนวทางเฉลยข้อ 1

```cpp
#include <expected>
#include <iostream>
#include <string>

std::expected<double, std::string> safe_divide(double a, double b) {
    if (b == 0.0) {
        return std::unexpected("division by zero");
    }
    return a / b;
}

int main() {
    auto r1 = safe_divide(10.0, 2.0);
    auto r2 = safe_divide(5.0, 0.0);

    if (r1) {
        std::cout << "10 / 2 = " << *r1 << "\n";
    }
    if (!r2) {
        std::cout << "5 / 0 error: " << r2.error() << "\n";
    }
}
```

```bash
g++ -Wall -Wextra -std=c++23 safe_divide.cpp -o safe_divide
./safe_divide
```

ทดสอบคอมไพล์และรันจริง ได้ผลลัพธ์:

```
10 / 2 = 5
5 / 0 error: division by zero
```

### แนวทางเฉลยข้อ 3

```cpp
#include <iostream>

constexpr long long factorial(int n) {
    if consteval {
        // ตอน compile-time: recursion ธรรมดา ไม่มีต้นทุน runtime เพราะคำนวณเสร็จตั้งแต่
        // ตอนคอมไพล์ ไม่หลงเหลือ call stack จริงในโปรแกรมที่รันสักนิดเดียว
        return (n <= 1) ? 1 : n * factorial(n - 1);
    } else {
        // ตอน runtime: ใช้ loop เพื่อหลีกเลี่ยงต้นทุนของการเรียกฟังก์ชันซ้อนกันจริงๆ
        long long result = 1;
        for (int i = 2; i <= n; ++i) {
            result *= i;
        }
        return result;
    }
}

int main() {
    constexpr long long compile_time_result = factorial(10);   // ใช้ path compile-time
    int n = 10;
    long long runtime_result = factorial(n);                    // ใช้ path runtime

    std::cout << "compile-time: " << compile_time_result << "\n";
    std::cout << "runtime:      " << runtime_result << "\n";
}
```

```bash
g++ -Wall -Wextra -std=c++23 factorial_consteval.cpp -o factorial_consteval
./factorial_consteval
```

ทดสอบคอมไพล์และรันจริง ได้ผลลัพธ์ (ทั้งสอง path คำนวณค่าเดียวกันได้ถูกต้อง แม้จะใช้
อัลกอริทึมคนละแบบกัน เพราะ `10!` มีค่าเท่ากันไม่ว่าจะคำนวณด้วยวิธีไหน):

```
compile-time: 3628800
runtime:      3628800
```

---

## สรุปท้ายบท

ใน Part นี้เราได้:

- เข้าใจภาพรวมของ C++23 ว่าเป็นการอัปเดตที่เน้นปรับปรุง Standard Library และเติมเต็ม
  ช่องว่างของ C++20 มากกว่าจะเป็นการปฏิวัติภาษาแบบ C++20
- เรียนรู้และทดสอบ `std::expected` สำหรับ error handling แบบไม่ throw exception พร้อม
  monadic operations (`and_then`, `transform`) ที่คอมไพล์และรันได้จริง
- เขียน multidimensional subscript operator (`matrix[i, j]`) ได้จริง พร้อมเข้าใจ
  ข้อควรระวังเรื่อง comma operator เมื่อคอมไพล์ด้วยมาตรฐานเก่ากว่า
- ใช้ `if consteval` แยกพฤติกรรม compile-time/runtime ของฟังก์ชันได้จริง
- เข้าใจแนวคิดของ deducing this และ `std::print`/`std::println` อย่างถูกต้อง แม้จะทดสอบ
  ยืนยันแล้วว่ายังใช้ไม่ได้บน g++ 13.3.0 (deducing this ใช้ได้บน Clang 18.1.3 ที่ทดสอบ
  เทียบ ส่วน `std::print` ยังใช้ไม่ได้ทั้งสองคอมไพเลอร์บนเครื่องนี้)
- สำรวจฟีเจอร์อื่นๆ ของ C++23 ที่ทดสอบแล้วว่าใช้งานได้จริง (`std::stacktrace`,
  ranges views ใหม่, static `operator()`, literal suffix `uz`, utility functions ต่างๆ)
  และที่ยังใช้ไม่ได้ (`std::mdspan`, `std::flat_map`/`flat_set`)
- ได้บทเรียนสำคัญเรื่องความน่าเชื่อถือของ feature-test macro และความจำเป็นที่ต้องทดลอง
  คอมไพล์จริงก่อนเชื่อว่าฟีเจอร์ใดใช้งานได้บน toolchain ที่ใช้อยู่จริง

การทดสอบทุกตัวอย่างในบทเรียนนี้อย่างจริงจังแทนที่จะเขียนตามทฤษฎีล้วนๆ แสดงให้เห็นความจริง
ที่สำคัญของวงการ C++: **มาตรฐานภาษาที่ประกาศออกมาไม่ได้แปลว่าคอมไพเลอร์ทุกตัวจะรองรับทันที**
การเป็นวิศวกร C++ มืออาชีพต้องรู้จักตรวจสอบและพิสูจน์ ไม่ใช่แค่เชื่อตามเอกสาร ทักษะนี้จะ
ติดตัวผู้เรียนไปตลอดเมื่อต้องทำงานกับ toolchain ที่หลากหลายในโลกจริง

ใน **Part 78** เราจะเปลี่ยนไปสำรวจโลกของ **Metaprogramming** เจาะลึก SFINAE, type_traits,
และการทำ Concept-based Dispatch ซึ่งเป็นเทคนิคขั้นสูงที่ผสมผสานทุกอย่างที่เรียนมาตลอด
Module F เข้าด้วยกัน

**ต่อไป:** [Part 78 — Metaprogramming](./part-078-metaprogramming.md)
