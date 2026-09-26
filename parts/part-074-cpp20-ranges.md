# Part 74: C++20 Ranges (Step 585–592)

> Module F — Modern C++ (C++11 ถึง C++23) | Part 74 จาก 125
> Step ที่ครอบคลุมใน Part นี้: Step 585–592
> Part ก่อนหน้า: [Part 73 — C++20 Concepts](./part-073-cpp20-concepts.md) | Part ถัดไป: [Part 75 — C++20 Coroutines](./part-075-cpp20-coroutines.md)

## เป้าหมายการเรียนรู้ (Learning Objectives)

หลังจบ Part นี้ ผู้เรียนจะสามารถ:

1. อธิบายได้ว่าปัญหาสองอย่างที่ Ranges แก้คืออะไร: การต้องส่ง `begin()`/`end()` ซ้ำๆ ทุกครั้งที่
   เรียก Algorithm และการ Chain หลาย Operation เข้าด้วยกันจนอ่านยาก
2. ใช้ `std::ranges::sort`, `std::ranges::find` และ Algorithm อื่นๆ ในรูปแบบ Ranges แทนการ
   เขียนแบบ `std::sort`/`std::find` เดิมที่เรียนใน Part 63
3. อธิบายแนวคิด View และ Lazy Evaluation ได้ พร้อมใช้งาน `std::views::filter` และ
   `std::views::transform`
4. เชื่อม View หลายตัวเข้าด้วยกันด้วย Pipe Operator (`|`) เพื่อสร้าง Pipeline แบบ Functional
   Programming
5. เขียนตัวอย่างจริงที่กรองและแปลงข้อมูลจาก `std::vector` ด้วย Ranges Pipeline เปรียบเทียบกับ
   วิธีเขียนแบบเดิมด้วย `std::copy_if`/`std::transform` จาก Part 63
6. เข้าใจข้อจำกัดสำคัญของ View เช่นเรื่อง Dangling Reference และรู้วิธี Materialize View ให้
   เป็น Container จริง

---

## บริบท: ทำไม C++20 ถึงต้องมี Ranges

ใน Part 63 เราเรียนเรื่อง STL Algorithm อย่างเจาะลึกไปแล้ว ซึ่ง Algorithm ทั้งหมดใน
`<algorithm>` (เช่น `std::sort`, `std::find`, `std::copy_if`, `std::transform`) ถูกออกแบบมา
ตั้งแต่ C++98 ให้ทำงานผ่าน **Iterator คู่หนึ่ง** เสมอ (`begin()` และ `end()`) การออกแบบแบบนี้
ยืดหยุ่นมาก (ใช้ได้กับ Array, Linked List, หรือแม้แต่ Stream) แต่ก็มีต้นทุนสองอย่างที่เจอกัน
ทุกวันในโค้ดจริง:

1. **ต้องส่ง `begin()`/`end()` ซ้ำๆ ทุกครั้ง** — `std::sort(v.begin(), v.end())` ต้องพิมพ์ชื่อ
   `v` ถึงสองครั้ง และเปิดช่องให้เกิดบั๊กแบบส่ง `begin()` ของ Container หนึ่งคู่กับ `end()` ของ
   อีก Container โดยไม่ตั้งใจ (Compiler ตรวจจับไม่ได้เสมอไปในโค้ดที่ซับซ้อน)
2. **การ Chain หลาย Operation เข้าด้วยกันอ่านยาก** — ถ้าต้องการ "กรองข้อมูล แล้วแปลงข้อมูล
   แล้วเรียงลำดับ" ด้วย Algorithm แบบเดิม ต้องสร้าง Container ชั่วคราวระหว่างแต่ละขั้นตอน
   ทำให้โค้ดกระจัดกระจายเป็นหลาย Statement และเปลือง Memory ในการเก็บผลลัพธ์ระหว่างทาง

C++20 แก้ปัญหาทั้งสองข้อนี้ด้วย **Ranges Library** (`<ranges>`) ซึ่งเปลี่ยนมุมมองจาก "ทำงาน
กับ Iterator คู่หนึ่ง" มาเป็น "ทำงานกับ Range เดียว" (Range คือสิ่งใดๆ ที่มี `begin()` และ
`end()` — Container ของ STL ทุกตัวถือเป็น Range โดยอัตโนมัติ) และเพิ่มแนวคิดใหม่ที่เรียกว่า
**View** ที่ทำให้ Chain การประมวลผลหลายขั้นตอนเข้าด้วยกันได้แบบ Lazy (คำนวณเมื่อจำเป็นเท่านั้น)
โดยไม่ต้องสร้าง Container ชั่วคราวระหว่างทางเลย

---

## 74.1 std::ranges::sort / std::ranges::find เทียบกับแบบเดิม (Step 585–586)

### ปัญหาการส่ง begin()/end() ซ้ำๆ

```cpp
#include <algorithm>
#include <iostream>
#include <vector>

int main() {
    std::vector<int> nums{5, 3, 8, 1, 9, 2};

    // วิธีเดิม (ก่อน C++20, ที่เรียนใน Part 63): ต้องส่ง begin()/end() ทุกครั้ง
    std::sort(nums.begin(), nums.end());
    auto it_old = std::find(nums.begin(), nums.end(), 8);
    std::cout << "วิธีเดิม พบ 8 ที่ index: " << (it_old - nums.begin()) << "\n";

    return 0;
}
```

สังเกตว่าต้องพิมพ์ `nums` ถึงสองครั้งในทุกการเรียก Algorithm — ถ้ามีการรีแฟกเตอร์โค้ดแล้วเผลอ
เปลี่ยน `nums.begin()` เป็น Container อื่นแต่ลืมเปลี่ยน `nums.end()` ตาม (หรือกลับกัน) จะได้
Undefined Behavior ทันทีโดยที่ Compiler อาจไม่เตือนอะไรเลย เพราะทั้งสอง Iterator เป็น Type
เดียวกัน

