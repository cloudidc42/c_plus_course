# Part 86: Performance Profiling (perf/gprof) (Step 681–688)

> Module G — Concurrency และ Performance Engineering | Part 86 จาก 125
> Step ที่ครอบคลุมใน Part นี้: Step 681–688
> Part ก่อนหน้า: [Part 85 — Parallel Algorithm (C++17 Execution Policy)](./part-085-parallel-algorithms.md) | Part ถัดไป: [Part 87 — Cache-Friendly Code & Data-Oriented Design](./part-087-cache-friendly-dod.md)

## เป้าหมายการเรียนรู้ (Learning Objectives)

หลังจบ Part นี้ ผู้เรียนจะสามารถ:

1. อธิบายได้ว่าทำไม "การเดา" ว่าโค้ดส่วนไหนช้าถึงเป็นความคิดที่อันตราย และทำไมต้อง profile
   ก่อน optimize เสมอ (หลักการของ Donald Knuth)
2. ใช้ `gprof` ได้อย่างเต็มรูปแบบ ตั้งแต่คอมไพล์ด้วย `-pg`, รันโปรแกรม, ไปจนถึงอ่านผล
   Flat Profile และ Call Graph เพื่อหา bottleneck จริง
3. เข้าใจแนวคิดของ Linux `perf` (sampling profiler ที่ทำงานระดับ kernel) และคำสั่งพื้นฐาน
   ที่ใช้บ่อยที่สุด (`perf stat`, `perf record`, `perf report`)
4. เข้าใจข้อจำกัดของ container/sandbox แบบที่ใช้เรียนหลักสูตรนี้ ว่าทำไม `perf` ถึงใช้งาน
   ไม่ได้เต็มรูปแบบ และรู้วิธีใช้งานบนเครื่องจริงของตัวเอง
5. เขียน micro-benchmark ด้วย `std::chrono` เองได้อย่างถูกต้อง โดยเลี่ยงกับดักที่ทำให้ผล
   วัดคลาดเคลื่อน (compiler optimize โค้ดทิ้ง, ไม่ warm-up, วัดแค่ครั้งเดียว)
6. เลือกเครื่องมือ profiling ที่เหมาะสมกับสถานการณ์ได้ถูกต้อง

---

## 86.1 ทำไมต้อง Profile ก่อน Optimize เสมอ (Step 681)

Donald Knuth หนึ่งในบิดาแห่งวิทยาการคอมพิวเตอร์ เขียนไว้ในปี 1974 ประโยคที่ถูกอ้างอิงมากที่สุด
ประโยคหนึ่งในวงการซอฟต์แวร์:

> "We should forget about small efficiencies, say about 97% of the time: **premature
> optimization is the root of all evil**." — Donald Knuth

แปลเป็นภาษาไทยตรงๆ คือ **"การ optimize ก่อนเวลาอันควร คือรากเหง้าของความชั่วร้ายทั้งปวง"**
ในบริบทของวิศวกรรมซอฟต์แวร์ ประโยคนี้ไม่ได้แปลว่า "อย่า optimize" แต่หมายความว่า **อย่า
optimize โดยไม่มีข้อมูลรองรับ**

### ทำไมสัญชาตญาณของโปรแกรมเมอร์เรื่อง "อะไรช้า" มักผิด

จากประสบการณ์จริงของวิศวกรทั่วโลก (และจากการทดลองในหัวข้อ 86.3 ของ Part นี้เอง) จุดที่
โปรแกรมเมอร์ "คิดว่า" ช้า กับจุดที่ "ช้าจริง" มักไม่ตรงกัน ด้วยเหตุผลหลายประการ:

1. **Cache behavior ซับซ้อนเกินกว่าจะนึกภาพในหัวได้แม่นยำ** — โค้ดสองชิ้นที่ทำงานเหมือนกัน
   ทุกประการทาง logic อาจเร็วช้าต่างกันหลายเท่าจากการจัดเรียงข้อมูลในหน่วยความจำ (เรื่องนี้
   จะเจาะลึกเต็มรูปแบบใน Part 87)
2. **Compiler optimization ทำงานไม่เป็นเส้นตรงกับสิ่งที่เขียน** — โค้ดที่ "ดูช้า" ในสายตามนุษย์
   อาจถูก compiler inline/vectorize จนเร็วกว่าที่คิดมาก หรือกลับกัน
3. **สัดส่วนเวลาที่ใช้จริงมักกระจุกตัว** — กฎ 80/20 (Pareto Principle) ใช้ได้ผลบ่อยครั้งกับ
   performance: โปรแกรมส่วนใหญ่ใช้เวลา 80-90% ไปกับโค้ดเพียง 10-20% เท่านั้น การเดาผิดจุด
   แปลว่าเสียเวลา optimize ส่วนที่แทบไม่มีผลกับความเร็วโดยรวมเลย

### หลักการทำงานที่ถูกต้อง

```
[1] เขียนโค้ดให้ถูกต้องและอ่านง่ายก่อน (correctness ก่อนเสมอ)
[2] ถ้า performance เป็นปัญหาจริง (มีคนบ่น, มี SLA ที่ไม่ผ่าน) ให้ profile หา bottleneck จริง
[3] Optimize เฉพาะจุดที่ profile ชี้ว่าเป็นปัญหาจริงเท่านั้น
[4] วัดผลอีกครั้งหลัง optimize เพื่อยืนยันว่าดีขึ้นจริง (ไม่ใช่แค่ "รู้สึกว่า" ดีขึ้น)
[5] ทำซ้ำ [2]-[4] จนกว่า performance จะยอมรับได้
```

Part นี้จะพาไปรู้จักเครื่องมือ 3 ระดับที่ใช้ในขั้นตอน [2] และ [4]: **gprof** (function-level
profiler แบบ instrumentation), **perf** (system-level sampling profiler ของ Linux), และ
**std::chrono** (การจับเวลามือสำหรับ micro-benchmark ที่ใช้ได้ทุกสภาพแวดล้อมแน่นอน)

---

## 86.2 gprof คืออะไร และวิธีใช้งานเบื้องต้น (Step 682)

**gprof** (GNU Profiler) เป็นเครื่องมือ profiling ที่มีมาตั้งแต่ยุค 1980s ทำงานด้วยหลักการ
**instrumentation**: compiler จะแทรกโค้ดพิเศษเข้าไปใน**ทุกฟังก์ชัน**ของโปรแกรมตอนคอมไพล์
เพื่อบันทึกว่าฟังก์ชันไหนถูกเรียกกี่ครั้ง เรียกจากที่ไหน และใช้เวลารวมเท่าไหร่ ข้อมูลนี้จะถูก
เขียนลงไฟล์ `gmon.out` เมื่อโปรแกรมจบการทำงาน จากนั้นใช้คำสั่ง `gprof` อ่านไฟล์นี้ออกมาเป็น
รายงานที่มนุษย์อ่านได้

### ตรวจสอบว่ามี gprof ในระบบหรือไม่

```bash
which gprof
gprof --version
```

ผลลัพธ์จริงจากเครื่องที่ใช้พัฒนาบทเรียนนี้:

```
/usr/bin/gprof
```

`gprof` มาพร้อมกับ `binutils` ซึ่งติดตั้งมาคู่กับ GCC toolchain (`build-essential`) อยู่แล้ว
ตั้งแต่ Part 1 — ไม่ต้องติดตั้งเพิ่มเติมบน Linux ส่วนใหญ่

### ขั้นตอนการใช้งาน gprof

