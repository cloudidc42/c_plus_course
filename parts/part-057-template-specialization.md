# Part 57: Template Specialization และ Variadic Template (Step 449–456)

> Module E — Templates, Generic Programming และ STL | Part 57 จาก 125
> Step ที่ครอบคลุมใน Part นี้: Step 449–456
> Part ก่อนหน้า: [Part 56 — Function Template และ Class Template](./part-056-function-class-templates.md) | Part ถัดไป: [Part 58 — แนะนำ STL](./part-058-stl-overview.md)

## เป้าหมายการเรียนรู้ (Learning Objectives)

หลังจบ Part นี้ ผู้เรียนจะสามารถ:

1. อธิบายว่าทำไมบางครั้ง Template แบบ generic ธรรมดา (ที่เรียนใน Part 56) ไม่เพียงพอ และ
   ต้องการพฤติกรรมพิเศษสำหรับบาง type โดยเฉพาะ
2. เขียน **Full Specialization** ของ Function Template สำหรับ type ที่ต้องการ logic ต่างจาก
   generic version (เช่น `const char*`)
3. เขียน **Full Specialization** ของ Class Template สำหรับ type ที่ต้องการ implementation
   ทั้งหมดแตกต่างไปจาก generic version
4. เขียน **Partial Specialization** ของ Class Template สำหรับกรณีที่ type ยังไม่ถูกระบุครบทุก
   parameter (เช่น เมื่อ template parameter สองตัวเป็น type เดียวกัน หรือเมื่อ parameter ตัวใด
   ตัวหนึ่งเป็น pointer)
5. เข้าใจและเขียน **Variadic Template** (`template <typename... Args>`) ที่รับ argument
   จำนวนเท่าใดก็ได้ ชนิดใดก็ได้
6. ใช้ **Parameter Pack Expansion** และ **`sizeof...`** จัดการกับกลุ่ม argument ที่มีจำนวนไม่
   แน่นอนได้อย่างถูกต้อง
7. เขียนฟังก์ชัน `printAll` ที่รับ argument กี่ตัวก็ได้ ชนิดอะไรก็ได้ ด้วยเทคนิค
   **Recursive Variadic Template** และเปรียบเทียบกับ **Fold Expression** (C++17) ที่กระชับกว่า

---

## 57.1 เมื่อ Generic Template ธรรมดาไม่พอ: Full Specialization ของ Function Template (Step 449)

ใน Part 56 เราเขียน function template ที่ใช้ logic**เดียวกันเป๊ะ**กับทุก type ที่ถูกส่งเข้ามา
แต่ในโลกจริงมีบาง type ที่ logic แบบ generic ใช้ไม่ได้ผลอย่างถูกต้อง ตัวอย่างคลาสสิกที่สุดคือ
การเปรียบเทียบ `const char*` (C-style string) ด้วย `==`:

```cpp
#include <cstring>
#include <iostream>
#include <string>

// generic version: ใช้ operator== เปรียบเทียบ (ใช้ได้กับ int, double, std::string ฯลฯ)
template <typename T>
bool isEqual(T a, T b) {
    std::cout << "[generic version] ";
    return a == b;
}

int main() {
    std::cout << isEqual(10, 10) << '\n';                  // ใช้ generic version -> ถูกต้อง
    std::cout << isEqual(std::string("hi"), std::string("hi")) << '\n';  // ถูกต้อง เพราะ std::string มี operator== เปรียบเนื้อหา

    const char* s1 = "hello";
    const char* s2 = "hello";
    std::cout << isEqual(s1, s2) << '\n';   // ปัญหา! เปรียบเทียบ "address" ไม่ใช่ "เนื้อหา"

    return 0;
}
```

ปัญหาคือ `const char*` เป็น **pointer** — เมื่อ `T = const char*` โค้ด `a == b` ใน generic
version จะกลายเป็นการเปรียบเทียบ**ที่อยู่หน่วยความจำ** ไม่ใช่เปรียบเทียบเนื้อหาของ string เลย
(ผลลัพธ์อาจเป็น `true` หรือ `false` ก็ได้ ขึ้นกับว่า compiler ทำ string literal pooling หรือไม่
— เป็นพฤติกรรมที่**ไม่น่าเชื่อถือ**และไม่ควรพึ่งพา)

### วิธีแก้ด้วย Full (Explicit) Specialization

เราสามารถบอก compiler ว่า "สำหรับ type นี้โดยเฉพาะ ให้ใช้ logic นี้แทน generic version" ด้วย
**Full Specialization**:

```cpp
#include <cstring>
#include <iostream>
#include <string>

template <typename T>
bool isEqual(T a, T b) {
    std::cout << "[generic version] ";
    return a == b;
}

// Full Specialization สำหรับ const char* โดยเฉพาะ
// เพราะ const char* เปรียบเทียบด้วย == จะเทียบ "ที่อยู่หน่วยความจำ" ไม่ใช่เนื้อหาข้างใน string
template <>
bool isEqual<const char*>(const char* a, const char* b) {
    std::cout << "[specialized for const char*] ";
    return std::strcmp(a, b) == 0;
}

int main() {
    std::cout << isEqual(10, 10) << '\n';                  // ใช้ generic version
    std::cout << isEqual(std::string("hi"), std::string("hi")) << '\n';  // generic version (string มี operator==)

    const char* s1 = "hello";
    const char* s2 = "hello";
    // s1 กับ s2 อาจเป็นคนละ address กัน (ขึ้นกับ compiler) แต่เนื้อหาเหมือนกัน
    std::cout << isEqual(s1, s2) << '\n';                   // ใช้ specialized version -> เทียบเนื้อหาถูกต้อง

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 full_spec_func.cpp -o full_spec_func
./full_spec_func
```

```
[generic version] 1
[generic version] 1
[specialized for const char*] 1
```

### อธิบาย Syntax ของ Full Specialization

- **`template <>`**: บรรทัดนี้บอกว่า "นี่ไม่ใช่ template ทั่วไปที่มี parameter เหลืออยู่แล้ว —
  ทุก type ถูกระบุครบถ้วนแล้ว" วงเล็บมุมว่างเปล่า `<>` คือสัญลักษณ์ของ Full Specialization
- **`bool isEqual<const char*>(const char* a, const char* b)`**: ระบุ type argument
  `<const char*>` ต่อท้ายชื่อฟังก์ชันชัดเจน บอกว่านี่คือ specialization สำหรับกรณี `T =
  const char*` โดยเฉพาะ
