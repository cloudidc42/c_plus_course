# Part 61: std::map, std::set, unordered_map/set (Step 481–488)

> Module E — Templates, Generic Programming และ STL | Part 61 จาก 125
> Step ที่ครอบคลุมใน Part นี้: Step 481–488
> Part ก่อนหน้า: [Part 60 — std::list, std::deque, std::forward_list](./part-060-list-deque-forwardlist.md) | Part ถัดไป: [Part 62 — STL Iterator แบบเจาะลึก](./part-062-stl-iterators.md)

## เป้าหมายการเรียนรู้ (Learning Objectives)

หลังจบ Part นี้ ผู้เรียนจะสามารถ:

1. ใช้งาน `std::map` เก็บข้อมูลแบบ key-value ที่เรียงลำดับตาม key โดยอัตโนมัติ พร้อมอธิบายได้ว่า
   เบื้องหลังใช้โครงสร้างข้อมูล **Red-Black Tree** ทำให้ทุก operation เป็น O(log n)
2. ใช้งาน `std::set` เก็บข้อมูลที่ไม่ซ้ำและเรียงลำดับอัตโนมัติ
3. อธิบายและหลีกเลี่ยงข้อผิดพลาดคลาสสิกของ `operator[]` ใน `std::map` — การใช้ `[]` เพื่อ
   "เช็ค" ว่ามี key อยู่หรือไม่ ทำให้เกิดการสร้าง element ใหม่โดยไม่ตั้งใจ
4. เลือกใช้ `find()`, `count()`, และ `at()` แทน `operator[]` ได้อย่างถูกต้องตามสถานการณ์
5. ใช้งาน `std::unordered_map`/`std::unordered_set` ที่ใช้ Hash Table เบื้องหลัง และเชื่อมโยง
   ความเข้าใจกับ Hash Table ที่เขียนขึ้นเองด้วยมือใน Part 22
6. เปรียบเทียบและเลือกระหว่าง ordered container (`map`/`set`) กับ unordered container
   (`unordered_map`/`unordered_set`) ได้อย่างถูกต้องตามความต้องการของโปรแกรมจริง
7. ใช้งาน `std::multimap`/`std::multiset` เมื่อต้องการเก็บ key ที่ซ้ำกันได้

---

## 61.1 std::map: Key-Value ที่เรียงลำดับด้วย Red-Black Tree (Step 481)

`std::map<Key, Value>` (จาก header `<map>`) คือ **Associative Container** ที่เก็บข้อมูล
เป็นคู่ key-value โดยที่ **key ต้องไม่ซ้ำกัน** และข้อมูลจะถูก**เรียงลำดับตาม key โดยอัตโนมัติ
เสมอ** (ใช้ `operator<` ของชนิดข้อมูล key เป็นตัวเปรียบเทียบ ค่าเริ่มต้น)

```cpp
#include <iostream>
#include <map>
#include <string>

int main() {
    std::map<std::string, int> age = {
        {"Somchai", 25},
        {"Suda", 30},
        {"Anan", 22}
    };

    age["Malee"] = 28;   // เพิ่มสมาชิกใหม่ด้วย operator[]

    // std::map เก็บข้อมูลเรียงตาม key โดยอัตโนมัติเสมอ (ใช้ operator< ของ key เปรียบเทียบ)
    std::cout << "รายชื่อเรียงตามตัวอักษร (key) โดยอัตโนมัติ:\n";
    for (const auto& [name, years] : age) {
        std::cout << "  " << name << " -> " << years << " ปี\n";
    }

    std::cout << "\nจำนวนสมาชิก: " << age.size() << '\n';
    std::cout << "อายุของ Suda: " << age["Suda"] << '\n';
    std::cout << "มี key \"Anan\" อยู่ไหม: " << std::boolalpha
              << (age.count("Anan") > 0) << '\n';
    std::cout << "มี key \"Wichai\" อยู่ไหม: "
              << (age.count("Wichai") > 0) << '\n';

    return 0;
}
```

ผลลัพธ์:

```
รายชื่อเรียงตามตัวอักษร (key) โดยอัตโนมัติ:
  Anan -> 22 ปี
  Malee -> 28 ปี
  Somchai -> 25 ปี
  Suda -> 30 ปี

จำนวนสมาชิก: 4
อายุของ Suda: 30
มี key "Anan" อยู่ไหม: true
มี key "Wichai" อยู่ไหม: false
```

สังเกตว่าแม้เราจะ insert ข้อมูลไม่เรียงลำดับ (`Somchai`, `Suda`, `Anan`, `Malee`) แต่เมื่อวน
loop ผ่าน `map` จะได้ผลลัพธ์เรียงตามตัวอักษรเสมอ — นี่คือคุณสมบัติที่ต่างจาก
`std::unordered_map` (ที่จะเรียนในหัวข้อ 61.5) อย่างชัดเจน

### Red-Black Tree: โครงสร้างข้อมูลที่ซ่อนอยู่เบื้องหลัง

`std::map` (และ `std::set`) เกือบทุก implementation ของ C++ (GCC/libstdc++, Clang/libc++,
MSVC) ใช้โครงสร้างข้อมูลที่เรียกว่า **Red-Black Tree** เบื้องหลัง ซึ่งเป็น **Self-Balancing
Binary Search Tree** ที่ต่อยอดจาก BST ธรรมดาที่เราเรียนใน Part 21

จำได้ไหมว่าปัญหาของ BST ธรรมดาคือ **ถ้า insert ข้อมูลที่เรียงลำดับอยู่แล้ว (เช่น 1, 2, 3,
4, 5, ...) ต้นไม้จะเอียงกลายเป็นเส้นตรง** (เหมือน linked list) ทำให้ complexity แย่ลงจาก
O(log n) กลายเป็น O(n) ในกรณีเลวร้ายที่สุด — **Red-Black Tree แก้ปัญหานี้** ด้วยการกำหนด
กฎพิเศษ (ทุก node มีสี "แดง" หรือ "ดำ" พร้อมกฎเรื่องความสูงของ black node ในทุกเส้นทาง)
ที่บังคับให้ต้นไม้ **ปรับสมดุลตัวเองอัตโนมัติทุกครั้งที่ insert/erase** รับประกันว่าความสูง
ของต้นไม้จะไม่มีวันเกิน `O(log n)` ไม่ว่าจะ insert ข้อมูลในลำดับใดก็ตาม

ผลลัพธ์คือทุก operation หลักของ `std::map`/`std::set` (insert, find, erase) มี complexity
เป็น **O(log n) ที่รับประกันได้แน่นอนในทุกกรณี (worst-case guarantee)** ไม่ใช่แค่กรณีเฉลี่ย
เหมือน Hash Table — นี่คือจุดเด่นสำคัญที่เราจะเปรียบเทียบกับ `unordered_map` ในหัวข้อ 61.6

