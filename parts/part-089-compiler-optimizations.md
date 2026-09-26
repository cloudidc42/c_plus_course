# Part 89: Compiler Optimization และ Assembly (Step 705–712)

> Module G — Concurrency และ Performance Engineering | Part 89 จาก 125
> Step ที่ครอบคลุมใน Part นี้: Step 705–712
> Part ก่อนหน้า: [Part 88 — SIMD และ Vectorization](./part-088-simd-vectorization.md) | Part ถัดไป: [Part 90 — Benchmarking ด้วย Google Benchmark](./part-090-google-benchmark.md)

## เป้าหมายการเรียนรู้ (Learning Objectives)

หลังจบ Part นี้ ผู้เรียนจะสามารถ:

1. อธิบายความหมายและ trade-off ของแต่ละ optimization level ของ GCC (`-O0` ถึง `-O3`, `-Os`, `-Ofast`) ได้อย่างถูกต้อง พร้อมเลือกใช้ให้เหมาะกับสถานการณ์จริง
2. อ่านและเปรียบเทียบ Assembly ที่ compiler สร้าง (ผ่าน `gcc -S`) ก่อนและหลังเปิด optimization เพื่อเห็นว่า Dead Code Elimination, Constant Folding, Loop Unrolling และ Inlining เกิดขึ้นจริงอย่างไร
3. เข้าใจว่า keyword `inline` เป็นเพียง "คำขอ" ไม่ใช่ "คำสั่งบังคับ" และ compiler มีอิสระเต็มที่ในการตัดสินใจว่าจะ inline ฟังก์ชันใดหรือไม่
4. อธิบายได้ว่า Undefined Behavior (UB) เป็นอันตรายต่อโปรแกรมที่ optimize แล้วอย่างไร โดยเห็นตัวอย่างจริงที่โปรแกรมเปลี่ยนพฤติกรรมโดยสิ้นเชิงเมื่อเปิด optimization
5. ใช้ `-march=native` เพื่อให้ compiler สร้างโค้ดที่ใช้ชุดคำสั่ง (Instruction Set) เต็มประสิทธิภาพของ CPU เครื่องที่คอมไพล์ พร้อมเข้าใจความเสี่ยงเรื่อง portability
6. ใช้ `-flto` (Link Time Optimization) เพื่อให้ compiler มองเห็นและ optimize ข้ามไฟล์ `.cpp` ได้ พร้อมพิสูจน์ผลด้วย Assembly จริง
7. เลือกชุด flag ที่เหมาะสมสำหรับขั้นตอนพัฒนา (debug build) และขั้นตอนส่งมอบจริง (release build) ได้อย่างมีเหตุผล

---

## 89.1 Optimization Level ของ GCC: -O0 ถึง -Ofast (Step 705)

ตลอดหลักสูตรนี้เราคอมไพล์โค้ดด้วย `-O0` มาโดยตลอด (ปิดการ optimize) เพราะสะดวกต่อการ debug
ด้วย GDB (ตัวแปรไม่ถูกยุบรวมหรือลบทิ้ง โค้ดตรงกับ source เป๊ะๆ) แต่ในโลกจริง โปรแกรมที่ส่งมอบ
ให้ผู้ใช้ (release build) แทบไม่มีใครปล่อยด้วย `-O0` เลย เพราะช้ากว่า optimized build มาก

GCC มี optimization level หลักๆ 6 ระดับ:

| Flag | ความหมาย | เวลาคอมไพล์ | ขนาดไบนารี | เหมาะกับ |
|---|---|---|---|---|
| `-O0` | ปิด optimize ทั้งหมด (ค่า default) | เร็วที่สุด | ใหญ่ที่สุด/ใกล้เคียง source | Debug ด้วย GDB |
| `-O1` | optimize พื้นฐาน ไม่เสียเวลาคอมไพล์มาก | เร็ว | เล็กลง | จุดกึ่งกลางเวลา build บ่อยๆ |
| `-O2` | optimize เกือบทั้งหมดที่ไม่แลกกับขนาดไบนารี | ปานกลาง | ปานกลาง | **Release build มาตรฐาน** |
| `-O3` | เพิ่ม aggressive loop optimization/vectorization จาก `-O2` | ช้าลง | ใหญ่ขึ้น (จาก unroll/inline) | โค้ดที่เน้นความเร็วเชิงตัวเลข/loop หนักๆ |
| `-Os` | เหมือน `-O2` แต่ตัด optimization ที่ทำให้ไบนารีใหญ่ขึ้นออก | ปานกลาง | **เล็กที่สุด** | Embedded/พื้นที่เก็บจำกัด |
| `-Ofast` | `-O3` + ผ่อนคลายมาตรฐาน IEEE 754 (`-ffast-math` และอื่นๆ) | ช้าลง | ใหญ่ขึ้น | งานตัวเลขที่ยอมรับความคลาดเคลื่อนได้ |

> **ข้อควรระวังสำคัญ**: `-O3` **ไม่ได้แปลว่าเร็วกว่า `-O2` เสมอไป** บางครั้ง aggressive
> loop unrolling ของ `-O3` ทำให้โค้ดใหญ่เกิน Instruction Cache (I-Cache) จนช้าลงกว่าเดิมก็มี
> การเลือก optimization level ที่ถูกต้องต้อง **วัดจริง** ด้วย benchmark (เรื่องที่จะเรียนใน Part 90)
> ไม่ใช่เดาจากตัวเลขที่สูงกว่า

### ทดลองวัดผลจริงบนเครื่อง

สร้างไฟล์ `sum_levels.cpp`:

```cpp
#include <cstdio>
#include <cstdlib>
#include <chrono>

long sum_to_n(long n) {
    long total = 0;
    for (long i = 1; i <= n; ++i) {
        total += i;
    }
    return total;
}

int main(int argc, char** argv) {
    // n มาจาก argv (runtime) ไม่ใช่ compile-time constant
    // เพื่อป้องกันไม่ให้ compiler "คำนวณคำตอบล่วงหน้า" ทิ้งลูปทั้งหมด (จะพูดถึงใน 89.2)
    long n = (argc > 1) ? atol(argv[1]) : 1000000000L;

    auto t0 = std::chrono::steady_clock::now();
    long result = sum_to_n(n);
    auto t1 = std::chrono::steady_clock::now();

    double ms = std::chrono::duration<double, std::milli>(t1 - t0).count();
    printf("n=%ld result=%ld time=%.3f ms\n", n, result, ms);
    return 0;
}
```

คอมไพล์ทีละ level แล้ววัดผลจริง (รันบนเครื่องที่ใช้เขียนบทเรียนนี้ CPU Intel Xeon 4 core @2.1GHz):

```bash
for level in O0 O1 O2 O3 Os Ofast; do
    g++ -${level} -Wall -Wextra -Wpedantic -std=c++17 sum_levels.cpp -o sum_${level}
done
for level in O0 O1 O2 O3 Os Ofast; do
    echo "-${level}:"; ./sum_${level} 1000000000
done
```

ผลลัพธ์ที่วัดได้จริง:

```
-O0:
n=1000000000 result=500000000500000000 time=2669.248 ms
-O1:
n=1000000000 result=500000000500000000 time=354.961 ms
-O2:
n=1000000000 result=500000000500000000 time=353.129 ms
-O3:
n=1000000000 result=500000000500000000 time=359.362 ms
-Os:
n=1000000000 result=500000000500000000 time=358.041 ms
-Ofast:
n=1000000000 result=500000000500000000 time=352.218 ms
```

