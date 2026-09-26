# Part 90: Benchmarking ด้วย Google Benchmark (Step 713–720)

> Module G — Concurrency และ Performance Engineering | Part 90 จาก 125
> Step ที่ครอบคลุมใน Part นี้: Step 713–720
> Part ก่อนหน้า: [Part 89 — Compiler Optimization และ Assembly](./part-089-compiler-optimizations.md) | Part ถัดไป: Part 91 — CMake ตั้งแต่พื้นฐานถึงขั้นสูง (Module H)

## เป้าหมายการเรียนรู้ (Learning Objectives)

หลังจบ Part นี้ ผู้เรียนจะสามารถ:

1. อธิบายได้ว่าทำไมการจับเวลาโค้ดด้วยมือ (แบบที่ทำใน Part 86) ไม่น่าเชื่อถือพอสำหรับงาน
   Performance Engineering จริงจัง และ Google Benchmark แก้ปัญหาเหล่านั้นอย่างไร
2. ติดตั้งและ link ไลบรารี Google Benchmark เข้ากับโปรเจกต์ C++ ได้ด้วยตัวเอง
3. เขียนโปรแกรม benchmark แรกด้วยมาโคร `BENCHMARK` และ `BENCHMARK_MAIN` ได้อย่างถูกต้อง
4. ใช้ `benchmark::DoNotOptimize` และ `benchmark::ClobberMemory` เพื่อป้องกันไม่ให้ compiler
   optimize โค้ดที่กำลังวัดผลทิ้งไปโดยไม่รู้ตัว
5. อ่านและตีความผลลัพธ์ของ Google Benchmark ได้ครบทุกคอลัมน์ (Time, CPU, Iterations) รวมถึง
   เข้าใจความแตกต่างระหว่าง Wall-clock Time กับ CPU Time
6. เปรียบเทียบประสิทธิภาพของ 2 วิธี implementation ที่ทำงานอย่างเดียวกันด้วยข้อมูลทางสถิติจริง
   (ไม่ใช่การเดาหรือจับเวลาครั้งเดียว)
7. เขียน Benchmark Fixture สำหรับกรณีที่ต้อง setup ข้อมูลซับซ้อนก่อนวัดผล โดยไม่ให้เวลา setup
   ปนเข้าไปในผลการวัด

---

## 90.1 ทำไมการจับเวลาด้วยมือไม่พอ (Step 713)

ใน **Part 86** เราเรียนรู้การ profile โปรแกรมด้วย `perf`/`gprof` และเคยจับเวลาโค้ดด้วย
`std::chrono` แบบจับเวลาครั้งเดียว (start → run → stop) วิธีนี้ใช้งานได้ในหลายกรณี แต่มีจุดอ่อน
สำคัญหลายข้อที่ทำให้ไม่เหมาะกับการเปรียบเทียบประสิทธิภาพอย่างจริงจัง ลองพิสูจน์ด้วยการทดลองจริง:

```cpp
#include <cstdio>
#include <chrono>
#include <vector>

int main() {
    const int N = 1000000;
    auto t0 = std::chrono::steady_clock::now();
    std::vector<int> v;
    for (int i = 0; i < N; i++) v.push_back(i);
    auto t1 = std::chrono::steady_clock::now();
    double ms = std::chrono::duration<double, std::milli>(t1 - t0).count();
    printf("manual timing: %.4f ms (size=%zu)\n", ms, v.size());
    return 0;
}
```

รันซ้ำ 5 ครั้งติดกันบนเครื่องเดียวกัน โปรแกรมเดียวกัน ไม่มีอะไรเปลี่ยนแปลงเลย:

```
manual timing: 6.1590 ms (size=1000000)
manual timing: 5.8772 ms (size=1000000)
manual timing: 4.8748 ms (size=1000000)
manual timing: 5.3578 ms (size=1000000)
manual timing: 4.4271 ms (size=1000000)
```

ผลลัพธ์แกว่งไปมาระหว่าง **4.43 ms ถึง 6.16 ms** (ต่างกันเกือบ 40%!) ทั้งที่รันโปรแกรมเดียวกัน
ทุกประการ — นี่คือสิ่งที่เรียกว่า **Noise/Jitter** ซึ่งเกิดจากหลายสาเหตุร่วมกัน: OS scheduler สลับ
งานอื่นเข้ามาแทรก, CPU frequency scaling (Turbo Boost ปรับความเร็วขึ้นลง), สถานะของ Cache
ที่ต่างกันในแต่ละรอบ, และ CPU ยังไม่ "warm up" เต็มที่ในการรันครั้งแรกๆ

ปัญหาของการจับเวลาด้วยมือแบบนี้มี 3 ข้อหลัก:

1. **ไม่มี Warm-up**: การรันครั้งแรกมักช้ากว่าปกติ (instruction cache ยังไม่โหลด, CPU ยังไม่เร่ง
   ความเร็วเต็มที่) ทำให้ผลลัพธ์ตัวแรกมักเบี่ยงเบนจากค่าจริง
2. **จำนวนรอบไม่พอทางสถิติ**: รันแค่ครั้งเดียวไม่สามารถบอกได้เลยว่าผลลัพธ์ที่ได้เป็น "ค่าปกติ"
   หรือเป็นค่าผิดปกติ (outlier) จากการรบกวนของระบบในจังหวะนั้นพอดี
3. **เสี่ยงถูก compiler optimize โค้ดทิ้ง**: ถ้าผลลัพธ์ที่คำนวณได้ไม่ถูกใช้งานต่อ (เหมือนตัวอย่าง
   Dead Code Elimination ที่เราเห็นใน Part 89) compiler อาจตัดโค้ดที่เรากำลังพยายามวัดทิ้งไปเลย
   ทำให้ตัวเลขที่วัดได้ "เร็วเกินจริง" อย่างน่าสงสัย

**Google Benchmark** คือไลบรารีมาตรฐานอุตสาหกรรม (พัฒนาโดยทีม Google เอง ใช้ในโปรเจกต์ระดับ
โลกอย่าง Abseil, LLVM, TensorFlow) ที่ถูกออกแบบมาแก้ปัญหาทั้ง 3 ข้อนี้โดยเฉพาะ:

- รัน**หลายรอบอัตโนมัติ**จนกว่าผลลัพธ์จะเสถียรทางสถิติ (ปรับจำนวน iteration เองโดยอัตโนมัติ)
- คำนวณทั้ง **Wall-clock Time** และ **CPU Time** แยกกัน
- มีฟังก์ชัน `DoNotOptimize`/`ClobberMemory` ที่บอก compiler อย่างชัดเจนว่า "ห้าม optimize
  ส่วนนี้ทิ้ง เพราะเรากำลังวัดผลอยู่"
- รองรับการรัน benchmark ซ้ำหลายรอบ (repetitions) เพื่อดูค่า mean/median/stddev

---

## 90.2 ติดตั้งและ Link Google Benchmark (Step 714)

บน Ubuntu/Debian ติดตั้งผ่าน `apt` ได้ตรงๆ:

```bash
sudo apt update
sudo apt install libbenchmark-dev -y
```

ตรวจสอบว่าติดตั้งสำเร็จ:

```bash
$ dpkg -l | grep benchmark
ii  libbenchmark-dev:amd64  1.8.3-3  amd64  Microbenchmark support library, development files
ii  libbenchmark1.8.3:amd64 1.8.3-3  amd64  Microbenchmark support library, shared library
```

ไลบรารีนี้ประกอบด้วย 2 ส่วนหลัก:

| ไฟล์ | หน้าที่ |
|---|---|
| `/usr/include/benchmark/benchmark.h` | Header หลักที่เรา `#include` |
| `/usr/lib/x86_64-linux-gnu/libbenchmark.so` | Shared library หลัก ต้อง link ด้วย `-lbenchmark` |
| `/usr/lib/x86_64-linux-gnu/libbenchmark_main.a` | (ทางเลือก) มี `main()` ให้พร้อม ไม่ต้องเขียนเอง |