### วิธีใหม่: std::ranges::sort และ std::ranges::find

C++20 เพิ่ม Algorithm ชุดใหม่ใน Namespace `std::ranges` ที่รับ **Range ทั้งก้อน** (เช่น
`std::vector` ทั้งตัว) แทนที่จะรับ Iterator คู่หนึ่ง:

```cpp
#include <algorithm>
#include <iostream>
#include <ranges>
#include <vector>

int main() {
    std::vector<int> nums{5, 3, 8, 1, 9, 2};

    // วิธีใหม่ (C++20 ranges): ส่ง container ตรงๆ ไม่ต้องเรียก .begin()/.end() เอง
    // ลดโอกาสพลาดแบบส่ง begin() ของ container หนึ่งคู่กับ end() ของอีก container
    std::ranges::sort(nums);
    auto it_new = std::ranges::find(nums, 8);
    std::cout << "วิธีใหม่ พบ 8 ที่ index: " << (it_new - nums.begin()) << "\n";

    return 0;
}
```

ผลลัพธ์ (ทั้งสองเวอร์ชันให้ผลเหมือนกัน):

```
วิธีใหม่ พบ 8 ที่ index: 4
```

### เปรียบเทียบ Algorithm แบบเดิมกับแบบ Ranges

| ประเด็น | `std::sort`/`std::find` (Part 63) | `std::ranges::sort`/`std::ranges::find` (C++20) |
|---|---|---|
| Namespace | `std` | `std::ranges` |
| รับพารามิเตอร์ | Iterator คู่ (`begin`, `end`) | Range เดียว (Container ทั้งตัว) หรือ Iterator คู่ก็ยังใช้ได้ |
| ความเสี่ยง Mismatch Iterator | มี (ถ้าส่ง begin/end จากคนละ Container) | ไม่มี เพราะส่ง Container เดียวไม่ต้องจับคู่เอง |
| Type Safety | ตรวจสอบผ่าน Template ทั่วไป | ตรวจสอบด้วย Concept (เรียนใน Part 73) — Error ชัดเจนกว่าเมื่อใช้ผิด |
| รองรับ Projection (แปลงค่าก่อนเปรียบเทียบ) | ไม่มี ต้องเขียน Comparator เอง | มีในตัว (ดูหัวข้อ 74.7) |
| ใช้กับ View ได้ไหม | ไม่ได้ | ได้โดยตรง |

**สิ่งที่ Compiler ตรวจสอบเบื้องหลัง**: `std::ranges::sort` ถูกกำหนดด้วย Concept ที่ชื่อ
`std::ranges::random_access_range` ทำให้ถ้าส่ง Range ที่ไม่รองรับ Random Access (เช่น
`std::list` ที่เรียนใน Part 60) เข้าไป จะได้ Error แบบ Concept ที่ชัดเจน (`constraints not
satisfied`) แทนที่จะเป็น Error ลึกๆ ข้างในเนื้อหาของ Algorithm แบบที่เห็นใน Part 73

---

## 74.2 Views: แนวคิด Lazy Evaluation (Step 587)

### View คืออะไร

**View** คือ "มุมมอง" ของข้อมูลต้นฉบับ ที่ **ไม่ Copy ข้อมูลใหม่** และ **ไม่คำนวณอะไรจนกว่า
จะถูก Iterate จริง** (เรียกว่า Lazy Evaluation) ต่างจาก Algorithm แบบ `std::copy_if` หรือ
`std::transform` ใน Part 63 ที่ต้องสร้าง Container ปลายทางขึ้นมาเก็บผลลัพธ์ทันที

### std::views::filter

```cpp
#include <iostream>
#include <ranges>
#include <vector>

int main() {
    std::vector<int> nums{1, 2, 3, 4, 5, 6, 7, 8, 9, 10};

    // views::filter สร้าง "มุมมอง" (view) ของข้อมูลเดิม โดยยังไม่คำนวณจริงจนกว่าจะถูก iterate
    // (lazy evaluation) — ไม่ copy ข้อมูลใหม่ ไม่สร้าง container ใหม่
    auto even_view = std::views::filter(nums, [](int n) { return n % 2 == 0; });

    std::cout << "เลขคู่: ";
    for (int n : even_view) {
        std::cout << n << " ";
    }
    std::cout << "\n";

    return 0;
}
```

ผลลัพธ์:

```
เลขคู่: 2 4 6 8 10
```

บรรทัด `auto even_view = std::views::filter(nums, ...)` **ไม่ได้กรองข้อมูลใดๆ ทันที** มันแค่
สร้าง Object ตัวหนึ่ง (`even_view`) ที่ "จำ" ไว้ว่าต้องกรองข้อมูลจาก `nums` ด้วยเงื่อนไขอะไร
การกรองจริงๆ จะเกิดขึ้นก็ต่อเมื่อ `for (int n : even_view)` เริ่มวน Loop เท่านั้น และจะกรอง
ทีละตัวไปเรื่อยๆ ไม่ใช่กรองทั้งหมดล่วงหน้าในครั้งเดียว

### std::views::transform

```cpp
// views::transform สร้าง view ที่แปลงค่าทีละตัวตอน iterate เท่านั้น
auto squared_view = std::views::transform(nums, [](int n) { return n * n; });

std::cout << "ยกกำลังสอง: ";
for (int n : squared_view) {
    std::cout << n << " ";
}
std::cout << "\n";
```

ผลลัพธ์:

```
ยกกำลังสอง: 1 4 9 16 25 36 49 64 81 100
```

### ทำไม Lazy Evaluation ถึงสำคัญ

