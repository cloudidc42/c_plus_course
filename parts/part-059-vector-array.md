# Part 59: std::vector และ std::array แบบเจาะลึก (Step 465–472)

> Module E — Templates, Generic Programming และ STL | Part 59 จาก 125
> Step ที่ครอบคลุมใน Part นี้: Step 465–472
> Part ก่อนหน้า: [Part 58 — แนะนำ STL](./part-058-stl-overview.md) | Part ถัดไป: [Part 60 — std::list, std::deque, std::forward_list](./part-060-list-deque-forwardlist.md)

## เป้าหมายการเรียนรู้ (Learning Objectives)

หลังจบ Part นี้ ผู้เรียนจะสามารถ:

1. สร้าง เข้าถึง และแก้ไขข้อมูลใน `std::vector` ด้วย `push_back`, `pop_back`, `insert`, `erase`,
   `size`, `capacity` ได้อย่างถูกต้อง
2. อธิบายความแตกต่างระหว่าง `size()` กับ `capacity()` และเหตุผลที่ทั้งสองค่าไม่จำเป็นต้องเท่ากัน
3. อธิบายกลไก **reallocation** ของ vector เมื่อพื้นที่เต็ม และวิเคราะห์ได้ว่าทำไม `push_back`
   ยังคงเป็น **amortized O(1)** ทั้งที่บางครั้งต้อง copy ข้อมูลทั้งก้อนใหม่
4. ใช้ `reserve()` เพื่อลดจำนวนครั้งของการ reallocation เมื่อรู้ขนาดข้อมูลล่วงหน้า
5. ระบุและป้องกันปัญหา **Iterator Invalidation** ซึ่งเป็นบั๊กคลาสสิกที่พบบ่อยที่สุดของ vector
   ทั้งกรณี erase/insert ระหว่าง loop และกรณี pointer/reference ค้าง (dangling)
6. ใช้ `std::array` แทน C array แบบดั้งเดิมได้อย่างถูกต้อง พร้อมอธิบายข้อดีที่เหนือกว่า
7. ตัดสินใจเลือกระหว่าง `std::vector`, `std::array` และ C array ได้อย่างเหมาะสมกับสถานการณ์จริง

---

## 59.1 std::vector คืออะไร และการสร้าง/เข้าถึงข้อมูลเบื้องต้น (Step 465)

ใน Part 58 เราได้เห็นภาพรวมว่า `std::vector<T>` คือ **Sequence Container** ที่เก็บข้อมูลชนิด
`T` ต่อเนื่องกันในหน่วยความจำแบบ heap (คล้าย dynamic array ที่เราเคย `malloc`/`realloc` เองใน
Part 11) แต่ `vector` จัดการการขยายขนาดหน่วยความจำให้เราอัตโนมัติทั้งหมด ไม่ต้องยุ่งกับ
`malloc`/`free` เองอีกต่อไป

### วิธีสร้าง vector

```cpp
#include <iostream>
#include <vector>

int main() {
    std::vector<int> scores = {88, 95, 72, 60, 100};   // สร้างพร้อมค่าเริ่มต้น (initializer list)

    std::cout << "จำนวนสมาชิก: " << scores.size() << '\n';
    std::cout << "สมาชิกตัวแรก: " << scores.front() << '\n';
    std::cout << "สมาชิกตัวสุดท้าย: " << scores.back() << '\n';
    std::cout << "สมาชิกตัวที่ 2 (index 1): " << scores[1] << '\n';
    std::cout << "สมาชิกตัวที่ 2 ผ่าน at(): " << scores.at(1) << '\n';

    std::vector<double> prices(5, 9.99);   // สร้าง vector ขนาด 5 ตัว ทุกตัวเป็น 9.99
    std::cout << "\nprices มี " << prices.size() << " ตัว ทุกตัวเป็น: ";
    for (double p : prices) {
        std::cout << p << ' ';
    }
    std::cout << '\n';

    std::vector<int> empty_vec;   // vector ว่าง ยังไม่มีสมาชิก
    std::cout << "\nempty_vec.empty() = " << std::boolalpha << empty_vec.empty() << '\n';

    try {
        std::cout << scores.at(100) << '\n';   // index เกินขอบเขต
    } catch (const std::out_of_range& e) {
        std::cout << "จับ exception จาก at(): " << e.what() << '\n';
    }

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 s1_basic.cpp -o s1
./s1
```

ผลลัพธ์:

```
จำนวนสมาชิก: 5
สมาชิกตัวแรก: 88
สมาชิกตัวสุดท้าย: 100
สมาชิกตัวที่ 2 (index 1): 95
สมาชิกตัวที่ 2 ผ่าน at(): 95

prices มี 5 ตัว ทุกตัวเป็น: 9.99 9.99 9.99 9.99 9.99 

empty_vec.empty() = true
จับ exception จาก at(): vector::_M_range_check: __n (which is 100) >= this->size() (which is 5)
```

### `operator[]` กับ `at()` ต่างกันอย่างไร

| | `v[i]` | `v.at(i)` |
|---|---|---|
| ตรวจสอบขอบเขต | **ไม่ตรวจ** — index เกินคือ Undefined Behavior | ตรวจเสมอ — throw `std::out_of_range` ถ้าเกิน |
| ความเร็ว | เร็วที่สุด (ไม่มี overhead การเช็ค) | ช้ากว่าเล็กน้อยเพราะต้องเช็คทุกครั้ง |
| ใช้เมื่อไหร่ | เมื่อมั่นใจว่า index ถูกต้องแน่นอน (เช่นวน loop ด้วย `i < v.size()`) | เมื่อ index มาจากภายนอก (user input, network) ที่ไม่น่าเชื่อถือ |

นี่คือหลักการเดียวกับที่เราเรียนเรื่อง array ใน C (Part 7): C++ เพียงแค่ให้ทางเลือกที่ปลอดภัย
กว่า (`at()`) เพิ่มเข้ามา แต่ยังคงให้ทางเลือกที่เร็วที่สุด (`operator[]`) ไว้สำหรับกรณีที่ performance
สำคัญกว่า safety check

---

## 59.2 push_back, pop_back, insert, erase (Step 466)

การแก้ไขเนื้อหาของ vector ทำผ่านชุด method มาตรฐานเหล่านี้:

```cpp
#include <iostream>
#include <vector>

void print_vec(const std::string& label, const std::vector<int>& v) {
    std::cout << label << ": ";
    for (int x : v) {
        std::cout << x << ' ';
    }
    std::cout << '\n';
}

int main() {
    std::vector<int> queue_numbers;

    queue_numbers.push_back(101);
    queue_numbers.push_back(102);
    queue_numbers.push_back(103);
    print_vec("หลัง push_back 3 ครั้ง", queue_numbers);

    queue_numbers.pop_back();
    print_vec("หลัง pop_back", queue_numbers);

    // insert แทรกที่ตำแหน่งที่ต้องการ (คืนค่าเป็น iterator ที่ชี้ไปยังตำแหน่งที่แทรก)
    queue_numbers.insert(queue_numbers.begin() + 1, 999);
    print_vec("หลัง insert 999 ที่ index 1", queue_numbers);

    // erase ลบสมาชิกที่ตำแหน่งที่ระบุ (คืนค่าเป็น iterator ตัวถัดจากตัวที่ถูกลบ)
    queue_numbers.erase(queue_numbers.begin());
    print_vec("หลัง erase ตัวแรก", queue_numbers);

    // erase แบบช่วง [first, last)
    std::vector<int> range_demo = {1, 2, 3, 4, 5, 6, 7};
    range_demo.erase(range_demo.begin() + 1, range_demo.begin() + 4);
    print_vec("erase ช่วง [1,4)", range_demo);

    // emplace_back สร้าง object ตรงตำแหน่งในหน่วยความจำ ไม่ต้องสร้างชั่วคราวแล้ว copy/move
    std::vector<std::pair<int, int>> points;
    points.emplace_back(3, 4);
    points.emplace_back(5, 12);
    for (auto& p : points) {
        std::cout << "(" << p.first << ", " << p.second << ") ";
    }
    std::cout << '\n';

    // clear ลบทุกสมาชิก แต่ capacity อาจไม่เปลี่ยน
    range_demo.clear();
    std::cout << "หลัง clear: size = " << range_demo.size()
              << ", capacity = " << range_demo.capacity() << '\n';

    return 0;
}
```

ผลลัพธ์:

```
หลัง push_back 3 ครั้ง: 101 102 103 
หลัง pop_back: 101 102 
หลัง insert 999 ที่ index 1: 101 999 102 
หลัง erase ตัวแรก: 999 102 
erase ช่วง [1,4): 1 5 6 7 
(3, 4) (5, 12) 
หลัง clear: size = 0, capacity = 7
```

จุดที่ควรสังเกต:

- **`push_back` / `pop_back`** ทำงานที่ **ท้าย vector เท่านั้น** — เร็วที่สุด (amortized O(1))
  เพราะไม่ต้องขยับสมาชิกตัวอื่น
- **`insert` / `erase` ที่ตำแหน่งกลางหรือหน้า** ต้อง **shift สมาชิกทุกตัวที่อยู่ถัดจากตำแหน่งนั้น**
  ไปข้างหน้า/ข้างหลัง 1 ช่อง จึงมี complexity เป็น **O(n)** — ยิ่ง vector ใหญ่ ยิ่งช้า
- **`emplace_back`** ต่างจาก `push_back` ตรงที่มันสร้าง object **ในตำแหน่งหน่วยความจำจริง
  ของ vector โดยตรง** (in-place construction) ด้วยการส่ง constructor argument ตรงๆ
  ในขณะที่ `push_back` ต้องสร้าง object ชั่วคราวก่อนแล้วจึง copy/move เข้าไป — สำหรับ
  `std::pair<int,int>` แบบง่ายผลต่างอาจเล็กน้อย แต่กับ object ที่ constructor ทำงานหนัก
  (เช่น class ที่มี `std::string` หลายตัว) `emplace_back` จะเร็วกว่าอย่างชัดเจน
- **`clear()`** ลบสมาชิกทั้งหมด (`size()` กลายเป็น 0) แต่ **ไม่คืนหน่วยความจำที่จองไว้กลับให้ระบบ**
  — `capacity()` ยังคงเท่าเดิม เราจะอธิบายเหตุผลในหัวข้อถัดไป

---

## 59.3 size() กับ capacity() และกลไก Reallocation (Step 467)

นี่คือแนวคิดที่สำคัญที่สุดของ `std::vector` และเป็นสิ่งที่แยก vector ออกจาก array ธรรมดา:

- **`size()`** คือ **จำนวนสมาชิกที่มีอยู่จริงตอนนี้**
- **`capacity()`** คือ **จำนวนสมาชิกสูงสุดที่ vector รองรับได้ก่อนจะต้องขอหน่วยความจำก้อนใหม่**
  (นั่นคือขนาดของ "บ้าน" ที่จองไว้ ซึ่งอาจใหญ่กว่าจำนวนคนที่อยู่จริงตอนนี้)

เสมอ: `size() <= capacity()`

ลองวัดค่าจริงระหว่างการ `push_back` ต่อเนื่อง:

```cpp
#include <iostream>
#include <vector>

int main() {
    std::vector<int> v;
    std::size_t last_capacity = v.capacity();

    std::cout << "size\tcapacity\n";
    std::cout << v.size() << '\t' << v.capacity() << " (เริ่มต้น)\n";

    for (int i = 0; i < 20; ++i) {
        v.push_back(i);
        if (v.capacity() != last_capacity) {
            std::cout << v.size() << '\t' << v.capacity()
                      << "  <-- reallocation เกิดขึ้น!\n";
            last_capacity = v.capacity();
        }
    }

    return 0;
}
```

ผลลัพธ์ (คอมไพล์ด้วย GCC 13 / libstdc++ บน Linux):

```
size	capacity
0	0 (เริ่มต้น)
1	1  <-- reallocation เกิดขึ้น!
2	2  <-- reallocation เกิดขึ้น!
3	4  <-- reallocation เกิดขึ้น!
5	8  <-- reallocation เกิดขึ้น!
9	16  <-- reallocation เกิดขึ้น!
17	32  <-- reallocation เกิดขึ้น!
```

จะเห็นว่า vector เริ่มจาก `capacity() == 0` และทุกครั้งที่พื้นที่เต็ม (`size() == capacity()`)
และมีการ `push_back` เพิ่มอีก vector จะ:

1. จองหน่วยความจำก้อนใหม่ที่ **ใหญ่กว่าเดิม** (ใน libstdc++/GCC คือคูณด้วยประมาณ 2 เท่า
   — ไลบรารีอื่นอาจใช้ตัวคูณต่างกัน เช่น libc++ ของ Clang ใช้ประมาณ 2 เท่าเช่นกัน แต่
   MSVC ของ Visual Studio ใช้ประมาณ 1.5 เท่า — **มาตรฐาน C++ ไม่ได้กำหนดตัวคูณที่แน่นอน**
   เพียงแค่กำหนดว่า amortized complexity ของ `push_back` ต้องเป็น O(1))
2. **copy (หรือ move ถ้าชนิดข้อมูลรองรับ) สมาชิกทุกตัวจากที่เก่าไปยังที่ใหม่**
3. **คืนหน่วยความจำก้อนเก่ากลับให้ระบบ**
4. แทรกสมาชิกตัวใหม่เข้าไปในก้อนใหม่

ขั้นตอนนี้คือ **Reallocation** ซึ่งมี cost O(n) ในครั้งที่เกิดขึ้น — แต่คำถามคือ ถ้ามันเกิด O(n)
เป็นครั้งคราว แล้วทำไมเราถึงยังบอกว่า `push_back` เป็น O(1) ได้?

---

## 59.4 Amortized O(1) ของ push_back (Step 468)

คำตอบอยู่ที่คำว่า **Amortized (ถัวเฉลี่ยตลอดช่วงเวลา)** ลองวิเคราะห์จากผลลัพธ์ในหัวข้อก่อน:
reallocation เกิดขึ้นที่ size = 1, 2, 3, 5, 9, 17 — นั่นคือ **ยิ่ง vector ใหญ่ขึ้น ระยะห่างระหว่าง
การ reallocation แต่ละครั้งก็ยิ่งห่างขึ้นแบบ exponential (เท่าตัว)**

