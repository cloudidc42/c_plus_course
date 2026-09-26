# Part 72: ฟีเจอร์ C++14/17 (Step 569–576)

> Module F — Modern C++ (C++11 ถึง C++23) | Part 72 จาก 125
> Step ที่ครอบคลุมใน Part นี้: Step 569–576
> Part ก่อนหน้า: [Part 71 — constexpr และ Compile-Time Programming](./part-071-constexpr.md) | Part ถัดไป: [Part 73 — C++20 Concepts](./part-073-cpp20-concepts.md)

## เป้าหมายการเรียนรู้ (Learning Objectives)

หลังจบ Part นี้ ผู้เรียนจะสามารถ:

1. ทบทวนและใช้งาน Generic Lambda และอธิบายได้ว่า `constexpr` ใน C++14 "ผ่อนคลาย" กฎจาก C++11
   อย่างไร จนเขียนฟังก์ชัน compile-time ที่มี loop และตัวแปรท้องถิ่นได้
2. ใช้ `std::make_unique` แทนการเขียน `new` ตรงๆ และใช้ Binary Literal กับ Digit Separator
   เพื่อเขียนโค้ดตัวเลขที่อ่านง่ายขึ้น
3. ใช้ Structured Bindings (`auto [a, b] = ...`) แกะค่าจาก `pair`, `map`, และ `tuple` ได้อย่าง
   คล่องแคล่ว แทนการเขียน `.first`/`.second` หรือ `std::get<N>` แบบเดิม
4. ใช้ `if constexpr` เขียน Template ที่แตกกิ่งพฤติกรรมตาม Type ได้ตั้งแต่ตอน Compile โดยไม่ต้อง
   พึ่ง SFINAE ที่ซับซ้อน
5. ใช้ `std::optional` แทนการใช้ Pointer หรือ Sentinel Value เพื่อสื่อความหมาย "อาจไม่มีค่า"
   ได้อย่างปลอดภัยและชัดเจน
6. ใช้ `std::variant` เป็น Type-Safe Union แทน `union` ธรรมดาที่เรียนใน Part 10 และเข้าใจว่า
   ทำไมมันปลอดภัยกว่า
7. ใช้ `std::any`, `inline variable`, และ `std::filesystem` เบื้องต้นเพื่อแก้ปัญหาที่พบบ่อยใน
   โค้ดจริง เช่น การเก็บค่าที่ type ไม่แน่นอน การประกาศตัวแปร global ใน header และการอ่านไฟล์
   ในโฟลเดอร์

---

## บริบท: ทำไมต้องมี C++14 และ C++17

C++11 (ที่เราเรียนภาพรวมไปใน Part 69) เป็นการปฏิวัติครั้งใหญ่ของภาษา C++ แต่การเปลี่ยนแปลง
ขนาดใหญ่ขนาดนั้นย่อมทิ้ง "ช่องว่าง" และ "จุดที่ยังไม่ลงตัว" ไว้หลายจุด คณะกรรมการมาตรฐาน
C++ (ISO/IEC JTC1/SC22/WG21) จึงออกมาตรฐานสองฉบับถัดมาเพื่ออุดช่องว่างเหล่านั้น:

- **C++14 (2014)** — มาตรฐาน "แก้ไขเล็กน้อย" (Minor Release) เน้นเติมเต็มสิ่งที่ C++11 ทำได้
  ไม่สมบูรณ์ เช่น Lambda ที่ยังไม่รองรับ `auto` parameter, `constexpr` ที่เข้มงวดเกินไป, และ
  `std::make_shared` ที่มีแต่เวอร์ชัน shared_ptr ไม่มีเวอร์ชัน unique_ptr
- **C++17 (2017)** — มาตรฐาน "ขนาดกลาง" ที่เพิ่มฟีเจอร์ใหม่จำนวนมากที่โปรแกรมเมอร์ใช้งานจริง
  ทุกวันจนถึงปัจจุบัน (2026) เช่น Structured Bindings, `if constexpr`, `std::optional`,
  `std::variant`, และ `std::filesystem`

ทั้งสองมาตรฐานนี้ไม่ได้ "เปลี่ยนวิธีคิด" ของภาษาแบบ C++11 หรือ C++20 แต่เป็นเหมือนชุด
เครื่องมือช่างที่ทำให้งานเขียนโค้ดประจำวันสั้นลง อ่านง่ายขึ้น และปลอดภัยขึ้นอย่างเห็นได้ชัด
ฟีเจอร์ในบทนี้แทบทั้งหมดเป็นสิ่งที่โปรแกรมเมอร์ C++ มืออาชีพในปี 2026 ใช้งานทุกวัน

---

## 72.1 ทบทวน C++14: Generic Lambda และ Relaxed constexpr (Step 569)

### Generic Lambda (ทบทวนจาก Part 64)

ใน Part 64 เราเรียนเรื่อง Lambda Expression ไปแล้ว ซึ่งตอนนั้น Lambda ของ C++11 ต้องระบุ Type
ของพารามิเตอร์ตรงๆ เช่น `[](int a, int b) { return a + b; }` ปัญหาคือถ้าอยากได้ Lambda ที่ใช้ได้
กับหลาย Type (เช่น ทั้ง `int` และ `double`) ต้องเขียนซ้ำหลายตัว หรือใช้ Function Template แทน

C++14 แก้ปัญหานี้ด้วย **Generic Lambda**: ใช้ `auto` แทน Type ของพารามิเตอร์ได้ ทำให้ Lambda
กลายเป็นเหมือน Function Template แบบย่อในตัว:

```cpp
#include <iostream>
#include <string>

int main() {
    auto add = [](auto a, auto b) {
        return a + b;
    };

    std::cout << add(3, 4) << "\n";
    std::cout << add(2.5, 1.5) << "\n";
    std::cout << add(std::string("Hello, "), std::string("C++14!")) << "\n";
    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 generic_lambda.cpp -o generic_lambda
./generic_lambda
```

ผลลัพธ์:

```
7
4
Hello, C++14!
```

เบื้องหลัง Compiler จะสร้าง `operator()` แบบ Template ให้กับ Closure Object ของ Lambda โดย
อัตโนมัติ (คล้ายกับที่เราเขียน Function Template เอง) พูดง่ายๆ คือ `add` ในตัวอย่างนี้คือ
Object ที่มี Member Function หน้าตาประมาณ:

```cpp
struct __lambda_add {
    template <typename T, typename U>
    auto operator()(T a, U b) const {
        return a + b;
    }
};
```

### Relaxed constexpr

ใน Part 71 เราเรียนเรื่อง `constexpr` ไปแล้ว โดยกฎของ C++11 นั้นเข้มงวดมาก: ฟังก์ชัน
`constexpr` ต้องมี **statement เดียว** ที่เป็น `return` เท่านั้น (ห้ามมี loop, ห้ามประกาศตัวแปร
ท้องถิ่นที่เปลี่ยนค่าได้, ห้ามใช้ `if` แบบมี body) ทำให้การเขียนฟังก์ชันแบบวนซ้ำต้องเขียนเป็น
Recursion เท่านั้น