> **หมายเหตุ**: การ implement Red-Black Tree เต็มรูปแบบด้วยมือมีรายละเอียดซับซ้อนมาก
> (การหมุนต้นไม้ซ้าย/ขวา, การปรับสี node) ซึ่งเกินขอบเขตของหลักสูตรนี้ที่เน้นการ**ใช้งาน**
> `std::map`/`std::set` ให้เชี่ยวชาญ — สิ่งสำคัญที่ต้องเข้าใจคือ**ทำไม** operation ของมัน
> เป็น O(log n) เสมอ ไม่ใช่การเขียนโครงสร้างนี้เองทุกรายละเอียด

---

## 61.2 std::set: Unique Sorted Elements (Step 482)

`std::set<T>` (จาก header `<set>`) ใช้โครงสร้าง Red-Black Tree แบบเดียวกับ `map` ทุกประการ
เพียงแต่**เก็บแค่ค่าเดียว ไม่มี value คู่กับ key** — เปรียบเสมือน `std::map<T, void>` ในทาง
แนวคิด (แม้ implementation จริงจะไม่ได้เขียนแบบนั้นตรงๆ)

```cpp
#include <iostream>
#include <set>

int main() {
    std::set<int> unique_scores = {90, 70, 85, 70, 90, 60};   // ค่าซ้ำจะถูกทิ้งอัตโนมัติ

    std::cout << "จำนวนสมาชิกจริง (ค่าซ้ำถูกทิ้งไปแล้ว): " << unique_scores.size() << '\n';

    std::cout << "สมาชิกทั้งหมดเรียงจากน้อยไปมากอัตโนมัติ: ";
    for (int s : unique_scores) {
        std::cout << s << ' ';
    }
    std::cout << '\n';

    auto [it, inserted] = unique_scores.insert(75);
    std::cout << "insert(75): inserted = " << std::boolalpha << inserted
              << ", ค่าที่ตำแหน่งนั้น = " << *it << '\n';

    auto [it2, inserted2] = unique_scores.insert(75);   // แทรกซ้ำ
    std::cout << "insert(75) ซ้ำอีกครั้ง: inserted = " << inserted2
              << ", ค่าที่ตำแหน่งนั้น = " << *it2 << '\n';

    if (unique_scores.find(85) != unique_scores.end()) {
        std::cout << "พบค่า 85 ใน set\n";
    }

    unique_scores.erase(60);
    std::cout << "หลัง erase(60): ";
    for (int s : unique_scores) {
        std::cout << s << ' ';
    }
    std::cout << '\n';

    return 0;
}
```

ผลลัพธ์:

```
จำนวนสมาชิกจริง (ค่าซ้ำถูกทิ้งไปแล้ว): 4
สมาชิกทั้งหมดเรียงจากน้อยไปมากอัตโนมัติ: 60 70 85 90 
insert(75): inserted = true, ค่าที่ตำแหน่งนั้น = 75
insert(75) ซ้ำอีกครั้ง: inserted = false, ค่าที่ตำแหน่งนั้น = 75
พบค่า 85 ใน set
หลัง erase(60): 70 75 85 90
```

จุดที่ควรสังเกต: `insert()` ของ `set` (และ `map`) คืนค่าเป็น **`std::pair<iterator, bool>`**
โดย `bool` บอกว่า insert สำเร็จจริงหรือไม่ (ถ้า key/value นั้นมีอยู่แล้ว จะได้ `false` และ
`iterator` จะชี้ไปยัง element ที่มีอยู่เดิม ไม่ใช่ตัวใหม่) — ใน C++17 ขึ้นไปเราใช้
**Structured Bindings** (`auto [it, inserted] = ...`) เพื่อแยกค่าทั้งสองออกมาได้อย่างสะดวก
(ฟีเจอร์นี้เราจะเจาะลึกใน Part 72)

`std::set` ใช้บ่อยในสถานการณ์ที่ต้องการ **เก็บกลุ่มของค่าที่ไม่ซ้ำกันและต้องการลำดับ**
เช่น เก็บรายชื่อ ID ที่เคยประมวลผลไปแล้ว, เก็บ tag ที่ไม่ซ้ำของบทความ, หรือใช้แทน
mathematical set ในการคำนวณ union/intersection/difference (ผ่าน algorithm อย่าง
`std::set_union`, `std::set_intersection` ที่จะเรียนใน Part 63)

---

## 61.3 operator[] ของ map กับข้อควรระวังสำคัญ (Step 483)

`operator[]` ของ `std::map` มีพฤติกรรมที่ **แตกต่างจาก `vector` อย่างสิ้นเชิง** และเป็น
สาเหตุของบั๊กที่พบบ่อยที่สุดอย่างหนึ่งของ C++ มือใหม่ (และบางครั้งแม้แต่มือเก๋า):

> **`map::operator[](key)` จะสร้าง element ใหม่ให้ทันทีด้วยค่า default-constructed
> (เช่น `0` สำหรับ `int`, `""` สำหรับ `std::string`) ถ้า key นั้นยังไม่มีอยู่ในแผนที่**

ลองดูตัวอย่างบั๊กที่เกิดขึ้นจริงบ่อยมาก:

```cpp
#include <iostream>
#include <map>
#include <string>

int main() {
    std::map<std::string, int> inventory = {
        {"apple", 50},
        {"banana", 30}
    };

    std::cout << "ขนาด map ก่อนเช็ค: " << inventory.size() << '\n';

    // *** ตัวอย่างนี้แสดงข้อผิดพลาดที่พบบ่อย: ใช้ [] เพื่อ "เช็ค" ว่ามี key อยู่ไหม ***
    if (inventory["mango"] > 0) {   // BUG: ถ้าไม่มี "mango" อยู่ operator[] จะสร้างมันขึ้นมาทันที (ค่า 0)
        std::cout << "มี mango อยู่ในสต็อก\n";
    } else {
        std::cout << "ไม่มี mango ในสต็อก (หรือมี 0 ชิ้น)\n";
    }

    std::cout << "ขนาด map หลังเช็ค: " << inventory.size()
              << "  <-- เพิ่มขึ้นโดยไม่ตั้งใจ! เพราะ [] สร้าง key ใหม่ให้อัตโนมัติ\n";

    std::cout << "ตอนนี้ inventory มี \"mango\" อยู่จริงแล้ว ด้วยค่า = "
              << inventory["mango"] << '\n';

    return 0;
}
```

ผลลัพธ์:

```
ขนาด map ก่อนเช็ค: 2
ไม่มี mango ในสต็อก (หรือมี 0 ชิ้น)
ขนาด map หลังเช็ค: 3  <-- เพิ่มขึ้นโดยไม่ตั้งใจ! เพราะ [] สร้าง key ใหม่ให้อัตโนมัติ
ตอนนี้ inventory มี "mango" อยู่จริงแล้ว ด้วยค่า = 0
```

