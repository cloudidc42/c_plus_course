# Part 88: SIMD และ Vectorization เบื้องต้น (Step 697–704)

> Module G — Concurrency และ Performance Engineering | Part 88 จาก 125
> Step ที่ครอบคลุมใน Part นี้: Step 697–704
> Part ก่อนหน้า: [Part 87 — Cache-Friendly Code และ Data-Oriented Design](./part-087-cache-friendly-dod.md) | Part ถัดไป: [Part 89 — Compiler Optimization และ Assembly](./part-089-compiler-optimizations.md)

## เป้าหมายการเรียนรู้ (Learning Objectives)

หลังจบ Part นี้ ผู้เรียนจะสามารถ:

1. อธิบาย **SIMD (Single Instruction, Multiple Data)** ได้ว่าคืออะไร ต่างจากการประมวลผลแบบ
   ปกติ (Scalar) อย่างไร และทำไมถึงเป็นกุญแจสำคัญของ performance บน CPU สมัยใหม่
2. อธิบายว่า **Auto-vectorization** คืออะไร compiler เปิดใช้งานอัตโนมัติเมื่อไหร่ (`-O2`/`-O3`)
   และอ่าน assembly จริงด้วย `objdump` เพื่อยืนยันว่า loop ถูก vectorize จริงหรือไม่
3. ใช้ flag `-fopt-info-vec` เพื่อให้ compiler รายงานตรงๆ ว่า loop ไหน vectorize สำเร็จ loop
   ไหนไม่สำเร็จ และเพราะเหตุใด
4. เขียน loop ให้ compiler vectorize ได้ง่ายขึ้น โดยหลีกเลี่ยง loop-carried dependency และใช้
   `__restrict` บอก compiler ว่า pointer ไม่ overlap กัน
5. อธิบายภาพรวมของ SIMD intrinsics (`<immintrin.h>`, ตระกูล SSE/AVX/AVX2/AVX-512) และเขียน
   โปรแกรมที่เรียกใช้ intrinsics โดยตรงได้เอง คอมไพล์และรันได้จริงบน x86_64
6. วัด **speedup จริง** ระหว่าง scalar loop, auto-vectorized loop และ intrinsics ที่เขียนมือ
   ด้วย `-O3 -march=native` แล้วรายงานตัวเลขที่วัดได้จริง ไม่ใช่ตัวเลขทางทฤษฎี
7. รู้จักข้อผิดพลาดที่พบบ่อยเกี่ยวกับ SIMD/Vectorization เช่น การเชื่อว่า compiler vectorize
   ให้แล้วโดยไม่ตรวจสอบ หรือลืมผลกระทบของ floating-point reordering

---

## 88.1 SIMD คืออะไร (Step 697)

ใน Part 87 เราเห็นแล้วว่าการจัดวางข้อมูลใน memory ให้ cache-friendly ช่วยลดจำนวนครั้งที่ CPU
ต้องรอ RAM แต่ต่อให้ข้อมูลอยู่ใน L1 cache พร้อมหมดแล้ว CPU ปกติ (Scalar) ก็ยังประมวลผลได้
**ทีละค่า** ต่อหนึ่งคำสั่งอยู่ดี — นี่คือจุดที่ **SIMD (Single Instruction, Multiple Data)**
เข้ามาช่วย

**SIMD** คือความสามารถของ CPU ที่ใช้ **คำสั่งเดียว** ประมวลผล **ข้อมูลหลายค่าพร้อมกัน** ในรอบ
สัญญาณนาฬิกาเดียว โดยอาศัย register พิเศษที่กว้างกว่า register ปกติมาก:

```
Scalar (ปกติ):  1 คำสั่ง ADD  ทำงานกับ float 1 ตัว
                ┌────┐
                │ 3.0│  +  ┌────┐     ┌────┐
                └────┘     │ 2.0│  =  │ 5.0│
                           └────┘     └────┘
                (ต้องทำ 8 รอบ ถ้ามี float 8 ตัวที่ต้องบวก)

SIMD (AVX2):    1 คำสั่ง VADDPS ทำงานกับ float 8 ตัวพร้อมกันในคำสั่งเดียว
                ┌────┬────┬────┬────┬────┬────┬────┬────┐
                │ 3.0│ 1.0│ 4.0│ 1.0│ 5.0│ 9.0│ 2.0│ 6.0│  ymm register (256-bit)
                └────┴────┴────┴────┴────┴────┴────┴────┘
                  +     +     +     +     +     +     +     +
                ┌────┬────┬────┬────┬────┬────┬────┬────┐
                │ 2.0│ 7.0│ 1.0│ 8.0│ 2.0│ 8.0│ 1.0│ 8.0│
                └────┴────┴────┴────┴────┴────┴────┴────┘
                  =     =     =     =     =     =     =     =
                ┌────┬────┬────┬────┬────┬────┬────┬────┐
                │ 5.0│ 8.0│ 5.0│ 9.0│ 7.0│17.0│ 3.0│14.0│  (ผลลัพธ์ 8 ตัวพร้อมกัน)
                └────┴────┴────┴────┴────┴────┴────┴────┘
```

### ตระกูลของชุดคำสั่ง SIMD บน x86_64

CPU x86_64 มีชุดคำสั่ง SIMD หลายรุ่นต่อยอดกันมา แต่ละรุ่นเพิ่มความกว้างของ register:

| ชุดคำสั่ง | ปีที่เปิดตัว | ความกว้าง register | จำนวน float 32-bit ต่อคำสั่ง |
|---|---|---|---|
| SSE / SSE2 | 1999-2001 | 128 bit (`xmm`) | 4 |
| AVX | 2011 | 256 bit (`ymm`) | 8 |
| AVX2 | 2013 | 256 bit (`ymm`, เพิ่มคำสั่ง integer) | 8 |
| AVX-512 | 2016+ | 512 bit (`zmm`) | 16 |

เครื่องที่ใช้เขียนบทเรียนนี้รองรับ AVX-512 ด้วย ตรวจสอบได้จาก:

```bash
lscpu | grep -o "avx[a-z0-9_]*" | sort -u
```

```
avx
avx2
avx512_bf16
avx512bw
avx512cd
avx512dq
avx512f
avx512ifma
avx512vbmi
avx512vbmi2
avx512vl
avx512_vnni
avx_vnni
```

### ทำไม SIMD ถึงสำคัญ

งานประมวลผลข้อมูลจำนวนมากที่ทำ "การคำนวณเดียวกันซ้ำๆ กับข้อมูลคนละตัว" (เช่น บวกอาร์เรย์
สองก้อน, ประมวลผลภาพทีละพิกเซล, คำนวณ physics ของอนุภาคจำนวนมาก) เป็นรูปแบบงานที่เหมาะกับ
SIMD ที่สุด เพราะข้อมูลแต่ละตัวไม่ขึ้นกับกันและกัน (independent) — ตรงกับสถานการณ์ **Data
Parallelism** ทฤษฎีบอกว่าถ้าย้ายจาก scalar ไป AVX2 ได้เต็มที่ ควรได้ speedup ถึง 8 เท่า (สำหรับ
`float`) แต่ในทางปฏิบัติจริงมักได้น้อยกว่านั้นมาก เพราะมีปัจจัยอื่น เช่น memory bandwidth,
การจัดการส่วนที่เหลือ (remainder) ที่ไม่ครบ 8 ตัว, และ overhead อื่นๆ เข้ามาเกี่ยวข้อง — เราจะ
วัดตัวเลขจริงในหัวข้อถัดไป

---

## 88.2 Auto-Vectorization: สิ่งที่ Compiler ทำให้อัตโนมัติ (Step 698)

ข่าวดีคือโปรแกรมเมอร์**ไม่จำเป็นต้องเขียน SIMD ด้วยมือเสมอไป** compiler สมัยใหม่ (GCC, Clang)
มีความสามารถที่เรียกว่า **Auto-vectorization** คือการวิเคราะห์ loop ธรรมดาที่เราเขียนแล้วแปลง
เป็นคำสั่ง SIMD ให้อัตโนมัติ โดยที่ source code ไม่ต้องเปลี่ยนแปลงอะไรเลย

### ทดลองดู Assembly จริง: -O0 vs -O3 -march=native

มาดูของจริงกันด้วยฟังก์ชันบวกอาร์เรย์ธรรมดา:

```cpp
// vecadd.cpp - ตัวอย่าง loop ง่ายๆ ที่ compiler ควร auto-vectorize ได้
#include <cstdio>
#include <chrono>
#include <vector>
#include <cstdlib>

constexpr size_t N = 200'000'000;

// ฟังก์ชันแยกออกมาต่างหาก + ใช้ __restrict เพื่อบอก compiler ว่า a, b, c ไม่ overlap กัน
// ทำให้ compiler มั่นใจได้ว่า vectorize แล้วผลลัพธ์จะไม่ผิดเพี้ยน
void add_arrays(const float* __restrict a, const float* __restrict b,
                 float* __restrict c, size_t n) {
    for (size_t i = 0; i < n; ++i) {
        c[i] = a[i] + b[i];
    }
}

int main() {
    std::vector<float> a(N), b(N), c(N);
    for (size_t i = 0; i < N; ++i) {
        a[i] = static_cast<float>(i % 1000) * 0.5f;
        b[i] = static_cast<float>(i % 777) * 0.25f;
    }

    const int REPEAT = 5;
    double total_ms = 0.0;
    for (int r = 0; r < REPEAT; ++r) {
        auto t1 = std::chrono::steady_clock::now();
        add_arrays(a.data(), b.data(), c.data(), N);
        auto t2 = std::chrono::steady_clock::now();
        total_ms += std::chrono::duration<double, std::milli>(t2 - t1).count();
    }

    std::printf("N = %zu, REPEAT = %d\n", N, REPEAT);
    std::printf("เวลาเฉลี่ยต่อรอบ: %.3f ms\n", total_ms / REPEAT);
    std::printf("c[123456] = %f (กัน compiler optimize ทิ้งทั้งหมด)\n", c[123456]);
    return 0;
}
```