สังเกตว่า **`-O0` ช้ากว่า `-O1` ถึงเกือบ 7.5 เท่า** เพราะที่ `-O0` ตัวแปรทุกตัวถูกเก็บบนหน่วยความจำ
Stack จริง (ไม่ได้อยู่ใน Register) และไม่มีการยุบคำสั่งใดๆ เลย ส่วน `-O1` ถึง `-Ofast` ให้เวลาที่
ใกล้เคียงกันมากในตัวอย่างนี้ เพราะลูปนี้เรียบง่าย (loop-carried dependency แบบต่อเนื่อง) จน `-O1`
ก็จัดการเก็บ `total`/`i` ไว้ใน Register ได้เต็มประสิทธิภาพแล้ว — เป็นตัวอย่างที่ดีว่าประโยชน์ของ
`-O2`/`-O3` เหนือ `-O1` ไม่ได้มีในทุกกรณี ขึ้นกับลักษณะของโค้ด

### เมื่อ -Ofast เปลี่ยน "คำตอบ" ไม่ใช่แค่ความเร็ว

ตัวอย่างข้างบนเป็นเลขจำนวนเต็ม (`long`) ซึ่งการบวกเป็น associative จริงตามคณิตศาสตร์
แต่กับเลขทศนิยม (`float`/`double`) เรื่องนี้ซับซ้อนกว่า ลองดูตัวอย่างนี้:

```cpp
#include <cstdio>
#include <cstdlib>
#include <chrono>
#include <vector>

float array_sum(const float* data, int n) {
    float total = 0.0f;
    for (int i = 0; i < n; ++i) {
        total += data[i];
    }
    return total;
}

int main(int argc, char** argv) {
    int n = (argc > 1) ? atoi(argv[1]) : 100000000;
    std::vector<float> data(n, 1.5f);

    auto t0 = std::chrono::steady_clock::now();
    float result = array_sum(data.data(), n);
    auto t1 = std::chrono::steady_clock::now();
    double ms = std::chrono::duration<double, std::milli>(t1 - t0).count();
    printf("n=%d result=%f time=%.3f ms\n", n, result, ms);
    return 0;
}
```

รันด้วย `n = 100000000` ธาตุ ค่าละ `1.5f` (ผลรวมทางคณิตศาสตร์ที่ถูกต้องคือ 150,000,000):

```
-O2:    result=33554432.000000   time=85.339 ms
-O3:    result=33554432.000000   time=86.435 ms
-Ofast: result=134217728.000000  time=44.739 ms
```

ทั้งสามค่าผิดจากคำตอบจริง (150,000,000) ทั้งหมด — เพราะ `float` มี mantissa แค่ 24 บิต
เมื่อผลรวมมีค่าใหญ่กว่า 2²⁴ ≈ 16.7 ล้าน การบวก `1.5f` ที่มีขนาดเล็กเข้าไปจะถูก **ปัดทิ้งจนไม่มีผล**
(rounding error) ทำให้ผลรวมแบบบวกทีละตัว (sequential) จะ "ค้าง" ที่ประมาณ 2²⁵ = 33,554,432
พอดี แต่ที่ `-Ofast` ผลลัพธ์กลับเปลี่ยนไปเป็น 134,217,728 (= 2²⁷) เพราะ flag นี้เปิด `-ffast-math`
โดยปริยาย ซึ่งอนุญาตให้ compiler **จัดลำดับการบวกใหม่** (ละเมิด associativity ตามมาตรฐาน IEEE 754)
เพื่อ vectorize เป็นหลาย lane พร้อมกัน (เช่น บวกแยก 8 ยอดใน register 256-bit แล้วค่อยรวมทีหลัง)
ทำให้แต่ละ lane ปัดเศษ "ค้าง" ที่ค่าต่างกัน แล้วเมื่อรวมกันได้ผลที่ต่างจากการบวกทีละตัวโดยสิ้นเชิง

**ข้อสรุป**: `-Ofast` ไม่ใช่แค่ "เร็วกว่า" — มันเปลี่ยนความหมายทางคณิตศาสตร์ของโปรแกรมได้จริง
ห้ามใช้กับโค้ดที่ต้องการผลลัพธ์ทศนิยมแม่นยำตรงตามมาตรฐาน (เช่น การเงิน, การจำลองทางวิทยาศาสตร์
ที่ตรวจสอบผลด้วยค่าคงที่ตายตัว) โดยไม่ทดสอบผลกระทบอย่างละเอียดก่อน

### -Os กับขนาดไบนารี

`-Os` เหมาะกับ Embedded System ที่มี Flash/ROM จำกัด ลองดูตัวอย่างขนาดไฟล์จริงจากโปรแกรมที่มี
ฟังก์ชันคณิตศาสตร์ (`sin`, `cos`, `sqrt`, `fmod`) ผสมกันหลายจุด:

```bash
for level in O0 O1 O2 O3 Os; do
    g++ -${level} -Wall -Wextra -Wpedantic -std=c++17 size_demo.cpp -o size_${level} -lm
done
size size_O0 size_O1 size_O2 size_O3 size_Os
```

ผลลัพธ์จริง (คอลัมน์ `text` คือขนาดโค้ดจริงหน่วย byte):

```
   text	   data	    bss	    dec	    hex	filename
   2499	    648	      8	   3155	    c53	size_O0
   2414	    640	      8	   3062	    bf6	size_O1
   2436	    640	      8	   3084	    c0c	size_O2
   2436	    640	      8	   3084	    c0c	size_O3
   2371	    648	      8	   3027	    bd3	size_Os
```

`-Os` ให้ text segment เล็กที่สุด (2371 byte) แม้ `-O2`/`-O3` จะเร็วกว่าในการรันจริง

---

## 89.2 Dead Code Elimination และ Constant Folding (Step 706)

**Constant Folding**: ถ้าค่าทุกตัวที่เกี่ยวข้องกับนิพจน์รู้ล่วงหน้าตอน compile-time compiler จะ
คำนวณผลลัพธ์ไว้เลย ไม่ต้องสร้างโค้ดคำนวณตอน runtime

**Dead Code Elimination (DCE)**: ถ้าค่าที่คำนวณไม่เคยถูกใช้ที่ไหนเลย (ไม่มีผลต่อ observable
behavior ของโปรแกรม) compiler จะลบโค้ดส่วนนั้นทิ้งไปเลย

ดูตัวอย่างร่วมกัน:

```cpp
#include <cstdio>

int compute(int x) {
    int a = 10;
    int b = 20;
    int c = a + b;       // Constant Folding: compiler รู้ค่า a+b=30 ตั้งแต่ compile-time
    int unused = x * 2;  // Dead Code: ไม่เคยถูกใช้ที่ไหนต่อเลย
    (void)unused;
    return c;             // คืนค่า 30 เสมอ ไม่ว่า x จะเป็นอะไร
}

int main() {
    printf("%d\n", compute(99));
    return 0;
}
```

คอมไพล์เป็น Assembly เปรียบเทียบ:

```bash
g++ -O0 -Wall -Wextra -Wpedantic -std=c++17 -S dce_cf.cpp -o dce_cf_O0.s
g++ -O2 -Wall -Wextra -Wpedantic -std=c++17 -S dce_cf.cpp -o dce_cf_O2.s
```

Assembly ของ `compute()` ที่ `-O0` (ทำงานตาม source ตรงๆ ทุกบรรทัด):

```asm
_Z7computei:
    endbr64
    pushq   %rbp
    movq    %rsp, %rbp
    movl    %edi, -20(%rbp)
    movl    $10, -16(%rbp)   ; a = 10
    movl    $20, -12(%rbp)   ; b = 20
    movl    -16(%rbp), %edx
    movl    -12(%rbp), %eax
    addl    %edx, %eax       ; a + b (คำนวณตอน runtime จริงๆ)
    movl    %eax, -8(%rbp)   ; c = ผลบวก
    movl    -20(%rbp), %eax
    addl    %eax, %eax       ; x * 2 (คำนวณทั้งที่ไม่ได้ใช้!)
    movl    %eax, -4(%rbp)   ; unused = ผลคูณ
    movl    -8(%rbp), %eax   ; return c
    popq    %rbp
    ret
```

