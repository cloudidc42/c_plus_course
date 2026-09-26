# Part 71: constexpr และ Compile-Time Programming (Step 561–568)

> Module F — Modern C++ (C++11 ถึง C++23) | Part 71 จาก 125
> Step ที่ครอบคลุมใน Part นี้: Step 561–568
> Part ก่อนหน้า: [Part 70 — Move Semantics และ Rvalue Reference](./part-070-move-semantics.md) | Part ถัดไป: [Part 72 — ฟีเจอร์ C++14/17](./part-072-cpp14-17-features.md)

## เป้าหมายการเรียนรู้ (Learning Objectives)

หลังจบ Part นี้ ผู้เรียนจะสามารถ:

1. แยกแยะความแตกต่างสำคัญระหว่าง **`const`** และ **`constexpr`** ได้อย่างชัดเจน: `const` แค่
   ห้ามแก้ไขค่าหลังกำหนด แต่ `constexpr` บังคับให้ค่านั้นต้องรู้ได้ตั้งแต่ compile-time เสมอ
2. เขียน **constexpr function** ที่ทำงานได้ทั้งตอน compile-time และ runtime พร้อมเข้าใจ
   ข้อจำกัดที่เข้มงวดของ C++11 (ต้องมีแค่ `return` statement เดียว)
3. ใช้ประโยชน์จากการผ่อนคลายข้อจำกัดของ `constexpr` function ใน **C++14/17** (loop, ตัวแปร
   local, `if`/`else` ได้ตามใจ) เพื่อเขียนโค้ดที่อ่านง่ายขึ้นมากโดยไม่เสียความสามารถ compile-time
4. อธิบายได้ว่า `constexpr` เป็นสะพานเชื่อมไปสู่ **template metaprogramming** (Part 78) อย่างไร
   ผ่านการใช้ผลลัพธ์ของ `constexpr` function เป็น non-type template argument หรือขนาด array
5. พิสูจน์ได้ว่าการคำนวณหนึ่งๆ เกิดขึ้นที่ compile-time จริง ผ่านทั้ง `static_assert` และการอ่าน
   assembly output ที่ compiler สร้างขึ้น
6. ใช้ **`consteval`** (C++20) เพื่อบังคับให้ฟังก์ชันหนึ่งต้องถูกประเมินผลที่ compile-time เสมอ
   ไม่มีข้อยกเว้น และอธิบายความแตกต่างจาก `constexpr` ได้อย่างชัดเจน
7. เขียนตัวอย่างการใช้งานจริง เช่นการคำนวณ **factorial** และ **fibonacci** (รวมถึงจำนวนเฉพาะ)
   ให้เกิดขึ้นทั้งหมดที่ compile-time โดยไม่มีการคำนวณเหลืออยู่ตอนโปรแกรมรันจริงเลย

---

## 71.1 const กับ constexpr: ความแตกต่างที่สำคัญที่สุด (Step 561)

`const` เป็นคีย์เวิร์ดที่คุ้นเคยมาตั้งแต่ **Part 43** — มันบอกว่า **ตัวแปรนี้ห้ามแก้ไขค่าหลังจาก
กำหนดค่าเริ่มต้นแล้ว** แต่สิ่งที่หลายคนเข้าใจผิดคือ `const` **ไม่ได้บังคับว่าค่านั้นต้องรู้ตั้งแต่
compile-time** — ค่าของตัวแปร `const` สามารถมาจาก input ผู้ใช้, ไฟล์, หรือการคำนวณที่ซับซ้อนตอน
runtime ก็ได้ ขอแค่หลังจากกำหนดค่าแล้วห้ามแก้ไขอีกก็พอ

`constexpr` (เพิ่มเข้ามาใน C++11) เข้มงวดกว่ามาก: มันบอกว่า **ค่านี้ต้องสามารถคำนวณได้ที่
compile-time เสมอ** (compiler จะพยายามคำนวณมันในขณะ compile ไม่ใช่รอจนโปรแกรมรันจริง) ถ้าค่าที่
ให้มาไม่ใช่ compile-time constant จะเกิด **compile error ทันที**

```cpp
// 01_const_vs_constexpr.cpp
#include <iostream>

int main() {
    int runtimeInput = 7; // เปรียบเสมือนค่าที่มาจากผู้ใช้ตอน runtime (ในตัวอย่างนี้ hardcode เพื่อไม่ต้องรอ input)

    const int a = runtimeInput;   // ถูกต้อง: const แค่ "ห้ามแก้ไขค่า a หลังจากนี้" เท่านั้น
                                    // ค่าของ a ยังคงถูกกำหนดตอน runtime ได้ตามปกติ (ไม่บังคับ compile-time)
    std::cout << "a = " << a << " (const แต่รู้ค่าตอน runtime)\n";

    constexpr int b = 42;          // ถูกต้อง: 42 เป็นค่าคงที่ที่ compiler รู้ตั้งแต่ compile-time
    // constexpr int c = runtimeInput; // ผิด! runtimeInput ไม่ใช่ compile-time constant -> compile error ทันที

    std::cout << "b = " << b << " (constexpr: compiler รู้ค่าตั้งแต่ compile-time เสมอ)\n";

    // ประโยชน์ที่จับต้องได้: constexpr ใช้กำหนดขนาด array แบบ fixed-size ได้ (ต้องรู้ค่าตอน compile-time)
    constexpr int arraySize = 10;
    int fixedArray[arraySize]; // ถูกต้อง: arraySize เป็น compile-time constant
    for (int i = 0; i < arraySize; ++i) fixedArray[i] = i * i;
    std::cout << "fixedArray[5] = " << fixedArray[5] << '\n';

    // int badArray[a]; // ในมาตรฐาน C++ ปกติ ผิด! a เป็นแค่ const runtime-determined ไม่ใช่ compile-time constant
    //                   // (แม้ GCC/Clang บาง flag จะยอมแบบ extension ก็ตาม แต่ไม่ portable และ -Wpedantic จะเตือน)

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 01_const_vs_constexpr.cpp -o 01_ce
./01_ce
```

```
a = 7 (const แต่รู้ค่าตอน runtime)
b = 42 (constexpr: compiler รู้ค่าตั้งแต่ compile-time เสมอ)
fixedArray[5] = 25
```

### ตารางเปรียบเทียบ const กับ constexpr