เนื่องจาก Google Benchmark ใช้ thread ภายในสำหรับบางฟีเจอร์ (เช่น multi-threaded benchmark)
เราต้อง link `-lpthread` ควบคู่ไปด้วยเสมอ คำสั่งคอมไพล์มาตรฐานจึงเป็น:

```bash
g++ -O2 -Wall -Wextra -Wpedantic -std=c++17 my_benchmark.cpp -o my_benchmark -lbenchmark -lpthread
```

> **ข้อสังเกตสำคัญ**: ต้อง compile ด้วย optimization เปิดอยู่เสมอ (`-O2` ขึ้นไป) เวลาเขียน
> benchmark จริง เพราะเราต้องการวัดประสิทธิภาพของโค้ดแบบเดียวกับที่จะถูกใช้งานจริงใน production
> ถ้าวัดที่ `-O0` ตัวเลขที่ได้จะไม่สะท้อนประสิทธิภาพจริงเลย (เหมือนที่เห็นความต่างมหาศาลระหว่าง
> `-O0` กับ `-O2` ใน Part 89)

ทดสอบด้วยโปรแกรมเปล่าๆ ที่ทำแค่บวกเลข เพื่อยืนยันว่าทุกอย่างพร้อมใช้งาน:

```cpp
#include <benchmark/benchmark.h>

static void BM_test(benchmark::State& state) {
    for (auto _ : state) {
        int x = 1 + 1;
        benchmark::DoNotOptimize(x);
    }
}
BENCHMARK(BM_test);
BENCHMARK_MAIN();
```

```bash
g++ -O2 -Wall -Wextra -Wpedantic -std=c++17 t.cpp -o t -lbenchmark -lpthread
./t
```

ผลลัพธ์จริงที่รันบนเครื่องนี้:

```
2026-09-26T08:11:25+00:00
Running ./t
Run on (4 X 2100 MHz CPU s)
CPU Caches:
  L1 Data 48 KiB (x4)
  L1 Instruction 32 KiB (x4)
  L2 Unified 2048 KiB (x4)
  L3 Unified 266240 KiB (x1)
Load Average: 0.30, 0.24, 0.18
***WARNING*** Library was built as DEBUG. Timings may be affected.
-----------------------------------------------------
Benchmark           Time             CPU   Iterations
-----------------------------------------------------
BM_test         0.179 ns        0.178 ns   1000000000
```

สังเกตบรรทัด `***WARNING*** Library was built as DEBUG. Timings may be affected.` — แพ็กเกจ
`libbenchmark-dev` ที่ apt แจกมาถูก build โดยไม่เปิด `NDEBUG` (มี assertion ภายในไลบรารีเอง)
ซึ่งทำให้ตัวเลขที่ได้ **ช้ากว่าความเป็นจริงเล็กน้อย** แต่ยังคง**เชื่อถือได้สำหรับการเปรียบเทียบ
สัมพัทธ์** (relative comparison) ระหว่าง 2 วิธี implementation ซึ่งเป็นสิ่งที่เราสนใจเป็นหลักใน
บทเรียนนี้ ถ้าต้องการตัวเลข absolute ที่แม่นยำที่สุดสำหรับรายงานอย่างเป็นทางการ ควร build
Google Benchmark เองจาก source ด้วย `-DCMAKE_BUILD_TYPE=Release` (จะเรียนเรื่อง CMake ใน
Part 91)

Benchmark ยังรายงานข้อมูลฮาร์ดแวร์ของเครื่องให้อัตโนมัติทุกครั้ง (จำนวน core, ขนาด Cache
แต่ละระดับ) ซึ่งสำคัญมาก เพราะตัวเลข benchmark ที่ไม่มีบริบทของฮาร์ดแวร์ติดไปด้วยจะเปรียบเทียบ
ข้ามเครื่องไม่ได้เลย

---

## 90.3 เขียน Benchmark แรกด้วย BENCHMARK_MAIN (Step 715)

โครงสร้างพื้นฐานของโปรแกรม Google Benchmark มี 3 ส่วน:

1. **ฟังก์ชัน benchmark**: รับ `benchmark::State&` เป็นพารามิเตอร์ ข้างในมี loop `for (auto _ :
   state)` ที่ตัวไลบรารีเป็นคนควบคุมจำนวนรอบเอง (ไม่ใช่เราเขียนจำนวนรอบตายตัว)
2. **มาโคร `BENCHMARK(ชื่อฟังก์ชัน)`**: ลงทะเบียนฟังก์ชันนั้นเข้าสู่ระบบของ Google Benchmark
3. **มาโคร `BENCHMARK_MAIN()`**: สร้างฟังก์ชัน `main()` ให้อัตโนมัติ (parse command-line flags,
   รัน benchmark ทั้งหมดที่ลงทะเบียนไว้, พิมพ์ผลลัพธ์)

```cpp
#include <benchmark/benchmark.h>
#include <string>

static void BM_StringCreation(benchmark::State& state) {
    for (auto _ : state) {
        std::string empty_string;
        benchmark::DoNotOptimize(empty_string);
    }
}
BENCHMARK(BM_StringCreation);

static void BM_StringCopy(benchmark::State& state) {
    std::string x = "hello world from google benchmark";
    for (auto _ : state) {
        std::string copy(x);
        benchmark::DoNotOptimize(copy);
    }
}
BENCHMARK(BM_StringCopy);

BENCHMARK_MAIN();
```

```bash
g++ -O2 -Wall -Wextra -Wpedantic -std=c++17 bm_first.cpp -o bm_first -lbenchmark -lpthread
./bm_first
```

ผลลัพธ์จริง:

```
------------------------------------------------------------
Benchmark                  Time             CPU   Iterations
------------------------------------------------------------
BM_StringCreation      0.598 ns        0.560 ns   1000000000
BM_StringCopy           20.3 ns         20.2 ns     35270717
```

สังเกตจุดสำคัญ: เราไม่ได้เขียนโค้ดกำหนดเองว่า "รัน 1,000,000,000 รอบ" หรือ "รัน 35,270,717
รอบ" เลย — **Google Benchmark เป็นคนตัดสินใจจำนวนรอบเองโดยอัตโนมัติ** โดยจะรันซ้ำไปเรื่อยๆ
จนกว่าเวลารวมของ benchmark นั้นจะนานพอที่จะให้ผลลัพธ์เสถียรทางสถิติ (ค่า default คือรันจนเวลา
รวมอย่างน้อยประมาณ 0.5 วินาที) งานที่เร็วมาก (`BM_StringCreation` ใช้เวลาแค่ ~0.6 นาโนวินาที
ต่อรอบ) จึงต้องรันซ้ำมากถึงพันล้านรอบเพื่อให้ผลรวมมีนัยสำคัญทางสถิติ ในขณะที่งานที่ช้ากว่า
(`BM_StringCopy` ~20 นาโนวินาทีต่อรอบ) ใช้จำนวนรอบน้อยกว่ามาก

**ทางเลือกที่ไม่ต้องเขียน `BENCHMARK_MAIN()` เอง**: ถ้าต้องการ link กับ `libbenchmark_main.a`
แทนการใช้มาโคร (เช่น กรณีที่ต้องการปรับแต่ง `main()` เพิ่มเติมภายหลัง) สามารถละ
`BENCHMARK_MAIN()` ออกแล้ว link เพิ่มด้วย `-lbenchmark_main`:

```bash
g++ -O2 -Wall -Wextra -Wpedantic -std=c++17 bm_first_nomain.cpp -o bm_first_nomain \
    -lbenchmark_main -lbenchmark -lpthread
```

ตลอด Part นี้เราจะใช้ `BENCHMARK_MAIN()` เป็นหลัก เพราะเห็นภาพรวมของโปรแกรมชัดเจนกว่าสำหรับ
การเรียนรู้

---

## 90.4 อ่านผลลัพธ์: คอลัมน์ Time, CPU, Iterations (Step 716)