สร้าง object file สองแบบเพื่อเทียบ assembly:

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 -O0 -c vecadd.cpp -o vecadd_O0.o
g++ -Wall -Wextra -Wpedantic -std=c++17 -O3 -march=native -c vecadd.cpp -o vecadd_O3.o
objdump -d --disassemble=_Z10add_arraysPKfS0_Pfm vecadd_O0.o
objdump -d --disassemble=_Z10add_arraysPKfS0_Pfm vecadd_O3.o
```

(ชื่อฟังก์ชันหลัง `--disassemble=` เป็นชื่อที่ผ่าน name mangling ของ C++ แล้ว หาได้ด้วย
`objdump -t vecadd_O0.o | grep add_arrays` หรือ `nm vecadd_O0.o | c++filt`)

**Assembly จริงที่ `-O0` (ไม่ optimize เลย) — ประมวลผลทีละ 1 float:**

```
30:	48 8b 45 f8          	mov    -0x8(%rbp),%rax
...
35:	f3 0f 10 08          	movss  (%rax),%xmm1     ; โหลด float 1 ตัวจาก a
4c:	f3 0f 10 00          	movss  (%rax),%xmm0     ; โหลด float 1 ตัวจาก b
63:	f3 0f 58 c1          	addss  %xmm1,%xmm0      ; บวก float ตัวเดียว (Scalar Single)
67:	f3 0f 11 00          	movss  %xmm0,(%rax)     ; เก็บผลลัพธ์ 1 ตัว
6b:	48 83 45 f8 01       	addq   $0x1,-0x8(%rbp)  ; i++
```

สังเกตคำสั่ง `movss`/`addss` — ตัว `ss` ย่อมาจาก **"Scalar Single"** คือทำงานกับ float แค่
1 ตัวต่อคำสั่ง แม้จะใช้ register `xmm` (ซึ่งกว้าง 128 bit รองรับ 4 float) แต่ก็ใช้แค่ 1 ช่อง
เท่านั้น เพราะที่ `-O0` compiler ไม่พยายาม optimize อะไรเลย

**Assembly จริงที่ `-O3 -march=native` — ประมวลผลทีละ 8 float ด้วย AVX2:**

```
30:	c5 fc 10 0c 07       	vmovups (%rdi,%rax,1),%ymm1   ; โหลด float 8 ตัวจาก a เข้า ymm (256-bit)
35:	c5 f4 58 04 06       	vaddps (%rsi,%rax,1),%ymm1,%ymm0  ; บวกกับ b ทีละ 8 ตัวพร้อมกัน
3a:	c5 fc 11 04 02       	vmovups %ymm0,(%rdx,%rax,1)   ; เก็บผลลัพธ์ 8 ตัวพร้อมกัน
3f:	48 83 c0 20          	add    $0x20,%rax             ; เลื่อนไป 32 bytes (8 floats)
```

คำสั่ง `vmovups`/`vaddps` ทำงานกับ register `ymm` (256-bit) เต็มความกว้าง ตัว `ps` ย่อมาจาก
**"Packed Single"** คือประมวลผล float 8 ตัวพร้อมกันในคำสั่งเดียว! และหลัง loop หลักจบ compiler
ยังสร้างโค้ดจัดการ **remainder** (ส่วนที่เหลือไม่ครบ 8 ตัว) ด้วย `xmm` (4 ตัว) แล้วค่อย fallback
เป็น scalar (`vmovss`/`vaddss`) ทีละตัวสำหรับส่วนที่เหลือจริงๆ — ทั้งหมดนี้ **compiler ทำให้เอง
โดยที่เราไม่ต้องเขียนอะไรเพิ่มจาก loop ธรรมดาเลย**

### วัดผลจริง: O0 vs O2 vs O3 -march=native

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 -O0 vecadd.cpp -o vecadd_O0
g++ -Wall -Wextra -Wpedantic -std=c++17 -O2 vecadd.cpp -o vecadd_O2
g++ -Wall -Wextra -Wpedantic -std=c++17 -O3 -march=native vecadd.cpp -o vecadd_O3native
./vecadd_O0
./vecadd_O2
./vecadd_O3native
```

**ผลลัพธ์ที่วัดได้จริง (N = 200,000,000 floats, เฉลี่ย 5 รอบ):**

| Flag | เวลาเฉลี่ยต่อรอบ (ms) | Speedup เทียบ O0 |
|---|---:|---:|
| `-O0` | 355.73 | 1.00x |
| `-O2` | 185.99 | 1.91x |
| `-O3 -march=native` | 191.52 | 1.86x |

**ผลลัพธ์ที่ได้ไม่ตรงกับที่คาดหวังตอนแรก** — ทฤษฎีบอกว่า AVX2 ควรเร็วกว่า scalar ถึง 8 เท่า
แต่วัดจริงได้แค่ ~1.9 เท่า และ `-O3 -march=native` ก็ไม่ได้เร็วกว่า `-O2` เลย (แม้จะใช้ AVX2/
AVX-512 register ที่กว้างกว่า) **เหตุผลคือ loop นี้เป็น Memory-Bound ไม่ใช่ Compute-Bound**:
อาร์เรย์แต่ละก้อนมีขนาด 800 MB (200 ล้าน x 4 bytes) รวม 3 ก้อน = 2.4 GB ต้องอ่าน/เขียนผ่าน RAM
ทั้งหมด ซึ่งความเร็วถูกจำกัดด้วย **แบนด์วิดท์หน่วยความจำ (memory bandwidth)** ไม่ใช่ความเร็วของ
หน่วยคำนวณ (ALU) เลย ต่อให้ CPU บวกได้เร็วกว่านี้อีกกี่เท่า ก็ยังต้องรอข้อมูลไหลผ่าน RAM ด้วย
ความเร็วเท่าเดิม — นี่คือบทเรียนสำคัญที่สุดข้อหนึ่งของบทนี้: **SIMD ช่วยได้มากก็ต่อเมื่องานเป็น
Compute-Bound เท่านั้น ถ้างานเป็น Memory-Bound ต้องแก้ที่การเข้าถึงหน่วยความจำ (ตามที่เรียนใน
Part 87) ไม่ใช่แก่ที่การคำนวณ**

### ลองใหม่กับ Loop ที่ Compute-Bound มากขึ้น

มาลองกับ loop ที่คำนวณหนักขึ้นต่อ 1 element (polynomial) และใช้ข้อมูลที่เล็กพอจะอยู่ใน cache
ได้เกือบทั้งหมด (ไม่ต้องพึ่ง RAM บ่อย):

```cpp
// polybench.cpp - loop ที่ compute-bound มากขึ้น (คำนวณ polynomial ต่อ element)
#include <cstdio>
#include <chrono>
#include <vector>

constexpr size_t N = 2'000'000;   // เล็กพอจะอยู่ใน cache ได้เกือบทั้งหมด
constexpr int REPEAT = 300;

// คำนวณ polynomial ง่ายๆ: out[i] = a[i]^3 - 2*a[i]^2 + 3*a[i] + b[i]
// ไม่มี dependency ระหว่าง iteration เลย -> compiler vectorize ได้เต็มที่
void poly(const float* __restrict a, const float* __restrict b,
          float* __restrict out, size_t n) {
    for (size_t i = 0; i < n; ++i) {
        float x = a[i];
        out[i] = x * x * x - 2.0f * x * x + 3.0f * x + b[i];
    }
}

int main() {
    std::vector<float> a(N), b(N), out(N);
    for (size_t i = 0; i < N; ++i) {
        a[i] = static_cast<float>(i % 500) * 0.01f;
        b[i] = static_cast<float>(i % 300) * 0.02f;
    }

    auto t1 = std::chrono::steady_clock::now();
    for (int r = 0; r < REPEAT; ++r) {
        poly(a.data(), b.data(), out.data(), N);
    }
    auto t2 = std::chrono::steady_clock::now();

    double total_ms = std::chrono::duration<double, std::milli>(t2 - t1).count();
    std::printf("N = %zu, REPEAT = %d\n", N, REPEAT);
    std::printf("เวลารวม: %.3f ms, เฉลี่ยต่อรอบ: %.5f ms\n", total_ms, total_ms / REPEAT);
    std::printf("out[12345] = %f (กัน compiler optimize ทิ้งทั้งหมด)\n", out[12345]);
    return 0;
}
```

**ผลลัพธ์ที่วัดได้จริง (N = 2,000,000, REPEAT = 300 รอบ):**

| Flag | เวลาเฉลี่ยต่อรอบ (ms) | Speedup เทียบ O0 |
|---|---:|---:|
| `-O0` | 5.829 | 1.00x |
| `-O2` | 0.913 | 6.38x |
| `-O3 -march=native` | 0.953 | 6.12x |