- **compiler เลือก specialization ก่อน generic version เสมอเมื่อ type ตรงกันพอดี** — เมื่อเรา
  เรียก `isEqual(s1, s2)` โดยที่ `s1`, `s2` เป็น `const char*` compiler จะมองหา
  specialization ที่ตรงกับ `const char*` ก่อน ถ้าเจอก็ใช้ตัวนั้นทันที ถ้าไม่เจอจึงกลับไปใช้
  generic version แทน
- **Full Specialization ไม่ใช่ Overload** — แม้ syntax จะดูคล้ายกัน แต่ในทางเทคนิค
  specialization เป็นการบอกว่า "template ตัวเดิม เมื่อ T คือ type นี้ ให้ทำงานแบบนี้แทน" ไม่ใช่
  การสร้างฟังก์ชันใหม่ที่แยกจากกัน — ความแตกต่างนี้มีผลในหลายกรณีขั้นสูง (เช่นตอนที่ compiler
  เลือกว่าจะใช้ overload ตัวไหนก่อนที่จะมาพิจารณา specialization) แต่ในระดับที่เรียนตอนนี้
  จำแค่ว่า **"เขียน specialization ก็เพื่อบอกพฤติกรรมพิเศษสำหรับ type หนึ่งโดยเฉพาะ"** ก็เพียงพอ

---

## 57.2 Full Specialization ของ Class Template (Step 450)

หลักการเดียวกันนี้ใช้กับ **Class Template** ได้เช่นกัน — เมื่อ type บางตัวต้องการ
implementation ที่แตกต่างไปจาก generic version โดยสิ้นเชิง:

```cpp
#include <iostream>
#include <string>

// Generic version: บอกว่าชนิดข้อมูลนี้ "ไม่ใช่" ชนิดพิเศษที่รู้จัก
template <typename T>
class TypeDescriber {
public:
    static std::string describe() { return "ชนิดข้อมูลทั่วไป"; }
};

// Full Specialization สำหรับ bool โดยเฉพาะ
template <>
class TypeDescriber<bool> {
public:
    static std::string describe() { return "boolean (true/false)"; }
};

// Full Specialization สำหรับ std::string โดยเฉพาะ
template <>
class TypeDescriber<std::string> {
public:
    static std::string describe() { return "ข้อความ (std::string)"; }
};

int main() {
    std::cout << "int: " << TypeDescriber<int>::describe() << '\n';
    std::cout << "bool: " << TypeDescriber<bool>::describe() << '\n';
    std::cout << "string: " << TypeDescriber<std::string>::describe() << '\n';
    std::cout << "double: " << TypeDescriber<double>::describe() << '\n';
    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 full_spec_class.cpp -o full_spec_class
./full_spec_class
```

```
int: ชนิดข้อมูลทั่วไป
bool: boolean (true/false)
string: ข้อความ (std::string)
double: ชนิดข้อมูลทั่วไป
```

### จุดสำคัญ

- **`template <> class TypeDescriber<bool> { ... };`**: เขียน class ทั้งตัวใหม่หมด สำหรับกรณี
  `T = bool` โดยเฉพาะ — ไม่ได้ "override" บาง method เหมือน virtual function แต่เป็นการ
  **แทนที่ implementation ทั้งหมด** ของ class เมื่อใช้กับ type นั้น สามารถมี member function,
  member variable ที่ต่างไปจาก generic version โดยสิ้นเชิงได้เลย
- **`int` และ `double` ยังคงใช้ generic version** เพราะไม่มี specialization ที่ตรงกับมันโดยตรง
  — compiler จะ fallback ไปใช้ template หลัก (primary template) โดยอัตโนมัติ
- **ทำไมต้องรู้จักเทคนิคนี้**: Full Specialization ของ class template คือกลไกเบื้องหลังของ
  `std::vector<bool>` ในไลบรารีมาตรฐาน! `std::vector<bool>` เป็น specialization พิเศษที่เก็บ
  ข้อมูลแบบ **bit-packed** (1 bit ต่อ 1 ค่า แทนที่จะเป็น 1 byte เหมือน `bool` ปกติ) เพื่อ
  ประหยัดหน่วยความจำ — พฤติกรรมและ interface บางส่วนของมันจึงต่างจาก `std::vector<T>` ทั่วไป
  อย่างมีนัยสำคัญ (เรื่องนี้เป็นตัวอย่าง "กับดัก" ที่มีชื่อเสียงของ C++ ซึ่งจะพูดถึงอีกครั้งใน
  **Part 59**)

---

## 57.3 Partial Specialization ของ Class Template (Step 451)

**Partial Specialization** ต่างจาก Full Specialization ตรงที่**ยังไม่ได้ระบุ type ให้ครบทุก
parameter** — ยังเหลือ template parameter บางตัวเป็น generic อยู่ ใช้ในกรณีที่ต้องการ
พฤติกรรมพิเศษสำหรับ**รูปแบบ (pattern)** ของ type มากกว่า type เดี่ยวๆ ตัวใดตัวหนึ่ง

> **หมายเหตุสำคัญ**: Partial Specialization ใช้ได้กับ **Class Template เท่านั้น** — C++ ไม่
> อนุญาตให้เขียน Partial Specialization ของ **Function Template** ได้ (เป็นกฎของภาษาที่ต้อง
> จำไว้) ถ้าต้องการพฤติกรรมคล้าย partial specialization กับฟังก์ชัน ต้องใช้ Function
> Overloading ร่วมกับเทคนิคขั้นสูงอย่าง SFINAE หรือ `if constexpr` ที่จะเรียนใน Part 72 และ 78