**นี่คือบั๊กที่อันตรายเป็นพิเศษ** เพราะโค้ดนี้ **คอมไพล์ผ่านโดยไม่มี warning เลย** และ
ผลลัพธ์ในตอนแรก (`"ไม่มี mango ในสต็อก"`) ก็ดู "ถูกต้อง" — แต่ผลข้างเคียงที่ซ่อนอยู่คือ
`inventory` ตอนนี้มี key `"mango"` เพิ่มเข้ามาจริงๆ แล้ว (ด้วยค่า 0) ซึ่งอาจทำให้โค้ดส่วน
อื่นที่ loop ผ่าน `inventory` ทั้งหมด (เช่นแสดงรายการสินค้าทั้งหมดในร้าน) แสดงผล
`"mango: 0"` ออกมาอย่างผิดพลาด ทั้งที่ร้านไม่เคยมี mango เลยด้วยซ้ำ

---

## 61.4 find() vs operator[]: วิธีเช็คการมีอยู่ของ Key ที่ถูกต้อง (Step 484)

วิธีที่ถูกต้องในการ "เช็คว่ามี key นี้อยู่หรือไม่" **โดยไม่แก้ไข map** คือใช้ `find()`
หรือ `count()`:

```cpp
#include <iostream>
#include <map>
#include <stdexcept>
#include <string>

int main() {
    std::map<std::string, int> inventory = {
        {"apple", 50},
        {"banana", 30}
    };

    // วิธีที่ถูกต้อง: ใช้ find() เพื่อเช็คการมีอยู่ โดยไม่สร้าง key ใหม่
    auto it = inventory.find("mango");
    if (it != inventory.end()) {
        std::cout << "มี mango: " << it->second << " ชิ้น\n";
    } else {
        std::cout << "ไม่มี mango ในสต็อก\n";
    }
    std::cout << "ขนาด map หลังเช็คด้วย find(): " << inventory.size()
              << "  <-- ไม่เปลี่ยนแปลง\n";

    // at() ปลอดภัยกว่า [] เมื่อคาดหวังว่า key ต้องมีอยู่แล้ว (ไม่ต้องการสร้างใหม่โดยไม่ตั้งใจ)
    try {
        std::cout << inventory.at("mango") << '\n';
    } catch (const std::out_of_range& e) {
        std::cout << "at(\"mango\") throw: " << e.what() << '\n';
    }
    std::cout << "ขนาด map หลังใช้ at(): " << inventory.size() << "  <-- ยังไม่เปลี่ยนแปลง\n";

    // [] เหมาะกับกรณีที่ต้องการ "insert ถ้ายังไม่มี, อัปเดตถ้ามีอยู่แล้ว" โดยตั้งใจ
    inventory["apple"] += 10;   // อัปเดตค่าเดิม
    inventory["kiwi"] = 5;      // สร้างค่าใหม่โดยตั้งใจ
    std::cout << "\napple = " << inventory["apple"] << ", kiwi = " << inventory["kiwi"] << '\n';

    return 0;
}
```

ผลลัพธ์:

```
ไม่มี mango ในสต็อก
ขนาด map หลังเช็คด้วย find(): 2  <-- ไม่เปลี่ยนแปลง
at("mango") throw: map::at
ขนาด map หลังใช้ at(): 2  <-- ยังไม่เปลี่ยนแปลง

apple = 60, kiwi = 5
```

### สรุปเมื่อไหร่ใช้อะไร

| Method | สร้าง key ใหม่ถ้าไม่มีไหม | throw exception ถ้าไม่มีไหม | ใช้เมื่อไหร่ |
|---|---|---|---|
| `m[key]` | **สร้างให้** (ค่า default) | ไม่ throw | ต้องการ "insert ถ้ายังไม่มี อัปเดตถ้ามี" โดยตั้งใจ เช่น นับความถี่ (`++count[word]`) |
| `m.at(key)` | ไม่สร้าง | **throw `std::out_of_range`** | คาดหวังว่า key ต้องมีอยู่แล้วแน่นอน ถ้าไม่มีถือเป็นข้อผิดพลาดที่ควรหยุดทันที |
| `m.find(key)` | ไม่สร้าง | ไม่ throw (คืน `end()` ถ้าไม่พบ) | เช็คการมีอยู่ **และ** ต้องการ iterator ไปใช้ต่อ (เช่นจะแก้ไข value นั้น) |
| `m.count(key)` | ไม่สร้าง | ไม่ throw (คืน 0 หรือ 1 สำหรับ `map`) | เช็คการมีอยู่แบบง่ายๆ โดยไม่ต้องใช้ iterator ต่อ |

> **หมายเหตุ**: ตั้งแต่ C++20 มี method `m.contains(key)` ที่คืนค่า `bool` ตรงๆ อ่านง่ายกว่า
> `find(key) != end()` หรือ `count(key) > 0` — แต่เนื่องจากหลักสูตรนี้ยึด C++17 เป็นหลัก
> (จะเจาะลึก C++20 ใน Part 73–76) ตัวอย่างในบทเรียนนี้จึงใช้ `find()`/`count()` เป็นหลัก

> **กฎทองของหัวข้อนี้**: ถ้าแค่ต้องการ "เช็คว่ามีไหม" โดยไม่ตั้งใจจะแก้ไข map เลย
> **ห้ามใช้ `operator[]` เด็ดขาด** ให้ใช้ `find()` หรือ `count()` เสมอ

---

## 61.5 std::unordered_map/unordered_set: Hash Table เบื้องหลัง (Step 485)

`std::unordered_map<Key, Value>` และ `std::unordered_set<T>` (จาก header
`<unordered_map>` และ `<unordered_set>`) ใช้ **Hash Table** เป็นโครงสร้างข้อมูลเบื้องหลัง
แทน Red-Black Tree — นี่คือแนวคิดเดียวกับที่เรา**เขียน Hash Table ด้วยมือทั้งหมดใน Part 22**
(hash function, separate chaining, load factor, resize/rehash) เพียงแต่ตอนนี้ STL
implement ให้เราใช้งานสำเร็จรูป

```cpp
#include <iostream>
#include <string>
#include <unordered_map>
#include <unordered_set>

int main() {
    std::unordered_map<std::string, int> word_count;
    std::string words[] = {"the", "quick", "fox", "the", "fox", "the"};

    for (const auto& w : words) {
        ++word_count[w];   // ถ้ายังไม่มี key นี้ operator[] จะสร้างด้วยค่า 0 ก่อนแล้วค่อย ++
    }

    // unordered_map ไม่รับประกันลำดับใดๆ เลย -- ลำดับที่ print ออกมาขึ้นกับ hash function ภายใน
    std::cout << "จำนวนคำ (ลำดับไม่แน่นอน เพราะเป็น hash table):\n";
    for (const auto& [word, count] : word_count) {
        std::cout << "  " << word << " -> " << count << '\n';
    }

    std::unordered_set<int> unique_ids = {5, 3, 8, 3, 1, 5};
    std::cout << "\nunique_ids มีสมาชิก " << unique_ids.size() << " ตัว (ลำดับไม่แน่นอน)\n";

    std::cout << "find(8) เจอไหม: " << std::boolalpha
              << (unique_ids.find(8) != unique_ids.end()) << '\n';

    // ข้อมูล internal ของ hash table ที่ตรวจสอบได้: bucket_count, load_factor
    std::cout << "\nword_count.bucket_count() = " << word_count.bucket_count() << '\n';
    std::cout << "word_count.load_factor() = " << word_count.load_factor() << '\n';

    return 0;
}
```

