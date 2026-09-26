# Part 73: C++20 Concepts (Step 577–584)

> Module F — Modern C++ (C++11 ถึง C++23) | Part 73 จาก 125
> Step ที่ครอบคลุมใน Part นี้: Step 577–584
> Part ก่อนหน้า: [Part 72 — ฟีเจอร์ C++14/17](./part-072-cpp14-17-features.md) | Part ถัดไป: [Part 74 — C++20 Ranges](./part-074-cpp20-ranges.md)

## เป้าหมายการเรียนรู้ (Learning Objectives)

หลังจบ Part นี้ ผู้เรียนจะสามารถ:

1. อธิบายได้ว่าปัญหา "Error Message ยาวเป็นหน้าที่อ่านไม่รู้เรื่อง" ของ Template แบบเดิมคือ
   อะไร และทำไมมันถึงเป็นปัญหาใหญ่ในการพัฒนาโปรแกรมจริง
2. อธิบายได้ว่า Concept คืออะไรในเชิงแนวคิด และมันช่วยระบุ Constraint ของ Template Parameter
   ได้อย่างไร
3. เขียน `requires` Clause เพื่อจำกัดว่า Template Parameter ต้องมีคุณสมบัติอะไรบ้าง
4. เขียน Concept ของตัวเองได้ ทั้งแบบผสม Concept อื่น (Conjunction/Disjunction) และแบบใช้
   `requires` Expression ตรวจสอบว่า Type รองรับ Operation ที่ต้องการหรือไม่
5. ใช้ Concept มาตรฐานจาก `<concepts>` เช่น `std::integral`, `std::floating_point`,
   `std::same_as` ได้อย่างถูกต้อง
6. เปรียบเทียบ Error Message ก่อนและหลังใช้ Concept จากตัวอย่างจริงที่คอมไพล์แล้วเห็นความ
   แตกต่างชัดเจน
7. เขียน Function Template แบบย่อ (Abbreviated Function Template Syntax) ด้วย `auto` และ
   Concept ผสมกัน

---

## เกร็ดประวัติศาสตร์: Concepts เกือบได้เข้า C++11 มาก่อนแล้ว

Concepts ไม่ใช่แนวคิดใหม่ที่เพิ่งคิดขึ้นสำหรับ C++20 — จริงๆ แล้วมันถูกเสนอเข้าสู่มาตรฐานตั้งแต่
ช่วงพัฒนา C++11 ภายใต้ชื่อ "Concepts (C++0x)" แต่การออกแบบตอนนั้นซับซ้อนเกินไปและยังไม่ลงตัว
คณะกรรมการมาตรฐานจึงตัดสินใจถอดออกจาก C++11 ก่อนจะประกาศใช้จริงในนาทีสุดท้าย (2009) เพื่อไม่ให้
มาตรฐานทั้งฉบับล่าช้าไปอีกหลายปี ทีมงานใช้เวลาเกือบ 10 ปีถัดมาออกแบบใหม่ให้เรียบง่ายขึ้นภายใต้ชื่อ
"Concepts Lite" ก่อนจะได้เข้า C++20 ในที่สุด — เป็นตัวอย่างที่ดีว่าฟีเจอร์ภาษาใหญ่ๆ บางตัวต้องใช้
เวลาบ่มเพาะนานแค่ไหนกว่าจะพร้อมใช้งานจริง

---

## บริบท: ทำไม C++20 ถึงต้องมี Concepts

Template คือหัวใจของ Generic Programming ใน C++ มาตั้งแต่เรียนใน Part 56–57 แต่ Template
มีจุดอ่อนใหญ่มากจุดหนึ่งมาตลอด 20 กว่าปี: **มันไม่มีวิธีบอก Compiler ล่วงหน้าว่า Template
Parameter ต้องมีคุณสมบัติอะไรบ้าง** Template ทำงานแบบ "Duck Typing ตอน Compile-Time" คือ
ลองแทน Type เข้าไปก่อน ถ้าโค้ดข้างในใช้ Operation ที่ Type นั้นไม่รองรับ ก็จะ Error ตอนที่
Compiler พยายาม Instantiate Template นั้นจริงๆ (เรียกว่า "จุดใช้งาน" หรือ Point of
Instantiation) ซึ่งอาจจะอยู่ลึกเข้าไปหลายชั้นจาก Standard Library

ผลที่ตามมาคือ Error Message ที่ยาวเป็นสิบๆ ถึงร้อยกว่าบรรทัด ชี้ไปที่โค้ดข้างในของ
`<algorithm>` หรือ `<vector>` แทนที่จะชี้ตรงไปยังจุดที่ผู้เขียนโค้ดทำผิดจริงๆ Part นี้จะเริ่ม
จากการแสดงปัญหานี้ให้เห็นจริงด้วยตาตัวเอง ก่อนจะแนะนำ **Concepts** ซึ่งเป็นฟีเจอร์ที่ใหญ่ที่สุด
ตัวหนึ่งของ C++20 ที่เข้ามาแก้ปัญหานี้โดยตรง

---

## 73.1 ปัญหาที่ Concept แก้: Error Message ที่อ่านไม่รู้เรื่อง (Step 577)

ลองมาดูตัวอย่างที่พบได้บ่อยมากในโค้ดจริง: เขียน Function Template ของตัวเองที่เรียกใช้
`std::max_element` จาก `<algorithm>` (เรียนไปแล้วใน Part 63) ข้างใน:

```cpp
#include <algorithm>
#include <vector>

struct Point {
    int x;
    int y;
};

template <typename T>
T find_max(const std::vector<T>& values) {
    return *std::max_element(values.begin(), values.end());
}

int main() {
    std::vector<Point> points{{3, 1}, {1, 2}, {2, 0}};
    Point best = find_max(points);   // ผู้เรียก "ลืม" ว่า Point ต้องเปรียบเทียบกันได้
    (void)best;
    return 0;
}
```

`find_max` ดูเหมือนจะใช้ได้กับ Type อะไรก็ได้ที่อยู่ใน `std::vector` แต่จริงๆ แล้วมันต้องการ
ให้ `T` เปรียบเทียบด้วย `operator<` ได้ (เพราะ `std::max_element` ใช้ `operator<` เปรียบเทียบ
ค่าภายใน) ซึ่ง `Point` ในตัวอย่างนี้ไม่มี `operator<` ให้ ลองคอมไพล์ดูจริง:

```bash
g++ -Wall -Wextra -Wpedantic -std=c++20 -c find_max_before.cpp
```

**นี่คือ Error Message จริงที่ได้จาก g++ 13.3.0** (ตัดมาแสดงบางส่วน เพราะฉบับเต็มยาวกว่านี้):