C++14 "ผ่อนคลาย" กฎเหล่านี้ให้ฟังก์ชัน `constexpr` เขียนได้เกือบเหมือนฟังก์ชันปกติทุกอย่าง
(มี loop, มีตัวแปรท้องถิ่นหลายตัว, มี `if`/`else` ได้เต็มรูปแบบ) ขอแค่ผลลัพธ์สุดท้ายคำนวณได้
จริงตอน Compile-Time:

```cpp
#include <iostream>

// C++14: constexpr function มี loop, ตัวแปรท้องถิ่น, if ได้ (C++11 ทำแบบนี้ไม่ได้)
constexpr long long factorial(int n) {
    long long result = 1;
    for (int i = 2; i <= n; ++i) {
        result *= i;
    }
    return result;
}

// C++11 style เดิม: ต้องเขียนเป็น expression เดียว (recursive)
constexpr long long factorial_cpp11(int n) {
    return (n <= 1) ? 1 : n * factorial_cpp11(n - 1);
}

int main() {
    constexpr long long f10 = factorial(10);
    constexpr long long f10_old = factorial_cpp11(10);
    static_assert(f10 == 3628800, "factorial(10) ต้องเท่ากับ 3628800");

    std::cout << "10! (C++14 style) = " << f10 << "\n";
    std::cout << "10! (C++11 style) = " << f10_old << "\n";
    return 0;
}
```

ผลลัพธ์:

```
10! (C++14 style) = 3628800
10! (C++11 style) = 3628800
```

ทั้งสองฟังก์ชันให้ผลลัพธ์เหมือนกันและถูกคำนวณตอน Compile-Time เหมือนกัน (สังเกตว่าประกาศเป็น
`constexpr long long f10 = factorial(10);` ได้ ซึ่งบังคับให้ Compiler ต้องคำนวณค่าให้เสร็จก่อน
รันโปรแกรม) แต่เวอร์ชัน C++14 อ่านง่ายกว่ามาก และไม่มีความเสี่ยงเรื่อง Stack Overflow จาก
Recursion ลึกๆ เหมือนเวอร์ชัน C++11 หากดันไปเรียกตอน Runtime กับค่า `n` ที่มากๆ

> **หมายเหตุ**: ทั้งสองฟังก์ชันนี้ยังเรียกตอน Runtime ได้ปกติด้วย (เช่น `factorial(user_input)`
> ที่ `user_input` ไม่รู้ค่าตอน Compile) — คีย์เวิร์ด `constexpr` แปลว่า "คำนวณตอน Compile-Time
> **ได้** ถ้าจำเป็น" ไม่ใช่ "ต้องคำนวณตอน Compile-Time เสมอ"

---

## 72.2 std::make_unique และไวยากรณ์ตัวเลขใหม่ (Step 570)

### std::make_unique: เติมช่องว่างจาก Part 67

ใน Part 67 เราเรียนเรื่อง Smart Pointer และ `std::make_shared` ไปแล้ว ซึ่ง `std::make_shared`
มีมาตั้งแต่ C++11 แต่ `std::make_unique` (คู่หูของ `std::unique_ptr`) กลับตกหล่นไปจนถึง C++14
เหตุผลคือทีมมาตรฐานพลาดเวลาเข้าโค้งสุดท้ายของ C++11 ไม่ทัน จึงต้องรอมาเพิ่มใน C++14

```cpp
#include <iostream>
#include <memory>
#include <string>

struct Player {
    std::string name;
    int hp;
    Player(std::string n, int h) : name(std::move(n)), hp(h) {}
};

int main() {
    // std::make_unique (C++14) เติมช่องว่างที่ std::make_shared มีมาตั้งแต่ C++11
    auto p = std::make_unique<Player>("Arthas", 100);
    std::cout << p->name << " HP=" << p->hp << "\n";

    // Binary literal (C++14): เขียนเลขฐาน 2 ตรงๆ ด้วย 0b
    unsigned char flags = 0b1010'1100;   // digit separator (') ช่วยอ่านง่าย
    int million = 1'000'000;             // digit separator ใช้กับเลขฐาน 10 ได้เช่นกัน

    std::cout << "flags = " << static_cast<int>(flags) << "\n";
    std::cout << "million = " << million << "\n";
    return 0;
}
```

ผลลัพธ์:

```
Arthas HP=100
flags = 172
million = 1000000
```

**ทำไมต้องใช้ `make_unique` แทน `new` ตรงๆ?** เหตุผลเดียวกับที่เรียนไปแล้วใน Part 67 เรื่อง
`make_shared`:

| ประเด็น | `std::unique_ptr<T> p(new T(...))` | `auto p = std::make_unique<T>(...)` |
|---|---|---|
| ความปลอดภัยเมื่อมี Exception | เสี่ยง Memory Leak ถ้า `new` สำเร็จแต่โค้ดข้างเคียงโยน Exception ก่อนถูกส่งเข้า `unique_ptr` | ปลอดภัยกว่า เพราะการสร้าง Object กับการห่อด้วย Smart Pointer เกิดในฟังก์ชันเดียวกัน |
| ความยาวโค้ด | ต้องพิมพ์ชื่อ Type ซ้ำสองครั้ง (`unique_ptr<T>` และ `new T`) | พิมพ์ชื่อ Type ครั้งเดียว |
| Exception Safety โดยรวม | ต่ำกว่า | สูงกว่า — เป็น Best Practice ที่แนะนำใน C++ Core Guidelines |

### Binary Literal และ Digit Separator

C++14 เพิ่มไวยากรณ์เล็กๆ สองอย่างที่ช่วยเรื่องการอ่านโค้ดตัวเลข:

- **Binary Literal (`0b...`)**: ก่อนหน้านี้เขียนเลขฐาน 2 ตรงๆ ในโค้ดไม่ได้ (ต้องเขียนเป็น
  Hex แล้วแปลงเอง เช่น `0xAC` แทน `10101100`) ตอนนี้เขียน `0b10101100` ได้ตรงๆ ซึ่งมีประโยชน์
  มากตอนทำงานกับ Bit Flag (ทบทวน Bit Manipulation ได้ที่ Part 15)
- **Digit Separator (`'`)**: ใช้ Single Quote คั่นตัวเลขยาวๆ ให้อ่านง่ายขึ้น โดย Compiler จะ
  มองข้ามเครื่องหมายนี้ไปเลย ไม่มีผลต่อค่าจริงของตัวเลข ใช้ได้ทั้งเลขฐาน 10, ฐาน 16, ฐาน 2

```cpp
int a = 1'000'000;        // อ่านง่ายกว่า 1000000
long b = 0xFF'FF'FF'FFL;  // คั่น hex ทีละ byte
unsigned c = 0b1111'0000; // คั่น binary ทีละ nibble (4 bit)
```

---

## 72.3 Structured Bindings (Step 571)