Assembly ของ `compute()` ที่ `-O2` — เหลือแค่ 2 บรรทัด:

```asm
_Z7computei:
    endbr64
    movl    $30, %eax   ; คำนวณ a+b=30 ไว้ล่วงหน้าแล้ว ไม่มีการคำนวณ x*2 เหลืออยู่เลย
    ret
```

และที่ `main()` ยิ่งไปไกลกว่านั้นอีก — compiler **inline** `compute()` เข้าไปใน `main()` โดยตรง
แล้วเห็นว่าอาร์กิวเมนต์ `99` เป็นค่าคงที่ จึงคำนวณผลลัพธ์ทั้งหมดตั้งแต่ compile-time:

```asm
main:
    endbr64
    subq    $8, %rsp
    movl    $30, %edx        ; ค่าที่จะพิมพ์คือ 30 คำนวณไว้แล้วตอน compile-time
    ...
    call    __printf_chk@PLT
    ...
```

ไม่มีการเรียก `compute()` เหลืออยู่ใน `main()` เลยแม้แต่น้อย — นี่คือพลังของการรวม **Inlining +
Constant Folding + Dead Code Elimination** เข้าด้วยกัน ซึ่งเป็นเหตุผลสำคัญที่ทำไมโค้ดที่ `-O2`
มักจะเร็วกว่า `-O0` แบบก้าวกระโดดในหลายกรณี (ไม่ใช่แค่กรณีนี้กรณีเดียว)

---

## 89.3 Loop Unrolling (Step 707)

**Loop Unrolling** คือการที่ compiler "คลี่" ลูปที่รู้จำนวนรอบแน่นอนออกมาเป็นคำสั่งเรียงต่อกันตรงๆ
โดยไม่ต้องมีการกระโดด (jump) กลับไปตรวจเงื่อนไขซ้ำๆ ทำให้ลด overhead ของการวนลูป (เช่น การ
เพิ่มค่า counter, การเปรียบเทียบเงื่อนไข, การกระโดด) ลงไปได้มาก

```cpp
#include <cstdio>

long fixed_loop(const int arr[4]) {
    long sum = 0;
    for (int i = 0; i < 4; i++) {
        sum += arr[i];
    }
    return sum;
}

int main(int argc, char *argv[]) {
    (void)argv;
    int arr[4] = {argc, argc + 1, argc + 2, argc + 3};
    printf("%ld\n", fixed_loop(arr));
    return 0;
}
```

ที่ `-O0` ได้ลูปจริงพร้อม label กระโดดกลับ (`.L3`/`.L2`) ครบตามโครงสร้าง `for`:

```asm
_Z10fixed_loopPKi:
    ...
    movq    $0, -8(%rbp)      ; sum = 0
    movl    $0, -12(%rbp)     ; i = 0
    jmp     .L2
.L3:
    ... (โหลด arr[i] แล้วบวกเข้า sum, เพิ่ม i)
.L2:
    cmpl    $3, -12(%rbp)     ; ตรวจเงื่อนไข i <= 3 ทุกรอบ
    jle     .L3
    movq    -8(%rbp), %rax
    ret
```

ที่ `-O2` (ปิดการ vectorize ไว้ก่อนด้วย `-fno-tree-vectorize` เพื่อดู "unrolling ล้วนๆ" แยกจาก SIMD):

```bash
g++ -O2 -fno-tree-vectorize -Wall -Wextra -Wpedantic -std=c++17 -S unroll.cpp -o unroll_O2.s
```

```asm
_Z10fixed_loopPKi:
    endbr64
    movslq  4(%rdi), %rdx     ; โหลด arr[1]
    movslq  (%rdi), %rax      ; โหลด arr[0]
    addq    %rdx, %rax        ; arr[0] + arr[1]
    movslq  8(%rdi), %rdx     ; โหลด arr[2]
    addq    %rax, %rdx
    movslq  12(%rdi), %rax    ; โหลด arr[3]
    addq    %rdx, %rax
    ret
```

ไม่มี label กระโดด ไม่มีการตรวจเงื่อนไขซ้ำเลย — เพราะ compiler รู้ว่าลูปนี้วนแค่ 4 รอบตายตัว
(trip count คงที่และรู้ตอน compile-time) จึงเขียนโค้ดเรียงต่อกันตรงๆ ทั้ง 4 การบวกเลย
เรียกว่า **unroll เต็มรูปแบบ (fully unrolled)**

ถ้า**ไม่ปิด** vectorization (`-fno-tree-vectorize`) compiler ที่เก่งขึ้นไปอีกขั้นจะไม่แค่ unroll
แต่จะใช้ SIMD register (`xmm`) รวมข้อมูลทั้ง 4 ตัวและบวกพร้อมกันในคำสั่งเดียว (`paddq`) — เรื่องนี้
เราลงลึกไปแล้วใน **Part 88 (SIMD และ Vectorization)** ในที่นี้จะเห็นว่า Unrolling กับ Vectorization
เป็นคนละเทคนิคกัน แต่มักถูกใช้ร่วมกันเพื่อประสิทธิภาพสูงสุด

---

## 89.4 Inlining: การตัดสินใจของ Compiler ไม่ใช่ Keyword (Step 708)

ความเข้าใจผิดที่พบบ่อยที่สุดอย่างหนึ่งในภาษา C++ คือ "ใส่ `inline` แล้ว compiler จะ inline
ฟังก์ชันนั้นให้แน่นอน" — **ความจริงแล้วไม่ใช่เลย** ตั้งแต่ C++98 คำว่า `inline` มีความหมายหลักคือ
"อนุญาตให้ฟังก์ชันนี้ถูกนิยามซ้ำได้ในหลายๆ Translation Unit โดยไม่ผิดกฎ One Definition Rule"
(เพื่อให้ประกาศฟังก์ชันแบบเต็มไว้ในไฟล์ `.h` ได้) ส่วนเรื่อง "จะแทรกโค้ดเข้าไปแทนที่การเรียกฟังก์ชัน
จริงหรือไม่" นั้นเป็น**การตัดสินใจของ compiler ล้วนๆ** โดยพิจารณาจากขนาดของฟังก์ชัน ความซับซ้อน
จำนวนครั้งที่ถูกเรียก และ optimization level ที่ใช้

### Compiler inline ให้เองแม้ไม่ใส่ keyword

```cpp
#include <cstdio>

// ฟังก์ชันเล็กมาก -- ไม่มี inline keyword เลย แต่ compiler จะ inline ให้เองที่ -O2
int square(int x) {
    return x * x;
}

int main() {
    printf("%d\n", square(7));
    return 0;
}
```

ที่ `-O2` ตรวจสอบด้วย `objdump`/`-S` จะพบว่า `main()` ไม่มีการ `call` ไปยัง `square` เลย
มีแค่ `movl $49, ...` (7×7=49) คำนวณไว้ตั้งแต่ compile-time — compiler ตัดสินใจ inline เองล้วนๆ
โดยไม่สนใจว่าเราใส่ `inline` หรือไม่

### เมื่อ inline keyword ถูก "ปฏิเสธ"

ลองบังคับให้ compiler มี "งบประมาณการ inline" (inline budget) น้อยมากๆ ด้วย
`-finline-limit` เพื่อดูว่าเมื่อ compiler ตัดสินใจว่าฟังก์ชัน "ใหญ่เกินไป" จะเกิดอะไรขึ้น:

```cpp
#include <cstdio>

inline long heavy_work(int n) {
    long total = 0;
    for (int i = 1; i <= n; i++) {
        total += (i * i) % 97;
        total ^= (i << 2);
        total += (total >> 3);
    }
    return total;
}

long call_a(int n) { return heavy_work(n) + 1; }

int main(int argc, char *argv[]) {
    (void)argv;
    printf("%ld\n", call_a(argc));
    return 0;
}
```

```bash
# ค่า default -- compiler ตัดสินใจ "คุ้ม" จึง inline ให้ ไม่มี warning
g++ -O2 -Wall -Wextra -Wpedantic -Winline -std=c++17 -c inline_demo.cpp -o /dev/null

# บังคับจำกัดงบ inline ให้เหลือน้อยมาก (เพื่อสาธิตเท่านั้น ห้ามใช้ค่านี้จริง)
g++ -O2 -Wall -Wextra -Wpedantic -Winline -finline-limit=1 -std=c++17 -c inline_demo.cpp -o /dev/null
```

เมื่อจำกัด `-finline-limit=1` (ค่าที่เล็กเกินจริงเพื่อการสาธิต) เราจะได้ Warning จริงจาก
`-Winline` ที่บอกตรงๆ ว่า compiler **ปฏิเสธ** ที่จะ inline แม้เราจะประกาศ `inline` ไว้:

```
inline_demo.cpp: In function 'long int call_a(int)':
inline_demo.cpp:3:13: warning: inlining failed in call to 'long int heavy_work(int)':
  --param max-inline-insns-single limit reached [-Winline]
    3 | inline long heavy_work(int n) {
      |             ^~~~~~~~~~
inline_demo.cpp:13:39: note: called from here
   13 | long call_a(int n) { return heavy_work(n) + 1; }
      |                             ~~~~~~~~~~^~~
```

นี่คือหลักฐานที่ชัดเจนที่สุดว่า `inline` เป็นแค่ "คำขอ" — compiler มีสิทธิ์เต็มที่จะปฏิเสธถ้าเห็นว่า
การแทรกโค้ดเข้าไปแทนที่ทุกจุดที่เรียกจะทำให้ไบนารีใหญ่เกินคุ้ม (เสี่ยงต่อ I-Cache miss ที่ทำให้
โปรแกรมช้าลงในภาพรวม) ในทางกลับกัน compiler ก็มีสิทธิ์ inline ฟังก์ชันที่**ไม่ได้**ประกาศ `inline`
เลยด้วยซ้ำ ถ้าเห็นว่าเล็กพอและคุ้มค่า (เหมือนตัวอย่าง `square()` ด้านบน)

> **สรุป**: `inline` ใน C++ สมัยใหม่ควรมองว่าเป็นเรื่อง **Linkage** (อนุญาตให้นิยามซ้ำได้หลาย TU)
> ไม่ใช่เรื่อง Performance โดยตรง หากต้องการบังคับ compiler จริงๆ ต้องใช้ compiler-specific
> attribute เช่น `__attribute__((always_inline))` (GCC/Clang) ซึ่งก็ยังมีข้อจำกัดในบางกรณีอยู่ดี
> (เช่น ฟังก์ชัน recursive ไม่มีทางถูก inline สมบูรณ์ได้)

---

## 89.5 Undefined Behavior คือศัตรูตัวฉกาจของ Optimization (Step 709)

นี่คือหัวข้อที่สำคัญที่สุดของ Part นี้ มาตรฐานภาษา C++ กำหนดว่า **ถ้าโปรแกรมมี Undefined
Behavior (UB) compiler มีสิทธิ์ทำอะไรก็ได้** เพราะถือว่าโปรแกรมนั้น "ไม่ถูกต้องตามมาตรฐาน" ไปแล้ว
Compiler สมัยใหม่ใช้ข้อสมมติฐานนี้เป็นเครื่องมือ optimize อย่างจริงจัง: **compiler จะสมมติเสมอว่า
โปรแกรมของเราไม่มี UB** แล้ว optimize ตามข้อสมมติฐานนั้น — ถ้าโปรแกรมจริงมี UB ผลลัพธ์จึง
"พัง" ในแบบที่ไม่มีใครคาดเดาได้ และมักจะพังหนักขึ้นเมื่อเปิด optimization ระดับสูงขึ้น

### ตัวอย่างจริง: Signed Integer Overflow ทำให้ลูปกลายเป็น Infinite Loop

Signed integer overflow (เช่น `INT_MAX + 1`) เป็น **Undefined Behavior** ตามมาตรฐาน C++
(ต่างจาก unsigned ที่การ wrap-around ถูกนิยามไว้ชัดเจนว่าไม่ใช่ UB) ลองดูโค้ดนี้:

```cpp
// ตัวอย่างนี้มี Undefined Behavior โดยตั้งใจ เพื่อสาธิตผลกระทบของ optimization เท่านั้น
// ห้ามเขียนโค้ดแบบนี้ในงานจริงเด็ดขาด
#include <cstdio>
#include <climits>

int main() {
    int i = INT_MAX - 5;
    long steps = 0;
    while (i >= 0) {
        i++;      // เมื่อ i ไปถึง INT_MAX แล้ว i++ ต่อไปคือ signed overflow (UB)
        ++steps;
    }
    printf("steps = %ld, i = %d\n", steps, i);
    return 0;
}
```

ตามตรรกะที่มือใหม่คาดหวัง: `i` จะเพิ่มจาก `INT_MAX-5` ไปเรื่อยๆ จนถึง `INT_MAX` แล้ว "wrap"
กลับไปเป็นค่าติดลบ (`INT_MIN`) ทำให้เงื่อนไข `i >= 0` เป็นเท็จและลูปจบลง — เหมือนพฤติกรรมของ
เลข unsigned ที่วนกลับ แต่เพราะ `int` เป็น signed การ overflow แบบนี้คือ UB **ไม่ใช่พฤติกรรมที่
มาตรฐานรับประกัน**

คอมไพล์และรันจริงที่ `-O0` กับ `-O2`:

```bash
g++ -O0 -Wall -Wextra -Wpedantic -std=c++17 ub_overflow.cpp -o ubof_O0
g++ -O2 -Wall -Wextra -Wpedantic -std=c++17 ub_overflow.cpp -o ubof_O2
```

ที่ `-O2` GCC เตือนเราไว้ล่วงหน้าจริงๆ ระหว่างคอมไพล์:

```
ub_overflow.cpp: In function 'int main()':
ub_overflow.cpp:8:10: warning: iteration 5 invokes undefined behavior
  [-Waggressive-loop-optimizations]
    8 |         i++;
      |         ~^~
ub_overflow.cpp:7:14: note: within this loop
    7 |     while (i >= 0) {
      |            ~~^~~~
```

ผลการรันจริง:

```bash
$ timeout 3 ./ubof_O0
steps = 6, i = -2147483648        # จบลูปตามที่คาดหวัง (ใช้เวลาไม่ถึง 1 วินาที)

$ timeout 3 ./ubof_O2
(ไม่มีอะไรพิมพ์ออกมาเลย -- โปรแกรมค้าง จน timeout ฆ่าทิ้งหลัง 3 วินาที, exit code 124)
```

ที่ `-O0` โปรแกรม "ดูเหมือนทำงานถูกต้อง" เพราะ CPU จริงบวกเลขแบบ two's complement ซึ่งบังเอิญ
ทำให้ wrap-around แบบที่มือใหม่คาดหวัง แต่ที่ `-O2` เมื่อดู Assembly ของ `main()` เราจะพบสิ่งที่
น่าตกใจมาก:

```asm
main:
    endbr64
.L2:
    jmp     .L2      ; วนกระโดดไปมาเฉยๆ ไม่มีอะไรอย่างอื่นเลย!
```