ตารางผลลัพธ์ของ Google Benchmark มี 3 คอลัมน์หลักเสมอ:

| คอลัมน์ | ความหมาย |
|---|---|
| **Time** | Wall-clock Time เฉลี่ยต่อ 1 รอบ (เวลาจริงที่ผ่านไปตามนาฬิกา รวมเวลาที่ thread ถูกพัก/รอด้วย) |
| **CPU** | CPU Time เฉลี่ยต่อ 1 รอบ (เวลาที่ CPU ใช้ประมวลผลจริงๆ ไม่รวมเวลาที่ thread ถูกพัก) |
| **Iterations** | จำนวนรอบทั้งหมดที่ไลบรารีตัดสินใจรัน เพื่อให้ได้ผลลัพธ์ที่เสถียรทางสถิติ |

ในกรณีส่วนใหญ่ (โค้ดที่ทำงานคำนวณล้วนๆ ไม่มีการ sleep/รอ I/O) ค่า Time กับ CPU จะใกล้เคียงกัน
มาก แต่ทั้งสองค่ามีความหมายต่างกันจริงๆ ลองดูตัวอย่างที่ทำให้เห็นความต่างชัดเจน:

```cpp
#include <benchmark/benchmark.h>
#include <thread>
#include <chrono>

// จำลองงานที่ต้องรอ I/O -- ใช้เวลาจริง (wall-clock) นาน แต่ไม่ได้ใช้ CPU ทำงานหนักเลย
static void BM_SleepingTask(benchmark::State& state) {
    for (auto _ : state) {
        std::this_thread::sleep_for(std::chrono::microseconds(200));
    }
}
BENCHMARK(BM_SleepingTask)->Iterations(200);

BENCHMARK_MAIN();
```

```bash
g++ -O2 -Wall -Wextra -Wpedantic -std=c++17 -pthread bm_timecpu.cpp -o bm_timecpu \
    -lbenchmark -lpthread
./bm_timecpu
```

ผลลัพธ์จริง:

```
-------------------------------------------------------------------------
Benchmark                               Time             CPU   Iterations
-------------------------------------------------------------------------
BM_SleepingTask/iterations:200     287856 ns        17932 ns          200
```

**Time (Wall-clock) = 287,856 ns (~288 μs)** ใกล้เคียงกับเวลาที่เรา sleep จริง (200 μs บวก
overhead ของการเรียก syscall) ในขณะที่ **CPU Time = 17,932 ns (~18 μs)** ต่ำกว่ามาก เพราะระหว่าง
ที่ thread กำลัง sleep อยู่นั้น มันไม่ได้ใช้ CPU ทำงานเลย (OS เอา CPU ไปให้ thread อื่นทำงานแทน)
ตัวเลขนี้พิสูจน์ให้เห็นชัดว่า **Time วัด "เวลาที่ผู้ใช้รอ" ส่วน CPU วัด "งานที่ CPU ทำจริง"** — สำหรับ
โค้ดที่ทำงานแบบ multi-thread ขนานกันจริงๆ บางครั้ง CPU Time อาจ**สูงกว่า** Time เสียอีก (เพราะ
มีหลาย thread ทำงานพร้อมกันในช่วงเวลา wall-clock เดียวกัน แต่ CPU Time สะสมจากทุก thread รวมกัน)

สำหรับ benchmark ทั่วไปที่ไม่มีการ sleep/บล็อกด้วย I/O (ซึ่งเป็นกรณีส่วนใหญ่ในบทเรียนนี้) เรามัก
ดูที่คอลัมน์ **CPU** เป็นหลัก เพราะสะท้อนงานคำนวณจริงของ CPU โดยไม่ถูกรบกวนจากปัจจัยภายนอก
เช่น โปรแกรมอื่นแย่ง CPU ไปชั่วคราวระหว่างการวัดผล

---

## 90.5 เปรียบเทียบ 2 Implementation จริง: push_back แบบมี/ไม่มี reserve (Step 717)

นี่คือการใช้งานที่สำคัญที่สุดของ Google Benchmark: การเปรียบเทียบ 2 วิธี implementation ที่ทำงาน
อย่างเดียวกันแต่เขียนต่างกัน ใน **Part 59** เราเรียนเรื่อง `std::vector` และเคยพูดถึงว่าการเรียก
`reserve()` ล่วงหน้าช่วยลดจำนวนครั้งของการ reallocate หน่วยความจำ วันนี้เราจะพิสูจน์คำกล่าวอ้างนี้
ด้วยตัวเลขจริง ไม่ใช่แค่ทฤษฎี

```cpp
#include <benchmark/benchmark.h>
#include <vector>
#include <cstddef>

// ไม่ได้เรียก reserve() ล่วงหน้า -- vector ต้องขยายขนาด (reallocate + copy ข้อมูลเก่า)
// หลายครั้งตามที่ growth factor ของ std::vector กำหนด (มักจะเพิ่มเป็นราว 2 เท่าทุกครั้งที่เต็ม)
static void BM_PushBackNoReserve(benchmark::State& state) {
    const std::size_t n = static_cast<std::size_t>(state.range(0));
    for (auto _ : state) {
        std::vector<int> v;
        for (std::size_t i = 0; i < n; i++) {
            v.push_back(static_cast<int>(i));
        }
        benchmark::DoNotOptimize(v);
    }
}
BENCHMARK(BM_PushBackNoReserve)->Arg(1000)->Arg(10000)->Arg(100000)->Arg(1000000);

// เรียก reserve(n) ล่วงหน้า -- จองหน่วยความจำครั้งเดียวพอ ไม่มี reallocate ระหว่างทางเลย
static void BM_PushBackWithReserve(benchmark::State& state) {
    const std::size_t n = static_cast<std::size_t>(state.range(0));
    for (auto _ : state) {
        std::vector<int> v;
        v.reserve(n);
        for (std::size_t i = 0; i < n; i++) {
            v.push_back(static_cast<int>(i));
        }
        benchmark::DoNotOptimize(v);
    }
}
BENCHMARK(BM_PushBackWithReserve)->Arg(1000)->Arg(10000)->Arg(100000)->Arg(1000000);

BENCHMARK_MAIN();
```

จุดสำคัญของโค้ดนี้ที่ควรสังเกต:

- `state.range(0)` คือค่าพารามิเตอร์ที่มาจาก `->Arg(...)` ที่เรากำหนดไว้ตอนลงทะเบียน ทำให้
  benchmark เดียวกันถูกรันซ้ำด้วยขนาดข้อมูลหลายขนาดโดยอัตโนมัติ ไม่ต้องเขียนฟังก์ชันซ้ำหลายตัว
- `benchmark::DoNotOptimize(v)` ท้าย loop คือหัวใจสำคัญ — บอก compiler ว่า `v` "ถูกใช้งานจริง"
  ห้ามมองว่าเป็น dead code แล้วลบลูปทั้งหมดทิ้ง (รายละเอียดเต็มอยู่ในหัวข้อ 90.6)

```bash
g++ -O2 -Wall -Wextra -Wpedantic -std=c++17 bm_reserve.cpp -o bm_reserve -lbenchmark -lpthread
./bm_reserve
```

ผลลัพธ์จริงที่วัดได้บนเครื่องนี้:

```
-------------------------------------------------------------------------
Benchmark                               Time             CPU   Iterations
-------------------------------------------------------------------------
BM_PushBackNoReserve/1000            1198 ns         1192 ns       609330
BM_PushBackNoReserve/10000          11920 ns        11864 ns        56084
BM_PushBackNoReserve/100000        423753 ns       422796 ns         1716
BM_PushBackNoReserve/1000000      4962568 ns      4958863 ns          132
BM_PushBackWithReserve/1000           757 ns          756 ns       888662
BM_PushBackWithReserve/10000         8561 ns         8522 ns        87124
BM_PushBackWithReserve/100000      112285 ns       112055 ns         5238
BM_PushBackWithReserve/1000000    1988119 ns      1983146 ns          365
```

