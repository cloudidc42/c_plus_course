# Part 58: แนะนำ STL (Step 457–464)

> Module E — Templates, Generic Programming และ STL | Part 58 จาก 125
> Step ที่ครอบคลุมใน Part นี้: Step 457–464
> Part ก่อนหน้า: [Part 57 — Template Specialization และ Variadic Template](./part-057-template-specialization.md) | Part ถัดไป: [Part 59 — std::vector และ std::array](./part-059-vector-array.md)

## เป้าหมายการเรียนรู้ (Learning Objectives)

หลังจบ Part นี้ ผู้เรียนจะสามารถ:

1. อธิบายได้ว่า **STL (Standard Template Library)** คืออะไร ประกอบด้วยส่วนประกอบหลักใดบ้าง
   และแต่ละส่วนทำงานร่วมกันอย่างไร
2. อธิบายเหตุผลเชิงวิศวกรรมซอฟต์แวร์ว่าทำไมควรใช้ STL แทนการเขียน data structure เองในงานจริง
   เกือบทุกกรณี พร้อมเปรียบเทียบ effort ระหว่างการเขียนเองกับการใช้ STL
3. จำแนกประเภทของ Container ใน STL ได้ทั้งสามกลุ่ม: **Sequence Container**,
   **Associative Container**, และ **Container Adapter**
4. เลือก Container ที่เหมาะสมกับสถานการณ์ต่างๆ ได้ โดยพิจารณาจาก pattern การใช้งานและ
   ความซับซ้อนเชิงเวลา (Time Complexity) ของแต่ละ operation
5. เข้าใจแนวคิดของ **Iterator** ในฐานะ "สะพาน" ที่เชื่อม Container เข้ากับ Algorithm โดยไม่ต้อง
   ให้ Algorithm รู้จักรายละเอียดภายในของ Container แต่ละชนิด
6. เข้าใจแนวคิดของ **Algorithm** และ **Functor/Function Object** ในฐานะเครื่องมือประมวลผลข้อมูล
   ที่ทำงานร่วมกับ Container และ Iterator ได้ทุกชนิด
7. เขียนตัวอย่างสั้นๆ ที่ใช้งาน Container แต่ละชนิดได้ เพื่อเตรียมความพร้อมสำหรับการเจาะลึก
   ทีละตัวใน Part 59-63

---

## 58.1 STL คืออะไร ประกอบด้วยอะไรบ้าง (Step 457)

**STL (Standard Template Library)** คือชุดไลบรารีมาตรฐานของภาษา C++ ที่ประกอบด้วย
**data structure** และ **algorithm** สำเร็จรูปที่เขียนด้วยหลักการ **Generic Programming**
ผ่าน Template (ที่เราเพิ่งเรียนจบไปใน Part 56-57) ทำให้ใช้งานได้กับชนิดข้อมูลใดก็ได้ โดยไม่ต้อง
เขียน data structure หรือ algorithm เหล่านี้ขึ้นมาเองตั้งแต่ต้น

STL ประกอบด้วย 4 ส่วนประกอบหลักที่ทำงานร่วมกันอย่างเป็นระบบ:

```
┌─────────────┐        ┌─────────────┐        ┌─────────────┐
│  Container  │◄──────►│  Iterator   │◄──────►│  Algorithm  │
│ (เก็บข้อมูล) │        │ (ตัวชี้เดินผ่าน) │        │ (ประมวลผล)  │
└─────────────┘        └─────────────┘        └──────┬──────┘
                                                        │ รับพฤติกรรมจาก
                                                        ▼
                                              ┌───────────────────┐
                                              │ Functor / Lambda   │
                                              │ (พฤติกรรมที่ปรับแต่งได้) │
                                              └───────────────────┘
```

1. **Container (ภาชนะเก็บข้อมูล)**: class template ที่เก็บกลุ่มของข้อมูล เช่น `std::vector`,
   `std::list`, `std::map` — เทียบเท่ากับ data structure ที่เราเขียนเองด้วยมือใน Module B
   (Linked List ใน Part 19, Stack/Queue ใน Part 20, Tree ใน Part 21, Hash Table ใน Part 22)
   แต่ผ่านการทดสอบ ปรับแต่ง performance และแก้บั๊กมาแล้วนับล้านครั้งโดยผู้เชี่ยวชาญทั่วโลก
2. **Iterator (ตัวชี้เดินผ่านข้อมูล)**: object ที่ทำหน้าที่ "เดิน" ผ่านสมาชิกใน container ทีละตัว
   โดยมี interface ที่**เหมือนกัน**ไม่ว่า container ข้างใต้จะเป็นชนิดใด (คล้าย pointer ที่เรา
   คุ้นเคยจาก Part 8-9) — Iterator คือกลไกสำคัญที่ทำให้ Algorithm ตัวเดียวใช้ได้กับ Container
   หลายสิบชนิดโดยไม่ต้องรู้จักรายละเอียดภายในของแต่ละ container เลย
3. **Algorithm (อัลกอริทึมสำเร็จรูป)**: function template ใน `<algorithm>` เช่น `std::sort`,
   `std::find`, `std::transform`, `std::accumulate` ที่ทำงานผ่าน Iterator เท่านั้น (ไม่ผูกติด
   กับ container ชนิดใดชนิดหนึ่ง) เทียบเท่ากับ Sorting/Searching Algorithm ที่เขียนเองใน
   Part 23-24 แต่ optimize มาอย่างดีและ generic กับทุก type/container
4. **Functor / Function Object**: object ที่ overload `operator()` ทำให้ "เรียกใช้งานได้เหมือน
   ฟังก์ชัน" ใช้เป็น "พฤติกรรมที่ปรับแต่งได้" ส่งเข้าไปให้ Algorithm (เช่น เกณฑ์การเรียงลำดับ
   ใน `std::sort`, เงื่อนไขการค้นหาใน `std::find_if`) ในโค้ด Modern C++ มักถูกแทนที่ด้วย
   **Lambda Expression** (จะเรียนเต็มรูปแบบใน Part 64) ที่เขียนได้กระชับกว่ามาก

ตัวอย่างที่แสดงทั้ง 4 ส่วนประกอบทำงานร่วมกัน:

```cpp
#include <algorithm>
#include <iostream>
#include <vector>

int main() {
    // 1) Container: เก็บข้อมูล
    std::vector<int> numbers = {5, 3, 8, 1, 9, 2};

    // 2) Algorithm: ประมวลผลข้อมูลในภาชนะ (ทำงานผ่าน Iterator ไม่สนใจว่าภาชนะคือ container ชนิดไหน)
    std::sort(numbers.begin(), numbers.end());

    // 3) Iterator: ตัวชี้ที่ใช้ "เดิน" ผ่านข้อมูลในภาชนะทีละตัว
    std::cout << "หลังเรียงลำดับ: ";
    for (std::vector<int>::iterator it = numbers.begin(); it != numbers.end(); ++it) {
        std::cout << *it << ' ';
    }
    std::cout << '\n';

    // 4) Functor / Function Object: ส่ง "พฤติกรรม" เข้าไปเป็น argument ให้ algorithm
    auto count = std::count_if(numbers.begin(), numbers.end(),
                                [](int n) { return n % 2 == 0; });
    std::cout << "จำนวนเลขคู่: " << count << '\n';

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 stl_components.cpp -o stl_components
./stl_components
```