**ทั้งฟังก์ชัน `main()` เหลือแค่ 2 บรรทัด** ไม่มีแม้แต่การเรียก `printf` เลย! เหตุผลคือ: compiler
สมมติว่า signed overflow ไม่มีทางเกิดขึ้น (เพราะเป็น UB) ดังนั้นมันจึง**พิสูจน์ทางคณิตศาสตร์**ได้ว่า
ถ้า `i` เริ่มจากค่าที่ `>= 0` แล้วบวกทีละ 1 ไปเรื่อยๆ โดยไม่มี overflow ค่า `i` จะ `>= 0` ตลอดไป
ไม่มีทางเป็นเท็จได้เลย ลูปนี้จึง**ต้องเป็น infinite loop แน่นอนตามตรรกะที่ไม่มี UB** — และเนื่องจาก
โค้ดหลังลูป (`printf`) ไม่มีทางถูกรันได้เลย (ลูปไม่จบ) compiler จึงลบมันทิ้งไปทั้งหมด เหลือแค่
infinite loop เปล่าๆ

นี่คือบทเรียนสำคัญที่สุดของ Part นี้: **โค้ดที่มี UB อาจ "ดูทำงานถูกต้อง" ที่ `-O0` แต่พังโดยสิ้นเชิง
ที่ `-O2`/`-O3`** และการพังก็ไม่จำเป็นต้อง crash เสมอไป — บางครั้งมันคือ infinite loop, บางครั้ง
คือผลลัพธ์ที่ผิดแบบเงียบๆ, บางครั้งคือโค้ดความปลอดภัย (เช่นการเช็ค null pointer) ถูกลบทิ้งไปเฉยๆ

### บทเรียนเชิงปฏิบัติ

1. **`-Wall -Wextra` ไม่พอ** สำหรับจับ UB ทุกแบบ ควรใช้ Sanitizer (`-fsanitize=undefined`)
   ร่วมด้วยเสมอตอน develop/test (จะเรียนละเอียดใน Part 96)
2. อย่าไว้ใจว่าโค้ดที่ "รันผ่านตอน debug (`-O0`)" จะทำงานถูกต้องตอน release (`-O2`/`-O3`)
   เสมอ — UB คือ "ระเบิดเวลา" ที่ optimization level สูงมักจุดชนวนให้แสดงผล
3. เมื่อสงสัยว่าโค้ดที่ optimize แล้วทำงานแปลกไป ให้ตรวจสอบ UB เป็นอันดับแรกๆ (signed
   overflow, dereference null/dangling pointer, ใช้ตัวแปรที่ไม่ได้ initialize, out-of-bounds access,
   strict aliasing violation)

---

## 89.6 -march=native: ปลดล็อกพลังเต็มที่ของ CPU เครื่องนี้ (Step 710)

ค่า default ของ GCC คือ `-march=x86-64` ซึ่งสร้างโค้ดที่รันได้บน CPU x86-64 **แทบทุกรุ่น**
ตั้งแต่ปี 2003 เป็นต้นมา (baseline ที่ปลอดภัยที่สุด) แต่นั่นแปลว่า compiler จะ**ไม่กล้าใช้**ชุดคำสั่ง
ที่ทันสมัยกว่า เช่น AVX, AVX2, AVX-512, FMA แม้ CPU เครื่องที่รันจริงจะรองรับก็ตาม

`-march=native` บอกให้ compiler ตรวจสอบ CPU ของเครื่องที่**กำลังคอมไพล์อยู่ ณ ขณะนั้น** แล้วใช้
ชุดคำสั่งเต็มที่เท่าที่ CPU นั้นรองรับ

ตรวจสอบ CPU ของเครื่องนี้ก่อน:

```bash
$ lscpu | grep "Model name"
Model name:  Intel(R) Xeon(R) Processor @ 2.10GHz

$ cat /proc/cpuinfo | grep -m1 flags | tr ' ' '\n' | grep -E "avx|fma"
fma
avx
avx2
avx512f
avx512dq
...
```

CPU เครื่องนี้รองรับ AVX-512 เต็มรูปแบบ ลองเปรียบเทียบ Assembly ของฟังก์ชัน dot product:

```cpp
#include <cstdio>
#include <chrono>
#include <vector>

double dot_product(const double* a, const double* b, int n) {
    double sum = 0.0;
    for (int i = 0; i < n; i++) {
        sum += a[i] * b[i];
    }
    return sum;
}

int main() {
    const int n = 50000000;
    std::vector<double> a(n, 1.0001), b(n, 2.0002);

    auto t0 = std::chrono::steady_clock::now();
    double result = dot_product(a.data(), b.data(), n);
    auto t1 = std::chrono::steady_clock::now();
    double ms = std::chrono::duration<double, std::milli>(t1 - t0).count();
    printf("result=%.4f time=%.3f ms\n", result, ms);
    return 0;
}
```

```bash
g++ -O3 -Wall -Wextra -Wpedantic -std=c++17 march_demo.cpp -o march_generic
g++ -O3 -march=native -Wall -Wextra -Wpedantic -std=c++17 march_demo.cpp -o march_native
```

ตรวจสอบ register ที่ใช้จริงด้วย `objdump -d`:

```bash
$ objdump -d march_generic | grep -m1 -E "ymm|xmm"
1166: f2 0f 10 05 ...   movsd  ...,%xmm0        # SSE2 (128-bit) เท่านั้น

$ objdump -d march_native | grep -m1 -E "ymm|xmm"
1179: c4 e2 7d 19 05 ... vbroadcastsd ...,%ymm0  # AVX (256-bit) ใช้จริง!

$ objdump -d march_native | grep -m1 "vfmadd"
1471: c4 e2 d1 b9 04 c1  vfmadd231sd (%rcx,%rax,8),%xmm5,%xmm0   # FMA instruction
```

ยืนยันชัดเจนว่า `-march=native` ทำให้ compiler ใช้ `ymm` register (AVX, กว้าง 256-bit เทียบกับ
`xmm` ที่กว้างแค่ 128-bit) และคำสั่ง `vfmadd231sd` (Fused Multiply-Add — คูณและบวกในคำสั่งเดียว
ด้วยความแม่นยำสูงกว่าการคูณแล้วบวกแยกกัน) ได้จริง

ในตัวอย่างนี้เวลาทำงานจริงใกล้เคียงกันมาก (~80ms ทั้งคู่) เพราะ dot product ของ array ขนาดใหญ่
ขนาดนี้ (50 ล้าน `double` × 2 array = 800MB) เป็นงานที่ **ติดคอขวดที่ bandwidth หน่วยความจำ**
ไม่ใช่ที่ความเร็วคำนวณของ ALU — นี่คือบทเรียนสำคัญอีกข้อ: **SIMD/AVX ช่วยได้มากเมื่องานติดคอขวด
ที่การคำนวณ (compute-bound) แต่ช่วยได้น้อยมากเมื่องานติดคอขวดที่หน่วยความจำ (memory-bound)**
ต้องวัดผลจริงเสมอ ไม่ใช่เดาจากทฤษฎี

> **ข้อควรระวังสำคัญ**: ไบนารีที่คอมไพล์ด้วย `-march=native` **จะรันไม่ได้หรือ crash ด้วย
> Illegal Instruction (SIGILL)** บนเครื่องอื่นที่ CPU ไม่รองรับชุดคำสั่งเดียวกัน (เช่น เครื่องเก่ากว่า
> หรือ CPU cloud ที่ไม่ใช่รุ่นเดียวกับตอน build) **ห้ามใช้ `-march=native` สำหรับไบนารีที่จะแจกจ่าย
> ไปยังเครื่องอื่นที่ไม่รู้ว่า CPU เป็นรุ่นอะไร** เหมาะกับกรณีที่ compile และ deploy บนเครื่องเดียวกัน
> เท่านั้น (เช่น build ใน Docker container ที่จะรันบน CPU เดียวกันเป๊ะ) สำหรับไบนารีที่แจกจ่ายทั่วไป
> ควรใช้ `-march=x86-64-v2`/`-v3` (ระบุ baseline ที่กว้างกว่าแต่ยังทันสมัยกว่า default) แทน