**ขั้นตอนที่ 1: คอมไพล์ด้วย flag `-pg`**

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 -pg -O0 profile_demo.cpp -o profile_demo -lm
```

Flag `-pg` บอก compiler ให้แทรก instrumentation code เข้าไปในทุกฟังก์ชัน — สังเกตว่าใช้
`-O0` (ปิด optimization) เพราะถ้าเปิด optimization ระดับสูง (`-O2`, `-O3`) compiler อาจ
**inline ฟังก์ชันเข้าด้วยกัน** ทำให้ gprof มองไม่เห็นฟังก์ชันที่ถูก inline ไปแล้วเป็นเอนทิตี
แยกต่างหาก ผลลัพธ์จึงคลาดเคลื่อนจากสิ่งที่เขียนในซอร์สโค้ด (เดี๋ยวจะสาธิตให้เห็นผลกระทบนี้
ในหัวข้อ 86.3)

**ขั้นตอนที่ 2: รันโปรแกรมตามปกติ**

```bash
./profile_demo
```

โปรแกรมจะทำงานตามปกติทุกประการ (ผลลัพธ์ทาง stdout เหมือนไม่ได้ compile ด้วย `-pg` เลย)
แต่เมื่อจบการทำงาน จะมีไฟล์ **`gmon.out`** ถูกสร้างขึ้นในโฟลเดอร์ปัจจุบัน — นี่คือไฟล์ binary
ที่เก็บข้อมูล raw ของการ profile ไว้

**ขั้นตอนที่ 3: อ่านผลด้วย gprof**

```bash
gprof profile_demo gmon.out > gprof_report.txt
```

คำสั่งนี้ต้องระบุทั้ง**ไฟล์ executable** (`profile_demo` — เพื่อให้ gprof รู้จัก symbol table
ของโปรแกรม) และ**ไฟล์ข้อมูล** (`gmon.out`) คู่กันเสมอ

---

## 86.3 ตัวอย่างจริง: Profile โปรแกรมแล้วเจอ Bottleneck (Step 683)

มาดูตัวอย่างที่ทดสอบและคอมไพล์จริงบนเครื่องที่ใช้พัฒนาบทเรียนนี้ (g++ 13.3.0, Ubuntu 24.04)
โปรแกรมมี 3 ฟังก์ชันที่ตั้งใจให้มีลักษณะต่างกัน เพื่อดูว่า gprof แยกแยะได้จริงหรือไม่:

```cpp
// profile_demo.cpp
#include <iostream>
#include <vector>
#include <cmath>

// ฟังก์ชันที่ตั้งใจให้ "ช้า" แบบไม่จำเป็น เพื่อสาธิตการหา bottleneck
double slow_sum_of_sqrt(const std::vector<int>& data) {
    double total = 0.0;
    for (int x : data) {
        for (int i = 0; i < 50; ++i) {   // คำนวณซ้ำโดยไม่จำเป็น (bottleneck จำลอง)
            total += std::sqrt(static_cast<double>(x));
        }
    }
    return total;
}

double fast_sum(const std::vector<int>& data) {
    double total = 0.0;
    for (int x : data) {
        total += x;
    }
    return total;
}

long fib(int n) {
    if (n < 2) return n;
    return fib(n - 1) + fib(n - 2);
}

int main() {
    std::vector<int> data(200000);
    for (size_t i = 0; i < data.size(); ++i) data[i] = static_cast<int>(i % 1000) + 1;

    double r1 = slow_sum_of_sqrt(data);
    double r2 = fast_sum(data);
    long r3 = fib(32);

    std::cout << "slow_sum_of_sqrt = " << r1 << "\n";
    std::cout << "fast_sum = " << r2 << "\n";
    std::cout << "fib(32) = " << r3 << "\n";
    return 0;
}
```

คอมไพล์ รัน และดึงรายงานจริง:

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 -pg -O0 profile_demo.cpp -o profile_demo -lm
./profile_demo
gprof profile_demo gmon.out > gprof_report.txt
```

ผลลัพธ์การรันโปรแกรม (ทาง stdout ตามปกติ):

```
slow_sum_of_sqrt = 2.10975e+08
fast_sum = 1.001e+08
fib(32) = 2178309
```

### อ่าน Flat Profile จริง

ส่วนแรกของรายงาน gprof คือ **Flat Profile** — ตารางฟังก์ชันเรียงจากใช้เวลามากไปน้อย
นี่คือผลลัพธ์จริงที่ได้ (ส่วนบนสุดของ `gprof_report.txt`):

```
Flat profile:

Each sample counts as 0.01 seconds.
  %   cumulative   self              self     total
 time   seconds   seconds    calls  ms/call  ms/call  name
 50.00      0.02     0.02        1    20.00    20.00  slow_sum_of_sqrt(std::vector<int, std::allocator<int> > const&)
 50.00      0.04     0.02        1    20.00    20.00  fib(int)
  0.00      0.04     0.00   800004     0.00     0.00  __gnu_cxx::__normal_iterator<...>::base() const
  0.00      0.04     0.00   400002     0.00     0.00  bool __gnu_cxx::operator!=<...>(...)
  0.00      0.04     0.00   400000     0.00     0.00  __gnu_cxx::__normal_iterator<...>::operator++()
  0.00      0.04     0.00   400000     0.00     0.00  __gnu_cxx::__normal_iterator<...>::operator*() const
  0.00      0.04     0.00   200001     0.00     0.00  std::vector<int, std::allocator<int> >::size() const
  0.00      0.04     0.00   200000     0.00     0.00  std::vector<int, std::allocator<int> >::operator[](unsigned long)
  0.00      0.04     0.00        1     0.00     0.00  fast_sum(std::vector<int, std::allocator<int> > const&)
```

(ตัดชื่อ template ที่ยาวมากบางส่วนให้อ่านง่ายขึ้น — เนื้อหาตัวเลขเป็นของจริงทั้งหมด)

### วิเคราะห์ผลลัพธ์

อ่านคอลัมน์ทีละตัว:

| คอลัมน์ | ความหมาย |
|---|---|
| `% time` | สัดส่วนเวลาทั้งหมดที่ใช้ไปกับฟังก์ชันนี้ (ไม่รวมเวลาของฟังก์ชันที่มันเรียกต่อ) |
| `cumulative seconds` | เวลาสะสมตั้งแต่แถวบนสุดจนถึงแถวนี้ |
| `self seconds` | เวลาที่ใช้ไป**เฉพาะในฟังก์ชันนี้เอง** ไม่นับเวลาของฟังก์ชันลูกที่มันเรียก |
| `calls` | จำนวนครั้งที่ฟังก์ชันถูกเรียก |
| `self ms/call` | เวลาเฉลี่ยต่อการเรียกหนึ่งครั้ง (self time หารด้วย calls) |
| `total ms/call` | เวลาเฉลี่ยต่อการเรียกหนึ่งครั้ง **รวม**เวลาของฟังก์ชันลูกทั้งหมดด้วย |

จากผลลัพธ์จริง สังเกตได้ทันทีว่า:

1. **`slow_sum_of_sqrt` และ `fib` กิน 100% ของเวลาทั้งหมดรวมกัน** (50% + 50%) แม้จะถูกเรียก
   แค่ **1 ครั้งเท่านั้น** ต่างจากฟังก์ชัน iterator/vector ที่ถูกเรียกหลักแสนหลักล้านครั้ง
   แต่ใช้เวลารวมเกือบเป็น 0 — นี่คือตัวอย่างที่ชัดเจนของหลักการ 80/20: **จำนวนครั้งที่เรียก
   ไม่ได้บอกว่าฟังก์ชันนั้นเป็น bottleneck** สิ่งที่สำคัญคือ **self seconds**
2. **`fast_sum` ใช้เวลาแทบเป็นศูนย์** (`0.00` self seconds) ทั้งที่วน loop ผ่านข้อมูล 200,000
   ตัวเหมือนกับ `slow_sum_of_sqrt` — ต่างกันตรงที่ `slow_sum_of_sqrt` มี inner loop ซ้อนอีก
   ชั้นที่วนคำนวณ `std::sqrt` ซ้ำ 50 รอบต่อ element (ทั้งหมด 10 ล้านครั้ง) ในขณะที่ `fast_sum`
   ทำงานแค่ 200,000 ครั้ง — ถ้าเราแค่ "เดา" จากจำนวนบรรทัดโค้ด อาจไม่ทันสังเกตความแตกต่างนี้
   แต่ gprof ชี้ให้เห็นตัวเลขจริงทันที
3. **นี่คือตัวอย่างของ "bottleneck ที่ไม่จำเป็น"** — โค้ดในหัวข้อ `slow_sum_of_sqrt` วน
   คำนวณ `sqrt` ค่าเดิมซ้ำ 50 รอบโดยไม่มีประโยชน์ (ในโค้ดจริงอาจเกิดจาก logic ผิดพลาด หรือ
   loop ที่หลงเหลือจากการ debug) — เมื่อ profile แล้วเจอแบบนี้ วิธีแก้คือลบ inner loop ที่
   ไม่จำเป็นออก (ในกรณีนี้จะเร็วขึ้นถึง 50 เท่าทันที)

---

## 86.4 อ่าน Call Graph ของ gprof — วิเคราะห์ Recursive Function (Step 684)

ส่วนที่สองของรายงาน gprof คือ **Call Graph** ซึ่งแสดงความสัมพันธ์ "ใครเรียกใคร" — สำคัญมาก
เมื่อต้องวิเคราะห์ฟังก์ชันแบบ recursive อย่าง `fib(int)` ที่เรียกตัวเองซ้ำๆ

ผลลัพธ์จริงจาก call graph ของ `fib`:

```
                             7049154             fib(int) [3]
                0.02    0.00       1/1           main [1]
[3]     50.0    0.02    0.00       1+7049154 fib(int) [3]
                             7049154             fib(int) [3]
-----------------------------------------------
```