```cpp
#include <iostream>
#include <string>

// Generic version: class template ที่มี 2 type parameter
template <typename T1, typename T2>
class Pair {
public:
    Pair(T1 first, T2 second) : first_(std::move(first)), second_(std::move(second)) {}
    void print() const {
        std::cout << "Pair ทั่วไป: (" << first_ << ", " << second_ << ")\n";
    }

private:
    T1 first_;
    T2 second_;
};

// Partial Specialization: กรณีที่ T1 และ T2 เป็น type เดียวกัน (T, T)
// สังเกตว่ายังมี template parameter เหลืออยู่ (T) ไม่ได้ระบุ type ที่แน่นอนแบบ full specialization
template <typename T>
class Pair<T, T> {
public:
    Pair(T first, T second) : first_(std::move(first)), second_(std::move(second)) {}
    void print() const {
        std::cout << "Pair แบบ type เดียวกัน: (" << first_ << ", " << second_ << ") ["
                  << "ใช้ partial specialization]\n";
    }

private:
    T first_;
    T second_;
};

// Partial Specialization อีกแบบ: กรณีที่ T2 เป็น pointer (T2 = U*)
template <typename T1, typename U>
class Pair<T1, U*> {
public:
    Pair(T1 first, U* second) : first_(std::move(first)), second_(second) {}
    void print() const {
        std::cout << "Pair แบบตัวที่สองเป็น pointer: (" << first_ << ", *" << *second_
                  << ") [ใช้ partial specialization สำหรับ pointer]\n";
    }

private:
    T1 first_;
    U* second_;
};

int main() {
    Pair<int, std::string> mixed(1, "hello");      // ใช้ generic version
    mixed.print();

    Pair<int, int> sameType(10, 20);               // ใช้ partial specialization Pair<T, T>
    sameType.print();

    int value = 99;
    Pair<std::string, int*> withPointer("age", &value);  // ใช้ partial specialization Pair<T1, U*>
    withPointer.print();

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 partial_spec.cpp -o partial_spec
./partial_spec
```

```
Pair ทั่วไป: (1, hello)
Pair แบบ type เดียวกัน: (10, 20) [ใช้ partial specialization]
Pair แบบตัวที่สองเป็น pointer: (age, *99) [ใช้ partial specialization สำหรับ pointer]
```

### วิธีอ่าน Syntax และเลือกว่า Compiler จะใช้ตัวไหน

- **`template <typename T> class Pair<T, T>`**: อ่านว่า "สำหรับกรณีที่ template parameter
  สองตัวของ `Pair` เป็น type เดียวกันเป๊ะ ให้ใช้ implementation นี้แทน" — ตัว `T` ที่ประกาศใน
  `template <typename T>` (บรรทัดบน) ถูกใช้ซ้ำสองครั้งใน `Pair<T, T>` (บรรทัดล่าง) เพื่อบังคับ
  ว่าทั้งสอง argument ต้องตรงกัน
- **`template <typename T1, typename U> class Pair<T1, U*>`**: อ่านว่า "สำหรับกรณีที่
  argument ตัวที่สองเป็น pointer ของ type ใดก็ได้ (`U*`) ให้ใช้ implementation นี้แทน" — `T1`
  ยังคงเป็น generic เต็มรูปแบบ มีแค่ `T2` ที่ถูก "จำกัดรูปแบบ" ให้ต้องเป็น pointer เท่านั้น
- **กฎการเลือก (Partial Ordering)**: เมื่อมีหลาย specialization ที่ตรงกับการเรียกใช้งาน
  compiler จะเลือกตัวที่**"เจาะจงที่สุด" (most specialized)** เสมอ — เรียงจากเจาะจงมากไปน้อย
  คือ: Full Specialization > Partial Specialization ที่เจาะจงกว่า > Partial Specialization
  ที่กว้างกว่า > Primary Template (generic version) ถ้า compiler ไม่สามารถตัดสินได้ว่า
  specialization ไหนเจาะจงกว่ากัน (เช่นสองตัวเจาะจงพอๆ กันแต่คนละแบบ) จะเกิด **ambiguous
  specialization error** ทันที
- **ประโยชน์ในโลกจริง**: `std::vector<T*>` ในบาง implementation ของ allocator library ใช้
  partial specialization แบบนี้เพื่อ optimize การจัดการ pointer โดยเฉพาะ และ type traits
  library ทั้งหมด (เช่น `std::is_pointer<T>`, `std::remove_reference<T>` ที่จะเรียนใน
  **Part 78**) ก็สร้างขึ้นจากกลไก Partial Specialization นี้เป๊ะๆ — เข้าใจเรื่องนี้ให้แน่นตอนนี้
  จะทำให้เรียน Metaprogramming ในอนาคตง่ายขึ้นมาก

---

## 57.4 Variadic Template เบื้องต้น: รับ Argument กี่ตัวก็ได้ (Step 452)

ทุก Template ที่เขียนมาจนถึงตอนนี้มี**จำนวน parameter คงที่** (`Pair<T1, T2>` มี 2 ตัวเสมอ)
แต่บางครั้งเราต้องการฟังก์ชันที่รับ argument **จำนวนเท่าใดก็ได้** — เช่น `printf` ใน C ที่รับ
argument กี่ตัวก็ได้ (แม้จะไม่ type-safe เพราะใช้ `...` แบบ C-style varargs ที่เรียนตอน Part 3)

C++11 แนะนำ **Variadic Template** ที่ทำสิ่งเดียวกันได้แบบ **type-safe เต็มรูปแบบ**:

```cpp
#include <iostream>

// Variadic Template: Args... คือ "parameter pack" รับ type ได้กี่ตัวก็ได้ (รวม 0 ตัว)
template <typename... Args>
void countArgs(Args... args) {
    // sizeof...(Args) หรือ sizeof...(args) นับจำนวน argument ใน pack ตอน compile-time
    std::cout << "จำนวน argument: " << sizeof...(args) << '\n';
}

int main() {
    countArgs();                 // 0 argument
    countArgs(1);                 // 1 argument
    countArgs(1, 2.5, "three");   // 3 argument คนละ type กัน
    countArgs('a', 'b', 'c', 'd'); // 4 argument
    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 variadic_basic.cpp -o variadic_basic
./variadic_basic
```

```
จำนวน argument: 0
จำนวน argument: 1
จำนวน argument: 3
จำนวน argument: 4
```

### อธิบาย Syntax ใหม่

- **`typename... Args`**: จุดสามจุด (`...`) หลัง `typename` ประกาศว่า `Args` ไม่ใช่ type
  เดียว แต่เป็น **Template Parameter Pack** — "กลุ่ม" ของ type จำนวนเท่าใดก็ได้ (ตั้งแต่ 0
  type ขึ้นไป)
- **`Args... args`**: จุดสามจุดหลัง `Args` ในรายการ parameter ของฟังก์ชัน ประกาศว่า `args`
  เป็น **Function Parameter Pack** — กลุ่มของ parameter ที่มี type ตรงกับแต่ละตัวใน `Args`
  ตามลำดับ