ผลลัพธ์ (ลำดับการพิมพ์อาจต่างกันไปในแต่ละเครื่อง/แต่ละไลบรารี เพราะไม่มีการรับประกันลำดับ):

```
จำนวนคำ (ลำดับไม่แน่นอน เพราะเป็น hash table):
  fox -> 2
  quick -> 1
  the -> 3

unique_ids มีสมาชิก 4 ตัว (ลำดับไม่แน่นอน)
find(8) เจอไหม: true

word_count.bucket_count() = 13
word_count.load_factor() = 0.230769
```

### เชื่อมโยงกับ Part 22: มันคือ Hash Table ตัวเดียวกัน

ทุกแนวคิดที่เราเรียนใน Part 22 ยังคงอยู่ครบใน `std::unordered_map`:

| แนวคิดจาก Part 22 (Hash Table ที่เขียนเอง) | ใน std::unordered_map |
|---|---|
| Hash Function (`hash(key) % capacity`) | มี default hash function ให้ในตัว (`std::hash<Key>`) — override ได้ถ้าต้องการ custom hash |
| Separate Chaining (แก้ collision ด้วย linked list ใน bucket) | libstdc++/libc++ ใช้ separate chaining เป็นค่าเริ่มต้น |
| Load Factor (`size / bucket_count`) | เข้าถึงได้ตรงๆ ผ่าน `.load_factor()` |
| Resize/Rehash เมื่อ load factor สูงเกินไป | เกิดอัตโนมัติเมื่อเกิน `.max_load_factor()` (ค่าเริ่มต้น 1.0) — ปรับล่วงหน้าได้ด้วย `.reserve(n)` เหมือน `vector` |
| Complexity เฉลี่ย O(1), กรณีเลวร้าย O(n) | เหมือนกันทุกประการ — ถ้า hash function แย่มาก (ทุก key ชนกันหมด) จะเสื่อมเหลือ O(n) |

นี่คือเหตุผลสำคัญที่หลักสูตรนี้ให้เขียน Hash Table ด้วยมือเองก่อนใน Part 22: เพื่อให้เข้าใจ
ว่า `unordered_map` **ไม่ใช่เวทมนตร์** แต่เป็นการ implement แนวคิดเดียวกันกับที่เราเข้าใจ
อยู่แล้วอย่างละเอียด เพียงแค่ทำให้ generic ใช้ได้กับทุกชนิดข้อมูล และผ่านการปรับแต่ง
ประสิทธิภาพมาอย่างดีจากทีมพัฒนา standard library

---

## 61.6 เปรียบเทียบ Ordered vs Unordered: O(log n) กับ O(1) เฉลี่ย (Step 486)

| หัวข้อ | `std::map` / `std::set` (Ordered) | `std::unordered_map` / `std::unordered_set` |
|---|---|---|
| โครงสร้างเบื้องหลัง | Red-Black Tree | Hash Table |
| Complexity ของ find/insert/erase | **O(log n)** รับประกันทุกกรณี (worst-case) | **O(1) โดยเฉลี่ย**, O(n) กรณีเลวร้ายที่สุด (hash collision มาก) |
| ลำดับของสมาชิก | เรียงตาม key เสมอ (`operator<`) | **ไม่มีลำดับที่แน่นอน** ขึ้นกับ hash function |
| ต้องการอะไรจาก key | `operator<` (comparison) | `std::hash<Key>` และ `operator==` |
| ความเร็วในทางปฏิบัติ (ข้อมูลจำนวนมาก) | ช้ากว่า เพราะต้องเดินต้นไม้ log n ครั้ง | **เร็วกว่ามาก** โดยทั่วไป เพราะ O(1) เฉลี่ย |
| Iterator เดินตามลำดับ (in-order traversal) | ได้ทันที (`begin()` ถึง `end()` เรียงตาม key) | ไม่มีลำดับที่มีความหมาย |
| Memory overhead | ต่ำกว่า (แค่ pointer ซ้าย-ขวา-แม่ต่อ node) | สูงกว่าเล็กน้อย (โครงสร้าง bucket array + linked list) |

ลองวัดผลจริงเปรียบเทียบความเร็วการค้นหาข้อมูลจำนวนมาก:

```cpp
#include <chrono>
#include <iostream>
#include <map>
#include <unordered_map>

int main() {
    const int N = 500000;

    std::map<int, int> ordered_map;
    std::unordered_map<int, int> hash_map;

    for (int i = 0; i < N; ++i) {
        ordered_map[i] = i * 2;
        hash_map[i] = i * 2;
    }

    long long sum1 = 0;
    auto start1 = std::chrono::steady_clock::now();
    for (int i = 0; i < N; ++i) {
        sum1 += ordered_map.find(i)->second;
    }
    auto end1 = std::chrono::steady_clock::now();

    long long sum2 = 0;
    auto start2 = std::chrono::steady_clock::now();
    for (int i = 0; i < N; ++i) {
        sum2 += hash_map.find(i)->second;
    }
    auto end2 = std::chrono::steady_clock::now();

    auto us1 = std::chrono::duration_cast<std::chrono::milliseconds>(end1 - start1).count();
    auto us2 = std::chrono::duration_cast<std::chrono::milliseconds>(end2 - start2).count();

    std::cout << "sum1 = " << sum1 << ", sum2 = " << sum2 << '\n';
    std::cout << "std::map (red-black tree, O(log n)): ค้นหา " << N << " ครั้ง ใช้เวลา "
              << us1 << " ms\n";
    std::cout << "std::unordered_map (hash table, O(1) เฉลี่ย): ค้นหา " << N << " ครั้ง ใช้เวลา "
              << us2 << " ms\n";

    return 0;
}
```

ผลลัพธ์ (คอมไพล์ด้วย `-O2`):

```
sum1 = 249999500000, sum2 = 249999500000
std::map (red-black tree, O(log n)): ค้นหา 500000 ครั้ง ใช้เวลา 62 ms
std::unordered_map (hash table, O(1) เฉลี่ย): ค้นหา 500000 ครั้ง ใช้เวลา 5 ms
```

`unordered_map` เร็วกว่าประมาณ **12 เท่า** ในการค้นหาข้อมูลจำนวนมาก — นี่คือเหตุผลว่าทำไม
**ถ้าไม่ต้องการลำดับของข้อมูล ควรเลือก `unordered_map`/`unordered_set` เป็นค่าเริ่มต้นก่อน
เสมอ** เพราะเร็วกว่าในเกือบทุกกรณีการใช้งานจริง

---

## 61.7 เมื่อไหร่ต้องการ Ordering จริงๆ (Step 487)

