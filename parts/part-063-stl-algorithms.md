# Part 63: STL Algorithm แบบเจาะลึก (Step 497–504)

> Module E — Templates, Generic Programming และ STL | Part 63 จาก 125
> Step ที่ครอบคลุมใน Part นี้: Step 497–504
> Part ก่อนหน้า: [Part 62 — STL Iterator แบบเจาะลึก](./part-062-stl-iterators.md) | Part ถัดไป: [Part 64 — Function Object, Lambda, std::function](./part-064-lambda-functors.md)

## เป้าหมายการเรียนรู้ (Learning Objectives)

หลังจบ Part นี้ ผู้เรียนจะสามารถ:

1. อธิบายได้ว่าทำไม header `<algorithm>` ถึงสำคัญ และทำไมควรใช้ algorithm สำเร็จรูป
   ของ STL แทนการเขียน sort/find เองแบบที่เรียนใน Part 23-24
2. ใช้ `std::sort` พร้อม custom comparator เพื่อเรียงข้อมูลตามเงื่อนไขที่ต้องการ
3. ใช้ `std::find` และ `std::find_if` ค้นหาข้อมูลใน container แบบ generic
4. ใช้ `std::transform` แปลงข้อมูลจาก range หนึ่งไปอีก range หนึ่ง
5. ใช้ `std::accumulate` (จาก `<numeric>`), `std::count`, `std::count_if` สรุปผล
   ข้อมูลจำนวนมากด้วยโค้ดสั้นกระชับ
6. ใช้ `std::for_each` และ `std::copy` ได้อย่างถูกต้อง
7. เข้าใจและใช้ **erase-remove idiom** ผ่าน `std::remove`/`std::remove_if` ได้
   อย่างปลอดภัย ซึ่งเป็นเทคนิคสำคัญที่สุดอย่างหนึ่งของ STL
8. ใช้ `std::unique` ร่วมกับการ sort เพื่อกำจัดข้อมูลซ้ำ และเห็นภาพว่า algorithm
   ของ STL ผสานกับ lambda expression (ปูทางสู่ Part 64) ได้อย่างไร

---

## 63.1 ทำไม `<algorithm>` ถึงสำคัญ: เขียนเองเทียบกับใช้ของสำเร็จรูป (Step 497)

ใน Part 23 เราเขียน **Bubble Sort** ด้วยมือ และใน Part 24 เราเขียน **Linear
Search** ด้วยมือเช่นกัน โค้ดเหล่านั้นสำคัญมากเพราะทำให้เราเข้าใจว่า Sorting และ
Searching ทำงานอย่างไร "ข้างใน" แต่ในการเขียนโปรแกรมจริงระดับมืออาชีพ เราแทบไม่
เคยเขียน Sort หรือ Find เองอีกเลย เพราะ Header `<algorithm>` ของ C++ มีฟังก์ชัน
สำเร็จรูปให้ใช้กว่า 100 ตัว ที่ผ่านการ optimize มาอย่างดีจากผู้เชี่ยวชาญ

```cpp
#include <cstdio>
#include <vector>
#include <algorithm>
#include <utility>

// bubble sort ที่เคยเขียนเองใน Part 23
void bubble_sort(std::vector<int>& v) {
    std::size_t n = v.size();
    for (std::size_t i = 0; i < n; ++i) {
        for (std::size_t j = 0; j + 1 < n - i; ++j) {
            if (v[j] > v[j + 1]) {
                std::swap(v[j], v[j + 1]);
            }
        }
    }
}

int main(void) {
    std::vector<int> a = {5, 3, 8, 1, 9, 2};
    std::vector<int> b = a; // สำเนาไว้เปรียบเทียบ

    bubble_sort(a);
    std::sort(b.begin(), b.end());

    std::printf("bubble_sort: ");
    for (int x : a) std::printf("%d ", x);
    std::printf("\n");

    std::printf("std::sort:   ");
    for (int x : b) std::printf("%d ", x);
    std::printf("\n");

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 sort_compare.cpp -o sort_compare
./sort_compare
# bubble_sort: 1 2 3 5 8 9
# std::sort:   1 2 3 5 8 9
```

ผลลัพธ์เหมือนกันทุกประการ แต่เบื้องหลังต่างกันมาก:

| หัวข้อเปรียบเทียบ | `bubble_sort` ที่เขียนเอง | `std::sort` |
|---|---|---|
| Time Complexity | O(n²) เสมอ ไม่ว่าข้อมูลจะเรียงมาก่อนหรือไม่ | เฉลี่ย O(n log n) — ใช้ **Introsort** (ผสม Quicksort + Heapsort + Insertion Sort) |
| ความยืดหยุ่นต่อชนิดข้อมูล | ต้องเขียนใหม่ทุกครั้งที่ชนิดข้อมูลเปลี่ยน (เว้นแต่ทำเป็น template เอง) | ใช้ Template ทำให้ใช้ได้กับทุกชนิดข้อมูลที่เปรียบเทียบกันได้ (`<`) ทันที |
| ความสามารถกำหนดเงื่อนไขการเรียง | ต้องแก้โค้ดเงื่อนไข `if (v[j] > v[j+1])` เอง | ส่ง comparator (lambda/functor) เข้าไปเป็นพารามิเตอร์ที่ 3 ได้เลย (63.2) |
| ผ่านการทดสอบและ optimize | ทดสอบเองเท่านั้น | ผ่านการทดสอบและปรับแต่งจากผู้เชี่ยวชาญ implement ตาม ISO C++ Standard นับล้าน edge case |
| การบำรุงรักษาในทีมใหญ่ | ต้อง maintain โค้ดเองตลอดไป | เป็นส่วนหนึ่งของ Standard Library ที่ compiler ทุกตัวรองรับเหมือนกัน |

**บทเรียนสำคัญของหัวข้อนี้ไม่ใช่ "ห้ามเขียนเองอีกต่อไป"** — ตรงกันข้าม การได้
เขียน Bubble Sort เองใน Part 23 ทำให้เราเข้าใจว่า `std::sort` กำลังทำอะไรอยู่
เบื้องหลัง และเข้าใจว่าทำไมมันถึงเร็วกว่า สิ่งที่ Part นี้จะสอนคือ **เมื่อไหร่ควร
ใช้เครื่องมือสำเร็จรูป** — และคำตอบคือ **แทบทุกครั้งในงานจริง** ยกเว้นกรณีพิเศษที่
ต้องเขียนอัลกอริทึมเฉพาะทางที่ STL ไม่มีให้ (ซึ่งพบได้น้อยมากในงาน Application
ทั่วไป)