Structured Bindings คือฟีเจอร์ C++17 ที่ทำให้ "แกะ" ค่าจาก `struct`, `pair`, `tuple` หรือ
Array ออกมาเป็นตัวแปรแยกกันได้ในบรรทัดเดียว ด้วยไวยากรณ์ `auto [a, b, c] = ...`

### กับ std::pair

ก่อนหน้านี้การเข้าถึงค่าใน `std::pair` ต้องใช้ `.first` และ `.second` ซึ่งไม่สื่อความหมาย:

```cpp
std::pair<std::string, int> student{"Somchai", 95};
std::cout << student.first << " ได้คะแนน " << student.second << "\n";  // อ่านไม่รู้เรื่องว่า first/second คืออะไร
```

ด้วย Structured Bindings เขียนได้ชัดเจนกว่ามาก:

```cpp
auto [name, score] = student;
std::cout << name << " ได้คะแนน " << score << "\n";
```

### กับ std::map (กรณีใช้บ่อยที่สุด)

การวน Loop ผ่าน `std::map` ที่เรียนใน Part 61 แบบเดิมต้องเขียน `it->first` / `it->second`:

```cpp
for (const auto& it : ages) {
    std::cout << it.first << " อายุ " << it.second << " ปี\n";   // อ่านยาก
}
```

ส่วนแบบ Structured Bindings อ่านเหมือนภาษาพูดตรงๆ:

```cpp
for (const auto& [person, age] : ages) {
    std::cout << person << " อายุ " << age << " ปี\n";
}
```

### กับ std::tuple

```cpp
std::tuple<int, int, int> get_rgb() {
    return {255, 128, 0};
}

auto [r, g, b] = get_rgb();
std::cout << "RGB = (" << r << ", " << g << ", " << b << ")\n";
```

### ตัวอย่างเต็ม

```cpp
#include <iostream>
#include <map>
#include <string>
#include <tuple>

std::tuple<int, int, int> get_rgb() {
    return {255, 128, 0};
}

int main() {
    // 1) กับ std::pair
    std::pair<std::string, int> student{"Somchai", 95};
    auto [name, score] = student;
    std::cout << name << " ได้คะแนน " << score << "\n";

    // 2) กับ std::map — วิธีเดิมต้องใช้ it->first / it->second
    std::map<std::string, int> ages{{"Anan", 20}, {"Bee", 22}};
    for (const auto& [person, age] : ages) {
        std::cout << person << " อายุ " << age << " ปี\n";
    }

    // 3) กับ std::tuple
    auto [r, g, b] = get_rgb();
    std::cout << "RGB = (" << r << ", " << g << ", " << b << ")\n";

    // 4) insert คืนค่า pair<iterator, bool> — เอา bool มาเช็คได้ทันที
    auto [it, inserted] = ages.insert({"Anan", 999});
    std::cout << "insert 'Anan' อีกครั้ง สำเร็จหรือไม่: "
              << std::boolalpha << inserted
              << " (ค่าที่มีอยู่: " << it->second << ")\n";

    return 0;
}
```

ผลลัพธ์:

```
Somchai ได้คะแนน 95
Anan อายุ 20 ปี
Bee อายุ 22 ปี
RGB = (255, 128, 0)
insert 'Anan' อีกครั้ง สำเร็จหรือไม่: false (ค่าที่มีอยู่: 20)
```

ตัวอย่างที่ 4 แสดงประโยชน์อีกแบบของ Structured Bindings: หลายฟังก์ชันของ STL (เช่น
`map::insert`) คืนค่าเป็น `pair` มาตั้งแต่ C++11 แต่ก่อน C++17 การเช็คว่า insert สำเร็จหรือไม่
ต้องเขียน `result.second` ซึ่งไม่สื่อความหมาย ตอนนี้ตั้งชื่อ `inserted` ให้ตรงกับความหมายได้เลย

> **ข้อควรรู้เชิงลึก**: Structured Bindings ไม่ได้สร้างตัวแปรใหม่ที่ "แยก" จากต้นฉบับเสมอไป
> ถ้าประกาศด้วย `auto&` ตัวแปรที่แกะออกมาจะเป็น Reference ไปยังค่าจริงในต้นฉบับ ทำให้แก้ไขค่า
> ผ่านตัวแปรที่แกะออกมาได้ เช่น `for (auto& [k, v] : my_map) v *= 2;` จะแก้ไขค่าใน `my_map`
> จริงๆ ส่วนถ้าใช้ `auto` เฉยๆ (ไม่มี `&`) จะเป็นการ Copy ค่าออกมา

---

## 72.4 if constexpr: Compile-Time Branching (Step 572)

### ปัญหาที่ if constexpr แก้

ลองนึกภาพ Function Template ที่อยากให้ทำงานต่างกันตาม Type ของ `T`:

```cpp
template <typename T>
std::string describe_bad(const T& value) {
    if (std::is_integral_v<T>) {                 // if ธรรมดา ไม่ใช่ if constexpr
        return "จำนวนเต็ม: " + std::to_string(value);
    } else {
        return "ค่าอื่น ๆ: " + std::to_string(value);  // compile พังเมื่อ T=std::string
    }
}
```

ปัญหาคือ `if` ธรรมดาเป็นการตัดสินใจตอน **Runtime** ซึ่งหมายความว่า Compiler ต้อง Generate
โค้ดของทั้งสองกิ่งออกมาเสมอ ไม่ว่ากิ่งไหนจะถูกเลือกใช้จริงหรือไม่ ถ้าเรียก
`describe_bad(std::string("hi"))` โค้ดกิ่ง `else` ที่เรียก `std::to_string(value)` จะถูก
Instantiate ด้วย `T = std::string` ซึ่ง `std::to_string` ไม่มี Overload ที่รับ `std::string`
เข้ามา ทำให้ Compile ไม่ผ่านทั้งที่ตอน Runtime กิ่งนั้นจะไม่มีทางถูกเลือก (เพราะ
`std::is_integral_v<std::string>` เป็น `false` เสมอ):

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 -c if_bad.cpp
```

Error จริงที่ได้ (Compiler แจ้งจากจุดที่ Instantiate Template ด้วย `T = std::string`):

```
if_bad.cpp: In instantiation of 'std::string describe_bad(const T&) [with T = std::__cxx11::basic_string<char>]':
if_bad.cpp:15:30:   required from here
if_bad.cpp:8:45: error: no matching function for call to 'to_string(const std::__cxx11::basic_string<char>&)'
    8 |         return "จำนวนเต็ม: " + std::to_string(value);
      |                               ~~~~~~~~~~~~~~^~~~~~~
```

### วิธีแก้: if constexpr

C++17 เพิ่ม `if constexpr` ซึ่งเป็นการตัดสินใจตอน **Compile-Time** — กิ่งที่ไม่ถูกเลือกจะ
ถูก "ตัดทิ้ง" ไปเลย ไม่ถูก Instantiate ให้เป็นโค้ดจริง จึงไม่มีปัญหาเรื่อง Type ไม่ตรงกัน:

```cpp
#include <iostream>
#include <string>
#include <type_traits>