- **`sizeof...(args)` หรือ `sizeof...(Args)`**: operator พิเศษที่นับจำนวนสมาชิกใน
  parameter pack **ตอน compile-time** (ไม่ใช่ตอน runtime) — สังเกตว่าไม่มีวงเล็บครอบ
  `sizeof...` แบบ `sizeof(...)` เหมือน `sizeof` ธรรมดา เพราะ `sizeof...` เป็น operator
  คนละตัวกับ `sizeof` แม้จะเขียนคล้ายกันก็ตาม
- **`countArgs()` เรียกได้แม้ไม่ส่ง argument เลย** — เพราะ parameter pack รับได้ตั้งแต่ 0 ตัว
  ขึ้นไป นี่คือความยืดหยุ่นที่ทำให้ variadic template เหมาะกับงานที่จำนวน argument ไม่แน่นอน
  โดยธรรมชาติ

---

## 57.5 Parameter Pack Expansion และ Fold Expression (Step 453)

การมี parameter pack เฉยๆ ยังไม่มีประโยชน์มากนัก จุดที่ทรงพลังจริงคือการ **"ขยาย" (expand)**
pack นั้นออกมาใช้งาน — C++ มีสอง generation ของเทคนิคนี้: **Parameter Pack Expansion แบบ
recursive** (มีมาตั้งแต่ C++11) และ **Fold Expression** (เพิ่มเข้ามาใน C++17 ที่กระชับกว่ามาก)

### Fold Expression (C++17)

```cpp
#include <iostream>

// ----- Fold Expression (ฟีเจอร์ใหม่ของ C++17) -----
// (args + ...) คือ Fold Expression: ขยาย args ทั้งหมดแล้ว "พับ" เข้าด้วยกันด้วย operator+
template <typename... Args>
auto sumAll(Args... args) {
    return (args + ...);   // เทียบเท่า: args1 + (args2 + (args3 + ... ))
}

// Fold Expression กับ operator<< เพื่อพิมพ์ค่าทุกตัวโดยไม่ต้องเขียน recursive function เลย
template <typename... Args>
void printAllFold(const Args&... args) {
    ((std::cout << args << ' '), ...);   // unary right fold ของ comma operator
    std::cout << '\n';
}

int main() {
    std::cout << "sumAll(1, 2, 3) = " << sumAll(1, 2, 3) << '\n';
    std::cout << "sumAll(1.5, 2.5, 3.0) = " << sumAll(1.5, 2.5, 3.0) << '\n';

    printAllFold(1, 2.5, "three", 'x');

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 fold_expression.cpp -o fold_expression
./fold_expression
```

```
sumAll(1, 2, 3) = 6
sumAll(1.5, 2.5, 3.0) = 7
1 2.5 three x 
```

### อธิบาย Fold Expression

- **`(args + ...)`**: เรียกว่า **Unary Right Fold** — syntax ทั่วไปคือ `(pack op ...)` โดย
  compiler จะขยายเป็น `arg1 op (arg2 op (arg3 op ... op argN))` โดยอัตโนมัติ ในที่นี้คือ
  `arg1 + (arg2 + (arg3 + ...))`
- **`auto` เป็น return type**: จำเป็นเพราะ**เราไม่รู้ล่วงหน้าว่าผลรวมจะเป็น type อะไร** —
  `sumAll(1, 2, 3)` คืนค่าเป็น `int` แต่ `sumAll(1.5, 2.5, 3.0)` คืนค่าเป็น `double` — `auto`
  ให้ compiler อนุมาน return type จาก expression ข้างในให้อัตโนมัติ (เทคนิคนี้เรียกว่า
  **Return Type Deduction** จะเรียนละเอียดใน Part 72)
- **`((std::cout << args << ' '), ...)`**: ซับซ้อนกว่าเล็กน้อย — เป็น fold expression ของ
  **comma operator** (`,`) ที่ครอบ sub-expression `std::cout << args << ' '` ไว้ ผลลัพธ์คือ
  ขยายเป็น `(std::cout << arg1 << ' '), (std::cout << arg2 << ' '), ...` ซึ่งพิมพ์ทีละตัว
  เรียงตามลำดับ — เทคนิคนี้เป็น**สำนวนมาตรฐาน (idiom)** ที่พบบ่อยมากในโค้ด Modern C++ สำหรับ
  พิมพ์ parameter pack โดยไม่ต้องเขียน recursive function
- **Fold Expression รองรับ operator หลายตัว**: `+`, `-`, `*`, `&&`, `||`, `,` และอื่นๆ อีก
  มากมาย ทั้งแบบ unary (ซ้ายหรือขวา) และ binary fold (ที่มีค่าเริ่มต้น) — รายละเอียดเต็มรูปแบบ
  จะกล่าวถึงอีกครั้งเมื่อพูดถึงฟีเจอร์ C++17 อย่างเป็นทางการใน **Part 72**

---

## 57.6 Recursive Variadic Template: เขียน `printAll` แบบคลาสสิก (Step 454)

ก่อนที่ C++17 จะมี Fold Expression นักพัฒนา C++ ใช้เทคนิค **Recursive Variadic Template** ใน
การขยาย parameter pack — เทคนิคนี้ยังคงสำคัญมากที่ต้องเข้าใจ เพราะ (1) โค้ดเก่าจำนวนมากยังใช้
รูปแบบนี้อยู่ (2) กรณีที่ logic ซับซ้อนกว่าแค่ "พับด้วย operator เดียว" fold expression ทำไม่ได้
ต้องกลับมาใช้ recursion (3) มันสอนให้เข้าใจแนวคิด **Compile-Time Recursion** ที่เป็นรากฐานของ
Template Metaprogramming ทั้งหมด (Part 78)

```cpp
#include <iostream>

// ----- Recursive Variadic Template (แบบคลาสสิก ก่อน C++17) -----

// Base case: เมื่อไม่มี argument เหลือแล้ว ให้หยุด recursion
void printAll() {
    std::cout << '\n';
}

// Recursive case: ดึง argument ตัวแรกออกมาพิมพ์ ที่เหลือส่งต่อแบบ recursive
template <typename First, typename... Rest>
void printAll(const First& first, const Rest&... rest) {
    std::cout << first;
    if (sizeof...(rest) > 0) {
        std::cout << ", ";
    }
    printAll(rest...);   // ขยาย parameter pack ที่เหลือ แล้วเรียกตัวเองซ้ำ (recursion ที่ compile-time)
}

int main() {
    printAll(1, 2, 3);
    printAll("hello", 3.14, 'x', true);
    printAll(42);
    printAll();   // เรียก overload ที่ไม่รับ argument ตรงๆ

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 printall_recursive.cpp -o printall_recursive
./printall_recursive
```

