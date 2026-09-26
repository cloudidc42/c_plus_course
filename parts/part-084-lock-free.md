# Part 84: Lock-Free Programming เบื้องต้น (Step 665–672)

> Module G — Concurrency และ Performance Engineering | Part 84 จาก 125
> Step ที่ครอบคลุมใน Part นี้: Step 665–672
> Part ก่อนหน้า: [Part 83 — std::future/async/promise](./part-083-async-future-promise.md) | Part ถัดไป: [Part 85 — Parallel Algorithm (C++17 Execution Policy)](./part-085-parallel-algorithms.md)

## เป้าหมายการเรียนรู้ (Learning Objectives)

หลังจบ Part นี้ ผู้เรียนจะสามารถ:

1. อธิบายได้ว่า mutex มี overhead จากอะไรบ้าง (context switch, kernel involvement, priority inversion)
   และทำไมในบางสถานการณ์ overhead นี้ถึงยอมรับไม่ได้
2. อธิบายแนวคิดของ **lock-free programming** ได้ถูกต้อง และแยกความแตกต่างจาก "wait-free"
   และ "obstruction-free"
3. อธิบายหลักการทำงานของ **Compare-And-Swap (CAS)** ทั้งในระดับ hardware instruction และ
   ระดับภาษา C++
4. ใช้ `std::atomic::compare_exchange_weak` และ `compare_exchange_strong` ได้อย่างถูกต้อง
   รู้ว่าเมื่อไหร่ควรใช้ตัวไหน
5. เขียน **Treiber Stack** — lock-free stack แบบง่ายที่สุด — ด้วย `std::atomic` + CAS loop ได้เอง
   และทดสอบด้วย multi-thread จริง
6. อธิบาย **ABA problem** ได้ พร้อมสาธิตให้เห็นภาพจริง และรู้แนวทางป้องกัน (hazard pointer,
   tagged pointer, epoch-based reclamation) ในระดับแนวคิด
7. ประเมินได้ว่าเมื่อไหร่ "ควร" และ "ไม่ควร" เขียน lock-free data structure เองในงานจริง
   และรู้จัก library ที่ผ่านการทดสอบมาแล้วที่ควรใช้แทน

---

## 84.1 ทำไมต้องมี Lock-Free Programming (Step 665)

ใน Part 82 เราเรียนเรื่อง `std::mutex` ไปแล้วว่าเป็นเครื่องมือหลักในการป้องกัน race condition
เมื่อหลาย thread เข้าถึงข้อมูลร่วมกัน แต่ `std::mutex` ไม่ใช่เครื่องมือที่ "ฟรี" — มันมี **overhead**
ที่ซ่อนอยู่ ซึ่งในระบบที่ต้องการ latency ต่ำมากๆ (เช่น high-frequency trading, audio processing,
real-time system, game engine) overhead นี้อาจกลายเป็นปัญหาใหญ่

### Overhead ของ Mutex มาจากไหน

**1. Context Switch**

เมื่อ thread A พยายาม `lock()` mutex ที่ thread B ถืออยู่ ระบบปฏิบัติการจะ:

1. เปลี่ยนสถานะ thread A จาก "running" เป็น "blocked" (ไม่ใช้ CPU ต่อ)
2. Scheduler เลือก thread อื่นมารันแทน (context switch — บันทึก/โหลด register, page table ฯลฯ)
3. เมื่อ thread B `unlock()` ระบบต้องปลุก thread A กลับมา "runnable" อีกครั้ง
4. Scheduler ต้องตัดสินใจอีกครั้งว่าจะให้ thread A ได้ CPU เมื่อไหร่

กระบวนการนี้ต้อง **เข้า kernel mode** (system call) ซึ่งมีค่าใช้จ่ายสูงกว่าการทำงานใน user mode
มาก — ในเครื่องทั่วไป context switch หนึ่งครั้งอาจใช้เวลาระดับ **ไมโครวินาที (microseconds)**
ในขณะที่การบวกเลขหนึ่งครั้งใช้เวลาระดับ **นาโนวินาที (nanoseconds)** เท่านั้น ต่างกันหลักพันเท่า

**2. Priority Inversion**

ปัญหาคลาสสิกที่เกิดกับระบบ real-time: สมมติมี thread 3 ระดับความสำคัญ — High, Medium, Low

```
Low thread    : lock(mutex) ... กำลังทำงานในส่วนวิกฤต (critical section)
High thread   : พยายาม lock(mutex) เดียวกัน -> ต้องรอ Low thread ปลดล็อกก่อน
Medium thread : ไม่แตะ mutex เลย แต่ใช้ CPU ทำงานยาวๆ แทรกเข้ามา
```

ผลลัพธ์คือ **High priority thread ต้องรอ Medium priority thread** โดยอ้อม ทั้งที่ตามทฤษฎี
High ควรมาก่อน Medium เสมอ — นี่คือ **Priority Inversion** ซึ่งเป็นสาเหตุของบั๊กที่โด่งดังใน
ยานอวกาศ Mars Pathfinder (1997) ที่ระบบ reset ตัวเองซ้ำๆ กลางอวกาศ เพราะ high-priority task
ถูกบล็อกโดย mutex ที่ low-priority task ถืออยู่ ในขณะที่ medium-priority task แย่ง CPU ไปเรื่อยๆ

**3. Contention และ Convoy Effect**

เมื่อหลาย thread แย่ง lock เดียวกันพร้อมกัน (**contention** สูง) thread ที่แพ้จะถูกใส่ใน queue
ของ mutex และต้องรอเป็นทอดๆ ยิ่งจำนวน thread เยอะ ยิ่งเกิด "ขบวนรถติด" (**convoy effect**)
ที่ throughput รวมของระบบลดลงฮวบฮาบ ทั้งที่มี CPU core เหลือว่างอยู่

### ตัวเลขจริง: Mutex vs Atomic

ลองวัด overhead จริงด้วยการเปรียบเทียบการเพิ่มค่าตัวนับ (counter) แบบใช้ `std::mutex`
กับแบบใช้ `std::atomic::fetch_add` (ซึ่งเป็นเทคนิค lock-free พื้นฐานที่สุด):

```cpp
// mutex_vs_atomic.cpp
#include <atomic>
#include <chrono>
#include <iostream>
#include <mutex>
#include <thread>
#include <vector>

constexpr int kThreads = 4;
constexpr long kIterationsPerThread = 2'000'000;

long mutex_counter = 0;
std::mutex mtx;

std::atomic<long> atomic_counter{0};

void increment_with_mutex() {
    for (long i = 0; i < kIterationsPerThread; ++i) {
        std::lock_guard<std::mutex> lock(mtx);
        ++mutex_counter;
    }
}

void increment_with_atomic() {
    for (long i = 0; i < kIterationsPerThread; ++i) {
        atomic_counter.fetch_add(1, std::memory_order_relaxed);
    }
}

template <typename Func>
double measure_ms(Func func) {
    std::vector<std::thread> threads;
    auto start = std::chrono::steady_clock::now();
    for (int i = 0; i < kThreads; ++i) threads.emplace_back(func);
    for (auto& t : threads) t.join();
    auto end = std::chrono::steady_clock::now();
    return std::chrono::duration<double, std::milli>(end - start).count();
}

int main() {
    double mutex_ms = measure_ms(increment_with_mutex);
    double atomic_ms = measure_ms(increment_with_atomic);

    std::cout << "mutex_counter  = " << mutex_counter  << "  เวลา = " << mutex_ms  << " ms\n";
    std::cout << "atomic_counter = " << atomic_counter << "  เวลา = " << atomic_ms << " ms\n";
    std::cout << "atomic เร็วกว่า mutex ประมาณ " << (mutex_ms / atomic_ms) << " เท่า\n";
    return 0;
}
```