`<algorithm>` ทำงานร่วมกับ Iterator ที่เรียนใน Part 62 เสมอ — algorithm แทบทุกตัว
รับพารามิเตอร์เป็นคู่ `(first, last)` หรือ range ของ iterator แทนที่จะรับ
container ตรงๆ นี่คือเหตุผลที่ทำให้ algorithm เดียวใช้ได้กับทั้ง `std::vector`,
`std::list`, `std::array`, หรือแม้แต่ raw array ได้โดยไม่ต้องเขียนแยกเวอร์ชัน

**แต่ก็มีข้อจำกัดที่ต้องระวัง**: algorithm บางตัวต้องการ Iterator category
ขั้นต่ำที่สูงกว่าตัวอื่น เช่น `std::sort` ต้องการ **Random Access Iterator**
เท่านั้น (เพราะอัลกอริทึม Introsort ต้องกระโดดเข้าถึงตำแหน่งกลางๆ ของข้อมูลได้
โดยตรงเพื่อความเร็วระดับ O(n log n)) การพยายามเรียก `std::sort` กับ
`std::list::iterator` (ซึ่งเป็นแค่ Bidirectional Iterator ตามตารางใน Part 62)
จะทำให้ **compile error ทันที** ไม่ใช่ error ตอนรันโปรแกรม:

```cpp
#include <cstdio>
#include <list>

int main(void) {
    std::list<int> l = {5, 3, 8, 1, 9, 2};

    // std::sort(l.begin(), l.end()); // ห้ามเขียนแบบนี้! ไม่ compile เพราะ list::iterator เป็นแค่ Bidirectional
    l.sort(); // ใช้ member function sort() ของ list เองแทน (ออกแบบมาเฉพาะสำหรับ linked list โดยเฉพาะ)

    for (int x : l) std::printf("%d ", x);
    std::printf("\n");

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 list_sort.cpp -o list_sort
./list_sort
# 1 2 3 5 8 9
```

นี่คือเหตุผลที่ `std::list` (และ `std::forward_list`) มี **member function
`sort()` ของตัวเอง** แยกต่างหากจาก `std::sort` ของ `<algorithm>` — เพราะ
`std::list::sort()` ถูกออกแบบมาให้ทำงานกับโครงสร้าง Linked List โดยเฉพาะ (ใช้
เทคนิค Merge Sort ที่จัดเรียง node โดยการสลับ pointer แทนที่จะสลับค่า) และให้
ผลลัพธ์ O(n log n) ได้เช่นกัน โดยไม่ต้องพึ่ง Random Access Iterator เลย จำไว้ว่า
**ถ้า compiler แจ้ง error ยาวๆ ที่พูดถึง iterator concept หรือ `operator-`/
`operator+` ไม่มีให้ใช้ ให้สงสัยไว้ก่อนว่าอาจเป็นเพราะ Iterator category ของ
container ที่ใช้ไม่ตรงกับที่ algorithm ต้องการ**

---

## 63.2 std::sort พร้อม Custom Comparator (Step 498)

`std::sort(first, last)` เรียงข้อมูลจากน้อยไปมากโดย default (ใช้ `operator<`)
แต่ถ้าต้องการเรียงแบบอื่น (มากไปน้อย, เรียงตาม field ของ struct ฯลฯ) สามารถส่ง
**comparator** เข้าไปเป็นพารามิเตอร์ตัวที่ 3 ได้:

```cpp
#include <cstdio>
#include <vector>
#include <algorithm>
#include <string>

struct Student {
    std::string name;
    int score;
};

int main(void) {
    std::vector<int> nums = {5, 3, 8, 1, 9, 2};

    // เรียงจากมากไปน้อยด้วย lambda comparator
    std::sort(nums.begin(), nums.end(), [](int x, int y) {
        return x > y;
    });
    for (int x : nums) std::printf("%d ", x);
    std::printf("\n");

    std::vector<Student> students = {
        {"Somchai", 75},
        {"Suda", 92},
        {"Anan", 88},
    };

    // เรียง struct ตามคะแนนจากมากไปน้อย
    std::sort(students.begin(), students.end(), [](const Student& s1, const Student& s2) {
        return s1.score > s2.score;
    });

    for (const auto& s : students) {
        std::printf("%-8s %d\n", s.name.c_str(), s.score);
    }

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 sort_comparator.cpp -o sort_comparator
./sort_comparator
```

ผลลัพธ์:

```
9 8 5 3 2 1
Suda     92
Anan     88
Somchai  75
```

Comparator ที่ส่งเข้าไปต้องเป็นฟังก์ชัน (หรืออะไรก็ตามที่เรียกได้แบบฟังก์ชัน
— เรียกว่า **Callable**) ที่รับ 2 พารามิเตอร์เป็นชนิดข้อมูลเดียวกับที่อยู่ใน
container และคืนค่า `bool` ที่หมายถึง **"a ควรอยู่ก่อน b หรือไม่"**
(Strict Weak Ordering) กฎสำคัญคือ:

- ถ้า `cmp(a, b)` คืน `true` แปลว่า `a` ต้องมาก่อน `b`
- ห้ามเขียนเงื่อนไขที่ใช้ `>=` หรือ `<=` เด็ดขาด (เช่น `return a >= b;`) เพราะจะ
  ละเมิดกฎ Strict Weak Ordering (เมื่อ `a == b` ทั้งสองด้านจะคืน `true` พร้อมกัน
  ซึ่งขัดแย้งในตัวเอง) ทำให้เกิด Undefined Behavior ได้ในบาง implementation
  (โปรแกรมอาจ crash หรือเรียงผิดแบบที่คาดเดาไม่ได้)

โค้ดในตัวอย่างใช้ **Lambda Expression** (`[](int x, int y) { return x > y; }`)
เป็น comparator ซึ่งเป็นวิธีที่นิยมที่สุดในปัจจุบัน — Part 64 จะพูดถึง Lambda
Expression อย่างละเอียดทุกแง่มุม แต่ในที่นี้ให้เข้าใจแค่ว่ามันคือฟังก์ชันแบบ
ไม่มีชื่อ (anonymous function) ที่นิยามและใช้งานได้ในบรรทัดเดียว สะดวกกว่าการ
ต้องประกาศฟังก์ชันแยกไว้ข้างนอกมาก

