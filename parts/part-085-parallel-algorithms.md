# Part 85: Parallel Algorithm (C++17 Execution Policy) (Step 673–680)

> Module G — Concurrency และ Performance Engineering | Part 85 จาก 125
> Step ที่ครอบคลุมใน Part นี้: Step 673–680
> Part ก่อนหน้า: [Part 84 — Lock-Free Programming เบื้องต้น](./part-084-lock-free.md) | Part ถัดไป: [Part 86 — Performance Profiling (perf/gprof)](./part-086-profiling-perf.md)

## เป้าหมายการเรียนรู้ (Learning Objectives)

หลังจบ Part นี้ ผู้เรียนจะสามารถ:

1. อธิบายได้ว่า **Execution Policy** ของ C++17 คืออะไร และทำไมมันถึงเป็นวิธีที่ "ง่ายที่สุด"
   ในการใช้ประโยชน์จาก CPU หลาย core โดยไม่ต้องเขียน thread เอง
2. แยกความแตกต่างระหว่าง `std::execution::seq`, `std::execution::par`, และ
   `std::execution::par_unseq` ได้อย่างถูกต้อง
3. เปลี่ยน `std::sort`, `std::transform`, `std::for_each` ธรรมดาให้ทำงานแบบขนานได้เพียงเพิ่ม
   parameter ตัวเดียว
4. วัด speedup จริงระหว่าง sequential กับ parallel execution บน array ขนาดใหญ่ด้วย
   `std::chrono` ได้ด้วยตัวเอง
5. รู้จักและป้องกัน **data race** ที่เกิดจาก lambda ที่แก้ไข shared state ที่ไม่ใช่ atomic
   ภายใน parallel algorithm
6. เข้าใจว่าทำไมบาง distro/compiler ต้อง link `-ltbb` เพื่อให้ execution policy ทำงานแบบ
   ขนานได้จริง และรู้วิธีตรวจสอบว่าระบบของตัวเองต้องการหรือไม่
7. ประเมินได้ว่าเมื่อไหร่ parallel algorithm "คุ้มค่า" จริง และเมื่อไหร่มันกลับทำให้ช้าลง

---

## 85.1 Execution Policy คืออะไร ทำไม C++17 ถึงเพิ่มเข้ามา (Step 673)

ก่อน C++17 ถ้าอยากให้การ sort หรือ transform ข้อมูลขนาดใหญ่ทำงานแบบขนานบนหลาย core เราต้อง
เขียน `std::thread` เอง แบ่งข้อมูลเป็นส่วนๆ (partition) ด้วยตัวเอง จัดการ synchronization เอง
และรวมผลลัพธ์เอง — เป็นงานที่ใช้เวลาและเสี่ยงต่อบั๊กมาก (ตามที่เห็นความซับซ้อนของ concurrency
มาตลอด Module G)

**C++17 แก้ปัญหานี้อย่างสง่างาม**: เพิ่ม **Execution Policy** เป็น parameter ตัวแรกของ STL
algorithm กว่า 69 ตัว (`std::sort`, `std::for_each`, `std::transform`, `std::reduce`,
`std::find`, `std::copy` ฯลฯ) ทำให้เราสามารถบอกให้ algorithm ทำงานแบบขนานได้ **โดยไม่ต้องเขียน
โค้ด thread จัดการเองเลยแม้แต่บรรทัดเดียว** — compiler และ standard library implementation
จะจัดการแบ่งงานและสร้าง thread ให้เราเบื้องหลังทั้งหมด

เปิดใช้งานโดย `#include <execution>` แล้วส่ง policy เป็น argument แรกของ algorithm:

```cpp
#include <algorithm>
#include <execution>
#include <vector>

std::vector<int> data = /* ... */;

std::sort(data.begin(), data.end());                          // เดิม: sequential เท่านั้น
std::sort(std::execution::par, data.begin(), data.end());     // ใหม่: อาจทำงานแบบขนาน
```

เพียงเพิ่ม `std::execution::par,` เข้าไปเป็น argument แรก โค้ดที่เหลือ**เหมือนเดิมทุกประการ**
— นี่คือเสน่ห์ของ execution policy: มันไม่เปลี่ยน signature หรือความหมายของ algorithm เลย
เปลี่ยนแค่ "วิธีการทำงานภายใน" เท่านั้น

### แนวคิดเบื้องหลัง: Amdahl's Law

เหตุผลที่ต้องมี parallel algorithm มาจากข้อเท็จจริงที่ว่า CPU สมัยใหม่ไม่ได้เร็วขึ้นด้วยการเพิ่ม
clock speed อีกต่อไป (ชนกำแพงฟิสิกส์ด้านความร้อนมาตั้งแต่กลางยุค 2000s) แต่เพิ่มจำนวน **core**
แทน — เครื่องทั่วไปในปี 2026 มีตั้งแต่ 4 ไปจนถึง 16+ core แต่โปรแกรมที่เขียนแบบ single-thread
จะใช้ประโยชน์จากได้แค่ **core เดียว** เท่านั้น ไม่ว่าเครื่องจะมีกี่ core ก็ตาม

**Amdahl's Law** สรุปเพดานของ speedup ที่เป็นไปได้จากการขนานงาน:

```
Speedup(N cores) = 1 / ( (1 - P) + P/N )

โดย P = สัดส่วนของโปรแกรมที่ขนานได้ (parallelizable portion)
     N = จำนวน core
```

ตัวอย่าง: ถ้าโปรแกรม 90% ขนานได้ (`P = 0.9`) บนเครื่อง 4 core:

```
Speedup = 1 / (0.1 + 0.9/4) = 1 / 0.325 ≈ 3.08 เท่า
```

จะเห็นว่า**ไม่มีวันได้ speedup เต็ม 4 เท่า** แม้จะมี 4 core เพราะยังมีส่วน 10% ที่ทำงานแบบ
sequential เสมอ (เช่น การอ่านผลลัพธ์รวม, การจัดสรรหน่วยความจำเริ่มต้น) — นี่คือกรอบความคิดที่
ต้องมีติดตัวเสมอเวลาประเมินว่าการขนานงานจะช่วยได้มากแค่ไหนจริงๆ

---