ลองคิดเป็นสมการ: ถ้าเราทำ `push_back` ทั้งหมด `n` ครั้งติดต่อกัน (เริ่มจาก vector ว่าง) และ
ทุกครั้งที่ reallocate ขนาดจะเพิ่มเป็น 2 เท่า จำนวน element ทั้งหมดที่ต้อง copy ตลอดประวัติศาสตร์
ของ vector คือ:

```
1 + 2 + 4 + 8 + ... + n/2  ≈  2n - 1   (อนุกรมเรขาคณิต)
```

นั่นคือ แม้จะมีการ copy เกิดขึ้นหลายครั้ง แต่ **ผลรวมของงาน copy ทั้งหมดตลอด n ครั้งของ
push_back ไม่เกิน 2n** ดังนั้นเมื่อเฉลี่ยงานทั้งหมดต่อการ `push_back` หนึ่งครั้ง (`total_work / n`)
จะได้ค่าคงที่ (constant) ไม่ขึ้นกับ `n` — นี่คือความหมายของ **Amortized O(1)**

เปรียบเทียบกับกรณีที่ vector ขยายทีละ 1 ตัว (**ไม่คูณ 2 แต่บวก 1**) แทน จะเกิด reallocation
ทุกครั้งที่ `push_back` (เพราะ capacity ตามไม่ทันเสมอ) ทำให้ต้อง copy ข้อมูลทั้งหมด
`1 + 2 + 3 + ... + n ≈ n²/2` ครั้ง — กลายเป็น **O(n²)** รวม ซึ่งช้ากว่ามาก นี่คือเหตุผลที่
ทุก implementation ของ `std::vector` เลือกใช้ **growth factor แบบ exponential (คูณ)**
ไม่ใช่แบบ linear (บวก) — เป็นการแลกเปลี่ยนระหว่าง "หน่วยความจำที่อาจสูญเปล่าไปบ้าง"
กับ "ความเร็วโดยรวมที่เร็วกว่ามาก"

ลองวัดผลจริงเปรียบเทียบจำนวนครั้งของ reallocation ระหว่างการ `push_back` 100,000 ครั้ง
โดยมี/ไม่มีการ `reserve()` ล่วงหน้า (หัวข้อถัดไป) เพื่อยืนยันแนวคิดนี้ด้วยตัวเลขจริง

---

## 59.5 reserve() เพื่อลด Reallocation (Step 469)

ถ้าเรา**รู้ล่วงหน้า**ว่าจะเก็บข้อมูลประมาณกี่ตัว เราสามารถบอก vector ให้จองพื้นที่ไว้ล่วงหน้า
ด้วย `reserve(n)` เพื่อ**ตัดขั้นตอน reallocation ทั้งหมดทิ้งไป**:

```cpp
#include <iostream>
#include <vector>

int count_reallocations(std::vector<int>& v, int n) {
    int count = 0;
    std::size_t last_capacity = v.capacity();
    for (int i = 0; i < n; ++i) {
        v.push_back(i);
        if (v.capacity() != last_capacity) {
            ++count;
            last_capacity = v.capacity();
        }
    }
    return count;
}

int main() {
    const int N = 100000;

    std::vector<int> without_reserve;
    int realloc_count_1 = count_reallocations(without_reserve, N);
    std::cout << "ไม่ใช้ reserve: push_back " << N << " ครั้ง เกิด reallocation "
              << realloc_count_1 << " ครั้ง (capacity สุดท้าย = "
              << without_reserve.capacity() << ")\n";

    std::vector<int> with_reserve;
    with_reserve.reserve(N);
    int realloc_count_2 = count_reallocations(with_reserve, N);
    std::cout << "ใช้ reserve(" << N << ") ล่วงหน้า: เกิด reallocation "
              << realloc_count_2 << " ครั้ง (capacity สุดท้าย = "
              << with_reserve.capacity() << ")\n";

    return 0;
}
```

ผลลัพธ์:

```
ไม่ใช้ reserve: push_back 100000 ครั้ง เกิด reallocation 18 ครั้ง (capacity สุดท้าย = 131072)
ใช้ reserve(100000) ล่วงหน้า: เกิด reallocation 0 ครั้ง (capacity สุดท้าย = 100000)
```

สังเกตว่าเมื่อ `reserve(100000)` แล้ว capacity สุดท้ายเท่ากับ 100000 พอดี (ไม่มีการปัดขึ้นเป็น
เลขยกกำลัง 2) ในขณะที่ไม่ `reserve` เลย capacity สุดท้ายกลายเป็น 131072 (2¹⁷) — มากกว่าที่
ต้องการใช้จริงถึง 31,072 ช่อง ซึ่งเป็นหน่วยความจำที่ถูกจองไว้แต่ไม่ได้ใช้ (สูญเปล่า)

### `reserve()` vs `resize()`

ห้ามสับสนสอง method นี้:

| | `reserve(n)` | `resize(n)` |
|---|---|---|
| เปลี่ยน `capacity()` | ใช่ (ขั้นต่ำ n) | อาจใช่ (ถ้า n > capacity เดิม) |
| เปลี่ยน `size()` | **ไม่** — ยังคงเท่าเดิม | **ใช่** — บังคับให้ size เป็น n |
| สมาชิกใหม่ | ไม่มีการสร้าง element ใหม่ | สร้าง element ใหม่ (default-constructed หรือค่าที่ระบุ) ถ้า n มากกว่า size เดิม |
| ใช้เมื่อไหร่ | รู้จำนวนสูงสุดที่จะ push_back แต่ยังไม่อยากมีสมาชิกจริง | ต้องการให้ vector มีสมาชิกจริง n ตัวทันที |

```cpp
std::vector<int> a;
a.reserve(10);        // a.size() == 0, a.capacity() >= 10 — เข้าถึง a[0] ยังคง UB!

std::vector<int> b;
b.resize(10);         // b.size() == 10, ทุกตัวเป็น 0 (ค่า default ของ int) — a[0] ใช้ได้ทันที
```

> **กฎทองข้อสำคัญ**: ถ้ารู้จำนวนข้อมูลล่วงหน้า (เช่น อ่านจากไฟล์ที่รู้จำนวนบรรทัด หรือรับ
> ค่า `n` จาก user ก่อนแล้วค่อยวน loop `push_back`) ควร `reserve()` เสมอ เพราะช่วยลด
> ทั้งจำนวน reallocation และลด memory fragmentation ได้จริงในโปรแกรมขนาดใหญ่

---

## 59.6 Iterator Invalidation: ปัญหาคลาสสิกที่พบบ่อยที่สุด (Step 470)

**Iterator Invalidation** คือสถานการณ์ที่ iterator (หรือ pointer/reference ไปยังสมาชิกของ
vector) **ใช้งานต่อไม่ได้อีกต่อไป** หลังจากมีการดำเนินการบางอย่างกับ vector — แต่ compiler
**ไม่เตือน** เรื่องนี้เลย เพราะมันไม่ใช่ syntax error แต่เป็น **Undefined Behavior ตอน runtime**

### กรณีที่ 1: erase ระหว่าง loop (บั๊กที่พบบ่อยที่สุด)