คราวนี้เห็น speedup ที่มากขึ้นชัดเจน (~6.4 เท่า) เพราะข้อมูลขนาด 2M floats (8 MB ต่ออาร์เรย์
รวม 24 MB ทั้ง 3 ก้อน) เกิน L2 cache ต่อ core (2 MiB) แต่ยังพอดีกับ L3 cache ของเครื่องนี้ ทำให้
เป็น loop ที่ compute เริ่มมีน้ำหนักเทียบเท่ากับการเข้าถึงหน่วยความจำมากขึ้น การ vectorize จึง
เห็นผลชัดกว่าตัวอย่างแรก แต่ยังคง**ไม่เห็นความต่างระหว่าง `-O2` กับ `-O3 -march=native`**
ซึ่งเราจะสืบสวนต่อในหัวข้อถัดไปว่าเกิดจากอะไร

---

## 88.3 ตรวจสอบว่า Vectorize จริงหรือไม่ ด้วย `-fopt-info-vec` (Step 699)

จากผลลัพธ์ในหัวข้อที่แล้วที่ `-O2` กับ `-O3 -march=native` ให้ความเร็วใกล้เคียงกันมาก คำถามคือ
**`-O2` vectorize loop นี้ไปแล้วหรือยัง?** แทนที่จะเดา เราให้ compiler บอกเราตรงๆ ด้วย flag
`-fopt-info-vec-optimized`:

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 -O2 -fopt-info-vec-optimized -c polybench.cpp -o /dev/null
```

**ผลลัพธ์จริง:**

```
polybench.cpp:14:26: optimized: loop vectorized using 16 byte vectors
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 -O3 -march=native -fopt-info-vec-optimized -c polybench.cpp -o /dev/null
```

**ผลลัพธ์จริง:**

```
polybench.cpp:14:26: optimized: loop vectorized using 32 byte vectors
polybench.cpp:14:26: optimized: loop vectorized using 16 byte vectors
polybench.cpp:14:26: optimized: loop vectorized using 32 byte vectors
```

นี่คือคำตอบของปริศนาในหัวข้อก่อนหน้า: **`-O2` ก็ vectorize loop นี้แล้วเช่นกัน** เพียงแต่ใช้
"16 byte vectors" (128-bit, เท่ากับ SSE) เพราะไม่มี `-march=native` บอกว่า CPU รองรับ AVX
ส่วน `-O3 -march=native` ใช้ "32 byte vectors" (256-bit, AVX2) ที่กว้างกว่า 2 เท่า — เหตุผล
ที่เวลาไม่ต่างกันมากคือ ตัว loop นี้ (N=2,000,000) มีขนาดข้อมูลใหญ่กว่า L2 cache ทำให้การรอ
โหลดข้อมูลจาก L3 กลายเป็นคอขวดที่บดบังประโยชน์ของ register ที่กว้างขึ้นไปเกือบหมด

> **ข้อเท็จจริงสำคัญที่ตำราเก่าหลายเล่มไม่ทันอัปเดต**: ในอดีต (GCC รุ่นเก่า) มีแค่ `-O3` เท่านั้น
> ที่เปิด `-ftree-vectorize` ให้อัตโนมัติ แต่ตั้งแต่ GCC 12 เป็นต้นมา (รวมถึง GCC 13.3.0 ที่ใช้
> ในบทเรียนนี้) `-O2` ก็เปิด **`-ftree-loop-vectorize`** และ **`-ftree-slp-vectorize`** ให้
> อัตโนมัติแล้วเช่นกัน ตรวจสอบได้ด้วยตัวเองผ่าน:
> ```bash
> g++ -O2 -Q --help=optimizers | grep -i vectorize
> ```
> ```
> -ftree-loop-vectorize       		[enabled]
> -ftree-slp-vectorize        		[enabled]
> -ftree-vectorize            		[disabled]
> ```
> (`-ftree-vectorize` เองเป็นแค่ flag "รวม" ที่ตั้งค่าอีกสองตัวข้างต้น ไม่ใช่ flag จริงที่มีผล
> โดยตรง — มันเลยรายงานว่า disabled แม้สองตัวที่มันควบคุมจะ enabled อยู่แล้วก็ตาม) สิ่งที่
> `-O3` เพิ่มเข้ามาจริงๆ เหนือ `-O2` ในเรื่อง vectorization คือ heuristic ที่ก้าวร้าวกว่า (กล้า
> vectorize loop ที่ซับซ้อนกว่า, ทำ loop unrolling ร่วมกับ vectorize มากกว่า) ไม่ใช่ "เปิด
> vectorization" อย่างที่เข้าใจกันมาก่อน

### ทดสอบกับข้อมูลที่เล็กพอจะอยู่ใน Cache

มาดูว่าเมื่อลดขนาดข้อมูลให้เล็กพอจะพอดี L2 cache (ไม่ต้องพึ่ง L3 บ่อย) ความกว้างของ register
จะเห็นผลชัดขึ้นหรือไม่ — ลด `N` จาก 2,000,000 เหลือ `100,000` และเพิ่ม `REPEAT` เป็น `6000`
เพื่อให้วัดเวลารวมได้แม่นยำ:

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 -O2 polybench_small.cpp -o polysmall_O2
g++ -Wall -Wextra -Wpedantic -std=c++17 -O3 -march=native polybench_small.cpp -o polysmall_O3native
```

**ผลลัพธ์ที่วัดได้จริง (N = 100,000, REPEAT = 6000, รันซ้ำ 2 ครั้ง):**

| Flag | เวลาเฉลี่ยต่อรอบ (ms) |
|---|---:|
| `-O2` (SSE, 16-byte) — รอบ 1 | 0.02434 |
| `-O2` (SSE, 16-byte) — รอบ 2 | 0.02551 |
| `-O3 -march=native` (AVX2, 32-byte) — รอบ 1 | 0.01905 |
| `-O3 -march=native` (AVX2, 32-byte) — รอบ 2 | 0.02103 |

คราวนี้เมื่อข้อมูลเล็กพอ (100,000 x 4 bytes x 3 อาร์เรย์ = 1.2 MB, พอดีกับ L2) `-O3
-march=native` เร็วกว่า `-O2` อย่างสม่ำเสมอราว **1.3 เท่า** — สอดคล้องกับทฤษฎีที่ว่า AVX2
(256-bit) ควรเร็วกว่า SSE (128-bit) มากกว่านี้เมื่อเป็น compute-bound มากขึ้น (ไม่ใช่ทั้ง 2
เท่าเป๊ะเพราะยังมี overhead อื่น เช่น loop control และการ store ผลลัพธ์กลับ cache)

**บทเรียนสำคัญของหัวข้อนี้**: **อย่าเชื่อว่า flag ทำงานตามที่คาดไว้โดยไม่ตรวจสอบ** ต้องใช้
`-fopt-info-vec` หรือ `objdump` ยืนยันเสมอว่า loop ถูก vectorize จริง และต้องวัดเวลาในสถานการณ์
ที่ใกล้เคียงงานจริง เพราะ **ผลของ vectorization ขึ้นกับว่างานเป็น compute-bound หรือ
memory-bound เป็นหลัก** ไม่ใช่แค่ "เปิด flag แล้วต้องเร็วขึ้นเสมอ"

---

## 88.4 เขียน Loop ให้ Compiler Vectorize ได้ง่ายขึ้น (Step 700)

Auto-vectorization ไม่ใช่เวทมนตร์ — compiler ต้อง**พิสูจน์ทางคณิตศาสตร์**ได้ว่าการจัดกลุ่ม
คำสั่งใหม่ (ทำหลาย iteration พร้อมกัน) จะให้ผลลัพธ์เหมือนเดิมทุกประการ ถ้า compiler พิสูจน์ไม่ได้
มันจะไม่กล้า vectorize เลย (เพราะจะทำให้โปรแกรมทำงานผิดจากที่ผู้เขียนตั้งใจ) ปัจจัยสำคัญ 2 ข้อ
ที่ทำให้ compiler "พิสูจน์ไม่ได้" คือ **Pointer Aliasing** และ **Loop-Carried Dependency**

### ปัญหาที่ 1: Pointer Aliasing และ `__restrict`

```cpp
#include <cstddef>

// ไม่มี __restrict -> compiler ไม่รู้ว่า a, b, c ชี้ทับซ้อนกันหรือไม่ อาจไม่กล้า vectorize
void add_no_restrict(float* a, float* b, float* c, size_t n) {
    for (size_t i = 0; i < n; ++i) {
        c[i] = a[i] + b[i];
    }
}

// มี __restrict -> รับประกันว่าไม่ทับซ้อนกัน compiler vectorize ได้อย่างมั่นใจ
void add_restrict(float* __restrict a, float* __restrict b, float* __restrict c, size_t n) {
    for (size_t i = 0; i < n; ++i) {
        c[i] = a[i] + b[i];
    }
}
```

ปัญหาของ `add_no_restrict` คือ: ถ้า `c` ชี้ไปที่ตำแหน่งเดียวกับ `a` แต่เยื้องไป 1 ตำแหน่ง
(เช่น `c == a + 1`) การประมวลผลทีละ 8 ค่าพร้อมกันอาจเขียนทับค่าที่ iteration ถัดไปยังไม่ทันอ่าน
— ทำให้ผลลัพธ์ผิดจาก scalar loop ต้นฉบับ! คำสำคัญ **`__restrict`** (GNU/MSVC extension ที่
compiler สมัยใหม่รองรับกันหมด แม้จะไม่ใช่ keyword มาตรฐานของ C++) คือคำมั่นสัญญาที่โปรแกรมเมอร์
ให้กับ compiler ว่า **"pointer ตัวนี้ไม่ทับซ้อนกับ pointer ตัวอื่นในขอบเขตนี้แน่นอน"**