## 85.2 std::execution::seq / par / par_unseq ความแตกต่าง (Step 674)

C++17 กำหนด policy tag ไว้ 3 ตัว (C++20 เพิ่มตัวที่ 4 คือ `unseq` สำหรับ vectorization
อย่างเดียวโดยไม่ขนาน thread) ทั้งหมดอยู่ใน namespace `std::execution`:

| Policy | ความหมาย | รับประกันอะไรบ้าง |
|---|---|---|
| `std::execution::seq` | บังคับให้ทำงานแบบ **sequential** (เหมือนไม่ใส่ policy เลย) | ลำดับการทำงานเป็นไปตามลำดับ element แน่นอน 100% |
| `std::execution::par` | อนุญาตให้ library ทำงานแบบ **ขนานหลาย thread** ได้ | ลำดับการทำงานของแต่ละ element **ไม่รับประกัน** แต่แต่ละ element ยังถูกประมวลผลแบบ "ทีละขั้นตอนติดกัน" ภายใน thread ของมัน (ไม่มีการสลับ instruction ระหว่างกลางของ element เดียวกัน) |
| `std::execution::par_unseq` | อนุญาตทั้งขนานหลาย thread **และ** ให้ compiler ทำ **vectorization** (SIMD) ผสมกันได้ | ยืดหยุ่นที่สุด แต่ต้องการให้โค้ดใน lambda/functor "ปลอดภัยต่อการสลับสับเปลี่ยนลำดับใดๆ ก็ได้" อย่างเข้มงวดกว่า `par` มาก (ห้าม lock, ห้าม allocate หน่วยความจำ, ห้ามเรียกฟังก์ชันที่ทำ synchronization ใดๆ ข้างใน) |

### ตัวอย่างการใช้ทั้ง 3 policy

```cpp
#include <algorithm>
#include <execution>
#include <vector>
#include <numeric>
#include <iostream>

int main() {
    std::vector<int> a(10), b(10), c(10);
    std::iota(a.begin(), a.end(), 1);   // 1..10
    std::iota(b.begin(), b.end(), 1);

    // seq: รับประกันลำดับ
    std::transform(std::execution::seq, a.begin(), a.end(), b.begin(), c.begin(),
                    [](int x, int y) { return x + y; });

    // par: อาจขนาน แต่แต่ละ element ยังทำงานครบถ้วนใน thread เดียว
    std::transform(std::execution::par, a.begin(), a.end(), b.begin(), c.begin(),
                    [](int x, int y) { return x + y; });

    // par_unseq: อาจขนาน + vectorize พร้อมกัน (ต้องเขียน lambda ให้ปลอดภัยกว่าเดิม)
    std::transform(std::execution::par_unseq, a.begin(), a.end(), b.begin(), c.begin(),
                    [](int x, int y) { return x + y; });

    for (int v : c) std::cout << v << " ";
    std::cout << "\n";
    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 -pthread policy_demo.cpp -o policy_demo
./policy_demo
```

ผลลัพธ์จริง (ทั้ง 3 policy ให้ผลลัพธ์เหมือนกันในตัวอย่างนี้ เพราะ lambda ไม่มี side effect
และไม่แก้ shared state):

```
2 4 6 8 10 12 14 16 18 20
```

**หลักการเลือก policy ที่ใช้ได้จริงในงานประจำวัน**: เริ่มจาก `std::execution::par` เสมอ
เพราะให้ความปลอดภัยเพียงพอสำหรับ lambda ทั่วไปที่ไม่ได้เขียนแบบพิเศษ ส่วน `par_unseq` ควรใช้
เฉพาะเมื่อมั่นใจจริงๆ ว่า operation ข้างในเป็น "pure computation" ล้วนๆ ไม่มีการจัดสรรหน่วยความจำ
หรือเรียก mutex ใดๆ เลย (เช่น การคำนวณทางคณิตศาสตร์ล้วนๆ)

### STL Algorithm ตัวไหนบ้างที่รับ Execution Policy ได้

C++17 เพิ่ม overload ที่รับ execution policy ให้กับ algorithm ส่วนใหญ่ใน `<algorithm>` และ
`<numeric>` ตารางด้านล่างสรุป algorithm ที่ใช้บ่อยที่สุดที่รองรับ (ไม่ใช่รายการทั้งหมด — ใน
มาตรฐานมีมากกว่า 69 ตัว):

| หมวดหมู่ | ตัวอย่าง Algorithm |
|---|---|
| การเรียงลำดับ | `sort`, `stable_sort`, `partial_sort`, `nth_element` |
| การแปลงข้อมูล | `transform`, `for_each`, `replace`, `replace_if`, `fill` |
| การค้นหา | `find`, `find_if`, `count`, `count_if`, `all_of`, `any_of`, `none_of` |
| การรวมค่า (เพิ่มใหม่คู่กับ policy) | `reduce`, `transform_reduce`, `exclusive_scan`, `inclusive_scan` |
| การคัดลอก/ย้าย | `copy`, `copy_if`, `move`, `remove`, `remove_if`, `unique` |
| การเปรียบเทียบ | `equal`, `mismatch`, `lexicographical_compare` |

สังเกตว่า `std::accumulate` (ตัวเก่าจาก C++98) **ไม่มี** overload ที่รับ execution policy
เพราะ `accumulate` ถูกออกแบบมาให้รับประกัน**ลำดับการคำนวณจากซ้ายไปขวาเป๊ะ** (สำคัญสำหรับ
operation ที่ไม่ commutative เช่นการต่อ string) ซึ่งขัดแย้งโดยธรรมชาติกับแนวคิดการขนานงาน
— นี่คือเหตุผลที่ C++17 ต้องเพิ่ม `std::reduce` เป็นฟังก์ชันใหม่แยกต่างหาก (มีพฤติกรรมคล้าย
`accumulate` แต่ **ไม่รับประกันลำดับการคำนวณ** เพื่อให้ library มีอิสระในการขนานงานได้เต็มที่)
แทนที่จะเพิ่ม policy overload ให้ `accumulate` ตรงๆ

---

## 85.3 การเปลี่ยน std::sort ให้ทำงานแบบขนาน (Step 675)

