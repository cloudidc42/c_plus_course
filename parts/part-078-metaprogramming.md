# Part 78: Metaprogramming ด้วย Template (Step 617–624)

> Module F — Modern C++ (C++11 ถึง C++23) | Part 78 จาก 125
> Step ที่ครอบคลุมใน Part นี้: Step 617–624
> Part ก่อนหน้า: [Part 77 — ภาพรวม C++23](./part-077-cpp23-overview.md) | Part ถัดไป: [Part 79 — CRTP และ Template Pattern ขั้นสูง](./part-079-crtp-patterns.md)

## เป้าหมายการเรียนรู้ (Learning Objectives)

หลังจบ Part นี้ ผู้เรียนจะสามารถ:

1. อธิบายได้ว่า **Template Metaprogramming (TMP)** คืออะไร และทำไมมันคือการ "เขียนโปรแกรมที่รันตอน
   คอมไพล์" ไม่ใช่ตอนรันจริง
2. อธิบายหลักการของ **SFINAE (Substitution Failure Is Not An Error)** และบอกได้ว่าทำไมมันถึงเป็น
   กลไกสำคัญที่ทำให้ Template ใน C++ ยุคก่อน C++20 "เลือก overload" ได้ตามชนิดข้อมูล
3. ใช้ `std::enable_if` เขียนฟังก์ชันที่มีพฤติกรรมต่างกันตามชนิดข้อมูล (Compile-Time Dispatch) ได้เอง
4. ใช้ Type Traits ที่สำคัญใน `<type_traits>` เช่น `std::is_integral`, `std::is_same`,
   `std::remove_reference`, `std::conditional` ได้อย่างถูกต้อง
5. เขียน **Detection Idiom** (ตรวจสอบว่า type หนึ่งมี member function หรือไม่ ที่ compile time)
   ด้วย SFINAE ได้
6. เปรียบเทียบ SFINAE แบบเก่ากับ Concept (จาก Part 73) และอธิบายได้ว่าทำไมภาษาถึงพัฒนาไปทาง
   Concept
7. เขียน Compile-Time Recursion ด้วย Template แบบดั้งเดิม (เช่น Fibonacci) และเปรียบเทียบกับ
   `constexpr` function จาก Part 71 เพื่อเห็นวิวัฒนาการของภาษา C++ ตลอด 3 ทศวรรษ
8. ระบุข้อผิดพลาดที่พบบ่อยที่สุดของ Metaprogramming แบบเก่า และรู้ว่าเมื่อไหร่ควรใช้เทคนิคไหน

---

## 78.1 Template Metaprogramming คืออะไร (Step 617)

**Template Metaprogramming (TMP)** คือเทคนิคการใช้ Template ของ C++ เป็น "ภาษาโปรแกรมย่อย" ที่ทำงาน
ที่ **Compile Time** แทนที่จะทำงานที่ **Runtime** แบบโค้ดปกติ

ฟังดูแปลก แต่ระบบ Template ของ C++ ถูกค้นพบ (โดยบังเอิญ!) ว่า **Turing-complete** ตั้งแต่ยุค C++98 —
Erwin Unruh สาธิตให้เห็นในปี 1994 ว่าเขาเขียนโปรแกรมที่คำนวณเลขจำนวนเฉพาะได้ทั้งหมดโดยใช้แค่กลไก
การ instantiate Template โดยไม่ต้องรันโปรแกรมเลยสักบรรทัด — Error message ตอนคอมไพล์คือ "ผลลัพธ์"
ของโปรแกรมนั้น!

นี่คือแนวคิดสำคัญที่ต้องเข้าใจก่อน: เวลาคอมไพเลอร์เจอ Template เช่น `Factorial<5>` มันจะไม่ได้แค่
"สร้างโค้ด" เฉยๆ แต่มันจะ **ประเมินผล (evaluate)** โครงสร้างของ Template นั้นซ้ำไปซ้ำมาจนกว่าจะได้
คำตอบสุดท้าย เหมือนการรันโปรแกรมทั่วไป เพียงแต่ "ตัวแปร" ในที่นี้คือชนิดข้อมูล (Type) และค่าคงที่
(Compile-Time Constant) แทนที่จะเป็นตัวแปรใน Memory

ลองดูตัวอย่างคลาสสิกที่สุด: การคำนวณแฟกทอเรียลด้วย Template

```cpp
#include <cstdio>

template <unsigned int N>
struct Factorial {
    static constexpr unsigned long long value = N * Factorial<N - 1>::value;
};

template <>
struct Factorial<0> {
    static constexpr unsigned long long value = 1;
};

int main() {
    printf("5! = %llu\n", Factorial<5>::value);
    printf("10! = %llu\n", Factorial<10>::value);
    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++20 factorial.cpp -o factorial
./factorial
```

ผลลัพธ์:

```
5! = 120
10! = 3628800
```

**เกิดอะไรขึ้นเบื้องหลัง?** เมื่อคอมไพเลอร์เจอ `Factorial<5>::value` มันจะต้อง instantiate
`Factorial<5>` ซึ่งต้องใช้ `Factorial<4>::value` ซึ่งต้อง instantiate `Factorial<4>` ต่อไปเรื่อยๆ
จนถึง `Factorial<0>` ที่เรา **Specialize** ไว้เป็นกรณีฐาน (Base Case) — เหมือนฟังก์ชัน recursive
ทุกประการ เพียงแต่ทุกอย่างเกิดขึ้นตอนคอมไพล์ ไม่ใช่ตอนรัน ค่า `Factorial<10>::value` จึงเป็น
**ค่าคงที่** ที่ถูกฝังลงใน Binary ไว้แล้ว โปรแกรมไม่ต้องคำนวณอะไรเลยตอนรันจริง

โครงสร้างนี้มีองค์ประกอบสำคัญ 2 ส่วนที่ทุก Template Metaprogram แบบเก่าใช้ร่วมกัน:

| องค์ประกอบ | หน้าที่ | เทียบกับโปรแกรมปกติ |
|---|---|---|
| Primary Template (`template <unsigned int N> struct Factorial`) | นิยาม "กฎการคำนวณ" ทั่วไป | เหมือน `if` ในฟังก์ชัน recursive |
| Full/Partial Specialization (`Factorial<0>`) | กรณีฐานที่หยุด recursion | เหมือน `return` ตอน base case |
| `static constexpr` member | เก็บ "ผลลัพธ์" ของการคำนวณ | เหมือนค่า return ของฟังก์ชัน |

**ทำไมต้องเรียนเทคนิคนี้ในปี 2026 ทั้งที่มี `constexpr` function ที่ทำสิ่งเดียวกันได้ง่ายกว่ามาก
(จะเห็นใน 78.7–78.8)?** เพราะโค้ดจริงจำนวนมหาศาลในไลบรารีเก่า (Boost, STL implementation เอง,
โปรเจกต์ enterprise ที่มีอายุ 10-20 ปี) เขียนด้วยเทคนิคนี้ และวิศวกร C++ มืออาชีพต้อง **อ่านโค้ดแบบ
นี้ให้ออก** แม้จะไม่เขียนมันเองในโปรเจกต์ใหม่แล้วก็ตาม

---

## 78.2 SFINAE คืออะไร — หลักการพื้นฐาน (Step 618)

**SFINAE** ย่อมาจาก **S**ubstitution **F**ailure **I**s **N**ot **A**n **E**rror

แปลตรงตัว: "ถ้าการแทนที่ (substitution) ชนิดข้อมูลเข้าไปใน Template Signature แล้วทำให้เกิดโค้ดที่
ผิดหลักไวยากรณ์ นั่น**ไม่ใช่ Compile Error** แต่คอมไพเลอร์จะ**ตัด overload นั้นออกจากตัวเลือกเงียบๆ**
แล้วไปลองตัวถัดไปแทน"