แม้ `unordered_map`/`unordered_set` จะเร็วกว่าโดยทั่วไป แต่มีหลายสถานการณ์ที่**จำเป็น**
ต้องใช้ `map`/`set` เพราะต้องการคุณสมบัติที่ hash table ให้ไม่ได้:

1. **ต้องการข้อมูลเรียงลำดับเสมอ** เช่น แสดงรายชื่อพนักงานเรียงตามตัวอักษร, แสดง
   leaderboard เรียงคะแนน, หรือสร้างรายงานที่ต้องเรียงตามวันที่ — ถ้าใช้ `unordered_map`
   แล้วต้อง sort เองทุกครั้งที่จะแสดงผล (`O(n log n)` ทุกครั้ง) มักจะช้ากว่าการใช้ `map`
   ตั้งแต่แรกซึ่งรักษาลำดับให้อัตโนมัติตลอดเวลา

2. **ต้องการค้นหาแบบ range** เช่น "หาสมาชิกทุกตัวที่ค่าอยู่ระหว่าง 50 ถึง 100" —
   `map`/`set` มี method `lower_bound()`/`upper_bound()`/`equal_range()` ที่ทำงานได้อย่าง
   มีประสิทธิภาพ O(log n) เพราะโครงสร้างต้นไม้เอื้อต่อการค้นหาแบบช่วง ในขณะที่
   `unordered_map` **ทำแบบนี้ไม่ได้เลย** (ต้องวนดูทุกตัว O(n))

3. **ต้องการหา min/max อย่างรวดเร็ว** — `map`/`set` หาได้ทันทีด้วย `begin()` (ค่าน้อยสุด)
   และ `rbegin()` (ค่ามากสุด) เพราะเรียงลำดับอยู่แล้ว เป็น O(1) หรือ O(log n) ขึ้นกับ
   implementation แต่ `unordered_map` ต้องวนดูทุกตัว O(n) เสมอ

4. **ต้องการ deterministic behavior ที่ทำซ้ำได้เป๊ะทุกครั้ง (reproducibility)** — บางระบบ
   (เช่น การทดสอบอัตโนมัติ, simulation ที่ต้องผลลัพธ์เหมือนเดิมทุกครั้ง) ต้องการลำดับที่
   คงที่แน่นอน ไม่ขึ้นกับ hash function หรือ hash seed แบบสุ่มที่บางไลบรารีใช้เพื่อความ
   ปลอดภัย (ป้องกัน Hash DoS Attack)

5. **ชนิดข้อมูลของ key ไม่มี hash function ที่ดี หรือ implement `std::hash` ยาก** แต่มี
   `operator<` ที่ implement ได้ง่ายกว่า (เช่น struct ที่ซับซ้อนมาก) — ในกรณีนี้ `map` อาจ
   เขียนง่ายกว่าในทางปฏิบัติ

> **หลักคิดสรุป**: เริ่มต้นด้วยคำถาม **"ฉันต้องการลำดับของข้อมูล หรือต้องการค้นหาแบบช่วง
> หรือไม่"** ถ้าตอบว่า "ไม่" ให้ใช้ `unordered_map`/`unordered_set` เพราะเร็วกว่า ถ้าตอบว่า
> "ใช่" ให้ใช้ `map`/`set` แม้จะช้ากว่าเล็กน้อยก็ตาม เพราะมันให้ความสามารถที่ hash table
> ให้ไม่ได้เลย

---

## 61.8 multimap และ multiset โดยย่อ (Step 488)

`std::multimap` และ `std::multiset` ทำงานเหมือน `map`/`set` ทุกประการ **ยกเว้นอนุญาตให้
มี key ซ้ำกันได้** — ยังคงใช้ Red-Black Tree เบื้องหลังเหมือนเดิม และยังคงเรียงลำดับตาม key
เสมอ

```cpp
#include <iostream>
#include <map>
#include <set>
#include <string>

int main() {
    std::multimap<std::string, std::string> student_courses;
    student_courses.insert({"Somchai", "Math"});
    student_courses.insert({"Somchai", "Physics"});
    student_courses.insert({"Somchai", "Chemistry"});
    student_courses.insert({"Suda", "Math"});

    std::cout << "วิชาที่ Somchai ลงทะเบียน:\n";
    auto range = student_courses.equal_range("Somchai");
    for (auto it = range.first; it != range.second; ++it) {
        std::cout << "  " << it->second << '\n';
    }

    std::cout << "จำนวนวิชาทั้งหมดของ Somchai: "
              << student_courses.count("Somchai") << '\n';

    std::multiset<int> scores = {80, 90, 80, 70, 90, 90};
    std::cout << "\nจำนวนคนที่ได้คะแนน 90: " << scores.count(90) << '\n';
    std::cout << "สมาชิกทั้งหมดของ multiset (ค่าซ้ำเก็บได้): ";
    for (int s : scores) {
        std::cout << s << ' ';
    }
    std::cout << '\n';

    return 0;
}
```

ผลลัพธ์:

```
วิชาที่ Somchai ลงทะเบียน:
  Math
  Physics
  Chemistry
จำนวนวิชาทั้งหมดของ Somchai: 3

จำนวนคนที่ได้คะแนน 90: 3
สมาชิกทั้งหมดของ multiset (ค่าซ้ำเก็บได้): 70 80 80 90 90 90
```

จุดที่ต้องรู้เกี่ยวกับ `multimap`/`multiset`:

- **ไม่มี `operator[]`** — เพราะ `[]` ต้องคืนค่าเดียวต่อ key แต่ `multimap` อาจมีหลายค่า
  ต่อ key เดียวกัน จึงไม่มีความหมายที่ชัดเจนพอที่จะ implement `[]` ได้ ต้องใช้ `insert()`,
  `find()`, `equal_range()` แทนเสมอ
- **`equal_range(key)`** คืนค่าเป็น `std::pair<iterator, iterator>` ที่แทนช่วง `[first,
  second)` ของสมาชิกทั้งหมดที่มี key นั้น — เป็นวิธีมาตรฐานในการดึงค่าทั้งหมดที่ตรงกับ
  key เดียวกันออกมา
- **`count(key)`** บอกจำนวนสมาชิกทั้งหมดที่มี key นั้น (ต่างจาก `map`/`set` ที่ `count()`
  คืนได้แค่ 0 หรือ 1 เท่านั้น เพราะ key ไม่ซ้ำ)
- มี `std::unordered_multimap`/`std::unordered_multiset` ด้วยเช่นกัน (hash table ที่ยอมให้
  key ซ้ำ) แต่ใช้งานน้อยกว่า `multimap`/`multiset` มากในทางปฏิบัติ เพราะเมื่อยอมรับ key
  ซ้ำอยู่แล้ว มักมีเหตุผลที่ต้องการลำดับไปด้วยพร้อมกัน