ทดสอบด้วย `-fopt-info-vec-all` เพื่อดูว่า compiler ตัดสินใจอย่างไรกับทั้งสองแบบ:

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 -O3 -march=native -fopt-info-vec-all \
    -c restrict_demo.cpp -o restrict_demo.o
```

**ผลลัพธ์จริงสำหรับ `add_no_restrict` (ไม่มี `__restrict`):**

```
restrict_demo.cpp:5:26: optimized: loop vectorized using 32 byte vectors
restrict_demo.cpp:5:26: optimized:  loop versioned for vectorization because of possible aliasing
```

**ผลลัพธ์จริงสำหรับ `add_restrict` (มี `__restrict`):**

```
restrict_demo.cpp:12:26: optimized: loop vectorized using 32 byte vectors
```

ผลลัพธ์ที่ได้น่าประหลาดใจ: **GCC 13 ฉลาดพอที่จะ vectorize `add_no_restrict` ได้เหมือนกัน!**
แต่ทำผ่านเทคนิคที่เรียกว่า **Loop Versioning**: มันสร้างโค้ดขึ้นมา 2 ชุด — ชุดที่ vectorize
แล้ว กับชุด scalar ธรรมดา — พร้อมใส่เงื่อนไขตรวจสอบ (runtime check) ว่า `a`, `b`, `c` ทับซ้อน
กันจริงหรือไม่ ณ ตอนรันโปรแกรม ถ้าไม่ทับซ้อนก็ใช้ชุดที่ vectorize แล้ว ถ้าทับซ้อนก็ fallback
ไปใช้ scalar เพื่อความปลอดภัย **ข้อเสียของวิธีนี้คือมี overhead เพิ่ม** (ต้องเช็คเงื่อนไขทุกครั้ง
ที่เรียกฟังก์ชัน และโค้ดมีขนาดใหญ่ขึ้นเป็นสองเท่า) ในขณะที่ `__restrict` ทำให้ compiler มั่นใจ
ได้ตั้งแต่ compile-time โดยไม่ต้องเสีย overhead ตรวจสอบตอนรันเลย **จึงยังคงแนะนำให้ใช้
`__restrict` เสมอเมื่อรู้แน่ชัดว่า pointer ไม่ทับซ้อนกัน** แทนที่จะหวังพึ่ง loop versioning
ของ compiler

### ปัญหาที่ 2: Loop-Carried Dependency

```cpp
#include <cstddef>

// loop ที่มี loop-carried dependency (ผลลัพธ์ iteration ถัดไปขึ้นกับ iteration ก่อนหน้า)
// -> vectorize ไม่ได้ในความหมายทั่วไป เพราะต้องรอผลลัพธ์ก่อนหน้าก่อนเสมอ
void prefix_sum(const float* __restrict a, float* __restrict out, size_t n) {
    float running = 0.0f;
    for (size_t i = 0; i < n; ++i) {
        running += a[i];
        out[i] = running;
    }
}
```

**ผลลัพธ์จริงจาก `-fopt-info-vec-all`:**

```
restrict_demo.cpp:21:26: missed: couldn't vectorize loop
restrict_demo.cpp:22:17: missed: not vectorized: unsupported use in stmt.
```

ปัญหาของ `prefix_sum` คือค่า `out[i]` ขึ้นอยู่กับ `running` ซึ่งเป็นผลสะสมจาก **ทุก** iteration
ก่อนหน้าโดยตรง (นี่คือ **prefix sum / running total** แบบคลาสสิก) การจะคำนวณ `out[5]` ได้ต้องรู้
`out[4]` ก่อนเสมอ — ไม่สามารถคำนวณ 8 ค่าพร้อมกันแบบอิสระได้เลยในความหมายตรงไปตรงมา (ในทาง
ปฏิบัติมีอัลกอริทึมพิเศษสำหรับ vectorize prefix sum ได้เหมือนกัน เช่น "prefix sum by doubling"
แต่ซับซ้อนกว่า loop ธรรมดามาก และ compiler ทั่วไปจะไม่ทำให้อัตโนมัติ)

**หลักการเขียน loop ให้ vectorize ง่าย สรุปเป็นข้อๆ**:

1. แต่ละ iteration ต้อง**ไม่ขึ้นกับผลลัพธ์ของ iteration อื่น** (independent)
2. ใช้ `__restrict` กับ pointer parameter ทุกตัวที่รู้แน่ชัดว่าไม่ overlap กัน
3. หลีกเลี่ยงการเรียกฟังก์ชันที่ compiler มองไม่ทะลุ (เช่น ฟังก์ชันจาก library ภายนอกที่ไม่มี
   source ให้ compiler วิเคราะห์) ภายใน loop — เปลี่ยนไปใช้ฟังก์ชันจาก `<cmath>` มาตรฐานที่
   compiler รู้จักเป็นพิเศษ (built-in) แทน
4. หลีกเลี่ยง branch (if/else) ที่ซับซ้อนภายใน loop เพราะ SIMD ทำ "เลือกทำแบบมีเงื่อนไข" ได้
   ยากกว่าปกติ (ต้องใช้เทคนิค masking ซึ่ง compiler ทำให้อัตโนมัติได้บ้างแต่ไม่เสมอไป)
5. ให้ compiler เห็นขนาด loop (`n`) และไม่มี aliasing ตั้งแต่ตอน compile ให้ได้มากที่สุด เช่น
   ผ่าน `__restrict` หรือใช้ `#pragma GCC ivdep` (บอก compiler ว่า "เชื่อฉันเถอะ ไม่มี
   dependency จริง" — ใช้เมื่อมั่นใจเท่านั้น เพราะถ้าใช้ผิดจะได้ Undefined Behavior)

---

## 88.5 เกริ่นนำ SIMD Intrinsics (Step 701)

แม้ auto-vectorization จะช่วยได้เยอะ แต่ก็มีข้อจำกัด: compiler ต้อง "เดา" pattern ที่เหมาะสม
เอง ถ้า algorithm ซับซ้อนเกินไป หรือใช้เทคนิคเฉพาะทาง (เช่น shuffle ข้อมูลข้าม lane, ใช้คำสั่ง
พิเศษเช่น popcount แบบ vectorized) compiler อาจ vectorize ได้ไม่เต็มประสิทธิภาพ หรือไม่
vectorize เลย ในกรณีนี้โปรแกรมเมอร์สามารถเขียน SIMD **ด้วยมือโดยตรง** ผ่าน **Intrinsics**

### Intrinsics คืออะไร

**Intrinsics** คือฟังก์ชันพิเศษที่ compiler รู้จักเป็นกรณีพิเศษ (built-in) แปลตรงเป็นคำสั่ง
SIMD ของ CPU แบบ 1 ต่อ 1 โดยไม่ต้องเขียน inline assembly เอง (ซึ่งยากและอันตรายกว่ามาก)
มองว่าเป็น "จุดกึ่งกลาง" ระหว่างการเขียน C++ ปกติ กับการเขียน assembly ตรงๆ

Header หลักที่รวม intrinsics ของ x86 ทั้งหมดไว้คือ `<immintrin.h>`:

```cpp
#include <immintrin.h>
```

### ชนิดข้อมูลและคำสั่งพื้นฐาน

| ชนิดข้อมูล (Type) | ความกว้าง | ใช้กับชุดคำสั่ง |
|---|---|---|
| `__m128` | 128-bit (4 x float) | SSE |
| `__m256` | 256-bit (8 x float) | AVX/AVX2 |
| `__m512` | 512-bit (16 x float) | AVX-512 |
| `__m128i` / `__m256i` / `__m512i` | เหมือนข้างบนแต่เก็บ integer | SSE2 / AVX2 / AVX-512 |
| `__m128d` / `__m256d` / `__m512d` | เหมือนข้างบนแต่เก็บ double | SSE2 / AVX / AVX-512 |

รูปแบบชื่อฟังก์ชันมีแพทเทิร์นคงที่ เข้าใจง่ายเมื่อรู้กติกา: `_mm256_<operation>_<type_suffix>`

| Suffix | ความหมาย |
|---|---|
| `ps` | Packed Single (float หลายตัว) |
| `pd` | Packed Double (double หลายตัว) |
| `epi32` | Packed Extended Integer 32-bit (int หลายตัว) |
| `ss` / `sd` | Scalar Single/Double (ค่าเดียว, ใช้ในกรณีพิเศษ) |

ตัวอย่างเช่น `_mm256_add_ps` แปลว่า "บวก float packed ในความกว้าง 256 bit (AVX2)" ส่วน
`_mm256_loadu_ps` แปลว่า "โหลด float packed 256-bit จาก memory แบบ unaligned" (`u` = unaligned,
ปลอดภัยกว่าเพราะไม่ต้องกังวลเรื่อง alignment ของ pointer แม้จะช้ากว่า aligned load เล็กน้อย
ในบางสถาปัตยกรรม)

---

## 88.6 ตัวอย่าง Intrinsics ที่ Compile และรันได้จริง (Step 702)

มาเขียนฟังก์ชันบวกอาร์เรย์ด้วย AVX2 intrinsics ด้วยมือกันจริงๆ:

```cpp
// intrinsics_demo.cpp - ตัวอย่างการเขียน SIMD ด้วยมือผ่าน AVX intrinsics
// คอมไพล์: g++ -Wall -Wextra -Wpedantic -std=c++17 -O2 -mavx2 intrinsics_demo.cpp -o intrinsics_demo
#include <cstdio>
#include <immintrin.h>  // header รวม intrinsics ของ SSE/AVX/AVX2/AVX-512 ทั้งหมด
#include <vector>

// บวกอาร์เรย์ float ทีละ 8 ตัวด้วย AVX2 (ยาว 256 bit = 8 x 32-bit float)
void add_avx2(const float* a, const float* b, float* out, size_t n) {
    size_t i = 0;
    // ส่วนหลัก: ประมวลผลทีละ 8 ตัวด้วยคำสั่ง SIMD เดียว
    for (; i + 8 <= n; i += 8) {
        __m256 va = _mm256_loadu_ps(a + i);   // โหลด float 8 ตัวจาก a เข้า register 256-bit
        __m256 vb = _mm256_loadu_ps(b + i);   // โหลด float 8 ตัวจาก b
        __m256 vc = _mm256_add_ps(va, vb);    // บวกทั้ง 8 คู่พร้อมกันในคำสั่งเดียว
        _mm256_storeu_ps(out + i, vc);        // เก็บผลลัพธ์กลับลง memory
    }
    // ส่วนที่เหลือ (remainder) ที่ไม่ครบ 8 ตัว ต้องทำแบบ scalar ปกติ
    for (; i < n; ++i) {
        out[i] = a[i] + b[i];
    }
}

int main() {
    const size_t n = 20; // จงใจใช้เลขที่ไม่ลงตัวกับ 8 เพื่อทดสอบ remainder loop ด้วย
    std::vector<float> a(n), b(n), out(n, 0.0f);
    for (size_t i = 0; i < n; ++i) {
        a[i] = static_cast<float>(i);
        b[i] = static_cast<float>(i) * 10.0f;
    }

    add_avx2(a.data(), b.data(), out.data(), n);

    std::printf("ผลลัพธ์การบวกด้วย AVX2 intrinsics (n=%zu):\n", n);
    for (size_t i = 0; i < n; ++i) {
        std::printf("out[%2zu] = %6.1f (คาดว่า %6.1f)\n", i, out[i], a[i] + b[i]);
    }
    return 0;
}
```

**ข้อสังเกตสำคัญของโค้ดนี้**: `n = 20` จงใจเลือกเป็นเลขที่ **ไม่ลงตัวกับ 8** (20 = 8 + 8 + 4)
เพื่อทดสอบว่าโค้ดจัดการ **remainder loop** ถูกต้องหรือไม่ — นี่คือรายละเอียดที่มือใหม่เขียน
intrinsics มักลืม (เขียนแต่ loop หลักที่ทำทีละ 8 ตัว แล้วลืมจัดการเศษที่เหลือ ทำให้ข้อมูล
ท้ายอาร์เรย์ไม่ถูกประมวลผลเลย)

คอมไพล์และรัน (**ต้องมี `-mavx2`** เพื่อบอก compiler ว่าอนุญาตให้ใช้คำสั่ง AVX2 ได้ ไม่เช่นนั้น
จะ compile error):

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 -O2 -mavx2 intrinsics_demo.cpp -o intrinsics_demo
./intrinsics_demo
```

**ผลลัพธ์ที่รันได้จริง (ถูกต้องครบทั้ง 20 ค่า รวมส่วน remainder ท้ายๆ ด้วย):**

```
ผลลัพธ์การบวกด้วย AVX2 intrinsics (n=20):
out[ 0] =    0.0 (คาดว่า    0.0)
out[ 1] =   11.0 (คาดว่า   11.0)
out[ 2] =   22.0 (คาดว่า   22.0)
...
out[15] =  165.0 (คาดว่า  165.0)
out[16] =  176.0 (คาดว่า  176.0)   <- เริ่มส่วน remainder (16, 17, 18, 19)
out[17] =  187.0 (คาดว่า  187.0)
out[18] =  198.0 (คาดว่า  198.0)
out[19] =  209.0 (คาดว่า  209.0)
```

ทุกค่าถูกต้องตรงกับที่คาดหวัง 100% ทั้งในส่วนที่ประมวลผลด้วย AVX2 (index 0-15) และส่วน
remainder ที่ทำแบบ scalar (index 16-19)

---

## 88.7 วัด Speedup จริง: Scalar vs Auto-Vectorize vs Intrinsics (Step 703)

มาเปรียบเทียบทั้ง 3 แนวทางในสถานการณ์เดียวกัน (kernel polynomial เดิมจาก 88.2) เพื่อดูว่า
**การเขียน intrinsics ด้วยมือคุ้มค่าแค่ไหนเมื่อเทียบกับปล่อยให้ compiler auto-vectorize ให้**:

```cpp
// poly_variants.cpp - เปรียบเทียบ 3 แนวทาง: scalar บังคับ, auto-vectorize โดย compiler,
// และ AVX2 intrinsics เขียนมือ สำหรับ kernel เดียวกัน (out[i] = a[i]^3 - 2a[i]^2 + 3a[i] + b[i])
//
// คอมไพล์ 3 แบบ (ดูรายละเอียดใน main text):
//   scalar:      g++ ... -O2 -mavx2 -fno-tree-vectorize -DVARIANT=0
//   autovec:     g++ ... -O3 -march=native               -DVARIANT=0
//   intrinsics:  g++ ... -O2 -mavx2                       -DVARIANT=1
#include <cstdio>
#include <chrono>
#include <vector>
#include <immintrin.h>

constexpr size_t N = 100000;
constexpr int REPEAT = 6000;

void poly_scalar(const float* __restrict a, const float* __restrict b,
                  float* __restrict out, size_t n) {
    for (size_t i = 0; i < n; ++i) {
        float x = a[i];
        out[i] = x * x * x - 2.0f * x * x + 3.0f * x + b[i];
    }
}

void poly_avx2(const float* __restrict a, const float* __restrict b,
               float* __restrict out, size_t n) {
    const __m256 two = _mm256_set1_ps(2.0f);
    const __m256 three = _mm256_set1_ps(3.0f);
    size_t i = 0;
    for (; i + 8 <= n; i += 8) {
        __m256 x = _mm256_loadu_ps(a + i);
        __m256 bv = _mm256_loadu_ps(b + i);
        __m256 x2 = _mm256_mul_ps(x, x);          // x^2
        __m256 x3 = _mm256_mul_ps(x2, x);         // x^3
        __m256 term2 = _mm256_mul_ps(two, x2);    // 2x^2
        __m256 term3 = _mm256_mul_ps(three, x);   // 3x
        __m256 r = _mm256_sub_ps(x3, term2);      // x^3 - 2x^2
        r = _mm256_add_ps(r, term3);              // + 3x
        r = _mm256_add_ps(r, bv);                 // + b
        _mm256_storeu_ps(out + i, r);
    }
    for (; i < n; ++i) {
        float x = a[i];
        out[i] = x * x * x - 2.0f * x * x + 3.0f * x + b[i];
    }
}

int main() {
    std::vector<float> a(N), b(N), out(N);
    for (size_t i = 0; i < N; ++i) {
        a[i] = static_cast<float>(i % 500) * 0.01f;
        b[i] = static_cast<float>(i % 300) * 0.02f;
    }

#if VARIANT == 1
    const char* name = "AVX2 intrinsics (มือเขียนเอง)";
#else
    const char* name = "Scalar / Auto-vectorize (แล้วแต่ flag ตอน compile)";
#endif

    auto t1 = std::chrono::steady_clock::now();
    for (int r = 0; r < REPEAT; ++r) {
#if VARIANT == 1
        poly_avx2(a.data(), b.data(), out.data(), N);
#else
        poly_scalar(a.data(), b.data(), out.data(), N);
#endif
    }
    auto t2 = std::chrono::steady_clock::now();

    double total_ms = std::chrono::duration<double, std::milli>(t2 - t1).count();
    std::printf("[%s]\n", name);
    std::printf("N = %zu, REPEAT = %d\n", N, REPEAT);
    std::printf("เวลารวม: %.3f ms, เฉลี่ยต่อรอบ: %.5f ms\n", total_ms, total_ms / REPEAT);
    std::printf("out[12345] = %f\n", out[12345]);
    return 0;
}
```

คอมไพล์ทั้ง 3 แบบ:

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 -O2 -mavx2 -fno-tree-vectorize -DVARIANT=0 \
    poly_variants.cpp -o poly_scalar_bin
g++ -Wall -Wextra -Wpedantic -std=c++17 -O3 -march=native -DVARIANT=0 \
    poly_variants.cpp -o poly_autovec_bin
g++ -Wall -Wextra -Wpedantic -std=c++17 -O2 -mavx2 -DVARIANT=1 \
    poly_variants.cpp -o poly_avx2_bin
```

(`-fno-tree-vectorize` ปิดการ vectorize อัตโนมัติทั้งหมด เพื่อบังคับให้เป็น pure scalar
สำหรับใช้เป็น baseline เทียบ)

**ผลลัพธ์ที่วัดได้จริง (N = 100,000, REPEAT = 6000, รันซ้ำ 2 ครั้งต่อแบบ):**