```
In file included from /usr/include/c++/13/bits/stl_algobase.h:71,
                 from /usr/include/c++/13/algorithm:60,
                 from find_max_before.cpp:1:
/usr/include/c++/13/bits/predefined_ops.h: In instantiation of 'constexpr bool __gnu_cxx::__ops::_Iter_less_iter::operator()(_Iterator1, _Iterator2) const [with _Iterator1 = __gnu_cxx::__normal_iterator<const Point*, std::vector<Point> >; _Iterator2 = __gnu_cxx::__normal_iterator<const Point*, std::vector<Point> >]':
/usr/include/c++/13/bits/stl_algo.h:5715:12:   required from 'constexpr _ForwardIterator std::__max_element(_ForwardIterator, _ForwardIterator, _Compare) [with _ForwardIterator = __gnu_cxx::__normal_iterator<const Point*, vector<Point> >; _Compare = __gnu_cxx::__ops::_Iter_less_iter]'
/usr/include/c++/13/bits/stl_algo.h:5739:43:   required from 'constexpr _FIter std::max_element(_FIter, _FIter) [with _FIter = __gnu_cxx::__normal_iterator<const Point*, vector<Point> >]'
find_max_before.cpp:11:29:   required from 'T find_max(const std::vector<T>&) [with T = Point]'
find_max_before.cpp:16:26:   required from here
/usr/include/c++/13/bits/predefined_ops.h:45:23: error: no match for 'operator<' (operand types are 'const Point' and 'const Point')
   45 |       { return *__it1 < *__it2; }
      |                ~~~~~~~^~~~~~~~
In file included from /usr/include/c++/13/bits/stl_algobase.h:67:
/usr/include/c++/13/bits/stl_iterator.h:1189:5: note: candidate: 'template<class _IteratorL, class _IteratorR, class _Container> constexpr std::__detail::__synth3way_t<_IteratorR, _IteratorL> __gnu_cxx::operator<=>(const __normal_iterator<_IteratorL, _Container>&, const __normal_iterator<_IteratorR, _Container>&)' (reversed)
 1189 |     operator<=>(const __normal_iterator<_IteratorL, _Container>& __lhs,
      |     ^~~~~~~~
/usr/include/c++/13/bits/stl_iterator.h:1189:5: note:   template argument deduction/substitution failed:
/usr/include/c++/13/bits/predefined_ops.h:45:23: note:   'const Point' is not derived from 'const __gnu_cxx::__normal_iterator<_IteratorL, _Container>'
   45 |       { return *__it1 < *__it2; }
      |                ~~~~~~~^~~~~~~~
/usr/include/c++/13/bits/stl_iterator.h:1208:5: note: candidate: 'template<class _Iterator, class _Container> constexpr std::__detail::__synth3way_t<_T1> __gnu_cxx::operator<=>(const __normal_iterator<_Iterator, _Container>&, const __normal_iterator<_Iterator, _Container>&)' (rewritten)
 1208 |     operator<=>(const __normal_iterator<_Iterator, _Container>& __lhs,
      |     ^~~~~~~~
```

ทั้งหมดนี้คือ **26 บรรทัด** ของ Error สำหรับความผิดพลาดที่จริงๆ แล้วสรุปได้ในประโยคเดียวว่า
"`Point` เปรียบเทียบด้วย `<` ไม่ได้" สังเกตปัญหาสำคัญ 3 อย่าง:

1. **Error ไม่ได้ชี้ไปที่จุดที่ผู้เขียนโค้ดทำผิด** (บรรทัดที่เรียก `find_max(points)`) แต่ชี้ไป
   ที่ไฟล์ Internal ของ Library อย่าง `predefined_ops.h` และ `stl_iterator.h`
2. **ต้องไล่อ่าน "Instantiation Stack"** (บรรทัดที่ขึ้นต้นด้วย "required from") ย้อนกลับขึ้นไป
   หลายชั้นกว่าจะเจอบรรทัดที่เป็นโค้ดของเราเอง (`find_max_before.cpp:11:29`)
3. **ชื่อ Type ภายในของ Library** เช่น `__gnu_cxx::__normal_iterator`, `__synth3way_t` ทำให้
   Error อ่านยากขึ้นไปอีก เพราะเป็นรายละเอียดการ Implement ที่ผู้ใช้ไม่ควรต้องรู้เลย

สำหรับ Template ที่ซับซ้อนกว่านี้ (เช่น Template ที่ซ้อนกันหลายชั้น หรือใช้ Type Trait
เยอะๆ) Error แบบนี้อาจยาวได้เป็น**หลายร้อยบรรทัด** และเป็นสาเหตุสำคัญที่ทำให้โปรแกรมเมอร์
C++ รุ่นก่อน C++20 ใช้เวลานานมากในการ Debug ปัญหา Type ผิดพลาดใน Template Code

---

## 73.2 Concept คืออะไร และ requires Clause (Step 578)

### Concept คืออะไร

**Concept** คือชื่อที่ตั้งให้กับ "เงื่อนไข" หรือ "ชุดคุณสมบัติ" ที่ Template Parameter ต้องมี
พูดง่ายๆ คือมันคือ **การประกาศ Constraint (ข้อจำกัด) ของ Template Parameter ให้ Compiler
ตรวจสอบตั้งแต่ตอน Compile ก่อนที่จะพยายาม Instantiate Template จริง** แทนที่จะปล่อยให้ Error
เกิดขึ้นลึกๆ ข้างในเนื้อหาของ Template เหมือนที่เห็นในหัวข้อก่อนหน้า

Concept ตอบคำถามแบบนี้ได้ทันทีจาก Signature ของ Template: "T ต้องเป็นตัวเลขไหม", "T ต้อง
เปรียบเทียบกันได้ไหม", "T ต้องมี Method ชื่อ `size()` ไหม" — ทั้งหมดนี้เขียนเป็น Concept ได้

### requires Clause

`requires` Clause คือไวยากรณ์ที่ใช้ "ผูก" Concept (หรือเงื่อนไข Compile-Time อื่นๆ) เข้ากับ
Template โดยเขียนต่อท้ายรายการ Template Parameter:

```cpp
#include <concepts>
#include <iostream>

template <typename T>
    requires std::integral<T>     // T ต้องเป็นจำนวนเต็มเท่านั้น
T double_it(T x) {
    return x * 2;
}

int main() {
    std::cout << double_it(21) << "\n";      // ใช้ได้ T = int
    // double_it(3.14);                      // Compile ไม่ผ่าน! double ไม่ใช่ integral
    return 0;
}
```

