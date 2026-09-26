# Part 62: STL Iterator แบบเจาะลึก (Step 489–496)

> Module E — Templates, Generic Programming และ STL | Part 62 จาก 125
> Step ที่ครอบคลุมใน Part นี้: Step 489–496
> Part ก่อนหน้า: [Part 61 — std::map, std::set, unordered_map/unordered_set](./part-061-map-set.md) | Part ถัดไป: [Part 63 — STL Algorithm แบบเจาะลึก](./part-063-stl-algorithms.md)

## เป้าหมายการเรียนรู้ (Learning Objectives)

หลังจบ Part นี้ ผู้เรียนจะสามารถ:

1. อธิบายได้ว่า **Iterator** คืออะไร และทำไมมันคือแนวคิดที่ generalize เรื่อง Pointer
   ให้ใช้กับ container ทุกชนิดของ STL ได้ในรูปแบบเดียวกัน
2. จำแนก Iterator ทั้ง 5 หมวด (Input, Output, Forward, Bidirectional, Random Access)
   และบอกได้ว่า container แต่ละชนิดใน STL ให้ Iterator หมวดไหน
3. ใช้ `begin()`, `end()`, `cbegin()`, `cend()`, `rbegin()`, `rend()`, `crbegin()`, `crend()`
   ได้อย่างถูกต้องและเข้าใจความแตกต่างของแต่ละตัว
4. ใช้ `std::advance`, `std::distance`, `std::next`, `std::prev` เพื่อเลื่อน iterator
   แบบ generic โดยไม่ต้องสนใจว่า container เบื้องหลังเป็นอะไร
5. อธิบายได้ว่า Range-based for loop ที่ใช้มาตั้งแต่ Part 5 นั้น compiler แปลง
   (desugar) เป็นโค้ดอะไรเบื้องหลัง
6. เขียน **Custom Iterator** สำหรับ class ที่เขียนขึ้นเอง เพื่อให้ใช้กับ range-based
   for loop ได้เหมือน container มาตรฐานของ STL

---

## 62.1 Iterator คืออะไร: Generalize แนวคิด Pointer (Step 489)

ย้อนกลับไปที่ Part 8-9 เราเรียนรู้ว่า **Pointer** ใน C สามารถใช้ "เดิน" ไปตาม array
ทีละช่องได้ ด้วยการทำ Pointer Arithmetic (`p++`, `*p`) ปัญหาคือเทคนิคนี้ใช้ได้เฉพาะ
กับข้อมูลที่เก็บแบบ **ต่อเนื่องกันในหน่วยความจำ (Contiguous Memory)** เท่านั้น เช่น
array หรือ `std::vector`

แต่ container บางชนิด เช่น `std::list` (Doubly Linked List) หรือ `std::map`
(โดยทั่วไป implement ด้วย Red-Black Tree) ไม่ได้เก็บข้อมูลต่อเนื่องกันในหน่วยความจำเลย
แต่ละ node กระจัดกระจายอยู่คนละที่ เชื่อมกันด้วย pointer ข้างใน ถ้าใช้ Pointer
Arithmetic แบบเดิม (`p + 1`) กับ node ของ linked list จะไม่มีทางเดินไปยัง node ถัดไป
ได้อย่างถูกต้อง เพราะตำแหน่งของ node ถัดไปในหน่วยความจำไม่สัมพันธ์กับตำแหน่งปัจจุบันเลย

**Iterator** คือคำตอบของปัญหานี้ มันคือ **object ที่ทำตัวเหมือน pointer** (overload
`operator*`, `operator++`, `operator!=` ฯลฯ ตามที่เราเรียนใน Part 51) แต่ภายในมัน
"รู้" วิธีเดินไปยังสมาชิกถัดไปอย่างถูกต้องตามโครงสร้างข้อมูลจริงของ container นั้นๆ
ผู้ใช้งานภายนอกไม่จำเป็นต้องรู้เลยว่าเบื้องหลังเป็น array, linked list, หรือ tree
แค่เรียก `++it` และ `*it` เหมือนกันหมดทุก container — นี่คือหัวใจของ **Generic
Programming** ที่ STL ใช้เชื่อม Container เข้ากับ Algorithm (Part 63)

```cpp
#include <cstdio>
#include <vector>

int main(void) {
    // แบบที่ 1: ใช้ pointer วนอ่าน array ธรรมดา (แบบที่เคยเรียนใน Part 8-9)
    int raw[5] = {10, 20, 30, 40, 50};
    int* p = raw;              // p ชี้ไปที่ raw[0]
    while (p != raw + 5) {     // raw + 5 คือตำแหน่ง "หลังตัวสุดท้าย" (one-past-the-end)
        std::printf("%d ", *p);
        ++p;
    }
    std::printf("\n");

    // แบบที่ 2: ใช้ iterator วนอ่าน std::vector ด้วยแนวคิดเดียวกันทุกประการ
    std::vector<int> v = {10, 20, 30, 40, 50};
    std::vector<int>::iterator it = v.begin();
    while (it != v.end()) {    // v.end() คือ iterator ที่ชี้ "หลังตัวสุดท้าย" เช่นกัน
        std::printf("%d ", *it);
        ++it;
    }
    std::printf("\n");

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 iterator_intro.cpp -o iterator_intro
./iterator_intro
# 10 20 30 40 50
# 10 20 30 40 50
```

สังเกตว่าโครงสร้างโค้ดทั้งสองแบบ **เหมือนกันทุกประการ** ต่างกันแค่ชนิดของตัวแปร
(`int*` กับ `std::vector<int>::iterator`) นี่ไม่ใช่เรื่องบังเอิญ — คนออกแบบ STL
(Alexander Stepanov) ตั้งใจออกแบบ Iterator ให้มี **Interface เหมือน Pointer** ทุก
ประการ (`*`, `++`, `!=`, `==`) เพื่อให้โค้ดที่เขียนสำหรับ pointer ธรรมดาสามารถ
ทำงานกับ Iterator ของ container ใดๆ ก็ได้แทบไม่ต้องแก้อะไรเลย และในความเป็นจริง
`int*` (raw pointer) ก็ถือเป็น Iterator ชนิดหนึ่งของ STL ด้วยเช่นกัน (เป็น Random
Access Iterator ที่ "แข็งแกร่ง" ที่สุด) — ตัวอย่างเช่น `std::sort(raw, raw + 5)`
สามารถเรียกใช้กับ raw array ได้ตรงๆ โดยไม่ต้องแปลงเป็น container ก่อนเลย