| คุณสมบัติ | `const` | `constexpr` |
|---|---|---|
| ห้ามแก้ไขค่าหลังกำหนด | ใช่ | ใช่ |
| ต้องรู้ค่าตั้งแต่ compile-time | **ไม่จำเป็น** (อาจมาจาก runtime) | **บังคับเสมอ** |
| ใช้กำหนดขนาด array แบบ fixed-size ได้ | ไม่ได้ (ในมาตรฐานมาตรฐาน) | **ได้เสมอ** |
| ใช้เป็น non-type template argument ได้ | ไม่ได้ (ถ้าไม่ใช่ compile-time constant) | **ได้เสมอ** |
| ใช้กับฟังก์ชันได้ (`constexpr` function) | ไม่มีความหมายแบบนี้ | **ได้** ทำให้ฟังก์ชันคำนวณที่ compile-time ได้ |

> **กฎจำง่าย**: `const` ตอบคำถาม "แก้ไขค่านี้ได้ไหม" (**คำตอบ: ไม่ได้**) ส่วน `constexpr`
> ตอบคำถาม "รู้ค่านี้ได้ตอนไหน" (**คำตอบ: ตั้งแต่ compile-time**) — `constexpr` ทุกตัวเป็น
> `const` โดยปริยายเสมอ (แก้ไขไม่ได้อยู่แล้ว) แต่ `const` ไม่จำเป็นต้องเป็น `constexpr`

---

## 71.2 constexpr Function: ข้อจำกัดที่เข้มงวดของ C++11 (Step 562)

`constexpr` ไม่ได้ใช้ได้แค่กับตัวแปรเท่านั้น แต่ใช้กับ **ฟังก์ชัน** ได้ด้วย — ฟังก์ชันที่ประกาศ
เป็น `constexpr` สามารถถูกเรียกได้ **ทั้งตอน compile-time และ runtime** (ต่างจาก `consteval`
ที่จะเรียนในหัวข้อ 71.7 ซึ่งบังคับ compile-time เท่านั้น) — ถ้าอาร์กิวเมนต์ที่ส่งเข้ามาเป็น
compile-time constant compiler จะพยายามคำนวณผลลัพธ์ตอน compile ให้ แต่ถ้าอาร์กิวเมนต์เป็นค่า
runtime มันก็ยังทำงานเป็นฟังก์ชันธรรมดาได้ตามปกติ

ข้อควรระวังคือ **C++11 กำหนดข้อจำกัดที่เข้มงวดมาก** สำหรับ constexpr function: ตัวฟังก์ชันต้อง
มี **แค่ `return` statement เดียวเท่านั้น** ไม่มีตัวแปร local ใหม่, ไม่มี loop, ไม่มี `if`/`else`
แบบ statement (ใช้ ternary operator `?:` แทนได้) วิธีเดียวที่จะทำ logic ที่ซับซ้อนได้คือใช้
**recursion** ร่วมกับ conditional operator

```cpp
// 02_cpp11_constexpr.cpp - constexpr function สไตล์ C++11 (ข้อจำกัดเข้มงวด)
#include <iostream>

// C++11 บังคับว่า constexpr function ต้องมี "แค่ return statement เดียว" ในตัว (ไม่มี if, loop,
// ตัวแปร local ใหม่ ฯลฯ) วิธีเดียวที่จะทำ logic ซับซ้อนได้คือใช้ recursion + conditional operator
constexpr int factorialCpp11(int n) {
    return (n <= 1) ? 1 : n * factorialCpp11(n - 1);
}

constexpr int maxCpp11(int a, int b) {
    return (a > b) ? a : b;
}

int main() {
    constexpr int f5 = factorialCpp11(5);   // คำนวณที่ compile-time (ยืนยันด้วย static_assert ด้านล่าง)
    static_assert(f5 == 120, "5! ต้องเท่ากับ 120");

    constexpr int m = maxCpp11(3, 7);
    static_assert(m == 7, "max(3, 7) ต้องเท่ากับ 7");

    std::cout << "5! = " << f5 << ", max(3,7) = " << m << '\n';

    // เรียกด้วยค่า runtime ก็ได้เหมือนกัน (constexpr function ใช้ได้ทั้ง compile-time และ runtime)
    int n;
    std::cout << "ทดสอบเรียกด้วยค่า runtime: ";
    n = 6;
    std::cout << "factorialCpp11(" << n << ") = " << factorialCpp11(n) << '\n';

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++11 02_cpp11_constexpr.cpp -o 02_ce
./02_ce
```

```
5! = 120, max(3,7) = 7
ทดสอบเรียกด้วยค่า runtime: factorialCpp11(6) = 720
```

สังเกตว่าโค้ดนี้คอมไพล์ด้วย `-std=c++11` ได้สำเร็จเพราะ `factorialCpp11` มีแค่ `return`
statement เดียว (ใช้ ternary operator `?:` แทนการเขียน `if`/`else`) และใช้ **recursion**
แทนการเขียน loop — นี่คือสำนวนบังคับที่โปรแกรมเมอร์ C++11 ยุคแรกต้องคุ้นเคย และเป็นเหตุผลสำคัญ
ที่ทำให้หลายคนมองว่า `constexpr` ใน C++11 "ใช้งานยาก" และ "จำกัดเกินไป" จนนำไปสู่การผ่อนคลาย
กฎครั้งใหญ่ใน C++14 (หัวข้อถัดไป)

---

## 71.3 C++14/17 ผ่อนคลายข้อจำกัดของ constexpr Function อย่างมาก (Step 563)

C++14 แก้ปัญหาความเข้มงวดเกินไปของ C++11 โดยผ่อนคลายกฎเกือบทั้งหมด: **constexpr function ใน
C++14 ขึ้นไปมีตัวแปร local, ใช้ loop (`for`, `while`), ใช้ `if`/`else` แบบ statement ได้ตามใจ**
แทบจะเขียนเหมือนฟังก์ชันธรรมดาทุกประการ ต่างกันแค่ compiler จะพยายามประเมินผลที่ compile-time
ให้เมื่อเป็นไปได้