`std::integral` ในตัวอย่างนี้คือ Concept มาตรฐานจาก `<concepts>` ที่จะพูดถึงรายละเอียดในหัวข้อ
73.4 ประโยค `requires std::integral<T>` แปลว่า "Template นี้จะถูกพิจารณาใช้งานได้ ก็ต่อเมื่อ
`T` เป็นจำนวนเต็มเท่านั้น" ถ้าใครพยายามเรียกด้วย Type ที่ไม่ใช่จำนวนเต็ม Compiler จะปฏิเสธ
การ Match Template ตัวนี้ตั้งแต่ขั้นตอน Overload Resolution เลย ไม่ต้องรอไปเจอปัญหาลึกๆ
ข้างในเนื้อหาฟังก์ชัน

### รูปแบบของ requires Clause

`requires` Clause เขียนได้หลายตำแหน่ง ซึ่งให้ผลลัพธ์เหมือนกันทุกแบบ (จะสาธิตทั้งหมดแบบ
ละเอียดในหัวข้อ 73.7):

```cpp
// แบบที่ 1: constraint แทนที่ typename ตรงๆ (กระชับที่สุด)
template <std::integral T>
T style1(T x) { return x + 1; }

// แบบที่ 2: requires clause ท้ายรายการ template parameter
template <typename T>
    requires std::integral<T>
T style2(T x) { return x + 1; }

// แบบที่ 3: requires clause ท้ายฟังก์ชัน (trailing requires)
template <typename T>
T style3(T x) requires std::integral<T> { return x + 1; }
```

---

## 73.3 การเขียน Concept ของตัวเอง (Step 579)

นอกจาก Concept มาตรฐานที่มีมาให้แล้ว เราเขียน Concept ของตัวเองได้ด้วยคีย์เวิร์ด `concept`
มีสองรูปแบบหลักที่ใช้บ่อย: **การรวม Concept อื่นด้วย Logical Operator** และ **การใช้
`requires` Expression ตรวจสอบว่า Type รองรับ Operation ที่ต้องการหรือไม่**

### รูปแบบที่ 1: รวม Concept ด้วย && และ ||

```cpp
#include <concepts>
#include <iostream>

// นิยาม concept ของตัวเอง: Numeric แปลว่า "T ต้องเป็นตัวเลข (integral หรือ floating point)"
template <typename T>
concept Numeric = std::integral<T> || std::floating_point<T>;

// requires clause: บังคับว่า T ที่ใช้กับฟังก์ชันนี้ต้องผ่าน concept Numeric
template <typename T>
    requires Numeric<T>
T average(T a, T b) {
    return (a + b) / 2;
}

int main() {
    std::cout << average(3, 7) << "\n";       // ใช้ได้: int เข้าเงื่อนไข integral
    std::cout << average(2.5, 3.5) << "\n";   // ใช้ได้: double เข้าเงื่อนไข floating_point
    return 0;
}
```

ผลลัพธ์:

```
5
3
```

ประโยค `concept Numeric = std::integral<T> || std::floating_point<T>;` อ่านได้ตรงตัวเลยว่า
"Numeric คือ Type ที่เป็น integral หรือ floating_point อย่างใดอย่างหนึ่ง" — Concept ใหม่ที่
สร้างขึ้นนี้ (`Numeric`) ใช้งานได้เหมือน Concept มาตรฐานทุกประการ ทั้งใน `requires` Clause
และแทนที่ `typename` ตรงๆ

### รูปแบบที่ 2: requires Expression — ตรวจสอบว่า Type "ทำอะไรได้บ้าง"

บางครั้งเราอยากตรวจสอบว่า Type รองรับ Operation ที่ซับซ้อนกว่าแค่ "เป็นตัวเลขไหม" เช่น
"พิมพ์ออก `std::cout` ได้ไหม" ซึ่งทำได้ด้วย `requires` Expression (สังเกตว่ามีทั้ง `requires`
Clause และ `requires` Expression ที่หน้าตาคล้ายกันแต่ทำหน้าที่ต่างกัน — Clause คือส่วนที่
"ผูก" Concept เข้ากับ Template ส่วน Expression คือไวยากรณ์ที่ใช้ **นิยาม** Concept ใหม่):

```cpp
#include <concepts>
#include <iostream>
#include <string>

// concept ที่ซับซ้อนกว่า: ใช้ requires-expression ตรวจว่า T มี method/operator ที่ต้องการ
template <typename T>
concept Printable = requires(T value) {
    { std::cout << value } -> std::same_as<std::ostream&>;
};

template <Printable T>
void print_it(const T& value) {
    std::cout << "ค่า: " << value << "\n";
}

int main() {
    print_it(42);
    print_it(std::string("hello concept"));
    return 0;
}
```

ผลลัพธ์:

```
ค่า: 42
ค่า: hello concept
```

อ่านไวยากรณ์ `requires(T value) { { std::cout << value } -> std::same_as<std::ostream&>; }`
ได้ทีละส่วนดังนี้:

- `requires(T value) { ... }` — ประกาศ Parameter สมมติชื่อ `value` ของ Type `T` ไว้ใช้ทดสอบ
  ข้างใน (ไม่ได้ถูกเรียกใช้งานจริง แค่ใช้ให้ Compiler ทดลอง Type-Check เท่านั้น)
- `{ std::cout << value }` — Compound Requirement: ทดสอบว่านิพจน์ `std::cout << value` เขียน
  ได้จริงไหม (Type `T` ต้องรองรับ `operator<<` กับ `std::ostream`)
- `-> std::same_as<std::ostream&>` — นอกจากเขียนได้แล้ว ผลลัพธ์ของนิพจน์นั้นต้องมี Type ตรงกับ
  `std::ostream&` ด้วย (นี่คือผลลัพธ์ที่ `operator<<` มาตรฐานคืนกลับมาเสมอ เพื่อให้ Chain
  `std::cout << a << b` ต่อกันได้)

ถ้าทุกเงื่อนไขข้างในเป็นจริง Concept `Printable` จะประเมินผลเป็น `true` สำหรับ Type นั้น

---

## 73.4 Concept มาตรฐานใน `<concepts>` (Step 580)

C++20 มากับ Concept สำเร็จรูปจำนวนมากใน Header `<concepts>` ที่ครอบคลุมความต้องการพื้นฐาน
ที่พบบ่อยที่สุด ตารางนี้สรุป Concept ที่ใช้งานบ่อยที่สุด:

| Concept | ความหมาย |
|---|---|
| `std::integral<T>` | `T` เป็นชนิดจำนวนเต็ม (`int`, `long`, `char`, `bool`, `unsigned` ต่างๆ) |
| `std::floating_point<T>` | `T` เป็นชนิดทศนิยม (`float`, `double`, `long double`) |
| `std::same_as<T, U>` | `T` กับ `U` เป็นชนิดเดียวกัน**เป๊ะๆ** (ไม่มีการแปลงชนิดให้) |
| `std::convertible_to<From, To>` | ค่าชนิด `From` แปลงเป็น `To` ได้โดยปริยาย (Implicit Conversion) |
| `std::default_initializable<T>` | `T` สร้างด้วย Default Constructor ได้ (`T x;` ใช้งานได้) |
| `std::copyable<T>` | `T` Copy ได้ (มี Copy Constructor และ Copy Assignment) |
| `std::equality_comparable<T>` | `T` เปรียบเทียบด้วย `==` และ `!=` ได้ |
| `std::totally_ordered<T>` | `T` เปรียบเทียบด้วย `<`, `>`, `<=`, `>=` ได้ครบ (เรียงลำดับได้เต็มรูปแบบ) |

### ตัวอย่างการใช้งาน

```cpp
#include <concepts>
#include <iostream>

// std::integral<T>: T ต้องเป็นชนิดจำนวนเต็ม (int, long, char, bool, ...)
template <std::integral T>
T double_it(T x) {
    return x * 2;
}

// std::floating_point<T>: T ต้องเป็นชนิดทศนิยม (float, double, long double)
template <std::floating_point T>
T half_it(T x) {
    return x / 2;
}

// std::same_as<T, U>: บังคับว่า T ต้องเป็นชนิดเดียวกับ U เป๊ะๆ (ไม่แปลงชนิดให้)
template <typename T, typename U>
    requires std::same_as<T, U>
bool same_type_equal(T a, U b) {
    return a == b;
}

int main() {
    std::cout << double_it(21) << "\n";
    std::cout << half_it(21.0) << "\n";
    std::cout << std::boolalpha << same_type_equal(5, 5) << "\n";
    // same_type_equal(5, 5.0) จะคอมไพล์ไม่ผ่าน เพราะ int กับ double ไม่ใช่ same_as กัน
    return 0;
}
```

ผลลัพธ์:

```
42
10.5
true
```

ลองพิสูจน์คำกล่าวในคอมเมนต์บรรทัดสุดท้ายว่าเป็นจริง โดยเปลี่ยนมาเรียก
`same_type_equal(5, 5.0)` จริงๆ:

```
error: no matching function for call to 'same_type_equal(int, double)'
note: candidate: 'template<class T, class U>  requires  same_as<T, U> bool same_type_equal(T, U)'
note:   template argument deduction/substitution failed:
note: constraints not satisfied
```

สังเกตว่า Error สั้นกระชับมาก และบอกตรงๆ ว่า `constraints not satisfied` (เงื่อนไข Constraint
ไม่ผ่าน) — นี่คือรสชาติของ Error Message แบบ Concepts ที่จะเปรียบเทียบให้เห็นชัดเจนกว่านี้อีก
ในหัวข้อถัดไป

> **ทำไม `std::same_as<T, U>` ถึงสำคัญ**: ถ้าไม่มี Constraint นี้ Compiler จะพยายามแปลง `int`
> เป็น `double` (หรือกลับกัน) ให้อัตโนมัติเพื่อให้ `operator==` ทำงานได้ ซึ่งบางครั้งไม่ใช่สิ่งที่
> ผู้เขียนโค้ดตั้งใจ — `same_as` บังคับให้ผู้เรียกต้องส่ง Type ที่ตรงกันเป๊ะๆ เท่านั้น ป้องกัน
> การเปรียบเทียบข้าม Type ที่อาจทำให้เกิดบั๊กเงียบๆ

---

## 73.5 เปรียบเทียบ Error Message ก่อน/หลังใช้ Concept (Step 581)

ตอนนี้เรามาดูฟังก์ชัน `find_max` ตัวเดิมจากหัวข้อ 73.1 อีกครั้ง แต่คราวนี้เพิ่ม Concept
`std::totally_ordered` เข้าไปจำกัด Type Parameter:

```cpp
#include <algorithm>
#include <concepts>
#include <vector>

struct Point {
    int x;
    int y;
};

template <std::totally_ordered T>
T find_max(const std::vector<T>& values) {
    return *std::max_element(values.begin(), values.end());
}

int main() {
    std::vector<Point> points{{3, 1}, {1, 2}, {2, 0}};
    Point best = find_max(points);
    (void)best;
    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++20 -c find_max_after.cpp
```

**Error Message จริงที่ได้หลังใช้ Concept:**

```
find_max_after.cpp: In function 'int main()':
find_max_after.cpp:17:26: error: no matching function for call to 'find_max(std::vector<Point>&)'
   17 |     Point best = find_max(points);
      |                  ~~~~~~~~^~~~~~~~
find_max_after.cpp:11:3: note: candidate: 'template<class T>  requires  totally_ordered<T> T find_max(const std::vector<T>&)'
   11 | T find_max(const std::vector<T>& values) {
      |   ^~~~~~~~
find_max_after.cpp:11:3: note:   template argument deduction/substitution failed:
find_max_after.cpp:11:3: note: constraints not satisfied
In file included from /usr/include/c++/13/compare:37,
                 from /usr/include/c++/13/bits/stl_pair.h:65,
                 from /usr/include/c++/13/bits/stl_algobase.h:64,
                 from /usr/include/c++/13/algorithm:60,
                 from find_max_after.cpp:1:
/usr/include/c++/13/concepts: In substitution of 'template<class T>  requires  totally_ordered<T> T find_max(const std::vector<T>&) [with T = Point]':
find_max_after.cpp:17:26:   required from here
/usr/include/c++/13/concepts:294:15:   required for the satisfaction of '__weakly_eq_cmp_with<_Tp, _Tp>' [with _Tp = Point]
/usr/include/c++/13/concepts:304:13:   required for the satisfaction of 'equality_comparable<_Tp>' [with _Tp = Point]
/usr/include/c++/13/concepts:333:13:   required for the satisfaction of 'totally_ordered<T>' [with T = Point]
/usr/include/c++/13/concepts:295:4:   in requirements with 'std::remove_reference_t<_Tp>& __t', 'std::remove_reference_t<_Up>& __u' [with _Tp = Point; _Up = Point]
/usr/include/c++/13/concepts:296:17: note: the required expression '(__t == __u)' is invalid
  296 |           { __t == __u } -> __boolean_testable;
      |             ~~~~^~~~~~
cc1plus: note: set '-fconcepts-diagnostics-depth=' to at least 2 for more detail
```

### เปรียบเทียบแบบเคียงข้าง