คอมไพล์และรันจริง:

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 -pthread -O2 mutex_vs_atomic.cpp -o mutex_vs_atomic
./mutex_vs_atomic
```

ผลลัพธ์จริงที่วัดได้ (บนเครื่อง 4 core ที่ใช้พัฒนาบทเรียนนี้ — ตัวเลขจะต่างกันไปตามเครื่อง
ของแต่ละคน แต่แนวโน้มจะเหมือนกัน):

```
mutex_counter  = 8000000  เวลา = 476.97 ms
atomic_counter = 8000000  เวลา = 128.996 ms
atomic เร็วกว่า mutex ประมาณ 3.7 เท่า
```

จะเห็นว่า `atomic` เร็วกว่า `mutex` ถึง 3-5 เท่าในกรณีนี้ (รันซ้ำหลายครั้งจะได้ผลใกล้เคียงกัน
เสมอ) เพราะ `fetch_add` เป็นเพียง **CPU instruction เดียว** ที่ทำงานแบบ atomic ในระดับฮาร์ดแวร์
โดยไม่ต้องเข้า kernel, ไม่ต้อง context switch, ไม่ต้องมี queue รอคิว — นี่คือแรงจูงใจหลักของ
lock-free programming

---

## 84.2 Lock-Free คืออะไรกันแน่ (Step 666)

**Lock-free programming** คือการเขียนโปรแกรมที่หลาย thread เข้าถึงข้อมูลร่วมกันได้อย่างปลอดภัย
**โดยไม่ใช้ mutex หรือ lock ใดๆ เลย** แต่ใช้ **atomic operation** (โดยเฉพาะ Compare-And-Swap)
เป็นกลไกในการประสานงานแทน

### นิยามที่ต้องแม่นยำ: Lock-Free ≠ ไม่มีการรอ

ความเข้าใจผิดที่พบบ่อยที่สุดคือคิดว่า "lock-free แปลว่าไม่มี thread ไหนต้องรอเลย" ซึ่ง**ไม่จริง**
คำว่า "lock-free" มีนิยามทางวิชาการที่เจาะจงกว่านั้นมาก มันอยู่ในตระกูล **non-blocking guarantee**
ที่แบ่งเป็น 3 ระดับ จากอ่อนไปแก่:

| ระดับ | นิยาม | ตัวอย่าง |
|---|---|---|
| **Obstruction-free** | Thread หนึ่งจะทำงานสำเร็จได้ ถ้ามันได้รันคนเดียวโดยไม่มี thread อื่นมาแทรก (ในทางปฏิบัติแทบไม่มีการรับประกันอะไรเมื่อมีการแข่งขัน) | Algorithm ที่ retry ไม่จำกัดจนกว่าจะไม่มีคนแย่ง |
| **Lock-free** | **อย่างน้อยหนึ่ง thread ในระบบจะคืบหน้าได้เสมอ** ในเวลาจำกัด แม้ thread อื่นจะถูก suspend หรือ crash กลางคัน (แต่ thread ตัวใดตัวหนึ่งอาจต้อง retry ไปเรื่อยๆ ถ้าโชคร้ายโดนแย่งซ้ำๆ) | Treiber Stack ที่เราจะเขียนใน Part นี้ |
| **Wait-free** | **ทุก thread** รับประกันว่าจะเสร็จงานได้ในจำนวนขั้นตอนจำกัด ไม่ว่า thread อื่นจะทำอะไรอยู่ | ยากที่สุด ต้องออกแบบพิเศษ (เช่น wait-free queue ของ Michael & Scott บางเวอร์ชัน) |

ทั้งสามระดับนี้เรียกรวมกันว่า **non-blocking algorithm** เพราะมีคุณสมบัติสำคัญร่วมกันคือ
**ไม่มี thread ไหนสามารถบล็อก thread อื่นได้ตลอดไป** — ถ้า thread A ถูก OS suspend กลางคัน
(เช่น hit page fault, ถูก scheduler เตะออกจาก CPU) thread B, C, D จะยังคงทำงานต่อไปได้
ซึ่งต่างจาก mutex ที่ถ้า thread ที่ถือ lock ถูก suspend หรือตาย ทุก thread ที่รอ lock นั้นจะ
**ค้างตลอดไป**

### หลักการสำคัญ: แทนที่ "ล็อกแล้วแก้" ด้วย "ลองแก้ แล้วเช็คว่าใครแก้ก่อน"

แนวคิดหลักของ lock-free คือเปลี่ยนรูปแบบการทำงานจาก:

```
[แบบ Mutex]  lock() -> อ่าน -> แก้ไข -> เขียนกลับ -> unlock()
             (รับประกันว่าไม่มีใครแทรกระหว่างทาง เพราะกันด้วยกุญแจ)
```

เป็น:

```
[แบบ Lock-Free]  loop {
                     อ่านค่าปัจจุบัน (old_value)
                     คำนวณค่าใหม่จาก old_value (new_value)
                     ลองเขียน new_value เข้าไปแบบ atomic "เฉพาะตอนที่ค่ายังเป็น old_value อยู่"
                     ถ้าสำเร็จ -> จบ
                     ถ้าล้มเหลว (มีคนอื่นแก้ไปก่อนแล้ว) -> วนกลับไปอ่านใหม่แล้วลองอีกครั้ง
                  }