> **สรุปแนวคิด**: Iterator = "Pointer เวอร์ชัน Generalize" ที่ใช้ interface เดียวกัน
> ในการเข้าถึงข้อมูล ไม่ว่าโครงสร้างข้อมูลเบื้องหลังจะเป็นอะไร

---

## 62.2 Iterator Category ทั้ง 5 แบบ (Step 490)

ไม่ใช่ Iterator ทุกตัวจะมีความสามารถเท่ากัน เพราะโครงสร้างข้อมูลเบื้องหลังต่างกัน
เช่น `std::vector` เก็บข้อมูลต่อเนื่องกันจึงกระโดดไปตำแหน่งใดก็ได้ทันที แต่
`std::forward_list` เป็น Singly Linked List จึงเดินได้แค่ทิศทางเดียว มาตรฐาน C++
จึงแบ่ง Iterator ออกเป็น **5 หมวด (Category)** เรียงจากความสามารถน้อยไปมาก โดยหมวด
ที่มีความสามารถสูงกว่าจะรองรับทุกอย่างที่หมวดต่ำกว่าทำได้ด้วย (ยกเว้น Output
Iterator ที่แยกออกไปต่างหาก เพราะเน้นการเขียนอย่างเดียว):

| Category | ความสามารถ | ตัวดำเนินการที่ใช้ได้ | ตัวอย่าง |
|---|---|---|---|
| **Input Iterator** | อ่านค่าได้ครั้งเดียว เดินหน้าทางเดียว (single-pass) | `*it` (อ่านอย่างเดียว), `++it`, `==`, `!=` | `std::istream_iterator` |
| **Output Iterator** | เขียนค่าได้ครั้งเดียว เดินหน้าทางเดียว (single-pass) | `*it = value`, `++it` | `std::ostream_iterator`, `std::back_insert_iterator` |
| **Forward Iterator** | อ่าน/เขียนได้ เดินหน้าทางเดียว แต่ **เดินซ้ำได้หลายรอบ** (multi-pass) | ทุกอย่างของ Input/Output + คัดลอก iterator แล้วใช้ซ้ำได้ | `std::forward_list::iterator`, `std::unordered_map::iterator` |
| **Bidirectional Iterator** | เหมือน Forward แต่เดิน**ถอยหลังได้ด้วย** | ทุกอย่างของ Forward + `--it` | `std::list::iterator`, `std::map::iterator`, `std::set::iterator` |
| **Random Access Iterator** | เข้าถึงตำแหน่งใดก็ได้ทันที O(1) | ทุกอย่างของ Bidirectional + `it + n`, `it - n`, `it[n]`, `it1 - it2`, `<`, `>`, `<=`, `>=` | `std::vector::iterator`, `std::deque::iterator`, `std::array::iterator`, raw pointer |

ความสัมพันธ์แบบลำดับชั้น (hierarchy) มองเป็นภาพได้ดังนี้:

```
Input Iterator ──┐
                  ├──> Forward Iterator ──> Bidirectional Iterator ──> Random Access Iterator
Output Iterator ──┘
```

ลองมาดูตัวอย่างที่แสดงความสามารถของแต่ละ Category จริงๆ ผ่านโค้ด:

```cpp
#include <cstdio>
#include <vector>
#include <list>
#include <forward_list>
#include <iterator>
#include <type_traits>

template <typename Iterator>
void print_category() {
    using Category = typename std::iterator_traits<Iterator>::iterator_category;
    if constexpr (std::is_same_v<Category, std::random_access_iterator_tag>) {
        std::printf("Random Access Iterator\n");
    } else if constexpr (std::is_same_v<Category, std::bidirectional_iterator_tag>) {
        std::printf("Bidirectional Iterator\n");
    } else if constexpr (std::is_same_v<Category, std::forward_iterator_tag>) {
        std::printf("Forward Iterator\n");
    } else if constexpr (std::is_same_v<Category, std::input_iterator_tag>) {
        std::printf("Input Iterator\n");
    } else {
        std::printf("Output/Unknown Iterator\n");
    }
}

int main(void) {
    std::printf("vector<int>::iterator        -> ");
    print_category<std::vector<int>::iterator>();

    std::printf("list<int>::iterator          -> ");
    print_category<std::list<int>::iterator>();

    std::printf("forward_list<int>::iterator  -> ");
    print_category<std::forward_list<int>::iterator>();

    // ตัวอย่างความสามารถของ random access iterator (มีเฉพาะ vector/deque/array)
    std::vector<int> v = {1, 2, 3, 4, 5};
    auto it = v.begin();
    it += 2;                       // กระโดดข้ามได้โดยตรง O(1)
    std::printf("it += 2 -> %d\n", *it);
    std::printf("it[1]   -> %d\n", it[1]);        // random access เท่านั้นที่ใช้ [] ได้
    std::printf("v.end() - v.begin() = %td\n", v.end() - v.begin()); // ลบ iterator กันได้

    // bidirectional iterator (list) เดินถอยหลังได้ แต่กระโดดทีเดียวไม่ได้
    std::list<int> lst = {1, 2, 3, 4, 5};
    auto lit = lst.end();
    --lit;                          // เดินถอยหลังทีละก้าว
    std::printf("last of list = %d\n", *lit);

    // forward iterator (forward_list) เดินหน้าได้ทางเดียว ไม่มี -- ให้ใช้
    std::forward_list<int> flst = {1, 2, 3};
    auto fit = flst.begin();
    ++fit;
    std::printf("second of forward_list = %d\n", *fit);

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 categories.cpp -o categories
./categories
```

ผลลัพธ์:

```
vector<int>::iterator        -> Random Access Iterator
list<int>::iterator          -> Bidirectional Iterator
forward_list<int>::iterator  -> Forward Iterator
it += 2 -> 3
it[1]   -> 4
v.end() - v.begin() = 5
last of list = 5
second of forward_list = 2
```

โค้ดนี้ใช้ `std::iterator_traits<Iterator>::iterator_category` ซึ่งเป็นกลไกที่ STL
ใช้ **สอบถามความสามารถ** ของ Iterator แต่ละตัวที่ compile-time ผ่าน template และ
`if constexpr` (Part 72 จะพูดถึง `if constexpr` แบบละเอียด) — เบื้องหลังของ
`std::sort`, `std::advance` และ algorithm อื่นๆ ใน Part 63 ก็ใช้กลไกนี้เพื่อเลือก
implementation ที่เร็วที่สุดสำหรับ Iterator category ที่ได้รับมา

ต่อมาคือ **Input Iterator** และ **Output Iterator** ที่มีข้อจำกัดพิเศษคือ
**single-pass** (เดินได้รอบเดียว อ่านซ้ำไม่ได้) ตัวอย่างที่ชัดที่สุดคือการอ่าน/เขียน
stream:

```cpp
#include <cstdio>
#include <sstream>
#include <iterator>
#include <vector>
#include <algorithm>

int main(void) {
    // Input iterator: อ่านค่าได้ครั้งเดียวจากลำดับ (single-pass), อ่านซ้ำไม่ได้
    std::istringstream iss("10 20 30 40");
    std::istream_iterator<int> in_it(iss);
    std::istream_iterator<int> in_end;      // default-constructed = end-of-stream

    std::vector<int> values;
    while (in_it != in_end) {
        values.push_back(*in_it);
        ++in_it;
    }
    for (int x : values) {
        std::printf("%d ", x);
    }
    std::printf("\n");

    // Output iterator: เขียนได้อย่างเดียว ครั้งเดียว เดินหน้าทางเดียว (single-pass)
    std::ostringstream oss;
    std::ostream_iterator<int> out_it(oss, ",");
    std::copy(values.begin(), values.end(), out_it);
    std::printf("%s\n", oss.str().c_str());

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 input_output_iter.cpp -o input_output_iter
./input_output_iter
# 10 20 30 40
# 10,20,30,40,
```

`std::istream_iterator<int>` อ่านตัวเลขจาก stream ทีละตัว ทุกครั้งที่ `++it`
มันจะอ่านค่าถัดไปจาก stream จริงๆ (เดินหน้าเปลี่ยนสถานะของ stream ตลอดเวลา) จะย้อน
กลับไปอ่านค่าเดิมซ้ำไม่ได้ ส่วน `std::ostream_iterator<int>` ก็เช่นกัน แต่ทำหน้าที่
เขียนแทนที่จะอ่าน — เมื่อ `*out_it = value` มันจะพิมพ์ค่าและ delimiter (`,`)
ออกไปที่ stream ทันที

---

## 62.3 ตาราง Container กับ Iterator Category ที่รองรับ (Step 491)

หัวข้อนี้จะสรุป Iterator Category ของทุก container ที่เรียนมาตั้งแต่ Part 58-61
ไว้ในตารางเดียว เพื่อให้เห็นภาพรวมและใช้เป็นตารางอ้างอิงได้ตลอดหลักสูตร:

| Container | Iterator Category | เหตุผล |
|---|---|---|
| `std::vector` | Random Access | เก็บข้อมูลต่อเนื่องในหน่วยความจำ (contiguous array) |
| `std::array` | Random Access | เก็บข้อมูลต่อเนื่องเหมือน raw array ตายตัว |
| `std::deque` | Random Access | เก็บเป็น "chunk" ต่อเนื่อง แต่มี index mapping ให้เข้าถึง O(1) ได้ |
| `std::string` | Random Access | ภายในเป็น contiguous character array เหมือน `vector<char>` |
| `std::list` | Bidirectional | Doubly Linked List — เดินหน้า/ถอยหลังได้ แต่กระโดดข้ามไม่ได้ |
| `std::map` / `std::multimap` | Bidirectional | ภายในเป็น Balanced Binary Search Tree (มักเป็น Red-Black Tree) |
| `std::set` / `std::multiset` | Bidirectional | เช่นเดียวกับ `map` เพราะโครงสร้างต้นไม้แบบเดียวกัน |
| `std::forward_list` | Forward | Singly Linked List — เดินได้ทิศทางเดียวเท่านั้น |
| `std::unordered_map` / `unordered_set` (และ multi-) | Forward | ภายในเป็น Hash Table ที่แต่ละ bucket เป็น linked list เดินได้ทางเดียว |
| `std::stack`, `std::queue`, `std::priority_queue` | **ไม่มี iterator เลย** | เป็น **Container Adapter** ออกแบบมาให้เข้าถึงได้เฉพาะปลายเท่านั้น (Part 60) |
| Raw pointer / raw array | Random Access (Contiguous Iterator ใน C++20) | ต่อเนื่องในหน่วยความจำโดยธรรมชาติ |

จุดที่มือใหม่มักงงคือ **`std::unordered_map` ทำไมเข้าถึงข้อมูลด้วย key ได้แบบ O(1)
เฉลี่ย แต่ iterator กลับเป็นแค่ Forward ไม่ใช่ Random Access?** คำตอบคือ
"เข้าถึงด้วย key" (ผ่าน hash function) กับ "เดิน iterator ไปตามลำดับสมาชิก" เป็น
คนละเรื่องกัน การเข้าถึงด้วย key ใช้ hash function คำนวณตำแหน่ง bucket โดยตรง
แต่การ "เดิน" ไปทีละตัวต้องไล่ผ่านทุก bucket และทุก node ในแต่ละ bucket
ตามลำดับที่ภายในจัดเก็บไว้ (ซึ่งไม่มีความหมายเชิงตำแหน่งเชิงตัวเลขให้กระโดดข้ามได้)
จึงทำได้แค่ `++it` ทีละก้าวเท่านั้น

Iterator Category ยังส่งผลต่อ **ความซับซ้อนของเวลา (Time Complexity)** ของฟังก์ชัน
generic อย่าง `std::advance` ด้วย — ลองดูตัวอย่าง:

```cpp
#include <cstdio>
#include <vector>
#include <forward_list>
#include <iterator>

int main(void) {
    std::vector<int> v = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10};
    auto vit = v.begin();
    std::advance(vit, 5);      // vector: random access -> กระโดดทีเดียว O(1)
    std::printf("vector advance(5): %d\n", *vit);

    std::forward_list<int> fl = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10};
    auto fit = fl.begin();
    std::advance(fit, 5);      // forward_list: forward เท่านั้น -> เดินทีละก้าว O(n)
    std::printf("forward_list advance(5): %d\n", *fit);

    auto next_it = std::next(v.begin(), 3);
    std::printf("next: %d\n", *next_it);

    auto prev_it = std::prev(v.end(), 2);
    std::printf("prev: %d\n", *prev_it);

    auto dist = std::distance(v.begin(), v.end());
    std::printf("distance: %td\n", dist);

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 advance_distance.cpp -o advance_distance
./advance_distance
```

ผลลัพธ์:

```
vector advance(5): 6
forward_list advance(5): 6
next: 4
prev: 9
distance: 10
```

`std::advance(it, n)` เรียก `it += n` ตรงๆ ถ้า Iterator เป็น Random Access
(ทำได้ใน O(1)) แต่ถ้าเป็น Forward หรือ Bidirectional จะเรียก `++it` วนซ้ำ `n` รอบ
แทน (ทำได้ใน O(n)) — โค้ดที่เราเขียนเรียก `std::advance` เหมือนกันทุกประการ แต่
compiler จะเลือก implementation ที่เหมาะสมที่สุดให้เองผ่านกลไก `iterator_traits`
ที่เห็นในหัวข้อก่อนหน้า นี่คือประโยชน์สำคัญของการแบ่ง Category: เขียนโค้ด generic
ได้ครั้งเดียว โดยที่ประสิทธิภาพยังคงเหมาะสมกับแต่ละ container โดยอัตโนมัติ

