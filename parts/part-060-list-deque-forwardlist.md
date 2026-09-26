# Part 60: std::list, std::deque, std::forward_list (Step 473–480)

> Module E — Templates, Generic Programming และ STL | Part 60 จาก 125
> Step ที่ครอบคลุมใน Part นี้: Step 473–480
> Part ก่อนหน้า: [Part 59 — std::vector และ std::array แบบเจาะลึก](./part-059-vector-array.md) | Part ถัดไป: [Part 61 — std::map, std::set, unordered_map/set](./part-061-map-set.md)

## เป้าหมายการเรียนรู้ (Learning Objectives)

หลังจบ Part นี้ ผู้เรียนจะสามารถ:

1. ใช้งาน `std::list` (doubly linked list) ด้วย `push_front`, `push_back`, `insert`, `erase`
   ได้อย่างถูกต้อง และอธิบายได้ว่าทำไมทุก operation เหล่านี้เป็น O(1)
2. อธิบายได้ว่าทำไม `std::list` ไม่มี `operator[]` และไม่มี random access โดยทั่วไป
3. ใช้งาน `std::deque` (double-ended queue) และอธิบายโครงสร้างการจัดเก็บข้อมูลแบบ chunk
   ที่อยู่เบื้องหลัง พร้อมเปรียบเทียบกับ `std::vector`
4. ใช้งาน `std::forward_list` (singly linked list) ผ่าน `insert_after`/`erase_after`
   และอธิบายได้ว่าทำไมมันประหยัดหน่วยความจำกว่า `std::list`
5. อ่านและใช้ตารางเปรียบเทียบ complexity ของทุก operation หลักระหว่าง `vector`, `list`,
   `deque`, `forward_list` เพื่อวิเคราะห์ performance ของโค้ดจริงได้
6. ตัดสินใจเลือก container ที่เหมาะสมกับสถานการณ์การใช้งานจริงในโปรเจกต์

---

## 60.1 std::list: Doubly Linked List ของ STL (Step 473)

`std::list<T>` (จาก header `<list>`) คือ **Doubly Linked List** ที่ implement โดย STL ให้
เราใช้งานพร้อม interface มาตรฐาน แนวคิดตรงกับ Doubly Linked List ที่เราเขียนขึ้นเองด้วยมือ
ใน Part 19 ทุกประการ — แต่ละสมาชิก (node) เก็บ **pointer ไปยัง node ก่อนหน้าและ node ถัดไป**
พร้อมกับข้อมูลจริง ทำให้เดินหน้า-ถอยหลังได้ทั้งสองทิศทาง

```cpp
#include <iostream>
#include <iterator>
#include <list>

int main() {
    std::list<int> tasks = {10, 20, 30};

    std::cout << "จำนวนสมาชิก: " << tasks.size() << '\n';
    std::cout << "ตัวแรก (front): " << tasks.front() << '\n';
    std::cout << "ตัวสุดท้าย (back): " << tasks.back() << '\n';

    std::cout << "สมาชิกทั้งหมด: ";
    for (int t : tasks) {
        std::cout << t << ' ';
    }
    std::cout << '\n';

    // list ไม่มี operator[] และไม่มี .at() เพราะไม่รองรับ random access
    // ต้องเดินผ่าน iterator เท่านั้น
    auto it = tasks.begin();
    std::advance(it, 1);   // เดิน iterator ไปข้างหน้า 1 ตำแหน่ง -- O(n) สำหรับ list
    std::cout << "สมาชิกตัวที่ 2 (ผ่าน iterator): " << *it << '\n';

    return 0;
}
```

ผลลัพธ์:

```
จำนวนสมาชิก: 3
ตัวแรก (front): 10
ตัวสุดท้าย (back): 30
สมาชิกทั้งหมด: 10 20 30 
สมาชิกตัวที่ 2 (ผ่าน iterator): 20
```

`std::list` มี iterator ประเภท **Bidirectional Iterator** (เดินหน้า-ถอยหลังได้ด้วย `++`/`--`
แต่กระโดดข้ามหลายตำแหน่งในครั้งเดียวไม่ได้ เช่น `it + 5` ใช้ไม่ได้ ต้องใช้ `std::advance(it, 5)`
ซึ่งจะเดินทีละก้าว) เราจะเจาะลึกเรื่องประเภทของ iterator ทั้งหมดใน Part 62

---

## 60.2 push_front, push_back, insert, erase ของ list: ทำไม O(1) ทุกตำแหน่ง (Step 474)

จุดเด่นที่สุดของ `std::list` คือ **การ insert/erase ที่ตำแหน่งใดก็ตาม (ถ้ามี iterator ชี้ไปที่
ตำแหน่งนั้นอยู่แล้ว) เป็น O(1) เสมอ** ไม่ว่าจะเป็นหน้า กลาง หรือท้าย — ต่างจาก `vector` ที่
insert/erase ตรงกลางหรือหน้าต้อง O(n) เพราะต้อง shift ข้อมูล

```cpp
#include <algorithm>
#include <iostream>
#include <list>
#include <string>

void print_list(const std::string& label, const std::list<int>& lst) {
    std::cout << label << ": ";
    for (int x : lst) {
        std::cout << x << ' ';
    }
    std::cout << '\n';
}

int main() {
    std::list<int> lst = {2, 3, 4};

    lst.push_front(1);   // O(1) เสมอ -- ต่างจาก vector ที่ insert หน้าเป็น O(n)
    lst.push_back(5);    // O(1) เสมอ
    print_list("หลัง push_front(1) และ push_back(5)", lst);

    // หา iterator ไปยังค่า 3 แล้ว insert ก่อนหน้ามัน -- O(1) เพราะรู้ตำแหน่งแล้ว
    auto it = lst.begin();
    ++it; ++it;   // เลื่อนไปที่ตำแหน่งของค่า 3 (ลำดับตอนนี้: 1, 2, 3, 4, 5)
    lst.insert(it, 999);
    print_list("หลัง insert(999) ก่อนตำแหน่งของ 3", lst);

    // เก็บ iterator ไปยังสมาชิกตัวหนึ่งไว้ แล้วลบสมาชิกตัวอื่นออก
    auto it_keep = std::find(lst.begin(), lst.end(), 4);
    lst.erase(lst.begin());   // ลบสมาชิกตัวแรกออก (ค่า 1)
    lst.push_back(100);       // เพิ่มสมาชิกใหม่ที่ท้าย

    // it_keep ยังคงใช้งานได้ปกติ! นี่คือข้อดีสำคัญของ list เมื่อเทียบกับ vector
    std::cout << "it_keep ยังชี้ไปที่ค่า: " << *it_keep << " (ยังใช้งานได้ปกติ)\n";
    print_list("สุดท้าย", lst);

    // erase คืน iterator ตัวถัดไป เหมือน vector
    auto e_it = lst.begin();
    e_it = lst.erase(e_it);
    std::cout << "หลัง erase ตัวแรก, iterator ที่คืนมาชี้ไปที่: " << *e_it << '\n';

    return 0;
}
```

ผลลัพธ์:

```
หลัง push_front(1) และ push_back(5): 1 2 3 4 5 
หลัง insert(999) ก่อนตำแหน่งของ 3: 1 2 999 3 4 5 
it_keep ยังชี้ไปที่ค่า: 4 (ยังใช้งานได้ปกติ)
สุดท้าย: 2 999 3 4 5 100 
หลัง erase ตัวแรก, iterator ที่คืนมาชี้ไปที่: 999
```

### เหตุผลที่ operation เหล่านี้เป็น O(1)

การ `insert`/`erase` ของ `list` ที่ตำแหน่งที่มี iterator ชี้อยู่แล้ว **ไม่ต้อง shift ข้อมูล
ใดๆ เลย** — สิ่งที่ต้องทำมีแค่การ **ปรับ pointer ของ node ข้างเคียงให้ชี้ข้าม node ที่
แทรก/ลบ** เท่านั้น (เหมือนเทคนิคที่เราเขียนเองด้วยมือใน Part 19) ปริมาณงานคงที่ ไม่ขึ้นกับ
ขนาดของ list เลย

### กฎการ invalidate ของ list — ข้อดีที่เหนือกว่า vector อย่างชัดเจน

| การกระทำ | ผลต่อ iterator/pointer/reference ของสมาชิกอื่น |
|---|---|
| `insert` (ทุกตำแหน่ง) | **ไม่มีตัวไหน invalid เลย** แม้แต่ตัวเดียว |
| `erase` (ตำแหน่งใดก็ตาม) | invalid **เฉพาะตัวที่ถูกลบเท่านั้น** ตัวอื่นทั้งหมดยังใช้ได้ปกติ |
| `push_front` / `push_back` | ไม่มีตัวไหน invalid เลย |
| `pop_front` / `pop_back` | invalid เฉพาะตัวที่ถูกลบ |

นี่คือความแตกต่างที่สำคัญที่สุดเมื่อเทียบกับ `vector` (ดู Part 59 หัวข้อ 59.6) — ถ้าโปรแกรม
ต้องเก็บ iterator ไว้ใช้อ้างอิงตำแหน่งข้ามเวลานาน พร้อมกับมีการแก้ไข container บ่อยๆ
`list` ปลอดภัยกว่ามาก

---

## 60.3 ทำไม list ไม่มี Random Access (Step 475)

`std::list` **ไม่มี** `operator[]` และ**ไม่มี** `.at()` เลย ลองคอมไพล์ดู:

```cpp
#include <iostream>
#include <list>

int main() {
    std::list<int> lst = {10, 20, 30};
    std::cout << lst[1] << '\n';   // ERROR: list ไม่มี operator[]
    return 0;
}
```

```bash
$ g++ -Wall -Wextra -Wpedantic -std=c++17 no_random.cpp -o no_random
no_random.cpp: In function 'int main()':
no_random.cpp:6:21: error: no match for 'operator[]' (operand types are 'std::__cxx11::list<int>' and 'int')
    6 |     std::cout << lst[1] << '\n';   // ERROR: list ไม่มี operator[]
      |                     ^
```

**เหตุผล**: เพราะ node ของ `list` **กระจัดกระจายอยู่คนละที่ในหน่วยความจำ** (แต่ละ node
จองแยกกันด้วย `new` ตอน insert) ไม่มีสูตรคำนวณที่อยู่ของสมาชิกตัวที่ `i` ได้โดยตรงเหมือน
array (`base_address + i * sizeof(T)`) — การจะไปถึงสมาชิกตัวที่ `i` ได้ ต้อง**เดินตาม
pointer ทีละ node ตั้งแต่ตัวแรก (หรือตัวสุดท้าย) ไปเรื่อยๆ** ซึ่งมี complexity เป็น **O(n)**
ไม่ใช่ O(1)

มาตรฐาน C++ จึงตัดสินใจ**ไม่ให้** `list` มี `operator[]` เลย เพื่อป้องกันไม่ให้โปรแกรมเมอร์
เขียนโค้ดที่ดูเหมือนเร็ว (`lst[i]`) แต่จริงๆ ช้าเป็น O(n) โดยไม่รู้ตัว — ถ้าต้องการเข้าถึง
ตำแหน่งที่ `i` จริงๆ ต้องเขียนให้ชัดเจนด้วย `std::next`/`std::advance` เพื่อสื่อว่า
"นี่คือการเดินทีละก้าว มีต้นทุน":

```cpp
auto it = std::next(lst.begin(), i);   // ชัดเจนว่าต้องเดิน i ก้าว (O(i))
```

### เปรียบเทียบความเร็วจริง: vector vs list

```cpp
#include <chrono>
#include <iostream>
#include <iterator>
#include <list>
#include <vector>

int main() {
    const int N = 200000;
    std::vector<int> v(N, 1);
    std::list<int> l(N, 1);

    auto start1 = std::chrono::steady_clock::now();
    long long sum_v = 0;
    for (int i = 0; i < N; ++i) {
        sum_v += v[i];   // O(1) ต่อครั้ง เพราะ random access
    }
    auto end1 = std::chrono::steady_clock::now();

    auto start2 = std::chrono::steady_clock::now();
    long long sum_l = 0;
    for (auto it = l.begin(); it != l.end(); ++it) {
        sum_l += *it;   // O(1) ต่อครั้งเช่นกัน "ถ้าเดินทีละก้าว" (sequential) ไม่ใช่สุ่มตำแหน่ง
    }
    auto end2 = std::chrono::steady_clock::now();

    // เข้าถึงสมาชิกกลาง list ด้วย std::next ต้องเดินทีละก้าวจริงๆ -- O(n)
    auto start3 = std::chrono::steady_clock::now();
    auto mid_it = std::next(l.begin(), N / 2);
    auto end3 = std::chrono::steady_clock::now();

    auto start4 = std::chrono::steady_clock::now();
    int mid_v = v[N / 2];
    auto end4 = std::chrono::steady_clock::now();

    std::cout << "sum_v = " << sum_v << ", sum_l = " << sum_l << '\n';
    std::cout << "vector sequential loop: "
              << std::chrono::duration_cast<std::chrono::microseconds>(end1 - start1).count()
              << " us\n";
    std::cout << "list sequential loop:   "
              << std::chrono::duration_cast<std::chrono::microseconds>(end2 - start2).count()
              << " us\n";
    std::cout << "list std::next(mid):    "
              << std::chrono::duration_cast<std::chrono::microseconds>(end3 - start3).count()
              << " us (mid value = " << *mid_it << ")\n";
    std::cout << "vector v[mid]:          "
              << std::chrono::duration_cast<std::chrono::microseconds>(end4 - start4).count()
              << " us (mid value = " << mid_v << ")\n";

    return 0;
}
```

ผลลัพธ์ (คอมไพล์ด้วย `-O2`):

```
sum_v = 200000, sum_l = 200000
vector sequential loop: 103 us
list sequential loop:   640 us
list std::next(mid):    569 us (mid value = 1)
vector v[mid]:          0 us (mid value = 1)
```

สังเกตสองอย่างที่สำคัญ:

1. **`vector v[mid]` ใช้เวลาแทบ 0** เพราะเป็น O(1) แท้ๆ ในขณะที่ **`list std::next(mid)`**
   ใช้เวลานานถึง 569 microseconds เพราะต้องเดินทีละ node ถึง 100,000 ครั้ง (O(n))