สร้างตารางเปรียบเทียบอัตราเร่ง (speedup) จากตัวเลขจริงข้างบน:

| ขนาดข้อมูล (n) | ไม่มี reserve (ns) | มี reserve (ns) | เร็วขึ้นกี่เท่า |
|---|---|---|---|
| 1,000 | 1,192 | 756 | 1.58× |
| 10,000 | 11,864 | 8,522 | 1.39× |
| 100,000 | 422,796 | 112,055 | **3.77×** |
| 1,000,000 | 4,958,863 | 1,983,146 | **2.50×** |

ตัวเลขจริงยืนยันสิ่งที่ทฤษฎีบอกไว้ใน Part 59 อย่างชัดเจน: การเรียก `reserve()` ล่วงหน้าเมื่อรู้
ขนาดข้อมูลคร่าวๆ ช่วยลดเวลาได้จริงถึง 2.5-3.8 เท่าในกรณีขนาดข้อมูลใหญ่ ยิ่งข้อมูลเยอะขึ้นเท่าไหร่
ผลต่างก็ยิ่งเห็นชัดเจนขึ้น (เพราะจำนวนครั้งของการ reallocate + copy ข้อมูลเก่าที่หลีกเลี่ยงได้
เพิ่มขึ้นตามขนาด) นี่คือพลังของการวัดผลด้วยเครื่องมือที่เหมาะสม แทนที่จะเดาหรือเชื่อทฤษฎีเฉยๆ

---

## 90.6 DoNotOptimize และ ClobberMemory โดยละเอียด (Step 718)

### ทำไมต้องมี DoNotOptimize

เราพูดถึง Dead Code Elimination ไปแล้วใน **Part 89** — ถ้า compiler เห็นว่าผลลัพธ์ของการคำนวณ
ไม่ถูกใช้งานที่ไหนเลย มันมีสิทธิ์เต็มที่ (และมักจะทำจริง) ลบโค้ดที่คำนวณนั้นทิ้งทั้งหมด ปัญหาคือ
ใน microbenchmark เรามักเขียนโค้ดแบบ "คำนวณแล้วไม่ได้ใช้ผลต่อ" อยู่บ่อยๆ เพราะเราสนใจแค่ "เวลา
ที่ใช้คำนวณ" ไม่ได้สนใจผลลัพธ์จริงๆ — นี่คือกับดักที่อันตรายมาก ลองพิสูจน์ด้วยตัวอย่างจริง:

```cpp
#include <benchmark/benchmark.h>

// ผิดพลาด: ไม่ได้ใช้ DoNotOptimize เลย -- compiler มองว่าผลลัพธ์ไม่ถูกใช้ที่ไหนต่อ
// จึง optimize ลูปทั้งหมดทิ้งไปเลย (dead code elimination) กลายเป็น benchmark เปล่าๆ
static void BM_SumBad(benchmark::State& state) {
    for (auto _ : state) {
        long sum = 0;
        for (int i = 0; i < 1000; i++) {
            sum += i;
        }
        // ไม่มี DoNotOptimize(sum) ตรงนี้ -- sum ไม่ถูกใช้ต่อเลย
    }
}
BENCHMARK(BM_SumBad);

// ถูกต้อง: บอก compiler ว่า sum "ถูกใช้งานจริง" ห้าม optimize ทิ้ง
static void BM_SumGood(benchmark::State& state) {
    for (auto _ : state) {
        long sum = 0;
        for (int i = 0; i < 1000; i++) {
            sum += i;
        }
        benchmark::DoNotOptimize(sum);
    }
}
BENCHMARK(BM_SumGood);

BENCHMARK_MAIN();
```

```bash
g++ -O2 -Wall -Wextra -Wpedantic -std=c++17 bm_pitfall.cpp -o bm_pitfall -lbenchmark -lpthread
./bm_pitfall
```

ผลลัพธ์จริงที่น่าตกใจ:

```
-----------------------------------------------------
Benchmark           Time             CPU   Iterations
-----------------------------------------------------
BM_SumBad       0.000 ns        0.000 ns   1000000000
BM_SumGood      0.356 ns        0.354 ns   1000000000
```

**`BM_SumBad` วัดได้ 0.000 นาโนวินาที** — ไม่ใช่เพราะการบวกเลข 1,000 รอบนั้นเร็วขนาดนั้นจริง
แต่เพราะ compiler เห็นว่าตัวแปร `sum` ที่คำนวณในลูปไม่เคยถูกใช้งานที่ไหนต่อเลยหลังจากลูปจบ
(ไม่มีการ `return`, ไม่มีการพิมพ์, ไม่มีการส่งต่อไปที่ไหน) จึง**ลบลูปคำนวณทั้งหมดทิ้งไปเกลี้ยง**
ตาม Dead Code Elimination เหมือนที่เห็นในตัวอย่าง Part 89 — สิ่งที่เราวัดได้จริงๆ คือ "เวลาของ
ลูปเปล่าๆ ที่ไม่ทำอะไรเลย" ไม่ใช่เวลาของการบวกเลขตามที่ตั้งใจ

`benchmark::DoNotOptimize(sum)` แก้ปัญหานี้โดยบอก compiler ผ่านกลไก inline assembly พิเศษ
ว่า "ค่านี้อาจถูกอ่าน/เขียนจากภายนอกที่ compiler มองไม่เห็น" ทำให้ compiler ไม่กล้าตัดโค้ดที่
คำนวณค่านั้นทิ้ง (เพราะไม่รู้ว่า "ภายนอก" นั้นต้องการใช้ค่าที่ถูกต้องหรือไม่) ผลคือ `BM_SumGood`
วัดได้ 0.356 นาโนวินาที ซึ่งสมเหตุสมผลกว่ามากสำหรับการบวกเลข 1,000 ครั้ง

> **กฎทองของการเขียน Benchmark**: ทุกค่าที่คำนวณในลูป benchmark ที่ไม่ได้ถูก `return` หรือ
> ใช้งานต่ออย่างชัดเจน **ต้องผ่าน `benchmark::DoNotOptimize()` เสมอ** ไม่มีข้อยกเว้น มิเช่นนั้น
> ตัวเลขที่วัดได้จะไม่มีความหมายอะไรเลย — และที่อันตรายที่สุดคือตัวเลข "0.000 ns" หรือ "เร็วผิดปกติ"
> แบบนี้ **ไม่ error ไม่เตือนอะไรทั้งสิ้น** ต้องอาศัยสามัญสำนึกของผู้เขียนในการสังเกตความผิดปกติเอง

### ClobberMemory: เมื่อ DoNotOptimize ยังไม่พอ

`DoNotOptimize` ป้องกันไม่ให้ค่าถูกลบทิ้ง แต่ไม่ได้รับประกันว่าค่านั้นจะถูก**เขียนลงหน่วยความจำ
จริง** (compiler อาจยังคงค่าไว้ใน register ต่อไปได้) ในบางกรณีที่เราต้องการวัดผลกระทบของ
การเขียนหน่วยความจำจริงๆ (เช่น วัด cache behavior) เราต้องใช้ `benchmark::ClobberMemory()`
เพิ่มเติม เพื่อบอก compiler ว่า "หน่วยความจำทั้งหมดอาจถูกเปลี่ยนแปลงไปแล้ว ห้ามสมมติว่าค่าที่
เคยอ่านไว้ใน register ยังคงถูกต้อง ต้องเขียนค่าที่ค้างอยู่ลง memory จริงก่อน":

```cpp
#include <benchmark/benchmark.h>
#include <vector>
#include <cstddef>

static void BM_VectorWrite(benchmark::State& state) {
    std::vector<int> v(1000);
    for (auto _ : state) {
        for (std::size_t i = 0; i < v.size(); i++) {
            v[i] = static_cast<int>(i);
        }
        benchmark::ClobberMemory(); // บังคับให้ค่าที่เขียนถูก flush ลง memory จริง ไม่ค้างใน register
    }
    benchmark::DoNotOptimize(v.data());
}
BENCHMARK(BM_VectorWrite);

BENCHMARK_MAIN();
```