| แนวทาง | รอบ 1 (ms/รอบ) | รอบ 2 (ms/รอบ) | Speedup เทียบ Scalar (เฉลี่ย) |
|---|---:|---:|---:|
| Scalar (`-fno-tree-vectorize`) | 0.09939 | 0.10611 | 1.00x |
| Auto-vectorize (`-O3 -march=native`) | 0.01905 | 0.02103 | **~5.13x** |
| AVX2 Intrinsics (มือเขียน) | 0.02304 | 0.01817 | **~4.99x** |

ผลลัพธ์ที่ได้จริงชี้ประเด็นสำคัญที่สุดของ Part นี้: **ในสถานการณ์นี้ การเขียน intrinsics ด้วยมือ
ให้ผลลัพธ์ไม่ต่างจาก auto-vectorize เลย** (ทั้งคู่ให้ speedup ประมาณ 5 เท่าเทียบกับ scalar
ใกล้เคียงกันมาก บางรอบ auto-vectorize ยังเร็วกว่า intrinsics ด้วยซ้ำ) ทั้งที่เราเสียเวลาเขียน
intrinsics มากกว่าหลายเท่า ต้องคิดเรื่อง register manual, จัดการ remainder loop เอง, และเสี่ยง
เขียนผิดพลาดได้ง่ายกว่ามาก

**ข้อสรุปเชิงปฏิบัติ**: สำหรับ kernel เรียบง่ายแบบนี้ (arithmetic ตรงไปตรงมา ไม่มี branch,
ไม่มี dependency) **ให้เชื่อใจ auto-vectorization ของ compiler ก่อนเสมอ** และตรวจสอบด้วย
`-fopt-info-vec` ว่ามัน vectorize สำเร็จจริง การเขียน intrinsics ด้วยมือควรสงวนไว้สำหรับกรณีที่
auto-vectorization ล้มเหลวจริงๆ (ตรวจสอบแล้วเจอ "missed" message) หรือ algorithm ที่ compiler
ไม่มีทางเดาลาย pattern ได้เอง (เช่น การใช้คำสั่งพิเศษเฉพาะทาง อย่าง shuffle/permute ข้าม lane,
horizontal reduction ที่ซับซ้อน, หรือ bit manipulation ระดับต่ำ)

---

## 88.8 เมื่อไหร่ควรใช้ Auto-Vectorization พอ เมื่อไหร่ต้องเขียน Intrinsics เอง (Step 704)

จากทุกการทดลองในบทนี้ สรุปเป็นแนวทางตัดสินใจได้ดังนี้:

| สถานการณ์ | แนวทางที่แนะนำ |
|---|---|
| Loop เรียบง่าย ไม่มี dependency, ไม่มี aliasing | ปล่อยให้ compiler auto-vectorize + ตรวจสอบด้วย `-fopt-info-vec` |
| ตรวจสอบแล้วพบว่า compiler ไม่ vectorize (missed) | ลองปรับโค้ด (`__restrict`, แยกฟังก์ชัน, ลด branch) ก่อน แล้วค่อยตรวจสอบใหม่ |
| ปรับโค้ดแล้วยังไม่ vectorize เพราะ algorithm ซับซ้อนเกิน heuristic ของ compiler | พิจารณาเขียน intrinsics ด้วยมือ |
| ต้องการควบคุมการใช้ CPU feature เฉพาะรุ่น (เช่น AVX-512 VNNI สำหรับ ML inference) | เขียน intrinsics โดยตรง หรือใช้ library ที่ทำให้แล้ว (เช่น Google Highway, xsimd) |
| Portability ข้าม CPU หลายสถาปัตยกรรม (x86, ARM) สำคัญกว่า performance สูงสุด | หลีกเลี่ยง intrinsics เฉพาะ x86 ใช้ library แบบ portable (`std::experimental::simd`, xsimd) แทน |

### ข้อจำกัดสำคัญของการเขียน Intrinsics ด้วยมือ

1. **Portability ต่ำ**: โค้ดที่เขียนด้วย `<immintrin.h>` ผูกติดกับสถาปัตยกรรม x86/x86_64
   เท่านั้น รันบน ARM (เช่น Apple Silicon, มือถือ) ไม่ได้เลย ต้องเขียนเวอร์ชัน NEON แยกต่างหาก
   ถ้าต้องการ portability ข้าม platform
2. **อ่านยาก บำรุงรักษายาก**: เทียบกับ loop ธรรมดา 3 บรรทัด โค้ด intrinsics ที่ทำงานเหมือนกัน
   อาจยาวกว่าหลายเท่าและเข้าใจยากกว่ามากสำหรับคนที่ไม่คุ้นเคย
3. **ต้องระวังการเปลี่ยนแปลงของ Floating-Point Rounding**: การจัดกลุ่มการบวกใหม่ (เช่นใน
   horizontal reduction ของ dot product) อาจให้ผลลัพธ์ต่างจาก scalar เล็กน้อย เพราะการบวกเลข
   ทศนิยมไม่เป็นไปตาม associative law อย่างเคร่งครัด (`(a+b)+c` อาจไม่เท่ากับ `a+(b+c)` เป๊ะ
   เพราะการปัดเศษ) รายละเอียดนี้จะกล่าวถึงอีกครั้งในแบบฝึกหัดท้ายบท
4. **ต้องคอมไพล์แยกตาม target CPU feature**: ถ้า deploy ไปยังเครื่องที่ไม่รองรับ AVX2 (CPU
   เก่า) โปรแกรมที่คอมไพล์ด้วย `-mavx2` จะรันไม่ได้เลย (ได้ `Illegal instruction`) ต้องมีกลไก
   ตรวจสอบ CPU feature ที่ runtime (เช่น `__builtin_cpu_supports("avx2")`) และมีโค้ด fallback
   สำหรับ CPU ที่ไม่รองรับ ถ้าต้องการแจกจ่ายโปรแกรมให้ทำงานได้กว้างขวาง

บทถัดไป (**Part 89 — Compiler Optimization และ Assembly**) จะพาไปดูภาพรวมของ optimization
flag อื่นๆ ที่ compiler ทำให้นอกเหนือจาก vectorization ทั้งหมด ซึ่งจะช่วยให้เข้าใจภาพรวมของ
"อะไรที่ควรปล่อยให้ compiler ทำ" กับ "อะไรที่ต้องลงมือทำเองบ้าง" ได้สมบูรณ์ยิ่งขึ้น

---

## ข้อผิดพลาดที่พบบ่อย (Common Pitfalls)

1. **เชื่อว่า Compiler Vectorize ให้แล้วโดยไม่ตรวจสอบ**: มือใหม่จำนวนมากเปิด `-O3
   -march=native` แล้วสันนิษฐานว่า loop ของตัวเอง "ต้อง" ถูก vectorize แน่นอน ทั้งที่จริงแล้ว
   compiler อาจ "missed" (พลาด) ไปเงียบๆ โดยไม่มี error ใดๆ เลย (โปรแกรมยังคอมไพล์ผ่านและรันได้
   ถูกต้อง แค่ช้ากว่าที่ควรจะเป็น) ต้องตรวจสอบด้วย `-fopt-info-vec-missed` หรือ `objdump` เสมอ
   เมื่อ performance เป็นเรื่องสำคัญ

2. **สับสนระหว่าง "-O3 vectorize" กับ "-O2 ไม่ vectorize"**: อย่างที่พิสูจน์ในหัวข้อ 88.3
   GCC สมัยใหม่ (12 ขึ้นไป) เปิด `-ftree-loop-vectorize` ตั้งแต่ `-O2` แล้ว ความเชื่อเก่าที่ว่า
   "ต้อง -O3 เท่านั้นถึงจะ vectorize" ใช้ไม่ได้กับ compiler รุ่นใหม่อีกต่อไป ต้องตรวจสอบ
   เวอร์ชัน compiler ที่ใช้จริงเสมอแทนที่จะเชื่อความรู้เก่า

3. **ลืมว่า SIMD ช่วยได้แค่กับงานที่เป็น Compute-Bound**: อย่างที่วัดได้จริงในหัวข้อ 88.2 ถ้า
   งาน memory-bound (ข้อมูลใหญ่กว่า cache มาก, bottleneck อยู่ที่แบนด์วิดท์ RAM) การเปลี่ยน
   จาก SSE เป็น AVX-512 แทบไม่ช่วยอะไรเลย ต้องแก้ปัญหาการเข้าถึงหน่วยความจำก่อน (ตามที่เรียน
   ใน Part 87) ไม่ใช่หวังพึ่ง SIMD อย่างเดียว

4. **ลืมจัดการ Remainder Loop เมื่อเขียน Intrinsics เอง**: ถ้าข้อมูลมีจำนวนไม่ลงตัวกับความกว้าง
   ของ SIMD register (เช่น N = 100 ไม่ลงตัวกับ 8 สำหรับ AVX2) ต้องมี loop แยกจัดการส่วนที่เหลือ
   (remainder) เสมอ มือใหม่มักลืมส่วนนี้ ทำให้ข้อมูลท้ายอาร์เรย์ไม่ถูกประมวลผลเลย (bug เงียบ
   ที่ตรวจจับยากถ้า test case ใช้ N ที่ลงตัวพอดีเสมอ)