2. แม้แต่การวน loop แบบ **sequential** (เดินทีละตัวตั้งแต่ต้นจนจบ ซึ่งทั้งคู่เป็น O(n) รวม
   ทางทฤษฎี) `vector` ก็ยังเร็วกว่า `list` ถึง ~6 เท่า! เหตุผลคือ **Cache Locality**
   (ที่เราจะเรียนลึกใน Part 87): ข้อมูลของ `vector` เรียงต่อเนื่องกันในหน่วยความจำ ทำให้ CPU
   Cache โหลดข้อมูลมาล่วงหน้าได้อย่างมีประสิทธิภาพ (prefetching) ในขณะที่ node ของ `list`
   กระจัดกระจายอยู่คนละที่ ทำให้เกิด **Cache Miss** บ่อยครั้ง แม้จำนวนครั้งของการเข้าถึงจะ
   เท่ากันทางทฤษฎี Big-O ก็ตาม

> **บทเรียนสำคัญ**: Big-O บอกแนวโน้มการเติบโตของงาน แต่ไม่ได้บอก "เวลาจริง" เสมอไป
> ในทางปฏิบัติ `vector` มักเร็วกว่า `list` แม้ในสถานการณ์ที่ทฤษฎีบอกว่า complexity เท่ากัน
> เพราะปัจจัยเรื่อง hardware cache มีผลมหาศาลกับโค้ดสมัยใหม่

---

## 60.4 std::deque: Double-Ended Queue (Step 476)

`std::deque<T>` (อ่านว่า "เด็ก" — ย่อจาก **D**ouble-**E**nded **QUE**ue, จาก header `<deque>`)
คือ container ที่รองรับการ **push/pop ได้เร็วทั้งสองด้าน (หน้าและหลัง)** พร้อมกับยังคงมี
**random access** (`operator[]`, `.at()`) เหมือน `vector`

```cpp
#include <deque>
#include <iostream>
#include <string>

void print_deque(const std::string& label, const std::deque<int>& dq) {
    std::cout << label << ": ";
    for (int x : dq) {
        std::cout << x << ' ';
    }
    std::cout << '\n';
}

int main() {
    std::deque<int> dq = {2, 3, 4};

    dq.push_front(1);   // O(1) amortized -- deque ทำได้ทั้งสองด้าน ต่างจาก vector!
    dq.push_back(5);    // O(1) amortized
    print_deque("หลัง push_front(1) และ push_back(5)", dq);

    dq.pop_front();      // O(1) -- ลบหน้าได้เร็ว ต่างจาก vector ที่ต้อง shift ทั้งก้อน
    dq.pop_back();       // O(1)
    print_deque("หลัง pop_front และ pop_back", dq);

    // deque มี operator[] และ at() เหมือน vector เพราะรองรับ random access
    std::cout << "dq[1] = " << dq[1] << '\n';
    std::cout << "dq.at(0) = " << dq.at(0) << '\n';

    // insert/erase ตรงกลาง deque ยังคงเป็น O(n) เหมือน vector (ไม่ใช่จุดเด่นของ deque)
    dq.insert(dq.begin() + 1, 999);
    print_deque("หลัง insert(999) ที่ index 1", dq);

    return 0;
}
```

ผลลัพธ์:

```
หลัง push_front(1) และ push_back(5): 1 2 3 4 5 
หลัง pop_front และ pop_back: 2 3 4 
dq[1] = 3
dq.at(0) = 2
หลัง insert(999) ที่ index 1: 2 999 3 4
```

### โครงสร้างการจัดเก็บข้อมูลเบื้องหลัง: Chunk-Based Storage

`vector` เก็บข้อมูลในหน่วยความจำก้อนเดียวที่ต่อเนื่องกันทั้งหมด ทำให้ `push_front` ต้อง
shift ทุกตัวไปทางขวา (O(n)) เสมอ — `deque` แก้ปัญหานี้ด้วยการเก็บข้อมูลเป็น **หลายก้อนย่อย
(chunk/block) ที่มีขนาดคงที่** โดยมี "แผนที่" (map — ไม่ใช่ `std::map`, แต่เป็นชื่อเรียก
internal structure) เก็บ pointer ไปยังแต่ละ chunk อีกที เมื่อต้อง `push_front` มันแค่จอง
chunk ใหม่เพิ่มด้านหน้า โดย**ไม่ต้อง shift ข้อมูลเดิมเลย** และเมื่อ `push_back` เต็ม chunk
ปัจจุบันก็แค่จอง chunk ใหม่ด้านหลังเพิ่ม

เราสามารถพิสูจน์การมีอยู่ของ chunk เหล่านี้ได้ด้วยการตรวจสอบที่อยู่หน่วยความจำจริงของ
สมาชิกที่ติดกัน:

```cpp
#include <cstdint>
#include <deque>
#include <iostream>

int main() {
    const int N = 300;
    std::deque<int> dq(N, 0);

    std::cout << "ระยะห่างระหว่างที่อยู่ของสมาชิกที่ติดกัน (bytes) ในช่วง index 0-299:\n";
    std::cout << "ปกติควรเป็น " << sizeof(int) << " ไบต์เท่ากันหมด (int ตัวถัดไปติดกัน)\n";
    std::cout << "ยกเว้นตอนข้ามขอบเขตของ chunk ซึ่งระยะห่างจะกระโดดผิดปกติ\n\n";

    for (int i = 1; i < N; ++i) {
        auto addr_prev = reinterpret_cast<std::uintptr_t>(&dq[i - 1]);
        auto addr_curr = reinterpret_cast<std::uintptr_t>(&dq[i]);
        std::intptr_t diff = static_cast<std::intptr_t>(addr_curr) - static_cast<std::intptr_t>(addr_prev);
        if (diff != static_cast<std::intptr_t>(sizeof(int))) {
            std::cout << "พบรอยต่อ chunk ที่ index " << i - 1 << " -> " << i
                      << " (ระยะห่าง = " << diff << " ไบต์ แทนที่จะเป็น " << sizeof(int) << ")\n";
        }
    }

    return 0;
}
```

ผลลัพธ์ (GCC 13 / libstdc++):

```
ระยะห่างระหว่างที่อยู่ของสมาชิกที่ติดกัน (bytes) ในช่วง index 0-299:
ปกติควรเป็น 4 ไบต์เท่ากันหมด (int ตัวถัดไปติดกัน)
ยกเว้นตอนข้ามขอบเขตของ chunk ซึ่งระยะห่างจะกระโดดผิดปกติ

พบรอยต่อ chunk ที่ index 127 -> 128 (ระยะห่าง = 20 ไบต์ แทนที่จะเป็น 4)
พบรอยต่อ chunk ที่ index 255 -> 256 (ระยะห่าง = 20 ไบต์ แทนที่จะเป็น 4)
```