```
1, 2, 3
hello, 3.14, x, 1
42

```

(บรรทัดว่างสุดท้ายมาจาก `printAll()` ที่เรียกโดยไม่ส่ง argument เลย ซึ่งตรงกับ base case ที่
พิมพ์แค่ `'\n'`)

### เดินตาม Recursion ทีละขั้น

พิจารณาการเรียก `printAll(1, 2, 3)`:

1. **เรียกครั้งที่ 1**: `First = int (1)`, `Rest... = {2, 3}` → พิมพ์ `1` ตามด้วย `", "` (เพราะ
   `sizeof...(rest) == 2 > 0`) แล้วเรียก `printAll(2, 3)` ต่อ
2. **เรียกครั้งที่ 2**: `First = int (2)`, `Rest... = {3}` → พิมพ์ `2` ตามด้วย `", "` (เพราะ
   `sizeof...(rest) == 1 > 0`) แล้วเรียก `printAll(3)` ต่อ
3. **เรียกครั้งที่ 3**: `First = int (3)`, `Rest... = {}` (ว่างเปล่า) → พิมพ์ `3` **ไม่มี** `", "`
   ต่อท้าย (เพราะ `sizeof...(rest) == 0`) แล้วเรียก `printAll()` (ไม่มี argument) ต่อ
4. **เรียกครั้งที่ 4 (Base Case)**: ตรงกับ overload ที่ไม่รับ parameter เลย → พิมพ์ `'\n'`
   เพื่อจบบรรทัด แล้ว**หยุด recursion**

### ทำไม Base Case ต้องเป็น "ฟังก์ชันธรรมดา" ไม่ใช่ Template

สังเกตว่า `void printAll()` **ไม่ใช่** template (ไม่มี `template <...>` ข้างบน) — นี่คือ
รายละเอียดที่สำคัญมาก: ถ้าเราลองเขียน parameter pack ให้ว่างได้ในตัว template เดียว (เช่น
`template <typename... Args> void printAll(Args... args)` แล้วไม่มี base case แยก) จะไม่มี
จุดสิ้นสุดของ recursion อย่างชัดเจน — เมื่อ pack เหลือ 0 ตัว การเรียก `printAll(rest...)` ที่
ไม่มี argument จะต้อง**เลือก overload ที่ถูกต้อง**ระหว่าง template แบบ variadic (ที่ instantiate
ด้วย 0 argument ได้เหมือนกัน) กับฟังก์ชันธรรมดาแบบไม่มี argument เลย — C++ กำหนดกฎว่า
**ฟังก์ชันธรรมดา (non-template) จะถูกเลือกก่อนเสมอเมื่อ match พอดีกัน** เมื่อเทียบกับ function
template ทำให้ `printAll()` ธรรมดาถูกเรียกแทนที่จะพยายาม instantiate variadic template ด้วย
0 type ต่อไปเรื่อยๆ — การแยก base case เป็นฟังก์ชันธรรมดาชัดเจนแบบนี้จึงเป็น**สำนวนมาตรฐาน**ที่
ใช้กันทั่วไปเพื่อความชัดเจนและป้องกันความกำกวม

### เปรียบเทียบ Recursive Variadic Template กับ Fold Expression

| ประเด็น | Recursive Variadic Template | Fold Expression (C++17) |
|---|---|---|
| ความยาวโค้ด | ต้องเขียน 2 overload (base case + recursive case) | เขียนบรรทัดเดียวจบ |
| ความยืดหยุ่นของ logic | สูงมาก ใส่ logic ซับซ้อนระหว่างแต่ละ argument ได้อิสระ | จำกัดแค่การพับด้วย operator เดียว |
| ใช้ได้กับมาตรฐานใด | C++11 ขึ้นไป | C++17 ขึ้นไปเท่านั้น |
| ความเร็วตอน compile | ช้ากว่าเล็กน้อย (มีการ instantiate ซ้อนหลายชั้น) | เร็วกว่า |
| เหมาะกับ | logic ที่ต้องแทรกเงื่อนไขระหว่างทาง (เช่นใส่ `", "` คั่น) | การรวมค่าง่ายๆ (sum, and, or, พิมพ์ต่อกัน) |

ในทางปฏิบัติ โค้ด Modern C++ (C++17 ขึ้นไป) มักเลือกใช้ **Fold Expression ก่อนเสมอถ้าทำได้**
เพราะสั้นกว่าและ compile เร็วกว่า แล้วจึงหันไปใช้ **Recursive Variadic Template เมื่อ logic
ซับซ้อนเกินกว่าที่ fold expression จะรองรับได้** (เช่นตัวอย่าง `printAll` ด้านบนที่ต้องใส่
`", "` คั่นระหว่างค่าแบบมีเงื่อนไข ซึ่งทำได้ยากกว่าด้วย fold expression ตรงๆ)

---

## 57.7 ตัวอย่างเพิ่มเติม: ฟังก์ชัน Variadic ที่ใช้งานได้จริง (Step 455)

ลองนำสิ่งที่เรียนมาประยุกต์ใช้กับปัญหาที่พบได้จริง — ฟังก์ชันหาค่ามากที่สุดจาก argument
**กี่ตัวก็ได้** (ต่อยอดจาก `myMax` ที่รับแค่ 2 argument ใน Part 56):

```cpp
#include <iostream>

template <typename T>
T maxOfAll(T value) {
    return value;   // base case: เหลือ argument เดียว ก็คือค่ามากที่สุดแล้ว
}

template <typename T, typename... Rest>
T maxOfAll(T first, Rest... rest) {
    T maxOfRest = maxOfAll(rest...);
    return (first > maxOfRest) ? first : maxOfRest;
}

int main() {
    std::cout << maxOfAll(3, 7, 2, 9, 4) << '\n';
    std::cout << maxOfAll(1.5, 2.5, 0.5) << '\n';
    std::cout << maxOfAll(42) << '\n';
    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 maxofall.cpp -o maxofall
./maxofall
```

```
9
2.5
42
```

