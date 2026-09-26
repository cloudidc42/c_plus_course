# Part 119: เตรียมสัมภาษณ์งาน: Data Structures & Algorithms สไตล์ LeetCode ด้วย C++ (Step 945–952)

> Module J — Professional และ World-Class Practices | Part 119 จาก 125
> Step ที่ครอบคลุมใน Part นี้: Step 945–952
> Part ก่อนหน้า: [Part 118 — พื้นฐาน Game Development ด้วย C++](./part-118-game-dev-basics.md) | Part ถัดไป: [Part 120 — เส้นทางอาชีพ: จาก Junior สู่ Senior/Staff Engineer และ Open Source](./part-120-career-path.md)

## เป้าหมายการเรียนรู้ (Learning Objectives)

หลังจบ Part นี้ ผู้เรียนจะสามารถ:

1. อธิบายรูปแบบการสัมภาษณ์งานสาย C++ ที่พบได้จริงในบริษัทซอฟต์แวร์ปัจจุบัน (2026) ทั้ง Coding
   Round, System/Low-Level Design Round, Behavioral Round และรอบเจาะลึกภาษา C++ โดยเฉพาะ
2. จดจำและประยุกต์ใช้ "Pattern" การแก้โจทย์ 4 แบบที่ครอบคลุมโจทย์ LeetCode ส่วนใหญ่ที่ออกสอบจริง:
   Two Pointers, Sliding Window, Fast & Slow Pointer และ Binary Search on Answer
3. แก้โจทย์คลาสสิกระดับ Interview 7 ข้อด้วย C++ ที่คอมไพล์และรันได้จริง 100% พร้อมวิเคราะห์
   Time/Space Complexity อย่างละเอียดทุกข้อ
4. เชื่อมโยงความรู้พื้นฐานจาก Part 19 (Linked List), Part 20 (Stack/Queue), Part 21-22
   (Tree/Hash Table) และ Part 23-25 (Sorting/Searching/Recursion/DP) เข้ากับบริบทของการสัมภาษณ์
   งานจริงที่จำกัดเวลา
5. ใช้เทคนิคการสื่อสารที่มืออาชีพใช้ระหว่างแก้โจทย์ในห้องสัมภาษณ์ (คิดออกเสียง, ถามคำถาม
   ชี้แจง edge case) ผ่านกรอบคิด UMPIRE Method
6. หลีกเลี่ยงข้อผิดพลาดที่พบบ่อยที่สุดของผู้สมัครภายใต้ความกดดันของเวลา เช่น off-by-one error,
   การลืม edge case และ integer overflow
7. เตรียมตัวเบื้องต้นสำหรับ System Design/Low-Level Design Round สำหรับตำแหน่งสาย C++ โดยเฉพาะ
   ผ่านตัวอย่าง LRU Cache ที่ผสมทั้งการเขียนโค้ดและการออกแบบระบบเข้าด้วยกัน

---

## 119.1 รูปแบบการสัมภาษณ์งานสาย C++ ทั่วไป (Step 945)

การสัมภาษณ์งานสาย Software Engineer ที่ใช้ C++ เป็นภาษาหลัก (Game Engine, Trading System,
Embedded, Infrastructure, Backend ประสิทธิภาพสูง ฯลฯ) มักมีโครงสร้างคล้ายกันทั่วโลก แม้รายละเอียด
จะต่างกันไปตามบริษัทและระดับตำแหน่ง โดยทั่วไปกระบวนการทั้งหมดแบ่งเป็นขั้นตอนดังนี้:

```
[1] Resume Screening        -> ฝ่าย HR/Recruiter คัดกรองจากประวัติ
      │
      ▼
[2] Online Assessment (OA)  -> โจทย์โค้ดอัตโนมัติ (HackerRank/CodeSignal) 60-120 นาที (บางบริษัทข้ามขั้นนี้)
      │
      ▼
[3] Phone/Video Screen      -> คุยกับ Engineer 1 คน 45-60 นาที โจทย์โค้ด 1-2 ข้อ + ถามพื้นฐาน
      │
      ▼
[4] Onsite / Virtual Onsite Loop (3-6 รอบ ในวันเดียวหรือกระจายหลายวัน)
      │
      ├─ Coding Round (Algorithm/DS)      x 1-3 รอบ
      ├─ C++ Language Deep-Dive Round     x 0-1 รอบ (เฉพาะตำแหน่งสาย C++ โดยตรง)
      ├─ System/Low-Level Design Round    x 0-2 รอบ (มากขึ้นตามระดับ Senior)
      ├─ Behavioral Round                 x 1 รอบ
      │
      ▼
[5] Hiring Committee / Offer Decision
```

### ตารางสรุปแต่ละรอบและสิ่งที่ถูกประเมิน

| รอบ | ระยะเวลา | สิ่งที่ถูกประเมิน | ตัวอย่างคำถาม |
|---|---|---|---|
| Online Assessment | 60–120 นาที | ความถูกต้องของโค้ด, ผ่าน test case ทั้งหมดหรือไม่ | โจทย์ระดับ Easy-Medium 2-4 ข้อ |
| Coding Round | 45–60 นาที | กระบวนการคิด, การเลือก Data Structure, Complexity, การจัดการ edge case, คุณภาพโค้ด | Two Sum, Merge Intervals, BFS บน grid |
| C++ Deep-Dive Round | 30–60 นาที | ความเข้าใจ Memory Model, Move Semantics, RAII, Undefined Behavior, Virtual Dispatch | "อธิบาย Rule of Five", "Virtual function ทำงานอย่างไรใน memory" |
| System/Low-Level Design | 45–60 นาที | การออกแบบ Class/API/สถาปัตยกรรมให้ขยายได้ (extensible), การจัดการ Concurrency | ออกแบบ LRU Cache, Rate Limiter, Parking Lot |
| Behavioral | 30–45 นาที | Soft skill, การทำงานเป็นทีม, การรับมือความขัดแย้ง, Leadership (สำหรับ Senior ขึ้นไป) | "เล่าเหตุการณ์ที่คุณไม่เห็นด้วยกับทีม" |

### ความแตกต่างตามระดับตำแหน่ง

น้ำหนักของแต่ละรอบจะเปลี่ยนไปตามระดับตำแหน่งที่สมัคร (รายละเอียดเรื่อง Career Level จะอธิบาย
เจาะลึกใน **Part 120**):

- **Junior/New Grad**: เน้น Coding Round เกือบทั้งหมด (2-3 รอบ) เพราะบริษัทต้องการดูพื้นฐาน
  Data Structure & Algorithm ว่าแน่นแค่ไหน ยังไม่คาดหวัง System Design ที่ลึก
- **Mid-level (2-5 ปี)**: Coding Round ลดลงเหลือ 1-2 รอบ เริ่มมี Low-Level Design 1 รอบ และ
  C++ Deep-Dive อาจถูกถามแทรกในรอบ Coding เลย
- **Senior ขึ้นไป**: Coding Round อาจเหลือแค่ 1 รอบ (แต่คาดหวังว่าจะแก้ได้เร็วและคุณภาพโค้ด
  สูงมาก) เพิ่ม System Design เต็มรอบ และ Behavioral Round จะเจาะลึกเรื่อง Leadership/Mentorship
  มากขึ้น

> **ข้อสังเกตสำคัญ**: ต่างจากตำแหน่งสาย Web/Full-stack ทั่วไปที่ System Design มักหมายถึง
> "High-Level Design" (ออกแบบระบบกระจาย, Load Balancer, Database Sharding) ตำแหน่งสาย C++
> จำนวนมาก (Game, Embedded, Trading, Infrastructure) มักเน้น **Low-Level Design (LLD)**:
> ออกแบบ Class, Interface, และการจัดการ Memory/Concurrency ของระบบเดียว (single machine)
> มากกว่า เราจะกลับมาเจาะประเด็นนี้ใน 119.8

---

## 119.2 เทคนิคจดจำ Pattern แทนการท่องจำโจทย์ (Step 946)

ความผิดพลาดที่พบบ่อยที่สุดของผู้เตรียมสัมภาษณ์มือใหม่คือ **พยายามท่องจำวิธีแก้โจทย์เป็นข้อๆ**
ซึ่งไม่ยั่งยืนเลย เพราะโจทย์จริงในห้องสัมภาษณ์แทบไม่มีทางตรงกับที่เคยฝึกมาเป๊ะๆ

วิธีที่มีประสิทธิภาพกว่ามากคือการจดจำ **Pattern** (รูปแบบการแก้ปัญหา) จำนวนไม่มาก แล้วฝึก
"มองโจทย์ใหม่ให้ทะลุ" ว่าตรงกับ Pattern ไหน ในหัวข้อนี้จะแนะนำ 4 Pattern ที่ครอบคลุมโจทย์
ส่วนใหญ่ที่ออกสอบจริง

### Pattern 1: Two Pointers

ใช้เมื่อ **ข้อมูลเรียงลำดับแล้ว (sorted)** หรือมีคุณสมบัติ monotonic บางอย่าง แล้วต้องการหาคู่/
กลุ่มของสมาชิกที่สัมพันธ์กัน โดยใช้ pointer สองตัววิ่งเข้าหากันหรือวิ่งไปทางเดียวกันด้วยความเร็ว
ต่างกัน แทนที่จะวน loop ซ้อนกัน (ลด O(n²) เหลือ O(n))

```cpp
#include <iostream>
#include <vector>

// Two Pointers บนอาร์เรย์ที่เรียงแล้ว: หาคู่ตัวเลขสองตัวที่บวกกันได้เท่ากับ target
// คืนค่าเป็น index คู่แรกที่เจอ (0-based) หรือ {-1, -1} ถ้าไม่เจอ
std::pair<int, int> two_sum_sorted(const std::vector<int>& nums, int target) {
    int left = 0;
    int right = static_cast<int>(nums.size()) - 1;

    while (left < right) {
        long long sum = static_cast<long long>(nums[left]) + nums[right];
        if (sum == target) {
            return {left, right};
        } else if (sum < target) {
            ++left;   // ผลรวมน้อยไป ต้องขยับตัวซ้ายไปทางขวาเพื่อเพิ่มค่า
        } else {
            --right;  // ผลรวมมากไป ต้องขยับตัวขวามาทางซ้ายเพื่อลดค่า
        }
    }
    return {-1, -1};
}

int main() {
    std::vector<int> nums = {1, 3, 5, 7, 9, 11, 14};

    auto [i, j] = two_sum_sorted(nums, 16);
    std::cout << "target=16 -> index (" << i << ", " << j << ") "
              << "values (" << nums[i] << ", " << nums[j] << ")\n";

    auto [k, l] = two_sum_sorted(nums, 100);
    std::cout << "target=100 -> index (" << k << ", " << l << ")\n";

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 two_pointers.cpp -o two_pointers
./two_pointers
# target=16 -> index (2, 5) values (5, 11)
# target=100 -> index (-1, -1)
```