นี่คือกลไกที่ทำให้ Function Overload Resolution ของ C++ "ฉลาด" กว่าที่คนทั่วไปคิด ลองดูตัวอย่างง่ายๆ
ก่อนที่จะเข้าใจว่าทำไมมันสำคัญ:

```cpp
#include <iostream>

// overload สำหรับ type ที่มี member function .size()
template <typename T>
auto get_length(const T& container) -> decltype(container.size()) {
    return container.size();
}

// overload สำหรับ raw pointer (ไม่มี .size())
template <typename T>
size_t get_length(const T* ptr) {
    (void)ptr;
    return 1;  // pointer เดี่ยวๆ ถือว่ามีความยาว 1
}

int main() {
    std::string s = "hello";
    int arr = 42;

    std::cout << get_length(s) << "\n";    // เรียก overload แรก เพราะ string มี .size()
    std::cout << get_length(&arr) << "\n"; // เรียก overload ที่สอง เพราะ int* ไม่มี .size()
    return 0;
}
```

เวลาคอมไพเลอร์เจอ `get_length(&arr)` มันจะลองแทนที่ `T = int` เข้าไปใน overload แรกก่อน (ตามลำดับ
การพิจารณา) ได้ `decltype((&arr)->size())` ซึ่ง**ผิดหลักไวยากรณ์**เพราะ `int*` ไม่มี member ชื่อ
`size()` — ถ้าเป็น error ธรรมดา โปรแกรมทั้งไฟล์จะคอมไพล์ไม่ผ่านทันที แต่เพราะกฎ SFINAE คอมไพเลอร์แค่
**ไม่นับ overload นี้เป็นตัวเลือก** แล้วไปเจอ overload ที่สองที่รับ `T*` ได้พอดี จึงเลือกตัวนั้นแทน

**กฎสำคัญที่ต้องจำ**: SFINAE ใช้ได้เฉพาะตอนที่ความผิดพลาดเกิดขึ้นใน
**"Immediate Context"** ของการแทนที่ Template Parameter เท่านั้น (เช่น ใน Return Type, ใน Template
Parameter List, ใน Function Parameter List) ถ้า Error เกิด**ลึกเข้าไปในเนื้อ Function Body**
หลังจากที่ Overload ถูกเลือกไปแล้ว นั่นจะเป็น **Compile Error จริง** ไม่ใช่ SFINAE

นี่คือความแตกต่างที่มือใหม่สับสนบ่อยที่สุด: SFINAE คือกลไก "คัดเลือกก่อนเข้าแข่ง" ไม่ใช่ "ตรวจสอบ
หลังเข้าแข่งแล้ว"

---

## 78.3 SFINAE ตัวอย่างจริง: เลือก Overload ตาม Type ด้วย `std::enable_if` (Step 619)

เทคนิคที่ใช้ SFINAE บ่อยที่สุดในโค้ดจริงคือการใช้ `std::enable_if` (จาก `<type_traits>`) เพื่อ
"เปิด/ปิด" overload ตามเงื่อนไขของชนิดข้อมูลอย่างชัดเจน แทนที่จะพึ่งพา `decltype` ที่ซับซ้อนและอ่านยาก

`std::enable_if<Condition, T>` ทำงานดังนี้:

- ถ้า `Condition` เป็น `true` → `std::enable_if<Condition, T>::type` จะมีอยู่จริง และเท่ากับ `T`
- ถ้า `Condition` เป็น `false` → `std::enable_if<Condition, T>::type` **ไม่มีอยู่จริง** (ไม่ได้
  ประกาศ member `type` ไว้เลย) ทำให้เกิด Substitution Failure → SFINAE ตัด overload นั้นทิ้งไป

```cpp
#include <iostream>
#include <type_traits>

template <typename T>
typename std::enable_if<std::is_integral<T>::value, void>::type
describe(T value) {
    std::cout << value << " เป็นชนิด integral (จำนวนเต็ม)\n";
}

template <typename T>
typename std::enable_if<std::is_floating_point<T>::value, void>::type
describe(T value) {
    std::cout << value << " เป็นชนิด floating-point (ทศนิยม)\n";
}

int main() {
    describe(42);
    describe(3.14);
    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++20 describe.cpp -o describe && ./describe
```

ผลลัพธ์:

```
42 เป็นชนิด integral (จำนวนเต็ม)
3.14 เป็นชนิด floating-point (ทศนิยม)
```

**อ่านโค้ดทีละส่วน:**

- `std::is_integral<T>::value` คือค่า `bool` (compile-time constant) ที่บอกว่า `T` เป็นจำนวนเต็ม
  หรือไม่ (`int`, `long`, `char`, `bool`, ฯลฯ)
- `std::enable_if<std::is_integral<T>::value, void>::type` แปลว่า "ถ้า `T` เป็น integral ให้
  ประกาศ `type` เป็น `void`" — เราใช้ `void` เพราะฟังก์ชันนี้ไม่คืนค่าอะไร
- `typename` ข้างหน้าจำเป็นต้องมีเสมอ เพราะ `std::enable_if<...>::type` เป็น **Dependent Type**
  (ชนิดที่ขึ้นอยู่กับ Template Parameter `T`) คอมไพเลอร์ไม่รู้ล่วงหน้าว่า `::type` คือชนิดข้อมูล
  หรือค่าคงที่ ต้องบอกมันชัดๆ ด้วย `typename` (ดูรายละเอียดเพิ่มเติมใน Common Pitfalls)

เมื่อเรียก `describe(42)` (int) คอมไพเลอร์ลองแทนที่ `T = int` เข้าไปทั้งสอง overload:

1. Overload แรก: `enable_if<is_integral<int>::value=true, void>::type` → ได้ `void` → **ใช้งานได้**
2. Overload ที่สอง: `enable_if<is_floating_point<int>::value=false, void>::type` → **ไม่มี `::type`
   อยู่จริง** → Substitution Failure → SFINAE ตัด overload นี้ทิ้งไปเงียบๆ

ผลคือมีเพียง overload เดียวที่เหลือรอด คอมไพเลอร์จึงเรียกมันโดยไม่มีปัญหาเรื่อง Ambiguous

### Pattern ที่ 2: enable_if เป็น Template Parameter เพิ่มเติม (นิยมกว่าในโค้ดจริง)

อีกวิธีที่พบบ่อยกว่าคือใส่ `enable_if` เป็นพารามิเตอร์ Template ตัวที่สองแบบ default value
เพื่อไม่ให้ไปรบกวน Return Type ของฟังก์ชัน (อ่านง่ายกว่าเมื่อ Return Type ซับซ้อน):

```cpp
#include <iostream>
#include <type_traits>

template <typename T,
          typename = typename std::enable_if<std::is_integral<T>::value>::type>
T double_value(T x) {
    return x * 2;
}

int main() {
    std::cout << double_value(21) << "\n";  // OK: int เป็น integral
    // std::cout << double_value(2.5) << "\n";  // ถ้าเปิดบรรทัดนี้: compile error
    //                                            เพราะไม่มี overload ไหนรับ double ได้เลย
    return 0;
}
```

หมายเหตุ: `std::enable_if<Condition>` (ไม่ระบุ argument ตัวที่สอง) จะ default เป็น `void` ให้
อัตโนมัติ ซึ่งเพียงพอเพราะในที่นี้เราแค่ต้องการใช้มันเป็น "ประตูเปิด-ปิด" ไม่ได้ต้องการใช้ผลลัพธ์
`::type` ไปทำอะไรต่อ

### เทคนิคเก่ากว่า `enable_if`: Tag Dispatch