```

รูปแบบ "ลอง-เช็ค-ทำซ้ำ" นี้คือหัวใจของ **Compare-And-Swap (CAS)** ซึ่งเราจะเจาะลึกในหัวข้อถัดไป

---

## 84.3 Compare-And-Swap (CAS) หลักการทำงาน (Step 667)

**Compare-And-Swap** คือ CPU instruction พิเศษที่ทำ 3 อย่างพร้อมกันแบบ **atomic** (ไม่มีใคร
แทรกกลางคันได้ แม้แต่ interrupt หรือ thread อื่นบน core เดียวกัน):

1. อ่านค่าปัจจุบันของหน่วยความจำตำแหน่งหนึ่ง
2. เปรียบเทียบกับค่าที่คาดหวัง (**expected**)
3. ถ้าตรงกัน → เขียนค่าใหม่ (**desired**) ทับลงไป และคืนค่า "สำเร็จ"
   ถ้าไม่ตรงกัน → ไม่เขียนอะไรเลย และคืนค่า "ล้มเหลว" พร้อมค่าจริงปัจจุบัน

เขียนเป็น pseudo-code (ในความเป็นจริงมันคือ CPU instruction เดียว เช่น `CMPXCHG` บน x86
หรือ `LDXR`/`STXR` บน ARM ไม่ใช่ฟังก์ชันที่แยกเป็น 3 ขั้นตอนแบบนี้จริงๆ — แต่เขียนแบบนี้เพื่อ
ให้เห็นภาพ):

```
bool compare_and_swap(T* address, T expected, T desired) {
    // ทั้งหมดนี้เกิดขึ้นเป็น "หนึ่งจังหวะ" ที่แบ่งแยกไม่ได้ (atomic) ในระดับฮาร์ดแวร์
    if (*address == expected) {
        *address = desired;
        return true;
    }
    return false;
}
```

ความมหัศจรรย์อยู่ที่คำว่า "atomic" — ถ้าไม่มี CAS เราจะต้องเขียนโค้ดแบบ

```cpp
if (*address == expected) {   // [1] อ่านและเปรียบเทียบ
    *address = desired;        // [2] เขียนค่าใหม่
}
```

ซึ่งระหว่างขั้นตอน [1] กับ [2] อาจมี thread อื่นมาแก้ `*address` แทรกได้พอดี ทำให้เกิด race
condition — CAS แก้ปัญหานี้โดยรวมทั้งสองขั้นตอนให้เป็น **หนึ่ง CPU instruction** ที่ฮาร์ดแวร์
รับประกันความ atomic ให้โดยตรง (ผ่านกลไก cache coherency protocol ของ CPU เช่น MESI)

### ทำไม CAS ถึงเพียงพอสำหรับสร้างโครงสร้างข้อมูล lock-free เกือบทุกชนิด

ในทางทฤษฎีวิทยาการคอมพิวเตอร์ CAS ถูกพิสูจน์แล้วว่ามี **consensus number เป็นอนันต์**
(ตามงานของ Maurice Herlihy) หมายความว่า CAS สามารถใช้แก้ปัญหา "การตกลงกัน (consensus)"
ของ thread จำนวนเท่าใดก็ได้ ทำให้มันเป็น building block ที่ทรงพลังที่สุดตัวหนึ่งสำหรับสร้าง
lock-free algorithm — ต่างจาก atomic read/write ธรรมดา หรือ `test-and-set` ที่มี consensus
number แค่ 1 หรือ 2 เท่านั้น (รายละเอียดเชิงลึกของทฤษฎีนี้อยู่นอกขอบเขตของหลักสูตร แต่รู้ไว้ว่า
นี่คือเหตุผลว่าทำไม CAS ถึงเป็น "อาวุธหลัก" ของ lock-free programming แทบทุกที่)

---

## 84.4 std::atomic::compare_exchange_weak และ compare_exchange_strong (Step 668)

C++11 เปิดให้เราเรียกใช้ CAS ผ่านเมธอดของ `std::atomic<T>` สองตัว:

```cpp
bool compare_exchange_strong(T& expected, T desired,
                              std::memory_order success = std::memory_order_seq_cst,
                              std::memory_order failure = std::memory_order_seq_cst);

bool compare_exchange_weak(T& expected, T desired,
                            std::memory_order success = std::memory_order_seq_cst,
                            std::memory_order failure = std::memory_order_seq_cst);
```

พฤติกรรมสำคัญที่ต้องจำ:

- `expected` เป็น **reference** — ถ้า CAS ล้มเหลว (ค่าจริงไม่ตรงกับที่คาด) ฟังก์ชันจะ
  **อัปเดตค่าใน `expected` ให้เป็นค่าจริงปัจจุบันโดยอัตโนมัติ** ทำให้เราเอาไปลองใหม่ได้ทันที
  โดยไม่ต้อง `load()` ซ้ำ
- คืนค่า `true` ถ้าสำเร็จ (ค่าถูกเปลี่ยนเป็น `desired`), คืนค่า `false` ถ้าล้มเหลว

ตัวอย่างที่แสดงพฤติกรรมทั้งสองกรณีชัดๆ:

```cpp
// cas_single.cpp
#include <atomic>
#include <iostream>