---

## 89.7 -flto: Link Time Optimization (Step 711)

ปกติแล้ว GCC จะ optimize **ทีละไฟล์ `.cpp` เท่านั้น** (เรียกว่า per-Translation-Unit optimization)
เพราะแต่ละไฟล์ถูกคอมไพล์แยกกันเป็น `.o` ก่อนแล้วค่อยเอามา link รวมกันทีหลัง — แปลว่าถ้าฟังก์ชัน
`A()` อยู่ไฟล์หนึ่ง เรียกใช้ฟังก์ชัน `B()` ที่อยู่อีกไฟล์หนึ่ง compiler **มองไม่เห็น** ว่า `B()` ทำอะไร
ตอนกำลัง optimize ไฟล์ของ `A()` จึง inline ข้ามไฟล์ไม่ได้เลย

`-flto` (Link Time Optimization) แก้ปัญหานี้โดยเก็บข้อมูล intermediate representation ไว้ใน
`.o` file แล้วให้ **linker** เป็นคนทำ optimization ข้ามไฟล์อีกรอบตอน link จริง

ทดลองด้วย 2 ไฟล์:

`mathutil.h`:
```cpp
#ifndef MATHUTIL_H
#define MATHUTIL_H
int triple(int x);
#endif
```

`mathutil.cpp`:
```cpp
#include "mathutil.h"

int triple(int x) {
    return x * 3;
}
```

`lto_main.cpp`:
```cpp
#include <cstdio>
#include "mathutil.h"

int main() {
    int result = triple(14);
    printf("%d\n", result);
    return 0;
}
```

**คอมไพล์แบบปกติ (ไม่มี LTO):**

```bash
g++ -O2 -Wall -Wextra -Wpedantic -std=c++17 -c mathutil.cpp -o mathutil.o
g++ -O2 -Wall -Wextra -Wpedantic -std=c++17 -c lto_main.cpp -o lto_main.o
g++ -O2 mathutil.o lto_main.o -o lto_nolto
```

ดู Assembly ของ `main()` ด้วย `objdump -d lto_nolto`:

```asm
<main>:
    endbr64
    sub    $0x8,%rsp
    mov    $0xe,%edi          ; 0xe = 14
    call   1180 <_Z6triplei> ; ยังคง "เรียก" ฟังก์ชัน triple() จริง ข้ามไฟล์ inline ไม่ได้
    lea    ...(%rip),%rsi
    ...
```

**คอมไพล์แบบมี LTO:**

```bash
g++ -O2 -flto -Wall -Wextra -Wpedantic -std=c++17 -c mathutil.cpp -o mathutil_lto.o
g++ -O2 -flto -Wall -Wextra -Wpedantic -std=c++17 -c lto_main.cpp -o lto_main_lto.o
g++ -O2 -flto mathutil_lto.o lto_main_lto.o -o lto_yes
```

```asm
<main>:
    endbr64
    sub    $0x8,%rsp
    mov    $0x2a,%edx        ; 0x2a = 42 (= 14*3 คำนวณไว้ล่วงหน้าแล้ว!)
    mov    $0x2,%edi
    xor    %eax,%eax
    lea    ...(%rip),%rsi
    call   1050 <__printf_chk@plt>
    ...
```

ไม่มีการเรียก `triple()` เหลืออยู่เลย — linker มองเห็นทั้งสองไฟล์พร้อมกัน จึง inline `triple(14)`
ข้ามไฟล์แล้วคำนวณผลลัพธ์ (42) ไว้ล่วงหน้าได้ทันที ทั้งสองไบนารีให้ผลลัพธ์ถูกต้องเหมือนกัน (`42`)
แต่ `lto_yes` ทำงานเร็วกว่าเพราะไม่มี function call overhead เหลืออยู่เลย

**ในโปรเจกต์จริงที่มีไฟล์เป็นร้อยเป็นพันไฟล์** `-flto` มักให้ผลต่างที่มีนัยสำคัญมาก เพราะเปิดโอกาส
ให้ compiler มองเห็นภาพรวมทั้งโปรเจกต์แทนที่จะมองทีละไฟล์แยกกัน แลกมาด้วยเวลา **link ที่นานขึ้น
มาก** (เพราะ optimization หนักๆ ย้ายไปทำตอน link แทน) จึงมักเปิดใช้เฉพาะตอน build โหมด Release
เท่านั้น ไม่ใช้ตอน develop ที่ต้อง build บ่อยๆ

---

## 89.8 สรุปแนวทางเลือก Flag สำหรับ Debug Build และ Release Build (Step 712)

รวบยอดทุกอย่างที่เรียนมาใน Part นี้เป็นแนวทางปฏิบัติจริง:

| สถานการณ์ | Flag ที่แนะนำ | เหตุผล |
|---|---|---|
| กำลัง develop/debug ด้วย GDB | `-O0 -g -Wall -Wextra -Wpedantic` | ตัวแปรตรงกับ source เป๊ะ debug ง่ายที่สุด |
| Build ทดสอบ (CI, unit test) | `-O1 -g -fsanitize=address,undefined` | สมดุลระหว่างความเร็วกับการจับ Sanitizer ได้ |
| Release build ทั่วไป | `-O2 -DNDEBUG` | มาตรฐานอุตสาหกรรม ปลอดภัยและเร็ว |
| Release + ต้องการ portability ข้ามเครื่อง | `-O2 -march=x86-64-v2` (หรือ v3) | เร็วขึ้นแต่ยังรันข้ามเครื่องรุ่นใกล้เคียงได้ |
| Deploy บนเครื่องเดียวกับที่ build (เช่น Docker เดียวกัน) | `-O3 -march=native -flto` | ดึงประสิทธิภาพสูงสุดเท่าที่ CPU ให้ได้ |
| งานตัวเลขที่ยอมรับ precision คลาดเคลื่อนได้ | เพิ่ม `-Ofast` จากข้างบน | เร็วขึ้นอีก แต่ต้องทดสอบผลลัพธ์อย่างละเอียด |
| Embedded/พื้นที่โปรแกรมจำกัดมาก | `-Os` | ไบนารีเล็กที่สุดในบรรดา optimization level ที่มี |

**กฎทองข้อใหม่ของหลักสูตรนี้**: ไม่ว่าจะใช้ optimization level ไหน ให้เปิด `-Wall -Wextra
-Wpedantic` เสมอ และก่อนปล่อย production จริงควรรันผ่าน Sanitizer (`-fsanitize=undefined,address`)
อย่างน้อยหนึ่งรอบเต็ม เพราะดังที่เห็นใน 89.5 — UB ที่ `-O0` "ดูทำงานได้" ไม่ได้แปลว่าโค้ดถูกต้อง

---

## ข้อผิดพลาดที่พบบ่อย (Common Pitfalls)

1. **เข้าใจผิดว่า `-O3` เร็วกว่า `-O2` เสมอ** — ในความเป็นจริง aggressive unrolling/inlining ของ
   `-O3` อาจทำให้โค้ดใหญ่จน I-Cache miss บ่อยขึ้นจนช้าลงกว่า `-O2` ได้ในบางเวิร์กโหลด ต้องวัดจริง