| ประเด็น | ก่อนใช้ Concept (73.1) | หลังใช้ Concept (73.5) |
|---|---|---|
| บรรทัดแรกของ Error ชี้ไปที่ | ไฟล์ Internal ของ Library (`predefined_ops.h`) | ตรงจุดที่เรียก `find_max(points)` ในโค้ดของเราเอง |
| ข้อความสรุปปัญหา | ต้องอนุมานเอาเองจาก `no match for 'operator<'` ที่ฝังอยู่ลึกๆ | บอกตรงๆ ว่า `constraints not satisfied` ตั้งแต่บรรทัดที่ 5 |
| ต้องไล่ Instantiation Stack กี่ชั้นกว่าจะเจอบรรทัดของเรา | 4 ชั้น (ผ่าน `__max_element`, `max_element`, `find_max`) | 1 ชั้น (ชี้ตรงจุดเรียกทันทีตั้งแต่บรรทัดแรก) |
| ระบุ Requirement ที่ขาดหายไปชัดเจนไหม | ไม่ชัดเจน ต้องตีความจากชื่อฟังก์ชัน `__gnu_cxx::__ops::_Iter_less_iter` | ชัดเจน: `the required expression '(__t == __u)' is invalid` |
| Compiler แนะนำวิธีดูรายละเอียดเพิ่มไหม | ไม่มี | มี (`set '-fconcepts-diagnostics-depth=' to at least 2 for more detail`) |

**ข้อสังเกตสำคัญ**: Error หลังใช้ Concept ก็ยังมีส่วนที่ไปโผล่ในไฟล์ `<concepts>` ของ Library
อยู่บ้าง (เพราะ `std::totally_ordered` เป็น Concept ที่ประกอบขึ้นจาก Concept ย่อยหลายตัว) แต่
สิ่งที่เปลี่ยนไปอย่างชัดเจนคือ **ประโยคสรุปตั้งแต่ต้น** (`constraints not satisfied`) และ
**Instantiation Stack ที่สั้นลงมาก** ทำให้ผู้เขียนโค้ดรู้ทันทีว่าปัญหาคือ Type ไม่ตรงตาม
Constraint ที่ประกาศไว้ ไม่ใช่ต้องมานั่งไล่อ่านโค้ดภายในของ `<algorithm>` เอง — ยิ่งถ้าเขียน
Concept เองแบบง่ายๆ (ไม่ซับซ้อนเท่า `totally_ordered`) Error ที่ได้จะสั้นและชัดเจนกว่านี้อีก
มาก ดังที่เห็นในตัวอย่าง `same_type_equal` ของหัวข้อก่อนหน้า

---

## 73.6 Abbreviated Function Template Syntax (Step 582)

C++20 เปิดโอกาสให้เขียน Function Template แบบย่อได้ โดยใช้ `auto` (หรือ Concept ตามด้วย
`auto`) แทนพารามิเตอร์ตรงๆ โดยไม่ต้องประกาศ `template<typename T>` แยกบรรทัดเลย ไวยากรณ์นี้
เรียกว่า **Abbreviated Function Template Syntax**

```cpp
#include <concepts>
#include <iostream>

// เขียนแบบ template ปกติ พร้อม constraint
template <std::integral T>
T add_verbose(T a, T b) {
    return a + b;
}

// เขียนแบบย่อ (Abbreviated Function Template, C++20): ใช้ concept แทนคำว่า typename
// ตรงพารามิเตอร์ได้เลย ไม่ต้องประกาศ template<...> แยกบรรทัด — คอมไพเลอร์สร้าง
// template parameter ให้อัตโนมัติเบื้องหลัง
std::integral auto add_short(std::integral auto a, std::integral auto b) {
    return a + b;
}

// auto ธรรมดา (ไม่มี constraint) ก็เขียนแบบย่อได้เช่นกันมาตั้งแต่ C++20
auto add_generic(auto a, auto b) {
    return a + b;
}

int main() {
    std::cout << add_verbose(2, 3) << "\n";
    std::cout << add_short(4, 5) << "\n";
    std::cout << add_generic(1.5, 2.5) << "\n";
    return 0;
}
```

ผลลัพธ์:

```
5
9
4
```

`add_verbose` และ `add_short` เทียบเท่ากันทุกประการในสายตาของ Compiler — ทั้งคู่ถูกแปลงเป็น
Function Template ที่มี Constraint `std::integral<T>` เหมือนกัน เพียงแต่ `add_short` เขียน
สั้นกว่ามาก โดย Compiler จะสร้าง Template Parameter ให้อัตโนมัติหนึ่งตัวสำหรับ `auto`
แต่ละตัวที่ปรากฏในรายการพารามิเตอร์ (สังเกตว่า `add_short` มี `std::integral auto` สองตัว
ในพารามิเตอร์ ซึ่งหมายความว่าจริงๆ แล้วมันคือ Template ที่มี **สอง** Type Parameter แยกกัน
ไม่ใช่ตัวเดียว — ต่างจาก `add_verbose` ที่บังคับให้ `a` กับ `b` เป็น Type เดียวกัน (`T`) เป๊ะๆ)

`add_generic` แสดงให้เห็นว่าแม้ไม่มี Concept กำกับเลย ก็ยังเขียนแบบย่อได้เหมือนกัน — นี่คือ
Syntax เดียวกับที่ Generic Lambda ของ C++14 (หัวข้อ 72.1) ใช้อยู่แล้ว เพียงแต่ตอนนี้ขยายมาใช้
กับ Function ธรรมดา (ไม่ใช่แค่ Lambda) ได้ด้วยใน C++20

### เมื่อไหร่ควรใช้แบบเต็ม เมื่อไหร่ควรใช้แบบย่อ

| สถานการณ์ | แนะนำให้ใช้ |
|---|---|
| พารามิเตอร์หลายตัวต้องเป็น Type เดียวกันเป๊ะๆ | แบบเต็ม (`template <typename T> f(T a, T b)`) — ประกาศ `T` ครั้งเดียวแล้วใช้ซ้ำ |
| แต่ละพารามิเตอร์เป็นคนละ Type กันได้ ไม่สนใจว่าต้องตรงกัน | แบบย่อ (`f(auto a, auto b)`) — เขียนสั้นกว่ามาก |
| ต้องใช้ชื่อ Type Parameter (`T`) ซ้ำในตัว Return Type หรือ Body ของฟังก์ชันแบบซับซ้อน | แบบเต็ม — เพราะแบบย่อไม่มีชื่อ `T` ให้อ้างอิงตรงๆ |
| ฟังก์ชันสั้นๆ ที่ Concept ของแต่ละพารามิเตอร์ชัดเจนอยู่แล้ว | แบบย่อ — อ่านง่ายและกระชับกว่า |

---

## 73.7 requires Clause ในตำแหน่งต่างๆ และการรวม Concept (Step 583)

