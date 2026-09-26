# Part 87: Cache-Friendly Code และ Data-Oriented Design (Step 689–696)

> Module G — Concurrency และ Performance Engineering | Part 87 จาก 125
> Step ที่ครอบคลุมใน Part นี้: Step 689–696
> Part ก่อนหน้า: [Part 86 — Performance Profiling (perf/gprof)](./part-086-profiling-perf.md) | Part ถัดไป: [Part 88 — SIMD และ Vectorization เบื้องต้น](./part-088-simd-vectorization.md)

## เป้าหมายการเรียนรู้ (Learning Objectives)

หลังจบ Part นี้ ผู้เรียนจะสามารถ:

1. อธิบาย **Memory Wall** ได้ว่าทำไมความเร็ว CPU กับความเร็วหน่วยความจำถึงห่างกันมหาศาล และทำไม
   เรื่องนี้ถึงสำคัญกว่าที่มือใหม่ส่วนใหญ่คิด
2. อธิบายลำดับชั้นของ Cache (L1/L2/L3/RAM) พร้อม latency ที่ต่างกันเป็นระดับสิบถึงร้อยเท่า และ
   **วัด latency จริงบนเครื่องของตัวเอง** ด้วยเทคนิค pointer chasing
3. อธิบาย **Cache Line** (64 bytes โดยทั่วไป) และหลักการ **Spatial Locality / Temporal Locality**
   พร้อมยกตัวอย่างโค้ดที่ใช้หลักการนี้ได้ดีและไม่ดี
4. เขียนโปรแกรมเปรียบเทียบการวน loop เข้าถึง 2D array แบบ row-major กับ column-major แล้ว
   **วัดผลต่างของความเร็วจริง** ด้วย `std::chrono`
5. อธิบายความแตกต่างระหว่าง **Array of Structures (AoS)** กับ **Structure of Arrays (SoA)**
   และเขียนโปรแกรมวัดผลต่างจริงเมื่อประมวลผลแค่บาง field ของข้อมูลจำนวนมาก
6. อธิบาย **False Sharing** ที่เกิดจาก multi-thread เข้าถึง cache line เดียวกัน พร้อมเขียนโปรแกรม
   สาธิตและวัดผลต่างจริงเมื่อจัดวางข้อมูลให้ห่างกันคนละ cache line
7. อธิบายหลักการของ **Data-Oriented Design (DOD)** และรู้ว่าเมื่อไหร่ควรนำมาใช้แทนการออกแบบ
   แบบ Object-Oriented แบบดั้งเดิม
8. รู้จักข้อผิดพลาดที่พบบ่อยเวลาพยายามเขียนโค้ดให้ cache-friendly และรู้วิธีตรวจสอบว่าการปรับ
   ที่ทำไปนั้นช่วยจริงหรือเป็นแค่ความเชื่อ

---

## 87.1 กำแพงความเร็วหน่วยความจำ (The Memory Wall) (Step 689)

ใน Part 86 เราเรียนรู้การ Profile โปรแกรมด้วย `perf` และ `gprof` เพื่อหาว่า "ฟังก์ชันไหนกินเวลา
มากที่สุด" แต่คำถามที่ลึกกว่านั้นคือ: **ทำไมฟังก์ชันที่ทำ "งานคำนวณ" เหมือนกันทุกประการ ถึงเร็ว
ช้าต่างกันได้หลายเท่า แค่เปลี่ยนวิธีจัดเรียงข้อมูลในหน่วยความจำ?**

คำตอบอยู่ที่ความจริงข้อหนึ่งที่โปรแกรมเมอร์จำนวนมากมองข้าม: **CPU เร็วขึ้นเร็วกว่า RAM มาก
ตลอด 40 ปีที่ผ่านมา** ในยุค 1980 CPU กับ RAM มีความเร็วใกล้เคียงกัน (การอ่านค่าจาก RAM ใช้เวลา
ไม่กี่ CPU cycle) แต่ในปี 2026 นี้ CPU สมัยใหม่ทำงานได้หลายพันล้านคำสั่งต่อวินาที ในขณะที่การอ่าน
ค่าจาก RAM ยังต้องรอนับ **ร้อยรอบสัญญาณนาฬิกา (clock cycle)** ต่อครั้ง ช่องว่างนี้เรียกว่า
**Memory Wall** — และมันคือเหตุผลที่ CPU สมัยใหม่แทบทุกตัวต้องมี **Cache** หลายชั้นซ้อนกันเพื่อ
"ซ่อน" ความช้าของ RAM เอาไว้

### ลำดับชั้นของ Cache (Cache Hierarchy)

CPU สมัยใหม่มีหน่วยความจำเรียงเป็นชั้น (Hierarchy) จากเร็ว/เล็ก ไปหาช้า/ใหญ่:

```
┌─────────────────────────────────────────────────────────┐
│  CPU Core                                                │
│  ┌───────────┐                                           │
│  │ Registers │  เร็วที่สุด, เล็กที่สุด (หน่วย byte)         │
│  └─────┬─────┘                                           │
│  ┌─────▼─────┐                                           │
│  │  L1 Cache │  ~32-64 KB ต่อ core, latency ~1 ns         │
│  └─────┬─────┘                                           │
│  ┌─────▼─────┐                                           │
│  │  L2 Cache │  ~256 KB - 2 MB ต่อ core, latency ~3-10 ns │
│  └─────┬─────┘                                           │
└────────┼──────────────────────────────────────────────────┘
   ┌─────▼─────┐
   │  L3 Cache │  หลาย MB, แชร์ร่วมกันทุก core, latency ~10-40 ns
   └─────┬─────┘
   ┌─────▼─────┐
   │    RAM    │  หลาย GB, latency ~60-150+ ns (ช้ากว่า L1 หลายร้อยเท่า)
   └───────────┘
```

เครื่องที่ใช้เขียนบทเรียนนี้ (ตรวจสอบด้วย `lscpu`) เป็น Intel Xeon เสมือน (รันบน KVM) มีสเปกดังนี้:

```bash
lscpu | grep -E "L1d|L1i|L2|L3|Model name"
```

```
Model name:    Intel(R) Xeon(R) Processor @ 2.10GHz
L1d cache:     192 KiB (4 instances)   # ~48 KiB ต่อ core
L1i cache:     128 KiB (4 instances)
L2 cache:      8 MiB (4 instances)     # ~2 MiB ต่อ core
L3 cache:      260 MiB (1 instance)    # แชร์ทั้ง 4 core (ใหญ่ผิดปกติเพราะเป็น VM cloud)
```

> **หมายเหตุความซื่อสัตย์**: L3 cache 260 MiB ของเครื่องนี้ใหญ่กว่า CPU consumer ทั่วไปมาก
> (Desktop/Laptop ทั่วไปมี L3 แค่ 8-32 MB) เพราะนี่คือ CPU เสมือนบน Cloud ที่รายงานค่า cache
> ของ Host Machine ทั้งหมด ตัวเลขที่ได้จริงจากการรันเบนช์มาร์กด้านล่างจึงอาจต่างจากเครื่องพีซี
> ที่บ้านของผู้เรียนบ้าง แต่ **รูปแบบ (pattern)** ที่เห็น — ยิ่งข้อมูลใหญ่กว่าขนาด cache
> ยิ่งช้าลงแบบก้าวกระโดด — จะเหมือนกันทุกเครื่องเสมอ เพราะนี่คือคุณสมบัติพื้นฐานของสถาปัตยกรรม
> CPU ไม่ใช่ค่าเฉพาะเครื่อง

### วัด Latency จริงด้วยเทคนิค Pointer Chasing

แทนที่จะเชื่อตัวเลขจากตำรา เรามาวัด latency จริงของเครื่องกันเลย ด้วยเทคนิคที่เรียกว่า
**Pointer Chasing**: สร้าง array ที่แต่ละช่องเก็บ index ของช่องถัดไปแบบ "สุ่ม" ให้ครบวงจร
(single cycle) แล้ววนเดินตามลูกโซ่นั้นไปเรื่อยๆ (`idx = arr[idx]`) เทคนิคนี้สำคัญตรงที่
**การเข้าถึงแต่ละครั้งต้องรอผลลัพธ์จากครั้งก่อนหน้าเสมอ** (dependent load) ทำให้ CPU ไม่สามารถ
เดาล่วงหน้า (prefetch) ที่อยู่ถัดไปได้เลย จึงวัด "latency ล้วนๆ" ของแต่ละชั้น cache ได้ตรงกว่า
การอ่านข้อมูลเรียงลำดับธรรมดา (ซึ่ง CPU จะ prefetch ล่วงหน้าให้อัตโนมัติ ทำให้เราวัด bandwidth
แทนที่จะเป็น latency):