Comparator ยังใช้เรียง **หลายเงื่อนไขพร้อมกัน (Multi-key Sort)** ได้ด้วย เช่น
เรียงตามคะแนนก่อน ถ้าคะแนนเท่ากันค่อยเรียงตามชื่อ:

```cpp
#include <cstdio>
#include <vector>
#include <algorithm>
#include <string>

struct Student {
    std::string name;
    int score;
};

int main(void) {
    std::vector<Student> students = {
        {"Somchai", 88},
        {"Suda", 92},
        {"Anan", 88},
        {"Malee", 92},
    };

    // เรียงตามคะแนนมากไปน้อยก่อน ถ้าคะแนนเท่ากันให้เรียงตามชื่อ (a-z)
    std::sort(students.begin(), students.end(), [](const Student& s1, const Student& s2) {
        if (s1.score != s2.score) {
            return s1.score > s2.score;
        }
        return s1.name < s2.name;
    });

    for (const auto& s : students) {
        std::printf("%-8s %d\n", s.name.c_str(), s.score);
    }

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 multikey_sort.cpp -o multikey_sort
./multikey_sort
```

ผลลัพธ์:

```
Malee    92
Suda     92
Anan     88
Somchai  88
```

สังเกตว่า Malee กับ Suda คะแนนเท่ากัน (92) จึงเรียงตามชื่อ (a-z) ต่อ เช่นเดียวกับ
Anan กับ Somchai (88) — เทคนิคนี้ (เช็คเงื่อนไขหลักก่อน ถ้าเท่ากันค่อยไปเช็ค
เงื่อนไขรอง) ใช้ได้กับกี่เงื่อนไขก็ได้ตามต้องการ เพียงแค่เขียนซ้อน `if` ต่อกันไป

**ข้อควรรู้เพิ่มเติม**: `std::sort` **ไม่รับประกัน** ว่าสมาชิกที่ถือว่า
"เท่ากัน" ตาม comparator (เช่น คะแนนเท่ากันในตัวอย่างที่ยังไม่ได้เรียงตามชื่อ
ต่อ) จะยังคงเรียงตามลำดับเดิมก่อน sort อยู่หรือไม่ (เรียกว่าไม่ **stable**)
ถ้าต้องการการเรียงที่คงลำดับเดิมของสมาชิกที่เท่ากันไว้ (เช่น ต้องการเรียงตาม
คะแนนอย่างเดียว แต่รักษาลำดับที่มาก่อนไว้สำหรับคนคะแนนเท่ากัน) ให้ใช้
**`std::stable_sort`** แทน ซึ่งมี signature และวิธีใช้เหมือน `std::sort`
ทุกประการ ต่างกันแค่การรับประกันความเสถียรของลำดับ (แลกมาด้วยประสิทธิภาพที่
อาจช้ากว่าเล็กน้อยในบาง implementation)

---

## 63.3 std::find และ std::find_if (Step 499)

`std::find(first, last, value)` ค้นหาค่า `value` ใน range `[first, last)` แบบ
Linear Search (O(n)) คืน iterator ที่ชี้ไปยังตำแหน่งที่เจอ หรือคืน `last` ถ้า
ไม่เจอเลย ส่วน `std::find_if(first, last, predicate)` ค้นหาสมาชิกตัวแรกที่ทำให้
`predicate` (ฟังก์ชันที่คืน `bool`) เป็นจริง:

```cpp
#include <cstdio>
#include <vector>
#include <algorithm>
#include <iterator>

int main(void) {
    std::vector<int> v = {4, 8, 15, 16, 23, 42};

    auto it = std::find(v.begin(), v.end(), 16);
    if (it != v.end()) {
        std::printf("เจอ 16 ที่ index %td\n", std::distance(v.begin(), it));
    }

    auto it2 = std::find_if(v.begin(), v.end(), [](int x) {
        return x % 2 != 0; // หาเลขคี่ตัวแรก
    });
    if (it2 != v.end()) {
        std::printf("เลขคี่ตัวแรกคือ %d\n", *it2);
    }

    auto it3 = std::find(v.begin(), v.end(), 999);
    if (it3 == v.end()) {
        std::printf("ไม่พบ 999 ใน vector\n");
    }

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 find_demo.cpp -o find_demo
./find_demo
```

ผลลัพธ์:

```
เจอ 16 ที่ index 3
เลขคี่ตัวแรกคือ 15
ไม่พบ 999 ใน vector
```

รูปแบบ **"ค้นหาแล้วเช็คกับ `end()`"** (`if (it != v.end())`) เป็น idiom
มาตรฐานของ STL ที่ต้องจำให้ขึ้นใจ — algorithm ค้นหาแทบทุกตัวของ STL จะคืน
`end()` เมื่อไม่พบผลลัพธ์ (ไม่ใช่คืนค่าพิเศษอย่าง `-1` แบบที่บางภาษาใช้ หรือ
`nullptr` แบบฟังก์ชันค้นหาใน Part 24) เพราะ `end()` เป็นสิ่งที่มีอยู่แล้วในทุก
container โดยไม่ต้องมีเงื่อนไขพิเศษ

**ทำไมใช้ `std::find_if` แทนการเขียนลูปเช็คเงื่อนไขเอง?** เพราะ 1) โค้ดสั้นและ
สื่อความหมายชัดเจนกว่า (บอกทันทีว่า "กำลังค้นหา" ไม่ใช่ "กำลังวนลูปทำอะไรสักอย่าง")
2) ใช้ได้กับ container ใดๆ ที่มี iterator เหมือนกันทุกประการ ไม่ต้องเขียนแยกตาม
ชนิด container และ 3) เปิดโอกาสให้ compiler/library ปรับแต่งประสิทธิภาพภายในได้
โดยไม่กระทบโค้ดฝั่งผู้ใช้เลย

`std::find` เป็น **Linear Search** เสมอ (O(n)) เหมือนกับที่เรียนใน Part 24
ไม่ว่าข้อมูลจะเรียงลำดับไว้แล้วหรือไม่ก็ตาม แต่ถ้าข้อมูลใน range **เรียงลำดับ
ไว้แล้ว** (sorted) `<algorithm>` มีฟังก์ชันที่เร็วกว่ามากให้ใช้แทน นั่นคือ
`std::binary_search` และ `std::lower_bound` ที่ทำงานแบบ **Binary Search**
(O(log n)) ตามหลักการที่เรียนใน Part 24 เช่นกัน:

```cpp
#include <cstdio>
#include <vector>
#include <algorithm>

int main(void) {
    std::vector<int> v = {1, 3, 5, 7, 9, 11};

    bool found = std::binary_search(v.begin(), v.end(), 7);
    std::printf("พบ 7 หรือไม่: %s\n", found ? "พบ" : "ไม่พบ");

    auto it = std::lower_bound(v.begin(), v.end(), 6);
    std::printf("ตำแหน่งแรกที่ >= 6 คือค่า %d\n", *it);

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 binary_search_demo.cpp -o binary_search_demo
./binary_search_demo
# พบ 7 หรือไม่: พบ
# ตำแหน่งแรกที่ >= 6 คือค่า 7
```

`std::binary_search` คืนแค่ `bool` (พบหรือไม่พบ) ส่วน `std::lower_bound` คืน
iterator ที่ชี้ไปยัง **ตำแหน่งแรกที่มีค่าไม่น้อยกว่า** ค่าที่ค้นหา (มีประโยชน์
มากเวลาต้องการรู้ "ตำแหน่งที่ควรแทรก" ข้อมูลใหม่เพื่อให้ยังคงเรียงลำดับอยู่)
**ข้อแม้สำคัญที่ต้องจำ**: ทั้งสองฟังก์ชันนี้ใช้ได้เฉพาะกับ range ที่**เรียง
ลำดับไว้แล้วเท่านั้น** ถ้าเรียกกับข้อมูลที่ไม่ได้เรียงลำดับ ผลลัพธ์จะไม่
ถูกต้องโดยไม่มี error หรือ warning ใดๆ เตือนเลย (เป็น Undefined Behavior ตาม
ข้อกำหนดของมาตรฐาน)

---

## 63.4 std::transform (Step 500)

`std::transform` แปลงข้อมูลจาก range ต้นทางไปเก็บใน range ปลายทางโดยใช้ฟังก์ชัน
ที่กำหนด มีสองรูปแบบหลัก: แปลงจาก 1 range หรือรวม 2 range เข้าด้วยกัน:

```cpp
#include <cstdio>
#include <vector>
#include <algorithm>

int main(void) {
    std::vector<int> v = {1, 2, 3, 4, 5};
    std::vector<int> squared(v.size());

    std::transform(v.begin(), v.end(), squared.begin(), [](int x) {
        return x * x;
    });
    for (int x : squared) std::printf("%d ", x);
    std::printf("\n");

    // transform สองช่วงเข้าด้วยกันแบบ element-wise
    std::vector<int> a = {1, 2, 3};
    std::vector<int> b = {10, 20, 30};
    std::vector<int> sum(a.size());

    std::transform(a.begin(), a.end(), b.begin(), sum.begin(), [](int x, int y) {
        return x + y;
    });
    for (int x : sum) std::printf("%d ", x);
    std::printf("\n");

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 transform_demo.cpp -o transform_demo
./transform_demo
# 1 4 9 16 25
# 11 22 33
```

**สิ่งสำคัญที่ต้องระวัง**: `std::transform` **ไม่ได้สร้างพื้นที่เก็บข้อมูลปลายทาง
ให้เอง** — ในตัวอย่างแรก `squared` ถูก `resize` ล่วงหน้าด้วย
`std::vector<int> squared(v.size())` เพื่อจองพื้นที่ให้พอกับจำนวนสมาชิกต้นทาง
ก่อนเรียก `std::transform` ถ้าไม่จองพื้นที่ล่วงหน้าและใช้ `squared.begin()` เป็น
output iterator ตรงๆ จะเป็น Undefined Behavior ทันที (เขียนทับหน่วยความจำที่ยัง
ไม่มีอยู่จริง) — ถ้าต้องการหลีกเลี่ยงปัญหานี้ ให้ใช้ `std::back_inserter`
แทน ซึ่งจะกล่าวถึงในหัวข้อ 63.6

---

## 63.5 std::accumulate (`<numeric>`), std::count, std::count_if (Step 501)

`std::accumulate` อยู่ใน header **`<numeric>`** (แยกจาก `<algorithm>`) ทำหน้าที่
"พับรวม" (fold) ค่าทั้งหมดใน range ให้เหลือค่าเดียว โดย default จะบวกทุกตัวเข้า
ด้วยกัน แต่สามารถกำหนดการดำเนินการเองผ่านพารามิเตอร์ตัวที่ 4 ได้:

```cpp
#include <cstdio>
#include <vector>
#include <numeric>
#include <algorithm>

int main(void) {
    std::vector<int> v = {1, 2, 3, 4, 5};

    int total = std::accumulate(v.begin(), v.end(), 0);
    std::printf("ผลรวม: %d\n", total);

    int product = std::accumulate(v.begin(), v.end(), 1, [](int acc, int x) {
        return acc * x;
    });
    std::printf("ผลคูณ: %d\n", product);

    auto count_even = std::count_if(v.begin(), v.end(), [](int x) {
        return x % 2 == 0;
    });
    std::printf("จำนวนเลขคู่: %td\n", count_even);

    auto count_three = std::count(v.begin(), v.end(), 3);
    std::printf("จำนวนที่มีค่า 3: %td\n", count_three);

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 accumulate_demo.cpp -o accumulate_demo
./accumulate_demo
```

ผลลัพธ์:

```
ผลรวม: 15
ผลคูณ: 120
จำนวนเลขคู่: 2
จำนวนที่มีค่า 3: 1
```

พารามิเตอร์ตัวที่ 3 ของ `std::accumulate` (เช่น `0` หรือ `1` ในตัวอย่าง) คือ
**ค่าเริ่มต้น (Initial Value)** — ถ้าจะหาผลรวมต้องเริ่มที่ `0` ถ้าจะหาผลคูณต้อง
เริ่มที่ `1` (เพราะ `x * 1 = x` แต่ `x * 0 = 0` เสมอ) เป็นจุดที่มือใหม่ลืมบ่อย
มาก — ลืมใส่ initial value ที่ถูกต้องจะทำให้ผลลัพธ์ผิดทันทีแม้ logic ส่วนอื่น
จะถูกต้องหมดแล้ว

`std::count` และ `std::count_if` ทำงานคล้ายกับ `std::find`/`std::find_if` แต่
คืนค่าเป็น **จำนวน** ที่พบ (ชนิด `ptrdiff_t` แสดงผลด้วย `%td`) แทนที่จะคืน
iterator — ใช้บ่อยเวลาต้องการสถิติสรุปข้อมูล เช่น "มีสินค้ากี่ชิ้นที่หมดสต็อก"
หรือ "มีนักเรียนกี่คนที่สอบผ่าน"