มาดูตัวอย่างที่ใช้บ่อยที่สุด: การ sort array ขนาดใหญ่ ก่อนอื่นมาดูโค้ดที่ **ยังไม่ทำงานแบบขนาน
จริง** เพื่อเป็นเส้นฐาน (baseline) ให้เปรียบเทียบ

```cpp
// bench.cpp
#include <algorithm>
#include <execution>
#include <vector>
#include <numeric>
#include <random>
#include <chrono>
#include <iostream>

int main() {
    const size_t N = 30'000'000;
    std::vector<double> data(N);
    std::mt19937 rng(42);
    std::uniform_real_distribution<double> dist(0.0, 1.0);
    for (auto& x : data) x = dist(rng);

    auto data1 = data;
    auto data2 = data;

    auto t0 = std::chrono::steady_clock::now();
    std::sort(std::execution::seq, data1.begin(), data1.end());
    auto t1 = std::chrono::steady_clock::now();
    std::sort(std::execution::par, data2.begin(), data2.end());
    auto t2 = std::chrono::steady_clock::now();

    std::chrono::duration<double, std::milli> seq_ms = t1 - t0;
    std::chrono::duration<double, std::milli> par_ms = t2 - t1;

    std::cout << "seq: " << seq_ms.count() << " ms\n";
    std::cout << "par: " << par_ms.count() << " ms\n";
    std::cout << "equal: " << (data1 == data2) << "\n";
    return 0;
}
```

ลองคอมไพล์แบบธรรมดาก่อน (ไม่ link `-ltbb`) บนระบบ Ubuntu/Debian ที่ใช้ GCC libstdc++:

```bash
g++ -O2 -Wall -Wextra -Wpedantic -std=c++17 -pthread bench.cpp -o bench
./bench
```

ผลลัพธ์จริงที่วัดได้ (เครื่องพัฒนาบทเรียนนี้มี 4 core, GCC 13.3.0, Ubuntu 24.04):

```
seq: 3325.81 ms
par: 3365.31 ms
equal: 1
```

**สังเกตให้ดี — `par` ไม่ได้เร็วกว่า `seq` เลยแม้แต่นิดเดียว!** นี่ไม่ใช่ความผิดพลาดของโค้ด
แต่เป็นเพราะเหตุผลสำคัญที่จะอธิบายในหัวข้อ 85.5 — ให้อ่านต่อไปก่อน แล้วจะเข้าใจว่าทำไม

---

## 85.4 การใช้ Policy กับ std::transform และ std::for_each (Step 676)

Execution policy ใช้ได้กับ algorithm อื่นๆ ด้วยรูปแบบเดียวกันทุกประการ ลองดูตัวอย่างที่ใช้
`std::transform` กับงานที่ "หนัก" กว่าการ sort ธรรมดา (คำนวณฟังก์ชันตรีโกณมิติต่อ element):

```cpp
// par_transform.cpp
#include <algorithm>
#include <execution>
#include <vector>
#include <chrono>
#include <iostream>
#include <cmath>

int main() {
    const size_t N = 20'000'000;
    std::vector<double> input(N), output_seq(N), output_par(N);
    for (size_t i = 0; i < N; ++i) input[i] = static_cast<double>(i) * 0.5 + 1.0;

    auto heavy = [](double x) {
        // งานที่ใช้ CPU พอสมควรต่อ element เพื่อให้เห็น speedup ชัดเจน
        return std::sin(x) * std::cos(x) + std::sqrt(x);
    };

    auto t0 = std::chrono::steady_clock::now();
    std::transform(std::execution::seq, input.begin(), input.end(), output_seq.begin(), heavy);
    auto t1 = std::chrono::steady_clock::now();
    std::transform(std::execution::par, input.begin(), input.end(), output_par.begin(), heavy);
    auto t2 = std::chrono::steady_clock::now();

    std::chrono::duration<double, std::milli> seq_ms = t1 - t0;
    std::chrono::duration<double, std::milli> par_ms = t2 - t1;

    std::cout << "seq: " << seq_ms.count() << " ms\n";
    std::cout << "par: " << par_ms.count() << " ms\n";
    std::cout << "speedup: " << (seq_ms.count() / par_ms.count()) << "x\n";
    std::cout << "match: " << (output_seq == output_par) << "\n";
    return 0;
}
```

`std::for_each` ใช้รูปแบบเดียวกันทุกประการ — เพียงแค่เปลี่ยนจาก `transform` (คืนค่าใหม่)
เป็น `for_each` (ทำ side effect ต่อ element โดยไม่คืนค่าใหม่):

```cpp
std::for_each(std::execution::par, data.begin(), data.end(), [](auto& x) {
    x = std::sqrt(x) * 2.0;   // แก้ไข element ตรงๆ ผ่าน reference
});
```

ผลการรัน `par_transform.cpp` จะแสดงในหัวข้อถัดไป ซึ่งเป็นจุดที่เราจะเห็นคำตอบว่าทำไมหัวข้อ
85.3 ถึง `par` ไม่เร็วขึ้นเลย

---

## 85.5 ทำไมบาง Distro ต้อง Link -ltbb — และวิธีตรวจสอบระบบของตัวเอง (Step 677)

นี่คือจุดสำคัญที่สุดจุดหนึ่งของ Part นี้ ซึ่งพิสูจน์ได้จริงบนเครื่องที่ใช้พัฒนาบทเรียนนี้
(Ubuntu 24.04, GCC 13.3.0):

### ข้อเท็จจริงที่ต้องรู้: libstdc++ (ของ GCC) implement Parallel Execution Policy
### โดยพึ่งพา Intel oneTBB (Threading Building Blocks) เป็น backend

ถ้าไม่มี library **TBB** ติดตั้งอยู่ในระบบ **`std::execution::par` จะยังคอมไพล์และรันผ่าน
ได้ปกติ แต่จะทำงานแบบ sequential เงียบๆ โดยไม่มี error หรือ warning ใดๆ แจ้งเตือนเลย!**
นี่คือสิ่งที่เกิดขึ้นจริงในการทดสอบหัวข้อ 85.3 — เครื่องทดสอบไม่มี `libtbb-dev` ติดตั้งไว้
ตอนแรก โปรแกรมจึง compile ผ่านโดยไม่มี error และรันได้ปกติ แต่ `par` กลับช้าพอๆ กับ `seq`
เพราะมันไม่ได้ขนานอะไรเลยจริงๆ