`multimap` มักถูกใช้ในสถานการณ์ **"จัดกลุ่มข้อมูลตาม key"** เช่น จัดกลุ่มนักเรียนตามเกรด,
จัดกลุ่ม transaction ตามวันที่, หรือสร้าง adjacency list ของ graph ที่ node หนึ่งเชื่อมกับ
หลาย node (key คือ node ต้นทาง, value คือ node ปลายทางแต่ละอัน)

---

## ข้อผิดพลาดที่พบบ่อย (Common Pitfalls)

1. **ใช้ `operator[]` เพื่อ "เช็ค" ว่ามี key อยู่ไหม** — ดังที่แสดงในหัวข้อ 61.3 นี่คือบั๊ก
   ที่พบบ่อยที่สุดของ `map` เพราะโค้ดดูเหมือนถูกต้องและคอมไพล์ผ่านไม่มี warning แต่จะทำให้
   `map` มี key ใหม่เพิ่มเข้ามาโดยไม่ตั้งใจทุกครั้ง ต้องใช้ `find()` หรือ `count()` แทนเสมอ
   เมื่อแค่ต้องการเช็คการมีอยู่

2. **คาดหวังว่า `std::unordered_map` จะรักษาลำดับการ insert หรือเรียงตาม key** —
   `unordered_map` **ไม่รับประกันลำดับใดๆ เลย** ลำดับที่ได้จากการวน loop อาจเปลี่ยนไปได้
   แม้จะ insert ข้อมูลชุดเดิมก็ตาม (โดยเฉพาะถ้ามีการ rehash เกิดขึ้นระหว่างทาง) ถ้าต้องการ
   ลำดับที่แน่นอน ต้องใช้ `map`/`set` แทน

3. **ใช้ `at()` โดยไม่ครอบ try-catch เมื่อไม่แน่ใจว่า key มีอยู่จริง** — `at()` จะ throw
   `std::out_of_range` ทันทีถ้าไม่มี key นั้น ถ้าโปรแกรมไม่ได้ handle exception นี้ไว้
   โปรแกรมจะ crash (terminate) ทันที ควรใช้ `find()`/`count()` ก่อนเช็คถ้าไม่แน่ใจ หรือ
   ครอบ `at()` ด้วย try-catch ถ้าต้องการให้ error นั้นเป็นเหตุการณ์พิเศษที่จัดการได้

4. **ลืมว่า key ของ `map`/`set` ต้อง immutable ในทางความหมาย** — การแก้ไขค่า key
   โดยตรง (เช่นผ่าน `const_cast` เพื่อแอบแก้ไข key ใน `std::set` ซึ่งเป็น `const` โดย
   ธรรมชาติ) เป็น Undefined Behavior เพราะจะทำให้โครงสร้าง Red-Black Tree เสียหาย
   (ต้นไม้ถูกจัดเรียงตาม key เดิม ถ้า key เปลี่ยนแบบไม่ผ่านกลไกของ container ต้นไม้จะ
   ไม่ตรงกับข้อมูลจริงอีกต่อไป) ถ้าต้องการเปลี่ยน key ต้อง erase ตัวเก่าแล้ว insert ตัวใหม่

5. **เลือก `unordered_map` โดยไม่รู้ว่าต้องการ ordering** แล้วมาพบทีหลังว่าต้อง sort เอง
   ทุกครั้งที่แสดงผล (ทำให้เสียเวลา O(n log n) ซ้ำๆ) — ควรคิดล่วงหน้าตั้งแต่ตอนออกแบบว่า
   โปรแกรมต้องการลำดับของข้อมูลหรือไม่ ถ้าต้องการบ่อยๆ ให้ใช้ `map`/`set` ตั้งแต่แรก

6. **ใช้ custom struct เป็น key ของ `unordered_map` โดยไม่ implement `std::hash`
   specialization และ `operator==`** — จะเจอ compile error ทันที เพราะ `unordered_map`
   ต้องการทั้งสองอย่างเพื่อคำนวณ hash และเทียบ key ที่ hash ชนกัน (ในขณะที่ `map` ต้องการ
   แค่ `operator<` เพียงอย่างเดียว) นี่เป็นเหตุผลหนึ่งที่บางครั้ง `map` เขียนง่ายกว่าเมื่อ
   key เป็น custom type ที่ซับซ้อน

---

## แบบฝึกหัดท้ายบท

1. เขียนโปรแกรมนับความถี่ของคำในประโยคหนึ่ง โดยใช้ `std::map<std::string, int>` แล้วพิมพ์
   ผลลัพธ์ (ผลลัพธ์จะเรียงตามตัวอักษรโดยอัตโนมัติ)

2. เขียนฟังก์ชัน `bool has_key_safely(const std::map<std::string, int>& m, const
   std::string& key)` ที่เช็คว่ามี key อยู่หรือไม่ **โดยไม่แก้ไข** `m` เลย (พิสูจน์ด้วยการ
   เช็ค `m.size()` ก่อนและหลังเรียกฟังก์ชันว่าต้องเท่ากันเสมอ)

3. เขียนโปรแกรมเปรียบเทียบเวลาการค้นหาข้อมูล 500,000 รายการ ระหว่าง `std::map` กับ
   `std::unordered_map` (คล้ายตัวอย่างในหัวข้อ 61.6) แล้วอธิบายผลลัพธ์ที่ได้ด้วยคำพูดของ
   ตัวเอง

4. เขียนโปรแกรมรับจำนวนเต็มจาก terminal เก็บใน `std::set<int, std::greater<int>>`
   (เรียงจากมากไปน้อย) แล้วพิมพ์ผลลัพธ์ที่ไม่ซ้ำกันเรียงจากมากไปน้อย

5. เขียนโปรแกรมใช้ `std::multimap<char, std::string>` จัดกลุ่มรายชื่อนักเรียนตามเกรด
   (`'A'`, `'B'`, `'C'`) แล้วพิมพ์รายชื่อแยกตามกลุ่มเกรด โดยใช้ประโยชน์จากการที่
   `multimap` เรียงลำดับตาม key ให้อัตโนมัติ

6. อธิบายด้วยคำพูดของตัวเอง (หรือเขียนโค้ดทดสอบประกอบ) ว่าทำไมการใช้ `struct Point {int
   x, y;}` เป็น key ของ `std::unordered_map<Point, std::string>` โดยตรง **ไม่สามารถ
   compile ผ่านได้** ในขณะที่ใช้เป็น key ของ `std::map<Point, std::string>` ได้ ถ้ามีการ
   implement `operator<` ให้กับ `Point`

### แนวทางเฉลยข้อ 1

```cpp
#include <iostream>
#include <map>
#include <sstream>
#include <string>

int main() {
    std::string sentence = "the quick brown fox jumps over the lazy dog the fox runs";
    std::map<std::string, int> word_count;

    std::istringstream iss(sentence);
    std::string word;
    while (iss >> word) {
        ++word_count[word];
    }

    std::cout << "ความถี่ของแต่ละคำ (เรียงตามตัวอักษรอัตโนมัติ เพราะเป็น std::map):\n";
    for (const auto& [w, count] : word_count) {
        std::cout << "  " << w << ": " << count << '\n';
    }

    return 0;
}
```