**สัญญาณที่บอกว่าควรใช้ Two Pointers**: โจทย์พูดถึงอาร์เรย์ที่เรียงแล้ว, การหาคู่ที่ผลรวม/ผลต่าง
ตรงเงื่อนไข, การตรวจ palindrome, หรือการแบ่งพาร์ทิชันอาร์เรย์ (เช่น Dutch National Flag Problem)

### Pattern 2: Sliding Window

ใช้เมื่อโจทย์ถามหา **subarray/substring ที่ต่อเนื่องกัน (contiguous)** ที่ทำให้เงื่อนไขบางอย่าง
เป็นจริง (ผลรวมมากที่สุด, ความยาวสั้นที่สุด, ไม่มีตัวซ้ำ) แทนที่จะวน loop ตรวจทุก subarray
(O(n²) หรือแย่กว่า) เราขยับ "หน้าต่าง" (window) ไปทีละก้าว หน้าต่างมีสองแบบ: **ขนาดคงที่
(fixed-size)** และ **ขนาดยืดหยุ่น (variable-size)**

```cpp
#include <iostream>
#include <vector>

// Fixed-size Sliding Window: หาผลรวมมากที่สุดของ subarray ที่มีขนาด k พอดี
long long max_sum_fixed_window(const std::vector<int>& nums, std::size_t k) {
    if (nums.size() < k || k == 0) {
        return 0;
    }

    long long window_sum = 0;
    for (std::size_t i = 0; i < k; ++i) {
        window_sum += nums[i];
    }

    long long best = window_sum;
    for (std::size_t i = k; i < nums.size(); ++i) {
        window_sum += nums[i];           // เพิ่มสมาชิกใหม่ที่เข้ามาทางขวา
        window_sum -= nums[i - k];       // ตัดสมาชิกเก่าที่หลุดออกทางซ้าย
        if (window_sum > best) {
            best = window_sum;
        }
    }
    return best;
}

int main() {
    std::vector<int> nums = {2, 1, 5, 1, 3, 2};
    std::cout << "max sum of window size 3 = "
              << max_sum_fixed_window(nums, 3) << "\n"; // ควรได้ 9 (5+1+3)

    std::vector<int> nums2 = {4, 4, 4, 4};
    std::cout << "max sum of window size 2 = "
              << max_sum_fixed_window(nums2, 2) << "\n"; // ควรได้ 8

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 sliding_window.cpp -o sliding_window
./sliding_window
# max sum of window size 3 = 9
# max sum of window size 2 = 8
```

หน้าต่างขนาด**ยืดหยุ่น**จะซับซ้อนขึ้นเล็กน้อย เพราะต้องมี pointer สองตัว (`window_start`,
`window_end`) ที่ขยับเป็นอิสระจากกันตามเงื่อนไข เราจะเห็นแบบเต็มรูปแบบในโจทย์ **Longest
Substring Without Repeating Characters** ที่ 119.4

### Pattern 3: Fast & Slow Pointer (Floyd's Cycle Detection)

ใช้กับ **Linked List** เป็นหลัก เพื่อตรวจจับ cycle, หา node กึ่งกลาง, หรือหาจุดเริ่มต้นของ cycle
โดยใช้ pointer สองตัวที่วิ่งด้วยความเร็วต่างกัน (ตัวหนึ่งก้าวละ 1, อีกตัวก้าวละ 2) — ถ้ามี cycle
จริง สองตัวจะวิ่งมาชนกันเสมอ (เปรียบเหมือนนักวิ่งสองคนวิ่งวนสนามด้วยความเร็วต่างกัน สุดท้าย
คนที่เร็วกว่าจะวิ่งแซงมาชนคนที่ช้ากว่าจากด้านหลังจนได้ ถ้าสนามเป็นวงปิด)

```cpp
#include <iostream>

struct ListNode {
    int val;
    ListNode* next;
    explicit ListNode(int v) : val(v), next(nullptr) {}
};

// Fast & Slow Pointer (Floyd's Cycle Detection): ตรวจว่า linked list มี cycle หรือไม่
bool has_cycle(ListNode* head) {
    ListNode* slow = head;
    ListNode* fast = head;

    while (fast != nullptr && fast->next != nullptr) {
        slow = slow->next;         // เดินทีละ 1 ก้าว
        fast = fast->next->next;   // เดินทีละ 2 ก้าว
        if (slow == fast) {
            return true;           // ถ้าวิ่งชนกัน แปลว่ามี cycle แน่นอน
        }
    }
    return false;                  // fast ไปถึง nullptr ก่อน แปลว่าไม่มี cycle
}

int main() {
    // สร้าง list ที่ไม่มี cycle: 1 -> 2 -> 3 -> nullptr
    ListNode a(1), b(2), c(3);
    a.next = &b;
    b.next = &c;
    std::cout << std::boolalpha;
    std::cout << "no cycle -> " << has_cycle(&a) << "\n"; // false

    // สร้าง list ที่มี cycle: 1 -> 2 -> 3 -> 2 (วนกลับ)
    c.next = &b;
    std::cout << "with cycle -> " << has_cycle(&a) << "\n"; // true

    // ตัด cycle ออกก่อนจบโปรแกรม ป้องกันปัญหาตอน object ถูกทำลาย (ในที่นี้ไม่ได้ใช้ heap
    // จึงไม่มีปัญหาเรื่อง memory leak แต่เป็นนิสัยที่ดีเมื่อทำงานกับ cycle จริง)
    c.next = nullptr;

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 fast_slow.cpp -o fast_slow
./fast_slow
# no cycle -> false
# with cycle -> true
```

**ทำไมวิธีนี้ถึงดีกว่าการใช้ `unordered_set` เก็บ node ที่เคยเจอ?** วิธี hash set ก็ตรวจจับ
cycle ได้เหมือนกัน แต่ใช้ O(n) space เพิ่ม ในขณะที่ Fast & Slow Pointer ใช้ **O(1) space**
เท่านั้น — คำถามแนว "ทำได้ไหมโดยใช้ O(1) space" (follow-up question) เป็นสัญญาณที่บอกว่า
ผู้สัมภาษณ์กำลังคาดหวัง Pattern นี้อยู่

### Pattern 4: Binary Search on Answer

Binary Search ไม่ได้ใช้ได้แค่กับการค้นหาค่าในอาร์เรย์ที่เรียงแล้ว (ตามที่เรียนใน **Part 24**)
เท่านั้น แต่ยังใช้ได้กับโจทย์ที่ถามหา **"ค่าที่น้อยที่สุด/มากที่สุดที่ทำให้เงื่อนไขเป็นจริง"**
ได้ด้วย ตราบใดที่ฟังก์ชันตรวจสอบเงื่อนไข (feasibility check) มีคุณสมบัติ **monotonic**
(ยิ่งค่าคำตอบมากขึ้น เงื่อนไขยิ่งง่ายขึ้นเรื่อยๆ หรือยากขึ้นเรื่อยๆ ทิศทางเดียว ไม่สลับไปมา)

ตัวอย่างคลาสสิกคือโจทย์ **"Koko Eating Bananas"**: Koko มีกล้วยหลายกอง แต่ละชั่วโมงเลือก
ความเร็วคงที่ `k` (กองต่อชั่วโมง) กินได้กองเดียวต่อชั่วโมง ต้องกินให้หมดภายใน `h` ชั่วโมง
หาความเร็ว `k` ที่**น้อยที่สุด**ที่ทำให้กินทันเวลา

```cpp
#include <iostream>
#include <vector>
#include <algorithm>

// Binary Search on Answer: "Koko Eating Bananas"
// Koko มีกล้วย piles[i] กองในแต่ละกอง ต้องกินให้หมดภายใน h ชั่วโมง แต่ละชั่วโมงเลือก
// ความเร็วคงที่ k (กองต่อชั่วโมง) กินได้กองเดียวต่อชั่วโมง ถ้ากองเหลือน้อยกว่า k ก็ยังนับ
// เป็น 1 ชั่วโมงเช่นกัน หาความเร็ว k ที่น้อยที่สุดที่ทำให้กินหมดทันเวลา h
long long hours_needed(const std::vector<int>& piles, long long speed) {
    long long hours = 0;
    for (int pile : piles) {
        // เทียบเท่า ceil(pile / speed) โดยไม่ใช้ floating point
        hours += (pile + speed - 1) / speed;
    }
    return hours;
}

int min_eating_speed(const std::vector<int>& piles, int h) {
    long long lo = 1;
    long long hi = *std::max_element(piles.begin(), piles.end());

    // คุณสมบัติสำคัญของ Binary Search on Answer: ฟังก์ชัน hours_needed(speed) เป็น
    // monotonic (ยิ่ง speed มาก ยิ่งใช้เวลาน้อยลงหรือเท่าเดิม) ทำให้แบ่งครึ่งค้นหาได้
    while (lo < hi) {
        long long mid = lo + (hi - lo) / 2;
        if (hours_needed(piles, mid) <= h) {
            hi = mid;       // mid ใช้งานได้ ลองความเร็วที่ต่ำกว่านี้ดูอีก
        } else {
            lo = mid + 1;   // mid ช้าเกินไป ต้องเพิ่มความเร็ว
        }
    }
    return static_cast<int>(lo);
}

int main() {
    std::vector<int> piles1 = {3, 6, 7, 11};
    std::cout << "min speed (h=8) = " << min_eating_speed(piles1, 8) << "\n"; // 4

    std::vector<int> piles2 = {30, 11, 23, 4, 20};
    std::cout << "min speed (h=5) = " << min_eating_speed(piles2, 5) << "\n"; // 30
    std::cout << "min speed (h=6) = " << min_eating_speed(piles2, 6) << "\n"; // 23

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 binary_search_answer.cpp -o binary_search_answer
./binary_search_answer
# min speed (h=8) = 4
# min speed (h=5) = 30
# min speed (h=6) = 23
```

**สัญญาณที่บอกว่าควรใช้ Binary Search on Answer**: โจทย์มีคำว่า "หาค่าน้อยที่สุด/มากที่สุดที่
ทำให้..." และมีฟังก์ชันตรวจสอบที่คำนวณได้ตรงไปตรงมา (แม้จะช้าหน่อย เช่น O(n)) — เมื่อรวมกับ
Binary Search แล้วความซับซ้อนโดยรวมจะกลายเป็น O(n log(max_value)) ซึ่งเร็วกว่าการไล่ทีละค่า
(O(n × max_value)) มาก

### สรุปเปรียบเทียบ 4 Pattern

| Pattern | ใช้เมื่อไหร่ | Complexity ที่ได้ | ตัวอย่างโจทย์ที่พบบ่อย |
|---|---|---|---|
| Two Pointers | อาร์เรย์เรียงแล้ว/มี monotonic property, หาคู่/พาร์ทิชัน | O(n) จาก O(n²) | Two Sum II, Container With Most Water |
| Sliding Window | หา subarray/substring ต่อเนื่องที่ดีที่สุด | O(n) จาก O(n²) หรือแย่กว่า | Longest Substring, Max Sum Subarray |
| Fast & Slow Pointer | Linked List: cycle, จุดกึ่งกลาง | O(n) เวลา, O(1) space | Linked List Cycle, Middle of List |
| Binary Search on Answer | หาค่า min/max ที่ทำให้เงื่อนไข monotonic เป็นจริง | O(n log(range)) | Koko Eating Bananas, Ship Packages |