`std::accumulate` ไม่จำเป็นต้องใช้กับตัวเลขเท่านั้น — ตราบใดที่ชนิดข้อมูลของ
ค่าเริ่มต้นรองรับ operation ที่กำหนด (default คือ `operator+`) ก็ใช้ได้หมด
เช่น การต่อ (concatenate) `std::string` หลายตัวเข้าด้วยกัน:

```cpp
#include <cstdio>
#include <vector>
#include <string>
#include <numeric>
#include <iterator>

int main(void) {
    std::vector<std::string> words = {"C++", "is", "powerful"};

    std::string sentence = std::accumulate(
        std::next(words.begin()), words.end(), words.front(),
        [](std::string acc, const std::string& w) {
            return acc + " " + w;
        });

    std::printf("%s\n", sentence.c_str());

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 accumulate_string.cpp -o accumulate_string
./accumulate_string
# C++ is powerful
```

ตัวอย่างนี้เริ่มค่าเริ่มต้นด้วย `words.front()` (คำแรก) แล้ววนไล่ต่อคำที่เหลือ
ตั้งแต่ `std::next(words.begin())` (ตัวที่สอง) เป็นต้นไป โดยใช้ lambda เป็น
ตัวกำหนด operation การรวมค่า (แทนที่จะบวกตัวเลขแบบเดิม กลายเป็นการต่อ string
พร้อมช่องว่างคั่น) — แสดงให้เห็นว่า `std::accumulate` เป็น algorithm แบบ
**fold** ที่ทั่วไปกว่าแค่ "หาผลรวม" มาก สามารถ "พับรวม" ข้อมูลชนิดใดก็ได้ตาม
กฎที่เรากำหนดเอง

---

## 63.6 std::for_each และ std::copy (Step 502)

`std::for_each(first, last, func)` เรียก `func` กับสมาชิกทุกตัวใน range
ตามลำดับ ทำหน้าที่คล้ายลูป `for` ธรรมดา แต่เขียนในรูปแบบ functional และสื่อ
เจตนาว่า "จะทำอะไรบางอย่างกับสมาชิกทุกตัว" ชัดเจนกว่า ส่วน `std::copy` คัดลอก
ข้อมูลจาก range หนึ่งไปยังอีก range หนึ่ง:

```cpp
#include <cstdio>
#include <vector>
#include <algorithm>
#include <iterator>

int main(void) {
    std::vector<int> v = {1, 2, 3, 4, 5};

    std::for_each(v.begin(), v.end(), [](int x) {
        std::printf("%d ", x * 2);
    });
    std::printf("\n");

    std::vector<int> dest(v.size());
    std::copy(v.begin(), v.end(), dest.begin());
    for (int x : dest) std::printf("%d ", x);
    std::printf("\n");

    // copy ไปยัง container ใหม่ทั้งหมดโดยไม่ต้อง resize ล่วงหน้า ด้วย back_inserter
    std::vector<int> dest2;
    std::copy(v.begin(), v.end(), std::back_inserter(dest2));
    for (int x : dest2) std::printf("%d ", x);
    std::printf("\n");

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 foreach_copy.cpp -o foreach_copy
./foreach_copy
# 2 4 6 8 10
# 1 2 3 4 5
# 1 2 3 4 5
```

`std::back_inserter(dest2)` คืน **Output Iterator** ชนิดพิเศษ (เรียกว่า
`std::back_insert_iterator`) ที่ทุกครั้งที่มีการเขียนค่าผ่านมัน (`*it = value`)
จะไปเรียก `dest2.push_back(value)` ให้อัตโนมัติ แทนที่จะเขียนทับตำแหน่งที่มีอยู่
แล้ว — วิธีนี้แก้ปัญหาที่พูดถึงใน 63.4 ได้เลย (ไม่ต้อง `resize` container
ปลายทางล่วงหน้าอีกต่อไป) เป็นตัวอย่างที่ดีว่า Iterator ใน STL ไม่จำเป็นต้อง
"ชี้ไปยังหน่วยความจำที่มีอยู่จริง" เสมอไป แต่สามารถเป็น "ตัวห่อหุ้มพฤติกรรม"
(behavior wrapper) ได้ด้วย ซึ่งเป็นการนำแนวคิด Iterator ที่เรียนใน Part 62
ไปใช้ได้อย่างสร้างสรรค์

หากต้องการคัดลอกเฉพาะสมาชิกที่ตรงเงื่อนไข (ไม่ใช่คัดลอกทั้งหมดแบบ `std::copy`)
มี `std::copy_if` ให้ใช้ ซึ่งรวมความสามารถของ `std::copy` เข้ากับเงื่อนไขแบบ
`std::find_if` ไว้ในตัวเดียว:

```cpp
#include <cstdio>
#include <vector>
#include <algorithm>
#include <iterator>

int main(void) {
    std::vector<int> v = {1, 2, 3, 4, 5, 6, 7, 8};
    std::vector<int> evens;

    std::copy_if(v.begin(), v.end(), std::back_inserter(evens), [](int x) {
        return x % 2 == 0;
    });

    for (int x : evens) std::printf("%d ", x);
    std::printf("\n");

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 copy_if_demo.cpp -o copy_if_demo
./copy_if_demo
# 2 4 6 8
```

---

## 63.7 std::remove / std::remove_if และ Erase-Remove Idiom (Step 503)

หัวข้อนี้คือหนึ่งในกับดักที่โปรแกรมเมอร์ C++ ทุกคนต้องเจอะเจอ และเป็นเทคนิคที่
**สำคัญที่สุด** อย่างหนึ่งของ STL: `std::remove` และ `std::remove_if`
**ไม่ได้ลบข้อมูลออกจาก container จริงๆ**