2. **ใช้ `-Ofast` กับโค้ดที่ต้องการความแม่นยำทศนิยม** — อย่างที่เห็นใน 89.1 ผลลัพธ์อาจเปลี่ยนไป
   จากการจัดลำดับการบวก/คูณใหม่ (`-ffast-math`) ไม่ใช่แค่เรื่องความเร็วเท่านั้น
3. **มั่นใจว่าโค้ดถูกต้องเพราะรันผ่านตอน `-O0`** — โค้ดที่มี UB สามารถ "ดูทำงานถูกต้อง" ที่ `-O0`
   แล้วพังทันทีที่เปิด `-O2`/`-O3` (ตัวอย่าง infinite loop จาก signed overflow ใน 89.5)
4. **ใส่ `inline` แล้วคาดหวังว่าฟังก์ชันจะถูก inline แน่นอน** — compiler มีสิทธิ์ปฏิเสธเสมอถ้าเห็นว่า
   ฟังก์ชันใหญ่เกินไปหรือถูกเรียกจากหลายจุดจนไม่คุ้ม (ตรวจสอบได้ด้วย `-Winline`)
5. **แจกจ่ายไบนารีที่ build ด้วย `-march=native` ไปยังเครื่องอื่น** — ถ้าเครื่องปลายทาง CPU ไม่รองรับ
   ชุดคำสั่งเดียวกัน (เช่น ไม่มี AVX-512) โปรแกรมจะ crash ด้วย `SIGILL` (Illegal Instruction) ทันที
6. **ลืมว่า `-flto` ทำให้เวลา link นานขึ้นมาก** — ถ้าเปิดใช้ตอน develop ที่ build บ่อยๆ จะทำให้
   วงจร edit-compile-run ช้าลงโดยไม่จำเป็น ควรเปิดเฉพาะตอน build release เท่านั้น

---

## แบบฝึกหัดท้ายบท

1. เขียนฟังก์ชันคำนวณ Fibonacci แบบวนลูป (ไม่ใช่ recursive) แล้วคอมไพล์ด้วยทั้ง 6 optimization
   level (`-O0`, `-O1`, `-O2`, `-O3`, `-Os`, `-Ofast`) วัดเวลาการทำงานจริงด้วย `std::chrono`
   ที่ n = 1,000,000,000 แล้วสร้างตารางเปรียบเทียบผลลัพธ์ด้วยตัวเอง (ทำให้ n มาจาก `argv` เพื่อ
   ป้องกัน compiler คำนวณคำตอบไว้ล่วงหน้าทิ้งลูปทั้งหมด)
2. คอมไพล์ฟังก์ชันในข้อ 1 ด้วย `-S` ที่ `-O0` และ `-O2` แล้วเปิดไฟล์ `.s` ทั้งสองไฟล์ดูด้วยตัวเอง
   อธิบายเป็นคำพูดตัวเองว่า optimization อะไรเกิดขึ้นบ้าง (ใช้ความรู้จาก 89.2 และ 89.3)
3. เขียนโปรแกรมที่มี Undefined Behavior จากการเข้าถึง array นอกขอบเขต (out-of-bounds access)
   แทนที่จะเป็น signed overflow แล้วเปรียบเทียบพฤติกรรมที่ `-O0` กับ `-O2` ว่าต่างกันหรือไม่
   อย่างไร (คำใบ้: ลองรันผ่าน `-fsanitize=address` ด้วยเพื่อดูว่า Sanitizer จับ UB นี้ได้จริง)
4. ทดลองใช้ `-finline-limit` ค่าต่างๆ กับฟังก์ชัน `heavy_work` ในหัวข้อ 89.4 (ลอง 1, 10, 50, 100,
   ค่า default) พร้อม `-Winline` แล้วสังเกตว่าที่ค่าประมาณเท่าไหร่ compiler เริ่มยอม inline ให้
5. สร้างโปรเจกต์ 3 ไฟล์ (`.h` + 2 `.cpp`) ที่มีฟังก์ชันเรียกกันข้ามไฟล์อย่างน้อย 2 ชั้น (A เรียก B,
   B เรียก C) แล้วเปรียบเทียบ Assembly ของ `main()` เมื่อ build แบบไม่มี `-flto` กับมี `-flto`
6. (โบนัส) ตรวจสอบด้วย `lscpu`/`/proc/cpuinfo` ว่าเครื่องของคุณรองรับชุดคำสั่งอะไรบ้าง แล้วลอง
   คอมไพล์โค้ดที่มีลูปคำนวณเลขทศนิยมจำนวนมากด้วย `-O3` เทียบกับ `-O3 -march=native` วัดเวลา
   จริง และอธิบายว่าทำไมผลต่างอาจมีหรือไม่มีนัยสำคัญ ขึ้นกับว่างานนั้น compute-bound หรือ
   memory-bound

### แนวทางเฉลยข้อ 1

```cpp
#include <cstdio>
#include <cstdlib>
#include <chrono>

long fib_iterative(long n) {
    if (n <= 1) return n;
    long a = 0, b = 1;
    for (long i = 2; i <= n; i++) {
        long next = a + b;
        a = b;
        b = next;
    }
    return b;
}

int main(int argc, char** argv) {
    long n = (argc > 1) ? atol(argv[1]) : 1000000000L;
    auto t0 = std::chrono::steady_clock::now();
    long result = fib_iterative(n);
    auto t1 = std::chrono::steady_clock::now();
    double ms = std::chrono::duration<double, std::milli>(t1 - t0).count();
    printf("n=%ld result=%ld time=%.3f ms\n", n, result, ms);
    return 0;
}
```

```bash
for level in O0 O1 O2 O3 Os Ofast; do
    g++ -${level} -Wall -Wextra -Wpedantic -std=c++17 fib.cpp -o fib_${level}
done
for level in O0 O1 O2 O3 Os Ofast; do
    echo -n "-${level}: "; ./fib_${level} 1000000000
done
```

ผลลัพธ์จริงที่วัดได้บนเครื่องนี้:

```
-O0: n=1000000000 result=3311503426941990459 time=1600.718 ms
-O1: n=1000000000 result=3311503426941990459 time=370.025 ms
-O2: n=1000000000 result=3311503426941990459 time=368.078 ms
-O3: n=1000000000 result=3311503426941990459 time=363.861 ms
-Os: n=1000000000 result=3311503426941990459 time=365.054 ms
-Ofast: n=1000000000 result=3311503426941990459 time=373.595 ms
```

สังเกตว่าค่า `result` เท่ากันทุก optimization level (`3311503426941990459`) แต่ค่านี้**ไม่ใช่
เลข Fibonacci ตัวที่ 1,000,000,000 จริงตามคณิตศาสตร์** เพราะ Fibonacci โตแบบ exponential
อย่างรวดเร็วมาก ค่าจริงเกินขอบเขตของ `long` (64-bit signed integer สูงสุดประมาณ 9.2×10¹⁸)
ไปตั้งแต่ประมาณลำดับที่ 93 แล้ว ตัวเลขที่เห็นคือผลของ **signed integer overflow ที่เกิดขึ้นซ้ำๆ
หลายล้านครั้ง** ตลอดการวนลูป — นี่คือ Undefined Behavior เช่นเดียวกับที่เรียนในหัวข้อ 89.5
แต่แบบฝึกหัดนี้ไม่ได้ตั้งใจไปโฟกัสที่ประเด็นนั้น (จุดประสงค์หลักคือฝึกวัดผล 6 optimization level)
สิ่งที่ควรสังเกตจากตารางนี้คือรูปแบบเดียวกับตัวอย่าง `sum_to_n` ในหัวข้อ 89.1: `-O0` ช้ากว่า
`-O1` ขึ้นไปถึงประมาณ 4.3 เท่า (1600.7 ms เทียบกับ ~365-374 ms) ในขณะที่ `-O1` ถึง `-Ofast`
ให้เวลาที่ใกล้เคียงกันมาก เพราะลูปนี้เป็น loop-carried dependency แบบง่ายที่ `-O1` ก็เพียงพอจะ
เก็บตัวแปร `a`/`b`/`next` ไว้ใน register ได้เต็มที่แล้ว ไม่มีช่องว่างให้ `-O2`/`-O3` ปรับปรุงต่อได้
มากนัก — ยืนยันบทเรียนสำคัญของ Part นี้อีกครั้งว่าการอัปเกรด optimization level ไม่ได้ให้ผลตอบแทน
เท่ากันเสมอไป ขึ้นอยู่กับโครงสร้างของโค้ดเป็นหลัก