| ประเด็น | Algorithm แบบ Eager (Part 63) เช่น `std::copy_if` | View แบบ Lazy (`std::views::filter`) |
|---|---|---|
| เวลาคำนวณ | ทำงานทันทีตอนเรียก คำนวณครบทุกตัวก่อน | รอจนกว่าจะถูก Iterate จริง คำนวณทีละตัว |
| การใช้ Memory | ต้องสร้าง Container ใหม่เก็บผลลัพธ์ | ไม่สร้าง Container ใหม่ ใช้ Memory คงที่ |
| เหมาะกับข้อมูลขนาดใหญ่/ไม่รู้จบไหม | ต้องรอครบก่อนถึงใช้ผลลัพธ์ได้ | ใช้ร่วมกับ `std::views::iota` (ลำดับไม่จำกัด) ได้ เพราะคำนวณทีละตัวตามที่ต้องใช้จริง |
| การ Chain หลายขั้นตอน | ต้องสร้าง Container ชั่วคราวระหว่างแต่ละขั้นตอน | Chain กันได้โดยไม่มี Container ชั่วคราวเลย (หัวข้อถัดไป) |

---

## 74.3 การ Chain View ด้วย Pipe Operator (Step 588–589)

จุดเด่นที่ทำให้ Ranges เปลี่ยนวิธีเขียนโค้ด C++ ไปอย่างสิ้นเชิงคือ **Pipe Operator (`|`)**
ที่ยืมแนวคิดมาจาก Shell Script บน Linux (ที่เรียนพื้นฐานไปตั้งแต่ Part 29 เรื่อง Pipe/FIFO)
และภาษาสาย Functional Programming — ข้อมูลไหลจากซ้ายไปขวาผ่านแต่ละ View เหมือนสายพาน
ลำเลียง:

```cpp
#include <iostream>
#include <ranges>
#include <vector>

int main() {
    std::vector<int> nums{1, 2, 3, 4, 5, 6, 7, 8, 9, 10};

    // ใช้ pipe operator (|) เชื่อม view หลายตัวต่อกันเป็น pipeline
    // อ่านจากซ้ายไปขวาเหมือนสายพานลำเลียงข้อมูล: กรองก่อน แล้วค่อยแปลง
    auto pipeline = nums
                   | std::views::filter([](int n) { return n % 2 == 0; })
                   | std::views::transform([](int n) { return n * n; });

    std::cout << "เลขคู่ที่ยกกำลังสองแล้ว: ";
    for (int n : pipeline) {
        std::cout << n << " ";
    }
    std::cout << "\n";

    // เพิ่ม views::take เพื่อจำกัดจำนวนผลลัพธ์ (lazy เช่นกัน หยุดคำนวณทันทีที่ครบจำนวน)
    auto first_two = nums
                    | std::views::filter([](int n) { return n % 2 == 0; })
                    | std::views::transform([](int n) { return n * n; })
                    | std::views::take(2);

    std::cout << "เอาแค่ 2 ตัวแรก: ";
    for (int n : first_two) {
        std::cout << n << " ";
    }
    std::cout << "\n";

    return 0;
}
```

ผลลัพธ์:

```
เลขคู่ที่ยกกำลังสองแล้ว: 4 16 36 64 100
เอาแค่ 2 ตัวแรก: 4 16
```

ตัวอย่าง `first_two` แสดงพลังของ Lazy Evaluation ได้ชัดเจนมาก: Pipeline นี้จะกรองและแปลงค่า
**แค่พอให้ได้ผลลัพธ์ 2 ตัว** เท่านั้น แล้วหยุดทันที ไม่จำเป็นต้องประมวลผลเลข 6 ตัวสุดท้ายของ
`nums` เลย (เพราะได้ 4, 16 ครบ 2 ตัวตั้งแต่ตรวจ `2` และ `4` ในลิสต์ต้นฉบับ) ซึ่งถ้าเขียนด้วย
Algorithm แบบ Eager ของ Part 63 จะต้องกรองและแปลงข้อมูล**ทั้งหมด**ก่อนแล้วค่อยตัดมาแค่ 2 ตัว
ทีหลัง เปลืองการคำนวณโดยไม่จำเป็น

### views::iota: สร้างลำดับตัวเลขแบบ Lazy

`std::views::iota` สร้างลำดับตัวเลขต่อเนื่อง (คล้าย `range()` ของภาษา Python) โดยไม่ต้องสร้าง
`std::vector` เก็บตัวเลขทั้งหมดขึ้นมาก่อน:

```cpp
#include <iostream>
#include <ranges>
#include <vector>

int main() {
    // views::iota สร้างลำดับตัวเลขต่อเนื่อง (คล้าย range() ใน Python) แบบ lazy
    // ไม่ต้องสร้าง vector ของเลข 1-20 ขึ้นมาจริงๆ ก่อน
    auto pipeline = std::views::iota(1, 21)
                   | std::views::filter([](int n) { return n % 3 == 0; })
                   | std::views::transform([](int n) { return n * n; })
                   | std::views::reverse;

    std::cout << "ตัวคูณของ 3 ตั้งแต่ 1-20 ยกกำลังสอง เรียงจากมากไปน้อย: ";
    for (int n : pipeline) {
        std::cout << n << " ";
    }
    std::cout << "\n";

    // views::drop ข้ามสมาชิกช่วงต้น, views::take เอาแค่บางส่วน — ใช้ทำ pagination ได้ง่ายๆ
    std::vector<int> data{10, 20, 30, 40, 50, 60, 70, 80};
    auto page2 = data | std::views::drop(3) | std::views::take(3);

    std::cout << "หน้า 2 (ข้าม 3 ตัวแรก เอา 3 ตัวถัดไป): ";
    for (int n : page2) {
        std::cout << n << " ";
    }
    std::cout << "\n";

    return 0;
}
```

ผลลัพธ์:

```
ตัวคูณของ 3 ตั้งแต่ 1-20 ยกกำลังสอง เรียงจากมากไปน้อย: 324 225 144 81 36 9
หน้า 2 (ข้าม 3 ตัวแรก เอา 3 ตัวถัดไป): 40 50 60
```

### สรุป View ที่ใช้บ่อยที่สุดใน `<ranges>`