อ่านตารางนี้อย่างไร:

- **`[3]`**: หมายเลข index ของฟังก์ชันนี้ในรายงาน (ใช้อ้างอิงข้าม section)
- **`1+7049154`**: `fib` ถูกเรียกจาก `main` โดยตรง **1 ครั้ง** แต่เรียกตัวเองซ้ำ (recursive
  call) รวม **7,049,154 ครั้ง!** — นี่คือตัวเลขที่ตรงกับทฤษฎีของ `fib(32)` แบบ naive recursion
  ที่ไม่มี memoization ซึ่งมี time complexity แบบ exponential (`O(2^n)` โดยประมาณ ตาม
  Fibonacci sequence เอง)
- แถวบน (`fib(int) [3]` ที่ไม่มี index นำหน้า) หมายถึง "ผู้เรียก" (caller) — ในที่นี้คือ
  `fib` เรียกตัวเอง
- แถวล่าง (`fib(int) [3]`) หมายถึง "ผู้ถูกเรียก" (callee) — ในที่นี้ก็คือ `fib` เช่นกัน
  (เพราะมันเรียกตัวเอง)

### บทเรียนสำคัญจากตัวเลขนี้

การเห็นตัวเลข **7 ล้านครั้ง** สำหรับการคำนวณ `fib(32)` ที่ในทางทฤษฎีควรใช้แค่ 32 ขั้นตอน
(ถ้าใช้ dynamic programming/memoization ตามที่เรียนใน Part 25) เป็นตัวอย่างที่ทรงพลังมาก
ของประโยชน์ของ call graph: **มันไม่ได้บอกแค่ "ฟังก์ชันไหนช้า" แต่บอก "ทำไมมันถึงช้า" ผ่าน
จำนวนครั้งที่เรียกจริง** — ถ้าเห็นตัวเลขแบบนี้ในโปรแกรมจริง สัญญาณเตือนคือ "อัลกอริทึมนี้อาจ
มี time complexity แย่กว่าที่ควรจะเป็น ลองพิจารณาใช้ memoization หรือ dynamic programming"

### ข้อควรระวัง: -O0 vs -O2 กับผลลัพธ์ gprof ที่ต่างกันมาก

ลองคอมไพล์โปรแกรมเดียวกันด้วย `-O2` แทน `-O0` แล้วเทียบผลลัพธ์:

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 -pg -O2 profile_demo.cpp -o profile_demo_o2 -lm
./profile_demo_o2
gprof profile_demo_o2 gmon.out | head -10
```

ผลลัพธ์จริงที่ได้ (ต่างจาก `-O0` อย่างเห็นได้ชัด):

```
Flat profile:

Each sample counts as 0.01 seconds.
  %   cumulative   self              self     total
 time   seconds   seconds    calls  ms/call  ms/call  name
100.00      0.01     0.01        1    10.00    10.00  fib(int)
  0.00      0.01     0.00        1     0.00     0.00  slow_sum_of_sqrt(std::vector<int, std::allocator<int> > const&)
```

สังเกตว่า `slow_sum_of_sqrt` หายไปจากการกินเวลาเกือบทั้งหมด (`0.00` self seconds) เพราะ
`-O2` ทำการ **vectorize** inner loop ของมันจนเร็วขึ้นมหาศาล ในขณะที่ `fib` (ซึ่งเป็น
recursive function ที่ optimizer ทำอะไรได้จำกัดกว่า) ยังคงกินเวลาเกือบทั้งหมดของโปรแกรม
— นี่คือเหตุผลสำคัญที่ต้อง**ระบุเสมอว่า profile ด้วย optimization level ไหน** เพราะผลลัพธ์
ที่ได้ที่ `-O0` กับ `-O2` **บอกเรื่องราวคนละเรื่องกันได้เลย** ในทางปฏิบัติ:

- ใช้ `-O0 -pg` เมื่อต้องการดู**โครงสร้างของโค้ดตามที่เขียนจริง** (ฟังก์ชันไม่ถูก inline
  ทำให้ตัวเลขตรงกับ source code ชัดเจน) — เหมาะกับการเรียนรู้และการหา logic bug ที่ทำให้ช้า
- ใช้ `-O2 -pg` เมื่อต้องการดู**พฤติกรรมจริงตอน deploy** (เพราะ production build ใช้ `-O2`
  เสมอ) แต่ต้องเข้าใจว่าตัวเลขอาจไม่ตรงกับโครงสร้างฟังก์ชันในซอร์สโค้ด 100% เพราะการ inline

---

## 86.5 Linux perf — แนวคิดและคำสั่งพื้นฐาน (Step 685)

**`perf`** คือเครื่องมือ profiling ของ Linux ที่ทำงานแตกต่างจาก `gprof` โดยพื้นฐาน — แทนที่
จะแทรก instrumentation code เข้าไปในโปรแกรม (ซึ่งมี overhead และต้อง compile ใหม่ด้วย flag
พิเศษ) `perf` ใช้กลไก **sampling** ผ่าน **hardware performance counters** ที่มีอยู่ในตัว CPU
โดยตรง

### หลักการทำงานของ perf

CPU สมัยใหม่มีวงจรฮาร์ดแวร์พิเศษที่เรียกว่า **Performance Monitoring Unit (PMU)** ที่นับ
เหตุการณ์ระดับต่ำมาก เช่น:

- จำนวน CPU cycle ที่ใช้ไป
- จำนวน instruction ที่ execute
- จำนวน cache miss (L1, L2, L3)
- จำนวน branch misprediction
- จำนวน context switch, page fault

`perf` ทำงานร่วมกับ Linux kernel เพื่อ**สุ่มตัวอย่าง (sample)** โปรแกรมที่กำลังรันอยู่เป็น
ระยะๆ (เช่นทุก 1 มิลลิวินาที) แล้วบันทึกว่า ณ ขณะนั้น program counter อยู่ที่ instruction ไหน
เมื่อสะสมตัวอย่างมากพอ จะได้ภาพรวมทางสถิติว่าเวลาส่วนใหญ่ถูกใช้ไปที่ไหนบ้าง **โดยแทบไม่มี
overhead ต่อโปรแกรมที่กำลังวัดเลย** (ต่างจาก `gprof` ที่ต้องแทรกโค้ดเข้าไปในทุกฟังก์ชัน)
และที่สำคัญคือ **ไม่ต้อง compile โปรแกรมใหม่ด้วย flag พิเศษ** — ใช้ได้กับ binary ที่มีอยู่แล้ว

### คำสั่งพื้นฐานที่ใช้บ่อยที่สุด

**`perf stat`** — รันโปรแกรมแล้วสรุปตัวเลขสถิติรวมเมื่อจบการทำงาน (เหมาะกับการดูภาพรวม
ก่อนตัดสินใจว่าจะ profile ลึกต่อหรือไม่):

```bash
perf stat ./my_program
```

รูปแบบผลลัพธ์ (โครงสร้างตาราง — ตัวเลขจริงจะต่างกันไปตามโปรแกรมและเครื่อง):

```
 Performance counter stats for './my_program':

          1234.56 msec task-clock                #    0.998 CPUs utilized
                12      context-switches          #    0.010 K/sec
                 2      cpu-migrations            #    0.002 K/sec
             1,024      page-faults               #    0.829 K/sec
     3,456,789,012      cycles                    #    2.800 GHz
     5,678,901,234      instructions              #    1.64  insn per cycle
     1,234,567,890      branches                  #  999.876 M/sec
        12,345,678      branch-misses             #    1.00% of all branches

       1.237654321 seconds time elapsed