### แนวทางเฉลยข้อ 3

```cpp
// ตัวอย่างนี้มี Undefined Behavior โดยตั้งใจ (out-of-bounds write) เพื่อการศึกษาเท่านั้น
#include <cstdio>

int main() {
    int arr[5] = {1, 2, 3, 4, 5};
    int index = 10;          // นอกขอบเขตของ arr (มีแค่ index 0-4)
    arr[index] = 999;        // UB: เขียนทับหน่วยความจำนอกขอบเขต array
    printf("%d\n", arr[0]);
    return 0;
}
```

```bash
g++ -O0 -Wall -Wextra -Wpedantic -std=c++17 oob.cpp -o oob_O0
g++ -O2 -Wall -Wextra -Wpedantic -std=c++17 oob.cpp -o oob_O2
./oob_O0; echo "exit=$?"
./oob_O2; echo "exit=$?"
```

ผลจริงที่วัดได้บนเครื่องนี้พลิกความคาดหมายของมือใหม่หลายคน:

```
$ ./oob_O0
Segmentation fault
exit=139

$ ./oob_O2
1
exit=0
```

ที่ `-O0` โปรแกรม **crash ด้วย Segmentation Fault** ทันที เพราะ compiler จัด layout ตัวแปรบน
stack แบบตรงไปตรงมาตามลำดับที่ประกาศ ทำให้ offset ที่ `index=10` ชี้ไปตกนอกขอบเขตหน้า
หน่วยความจำที่โปรแกรมได้รับอนุญาตให้เข้าถึง ในขณะที่ `-O2` compiler จัด layout ตัวแปรใหม่หมด
(อาจย้ายตำแหน่ง `arr` หรือแม้แต่ไม่จองพื้นที่ให้ `index` เป็นตัวแปรจริงบน stack เลยเพราะมันเป็น
ค่าคงที่ `10`) ทำให้ offset ที่ผิดพลาดกลับไปตกในพื้นที่ stack ที่ยังใช้งานได้อยู่พอดี โปรแกรมจึง
**"ดูเหมือนทำงานถูกต้อง"** และพิมพ์ `1` ออกมาโดยไม่ crash เลย

นี่คือบทเรียนสำคัญที่สุดของข้อนี้ และเป็นเหตุผลว่าทำไม UB ถึงอันตรายมาก: **ผลลัพธ์ของ UB
ไม่ได้แย่ลงเสมอเมื่อเปิด optimization — บางครั้งมันกลับ "ดูดีขึ้น" (ไม่ crash) ทั้งที่โค้ดยังคงผิด
เหมือนเดิมทุกประการ** ทำให้บั๊กแบบนี้หลุดรอดจากการทดสอบด้วยตาเปล่าได้ง่ายมาก โดยเฉพาะถ้า
ทีมพัฒนาทดสอบที่ build debug (`-O0`) แล้วเห็น crash ชัดเจนจนแก้ไป แต่ถ้าทดสอบที่ build
release (`-O2`) เท่านั้น บั๊กนี้จะไม่ถูกจับเลยแม้แต่น้อย

ตรวจสอบด้วย Sanitizer เพื่อจับ UB นี้ให้ชัดเจน โดยไม่ต้องพึ่งการเดาพฤติกรรมจาก optimization level:

```bash
g++ -O1 -g -fsanitize=address -Wall -Wextra -Wpedantic -std=c++17 oob.cpp -o oob_asan
./oob_asan
```

```
=================================================================
==15724==ERROR: AddressSanitizer: stack-buffer-overflow on address 0x7ff021400048 ...
WRITE of size 4 at 0x7ff021400048 thread T0
    #0 0x561839d8d41b in main .../oob.cpp:7
    ...
  This frame has 1 object(s):
    [32, 52) 'arr' (line 5) <== Memory access at offset 72 overflows this variable
SUMMARY: AddressSanitizer: stack-buffer-overflow .../oob.cpp:7 in main
```

AddressSanitizer รายงาน `stack-buffer-overflow` ทันทีอย่างชัดเจน พร้อมเลขบรรทัดที่ผิด
(`oob.cpp:7`) และยังบอกด้วยว่าตัวแปรที่ถูกเขียนทับเกินขอบเขตคือ `arr` — ไม่ว่าโปรแกรมจริงจะ
crash หรือไม่ crash ที่ optimization level ใดก็ตาม `-fsanitize=address` (จะเรียนละเอียดใน
Part 96) จะจับบั๊กนี้ได้เสมอ เพราะมันตรวจสอบทุกการเข้าถึงหน่วยความจำโดยตรง ไม่ได้อาศัยการสังเกต
ว่าโปรแกรม crash หรือไม่ ซึ่งดังที่เห็นข้างบนเป็นวิธีสังเกตที่เชื่อถือไม่ได้เลย

---

## สรุปท้ายบท

ใน Part นี้เราได้:

- เข้าใจความหมายและ trade-off ของ optimization level ทั้ง 6 แบบของ GCC พร้อมข้อมูลวัดจริง
- เห็น Dead Code Elimination, Constant Folding, Loop Unrolling และ Inlining เกิดขึ้นจริงใน
  Assembly ก่อนและหลัง optimize
- เข้าใจอย่างลึกซึ้งว่า `inline` เป็นคำขอไม่ใช่คำสั่งบังคับ ผ่านการพิสูจน์ด้วย `-Winline` จริง
- เห็นตัวอย่างที่น่าตกใจว่า Undefined Behavior (signed integer overflow) ทำให้โปรแกรมกลายเป็น
  infinite loop ทันทีที่เปิด `-O2` ทั้งที่ `-O0` ดูเหมือนทำงานถูกต้อง
- ใช้ `-march=native` เพื่อปลดล็อกชุดคำสั่ง AVX/FMA เต็มรูปแบบ พร้อมเข้าใจความเสี่ยงเรื่อง
  portability
- ใช้ `-flto` เพื่อให้ compiler มองเห็นและ optimize ข้ามไฟล์ พิสูจน์ด้วย Assembly จริง
- ได้แนวทางเลือก flag ที่เหมาะสมสำหรับแต่ละสถานการณ์ ตั้งแต่ debug จนถึง production release

ความรู้เรื่อง Compiler Optimization ทั้งหมดนี้จะไร้ประโยชน์ถ้าเราไม่มีเครื่องมือวัดผลที่เชื่อถือได้
ทางสถิติ ใน **Part 90** เราจะเรียนรู้ **Google Benchmark** ไลบรารีมาตรฐานอุตสาหกรรมสำหรับ
วัดประสิทธิภาพโค้ด C++ อย่างแม่นยำ แก้ปัญหาการจับเวลาด้วยมือที่มี jitter สูงและเสี่ยงถูก compiler
optimize โค้ดที่กำลังวัดทิ้งไปโดยไม่รู้ตัว

**ต่อไป:** [Part 90 — Benchmarking ด้วย Google Benchmark](./part-090-google-benchmark.md)