```cpp
#include <cstdio>
#include <vector>
#include <algorithm>

int main(void) {
    std::vector<int> v = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10};

    // บั๊กคลาสสิก: std::remove ไม่ได้ลบข้อมูลออกจาก container จริงๆ แค่ "เรียงย้าย" ตำแหน่งเท่านั้น
    auto new_end = std::remove(v.begin(), v.end(), 5);
    std::printf("หลัง remove (ยังไม่ erase) size=%zu: ", v.size());
    for (int x : v) std::printf("%d ", x);
    std::printf("\n");

    // ต้องเรียก erase ต่อเสมอเพื่อลบข้อมูล "ขยะ" ที่เหลือค้างท้าย container จริงๆ
    v.erase(new_end, v.end());
    std::printf("หลัง erase จริง size=%zu:      ", v.size());
    for (int x : v) std::printf("%d ", x);
    std::printf("\n");

    // แบบที่ใช้กันจริงในโค้ด production: เขียนรวดเดียวเรียกว่า "erase-remove idiom"
    std::vector<int> v2 = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10};
    v2.erase(std::remove_if(v2.begin(), v2.end(), [](int x) {
                 return x % 2 == 0; // ลบเลขคู่ทั้งหมด
             }),
             v2.end());
    for (int x : v2) std::printf("%d ", x);
    std::printf("\n");

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 remove_erase.cpp -o remove_erase
./remove_erase
```

ผลลัพธ์:

```
หลัง remove (ยังไม่ erase) size=10: 1 2 3 4 6 7 8 9 10 10
หลัง erase จริง size=9:      1 2 3 4 6 7 8 9 10
1 3 5 7 9
```

สังเกตผลลัพธ์บรรทัดแรกให้ดี — หลังเรียก `std::remove(v.begin(), v.end(), 5)`
ขนาดของ `v` **ยังคงเป็น 10 เท่าเดิม** และเลข `10` ปรากฏซ้ำสองครั้งท้าย vector!
นี่ไม่ใช่บั๊ก แต่เป็นการทำงานที่ถูกต้องตามที่ออกแบบไว้ — เหตุผลคือ **`std::remove`
ไม่มีสิทธิ์เปลี่ยนขนาดของ container ได้เลย** เพราะมันรับพารามิเตอร์เป็นแค่
Iterator (`first`, `last`) เท่านั้น ไม่ได้รับ reference ไปยัง container โดยตรง
(นี่คือข้อจำกัดโดยธรรมชาติของการออกแบบ Algorithm ให้แยกออกจาก Container ตาม
สถาปัตยกรรมของ STL ที่พูดถึงใน Part 62) สิ่งที่มันทำได้จริงๆ คือ **ย้าย**
สมาชิกที่ไม่ตรงเงื่อนไขมาไว้ด้านหน้าตามลำดับเดิม แล้วคืน iterator (`new_end`)
ที่ชี้ไปยังจุดที่ควรตัดจบข้อมูล "ที่แท้จริง" — ส่วนที่เหลือด้านหลัง `new_end`
จะกลายเป็นค่าที่ไม่แน่นอน (มักเป็นสำเนาของค่าที่ถูกย้ายไปแล้ว) ที่ต้องถูกลบทิ้ง
ด้วยเมธอด `erase()` ของ container เองอีกที

รูปแบบการเขียนที่ใช้กันเป็นมาตรฐานทั่วโลกเรียกว่า **Erase-Remove Idiom**:

```
container.erase(std::remove(container.begin(), container.end(), value),
                 container.end());
```

หรือกับเงื่อนไข (`remove_if`):

```
container.erase(std::remove_if(container.begin(), container.end(), predicate),
                 container.end());
```

ต้องจำ idiom นี้ให้ขึ้นใจ เพราะเป็นวิธีเดียวที่ถูกต้องในการลบสมาชิกหลายตัวออกจาก
`std::vector`/`std::deque`/`std::string` ตามเงื่อนไข (สำหรับ `std::list` มี
เมธอด `remove()`/`remove_if()` ของตัวเองที่ลบข้อมูลจริงในขั้นตอนเดียว ไม่ต้องใช้
idiom นี้ เพราะโครงสร้าง linked list ลบ node กลางได้ใน O(1) โดยไม่ต้องย้าย
ข้อมูลเหมือน array)

Erase-Remove Idiom ใช้ได้กับ `std::string` เช่นกัน เพราะภายในมันคือ contiguous
character array ที่มี `erase()` และรองรับ iterator แบบเดียวกับ `std::vector`
ทุกประการ — ตัวอย่างที่ใช้บ่อยมากในโค้ดจริงคือการลบช่องว่างทั้งหมดออกจาก
ข้อความ:

```cpp
#include <cstdio>
#include <string>
#include <algorithm>
#include <cctype>

int main(void) {
    std::string s = "Hello,   World!  How are   you?";

    s.erase(std::remove_if(s.begin(), s.end(), [](unsigned char c) {
                return std::isspace(c);
            }),
            s.end());

    std::printf("%s\n", s.c_str());

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 remove_whitespace.cpp -o remove_whitespace
./remove_whitespace
# Hello,World!Howareyou?
```

สังเกตว่า `std::isspace` รับพารามิเตอร์เป็น `unsigned char` (ไม่ใช่ `char`
ตรงๆ) — นี่เป็น convention ที่ถูกต้องตามมาตรฐานเสมอเมื่อเรียกฟังก์ชันตระกูล
`<cctype>` เพราะถ้า `char` เป็น signed บนระบบนั้น (ขึ้นกับ compiler/platform)
และค่าตัวอักษรเป็นค่าติดลบ (เช่นตัวอักษรที่ไม่ใช่ ASCII มาตรฐาน) การส่งเข้า
ฟังก์ชันเหล่านี้ตรงๆ จะเป็น Undefined Behavior — การ cast เป็น
`unsigned char` ก่อนเสมอคือวิธีที่ปลอดภัยที่สุด

---

## 63.8 std::unique และการผสาน Algorithm กับ Lambda (Step 504)

`std::unique(first, last)` ลบสมาชิกที่ซ้ำกัน **แต่ลบเฉพาะที่อยู่ติดกัน
(consecutive) เท่านั้น** และทำงานแบบเดียวกับ `std::remove` ทุกประการ (คืน
iterator ที่ต้องเอาไปใช้กับ `erase()` ต่อ ไม่ได้ลบข้อมูลออกจาก container จริง)

```cpp
#include <cstdio>
#include <vector>
#include <algorithm>

int main(void) {
    std::vector<int> v = {1, 1, 2, 2, 2, 3, 1, 1, 4};

    // unique ลบเฉพาะค่าซ้ำที่ "ติดกัน" เท่านั้น ต้อง sort ก่อนถ้าต้องการ unique ทั้งชุด
    auto new_end = std::unique(v.begin(), v.end());
    v.erase(new_end, v.end());
    for (int x : v) std::printf("%d ", x);
    std::printf("\n");

    std::vector<int> v2 = {4, 1, 3, 1, 2, 4, 2, 3};
    std::sort(v2.begin(), v2.end());
    v2.erase(std::unique(v2.begin(), v2.end()), v2.end());
    for (int x : v2) std::printf("%d ", x);
    std::printf("\n");

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 unique_demo.cpp -o unique_demo
./unique_demo
```