`std::next(it, n)` และ `std::prev(it, n)` ทำงานคล้าย `std::advance` แต่คืนค่า
iterator ตัวใหม่กลับมาแทนที่จะแก้ไข iterator เดิม (ใช้บ่อยเวลาที่ไม่ต้องการแก้ไข
iterator ตัวต้นฉบับ) ส่วน `std::distance(first, last)` คำนวณ "ระยะห่าง" ระหว่าง
iterator สองตัว คืนค่าเป็น `ptrdiff_t` (แสดงผลด้วย `%td`) — สำหรับ Random Access
Iterator คำนวณด้วยการลบตรงๆ O(1) แต่สำหรับ Iterator category อื่นต้องเดินนับ
ทีละก้าว O(n)

---

## 62.4 begin() / end() / cbegin() / cend() (Step 492)

ทุก container ของ STL มีฟังก์ชัน `begin()` และ `end()` เป็นคู่เสมอ:

- **`begin()`** คืน iterator ที่ชี้ไปยัง **สมาชิกตัวแรก**
- **`end()`** คืน iterator ที่ชี้ไปยัง **ตำแหน่งถัดจากสมาชิกตัวสุดท้าย** (one-past-
  the-end) **ไม่ใช่ตัวสุดท้าย** — การ dereference `*end()` เป็น Undefined
  Behavior เสมอ ห้ามทำเด็ดขาด

การออกแบบให้ `end()` ชี้ "เลยตัวสุดท้ายไปหนึ่งช่อง" (ไม่ใช่ชี้ตัวสุดท้ายพอดี)
ทำให้เขียนเงื่อนไขวนซ้ำ `it != end()` ได้ง่ายและใช้ได้กับ container ว่างเปล่าด้วย
(ถ้า container ว่าง `begin() == end()` ทันที ลูปจะไม่ทำงานเลยแม้แต่รอบเดียว
โดยไม่ต้องเช็ค edge case พิเศษ)

นอกจาก `begin()`/`end()` แล้วยังมีคู่ `cbegin()`/`cend()` ที่คืน **`const_iterator`
เสมอ** ไม่ว่า container ต้นทางจะเป็น `const` หรือไม่ก็ตาม (ตัว `c` ย่อมาจาก
"constant") ต่างจาก `begin()`/`end()` ที่จะคืน `const_iterator` ก็ต่อเมื่อเรียก
จาก container ที่เป็น `const` เท่านั้น:

```cpp
#include <cstdio>
#include <vector>

void print_vector(const std::vector<int>& v) {
    // container เป็น const จึงต้องใช้ const_iterator (begin() บน const object จะคืน const_iterator เองอัตโนมัติ)
    for (std::vector<int>::const_iterator it = v.begin(); it != v.end(); ++it) {
        std::printf("%d ", *it);
    }
    std::printf("\n");
}

int main(void) {
    std::vector<int> v = {1, 2, 3, 4, 5};

    // cbegin()/cend() คืน const_iterator เสมอ ไม่ว่า container ต้นทางจะเป็น const หรือไม่
    for (auto it = v.cbegin(); it != v.cend(); ++it) {
        // *it = 99; // ถ้าเปิดบรรทัดนี้จะ compile error ทันที เพราะแก้ไขผ่าน const_iterator ไม่ได้
        std::printf("%d ", *it);
    }
    std::printf("\n");

    print_vector(v);

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 cbegin_cend.cpp -o cbegin_cend
./cbegin_cend
# 1 2 3 4 5
# 1 2 3 4 5
```

**ทำไมต้องมี `cbegin()`/`cend()` ทั้งที่ `begin()`/`end()` บน `const` object ก็คืน
`const_iterator` อยู่แล้ว?** เพราะบางครั้งเรามี container ที่**ไม่ใช่** `const`
(เช่นตัวแปร `v` ใน `main()` ด้านบน) แต่**ตั้งใจ**จะแค่อ่านค่าอย่างเดียวไม่แก้ไข
การเรียก `cbegin()`/`cend()` ตรงๆ ทำให้ **สื่อเจตนา (Intent)** ในโค้ดได้ชัดเจนว่า
"ส่วนนี้จะไม่มีการแก้ไขข้อมูลแน่นอน" ซึ่งเป็นหลักการ **const-correctness** ที่ดี
(สอดคล้องกับที่เรียนเรื่อง `const` ใน Part 43) และยังช่วยให้ compiler ช่วยตรวจจับ
บั๊กจากการแก้ไขข้อมูลโดยไม่ตั้งใจได้ตั้งแต่ compile-time

---

## 62.5 rbegin() / rend(): การวนซ้ำแบบย้อนกลับ (Step 493)

ทุก container ที่รองรับ Bidirectional Iterator ขึ้นไป (Bidirectional และ Random
Access) จะมีฟังก์ชัน `rbegin()`/`rend()` ให้เพิ่มเติม เพื่อ **วนซ้ำจากท้ายไปหน้า**
โดยไม่ต้องเขียนลูปกลับด้านเอง:

```cpp
#include <cstdio>
#include <vector>

int main(void) {
    std::vector<int> v = {1, 2, 3, 4, 5};

    std::printf("เดินหน้า:          ");
    for (auto it = v.begin(); it != v.end(); ++it) {
        std::printf("%d ", *it);
    }
    std::printf("\n");

    std::printf("ถอยหลัง:           ");
    for (auto rit = v.rbegin(); rit != v.rend(); ++rit) {
        std::printf("%d ", *rit);
    }
    std::printf("\n");

    std::printf("ถอยหลังแบบ const:  ");
    for (auto rit = v.crbegin(); rit != v.crend(); ++rit) {
        std::printf("%d ", *rit);
    }
    std::printf("\n");

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 reverse_iter.cpp -o reverse_iter
./reverse_iter
```

ผลลัพธ์:

```
เดินหน้า:          1 2 3 4 5
ถอยหลัง:           5 4 3 2 1
ถอยหลังแบบ const:  5 4 3 2 1
```