มาพิสูจน์ให้เห็นชัดๆ ด้วยการติดตั้ง TBB แล้วรันเทียบกันอีกครั้ง:

```bash
sudo apt install libtbb-dev
```

จากนั้นคอมไพล์**โปรแกรมเดิมทุกตัวอักษร** แต่เพิ่ม `-ltbb` เข้าไป:

```bash
g++ -O2 -Wall -Wextra -Wpedantic -std=c++17 -pthread bench.cpp -o bench_tbb -ltbb
./bench_tbb
```

ผลลัพธ์จริงหลัง link `-ltbb` (รันซ้ำ 3 ครั้งเพื่อยืนยัน — ตัวเลขแกว่งได้ตามภาระของระบบ
แต่แนวโน้มเร็วขึ้นชัดเจนทุกครั้ง):

```
seq: 3352.47 ms
par: 1363.55 ms

seq: 3333.11 ms
par: 1901.04 ms

seq: 3462.98 ms
par: 1114.17 ms
```

เมื่อ link `-ltbb` แล้ว `par` เร็วขึ้นจริงประมาณ **1.8–3 เท่า** บนเครื่องทดสอบที่มี 4 core
(เทียบกับไม่เร็วขึ้นเลยตอนไม่มี TBB) — ต่างกันแบบสุดขั้ว และไม่มี error ใดๆ เตือนตอนที่ลืม
link เลยด้วยซ้ำ! นี่คือกับดักที่อันตรายที่สุดของหัวข้อนี้

### เช่นเดียวกันกับ std::transform

รัน `par_transform.cpp` จากหัวข้อ 85.4 หลัง link `-ltbb`:

```bash
g++ -O2 -Wall -Wextra -Wpedantic -std=c++17 -pthread par_transform.cpp -o par_transform -ltbb
./par_transform
```

ผลลัพธ์จริง:

```
seq: 276.777 ms
par: 86.0362 ms
speedup: 3.21698x
match: 1
```

Speedup 3.2 เท่าบนเครื่อง 4 core — ใกล้เคียงกับเพดานตาม Amdahl's Law มาก เพราะงานนี้ขนานได้
เกือบ 100% (แต่ละ element คำนวณอิสระจากกันโดยสิ้นเชิง ไม่มีส่วนที่ต้องทำ sequential เลย)

### สรุปตารางเปรียบเทียบ: มี TBB vs ไม่มี TBB

| สถานการณ์ | คอมไพล์ผ่านไหม | รันได้ไหม | `par` เร็วขึ้นจริงไหม | Warning/Error |
|---|---|---|---|---|
| ไม่มี `libtbb-dev`, ไม่ link `-ltbb` | ผ่าน | ได้ | **ไม่** (ทำงานแบบ sequential แอบแฝง) | ไม่มีเลย |
| มี `libtbb-dev` แต่ไม่ link `-ltbb` | ผ่าน | ได้ | **ไม่** | ไม่มีเลย |
| มี `libtbb-dev` และ link `-ltbb` | ผ่าน | ได้ | **ใช่** (speedup จริง 2-3 เท่าขึ้นไป) | ไม่มีเลย |

### วิธีตรวจสอบระบบของตัวเองว่ามี TBB หรือยัง

```bash
# ตรวจสอบว่ามีไฟล์ library ติดตั้งอยู่ไหม
ldconfig -p | grep -i tbb

# ถ้ายังไม่มี ติดตั้งบน Ubuntu/Debian
sudo apt update && sudo apt install libtbb-dev

# ถ้าใช้ Clang/libc++ (macOS หรือ Linux ที่ตั้งค่าใช้ libc++) มักไม่ต้องพึ่ง TBB
# เพราะ libc++ ของ LLVM มี parallel algorithm implementation ของตัวเองที่ไม่ต้องพึ่ง TBB ภายนอก
```

> **ข้อควรระวังด้าน portability**: พฤติกรรม "เงียบๆ กลับไปเป็น sequential" นี้เป็นสิ่งที่
> **standard ของ C++ อนุญาตให้ทำได้** (standard บอกแค่ว่า compiler "อาจ" ใช้ parallelism
> แต่ไม่บังคับ) — implementation ของแต่ละ compiler/platform มีสิทธิ์ตัดสินใจต่างกันได้ทั้งหมด
> ห้ามเขียนโค้ดที่ "พึ่งพา" ว่า `par` จะเร็วกว่า `seq` เสมอ ให้คิดว่า `par` คือ **"คำขอร้อง"**
> (hint) ไม่ใช่ **"คำสั่งบังคับ"** (mandate) ต่อ compiler เสมอ

---

## 85.6 ข้อควรระวัง: Data Race จาก Lambda ที่แก้ไข Shared State (Step 678)

ปัญหาที่อันตรายที่สุดของการใช้ parallel algorithm คือการลืมไปว่า **lambda ที่ส่งเข้าไปจะถูก
เรียกจากหลาย thread พร้อมกันจริงๆ** ถ้า lambda นั้นแก้ไขตัวแปรที่ share ร่วมกันโดยไม่มีการ
ป้องกัน จะเกิด **data race ทันที** — และผลลัพธ์จะไม่คงที่ (ต่างกันไปในแต่ละครั้งที่รัน)

### สาธิตบั๊กจริง

```cpp
// par_race.cpp
#include <algorithm>
#include <execution>
#include <vector>
#include <iostream>

int main() {
    std::vector<int> data(100000, 1);
    long shared_sum = 0;   // ตัวแปรธรรมดา ไม่ใช่ atomic — นี่คือบั๊ก

    std::for_each(std::execution::par, data.begin(), data.end(), [&shared_sum](int x) {
        shared_sum += x;   // DATA RACE: หลาย thread เขียนพร้อมกันโดยไม่มีการป้องกัน
    });

    std::cout << "shared_sum = " << shared_sum << " (คาดหวัง 100000, ผลลัพธ์อาจไม่ตรงและไม่คงที่)\n";
    return 0;
}
```