5. **ใช้ `-march=native` แล้ว Deploy ไปเครื่องอื่นโดยไม่ระวัง**: `-march=native` compile
   โปรแกรมให้ใช้ CPU feature ทั้งหมดของเครื่องที่ compile อยู่ ถ้า deploy binary นั้นไปรันบน
   เครื่องอื่นที่ CPU รุ่นเก่ากว่า (ไม่มี AVX-512 หรือแม้แต่ AVX2) โปรแกรมจะ crash ด้วย
   `Illegal instruction` ทันที ควรใช้ `-march=x86-64-v2/v3` (กำหนด baseline ชัดเจน) หรือ
   ตรวจสอบ CPU feature runtime แทนสำหรับโปรแกรมที่ต้อง distribute ไปหลายเครื่อง

6. **ไม่ตระหนักว่า Floating-Point Reordering เปลี่ยนผลลัพธ์ได้**: การ vectorize การคำนวณ
   floating-point (โดยเฉพาะ reduction เช่น การหาผลรวม/dot product) เปลี่ยนลำดับการบวกจาก
   ที่เขียนใน source code ซึ่งอาจทำให้ผลลัพธ์ต่างจาก scalar เล็กน้อย (ในระดับ rounding error)
   สำหรับงานที่ต้องการผลลัพธ์ที่ **reproducible เป๊ะๆ ทุก byte** (เช่น การคำนวณทางการเงินบาง
   ประเภท, การทดสอบ regression ที่เทียบผลลัพธ์แบบ exact match) ต้องระวังเรื่องนี้เป็นพิเศษ
   และอาจต้องปิด reordering ด้วย `-ffp-contract=off` หรือหลีกเลี่ยง `-ffast-math`

7. **เขียน Intrinsics ทั้งที่ Auto-Vectorization ทำได้ดีอยู่แล้ว**: ดังที่วัดได้จริงในหัวข้อ 88.7
   สำหรับ kernel เรียบง่าย การเขียน intrinsics ด้วยมือแทบไม่ได้ speedup เพิ่มเติมเลยเมื่อเทียบกับ
   auto-vectorize แต่เสียเวลาพัฒนาและความสามารถในการอ่าน/บำรุงรักษาไปมาก ควรลอง auto-vectorize
   ก่อนเสมอ แล้วเขียน intrinsics เฉพาะกรณีที่พิสูจน์แล้วว่าจำเป็นจริงๆ

---

## แบบฝึกหัดท้ายบท

1. รันคำสั่ง `g++ -O2 -Q --help=optimizers | grep -i vectorize` และ
   `g++ -O3 -Q --help=optimizers | grep -i vectorize` บนเครื่องของตัวเอง เปรียบเทียบผลกับที่
   แสดงในบทเรียน (GCC 13.3.0) ถ้าใช้ compiler เวอร์ชันอื่นหรือ Clang ผลต่างกันหรือไม่ อย่างไร

2. เขียนฟังก์ชัน **dot product** (ผลรวมของ `a[i] * b[i]` ทุกตัว) ด้วย AVX2 intrinsics เปรียบเทียบ
   ความเร็วกับเวอร์ชัน scalar ธรรมดา และตรวจสอบว่าผลลัพธ์ทั้งสองแบบตรงกันเป๊ะหรือมีความคลาดเคลื่อน
   เล็กน้อย อธิบายสาเหตุถ้ามีความคลาดเคลื่อน

3. ลบ `__restrict` ออกจาก `add_arrays` ใน `vecadd.cpp` (ทั้ง 3 parameter) แล้ววัดเวลาเทียบกับ
   เวอร์ชันเดิม ตรวจสอบด้วย `-fopt-info-vec-all` ว่า compiler ยัง vectorize ได้หรือไม่ ถ้าได้
   ทำผ่านเทคนิคอะไร

4. คอมไพล์ `polybench.cpp` (จากหัวข้อ 88.2) ด้วย `-msse4.2`, `-mavx2`, และ `-mavx512f` ตามลำดับ
   (ทั้งหมดใช้ `-O3`) วัดเวลาที่ N = 100,000 แล้วเปรียบเทียบว่าความกว้างของ SIMD register
   (128/256/512-bit) ส่งผลต่อความเร็วมากน้อยแค่ไหนในทางปฏิบัติจริง

5. เขียนฟังก์ชันที่มี branch ภายใน loop เช่น `out[i] = (a[i] > 0) ? a[i] : -a[i];` (absolute
   value แบบเขียนเอง) ตรวจสอบด้วย `-fopt-info-vec` ว่า compiler ยัง vectorize ได้หรือไม่
   (คำใบ้: ลองเทียบกับการใช้ `std::abs` แทน if/else ดูว่าผลต่างกันหรือไม่)

6. อธิบายด้วยคำพูดตัวเอง (ไม่ต้องเขียนโค้ด): ทำไมโปรแกรมที่ compile ด้วย `-march=native` บน
   เครื่อง A แล้วนำไป deploy บนเครื่อง B ที่ CPU รุ่นเก่ากว่า ถึงอาจ crash ทันทีที่รัน และมีวิธี
   ป้องกันปัญหานี้อย่างไรบ้างในทางปฏิบัติ (สำหรับซอฟต์แวร์ที่ต้อง distribute ไปหลายเครื่อง)

### แนวทางเฉลยข้อ 2

```cpp
// dotproduct.cpp - เฉลยแบบฝึกหัด: dot product ด้วย AVX2 intrinsics
// คอมไพล์: g++ -Wall -Wextra -Wpedantic -std=c++17 -O2 -mavx2 dotproduct.cpp -o dotproduct
#include <cstdio>
#include <immintrin.h>
#include <vector>
#include <chrono>

float dot_scalar(const float* a, const float* b, size_t n) {
    float sum = 0.0f;
    for (size_t i = 0; i < n; ++i) sum += a[i] * b[i];
    return sum;
}

float dot_avx2(const float* a, const float* b, size_t n) {
    __m256 acc = _mm256_setzero_ps();        // vector สะสมผลรวม 8 ช่องพร้อมกัน
    size_t i = 0;
    for (; i + 8 <= n; i += 8) {
        __m256 va = _mm256_loadu_ps(a + i);
        __m256 vb = _mm256_loadu_ps(b + i);
        acc = _mm256_add_ps(acc, _mm256_mul_ps(va, vb)); // acc += a[i..i+8] * b[i..i+8]
    }
    // ลดรูป (horizontal reduce) 8 ช่องใน acc ให้เหลือค่าเดียว
    alignas(32) float tmp[8];
    _mm256_store_ps(tmp, acc);
    float sum = tmp[0] + tmp[1] + tmp[2] + tmp[3] + tmp[4] + tmp[5] + tmp[6] + tmp[7];
    // จัดการส่วนที่เหลือ (remainder) แบบ scalar
    for (; i < n; ++i) sum += a[i] * b[i];
    return sum;
}

int main() {
    constexpr size_t N = 10'000'000;
    std::vector<float> a(N), b(N);
    for (size_t i = 0; i < N; ++i) {
        a[i] = static_cast<float>(i % 13) * 0.1f;
        b[i] = static_cast<float>(i % 7) * 0.2f;
    }

    constexpr int REPEAT = 30;

    auto t1 = std::chrono::steady_clock::now();
    float r1 = 0.0f;
    for (int r = 0; r < REPEAT; ++r) r1 = dot_scalar(a.data(), b.data(), N);
    auto t2 = std::chrono::steady_clock::now();

    float r2 = 0.0f;
    for (int r = 0; r < REPEAT; ++r) r2 = dot_avx2(a.data(), b.data(), N);
    auto t3 = std::chrono::steady_clock::now();

    double scalar_ms = std::chrono::duration<double, std::milli>(t2 - t1).count() / REPEAT;
    double avx2_ms = std::chrono::duration<double, std::milli>(t3 - t2).count() / REPEAT;

    std::printf("scalar: %.4f ms/รอบ, ผลลัพธ์ = %f\n", scalar_ms, r1);
    std::printf("avx2:   %.4f ms/รอบ, ผลลัพธ์ = %f\n", avx2_ms, r2);
    std::printf("ผลต่างสัมพัทธ์ของผลลัพธ์: %f\n", static_cast<double>(r1) - static_cast<double>(r2));
    std::printf("speedup: %.2fx\n", scalar_ms / avx2_ms);
    return 0;
}
```