`v.rbegin()` คืน `reverse_iterator` ที่ชี้ไปยัง **สมาชิกตัวสุดท้าย** และ `v.rend()`
คือตำแหน่ง "ก่อนตัวแรก" (before-the-beginning) เวลาเรียก `++rit` บน
`reverse_iterator` จริงๆ แล้วภายในมันจะ **เดินถอยหลัง** บน iterator ปกติแทน — นี่คือ
เหตุผลที่ `reverse_iterator` ต้องอาศัย iterator ที่รองรับอย่างน้อย Bidirectional
(ต้องมี `--it` ให้ใช้) `std::forward_list` จึงไม่มี `rbegin()`/`rend()` ให้เรียก
เพราะ Iterator ของมันเดินถอยหลังไม่ได้เลย

ส่วน `crbegin()`/`crend()` ก็เป็นเวอร์ชัน `const` ของ `reverse_iterator`
เหมือนกับความสัมพันธ์ระหว่าง `cbegin()`/`cend()` กับ `begin()`/`end()` ทุกประการ

> **หมายเหตุขั้นสูง**: ถ้าต้องแปลง `reverse_iterator` กลับเป็น iterator ปกติ (เช่น
> เพื่อใช้กับ `erase()`) ต้องเรียก `.base()` แล้วมักต้องปรับตำแหน่งถอยกลับ 1 ช่อง
> ด้วย เพราะ `reverse_iterator::base()` จะคืน iterator ที่ชี้ไปยังตำแหน่ง **ถัดไป**
> จากตำแหน่งที่ `reverse_iterator` กำลังชี้อยู่จริง (ผลจากการออกแบบภายในของ
> `reverse_iterator`) รายละเอียดนี้จะกลับมาเจออีกครั้งเมื่อใช้ `erase()` ร่วมกับ
> reverse iterator ในโค้ดขั้นสูง

---

## 62.6 Range-based for Loop ทำงานอย่างไรเบื้องหลัง (Step 494)

ตั้งแต่ Part 5 เราใช้ range-based for loop กันมาโดยตลอด:

```
for (int x : v) {
    std::printf("%d ", x);
}
```

โค้ดนี้ดู "มายากล" เพราะไม่ต้องเขียน iterator เองเลย แต่ในความเป็นจริง compiler
จะ **desugar** (แปลง) โค้ดนี้เป็นโค้ดที่ใช้ `begin()`/`end()` ตรงๆ ตั้งแต่ C++11
เป็นต้นมา ลองเขียนโค้ดที่แปลงมือเองเพื่อดูให้เห็นภาพ:

```cpp
#include <cstdio>
#include <vector>

int main(void) {
    std::vector<int> v = {1, 2, 3, 4, 5};

    // (1) โค้ดที่เราเขียน
    for (int x : v) {
        std::printf("%d ", x);
    }
    std::printf("\n");

    // (2) สิ่งที่ compiler แปลงให้เบื้องหลัง (เขียนเลียนแบบเพื่อความเข้าใจ)
    {
        auto&& range = v;                 // ผูก reference กับ range-expression (v)
        auto it = range.begin();          // เรียก begin() จริงๆ
        auto end_it = range.end();        // เรียก end() จริงๆ
        for (; it != end_it; ++it) {
            int x = *it;                  // dereference ทุกรอบ
            std::printf("%d ", x);
        }
        std::printf("\n");
    }

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 range_for_desugar.cpp -o range_for_desugar
./range_for_desugar
# 1 2 3 4 5
# 1 2 3 4 5
```

รูปแบบที่มาตรฐาน C++ ระบุไว้จริงๆ (ในเชิงแนวคิด) มีลักษณะประมาณนี้:

```
{
    auto&& __range = range_expression;
    auto __begin = begin_expr(__range);   // เรียก __range.begin() หรือ begin(__range) ผ่าน ADL
    auto __end   = end_expr(__range);     // เรียก __range.end() หรือ end(__range) ผ่าน ADL
    for ( ; __begin != __end; ++__begin) {
        range_declaration = *__begin;     // เช่น int x = *__begin;
        loop_statement
    }
}
```

(บล็อกนี้เป็นภาพเชิงแนวคิดที่เขียนตามคำอธิบายในมาตรฐาน ไม่ใช่โค้ดที่คอมไพล์ได้จริง
เพราะ `range_expression`, `begin_expr`, `range_declaration`, `loop_statement`
เป็นแค่ชื่อ placeholder ที่แทนส่วนต่างๆ ของโค้ดจริงที่เราเขียน)

ตัวแปรที่ compiler สร้างขึ้น (`__range`, `__begin`, `__end`) เป็นชื่อที่ไม่ชนกับ
โค้ดของเรา (compiler สงวนชื่อขึ้นต้นด้วย `__` ไว้ใช้ภายในเอง) จุดสำคัญที่ต้องเข้าใจ
คือ **range-based for loop ไม่ใช่ฟีเจอร์วิเศษของภาษา** แต่เป็นแค่ **Syntactic
Sugar** (น้ำตาลทางไวยากรณ์ — โค้ดที่เขียนสั้นลงแต่ไม่มีความสามารถเพิ่มขึ้นจากที่
เขียนเองได้อยู่แล้ว) ที่เรียก `begin()`/`end()` ให้อัตโนมัติเท่านั้น เหตุผลที่มัน
ทำงานได้กับทั้ง `std::vector`, `std::list`, `std::map`, raw array และ container
ที่เราเขียนขึ้นเอง (หัวข้อถัดไป) ก็เพราะ **ทุกอย่างที่มี `begin()`/`end()` ที่
คืน iterator รองรับ `!=`, `++`, `*` ได้ ก็ใช้กับ range-based for ได้ทันที**
โดยไม่จำเป็นต้องเป็น Iterator category สูงๆ เลยด้วยซ้ำ — แค่ Input/Forward ก็พอ

สำหรับ raw array การเรียก `begin(arr)`/`end(arr)` จะ resolve ไปที่ฟังก์ชัน
`std::begin`/`std::end` แบบ free function (ไม่ใช่ member function เพราะ array
ไม่มี method) ซึ่งคำนวณจาก `arr` และ `arr + N` ตรงๆ นี่คือเหตุผลที่โค้ดแบบ
`for (int x : raw_array)` ใช้งานได้เช่นกัน

---

## 62.7 เขียน Custom Iterator ของตัวเอง — ตอนที่ 1: จากศูนย์ (Step 495)

ตอนนี้เราเข้าใจแล้วว่า range-based for loop ต้องการแค่ `begin()`/`end()` ที่คืน
object ซึ่งรองรับ `*`, `++`, `!=` ก็เพียงพอ ในหัวข้อนี้เราจะเขียน class ของตัวเอง
ที่มี **custom iterator** เพื่อให้ใช้กับ range-based for ได้เหมือน container
มาตรฐานของ STL ทุกประการ