ก่อนที่ `std::enable_if` จะถูกใช้อย่างแพร่หลาย (และปัจจุบันก็ยังพบในโค้ดเก่าจำนวนมาก) มีอีกเทคนิค
หนึ่งที่ทำ Compile-Time Dispatch ได้เช่นกัน เรียกว่า **Tag Dispatch** — แทนที่จะ "ปิด" overload
ที่ไม่ตรงเงื่อนไขด้วย SFINAE เราจะส่ง **object เล็กๆ ที่บอกชนิด** (เช่น `std::true_type` /
`std::false_type`) เข้าไปเป็น parameter เพิ่มเติม แล้วให้ Overload Resolution ปกติเลือก overload
ที่ตรงกับ "ป้ายกำกับ" (tag) นั้น:

```cpp
#include <iostream>
#include <type_traits>

template <typename T>
void advance_impl(T& it, int n, std::true_type /* is_pointer */) {
    it += n;
    std::cout << "ใช้ O(1) advance (pointer / random access)\n";
}

template <typename T>
void advance_impl(T& it, int n, std::false_type /* is_pointer */) {
    for (int i = 0; i < n; ++i) {
        ++it;
    }
    std::cout << "ใช้ O(n) advance (sequential)\n";
}

template <typename T>
void my_advance(T& it, int n) {
    advance_impl(it, n, std::is_pointer<T>{});
}

int main() {
    int arr[5] = {1, 2, 3, 4, 5};
    int* p = arr;
    my_advance(p, 3);
    std::cout << *p << "\n";
    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++20 tag_dispatch.cpp -o tag_dispatch && ./tag_dispatch
```

ผลลัพธ์:

```
ใช้ O(1) advance (pointer / random access)
4
```

`std::is_pointer<T>{}` สร้าง object ชั่วคราวชนิด `std::true_type` หรือ `std::false_type` (ทั้งสอง
เป็น "Tag Type" ที่ไม่มีข้อมูลข้างในเลย มีไว้แค่บอกชนิด) แล้ว Overload Resolution ปกติ (ไม่ใช่
SFINAE) จะเลือก `advance_impl` เวอร์ชันที่รับ parameter ตัวที่สามตรงกับชนิดของ tag นั้น เทคนิคนี้คือ
รากฐานของฟังก์ชัน `std::advance` และ `std::distance` ใน STL จริงที่เลือกวิธีเลื่อน Iterator ต่างกัน
ตามประเภทของ Iterator (Random Access vs Forward/Bidirectional) มาตั้งแต่ C++98

**เทียบ Tag Dispatch กับ `enable_if`:** Tag Dispatch อ่านง่ายกว่าเล็กน้อยเพราะใช้ Overload
Resolution ปกติ ไม่ต้องยุ่งกับ SFINAE โดยตรง แต่มีข้อจำกัดคือต้องมี **object ที่ส่งได้จริง**
เป็นตัวแทนของเงื่อนไข (ซึ่งใช้ได้ดีกับ `true_type`/`false_type` ที่มีแค่สองค่า) ในขณะที่ `enable_if`
ยืดหยุ่นกว่าเพราะรับเงื่อนไขที่ซับซ้อนเป็นนิพจน์ `bool` ใดๆ ก็ได้โดยตรง ทั้งสองเทคนิคถูกแทนที่ด้วย
`if constexpr` (Part 72) และ Concept (Part 73, 78.6) เกือบทั้งหมดในโค้ดสมัยใหม่

---

## 78.4 Type Traits ที่สำคัญใน `<type_traits>` (Step 620)

`<type_traits>` คือ Header ที่รวบรวม **Template ที่ตรวจสอบ/แปลงข้อมูลเกี่ยวกับชนิดข้อมูล** ที่
Compile Time ทั้งหมด เป็นเครื่องมือพื้นฐานที่ SFINAE, `enable_if`, และแม้แต่ Concept (Part 73)
ล้วนพึ่งพาอยู่เบื้องหลัง

### กลุ่มที่ 1: Trait ที่ตรวจสอบ (Predicate Traits) — คืนค่าเป็น `::value` แบบ `bool`

```cpp
#include <iostream>
#include <type_traits>

int main() {
    std::cout << std::boolalpha;

    std::cout << "is_same<int,int> = " << std::is_same<int, int>::value << "\n";
    std::cout << "is_same<int,unsigned int> = "
              << std::is_same<int, unsigned int>::value << "\n";

    using T1 = std::remove_reference<int&>::type;
    static_assert(std::is_same<T1, int>::value, "T1 ต้องเป็น int");
    std::cout << "remove_reference<int&>::type คือ int แล้ว: OK\n";

    using Choice = std::conditional<true, int, double>::type;
    static_assert(std::is_same<Choice, int>::value, "Choice ต้องเป็น int");
    std::cout << "conditional<true, int, double>::type คือ int แล้ว: OK\n";

    return 0;
}
```

| Trait | ใช้ทำอะไร | ตัวอย่าง |
|---|---|---|
| `std::is_integral<T>` | ตรวจว่า `T` เป็นจำนวนเต็มหรือไม่ | `is_integral<int>::value == true` |
| `std::is_floating_point<T>` | ตรวจว่า `T` เป็นทศนิยมหรือไม่ | `is_floating_point<double>::value == true` |
| `std::is_pointer<T>` | ตรวจว่า `T` เป็น pointer หรือไม่ | `is_pointer<int*>::value == true` |
| `std::is_same<T, U>` | ตรวจว่า `T` กับ `U` เป็นชนิดเดียวกันเป๊ะหรือไม่ (รวม const/reference) | `is_same<int, int&>::value == false` |
| `std::is_base_of<Base, Derived>` | ตรวจว่า `Derived` สืบทอดจาก `Base` หรือไม่ | ใช้บ่อยกับ CRTP/Design Pattern |
| `std::is_class<T>` | ตรวจว่า `T` เป็น class/struct หรือไม่ | แยกจาก primitive type |

### กลุ่มที่ 2: Trait ที่แปลงชนิด (Transformation Traits) — คืนค่าเป็น `::type`

| Trait | ใช้ทำอะไร | ตัวอย่าง |
|---|---|---|
| `std::remove_reference<T>` | ลบ `&`/`&&` ออกจาก `T` | `remove_reference<int&>::type == int` |
| `std::remove_const<T>` | ลบ `const` ออกจาก `T` | `remove_const<const int>::type == int` |
| `std::add_pointer<T>` | เติม `*` ให้ `T` | `add_pointer<int>::type == int*` |
| `std::decay<T>` | ทำ `T` ให้เหมือนตอนถูก pass by value (ลบ reference/const/array→pointer) | ใช้บ่อยใน generic code |
| `std::conditional<B, T, F>` | เลือก `T` ถ้า `B` เป็นจริง ไม่งั้นเลือก `F` — เหมือน `? :` แต่ทำงานกับ**ชนิดข้อมูล**แทนค่า | `conditional<true, int, double>::type == int` |

**เคล็ดลับสำคัญ**: ตั้งแต่ C++14 เป็นต้นมา แทบทุก Trait มี **helper alias** ที่ช่วยตัด
`typename ...::type` และ `...::value` ออกได้ ทำให้อ่านง่ายขึ้นมาก:

```cpp
#include <type_traits>

template <typename T>
void demo() {
    // แบบเก่า (C++11)
    using OldStyle = typename std::remove_reference<T>::type;
    constexpr bool old_check = std::is_integral<T>::value;

    // แบบใหม่ (C++14+) — สั้นและอ่านง่ายกว่ามาก
    using NewStyle = std::remove_reference_t<T>;
    constexpr bool new_check = std::is_integral_v<T>;

    static_assert(std::is_same_v<OldStyle, NewStyle>, "ทั้งสองแบบต้องได้ชนิดเดียวกัน");
    static_assert(old_check == new_check, "ทั้งสองแบบต้องได้ค่าเดียวกัน");
}

int main() {
    demo<int&>();
    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++20 alias_vs_full.cpp -o alias_vs_full && ./alias_vs_full
```