```bash
g++ -O2 -Wall -Wextra -Wpedantic -std=c++17 bm_clobber.cpp -o bm_clobber -lbenchmark -lpthread
./bm_clobber
```

```
---------------------------------------------------------
Benchmark               Time             CPU   Iterations
---------------------------------------------------------
BM_VectorWrite        144 ns          143 ns      4874013
```

โดยสรุป: ใช้ `DoNotOptimize` กับ**ค่าที่อ่านผลลัพธ์** (ป้องกันการลบโค้ดคำนวณทิ้ง) และใช้
`ClobberMemory` เมื่อต้องการยืนยันว่า**การเขียนหน่วยความจำ**เกิดขึ้นจริงในทุกรอบ (สำคัญกับ
benchmark ที่วัดเรื่อง memory write/cache behavior โดยเฉพาะ) ในกรณีทั่วไปที่ไม่ได้เจาะจงวัด
memory behavior การใช้ `DoNotOptimize` เพียงอย่างเดียวก็เพียงพอแล้ว

---

## 90.7 Benchmark Fixture สำหรับ Setup ที่ซับซ้อน (Step 719)

บางครั้งก่อนวัดผลจริง เราต้องเตรียมข้อมูลที่ใช้เวลานาน (เช่น สร้างข้อมูลสุ่มขนาดใหญ่, เชื่อมต่อ
ฐานข้อมูล, โหลดไฟล์) ถ้าเขียนการเตรียมข้อมูลนี้ไว้**ข้างใน** loop `for (auto _ : state)` เวลา
เตรียมข้อมูลจะถูกนับรวมเข้าไปในผลวัดด้วย ทำให้ตัวเลขที่ได้ผิดเพี้ยนไปจากสิ่งที่ต้องการวัดจริงๆ

**Fixture** คือ class ที่สืบทอดจาก `benchmark::Fixture` มีเมธอด `SetUp()`/`TearDown()` ที่ถูก
เรียก**ก่อน/หลัง**การวัดผลของแต่ละรอบเท่านั้น (ไม่ถูกนับเวลา):

```cpp
#include <benchmark/benchmark.h>
#include <algorithm>
#include <random>
#include <vector>
#include <cstddef>

// Fixture: ใช้เมื่อ benchmark ต้องมีการ setup ที่ซับซ้อน/ใช้เวลานาน
// และไม่อยากให้เวลา setup นั้นถูกนับรวมเข้าไปในผลวัด
class SortFixture : public benchmark::Fixture {
public:
    std::vector<int> data;

    // SetUp ถูกเรียก "ก่อน" ทุกรอบการวัดเวลา (ไม่ถูกนับเวลา)
    void SetUp(const benchmark::State& state) override {
        const std::size_t n = static_cast<std::size_t>(state.range(0));
        data.resize(n);
        std::mt19937 rng(42); // seed คงที่ ทำให้ผลทดลองซ้ำได้ (reproducible)
        std::uniform_int_distribution<int> dist(0, 1000000);
        for (std::size_t i = 0; i < n; i++) {
            data[i] = dist(rng);
        }
    }

    // TearDown ถูกเรียก "หลัง" ทุกรอบ (ไม่ถูกนับเวลาเช่นกัน)
    void TearDown(const benchmark::State&) override {
        data.clear();
        data.shrink_to_fit();
    }
};

BENCHMARK_DEFINE_F(SortFixture, StdSort)(benchmark::State& state) {
    for (auto _ : state) {
        state.PauseTiming();                 // หยุดจับเวลาชั่วคราว
        std::vector<int> copy = data;        // ทำสำเนาข้อมูล (ไม่อยากนับเวลา copy)
        state.ResumeTiming();                // เริ่มจับเวลาต่อ
        std::sort(copy.begin(), copy.end());
        benchmark::DoNotOptimize(copy);
    }
}
BENCHMARK_REGISTER_F(SortFixture, StdSort)->Arg(1000)->Arg(100000);

BENCHMARK_MAIN();
```

```bash
g++ -O2 -Wall -Wextra -Wpedantic -std=c++17 bm_fixture.cpp -o bm_fixture -lbenchmark -lpthread
./bm_fixture
```

ผลลัพธ์จริง:

```
---------------------------------------------------------------------
Benchmark                           Time             CPU   Iterations
---------------------------------------------------------------------
SortFixture/StdSort/1000         7935 ns         7933 ns        85442
SortFixture/StdSort/100000    6880901 ns      6869157 ns          100
```

จุดสำคัญของโครงสร้าง Fixture:

- `BENCHMARK_DEFINE_F(ชื่อ Fixture, ชื่อ Test)` แทนที่ `static void` ธรรมดา ทำให้ฟังก์ชัน
  benchmark เข้าถึงสมาชิกของ class (`data`) ได้โดยตรง
- `BENCHMARK_REGISTER_F(...)` ใช้แทน `BENCHMARK(...)` สำหรับลงทะเบียน Fixture
- `state.PauseTiming()`/`state.ResumeTiming()` ใช้เมื่อมีบางส่วน**ภายใน**ลูปวัดผลที่ไม่ต้องการนับ
  เวลา (ในตัวอย่างนี้คือการ copy ข้อมูลต้นฉบับก่อน sort เพราะ `std::sort` เรียงข้อมูลแบบ in-place
  ถ้าไม่ copy ใหม่ทุกรอบ ข้อมูลจะถูกเรียงไปแล้วตั้งแต่รอบแรก ทำให้รอบถัดไปวัด sort ข้อมูลที่เรียง
  อยู่แล้ว ซึ่งเร็วผิดปกติและไม่ตรงกับสถานการณ์จริง)

> **ข้อควรระวัง**: `PauseTiming()`/`ResumeTiming()` มี overhead ของตัวมันเองพอสมควร (ต้อง
> เรียก syscall จับเวลาเพิ่มขึ้น) ถ้าเป็นไปได้ควรออกแบบ benchmark ให้ไม่ต้อง pause/resume บ่อยๆ
> ในโค้ด production-grade benchmark จริง มักใช้เทคนิคเตรียมข้อมูลสำเนาไว้ล่วงหน้าหลายชุดใน
> `SetUp()` แทน เพื่อหลีกเลี่ยง overhead ของการ pause/resume ในลูปที่รันบ่อยมาก

---

## 90.8 ควบคุมการรันด้วย Command-Line Flags และอ่านค่าสถิติ (Step 720)

Google Benchmark มี command-line flags ให้ควบคุมพฤติกรรมการรันได้ละเอียดโดยไม่ต้องแก้โค้ด
หรือคอมไพล์ใหม่เลย ที่ใช้บ่อยที่สุด:

| Flag | หน้าที่ |
|---|---|
| `--benchmark_filter=<regex>` | รันเฉพาะ benchmark ที่ชื่อตรงกับ regex ที่กำหนด |
| `--benchmark_repetitions=<n>` | รันทั้งชุดซ้ำ n รอบ แล้วรายงานค่าสถิติ (mean/median/stddev/cv) |
| `--benchmark_min_time=<n>s` | กำหนดเวลาขั้นต่ำที่ต้องรันต่อ 1 benchmark (ค่า default ประมาณ 0.5s) |
| `--benchmark_format=<console\|json\|csv>` | เลือกรูปแบบผลลัพธ์ (มีประโยชน์มากตอนเก็บผลไปวิเคราะห์ต่อ) |

ลองรัน `bm_reserve` (จากหัวข้อ 90.5) เฉพาะกรณี `WithReserve/1000000` ซ้ำ 5 รอบ เพื่อดูค่า
สถิติความเสถียร:

```bash
./bm_reserve --benchmark_filter=WithReserve/1000000 --benchmark_repetitions=5
```

ผลลัพธ์จริง:

```
--------------------------------------------------------------------------------
Benchmark                                      Time             CPU   Iterations
--------------------------------------------------------------------------------
BM_PushBackWithReserve/1000000           1886993 ns      1881860 ns          350
BM_PushBackWithReserve/1000000           1927294 ns      1920413 ns          350
BM_PushBackWithReserve/1000000           2129331 ns      2123373 ns          350
BM_PushBackWithReserve/1000000           2176323 ns      2174742 ns          350
BM_PushBackWithReserve/1000000           2148392 ns      2138796 ns          350
BM_PushBackWithReserve/1000000_mean      2053666 ns      2047837 ns            5
BM_PushBackWithReserve/1000000_median    2129331 ns      2123373 ns            5
BM_PushBackWithReserve/1000000_stddev     135548 ns       135895 ns            5
BM_PushBackWithReserve/1000000_cv           6.60 %          6.64 %             5
```

ข้อมูลนี้บอกอะไรเราบ้าง:

- **`_mean`**: ค่าเฉลี่ยของทั้ง 5 รอบ (2,053,666 ns) — ใกล้เคียงกับตัวเลขเดี่ยวๆ ที่เราเห็นในหัวข้อ
  90.5 (1,988,119 ns) แต่แม่นยำกว่าเพราะมาจากหลายรอบ
- **`_median`**: ค่ากึ่งกลางเมื่อเรียงลำดับ ทนทานต่อ outlier มากกว่า mean
- **`_stddev`**: ส่วนเบี่ยงเบนมาตรฐาน (135,548 ns) บอกว่าผลลัพธ์แต่ละรอบกระจายตัวมากแค่ไหน
- **`_cv`** (Coefficient of Variation): stddev หารด้วย mean คิดเป็นเปอร์เซ็นต์ (6.60%) — ยิ่งค่านี้
  ต่ำยิ่งแปลว่าผลลัพธ์เสถียร ค่าที่สูงเกิน 10-15% มักบ่งบอกว่าเครื่องมีสิ่งรบกวนมาก (เช่น โปรแกรม
  พื้นหลังอื่นแย่ง CPU) ควรปิดโปรแกรมพื้นหลังที่ไม่จำเป็นแล้วรันใหม่ก่อนสรุปผล

การรันซ้ำหลายรอบแบบนี้คือคำตอบที่แท้จริงของปัญหาที่เราเห็นในหัวข้อ 90.1 — แทนที่จะเชื่อตัวเลข
จากการรันครั้งเดียว (ซึ่งอาจสูงหรือต่ำผิดปกติได้) เรามีค่าทางสถิติที่บอกทั้ง "ค่ากลาง" และ "ความ
เชื่อถือได้ของค่านั้น" ไปพร้อมกัน

### โบนัส: ให้ Google Benchmark ฟิต Big-O ให้อัตโนมัติด้วย Complexity()

ใน **Part 24** เราเรียนเรื่อง Big-O Notation ในเชิงทฤษฎี Google Benchmark มีฟีเจอร์ที่ช่วยยืนยัน
ความซับซ้อนเชิงทฤษฎีด้วยข้อมูลวัดจริงได้ในตัว ผ่านเมธอด `->Complexity()` และ
`state.SetComplexityN(...)`:

```cpp
#include <benchmark/benchmark.h>
#include <vector>
#include <algorithm>
#include <cstddef>

static void BM_StdSortComplexity(benchmark::State& state) {
    const std::size_t n = static_cast<std::size_t>(state.range(0));
    for (auto _ : state) {
        state.PauseTiming();
        std::vector<int> v(n);
        for (std::size_t i = 0; i < n; i++) v[i] = static_cast<int>(n - i);
        state.ResumeTiming();
        std::sort(v.begin(), v.end());
        benchmark::DoNotOptimize(v);
    }
    state.SetComplexityN(state.range(0)); // บอก Google Benchmark ว่า "N" ของ input คือเท่าไหร่
}
BENCHMARK(BM_StdSortComplexity)
    ->RangeMultiplier(4)
    ->Range(1000, 256000)
    ->Complexity(benchmark::oNLogN); // บอกว่าคาดหวังความซับซ้อนแบบ O(N log N)

BENCHMARK_MAIN();
```

```bash
g++ -O2 -Wall -Wextra -Wpedantic -std=c++17 bm_complexity.cpp -o bm_complexity \
    -lbenchmark -lpthread
./bm_complexity
```

ผลลัพธ์จริง:

```
----------------------------------------------------------------------
Benchmark                            Time             CPU   Iterations
----------------------------------------------------------------------
BM_StdSortComplexity/1000         3962 ns         3964 ns       177233
BM_StdSortComplexity/1024         4117 ns         4106 ns       169510
BM_StdSortComplexity/4096        19363 ns        19361 ns        37788
BM_StdSortComplexity/16384       85566 ns        85453 ns         8272
BM_StdSortComplexity/65536      393256 ns       393195 ns         1801
BM_StdSortComplexity/256000    1728507 ns      1727398 ns          408
BM_StdSortComplexity_BigO         0.38 NlgN       0.38 NlgN
BM_StdSortComplexity_RMS             0 %             0 %
```

สองบรรทัดสุดท้ายคือของแถมพิเศษ: Google Benchmark เก็บข้อมูล (N, เวลา) จากทุกขนาดที่รันไป
แล้วใช้วิธี **Least-Squares Fitting** หาสัมประสิทธิ์ที่เหมาะสมที่สุดเพื่อฟิตกับเส้นโค้ง O(N log N)
ที่เราระบุไว้ ได้ผลเป็น `0.38 NlgN` (แปลว่าเวลาโดยประมาณคือ `0.38 × N × log(N)` หน่วยนาโนวินาที)
และ `RMS` (Root Mean Square ของค่าคลาดเคลื่อนจากเส้นฟิต) เท่ากับ 0% แทบสมบูรณ์แบบ — ยืนยัน
ด้วยข้อมูลจริงว่า `std::sort` มีความซับซ้อนแบบ O(N log N) ตรงตามที่มาตรฐาน C++ รับประกันไว้
เครื่องมือแบบนี้มีประโยชน์มากเวลาต้องการตรวจสอบว่า Algorithm ที่เขียนขึ้นเองมีความซับซ้อนตรงตาม
ที่ออกแบบไว้จริงหรือไม่ (เช่น เผลอเขียนโค้ดที่ควรจะเป็น O(N) แต่กลายเป็น O(N²) โดยไม่รู้ตัว
มักจะสังเกตได้จากค่า `RMS` ที่สูงผิดปกติเมื่อลองฟิตกับเส้นโค้งที่คาดหวังไว้)

### การนำไปใช้ใน CI/CD

ในทางปฏิบัติ ทีมวิศวกรรมมืออาชีพมักรัน benchmark เหล่านี้อัตโนมัติใน CI (จะเรียนเรื่อง GitHub
Actions ใน Part 94) โดยส่งออกผลลัพธ์เป็น `--benchmark_format=json` แล้วนำไปเปรียบเทียบกับ
ผลลัพธ์ของ commit ก่อนหน้า เพื่อจับ **Performance Regression** (โค้ดที่แก้ใหม่ทำให้ช้าลงโดยไม่ได้
ตั้งใจ) โดยอัตโนมัติ ก่อนที่โค้ดนั้นจะถูก merge เข้า branch หลัก — นี่คือเหตุผลที่ Google Benchmark
กลายเป็นเครื่องมือมาตรฐานในวงการ ไม่ใช่แค่เครื่องมือทดลองในห้องเรียนเท่านั้น

---

## ข้อผิดพลาดที่พบบ่อย (Common Pitfalls)

1. **ลืมใส่ `benchmark::DoNotOptimize()`** — เป็นข้อผิดพลาดที่พบบ่อยที่สุดและอันตรายที่สุด
   เพราะไม่มี error/warning ใดๆ เลย ได้แค่ตัวเลขที่ "เร็วเกินจริงอย่างน่าสงสัย" (มักจะใกล้ 0 ns)
   ดังที่พิสูจน์ในหัวข้อ 90.6 — ทุกครั้งที่เห็นตัวเลข benchmark ที่ต่ำผิดปกติ ให้สงสัยเรื่องนี้ก่อนเลย