ผลลัพธ์:

```
ความถี่ของแต่ละคำ (เรียงตามตัวอักษรอัตโนมัติ เพราะเป็น std::map):
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

จุดสำคัญของเฉลยนี้: การใช้ `++word_count[word]` เป็นการใช้ `operator[]` **อย่างถูกวิธี**
(ต่างจากตัวอย่างบั๊กในหัวข้อ 61.3) เพราะที่นี่เรา**ตั้งใจ**ให้สร้าง key ใหม่ด้วยค่าเริ่มต้น
0 ถ้ายังไม่เคยเจอคำนั้นมาก่อน แล้วค่อยบวกเพิ่มทันที — นี่คือ pattern มาตรฐานสำหรับการนับ
ความถี่ (frequency counting) ที่ใช้ `operator[]` ได้อย่างเหมาะสม เพราะเราต้องการผลลัพธ์
แบบ "insert ถ้ายังไม่มี อัปเดตถ้ามีอยู่แล้ว" พอดี

### แนวทางเฉลยข้อ 4

```cpp
#include <iostream>
#include <set>

int main() {
    std::set<int, std::greater<int>> descending_unique;   // เรียงจากมากไปน้อยด้วย comparator

    int input = 0;
    std::cout << "ป้อนจำนวนเต็ม (พิมพ์ -1 เพื่อจบ): ";
    while (std::cin >> input && input != -1) {
        descending_unique.insert(input);
    }

    std::cout << "เลขไม่ซ้ำ เรียงจากมากไปน้อย: ";
    for (int n : descending_unique) {
        std::cout << n << ' ';
    }
    std::cout << '\n';

    return 0;
}
```

ทดสอบด้วย input `5 3 9 3 1 9 -1`:

```
เลขไม่ซ้ำ เรียงจากมากไปน้อย: 9 5 3 1
```

จุดสำคัญ: `std::set<int>` ปกติใช้ `std::less<int>` (เรียงจากน้อยไปมาก) เป็น comparator
เริ่มต้น การส่ง `std::greater<int>` เป็น template parameter ตัวที่สองเข้าไปทำให้ `set`
เปลี่ยนวิธีเปรียบเทียบเป็น "มากกว่า" แทน ซึ่งมีผลให้ Red-Black Tree เบื้องหลังจัดเรียง
โครงสร้างในทิศทางตรงกันข้าม — นี่เป็นตัวอย่างที่ดีของการที่ `map`/`set` ยืดหยุ่นกว่า
`unordered_map`/`unordered_set` ในการปรับแต่งลำดับการแสดงผล เพราะ `unordered_map` ไม่มี
concept ของ "การเรียงลำดับ" ให้ปรับแต่งเลยตั้งแต่ต้น

### แนวทางเฉลยข้อ 3

```cpp
#include <chrono>
#include <iostream>
#include <map>
#include <unordered_map>

int main() {
    const int N = 500000;

    std::map<int, int> ordered_map;
    std::unordered_map<int, int> hash_map;

    for (int i = 0; i < N; ++i) {
        ordered_map[i] = i * 2;
        hash_map[i] = i * 2;
    }

    long long sum1 = 0;
    auto start1 = std::chrono::steady_clock::now();
    for (int i = 0; i < N; ++i) {
        sum1 += ordered_map.find(i)->second;
    }
    auto end1 = std::chrono::steady_clock::now();

    long long sum2 = 0;
    auto start2 = std::chrono::steady_clock::now();
    for (int i = 0; i < N; ++i) {
        sum2 += hash_map.find(i)->second;
    }
    auto end2 = std::chrono::steady_clock::now();

    auto us1 = std::chrono::duration_cast<std::chrono::milliseconds>(end1 - start1).count();
    auto us2 = std::chrono::duration_cast<std::chrono::milliseconds>(end2 - start2).count();

    std::cout << "sum1 = " << sum1 << ", sum2 = " << sum2 << '\n';
    std::cout << "std::map (red-black tree, O(log n)): ค้นหา " << N << " ครั้ง ใช้เวลา "
              << us1 << " ms\n";
    std::cout << "std::unordered_map (hash table, O(1) เฉลี่ย): ค้นหา " << N << " ครั้ง ใช้เวลา "
              << us2 << " ms\n";

    return 0;
}
```

ผลลัพธ์ (คอมไพล์ด้วย `-O2`):

```
sum1 = 249999500000, sum2 = 249999500000
std::map (red-black tree, O(log n)): ค้นหา 500000 ครั้ง ใช้เวลา 62 ms
std::unordered_map (hash table, O(1) เฉลี่ย): ค้นหา 500000 ครั้ง ใช้เวลา 5 ms
```

**คำอธิบาย**: แม้ทั้งสอง container จะเก็บข้อมูล key-value จำนวนเท่ากัน แต่ `unordered_map`
เร็วกว่าประมาณ 12 เท่า เพราะการค้นหาแต่ละครั้งของมันใช้แค่การคำนวณ hash แล้วกระโดดไปยัง
bucket ที่ถูกต้องทันที (O(1) เฉลี่ย) ในขณะที่ `map` ต้องเดินไล่ผ่านโครงสร้างต้นไม้ทีละชั้น
(O(log n)) — สำหรับ N = 500,000 นั้น log₂(500000) ≈ 19 ขั้นตอนต่อการค้นหาหนึ่งครั้ง ซึ่ง
มากกว่า "การคำนวณ hash หนึ่งครั้ง" ของ `unordered_map` อย่างชัดเจน ผลต่างนี้จะยิ่งเห็นชัด
ขึ้นเรื่อยๆ เมื่อ N มีขนาดใหญ่ขึ้น เพราะ O(log n) เติบโตช้ากว่า O(n) ก็จริง แต่ก็ยังคง
เติบโตตาม n ในขณะที่ O(1) ไม่เติบโตเลย

### แนวทางเฉลยข้อ 5

```cpp
#include <iostream>
#include <map>
#include <string>

int main() {
    std::multimap<char, std::string> students_by_grade;
    students_by_grade.insert({'A', "Somchai"});
    students_by_grade.insert({'B', "Suda"});
    students_by_grade.insert({'A', "Anan"});
    students_by_grade.insert({'C', "Malee"});
    students_by_grade.insert({'A', "Wichai"});

    // เดิน iterate ทั้ง multimap ทีเดียว -- key จะถูกจัดกลุ่มติดกันโดยอัตโนมัติเสมอ (เรียงตาม key)
    char current_grade = '\0';
    for (const auto& [grade, name] : students_by_grade) {
        if (grade != current_grade) {
            std::cout << "\nเกรด " << grade << ":\n";
            current_grade = grade;
        }
        std::cout << "  " << name << '\n';
    }

    return 0;
}
```

ผลลัพธ์:

```
เกรด A:
  Somchai
  Anan
  Wichai

เกรด B:
  Suda

เกรด C:
  Malee