โปรแกรมนี้ไม่พิมพ์อะไรออกมาเลย (คอมไพล์ผ่านและรันจบเงียบๆ) เพราะจุดประสงค์คือให้ `static_assert`
ยืนยันที่ compile time ว่ารูปแบบเต็ม (`typename ...::type`, `...::value`) กับรูปแบบย่อ (`_t`, `_v`)
ให้ผลลัพธ์เหมือนกันทุกประการ — ถ้าไม่ตรงกัน โปรแกรมจะคอมไพล์ไม่ผ่านทันที

โค้ดจริงในปี 2026 แทบจะใช้แต่รูปแบบ `_v` และ `_t` เท่านั้น แต่บทความ/ไลบรารีเก่าจำนวนมากยังใช้
รูปแบบเต็ม ผู้เรียนจึงต้องอ่านทั้งสองแบบให้ออก

### การรวม Type Traits หลายตัวเข้าด้วยกัน

ในโค้ดจริง เรามักต้องรวมเงื่อนไขหลายอย่างเข้าด้วยกันด้วย `&&`/`||` ธรรมดา เพราะ `::value` ของทุก
Trait คือแค่ `constexpr bool` ธรรมดา:

```cpp
#include <iostream>
#include <type_traits>

template <typename T>
typename std::enable_if<std::is_arithmetic<T>::value, T>::type
clamp_positive(T value) {
    return value < T{0} ? T{0} : value;
}

int main() {
    std::cout << clamp_positive(-5) << "\n";   // int: ติดลบ -> ปัดเป็น 0
    std::cout << clamp_positive(3.5) << "\n";  // double: บวกอยู่แล้ว -> คงค่าเดิม
    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++20 clamp_positive.cpp -o clamp_positive && ./clamp_positive
```

ผลลัพธ์:

```
0
3.5
```

`std::is_arithmetic<T>` ครอบคลุมทั้ง `is_integral` และ `is_floating_point` เข้าด้วยกัน (คือ
`is_integral<T>::value || is_floating_point<T>::value` โดยนิยาม) ทำให้ `clamp_positive` ใช้ได้กับ
ทั้งจำนวนเต็มและทศนิยมด้วย overload เดียว นี่คือตัวอย่างว่าทำไมการเลือก Trait ที่ "กว้างพอดี" กับ
ความต้องการ (ไม่แคบเกินไปจนต้องเขียนหลาย overload, ไม่กว้างเกินไปจนรับ type ที่ไม่ต้องการ) เป็น
ทักษะสำคัญของการออกแบบ Generic Code ที่ดี

---

## 78.5 Detection Idiom: ตรวจสอบว่า Type มี Member Function หรือไม่ (Step 621)

หนึ่งในการใช้งาน SFINAE ที่ทรงพลังที่สุดคือ **Detection Idiom** — เทคนิคตรวจสอบที่ Compile Time ว่า
"ชนิดข้อมูล `T` มี member function ชื่อนี้อยู่หรือไม่" โดยไม่ต้องรันโปรแกรมเลย นี่คือรากฐานของ
Library ระดับ production หลายตัว (เช่น การตรวจสอบว่า container รองรับ `.reserve()` หรือไม่ก่อน
เรียกมันใน `std::vector`)

```cpp
#include <iostream>
#include <string>
#include <type_traits>
#include <utility>

template <typename T, typename = void>
struct has_to_string : std::false_type {};

template <typename T>
struct has_to_string<T, std::void_t<decltype(std::declval<T>().to_string())>>
    : std::true_type {};

struct Foo {
    std::string to_string() const { return "Foo"; }
};

struct Bar {};

int main() {
    std::cout << std::boolalpha;
    std::cout << "has_to_string<Foo> = " << has_to_string<Foo>::value << "\n";
    std::cout << "has_to_string<Bar> = " << has_to_string<Bar>::value << "\n";
    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++20 detection.cpp -o detection && ./detection
```

ผลลัพธ์:

```
has_to_string<Foo> = true
has_to_string<Bar> = false
```

**แกะกลไกทีละส่วน — นี่คือหนึ่งใน pattern ที่อ่านยากที่สุดในภาษา C++ ยุคก่อน Concept:**

1. `template <typename T, typename = void> struct has_to_string : std::false_type {};`
   คือ **Primary Template** — กรณี default ที่บอกว่า "ไม่มี" (`false_type`) พารามิเตอร์ตัวที่สอง
   เป็น `void` โดย default เผื่อไว้สำหรับ specialization

2. `std::declval<T>()` คือฟังก์ชันมายากลที่ "สร้าง reference ปลอมๆ ของ `T`" โดยไม่ต้องมี object
   จริงเลย (ใช้ได้เฉพาะใน unevaluated context เช่น `decltype`, `sizeof` เท่านั้น — ห้ามเรียกจริง)
   มีประโยชน์มากตอนที่ `T` อาจจะไม่มี default constructor ก็ยังตรวจสอบ member function ได้

3. `decltype(std::declval<T>().to_string())` คือชนิดข้อมูลที่ `.to_string()` คืนกลับมา — ถ้า `T`
   ไม่มี member นี้ นิพจน์นี้จะ**ผิดหลักไวยากรณ์** (Substitution Failure)

4. `std::void_t<...>` เป็น Trait พิเศษที่รับ Type อะไรก็ได้กี่ตัวก็ได้ แล้วคืน `void` เสมอ (มีไว้
   เป็น "กับดัก SFINAE" — ถ้า Type ข้างในมันประเมินไม่ได้ ทั้งนิพจน์ก็จะประเมินไม่ได้ตามไปด้วย)

5. `struct has_to_string<T, std::void_t<...>> : std::true_type {}` คือ **Partial Specialization**
   ที่จะถูกเลือกใช้**ก็ต่อเมื่อ** `std::void_t<decltype(...)>` แทนที่แล้วได้ `void` จริง (นั่นคือ
   `.to_string()` มีอยู่จริง) ถ้าไม่มี SFINAE จะตัด specialization นี้ทิ้ง เหลือแค่ Primary Template
   (`false_type`) ให้ใช้แทน

Pattern นี้เรียกว่า **`void_t` Detection Idiom** ถูกเผยแพร่ครั้งแรกโดย Walter E. Brown ในปี 2014
และกลายเป็นมาตรฐานโดยพฤตินัยสำหรับตรวจสอบความสามารถของ Type ก่อนที่ Concept จะมาแทนที่ใน C++20

---

## 78.6 SFINAE vs Concepts: ทำไมภาษาถึงพัฒนาไปทาง Concept (Step 622)

ใน Part 73 เราได้เรียนรู้ **Concept** ของ C++20 ไปแล้ว ตอนนี้เรามาดูกันตรงๆ ว่า Concept แก้ปัญหา
อะไรของ SFINAE ได้บ้าง โดยใช้ตัวอย่าง `has_to_string` เดิมเทียบกัน

```cpp
#include <concepts>
#include <iostream>
#include <string>

template <typename T>
concept HasToString = requires(const T& t) {
    { t.to_string() } -> std::convertible_to<std::string>;
};

struct Foo {
    std::string to_string() const { return "Foo"; }
};

struct Bar {};

void print_info(const HasToString auto& x) {
    std::cout << "ค่า: " << x.to_string() << "\n";
}

int main() {
    Foo f;
    print_info(f);
    std::cout << std::boolalpha;
    std::cout << "HasToString<Foo> = " << HasToString<Foo> << "\n";
    std::cout << "HasToString<Bar> = " << HasToString<Bar> << "\n";
    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++20 concept_equiv.cpp -o concept_equiv && ./concept_equiv
```