```cpp
// 03_cpp14_relaxed.cpp - C++14 ผ่อนคลายข้อจำกัดของ constexpr function มาก
#include <iostream>

// C++14 ขึ้นไป: constexpr function มีตัวแปร local, loop, if/else, หลาย statement ได้ตามใจ
// (แทบจะเขียนเหมือนฟังก์ชันธรรมดาทุกประการ ต่างกันแค่ compiler "พยายาม" ประเมินที่ compile-time ให้)
constexpr int factorialCpp14(int n) {
    int result = 1;                 // ประกาศตัวแปร local ได้ (C++11 ทำไม่ได้)
    for (int i = 2; i <= n; ++i) {  // ใช้ loop ได้ (C++11 ทำไม่ได้)
        result *= i;
    }
    return result;
}

constexpr int fibonacciCpp14(int n) {
    if (n <= 1) return n;            // ใช้ if ได้ตามใจ (C++11 ทำไม่ได้ในความหมายนี้)
    int a = 0, b = 1;
    for (int i = 2; i <= n; ++i) {
        int next = a + b;
        a = b;
        b = next;
    }
    return b;
}

int main() {
    constexpr int f10 = factorialCpp14(10);
    static_assert(f10 == 3628800, "10! ต้องเท่ากับ 3628800");

    constexpr int fib10 = fibonacciCpp14(10);
    static_assert(fib10 == 55, "fibonacci(10) ต้องเท่ากับ 55");

    std::cout << "10! = " << f10 << '\n';
    std::cout << "fibonacci(10) = " << fib10 << '\n';

    return 0;
}
```

ลองคอมไพล์ด้วย `-std=c++11` เทียบกับ `-std=c++14` จะเห็นความแตกต่างชัดเจนมาก:

```bash
g++ -Wall -Wextra -Wpedantic -std=c++11 03_cpp14_relaxed.cpp -o 03_c11
```

```
03_cpp14_relaxed.cpp: In function 'constexpr int factorialCpp14(int)':
03_cpp14_relaxed.cpp:12:1: error: body of 'constexpr' function 'constexpr int factorialCpp14(int)' not a return-statement
   12 | }
      | ^
03_cpp14_relaxed.cpp: In function 'constexpr int fibonacciCpp14(int)':
03_cpp14_relaxed.cpp:23:1: error: body of 'constexpr' function 'constexpr int fibonacciCpp14(int)' not a return-statement
...
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++14 03_cpp14_relaxed.cpp -o 03_c14
./03_c14
```

```
10! = 3628800
fibonacci(10) = 55
```

โค้ดตัวเดียวกันเป๊ะ คอมไพล์ไม่ผ่านด้วย `-std=c++11` (เพราะมีตัวแปร local และ loop ในตัว
constexpr function) แต่คอมไพล์ผ่านและทำงานถูกต้องทันทีด้วย `-std=c++14` — นี่คือหลักฐานที่จับ
ต้องได้ว่ากฎของ `constexpr` function เปลี่ยนแปลงไปมากขนาดไหนระหว่างสองมาตรฐาน ตลอดหลักสูตรนี้
เราคอมไพล์ด้วย `-std=c++17` เป็นค่าเริ่มต้น ซึ่งสืบทอดกฎที่ผ่อนคลายของ C++14 มาเต็มรูปแบบ (และ
C++17 เพิ่มความสามารถอีกเล็กน้อย เช่นอนุญาตให้ใช้ใน `if constexpr` ที่จะเรียนใน Part 72)

> **ข้อควรรู้**: แม้กฎจะผ่อนคลายมากแล้ว แต่ constexpr function ยังคงมีข้อจำกัดพื้นฐานบางอย่าง
> เสมอ เช่น **ห้ามมี side effect ที่มองเห็นได้จากภายนอก** (เช่นห้ามแก้ไข global variable, ห้าม
> เรียก `std::cout`, ห้าม throw exception แบบไม่มีเงื่อนไข) เพราะสิ่งเหล่านี้ไม่มีความหมายใน
> โลกของ compile-time evaluation — compiler จะรายงาน error ทันทีถ้าเราลองใส่โค้ดเหล่านี้เข้าไป
> ในฟังก์ชันที่จะถูกเรียกใช้งานที่ compile-time จริงๆ

---

## 71.4 constexpr กับ Template Metaprogramming: สะพานเชื่อมสู่ Part 78 (Step 564)

หนึ่งในประโยชน์ที่ทรงพลังที่สุดของ `constexpr` function คือมันเป็น **ตัวแทนที่อ่านง่ายกว่ามาก**
สำหรับสิ่งที่ก่อน C++11 ต้องทำผ่าน **recursive template metaprogramming** (เทคนิคเก่าที่ใช้
template specialization คำนวณค่าที่ compile-time ซึ่งมีไวยากรณ์ที่อ่านยากและซับซ้อนมาก — จะ
เจาะลึกเทคนิคเก่านี้เปรียบเทียบกันใน **Part 78 — Metaprogramming**) `constexpr` function ทำให้
เราคำนวณค่าที่ compile-time ด้วย **โค้ด C++ ปกติที่อ่านง่าย** แทน

```cpp
// 04_constexpr_template.cpp - constexpr เป็นสะพานเชื่อมสู่ template metaprogramming (Part 78)
#include <array>
#include <iostream>

constexpr int square(int n) {
    return n * n;
}

// Non-type template parameter ต้องเป็นค่าที่รู้ตอน compile-time เท่านั้น
// constexpr function ทำให้เราคำนวณค่านั้นด้วยโค้ด C++ ธรรมดา แทนที่จะพึ่ง template metaprogramming
// แบบ recursive template ที่อ่านยากแบบเก่า (ก่อน C++11)
template <int N>
struct FixedBuffer {
    std::array<int, N> data{};
    static constexpr int capacity = N;
};

int main() {
    // ใช้ผลลัพธ์ของ constexpr function เป็นขนาดของ std::array ได้โดยตรง
    std::array<int, square(4)> arr{}; // เทียบเท่า std::array<int, 16>
    std::cout << "arr.size() = " << arr.size() << '\n';

    // ใช้เป็น non-type template argument ได้โดยตรงเช่นกัน
    FixedBuffer<square(3)> buffer; // เทียบเท่า FixedBuffer<9>
    std::cout << "buffer.capacity = " << buffer.capacity << '\n';
    std::cout << "buffer.data.size() = " << buffer.data.size() << '\n';

    static_assert(square(5) == 25, "square(5) ต้องเท่ากับ 25 เสมอ ตรวจสอบได้ที่ compile-time");

    return 0;
}
```

```
arr.size() = 16
buffer.capacity = 9
buffer.data.size() = 9
```

ก่อน C++11 การคำนวณ `square(4)` ที่ compile-time ต้องเขียนเป็น template metaprogram แบบนี้แทน:

```cpp
// สไตล์เก่าก่อน C++11 (แค่โชว์เป็นตัวอย่างเปรียบเทียบ ไม่ต้องเข้าใจลึกตอนนี้ จะเจาะลึกใน Part 78)
template <int N>
struct Square {
    static constexpr int value = N * N;
};
// ใช้งาน: Square<4>::value  (อ่านยากกว่า square(4) มาก และขยายเป็น logic ซับซ้อนได้ยากกว่ามาก)
```

เห็นได้ชัดว่า `constexpr` function อ่านง่ายกว่ามาก และเขียน logic ที่ซับซ้อน (loop, condition)
ได้ตามธรรมชาติ นี่คือเหตุผลที่ตั้งแต่ C++11/14 เป็นต้นมา นักพัฒนา C++ นิยมใช้ `constexpr`
function แทนเทคนิค template metaprogramming แบบเก่าในแทบทุกกรณีที่เป็นไปได้ (เก็บเทคนิคแบบเก่า
ไว้ใช้เฉพาะกรณีที่ต้องการเลือก type ที่ compile-time ซึ่ง `constexpr` function ทำไม่ได้ — จะ
อธิบายรายละเอียดใน Part 78)

---

## 71.5 พิสูจน์การคำนวณที่ Compile-Time ด้วย static_assert (Step 565)

คำถามที่สำคัญคือ: เรามั่นใจได้อย่างไรว่าการเรียก `constexpr` function หนึ่งๆ ถูกคำนวณที่
**compile-time จริง** ไม่ใช่แค่ "เขียนโค้ดที่ทำงานถูกต้องตอน runtime" เฉยๆ? มีสองวิธีหลักที่ใช้
พิสูจน์ได้อย่างแน่ชัด วิธีแรกคือการใช้ `static_assert` และตัวแปร `constexpr`

```cpp
// 05a_static_assert_proof.cpp - พิสูจน์ compile-time evaluation ด้วย static_assert
#include <iostream>

constexpr long long factorial(int n) {
    long long result = 1;
    for (int i = 2; i <= n; ++i) result *= i;
    return result;
}

int main() {
    // วิธีที่ 1: static_assert -- ถ้า factorial(10) ไม่ใช่ compile-time constant โปรแกรมจะ compile ไม่ผ่านเลย
    static_assert(factorial(10) == 3628800LL, "ต้องคำนวณได้ที่ compile-time");

    // วิธีที่ 2: มอบค่าให้ constexpr variable -- ถ้าคำนวณไม่ได้ตอน compile-time จะ error ทันที
    constexpr long long compileTimeResult = factorial(15);
    std::cout << "15! (compile-time) = " << compileTimeResult << '\n';

    return 0;
}
```

```
15! (compile-time) = 1307674368000
```

ทั้งสองวิธีนี้เป็น **การพิสูจน์เชิงตรรกะ** ไม่ใช่แค่การสังเกต: `static_assert` **ต้องการ**
compile-time constant expression เป็นเงื่อนไข ถ้า `factorial(10)` ไม่สามารถคำนวณได้ที่
compile-time (เช่นถ้าลืมใส่ `constexpr` หน้าฟังก์ชัน หรือฟังก์ชันมี side effect ที่ทำให้ไม่ใช่
constant expression) **โปรแกรมจะ compile ไม่ผ่านทันที** เช่นเดียวกับการกำหนดค่าให้ตัวแปร
`constexpr` — ถ้าไม่มั่นใจ 100% ว่าเป็น compile-time evaluation ได้จริง ให้ลองลบคำว่า `constexpr`
ออกจากหน้าฟังก์ชัน `factorial` ดู จะเห็นว่าทั้ง `static_assert` และการกำหนดค่าตัวแปร `constexpr`
ทั้งสองบรรทัด **compile ไม่ผ่านทันที** เพราะฟังก์ชันธรรมดาไม่สามารถถูกเรียกในบริบทที่ต้องการ
compile-time constant ได้เลย

---

## 71.6 พิสูจน์การคำนวณที่ Compile-Time ด้วยการอ่าน Assembly Output (Step 566)

วิธีที่สองที่ "เห็นภาพ" ชัดเจนยิ่งกว่าคือการดู **assembly output** ที่ compiler สร้างขึ้นจริง —
ถ้าค่าถูกคำนวณที่ compile-time แล้ว โปรแกรมที่รันจริง **ไม่ควรมีการคำนวณเหลืออยู่เลย** ค่านั้น
ควรปรากฏเป็นแค่ **ค่าคงที่ (immediate value)** ฝังอยู่ในโค้ดโดยตรง

```cpp
// 05b_prove_compile_time_asm.cpp - พิสูจน์ด้วยการเปรียบเทียบ assembly
#include <iostream>

constexpr long long factorial(int n) {
    long long result = 1;
    for (int i = 2; i <= n; ++i) result *= i;
    return result;
}

int main() {
    // มอบค่าให้ constexpr variable -- คำนวณที่ compile-time แน่นอน
    constexpr long long compileTimeResult = factorial(15);
    std::cout << "15! (compile-time) = " << compileTimeResult << '\n';

    // เทียบกับเวอร์ชันที่ "บังคับ" ให้คำนวณตอน runtime โดยใช้ volatile กันไม่ให้ compiler มองเห็น
    // ค่าคงที่ตั้งแต่ compile-time (volatile บอก compiler ว่า "ค่านี้อาจเปลี่ยนได้จากภายนอก
    // ห้าม optimize ทิ้ง") แล้วดู assembly (-S) เทียบกันว่าต่างกันจริง
    volatile int runtimeN = 15;
    long long runtimeResult = factorial(runtimeN); // ต้องคำนวณจริงตอนรัน เพราะ runtimeN ไม่ใช่ compile-time constant
    std::cout << "15! (runtime, ผ่าน volatile)  = " << runtimeResult << '\n';

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 -O2 05b_prove_compile_time_asm.cpp -o 05b
./05b
g++ -Wall -Wextra -Wpedantic -std=c++17 -O2 -S 05b_prove_compile_time_asm.cpp -o 05b.s
```

```
15! (compile-time) = 1307674368000
15! (runtime, ผ่าน volatile)  = 1307674368000
```

เปิดไฟล์ `05b.s` ขึ้นมาดู จะพบความแตกต่างที่ชัดเจนมากระหว่างสองส่วน:

```asm
; ส่วนของ compileTimeResult (ผ่าน constexpr):
        movabsq $1307674368000, %rsi     ; ค่าคงที่ถูกฝังตรงๆ ไม่มีการคำนวณเหลืออยู่เลย!

; ส่วนของ runtimeResult (ผ่าน volatile):
.L7:
        imulq   %rax, %rbx               ; มี loop คูณเลขจริงๆ เกิดขึ้นตอนโปรแกรมรัน
        ...
.L8:
        imulq   %rdx, %rbx
        ...
```