| View | ความหมาย |
|---|---|
| `std::views::filter(range, pred)` | เก็บเฉพาะสมาชิกที่ `pred` คืน `true` |
| `std::views::transform(range, func)` | แปลงสมาชิกแต่ละตัวด้วย `func` |
| `std::views::take(range, n)` | เอาแค่ `n` ตัวแรก |
| `std::views::drop(range, n)` | ข้าม `n` ตัวแรก เอาที่เหลือ |
| `std::views::reverse(range)` | กลับลำดับสมาชิก (ต้องเป็น Bidirectional Range) |
| `std::views::iota(start, end)` | สร้างลำดับตัวเลข `start` ถึง `end - 1` แบบ Lazy |

---

## 74.4 ตัวอย่างจริง: กรองและแปลงข้อมูลจาก vector ของ struct (Step 590)

มาดูตัวอย่างที่ใกล้เคียงงานจริงมากขึ้น: มีรายชื่อพนักงาน อยากได้ "ชื่อของพนักงานแผนก
Engineering ทั้งหมด" เปรียบเทียบวิธีเขียนแบบเดิม (Part 63) กับแบบ Ranges Pipeline:

```cpp
#include <algorithm>
#include <iostream>
#include <ranges>
#include <string>
#include <vector>

struct Employee {
    std::string name;
    std::string department;
    double salary;
};

int main() {
    std::vector<Employee> staff{
        {"Anan",   "Engineering", 55000},
        {"Bee",    "Sales",       32000},
        {"Chai",   "Engineering", 68000},
        {"Duangta","Marketing",   40000},
        {"Eak",    "Engineering", 47000},
    };

    // ===== วิธีเดิม (สไตล์ Part 63): ใช้ copy_if + transform สอง loop แยกกัน =====
    std::vector<Employee> eng_old;
    std::copy_if(staff.begin(), staff.end(), std::back_inserter(eng_old),
                 [](const Employee& e) { return e.department == "Engineering"; });

    std::vector<std::string> names_old;
    std::transform(eng_old.begin(), eng_old.end(), std::back_inserter(names_old),
                    [](const Employee& e) { return e.name; });

    std::cout << "วิธีเดิม: ";
    for (const auto& n : names_old) std::cout << n << " ";
    std::cout << "\n";

    // ===== วิธีใหม่ (ranges pipeline): กรองแล้วแปลงในบรรทัดเดียว อ่านเป็นขั้นตอนตามลำดับ =====
    auto eng_names_view = staff
        | std::views::filter([](const Employee& e) { return e.department == "Engineering"; })
        | std::views::transform([](const Employee& e) { return e.name; });

    std::cout << "วิธีใหม่ (ranges): ";
    for (const auto& n : eng_names_view) std::cout << n << " ";
    std::cout << "\n";

    return 0;
}
```

ผลลัพธ์ (ทั้งสองวิธีให้ผลเหมือนกัน):

```
วิธีเดิม: Anan Chai Eak
วิธีใหม่ (ranges): Anan Chai Eak
```

### เปรียบเทียบสองวิธีอย่างละเอียด

| ประเด็น | วิธีเดิม (Part 63) | วิธีใหม่ (Ranges Pipeline) |
|---|---|---|
| จำนวน Container ชั่วคราวที่ต้องสร้าง | 2 ตัว (`eng_old`, `names_old`) | 0 ตัว (จนกว่าจะ Materialize จริง) |
| จำนวนครั้งที่ข้อมูลถูกวน Loop | 2 ครั้ง (`copy_if` แล้ว `transform`) | มองจากภายนอกเหมือนวนครั้งเดียวตอน `for` |
| ลำดับการอ่านโค้ด | ต้องอ่านย้อนไปมาระหว่าง Container กับ Loop | อ่านจากบนลงล่างตามลำดับขั้นตอนจริง (กรอง → แปลง) |
| ต้องตั้งชื่อ Container ชั่วคราว | ต้องตั้งชื่อ `eng_old` ทั้งที่เป็นแค่ผลลัพธ์ระหว่างทาง | ไม่ต้องตั้งชื่อ ไม่มี Container ระหว่างทางให้ตั้งชื่อ |
| การใช้ Memory | จองพื้นที่จริงสำหรับทั้ง `eng_old` และ `names_old` | ไม่จองพื้นที่เพิ่มจนกว่าจะ Materialize |

### Materialize View ให้เป็น Container จริง

View เป็นแค่ "มุมมอง" ที่อ้างอิงข้อมูลต้นฉบับอยู่เสมอ ถ้าต้องการนำผลลัพธ์ไปเก็บเป็น
`std::vector` จริงๆ (เช่น เพื่อส่งต่อให้ฟังก์ชันอื่น หรือเก็บไว้ใช้ทีหลังหลังจากข้อมูลต้นฉบับ
เปลี่ยนไปแล้ว) ต้อง Copy ออกมาด้วยตัวเอง เพราะ **C++20 ยังไม่มี `std::ranges::to`**
(ฟังก์ชันนี้เพิ่งถูกเพิ่มเข้ามาใน **C++23** ซึ่งจะพูดถึงภาพรวมใน Part 77):

```cpp
// การ "แปลง view ให้เป็น container จริง" (materialize) — ใน C++20 ยังไม่มี
// std::ranges::to (เพิ่มใน C++23) จึงต้อง copy ออกมาด้วยตัวเอง
std::vector<std::string> eng_names_vec(eng_names_view.begin(), eng_names_view.end());
std::cout << "จำนวนวิศวกร: " << eng_names_vec.size() << " คน\n";
```

ผลลัพธ์:

```
จำนวนวิศวกร: 3 คน
```

วิธีนี้ใช้ Constructor ของ `std::vector` ที่รับ Iterator คู่หนึ่ง (แบบเดียวกับที่เรียนใน
Part 59) — เมื่อ Constructor นี้ Iterate ผ่าน `eng_names_view` จริงๆ นั่นคือจุดที่การกรองและ
การแปลงค่าทั้งหมดถูกคำนวณออกมาจริง แล้วผลลัพธ์ถูก Copy เข้าไปเก็บใน `eng_names_vec`