ผลลัพธ์นี้ยืนยันว่า libstdc++ (ไลบรารีมาตรฐานของ GCC) แบ่งเก็บข้อมูล `int` เป็น chunk
ละ **128 ตัว** พอดี (เพราะ libstdc++ กำหนดขนาด chunk ไว้ที่ประมาณ 512 ไบต์ต่อ chunk และ
512 / 4 ไบต์ = 128) — ตัวเลขนี้เป็น **รายละเอียดการ implement เฉพาะของแต่ละไลบรารี**
(Clang/libc++ หรือ MSVC อาจใช้ขนาด chunk ต่างกัน) มาตรฐาน C++ **ไม่ได้กำหนดขนาด chunk**
ไว้ตายตัว เพียงรับประกันว่า `deque` ต้องมี complexity ตามที่กำหนดเท่านั้น

### กฎการ invalidate ของ deque

| การกระทำ | ผลต่อ iterator | ผลต่อ reference/pointer |
|---|---|---|
| `push_back` / `push_front` | **iterator ทุกตัว invalid หมด** | reference/pointer ไปยังสมาชิกเดิม **ยังใช้ได้** |
| `insert`/`erase` ที่ `begin()` หรือ `end()` พอดี | iterator ทุกตัว invalid | reference/pointer ยังใช้ได้ |
| `insert`/`erase` ที่ตำแหน่งอื่น (ตรงกลาง) | **iterator ทุกตัว invalid** | **reference/pointer ทุกตัว invalid ด้วย** |

จุดที่ต้องระวังเป็นพิเศษ: **`deque` invalidate iterator ทุกตัวง่ายกว่า `vector` มาก** —
แม้แต่การ `push_back` ที่ไม่ทำให้ `vector` invalidate อะไรเลย (ถ้ายังไม่เต็ม capacity)
ก็ยัง invalidate iterator **ทุกตัว** ของ `deque` เสมอ (แต่ reference/pointer ยังปลอดภัย)
เพราะโครงสร้าง chunk ภายในอาจถูกจัดเรียงใหม่ได้ทุกครั้งที่ขอบเขตเปลี่ยน

---

## 60.5 เปรียบเทียบ deque กับ vector (Step 477)

| หัวข้อ | `std::vector` | `std::deque` |
|---|---|---|
| การจัดเก็บข้อมูล | ก้อนเดียวต่อเนื่องกันทั้งหมด | หลาย chunk ที่ไม่ต่อเนื่องกัน |
| `push_back` | O(1) amortized | O(1) amortized |
| `push_front` | **O(n)** (ต้อง shift ทุกตัว) | **O(1) amortized** |
| `pop_front` | **O(n)** | **O(1)** |
| Random access `[]` | O(1) และเร็วมาก (contiguous) | O(1) แต่ช้ากว่า vector เล็กน้อย (ต้องคำนวณ 2 ชั้น: หา chunk แล้วหา offset ใน chunk) |
| Cache locality | ดีที่สุด (ข้อมูลต่อเนื่อง) | ดีในระดับ chunk แต่ไม่เท่า vector |
| ส่งเป็น C array ได้ไหม (`.data()`) | ได้ (`T*` ต่อเนื่องจริง) | **ไม่ได้** (ข้อมูลไม่ต่อเนื่องกันทั้งก้อน `deque` จึงไม่มี `.data()`) |
| ใช้เมื่อไหร่ | เป็นค่าเริ่มต้นที่ควรใช้ก่อนเสมอ | ต้องการ push/pop ทั้งสองด้านบ่อยๆ (คิว, sliding window) |

จุดสำคัญที่มักเข้าใจผิด: หลายคนคิดว่า `deque` "ดีกว่า" `vector` เพราะทำได้มากกว่า
(push ได้ทั้งสองด้าน) แต่ในความเป็นจริง **`vector` ยังคงเร็วกว่า `deque` ในการเข้าถึงข้อมูล
แบบ random access และการวน loop ตามลำดับ** เพราะข้อมูลต่อเนื่องกันจริงในหน่วยความจำ
(cache-friendly กว่า) ดังนั้นหลักการที่ถูกต้องคือ: **ใช้ `vector` เป็นค่าเริ่มต้นเสมอ
เปลี่ยนไปใช้ `deque` เฉพาะเมื่อต้องการ `push_front`/`pop_front` บ่อยๆ จริงๆ เท่านั้น**

---

## 60.6 std::forward_list: Singly Linked List ที่ประหยัดหน่วยความจำ (Step 478)

`std::forward_list<T>` (จาก header `<forward_list>`) คือ **Singly Linked List** — แต่ละ
node เก็บ pointer ไปยัง node **ถัดไปเพียงทิศทางเดียว** (ไม่มี pointer ย้อนกลับเหมือน `list`)
ทำให้ **ประหยัดหน่วยความจำต่อสมาชิกมากกว่า `list`** แต่แลกมาด้วยการ**เดินถอยหลังไม่ได้เลย**

```cpp
#include <forward_list>
#include <iostream>
#include <list>

int main() {
    std::forward_list<int> flist = {10, 20, 30};

    // forward_list ไม่มี size()! เพราะการนับต้องเดินทั้งลิสต์ O(n) มาตรฐานจึงตัดออก
    // เพื่อบังคับให้โปรแกรมเมอร์รู้ตัวว่า "การนับสมาชิกของ forward_list มีต้นทุน"
    int count = 0;
    for (int x : flist) {
        (void)x;
        ++count;
    }
    std::cout << "จำนวนสมาชิก (นับเอง): " << count << '\n';

    // ไม่มี push_back / back() เพราะเป็น singly-linked -- เดินย้อนกลับไม่ได้ จึงหา "ท้าย" ไม่ได้แบบ O(1)
    flist.push_front(5);
    std::cout << "front หลัง push_front(5): " << flist.front() << '\n';

    // insert_after / erase_after แทนที่ insert/erase ปกติ เพราะต้องมี pointer ไปโหนดก่อนหน้าเสมอ
    auto it = flist.before_begin();   // iterator พิเศษที่ชี้ "ก่อนตัวแรก"
    flist.insert_after(it, 0);
    std::cout << "หลัง insert_after(before_begin(), 0): ";
    for (int x : flist) {
        std::cout << x << ' ';
    }
    std::cout << '\n';

    // เปรียบเทียบขนาดหน่วยความจำต่อโหนด: forward_list มี pointer เดียว, list มี 2 pointer
    std::cout << "\nsizeof(std::forward_list<int>) = " << sizeof(std::forward_list<int>) << '\n';
    std::cout << "sizeof(std::list<int>)         = " << sizeof(std::list<int>) << '\n';

    return 0;
}
```

ผลลัพธ์:

```
จำนวนสมาชิก (นับเอง): 3
front หลัง push_front(5): 5
หลัง insert_after(before_begin(), 0): 0 5 10 20 30 

sizeof(std::forward_list<int>) = 8
sizeof(std::list<int>)         = 24
```

### ทำไมต้องใช้ insert_after / erase_after แทน insert / erase ปกติ