template <typename T>
std::string describe(const T& value) {
    if constexpr (std::is_integral_v<T>) {
        return "จำนวนเต็ม: " + std::to_string(value);
    } else if constexpr (std::is_floating_point_v<T>) {
        return "จำนวนทศนิยม: " + std::to_string(value);
    } else {
        return "ค่าอื่น ๆ (ไม่ใช่ตัวเลข)";
    }
}

int main() {
    std::cout << describe(42) << "\n";
    std::cout << describe(3.14) << "\n";
    std::cout << describe(std::string("hi")) << "\n";
    return 0;
}
```

ผลลัพธ์:

```
จำนวนเต็ม: 42
จำนวนทศนิยม: 3.140000
ค่าอื่น ๆ (ไม่ใช่ตัวเลข)
```

เมื่อ Compiler ประมวลผล `describe(std::string("hi"))` มันจะเห็นว่า `T = std::string` ทำให้
`std::is_integral_v<T>` และ `std::is_floating_point_v<T>` เป็น `false` ทั้งคู่ Compiler จึง
ตัดโค้ดสองกิ่งแรกทิ้งไปเลย เหลือแค่กิ่ง `else` เท่านั้นที่ถูก Instantiate จริง — โค้ดที่เรียก
`std::to_string` กับ `std::string` จึงไม่มีทางถูกสร้างขึ้นมา ไม่มี Error

### เปรียบเทียบ if ธรรมดา กับ if constexpr

| ประเด็น | `if` ธรรมดา | `if constexpr` |
|---|---|---|
| เวลาตัดสินใจ | Runtime | Compile-Time |
| โค้ดที่ไม่ถูกเลือก | ยังถูก Generate และต้อง Compile ผ่านทุกกิ่ง | ถูกตัดทิ้ง ไม่ต้อง Compile ผ่านก็ได้ (ถ้าไม่ถูกเลือก) |
| ใช้ได้กับเงื่อนไขแบบไหน | เงื่อนไขอะไรก็ได้ รวมถึงค่าที่รู้เฉพาะตอน Runtime | ต้องเป็นเงื่อนไขที่ประเมินผลได้ตอน Compile-Time เท่านั้น (เช่น `type_traits`, `sizeof`, `constexpr` อื่นๆ) |
| ใช้แทน SFINAE ได้ไหม | ไม่ได้ | ได้ในหลายกรณี และอ่านง่ายกว่า SFINAE มาก |

> **ทำไมไม่ใช้ SFINAE ไปเลย?** SFINAE (Substitution Failure Is Not An Error) เป็นเทคนิคเก่า
> ที่ทำสิ่งคล้ายกันได้ด้วย `std::enable_if` แต่ไวยากรณ์ซับซ้อนและอ่านยากกว่า `if constexpr`
> มาก — เราจะกลับมาดู SFINAE แบบเจาะลึกอีกครั้งใน Part 78 (Metaprogramming) เพื่อเข้าใจว่า
> `if constexpr` แก้ปัญหาเดียวกันได้ง่ายกว่าอย่างไร

---

## 72.5 std::optional (Step 573)

### ปัญหาที่ optional แก้

ลองนึกภาพฟังก์ชันค้นหาข้อมูลที่ "อาจจะไม่พบ" ก่อน C++17 มีสองวิธีหลักที่ใช้แก้ปัญหานี้:

1. **คืนค่าเป็น Pointer** — ใช้ `nullptr` แทน "ไม่พบ" แต่ผู้เรียกต้องจำให้ได้ทุกครั้งว่าต้อง
   เช็ค `nullptr` ก่อนใช้ ไม่งั้นโปรแกรมจะ Crash (Dereference Null Pointer)
2. **ใช้ Sentinel Value** — เช่นคืนค่า `-1` แทน "ไม่พบ" ซึ่งใช้ไม่ได้ถ้าค่า `-1` เป็นค่าที่
   ถูกต้องได้จริงในโดเมนของปัญหา (เช่นอุณหภูมิติดลบ)

ทั้งสองวิธีมีปัญหาร่วมกันคือ **Type ของค่าที่คืนไม่ได้สื่อความหมายว่า "อาจไม่มีค่า" อยู่ในตัวเอง**
ผู้เรียกต้องรู้เอง (จากอ่านเอกสารหรือชื่อฟังก์ชัน) ว่าต้องเช็คอะไรบ้าง

### วิธีแก้: std::optional<T>

`std::optional<T>` เป็น Wrapper ที่ "อาจจะมี" หรือ "อาจจะไม่มี" ค่าของ Type `T` อยู่ข้างใน
Type ของมันเองสื่อความหมายชัดเจนตั้งแต่ Signature ของฟังก์ชัน:

```cpp
#include <iostream>
#include <optional>
#include <string>
#include <vector>

struct User {
    std::string name;
    int age;
};

std::vector<User> g_users = {{"Anan", 25}, {"Bee", 30}};

// วิธีเดิม (ก่อน C++17): ใช้ pointer แทน "ไม่พบ" หรือใช้ sentinel value เช่น age = -1
User* find_user_old(const std::string& name) {
    for (auto& u : g_users) {
        if (u.name == name) return &u;
    }
    return nullptr;   // ต้องเช็ค nullptr ทุกครั้งที่เรียกใช้ ไม่งั้น dereference พัง
}

// วิธีใหม่ (C++17): std::optional สื่อความหมาย "อาจไม่มีค่า" ได้ตรงๆ ไม่ต้องใช้ pointer
std::optional<User> find_user(const std::string& name) {
    for (const auto& u : g_users) {
        if (u.name == name) return u;
    }
    return std::nullopt;   // ระบุชัดเจนว่า "ไม่มีค่า"
}