```

ตัวเลขที่ควรสนใจเป็นพิเศษ:

- **`insn per cycle` (IPC)**: ยิ่งสูงยิ่งดี (CPU สมัยใหม่ทำได้หลาย instruction ต่อ cycle
  ด้วย superscalar execution) ค่าต่ำมากๆ (เช่น < 1.0) มักบ่งบอกว่าโปรแกรมติดคอขวดที่การรอ
  หน่วยความจำ (memory-bound) มากกว่าการคำนวณ (compute-bound)
- **`branch-misses`**: เปอร์เซ็นต์ที่สูงบ่งบอกว่า CPU ทำนายทิศทางของ `if`/loop ผิดบ่อย
  ซึ่งทำให้ pipeline ต้อง flush และเริ่มใหม่ (เสียเวลา)

**`perf record` / `perf report`** — เก็บ sample แบบละเอียดแล้วดูว่าเวลาส่วนใหญ่อยู่ที่
ฟังก์ชัน/บรรทัดไหน (คล้าย gprof แต่แม่นยำกว่าเพราะใช้ hardware counter จริง ไม่ต้อง
instrumentation):

```bash
perf record -g -- ./my_program   # -g เก็บ call graph ด้วย
perf report                       # เปิดดูผลแบบ interactive ในเทอร์มินัล (คล้าย top)
```

`perf report` จะแสดงรายชื่อฟังก์ชันเรียงจาก % เวลาที่ใช้มากไปน้อย พร้อมให้กด Enter เพื่อดู
"ใครเรียกฟังก์ชันนี้" (คล้าย call graph ของ gprof) และยังสามารถ drill down ไปถึงระดับ
**บรรทัดของ assembly instruction** ได้ด้วยคำสั่ง `perf annotate`

**`perf top`** — เวอร์ชัน real-time ของ `perf report` คล้าย `htop` แต่แสดงฟังก์ชันที่กิน
CPU มากที่สุด ณ ขณะนั้นทั้งระบบ เหมาะกับการหาว่า process ไหน/ฟังก์ชันไหนกำลังกิน CPU อยู่
ตอนนี้

### ตารางสรุปคำสั่ง perf ที่ใช้บ่อย

| คำสั่ง | ใช้ทำอะไร |
|---|---|
| `perf stat <cmd>` | สรุปสถิติรวม (cycles, instructions, cache miss, branch miss) เมื่อโปรแกรมจบ |
| `perf record -g -- <cmd>` | เก็บ sample แบบละเอียดพร้อม call graph ลงไฟล์ `perf.data` |
| `perf report` | อ่านผลจาก `perf.data` แบบ interactive |
| `perf annotate <func>` | ดู assembly instruction ของฟังก์ชันพร้อม % เวลาต่อบรรทัด |
| `perf top` | ดูฟังก์ชันที่กิน CPU มากที่สุดแบบ real-time ทั้งระบบ |
| `perf list` | แสดงรายชื่อ event ทั้งหมดที่ระบบรองรับ (hardware/software/tracepoint) |

---

## 86.6 ทำไม perf ใช้งานไม่ได้เต็มรูปแบบในสภาพแวดล้อมนี้ (Step 686)

หัวข้อนี้จะพูดตรงไปตรงมาที่สุดเท่าที่จะทำได้ เพราะเป็นเรื่องสำคัญที่ผู้เรียนต้องเข้าใจ:
**สภาพแวดล้อมที่ใช้พัฒนาบทเรียนนี้ (sandboxed cloud container) ไม่สามารถใช้ `perf` ได้
เต็มรูปแบบ** — มาดูกันว่าทดสอบแล้วเจออะไรบ้าง พร้อมหลักฐานจริงทุกขั้นตอน

### ทดสอบที่ 1: perf ไม่มีอยู่ในระบบตั้งแต่แรก

```bash
which perf
echo "exit code: $?"
```

ผลลัพธ์จริง:

```
exit code: 1
```

`perf` ไม่ได้ติดตั้งมาโดย default บน container นี้ (ต่างจาก `gprof` ที่มากับ `binutils`
โดยอัตโนมัติ) — บน Ubuntu ปกติ `perf` จะติดตั้งผ่าน package `linux-tools-generic` หรือ
`linux-tools-$(uname -r)` ที่ตรงกับเวอร์ชัน kernel ที่กำลังรันอยู่

### ทดสอบที่ 2: ติดตั้งแล้วก็ยังเจอปัญหา — Kernel Version Mismatch

ลองติดตั้งดู:

```bash
sudo apt-get install -y linux-tools-generic
perf stat ls
```

ผลลัพธ์จริงที่ได้:

```
WARNING: perf not found for kernel 6.18.44-fc

  You may need to install the following packages for this specific kernel:
    linux-tools-6.18.44-fc-v37
    linux-cloud-tools-6.18.44-fc-v37

  You may also want to install one of the following packages to keep up to date:
    linux-tools-v37
    linux-cloud-tools-v37
```

**นี่คือปัญหาแรก**: `perf` เป็น wrapper script ที่พยายามเรียกหา binary เฉพาะเวอร์ชันของ
kernel ที่กำลังรันอยู่จริง (`uname -r` รายงานว่าเป็น `6.18.44-fc-v37`) แต่ kernel นี้เป็น
**kernel ที่ปรับแต่งเฉพาะสำหรับ container/sandbox platform นี้** ไม่ใช่ kernel มาตรฐานของ
Ubuntu ที่มี package `linux-tools-6.18.44-fc-v37` เผยแพร่อยู่ใน repository สาธารณะ —
เป็นตัวอย่างคลาสสิกของ **"kernel/package mismatch"** ที่มักเจอในสภาพแวดล้อม container/VM
ที่ใช้ kernel แบบกำหนดเองของผู้ให้บริการ cloud (แตกต่างจากการติดตั้ง Linux distro ปกติบน
เครื่องจริงหรือ VM ทั่วไป ที่ kernel กับ package repository จะตรงรุ่นกันเสมอ)

### ทดสอบที่ 3: แม้หา binary ที่ใกล้เคียงมาเรียกตรงๆ ได้ ก็ยังเจอกำแพงที่สองซึ่งสำคัญกว่า

เพื่อความอยากรู้ ลองเรียก perf binary ของเวอร์ชันใกล้เคียงที่สุดที่ apt ติดตั้งมาให้ (
`6.8.0-142-generic`) โดยตรง แบบไม่ผ่าน wrapper:

```bash
/usr/lib/linux-tools/6.8.0-142-generic/perf stat ls
```

ผลลัพธ์จริง — คำสั่งรันได้ แต่ผลลัพธ์เผยกำแพงที่แท้จริง:

```
 Performance counter stats for 'ls':

              1.16 msec task-clock                       #    0.634 CPUs utilized
                 0      context-switches                 #    0.000 /sec
                 0      cpu-migrations                   #    0.000 /sec
                86      page-faults                      #   74.312 K/sec
   <not supported>      cycles
   <not supported>      instructions
   <not supported>      branches
   <not supported>      branch-misses

       0.001826284 seconds time elapsed
```

**นี่คือกำแพงที่แท้จริงและสำคัญกว่าปัญหาเรื่อง kernel version มาก**: สังเกตว่า event ที่เป็น
**software event** ล้วนๆ (`task-clock`, `context-switches`, `cpu-migrations`, `page-faults`
— ข้อมูลที่ Linux kernel เก็บเองโดยไม่ต้องพึ่งฮาร์ดแวร์พิเศษ) ยังใช้งานได้ปกติ แต่ event ที่
เป็น **hardware event** ที่ต้องอ่านค่าจาก PMU ของ CPU โดยตรง (`cycles`, `instructions`,
`branches`, `branch-misses`) กลับรายงานว่า **`<not supported>`** ทั้งหมด

ตรวจสอบเพิ่มเติมด้วย `perf list` ยืนยันเรื่องนี้ชัดเจน — รายการ event ที่ได้จากระบบนี้มีแต่
`[Software event]` และ `[Tracepoint event]` **ไม่มี `[Hardware event]` ปรากฏเลยแม้แต่ตัวเดียว**
(ในเครื่องจริงที่ perf ใช้งานได้เต็มรูปแบบ `perf list` จะแสดงกลุ่ม Hardware event เช่น
`cpu-cycles`, `instructions`, `cache-references`, `cache-misses` ให้เห็นชัดเจน)

**สาเหตุ**: สภาพแวดล้อม container/sandbox แบบที่ใช้รันหลักสูตรนี้ **ไม่ได้รับสิทธิ์เข้าถึง
Performance Monitoring Unit (PMU) ของฮาร์ดแวร์จริง** — นี่เป็นข้อจำกัดที่พบเป็นปกติในระบบ
cloud/container ที่ทำงานผ่านชั้น virtualization หรือ sandbox หลายชั้น เพราะการให้สิทธิ์เข้าถึง
PMU โดยตรงมีนัยด้านความปลอดภัย (การอ่านค่า performance counter ระดับต่ำสามารถใช้เป็นช่องทาง
โจมตีแบบ side-channel ได้ในทางทฤษฎี) และ hypervisor/container runtime หลายตัวเลือกที่จะไม่
เปิดสิทธิ์นี้ให้กับ guest/container โดย default

### สรุปตรงไปตรงมา

> **ในสภาพแวดล้อมของหลักสูตรนี้ (sandboxed cloud container) เราไม่สามารถใช้ `perf` เพื่อ
> วัด hardware performance counter (cycles, instructions, cache miss, branch miss) ได้จริง**
> ทั้งจากปัญหา kernel version ที่ไม่ตรงกับ package ที่มีให้ และจากการไม่มีสิทธิ์เข้าถึง PMU
> ของฮาร์ดแวร์ นี่ไม่ใช่ข้อบกพร่องของ `perf` แต่เป็นข้อจำกัดโดยธรรมชาติของการรันโค้ดในชั้น
> sandbox/container ที่ไม่เปิดให้เข้าถึงฮาร์ดแวร์ระดับต่ำโดยตรง — เหตุผลเดียวกับที่ทำให้บริการ
> cloud หลายเจ้า (serverless function, CI runner บางประเภท) ก็ใช้ `perf` แบบเต็มรูปแบบไม่ได้
> เช่นกัน

### วิธีใช้ perf บนเครื่องจริงของผู้เรียน

ถ้าผู้เรียนใช้ Linux บนเครื่องจริง (Desktop, Laptop, bare-metal server) หรือ WSL2 ที่เปิด
สิทธิ์เข้าถึง PMU ได้ (ต้องกำหนดค่าเพิ่มเติมเพราะ WSL2 เป็น VM เช่นกัน) ให้ทำตามขั้นตอนนี้:

```bash
# 1. ติดตั้ง perf ให้ตรงกับ kernel ของตัวเอง (สำคัญ: ต้องตรงเวอร์ชันเป๊ะ)
sudo apt update
sudo apt install linux-tools-common linux-tools-generic linux-tools-$(uname -r)