ในการ `insert`/`erase` ของ `list` ปกติ (doubly linked) เราสามารถ insert/erase "ก่อนหน้า"
iterator ที่ให้มาได้ เพราะแต่ละ node รู้ว่า node ก่อนหน้าตัวเองคือใคร (มี pointer ย้อนกลับ)
แต่ `forward_list` **ไม่มี pointer ย้อนกลับ** — ถ้าเรามี iterator ชี้ไปยัง node X และต้องการ
ลบ node X ออก เราจำเป็นต้องรู้ว่า **node ที่อยู่ก่อนหน้า X คือใคร** เพื่อปรับ pointer
ของมันให้ข้าม X ไป แต่ในเมื่อไม่มีทางเดินย้อนกลับจาก X ไปหา node ก่อนหน้าได้เลย (ต้องเดิน
จากตัวแรกใหม่ ซึ่งเสีย O(n)) มาตรฐาน C++ จึงออกแบบ interface ของ `forward_list` ให้ต้อง
ระบุ **"insert/erase หลังจาก iterator ที่ให้มา"** (`insert_after`/`erase_after`) แทน เพื่อ
รับประกันว่า operation เหล่านี้ยังคงเป็น O(1) เสมอ (ไม่ต้องเดินหา node ก่อนหน้า)

และเพื่อให้ insert/erase ที่**ตำแหน่งแรกสุด**ทำได้ด้วย จึงมี `before_begin()` ซึ่งเป็น
iterator พิเศษที่ชี้ไปยังตำแหน่ง "ก่อนตัวแรก" (dereference ไม่ได้ ใช้ได้แค่กับ `_after`
method เท่านั้น)

### ตัวอย่างการลบสมาชิกที่ตรงเงื่อนไขออกจาก forward_list

```cpp
#include <forward_list>
#include <iostream>

void remove_even_numbers(std::forward_list<int>& flist) {
    auto prev = flist.before_begin();   // iterator ไปยัง "ก่อนตัวแรก" เสมอ
    auto curr = flist.begin();

    while (curr != flist.end()) {
        if (*curr % 2 == 0) {
            curr = flist.erase_after(prev);   // ลบสมาชิกหลัง prev, curr กลายเป็นตัวถัดไปอัตโนมัติ
            // prev ไม่ต้องขยับ เพราะตัวที่อยู่ก่อนหน้ายังเป็นตัวเดิม
        } else {
            prev = curr;
            ++curr;
        }
    }
}

int main() {
    std::forward_list<int> flist = {1, 2, 3, 4, 5, 6, 7, 8};

    remove_even_numbers(flist);

    std::cout << "หลังลบเลขคู่ทั้งหมด: ";
    for (int x : flist) {
        std::cout << x << ' ';
    }
    std::cout << '\n';

    return 0;
}
```

ผลลัพธ์:

```
หลังลบเลขคู่ทั้งหมด: 1 3 5 7
```

สังเกตว่าต้องใช้ **สอง iterator เดินคู่กัน** (`prev` และ `curr`) เสมอเมื่อทำงานกับ
`forward_list` แบบนี้ — เป็น pattern มาตรฐานที่ต้องจำให้ขึ้นใจเมื่อเขียนโค้ดกับ singly
linked list ไม่ว่าจะเป็น STL หรือที่เขียนเอง

### เมื่อไหร่ควรใช้ forward_list แทน list

`forward_list` เหมาะกับสถานการณ์ที่**ต้องการประหยัดหน่วยความจำถึงที่สุด** และมีจำนวน
สมาชิกมหาศาล (หลักล้านขึ้นไป) โดยที่**ไม่ต้องการเดินย้อนกลับ**และ**ไม่ต้องการรู้จำนวน
สมาชิกบ่อยๆ** เช่น การ implement graph adjacency list ขนาดใหญ่ หรือ memory pool ภายใน
ระบบระดับต่ำ — ในโค้ด application ทั่วไป `std::list` ยังคงเป็นตัวเลือกที่ใช้บ่อยกว่ามาก
เพราะยืดหยุ่นกว่า (เดินสองทิศทางได้ มี `size()`, `back()`, `push_back()`)

---

## 60.7 ตารางเปรียบเทียบ Complexity ของทุก Operation (Step 479)

นี่คือตารางสรุปที่สำคัญที่สุดของ Part นี้ ควรจดจำให้ขึ้นใจ:

| Operation | `vector` | `deque` | `list` | `forward_list` |
|---|---|---|---|---|
| Random access `v[i]` | **O(1)** | O(1) (ช้ากว่า vector เล็กน้อย) | **ไม่มี** (ต้องเดิน O(n)) | **ไม่มี** (ต้องเดิน O(n)) |
| `push_back` | O(1) amortized | O(1) amortized | O(1) | **ไม่มี** method นี้ |
| `push_front` | **O(n)** | O(1) amortized | O(1) | O(1) (ผ่าน `push_front`) |
| `pop_back` | O(1) | O(1) | O(1) | **ไม่มี** method นี้ |
| `pop_front` | **O(n)** | O(1) | O(1) | O(1) |
| `insert`/`erase` ตรงกลาง (มี iterator แล้ว) | O(n) (shift ข้อมูล) | O(n) (shift ข้อมูล) | **O(1)** | **O(1)** (ผ่าน `_after`) |
| ค้นหาตำแหน่ง iterator ที่ต้องการ (จากค่า) | O(n) | O(n) | O(n) | O(n) |
| `size()` | O(1) | O(1) | O(1) | **ไม่มี** (ต้องนับเอง O(n)) |
| หน่วยความจำต่อสมาชิก (overhead) | ไม่มี overhead พิเศษ | overhead เล็กน้อยจากโครงสร้าง chunk | +2 pointers ต่อ node | +1 pointer ต่อ node |
| Cache locality | **ดีที่สุด** | ดี (ระดับ chunk) | แย่ (กระจัดกระจาย) | แย่ (กระจัดกระจาย) |
| Iterator เดินถอยหลังได้ไหม | ได้ (Random Access) | ได้ (Random Access) | ได้ (Bidirectional) | **ไม่ได้** (Forward เท่านั้น) |
| Iterator invalidate เมื่อ insert/erase ตรงกลาง | ทุกตัวตั้งแต่จุดนั้นเป็นต้นไป (หรือทั้งหมดถ้า reallocate) | ทุกตัว invalid | เฉพาะตัวที่ถูกลบเท่านั้น | เฉพาะตัวที่ถูกลบเท่านั้น |

> **ข้อควรระวัง**: ตารางนี้บอก **Time Complexity** (Big-O) เท่านั้น ในทางปฏิบัติ ปัจจัยเรื่อง
> Cache Locality (ดังที่แสดงในหัวข้อ 60.3) มักทำให้ `vector` เร็วกว่า `list`/`deque` ในหลาย
> สถานการณ์ แม้ทฤษฎีจะบอกว่า complexity เท่ากันหรือแย่กว่าก็ตาม — กฎทองคือ **วัดจริงเสมอ
> ก่อนสรุปว่า container ไหนเร็วกว่ากันในโค้ดของเรา** (จะเรียนเรื่อง benchmark อย่างเป็น
> ทางการใน Part 90)

---

## 60.8 แนวทางเลือก Container ให้ถูกตาม Use Case จริง (Step 480)