int main() {
    if (auto user = find_user("Bee")) {
        std::cout << "พบ: " << user->name << " อายุ " << user->age << "\n";
    } else {
        std::cout << "ไม่พบผู้ใช้\n";
    }

    auto missing = find_user("Chai");
    std::cout << "พบ 'Chai' หรือไม่: " << std::boolalpha << missing.has_value() << "\n";

    // value_or ให้ค่า default เมื่อไม่มีค่า
    std::optional<int> maybe_score;
    std::cout << "คะแนน default: " << maybe_score.value_or(0) << "\n";

    // การเข้าถึงค่าที่ไม่มีด้วย .value() จะโยน std::bad_optional_access
    try {
        std::optional<int> empty_opt;
        std::cout << empty_opt.value() << "\n";
    } catch (const std::bad_optional_access& e) {
        std::cout << "จับ exception ได้: " << e.what() << "\n";
    }

    return 0;
}
```

ผลลัพธ์:

```
พบ: Bee อายุ 30
พบ 'Chai' หรือไม่: false
คะแนน default: 0
จับ exception ได้: bad optional access
```

### สรุป API ที่ใช้บ่อยของ std::optional

| เมธอด/ไวยากรณ์ | ความหมาย |
|---|---|
| `if (opt)` หรือ `opt.has_value()` | เช็คว่ามีค่าอยู่ข้างในหรือไม่ |
| `*opt` หรือ `opt->member` | เข้าถึงค่าข้างใน (ไม่เช็ค — ถ้าไม่มีค่าจริงจะเป็น UB เหมือน dereference nullptr) |
| `opt.value()` | เข้าถึงค่าข้างใน พร้อมเช็คให้ ถ้าไม่มีค่าจะโยน `std::bad_optional_access` |
| `opt.value_or(default_val)` | คืนค่าข้างในถ้ามี ไม่งั้นคืน `default_val` แทน |
| `std::nullopt` | ค่าพิเศษที่ใช้แทน "ไม่มีค่า" กำหนดให้ `optional` ได้โดยตรง |

### เปรียบเทียบสามวิธี

| วิธี | ปลอดภัยจาก Null Dereference | สื่อความหมายใน Signature | ใช้กับ Type ที่ไม่มี "ค่าพิเศษ" ได้ |
|---|---|---|---|
| Pointer + `nullptr` | ไม่ (ผู้เรียกต้องเช็คเอง) | ไม่ชัดเจน (Pointer อาจหมายถึงอย่างอื่นก็ได้) | ได้ แต่ต้องมี Object อยู่ที่ไหนสักแห่งให้ชี้ไป |
| Sentinel Value | ไม่ (ต้องรู้ค่าพิเศษล่วงหน้า) | ไม่ชัดเจนเลย | ไม่ได้ ถ้าทุกค่าใน Type นั้นถูกต้องหมด |
| `std::optional<T>` | ปลอดภัยกว่า (`value()` เช็คให้) | ชัดเจนมาก | ได้ทุกกรณี |

---

## 72.6 std::variant (Step 574)

### ทบทวน union จาก Part 10 และปัญหาของมัน

ใน Part 10 เราเรียนเรื่อง `union` ไปแล้ว ซึ่ง `union` ให้ Memory ก้อนเดียวที่ใช้ร่วมกันได้
หลาย Type แต่ปัญหาใหญ่คือ **`union` ไม่รู้ว่าตอนนี้กำลังเก็บ Type ไหนอยู่จริง**:

```cpp
union OldUnsafeUnion {
    int as_int;
    float as_float;
};

OldUnsafeUnion u;
u.as_int = 42;
std::cout << u.as_float;   // อ่านผิด field! คอมไพล์ผ่านสบายๆ แต่ค่าที่ได้ไม่มีความหมาย (UB)
```

Compiler ไม่มีทางเตือนเราได้เลยว่ากำลังอ่าน Field ผิดจากที่เขียนล่าสุด เพราะ `union` ไม่เก็บ
ข้อมูลว่า "Active Member" คือตัวไหน — ทั้งหมดขึ้นอยู่กับวินัยของโปรแกรมเมอร์เอง

### วิธีแก้: std::variant

`std::variant<Types...>` คือ "Type-Safe Union" ที่เก็บได้ครั้งละ 1 Type จากรายการที่กำหนด
และ **รู้ตัวเองเสมอ** ว่าตอนนี้ Active Type คือตัวไหน:

```cpp
#include <iostream>
#include <string>
#include <vector>
#include <variant>

using Value = std::variant<int, double, std::string>;

struct Printer {
    void operator()(int i) const { std::cout << "int: " << i << "\n"; }
    void operator()(double d) const { std::cout << "double: " << d << "\n"; }
    void operator()(const std::string& s) const { std::cout << "string: " << s << "\n"; }
};

int main() {
    std::vector<Value> values;
    values.push_back(42);
    values.push_back(3.14);
    values.push_back(std::string("hello"));

    for (const auto& v : values) {
        std::visit(Printer{}, v);
    }

    // std::get กับ index ที่ผิด type จะโยน std::bad_variant_access
    Value v = 100;
    std::cout << "index ปัจจุบัน: " << v.index() << "\n";   // 0 = int, 1 = double, 2 = string

    if (auto* p = std::get_if<int>(&v)) {
        std::cout << "เข้าถึงแบบปลอดภัยด้วย get_if: " << *p << "\n";
    }

    try {
        std::cout << std::get<std::string>(v) << "\n";  // v ตอนนี้เป็น int ไม่ใช่ string
    } catch (const std::bad_variant_access& e) {
        std::cout << "จับ exception ได้: " << e.what() << "\n";
    }

    return 0;
}
```

ผลลัพธ์:

```
int: 42
double: 3.14
string: hello
index ปัจจุบัน: 0
เข้าถึงแบบปลอดภัยด้วย get_if: 100
จับ exception ได้: std::get: wrong index for variant
```

### กลไกสำคัญของ std::variant

- **`v.index()`** — คืนตำแหน่ง (0-based) ของ Type ที่ Active อยู่ตอนนี้ ตามลำดับที่ประกาศใน
  `std::variant<Types...>`
- **`std::get<T>(v)`** หรือ **`std::get<N>(v)`** — เข้าถึงค่าโดยระบุ Type หรือตำแหน่งตรงๆ
  ถ้า Type/ตำแหน่งไม่ตรงกับ Active Type จะโยน `std::bad_variant_access`
- **`std::get_if<T>(&v)`** — เวอร์ชันที่ไม่โยน Exception คืนค่าเป็น Pointer (`nullptr` ถ้าไม่
  ตรง Type) เหมาะกับกรณีที่อยากเช็คแบบไม่ใช้ `try/catch`
- **`std::visit(Callable, v)`** — วิธีที่แนะนำที่สุดในการ "ทำงานกับค่าข้างใน `variant` โดยไม่
  ต้องรู้ล่วงหน้าว่า Active Type คือตัวไหน" โดยส่ง Callable (เช่น `struct` ที่มี `operator()`
  Overload ครบทุก Type อย่างในตัวอย่าง หรือจะใช้ Generic Lambda ที่มี `if constexpr` ข้างใน
  ก็ได้) เข้าไป Compiler จะเลือก Overload ที่ตรงกับ Active Type ให้อัตโนมัติ

### เปรียบเทียบ union กับ std::variant

| ประเด็น | `union` (Part 10) | `std::variant` (C++17) |
|---|---|---|
| รู้ Active Type หรือไม่ | ไม่รู้ ต้องเก็บ Tag เอง (Tagged Union) | รู้เสมอ ผ่าน `.index()` |
| อ่าน Type ผิดแล้วเกิดอะไร | Undefined Behavior เงียบๆ | โยน Exception (`std::get`) หรือคืน `nullptr` (`get_if`) |
| รองรับ Type ที่มี Constructor/Destructor ซับซ้อน (เช่น `std::string`) | ต้องจัดการเองด้วยมือ (Placement New) ยุ่งยากมาก | จัดการให้อัตโนมัติ |
| Syntax การใช้งาน | เข้าถึง Field ตรงๆ (`u.as_int`) | ต้องผ่าน `std::get`/`std::visit`/`get_if` |

---

## 72.7 std::any และ inline Variable (Step 575)

### std::any: เก็บค่า Type อะไรก็ได้

`std::variant` ต้องระบุ "รายการ Type ที่เป็นไปได้" ไว้ล่วงหน้าตอนประกาศ Type
(`std::variant<int, double, std::string>`) แต่บางสถานการณ์เราไม่รู้ล่วงหน้าเลยว่าจะเก็บ Type
อะไรบ้าง (เช่น กล่องเก็บ Property แบบทั่วไปของ Config หรือ Plugin System) — กรณีนี้ใช้
`std::any` ซึ่งเก็บค่าของ **Type ใดก็ได้** โดยไม่ต้องระบุล่วงหน้า แลกมาด้วยการที่ต้องเช็ค
Type ตอน Runtime ทุกครั้งที่จะดึงค่ากลับมาใช้:

```cpp
#include <any>
#include <iostream>
#include <string>
#include <vector>