ผลลัพธ์:

```
ค่า: Foo
HasToString<Foo> = true
HasToString<Bar> = false
```

โค้ด Concept ทำสิ่งเดียวกันกับ `void_t` Detection Idiom ทุกประการ แต่สั้นกว่า อ่านง่ายกว่า และที่
สำคัญที่สุดคือ **Error Message เมื่อใช้ผิดจะชัดเจนกว่ามาก** ลองเปรียบเทียบตารางสรุป:

| ประเด็น | SFINAE + `enable_if`/`void_t` (แบบเก่า) | Concept (C++20, Part 73) |
|---|---|---|
| Syntax | ซับซ้อน ต้องรู้ trick หลายชั้น (`void_t`, `declval`, partial specialization) | อ่านเป็นภาษาธรรมชาติ (`requires`) |
| Error Message | ยาวมาก มักพ่น Template Instantiation หลายสิบบรรทัด ชี้ไปที่ internal ของ library | สั้น กระชับ ชี้ตรงไปที่เงื่อนไขที่ไม่ผ่าน |
| การนำไปใช้ซ้ำ | ต้องเขียน trait struct แยกทุกครั้ง | ประกาศ `concept` ครั้งเดียว ใช้ซ้ำได้ทุกที่ |
| การรวมเงื่อนไข | ต้องเขียน `&&`/`||` ด้วย `::value` เอง อ่านยาก | ใช้ `&&`, `||` ตรงๆ ใน `requires` clause ได้เลย |
| รองรับใน compiler | ทุกเวอร์ชันตั้งแต่ C++98 (บางเทคนิคต้องมี C++11 ขึ้นไป) | C++20 ขึ้นไปเท่านั้น |
| Overload Resolution | อาศัย SFINAE (ตัด overload เงียบๆ) ซึ่งบางครั้งพฤติกรรมเข้าใจยาก | Concept เป็นส่วนหนึ่งของ Type System โดยตรง จัดลำดับ overload ตาม "ความเฉพาะเจาะจง" ได้ชัดเจนกว่า |

**สรุปสั้นๆ**: SFINAE คือ "ผลข้างเคียงที่ฉลาด" ของกฎภาษาที่ถูกค้นพบและนำมาใช้ประโยชน์ ส่วน Concept
คือ "ฟีเจอร์ที่ถูกออกแบบมาโดยเฉพาะ" เพื่อทำสิ่งที่ SFINAE เคยทำ แต่ทำได้ดีกว่าในทุกมิติ ถ้าเขียน
โปรเจกต์ใหม่ที่ใช้ C++20 ขึ้นไปได้ **ควรใช้ Concept เสมอ** แต่การเข้าใจ SFINAE ยังจำเป็นเพราะ:

1. โค้ด Legacy จำนวนมหาศาลยังใช้ SFINAE อยู่ (STL implementation เองก็ใช้ SFINAE เก็บไว้ภายใน)
2. โปรเจกต์ที่ต้อง compile ด้วย C++11/14/17 (Embedded, Enterprise เก่า) ยังต้องพึ่ง SFINAE
3. Concept เองก็ implement อยู่บนหลักการเดียวกับ SFINAE ในระดับ compiler เข้าใจ SFINAE จะช่วยให้
   เข้าใจว่า Concept "ทำงานยังไง" ลึกซึ้งขึ้น ไม่ใช่แค่ "ใช้ยังไง"

---

## 78.7 Compile-Time Recursion แบบดั้งเดิม: Fibonacci ด้วย Template (Step 623)

มาดูตัวอย่าง Template Metaprogramming แบบคลาสสิกอีกตัวที่ใช้กันมากในหนังสือและบทความยุค C++98-C++03
ก่อนที่ `constexpr` จะถือกำเนิดขึ้นใน C++11:

```cpp
#include <iostream>

// ---- แบบเก่า: compile-time recursion ผ่าน template ----
template <unsigned int N>
struct FibTmpl {
    static constexpr unsigned long long value =
        FibTmpl<N - 1>::value + FibTmpl<N - 2>::value;
};

template <>
struct FibTmpl<0> {
    static constexpr unsigned long long value = 0;
};

template <>
struct FibTmpl<1> {
    static constexpr unsigned long long value = 1;
};

int main() {
    std::cout << "Fib(20) ผ่าน template metaprogramming = " << FibTmpl<20>::value << "\n";
    return 0;
}
```

รูปแบบนี้เหมือน `Factorial` ใน 78.1 ทุกประการ: มี Primary Template ที่นิยามกฎ `Fib(n) = Fib(n-1) +
Fib(n-2)`, มี Full Specialization สองตัวเป็นกรณีฐาน (`Fib(0)=0`, `Fib(1)=1`)

**ข้อสังเกตสำคัญที่ต้องรู้**: การเขียนแบบนี้มี **ข้อจำกัดที่ร้ายแรง** สองอย่าง:

1. **รับเฉพาะค่าคงที่ที่รู้ตอนคอมไพล์เท่านั้น** — `FibTmpl<n>` โดยที่ `n` เป็นตัวแปรที่รู้ค่าตอน
   รันจริง (runtime) นั้น **เป็นไปไม่ได้เลย** เพราะ Template Parameter ต้องเป็นค่าคงที่ตอนคอมไพล์
2. **Compiler มี "Template Instantiation Depth" จำกัด** (โดย default บน GCC มักอยู่ที่ 900) ถ้าเรา
   ลองเรียก `FibTmpl<2000>::value` (ซึ่งใน implementation ตรงไปตรงมาแบบนี้จะ instantiate ต้นไม้
   ขนาดเลขชี้กำลังของ Template นับล้านตัว) คอมไพเลอร์จะพ่น error
   `template instantiation depth exceeds maximum` และคอมไพล์นานมากหรือไม่จบเลย นี่คือราคาที่ต้อง
   จ่ายของการ "คำนวณ" ด้วยระบบ Type ที่ไม่ได้ถูกออกแบบมาเพื่อการคำนวณโดยตรง

---

## 78.8 เทียบกับ `constexpr` Function: วิวัฒนาการของภาษา C++ (Step 624)

ใน **Part 71** เราได้เรียนรู้ `constexpr` function ไปแล้ว ทีนี้มาดูกันว่าปัญหาเดียวกัน (คำนวณ
Fibonacci ที่ compile time) แก้ด้วย `constexpr` function ได้ **ง่ายกว่ามากแค่ไหน**:

```cpp
#include <iostream>

// ---- แบบใหม่: constexpr function (Part 71) ----
constexpr unsigned long long fib_constexpr(unsigned int n) {
    if (n < 2) {
        return n;
    }
    unsigned long long a = 0;
    unsigned long long b = 1;
    for (unsigned int i = 2; i <= n; ++i) {
        unsigned long long c = a + b;
        a = b;
        b = c;
    }
    return b;
}

int main() {
    // เรียกที่ compile time (ผลลัพธ์ถูกฝังใน binary ตั้งแต่คอมไพล์)
    constexpr unsigned long long compile_time_result = fib_constexpr(20);
    std::cout << "Fib(20) ผ่าน constexpr (compile-time) = " << compile_time_result << "\n";

    // เรียกที่ runtime ด้วยค่าที่รู้ตอนรันจริงก็ได้เหมือนกัน! (จุดต่างสำคัญจาก Template)
    unsigned int n_from_user = 15;  // สมมติว่าอ่านมาจาก std::cin
    std::cout << "Fib(15) ผ่าน constexpr (runtime)      = " << fib_constexpr(n_from_user) << "\n";

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++20 fib_constexpr.cpp -o fib_constexpr && ./fib_constexpr
```