| สถานการณ์ใช้งานจริง | Container ที่แนะนำ | เหตุผล |
|---|---|---|
| เก็บข้อมูลทั่วไป ไม่รู้ pattern การใช้งานล่วงหน้า | `std::vector` | ค่าเริ่มต้นที่ดีที่สุดเสมอ เร็วและ cache-friendly ที่สุด |
| Queue (FIFO): เข้าคิวท้าย ออกคิวหน้า | `std::deque` | `push_back`/`pop_front` เป็น O(1) ทั้งคู่ |
| Stack (LIFO): เข้า-ออกด้านเดียวกัน | `std::vector` (ผ่าน `push_back`/`pop_back`) | เร็วที่สุดสำหรับ operation ที่ท้ายอย่างเดียว |
| Sliding Window ที่ต้อง push/pop ทั้งสองด้านบ่อยๆ | `std::deque` | ออกแบบมาสำหรับกรณีนี้โดยเฉพาะ |
| Insert/erase ตรงกลางบ่อยมาก โดยรู้ตำแหน่ง iterator อยู่แล้ว (เช่น implement LRU Cache, Text Editor buffer) | `std::list` | O(1) ต่อครั้ง ไม่ต้อง shift ข้อมูล |
| ข้อมูลจำนวนมหาศาล ต้องประหยัด memory ถึงที่สุด และไม่ต้องเดินถอยหลังหรือรู้ขนาดบ่อยๆ | `std::forward_list` | overhead ต่อ node น้อยที่สุด (1 pointer) |
| ต้องการส่ง raw pointer ต่อเนื่องให้ C API (`T*`, `.data()`) | `std::vector` | เป็น container เดียวที่รับประกันความต่อเนื่องแบบเต็มก้อนและมี `.data()` ที่ใช้กับ C ได้ตรงไปตรงมา |
| ขนาดคงที่รู้ตอน compile-time ไม่มีวันเปลี่ยน | `std::array` (จาก Part 59) | ไม่มี heap allocation เลย เร็วที่สุด |

### หลักคิดสรุปสั้นๆ ที่ใช้ได้ในเกือบทุกสถานการณ์

> **เริ่มต้นด้วย `std::vector` เสมอ** แล้วเปลี่ยนไปใช้ container อื่นก็ต่อเมื่อวัดผลจริง
> (profile) แล้วพบว่า `vector` เป็นคอขวด (bottleneck) ของโปรแกรม และรู้แน่ชัดว่า container
> อื่นจะแก้ปัญหานั้นได้จริง — นี่คือคำแนะนำที่ตรงกับ C++ Core Guidelines (SL.con.2:
> "Prefer using STL vector by default unless you have a reason to use a different container")
> และเป็นแนวทางที่วิศวกร C++ มืออาชีพส่วนใหญ่ยึดถือ

---

## ข้อผิดพลาดที่พบบ่อย (Common Pitfalls)

1. **ใช้ `lst[i]` กับ `std::list`** — compile error ทันที เพราะ `list` ไม่มี `operator[]`
   ถ้าจำเป็นต้องเข้าถึงตำแหน่งที่ `i` ต้องใช้ `std::next(lst.begin(), i)` แต่ควรถามตัวเองก่อน
   ว่าทำไมถึงต้องการ random access กับ `list` — อาจเป็นสัญญาณว่าควรใช้ `vector`/`deque`
   แทนตั้งแต่แรก

2. **เข้าใจผิดว่า `deque` เร็วกว่า `vector` เสมอเพราะทำได้มากกว่า** — ในความเป็นจริง
   `vector` มักเร็วกว่าในการเข้าถึงข้อมูลแบบ sequential/random เพราะข้อมูลต่อเนื่องกันจริง
   ควรใช้ `deque` เฉพาะเมื่อต้องการ `push_front`/`pop_front` บ่อยจริงๆ เท่านั้น

3. **คาดหวังว่า `forward_list` มี `size()`, `back()`, หรือ `push_back()`** — ไม่มีทั้งสาม
   เพราะการ implement ให้มีประสิทธิภาพ O(1) จริงทำไม่ได้ (การนับ `size()` ต้องเดินทั้งลิสต์
   O(n), การหา `back()` ก็เช่นกันเพราะเดินย้อนกลับจากท้ายไม่ได้) มาตรฐานเลือกตัดออกไปเลย
   แทนที่จะให้มี method ที่ดูเหมือนเร็วแต่จริงๆ ช้า

4. **ใช้ `insert`/`erase` ธรรมดากับ `forward_list`** — `forward_list` มีแค่ `insert_after`/
   `erase_after` เท่านั้น เพราะไม่มี pointer ย้อนกลับไปหา node ก่อนหน้า ต้องจำ pattern
   "เดินสอง iterator คู่กัน (`prev`, `curr`)" เสมอเมื่อต้องประมวลผลแบบมีเงื่อนไข

5. **ลืมว่า `deque` ไม่มี `.data()`** — เพราะข้อมูลไม่ต่อเนื่องกันทั้งก้อน (แบ่งเป็น chunk)
   จึงไม่มีวิธีคืนค่า `T*` ตัวเดียวที่ใช้เข้าถึงสมาชิกทั้งหมดแบบต่อเนื่องได้ ถ้าต้องการส่งข้อมูล
   ไปให้ C API ที่รับ `T*` ต้อง copy ไปยัง `vector` ก่อน หรือใช้ `vector` แทนตั้งแต่แรก

6. **คิดว่า `list::insert` ที่ตำแหน่งกลาง "เร็วเสมอ" โดยไม่นับเวลาหา iterator** — `insert`
   ของ `list` เป็น O(1) **ก็ต่อเมื่อมี iterator ชี้ไปยังตำแหน่งนั้นอยู่แล้ว** แต่ถ้าต้อง
   **หา** ตำแหน่งนั้นก่อนด้วยการเดิน (`std::find` หรือ `std::advance`) ขั้นตอนการหานั้นเป็น
   O(n) เสมอ รวมแล้วการ "insert ที่ตำแหน่งที่ n จากค่า" ของ `list` จึงยังคงเป็น O(n) โดยรวม
   ไม่ใช่ O(1) เหมือนที่หลายคนเข้าใจผิด — ข้อดี O(1) ของ `list` เกิดขึ้นเฉพาะเมื่อ**เดิน
   iterator ไปพร้อมกับ loop อยู่แล้ว** (ไม่ต้องหาใหม่)

---

## แบบฝึกหัดท้ายบท

1. เขียน class `TicketQueue` ที่ใช้ `std::deque<std::string>` เป็นตัวเก็บข้อมูลภายใน
   พร้อม method `enqueue(name)` (เข้าคิวท้าย), `dequeue()` (เรียกคิวจากหน้า, throw exception
   ถ้าคิวว่าง), และ `print_status()` (แสดงคิวปัจจุบัน)

2. เขียนโปรแกรมที่ใช้ `std::list<int>` แล้วลบสมาชิกที่มีค่าเท่ากับสมาชิกที่อยู่ **ก่อนหน้า
   ติดกัน** ออก (เช่น `{1, 1, 2, 3, 3, 3, 4}` กลายเป็น `{1, 2, 3, 4}`) โดยใช้ erase-return
   idiom ที่ปลอดภัยเหมือนที่เรียนใน Part 59 (ห้ามใช้ `std::unique` ที่มีให้สำเร็จรูปแล้ว
   ให้เขียน loop เอง เพื่อฝึกความเข้าใจ iterator ของ list)