สังเกตว่า `maxOfAll` ใช้แพทเทิร์นเดียวกับ `printAll` เป๊ะ: **base case แยกต่างหาก** (ในที่นี้
เป็น template ที่รับ argument เดียว แทนที่จะเป็นฟังก์ชันธรรมดาไม่มี argument — เพราะ "ค่ามากสุด
ของกลุ่มที่มีสมาชิกตัวเดียว" ก็คือตัวมันเอง ไม่ต้องมีกรณี 0 argument เพราะไม่มีความหมายทาง
คณิตศาสตร์) และ **recursive case** ที่ดึง argument ตัวแรกออกมาเทียบกับผลลัพธ์ของ argument
ที่เหลือทั้งหมด — นี่คือรูปแบบการคิดแบบ **Divide and Conquer** ที่ทำงานที่ **compile-time**
ทั้งหมด (ทบทวนแนวคิด Recursion จาก Part 25 — ต่างกันตรงที่ตอนนั้นเป็น runtime recursion บน
ค่าตัวแปร แต่ที่นี่เป็น compile-time recursion บนจำนวน type ใน parameter pack)

### ผสมผสาน Variadic Template เข้ากับงานจริง: Logging Function

ตัวอย่างการใช้งานที่พบบ่อยที่สุดของ Variadic Template ในโค้ด production คือฟังก์ชัน logging
ที่รับข้อมูลกี่ชิ้นก็ได้มาประกอบเป็นข้อความเดียว:

```cpp
#include <iostream>
#include <string>

// ผสมผสาน Variadic Template + Fold Expression: logMessage รับ argument กี่ตัวก็ได้
// แล้วพิมพ์แบบมี prefix ระบุระดับความสำคัญของ log
template <typename... Args>
void logMessage(const std::string& level, const Args&... args) {
    std::cout << "[" << level << "] ";
    ((std::cout << args << ' '), ...);   // fold expression (ทบทวนจาก 57.5)
    std::cout << '\n';
}

int main() {
    logMessage("INFO", "เซิร์ฟเวอร์เริ่มทำงานที่พอร์ต", 8080);
    logMessage("ERROR", "เชื่อมต่อฐานข้อมูลล้มเหลว รหัส:", 500, "ลองใหม่ครั้งที่", 3);
    logMessage("DEBUG", "ค่าที่คำนวณได้:", 3.14159, "สถานะ:", true);
    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 logger.cpp -o logger
./logger
```

```
[INFO] เซิร์ฟเวอร์เริ่มทำงานที่พอร์ต 8080 
[ERROR] เชื่อมต่อฐานข้อมูลล้มเหลว รหัส: 500 ลองใหม่ครั้งที่ 3 
[DEBUG] ค่าที่คำนวณได้: 3.14159 สถานะ: 1 
```

ฟังก์ชันเดียวนี้รับ argument ได้ทั้งข้อความ ตัวเลขจำนวนเต็ม ทศนิยม และ boolean ผสมกันในจำนวน
เท่าใดก็ได้ — เทียบกับสมัยก่อนที่ต้องเขียน overload ของ `logMessage` แยกกันสำหรับทุกจำนวน/
ชนิดของ argument ที่เป็นไปได้ (หรือใช้ C-style varargs ที่ไม่ type-safe) เห็นได้ชัดว่า
Variadic Template ทำให้โค้ดสั้นลงมากและปลอดภัยกว่าอย่างมาก (สังเกตว่า `true` ถูกพิมพ์เป็น `1`
เพราะ `std::cout` แสดง `bool` เป็นตัวเลขโดย default — ถ้าต้องการให้แสดงเป็น `true`/`false`
ต้องใช้ `std::boolalpha` ซึ่งเป็นรายละเอียดของ stream formatting ที่ทบทวนได้จาก Part 42)

---

## ข้อผิดพลาดที่พบบ่อย (Common Pitfalls)

1. **พยายามเขียน Partial Specialization ของ Function Template** — C++ **ไม่อนุญาต**
   syntax แบบ `template <typename T> void f<T, int>(...)` สำหรับฟังก์ชัน (ต่างจาก class
   template ที่ทำได้ใน 57.3) ถ้าเจอ compile error ประหลาดที่บอกประมาณ "function template
   partial specialization is not allowed" ให้เปลี่ยนไปใช้ Function Overloading ปกติแทน หรือ
   ถ้าจำเป็นต้องมี logic แบบ partial specialization จริงๆ ให้ห่อ logic นั้นไว้ใน struct/class
   ที่ทำ partial specialization ได้ แล้วเรียกใช้จากฟังก์ชันอีกที (เทคนิคนี้เรียกว่า
   "Tag Dispatch" จะกล่าวถึงใน Part 78)
2. **ลืมเขียน Base Case ให้ Recursive Variadic Template** — ถ้า `printAll` มีแค่ recursive
   case โดยไม่มีฟังก์ชัน `printAll()` (ไม่รับ parameter) รองรับกรณี pack ว่างเปล่า จะได้
   compile error ยาวเหยียดที่บอกประมาณ "no matching function" ณ จุดที่ pack หมดพอดี ข้อความ
   error ของ template ที่ซ้อนกันลึกๆ แบบนี้มักจะยาวมาก ให้อ่านจากบรรทัดบนสุดเสมอ
3. **ใช้ Fold Expression กับ operator ที่ไม่รองรับ หรือ pack ว่างเปล่าโดยไม่ระบุค่าเริ่มต้น** —
   `(args + ...)` เมื่อ `args` มี 0 ตัว จะ compile error ทันที (ไม่มีค่าเริ่มต้นให้ fold) ยกเว้น
   operator บางตัวที่มีค่าเริ่มต้นตามธรรมชาติ (`&&` เริ่มที่ `true`, `||` เริ่มที่ `false`,
   `,` ทำงานได้แม้ pack ว่าง) ถ้าต้องการรองรับกรณี pack ว่างเปล่าอย่างปลอดภัย ให้ใช้
   **Binary Fold** ที่มีค่าเริ่มต้นชัดเจน เช่น `(0 + ... + args)` แทน `(args + ...)`
4. **สับสนระหว่าง Full Specialization กับ Overload ธรรมดา** — การเขียน
   `template<> bool isEqual<const char*>(const char*, const char*)` **ต้องมี** primary
   template (`template <typename T> bool isEqual(T, T)`) ประกาศไว้ก่อนเสมอ ถ้าลืมประกาศ
   primary template จะกลายเป็น error ทันที เพราะ compiler ไม่รู้ว่า specialize อะไรอยู่