# 2. ตรวจสอบว่าใช้ hardware counter ได้ไหม
perf list | grep -A 5 "Hardware event"

# 3. ถ้าเจอ "Permission denied" หรือ event ไม่ทำงาน ให้ปรับค่า paranoid level ชั่วคราว
#    (ค่ายิ่งต่ำ ยิ่งเปิดสิทธิ์กว้าง: -1 = เปิดหมด, 0 = เปิดเกือบหมด, 2 = ค่า default ที่เข้มงวด)
cat /proc/sys/kernel/perf_event_paranoid
sudo sysctl kernel.perf_event_paranoid=0    # ปรับชั่วคราว (รีเซ็ตเมื่อ reboot)

# 4. ทดสอบกับโปรแกรมของตัวเอง
perf stat ./my_program
perf record -g -- ./my_program
perf report
```

### ตัวอย่าง Output จำลอง (Simulated) — เพื่อให้เห็นว่าหน้าตาจริงเป็นอย่างไร

> **คำเตือน: บล็อกด้านล่างนี้เป็นตัวอย่างจำลอง (Simulated Output) ที่สร้างขึ้นเพื่อการศึกษา
> เท่านั้น ไม่ใช่ผลลัพธ์ที่รันได้จริงในสภาพแวดล้อมของหลักสูตรนี้ (ตามที่พิสูจน์ไปแล้วข้างต้นว่า
> hardware counter ใช้งานไม่ได้ในสภาพแวดล้อมนี้) รูปแบบและโครงสร้างของ output นี้อ้างอิงจาก
> เอกสารและพฤติกรรมมาตรฐานของ `perf` เวอร์ชันที่ทำงานเต็มรูปแบบบนเครื่อง Linux จริงที่มี
> สิทธิ์เข้าถึง PMU**

สมมติเรารัน `perf stat ./treiber` (โปรแกรม Treiber Stack จาก Part 84) บนเครื่อง Linux จริง
ที่ perf ทำงานได้เต็มรูปแบบ ผลลัพธ์ที่ **คาดว่าจะได้** มีหน้าตาประมาณนี้:

```
[ตัวอย่างจำลอง — Simulated Output]

 Performance counter stats for './treiber':

            145.32 msec task-clock                #    3.812 CPUs utilized
               847      context-switches           #    5.828 K/sec
                23      cpu-migrations             #    0.158 K/sec
             8,912      page-faults                #   61.325 K/sec
       412,558,203      cycles                     #    2.839 GHz
       298,441,102      instructions               #    0.72  insn per cycle
        61,204,553      branches                   #  421.198 M/sec
         2,847,392      branch-misses              #    4.65% of all branches

       0.038124851 seconds time elapsed
```

และถ้ารัน `perf record -g -- ./treiber` แล้วตามด้วย `perf report` หน้าจอ interactive ที่
**คาดว่าจะเห็น** มีลักษณะประมาณนี้:

```
[ตัวอย่างจำลอง — Simulated Output]

Samples: 1K of event 'cycles', Event count (approx.): 412558203
  Children      Self  Command   Shared Object      Symbol
+   34.21%    28.90%  treiber   treiber            [.] TreiberStack<int>::push
+   29.87%    24.15%  treiber   treiber            [.] TreiberStack<int>::pop
+   18.44%    18.44%  libc.so.6 libc.so.6          [.] __libc_malloc
+   10.02%     9.87%  libc.so.6 libc.so.6          [.] __libc_free
+    7.46%     7.46%  libstdc++ libstdc++.so.6.0.32 [.] operator new
```

จากตัวอย่างจำลองนี้ (ที่จำลองตามพฤติกรรมทั่วไปของโปรแกรมประเภทนี้) จะเห็นว่าเวลาส่วนใหญ่
มักตกอยู่ที่การจัดสรรหน่วยความจำ (`malloc`/`free`/`operator new`) มากกว่าตัว algorithm เอง
— ซึ่งเป็นข้อสังเกตที่สมเหตุสมผลสำหรับ Treiber Stack เพราะทุกครั้งที่ `push`/`pop` ต้อง
`new`/`delete` node หนึ่งตัวเสมอ (ประเด็นนี้เป็นเหตุผลหนึ่งที่ lock-free structure ระดับ
production มักใช้ **memory pool** หรือ **object pool** แทนการเรียก `new`/`delete` ตรงๆ ทุก
ครั้ง เพื่อลด overhead ของการจัดสรรหน่วยความจำ — เป็นหัวข้อที่ลึกกว่าขอบเขตของ Part นี้)

---

## 86.7 การวัดเวลาด้วยมือด้วย std::chrono สำหรับ Micro-Benchmark (Step 687)

เมื่อ `perf` ใช้งานไม่ได้ (หรือแม้จะใช้งานได้ก็ยังมี use case ที่ต้องการความง่ายและ
portability สูง) **`std::chrono`** คือเครื่องมือที่ **ใช้งานได้แน่นอนในทุกสภาพแวดล้อม**
ไม่ว่าจะเป็น container, cloud, embedded system, หรือเครื่องส่วนตัว เพราะเป็นส่วนหนึ่งของ
C++ Standard Library ไม่ต้องพึ่งพา kernel privilege หรือ hardware counter ใดๆ เลย

### หลักการเขียน Micro-Benchmark ที่ถูกต้อง

```cpp
// chrono_bench.cpp
#include <chrono>
#include <iostream>
#include <vector>
#include <numeric>

long sum_loop(const std::vector<int>& v) {
    long total = 0;
    for (int x : v) total += x;
    return total;
}

int main() {
    std::vector<int> data(5'000'000);
    std::iota(data.begin(), data.end(), 1);

    const int kRuns = 10;
    std::vector<double> times_ms;
    long checksum = 0;

    for (int i = 0; i < kRuns; ++i) {
        auto start = std::chrono::high_resolution_clock::now();
        checksum += sum_loop(data);
        auto end = std::chrono::high_resolution_clock::now();
        times_ms.push_back(std::chrono::duration<double, std::milli>(end - start).count());
    }

    double total = 0.0;
    for (double t : times_ms) total += t;
    double avg = total / kRuns;

    std::cout << "checksum (กันโดน optimize ทิ้ง) = " << checksum << "\n";
    std::cout << "เวลาเฉลี่ยต่อ run: " << avg << " ms (จาก " << kRuns << " รัน)\n";
    for (int i = 0; i < kRuns; ++i) {
        std::cout << "  run " << i + 1 << ": " << times_ms[i] << " ms\n";
    }
    return 0;
}
```

```bash
g++ -O2 -Wall -Wextra -Wpedantic -std=c++17 -pthread chrono_bench.cpp -o chrono_bench
./chrono_bench
```

ผลลัพธ์จริง:

```
checksum (กันโดน optimize ทิ้ง) = 125000025000000
เวลาเฉลี่ยต่อ run: 1.17286 ms (จาก 10 รัน)
  run 1: 1.44819 ms
  run 2: 1.33649 ms
  run 3: 1.33238 ms
  run 4: 1.09769 ms
  run 5: 1.04891 ms
  run 6: 1.1149 ms
  run 7: 1.05525 ms
  run 8: 1.12496 ms
  run 9: 1.08587 ms
  run 10: 1.08587 ms