3. เขียนโปรแกรมวัดเวลาเปรียบเทียบการ `insert` 5,000 ครั้งที่**ตำแหน่งกลางเดิม** ระหว่าง
   `std::vector` กับ `std::list` (สำหรับ list ให้หา iterator ตำแหน่งกลางไว้ล่วงหน้าครั้งเดียว
   แล้ว insert ซ้ำที่ iterator เดิม) ใช้ `<chrono>` วัดเวลาแล้วอธิบายผลลัพธ์ที่ได้

4. เขียนฟังก์ชัน `void remove_even_numbers(std::forward_list<int>& flist)` ที่ลบสมาชิก
   ที่เป็นเลขคู่ทั้งหมดออกจาก `forward_list` โดยใช้ `insert_after`/`erase_after` และ
   pattern เดิน iterator คู่ (`prev`, `curr`) ให้ถูกต้อง

5. เขียนฟังก์ชัน `std::size_t count_forward_list(const std::forward_list<int>& flist)`
   ที่นับจำนวนสมาชิกของ `forward_list` เอง (เพราะมันไม่มี `size()`) แล้ววิเคราะห์ว่า
   complexity ของฟังก์ชันนี้คือเท่าไหร่

6. สมมติสถานการณ์ 4 แบบต่อไปนี้ ให้เลือก container ที่เหมาะสมที่สุดพร้อมอธิบายเหตุผล
   สั้นๆ: (ก) ระบบจัดคิวลูกค้าธนาคารที่เรียกคิวจากหน้าและรับคิวใหม่ที่ท้ายตลอดเวลา
   (ข) โปรแกรม text editor ที่ต้องแทรก/ลบตัวอักษรตรงตำแหน่ง cursor บ่อยมาก
   (ค) เก็บพิกัด (x, y, z) ของจุดคงที่ 3 จุดที่ไม่มีวันเปลี่ยนแปลง
   (ง) เก็บ log ข้อความนับล้านบรรทัดที่อ่านตามลำดับอย่างเดียว ไม่มีการแทรก/ลบระหว่างทาง

### แนวทางเฉลยข้อ 1

```cpp
#include <deque>
#include <iostream>
#include <stdexcept>
#include <string>

class TicketQueue {
public:
    void enqueue(const std::string& name) {
        queue_.push_back(name);
    }

    std::string dequeue() {
        if (queue_.empty()) {
            throw std::runtime_error("คิวว่าง ไม่มีใครให้เรียก");
        }
        std::string front_person = queue_.front();
        queue_.pop_front();
        return front_person;
    }

    void print_status() const {
        std::cout << "คิวปัจจุบัน (" << queue_.size() << " คน): ";
        for (const auto& name : queue_) {
            std::cout << name << ' ';
        }
        std::cout << '\n';
    }

private:
    std::deque<std::string> queue_;
};

int main() {
    TicketQueue q;
    q.enqueue("A101");
    q.enqueue("A102");
    q.enqueue("A103");
    q.print_status();

    std::cout << "เรียกคิว: " << q.dequeue() << '\n';
    q.print_status();

    q.enqueue("A104");
    q.print_status();

    std::cout << "เรียกคิว: " << q.dequeue() << '\n';
    std::cout << "เรียกคิว: " << q.dequeue() << '\n';
    q.print_status();

    return 0;
}
```

ผลลัพธ์:

```
คิวปัจจุบัน (3 คน): A101 A102 A103 
เรียกคิว: A101
คิวปัจจุบัน (2 คน): A102 A103 
คิวปัจจุบัน (3 คน): A102 A103 A104 
เรียกคิว: A102
เรียกคิว: A103
คิวปัจจุบัน (1 คน): A104
```

จุดสำคัญ: เลือกใช้ `std::deque` เพราะต้องการทั้ง `push_back` (เข้าคิวท้าย) และ `pop_front`
(ออกคิวหน้า) ที่เป็น O(1) ทั้งคู่ — ถ้าใช้ `std::vector` แทน `pop_front` จะกลายเป็น O(n)
เพราะต้อง shift สมาชิกทุกตัว

### แนวทางเฉลยข้อ 4

```cpp
#include <forward_list>
#include <iostream>

void remove_even_numbers(std::forward_list<int>& flist) {
    auto prev = flist.before_begin();   // iterator ไปยัง "ก่อนตัวแรก" เสมอ
    auto curr = flist.begin();

    while (curr != flist.end()) {
        if (*curr % 2 == 0) {
            curr = flist.erase_after(prev);   // ลบสมาชิกหลัง prev, curr กลายเป็นตัวถัดไปอัตโนมัติ
            // prev ไม่ต้องขยับ เพราะตัวที่อยู่ก่อนหน้ายังเป็นตัวเดิม
        } else {
            prev = curr;
            ++curr;
        }
    }
}

int main() {
    std::forward_list<int> flist = {1, 2, 3, 4, 5, 6, 7, 8};

    remove_even_numbers(flist);

    std::cout << "หลังลบเลขคู่ทั้งหมด: ";
    for (int x : flist) {
        std::cout << x << ' ';
    }
    std::cout << '\n';

    return 0;
}
```

ผลลัพธ์:

```
หลังลบเลขคู่ทั้งหมด: 1 3 5 7
```

จุดสำคัญของเฉลยนี้: เมื่อ `*curr` เป็นเลขคู่ เราลบด้วย `erase_after(prev)` (ลบสมาชิก
**ที่อยู่หลัง** `prev` ซึ่งก็คือ `curr` นั่นเอง) และรับค่าที่คืนกลับมาเก็บไว้ที่ `curr` ทันที
โดย **ไม่ขยับ `prev`** เพราะสมาชิกที่ `prev` ชี้อยู่ยังคงเป็นตัวเดิม (ไม่ได้ถูกลบ) แต่เมื่อ
`*curr` ไม่ใช่เลขคู่ เราขยับทั้ง `prev` และ `curr` ไปพร้อมกัน — pattern การเดิน iterator
คู่แบบนี้เป็นสิ่งที่ต้องฝึกให้คล่องเมื่อทำงานกับ singly linked list ไม่ว่าจะเป็น `forward_list`
ของ STL หรือ linked list ที่เขียนขึ้นเอง

### แนวทางเฉลยข้อ 3

```cpp
#include <chrono>
#include <iostream>
#include <iterator>
#include <list>
#include <vector>

int main() {
    const int INITIAL_SIZE = 50000;
    const int INSERT_COUNT = 5000;

    std::vector<int> v(INITIAL_SIZE, 0);
    std::list<int> l(INITIAL_SIZE, 0);

    auto start_v = std::chrono::steady_clock::now();
    for (int i = 0; i < INSERT_COUNT; ++i) {
        v.insert(v.begin() + v.size() / 2, i);   // insert ตรงกลางทุกครั้ง -- ต้อง shift O(n)
    }
    auto end_v = std::chrono::steady_clock::now();

    auto mid_it = l.begin();
    std::advance(mid_it, l.size() / 2);   // หาตำแหน่งกลางครั้งเดียว

    auto start_l = std::chrono::steady_clock::now();
    for (int i = 0; i < INSERT_COUNT; ++i) {
        l.insert(mid_it, i);   // insert ที่ iterator เดิมซ้ำๆ -- O(1) ต่อครั้งเพราะรู้ตำแหน่งแล้ว
    }
    auto end_l = std::chrono::steady_clock::now();

    auto us_v = std::chrono::duration_cast<std::chrono::microseconds>(end_v - start_v).count();
    auto us_l = std::chrono::duration_cast<std::chrono::microseconds>(end_l - start_l).count();

    std::cout << "vector: insert " << INSERT_COUNT << " ครั้งที่กลาง vector ขนาด "
              << INITIAL_SIZE << " ใช้เวลา " << us_v << " us\n";
    std::cout << "list:   insert " << INSERT_COUNT << " ครั้งที่ iterator กลางเดิม ใช้เวลา "
              << us_l << " us\n";

    return 0;
}
```