มาดูภาพรวมของทุกวิธีในการเขียน Constraint ให้ Template พร้อมกันในที่เดียว รวมถึงการรวม
หลาย Concept เข้าด้วยกันด้วย `&&` (Conjunction — ต้องผ่านทุกเงื่อนไข) และ `||` (Disjunction
— ผ่านเงื่อนไขใดเงื่อนไขหนึ่งพอ) และการผสม Concept กับนิพจน์ Compile-Time ธรรมดา:

```cpp
#include <concepts>
#include <iostream>

// วิธีที่ 1: constraint แทนที่ typename ตรงๆ
template <std::integral T>
T style1(T x) { return x + 1; }

// วิธีที่ 2: requires clause ท้ายรายการ template parameter
template <typename T>
    requires std::integral<T>
T style2(T x) { return x + 1; }

// วิธีที่ 3: requires clause ท้ายฟังก์ชัน (trailing requires) — มีประโยชน์เมื่อ
// constraint ต้องอ้างอิงพารามิเตอร์ของฟังก์ชัน ไม่ใช่แค่ตัว type parameter
template <typename T>
T style3(T x) requires std::integral<T> { return x + 1; }

// รวมหลาย concept ด้วย && (conjunction) และ || (disjunction)
template <typename T>
    requires std::integral<T> || std::floating_point<T>
T style4(T x) { return x + 1; }

template <typename T>
    requires std::integral<T> && (sizeof(T) >= 4)
T style5(T x) { return x + 1; }   // ผสม concept กับนิพจน์ compile-time ปกติได้ด้วย

int main() {
    std::cout << style1(1) << " " << style2(2) << " " << style3(3)
              << " " << style4(4.5) << " " << style5(5) << "\n";
    return 0;
}
```

ผลลัพธ์:

```
2 3 4 5.5 6
```

`style5` แสดงให้เห็นว่า `requires` Clause ไม่จำเป็นต้องมีแค่ Concept เท่านั้น — ผสมกับนิพจน์
Compile-Time ทั่วไปที่ให้ผลเป็น `bool` ได้ด้วย (`sizeof(T) >= 4` คือเงื่อนไขที่ตรวจสอบว่า Type
มีขนาดอย่างน้อย 4 Byte เช่น `int`/`long` ผ่าน แต่ `short`/`char` จะไม่ผ่าน)

### ทำไมต้องมี "Trailing requires" (style3) ด้วย ในเมื่อ style2 ก็ทำแบบเดียวกันได้

เหตุผลหลักคือ Trailing `requires` เขียนที่ตำแหน่ง**หลังพารามิเตอร์ของฟังก์ชัน** ทำให้อ้างอิง
พารามิเตอร์ของฟังก์ชันได้โดยตรงในกรณีที่ Constraint ต้องพึ่งค่าพารามิเตอร์ (ไม่ใช่แค่ Type)
เช่น การตรวจสอบด้วย `requires` Expression ที่ต้องใช้ Object ของพารามิเตอร์จริง ในขณะที่
`requires` Clause ตำแหน่งอื่น (style2) เขียนก่อนเห็นพารามิเตอร์ของฟังก์ชันเสียอีก จึงใช้ได้
เฉพาะกับ Type Parameter ของ Template เท่านั้น สำหรับ Concept ธรรมดาอย่างในตัวอย่างนี้ ทั้งสาม
แบบให้ผลเหมือนกันทุกประการ — เลือกใช้ตามความชอบด้านความอ่านง่ายได้เลย

---

## 73.8 ตัวอย่างเชิงปฏิบัติ: รวมทุกอย่างเข้าด้วยกัน (Step 584)

มาดูตัวอย่างที่ผสมทุกเทคนิคที่เรียนมาในบทนี้เข้าด้วยกัน: การเขียน Concept ที่ตรวจสอบว่า Type
มี `.size()` และ Iterate ได้ (`Iterable`), การใช้ Concept รวมกันหลายตัวด้วย `&&`, และการ
Overload ฟังก์ชันแยกตาม Concept:

```cpp
#include <concepts>
#include <iostream>
#include <string>
#include <vector>

// concept สำหรับ "อะไรก็ตามที่มี .size() แล้วคืนค่าที่แปลงเป็น size_t ได้"
template <typename T>
concept Sized = requires(const T& c) {
    { c.size() } -> std::convertible_to<std::size_t>;
};

// concept สำหรับ container ที่ iterate ด้วย range-based for ได้ (มี begin/end)
template <typename T>
concept Iterable = requires(T& c) {
    c.begin();
    c.end();
};

template <typename T>
    requires Sized<T> && Iterable<T>
void print_summary(const T& container) {
    std::cout << "จำนวนสมาชิก: " << container.size() << " -> ";
    for (const auto& item : container) {
        std::cout << item << " ";
    }
    std::cout << "\n";
}

// Overload สอง version แยกด้วย concept: เวอร์ชันสำหรับ integral กับ floating point
// เลือก overload ที่ถูกต้องตอน compile-time โดยอัตโนมัติ ไม่ต้องเช็ค type เอง
void report(std::integral auto value) {
    std::cout << "ค่าจำนวนเต็ม: " << value << " (ไม่มีจุดทศนิยม)\n";
}

void report(std::floating_point auto value) {
    std::cout << "ค่าทศนิยม: " << value << "\n";
}

int main() {
    std::vector<int> nums{1, 2, 3, 4};
    print_summary(nums);

    std::string text = "hi";
    print_summary(text);

    report(10);
    report(3.14);

    return 0;
}
```

ผลลัพธ์:

```
จำนวนสมาชิก: 4 -> 1 2 3 4
จำนวนสมาชิก: 2 -> h i
ค่าจำนวนเต็ม: 10 (ไม่มีจุดทศนิยม)
ค่าทศนิยม: 3.14
```

สังเกตว่า `print_summary` ใช้ได้ทั้งกับ `std::vector<int>` และ `std::string` เพราะทั้งคู่ผ่าน
Concept `Sized` (มี `.size()`) และ `Iterable` (มี `.begin()`/`.end()`) — ถ้ามีคนพยายามเรียก
`print_summary` ด้วย Type ที่ไม่มี `.size()` เช่น `int` เพียวๆ Compiler จะปฏิเสธตั้งแต่ขั้นตอน
เลือก Overload พร้อมข้อความ `constraints not satisfied` ทันที ไม่ต้องรอให้เข้าไปถึงบรรทัด
`container.size()` ข้างในฟังก์ชันก่อน

ส่วน `report` แสดงให้เห็นว่า Concept ใช้แยก Overload ของฟังก์ชันได้ด้วย (คล้ายกับที่
`if constexpr` ใน Part 72 ใช้แยกพฤติกรรม**ภายใน**ฟังก์ชันเดียว) วิธีนี้เหมาะเมื่ออยากให้แต่ละ
Type มีฟังก์ชันของตัวเองแยกกันชัดเจน แทนที่จะเขียนฟังก์ชันเดียวที่มีเงื่อนไขซับซ้อนข้างใน