2. **ลืม link `-lpthread`** — Google Benchmark ใช้ threading ภายในสำหรับบางฟีเจอร์ ถ้าลืม
   `-lpthread` มักจะได้ linker error ประเภท `undefined reference to pthread_create` ทันที
3. **วัด benchmark ที่ `-O0`** — ตัวเลขที่ได้จะไม่สะท้อนประสิทธิภาพจริงของโค้ดที่จะถูกใช้งานใน
   production เลย (ดูความต่างมหาศาลระหว่าง optimization level ใน Part 89 ประกอบ) ควร benchmark
   ที่ `-O2` เป็นอย่างน้อยเสมอ
4. **เขียนการเตรียมข้อมูล (setup) ไว้ในลูปวัดผลโดยตรง** แทนที่จะใช้ Fixture หรือเตรียมข้อมูลไว้
   ก่อนเริ่ม `for (auto _ : state)` ทำให้เวลา setup ปนเข้าไปในผลวัด บิดเบือนตัวเลขที่ได้
5. **เชื่อผลจากการรันครั้งเดียว** โดยไม่ใช้ `--benchmark_repetitions` ตรวจสอบความเสถียรทางสถิติ
   ก่อนสรุปผล โดยเฉพาะเมื่อผลต่างระหว่าง 2 วิธีที่เปรียบเทียบมีขนาดใกล้เคียงกันมาก (เช่น ต่างกัน
   แค่ 2-3%) ซึ่งอาจเป็นแค่ noise ไม่ใช่ความต่างที่มีนัยสำคัญจริง
6. **เปรียบเทียบผล benchmark ข้ามเครื่องหรือข้ามช่วงเวลาที่ต่างกันมาก** โดยไม่บันทึกข้อมูล
   ฮาร์ดแวร์ (CPU, จำนวน core, ความเร็ว) กำกับไว้ด้วย — ตัวเลข benchmark มีความหมายเฉพาะเมื่อ
   เทียบกับบริบทของเครื่องที่ใช้วัดเท่านั้น (Google Benchmark พิมพ์ข้อมูลนี้ให้อัตโนมัติทุกครั้ง
   อย่าตัดทิ้งเวลาบันทึกผลลัพธ์เก็บไว้)

---

## แบบฝึกหัดท้ายบท

1. เขียน benchmark เปรียบเทียบการต่อ string ด้วย `operator+=` ทีละตัวอักษร 10,000 ครั้ง กับ
   การใช้ `std::string::reserve()` ก่อนแล้วค่อย `+=` แบบเดียวกัน ว่าต่างกันมากแค่ไหน (ทำนองเดียว
   กับตัวอย่าง `push_back` ในหัวข้อ 90.5)
2. หยิบโค้ดจากข้อ 1 มาลบ `benchmark::DoNotOptimize()` ออกทั้งหมดโดยตั้งใจ แล้วสังเกตว่าผลลัพธ์
   เปลี่ยนไปอย่างไร อธิบายด้วยคำพูดตัวเองว่าทำไมถึงเกิดเหตุการณ์นั้น (อ้างอิงความรู้จาก Part 89
   เรื่อง Dead Code Elimination ประกอบ)
3. เขียน Benchmark Fixture ที่ทำการค้นหาค่าด้วย `std::find` ใน `std::vector<int>` ที่มีข้อมูลสุ่ม
   10,000 ตัว เทียบกับการค้นหาด้วย `std::unordered_set<int>::find` ที่มีข้อมูลชุดเดียวกัน (ต้อง
   ใช้ Fixture เพราะการสร้าง `unordered_set` จากข้อมูลสุ่มมีค่าใช้จ่ายสูง ไม่ควรนับเวลานี้)
4. ทดลองใช้ `--benchmark_repetitions=10` กับ benchmark ใดก็ได้ที่เขียนไว้ แล้วดูค่า `_cv`
   (Coefficient of Variation) ถ้าค่าสูงเกิน 10% ให้ลองหาสาเหตุ (เช่น ปิดโปรแกรมอื่นที่กำลังรันอยู่
   บนเครื่องแล้วรันใหม่) และสังเกตว่าค่า `_cv` ลดลงหรือไม่
5. เปรียบเทียบ `std::vector<int>` กับ `std::list<int>` สำหรับการ insert ข้อมูล 100,000 ตัวที่
   ตำแหน่งท้ายสุดเสมอ (`push_back`) ด้วย Google Benchmark จริง แล้วอธิบายผลลัพธ์ที่ได้โดยเชื่อมโยง
   กับความรู้เรื่อง Cache-Friendly Code จาก **Part 87**
6. (โบนัส) ลองรัน benchmark เดียวกันด้วย `--benchmark_format=json` แล้วส่งผลลัพธ์ไปยังไฟล์
   ดู structure ของ JSON ที่ได้ อธิบายว่าถ้าจะนำไปใช้ตรวจจับ Performance Regression อัตโนมัติใน
   CI/CD (ดังที่กล่าวถึงในหัวข้อ 90.8) ควรดึง field ไหนออกมาเปรียบเทียบระหว่าง commit เก่ากับใหม่

### แนวทางเฉลยข้อ 1

```cpp
#include <benchmark/benchmark.h>
#include <string>

static void BM_StringAppendNoReserve(benchmark::State& state) {
    for (auto _ : state) {
        std::string s;
        for (int i = 0; i < 10000; i++) {
            s += 'x';
        }
        benchmark::DoNotOptimize(s);
    }
}
BENCHMARK(BM_StringAppendNoReserve);

static void BM_StringAppendWithReserve(benchmark::State& state) {
    for (auto _ : state) {
        std::string s;
        s.reserve(10000);
        for (int i = 0; i < 10000; i++) {
            s += 'x';
        }
        benchmark::DoNotOptimize(s);
    }
}
BENCHMARK(BM_StringAppendWithReserve);

BENCHMARK_MAIN();
```

```bash
g++ -O2 -Wall -Wextra -Wpedantic -std=c++17 bm_string.cpp -o bm_string -lbenchmark -lpthread
./bm_string
```

ผลลัพธ์จริงที่วัดได้บนเครื่องนี้:

```
---------------------------------------------------------------------
Benchmark                           Time             CPU   Iterations
---------------------------------------------------------------------
BM_StringAppendNoReserve         7860 ns         7833 ns        87476
BM_StringAppendWithReserve       7520 ns         7448 ns        72146
```

สังเกตว่าผลต่างในกรณีนี้ (`reserve` เร็วกว่าประมาณ 4-5% เท่านั้น) **น้อยกว่ามาก**เมื่อเทียบกับ
กรณี `std::vector<int>` ในหัวข้อ 90.5 ที่เร็วขึ้นถึง 2.5-3.8 เท่า ทั้งที่กลไกภายในเป็นแบบเดียวกัน
(dynamic array ที่ขยายขนาดเป็นทวีคูณเมื่อเต็ม) เหตุผลคือ `std::string` แต่ละตัวอักษรมีขนาดแค่
1 byte เทียบกับ `int` ที่มีขนาด 4 byte ทำให้ต้นทุนของการ copy ข้อมูลเก่าตอน reallocate (ซึ่งเป็น
สิ่งที่ `reserve()` ช่วยหลีกเลี่ยง) มีสัดส่วนน้อยกว่ามากเมื่อเทียบกับต้นทุนรวมของการต่อ string
ทีละตัวอักษร (ที่มี overhead ของการเรียกฟังก์ชัน `operator+=` ซ้ำ 10,000 ครั้งเป็นตัวถ่วงหลัก)
นี่คือตัวอย่างที่ดีว่าทำไมต้อง**วัดจริงทุกครั้ง** แม้จะเป็นโครงสร้างข้อมูลที่มีกลไกคล้ายกัน ผลต่าง
ที่ได้ก็อาจไม่เหมือนกันเลยขึ้นกับขนาดของ element และลักษณะงานที่แท้จริง จุดสำคัญของแบบฝึกหัด
นี้คือการฝึกเขียนโครงสร้าง benchmark สำหรับเปรียบเทียบ 2 วิธีด้วยตัวเอง ไม่ใช่แค่ก็อปโค้ดตัวอย่าง
มาใช้ซ้ำ และฝึกตีความผลลัพธ์อย่างมีวิจารณญาณแทนที่จะคาดเดาผลไว้ล่วงหน้า