### รายการ Algorithm ใน std::ranges ที่ใช้บ่อยเพิ่มเติม

นอกจาก `sort` และ `find` ที่สาธิตไปแล้ว `std::ranges` ยังมี Algorithm อีกจำนวนมากที่เป็น
เวอร์ชัน Ranges ของ Algorithm ใน `<algorithm>` ที่เรียนไปแล้วใน Part 63 ตารางนี้สรุปตัวที่พบ
บ่อยที่สุด (ทุกตัวรับ Container ทั้งตัวได้โดยตรง เหมือน `sort`/`find`):

| Algorithm แบบเดิม (Part 63) | เวอร์ชัน Ranges |
|---|---|
| `std::for_each(v.begin(), v.end(), f)` | `std::ranges::for_each(v, f)` |
| `std::count_if(v.begin(), v.end(), pred)` | `std::ranges::count_if(v, pred)` |
| `std::all_of` / `std::any_of` / `std::none_of` | `std::ranges::all_of` / `std::ranges::any_of` / `std::ranges::none_of` |
| `std::max_element(v.begin(), v.end())` | `std::ranges::max_element(v)` |
| `std::reverse(v.begin(), v.end())` | `std::ranges::reverse(v)` |
| `std::unique(v.begin(), v.end())` | `std::ranges::unique(v)` |
| `std::copy(src.begin(), src.end(), dst_it)` | `std::ranges::copy(src, dst_it)` |

รูปแบบเหมือนกันหมด: Algorithm ตัวไหนที่เดิมรับ Iterator คู่หนึ่ง เวอร์ชัน `std::ranges::` จะ
รับ Range เดียวแทนได้เสมอ (แต่ก็ยังรับ Iterator คู่แบบเดิมได้ด้วยถ้าจำเป็น เพื่อ Backward
Compatibility) ทำให้เมื่อคุ้นเคยกับรูปแบบนี้แล้ว การเปลี่ยนจาก Algorithm แบบเดิมมาเป็น Ranges
ทำได้เกือบจะแค่เติมคำว่า `ranges::` แล้วตัด `.begin()`/`.end()` ออกเท่านั้น

---

## 74.5 ข้อควรระวังของ View: Dangling Reference (Step 591)

เพราะ View เป็นแค่ "มุมมอง" ที่อ้างอิงข้อมูลต้นฉบับ (ไม่ Copy ข้อมูล) จึงมีความเสี่ยงสำคัญ:
**ถ้าข้อมูลต้นฉบับถูกทำลายไปแล้ว แต่ View ยังถูกใช้งานอยู่ จะเกิด Dangling Reference**
(ปัญหาเดียวกับ Dangling Pointer ที่เรียนใน Part 9 และ 11) ลองดูตัวอย่างที่อันตราย:

```cpp
#include <ranges>
#include <vector>

std::vector<int> make_numbers() {
    return {5, 3, 8, 1, 9};
}

int main() {
    // อันตราย: find บน vector ชั่วคราว (rvalue) ที่จะถูกทำลายทันทีหลังบรรทัดนี้จบ
    // ถ้าเก็บ iterator ไว้ใช้ต่อ จะกลายเป็น dangling iterator ทันที
    auto it = std::ranges::find(make_numbers(), 8);
    return *it;   // UB ถ้าคอมไพล์ผ่าน — แต่ ranges จะไม่ยอมให้คอมไพล์ผ่านตั้งแต่แรก
}
```

`make_numbers()` คืนค่า `std::vector<int>` แบบ Temporary Object (Rvalue) ที่จะถูกทำลายทันที
หลังจบ Statement (ตามกฎ Object Lifetime ที่เรียนใน Part 43 และ 70) ถ้า `std::ranges::find`
คืน Iterator ที่ยังชี้เข้าไปใน Memory ของ Vector ที่ถูกทำลายไปแล้ว การ `*it` ต่อจะเป็น
Undefined Behavior ทันที

**ข่าวดี**: นี่คือจุดที่ Ranges Library ฉลาดกว่า Algorithm แบบเดิมมาก เพราะมันตรวจจับ
สถานการณ์นี้ได้ตั้งแต่ตอน **Compile-Time** ด้วยกลไก Concept ที่ชื่อ
`std::ranges::borrowed_range` ลองคอมไพล์โค้ดข้างบนดูจริง:

```bash
g++ -Wall -Wextra -Wpedantic -std=c++20 -c dangling_demo.cpp
```

**Error Message จริงที่ได้:**

```
dangling_demo.cpp: In function 'int main()':
dangling_demo.cpp:12:12: error: no match for 'operator*' (operand type is 'std::ranges::dangling')
   12 |     return *it;   // UB ถ้าคอมไพล์ผ่าน — แต่ ranges จะไม่ยอมให้คอมไพล์ผ่านตั้งแต่แรก
      |            ^~~
```

สังเกตว่า `std::ranges::find` เมื่อพบว่า Range ที่ส่งเข้าไปเป็น Temporary ที่ไม่ปลอดภัยจะเก็บ
Iterator ต่อ จะไม่คืนค่าเป็น Iterator จริงๆ แต่คืนค่าเป็น Type พิเศษชื่อ `std::ranges::dangling`
แทน ซึ่ง Type นี้ **ไม่มี `operator*`** จึงทำให้โค้ดที่พยายาม Dereference มันไม่ผ่านการ Compile
ตั้งแต่แรก — Bug ที่เดิมทีจะเป็น Undefined Behavior ตอน Runtime (อาจจะรันได้บางครั้ง Crash
บางครั้ง แล้วแต่ดวง) ถูกจับได้ตั้งแต่ตอน Compile แทน