ผลลัพธ์:

```
1 2 3 1 4
1 2 3 4
```

จาก vector แรก `{1, 1, 2, 2, 2, 3, 1, 1, 4}` หลัง `unique` เหลือ
`{1, 2, 3, 1, 4}` — สังเกตว่า `1` ยังปรากฏสองครั้ง (ตำแหน่งแรกกับตำแหน่งที่ 4)
เพราะมันไม่ได้ติดกัน `std::unique` จึงไม่รู้ว่าเป็นค่าซ้ำ ถ้าต้องการให้เหลือ
ค่าที่ไม่ซ้ำกันทั้งชุดจริงๆ (แบบ vector ที่สอง) ต้อง **`std::sort` ก่อนเสมอ**
เพื่อทำให้ค่าที่เท่ากันมาอยู่ติดกันก่อน แล้วค่อยเรียก `std::unique` ตามด้วย
`erase` — รูปแบบ "sort + unique + erase" นี้เป็นวิธีมาตรฐานในการได้มาซึ่ง
"set ของค่าที่ไม่ซ้ำกันและเรียงลำดับแล้ว" จาก `std::vector` (ถ้าต้องการ set จริง
ๆ ตั้งแต่แรกโดยไม่ต้องทำหลายขั้นตอนแบบนี้ ให้พิจารณาใช้ `std::set` ตาม Part 61
แทน ขึ้นอยู่กับว่า pattern การใช้งานเหมาะกับอะไรมากกว่า)

ถ้าต้องการ **เก็บผลลัพธ์ที่ไม่ซ้ำไว้ใน container ใหม่** โดยไม่แก้ไข container
ต้นฉบับเลย มี `std::unique_copy` ให้ใช้ ซึ่งรวมพฤติกรรมของ `std::unique` กับ
`std::copy` ไว้ในฟังก์ชันเดียว:

```cpp
#include <cstdio>
#include <vector>
#include <algorithm>
#include <iterator>

int main(void) {
    std::vector<int> v = {1, 1, 2, 2, 3, 3, 3, 4};
    std::vector<int> result;

    std::unique_copy(v.begin(), v.end(), std::back_inserter(result));

    for (int x : result) std::printf("%d ", x);
    std::printf("\n");

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 unique_copy_demo.cpp -o unique_copy_demo
./unique_copy_demo
# 1 2 3 4
```

เช่นเดียวกับ `std::unique` ตรงๆ `std::unique_copy` ก็ยังคงกำจัดเฉพาะค่าซ้ำที่
ติดกันเท่านั้น (`v` ในตัวอย่างนี้เรียงลำดับมาแล้วพอดี) ถ้าข้อมูลต้นฉบับยังไม่
เรียงลำดับ ก็ยังต้อง `std::sort` ก่อนเสมอเช่นเดียวกับที่อธิบายไปข้างต้น

ตลอด Part นี้เราใช้ **Lambda Expression** เป็น argument ให้กับ algorithm แทบทุก
ตัว (`std::sort`, `std::find_if`, `std::transform`, `std::accumulate`,
`std::count_if`, `std::for_each`, `std::remove_if`) นี่ไม่ใช่เรื่องบังเอิญ —
มันคือรูปแบบการเขียนโค้ด C++ สมัยใหม่ที่พบมากที่สุดในโค้ดจริง เพราะ Lambda
ทำให้เขียนฟังก์ชันเล็กๆ แบบใช้ครั้งเดียวได้ตรงจุดที่ใช้งานเลย โดยไม่ต้องเสียเวลา
ไปประกาศฟังก์ชันแยกไว้ที่อื่น ใน **Part 64** เราจะเจาะลึก Lambda Expression
อย่างละเอียดทุกแง่มุม พร้อมทั้งเรียนรู้ **Function Object (Functor)** ที่เป็น
ต้นกำเนิดแนวคิดก่อน Lambda จะถือกำเนิดขึ้น และ **`std::function`** ที่ใช้เก็บ
Callable object ใดๆ ไว้ในตัวแปรเดียว

---

## ข้อผิดพลาดที่พบบ่อย (Common Pitfalls)

1. **เรียก `std::remove`/`std::remove_if` แล้วไม่เรียก `erase()` ต่อ** — เป็น
   บั๊กที่พบบ่อยที่สุดในหัวข้อนี้ ทำให้ container ยังมีขนาดเท่าเดิมและมีข้อมูล
   ซ้ำ/ขยะค้างอยู่ท้าย container โดยที่โค้ด compile ผ่านและไม่มี error หรือ
   warning ใดๆ เตือนเลย ต้องจำ Erase-Remove Idiom ให้ขึ้นใจเสมอ
2. **เรียก `std::unique` โดยไม่ `sort` ก่อน** — ได้ผลลัพธ์ที่ดู "เกือบถูก" (ลบ
   ค่าซ้ำบางส่วนออกไปจริง) แต่ไม่ครบถ้วน ทำให้เข้าใจผิดว่าโค้ดทำงานถูกต้องแล้ว
   ทั้งที่ข้อมูลซ้ำที่ไม่ติดกันยังหลงเหลืออยู่
3. **ใช้ `std::transform` เขียนลง iterator ปลายทางที่ยังไม่มีพื้นที่จองไว้** —
   ไม่ `resize` container ปลายทางล่วงหน้าและไม่ใช้ `std::back_inserter` เป็น
   Undefined Behavior (เขียนทับหน่วยความจำนอกขอบเขตของ container)
4. **เขียน comparator ให้ `std::sort` แบบผิดกฎ Strict Weak Ordering** — เช่นใช้
   `>=` หรือ `<=` แทน `>` หรือ `<` ล้วนๆ ทำให้เกิด Undefined Behavior ที่อาจ
   แสดงผลเป็นการ crash หรือผลลัพธ์เรียงผิดแบบสุ่มไม่คงที่ในแต่ละครั้งที่รัน
5. **ลืมว่า `std::find`/`std::count` เป็น Linear Search เสมอ (O(n))** — ถ้าต้อง
   ค้นหาบ่อยๆ ในข้อมูลจำนวนมาก ควรพิจารณาใช้ `std::map`/`std::set`/
   `std::unordered_map` (Part 61) ที่ค้นหาได้เร็วกว่ามาก แทนที่จะเก็บข้อมูลใน
   `std::vector` แล้วเรียก `std::find` วนซ้ำๆ