```cpp
// *** ตัวอย่างนี้มีบั๊กโดยตั้งใจ (Undefined Behavior) เพื่อสาธิต iterator invalidation ***
#include <iostream>
#include <vector>

int main() {
    std::vector<int> nums = {1, 2, 3, 4, 5, 6};

    for (auto it = nums.begin(); it != nums.end(); ++it) {
        if (*it % 2 == 0) {
            nums.erase(it);   // BUG: erase ทำให้ it (และ iterator หลังจากนี้) invalid ทันที
                              // การ ++it ในรอบถัดไปของ for-loop จึงเป็น Undefined Behavior
        }
    }

    for (int n : nums) {
        std::cout << n << ' ';
    }
    std::cout << '\n';

    return 0;
}
```

โค้ดนี้ **คอมไพล์ผ่านโดยไม่มี warning แม้แต่ตัวเดียว** แต่เมื่อรันจริงบนเครื่องทดสอบ
(GCC 13 + libstdc++, Linux) จะได้ผลลัพธ์:

```bash
$ g++ -Wall -Wextra -Wpedantic -std=c++17 buggy.cpp -o buggy
$ ./buggy
Segmentation fault (core dumped)
```

**โปรแกรม crash ทันที** เพราะ `erase(it)` ทำให้ `it` กลายเป็น iterator ที่ไม่ถูกต้อง (invalid)
แล้ว compound statement `++it` ในหัว for-loop จึงพยายามขยับ iterator ที่ใช้งานไม่ได้แล้ว
— นี่คือ Undefined Behavior ที่แท้จริง: ผลลัพธ์อาจเป็น segfault (อย่างที่เห็น), อาจข้ามสมาชิกไป
เงียบๆ โดยไม่ crash แต่ได้ผลลัพธ์ผิด, หรืออาจ "ดูเหมือนทำงานถูก" บนเครื่องหนึ่งแต่พังบนอีก
เครื่องหนึ่งก็ได้ — นี่คือสิ่งที่ทำให้บั๊กประเภทนี้อันตรายมาก เพราะทดสอบผ่านในเครื่อง dev
แต่ไปพังที่ production

### เหตุผลเบื้องหลัง: ทำไม erase ถึงทำให้ iterator invalid

เมื่อ `erase(it)` ลบสมาชิกที่ตำแหน่ง `it` ออก vector ต้อง **shift สมาชิกทุกตัวที่อยู่หลังจากนั้น
มาเติมช่องว่าง** (ย้ายข้อมูลจริงในหน่วยความจำ) ทำให้ iterator ทุกตัวที่ชี้ไปยังตำแหน่งตั้งแต่
จุดที่ลบเป็นต้นไป **ชี้ผิดตำแหน่งไปในทันที** (มาตรฐาน C++ กำหนดว่า iterator ที่ตำแหน่งที่ลบ
และหลังจากนั้นทั้งหมดถือว่า invalid)

### วิธีแก้ที่ถูกต้อง: ใช้ค่าที่ erase() คืนกลับมา

`erase()` **คืนค่าเป็น iterator ตัวใหม่ที่ยังใช้งานได้** ชี้ไปยังสมาชิกตัวถัดจากตัวที่ถูกลบ
เราต้องใช้ค่านี้แทนการ `++it` เอง:

```cpp
#include <iostream>
#include <vector>

int main() {
    std::vector<int> nums = {1, 2, 3, 4, 5, 6};

    // วิธีที่ถูกต้อง: ใช้ค่าที่ erase() คืนกลับมาเป็น iterator ตัวถัดไปเสมอ
    for (auto it = nums.begin(); it != nums.end(); /* ไม่ ++it ตรงนี้ */) {
        if (*it % 2 == 0) {
            it = nums.erase(it);   // erase คืน iterator ที่ชี้ไปยังสมาชิกถัดจากตัวที่ถูกลบ
        } else {
            ++it;
        }
    }

    for (int n : nums) {
        std::cout << n << ' ';
    }
    std::cout << '\n';

    return 0;
}
```

ผลลัพธ์ (ถูกต้อง):

```
1 3 5 
```

> **หมายเหตุ**: ตั้งแต่ C++20 มีฟังก์ชัน `std::erase_if` (ใน `<vector>`) ที่ทำงานนี้ให้ในบรรทัด
> เดียวและปลอดภัยกว่า: `std::erase_if(nums, [](int n){ return n % 2 == 0; });` — เราจะเจาะลึก
> เรื่อง lambda และ algorithm เหล่านี้ใน Part 63–64

### กรณีที่ 2: reallocation ทำให้ pointer/reference ค้าง (dangling)

Iterator invalidation ไม่ได้เกิดแค่ตอน erase/insert เท่านั้น — `push_back` ที่ทำให้เกิด
**reallocation** ก็ทำให้ **iterator, pointer, และ reference ทุกตัวที่ชี้เข้าไปใน vector เดิม
กลายเป็น dangling ทันที** เพราะหน่วยความจำก้อนเก่าถูกคืนกลับไปแล้ว:

```cpp
#include <iostream>
#include <vector>

int main() {
    std::vector<int> v;
    v.reserve(2);
    v.push_back(1);
    v.push_back(2);

    int* p_first = &v[0];
    std::cout << "ก่อน push_back ตัวที่ 3: capacity = " << v.capacity()
              << ", ที่อยู่ v[0] = " << static_cast<void*>(p_first) << '\n';

    v.push_back(3);   // capacity เต็มพอดี (2) จึงต้อง reallocate หน่วยความจำใหม่ทั้งก้อน

    std::cout << "หลัง push_back ตัวที่ 3: capacity = " << v.capacity()
              << ", ที่อยู่ v[0] ใหม่ = " << static_cast<void*>(&v[0]) << '\n';
    std::cout << "p_first ยังชี้ไปที่หน่วยความจำเก่าที่ถูกคืนไปแล้ว "
              << "(dangling pointer) -- ห้าม dereference ต่อ!\n";

    return 0;
}
```

ผลลัพธ์:

```
ก่อน push_back ตัวที่ 3: capacity = 2, ที่อยู่ v[0] = 0x556c73dc62b0
หลัง push_back ตัวที่ 3: capacity = 4, ที่อยู่ v[0] ใหม่ = 0x556c73dc72e0
p_first ยังชี้ไปที่หน่วยความจำเก่าที่ถูกคืนไปแล้ว (dangling pointer) -- ห้าม dereference ต่อ!
```

ที่อยู่หน่วยความจำเปลี่ยนไปจริง — `p_first` กลายเป็น dangling pointer ทันทีที่เกิด
reallocation แม้โค้ดจะดูเหมือนไม่มีอะไรผิดปกติก็ตาม

### กฎการจำ Iterator/Pointer/Reference Invalidation ของ vector