int main() {
    std::atomic<int> value{10};

    // กรณีสำเร็จ: expected ตรงกับค่าจริงใน value
    int expected1 = 10;
    bool ok1 = value.compare_exchange_strong(expected1, 20);
    std::cout << "CAS ครั้งที่ 1: expected=10, value เดิม=10 -> "
              << std::boolalpha << ok1 << ", value ตอนนี้ = " << value.load() << "\n";

    // กรณีล้มเหลว: expected ไม่ตรงกับค่าจริง (value ถูกเปลี่ยนไปเป็น 20 แล้ว)
    int expected2 = 10;   // ยังคิดว่า value เป็น 10 อยู่ (ข้อมูลเก่า)
    bool ok2 = value.compare_exchange_strong(expected2, 999);
    std::cout << "CAS ครั้งที่ 2: expected=10, value จริง=20 -> "
              << ok2 << ", expected ถูกอัปเดตเป็น = " << expected2
              << ", value ยังคงเป็น = " << value.load() << "\n";
    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 -pthread -O2 cas_single.cpp -o cas_single
./cas_single
```

ผลลัพธ์จริง:

```
CAS ครั้งที่ 1: expected=10, value เดิม=10 -> true, value ตอนนี้ = 20
CAS ครั้งที่ 2: expected=10, value จริง=20 -> false, expected ถูกอัปเดตเป็น = 20, value ยังคงเป็น = 20
```

### weak vs strong: ต่างกันตรงไหน

| | `compare_exchange_strong` | `compare_exchange_weak` |
|---|---|---|
| Spurious failure (ล้มเหลวทั้งที่ค่าตรงกันจริงๆ) | **ไม่มี** — ถ้าค่าตรงกันจะสำเร็จเสมอ | **มีได้** — อาจคืน `false` ทั้งที่ `*address == expected` จริงๆ (พบบนสถาปัตยกรรมที่ใช้ load-linked/store-conditional เช่น ARM, PowerPC) |
| ความเร็วบน loop | อาจช้ากว่าเล็กน้อยในบางสถาปัตยกรรม เพราะภายในมัก implement เป็น loop ของ weak ซ้ำจนกว่าจะชัวร์ | เร็วกว่าเล็กน้อยเมื่อใช้ใน **loop อยู่แล้ว** เพราะไม่ต้อง retry ซ้อน retry |
| เหมาะกับ | เรียกครั้งเดียว ไม่ได้อยู่ใน loop (เช่น "ลองอัปเดต flag ครั้งเดียว") | อยู่ใน `while` loop (รูปแบบที่พบบ่อยที่สุดใน lock-free algorithm) |

**กฎง่ายๆ ที่จำได้เสมอ**: ถ้าโค้ดของเราเป็นรูปแบบ

```cpp
T old_val = atomic_var.load();
T new_val;
do {
    new_val = compute(old_val);
} while (!atomic_var.compare_exchange_weak(old_val, new_val));
```

ให้ใช้ `compare_exchange_weak` เสมอ เพราะเรา retry อยู่แล้วถ้ามัน spurious fail ก็แค่วนอีกรอบ
ซึ่งมักจะเร็วกว่า `strong` บนสถาปัตยกรรมที่ไม่ใช่ x86 (บน x86 ทั้งสองตัวมีประสิทธิภาพเท่ากัน
เพราะ x86 มี `CMPXCHG` instruction ตรงๆ ไม่มี spurious failure อยู่แล้ว แต่โค้ดที่เขียนต้อง
portable ไปยัง ARM ด้วย เช่น Apple M-series, มือถือ, Raspberry Pi)

ตัวอย่างการใช้ `compare_exchange_weak` ใน loop จริง — เพิ่มค่าตัวนับด้วย CAS แทน `fetch_add`
(เพื่อความเข้าใจกลไก แม้ในทางปฏิบัติควรใช้ `fetch_add` ตรงๆ เพราะมันคือ CAS loop ที่คอมไพเลอร์/
ฮาร์ดแวร์ทำให้เราอยู่แล้ว):

```cpp
// cas_basic.cpp
#include <atomic>
#include <iostream>
#include <thread>
#include <vector>

std::atomic<long> counter{0};

void increment_with_cas(int iterations) {
    for (int i = 0; i < iterations; ++i) {
        long old_val = counter.load(std::memory_order_relaxed);
        long new_val;
        do {
            new_val = old_val + 1;
        } while (!counter.compare_exchange_weak(
                     old_val, new_val,
                     std::memory_order_relaxed,
                     std::memory_order_relaxed));
    }
}

int main() {
    const int kThreads = 8;
    const int kIterations = 100000;
    std::vector<std::thread> threads;
    for (int i = 0; i < kThreads; ++i) {
        threads.emplace_back(increment_with_cas, kIterations);
    }
    for (auto& t : threads) t.join();

    std::cout << "counter = " << counter.load()
              << " (expected " << kThreads * kIterations << ")\n";
    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 -pthread -O2 cas_basic.cpp -o cas_basic
./cas_basic
```

ผลลัพธ์จริง (รันซ้ำหลายครั้งได้ผลตรงกันเสมอ เพราะ CAS รับประกัน correctness):

```
counter = 800000 (expected 800000)
```

สังเกตว่า `do-while` loop ข้างในไม่ต้อง `load()` ใหม่หลัง `compare_exchange_weak` ล้มเหลว
เพราะ `old_val` (ที่ส่งเป็น reference) ถูกอัปเดตให้เป็นค่าจริงปัจจุบันให้อัตโนมัติแล้ว — นี่คือ
รูปแบบมาตรฐานของ "CAS retry loop" ที่จะปรากฏซ้ำๆ ตลอด Part นี้

---

## 84.5 เขียน Lock-Free Stack: Treiber Stack — ส่วน Push (Step 669)

ถึงเวลานำความรู้เรื่อง CAS ไปสร้างโครงสร้างข้อมูลจริง **Treiber Stack** (ตั้งชื่อตาม R. Kent
Treiber ผู้เผยแพร่ algorithm นี้ในปี 1986) เป็น lock-free stack ที่ **ง่ายที่สุดในโลกของ
lock-free data structure** และมักเป็นตัวอย่างแรกที่ใช้สอนแนวคิดนี้เสมอ

โครงสร้างพื้นฐานคือ **singly linked list** ที่ `head` เป็น `std::atomic<Node*>` แทนที่จะเป็น
pointer ธรรมดา:

```cpp
template <typename T>
class TreiberStack {
private:
    struct Node {
        T value;
        Node* next;
    };
    std::atomic<Node*> head_{nullptr};

public:
    void push(T value) {
        Node* new_node = new Node{std::move(value), nullptr};
        new_node->next = head_.load(std::memory_order_relaxed);
        while (!head_.compare_exchange_weak(
                   new_node->next, new_node,
                   std::memory_order_release,
                   std::memory_order_relaxed)) {
            // compare_exchange_weak ล้มเหลว: new_node->next ถูกอัปเดตเป็นค่า head_ ปัจจุบันให้อัตโนมัติ
            // วน loop ใหม่โดยไม่ต้อง load ซ้ำ
        }
    }

    // ... pop() จะเขียนในหัวข้อถัดไป
};
```

### อธิบายทีละบรรทัดของ push()

1. สร้าง `new_node` ใหม่บน heap เก็บค่าที่จะ push
2. ตั้งค่า `new_node->next` ให้ชี้ไปที่ `head_` ปัจจุบัน (การเชื่อม node ใหม่เข้ากับ stack เดิม)
3. เข้า loop CAS: "ลองเปลี่ยน `head_` จากค่าที่ `new_node->next` ชี้อยู่ ให้กลายเป็น `new_node`"
   - **ถ้าสำเร็จ**: แปลว่าไม่มีใครมาแก้ `head_` แทรกระหว่างขั้นตอน 2-3 → `head_` ตอนนี้คือ
     `new_node` แล้ว จบการทำงาน
   - **ถ้าล้มเหลว**: แปลว่ามี thread อื่น push (หรือ pop) แทรกเข้ามาก่อน ทำให้ `head_` ไม่ตรง
     กับที่เราคาด → `new_node->next` ถูกอัปเดตเป็นค่า `head_` ล่าสุดโดยอัตโนมัติ (คุณสมบัติของ
     `compare_exchange_weak` ที่อธิบายไปแล้ว) → วน loop ลองใหม่ทันที

**Memory order ทำไมใช้ `release`/`relaxed`**: การ push ต้องรับประกันว่าเมื่อ thread อื่นมองเห็น
`head_` ชี้มาที่ `new_node` แล้ว มันต้องมองเห็น**เนื้อหาข้างใน** `new_node` (ค่า `value` และ
`next`) ที่เขียนเสร็จสมบูรณ์แล้วด้วย — นี่คือหน้าที่ของ `memory_order_release` (เขียนเสร็จก่อน
แล้วค่อย "ปล่อย" ให้คนอื่นเห็น) เราจะเจาะลึกเรื่อง memory order เต็มรูปแบบใน Part 82
(ถ้ายังไม่แน่ใจเรื่องนี้ ให้กลับไปทบทวน Happens-Before Relationship ก่อน)

---

## 84.6 เขียน Lock-Free Stack: Treiber Stack — ส่วน Pop และทดสอบจริง (Step 670)

ส่วน `pop()` มีรูปแบบคล้ายกัน แต่ทิศทางตรงข้าม — เอา `head_` ปัจจุบันออก แล้วเลื่อน `head_`
ไปที่ `next` ของมัน:

```cpp
#include <atomic>
#include <iostream>
#include <optional>
#include <thread>
#include <vector>

template <typename T>
class TreiberStack {
public:
    void push(T value) {
        Node* new_node = new Node{std::move(value), nullptr};
        new_node->next = head_.load(std::memory_order_relaxed);
        while (!head_.compare_exchange_weak(
                   new_node->next, new_node,
                   std::memory_order_release,
                   std::memory_order_relaxed)) {
        }
    }

    std::optional<T> pop() {
        Node* old_head = head_.load(std::memory_order_acquire);
        while (old_head &&
               !head_.compare_exchange_weak(
                   old_head, old_head->next,
                   std::memory_order_acquire,
                   std::memory_order_acquire)) {
            // ล้มเหลว: old_head ถูกอัปเดตเป็นค่าใหม่ วน loop ใหม่
        }
        if (!old_head) return std::nullopt;
        T result = std::move(old_head->value);
        delete old_head;   // ดูหัวข้อ 84.7: นี่คือจุดเสี่ยงของ ABA/use-after-free ถ้ามีหลาย thread pop พร้อมกัน
        return result;
    }

    ~TreiberStack() {
        while (pop()) {}
    }

private:
    struct Node {
        T value;
        Node* next;
    };
    std::atomic<Node*> head_{nullptr};
};

int main() {
    TreiberStack<int> stack;
    const int kProducers = 4;
    const int kItemsPerProducer = 20000;

    std::vector<std::thread> producers;
    for (int p = 0; p < kProducers; ++p) {
        producers.emplace_back([&stack, p]() {
            for (int i = 0; i < kItemsPerProducer; ++i) {
                stack.push(p * kItemsPerProducer + i);
            }
        });
    }
    for (auto& t : producers) t.join();

    std::atomic<int> popped_count{0};
    std::vector<std::thread> consumers;
    for (int c = 0; c < kProducers; ++c) {
        consumers.emplace_back([&stack, &popped_count]() {
            while (stack.pop()) {
                popped_count.fetch_add(1, std::memory_order_relaxed);
            }
        });
    }
    for (auto& t : consumers) t.join();

    std::cout << "popped_count = " << popped_count.load() << " (expected "
              << kProducers * kItemsPerProducer << ")\n";
    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 -pthread -O2 treiber.cpp -o treiber
./treiber
```

ผลลัพธ์จริง (รันซ้ำ 3 ครั้งเพื่อยืนยันความสม่ำเสมอ):

```
popped_count = 80000 (expected 80000)
popped_count = 80000 (expected 80000)
popped_count = 80000 (expected 80000)
```

4 producer thread push พร้อมกันทั้งหมด 80,000 ค่า และ 4 consumer thread pop พร้อมกันจนหมด
โดย**ไม่มี mutex เลยสักบรรทัดเดียว** — นี่คือพลังของ lock-free programming: หลาย thread
เข้าถึงโครงสร้างข้อมูลร่วมกันได้อย่างปลอดภัย (ในแง่ความถูกต้องของจำนวนที่ pop ได้) โดยอาศัย
CAS ล้วนๆ

> **หมายเหตุสำคัญ**: ผลลัพธ์ข้างต้น "ถูกต้อง" ในแง่จำนวน element ที่ pop ได้ครบ แต่โค้ดนี้
> **ยังมีบั๊กด้านความปลอดภัยของหน่วยความจำ** ที่ซ่อนอยู่เมื่อมีหลาย consumer thread pop พร้อมกัน
> จะอธิบายรายละเอียดในหัวข้อ 84.7 (ABA problem) — การรันแล้ว "ดูเหมือนถูก" ไม่ได้แปลว่า
> โค้ด lock-free ปลอดภัย 100% เสมอไป นี่คือกับดักที่อันตรายที่สุดของสาย lock-free

---

## 84.7 ABA Problem (Step 671)

**ABA problem** คือปัญหาคลาสสิกที่สุดของ lock-free programming เกิดจากข้อจำกัดพื้นฐานของ CAS:
**CAS เช็คได้แค่ "ค่าตรงกันหรือไม่" มันไม่รู้ "ประวัติ" ว่าค่านั้นเคยถูกเปลี่ยนไปแล้วกลับมาเป็น
เหมือนเดิมหรือเปล่า**

### สาธิตปัญหาด้วยตัวอย่างง่ายๆ

```cpp
// aba_demo.cpp
#include <atomic>
#include <chrono>
#include <iostream>
#include <thread>

int main() {
    std::atomic<int> value{100};

    // Thread A: อ่านค่า แล้ว "หน่วงเวลา" ก่อน CAS (จำลองการโดน context switch)
    std::thread thread_a([&value]() {
        int expected = value.load();               // อ่านได้ 100
        std::this_thread::sleep_for(std::chrono::milliseconds(100));
        bool success = value.compare_exchange_strong(expected, 999);
        std::cout << "Thread A: CAS(100 -> 999) success = " << std::boolalpha
                  << success << ", value ตอนนี้ = " << value.load() << "\n";
    });

    // Thread B: เปลี่ยนค่า 100 -> 200 -> กลับมาเป็น 100 ก่อน Thread A จะ CAS เสร็จ
    std::thread thread_b([&value]() {
        std::this_thread::sleep_for(std::chrono::milliseconds(20));
        value.store(200);
        std::this_thread::sleep_for(std::chrono::milliseconds(20));
        value.store(100);   // ค่ากลับมาเป็น 100 เหมือนเดิม แต่ "ประวัติ" เปลี่ยนไปแล้ว
        std::cout << "Thread B: เปลี่ยนค่า 100 -> 200 -> 100 เสร็จแล้ว\n";
    });

    thread_a.join();
    thread_b.join();
    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 -pthread -O2 aba_demo.cpp -o aba_demo
./aba_demo
```

ผลลัพธ์จริง:

```
Thread B: เปลี่ยนค่า 100 -> 200 -> 100 เสร็จแล้ว
Thread A: CAS(100 -> 999) success = true, value ตอนนี้ = 999
```

Thread A `compare_exchange_strong` สำเร็จ (`success = true`) เพราะตอนที่มัน CAS ค่าของ
`value` "บังเอิญ" กลับมาเป็น 100 พอดี (เหมือนตอนที่ Thread A อ่านครั้งแรก) — CAS จึงมองว่า
"ค่าไม่เปลี่ยน" ทั้งที่จริงๆ แล้วมันเปลี่ยนไปเป็น 200 แล้วเปลี่ยนกลับมาเป็น 100 (A**→**B**→**A)
สองครั้ง ระหว่างที่ Thread A ไม่ได้มอง

### ทำไมมันถึงอันตรายกับ Treiber Stack

ในกรณีของ `int` ธรรมดา ABA อาจไม่มีผลเสียหายอะไร (100 ก็คือ 100 ไม่ว่าจะผ่านมากี่รอบ) แต่ใน
กรณีของ **pointer** (เช่น `Node*` ใน Treiber Stack) ปัญหาจะร้ายแรงกว่ามาก เพราะ "ที่อยู่
หน่วยความจำเดิม" อาจถูก `delete` ไปแล้ว และถูกจัดสรรใหม่ (`new`) ให้กลายเป็น object อื่นที่
บังเอิญได้ address เดียวกัน! ลำดับเหตุการณ์ที่เป็นไปได้ในโค้ด `pop()` ของเรา:

```
1. Thread A เรียก pop() -> อ่าน old_head = Node@0x1000 (สมมติค่านี้)
2. Thread A ถูก scheduler สลับออกไป (ก่อนจะถึงบรรทัด compare_exchange_weak)
3. Thread B เรียก pop() สำเร็จ -> เอา Node@0x1000 ออกจาก stack -> delete Node@0x1000
4. Thread B เรียก push() ค่าใหม่ -> new Node -> allocator คืน address 0x1000 กลับมาให้ (บังเอิญ
   ซ้ำ เพราะเพิ่ง free ไปหมาดๆ) -> Node@0x1000 กลายเป็น node ใหม่ที่มี ->next ต่างจากเดิม!
5. Thread A ถูกปลุกกลับมาทำงานต่อ -> compare_exchange_weak(old_head=Node@0x1000, ...)
   เช็คว่า head_ ยังชี้ไปที่ 0x1000 อยู่ไหม -> "ใช่!" (เพราะ Node ใหม่ดันได้ address เดิม)
   -> CAS สำเร็จ! -> head_ ถูกตั้งเป็น old_head->next ซึ่งเป็นค่า next ของ "Node เก่า" ที่ตายไปแล้ว
   ไม่ใช่ next ของ Node ใหม่ที่เพิ่ง push เข้าไป -> stack พังโครงสร้าง (corrupted linked list)
```

นี่คือ **use-after-free bug ที่แฝงตัวอยู่ในโค้ด Treiber Stack ฉบับพื้นฐาน** ที่เราเขียนไปในหัวข้อ
84.6 — โค้ดรันผ่านและให้ผลลัพธ์ถูกต้องในการทดสอบข้างต้น เพราะเป็นบั๊กที่ต้องอาศัย**จังหวะโชคร้าย
พอดี** (timing-dependent) ถึงจะเกิด ซึ่งอาจไม่เกิดเลยเป็นพันๆ ครั้งที่รัน แล้วอยู่ๆ ก็เกิดขึ้นใน
production หลังจากใช้งานมาเป็นปี — นี่คือเหตุผลที่บั๊ก lock-free ถูกจัดว่าเป็นหนึ่งในบั๊กที่
**debug ยากที่สุด** ในวงการ

### แนวทางป้องกัน ABA Problem (ระดับแนวคิด)

| เทคนิค | หลักการ | ข้อเสีย |
|---|---|---|
| **Tagged Pointer / ABA Counter** | แนบตัวนับ (counter) ไปกับ pointer ทุกครั้งที่เปลี่ยนค่า (เช่นใช้ `std::atomic<std::pair<Node*, uint64_t>>` หรือ pack ใส่ใน 128-bit ด้วย `DCAS`) ทำให้ CAS เช็คทั้ง pointer และ counter พร้อมกัน ถ้า pointer ถูกเปลี่ยนไปกี่รอบก็ตาม counter จะไม่มีวันซ้ำ | ต้องการ double-width CAS (`cmpxchg16b` บน x86-64) ซึ่งไม่ทุกแพลตฟอร์มรองรับ |
| **Hazard Pointer** | แต่ละ thread ประกาศ "pointer ที่กำลังใช้งานอยู่ตอนนี้" ไว้ใน global registry ก่อน `delete` node ใดๆ ต้องเช็คก่อนว่าไม่มี thread ไหนประกาศ hazard pointer ชี้ไปที่ node นั้นอยู่ | ซับซ้อนมากในการ implement ให้ถูกต้อง มี overhead จากการ scan registry |
| **Epoch-Based Reclamation (EBR)** | แบ่งเวลาเป็น "epoch" thread ที่กำลังทำงานอยู่จะ pin ตัวเองไว้ที่ epoch ปัจจุบัน หน่วยความจำที่ถูก "free" จะไม่ถูกคืนจริงจนกว่าทุก thread จะผ่าน epoch นั้นไปแล้ว (ใช้ใน library อย่าง Facebook Folly, Linux kernel RCU) | ต้องมี background thread หรือกลไก reclaim แยก เพิ่มความซับซ้อนของระบบโดยรวม |
| **Garbage-Collected Language** | ในภาษาที่มี GC (Java, Go, C#) ปัญหานี้เบาลงมาก เพราะหน่วยความจำจะไม่ถูกคืนกลับให้ OS จนกว่าจะไม่มีใครอ้างอิงถึงจริงๆ | C++ ไม่มี GC ในตัว ต้องจัดการเอง |

การ implement เทคนิคเหล่านี้อย่างถูกต้องซับซ้อนเกินขอบเขตของ Part เบื้องต้นนี้ (แต่ละเทคนิค
สมควรมีเนื้อหาแยกเป็น chapter ของตัวเองในหนังสือ lock-free ระดับสูง) สิ่งที่ผู้เรียนต้องจดจำ
จาก Part นี้คือ **ต้องรู้ว่าปัญหานี้มีอยู่จริง และรู้ว่ามันแก้ได้ด้วยเทคนิคอะไรบ้างในระดับแนวคิด**
ส่วนการ implement จริงในงานสาย production ให้ใช้ library ที่ทดสอบมาแล้ว (หัวข้อถัดไป)

---

## 84.8 ข้อควรระวังอย่างจริงจัง: เมื่อไหร่ควร (และไม่ควร) เขียน Lock-Free เอง (Step 672)

หลังจากเห็นทั้งพลังและกับดักของ lock-free programming แล้ว ต้องพูดตรงๆ อย่างจริงจังว่า:

> **Lock-free programming เขียนยากกว่า และ debug ยากกว่า mutex-based programming มาก — มากจน
> ในงานจริงส่วนใหญ่ ไม่คุ้มที่จะเขียนโครงสร้างข้อมูล lock-free เองตั้งแต่ต้น**

### เหตุผลที่ lock-free ยากกว่าที่คิด

1. **บั๊กเป็นแบบ timing-dependent** — ตามที่เห็นใน ABA problem โค้ดอาจรันถูกต้อง 99,999 ครั้ง
   แล้วพังในครั้งที่ 100,000 พอดี เพราะจังหวะ scheduler ไม่เอื้ออำนวย ทำให้ unit test ทั่วไป
   **จับบั๊กพวกนี้ไม่ได้เลย**
2. **Memory reclamation คือปัญหาที่แท้จริง** — การตัดสินใจว่า "เมื่อไหร่ปลอดภัยที่จะ `delete`
   node" (โดยไม่ทำให้ thread อื่นที่กำลังอ่าน node นั้นอยู่พังตาม) คือปัญหาที่ยากที่สุดของสาย
   lock-free ทั้งวงการ ยากกว่าตัว algorithm หลักด้วยซ้ำ
3. **การพิสูจน์ความถูกต้องต้องใช้ทฤษฎีขั้นสูง** — นักวิจัย lock-free ใช้เครื่องมืออย่าง model
   checker (TLA+, SPIN) เพื่อพิสูจน์ว่า algorithm ถูกต้องภายใต้ทุก interleaving ที่เป็นไปได้
   ของ thread ซึ่งเป็นจำนวนมหาศาลเกินกว่ามนุษย์จะไล่เช็คเองด้วยตา
4. **Memory order ผิดแม้แต่นิดเดียวก็พังได้** — การเลือก `memory_order` ผิด (เช่นใช้ `relaxed`
   ตรงจุดที่ควรใช้ `acquire`/`release`) อาจทำให้โค้ดทำงานถูกต้อง 100% บน x86 (ซึ่งมี memory
   model ที่ "เข้มงวด" อยู่แล้วโดยธรรมชาติของฮาร์ดแวร์) แต่พังทันทีบน ARM (ซึ่งมี memory model
   ที่ "หลวม" กว่ามาก) ทำให้บั๊กแบบนี้มักไม่ถูกพบจนกว่าจะ deploy ไปรันบนแพลตฟอร์มอื่น
5. **Portability** — lock-free algorithm บางแบบพึ่งพา instruction เฉพาะแพลตฟอร์ม (เช่น
   `cmpxchg16b`) ที่ไม่มีในทุก CPU

### คำแนะนำที่ควรยึดถือในงานจริง

| สถานการณ์ | สิ่งที่ควรทำ |
|---|---|
| งานทั่วไป ต้องการ thread-safe container | ใช้ `std::mutex` + container ปกติ (`std::vector`, `std::queue`) — ง่าย ปลอดภัย พิสูจน์ถูกต้องได้ง่าย |
| Contention สูงมากจริงๆ วัด profile แล้วพบว่า mutex คือ bottleneck จริง | พิจารณาใช้ **library ที่ทดสอบมาแล้วในงาน production จริง** เช่น `boost::lockfree::queue`, `boost::lockfree::stack`, `moodycamel::ConcurrentQueue`, Intel TBB's `concurrent_queue` แทนการเขียนเอง |
| ต้องการ atomic counter/flag ธรรมดา | ใช้ `std::atomic<T>` ตรงๆ (เช่น `fetch_add`, `exchange`) — นี่คือ "lock-free" ระดับพื้นฐานที่สุด และปลอดภัยเพราะ compiler/hardware implement ให้อย่างถูกต้องแล้ว ไม่ต้องเขียน CAS loop เอง |
| งานวิจัย/เรียนรู้เชิงลึก | เขียนเอง! แต่ทดสอบด้วยเครื่องมืออย่าง **ThreadSanitizer** (`-fsanitize=thread`) อย่างเข้มข้น และรันซ้ำหลายพันครั้งบน CPU หลาย core เพื่อหาโอกาสให้บั๊ก timing-dependent โผล่ออกมา |

**กฎทองของ Part นี้**: การเขียน `TreiberStack` เองใน Part นี้มีจุดประสงค์เพื่อ**ความเข้าใจ**
กลไกเบื้องหลังของ lock-free programming เท่านั้น — ในโค้ด production จริง ให้ใช้ library ที่
ผ่านการทดสอบอย่างเข้มข้นจากทีมวิจัยและวิศวกรระดับโลกมาแล้วนับหมื่นชั่วโมง เว้นแต่จะมีเหตุผล
ที่ชัดเจนและวัดผลได้จริงว่าจำเป็นต้องเขียนเอง (เช่น ทำงานในระบบ embedded ที่ไม่มี dynamic
allocation ให้ link library ภายนอกได้)

---

## ข้อผิดพลาดที่พบบ่อย (Common Pitfalls)

1. **คิดว่า `std::atomic` ทำให้ทุกอย่าง thread-safe อัตโนมัติ** — `std::atomic<T>` รับประกัน
   ความปลอดภัยแค่กับ**การอ่าน/เขียนตัวแปรนั้นตัวเดียว** เท่านั้น ถ้าต้องอัปเดตค่าหลายตัวให้
   สอดคล้องกัน (เช่น "เปลี่ยน `head` และ `size` พร้อมกัน") การทำ `head.store(...)` ตามด้วย
   `size.store(...)` แยกกันสองคำสั่ง**ไม่ปลอดภัย** เพราะระหว่างสองคำสั่งนี้ thread อื่นอาจ
   มองเห็นสถานะที่ไม่สอดคล้องกัน (`head` เปลี่ยนแล้วแต่ `size` ยังไม่เปลี่ยน)

2. **ลืม ABA problem แล้วเขียน pointer-based lock-free structure โดยไม่มีการป้องกันเลย** —
   ตามที่สาธิตในหัวข้อ 84.7 บั๊กนี้ไม่ปรากฏในการทดสอบทั่วไปแต่จะโผล่มาใน production ภายใต้
   load สูงและจังหวะโชคร้าย

3. **ใช้ `compare_exchange_strong` ใน loop โดยไม่จำเป็น** — ถ้าอยู่ใน retry loop อยู่แล้ว
   ควรใช้ `compare_exchange_weak` เสมอเพื่อประสิทธิภาพที่ดีกว่าบนสถาปัตยกรรมที่ไม่ใช่ x86

4. **มองข้าม memory order แล้วใช้ default (`seq_cst`) ทุกที่โดยไม่คิด** — `seq_cst` ปลอดภัย
   ที่สุดแต่ก็ช้าที่สุด (ต้องมี memory fence เต็มรูปแบบ) การเลือก `acquire`/`release`/`relaxed`
   ให้ถูกจุดช่วยเพิ่มประสิทธิภาพได้มาก แต่ **ต้องเข้าใจ Happens-Before Relationship อย่างลึกซึ้ง
   ก่อนเปลี่ยนจาก default** ไม่เช่นนั้นจะกลายเป็นการสร้างบั๊กที่มองไม่เห็นแทน

5. **เขียน lock-free structure เองในโปรเจกต์ production โดยไม่มีเหตุผลจำเป็นจริง** — ตามที่
   เน้นย้ำในหัวข้อ 84.8 การเขียนเองมีความเสี่ยงสูงกว่าที่ผู้เขียนโค้ดส่วนใหญ่ประเมินไว้มาก
   ควรใช้ library ที่ผ่านการทดสอบแล้วเป็นค่าเริ่มต้นเสมอ

6. **ลืม `-pthread` ตอนคอมไพล์โปรแกรมที่ใช้ `std::thread`/`std::atomic` กับหลาย thread** —
   แม้ `std::atomic` เองไม่จำเป็นต้อง link `pthread` เสมอไป แต่โปรแกรมที่สร้าง `std::thread`
   จริงต้อง `-pthread` เสมอบน Linux/GCC ไม่เช่นนั้นอาจ link ไม่ผ่านหรือเกิดพฤติกรรมแปลกๆ
   ตอนรัน

---

## แบบฝึกหัดท้ายบท

1. เขียนฟังก์ชัน `atomic_max(std::atomic<int>& value, int candidate)` ที่อัปเดต `value` ให้
   เป็นค่าที่มากกว่าระหว่าง `value` ปัจจุบันกับ `candidate` โดยใช้ CAS loop (ห้ามใช้ mutex)
   ทดสอบด้วยหลาย thread ที่เรียกพร้อมกันด้วยค่าสุ่มต่างๆ แล้วยืนยันว่าผลลัพธ์สุดท้ายคือค่า
   มากที่สุดจริงๆ

2. เพิ่มเมธอด `size()` ให้กับ `TreiberStack` ที่เขียนใน Part นี้ โดยใช้ `std::atomic<int>`
   แยกต่างหากที่อัปเดตคู่กับ `push`/`pop` อธิบายว่าทำไมค่าที่ได้จาก `size()` อาจไม่ตรงกับ
   จำนวนจริง "ในบางจังหวะ" ถ้ามีหลาย thread ทำงานพร้อมกัน (hint: มันคือค่าที่ "เคย" ถูกต้อง
   ณ จังหวะใดจังหวะหนึ่ง ไม่ใช่ atomic snapshot ของทั้งสอง field พร้อมกัน)

3. แก้ไข `aba_demo.cpp` ให้ Thread A ใช้ `compare_exchange_weak` แทน `compare_exchange_strong`
   แล้วสังเกตว่าผลลัพธ์เปลี่ยนไปหรือไม่ อธิบายว่าทำไม (hint: บน x86 ผลจะไม่ต่างกันเลย)

4. เขียนโปรแกรมที่มี `std::atomic<bool> flag{false}` และใช้ `compare_exchange_strong` เพื่อ
   ทำให้แน่ใจว่ามีแค่ thread เดียวเท่านั้นที่จะพิมพ์ข้อความ "ฉันมาถึงก่อน!" ได้ ทั้งที่มี 10
   thread แข่งกันเรียกพร้อมกัน (นี่คือรูปแบบพื้นฐานของ "compare-and-set flag" ที่ใช้บ่อยมาก
   ในโค้ด initialization แบบ thread-safe)

5. อธิบายด้วยคำพูดของตัวเอง (ไม่ต้องเขียนโค้ด) ว่าทำไม `TreiberStack::pop()` ในหัวข้อ 84.6
   ถึงยังมีความเสี่ยงเรื่อง use-after-free เมื่อมีหลาย consumer thread pop พร้อมกัน แม้ว่า
   ผลการทดสอบจำนวน `popped_count` จะออกมาถูกต้องเสมอก็ตาม

6. ค้นคว้าเพิ่มเติม (ไม่ต้องเขียนโค้ด): เปรียบเทียบ `boost::lockfree::stack` กับ
   `moodycamel::ConcurrentQueue` ว่าแต่ละตัวเหมาะกับ use case แบบไหน (single-producer
   single-consumer vs multi-producer multi-consumer) และรายงานสรุปสั้นๆ

### แนวทางเฉลยข้อ 1

```cpp
// exercise1_atomic_max.cpp
#include <atomic>
#include <iostream>
#include <random>
#include <thread>
#include <vector>

void atomic_max(std::atomic<int>& value, int candidate) {
    int current = value.load(std::memory_order_relaxed);
    while (candidate > current &&
           !value.compare_exchange_weak(current, candidate,
                                         std::memory_order_relaxed,
                                         std::memory_order_relaxed)) {
        // ล้มเหลว: current ถูกอัปเดตเป็นค่าจริงปัจจุบันแล้ว วน loop ใหม่
        // เงื่อนไข candidate > current จะถูกเช็คใหม่ทุกรอบ
        // ถ้า current กลายเป็นค่าที่มากกว่า candidate อยู่แล้ว loop จะหยุดโดยไม่ทำอะไร (ถูกต้อง)
    }
}

int main() {
    std::atomic<int> value{0};
    const int kThreads = 8;
    std::vector<std::thread> threads;
    std::vector<int> candidates = {42, 17, 99, 3, 250, 88, 5, 199};

    for (int i = 0; i < kThreads; ++i) {
        threads.emplace_back(atomic_max, std::ref(value), candidates[i]);
    }
    for (auto& t : threads) t.join();

    std::cout << "ค่าสูงสุดที่ได้ = " << value.load() << " (คาดหวัง 250)\n";
    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 -pthread -O2 exercise1_atomic_max.cpp -o ex1
./ex1
# ค่าสูงสุดที่ได้ = 250 (คาดหวัง 250)
```

จุดสำคัญของเฉลยนี้คือเงื่อนไข `candidate > current` ต้องอยู่**ใน**เงื่อนไขของ `while` (เช็คใหม่
ทุกรอบ) ไม่ใช่เช็คแค่ครั้งเดียวก่อนเข้า loop เพราะระหว่างที่เรา retry อยู่ อาจมี thread อื่นได้
อัปเดต `value` ให้มากกว่า `candidate` ของเราไปแล้ว ซึ่งถ้าเป็นแบบนั้นเราต้องหยุดทันทีโดยไม่เขียน
ทับค่าที่มากกว่าด้วยค่าที่น้อยกว่า

### แนวทางเฉลยข้อ 4

```cpp
// exercise4_first_flag.cpp
#include <atomic>
#include <iostream>
#include <thread>
#include <vector>

std::atomic<bool> flag{false};

void race_to_be_first(int id) {
    bool expected = false;
    if (flag.compare_exchange_strong(expected, true)) {
        std::cout << "Thread " << id << " มาถึงก่อน!\n";
    }
    // thread อื่นที่ CAS ล้มเหลว (expected ถูกอัปเดตเป็น true) จะไม่พิมพ์อะไรเลย
}

int main() {
    const int kThreads = 10;
    std::vector<std::thread> threads;
    for (int i = 0; i < kThreads; ++i) {
        threads.emplace_back(race_to_be_first, i);
    }
    for (auto& t : threads) t.join();
    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 -pthread -O2 exercise4_first_flag.cpp -o ex4
./ex4
# ตัวอย่างผลลัพธ์ (thread ไหนชนะขึ้นกับ scheduler แต่ละครั้ง):
# Thread 3 มาถึงก่อน!
```

หลักการสำคัญ: `compare_exchange_strong(expected, true)` จะสำเร็จได้เพียง**ครั้งเดียวเท่านั้น**
ในทั้งโปรแกรม เพราะหลังจาก thread แรกเปลี่ยน `flag` เป็น `true` สำเร็จ ทุก thread ที่มาทีหลัง
จะพบว่า `expected` (`false`) ไม่ตรงกับค่าจริง (`true`) จึง CAS ล้มเหลวเสมอ — นี่คือรูปแบบ
พื้นฐานที่สุดของ "การใช้ CAS เพื่อการันตี exactly-once execution" ที่ใช้ใน production จริง
บ่อยมาก เช่น การทำ thread-safe lazy initialization

---

## สรุปท้ายบท

ใน Part นี้เราได้:

- เข้าใจ overhead จริงของ `std::mutex` (context switch, priority inversion, convoy effect)
  และเห็นตัวเลขเปรียบเทียบจริงระหว่าง mutex กับ atomic
- แยกความแตกต่างของ obstruction-free, lock-free, wait-free ได้อย่างแม่นยำ
- เข้าใจหลักการทำงานของ Compare-And-Swap (CAS) ทั้งระดับ concept และระดับ C++
- ใช้ `compare_exchange_weak`/`compare_exchange_strong` ได้อย่างถูกต้อง รู้ว่าเมื่อไหร่ควร
  ใช้ตัวไหน
- เขียน Treiber Stack แบบ lock-free เต็มรูปแบบด้วยตัวเอง และทดสอบด้วย multi-thread จริง
- เข้าใจ ABA problem อย่างลึกซึ้งผ่านการสาธิตจริง พร้อมรู้แนวทางป้องกันในระดับแนวคิด
  (tagged pointer, hazard pointer, epoch-based reclamation)
- ได้รับคำแนะนำที่ตรงไปตรงมาว่า lock-free programming เขียนยากและ debug ยากกว่า mutex มาก
  ในงานจริงควรใช้ library ที่ทดสอบมาแล้วเป็นค่าเริ่มต้นเสมอ

Lock-free programming เป็นหนึ่งในหัวข้อที่ "ลึกที่สุด" ของ concurrency — สิ่งที่เรียนใน Part นี้
เป็นเพียงยอดภูเขาน้ำแข็ง แต่ก็เพียงพอที่จะทำให้เข้าใจโค้ดในระดับ production ที่ใช้ atomic/CAS
ได้ และรู้จักประเมินความเสี่ยงเมื่อต้องตัดสินใจว่าจะเขียนเองหรือใช้ library ใน **Part 85**
เราจะเปลี่ยนโฟกัสไปที่การใช้ประโยชน์จาก multi-core ในอีกมุมหนึ่ง: **Parallel Algorithm ของ
C++17** ที่ทำให้ STL algorithm ทำงานแบบขนานได้ง่ายๆ เพียงเพิ่ม parameter ตัวเดียว

**ต่อไป:** [Part 85 — Parallel Algorithm (C++17 Execution Policy)](./part-085-parallel-algorithms.md)