---

## 119.3 โจทย์ที่ 1-2: Two Sum และ Valid Parentheses (Step 947)

จากหัวข้อนี้เป็นต้นไป เราจะแก้โจทย์คลาสสิกที่ออกสอบบ่อยที่สุดในโลกจริง ทีละข้อ ตามขั้นตอน
มาตรฐานที่ควรใช้ในห้องสัมภาษณ์เสมอ: **(1) ทำความเข้าใจโจทย์ (2) คิดวิธี Brute Force ก่อน
(3) หาวิธีที่ดีกว่า (4) เขียนโค้ด (5) วิเคราะห์ complexity (6) ตรวจ edge case**

### โจทย์ 1: Two Sum (LeetCode #1)

**โจทย์**: ให้อาร์เรย์ `nums` (ไม่เรียงลำดับ) และค่า `target` หา index สองตัวที่บวกกันได้
`target` พอดี สมมติว่ามีคำตอบเพียงชุดเดียวเสมอ และห้ามใช้ตัวเดียวกันซ้ำสองครั้ง

**วิธี Brute Force**: วน loop ซ้อนกันตรวจทุกคู่ — O(n²) เวลา, O(1) space

**วิธีที่ดีกว่า**: ใช้ `unordered_map` เก็บ (ค่า → index) ที่เจอมาแล้ว ระหว่างวน loop เพียง
รอบเดียว ให้เช็คว่า `target - nums[i]` (ส่วนเติมเต็ม/complement) เคยเจอมาก่อนหรือยัง —
ลดเหลือ O(n) เวลา แลกกับ O(n) space

```cpp
#include <iostream>
#include <vector>
#include <unordered_map>
#include <stdexcept>

// LeetCode #1: Two Sum
// โจทย์: ให้อาร์เรย์ nums (ไม่เรียงลำดับ) และค่า target หา index สองตัวที่บวกกันได้ target
// สมมติว่ามีคำตอบเพียงชุดเดียวเสมอ และห้ามใช้ตัวเดียวกันซ้ำสองครั้ง
std::pair<int, int> two_sum(const std::vector<int>& nums, int target) {
    // เก็บ (ค่า -> index) ที่เจอมาแล้ว เพื่อเช็ค complement ได้ใน O(1) โดยเฉลี่ย
    std::unordered_map<int, int> seen;
    seen.reserve(nums.size() * 2);

    for (int i = 0; i < static_cast<int>(nums.size()); ++i) {
        int complement = target - nums[i];
        auto it = seen.find(complement);
        if (it != seen.end()) {
            return {it->second, i};
        }
        seen[nums[i]] = i;
    }
    throw std::invalid_argument("ไม่พบคู่ตัวเลขที่บวกกันได้ target");
}

int main() {
    std::vector<int> nums = {2, 7, 11, 15};
    auto [i, j] = two_sum(nums, 9);
    std::cout << "index (" << i << ", " << j << ") -> values ("
              << nums[i] << " + " << nums[j] << ") = 9\n";

    std::vector<int> nums2 = {3, 2, 4};
    auto [k, l] = two_sum(nums2, 6);
    std::cout << "index (" << k << ", " << l << ") -> values ("
              << nums2[k] << " + " << nums2[l] << ") = 6\n";

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 two_sum.cpp -o two_sum
./two_sum
# index (0, 1) -> values (2 + 7) = 9
# index (1, 2) -> values (2 + 4) = 6
```

**วิเคราะห์ complexity**: Time O(n) — วน loop ครั้งเดียว, ค้นหาใน `unordered_map` เฉลี่ย O(1)
ต่อครั้ง. Space O(n) — เก็บค่าไว้ใน map สูงสุด n ตัว

**เหตุใดจึงเลือกเช็ค complement "ก่อน" แล้วค่อย insert `nums[i]`**: เพื่อป้องกันการใช้ตัวเดียวกัน
ซ้ำสองครั้ง (เช่นถ้า `target = 4` และ `nums[i] = 2` ถ้า insert ก่อนแล้วค่อยเช็ค จะเจอ
`complement = 2` ที่เพิ่ง insert ไปเอง ซึ่งผิดเงื่อนไข)

**Edge case ที่ควรถามผู้สัมภาษณ์**: ถ้าไม่มีคำตอบเลยจะให้ทำอย่างไร (โยน exception, คืนค่า
`{-1,-1}`, หรือ `std::optional`)? มีตัวเลขซ้ำในอาร์เรย์ไหม? อาร์เรย์ว่างได้ไหม?

### โจทย์ 2: Valid Parentheses (LeetCode #20)

**โจทย์**: ให้สตริงที่มีเฉพาะ `()[]{}` ตรวจว่าวงเล็บทุกคู่ถูกเปิด-ปิดอย่างถูกต้องและสมดุลหรือไม่
(ทบทวนแนวคิด **Stack** จาก **Part 20** โดยตรง — นี่คือ Application ของ Stack ที่ใช้จริงในทุก
compiler/parser ของโลก)

**แนวคิด**: ทุกครั้งที่เจอวงเล็บเปิด ให้ `push` ลง stack ทุกครั้งที่เจอวงเล็บปิด ให้ตรวจว่า
ตัวบนสุดของ stack จับคู่กันได้หรือไม่ ถ้าใช่ให้ `pop` ออก ถ้าไม่ใช่หรือ stack ว่างอยู่แล้วให้
คืนค่า `false` ทันที เมื่ออ่านจบสตริง ถ้า stack ว่างพอดีแปลว่าสมดุลจริง

```cpp
#include <iostream>
#include <string>
#include <stack>
#include <unordered_map>

// LeetCode #20: Valid Parentheses (ทบทวนแนวคิด Stack จาก Part 20)
// โจทย์: ให้สตริงที่มีเฉพาะ ()[]{} ตรวจว่าวงเล็บทุกคู่ถูกเปิด-ปิดอย่างถูกต้องและสมดุลหรือไม่
bool is_valid_parentheses(const std::string& s) {
    std::stack<char> open_stack;
    // map จาก "วงเล็บปิด" -> "วงเล็บเปิดที่ควรจับคู่กัน"
    const std::unordered_map<char, char> match = {
        {')', '('}, {']', '['}, {'}', '{'}
    };

    for (char c : s) {
        if (c == '(' || c == '[' || c == '{') {
            open_stack.push(c);
        } else {
            // เจอวงเล็บปิด: stack ต้องไม่ว่าง และตัวบนสุดต้องจับคู่กันได้
            if (open_stack.empty() || open_stack.top() != match.at(c)) {
                return false;
            }
            open_stack.pop();
        }
    }
    // สมดุลจริง ก็ต่อเมื่อไม่มีวงเล็บเปิดค้างอยู่เลยหลังอ่านจบสตริง
    return open_stack.empty();
}

int main() {
    std::cout << std::boolalpha;
    std::cout << "\"()[]{}\"  -> " << is_valid_parentheses("()[]{}") << "\n";   // true
    std::cout << "\"(]\"      -> " << is_valid_parentheses("(]") << "\n";       // false
    std::cout << "\"([{}])\"  -> " << is_valid_parentheses("([{}])") << "\n";   // true
    std::cout << "\"(((\"     -> " << is_valid_parentheses("(((") << "\n";      // false
    std::cout << "\"\"        -> " << is_valid_parentheses("") << "\n";        // true (edge case)

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 valid_parentheses.cpp -o valid_parentheses
./valid_parentheses
# "()[]{}"  -> true
# "(]"      -> false
# "([{}])"  -> true
# "((("     -> false
# ""        -> true
```

**วิเคราะห์ complexity**: Time O(n) — อ่านสตริงครั้งเดียว, Space O(n) ในกรณีเลวร้ายที่สุด
(สตริงที่มีแต่วงเล็บเปิดล้วนๆ เช่น `"((((("`)

**เหตุผลที่ต้องเช็ค `open_stack.empty()` ก่อนเสมอ**: ถ้าเจอวงเล็บปิดตัวแรกสุดของสตริงโดยที่
ยังไม่มีวงเล็บเปิดเลย (เช่น `")("`) การเรียก `.top()` บน stack ที่ว่างเปล่าคือ **Undefined
Behavior** ทันที — นี่เป็นจุดที่กรรมการสัมภาษณ์มักจับตาดูเป็นพิเศษว่าผู้สมัครนึกถึงหรือไม่

**Edge case ที่ควรถามผู้สัมภาษณ์**: สตริงว่างถือว่า valid หรือไม่ (ตามธรรมเนียม LeetCode
ถือว่า valid), สตริงมีตัวอักษรอื่นปนมาไหม (โจทย์จริงมักการันตีว่ามีแค่วงเล็บ 6 ตัวเท่านั้น)

---

## 119.4 โจทย์ที่ 3-4: Longest Substring Without Repeating Characters และ Merge Intervals (Step 948)

### โจทย์ 3: Longest Substring Without Repeating Characters (LeetCode #3)

**โจทย์**: หาความยาวของ substring (ต่อเนื่องกัน) ที่ยาวที่สุดที่ไม่มีตัวอักษรซ้ำกันเลย

**วิธี Brute Force**: ตรวจทุก substring ที่เป็นไปได้ — O(n³) หรือ O(n²) ถ้า optimize เล็กน้อย

**วิธีที่ดีกว่า**: **Sliding Window แบบ variable-size** — ขยาย `right` ไปเรื่อยๆ ถ้าเจอตัวซ้ำ
ที่**อยู่ภายในหน้าต่างปัจจุบัน** ให้เลื่อน `window_start` ไปอยู่หลังตำแหน่งที่ซ้ำนั้นทันที
(ไม่ใช่เลื่อนทีละ 1 ซึ่งจะช้ากว่า)

```cpp
#include <iostream>
#include <string>
#include <unordered_map>
#include <algorithm>

// LeetCode #3: Longest Substring Without Repeating Characters
// โจทย์: หาความยาวของ substring ที่ยาวที่สุดที่ไม่มีตัวอักษรซ้ำกันเลย
int length_of_longest_substring(const std::string& s) {
    std::unordered_map<char, int> last_index; // ตัวอักษร -> index ล่าสุดที่เจอ
    int best = 0;
    int window_start = 0; // ขอบซ้ายของ sliding window (variable-size window)

    for (int right = 0; right < static_cast<int>(s.size()); ++right) {
        char c = s[right];
        auto it = last_index.find(c);
        // ถ้าเจอตัวซ้ำ "ภายในหน้าต่างปัจจุบัน" ต้องเลื่อนขอบซ้ายมาอยู่หลังตัวซ้ำนั้นทันที
        if (it != last_index.end() && it->second >= window_start) {
            window_start = it->second + 1;
        }
        last_index[c] = right;
        best = std::max(best, right - window_start + 1);
    }
    return best;
}

int main() {
    std::cout << "\"abcabcbb\" -> " << length_of_longest_substring("abcabcbb") << "\n"; // 3 (abc)
    std::cout << "\"bbbbb\"    -> " << length_of_longest_substring("bbbbb") << "\n";    // 1 (b)
    std::cout << "\"pwwkew\"   -> " << length_of_longest_substring("pwwkew") << "\n";   // 3 (wke)
    std::cout << "\"\"         -> " << length_of_longest_substring("") << "\n";         // 0 (edge case)
    std::cout << "\" \"        -> " << length_of_longest_substring(" ") << "\n";        // 1 (edge case)

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 longest_substring.cpp -o longest_substring
./longest_substring
# "abcabcbb" -> 3
# "bbbbb"    -> 1
# "pwwkew"   -> 3
# ""         -> 0
# " "        -> 1
```