**วิธีแก้ที่ถูกต้อง**: เก็บ Vector ไว้ในตัวแปรก่อน แล้วค่อยเรียก `std::ranges::find` กับตัวแปร
นั้น (ซึ่งเป็น Lvalue ที่มี Lifetime ยาวพอ):

```cpp
std::vector<int> nums = make_numbers();   // เก็บไว้ในตัวแปรก่อน (Lvalue)
auto it = std::ranges::find(nums, 8);     // ปลอดภัย เพราะ nums ยังมีชีวิตอยู่ตลอด scope นี้
if (it != nums.end()) {
    return *it;
}
```

---

## 74.6 Projection: ฟีเจอร์เสริมที่ทำให้ Ranges เขียนสั้นลงอีก (Step 592)

นอกจาก View และ Pipe Operator แล้ว Algorithm ในกลุ่ม `std::ranges` ยังมีฟีเจอร์ที่เรียกว่า
**Projection** ซึ่งเป็นพารามิเตอร์เสริมที่บอกว่า "เอาค่าอะไรจาก Object มาใช้เปรียบเทียบ" โดย
ไม่ต้องเขียน Comparator/Lambda เอง:

```cpp
#include <algorithm>
#include <iostream>
#include <ranges>
#include <string>
#include <vector>

struct Employee {
    std::string name;
    double salary;
};

int main() {
    std::vector<Employee> staff{
        {"Anan", 55000}, {"Bee", 32000}, {"Chai", 68000}
    };

    // Projection: บอก ranges::sort ว่า "เอาค่าอะไรมาเทียบ" โดยไม่ต้องเขียน comparator
    // แบบเทียบ e1.salary < e2.salary เอง — เขียนกระชับและอ่านง่ายกว่ามาก
    std::ranges::sort(staff, std::ranges::less{}, &Employee::salary);

    for (const auto& e : staff) {
        std::cout << e.name << ": " << e.salary << "\n";
    }

    // ใช้ projection กับ find เพื่อค้นหาจากชื่อ โดยไม่ต้องเขียน lambda เปรียบเทียบเอง
    auto it = std::ranges::find(staff, "Bee", &Employee::name);
    if (it != staff.end()) {
        std::cout << "พบ Bee เงินเดือน: " << it->salary << "\n";
    }

    return 0;
}
```

ผลลัพธ์:

```
Bee: 32000
Anan: 55000
Chai: 68000
พบ Bee เงินเดือน: 32000
```

เทียบกับวิธีเดิมที่ต้องเขียน Lambda เปรียบเทียบเองแบบใน Part 63:

```cpp
// วิธีเดิม (Part 63): ต้องเขียน lambda ดึงค่า salary มาเปรียบเทียบเอง
std::sort(staff.begin(), staff.end(),
          [](const Employee& a, const Employee& b) { return a.salary < b.salary; });

// วิธีเดิม: find ด้วยเงื่อนไขจาก field ต้องใช้ find_if + lambda
auto it_old = std::find_if(staff.begin(), staff.end(),
                            [](const Employee& e) { return e.name == "Bee"; });
```

พารามิเตอร์ตัวที่สามของ `std::ranges::sort` และ `std::ranges::find` (`&Employee::salary`,
`&Employee::name`) คือ Pointer-to-Member (เรียนพื้นฐานไปแล้วใน Part 53 เรื่อง Static Member
และ Class Design) ที่บอกว่า "ก่อนเปรียบเทียบ ให้ดึงค่าจาก Field นี้ออกมาก่อน" ทำให้ไม่ต้อง
เขียน Lambda สั้นๆ ซ้ำๆ ทุกครั้งที่ต้องการเรียงหรือค้นหาตาม Field ใดๆ ของ Object

---

## ข้อผิดพลาดที่พบบ่อย (Common Pitfalls)

1. **เก็บ View ไว้ใช้ต่อหลังจาก Container ต้นฉบับถูกทำลายหรือแก้ไข** — View ไม่ได้ Copy
   ข้อมูล มันอ้างอิงต้นฉบับตลอดเวลา ถ้า Container ต้นฉบับถูก `push_back` จนต้อง Reallocate
   Memory ใหม่ (ทบทวนกลไกนี้ได้จาก Part 59) View เก่าที่สร้างไว้ก่อนหน้าอาจกลายเป็น Dangling
   ทันทีเหมือนกับ Iterator ที่ Invalidate
2. **คาดหวังว่า View จะคำนวณครบทุกตัวทันทีที่สร้าง** — เพราะเป็น Lazy Evaluation ถ้าเขียน
   Lambda ใน `views::filter`/`views::transform` ที่มี Side Effect (เช่น พิมพ์ค่าออกทางหน้าจอ
   หรือแก้ไขตัวแปร Global) จะเห็นว่า Side Effect นั้นเกิดขึ้น **ทุกครั้ง** ที่ View ถูก Iterate
   ไม่ใช่แค่ครั้งเดียวตอนสร้าง Pipeline — ถ้า Iterate View เดียวกันซ้ำสองรอบ Side Effect ก็จะ
   เกิดซ้ำสองรอบตามไปด้วย
3. **ลืมว่า `std::ranges::to` ยังไม่มีใน C++20** — ผู้เรียนที่เคยเห็นตัวอย่างโค้ด C++23 มา
   อาจพยายามเขียน `auto v = my_view | std::ranges::to<std::vector>();` ซึ่งจะ Compile ไม่ผ่าน
   ถ้าใช้ `-std=c++20` ต้อง Materialize ด้วย Constructor ของ Container ที่รับ Iterator คู่หนึ่ง
   แทน ตามที่แสดงในหัวข้อ 74.4