---

## แบบฝึกหัดท้ายบท

1. เขียนโปรแกรมที่ใช้ `std::count_if` นับจำนวนคำที่ยาวเกิน 3 ตัวอักษรใน
   `std::vector<std::string>`
2. ใช้ Erase-Remove Idiom ลบสมาชิกที่เป็นค่าลบทั้งหมดออกจาก `std::vector<int>`
3. ใช้ `std::transform` แปลง `std::vector<std::string>` ให้เป็นตัวพิมพ์ใหญ่
   ทั้งหมด (ใช้ `std::toupper` จาก `<cctype>`)
4. เขียนโปรแกรมใช้ `std::accumulate` หาค่าเฉลี่ย (average) ของ
   `std::vector<double>`
5. อธิบายด้วยคำพูดของตัวเองว่าทำไม `std::unique` ถึงต้องเรียก `std::sort` ก่อน
   ถ้าต้องการลบค่าซ้ำ "ทั้งหมด" ไม่ใช่แค่ที่อยู่ติดกัน
6. เขียนโปรแกรมที่รวม `std::sort` + `std::unique` + `erase()` ไว้ในสามบรรทัด
   เพื่อให้ได้ `std::vector<int>` ที่เรียงลำดับแล้วและไม่มีค่าซ้ำเลย

### แนวทางเฉลยข้อ 1

```cpp
#include <cstdio>
#include <vector>
#include <string>
#include <algorithm>

int main(void) {
    std::vector<std::string> words = {"cat", "elephant", "dog", "hippopotamus", "ant"};

    auto count_long = std::count_if(words.begin(), words.end(), [](const std::string& w) {
        return w.size() > 3;
    });

    std::printf("จำนวนคำที่ยาวเกิน 3 ตัวอักษร: %td\n", count_long);

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 ex1.cpp -o ex1
./ex1
# จำนวนคำที่ยาวเกิน 3 ตัวอักษร: 2
```

`elephant` (8 ตัวอักษร) และ `hippopotamus` (12 ตัวอักษร) เป็นคำที่ยาวเกิน 3
ตัวอักษร ส่วน `cat`, `dog`, `ant` (3 ตัวอักษรพอดี) ไม่นับ เพราะเงื่อนไขคือ
`w.size() > 3` (มากกว่า ไม่ใช่มากกว่าเท่ากับ) — เป็นตัวอย่างที่ดีว่า
`std::count_if` ทำให้การนับข้อมูลตามเงื่อนไขใดๆ ก็ตามเขียนได้ในบรรทัดเดียว
โดยไม่ต้องประกาศตัวแปรนับ (`int count = 0;`) แล้ววนลูปเช็คเงื่อนไขเองแบบเดิม

### แนวทางเฉลยข้อ 6

```cpp
#include <cstdio>
#include <vector>
#include <algorithm>

int main(void) {
    std::vector<int> v = {5, 3, 8, 3, 1, 5, 9, 1};

    std::sort(v.begin(), v.end());
    v.erase(std::unique(v.begin(), v.end()), v.end());

    for (int x : v) std::printf("%d ", x);
    std::printf("\n");

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 ex6.cpp -o ex6
./ex6
# 1 3 5 8 9
```

โค้ดนี้คือรูปแบบ "sort + unique + erase" ที่พูดถึงในหัวข้อ 63.8 บรรทัดแรก
`std::sort` เรียงข้อมูลจากน้อยไปมาก ทำให้ค่าที่เท่ากันมาอยู่ติดกัน บรรทัดที่สอง
`std::unique(...)` ย้ายค่าที่ไม่ซ้ำมาไว้ด้านหน้าแล้วคืน iterator ตัดจบที่แท้จริง
และ `v.erase(...)` ลบส่วนที่เหลือด้านหลัง (ค่าขยะ) ออกจริง ผลลัพธ์สุดท้ายคือ
`{1, 3, 5, 8, 9}` ซึ่งเรียงลำดับแล้วและไม่มีค่าซ้ำเลย

---

## สรุปท้ายบท

ใน Part นี้เราได้:

- เข้าใจเหตุผลที่ควรใช้ `<algorithm>` ของ STL แทนการเขียน sort/find เอง
  โดยยังคงเข้าใจหลักการเบื้องหลังจากที่เคยเขียนเองใน Part 23-24
- ใช้ `std::sort` พร้อม custom comparator เรียงข้อมูลตามเงื่อนไขใดๆ ที่ต้องการ
- ใช้ `std::find`/`std::find_if` ค้นหาข้อมูลแบบ generic ที่ใช้ได้กับทุก
  container
- ใช้ `std::transform` แปลงข้อมูลจาก range หนึ่งไปอีก range หนึ่ง ทั้งแบบ
  single-range และ two-range
- ใช้ `std::accumulate`, `std::count`, `std::count_if` สรุปผลข้อมูลจำนวนมาก
  ได้อย่างกระชับ
- ใช้ `std::for_each` และ `std::copy` (รวมถึง `std::back_inserter`) ได้อย่าง
  ถูกต้อง
- เข้าใจและใช้ **Erase-Remove Idiom** ได้อย่างปลอดภัย ซึ่งเป็นเทคนิคที่ต้องใช้
  แทบทุกครั้งที่ต้องการลบสมาชิกหลายตัวออกจาก `std::vector` ตามเงื่อนไข
- ใช้ `std::unique` ร่วมกับ `std::sort` เพื่อกำจัดข้อมูลซ้ำทั้งชุด และเห็นว่า
  Lambda Expression กลายเป็นส่วนหนึ่งของการเขียนโค้ดร่วมกับ Algorithm อย่าง
  แยกไม่ออกในโค้ด C++ สมัยใหม่

ใน **Part 64** เราจะเจาะลึกเรื่อง **Function Object (Functor), Lambda
Expression และ `std::function`** อย่างเต็มรูปแบบ ตั้งแต่ต้นกำเนิดของแนวคิด
Functor ก่อนที่ Lambda จะถือกำเนิดขึ้นใน C++11 ไปจนถึงการเก็บ Callable
ทุกชนิดไว้ในตัวแปรเดียวด้วย `std::function` ซึ่งเป็นพื้นฐานสำคัญของการออกแบบ
ระบบ Callback ในโปรแกรมจริง

**ต่อไป:** [Part 64 — Function Object, Lambda, std::function](./part-064-lambda-functors.md)