| การกระทำ | ผลต่อ iterator/pointer/reference |
|---|---|
| `push_back` / `insert` (ไม่เกิด reallocation) | ตัวที่อยู่**ก่อน**ตำแหน่งที่แทรกยังใช้ได้ ตัวที่อยู่ตั้งแต่ตำแหน่งที่แทรกเป็นต้นไป invalid |
| `push_back` / `insert` (**เกิด** reallocation) | **ทุกตัว invalid หมด** ไม่มีข้อยกเว้น |
| `erase` | ตัวที่อยู่**ก่อน**ตำแหน่งที่ลบยังใช้ได้ ตัวที่อยู่ตั้งแต่ตำแหน่งที่ลบเป็นต้นไป invalid |
| `pop_back` | iterator/pointer/reference ไปยัง `back()` เดิม invalid ตัวอื่นยังใช้ได้ |
| `clear()` | **ทุกตัว invalid หมด** |
| `reserve()` (ถ้าเปลี่ยน capacity จริง) | **ทุกตัว invalid หมด** (เพราะเกิด reallocation) |

> **แนวปฏิบัติที่ปลอดภัยที่สุด**: อย่าเก็บ iterator/pointer/reference ของ vector ไว้ใช้ข้าม
> การเรียก method ที่แก้ไข vector เด็ดขาด ถ้าจำเป็นต้องอ้างอิงตำแหน่งข้ามเวลา ให้เก็บเป็น
> **index (size_t)** แทน เพราะ index ไม่มีวัน invalidate (ตราบใดที่ index นั้นยังอยู่ในขอบเขต)

---

## 59.7 std::array: Fixed-Size Array แบบ Modern C++ (Step 471)

`std::array<T, N>` (จาก header `<array>`) คือ wrapper รอบ C array แบบดั้งเดิม (`T arr[N]`)
ที่เพิ่ม interface แบบ STL container เข้ามา โดยที่**ขนาด N ต้องรู้ตอน compile-time เสมอ**
(เป็น template parameter) และ**ไม่สามารถเปลี่ยนขนาดได้หลังสร้าง** — ต่างจาก `std::vector`
ที่ขนาดเปลี่ยนแปลงได้แบบ dynamic

```cpp
#include <array>
#include <iostream>
#include <numeric>
#include <vector>

// std::array รู้ขนาดของตัวเองตอน compile-time จึงส่งผ่านฟังก์ชันโดยรู้ size() ได้เสมอ
// ต่างจาก C array ที่ decay เป็น pointer แล้ว "ลืม" ขนาดของตัวเองทันที
double average(const std::array<int, 5>& arr) {
    return std::accumulate(arr.begin(), arr.end(), 0.0) / arr.size();
}

void print_c_array(const int* arr, std::size_t n) {
    // ฟังก์ชันที่รับ C array ต้องรับขนาด (n) แยกต่างหากเสมอ เพราะ arr decay เหลือแค่ pointer
    for (std::size_t i = 0; i < n; ++i) {
        std::cout << arr[i] << ' ';
    }
    std::cout << '\n';
}

int main() {
    std::array<int, 5> fixed_scores = {90, 85, 77, 88, 95};

    std::cout << "std::array size() = " << fixed_scores.size() << '\n';
    std::cout << "ค่าเฉลี่ย = " << average(fixed_scores) << '\n';

    // C array แบบดั้งเดิม
    int c_scores[5] = {90, 85, 77, 88, 95};
    std::cout << "sizeof(c_scores) = " << sizeof(c_scores)
              << " bytes, sizeof(int) = " << sizeof(int)
              << " -> จำนวนสมาชิก = " << sizeof(c_scores) / sizeof(c_scores[0]) << '\n';
    print_c_array(c_scores, 5);

    // std::array รองรับ range-based for, .begin()/.end(), เปรียบเทียบด้วย == ได้ตรงๆ
    std::array<int, 5> another = {90, 85, 77, 88, 95};
    std::cout << "fixed_scores == another: " << std::boolalpha
              << (fixed_scores == another) << '\n';

    // std::array ก็อปปี้ทั้งก้อนได้ตรงไปตรงมาด้วย operator=
    std::array<int, 5> copy_of_scores = fixed_scores;
    copy_of_scores[0] = 100;
    std::cout << "fixed_scores[0] = " << fixed_scores[0]
              << ", copy_of_scores[0] = " << copy_of_scores[0] << '\n';

    // std::array อยู่บน stack เหมือน C array (ไม่มี heap allocation ซ่อนอยู่)
    // ต่างจาก std::vector ที่ตัว object เก็บแค่ pointer/size/capacity บน stack
    // แต่ข้อมูลจริงอยู่บน heap เสมอ
    std::cout << "sizeof(std::array<int,5>) = " << sizeof(std::array<int, 5>) << '\n';
    std::cout << "sizeof(std::vector<int>)  = " << sizeof(std::vector<int>) << '\n';

    return 0;
}
```

ผลลัพธ์:

```
std::array size() = 5
ค่าเฉลี่ย = 87
sizeof(c_scores) = 20 bytes, sizeof(int) = 4 -> จำนวนสมาชิก = 5
90 85 77 88 95 
fixed_scores == another: true
fixed_scores[0] = 90, copy_of_scores[0] = 100
sizeof(std::array<int,5>) = 20
sizeof(std::vector<int>)  = 24
```

สังเกตบรรทัดสุดท้าย: `sizeof(std::array<int,5>)` เท่ากับ 20 ไบต์พอดี (5 × 4 ไบต์ของ `int`)
— **เหมือน C array เป๊ะ ไม่มี overhead เพิ่มเลย** ในขณะที่ `sizeof(std::vector<int>)` เท่ากับ
24 ไบต์เสมอ **ไม่ว่า vector จะมีสมาชิกกี่ตัวก็ตาม** เพราะ vector object เก็บแค่ 3 อย่าง
(pointer ไปยังข้อมูลจริง, size, capacity) ส่วนข้อมูลจริงอยู่บน heap แยกต่างหาก

### ทำไม std::array ดีกว่า C array แบบดั้งเดิม

1. **รู้ขนาดตัวเองเสมอ**: `arr.size()` ใช้ได้ทันที ไม่ต้องคำนวณ `sizeof(arr) / sizeof(arr[0])`
   ที่พลาดได้ง่ายเวลา array decay เป็น pointer เมื่อส่งผ่านฟังก์ชัน
2. **ไม่ decay เป็น pointer**: ส่งผ่านฟังก์ชันโดยรักษาข้อมูลขนาดไว้ได้เสมอ (เห็นได้จาก
   `average(const std::array<int, 5>&)` ที่รู้ว่ามี 5 ตัวแน่นอน)
3. **มี iterator (`begin()`/`end()`)**: ใช้กับ algorithm ของ STL (`std::accumulate`,
   `std::sort` ฯลฯ) ได้ตรงๆ เหมือน container อื่น
4. **เปรียบเทียบด้วย `==` ได้ตรงๆ**: C array เทียบด้วย `==` จะเป็นการเทียบ pointer (ที่อยู่)
   ไม่ใช่เทียบเนื้อหา แต่ `std::array` เทียบเนื้อหาจริงตามที่คาดหวัง
5. **Copy ได้ตรงไปตรงมาด้วย `=`**: C array copy ทั้งก้อนด้วย `=` ไม่ได้ (ต้องใช้ `memcpy`
   หรือ loop เอง) แต่ `std::array` ใช้ `operator=` ได้เหมือน object ทั่วไป
6. **ตรวจสอบขอบเขตได้ด้วย `.at()`**: เหมือน `vector::at()` — throw `std::out_of_range`
   เมื่อ index เกิน