ผลลัพธ์ของทั้งสองค่าเท่ากันทุกประการ (`1307674368000`) แต่เบื้องหลังต่างกันโดยสิ้นเชิง:
`compileTimeResult` ปรากฏเป็นคำสั่ง `movabsq` ตัวเดียวที่ฝังค่าคงที่ตรงๆ ลงในโปรแกรม (ไม่มีการ
คูณเลขใดๆ เหลืออยู่เลยตอนรันจริง) ในขณะที่ `runtimeResult` ที่ถูกบังคับให้คำนวณตอน runtime ผ่าน
`volatile` (ซึ่งบอก compiler ว่า "ห้าม optimize ค่านี้ทิ้ง เพราะมันอาจเปลี่ยนแปลงได้จากภายนอก
โปรแกรม") ยังคงมีคำสั่ง `imulq` (คูณเลข) อยู่ในลูปจริงๆ — นี่คือหลักฐานที่จับต้องได้ที่สุดว่า
`constexpr` ไม่ได้เป็นแค่ "คำสัญญา" แต่ส่งผลจริงต่อโค้ดที่ compiler สร้างขึ้นมา

---

## 71.7 consteval (C++20): บังคับ Compile-Time เสมอ (Step 567)

`constexpr` function ที่เรียนมาทั้งหมดมีจุดหนึ่งที่อาจไม่เหมาะกับบางสถานการณ์: **มันยอมให้
เรียกด้วยค่า runtime ได้ด้วย** (ถ้าอาร์กิวเมนต์ไม่ใช่ compile-time constant มันก็แค่ทำงานเป็น
ฟังก์ชันธรรมดาที่รันตอน runtime) บางครั้งเราต้องการ **บังคับ 100%** ว่าฟังก์ชันหนึ่งต้องถูก
ประเมินผลที่ compile-time เท่านั้นเสมอ ไม่มีข้อยกเว้น (เช่นฟังก์ชันที่ validate ค่าคงที่ตอน
compile หรือฟังก์ชันที่ผลลัพธ์ควรถูกฝังในโปรแกรมเสมอด้วยเหตุผลด้านความปลอดภัยหรือประสิทธิภาพ)

C++20 เพิ่มคีย์เวิร์ด **`consteval`** มาแก้ปัญหานี้โดยเฉพาะ — ฟังก์ชันที่ประกาศเป็น
`consteval` (เรียกว่า **immediate function**) **ต้อง** ถูกประเมินผลที่ compile-time ทุกครั้งที่
ถูกเรียก ถ้าไม่สามารถทำได้ (เช่นอาร์กิวเมนต์เป็นค่า runtime) จะเกิด **compile error ทันที**

> **ต้องคอมไพล์ด้วย `-std=c++20` สำหรับตัวอย่างในหัวข้อนี้** เพราะ `consteval` เป็นฟีเจอร์ของ
> C++20 เท่านั้น (จะเจาะลึกฟีเจอร์อื่นๆ ของ C++20 อย่างเต็มรูปแบบใน Part 73-76)

```cpp
// 06_consteval.cpp - C++20 (ต้องคอมไพล์ด้วย -std=c++20)
#include <iostream>

constexpr int squareConstexpr(int n) { return n * n; }
consteval int squareConsteval(int n) { return n * n; } // ต้อง evaluate ที่ compile-time เสมอ ห้ามรันจริงตอน runtime เด็ดขาด

int main() {
    volatile int runtimeN = 5;

    std::cout << squareConstexpr(4) << '\n';              // compile-time (ค่าคงที่ตรงๆ)
    std::cout << squareConstexpr(runtimeN) << '\n';        // runtime ก็ได้ (constexpr ไม่บังคับ)

    std::cout << squareConsteval(4) << '\n';               // ผ่าน: 4 เป็น compile-time constant

    // squareConsteval(runtimeN); // ผิด! compile error ทันที เพราะ runtimeN ไม่ใช่ compile-time constant
    //                             // consteval "บังคับ" ทุกการเรียกต้องประเมินผลที่ compile-time เท่านั้น

    constexpr int fixed = squareConsteval(6); // ก็ต้องเรียกในบริบทที่เป็น compile-time constant เท่านั้น
    std::cout << "fixed = " << fixed << '\n';

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++20 06_consteval.cpp -o 06_ce
./06_ce
```

```
16
25
16
fixed = 36
```

ถ้าลอง uncomment บรรทัด `squareConsteval(runtimeN);` แล้วคอมไพล์ใหม่ จะได้ error ที่ชัดเจนมาก:

```
error: the value of 'runtimeN' is not usable in a constant expression
note: 'volatile int runtimeN' is not const
```

### ตารางเปรียบเทียบ constexpr กับ consteval

| คุณสมบัติ | `constexpr` function | `consteval` function |
|---|---|---|
| เรียกด้วยค่า runtime ได้ | **ได้** (ทำงานเป็นฟังก์ชันธรรมดา) | **ไม่ได้เลย** compile error ทันที |
| เรียกด้วยค่า compile-time constant | ได้ (compiler พยายามคำนวณให้) | ได้ (บังคับคำนวณเสมอ) |
| ใช้ได้ตั้งแต่มาตรฐานไหน | C++11 (ผ่อนคลายเพิ่มใน C++14/17) | **C++20** เท่านั้น |
| เหมาะกับสถานการณ์ | ฟังก์ชันที่อยากให้ "เร็วขึ้นเมื่อทำได้" แต่ยังใช้กับค่า runtime ได้ด้วย | ฟังก์ชันที่ต้อง "รับประกัน 100%" ว่าประมวลผลที่ compile-time เท่านั้น (เช่น การ validate ค่าคงที่, compile-time hashing) |

> **กฎการเลือกใช้**: ใช้ `constexpr` เป็นค่าเริ่มต้นเสมอสำหรับฟังก์ชันที่ **อาจ** ถูกเรียกด้วย
> ค่า compile-time หรือ runtime ก็ได้ (ยืดหยุ่นกว่า ใช้ได้กว้างกว่า) ใช้ `consteval` เฉพาะเมื่อ
> ต้องการ **บังคับ** ว่าฟังก์ชันนั้นต้องไม่มีทางถูกเรียกด้วยค่า runtime เด็ดขาด (เช่นฟังก์ชันที่
> ใช้สร้างค่าคงที่สำหรับ template parameter หรือฟังก์ชันที่การรันตอน runtime จะไม่มีความหมายอะไร
> เลย)

---

## 71.8 ตัวอย่างจริง: Factorial, Fibonacci, และจำนวนเฉพาะ ที่ Compile-Time (Step 568)

มาถึงหัวข้อสุดท้ายของ Part นี้ เราจะรวบยอดทุกอย่างที่เรียนมาเข้าด้วยกันเป็นตัวอย่างที่ใช้งานได้
จริง: สร้าง **ตาราง fibonacci** ที่คำนวณเสร็จสมบูรณ์ตั้งแต่ compile-time โดยใช้
`std::array` ร่วมกับ `constexpr` function

```cpp
// 07_factorial_fibonacci_table.cpp - ตัวอย่างจริง: สร้างตาราง factorial/fibonacci ที่ compile-time
#include <array>
#include <iostream>

constexpr long long factorial(int n) {
    return (n <= 1) ? 1 : n * factorial(n - 1);
}

constexpr int fibonacci(int n) {
    int a = 0, b = 1;
    for (int i = 0; i < n; ++i) {
        int next = a + b;
        a = b;
        b = next;
    }
    return a;
}

// สร้างตาราง fibonacci ทั้ง 15 ค่าแรก "ที่ compile-time ทั้งหมด" ด้วย constexpr function ที่คืน std::array
// -- โปรแกรมที่รันจริงจะไม่มีการคำนวณ loop นี้เกิดขึ้นเลยแม้แต่ครั้งเดียวตอน runtime
constexpr std::array<int, 15> makeFibonacciTable() {
    std::array<int, 15> table{};
    for (std::size_t i = 0; i < table.size(); ++i) {
        table[i] = fibonacci(static_cast<int>(i));
    }
    return table;
}

int main() {
    constexpr long long f10 = factorial(10);
    static_assert(f10 == 3628800LL);

    constexpr auto fibTable = makeFibonacciTable(); // ตารางทั้งหมดฝังอยู่ในไฟล์ executable แล้วตั้งแต่ compile
    static_assert(fibTable[10] == 55, "fibonacci(10) ต้องเท่ากับ 55");

    std::cout << "10! = " << f10 << '\n';
    std::cout << "ตาราง fibonacci 15 ค่าแรก: ";
    for (int v : fibTable) std::cout << v << ' ';
    std::cout << '\n';

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 07_factorial_fibonacci_table.cpp -o 07_table
./07_table
```

```
10! = 3628800
ตาราง fibonacci 15 ค่าแรก: 0 1 1 2 3 5 8 13 21 34 55 89 144 233 377
```

ผลลัพธ์ที่สำคัญที่สุดของตัวอย่างนี้ไม่ใช่แค่ตัวเลขที่พิมพ์ออกมา แต่คือ **สิ่งที่ไม่เกิดขึ้นตอน
รันโปรแกรม**: `fibTable` ทั้ง 15 ค่า ถูกคำนวณเสร็จสมบูรณ์ตั้งแต่ตอน compile และฝังอยู่ในไฟล์
executable เป็นข้อมูลคงที่ (คล้ายกับที่พิสูจน์ด้วย assembly ในหัวข้อ 71.6) — โปรแกรมที่รันจริง
แค่ **อ่าน** ค่าที่คำนวณไว้แล้วออกมาพิมพ์เท่านั้น ไม่มีการวนลูปคำนวณ fibonacci เกิดขึ้นเลยแม้แต่
ครั้งเดียวตอน runtime ซึ่งเป็นประโยชน์อย่างมากสำหรับโปรแกรมที่ต้องการ **lookup table** ขนาดเล็ก
ถึงกลางที่ใช้ค่าคงที่ซ้ำๆ (เช่นตาราง sine/cosine ในเกม, ตาราง CRC checksum, ตารางการแปลงหน่วย)
เพราะได้ทั้งความเร็วสูงสุด (ไม่มีการคำนวณตอน runtime เลย) และความปลอดภัยสูงสุด (ค่าถูกตรวจสอบ
ด้วย `static_assert` ตั้งแต่ตอน compile ว่าถูกต้องแน่นอน)

---

## ข้อผิดพลาดที่พบบ่อย (Common Pitfalls)

1. **เข้าใจว่า `const` กับ `constexpr` เป็นสิ่งเดียวกัน** — `const` ห้ามแก้ไขค่า แต่ไม่บังคับว่า
   ต้องรู้ค่าตั้งแต่ compile-time (อาจมาจาก runtime ก็ได้) ส่วน `constexpr` บังคับว่าต้องรู้ค่า
   ตั้งแต่ compile-time เสมอ (หัวข้อ 71.1)
2. **เขียน `constexpr` function ที่มี loop/ตัวแปร local แล้วสงสัยว่าทำไม compile ไม่ผ่านด้วย
   `-std=c++11`** — ต้องจำไว้ว่า C++11 มีข้อจำกัดเข้มงวดมาก (แค่ `return` statement เดียว) การ
   ผ่อนคลายเกิดขึ้นใน C++14 เป็นต้นไป ถ้าโค้ดเบสยังต้องรองรับ compiler เก่าที่ตั้งไว้เป็น C++11
   ต้องเขียนด้วยสไตล์ recursion + ternary operator แทน (หัวข้อ 71.2-71.3)
3. **คิดว่าฟังก์ชัน `constexpr` จะถูกคำนวณที่ compile-time เสมอโดยอัตโนมัติ** — ในความเป็นจริง
   `constexpr` แค่ "อนุญาต" ให้คำนวณที่ compile-time ได้เท่านั้น ถ้าเรียกด้วยค่า runtime มันก็
   ทำงานเป็นฟังก์ชันธรรมดาที่รันจริงตอน runtime (ตรงข้ามกับ `consteval` ที่บังคับเสมอ) ถ้า
   ต้องการยืนยันว่าคำนวณที่ compile-time จริง ต้องพิสูจน์ด้วย `static_assert`/`constexpr`
   variable หรืออ่าน assembly (หัวข้อ 71.5-71.6)
4. **ใส่ side effect (เช่น `std::cout`, แก้ไข global variable) ใน constexpr function แล้ว
   คาดหวังให้มันทำงานที่ compile-time ได้** — `constexpr` function ต้องเป็น "pure-ish" (ไม่มี
   side effect ที่มองเห็นได้จากภายนอก) เพราะแนวคิดของ compile-time evaluation ไม่มีความหมายกับ
   การพิมพ์ข้อความออกหน้าจอหรือแก้ไขสถานะภายนอกฟังก์ชัน
5. **สับสนระหว่าง `constexpr` กับ `consteval`** — ใช้ `consteval` ทั้งที่จริงๆ ต้องการความ
   ยืดหยุ่นให้เรียกด้วยค่า runtime ได้ด้วย (ควรใช้ `constexpr` แทน) หรือใช้ `constexpr` ทั้งที่
   ต้องการการรับประกัน 100% ว่าต้องเป็น compile-time เท่านั้น (ควรใช้ `consteval` แทน)
6. **ลืมว่า `consteval` เป็นฟีเจอร์ของ C++20 เท่านั้น** — ถ้าโค้ดเบสยังตั้ง `-std=c++17` หรือ
   ต่ำกว่า การใช้ `consteval` จะทำให้ compile ไม่ผ่านทันที ต้องตรวจสอบให้แน่ใจว่า build system
   ตั้ง standard version ที่รองรับก่อนใช้งาน
7. **ใช้ recursive constexpr function ที่มี recursion ลึกเกินไปโดยไม่ตรวจสอบ** — compiler มี
   ขีดจำกัดจำนวนขั้นของการประเมินผลที่ compile-time (`-fconstexpr-depth` ใน GCC ค่าเริ่มต้นมัก
   อยู่ที่ 512 หรือมากกว่า แล้วแต่เวอร์ชัน) ถ้าฟังก์ชัน recursive แบบ `factorial` หรือ
   `fibonacci` ถูกเรียกด้วยค่าที่ใหญ่เกินไปจนเกินขีดจำกัดนี้ จะเกิด compile error แจ้งว่า
   "constexpr evaluation depth exceeds maximum" ควรใช้เวอร์ชันที่เขียนด้วย loop (สไตล์ C++14
   ขึ้นไป) แทนถ้าค่าที่ต้องคำนวณอาจมีขนาดใหญ่ เพราะ loop ไม่กินขีดจำกัดความลึกของ recursion

---

## แบบฝึกหัดท้ายบท

1. เขียน `constexpr bool isPrime(int n)` ที่ตรวจสอบว่า `n` เป็นจำนวนเฉพาะหรือไม่ แล้วเขียน
   `constexpr` function อีกตัวที่คืน `std::array<int, 10>` ของจำนวนเฉพาะ 10 ตัวแรก ยืนยันด้วย
   `static_assert` ว่าตัวที่ 10 คือ 29
2. แปลงฟังก์ชันต่อไปนี้ (ที่ยังไม่ใช่ `constexpr`) ให้กลายเป็น `constexpr` function ที่คำนวณได้
   ที่ compile-time โดยไม่เปลี่ยนพฤติกรรมตอนเรียกด้วยค่า runtime:
   ```cpp
   int power(int base, int exponent) {
       int result = 1;
       for (int i = 0; i < exponent; ++i) {
           result *= base;
       }
       return result;
   }
   ```
3. เขียน `consteval` function ที่คำนวณ checksum อย่างง่ายจากตัวเลขสามตัว (เช่นบวกกันแล้ว mod
   256) แล้วพิสูจน์ด้วยการ uncomment โค้ดที่พยายามเรียกมันด้วยตัวแปร `volatile` ว่าเกิด compile
   error ขึ้นจริง พร้อมอธิบายข้อความ error ที่ได้
4. ใช้ `-S` คอมไพล์โปรแกรมที่มีทั้งฟังก์ชัน `constexpr` ที่เรียกด้วยค่าคงที่ และฟังก์ชันเดียวกัน
   ที่เรียกด้วยค่า `volatile` (บังคับ runtime) แล้วเปิดไฟล์ `.s` ดูด้วยตัวเอง หาบรรทัดที่แสดงว่า
   ฝั่งไหนถูกคำนวณที่ compile-time (ไม่มี loop คูณ/บวกเหลืออยู่) และฝั่งไหนคำนวณตอน runtime
5. เขียน `constexpr int gcd(int a, int b)` แบบ recursive (Euclidean algorithm) พร้อม
   `consteval` เวอร์ชันที่บังคับ compile-time เสมอ แล้วทดสอบทั้งสองแบบด้วย `static_assert` และ
   ด้วยค่า runtime (เฉพาะเวอร์ชัน `constexpr`)
6. อธิบายด้วยคำพูดตัวเองว่าทำไม `constexpr` function ถึง "ห้ามมี side effect" และยกตัวอย่างโค้ด
   ที่พยายามใส่ `std::cout` เข้าไปใน `constexpr` function แล้วดูว่า compiler แจ้ง error อย่างไร
   เมื่อพยายามเรียกมันในบริบทที่ต้องการ compile-time constant (เช่น `static_assert` หรือ
   `constexpr` variable)

### แนวทางเฉลยข้อ 1

```cpp
// ex1_prime_table.cpp
#include <array>
#include <iostream>

constexpr bool isPrime(int n) {
    if (n < 2) return false;
    for (int i = 2; i * i <= n; ++i) {
        if (n % i == 0) return false;
    }
    return true;
}

constexpr std::array<int, 10> firstTenPrimes() {
    std::array<int, 10> primes{};
    int count = 0;
    int candidate = 2;
    while (count < 10) {
        if (isPrime(candidate)) {
            primes[static_cast<std::size_t>(count)] = candidate;
            ++count;
        }
        ++candidate;
    }
    return primes;
}

int main() {
    static_assert(isPrime(2), "2 เป็นจำนวนเฉพาะ");
    static_assert(isPrime(17), "17 เป็นจำนวนเฉพาะ");
    static_assert(!isPrime(18), "18 ไม่ใช่จำนวนเฉพาะ");
    static_assert(!isPrime(1), "1 ไม่ใช่จำนวนเฉพาะ");

    constexpr auto primes = firstTenPrimes();
    static_assert(primes[9] == 29, "จำนวนเฉพาะตัวที่ 10 ต้องเป็น 29");

    std::cout << "จำนวนเฉพาะ 10 ตัวแรก: ";
    for (int p : primes) std::cout << p << ' ';
    std::cout << '\n';

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 ex1_prime_table.cpp -o ex1
./ex1
```

```
จำนวนเฉพาะ 10 ตัวแรก: 2 3 5 7 11 13 17 19 23 29
```

จุดสำคัญของเฉลยนี้คือ `isPrime` ถูกเรียกซ้ำหลายครั้งภายใน `firstTenPrimes` (ทุกครั้งที่ตรวจสอบ
`candidate` ใหม่) และทั้งหมดเกิดขึ้นที่ compile-time เพราะทั้งสองฟังก์ชันเป็น `constexpr` และ
`primes` ถูกประกาศเป็น `constexpr auto` — ถ้าลองลบ `constexpr` ออกจากการประกาศ `primes` (เหลือ
แค่ `auto primes = firstTenPrimes();`) โปรแกรมจะยังทำงานถูกต้องเหมือนเดิม แต่ `static_assert`
ที่ตรวจสอบ `primes[9] == 29` จะ **compile ไม่ผ่านทันที** เพราะ `primes` ไม่ใช่ compile-time
constant อีกต่อไป (แม้ตัวฟังก์ชันจะยังเป็น `constexpr` ก็ตาม) — นี่คือตัวอย่างที่ดีที่ตอกย้ำว่า
`constexpr` เป็นแค่ "ความสามารถ" ไม่ใช่ "การรับประกัน" อัตโนมัติ

### แนวทางเฉลยข้อ 5

```cpp
// ex5_gcd.cpp
#include <iostream>

constexpr int gcd(int a, int b) {
    return (b == 0) ? a : gcd(b, a % b);
}

consteval int gcdStrict(int a, int b) {  // เวอร์ชันที่บังคับให้เป็น compile-time เท่านั้น
    return (b == 0) ? a : gcdStrict(b, a % b);
}

int main() {
    static_assert(gcd(48, 18) == 6, "gcd(48, 18) ต้องเท่ากับ 6");
    static_assert(gcd(17, 5) == 1, "17 กับ 5 เป็นจำนวนเฉพาะสัมพัทธ์กัน");

    constexpr int result = gcdStrict(1071, 462);
    static_assert(result == 21, "gcd(1071, 462) ต้องเท่ากับ 21");

    std::cout << "gcd(48, 18) = " << gcd(48, 18) << '\n';
    std::cout << "gcdStrict(1071, 462) = " << result << '\n';

    // gcd ยังเรียกด้วยค่า runtime ได้ตามปกติ เพราะเป็นแค่ constexpr ไม่ใช่ consteval
    volatile int x = 100, y = 75;
    std::cout << "gcd(runtime 100, 75) = " << gcd(x, y) << '\n';

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++20 ex5_gcd.cpp -o ex5
./ex5
```

```
gcd(48, 18) = 6
gcdStrict(1071, 462) = 21
gcd(runtime 100, 75) = 25
```

เฉลยนี้แสดงให้เห็นความแตกต่างระหว่าง `constexpr` กับ `consteval` ได้ชัดเจนที่สุด: `gcd`
(constexpr) เรียกได้ทั้งกับค่า compile-time (ใน `static_assert`) และค่า runtime (ผ่าน
`volatile`) ในฟังก์ชันเดียวกันโดยไม่ต้องเขียนสองเวอร์ชัน ในขณะที่ `gcdStrict` (consteval) ถูก
บังคับให้เรียกได้เฉพาะกับค่าที่ compiler รู้ตั้งแต่ compile-time เท่านั้น ถ้าลองเปลี่ยนบรรทัด
`gcdStrict(1071, 462)` ให้เป็น `gcdStrict(x, y)` (ตัวแปร `volatile`) จะได้ compile error ทันที
เหมือนที่แสดงในหัวข้อ 71.7

---

## สรุปท้ายบท

ใน Part นี้เราได้:

- แยกแยะความแตกต่างสำคัญระหว่าง **`const`** (ห้ามแก้ไขค่า แต่ไม่บังคับ compile-time) กับ
  **`constexpr`** (บังคับให้รู้ค่าตั้งแต่ compile-time เสมอ)
- เขียน **constexpr function** และเข้าใจข้อจำกัดที่เข้มงวดของ C++11 (แค่ `return` statement
  เดียว) เทียบกับการผ่อนคลายครั้งใหญ่ใน **C++14/17** (loop, ตัวแปร local, `if`/`else` ได้ตามใจ)
- เห็นว่า `constexpr` เป็นสะพานเชื่อมสู่ **template metaprogramming** ที่จะเจาะลึกเต็มรูปแบบใน
  Part 78 โดยทำให้เขียนโค้ดที่คำนวณค่าที่ compile-time ได้อ่านง่ายกว่าเทคนิคเก่ามาก
- พิสูจน์การคำนวณที่ compile-time ได้อย่างแน่ชัดทั้งด้วย **`static_assert`**/`constexpr`
  variable และด้วยการอ่าน **assembly output** โดยตรง
- ใช้ **`consteval`** (C++20) เพื่อบังคับให้ฟังก์ชันหนึ่งต้องถูกประเมินผลที่ compile-time เสมอ
  ไม่มีข้อยกเว้น และเปรียบเทียบกับความยืดหยุ่นของ `constexpr` ได้อย่างชัดเจน
- สร้างตัวอย่างจริงที่ใช้งานได้: ตาราง **factorial/fibonacci/จำนวนเฉพาะ** ที่คำนวณเสร็จสมบูรณ์
  ตั้งแต่ compile-time โดยไม่มีการคำนวณเหลืออยู่ตอนโปรแกรมรันจริงเลย

Compile-time programming ที่เรียนใน Part นี้เป็นหนึ่งในเสาหลักที่ทำให้ C++ ยังคงเป็นภาษาที่เร็ว
ที่สุดภาษาหนึ่งของโลก แม้จะมีนามธรรม (abstraction) ระดับสูงให้ใช้งานมากมายก็ตาม เพราะนามธรรม
เหล่านั้นจำนวนมาก "หายไป" ตั้งแต่ตอน compile ไม่เหลือค่าใช้จ่ายด้านประสิทธิภาพให้เห็นตอน runtime
เลยแม้แต่น้อย ใน **Part 72** เราจะไปดูฟีเจอร์อื่นๆ ของ **C++14 และ C++17** ที่เพิ่มเข้ามาเสริม
รากฐานของ C++11 ให้แข็งแกร่งยิ่งขึ้น ตั้งแต่ **structured bindings**, **`if constexpr`**
(ส่วนขยายของ `constexpr` ที่เพิ่งเรียนไปที่ใช้ใน generic code), ไปจนถึง **`std::optional`**,
**`std::variant`**, และ **`std::any`**

**ต่อไป:** [Part 72 — ฟีเจอร์ C++14/17](./part-072-cpp14-17-features.md)