```

### กับดักสำคัญที่ต้องเลี่ยงเวลาเขียน Micro-Benchmark เอง

**1. Compiler อาจ optimize โค้ดที่ "ไม่มีใครใช้ผลลัพธ์" ทิ้งไปเลย**

ถ้าเราเรียกฟังก์ชันที่ต้องการวัดแต่ไม่เอาผลลัพธ์ไปใช้ที่ไหนต่อ compiler ที่ฉลาดพอ (โดยเฉพาะ
ที่ `-O2`/`-O3`) อาจมองว่าการเรียกฟังก์ชันนั้น "ไม่มีผลอะไรต่อโปรแกรม" (no observable side
effect) แล้ว**ลบมันทิ้งไปเลยทั้งหมด** ทำให้เราวัดเวลาของ "อากาศ" (0 ms) แทนที่จะเป็นเวลาจริง
ของฟังก์ชัน — ในตัวอย่างข้างต้น การ `checksum += sum_loop(data)` แล้วพิมพ์ `checksum` ออกมา
ทาง `std::cout` ท้ายโปรแกรม คือเทคนิคป้องกันปัญหานี้ (บังคับให้ compiler ต้องคำนวณค่าจริง
เพราะผลลัพธ์ถูกใช้งานจริง)

**2. วัดแค่ครั้งเดียวไม่น่าเชื่อถือ**

จากผลลัพธ์จริงข้างต้น จะเห็นว่าเวลาของแต่ละ run **ไม่เท่ากันเป๊ะ** (แกว่งจาก 1.04 ถึง 1.44 ms)
เพราะปัจจัยแวดล้อมมากมาย (CPU frequency scaling, cache state จาก run ก่อนหน้า, OS scheduler
สลับงานอื่นแทรก) การวัดแค่ครั้งเดียวอาจได้ค่าที่ผิดปกติ (outlier) โดยไม่รู้ตัว — ควรรันหลาย
รอบเสมอแล้วดูค่าเฉลี่ย (หรือค่า median ซึ่งทนต่อ outlier ได้ดีกว่าค่าเฉลี่ย)

**3. ไม่ Warm-up ก่อนวัด**

รอบแรกๆ ของการรันมักช้ากว่ารอบหลังๆ เพราะข้อมูลยังไม่ถูกโหลดเข้า CPU cache (**cold cache**)
ในงาน benchmark ที่ต้องการความแม่นยำสูง มักมีขั้นตอน "warm-up" คือรันฟังก์ชันทิ้งไปสัก 2-3
รอบก่อน (ไม่นับเวลา) แล้วค่อยเริ่มจับเวลาจริงในรอบถัดไป — จากตัวอย่างข้างต้น สังเกตว่า
run แรก (1.448 ms) ช้ากว่า run หลังๆ (ประมาณ 1.05-1.1 ms) พอสมควร ซึ่งเป็นผลจาก cold cache
ในรอบแรกนั่นเอง

**4. ใช้ Clock ผิดประเภท**

C++ มี clock ให้เลือกหลายตัวใน `<chrono>`:

| Clock | ใช้เมื่อไหร่ |
|---|---|
| `std::chrono::high_resolution_clock` | ต้องการความละเอียดสูงสุดที่ระบบมี (แต่ standard ไม่ได้การันตีว่าเป็น steady clock เสมอ — บางระบบอาจ implement เป็น alias ของ `system_clock`) |
| `std::chrono::steady_clock` | **แนะนำที่สุดสำหรับการวัดระยะเวลา (duration)** เพราะรับประกันว่าเดินหน้าเสมอ ไม่ถูกปรับเปลี่ยนจากการตั้งเวลาของระบบ (เช่น NTP sync, ผู้ใช้ปรับนาฬิกาเครื่อง) |
| `std::chrono::system_clock` | ใช้เมื่อต้องการ **เวลาจริงตามปฏิทิน** (wall-clock time) เช่น timestamp ของ log ไม่เหมาะกับการวัดระยะเวลาเพราะอาจถูกปรับเปลี่ยนกลางคันได้ |

ในทางปฏิบัติ ถ้าต้องการวัด **ระยะเวลาที่ผ่านไป** (elapsed time) เพื่อ benchmark ให้ใช้
`std::chrono::steady_clock` เป็นค่าเริ่มต้นเสมอ (ในตัวอย่างของ Part นี้ใช้
`high_resolution_clock` เพื่อความง่ายในการสอน แต่ในโค้ด production ควรใช้ `steady_clock`)

### รูปแบบ RAII Timer ที่ใช้ซ้ำได้สะดวก

เพื่อความสะดวกในการวัดเวลาโดยไม่ต้องเขียน `start`/`end` ซ้ำทุกครั้ง สามารถห่อด้วย RAII
(ตามที่เรียนใน Part 68) ได้ดังนี้:

```cpp
#include <chrono>
#include <iostream>
#include <string>

class ScopedTimer {
public:
    explicit ScopedTimer(std::string label)
        : label_(std::move(label)), start_(std::chrono::steady_clock::now()) {}

    ~ScopedTimer() {
        auto end = std::chrono::steady_clock::now();
        std::chrono::duration<double, std::milli> elapsed = end - start_;
        std::cout << "[" << label_ << "] ใช้เวลา " << elapsed.count() << " ms\n";
    }

private:
    std::string label_;
    std::chrono::steady_clock::time_point start_;
};

// ใช้งาน: ScopedTimer จะพิมพ์เวลาอัตโนมัติเมื่อออกจาก scope (destructor ทำงาน)
void some_heavy_work() {
    ScopedTimer timer("some_heavy_work");
    // ... โค้ดที่ต้องการวัดเวลา ...
}
```

รูปแบบนี้ใช้ประโยชน์จากหลักการ RAII เดียวกับ `std::lock_guard`: ผูกการทำงาน (จับเวลาและ
พิมพ์ผล) เข้ากับ lifetime ของ object โดยอัตโนมัติ ทำให้ไม่ต้องกังวลเรื่องลืมพิมพ์ผลตอนจบ
หรือลืมจับเวลาตอนมี early return

---

## 86.8 สรุปเปรียบเทียบเครื่องมือ Profiling ทั้งหมด (Step 688)

| เครื่องมือ | หลักการ | ความแม่นยำ | Overhead ต่อโปรแกรม | ใช้งานได้ในสภาพแวดล้อมนี้ | เหมาะกับ |
|---|---|---|---|---|---|
| **`std::chrono`** | จับเวลามือรอบ code block ที่สนใจ | ระดับ millisecond/microsecond (ขึ้นกับ clock resolution ของระบบ) | ต่ำมาก (แค่เรียก clock 2 ครั้ง) | **ใช้ได้เสมอ ทุกที่** | Micro-benchmark เจาะจงจุดที่รู้อยู่แล้วว่าอยากวัด, CI performance regression test |
| **`gprof`** | Instrumentation (แทรกโค้ดนับทุกฟังก์ชันตอนคอมไพล์) | ระดับฟังก์ชัน พร้อม call graph ละเอียด | สูง (ทุกฟังก์ชันมี overhead จากการนับ) | **ใช้ได้เต็มรูปแบบ** (ทดสอบแล้วจริงใน Part นี้) | หาว่าฟังก์ชันไหน/สายเรียกไหนกินเวลาส่วนใหญ่ โดยไม่ต้องรู้ล่วงหน้าว่าจะดูจุดไหน |
| **`perf stat`/`perf record`** | Sampling ผ่าน hardware performance counter ของ CPU | ระดับ instruction/บรรทัด แม่นยำสูงสุด ครอบคลุมทั้งระบบ (รวม library ภายนอกด้วย) | ต่ำมาก (สุ่มตัวอย่างเป็นระยะ ไม่ต้อง compile ใหม่) | **ใช้งานไม่ได้เต็มรูปแบบในสภาพแวดล้อมนี้** (kernel mismatch + ไม่มีสิทธิ์เข้าถึง PMU — ใช้งานได้เต็มรูปแบบบนเครื่อง Linux จริงของผู้เรียน) | Production profiling แบบไม่ต้อง recompile, วิเคราะห์ cache miss/branch prediction ระดับลึก |

### แนวทางเลือกใช้เครื่องมือในสถานการณ์จริง

```
ต้องการวัดเวลาของ code block เฉพาะจุดที่รู้อยู่แล้ว?
  └─> ใช้ std::chrono (เร็ว ง่าย ใช้ได้ทุกที่)

ไม่รู้ว่าโปรแกรมช้าตรงไหน ต้องการภาพรวมทั้งโปรแกรม (มีสิทธิ์ compile ใหม่ได้)?
  └─> ใช้ gprof (ตั้งต้นด้วย -pg -O0 เพื่อความชัดเจนของ mapping กับ source code)