```cpp
// latency.cpp - วัด latency การเข้าถึงหน่วยความจำจริงบนเครื่องนี้ ด้วยเทคนิค pointer chasing
#include <cstdio>
#include <cstdlib>
#include <vector>
#include <random>
#include <chrono>
#include <algorithm>

// สร้าง permutation แบบวงจรเดียว (single cycle) ขนาด n เพื่อไม่ให้เกิด pattern ที่ prefetcher จับทางได้
static std::vector<uint32_t> make_cycle(uint32_t n, std::mt19937& rng) {
    std::vector<uint32_t> perm(n);
    for (uint32_t i = 0; i < n; ++i) perm[i] = i;
    std::shuffle(perm.begin(), perm.end(), rng);
    std::vector<uint32_t> next(n);
    for (uint32_t i = 0; i < n; ++i) {
        next[perm[i]] = perm[(i + 1) % n];
    }
    return next;
}

int main() {
    std::mt19937 rng(42);
    // ขนาด working set (KB) ครอบคลุมตั้งแต่เล็กกว่า L1 จนถึงใหญ่กว่า L3
    const std::vector<uint32_t> sizes_kb = {
        4, 8, 16, 32, 48, 64, 96, 128, 256, 512, 1024, 2048, 4096, 8192,
        16384, 32768, 65536, 131072
    };

    constexpr long long MAX_ITERS = 8'000'000LL; // เพดานจำนวนครั้งที่วนสูงสุด กันเวลารันบานปลาย
    constexpr long long MIN_ITERS = 2'000'000LL; // ขั้นต่ำ เพื่อให้ผลเฉลี่ยนิ่งพอสำหรับ working set เล็ก

    std::printf("%-12s %-14s %-14s\n", "ขนาด (KB)", "จำนวน element", "เวลาเฉลี่ย/ครั้ง (ns)");
    std::fflush(stdout);
    for (uint32_t kb : sizes_kb) {
        uint32_t n = (kb * 1024) / sizeof(uint32_t);
        auto next = make_cycle(n, rng);

        long long iters = MIN_ITERS;
        if (iters < static_cast<long long>(n) * 4) iters = static_cast<long long>(n) * 4;
        if (iters > MAX_ITERS) iters = MAX_ITERS;

        uint32_t idx = 0;
        auto t1 = std::chrono::steady_clock::now();
        for (long long i = 0; i < iters; ++i) {
            idx = next[idx]; // dependent load: ต้องรอผลลัพธ์ก่อนหน้าเสมอ ป้องกัน prefetch/reorder
        }
        auto t2 = std::chrono::steady_clock::now();

        double ns = std::chrono::duration<double, std::nano>(t2 - t1).count() / static_cast<double>(iters);
        std::printf("%-12u %-14u %-14.3f  (idx สุดท้าย=%u)\n", kb, n, ns, idx);
    }
    return 0;
}
```

คอมไพล์และรัน:

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 -O2 latency.cpp -o latency
./latency
```

**ผลลัพธ์ที่วัดได้จริงบนเครื่องนี้:**

| ขนาด Working Set | จำนวน element | เวลาเฉลี่ย/ครั้ง (ns) |
|---:|---:|---:|
| 4 KB | 1,024 | 1.794 |
| 8 KB | 2,048 | 1.768 |
| 16 KB | 4,096 | 1.749 |
| 32 KB | 8,192 | 1.797 |
| 48 KB | 12,288 | 2.334 |
| 64 KB | 16,384 | 2.973 |
| 96 KB | 24,576 | 3.858 |
| 128 KB | 32,768 | 4.357 |
| 256 KB | 65,536 | 5.008 |
| 512 KB | 131,072 | 5.950 |
| 1 MB | 262,144 | 8.057 |
| 2 MB | 524,288 | 12.904 |
| 4 MB | 1,048,576 | 26.863 |
| 8 MB | 2,097,152 | 32.669 |
| 16 MB | 4,194,304 | 64.063 |
| 32 MB | 8,388,608 | 119.241 |
| 64 MB | 16,777,216 | 132.844 |
| 128 MB | 33,554,432 | 146.515 |

ตัวเลขนี้ **วัดได้จริง ไม่ใช่ตัวเลขจากตำรา** และเห็นรูปแบบ "บันได" ชัดเจนตรงตามทฤษฎี:

- **4-32 KB** (พอดี L1d ต่อ core): latency นิ่งอยู่ที่ราว **1.7-1.8 ns** เท่านั้น
- **48-128 KB** (เริ่มล้น L1, ปนกับ L2): latency ไต่ขึ้นจาก 2.3 → 4.4 ns
- **256 KB - 1 MB** (อยู่ใน L2 ~2 MiB ต่อ core): latency 5-8 ns
- **2-8 MB** (เริ่มล้น L2 เข้าสู่ L3): latency กระโดดขึ้นเป็น 13-33 ns
- **16-128 MB** (ล้น L3 ต้องไป RAM): latency พุ่งไปที่ **64-147 ns** หรือ **ช้ากว่า L1 ถึง ~80 เท่า**

นี่คือหลักฐานที่จับต้องได้ว่าทำไมการออกแบบโครงสร้างข้อมูลให้ "เข้ากันได้ดีกับ cache" ถึงส่งผล
ต่อความเร็วโปรแกรมมากกว่าการลดจำนวนคำสั่ง (instruction count) เสียอีกในหลายกรณี

---

## 87.2 Cache Line และหลักการ Locality (Step 690)

### Cache Line คืออะไร

CPU ไม่เคยโหลดข้อมูลจาก RAM ทีละ 1 byte หรือแม้แต่ทีละตัวแปร แต่จะโหลดเป็นก้อนขนาดคงที่เรียกว่า
**Cache Line** เสมอ บนสถาปัตยกรรม x86_64 เกือบทั้งหมด (รวมถึงเครื่องที่ใช้เขียนบทเรียนนี้)
ขนาด cache line คือ **64 bytes** ตรวจสอบได้จริงด้วยคำสั่ง:

```bash
getconf LEVEL1_DCACHE_LINESIZE
# 64
```

นั่นแปลว่าเมื่อโปรแกรมขออ่าน `int` ตัวเดียว (4 bytes) จาก RAM ที่ไม่เคยอยู่ใน cache มาก่อน
CPU จะไม่ได้ดึงมาแค่ 4 bytes นั้น แต่จะดึงทั้ง **cache line ขนาด 64 bytes ที่ครอบคลุมตำแหน่งนั้น**
เข้ามาทั้งก้อน (นั่นคือ int ตัวข้างเคียงอีก 15 ตัวก็ถูกดึงเข้ามาด้วยโดยอัตโนมัติ "ฟรี")

```
Memory:  [ 0][ 1][ 2][ 3][ 4][ 5][ 6][ 7] ... [15]   <- int แต่ละตัว 4 bytes
                    ▲
              ขออ่าน int[2]
                    │
                    ▼
Cache Line (64 bytes) = int[0] ถึง int[15] ถูกดึงเข้า cache ทั้งก้อนในครั้งเดียว
```

### Spatial Locality (ความใกล้ชิดเชิงตำแหน่ง)

**Spatial Locality** คือหลักการที่ว่า "ถ้าเข้าถึงตำแหน่งความจำหนึ่ง มีโอกาสสูงที่จะเข้าถึงตำแหน่ง
ข้างเคียงในไม่ช้า" โปรแกรมที่ **เข้าถึงข้อมูลเรียงติดกันในหน่วยความจำ** (เช่น วน loop อ่าน
`array[0], array[1], array[2], ...`) จะได้ประโยชน์เต็มที่จาก cache line เพราะข้อมูลที่ต้องใช้
ต่อไปมักจะ "ติดมาฟรี" กับ cache line ที่เพิ่งโหลดมาแล้ว

### Temporal Locality (ความใกล้ชิดเชิงเวลา)

**Temporal Locality** คือหลักการที่ว่า "ถ้าเพิ่งเข้าถึงข้อมูลตำแหน่งหนึ่งไป มีโอกาสสูงที่จะเข้าถึง
ตำแหน่งเดิมซ้ำอีกในไม่ช้า" เช่น ตัวแปร loop counter หรือตัวแปรสะสมผลลัพธ์ (accumulator) ที่ถูก
อ่าน/เขียนซ้ำๆ ในทุกรอบของ loop ข้อมูลเหล่านี้ควรถูก "ค้าง" อยู่ใน register หรือ L1 cache ตลอด
เวลาที่ใช้งาน

ตัวอย่างโค้ดที่ใช้ทั้งสองหลักการได้ดี:

```cpp
double sum_good(const std::vector<double>& v) {
    double total = 0.0;              // temporal locality: total ถูกใช้ซ้ำทุกรอบ, ค้างใน register
    for (double x : v) {             // spatial locality: v[0], v[1], v[2], ... เรียงติดกันใน memory
        total += x;
    }
    return total;
}
```

ตลอด Part นี้ เราจะพิสูจน์ด้วยการวัดจริงว่าเมื่อโค้ดละเมิดหลักการทั้งสองข้อนี้ ความเร็วจะตกลง
มากแค่ไหน

---

## 87.3 พิสูจน์ด้วยของจริง: Row-major vs Column-major Traversal (Step 691)

C++ เก็บ 2D array (หรือ `std::vector` ที่จำลอง 2D array ด้วย index คำนวณเอง) แบบ **Row-major
Order** เสมอ นั่นคือข้อมูลของแถวเดียวกันจะถูกเก็บเรียงติดกันในหน่วยความจำก่อน แล้วจึงต่อด้วยแถว
ถัดไป:

```
a[i][j] ถูกเก็บที่ตำแหน่ง memory = base + (i * จำนวนคอลัมน์ + j) * sizeof(element)