5. **ลืมว่า `sizeof...(pack)` ต้องมีวงเล็บครอบชื่อ pack เสมอ** — เขียน `sizeof... args`
   (ไม่มีวงเล็บ) จะเป็น syntax error ต้องเขียน `sizeof...(args)` เท่านั้น และห้ามเว้นวรรค
   ระหว่าง `sizeof` กับ `...` ผิดที่ (ที่ถูกคือ `sizeof...` ติดกันเป็น operator เดียว)
6. **เขียน Ambiguous Partial Specialization โดยไม่รู้ตัว** — ถ้ามี partial specialization
   สองตัวที่ "เจาะจงพอกัน" สำหรับ argument ชุดเดียวกัน (เช่น `Pair<T, T*>` กับ `Pair<T*, T>`
   เมื่อเรียกด้วย `Pair<int*, int*>`) compiler จะไม่สามารถตัดสินได้ว่าจะใช้ตัวไหน เกิด
   **ambiguous specialization** compile error — วิธีป้องกันคือออกแบบ specialization ให้
   ครอบคลุมกรณีที่แยกจากกันชัดเจน หรือเพิ่ม specialization ที่เจาะจงกว่าอีกชั้นเพื่อคลี่คลาย
   ความกำกวม (เช่นเพิ่ม `Pair<T*, T*>` แยกต่างหากสำหรับกรณีที่ทั้งคู่เป็น pointer พร้อมกัน)

---

## แบบฝึกหัดท้ายบท

1. เขียน Full Specialization ของ `isEqual<T>` (จาก 57.1) สำหรับ `float` โดยเฉพาะ ที่เปรียบเทียบ
   ด้วยค่า **epsilon** แทนการใช้ `==` ตรงๆ (เหตุผล: floating point มีปัญหาความคลาดเคลื่อนจาก
   การปัดเศษ ทำให้ค่าที่ "ควรจะ" เท่ากันทางคณิตศาสตร์อาจต่างกันเล็กน้อยในระดับ bit)
2. เขียน Partial Specialization เพิ่มเติมให้ `Pair<T1, T2>` (จาก 57.3) สำหรับกรณีที่ **ทั้งสอง
   parameter เป็น pointer พร้อมกัน** (`Pair<T*, U*>`) ที่พิมพ์เนื้อหาที่ pointer ชี้ไปแทนที่จะ
   พิมพ์ address ตรงๆ (คำใบ้: ระวังปัญหา ambiguous specialization กับ `Pair<T, T>` เดิม —
   ลองคิดดูว่าถ้าเรียก `Pair<int*, int*>` จะเลือก specialization ไหน และจะแก้ไขอย่างไรถ้าเกิด
   ความกำกวม)
3. เขียน variadic function template `sumOfAll` ที่รับ argument กี่ตัวก็ได้ (ชนิดตัวเลขทั้งหมด)
   แบบ **Recursive Variadic Template** (ไม่ใช้ fold expression) แล้วเปรียบเทียบความยาวโค้ดกับ
   `sumAll` แบบ fold expression ใน 57.5
4. เขียน variadic function template `makeVector` ที่รับ argument กี่ตัวก็ได้ (ชนิดเดียวกัน
   ทั้งหมด) แล้วคืนค่าเป็น `std::vector<T>` ที่มีสมาชิกครบตามที่ส่งเข้ามา
5. อธิบายด้วยคำพูดตัวเอง (ไม่ต้องเขียนโค้ด) ว่าทำไม Base Case ของ `printAll` ต้องเขียนเป็น
   ฟังก์ชันธรรมดา (non-template) แทนที่จะปล่อยให้ variadic template จัดการกรณี 0 argument เอง
6. เขียน Full Specialization ของ `TypeDescriber<T>` (จาก 57.2) เพิ่มอีกหนึ่งตัวสำหรับ
   `std::vector<int>` โดยเฉพาะ (ต้อง `#include <vector>`) ที่คืนค่า
   `"เวกเตอร์ของจำนวนเต็ม (std::vector<int>)"`

### แนวทางเฉลยข้อ 1: Full Specialization สำหรับ `float` ด้วย Epsilon

```cpp
#include <cmath>
#include <iostream>

template <typename T>
bool isEqual(T a, T b) {
    std::cout << "[generic version] ";
    return a == b;
}

// Full Specialization สำหรับ float: เปรียบเทียบด้วย epsilon เพราะ floating point
// มีปัญหาเรื่องความคลาดเคลื่อนจากการปัดเศษ (rounding error) การใช้ == ตรงๆ กับ float
// อันตรายมาก เพราะค่าที่ "ควรจะ" เท่ากันทางคณิตศาสตร์ อาจต่างกันเล็กน้อยในระดับ bit
template <>
bool isEqual<float>(float a, float b) {
    std::cout << "[specialized for float, ใช้ epsilon] ";
    const float epsilon = 1e-5f;
    return std::fabs(a - b) < epsilon;
}

int main() {
    std::cout << isEqual(10, 10) << '\n';

    float f1 = 0.1f + 0.2f;   // ผลลัพธ์จริงอาจไม่เท่ากับ 0.3f เป๊ะๆ เพราะ floating point rounding
    float f2 = 0.3f;
    std::cout << isEqual(f1, f2) << '\n';   // ใช้ specialized version -> true ด้วย epsilon

    return 0;
}
```

ผลลัพธ์:

```
[generic version] 1
[specialized for float, ใช้ epsilon] 1
```

จุดสำคัญ: ถ้าลองเปลี่ยนบรรทัดสุดท้ายจากการเรียก `isEqual(f1, f2)` (ที่ deduce เป็น
`isEqual<float>` และใช้ specialized version) ไปเป็นการเปรียบเทียบด้วย `f1 == f2` ตรงๆ แบบ
generic (ลองปิด specialization ดูชั่วคราว) จะพบว่าผลลัพธ์อาจเป็น `false` ได้ในบาง compiler/
optimization level เพราะ `0.1f + 0.2f` ไม่ได้เท่ากับ `0.3f` แบบ bit-exact เสมอไป — นี่คือ
เหตุผลที่ **ห้ามใช้ `==` เปรียบเทียบ floating point ตรงๆ ในโค้ดจริงเด็ดขาด** ต้องใช้เทคนิค
epsilon comparison แบบนี้เสมอ

### แนวทางเฉลยข้อ 3: `sumOfAll` แบบ Recursive Variadic Template