---

## ข้อผิดพลาดที่พบบ่อย (Common Pitfalls)

1. **ลืมว่า Concept แค่ "จำกัด" ไม่ได้ "เปลี่ยนพฤติกรรม"** — Concept ทำหน้าที่บอก Compiler ว่า
   Type ไหนใช้ Template นี้ได้บ้าง แต่ไม่ได้ทำให้โค้ดข้างในฟังก์ชันทำงานถูกต้องขึ้นมาเอง ถ้า
   เขียน Logic ผิดข้างใน Concept ก็ช่วยอะไรไม่ได้
2. **สับสนระหว่าง `requires` Clause กับ `requires` Expression** — `requires SomeConcept<T>`
   (ไม่มีวงเล็บพารามิเตอร์) คือการ "เรียกใช้" Concept ที่มีอยู่แล้ว ส่วน
   `requires(T x) { ... }` (มีวงเล็บพารามิเตอร์และ Block) คือการ "นิยาม" เงื่อนไขใหม่ ทั้งสอง
   ใช้คำว่า `requires` เหมือนกันแต่ทำหน้าที่ต่างกันโดยสิ้นเชิง
3. **คาดหวังว่า Concept จะทำให้ Error สั้นลงเหลือบรรทัดเดียวเสมอ** — อย่างที่เห็นในหัวข้อ 73.5
   Error หลังใช้ Concept อาจยังมีรายละเอียดจาก Library ปนอยู่บ้าง (โดยเฉพาะ Concept มาตรฐานที่
   ประกอบจาก Concept ย่อยหลายชั้น) สิ่งที่ Concept รับประกันคือ **ข้อความสรุปที่ชัดเจนกว่า**
   และ **Instantiation Stack ที่สั้นลง** ไม่ใช่ Error ที่สั้นลงเหลือบรรทัดเดียวเสมอไป
4. **เขียน Concept ที่ตรวจสอบแค่ Syntax แต่ไม่ตรงกับ Semantic ที่ต้องการจริง** — เช่น Concept
   `Printable` ในหัวข้อ 73.3 เช็คแค่ว่า `operator<<` เขียนได้ ไม่ได้เช็คว่าผลลัพธ์ที่พิมพ์ออกมา
   "สื่อความหมาย" จริงหรือไม่ — Concept ตรวจสอบได้แค่ระดับไวยากรณ์/Interface เท่านั้น ไม่ใช่
   ตรวจสอบความถูกต้องเชิงตรรกะของโปรแกรม
5. **ใช้ Abbreviated Function Template (`auto` parameter) แล้วลืมว่าแต่ละ `auto` เป็นคนละ
   Type Parameter กัน** — `void f(auto a, auto b)` ไม่ได้บังคับว่า `a` กับ `b` ต้อง Type
   เดียวกัน ถ้าต้องการบังคับให้ตรงกัน ต้องกลับไปใช้ Syntax แบบเต็ม (`template <typename T>
   void f(T a, T b)`) หรือเพิ่ม `requires std::same_as<decltype(a), decltype(b)>` เอง
6. **นำ Concept ไปแทนที่ `static_assert` แบบเดิมโดยไม่จำเป็น** — ถ้า Template มีอยู่ตัวเดียว
   ไม่มี Overload ให้เลือก และแค่ต้องการเช็คเงื่อนไขง่ายๆ การใช้ `static_assert` ข้างในฟังก์ชัน
   ก็ยังเป็นทางเลือกที่เข้าใจง่ายอยู่ — Concept จะเห็นประโยชน์ชัดเจนที่สุดเมื่อมีหลาย Overload
   ที่ต้องเลือกตาม Type หรือเมื่อต้องการให้ Error เกิดเร็วที่สุดตั้งแต่ขั้นตอน Overload
   Resolution

---

## แบบฝึกหัดท้ายบท

1. เขียน Concept ชื่อ `Hashable` ที่ตรวจสอบว่า Type `T` มี `std::hash<T>` ใช้งานได้
   (คำใบ้: ใช้ `requires(T value) { { std::hash<T>{}(value) } -> std::convertible_to<std::size_t>; }`)
   แล้วเขียน Function Template ที่รับได้เฉพาะ Type ที่ผ่าน Concept นี้

2. คอมไพล์ตัวอย่าง `find_max` เวอร์ชัน "ก่อนใช้ Concept" (หัวข้อ 73.1) ด้วยตัวเอง นับจำนวน
   บรรทัดของ Error ที่ได้ (`g++ ... 2>&1 | wc -l`) แล้วเปรียบเทียบกับเวอร์ชัน "หลังใช้ Concept"
   (หัวข้อ 73.5) เขียนสรุปสั้นๆ ว่าอ่านง่ายขึ้นตรงจุดไหนบ้าง

3. เขียน Concept ชื่อ `Comparable3Way` ที่ตรวจสอบว่า Type รองรับ `operator<=>` (Three-Way
   Comparison ของ C++20) แล้วทดสอบกับ `int` และ `struct` ที่ไม่มี `operator<=>`

4. เขียน Function Template แบบย่อ (Abbreviated Function Template) ชื่อ `clamp_value` ที่รับ
   พารามิเตอร์ 3 ตัว (`value`, `low`, `high`) โดยบังคับด้วย `std::totally_ordered auto` แล้ว
   คืนค่าที่ถูก "หนีบ" ให้อยู่ในช่วง `[low, high]`

5. เขียน Concept ของตัวเองชื่อ `Container` ที่รวม `Sized` และ `Iterable` จากหัวข้อ 73.8 เข้า
   ด้วยกัน (ใช้ `&&`) แล้วนำไปใช้กับฟังก์ชันที่รับ Container อะไรก็ได้แล้วพิมพ์จำนวนสมาชิก

6. เขียน Concept `EqualityComparableWith<T, U>` ของตัวเอง (ไม่ใช้ Concept มาตรฐาน) ที่ตรวจสอบ
   ว่า Object ของ Type `T` เปรียบเทียบ `==` กับ Object ของ Type `U` ได้ แล้วทดสอบกับคู่ Type
   ที่เปรียบเทียบกันได้และคู่ที่เปรียบเทียบกันไม่ได้

### แนวทางเฉลยข้อ 1

```cpp
#include <concepts>
#include <functional>
#include <iostream>
#include <string>

template <typename T>
concept Hashable = requires(T value) {
    { std::hash<T>{}(value) } -> std::convertible_to<std::size_t>;
};

template <Hashable T>
void show_hash(const T& value) {
    std::size_t h = std::hash<T>{}(value);
    std::cout << "hash ของค่า: " << h << "\n";
}

int main() {
    show_hash(42);
    show_hash(std::string("hello"));
    return 0;
}
```