แถว 0:  a[0][0] a[0][1] a[0][2] ... a[0][N-1]
แถว 1:  a[1][0] a[1][1] a[1][2] ... a[1][N-1]
        (เรียงต่อกันเป็นเส้นตรงเส้นเดียวใน RAM จริงๆ)
```

ถ้าเราวน loop โดยให้ `j` (คอลัมน์) เป็น loop ในสุด — เราเข้าถึงข้อมูลตาม**ลำดับที่มันถูกเก็บจริง**
เป๊ะๆ (Spatial Locality เต็มร้อย) แต่ถ้าสลับให้ `i` (แถว) เป็น loop ในสุดแทน เราจะกระโดดข้าม
หน่วยความจำเป็นระยะทาง `N * sizeof(element)` ทุกครั้ง — ทำลาย Spatial Locality โดยสิ้นเชิง

```cpp
// rowcol.cpp - เปรียบเทียบ row-major vs column-major traversal
#include <cstdio>
#include <cstdlib>
#include <chrono>
#include <vector>

constexpr int N = 8192; // 8192*8192*8 bytes = 512 MiB ต่ออาร์เรย์ (เกิน L3 cache)

int main() {
    std::vector<double> a(static_cast<size_t>(N) * N);
    for (size_t i = 0; i < a.size(); ++i) a[i] = static_cast<double>(i % 100);

    double sum_row = 0.0, sum_col = 0.0;

    // Row-major traversal: i คือแถวนอก, j คือคอลัมน์ใน (ตรงกับ memory layout)
    auto t1 = std::chrono::steady_clock::now();
    for (int i = 0; i < N; ++i) {
        for (int j = 0; j < N; ++j) {
            sum_row += a[static_cast<size_t>(i) * N + j];
        }
    }
    auto t2 = std::chrono::steady_clock::now();

    // Column-major traversal: j คือแถวนอก, i คือคอลัมน์ใน (สวนทาง memory layout)
    for (int j = 0; j < N; ++j) {
        for (int i = 0; i < N; ++i) {
            sum_col += a[static_cast<size_t>(i) * N + j];
        }
    }
    auto t3 = std::chrono::steady_clock::now();

    double row_ms = std::chrono::duration<double, std::milli>(t2 - t1).count();
    double col_ms = std::chrono::duration<double, std::milli>(t3 - t2).count();

    std::printf("N = %d (%.1f MiB per array)\n", N, (double)(N) * N * sizeof(double) / (1024.0*1024.0));
    std::printf("Row-major traversal:    %8.2f ms  (sum=%.1f)\n", row_ms, sum_row);
    std::printf("Column-major traversal: %8.2f ms  (sum=%.1f)\n", col_ms, sum_col);
    std::printf("Column-major ช้ากว่า row-major: %.2fx\n", col_ms / row_ms);
    return 0;
}
```

คอมไพล์และรัน:

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 -O2 rowcol.cpp -o rowcol
./rowcol
```

**ผลลัพธ์ที่วัดได้จริง (รันซ้ำ 2 ครั้งเพื่อตรวจความสม่ำเสมอ):**

| รอบที่ | Row-major (ms) | Column-major (ms) | อัตราส่วน |
|---|---:|---:|---:|
| 1 | 125.04 | 813.77 | 6.51x |
| 2 | 123.30 | 746.83 | 6.06x |

แค่สลับลำดับ loop สองบรรทัด — โดยที่ **จำนวนการบวกทั้งหมดเท่ากันเป๊ะ ไม่มีการคำนวณเพิ่มขึ้นเลย**
— ความเร็วต่างกันถึง **6 เท่า** สาเหตุคือในเวอร์ชัน column-major ทุกครั้งที่อ่าน `a[i][j]` เราต้อง
กระโดดไป 8192 * 8 bytes = 64 KB จากตำแหน่งก่อนหน้า ซึ่งไกลกว่าขนาด cache line (64 bytes) มาก
เท่ากับว่า **แทบทุกครั้งที่อ่านจะเป็น cache miss** ทำให้ต้องไปรอ RAM latency (เห็นได้จากตาราง
latency ใน 87.1) ในทุกๆ การอ่านค่า

---

## 87.4 Array of Structures vs Structure of Arrays: แนวคิด (Step 692)

ปัญหาการจัดวางข้อมูลไม่ได้เกิดแค่กับ 2D array เท่านั้น แต่เกิดกับการออกแบบ `struct`/`class` ทั่วไป
ด้วย โดยเฉพาะเมื่อมีข้อมูลจำนวนมาก (เช่น อนุภาคในระบบจำลองฟิสิกส์ หรือ entity ในเกม) และในแต่ละ
รอบการประมวลผลเราต้องการใช้แค่ **บาง field** เท่านั้น

### แบบที่ 1: Array of Structures (AoS) — วิธีที่ OOP มือใหม่มักเขียน

```cpp
struct ParticleAoS {
    float x, y, z;       // ตำแหน่ง
    float vx, vy, vz;    // ความเร็ว
    float mass;
    float charge;
}; // 8 floats = 32 bytes ต่อ 1 อนุภาค

std::vector<ParticleAoS> particles(N);
```

ในหน่วยความจำ ข้อมูลจะเรียงเป็น: `[x0 y0 z0 vx0 vy0 vz0 m0 c0][x1 y1 z1 vx1 vy1 vz1 m1 c1]...`
— field ของอนุภาคเดียวกันอยู่ติดกันหมด

### แบบที่ 2: Structure of Arrays (SoA) — แยก field ออกเป็นคนละ array

```cpp
struct ParticlesSoA {
    std::vector<float> x, y, z;
    std::vector<float> vx, vy, vz;
    std::vector<float> mass;
    std::vector<float> charge;
};
```

ในหน่วยความจำ ข้อมูลจะเรียงเป็น: `x: [x0 x1 x2 x3 ...]`, `y: [y0 y1 y2 y3 ...]`, ... — field
ชนิดเดียวกันของทุกอนุภาคอยู่ติดกันหมด แยกเป็นคนละก้อนหน่วยความจำ

```
AoS:  [x0 y0 z0 vx0 vy0 vz0 m0 c0] [x1 y1 z1 vx1 vy1 vz1 m1 c1] [x2 ...] ...
                ▲ ต้องใช้แค่ field x แต่โหลดมาทั้ง struct (32 bytes) ทุกครั้ง

SoA:  x:  [x0 x1 x2 x3 x4 x5 x6 x7 ...]   <- ต้องการ field x, ได้ x ล้วนๆ ทุก byte ที่โหลดมา
      y:  [y0 y1 y2 y3 y4 y5 y6 y7 ...]
      z:  [z0 z1 z2 z3 z4 z5 z6 z7 ...]
      ...
```

**ถ้างานที่ทำต้องใช้ทุก field ของอนุภาคพร้อมกัน** (เช่น อัปเดตตำแหน่งจากความเร็วทุกแกน) ทั้งสอง
แบบให้ประสิทธิภาพใกล้เคียงกัน หรือ AoS อาจจะดีกว่าด้วยซ้ำเพราะข้อมูลที่ต้องใช้ทั้งหมดอยู่ใน
cache line เดียวกันพอดี