### ข้อจำกัดที่ std::array ยังคงมีเหมือน C array

- **ขนาดต้องรู้ตอน compile-time**: `std::array<int, n>` โดยที่ `n` เป็นตัวแปรที่รู้ค่าตอน
  runtime เท่านั้น **ทำไม่ได้** (compile error) — ถ้าต้องการขนาดที่กำหนดตอน runtime
  ต้องใช้ `std::vector`
- **อยู่บน stack**: ถ้าขนาดใหญ่เกินไป (เช่น `std::array<double, 10000000>`) อาจทำให้เกิด
  **Stack Overflow** เหมือน C array ขนาดใหญ่บน stack — กรณีแบบนี้ควรใช้ `std::vector`
  (ซึ่งเก็บข้อมูลจริงบน heap) แทน

---

## 59.8 เลือกใช้ vector, array หรือ C array เมื่อไหร่ (Step 472)

| สถานการณ์ | ควรใช้ |
|---|---|
| ไม่รู้จำนวนสมาชิกล่วงหน้า / จำนวนเปลี่ยนแปลงตอน runtime | `std::vector` |
| รู้จำนวนสมาชิกแน่นอนตอน compile-time และไม่เปลี่ยนแปลงอีกเลย (เช่น พิกัด 3 มิติ, วันในสัปดาห์ 7 วัน, เลขทะเบียนบัตร 13 หลัก) | `std::array` |
| ต้องการ interoperate กับ C API เก่า (`extern "C"` function ที่รับ `T*`) | `std::vector::data()` หรือ `std::array::data()` — ทั้งคู่ให้ raw pointer ที่ compatible กับ C ได้ |
| เขียนโค้ดใหม่ในโปรเจกต์ C++ ปกติทั่วไป | หลีกเลี่ยง C array แบบดั้งเดิม (`int arr[N]`) ให้มากที่สุด — ใช้ `std::vector`/`std::array` แทนเสมอตาม C++ Core Guidelines |
| ข้อมูลขนาดใหญ่มาก (นับล้าน element) ที่ขนาดคงที่ตอน compile-time | `std::array` ก็ยังใช้ได้ แต่ระวัง stack overflow ถ้าประกาศเป็น local variable — พิจารณาใช้ `std::vector` หรือ `static`/heap แทน |
| ต้องการ push/pop/insert/erase สมาชิกระหว่างการทำงาน | `std::vector` เท่านั้น (`std::array` ไม่มี method เหล่านี้เพราะขนาดคงที่) |

### สรุปเป็นหลักคิดง่ายๆ

> **ถ้าขนาดเปลี่ยนได้ → `std::vector` | ถ้าขนาดตายตัวรู้ล่วงหน้า → `std::array` |
> ถ้าไม่มีเหตุผลพิเศษจริงๆ → อย่าใช้ C array ดิบๆ**

นี่คือ default mindset ที่วิศวกร C++ มืออาชีพในปี 2026 ใช้กันเป็นมาตรฐาน — C array แบบดั้งเดิม
ยังคงมีที่ใช้ (เช่น embedded programming ที่ห้ามมี heap allocation เลย, หรือ interfacing
กับ hardware driver) แต่สำหรับโค้ด application ทั่วไป `std::vector` และ `std::array` ปลอดภัย
กว่าและอ่านง่ายกว่าเสมอ

---

## ส่วนเสริม: กรณีพิเศษของ std::vector\<bool\>

ก่อนจะไปหัวข้อข้อผิดพลาดที่พบบ่อย มีข้อยกเว้นหนึ่งที่ควรรู้ไว้เพราะเป็นคำถามสัมภาษณ์งาน
ยอดฮิต: **`std::vector<bool>` ไม่ได้ทำงานเหมือน `vector<T>` ชนิดอื่นเลย**

```cpp
#include <iostream>
#include <type_traits>
#include <vector>

int main() {
    std::vector<bool> flags = {true, false, true};

    // v[0] ไม่ได้คืนค่า bool& จริงๆ แต่คืน std::vector<bool>::reference (proxy object)
    // เพราะ vector<bool> เก็บข้อมูลแบบ "บิตอัด" (1 บิตต่อ 1 ค่า) ไม่ใช่ 1 ไบต์ต่อค่าเหมือน bool ปกติ
    auto elem = flags[0];
    std::cout << "std::is_same<decltype(elem), bool>::value = "
              << std::boolalpha << std::is_same<decltype(elem), bool>::value << '\n';

    flags[1] = true;
    for (bool b : flags) {
        std::cout << b << ' ';
    }
    std::cout << '\n';

    std::cout << "sizeof(bool) = " << sizeof(bool) << " bytes ต่อค่า (ปกติ)\n";
    std::cout << "แต่ std::vector<bool> ใช้พื้นที่จริงประมาณ 1 บิตต่อค่า (bit-packed)\n";

    return 0;
}
```

ผลลัพธ์:

```
std::is_same<decltype(elem), bool>::value = false
true true true 
sizeof(bool) = 1 bytes ต่อค่า (ปกติ)
แต่ std::vector<bool> ใช้พื้นที่จริงประมาณ 1 บิตต่อค่า (bit-packed)
```

**เหตุผล**: `std::vector<bool>` เป็น **template specialization พิเศษ** ของ `vector` ที่ถูก
ออกแบบมาให้ประหยัดหน่วยความจำ โดยเก็บค่าจริงแบบ **บีบอัดทีละบิต** (1 บิตต่อ 1 ค่า `bool`)
แทนที่จะเก็บทีละไบต์แบบปกติ ผลข้างเคียงคือ `operator[]` ของมัน**ไม่สามารถคืนค่าเป็น
`bool&` จริงๆ ได้** (เพราะไม่มี "ที่อยู่ของบิตเดี่ยว" ในภาษา C++) จึงต้องคืนเป็น **proxy
object** (`std::vector<bool>::reference`) แทน ซึ่งทำให้โค้ดบางแบบที่ใช้ได้กับ
`vector<T>` ชนิดอื่น (เช่น การส่ง `bool&` ออกจากฟังก์ชัน หรือใช้กับ generic code ที่
คาดหวัง `T&`) ใช้ไม่ได้กับ `vector<bool>`

ด้วยเหตุนี้ C++ Core Guidelines และวิศวกรมืออาชีพจำนวนมากจึงแนะนำให้**หลีกเลี่ยง
`std::vector<bool>`** ในโค้ด generic หรือโค้ดที่ต้องการ reference จริง — ถ้าต้องการ array
ของค่า boolean จำนวนมากที่ต้องประหยัดหน่วยความจำจริงๆ ให้พิจารณา `std::bitset` (ถ้าขนาด
คงที่รู้ตอน compile-time) หรือ `std::deque<bool>` / `std::vector<char>` แทน

---

## ข้อผิดพลาดที่พบบ่อย (Common Pitfalls)

1. **ใช้ `operator[]` เข้าถึง index ที่เกินขอบเขต** — ไม่มี error หรือ warning ใดๆ ตอน compile
   หรือแม้แต่ตอน run (อาจ "ดูเหมือน" ทำงานได้เพราะไปอ่าน/เขียนหน่วยความจำที่ไม่ได้เป็นของ
   vector) นี่คือ Undefined Behavior แท้ๆ ถ้าไม่มั่นใจว่า index ถูกต้องเสมอ ให้ใช้ `.at()`
   แทนระหว่างพัฒนา/debug