4. **ใช้ `std::views::reverse` กับ Range ที่ไม่รองรับการเดินย้อนกลับ** — View บางตัวต้องการ
   Range ที่มีคุณสมบัติเฉพาะ (เช่น `reverse` ต้องการ Bidirectional Range เป็นอย่างน้อย) ถ้าใช้
   กับ Range ที่รองรับแค่ Forward Iteration (เช่น `std::forward_list` จาก Part 60) จะได้ Error
   แบบ Concept ที่ชัดเจน แต่ก็ยังเป็นสิ่งที่ต้องระวังตั้งแต่ตอนออกแบบโค้ด
5. **สร้าง Pipeline ที่ยาวและซับซ้อนเกินไปจนอ่านยากกว่าเดิม** — Pipe Operator ทำให้เขียนโค้ด
   สั้นได้ก็จริง แต่ถ้า Chain View มากกว่า 4-5 ตัวติดกันในบรรทัดเดียว อาจทำให้อ่านยากพอๆ กับ
   โค้ดแบบเดิม ควรแยก Pipeline ยาวๆ ออกเป็นตัวแปรย่อยที่ตั้งชื่อสื่อความหมาย
6. **ลืมว่า Algorithm บางตัวใน `std::ranges` ยังต้องการ Concept เฉพาะของ Range** — เช่น
   `std::ranges::sort` ต้องการ `random_access_range` ถ้าส่ง `std::list` (ซึ่งเป็นแค่
   `bidirectional_range`) เข้าไปจะ Compile ไม่ผ่าน ต่างจาก `std::list::sort()` ที่เป็น Member
   Function เฉพาะของ `std::list` เอง (เรียนไปแล้วใน Part 60) ที่ยังใช้ได้ปกติ

---

## แบบฝึกหัดท้ายบท

1. เขียนโปรแกรมที่มี `std::vector<int>` ค่าคละกันทั้งบวกและลบ ใช้ `std::views::filter` กรอง
   เอาเฉพาะค่าบวก แล้วใช้ `std::ranges::sort` เรียงจากน้อยไปมาก (Materialize ผลลัพธ์เป็น
   `std::vector` ใหม่ก่อนเรียง เพราะ View ที่ได้จาก `filter` ไม่รองรับ Random Access เสมอไป)

2. ใช้ `std::views::iota` สร้างเลข 1 ถึง 100 แล้ว Chain ด้วย Pipe Operator ให้เหลือเฉพาะเลขที่
   หารด้วย 7 ลงตัว จากนั้นพิมพ์ผลรวมของเลขเหล่านั้นด้วย `std::accumulate` (ทบทวนจาก Part 63)
   หลังจาก Materialize เป็น `std::vector` แล้ว

3. เขียน `struct Product { std::string name; double price; int stock; };` สร้าง
   `std::vector<Product>` อย่างน้อย 5 รายการ แล้วใช้ Ranges Pipeline หา "ชื่อสินค้าที่ราคา
   มากกว่า 100 บาท เรียงจากราคามากไปน้อย" โดยใช้ Projection กับ `std::ranges::sort`

4. ทดลองเขียนโค้ดที่ทำให้เกิด `std::ranges::dangling` ด้วยตัวเอง (เหมือนตัวอย่างในหัวข้อ 74.5
   แต่เปลี่ยนเป็นใช้ `std::ranges::max_element` แทน `std::ranges::find`) คอมไพล์ดูจริงแล้ว
   บันทึก Error Message ที่ได้

5. เขียนฟังก์ชันที่รับ `std::vector<std::string>` แล้วใช้ Ranges Pipeline คืนค่าเป็น
   `std::vector<std::string>` ใหม่ที่มีเฉพาะคำที่ยาวมากกว่า 3 ตัวอักษร และแปลงทุกตัวอักษร
   เป็นตัวพิมพ์ใหญ่ (ใช้ `std::transform` ของ `<algorithm>` ผสมกับ `::toupper` ข้างใน
   `std::views::transform`)

6. เปรียบเทียบเวลาที่ใช้เขียนโค้ด (จำนวนบรรทัด/ความซับซ้อน) ระหว่างการแก้โจทย์ข้อ 3 ด้วย
   Ranges Pipeline กับการเขียนด้วย `std::copy_if` + `std::sort` + `std::transform` แบบ Part 63
   สรุปเป็นข้อคิดเห็นสั้นๆ ว่าแบบไหนอ่านง่ายกว่าในความเห็นของผู้เรียนเอง

### แนวทางเฉลยข้อ 1

```cpp
#include <algorithm>
#include <iostream>
#include <ranges>
#include <vector>

int main() {
    std::vector<int> numbers{-5, 3, -2, 8, -1, 9, 0, -7, 4};

    // กรองเอาเฉพาะค่าบวก (0 ไม่นับเป็นบวก) แล้ว materialize เป็น vector ใหม่
    auto positive_view = numbers | std::views::filter([](int n) { return n > 0; });
    std::vector<int> positives(positive_view.begin(), positive_view.end());

    // เรียงจากน้อยไปมากด้วย ranges::sort (ต้องการ random_access_range ซึ่ง vector รองรับ)
    std::ranges::sort(positives);

    std::cout << "ค่าบวกที่เรียงแล้ว: ";
    for (int n : positives) {
        std::cout << n << " ";
    }
    std::cout << "\n";

    return 0;
}
```

ผลลัพธ์ที่ควรได้:

```
ค่าบวกที่เรียงแล้ว: 3 4 8 9
```

### แนวทางเฉลยข้อ 3