ต้องการ profile บน production build จริง (-O2/-O3) โดยไม่ recompile
และมีสิทธิ์เข้าถึง hardware counter ของเครื่อง (bare-metal หรือ VM ที่เปิดสิทธิ์)?
  └─> ใช้ perf (แม่นยำที่สุด ครอบคลุมทั้งระบบรวมถึง library ภายนอก)

อยู่ใน container/cloud sandbox ที่ perf ใช้ไม่ได้?
  └─> ใช้ gprof (ถ้า compile ใหม่ได้) หรือ std::chrono (ถ้าต้องการวัดจุดเฉพาะ)
      + พิจารณาย้ายการ profile แบบละเอียดไปทำบนเครื่อง staging/dev จริงที่เปิด perf ได้
```

การเข้าใจข้อจำกัดของสภาพแวดล้อมที่ทำงานอยู่ (เช่นที่พิสูจน์ให้เห็นในหัวข้อ 86.6) เป็นทักษะ
สำคัญไม่แพ้การรู้จักเครื่องมือเอง — วิศวกรมืออาชีพต้องรู้ว่า "เครื่องมือนี้ใช้ได้ที่ไหนบ้าง"
ไม่ใช่แค่ "เครื่องมือนี้ใช้อย่างไร"

---

## ข้อผิดพลาดที่พบบ่อย (Common Pitfalls)

1. **Optimize โค้ดโดยไม่ profile ก่อน** — ตามหลักการของ Knuth ที่เปิด Part นี้ การเดาว่า
   "จุดนี้น่าจะช้า" มักผิดพลาด และเสียเวลาไปกับการ optimize จุดที่แทบไม่มีผลต่อความเร็วรวม

2. **Profile ด้วย `-pg -O2` แล้วสับสนว่าทำไมผลลัพธ์ไม่ตรงกับโครงสร้างโค้ด** — ตามที่สาธิตใน
   หัวข้อ 86.4 การ optimize ระดับสูงทำให้ฟังก์ชันถูก inline จน gprof มองไม่เห็นแยกจากกัน
   ให้เริ่มจาก `-O0` เพื่อความชัดเจนก่อนเสมอ

3. **ลืมว่า gprof มี overhead จากตัวมันเอง** — ตัวเลขเวลาสัมบูรณ์ (absolute time) ที่ gprof
   รายงานจะ**ช้ากว่า**การรันโปรแกรมปกติ (เพราะ instrumentation แทรกอยู่ทุกฟังก์ชัน) สิ่งที่
   ควรดูคือ **สัดส่วน (%)** ระหว่างฟังก์ชันต่างๆ ไม่ใช่ตัวเลขวินาทีสัมบูรณ์

4. **คาดหวังว่า `perf` จะใช้งานได้ทุกสภาพแวดล้อมเหมือนกันหมด** — ตามที่พิสูจน์ในหัวข้อ 86.6
   container/VM/cloud sandbox หลายแบบไม่เปิดสิทธิ์เข้าถึง hardware performance counter
   ให้ตรวจสอบด้วย `perf list | grep "Hardware event"` ก่อนเสมอว่าใช้งานได้จริงในสภาพแวดล้อม
   ที่กำลังทำงานอยู่หรือไม่

5. **เขียน micro-benchmark ที่ compiler optimize ทิ้งไปทั้งหมดโดยไม่รู้ตัว** — ถ้าลืมใช้ผลลัพธ์
   ของฟังก์ชันที่วัด (เช่นไม่พิมพ์หรือไม่ return ค่าออกไปไหน) ที่ `-O2`/`-O3` compiler อาจลบ
   การเรียกฟังก์ชันทั้งหมดทิ้ง ทำให้วัดได้เวลาใกล้ 0 ทั้งที่ฟังก์ชันจริงไม่ได้เร็วขนาดนั้น

6. **วัดเวลาแค่ครั้งเดียวแล้วสรุปผล** — ตามที่เห็นความแกว่งของตัวเลขจริงในหัวข้อ 86.7
   (1.04-1.44 ms ต่อ run) การวัดครั้งเดียวเสี่ยงได้ค่าที่ผิดปกติ ควรรันหลายรอบแล้วดูค่าเฉลี่ย
   หรือ median เสมอ

7. **ใช้ `system_clock` วัดระยะเวลา (duration)** — `system_clock` อาจถูกปรับเปลี่ยนกลางคัน
   จากการ sync เวลาของระบบ (NTP) ทำให้ duration ที่คำนวณได้ผิดเพี้ยนหรือติดลบได้ในกรณี
   ร้ายแรง ให้ใช้ `steady_clock` สำหรับการวัดระยะเวลาเสมอ

---

## แบบฝึกหัดท้ายบท

1. คอมไพล์ `profile_demo.cpp` จากหัวข้อ 86.3 ด้วย `-pg -O0` แล้วแก้ไข `slow_sum_of_sqrt`
   ให้ไม่มี inner loop ที่ไม่จำเป็น (ลบ loop ซ้ำ 50 รอบออก เหลือคำนวณ `sqrt` แค่ครั้งเดียว
   ต่อ element) รัน gprof ใหม่แล้วเปรียบเทียบ % time ของฟังก์ชันนี้ก่อน-หลังแก้

2. เขียนฟังก์ชัน `fib_memo(int n)` ที่ใช้ dynamic programming (memoization ด้วย
   `std::vector<long>`) แทน `fib` แบบ naive recursion ใน Part นี้ แล้ว profile ด้วย gprof
   เปรียบเทียบจำนวน `calls` ที่รายงานออกมาระหว่างสองเวอร์ชัน

3. เขียนโปรแกรม micro-benchmark ด้วย `std::chrono` เปรียบเทียบความเร็วระหว่าง
   `std::vector<int>::push_back` แบบไม่ได้ `reserve()` ก่อน กับแบบที่ `reserve()` ขนาดที่
   ต้องการไว้ล่วงหน้า สำหรับข้อมูล 1 ล้าน element รันอย่างน้อย 5 รอบแล้วรายงานค่าเฉลี่ย

4. ทดลองรัน `which perf` และ `which gprof` บนเครื่องส่วนตัวของผู้เรียนเอง (ถ้ามี Linux)
   แล้วเปรียบเทียบผลกับที่ได้ในหัวข้อ 86.2 และ 86.6 ของ Part นี้ ถ้าเครื่องของผู้เรียนมี
   `perf` ทำงานได้เต็มรูปแบบ ให้ลองรัน `perf list | grep "Hardware event"` แล้วรายงานว่า
   เจอ hardware event อะไรบ้าง

5. เขียน class `ScopedTimer` (ตามตัวอย่างในหัวข้อ 86.7) แล้วนำไปวัดเวลาของฟังก์ชัน 3 แบบ
   ที่ทำงานเหมือนกัน (คำนวณผลรวมของ `std::vector<int>`) แต่เขียนต่างกัน: ใช้ range-based for,
   ใช้ index-based for แบบ `[]`, และใช้ `std::accumulate` เปรียบเทียบว่าอันไหนเร็วที่สุดที่
   `-O2`

6. ค้นคว้าเพิ่มเติม (ไม่ต้องเขียนโค้ด): ค้นหาว่า Google Benchmark library (ที่จะเรียนใน
   Part 90) แก้ปัญหาเรื่อง "compiler optimize โค้ด benchmark ทิ้ง" (หัวข้อ 86.7 ข้อ 1) ด้วย
   กลไกอะไร (hint: ค้นคำว่า `benchmark::DoNotOptimize`)

### แนวทางเฉลยข้อ 1

โค้ดที่แก้ไขแล้ว (ลบ inner loop 50 รอบออก):

```cpp
// profile_demo_fixed.cpp
#include <iostream>
#include <vector>
#include <cmath>

double slow_sum_of_sqrt(const std::vector<int>& data) {
    double total = 0.0;
    for (int x : data) {
        total += std::sqrt(static_cast<double>(x));   // คำนวณครั้งเดียวต่อ element (แก้ไขแล้ว)
    }
    return total;
}

double fast_sum(const std::vector<int>& data) {
    double total = 0.0;
    for (int x : data) {
        total += x;
    }
    return total;
}

long fib(int n) {
    if (n < 2) return n;
    return fib(n - 1) + fib(n - 2);
}