2. **erase หรือ insert ระหว่าง range-based for loop หรือ loop ที่ใช้ iterator เดิมต่อ** —
   ตามที่แสดงในหัวข้อ 59.6 การ `erase`/`insert` ทำให้ iterator ตั้งแต่ตำแหน่งนั้นเป็นต้นไป
   invalid ทันที ต้องใช้ค่าที่ `erase()`/`insert()` คืนกลับมาเสมอ ไม่ใช่ `++it` เอง

3. **เก็บ pointer หรือ reference ไปยังสมาชิกของ vector ไว้ใช้ในภายหลัง แล้วมี `push_back`
   คั่นกลาง** — ถ้า `push_back` นั้นทำให้เกิด reallocation (เช่น `size() == capacity()`)
   pointer/reference เดิมทั้งหมดจะกลายเป็น dangling ทันที การ dereference ต่อคือ UB
   วิธีป้องกัน: เก็บเป็น **index** แทนเสมอถ้าต้องอ้างอิงข้ามการแก้ไข vector

4. **สับสนระหว่าง `size()` กับ `capacity()`** — เช่น เข้าใจผิดว่า `clear()` แล้ว `capacity()`
   จะกลับไปเป็น 0 (จริงๆ ไม่เปลี่ยน) หรือคิดว่า `reserve(n)` จะทำให้ `size()` เป็น n ทันที
   (จริงๆ `size()` ไม่เปลี่ยน ต้อง `push_back` เพิ่มเอง หรือใช้ `resize()` แทน)

5. **ประกาศ `std::array` ขนาดใหญ่มากเป็น local variable** เช่น
   `std::array<double, 5000000> buffer;` ใน function — จะทำให้เกิด **Stack Overflow**
   ทันทีที่ฟังก์ชันถูกเรียก เพราะ stack มีขนาดจำกัด (โดยทั่วไปไม่กี่ MB) ในขณะที่ข้อมูลนี้
   ต้องการหลายสิบ MB — กรณีข้อมูลขนาดใหญ่ควรใช้ `std::vector` ที่เก็บข้อมูลจริงบน heap

6. **ลืมว่า `insert`/`erase` ที่ตำแหน่งกลาง/หน้าของ vector เป็น O(n) ไม่ใช่ O(1)** — ถ้าโปรแกรม
   ต้อง insert/erase บ่อยๆ ที่ตำแหน่งอื่นที่ไม่ใช่ท้าย vector ควรพิจารณา `std::list` หรือ
   `std::deque` แทน (จะเรียนใน Part 60)

---

## แบบฝึกหัดท้ายบท

1. เขียนโปรแกรมที่รับจำนวนเต็มจาก terminal ทีละตัว (จบด้วยการพิมพ์ `-1`) เก็บใน
   `std::vector<int>` แล้วพิมพ์จำนวนตัวเลขทั้งหมด ผลรวม และค่าเฉลี่ย

2. เขียนฟังก์ชัน `void remove_odd_numbers(std::vector<int>& v)` ที่ลบสมาชิกที่เป็นเลขคี่
   ทั้งหมดออกจาก vector โดยใช้ erase-return idiom ที่ปลอดภัย (ห้ามใช้วิธีที่มี iterator
   invalidation)

3. เขียนโปรแกรมทดลองวัด growth factor ของ `std::vector<int>` บนเครื่องของตัวเอง
   (คล้ายตัวอย่างในหัวข้อ 59.3) แล้วลองอธิบายว่าทำไมตัวเลขที่ได้อาจไม่ตรงกับตัวอย่างในบทเรียน
   เป๊ะๆ ถ้าใช้ compiler/standard library คนละตัว (เช่น Clang กับ libc++, หรือ MSVC)

4. เขียนโปรแกรมเปรียบเทียบเวลาที่ใช้ `push_back` จำนวน 1,000,000 ครั้ง โดยมีเวอร์ชันที่
   `reserve()` ล่วงหน้า กับเวอร์ชันที่ไม่ `reserve()` เลย ใช้ `<chrono>` วัดเวลาและพิมพ์ผลต่าง

5. เขียน function template `T find_max(const std::array<T, N>& arr)` ที่หา ค่ามากที่สุด
   ใน `std::array` ชนิดใดก็ได้ที่รองรับ `operator>` โดยไม่ต้องรับพารามิเตอร์ขนาดแยกต่างหาก

6. อธิบายด้วยคำพูดของตัวเอง (เขียนเป็นคอมเมนต์ในโค้ด หรือเขียนแยกเป็นข้อความ) ว่าทำไม
   `std::array<int, n>` โดยที่ `n` เป็นตัวแปร (ไม่ใช่ constant expression) ถึง compile ไม่ผ่าน
   ลองเขียนโค้ดทดสอบจริงแล้วอ่าน error message ที่ compiler แจ้ง

### แนวทางเฉลยข้อ 1

```cpp
#include <iostream>
#include <numeric>
#include <vector>

int main() {
    std::vector<int> numbers;
    int input = 0;

    std::cout << "ป้อนจำนวนเต็ม (พิมพ์ -1 เพื่อจบ): ";
    while (std::cin >> input && input != -1) {
        numbers.push_back(input);
    }

    if (numbers.empty()) {
        std::cout << "ไม่มีข้อมูล\n";
        return 0;
    }

    long long sum = std::accumulate(numbers.begin(), numbers.end(), 0LL);
    double average = static_cast<double>(sum) / static_cast<double>(numbers.size());

    std::cout << "จำนวนตัวเลขทั้งหมด: " << numbers.size() << '\n';
    std::cout << "ผลรวม: " << sum << '\n';
    std::cout << "ค่าเฉลี่ย: " << average << '\n';

    return 0;
}
```

ทดสอบด้วย input `1 2 3 4 5 -1` จะได้:

```
จำนวนตัวเลขทั้งหมด: 5
ผลรวม: 15
ค่าเฉลี่ย: 3
```

จุดสำคัญ: ใช้ `std::accumulate` (จาก `<numeric>`) แทนการเขียน loop บวกเองเพื่อความกระชับ
และใช้ `0LL` (long long literal) เป็นค่าเริ่มต้นเพื่อป้องกัน overflow ถ้าผลรวมมีค่ามาก

### แนวทางเฉลยข้อ 2

```cpp
#include <iostream>
#include <vector>

void remove_odd_numbers(std::vector<int>& v) {
    for (auto it = v.begin(); it != v.end(); /* ตั้งใจไม่ ++it ตรงนี้ */) {
        if (*it % 2 != 0) {
            it = v.erase(it);
        } else {
            ++it;
        }
    }
}

int main() {
    std::vector<int> data = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10};

    remove_odd_numbers(data);

    std::cout << "หลังลบเลขคี่ทั้งหมด: ";
    for (int n : data) {
        std::cout << n << ' ';
    }
    std::cout << '\n';

    return 0;
}
```

ผลลัพธ์:

```
หลังลบเลขคี่ทั้งหมด: 2 4 6 8 10
```