```cpp
#include <algorithm>
#include <iostream>
#include <ranges>
#include <string>
#include <vector>

struct Product {
    std::string name;
    double price;
    int stock;
};

int main() {
    std::vector<Product> products{
        {"เมาส์",        250.0,  30},
        {"คีย์บอร์ด",     890.0,  15},
        {"จอมอนิเตอร์",   4500.0,  8},
        {"หูฟัง",        150.0,  20},
        {"เก้าอี้เกมมิ่ง", 3200.0,  5},
    };

    // กรองเฉพาะสินค้าราคามากกว่า 100 แล้ว materialize ก่อน เพราะ ranges::sort ต้องการ
    // random_access_range ซึ่ง view จาก filter เพียงอย่างเดียวไม่รับประกันเสมอไป
    auto expensive_view = products | std::views::filter([](const Product& p) { return p.price > 100.0; });
    std::vector<Product> expensive(expensive_view.begin(), expensive_view.end());

    // ใช้ projection เรียงจากราคามากไปน้อย (greater แทน less)
    std::ranges::sort(expensive, std::ranges::greater{}, &Product::price);

    std::cout << "สินค้าราคา > 100 บาท เรียงจากแพงไปถูก:\n";
    for (const auto& p : expensive) {
        std::cout << "  " << p.name << " : " << p.price << " บาท\n";
    }

    return 0;
}
```

ผลลัพธ์ที่ควรได้:

```
สินค้าราคา > 100 บาท เรียงจากแพงไปถูก:
  จอมอนิเตอร์ : 4500 บาท
  เก้าอี้เกมมิ่ง : 3200 บาท
  คีย์บอร์ด : 890 บาท
  เมาส์ : 250 บาท
  หูฟัง : 150 บาท
```

### แนวทางเฉลยข้อ 2

```cpp
#include <algorithm>
#include <iostream>
#include <numeric>
#include <ranges>
#include <vector>

int main() {
    auto multiples_of_7 = std::views::iota(1, 101)
                         | std::views::filter([](int n) { return n % 7 == 0; });

    std::vector<int> result(multiples_of_7.begin(), multiples_of_7.end());

    int total = std::accumulate(result.begin(), result.end(), 0);

    std::cout << "จำนวนตัวคูณของ 7 ใน 1-100: " << result.size() << " ตัว\n";
    std::cout << "ผลรวม: " << total << "\n";

    return 0;
}
```

ผลลัพธ์ที่ควรได้:

```
จำนวนตัวคูณของ 7 ใน 1-100: 14 ตัว
ผลรวม: 735
```

### แนวทางเฉลยข้อ 4

```cpp
#include <algorithm>
#include <ranges>
#include <vector>

std::vector<int> make_numbers() {
    return {5, 3, 8, 1, 9};
}

int main() {
    // เปลี่ยนจาก find มาเป็น max_element — อันตรายแบบเดียวกัน: vector ชั่วคราวถูกทำลาย
    // ก่อนที่ Iterator จะถูกใช้งาน
    auto it = std::ranges::max_element(make_numbers());
    return *it;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++20 -c ex4_dangling.cpp
```

Error Message จริงที่ได้ (เหมือนกับกรณี `find` ในหัวข้อ 74.5 เป๊ะๆ เพราะ Algorithm ทุกตัวใน
`std::ranges` ใช้กลไก `borrowed_range`/`dangling` แบบเดียวกัน):

```
ex4_dangling.cpp: In function 'int main()':
ex4_dangling.cpp:11:12: error: no match for 'operator*' (operand type is 'std::ranges::dangling')
   11 |     return *it;
      |            ^~~
```

วิธีแก้เหมือนเดิมคือเก็บผลลัพธ์ของ `make_numbers()` ไว้ในตัวแปรก่อน แล้วค่อยเรียก
`std::ranges::max_element` กับตัวแปรนั้น

---

## สรุปท้ายบท

ใน Part นี้เราได้:

- เข้าใจปัญหาสองอย่างของ Algorithm แบบเดิมที่ Ranges เข้ามาแก้: การต้องส่ง `begin()`/`end()`
  ซ้ำๆ และการ Chain หลาย Operation ที่ต้องสร้าง Container ชั่วคราวระหว่างทาง
- ใช้ `std::ranges::sort`/`std::ranges::find` แทน `std::sort`/`std::find` แบบเดิม โดยส่ง
  Container ทั้งตัวแทน Iterator คู่หนึ่ง
- เข้าใจแนวคิด View และ Lazy Evaluation ผ่าน `std::views::filter` และ `std::views::transform`
- เชื่อม View หลายตัวเข้าด้วยกันด้วย Pipe Operator (`|`) สร้าง Pipeline แบบ Functional
  Programming ที่อ่านเป็นขั้นตอนจากซ้ายไปขวา
- เขียนตัวอย่างจริงเปรียบเทียบการกรองและแปลงข้อมูลจาก `std::vector<Employee>` ระหว่างวิธีเดิม
  (Part 63) กับ Ranges Pipeline และรู้วิธี Materialize View ให้เป็น Container จริง
- เข้าใจข้อจำกัดสำคัญของ View เรื่อง Dangling Reference และเห็นว่า `std::ranges::dangling`
  ช่วยจับปัญหานี้ได้ตั้งแต่ Compile-Time แทนที่จะเป็น Undefined Behavior ตอน Runtime
- ใช้ Projection ทำให้ `std::ranges::sort`/`std::ranges::find` เขียนสั้นลงโดยไม่ต้องเขียน
  Lambda เปรียบเทียบเอง

Ranges เปลี่ยนวิธีเขียน Algorithm ให้กระชับและปลอดภัยขึ้นมาก แต่ยังเป็นแค่การเขียนโค้ดแบบ
Synchronous (ทำงานตามลำดับ รอจนเสร็จทีละขั้นตอน) ปัญหาถัดไปที่ C++20 เข้ามาแก้คือการเขียน
โค้ดที่ต้อง "หยุดรอ" อะไรบางอย่างโดยไม่บล็อกการทำงานของโปรแกรมทั้งหมด (เช่น รอข้อมูลจาก
Network หรือ I/O) ซึ่งนำไปสู่ฟีเจอร์ที่ซับซ้อนและทรงพลังที่สุดตัวหนึ่งของ C++20 — ใน
**Part 75** เราจะเรียนรู้ **C++20 Coroutines** ผ่านคีย์เวิร์ด `co_await`, `co_yield`, และ
`co_return`

**ต่อไป:** [Part 75 — C++20 Coroutines](./part-075-cpp20-coroutines.md)