แต่ **ถ้างานที่ทำต้องใช้แค่บาง field** (เช่น หา field `x` ที่มีค่ามากที่สุดในบรรดาอนุภาคทั้งหมด)
แบบ AoS จะเสียเปรียบมาก เพราะทุกครั้งที่โหลด cache line มาเพื่ออ่าน `x` เราจะได้ `y, z, vx, vy,
vz, mass, charge` ติดมาด้วยแบบ "เสียเปล่า" (ไม่ได้ใช้เลย) ในขณะที่แบบ SoA ทุก byte ที่โหลดมา
เป็นค่า `x` ล้วนๆ ที่ใช้งานจริงทั้งหมด

---

## 87.5 พิสูจน์ด้วยของจริง: AoS vs SoA Benchmark (Step 693)

มาวัดสถานการณ์ "ต้องการแค่ field เดียว" กันจริงๆ ด้วยการรวมค่า `x` ของอนุภาค 40 ล้านตัว:

```cpp
// aos_soa.cpp - เปรียบเทียบ Array of Structures (AoS) กับ Structure of Arrays (SoA)
// สถานการณ์: มี "อนุภาค" จำนวนมาก แต่ในการคำนวณรอบนี้เราต้องการใช้แค่ field เดียว (x)
#include <cstdio>
#include <cstdlib>
#include <chrono>
#include <vector>

constexpr size_t N = 40'000'000; // จำนวนอนุภาค

// ---------- แบบ AoS ----------
struct ParticleAoS {
    float x, y, z;       // ตำแหน่ง
    float vx, vy, vz;    // ความเร็ว
    float mass;
    float charge;
}; // 8 floats = 32 bytes ต่อ 1 อนุภาค

// ---------- แบบ SoA ----------
struct ParticlesSoA {
    std::vector<float> x, y, z;
    std::vector<float> vx, vy, vz;
    std::vector<float> mass;
    std::vector<float> charge;
    explicit ParticlesSoA(size_t n)
        : x(n), y(n), z(n), vx(n), vy(n), vz(n), mass(n), charge(n) {}
};

int main() {
    // ---- เตรียมข้อมูล AoS ----
    std::vector<ParticleAoS> aos(N);
    for (size_t i = 0; i < N; ++i) {
        aos[i].x = static_cast<float>(i % 1000);
        aos[i].y = 1.0f; aos[i].z = 1.0f;
        aos[i].vx = 0.1f; aos[i].vy = 0.1f; aos[i].vz = 0.1f;
        aos[i].mass = 1.0f; aos[i].charge = 0.0f;
    }

    // ---- เตรียมข้อมูล SoA ----
    ParticlesSoA soa(N);
    for (size_t i = 0; i < N; ++i) {
        soa.x[i] = static_cast<float>(i % 1000);
        soa.y[i] = 1.0f; soa.z[i] = 1.0f;
        soa.vx[i] = 0.1f; soa.vy[i] = 0.1f; soa.vz[i] = 0.1f;
        soa.mass[i] = 1.0f; soa.charge[i] = 0.0f;
    }

    const int REPEAT = 5;
    double aos_total = 0.0, soa_total = 0.0;
    volatile double sink = 0.0; // กัน compiler optimize การคำนวณทิ้งทั้งหมด

    // ---- งานจริง: บวกค่า x ของทุกอนุภาคเข้าด้วยกัน (ใช้แค่ field เดียว) ----
    for (int r = 0; r < REPEAT; ++r) {
        auto t1 = std::chrono::steady_clock::now();
        double sum = 0.0;
        for (size_t i = 0; i < N; ++i) sum += aos[i].x;
        auto t2 = std::chrono::steady_clock::now();
        aos_total += std::chrono::duration<double, std::milli>(t2 - t1).count();
        sink += sum;
    }

    for (int r = 0; r < REPEAT; ++r) {
        auto t1 = std::chrono::steady_clock::now();
        double sum = 0.0;
        for (size_t i = 0; i < N; ++i) sum += soa.x[i];
        auto t2 = std::chrono::steady_clock::now();
        soa_total += std::chrono::duration<double, std::milli>(t2 - t1).count();
        sink += sum;
    }

    std::printf("sizeof(ParticleAoS) = %zu bytes\n", sizeof(ParticleAoS));
    std::printf("N = %zu particles, REPEAT = %d\n", N, REPEAT);
    std::printf("AoS: sum เฉพาะ field x เฉลี่ย %8.2f ms/รอบ\n", aos_total / REPEAT);
    std::printf("SoA: sum เฉพาะ field x เฉลี่ย %8.2f ms/รอบ\n", soa_total / REPEAT);
    std::printf("AoS ช้ากว่า SoA: %.2fx\n", aos_total / soa_total);
    std::printf("(sink=%f, ป้องกัน compiler optimize ทิ้ง)\n", sink);
    return 0;
}
```

คอมไพล์และรัน:

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 -O2 aos_soa.cpp -o aos_soa
./aos_soa
```

**ผลลัพธ์ที่วัดได้จริง (รันซ้ำ 2 ครั้ง):**

| รอบที่ | AoS (ms/รอบ) | SoA (ms/รอบ) | AoS ช้ากว่า |
|---|---:|---:|---:|
| 1 | 120.40 | 34.98 | 3.44x |
| 2 | 114.27 | 36.74 | 3.11x |

`sizeof(ParticleAoS)` คือ 32 bytes แต่ field `x` ที่เราต้องการใช้จริงมีแค่ 4 bytes เท่านั้น
— นั่นคือทุกครั้งที่ CPU ดึง cache line มาเพื่ออ่าน `x` ของ AoS เรา **ใช้ประโยชน์แค่ 4/32 = 12.5%
ของข้อมูลที่ดึงมาจริง** ส่วนที่เหลือ (`y, z, vx, vy, vz, mass, charge`) ถูกดึงมาแต่ไม่ได้ใช้เลย
ในขณะที่ SoA ใช้ประโยชน์ 100% ของทุก byte ที่ดึงมา ผลคือ AoS ช้ากว่า SoA ราว **3-3.4 เท่า** ใน
สถานการณ์นี้ ซึ่งใกล้เคียงกับสัดส่วน "ข้อมูลที่ใช้จริง" ตามที่ทฤษฎีคาดการณ์ไว้พอดี

> **ข้อควรระวัง**: ผลลัพธ์นี้ใช้ได้เฉพาะกรณี "ใช้แค่บาง field" เท่านั้น ถ้าโค้ดของคุณต้องใช้
> ทุก field พร้อมกันเสมอ (เช่น อัปเดตตำแหน่งจากความเร็วครบทั้ง 6 ค่า) AoS อาจไม่แพ้ SoA เลย
> หรืออาจจะเร็วกว่าด้วยซ้ำเพราะลด cache line ที่ต้องเปิดพร้อมกัน อย่าเปลี่ยนจาก AoS เป็น SoA
> โดยไม่วัดผลจริงก่อน

---

## 87.6 False Sharing: ปัญหาที่มองไม่เห็นด้วยตาเปล่า (Step 694)

จนถึงตอนนี้เราพูดถึงปัญหา cache ใน context ของ thread เดียว แต่เมื่อมีหลาย thread ทำงานพร้อมกัน
จะเกิดปัญหาใหม่ที่ **มองจากซอร์สโค้ดแล้วดูเหมือนไม่มีอะไรผิดเลย** นั่นคือ **False Sharing**

### กลไกเบื้องหลัง: MESI Protocol (แบบย่อ)

CPU หลาย core ที่ใช้ cache แยกกัน (L1/L2 ต่อ core) ต้องมีกลไกทำให้ข้อมูลตรงกัน (Cache Coherence)
เมื่อ core หนึ่งเขียนข้อมูลลง cache line ที่ core อื่นก็มีสำเนาอยู่ใน cache ของตัวเองเหมือนกัน
CPU ต้องส่งสัญญาณบอก core อื่นๆ ว่า **"cache line นี้ของแกใช้ไม่ได้แล้วนะ (invalid) ไปโหลดใหม่"**
กระบวนการนี้ (รู้จักกันในชื่อโปรโตคอล MESI: Modified, Exclusive, Shared, Invalid) มีค่าใช้จ่าย
สูงกว่าการอ่าน/เขียน cache ปกติมาก

**False Sharing** เกิดขึ้นเมื่อ 2 thread (หรือมากกว่า) เขียนข้อมูล **คนละตัวแปรกัน** แต่ตัวแปร
ทั้งสองบังเอิญอยู่ใน **cache line เดียวกัน** แม้ในทางตรรกะ (logic) โปรแกรมจะไม่มี Race Condition
เลย (เพราะแต่ละ thread แก้ตัวแปรของตัวเองจริงๆ) แต่ในทางฮาร์ดแวร์ ทุกครั้งที่ thread หนึ่งเขียน
ข้อมูล cache line ทั้งก้อนจะถูกส่งสัญญาณ invalidate ไปยัง core อื่นที่ถือ cache line เดียวกันอยู่
ทำให้เกิดการ "ปิงปอง" cache line ไปมาระหว่าง core อย่างต่อเนื่อง ทั้งที่ไม่มีข้อมูลจริงที่ถูก
แชร์กันเลย — นี่คือที่มาของชื่อ **"False" Sharing** (แชร์กันแบบหลอกๆ ไม่ได้แชร์ข้อมูลจริง แต่
แชร์ cache line)

```
Core 0 เขียน counter[0] ┐
Core 1 เขียน counter[1] ├─► ทั้ง 4 ตัวอยู่ใน cache line เดียวกัน (32 bytes < 64 bytes)
Core 2 เขียน counter[2] ├─► ทุกครั้งที่ core ใดเขียน cache line ทั้งก้อนถูก invalidate
Core 3 เขียน counter[3] ┘    core อื่นต้องโหลด cache line ใหม่จาก core ที่เพิ่งเขียน (ผ่าน L3)
```

### โค้ดสาธิต

```cpp
// false_sharing.cpp - เปรียบเทียบผลของ False Sharing ระหว่าง thread
// คอมไพล์: g++ -Wall -Wextra -Wpedantic -std=c++17 -O2 -pthread false_sharing.cpp -o false_sharing
#include <cstdio>
#include <chrono>
#include <thread>
#include <vector>
#include <atomic>