จุดสำคัญของเฉลยนี้คือการ**ไม่ `++it` ในหัว for-loop** แต่ปล่อยให้การ `++it` เกิดขึ้นเฉพาะ
กรณีที่ **ไม่ได้ erase** เท่านั้น ส่วนกรณีที่ erase จะใช้ค่าที่ `erase()` คืนกลับมาแทนเสมอ
— แพทเทิร์นนี้ (erase-return idiom) เป็นวิธีมาตรฐานที่ปลอดภัยที่สุดสำหรับการลบสมาชิกที่
ตรงตามเงื่อนไขออกจาก vector ระหว่างวน loop

### แนวทางเฉลยข้อ 4

```cpp
#include <chrono>
#include <iostream>
#include <vector>

int main() {
    const int N = 1000000;

    auto start1 = std::chrono::steady_clock::now();
    std::vector<int> without_reserve;
    for (int i = 0; i < N; ++i) {
        without_reserve.push_back(i);
    }
    auto end1 = std::chrono::steady_clock::now();
    auto duration1 = std::chrono::duration_cast<std::chrono::microseconds>(end1 - start1);

    auto start2 = std::chrono::steady_clock::now();
    std::vector<int> with_reserve;
    with_reserve.reserve(N);
    for (int i = 0; i < N; ++i) {
        with_reserve.push_back(i);
    }
    auto end2 = std::chrono::steady_clock::now();
    auto duration2 = std::chrono::duration_cast<std::chrono::microseconds>(end2 - start2);

    std::cout << "ไม่ reserve: " << duration1.count() << " microseconds\n";
    std::cout << "reserve ล่วงหน้า: " << duration2.count() << " microseconds\n";

    return 0;
}
```

ผลลัพธ์จากการรันสองครั้งติดต่อกันบนเครื่องทดสอบ (คอมไพล์ด้วย `-O2`):

```
ไม่ reserve: 5518 microseconds
reserve ล่วงหน้า: 3530 microseconds
ไม่ reserve: 7041 microseconds
reserve ล่วงหน้า: 4246 microseconds
```

เวอร์ชันที่ `reserve()` ล่วงหน้าเร็วกว่าประมาณ 35–40% อย่างสม่ำเสมอในทุกการรัน แม้แต่ละครั้ง
ตัวเลขจริงจะแตกต่างกันไปตามภาระของเครื่องตอนนั้น (นี่คือธรรมชาติของการวัดเวลาแบบ wall-clock)
แต่แนวโน้มที่ `reserve()` เร็วกว่าเสมอเป็นสิ่งที่คาดการณ์ได้แน่นอน เพราะตัดขั้นตอน copy
ข้อมูลซ้ำๆ ระหว่าง reallocation ออกไปทั้งหมด

### แนวทางเฉลยข้อ 6

ลองคอมไพล์โค้ดนี้ดู:

```cpp
#include <array>
#include <iostream>

int main() {
    int n = 5;
    std::cin >> n;
    std::array<int, n> arr;   // ERROR: n ไม่ใช่ constant expression
    return 0;
}
```

```bash
$ g++ -Wall -Wextra -Wpedantic -std=c++17 ex6_fail.cpp -o ex6
ex6_fail.cpp: In function 'int main()':
ex6_fail.cpp:7:22: error: the value of 'n' is not usable in a constant expression
    7 |     std::array<int, n> arr;
      |                      ^
ex6_fail.cpp:5:9: note: 'int n' is not const
    5 |     int n = 5;
      |         ^
```

**คำอธิบาย**: พารามิเตอร์ตัวที่สองของ `std::array<T, N>` คือ `N` ต้องเป็น **template
non-type parameter** ซึ่งค่าของมันต้องรู้แน่นอนตอน **compile-time** (เรียกว่า
**constant expression**) เพราะ compiler ต้องใช้ค่า `N` นี้ในการคำนวณ**ขนาดของ object**
บน stack ตั้งแต่ตอนคอมไพล์ ก่อนที่โปรแกรมจะรันด้วยซ้ำ — แต่ในตัวอย่างข้างต้น `n` เป็น
ตัวแปรธรรมดาที่รับค่าจาก `std::cin` ซึ่งเป็นค่าที่**รู้ได้ตอน runtime เท่านั้น** compiler
จึงไม่สามารถกำหนดขนาดของ `arr` ได้ตั้งแต่ตอนคอมไพล์ ทำให้เกิด compile error ทันที

นี่คือความแตกต่างพื้นฐานที่สุดระหว่าง `std::array` กับ `std::vector`: ถ้าขนาดข้อมูลรู้ได้
เฉพาะตอน runtime (เช่น มาจาก user input, ไฟล์, หรือ network) **ต้องใช้ `std::vector` เท่านั้น**
`std::array` ใช้ไม่ได้เลยในสถานการณ์แบบนี้ ไม่ว่าจะพยายามอย่างไรก็ตาม

---

## สรุปท้ายบท

ใน Part นี้เราได้เจาะลึก `std::vector` และ `std::array` ซึ่งเป็น container พื้นฐานที่สุดและ
ถูกใช้บ่อยที่สุดของ STL:

- `std::vector` คือ dynamic array ที่จัดการหน่วยความจำให้อัตโนมัติ พร้อม method ครบครัน
  (`push_back`, `pop_back`, `insert`, `erase`, `size`, `capacity`)
- `size()` กับ `capacity()` เป็นคนละแนวคิดกัน — `capacity()` คือพื้นที่ที่จองไว้
  ซึ่งมักมากกว่า `size()` จริง
- Reallocation เกิดขึ้นเมื่อ `size() == capacity()` และมีการเพิ่มสมาชิกอีก โดยขยายขนาด
  แบบ exponential (คูณ ~2 เท่า) ทำให้ `push_back` ยังคงเป็น **amortized O(1)**
- `reserve()` ช่วยลดจำนวน reallocation ได้อย่างมีนัยสำคัญเมื่อรู้ขนาดข้อมูลล่วงหน้า
- **Iterator Invalidation** เป็นบั๊กคลาสสิกที่อันตรายที่สุดของ vector ทั้งจาก erase/insert
  ระหว่าง loop และจาก reallocation ที่ทำให้ pointer/reference ค้าง ต้องใช้ erase-return
  idiom เสมอและหลีกเลี่ยงการเก็บ pointer/reference ข้ามการแก้ไข vector
- `std::array` คือ fixed-size array ที่ปลอดภัยและสะดวกกว่า C array แบบดั้งเดิม โดยไม่มี
  overhead เพิ่มเติมเลย (`sizeof` เท่ากันเป๊ะ) เหมาะกับข้อมูลที่ขนาดรู้แน่นอนตอน compile-time

ใน **Part 60** เราจะเจาะลึก container ตระกูล linked-list ของ STL ได้แก่ `std::list`,
`std::deque`, และ `std::forward_list` พร้อมตารางเปรียบเทียบ complexity ของทุก operation
เพื่อให้ตัดสินใจเลือก container ที่เหมาะสมกับสถานการณ์จริงได้อย่างมั่นใจ

**ต่อไป:** [Part 60 — std::list, std::deque, std::forward_list](./part-060-list-deque-forwardlist.md)