ตัวอย่างแรก: class `Countdown` ที่ generate ลำดับตัวเลขนับถอยหลังจาก `start` ลงไป
จนถึง 0 (ไม่รวม 0) โดยไม่ต้องเก็บข้อมูลไว้ในหน่วยความจำเลยแม้แต่ตัวเดียว
(สร้างค่าขึ้นมา "แบบ lazy" ทีละตัวตอนที่ iterator เดินไปถึง — concept นี้จะกลับมา
เจออีกครั้งอย่างเข้มข้นใน Part 74 เรื่อง C++20 Ranges):

```cpp
#include <cstdio>
#include <iterator>
#include <cstddef>

// class ของเราเอง: สร้าง sequence นับถอยหลังจาก start ลงไปจนถึง 0 (ไม่รวม 0)
class Countdown {
public:
    explicit Countdown(int start) : start_(start) {}

    class Iterator {
    public:
        // nested typedef ที่จำเป็นสำหรับให้ std::iterator_traits และ algorithm ต่างๆ ใช้งานได้
        using iterator_category = std::forward_iterator_tag;
        using value_type        = int;
        using difference_type   = std::ptrdiff_t;
        using pointer           = const int*;
        using reference         = const int&;

        Iterator() : value_(0) {}                 // forward iterator ต้อง default-constructible ได้
        explicit Iterator(int value) : value_(value) {}

        reference operator*() const { return value_; }
        pointer operator->() const { return &value_; }

        Iterator& operator++() {                   // pre-increment
            --value_;
            return *this;
        }
        Iterator operator++(int) {                  // post-increment
            Iterator tmp = *this;
            --value_;
            return tmp;
        }

        bool operator==(const Iterator& other) const { return value_ == other.value_; }
        bool operator!=(const Iterator& other) const { return !(*this == other); }

    private:
        int value_;
    };

    Iterator begin() const { return Iterator(start_); }
    Iterator end()   const { return Iterator(0); }   // จุดสิ้นสุดคือ 0 (exclusive)

private:
    int start_;
};

int main(void) {
    Countdown cd(5);

    // เพราะเรามี begin()/end() ที่คืน iterator ที่รองรับ != , ++ , * ครบ
    // range-based for loop จึงใช้กับ Countdown ได้ทันที ไม่ต่างจาก std::vector เลย
    for (int x : cd) {
        std::printf("%d ", x);
    }
    std::printf("Liftoff!\n");

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 countdown.cpp -o countdown
./countdown
# 5 4 3 2 1 Liftoff!
```

มาดูส่วนประกอบสำคัญของ custom iterator ทีละส่วน:

1. **Nested typedef 5 ตัว** (`iterator_category`, `value_type`, `difference_type`,
   `pointer`, `reference`) — นี่คือ "สัญญา" ที่บอกให้ `std::iterator_traits`
   (และ algorithm ต่างๆ ใน Part 63) รู้จักคุณสมบัติของ iterator ตัวนี้ ถ้าขาด
   ส่วนนี้ไป iterator ของเราจะใช้กับ `std::iterator_traits` หรือ generic algorithm
   บางตัวไม่ได้ (แม้ range-based for จะยังใช้งานได้อยู่ก็ตาม เพราะมันต้องการแค่
   `*`, `++`, `!=`)
2. **`operator*()`** — คืนค่าปัจจุบันที่ iterator ชี้อยู่ (ในที่นี้คืนแบบ
   read-only ผ่าน `const int&` เพราะค่าถูก generate ขึ้นมาสดๆ ไม่มีที่เก็บถาวร
   ให้แก้ไขได้จริง)
3. **`operator++()`** (pre-increment) และ **`operator++(int)`** (post-increment,
   สังเกต parameter `int` หลอกที่ไม่ได้ใช้จริง เป็น convention ของภาษาที่กำหนดไว้
   ตั้งแต่ Part 51 เรื่อง Operator Overloading) — ทำหน้าที่เดินไปยังค่าถัดไป
4. **`operator==`/`operator!=`** — ใช้เปรียบเทียบว่า iterator สองตัวชี้ตำแหน่ง
   เดียวกันหรือไม่ ซึ่งเป็นเงื่อนไขหยุดของลูป (`it != end()`)
5. **Default constructor** (`Iterator()`) — Forward Iterator ตามมาตรฐาน C++
   ต้อง **DefaultConstructible** ได้ (สร้างโดยไม่ระบุค่าเริ่มต้นได้) เป็นหนึ่งใน
   ข้อกำหนดของ Forward Iterator concept

---

## 62.8 เขียน Custom Iterator — ตอนที่ 2: การส่งต่อ Iterator ของ Container ภายใน (Step 496)

ในทางปฏิบัติ class ส่วนใหญ่ที่เขียนขึ้นในโปรเจกต์จริง**ไม่ได้เขียน iterator ขึ้น
ใหม่ทั้งหมดตั้งแต่ศูนย์แบบ `Countdown`** เพราะส่วนใหญ่มักจะห่อหุ้ม (wrap)
container ของ STL ไว้ภายในอยู่แล้ว (เช่น เก็บข้อมูลจริงใน `std::vector` แต่
ต้องการให้คลาสของเรามี interface เฉพาะทางเพิ่มเติม) วิธีที่ง่ายและใช้บ่อยที่สุดคือ
**ส่งต่อ (forward) iterator ของ container ภายในออกไปตรงๆ**:

```cpp
#include <cstdio>
#include <vector>

// แนวทางที่ใช้กันบ่อยที่สุดในโค้ดจริง: ไม่ต้องเขียน iterator class เอง
// แค่ "ส่งต่อ" (forward) iterator ของ container ภายในออกไปให้ผู้ใช้ตรงๆ
class IntBag {
public:
    void add(int value) { data_.push_back(value); }

    std::vector<int>::iterator begin() { return data_.begin(); }
    std::vector<int>::iterator end()   { return data_.end(); }
    std::vector<int>::const_iterator begin() const { return data_.begin(); }
    std::vector<int>::const_iterator end()   const { return data_.end(); }

private:
    std::vector<int> data_;
};

int main(void) {
    IntBag bag;
    bag.add(10);
    bag.add(20);
    bag.add(30);

    for (int x : bag) {
        std::printf("%d ", x);
    }
    std::printf("\n");

    const IntBag& const_bag = bag;
    for (int x : const_bag) {
        std::printf("%d ", x);
    }
    std::printf("\n");

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 intbag.cpp -o intbag
./intbag
# 10 20 30
# 10 20 30
```