ผลลัพธ์ (คอมไพล์ด้วย `-O2`):

```
vector: insert 5000 ครั้งที่กลาง vector ขนาด 50000 ใช้เวลา 12402 us
list:   insert 5000 ครั้งที่ iterator กลางเดิม ใช้เวลา 160 us
```

ผลต่างประมาณ **77 เท่า** — ยืนยันทฤษฎีในหัวข้อ 60.7 ได้อย่างชัดเจน: การ `insert` ตรงกลาง
`vector` ต้อง shift สมาชิกครึ่งหนึ่งของ vector ทุกครั้ง (O(n) ต่อครั้ง รวม O(n × k) เมื่อ
ทำ k ครั้ง) ในขณะที่ `list` ที่มี iterator ชี้ตำแหน่งอยู่แล้วใช้แค่การปรับ pointer เท่านั้น
(O(1) ต่อครั้ง รวม O(k)) ข้อสังเกตสำคัญ: การทดลองนี้ตั้งใจ**หา iterator ตำแหน่งกลางไว้
ล่วงหน้าเพียงครั้งเดียว** แล้วใช้ iterator ตัวเดิมซ้ำ — ถ้าต้องหาตำแหน่งใหม่ทุกครั้งด้วย
`std::advance` ผลลัพธ์จะไม่ต่างกันมากขนาดนี้ เพราะการ "หาตำแหน่ง" ของ `list` เองก็เป็น
O(n) เช่นกัน (ตามที่อธิบายไว้ในข้อผิดพลาดที่พบบ่อยข้อ 6)

### แนวทางเฉลยข้อ 6

| สถานการณ์ | Container ที่เลือก | เหตุผล |
|---|---|---|
| (ก) คิวธนาคาร เรียกคิวจากหน้า รับคิวใหม่ที่ท้าย | `std::deque` | ต้องการ `push_back` และ `pop_front` ที่เป็น O(1) ทั้งคู่ — ตรงกับจุดเด่นของ `deque` โดยตรง |
| (ข) Text editor แทรก/ลบตัวอักษรที่ตำแหน่ง cursor บ่อยมาก | `std::list` (หรือโครงสร้างขั้นสูงกว่าอย่าง Rope/Gap Buffer ในงานจริงระดับ production) | ถ้า cursor เดินไปพร้อมกับ iterator อยู่แล้ว การแทรก/ลบที่ตำแหน่งนั้นเป็น O(1) ด้วย `list` — ในทางปฏิบัติ editor ระดับ production มักใช้โครงสร้างที่ซับซ้อนกว่านี้ (Gap Buffer, Rope) แต่ `list` เป็นจุดเริ่มต้นที่เข้าใจง่ายกว่า `vector` ที่ต้อง shift ข้อมูลทุกครั้ง |
| (ค) พิกัด (x, y, z) คงที่ 3 จุด | `std::array<double, 3>` | ขนาดรู้แน่นอนตอน compile-time และไม่มีวันเปลี่ยน — `std::array` ไม่มี overhead ใดๆ เพิ่มเลย (จาก Part 59) |
| (ง) Log นับล้านบรรทัด อ่านตามลำดับอย่างเดียว | `std::vector` | ไม่มีการแทรก/ลบระหว่างทาง จึงไม่ต้องการจุดเด่นของ `list`/`deque` เลย ในขณะที่ `vector` ให้ cache locality ที่ดีที่สุดสำหรับการอ่านตามลำดับจำนวนมาก และประหยัดหน่วยความจำที่สุด (ไม่มี overhead ต่อ node เหมือน `list`) |

หลักการเบื้องหลังการเลือกทั้ง 4 ข้อคือการถามคำถามเดียวกันเสมอ: **"pattern การเข้าถึงและ
แก้ไขข้อมูลของโปรแกรมนี้คืออะไร"** แล้วจับคู่กับจุดแข็งของแต่ละ container ตามตารางใน
หัวข้อ 60.7 — ไม่ใช่เลือกจาก "container ไหนดูทันสมัยกว่า" หรือ "container ไหนมี method
เยอะกว่า"

---

## สรุปท้ายบท

ใน Part นี้เราได้เจาะลึก container ตระกูล linked-list และ double-ended ของ STL:

- `std::list` (doubly linked list) รองรับ `push_front`/`push_back`/`insert`/`erase` แบบ
  O(1) ทุกตำแหน่ง (เมื่อมี iterator ชี้อยู่แล้ว) แต่**ไม่มี random access** เพราะการเข้าถึง
  ตำแหน่งที่ `i` ต้องเดินทีละ node เสมอ (O(n))
- `std::deque` เก็บข้อมูลแบบแบ่งเป็น chunk ทำให้ `push_front`/`pop_front` เป็น O(1) เหมือน
  `push_back`/`pop_back` พร้อมยังคงมี random access ได้ แต่ไม่มี `.data()` และ cache
  locality สู้ `vector` ไม่ได้
- `std::forward_list` (singly linked list) ประหยัดหน่วยความจำต่อสมาชิกมากที่สุด (1 pointer
  ต่อ node) แต่เดินถอยหลังไม่ได้และไม่มี `size()`/`back()`/`push_back()` ต้องใช้
  `insert_after`/`erase_after` คู่กับ pattern การเดิน iterator สองตัว
- ตารางเปรียบเทียบ complexity เป็นเครื่องมือสำคัญที่ต้องใช้ทุกครั้งก่อนเลือก container
  แต่ต้องจำไว้เสมอว่า **Cache Locality** มีผลต่อความเร็วจริงมากพอๆ กับ (หรือมากกว่า)
  Big-O ในหลายสถานการณ์
- หลักคิดที่ใช้ได้ในเกือบทุกกรณี: **เริ่มต้นด้วย `vector` เสมอ แล้วค่อยเปลี่ยนเมื่อวัดผล
  จริงแล้วพบว่าจำเป็น**

ใน **Part 61** เราจะเข้าสู่ container ตระกูล **Associative** ได้แก่ `std::map`, `std::set`,
`std::unordered_map`, และ `std::unordered_set` ซึ่งใช้โครงสร้างข้อมูลแบบ red-black tree
และ hash table เบื้องหลัง เพื่อให้ค้นหาข้อมูลด้วย key ได้อย่างมีประสิทธิภาพ

**ต่อไป:** [Part 61 — std::map, std::set, unordered_map/set](./part-061-map-set.md)