```

จุดสำคัญของเฉลยนี้: เราไม่จำเป็นต้อง sort หรือจัดกลุ่มข้อมูลเองเลย เพราะ `multimap`
รับประกันว่าสมาชิกที่มี key เดียวกันจะถูกเก็บ**ติดกันเสมอ**ตามลำดับที่ insert เข้าไป
(insertion order ภายในกลุ่ม key เดียวกัน) และกลุ่มต่างๆ จะเรียงตาม key จากน้อยไปมาก
โดยอัตโนมัติ — โค้ดจึงแค่ตรวจสอบว่า `grade` เปลี่ยนไปจากรอบก่อนหน้าหรือไม่ เพื่อรู้ว่า
เมื่อไหร่ควรขึ้นหัวข้อกลุ่มใหม่ ไม่ต้องใช้ `equal_range()` ก็ได้ถ้าต้องการวนดูทุกกลุ่ม
พร้อมกันแบบนี้ (แต่ถ้าต้องการดูแค่กลุ่มเดียว เช่น "เกรด A มีใครบ้าง" `equal_range('A')`
จะสะดวกกว่าตามที่แสดงในหัวข้อ 61.8)

### แนวทางเฉลยข้อ 6

ลองคอมไพล์โค้ดที่ผิดดูก่อน:

```cpp
#include <string>
#include <unordered_map>

struct Point {
    int x;
    int y;
};

int main() {
    std::unordered_map<Point, std::string> point_names;   // ERROR: ไม่มี std::hash<Point>
    return 0;
}
```

```bash
$ g++ -Wall -Wextra -Wpedantic -std=c++17 ex6_fail.cpp -o ex6
ex6_fail.cpp: In function 'int main()':
ex6_fail.cpp:10:44: error: use of deleted function 'std::unordered_map<...>::unordered_map()'
   10 |     std::unordered_map<Point, std::string> point_names;
      |                                            ^~~~~~~~~~~
```

ในขณะที่ `std::map` ใช้ `Point` เป็น key ได้ทันที ถ้าเรา implement `operator<` ให้:

```cpp
#include <iostream>
#include <map>
#include <string>

struct Point {
    int x;
    int y;
};

bool operator<(const Point& a, const Point& b) {
    if (a.x != b.x) {
        return a.x < b.x;
    }
    return a.y < b.y;
}

int main() {
    std::map<Point, std::string> point_names;   // ใช้ได้ เพราะ map ต้องการแค่ operator<
    point_names[{0, 0}] = "Origin";
    point_names[{1, 1}] = "Diagonal";

    for (const auto& [p, name] : point_names) {
        std::cout << "(" << p.x << ", " << p.y << ") -> " << name << '\n';
    }

    return 0;
}
```

ผลลัพธ์:

```
(0, 0) -> Origin
(1, 1) -> Diagonal
```

**คำอธิบาย**: `std::map`/`std::set` ต้องการแค่วิธี**เปรียบเทียบ**ระหว่าง key สอง
ตัวว่าตัวไหน "น้อยกว่า" ตัวไหน (ผ่าน `operator<` ซึ่งเป็น default comparator หรือจะส่ง
comparator เองก็ได้อย่างที่เห็นในเฉลยข้อ 4) เพื่อใช้ตัดสินใจว่าจะวาง node นั้นไว้ตำแหน่งไหน
ของ Red-Black Tree — เป็น requirement ที่ implement ให้ struct ใดๆ ได้ง่าย

แต่ `std::unordered_map`/`std::unordered_set` ต้องการทั้ง **`std::hash<Key>`** (เพื่อคำนวณ
ว่า key นี้ควรอยู่ bucket ไหน) **และ `operator==`** (เพื่อเทียบว่า key ที่ hash ชนกันคือ
ตัวเดียวกันจริงหรือไม่) — มาตรฐาน C++ **ไม่ได้ generate `std::hash` ให้ struct ที่ผู้ใช้
กำหนดเองโดยอัตโนมัติ** (ต่างจาก `operator<`/`operator==` ที่ในบางกรณี compiler ช่วย
generate ให้ได้ผ่าน `= default` ตั้งแต่ C++20) ผู้เขียนโค้ดต้อง**เขียน specialization ของ
`std::hash<Point>` เอง** ก่อนถึงจะใช้ `Point` เป็น key ของ `unordered_map` ได้ — ซึ่งมี
ขั้นตอนมากกว่าการ implement `operator<` เพียงตัวเดียวสำหรับ `map` อย่างชัดเจน นี่คือเหตุผล
ในทางปฏิบัติข้อหนึ่งที่บางครั้งวิศวกรเลือกใช้ `map` แทน `unordered_map` เมื่อ key เป็น
custom type ที่ซับซ้อนและยังไม่คุ้นเคยกับการเขียน hash specialization

---

## สรุปท้ายบท

ใน Part นี้เราได้เจาะลึก container ตระกูล Associative ของ STL ซึ่งใช้สำหรับค้นหาข้อมูล
ด้วย key อย่างมีประสิทธิภาพ:

- `std::map`/`std::set` ใช้ **Red-Black Tree** (self-balancing BST) เบื้องหลัง ทำให้ทุก
  operation เป็น **O(log n) รับประกันในทุกกรณี** พร้อมเรียงลำดับตาม key ให้อัตโนมัติเสมอ
- `operator[]` ของ `map` **สร้าง element ใหม่ทันทีถ้า key ไม่มีอยู่** — ต้องใช้ `find()`
  หรือ `count()` แทนเสมอเมื่อแค่ต้องการเช็คการมีอยู่ ไม่ใช่ `[]`
- `std::unordered_map`/`std::unordered_set` ใช้ **Hash Table** เบื้องหลัง (แนวคิดเดียวกับ
  ที่เขียนเองใน Part 22) ให้ complexity **O(1) โดยเฉลี่ย** แต่ไม่รับประกันลำดับใดๆ
- เมื่อไม่ต้องการ ordering ให้เลือก `unordered_map`/`unordered_set` เพราะเร็วกว่ามาก
  ในทางปฏิบัติ แต่เมื่อต้องการลำดับ, range query, หรือ min/max อย่างรวดเร็ว ต้องใช้
  `map`/`set` เพราะเป็นสิ่งที่ hash table ให้ไม่ได้
- `std::multimap`/`std::multiset` เหมาะกับการจัดกลุ่มข้อมูลตาม key ที่ซ้ำกันได้ ผ่าน
  `equal_range()` และ `count()`

ใน **Part 62** เราจะเจาะลึกเรื่อง **STL Iterator** อย่างเป็นระบบ ทั้งประเภทของ iterator
ทั้งหมด (Input, Output, Forward, Bidirectional, Random Access) ที่เราได้เจอผ่านๆ มาใน
Part 59-61 และวิธีเขียน Custom Iterator ของเราเองสำหรับ container ที่สร้างขึ้นเอง

**ต่อไป:** [Part 62 — STL Iterator แบบเจาะลึก](./part-062-stl-iterators.md)