**วิเคราะห์ complexity**: Time O(n) — pointer `right` วิ่งผ่านสตริงแค่รอบเดียว, Space
O(min(n, alphabet_size)) — map เก็บได้มากสุดเท่าจำนวนตัวอักษรที่ต่างกัน

**จุดที่ผู้สมัครส่วนใหญ่พลาด**: การเช็ค `it->second >= window_start` เป็นจุดสำคัญที่สุด — ถ้า
ไม่เช็คเงื่อนไขนี้ (เช็คแค่ "เคยเจอตัวนี้มาก่อนไหม" เฉยๆ) จะทำให้ `window_start` ถอยหลังได้
ในบางกรณี (เช่น input `"abba"`) ซึ่งทำให้คำตอบผิด — นี่คือ edge case แบบคลาสสิกที่ต้องยกตัวอย่าง
ทดสอบด้วยตัวเองเสมอก่อนบอกว่าโค้ดเสร็จแล้ว

### โจทย์ 4: Merge Intervals (LeetCode #56)

**โจทย์**: ให้ช่วง `[start, end]` หลายช่วง (ไม่เรียงลำดับ) ให้รวมช่วงที่ทับซ้อนกันเข้าด้วยกัน
เป็นช่วงเดียว

**แนวคิด**: **เรียงลำดับก่อนเสมอ** ตามจุดเริ่มต้น (`start`) จากนั้นวน loop เพียงรอบเดียว —
ถ้าช่วงปัจจุบันเริ่มก่อนหรือเท่ากับจุดสิ้นสุดของช่วงล่าสุดในผลลัพธ์ แปลว่าทับซ้อนกัน ให้ขยาย
ขอบเขต มิฉะนั้นให้เพิ่มเป็นช่วงใหม่

```cpp
#include <iostream>
#include <vector>
#include <algorithm>

// LeetCode #56: Merge Intervals
// โจทย์: ให้ช่วง [start, end] หลายช่วง (ไม่เรียงลำดับ) ให้รวมช่วงที่ทับซ้อนกันเข้าด้วยกัน
using Interval = std::pair<int, int>;

std::vector<Interval> merge_intervals(std::vector<Interval> intervals) {
    if (intervals.empty()) {
        return {};
    }

    // ขั้นตอนที่ 1: เรียงตามจุดเริ่มต้นก่อนเสมอ มิฉะนั้นตรรกะการรวมด้านล่างจะผิด
    std::sort(intervals.begin(), intervals.end(),
              [](const Interval& a, const Interval& b) { return a.first < b.first; });

    std::vector<Interval> result;
    result.push_back(intervals[0]);

    for (std::size_t i = 1; i < intervals.size(); ++i) {
        Interval& last = result.back();
        const Interval& current = intervals[i];

        if (current.first <= last.second) {
            // ทับซ้อนกัน (หรือแตะกันพอดี) -> ขยายขอบเขตของช่วงล่าสุด
            last.second = std::max(last.second, current.second);
        } else {
            // ไม่ทับซ้อน -> เป็นช่วงใหม่
            result.push_back(current);
        }
    }
    return result;
}

void print_intervals(const std::vector<Interval>& intervals) {
    for (const auto& [start, end] : intervals) {
        std::cout << "[" << start << "," << end << "] ";
    }
    std::cout << "\n";
}

int main() {
    std::vector<Interval> a = {{1, 3}, {2, 6}, {8, 10}, {15, 18}};
    print_intervals(merge_intervals(a)); // [1,6] [8,10] [15,18]

    std::vector<Interval> b = {{1, 4}, {4, 5}};
    print_intervals(merge_intervals(b)); // [1,5]  (แตะกันพอดีที่ 4 ก็ต้องรวม)

    std::vector<Interval> c = {}; // edge case: ไม่มีช่วงเลย
    print_intervals(merge_intervals(c)); // (บรรทัดว่าง)

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 merge_intervals.cpp -o merge_intervals
./merge_intervals
# [1,6] [8,10] [15,18]
# [1,5]
#
```

**วิเคราะห์ complexity**: Time O(n log n) — คอขวดอยู่ที่การ sort ไม่ใช่การวน loop รวมช่วง
(ซึ่งเป็น O(n)), Space O(n) สำหรับผลลัพธ์ (หรือ O(log n) ถึง O(n) เพิ่มเติมสำหรับ `std::sort`
เอง ขึ้นกับ implementation)

**Edge case ที่ควรถามผู้สัมภาษณ์**: ช่วงที่ "แตะกันพอดี" เช่น `[1,4]` กับ `[4,5]` ถือว่าทับซ้อน
ต้องรวมกันหรือไม่ (ในตัวอย่างข้างต้นเราถือว่าต้องรวม — สังเกตเงื่อนไข `<=` ไม่ใช่ `<`) และ
รายการช่วงว่างเปล่าควรคืนอะไร

---

## 119.5 โจทย์ที่ 5-6: Reverse Linked List และ Kth Largest Element (Step 949)

### โจทย์ 5: Reverse Linked List (LeetCode #206)

**โจทย์**: กลับทิศทางของ Singly Linked List ทั้งเส้น (ทบทวนโครงสร้างข้อมูลจาก **Part 19**
โดยตรง — นี่คือหนึ่งในโจทย์ Linked List ที่ถูกถามบ่อยที่สุดในโลก เพราะทดสอบความเข้าใจเรื่อง
Pointer ได้ตรงจุดที่สุด)

มีสองวิธีหลักที่ควรรู้ทั้งคู่ เพราะผู้สัมภาษณ์มักถาม follow-up ว่า "เขียนอีกแบบได้ไหม":

```cpp
#include <iostream>

// LeetCode #206: Reverse Linked List (ทบทวนโครงสร้าง Linked List จาก Part 19)
struct ListNode {
    int val;
    ListNode* next;
    explicit ListNode(int v, ListNode* n = nullptr) : val(v), next(n) {}
};

// วิธีที่ 1: Iterative — ใช้ pointer สามตัว (prev, curr, next_node) วิ่งไปพร้อมกัน
ListNode* reverse_list_iterative(ListNode* head) {
    ListNode* prev = nullptr;
    ListNode* curr = head;

    while (curr != nullptr) {
        ListNode* next_node = curr->next; // เก็บ node ถัดไปไว้ก่อนจะตัดสาย
        curr->next = prev;                // กลับทิศทางลูกศร
        prev = curr;                      // เลื่อน prev มาที่ curr
        curr = next_node;                 // เลื่อน curr ไปยัง node ที่เก็บไว้
    }
    return prev; // prev คือ head ใหม่ของ list ที่กลับด้านแล้ว
}

// วิธีที่ 2: Recursive — สั้นกว่าแต่ใช้ O(n) space บน call stack
ListNode* reverse_list_recursive(ListNode* head) {
    if (head == nullptr || head->next == nullptr) {
        return head; // base case: list ว่างหรือมีสมาชิกเดียว
    }
    ListNode* new_head = reverse_list_recursive(head->next);
    head->next->next = head; // ให้ node ถัดไปชี้กลับมาที่ตัวเอง
    head->next = nullptr;    // ตัดสายเดิมทิ้งเพื่อไม่ให้เกิด cycle
    return new_head;
}

void print_list(ListNode* head) {
    while (head != nullptr) {
        std::cout << head->val;
        if (head->next != nullptr) {
            std::cout << " -> ";
        }
        head = head->next;
    }
    std::cout << "\n";
}

void free_list(ListNode* head) {
    while (head != nullptr) {
        ListNode* next_node = head->next;
        delete head;
        head = next_node;
    }
}

int main() {
    // 1 -> 2 -> 3 -> 4 -> 5 -> nullptr
    ListNode* head1 = new ListNode(1, new ListNode(2, new ListNode(3, new ListNode(4, new ListNode(5)))));
    std::cout << "before: ";
    print_list(head1);
    ListNode* reversed1 = reverse_list_iterative(head1);
    std::cout << "after (iterative): ";
    print_list(reversed1);
    free_list(reversed1);

    ListNode* head2 = new ListNode(10, new ListNode(20, new ListNode(30)));
    ListNode* reversed2 = reverse_list_recursive(head2);
    std::cout << "after (recursive): ";
    print_list(reversed2);
    free_list(reversed2);

    // edge case: list ว่าง
    ListNode* empty_head = nullptr;
    ListNode* still_empty = reverse_list_iterative(empty_head);
    std::cout << "empty list result is nullptr: " << std::boolalpha << (still_empty == nullptr) << "\n";

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 reverse_linked_list.cpp -o reverse_linked_list
./reverse_linked_list
# before: 1 -> 2 -> 3 -> 4 -> 5
# after (iterative): 5 -> 4 -> 3 -> 2 -> 1
# after (recursive): 30 -> 20 -> 10
# empty list result is nullptr: true
```

**วิเคราะห์ complexity**:

| วิธี | Time | Space | ข้อดี | ข้อเสีย |
|---|---|---|---|---|
| Iterative | O(n) | O(1) | ไม่มีความเสี่ยง Stack Overflow, เร็วกว่าในทางปฏิบัติ | โค้ดต้องไล่ pointer สามตัวให้ถูกลำดับ |
| Recursive | O(n) | O(n) (call stack) | โค้ดสั้น อ่านง่ายกว่ามาก | List ยาวมากอาจ Stack Overflow ได้จริง |

> **หมายเหตุสำคัญในห้องสัมภาษณ์จริง**: โครงสร้าง `struct ListNode` ที่ใช้เก็บโหนดในโจทย์
> LeetCode จริง (บนเว็บไซต์) มักถูก**กำหนดมาให้แล้ว** ผู้สมัครไม่ต้องออกแบบเอง และไม่ต้อง
> กังวลเรื่อง memory management ของ node เหล่านี้มากนัก (ระบบ LeetCode จะจัดการให้) แต่ถ้า
> สัมภาษณ์แบบ Whiteboard/Google Doc ที่ต้องเขียนโค้ดที่คอมไพล์ได้จริงแบบในหลักสูตรนี้ ควรใส่ใจ
> เรื่องการ `delete` node ให้ครบด้วยเสมอ ตามที่ `free_list()` แสดงไว้ข้างบน

### โจทย์ 6: Kth Largest Element in an Array (LeetCode #215)