```cpp
#include <iostream>

template <typename T>
T sumOfAll(T value) {
    return value;   // base case: เหลือ argument เดียว ผลรวมก็คือค่านั้นเอง
}

template <typename T, typename... Rest>
T sumOfAll(T first, Rest... rest) {
    return first + sumOfAll(rest...);   // recursion: บวกตัวแรกเข้ากับผลรวมของที่เหลือ
}

int main() {
    std::cout << sumOfAll(1, 2, 3, 4, 5) << '\n';
    std::cout << sumOfAll(1.5, 2.5) << '\n';
    std::cout << sumOfAll(100) << '\n';
    return 0;
}
```

ผลลัพธ์:

```
15
4
100
```

เทียบกับ `sumAll` แบบ fold expression ใน 57.5 (`return (args + ...);` บรรทัดเดียวจบ) จะเห็นว่า
เวอร์ชัน recursive ต้องเขียนถึง 2 function template (base case และ recursive case) ยาวกว่า
ชัดเจน แต่ข้อดีคือมองเห็น**ขั้นตอนการคำนวณทีละ argument ได้ชัดเจนกว่ามาก** ซึ่งมีประโยชน์เมื่อ
ต้องเพิ่ม logic พิเศษระหว่างแต่ละขั้น (เช่น log ค่าทุกครั้งที่บวก หรือหยุด recursion ทันทีเมื่อ
เจอเงื่อนไขบางอย่าง) ซึ่งเป็นสิ่งที่ fold expression ทำได้ยากหรือทำไม่ได้เลยในบางกรณี — นี่คือ
เหตุผลที่ทั้งสองเทคนิคยังคงอยู่คู่กันในโค้ด Modern C++ แทนที่ fold expression จะมาแทนที่
recursive variadic template ไปเสียทั้งหมด

### แนวทางเฉลยข้อ 4: `makeVector` แบบ Variadic

```cpp
#include <iostream>
#include <string>
#include <vector>

// variadic function template ที่รับ argument กี่ตัวก็ได้ (type เดียวกันทั้งหมด) แล้วสร้าง
// std::vector<T> จากมัน
template <typename T, typename... Rest>
std::vector<T> makeVector(T first, Rest... rest) {
    return std::vector<T>{first, rest...};   // ขยาย parameter pack เข้าไปใน brace-init-list
}

int main() {
    auto numbers = makeVector(1, 2, 3, 4, 5);
    for (int n : numbers) {
        std::cout << n << ' ';
    }
    std::cout << '\n';

    auto words = makeVector<std::string>("หนึ่ง", "สอง", "สาม");
    for (const auto& w : words) {
        std::cout << w << ' ';
    }
    std::cout << '\n';

    return 0;
}
```

ผลลัพธ์:

```
1 2 3 4 5 
หนึ่ง สอง สาม 
```

จุดสำคัญ: `makeVector<std::string>("หนึ่ง", "สอง", "สาม")` ต้องระบุ `<std::string>` ชัดเจน
เพราะ `"หนึ่ง"` เป็น `const char*` โดยธรรมชาติ ไม่ใช่ `std::string` — ถ้าไม่ระบุ type เอง
compiler จะ deduce `T = const char*` แทน ทำให้ได้ `std::vector<const char*>` ซึ่งอาจไม่ใช่สิ่ง
ที่ต้องการ (ทบทวนแนวคิดนี้จาก Template Argument Deduction ใน Part 56 ข้อ 56.3 — บทเรียนเดียวกัน
ว่าเมื่อไหร่ต้องระบุ type เองอย่างชัดเจน) ส่วน `std::vector<T>{first, rest...}` ใช้เทคนิคการ
ขยาย parameter pack เข้าไปใน **brace-init-list** โดยตรง ซึ่งเป็นวิธีที่กระชับกว่าการ `push_back`
สมาชิกทีละตัวด้วย recursion

---

## สรุปท้ายบท

Part นี้ต่อยอดจากรากฐาน Template ใน Part 56 ไปสู่เทคนิคขั้นสูงสองกลุ่มที่ใช้บ่อยมากในโค้ด
Modern C++ และเป็นพื้นฐานของ STL ทั้งไลบรารี:

- **Full Specialization** ทั้งของ Function Template และ Class Template — เขียนพฤติกรรมพิเศษ
  สำหรับ type หนึ่งโดยเฉพาะ (เช่น `const char*` ที่ต้องเปรียบเทียบด้วย `strcmp` แทน `==`)
- **Partial Specialization** ของ Class Template — เขียนพฤติกรรมพิเศษสำหรับ**รูปแบบ**ของ type
  (เช่น เมื่อ parameter สองตัวเป็น type เดียวกัน หรือเมื่อเป็น pointer) พร้อมเข้าใจกฎ
  Partial Ordering ที่ compiler ใช้เลือก specialization ที่เจาะจงที่สุด
- **Variadic Template** (`template <typename... Args>`) — เขียนฟังก์ชันและ template ที่รับ
  argument จำนวนเท่าใดก็ได้อย่าง type-safe เต็มรูปแบบ
- **Parameter Pack Expansion** ทั้งสองรูปแบบ: **Recursive Variadic Template** (แบบคลาสสิก
  ยืดหยุ่นสูง) และ **Fold Expression** (C++17 กระชับกว่ามาก) พร้อมรู้ว่าเมื่อไหร่ควรใช้แบบไหน
- ประยุกต์ทุกเทคนิคเข้าด้วยกันเขียน `printAll`, `maxOfAll`, และ `logMessage` ที่ใช้งานได้จริง

เทคนิคทั้งหมดใน Part 56-57 คือรากฐานที่ทำให้ **STL (Standard Template Library)** ทำงานได้อย่าง
ทรงพลัง — container ทุกตัว (`vector`, `map`, `list`) เป็น class template, algorithm ทุกตัว
(`sort`, `find`, `transform`) เป็น function template, และ `std::vector<bool>` ที่เราพูดถึงใน
57.2 ก็คือ full specialization ตัวจริงที่มีอยู่ในไลบรารีมาตรฐาน ตอนนี้เราพร้อมแล้วที่จะก้าวเข้าสู่
**STL อย่างเป็นทางการ** ใน **Part 58** ซึ่งจะแนะนำภาพรวมของ Container, Iterator, Algorithm และ
Function Object ทั้งหมด ก่อนจะเจาะลึกทีละตัวใน Part ถัดๆ ไป

**ต่อไป:** [Part 58 — แนะนำ STL](./part-058-stl-overview.md)