คอมไพล์และรันจริง:

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 -O2 -mavx2 dotproduct.cpp -o dotproduct
./dotproduct
```

**ผลลัพธ์ที่วัดได้จริง (N = 10,000,000, REPEAT = 30 รอบ, รันซ้ำ 2 ครั้ง):**

| รอบที่ | scalar (ms/รอบ) | avx2 (ms/รอบ) | speedup | ผลลัพธ์ scalar | ผลลัพธ์ avx2 |
|---|---:|---:|---:|---:|---:|
| 1 | 9.5607 | 5.2399 | 1.82x | 3592717.50 | 3599763.00 |
| 2 | 9.8767 | 5.0681 | 1.95x | 3592717.50 | 3599763.00 |

**คำอธิบาย 2 ประเด็นสำคัญ**:

1. **Speedup ~1.8-2 เท่า ไม่ใช่ 8 เท่าตามทฤษฎี**: เพราะข้อมูล N = 10 ล้าน floats กินพื้นที่
   40 MB ต่ออาร์เรย์ (รวม 80 MB ทั้งสองอาร์เรย์) ใหญ่เกิน L2 cache มาก งานนี้จึงเป็น
   memory-bound บางส่วน สอดคล้องกับที่เรียนในหัวข้อ 88.2

2. **ผลลัพธ์ scalar กับ avx2 ไม่ตรงกันเป๊ะ** (3592717.50 vs 3599763.00 ต่างกันประมาณ 0.2%)
   — นี่ไม่ใช่ bug! สาเหตุคือ **การบวกเลขทศนิยม (floating-point) ไม่เป็นไปตาม associative
   law อย่างเคร่งครัด** เวอร์ชัน scalar บวกสะสมทีละตัวเรียงลำดับ (`sum = ((...((0+p0)+p1)+p2...)`)
   ในขณะที่เวอร์ชัน AVX2 บวกสะสมแยกเป็น 8 "ราง" ขนานกัน (`acc[0]` สะสมจาก `p0, p8, p16, ...`,
   `acc[1]` สะสมจาก `p1, p9, p17, ...` เป็นต้น) แล้วค่อยรวม 8 รางเข้าด้วยกันตอนท้าย ลำดับการ
   บวกที่ต่างกันทำให้ค่า rounding error สะสมต่างกันเล็กน้อย นี่คือเหตุผลที่ในหัวข้อ 88.8 เตือน
   ว่างานที่ต้องการผลลัพธ์ floating-point แบบ **reproducible เป๊ะทุก byte** ต้องระวังเรื่องการ
   vectorize เป็นพิเศษ

### แนวทางเฉลยข้อ 4

```cpp
// width_compare.cpp - เฉลยแบบฝึกหัด: เปรียบเทียบผลของความกว้าง SIMD register (SSE/AVX2/AVX-512)
// โดยปล่อยให้ compiler auto-vectorize เอง (ไม่เขียน intrinsics มือ) แค่เปลี่ยน -march
#include <cstdio>
#include <chrono>
#include <vector>

constexpr size_t N = 100000;
constexpr int REPEAT = 6000;

void poly(const float* __restrict a, const float* __restrict b,
          float* __restrict out, size_t n) {
    for (size_t i = 0; i < n; ++i) {
        float x = a[i];
        out[i] = x * x * x - 2.0f * x * x + 3.0f * x + b[i];
    }
}

int main() {
    std::vector<float> a(N), b(N), out(N);
    for (size_t i = 0; i < N; ++i) {
        a[i] = static_cast<float>(i % 500) * 0.01f;
        b[i] = static_cast<float>(i % 300) * 0.02f;
    }

    auto t1 = std::chrono::steady_clock::now();
    for (int r = 0; r < REPEAT; ++r) poly(a.data(), b.data(), out.data(), N);
    auto t2 = std::chrono::steady_clock::now();

    double total_ms = std::chrono::duration<double, std::milli>(t2 - t1).count();
    std::printf("เฉลี่ยต่อรอบ: %.5f ms  (out[0]=%f)\n", total_ms / REPEAT, out[0]);
    return 0;
}
```

คอมไพล์ทั้ง 3 แบบ (ทั้งหมดใช้ `-O3` แต่เปลี่ยนแค่ target CPU feature):

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 -O3 -msse4.2   width_compare.cpp -o w_sse
g++ -Wall -Wextra -Wpedantic -std=c++17 -O3 -mavx2     width_compare.cpp -o w_avx2
g++ -Wall -Wextra -Wpedantic -std=c++17 -O3 -mavx512f  width_compare.cpp -o w_avx512
```

**ผลลัพธ์ที่วัดได้จริง (รันซ้ำ 2 ครั้งต่อแบบ):**

| ชุดคำสั่ง | ความกว้าง register | รอบ 1 (ms/รอบ) | รอบ 2 (ms/รอบ) |
|---|---|---:|---:|
| SSE4.2 | 128-bit (4 floats) | 0.02503 | 0.02492 |
| AVX2 | 256-bit (8 floats) | 0.01927 | 0.02242 |
| AVX-512 | 512-bit (16 floats) | 0.01563 | 0.01788 |

**คำอธิบาย**: ผลลัพธ์แสดงแนวโน้มตามทฤษฎีชัดเจน — ยิ่ง register กว้างขึ้น ยิ่งเร็วขึ้น
(SSE → AVX2 → AVX-512 เร็วขึ้นตามลำดับทุกครั้งที่ทดสอบ) แต่ **อัตราการเพิ่มขึ้นไม่เป็นเชิงเส้น
ตามความกว้าง register** (ถ้าเป็นเชิงเส้นเป๊ะ AVX-512 ควรเร็วกว่า SSE ถึง 4 เท่า แต่วัดได้จริง
แค่ประมาณ 1.4-1.6 เท่า) เพราะยังมี overhead อื่นที่ไม่ได้ลดลงตามความกว้าง register เช่น
การเข้าถึงหน่วยความจำ (แม้ข้อมูลจะพอดี cache แต่ก็ยังต้องผ่าน load/store unit), loop control
overhead, และการที่ CPU บางรุ่นลด clock speed ลงเมื่อใช้คำสั่ง AVX-512 หนักๆ ต่อเนื่อง
(ปรากฏการณ์ที่เรียกว่า **AVX-512 frequency throttling** ซึ่งเป็นที่รู้จักกันดีใน Intel CPU
บางรุ่น) **บทเรียนสำคัญ**: การเลือกใช้ SIMD ที่กว้างที่สุดเท่าที่ CPU รองรับ ไม่ได้แปลว่าจะได้
ความเร็วเพิ่มขึ้นตามสัดส่วนเสมอไป ต้องวัดผลจริงในแต่ละสถานการณ์

---

## สรุปท้ายบท

ใน Part นี้เราได้:

- เข้าใจ **SIMD (Single Instruction, Multiple Data)** และตระกูลชุดคำสั่งบน x86_64 (SSE, AVX,
  AVX2, AVX-512) ที่กว้างขึ้นเรื่อยๆ ตามยุคสมัย
- เห็น **Assembly จริง** ที่ต่างกันระหว่าง `-O0` (คำสั่ง scalar `addss`) กับ `-O3 -march=native`
  (คำสั่ง vectorized `vaddps` บน `ymm` register) ด้วย `objdump`
- **พิสูจน์ด้วยการวัดจริง** ว่า Auto-vectorization ช่วยได้มากในงาน compute-bound (~6.4 เท่าใน
  บาง case) แต่แทบไม่ช่วยเลยในงาน memory-bound (~1.9 เท่าเท่านั้น ไม่ว่าจะใช้ SIMD กว้างแค่ไหน)
- รู้ความจริงสำคัญที่ตำราเก่าไม่ทันอัปเดต: **GCC 12 ขึ้นไปเปิด auto-vectorization ตั้งแต่ `-O2`
  แล้ว** ไม่ใช่แค่ `-O3` เหมือนที่เข้าใจกันมาก่อน — ตรวจสอบด้วย `-fopt-info-vec` เสมอแทนการเดา
- เข้าใจว่า **Pointer Aliasing** และ **Loop-Carried Dependency** เป็น 2 อุปสรรคหลักที่ทำให้
  compiler vectorize ไม่ได้ และรู้วิธีแก้ด้วย `__restrict`
- เขียน **SIMD Intrinsics** ด้วยมือผ่าน `<immintrin.h>` ได้จริง คอมไพล์และรันถูกต้อง 100%
  รวมถึงจัดการ remainder loop
- **วัด speedup จริง** เปรียบเทียบ scalar, auto-vectorize, และ intrinsics แล้วพบว่าสำหรับ
  kernel เรียบง่าย การเขียน intrinsics ด้วยมือแทบไม่ได้ประโยชน์เพิ่มเติมเหนือ auto-vectorize
  เลย (ทั้งคู่ได้ speedup ~5 เท่าใกล้เคียงกัน) ซึ่งเป็นบทเรียนสำคัญเรื่องการเลือกใช้เวลาพัฒนา
  ให้คุ้มค่า

รวมกับสิ่งที่เรียนใน **Part 87** เรื่อง Cache-Friendly Code และ Data-Oriented Design ตอนนี้
เรามีเครื่องมือ 2 ชิ้นสำคัญสำหรับ performance engineering: **จัดข้อมูลให้ cache-friendly**
(ลด latency ของการเข้าถึงหน่วยความจำ) และ **ใช้ SIMD ให้เต็มที่** (เพิ่ม throughput ของการ
คำนวณ) ทั้งสองเทคนิคนี้ทำงานเสริมกันได้ดีมาก — ข้อมูลที่จัดแบบ SoA (จาก Part 87) มักจะ
vectorize ได้ง่ายกว่า AoS พอดี เพราะข้อมูลชนิดเดียวกันเรียงติดกันเป็นแถวยาวที่ SIMD register
โหลดได้ทีเดียวหลายค่า

ใน **Part 89** เราจะขยายมุมมองจาก vectorization ไปสู่ภาพรวมทั้งหมดของ **Compiler
Optimization** — ทำความเข้าใจว่า `-O1`, `-O2`, `-O3` แต่ละระดับทำอะไรบ้าง (inlining, loop
unrolling, dead code elimination, ฯลฯ) และฝึกอ่าน Assembly Output เพื่อเข้าใจว่า compiler
"คิด" อย่างไรกับโค้ดของเรา ซึ่งจะทำให้เราเขียนโค้ดที่ compiler-friendly ได้ดียิ่งขึ้นไปอีก

**ต่อไป:** [Part 89 — Compiler Optimization และ Assembly](./part-089-compiler-optimizations.md)