**โจทย์**: หาค่าที่ใหญ่เป็นอันดับที่ `k` ในอาร์เรย์ที่ไม่เรียงลำดับ

**วิธี Brute Force**: sort ทั้งอาร์เรย์แล้วอ่านตำแหน่งที่ `n - k` — O(n log n)

**วิธีที่ดีกว่าเมื่อ `k` เล็ก**: ใช้ **Min-Heap ขนาด k** (`std::priority_queue` ที่เรียนจะ
เชื่อมโยงกับแนวคิด Tree ใน **Part 21**) — เก็บเฉพาะ k ตัวที่มากที่สุดเท่าที่เจอมา ยอดฮีป
(top ของ min-heap) คือคำตอบ

```cpp
#include <iostream>
#include <vector>
#include <queue>
#include <algorithm>
#include <stdexcept>

// LeetCode #215: Kth Largest Element in an Array
// วิธี: ใช้ Min-Heap ขนาด k — เก็บเฉพาะ k ตัวที่มากที่สุดเท่าที่เจอมา ยอดฮีปคือคำตอบ
// Complexity: O(n log k) เวลา, O(k) พื้นที่ — ดีกว่า sort ทั้งอาร์เรย์ (O(n log n)) เมื่อ k เล็ก
int find_kth_largest(const std::vector<int>& nums, int k) {
    if (k <= 0 || static_cast<std::size_t>(k) > nums.size()) {
        throw std::invalid_argument("k ต้องอยู่ในช่วง [1, nums.size()]");
    }

    std::priority_queue<int, std::vector<int>, std::greater<int>> min_heap;

    for (int num : nums) {
        min_heap.push(num);
        if (min_heap.size() > static_cast<std::size_t>(k)) {
            min_heap.pop(); // ตัดตัวที่เล็กที่สุดออก เหลือไว้แค่ k ตัวที่ใหญ่สุด
        }
    }
    return min_heap.top(); // ยอดของ min-heap ขนาด k คือตัวที่ใหญ่เป็นอันดับ k
}

int main() {
    std::vector<int> nums1 = {3, 2, 1, 5, 6, 4};
    std::cout << "k=2 in {3,2,1,5,6,4} -> " << find_kth_largest(nums1, 2) << "\n"; // 5

    std::vector<int> nums2 = {3, 2, 3, 1, 2, 4, 5, 5, 6};
    std::cout << "k=4 in {3,2,3,1,2,4,5,5,6} -> " << find_kth_largest(nums2, 4) << "\n"; // 4

    // เทียบกับวิธี sort ตรงๆ เพื่อยืนยันว่าคำตอบตรงกัน
    std::vector<int> nums3 = nums1;
    std::sort(nums3.rbegin(), nums3.rend());
    std::cout << "sort-based check k=2 -> " << nums3[1] << "\n"; // ต้องได้ 5 เท่ากัน

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 kth_largest.cpp -o kth_largest
./kth_largest
# k=2 in {3,2,1,5,6,4} -> 5
# k=4 in {3,2,3,1,2,4,5,5,6} -> 4
# sort-based check k=2 -> 5
```

**วิเคราะห์ complexity**: Time O(n log k) — แต่ละครั้งที่ push/pop บน heap ขนาด k ใช้เวลา
O(log k), ทำ n ครั้ง. Space O(k) สำหรับ heap

**ทางเลือกที่สาม (Follow-up ที่ผู้สัมภาษณ์ระดับ Senior มักถาม)**: อัลกอริทึม **Quickselect**
(ดัดแปลงจาก Quick Sort ที่เรียนใน **Part 23**) สามารถหาคำตอบได้ใน **O(n) โดยเฉลี่ย** (แม้
worst case จะเป็น O(n²) เหมือน Quick Sort) โดยไม่ต้องใช้ heap เลย เทคนิคคือ partition
อาร์เรย์แบบเดียวกับ Quick Sort แต่ recurse ลงไปแค่ฝั่งเดียวที่มีคำตอบอยู่ ไม่ใช่ทั้งสองฝั่ง

---

## 119.6 โจทย์ที่ 7: Number of Islands ด้วย BFS/DFS (Step 950)