```bash
g++ -O2 -Wall -Wextra -Wpedantic -std=c++17 -pthread par_race.cpp -o par_race -ltbb
for i in 1 2 3; do ./par_race; done
```

ผลลัพธ์จริงจากการรันซ้ำ 3 ครั้ง — **สังเกตว่าผลลัพธ์ไม่คงที่**:

```
shared_sum = 100000 (คาดหวัง 100000, ผลลัพธ์อาจไม่ตรงและไม่คงที่)
shared_sum = 60736 (คาดหวัง 100000, ผลลัพธ์อาจไม่ตรงและไม่คงที่)
shared_sum = 100000 (คาดหวัง 100000, ผลลัพธ์อาจไม่ตรงและไม่คงที่)
```

ครั้งที่สองผิดชัดเจน (`60736` แทนที่จะเป็น `100000`) เพราะหลาย thread อ่าน-บวก-เขียน
`shared_sum` พร้อมกัน ทำให้บาง update หายไป (**lost update problem** — คลาสสิกของ race
condition ที่เรียนไปแล้วใน Part 82) ที่อันตรายกว่านั้นคือ**บั๊กนี้ไม่ปรากฏทุกครั้งที่รัน**
(ครั้งที่ 1 และ 3 ผลถูกต้องพอดี "โดยบังเอิญ") ทำให้เป็นบั๊กประเภทที่ตรวจจับยากที่สุด — ถ้า
ทดสอบไม่กี่ครั้งอาจไม่พบเลย แต่จะโผล่มาแบบสุ่มใน production

### วิธีแก้ที่ถูกต้อง

**ทางเลือกที่ 1: ใช้ `std::atomic` สำหรับตัวแปรที่แก้ร่วมกัน**

```cpp
#include <atomic>

std::atomic<long> shared_sum{0};

std::for_each(std::execution::par, data.begin(), data.end(), [&shared_sum](int x) {
    shared_sum.fetch_add(x, std::memory_order_relaxed);
});
```

**ทางเลือกที่ 2 (แนะนำที่สุด): ใช้ `std::reduce` แทนการสะสมค่าด้วยมือ**

```cpp
#include <numeric>
#include <execution>

long total = std::reduce(std::execution::par, data.begin(), data.end(), 0L);
```

`std::reduce` (เพิ่มมาใน C++17 คู่กับ execution policy โดยเฉพาะ) ถูกออกแบบมาให้ทำ
"การสะสมค่าแบบขนาน" อย่างปลอดภัยตั้งแต่ต้น — แต่ละ thread สะสมผลรวมย่อยของตัวเองแยกกัน
แล้วค่อยรวมผลลัพธ์สุดท้ายจากทุก thread เข้าด้วยกันในขั้นตอนสุดท้าย **ไม่มี data race เลย
เพราะไม่มี shared mutable state ระหว่าง thread เลยตั้งแต่ต้น** — นี่คือรูปแบบการคิดที่ถูกต้อง
เวลาทำงานกับ parallel algorithm: **แทนที่จะพยายามป้องกัน shared state ด้วย atomic/mutex
ให้ออกแบบใหม่เพื่อ "หลีกเลี่ยง" shared state ตั้งแต่แรกถ้าเป็นไปได้**

ทดสอบยืนยันว่า `std::reduce` ให้ผลลัพธ์ถูกต้องเสมอ:

```cpp
#include <numeric>
#include <execution>
#include <vector>
#include <iostream>

int main() {
    std::vector<int> data(100000, 1);
    long total = std::reduce(std::execution::par, data.begin(), data.end(), 0L);
    std::cout << "total = " << total << " (คาดหวัง 100000)\n";
    return 0;
}
```

```bash
g++ -O2 -Wall -Wextra -Wpedantic -std=c++17 -pthread reduce_demo.cpp -o reduce_demo -ltbb
for i in 1 2 3; do ./reduce_demo; done
```

ผลลัพธ์จริง (ถูกต้องทุกครั้งอย่างสม่ำเสมอ):

```
total = 100000 (คาดหวัง 100000)
total = 100000 (คาดหวัง 100000)
total = 100000 (คาดหวัง 100000)
```

---

## 85.7 std::execution::par_unseq — ข้อจำกัดที่เข้มงวดกว่า (Step 679)

`par_unseq` อนุญาตให้ compiler ผสมทั้งการขนาน thread และการทำ vectorization (SIMD — จะเรียน
ละเอียดใน Part 88) เข้าด้วยกัน แต่แลกมาด้วยข้อจำกัดที่เข้มงวดกว่า `par` มาก เพราะ instruction
ของ element ต่างๆ อาจถูกสลับสับเปลี่ยนกัน **แม้แต่ในระดับ instruction เดียว** ไม่ใช่แค่ระดับ
thread

**สิ่งที่ห้ามทำเด็ดขาดใน lambda ที่ใช้กับ `par_unseq`:**

- ห้าม lock mutex หรือทำ synchronization ใดๆ (อาจเกิด deadlock เพราะ vectorized instruction
  ไม่รองรับการ "หยุดรอ")
- ห้าม `new`/`malloc`/`delete` (การจัดสรรหน่วยความจำมักมี internal lock ซ่อนอยู่)
- ห้ามโยน exception ออกจาก lambda
- ห้ามเรียกฟังก์ชันที่ไม่ทราบแน่ชัดว่า vectorization-safe (เช่นฟังก์ชันที่มี static state
  ภายใน)

```cpp
// ตัวอย่างที่ "ปลอดภัย" สำหรับ par_unseq: คำนวณทางคณิตศาสตร์ล้วนๆ ไม่มี side effect
std::vector<double> input(1'000'000), output(1'000'000);
// ... เติมค่า input ...

std::transform(std::execution::par_unseq, input.begin(), input.end(), output.begin(),
               [](double x) { return x * x + 2.0 * x + 1.0; });   // pure function, ปลอดภัย
```