int main() {
    // std::any เก็บ "ค่าของ type อะไรก็ได้" โดยไม่ต้องมี template parameter ล่วงหน้า
    // ต่างจาก variant ตรงที่ any ไม่จำกัดว่า type ไหนได้บ้าง (แลกมาด้วยการเช็ค type ตอน runtime)
    std::vector<std::any> bag;
    bag.push_back(42);
    bag.push_back(std::string("กระเป๋าใส่ของ"));
    bag.push_back(3.14);

    for (const auto& item : bag) {
        if (item.type() == typeid(int)) {
            std::cout << "int: " << std::any_cast<int>(item) << "\n";
        } else if (item.type() == typeid(std::string)) {
            std::cout << "string: " << std::any_cast<std::string>(item) << "\n";
        } else if (item.type() == typeid(double)) {
            std::cout << "double: " << std::any_cast<double>(item) << "\n";
        }
    }

    std::any a = 10;
    try {
        std::cout << std::any_cast<std::string>(a) << "\n";  // a เก็บ int ไม่ใช่ string
    } catch (const std::bad_any_cast& e) {
        std::cout << "จับ exception ได้: " << e.what() << "\n";
    }

    return 0;
}
```

ผลลัพธ์:

```
int: 42
string: กระเป๋าใส่ของ
double: 3.14
จับ exception ได้: bad any_cast
```

### เปรียบเทียบ optional, variant, any

| Type | ใช้เมื่อไหร่ |
|---|---|
| `std::optional<T>` | มีค่าแค่ Type เดียว แต่ "อาจไม่มีค่า" |
| `std::variant<A, B, C>` | มีค่าได้หลาย Type ที่รู้ล่วงหน้าครบแล้ว (ปิดตาย ไม่ขยายได้) |
| `std::any` | มีค่าได้ Type ใดก็ได้ ไม่รู้ล่วงหน้า (เปิดกว้างที่สุด แต่ปลอดภัยน้อยที่สุด) |

> โดยทั่วไปควรเลือกใช้ตัวที่ "จำกัดที่สุดเท่าที่พอจะใช้ได้" เสมอ — ถ้ารู้ว่า Type มีกี่แบบ
> แน่นอน ให้ใช้ `variant` แทน `any` เพราะ `variant` ให้ Compiler ช่วยตรวจสอบความถูกต้องได้
> มากกว่า (เช่น `std::visit` บังคับให้ Handle ทุก Type ที่เป็นไปได้ ไม่งั้น Compile ไม่ผ่าน)

### inline Variable

ปัญหาคลาสสิกก่อน C++17: ถ้าประกาศตัวแปร Global ไว้ใน Header File แล้ว Header นั้นถูก
`#include` เข้าไปในหลายไฟล์ `.cpp` (หลาย Translation Unit ตามที่เรียนใน Part 17) จะเจอ
Linker Error "multiple definition" เพราะแต่ละ Translation Unit จะได้สำเนาตัวแปรนั้นไปคนละชุด

ไฟล์ `config_bad.hpp`:

```cpp
#ifndef CONFIG_BAD_HPP
#define CONFIG_BAD_HPP

// ก่อน C++17: ถ้าประกาศตัวแปร global แบบนี้ใน header แล้ว include เข้าหลาย .cpp
// จะเจอ "multiple definition" ตอน link เพราะแต่ละ .cpp translation unit
// จะได้ตัวแปรนี้ไปคนละชุด (นิยามซ้ำ)
double tax_rate = 0.07;

#endif
```

ถ้า `main.cpp` และ `a.cpp` ต่าง `#include "config_bad.hpp"` แล้ว Link เข้าด้วยกัน:

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 main.cpp a.cpp -o bad
```

จะได้ Error จริงจาก Linker:

```
/usr/bin/ld: /tmp/ccd1DmZL.o:(.data+0x0): multiple definition of `tax_rate'; /tmp/ccdoBKhq.o:(.data+0x0): first defined here
collect2: error: ld returned 1 exit status
```

**วิธีแก้แบบเก่า** ก่อน C++17 คือประกาศ `extern double tax_rate;` ใน Header แล้วไปนิยามค่า
จริงในไฟล์ `.cpp` ไฟล์เดียว (เทคนิคเดียวกับ Global Variable ข้าม Translation Unit ที่เรียนใน
Part 17) ซึ่งต้องแยกไฟล์ 2 ไฟล์เสมอ

**วิธีแก้แบบ C++17** ใช้คีย์เวิร์ด `inline` กับตัวแปร (ไม่ใช่แค่ฟังก์ชันเหมือนที่เรียนใน
Part 44) บอก Linker ว่า "ถ้าเจอนิยามซ้ำในหลาย Translation Unit ให้รวมเป็นตัวเดียวกัน แทนที่
จะ Error":

ไฟล์ `config_good.hpp`:

```cpp
#ifndef CONFIG_GOOD_HPP
#define CONFIG_GOOD_HPP

// C++17: inline variable บอก linker ว่า "ถ้าเจอนิยามซ้ำในหลาย translation unit
// ให้ถือว่าเป็นตัวแปรเดียวกัน (รวมเป็นตัวเดียว)" ไม่ error อีกต่อไป
inline double tax_rate = 0.07;

#endif
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 main.cpp a.cpp -o good
./good
```

ผลลัพธ์:

```
main.cpp: tax_rate = 0.07
a.cpp: tax_rate = 0.07
```

ทั้งสองไฟล์เห็นค่า `tax_rate` เดียวกันจริงๆ (ไม่ใช่คนละสำเนา) และ Link ผ่านโดยไม่มี Error
ฟีเจอร์นี้มีประโยชน์มากในการเขียน Header-Only Library ซึ่งเป็นรูปแบบที่นิยมมากขึ้นเรื่อยๆ
ในวงการ C++ สมัยใหม่ (เช่นไลบรารีที่เราจะใช้ใน Part 107 อย่าง nlohmann/json ก็เป็น
Header-Only Library)

---

## 72.8 std::filesystem เบื้องต้น (Step 576)

ก่อน C++17 การเขียนโค้ดที่ทำงานกับไฟล์และโฟลเดอร์ (List ไฟล์ในโฟลเดอร์, เช็คว่าไฟล์มีอยู่จริง
ไหม, เช็คขนาดไฟล์) ต้องพึ่งฟังก์ชันเฉพาะของแต่ละ OS (`opendir`/`readdir` บน POSIX,
`FindFirstFile`/`FindNextFile` บน Windows) ทำให้โค้ดไม่ Portable