**โจทย์ (LeetCode #200)**: ให้ grid 2 มิติที่ประกอบด้วย `'1'` (แผ่นดิน) และ `'0'` (น้ำ)
นับจำนวนเกาะทั้งหมด โดยเกาะคือกลุ่มแผ่นดินที่เชื่อมต่อกันในแนวขึ้น-ลง-ซ้าย-ขวาเท่านั้น
(ไม่นับแนวทแยงมุม) — โจทย์นี้เป็นตัวแทนของโจทย์กลุ่ม **Graph Traversal บน Grid** ที่ออกสอบ
บ่อยมาก และประยุกต์ใช้ทั้ง BFS และ DFS ที่ทบทวนแนวคิดมาจาก Queue (**Part 20**) และ Tree
Traversal (**Part 21**)

**แนวคิด**: วน loop ตรวจทุกช่องใน grid เมื่อเจอ `'1'` ที่ยังไม่เคยเยี่ยมชม ให้เพิ่มตัวนับเกาะ
ขึ้น 1 แล้ว "จม" แผ่นดินทั้งหมดที่เชื่อมกับจุดนั้นให้กลายเป็น `'0'` (ทำเครื่องหมายว่าเยี่ยมชม
แล้ว) ด้วย BFS หรือ DFS เพื่อไม่ให้นับซ้ำ

```cpp
#include <iostream>
#include <vector>
#include <string>
#include <queue>

// LeetCode #200: Number of Islands
// โจทย์: grid เป็น '1' (แผ่นดิน) และ '0' (น้ำ) นับจำนวนเกาะ (กลุ่มแผ่นดินที่เชื่อมกันแนวขึ้น
// ลง ซ้าย ขวา เท่านั้น ไม่นับแนวทแยง)

// วิธีที่ 1: DFS แบบ recursive
void dfs_sink(std::vector<std::string>& grid, int r, int c) {
    int rows = static_cast<int>(grid.size());
    int cols = static_cast<int>(grid[0].size());

    if (r < 0 || r >= rows || c < 0 || c >= cols || grid[r][c] != '1') {
        return; // ออกนอก grid หรือไม่ใช่แผ่นดินแล้ว ให้หยุด
    }
    grid[r][c] = '0'; // "จม" แผ่นดินที่มาเยือนแล้ว ป้องกันการนับซ้ำ

    dfs_sink(grid, r + 1, c);
    dfs_sink(grid, r - 1, c);
    dfs_sink(grid, r, c + 1);
    dfs_sink(grid, r, c - 1);
}

int num_islands_dfs(std::vector<std::string> grid) {
    if (grid.empty() || grid[0].empty()) {
        return 0;
    }
    int count = 0;
    for (int r = 0; r < static_cast<int>(grid.size()); ++r) {
        for (int c = 0; c < static_cast<int>(grid[0].size()); ++c) {
            if (grid[r][c] == '1') {
                ++count;
                dfs_sink(grid, r, c);
            }
        }
    }
    return count;
}

// วิธีที่ 2: BFS แบบ iterative — หลีกเลี่ยงปัญหา stack overflow บน grid ที่ใหญ่มาก
int num_islands_bfs(std::vector<std::string> grid) {
    if (grid.empty() || grid[0].empty()) {
        return 0;
    }
    int rows = static_cast<int>(grid.size());
    int cols = static_cast<int>(grid[0].size());
    int count = 0;

    const int dr[4] = {1, -1, 0, 0};
    const int dc[4] = {0, 0, 1, -1};

    for (int sr = 0; sr < rows; ++sr) {
        for (int sc = 0; sc < cols; ++sc) {
            if (grid[sr][sc] != '1') {
                continue;
            }
            ++count;
            grid[sr][sc] = '0';
            std::queue<std::pair<int, int>> q;
            q.push({sr, sc});

            while (!q.empty()) {
                auto [r, c] = q.front();
                q.pop();
                for (int d = 0; d < 4; ++d) {
                    int nr = r + dr[d];
                    int nc = c + dc[d];
                    if (nr >= 0 && nr < rows && nc >= 0 && nc < cols && grid[nr][nc] == '1') {
                        grid[nr][nc] = '0';
                        q.push({nr, nc});
                    }
                }
            }
        }
    }
    return count;
}

int main() {
    std::vector<std::string> grid1 = {
        "11110",
        "11010",
        "11000",
        "00000"
    };
    std::cout << "grid1 DFS -> " << num_islands_dfs(grid1) << "\n"; // 1
    std::cout << "grid1 BFS -> " << num_islands_bfs(grid1) << "\n"; // 1

    std::vector<std::string> grid2 = {
        "11000",
        "11000",
        "00100",
        "00011"
    };
    std::cout << "grid2 DFS -> " << num_islands_dfs(grid2) << "\n"; // 3
    std::cout << "grid2 BFS -> " << num_islands_bfs(grid2) << "\n"; // 3

    std::vector<std::string> grid3 = {}; // edge case: grid ว่าง
    std::cout << "grid3 (empty) -> " << num_islands_dfs(grid3) << "\n"; // 0

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 number_of_islands.cpp -o number_of_islands
./number_of_islands
# grid1 DFS -> 1
# grid1 BFS -> 1
# grid2 DFS -> 3
# grid2 BFS -> 3
# grid3 (empty) -> 0
```

**วิเคราะห์ complexity**: ทั้งสองวิธี Time O(rows × cols) — เยี่ยมชมทุกช่องอย่างมากแค่ครั้ง
เดียว, Space O(rows × cols) worst case (สำหรับ recursion stack ของ DFS หรือ queue ของ BFS
เมื่อทั้ง grid เป็นแผ่นดินผืนเดียว)

**DFS หรือ BFS แบบไหนควรใช้ในห้องสัมภาษณ์**: ทั้งสองแบบให้คำตอบถูกต้องเหมือนกันและมี
complexity เท่ากัน แต่ **DFS แบบ recursive มีความเสี่ยง Stack Overflow** ถ้า grid ใหญ่มาก
(เช่น 1000×1000 ที่เป็นแผ่นดินทั้งผืน จะ recurse ลึกถึง 1,000,000 ชั้น) ในขณะที่ **BFS แบบ
iterative ใช้ heap memory ผ่าน `std::queue` จึงไม่มีข้อจำกัดนี้** — เป็นเหตุผลเชิงปฏิบัติที่
ควรพูดถึงเมื่อผู้สัมภาษณ์ถามว่า "เลือกวิธีไหนและทำไม" (เทียบเคียงกับที่เคยพูดถึงข้อจำกัดของ
Recursion ใน **Part 25**)

---

## 119.7 เทคนิคการสื่อสารระหว่างแก้โจทย์ในห้องสัมภาษณ์จริง (Step 951)

ผู้สมัครจำนวนมากที่ **เขียนโค้ดเก่งมาก** กลับสอบตกรอบ Coding เพราะเงียบเกินไป หรือกระโดดเขียน
โค้ดโดยไม่อธิบายอะไรเลย ในความเป็นจริง **กระบวนการคิด (process)** สำคัญพอๆ กับคำตอบสุดท้าย
เพราะผู้สัมภาษณ์กำลังจำลองสถานการณ์ทำงานจริงที่ต้องสื่อสารกับเพื่อนร่วมทีมตลอดเวลา

### กรอบคิด UMPIRE Method

กรอบคิดที่ใช้กันแพร่หลายในการเตรียมสัมภาษณ์คือ **UMPIRE**: Understand, Match, Plan,
Implement, Review, Evaluate ลองไล่ตามกรอบนี้กับโจทย์ Two Sum เป็นตัวอย่าง:

| ขั้นตอน | สิ่งที่ต้องทำ | ตัวอย่างคำพูด (แปลเป็นไทย) |
|---|---|---|
| **U**nderstand | อ่านโจทย์ซ้ำด้วยคำพูดตัวเอง ถามคำถามชี้แจง | "ให้ผมสรุปโจทย์ก่อนนะครับ: หา index สองตัวที่บวกกันได้ target ใช่ไหมครับ อาร์เรย์นี้อนุญาตให้มีเลขซ้ำไหมครับ" |
| **M**atch | จับคู่โจทย์กับ Pattern ที่รู้จัก | "โจทย์นี้ดูคล้าย pattern hash map สำหรับหาคู่ที่ผลรวมตรงเงื่อนไข" |
| **P**lan | อธิบายแนวทางก่อนลงมือเขียน (พูดออกเสียง) | "ผมจะใช้ unordered_map เก็บค่าที่เจอมาแล้วกับ index ของมัน วน loop ครั้งเดียวแล้วเช็ค complement ครับ" |
| **I**mplement | เขียนโค้ดจริง พร้อมพูดสิ่งที่กำลังพิมพ์เป็นระยะ | "ตอนนี้ผมกำลังประกาศ unordered_map<int,int> ชื่อ seen ครับ" |
| **R**eview | ไล่ trace โค้ดด้วย test case ตัวอย่างก่อนบอกว่าเสร็จ | "ลองไล่ด้วย [2,7,11,15] target=9 ดูนะครับ..." |
| **E**valuate | สรุป complexity และถามว่ามีวิธีที่ดีกว่านี้ไหม | "Time complexity คือ O(n), Space คือ O(n) ครับ มี trade-off ตรงที่ใช้ memory เพิ่ม แลกกับความเร็ว" |

### หลักการพูดคุยที่ควรฝึกให้เป็นนิสัย

1. **คิดออกเสียงเสมอ (Think Aloud)**: อย่าเงียบเกิน 30 วินาที ต่อให้กำลังคิดอยู่เฉยๆ ก็ควร
   พูดว่ากำลังคิดอะไรอยู่ เช่น "ผมกำลังคิดว่า approach แบบ two-pointer จะเวิร์กกับ input นี้
   ไหม เพราะอาร์เรย์ยังไม่เรียงลำดับ..."
2. **ถามคำถามชี้แจง (Clarifying Questions) ก่อนเริ่มเขียนโค้ดเสมอ** โดยเฉพาะเรื่อง:
   - ขนาดของ input (n เล็กแค่ไหน ใหญ่แค่ไหน — มีผลต่อการเลือก complexity ที่ยอมรับได้)
   - ข้อมูลซ้ำได้ไหม (duplicates)
   - ค่าติดลบหรือศูนย์เป็นไปได้ไหม
   - input ว่างเปล่าได้ไหม ต้องจัดการอย่างไร
   - รับประกันว่ามีคำตอบเสมอไหม หรือต้องจัดการกรณีไม่มีคำตอบด้วย
3. **เริ่มจาก Brute Force เสมอถ้านึกวิธีที่ดีที่สุดไม่ออกทันที** การพูดว่า "ผมนึกวิธี O(n²)
   ออกก่อน แล้วจะลองหาวิธีที่เร็วกว่านี้" ดีกว่าเงียบไปเรื่อยๆ เพราะแสดงให้เห็นกระบวนการคิด
   และอย่างน้อยก็ได้คะแนนบางส่วนแม้เวลาจะหมดก่อนเขียนวิธีที่ดีที่สุดเสร็จ
4. **ถ้าติดขัดจริงๆ ให้ขอ hint อย่างมืออาชีพ** เช่น "ผมกำลังคิดสอง approach อยู่ คือ A และ B
   คุณพอจะให้ hint ได้ไหมครับว่าผมกำลังไปถูกทางหรือเปล่า" — การขอความช่วยเหลืออย่างมีเหตุผล
   ดีกว่าการนั่งเงียบหรือพยายามปิดบังว่าติดขัด เพราะในงานจริงการรู้จักขอความช่วยเหลือเมื่อจำเป็น
   ก็เป็นทักษะสำคัญเช่นกัน
5. **สรุป Complexity ทุกครั้งโดยไม่ต้องรอให้ถาม** เป็นสัญญาณที่บอกผู้สัมภาษณ์ว่าผู้สมัครคิด
   เรื่องประสิทธิภาพเป็นธรรมชาติ ไม่ใช่แค่ทำให้โค้ด "รันผ่าน" เท่านั้น

### ตัวอย่างบทสนทนาสั้นๆ ระหว่างแก้โจทย์ Merge Intervals

> **ผู้สมัคร**: "ผมสรุปโจทย์ก่อนนะครับ: ให้ list ของช่วง [start, end] ที่อาจไม่เรียงลำดับ
> ให้รวมช่วงที่ทับซ้อนกัน แล้วคืน list ของช่วงที่รวมแล้วใช่ไหมครับ? แล้วถ้าสองช่วงแตะกันพอดี
> เช่น [1,4] กับ [4,5] ถือว่าต้องรวมกันไหมครับ?"
>
> **ผู้สัมภาษณ์**: "ใช่ครับ ต้องรวมกัน"
>
> **ผู้สมัคร**: "เข้าใจแล้วครับ วิธีที่ผมคิดคือต้อง sort ช่วงตาม start ก่อน เพราะถ้าไม่ sort
> การเช็คว่าทับซ้อนกันหรือไม่จะซับซ้อนกว่ามาก พอ sort แล้วผมจะวน loop รอบเดียว เทียบช่วง
> ปัจจุบันกับช่วงล่าสุดที่อยู่ใน result — ถ้า start ของช่วงปัจจุบันน้อยกว่าหรือเท่ากับ end
> ของช่วงล่าสุด ก็ขยาย end ของช่วงล่าสุด ไม่งั้นก็ push เป็นช่วงใหม่ครับ Time complexity
> จะเป็น O(n log n) จากการ sort เป็นหลัก ผมขอเริ่มเขียนโค้ดเลยนะครับ"

สังเกตว่าผู้สมัครอธิบายแนวทางและ complexity **ก่อน** ลงมือเขียนโค้ดจริง ซึ่งเป็นสิ่งที่ผู้
สัมภาษณ์ทุกคนอยากได้ยิน

---

## 119.8 เตรียมตัว System/Low-Level Design Round สำหรับตำแหน่งสาย C++ (Step 952)

ตามที่กล่าวไว้ใน 119.1 ตำแหน่งสาย C++ จำนวนมาก (โดยเฉพาะ Game, Embedded, Infrastructure,
Trading System) มักเน้น **Low-Level Design (LLD)** มากกว่า High-Level Design (HLD) แบบระบบ
กระจายที่ถามในสาย Web/Backend ทั่วไป (ซึ่งจะฝึกแบบเต็มรูปแบบใน Module K — Capstone Projects)

### LLD คืออะไร และต่างจาก HLD อย่างไร

| หัวข้อ | Low-Level Design (LLD) | High-Level Design (HLD) |
|---|---|---|
| ขอบเขต | ออกแบบ Class/Interface/Object ภายในระบบเดียว | ออกแบบสถาปัตยกรรมระบบกระจายทั้งหมด |
| ตัวอย่างโจทย์ | ออกแบบ Parking Lot, Elevator, LRU Cache, Rate Limiter | ออกแบบ Twitter, URL Shortener, Chat System ระดับล้านผู้ใช้ |
| สิ่งที่ถูกประเมิน | OOP Design, SOLID Principles, Thread-safety, Memory Management | Scalability, Database Sharding, Load Balancing, Caching Layer |
| พบบ่อยในตำแหน่งสาย | C++ System/Game/Embedded/Trading | Backend/Full-stack ทั่วไป |
| เชื่อมโยงกับหลักสูตรนี้ | Module D (OOP), Module E (Template/STL), Part 66-68 (Allocator/RAII) | Module I (Web Dev), Module K (Capstone) |

### ตัวอย่าง LLD ที่ผสมทั้งการออกแบบและการเขียนโค้ด: LRU Cache

โจทย์ **LRU (Least Recently Used) Cache** เป็นตัวอย่างที่ดีที่สุดของการผสาน Coding Round
เข้ากับ System/LLD Round เพราะต้องทั้งออกแบบโครงสร้างข้อมูลที่ตอบโจทย์ Requirement (get/put
ต้องเป็น O(1)) และเขียนโค้ดที่ทำงานถูกต้องจริง

**ข้อกำหนด**: ออกแบบ Cache ที่มีความจุ (`capacity`) คงที่ รองรับ `get(key)` และ
`put(key, value)` โดยทั้งสอง operation ต้องทำงานใน **O(1) โดยเฉลี่ย** เมื่อ cache เต็มและมี
การ `put` ค่าใหม่เข้ามา ต้องเบียดตัวที่ **ไม่ถูกใช้งานนานที่สุด** ออกไปก่อน

**แนวคิดการออกแบบ**: ต้องมีโครงสร้างข้อมูลสองส่วนทำงานร่วมกัน:

1. **`unordered_map<key, iterator>`** — สำหรับค้นหาตำแหน่งของ key ใน O(1)
2. **`std::list<pair<key, value>>`** (Doubly Linked List) — สำหรับจัดลำดับการใช้งานล่าสุด
   โดยหัว list คือ "ใช้ล่าสุด" และท้าย list คือ "ใช้งานนานที่สุด" (candidate ที่จะถูกเบียดออก)

การเลือก `std::list` (ไม่ใช่ `std::vector`) สำคัญมาก เพราะการย้าย node ไปหน้า list
(`splice`) ทำได้ใน O(1) โดยไม่ต้องขยับข้อมูลเลย ในขณะที่ `std::vector` ต้องเลื่อนข้อมูล
ทั้งหมด O(n)

```cpp
#include <iostream>
#include <unordered_map>
#include <list>
#include <stdexcept>

// LRU Cache — โจทย์คลาสสิกที่เชื่อมโยง Coding Round เข้ากับ System Design Round
// ต้องให้ get(key) และ put(key, value) ทำงานใน O(1) โดยเฉลี่ยทั้งคู่
// เทคนิค: ผสม unordered_map (ค้นหาเร็ว) เข้ากับ doubly linked list (จัดลำดับการใช้งานล่าสุด)
class LRUCache {
public:
    explicit LRUCache(std::size_t capacity) : capacity_(capacity) {}

    // คืนค่า -1 ถ้าไม่พบ key (สมมติว่าค่าจริงในโจทย์นี้ไม่ติดลบ)
    int get(int key) {
        auto it = index_.find(key);
        if (it == index_.end()) {
            return -1;
        }
        // ย้าย item ที่เพิ่งถูกใช้ไปไว้หน้าสุดของ list (แปลว่า "ใช้ล่าสุด")
        usage_order_.splice(usage_order_.begin(), usage_order_, it->second);
        return it->second->second;
    }

    void put(int key, int value) {
        auto it = index_.find(key);
        if (it != index_.end()) {
            it->second->second = value;
            usage_order_.splice(usage_order_.begin(), usage_order_, it->second);
            return;
        }

        if (index_.size() >= capacity_) {
            // ตัด item ที่ใช้งานนานสุด (ท้าย list) ออกก่อน
            auto& [old_key, old_value] = usage_order_.back();
            (void)old_value;
            index_.erase(old_key);
            usage_order_.pop_back();
        }

        usage_order_.emplace_front(key, value);
        index_[key] = usage_order_.begin();
    }

private:
    using Pair = std::pair<int, int>; // {key, value}
    std::size_t capacity_;
    std::list<Pair> usage_order_;                                   // หน้า = ใช้ล่าสุด, ท้าย = เก่าสุด
    std::unordered_map<int, std::list<Pair>::iterator> index_;      // key -> ตำแหน่งใน list
};

int main() {
    LRUCache cache(2);

    cache.put(1, 100);
    cache.put(2, 200);
    std::cout << "get(1) = " << cache.get(1) << "\n"; // 100 (ทำให้ key=1 กลายเป็นล่าสุด)

    cache.put(3, 300); // capacity เต็ม (2) -> ต้องเบียด key=2 ออก (ใช้งานนานสุด)
    std::cout << "get(2) = " << cache.get(2) << "\n"; // -1 (ถูกเบียดออกไปแล้ว)

    cache.put(4, 400); // เบียด key=1 ออก (เพราะ key=3 ถูกใช้ล่าสุดกว่า)
    std::cout << "get(1) = " << cache.get(1) << "\n"; // -1
    std::cout << "get(3) = " << cache.get(3) << "\n"; // 300
    std::cout << "get(4) = " << cache.get(4) << "\n"; // 400

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 lru_cache.cpp -o lru_cache
./lru_cache
# get(1) = 100
# get(2) = -1
# get(1) = -1
# get(3) = 300
# get(4) = 400
```

**วิเคราะห์ complexity**: `get()` และ `put()` ทั้งคู่เป็น **O(1) โดยเฉลี่ย** — การค้นหาใน
`unordered_map` เฉลี่ย O(1), การย้าย/ลบ/เพิ่ม node ใน `std::list` ด้วย iterator ที่รู้ตำแหน่ง
อยู่แล้วก็เป็น O(1) เช่นกัน (ต่างจากการค้นหาตำแหน่งใน list แบบไม่รู้ iterator ซึ่งจะเป็น O(n))

**คำถาม Follow-up ที่มักตามมา (แนว System Design)**:

- "ถ้าต้องรองรับหลาย thread เข้าถึง cache พร้อมกันจะทำอย่างไร" → เพิ่ม `std::mutex` ป้องกัน
  ทั้ง `get()`/`put()` (ทบทวนจาก **Part 32** และ **Part 82**) แต่ mutex ตัวเดียวจะกลายเป็น
  คอขวด ถ้าต้องการ scale ต่อควรพิจารณา **Sharding** cache ออกเป็นหลายส่วนตาม hash ของ key
- "ถ้าข้อมูลใหญ่เกินจะเก็บใน RAM เครื่องเดียวจะทำอย่างไร" → นี่คือจุดเปลี่ยนผ่านสู่ HLD:
  พิจารณา Distributed Cache (เช่นแนวคิดของ Redis) ที่กระจายข้อมูลไปหลายเครื่อง
- "ถ้าต้องการ TTL (Time-To-Live) ให้แต่ละ key หมดอายุอัตโนมัติจะออกแบบอย่างไร" → เพิ่ม
  timestamp เข้าไปใน node และมี background thread หรือ lazy expiration ตอน `get()`

### หัวข้อ LLD อื่นที่พบบ่อยสำหรับตำแหน่งสาย C++

- **Thread-safe Bounded Queue / Producer-Consumer Queue** (ต่อยอดจาก Part 32)
- **Rate Limiter** (Token Bucket / Sliding Window Counter)
- **Parking Lot System** (ฝึก OOP: Inheritance/Polymorphism จาก Module D)
- **Object Pool Allocator** (ต่อยอดจาก Part 66 Custom Allocator และ Part 68 RAII)
- **Elevator System** (ฝึกการออกแบบ State Machine และ Concurrency)

### กรอบคิดสำหรับ HLD เบื้องต้น (สำหรับตำแหน่ง C++ สาย Backend/Infrastructure)

สำหรับตำแหน่ง C++ สาย Backend หรือ Infrastructure ที่อาจถาม HLD เต็มรูปแบบ (โดยเฉพาะระดับ
Senior ขึ้นไป) ให้ใช้กรอบคิดนี้เป็นโครงร่างเสมอ:

1. **Clarify Requirements** — Functional (ระบบต้องทำอะไรได้บ้าง) และ Non-functional
   (จำนวนผู้ใช้, Latency ที่ยอมรับได้, Consistency vs Availability)
2. **Capacity Estimation** — ประมาณ QPS (Query per Second), ปริมาณข้อมูลต่อวัน/ปี
3. **API Design** — ออกแบบ Endpoint/Interface หลักๆ
4. **Data Model** — ออกแบบ Schema/โครงสร้างข้อมูลหลัก
5. **High-Level Component Diagram** — วาดภาพรวม Client → Load Balancer → Service → Database
6. **Deep Dive** — เจาะจุดที่ผู้สัมภาษณ์สนใจเป็นพิเศษ (มักเป็นจุดคอขวดของระบบ)
7. **Identify Bottleneck & Trade-off** — พูดถึงข้อจำกัดและวิธีแก้ (Caching, Sharding,
   Replication, Async Processing)

หัวข้อนี้จะถูกฝึกแบบเต็มรูปแบบผ่านการลงมือทำจริงใน **Module K (Part 121-124)** ที่จะสร้าง
ระบบ Backend, Chat Server, Distributed Key-Value Store และ Web Framework ตั้งแต่ต้นจนจบ

---

## ข้อผิดพลาดที่พบบ่อย (Common Pitfalls)

1. **Off-by-one Error ภายใต้ความกดดันของเวลา**: ลืมว่า loop ควรเป็น `<` หรือ `<=`, คำนวณ
   `mid` ผิดจนเกิด infinite loop ใน Binary Search, ลืมว่า index สุดท้ายของอาร์เรย์คือ
   `size() - 1` ไม่ใช่ `size()` — วิธีป้องกันที่ดีที่สุดคือ **ไล่ trace ด้วยตัวอย่างเล็กๆ
   ด้วยมือเสมอก่อนบอกว่าโค้ดเสร็จแล้ว**
2. **ลืม Edge Case พื้นฐาน**: input ว่างเปล่า (`{}`, `""`), input มีสมาชิกตัวเดียว, input ที่
   ทุกตัวเหมือนกันหมด, `nullptr` สำหรับ Linked List/Tree — ควรถามและทดสอบ edge case เหล่านี้
   เป็นกิจวัตรทุกโจทย์
3. **Integer Overflow**: ผลรวมของอาร์เรย์ `int` ขนาดใหญ่อาจเกินขอบเขตของ `int` (สูงสุด
   ประมาณ 2.1 พันล้าน) ควรใช้ `long long` เมื่อมีการบวกสะสม (accumulate) จำนวนมาก อย่างที่
   เห็นในโจทย์ Two Pointers และ Binary Search on Answer ข้างต้นที่ใช้ `long long` สำหรับ
   ผลรวม/ชั่วโมงโดยเจตนา
4. **สับสน Complexity ของ `unordered_map`**: `find()`/`insert()` เป็น **O(1) โดยเฉลี่ย**
   เท่านั้น ไม่ใช่ O(1) เสมอไป — worst case (เมื่อ hash function แย่หรือถูกโจมตี) อาจเป็น
   O(n) ได้ ควรพูดถึงข้อจำกัดนี้เมื่อผู้สัมภาษณ์ถามลึก (ทบทวนจาก **Part 22**)
5. **ใช้ DFS แบบ Recursive กับ Input ที่อาจใหญ่มาก**: อย่างที่เห็นใน Number of Islands การ
   recursion ลึกเกินไปอาจทำให้ Stack Overflow ได้จริง ถ้าไม่แน่ใจขนาด input ควรเลือก BFS
   แบบ iterative หรือแปลง DFS ให้เป็น iterative ด้วย stack ของตัวเองแทน
6. **ลืมจัดการ Cycle ก่อน `delete` หรือ recurse ซ้ำ**: เมื่อทำงานกับ Linked List ที่อาจมี
   cycle การเรียก `delete` แบบวน loop ธรรมดาโดยไม่ตรวจ cycle ก่อนจะกลายเป็น infinite loop
   ทันที
7. **กระโดดเขียนโค้ดทันทีโดยไม่ถามคำถามชี้แจง**: ทำให้เสียเวลาเขียนวิธีแก้ที่ผิดโจทย์ตั้งแต่
   ต้น (เช่น สมมติว่าอาร์เรย์เรียงแล้วทั้งที่โจทย์ไม่ได้บอก)
8. **เงียบนานเกินไประหว่างคิด**: ผู้สัมภาษณ์ไม่มีทางรู้ว่ากำลังคิดอะไรอยู่ถ้าไม่พูดออกมา อาจ
   ทำให้ถูกประเมินว่า "คิดไม่ออก" ทั้งที่จริงกำลังคิดถูกทางอยู่
9. **ไม่ทดสอบโค้ดด้วยตัวอย่างก่อนบอกว่า "เสร็จแล้ว"**: ควรไล่ trace อย่างน้อย 1-2 ตัวอย่าง
   (รวม edge case) ด้วยตัวเองก่อนเสมอ เหมือนที่ทำใน `main()` ของทุกโจทย์ในบทนี้
10. **ท่องจำโค้ดมาโดยไม่เข้าใจ**: เมื่อผู้สัมภาษณ์เปลี่ยนเงื่อนไขเล็กน้อย (เช่น "ถ้าอาร์เรย์นี้
    เรียงแล้วล่ะ จะทำได้เร็วกว่านี้ไหม") ผู้สมัครที่ท่องจำมาแบบไม่เข้าใจ pattern จะตอบไม่ได้ทันที

---

## แบบฝึกหัดท้ายบท

1. **Best Time to Buy and Sell Stock**: ให้อาร์เรย์ราคาหุ้นรายวัน `prices` หากำไรมากที่สุดจาก
   การซื้อ 1 ครั้งแล้วขาย 1 ครั้ง (ต้องซื้อก่อนขายเสมอ ถ้าทำกำไรไม่ได้เลยให้คืนค่า 0)
2. **Group Anagrams**: ให้ array ของสตริง จัดกลุ่มคำที่เป็น anagram กัน (ใช้ตัวอักษรชุดเดียวกัน
   แต่เรียงต่างกัน) ให้อยู่กลุ่มเดียวกัน
3. **Find Minimum in Rotated Sorted Array**: อาร์เรย์ที่เรียงแล้วถูกหมุน (rotate) ไปบางจำนวน
   ตำแหน่ง (เช่น `[4,5,6,7,0,1,2]`) หาค่าต่ำสุดในอาร์เรย์นี้โดยใช้เวลา O(log n)
4. **Implement Queue using Two Stacks**: ให้ implement Queue (FIFO) โดยใช้ Stack (LIFO) สอง
   ตัวเท่านั้น (ทบทวนแนวคิดจาก **Part 20**)
5. **Linked List Cycle II**: ต่อยอดจากโจทย์ Fast & Slow Pointer ใน 119.2 — ถ้า Linked List มี
   cycle จริง ให้หา**โหนดที่เป็นจุดเริ่มต้นของ cycle** นั้น (ไม่ใช่แค่ตรวจว่ามี cycle หรือไม่)
6. **Top K Frequent Elements**: ให้อาร์เรย์ของจำนวนเต็ม หา `k` ค่าที่ปรากฏบ่อยที่สุด (ทบทวน
   แนวคิด Hash Table จาก **Part 22** ร่วมกับ Heap จากโจทย์ Kth Largest Element ใน 119.5)

### แนวทางเฉลยข้อ 1: Best Time to Buy and Sell Stock

**แนวคิด**: วน loop ครั้งเดียว เก็บราคาต่ำสุดที่เจอมาแล้ว (`min_price_so_far`) ทุกวัน ถ้าราคา
วันนี้ต่ำกว่าราคาต่ำสุดเดิม ให้อัปเดตราคาต่ำสุด ไม่เช่นนั้นให้คำนวณกำไรถ้าขายวันนี้แล้วเทียบกับ
กำไรที่ดีที่สุดที่เคยเจอ (คล้ายกับ Pattern Sliding Window ที่มี "ตัวติดตามค่าที่ดีที่สุด" วิ่งไป
พร้อมกับ pointer เดียว)

```cpp
#include <iostream>
#include <vector>
#include <algorithm>

// แบบฝึกหัดข้อ 1: Best Time to Buy and Sell Stock (ซื้อขายได้ครั้งเดียว)
// หากำไรมากที่สุดจากการซื้อ 1 ครั้งแล้วขาย 1 ครั้ง (ต้องซื้อก่อนขายเสมอ)
int max_profit(const std::vector<int>& prices) {
    if (prices.empty()) {
        return 0;
    }
    int min_price_so_far = prices[0];
    int best_profit = 0;

    for (std::size_t i = 1; i < prices.size(); ++i) {
        if (prices[i] < min_price_so_far) {
            min_price_so_far = prices[i]; // เจอราคาต่ำสุดใหม่ -> อัปเดตจุดซื้อที่ดีที่สุด
        } else {
            int profit_if_sell_today = prices[i] - min_price_so_far;
            best_profit = std::max(best_profit, profit_if_sell_today);
        }
    }
    return best_profit;
}

int main() {
    std::vector<int> p1 = {7, 1, 5, 3, 6, 4};
    std::cout << "p1 -> " << max_profit(p1) << "\n"; // 5 (ซื้อวันที่ราคา 1 ขายวันที่ราคา 6)

    std::vector<int> p2 = {7, 6, 4, 3, 1};
    std::cout << "p2 -> " << max_profit(p2) << "\n"; // 0 (ราคาลดลงตลอด ไม่ควรซื้อขายเลย)

    std::vector<int> p3 = {};
    std::cout << "p3 (empty) -> " << max_profit(p3) << "\n"; // 0 (edge case)

    std::vector<int> p4 = {5};
    std::cout << "p4 (single) -> " << max_profit(p4) << "\n"; // 0 (edge case: มีวันเดียว)

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 stock.cpp -o stock
./stock
# p1 -> 5
# p2 -> 0
# p3 (empty) -> 0
# p4 (single) -> 0
```

**วิเคราะห์**: Time O(n), Space O(1) — เป็นตัวอย่างที่ดีของการมองโจทย์ที่ดูเหมือนต้องใช้
Two Pointers (ซื้อ-ขาย) แต่จริงๆ แล้วแก้ได้ด้วย **single-pass tracking** ที่ง่ายกว่ามาก

### แนวทางเฉลยข้อ 2: Group Anagrams

**แนวคิด**: คำสองคำจะเป็น anagram กันก็ต่อเมื่อ**ตัวอักษรที่เรียงแล้วเหมือนกันทุกประการ** จึง
ใช้ "สตริงที่เรียงตัวอักษรแล้ว" เป็น **key** ของ `unordered_map` เพื่อจัดกลุ่มคำที่มี key
เดียวกันเข้าด้วยกัน

```cpp
#include <iostream>
#include <vector>
#include <string>
#include <unordered_map>
#include <algorithm>

// แบบฝึกหัดข้อ 2: Group Anagrams
// จัดกลุ่มคำที่เป็น anagram กัน (ใช้ตัวอักษรชุดเดียวกันแต่เรียงต่างกัน) ให้อยู่กลุ่มเดียวกัน
std::vector<std::vector<std::string>> group_anagrams(const std::vector<std::string>& strs) {
    // key: ตัวอักษรของคำที่ถูกเรียงแล้ว (คำที่เป็น anagram กันจะได้ key เดียวกันเป๊ะ)
    std::unordered_map<std::string, std::vector<std::string>> groups;
    groups.reserve(strs.size() * 2);

    for (const std::string& s : strs) {
        std::string key = s;
        std::sort(key.begin(), key.end());
        groups[key].push_back(s);
    }

    std::vector<std::vector<std::string>> result;
    result.reserve(groups.size());
    for (auto& [key, words] : groups) {
        result.push_back(std::move(words));
    }
    return result;
}

int main() {
    std::vector<std::string> input = {"eat", "tea", "tan", "ate", "nat", "bat"};
    auto groups = group_anagrams(input);

    std::cout << "จำนวนกลุ่มทั้งหมด: " << groups.size() << "\n"; // ควรได้ 3 กลุ่ม
    for (auto& group : groups) {
        std::sort(group.begin(), group.end()); // เรียงแค่เพื่อให้ผลลัพธ์ print อ่านง่าย/คงที่
        std::cout << "{ ";
        for (const auto& word : group) {
            std::cout << word << " ";
        }
        std::cout << "}\n";
    }

    std::vector<std::string> empty_input = {};
    auto empty_result = group_anagrams(empty_input);
    std::cout << "empty input -> จำนวนกลุ่ม: " << empty_result.size() << "\n"; // 0

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 group_anagrams.cpp -o group_anagrams
./group_anagrams
# จำนวนกลุ่มทั้งหมด: 3
# { bat }
# { nat tan }
# { ate eat tea }
# empty input -> จำนวนกลุ่ม: 0
```

**วิเคราะห์**: Time O(n × k log k) โดย `n` คือจำนวนคำและ `k` คือความยาวเฉลี่ยของแต่ละคำ
(เพราะต้อง sort ตัวอักษรในทุกคำ), Space O(n × k) สำหรับเก็บผลลัพธ์และ map

**คำใบ้สำหรับข้อ 3-6**: ข้อ 3 ใช้ Pattern **Binary Search on Answer/Modified Binary Search**
โดยเทียบค่ากลาง (`mid`) กับค่าปลายขวา (`right`) เพื่อตัดสินว่าจุดหมุน (rotation point) อยู่ครึ่ง
ไหน — ข้อ 4 ใช้สอง stack โดย stack หนึ่งสำหรับ `enqueue` และอีกอันสำหรับ `dequeue` (ย้ายข้อมูล
ข้ามกันเฉพาะตอนจำเป็น) — ข้อ 5 ต่อยอดจาก Fast & Slow Pointer โดยเมื่อ `slow` กับ `fast` ชนกัน
แล้ว ให้ตั้ง pointer ใหม่จาก `head` แล้ววิ่งพร้อมกับ `slow` ทีละก้าว จุดที่ชนกันครั้งที่สองคือ
จุดเริ่มต้นของ cycle (พิสูจน์ได้ด้วยคณิตศาสตร์ระยะทาง) — ข้อ 6 ใช้ `unordered_map` นับความถี่
ก่อน แล้วใช้ Min-Heap ขนาด `k` แบบเดียวกับโจทย์ Kth Largest Element เพื่อหา `k` ค่าที่บ่อยที่สุด

---

## สรุปท้ายบท

ใน Part นี้เราได้:

- ทำความเข้าใจโครงสร้างการสัมภาษณ์งานสาย C++ ทั้ง Coding Round, C++ Deep-Dive Round,
  System/Low-Level Design Round และ Behavioral Round พร้อมน้ำหนักที่เปลี่ยนไปตามระดับตำแหน่ง
- เรียนรู้ 4 Pattern หลักที่ครอบคลุมโจทย์ LeetCode ส่วนใหญ่: Two Pointers, Sliding Window,
  Fast & Slow Pointer และ Binary Search on Answer พร้อมโค้ดที่คอมไพล์และรันได้จริงทุกตัวอย่าง
- แก้โจทย์คลาสสิก 7 ข้อครบทุกข้อ (Two Sum, Valid Parentheses, Longest Substring Without
  Repeating Characters, Merge Intervals, Reverse Linked List, Kth Largest Element, Number
  of Islands) พร้อมวิเคราะห์ complexity และเชื่อมโยงกลับไปยัง Part 19-25 ที่เรียนมาก่อนหน้านี้
- ฝึกกรอบคิด UMPIRE Method และเทคนิคการสื่อสารที่มืออาชีพใช้ในห้องสัมภาษณ์จริง
- ทำความเข้าใจความแตกต่างระหว่าง Low-Level Design และ High-Level Design ผ่านตัวอย่าง
  LRU Cache ที่ผสมทั้งการเขียนโค้ดและการออกแบบระบบเข้าด้วยกัน

ทักษะการแก้โจทย์ DS&A ที่ฝึกใน Part นี้เป็นเพียง "ใบเบิกทาง" เข้าสู่งาน แต่สิ่งที่จะกำหนด
ความก้าวหน้าในอาชีพระยะยาวคือทักษะที่กว้างกว่านั้นมาก ใน **Part 120** ซึ่งเป็น Part สุดท้าย
ของ Module J เราจะมองภาพรวมทั้งเส้นทางอาชีพสาย C++ ตั้งแต่ Junior จนถึง Staff/Principal
Engineer การสร้าง Portfolio ที่โดดเด่น และการมีส่วนร่วมกับ Open Source Community

**ต่อไป:** [Part 120 — เส้นทางอาชีพ: จาก Junior สู่ Senior/Staff Engineer และ Open Source](./part-120-career-path.md)