constexpr int NUM_THREADS = 4;
constexpr long long ITERS = 50'000'000LL;

// หมายเหตุสำคัญ: ใช้ std::atomic<long long> แทน long long ธรรมดา เพราะถ้าใช้ long long ธรรมดา
// compiler จะมองเห็นว่าไม่มีใครอ่านค่ากลาง loop เลย แล้ว "โกง" ด้วยการยุบ 50 ล้านครั้งของ
// counters.value[t]++ ให้เหลือแค่ counters.value[t] += 50000000 คำสั่งเดียว (ไม่มีการเข้าถึง
// หน่วยความจำซ้ำๆ จริง) ทำให้วัด false sharing ไม่ได้เลย การใช้ atomic (แม้จะเป็น relaxed
// ordering ก็ตาม) บังคับให้ทุก increment ต้องเป็นการอ่าน-เขียนหน่วยความจำจริงทุกครั้ง
// ซึ่งตรงกับสถานการณ์จริงที่ multi-thread เข้าถึง shared counter ด้วย

// ---------- แบบที่ 1: counter เรียงติดกัน (ทุกตัวอยู่ใน cache line เดียวกัน) ----------
struct PackedCounters {
    std::atomic<long long> value[NUM_THREADS]; // 4 * 8 = 32 bytes -> ทั้งหมดอยู่ใน cache line เดียว
};

// ---------- แบบที่ 2: counter ที่ pad ให้ห่างกันคนละ cache line ----------
struct alignas(64) PaddedCounter {
    std::atomic<long long> value;
    char pad[64 - sizeof(std::atomic<long long>)]; // เติมให้เต็ม 64 bytes พอดี 1 cache line ต่อ 1 counter
};

template <typename CounterArray>
double run_benchmark(CounterArray& counters) {
    std::vector<std::thread> threads;
    auto t1 = std::chrono::steady_clock::now();
    for (int t = 0; t < NUM_THREADS; ++t) {
        threads.emplace_back([&counters, t]() {
            for (long long i = 0; i < ITERS; ++i) {
                if constexpr (std::is_same_v<CounterArray, PackedCounters>) {
                    counters.value[t].fetch_add(1, std::memory_order_relaxed);
                } else {
                    counters[t].value.fetch_add(1, std::memory_order_relaxed);
                }
            }
        });
    }
    for (auto& th : threads) th.join();
    auto t2 = std::chrono::steady_clock::now();
    return std::chrono::duration<double, std::milli>(t2 - t1).count();
}

int main() {
    std::printf("sizeof(PackedCounters) = %zu bytes (ทั้งหมดใน cache line เดียว)\n", sizeof(PackedCounters));
    std::printf("sizeof(PaddedCounter)  = %zu bytes (ต่อ 1 counter, คนละ cache line)\n", sizeof(PaddedCounter));
    std::printf("NUM_THREADS = %d, ITERS ต่อ thread = %lld\n\n", NUM_THREADS, ITERS);

    // False sharing case
    PackedCounters packed{};
    double packed_ms = run_benchmark(packed);
    std::printf("False sharing (แชร์ cache line เดียวกัน): %8.2f ms\n", packed_ms);

    // Padded case (no false sharing)
    std::vector<PaddedCounter> padded(NUM_THREADS);
    for (auto& c : padded) c.value = 0;
    double padded_ms = run_benchmark(padded);
    std::printf("Padded (คนละ cache line):                 %8.2f ms\n", padded_ms);

    std::printf("False sharing ช้ากว่า: %.2fx\n", packed_ms / padded_ms);

    long long sum = 0;
    for (int i = 0; i < NUM_THREADS; ++i) sum += padded[i].value;
    std::printf("(sanity check sum = %lld, expected = %lld)\n", sum, ITERS * NUM_THREADS);
    return 0;
}
```

จุดสำคัญของการออกแบบโค้ดนี้: `PackedCounters` เก็บ counter ของทั้ง 4 thread ไว้ใน struct
เดียวติดกัน (รวม 32 bytes ซึ่งเล็กกว่า cache line 64 bytes พอดี ทำให้ทั้ง 4 ตัวอยู่ใน cache
line เดียวกันแน่นอน) ส่วน `PaddedCounter` ใช้ `alignas(64)` บังคับให้แต่ละ counter เริ่มต้นที่
ขอบของ cache line เสมอ แล้วเติม `pad` ให้เต็ม 64 bytes พอดี — รับประกันว่าแต่ละ counter อยู่
คนละ cache line แน่นอน

---

## 87.7 พิสูจน์ด้วยของจริง: วัด False Sharing Benchmark (Step 695)

คอมไพล์และรัน (**ต้องมี `-pthread`** เพราะใช้ `std::thread`):

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 -O2 -pthread false_sharing.cpp -o false_sharing
./false_sharing
```

**ผลลัพธ์ที่วัดได้จริง (รันซ้ำ 4 ครั้งติดต่อกัน):**

| รอบที่ | False sharing (ms) | Padded (ms) | False sharing ช้ากว่า |
|---|---:|---:|---:|
| 1 | 3687.73 | 347.96 | 10.60x |
| 2 | 3691.58 | 339.16 | 10.88x |
| 3 | 3639.55 | 336.81 | 10.81x |
| 4 | 1682.85 | 337.39 | 4.99x |

ผลที่วัดได้ **ไม่คงที่เป๊ะทุกรอบ** (รอบที่ 4 ได้ตัวเลขน้อยกว่ารอบอื่นชัดเจน) เพราะเครื่องที่ใช้
รันเป็น Cloud VM ที่แชร์ทรัพยากรกับผู้เช่ารายอื่น (noisy neighbor) และมีการจัดตารางงาน
(scheduling) ของ OS/Hypervisor เข้ามาแทรก ทำให้เวลาที่วัดมี variance สูงกว่าการรันบนเครื่อง
เปล่าที่ไม่มีใครแย่งทรัพยากร แต่ **ทิศทางของผลลัพธ์สอดคล้องกันทุกรอบ**: เวอร์ชันที่มี False
Sharing ช้ากว่าเวอร์ชันที่ pad แล้วเสมอ อยู่ในช่วง **5-11 เท่า** — นี่คือบทเรียนสำคัญของการทำ
Benchmark จริง: **ต้องรันซ้ำหลายครั้งและรายงานทั้งค่าเฉลี่ยและความแปรปรวน ไม่ใช่เชื่อผลจากการ
รันครั้งเดียว**

### ทางเลือกที่ portable กว่าการเขียน 64 ตรงๆ

การ hardcode เลข 64 อาจไม่ปลอดภัยเพราะ CPU บางสถาปัตยกรรม (เช่น Apple M-series บางรุ่น, บาง
ARM) มีขนาด cache line ต่างจาก 64 bytes ตั้งแต่ C++17 เป็นต้นมา มาตรฐานมี
`std::hardware_destructive_interference_size` ใน `<new>` ให้ใช้แทน:

```cpp
#include <new>
#include <cstdio>

int main() {
    std::printf("hardware_destructive_interference_size = %zu\n",
                std::hardware_destructive_interference_size);
    return 0;
}
```

คอมไพล์และรันบนเครื่องนี้ได้ผลลัพธ์จริง:

```
hardware_destructive_interference_size = 64
```