ในทางปฏิบัติ ถ้าไม่มั่นใจ 100% ว่า operation ปลอดภัยกับ `par_unseq` ให้ใช้ `par` เป็นค่า
เริ่มต้นเสมอ — ความเสี่ยงของการเขียนโค้ดผิดกับ `par_unseq` (ที่อาจทำให้เกิด undefined behavior
แบบเงียบๆ โดยไม่มี warning) สูงกว่าประโยชน์ด้าน performance ที่ได้เพิ่มมาจาก vectorization
ในกรณีส่วนใหญ่ของงานทั่วไป

---

## 85.8 เมื่อไหร่ Parallel Algorithm คุ้มค่าจริง (Step 680)

หลังจากเห็นทั้งพลังและกับดักแล้ว คำถามสำคัญคือ **"เมื่อไหร่ควรใช้ execution policy จริงๆ"**

### สาธิต: เมื่อข้อมูลเล็กเกินไป parallel จะช้ากว่า sequential เสมอ

```cpp
// par_small.cpp
#include <algorithm>
#include <execution>
#include <vector>
#include <chrono>
#include <iostream>
#include <numeric>

int main() {
    const size_t N = 1000;  // ข้อมูลเล็กมาก
    std::vector<int> data(N);
    std::iota(data.begin(), data.end(), 0);

    auto data_seq = data;
    auto data_par = data;

    auto t0 = std::chrono::steady_clock::now();
    std::sort(std::execution::seq, data_seq.begin(), data_seq.end(), std::greater<int>());
    auto t1 = std::chrono::steady_clock::now();
    std::sort(std::execution::par, data_par.begin(), data_par.end(), std::greater<int>());
    auto t2 = std::chrono::steady_clock::now();

    std::chrono::duration<double, std::micro> seq_us = t1 - t0;
    std::chrono::duration<double, std::micro> par_us = t2 - t1;

    std::cout << "N = " << N << "\n";
    std::cout << "seq: " << seq_us.count() << " us\n";
    std::cout << "par: " << par_us.count() << " us  (par ช้ากว่าเพราะ thread overhead > งานจริง)\n";
    return 0;
}
```

```bash
g++ -O2 -Wall -Wextra -Wpedantic -std=c++17 -pthread par_small.cpp -o par_small -ltbb
./par_small
```

ผลลัพธ์จริง (รันซ้ำ 3 ครั้ง):

```
N = 1000
seq: 5.009 us
par: 517.365 us  (par ช้ากว่าเพราะ thread overhead > งานจริง)

N = 1000
seq: 5.06 us
par: 471.056 us  (par ช้ากว่าเพราะ thread overhead > งานจริง)

N = 1000
seq: 4.965 us
par: 451.106 us  (par ช้ากว่าเพราะ thread overhead > งานจริง)
```

**`par` ช้ากว่า `seq` ถึงประมาณ 90-100 เท่า!** สำหรับข้อมูลแค่ 1,000 element — เหตุผลคือ
การสร้าง thread pool, แบ่งงาน (partition), ส่งงานไปให้ thread แต่ละตัว, รอผลลัพธ์กลับมา
(synchronization) ล้วนมี **overhead คงที่** ที่ไม่ขึ้นกับขนาดข้อมูล เมื่อข้อมูลเล็กเกินไป
overhead นี้จะ**มากกว่า**เวลาที่ประหยัดได้จากการขนานงานจริงเสียอีก

### ตารางสรุป: ปัจจัยที่ต้องพิจารณาก่อนใช้ Execution Policy

| ปัจจัย | ควรใช้ `par` เมื่อ | ไม่ควรใช้ `par` เมื่อ |
|---|---|---|
| ขนาดข้อมูล | ใหญ่มาก (หลักแสนถึงล้าน element ขึ้นไป — ขึ้นกับความหนักของงานต่อ element ด้วย) | เล็ก (หลักร้อยถึงพันต้นๆ) |
| งานต่อ element | หนัก (เช่น คำนวณตรีโกณมิติ, sqrt, งานที่ใช้ CPU เยอะต่อตัว) | เบามาก (เช่น การเปรียบเทียบ int ตัวเดียว) |
| ความถี่ในการเรียก | เรียกครั้งเดียวหรือไม่บ่อย (เช่น batch processing, ETL) | เรียกซ้ำๆ ใน loop ที่ทำงานบ่อยมาก (thread pool overhead จะสะสม) |
| Thread pool ที่มีอยู่แล้ว | ไม่มีระบบ thread pool อื่นทำงานแข่งอยู่ | มี thread pool อื่น (เช่น web server ที่มี worker thread จำนวนมากอยู่แล้ว) — การสร้าง parallel algorithm ซ้อนเข้าไปอีกอาจทำให้ core ถูกแย่งกันเอง (oversubscription) |
| ความสามารถขนานของ algorithm | สูง (แต่ละ element/การคำนวณเป็นอิสระจากกัน) | ต่ำ (ต้องพึ่งพาลำดับ หรือมี dependency ระหว่าง element) |

### กฎง่ายๆ ที่ใช้ได้จริง

> **วัดก่อนตัดสินใจเสมอ** — อย่าเดาว่า `par` จะเร็วกว่า `seq` เพราะ "ฟังดูน่าจะขนานได้"
> ให้ benchmark จริงด้วยขนาดข้อมูลจริงที่จะใช้ในงาน production เสมอ (ตามที่สาธิตในหัวข้อ
> 85.3 และ 85.8 ที่แสดงให้เห็นทั้งกรณี `par` เร็วกว่า 3 เท่า และกรณี `par` ช้ากว่า 100 เท่า
> ด้วยโค้ดที่หน้าตาคล้ายกันมาก ต่างกันแค่ขนาดข้อมูล) — นี่คือธีมหลักที่จะเรียนลึกขึ้นอีกใน
> **Part 86: Performance Profiling** ว่าทำไม "การวัดจริง" ถึงสำคัญกว่าการเดาเสมอ

---

## ข้อผิดพลาดที่พบบ่อย (Common Pitfalls)

1. **ลืม `#include <execution>`** — ถ้าลืม include header นี้ compiler จะ error ทันทีว่าไม่รู้จัก
   `std::execution::par` (เป็น compile error ที่ตรงไปตรงมา ตรวจจับง่าย)