สังเกตว่า `IntBag` มี `begin()`/`end()` สองคู่ — คู่แรกเป็น non-`const` member
function คืน `std::vector<int>::iterator` (แก้ไขข้อมูลได้) และคู่ที่สองเป็น
`const` member function คืน `std::vector<int>::const_iterator` (อ่านอย่างเดียว)
นี่คือ **Overload Resolution** ที่เรียนมาตั้งแต่ Part 44: เมื่อเรียก
`bag.begin()` จาก object ที่ไม่ใช่ `const` (`bag`) compiler จะเลือก overload
แรก แต่เมื่อเรียกจาก `const_bag` (ที่เป็น `const IntBag&`) compiler จะเลือก
overload ที่สองให้อัตโนมัติ — สอดคล้องกับหลัก const-correctness ที่พูดถึงใน 62.4

**เมื่อไหร่ควรเขียน custom iterator เองตั้งแต่ศูนย์ (แบบ `Countdown`) เมื่อไหร่ควร
ส่งต่อ iterator ของ container ภายใน (แบบ `IntBag`)?**

| สถานการณ์ | แนวทางที่แนะนำ |
|---|---|
| Class ห่อหุ้ม container ของ STL ไว้ภายใน (เช่น wrapper, adapter) | ส่งต่อ iterator ของ container ภายในตรงๆ (แบบ `IntBag`) |
| Class ไม่มีการเก็บข้อมูลจริง แต่ **generate ค่าตามกฎ** (เช่น sequence, range, view) | เขียน custom iterator เองตั้งแต่ศูนย์ (แบบ `Countdown`) |
| Class เก็บข้อมูลด้วยโครงสร้างพิเศษที่ STL ไม่มีให้ (เช่น Linked List ที่เขียนเอง จาก Part 19) | เขียน custom iterator เอง โดยให้ iterator รู้จักโครงสร้าง node ภายใน |

---

## ข้อผิดพลาดที่พบบ่อย (Common Pitfalls)

1. **Dereference `end()` โดยตรง** — `end()` ไม่ได้ชี้ไปยังสมาชิกตัวสุดท้าย แต่ชี้
   ไปยังตำแหน่ง "หลังตัวสุดท้าย" การเขียน `*container.end()` เป็น **Undefined
   Behavior** เสมอ ถ้าต้องการสมาชิกตัวสุดท้ายให้ใช้ `*container.rbegin()`
   หรือ `*std::prev(container.end())` แทน
2. **ใช้ `+=`, `[]`, หรือลบ iterator กันกับ container ที่ไม่ใช่ Random Access** —
   เช่นพยายามเขียน `list_it += 3` หรือ `list_it[2]` กับ `std::list::iterator`
   จะเจอ compile error ทันที เพราะ `std::list::iterator` เป็นแค่ Bidirectional
   Iterator ไม่มี `operator+=` หรือ `operator[]` ให้ใช้ (ต้องใช้ `std::advance`
   หรือ `std::next` แทน ซึ่งจะเลือกวิธีที่เหมาะสมให้อัตโนมัติ)
3. **Invalidate iterator แล้วยังใช้งานต่อ** — เมื่อเรียก `push_back`,
   `insert`, `erase` กับบาง container (เช่น `std::vector` ที่อาจ reallocate
   หน่วยความจำใหม่ทั้งก้อน) iterator เดิมที่ถืออยู่อาจกลายเป็น **Dangling
   Iterator** ทันที การใช้งาน iterator ที่ invalidate แล้วต่อเป็น Undefined
   Behavior — เรื่องนี้จะพูดถึงอย่างละเอียดอีกครั้งเมื่อพูดถึง `erase()` ร่วมกับ
   algorithm ใน Part 63 (erase-remove idiom)
4. **ลืมใส่ default constructor ใน custom Forward Iterator** — ถ้าจะให้ Iterator
   ของ class ที่เขียนเองถูกจัดว่าเป็น Forward Iterator ขึ้นไปตามมาตรฐาน (ใช้กับ
   algorithm บางตัวที่ต้องการ multi-pass guarantee) ต้องมี default constructor
   เสมอ มิฉะนั้นแม้โค้ดจะ compile ผ่านและใช้กับ range-based for ได้ปกติ แต่จะไม่
   ตรงตาม Forward Iterator concept อย่างเคร่งครัด
5. **สับสนระหว่าง `reverse_iterator::base()` กับตำแหน่งที่ `reverse_iterator`
   ชี้อยู่จริง** — `rit.base()` จะคืน iterator ปกติที่ชี้ไปยังตำแหน่ง **ถัดไป**
   จากที่ `rit` กำลังชี้ ไม่ใช่ตำแหน่งเดียวกัน ถ้าลืมจุดนี้แล้วนำไปใช้กับ
   `erase()` ตรงๆ อาจลบผิดตัวได้

---

## แบบฝึกหัดท้ายบท

1. เขียนฟังก์ชัน template `sum_range(first, last)` ที่รับ iterator สองตัว
   (begin, end) แล้วคืนผลรวมของสมาชิกทั้งหมด โดยต้องใช้ได้กับทั้ง
   `std::vector<int>`, `std::list<int>`, และ `std::set<int>` โดยไม่แก้โค้ด
   ฟังก์ชันเลยแม้แต่นิดเดียว
2. ใช้ `std::advance` และ `std::distance` กับ `std::list<int>` เพื่อหาสมาชิก
   ที่อยู่ตำแหน่งกึ่งกลางของ list โดยห้ามใช้ `operator[]` (เพราะ `list` ไม่มี
   ให้ใช้)
3. เขียนลูปแสดงผล `std::map<std::string, int>` จากคีย์มากไปน้อยโดยใช้
   `rbegin()`/`rend()` (ไม่ต้องเรียง sort เอง เพราะ `map` เรียงลำดับ key ให้อยู่
   แล้ว)
4. สร้าง class `Fibonacci` ที่ generate ลำดับฟีโบนัชชี N ตัวแรกผ่าน custom
   iterator ของตัวเอง (ไม่ต้องเก็บค่าไว้ล่วงหน้าใน container ใดๆ) ให้ใช้กับ
   range-based for loop ได้
5. อธิบายด้วยคำพูดของตัวเองว่าทำไม `std::unordered_map` จึงมี Iterator category
   เป็น Forward เท่านั้น ทั้งที่การค้นหาด้วย key ทำได้เร็วระดับ O(1) โดยเฉลี่ย
6. ลองเขียนโค้ดที่พยายามใช้ `it += 3` กับ `std::list<int>::iterator` แล้วดูว่า
   compiler แจ้ง error ว่าอย่างไร พร้อมอธิบายว่าทำไมถึง error

### แนวทางเฉลยข้อ 1