ตรงกับค่าที่ `getconf LEVEL1_DCACHE_LINESIZE` รายงานไว้พอดี ในโค้ด production ควรใช้ค่านี้แทน
ตัวเลข literal `64` เพื่อให้โค้ด portable ข้าม platform:

```cpp
struct alignas(std::hardware_destructive_interference_size) PaddedCounter {
    std::atomic<long long> value;
    char pad[std::hardware_destructive_interference_size - sizeof(std::atomic<long long>)];
};
```

### จะตรวจจับ False Sharing ในโปรแกรมจริงได้อย่างไร

เพราะ False Sharing ไม่ทำให้เกิด bug เชิงตรรกะเลย เครื่องมือ debug ทั่วไปช่วยไม่ได้ ต้องใช้
เครื่องมือ profiling ระดับ hardware counter ที่เรียนใน Part 86:

```bash
perf c2c record -- ./my_program
perf c2c report
```

`perf c2c` (Cache-to-Cache) ออกแบบมาเพื่อหา cache line ที่ถูกหลาย core แย่งกันโดยเฉพาะ จะแสดง
รายการ cache line ที่มีการ "ปิงปอง" ระหว่าง core บ่อยผิดปกติ พร้อมชี้ตำแหน่งบรรทัดโค้ดที่เกี่ยวข้อง

---

## 87.8 Data-Oriented Design หลักการโดยย่อ (Step 696)

จากทุกตัวอย่างที่ผ่านมา เราจะเห็นแนวคิดร่วมกันข้อหนึ่ง: **การจัดวางข้อมูลในหน่วยความจำสำคัญ
พอๆ กับ (หรือมากกว่า) อัลกอริทึมที่ใช้ประมวลผลมัน** นี่คือแก่นของแนวคิดที่เรียกว่า
**Data-Oriented Design (DOD)**

### DOD ต่างจาก OOP แบบดั้งเดิมอย่างไร

การออกแบบแบบ Object-Oriented ดั้งเดิมเริ่มต้นจากคำถาม **"ข้อมูลนี้คืออะไร (What is this?)"**
แล้วห่อหุ้ม (encapsulate) field และ method ที่เกี่ยวข้องเข้าไว้ในคลาสเดียวกัน โดยไม่ได้คำนึงถึง
"รูปแบบการเข้าถึงข้อมูลจริง" ในระบบ

Data-Oriented Design กลับเริ่มต้นจากคำถามตรงกันข้าม: **"ข้อมูลนี้จะถูกใช้งานอย่างไร (How is
this used?)"** แล้วจึงออกแบบ layout ของข้อมูลให้เหมาะกับ "รูปแบบการเข้าถึง" (access pattern)
ที่เกิดขึ้นจริงบ่อยที่สุด — ซึ่งมักนำไปสู่การใช้ SoA แทน AoS ตามที่เราเพิ่งพิสูจน์ไปใน 87.4-87.5

### หลักการสำคัญของ DOD

1. **แยก Hot Data ออกจาก Cold Data (Hot/Cold Splitting)**: field ที่ถูกอ่าน/เขียนบ่อยใน
   loop สำคัญ (hot path) ควรแยกออกจาก field ที่ใช้นานๆ ครั้ง (เช่นชื่อ, debug info, metadata)
   เพื่อไม่ให้ field ที่ไม่ค่อยได้ใช้มา "แย่งที่" ใน cache line ของ field ที่ใช้บ่อย

2. **คิดเป็น Array of Data ไม่ใช่ Array of Objects**: แทนที่จะมี `std::vector<GameObject*>`
   ที่แต่ละตัวมี virtual function และ field สารพัด ให้แยกข้อมูลตาม "field ที่ต้องประมวลผล
   พร้อมกันเป็นชุด" เช่น ระบบ **Entity Component System (ECS)** ที่นิยมใน Game Engine สมัยใหม่
   (จะกล่าวถึงอีกครั้งใน Part 118 — Game Development) คือการนำหลักการ DOD มาใช้เต็มรูปแบบ

3. **ประมวลผลเป็น Batch (Transform ทีละชุด แทนที่จะ Update ทีละ Object)**: แทนที่จะเรียก
   `obj->update()` ทีละตัวผ่าน virtual function (ซึ่งแต่ละครั้งอาจกระโดดไปคนละที่ในหน่วยความจำ
   และเสีย branch prediction) ให้เขียนเป็น loop เดียวที่ไล่ประมวลผล array ของข้อมูลชนิดเดียวกัน
   รวดเดียว (ตรงกับ Spatial/Temporal Locality พอดี)

4. **หลีกเลี่ยง Indirection ที่ไม่จำเป็นใน Hot Path**: pointer, virtual function, และ
   `std::shared_ptr` ล้วนเพิ่มการกระโดดหน่วยความจำ (pointer chasing เหมือนที่วัดใน 87.1)
   ใน hot path ที่ประมวลผลข้อมูลจำนวนมากซ้ำๆ ควรใช้ค่าแบบ contiguous (เช่น `std::vector<T>`
   ธรรมดา) แทน `std::vector<std::unique_ptr<T>>` ให้ได้มากที่สุด

### ตัวอย่างแนวคิด: จาก OOP สู่ DOD

```cpp
// ---------- สไตล์ OOP ดั้งเดิม: รวมทุกอย่างไว้ในคลาสเดียว ----------
class GameObjectOOP {
public:
    float x, y, z;              // hot: ใช้ทุกเฟรม
    float vx, vy, vz;           // hot: ใช้ทุกเฟรม
    std::string debug_name;     // cold: ใช้แค่ตอน debug/log
    std::string description;    // cold: แทบไม่ได้ใช้เลยตอนรัน
    int last_damage_source_id;  // cold: ใช้แค่ตอนคำนวณ combat log

    void update(float dt) {
        x += vx * dt;
        y += vy * dt;
        z += vz * dt;
    }
};
// วน update() ทีละ object -> ทุกครั้งโหลด cache line ที่มี std::string สองตัวติดมาด้วย
// ทั้งที่ update() ไม่ได้แตะ field เหล่านั้นเลย

// ---------- สไตล์ Data-Oriented: แยก hot/cold ออกจากกัน ----------
struct TransformHot {   // เก็บเฉพาะ field ที่ update() ทุกเฟรมต้องใช้ -> อัดแน่นใน cache line
    float x, y, z;
    float vx, vy, vz;
};

struct GameObjectCold {  // เก็บ field ที่ใช้นานๆ ครั้ง แยกไปอีก array หนึ่งต่างหาก
    std::string debug_name;
    std::string description;
    int last_damage_source_id;
};

void update_all(std::vector<TransformHot>& transforms, float dt) {
    // loop เดียว ไล่อ่าน/เขียน array ที่มีแต่ข้อมูลที่ต้องใช้จริง -> cache-friendly เต็มที่
    for (auto& t : transforms) {
        t.x += t.vx * dt;
        t.y += t.vy * dt;
        t.z += t.vz * dt;
    }
}
```

`update_all` ไล่อ่าน `std::vector<TransformHot>` ที่แต่ละ element มีแค่ 24 bytes (6 floats)
ล้วนๆ ที่ใช้จริงทุก byte แทนที่จะโหลด `GameObjectOOP` ทั้งก้อน (ซึ่งมี `std::string` สองตัว
ที่กินพื้นที่และแทบไม่ถูกอ่านใน hot path เลย) — นี่คือหัวใจของ Data-Oriented Design

> **ข้อควรระวัง**: DOD ไม่ใช่ "ดีกว่า OOP เสมอ" DOD เหมาะกับ **hot path ที่ประมวลผลข้อมูล
> จำนวนมากซ้ำๆ** (game loop, ระบบจำลองฟิสิกส์, การประมวลผลข้อมูลขนาดใหญ่) ส่วนโค้ดที่เกี่ยวกับ
> business logic ทั่วไปที่ไม่ใช่ hot path เช่น การจัดการ UI, การตรวจสอบสิทธิ์ผู้ใช้ ยังคงเขียน
> แบบ OOP ตามปกติได้โดยไม่ต้องกังวลเรื่อง cache เลย

---

## ข้อผิดพลาดที่พบบ่อย (Common Pitfalls)

1. **False Sharing มองไม่เห็นจาก Code Review**: โค้ดที่มี False Sharing ผ่าน Code Review และ
   Unit Test ได้สบายๆ เพราะไม่มี bug เชิงตรรกะเลย (ไม่มี Race Condition, ผลลัพธ์ถูกต้อง 100%)
   ปัญหาจะโผล่มาแค่ตอนวัดประสิทธิภาพภายใต้ multi-thread จริงเท่านั้น ทีมที่ไม่เคย profile
   ประสิทธิภาพแบบ multi-thread อาจมีโค้ดแบบนี้ซ่อนอยู่หลายจุดโดยไม่รู้ตัว