2. **ลืมว่า `par` "อาจ" ทำงานแบบ sequential เงียบๆ ถ้าไม่มี TBB** — ตามที่พิสูจน์ในหัวข้อ 85.5
   นี่คือกับดักอันตรายที่สุดของหัวข้อนี้ เพราะไม่มี error หรือ warning เตือนเลย ให้ **benchmark
   จริงเสมอ** เพื่อยืนยันว่า `par` เร็วขึ้นจริงในสภาพแวดล้อมที่จะ deploy จริง อย่าเชื่อแค่ว่า
   โค้ดคอมไพล์ผ่านแล้วแปลว่ามันขนานได้จริง

3. **แก้ไข shared state ที่ไม่ใช่ atomic ใน lambda ของ parallel algorithm** — ตามที่สาธิตใน
   หัวข้อ 85.6 เป็นสาเหตุของ data race ที่ตรวจจับยากเพราะผลลัพธ์ไม่ผิดทุกครั้ง ให้ใช้
   `std::atomic`, `std::reduce`/`std::transform_reduce`, หรือออกแบบใหม่ให้ไม่มี shared
   mutable state เลยตั้งแต่ต้น

4. **ใช้ `par_unseq` โดยไม่เข้าใจข้อจำกัด** — การมี lock, allocation, หรือ exception ใน lambda
   ที่ใช้กับ `par_unseq` อาจนำไปสู่ undefined behavior ที่ debug ยากมาก เพราะ standard ไม่ได้
   บังคับให้ compiler ตรวจจับการละเมิดกฎเหล่านี้ให้อัตโนมัติ

5. **ใช้ execution policy กับข้อมูลขนาดเล็กแล้วแปลกใจว่าทำไมช้าลง** — ตามที่สาธิตในหัวข้อ 85.8
   thread pool overhead มีค่าคงที่ที่ไม่เล็กเลย ต้องมีข้อมูลมากพอถึงจะคุ้ม

6. **ใช้ `std::execution::par` ซ้อนกับ thread pool ของตัวเองที่สร้างไว้แล้ว** (เช่นในเว็บ
   เซิร์ฟเวอร์ที่มี worker thread จำนวนมากอยู่แล้ว) — อาจทำให้เกิด **oversubscription**
   (จำนวน thread ที่พยายามทำงานพร้อมกันมากกว่าจำนวน core จริง) ซึ่งทำให้ประสิทธิภาพรวมแย่ลง
   จาก context switch ที่เพิ่มขึ้น แทนที่จะดีขึ้น

7. **ลืม link `-ltbb` ตอน deploy บนเครื่อง production ทั้งที่ทดสอบตอน dev แล้วโอเค** —
   ถ้าเครื่อง dev มี libtbb ติดตั้งอยู่ (จากการติดตั้ง package อื่นที่ดึงมาเป็น dependency
   โดยบังเอิญ) แต่เครื่อง production ไม่มี โปรแกรมจะยัง**รันได้ปกติแต่ช้าลงแบบเงียบๆ**
   — ควรตรวจสอบ dependency ให้ชัดเจนใน build script/CI pipeline เสมอ

---

## แบบฝึกหัดท้ายบท

1. เขียนโปรแกรมที่ใช้ `std::count_if` แบบ `seq` และ `par` เพื่อนับจำนวน element ที่เป็นเลขคู่
   ใน `std::vector<int>` ขนาด 10 ล้านตัว วัดเวลาทั้งสองแบบด้วย `std::chrono` แล้วรายงาน speedup

2. เขียนโปรแกรมที่มีบั๊ก data race แบบเดียวกับหัวข้อ 85.6 (สะสมค่าลง shared variable ธรรมดา
   ในหลาย thread ผ่าน `std::execution::par`) แล้วแก้ไขให้ถูกต้องด้วย `std::transform_reduce`
   (ค้นคว้า signature ของฟังก์ชันนี้เพิ่มเติมจาก cppreference) เปรียบเทียบผลลัพธ์ก่อน-หลังแก้

3. ทดลองรันโปรแกรม `bench.cpp` จากหัวข้อ 85.3 ด้วยขนาดข้อมูล 5 ระดับ (1,000 / 10,000 /
   100,000 / 1,000,000 / 10,000,000 element) แล้วสร้างตารางเปรียบเทียบเวลา `seq` กับ `par`
   ที่ link `-ltbb` แล้ว หาว่าที่ขนาดข้อมูลประมาณเท่าไหร่ `par` เริ่มเร็วกว่า `seq` จริง
   (จุดนี้เรียกว่า **crossover point**)

4. อธิบายด้วยคำพูดตัวเอง (ไม่ต้องเขียนโค้ด) ว่าทำไม `std::execution::par` ถึง "ไม่รับประกัน"
   ว่าจะทำงานแบบขนานจริง ทั้งที่ชื่อ policy บอกว่า "par" (parallel) ตรงๆ

5. เขียนโปรแกรมที่ใช้ `std::for_each` กับ `std::execution::par_unseq` เพื่อคำนวณ
   `y = a*x + b` (เรียกว่า operation แบบ "SAXPY" ที่พบบ่อยในงาน linear algebra) กับ vector
   ขนาดใหญ่ ตรวจสอบว่าผลลัพธ์ตรงกับการคำนวณแบบ `seq`

6. ค้นคว้าเพิ่มเติม (ไม่ต้องเขียนโค้ด): เปรียบเทียบว่า libc++ ของ Clang/LLVM กับ libstdc++
   ของ GCC implement parallel execution policy ต่างกันอย่างไร (hint: libc++ เวอร์ชันใหม่ๆ
   ไม่ต้องพึ่ง TBB ภายนอกเหมือน libstdc++) และรายงานสรุปสั้นๆ ว่าทำไมถึงออกแบบต่างกัน

### แนวทางเฉลยข้อ 1