ผลลัพธ์:

```
Fib(20) ผ่าน constexpr (compile-time) = 6765
Fib(15) ผ่าน constexpr (runtime)      = 610
```

มาดูตารางเปรียบเทียบทั้งสองแนวทางโดยตรง เพื่อสรุปวิวัฒนาการของภาษา:

| ประเด็น | Template Recursion (C++98) | `constexpr` Function (C++11+, Part 71) |
|---|---|---|
| ไวยากรณ์ | ต้องเขียน `struct`, specialization, `::value` | เขียนเหมือนฟังก์ชันปกติทุกประการ |
| อ่าน/เข้าใจได้ง่าย | ยาก ต้องรู้กลไก Template instantiation | ง่าย เหมือนฟังก์ชันธรรมดาที่ทุกคนเขียนเป็น |
| ใช้ตัวแปร Loop ได้ไหม | ไม่ได้ ต้องใช้ recursion ผ่าน template เท่านั้น | ได้เต็มที่ (`for`, `while`, `if`) ตั้งแต่ C++14 |
| เรียกด้วยค่า runtime ได้ไหม | **ไม่ได้เลย** ต้องเป็นค่าคงที่ compile-time เท่านั้น | **ได้** — ฟังก์ชันเดียวใช้ได้ทั้ง compile-time และ runtime |
| ข้อจำกัด Instantiation Depth | มี (มักพังที่ n ~500-900 ขึ้นกับ compiler) | ไม่มีปัญหานี้ (แต่มี constexpr evaluation step limit ซึ่งสูงกว่ามาก และปรับได้ด้วย flag) |
| Error Message เมื่อผิดพลาด | ยาวมาก ซับซ้อน | สั้น ตรงประเด็น เหมือน error ของฟังก์ชันปกติ |

**นี่คือบทเรียนที่สำคัญที่สุดของ Part นี้**: Template Metaprogramming แบบดั้งเดิมถือกำเนิดขึ้นเพราะ
มันคือ**หนทางเดียว**ที่มีในยุค C++98 ที่จะทำให้เกิดการคำนวณที่ compile time ได้ แต่มันไม่ได้ถูก
**ออกแบบมาเพื่อสิ่งนี้โดยเฉพาะ** — มันคือ "การแฮ็กระบบ Type ให้ทำหน้าที่เป็นภาษาโปรแกรมมิ่ง" เมื่อ
C++11 นำ `constexpr` เข้ามา (และแข็งแกร่งขึ้นเรื่อยๆ ใน C++14/17/20) มันคือฟีเจอร์ที่ **ถูกออกแบบมา
โดยเฉพาะ** เพื่อการคำนวณที่ compile time พร้อม syntax ของฟังก์ชันปกติทุกประการ

แนวโน้มนี้เกิดซ้ำแล้วซ้ำเล่าในประวัติศาสตร์ C++: SFINAE → Concept (78.6), Template Recursion →
`constexpr` (78.8) — ภาษาพัฒนาไปในทิศทางที่ "เอาเทคนิคที่เคยต้องแฮ็กด้วย Template มาทำให้เป็น
ฟีเจอร์ระดับภาษาโดยตรง อ่านง่ายขึ้น ปลอดภัยขึ้น Error ชัดเจนขึ้น" ผู้เรียนที่เข้าใจทั้งสองยุคจะเข้าใจ
"ทำไม" Modern C++ ถึงถูกออกแบบมาแบบที่เป็นอยู่ ไม่ใช่แค่ "จำ syntax ใหม่" เฉยๆ

---

## ข้อผิดพลาดที่พบบ่อย (Common Pitfalls)

1. **ลืม `typename` หน้า Dependent Type แล้วงงว่าทำไม error** — โค้ดต่อไปนี้จะ error บน C++17
   ลงมา:

   ```cpp
   template <typename T>
   std::enable_if<std::is_integral<T>::value, T>::type   // ผิด! ขาด typename
   square(T x) { return x * x; }
   ```

   ```
   error: need 'typename' before 'std::enable_if<std::is_integral<_Tp>::value, T>::type'
   because 'std::enable_if<std::is_integral<_Tp>::value, T>' is a dependent scope
   ```

   ต้องเขียนเป็น `typename std::enable_if<...>::type` เสมอเมื่ออยู่ในบริบทที่ชนิดข้อมูลขึ้นอยู่กับ
   Template Parameter **ข้อควรระวังพิเศษ**: ตั้งแต่ C++20 เป็นต้นไป ข้อบังคับนี้ผ่อนคลายลงในบาง
   บริบท (ตาม proposal P0634 "Down with typename!") เช่น **ใน Return Type ของฟังก์ชัน** โค้ดด้านบน
   จะ**คอมไพล์ผ่านได้เฉยๆ โดยไม่ต้องใส่ `typename`** เมื่อใช้ `-std=c++20` แต่กฎนี้ใช้ได้เฉพาะบาง
   บริบทเท่านั้น (ไม่ครอบคลุมการประกาศตัวแปรหรือ type alias) จึงยังแนะนำให้เขียน `typename` ให้
   ครบเสมอเพื่อให้โค้ดคอมไพล์ได้ในทุกมาตรฐาน ไม่ใช่แค่ C++20

2. **ออกแบบเงื่อนไข `enable_if` ทับซ้อนกัน จนเกิด Ambiguous Overload** — ถ้าสอง overload มีเงื่อนไข
   ที่เป็นจริงพร้อมกันได้ (เช่น `is_integral` กับ `is_signed` ซึ่ง `int` เข้าเงื่อนไขทั้งคู่)
   คอมไพเลอร์จะไม่รู้ว่าจะเลือกตัวไหน:

   ```
   error: call of overloaded 'classify(int)' is ambiguous
   note: candidate: '...enable_if<is_integral<_Tp>::value,...
   note: candidate: '...enable_if<is_signed<_Tp>::value,...
   ```

   ทางแก้คือออกแบบเงื่อนไขให้ **แยกกันเด็ดขาด (Mutually Exclusive)** เสมอ เช่นใช้
   `is_integral<T>::value && !is_same<T, bool>::value` คู่กับเงื่อนไขตรงข้ามอย่างชัดเจน

3. **สับสนระหว่าง Error ที่เกิดจาก SFINAE กับ Error ธรรมดา** — SFINAE ใช้ได้เฉพาะความผิดพลาดที่
   เกิดใน "Immediate Context" ของ Template signature เท่านั้น ถ้า error เกิดลึกเข้าไปใน Function
   Body (หลังจาก Overload ถูกเลือกไปแล้ว) จะเป็น **Hard Error** ที่ทำให้คอมไพล์ล้มเหลวทั้งไฟล์ทันที
   ไม่ใช่แค่ตัด overload นั้นทิ้งเงียบๆ

4. **ใช้ Template Recursion คำนวณค่าที่ N ใหญ่เกินไป จนชน Instantiation Depth Limit** — อย่างที่
   เห็นใน 78.7 การ Recursion แบบ Naive (ไม่มี Memoization) จะสร้าง Template instantiation จำนวน
   เลขชี้กำลังตาม N เพิ่มขึ้น ถ้าจำเป็นต้องทำจริงๆ ควรใช้ `constexpr` function แทนเสมอในโค้ดสมัยใหม่

5. **ลืมว่า `std::is_same<T, U>` ตรวจสอบแบบเป๊ะ รวม const/reference ด้วย** — `is_same<int, int&>`
   และ `is_same<int, const int>` ล้วนเป็น `false` มือใหม่มักคาดหวังว่ามันจะ "มองข้าม" ความต่างเหล่านี้
   ถ้าต้องการเปรียบเทียบแบบไม่สนใจ reference/const ต้องใช้ `std::is_same<std::decay_t<T>, U>` แทน