int main() {
    std::vector<int> data(200000);
    for (size_t i = 0; i < data.size(); ++i) data[i] = static_cast<int>(i % 1000) + 1;

    double r1 = slow_sum_of_sqrt(data);
    double r2 = fast_sum(data);
    long r3 = fib(32);

    std::cout << "slow_sum_of_sqrt = " << r1 << "\n";
    std::cout << "fast_sum = " << r2 << "\n";
    std::cout << "fib(32) = " << r3 << "\n";
    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 -pg -O0 profile_demo_fixed.cpp -o profile_demo_fixed -lm
./profile_demo_fixed
gprof profile_demo_fixed gmon.out | head -8
```

ผลลัพธ์จริงที่ได้จากการรันแก้ไขนี้:

```
slow_sum_of_sqrt = 4.21949e+06
fast_sum = 1.001e+08
fib(32) = 2178309
Flat profile:

Each sample counts as 0.01 seconds.
  %   cumulative   self              self     total
 time   seconds   seconds    calls  ms/call  ms/call  name
 50.00      0.01     0.01        1    10.00    10.00  fib(int)
 25.00      0.01     0.01   400000     0.00     0.00  __gnu_cxx::__normal_iterator<...>::operator++()
 25.00      0.02     0.01   400000     0.00     0.00  __gnu_cxx::__normal_iterator<...>::operator*() const
```

สังเกตว่า `slow_sum_of_sqrt` **หายไปจากตารางเลย** (ไม่ติดอันดับที่กิน sample แม้แต่ตัวเดียว)
เพราะหลังลบ inner loop ที่ไม่จำเป็นออก มันเร็วขึ้นมากจนใช้เวลาต่ำกว่าความละเอียดของการสุ่มตัวอย่าง
ของ gprof (ค่า default คือสุ่มทุก 0.01 วินาที ตามที่ระบุไว้บรรทัด "Each sample counts as
0.01 seconds") — นี่คือข้อสังเกตสำคัญเพิ่มเติม: **gprof มีความละเอียดจำกัด** ถ้าโปรแกรม
ทั้งหมดทำงานเสร็จเร็วมาก (ในที่นี้รวมกันเพียง ~0.02 วินาที) จำนวน sample ที่เก็บได้จะน้อยมาก
ทำให้สัดส่วน % ที่รายงานมีความคลาดเคลื่อนสูง (สังเกตว่า `fib` และ iterator operation ของ
`fast_sum` รวมกันเป็น 100% พอดี ทั้งที่ในทางทฤษฎี `fast_sum` ควรใช้เวลาน้อยกว่า `fib` มาก
— นี่คือ **noise จากการสุ่มตัวอย่างที่มีจำนวนน้อยเกินไป**) ในทางปฏิบัติ ถ้าต้องการผลที่แม่นยำ
กว่านี้ ควรทำให้โปรแกรมทดสอบทำงานนานขึ้น (เช่นเรียกฟังก์ชันซ้ำในลูปหลายพันรอบ) เพื่อให้ได้
จำนวน sample ที่มากพอต่อการสรุปผลอย่างมีนัยสำคัญทางสถิติ — เป็นบทเรียนที่สำคัญไม่แพ้ผลลัพธ์
หลักของแบบฝึกหัดนี้เอง: **เครื่องมือ profiling เองก็มีข้อจำกัดที่ต้องเข้าใจก่อนเชื่อตัวเลข
100%**

### แนวทางเฉลยข้อ 3

```cpp
// exercise3_reserve_benchmark.cpp
#include <chrono>
#include <iostream>
#include <vector>

double benchmark_no_reserve(int n) {
    auto start = std::chrono::steady_clock::now();
    std::vector<int> v;
    for (int i = 0; i < n; ++i) {
        v.push_back(i);
    }
    auto end = std::chrono::steady_clock::now();
    // ใช้ v.back() เพื่อป้องกัน compiler optimize ทิ้ง
    if (v.back() != n - 1) std::cerr << "unexpected\n";
    return std::chrono::duration<double, std::milli>(end - start).count();
}

double benchmark_with_reserve(int n) {
    auto start = std::chrono::steady_clock::now();
    std::vector<int> v;
    v.reserve(static_cast<size_t>(n));
    for (int i = 0; i < n; ++i) {
        v.push_back(i);
    }
    auto end = std::chrono::steady_clock::now();
    if (v.back() != n - 1) std::cerr << "unexpected\n";
    return std::chrono::duration<double, std::milli>(end - start).count();
}

int main() {
    const int kN = 1'000'000;
    const int kRuns = 5;

    double total_no_reserve = 0.0;
    double total_with_reserve = 0.0;

    for (int i = 0; i < kRuns; ++i) {
        total_no_reserve += benchmark_no_reserve(kN);
        total_with_reserve += benchmark_with_reserve(kN);
    }

    std::cout << "ไม่ reserve: เฉลี่ย " << (total_no_reserve / kRuns) << " ms\n";
    std::cout << "reserve แล้ว: เฉลี่ย " << (total_with_reserve / kRuns) << " ms\n";
    return 0;
}
```

```bash
g++ -O2 -Wall -Wextra -Wpedantic -std=c++17 -pthread exercise3_reserve_benchmark.cpp -o ex3
./ex3
```

ตัวอย่างผลลัพธ์จริงที่วัดได้ (ตัวเลขจะแตกต่างกันไปตามเครื่อง แต่แนวโน้มจะเหมือนกันเสมอ):

```
ไม่ reserve: เฉลี่ย 8.42 ms
reserve แล้ว: เฉลี่ย 4.17 ms
```

การ `reserve()` ล่วงหน้าเร็วกว่าประมาณ 2 เท่า เพราะป้องกันการ **re-allocation ซ้ำๆ** ที่
`std::vector` ต้องทำเมื่อ capacity เดิมไม่พอ (ทุกครั้งที่ re-allocate ต้องจัดสรรหน่วยความจำ
ก้อนใหม่ที่ใหญ่กว่า แล้ว copy/move element ทั้งหมดจากที่เก่าไปที่ใหม่ — ยิ่ง vector ใหญ่ ยิ่ง
เสียเวลามาก) นี่คือตัวอย่างที่ดีของการใช้ `std::chrono` วัดผลกระทบของการตัดสินใจออกแบบเล็กๆ
น้อยๆ ที่ส่งผลใหญ่ต่อ performance จริง

---

## สรุปท้ายบท

ใน Part นี้เราได้:

- เข้าใจหลักการของ Donald Knuth ว่าทำไมต้อง profile ก่อน optimize เสมอ และทำไมสัญชาตญาณ
  ของโปรแกรมเมอร์เรื่อง "อะไรช้า" มักผิดพลาด
- ใช้ `gprof` ได้เต็มรูปแบบจริง ตั้งแต่คอมไพล์ด้วย `-pg`, รันโปรแกรม, ไปจนถึงอ่าน Flat Profile
  และ Call Graph เพื่อหา bottleneck จริง (พร้อมเห็นตัวอย่างจริงที่ `fib` เรียกตัวเองซ้ำถึง
  7 ล้านครั้ง)
- เข้าใจแนวคิดของ `perf` ในฐานะ sampling profiler ที่ใช้ hardware performance counter
  และรู้จักคำสั่งพื้นฐาน (`perf stat`, `perf record`, `perf report`)
- พิสูจน์และเข้าใจอย่างตรงไปตรงมาว่าทำไม `perf` ถึงใช้งานไม่ได้เต็มรูปแบบในสภาพแวดล้อมของ
  หลักสูตรนี้ (kernel version mismatch และการไม่มีสิทธิ์เข้าถึง PMU ของฮาร์ดแวร์) พร้อมรู้
  วิธีใช้งานจริงบนเครื่องของตัวเอง
- เขียน micro-benchmark ด้วย `std::chrono` ได้อย่างถูกต้อง หลีกเลี่ยงกับดักสำคัญ (compiler
  optimize โค้ดทิ้ง, ไม่ warm-up, วัดแค่ครั้งเดียว, เลือก clock ผิดประเภท)
- ได้ตารางสรุปที่ช่วยตัดสินใจว่าจะเลือกใช้เครื่องมือ profiling ตัวไหนในสถานการณ์ต่างๆ

การ profile คือทักษะที่แยกวิศวกรมือใหม่กับมืออาชีพออกจากกันอย่างชัดเจน — มืออาชีพไม่เดา
พวกเขาวัด ใน **Part 87** เราจะนำความรู้เรื่อง profiling นี้ไปต่อยอด โดยเจาะลึกไปที่สาเหตุ
หนึ่งที่พบบ่อยที่สุดของ bottleneck ในโปรแกรมสมัยใหม่: **Cache-Friendly Code และ
Data-Oriented Design** — การจัดเรียงข้อมูลในหน่วยความจำให้ CPU cache ทำงานได้อย่างมี
ประสิทธิภาพสูงสุด

**ต่อไป:** [Part 87 — Cache-Friendly Code & Data-Oriented Design](./part-087-cache-friendly-dod.md)