### แนวทางเฉลยข้อ 3

```cpp
#include <benchmark/benchmark.h>
#include <algorithm>
#include <random>
#include <unordered_set>
#include <vector>

class SearchFixture : public benchmark::Fixture {
public:
    std::vector<int> vec_data;
    std::unordered_set<int> set_data;
    int target = -1; // ค่าที่ตั้งใจให้ "หาไม่เจอ" เพื่อวัด worst-case ของการค้นหาทั้ง 2 แบบ

    void SetUp(const benchmark::State&) override {
        std::mt19937 rng(123);
        std::uniform_int_distribution<int> dist(0, 1000000);
        vec_data.clear();
        set_data.clear();
        for (int i = 0; i < 10000; i++) {
            int v = dist(rng);
            vec_data.push_back(v);
            set_data.insert(v);
        }
    }

    void TearDown(const benchmark::State&) override {
        vec_data.clear();
        set_data.clear();
    }
};

BENCHMARK_DEFINE_F(SearchFixture, VectorFind)(benchmark::State& state) {
    for (auto _ : state) {
        auto it = std::find(vec_data.begin(), vec_data.end(), target);
        benchmark::DoNotOptimize(it);
    }
}
BENCHMARK_REGISTER_F(SearchFixture, VectorFind);

BENCHMARK_DEFINE_F(SearchFixture, UnorderedSetFind)(benchmark::State& state) {
    for (auto _ : state) {
        auto it = set_data.find(target);
        benchmark::DoNotOptimize(it);
    }
}
BENCHMARK_REGISTER_F(SearchFixture, UnorderedSetFind);

BENCHMARK_MAIN();
```

```bash
g++ -O2 -Wall -Wextra -Wpedantic -std=c++17 bm_search.cpp -o bm_search -lbenchmark -lpthread
./bm_search
```

ผลลัพธ์จริงที่วัดได้บนเครื่องนี้:

```
-------------------------------------------------------------------------
Benchmark                               Time             CPU   Iterations
-------------------------------------------------------------------------
SearchFixture/VectorFind             2326 ns         2310 ns       301943
SearchFixture/UnorderedSetFind       7.16 ns         7.11 ns     97875299
```

ผลต่างมหาศาลถึง**ประมาณ 325 เท่า** ยืนยันทฤษฎีความซับซ้อนของ Algorithm ได้อย่างชัดเจน:
`VectorFind` มีความซับซ้อน O(n) (ต้องไล่ตรวจทีละตัวจนสุด array 10,000 ตัวเพราะกำหนดให้
`target = -1` หาไม่เจอเสมอ เป็น worst-case ของการค้นหาแบบเชิงเส้น) ในขณะที่
`UnorderedSetFind` มีความซับซ้อนเฉลี่ย O(1) ผ่านกลไก hash table ไม่ว่าจะมีข้อมูลกี่ตัวก็ตาม
เวลาค้นหาแทบไม่เปลี่ยนแปลง จุดสำคัญของแบบฝึกหัดนี้คือการฝึกใช้ Fixture กับข้อมูลที่ต้องเตรียม
2 โครงสร้างพร้อมกัน (`vec_data` และ `set_data`) โดยไม่ให้เวลาสร้าง `unordered_set` (ซึ่งมี
ค่าใช้จ่ายสูงกว่าการสร้าง `vector` มาก เพราะต้องคำนวณ hash และจัดการ bucket ของทุกสมาชิก)
ปนเข้าไปในผลวัดความเร็วของการค้นหาที่เราสนใจจริงๆ — ถ้าลืมใช้ Fixture แล้วสร้าง
`unordered_set` ใหม่ทุกรอบในลูปวัดผล ตัวเลขที่ได้จะสะท้อนต้นทุนการสร้าง ไม่ใช่ต้นทุนการค้นหา
ซึ่งเป็นคนละเรื่องกันโดยสิ้นเชิง

---

## สรุปท้ายบท

ใน Part นี้เราได้:

- พิสูจน์ด้วยการวัดจริงว่าการจับเวลาด้วยมือมี jitter สูงถึงเกือบ 40% ระหว่างการรันแต่ละครั้ง
  แม้จะเป็นโปรแกรมเดียวกันทุกประการ ซึ่งเป็นเหตุผลที่ต้องใช้เครื่องมือ benchmark ที่จัดการเรื่อง
  สถิติให้อัตโนมัติ
- ติดตั้งและ link Google Benchmark (`-lbenchmark -lpthread`) บนเครื่องจริงสำเร็จ
- เขียนโปรแกรม benchmark แรกด้วย `BENCHMARK`/`BENCHMARK_MAIN` และอ่านผลลัพธ์ได้ครบทุก
  คอลัมน์ (Time/CPU/Iterations) รวมถึงเข้าใจความต่างระหว่าง Wall-clock Time กับ CPU Time
- เปรียบเทียบ `std::vector::push_back` แบบมีและไม่มี `reserve()` ด้วยตัวเลขจริง พบว่าการ
  `reserve()` ล่วงหน้าช่วยให้เร็วขึ้นถึง 2.5-3.8 เท่าเมื่อข้อมูลมีขนาดใหญ่
- เข้าใจอย่างลึกซึ้งว่าทำไมต้องใช้ `benchmark::DoNotOptimize()` เสมอ ผ่านตัวอย่างจริงที่ลืมใส่
  แล้วได้ผลลัพธ์ 0.000 ns เพราะ compiler ลบโค้ดที่วัดทิ้งไปทั้งหมด
- เขียน Benchmark Fixture สำหรับกรณีที่ต้อง setup ข้อมูลซับซ้อน โดยไม่ให้เวลา setup ปนเข้าไป
  ในผลวัด และใช้ `PauseTiming()`/`ResumeTiming()` เมื่อจำเป็น
- ใช้ command-line flags ควบคุมการรัน benchmark และอ่านค่าสถิติ (mean/median/stddev/cv)
  เพื่อยืนยันความเสถียรของผลลัพธ์ก่อนสรุปผลจริง

นี่คือ Part สุดท้ายของ **Module G — Concurrency และ Performance Engineering** ตลอด 10 Part
ที่ผ่านมา (Part 81-90) เราเดินทางจาก `std::thread` พื้นฐาน ผ่าน Memory Model, Lock-Free
Programming, Parallel Algorithm, Profiling, Cache-Friendly Design, SIMD, Compiler
Optimization จนมาถึงการวัดผลอย่างเป็นวิทยาศาสตร์ด้วย Google Benchmark — ตอนนี้ผู้เรียนมี
เครื่องมือครบมือสำหรับการเขียนโค้ด C++ ที่เร็วและวัดผลได้จริง ไม่ใช่แค่ "คิดว่าเร็ว"

ใน **Module H — Build Systems, Testing และ Tooling** ที่กำลังจะเริ่มต้นใน Part 91 เราจะยกระดับ
จากการคอมไพล์ไฟล์เดียวด้วย `g++` ตรงๆ ไปสู่การจัดการโปรเจกต์ C++ ขนาดใหญ่อย่างเป็นระบบด้วย
**CMake** ตั้งแต่พื้นฐานไปจนถึงขั้นสูง ซึ่งเป็นทักษะที่ขาดไม่ได้เลยสำหรับการทำงานในโปรเจกต์จริง
ระดับอุตสาหกรรม

**ต่อไป:** Part 91 — CMake ตั้งแต่พื้นฐานถึงขั้นสูง (Module H)