C++17 เพิ่ม Library `<filesystem>` (namespace `std::filesystem`) ที่ให้ API เดียวใช้ได้ข้าม
แพลตฟอร์ม อิงจากไลบรารี Boost.Filesystem ที่ใช้กันแพร่หลายมาก่อนแล้ว

### ตัวอย่าง: List ไฟล์ในโฟลเดอร์

สมมติมีโครงสร้างโฟลเดอร์ `demo_folder/` ดังนี้:

```
demo_folder/
├── notes.txt
├── report.pdf
└── subdir/
    └── inner.txt
```

```cpp
#include <algorithm>
#include <filesystem>
#include <iostream>
#include <vector>

namespace fs = std::filesystem;

int main() {
    fs::path dir = "demo_folder";

    if (!fs::exists(dir) || !fs::is_directory(dir)) {
        std::cerr << "ไม่พบโฟลเดอร์: " << dir.string() << "\n";
        return 1;
    }

    // เก็บ entry ลง vector ก่อน แล้วเรียงชื่อ เพราะ directory_iterator
    // ไม่การันตีลำดับการอ่านไฟล์ (ขึ้นกับระบบไฟล์ของ OS)
    std::vector<fs::directory_entry> entries(fs::directory_iterator(dir),
                                              fs::directory_iterator{});
    std::sort(entries.begin(), entries.end(),
              [](const auto& a, const auto& b) { return a.path() < b.path(); });

    std::cout << "รายการในโฟลเดอร์ " << dir.string() << ":\n";
    for (const auto& entry : entries) {
        std::cout << (entry.is_directory() ? "[dir]  " : "[file] ")
                  << entry.path().filename().string();
        if (entry.is_regular_file()) {
            std::cout << " (" << fs::file_size(entry.path()) << " bytes)";
        }
        std::cout << "\n";
    }

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 list_dir.cpp -o list_dir
./list_dir
```

ผลลัพธ์ (ขนาดไฟล์เป็น 0 เพราะไฟล์ตัวอย่างว่างเปล่า):

```
รายการในโฟลเดอร์ demo_folder:
[file] notes.txt (0 bytes)
[file] report.pdf (0 bytes)
[dir]  subdir
```

### ฟังก์ชัน/Type ที่ใช้บ่อยใน std::filesystem