6. **เขียน `std::declval<T>()` แล้วเผลอเรียกมันจริงนอก `decltype`/`sizeof`** — `std::declval` ถูก
   ออกแบบมาให้ใช้เฉพาะใน **Unevaluated Context** เท่านั้น (คือบริบทที่คอมไพเลอร์แค่ "ดูชนิดข้อมูล"
   โดยไม่รันโค้ดจริง) ถ้าเผลอเรียกมันในโค้ดที่รันจริง โปรแกรมจะ**คอมไพล์ไม่ผ่าน** (linker error
   undefined reference) เพราะฟังก์ชันนี้ไม่มี implementation จริงโดยตั้งใจ

---

## แบบฝึกหัดท้ายบท

1. เขียน function template `is_power_of_two<N>` (เป็น `struct` แบบ `Factorial` ใน 78.1) ที่คำนวณที่
   compile time ว่าเลขจำนวนเต็ม `N` เป็นเลขยกกำลังสอง (1, 2, 4, 8, 16, ...) หรือไม่ ผลลัพธ์เก็บใน
   `static constexpr bool value`

2. เขียนฟังก์ชัน `print_type_category(T value)` ด้วย `std::enable_if` สาม overload แยกสำหรับ
   `is_integral`, `is_floating_point`, และ `is_same<T, bool>` (ระวัง: `bool` ก็เป็น `is_integral`
   ด้วย ต้องออกแบบเงื่อนไขไม่ให้ทับซ้อนกันตาม Pitfall ข้อ 2)

3. เขียน Detection Idiom ชื่อ `has_begin_end<T>` ที่ตรวจสอบว่า Type หนึ่งมีทั้ง `.begin()` และ
   `.end()` หรือไม่ (ใช้ตรวจสอบว่าเป็น "container-like" หรือไม่) แล้วทดสอบกับ `std::vector<int>`
   กับ `int`

4. เขียน Concept เทียบเท่ากับ `has_begin_end` ในข้อ 3 โดยใช้ syntax `requires` แบบ Part 73
   เปรียบเทียบจำนวนบรรทัดและความอ่านง่ายกับข้อ 3

5. เขียน `struct SumTmpl<N>` ที่คำนวณผลรวม `1 + 2 + ... + N` ด้วย Template Recursion แล้วเขียน
   `constexpr` function `sum_constexpr(n)` ที่ทำสิ่งเดียวกัน เปรียบเทียบโค้ดทั้งสองแบบด้วย
   `static_assert` ว่าให้ผลลัพธ์ตรงกัน

6. อธิบายด้วยคำพูดตัวเอง (เขียนเป็นคอมเมนต์ในโค้ดหรือข้อความ) ว่าทำไมโค้ดต่อไปนี้ถึง Ambiguous
   Overload พร้อมแก้ไขให้ถูกต้อง:

   ```cpp
   template <typename T>
   typename std::enable_if<std::is_arithmetic<T>::value, void>::type
   process(T) { /* ... */ }

   template <typename T>
   typename std::enable_if<std::is_integral<T>::value, void>::type
   process(T) { /* ... */ }
   ```

### แนวทางเฉลยข้อ 1

```cpp
#include <iostream>

template <unsigned int N>
struct is_power_of_two {
    static constexpr bool value = (N > 0) && (N % 2 == 0) && is_power_of_two<N / 2>::value;
};

template <>
struct is_power_of_two<1> {
    static constexpr bool value = true;
};

template <>
struct is_power_of_two<0> {
    static constexpr bool value = false;
};

int main() {
    std::cout << std::boolalpha;
    std::cout << "is_power_of_two<16> = " << is_power_of_two<16>::value << "\n";
    std::cout << "is_power_of_two<18> = " << is_power_of_two<18>::value << "\n";
    std::cout << "is_power_of_two<1>  = " << is_power_of_two<1>::value << "\n";
    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++20 ex1.cpp -o ex1 && ./ex1
```

ผลลัพธ์:

```
is_power_of_two<16> = true
is_power_of_two<18> = false
is_power_of_two<1>  = true
```

**อธิบาย**: `is_power_of_two<N>` ใช้กฎว่า "เลขยกกำลังสองต้องหารด้วย 2 ลงตัวไปเรื่อยๆ จนเหลือ 1"
จึงเช็ค `N % 2 == 0` แล้ว recursive ไปที่ `N/2` โดยมีกรณีฐานสองแบบ: `is_power_of_two<1>` (จริงเสมอ
เพราะ 2^0 = 1) และ `is_power_of_two<0>` (เท็จเสมอ กันกรณี N=0 ซึ่งไม่ใช่เลขยกกำลังสอง)

### แนวทางเฉลยข้อ 2

```cpp
#include <iostream>
#include <type_traits>

template <typename T>
typename std::enable_if<std::is_same<T, bool>::value, void>::type
print_type_category(T value) {
    std::cout << std::boolalpha << value << " เป็นชนิด bool\n";
}

template <typename T>
typename std::enable_if<std::is_integral<T>::value && !std::is_same<T, bool>::value, void>::type
print_type_category(T value) {
    std::cout << value << " เป็นชนิด integral (ไม่ใช่ bool)\n";
}

template <typename T>
typename std::enable_if<std::is_floating_point<T>::value, void>::type
print_type_category(T value) {
    std::cout << value << " เป็นชนิด floating-point\n";
}

int main() {
    print_type_category(true);
    print_type_category(42);
    print_type_category(3.14);
    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++20 ex2.cpp -o ex2 && ./ex2
```

ผลลัพธ์:

```
true เป็นชนิด bool
42 เป็นชนิด integral (ไม่ใช่ bool)
3.14 เป็นชนิด floating-point
```

**อธิบาย**: จุดสำคัญที่สุดของโจทย์นี้คือการตระหนักว่า `bool` เป็น `is_integral` ด้วย (ตามมาตรฐาน
C++ `bool` ถูกจัดเป็นหนึ่งใน Integral Type) ถ้า overload ของ `bool` กับ overload ของ `is_integral`
ธรรมดาเขียนเงื่อนไขแยกกันโดยไม่ระวัง จะเกิด Ambiguous Overload ทันทีเมื่อเรียกด้วย `bool` (ตาม
Pitfall ข้อ 2) การแก้คือเติม `&& !std::is_same<T, bool>::value` เข้าไปใน overload ของ integral
ทั่วไป เพื่อ "กันขอบเขต" ไม่ให้ทับซ้อนกับ overload ของ `bool` โดยเฉพาะ

### แนวทางเฉลยข้อ 3

```cpp
#include <iostream>
#include <type_traits>
#include <utility>
#include <vector>

template <typename T, typename = void>
struct has_begin_end : std::false_type {};

template <typename T>
struct has_begin_end<T, std::void_t<decltype(std::declval<T>().begin()),
                                     decltype(std::declval<T>().end())>>
    : std::true_type {};

int main() {
    std::cout << std::boolalpha;
    std::cout << "has_begin_end<std::vector<int>> = "
              << has_begin_end<std::vector<int>>::value << "\n";
    std::cout << "has_begin_end<int> = " << has_begin_end<int>::value << "\n";
    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++20 ex3.cpp -o ex3 && ./ex3
```

ผลลัพธ์:

```
has_begin_end<std::vector<int>> = true
has_begin_end<int> = false
```