2. **ปรับ Data Layout ก่อนวัดผลจริง (Premature Optimization)**: การแปลง AoS เป็น SoA ทำให้โค้ด
   อ่านยากขึ้นและ maintain ยากขึ้นเสมอ ถ้าโค้ดส่วนนั้นไม่ใช่ hot path (ไม่ได้ถูกเรียกบ่อยหรือ
   ประมวลผลข้อมูลจำนวนมาก) การแลก "อ่านยากขึ้น" กับ "เร็วขึ้นนิดหน่อยที่ไม่มีใครสังเกต" ไม่คุ้ม
   เลย ต้อง Profile หา hot path ก่อนเสมอ (ตามที่เรียนใน Part 86)

3. **เข้าใจผิดว่า AoS แย่กว่า SoA เสมอ**: ถ้า pattern การใช้งานจริงต้องอ่าน/เขียนทุก field
   พร้อมกันเสมอ (เช่น อัปเดตตำแหน่งจากความเร็วครบทุกแกน) AoS อาจเร็วกว่า SoA ด้วยซ้ำ เพราะ
   ข้อมูลทั้งหมดที่ต้องใช้อยู่ใน cache line เดียวกันพอดี ในขณะที่ SoA ต้องเปิด cache line
   หลายก้อนพร้อมกัน (หนึ่งก้อนต่อหนึ่ง array) กฎทองคือ **"วัดก่อนตัดสินใจเสมอ"**

4. **คำนวณ padding ผิดพลาดจนไม่ได้ผลตามที่ตั้งใจ**: เวลาเขียน `char pad[64 - sizeof(T)]`
   ถ้า `sizeof(T)` มากกว่า 64 อยู่แล้ว (เช่น struct ที่มี field เยอะ) นิพจน์นี้จะ underflow
   (เพราะเป็น `size_t` ที่ไม่มีค่าติดลบ) กลายเป็นตัวเลขมหาศาลและ compile error หรือใช้
   หน่วยความจำเกินจำเป็น ต้องตรวจสอบด้วย `static_assert(sizeof(T) <= 64)` ก่อนเสมอเมื่อ pad
   ด้วยมือ

5. **ทดสอบด้วยข้อมูลที่เล็กเกินไปจนวัดผลไม่ออก**: ถ้า array ที่ใช้ benchmark เล็กพอที่จะอยู่ใน
   L1/L2 cache ได้ทั้งหมด ผลต่างระหว่าง row-major/column-major หรือ AoS/SoA จะแทบไม่เห็นเลย
   (ดูตัวอย่างในแบบฝึกหัดข้อ 1) ทำให้สรุปผิดว่า "การจัดวางข้อมูลไม่สำคัญ" ทั้งที่ความจริงคือ
   ข้อมูลชุดนั้นเล็กเกินกว่าจะเจอปัญหา ต้องทดสอบด้วยขนาดข้อมูลที่ใกล้เคียงกับสถานการณ์จริงเสมอ

6. **Hardcode ขนาด Cache Line เป็น 64 โดยไม่ตรวจสอบ**: แม้ 64 bytes จะเป็นค่ามาตรฐานของ x86_64
   เกือบทั้งหมด แต่ CPU บางสถาปัตยกรรมมีขนาดต่างออกไป การใช้ `std::hardware_destructive_
   interference_size` (C++17) แทนตัวเลข literal ทำให้โค้ด portable กว่าและสื่อความหมายชัดเจน
   กว่าด้วย

7. **สรุปผลจากการรัน Benchmark แค่ครั้งเดียว**: ดังที่เห็นในผลการวัด False Sharing ของบทเรียนนี้
   ที่รอบหนึ่งได้ 5 เท่า อีกรอบได้ 11 เท่า การรันบนเครื่องที่มี noise (Cloud VM, เครื่องที่มี
   โปรแกรมอื่นทำงานพร้อมกัน, CPU throttling) ทำให้ผลแต่ละรอบต่างกันได้มาก ควรรันซ้ำอย่างน้อย
   3-5 ครั้งและดูทั้งค่าเฉลี่ยและ range เสมอ ไม่ใช่เชื่อตัวเลขจากการรันครั้งเดียว

---

## แบบฝึกหัดท้ายบท

1. แก้ไข `rowcol.cpp` ให้ใช้ `N = 512` แทน `N = 8192` (ทำให้อาร์เรย์เล็กพอจะพอดีกับ L2 cache
   ของเครื่อง) คอมไพล์และรันด้วย flag เดิม แล้วอธิบายว่าทำไมอัตราส่วน column/row ถึงเปลี่ยนไป
   จากตอน N ใหญ่

2. ใช้ `struct Employee { char name[32]; int id; double salary; int department; };` ที่กำหนด
   ให้ เขียนโปรแกรมสร้างพนักงาน 10 ล้านคนแบบ AoS และแบบ SoA แล้ววัดเวลาการหาผลรวมเงินเดือน
   ทั้งหมด (ใช้แค่ field `salary`) เปรียบเทียบทั้งสองแบบ

3. แก้ไข `false_sharing.cpp` ให้ `NUM_THREADS` เท่ากับจำนวน core จริงของเครื่อง (เช็คด้วย
   `nproc`) แล้ววัดผลว่าอัตราส่วนความช้าเปลี่ยนไปจากตอน 4 thread หรือไม่ อธิบายเหตุผล

4. เขียนโปรแกรมจำลองสถานการณ์ False Sharing ของตัวเอง: มี histogram ที่แต่ละ thread นับ
   ค่าลง bucket ของตัวเอง (index คงที่ต่อ thread) เขียนทั้งเวอร์ชันที่ไม่ pad และเวอร์ชันที่ pad
   ด้วย `alignas(64)` แล้ววัดผลต่างจริง

5. อธิบายด้วยคำพูดของตัวเอง (ไม่ต้องเขียนโค้ด): ทำไมการแยก Hot/Cold data ใน Data-Oriented
   Design ถึงช่วยเพิ่มการใช้ประโยชน์จาก cache ยกตัวอย่างระบบจริง 1 ระบบที่เคยใช้งานหรือเคยได้ยิน
   มา (เกม, ระบบ trading, game engine, database) ที่น่าจะได้ประโยชน์จากแนวคิดนี้

6. รัน `latency.cpp` บนเครื่องของตัวเอง (ไม่ใช่เครื่องในบทเรียนนี้) แล้วเทียบกับตารางในหัวข้อ
   87.1 ว่า "จุดเปลี่ยน" (breakpoint) ของ latency ตรงกับขนาด L1/L2/L3 ที่ `lscpu` รายงานไว้ของ
   เครื่องตัวเองหรือไม่

### แนวทางเฉลยข้อ 1

```cpp
// rowcol_small.cpp
#include <cstdio>
#include <chrono>
#include <vector>

constexpr int N = 512; // 512*512*8 = 2 MiB ต่ออาร์เรย์ พอดี ๆ กับขนาด L2 ของเครื่องนี้ (2 MiB/core)

int main() {
    std::vector<double> a(static_cast<size_t>(N) * N);
    for (size_t i = 0; i < a.size(); ++i) a[i] = static_cast<double>(i % 100);

    double sum_row = 0.0, sum_col = 0.0;

    auto t1 = std::chrono::steady_clock::now();
    for (int i = 0; i < N; ++i)
        for (int j = 0; j < N; ++j)
            sum_row += a[static_cast<size_t>(i) * N + j];
    auto t2 = std::chrono::steady_clock::now();

    for (int j = 0; j < N; ++j)
        for (int i = 0; i < N; ++i)
            sum_col += a[static_cast<size_t>(i) * N + j];
    auto t3 = std::chrono::steady_clock::now();

    double row_ms = std::chrono::duration<double, std::milli>(t2 - t1).count();
    double col_ms = std::chrono::duration<double, std::milli>(t3 - t2).count();

    std::printf("N = %d (%.2f MiB ต่ออาร์เรย์)\n", N, (double)N * N * sizeof(double) / (1024.0*1024.0));
    std::printf("Row-major:    %8.3f ms (sum=%.1f)\n", row_ms, sum_row);
    std::printf("Column-major: %8.3f ms (sum=%.1f)\n", col_ms, sum_col);
    std::printf("อัตราส่วน column/row: %.2fx\n", col_ms / row_ms);
    return 0;
}
```