```
หลังเรียงลำดับ: 1 2 3 5 8 9 
จำนวนเลขคู่: 2
```

สังเกตแพทเทิร์นสำคัญ: `numbers.begin()` และ `numbers.end()` คือ**คู่ Iterator ที่กำหนดขอบเขต**
ของข้อมูลที่ต้องการประมวลผล (`begin()` ชี้ไปยังสมาชิกตัวแรก, `end()` ชี้ไปยัง "ตำแหน่งถัดจาก
สมาชิกตัวสุดท้าย" ไม่ใช่ตัวสุดท้ายเอง) — แพทเทิร์นการส่งคู่ `begin()`/`end()` ให้ algorithm นี้
จะปรากฏซ้ำๆ **ทุกครั้ง**ตลอด Module E นี้ ควรจดจำให้แม่นตั้งแต่ตอนนี้

---

## 58.2 ทำไมควรใช้ STL แทนการเขียนเอง (Step 458)

นี่คือคำถามที่สำคัญที่สุดของ Part นี้: **ในเมื่อเราเขียน Linked List, Stack, Queue, Tree, Hash
Table เองมาแล้วทั้งหมดใน Module B ทำไมยังต้องเรียนรู้ STL อีก ทำไมไม่ใช้ของที่เขียนเองต่อไป?**

### เปรียบเทียบ Effort: Linked List ที่เขียนเองใน Part 19 vs `std::list`

ย้อนกลับไปดู Part 19 — การเขียน Doubly Linked List ให้ทำงานถูกต้องและปลอดภัยต้องมี:

- นิยาม `struct Node` พร้อม pointer `next`/`prev`
- ฟังก์ชัน `insertFront`, `insertBack`, `insertAt`, `removeFront`, `removeBack`, `removeAt`
- ฟังก์ชัน `traverse` (เดินหน้า) และ `traverseReverse` (เดินย้อนกลับ)
- การจัดการ memory ด้วยมือทั้งหมด (`malloc`/`free` ใน C หรือ `new`/`delete` ใน C++ยุคแรก)
  พร้อมระวัง memory leak และ dangling pointer อย่างเข้มงวด
- Edge case จำนวนมาก: ลบ node ตัวเดียวที่เหลืออยู่, แทรกที่ตำแหน่งแรก/ตำแหน่งสุดท้าย, list ว่าง
- ฟังก์ชัน destructor ที่ต้องวน loop ปลดปล่อยทุก node เพื่อไม่ให้เกิด memory leak

รวมแล้วโค้ด Linked List ที่ทำงานถูกต้องและปลอดภัยจริงมักมีความยาว **150-300+ บรรทัด**
และยังต้องผ่านการทดสอบอย่างละเอียดเพื่อให้มั่นใจว่าไม่มีบั๊กซ่อนอยู่ (memory leak, use-after-free,
off-by-one error ในการนับตำแหน่ง)

เทียบกับการใช้ `std::list` ที่มีมาให้พร้อมใช้งานทันที:

```cpp
#include <iostream>
#include <list>

int main() {
    // เทียบกับ Linked List ที่เขียนเองใน Part 19 (ต้องเขียน Node, head/tail pointer,
    // insertFront/insertBack/remove/traverse เองทั้งหมดหลายสิบบรรทัด และดูแล memory เอง)
    // std::list คือ Doubly Linked List ที่ทดสอบมาอย่างดีแล้ว ใช้งานได้ทันทีในบรรทัดเดียว
    std::list<std::string> playlist;
    playlist.push_back("เพลงที่ 1");
    playlist.push_back("เพลงที่ 2");
    playlist.push_front("เพลงที่ 0");   // แทรกหน้าสุด O(1) เหมือน linked list ที่เขียนเอง

    for (const auto& song : playlist) {
        std::cout << song << '\n';
    }

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 stl_vs_diy.cpp -o stl_vs_diy
./stl_vs_diy
```

```
เพลงที่ 0
เพลงที่ 1
เพลงที่ 2
```

**5 บรรทัด** เทียบกับ **150-300+ บรรทัด** — และ `std::list` ยังมี**ประกัน**ที่โค้ดที่เขียนเอง
มักไม่มี:

| ประเด็น | Linked List เขียนเอง (Part 19) | `std::list` |
|---|---|---|
| Memory Safety | ต้องระวัง memory leak/dangling pointer เอง | รับประกันด้วย RAII (ทบทวน Part 68) |
| ทดสอบมาแล้วแค่ไหน | เท่าที่ผู้เขียนทดสอบเอง | ทดสอบโดยผู้เชี่ยวชาญนับพันคนทั่วโลกนานหลายสิบปี |
| Performance | ขึ้นกับฝีมือผู้เขียน | ปรับแต่ง (optimize) มาอย่างละเอียดสำหรับแต่ละ compiler/platform |
| ฟีเจอร์เสริม | ต้องเขียนเพิ่มเอง (sort, reverse, merge, unique ฯลฯ) | มี method สำเร็จรูปให้ครบ (`sort()`, `reverse()`, `merge()`, `unique()`) |
| ใช้ร่วมกับ Algorithm อื่น | ต้องเขียน adapter เอง | ใช้กับ `<algorithm>` ได้ทันทีผ่าน Iterator |
| เวลาที่ใช้พัฒนา | หลายชั่วโมงถึงหลายวัน | นาทีเดียว (`#include <list>`) |

### กฎทองของ Software Engineering ในโลกจริง

> **"ห้ามเขียน Data Structure พื้นฐานขึ้นมาเองในงาน Production เด็ดขาด นอกจากมีเหตุผลเฉพาะทาง
> ที่จำเป็นจริงๆ เท่านั้น"**

เหตุผลเบื้องหลังกฎนี้ไม่ใช่แค่เรื่อง "ประหยัดเวลา" แต่เป็นเรื่อง **ความน่าเชื่อถือ**: STL ผ่าน
การใช้งานจริงในซอฟต์แวร์นับล้านโปรแกรมทั่วโลกมานานกว่า 25 ปี บั๊กที่เป็นไปได้เกือบทั้งหมดถูกพบ
และแก้ไขไปแล้ว ในขณะที่ data structure ที่เขียนขึ้นใหม่มีความเสี่ยงที่จะมีบั๊กที่ยังไม่ถูกค้นพบ
เสมอ — เวลาที่ประหยัดได้จากการไม่ต้อง debug บั๊กเหล่านี้ในอนาคตมีค่ามากกว่าเวลาที่ใช้เขียนเอง
มากมายหลายเท่า

**แล้วทำไม Module B ถึงให้เขียน Data Structure เองทั้งหมด?** เพราะการเข้าใจว่า Linked List,
Stack, Tree, Hash Table **ทำงานอย่างไรข้างใน** เป็นสิ่งจำเป็นสำหรับ (1) การเลือก Container
ที่เหมาะสมกับสถานการณ์ (ต้องรู้ว่า `std::list` ทำงานแบบ linked list ข้างใน จึงจะเข้าใจว่าทำไม
`push_front` เร็ว แต่การเข้าถึง element กลางลิสต์ช้า) (2) การสัมภาษณ์งานสาย Software Engineer
ที่มักถามเรื่อง data structure พื้นฐาน (จะฝึกฝนเจาะลึกอีกครั้งใน **Part 119**) และ (3) การ
debug เมื่อ STL container ทำงานไม่ตรงกับที่คาดหวัง (ต้องเข้าใจกลไกภายในถึงจะรู้ว่าทำไม) —
กล่าวคือ **เราเรียนเขียนเองเพื่อ "เข้าใจ" ไม่ใช่เพื่อ "ใช้งานจริง"**

---

## 58.3 ภาพรวม Container ทั้งหมดใน STL (Step 459)

STL แบ่ง Container ออกเป็น **3 กลุ่มใหญ่** ตามลักษณะการจัดเก็บและเข้าถึงข้อมูล:

### กลุ่มที่ 1: Sequence Container (ภาชนะแบบลำดับ)

เก็บข้อมูลเรียงตามลำดับที่ใส่เข้าไป (ไม่มีการจัดเรียงอัตโนมัติตามค่า) — เข้าถึงสมาชิกได้ผ่าน
**ตำแหน่ง (position)**

```cpp
#include <array>
#include <deque>
#include <forward_list>
#include <iostream>
#include <list>
#include <vector>

int main() {
    // std::vector: array แบบขยายขนาดได้, จำ contiguous memory, เข้าถึงสุ่ม O(1)
    std::vector<int> v = {1, 2, 3};
    v.push_back(4);
    std::cout << "vector: ";
    for (int x : v) std::cout << x << ' ';
    std::cout << '\n';

    // std::array: array ขนาดคงที่ตอน compile-time (non-type template parameter, ทบทวน Part 56)
    std::array<int, 3> a = {10, 20, 30};
    std::cout << "array: ";
    for (int x : a) std::cout << x << ' ';
    std::cout << '\n';

    // std::deque: เพิ่ม/ลบได้เร็วทั้งหัวและท้าย O(1)
    std::deque<int> dq = {1, 2, 3};
    dq.push_front(0);
    dq.push_back(4);
    std::cout << "deque: ";
    for (int x : dq) std::cout << x << ' ';
    std::cout << '\n';

    // std::list: doubly linked list, แทรก/ลบตรงกลางเร็ว O(1) (เมื่อมี iterator อยู่แล้ว)
    std::list<int> lst = {1, 2, 3};
    lst.push_front(0);
    std::cout << "list: ";
    for (int x : lst) std::cout << x << ' ';
    std::cout << '\n';

    // std::forward_list: singly linked list, ประหยัดหน่วยความจำกว่า list แต่เดินได้ทิศทางเดียว
    std::forward_list<int> flst = {1, 2, 3};
    flst.push_front(0);
    std::cout << "forward_list: ";
    for (int x : flst) std::cout << x << ' ';
    std::cout << '\n';

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 sequence_containers.cpp -o sequence_containers
./sequence_containers
```

```
vector: 1 2 3 4 
array: 10 20 30 
deque: 0 1 2 3 4 
list: 0 1 2 3 
forward_list: 0 1 2 3 
```

รายละเอียดเชิงลึกของแต่ละตัว (ความซับซ้อนเชิงเวลาของแต่ละ operation, การจัดการหน่วยความจำ
ภายใน, เมื่อไหร่ควรใช้ตัวไหน) จะเจาะลึกเต็มรูปแบบใน **Part 59** (`vector`/`array`) และ
**Part 60** (`list`/`deque`/`forward_list`)

### กลุ่มที่ 2: Associative Container (ภาชนะแบบเชื่อมโยง)

เก็บข้อมูลแบบ **key-value** หรือเก็บค่าที่ไม่ซ้ำกัน โดยจัดเรียงหรือจัดกลุ่มข้อมูลอัตโนมัติผ่าน
**key** ไม่ใช่ตำแหน่งที่ใส่เข้าไป

```cpp
#include <iostream>
#include <map>
#include <set>
#include <string>
#include <unordered_map>
#include <unordered_set>

int main() {
    // std::map: key-value ที่เรียงลำดับ key อัตโนมัติ (Red-Black Tree), O(log n)
    std::map<std::string, int> ages;
    ages["Somchai"] = 20;
    ages["Malee"] = 22;
    ages["Anan"] = 19;
    std::cout << "map (เรียงตาม key อัตโนมัติ): ";
    for (const auto& [name, age] : ages) {   // structured bindings (C++17, ทบทวนใน Part 72)
        std::cout << name << "=" << age << ' ';
    }
    std::cout << '\n';

    // std::unordered_map: key-value แบบ hash table, ไม่เรียงลำดับ แต่ O(1) โดยเฉลี่ย
    std::unordered_map<std::string, int> fastAges(ages.begin(), ages.end());
    std::cout << "unordered_map ค้นหา Malee: " << fastAges["Malee"] << '\n';

    // std::set: เก็บค่าที่ไม่ซ้ำกัน เรียงลำดับอัตโนมัติ
    std::set<int> uniqueNumbers = {5, 3, 5, 1, 3, 9};
    std::cout << "set (ไม่ซ้ำ + เรียงลำดับ): ";
    for (int n : uniqueNumbers) std::cout << n << ' ';
    std::cout << '\n';

    // std::unordered_set: เก็บค่าที่ไม่ซ้ำกัน แบบ hash table เร็วกว่าแต่ไม่เรียงลำดับ
    std::unordered_set<int> fastUnique = {5, 3, 5, 1, 3, 9};
    std::cout << "unordered_set ขนาด: " << fastUnique.size() << '\n';

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 associative_containers.cpp -o associative_containers
./associative_containers
```

```
map (เรียงตาม key อัตโนมัติ): Anan=19 Malee=22 Somchai=20 
unordered_map ค้นหา Malee: 22
set (ไม่ซ้ำ + เรียงลำดับ): 1 3 5 9 
unordered_set ขนาด: 4
```

สังเกตว่า `map` แสดงผลเรียงตามตัวอักษร (`Anan`, `Malee`, `Somchai`) โดยอัตโนมัติ **โดยไม่ต้อง
เรียก sort เอง** — เพราะ `std::map` เก็บข้อมูลด้วยโครงสร้าง **Red-Black Tree** (ต่อยอดจาก
Binary Search Tree ที่เรียนใน Part 21) ที่รักษาลำดับของ key ไว้เสมอ ในขณะที่
`std::unordered_map`/`std::unordered_set` ใช้ **Hash Table** (ต่อยอดจาก Part 22) ที่เร็วกว่า
โดยเฉลี่ยแต่ไม่รักษาลำดับใดๆ ไว้เลย รายละเอียดเต็มรูปแบบจะเจาะลึกใน **Part 61**

### กลุ่มที่ 3: Container Adapter (ภาชนะแบบดัดแปลง)

ไม่ใช่ container ที่แท้จริง แต่เป็น "เปลือกห่อหุ้ม (wrapper)" ที่จำกัด interface ของ container
อื่นให้เหลือแค่พฤติกรรมเฉพาะทาง — คล้ายกับ `Stack<T>` และ `Queue<T>` ที่เราเขียนเองใน Part 56
ทุกประการ (ความจริงแล้วนั่นคือการจำลอง Container Adapter ของ STL นั่นเอง)

```cpp
#include <iostream>
#include <queue>
#include <stack>

int main() {
    // std::stack: LIFO container adapter (default ใช้ std::deque เป็นตัวเก็บข้อมูลจริงข้างใน)
    std::stack<int> st;
    st.push(1);
    st.push(2);
    st.push(3);
    std::cout << "stack top: " << st.top() << '\n';
    st.pop();
    std::cout << "หลัง pop, top: " << st.top() << '\n';

    // std::queue: FIFO container adapter
    std::queue<std::string> q;
    q.push("คนแรก");
    q.push("คนที่สอง");
    std::cout << "queue front: " << q.front() << '\n';
    q.pop();
    std::cout << "หลัง pop, front: " << q.front() << '\n';

    // std::priority_queue: คิวที่ดึงค่า "มากที่สุด" ออกมาก่อนเสมอ (default เป็น max-heap)
    std::priority_queue<int> pq;
    pq.push(5);
    pq.push(1);
    pq.push(9);
    pq.push(3);
    std::cout << "priority_queue ดึงตามลำดับ: ";
    while (!pq.empty()) {
        std::cout << pq.top() << ' ';
        pq.pop();
    }
    std::cout << '\n';

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 container_adapters.cpp -o container_adapters
./container_adapters
```

```
stack top: 3
หลัง pop, top: 2
queue front: คนแรก
หลัง pop, front: คนที่สอง
priority_queue ดึงตามลำดับ: 9 5 3 1
```

**ทำไมเรียกว่า "Adapter"**: `std::stack<T>` ไม่ได้ implement โครงสร้างข้อมูลของตัวเองเลย —
โดย default มันห่อหุ้ม `std::deque<T>` ไว้ข้างใน แล้วเปิดให้ใช้ได้แค่ `push`/`pop`/`top` เท่านั้น
(ปิดการเข้าถึงแบบสุ่มและการแทรกตรงกลางที่ `deque` มีอยู่จริงข้างใน) เราสามารถเปลี่ยนตัวเก็บข้อมูล
ข้างในได้ด้วย template parameter ตัวที่สอง เช่น `std::stack<int, std::vector<int>>` เพื่อบังคับ
ให้ใช้ `vector` แทน `deque` เป็นตัวเก็บข้อมูลจริง — นี่คือแนวคิด **Adapter Pattern** ซึ่งเป็นหนึ่ง
ใน Design Pattern ที่จะเรียนอย่างเป็นทางการใน **Part 98**

---

## 58.4 ตารางสรุป: เลือก Container ให้เหมาะกับงาน (Step 460)

ตารางนี้คือ**เครื่องมืออ้างอิงที่สำคัญที่สุด**ของ Part นี้ ควรกลับมาดูซ้ำทุกครั้งที่ต้องเลือก
container ในการเขียนโปรแกรมจริง:

| Container | ประเภท | เข้าถึงแบบสุ่ม | แทรก/ลบหัว-ท้าย | แทรก/ลบตรงกลาง | ค้นหา | เรียงลำดับอัตโนมัติ | ใช้เมื่อไหร่ |
|---|---|---|---|---|---|---|---|
| `std::vector` | Sequence | O(1) | ท้าย O(1)*, หัว O(n) | O(n) | O(n) | ไม่ | ค่า default แรกที่ควรเลือกเสมอ เข้าถึงแบบสุ่มบ่อย |
| `std::array` | Sequence | O(1) | ไม่รองรับ (ขนาดคงที่) | ไม่รองรับ | O(n) | ไม่ | รู้ขนาดแน่นอนตอน compile-time ต้องการ overhead ต่ำสุด |
| `std::deque` | Sequence | O(1) | หัว-ท้าย O(1) | O(n) | O(n) | ไม่ | ต้องเพิ่ม/ลบทั้งหัวและท้ายบ่อย (queue, sliding window) |
| `std::list` | Sequence | O(n) | หัว-ท้าย O(1) | O(1)** | O(n) | ไม่ | แทรก/ลบตรงกลางบ่อยมาก โดยมี iterator อยู่แล้ว |
| `std::forward_list` | Sequence | O(n) | หัว O(1) | O(1)** | O(n) | ไม่ | เหมือน list แต่ต้องการประหยัดหน่วยความจำสูงสุด เดินทิศทางเดียวพอ |
| `std::map` | Associative | - | - | O(log n) | O(log n) | ใช่ (ตาม key) | ต้องการ key-value ที่เรียงลำดับ หรือ range query |
| `std::set` | Associative | - | - | O(log n) | O(log n) | ใช่ | ต้องการเก็บค่าไม่ซ้ำแบบเรียงลำดับ |
| `std::unordered_map` | Associative | - | - | O(1)*** | O(1)*** | ไม่ | ต้องการ key-value เร็วที่สุด ไม่สนลำดับ |
| `std::unordered_set` | Associative | - | - | O(1)*** | O(1)*** | ไม่ | ต้องการเก็บค่าไม่ซ้ำเร็วที่สุด ไม่สนลำดับ |
| `std::stack` | Adapter | - | LIFO เท่านั้น | - | - | - | ต้องการพฤติกรรม LIFO ล้วนๆ (undo, backtracking, DFS) |
| `std::queue` | Adapter | - | FIFO เท่านั้น | - | - | - | ต้องการพฤติกรรม FIFO ล้วนๆ (task queue, BFS) |
| `std::priority_queue` | Adapter | - | ดึงค่าสูงสุด/ต่ำสุดเท่านั้น | - | - | ใช่ (heap) | ต้องการดึงค่ามาก/น้อยสุดซ้ำๆ (Dijkstra, scheduling) |

`*` amortized O(1) — เฉลี่ยแล้วเร็ว แต่บางครั้งต้อง reallocate ทำให้ครั้งนั้นช้ากว่าปกติ
(รายละเอียดเรื่อง reallocation จะเจาะลึกใน Part 59)
`**` O(1) เฉพาะเมื่อมี iterator ชี้ตำแหน่งอยู่แล้ว — การ**หา**ตำแหน่งก่อนยังคงเป็น O(n)
`***` โดยเฉลี่ย (average case) — worst case อาจแย่ลงถึง O(n) ถ้า hash function ออกแบบไม่ดี
(รายละเอียดเรื่อง hash collision จะเจาะลึกใน Part 61 ต่อยอดจาก Part 22)

### กฎการเลือกอย่างง่ายสำหรับมือใหม่

1. **ไม่แน่ใจว่าจะใช้อะไร ให้เริ่มจาก `std::vector` เสมอ** — เป็น container ที่ใช้บ่อยที่สุด
   ในโค้ดจริงกว่า 80% ของกรณีทั้งหมด เพราะ contiguous memory ทำให้ cache-friendly (จะอธิบาย
   เรื่อง cache locality อย่างละเอียดใน Part 87) และมี interface ที่ยืดหยุ่นที่สุด
2. **ถ้าต้องการ key-value และค้นหาบ่อย ให้ใช้ `std::unordered_map` ก่อน** เว้นแต่ต้องการผลลัพธ์
   ที่เรียงลำดับตาม key เสมอ (เช่นจะ iterate แสดงผลตามลำดับ) จึงค่อยเลือก `std::map`
3. **ถ้าต้องแทรก/ลบตรงกลางบ่อยมากๆ** (และมี iterator ชี้ตำแหน่งอยู่แล้วจากขั้นตอนก่อนหน้า)
   ให้พิจารณา `std::list` — แต่ในทางปฏิบัติ สถานการณ์แบบนี้พบได้น้อยกว่าที่คิด และหลายครั้ง
   `std::vector` ยังคงเร็วกว่าในทางปฏิบัติเพราะ cache locality (จะพิสูจน์ด้วยการวัดจริงใน
   Part 60)
4. **ถ้าต้องการ LIFO/FIFO/priority queue ล้วนๆ ไม่ต้องการความสามารถอื่น** ให้ใช้ Container
   Adapter (`stack`/`queue`/`priority_queue`) เพราะ interface ที่จำกัดช่วยป้องกันบั๊กจากการ
   ใช้งานผิดวิธี (เช่นเผลอเข้าถึง element กลาง stack ที่ไม่ควรทำได้ตั้งแต่แรก)

---

## 58.5 Iterator: สะพานเชื่อม Container กับ Algorithm (Step 461)

**Iterator** คือ object ที่เลียนแบบพฤติกรรมของ pointer (`*`, `++`, `==`, `!=`) แต่นามธรรม
มากพอที่จะใช้ได้กับ container ที่มีโครงสร้างภายในต่างกันโดยสิ้นเชิง — นี่คือกลไกที่ทำให้
Algorithm ตัวเดียวกัน (เช่น `std::sort`) เขียนโค้ด**เพียงครั้งเดียว** แต่ใช้ได้กับทั้ง `vector`,
`deque`, `array`

```cpp
#include <iostream>
#include <iterator>
#include <list>
#include <vector>

int main() {
    std::vector<int> v = {1, 2, 3, 4, 5};

    // Random Access Iterator (vector, array, deque): เดินหน้า/ถอยหลัง/กระโดดได้ในเวลา O(1)
    auto it = v.begin();
    it += 3;   // กระโดดไปตำแหน่งที่ 3 ได้ตรงๆ เพราะ vector เก็บข้อมูลแบบ contiguous memory
    std::cout << "v.begin() + 3 = " << *it << '\n';

    std::list<int> lst = {1, 2, 3, 4, 5};

    // Bidirectional Iterator (list, map, set): เดินหน้า/ถอยหลังได้ แต่กระโดดตรงๆ ไม่ได้
    auto lit = lst.begin();
    std::advance(lit, 3);   // ต้องใช้ std::advance ซึ่งจะเดินทีละ step ให้ 3 ครั้ง (O(n))
    std::cout << "std::advance(lst.begin(), 3) = " << *lit << '\n';

    // const_iterator: ใช้แค่ "อ่าน" ห้ามแก้ไขค่าที่ iterator ชี้อยู่
    std::vector<int>::const_iterator cit = v.cbegin();
    std::cout << "const_iterator อ่านค่าแรก: " << *cit << '\n';

    // reverse_iterator: เดินย้อนจากท้ายไปหน้า
    std::cout << "เดินย้อนกลับด้วย rbegin/rend: ";
    for (auto rit = v.rbegin(); rit != v.rend(); ++rit) {
        std::cout << *rit << ' ';
    }
    std::cout << '\n';

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 iterators.cpp -o iterators
./iterators
```

```
v.begin() + 3 = 4
std::advance(lst.begin(), 3) = 4
const_iterator อ่านค่าแรก: 1
เดินย้อนกลับด้วย rbegin/rend: 5 4 3 2 1 
```

### หมวดหมู่ของ Iterator (Iterator Category)

Iterator ไม่ได้มีความสามารถเท่ากันหมด — STL แบ่งออกเป็นหมวดหมู่ตามความสามารถ เรียงจากน้อยไป
มาก:

| หมวดหมู่ | ความสามารถ | ตัวอย่าง Container |
|---|---|---|
| Input/Output Iterator | อ่าน/เขียนได้ครั้งเดียว เดินหน้าทางเดียว | `std::istream_iterator` |
| Forward Iterator | เดินหน้าซ้ำได้หลายรอบ | `std::forward_list` |
| Bidirectional Iterator | เดินหน้าและถอยหลังได้ | `std::list`, `std::map`, `std::set` |
| Random Access Iterator | กระโดดไปตำแหน่งใดก็ได้ในเวลา O(1) เหมือน pointer ธรรมดา | `std::vector`, `std::deque`, `std::array` |

Algorithm บางตัวต้องการ Iterator ที่มีความสามารถระดับหนึ่งเป็นอย่างน้อย — เช่น `std::sort`
ต้องการ **Random Access Iterator** เท่านั้น (เพราะ Sorting Algorithm ที่ใช้ภายในต้องกระโดดไป
เปรียบเทียบตำแหน่งต่างๆ อย่างรวดเร็ว) นี่คือเหตุผลที่ **`std::sort(myList.begin(),
myList.end())` จะ compile error ทันที** ถ้า `myList` เป็น `std::list` (เพราะ `std::list` มีแค่
Bidirectional Iterator) — `std::list` จึงมี method `sort()` ของตัวเองแยกต่างหากที่ implement
algorithm การเรียงลำดับที่เหมาะกับ Bidirectional Iterator โดยเฉพาะ รายละเอียดเชิงลึกของ
Iterator Category ทั้งหมด (รวมถึงการเขียน Custom Iterator ของตัวเอง) จะเรียนใน **Part 62**

---

## 58.6 Algorithm และ Functor: ประมวลผลข้อมูลแบบ Generic (Step 462)

`<algorithm>` คือ header ที่รวม function template สำเร็จรูปกว่า **100 ตัว** ที่ทำงานผ่าน
Iterator เท่านั้น ทำให้ใช้ได้กับ container เกือบทุกชนิด (ยกเว้นบางตัวที่ต้องการ Iterator
category ขั้นสูงกว่าที่ container นั้นมี ดังที่อธิบายใน 58.5)

```cpp
#include <algorithm>
#include <functional>
#include <iostream>
#include <numeric>
#include <vector>

// Functor (Function Object): struct/class ที่ overload operator() ให้ "เรียกได้เหมือนฟังก์ชัน"
struct IsGreaterThan {
    int threshold;
    bool operator()(int value) const { return value > threshold; }
};

int main() {
    std::vector<int> numbers = {5, 3, 8, 1, 9, 2, 7};

    // sort พร้อม functor เป็น comparator แบบ custom (เรียงจากมากไปน้อย)
    std::sort(numbers.begin(), numbers.end(), std::greater<int>());
    std::cout << "เรียงจากมากไปน้อย: ";
    for (int n : numbers) std::cout << n << ' ';
    std::cout << '\n';

    // find_if พร้อม functor ของเราเอง
    auto it = std::find_if(numbers.begin(), numbers.end(), IsGreaterThan{6});
    if (it != numbers.end()) {
        std::cout << "ค่าแรกที่มากกว่า 6: " << *it << '\n';
    }

    // find_if พร้อม lambda (ฟังก์ชันไม่มีชื่อ) แทน functor -- นิยมกว่าใน Modern C++
    auto it2 = std::find_if(numbers.begin(), numbers.end(), [](int n) { return n < 3; });
    if (it2 != numbers.end()) {
        std::cout << "ค่าแรกที่น้อยกว่า 3: " << *it2 << '\n';
    }

    // accumulate: รวมค่าทั้งหมดเข้าด้วยกัน
    int sum = std::accumulate(numbers.begin(), numbers.end(), 0);
    std::cout << "ผลรวมทั้งหมด: " << sum << '\n';

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 algorithms_functors.cpp -o algorithms_functors
./algorithms_functors
```

```
เรียงจากมากไปน้อย: 9 8 7 5 3 2 1 
ค่าแรกที่มากกว่า 6: 9
ค่าแรกที่น้อยกว่า 3: 2
ผลรวมทั้งหมด: 35
```

### ทำไม Functor ถึงสำคัญ

`IsGreaterThan` ในตัวอย่างคือ **Functor** — struct ธรรมดาที่ overload `operator()` (ทบทวน
Operator Overloading จาก Part 51) ทำให้ `IsGreaterThan{6}` สามารถถูกเรียกใช้เหมือนฟังก์ชันได้
(`obj(value)`) จุดที่ทรงพลังคือ **Functor สามารถเก็บ state ภายในตัวเองได้** (ในที่นี้คือ
`threshold`) ต่างจากฟังก์ชันธรรมดาที่ไม่มี state ผูกติดมาด้วย — นี่คือเหตุผลที่ STL algorithm
จำนวนมากรับ functor แทนที่จะรับ function pointer ตรงๆ

ในโค้ด Modern C++ (ตั้งแต่ C++11 เป็นต้นมา) **Lambda Expression** (`[](int n) { return n < 3;
}`) กลายเป็นทางเลือกที่นิยมกว่า Functor มากในกรณีทั่วไป เพราะเขียนกระชับกว่า ไม่ต้องประกาศ
struct แยกต่างหาก — แต่ Functor ยังคงมีที่ใช้งานเมื่อต้องการ logic ที่ซับซ้อนกว่าหรือต้องการ
นำกลับมาใช้ซ้ำในหลายที่ (reusability) เรื่อง Lambda และความสัมพันธ์กับ Functor จะเรียนอย่าง
ละเอียดเต็มรูปแบบใน **Part 64** ส่วน Algorithm ทั้งหมดใน `<algorithm>` (มากกว่า 100 ตัว) จะ
เจาะลึกใน **Part 63**

---

## 58.7 ภาพรวมเส้นทางการเรียนรู้ต่อจากนี้ (Step 463)

Part นี้เป็นเพียง**แผนที่ภาพรวม** ของ STL ทั้งหมด — แต่ละหัวข้อที่กล่าวถึงสั้นๆ ในที่นี้จะถูก
เจาะลึกอย่างเต็มรูปแบบใน Part ถัดๆ ไปของ Module E:

```
Part 58 (ตอนนี้)  →  ภาพรวมทั้งหมด: Container/Iterator/Algorithm/Functor
        │
        ▼
Part 59  →  std::vector และ std::array แบบเจาะลึก (reallocation, capacity, การจัดการ memory)
        │
        ▼
Part 60  →  std::list, std::deque, std::forward_list: เปรียบเทียบและเลือกใช้จริง
        │
        ▼
Part 61  →  std::map, std::set, unordered_map/unordered_set: โครงสร้างภายใน Red-Black Tree
        │      และ Hash Table
        ▼
Part 62  →  STL Iterator แบบเจาะลึก: ทุก category, การเขียน Custom Iterator เอง
        │
        ▼
Part 63  →  STL Algorithm แบบเจาะลึก: sort/find/transform/accumulate และอีกกว่า 100 ตัว
        │
        ▼
Part 64  →  Function Object, Lambda Expression, std::function
        │
        ▼
Part 65-68  →  string ขั้นสูง, Custom Allocator, Smart Pointer, RAII Pattern
```

การเรียนตามลำดับนี้สำคัญมาก เพราะแต่ละ Part สร้างพื้นฐานให้ Part ถัดไป — เช่น การเข้าใจ
Iterator Category ใน Part 62 จะช่วยให้เข้าใจว่าทำไม Algorithm บาง ตัวใน Part 63 ใช้กับบาง
container ไม่ได้ และการเข้าใจ Lambda ใน Part 64 จะทำให้เขียนโค้ดที่ใช้ Algorithm ร่วมกับ
Container ได้อย่างกระชับและทรงพลังที่สุด

---

## ข้อผิดพลาดที่พบบ่อย (Common Pitfalls)

1. **ใช้ `std::vector` เป็นค่าเริ่มต้นเสมอโดยไม่คิด แม้ pattern การใช้งานไม่เหมาะสม** — เช่น
   เขียนโปรแกรมที่ต้อง `push_front`/`pop_front` บ่อยมากแต่ยังใช้ `std::vector` (ที่ O(n) สำหรับ
   operation เหล่านี้) ทำให้ performance แย่ลงอย่างมีนัยสำคัญเมื่อข้อมูลมีขนาดใหญ่ ควรพิจารณา
   `std::deque` แทนเมื่อ pattern การใช้งานตรงกับตาราง 58.4
2. **เรียก `std::sort` กับ `std::list` โดยตรง** — จะได้ compile error ทันทีเพราะ `std::sort`
   ต้องการ Random Access Iterator แต่ `std::list` มีแค่ Bidirectional Iterator ต้องใช้
   `myList.sort()` (method ของ `std::list` เอง) แทน
3. **ใช้ `std::map`/`std::set` เมื่อไม่จำเป็นต้องเรียงลำดับ** — ถ้าไม่สนใจลำดับของ key เลย
   `std::unordered_map`/`std::unordered_set` เร็วกว่ามาก (O(1) เฉลี่ย เทียบกับ O(log n))
   การเลือก `map`/`set` ทั้งที่ไม่จำเป็นเป็นการเสีย performance โดยไม่ได้ประโยชน์อะไรเพิ่ม
4. **คิดว่า Container Adapter (`stack`/`queue`) เป็น container จริงที่ implement เอง** —
   ทำให้เข้าใจผิดว่าเปลี่ยน underlying container ไม่ได้ ทั้งที่จริงสามารถระบุ container ข้างใน
   เองได้ผ่าน template parameter ตัวที่สอง เช่น `std::stack<int, std::vector<int>>`
5. **เข้าใจผิดว่า Iterator ทุกตัวมีความสามารถเท่ากัน** — พยายามเขียน `it += 5` กับ iterator ของ
   `std::list` หรือ `std::map` จะ compile error เพราะ Bidirectional Iterator รองรับแค่ `++`/`--`
   ไม่รองรับการกระโดด (`+=`) ต้องใช้ `std::advance(it, 5)` แทน (ซึ่งจะเดินทีละ step ให้เอง แต่
   ยังคงเป็น O(n) ไม่ใช่ O(1))
6. **ลืมว่า `unordered_map`/`unordered_set` ไม่รักษาลำดับใดๆ เลย** — เขียนโค้ดที่คาดหวังว่าการ
   `for` loop ผ่าน `unordered_map` จะได้ลำดับเดิมทุกครั้งที่รัน (เช่น เรียงตามลำดับที่ insert)
   เป็นสมมติฐานที่**ผิด**และอันตราย ลำดับของ `unordered_map` ขึ้นกับ hash function และ
   implementation ภายในของ compiler แต่ละตัว ไม่รับประกันอะไรเลย ถ้าต้องการลำดับที่แน่นอนต้อง
   ใช้ `std::map` แทน

---

## แบบฝึกหัดท้ายบท

1. เขียนโปรแกรมที่ใช้ `std::set<int>` กำจัดค่าซ้ำออกจาก `std::vector<int>` ที่มีค่าซ้ำกันอยู่
   (สร้าง `vector` ที่มีค่าซ้ำ ใส่เข้า `set` แล้วดึงกลับมาแสดงผล ค่าที่ได้ต้องไม่ซ้ำและเรียง
   ลำดับ)
2. เขียนโปรแกรมนับความถี่ของคำในประโยค (word frequency counter) โดยใช้ `std::map<std::string,
   int>` เก็บผลลัพธ์ แล้วแสดงผลเรียงตามตัวอักษร
3. เขียนโปรแกรมตรวจสอบวงเล็บสมดุล (balanced parentheses) — รับ string ที่มี `()`, `[]`, `{}`
   ผสมกัน แล้วใช้ `std::stack<char>` ตรวจสอบว่าวงเล็บทุกคู่ปิดถูกต้องและตรงชนิดกันหรือไม่
4. เขียนโปรแกรมที่ใช้ `std::priority_queue<int>` หาค่า **k ตัวที่มากที่สุด** จาก `vector<int>`
   ที่กำหนด (คำใบ้: ใส่ทุกค่าเข้า priority_queue แล้วดึงออกมา k ครั้ง)
5. จับคู่สถานการณ์ต่อไปนี้กับ Container ที่เหมาะสมที่สุดจากตาราง 58.4 พร้อมให้เหตุผลสั้นๆ:
   (ก) ระบบจัดคิวลูกค้าธนาคารแบบมาก่อนได้ก่อน (ข) เก็บรายชื่อนักเรียนที่ไม่ซ้ำกันเรียงตาม
   ตัวอักษร (ค) เก็บ log เหตุการณ์ที่เพิ่มเข้ามาเรื่อยๆ ท้ายสุด แล้วอ่านย้อนหลังบ่อย
   (ง) ระบบแปลรหัสไปรษณีย์เป็นชื่อจังหวัด ที่ต้องการค้นหาเร็วที่สุดโดยไม่สนใจลำดับ
6. เขียนโปรแกรมเปรียบเทียบเวลาที่ใช้ `push_front` 100,000 ครั้งระหว่าง `std::vector<int>` กับ
   `std::deque<int>` (ใช้ `<chrono>` จับเวลา ทบทวนวิธีจับเวลาได้จาก Part 24) แล้วอธิบายผลลัพธ์
   ที่ได้ด้วยความรู้เรื่อง Time Complexity จากตาราง 58.4

### แนวทางเฉลยข้อ 2: Word Frequency Counter ด้วย `std::map`

```cpp
#include <iostream>
#include <map>
#include <sstream>
#include <string>

int main() {
    std::string text = "the quick brown fox jumps over the lazy dog the fox runs";
    std::map<std::string, int> freq;

    std::istringstream iss(text);
    std::string word;
    while (iss >> word) {
        ++freq[word];   // ถ้ายังไม่มี key นี้ map จะสร้างให้อัตโนมัติด้วยค่าเริ่มต้น 0 แล้วค่อยบวก 1
    }

    std::cout << "ความถี่ของแต่ละคำ (เรียงตามตัวอักษรอัตโนมัติเพราะ map):\n";
    for (const auto& [word_, count] : freq) {
        std::cout << "  " << word_ << ": " << count << '\n';
    }

    return 0;
}
```

ผลลัพธ์:

```
ความถี่ของแต่ละคำ (เรียงตามตัวอักษรอัตโนมัติเพราะ map):
  brown: 1
  dog: 1
  fox: 2
  jumps: 1
  lazy: 1
  over: 1
  quick: 1
  runs: 1
  the: 3
```

จุดสำคัญที่สุดของโค้ดนี้คือบรรทัด `++freq[word];` — เมื่อ `word` ยังไม่เคยมีอยู่ใน `map` เลย
`operator[]` ของ `std::map` จะ**สร้าง key ใหม่ให้อัตโนมัติ** พร้อมค่าเริ่มต้นเป็น `int{}` คือ
`0` (ทบทวนแนวคิด default-initialization ของ type ต่างๆ) จากนั้น `++` จึงเพิ่มค่าเป็น `1` ทันที
— แพทเทิร์น `++map[key];` แบบนี้เป็นสำนวนมาตรฐานที่พบบ่อยที่สุดสำหรับการนับความถี่ด้วย
`std::map`/`std::unordered_map` ในโค้ด C++ ทั่วโลก และการที่ผลลัพธ์เรียงตามตัวอักษรมาให้เองก็
เป็นผลจากการที่ `std::map` เก็บข้อมูลด้วย Red-Black Tree ที่รักษาลำดับ key ไว้เสมอ ดังที่อธิบาย
ใน 58.3 — ไม่ต้องเขียน sort เพิ่มเติมแม้แต่บรรทัดเดียว

### แนวทางเฉลยข้อ 3: ตรวจสอบวงเล็บสมดุลด้วย `std::stack`

```cpp
#include <iostream>
#include <stack>
#include <string>

bool isBalanced(const std::string& expr) {
    std::stack<char> st;
    for (char c : expr) {
        if (c == '(' || c == '[' || c == '{') {
            st.push(c);
        } else if (c == ')' || c == ']' || c == '}') {
            if (st.empty()) return false;   // ปิดวงเล็บโดยไม่มีตัวเปิดรออยู่
            char top = st.top();
            st.pop();
            if ((c == ')' && top != '(') ||
                (c == ']' && top != '[') ||
                (c == '}' && top != '{')) {
                return false;   // ชนิดวงเล็บไม่ตรงกัน เช่น "(]"
            }
        }
    }
    return st.empty();   // ต้องไม่มีวงเล็บเปิดเหลือค้างอยู่
}

int main() {
    std::string tests[] = {"(a + b) * [c - d]", "{[()]}", "(a + b]", "((a)", "a + b"};
    for (const auto& t : tests) {
        std::cout << "\"" << t << "\" -> " << (isBalanced(t) ? "สมดุล" : "ไม่สมดุล") << '\n';
    }
    return 0;
}
```

ผลลัพธ์:

```
"(a + b) * [c - d]" -> สมดุล
"{[()]}" -> สมดุล
"(a + b]" -> ไม่สมดุล
"((a)" -> ไม่สมดุล
"a + b" -> สมดุล
```

จุดสำคัญ: `isBalanced` ใช้ `std::stack<char>` ตามธรรมชาติของปัญหานี้เป๊ะๆ — ทุกครั้งที่เจอ
วงเล็บเปิด เราต้อง "จำ" มันไว้เพื่อรอให้มีวงเล็บปิดที่ตรงชนิดกันมาปิดในภายหลัง และวงเล็บที่ปิด
**ล่าสุด**ต้องตรงกับวงเล็บที่ **เปิดล่าสุด** เสมอ (Last In, First Out) นี่คือเหตุผลว่าทำไม
`stack` ถึงเป็น container ที่เหมาะสมที่สุดสำหรับปัญหานี้โดยธรรมชาติ — สังเกตเงื่อนไข 2 ข้อที่
ต้องตรวจสอบให้ครบ: (1) `st.empty()` ตอนเจอวงเล็บปิด หมายถึงปิดวงเล็บที่ไม่เคยถูกเปิดมาก่อน และ
(2) หลังวนลูปจบ `st.empty()` ต้องเป็นจริง หมายถึงไม่มีวงเล็บเปิดที่ค้างไม่ได้ถูกปิดเลย (ดังที่
เห็นในเคส `"((a)"` ที่ผ่านการตรวจสอบระหว่างทางทุกครั้ง แต่ล้มเหลวตรงเงื่อนไขสุดท้ายเพราะยังมี
`(` ค้างอยู่ใน stack 1 ตัว)

---

## สรุปท้ายบท

Part นี้ให้ภาพรวมของ **STL (Standard Template Library)** ทั้งระบบ ซึ่งเป็นเครื่องมือที่เรา
จะใช้งานอย่างเข้มข้นที่สุดตลอด Module E และตลอดหลักสูตรที่เหลือ:

- STL ประกอบด้วย 4 ส่วนประกอบที่ทำงานร่วมกัน: **Container** (เก็บข้อมูล), **Iterator**
  (เดินผ่านข้อมูล), **Algorithm** (ประมวลผล), และ **Functor/Lambda** (พฤติกรรมที่ปรับแต่งได้)
- ควรใช้ STL แทนการเขียน data structure เองในงานจริงแทบทุกกรณี เพราะผ่านการทดสอบและ optimize
  มาอย่างละเอียดกว่า data structure ที่เขียนขึ้นใหม่ — เปรียบเทียบให้เห็นชัดจาก Linked List
  150-300 บรรทัดใน Part 19 เทียบกับ `std::list` ที่ใช้งานได้ทันทีในไม่กี่บรรทัด
- Container แบ่งเป็น 3 กลุ่ม: **Sequence** (`vector`, `array`, `deque`, `list`,
  `forward_list`), **Associative** (`map`, `set`, `unordered_map`, `unordered_set`), และ
  **Adapter** (`stack`, `queue`, `priority_queue`)
- มีตารางสรุปการเลือก container ตาม Time Complexity ของแต่ละ operation ซึ่งเป็นเครื่องมือ
  อ้างอิงสำคัญสำหรับการตัดสินใจในโค้ดจริง
- Iterator มีหลาย category (Input/Output, Forward, Bidirectional, Random Access) ที่กำหนดว่า
  Algorithm ตัวไหนใช้ได้กับ Container ตัวไหนบ้าง

ตอนนี้เรามี**แผนที่ภาพรวม**ของ STL ครบถ้วนแล้ว ใน **Part 59** เราจะเริ่มเจาะลึก container
ตัวแรกและสำคัญที่สุดคือ **`std::vector` และ `std::array`** อย่างละเอียด — รวมถึงกลไกการ
reallocate หน่วยความจำที่เคยกล่าวถึงสั้นๆ ใน Part 55 (ตอนออกแบบ Library Management System)
และความแตกต่างเรื่อง capacity กับ size ที่เป็นรากฐานสำคัญสำหรับการเขียนโค้ดที่มีประสิทธิภาพ
ด้วย STL

**ต่อไป:** [Part 59 — std::vector และ std::array](./part-059-vector-array.md)