| ชื่อ | ความหมาย |
|---|---|
| `fs::path` | Type แทน "เส้นทางไฟล์" ทำงานได้ทั้ง Windows (`\`) และ Linux/macOS (`/`) โดยไม่ต้องกังวลเรื่องตัวคั่น |
| `fs::exists(path)` | เช็คว่าไฟล์/โฟลเดอร์นั้นมีอยู่จริงในระบบไฟล์ไหม |
| `fs::is_directory(path)` / `fs::is_regular_file(path)` | เช็คชนิดของ Entry |
| `fs::directory_iterator(path)` | Iterate ไฟล์/โฟลเดอร์ **ชั้นเดียว** ในโฟลเดอร์ที่ระบุ |
| `fs::recursive_directory_iterator(path)` | Iterate ไฟล์/โฟลเดอร์ **ทุกชั้น** ลงไปในโฟลเดอร์ย่อยทั้งหมด |
| `fs::file_size(path)` | คืนขนาดไฟล์เป็น Byte |
| `fs::create_directory(path)` / `fs::remove(path)` | สร้าง/ลบไฟล์หรือโฟลเดอร์ |

> **หมายเหตุเรื่อง Compiler เก่า**: บน GCC รุ่นเก่ากว่า 9.0 อาจต้องเพิ่ม Flag `-lstdc++fs`
> ตอน Link (เพราะตอนนั้น `<filesystem>` ยังถือเป็น Library แยกต่างหาก) แต่ GCC 9.0 ขึ้นไป
> (รวมถึง GCC 13 ที่ใช้ในหลักสูตรนี้) รวม `<filesystem>` เข้ากับ `libstdc++` หลักแล้ว จึงไม่ต้อง
> ใส่ Flag พิเศษนี้อีก

เราจะกลับมาเจาะลึก `std::filesystem` อีกครั้งแบบเต็มรูปแบบ (การ Copy/Move ไฟล์, การเดิน
Directory Tree แบบซับซ้อน, Symbolic Link) ในบทที่เกี่ยวกับ Build System และ Deploy ช่วงหลัง
ของหลักสูตร ตอนนี้ขอให้จำแค่ว่ามันมีอยู่และใช้แทน `opendir`/`FindFirstFile` แบบเดิมได้เมื่อ
ต้องการโค้ดที่ Portable ข้าม OS

---

## ข้อผิดพลาดที่พบบ่อย (Common Pitfalls)

1. **ลืมว่า `if constexpr` ยังต้อง Compile ผ่านสำหรับทุก Type ที่ถูกเรียกจริง** — `if constexpr`
   ตัดกิ่งที่ "ไม่ถูกเลือก" ทิ้งได้ก็จริง แต่กิ่งที่ **ถูกเลือกจริง** สำหรับ Type นั้นๆ ยังต้อง
   เขียนโค้ดที่ถูกต้องสำหรับ Type นั้นอยู่ดี — มันไม่ได้ปิดการตรวจสอบ Syntax ของภาษา
2. **เข้าถึงค่าของ `std::optional` ด้วย `*opt` หรือ `opt->` โดยไม่เช็คก่อน** — ถ้า `opt` ไม่มี
   ค่าจริง การทำแบบนี้คือ Undefined Behavior เหมือน Dereference `nullptr` เป๊ะๆ ควรเช็ค
   `if (opt)` หรือใช้ `opt.value()`/`opt.value_or()` แทนเสมอเมื่อไม่แน่ใจ
3. **ลืมว่า `std::variant` ไม่มี State ว่างเปล่าโดย Default** — ต่างจาก `std::optional` ตรงที่
   `std::variant<int, double>` จะ Active เป็น Type แรกเสมอ (คือ `int` ค่า 0) ตอนสร้างแบบไม่ระบุ
   ค่าเริ่มต้น ไม่ใช่ "ว่างเปล่า" แบบ `optional` — ถ้าต้องการ "ไม่มีค่าเลย" ให้เพิ่ม
   `std::monostate` เป็น Type แรกใน `variant`
4. **ใช้ `std::any_cast<T>` โดยไม่จับ Exception หรือไม่เช็ค Type ก่อน** — เหมือนกับ
   `std::get<T>` ของ `variant` ถ้า Type ไม่ตรง จะโยน `std::bad_any_cast` ทันที ควรเช็ค
   `item.type() == typeid(T)` ก่อน หรือใช้ `std::any_cast<T>(&item)` เวอร์ชันที่คืน Pointer
   (`nullptr` ถ้าไม่ตรง) แทนถ้าไม่อยากใช้ `try/catch`
5. **สับสนระหว่าง `directory_iterator` กับ `recursive_directory_iterator`** — ถ้าต้องการไฟล์
   ในโฟลเดอร์ย่อยด้วย แต่ใช้ `directory_iterator` เฉยๆ จะเห็นแค่ชั้นบนสุด (โฟลเดอร์ย่อยจะ
   ปรากฏเป็น Entry แบบ `is_directory() == true` แต่จะไม่ลงไปดูข้างในให้อัตโนมัติ)
6. **คาดหวังว่า `structured bindings` กับ `auto` เฉยๆ จะแก้ไขต้นฉบับได้** — `auto [a, b] = pair`
   เป็นการ Copy ค่าออกมา ถ้าต้องการแก้ไขค่าต้นฉบับผ่านตัวแปรที่แกะออกมา ต้องใช้ `auto& [a, b]`
   หรือ `const auto& [a, b]` ถ้าแค่อยากอ่านโดยไม่ Copy

---

## แบบฝึกหัดท้ายบท

1. เขียนฟังก์ชัน Generic Lambda ที่รับพารามิเตอร์ 2 ตัวและคืนค่าตัวที่ "มากกว่า" (คล้าย
   `std::max` แต่เขียนเอง) ทดสอบกับทั้ง `int` และ `std::string`

2. เขียนฟังก์ชัน `std::optional<double> safe_divide(double a, double b)` ที่คืน `std::nullopt`
   เมื่อ `b == 0` แทนการปล่อยให้เกิด Division by Zero แล้วเขียน `main()` ทดสอบทั้งกรณีหารได้
   และหารด้วยศูนย์

3. เขียน Function Template ที่ใช้ `if constexpr` เช็คว่า Type ที่ส่งเข้ามาเป็น Pointer หรือไม่
   (ใช้ `std::is_pointer_v<T>`) ถ้าเป็น Pointer ให้ Dereference ก่อนพิมพ์ค่า ถ้าไม่ใช่ให้พิมพ์
   ค่าตรงๆ

4. สร้าง `std::variant<int, std::string, bool>` เก็บใน `std::vector` อย่างน้อย 4 ค่าคละ Type
   กัน แล้วใช้ `std::visit` พิมพ์ค่าทุกตัวออกมาพร้อมระบุ Type ของแต่ละตัว

5. เขียนโปรแกรมที่ใช้ `std::filesystem::recursive_directory_iterator` นับจำนวนไฟล์ทั้งหมด
   (ไม่นับโฟลเดอร์) ในโฟลเดอร์ที่กำหนด รวมถึงไฟล์ในโฟลเดอร์ย่อยทุกชั้น

6. ทำ Header File ของตัวเองที่มีตัวแปร `inline constexpr` (รวมสองฟีเจอร์เข้าด้วยกัน) เก็บค่า
   Config บางอย่าง (เช่น `inline constexpr int kMaxRetries = 3;`) แล้ว `#include` เข้าไปใน
   ไฟล์ `.cpp` สองไฟล์ ทดสอบว่า Link ผ่านโดยไม่มี "multiple definition"

### แนวทางเฉลยข้อ 2

```cpp
#include <iostream>
#include <optional>

std::optional<double> safe_divide(double a, double b) {
    if (b == 0.0) {
        return std::nullopt;
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
        std::cout << "5 / 0 ไม่สามารถหารได้ (ได้ std::nullopt)\n";
    }

    std::cout << "ใช้ value_or สำหรับกรณีหารด้วยศูนย์: " << r2.value_or(-1.0) << "\n";

    return 0;
}
```

ผลลัพธ์ที่ควรได้:

```
10 / 2 = 5
5 / 0 ไม่สามารถหารได้ (ได้ std::nullopt)
ใช้ value_or สำหรับกรณีหารด้วยศูนย์: -1
```

### แนวทางเฉลยข้อ 4

```cpp
#include <iostream>
#include <string>
#include <variant>
#include <vector>

using Item = std::variant<int, std::string, bool>;

struct ItemPrinter {
    void operator()(int i) const {
        std::cout << "[int] " << i << "\n";
    }
    void operator()(const std::string& s) const {
        std::cout << "[string] " << s << "\n";
    }
    void operator()(bool b) const {
        std::cout << "[bool] " << std::boolalpha << b << "\n";
    }
};

int main() {
    std::vector<Item> items;
    items.push_back(100);
    items.push_back(std::string("Modern C++"));
    items.push_back(true);
    items.push_back(false);

    for (const auto& item : items) {
        std::visit(ItemPrinter{}, item);
    }

    return 0;
}
```

ผลลัพธ์ที่ควรได้:

```
[int] 100
[string] Modern C++
[bool] true
[bool] false
```

---

## สรุปท้ายบท

ใน Part นี้เราได้:

- ทบทวน Generic Lambda จาก C++14 และเข้าใจว่า Relaxed `constexpr` ทำให้เขียนฟังก์ชัน
  Compile-Time ได้เป็นธรรมชาติเหมือนฟังก์ชันปกติมากขึ้น
- รู้จัก `std::make_unique` ที่เติมช่องว่างจาก `std::make_shared` และไวยากรณ์ Binary Literal /
  Digit Separator ที่ช่วยให้โค้ดตัวเลขอ่านง่ายขึ้น
- ใช้ Structured Bindings แกะค่าจาก `pair`, `map`, `tuple` ได้อย่างกระชับและสื่อความหมาย
- ใช้ `if constexpr` แก้ปัญหา Template ที่ต้องแตกกิ่งพฤติกรรมตาม Type โดยไม่ต้องพึ่ง SFINAE
  ที่ซับซ้อน
- ใช้ `std::optional` แทน Pointer/Sentinel Value, ใช้ `std::variant` แทน `union` แบบเดิมให้
  Type-Safe ขึ้น และใช้ `std::any` เมื่อไม่รู้ Type ล่วงหน้าเลย
- รู้จัก `inline variable` ที่แก้ปัญหา Multiple Definition ของตัวแปร Global ใน Header และ
  `std::filesystem` เบื้องต้นสำหรับอ่านไฟล์/โฟลเดอร์แบบ Portable ข้าม OS

ฟีเจอร์ทั้งหมดใน Part นี้เป็นเครื่องมือที่ใช้งานทุกวันในโค้ด C++ สมัยใหม่ แต่ยังไม่ได้แตะ
เรื่อง "ระบบตรวจสอบ Type ของ Template ที่ทรงพลังกว่าเดิม" ซึ่งเป็นสิ่งที่ C++20 นำมาเปลี่ยน
วงการอีกครั้งด้วยฟีเจอร์ที่ชื่อว่า **Concepts** — ใน **Part 73** เราจะเห็นปัญหาจริงที่เกิดจาก
Error Message ของ Template ที่อ่านไม่รู้เรื่อง และวิธีที่ Concepts เข้ามาแก้ปัญหานี้อย่าง
หมดจด

**ต่อไป:** [Part 73 — C++20 Concepts](./part-073-cpp20-concepts.md)