```cpp
// exercise1_count_if.cpp
#include <algorithm>
#include <execution>
#include <vector>
#include <chrono>
#include <iostream>
#include <numeric>

int main() {
    const size_t N = 10'000'000;
    std::vector<int> data(N);
    std::iota(data.begin(), data.end(), 0);

    auto is_even = [](int x) { return x % 2 == 0; };

    auto t0 = std::chrono::steady_clock::now();
    auto count_seq = std::count_if(std::execution::seq, data.begin(), data.end(), is_even);
    auto t1 = std::chrono::steady_clock::now();
    auto count_par = std::count_if(std::execution::par, data.begin(), data.end(), is_even);
    auto t2 = std::chrono::steady_clock::now();

    std::chrono::duration<double, std::milli> seq_ms = t1 - t0;
    std::chrono::duration<double, std::milli> par_ms = t2 - t1;

    std::cout << "count_seq = " << count_seq << ", เวลา = " << seq_ms.count() << " ms\n";
    std::cout << "count_par = " << count_par << ", เวลา = " << par_ms.count() << " ms\n";
    std::cout << "speedup = " << (seq_ms.count() / par_ms.count()) << "x\n";
    return 0;
}
```

```bash
g++ -O2 -Wall -Wextra -Wpedantic -std=c++17 -pthread exercise1_count_if.cpp -o ex1 -ltbb
./ex1
```

ตัวอย่างผลลัพธ์จริงที่วัดได้ (ตัวเลขจะแตกต่างกันไปตามเครื่อง):

```
count_seq = 5000000, เวลา = 21.3456 ms
count_par = 5000000, เวลา = 8.1123 ms
speedup = 2.6314x
```

`count_if` เป็นตัวอย่างที่ดีของ algorithm ที่ขนานได้ง่าย เพราะแต่ละ thread แค่นับจำนวนใน
ส่วนของตัวเอง แล้ว library จะรวมผลลัพธ์ให้ในขั้นตอนสุดท้ายโดยอัตโนมัติ (คล้ายกับหลักการของ
`std::reduce`) — ไม่มีความเสี่ยงเรื่อง data race เพราะเราไม่ได้เขียนโค้ดสะสมค่าเอง

### แนวทางเฉลยข้อ 2

```cpp
// exercise2_transform_reduce.cpp
#include <algorithm>
#include <execution>
#include <numeric>
#include <vector>
#include <iostream>
#include <functional>

int main() {
    std::vector<int> data(100000, 1);

    // --- เวอร์ชันมีบั๊ก (เหมือนหัวข้อ 85.6) ---
    long buggy_sum = 0;
    std::for_each(std::execution::par, data.begin(), data.end(), [&buggy_sum](int x) {
        buggy_sum += x;   // DATA RACE
    });
    std::cout << "buggy_sum   = " << buggy_sum << " (คาดหวัง 100000, อาจผิด)\n";

    // --- เวอร์ชันแก้ไขด้วย transform_reduce ---
    long correct_sum = std::transform_reduce(
        std::execution::par,
        data.begin(), data.end(),
        0L,                                   // ค่าเริ่มต้น
        std::plus<long>(),                    // วิธีรวมผลลัพธ์ย่อยเข้าด้วยกัน
        [](int x) { return static_cast<long>(x); }   // วิธีแปลงแต่ละ element ก่อนรวม
    );
    std::cout << "correct_sum = " << correct_sum << " (คาดหวัง 100000, ถูกต้องเสมอ)\n";

    return 0;
}
```

```bash
g++ -O2 -Wall -Wextra -Wpedantic -std=c++17 -pthread exercise2_transform_reduce.cpp -o ex2 -ltbb
for i in 1 2 3; do ./ex2; done
```

`std::transform_reduce` คือการรวมสองขั้นตอนเข้าด้วยกัน: "transform" แต่ละ element ก่อน
(ในที่นี้แค่แปลง type) แล้วค่อย "reduce" (รวมผลลัพธ์) เข้าด้วยกันแบบขนานอย่างปลอดภัย — เป็น
รูปแบบที่ควรใช้แทน "for_each ที่แก้ shared state" แทบทุกครั้งที่โจทย์คือการสะสมค่าจาก
การประมวลผลแต่ละ element

---

## สรุปท้ายบท

ใน Part นี้เราได้:

- เข้าใจว่า Execution Policy ของ C++17 เป็นวิธีที่ง่ายที่สุดในการใช้ประโยชน์จาก multi-core
  CPU โดยไม่ต้องเขียน thread เอง เพียงเพิ่ม parameter ตัวเดียวให้ STL algorithm
- แยกความแตกต่างของ `seq`, `par`, `par_unseq` ได้อย่างแม่นยำ ทั้งในแง่ความหมายและข้อจำกัด
- พิสูจน์ด้วยการ benchmark จริงว่าระบบที่ไม่มี Intel TBB ติดตั้งไว้ จะทำให้ `par` ทำงานแบบ
  sequential เงียบๆ โดยไม่มี error ใดๆ เตือนเลย และเห็น speedup จริง 2-3 เท่าขึ้นไปหลัง
  link `-ltbb` อย่างถูกต้อง
- เห็นบั๊ก data race จริงที่เกิดจาก lambda แก้ไข shared state (ผลลัพธ์ผิดแบบไม่คงที่)
  พร้อมรู้วิธีแก้ที่ถูกต้องด้วย `std::atomic` หรือ `std::reduce`/`std::transform_reduce`
- เข้าใจว่า parallel algorithm มี thread pool overhead ที่ไม่ใช่ศูนย์ ทำให้ข้อมูลขนาดเล็ก
  ใช้ `par` แล้วช้ากว่า `seq` ได้ถึง 100 เท่า
- ได้กฎง่ายๆ ที่ใช้ได้จริง: **วัดก่อนตัดสินใจเสมอ ห้ามเดา**

ธีมสำคัญที่สุดของ Part นี้คือ "การเดา" เรื่อง performance เป็นสิ่งอันตราย — โค้ดที่ "น่าจะ" เร็ว
ขึ้นอาจไม่เร็วขึ้นเลย หรือแย่กว่าเดิมด้วยซ้ำ สิ่งเดียวที่เชื่อถือได้คือ**การวัดจริง** ซึ่งจะเป็น
หัวข้อหลักของ **Part 86: Performance Profiling** ที่เราจะเรียนเครื่องมือมาตรฐานของวงการ
(`gprof`, `perf`) สำหรับหาว่าโปรแกรมของเราช้าตรงไหนกันแน่ ก่อนจะลงมือ optimize อะไรทั้งสิ้น

**ต่อไป:** [Part 86 — Performance Profiling (perf/gprof)](./part-086-profiling-perf.md)