**อธิบาย**: ใช้เทคนิคเดียวกับ `has_to_string` ใน 78.5 ทุกประการ แต่ครั้งนี้ `std::void_t` รับ
`decltype` สองตัว (ทั้ง `.begin()` และ `.end()`) พร้อมกัน — ถ้า Type ใดขาดอย่างใดอย่างหนึ่งไป
Substitution จะล้มเหลวทั้งคู่ ทำให้ Partial Specialization ถูกตัดออก เหลือแค่ Primary Template
(`false_type`)

### แนวทางเฉลยข้อ 4

```cpp
#include <concepts>
#include <iostream>
#include <vector>

template <typename T>
concept HasBeginEnd = requires(T t) {
    t.begin();
    t.end();
};

template <HasBeginEnd T>
void describe_container(const T&) {
    std::cout << "เป็น container-like\n";
}

int main() {
    describe_container(std::vector<int>{});
    std::cout << std::boolalpha << HasBeginEnd<int> << "\n";
    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++20 ex4.cpp -o ex4 && ./ex4
```

ผลลัพธ์:

```
เป็น container-like
false
```

**อธิบาย**: เทียบกับข้อ 3 ที่ต้องเขียน `struct` สองตัว (Primary Template + Partial Specialization),
ใช้ `void_t`, `declval` และ `decltype` ประกอบกันถึง 6-7 บรรทัดของโค้ดที่อ่านยาก ข้อนี้ใช้ Concept
เพียง **4 บรรทัด** (`requires(T t) { t.begin(); t.end(); }`) ก็ได้ผลลัพธ์เดียวกันทุกประการ และยัง
นำ Concept นั้นไปใช้เป็นตัวจำกัด Template Parameter ได้ตรงๆ (`template <HasBeginEnd T>`) โดยไม่ต้อง
เขียน `enable_if` ห่อทับอีกชั้น นี่คือหลักฐานที่จับต้องได้ของสิ่งที่ตารางเปรียบเทียบใน 78.6 พูดถึง

### แนวทางเฉลยข้อ 5

```cpp
#include <iostream>

template <unsigned int N>
struct SumTmpl {
    static constexpr unsigned long long value = N + SumTmpl<N - 1>::value;
};

template <>
struct SumTmpl<0> {
    static constexpr unsigned long long value = 0;
};

constexpr unsigned long long sum_constexpr(unsigned int n) {
    unsigned long long total = 0;
    for (unsigned int i = 1; i <= n; ++i) {
        total += i;
    }
    return total;
}

int main() {
    static_assert(SumTmpl<100>::value == sum_constexpr(100), "ต้องตรงกัน");
    std::cout << "Sum(100) = " << SumTmpl<100>::value << "\n";
    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++20 ex5.cpp -o ex5 && ./ex5
```

ผลลัพธ์:

```
Sum(100) = 5050
```

**อธิบาย**: `static_assert` ใน `main()` ยืนยันว่าทั้งสองวิธี (Template Recursion กับ `constexpr`
Loop) ให้ผลลัพธ์ตรงกัน**ตั้งแต่ตอนคอมไพล์** — ถ้าผลลัพธ์ไม่ตรงกัน โปรแกรมจะ**คอมไพล์ไม่ผ่านเลย**
ไม่ใช่แค่รันแล้วได้ผลลัพธ์ผิด นี่คือพลังของการเขียน Test ที่ compile time ซึ่งเป็นเอกลักษณ์ของ C++
ที่ภาษาอื่นส่วนใหญ่ทำไม่ได้

### แนวทางเฉลยข้อ 6

โค้ดในโจทย์ Ambiguous เพราะ `int` (ตัวอย่างเช่น) ทำให้ทั้ง `is_arithmetic<int>::value` และ
`is_integral<int>::value` เป็น `true` พร้อมกัน คอมไพเลอร์จึงมี Candidate สอง overload ที่ใช้ได้
พร้อมกันและตัดสินใจไม่ได้ว่าจะเลือกตัวไหน (ไม่มีตัวไหน "เจาะจงกว่า" อีกตัวในสายตาของ Overload
Resolution เพราะทั้งคู่มาจากกลไก SFINAE คนละเงื่อนไข ไม่ใช่ Partial Ordering ของ Template ปกติ)

วิธีแก้ที่ถูกต้องคือทำให้เงื่อนไขแยกจากกันเด็ดขาด เช่น จำกัด overload แรกให้เหลือเฉพาะ
floating-point ซึ่งไม่ทับกับ `is_integral`:

```cpp
#include <type_traits>

template <typename T>
typename std::enable_if<std::is_floating_point<T>::value, void>::type
process(T) { /* จัดการ float/double */ }

template <typename T>
typename std::enable_if<std::is_integral<T>::value, void>::type
process(T) { /* จัดการ int/long/char/bool */ }

int main() {
    process(3.14);  // เรียก overload แรก (float)
    process(42);    // เรียก overload ที่สอง (integral)
    return 0;
}
```

ตอนนี้ `is_floating_point<T>` และ `is_integral<T>` ไม่มีทางเป็นจริงพร้อมกันได้เลยสำหรับ `T` ใดๆ
(ชนิดข้อมูลหนึ่งเป็นได้แค่อย่างใดอย่างหนึ่ง) จึงไม่มีทาง Ambiguous อีกต่อไป

---

## สรุปท้ายบท

ใน Part นี้เราได้:

- เข้าใจว่า Template Metaprogramming คือการใช้ระบบ Template ของ C++ เป็นภาษาที่ "รัน" ตอนคอมไพล์
  ผ่านตัวอย่าง `Factorial` และ Fibonacci แบบ Recursion
- เข้าใจกลไก SFINAE (Substitution Failure Is Not An Error) อย่างละเอียด ทั้งหลักการและข้อจำกัด
  (Immediate Context)
- ใช้ `std::enable_if` เขียน Compile-Time Dispatch เลือก overload ตามชนิดข้อมูลได้เอง ทั้งสอง
  Pattern หลัก (enable_if เป็น return type และเป็น default template parameter)
- รู้จัก Type Traits สำคัญใน `<type_traits>` ทั้งกลุ่ม Predicate (`is_integral`, `is_same`) และ
  กลุ่ม Transformation (`remove_reference`, `conditional`) พร้อมรูปแบบย่อ `_v`/`_t`
- เขียน Detection Idiom ด้วย `void_t` เพื่อตรวจสอบความสามารถของ Type ที่ compile time
- เปรียบเทียบ SFINAE กับ Concept (Part 73) อย่างละเอียดในตาราง และเข้าใจว่าทำไม Concept ถึงมาแทนที่
  ในโค้ดสมัยใหม่
- เห็นวิวัฒนาการของภาษาผ่านสองคู่เทียบ: SFINAE→Concept และ Template Recursion→`constexpr` (Part 71)
  ซึ่งเป็นแพทเทิร์นซ้ำๆ ของ C++ ที่ "แปลงเทคนิคแฮ็กให้เป็นฟีเจอร์ภาษาที่อ่านง่ายและปลอดภัยกว่า"

Metaprogramming แบบเก่านี้คือรากฐานสำคัญของอีกเทคนิคหนึ่งที่ใช้ Template ในรูปแบบที่แตกต่างออกไป
โดยสิ้นเชิง — **CRTP (Curiously Recurring Template Pattern)** ซึ่งใช้ Template ไม่ใช่เพื่อคำนวณ
ค่าคงที่ แต่เพื่อสร้าง **Static Polymorphism** ที่ทำงานได้เร็วเทียบเท่า C ธรรมดาโดยไม่มี Overhead
ของ `virtual` function เลย ใน **Part 79** เราจะเจาะลึกแพทเทิร์นนี้พร้อมเปรียบเทียบ Performance
กับ Dynamic Polymorphism จาก Part 49 อย่างจริงจัง

**ต่อไป:** [Part 79 — CRTP และ Template Pattern ขั้นสูง](./part-079-crtp-patterns.md)