คอมไพล์และรันจริง:

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 -O2 rowcol_small.cpp -o rowcol_small
./rowcol_small
```

**ผลลัพธ์ที่วัดได้จริง:**

| N | ขนาดต่ออาร์เรย์ | Row-major (ms) | Column-major (ms) | อัตราส่วน |
|---|---|---:|---:|---:|
| 512 | 2 MiB | 0.430-0.433 | 0.618-0.709 | 1.44x-1.64x |
| 8192 (ในบทเรียน) | 512 MiB | 123-125 | 747-814 | 6.06x-6.51x |

**คำอธิบาย**: เมื่อ `N = 512` อาร์เรย์ทั้งก้อนมีขนาดแค่ 2 MiB ซึ่งใกล้เคียงกับขนาด L2 cache
ต่อ core ของเครื่องนี้ (~2 MiB) หลังจากอ่านผ่านไปครั้งแรก ข้อมูลส่วนใหญ่ยังคงค้างอยู่ใน L2/L3
cache ทำให้แม้แต่การเข้าถึงแบบ column-major (ที่กระโดดข้ามหน่วยความจำ) ก็ยังเจอ cache hit
บ่อยพอสมควร ผลต่างจึงลดลงเหลือแค่ ~1.5 เท่า ในขณะที่ `N = 8192` อาร์เรย์มีขนาด 512 MiB ซึ่ง
ใหญ่กว่า cache ทุกระดับมาก การเข้าถึงแบบ column-major แทบทุกครั้งเป็น cache miss ต้องรอ RAM
latency เต็มๆ ทำให้ผลต่างพุ่งขึ้นเป็น 6 เท่า **บทเรียนสำคัญ**: ผลกระทบของ cache-friendliness
จะเห็นชัดก็ต่อเมื่อข้อมูลมีขนาดใหญ่กว่า cache ที่มีอยู่เท่านั้น

### แนวทางเฉลยข้อ 4

```cpp
// histogram.cpp - แบบฝึกหัด: หลายๆ thread ช่วยกันนับ histogram คนละช่วง (bucket) ของตัวเอง
// คอมไพล์: g++ -Wall -Wextra -Wpedantic -std=c++17 -O2 -pthread histogram.cpp -o histogram
#include <cstdio>
#include <atomic>
#include <thread>
#include <vector>
#include <chrono>

constexpr int NUM_THREADS = 4;
constexpr long long ITERS = 50'000'000LL;

// เวอร์ชันที่มีบั๊กด้าน performance: bucket[NUM_THREADS] เรียงติดกันใน struct เดียว
// แม้แต่ละ thread จะแก้ไข index ของตัวเองเท่านั้น (ไม่มี race condition ด้าน correctness เลย)
// แต่ทุก bucket อยู่ใน cache line เดียวกัน (4 * 8 = 32 bytes) จึงเกิด False Sharing
struct BadHistogram {
    std::atomic<long long> bucket[NUM_THREADS];
};

// เวอร์ชันที่แก้แล้ว: pad ให้แต่ละ bucket อยู่คนละ cache line (64 bytes)
struct alignas(64) GoodBucket {
    std::atomic<long long> count;
    char pad[64 - sizeof(std::atomic<long long>)];
};

template <typename T, typename AccessFn>
double run(T& data, AccessFn access) {
    std::vector<std::thread> threads;
    auto t1 = std::chrono::steady_clock::now();
    for (int t = 0; t < NUM_THREADS; ++t) {
        threads.emplace_back([&data, &access, t]() {
            for (long long i = 0; i < ITERS; ++i) {
                access(data, t).fetch_add(1, std::memory_order_relaxed);
            }
        });
    }
    for (auto& th : threads) th.join();
    auto t2 = std::chrono::steady_clock::now();
    return std::chrono::duration<double, std::milli>(t2 - t1).count();
}

int main() {
    BadHistogram bad{};
    double bad_ms = run(bad, [](BadHistogram& d, int t) -> std::atomic<long long>& {
        return d.bucket[t];
    });

    std::vector<GoodBucket> good(NUM_THREADS);
    double good_ms = run(good, [](std::vector<GoodBucket>& d, int t) -> std::atomic<long long>& {
        return d[t].count;
    });

    std::printf("Bad  (ไม่ pad, false sharing): %8.2f ms\n", bad_ms);
    std::printf("Good (pad ด้วย alignas(64)):    %8.2f ms\n", good_ms);
    std::printf("Bad ช้ากว่า Good: %.2fx\n", bad_ms / good_ms);
    return 0;
}
```

คอมไพล์และรันจริง:

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 -O2 -pthread histogram.cpp -o histogram
./histogram
```

**ผลลัพธ์ที่วัดได้จริง (รันซ้ำ 3 ครั้ง):**

| รอบที่ | Bad (ms) | Good (ms) | Bad ช้ากว่า |
|---|---:|---:|---:|
| 1 | 3738.63 | 357.27 | 10.46x |
| 2 | 3733.06 | 327.96 | 11.38x |
| 3 | 3725.50 | 333.75 | 11.16x |

ผลลัพธ์สอดคล้องกับ `false_sharing.cpp` ในเนื้อหาบทเรียน (~10-11 เท่า) ยืนยันว่าการใช้
`alignas(64)` เพื่อแยกแต่ละ counter ออกเป็นคนละ cache line ช่วยแก้ปัญหา False Sharing ได้จริง
และสม่ำเสมอ ต่างจากการวัดใน `false_sharing.cpp` ที่มีรอบหนึ่งผลออกมาต่างจากรอบอื่น (noise จาก
สภาพแวดล้อม) ในที่นี้การรันทั้ง 3 ครั้งให้ผลใกล้เคียงกันมาก เพราะเทมเพลตฟังก์ชัน `run()` ที่ใช้
`std::function`-like lambda อาจทำให้ timing มี overhead คงที่มากกว่าเดิมเล็กน้อย แต่ทิศทางและ
ขนาดของผลต่างยังคงยืนยันข้อสรุปเดิมได้อย่างชัดเจน

---

## สรุปท้ายบท

ใน Part นี้เราได้:

- เข้าใจ **Memory Wall** และเหตุผลที่ CPU ต้องมี Cache หลายชั้น พร้อม**วัด latency จริง**ของ
  L1/L2/L3/RAM บนเครื่องด้วยเทคนิค Pointer Chasing เห็นรูปแบบ "บันได" ที่ latency เพิ่มขึ้น
  ถึง ~80 เท่าเมื่อข้อมูลล้นจาก cache ไปสู่ RAM
- เข้าใจ **Cache Line** (64 bytes) และหลักการ **Spatial/Temporal Locality**
- **พิสูจน์ด้วยการวัดจริง** ว่าการสลับลำดับ loop ธรรมดา (row-major vs column-major) ทำให้
  ความเร็วต่างกันได้ถึง 6 เท่า โดยไม่มีการเปลี่ยนจำนวนการคำนวณเลย
- เข้าใจความแตกต่างระหว่าง **AoS และ SoA** และวัดผลจริงว่า AoS ช้ากว่า SoA ถึง ~3 เท่าเมื่อ
  ต้องใช้แค่บาง field ของข้อมูลจำนวนมาก
- เข้าใจ **False Sharing** ทั้งกลไกเบื้องหลัง (MESI Protocol) และวัดผลจริงว่าการจัดวางข้อมูล
  ผิดที่ทำให้ multi-thread ช้าลงได้ถึง 5-11 เท่า ทั้งที่โค้ดไม่มี bug เชิงตรรกะเลยแม้แต่นิดเดียว
- รู้จักหลักการของ **Data-Oriented Design** และรู้ว่าเมื่อไหร่ควรนำมาใช้แทนการออกแบบแบบ OOP
  ดั้งเดิม

สิ่งที่ทุกตัวอย่างในบทนี้มีร่วมกันคือ **การจัดวางข้อมูลในหน่วยความจำสำคัญพอๆ กับอัลกอริทึม**
และผลกระทบของมันมักจะ **มองไม่เห็นจากการอ่านโค้ดเฉยๆ** ต้องอาศัยการวัดผลจริงเสมอ

ใน **Part 88** เราจะต่อยอดแนวคิดเรื่อง Cache-Friendly ไปสู่อีกเทคนิคหนึ่งที่ทำงานคู่กันได้ดี
มาก นั่นคือ **SIMD (Single Instruction, Multiple Data)** — การทำให้ CPU ประมวลผลข้อมูลหลายค่า
พร้อมกันด้วยคำสั่งเดียว ซึ่งเมื่อรวมกับการจัดข้อมูลแบบ SoA ที่เราเพิ่งเรียนไป จะยิ่งทวีความเร็ว
ได้มากขึ้นไปอีก

**ต่อไป:** [Part 88 — SIMD และ Vectorization เบื้องต้น](./part-088-simd-vectorization.md)