ผลลัพธ์ที่ควรได้ (ค่า hash จริงอาจต่างกันไปในแต่ละเครื่อง เพราะ `std::hash` ไม่ได้การันตี
ค่าคงที่ข้ามแพลตฟอร์ม แต่โปรแกรมต้อง Compile และรันผ่านโดยไม่มี Error):

```
hash ของค่า: 42
hash ของค่า: <ตัวเลข hash ของ "hello">
```

### แนวทางเฉลยข้อ 4

```cpp
#include <concepts>
#include <iostream>

// Abbreviated Function Template: value, low, high ทุกตัวต้อง totally_ordered
// และเป็น type เดียวกัน (เพราะใช้ auto ตัวเดียวกันไม่ได้ — ต้องผูกด้วย requires เพิ่ม)
auto clamp_value(std::totally_ordered auto value,
                  std::totally_ordered auto low,
                  std::totally_ordered auto high) {
    if (value < low) return low;
    if (value > high) return high;
    return value;
}

int main() {
    std::cout << clamp_value(15, 0, 10) << "\n";   // เกินขอบบน -> ได้ 10
    std::cout << clamp_value(-5, 0, 10) << "\n";   // ต่ำกว่าขอบล่าง -> ได้ 0
    std::cout << clamp_value(5, 0, 10) << "\n";    // อยู่ในช่วงพอดี -> ได้ 5
    return 0;
}
```

ผลลัพธ์ที่ควรได้:

```
10
0
5
```

> **ข้อสังเกตเพิ่มเติม**: เพราะ `value`, `low`, `high` แต่ละตัวเป็นคนละ Template Parameter
> กัน (ตามที่เตือนไว้ใน Common Pitfalls ข้อ 5) โค้ดนี้ทำงานได้แม้ผู้เรียกจะส่ง Type ผสมกัน
> เช่น `clamp_value(5, 0.0, 10)` แต่ `if (value < low)` จะยังคอมไพล์ผ่านได้ก็เพราะ Compiler
> แปลง `int` กับ `double` ให้เทียบกันได้อัตโนมัติ — ถ้าต้องการบังคับให้ทั้งสามตัวเป็น Type
> เดียวกันเป๊ะๆ ต้องกลับไปเขียนแบบ `template <std::totally_ordered T> T clamp_value(T value, T low, T high)` แทน

### แนวทางเฉลยข้อ 3

```cpp
#include <compare>
#include <concepts>
#include <iostream>

// concept ตรวจสอบว่า T รองรับ operator<=> (Three-Way Comparison ของ C++20)
// และผลลัพธ์เทียบกับ 0 ได้ (เพื่อยืนยันว่าใช้เป็นการเรียงลำดับได้จริง)
template <typename T>
concept Comparable3Way = requires(const T& a, const T& b) {
    { a <=> b };
};

struct NoSpaceship {
    int value;
};

struct WithSpaceship {
    int value;
    auto operator<=>(const WithSpaceship&) const = default;
};

template <Comparable3Way T>
void check(const T&) {
    std::cout << "Type นี้รองรับ operator<=> ใช้งานได้\n";
}

int main() {
    check(42);                 // int รองรับ operator<=> อยู่แล้วโดยธรรมชาติ
    check(WithSpaceship{1});   // struct ที่ประกาศ operator<=> เอง (default) ก็ผ่าน

    static_assert(Comparable3Way<int>);
    static_assert(!Comparable3Way<NoSpaceship>);   // NoSpaceship ไม่มี operator<=> เลย

    return 0;
}
```

ผลลัพธ์ที่ควรได้:

```
Type นี้รองรับ operator<=> ใช้งานได้
Type นี้รองรับ operator<=> ใช้งานได้
```

สังเกตการใช้ `static_assert(Comparable3Way<int>)` และ `static_assert(!Comparable3Way<NoSpaceship>)`
— นี่คือวิธีทดสอบ Concept ของตัวเองโดยตรงโดยไม่ต้องเรียกฟังก์ชันจริง เพราะ Concept ประเมินผล
เป็นค่า `bool` ที่รู้ผลได้ตั้งแต่ Compile-Time เสมอ ทำให้ตรวจสอบด้วย `static_assert` ได้ทันที
และเป็นวิธีที่แนะนำมากในการเขียน Unit Test ให้กับ Concept ที่เขียนขึ้นเอง ก่อนจะนำไปใช้งานจริง

---

## สรุปท้ายบท

ใน Part นี้เราได้:

- เห็นปัญหาจริงของ Template แบบเดิมด้วยตาตัวเอง: Error Message ยาวหลายสิบบรรทัดที่ชี้ไปยัง
  โค้ด Internal ของ Library แทนที่จะชี้ไปยังจุดที่ผู้เขียนโค้ดทำผิดจริง
- เข้าใจว่า Concept คือการประกาศ Constraint ของ Template Parameter และใช้ `requires` Clause
  ผูก Concept เข้ากับ Template ได้หลายตำแหน่ง
- เขียน Concept ของตัวเองได้ทั้งแบบรวม Concept อื่นด้วย `&&`/`||` และแบบใช้ `requires`
  Expression ตรวจสอบ Interface ของ Type
- ใช้ Concept มาตรฐานจาก `<concepts>` เช่น `std::integral`, `std::floating_point`,
  `std::same_as`, `std::totally_ordered`
- เปรียบเทียบ Error Message จริงก่อนและหลังใช้ Concept เห็นชัดเจนว่า Concept ทำให้ Error
  ชี้ตรงจุดกว่า และมี Instantiation Stack ที่สั้นลงมาก
- เขียน Function Template แบบย่อด้วย Abbreviated Function Template Syntax (`std::integral
  auto`, `auto`)

Concepts แก้ปัญหาเรื่อง "การจำกัด Type ของ Template ให้ปลอดภัยและอ่านง่าย" ได้อย่างหมดจด แต่
ยังไม่ได้แตะปัญหาอีกอย่างที่โปรแกรมเมอร์ C++ เจอทุกวัน: การเขียน Algorithm ที่ต้องส่ง
`begin()`/`end()` ซ้ำๆ ทุกครั้ง และการ Chain หลาย Operation เข้าด้วยกันจนโค้ดอ่านยาก — ใน
**Part 74** เราจะเรียนรู้ **C++20 Ranges** ซึ่งแก้ปัญหาทั้งสองข้อนี้ด้วยแนวคิด Pipeline แบบ
Functional Programming

**ต่อไป:** [Part 74 — C++20 Ranges](./part-074-cpp20-ranges.md)