```cpp
#include <cstdio>
#include <vector>
#include <list>
#include <set>
#include <iterator>

template <typename Iterator>
auto sum_range(Iterator first, Iterator last) {
    typename std::iterator_traits<Iterator>::value_type total{};
    for (; first != last; ++first) {
        total += *first;
    }
    return total;
}

int main(void) {
    std::vector<int> v = {1, 2, 3, 4, 5};
    std::list<int> l = {10, 20, 30};
    std::set<int> s = {100, 200, 300};

    std::printf("vector sum: %d\n", sum_range(v.begin(), v.end()));
    std::printf("list sum:   %d\n", sum_range(l.begin(), l.end()));
    std::printf("set sum:    %d\n", sum_range(s.begin(), s.end()));

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 ex1.cpp -o ex1
./ex1
# vector sum: 15
# list sum:   60
# set sum:    600
```

จุดสำคัญของเฉลยนี้คือการใช้ `typename std::iterator_traits<Iterator>::value_type`
เพื่อประกาศตัวแปร `total` ให้มีชนิดข้อมูลตรงกับสิ่งที่ iterator ชี้อยู่ **โดยไม่ต้อง
รู้ล่วงหน้า** ว่าเรียกจาก container ชนิดไหน (คำว่า `typename` จำเป็นตรงนี้เพราะ
compiler ไม่รู้ล่วงหน้าว่า `iterator_traits<Iterator>::value_type` เป็นชนิดข้อมูล
หรือเป็นค่าคงที่ — จะพูดถึงเหตุผลลึกๆ ของ `typename` ในบริบท dependent type
อีกครั้งใน Part 78 เรื่อง Metaprogramming) ฟังก์ชันนี้ทำงานได้กับทุก container
ที่มี iterator ครบตามที่ต้องการ (`!=`, `++`, `*`) นี่คือพลังที่แท้จริงของ Generic
Programming ที่ Iterator เป็นตัวเชื่อม

### แนวทางเฉลยข้อ 4

```cpp
#include <cstdio>
#include <iterator>
#include <cstddef>

class Fibonacci {
public:
    explicit Fibonacci(int count) : count_(count) {}

    class Iterator {
    public:
        using iterator_category = std::input_iterator_tag;
        using value_type        = long long;
        using difference_type   = std::ptrdiff_t;
        using pointer           = const long long*;
        using reference         = const long long&;

        Iterator(int index, long long a, long long b) : index_(index), a_(a), b_(b) {}

        reference operator*() const { return a_; }

        Iterator& operator++() {
            long long next = a_ + b_;
            a_ = b_;
            b_ = next;
            ++index_;
            return *this;
        }

        bool operator==(const Iterator& other) const { return index_ == other.index_; }
        bool operator!=(const Iterator& other) const { return !(*this == other); }

    private:
        int index_;
        long long a_;
        long long b_;
    };

    Iterator begin() const { return Iterator(0, 0, 1); }
    Iterator end()   const { return Iterator(count_, 0, 1); }

private:
    int count_;
};

int main(void) {
    Fibonacci fib(10);
    for (long long x : fib) {
        std::printf("%lld ", x);
    }
    std::printf("\n");
    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 ex4.cpp -o ex4
./ex4
# 0 1 1 2 3 5 8 13 21 34
```

`Iterator` ของ `Fibonacci` เก็บสถานะไว้แค่ 3 ตัว (`index_`, `a_`, `b_`) และคำนวณ
ค่าถัดไปทีละก้าวตอน `++` ถูกเรียก โดยไม่มีการเก็บลำดับทั้งหมดไว้ในหน่วยความจำเลย —
นี่คือแนวคิดของ **Lazy Evaluation** ที่ประหยัดหน่วยความจำมาก เหมาะกับการ generate
ลำดับที่อาจยาวมากหรือไม่มีที่สิ้นสุด (จัดเป็น `input_iterator_tag` เพราะ
`operator++` เปลี่ยนค่าภายใน object เอง ทำให้การคัดลอก iterator แล้วเดินซ้ำสอง
ทางพร้อมกันจะได้ค่าไม่ตรงกันตามลำดับที่คาดหวัง — จึงไม่นับเป็น Forward Iterator
เต็มรูปแบบ)

---

## สรุปท้ายบท

ใน Part นี้เราได้:

- เข้าใจว่า Iterator คือแนวคิดที่ generalize เรื่อง Pointer ให้ใช้กับโครงสร้างข้อมูล
  ทุกชนิดของ STL ได้ด้วย interface เดียวกัน (`*`, `++`, `==`, `!=`)
- จำแนก Iterator ทั้ง 5 หมวด (Input, Output, Forward, Bidirectional, Random
  Access) และรู้ว่า container แต่ละชนิดของ STL ให้ Iterator หมวดไหน พร้อมเหตุผล
  เชิงโครงสร้างข้อมูลเบื้องหลัง
- ใช้ `begin()`/`end()`/`cbegin()`/`cend()`/`rbegin()`/`rend()` ได้อย่างถูกต้อง
  และเข้าใจความแตกต่างของแต่ละคู่
- ใช้ `std::advance`, `std::distance`, `std::next`, `std::prev` เขียนโค้ด
  generic ที่ทำงานได้กับ Iterator category ต่างๆ อย่างมีประสิทธิภาพเหมาะสม
- เข้าใจว่า range-based for loop เป็นแค่ syntactic sugar ที่ compiler แปลงเป็น
  การเรียก `begin()`/`end()` และ `!=`/`++`/`*` เบื้องหลัง
- เขียน custom iterator สำหรับ class ของตัวเองได้ทั้งสองแนวทาง (เขียนขึ้นใหม่
  ทั้งหมด และส่งต่อ iterator ของ container ภายใน) ทำให้ class ที่ออกแบบเองใช้กับ
  range-based for loop ได้เหมือน container มาตรฐานของ STL ทุกประการ

ความเข้าใจเรื่อง Iterator ใน Part นี้คือกุญแจสำคัญที่จะทำให้ **Part 63** ซึ่งจะ
พูดถึง **STL Algorithm** (`std::sort`, `std::find`, `std::transform` ฯลฯ)
เข้าใจง่ายขึ้นมาก เพราะ Algorithm ทุกตัวของ STL รับพารามิเตอร์เป็น Iterator
และเลือกวิธีทำงานที่เหมาะสมตาม Iterator Category ที่ได้รับมา — Iterator คือ
"กาว" ที่เชื่อม Container กับ Algorithm เข้าด้วยกันตามสถาปัตยกรรมดั้งเดิมของ STL

**ต่อไป:** [Part 63 — STL Algorithm แบบเจาะลึก](./part-063-stl-algorithms.md)
